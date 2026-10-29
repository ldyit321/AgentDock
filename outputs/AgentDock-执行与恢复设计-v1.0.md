# AgentDock 执行与恢复设计

版本：v1.0 评审稿；日期：2026-10-28。范围：执行调度、持久化一致性、问答与审批、文件恢复及故障边界。本文全部新增实现选择为 **D（建议）**，兼容性与故障注入为 **O（待验证）**；用户确认和实测范围以需求基线为准。本文不是实施授权，未安装依赖、启动服务或修改现有 Server。

## 1. 栈与职责

建议 Windows Python 3.12、FastAPI、LangGraph 1.x、SQLAlchemy 2、Alembic、psycopg 3 与 PostgreSQL 17。数据库建议作为新增独立本地 Docker 服务运行；它尚未安装或初始化，不能与现有 OpenSandbox Server 混称一个服务。版本小版本和依赖锁定在实施前验证。

API 处理身份、输入、权限、取消及答复落库，不在 HTTP 请求内运行图。一个独立 Python worker 进程以异步任务提供有限并发；全局和每用户活动执行上限由配置决定。PostgreSQL `runs` 表兼作持久队列；不用 Redis/Celery，不创建微服务框架。worker 内的调度器、执行器、等待归并器和清理扫描器是清楚的模块，不是新的服务。LangGraph 检查点也存 PostgreSQL，但与业务表的事务边界分开；同库不代表天然同事务。

业务服务接收显式身份、运行上下文、工作区和适配接口，不接收 HTTP 对象或 SDK 内部对象。图保存可序列化数据、引用、预算和决定；数据库连接、MCP 客户端、Sandbox 管理 key、Bearer、签名下载地址和凭据明文仅存在于执行边界。检查点本身仍可能含用户敏感输入，需按归属保护和清理。

## 2. 数据约束与状态

命名以主技术设计实体表为准：本文的fence指runs.lease_epoch，worker_id对应lease_owner，不另建同义列。交互组为逻辑抽象：待用户处理时questions.status=open、approvals.status=pending；下文泛称“pending组”不代表问题表也新增pending枚举。问题答复为answered，审批答复为approved/denied，preparing阶段都不对用户开放提交。

建议关键表：`runs`、`run_events`、`tool_calls`、`questions`、`question_responses`、`approvals`、`resume_requests`、`todos/todo_changes`、`sandbox_leases`、`workspace_snapshots`、文件索引及 LangGraph saver 表。各表显式关联 owner、conversation 和 run；所有入口先按当前身份查询，不能信任请求中的 user_id、路径、问题 ID 或 thread_id。跨表归属使用复合约束或服务端事务校验，不能只靠 UUID 难猜。

`runs` 至少含 status、version、worker_id、lease_until、fence、heartbeat_at、cancel_requested_at、deadline_at、wait_deadline_at、budget_used、checkpoint_ref、graph_version、next_event_seq。数据库时间为租约和过期判定依据。默认隔离级别下对涉及一致性的 run 行使用行锁；事务短小，不在持锁期间等待模型、SDK 或用户。

| 状态 | 含义及出路 |
|---|---|
| queued | 初次输入已接受，尚未 claim；可取消或到期 |
| preparing | claim 后校验配置、准备工作区与实例 |
| running | 图与工具执行；可进入校验、等待、失败或取消 |
| validating | 按任务产物要求和证据检查；成功或失败 |
| waiting_user / waiting_approval | 等待已持久化且可恢复的交互；不占执行槽 |
| resuming | 答复已接受、待 claim 或正在重建环境；不是已执行成功 |
| cancelling | 已接受取消，正在停止后续调用并处理在途动作 |
| succeeded / failed / cancelled | 已知业务结果终态 |
| interrupted | 崩溃或副作用未知，停止自动执行，需核对 |
| expired | Run 或等待期限已到；禁止迟到答复启动执行 |

