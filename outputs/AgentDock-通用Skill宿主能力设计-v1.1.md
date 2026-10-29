# AgentDock 通用 Skill 宿主能力补充设计

版本：v1.1评审补充｜2026-10-28。起因：用户要求用`D:/Work/skills`的真实套件评估通用方案，主入口为`crowncad-modeling-assistant`，并要求补缺口但不改成CAD专用系统。

本文扩展[主设计](AgentDock-技术设计-v1.1.md)的通用宿主契约；覆盖其中原先“OAuth后续、单Skill包、无外部视图”的不足。所有新增实现细节为D，实际能力均待验证。没有执行样本、登录、建模或修改源文件。

## 1. 结论与通用原则

原设计可以承载基础理解、脚本、文件、问答和工具循环，但不足以覆盖完整样本。缺口并非需要一套CAD专用Agent，而是宿主尚缺依赖包、原生环境、OAuth、外部视图、跨环境资源和长期资料发布等能力。

保持一个LangGraph控制循环、一套工具守卫和持久账本。领域的建模顺序、单位、几何判断、查询参数和完成标准由Skill指导及工具响应表达，不写进平台节点、路由或数据库固定字段。

新增“Registry／Port／Profile”均为少量结构化记录、仓储与适配函数，不建立动态插件执行框架，不自动导入用户Python进入主进程。具体MCP工具名、连接和参数映射通过配置注册，不在业务里出现`if skill_name == crowncad...`。

## 2. SkillBundle：套件与依赖闭包

### 2.1 两种包

继续接受单根SKILL.md目录；同时支持多目录Bundle。Bundle可以由管理员本地导入、用户ZIP上传；本次仅评估目录，不自动导入。

Bundle记录bundle_id、owner/scope、entry_skill、成员name/version/hash、依赖边、资源索引、可选runtime需求和版本。可选manifest显式声明依赖，纯Markdown包仍能读取；相对链接静态发现只是候选依赖，不自动授予权限或推断全部动态引用。

### 2.2 安全解析

- ZIP条目禁止路径越界；Markdown中的`../兄弟Skill/`可以在同一获准Bundle内解析。二者不是同一种路径行为。
- `resolve_skill_reference(origin_version, reference)`先规范路径，再检查仍在Bundle闭包中及目标有权限；不能据此访问D盘其他目录。
- 所有成员固定版本和hash，命名空间为owner/bundle/name，私有同名Skill不劫持公共依赖。
- 分支按需读取，已读内容去重；循环引用不递归无限加载。依赖图有环可以呈现为相互参考，加载器用visited和深度预算停止，不把任何环都判为不可安装。
- 越出Bundle的服务README、历史generated证据或项目AGENTS引用，标明外部资料依赖，由用户／管理员显式提供；不能虚构接口手册，也不能自动读宿主相对目标。

### 2.3 用户限定范围

默认选择主入口时，UI展示其可用依赖集合并要求范围已经覆盖这些成员；模型有效范围是服务端计算的获准闭包。用户可以关闭依赖扩展并严格只允许选定成员；缺依赖时说明受阻分支，不能因为主Skill写“调用另一个”就扩大授权。

本样本应作为8个有关系的成员导入，不修改其源文案；安装记录保存解析出的声明与人工补全信息，不能把补全变成伪造作者声明。

## 3. RuntimeProfile：可复现执行环境

管理员登记：profile_id/revision、image digest、OS/arch、Python版本/ABI、获准包及lock、wheelhouse、系统动态库、字体、渲染模式、资源限额、网络策略、命令范围、探针结果与验证时间。

Skill声明能力需求，平台选择用户获准且匹配的profile。requirements不自动决定镜像、开网或提升资源，也不允许模型任意换Python版本。精确依赖不兼容时报告缺口，只有获得批准的兼容方案才可替换，不静默放宽pin。

MVP保留默认Python profile，并能配置额外原生库／科学计算／文档转换profile；这是同一执行适配，不是写CAD执行器。脚本使用独立子进程，固定cwd、解释器和该脚本导入根，避免多个`import scripts`互相污染；不在Web进程import上传代码。

脚本内部没有timeout时，外层仍限制整个进程树。原生库崩溃、超内存、中文字体缺失、输出图片不正确要分别验收。`.venv`／site-packages不进工作快照，依赖按profile重建。

样本的Windows/Python3.14.7声明不是Linux Sandbox的实测。不能为了兼容而回到Windows宿主直接执行不可信脚本，也不能先承诺依赖都可安装。

