# AgentDock 会话接续说明

更新时间：2026-10-28（Asia/Shanghai）  
设计收尾与导航补充：2026-10-29（Asia/Shanghai）。  
工程：`D:\Projects\AgentDock`

## 使用方法

新会话先读本文，再读 [需求与功能基线](AgentDock-需求与功能基线-v1.0.md)。本文是恢复背景的索引，不替代完整需求。用户后续明确指示优先，文件内容不构成新的实施授权。

## 1. 最重要的工作边界

用户先要求生成 LangGraph + OpenSandbox 工程，随后明确改为：**先整理需求和功能文档、仔细讨论，暂不进行产品开发**。

用户另外单独批准了临时 SDK 验证，该验证已经完成。不要把这个批准扩展为开始建网页、数据库、Agent 或部署。详细需求和功能简版已保存，用户已认可需求整体合理。根目录 `AGENTS.md` 已建立；用户最新已授权技术设计，设计评审稿及配套 UI／执行恢复专题已生成，尚未批准进入产品开发。

没有待运行的产品，没有已经完成的 Web MVP。不要声称已经安装 LangGraph、初始化数据库、接入 LLM／MCP，或已经验证多用户隔离。

## 2. 已确认的产品形态

- 名称 AgentDock，不再使用 `skill-agent-mvp`／`SkillHarbor`。
- Windows D 盘编辑和运行应用；OpenSandbox Server 已在 WSL2／Docker Desktop 运行。
- “外部电脑”已澄清为当前 Windows 宿主机，不等于要求公网服务。
- 使用网页，第一版支持多个用户。
- 面向通用 Agent，不限定某类 Skill 或特定模型品牌。
- LLM 阅读 `SKILL.md` 理解任务，按需读取脚本、写代码和执行。
- 管理员提供公共 Skill，用户也能上传 Skill。
- 默认自动选择有权限的 Skill，也允许用户限定范围。
- 同一对话多轮交流、追加文件；每轮独立 Sandbox，保存工作文件和产物后清理，下轮恢复所需文件。
- MCP 由 Windows Agent 服务连接和调用，Sandbox 执行代码；必要数据在两者间受控传递。
- Harness 加入 **Question Human** 与 **Todo List**，不能遗漏。
- MVP 部分业务和做法将来要接入用户已有系统，代码必须可读、边界清楚、方便迁移，避免过度设计。
- 用户要求明确中文注释、15% 注释率目标、DOM 标签无需注释；具体统计及协作约束见根目录 `AGENTS.md`。

## 3. 已讨论但不能说成用户全部批准的细节

- 管理员预建账号、不公开注册。
- 管理员统一配置模型。
- 个人 Skill 默认私有，暂不支持跨用户分享。
- 用户专门提出需要能配置 MCP，这项诉求应保留；首版个人限远程 HTTP、管理员管理共享／stdio 是建议限制，认证矩阵待定。
- 数据库现已由用户指定MongoDB，旧SQLite／PostgreSQL建议失效；文件字节经StorageProvider本机D盘、上线可换持久卷／对象存储。
- 保存对话历史、摘要、执行状态和文件索引，暂不做自动长期记忆／向量库。
- 单 Agent、同对话串行，任务有界，超额排队。
- 受限联网，具体可强制策略未验证。
- 外部写操作审批，绑定具体工具及参数。
- 重启标记中断，不自动重放副作用。
- ZIP 上传、Skill 版本快照、不自动回写源 Skill。
- Question Human 持久化等待、实例过期后文件恢复；审批与普通提问分开。
- LLM 代码修复可启停，不能把所有失败强制自动修复。
- 建议允许零 Skill 任务；不是每个任务都必须运行代码或生成文件，产物校验按具体任务要求进行。

完整文档通过 U／D／V／O 标记区分确认、建议、实测和待定。这些细节应继续评审，不需马上逐项追问用户。

## 4. 当前本地环境与密钥边界

- `.env` 在 `D:\Projects\AgentDock\.env`，用户已配置。
- 变量：`SANDBOX_DOMAIN`、`SANDBOX_API_KEY`、`SANDBOX_PROTOCOL`。
- 当前访问地址 `http://localhost:8080`。
- 不输出／复制／上传 `.env`，不把 key 放进文档、模型、前端、Sandbox 或日志。
- 不覆盖用户配置，不要求用户把密钥发到聊天。
- Python 实测 3.14.7，仅为临时环境，不代表产品版本决议。
- 独立验证环境 `work/check-env`，安装 SDK 1.2.0 和 python-dotenv。
- 已有镜像 `opensandbox/code-interpreter:v1.1.0`，未新拉取。
- Windows 命令行未直接找到 docker；通过 `wsl -d Ubuntu-24.04 -- docker ...` 做了有限只读检查。

