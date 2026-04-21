# 如何创建自定义路由加载器

简单的应用可以在单个配置文件中定义所有路由——通常是 `config/routes.yaml`（参见路由创建相关章节）。然而，在大多数应用中，从不同资源导入路由定义是很常见的做法：控制器文件中的 PHP 属性、存储在某个目录中的 YAML 或 PHP 文件等。

## 内置路由加载器

Symfony 为最常见的需求提供了几种路由加载器：

```yaml
# config/routes.yaml
app_file:
    # 从存储在某个 Bundle 中的给定路由文件加载路由
    resource: '@AcmeBundle/Resources/config/routing.yaml'

app_psr4:
    # 从给定 PSR-4 命名空间根下找到的控制器的 PHP 属性加载路由
    resource:
        path: '../src/Controller/'
        namespace: App\Controller
    type: attribute

app_attributes:
    # 从该目录中找到的控制器的 PHP 属性加载路由
    resource: '../src/Controller/'
    type:     attribute

app_class_attributes:
    # 从给定类的 PHP 属性加载路由
    resource: App\Controller\MyController
    type:     attribute

app_directory:
    # 从该目录中找到的 YAML 或 PHP 文件加载路由
    resource: '../legacy/routing/'
    type:     directory

app_bundle:
    # 从某个 Bundle 目录中找到的 YAML 或 PHP 文件加载路由
    resource: '@AcmeOtherBundle/Resources/config/routing/'
    type:     directory
```

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

return Routes::config([
    'app_file' => [
        // 从存储在某个 Bundle 中的给定路由文件加载路由
        'resource' => '@AcmeBundle/Resources/config/routing.yaml',
    ],
    'app_psr4' => [
        // 从给定 PSR-4 命名空间根下找到的控制器的 PHP 属性加载路由
        'resource' => [
            'path' => '../src/Controller/',
            'namespace' => 'App\Controller',
        ],
        'type' => 'attribute',
    ],
    'app_attributes' => [
        // 从该目录中找到的控制器的 PHP 属性加载路由
        'resource' => '../src/Controller/',
        'type' => 'attribute',
    ],
    'app_class_attributes' => [
        // 从给定类的 PHP 属性加载路由
        'resource' => 'App\Controller\MyController',
        'type' => 'attribute',
    ],
    'app_directory' => [
        // 从该目录中找到的 YAML 或 PHP 文件加载路由
        'resource' => '../legacy/routing/',
        'type' => 'directory',
    ],
    'app_bundle' => [
        // 从某个 Bundle 目录中找到的 YAML 或 PHP 文件加载路由
        'resource' => '@AcmeOtherBundle/Resources/config/routing/',
        'type' => 'directory',
    ],
]);
```

> **注意：** 导入资源时，键（例如 `app_file`）是集合的名称。请确保每个文件中该键是唯一的，不会被其他行覆盖。

如果你的应用需求不同，可以按照下一节的说明创建自己的自定义路由加载器。

## 什么是自定义路由加载器

自定义路由加载器使你能够根据某些约定、模式或集成来生成路由。这种用例的一个例子是 [OpenAPI-Symfony-Routing](https://github.com/Tobion/OpenAPI-Symfony-Routing) 库，它根据 OpenAPI/Swagger 属性生成路由。另一个例子是 [SonataAdminBundle](https://github.com/sonata-project/SonataAdminBundle)，它根据 CRUD 约定创建路由。

## 加载路由

Symfony 应用中的路由由 `Symfony\Bundle\FrameworkBundle\Routing\DelegatingLoader` 加载。该加载器使用多个其他加载器（委托）来加载不同类型的资源，例如 YAML 文件或控制器文件中的 `#[Route]` 属性。专门的加载器实现 `Symfony\Component\Config\Loader\LoaderInterface`，因此有两个重要方法：`supports` 和 `load`。

以 `routes.yaml` 中的这几行为例：

```yaml
# config/routes.yaml
controllers:
    resource: ../src/Controller/
    type: attribute
```

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

return Routes::config([
    'controllers' => [
        'resource' => '../src/Controller/',
        'type' => 'attribute',
    ],
]);
```

当主加载器解析此配置时，它会尝试所有已注册的委托加载器，并以给定的资源（`../src/Controller/`）和类型（`attribute`）作为参数调用它们的 `supports` 方法。当某个加载器返回 `true` 时，将调用其 `load` 方法，该方法应返回一个包含 `Symfony\Component\Routing\Route` 对象的 `Symfony\Component\Routing\RouteCollection`。

> **注意：** 以这种方式加载的路由将被路由器缓存，与以默认格式（例如 YAML、PHP 文件）定义时相同。

## 使用自定义服务加载路由

使用普通的 Symfony 服务是以自定义方式加载路由的最简单方法。它比创建完整的自定义路由加载器容易得多，因此你应该始终先考虑这个选项。

为此，将 `type: service` 定义为已加载路由资源的类型，并配置要调用的服务和方法：

```yaml
# config/routes.yaml
admin_routes:
    resource: 'admin_route_loader::loadRoutes'
    type: service
```

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

return Routes::config([
    'admin_routes' => [
    'resource' => 'admin_route_loader::loadRoutes',
        'type' => 'service',
    ],
]);
```

在此示例中，路由通过调用 ID 为 `admin_route_loader` 的服务的 `loadRoutes()` 方法来加载。你的服务不需要继承或实现任何特殊类，但被调用的方法必须返回一个 `Symfony\Component\Routing\RouteCollection` 对象。

