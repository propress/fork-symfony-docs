# 路由

当应用程序接收到请求时，它会调用一个控制器动作来生成响应。路由配置定义了对于每个传入的 URL 应该运行哪个动作。它还提供了其他有用的功能，例如生成对 SEO 友好的 URL（例如 `/read/intro-to-symfony` 而不是 `index.php?article_id=57`）。

## 创建路由

路由可以使用 YAML、PHP 或属性（Attributes）进行配置。所有格式提供相同的功能和性能，因此请选择你喜欢的方式。Symfony 推荐使用属性，因为将路由和控制器放在同一个地方非常方便。

### 使用属性创建路由

PHP 属性允许你在与路由关联的控制器代码旁边定义路由。在使用 Symfony Flex 的 Symfony 应用程序中，属性默认是启用的，因此你可以立即开始使用它们。

> **注意：**
> 如果你的项目没有使用 Symfony Flex，你必须通过创建以下配置文件来手动启用属性路由：
>
> ```yaml
> # config/routes.yaml
> controllers:
>     resource: routing.controllers
> ```
>
> 这告诉 Symfony 在你的应用程序中查找 `#[Route]` 属性，并注册路由及其关联的控制器。

假设你想为应用程序中的 `/blog` URL 定义一个路由。为此，创建一个如下所示的控制器类：

```php
// src/Controller/BlogController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class BlogController extends AbstractController
{
    #[Route('/blog', name: 'blog_list')]
    public function list(): Response
    {
        // ...
    }
}
```

此配置定义了一个名为 `blog_list` 的路由，当用户请求 `/blog` URL 时匹配该路由。当匹配发生时，应用程序将运行 `BlogController` 类的 `list()` 方法。

> **注意：**
> 在匹配路由时，URL 的查询字符串不被考虑在内。在本示例中，像 `/blog?foo=bar` 和 `/blog?foo=bar&bar=foo` 这样的 URL 也会匹配 `blog_list` 路由。

> **警告：**
> 如果你在同一个文件中定义了多个 PHP 类，Symfony 只会加载第一个类的路由，忽略所有其他路由。路由属性始终优先于在 YAML 或 PHP 文件中定义的路由，Symfony 将始终加载路由属性。

路由名称（`blog_list`）目前并不重要，但当生成 URL 时它将变得至关重要。你只需记住每个路由名称在应用程序中必须是唯一的。

### 在 YAML 或 PHP 文件中创建路由

除了在控制器类中定义路由，你还可以在单独的 YAML 或 PHP 文件中定义它们。主要优点是它们不需要任何额外的依赖项。主要缺点是在检查某个控制器动作的路由时，你必须处理多个文件。

以下示例展示了如何在 YAML 或 PHP 中定义一个名为 `blog_list` 的路由，该路由将 `/blog` URL 与 `BlogController` 的 `list()` 动作关联起来：

**YAML**

```yaml
# config/routes.yaml
blog_list:
    path: /blog
    # the controller value has the format 'controller_class::method_name'
    controller: App\Controller\BlogController::list

    # if the action is implemented as the __invoke() method of the
    # controller class, you can skip the '::method_name' part:
    # controller: App\Controller\BlogController
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\BlogController;

return Routes::config([
    'blog_list' => [
        'path' => '/blog',
        // the controller value has the format [controller_class, method_name]
        'controller' => [BlogController::class, 'list'],

        // if the action is implemented as the __invoke() method of the
        // controller class, you can skip the 'method_name' part:
        // 'controller' => BlogController::class,
    ],
]);
```

### 匹配 HTTP 方法

默认情况下，路由匹配任何 HTTP 方法（`GET`、`POST`、`PUT` 等）。使用 `methods` 选项来限制每个路由应该响应的方法：

**属性**

```php
// src/Controller/BlogApiController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class BlogApiController extends AbstractController
{
    #[Route('/api/posts/{id}', methods: ['GET', 'HEAD'])]
    public function show(int $id): Response
    {
        // ... return a JSON response with the post
    }

    #[Route('/api/posts/{id}', methods: ['PUT'])]
    public function edit(int $id): Response
    {
        // ... edit a post
    }
}
```

**YAML**

```yaml
# config/routes.yaml
api_post_show:
    path:       /api/posts/{id}
    controller: App\Controller\BlogApiController::show
    methods:    GET|HEAD

api_post_edit:
    path:       /api/posts/{id}
    controller: App\Controller\BlogApiController::edit
    methods:    PUT
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\BlogApiController;

return Routes::config([
    'api_post_show' => [
        'path' => '/api/posts/{id}',
        'controller' => [BlogApiController::class, 'show'],
        'methods' => ['GET', 'HEAD'],
    ],
    'api_post_edit' => [
        'path' => '/api/posts/{id}',
        'controller' => [BlogApiController::class, 'edit'],
        'methods' => ['PUT'],
    ],
]);
```

> **提示：**
> HTML 表单只支持 `GET` 和 `POST` 方法。如果你从 HTML 表单中调用一个使用不同方法的路由，请添加一个名为 `_method` 的隐藏字段，其值为要使用的方法（例如 `<input type="hidden" name="_method" value="PUT">`）。如果你使用 Symfony Forms 创建表单，当 `framework.http_method_override` 选项为 `true` 时，这将自动为你完成。
>
> 出于安全考虑，你可以使用 `framework.allowed_http_method_override` 选项来限制可以被覆盖的 HTTP 方法。

### 匹配环境

使用 `env` 选项仅当当前配置环境与给定值匹配时才注册路由：

**属性**

```php
// src/Controller/DefaultController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class DefaultController extends AbstractController
{
    #[Route('/tools', name: 'tools', env: 'dev')]
    public function developerTools(): Response
    {
        // ...
    }

    // You can also pass an array of environments
    #[Route('/tools', name: 'tools', env: ['dev', 'test'])]
    public function developerTools(): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
when@dev:
    tools:
        path: /tools
        controller: App\Controller\DefaultController::developerTools
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\DefaultController;

return Routes::config([
    'when@dev' => [
        'tools' => [
            'path' => '/tools',
            'controller' => [DefaultController::class, 'developerTools'],
        ],
    ],
]);
```

### 匹配表达式

如果你需要某个路由根据任意匹配逻辑进行匹配，请使用 `condition` 选项：

**属性**

```php
// src/Controller/DefaultController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class DefaultController extends AbstractController
{
    #[Route(
        '/contact',
        name: 'contact',
        condition: "context.getMethod() in ['GET', 'HEAD'] and request.headers.get('User-Agent') matches '/firefox/i'",
        // expressions can also include config parameters:
        // condition: "request.headers.get('User-Agent') matches '%app.allowed_browsers%'"
    )]
    public function contact(): Response
    {
        // ...
    }

    #[Route(
        '/posts/{id}',
        name: 'post_show',
        // expressions can retrieve route parameter values using the "params" variable
        condition: "params['id'] < 1000"
    )]
    public function showPost(int $id): Response
    {
        // ... return a JSON response with the post
    }
}
```

**YAML**

```yaml
# config/routes.yaml
contact:
    path:       /contact
    controller: App\Controller\DefaultController::contact
    condition:  "context.getMethod() in ['GET', 'HEAD'] and request.headers.get('User-Agent') matches '/firefox/i'"
    # expressions can also include configuration parameters:
    # condition: "request.headers.get('User-Agent') matches '%app.allowed_browsers%'"
    # expressions can even use environment variables:
    # condition: "context.getHost() == env('APP_MAIN_HOST')"

post_show:
    path:       /posts/{id}
    controller: App\Controller\DefaultController::showPost
    # expressions can retrieve route parameter values using the "params" variable
    condition:  "params['id'] < 1000"
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\DefaultController;

return Routes::config([
    'contact' => [
        'path' => '/contact',
        'controller' => [DefaultController::class, 'contact'],
        'condition' => 'context.getMethod() in ["GET", "HEAD"] and request.headers.get("User-Agent") matches "/firefox/i"',
        // expressions can also include configuration parameters:
        // 'condition' => 'request.headers.get("User-Agent") matches "%app.allowed_browsers%"',
        // expressions can even use environment variables:
        // 'condition' => 'context.getHost() == env("APP_MAIN_HOST")',
    ],
    'post_show' => [
        'path' => '/posts/{id}',
        'controller' => [DefaultController::class, 'showPost'],
        // expressions can retrieve route parameter values using the "params" variable
        'condition' => 'params["id"] < 1000',
    ],
]);
```

