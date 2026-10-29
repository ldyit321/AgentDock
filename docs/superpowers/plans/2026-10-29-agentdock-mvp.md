# AgentDock MVP Implementation Plan

> **2026-10-29修订：本文件仅保留技术任务拆分参考。用户已要求暂停并回退试启动内容；现行执行方式以[逐节点实施与交付计划v1.2](../../../outputs/AgentDock-逐节点实施与交付计划-v1.2.md)及AGENTS.md第10节为准。每节点完成后输出交付报告，根据实际实现形成验收清单并停止，不自动连续执行。当前所有任务均未验收完成；认可计划不授权开工。**

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在Windows工程中交付可运行的多用户通用Agent网页MVP，以一个实际Skill套件完整工作流验收，并保留确定性无模型验证路径。

**Architecture:** 模块化单体，一份后端工程、一个API进程和一个独立Worker。MongoDB副本集保存业务、队列及检查点，不可变文件由StorageProvider管理；LangGraph通过明确的模型、MCP和Sandbox适配器执行任务。领域方法保留在Skill，不建立CAD专用图节点。

**Tech Stack:** Vue 3＋JavaScript＋Element Plus/Vite；Python 3.12/FastAPI/Pydantic/LangGraph；MongoDB 8.x/PyMongo Async；OpenSandbox SDK 1.2.0起步。准确依赖版本在任务1核验并锁定，本文不声称它们已经安装。

**Spec:** [主技术设计](../../../outputs/AgentDock-技术设计-v1.1.md)、[需求基线](../../../outputs/AgentDock-需求与功能基线-v1.0.md)、[执行恢复](../../../outputs/AgentDock-执行与恢复设计-v1.1.md)、[通用宿主](../../../outputs/AgentDock-通用Skill宿主能力设计-v1.1.md)。UI、存储、上下文、多模态专题经主稿进入；根目录AGENTS.md约束全程。

## Global Constraints

- 工程根目录为`D:\Projects\AgentDock`；应用在Windows开发运行，不在Ubuntu编辑工程。
- 第一版多用户网页；前端Vue 3＋JavaScript，Element Plus；数据库MongoDB，不用React／TS业务源码／PostgreSQL。
- 中文注释率目标15%，按AGENTS.md汇总口径；DOM／Vue模板标记排除，JS逻辑不排除。
- `.env`不得覆盖、输出或提交；管理密钥不得进入前端、模型、Skill或Sandbox。
- 不在宿主机执行上传脚本；结构化argv进入适配层，当前镜像字符串兼容仅集中安全引用转换。
- 用户范围授权允许连续执行；删除、覆盖、公开发布、正式归档、范围扩展需要独立确认。摘要和Skill文字不能授予权限。
- 首版单机多用户、单API／单Worker；MongoDB单节点副本集支持事务但不提供高可用；本地文件必须持久化并可备份。
- 无模型Key可运行确定性流程；真实Agent需要模型配置。无Key不等于假装LLM已读Skill。
- 本文是计划，所有复选框尚未完成；不授权安装、部署或真实外部业务写入。本文建议的测试命令均留待开发阶段执行。

## Review Focus

1. 用户B拿到用户A的file／question／connection／SSE标识：拒绝访问且不泄露内容，任务2、3、12、14覆盖。
2. 工具远端已写入但响应丢失或进程崩溃：标unknown并跨对话阻断后续受控写，任务6、10、14覆盖。
3. 重复答复、答复后立刻重启、检查点写入未结束：只消费一次且恢复到确定指针，任务1、7、14覆盖。
4. 恶意ZIP、越界相对路径、输出洪水或超限文档：边界内拒绝／截断并清理，不执行宿主代码，任务3、4、5、9覆盖。
5. 长对话中扫描图片或旧授权被摘要替代：原图可回读、实际入模，授权回数据库核验，任务8、9、11覆盖。

## 阶段与验收门

