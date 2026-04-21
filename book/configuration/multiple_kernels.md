# 如何使用单个内核创建多个 Symfony 应用

在 Symfony 应用中，传入的请求通常由 `public/index.php` 中的前端控制器处理，该控制器实例化 `src/Kernel.php` 类以创建应用内核。这个内核加载 Bundle、配置，并处理请求以生成响应。

当前 Kernel 类的实现是单个应用的便捷默认方式。然而，它也可以管理多个应用。虽然 Kernel 通常以不同的配置基于各种[环境](../configuration.md#配置环境)运行同一个应用，但它也可以被调整为使用特定的 Bundle 和配置运行不同的应用。

以下是使用单个 Kernel 创建多个应用的一些常见使用场景：

* 定义了 API 的应用可以分为两个部分以提高性能。第一部分服务常规 Web 应用，第二部分专门响应 API 请求。这种方法需要为第二部分加载更少的 Bundle 并启用更少的功能，从而优化性能；
* 高度敏感的应用可以分为两部分以增强安全性。第一部分只加载与应用公开部分对应的路由。第二部分加载应用的其余部分，其访问受到 Web 服务器的保护；
* 单体应用可以逐渐转变为更分布式的架构，例如微服务。这种方法允许在仍然共享公共配置和组件的同时，无缝迁移大型应用。

## 将单个应用转变为多个应用

以下是将单个应用转换为支持多个应用所需的步骤：

1. 创建新应用；
2. 更新 Kernel 类以支持多个应用；
3. 添加新的 `APP_ID` 环境变量；
4. 更新前端控制器。

以下示例展示了如何为新 Symfony 项目的 API 创建新应用。

### 步骤一：创建新应用

此示例遵循[共享内核](http://ddd.fed.wiki.org/view/shared-kernel)模式：所有应用维护独立的上下文，但如果需要，它们可以共享公共的 Bundle、配置和代码。最优方法取决于你的具体需求和要求，因此由你决定哪种最适合你的项目。

首先，在项目根目录创建一个新的 `apps` 目录，用于保存所有必要的应用。每个应用将遵循类似于 [Symfony 最佳实践](../best_practices.md)中描述的简化目录结构：

```text
your-project/
├─ apps/
│  └─ api/
│     ├─ config/
│     │  ├─ bundles.php
│     │  ├─ routes.yaml
│     │  └─ services.yaml
│     └─ src/
├─ bin/
│  └─ console
├─ config/
├─ public/
│  └─ index.php
├─ src/
│  └─ Kernel.php
```

> **注意：** 注意项目根目录中的 `config/` 和 `src/` 目录将代表 `apps/` 目录中所有应用之间的共享上下文。因此，你应该仔细考虑什么是公共的，什么应该放在特定应用中。

> **提示：** 你也可以考虑将共享上下文的命名空间从 `App` 重命名为 `Shared`，这将使其更容易区分并为该上下文提供更清晰的含义。

由于新的 `apps/api/src/` 目录将托管与 API 相关的 PHP 代码，你需要更新 `composer.json` 文件以将其包含在自动加载部分中：

```json
{
    "autoload": {
        "psr-4": {
            "Shared\\": "src/",
            "Api\\": "apps/api/src/"
        }
    }
}
```

此外，不要忘记运行 `composer dump-autoload` 以生成自动加载文件。

### 步骤二：更新 Kernel 类以支持多个应用

由于将有多个应用，最好向 Kernel 添加一个新的 `string $id` 属性以标识正在加载的应用。此属性还允许你拆分缓存、日志和配置文件，以避免与其他应用发生冲突。此外，它有助于性能优化，因为每个应用只会加载所需的资源：

```php
// src/Kernel.php
namespace Shared;

use Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\Component\HttpKernel\Kernel as BaseKernel;
use Symfony\Component\Routing\Loader\Configurator\RoutingConfigurator;

class Kernel extends BaseKernel
{
    use MicroKernelTrait { getConfigDir as getSharedConfigDir; }

    public function __construct(string $environment, bool $debug, private string $id)
    {
        parent::__construct($environment, $debug);
    }

    public function getAppConfigDir(): string
    {
        return $this->getProjectDir().'/apps/'.$this->id.'/config';
    }

    public function registerBundles(): iterable
    {
        $sharedBundles = require $this->getSharedConfigDir().'/bundles.php';
        $appBundles = require $this->getAppConfigDir().'/bundles.php';

        // load common bundles, such as the FrameworkBundle, as well as
        // specific bundles required exclusively for the app itself
        foreach (array_merge($sharedBundles, $appBundles) as $class => $envs) {
            if ($envs[$this->environment] ?? $envs['all'] ?? false) {
                yield new $class();
            }
        }
    }

    public function getCacheDir(): string
    {
        // divide cache for each application
        return ($_SERVER['APP_CACHE_DIR'] ?? $this->getProjectDir().'/var/cache').'/'.$this->id.'/'.$this->environment;
    }

    public function getLogDir(): string
    {
        // divide logs for each application
        return ($_SERVER['APP_LOG_DIR'] ?? $this->getProjectDir().'/var/log').'/'.$this->id;
    }

    protected function configureContainer(ContainerConfigurator $container): void
    {
        // load common config files, such as the framework.yaml, as well as
        // specific configs required exclusively for the app itself
        $this->doConfigureContainer($container, $this->getSharedConfigDir());
        $this->doConfigureContainer($container, $this->getAppConfigDir());
    }

    protected function configureRoutes(RoutingConfigurator $routes): void
    {
        // load common routes files, such as the routes/framework.yaml, as well as
        // specific routes required exclusively for the app itself
        $this->doConfigureRoutes($routes, $this->getSharedConfigDir());
        $this->doConfigureRoutes($routes, $this->getAppConfigDir());
    }

    private function doConfigureContainer(ContainerConfigurator $container, string $configDir): void
    {
        $container->import($configDir.'/{packages}/*.{php,yaml}');
        $container->import($configDir.'/{packages}/'.$this->environment.'/*.{php,yaml}');

        if (is_file($configDir.'/services.yaml')) {
            $container->import($configDir.'/services.yaml');
            $container->import($configDir.'/{services}_'.$this->environment.'.yaml');
        } else {
            $container->import($configDir.'/{services}.php');
        }
    }

    private function doConfigureRoutes(RoutingConfigurator $routes, string $configDir): void
    {
        $routes->import($configDir.'/{routes}/'.$this->environment.'/*.{php,yaml}');
        $routes->import($configDir.'/{routes}/*.{php,yaml}');

        if (is_file($configDir.'/routes.yaml')) {
            $routes->import($configDir.'/routes.yaml');
        } else {
            $routes->import($configDir.'/{routes}.php');
        }

        if (false !== ($fileName = (new \ReflectionObject($this))->getFileName())) {
            $routes->import($fileName, 'attribute');
        }
    }
}
```

此示例重用了默认实现，根据给定的配置目录导入配置和路由。如前所示，这种方法将同时导入共享资源和应用特定资源。

### 步骤三：添加新的 APP_ID 环境变量

接下来，定义一个标识当前应用的新环境变量。可以将这个新变量添加到 `.env` 文件以提供默认值，但通常应将其添加到你的 Web 服务器配置中。

```bash
# .env
APP_ID=api
```

> **警告：** 此变量的值必须与 `apps/` 中的应用目录匹配，因为它在 Kernel 中用于加载特定的应用配置。

### 步骤四：更新前端控制器

在最后一步中，更新前端控制器 `public/index.php` 和 `bin/console` 以将 `APP_ID` 变量的值传递给 Kernel 实例。这将允许 Kernel 加载并运行指定的应用：

```php
// public/index.php
use Shared\Kernel;
// ...

return function (array $context): Kernel {
    return new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG'], $context['APP_ID']);
};
```

与配置所需的 `APP_ENV` 和 `APP_DEBUG` 值类似，现在 Kernel 构造函数的第三个参数也是必要的，用于设置从外部配置派生的应用 ID。

对于第二个前端控制器，定义一个新的 console 选项，以允许在 CLI 上下文中传递应用 ID：

```php
// bin/console
use Shared\Kernel;
use Symfony\Bundle\FrameworkBundle\Console\Application;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;

return function (InputInterface $input, array $context): Application {
    $kernel = new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG'], $input->getParameterOption(['--id', '-i'], $context['APP_ID']));

    $application = new Application($kernel);
    $application->getDefinition()
        ->addOption(new InputOption('--id', '-i', InputOption::VALUE_REQUIRED, 'The App ID'))
    ;

    return $application;
};
```

就这些！

## 执行命令

`bin/console` 脚本（用于运行 Symfony 命令）始终使用 `Kernel` 类来构建应用并加载命令。如果你需要为特定应用运行 console 命令，可以提供 `--id` 选项以及适当的标识值：

```terminal
php bin/console cache:clear --id=api
// or
php bin/console cache:clear -iapi

// alternatively
export APP_ID=api
php bin/console cache:clear
```

你可能想要更新 composer 的 auto-scripts 部分以同时运行多个命令。此示例展示了两个不同应用（分别名为 `api` 和 `admin`）的命令：

```json
{
    "scripts": {
        "auto-scripts": {
            "cache:clear -iapi": "symfony-cmd",
            "cache:clear -iadmin": "symfony-cmd",
            "assets:install %PUBLIC_DIR% -iapi": "symfony-cmd",
            "assets:install %PUBLIC_DIR% -iadmin --no-cleanup": "symfony-cmd"
        }
    }
}
```

然后，运行 `composer auto-scripts` 来测试它！

> **注意：** 每个 console 脚本（例如 `bin/console -iapi` 和 `bin/console -iadmin`）可用的命令可能不同，因为它们取决于为每个应用启用的 Bundle，这可能是不同的。

## 渲染模板

假设你需要创建另一个名为 `admin` 的应用。如果你遵循 [Symfony 最佳实践](../best_practices.md)，共享 Kernel 模板将位于项目根目录的 `templates/` 目录中。对于 admin 特定的模板，你可以创建一个新目录 `apps/admin/templates/`，需要在 Admin 应用下手动配置它：

```yaml
# apps/admin/config/packages/twig.yaml
twig:
    paths:
        '%kernel.project_dir%/apps/admin/templates': Admin
```

然后，使用这个 Twig 命名空间来引用仅在 Admin 应用中的任何模板，例如 `@Admin/form/fields.html.twig`。

## 运行测试

在 Symfony 应用中，功能测试通常默认继承自 `Symfony\Bundle\FrameworkBundle\Test\WebTestCase` 类。在其父类 `KernelTestCase` 中，有一个名为 `createKernel()` 的方法，它尝试创建负责在测试期间运行应用的内核。然而，此方法的当前逻辑不包括新的应用 ID 参数，因此你需要更新它：

```php
// apps/api/tests/ApiTestCase.php
namespace Api\Tests;

use Shared\Kernel;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
use Symfony\Component\HttpKernel\KernelInterface;

class ApiTestCase extends WebTestCase
{
    protected static function createKernel(array $options = []): KernelInterface
    {
        $env = $options['environment'] ?? $_ENV['APP_ENV'] ?? $_SERVER['APP_ENV'] ?? 'test';
        $debug = $options['debug'] ?? (bool) ($_ENV['APP_DEBUG'] ?? $_SERVER['APP_DEBUG'] ?? true);

        return new Kernel($env, $debug, 'api');
    }
}
```

> **注意：** 此示例使用硬编码的应用 ID 值，因为继承此 `ApiTestCase` 类的测试将专注于 `api` 测试。

现在，在 `apps/api/` 应用中创建一个 `tests/` 目录。然后，更新 `composer.json` 文件和 `phpunit.xml` 配置以包含它：

```json
{
    "autoload-dev": {
        "psr-4": {
            "Shared\\Tests\\": "tests/",
            "Api\\Tests\\": "apps/api/tests/"
        }
    }
}
```

记得运行 `composer dump-autoload` 以生成自动加载文件。

以下是 `phpunit.xml` 文件所需的更新：

```xml
<testsuites>
    <testsuite name="shared">
        <directory>tests</directory>
    </testsuite>
    <testsuite name="api">
        <directory>apps/api/tests</directory>
    </testsuite>
</testsuites>
```

## 添加更多应用

现在你可以根据需要开始添加更多应用，例如用于管理项目配置和权限的 `admin` 应用。要做到这一点，你只需要重复步骤一：

```text
your-project/
├─ apps/
│  ├─ admin/
│  │  ├─ config/
│  │  │  ├─ bundles.php
│  │  │  ├─ routes.yaml
│  │  │  └─ services.yaml
│  │  └─ src/
│  └─ api/
│     └─ ...
```

此外，你可能需要更新 Web 服务器配置，在不同域名下设置 `APP_ID=admin`。
