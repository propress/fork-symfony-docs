# Symfony 框架最佳实践

本文描述了**使用 Symfony 开发 Web 应用程序的最佳实践**，这些实践契合 Symfony 创始人所设想的哲学理念。

如果你不认同其中某些建议，可以将它们视为一个**起点**，然后**加以扩展并适配你的具体需求**。你甚至可以完全忽略它们，继续使用自己的最佳实践和方法论。Symfony 足够灵活，可以适应你的需求。

本文假设你已经有过 Symfony 应用开发经验。如果还没有，请先阅读文档的 [入门指南](setup.md) 部分。

> **提示**
>
> Symfony 提供了一个名为 [Symfony Demo](https://github.com/symfony/demo) 的示例应用，它遵循了所有这些最佳实践，你可以在实践中体验它们。

---

## 创建项目

### 使用 Symfony 二进制文件创建 Symfony 应用

Symfony 二进制文件是你 [下载 Symfony](https://symfony.com/download) 时在机器上创建的可执行命令。它提供了多种实用功能，包括创建新 Symfony 应用最简便的方式：

```terminal
$ symfony new my_project_directory
```

在内部，此 Symfony 二进制命令会执行所需的 [Composer](https://getcomposer.org/) 命令，根据当前稳定版本[创建一个新的 Symfony 应用](setup.md#创建-symfony-应用)。

### 使用默认目录结构

除非你的项目遵循某种规定了特定目录结构的开发实践，否则请遵循 Symfony 的默认目录结构。它结构扁平、含义自明，且不与 Symfony 强耦合：

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

---

## 配置

### 使用环境变量进行基础设施配置

这些配置选项的值会因机器不同而不同（例如从开发机到生产服务器），但不会修改应用行为。

在项目中[使用环境变量](configuration.md#使用环境变量)来定义这些选项，并创建多个 `.env` 文件来[按环境配置环境变量](configuration.md#按环境配置)。

### 使用 Secrets 存储敏感信息 {#use-secret-for-sensitive-information}

当你的应用有敏感配置（如 API 密钥）时，应通过 [Symfony 的 secrets 管理系统](configuration/secrets.md) 安全存储这些配置。

### 使用参数进行应用配置

这些选项用于修改应用行为，例如邮件通知的发件人或启用的[功能开关](https://en.wikipedia.org/wiki/Feature_toggle)。它们的值不因机器而异，因此不要将其定义为环境变量。

在 `config/services.yaml` 文件中将这些选项定义为[参数](configuration.md#配置参数)。你可以在 `config/services_dev.yaml` 和 `config/services_prod.yaml` 文件中[按环境](configuration.md#配置环境)覆盖这些选项。

除非应用配置需要被多次复用且需要严格验证，否则*不要*使用 [Config 组件](components/config.md) 来定义这些选项。

### 使用简短且带前缀的参数名

建议使用 `app.` 作为[参数](configuration.md#配置参数)的前缀，以避免与 Symfony 及第三方 Bundle/Library 参数冲突。然后，只用一两个词描述参数的用途：

```yaml
# config/services.yaml
parameters:
    # 不要这样做：'dir' 过于泛泛，没有传递任何含义
    app.dir: '...'
    # 应该这样做：简短但易于理解的名称
    app.contents_dir: '...'
    # 可以使用点、下划线、短横线或不使用任何分隔符，
    # 但要始终保持一致，对所有参数使用同一格式
    app.dir.contents: '...'
    app.contents-dir: '...'
```

### 使用常量定义很少变化的选项

诸如列表中显示的条目数量等配置选项很少变化。与其将它们定义为[配置参数](configuration.md#配置参数)，不如在相关类中将其定义为 PHP 常量：

```php
// src/Entity/Post.php
namespace App\Entity;

class Post
{
    public const NUMBER_OF_ITEMS = 10;

    // ...
}
```

使用常量的主要优势在于你可以在任何地方使用它们，包括 Twig 模板和 Doctrine 实体，而参数只能在有权访问[服务容器](service_container.md)的地方使用。

使用常量处理此类配置值的唯一明显缺点是，在测试中重新定义其值比较复杂。

---

## 业务逻辑

### 不要创建 Bundle 来组织应用逻辑 {#best-practice-no-application-bundles}

当 Symfony 2.0 发布时，应用使用 [Bundle](bundles.md) 将代码划分为逻辑功能：UserBundle、ProductBundle、InvoiceBundle 等。然而，Bundle 的本意是可作为独立软件单元复用的东西。

如果你需要在多个项目中复用某个功能，请为其创建一个 Bundle（放在私有仓库中，不要公开发布）。对于应用其余部分的代码，使用 PHP 命名空间来组织代码，而不是 Bundle。

### 使用自动装配（Autowiring）自动化配置应用服务

[服务自动装配](service_container/autowiring.md) 是一项读取构造函数（或其他方法）上类型提示并自动将正确服务传递给每个方法的功能，使得不必显式配置服务，从而简化应用维护。

将它与[服务自动配置](service_container.md#服务自动配置)结合使用，还可以为需要[服务标签](service_container/tags.md)的服务（如 Twig 扩展、事件订阅者等）自动添加标签。

### 尽可能将服务设为私有

[将服务设为私有](service_container/alias_private.md#设置容器公开性)可防止通过 `$container->get()` 访问这些服务。你必须改用适当的依赖注入。

### 使用 YAML 格式配置你自己的服务

如果你使用[默认的 services.yaml 配置](service_container.md#服务容器服务加载示例)，大多数服务将被自动配置。但在某些边缘情况下，你需要手动配置服务（或其部分）。

YAML 是配置服务的推荐格式，因为它对新手友好且简洁，但 Symfony 也支持 PHP 配置。

### 使用 Attribute 定义 Doctrine 实体映射

Doctrine 实体是存储在某个"数据库"中的普通 PHP 对象。Doctrine 通过为模型类配置的映射元数据来了解你的实体。

Doctrine 支持多种元数据格式，但推荐使用 PHP Attribute，因为它是设置和查找映射信息迄今最方便、最灵活的方式。

---

## 控制器

### 让控制器继承 `AbstractController` 基础控制器

Symfony 提供了一个[基础控制器](controller.md#基础控制器类与服务)，其中包含最常见需求的快捷方式，如渲染模板或检查安全权限。

让控制器继承此基础控制器会将应用与 Symfony 耦合。耦合通常不好，但在此情况下可以接受，因为控制器不应包含任何业务逻辑。控制器只应包含几行*胶水代码*，因此你不是在耦合应用的重要部分。

### 使用 Attribute 配置路由、缓存和安全 {#best-practice-controller-attributes}

使用 Attribute 配置路由、缓存和安全可以简化配置。你不需要浏览用不同格式（YAML、PHP）创建的多个文件：所有配置就在你需要的地方，且只使用一种格式。

### 使用依赖注入获取服务

如果你继承了基础 `AbstractController`，只能通过 `$this->container->get()` 从容器直接访问最常用的服务（如 `twig`、`router`、`doctrine` 等）。你必须改用依赖注入，通过[类型提示 Action 方法参数](controller.md#访问服务)或构造函数参数来获取服务。

### 在方便时使用实体值解析器

如果你在使用 [Doctrine](doctrine.md)，可以*可选地*使用 [EntityValueResolver](doctrine.md#实体值解析器) 自动查询实体并将其作为参数传递给控制器。如果找不到实体，还会自动显示 404 页面。

如果从路由变量获取实体的逻辑比较复杂，与其配置 EntityValueResolver，不如直接在控制器内进行 Doctrine 查询（例如调用 [Doctrine 仓库方法](doctrine.md)）。

---

## 模板

### 模板名称和变量使用蛇形命名法（snake_case）

模板名称、目录和变量使用小写蛇形命名法（例如用 `user_profile` 而非 `userProfile`，用 `product/edit_form.html.twig` 而非 `Product/EditForm.html.twig`）。

### 模板片段以下划线为前缀

模板片段（也称为*"局部模板"*）允许你[复用模板内容](templates.md#复用模板内容)。以下划线为其名称添加前缀，以便与完整模板更好地区分（例如 `_user_metadata.html.twig` 或 `_caution_message.html.twig`）。

---

## 表单

### 将表单定义为 PHP 类

[在类中创建表单](forms.md#在类中创建表单)允许在应用的不同部分复用它们。此外，不在控制器中创建表单可以简化控制器的代码和维护。

### 在模板中添加表单按钮

表单类应对其使用场景保持无感知。例如，用于创建和编辑条目的表单按钮，应根据使用场景从"新增"变为"保存更改"。

建议在模板中而非表单类或控制器中添加按钮。这也改善了关注点分离，因为按钮样式（CSS 类和其他属性）在模板中定义，而不是在 PHP 类中。

但是，如果你创建了一个[带有多个提交按钮的表单](forms.md#处理带有多个按钮的表单)，应在控制器中而非模板中定义它们。否则，在控制器中处理表单时将无法检查哪个按钮被点击。

### 在底层对象上定义验证约束

将[验证约束](reference/constraints.md)附加到表单字段而非映射对象，会导致验证无法在其他表单或使用该对象的其他地方复用。

### 使用单一 Action 渲染和处理表单 {#best-practice-handle-form}

[渲染表单](forms.md#渲染表单)和[处理表单](forms.md#处理表单)是处理表单时的两项主要任务。两者非常相似（大多数时候几乎完全相同），因此让单一控制器 Action 同时处理两者要简单得多。

---

## 国际化 {#best-practice-internationalization}

### 使用 XLIFF 格式存储翻译文件

在 Symfony 支持的所有翻译格式（PHP、Qt、`.po`、`.mo`、JSON、CSV、INI 等）中，`XLIFF` 和 `gettext` 在专业翻译人员使用的工具中得到最佳支持。由于 `XLIFF` 基于 XML，你可以在编写时验证文件内容。

Symfony 还支持 XLIFF 文件中的注释，使其对翻译人员更加友好。好的翻译都需要上下文，而这些 XLIFF 注释允许你定义这种上下文。

### 使用键而非内容字符串进行翻译

使用键可以简化翻译文件的管理，因为你可以更改模板、控制器和服务中的原始内容，而无需更新所有翻译文件。

键应始终描述其*用途*，而*非*其位置。例如，如果表单有一个标签为"Username"的字段，那么好的键应该是 `label.username`，而*不是* `edit_form.label.username`。

---

## 安全

### 定义单个防火墙

除非你有两个合理的不同认证系统和用户（例如，主站的表单登录和仅用于 API 的令牌系统），否则建议只使用一个防火墙以保持简单。

### 使用 `auto` 密码哈希器

[auto 密码哈希器](security.md#auto-密码哈希器) 会根据你的 PHP 安装自动选择最佳的编码器/哈希器。目前，默认的自动哈希器是 `bcrypt`。

### 使用 Voter 实现细粒度的安全限制

如果你的安全逻辑很复杂，应该创建自定义的[安全 Voter](security/voters.md)，而不是在 `#[Security]` 属性中定义冗长的表达式。

---

## Web 资源

### 使用 AssetMapper 管理 Web 资源 {#use-webpack-encore-to-process-web-assets}

Web 资源是使网站前端看起来美观且运行良好的 CSS、JavaScript 和图像文件。[AssetMapper](frontend/asset_mapper.md) 让你可以编写现代 JavaScript 和 CSS，而无需使用 [Webpack](https://webpack.js.org/)（直接使用或通过 [Webpack Encore](frontend/encore/index.md)）这样的打包工具的复杂性。

---

## 测试

### 对 URL 进行冒烟测试

在软件工程中，[冒烟测试](https://en.wikipedia.org/wiki/Smoke_testing_(software)) 是指*"初步测试，以发现足以拒绝一个潜在软件发布的简单故障"*。使用 [PHPUnit 数据提供者](https://docs.phpunit.de/en/13.1/writing-tests-for-phpunit.html#data-providers)，你可以定义一个功能测试，检查所有应用 URL 是否成功加载：

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

在创建应用时就添加此测试，因为它只需很少的工作量，并可检查没有任何页面返回错误。之后，你可以为每个页面添加更具体的测试。

### 在功能测试中硬编码 URL {#hardcode-urls-in-a-functional-test}

在 Symfony 应用中，建议[使用路由生成 URL](routing.md#生成-url)，以便在 URL 变更时自动更新所有链接。但是，如果公开的 URL 发生变化，除非你设置了到新 URL 的重定向，否则用户将无法访问它。

这就是为什么建议在测试中使用原始 URL 而非从路由生成它们。每当路由发生变化时，测试就会失败，你就会知道需要设置重定向。
