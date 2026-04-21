# Bundle 系统 {#page-creation-bundles}

> **警告**
>
> 在 Symfony 4.0 之前的版本中，推荐使用 Bundle 来组织自己的应用代码。现在[不再推荐这样做](best_practices.md#best-practice-no-application-bundles)，Bundle 应仅用于在多个应用之间共享代码和功能。

> **视频教程**
>
> 更偏好视频教程？请查看 [Symfony Bundle 开发视频系列](https://symfonycasts.com/screencast/bundle-development)。

Bundle 类似于其他软件中的插件，但更为强大。Symfony 框架的核心功能就是通过 Bundle 实现的（FrameworkBundle、SecurityBundle、DebugBundle 等）。Bundle 也通过[第三方 Bundle](https://github.com/search?q=topic%3Asymfony-bundle&type=Repositories) 用于在应用中添加新功能。

应用中使用的 Bundle 必须在 `config/bundles.php` 文件中按[环境](configuration.md#配置环境)启用：

```php
// config/bundles.php
return [
    // 'all' 表示该 Bundle 在任何 Symfony 环境中都启用
    Symfony\Bundle\FrameworkBundle\FrameworkBundle::class => ['all' => true],
    // ...

    // 此 Bundle 仅在 'dev' 环境中启用
    Symfony\Bundle\DebugBundle\DebugBundle::class => ['dev' => true],
    // ...

    // 此 Bundle 仅在 'dev' 和 'test' 环境中启用，因此不能在 'prod' 中使用
    Symfony\Bundle\WebProfilerBundle\WebProfilerBundle::class => ['dev' => true, 'test' => true],
    // ...
];
```

> **提示**
>
> 在使用 [Symfony Flex](setup.md#symfony-flex) 的默认 Symfony 应用中，安装/移除 Bundle 时会自动为你启用/禁用 Bundle，因此你不需要查看或编辑此 `bundles.php` 文件。

---

## 创建 Bundle

本节将创建并启用一个新 Bundle，以说明只需几个步骤即可完成。新 Bundle 名为 AcmeBlogBundle，其中 `Acme` 部分是示例名称，应替换为代表你或你的组织的某个"厂商"名称（例如，名为 `Abc` 的公司可使用 AbcBlogBundle）。

首先创建一个名为 `AcmeBlogBundle` 的新类：

```php
// src/AcmeBlogBundle.php
namespace Acme\BlogBundle;

use Symfony\Component\HttpKernel\Bundle\AbstractBundle;

class AcmeBlogBundle extends AbstractBundle
{
}
```

> **警告**
>
> 如果你的 Bundle 必须与旧版 Symfony 兼容，则需要改为继承 `Symfony\Component\HttpKernel\Bundle\Bundle`。

> **提示**
>
> AcmeBlogBundle 这个名称遵循标准的 [Bundle 命名规范](#bundle-命名规范)。你也可以选择将 Bundle 名称缩短为 BlogBundle，只需将此类命名为 BlogBundle（并将文件命名为 `BlogBundle.php`）。

这个空类是创建新 Bundle 所需的唯一部分。虽然通常是空的，但这个类很强大，可用于自定义 Bundle 的行为。创建好 Bundle 后，启用它：

```php
// config/bundles.php
return [
    // ...
    Acme\BlogBundle\AcmeBlogBundle::class => ['all' => true],
];
```

虽然它还什么都不做，但 AcmeBlogBundle 现已可以使用了。

---

## Bundle 目录结构 {#bundles-directory-structure}

Bundle 的目录结构旨在帮助保持所有 Symfony Bundle 之间代码的一致性。它遵循一套约定，但在需要时可以灵活调整：

**`assets/`**
包含 Web 资源源文件，如 JavaScript 和 TypeScript 文件、CSS 和 Sass 文件，以及与 Bundle 相关的图像和其他不在 `public/` 中的资源（例如 Stimulus 控制器）。

**`config/`**
存放配置，包括路由配置（例如 `routes.php`）。

**`public/`**
包含 Web 资源（图像、编译后的 CSS 和 JavaScript 文件等），通过 `assets:install` 控制台命令复制或符号链接到项目的 `public/` 目录。

**`src/`**
包含与 Bundle 逻辑相关的所有 PHP 类（例如 `Controller/CategoryController.php`）。

**`templates/`**
存放按控制器名称组织的模板（例如 `category/show.html.twig`）。

**`tests/`**
存放 Bundle 的所有测试。

**`translations/`**
存放按域和语言区域组织的翻译（例如 `AcmeBlogBundle.en.xlf`）。

> **提示**
>
> 建议在 Bundle 的 `composer.json` 文件中使用 [PSR-4](https://www.php-fig.org/psr/psr-4/) 自动加载标准。使用命名空间作为键，Bundle 主类的位置（相对于 `composer.json`）作为值。由于主类位于 Bundle 的 `src/` 目录中：
>
> ```json
> {
>     "autoload": {
>         "psr-4": {
>             "Acme\\BlogBundle\\": "src/"
>         }
>     },
>     "autoload-dev": {
>         "psr-4": {
>             "Acme\\BlogBundle\\Tests\\": "tests/"
>         }
>     }
> }
> ```

---

## 开发可复用的 Bundle

Bundle 旨在成为独立于任何特定 Symfony 应用之外的可复用代码片段。但是，Bundle 无法独立运行：它必须在应用内注册才能执行其代码。

在开发过程中这可能有些挑战。在自己的仓库中开发 Bundle 时，周围没有 Symfony 应用，因此需要一种方法在真实的应用环境中测试你的更改。

有两种常见的方法，取决于你的 Bundle 是否已经发布。

### 使用本地路径仓库

如果你的 Bundle 尚未发布（例如，它在 Packagist 上不可用），你可以从任何用于测试的 Symfony 应用中将 Composer 指向本地 Bundle 目录。

编辑你应用的 `composer.json` 文件并添加以下内容：

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

然后，在你的应用中像往常一样安装 Bundle：

```terminal
$ composer require acme/blog-bundle
```

Composer 会在你的本地 Bundle 目录创建一个符号链接（symlink），因此你在 `AcmeBlogBundle/` 目录中所做的任何更改都会立即在应用中生效。现在可以在 `config/bundles.php` 中启用 Bundle：

```php
return [
    // ...
    Acme\BlogBundle\AcmeBlogBundle::class => ['all' => true],
];
```

这种设置在早期开发阶段非常理想，因为它允许快速迭代，而无需发布或重建归档文件。

### 链接已发布的 Bundle

如果你的 Bundle 已经公开发布（例如，已发布到 Packagist），你仍可以在本地开发它，同时在 Symfony 应用中测试它。

在你的应用中，将已安装的 Bundle 替换为指向本地开发副本的符号链接。例如，如果你的 Bundle 安装在 `vendor/acme/blog-bundle/` 下，而本地副本在 `~/Projects/AcmeBlogBundle/`：

```terminal
$ rm -rf vendor/acme/blog-bundle/
$ ln -s ~/Projects/AcmeBlogBundle/ vendor/acme/blog-bundle
```

Symfony 现在将直接使用你的本地 Bundle。你可以编辑代码、运行测试，并立即看到更改。完成后，恢复 vendor 文件夹或用 Composer 重新安装包，以返回到已发布的版本。

---

## 延伸阅读

- [覆盖 Bundle](bundles/override.md)
- [Bundle 最佳实践](bundles/best_practices.md)
- [Bundle 配置](bundles/configuration.md)
- [Bundle 扩展](bundles/extension.md)
- [Bundle Prepend 扩展](bundles/prepend_extension.md)
