# 可复用 Bundle 的最佳实践

本文重点介绍如何组织**可复用 bundle（reusable bundles）**，使其既可配置
（configurable）又可扩展（extendable）。所谓可复用 bundle，是指那些准备在公司
内部多个项目中私有共享，或公开发布后供任何 Symfony 项目安装使用的 bundle。

## Bundle 名称

bundle 同时也是一个 PHP 命名空间（namespace）。该命名空间必须遵循
[PSR-4][psr-4] 中针对 PHP 命名空间与类名的互操作标准（interoperability standard）：
它以 vendor 段开头，后面可以跟零个或多个分类段（category segments），最后以命名空间
短名称结尾，而这个短名称必须以 ``Bundle`` 结束。

只要您在某个命名空间中加入“bundle 类（bundle class）”
——也就是一个继承 ``Symfony\Component\HttpKernel\Bundle\Bundle`` 的类——
该命名空间就会成为一个 bundle。bundle 类名必须满足以下规则：

- 只能使用字母、数字和下划线；
- 使用 StudlyCaps 命名法（即首字母大写的 camelCase）；
- 名称应简洁且有描述性（不超过两个单词）；
- 以前缀形式包含 vendor 名称（以及可选的 category 命名空间）；
- 以 ``Bundle`` 作为后缀。

下面是一些有效的 bundle 命名空间和类名：

| Namespace | Bundle Class Name |
| --- | --- |
| ``Acme\Bundle\BlogBundle`` | `AcmeBlogBundle` |
| ``Acme\BlogBundle`` | `AcmeBlogBundle` |

按惯例，bundle 类的 ``getName()`` 方法应返回类名本身。

> [!NOTE]
> 如果您公开分享 bundle，则必须使用 bundle 类名作为仓库名称。例如应使用
> ``AcmeBlogBundle``，而不是 ``BlogBundle``。

> [!NOTE]
> Symfony 核心 Bundles 不会在 Bundle 类名前加上 ``Symfony`` 前缀，
> 并且始终会添加 ``Bundle`` 子命名空间；例如：
> ``Symfony\Bundle\FrameworkBundle\FrameworkBundle``。

每个 bundle 都有一个 alias，它是 bundle 名称的简短小写下划线版本
（例如 ``AcmeBlogBundle`` 的 alias 是 ``acme_blog``）。这个 alias 用于保证
项目内部的唯一性，也用于定义 bundle 的配置选项（下文会给出一些使用示例）。

## 目录结构

下面是推荐的 ``AcmeBlogBundle`` 目录结构：

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

> [!NOTE]
> 当您的 bundle 类继承推荐的
> ``Symfony\Component\HttpKernel\Bundle\AbstractBundle`` 时，默认就会使用
> 这一目录结构。如果您的 bundle 继承的是
> ``Symfony\Component\HttpKernel\Bundle\Bundle``，则必须像下面这样重写
> ``getPath()`` 方法：

```php
use Symfony\Component\HttpKernel\Bundle\Bundle;

class AcmeBlogBundle extends Bundle
{
    public function getPath(): string
    {
        return \dirname(__DIR__);
    }
}
```

**以下文件是必需的**，因为自动化工具会依赖这种结构约定：

- ``src/AcmeBlogBundle.php``：这个类会把一个普通目录转换成 Symfony bundle
  （请替换为您自己的 bundle 名称）；
- ``README.md``：包含 bundle 的基本说明，通常也会给出一些基础示例以及完整文档的链接
  （它也可以使用 GitHub 支持的其他标记格式，例如 ``README.rst``）；
- ``LICENSE``：代码所采用许可证（license）的完整文本。多数第三方 bundle 会使用
  MIT 许可证，但您也可以[选择任何许可证][choose-any-license]；
- ``docs/index.md``：bundle 文档的根文件。

对最常用的类和文件来说，子目录层级应尽量保持最小。最多不应超过两层。

bundle 目录应视为只读（read-only）。如果您需要写入临时文件，应将其保存在宿主应用
（host application）的 ``cache/`` 或 ``log/`` 目录下。工具可以在 bundle 目录结构中
生成文件，但前提是这些生成文件会作为仓库内容的一部分被提交。

下面这些类和文件有各自推荐或约定的存放位置（有些是强制性的，有些只是大多数开发者
遵循的约定）：

