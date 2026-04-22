# Symfony 框架最佳实践

本文介绍了**用于开发 Symfony Web 应用的最佳实践（best practices）**。
这些实践符合 Symfony 原始创建者所倡导的理念（philosophy）。

如果您不同意其中某些建议，也可以把它们当作一个良好的**起点**，
然后再**扩展并调整为适合您具体需求**的做法。您甚至可以完全忽略它们，
继续使用自己的最佳实践和方法论（methodologies）。Symfony 足够灵活，
可以适应您的需求。

本文假设您已经具备开发 Symfony 应用的经验。如果还没有，请先阅读文档中的
[入门（Getting Started）](./setup.md)部分。

> [!TIP]
> Symfony 提供了一个名为 [Symfony Demo][symfony-demo] 的示例应用，
> 它遵循了本文提到的所有最佳实践，因此您可以在真实项目中看到这些做法。

## 创建项目

### 使用 Symfony Binary 创建 Symfony 应用

Symfony binary 是您在本机下载 Symfony 时安装的一个可执行命令。它提供了多种
实用功能，其中包括创建新的 Symfony 应用的最简单方式：

```bash
$ symfony new my_project_directory
```

在内部，这条 Symfony binary 命令会执行所需的
[Composer][composer] 命令，以基于当前稳定版本
[创建新的 Symfony 应用](./setup.md)。

### 使用默认目录结构

除非您的项目遵循某种强制规定目录结构的开发实践，否则请使用 Symfony 默认的
目录结构。它是扁平的（flat）、直观的（self-explanatory），并且不与 Symfony
强耦合：

```text
your_project/
├─ assets/
├─ bin/
│  └─ console
├─ config/
│  ├─ packages/
│  ├─ routes/
│  └─ services.yaml
├─ migrations/
├─ public/
│  ├─ build/
│  └─ index.php
├─ src/
│  ├─ Kernel.php
│  ├─ Command/
│  ├─ Controller/
│  ├─ DataFixtures/
│  ├─ Entity/
│  ├─ EventSubscriber/
│  ├─ Form/
│  ├─ Repository/
│  ├─ Security/
│  └─ Twig/
├─ templates/
├─ tests/
├─ translations/
├─ var/
│  ├─ cache/
│  └─ log/
└─ vendor/
```

## 配置

### 将环境变量用于基础设施配置

这些选项的值会因机器不同而变化（例如从您的开发机到生产服务器），
但它们不会修改应用本身的行为。

请在项目中[使用环境变量（env vars）](./configuration.md)来定义这些选项，
并创建多个 ``.env`` 文件，以便[按环境配置环境变量](./configuration.md)。

> 📝 译者注：这里的“基础设施配置”通常指数据库地址、缓存服务地址、
> 第三方 API 主机名等部署相关信息。它们会随环境变化，但不应改变业务逻辑。

### 将 Secrets 用于敏感信息

当您的应用包含敏感配置时，例如 API key，您应通过
[Symfony 的 secrets 管理系统](./configuration/secrets.md)
安全地存储这些信息。

### 将 Parameters 用于应用配置

这些选项用于修改应用行为，例如邮件通知的发送者，或者启用的
[feature toggles][feature-toggles]。它们的值不会随着机器变化，因此不应将它们
定义为环境变量。

请将这些选项定义为 ``config/services.yaml`` 文件中的
[parameters](./configuration.md)。您还可以在
``config/services_dev.yaml`` 和 ``config/services_prod.yaml`` 文件中，
按[环境（environment）](./configuration.md)覆盖这些选项。

除非某项应用配置会被重复复用多次并且需要严格验证，否则**不要**使用
[Config 组件（Config component）](./components/config.md)来定义这些选项。

### 使用简短且带前缀的 Parameter 名称

建议将 ``app.`` 用作 [parameters](./configuration.md) 的前缀，以避免与
Symfony 及第三方 bundle/library 的参数发生冲突。然后，只用一到两个词来描述
该参数的用途：