`condition` 选项的值是使用任何有效表达式语言语法的表达式，可以使用 Symfony 创建的以下任意变量：

**`context`**
`Symfony\Component\Routing\RequestContext` 的实例，包含有关正在匹配的路由的最基本信息。

**`request`**
表示当前请求的 Symfony Request 对象。

**`params`**
当前路由匹配的路由参数数组。

你还可以使用以下函数：

**`env(string $name)`**
使用环境变量处理器返回变量的值。

**`service(string $alias)`**
返回路由条件服务。

首先，将 `#[AsRoutingConditionService]` 属性或 `routing.condition_service` 标签添加到你想在路由条件中使用的服务：

```php
use Symfony\Bundle\FrameworkBundle\Routing\Attribute\AsRoutingConditionService;
use Symfony\Component\HttpFoundation\Request;

#[AsRoutingConditionService(alias: 'route_checker')]
class RouteChecker
{
    public function check(Request $request): bool
    {
        // ...
    }
}
```

然后，使用 `service()` 函数在条件中引用该服务：

```php
// Controller (using an alias):
#[Route(condition: "service('route_checker').check(request)")]
// Or without alias:
#[Route(condition: "service('App\\\Service\\\RouteChecker').check(request)")]
```

在内部，表达式被编译成原始 PHP 代码。因此，使用 `condition` 键不会产生额外的开销，只需要底层 PHP 执行所需的时间。

> **警告：**
> 在生成 URL 时，条件**不**会被考虑（这将在本文后面进行说明）。

### 调试路由

随着应用程序的增长，你最终会有*很多*路由。Symfony 包含一些命令来帮助你调试路由问题。首先，`debug:router` 命令按 Symfony 评估路由的顺序列出所有应用程序路由：

```bash
$ php bin/console debug:router

----------------  -------  --------------------------------------------
Name              Method   Path
----------------  -------  --------------------------------------------
homepage          ANY      /
contact           GET      /contact
contact_process   POST     /contact
article_show      ANY      /articles/{_locale}/{year}/{title}.{_format}
blog              ANY      /blog/{page}
blog_show         ANY      /blog/{slug}
----------------  -------  --------------------------------------------

# pass this option to also display all the defined route aliases
$ php bin/console debug:router --show-aliases

# pass this option to also display the associated controllers with the routes
$ php bin/console debug:router --show-controllers

# pass this option to only display routes that match the given HTTP method
# (you can use the special value ANY to see routes that match any method)
$ php bin/console debug:router --method=GET
$ php bin/console debug:router --method=ANY
```

将某个路由的名称（或名称的一部分）传递给此参数以打印路由详细信息：

```bash
$ php bin/console debug:router app_lucky_number

+-------------+---------------------------------------------------------+
| Property    | Value                                                   |
+-------------+---------------------------------------------------------+
| Route Name  | app_lucky_number                                        |
| Path        | /lucky/number/{max}                                     |
| ...         | ...                                                     |
| Options     | compiler_class: Symfony\Component\Routing\RouteCompiler |
|             | utf8: true                                              |
+-------------+---------------------------------------------------------+
```

另一个命令是 `router:match`，它显示哪个路由将匹配给定的 URL。它对于找出为什么某个 URL 没有执行你期望的控制器动作很有用：

```bash
$ php bin/console router:match /lucky/number/8

  [OK] Route "app_lucky_number" matches
```

## 路由参数

前面的示例定义的路由中，URL 从不改变（例如 `/blog`）。然而，通常会定义 URL 中某些部分是可变的路由。例如，显示某篇博客文章的 URL 可能包括标题或 slug（例如 `/blog/my-first-post` 或 `/blog/all-about-symfony`）。

在 Symfony 路由中，可变部分被包裹在 `{ }` 中。例如，显示博客文章内容的路由被定义为 `/blog/{slug}`：

**属性**

```php
// src/Controller/BlogController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class BlogController extends AbstractController
{
    // ...

    #[Route('/blog/{slug}', name: 'blog_show')]
    public function show(string $slug): Response
    {
        // $slug will equal the dynamic part of the URL
        // e.g. at /blog/yay-routing, then $slug='yay-routing'

        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
blog_show:
    path:       /blog/{slug}
    controller: App\Controller\BlogController::show
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\BlogController;

return Routes::config([
    'blog_show' => [
        'path' => '/blog/{slug}',
        'controller' => [BlogController::class, 'show'],
    ],
]);
```

可变部分的名称（本示例中为 `{slug}`）用于创建一个 PHP 变量，该变量存储路由内容并传递给控制器。如果用户访问 `/blog/my-first-post` URL，Symfony 将执行 `BlogController` 类中的 `show()` 方法，并将 `$slug = 'my-first-post'` 参数传递给 `show()` 方法。

路由可以定义任意数量的参数，但每个参数在每个路由上只能使用一次（例如 `/blog/posts-about-{category}/page/{pageNumber}`）。

### 参数验证

假设你的应用程序有一个 `blog_show` 路由（URL：`/blog/{slug}`）和一个 `blog_list` 路由（URL：`/blog/{page}`）。由于路由参数接受任意值，因此无法区分这两个路由。

如果用户请求 `/blog/my-first-post`，两个路由都会匹配，Symfony 将使用先定义的路由。要解决这个问题，请使用 `requirements` 选项向 `{page}` 参数添加一些验证：

**属性**

```php
// src/Controller/BlogController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class BlogController extends AbstractController
{
    #[Route('/blog/{page}', name: 'blog_list', requirements: ['page' => '\d+'])]
    public function list(int $page): Response
    {
        // ...
    }

    #[Route('/blog/{slug}', name: 'blog_show')]
    public function show(string $slug): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
blog_list:
    path:       /blog/{page}
    controller: App\Controller\BlogController::list
    requirements:
        page: '\d+'

blog_show:
    path:       /blog/{slug}
    controller: App\Controller\BlogController::show
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\BlogController;

return Routes::config([
    'blog_list' => [
        'path' => '/blog/{page}',
        'controller' => [BlogController::class, 'list'],
        'requirements' => ['page' => '\d+'],
    ],
    'blog_show' => [
        'path' => '/blog/{slug}',
        'controller' => [BlogController::class, 'show'],
    ],
]);
```

`requirements` 选项定义了路由参数必须匹配以使整个路由匹配的 PHP 正则表达式。在本示例中，`\d+` 是一个匹配任意长度*数字*的正则表达式。现在：

| URL                     | 路由          | 参数                          |
|-------------------------|---------------|-------------------------------|
| `/blog/2`               | `blog_list`   | `$page` = `2`                 |
| `/blog/my-first-post`   | `blog_show`   | `$slug` = `my-first-post`     |

> **提示：**
> `Symfony\Component\Routing\Requirement\Requirement` 枚举包含了一组常用的正则表达式常量，例如数字、日期和 UUID，可用作路由参数要求。
>
> **属性**
>
> ```php
> // src/Controller/BlogController.php
> namespace App\Controller;
>
> use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
> use Symfony\Component\HttpFoundation\Response;
> use Symfony\Component\Routing\Attribute\Route;
> use Symfony\Component\Routing\Requirement\Requirement;
>
> class BlogController extends AbstractController
> {
>     #[Route('/blog/{page}', name: 'blog_list', requirements: ['page' => Requirement::DIGITS])]
>     public function list(int $page): Response
>     {
>         // ...
>     }
> }
> ```
>
> **YAML**
>
> ```yaml
> # config/routes.yaml
> blog_list:
>     path:       /blog/{page}
>     controller: App\Controller\BlogController::list
>     requirements:
>         page: !php/const Symfony\Component\Routing\Requirement\Requirement::DIGITS
> ```
>
> **PHP**
>
> ```php
> // config/routes.php
> namespace Symfony\Component\Routing\Loader\Configurator;
>
> use App\Controller\BlogController;
> use Symfony\Component\Routing\Requirement\Requirement;
>
> return Routes::config([
>     'blog_list' => [
>         'path' => '/blog/{page}',
>         'controller' => [BlogController::class, 'list'],
>         'requirements' => ['page' => Requirement::DIGITS],
>     ],
> ]);
> ```

> **提示：**
> 路由要求（以及路由路径）可以包含配置参数，这对于定义一次复杂的正则表达式并在多个路由中复用它们非常有用。

