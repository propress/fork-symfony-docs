# 缓存（Cache）

使用缓存是让应用运行更快的好方法。Symfony 的缓存组件内置了许多针对不同存储后端的适配器，每个适配器都以高性能为目标开发。

以下示例展示了缓存的典型用法：

```php
use Symfony\Contracts\Cache\ItemInterface;

// 该回调函数仅在缓存未命中时执行
$value = $pool->get('my_cache_key', function (ItemInterface $item): string {
    $item->expiresAfter(3600);

    // ... 执行 HTTP 请求或繁重计算
    $computedValue = 'foobar';

    return $computedValue;
});

echo $value; // 'foobar'

// ... 删除缓存键
$pool->delete('my_cache_key');
```

Symfony 支持 Cache Contracts 和 PSR-6/16 接口。你可以在[组件文档](components/cache.md)中了解更多。

---

## 使用 FrameworkBundle 配置缓存 {#cache-configuration-with-frameworkbundle}

配置缓存组件时，你需要了解以下几个概念：

**Pool（池）**
这是你与之交互的服务。每个池都有自己独立的命名空间和缓存条目，不同池之间永远不会发生冲突。

**Adapter（适配器）**
适配器是用于创建池的*模板*。

**Provider（提供者）**
提供者是某些适配器用来连接存储后端的服务。Redis 和 Memcached 是此类适配器的示例。如果使用 DSN 作为提供者，则会自动创建一个服务。

缓存组件内置了一系列预配置的适配器：

- [cache.adapter.apcu](components/cache/adapters/apcu_adapter.md)
- [cache.adapter.array](components/cache/adapters/array_cache_adapter.md)
- [cache.adapter.doctrine_dbal](components/cache/adapters/doctrine_dbal_adapter.md)
- [cache.adapter.filesystem](components/cache/adapters/filesystem_adapter.md)
- [cache.adapter.memcached](components/cache/adapters/memcached_adapter.md)
- [cache.adapter.pdo](components/cache/adapters/pdo_adapter.md)
- [cache.adapter.psr6](components/cache/adapters/proxy_adapter.md)
- [cache.adapter.redis](components/cache/adapters/redis_adapter.md)
- `cache.adapter.redis_tag_aware`（针对标签优化的 Redis 适配器）

