# 从数据库查找路由：Symfony CMF DynamicRouter

Symfony 核心路由系统非常擅长处理复杂的路由集合。在部署期间会转储一个经过高度优化的路由缓存。

然而，当处理大量数据且每条数据都需要一个易读的 URL 时（例如出于搜索引擎优化目的），路由可能会变慢。此外，如果路由需要由用户编辑，则路由缓存需要频繁重建。

针对这些情况，`DynamicRouter` 提供了一种替代方案：

* 路由存储在数据库中；
* 路径字段上有数据库索引，查找可扩展到大量不同的路由；
* 写操作只影响数据库的索引，非常高效。

当所有路由在部署时都已知且数量不太多时，使用[自定义路由加载器](custom_route_loader.md)是添加更多路由的首选方式。当只处理一种类型的对象时，路由上的 slug 参数结合 Doctrine 实体值解析器即可正常工作。

当你需要具有 Symfony 完整功能集的 `Route` 对象时，`DynamicRouter` 非常有用。每条路由可以定义一个特定的控制器，因此你可以将 URL 结构与应用逻辑解耦。

DynamicRouter 内置支持 Doctrine ORM 和 Doctrine PHPCR-ODM，同时提供 `ContentRepositoryInterface` 以便编写自定义加载器，例如用于其他数据库类型、REST API 或其他任何来源。

DynamicRouter 在 [Symfony CMF 文档](https://symfony.com/doc/current/cmf/bundles/routing/dynamic.html)中有详细说明。