| 阶段 | 任务 | 可独立检查的结果 |
|---|---|---|
| A 基础与风险验证 | 1～4 | 兼容证据、身份隔离、文件和Skill版本入库 |
| B 可恢复确定性执行 | 5～7 | 无模型也能执行、保存、问答等待和恢复；未知副作用不重放 |
| C 模型与通用宿主 | 8～11 | 真正读Skill、多模态、上下文、MCP/OAuth、外部资源和视图 |
| D 网页与验收 | 12～14 | 完整网页、多轮操作、两用户隔离、一个真实工作流证据 |

这些子系统共享身份、文件和Run状态契约，本轮用一个顺序集成计划避免平行定义冲突；阶段不是删减首版功能。若后续并行开发，先完成任务1～3，再为独立模块拆独立计划。

## 文件结构与接口约定

以下均为**计划创建**，不是当前存在的产品文件。已有`work/verify_opensandbox.py`仅作历史证据，不复制成产品执行器。

```text
backend/
  pyproject.toml, uv.lock
  src/agentdock/
    settings.py, bootstrap.py
    domain/contracts.py, errors.py
    api/app.py, auth.py, conversations.py, files.py, runs.py, skills.py, connections.py, events.py
    persistence/mongo.py, repositories.py, unit_of_work.py
    services/identity.py, files.py, skills.py, runs.py, interactions.py, policies.py, assets.py
    harness/graph.py, tools.py, context.py, prompts.py
    integrations/storage/local.py
    integrations/sandbox/client.py, workspace.py, profiles.py
    integrations/llm/client.py, media.py
    integrations/documents/parser.py
    integrations/mcp/client.py, auth.py, resources.py, views.py
    worker/main.py, scheduler.py, checkpoints.py, cleanup.py
  migrations/001_initial.py
  tests/unit/, tests/integration/, tests/fixtures/
frontend/
  package.json, package-lock.json, vite.config.js
  src/app/router.js, session.js, theme.css
  src/api/client.js, events.js, contracts.js
  src/features/auth/LoginPage.vue
  src/features/conversations/WorkbenchPage.vue, useConversation.js
  src/features/skills/SkillsPage.vue
  src/features/connections/ConnectionsPage.vue
  src/features/admin/AdminPage.vue
  src/components/QuestionPanel.vue, ApprovalPanel.vue, TodoPanel.vue, ArtifactPanel.vue
  tests/
scripts/start-dev.ps1, check-comments.py
infra/compose.mongo.yml
.env.example, README.md
```

`domain/contracts.py`只放真正跨模块的Pydantic DTO／枚举：Actor、FileRef、RunContext、RunRequest、RunView、CommandSpec、ExecutionResult、SnapshotRef、InteractionAnswer、ContextSnapshot、ModelReply、McpCall、ToolObservation、TaskGrant、AssetManifest。ID用str，时间用带时区datetime，文件字节不嵌进DTO；凭据和SDK客户端不进入可序列化DTO。模块内部类型就近定义，文件变大时按责任拆分，禁止写成万能contracts／repositories文件。

下文异步Python接口均使用`async def`；构造依赖在bootstrap显式注入，Actor由身份服务生成。测试命令在工程根目录执行；后端使用`uv run --project backend pytest backend/tests/...`，前端使用`npm --prefix frontend ...`。每项先写失败测试、确认确实因缺失目标行为失败，再实现并通过；依赖缺失或环境故障不算有效红灯。每项完成在仓库存在且允许提交时创建小提交，不提交`.env`、data或work虚拟环境；本计划不自动初始化或提交仓库。

## Task 1：依赖与持久化兼容性门

**Files:** Create `backend/pyproject.toml`、`backend/uv.lock`、`infra/compose.mongo.yml`、`backend/tests/integration/test_checkpoint_boundary.py`、`docs/verification/compatibility.md`。

**Interfaces:** 消费已有SDK报告；产出锁定版本、Mongo副本集配置和`CheckpointStore`可实现性结论，交任务6。不修改现有Server配置。