> **注意**
>
> 还有一个特殊的 `cache.adapter.system` 适配器，建议将其用于[系统缓存](#系统缓存与应用缓存)。此适配器使用一些逻辑根据你的系统（PHP 文件或 APCu）动态选择最佳存储方式。

部分适配器可通过快捷方式配置：

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

---

## 系统缓存与应用缓存 {#cache-app-system}

默认情况下，始终启用两个缓存池：`cache.system` 和 `cache.app`。

`cache.system` 由 Symfony 组件（如注解、序列化器和验证）**在内部**使用。它也**可用于应用**代码，但有特定限制：

1. 条目必须可从源代码推导出来，并可在 `CacheWarmer` 预热期间重新生成。
2. 缓存内容仅在源代码变更时才能变化（即在部署时，而非运行时）；部署后视其为只读。

默认情况下，`cache.system` 使用 `cache.adapter.system`，该适配器写入文件系统并在可用时链式使用 APCu。大多数情况下，默认配置是正确的选择。

> **提示**
>
> 虽然可以重新配置 `system` 缓存，但建议保持 Symfony 应用的默认配置。

`cache.app` 是应用和 Bundle 代码的**通用数据缓存**。此池中的数据不需要在部署时清空。它默认使用 `cache.adapter.filesystem`，但在可用时建议配置更快的适配器（如 Redis），以确保缓存数据在部署后仍然有效，并在多服务器环境中的多个实例间共享。

[自定义池](#创建自定义命名空间池)默认使用 `cache.app` 作为其适配器，除非另行配置。使用**自动装配**时，`cache.app` 会自动注入到任何类型提示为 `CacheItemPoolInterface`、`AdapterInterface` 或 `CacheInterface` 的服务参数中。

你可以通过 `app` 和 `system` 键配置每个预定义池所使用的适配器：

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

---

## 创建自定义（命名空间）池 {#cache-create-pools}

你也可以创建更多自定义池：

```yaml
# config/packages/cache.yaml
framework:
    cache:
        default_memcached_provider: 'memcached://localhost'

        pools:
            # 创建 "custom_thing.cache" 服务
            # 可通过 "CacheInterface $customThingCache" 自动装配
            # 使用 "app" 缓存配置
            custom_thing.cache:
                adapter: cache.app

            # 创建 "my_cache_pool" 服务
            # 可通过 "CacheInterface $myCachePool" 自动装配
            my_cache_pool:
                adapter: cache.adapter.filesystem

            # 使用上面的 default_memcached_provider
            acme.cache:
                adapter: cache.adapter.memcached

            # 控制适配器配置
            foobar.cache:
                adapter: cache.adapter.memcached
                provider: 'memcached://user:password@example.com'

            # 使用 "foobar.cache" 池作为后端，但控制生命周期
            # （和所有池一样，具有独立的缓存命名空间）
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
                'custom_thing.cache' => ['adapter' => 'cache.app'],
                'my_cache_pool' => ['adapter' => 'cache.adapter.filesystem'],
                'acme.cache' => ['adapter' => 'cache.adapter.memcached'],
                'foobar.cache' => [
                    'adapter' => 'cache.adapter.memcached',
                    'provider' => 'memcached://user:password@example.com',
                ],
                'short_cache' => [
                    'adapter' => 'foobar.cache',
                    'default_lifetime' => 60,
                ],
            ],
        ],
    ],
]);
```

每个池管理一组独立的缓存键：不同池的键*永远*不会冲突，即使它们共享同一后端。这是通过为键添加命名空间前缀实现的，前缀由池名称、缓存适配器类名称以及一个[可配置的种子](reference/configuration/framework.md)（默认为项目目录和已编译的容器类）的哈希值生成。

每个自定义池都会成为一个服务，服务 ID 就是池的名称（例如 `custom_thing.cache`）。同时为每个池创建一个使用驼峰命名法的自动装配别名——例如，`custom_thing.cache` 可以通过将参数命名为 `$customThingCache` 并类型提示为 `CacheInterface` 或 `CacheItemPoolInterface` 来自动注入：

```php
use Symfony\Contracts\Cache\CacheInterface;
// ...

// 在控制器方法中
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

> **提示**
>
> 如果需要命名空间与第三方应用互操作，可以通过设置 `cache.pool` 服务标签的 `namespace` 属性来控制自动生成过程。例如，可以覆盖适配器的服务定义：
>
> ```yaml
> # config/services.yaml
> services:
>     # ...
>     app.cache.adapter.redis:
>         parent: 'cache.adapter.redis'
>         tags:
>             - { name: 'cache.pool', namespace: 'my_custom_namespace' }
> ```

---

## 自定义提供者选项

部分提供者有特定选项可配置。[RedisAdapter](components/cache/adapters/redis_adapter.md) 允许你创建带有 `timeout`、`retry_interval` 等选项的提供者。要使用非默认值，需要创建自己的 `\Redis` 提供者，并在配置池时使用它。

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

---

## 创建缓存链

不同的缓存适配器各有优缺点。有些非常快但只适合存储小条目，有些可以存储大量数据但速度较慢。为了两全其美，你可以使用适配器链。

缓存链将多个缓存池合并为一个。在缓存链中存储条目时，Symfony 会依次将其存储到所有池中。获取条目时，Symfony 尝试从第一个池获取，如果找不到，则尝试下一个池，直到找到条目或抛出异常。由于这种行为，建议在链中按从最快到最慢的顺序定义适配器。

如果在某个池中存储条目时发生错误，Symfony 会在其他池中存储它，且不会抛出异常。之后检索该条目时，Symfony 会自动将条目存储到所有缺少它的池中。

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

---

## 使用缓存标签 {#cache-using-cache-tags}

在有许多缓存键的应用中，组织存储的数据以便更高效地使缓存失效非常有用。一种实现方式是使用缓存标签。可以向缓存条目添加一个或多个标签，然后通过一次函数调用使所有具有相同标签的条目失效：

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

        // 删除所有带 "bar" 标签的缓存键
        $this->myCachePool->invalidateTags(['bar']);
    }
}
```

缓存适配器需要实现 `TagAwareCacheInterface` 才能启用此功能，可通过以下配置添加：

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

标签默认存储在同一个池中，这在大多数情况下是合适的。但有时将标签存储在不同的池中更好，可以通过指定适配器来实现：

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

> **注意**
>
> `TagAwareCacheInterface` 接口会自动装配到 `cache.app` 服务。

---

## 清除缓存

要清除缓存，可使用 `bin/console cache:pool:clear [pool]` 命令。这将从存储中删除所有条目，你必须重新计算所有值。你也可以将池分组到"缓存清除器"中，默认有 3 个缓存清除器：

- `cache.global_clearer`
- `cache.system_clearer`
- `cache.app_clearer`

全局清除器清除每个池中的所有缓存条目。系统缓存清除器用于 `bin/console cache:clear` 命令。应用清除器是默认的清除器。

查看所有可用的缓存池：

```terminal
$ php bin/console cache:pool:list
```

清除单个池：

```terminal
$ php bin/console cache:pool:clear my_cache_pool
```

清除所有自定义池：

```terminal
$ php bin/console cache:pool:clear cache.app_clearer
```

清除所有缓存池：

```terminal
$ php bin/console cache:pool:clear --all
```

清除除某些池外的所有缓存池：

```terminal
$ php bin/console cache:pool:clear --all --exclude=my_cache_pool --exclude=another_cache_pool
```

清除所有缓存：

```terminal
$ php bin/console cache:pool:clear cache.global_clearer
```

按标签清除缓存：

```terminal
# 从所有支持标签的池中使 tag1 失效
$ php bin/console cache:pool:invalidate-tags tag1

# 从所有支持标签的池中使 tag1 和 tag2 失效
$ php bin/console cache:pool:invalidate-tags tag1 tag2

# 从 cache.app 池中使 tag1 和 tag2 失效
$ php bin/console cache:pool:invalidate-tags tag1 tag2 --pool=cache.app

# 从 cache1 和 cache2 池中使 tag1 和 tag2 失效
$ php bin/console cache:pool:invalidate-tags tag1 tag2 -p cache1 -p cache2
```

---

## 加密缓存

要使用 `libsodium` 加密缓存，可以使用 `SodiumMarshaller`。

首先，生成一个安全密钥并将其作为 `CACHE_DECRYPTION_KEY` 添加到你的 [secret 存储](configuration/secrets.md)：

```terminal
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

> **危险**
>
> 这将加密缓存条目的**值**，但不会加密缓存**键**。请注意不要在键中泄露敏感数据。

配置多个密钥时，第一个密钥用于读写，其他密钥仅用于读取。一旦使用旧密钥加密的所有缓存条目过期，你就可以完全删除 `OLD_CACHE_DECRYPTION_KEY`。

---

## 异步计算缓存值

缓存组件使用[概率性提前过期](https://en.wikipedia.org/wiki/Cache_stampede#Probabilistic_early_expiration)算法来防止[缓存雪崩](components/cache.md)问题。这意味着一些缓存条目在仍然有效时就被选为提前过期。

默认情况下，过期的缓存条目是同步计算的。但是，你可以通过使用 [Messenger 组件](components/messenger.md)将值计算委托给后台工作进程来异步计算它们。在这种情况下，当查询某个条目时，会立即返回其缓存值，并通过 Messenger 总线分发一个 `EarlyExpirationMessage`。

当消息消费者处理此消息时，刷新后的缓存值将异步计算。下次查询该条目时，刷新后的值将是最新的并被返回。

首先，创建一个服务来计算条目的值：

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

        // 这只是一个随机示例；你需要在这里做自己的计算
        return sprintf('#%06X', mt_rand(0, 0xFFFFFF));
    }
}
```

此缓存值将从控制器、其他服务等地方请求。以下示例从控制器中请求该值：

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
        // 传递给缓存刷新条目的服务方法
        $cachedValue = $asyncCache->get('my_value', $cacheComputation);

        // ...
    }
}
```

最后，配置一个新的缓存池（例如名为 `async.cache`），使其使用消息总线在工作进程中计算值：

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

现在启动消费者：

```terminal
$ php bin/console messenger:consume async_bus
```

完成！现在，每当从此缓存池查询条目时，将立即返回其缓存值。如果该条目被选为提前过期，将通过总线发送一条消息，以安排后台计算来刷新该值。
