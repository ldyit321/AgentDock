# AgentDock 执行与恢复设计

版本：v1.1 MongoDB 评审稿；日期：2026-10-28。本文替代 v1.0 中 PostgreSQL、SQLAlchemy、SQL 行锁和 saver 同事务守卫的方案；v1.0 原文保留，仅供历史追溯。MongoDB 替换来自用户最新指示，其余新增机制与配额为 **D（建议）**，驱动/saver 兼容性及故障行为为 **O（待验证）**。本次只编写设计，不安装、初始化、运行服务、读取或覆盖 `.env`。

## 1. 组件与可信边界

Windows Python 3.12、FastAPI、LangGraph 1.x 保持不变；业务数据库驱动使用官方 PyMongo Async API，不引入 Motor。建议 MongoDB 8.x 本地单节点 replica set，支持必要的多文档事务，但没有高可用：节点停止时整个数据库不可用。不能用 standalone 启动后声称具备事务，也不把单节点的 majority 写确认解释为副本容灾。具体驱动、MongoDB 与 `langgraph-checkpoint-mongodb` 兼容版本由主设计锁定并实测。

一个独立 Python worker 进程，异步有限并发；`runs` 集合作持久队列，不用 Redis/Celery。API 处理身份、上传、输入、答案、取消和查询；worker 驱动图、执行工具、归并等待和清理。调度控制文档仅是短事务的共享写入点，不引入通用分布式锁、自动选主或多 worker 接管框架。

LangGraph 使用兼容版 MongoDBSaver。业务 AsyncMongoClient 和 saver 的客户端不能因指向同一数据库就假定共享事务/session。官方 MongoDBSaver 已提供 aput/aput_writes 等异步包装，内部会包装 blocking put，不能说它没有异步方法。优先使用锁定兼容版本的现成 async wrapper，不主动另造包装层，也不引用旧 aio/Motor 方案；禁止直接在事件循环调用 blocking 方法。底层 executor 的线程并发、连接池、生命周期、取消后线程是否仍执行以及关闭顺序均待验证；API 名称带 async 不代表取消会停止底层同步工作。不直接修改 SDK saver 源码来拼接事务。图保存必要数据和引用；凭据、数据库连接、SDK/MCP 对象不进入 graph state/checkpoint。

业务层显式接收 owner、conversation、run、权限、预算和适配接口，不依赖 HTTP 对象或隐式全局当前用户。模型、Skill、上传、脚本及 MCP 结果均为不可信数据，不可改变权限。

## 2. 文档模型、索引与状态

字段以主稿snake_case命名为准。本文camelCase仅作语义缩写：ownerId→owner_user_id，runId→run_id，conversationId→conversation_id，state→status，epoch→lease_epoch，workerId→lease_owner，activeRunId→active_run_id，workingSnapshotId→working_snapshot_id；不能另建同义字段。待答组逻辑pending映射questions.status=open、approvals.status=pending，题组answered与审批approved/denied分开。

集合建议：users、conversations、messages、runs、scheduler_control、user_usage、capacity_reservations、run_events、tool_calls、questions、question_responses、approvals、resume_requests、todos/todo_changes、sandbox_leases、workspace_snapshots、files，以及 saver 自有集合。MongoDB 引用不提供关系外键；服务端每次验证归属、资源存在、启停状态和作用域，validator 约束类型、必填、枚举及有界结构，唯一索引约束逻辑标识。不得依赖 ObjectId/UUID 难猜实现授权，不把全部对话消息或日志嵌入一个无限增长文档。

Run 至少保存 ownerId、conversationId、state、epoch、version、workerId、leaseUntil、heartbeatAt、cancelRequestedAt、deadlineAt、waitDeadlineAt、budgetUsed、checkpointRef、graphVersion、nextEventSeq。epoch 为每次调度尝试单调递增的 fencing token；version 为文档乐观版本。核心变更使用 `findOneAndUpdate`，过滤 `_id+ownerId+预期state+epoch+version`，更新状态同时递增 version。匹配失败须重新读取解释冲突，不退化为只按 ID 覆盖。权限/取消等其他事实与 Run 同事务确认，需要时同时 CAS 被引用文档的版本。

