# AgentDock Skill 运行依赖评估

日期：2026-10-28；对象：用户提供的 `D:/Work/skills` 八个 Skill，主入口 `crowncad-modeling-assistant`。方法：只读目录清单、Python源码、requirements与相关references，并对照AgentDock v1.1设计。没有执行任何样本脚本、pip、smoke check、网络、CAD操作或读取凭据，也未修改样本。本文是**静态满足性评估，不是集成实测通过**；下面新增机制为D建议，平台/性能/协议兼容为O。

结论：现有通用边界方向适用，但不能把八个目录原样上传后即宣称完整运行。关键缺口是跨Skill依赖闭包、原生运行环境、工程虚拟根、跨对话长期资料发布、远端证据导入以及工具内部重试策略。应以通用机制补齐，不为CAD增加业务硬编码。OAuth与浏览器入口另由主设计评估，本报告仅指出它们是独立外部状态。

## 1. 样本范围与脚本清单

只发现两个requirements文件。共15个Python文件，其中2个为`__init__.py`，13个为逻辑/命令脚本；“8个Skill”不等于“8个独立可执行程序”。源入口以自然语言路由专项能力，明确技能调用是读取指导，不默认创建子代理。证据：[主入口第12行](D:/Work/skills/crowncad-modeling-assistant/SKILL.md:12)、[跨技能路由第64行](D:/Work/skills/crowncad-modeling-assistant/SKILL.md:64)。

| Skill | 实际Python/依赖 | 参数与调用边界 |
|---|---|---|
| crowncad-modeling-assistant | 无Python、无requirements | 通过兄弟Skill指导和MCP推进，不是固定CLI入口；相关路径相对当前Skill解析，见入口14、66—78行 |
| assembly-modeling | 无Python、无requirements | 调用执行器、几何分析及assembly_verification资料比较；见[入口12行](D:/Work/skills/assembly-modeling/SKILL.md:12)、71行 |
| engineering-drawing-analysis | 无Python、无requirements | 需要图纸真实理解与四份结构化状态文件，不自带PDF解析/OCR可执行器；见[入口18行](D:/Work/skills/engineering-drawing-analysis/SKILL.md:18)、22—29行 |
| step-model-acceptance | 无Python、无requirements | 复用几何Skill，另经MCP同步导出并记录任务ID；见[入口14行](D:/Work/skills/step-model-acceptance/SKILL.md:14)、[流程15行](D:/Work/skills/step-model-acceptance/references/acceptance-workflow.md:15) |
| standard-component-matching | standard_component_tool.py；仅标准库 | `--input`可选，不给则stdin JSON；stdout JSON，invalid_request退出2，否则0；见[脚本183行](D:/Work/skills/standard-component-matching/scripts/standard_component_tool.py:183) |
| model-geometry-analysis | environment.py、render_step.py；3个固定重依赖 | 环境`check`或`setup --venv … [--wheelhouse …]`；渲染参数下述；见[environment.py 82行](D:/Work/skills/model-geometry-analysis/scripts/environment.py:82) |
| crownscript-executor | __init__、extract_topology、render_topology、geometry_measurements、verification、assembly_verification、sketch_plane、sketch_slot | 两个CLI，其余主要为供生成器导入的函数模块；Pillow仅预览依赖，不能把它当作CAD执行SDK |
| modeling-case-library | __init__、find_cases、save_case、model_case_library；仅标准库 | `search --workspace … --term …`（term可重复）、`rebuild-index --workspace …`；`save_case --workspace … --source … [--required-file …]`；见[find_cases 17行](D:/Work/skills/modeling-case-library/scripts/find_cases.py:17)、[save_case 23行](D:/Work/skills/modeling-case-library/scripts/save_case.py:23) |

### 1.1 实际imports