| Type | Directory |
| --- | --- |
| Commands | ``src/Command/`` |
| Controllers | ``src/Controller/`` |
| Service Container Extensions | ``src/DependencyInjection/`` |
| Doctrine ORM entities | ``src/Entity/`` |
| Doctrine ODM documents | ``src/Document/`` |
| Event Listeners | ``src/EventListener/`` |
| Configuration (routes, services, etc.) | ``config/`` |
| Web Assets (compiled CSS and JS, images) | ``public/`` |
| Web Asset sources (``.scss``, ``.ts``, Stimulus) | ``assets/`` |
| Translation files | ``translations/`` |
| Validation (when not using attributes) | ``config/validation/`` |
| Serialization (when not using attributes) | ``config/serialization/`` |
| Templates | ``templates/`` |
| Unit and Functional Tests | ``tests/`` |

## 类

bundle 的目录结构同时也会映射为命名空间层级。例如，一个保存在
``src/Controller/ContentController.php`` 中的 ``ContentController`` 控制器，
其完整类名（fully qualified class name）将是
``Acme\BlogBundle\Controller\ContentController``。

所有类和文件都必须遵循 [Symfony 编码规范](../contributing/code/standards.md)。

某些类应被视为 façade，它们应尽量保持简短，例如 Commands、Helpers、Listeners
和 Controllers。

与事件调度器（event dispatcher）对接的类，应以 ``Listener`` 为后缀。

异常类应保存在 ``Exception`` 子命名空间下。

## Vendors

bundle 不应内嵌第三方 PHP 库，而应依赖 Symfony 的标准自动加载机制。

bundle 也不应内嵌以 JavaScript、CSS 或其他语言编写的第三方库。

## Doctrine Entities/Documents

如果 bundle 包含 Doctrine ORM entities 和/或 ODM documents，建议使用保存在
``config/doctrine/`` 下的 XML 文件来定义映射（mapping）。这样您就可以通过
[Symfony 覆盖 bundle 部分内容的标准机制](./override.md) 来覆盖这些映射。
如果使用 attributes 定义映射，则无法做到这一点。

## 测试

bundle 应附带一个使用 PHPUnit 编写并保存在 ``tests/`` 目录中的测试套件。
测试应遵循以下原则：

- 测试套件必须能够在示例应用中通过简单的 ``phpunit`` 命令执行；
- 功能测试（functional tests）应只用于测试响应输出，以及您可能拥有的一些 profiling
  信息；
- 测试应至少覆盖 95% 的代码库。

> [!NOTE]
> 测试套件中不应包含 ``AllTests.php`` 这类脚本，而应依赖
> ``phpunit.dist.xml`` 文件的存在。

## 持续集成

持续测试 bundle 代码，包括它的所有 commit 和 pull request，是一种被称为
持续集成（Continuous Integration）的良好实践。有多个服务可为开源项目免费提供
这一能力，例如 [GitHub Actions][github-actions]。

bundle 至少应测试以下内容：

- 依赖项的最低边界（通过运行 ``composer update --prefer-lowest``）；
- 所支持的 PHP 版本；
- 所支持的所有 Symfony 主版本（例如如果同时声明支持 ``6.4`` 和 ``7.x``，
  就应都覆盖）。

因此，一个支持 PHP 7.4、8.3、8.4，以及 Symfony 6.4 和 7.x 的 bundle，
其测试矩阵至少应如下所示：

| PHP version | Symfony version | Composer flags |
| --- | --- | --- |
| 7.4 | ``6.4`` | ``--prefer-lowest`` |
| 8.3 | ``7.*`` |  |
| 8.4 | ``7.*`` |  |

> [!TIP]
> 运行测试时，应将 ``SYMFONY_DEPRECATIONS_HELPER`` 环境变量设置为
> ``max[direct]=0``。这样可以确保 bundle 中没有代码直接使用已弃用功能
> （deprecated features）。
>
> 在执行最低依赖版本测试时，可以把这个变量设置为 ``disabled=1``。

### 指定特定的 Symfony 版本

您可以结合 Symfony Flex 使用特殊的 ``SYMFONY_REQUIRE`` 环境变量来安装指定的
Symfony 版本：

