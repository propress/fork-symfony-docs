# 贯穿项目：bookstore（在线书店）

本书所有代码示例都服务于这一个真实项目。每一章都会给它加一个新功能，而不是扔给你一堆彼此孤立的片段。

---

## 为什么要有一个贯穿项目

新手读框架文档最大的痛苦：

- 每章示例用的类名都不一样（一会儿 `Article`、一会儿 `Post`、一会儿 `BlogItem`），**换一个场景脑子就要重新开机**。
- 示例只给一个片段，不告诉你"放进真实项目里其他部分要怎么改"。
- 读完 20 章不知道它们串在一起是什么样子。

bookstore 解决这三个问题。

---

## 业务背景（一句话）

> 一个在线书店。用户能浏览图书、下单、收到邮件通知；管理员能上下架图书、查看订单。

没有什么花哨的。但这个业务刚好能把 Symfony 的所有重要组件都用上一遍。

## 角色

| 角色 | 简称 | 能做什么 |
| --- | --- | --- |
| 游客（Guest） | — | 浏览、搜索图书 |
| 买家（Customer） | CUST | 游客能做的 + 下单、查看自己的订单 |
| 管理员（Admin） | ADMIN | 所有权限 + 上下架图书、查看所有订单、导出报表 |

## 核心实体（领域模型）

```
┌─────────────┐        ┌─────────────┐        ┌─────────────┐
│   Author    │──1:N──│    Book     │──N:M──│  Category   │
└─────────────┘        └─────────────┘        └─────────────┘
                              │
                              │ 1:N
                              ↓
                        ┌─────────────┐
                        │  OrderItem  │
                        └─────────────┘
                              │
                              │ N:1
                              ↓
                        ┌─────────────┐        ┌─────────────┐
                        │    Order    │──N:1──│    User     │
                        └─────────────┘        └─────────────┘
```

*图：bookstore 的核心实体关系。`User` 在 Symfony Security 里也是安全主体。*

## 技术栈

| 组件 | 选型 | 理由 |
| --- | --- | --- |
| 框架 | Symfony **8.0** | 本书基准 |
| PHP | **8.4+** | Symfony 8.0 官方最低要求（见 `setup.rst`） |
| 数据库 | PostgreSQL 15+（MySQL 8 也可） | PG 在全文检索/JSON 支持上更优，但全书避免 PG 特性语法 |
| ORM | Doctrine ORM 3 | Symfony 8.0 默认 |
| 模板 | Twig | — |
| 前端 | 本书主线不依赖前端框架；引入 AssetMapper（21 章讲） | 减少干扰 |
| 异步队列 | Messenger + Doctrine Transport（开发）；AMQP（部署章） | 0 基础最容易跑通 |
| 缓存 | Symfony Cache + filesystem（dev）/ Redis（prod） | — |
| 邮件 | Symfony Mailer + mailtrap（dev）| — |
| 测试 | PHPUnit + Panther（E2E 可选） | — |

## 完整功能清单（按章推进）