- [ ] 写`test_checkpoint_promotion_waits_for_pending_writes`：延迟saver写时不得发布新confirmed指针；取消后未完成写不能被下一执行者读取为已确认状态。
- [ ] 在获准开发后，核对官方LangGraph示例、SDK、MCP、MongoDBSaver实际API；锁定Windows Python3.12可解依赖集合。记录版本、来源日期及Node要求，不称当前最新。
- [ ] 运行上述集成测试确认目标行为尚未实现；部署隔离测试副本集前检查端口／卷名不覆盖现有数据。
- [ ] 验证事务、唯一索引冲突、saver异步封装、pending writes、interrupt／resume；形成通过或阻断报告。若不能安全绑定确认指针，停止恢复功能实施，先修订方案，不改saver私有源码冒充原子事务。
- [ ] 运行`uv run --project backend pytest backend/tests/integration/test_checkpoint_boundary.py -v`，目标全部PASS；记录真实环境及取消窗口证据，再提交`chore: pin verified runtime dependencies`。

## Task 2：身份、Mongo仓储与API基础

**Files:** Create `settings.py`、`bootstrap.py`、`domain/contracts.py`、`domain/errors.py`、`persistence/{mongo,repositories,unit_of_work}.py`、`services/identity.py`、`api/{app,auth}.py`（均在上述后端根）、`backend/migrations/001_initial.py`、`backend/tests/integration/test_identity.py`。

**Interfaces:** `IdentityService.authenticate(session_token: str) -> Actor`；`require_owner(actor: Actor, owner_id: str) -> None`；仓储构造时注入Mongo客户端，事务session每用例独立。

- [ ] 写`test_private_resource_rejects_other_user`、`test_session_revocation`：A/B账号隔离，撤销Cookie立即失效，错误不含敏感字段；管理员角色不自动获得私有内容读取权。
- [ ] 运行`uv run --project backend pytest backend/tests/integration/test_identity.py -v`确认目标失败。
- [ ] 实现Argon2id、不透明Cookie会话、CSRF／Origin检查、登录限速、角色和归属校验、统一错误脱敏；迁移建唯一索引和schema_version，账号创建通过管理员受限入口，不开放公开注册（采用D01可调整默认）。
- [ ] 重跑身份测试全部PASS；新建库／重复迁移结果相同，原数据不删除；提交`feat: add authenticated application foundation`。

## Task 3：用户文件、不可变发布与快照

**Files:** Create `services/files.py`、`integrations/storage/local.py`、`api/files.py`、`backend/tests/integration/test_files.py`。

**Interfaces:** `FileService.upload(actor: Actor, conversation_id: str, stream: AsyncIterator[bytes], name: str) -> FileRef`；`open_owned(actor: Actor, file_id: str) -> AsyncIterator[bytes]`；`publish_snapshot(context: RunContext, manifest: list[FileRef]) -> SnapshotRef`。StorageProvider使用逻辑key，不把Windows路径返回模型。

- [ ] 写`test_upload_publish_interrupted`、`test_same_name_distinct_version`、`test_foreign_file_denied`：中断上传不可ready，同名不覆盖，B无法下载A文件；输入超过20MiB中止（D默认值配置化）。
- [ ] 运行`uv run --project backend pytest backend/tests/integration/test_files.py -v`确认失败。
- [ ] 实现流式限额、staging→唯一不可变key→元数据ready、SHA256、按用户配额原子预留／释放；快照版本CAS，无变更保留原working_snapshot_id，GC先确认引用。
- [ ] 重跑测试PASS，补偿失败重试不双扣配额；提交`feat: isolate immutable files and workspace snapshots`。

## Task 4：SkillBundle与运行环境选择

**Files:** Create `services/skills.py`、`api/skills.py`、`integrations/sandbox/profiles.py`、`backend/tests/unit/test_skill_bundle.py`、`backend/tests/fixtures/skills/`。

**Interfaces:** `SkillService.import_bundle(actor: Actor, archive_id: str, visibility: str) -> str`；`resolve(actor: Actor, version_ids: list[str]) -> SkillBundle`（SkillBundle就近定义）；`select_profile(bundle: SkillBundle) -> RuntimeProfile`。配置只接受管理员登记的环境，不允许Skill提供任意宿主安装命令。