## 4. 虚拟工程根与领域状态

Sandbox物化PortableWorkspace，包含逻辑工程根、bundle只读源副本／工作副本、输入、临时输出、任务文件和已授权资料。布局可配置，例如`.agents/skills`与`generated`，但该名称只是本样本的一种布局，不成为平台唯一目录标准。

脚本参数和相对导入通过虚拟根解析；不同Run隔离，实际字节仍由StorageProvider及快照保存，不挂载整个D盘。源Skill文件作为只读源版本，若在容器中被改写也不能回写宿主源区；发布服务根据允许写入范围校验差异。

DomainStateRef记录domain_task_id、schema_id/version、文件集合、hash、工作快照、证据和外部对象引用。这不是平台理解全部领域字段，也不是又一份Todo或Graph checkpoint。JSON/Markdown等状态文件按Skill在工作副本维护，保存时有版本；平台保存来源与访问能力，不能仅凭文件内`approved=true`授予权限。

平台run_id、Skill内任务UUID、外部taskId和资源revision分开关联。摘要保留状态入口，按需回读权威文件；用户回答后领域分析仍需复核，不能将“收到回答”直接当成问题已解决。

## 5. 认证提供者与OAuth宿主

### 5.1 认证类型

无认证、静态Bearer继续支持；增加标准远程MCP OAuth授权能力作为计划内宿主能力。不是所有连接都必须OAuth，也不为具体CAD登录写一套旁路凭据逻辑。

AuthProvider统一提供begin_authorization、handle_callback、get_access_token、refresh、revoke及状态；由协议库和固定版本实现，Skill不能自行拼授权端点或访问本地凭据。

OAuth使用服务端关联状态、防伪state、PKCE、允许回调、资源/受众和scope校验；发现、注册、刷新支持矩阵需在实际服务器核验。Token密文按user＋connection＋issuer／audience管理，refresh和连接revision原子更新。403业务权限错误不自动重走登录，401根据协议有界刷新／引导授权，不无限循环。

### 5.2 人机交接

工具返回结构化auth_required，Run进入waiting_user并关联interaction.kind=connection_authorization；网页打开正式授权入口，用户自行登录。回调由服务端验证并提交连接状态；只有真实授权结果和复检通过才能恢复，不把问题卡的“我已登录”当作协议成功。

认证跳转、code、Token不写聊天、summary、checkpoint和日志。模型只看连接状态、获准能力和不含秘密的阻塞原因。Skill要求OAuth而宿主未完成支持时，标明“可离线分析，在线执行待授权能力”，不退回偷读凭据。

MCP授权身份和外部业务网页登录分别管理；一个成功不证明另一个成功。

## 6. CapabilityRegistry与外部资源绑定

Registry保存获准连接／工具的实际name、JSON schema hash、auth状态、effect分类、query成本提示、resource返回类型、内部重试语义、支持取消／幂等／结果查询的能力及验证范围。

Skill通过文字／可选声明提出需求，实际工具目录是调用签名依据；不按样本文档发明不存在的`health`或授权工具。安装可标“依赖齐全”，初始化可标“协议连接成功”，业务授权和写入有效性需另外证据。

外部对象用ExternalResourceBinding统一保存连接、账号、资源类型、真实ID、版本／revision／etag、采集时间、来源调用及有效性。字段可以是领域定义的结构化map，但权限相关抽取由已批准工具映射配置，不从模型任意JSON解释出新授权。

外部写后使相关证据和缓存标stale，后续需要时重新读取。若服务器资源revision未改变也可能内部模型已变，必须按服务契约记录采集时点与状态范围，不能仅凭相同ID或文件路径宣称证据仍新鲜。

## 7. ResourceImporter：工具返回资源真正落地

支持标准MCP resource、嵌入图像／文本和已登记工具的结构化资源字段；业务字段映射由适配配置确定，不假定所有服务器返回同一artifacts协议。

导入流程：来源工具调用／用户绑定→提取opaque ResourceHandle→验证允许的下载目标、期限及大小→服务端限流下载→核对hash/byteSize→安全解包（若需）→保存不可变files记录→给模型和Sandbox内部file_id。

签名URL只在服务端短期受保护保存，图和摘要保留opaque handle与已导入file_id，不持久化带凭据URL。服务器路径不是本地路径；MCP给出的URL也要通过SSRF及重定向检查，内网目标必须精确批准。