| 章 | 给 bookstore 加什么 | 关键产物 |
| --- | --- | --- |
| 00 | `symfony new bookstore --webapp` 初始化 | 空项目跑起来、首页显示 Welcome |
| 01 | 画出本项目的请求生命周期实景图 | `docs/architecture.md`（本项目自己的） |
| 02 | 加 `/books` 和 `/books/{id}` 路由（先返假数据） | 路由注册 |
| 03 | 写 `BookController`、返回 JSON 和 HTML 两种响应 | 第一个真控制器 |
| 04 | 用 Twig 把 `/books` 渲染出页面（layout + 列表 + 详情） | `templates/book/*.html.twig` |
| 05 | 给管理员加"新增图书"表单 | `BookType` + `/admin/books/new` |
| 06 | 给 `BookType` 加校验（ISBN 格式、价格 > 0、书名非空） | Constraint + CustomConstraint |
| 07 | 把假数据换成 Doctrine 实体，跑 migration，用 Fixture 灌入种子数据 | `Book` / `Author` / `Category` / `Order` / `OrderItem` / `User` 实体 |
| 08 | 加登录（表单 + JSON API 两套）、访问控制、Voter（"只有订单主人能查看自己的订单"） | `security.yaml` + `LoginFormAuthenticator` + `OrderVoter` |
| 09 | 把 `BookService` / `OrderService` 抽出来、用 tagged service 实现"图书上下架策略" | `PricingStrategyInterface` 族 + tagged locator |
| 10 | "订单创建后 → 发事件 → 多个订阅者响应"（发邮件、写积分、写审计日志） | `OrderPlacedEvent` + 3 个 Subscriber |
| 11 | 把数据库连接、邮件、Redis 地址全部挪到 `.env`，并加 `%env(resolve:...)%` / `%env(int:...)%` 示例 | `.env` / `.env.local.dist` |
| 12 | 写 CLI 命令 `bookstore:report:sales` 导出日销售报表 | Command + Lock + Signal |
| 13 | 把 10 章的"发邮件 subscriber" 改成异步消息（Messenger） | `OrderPlacedMessage` + 异步 Handler |
| 14 | 给首页图书列表加缓存（含按用户角色分桶） | `CacheInterface` + invalidation |
| 15 | 加一个对接第三方"ISBN 查询 API"的服务 | `HttpClientInterface` + 重试 + 指标 |
| 16 | 真正配置 Mailer + Notifier（邮箱 + 站内信） | `OrderConfirmationMailer` |
| 17 | 对外 REST API：`/api/books` JSON + `/api/books.xml` | Serializer + groups |
| 18 | 订单状态机：`cart → placed → paid → shipped → delivered / cancelled` | `workflow.yaml` |
| 19 | 给 bookstore 写一整套 smoke / unit / functional / e2e 测试 | `tests/` 全家福 |
| 20 | 把通用"审计日志"能力抽成独立 Bundle `AcmeAuditBundle` | 学会写 Bundle |
| 21 | 部署到单机 + 调优（opcache / preload / APCu / Cache warmup） | `Dockerfile` + 压测结果 |
| 22 | 用 Profiler 和 `debug:*` 对上面所有代码复盘 | 调试实战 |
| 23 | 带着读者读 Symfony 源码里 `bookstore` 实际触发的几条关键路径 | 4 条"从代码到源码"的追踪实录 |
| 24 | 跨模块食谱：从零搭一个"多租户 SaaS 版 bookstore" | 综合 recipe |

## 仓库约定

- 每章完工时的 bookstore 代码**作为 diff 思路写在章末 `12_bookstore_增量.md`**。不强制把每一章的可运行快照都打进本仓库（那会让仓库膨胀），而是让读者跟着做。
- 当某一章的增量过于复杂时（如 08 / 13 章），我们在 `24_advanced_recipes/` 中给出一个"完整参考实现"的链接或内嵌代码。
- bookstore 的命名空间：`App\`（Symfony 默认）。
- bookstore 的 vendor 前缀（用于将来拆 Bundle）：`Acme\`。

## 读者怎么用这个项目

**选项 A：跟着做（强烈推荐）**

```bash
# 在 00 章里你会完整执行这两行
symfony new bookstore --version="8.0.*" --webapp
cd bookstore
```

然后每章跟着 `12_bookstore_增量.md` 加代码。读完 24 章，你手上就是一个 **你自己写的、完整的 Symfony 项目**。

**选项 B：只读**

直接看每章的 `12_bookstore_增量.md`，它里面的代码片段都是自包含的。但**强烈不推荐**，因为"只读"的人读完都会发现自己记不住。

---

## 命名风格（写代码时遵守）

- 类名：`PascalCase`
- 属性 / 方法：`camelCase`
- 路由名：`book_index`、`book_show`、`admin_book_new`（snake_case，管理员加前缀 `admin_`）
- 路由 path：`/books`、`/books/{id}`、`/admin/books/new`
- 服务 ID：默认用类 FQN（Symfony 默认行为），不要自定义字符串 ID
- 环境变量名：`BOOKSTORE_FEATURE_X`、`DATABASE_URL`（Symfony 官方约定）

---

> **下一步**：跟随 [`00_getting_started/`](./00_getting_started/) 初始化这个项目。