如果你使用自动配置，你的类应该实现 `Symfony\Bundle\FrameworkBundle\Routing\RouteLoaderInterface` 接口以自动打标签。如果你**未使用自动配置**，请手动为其添加 `routing.route_loader` 标签。

> **注意：** 使用服务路由加载器定义的路由将被框架自动缓存。因此，每当你的服务需要加载新路由时，不要忘记清除缓存。

> **提示：** 如果你的服务是可调用的（invokable），则无需指定要使用的方法。

## 创建自定义加载器

要从某个自定义来源加载路由（即从属性、YAML 或 PHP 文件以外的来源），你需要创建一个自定义路由加载器。该加载器必须实现 `Symfony\Component\Config\Loader\LoaderInterface`。

在大多数情况下，继承 `Symfony\Component\Config\Loader\Loader` 比自行实现 `Symfony\Component\Config\Loader\LoaderInterface` 更容易。

下面的示例加载器支持加载类型为 `extra` 的路由资源。类型名称不应与可能支持相同类型资源的其他加载器冲突。请根据你所做的事情取一个特定的名称。资源名称本身在示例中实际上并不使用：

```php
// src/Routing/ExtraLoader.php
namespace App\Routing;

use Symfony\Component\Config\Loader\Loader;
use Symfony\Component\Routing\Route;
use Symfony\Component\Routing\RouteCollection;

class ExtraLoader extends Loader
{
    private bool $isLoaded = false;

    public function load($resource, ?string $type = null): RouteCollection
    {
        if (true === $this->isLoaded) {
            throw new \RuntimeException('Do not add the "extra" loader twice');
        }

        $routes = new RouteCollection();

        // prepare a new route
        $path = '/extra/{parameter}';
        $defaults = [
            '_controller' => 'App\Controller\ExtraController::extra',
        ];
        $requirements = [
            'parameter' => '\d+',
        ];
        $route = new Route($path, $defaults, $requirements);

        // add the new route to the route collection
        $routeName = 'extraRoute';
        $routes->add($routeName, $route);

        $this->isLoaded = true;

        return $routes;
    }

    public function supports($resource, ?string $type = null): bool
    {
        return 'extra' === $type;
    }
}
```

确保你指定的控制器确实存在。在本例中，你需要在 `ExtraController` 中创建一个 `extra()` 方法：

```php
// src/Controller/ExtraController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;

class ExtraController extends AbstractController
{
    public function extra(mixed $parameter): Response
    {
        return new Response($parameter);
    }
}
```

现在为 `ExtraLoader` 定义一个服务：

```yaml
# config/services.yaml
services:
    # ...

    App\Routing\ExtraLoader:
        tags: [routing.loader]
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Routing\ExtraLoader;

return App::config([
    'services' => [
        ExtraLoader::class => [
            'tags' => ['routing.loader'],
        ],
    ],
]);
```

注意标签 `routing.loader`。所有带有此**标签**的服务都将被标记为潜在的路由加载器，并作为专门的路由加载器添加到 `routing.loader` **服务**中，该服务是 `Symfony\Bundle\FrameworkBundle\Routing\DelegatingLoader` 的实例。

### 使用自定义加载器

如果你没有做其他操作，你的自定义路由加载器将**不会**被调用。还需要在路由配置中添加几行：

```yaml
# config/routes.yaml
app_extra:
    resource: .
    type: extra
```

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

return Routes::config([
    'app_extra' => [
        'resource' => '.',
        'type' => 'extra',
    ],
]);
```

这里重要的部分是 `type` 键。其值应为 `extra`，因为这是 `ExtraLoader` 支持的类型，这将确保调用其 `load()` 方法。`resource` 键对 `ExtraLoader` 来说无关紧要，因此将其设置为 `.`（单个点）。

> **注意：** 使用自定义路由加载器定义的路由将被框架自动缓存。因此，每当你更改加载器类本身时，不要忘记清除缓存。

## 更高级的加载器

如果你的自定义路由加载器如上所示继承自 `Symfony\Component\Config\Loader\Loader`，你还可以利用所提供的解析器（`Symfony\Component\Config\Loader\LoaderResolver` 的实例）来加载二级路由资源。

你仍然需要实现 `supports` 和 `load`。每当你想要加载另一个资源时——例如 YAML 路由配置文件——可以调用 `import` 方法：

```php
// src/Routing/AdvancedLoader.php
namespace App\Routing;

use Symfony\Component\Config\Loader\Loader;
use Symfony\Component\Routing\RouteCollection;

class AdvancedLoader extends Loader
{
    public function load($resource, ?string $type = null): RouteCollection
    {
        $routes = new RouteCollection();

        $resource = '@ThirdPartyBundle/Resources/config/routes.yaml';
        $type = 'yaml';

        $importedRoutes = $this->import($resource, $type);

        $routes->addCollection($importedRoutes);

        return $routes;
    }

    public function supports($resource, ?string $type = null): bool
    {
        return 'advanced_extra' === $type;
    }
}
```

> **注意：** 导入的路由配置的资源名称和类型可以是路由配置加载器通常支持的任何内容（YAML、PHP、属性等）。

> **注意：** 对于更高级的用法，请查看 Symfony CMF 项目提供的 [ChainRouter](https://symfony.com/doc/current/cmf/components/routing/chain.html)。该路由器允许应用使用两个或多个组合路由器，例如在编写自定义路由器时继续使用默认的 Symfony 路由系统。
