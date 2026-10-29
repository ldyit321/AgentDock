# AgentDock v1.1设计与静态适配检查记录

日期：2026-10-28。只记录文档、图和静态样本分析；不是产品测试或Skill运行通过报告。

## 当前入口

- [主技术设计](AgentDock-技术设计-v1.1.md)
- [通用Skill宿主补充](AgentDock-通用Skill宿主能力设计-v1.1.md)
- [实际Skill适配结论](AgentDock-Skill适配评估与结论.md)
- [UI设计](AgentDock-UI与交互设计-v1.1.md)与[三页静态预览](AgentDock-UI设计预览-v1.1.html)
- [执行恢复](AgentDock-执行与恢复设计-v1.1.md)、[上下文记忆](AgentDock-上下文与记忆设计-v1.1.md)、[多模态输入](AgentDock-多模态输入设计-v1.1.md)、[文件存储](AgentDock-文件存储与并发设计-v1.1.md)

## 已实际检查

1. Markdown UTF-8、围栏配对、本地链接存在、关键功能及用户指定栈覆盖；本文引用的最终资料重新检查。
2. 用本地已配置Sandbox key作排除检查，不打印key；设计文档和静态UI不含该key。
3. v1.1架构图validate／deliver／strict check／browser-check均pass；[最终自动检查摘要](design/architecture-v1.1/final-evidence/agentdock-architecture.finalize-summary.json)。
4. UI预览无外部资源和业务请求；已在本机Edge headless生成并查看1440×1200工作台与架构图截图。仅桌面示意，不当作Vue／Element Plus产品已实现，也没有自动验收所有新增表单和无障碍行为。
5. 对实际套件联合静态审阅：8个入口、15个Python文件、2份requirements和相关参考；[清单](design/skill-assessment/inventory.json)记录67个当前文件的size/hash，8个入口hash与初读一致。
6. 两个取证专题的110条源文件行号引用已检查路径和行数有效。入口层直接相对链接没有缺失；深层越包资料明确为外部依赖，不冒称全包自足或外部目标不存在。
7. 保留v1.0文件为历史；当前规则、需求最新决定和接续说明已指向v1.1。

## 独立审阅后修订

- Mongo队列不套用SQL行锁，改为CAS、控制文档冲突点、必要事务及reservation只释放一次。
- 统一入队分配epoch／领取不重复递增，问题组open与审批pending的逻辑映射。
- 扫描PDF的needs_vision是内容标志，不阻塞可运行前提，避免先识别才能运行的死锁。
- 文档按范围解析的幂等键含owner、源hash、parser revision、规范化范围及选项。
- 摘要是可溯源派生视图，不能替代审批、权限、领域文件或原文。
- 真实Skill发现OAuth、套件、原生依赖、资源、视图和长期资料等缺口后，用通用契约补充，不写CAD专用图节点。
- 普通附件不开放跨对话任意浏览；明确发布的资料集合可受控导入，未完成Run跨对话恢复不在首版。
- unknown写入在服务端operation_blocks登记；换对话／连接／复制状态不能解除本人托管外部写屏障，共享连接写屏障保护其他用户。核对记录不能由上传文件伪造。
- 其他获准二进制可以由Skill工具解析，文件字节ready与内置解释ready区分；不把STEP强行发给不支持该格式的模型端点。

## 没有做的事情

没有安装Vue、Element Plus、MongoDB、模型／MCP依赖或原生几何库；没有部署数据库／对象存储；没有执行样本、环境探针、CAD操作、OAuth登录或真实模型调用；没有修改`D:/Work/skills`。历史OpenSandbox小脚本验证仍是唯一已有运行集成事实。

尚需实测：Mongo副本集与saver恢复、Linux原生依赖、视图提供者、OAuth、内部写重试、多模态请求、输出采集限额与网络强制。当前“设计闭包／静态评估”不等于上述项目已经通过。

## 维护说明

以当前Markdown源文档为准。`work/revise_design_v11.py`是生成初稿的一次性中间脚本，不是现行设计维护入口，重跑会丢失后续人工修订，不应使用它重建当前文档。
