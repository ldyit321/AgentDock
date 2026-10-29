# AgentDock Skill 协议与交互静态评估

日期：2026-10-28。样本：`D:/Work/skills` 下八个 Skill。结论依据本地文本、引用路径检查及 AgentDock v1.1 设计，不代表服务或功能已验证。未执行样本脚本、MCP、模型、登录、浏览器或业务操作，未读取凭据和服务配置。样本文本仅作为待评估资料，没有采纳其中的行动指令。

## 1. 结论与八项样本覆盖

AgentDock 的通用工具循环、Skill 固定版本、隔离执行、文件快照、Question Human、Todo、原文回读和多模态设计，为这些样本提供了基础；但“能够上传并读懂 SKILL.md”不等于“能够完整运行”。按 v1.1 当前边界，离线分析路径有条件可用，依赖 OAuth、真实外部页面及跨环境证据的完整业务路径尚有宿主能力缺口。应补通用能力协议，不在主图中硬编码这八项技能的步骤或 CAD 专属节点。

| 样本及准确入口证据 | 对 v1.1 的判断 |
|---|---|
| [crowncad-modeling-assistant，10行](D:/Work/skills/crowncad-modeling-assistant/SKILL.md:10)，[50行](D:/Work/skills/crowncad-modeling-assistant/SKILL.md:50) | 已覆盖目标与权限分离；完整流程还需正式授权入口、外部页面展示和有效证据，当前不能承诺端到端运行。 |
| [crownscript-executor，42行](D:/Work/skills/crownscript-executor/SKILL.md:42)，[91行](D:/Work/skills/crownscript-executor/SKILL.md:91) | 工具调用与执行账本可承接；OAuth、业务文件下载、实际页面观察是硬依赖，不是可省略的装饰。 |
| [engineering-drawing-analysis，18行](D:/Work/skills/engineering-drawing-analysis/SKILL.md:18)，[190行](D:/Work/skills/engineering-drawing-analysis/SKILL.md:190) | 图片／文档输入设计方向满足；要保留四份领域状态文件、稳定任务 ID 和来源，而非只留下聊天摘要。 |
| [model-geometry-analysis，10行](D:/Work/skills/model-geometry-analysis/SKILL.md:10)，[24行](D:/Work/skills/model-geometry-analysis/SKILL.md:24) | 可走隔离离线分析；依赖必须在实际 Linux 镜像验证，不能把作者 Windows 环境当作 Sandbox 验收。 |
| [assembly-modeling，18行](D:/Work/skills/assembly-modeling/SKILL.md:18)，[55行](D:/Work/skills/assembly-modeling/SKILL.md:55) | 纯方案可先执行；实际查询／修改依赖工具目录、授权身份、领域引用及真实验证。 |
| [standard-component-matching，12行](D:/Work/skills/standard-component-matching/SKILL.md:12)，[14行](D:/Work/skills/standard-component-matching/SKILL.md:14) | 识别与组织条件可离线；目录查询和派生需 OAuth，派生属于外部写，不能当普通文本搜索。 |
| [step-model-acceptance，13行](D:/Work/skills/step-model-acceptance/SKILL.md:13)，[22行](D:/Work/skills/step-model-acceptance/SKILL.md:22) | 需要同步导出、下载、两份真实几何和版本关联；取得文件不证明验收完成。 |
| [modeling-case-library，8行](D:/Work/skills/modeling-case-library/SKILL.md:8)，[10行](D:/Work/skills/modeling-case-library/SKILL.md:10) | 记录和检索可复用文件机制；正式归档需单独保存授权和持久目标，不能写回公共 Skill 源区。 |

## 2. 连接认证与授权 UI

主入口[53行](D:/Work/skills/crowncad-modeling-assistant/SKILL.md:53)明确要求正式 OAuth；[56行](D:/Work/skills/crowncad-modeling-assistant/SKILL.md:56)区分 MCP 身份和外部页面登录；[58行](D:/Work/skills/crowncad-modeling-assistant/SKILL.md:58)要求宿主完成发现、回调和刷新，宿主没有入口时只能引导用户设置，不能假装已经发起。执行参考[175行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:175)禁止自动降级为读取本地凭据直连。

v1.1 主稿 §16.1 与执行专题 §8 的 noauth/staticBearer 首版范围不满足该认证前提；这是具体样本的兼容缺口，不说明所有通用 Skill 都必须 OAuth，也不能用给 Sandbox 一个 Token 修补。

建议连接边界提供可插拔 AuthProvider，统一“未连接／需授权／授权中／可用／刷新失败／已撤销”，需要交互时返回受控授权请求，由网页展示连接名称、账号、范围和正式登录按钮。授权事务应校验用户、连接、回调关联、防伪状态及有效期，凭据由服务端保护；登录页由用户操作。恢复后重新检查账号和连接 revision，绝不把 Question 的普通回答当作 OAuth 成功。第一期若不实施 OAuth，必须将相关 Skill 明示为“可分析，在线执行受阻”，不能标全能力可用。