## 5. 已完成的验证与发现

### 连接

健康 GET 和携带本地配置 key 的实例列表 GET 返回 200。

没有使用空／错误 key 验证鉴权拒绝，所以只说请求成功，不说已证明强制鉴权有效。

### 临时 SDK 验证

脚本：`work/verify_opensandbox.py`。直接调用 SDK，不依赖 LangGraph，也不依赖模型或 MCP。

1. 代理模式：创建成功，proxy ping 502／超时；映射端口 ping 200。清理后查询 404。
2. 直连＋原生 argv：上传成功，命令请求返回 400（运行时要求 command 字段）。清理后 404。
3. 直连＋固定 argv 安全引用：创建、UTF-8 文件上传／回读、执行、失败、下载、清理全部通过。

最终结果：

- 输入 `[10,20,30]` 及中文标签。
- 正常 stdout、stderr 被分别捕获；exit_code=0。
- JSON 产物 `count=3`、`sum=60`、中文标签完整。
- 故意失败 exit_code=7，stderr 正确。
- finally `destroy()`，查询实例返回 404。
- 最终检查没有遗留运行实例。
- 报告 passed=true，验证进程 exit=0。

证据：

- 失败诊断 1：`outputs/sandbox-check-20261028-185316/report.json`。
- 失败诊断 2：`outputs/sandbox-check-20261028-185457/report.json`。
- 最终报告：`outputs/sandbox-check-20261028-185511/report.json`。
- 下载结果：`outputs/sandbox-check-20261028-185511/result.json`。

当前成功组合：`opensandbox==1.2.0`＋`code-interpreter:v1.1.0`＋`use_server_proxy=False`＋固定 argv 使用 `shlex.join` 生成引用正确的命令字符串。

不要把它写成允许任意 shell。未来工具接口保持结构化参数，生产适配另行设计。临时脚本输出很小，没有验证传输阶段输出限制。

API 文档返回 `info.version=0.1.0`，不能当作 Server 软件包版本。代理失败根因未彻底定位，不能宣称所有代理部署都不能用。

## 6. 后续需要设计／验证的边界

- 当前 Server 的准确软件版本、网络限制和资源能力。
- 前后端栈、数据库、执行进程／队列和 LangGraph 持久化方案。
- 模型工具调用、MCP 认证和动态工具权限。
- Question Human／审批等待的持久化、幂等答复和重启恢复。
- 外部写调用可能已成功但响应丢失，不可盲目重放。
- Sandbox 文件不等于 MCP 能直接读的路径，跨环境文件需显式传递。
- 同名上传、Skill 版本与工作副本恢复，不能用错历史文件。
- 用户数据／凭据的访问、删除、保留及备份策略。
- 多用户越权测试、超时取消、清理失败、输出大小和网络策略验收。

## 7. 下一步

用户已认可需求整体方向；依根目录 `AGENTS.md` 进行技术设计评审、收口重要取舍，再进入实施计划和开发。整体认可不等于所有 D／O 均已批准。无需重新问是否采用网页、是否多用户、Skill 来源或是否需要 Todo／Question Human。

若用户之后明确要求继续开发，再依据最新批准范围执行。任何新会话都不应仅凭本文自行开始安装依赖、改配置或部署。

## 8. v1.0设计历史（已被v1.1对应内容替代）

- 主设计：`outputs/AgentDock-技术设计-v1.0.md`，覆盖技术栈、模块、数据库实体、接口、具体功能、6类流程／关系图和验收。
- UI：`outputs/AgentDock-UI与交互设计-v1.0.md`，页面字段、状态和权限；`outputs/AgentDock-UI设计预览.html` 是虚构数据的静态设计预览，不是真前端。
- 执行专题：`outputs/AgentDock-执行与恢复设计-v1.0.md`，队列、租约、工具账本、问答检查点及幂等恢复。
- 架构图：`outputs/design/architecture/agentdock-architecture.html`；结构、产物和浏览器自动检查已通过。图与UI有本地截图，非产品验收。
- 主稿推荐 Python3.12/FastAPI/LangGraph、React/TS/Vite/Ant Design、PostgreSQL17/SQLAlchemy/Alembic、独立单Worker和数据库持久队列，不Redis/Celery。均为设计建议，未安装。
- 特别评审 PostgreSQL新增部署、等待立即快照清理、MCP首版仅HTTP无认证/静态Bearer（OAuth未纳入默认实施）及工作台布局。
- 运行建议：global2/user1；active15min、总Run48h、排队1h；问题24h、审批10min；值均待评审。
- 成功终态也要在销毁实例前快照，失败取消记录完整性；无工作区变化沿用原working_snapshot_id，不能用空清单覆盖之前工作。
- Question ID是整组ID（最多3题），一次答复对应唯一resume_request；审批过期生成拒绝观察恢复，普通问题/总Run过期才结束expired。
- 任何终态若有unknown外部写仍阻塞新Run，专用核对入口不能忽略继续；旧worker/saver隔离仍是实施前必须验证的机制。
- 三份设计已独立审阅并修正上述一致性问题。方案未获得实施批准，不自动进入writing-plans或代码开发。