活动状态为 queued、preparing、running、validating、waiting_user、waiting_approval、resuming、cancelling。对 conversation_id 建活动状态部分唯一索引，保证一个会话唯一活动 Run；事务创建 Run、输入消息和初始事件。等待期间仍占会话活动锁，新的普通轮次返回冲突并引导答复/取消；不偷偷变成第二个运行。上传可作为当前待答交互附件，不等于启动新轮次。interrupted 为停止执行的终态；同会话新 Run 前，若存在未核对的外部写，业务服务额外拒绝启动，避免通过新 Run 绕过未知动作。

清理状态独立记录：not_created、active、cleanup_pending、cleaned、cleanup_failed、cleanup_unverified；替代实例各有一条 lease。succeeded 可以与 cleanup_failed 并存，网页必须同时展示。

## 3. Claim、心跳与隔离旧 worker

调度器仅选 queued 和具备恢复条件的 resuming，通过 `FOR UPDATE SKIP LOCKED` 锁定候选，事务内检查用户/全局限额、更新 worker_id、lease_until 并令 fence 单调递增。首次 queued 改 preparing；恢复保留 resuming。活动执行限额统计租约有效的执行状态；单 worker 仍保留数据库约束，防止误启动第二个进程。建议另加 Windows OS 级进程运行锁（D），worker 启动时取得并在整个生命周期持有，第二个进程取锁失败即退出；不根据数据库租约超时自动夺取仍存活进程的 OS 锁。仅确认旧进程退出并取得锁后才做启动归并；PID 文件只能诊断，不能代替真实互斥锁。

执行上下文携带 `(run_id, worker_id, fence)`。独立心跳协程按配置频率续租，只在原 fence、持有者和允许状态均匹配时更新；续租频率应明显短于租约。心跳失败、租约过期或更新为零行后，立即停止调度新动作并取消本地任务。每次业务写、事件提交、创建实例与工具发送前，都重新检查 fence、归属、预算、取消标记和权限；工具回包也必须 fenced 更新，旧 worker 不能覆盖新状态。

租约过期不把 running 自动改回 queued。扫描器锁行、增加 fence、标 interrupted，记录未完成动作和待清理实例；不自动重放图。等待态没有执行租约，不因 worker 重启被当成失联运行。queued 未 claim 可正常重启后调度；resuming 若尚无 resume 发送记录可重新 claim，已经开始图恢复而未确认落点的则先归并，不能直接再发 Command。

fence 只能拦截本地后续提交，无法撤回已送往外部的 HTTP 请求。最后一次检查与真正发送之间仍有时间窗口；租约变化后在途调用可能成功。此时保存迟到响应为待核对证据，不能让旧结果推动图。也不能声称数据库锁实现了外部系统的精确一次写入。

检查点同样需要隔离旧写入：实施时对 saver 的 checkpoint、pending writes 等全部写入口加统一守卫，在同一数据库事务中锁 run、校验 fence 后执行 saver 写；不能只在调用 saver 前检查再释放锁。该适配为 O，需验证固定版本 saver 是否可共享事务/连接，以及全部写入口能否受到同一守卫保护。若不兼容，应停止恢复验收并重新评审取舍，不能裸用 saver 最新值，也不预先引入第二套 namespace 复制系统或以未经证明的自制恢复替代 MVP 验收。

## 4. 图边界与工具账本

服务端构造 `thread_id = user:{uid}:conversation:{cid}:run:{rid}`，不接受客户端任意 thread_id；checkpoint_ns 由图版本/应用控制。恢复绑定数据库保存的 thread、namespace、checkpoint ID、graph_version，不能按客户端提供的“最新检查点”恢复。跨 Run 的对话上下文由业务服务选择历史消息、摘要和文件显式注入，不靠复用上一个 Run 的 thread 偷渡状态。活跃等待期间不热换图版本；不兼容升级需要迁移方案或停止恢复。

图节点建议：prepare_context → model_decide → normalize_calls → execute_one / interaction_barrier → record_observations → model_decide → validate_result。一次模型返回多个 tool_calls 时按稳定序号串行，逐个完成账本和观察，再推进游标；不使用默认并行工具节点执行潜在副作用。

