# 使用 MicroKernelTrait 构建自己的框架

Symfony 应用中包含的默认 `Kernel` 类使用 `Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait` 在同一个类中配置 Bundle、路由和服务容器。

这种微内核方法非常灵活，允许你控制应用的结构和功能。

## 单文件 Symfony 应用

从一个完全空的目录开始，通过 Composer 安装以下 Symfony 组件：

```terminal
$ composer require symfony/framework-bundle symfony/runtime
```

接下来，创建一个 `index.php` 文件，定义内核类并运行它：

```php-attributes
// index.php
use Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Kernel as BaseKernel;
use Symfony\Component\Routing\Attribute\Route;

require_once dirname(__DIR__).'/vendor/autoload_runtime.php';

class Kernel extends BaseKernel
{
    use MicroKernelTrait;

    protected function configureContainer(ContainerConfigurator $container): void
    {
        // PHP equivalent of config/packages/framework.yaml
        $container->extension('framework', [
            'secret' => 'S0ME_SECRET'
        ]);
    }

    #[Route('/random/{limit}', name: 'random_number')]
    public function randomNumber(int $limit): JsonResponse
    {
        return new JsonResponse([
            'number' => random_int(0, $limit),
        ]);
    }
}

return static function (array $context) {
    return new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG']);
};
```

```php
// index.php
use Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Kernel as BaseKernel;
use Symfony\Component\Routing\Loader\Configurator\RoutingConfigurator;

require_once dirname(__DIR__).'/vendor/autoload_runtime.php';

class Kernel extends BaseKernel
{
    use MicroKernelTrait;

    protected function configureContainer(ContainerConfigurator $container): void
    {
        // PHP equivalent of config/packages/framework.yaml
        $container->extension('framework', [
            'secret' => 'S0ME_SECRET'
        ]);
    }

    protected function configureRoutes(RoutingConfigurator $routes): void
    {
        $routes->add('random_number', '/random/{limit}')->controller([$this, 'randomNumber']);
    }

    public function randomNumber(int $limit): JsonResponse
    {
        return new JsonResponse([
            'number' => random_int(0, $limit),
        ]);
    }
}

return static function (array $context) {
    return new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG']);
};
```

就这些！要测试它，启动 Symfony 本地 Web 服务器：

```terminal
$ symfony server:start
```

然后在浏览器中查看 JSON 响应：http://localhost:8000/random/10

> **提示：** 如果你的内核只定义了一个控制器，可以使用可调用方法：
>
> ```php
> class Kernel extends BaseKernel
> {
>     use MicroKernelTrait;
>
>     // ...
>
>     #[Route('/random/{limit}', name: 'random_number')]
>     public function __invoke(int $limit): JsonResponse
>     {
>         // ...
>     }
> }
> ```

## "微"内核的方法

使用 `MicroKernelTrait` 时，你的内核需要恰好三个方法来定义你的 Bundle、服务和路由：

**registerBundles()**
这与普通内核中看到的 `registerBundles()` 相同。默认情况下，微内核只注册 `FrameworkBundle`。如果需要注册更多 Bundle，请覆盖此方法：

```php
use Symfony\Bundle\FrameworkBundle\FrameworkBundle;
use Symfony\Bundle\TwigBundle\TwigBundle;
// ...

class Kernel extends BaseKernel
{
    use MicroKernelTrait;

    // ...

    public function registerBundles(): array
    {
        yield new FrameworkBundle();
        yield new TwigBundle();
    }
}
```

**configureContainer(ContainerConfigurator $container)**
此方法构建和配置容器。实际上，你将使用 `extension()` 来配置不同的 Bundle（这等同于你在普通 `config/packages/*` 文件中看到的内容）。你也可以直接在 PHP 中注册服务或加载外部配置文件（如下所示）。

**configureRoutes(RoutingConfigurator $routes)**
在此方法中，你可以使用 `RoutingConfigurator` 对象在应用中定义路由，并将它们与同一文件中定义的控制器关联。

但是，使用 PHP 属性定义控制器路由更为方便，如上所示。这就是为什么此方法通常仅用于加载外部路由文件（例如来自 Bundle 的路由文件），如下所示。

## 向"微"内核添加接口

