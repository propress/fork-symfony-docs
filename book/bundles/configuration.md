# 如何为 Bundle 创建友好的配置

如果您打开主应用的配置目录（通常是 ``config/packages/``），会看到不少不同的文件，
例如 ``framework.yaml``、``twig.yaml`` 和 ``doctrine.yaml``。这些文件分别负责配置
某个特定 bundle，让您可以在较高层级定义选项，然后由 bundle 根据这些设置完成底层的、
复杂的改动。

例如，下面这段配置会告诉 ``FrameworkBundle`` 启用表单集成（form integration），
而这背后其实涉及大量服务定义以及其他相关组件的集成：

```yaml
# config/packages/framework.yaml
framework:
    form: true
```

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'form' => true,
    ],
]);
```

为 bundle 创建友好配置（friendly configuration）主要有两种方式：

1. 使用[主 bundle 类](#使用-abstractbundle-类)：
   这是为新 bundle 以及遵循推荐目录结构的 bundle 所推荐的方式；
2. 使用 [Bundle Extension 类](#使用-bundle-extension)：
   这是传统做法，但如今只推荐给沿用旧版目录结构的 bundle。

## 使用 AbstractBundle 类

对于继承 ``Symfony\Component\HttpKernel\Bundle\AbstractBundle`` 的 bundle，
您可以把所有与配置处理相关的逻辑都写在这个类中：

```php
// src/AcmeSocialBundle.php
namespace Acme\SocialBundle;

use Symfony\Component\Config\Definition\Configurator\DefinitionConfigurator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\Component\HttpKernel\Bundle\AbstractBundle;

class AcmeSocialBundle extends AbstractBundle
{
    public function configure(DefinitionConfigurator $definition): void
    {
        $definition->rootNode()
            ->children()
                ->arrayNode('twitter')
                    ->children()
                        ->integerNode('client_id')->end()
                        ->scalarNode('client_secret')->end()
                    ->end()
                ->end() // twitter
            ->end()
        ;
    }

    public function loadExtension(array $config, ContainerConfigurator $container, ContainerBuilder $builder): void
    {
        // "$config" 变量已经完成合并和处理，因此您可以直接用它来配置服务容器
        // （如果定义的是 extension 类，您还需要自己完成这一步）
        $container->services()
            ->get('acme_social.twitter_client')
            ->arg(0, $config['twitter']['client_id'])
            ->arg(1, $config['twitter']['client_secret'])
        ;
    }
}
```

> [!NOTE]
> ``configure()`` 和 ``loadExtension()`` 方法只会在编译期（compile time）调用。

> [!TIP]
> ``AbstractBundle::configure()`` 方法还允许您从一个或多个文件中导入配置定义：

```php
// src/AcmeSocialBundle.php
namespace Acme\SocialBundle;

// ...
class AcmeSocialBundle extends AbstractBundle
{
    public function configure(DefinitionConfigurator $definition): void
    {
        $definition->import('../config/definition.php');
        // 您也可以使用 glob 模式
        //$definition->import('../config/definition/*.php');
    }

    // ...
}
```

```php
// config/definition.php
use Symfony\Component\Config\Definition\Configurator\DefinitionConfigurator;

return static function (DefinitionConfigurator $definition): void {
    $definition->rootNode()
        ->children()
            ->scalarNode('foo')->defaultValue('bar')->end()
        ->end()
    ;
};
```

## 使用 Bundle Extension

这是为 bundle 创建友好配置的传统方式。对于新 bundle，推荐您
[使用主 bundle 类](#使用-abstractbundle-类)，但传统的 extension 类方式依然有效。

假设您要创建一个新的 bundle——``AcmeSocialBundle``——它提供与 X/Twitter 的集成。
为了让用户能够配置这个 bundle，您可以提供如下形式的配置：

```yaml
# config/packages/acme_social.yaml
acme_social:
    twitter:
        client_id: 123
        client_secret: your_secret