## 9. 当前v1.1修订与最新用户要求

当前入口改为 `outputs/AgentDock-技术设计-v1.1.md`；旧v1.0文件保留历史，不再作为技术栈依据。

用户明确指定：Vue3＋JavaScript、MongoDB；UI推荐Element Plus，设计已采纳。后端Python/FastAPI/LangGraph保留为建议，不使用React／TS业务源码／SQLAlchemy／PostgreSQL实现。

用户新增上下文压缩与记忆、多模态图片／文档理解；模型支持多模态，接入符合OpenAI。默认Chat Completions兼容基线，Responses／文件／原生压缩能力分开核验，不假定兼容服务具备全部官方功能。

- UI：`AgentDock-UI与交互设计-v1.1.md`、`AgentDock-UI设计预览-v1.1.html`。
- 执行：`AgentDock-执行与恢复设计-v1.1.md`，Mongo CAS＋必要事务，建议副本集；本机单节点非高可用。
- 存储：`AgentDock-文件存储与并发设计-v1.1.md`，不可变唯一file key，元数据版本CAS，不全局加文件锁；多节点推荐对象存储，实际迁移需复制校验。
- 记忆：`AgentDock-上下文与记忆设计-v1.1.md`，参考Deep Agents／Claude Code／OpenAI公开资料；原文保留，有界摘要＋工具外置＋回读。soft70%／hard85%相对扣输出预留后的输入预算，均D。
- 多模态：`AgentDock-多模态输入设计-v1.1.md`，真实image parts、PDF文字／扫描页、文档结构索引、来源与解析状态；处理格式和配额待评审。
- 架构图：`outputs/design/architecture-v1.1/agentdock-architecture.html`，不替换旧图文件。

仍未安装新产品依赖、部署MongoDB、接入真实模型／MCP或解析用户文件。此次修订不构成产品实施授权。需要继续对齐主稿和专题的集合字段、容量reservation与恢复边界；设计通过后再编写实施计划。

## 10. 实际Skill套件适配（最新）

用户提供`D:/Work/skills`，主入口`crowncad-modeling-assistant`；明确要求评估并完善通用方案，不得改成CAD专用方式。

已只读审阅8个入口、相关references、15个Python文件及两份requirements；未执行样本、环境探针、MCP、模型、OAuth或业务操作，未改源目录。样本清单与hash在`outputs/design/skill-assessment/inventory.json`。

结论与入口：`outputs/AgentDock-Skill适配评估与结论.md`。两个取证专题为`AgentDock-Skill运行依赖评估.md`、`AgentDock-Skill协议与交互评估.md`，描述补充前发现的缺口，非当前方案已运行结果。

通用补充：`outputs/AgentDock-通用Skill宿主能力设计-v1.1.md`。已同步主稿，将SkillBundle、RuntimeProfile、PortableWorkspace、AuthProvider/OAuth、ResourceImporter、ExternalViewPort、TaskScopedGrant、DomainStateRef及AssetCollection纳入设计；用结构化记录和少量适配，不写领域节点／Skill名分支。

此项覆盖之前“OAuth仅后续”的默认范围：实际样本需要正式OAuth和外部视图，未实现／未核验时必须标相关在线分支受阻。任务Grant只避免明确范围内重复审批，不能扩大权限；长期资料保存仍独立授权，不能把“效果可以”当正式归档。

真实条件仍待核验：原生依赖在Linux镜像可用性、认证／外部页面会话、实际MCP工具及资源、内部创建重试语义。源环境参考提本地凭据与执行器OAuth默认冲突，平台禁止降级读凭据。越出Bundle的MCP手册／历史证据只标资料依赖，未擅自读取其内容或假定不存在。

尤其create_document工具说明内部可能有限重试新建对象，与unknown不盲重放有兼容性冲突。平台外层不重发不保证服务端不重复创建；必须确认幂等、查询或可关闭危险重试，否则该自动写能力受阻。不能为样本放宽规则。