- 标准匹配脚本：argparse、json、re、sys、pathlib.Path、typing（Any/Dict/List/Mapping/Sequence），[源码5行](D:/Work/skills/standard-component-matching/scripts/standard_component_tool.py:5)。静态代码中没有网络下载依赖；目录查询/派生由MCP参考描述，不是该离线脚本完成。
- 几何environment：argparse、importlib.metadata、json、pathlib、subprocess、sys、tempfile、venv；check内延迟导入OCP的BRepPrimAPI/BRepMesh/BRep/TopExp/TopAbs/TopoDS/TopLoc，以及matplotlib、pyplot、numpy，[源码2行](D:/Work/skills/model-geometry-analysis/scripts/environment.py:2)、15行。
- render_step：argparse、Counter、hashlib、importlib.metadata、json、math、Path、sys；render内导入OCP的STEPControl/IFSelect/TopExp/TopAbs/TopoDS/BRepAdaptor/BRepMesh/BRep/TopLoc/BRepGProp/GProp，matplotlib.pyplot/MaxNLocator和mpl_toolkits.mplot3d.art3d.Poly3DCollection，[源码2行](D:/Work/skills/model-geometry-analysis/scripts/render_step.py:2)、23行。
- 执行器extract_topology：argparse/json/Counter/Path/sys，[源码4行](D:/Work/skills/crownscript-executor/scripts/extract_topology.py:4)。render_topology：argparse/json/math/sys/Mapping/Path/Any及同Skill的scripts.geometry_measurements，绘图时延迟导入PIL.Image/ImageDraw/ImageFont，[源码5行](D:/Work/skills/crownscript-executor/scripts/render_topology.py:5)、15、329行。
- geometry_measurements为math/Mapping/Any；verification为json/math/Mapping/Sequence/deepcopy/Path/Any；assembly_verification为json/math/deepcopy/Path/Any；sketch_plane为math/Mapping/Sequence；sketch_slot为math/Any/Sequence。证据分别见各[geometry_measurements 6行](D:/Work/skills/crownscript-executor/scripts/geometry_measurements.py:6)、[verification 6行](D:/Work/skills/crownscript-executor/scripts/verification.py:6)、[assembly_verification 9行](D:/Work/skills/crownscript-executor/scripts/assembly_verification.py:9)、[sketch_plane 9行](D:/Work/skills/crownscript-executor/scripts/sketch_plane.py:9)、[sketch_slot 13行](D:/Work/skills/crownscript-executor/scripts/sketch_slot.py:13)。这些库模块没有独立argparse CLI，不应凭文件名发明参数。
- 案例命令入口为argparse/json/sys/Path并导入scripts.model_case_library；公共模块依赖filecmp/hashlib/json/os/re/shutil/stat/tempfile/time/Iterator/Sequence/contextmanager/datetime/timezone/Path/PurePosixPath；锁函数按OS延迟导入msvcrt或fcntl，[公共模块13行](D:/Work/skills/modeling-case-library/scripts/model_case_library.py:13)、356、372行。无数据库、对象存储或跨用户授权实现。

这些是直接imports，不是完整传递依赖清单；没有启动Python去解析/导入任意样本模块。

### 1.2 requirements与可执行参数

几何requirements声明：`cadquery-ocp==8.0.1.1.0`、`matplotlib==3.11.2`、`numpy==2.5.3`，注释仅声明Windows x64/Python3.14.7组合已验证，其他环境需smoke check；见[requirements第1行](D:/Work/skills/model-geometry-analysis/requirements.txt:1)。环境脚本允许Python3.12—3.14，[environment.py 13行](D:/Work/skills/model-geometry-analysis/scripts/environment.py:13)。这两者只描述样本宣称的范围，不是本次验证结论，也不证明AgentDock Linux Sandbox/Python3.12有相应wheel。

`render_step`必需`--input`、`--output-dir`、`--up-axis x|y|z`、`--length-unit`；可选`--deflection`（默认0.3）、可重复`--view elevation,azimuth`、`--bounds`六个数、`--mesh-edges`，[源码170行](D:/Work/skills/model-geometry-analysis/scripts/render_step.py:170)。单位只是标签，上方轴只是相机方向；脚本没有单位换算或通用几何等价判定。最多12视图，[源码105行](D:/Work/skills/model-geometry-analysis/scripts/render_step.py:105)；输出views.png/geometry.json/render_manifest.json，[源码145行](D:/Work/skills/model-geometry-analysis/scripts/render_step.py:145)。

执行器requirements仅`Pillow>=11,<13`，[依赖1行](D:/Work/skills/crownscript-executor/requirements.txt:1)。`render_topology --topology … --output … [--views overall front top right] [--mode wireframe|planar] [--title …] [--chord-error …]`；views支持其中多个，默认四个，chord-error默认0.05，[源码480行](D:/Work/skills/crownscript-executor/scripts/render_topology.py:480)。它绘制已保存拓扑，不是原生BRep模型验收。

`extract_topology file [--summary] [--scope element|object] [--kind …] [--id …] [--owner-id …] [--fields …]`，kind/id/owner-id可重复，fields为顶层字段列表；[源码116行](D:/Work/skills/crownscript-executor/scripts/extract_topology.py:116)。退出0只说明提取完成，仍须检查missingIds、missingFields、coverage和issues，见[分层说明42行](D:/Work/skills/crownscript-executor/references/entity-modeling/details/topology-layer-design.md:42)。