- [ ] 写`test_zip_traversal_rejected`、`test_sibling_reference_within_bundle`、`test_private_dependency_denied`：绝对路径／软链接／越界拒绝；合法兄弟引用保留；同名不同版本和owner不串包。
- [ ] 运行`uv run --project backend pytest backend/tests/unit/test_skill_bundle.py -v`确认失败。
- [ ] 实现有界解包、SKILL.md读取、依赖图循环检测、作用域与hash锁定；公共源只读，私有源隔离；requirements为资料而非自动获准pip，RuntimeProfile绑定镜像digest、依赖锁和资源。
- [ ] 重跑测试PASS；实际`D:/Work/skills`仅只读对照，不自动导入、更改或执行；提交`feat: version authorized skill bundles`。

## Task 5：Sandbox执行与确定性脚本闭环

**Files:** Create `integrations/sandbox/{client,workspace}.py`、`worker/cleanup.py`、`backend/tests/integration/test_sandbox_execution.py`、`backend/tests/fixtures/skills/tabular-report/`（SKILL.md、scripts/report.py、requirements.txt、input.csv）。

**Interfaces:** `SandboxExecutor.execute(context: RunContext, command: CommandSpec) -> ExecutionResult`；`WorkspaceService.restore(context: RunContext, snapshot: SnapshotRef | None) -> None`；`finalize(context: RunContext) -> SnapshotRef | None`。CommandSpec包含argv、cwd、timeout_s；ExecutionResult明确stdout、stderr、exit_code和truncated。

- [ ] 写`test_exit_seven_preserved`、`test_timeout_cleans_instance`、`test_output_flood_bounded`、`test_success_snapshot_before_destroy`；模拟SDK保证失败分支可确定复现，再单独标实际SDK测试。
- [ ] 运行`uv run --project backend pytest backend/tests/integration/test_sandbox_execution.py -v`确认目标失败，不无理由重跑历史小输出探针。
- [ ] 实现按需创建、固定argv安全引用兼容、脚本输入上传、输出与退出码采集、产物下载校验、finally清理＋TTL、清理状态；默认命令60s、stdout/stderr各64KiB、代码256KiB，均配置化。核实SDK是否流式限制；若只能执行后截断，标传输限制未满足并采用真实可控运行包装，不能声称已防输出洪水。
- [ ] 跑测试PASS；无LLM Key时用明确deterministic模式执行测试Skill并下载报告，正常exit0／故意exit7均留证，宿主不执行上传代码；提交`feat: execute bounded sandbox workloads`。

## Task 6：持久队列、执行账本与检查点

**Files:** Create `services/runs.py`、`worker/{main,scheduler,checkpoints}.py`、`api/runs.py`、`backend/tests/integration/test_run_recovery.py`。

**Interfaces:** `RunService.submit_run(actor: Actor, conversation_id: str, request: RunRequest) -> RunView`；`claim_next(worker_id: str) -> RunContext | None`；`CheckpointStore.confirm(run_id: str, epoch: int, checkpoint_id: str) -> None`；`ToolLedger.mark_unknown(call_id: str, reason: str) -> None`（ledger在services/runs.py初始实现，变大再拆）。

- [ ] 写`test_enqueue_epoch_not_incremented_at_claim`、`test_reservation_release_once`、`test_lost_write_response_blocks_owner`、`test_old_worker_cannot_promote_checkpoint`：global2/user1，重复释放不负数，发送后断线不自动重放；共享连接额外跨用户阻断该连接写。
- [ ] 运行`uv run --project backend pytest backend/tests/integration/test_run_recovery.py -v`确认失败。
- [ ] 实现scheduler_control、user_usage、capacity_reservations冲突点事务、同对话单非终态Run、单Worker OS锁、心跳epoch CAS；planned→authorized→sending先持久化再调用。unknown生成operation_blocks，只有核对服务可解除；不把checkpoint和业务写当同一事务。
- [ ] 重跑PASS；重启只恢复确定的等待／排队状态，活跃执行中断，未完成saver窗口不选“最新”冒充已确认；提交`feat: persist bounded runs and safe recovery`。

## Task 7：Question Human、审批与Todo