| 状态 | 含义 |
|---|---|
| queued | 初次输入已持久化并获排队名额 |
| preparing | 已 claim，准备上下文/工作环境 |
| running / validating | 执行图/工具，或验证任务结果 |
| waiting_user / waiting_approval | 可恢复的持久交互；占会话锁、不占执行槽 |
| resuming | 答复已接受并排队，或正在恢复准备 |
| cancelling | 已请求取消，正在停止后续执行 |
| succeeded / failed / cancelled | 已知业务结果终态 |
| interrupted | 执行意外中止或外部副作用未知，停止自动执行 |
| expired | 排队、普通问题或总 Run 期限耗尽 |

conversations 保存 activeRunId、version、blockedUnknown 和 workingSnapshotId/generation。创建 Run 的短事务先检查 owner/会话状态与 blockedUnknown，再以 activeRunId 为空和预期 version CAS 占用；同一事务写入 Run、输入 Message、初始 Event、排队 reservation 和计数。所有新 Run 入口均执行该协议，不靠对 runs 的 count 查询判断会话空闲。等待保留 activeRunId；终态清空时匹配当前 runId/version，不能误清其他 Run。unknown 可伴随任何终态，blockedUnknown 独立保留，不能通过新 Run 绕过。

必要唯一索引包括：messages 的会话顺序、run_events(runId,seq)、tool_calls(runId,modelTurnId,toolCallId)、question_responses(questionId)（并校验submissionId幂等）、resume_requests(groupKind,groupId)、capacity_reservations(runId,epoch)、快照(sourceRun,generation)。questions 的 groupKey 与 approvals 的 groupKey 分别唯一；跨集合“一组 pending”由 Run.pendingInteraction CAS 及事务保证。查询队列按 state/eligibleAt/createdAt 建普通索引；索引不替代权限过滤。不存在的可选字段需明确 null/缺省语义，避免稀疏或部分索引误约束合法数据。

## 3. 队列容量、claim 与槽位释放

不能将 SQL 的 SKIP LOCKED 换成查找后更新，也不能只在事务中 count 活动 Run；并发事务可能读到同一旧容量。建议 scheduler_control 使用一个固定控制文档，包含 revision、queueCount、activeCount 和全局上限；用户计数独立文档。所有改变队列/执行容量的路径，在同一短事务首先写 control.revision，再更新用户计数、Run 和 reservation；该共同写入点使竞争事务产生写冲突并重试，避免只读谓词造成的容量幻读。该单机 MVP 写入热点是有意取舍，不扩展为分布式锁。

事务统一使用兼容的 primary、snapshot read concern 与 majority write concern，边界和配置在实施验证；数据库时间表达式用于期限/租约计算，测试时注入受控时钟。事务回调可能重跑，回调内只有数据库操作，绝不发送模型、MCP、Sandbox 请求或进行文件发布。提交结果未知时按稳定 operationId 查已提交记录后归并，不能无条件再次加减计数。

首次入队创建 `(runId,epoch=1)` reservation，phase=queued；增加全局/用户 queueCount。调度器可在事务外读取候选 ID，事务内再次验证；写 control 后检查容量，CAS Run 从 queued 或可 claim 的 resuming 转执行，写 worker/lease，reservation queued→active，queueCount 减一、activeCount 加一。候选被其他路径取消或到期则匹配失败并放弃；事务成功才开始外部准备。恢复入队递增 epoch、创建下一份 queued reservation，旧 reservation 不复用。候选必须存在同epoch的queued reservation；resuming状态本身不代表已经获得队列资格。

