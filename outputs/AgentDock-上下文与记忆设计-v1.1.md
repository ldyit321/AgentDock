# AgentDock 上下文与记忆设计

版本：v1.1 评审稿；日期：2026-10-28（Asia/Shanghai）。

依据：[需求基线](AgentDock-需求与功能基线-v1.0.md) FR08／FR09／FR11～FR14、[主技术设计 v1.1](AgentDock-技术设计-v1.1.md)、[执行与恢复设计](AgentDock-执行与恢复设计-v1.1.md)。本文替代 v1.0 主设计中“最近12条消息”的简化建议，仅设计，不代表已安装、开发或验证。

**方向：原始记录保留、工作上下文有界、摘要可溯源、运行状态以业务账本为准。** 用户新增长会话、压缩与记忆要求为 U；本文阈值、字段、预算和长期记忆范围为 D，兼容性与效果待 V。Vue 3＋JavaScript＋Element Plus、MongoDB 为用户指定；Python／FastAPI／LangGraph沿用主设计。默认用可解释的业务上下文装配，不把所有历史塞入 LangGraph messages，也不为上下文管理额外引入 Deep Agents SDK、向量数据库或多 Agent。

## 1. 借鉴的现成机制与选择

| 实践及官方依据 | 可借鉴机制 | AgentDock 的取舍 |
|---|---|---|
| [Deep Agents Context engineering](https://docs.langchain.com/oss/python/deepagents/context-engineering)，Compression／Offloading／Summarization | 大工具内容外置后保留预览与引用；接近限制才摘要；原记录仍可取回 | 采用“先限量与外置、再摘要”的顺序；存储走本项目权限接口，不向模型暴露宿主机路径；不照搬其默认阈值 |
| [LangChain Context Management for Deep Agents](https://www.langchain.com/blog/context-management-for-deepagents)，Targeted evals | 用强制压缩测试目标延续与旧事实找回能力 | 作为专项验收方法，不能用摘要生成成功替代任务正确性 |
| [Anthropic Effective context engineering](https://www.anthropic.com/research/effective-context-engineering-for-ai-agents)，Compaction／Structured note-taking | 摘要保留决策、未解决事项；结构化笔记承接长任务 | 使用任务简报和证据引用；不自动建立跨用户或跨对话知识库，不照搬其文件数量经验值 |
| [Claude Code How it works](https://code.claude.com/docs/en/how-claude-code-works)，When context fills up | 自动摘要；持久规则单独装载；重复压缩无收益时停止 | Harness规则每次由服务端装配；摘要无收益则有界失败，避免不断花费模型调用 |
| [OpenAI Compaction](https://developers.openai.com/api/docs/guides/compaction)，Server-side／Standalone compact endpoint | 提供原生压缩，返回不透明状态供后续调用 | 可选 provider 能力，非跨模型默认；业务原文、审批与状态仍由本项目保存 |

以上资料于本次设计实际搜索并打开。参考的是机制而非品牌绑定；外部文档可能继续更新，实施锁版本后重新核验。Deep Agents 是现成 Harness 实践，Claude Code 是另一种现成 Agent 实践；OpenAI项是提供商API能力，三者不混称为同一层框架。摘要会损失细节，不能承诺无限会话或无损记忆。

备选方案：只截近期消息成本低但容易忘记目标；完全依赖原生压缩受供应商协议约束；**推荐显式业务摘要＋受控按需取回**，允许特定模型后续启用原生压缩。MVP仍是单Agent，不因参考文章讨论子Agent而扩大范围。

## 2. 四类状态及真实性边界

| 类型 | 保存内容 | 生命周期与权威 |
|---|---|---|
| 原始历史／工具账本 | 用户消息、可展示的助手回复、工具调用与结果引用、事件 | 同对话持久保存；压缩不删除或覆盖原文；依保留／删除策略清理，不承诺永久保存 |
| 短期工作上下文 | 本次模型实际输入：规则、简报、摘要、近期完整组、片段 | 每次调用重新装配，有 token 上限；是投影，不是全部历史 |
| LangGraph checkpoint | 节点、待恢复位置、消息小窗口、上下文版本、业务引用 | 一个 Run 一个 graph thread；仅恢复该 Run，不提供跨Run的聊天记忆 |
| 长期记忆 | 跨对话偏好、经验或知识 | MVP默认关闭自动写入与自动检索；将来需用户可查看、纠正、撤销及隔离设计，另行评审 |

同一 conversation 跨 Run 的任务简报和摘要属于对话接续，不等同跨对话长期记忆。文件工作快照保存真实工作文件；context_snapshot保存模型输入组成；graph checkpoint保存执行位置，三者不能替换。摘要中的“已完成”不能更新Todo，摘要中的“已批准”不能消费审批，摘要中的“已上传”不能发布产物。当前工具账本、Question／Approval、Todo、文件索引和权限政策由业务服务读取确认。

原始模型输入保留来源及不可信标记；服务端凭据在装配前就不可进入消息。意外秘密的检测、隔离与按数据政策处理是独立流程，禁止把秘密写进摘要、模型输入、嵌入索引或公开日志。原文不可变指业务正常追加写，不阻止依法删除或用户删除；删除后相应摘要、快照、备份访问按主设计撤销。

## 3. 输入预算与触发条件

每个模型配置显式记录 `model_context_window`、可选 `max_input_tokens`、`max_output_tokens`、计数方式及配置revision。未知容量不能猜成大窗口，管理员配置保守上限后才能启用。定义：

```text
C = 模型总上下文窗口
O = 本次输出预留（含提供商计入输出预算的推理token）
M = 安全余量
B = min(模型独立输入上限（若有）, C - O - M)
I = 本次完整输入估算：规则 + tools schemas + 简报 + 摘要 + 近期组 + 文件/结果片段 + 协议开销
u = I / B
```

`O` 必须同时受模型输出上限和Run剩余预算限制；不能为多塞历史默默缩到无法回答。M建议起步取C的5%，实际由适配验收调整；图像／音频／附件计费换算和协议隐藏开销也要考虑。动态MCP工具schemas在装配时真实计数，预留下一次有界工具观察的空间；工具目录只加载当前获准且相关的工具，不通过裁断JSON schema来缩小。工具定义本身过大则缩小获准集合或报配置错误。

建议 soft=70%、hard=85%，**均相对扣除输出预留与安全余量后的B**，不是C。soft触发外置、去重和摘要候选评估；hard是常规装配门限，超过则不得发送普通推理请求，先降载。摘要后目标回到B的55%以内；若核心材料本身超过目标，但仍低于hard，可标记受限继续。70/85及55均为D，需用实际模型和中英文任务测量，不宣称官方统一推荐。

例如C=128000、O=8192、M=6400且没有更低的独立输入上限，则B=113408；soft约79386、hard约96397。未来工具输出空间位于hard剩余的B中，不再当作可随意填满的历史空间。新工具观察到来后必须再次完整计数，不能只按上一次百分比判断。

计数优先用提供商请求计数接口／可靠匹配的tokenizer；实际返回的usage用于核对并校准后续估算，缓存命中token依旧占上下文，不因价格优惠扣除。未知tokenizer采用保守估计并乘估算系数，起步1.3，记录估算方法及置信状态；中英文、代码、JSON分布需分别压测，不把固定字符/token比当准确值。缺usage显示未知，不能填0。

遇明确上下文过长错误，仅允许一次确定性降载重试：减少可选片段、缩小近期完整组、使用有效摘要；产生新snapshot并关联原尝试，输出预留不变。若仍过长或核心材料放不下，返回 `CONTEXT_BUDGET_EXCEEDED`，保留历史并提示选择更小范围／更大合适模型；不能无限“摘要→重试”，也不能因此重放工具。

## 4. 每次调用如何装配

1. 校验user／conversation／Run归属、conversation版本和Worker fencing token；读取已提交的历史水位与有效摘要指针。
2. 加载服务端规则、当前政策revision、工具schema集合；这些来自当前配置，不能从旧摘要恢复权限。
3. 加载结构化任务简报、被选Skill的固定版本、最新文件与工作快照索引。先传有界目录和摘要，按需读取正文，不重复复制整个Skill包。
4. 选取覆盖到 `covers_until_seq` 的有效摘要，并加入其后近期消息完整组；摘要覆盖原文默认不再次整段加入，按明确问题引用原文片段时记录去重原因。
5. 填充本轮用户输入、当前待办、必要工具观察和按需文件片段；做协议配对、权限、版本及token校验，必要时进入压缩。
6. 固化不可变context_snapshot清单后才能请求模型；响应以该snapshot关联usage、错误及后续事件。归属或版本变化则重新装配，禁止“先调用后补记”。

结构化简报至少包含：`task_brief`（当前目标、范围、交付要求）、`facts`（事实及证据）、`decisions`（用户确认与建议分开）、`pending_questions`、`approval_refs`（只读状态引用）、`chosen_skill_versions`、`file_refs`、`todo_refs`、`unknown_actions`及`next_steps`。未回答问题、未决审批、外部副作用unknown、最新用户纠正、失败证据、输入文件版本优先保留。可选历史分析和重复日志先退出上下文。

完整组定义为用户语义轮次及其中assistant tool_call与所有对应tool_result；同一assistant并行发出多个调用时，要保留全部匹配结果或整组摘要，不留悬空call_id。不截断JSON参数、工具结果消息结构或安全说明。压缩只发生在协议完整边界；待执行调用及待返回结果不作为可压缩的已完成组。大型参数正文可在历史投影中变成有出处的概述，但不得据此重新构造或执行原调用。

## 5. 工具结果、文件与按需回读

工具采集阶段就限制bytes、行数、时间和产物大小，不能收完无限响应才裁剪。建议单工具传给模型的预览不超过2000 tokens，并受当前输入余量约束；单次read_more也有同级上限，Run累计读取计入工具及token预算。超限注明截断、原因、是否保全完整内容，禁止把截断结果表述为完整结果。

原结果在容量允许时保存为文件记录，模型只获得 `result_id/file_id`、内容摘要、大小、片段与游标。`read_more(result_id, cursor, limit)`、按来源搜索和指定消息回读由服务端鉴权，并返回有限内容；游标绑定资源版本，不能接受任意宿主路径。未保存完整响应时只允许读取实际捕获部分，不伪造“全文可查”。二进制不直接入prompt，多模态走适配预算。

文件内容使用StorageProvider：本地阶段LocalStorageProvider，上线S3兼容对象存储；不可变object key＋内容hash，数据库CAS发布当前引用，摘要只存逻辑file_id／version／片段位置。模型不获得存储密钥、长期直链或D盘路径。引用文件被删除、失去权限或hash变化时不能读取旧名称下的新内容，应返回来源不可用或显式加载新版本。对象先成功落盘／上传且校验，再发布DB引用；失败留下的孤儿对象按主设计回收。

### 5.1 多模态上下文（新增用户要求）

模型支持多模态、按OpenAI兼容协议接入并理解图片／文档为U。默认Chat Completions多模态messages，Responses为可选能力适配；“兼容OpenAI”不能推断支持Responses、原生PDF或compaction。具体协议与解析流程见主设计链接的多模态专题。

图片、扫描文档和含图表的PDF页应构造成真实image parts，不能只传文件名、文字摘要，或把所有媒介自动OCR后丢弃视觉。原生input_file仅在model_capabilities明确支持时使用；否则提取文本，并把需视觉理解的扫描／图表页渲染为image parts。页数、分辨率、图片数量、detail模式及估算tokens由能力配置与Run预算共同限制；超预算时分批选页并明确覆盖范围，不能声称看完未加载页面。

原图与原始文档通过Mongo文件索引关联StorageProvider的不可变对象；解析、OCR、页图等派生结果记录source_file_id、source_version、page范围及parser_revision，归属当前用户和对话。checkpoint与摘要只放结构化media refs，不持base64、可复用签名URL或存储秘密；每次invoke重新鉴权，读取所选字节并由provider适配器构造parts，传输数据不回写巨大图状态。context_snapshot记录实际选中的文件版本、页码、图像hash、detail及计数方法，便于解释模型真正看到哪些页。

摘要对视觉结论标注来源file／page refs、可信范围和不确定性；有后续视觉问题时按权限重新读取原图／局部页，不能依赖文字摘要回答精细图像问题。媒体估算未知时采用保守页／图上限与系数，错误降载仍受一次重试限制；文件撤权或删除后旧摘要引用不能绕过访问检查。

## 6. 摘要生成、校验、发布与回退

按token压力、近期完整组边界、重要里程碑决定摘要范围，不按固定12条消息计数。每次摘要输入是前一有效摘要＋新增连续已提交原文区间＋权威状态投影，不能只反复压缩摘要而永不核对原文。摘要模型调用也要先预算；待压缩区间太大时按完整组分段推进，每段均计入摘要总预算，不能一次发送已经过长的输入。

候选摘要写成固定结构，事实／决定／限制／未决事项每项携带message seq、tool_call_id或file版本证据；不确定项标记 `uncertain`，矛盾项保留双方来源。保留“用户决定／建议／实测／待定”的差别，不把猜测升级为事实。最新用户纠正覆盖旧工作假设，但原证据仍可回查。

发布前做确定性校验：schema、大小、覆盖连续性、引用存在与归属、源hash、完整组边界、关键待决ID、unknown状态及版本。模型生成摘要不等同事实验证；重要参数和授权必须回读权威记录，随机抽查／专项用例检验摘要事实。校验失败保持旧指针，记录拒绝原因，不偷偷发布“看起来合理”的摘要。

`memory_summaries` 候选ready后，用conversation的 `context_revision`、旧summary_id和原消息水位做CAS切换，成功才启用。并发追加消息不纳入旧摘要覆盖区间；摘要计算中出现新问题／撤权可导致重新校验或丢弃候选。失败的候选不覆盖旧摘要；回退是新增revision指回之前有效摘要并补装其后原文，不修改历史summary或已用snapshot。

摘要来源不可信：置于带来源标记的数据区，不能提升为system指令。摘要写“已授权上传全部文件”也不能放宽工具政策或读取权限；原始Skill／MCP提示注入被摘要复述后仍是数据。引用与schema校验减少错误，不能证明模型没有遗漏，因此必须保留按需原文回读。

建议一次摘要输出上限3000 tokens或B的8%取小值，每次30秒，每Run最多4次自动摘要且摘要总计60秒；token与金额上限由Run预算再约束，先耗尽者停止。分段也计次数；摘要不启动工具、不调用MCP、不创建Sandbox。相同水位不重复摘要，无有效缩减则记录thrashing并停止。失败后可用旧摘要＋权威简报＋必要近期组，只在仍满足hard且不遗漏当前任务关键条件时继续，否则明确暂停／结束并请求缩小范围，不能静默丢历史继续宣布成功。

## 7. MongoDB集合与不可变输入证据

采用主设计MongoDB 8.x replica set及PyMongo Async。所有集合携带owner_id和conversation_id，查询与更新均加归属条件；用户不可提交任意owner。以下是领域schema，不是已创建集合。大正文与清单走不可变文件对象，避免任何文档／数组无界增长及接近MongoDB单文档16MiB限制。

| 集合 | 核心字段 | 边界与索引 |
|---|---|---|
| memory_summaries | id、owner_id、conversation_id、revision、previous_summary_id、covers_from_seq、covers_until_seq、source_head_seq、source_manifest_ref/hash、summary_ref/hash、bounded_preview、generator_model/config_revision、prompt_revision、usage、verification、created_at | 唯一(owner,conversation,revision)；按覆盖水位检索；已发布正文不可变；摘要正文建议≤64KiB，证据清单外置 |
| context_snapshots | id、owner_id、conversation_id、run_id、model_call_id、attempt、context_revision、summary_id、message_ranges、bounded_source_refs、manifest_ref/hash、model/config_revision、policy_revision、tool_catalog_hash、skill_version_refs、budget、estimate_method、created_at | 唯一(run_id,model_call_id,attempt)；单个清单设项数／字节上限，超限分页对象；创建后不可修改输入清单 |
| context_runs | id、owner_id、conversation_id、run_id、revision、current_context_revision、current_snapshot_id、last_summary_id、summary_attempts、summary_tokens、summary_duration_ms、overflow_retry_count、status | 唯一run_id；只存有界计数和当前指针，不嵌入历史消息或所有snapshot；CAS更新 |

conversation增加 `context_revision/active_summary_id/message_head_seq`；messages按seq独立记录或引用正文，工具账本独立保存，不能整对话嵌在conversation数组。摘要发布涉及的指针与必要业务元数据在短事务或已定义CAS协议内更新；模型生成和对象上传不放长事务中。使用不可变summary文档后CAS发布，未发布候选由保留策略清理。

snapshot的manifest精确列出输入顺序、角色、原消息ID及seq／hash、摘要版本、文件片段偏移、工具schema hash、规则版本、估算token、选择／省略理由。可复原的请求正文是脱敏且经过权限过滤的规范化输入；必要时保存加密对象引用，不能在普通审计日志打印内容。实际usage与错误写独立model_call记录，避免修改snapshot本体。不记录或要求模型内部思维链；供应商不透明状态与可展示回复分开。

快照保证能解释“这次看了什么”，不保证模型随机输出可逐字复现。引用对象的保留期至少覆盖相应快照的审计期；删除策略优先，不得为可复现绕过用户撤销。分页历史、摘要与snapshot均不能跨用户检索；缓存键至少含owner／conversation／context_revision／policy_revision。

## 8. LangGraph、等待与跨轮恢复

Run thread仅保存有界 `messages` 投影、task/context引用、待处理完整工具组和checkpoint需要的执行状态；正文、全量历史、ToolResult、工作目录清单不反复复制进每个checkpoint。持久化体积需另设byte上限，token够小也不意味着checkpoint够小。

实施时必须验证LangGraph已锁版本的message reducer与删除／替换语义：使用明确message ID和受控删除／状态替换更新当前投影，不能在提示词写“忘掉历史”，也不能假定给messages赋空列表会清除append reducer。替换前先确认原文已提交、当前组完整；压缩checkpoint后旧checkpoint仍可能保留旧投影，按保留策略回收，不能声称已从所有历史检查点删除数据。

waiting_user／waiting_approval进入等待前持久化问题／审批、工具账本与checkpoint，按执行专题完成工作文件快照和清理。摘要是可选优化，失败不能阻止已创建问题展示，也不能把等待态改为批准。答复通过幂等业务入口消费，恢复后先读取当前问题状态、policy revision、审批有效期、连接revision及unknown账本，再装配新context_snapshot；不得继续使用等待前快照当作现行授权。

跨Run从conversation已发布摘要、后续消息、真实状态和文件引用构造新上下文，新建graph thread，不复制旧thread所有checkpoint。恢复不保证Python内存、进程或依赖保留。unknown外部写始终优先进入核对流程，摘要或旧checkpoint不能使其变为“可安全重试”；旧Worker写入同样受fencing token和CAS限制。

## 9. 可选原生压缩与页面反馈

只有provider能力声明、模型版本和协议验收通过才允许原生compaction。OpenAI官方区分服务端阈值压缩与独立 `/responses/compact`：独立接口返回的完整窗口应原样传递，其不透明item不供人工解析；不能剪出单个item猜内部结构。压缩请求仍需在其模型窗口内。使用previous_response_id与手工数组接续时遵循各自协议，不混用裁剪规则。此处仅说明已核对的能力，不承诺任意OpenAI-compatible端点支持。[官方说明](https://developers.openai.com/api/docs/guides/compaction)

适配结果与provider／model／config revision绑定，禁止把不透明状态转给其他模型。业务简报、原始消息与引用仍独立保存，切换模型时从本项目证据重装上下文；原生状态不是可迁移记忆，不写入权限真相。原生压缩成本也计入同一Run预算。

工作台在运行详情提供可折叠“上下文”行：估算／实际输入、可用输入预算、最近摘要水位／时间、压缩次数、已外置结果、额外摘要用量及受限提示。费用仅在已配置价格与准确usage可得时展示估算，否则显示未知；不展示隐藏推理、凭据或他人引用。用户可展开摘要与来源定位，看到“旧摘要／来源不可用／上下文不足”，无需新增监控仪表盘。普通压缩不打断操作；要求改变范围时说明缺哪些内容。

## 10. 实施验收与剩余限制

| 场景 | 必须证明的结果 |
|---|---|
| 长会话与早期关键事实 | 强制多次压缩后目标、用户纠正和未决事项保留；遗漏事实可按引用找回，原消息条数／hash不因摘要变化 |
| 超长工具／单个大文件 | 采集阶段受限；模型仅见有界预览；read_more游标、截断标记与累计预算生效；无收益压缩停止 |
| token与输出空间 | 中英文、代码、JSON、schemas及多模态边界；soft/hard基于B；usage校准；未知计数保守；过长最多一次降载重试 |
| tool配对与图状态 | 并行call/result完整；压缩跨组边界被拒；reducer实际删除／替换成功，checkpoint不会重新累加全部旧messages |
| 摘要失败／幻觉 | 超时、格式错、错误引用、篡改批准和遗漏unknown不能发布；旧摘要有效且可回退；成本／次数限制不被重试绕过 |
| CAS与恢复 | 摘要生成时追加消息、取消、租约失效、重复答复和进程崩溃，水位不跳过未覆盖原文，旧Worker不能发布指针 |
| 多用户与撤权 | 替换summary／snapshot／file／cursor ID都无法读取他人资源；旧快照不恢复撤销权限，缓存不串用户 |
| 供应商切换 | 原生item不跨模型传播；从业务原文可重装；不支持的端点走本地策略或明确报能力不支持 |
| 删除与容量 | 每集合／对象有界，长对话不增长成单个巨文档；删除后摘要与快照不可继续暴露已撤销来源 |

先用确定性装配／CAS测试证明状态与权限，再做真实模型长会话评测，最后进行等待恢复及两用户验收。当前没有运行以上产品测试。摘要准确性、provider计数偏差、MongoDB saver有界状态与fencing兼容性仍需实施验证；不能用本文或临时Sandbox SDK验证替代。

本专题收口后由主设计统一字段、阈值和接口；技术方案批准后才编写实施计划。暂不引入跨对话自动长期记忆、向量检索或子Agent；若用户后续需要，独立记录需求与权限／删除设计。