错误证据包存在不代表调用成功；原始isError、exit或业务错误继续保留。包内脚本标为证据，不自动执行；如真正需要执行其中代码，要作为新受控任务另判权限和来源。

资源下载失败／过期不自动重放导出／创建等写操作。若有独立续期／状态工具则按其已验证能力处理；没有则保留taskId和“证据未取得”，不能虚构续签。只读重新采集也必须标新时间，不回填已变化的旧阶段。

工具参数需要大脚本／文本时，可由通用参数物化服务从授权file_id读取并填入指定JSON参数位置，先限制大小、校验hash、计算审批指纹再调用。不是将脚本改成本地Python执行，也不是让模型用文件路径代替工具实际需要的字符串。

## 8. ExternalViewPort：展示与真实观察

MVP增加通用视图适配接口，允许通过已注册MCP／受信浏览器连接提供能力，不把Codex当前桌面工具当作AgentDock产品天然具备的功能。

| 能力 | 返回与约束 |
|---|---|
| open_target | 使用获准工具产生的目标入口，或展示用户点击链接；不拼接未知工作区地址 |
| bind_view | 明确目标资源、页面／窗口会话和用户，不取第一个窗口碰运气 |
| get_view_state | 读取可见目标、加载与渲染状态，状态不可知就标未验证 |
| capture_view | 由获授权的工具获取画面，导入file_id并关联资源／revision／时点 |
| control_view | 已注册的视角／缩放等操作，检查范围与授权，不把接口存在当用户授权 |
| interact | 可能改变页面／业务数据的动作单独审批或按有效范围授权，不能因截图失败自动启用 |

外部网站不一定允许iframe，默认支持合法新标签页与截图展示；用户浏览器Cookie不自动等于服务端浏览器会话。若仅用户手动打开而Agent没有实际画面，只能报告“用户可查看”，不能声称Agent已观察。

截图经ResourceImporter入库后走多模态真实image parts。EvidenceRef区分参考渲染、外部当前视图、工具测量和人工确认；画面发送成功、下载成功与模型实际查看是不同事件。DomainStateRef保存这项证据检验了什么，不以截图替代数值／连通等领域校验。

没有可用的视图适配或覆盖动作的授权时，不越权访问宿主浏览器；继续独立分析，相关最终验收保持未验证。

## 9. 任务范围授权与危险动作

逐操作Approval继续作为默认；增加用户可明确授予的TaskScopedGrant，避免连续任务每个普通步骤都重新问。Grant必须由可信UI根据用户选择建立，Skill／模型不能自行创建。

Grant包含user、Run/逻辑任务、connection/auth主体、资源选择器、允许工具和参数边界、时间／次数／资源额度、禁止项、是否允许创建子资源、政策revision及撤销条件。每次动作守卫判断是否覆盖，记录实际参数hash和所用Grant；越界、账号变化、删除／覆盖、公共发布或用户要求逐次批准时重新审批。

不能仅凭自然语言“帮我完成”推导无限权限。若远程脚本参数内部能绕过上层资源约束、没有可验证资源边界，就不能宣称Grant可强制限制内部影响；需要服务器自身权限或退回逐阶段具体代码hash审批。平台不通过编写CAD语言解析器解决所有脚本副作用。

unknown阻塞不只存在于conversation字段：工具账本产生未核对外部写时，同事务建立服务端operation_blocks。MVP保守按owner阻塞该用户通过所有受管外部连接继续写入；共享连接再阻塞该connection供其他用户的后续写，仅读取不受此写屏障影响。每次托管外部写入口都检查，换对话／新连接／复制文件不能清除本人屏障。解除只由核对服务针对原action处理，文件里的domain_task_id和“已核对”文字不生效。更细的账号／资源粒度可后续优化，无法验证连接身份时不承诺跨账号推断。

正式长期归档与“效果可以”是不同授权；工作快照正常保存与发布到长期资料库也不同。审批仍绑定当前源版本和目标范围，复制接续包不继承旧OAuth和已消费批准。

## 10. 工具内部重试的兼容性门槛

工具登记internal_retry_semantics、idempotency_support、result_lookup、side_effect_bound和认证主体；这些需要可信配置和验证，不能仅信任工具自己说安全。

本平台外层不重放写入，不证明远程工具内部没有重复创建。对于内部超时后可能重建多个对象的工具：必须确认可关闭该行为、提供可靠幂等／查询协议，或明确作为不满足自动执行要求的连接能力阻塞。不能为样本而把unknown当失败重发。