**Files:** Create `services/interactions.py`、`harness/graph.py`、`backend/tests/integration/test_interactions.py`；Modify任务6运行与检查点服务。

**Interfaces:** `InteractionService.answer_question(actor: Actor, question_id: str, answer: InteractionAnswer) -> RunView`；`decide_approval(actor: Actor, approval_id: str, allowed: bool) -> RunView`；`set_todos(context: RunContext, items: list[TodoItem]) -> None`，TodoItem就近定义，完成状态与执行证据分开。

- [ ] 写`test_duplicate_answer_applied_once`、`test_answer_then_restart`、`test_multi_tool_interrupt_keeps_pairs`、`test_wait_snapshot_cleanup`：整组question_id最多3题，重复答复只一个resume_request；第三方用户拒绝；调用结果成对，不丢同批其他工具。
- [ ] 运行`uv run --project backend pytest backend/tests/integration/test_interactions.py -v`确认失败。
- [ ] 实现prepare→interrupt检查点→publish→答复事务→apply_answer检查点→后续工具；等待释放active名额、保留对话占用、快照清理。问题24h／审批10min／Run48h为默认；审批过期产生拒绝观察，未回答不当授权。
- [ ] 重跑PASS，恢复不能重执行interrupt前副作用；取消/过期释放一次，保留清理证据；提交`feat: persist human questions approvals and todos`。

## Task 8：模型循环、Skill读取与上下文

**Files:** Create `harness/{tools,context,prompts}.py`、`integrations/llm/client.py`、`backend/tests/unit/test_context.py`、`backend/tests/integration/test_agent_loop.py`；Modify `harness/graph.py`。

**Interfaces:** `ContextBuilder.build(context: RunContext) -> ContextSnapshot`；`ModelClient.complete(snapshot: ContextSnapshot) -> ModelReply`；`ToolRouter.invoke(context: RunContext, name: str, arguments: dict) -> ToolObservation`。模型适配只接受已装配上下文，不能直接取密钥／任意文件。

- [ ] 写`test_skill_content_precedes_script_action`、`test_tool_history_pairs_survive_compaction`、`test_summary_cannot_authorize`、`test_no_key_deterministic_mode`：模型必须得到可见版本SKILL.md内容；历史截断不拆call/result；无Key的Agent端点明确配置缺失，deterministic路径仍运行。
- [ ] 运行`uv run --project backend pytest backend/tests/unit/test_context.py backend/tests/integration/test_agent_loop.py -v`确认失败；模型用可控fake断言真实请求结构，不以fake证明真实模型能力。
- [ ] 实现select/read_skill→plan/model→串行tools→observe→final循环、30步上限和可关闭最多2次修复；修复只改工作副本，基础设施故障不修改业务脚本。提示词引用规则，服务端单独执行权限和预算。
- [ ] 实现输入预算B=min(max_input,C-O-M)、soft70%B／hard85%B／target55%B、来源摘要CAS、工具外置read_more、逐次context_snapshot、原文可回读；没有tokenizer时保守1.3估计，模型能力配置不硬编码品牌。
- [ ] 重跑PASS，补`test_compaction_exhaustion_is_bounded`证明同水位不重复压缩且超预算明确终止；提交`feat: run skill aware bounded model loops`。

## Task 9：文档解析与真实多模态

**Files:** Create `integrations/documents/parser.py`、`integrations/llm/media.py`、`backend/tests/unit/test_document_inputs.py`、`backend/tests/fixtures/documents/`；Modify `services/files.py`、`worker/scheduler.py`、`harness/context.py`。

**Interfaces:** `DocumentParser.parse(context: RunContext, file: FileRef, options: ParseOptions) -> ParsedDocument`（两个DTO就近定义）；`MediaAdapter.build_parts(actor: Actor, document: ParsedDocument, page_ids: list[str]) -> list[dict]`。只在请求发送边界生成image parts，检查点保存引用。