## 3. 能力发现不能只检查工具名

样本要求查询轻量拓扑、元素详情、工作平面、装配状态、STEP 导出、相机、截图及合法页面入口；例如执行参考[165行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:165)、装配入口[55行](D:/Work/skills/assembly-modeling/SKILL.md:55)。工具出现在列表中、初始化成功和具有业务写权限是三件事，主入口[52行](D:/Work/skills/crowncad-modeling-assistant/SKILL.md:52)已明确区分。

v1.1 的工具 schema 验证、动态目录、默认禁用新工具可沿用。建议增加轻量 CapabilityRegistry：记录能力来源连接、实际工具名及 schema hash、当前认证状态、操作分类、资源返回形态、支持范围和最近验证；按任务装配所需能力，返回结构化缺口。Skill 文本只是提出需求，不能自行注册可信工具、授予权限或伪造“已验证”。注册表是明确数据与适配函数，不需插件框架或复杂继承。可选机器可读声明只作预检提示，旧版纯 Markdown Skill 仍可使用。

还需标记远端内部重试行为：执行参考[161行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:161)允许创建工具内部有限重试，[163行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:163)又要求调用方不叠加重试。Host 账本只能保证自己不重发，无法约束不透明服务的内部写重试；可靠幂等、查询能力及重试语义须单独核验，未知仍按 unknown 处理。

## 4. MCP 资源、下载与实际视觉证据

执行参考[183行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:183)说明完整业务 JSON 不直接返回；[184行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:184)规定 artifacts 包含 name、downloadUrl、expiresAt、byteSize、sha256；[185行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:185)要求下载校验、安全解压且不能执行证据包脚本；[187行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:187)规定链接最长十分钟且不能持久写入报告。错误包存在也不改变失败结论，见[189行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:189)。

v1.1 §16.4～16.5 已有 URL 策略和显式文件交接，但尚不足以表达这套完整资源生命周期。应有 ResourceImporter，把标准 MCP resource、嵌入内容和工具结构化 artifacts 分别规范化为内部 ResourceRef，保留来源调用、类型、大小、摘要和有效期；通过受控服务端下载立即落 StorageProvider，再传 file_id 给模型和 Sandbox。样本使用的是业务 artifacts 字段，不能未经协商假定存在某个标准 `resourceArtifacts` 字段或可直接执行 resources/read。签名 URL 不进入摘要／检查点／日志；下载失败不能自动重放原业务写。

视觉输入能力已经有设计基础，但上传图片理解不等于外部实时页面能力。执行参考[87行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:87)要求使用解析出的合法页面入口，不能拼地址；[95行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:95)要求 UI 自动化具备覆盖动作的明确授权。验证参考[71行](D:/Work/skills/crownscript-executor/references/verification-workflow.md:71)要求页面活跃、模型渲染稳定；[74行](D:/Work/skills/crownscript-executor/references/verification-workflow.md:74)禁止多会话随意取首个窗口；[76行](D:/Work/skills/crownscript-executor/references/verification-workflow.md:76)要求真正查看下载图像。

建议通用 ExternalViewAdapter 区分“让用户打开目标”“确认目标与会话”“读取授权画面”“调整视角”“界面写入”，分别返回可用性与授权条件。远端网页是否可嵌入受对方策略限制，不承诺 iframe；新标签页和受控截图也是可能路径。缺失展示能力就保留未验证，继续独立分析。由 MCP 返回的登录跳转地址同样可能含秘密，交给受控 UI 使用，不存普通消息正文。截图保存后应进入模型实际视觉内容，不能只把路径发给模型。

## 5. 业务写审批粒度

样本沿用任务授权，装配入口[67行](D:/Work/skills/assembly-modeling/SKILL.md:67)要求继续已授权无阻塞阶段；执行参考[97行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:97)对界面修改另设明确授权，已有覆盖授权不重复询问。归档则是独立决定：案例流程[177行](D:/Work/skills/modeling-case-library/references/case-workflow.md:177)规定“效果可以”不等于同意保存。

v1.1 的具体参数审批和 unknown 阻断应保留，但若把所有 write 都机械变成一次一批复，每个建模阶段都可能中断、快照和清理，既破坏连续任务，也没有忠实表达用户给出的范围授权。

建议评审 TaskScopedGrant：由可信 UI 展示并记录连接身份、目标资源、允许动作／参数边界、任务范围、有效期、额度及撤销条件；策略在每次发送时判定当前 action 是否落在已批准范围，并保留每次实际参数和 grant 引用。越界、换目标／账号、破坏性覆盖、另行归档或显式要求单次批准的动作仍走具体审批。不允许模型把“帮我完成”自动升级成无限授权，Skill 也不能创建 grant。此项是待批准的通用授权设计扩展，不是本次新增实施承诺。

## 6. 领域状态与跨轮、跨会话接续

