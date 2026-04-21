# 如何在 Bundle 内加载服务配置

Bundle 创建的服务不是在应用使用的主 `config/services.yaml` 文件中定义的，而是在 Bundle 本身中定义。本文介绍如何使用 Bundle 目录结构创建和加载服务文件。

有两种不同的方式：

1. **在主 Bundle 类中加载服务**：推荐用于新 Bundle 以及遵循推荐目录结构的 Bundle；
2. **创建扩展类来加载服务配置文件**：这是传统方式，但现在仅推荐用于遵循旧式目录结构的 Bundle。

## 直接在 Bundle 类中加载服务

在继承 `Symfony\Component\HttpKernel\Bundle\AbstractBundle` 类的 Bundle 中，你可以定义 `Symfony\Component\HttpKernel\Bundle\AbstractBundle::loadExtension` 方法，从配置文件中加载服务定义：

```php
// ...
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\Component\HttpKernel\Bundle\AbstractBundle;

class AcmeHelloBundle extends AbstractBundle
{
    public function loadExtension(array $config, ContainerConfigurator $container, ContainerBuilder $builder): void
    {
        // 加载一个 PHP 或 YAML 文件
        $container->import('../config/services.php');

        // 你也可以添加或替换参数和服务
        $container->parameters()
            ->set('acme_hello.phrase', $config['phrase'])
        ;

        if ($config['scream']) {
            $container->services()
                ->get('acme_hello.printer')
                    ->class(ScreamingPrinter::class)
            ;
        }
    }
}
```

此方法的工作方式与下面介绍的 `Extension::load()` 方法类似，但使用了更简单的新 API 来定义和导入服务配置。

> **注意：**
> 与 `Extension::load()` 中的 `$configs` 参数不同，`$config` 参数已经由 `AbstractBundle` 合并和处理过。

> **注意：**
> `loadExtension()` 仅在编译时调用。

## 创建扩展类

这是在 Bundle 中加载服务定义的传统方式。对于新 Bundle，建议在主 Bundle 类中加载服务，但传统的创建扩展类的方式仍然有效。

依赖注入扩展被定义为遵循以下约定的类（稍后你将了解如何跳过这些约定）：

* 它必须位于 Bundle 的 `DependencyInjection` 命名空间中；

* 它必须实现 `Symfony\Component\DependencyInjection\Extension\ExtensionInterface`，通常通过继承 `Symfony\Component\DependencyInjection\Extension\Extension` 类来实现；

* 名称等于 Bundle 名称，将 `Bundle` 后缀替换为 `Extension`（例如，AcmeBundle 的扩展类称为 `AcmeExtension`，AcmeHelloBundle 的扩展类称为 `AcmeHelloExtension`）。

AcmeHelloBundle 的扩展应该如下所示：

```php
// src/DependencyInjection/AcmeHelloExtension.php
namespace Acme\HelloBundle\DependencyInjection;

use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Extension\Extension;

class AcmeHelloExtension extends Extension
{
    public function load(array $configs, ContainerBuilder $container): void
    {
        // ... 稍后你将在此处加载文件
    }
}
```

### 手动注册扩展类

当不遵循约定时，你必须手动注册你的扩展。为此，你应该覆盖 `Bundle::getContainerExtension()` 方法以返回扩展的实例：

```php
// ...
use Acme\HelloBundle\DependencyInjection\UnconventionalExtensionClass;
use Symfony\Component\DependencyInjection\Extension\ExtensionInterface;

class AcmeHelloBundle extends Bundle
{
    public function getContainerExtension(): ?ExtensionInterface
    {
        return new UnconventionalExtensionClass();
    }
}
```

此外，当新扩展类名不遵循命名约定时，你还必须覆盖 `Extension::getAlias()` 方法以返回正确的 DI 别名。DI 别名是在容器中（例如在 `config/packages/` 文件中）引用 Bundle 时使用的名称。默认情况下，这是通过去掉 `Extension` 后缀并将类名转换为下划线来完成的（例如 `AcmeHelloExtension` 的 DI 别名是 `acme_hello`）。

### 使用 `load()` 方法

在 `load()` 方法中，将加载与此扩展相关的所有服务和参数。此方法接收的不是实际的容器实例，而是其副本。此容器只包含实际容器中的参数。加载服务和参数后，该副本将与实际容器合并，以确保所有服务和参数也被添加到实际容器中。

在 `load()` 方法中，你可以使用 PHP 代码注册服务定义，但更常见的做法是将这些定义放在配置文件中（使用 YAML 或 PHP 格式）。

例如，假设你的 Bundle 的 `config/` 目录中有一个名为 `services.php` 的文件，你的 `load()` 方法如下所示：

```php
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\Loader\PhpFileLoader;

// ...
public function load(array $configs, ContainerBuilder $container): void
{
    $loader = new PhpFileLoader(
        $container,
        new FileLocator(__DIR__.'/../../config')
    );
    $loader->load('services.php');
}
```

另一个可用的加载器是 `YamlFileLoader`。

### 使用配置来更改服务

扩展也是处理特定 Bundle 配置的类（例如 `config/packages/<bundle_alias>.yaml` 中的配置）。要了解更多相关内容，请参阅[如何为 Bundle 创建友好的配置](configuration.md)。
