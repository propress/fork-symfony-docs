# 可复用 Bundle 的最佳实践

本文介绍如何构建**可复用 Bundle** 的结构，使其具备可配置性和可扩展性。可复用 Bundle 是指那些可在公司内多个项目中私下共享，或公开发布供任意 Symfony 项目安装使用的 Bundle。

## Bundle 命名规范

Bundle 同时也是一个 PHP 命名空间。该命名空间必须遵循 [PSR-4](https://www.php-fig.org/psr/psr-4/) 关于 PHP 命名空间和类名的互操作性标准：以厂商（vendor）段开头，后跟零个或多个分类段，最后以命名空间短名称结尾，且必须以 `Bundle` 结尾。

当你向某个命名空间添加"bundle 类"（即继承 `Symfony\Component\HttpKernel\Bundle\Bundle` 的类）时，该命名空间就成为一个 Bundle。Bundle 类名必须遵循以下规则：

* 只使用字母数字字符和下划线；
* 使用 StudlyCaps 命名（即首字母大写的驼峰式命名）；
* 使用简短且具有描述性的名称（不超过两个单词）；
* 以厂商名称（以及可选的分类命名空间）作为前缀；
* 以 `Bundle` 作为后缀。

以下是一些有效的 Bundle 命名空间和类名示例：

| 命名空间 | Bundle 类名 |
|---|---|
| `Acme\Bundle\BlogBundle` | AcmeBlogBundle |
| `Acme\BlogBundle` | AcmeBlogBundle |

按照惯例，Bundle 类的 `getName()` 方法应返回类名。

> **注意：**
> 如果你公开分享你的 Bundle，必须使用 Bundle 类名作为仓库名称（例如 AcmeBlogBundle 而不是 BlogBundle）。

> **注意：**
> Symfony 核心 Bundle 不以 `Symfony` 为 Bundle 类名前缀，并且始终添加 `Bundle` 子命名空间；例如：`Symfony\Bundle\FrameworkBundle\FrameworkBundle`。

每个 Bundle 都有一个别名，即 Bundle 名称的小写下划线版本（AcmeBlogBundle 对应 `acme_blog`）。该别名用于在项目中强制唯一性，以及定义 Bundle 的配置选项（参见下面的使用示例）。

## 目录结构

以下是 AcmeBlogBundle 的推荐目录结构：

```text
<your-bundle>/
├── assets/
├── config/
├── docs/
│   └─ index.md
├── public/
├── src/
│   ├── Controller/
│   ├── DependencyInjection/
│   └── AcmeBlogBundle.php
├── templates/
├── tests/
├── translations/
├── LICENSE
└── README.md
```

> **注意：**
> 当你的 Bundle 类继承推荐的 `Symfony\Component\HttpKernel\Bundle\AbstractBundle` 时，默认使用此目录结构。如果你的 Bundle 继承 `Symfony\Component\HttpKernel\Bundle\Bundle` 类，则需要按如下方式覆盖 `getPath()` 方法：
>
> ```php
> use Symfony\Component\HttpKernel\Bundle\Bundle;
>
> class AcmeBlogBundle extends Bundle
> {
>     public function getPath(): string
>     {
>         return \dirname(__DIR__);
>     }
> }
> ```

**以下文件是必需的**，因为它们确保自动化工具可以依赖的结构规范：

* `src/AcmeBlogBundle.php`：这是将普通目录转变为 Symfony Bundle 的类（将其改为你的 Bundle 名称）；
* `README.md`：此文件包含 Bundle 的基本描述，通常展示一些基本示例并链接到完整文档（可使用 GitHub 支持的任何标记格式，如 `README.rst`）；
* `LICENSE`：代码所用许可证的完整内容。大多数第三方 Bundle 以 MIT 许可证发布，但你可以[选择任意许可证](https://choosealicense.com/)；
* `docs/index.md`：Bundle 文档的根文件。

最常用的类和文件所在子目录的深度应尽量保持最小，最多两层。

Bundle 目录是只读的。如果需要写入临时文件，请将其存储在宿主应用的 `cache/` 或 `log/` 目录下。工具可以在 Bundle 目录结构中生成文件，但前提是生成的文件将成为仓库的一部分。

以下类和文件有特定的存放位置（有些是必需的，另一些只是大多数开发者遵循的惯例）：

| 类型 | 目录 |
|---|---|
| 命令（Commands） | `src/Command/` |
| 控制器（Controllers） | `src/Controller/` |
| 服务容器扩展（Service Container Extensions） | `src/DependencyInjection/` |
| Doctrine ORM 实体（Doctrine ORM entities） | `src/Entity/` |
| Doctrine ODM 文档（Doctrine ODM documents） | `src/Document/` |
| 事件监听器（Event Listeners） | `src/EventListener/` |
| 配置（路由、服务等） | `config/` |
| Web 资产（编译后的 CSS 和 JS、图片） | `public/` |
| Web 资产源文件（`.scss`、`.ts`、Stimulus） | `assets/` |
| 翻译文件（Translation files） | `translations/` |
| 验证（不使用属性时） | `config/validation/` |
| 序列化（不使用属性时） | `config/serialization/` |
| 模板（Templates） | `templates/` |
| 单元测试和功能测试 | `tests/` |

## 类

Bundle 目录结构用作命名空间层次结构。例如，存储在 `src/Controller/ContentController.php` 中的 `ContentController` 控制器，其完全限定类名为 `Acme\BlogBundle\Controller\ContentController`。

所有类和文件必须遵循 [Symfony 编码标准](../contributing/code/standards.md)。

某些类应该被视为门面（facade），应尽量简短，例如命令（Commands）、帮助类（Helpers）、监听器（Listeners）和控制器（Controllers）。

连接到事件分发器的类应以 `Listener` 为后缀。

异常类应存储在 `Exception` 子命名空间中。

## 第三方库（Vendors）

Bundle 不得内嵌第三方 PHP 库，而应依赖标准的 Symfony 自动加载机制。

Bundle 也不应内嵌以 JavaScript、CSS 或其他语言编写的第三方库。

## Doctrine 实体/文档

如果 Bundle 包含 Doctrine ORM 实体和/或 ODM 文档，建议使用存储在 `config/doctrine/` 中的 XML 文件来定义其映射。这允许你使用[覆盖 Bundle 部分的标准 Symfony 机制](override.md)来覆盖该映射。使用属性定义映射时则无法实现此功能。

## 测试

Bundle 应附带使用 PHPUnit 编写并存储在 `tests/` 目录下的测试套件。测试应遵循以下原则：

* 测试套件必须可以通过从示例应用运行简单的 `phpunit` 命令来执行；
* 功能测试应仅用于测试响应输出以及一些性能分析信息（如果有的话）；
* 测试应覆盖至少 95% 的代码库。

> **注意：**
> 测试套件不得包含 `AllTests.php` 脚本，而必须依赖 `phpunit.dist.xml` 文件的存在。

## 持续集成

持续测试 Bundle 代码，包括所有提交和拉取请求，是一种称为持续集成的良好实践。有几个服务为开源项目免费提供此功能，例如 [GitHub Actions](https://docs.github.com/en/free-pro-team@latest/actions)。

Bundle 至少应测试：

* 其依赖项的下界（通过运行 `composer update --prefer-lowest`）；
* 支持的 PHP 版本；
* 所有受支持的主要 Symfony 版本（例如，如果声明同时支持 `6.4` 和 `7.x`，则两者都要测试）。

因此，支持 PHP 7.4、8.3 和 8.4，以及 Symfony 6.4 和 7.x 的 Bundle 至少应具有以下测试矩阵：

| PHP 版本 | Symfony 版本 | Composer 标志 |
|---|---|---|
| 7.4 | `6.4` | `--prefer-lowest` |
| 8.3 | `7.*` | |
| 8.4 | `7.*` | |

> **提示：**
> 测试应将 `SYMFONY_DEPRECATIONS_HELPER` 环境变量设置为 `max[direct]=0` 来运行。这确保 Bundle 中的代码不会直接使用已废弃的特性。
>
> 最低依赖项测试可以将该变量设置为 `disabled=1` 来运行。

### 指定特定 Symfony 版本

你可以将特殊的 `SYMFONY_REQUIRE` 环境变量与 Symfony Flex 一起使用，以安装特定的 Symfony 版本：

```bash
# 要求所有 Symfony 包使用 Symfony 7.x
export SYMFONY_REQUIRE=7.*
# 或者你可以运行此命令来更新 composer.json 配置
# composer config extra.symfony.require "7.*"

# 在 CI 环境中安装 Symfony Flex
composer global config --no-plugins allow-plugins.symfony/flex true
composer global require --no-progress --no-scripts --no-plugins symfony/flex

# 安装依赖项（推荐使用 --prefer-dist 和 --no-progress 以获得更好的输出和更快的下载速度）
composer update --prefer-dist --no-progress
```

> **警告：**
> 如果你想缓存 Composer 依赖项，**不要**缓存 `vendor/` 目录，因为这会产生副作用。请改为缓存 `$HOME/.composer/cache/files`。

## 安装

Bundle 应在其 `composer.json` 文件中设置 `"type": "symfony-bundle"`。这样，[Symfony Flex](../setup.md#symfony-flex) 就能在安装时自动启用你的 Bundle。

如果你的 Bundle 需要任何设置（例如配置、新文件、对 `.gitignore` 的更改等），则应创建一个 [Symfony Flex recipe](https://github.com/symfony/recipes)。

## 文档

所有类和函数必须附有完整的 PHPDoc。

还应在 `docs/` 目录中提供详尽的文档。索引文件（例如 `docs/index.rst` 或 `docs/index.md`）是唯一的必需文件，必须作为文档的入口点。[reStructuredText (rST)](../contributing/documentation/format.md) 是用于在 Symfony 网站上渲染文档的格式。

### 安装说明

为了方便第三方 Bundle 的安装，请考虑在你的 `README.md` 文件中使用以下标准化说明。

```markdown
Installation
============

Make sure Composer is installed globally, as explained in the
[installation chapter](https://getcomposer.org/doc/00-intro.md)
of the Composer documentation.

Applications that use Symfony Flex
----------------------------------

Open a command console, enter your project directory and execute:

```console
composer require <package-name>
```

Applications that don't use Symfony Flex
----------------------------------------

### Step 1: Download the Bundle

Open a command console, enter your project directory and execute the
following command to download the latest stable version of this bundle:

```console
composer require <package-name>
```

### Step 2: Enable the Bundle

Then, enable the bundle by adding it to the list of registered bundles
in the `config/bundles.php` file of your project:

```php
// config/bundles.php

return [
    // ...
    <vendor>\<bundle-name>\<bundle-long-name>::class => ['all' => true],
];
```
```

上面的示例假设你正在安装最新的稳定版本，因此不需要提供包版本号（例如 `composer require friendsofsymfony/user-bundle`）。如果安装说明引用某个过去的 Bundle 版本或不稳定版本，请包含版本约束（例如 `composer require friendsofsymfony/user-bundle "~2.0@dev"`）。

你可以根据需要添加更多安装步骤（*步骤 3*、*步骤 4* 等），以说明其他必要的安装任务，例如注册路由或导出资产。

## 路由

如果 Bundle 提供路由，它们必须以 Bundle 别名为前缀。例如，如果你的 Bundle 名为 AcmeBlogBundle，则其所有路由必须以 `acme_blog_` 为前缀。

## 模板

如果 Bundle 提供模板，它们必须使用 Twig。Bundle 不得提供主布局，除非它提供了完整的可运行应用程序。

## 翻译文件

如果 Bundle 提供消息翻译，它们必须以 XLIFF 格式定义；域名应以 Bundle 名称命名（`AcmeBlog`）。

Bundle 不得覆盖其他 Bundle 中已有的消息。

翻译域必须与翻译文件名匹配。例如，如果翻译域为 `AcmeBlog`，则英文翻译文件名应为 `AcmeBlog.en.xlf`。

## 配置

为提供更大的灵活性，Bundle 可以使用 Symfony 内置机制提供可配置的设置。

对于简单的配置设置，依赖 Symfony 配置的默认 `parameters` 条目。Symfony 参数是简单的键/值对，值可以是任意有效的 PHP 值。每个参数名称应以 Bundle 别名开头，但这只是最佳实践建议。参数名称的其余部分使用句点（`.`）分隔不同部分（例如 `acme_blog.author.email`）。

终端用户可以在任何配置文件中提供值：

```yaml
# config/services.yaml
parameters:
    acme_blog.author.email: 'fabien@example.com'
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return static function (ContainerConfigurator $container): void {
    $container->parameters()
        ->set('acme_blog.author.email', 'fabien@example.com')
    ;
};
```

在你的代码中从容器检索配置参数：

```php
$container->getParameter('acme_blog.author.email');
```

虽然此机制所需的工作量最少，但你应该考虑使用更高级的[语义 Bundle 配置](configuration.md)，使你的配置更加健壮。

## 版本控制

Bundle 必须遵循[语义化版本标准](https://semver.org/)进行版本控制。

## 服务

如果 Bundle 定义了服务，它们必须以 Bundle 别名为前缀，而不是像在项目服务中那样使用完全限定类名。例如，AcmeBlogBundle 的服务必须以 `acme_blog` 为前缀。原因是 Bundle 不应依赖服务自动装配或自动配置等特性，以免在编译应用服务时产生额外开销。

此外，不打算由应用直接使用的服务应[定义为私有服务](../service_container.md#container-private-services)。对于公共服务，应[从接口/类创建别名](../service_container.md#service-autowiring-alias)到服务 ID。例如，在 MonologBundle 中，从 `Psr\Log\LoggerInterface` 创建了一个到 `logger` 的别名，以便 `LoggerInterface` 类型提示可用于自动装配。

服务不应使用自动装配或自动配置，所有服务都应显式定义。

> **提示：**
> 如果不打算让终端用户使用该服务 ID，你可以通过在其前面加上点（`.`）来将其标记为*隐藏*（例如 `.acme_blog.logger`）。这可以防止该服务出现在默认的 `debug:container` 命令输出中。

> 另请参阅：
> 你可以阅读以下文章了解更多关于在 Bundle 中加载服务的内容：[如何在 Bundle 内加载服务配置](extension.md)。

## Composer 元数据

`composer.json` 文件至少应包含以下元数据：

`name`
: 由厂商名称和 Bundle 短名称组成。如果你以个人名义而非代表公司发布 Bundle，请使用你的个人名称（例如 `johnsmith/blog-bundle`）。从 Bundle 短名称中去掉厂商名称，并用连字符分隔每个单词。例如：AcmeBlogBundle 变为 `blog-bundle`，AcmeSocialConnectBundle 变为 `social-connect-bundle`。

`description`
: 对 Bundle 用途的简要说明。

`type`
: 使用 `symfony-bundle` 值。

`license`
: 包含[有效许可证标识符](https://spdx.org/licenses/)的字符串（或字符串数组），例如 `MIT`。

`autoload`
: Symfony 使用此信息加载 Bundle 的类。建议使用 [PSR-4](https://www.php-fig.org/psr/psr-4/) 自动加载标准：以命名空间为键，以 Bundle 主类的位置（相对于 `composer.json`）为值。由于主类位于 Bundle 的 `src/` 目录中：

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

为了让开发者更容易找到你的 Bundle，请将其注册到 [Packagist](https://packagist.org/)，这是 Composer 包的官方仓库。

## 资源

如果 Bundle 引用了任何资源（配置文件、翻译文件等），你可以使用物理路径（例如 `__DIR__/config/services.xml`）。

过去，我们建议只使用逻辑路径（例如 `@AcmeBlogBundle/config/services.xml`）并通过 Symfony 内核提供的[资源定位器](../http_cache.md#http-kernel-resource-locator)来解析它们，但这不再是推荐的做法。

## 了解更多

* [如何在 Bundle 内加载服务配置](extension.md)
* [如何为 Bundle 创建友好的配置](configuration.md)
* [创建 UX Bundle](../frontend/create_ux_bundle.md)