```yaml
# config/services.yaml
parameters:
    # 不要这样做：'dir' 过于泛化，无法表达明确含义
    app.dir: '...'
    # 应该这样做：名称简短，但容易理解
    app.contents_dir: '...'
    # 可以使用点、下划线、短横线，或者什么都不用，但请始终保持一致，
    # 并为所有参数使用同一种格式
    app.dir.contents: '...'
    app.contents-dir: '...'
```

### 使用常量定义很少变化的选项

某些配置选项，例如某个列表中要显示的条目数量，很少发生变化。与其把它们定义成
[配置参数（configuration parameters）](./configuration.md)，不如将它们定义为
相关类中的 PHP 常量（constants）。例如：

```php
// src/Entity/Post.php
namespace App\Entity;

class Post
{
    public const NUMBER_OF_ITEMS = 10;

    // ...
}
```

常量的主要优势在于您可以在任何地方使用它们，包括 Twig 模板和 Doctrine
实体；而 parameters 只能在能够访问
[服务容器（service container）](./service_container.md) 的位置中使用。

不过，将这类配置值定义为常量的一个明显缺点是：在测试中重新定义其值会比较复杂。

## 业务逻辑

### 不要为了组织应用逻辑而创建 Bundle

当 Symfony 2.0 发布时，应用会使用 [bundles](./bundles.md) 按逻辑功能划分代码：
UserBundle、ProductBundle、InvoiceBundle 等。不过，bundle 的本意是可作为独立软件
片段被复用的东西。

如果您确实需要在多个项目中复用某个功能，就为它创建一个 bundle
（放在私有仓库中，不要公开发布）。至于应用中其余的代码，请使用 PHP 命名空间
（namespaces）而不是 bundles 来组织。

### 使用 Autowiring 自动完成应用服务配置

[服务自动装配（Service autowiring）](./service_container/autowiring.md) 会读取构造
函数（或其他方法）上的类型提示（type-hints），并自动把正确的服务传递给各个方法，
这样您就不必显式配置服务，也能让应用维护更加简单。

请将它与[服务自动配置（service autoconfiguration）](./service_container.md)
结合使用，这样还可以自动为需要的服务添加
[服务标签（service tags）](./service_container/tags.md)，例如 Twig 扩展、
事件订阅器等。

### 服务应尽可能设为 Private

请[将服务设为 private](./service_container.md)，避免您通过
``$container->get()`` 访问这些服务。相反，您应使用正确的依赖注入
（dependency injection）。

### 使用 YAML 格式配置您自己的服务

如果您使用[默认的 ``services.yaml`` 配置](./service_container.md)，大多数服务都会
自动完成配置。不过，在某些边缘场景（edge cases）中，您仍需要手动配置服务
（或服务的某些部分）。

YAML 是推荐用于配置服务的格式，因为它对新手更友好且更简洁，不过 Symfony 也支持
PHP 配置。

### 使用 Attributes 定义 Doctrine 实体映射

Doctrine 实体（entities）是存储在某种“数据库”中的普通 PHP 对象。Doctrine
只能通过为模型类配置的映射元数据（mapping metadata）来了解这些实体。

Doctrine 支持多种元数据格式，但推荐使用 PHP attributes，因为它们是目前为止
设置和查找映射信息最方便、最高效的方式。

## 控制器

### 让控制器继承 ``AbstractController`` 基类

Symfony 提供了一个[基础控制器（base controller）](./controller.md)，其中包含了
最常见需求的快捷方式（shortcuts），例如渲染模板或检查安全权限。

让控制器继承这个基类会让您的应用与 Symfony 产生耦合（coupling）。通常来说，
耦合不是好事，但在这里可能是可以接受的，因为控制器不应该包含任何业务逻辑。
控制器应只包含少量“胶水代码（glue code）”，因此被耦合的并不是应用最重要的部分。

