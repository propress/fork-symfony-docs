# Bundle 系统

> [!WARNING]
> 在 Symfony 4.0 之前的版本中，官方曾建议您使用 bundle 来组织自己的应用代码。
> 现在[已不再推荐这样做](./best_practices.md)，bundle 应只用于在多个应用之间共享
> 代码和功能。

> [!NOTE]
> 如果您更喜欢视频教程，可以查看
> [Symfony Bundle Development screencast series][bundle-development-screencast]。

bundle 与其他软件中的插件（plugin）类似，但能力更强。Symfony 框架的核心功能就是
通过 bundle 实现的（例如 ``FrameworkBundle``、``SecurityBundle``、
``DebugBundle`` 等）。bundle 也常用于通过
[第三方 bundle][third-party-bundles] 为应用添加新功能。

应用中使用的 bundle 必须在 ``config/bundles.php`` 文件中按
[环境（environment）](./configuration.md)启用：

```php
// config/bundles.php
return [
    // 'all' 表示该 bundle 会在任何 Symfony 环境中启用
    Symfony\Bundle\FrameworkBundle\FrameworkBundle::class => ['all' => true],
    // ...

    // 这个 bundle 只在 'dev' 环境启用
    Symfony\Bundle\DebugBundle\DebugBundle::class => ['dev' => true],
    // ...

    // 这个 bundle 只在 'dev' 和 'test' 中启用，因此不能在 'prod' 中使用
    Symfony\Bundle\WebProfilerBundle\WebProfilerBundle::class => ['dev' => true, 'test' => true],
    // ...
];
```

> [!TIP]
> 在使用 [Symfony Flex](./setup/flex.md) 的默认 Symfony 应用中，安装或删除
> bundle 时，系统会自动为您启用或禁用它们，因此您通常不需要查看或编辑这个
> ``bundles.php`` 文件。

## 创建 Bundle

本节将创建并启用一个新的 bundle，用来说明整个过程只需要少量步骤。
这里的新 bundle 叫作 ``AcmeBlogBundle``，其中的 ``Acme`` 只是示例名称，
应替换为能够代表您或您所在组织的某个“vendor”名称
（例如某家公司可使用 ``AbcBlogBundle``）。

首先，创建一个名为 ``AcmeBlogBundle`` 的新类：

```php
// src/AcmeBlogBundle.php
namespace Acme\BlogBundle;

use Symfony\Component\HttpKernel\Bundle\AbstractBundle;

class AcmeBlogBundle extends AbstractBundle
{
}
```

> [!WARNING]
> 如果您的 bundle 需要兼容更早的 Symfony 版本，就必须继承
> ``Symfony\Component\HttpKernel\Bundle\Bundle``。

> [!TIP]
> ``AcmeBlogBundle`` 这个名称遵循标准的 Bundle 命名约定
> （bundle naming conventions）。您也可以通过将此类命名为 ``BlogBundle``
> （并将文件命名为 ``BlogBundle.php``）来把 bundle 名称简化为 ``BlogBundle``。

这个空类就是创建新 bundle 所需的唯一代码。尽管它通常是空的，但这个类功能很强，
可用于自定义 bundle 的行为。现在既然已经创建了 bundle，就启用它：

```php
// config/bundles.php
return [
    // ...
    Acme\BlogBundle\AcmeBlogBundle::class => ['all' => true],
];
```

虽然它现在还没有任何实际功能，但 ``AcmeBlogBundle`` 已经可以投入使用了。

## Bundle 目录结构

bundle 的目录结构旨在帮助所有 Symfony bundle 保持代码一致性。它遵循一组约定，
但也允许您在必要时灵活调整：

``assets/``
: 包含 Web 资源的源码，例如 JavaScript、TypeScript、CSS、Sass 文件，
  也包括不在 ``public/`` 中的图片以及其他与 bundle 相关的资源
  （例如 Stimulus controllers）。

``config/``
: 存放配置，包括路由配置（例如 ``routes.php``）。

``public/``
: 包含 Web 资源（图片、编译后的 CSS 和 JavaScript 文件等），并通过
  ``assets:install`` 控制台命令复制或符号链接到项目的 ``public/`` 目录中。