每个逻辑动作先落 `tool_calls` 账本，唯一键 `(run_id, model_turn_id, tool_call_id)`；保存工具、连接/策略版本、规范化参数指纹 argsHash、分类、审批引用、attempt、fence、幂等键、结果引用和状态。状态统一为 planned、authorized、sending、succeeded、failed、unknown、rejected、cancelled。`sending` 必须先提交再发送，响应成功后单独提交结果，因此崩溃后 sending 一律按 unknown 考虑，即使可能尚未实际发出。已 succeeded 动作在图重进时读取已保存结果，不重调工具；结果大文件先安全落盘再提交引用，不能只记一个无法恢复的“完成”。

仅无副作用读取可以按白名单、次数和预算有限重试。声明为只读不等于真实只读；工具分类由服务端策略决定。外部写在发送前检查当前权限、连接是否停用、fence、取消和审批指纹及期限，并将批准与逻辑 action 绑定。参数、目标、认证主体或授权范围变化要重新审批。发送超时、连接断开或 worker 崩溃不触发自动重试；有可靠远端幂等键或查询接口时可查询同一 action 的结果，不能以“再发一次试试”核对。没有证明则 interrupted 并阻塞后续外部写，交由用户核对。批准记录不意味着远端已经执行，也不能重新授权另一动作。

## 5. Question/Approval 与图的可恢复归并

官方说明 interrupt 恢复会从所在节点起点重跑，并要求中断前副作用幂等；因此不能在混合执行节点中先外部写再提问。每次等待使用独立 barrier 节点且一次只调用一个 interrupt，不吞掉其控制异常。参见 [LangGraph Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)。

逻辑交互组由 `questions` 或 `approvals` 的一行承载，不另设 interaction_groups 物理表。QuestionID 就是整组 ID；`questions.question_json` 存最多 3 题，`question_responses` 存整组答案。Approval 组与普通 Question 组分开；每 Run 同时最多一组 pending，借 run 行锁和当前交互引用约束跨表唯一性。`resume_requests(group_kind, group_id)` 建唯一约束，group_kind 区分 question/approval。group_key 在上游已持久化图状态中确定，例如 run＋模型轮次＋call_id＋交互轮次；不能在重跑时重新随机生成。内容指纹不同但 key 相同立即报一致性错误。barrier 之前没有未完成的副作用；节点重跑只 upsert 交互意图和读取已保存快照，再调用 interrupt。答案校验失败创建下一交互轮次，不在一个节点里动态增加 interrupt 次序。

模型 human call 与其他工具混在一批时，先执行允许的先序工具并完成其配对消息，到 barrier 停住其余调用；恢复后为 human call 写入准确 tool_call_id 的 ToolMessage，再继续剩余串行游标。审批是待执行 action 的前置 barrier，批准前不生成虚假的“工具成功”结果。未完成整批配对前不再调用模型；未知外部写时直接停住，不能把未配对消息发给下一轮模型。拒绝或取消为对应 call 生成明确拒绝/取消观察，后续方案仍经策略检查。

不能将写 Question、保存 checkpoint、接受答案、Command(resume) 当作一个原子事务，采用可对账的四阶段协议：

1. **准备等待**：图先生成稳定 group_key 与意图；服务端幂等 upsert 为 preparing，完成第 6 节文件快照。barrier 调用 interrupt 后，由 saver 完成持久化。
2. **发布等待**：驱动器读取已落盘的中断 checkpoint，确认 group_key、payload 指纹及 interrupt 标识匹配，再以 fenced 事务更新 group 为 pending、关联 checkpoint_ref，并将 Run 改 waiting_*、追加事件。只有此时 UI 可以回答；业务行先写而 checkpoint 未写时只能显示“准备等待”，不能提前接受答案。
3. **接受答案**：API 锁 run 和 pending group，校验 owner、期限、结构及附件归属。幂等键 `(group_kind, group_id, client_submission_id)`；同键同内容返回原结果，同键异内容冲突；另一个键再次答已关闭组返回冲突。事务插入不可变答案/审批决定，将 group 改 answered，创建唯一 resume_request，Run 改 resuming 并写事件。API 不直接调用 graph，不以 HTTP 重试重跑图。
4. **恢复归并**：worker claim，确认等待 checkpoint 对应当前 group，以答案 ID/指纹构成 resume payload。barrier 恢复后仅校验并接纳答案；接着独立 `accept_answer` 节点将 applied_answer_id 写入图状态并持久化 checkpoint，后续工具节点才允许执行。驱动器看到该标记后把 resume_request 改 applied。即使业务标记尚未写而进程已死，也按 checkpoint 中 applied_answer_id 归并，不重复投递。

