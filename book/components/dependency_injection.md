# DependencyInjection 组件

DependencyInjection 组件实现了一个兼容 [PSR-11] 的服务容器，允许你以标准化和集中化的方式构建应用程序中的对象。

有关依赖注入和服务容器的介绍，请参阅 [/service_container](service_container.md)。

## 安装

```terminal
$ composer require symfony/dependency-injection
```

## 基本用法

> **参见：**
> 本文介绍了如何在任意 PHP 应用程序中将 DependencyInjection 功能作为独立组件使用。阅读 [/service_container](service_container.md) 文章以了解如何在 Symfony 应用程序中使用它。

假设你有一个如下所示的 `Mailer` 类，想将其作为服务提供：

```php
class Mailer
{
    private string $transport;

    public function __construct()
    {
        $this->transport = 'sendmail';
    }

    // ...
}
```

你可以将其注册为容器中的服务：

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;

$container = new ContainerBuilder();
$container->register('mailer', 'Mailer');
```

改进此类使其更灵活的方法是让容器设置所使用的 `transport`。如果将该类更改为通过构造函数传入：

```php
class Mailer
{
    public function __construct(
        private string $transport,
    ) {
    }

    // ...
}
```

然后在容器中设置传输方式的选择：

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;

$container = new ContainerBuilder();
$container
    ->register('mailer', 'Mailer')
    ->addArgument('sendmail');
```

现在这个类灵活多了，因为你已将传输方式的选择从实现中分离出来放入容器。

你选择的邮件传输方式可能是其他服务需要了解的。你可以通过将其设置为容器中的参数来避免在多个地方更改它，然后在 `Mailer` 服务的构造函数参数中引用该参数：

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;

$container = new ContainerBuilder();
$container->setParameter('mailer.transport', 'sendmail');
$container
    ->register('mailer', 'Mailer')
    ->addArgument('%mailer.transport%');
```

现在 `mailer` 服务已在容器中，可以将其作为其他类的依赖项注入。如果你有一个 `NewsletterManager` 类：

```php
class NewsletterManager
{
    public function __construct(
        private \Mailer $mailer,
    ) {
    }

    // ...
}
```

在定义 `newsletter_manager` 服务时，`mailer` 服务尚不存在。使用 `Reference` 类告诉容器在初始化 newsletter manager 时注入 `mailer` 服务：

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Reference;

$container = new ContainerBuilder();

$container->setParameter('mailer.transport', 'sendmail');
$container
    ->register('mailer', 'Mailer')
    ->addArgument('%mailer.transport%');

$container
    ->register('newsletter_manager', 'NewsletterManager')
    ->addArgument(new Reference('mailer'));
```

如果 `NewsletterManager` 不需要 `Mailer` 而注入它只是可选的，则可以使用 setter 注入：

```php
class NewsletterManager
{
    private \Mailer $mailer;

    public function setMailer(\Mailer $mailer): void
    {
        $this->mailer = $mailer;
    }

    // ...
}
```

现在你可以选择不向 `NewsletterManager` 注入 `Mailer`。但如果你确实想注入，容器可以调用 setter 方法：

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Reference;

$container = new ContainerBuilder();

$container->setParameter('mailer.transport', 'sendmail');
$container
    ->register('mailer', 'Mailer')
    ->addArgument('%mailer.transport%');

$container
    ->register('newsletter_manager', 'NewsletterManager')
    ->addMethodCall('setMailer', [new Reference('mailer')]);
```

然后你可以从容器中获取 `newsletter_manager` 服务：

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;

$container = new ContainerBuilder();

// ...

$newsletterManager = $container->get('newsletter_manager');
```

### 获取不存在的服务

默认情况下，当你尝试获取不存在的服务时，会看到一个异常。你可以按如下方式覆盖此行为：

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\ContainerInterface;

$containerBuilder = new ContainerBuilder();

// ...

// the second argument is optional and defines what to do when the service doesn't exist
$newsletterManager = $containerBuilder->get('newsletter_manager', ContainerInterface::EXCEPTION_ON_INVALID_REFERENCE);
```

以下是所有可能的行为：

* `ContainerInterface::EXCEPTION_ON_INVALID_REFERENCE`：在编译时抛出异常（这是**默认**行为）；
* `ContainerInterface::RUNTIME_EXCEPTION_ON_INVALID_REFERENCE`：在运行时、尝试访问缺失服务时抛出异常；
* `ContainerInterface::NULL_ON_INVALID_REFERENCE`：返回 `null`；
* `ContainerInterface::IGNORE_ON_INVALID_REFERENCE`：忽略请求引用的包装命令（例如，如果服务不存在则忽略 setter）；
* `ContainerInterface::IGNORE_ON_UNINITIALIZED_REFERENCE`：忽略/返回未初始化服务或无效引用的 `null`。

## 避免代码依赖于容器

虽然你可以直接从容器中检索服务，但最好尽量减少这种做法。例如，在 `NewsletterManager` 中，你将 `mailer` 服务注入其中，而不是从容器中请求它。你本可以注入容器并从中检索 `mailer` 服务，但这样会将其与这个特定容器绑定，使该类难以在其他地方复用。

你将需要在某些时候从容器中获取服务，但这应该尽量少，并且在应用程序的入口点处进行。

## 使用配置文件设置容器

除了使用 PHP 设置服务外，还可以使用配置文件。这允许你使用 YAML 或 PHP 编写服务定义，而不是像上面示例那样用 PHP 定义服务。在除了最小应用程序之外的所有情况下，通过将服务定义移到一个或多个配置文件中来组织它们是合理的。为此，你还需要安装 [Config 组件](config.md)。

加载 PHP 配置文件：

```php
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\PhpFileLoader;

$container = new ContainerBuilder();
$loader = new PhpFileLoader($container, new FileLocator(__DIR__));
$loader->load('services.php');
```

加载 YAML 配置文件：

```php
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;

$container = new ContainerBuilder();
$loader = new YamlFileLoader($container, new FileLocator(__DIR__));
$loader->load('services.yaml');
```

> **注意：**
> 如果你想加载 YAML 配置文件，还需要安装 [Yaml 组件](yaml.md)。

如果你确实想使用 PHP 创建服务，可以将其移到单独的配置文件中并以类似方式加载：

```php
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\PhpFileLoader;

$container = new ContainerBuilder();
$loader = new PhpFileLoader($container, new FileLocator(__DIR__));
$loader->load('services.php');
```

现在可以使用配置文件设置 `newsletter_manager` 和 `mailer` 服务：

```yaml
# config/services.yaml
parameters:
    # ...
    mailer.transport: sendmail

services:
    mailer:
        class:     Mailer
        arguments: ['%mailer.transport%']
    newsletter_manager:
        class:     NewsletterManager
        calls:
            - [setMailer, ['@mailer']]
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Mailer;
use App\NewsletterManager;

return App::config([
    'parameters' => [
        // ...
        'mailer.transport' => 'sendmail',
    ],
    'services' => [
        'mailer' => [
            'class' => Mailer::class,
            'arguments' => [param('mailer.transport')],
        ],
        'newsletter_manager' => [
            'class' => NewsletterManager::class,
            'calls' => [
                'setMailer' => [service('mailer')],
            ],
        ],
    ],
]);
```

[PSR-11]: https://www.php-fig.org/psr/psr-11/