用户可以知晓并批准某个明确多对象操作，但这不自动修复未知结果的幂等性。若服务器无法提供需要的语义，设计只能明确限制该能力，不能靠文档宣称已经兼容。

## 11. 用户长期资料集合

新增AssetCollection作为用户主动保存的资料库，不是模型自动长期记忆。字段owner/scope、collection_id、revision、manifest_ref、permissions；版本内容不可变，默认私人，公共发布由管理员另授权。

运行时把获准资料快照导入虚拟工作根。脚本在Sandbox内执行查找、生成或重建索引；其本地文件锁仅约束本次副本，不是宿主或对象存储的发布锁。

正式发布：用户授权→固定源快照与目标集合→在staging生成内容和索引→校验文件、来源、容量与差异→StorageProvider存新对象→Mongo按expected_collection_revision CAS切换manifest。冲突保留候选供比较，不覆盖他人更新。内容已保存而索引失败要分别记录，不用统一exit非零推断“没有任何副作用”后盲重试。

共享资料只读导入，不把脚本本地更新同步回公共源。跨对话仅支持显式选择已发布、获准的AssetCollection资料，或重新上传原件；首版不提供未完成Run的跨对话恢复／接续包重放。资料来自本平台历史时由服务端保留source_run/action来源，不从用户文件相信一个origin_id。导入是新任务输入，不继承旧OAuth、Grant或审批；operation_blocks写屏障仍按服务端账本生效，另起会话不等于核对完成。

## 12. 数据与UI补充

| 新增对象／集合 | 关键字段 |
|---|---|
| skill_bundles / bundle_versions | owner/scope、entry、成员版本／hash、依赖图、layout、发布状态 |
| runtime_profiles | revision、image digest、ABI／依赖／字体／资源与验证范围 |
| capability_bindings | connection/tool/schema hash、认证／资源／视图／重试能力映射及可信来源 |
| authorization_flows | owner、connection、state hash、PKCE受保护引用、callback、expiry、状态；不入图 |
| execution_grants | owner/task/Run、目标／工具／参数范围、额度、期限、policy revision、撤销 |
| external_resource_bindings | 账号／连接、资源类型与ID、version、时点、来源及有效状态 |
| imported_resources | 来源call、opaque handle、expected hash/size、file_id、完整性与错误；秘密URL单独短期保护 |
| domain_state_refs | 领域任务、schema、文件／hash、来源快照、外部引用与证据 |
| asset_collections / asset_versions | owner/scope、revision、manifest、发布授权和索引状态 |
| operation_blocks | owner／shared connection、原tool_call、原因、状态及核对记录；服务端维护，普通文件不能清除 |

UI增加Bundle成员／依赖范围、运行环境可用性、连接OAuth登录状态、外部目标／视图与证据、任务范围授权和私人资料库入口。均复用现有列表、详情、问题／审批和文件组件，不建CAD专用页面。用户能看到“缺少什么、哪个分支受阻”，不只得到一条脚本错误。

## 13. 代表性验收与非特化约束

1. 同一Bundle跨Skill链接按需读取，恶意越包链接被拒，缺依赖不虚构工具。
2. 不同CLI的scripts包不污染；原生库／字体在实际Linux profile探针通过后才标可用。
3. OAuth授权／刷新／撤销与外部网页登录分别验证，多用户连接隔离。
4. 资源URL过期、错误hash、错误包、ZIP逃逸和私网重定向按预期处理，不重放写操作。
5. 外部截图真实进入视觉请求，旧画面或参考图不能冒充当前对象验收。
6. 范围授权连续执行不反复确认，越界／公共发布／凭据变化会重新确认；未知写继续阻塞。
7. 文件快照、领域状态、图checkpoint、远端对象状态各自恢复，不互相冒充。
8. 长期资料发布并发CAS、索引部分失败与用户授权独立核对。

通用性反证要求：用至少一个非CAD套件复用这些机制，例如文档转换套件（原生依赖＋多Skill）和内容发布套件（OAuth＋外部写＋资源导入）。平台图和核心服务中不出现领域Skill名分支；样本差异仅存在于Skill内容、配置、运行profile及验收数据中。

完整在线路径仍需真实MCP、授权、视图适配和镜像验收。本文完成设计闭包，不声称外部服务器的内部重试、Linux依赖或工具签名已经验证。
