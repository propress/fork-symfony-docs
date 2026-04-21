# 写作进度追踪

- 基准：`propress/fork-symfony-docs` @ `8.0`
- 当前工作分支：`copilot/create-symfony-8-handbook`
- 最近更新：2026-04-21
- 当前总进度：**0 / 25 章完成**（骨架阶段）
- 贯穿项目 bookstore 当前进度：尚未初始化（将在 00 章完成）

---

## 章节清单

> 状态定义：`todo` 未动 | `in_progress` 进行中（必须写明断点）| `done` 全节 14 个小节齐全 | `draft` 主要小节完成但 07/08/09/10 尚未补齐

| 编号 | 模块 | 状态 | 最近提交 | 备注 |
| ---- | --- | --- | --- | --- |
| 00 | getting_started | `in_progress` | — | 已完成 README + 01 节，下一刀：`02_第一次请求-响应全链路.md` |
| 01 | architecture_overview | `todo` | — | 依赖：00 |
| 02 | routing | `todo` | — | 依赖：01, 23, 22 |
| 03 | controller | `todo` | — | 依赖：02 |
| 04 | templates_twig | `todo` | — | — |
| 05 | forms | `todo` | — | 依赖：06 |
| 06 | validation | `todo` | — | — |
| 07 | doctrine | `todo` | — | 依赖：09, 11 |
| 08 | security | `todo` | — | 依赖：09, 10 |
| 09 | service_container | `todo` | — | 依赖：01 |
| 10 | event_dispatcher | `todo` | — | 依赖：09 |
| 11 | configuration_and_env | `todo` | — | — |
| 12 | console | `todo` | — | — |
| 13 | messenger | `todo` | — | 依赖：09, 10 |
| 14 | cache | `todo` | — | — |
| 15 | http_client | `todo` | — | — |
| 16 | mailer_notifier | `todo` | — | — |
| 17 | serializer | `todo` | — | — |
| 18 | workflow | `todo` | — | — |
| 19 | testing | `todo` | — | — |
| 20 | bundles | `todo` | — | — |
| 21 | deployment_and_performance | `todo` | — | — |
| 22 | debugging_and_profiler | `todo` | — | ★ 专章，尽早写 |
| 23 | reading_source_and_docs | `todo` | — | ★ 专章，最早写 |
| 24 | advanced_recipes | `todo` | — | 最后写 |
| 99 | reference | `todo` | — | 最后写 |

> "done" 的硬性定义：目录内 14 个小节（README + 01~13 + 99）齐全，其中 **07 选择指南 / 08 常见报错图鉴 / 09 陷阱与反模式 / 10 调试手册** 不得缺席。

---

## 下次开工清单（Next Actions）

**精确切入点**：打开 `00_getting_started/` 目录，下一个要创建的文件是：

1. **`all_in_one_book/00_getting_started/02_第一次请求-响应全链路.md`**
   - 切入方式：从 00/01 节已经跑通的 `hello` 控制器开始，追踪 `public/index.php` → `Kernel` → `HttpKernel::handle` → 路由匹配 → 控制器解析 → 响应返回 → `TerminateEvent` 的完整链路。
   - 必须包含：ASCII 时序图、每一步对应的源码文件绝对路径（相对 vendor）、一次 `dump()` 在哪一步出现在 Profiler 的哪个面板。
2. `00_getting_started/03_项目骨架逐文件解剖.md`
   - 切入方式：对一个 `symfony new` 出来的空项目，从 `composer.json` 到 `bin/console` 每个顶层文件 / 目录一条一条讲清楚。现在 01 节只讲了主要的 6 个，剩余的（`symfony.lock`、`.env.test`、`importmap.php`、`assets/`、`migrations/`、`translations/`）要在这里补齐。