```

```php
// config/packages/acme_social.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'acme_social' => [
        'twitter' => [
            'client_id' => 123,
            'client_secret' => 'your_secret',
        ],
    ],
]);
```

核心思想是：不要让用户去覆盖一个个独立参数，而是只暴露少量经过精心设计的配置项。
随后，作为 bundle 开发者，您再在某个 ``Extension`` 类中解析这些配置，并加载正确的
服务与参数。

> [!NOTE]
> 您的 bundle 配置根键（上例中的 ``acme_social``）会根据 bundle 名称自动推导得出；
> 它是去掉 ``Bundle`` 后缀后的 bundle 名称的 [snake case][snake-case] 形式。

> [!SEEALSO]
> 关于 extension 的更多说明，请参阅 [Bundle Extension](./extension.md)。

> [!TIP]
> 如果某个 bundle 提供了 Extension 类，那么通常**不应该**再去覆盖这个 bundle 的
> 服务容器参数。因为只要 extension 类存在，所有应允许配置的设置就都应该通过它暴露。
> 换句话说，extension 类定义了所有需要维持向后兼容（backward compatibility）的
> 公共配置项。

> [!SEEALSO]
> 关于依赖注入容器中的参数处理，请参阅
> [在服务定义中使用参数](../configuration/using_parameters_in_dic.md)。

### 处理 ``$configs`` 数组

首先，您需要按照 [Bundle Extension](./extension.md) 中的说明创建一个 extension 类。

每当用户在配置文件中包含 ``acme_social`` 键（也就是 DI alias）时，其下的配置都会被
加入一个配置数组中，并传递给您的 extension 的 ``load()`` 方法
（Symfony 会自动将配置转换为数组）。

对于上一节中的配置示例，传给 ``load()`` 方法的数组将如下所示：

```php
[
    [
        'twitter' => [
            'client_id' => 123,
            'client_secret' => 'your_secret',
        ],
    ],
]
```

请注意，这里是“数组的数组（array of arrays）”，而不是单个扁平数组。这是有意设计，
因为这样 Symfony 才能解析多个配置资源。例如，如果 ``acme_social`` 在另一个配置文件
（比如 ``config/packages/dev/acme_social.yaml``）中再次出现，并且下面的值不同，
那么传入的数组可能会是这样：

```php
[
    // 来自 config/packages/acme_social.yaml 的值
    [
        'twitter' => [
            'client_id' => 123,
            'client_secret' => 'your_secret',
        ],
    ],
    // 来自 config/packages/dev/acme_social.yaml 的值
    [
        'twitter' => [
            'client_id' => 456,
        ],
    ],
]
```

这两个数组的顺序取决于哪一个先被设置。

但不用担心！Symfony 的 Config 组件会帮助您合并这些值、提供默认值，并在配置不正确时
向用户返回校验错误。具体做法如下：在 ``DependencyInjection`` 目录中创建一个
``Configuration`` 类，并构建一棵树，用来定义 bundle 配置的结构。

处理上述示例配置所需的 ``Configuration`` 类如下：

```php
// src/DependencyInjection/Configuration.php
namespace Acme\SocialBundle\DependencyInjection;

use Symfony\Component\Config\Definition\Builder\TreeBuilder;
use Symfony\Component\Config\Definition\ConfigurationInterface;

class Configuration implements ConfigurationInterface
{
    public function getConfigTreeBuilder(): TreeBuilder
    {
        $treeBuilder = new TreeBuilder('acme_social');

        $treeBuilder->getRootNode()
            ->children()
                ->arrayNode('twitter')
                    ->children()
                        ->integerNode('client_id')->end()
                        ->scalarNode('client_secret')->end()
                    ->end()
                ->end() // twitter
            ->end()
        ;

        return $treeBuilder;
    }
}
```

> [!SEEALSO]
> ``Configuration`` 类可以比这里展示的复杂得多，它支持 prototype 节点、
> 高级校验、复数/单数规范化（plural/singular normalization）以及高级合并逻辑。
> 您可以在 [Config 组件文档](../components/config/definition.md) 中了解更多。
> 您也可以查看一些核心的 Configuration 类，例如
> [FrameworkBundle Configuration][frameworkbundle-configuration] 和
> [TwigBundle Configuration][twigbundle-configuration]。

现在，您可以在 ``load()`` 方法中使用这个类来完成配置合并并强制执行校验
（例如如果传入了额外的未知选项，就会抛出异常）：

```php
// src/DependencyInjection/AcmeSocialExtension.php
public function load(array $configs, ContainerBuilder $container): void
{
    $configuration = new Configuration();

    $config = $this->processConfiguration($configuration, $configs);

    // 现在您可以使用这两个配置键：
    // $config['twitter']['client_id'] 和 $config['twitter']['client_secret']
}
```

``processConfiguration()`` 方法会使用您在 ``Configuration`` 类中定义的配置树，
对所有配置数组进行校验、规范化和合并。

现在，您就可以使用 ``$config`` 变量修改 bundle 提供的服务了。例如，假设您的 bundle
包含如下示例配置：

```php
// src/Resources/config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Acme\SocialBundle\TwitterClient;