## 2. 环境与路径假设

### 2.1 重依赖不能等价为临时pip

environment.setup创建venv、选择Windows Scripts/python.exe或POSIX bin/python，并用`pip install --only-binary=:all: -r requirements`；wheelhouse分支加`--no-index --find-links`，之后运行check及pip freeze，[源码64行](D:/Work/skills/model-geometry-analysis/scripts/environment.py:64)。这些subprocess.run没有显式timeout；AgentDock必须从外层强制限制整个进程树，不能认为脚本自行控制安装时长。环境参考明确三个直接版本不是跨平台传递依赖锁，[环境参考25行](D:/Work/skills/model-geometry-analysis/references/environment.md:25)。

建议通用runtime profile由管理员预制并验证：image digest、OS/arch、Python ABI、依赖锁/离线wheel集合、系统库、字体、资源/命令上限、能力探针结果；Skill可声明需要的能力，服务端选择获准profile，不让Skill任意指定镜像。原生库、无头渲染和字体可由profile提供，不将OCP包安装进Windows应用环境。不为本样本写CAD专属执行分支；其他视频、科学计算、Office转换同用此机制。

现有主稿13.4仅约束requirements来源，尚不足以证明二进制兼容或可复现。受控动态pip只能是profile内有限补充，先静态拒绝VCS/任意索引/凭据URL/本地路径，锁定并限额；安装失败是环境错误，不让模型换版本无限修复。不能因样本文档提示安装就自动开网或提升权限。Matplotlib使用Agg，[render_step 34行](D:/Work/skills/model-geometry-analysis/scripts/render_step.py:34)；Pillow字体候选含Windows/Linux/macOS路径，[render_topology 293行](D:/Work/skills/crownscript-executor/scripts/render_topology.py:293)，Linux有字体候选不等于中文渲染已验收。

### 2.2 Skill bundle与工程虚拟根

主入口相对兄弟目录读执行器，[入口14行](D:/Work/skills/crowncad-modeling-assistant/SKILL.md:14)，STEP验收复用另一个Skill的render_step，[验收流程23行](D:/Work/skills/step-model-acceptance/references/acceptance-workflow.md:23)。复制单个根SKILL.md并把目录改名会破坏链接。建议可选SkillBundle manifest：entrySkill、各Skill固定版本/摘要及依赖、获准runtime能力；发布时检测缺依赖、循环、路径越界及同名冲突。用户限定Skill范围同样限制依赖，不能依赖闭包自动扩权；不满足则说明缺失并停相应分支。

脚本多处使用泛名`scripts`包和sys.path.insert，如[render_topology 14行](D:/Work/skills/crownscript-executor/scripts/render_topology.py:14)、[find_cases 8行](D:/Work/skills/modeling-case-library/scripts/find_cases.py:8)。单进程同时把多个Skill根加入sys.path会存在同名包污染。通用执行器每次独立Sandbox子进程、固定cwd/解释器/导入根，不在API进程import上传脚本，不把全部Skill根同时加入全局PYTHONPATH。

样本命令假定项目根下`.agents/skills/<name>`和`generated/runs/models/drawing-analysis`。现有主设计仅`/workspace/input/skills/work/output`分区，不能直接满足这些相对路径。建议Sandbox内物化一个可移植工程虚拟根：只读bundle布局保留`.agents/skills`兄弟关系；本Run工作副本映射generated等逻辑路径；file_id/key与逻辑路径的清单负责恢复。虚拟根不是D盘宿主挂载，不给脚本真实工程或共享凭据目录。示例中的`D:/parts/...`只当待映射输入，不自动获准读取宿主文件，见[STEP示例8行](D:/Work/skills/model-geometry-analysis/references/step-analysis.md:8)。

图纸状态也依赖仓库相对路径和独立analysis_task_id，见[状态规范13行](D:/Work/skills/engineering-drawing-analysis/references/state-file-schema.md:13)；Run ID、Skill内部任务UUID、远端taskId和文档版本是不同标识，必须保留关联，不能互相替代。

## 3. 长期资料：只读共享与个人发布

案例库不是Skill源文件，也不只是本对话短期快照。`--workspace`必须是真实工程根，程序硬推`generated/models`，[公共模块56行](D:/Work/skills/modeling-case-library/scripts/model_case_library.py:56)；源必须来自工程`generated/runs`，[源码108行](D:/Work/skills/modeling-case-library/scripts/model_case_library.py:108)。单次Sandbox销毁会丢失未导出的库更新；恢复一个对话快照也不自动构成跨对话资料库。