使用 `MicroKernelTrait` 时，你也可以实现 `CompilerPassInterface` 以自动将内核本身注册为编译器传递，如专门的[编译器传递部分](../service_container.md#内核作为编译器传递)所述。如果在使用 `MicroKernelTrait` 时实现了 `Symfony\Component\DependencyInjection\Extension\ExtensionInterface`，那么内核将自动注册为扩展。你可以在关于[使用扩展管理配置](../service_container.md#使用扩展管理配置)的专门章节中了解更多内容。

还可以实现 `EventSubscriberInterface` 以直接从内核处理事件，它也会自动注册：

```php
// ...
use App\Exception\Danger;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;
use Symfony\Component\HttpKernel\KernelEvents;

class Kernel extends BaseKernel implements EventSubscriberInterface
{
    use MicroKernelTrait;

    // ...

    public function onKernelException(ExceptionEvent $event): void
    {
        if ($event->getThrowable() instanceof Danger) {
            $event->setResponse(new Response('It\'s dangerous to go alone. Take this ⚔'));
        }
    }

    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::EXCEPTION => 'onKernelException',
        ];
    }
}
```

## 高级示例：Twig、属性与 Web 调试工具栏

`MicroKernelTrait` 的目的*不是*拥有单文件应用。相反，其目标是赋予你选择 Bundle 和结构的能力。

首先，你可能希望将 PHP 类放在 `src/` 目录中。配置 `composer.json` 文件以从那里加载：

```json
{
    "require": {
        "...": "..."
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

然后，运行 `composer dump-autoload` 以转储新的自动加载配置。

现在，假设你想为应用定义自定义配置、使用 Twig 并通过属性加载路由。与其将*所有内容*放在 `index.php` 中，不如创建一个新的 `src/Kernel.php` 来保存内核。现在它看起来像这样：

```php
// src/Kernel.php
namespace App;

use App\DependencyInjection\AppExtension;
use Symfony\Bundle\FrameworkBundle\FrameworkBundle;
use Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait;
use Symfony\Bundle\TwigBundle\TwigBundle;
use Symfony\Bundle\WebProfilerBundle\WebProfilerBundle;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\Component\HttpKernel\Kernel as BaseKernel;
use Symfony\Component\Routing\Loader\Configurator\RoutingConfigurator;

class Kernel extends BaseKernel
{
    use MicroKernelTrait;

    public function registerBundles(): iterable
    {
        yield new FrameworkBundle();
        yield new TwigBundle();

        if ('dev' === $this->getEnvironment()) {
            yield new WebProfilerBundle();
        }
    }

    protected function build(ContainerBuilder $containerBuilder): void
    {
        $containerBuilder->registerExtension(new AppExtension());
    }

    protected function configureContainer(ContainerConfigurator $container): void
    {
        $container->import(__DIR__.'/../config/framework.yaml');

        // register all classes in /src/ as service
        $container->services()
            ->load('App\\', __DIR__.'/*')
            ->autowire()
            ->autoconfigure()
        ;

        // configure WebProfilerBundle only if the bundle is enabled
        if (isset($this->bundles['WebProfilerBundle'])) {
            $container->extension('web_profiler', [
                'toolbar' => true,
                'intercept_redirects' => false,
            ]);
        }
    }

    protected function configureRoutes(RoutingConfigurator $routes): void
    {
        // import the WebProfilerRoutes, only if the bundle is enabled
        if (isset($this->bundles['WebProfilerBundle'])) {
            $routes->import('@WebProfilerBundle/Resources/config/routing/wdt.php', 'php')->prefix('/_wdt');
            $routes->import('@WebProfilerBundle/Resources/config/routing/profiler.php', 'php')->prefix('/_profiler');
        }

        // load the routes defined as PHP attributes
        $routes->import(__DIR__.'/Controller/', 'attribute');
    }

    // optionally, you can define the getCacheDir() and getLogDir() methods
    // to override the default locations for these directories
}
```

继续之前，运行以下命令以添加对新依赖项的支持：

```terminal
$ composer require symfony/yaml symfony/twig-bundle symfony/web-profiler-bundle
```

接下来，创建一个新的扩展类，定义你的应用配置，并根据 `foo` 值有条件地添加服务：

```php
// src/DependencyInjection/AppExtension.php
namespace App\DependencyInjection;

use Symfony\Component\Config\Definition\Configurator\DefinitionConfigurator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Extension\AbstractExtension;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

class AppExtension extends AbstractExtension
{
    public function configure(DefinitionConfigurator $definition): void
    {
        $definition->rootNode()
            ->children()
                ->booleanNode('foo')->defaultTrue()->end()
            ->end();
    }

    public function loadExtension(array $config, ContainerConfigurator $containerConfigurator, ContainerBuilder $containerBuilder): void
    {
        if ($config['foo']) {
            $containerBuilder->register('foo_service', \stdClass::class);
        }
    }
}
```

与之前的内核不同，这个内核加载一个外部的 `config/framework.yaml` 文件，因为配置开始变大：

```yaml
# config/framework.yaml
framework:
    secret: S0ME_SECRET
    profiler: { only_exceptions: false }
```

```php
// config/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
    'secret' => 'SOME_SECRET',
        'profiler' => ['only_exceptions' => false],
    ],
]);
```

这也从 `src/Controller/` 目录加载属性路由，该目录中有一个文件：

```php
// src/Controller/MicroController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class MicroController extends AbstractController
{
    #[Route('/random/{limit}')]
    public function randomNumber(int $limit): Response
    {
        $number = random_int(0, $limit);

        return $this->render('micro/random.html.twig', [
            'number' => $number,
        ]);
    }
}
```

模板文件应位于项目根目录的 `templates/` 目录中。此模板位于 `templates/micro/random.html.twig`：

```html+twig
<!-- templates/micro/random.html.twig -->
<!DOCTYPE html>
<html>
    <head>
        <title>Random action</title>
    </head>
    <body>
        <p>{{ number }}</p>
    </body>
</html>
```

最后，你需要一个前端控制器来启动和运行应用。创建 `public/index.php`：

```php
// public/index.php
use App\Kernel;
use Symfony\Component\HttpFoundation\Request;

require __DIR__.'/../vendor/autoload.php';

$kernel = new Kernel('dev', true);
$request = Request::createFromGlobals();
$response = $kernel->handle($request);
$response->send();
$kernel->terminate($request, $response);
```

就这些！`/random/10` URL 将正常工作，Twig 将渲染，你甚至会在底部看到 Web 调试工具栏。最终结构如下所示：

```text
your-project/
├─ config/
│  └─ framework.yaml
├─ public/
|  └─ index.php
├─ src/
|  ├─ Controller
|  |  └─ MicroController.php
|  └─ Kernel.php
├─ templates/
|  └─ micro/
|     └─ random.html.twig
├─ var/
|  ├─ cache/
│  └─ log/
├─ vendor/
│  └─ ...
├─ composer.json
└─ composer.lock
```

与之前一样，你可以使用 Symfony 本地 Web 服务器：

```terminal
$ symfony server:start
```

然后在浏览器中访问该页面：http://localhost:8000/random/10
