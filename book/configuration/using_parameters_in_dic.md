# 在依赖注入类中使用参数

你已经了解了如何在 [Symfony 服务容器](../service_container.md#服务容器参数)中使用配置参数。有些特殊情况，例如当你想使用 `%kernel.debug%` 参数让 Bundle 中的服务进入调试模式时。对于这种情况，需要做更多工作才能让系统理解参数值。默认情况下，你的参数 `%kernel.debug%` 将被视为字符串。参考以下示例：

```php
// inside Configuration class
$rootNode
    ->children()
        ->booleanNode('logging')->defaultValue('%kernel.debug%')->end()
        // ...
    ->end()
;

// inside the Extension class
$config = $this->processConfiguration($configuration, $configs);
var_dump($config['logging']);
```

现在，仔细检查结果：

```yaml
my_bundle:
    logging: true
    # true，如预期

my_bundle:
    logging: '%kernel.debug%'
    # true/false（取决于 Kernel 类的第二个参数），
    # 如预期，因为配置中的 %kernel.debug% 在传递给
    # 扩展之前会被求值

my_bundle: ~
# 传递字符串 "%kernel.debug%"。
# 这始终被视为 true。
# Configurator 不知道
# "%kernel.debug%" 是一个参数。
```

```php
$container->loadFromExtension('my_bundle', [
        'logging' => true,
        // true，如预期
    ]
);

$container->loadFromExtension('my_bundle', [
        'logging' => "%kernel.debug%",
        // true/false（取决于 Kernel 的第二个参数），
        // 如预期，因为配置中的 %kernel.debug% 在传递给
        // 扩展之前会被求值
    ]
);

$container->loadFromExtension('my_bundle');
// 传递字符串 "%kernel.debug%"。
// 这始终被视为 true。
// Configurator 不知道
// "%kernel.debug%" 是一个参数。
```

为了支持此使用场景，`Configuration` 类必须通过扩展注入此参数，如下所示：

```php
namespace App\DependencyInjection;

use Symfony\Component\Config\Definition\Builder\TreeBuilder;
use Symfony\Component\Config\Definition\ConfigurationInterface;

class Configuration implements ConfigurationInterface
{
    private bool $debug;

    public function __construct(private bool $debug)
    {
    }

    public function getConfigTreeBuilder(): TreeBuilder
    {
        $treeBuilder = new TreeBuilder('my_bundle');

        $treeBuilder->getRootNode()
            ->children()
                // ...
                ->booleanNode('logging')->defaultValue($this->debug)->end()
                // ...
            ->end()
        ;

        return $treeBuilder;
    }
}
```

并通过 `Extension` 类在 `Configuration` 的构造函数中设置它：

```php
namespace App\DependencyInjection;

use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\HttpKernel\DependencyInjection\Extension;

class AppExtension extends Extension
{
    // ...

    public function getConfiguration(array $config, ContainerBuilder $container): Configuration
    {
        return new Configuration($container->getParameter('kernel.debug'));
    }
}
```

> **提示：** 在 `Configurator` 类中有一些使用 `%kernel.debug%` 的示例，例如在 TwigBundle 中。然而，这是因为默认参数值由 Extension 类设置。
