# Config 组件

Config 组件提供实用工具来定义和管理 PHP 应用的配置选项。它允许您：

* 定义配置结构、其验证规则、默认值和文档；
* 支持不同的配置格式（YAML、XML、INI 等）；
* 将来自不同来源的多个配置合并到单个配置中。

> [!NOTE]
> 您不必使用此组件来配置 Symfony 应用。相反，请阅读有关如何配置 Symfony 应用的文档。

## 安装

```bash
$ composer require symfony/config
```

如果您在 Symfony 应用之外使用此组件，则必须在代码中引入 Composer 生成的 `vendor/autoload.php` 文件来启用类自动加载机制。更多信息请阅读[此文章](./using_components.md)。

## 了解更多

* [缓存配置](./config/caching.md)
* [定义和处理配置值](./config/definition.md)
* [加载资源](./config/resources.md)
* [Bundle 配置](../bundles/configuration.md)
* [如何使用编译器扩展简化配置](../bundles/extension.md)
* [如何使用 Bundle 扩展简化配置](../bundles/prepend_extension.md)