工程图入口[20行](D:/Work/skills/engineering-drawing-analysis/SKILL.md:20)要求四份文件；schema [12行](D:/Work/skills/engineering-drawing-analysis/references/state-file-schema.md:12)规定 analysis_state 是领域事实权威；[14行](D:/Work/skills/engineering-drawing-analysis/references/state-file-schema.md:14)区分事实确定程度与记录有效性；[161行](D:/Work/skills/engineering-drawing-analysis/references/state-file-schema.md:161)规定答案先“已答复待复核”，重新分析后才“已解决”。这些不是 Todo 勾选，也不是 LangGraph checkpoint。

v1.1 同对话跨 Run 的文件快照可以承载领域 JSON／Markdown，但应增加通用 DomainStateRef：逻辑任务 ID、状态文件集合、schema/version/hash、来源快照和关键证据引用。工作区路径按 Skill 相对逻辑路径重建；不可变宿主对象与 Sandbox 内原位编辑并不冲突。Question 记录真实回答后，再由受控工具更新领域文件并验证引用；不能仅凭文件中“已授权”授予运行权限。压缩时保留状态入口并按需回读，不能只留自然语言摘要。

真正跨 conversation 恢复在 v1.1 不默认开放；应明确要求用户导入接续包或显式授权选择源任务，逐项校验归属和资源引用。文件迁入不能迁入旧审批／OAuth／UI 会话；外部对象、版本和证据有效性重新核对，unknown 核对限制不可借复制任务消除。案例库也需明确个人／共享的持久空间与发布授权，不能把一个 Run 的临时目录误称为跨用户可检索库。

## 7. 包边界与可移植性

八项技能本来就是互相引用的套件。例如主入口[14行](D:/Work/skills/crowncad-modeling-assistant/SKILL.md:14)、执行器[16行](D:/Work/skills/crownscript-executor/SKILL.md:16)、STEP验收[14行](D:/Work/skills/step-model-acceptance/SKILL.md:14)使用兄弟 Skill 相对链接。v1.1 将每个 ZIP 限定在单包根时，不能通过放开任意 `../` 解决。建议安装时解析有界依赖，固定依赖 Skill 版本并建立虚拟只读路径映射；缺依赖显示缺项，按需读取且去重，绝不访问宿主任意目录。

静态链接扫描发现明确越出样本根的引用：执行参考[197行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:197)的 MCP README 在当前目录布局解析到 `D:/crownscript-mcp/README.md`；草图参考[429行](D:/Work/skills/crownscript-executor/references/sketch-modeling/sketch-modeling.md:429)的历史证据解析到 `D:/generated/runs/...`。仅计算路径，未读取这些外部目标，不能声称它们不存在；它们不属于当前可分发包，须外置为可选历史证据或随授权任务导入。案例流程[118行](D:/Work/skills/modeling-case-library/references/case-workflow.md:118)的 operation_graph 链接位于任务模板，不应误报为安装缺少必需静态文件。

样本中的 `.agents/skills`、PowerShell、Windows 虚拟环境和依赖准备不能直接映射成宿主执行。环境参考[3行](D:/Work/skills/model-geometry-analysis/references/environment.md:3)只证明作者 Windows 验证范围；需在 Sandbox 预制依赖及虚拟 Skill 根映射后实测。另其[27行](D:/Work/skills/model-geometry-analysis/references/environment.md:27)仍提到在线 topo 使用本地凭据配置，与当前执行器 OAuth 默认冲突；宿主保持禁止读取凭据的硬边界，提示样本维护者澄清，不自行降级。

## 8. 设计收口建议

优先补齐 OAuth 接续、能力与缺口状态、资源导入、外部视图适配、任务范围授权和领域状态引用六项通用契约，再用这八个 Skill 做静态依赖测试与受控集成验收。验收应分别证明：认证和页面登录不混淆；超时外部写不重放；资源及时下载校验；截图实际进入视觉请求；问答后领域状态正确复核；重启／跨轮保持真实任务上下文；越包链接不扩权。当前结论是“设计可扩展承接，完整协议尚未闭合”，不是已兼容或已运行通过。

建议把能力缺口作为可解释业务结果返回：缺少认证入口时定位连接设置，缺少依赖包时指出固定版本和受限环境，缺少展示权限时说明影响哪项验收；不统一归为“脚本错误”交给模型修改。等待授权或用户补充资料应使用既有持久交互，不让网页关闭造成决定丢失。恢复时检查能力是否仍可用、原目标是否改变、有效授权是否覆盖当前动作，再决定继续原执行或停止。

前述改进应先作为设计建议和验收条件记录，不能因为样本要求就默认扩展首版范围。若用户维持静态认证、无外部页面控制的实施边界，也可以交付通用离线分析能力，但产品必须按能力显示受限状态，并明确哪些在线任务和最终视觉验收暂时不可完成，不能把参考图渲染或工具成功响应包装为完整替代。