进入等待、失败、成功、取消或中断时，容量释放使用同一事务：CAS reservation 的 `(runId,epoch,phase=active)`→released，CAS Run 状态并清除 lease，activeCount 减一，同时写事件。排队取消则 queued→released、仅减 queueCount。重复执行只会有一个 CAS 成功，不能重复减数；释放请求携带旧 epoch 时不可释放新恢复尝试的槽。执行计数释放以已提交 reservation 为准，不以 worker 内存信号量或超时推测为准。恢复答复接纳时要重新申请 queue 名额；队列满返回可重试冲突并保留原 pending 问题，不能先承诺已接受再丢失调度。清理有独立小并发，不占模型执行额度。

扫描器可对计数做诊断性核对；发现计数与 reservation 不一致先停止新 claim 并报告，不能在并发运行中盲目重算覆盖。修复需在唯一 worker 停止调度的维护边界执行，保存差异与操作记录。

## 4. 单 worker、租约与检查点边界

worker 启动持有 Windows OS 级生命周期互斥锁，第二进程取锁失败即退出。PID 文件只作诊断；不能因为 lease 超时而夺取仍存活旧进程的运行权。不做自动 failover。重启前确认旧 worker 已退出；若 executor 中同步图调用仍未停止，不得在同进程释放图执行占用并启动同 Run 的新调用。线程取消不等于底层调用停止；无法收敛则停止 worker，由明确的进程终止/重启流程处理。

心跳以 runId/workerId/epoch/允许 state 条件续租并递增 version；失联、CAS 失败或租约到期停止后续工具。发送前及结果落库前都检查当前 epoch、取消、预算、权限与审批。旧 epoch 的结果不能推进 Run、权威快照或 checkpoint 指针；迟到回包可保存到隔离的核对证据，不能冒作当前执行结果。fence 无法撤销已发出的 HTTP 请求，检查到发送仍有窗口，因此不能承诺外部精确一次。

租约过期的执行 Run 标 interrupted，而非自动 queued；在释放槽位、清理实例前先确保旧本地执行已停止。旧进程仍活着时只能停止调度/请求取消，不让另一执行接管。新 worker 获得 OS 锁后，先做启动核对：未 claim 的 queued 可继续；waiting 可展示；执行中且失去进程的 Run 中断；不从任意 checkpoint 自动重跑工具。

thread_id 服务端统一构造 `user:{uid}:conversation:{cid}:run:{rid}`；namespace、checkpointId、graphVersion 及关联 epoch 保存在权威 checkpointRef 中。客户端不能提交任意 thread 或 checkpoint。驱动器只有在此次调用正常返回、saver 明确完成持久化、按精确 ID 读回内容并验证所属 epoch/等待意图后，才以 Run CAS 发布指针。不得裸用 saver latest、把任意新发现检查点升为权威，或允许旧 epoch 覆盖指针。恢复起点可以是上一epoch已经确认的等待指针：先校验它与当前resume_request的明确关联，再由本次epoch继续；不能把“新epoch不同”错误地当作所有历史等待指针失效，也不能接受未曾确认的旧epoch结果。

业务写与 saver 写不在默认同事务。进程可能在 saver 成功后、指针发布前死亡；这时旧指针不证明新 checkpoint 未存在。保守处理为 interrupted/待核对，不宣称透明恢复；不得猜测最新记录并自动提升。固定版本 MongoDBSaver 的 async/sync接口、显式 checkpointId 读取/恢复、pending writes、提交可见性、session 兼容及 executor 停止行为均 O。默认不改 saver 源码、不预建第二套 checkpoint 复制系统；核验不满足则停止相关恢复验收并评审能力边界。单 worker 无接管和权威指针降低竞态，不能替代这项核验。

## 5. 工具账本与副作用

工具调用物理集合为 tool_calls；状态固定 planned/authorized/sending/succeeded/failed/unknown/rejected/cancelled。唯一逻辑调用记录工具、连接及策略版本、目标、规范化 argsHash、审批引用、attempt、epoch、远端幂等键、结果引用和时间。每次状态转换以预期状态/版本 CAS，并在事务中验证 Run epoch，写事件；重进读到 succeeded 时回放已存结果而非再次调用。