脚本保存先建立.pending目录、复制、rename转正，[源码178行](D:/Work/skills/modeling-case-library/scripts/model_case_library.py:178)，索引以临时文件+os.replace发布，[源码312行](D:/Work/skills/modeling-case-library/scripts/model_case_library.py:312)；写锁Windows用msvcrt、POSIX用flock，最多15秒，[源码344行](D:/Work/skills/modeling-case-library/scripts/model_case_library.py:344)。这是单文件系统实现，不能把锁文件搬到S3后宣称跨节点互斥。

建议增加通用“用户资料集合/长期库”边界，不引入自动记忆或向量库：公共授权资料作为只读版本快照，私有库按owner管理；读取时选择有权限版本导入Sandbox。写操作先在本Run staging生成不可变集合快照，校验manifest、大小、来源与审批，再由可信服务以expected_collection_version CAS发布新指针/索引，保持内容发布与索引状态可独立解释。脚本本地锁只约束staging副本，不能充当服务端发布权限。

普通search明确只读，即索引过期也不写；显式rebuild-index才更新，[find_cases 27行](D:/Work/skills/modeling-case-library/scripts/find_cases.py:27)。readonly共享库可供检索，不允许脚本自动重建或写入；获准的修复索引作为独立服务动作。公共Skill不能因此继承他人的私有案例。长期库从一开始默认本人私有，公开发布需要另行权限策略，不能因目录名models即变公共。

save_case脚本明确授权及CAD完成检查由调用方负责，[save_case 3行](D:/Work/skills/modeling-case-library/scripts/save_case.py:3)。必须由AgentDock审批/发布服务绑定具体源快照、目标库和expected version；README写有“用户同意”不是可信授权。脚本返回saved但indexStatus=failed时退出2，参数错误也可退出2，[save_case 19行](D:/Work/skills/modeling-case-library/scripts/save_case.py:19)、32行；不能只按非零退出码宣布“没有保存”，盲重试可能覆盖事实。内容成功与索引失败应分别记账。

## 4. 远端证据与查询深度

样本明确MCP服务器路径不能作为调用端本地路径：返回artifacts含downloadUrl/expiresAt/byteSize/sha256，调用方下载校验后再分析；ZIP解压禁止越界和链接，包内脚本仅证据不能执行，见[交付流程183行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:183)。此能力目前不能只用“把MCP返回文字交给模型”满足。

建议通用受控资源导入服务：由当前owner/connection登记一次下载意图、验证地址/域名/重定向/DNS及大小限额，短时签名URL只在服务端临时使用；流式下载核对摘要后转为内部file_id/key，再复制到Sandbox。URL不写graph、摘要、永久日志或长期库，不原样作为公开artifact链接；业务产物下载继续走AgentDock授权接口。远端返回的serverPath只记录受控来源信息，不当作本机可读文件。签名地址携带访问权，不能仅靠“URL来自MCP”省掉地址策略。

下载URL最长10分钟且服务重启可失效，见[执行流程187行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:187)；截图签名资源同有限期，见[验证流程75行](D:/Work/skills/crownscript-executor/references/verification-workflow.md:75)。应先获取原始证据再进入长等待，过期不伪造续签或重发写操作；保存export taskId并核对已有任务，[STEP验收17行](D:/Work/skills/step-model-acceptance/references/acceptance-workflow.md:17)。摘要只能引用已导入内容，下载成功也不等于图片已真实查看/几何验收通过。

查询level与输出格式是两个独立参数：`full+compact`仍会执行全量查询，[分层说明5行](D:/Work/skills/crownscript-executor/references/entity-modeling/details/topology-layer-design.md:5)、13行。仅裁剪输出不能减少外部耗时；先ids、按目标查详情，确需full再查。通用Harness应支持工具能力元数据中的查询深度/成本提示、有界结果引用、按字段回读和覆盖声明，而不是硬编码某CAD工具名。地方性字段/ID筛选由Skill表达，权限与预算由服务端强制。

同一documentId/versionId不证明模型状态未变，[分层说明104行](D:/Work/skills/crownscript-executor/references/entity-modeling/details/topology-layer-design.md:104)。至少关联外部resource、revision/采集时刻、操作阶段、源文件hash和查询范围；外部写后使相关缓存失效。snapshot恢复的是本地文件，不恢复远端模型状态；先只读核对，不因恢复而重放几何。