### 使用 Attributes 配置路由、缓存和安全

使用 attributes 配置路由（routing）、缓存（caching）和安全（security）
可以简化配置。您无需在多个由不同格式（YAML、PHP）编写的文件之间来回查找；
所有配置都位于您真正需要它的位置，并且只使用一种格式。

### 使用依赖注入获取服务

如果您继承基础 ``AbstractController``，那么您只能直接通过容器中的
``$this->container->get()`` 访问少数最常见的服务
（例如 ``twig``、``router``、``doctrine`` 等）。
更推荐的做法是，使用依赖注入，通过 action 方法参数的类型提示或构造函数参数来
[获取服务](./controller/service.md)。

### 如果方便，可使用 Entity Value Resolvers

如果您正在使用 [Doctrine](./doctrine.md)，那么可以**按需**使用
[EntityValueResolver](./controller/value_resolver.md)，自动查询某个实体并将其作为参数
传入控制器。如果找不到实体，它还会自动显示 404 页面。

如果根据路由变量获取实体的逻辑更复杂，那么与其配置 EntityValueResolver，
不如直接在控制器内部执行 Doctrine 查询
（例如调用某个 [Doctrine repository 方法](./doctrine.md)）。

## 模板

### 对模板名称和变量使用 Snake Case

请对模板名称、目录名和变量名使用小写 snake_case
（例如 ``user_profile`` 而不是 ``userProfile``，
``product/edit_form.html.twig`` 而不是 ``Product/EditForm.html.twig``）。

### 为模板片段添加下划线前缀

模板片段（template fragments），也叫 *“partial templates”*，可让您
[复用模板内容](./templates.md)。建议在它们的名称前加上下划线，以便更容易与完整模板
区分开来（例如 ``_user_metadata.html.twig`` 或
``_caution_message.html.twig``）。

## 表单

### 将表单定义为 PHP 类

在[类中创建表单](./forms.md)可以让它们在应用的不同部分被复用。此外，不在控制器中
直接创建表单，也能让控制器代码更简单、更易维护。

### 在模板中添加表单按钮

表单类应与其具体使用位置无关。例如，一个同时用于创建和编辑条目的表单，
其按钮文本可能会因为使用场景不同而从 “Add new” 变成 “Save changes”。

因此，与其在表单类或控制器中添加按钮，不如在模板中添加按钮。这也能更好地实现
关注点分离（separation of concerns），因为按钮样式（CSS 类和其他属性）定义在
模板中，而不是 PHP 类中。

不过，如果您正在创建[带多个提交按钮的表单](./forms.md)，就应在控制器中定义这些
按钮，而不是在模板中定义。否则，在控制器中处理表单时，您将无法判断用户点击了
哪个按钮。

### 在底层对象上定义验证约束

如果把[验证约束（validation constraints）](./reference/constraints.md)
附加到表单字段上，而不是附加到映射对象（mapped object）上，那么这些验证规则就
无法在其他表单或对象被使用的其他位置中复用。

### 使用单个 Action 负责表单渲染与处理

[渲染表单](./forms.md)和[处理表单](./forms.md)是处理表单时的两个核心任务。
这两者非常相似（大多数情况下几乎完全相同），因此让单个控制器 action 同时处理它们
会简单得多。

## 国际化

### 对翻译文件使用 XLIFF 格式

在 Symfony 支持的所有翻译格式（PHP、Qt、``.po``、``.mo``、JSON、CSV、INI 等）
中，``XLIFF`` 和 ``gettext`` 对专业翻译人员所使用工具的支持最好。而且由于它基于
XML，您可以在编写 ``XLIFF`` 文件内容时对其进行验证。

Symfony 还支持在 XLIFF 文件中添加注释（notes），这会让它们对译者更友好。
归根结底，高质量翻译依赖上下文（context），而这些 XLIFF 注释正好允许您定义这种
上下文。