3. 补齐 00 章固定小节：`07_选择指南.md`（Symfony CLI vs Docker vs 原生 PHP 的选择）、`08_常见报错图鉴.md`（至少 6 条：端口占用、权限、PHP 版本不匹配、扩展缺失、composer 网络、`APP_ENV` 未生效）、`09_陷阱与反模式.md`、`10_调试手册.md`（`about` / `debug:router` / `debug:container` 首次亮相）、`12_bookstore_增量.md`（完成 bookstore 仓库初始化）、`13_FAQ.md`（≥10 条）、`99_速查表.md`。
4. 完成 00 章后，进入 **01_architecture_overview/**（请求生命周期总图、三大支柱：Kernel / Container / Bundle）。
5. 按任务书第 8 节，架构章之后**立刻**写 23（元技能）与 22（调试总论），因为后续所有章节都会引用它们。

> 每次会话开场第一件事：读本文件 → 锁定此列表第一个未完成项 → 在对话中明确声明"本次我将完成 X"，然后动手。

---

## 全局「语焉不详清单」

> 这是本书最大的差异化价值，来源：任务书第 3 节 + agent 持续扫描补充。每攻下一条就打勾并在对应章节落地，每发现一条新的就追加。

### 来自任务书第 3 节（初始集）

#### 3.1 Routing
- [ ] 四种路由声明方式（Attribute / YAML / XML / PHP）的适用场景、取舍矩阵、混用优先级
- [ ] 多模块项目中路由的"归属"与目录组织
- [ ] `prefix` / `name_prefix` / `host` / `condition` / `requirements` / `defaults` 的完整心智模型
- [ ] 国际化路由落地
- [ ] `#[Route]` 类上 vs 方法上 vs 组合
- [ ] `#[MapEntity]` / `#[MapQueryParameter]` / `#[MapRequestPayload]` 家族对比与选型
- [ ] `debug:router` / `router:match` 字段逐列解读

#### 3.2 Security
- [ ] Firewall / Access Control / Voter / Authenticator / Token / Passport 关系图
- [ ] 自定义 `RequestMatcherInterface` 在 `access_control.request_matcher` 下完整生效（含 YAML、服务标签、debug 验证）
- [ ] `path` / `host` / `methods` / `ips` / `attributes` / `request_matcher` 的组合与优先级
- [ ] 多 firewall 划分（前台 / 后台 / API）
- [ ] 自定义 Authenticator 从 0 到 1
- [ ] Voter vs `#[IsGranted]` vs `isGranted()`
- [ ] 登录表单 / JSON 登录 / API Token / JWT / OAuth 选型矩阵
- [ ] Remember Me / Login Throttling / Password Hasher 迁移
- [ ] `security.yaml` 每一行的必要性

#### 3.3 DI 容器
- [ ] 服务 ID vs 类名 vs 别名 vs 接口绑定
- [ ] `autowire` / `autoconfigure` / `public` / `bind` / `arguments` 各管什么
- [ ] "我的类没被注入"完整排查流程
- [ ] Tagged Service + `!tagged_iterator` / `!tagged_locator`
- [ ] Compiler Pass 从 0 到 1
- [ ] Decorator / Locator / Factory 的使用场景
- [ ] `_defaults` / `_instanceof`
- [ ] Public vs Private 与测试的关系

#### 3.4 Configuration / Env
- [ ] `.env` / `.env.local` / `.env.test` / 真实 env 的加载顺序
- [ ] `parameters` vs `services` 参数 vs 环境变量 vs 配置值的边界
- [ ] `%env(...)%` 处理器链完整清单与组合
- [ ] `config/packages/{env}/` 覆盖规则
- [ ] 自写 Bundle 的 `Configuration` + `Extension`

#### 3.5 Doctrine
- [ ] EntityManager / UnitOfWork / 持久化生命周期 / flush 边界
- [ ] Lazy / Eager / Fetch Join / DQL / QueryBuilder 选型
- [ ] Migration vs Schema Update vs Fixture 分工
- [ ] Repository 作为服务注入
- [ ] 事务 / 乐观锁 / 悲观锁
- [ ] N+1 识别与根治
- [ ] Attribute mapping vs XML mapping 选择

#### 3.6 Form
- [ ] "三段式"：buildForm / configureOptions / 数据模型
- [ ] DataMapper / DataTransformer / FormEvents 图解
- [ ] Embedded / Collection Form 完整实现
- [ ] Form + DTO vs Form + Entity 选型

#### 3.7 Event Dispatcher
- [ ] Listener vs Subscriber
- [ ] Kernel 事件完整时序图
- [ ] 自定义事件从 0 到 1

#### 3.8 Messenger
- [ ] Message / Handler / Bus / Transport / Middleware / Stamp 全景图
- [ ] Sync vs Async 选型
- [ ] Failed Transport 与重试
- [ ] "消息没被处理"调试流程
- [ ] Doctrine / AMQP / Redis transport 选型

#### 3.9 Console
- [ ] Command 作为服务的注册与自动发现
- [ ] Input / Output / Style / Progress / Table 全景
- [ ] Lock / Signal / LongRunning 正确姿势

#### 3.10 Cache / HttpClient / Mailer / Serializer / Validation / Workflow
- [ ] 每一个都要"多种适配器选型矩阵" + "不生效怎么排查"

#### 3.11 Bundle
- [ ] 什么时候该写 Bundle、什么时候服务就够
- [ ] AbstractBundle（8.x 新风格）vs 旧式
- [ ] 配置、扩展、资源、模板、翻译、路由的挂载点

#### 3.12 调试 / 观测
- [ ] `debug:container|router|autowiring|config|event-dispatcher|twig|firewall|form` 每个都要逐列解读
- [ ] Web Profiler 每个面板怎么读
- [ ] Xdebug + Symfony 常见配置

### agent 新识别（第一次扫描）

- [ ] **Symfony 8.0 vs 7.x / 6.x 的破坏性变化清单**——新手用 Google 搜到老文章踩坑最频繁的问题，需要有"老写法 → 8.0 等价写法"映射表。放在 00 章附录与 23 章。
- [ ] **`symfony` CLI vs `composer` vs `bin/console` 三者职责边界**——新手常见混淆："这个命令到底该用哪个？"。放在 00 章。
- [ ] **Flex recipe 合并冲突的处理**——`composer recipes`、`symfony.lock`、`.env` diff 在升级时的机制，新手看到冲突直接懵。放在 20 或 21 章。
- [ ] **`#[AsController]` / `#[AsCommand]` / `#[AsMessageHandler]` / `#[AsEventListener]` 一系列 `#[As*]` 属性的统一心智模型**——它们其实是 autoconfigure tag 的语法糖，但官方文档分散在各处，应该在 09 DI 章集中讲一次。
- [ ] **"为什么我的 `.env.local` 没生效"终极检查清单**——至少 8 个可能原因（被 dotenv cache 了、env 真实变量覆盖、APP_ENV 错误、文件编码 BOM、末尾无换行、引号错误等）。放在 11 章。

---

## 本次运行日志

### 2026-04-21 第一次运行

- 15:16 读取任务书，确认策略：严格执行第 8 节「第 1 次运行」——只建骨架 + 完成 00 章 README 与第 1 节，不预创建空文件。
- 15:18 探索仓库结构，确认当前分支 `copilot/create-symfony-8-handbook`，文档在仓库根（不是 `docs/`）；目标目录 `all_in_one_book/` 不存在。
- 15:20 建立骨架：`README.md` / `PROGRESS.md` / `STYLE_GUIDE.md` / `GLOSSARY.md` / `BOOKSTORE_PROJECT.md`。
- 15:25 撰写 `00_getting_started/README.md` 与 `01_心智准备_安装_项目解剖.md`（深度版，不是速览）。
- 15:40 自我代码审查 + 术语一致性核对（对 `GLOSSARY.md`），确认示例命令在 Symfony 8.0 / PHP 8.2+ 下有效。
- 15:45 提交并更新本文件。

**下次续跑切入点**：`all_in_one_book/00_getting_started/02_第一次请求-响应全链路.md`，从 01 节末尾跑起来的那个 `hello` 控制器开始追踪链路。
