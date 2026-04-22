# 缓存（Cache）

使用缓存是提高应用运行速度的好方法。Symfony 缓存组件提供了众多适配器（adapters），可连接到不同的存储系统。每个适配器都针对高性能而开发。

下面的例子展示了缓存的典型用法：

```php
use Symfony\Contracts\Cache\ItemInterface;

// 只有在缓存未命中（cache miss）时才会执行这个 callable
$value = $pool->get('my_cache_key', function (ItemInterface $item): string {
    $item->expiresAfter(3600);

    // ... 执行一些 HTTP 请求或耗时计算
    $computedValue = 'foobar';

    return $computedValue;
});

echo $value; // 'foobar'

// ... 要删除缓存键
$pool->delete('my_cache_key');
```

Symfony 支持 Cache Contracts 以及 PSR-6/16 接口。有关更多信息，请阅读[组件文档](./components/cache.md)。

## 使用 FrameworkBundle 配置缓存

在配置缓存组件时，有几个概念您需要了解：

**Pool（缓存池）**  
这是您将要交互的服务。每个 pool 都有自己的命名空间和缓存项。不同 pool 之间永远不会冲突。

**Adapter（适配器）**  
适配器是一个*模板*，您可以用它来创建 pool。

**Provider（提供者）**  
提供者是某些适配器用来连接到存储系统的服务。Redis 和 Memcached 就是使用 provider 的适配器示例。如果使用 DSN 作为 provider，Symfony 会自动创建对应的服务。

缓存组件预配置了一系列适配器：

* [cache.adapter.apcu](./components/cache/adapters/apcu_adapter.md)
* [cache.adapter.array](./components/cache/adapters/array_cache_adapter.md)
* [cache.adapter.doctrine_dbal](./components/cache/adapters/doctrine_dbal_adapter.md)
* [cache.adapter.filesystem](./components/cache/adapters/filesystem_adapter.md)
* [cache.adapter.memcached](./components/cache/adapters/memcached_adapter.md)
* [cache.adapter.pdo](./components/cache/adapters/pdo_adapter.md)
* [cache.adapter.psr6](./components/cache/adapters/proxy_adapter.md)
* [cache.adapter.redis](./components/cache/adapters/redis_adapter.md)
* `cache.adapter.redis_tag_aware`（Redis 适配器，针对标签优化）