### 对翻译使用 Key，而不是内容字符串

使用 key 可以简化翻译文件的管理，因为这样一来，您就可以在模板、控制器和服务中
修改原始内容，而无需更新所有翻译文件。

key 应始终描述它们的**用途**，而不是它们的**位置**。例如，如果某个表单中有一个
标签为 “Username” 的字段，那么一个合适的 key 应该是 ``label.username``，
而**不是** ``edit_form.label.username``。

## 安全

### 定义单个 Firewall

除非您确实存在两套完全不同的身份验证系统和用户
（例如主站使用表单登录，API 单独使用 token 系统），否则建议只保留一个 firewall，
以保持配置简单。

### 使用 ``auto`` 密码哈希器

[``auto`` 密码哈希器](./security.md)会根据您的 PHP 安装环境自动选择尽可能最佳的
编码器/哈希器（encoder/hasher）。当前默认的 auto hasher 是 ``bcrypt``。

### 使用 Voters 实现细粒度的安全限制

如果您的安全逻辑比较复杂，应创建自定义的
[security voters](./security/voters.md)，而不是在 ``#[Security]``
attribute 中编写很长的表达式。

## Web 资源

### 使用 AssetMapper 管理 Web 资源

Web 资源（web assets）是 CSS、JavaScript 和图片文件，它们共同决定了站点前端的
外观与交互效果。[AssetMapper](./frontend/asset_mapper.md) 允许您编写现代
JavaScript 和 CSS，而无需承担使用打包器（bundler）带来的复杂性，例如
[Webpack][webpack]（直接使用，或通过
[Webpack Encore](./frontend/encore/index.md) 使用）。

## 测试

### 对 URL 做 Smoke Test

在软件工程中，[smoke testing][smoke-testing] 指的是一种“初步测试，用来暴露那些
足以让候选软件版本被直接否决的严重而简单的故障”。借助
[PHPUnit data providers][phpunit-data-providers]，您可以定义一个功能测试
（functional test），检查应用中的所有 URL 是否都能成功加载：

```php
// tests/ApplicationAvailabilityFunctionalTest.php
namespace App\Tests;

use PHPUnit\Framework\Attributes\DataProvider;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class ApplicationAvailabilityFunctionalTest extends WebTestCase
{
    #[DataProvider('urlProvider')]
    public function testPageIsSuccessful($url): void
    {
        $client = self::createClient();
        $client->request('GET', $url);

        $this->assertResponseIsSuccessful();
    }

    public static function urlProvider(): \Generator
    {
        yield ['/'];
        yield ['/posts'];
        yield ['/post/fixture-post-1'];
        yield ['/blog/category/fixture-category'];
        yield ['/archives'];
        // ...
    }
}
```

在创建应用时就添加这个测试，因为它几乎不需要额外成本，却能确保您的页面都不会
返回错误。之后，您还可以为每个页面添加更具体的测试。

### 在功能测试中硬编码 URL

在 Symfony 应用中，推荐使用路由来[生成 URL](./routing.md)，这样当 URL 发生变化时，
所有链接都能自动更新。不过，如果某个公开 URL 发生了变化，而您又没有设置从旧 URL
到新 URL 的重定向（redirection），那么用户就无法继续访问它。

这也是为什么在测试中建议直接使用原始 URL，而不是根据路由生成 URL。只要路由一改，
测试就会失败，您也就会意识到必须配置一个重定向。

[symfony-demo]: https://github.com/symfony/demo
[composer]: https://getcomposer.org/
[feature-toggles]: https://en.wikipedia.org/wiki/Feature_toggle
[smoke-testing]: https://en.wikipedia.org/wiki/Smoke_testing_(software)
[webpack]: https://webpack.js.org/
[phpunit-data-providers]: https://docs.phpunit.de/en/13.1/writing-tests-for-phpunit.html#data-providers