## 5. 多状态、重试和预算

样本有本地文件/索引、外部文档/版本/模型、导出任务、签名证据、MCP会话、用户页面状态等彼此独立状态。Run成功不能替代外部验证；脚本`passed`不能证明CAD正确，STEP脚本也不提供通用体积、BRep等价或Hausdorff对比，见[STEP验收40行](D:/Work/skills/step-model-acceptance/references/acceptance-workflow.md:40)。应沿用工具账本、来源记录、验证范围和unknown会话阻塞。

存在必须显式处理的策略冲突：样本允许create_document工具内部在超时/连接/服务器错误时最多3次创建，并以_retry后缀区分可能遗留文档，[执行流程161行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:161)。这不满足AgentDock“写结果未知不自动重放”；外层不重试无法约束工具内部。建议工具登记增加副作用/内部重试说明和可配置能力核验，只有确认幂等或关闭不安全内部重试才开放该写路径；否则标不兼容/需核对，不为了样本放宽总策略。

样本修正上限3次，[执行流程156行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:156)，AgentDock建议repair_attempts=2。有效上限取服务策略、用户配置与Skill约束中的更严格值；Skill不能把服务端预算调高或绕过关闭开关。只读GET与详情批次各有不同重试/间隔，见[执行流程163行](D:/Work/skills/crownscript-executor/references/execution-workflow.md:163)、[分层说明88行](D:/Work/skills/crownscript-executor/references/entity-modeling/details/topology-layer-design.md:88)。需计入一次工具的实际耗时和累计请求数，禁止外层再叠加同样重试。

render_step收集所有面/三角形到内存，裁剪也在完整网格之后，[源码51行](D:/Work/skills/model-geometry-analysis/scripts/render_step.py:51)、94行；大模型可能超过60秒/内存或20MiB文件、100MiB快照限制。本次未测容量，不应声称调大timeout就够。建议通用profile声明CPU/内存/临时磁盘与任务阶段上限、预估输入规模；超限明确失败或请求缩小范围，不能截断网格却称完整。原生库崩溃/卡死需要Sandbox进程树强制终止和清理，不能靠Python异常处理保证。

现有15min活跃、60s命令、180s依赖阶段、30模型步骤、100MiB/1000files快照均为建议初值。复杂多文档工作可能不够，但缺实测不得自动放大；阶段化与检查点用于可解释继续，不以换阶段名重置累计预算。`.venv`和site-packages不进入工作快照，runtime profile保证可重建，不把巨型依赖包算作可移植工作成果。

## 6. 满足性与后续验收

| 通用要求 | v1.1覆盖程度 | 本样本结论/待补 |
|---|---|---|
| Skill读取、自动选择、中文说明 | 方向具备 | 需bundle闭包和获准版本关系，不能单根ZIP即称全套可用 |
| 代码隔离与参数 | 方向具备 | 所有样本/生成器必须Sandbox执行；cwd、独立imports和profile待验 |
| 重依赖/无头渲染 | 不足以证明可运行 | 需镜像/ABI/wheel/字体实测；不执行Windows样本准备命令 |
| 多轮/等待快照 | 已设计 | 补虚拟工程根、长期库与签名证据保全；不保存venv和远端内存 |
| 私有/共享资料 | Skill源有权限边界 | 跨对话用户库及CAS发布是新增通用缺口，不当自动记忆 |
| MCP证据/下载 | 有存储方向 | 需通用resource import、签名资源策略、错误证据保留 |
| 安全重试与unknown | 已设计严格边界 | 样本工具内部创建重试与服务策略有冲突，未核验前不放行 |
| 上下文/大结果 | 有预算和回读设计 | 保留查询深度、完整性与来源；不截断后冒充全量 |

建议只在技术方案批准后安排代表性验证：①静态bundle缺依赖/越权依赖；②批准profile里最小OCP网格/无头图与Pillow字体；③可移植虚拟根下各CLI及相对imports；④只读共享库搜索、个人staging归档、并发CAS冲突与索引部分失败；⑤签名URL过期/错hash/ZIP逃逸/私网重定向；⑥大网格时间内存上限与取消清理；⑦不同外部状态恢复及内部重试核验。可以先用确定性工具替身验证协议，实际MCP/CAD另按授权做集成；未做的都保持O。

本报告仅提出跨领域可复用的运行机制。没有为CAD写业务specialcase、修改用户Skill、扩大网络/浏览器权限或把样本说明里的历史验证升级为AgentDock的实测事实。