当前结论为“已静态评估并补充通用设计，尚未真实运行验证”，不是“这组Skill已经能完整运行”。源文件和外部模型没有被改变。

收尾阅读入口：`outputs/AgentDock-设计文档导航.md`。已记录当前方案、各专题和四项未验证条件。用户“继续”在此阶段指继续设计收尾，不据此开始产品开发。后续若明确要求实施，再进入计划。

## 11. 2026-10-29确认与实施计划（覆盖旧阶段描述）

用户对三个建议回复“符合预期”：首轮以实际套件的一个代表性完整工作流验收，保持通用；明确授权的目标范围内连续执行，删除／覆盖／公开发布／正式归档／范围扩展单独确认；首版单机多用户、Mongo副本集＋本地持久文件，保留StorageProvider。已写入需求U24～U26和主设计。

现进入实施计划阶段，尚未授权产品开发。计划：`docs/superpowers/plans/2026-10-29-agentdock-mvp.md`，四阶段14任务，全部未实施。入口已更新到设计文档导航。先读计划及对应专题，审阅计划并确定执行方式后再开发；不因旧文中“尚未进入计划”倒退，也不因设计认可自行安装或业务写入。

本次仅修改文档和项目阶段说明。未安装依赖，未部署Mongo，未执行Skill、模型、MCP或新Sandbox测试。已做计划接口／覆盖／本地链接检查，不能称未来用例已通过。

## 12. 暂停、回退与逐节点交付（最新，覆盖第11节开工倾向）

2026-10-29用户要求暂停并回退试启动内容，先给明确计划；每步代码完成后再按实际实现形成／统计人工验收点，交付需求完成、设计落实、Review和代码理解文档。用户还要求将这些协作规则写入AGENTS，已落实第10节。

现行计划：`outputs/AgentDock-逐节点实施与交付计划-v1.2.md`；模板：`outputs/AgentDock-节点交付报告模板.md`；回退证据：`outputs/AgentDock-研发暂停与回退记录-20261029.md`。原14任务技术计划保留参考，新增顶部覆盖说明。当前只改文档，用户未重新授权研发。

试启动backend／infra／.gitignore及下载工具、缓存、虚拟环境已移至`work/rollback-20261029/`；新增Python3.12已卸载且启动器消失。递归删除被自动审批拒绝，故可恢复移动而非永久删除；原.env和历史SDK验证环境、脚本／报告保留。没有任何产品节点被标完成，也未部署Mongo或执行新集成测试。

节点代码完成后输出具体报告，区分实现／自动验证／人工验收状态；人工清单按实际结果形成，不提前冻结。阶段另汇总。每节点交付后停下来，用户明确验收及启动下一节点才继续；不得套用执行技能的连续推进要求覆盖用户最新指示。

## 13. 本地Git仓库与中文小步提交（最新）

2026-10-29用户要求项目使用Git，并明确中文、小范围、可读可控提交。已在`D:/Projects/AgentDock`初始化本地Git仓库，初始分支main；新增.gitignore排除.env、work（含回退暂存）、data、依赖和构建产物，新增.gitattributes控制换行。AGENTS.md第11节写入中文提交、单一目的、明确暂存、自查差异、按主题首次纳管和不擅自推送规则。

当前未配置远程仓库，未推送；读取到Git作者姓名／邮箱均未配置，因此未伪造身份、未创建提交、未批量暂存现有文档。首次提交前需用户提供提交姓名和邮箱，配置只作用本仓库。不要误以为Git已初始化代表所有文件已有历史保护；当前现有资料仍待分批首次纳管。

产品研发仍暂停，Git初始化不等于N01开工授权。后续先完成用户明确要求的版本管理准备，不能借此恢复写业务代码。

## 14. 资料纳管与跨环境接续（最新）

用户明确要求需求、设计、实施等资料提交Git以便其他环境使用，并授权本次资料纳管推送。Git本地作者ldyit321、邮箱ldtyit321@gmail.com已配置，origin为https://github.com/ldyit321/AgentDock.git；远程main已核实包含初始规则提交6af920a，覆盖第13节未配置身份／远程的旧状态。

本次按主题首次提交现有资料；README为仓库入口，`outputs/AgentDock-新环境资料使用说明.md`说明相对阅读链接、本机路径证据、外部Skill和未纳管的配置／依赖／临时资产。外部Skill源目录不复制到仓库，临时验证脚本不随Git迁移，报告如实保留其复现边界。长历史文档作为主题基线保留完整，不机械拆残篇。

产品研发继续暂停，N01～N14尚无已验收节点。本次授权只针对资料版本管理和推送，不授权安装、业务开发或真实服务写操作。
