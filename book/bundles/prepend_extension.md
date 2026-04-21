# 如何简化多个 Bundle 的配置

在构建可复用和可扩展的应用程序时，开发者经常面临一个选择：创建一个大型 Bundle 还是多个小型 Bundle。创建单个 Bundle 的缺点是用户无法删除未使用的功能。创建多个 Bundle 的缺点是配置变得更加繁琐，设置通常需要对各个 Bundle 重复配置。

通过启用单个扩展来预置任意 Bundle 的设置，可以消除多 Bundle 方式的缺点。它可以使用 `config/*` 文件中定义的设置来预置设置，就像用户在应用配置中显式编写一样。

例如，这可以用于配置多个 Bundle 中使用的实体管理器名称。或者用于启用依赖于另一个已加载 Bundle 的可选功能。

要赋予扩展执行此操作的能力，它需要实现 `Symfony\Component\DependencyInjection\Extension\PrependExtensionInterface`：

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

在 `Symfony\Component\DependencyInjection\Extension\PrependExtensionInterface::prepend` 方法内部，开发者在每个已注册 Bundle 扩展的 `Symfony\Component\DependencyInjection\Extension\ExtensionInterface::load` 方法被调用之前，拥有对 `Symfony\Component\DependencyInjection\ContainerBuilder` 实例的完全访问权限。为了向 Bundle 扩展预置设置，开发者可以在 `Symfony\Component\DependencyInjection\ContainerBuilder` 实例上使用 `Symfony\Component\DependencyInjection\ContainerBuilder::prependExtensionConfig` 方法。由于此方法只是预置设置，因此在 `config/*` 文件中显式完成的任何其他设置都会覆盖这些预置设置。

以下示例说明了如何在多个 Bundle 中预置配置设置，以及在未注册某个特定 Bundle 时如何在多个 Bundle 中禁用标志：

```php
// src/Acme/HelloBundle/DependencyInjection/AcmeHelloExtension.php
public function prepend(ContainerBuilder $container): void
{
    // 获取所有 Bundle
    $bundles = $container->getParameter('kernel.bundles');
    // 确定 AcmeGoodbyeBundle 是否已注册
    if (!isset($bundles['AcmeGoodbyeBundle'])) {
        // 在 Bundle 中禁用 AcmeGoodbyeBundle
        $config = ['use_acme_goodbye' => false];
        foreach ($container->getExtensions() as $name => $extension) {
            match ($name) {
                // 在 acme_something 和 acme_other 的配置中
                // 将 use_acme_goodbye 设置为 false
                //
                // 注意，如果用户在 config/services.yaml 中
                // 手动将 use_acme_goodbye 配置为 true，
                // 则最终设置将为 true 而不是 false
                'acme_something', 'acme_other' => $container->prependExtensionConfig($name, $config),
                default => null
            };
        }
    }

    // 获取 AcmeHelloExtension 的配置（是配置列表）
    $configs = $container->getExtensionConfig($this->getAlias());

    // 以相反顺序迭代以在预置配置后保留原始顺序
    foreach (array_reverse($configs) as $config) {
        // 检查是否在 "acme_hello" 配置中设置了 entity_manager_name
        if (isset($config['entity_manager_name'])) {
            // 使用 entity_manager_name 预置 acme_something 设置
            $container->prependExtensionConfig('acme_something', [
                'entity_manager_name' => $config['entity_manager_name'],
            ]);
        }
    }
}
```

以上内容等同于在未注册 AcmeGoodbyeBundle 且 `acme_hello` 的 `entity_manager_name` 设置为 `non_default` 的情况下，将以下内容写入 `config/packages/acme_something.yaml`：

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

## 在 Bundle 类中预置扩展

如果你继承 `Symfony\Component\HttpKernel\Bundle\AbstractBundle` 类并定义 `Symfony\Component\HttpKernel\Bundle\AbstractBundle::prependExtension` 方法，你也可以直接在你的 Bundle 类中预置扩展配置：

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\Component\HttpKernel\Bundle\AbstractBundle;

class FooBundle extends AbstractBundle
{
    public function prependExtension(ContainerConfigurator $container, ContainerBuilder $builder): void
    {
        // 预置
        $builder->prependExtensionConfig('framework', [
            'cache' => ['prefix_seed' => 'foo/bar'],
        ]);

        // 从文件预置配置
        $container->import('../config/packages/cache.php');
    }
}
```

> **注意：**
> `prependExtension()` 方法与 `prepend()` 一样，仅在编译时调用。

或者，你可以使用 `Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator::extension` 方法的 `prepend` 参数：

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

## 多个 Bundle 使用 PrependExtensionInterface

如果有多个 Bundle 都预置了同一个扩展并定义了相同的键，则**首先**注册的 Bundle 将具有优先权：后续 Bundle 不会覆盖这个特定的配置设置。