恢复前必须检查 checkpoint：仍为对应 interrupt 且无 applied 标记，才允许以原 answer 恢复；已含 applied 标记则补业务状态并从后续已确认位置处理；不匹配或无法确定则 interrupted。恢复执行已经到后续工具的情况，仍以动作账本判定，不盲目 invoke。worker 不能先把 request 标 applied 再发 Command，否则发送失败会丢答案。UI 的“已接收”与“已应用”分开展示。

重启对账器只做确定性的归并：checkpoint 有 interrupt 而业务组缺失时，按其受控意图补建；组 preparing 但无 interrupt 时，仅在已持久化上游意图、快照可用且无 unknown 动作时重入纯 barrier；否则标 interrupted。答案已保存但未恢复的保留原内容；已取消/过期的组关闭，不接受迟到答复。定期扫描与 worker 启动扫描使用相同幂等规则。

## 6. 等待快照与新实例恢复

建议进入等待立即快照并清理实例，不保留短等待特例。暂停前先确保没有执行中的 Sandbox 命令或依赖进程；无法确认静止则等待准备失败，不能发布“可恢复”。只保存允许的工作根目录常规文件，验证路径、类型、数量、单文件/总大小及用户容量；排除缓存、环境、凭据、socket、符号链接和可重新准备的依赖。全目录限额内保存，不能只保留产物却宣称全部工作已保存。

文件先下载到受控临时目录，流式限额、计算摘要，形成不可变 manifest（逻辑路径、file_id、size、hash、版本）；校验完成后原子移动，再事务发布 snapshot ready 和文件索引。未发布文件由孤儿回收器清理。超限、读不全或摘要失败应明确 failed/可解释错误，关闭问题并清理；不静默裁剪成“成功快照”。文件与数据库不可能跨介质原子提交，ready 标志是唯一可用性入口。

snapshot ready 且 interrupt checkpoint 确认后发布等待；将所有旧实例标 cleanup_pending 并马上尝试 destroy，finally 和后台扫描重试，TTL 兜底。清理失败仍释放模型/执行槽，保留会话活动锁，清理扫描占独立小限额；未确认旧实例停止时不创建新实例，以免双实例执行。用户可以先提交答复，但恢复调度等待清理确认或明确中断处理。

答复后创建新 SandboxLease，实例 ID 永不假装沿用；校验清单归属和摘要后上传，固定 Skill 版本和依赖声明重新准备。只恢复文件与上下文，不恢复内存、变量、后台进程或整机镜像。没有代码需求时不必为了恢复创建实例。原实例残留、快照缺失或依赖准备失败明确分类，不能回到模型自动修脚本掩盖基础设施错误。


成功终态也必须在清理实例前保存本轮 snapshot ready，并记录 sourceRun 和 generation；验证完成后先完成快照发布，再将 Run 标 succeeded。纯文本/只读MCP等无工作区变更的任务不强制创建 Sandbox，沿用conversation.working_snapshot_id，不生成覆盖旧工作区的空清单；“本轮无快照”仅为运行记录。明确清空工作区的真实变更除外。快照失败不能宣称本轮成果已完整保存。失败/取消时尽力保存并标记完整或不完整，保存失败不妨碍后续清理且须记录原因；对有 unknown 动作的运行仍维持会话阻塞。下一轮按权威指针、sourceRun、generation、ready 和完整性检查，不能把上一轮旧快照冒充本轮成果；历史快照仅可明确标注其来源后作为恢复输入。
## 7. Todo、事件、取消和预算