发送授权短事务校验当前身份、连接启用、策略、预算、取消标记及审批期限/参数指纹，将 approved 决定绑定唯一逻辑 action，再置 sending 后提交；随后才发请求。批准不是远端成功，也不是可无限重试的授权。响应后再以 CAS 保存结果，窗口无法原子消除：sending 后崩溃、超时、断线一律按 unknown 处理，可能尚未发送也不能猜。

仅白名单无副作用读取可有界重试；“工具说只读”不自动成为可信策略。外部写不自动重放；远端可靠幂等键和查询能力可用于专用只读核对，不以再次发送试探。unknown 设置会话阻塞并中断后续业务，无论 Run 是 cancelled、expired 还是其他终态均保留。只允许授权核对或上传远端证据，由获准人员记录判定、证据和后续动作；不能提供“忽略继续”或另开 Run 绕过。已完成写操作不因取消/销毁 Sandbox 自动回滚。

多 tool_calls 按稳定序号串行。human/approval 是独立 barrier，前面完成的工具结果准确配对，后续调用游标持久化；恢复后先补对应 tool_call_id 的 ToolMessage，再继续剩余调用。整批未配对前不向模型发下一轮；未知写直接停止。拒绝产生明确 rejected 观察，不能伪造成功。模型修复开关由服务端执行：关闭时拒绝修复节点以及将同一失败改名后编辑/重跑的绕过路径；基础设施、权限、取消、unknown 不交给模型反复改业务代码。

## 6. 整组问答、审批与恢复归并

QuestionID 是整组 ID，questions.question_json 最多 3 题，question_responses 保存整组答案；审批由 approvals 独立承载。无 interaction_groups 物理集合。Run.pendingInteraction 保存 groupKind/groupId，CAS 空→当前组防止并存两组 pending；同事务写交互意图。groupKey 来源于已保存的模型轮次、callId、交互轮次，重跑稳定且内容指纹一致；不能重跑随机生成另一组。

LangGraph interrupt 会重进整个节点，故 barrier 中只有幂等 upsert/读取和一次 interrupt，不含外部写，不吞掉控制异常，不按运行时条件改变中断次数。正常回答与批准分别验证，普通消息“好的”不能隐式批准。四阶段协议如下：

1. **准备**：持久化稳定交互意图，组状态 preparing；有工作区则先形成 ready 快照，barrier 调用 interrupt，等待 saver 完成。
2. **发布**：从精确 checkpointId 验证 interrupt payload、groupKey 和指纹，CAS 发布 checkpointRef；业务事务将组变 pending、Run 变 waiting_*、写事件并恰好释放本 epoch reservation。只有此时 UI 允许答复；“已写问题”不等于“可恢复等待”。
3. **接收**：API 验证 owner、Run/组版本、pendingInteraction、期限、答案结构和附件。submissionId 同键同内容回原结果、同键异内容冲突；不同键再次提交已关闭组冲突。事务写整组不可变答案/决定、组 answered、唯一 resume_requests(groupKind,groupId)，申请恢复排队 reservation、Run resuming、事件。取消/过期竞争使用相同 Run CAS，失败方重新判断；API 不调用 graph。
4. **应用**：worker claim 后重新验证权威等待指针及原答案ID/指纹，以 Command(resume) 继续。独立 accept_answer 节点将 appliedAnswerId 写入图并完成持久化，再发布经确认的新指针；图需在此设置明确的持久化/发布门禁，下一工具节点先确认该指针与答案已接纳，不能仅按节点代码返回顺序假定驱动器已发布。门禁与固定版本图执行 API 的适配需验证，未通过则停止此恢复路径验收；后续工具节点才允许运行。业务 resume_request.applied 落库可晚于 checkpoint，归并以精确已确认 checkpoint 的标记为证，不能在发送 Command 前先标 applied。

