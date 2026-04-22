# 如何在 Bundle 中加载服务配置

bundle 创建的服务并不是定义在应用使用的主 ``config/services.yaml`` 文件中，
而是定义在 bundle 自身内部。本文会说明如何利用 bundle 的目录结构来创建并加载服务文件。

主要有两种做法：

1. 在[主 bundle 类](#直接在-bundle-类中加载服务)中加载服务：
   这是新 bundle 以及遵循推荐目录结构的 bundle 的推荐做法；
2. 创建 [Extension 类](#创建-extension-类)来加载服务配置文件：
   这是传统做法，但如今只推荐给遵循旧版目录结构的 bundle。

## 直接在 Bundle 类中加载服务

对于继承 ``Symfony\Component\HttpKernel\Bundle\AbstractBundle`` 的 bundle，
您可以定义 ``Symfony\Component\HttpKernel\Bundle\AbstractBundle::loadExtension``
方法，从配置文件中加载服务定义：

```php
// ...
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\Component\HttpKernel\Bundle\AbstractBundle;

class AcmeHelloBundle extends AbstractBundle
{
    public function loadExtension(array $config, ContainerConfigurator $container, ContainerBuilder $builder): void
    {
        // 加载 PHP 或 YAML 文件
        $container->import('../config/services.php');

        // 您也可以新增或替换参数与服务
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

这个方法与下文介绍的 ``Extension::load()`` 方法类似，但它使用了一套更简洁的新 API
来定义和导入服务配置。

> [!NOTE]
> 与 ``Extension::load()`` 中的 ``$configs`` 参数不同，``$config`` 参数已经由
> ``AbstractBundle`` 完成合并和处理。

> [!NOTE]
> ``loadExtension()`` 只会在编译期（compile time）调用。

## 创建 Extension 类

这是在 bundle 中加载服务定义的传统方式。对于新 bundle，推荐您
[在主 bundle 类中加载服务](#直接在-bundle-类中加载服务)，但传统的 Extension 类方式仍然可用。

依赖注入扩展（dependency injection extension）需要定义为一个满足以下约定的类
（稍后您会看到，如果有需要，也可以跳过这些约定）：

- 它必须位于 bundle 的 ``DependencyInjection`` 命名空间中；
- 它必须实现
  ``Symfony\Component\DependencyInjection\Extension\ExtensionInterface``，
  通常的做法是继承
  ``Symfony\Component\DependencyInjection\Extension\Extension`` 类；
- 它的名称应与 bundle 名称一致，只是把 ``Bundle`` 后缀替换为 ``Extension``
  （例如 ``AcmeBundle`` 的扩展类应叫作 ``AcmeExtension``，
  ``AcmeHelloBundle`` 的扩展类应叫作 ``AcmeHelloExtension``）。

``AcmeHelloBundle`` 的扩展类应如下所示：

```php
// src/DependencyInjection/AcmeHelloExtension.php
namespace Acme\HelloBundle\DependencyInjection;

use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Extension\Extension;

class AcmeHelloExtension extends Extension
{
    public function load(array $configs, ContainerBuilder $container): void
    {
        // ... 稍后您会在这里加载文件
    }
}
```

### 手动注册 Extension 类

如果没有遵循这些约定，您就必须手动注册 extension。为此，应重写
``Bundle::getContainerExtension()`` 方法，并返回该 extension 的实例：

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

此外，如果新的 Extension 类名不符合命名约定，您还必须重写
``Extension::getAlias()`` 方法，返回正确的 DI alias。DI alias 是容器中引用该 bundle
时使用的名称（例如在 ``config/packages/`` 文件中）。默认情况下，Symfony 会移除
``Extension`` 后缀，并将类名转换为下划线格式
（例如 ``AcmeHelloExtension`` 的 DI alias 是 ``acme_hello``）。

### 使用 ``load()`` 方法

``load()`` 方法会加载与此 extension 相关的所有服务和参数。该方法拿到的并不是真实容器
实例，而是它的一个副本。这个副本只包含真实容器中的参数。加载完服务和参数后，这个副本
会再合并回真实容器，以确保这些服务和参数也会进入实际容器。

在 ``load()`` 方法中，您可以直接用 PHP 代码注册服务定义，但更常见的做法是把这些
定义放进配置文件中（YAML 或 PHP 格式）。

例如，假设您的 bundle 在 ``config/`` 目录下有一个名为 ``services.php`` 的文件，
那么 ``load()`` 方法可以这样写：

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

另一种可用的加载器是 ``YamlFileLoader``。

### 使用配置来修改服务

Extension 类也负责处理该 bundle 的配置
（例如 ``config/packages/<bundle_alias>.yaml`` 中的配置）。更多信息请参阅
[Bundle 配置](./configuration.md)。