- [ ] 写`test_scanned_page_is_image_part`、`test_partial_pages_report_coverage`、`test_parse_key_includes_range`、`test_ready_binary_not_forced_to_text`：扫描页真进入图片请求，部分解析声明范围，STEP等允许的二进制可交工具，文件ready和解析ready分开。
- [ ] 运行`uv run --project backend pytest backend/tests/unit/test_document_inputs.py -v`确认失败。
- [ ] 核验候选解析库许可证后实现PDF文字／页面图、DOCX结构、XLSX数据索引、PPTX文字／图片；需外部渲染器的功能显示能力状态，未安装不假装支持。禁宏／主动内容，有界页数、像素和解压；解析单独slot1，不占满Run槽。
- [ ] 实现Chat兼容视觉请求；不支持tool-role图片时先补完整tool结果，再发送明确标记的用户角色观察媒体，不视为新授权。Files／Responses／原生PDF单独探测。
- [ ] 重跑PASS；真实模型验收前检查能力矩阵，日志和context_snapshot没有base64大图／临时签名URL；提交`feat: understand bounded document and image inputs`。

## Task 10：MCP连接、OAuth、工具策略与资源

**Files:** Create `integrations/mcp/{client,auth,resources}.py`、`services/policies.py`、`api/connections.py`、`backend/tests/integration/test_mcp_boundary.py`。

**Interfaces:** `McpGateway.call(context: RunContext, call: McpCall) -> ToolObservation`；`AuthService.begin(actor: Actor, connection_id: str) -> str`；`complete_callback(state: str, code: str) -> None`；`PolicyService.authorize(context: RunContext, call: McpCall) -> AuthorizationDecision`；`ResourceImporter.import_resource(context: RunContext, handle: str) -> FileRef`。新增DTO就近定义，不让调用方传解密凭据。

- [ ] 写`test_oauth_state_bound_to_owner`、`test_redirect_cannot_reach_forbidden_host`、`test_signed_resource_hash_mismatch`、`test_timeout_write_becomes_unknown`：PKCE/state单用且过期，重定向重新做出站检查，资源超限／hash不符不ready，401/403/网络故障分类。
- [ ] 运行`uv run --project backend pytest backend/tests/integration/test_mcp_boundary.py -v`确认失败。
- [ ] 实现用户私有远程HTTP、管理员共享／登记stdio、无认证／静态Bearer／标准OAuth；凭据AEAD加密且不进模型。区分API/模型、MCP、Sandbox三套网络策略，DNS／重定向检查及部署网络限制共同强制，HTTP私网服务须管理员明确允许。
- [ ] 实现opaque资源句柄、限量导入、来源／hash／有效期、下载失败不重发原导出工具。识别opaque远程程序无法可靠分类时走具体参数或代码hash审批，不凭模型自报只读。
- [ ] 重跑PASS；真实create_document内部重试先验证幂等／核对／关闭危险重试能力，否则自动写标blocked，不以外层不重试声称安全；提交`feat: connect authorized mcp tools and resources`。

## Task 11：任务范围授权、外部视图与长期资料

**Files:** Create `integrations/mcp/views.py`、`services/assets.py`、`backend/tests/integration/test_host_capabilities.py`；Modify `services/policies.py`、`domain/contracts.py`。

**Interfaces:** `PolicyService.evaluate_grant(context: RunContext, grant: TaskGrant, call: McpCall) -> AuthorizationDecision`；`ExternalViewPort.capture(context: RunContext, view_id: str) -> FileRef`；`AssetService.publish(actor: Actor, approval_id: str, manifest: AssetManifest) -> str`。领域状态引用DomainStateRef就近定义，指向有hash／schema／来源的文件版本。

- [ ] 写`test_grant_cannot_expand_target`、`test_summary_does_not_restore_revoked_grant`、`test_archive_requires_distinct_approval`、`test_cross_conversation_asset_has_no_old_grant`：权限从DB取，目标／时间／参数／次数越界拒绝；旧OAuth与审批不随资料复制。
- [ ] 运行`uv run --project backend pytest backend/tests/integration/test_host_capabilities.py -v`确认失败。
- [ ] 实现TaskScopedGrant、明确范围内连续执行、敏感操作另批、AssetCollection不可变版本发布；用户说“效果可以”不等于正式归档。DomainStateRef不混为Todo或checkpoint。
- [ ] 实现登记的外部视图适配与会话／目标绑定；截图作为文件进入任务9视觉请求。不能把当前Codex桌面工具当产品能力，目标站禁止嵌入时显示受控外链／截图，不能假设iframe可用。
- [ ] 重跑PASS；缺少真实视图后端时明确blocked，不用静态演示图代替实际模型证据；提交`feat: support scoped tasks views and reusable assets`。