重启时：已确认等待指针仍对应 interrupt且未应用答案，才可投递原答案；已确认指针有 appliedAnswerId 则补业务状态；指针未确认、内容冲突或图可能已越过工具但无法证明则 interrupted。checkpoint/业务不原子的所有缺口都须故障注入，不承诺所有断点无感恢复。答案已接收但未应用在 UI 分开显示；未答不是同意，迟到答案不启动终态 Run。校验不通过创建新交互轮次，不在单节点循环增加 interrupt。

审批10min到期且总Run未到期时，事务记 denied（审批决定）、tool_call rejected、保存拒绝观察并创建唯一恢复请求，图只走拒绝分支不发原外部写；队列满时保留唯一恢复意图并标admissionState=awaiting_capacity，Run为resuming但不算已入队；扫描器在容量可用时事务申请reservation并标queued，禁止无名额执行。该系统生成意图不会重复创建，不占额外执行槽，受总Run期限约束。用户答案队列满的行为仍按前述可重试拒绝处理。普通问题24h、排队1h或总Run48h到期则 expired；总期限优先。业务扫描器 CAS 执行过期并写事件/清理，MongoDB TTL 索引只用于允许物理删除的保留期数据，不能替代业务 expiry、计数释放或审批拒绝。

## 7. 文件、快照与产物发布

本地 MVP 用受控目录保存不可变 fileKey 对应内容，MongoDB 存归属/摘要/索引；不默认用 GridFS。工作目录只在 Sandbox 中由一个 Run 写，Windows 不执行上传Skill/模型代码，也不允许两个 Run 并发改同一可变文件。路径经工作区服务解析，不接受客户端/模型宿主绝对路径。未来多节点可换 S3 兼容对象存储：业务保存稳定 fileKey 和逻辑路径，不保存宿主路径；本稿不承诺已部署对象存储。

上传、工作副本、产物和快照先写临时对象，流式校验大小、hash、路径和类型，再以不覆盖的不可变 key 原子发布。本地使用同卷原子移动，对象存储使用新唯一 key 完成上传后核验；最后以数据库 version CAS/事务提交元数据和引用。发布冲突不覆盖旧文件；未被数据库引用的对象按安全宽限期回收，不能扫描时删除仍在提交窗口内的对象。文件发布与数据库没有跨介质原子事务，ready 是唯一可用性入口。

等待立即快照并清理，不保留短等待实例特例。先确认命令已静止，将允许工作根内常规文件完整保存，限制100MiB/1000files、单文件20MiB及用户容量；排除缓存、依赖环境、凭据、socket和链接。manifest 不可变，记录 fileKey/hash/size/逻辑路径、sourceRun、generation、完整性；超限/缺失/摘要不符明确失败并关闭问题、尝试清理，不静默丢文件仍称可恢复。snapshot ready与确认的 interrupt一起满足才能发布等待。

成功且有工作区变更时，清理前必须形成本轮 snapshot ready，再通过 conversation version CAS 发布 workingSnapshotId/generation，并标 succeeded；并发或旧 epoch不可更新权威指针。纯文本/只MCP且无文件变更不创建Sandbox、不用空清单覆盖旧工作区，本轮记录“无新快照”；明确清空工作区除外。失败/取消尽力保存并标完整/不完整，不冒用旧快照宣称本轮成果。后续恢复验证指针、sourceRun、generation、归属及完整性。

等待释放执行槽但保留 activeRunId；旧Sandbox立即 cleanup_pending并 destroy，失败可显示问题/接收答复，恢复前必须确认旧实例不可执行，否则阻塞。恢复按需新建SandboxLease，验证清单后上传、按固定Skill版本重新准备依赖；只恢复文件和上下文，不恢复变量/进程/内存。无代码任务不强制实例。每lease独立记录清理状态 not_created/active/cleanup_pending/cleaned/cleanup_failed/cleanup_unverified；finally、后台重试和TTL兜底，成功不掩盖清理失败。