```bash
# 这会让所有 Symfony 包都要求 Symfony 7.x
export SYMFONY_REQUIRE=7.*
# 另外，您也可以运行此命令来更新 composer.json 配置
# composer config extra.symfony.require "7.*"

# 在 CI 环境中安装 Symfony Flex
composer global config --no-plugins allow-plugins.symfony/flex true
composer global require --no-progress --no-scripts --no-plugins symfony/flex

# 安装依赖（建议使用 --prefer-dist 和 --no-progress，以获得更好的输出并缩短下载时间）
composer update --prefer-dist --no-progress
```

> [!WARNING]
> 如果您想缓存 Composer 依赖，**不要**缓存 ``vendor/`` 目录，因为这会带来副作用。
> 应改为缓存 ``$HOME/.composer/cache/files``。

## 安装

bundle 应在其 ``composer.json`` 文件中设置 ``"type": "symfony-bundle"``。
这样一来，[Symfony Flex](../setup/flex.md) 就能在安装 bundle 时自动启用它。

如果您的 bundle 需要额外设置（例如配置、新文件、对 ``.gitignore`` 的修改等），
那么您应创建一个 [Symfony Flex recipe][symfony-flex-recipe]。

## 文档

所有类和函数都必须提供完整的 PHPDoc。

同时，您还应在 ``docs/`` 目录中提供更完整的文档。索引文件
（例如 ``docs/index.rst`` 或 ``docs/index.md``）是唯一强制要求的文件，并且必须作为
整套文档的入口。[reStructuredText（rST）](../contributing/documentation/format.md)
是 Symfony 网站渲染文档所使用的格式。

### 安装说明

为了让第三方 bundle 的安装更简单，建议您在 ``README.md`` 中使用下面这种标准化说明。

#### Markdown 示例

````markdown
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
````

#### rST 示例

````rst
Installation
============

Make sure Composer is installed globally, as explained in the
`installation chapter`_ of the Composer documentation.

----------------------------------

Open a command console, enter your project directory and execute:

.. code-block:: terminal

    composer require <package-name>

Applications that don't use Symfony Flex
----------------------------------------

Step 1: Download the Bundle
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Open a command console, enter your project directory and execute the
following command to download the latest stable version of this bundle:

.. code-block:: terminal

    composer require <package-name>

Step 2: Enable the Bundle
~~~~~~~~~~~~~~~~~~~~~~~~~

Then, enable the bundle by adding it to the list of registered bundles
in the ``config/bundles.php`` file of your project::

    // config/bundles.php
    return [
        // ...
        <vendor>\<bundle-name>\<bundle-long-name>::class => ['all' => true],
    ];

.. _`installation chapter`: https://getcomposer.org/doc/00-intro.md
````

上面的示例假设您安装的是 bundle 的最新稳定版本，因此无需提供包版本号
（例如 ``composer require friendsofsymfony/user-bundle``）。如果安装说明指向的是
某个历史版本，或某个不稳定版本，则应包含版本约束（例如
``composer require friendsofsymfony/user-bundle "~2.0@dev"``）。

必要时，您还可以继续补充更多安装步骤（如 *Step 3*、*Step 4* 等），用于解释其他
必需的安装任务，例如注册路由或导出资源。

## 路由

如果 bundle 提供路由，则这些路由必须以前缀形式使用 bundle 的 alias。例如，如果
您的 bundle 名为 ``AcmeBlogBundle``，那么它的所有路由都必须以 ``acme_blog_``
作为前缀。

## 模板

如果 bundle 提供模板，则必须使用 Twig。bundle 不应提供主布局（main layout），
除非它提供的是一个完整可运行的应用。

## 翻译文件

如果 bundle 提供消息翻译（message translations），则必须使用 XLIFF 格式定义；
其 domain 应以 bundle 名称命名（例如 ``AcmeBlog``）。

bundle 不得覆盖其他 bundle 已有的消息。

翻译 domain 必须与翻译文件名匹配。例如，如果翻译 domain 是 ``AcmeBlog``，
那么英文翻译文件名就应为 ``AcmeBlog.en.xlf``。

## 配置

为了提供更高的灵活性，bundle 可以使用 Symfony 内建机制来提供可配置项。