## Task 12：网页基础、事件与管理界面

**Files:** Create上述前端配置与`src/app/`、`src/api/`、LoginPage、SkillsPage、ConnectionsPage、AdminPage；Create `api/events.py`、`frontend/tests/session.test.js`、`frontend/tests/access.spec.js`。

**Interfaces:** `apiRequest(path, options) -> Promise<object>`（JSDoc定义响应）；`subscribeConversation(conversationId, onEvent) -> () => void`返回关闭函数；事件结构沿用主设计，不向前端发送密钥。后台REST路由用主设计18.2清单，实施时以OpenAPI生成校验资料，业务JS不变成TS。

- [ ] 写`session.test.js`断言401清空会话、Cookie同源请求且CSRF请求头；`access.spec.js`断言B订阅A的SSE拒绝、重连先核验权限；secret字段不进入页面／响应。
- [ ] 运行`npm --prefix frontend run test -- --run`确认失败。
- [ ] 实现Vue3/JS/ElementPlus页面、路由守卫和服务端权限双层检查；Skill版本上传／发布、MCP OAuth连接状态、管理员模型／网络／环境配置。密码／密钥用只写表单不回显；管理角色不默认浏览他人文件。
- [ ] SSE持久事件ID＋重连补读、去重；间隙过期时刷新权威快照，不用事件内存缓存作唯一状态。开发代理与上线同源Cookie路径分别配置。
- [ ] 单测PASS，运行`npm --prefix frontend run build`和`npm --prefix frontend run test:e2e -- tests/access.spec.js`通过；提交`feat: add authenticated vue management screens`。

## Task 13：对话工作台与完整交互

**Files:** Create `features/conversations/{WorkbenchPage.vue,useConversation.js}`、共享四个Panel、`backend/src/agentdock/api/conversations.py`、`frontend/tests/workbench.spec.js`。

**Interfaces:** `useConversation(conversationId)`返回`messages, run, todos, interactions, artifacts, sendMessage, cancelRun, answerQuestion, decideApproval`；所有写入调用后端服务，不由UI直接改终态或归属。

- [ ] 写`workbench.spec.js`：上传图片／文档展示解析中与部分覆盖；Skill自动／限定；多轮追加；Question整组一次提交；审批超时和unknown有明确阻断；产物下载归属检查，取消后清理状态可见。
- [ ] 运行`npm --prefix frontend run test:e2e -- tests/workbench.spec.js`确认目标缺失导致失败。
- [ ] 按UI专题实现工作台、Todo、问题／审批、工具进度与证据、文件／产物预览、上下文用量和摘要来源；等待可刷新恢复、发送按钮根据服务端会话状态禁用，支持键盘／焦点／可读错误。
- [ ] 同命令PASS；宽屏与窄屏人工检查，无Key显示确定性模式／配置引导，不把fake回复呈现为真实模型响应；提交`feat: complete conversational agent workspace`。

## Task 14：代表工作流、故障验收与Windows交付

**Files:** Create `backend/tests/integration/test_acceptance.py`、`frontend/tests/multiuser.spec.js`、`scripts/{start-dev.ps1,check-comments.py}`、`.env.example`、`README.md`、`docs/verification/acceptance.md`；Modify依赖锁及相关使用说明。

**Interfaces:** 消费前13项全部接口；交付Windows启动步骤、备份／恢复步骤、测试报告与明确阻断项，不产生额外框架。测试配置与真实用户配置隔离。