## 8. Todo、事件、取消与配额

Todo 使用 version CAS，同事务存 TodoChange 的原因、actor和证据；冲突读取当前值后重新决策，不覆盖他人更新。completed 是计划记录，不决定 Run 成功。SSE 每次关键状态事务中对 Run.nextEventSeq 执行 `$inc` 并插入唯一(runId,seq)事件；事务失败不发布，重试按operationId去重。SSE id=`run_id:seq`，按身份授权和 Last-Event-ID补读，至少一次投递、客户端去重；临时token流不代替最终消息/状态。事件保留期外提示重新取快照。

取消事务 CAS 记录 cancelRequestedAt、改 cancelling并关闭待答组/未应用请求；worker禁止后续新动作，尝试终止代码和清理。排队或等待可直接完成取消并恰好释放对应reservation；执行中先确认本地执行退出再释放槽，不能把未停止的同步线程当已取消。外部写在途未知则 interrupted并保留取消意图和会话阻塞，不伪报已撤销。迟到证据和清理状态允许更新，不自动将终态改回running。

建议配额（均D）：global_active=2、user_active=1、queue_global=100、queue_user=10；活跃累计15min不含等待但含准备/恢复/执行/校验，总Run48h、排队1h、question24h、approval10min；command60s、model90s、max_model_steps30、repair_attempts2；用户存储1GiB、单文件20MiB、输入100MiB、快照100MiB/1000files；stdout/stderr各保留64KiB且采集阶段有界，截断可见。lease60s、heartbeat10s、poll1s。恢复不重置预算；模型/工具开销先预留后结算，重进复用结果不重复收费，也不赠送新预算。

MCP保留HTTP noauth/staticBearer，并依实际套件评估将标准OAuth AuthProvider纳入通用宿主设计；未实现／未核验认证和视图能力时相应在线分支标受阻；连接与凭据服务端按owner和工具策略保护，不进Sandbox或graph。Sandbox当前V仅SDK1.2.0＋镜像v1.1.0＋direct/use_server_proxy=false＋固定argv经shlex.quote/join成command的小文件闭环；proxy曾502、原生argv曾400，API版本0.1.0不代表Server版本。优先Execution.exit_code；不复用临时脚本冒充生产执行器。SDK内部HTTP日志、异常、headers、签名URL也需脱敏。


### 8.1 一致性细节与迁移边界

MongoDB validator 不能跨集合证明 owner 一致，唯一索引也不能证明引用对象未被停用。文件、Skill版本、MCP连接、Question、审批、Todo与checkpoint关联在服务端统一验证。停用/删除与发送竞争时，发送授权事务必须读并写受控版本标记，或按明确策略使撤销推进连接版本并令授权CAS失效；只在较早模型决策时查一次权限不够。即使如此，已发送请求仍无法追回，需保留撤销时刻与发送证据。

业务事务不共享同一个session并发执行数据库操作；每个事务按固定顺序串行读写控制文档、用户计数、会话、Run、reservation、交互/工具、事件。API取消、扫描过期和worker状态更新都遵守相同协议；某个入口遗漏controlDoc或reservation会破坏计数保证。事务重试仅限驱动支持的瞬时冲突，次数和总耗时有界，持续失败返回基础设施错误，不交给模型修改代码。稳定operationId由业务请求/动作生成，不在回调内每次随机生成。

恢复排队时若审批已批准但尚未发送就过期，仍走denied/rejected分支，不因为答案曾成功接收而延长有效期。已进入sending后发现期限过去不改变远端事实，按结果/unknown核对，不能直接标rejected掩盖已发送。Question附件的fileKey必须在接收整组答案之前完成发布，答案只引用有效且本人所属对象；重复答复不重复上传引用、创建恢复意图或收费。