对于简单的配置项，可以依赖 Symfony 配置中的默认 ``parameters`` 条目。Symfony
parameters 是简单的键值对（key/value pairs），而 value 可以是任何合法的 PHP 值。
每个 parameter 名称都应以 bundle alias 开头，尽管这只是最佳实践建议。参数名的
其余部分应使用点号（``.``）分隔不同层级（例如 ``acme_blog.author.email``）。

最终用户可以在任意配置文件中提供这些值：

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

您可以在代码中通过容器读取这些配置参数：

```php
$container->getParameter('acme_blog.author.email');
```

虽然这种机制实现成本最低，但您仍应考虑使用更高级的
[语义化 bundle 配置](./configuration.md)，让配置更健壮。

## 版本控制

bundle 必须遵循 [语义化版本控制标准][semantic-versioning-standard] 进行版本管理。

## 服务

如果 bundle 定义了服务，那么这些服务应以前缀形式使用 bundle alias，而不是像在项目中
那样直接使用完整类名。例如，``AcmeBlogBundle`` 的服务应以 ``acme_blog`` 为前缀。
原因是 bundle 不应依赖服务自动装配（autowiring）或自动配置（autoconfiguration）
等特性，从而避免在编译应用服务时引入额外开销。

此外，那些不打算让应用直接使用的服务，应[定义为 private](../service_container.md)。
对于公开服务，应创建从接口/类到服务 id 的 alias。例如在 MonologBundle 中，
就会创建从 ``Psr\Log\LoggerInterface`` 到 ``logger`` 的 alias，以便可以通过
``LoggerInterface`` 类型提示进行自动装配。

服务不应使用 autowiring 或 autoconfiguration。相反，所有服务都应显式定义。

> [!TIP]
> 如果您不打算让终端用户直接使用某个 service id，可以在它前面加一个点，将其标记为
> *hidden*（例如 ``.acme_blog.logger``）。这样它就不会显示在默认的
> ``debug:container`` 命令输出中。

> [!SEEALSO]
> 如果您想深入了解 bundle 中的服务加载，可以阅读这篇文章：
> [如何在 Bundle 中加载服务配置](./extension.md)。

## Composer 元数据

``composer.json`` 文件至少应包含以下元数据：

``name``
: 由 vendor 和 bundle 短名称组成。如果您是以个人身份发布 bundle，而不是代表公司发布，
  请使用您的个人名称（例如 ``johnsmith/blog-bundle``）。bundle 的短名称中不要重复
  vendor 名称，并且应使用连字符分隔单词。例如：``AcmeBlogBundle`` 会转换为
  ``blog-bundle``，``AcmeSocialConnectBundle`` 会转换为
  ``social-connect-bundle``。

``description``
: 对 bundle 用途的简要说明。

``type``
: 使用 ``symfony-bundle`` 作为值。

``license``
: 一个字符串（或字符串数组），其值应为[有效的许可证标识符][valid-license-identifier]，
  例如 ``MIT``。

``autoload``
: Symfony 会利用这部分信息加载 bundle 的类。建议使用 [PSR-4][psr-4]
  自动加载标准：以命名空间为 key，以 bundle 主类的位置（相对于 ``composer.json``）
  为 value。由于主类位于 bundle 的 ``src/`` 目录中，因此配置通常如下：

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

为了让开发者更容易发现您的 bundle，请将它注册到 [Packagist][packagist]，
也就是 Composer 包的官方仓库。

## 资源

如果 bundle 引用了任何资源（配置文件、翻译文件等），您可以使用物理路径
（例如 ``__DIR__/config/services.xml``）。

过去我们建议只使用逻辑路径（例如 ``@AcmeBlogBundle/config/services.xml``），
并通过 Symfony kernel 提供的[资源定位器（resource locator）](../components/http_kernel.md)
来解析这些路径，但现在已经不再推荐这种做法。

## 了解更多

- [Bundle Extension](./extension.md)
- [Bundle 配置](./configuration.md)
- [创建 UX Bundle](../frontend/create_ux_bundle.md)

[psr-4]: https://www.php-fig.org/psr/psr-4/
[symfony-flex-recipe]: https://github.com/symfony/recipes
[semantic-versioning-standard]: https://semver.org/
[packagist]: https://packagist.org/
[choose-any-license]: https://choosealicense.com/
[valid-license-identifier]: https://spdx.org/licenses/
[github-actions]: https://docs.github.com/en/free-pro-team@latest/actions