``src/``
: 包含与 bundle 逻辑相关的所有 PHP 类
  （例如 ``Controller/CategoryController.php``）。

``templates/``
: 存放模板，通常按控制器名称组织
  （例如 ``category/show.html.twig``）。

``tests/``
: 存放 bundle 的全部测试。

``translations/``
: 存放翻译文件，通常按 domain 和 locale 组织
  （例如 ``AcmeBlogBundle.en.xlf``）。

> [!TIP]
> 建议您在 bundle 的 ``composer.json`` 文件中使用
> [PSR-4][psr-4] 自动加载标准（autoload standard）。请使用命名空间作为 key，
> 使用 bundle 主类所在位置（相对于 ``composer.json``）作为 value。
> 由于主类位于 bundle 的 ``src/`` 目录中，因此配置通常如下：

```json
{
    "autoload": {
        "psr-4": {
            "Acme\\BlogBundle\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Acme\\BlogBundle\\Tests\\": "tests/"
        }
    }
}
```

## 开发可复用的 Bundle

bundle 的设计目标是成为可复用的代码单元（reusable pieces of code），并独立于
任何特定的 Symfony 应用存在。不过，bundle 本身不能单独运行：它必须注册到某个
应用内部，才能执行其中的代码。

这会让开发过程稍微复杂一些。当您在独立仓库中开发 bundle 时，周围并没有一个完整的
Symfony 应用，因此您需要一种方式，在真实应用环境中测试自己的改动。

通常有两种常见做法，具体选择取决于您的 bundle 是尚未发布，还是已经公开发布。

### 使用本地 Path Repository

如果您的 bundle 还没有发布（例如还未上架 Packagist），那么您可以在任何用于测试的
Symfony 应用中，让 Composer 直接指向本地 bundle 目录。

编辑应用的 ``composer.json`` 文件，并添加以下内容：

```json
{
    "repositories": [
        {
            "type": "path",
            "url": "/path/to/your/AcmeBlogBundle"
        }
    ],
    "require": {
        "acme/blog-bundle": "*"
    }
}
```

然后，在应用中像平常一样安装该 bundle：

```bash
$ composer require acme/blog-bundle
```

Composer 会为您的本地 bundle 目录创建一个符号链接（symbolic link，symlink），
因此您在 ``AcmeBlogBundle/`` 目录中所做的任何修改，都会立即反映在应用中。
现在，您可以在 ``config/bundles.php`` 中启用该 bundle：

```php
return [
    // ...
    Acme\BlogBundle\AcmeBlogBundle::class => ['all' => true],
];
```

这种方式非常适合开发初期，因为它允许您快速迭代，而无需先发布或重新构建归档包。

> 📝 译者注：``"type": "path"`` 是 Composer 提供的本地包引用机制，非常适合
> 在“正在开发的 bundle”和“用于验证效果的 Symfony 应用”之间建立实时联动。

### 链接已发布的 Bundle

如果您的 bundle 已经公开（例如已经发布到 Packagist），您仍然可以在本地开发它，
同时在 Symfony 应用中完成测试。

在应用中，将已安装的 bundle 替换为一个指向本地开发副本的符号链接。例如，假设该
bundle 安装在 ``vendor/acme/blog-bundle/`` 下，而您的本地副本位于
``~/Projects/AcmeBlogBundle/``：

```bash
$ rm -rf vendor/acme/blog-bundle/
$ ln -s ~/Projects/AcmeBlogBundle/ vendor/acme/blog-bundle
```

这样一来，Symfony 就会直接使用您的本地 bundle。您可以修改其代码、运行测试，并
立即看到变更效果。完成后，再恢复 ``vendor/`` 目录，或使用 Composer 重新安装该
包，以切回已发布版本。

## 了解更多

- [覆盖 Bundle](./bundles/override.md)
- [Bundle 最佳实践](./bundles/best_practices.md)
- [Bundle 配置](./bundles/configuration.md)
- [Bundle Extension](./bundles/extension.md)
- [预置 Extension 配置](./bundles/prepend_extension.md)

[third-party-bundles]: https://github.com/search?q=topic%3Asymfony-bundle&type=Repositories
[psr-4]: https://www.php-fig.org/psr/psr-4/
[bundle-development-screencast]: https://symfonycasts.com/screencast/bundle-development