> **提示：**
> 参数还支持 PCRE Unicode 属性，这些是匹配通用字符类型的转义序列。例如，`\p{Lu}` 匹配任何语言中的任何大写字符，`\p{Greek}` 匹配任何希腊字符，等等。

如果你愿意，可以使用语法 `{parameter_name<requirements>}` 在每个参数中内联要求。此功能使配置更简洁，但当要求复杂时，它可能会降低路由的可读性：

**属性**

```php
// src/Controller/BlogController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class BlogController extends AbstractController
{
    #[Route('/blog/{page<\d+>}', name: 'blog_list')]
    public function list(int $page): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
blog_list:
    path:       /blog/{page<\d+>}
    controller: App\Controller\BlogController::list
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\BlogController;

return Routes::config([
    'blog_list' => [
        'path' => '/blog/{page<\d+>}',
        'controller' => [BlogController::class, 'list'],
    ],
]);
```

### 可选参数

在上一个示例中，`blog_list` 的 URL 是 `/blog/{page}`。如果用户访问 `/blog/1`，它会匹配。但如果他们访问 `/blog`，它将**不**匹配。一旦你向路由添加了参数，它就必须有一个值。

你可以通过为 `{page}` 参数添加默认值，使 `blog_list` 在用户访问 `/blog` 时再次匹配。使用属性时，默认值在控制器动作的参数中定义。在其他配置格式中，它们使用 `defaults` 选项定义：

**属性**

```php
// src/Controller/BlogController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class BlogController extends AbstractController
{
    #[Route('/blog/{page}', name: 'blog_list', requirements: ['page' => '\d+'])]
    public function list(int $page = 1): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
blog_list:
    path:       /blog/{page}
    controller: App\Controller\BlogController::list
    defaults:
        page: 1
    requirements:
        page: '\d+'

blog_show:
    # ...
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\BlogController;

return Routes::config([
    'blog_list' => [
        'path' => '/blog/{page}',
        'controller' => [BlogController::class, 'list'],
        'defaults' => ['page' => 1],
        'requirements' => ['page' => '\d+'],
    ],
    'blog_show' => [
        // ...
    ],
]);
```

现在，当用户访问 `/blog` 时，`blog_list` 路由将匹配，`$page` 将默认为 `1`。

> **提示：**
> 默认值允许不匹配要求。

> **警告：**
> 你可以有多个可选参数（例如 `/blog/{slug}/{page}`），但可选参数之后的所有内容必须是可选的。例如，`/{page}/blog` 是一个有效路径，但 `page` 将始终是必需的（即 `/blog` 不会匹配此路由）。

如果你想在生成的 URL 中始终包含某个默认值（例如，在前面的示例中强制生成 `/blog/1` 而不是 `/blog`），请在参数名前添加 `!` 字符：`/blog/{!page}`

与要求一样，默认值也可以使用语法 `{parameter_name?default_value}` 在每个参数中内联。此功能与内联要求兼容，因此你可以在单个参数中内联两者：

**属性**

```php
// src/Controller/BlogController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class BlogController extends AbstractController
{
    #[Route('/blog/{page<\d+>?1}', name: 'blog_list')]
    public function list(int $page): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
blog_list:
    path:       /blog/{page<\d+>?1}
    controller: App\Controller\BlogController::list
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\BlogController;

return Routes::config([
    'blog_list' => [
        'path' => '/blog/{page<\d+>?1}',
        'controller' => [BlogController::class, 'list'],
    ],
]);
```

> **提示：**
> 要为任何参数提供 `null` 默认值，请在 `?` 字符后不添加任何内容（例如 `/blog/{page?}`）。如果这样做，请不要忘记更新相关控制器参数的类型，以允许传递 `null` 值（例如，将 `int $page` 替换为 `?int $page`）。

### 优先级参数

Symfony 按照路由定义的顺序评估路由。如果一个路由的路径匹配许多不同的模式，它可能会阻止其他路由被匹配。在 YAML 或 PHP 配置文件中，你可以在配置文件中上下移动路由定义来控制它们的优先级。在定义为 PHP 属性的路由中，这很难做到，因此你可以在这些路由中设置可选的 `priority` 参数来控制它们的优先级：

**属性**

```php
// src/Controller/BlogController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class BlogController extends AbstractController
{
    /**
     * This route has a greedy pattern and is defined first.
     */
    #[Route('/blog/{slug}', name: 'blog_show')]
    public function show(string $slug): Response
    {
        // ...
    }

    /**
     * This route could not be matched without defining a higher priority than 0.
     */
    #[Route('/blog/list', name: 'blog_list', priority: 2)]
    public function list(): Response
    {
        // ...
    }
}
```

优先级参数需要一个整数值。优先级较高的路由在优先级较低的路由之前排序。未定义时的默认值为 `0`。

### 参数转换

一个常见的路由需求是将存储在某个参数中的值（例如作为用户 ID 的整数）转换为另一个值（例如表示用户的对象）。此功能称为"参数转换器"。

现在，保留之前的路由配置，但更改控制器动作的参数。将 `string $slug` 替换为 `BlogPost $post`：

```php
// src/Controller/BlogController.php
namespace App\Controller;

use App\Entity\BlogPost;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class BlogController extends AbstractController
{
    // ...

    #[Route('/blog/{slug:post}', name: 'blog_show')]
    public function show(BlogPost $post): Response
    {
        // $post is the object whose slug matches the routing parameter

        // ...
    }
}
```

如果你的控制器参数包含对象的类型提示（本例中为 `BlogPost`），"参数转换器"将使用请求参数（本例中为 `slug`）进行数据库请求来查找该对象。如果没有找到对象，Symfony 会自动生成 404 响应。

`{slug:post}` 语法将名为 `slug` 的路由参数映射到名为 `$post` 的控制器参数。它还提示"参数转换器"使用 slug 从数据库中查找对应的 `BlogPost` 对象。

当从路由参数映射多个实体时，可能会发生名称冲突。在这个示例中，路由试图定义两个映射：一个用于作者，一个用于类别；两者都使用相同的 `name` 参数。这是不允许的，因为路由最终会声明 `name` 两次：

```php
#[Route('/search-book/{name:author}/{name:category}')]
```

这样的路由应该改用以下语法：

```php
#[Route('/search-book/{authorName:author.name}/{categoryName:category.name}')]
```

这样，路由参数名是唯一的（`authorName` 和 `categoryName`），"参数转换器"可以正确地将它们映射到控制器参数（`$author` 和 `$category`），并通过名称加载它们。

使用 `#[MapEntity]` 属性可以实现更高级的映射。请查阅 Doctrine 参数转换文档以了解如何自定义用于从路由参数获取对象的数据库查询。

### 支持的枚举参数

你可以将 PHP 支持的枚举（backed enumerations）用作路由参数，因为 Symfony 会自动将它们转换为其标量值。

```php
// src/Controller/OrderController.php
namespace App\Controller;

use App\Enum\OrderStatusEnum;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class OrderController extends AbstractController
{
    #[Route('/orders/list/{status}', name: 'list_orders_by_status')]
    public function list(OrderStatusEnum $status = OrderStatusEnum::Paid): Response
    {
        // ...
    }
}
```

### 特殊参数

除了你自己的参数，路由还可以包含 Symfony 创建的以下任何特殊参数：

**`_controller`**
此参数用于确定在路由匹配时执行哪个控制器和动作。

**`_format`**
匹配的值用于设置 `Request` 对象的"请求格式"。这用于设置响应的 `Content-Type`（例如，`json` 格式转换为 `application/json` 的 `Content-Type`）。

**`_fragment`**
用于设置片段标识符，这是 URL 的可选最后部分，以 `#` 字符开头，用于标识文档的某个部分。

**`_locale`**
用于在请求上设置语言区域。

**`_query`**
要添加到生成的 URL 中的查询参数数组。

你可以在单个路由和路由导入中包含这些属性（`_fragment` 除外）。Symfony 定义了一些具有相同名称（去掉前导下划线）的特殊属性，因此你可以更轻松地定义它们：

**属性**

```php
// src/Controller/ArticleController.php
namespace App\Controller;

// ...
class ArticleController extends AbstractController
{
    #[Route(
        path: '/articles/{_locale}/search.{_format}',
        locale: 'en',
        format: 'html',
        query: ['page' => 1],
        requirements: [
            '_locale' => 'en|fr',
            '_format' => 'html|xml',
        ],
    )]
    public function search(): Response
    {
    }
}
```