- [ ] 写验收用例：`test_two_users_isolated`、`test_cancel_and_cleanup`、`test_restart_with_pending_question`、`test_unknown_blocks_later_conversation`、`test_next_turn_restores_files_not_process`。纳入SSE、问题、资源、文件、连接、checkpoint越权。
- [ ] 运行`uv run --project backend pytest backend/tests/integration/test_acceptance.py -v`及`npm --prefix frontend run test:e2e -- tests/multiuser.spec.js`，保留失败证据。
- [ ] 首先用项目自带tabular-report fixture完成无Key确定性闭环，再用真模型完成读Skill、问答、脚本失败／最多2次修复、产物、多轮追加；两者分别记录，不混为同类通过。
- [ ] 在用户实际配置模型／完成OAuth且明确授权目标后，选真实套件中一个代表任务：导入参考→分析／必要提问→代码或工具操作→真实结果核验→下载产物；若为CAD任务须另给目标与可写文档范围，不擅自选真实生产文档。强原生依赖先在隔离Linux镜像smoke验证，失败留兼容报告，不回退宿主执行；可先选不依赖该分支的样例，但不得称全部套件兼容。
- [ ] 验证多用户额度、超时、超限输出／文件、资源失效、401／403、临时模型故障、清理失败与重启；模拟故障及真实服务证据分列。正式归档另授权，不为了验收主动发布或删除用户数据。
- [ ] 完成README：先决条件、Mongo副本集、API/Worker/前端启动、env字段、deterministic/agent模式、用户空间与备份恢复、配置迁移；`.env.example`只占位，不复制已有.env。启动脚本隐藏辅助进程窗口并提供明确停止方式，不能停止无关现有服务。
- [ ] 运行后端全套相关测试、Ruff、前端测试／构建和上述e2e；运行`python scripts/check-comments.py backend/src frontend/src`记录手写逻辑中文注释率及排除项，目标15%，不足明确修订，不能填充无意义注释。新备份在独立目录恢复校验内容hash和归属。
- [ ] `docs/verification/acceptance.md`逐项列出静态／fake／真实集成／多用户证据、实际版本及剩余限制；只有条件满足才称MVP验收通过。提交`test: verify agentdock mvp acceptance and windows setup`。

## 计划自检与覆盖

| 主设计范围 | 对应任务 |
|---|---|
| 3～7 技术栈、工程、身份、实体 | 1、2；各功能任务补相应集合和索引 |
| 8～10 排队、Harness、工具预算 | 5～8 |
| 11～13 Skill、文件、Sandbox | 3～5 |
| 14～15 人工交互、Todo | 7、13 |
| 16 MCP及通用宿主补充 | 4、10、11 |
| 17 上下文及多模态专题 | 8、9 |
| 18～19 API、SSE、UI | 12、13 |
| 20～24 取消、清理、保护、审计和验收 | 2、5～7、10、14；所有写服务记录actor/run/action及脱敏事件 |
| U24～U26 代表流程、范围授权、单机存储 | 3、11、14 |

自检结果：接口统一使用Actor／RunContext，文件只传FileRef；工具写与授权不能靠模型输出；任务6消费任务1兼容门，任务7依赖已确认指针，任务11媒体进入任务9而非只展示路径。五个Review Focus均落到有名测试，当前只核对计划覆盖，并未运行这些未来测试。

**首轮不包含：**多节点执行、对象存储实际部署、计费／组织SSO、通用插件市场、多Agent自动协作、自动跨会话长期记忆。存储接口与显式AssetCollection保留，不承诺已完成未来系统接入。

**仍需用户在实际联调时配合：**本地模型配置、必要OAuth登录、代表任务输入及目标范围授权；不需要把Key发到聊天。配额等D类默认可先配置化实施，不因本次三个取舍的确认称全部获批。实际Server资源／网络控制和远端内部重试语义必须以证据过门，不能由用户接受设计替代。

## 执行方式与当前状态

计划已完成文档自检，试启动内容已退出活动工程。后续必须逐节点授权、研发、输出完整报告并停靠用户验收；具体人工验收清单于代码完成后根据实际实现形成。Review方式如实记录，不因本文提及子Agent而自动分派。

当前全部实施复选框保持未勾选，研发暂停。用户审阅现行计划并明确授权具体节点后才重新进入产品开发；现有临时SDK成功报告仍只证明已测SDK场景。
