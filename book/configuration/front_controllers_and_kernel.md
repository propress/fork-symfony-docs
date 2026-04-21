# 前端控制器、内核与环境的协同工作原理

[配置环境](../configuration.md#配置环境)章节介绍了 Symfony 如何使用不同的配置设置来运行应用的基础知识。本节将更深入地介绍当应用启动时发生了什么。要介入这个过程，你需要了解以下三个协同工作的部分：

* [前端控制器](#前端控制器)
* [内核类](#内核类)
* [环境](#环境)

> **注意：** 通常情况下，你不需要定义自己的前端控制器或 `Kernel` 类，因为 Symfony 提供了合理的默认实现。本文旨在说明内部发生了什么。

## 前端控制器

[前端控制器](https://en.wikipedia.org/wiki/Front_Controller_pattern)是一种设计模式；它是应用处理的*所有*请求都会经过的一段代码。

在 Symfony Skeleton 中，这个角色由 `public/` 目录中的 `index.php` 文件承担。这是处理请求时运行的第一个 PHP 脚本。

前端控制器的主要目的是创建 `Kernel` 的实例（稍后详述），让它处理请求，并将响应结果返回给浏览器。

因为每个请求都通过它路由，所以前端控制器可以在设置内核之前执行全局初始化，或用额外功能来装饰内核。示例包括：

* 配置自动加载器或添加额外的自动加载机制；
* 通过将内核包装在 HttpCache 实例中来添加 HTTP 级别的缓存；
* 启用 Debug 组件。

你可以通过在 URL 中添加前端控制器来选择使用哪个，例如：

```text
http://localhost/index.php/some/path/...
```

如你所见，该 URL 包含要用作前端控制器的 PHP 脚本。你可以使用它切换到位于 `public/` 目录中的自定义前端控制器。

> **另请参阅：** 你几乎永远不希望在 URL 中显示前端控制器。可以通过配置 Web 服务器来实现这一点，如[配置 Web 服务器](../setup/web_server_configuration.md)所示。

从技术上讲，在命令行上运行 Symfony 时使用的 `bin/console` 脚本也是一个前端控制器，只是它不用于 Web，而是用于命令行请求。

## 内核类

`Symfony\Component\HttpKernel\Kernel` 是 Symfony 的核心。它负责设置应用使用的所有 Bundle，并为其提供应用的配置。然后在其 `Symfony\Component\HttpKernel\HttpKernelInterface::handle` 方法中创建服务容器并处理请求。

Symfony 应用中使用的内核继承自 `Symfony\Component\HttpKernel\Kernel`，并使用 `Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait`。`Kernel` 类保留了 `Symfony\Component\HttpKernel\KernelInterface` 中一些未实现的方法，而 `MicroKernelTrait` 定义了若干抽象方法，因此你必须全部实现它们：

`Symfony\Component\HttpKernel\KernelInterface::registerBundles`
必须返回运行应用所需的所有 Bundle 的数组。

`Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait::configureRoutes`
向应用添加单个路由或路由集合（例如加载某些配置文件中定义的路由）。

`Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait::configureContainer`
从配置文件加载应用配置或使用 `loadFromExtension()` 方法，也可以注册新的容器参数和服务。

为了填补这些（小）空白，你的应用需要扩展 Kernel 类并使用 MicroKernelTrait 来实现这些方法。Symfony 默认在 `src/Kernel.php` 文件中提供该内核。

该类使用环境名称（传递给内核的构造方法，可通过 `Symfony\Component\HttpKernel\Kernel::getEnvironment` 获取）来决定启用哪些 Bundle。这个逻辑在 `registerBundles()` 中实现。

你可以自由创建自己的替代或附加 `Kernel` 变体。你所需要做的就是调整（或添加新的）前端控制器以使用新内核。

> **注意：** `Kernel` 的名称和位置不是固定的。当将[多个内核放入单个应用](./multiple_kernels.md)时，可能有必要添加额外的子目录，例如 `src/admin/AdminKernel.php` 和 `src/api/ApiKernel.php`。重要的是你的前端控制器能够创建适当内核的实例。

> **注意：** `Kernel` 还可以用于更多用途，例如[覆盖默认目录结构](./override_dir_structure.md)。但很可能你不需要通过拥有多个 `Kernel` 实现来动态改变这些内容。

### 调试模式

`Kernel` 构造函数的第二个参数指定应用是否应以"调试模式"运行。无论[配置环境](../configuration.md#配置环境)如何，Symfony 应用都可以在调试模式设置为 `true` 或 `false` 的情况下运行。

这会影响应用中的许多事情，例如在错误页面上显示堆栈跟踪，或者缓存文件是否在每次请求时动态重建。虽然不是必须的，但调试模式通常在 `dev` 和 `test` 环境中设置为 `true`，在 `prod` 环境中设置为 `false`。

与[配置环境](../configuration.md#选择活动环境)类似，你也可以使用 `.env` 文件启用/禁用调试模式：

```bash
# .env
# 设置为 1 以启用调试模式
APP_DEBUG=0
```

通过在运行命令之前传递 `APP_DEBUG` 值，可以为命令覆盖该值：

```terminal
# 使用 .env 文件中定义的调试模式
$ php bin/console command_name

# 忽略 .env 文件并为该命令启用调试模式
$ APP_DEBUG=1 php bin/console command_name
```

在内部，调试模式的值成为[服务容器](../service_container.md)中使用的 `kernel.debug` 参数。如果你查看应用配置文件，你会看到该参数被使用，例如用于开启 Twig 的调试模式：

```yaml
# config/packages/twig.yaml
twig:
    debug: '%kernel.debug%'
```

```php
// config/packages/twig.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'twig' => [
        'debug' => '%kernel.debug%',
    ],
]);
```

## 环境

如上所述，`Kernel` 必须实现另一个方法——`Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait::configureContainer`。该方法负责从正确的*环境*加载应用的配置。

[配置环境](../configuration.md#配置环境)允许你使用不同的配置执行相同的代码。Symfony 默认提供三个环境，分别名为 `dev`、`prod` 和 `test`。

从技术上讲，这些名称只是从前端控制器传递给 `Kernel` 构造函数的字符串。该名称随后可用于 `configureContainer()` 方法中，以决定加载哪些配置文件。

Symfony 的默认 `Kernel` 类通过先加载 `config/packages/*` 中的配置文件，再加载 `config/packages/ENVIRONMENT_NAME/` 中的配置文件来实现此方法。如果你需要以更复杂的方式加载配置，可以自由地以不同方式实现此方法。

### 环境与缓存目录

Symfony 在很多方面利用了缓存：应用配置、路由配置、Twig 模板等都被缓存为存储在文件系统文件中的 PHP 对象。

默认情况下，这些缓存文件大部分存储在 `var/cache/` 目录中。但是，每个环境会缓存自己的一组文件：

```text
your-project/
├─ var/
│  ├─ cache/
│  │  ├─ dev/   # *dev* 环境的缓存目录
│  │  └─ prod/  # *prod* 环境的缓存目录
│  ├─ ...
```

有时，在调试时，检查缓存文件以了解某些功能的工作原理可能很有帮助。此时，请记住查看你正在使用的环境目录（开发和调试时通常是 `dev/`）。虽然可能有所不同，`var/cache/dev/` 目录包含以下内容：

`App_KernelDevDebugContainer.php`
缓存的"服务容器"，代表缓存的应用配置。

`url_generating_routes.php`
生成 URL 时使用的缓存路由配置。

`url_matching_routes.php`
用于路由匹配的缓存配置——在此查看用于将传入 URL 匹配到不同路由的编译正则表达式逻辑。

`twig/`
此目录包含所有缓存的 Twig 模板。

> **注意：** 你可以更改缓存目录的位置和名称。有关更多信息，请阅读[覆盖目录结构](./override_dir_structure.md)一文。