**YAML**

```yaml
# config/routes.yaml
article_search:
  path:        /articles/{_locale}/search.{_format}
  controller:  App\Controller\ArticleController::search
  locale:      en
  format:      html
  query:
      page:    1
  requirements:
      _locale: en|fr
      _format: html|xml
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\ArticleController;

return Routes::config([
    'article_show' => [
        'path' => '/articles/{_locale}/search.{_format}',
        'controller' => [ArticleController::class, 'search'],
        'locale' => 'en',
        'format' => 'html',
        'query' => ['page' => 1],
        'requirements' => ['_locale' => 'en|fr', '_format' => 'html|xml'],
    ],
]);
```

### 额外参数

在路由的 `defaults` 选项中，你可以选择性地定义路由配置中未包含的参数。这对于将额外参数传递给路由的控制器很有用：

**属性**

```php
// src/Controller/BlogController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class BlogController extends AbstractController
{
    #[Route('/blog/{page}', name: 'blog_index', defaults: ['page' => 1, 'title' => 'Hello world!'])]
    public function index(int $page, string $title): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
blog_index:
    path:       /blog/{page}
    controller: App\Controller\BlogController::index
    defaults:
        page: 1
        title: "Hello world!"
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\BlogController;

return Routes::config([
    'blog_index' => [
        'path' => '/blog/{page}',
        'controller' => [BlogController::class, 'index'],
        'defaults' => ['page' => 1, 'title' => 'Hello world!'],
    ],
]);
```

### 路由参数中的斜杠字符

路由参数可以包含除 `/` 斜杠字符以外的任何值，因为斜杠是用于分隔 URL 不同部分的字符。例如，如果 `/share/{token}` 路由中的 `token` 值包含 `/` 字符，则此路由将不会匹配。

一种可能的解决方案是更改参数要求，使其更宽松：

**属性**

```php
// src/Controller/DefaultController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class DefaultController extends AbstractController
{
    #[Route('/share/{token}', name: 'share', requirements: ['token' => '.+'])]
    public function share($token): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
share:
    path:       /share/{token}
    controller: App\Controller\DefaultController::share
    requirements:
        token: .+
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\DefaultController;

return Routes::config([
    'share' => [
        'path' => '/share/{token}',
        'controller' => [DefaultController::class, 'share'],
        'requirements' => ['token' => '.+'],
    ],
]);
```

> **注意：**
> 如果路由定义了多个参数，并且你对所有参数都应用了这个宽松的正则表达式，你可能会得到意外的结果。例如，如果路由定义是 `/share/{path}/{token}`，而 `path` 和 `token` 都接受 `/`，那么 `token` 将只获取最后一部分，其余部分由 `path` 匹配。

> **注意：**
> 如果路由包含特殊的 `{_format}` 参数，你不应该对允许斜杠的参数使用 `.+` 要求。例如，如果模式是 `/share/{token}.{_format}`，并且 `{token}` 允许任何字符，那么 `/share/foo/bar.json` URL 会将 `foo/bar.json` 视为 token，而格式将为空。可以通过将 `.+` 要求替换为 `[^.]+` 来解决这个问题，以允许除点以外的任何字符。

## 路由别名

路由别名允许你为同一路由使用多个名称，可用于为已重命名的路由提供向后兼容性。假设你有一个名为 `product_show` 的路由：

**属性**

```php
// src/Controller/ProductController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class ProductController
{
    #[Route('/product/{id}', name: 'product_show')]
    public function show(): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
product_show:
    path: /product/{id}
    controller: App\Controller\ProductController::show
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

return Routes::config([
    'product_show' => [
        'path' => '/product/{id}',
        'controller' => [ProductController::class, 'show'],
    ],
]);
```

现在，假设你想创建一个名为 `product_details` 的新路由，其行为与 `product_show` 完全相同。

你可以为它创建一个别名，而不是复制原始路由。

**属性**

```php
// src/Controller/ProductController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class ProductController
{
    // the "alias" argument assigns an alternate name to this route;
    // the alias will point to the actual route "product_show"
    #[Route('/product/{id}', name: 'product_show', alias: ['product_details'])]
    public function show(): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
product_show:
    path: /product/{id}
    controller: App\Controller\ProductController::show

product_details:
    # "alias" option refers to the name of the route declared above
    alias: product_show
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

return Routes::config([
    'product_show' => [
        'path' => '/product/{id}',
        'controller' => [ProductController::class, 'show'],
    ],
    'product_details' => [
        // "alias" option refers to the name of the route declared above
        'alias' => ['product_show'],
    ],
]);
```

在这个示例中，`product_show` 和 `product_details` 路由都可以在应用程序中使用，并将产生相同的结果。

> **注意：**
> YAML 和 PHP 配置格式是为你不拥有的路由定义别名的唯一方式。使用 PHP 属性时无法做到这一点。
>
> 例如，这允许你为 URL 生成使用自己的路由名称，同时仍然针对第三方 bundle 定义的路由。别名和原始路由不需要在同一文件或格式中声明。

### 废弃路由别名

路由别名可用于为已重命名的路由提供向后兼容性。

现在，假设你想用 `product_details` 替换 `product_show` 路由，并将旧路由标记为废弃。

在前面的示例中，别名 `product_details` 指向 `product_show` 路由。

要将 `product_show` 路由标记为废弃，你需要"切换"别名。`product_show` 变成别名，现在将指向 `product_details` 路由。这样，`product_show` 别名就可以被废弃了。

**属性**

```php
// src/Controller/ProductController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\DeprecatedAlias;
use Symfony\Component\Routing\Attribute\Route;

class ProductController
{
    // this outputs the following generic deprecation message:
    // Since acme/package 1.2: The "product_show" route alias is deprecated. You should stop using it, as it will be removed in the future.
    #[Route('/product/{id}',
        name: 'product_details',
        alias: new DeprecatedAlias(
            aliasName: 'product_show',
            package: 'acme/package',
            version: '1.2',
        ),
    )]
    // Or, you can also define a custom deprecation message (%alias_id% placeholder is available)
    #[Route('/product/{id}',
        name: 'product_details',
        alias: new DeprecatedAlias(
            aliasName: 'product_show',
            package: 'acme/package',
            version: '1.2',
            message: 'The "%alias_id%" route alias is deprecated. Please use "product_details" instead.',
        ),
    )]
    public function show(): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
# Move the concrete route definition under ``product_details``
product_details:
    path: /product/{id}
    controller: App\Controller\ProductController::show

# Define the alias and the deprecation under the ``product_show`` definition
product_show:
    alias: product_details

    # this outputs the following generic deprecation message:
    # Since acme/package 1.2: The "product_show" route alias is deprecated. You should stop using it, as it will be removed in the future.
    deprecated:
        package: 'acme/package'
        version: '1.2'

    # or

    # you can define a custom deprecation message (%alias_id% placeholder is available)
    deprecated:
        package: 'acme/package'
        version: '1.2'
        message: 'The "%alias_id%" route alias is deprecated. Please use "product_details" instead.'
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\ProductController;

return Routes::config([
    // Move the concrete route definition under ``product_details``
    'product_details' => [
        'path' => '/product/{id}',
        'controller' => [ProductController::class, 'show'],
    ],
    // Define the alias and the deprecation under the ``product_show`` definition
    'product_show' => [
        'alias' => 'product_details',

        // this outputs the following generic deprecation message:
        // Since acme/package 1.2: The "product_show" route alias is deprecated. You should stop using it, as it will be removed in the future.
        'deprecated' => [
            'package' => 'acme/package',
            'version' => '1.2',
        ],

        // or

        // you can define a custom deprecation message (%alias_id% placeholder is available)
        'deprecated' => [
            'package' => 'acme/package',
            'version' => '1.2',
            'message' => 'The "%alias_id%" route alias is deprecated. Please use "product_details" instead.',
        ],
    ],
]);
```

在这个示例中，每次使用 `product_show` 别名时，都会触发废弃警告，建议你停止使用此路由，改用 `product_details`。

该消息实际上是一个消息模板，它将 `%alias_id%` 占位符的出现替换为路由别名名称。你的模板中**必须**至少有一个 `%alias_id%` 占位符。

## 路由组和前缀

