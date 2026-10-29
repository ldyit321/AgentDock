# AgentDock 技术设计检查记录

日期：2026-10-28｜范围：设计文件与设计预览，不是产品测试报告。

## 1. 交付清单

| 文件 | 内容 |
|---|---|
| [技术设计主稿](AgentDock-技术设计-v1.0.md) | 推荐栈、方案比较、模块、实体、API、功能逻辑、状态与验收 |
| [UI与交互](AgentDock-UI与交互设计-v1.0.md) | 页面、线框、字段、操作、状态、权限与可访问性 |
| [UI设计预览](AgentDock-UI设计预览.html) | 三页静态设计示意，虚构数据，无服务调用 |
| [执行与恢复](AgentDock-执行与恢复设计-v1.0.md) | 队列、租约、工具账本、人工等待及检查点一致性 |
| [组件架构图](design/architecture/agentdock-architecture.html) | architecture 类型的可查看架构图，JSON源稿同目录保存 |
| [Mermaid源目录](design/mermaid/) | 组件、实体、Agent流程、Skill上传、问答时序、Run状态六份源码 |

## 2. 实际进行的检查

- 三份Markdown均可按UTF-8读取，无替换字符，代码围栏闭合。
- 文档本地链接存在，主要需求及设计要素覆盖检查通过。
- 使用本地配置的Sandbox API key做排除检查，三份设计文档和UI预览均未包含该密钥；检查未输出密钥。
- UI预览无外部script/link/img资源，无后端连接或真实用户数据。
- 架构图finalize依次通过validate、deliver、strict check、browser-check，结果ok=true。
- 实际浏览器为本机Edge headless；截图以1440×1200生成并查看：工作台、Skill库、MCP连接、架构图。关键内容在已查看的桌面截图中可读，未发现遮挡或跨页内容串显。
- 本次未执行移动端真实交互、键盘自动化、完整无障碍审计、产品前端测试或多用户测试，不扩大截图检查结论。
- 六份Mermaid源码已从主文档提取并保存；本次没有独立Mermaid渲染器语法验收，组件图另有已通过浏览器检查的HTML版本。

## 3. 独立审阅与修订

审阅发现并修订：

1. 正常成功Run也必须在清理前保存完整工作快照；失败和取消记录保存完整性。
2. 无工作区变更的Run沿用working_snapshot_id，不用空清单覆盖之前的工作成果。
3. Question ID代表整组问题，一次提交对应唯一恢复请求；主稿与专题统一表名、工具状态和SSE ID。
4. 审批10分钟到期是拒绝观察并恢复；普通问题24小时／总Run48小时等到期才结束expired。
5. 外部写unknown在任何终态仍阻塞新Run，必须经专用核对，不能忽略或重发绕过。
6. 页面等待示例与立即快照／清理策略一致，不再显示环境仍使用中。

主稿的PostgreSQL、MCP认证范围、配额和其他选择仍是建议，不因审阅而变成用户已批准。

## 4. 证据文件

- [最终架构图自动检查摘要](design/architecture/final-evidence/agentdock-architecture.finalize-summary.json)
- [架构图浏览器检查](design/architecture/final-evidence/agentdock-architecture.browser-check.json)
- [工作台截图](design/previews/ui-workbench.png)
- [Skill库截图](design/previews/ui-skills.png)
- [MCP连接截图](design/previews/ui-mcp.png)
- [架构图截图](design/previews/architecture.png)

## 5. 尚未进行的事情

未安装设计中的产品技术栈、未创建PostgreSQL、未搭建前后端工程、未实现LangGraph产品流程、未连接真实MCP或模型、未改OpenSandbox Server配置。本阶段只产出和检查设计资料。

后续先评审设计，再编写实施计划。网络强制、saver租约守卫、SDK输出采集限额等明确标出的兼容点仍需实施前验证。