Todo 是计划而非结果。每次更新包含 expected_version；事务 compare-and-swap，冲突返回当前版本，由 Agent 读取后重新决策，不覆盖用户/其他更新。TodoChange 记录前后状态、原因、actor 和 action/artifact 证据；completed 不会自动将 Run 标 succeeded，实际验证失败仍保留。不要用一个没有边界的 JSON 字典代替所有核心对象。

持久事件按 Run 分配单调 seq：锁 run 增 next_event_seq，与对应业务变化同事务写 event，唯一 `(run_id, seq)`。SSE 的 ID 为 run_id:seq，验证归属后从 Last-Event-ID 补读；通知仅用于唤醒，丢通知不丢事件。允许至少一次投递，客户端按 seq 去重；保留期外返回需要重新取快照。模型 token 可作临时流，最终内容和关键状态必须落库，页面不能把临时流当最终事实。

取消 API 幂等记录 cancel_requested_at 并改 cancelling，关闭未答问题/未应用恢复请求。worker 检查标记后停止新模型和工具、尝试中止命令并清理；等待无执行时可直接完成取消。发送中的外部写不能承诺撤销；已知已完成则记录事实，未知则 interrupted 并保留取消意图与未知动作，不伪报 cancelled。取消与答复竞争由同一 run 行锁串行：先取消则拒绝答复，先接受答案后取消则禁止后续发送，已发送仍按账本处理。

配置分别限制 Run 总墙钟 TTL（含等待）、执行预算、单次工具超时、等待 TTL、排队 TTL、模型/工具/步骤/修复次数、文件与输出容量。等待不消耗模型调用与执行槽，仍消耗总 TTL/存储；恢复不重置累计预算。普通问题等待24h、总Run 48h或排队1h到期时，锁行关闭交互和请求，更新 expired 并发起清理。仅审批10min过期且总Run尚未到期时，将审批记为 denied（这是审批决定，不新增工具状态），保存拒绝观察并创建唯一恢复请求，使图恢复到拒绝分支；禁止发送原外部写，关联 tool_call 落 rejected。总Run期限优先于审批拒绝恢复；不得把审批过期误判为整个Run必然expired。任何终态若存在 unknown 写入，均保留醒目标识与会话阻塞。与主设计一致的建议初值（均 D，未获逐项批准）：global_active=2、user_active=1、queue_global=100、queue_user=10；active_run_timeout=15min，仅累计准备/执行/校验/恢复准备，不含等待；question_wait=24h、approval=10min；command=60s、model_call=90s、max_model_steps=30、repair_attempts=2；per_user_storage=1GiB、per_file=20MiB、run_inputs=100MiB，stdout/stderr 各保留64KiB并在采集端限制，截断显式显示。worker lease=60s、heartbeat=10s、poll=1s。总墙钟 Run TTL 建议48h、排队TTL建议1h，快照总容量建议100MiB、文件数建议1000；这些新值均为 D，不能用每次等待重新计时重置总TTL。

代码修复有配置开关和次数上限；关闭时服务端禁止因失败进入修复节点，也禁止把同一失败改名为“新尝试”通过普通文件编辑/执行工具绕过。工具层依据失败关联、脚本版本和执行阶段检查；普通用户主动新任务不等于 Agent 可自行重置计数。基础设施、权限、取消和 unknown 不进入代码修复。

## 8. 适配约束与验证清单

MCP 首版建议远程 HTTP，仅 noauth/staticBearer；个人连接归本人，共享连接独立授权。OAuth 保留认证类型与凭据提供接口的扩展点，不承诺本期实施。禁止普通用户提交本地 stdio 启动命令；地址、重定向、DNS/私网目标、工具权限和输出限额在服务端校验。Sandbox 不持有 MCP 长期凭据，跨环境文件通过明确上传/下载协议传递。