一组路由共享某些选项是很常见的（例如，所有与博客相关的路由都以 `/blog` 开头）。这就是 Symfony 包含路由配置共享功能的原因。

当将路由定义为属性时，将公共配置放在控制器类的 `#[Route]` 属性中。在其他路由格式中，在导入路由时使用选项定义公共配置。

**属性**

```php
// src/Controller/BlogController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

#[Route('/blog', requirements: ['_locale' => 'en|es|fr'], name: 'blog_')]
class BlogController extends AbstractController
{
    #[Route('/{_locale}', name: 'index')]
    public function index(): Response
    {
        // ...
    }

    #[Route('/{_locale}/posts/{slug}', name: 'show')]
    public function show(string $slug): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
controllers:
    resource: routing.controllers
    # this is added to the beginning of all imported route URLs
    prefix: '/blog'
    # this is added to the beginning of all imported route names
    name_prefix: 'blog_'
    # these requirements are added to all imported routes
    requirements:
        _locale: 'en|es|fr'

    # An imported route with an empty URL will become "/blog/"
    # Uncomment this option to make that URL "/blog" instead
    # trailing_slash_on_root: false

    # you can optionally exclude some files/subdirectories when loading attributes
    # (the value must be a string or an array of PHP glob patterns)
    # exclude: '../src/Controller/{Debug*Controller.php}'
```

**PHP**

```php
// config/routes/attributes.php
namespace Symfony\Component\Routing\Loader\Configurator;

return Routes::config([
    'controllers' => [
        'resource' => '../../src/Controller/',
        'type' => 'attribute',
        // this is added to the beginning of all imported route URLs
        'prefix' => '/blog',
        // this is added to the beginning of all imported route names
        'name_prefix' => 'blog_',
        // these requirements are added to all imported routes
        'requirements' => ['_locale' => 'en|es|fr'],

        // An imported route with an empty URL will become "/blog/"
        // Uncomment this option to make that URL "/blog" instead
        // 'trailing_slash_on_root' => false,

        // you can optionally exclude some files/subdirectories when loading attributes
        // (the value must be a string or an array of PHP glob patterns)
        // 'exclude' => '../../src/Controller/{Debug*Controller.php}',
    ],
]);
```

> **警告：**
> `exclude` 选项仅在 `resource` 值为 glob 字符串时有效。如果你使用普通字符串（例如 `'../src/Controller'`），`exclude` 值将被忽略。

在这个示例中，`index()` 动作的路由将被称为 `blog_index`，其 URL 将是 `/blog/{_locale}`。`show()` 动作的路由将被称为 `blog_show`，其 URL 将是 `/blog/{_locale}/posts/{slug}`。两个路由还将验证 `_locale` 参数是否匹配类属性中定义的正则表达式。

> **注意：**
> 如果任何前缀路由定义了空路径，Symfony 将向其添加尾部斜杠。在前面的示例中，以 `/blog` 为前缀的空路径将导致 `/blog/` URL。如果你想避免这种行为，请将 `trailing_slash_on_root` 选项设置为 `false`（使用 PHP 属性时此选项不可用）：
>
> **YAML**
>
> ```yaml
> # config/routes.yaml
> controllers:
>     resource: routing.controllers
>     prefix:   '/blog'
>     trailing_slash_on_root: false
>     # ...
> ```
>
> **PHP**
>
> ```php
> // config/routes/attributes.php
> namespace Symfony\Component\Routing\Loader\Configurator;
>
> return Routes::config([
>     'controllers' => [
>         'resource' => '../../src/Controller/',
>         'type' => 'attribute',
>         'prefix' => '/blog',
>         'trailing_slash_on_root' => false,
>     ],
> ]);
> ```

> **参见：**
> Symfony 可以从不同来源导入路由，你甚至可以创建自己的路由加载器。

## 获取路由名称和参数

Symfony 创建的 `Request` 对象在"请求属性"中存储所有路由配置（例如名称和参数）。你可以通过 `Request` 对象在控制器中获取此信息：

```php
// src/Controller/BlogController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class BlogController extends AbstractController
{
    #[Route('/blog', name: 'blog_list')]
    public function list(Request $request): Response
    {
        $routeName = $request->attributes->get('_route');
        $routeParameters = $request->attributes->get('_route_params');

        // use this to get all the available attributes (not only routing ones):
        $allAttributes = $request->attributes->all();

        // ...
    }
}
```

在服务中，你可以通过注入 RequestStack 服务来获取此信息。在模板中，使用 Twig 全局 `app` 变量来获取当前路由名称（`app.current_route`）及其参数（`app.current_route_parameters`）。

## 特殊路由

Symfony 定义了一些特殊控制器，用于直接从路由配置渲染模板和重定向到其他路由，这样你就不必创建控制器动作。

### 直接从路由渲染模板

请阅读 Symfony 模板主文章中关于从路由渲染模板的部分。

### 直接从路由重定向到 URL 和路由

使用 `RedirectController` 重定向到其他路由和 URL：

**YAML**

```yaml
# config/routes.yaml
doc_shortcut:
    path: /doc
    controller: Symfony\Bundle\FrameworkBundle\Controller\RedirectController
    defaults:
        route: 'doc_page'
        # optionally you can define some arguments passed to the route
        page: 'index'
        version: 'current'
        # redirections are temporary by default (code 302) but you can make them permanent (code 301)
        permanent: true
        # add this to keep the original query string parameters when redirecting
        keepQueryParams: true
        # add this to keep the HTTP method when redirecting. The redirect status changes
        # * for temporary redirects, it uses the 307 status code instead of 302
        # * for permanent redirects, it uses the 308 status code instead of 301
        keepRequestMethod: true
        # add this to remove all original route attributes when redirecting
        ignoreAttributes: true
        # or specify which attributes to ignore:
        # ignoreAttributes: ['offset', 'limit']

legacy_doc:
    path: /legacy/doc
    controller: Symfony\Bundle\FrameworkBundle\Controller\RedirectController
    defaults:
        # this value can be an absolute path or an absolute URL
        path: 'https://legacy.example.com/doc'
        permanent: true
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use Symfony\Bundle\FrameworkBundle\Controller\RedirectController;

return Routes::config([
    'doc_shortcut' => [
        'path' => '/doc',
        'controller' => [RedirectController::class, 'doc_page'],
        'defaults' => [
            'route' => 'doc_page',
            // optionally you can define some arguments passed to the route
            'page' => 'index',
            'version' => 'current',
            // redirections are temporary by default (code 302) but you can make them permanent (code 301)
            'permanent' => true,
            // add this to keep the original query string parameters when redirecting
            'keepQueryParams' => true,
            // add this to keep the HTTP method when redirecting. The redirect status changes
            // * for temporary redirects, it uses the 307 status code instead of 302
            // * for permanent redirects, it uses the 308 status code instead of 301
            'keepRequestMethod' => true,
            // add this to remove all original route attributes when redirecting
            'ignoreAttributes' => true,
            // or specify which attributes to ignore:
            // 'ignoreAttributes' => ['offset', 'limit'],
        ],
    ],
    'legacy_doc' => [
        'path' => '/legacy/doc',
        'controller' => [RedirectController::class, 'legacy_doc'],
        'defaults' => [
            // this value can be an absolute path or an absolute URL
            'path' => 'https://legacy.example.com/doc',
            'permanent' => true,
        ],
    ],
]);
```

> **提示：**
> Symfony 还提供了一些在控制器内部重定向的实用工具。

#### 重定向带有尾部斜杠的 URL

从历史上看，URL 遵循 UNIX 惯例，在目录后添加尾部斜杠（例如 `https://example.com/foo/`），在文件后删除斜杠（`https://example.com/foo`）。虽然为两个 URL 提供不同的内容是可以的，但现在通常将两个 URL 视为相同的 URL 并在它们之间重定向。

Symfony 遵循此逻辑在带有和不带有尾部斜杠的 URL 之间重定向（但仅对 `GET` 和 `HEAD` 请求）：

| 路由 URL  | 如果请求的 URL 是 `/foo`              | 如果请求的 URL 是 `/foo/`             |
|-----------|---------------------------------------|---------------------------------------|
| `/foo`    | 匹配（`200` 状态响应）                | 执行 `301` 重定向到 `/foo`            |
| `/foo/`   | 执行 `301` 重定向到 `/foo/`           | 匹配（`200` 状态响应）                |

## 子域名路由

