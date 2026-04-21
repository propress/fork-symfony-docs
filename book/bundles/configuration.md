# 如何为 Bundle 创建友好的配置

如果你打开主应用配置目录（通常是 `config/packages/`），你会看到许多不同的文件，例如 `framework.yaml`、`twig.yaml` 和 `doctrine.yaml`。每个文件都配置一个特定的 Bundle，允许你在高层次上定义选项，然后让 Bundle 根据你的设置执行所有底层复杂的变更。

例如，以下配置告诉 FrameworkBundle 启用表单集成，这涉及相当多服务的定义以及其他相关组件的集成：

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

为 Bundle 创建友好配置有两种不同的方式：

1. **使用主 Bundle 类**：推荐用于新 Bundle 以及遵循推荐目录结构的 Bundle；
2. **使用 Bundle 扩展类**：这是传统方式，但现在仅推荐用于遵循旧式目录结构的 Bundle。

## 使用 AbstractBundle 类

在继承 `Symfony\Component\HttpKernel\Bundle\AbstractBundle` 类的 Bundle 中，你可以在该类中添加所有与处理配置相关的逻辑：

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
        // "$config" 变量已经过合并和处理，因此你可以
        // 直接使用它来配置服务容器（在定义扩展类时，
        // 你还必须自行执行合并和处理操作）
        $container->services()
            ->get('acme_social.twitter_client')
            ->arg(0, $config['twitter']['client_id'])
            ->arg(1, $config['twitter']['client_secret'])
        ;
    }
}
```

> **注意：**
> `configure()` 和 `loadExtension()` 方法仅在编译时调用。

> **提示：**
> `AbstractBundle::configure()` 方法还允许从一个或多个文件导入配置定义：
>
> ```php
> // src/AcmeSocialBundle.php
> namespace Acme\SocialBundle;
>
> // ...
> class AcmeSocialBundle extends AbstractBundle
> {
>     public function configure(DefinitionConfigurator $definition): void
>     {
>         $definition->import('../config/definition.php');
>         // 你也可以使用 glob 模式
>         //$definition->import('../config/definition/*.php');
>     }
>
>     // ...
> }
> ```
>
> ```php
> // config/definition.php
> use Symfony\Component\Config\Definition\Configurator\DefinitionConfigurator;
>
> return static function (DefinitionConfigurator $definition): void {
>     $definition->rootNode()
>         ->children()
>             ->scalarNode('foo')->defaultValue('bar')->end()
>         ->end()
>     ;
> };
> ```

## 使用 Bundle 扩展

这是为 Bundle 创建友好配置的传统方式。对于新 Bundle，建议使用主 Bundle 类，但传统的创建扩展类的方式仍然有效。

假设你正在创建一个新 Bundle——AcmeSocialBundle——它提供与 X/Twitter 的集成。为了让 Bundle 对用户可配置，你可以添加如下所示的配置：

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

其基本思想是，不让用户覆盖单个参数，而是让用户只配置几个专门创建的选项。作为 Bundle 开发者，你随后解析该配置，并在"扩展"类中加载正确的服务和参数。

> **注意：**
> Bundle 配置的根键（前面示例中的 `acme_social`）由你的 Bundle 名称自动确定（它是去掉 `Bundle` 后缀的 Bundle 名称的蛇形命名法）。

> 另请参阅：
> 在 [如何在 Bundle 内加载服务配置](extension.md) 中了解更多关于扩展的内容。

> **提示：**
> 如果一个 Bundle 提供了扩展类，那么你通常*不应该*从该 Bundle 覆盖任何服务容器参数。基本思路是，如果存在扩展类，则所有应该可配置的设置都应该出现在该类提供的配置中。换句话说，扩展类定义了所有将保持向后兼容性的公共配置设置。

> 另请参阅：
> 有关在依赖注入容器中处理参数的内容，请参阅 [在 DIC 中使用参数](../configuration/using_parameters_in_dic.md)。

### 处理 `$configs` 数组

首先，你需要按照 [如何在 Bundle 内加载服务配置](extension.md) 中所述创建一个扩展类。

每当用户在配置文件中包含 `acme_social` 键（即 DI 别名）时，其下的配置就会被添加到配置数组中，并传递给你的扩展的 `load()` 方法（Symfony 会自动将配置转换为数组）。

对于上一节中的配置示例，传递给你的 `load()` 方法的数组如下所示：

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

注意，这是一个*数组的数组*，而不仅仅是配置值的单一平级数组。这是有意为之的，因为它允许 Symfony 解析多个配置资源。例如，如果 `acme_social` 出现在另一个配置文件中——比如 `config/packages/dev/acme_social.yaml`——且其下有不同的值，则传入的数组可能如下所示：

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

两个数组的顺序取决于哪个先被设置。

但不用担心！Symfony 的 Config 组件将帮助你合并这些值，提供默认值，并在配置错误时向用户提供验证错误。其工作原理如下：在 `DependencyInjection` 目录中创建一个 `Configuration` 类，并构建一棵定义你的 Bundle 配置结构的树。

处理示例配置的 `Configuration` 类如下所示：

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

> 另请参阅：
> `Configuration` 类可能比这里展示的更复杂，支持"prototype"节点、高级验证、单复数规范化和高级合并。你可以在 [Config 组件文档](../components/config/definition.md) 中了解更多相关内容。你也可以通过查看一些核心 Configuration 类来了解其实际应用，例如 [FrameworkBundle Configuration](https://github.com/symfony/symfony/blob/master/src/Symfony/Bundle/FrameworkBundle/DependencyInjection/Configuration.php) 或 [TwigBundle Configuration](https://github.com/symfony/symfony/blob/master/src/Symfony/Bundle/TwigBundle/DependencyInjection/Configuration.php)。

现在可以在你的 `load()` 方法中使用此类来合并配置并强制验证（例如，如果传入了额外选项，将抛出异常）：

```php
// src/DependencyInjection/AcmeSocialExtension.php
public function load(array $configs, ContainerBuilder $container): void
{
    $configuration = new Configuration();

    $config = $this->processConfiguration($configuration, $configs);

    // 现在你拥有这两个配置键
    // $config['twitter']['client_id'] 和 $config['twitter']['client_secret']
}
```

`processConfiguration()` 方法使用你在 `Configuration` 类中定义的配置树来验证、规范化并合并所有配置数组。

现在，你可以使用 `$config` 变量来修改 Bundle 提供的服务。例如，假设你的 Bundle 具有以下示例配置：

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

在你的扩展中，你可以加载此配置并动态设置其参数：

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

> **提示：**
> 与其每次在扩展中提供配置选项时都调用 `processConfiguration()`，你可能希望使用 `Symfony\Component\HttpKernel\DependencyInjection\ConfigurableExtension` 来自动完成此操作：
>
> ```php
> // src/DependencyInjection/HelloExtension.php
> namespace Acme\HelloBundle\DependencyInjection;
>
> use Symfony\Component\DependencyInjection\ContainerBuilder;
> use Symfony\Component\HttpKernel\DependencyInjection\ConfigurableExtension;
>
> class AcmeHelloExtension extends ConfigurableExtension
> {
>     // 注意此方法叫 loadInternal 而不是 load
>     protected function loadInternal(array $mergedConfig, ContainerBuilder $container): void
>     {
>         // ...
>     }
> }
> ```
>
> 此类使用 `getConfiguration()` 方法来获取 Configuration 实例。

> **自行处理配置**
>
> 使用 Config 组件完全是可选的。`load()` 方法接收一个配置值数组。你可以自行解析这些数组（例如，通过覆盖配置并使用 `isset` 检查值是否存在）：
>
> ```php
> public function load(array $configs, ContainerBuilder $container): void
> {
>     $config = [];
>     // 让后续资源覆盖前面设置的值
>     foreach ($configs as $subConfig) {
>         $config = array_merge($config, $subConfig);
>     }
>
>     // ... 现在使用平级的 $config 数组
> }
> ```

## 修改另一个 Bundle 的配置

如果你有多个互相依赖的 Bundle，允许一个 `Extension` 类修改传递给另一个 Bundle 的 `Extension` 类的配置可能很有用。这可以通过使用 prepend 扩展来实现。更多详情请参阅 [如何简化多个 Bundle 的配置](prepend_extension.md)。

## 导出配置

`config:dump-reference` 命令使用 YAML 格式在控制台中导出 Bundle 的默认配置。

只要你的 Bundle 配置位于标准位置（`<YourBundle>/src/DependencyInjection/Configuration`）且没有构造函数，它就会自动工作。如果你的情况不同，你的 `Extension` 类必须覆盖 `Extension::getConfiguration()` 方法并返回你的 `Configuration` 实例。