控制文档只保存有限计数和配置，不嵌入所有用户、任务或等待列表；reservation与事件独立集合避免单文档膨胀。快照manifest在上限内也应限制BSON文档大小，必要时按固定片段存子记录，以一个ready元数据引用完整清单；不把大日志、二进制或整个Skill包塞进Run。所有原始工具输入/输出用受控内容引用保留，图内只放有界摘要，不能因为上下文压缩而删除审计账本。

GraphState.messages应有界，并在工具调用/观察完整配对的边界进行安全压缩；不能裁掉尚未配对的assistant/tool消息，或把摘要当作权限/批准事实。预算、原始文件、动作状态和答案归并仍由业务数据决定。上下文选择与摘要详见将由主稿配套交付的《AgentDock-上下文与记忆设计-v1.1》，本专题不另建第二套记忆规则。

未来多节点部署需要重新评审运行互斥、对象存储、队列容量热点和checkpoint并发，不能删除OS锁就称支持横向扩展。当前单机设计有意不实现自动故障接管；从本地文件换为S3适配不改变归属、不可变key、ready发布和CAS引用原则，也不自动提供数据库与对象的跨介质事务。
## 9. 验证与官方依据

本次仅文档审阅。实施必须验证：单文档CAS竞争、controlDoc容量冲突与事务回调重复；queue/active reservation释放竞态及旧epoch；OS锁/live线程不接管；saver同步桥接/精确指针/提交窗口；问答每个落盘窗口崩溃；审批过期拒绝恢复；外部写unknown跨终态阻塞；文件发布孤儿窗口及快照超限；SSE重复/断线、Todo冲突、跨用户全资源越权。区分模拟、实际服务集成和多用户验收，未运行不称通过。

官方核对仅支持基础语义，不代表本项目已验证兼容：

- [MongoDB 单文档原子性与事务](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/)：预期值过滤可用于CAS，跨文档原子性需要事务；本稿以共享控制文档写冲突额外约束容量。
- [MongoDB 事务运行约束](https://www.mongodb.com/docs/manual/core/transactions-production-consideration/)：需要replica set而非standalone，事务应短小；单节点用于本地开发不提供HA，部署可用性另行验收。
- [PyMongo 事务](https://www.mongodb.com/docs/languages/python/pymongo-driver/current/crud/transactions/)：session/事务由业务客户端明确传递，回调可能重试，不能把外部副作用放进回调。
- [MongoDBSaver官方API](https://langchain-mongodb.readthedocs.io/en/latest/langgraph_checkpoint_mongodb/saver/langgraph.checkpoint.mongodb.saver.MongoDBSaver.html)：优先现成异步包装，仍需验证底层阻塞调用、线程及取消行为。
- [LangGraph Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)：整节点重进，采用独立barrier、稳定标识和幂等前置准备。

v1.1的保证边界是：数据库业务状态可用CAS/必要事务防重复，外部副作用依然可能未知，checkpoint跨客户端依然非原子。兼容核验不通过就收窄验收并重新评审，不以更换数据库为由暗中降低恢复、安全和多用户要求。



## 11. 实际Skill套件补充

见[通用宿主能力设计](AgentDock-通用Skill宿主能力设计-v1.1.md)和[静态适配结论](AgentDock-Skill适配评估与结论.md)。OAuth接续由正式回调确认后生成唯一恢复意图；不能把问题答复当Token。范围Grant只减少已明确覆盖动作的重复审批，不绕过逐动作权限／epoch／unknown检查。外部状态与领域JSON通过DomainStateRef关联，Sandbox恢复不等于远端对象恢复；工具内部写重试也需要协议兼容门槛。长期资料发布使用独立授权、staging和Mongo版本CAS，不把脚本OS锁当对象存储发布锁。

unknown必须同事务登记operation_blocks，不只标当前conversation；每次托管外部写前按owner及共享connection检查，换对话／连接／复制文件不放行。MVP不支持未完成Run跨对话恢复；受控资料导入是新任务输入并保留平台来源，不继承旧Grant／OAuth。重新上传的domain_task_id不能创建或清除服务端来源和核对记录。