> [!NOTE]
> 还有一个特殊的 `cache.adapter.system` 适配器。建议将其用于[系统缓存](#系统缓存与应用缓存)。该适配器使用一些逻辑根据您的系统动态选择最佳存储（PHP 文件或 APCu）。

部分适配器可通过快捷方式配置。

```yaml
# config/packages/cache.yaml
framework:
    cache:
        directory: '%kernel.cache_dir%/pools' # 仅用于 cache.adapter.filesystem

        default_doctrine_dbal_provider: 'doctrine.dbal.default_connection'
        default_psr6_provider: 'app.my_psr6_service'
        default_redis_provider: 'redis://localhost'
        default_memcached_provider: 'memcached://localhost'
        default_pdo_provider: 'pgsql:host=localhost'
```

```php
// config/packages/cache.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'cache' => [
            'directory' => '%kernel.cache_dir%/pools', // 仅用于 cache.adapter.filesystem
            'default_doctrine_dbal_provider' => 'doctrine.dbal.default_connection',
            'default_psr6_provider' => 'app.my_psr6_service',
            'default_redis_provider' => 'redis://localhost',
            'default_memcached_provider' => 'memcached://localhost',
            'default_pdo_provider' => 'pgsql:host=localhost',
        ],
    ],
]);
```

## 系统缓存与应用缓存

默认情况下，始终会启用两个缓存池：`cache.system` 和 `cache.app`。

`cache.system` 在**内部**被 Symfony 组件（如注解、序列化器和验证）使用。它也**可供应用代码**使用，但有一些特定约束：

1. 缓存条目必须可从源代码派生，并可通过 `CacheWarmer` 在缓存预热（cache warmup）期间重新生成。
2. 缓存内容只能在源代码更改时（即在部署时）更改，不能在运行时更改；部署后应视为只读。

默认情况下，`cache.system` 使用 `cache.adapter.system`，它会写入文件系统，并在可用时链式使用 APCu。在大多数情况下，默认配置是正确的选择。

> [!TIP]
> 虽然可以重新配置 `system` 缓存，但建议保持 Symfony 为其应用的默认配置。

`cache.app` 是**通用数据缓存**，供应用和 bundle 代码使用。此池中的数据无需在部署时刷新。它默认使用 `cache.adapter.filesystem`，但建议在可用时配置更快的适配器（如 Redis），以确保缓存数据在部署后仍然存在，并在多服务器设置中可跨多个实例共享。

除非另有配置，[自定义池](#创建自定义命名空间池)会默认使用 `cache.app` 作为其适配器。使用**自动装配（autowiring）**时，`cache.app` 会自动注入到任何类型为 `CacheItemPoolInterface`、`AdapterInterface` 或 `CacheInterface` 的服务参数中。

您可以通过 `app` 和 `system` 键配置每个预定义池使用的适配器：

```yaml
# config/packages/cache.yaml
framework:
    cache:
        app: cache.adapter.filesystem
        system: cache.adapter.system
```

```php
// config/packages/cache.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'cache' => [
            'app' => 'cache.adapter.filesystem',
            'system' => 'cache.adapter.system',
        ],
    ],
]);
```

## 创建自定义（命名空间）池

您也可以创建更加自定义的池：

```yaml
# config/packages/cache.yaml
framework:
    cache:
        default_memcached_provider: 'memcached://localhost'

        pools:
            # 创建一个 "custom_thing.cache" 服务
            # 可通过 "CacheInterface $customThingCache" 自动装配
            # 使用 "app" 缓存配置
            custom_thing.cache:
                adapter: cache.app

            # 创建一个 "my_cache_pool" 服务
            # 可通过 "CacheInterface $myCachePool" 自动装配
            my_cache_pool:
                adapter: cache.adapter.filesystem

            # 使用上面定义的 default_memcached_provider
            acme.cache:
                adapter: cache.adapter.memcached

            # 控制适配器的配置
            foobar.cache:
                adapter: cache.adapter.memcached
                provider: 'memcached://user:password@example.com'

            # 使用 "foobar.cache" 池作为后端，但控制
            # 生命周期，并且（像所有池一样）拥有独立的缓存命名空间
            short_cache:
                adapter: foobar.cache
                default_lifetime: 60
```

```php
// config/packages/cache.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'cache' => [
            'default_memcached_provider' => 'memcached://localhost',
            'pools' => [
                // 创建一个 "custom_thing.cache" 服务
                // 可通过 "CacheInterface $customThingCache" 自动装配
                // 使用 "app" 缓存配置
                'custom_thing.cache' => [
                    'adapter' => 'cache.app',
                ],
                // 创建一个 "my_cache_pool" 服务
                // 可通过 "CacheInterface $myCachePool" 自动装配
                'my_cache_pool' => [
                    'adapter' => 'cache.adapter.filesystem',
                ],
                // 使用上面定义的 default_memcached_provider
                'acme.cache' => [
                    'adapter' => 'cache.adapter.memcached',
                ],
                // 控制适配器的配置
                'foobar.cache' => [
                    'adapter' => 'cache.adapter.memcached',
                    'provider' => 'memcached://user:password@example.com',
                ],
                // 使用 "foobar.cache" 池作为后端，但控制
                // 生命周期，并且（像所有池一样）拥有独立的缓存命名空间
                'short_cache' => [
                    'adapter' => 'foobar.cache',
                    'default_lifetime' => 60,
                ],
            ],
        ],
    ],
]);
```

每个 pool 管理一组独立的缓存键：不同 pool 的键*永远不会*冲突，即使它们共享同一后端。这是通过用命名空间作为键的前缀实现的，该命名空间由 pool 名称、缓存适配器类名以及一个可配置的 seed（默认为项目目录和编译后的容器类）的哈希值生成。

每个自定义 pool 都会成为一个服务，服务 ID 就是 pool 的名称（例如 `custom_thing.cache`）。Symfony 还会为每个 pool 创建一个自动装配别名，使用其名称的驼峰命名版本 —— 例如，通过将参数命名为 `$customThingCache` 并类型提示为 `Symfony\Contracts\Cache\CacheInterface` 或 `Psr\Cache\CacheItemPoolInterface`，就可以自动注入 `custom_thing.cache`：

```php
use Symfony\Contracts\Cache\CacheInterface;
// ...

// 从控制器方法
public function listProducts(CacheInterface $customThingCache): Response
{
    // ...
}

// 在服务中
public function __construct(private CacheInterface $customThingCache)
{
    // ...
}
```

> [!TIP]
> 如果您需要命名空间与第三方应用互操作，可以通过设置 `cache.pool` 服务标签的 `namespace` 属性来控制自动生成。例如，您可以覆盖适配器的服务定义：
> 
> ```yaml
> # config/services.yaml
> services:
>     # ...
> 
>     app.cache.adapter.redis:
>         parent: 'cache.adapter.redis'
>         tags:
>             - { name: 'cache.pool', namespace: 'my_custom_namespace' }
> ```

## 自定义 Provider 选项

某些 provider 有特定的可配置选项。[RedisAdapter](./components/cache/adapters/redis_adapter.md) 允许您使用 `timeout`、`retry_interval` 等选项创建 provider。要使用这些非默认值的选项，您需要创建自己的 `\Redis` provider，并在配置池时使用它。

```yaml
# config/packages/cache.yaml
framework:
    cache:
        pools:
            cache.my_redis:
                adapter: cache.adapter.redis
                provider: app.my_custom_redis_provider

services:
    app.my_custom_redis_provider:
        class: \Redis
        factory: ['Symfony\Component\Cache\Adapter\RedisAdapter', 'createConnection']
        arguments:
            - 'redis://localhost'
            - { retry_interval: 2, timeout: 10 }
```

```php
// config/packages/cache.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\Cache\Adapter\RedisAdapter;

return App::config([
    'framework' => [
        'cache' => [
            'pools' => [
                'cache.my_redis' => [
                    'adapter' => 'cache.adapter.redis',
                    'provider' => 'app.my_custom_redis_provider',
                ],
            ],
        ],
    ],
    'services' => [
        'app.my_custom_redis_provider' => [
            'class' => \Redis::class,
            'factory' => [RedisAdapter::class, 'createConnection'],
            'arguments' => ['redis://localhost', ['retry_interval' => 2, 'timeout' => 10]],
        ],
    ],
]);
```

## 创建缓存链（Cache Chain）

不同的缓存适配器各有优缺点。有些可能非常快但针对小数据优化，有些可能能够容纳大量数据但速度较慢。为了兼得两者的优点，您可以使用适配器链。

缓存链将多个缓存池组合成一个。当在缓存链中存储项时，Symfony 会依次将其存储到所有池中。当检索项时，Symfony 会尝试从第一个池获取它。如果找不到，它会尝试下一个池，直到找到项或抛出异常。由于这种行为，建议按照从最快到最慢的顺序定义链中的适配器。

如果在某个池中存储项时发生错误，Symfony 会将其存储在其他池中，并且不会抛出异常。稍后，当检索该项时，Symfony 会自动将该项存储到所有缺失的池中。

```yaml
# config/packages/cache.yaml
framework:
    cache:
        pools:
            my_cache_pool:
                default_lifetime: 31536000  # 一年
                adapters:
                  - cache.adapter.array
                  - cache.adapter.apcu
                  - {name: cache.adapter.redis, provider: 'redis://user:password@example.com'}
```

```php
// config/packages/cache.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'cache' => [
            'pools' => [
                'my_cache_pool' => [
                    'default_lifetime' => 31536000, // 一年
                    'adapters' => [
                        'cache.adapter.array',
                        'cache.adapter.apcu',
                        ['name' => 'cache.adapter.redis', 'provider' => 'redis://user:password@example.com'],
                    ],
                ],
            ],
        ],
    ],
]);
```

## 使用缓存标签（Cache Tags）

在有很多缓存键的应用中，组织存储的数据以便更高效地使缓存失效可能很有用。实现这一目标的一种方法是使用缓存标签。可以为缓存项添加一个或多个标签。所有具有相同标签的项都可以通过一次函数调用使其失效：

```php
use Symfony\Contracts\Cache\ItemInterface;
use Symfony\Contracts\Cache\TagAwareCacheInterface;

class SomeClass
{
    // 使用自动装配注入缓存池
    public function __construct(
        private TagAwareCacheInterface $myCachePool,
    ) {
    }

    public function someMethod(): void
    {
        $value0 = $this->myCachePool->get('item_0', function (ItemInterface $item): string {
            $item->tag(['foo', 'bar']);

            return 'debug';
        });

        $value1 = $this->myCachePool->get('item_1', function (ItemInterface $item): string {
            $item->tag('foo');

            return 'debug';
        });

        // 移除所有标记为 "bar" 的缓存键
        $this->myCachePool->invalidateTags(['bar']);
    }
}
```

缓存适配器需要实现 `Symfony\Contracts\Cache\TagAwareCacheInterface` 才能启用此功能。可以通过以下配置添加：

```yaml
# config/packages/cache.yaml
framework:
    cache:
        pools:
            my_cache_pool:
                adapter: cache.adapter.redis_tag_aware
                tags: true
```

```php
// config/packages/cache.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'cache' => [
            'pools' => [
                'my_cache_pool' => [
                    'adapter' => 'cache.adapter.redis_tag_aware',
                    'tags' => true,
                ],
            ],
        ],
    ],
]);
```

默认情况下，标签存储在同一个池中。这在大多数情况下都很好。但有时将标签存储在不同的池中可能更好。这可以通过指定适配器来实现。

```yaml
# config/packages/cache.yaml
framework:
    cache:
        pools:
            my_cache_pool:
                adapter: cache.adapter.redis
                tags: tag_pool
            tag_pool:
                adapter: cache.adapter.apcu
```

```php
// config/packages/cache.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'cache' => [
            'pools' => [
                'my_cache_pool' => [
                    'adapter' => 'cache.adapter.redis',
                    'tags' => 'tag_pool',
                ],
                'tag_pool' => [
                    'adapter' => 'cache.adapter.apcu',
                ],
            ],
        ],
    ],
]);
```

> [!NOTE]
> `Symfony\Contracts\Cache\TagAwareCacheInterface` 接口会自动装配到 `cache.app` 服务。

## 清除缓存

要清除缓存，您可以使用 `bin/console cache:pool:clear [pool]` 命令。这将从存储中删除所有条目，您必须重新计算所有值。您还可以将池分组到"缓存清除器（cache clearers）"中。默认有 3 个缓存清除器：

* `cache.global_clearer`
* `cache.system_clearer`
* `cache.app_clearer`

全局清除器会清除每个池中的所有缓存项。系统缓存清除器用于 `bin/console cache:clear` 命令。应用清除器是默认清除器。

查看所有可用的缓存池：

```bash
$ php bin/console cache:pool:list
```

清除一个池：

```bash
$ php bin/console cache:pool:clear my_cache_pool
```

清除所有自定义池：

```bash
$ php bin/console cache:pool:clear cache.app_clearer
```

清除所有缓存池：

```bash
$ php bin/console cache:pool:clear --all
```

清除所有缓存池，但排除某些池：

```bash
$ php bin/console cache:pool:clear --all --exclude=my_cache_pool --exclude=another_cache_pool
```

清除所有地方的所有缓存：

```bash
$ php bin/console cache:pool:clear cache.global_clearer
```

按标签清除缓存：

```bash
# 从所有可标记的池中使标签 tag1 失效
$ php bin/console cache:pool:invalidate-tags tag1

# 从所有可标记的池中使标签 tag1 和 tag2 失效
$ php bin/console cache:pool:invalidate-tags tag1 tag2

# 从 cache.app 池中使标签 tag1 和 tag2 失效
$ php bin/console cache:pool:invalidate-tags tag1 tag2 --pool=cache.app

# 从 cache1 和 cache2 池中使标签 tag1 和 tag2 失效
$ php bin/console cache:pool:invalidate-tags tag1 tag2 -p cache1 -p cache2
```

## 加密缓存

要使用 `libsodium` 加密缓存，您可以使用 `Symfony\Component\Cache\Marshaller\SodiumMarshaller`。

首先，您需要生成一个安全密钥并将其添加到您的[密钥存储](./configuration/secrets.md)中，作为 `CACHE_DECRYPTION_KEY`：

```bash
$ php -r 'echo base64_encode(sodium_crypto_box_keypair());'
```

然后，使用此密钥注册 `SodiumMarshaller` 服务：

```yaml
# config/packages/cache.yaml

# ...
services:
    Symfony\Component\Cache\Marshaller\SodiumMarshaller:
        decorates: cache.default_marshaller
        arguments:
            - ['%env(base64:CACHE_DECRYPTION_KEY)%']
            # 使用多个密钥以便轮换
            #- ['%env(base64:CACHE_DECRYPTION_KEY)%', '%env(base64:OLD_CACHE_DECRYPTION_KEY)%']
            - '@.inner'
```

```php
// config/packages/cache.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\Cache\Marshaller\SodiumMarshaller;

return App::config([
    'services' => [
        SodiumMarshaller::class => [
            'decorates' => 'cache.default_marshaller',
            'arguments' => [
                [env('CACHE_DECRYPTION_KEY')->base64()],
                // 使用多个密钥以便轮换
                // [env('CACHE_DECRYPTION_KEY')->base64(), env('OLD_CACHE_DECRYPTION_KEY')->base64()]
                service('.inner'),
            ],
        ],
    ],
]);
```

> [!DANGER]
> 这将加密缓存项的值，但不会加密缓存键。小心不要在键中泄露敏感数据。

配置多个密钥时，第一个密钥将用于读取和写入，其他密钥仅用于读取。一旦所有使用旧密钥加密的缓存项过期，您就可以完全删除 `OLD_CACHE_DECRYPTION_KEY`。

## 异步计算缓存值

缓存组件使用[概率性提前过期（probabilistic early expiration）][probabilistic-early-expiration]算法来防止[缓存雪崩（cache stampede）](./components/cache.md#cache_stampede-prevention)问题。这意味着某些缓存项会被选择提前过期，即使它们仍然是新鲜的。

默认情况下，已过期的缓存项是同步计算的。但是，您可以使用 [Messenger 组件](./components/messenger.md)将值计算委托给后台工作进程，从而异步计算它们。在这种情况下，当查询项时，会立即返回其缓存值，并通过 Messenger 总线分派 `Symfony\Component\Cache\Messenger\EarlyExpirationMessage`。

当消息使用者处理此消息时，会异步计算刷新后的缓存值。下次查询该项时，将返回刷新后的新值。

首先，创建一个将计算项值的服务：

```php
// src/Cache/CacheComputation.php
namespace App\Cache;

use Psr\Cache\CacheItemInterface;
use Symfony\Contracts\Cache\CallbackInterface;

class CacheComputation implements CallbackInterface
{
    public function __invoke(CacheItemInterface $item, bool &$save): string
    {
        $item->expiresAfter(5);

        // 这只是一个随机示例；这里您必须执行自己的计算
        return sprintf('#%06X', mt_rand(0, 0xFFFFFF));
    }
}
```

此缓存值将从控制器、其他服务等请求。在下面的例子中，该值从控制器请求：

```php
// src/Controller/CacheController.php
namespace App\Controller;

use App\Cache\CacheComputation;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Contracts\Cache\CacheInterface;
use Symfony\Contracts\Cache\ItemInterface;

class CacheController extends AbstractController
{
    #[Route('/cache', name: 'cache')]
    public function index(CacheInterface $asyncCache, CacheComputation $cacheComputation): Response
    {
        // 将刷新项的服务方法传递给缓存
        $cachedValue = $asyncCache->get('my_value', $cacheComputation)

        // ...
    }
}
```

最后，配置一个新的缓存池（例如名为 `async.cache`），它将使用消息总线在工作进程中计算值：

```yaml
# config/packages/framework.yaml
framework:
    cache:
        pools:
            async.cache:
                early_expiration_message_bus: messenger.default_bus

    messenger:
        transports:
            async_bus: '%env(MESSENGER_TRANSPORT_DSN)%'
        routing:
            'Symfony\Component\Cache\Messenger\EarlyExpirationMessage': async_bus
```

```php
// config/framework/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\Cache\Messenger\EarlyExpirationMessage;

return App::config([
    'framework' => [
        'cache' => [
            'pools' => [
                'async.cache' => [
                    'early_expiration_message_bus' => 'messenger.default_bus',
                ],
            ],
        ],
        'messenger' => [
            'transports' => [
                'async_bus' => env('MESSENGER_TRANSPORT_DSN'),
            ],
            'routing' => [
                EarlyExpirationMessage::class => 'async_bus',
            ],
        ],
    ],
]);
```

您现在可以启动消费者：

```bash
$ php bin/console messenger:consume async_bus
```

就是这样！现在，每当从此缓存池查询项时，都会立即返回其缓存值。如果它被选择提前过期，将通过总线发送一条消息来安排后台计算以刷新该值。

[probabilistic-early-expiration]: https://en.wikipedia.org/wiki/Cache_stampede#Probabilistic_early_expiration