路由可以配置 `host` 选项，要求传入请求的 HTTP 主机匹配某个特定值。在以下示例中，两个路由匹配相同的路径（`/`），但其中一个只响应特定的主机名：

**属性**

```php
// src/Controller/MainController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class MainController extends AbstractController
{
    #[Route('/', name: 'mobile_homepage', host: 'm.example.com')]
    public function mobileHomepage(): Response
    {
        // ...
    }

    #[Route('/', name: 'homepage')]
    public function homepage(): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
mobile_homepage:
    path:       /
    host:       m.example.com
    controller: App\Controller\MainController::mobileHomepage

homepage:
    path:       /
    controller: App\Controller\MainController::homepage
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\MainController;

return Routes::config([
    'mobile_homepage' => [
        'path' => '/',
        'host' => 'm.example.com',
        'controller' => [MainController::class, 'mobileHomepage'],
    ],
    'homepage' => [
        'path' => '/',
        'controller' => [MainController::class, 'homepage'],
    ],
]);
```

`host` 选项的值可以包含参数（这在多租户应用程序中很有用），这些参数也可以使用 `requirements` 进行验证：

**属性**

```php
// src/Controller/MainController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class MainController extends AbstractController
{
    #[Route(
        '/',
        name: 'mobile_homepage',
        host: '{subdomain}.example.com',
        defaults: ['subdomain' => 'm'],
        requirements: ['subdomain' => 'm|mobile'],
    )]
    public function mobileHomepage(): Response
    {
        // ...
    }

    #[Route('/', name: 'homepage')]
    public function homepage(): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
mobile_homepage:
    path:       /
    host:       "{subdomain}.example.com"
    controller: App\Controller\MainController::mobileHomepage
    defaults:
        subdomain: m
    requirements:
        subdomain: m|mobile

homepage:
    path:       /
    controller: App\Controller\MainController::homepage
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\MainController;

return Routes::config([
    'mobile_homepage' => [
        'path' => '/',
        'host' => '{subdomain}.example.com',
        'controller' => [MainController::class, 'mobileHomepage'],
        'defaults' => ['subdomain' => 'm'],
        'requirements' => ['subdomain' => 'm|mobile'],
    ],
    'homepage' => [
        'path' => '/',
        'controller' => [MainController::class, 'homepage'],
    ],
]);
```

在上面的示例中，`subdomain` 参数定义了一个默认值，因为否则每次使用这些路由生成 URL 时都需要包含一个子域名值。

> **提示：**
> 在导入路由时，你也可以设置 `host` 选项，使所有路由都需要该主机名。

> **注意：**
> 使用子域名路由时，你必须在功能测试中设置 `Host` HTTP 头，否则路由将不会匹配：
>
> ```php
> $crawler = $client->request(
>     'GET',
>     '/',
>     [],
>     [],
>     ['HTTP_HOST' => 'm.example.com']
>     // or get the value from some configuration parameter:
>     // ['HTTP_HOST' => 'm.'.$client->getContainer()->getParameter('domain')]
> );
> ```

> **提示：**
> 你也可以在 `host` 选项中使用内联默认值和要求格式：`{subdomain<m|mobile>?m}.example.com`

## 本地化路由（i18n）

如果你的应用程序被翻译成多种语言，每个路由可以为每个翻译语言区域定义不同的 URL。这避免了重复路由的需要，同时也减少了潜在的错误：

**属性**

```php
// src/Controller/CompanyController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class CompanyController extends AbstractController
{
    #[Route(path: [
        'en' => '/about-us',
        'nl' => '/over-ons'
        // optionally, you can define a path without a locale. It will be used
        // for any locale that does not match the locales above
        '/about-us',
    ], name: 'about_us')]
    public function about(): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
about_us:
    path:
        en: /about-us
        nl: /over-ons
    controller: App\Controller\CompanyController::about
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\CompanyController;

return Routes::config([
    'about_us' => [
        'path' => [
            'en' => '/about-us',
            'nl' => '/over-ons',
            // optionally, you can define a path without a locale. It will be used
            // for any locale that does not match the locales above
            '/about-us',
        ],
        'controller' => [CompanyController::class, 'about'],
    ],
]);
```

> **注意：**
> 使用 PHP 属性进行本地化路由时，你必须使用 `path` 命名参数来指定路径数组。

当匹配到一个本地化路由时，Symfony 会在整个请求期间自动使用相同的语言区域。

> **提示：**
> 当应用程序使用完整的"语言 + 地区"语言区域（例如 `fr_FR`、`fr_BE`）时，如果所有相关语言区域的 URL 相同，路由可以只使用语言部分（例如 `fr`）来避免重复相同的 URL。

国际化应用程序的一个常见需求是为所有路由添加语言区域前缀。这可以通过为每个语言区域定义不同的前缀来实现（如果你愿意，可以为默认语言区域设置空前缀）：

**YAML**

```yaml
# config/routes.yaml
controllers:
    prefix:
        en: '' # don't prefix URLs for English, the default locale
        nl: '/nl'
    resource: routing.controllers
```

**PHP**

```php
// config/routes/attributes.php
namespace Symfony\Component\Routing\Loader\Configurator;

return Routes::config([
    'controllers' => [
        'resource' => '../../src/Controller/',
        'type' => 'attribute',
        'prefix' => [
            'en' => '', // don't prefix URLs for English, the default locale
            'nl' => '/nl',
        ],
    ],
]);
```

> **注意：**
> 如果被导入的路由在其自身定义中包含特殊的 `_locale` 参数，Symfony 只会为该语言区域导入它，而不会为其他配置的语言区域前缀导入。
>
> 例如，如果一个路由在其定义中包含 `locale: 'en'`，并且它正在以 `en`（前缀：空）和 `nl`（前缀：`/nl`）语言区域导入，那么该路由只会在 `en` 语言区域中可用，而不会在 `nl` 语言区域中可用。

另一个常见需求是根据语言区域在不同域名上托管网站。这可以通过为每个语言区域定义不同的主机来实现。

**YAML**

```yaml
# config/routes.yaml
controllers:
    resource: routing.controllers
    host:
        en: 'www.example.com'
        nl: 'www.example.nl'
```

**PHP**

```php
// config/routes/attributes.php
namespace Symfony\Component\Routing\Loader\Configurator;

return Routes::config([
    'controllers' => [
        'resource' => '../../src/Controller/',
        'type' => 'attribute',
        'host' => [
            'en' => 'www.example.com',
            'nl' => 'www.example.nl',
        ],
    ],
]);
```

## 无状态路由

有时，当 HTTP 响应应该被缓存时，确保这种情况发生是很重要的。然而，每当在请求期间启动会话时，Symfony 将响应变为不可缓存的私有响应。

有关详细信息，请参阅 HTTP 缓存文档。

路由可以配置一个 `stateless` 布尔选项，以声明在匹配请求时不应使用会话：

**属性**

```php
// src/Controller/MainController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Routing\Attribute\Route;

class MainController extends AbstractController
{
    #[Route('/', name: 'homepage', stateless: true)]
    public function homepage(): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
homepage:
    controller: App\Controller\MainController::homepage
    path: /
    stateless: true
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\MainController;

return Routes::config([
    'homepage' => [
        'controller' => [MainController::class, 'homepage'],
        'path' => '/',
        'stateless' => true,
    ],
]);
```

现在，如果使用了会话，应用程序将根据你的 `kernel.debug` 参数报告：

- `enabled`：将抛出 `Symfony\Component\HttpKernel\Exception\UnexpectedSessionUsageException` 异常
- `disabled`：将记录警告

这将帮助你理解并希望修复应用程序中的意外行为。

## 生成 URL

路由系统是双向的：

1. 它们将 URL 与控制器关联（如前面的章节所述）；
2. 它们为给定的路由生成 URL。

从路由生成 URL 允许你不在 HTML 模板中手动编写 `<a href="...">` 值。此外，如果某个路由的 URL 发生变化，你只需更新路由配置，所有链接都会更新。

要生成 URL，你需要指定路由的名称（例如 `blog_show`）和路由定义的参数值（例如 `slug = my-blog-post`）。

因此，每个路由都有一个内部名称，在应用程序中必须是唯一的。如果你没有使用 `name` 选项显式设置路由名称，Symfony 将根据控制器和动作生成自动名称。

如果目标类有一个添加路由的 `__invoke()` 方法**并且**目标类只添加了一个路由，Symfony 会基于 FQCN 声明路由别名。Symfony 还会为每个只定义了一个路由的方法自动添加别名。考虑以下类：