return function (ContainerConfigurator $container) {
    $container->services()
        ->set('acme_social.twitter_client', TwitterClient::class)
            ->args([abstract_arg('client_id'), abstract_arg('client_secret')]);
};
```

在 extension 中，您可以加载这个配置文件并动态设置它的参数：

```php
// src/DependencyInjection/AcmeSocialExtension.php
namespace Acme\SocialBundle\DependencyInjection;

use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\Loader\PhpFileLoader;

public function load(array $configs, ContainerBuilder $container): void
{
    $loader = new PhpFileLoader($container, new FileLocator(dirname(__DIR__).'/Resources/config'));
    $loader->load('services.php');

    $configuration = new Configuration();
    $config = $this->processConfiguration($configuration, $configs);

    $definition = $container->getDefinition('acme_social.twitter_client');
    $definition->replaceArgument(0, $config['twitter']['client_id']);
    $definition->replaceArgument(1, $config['twitter']['client_secret']);
}
```

> [!TIP]
> 如果您的 extension 每次都要调用 ``processConfiguration()`` 来处理配置选项，
> 那么可以考虑改用
> ``Symfony\Component\HttpKernel\DependencyInjection\ConfigurableExtension``，
> 这样就能自动完成这一步：

```php
// src/DependencyInjection/HelloExtension.php
namespace Acme\HelloBundle\DependencyInjection;

use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\HttpKernel\DependencyInjection\ConfigurableExtension;

class AcmeHelloExtension extends ConfigurableExtension
{
    // 注意：这里的方法名是 loadInternal，而不是 load
    protected function loadInternal(array $mergedConfig, ContainerBuilder $container): void
    {
        // ...
    }
}
```

> 它会使用 ``getConfiguration()`` 方法来获取 ``Configuration`` 实例。

> [!NOTE]
> 您也可以自行处理配置。Config 组件完全是可选的。``load()`` 方法接收到的是一个配置值
> 数组，您完全可以自己解析这些数组（例如覆盖配置，并用 ``isset()`` 判断某个值是否存在）：

```php
public function load(array $configs, ContainerBuilder $container): void
{
    $config = [];
    // 让后面的资源覆盖前面设置的值
    foreach ($configs as $subConfig) {
        $config = array_merge($config, $subConfig);
    }

    // ... 现在使用这个扁平化后的 $config 数组
}
```

## 修改另一个 Bundle 的配置

如果您有多个相互依赖的 bundle，那么让一个 ``Extension`` 类去修改另一个 bundle 的
``Extension`` 类收到的配置，可能会很有帮助。这可以通过 prepend extension 实现。
详情请参阅 [预置 Extension 配置](./prepend_extension.md)。

## 导出配置

``config:dump-reference`` 命令会在控制台中以 YAML 格式导出某个 bundle 的默认配置。

只要 bundle 的配置类位于标准位置
（``<YourBundle>/src/DependencyInjection/Configuration``），并且没有构造函数，
它就会自动生效。如果您的情况不同，那么 ``Extension`` 类就必须重写
``Extension::getConfiguration()`` 方法，并返回您的 ``Configuration`` 实例。

[frameworkbundle-configuration]: https://github.com/symfony/symfony/blob/master/src/Symfony/Bundle/FrameworkBundle/DependencyInjection/Configuration.php
[twigbundle-configuration]: https://github.com/symfony/symfony/blob/master/src/Symfony/Bundle/TwigBundle/DependencyInjection/Configuration.php
[snake-case]: https://en.wikipedia.org/wiki/Snake_case
