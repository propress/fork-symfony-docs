# 如何简化多个 Bundle 的配置

在构建可复用且可扩展的应用时，开发者通常会面临一个选择：要么创建一个大型 bundle，
要么创建多个较小的 bundle。单一 bundle 的缺点是，用户无法移除未使用的功能；
多个 bundle 的缺点则是，配置会变得更繁琐，而且常常需要在不同 bundle 之间重复设置。

多 bundle 方案的这个缺点可以通过让一个 Extension 预置（prepend）任意 bundle 的设置来
解决。它可以读取 ``config/*`` 文件中定义的设置，并像用户已经在应用配置中显式写下这些
配置一样，把它们预置进去。

例如，这种能力可用于在多个 bundle 中统一配置要使用的实体管理器（entity manager）
名称，也可以在另一个 bundle 同样已加载的前提下启用某个可选功能。

要让一个 Extension 具备这项能力，它需要实现
``Symfony\Component\DependencyInjection\Extension\PrependExtensionInterface``：

```php
// src/DependencyInjection/AcmeHelloExtension.php
namespace Acme\HelloBundle\DependencyInjection;

use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Extension\PrependExtensionInterface;
use Symfony\Component\HttpKernel\DependencyInjection\Extension;

class AcmeHelloExtension extends Extension implements PrependExtensionInterface
{
    // ...

    public function prepend(ContainerBuilder $container): void
    {
        // ...
    }
}
```

在 ``PrependExtensionInterface::prepend()`` 方法内部，开发者可以在每个已注册 bundle 的
Extension 执行 ``ExtensionInterface::load()`` 之前，完整访问
``ContainerBuilder`` 实例。若要向某个 bundle extension 预置设置，可以对
``ContainerBuilder`` 实例调用 ``prependExtensionConfig()`` 方法。由于这个方法只会
预置设置，因此用户在 ``config/*`` 文件中显式写下的其他设置仍会覆盖这些预置值。

下面的示例展示了：如何为多个 bundle 预置某个配置项，以及当另一个特定 bundle 未注册时，
如何在多个 bundle 中关闭某个标记：

```php
// src/Acme/HelloBundle/DependencyInjection/AcmeHelloExtension.php
public function prepend(ContainerBuilder $container): void
{
    // 获取所有 bundle
    $bundles = $container->getParameter('kernel.bundles');
    // 判断 AcmeGoodbyeBundle 是否已注册
    if (!isset($bundles['AcmeGoodbyeBundle'])) {
        // 在相关 bundle 中禁用 AcmeGoodbyeBundle
        $config = ['use_acme_goodbye' => false];
        foreach ($container->getExtensions() as $name => $extension) {
            match ($name) {
                // 在 acme_something 和 acme_other 的配置中
                // 把 use_acme_goodbye 设为 false
                //
                // 注意：如果用户在 config/services.yaml 中手动把
                // use_acme_goodbye 配置为 true
                // 那么最终结果仍然会是 true，而不是 false
                'acme_something', 'acme_other' => $container->prependExtensionConfig($name, $config),
                default => null
            };
        }
    }

    // 获取 AcmeHelloExtension 的配置（它是一个配置列表）
    $configs = $container->getExtensionConfig($this->getAlias());

    // 倒序遍历，以便在 prepend 配置后保留原始顺序
    foreach (array_reverse($configs) as $config) {
        // 检查 "acme_hello" 配置中是否设置了 entity_manager_name
        if (isset($config['entity_manager_name'])) {
            // 为 acme_something 预置 entity_manager_name 设置
            $container->prependExtensionConfig('acme_something', [
                'entity_manager_name' => $config['entity_manager_name'],
            ]);
        }
    }
}
```

如果 ``AcmeGoodbyeBundle`` 未注册，并且 ``acme_hello`` 的 ``entity_manager_name``
设置为 ``non_default``，那么上面的代码就等价于在
``config/packages/acme_something.yaml`` 中写入以下内容：

```yaml
# config/packages/acme_something.yaml
acme_something:
    use_acme_goodbye: false
    entity_manager_name: non_default

acme_other:
    use_acme_goodbye: false
```

```php
// config/packages/acme_something.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'acme_something' => [
        'use_acme_goodbye' => false,
        'entity_manager_name' => 'non_default',
    ],
    'acme_other' => [
        'use_acme_goodbye' => false,
    ],
]);
```

> 📝 译者注：prepend 的优先级低于用户在应用配置文件中的显式配置，因此它更适合提供
> “默认联动配置”，而不是强制覆盖用户选择。

## 在 Bundle 类中预置 Extension 配置

如果您的 Bundle 类继承自 ``Symfony\Component\HttpKernel\Bundle\AbstractBundle``，
并定义了 ``Symfony\Component\HttpKernel\Bundle\AbstractBundle::prependExtension``
方法，那么也可以直接在 Bundle 类中预置 extension 配置：

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\Component\HttpKernel\Bundle\AbstractBundle;

class FooBundle extends AbstractBundle
{
    public function prependExtension(ContainerConfigurator $container, ContainerBuilder $builder): void
    {
        // prepend
        $builder->prependExtensionConfig('framework', [
            'cache' => ['prefix_seed' => 'foo/bar'],
        ]);

        // 从文件中预置配置
        $container->import('../config/packages/cache.php');
    }
}
```

> [!NOTE]
> ``prependExtension()`` 与 ``prepend()`` 一样，都只会在编译期（compile time）调用。

或者，您也可以使用 ``ContainerConfigurator::extension()`` 方法的 ``prepend`` 参数：

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\Component\HttpKernel\Bundle\AbstractBundle;

class FooBundle extends AbstractBundle
{
    public function prependExtension(ContainerConfigurator $container, ContainerBuilder $builder): void
    {
        // ...

        $container->extension('framework', [
            'cache' => ['prefix_seed' => 'foo/bar'],
        ], prepend: true);

        // ...
    }
}
```

## 多个 Bundle 同时使用 PrependExtensionInterface

如果有多个 bundle 都会为同一个 extension 进行 prepend，并且都定义了相同的键，
那么**最先**注册的 bundle 会优先生效：后续的 bundle 不会覆盖这个特定配置项。