```php
// src/Controller/MainController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Routing\Attribute\Route;

final class MainController extends AbstractController
{
    #[Route('/', name: 'homepage')]
    public function homepage(): Response
    {
        // ...
    }
}
```

Symfony 将添加一个名为 `App\Controller\MainController::homepage` 的路由别名。

### 在控制器中生成 URL

如果你的控制器继承自 `AbstractController`，请使用 `generateUrl()` 辅助方法：

```php
// src/Controller/BlogController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Routing\Generator\UrlGeneratorInterface;

class BlogController extends AbstractController
{
    #[Route('/blog', name: 'blog_list')]
    public function list(): Response
    {
        // generate a URL with no route arguments
        $signUpPage = $this->generateUrl('sign_up');

        // generate a URL with route arguments
        $userProfilePage = $this->generateUrl('user_profile', [
            'username' => $user->getUserIdentifier(),
        ]);

        // generated URLs are "absolute paths" by default. Pass a third optional
        // argument to generate different URLs (e.g. an "absolute URL")
        $signUpPage = $this->generateUrl('sign_up', [], UrlGeneratorInterface::ABSOLUTE_URL);

        // when a route is localized, Symfony uses by default the current request locale
        // pass a different '_locale' value if you want to set the locale explicitly
        $signUpPageInDutch = $this->generateUrl('sign_up', ['_locale' => 'nl']);

        // ...
    }
}
```

> **注意：**
> 如果你向 `generateUrl()` 方法传递了一些不属于路由定义的参数，它们将作为查询字符串包含在生成的 URL 中：
>
> ```php
> $this->generateUrl('blog', ['page' => 2, 'category' => 'Symfony']);
> // the 'blog' route only defines the 'page' parameter; the generated URL is:
> // /blog/2?category=Symfony
> ```

> **警告：**
> 虽然对象在用作占位符时会被转换为字符串，但在用作额外参数时不会被转换。因此，如果你将一个对象（例如 Uuid）作为额外参数的值传递，你需要显式地将其转换为字符串：
>
> ```php
> $this->generateUrl('blog', ['uuid' => (string) $entity->getUuid()]);
> ```

如果你的控制器不继承自 `AbstractController`，你需要在控制器中获取服务，并按照下一节的说明进行操作。

### 在服务中生成 URL

将 `router` Symfony 服务注入到你自己的服务中，并使用其 `generate()` 方法。使用服务自动装配时，你只需在服务构造函数中添加一个参数，并使用 `Symfony\Component\Routing\Generator\UrlGeneratorInterface` 类进行类型提示：

```php
// src/Service/SomeService.php
namespace App\Service;

use Symfony\Component\Routing\Generator\UrlGeneratorInterface;

class SomeService
{
    public function __construct(
        private UrlGeneratorInterface $urlGenerator,
    ) {
    }

    public function someMethod(): void
    {
        // ...

        // generate a URL with no route arguments
        $signUpPage = $this->urlGenerator->generate('sign_up');

        // generate a URL with route arguments
        $userProfilePage = $this->urlGenerator->generate('user_profile', [
            'username' => $user->getUserIdentifier(),
        ]);

        // generated URLs are "absolute paths" by default. Pass a third optional
        // argument to generate different URLs (e.g. an "absolute URL")
        $signUpPage = $this->urlGenerator->generate('sign_up', [], UrlGeneratorInterface::ABSOLUTE_URL);

        // when a route is localized, Symfony uses by default the current request locale
        // pass a different '_locale' value if you want to set the locale explicitly
        $signUpPageInDutch = $this->urlGenerator->generate('sign_up', ['_locale' => 'nl']);
    }
}
```

### 在模板中生成 URL

请阅读 Symfony 模板主文章中关于在页面之间创建链接的部分。

### 在 JavaScript 中生成 URL

如果你的 JavaScript 代码包含在 Twig 模板中，你可以使用 `path()` 和 `url()` Twig 函数生成 URL，并将其存储在 JavaScript 变量中。`escape()` 过滤器用于转义任何非 JavaScript 安全的值：

```twig
<script>
    const route = "{{ path('blog_show', {slug: 'my-blog-post'})|escape('js') }}";
</script>
```

如果你需要动态生成 URL，或者使用的是纯 JavaScript 代码，此解决方案将不起作用。在这种情况下，请考虑使用 [FOSJsRoutingBundle](https://github.com/FriendsOfSymfony/FOSJsRoutingBundle)。

### 在命令中生成 URL

在命令中生成 URL 的工作方式与在服务中生成 URL 相同。唯一的区别是命令不在 HTTP 上下文中执行。因此，如果你生成绝对 URL，你将获得 `http://localhost/` 作为主机名，而不是你真实的主机名。

解决方案是配置 `default_uri` 选项，以定义命令生成 URL 时使用的"请求上下文"：

**YAML**

```yaml
# config/packages/routing.yaml
framework:
    router:
        # ...
        default_uri: 'https://example.org/my/path/'
```

**PHP**

```php
// config/packages/routing.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'router' => [
            'default_uri' => 'https://example.org/my/path/',
        ],
    ],
]);
```

现在，在命令中生成 URL 时，你将获得预期的结果：

```php
// src/Command/MyCommand.php
namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Style\SymfonyStyle;
use Symfony\Component\Routing\Generator\UrlGeneratorInterface;
// ...

#[AsCommand(name: 'app:my-command')]
class MyCommand
{
    public function __construct(
        private UrlGeneratorInterface $urlGenerator,
    ) {
    }

    public function __invoke(SymfonyStyle $io): int
    {
        // generate a URL with no route arguments
        $signUpPage = $this->urlGenerator->generate('sign_up');

        // generate a URL with route arguments
        $userProfilePage = $this->urlGenerator->generate('user_profile', [
            'username' => $user->getUserIdentifier(),
        ]);

        // by default, generated URLs are "absolute paths". Pass a third optional
        // argument to generate different URIs (e.g. an "absolute URL")
        $signUpPage = $this->urlGenerator->generate('sign_up', [], UrlGeneratorInterface::ABSOLUTE_URL);

        // when a route is localized, Symfony uses by default the current request locale
        // pass a different '_locale' value if you want to set the locale explicitly
        $signUpPageInDutch = $this->urlGenerator->generate('sign_up', ['_locale' => 'nl']);

        // ...
    }
}
```

> **注意：**
> 默认情况下，为网络资产生成的 URL 使用相同的 `default_uri` 值，但你可以使用 `asset.request_context.base_path` 和 `asset.request_context.secure` 容器参数来更改它。

> **注意：**
> 默认情况下，在 HTTP 上下文之外生成的路由使用默认语言区域作为 `_locale` 参数的值。你可以通过在生成每个路由时为 `_locale` 参数提供不同的值来覆盖此设置。

### 检查路由是否存在

在高度动态的应用程序中，在使用路由生成 URL 之前，可能需要检查路由是否存在。在这种情况下，不要使用 `Symfony\Component\Routing\Router::getRouteCollection` 方法，因为它会重新生成路由缓存并降低应用程序速度。

相反，尝试生成 URL 并捕获路由不存在时抛出的 `Symfony\Component\Routing\Exception\RouteNotFoundException`：

```php
use Symfony\Component\Routing\Exception\RouteNotFoundException;

// ...

try {
    $url = $this->router->generate($routeName, $routeParameters);
} catch (RouteNotFoundException $e) {
    // the route is not defined...
}
```

### 强制在生成的 URL 上使用 HTTPS

> **注意：**
> 如果你的服务器在终止 SSL 的代理后面运行，请确保配置 Symfony 以在代理后面工作。
>
> 协议的配置仅用于非 HTTP 请求。`schemes` 选项与不正确的代理配置一起使用将导致重定向循环。

默认情况下，生成的 URL 使用与当前请求相同的 HTTP 协议。在控制台命令中，没有 HTTP 请求，URL 默认使用 `http`。你可以通过每个命令（通过路由器的 `getContext()` 方法）或使用以下配置参数全局更改此行为：

**YAML**

```yaml
# config/services.yaml
parameters:
    router.request_context.scheme: 'https'
    asset.request_context.secure: true
```

**PHP**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'router.request_context.scheme' => 'https',
        'asset.request_context.secure' => true,
    ],
]);
```

在控制台命令之外，使用 `schemes` 选项显式定义每个路由的协议：

**属性**

```php
// src/Controller/SecurityController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class SecurityController extends AbstractController
{
    #[Route('/login', name: 'login', schemes: ['https'])]
    public function login(): Response
    {
        // ...
    }
}
```

**YAML**

```yaml
# config/routes.yaml
login:
    path:       /login
    controller: App\Controller\SecurityController::login
    schemes:    [https]
