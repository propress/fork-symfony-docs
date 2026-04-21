# Contracts 组件

Contracts 组件提供了一组从 Symfony 组件中提取的抽象接口。它们可用于构建基于 Symfony 组件所证明有用的语义——这些语义已有经过实战检验的实现。

## 安装

Contracts 以独立包的形式提供，你可以仅安装项目真正需要的部分：

```terminal
$ composer require symfony/cache-contracts
$ composer require symfony/event-dispatcher-contracts
$ composer require symfony/deprecation-contracts
$ composer require symfony/http-client-contracts
$ composer require symfony/service-contracts
$ composer require symfony/translation-contracts
```

## 使用

此包中的抽象接口有助于实现松耦合和互操作性。通过将提供的接口用作类型提示，你可以复用任何符合其契约的实现——无论是 Symfony 组件，还是 PHP 社区提供的其他包。

根据其语义，某些接口可与自动装配结合使用，以便在你的类中无缝注入服务。

其他接口可作为标记接口使用，提示特定行为——该行为可通过自动配置或手动服务标记（或框架提供的其他方式）启用。

## 设计原则

* Contracts 按领域拆分，每个领域有独立的子命名空间；
* Contracts 是小而一致的 PHP 接口、trait、规范性文档块及参考测试套件的集合（如适用）；
* Contracts 必须有经过验证的实现才能进入此仓库；
* Contracts 必须与现有 Symfony 组件保持向后兼容。

实现了特定契约的包应在其 `composer.json` 文件的 `provide` 部分列出这些契约，使用 `symfony/*-implementation` 约定。例如：

```javascript
{
    "...": "...",
    "provide": {
        "symfony/cache-implementation": "3.0"
    }
}
```

## 常见问题

### 这与 PHP-FIG 的 PSR 有何不同？

在适用的情况下，提供的契约构建于 [PHP-FIG][] 的 PSR 之上。然而，PHP-FIG 有不同的目标和不同的流程。Symfony Contracts 专注于提供本身有用的抽象，同时仍与 Symfony 提供的实现保持兼容。

[PHP-FIG]: https://www.php-fig.org/