已有 V 仅证明 SDK 1.2.0、`opensandbox/code-interpreter:v1.1.0`、`use_server_proxy=False`、固定结构化 argv 安全引用成 command 字符串的小文件闭环；适配层以 `shlex.quote` 逐项引用（`shlex.join` 等价组织），不允许模型提交任意 shell。当前镜像原生 argv 请求曾 400，代理路径曾 502，Server 准确版本未知，API 描述 0.1.0 不能替代软件版本。实际退出码优先取 Execution.exit_code，不能只看 complete/error 内字段。证据为 `sandbox-check-20261028-185511/report.json` 及失败历史，临时脚本不直接变生产执行器。

SDK 内部 HTTP 日志、异常、URL、headers、请求/响应及追踪也须统一脱敏；不能只脱敏自己的 logger。禁止 key、Bearer、cookie、签名 URL 进入 SSE、模型、检查点和报告；默认关闭原始请求调试日志。

本稿仅做文档检查，未进行产品测试。实施验收必须覆盖：双 claim/过期 fence 写入及 saver 旧写拒绝；每个等待协议落盘窗口崩溃；重复/迟到答案、取消竞争；多 tool_calls 中断后的消息配对；外部写发送前/中/后崩溃；快照超限/缺文件/旧实例未停；SSE 重连去重；Todo 冲突；修复关闭绕过；跨用户所有 ID 越权。分别报告静态检查、模拟故障、实际服务集成和多用户验收，未运行不得标通过。


### 8.1 事务与核对的实施约束

业务事务统一按 conversation → run → interaction/action → event counter 的顺序锁行，避免取消、答复和 claim 使用相反顺序产生死锁；发生可重试数据库冲突时仅重试事务，不把网络调用放进事务重试闭包。发送授权事务需要同时验证批准仍有效、将批准绑定本 action、将 action 标 sending，然后提交；批准的“使用”表示唯一逻辑动作已取得发送资格，不代表已收到成功响应。sending 后的任何再次发送都受 unknown 规则约束。

恢复请求以 `(group_kind, group_id)` 唯一，一份完整答案覆盖该组所有必答项；附件先完成归属及内容校验后才可接受。普通对话消息不能被扫描器擅自解释为答案或审批。API 返回接收成功后即使浏览器断线，答案仍持久有效；worker 故障不会要求用户再点一次批准。若用户修改答案，须明确取消旧恢复意图并生成新交互版本，不能原地改写已应用的决定。

取消与过期后仍允许清理扫描更新清理结果及补充迟到外部证据，但不得把业务终态自动改回 running。unknown 无论伴随何种 Run 终态，均保留会话阻塞，不能忽略继续或用新Run绕过。仅开放专用只读核对入口或保存远端证据入口，由已获授权的人核对并判定；记录核对人、权限依据、远端证据、判定和后续允许动作。没有远端失败证据不能把 unknown 简单改 failed，未查明前不能发新业务写。业务执行错误和清理错误分别保留，不用最后一个异常覆盖最初原因。运行预算的预留、消耗和剩余额度需在同一事务约束，重试、恢复及图重进不会重复赠予预算；恢复读到旧账本结果也不应重复扣工具执行费用。
## 9. 官方语义核对与评审结论

核对日期：2026-10-28。官方页面会演进，不能直接把当前页面的新流式示例当成已锁定 LangGraph 1.x API。实施要锁实际小版本，并验证 interrupt、Command、checkpoint saver、pending writes 与 durable 配置。

- [LangGraph Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)：本稿采用整节点重进、稳定中断顺序和幂等前置准备，未把 interrupt 当数据库事务。
- [LangGraph Checkpointers](https://docs.langchain.com/oss/python/langgraph/checkpointers)：检查点属于 thread，支持保存图状态；它不是业务动作账本或外部系统的提交证明。
- [LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)：采用持久化 saver；内存 saver 不能满足重启要求。原 durable-execution 地址本次重定向到此页，因此不据旧地址夸大保证。

评审重点：数据库唯一约束控制会话并发；fence 和 checkpoint 写守卫控制旧 worker；动作账本将 unknown 与失败分开；四阶段等待协议解决答案与检查点不原子的窗口；立即快照、清理与替代实例明确恢复能力；审批和取消均不提供外部副作用的精确一次或回滚承诺。上述机制是待实施设计，不是已验证能力。