```

**PHP**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\SecurityController;

return Routes::config([
    'login' => [
        'path' => '/login',
        'controller' => [SecurityController::class, 'login'],
        'schemes' => ['https'],
    ],
]);
```

为 `login` 路由生成的 URL 将始终使用 HTTPS。这意味着当使用 `path()` Twig 函数生成 URL 时，如果原始请求的 HTTP 协议与路由使用的协议不同，你可能会获得绝对 URL 而不是相对 URL：

```twig
{# if the current scheme is HTTPS, generates a relative URL: /login #}
{{ path('login') }}

{# if the current scheme is HTTP, generates an absolute URL to change
   the scheme: https://example.com/login #}
{{ path('login') }}
```

对于传入请求，也会强制执行协议要求。如果你尝试使用 HTTP 访问 `/login` URL，你将自动被重定向到相同的 URL，但使用 HTTPS 协议。

如果你想强制一组路由使用 HTTPS，可以在导入它们时定义默认协议。以下示例对所有定义为属性的路由强制使用 HTTPS：

**YAML**

```yaml
# config/routes.yaml
controllers:
    schemes: [https]
    resource: routing.controllers
```

**PHP**

```php
// config/routes/attributes.php
namespace Symfony\Component\Routing\Loader\Configurator;

return Routes::config([
    'controllers' => [
        'resource' => '../../src/Controller/',
        'type' => 'attribute',
        'schemes' => ['https'],
    ],
]);
```

> **注意：**
> Security 组件提供了另一种通过 `requires_channel` 设置强制使用 HTTP 或 HTTPS 的方式。

### 签名 URI

签名 URI 是一个包含哈希值的 URI，该哈希值依赖于 URI 的内容。这样，你可以通过重新计算其哈希值并将其与 URI 中包含的哈希值进行比较来检查签名 URI 的完整性。

Symfony 通过 `Symfony\Component\HttpFoundation\UriSigner` 服务提供了签名 URI 的实用工具，你可以在服务或控制器中注入该服务：

```php
// src/Service/SomeService.php
namespace App\Service;

use Symfony\Component\HttpFoundation\UriSigner;

class SomeService
{
    public function __construct(
        private UriSigner $uriSigner,
    ) {
    }

    public function someMethod(): void
    {
        // ...

        // generate a URL yourself or get it somehow...
        $url = 'https://example.com/foo/bar?sort=desc';

        // sign the URL (it adds a query parameter called '_hash')
        $signedUrl = $this->uriSigner->sign($url);
        // $url = 'https://example.com/foo/bar?sort=desc&_hash=e4a21b9'

        // check the URL signature
        $uriSignatureIsValid = $this->uriSigner->check($signedUrl);
        // $uriSignatureIsValid = true

        // if you have access to the current Request object, you can use this
        // other method to pass the entire Request object instead of the URI:
        $uriSignatureIsValid = $this->uriSigner->checkRequest($request);
    }
}
```

出于安全原因，通常会使签名 URI 在一段时间后过期（例如在使用它们重置用户凭据时）。默认情况下，签名 URI 不会过期，但你可以使用 `Symfony\Component\HttpFoundation\UriSigner::sign` 的 `$expiration` 参数定义到期日期/时间：

```php
// src/Service/SomeService.php
namespace App\Service;

use Symfony\Component\HttpFoundation\UriSigner;

class SomeService
{
    public function __construct(
        private UriSigner $uriSigner,
    ) {
    }

    public function someMethod(): void
    {
        // ...

        // generate a URL yourself or get it somehow...
        $url = 'https://example.com/foo/bar?sort=desc';

        // sign the URL with an explicit expiration date
        $signedUrl = $this->uriSigner->sign($url, new \DateTimeImmutable('2050-01-01'));
        // $signedUrl = 'https://example.com/foo/bar?sort=desc&_expiration=2524608000&_hash=e4a21b9'

        // if you pass a \DateInterval, it will be added from now to get the expiration date
        $signedUrl = $this->uriSigner->sign($url, new \DateInterval('PT10S'));  // valid for 10 seconds from now
        // $signedUrl = 'https://example.com/foo/bar?sort=desc&_expiration=1712414278&_hash=e4a21b9'

        // you can also use a timestamp in seconds
        $signedUrl = $this->uriSigner->sign($url, 4070908800); // timestamp for the date 2099-01-01
        // $signedUrl = 'https://example.com/foo/bar?sort=desc&_expiration=4070908800&_hash=e4a21b9'
    }
}
```

> **注意：**
> 到期日期/时间通过 `_expiration` 查询参数作为时间戳包含在签名 URI 中。

如果你需要知道签名 URI 无效的原因，可以使用 `verify()` 方法，该方法在失败时抛出异常：

```php
use Symfony\Component\HttpFoundation\Exception\ExpiredSignedUriException;
use Symfony\Component\HttpFoundation\Exception\UnsignedUriException;
use Symfony\Component\HttpFoundation\Exception\UnverifiedSignedUriException;

// ...

try {
    $uriSigner->verify($uri); // $uri can be a string or Request object

    // the URI is valid
} catch (UnsignedUriException) {
    // the URI isn't signed
} catch (UnverifiedSignedUriException) {
    // the URI is signed but the signature is invalid
} catch (ExpiredSignedUriException) {
    // the URI is signed but expired
}
```

> **提示：**
> 如果安装了 `symfony/clock`，它将用于创建和验证过期时间。这允许你在测试中模拟当前时间。

验证传入请求的另一种方法是使用 `#[IsSignatureValid]` 属性。

在以下示例中，所有传入此控制器动作的请求都将被验证是否具有有效签名。如果签名缺失或无效，将抛出 `SignedUriException`：

```php
// src/Controller/SomeController.php
// ...

use Symfony\Component\HttpKernel\Attribute\IsSignatureValid;

#[IsSignatureValid]
public function someAction(): Response
{
    // ...
}
```

要将签名验证限制为特定的 HTTP 方法，请使用 `methods` 参数。这可以是一个字符串或方法数组：

```php
// Only validate POST requests
#[IsSignatureValid(methods: 'POST')]
public function createItem(): Response
{
    // ...
}

// Validate both POST and PUT requests
#[IsSignatureValid(methods: ['POST', 'PUT'])]
public function updateItem(): Response
{
    // ...
}
```

你也可以在控制器类级别应用 `#[IsSignatureValid]`。这样，控制器内的所有动作都将自动受到签名验证的保护：

```php
// src/Controller/SecureController.php
// ...

use Symfony\Component\HttpKernel\Attribute\IsSignatureValid;

#[IsSignatureValid]
class SecureController extends AbstractController
{
    public function index(): Response
    {
        // ...
    }

    public function submit(): Response
    {
        // ...
    }
}
```

此属性提供了一种声明性的方式，可以直接在控制器级别强制执行请求签名验证，有助于保持安全逻辑的一致性和可维护性。

## 故障排除

以下是使用路由时可能遇到的一些常见错误：

```text
Controller "App\\Controller\\BlogController::show()" requires that you
provide a value for the "$slug" argument.
```

当你的控制器方法有一个参数（例如 `$slug`）时会发生这种情况：

```php
public function show(string $slug): Response
{
    // ...
}
```

但你的路由路径中没有 `{slug}` 参数（例如它是 `/blog/show`）。在你的路由路径中添加 `{slug}`：`/blog/show/{slug}`，或为参数提供默认值（即 `$slug = null`）。

```text
Some mandatory parameters are missing ("slug") to generate a URL for route
"blog_show".
```

这意味着你正在尝试为 `blog_show` 路由生成 URL，但你没有传递 `slug` 值（这是必需的，因为路由路径中有 `{slug}` 参数）。要修复此问题，请在生成路由时传递 `slug` 值：

```php
$this->generateUrl('blog_show', ['slug' => 'slug-value']);
```

或者在 Twig 中：

```twig
{{ path('blog_show', {slug: 'slug-value'}) }}
```

## 深入了解路由
