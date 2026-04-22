# Contracts 组件

Contracts 组件提供了一组从 Symfony 组件中提取的抽象。它们可用于构建 Symfony 组件已证明有用的语义 - 并且已经具有经过实战测试的实现。

## 安装

Contracts 作为单独的包提供，因此您只能安装项目真正需要的包：

```bash
$ composer require symfony/cache-contracts
$ composer require symfony/event-dispatcher-contracts
$ composer require symfony/deprecation-contracts
$ composer require symfony/http-client-contracts
$ composer require symfony/service-contracts
$ composer require symfony/translation-contracts
```

如果您在 Symfony 应用之外使用此组件，则必须在代码中引入 Composer 生成的 `vendor/autoload.php` 文件来启用类自动加载机制。更多信息请阅读[此文章](./using_components.md)。

## 用法

此包中的抽象对于实现松散耦合和互操作性很有用。通过使用提供的接口作为类型提示，您可以重用任何符合其契约的实现。它可以是 Symfony 组件，也可以是 PHP 社区提供的另一个包。

根据它们的语义，一些接口可以与自动装配结合使用，以无缝地在类中注入服务。

其他接口可能作为标签接口很有用，用于提示在使用自动配置或手动服务标记（或框架提供的任何其他方式）时可以启用的特定行为。

## 设计原则

* Contracts 按域划分，每个都在自己的子命名空间中；
* Contracts 是小型且一致的 PHP 接口、trait、规范性 docblock 和参考测试套件（如果适用）的集合...；
* Contracts 必须有经过验证的实现才能进入此存储库；
* Contracts 必须与现有的 Symfony 组件向后兼容。

实现特定契约的包应在其 `composer.json` 文件的 `provide` 部分中列出它们，使用 `symfony/*-implementation` 约定。例如：

```json
{
    "...": "...",
    "provide": {
        "symfony/cache-implementation": "3.0"
    }
}
```

## 常见问题

### 这与 PHP-FIG 的 PSR 有何不同？

如果适用，提供的契约建立在 [PHP-FIG][php-fig] 的 PSR 之上。然而，PHP-FIG 有不同的目标和不同的流程。Symfony Contracts 专注于提供本身有用的抽象，同时仍与 Symfony 提供的实现兼容。

[php-fig]: https://www.php-fig.org/
