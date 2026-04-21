# 术语中英对照表（GLOSSARY）

本书术语的唯一真源。写作时遇到任何术语，第一件事是查这里；查不到就先加一条。

- **"保留英文"**列 = `yes` 表示正文直接保留英文（如 Kernel / Bundle），不译。
- **首现形式**：术语在一节内第一次出现时的写法。

| 英文 | 中文译法 | 保留英文 | 首现形式 | 说明 / 禁止译法 |
| --- | --- | --- | --- | --- |
| Kernel | 内核 | yes | "内核（Kernel）" | 代码里直接说 Kernel。不要译"核心"。 |
| HttpKernel | HttpKernel | yes | HttpKernel | 不译 |
| Container / DI Container | 容器 / DI 容器 | partial | "依赖注入容器（Dependency Injection Container，DI 容器）" | 不要译"依赖注入容具"（某些老书这么写） |
| Service | 服务 | no | 服务（Service） | — |
| Service ID | 服务 ID | yes | 服务 ID | 不要译"服务标识" |
| Bundle | Bundle | yes | Bundle | 不译。它不是"包"也不是"组件"。 |
| Component | 组件 | no | 组件（Component） | 特指 `symfony/*` 独立组件，与 Bundle 区分 |
| Extension | 扩展 | partial | "扩展（Extension）" | 指 DI Extension 类，不是 PHP 扩展 |
| Compiler Pass | Compiler Pass | yes | Compiler Pass | 不译 |
| Autowire / Autowiring | 自动装配 | no | 自动装配（autowire） | 动词译"自动装配"，名词可译"自动装配机制" |
| Autoconfigure | 自动配置 | no | 自动配置（autoconfigure） | — |
| Tag / Tagged Service | 标签 / 带标签服务 | no | "标签（tag）" | 不要译"记号" |
| Request | 请求 | no | 请求（Request） | 指 `Symfony\Component\HttpFoundation\Request` 时保留英文 |
| Response | 响应 | no | 响应（Response） | 同上 |
| Router | 路由器 | partial | "路由器（Router）" | 模块名保留英文：Routing 组件 |
| Route | 路由 | no | 路由（Route） | — |
| Controller | 控制器 | no | 控制器（Controller） | — |
| Firewall | Firewall | yes | Firewall | Security 的 Firewall 是术语，不译"防火墙" |
| Access Control | 访问控制 | no | 访问控制（access_control） | — |
| Authenticator | Authenticator | yes | Authenticator | 不译 |
| Token（Security） | Token | yes | 安全 Token | Security 的 Token，不译"令牌"（避免歧义） |
| Passport | Passport | yes | Passport | Symfony Security 的概念，不译"护照" |
| Voter | Voter | yes | Voter | 不译"投票者" |
| Form | 表单 | no | 表单（Form） | — |
| DataMapper | DataMapper | yes | DataMapper | 不译 |
| DataTransformer | DataTransformer | yes | DataTransformer | 不译 |
| Entity | 实体 | no | 实体（Entity） | — |
| EntityManager | EntityManager | yes | EntityManager | 不译"实体管理器" |
| UnitOfWork | UnitOfWork | yes | UnitOfWork | 不译 |
| Repository（Doctrine） | 仓储 | partial | "仓储（Repository）" | 第一次出现要加括号 |
| Migration | 迁移 | no | 迁移（Migration） | 特指 DB schema migration |
| Fixture | Fixture | yes | Fixture | 不译 |
| Event | 事件 | no | 事件（Event） | — |
| Event Dispatcher | 事件分发器 | partial | "事件分发器（Event Dispatcher）" | 代码类名保留英文 |
| Listener | 监听器 | no | 监听器（Listener） | — |
| Subscriber | 订阅者 | no | 订阅者（Subscriber） | — |
| Message（Messenger） | 消息 | no | 消息（Message） | — |
| Handler | Handler | yes | Handler | 不译"处理器"以避免与 PHP 原生 handler 混淆 |
| Bus | 总线 | no | 总线（Bus） | — |
| Transport | Transport | yes | Transport | 不译"传输" |
| Middleware | 中间件 | no | 中间件（Middleware） | — |
| Stamp | Stamp | yes | Stamp | 不译"戳"或"印章" |
| Cache | 缓存 | no | 缓存（Cache） | — |
| Pool | 池 | no | 池（Pool） | — |
| Serializer | 序列化器 | partial | "序列化器（Serializer）" | 代码里写 Serializer |
| Normalizer | Normalizer | yes | Normalizer | 不译 |
| Encoder | 编码器 | partial | "编码器（Encoder）" | — |
| Validator | 校验器 / 验证器 | partial | "校验器（Validator）" | **全书统一"校验器"**，禁止"验证器"混用 |
| Constraint | 约束 | no | 约束（Constraint） | — |
| Workflow | Workflow | yes | Workflow | 不译"工作流"（在代码语境里），一般描述可说"工作流（Workflow）" |
| Place / Transition | 状态 / 迁移 | no | 状态（Place）/ 迁移（Transition） | 注意 Transition 在 Doctrine 和 Workflow 里都出现，按上下文 |
| Twig | Twig | yes | Twig | — |
| Profiler | Profiler | yes | Profiler | 不译"分析器" |
| Toolbar | 工具条 | no | Web Debug Toolbar | UI 叫"调试工具条" |
| Recipe（Flex） | Recipe | yes | Flex Recipe | 不译"食谱"（口语里可以） |
| Environment（APP_ENV） | 环境 | no | 环境（dev/prod/test） | — |
| Parameter | 参数 | no | 参数（Parameter） | 特指容器参数时加英文 |
| Argument | 参数 | no | 参数（Argument） | 函数/构造器形参。与 Parameter 在中文里都叫"参数"，必要时用"容器参数 / 构造器参数"区分 |
| Attribute（PHP） | 属性 | partial | "PHP 属性（Attribute，8.0+）" | 与"类属性 property"区分：**成员变量叫"类属性 / property"，`#[...]` 叫"PHP Attribute" 或 "属性标注"** |
| Annotation | 注解 | no | 注解（Annotation） | Symfony 8 已弃用；只在讲历史时出现 |
| Dispatch | 分发 / 派发 | no | 分发 | 全书统一"分发"，禁止"触发 / 派发"混用 |
| Fire | 触发 | no | 触发 | 事件 fire = 触发；消息 dispatch = 分发 |
| Throw | 抛出 | no | 抛出 | 异常 throw = 抛出 |
| Inject | 注入 | no | 注入 | — |
| Decorate | 装饰 | no | 装饰（decorate） | Service Decorator = 服务装饰器 |
| Lazy / Eager | 懒加载 / 预加载 | no | — | 禁止"惰性 / 急切" |
| Placeholder | 占位符 | no | — | — |

## 新增规则

1. 新增术语必须**先在本表登记**，再在正文使用。
2. 若两个英文词的中文译法会导致歧义（例：Parameter / Argument 都译"参数"），必须加限定词或保留英文。
3. 本表按首字母排序只在"英文"列；当条目超过 80 条时重新按英文字母排序。
4. 修改已登记条目的中文译法是**全书级别的改动**，需同时全局替换并更新 `PROGRESS.md`。
