# Cache 组件

Cache 组件提供涵盖简单到高级缓存需求的功能。它原生实现了 [PSR-6][psr-6] 和 [Cache Contracts][cache-contracts]，以实现最大的互操作性。它专为性能和弹性而设计，附带了适用于最常见缓存后端的即用型适配器。它通过锁定和早期过期实现基于标签的失效和缓存雪崩保护。

> [!TIP]
> 该组件还包含在 PSR-6 和 PSR-16 之间转换的适配器。请参阅 [PSR-6 和 PSR-16 缓存适配器](./cache/psr6_psr16_adapters.md)。

## 安装

```bash
$ composer require symfony/cache
```

如果您在 Symfony 应用之外使用此组件，则必须在代码中引入 Composer 生成的 `vendor/autoload.php` 文件来启用类自动加载机制。更多信息请阅读[此文章](./using_components.md)。

## Cache Contracts 与 PSR-6

此组件包含*两种*不同的缓存方法：

**PSR-6 缓存**：  
一个通用缓存系统，涉及缓存"池（pools）"和缓存"项（items）"。

**Cache Contracts**：  
一种更简单但更强大的方式，基于重新计算回调来缓存值。

> [!TIP]
> 建议使用 Cache Contracts 方法：它需要更少的样板代码，并默认提供缓存雪崩保护。

## Cache Contracts

所有适配器都支持 Cache Contracts。它们只包含两个方法：`get()` 和 `delete()`。没有 `set()` 方法，因为 `get()` 方法既获取又设置缓存值。

您需要做的第一件事是实例化一个缓存适配器。本示例中使用了 `Symfony\Component\Cache\Adapter\FilesystemAdapter`：

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new FilesystemAdapter();
```

现在您可以使用此对象检索和删除缓存数据。`get()` 方法的第一个参数是键（key），这是一个任意字符串，您将其与缓存值关联，以便以后可以检索它。第二个参数是一个 PHP 可调用对象，当在缓存中找不到键时执行，以生成并返回值：

```php
use Symfony\Contracts\Cache\ItemInterface;

// 只有在缓存未命中时才会执行 callable
$value = $cache->get('my_cache_key', function (ItemInterface $item): string {
    $item->expiresAfter(3600);

    // ... 执行一些 HTTP 请求或耗时计算
    $computedValue = 'foobar';

    return $computedValue;
});

echo $value; // 'foobar'

// ... 要删除缓存键
$cache->delete('my_cache_key');
```

> [!NOTE]
> 使用缓存标签一次删除多个键。在[缓存失效](./cache/cache_invalidation.md)中阅读更多内容。

## 创建子命名空间

有时您需要创建应该缓存的数据的上下文相关变体。例如，用于呈现仪表板页面的数据可能生成成本高昂且每个用户都是唯一的，因此您不能为每个人缓存相同的数据。

在这种情况下，Symfony 允许您使用命名空间创建不同的缓存上下文。缓存命名空间是标识一组相关缓存项的任意字符串。该组件提供的所有缓存适配器都实现了 `Symfony\Contracts\Cache\NamespacedPoolInterface`，它提供了 `Symfony\Contracts\Cache\NamespacedPoolInterface::withSubNamespace` 方法。

此方法允许您通过透明地为键添加前缀来对缓存项进行命名空间划分：

```php
$userCache = $cache->withSubNamespace(sprintf('user-%d', $user->getId()));

$userCache->get('dashboard_data', function (ItemInterface $item): string {
    $item->expiresAfter(3600);

    return '...';
});
```

在此示例中，缓存项使用 `dashboard_data` 键，但它将在基于当前用户 ID 的命名空间下内部存储。这是自动处理的，因此您**不需要**手动为键添加前缀，如 `user-27.dashboard_data`。

如何定义缓存命名空间没有指导方针或限制。您可以根据应用的需要使它们尽可能细粒度或通用：

```php
$localeCache = $cache->withSubNamespace($request->getLocale());

$flagCache = $cache->withSubNamespace(
    $featureToggle->isEnabled('new_checkout') ? 'checkout-v2' : 'checkout-v1'
);

$channel = $request->attributes->get('_route')?->startsWith('api_') ? 'api' : 'web';
$channelCache = $cache->withSubNamespace($channel);
```

> [!TIP]
> 您可以将缓存命名空间与[缓存标签](../cache.md#使用缓存标签)结合使用，以满足更高级的需求。

没有内置方式按命名空间使缓存失效。相反，推荐的方法是更改命名空间本身。因此，在缓存命名空间中包含静态或动态版本控制数据是很常见的：

```php
// 对于简单的应用，递增的静态版本号可能就足够了
$userCache = $cache->withSubNamespace(sprintf('v1-user-%d', $user->getId()));

// 其他应用可能使用基于日期的动态版本控制（例如每月）
$userCache = $cache->withSubNamespace(sprintf('%s-user-%d', date('Ym'), $user->getId()));

// 甚至在用户数据更改时使缓存失效
$checksum = hash('xxh128', $user->getUpdatedAt()->format(DATE_ATOM));
$userCache = $cache->withSubNamespace(sprintf('user-%d-%s', $user->getId(), $checksum));
```

### 雪崩预防（Stampede Prevention）

Cache Contracts 还内置了[雪崩预防（Stampede prevention）][stampede-prevention]。这将在缓存冷（cold）的时刻消除 CPU 峰值。如果一个示例应用花费 5 秒来计算缓存 1 小时的数据，并且该数据每秒被访问 10 次，这意味着您大多数时候都有缓存命中，一切都很好。但 1 小时后，我们会收到 10 个对冷缓存的新请求。因此数据会再次计算。下一秒同样的事情发生。因此，在缓存再次预热之前，数据会被计算约 50 次。这就是您需要雪崩预防的地方。

第一种解决方案是使用锁定：只允许一个 PHP 进程（基于每个主机）一次计算特定的键。锁定默认内置，因此您无需执行任何操作，只需利用 Cache Contracts 即可。

第二种解决方案在使用 Cache Contracts 时也是内置的：不是等待完整延迟后才使值过期，而是在其过期日期之前重新计算它。[概率性早期过期（Probabilistic early expiration）][probabilistic-early-expiration]算法会随机为一个用户伪造缓存未命中，而其他用户仍然可以获得缓存值。您可以使用 `Symfony\Contracts\Cache\CacheInterface::get` 的第三个可选参数控制其行为，该参数是一个名为"beta"的浮点值。

默认情况下，beta 为 `1.0`，更高的值意味着更早重新计算。将其设置为 `0` 以禁用早期重新计算，将其设置为 `INF` 以强制立即重新计算：

```php
use Symfony\Contracts\Cache\ItemInterface;

$beta = 1.0;
$value = $cache->get('my_cache_key', function (ItemInterface $item): string {
    $item->expiresAfter(3600);
    $item->tag(['tag_0', 'tag_1']);

    return '...';
}, $beta);
```

### 可用的缓存适配器

以下缓存适配器可用：

* [APCu 缓存适配器](./cache/adapters/apcu_adapter.md)
* [数组缓存适配器](./cache/adapters/array_cache_adapter.md)
* [链式适配器](./cache/adapters/chain_adapter.md)
* [CouchbaseCollection 适配器](./cache/adapters/couchbasecollection_adapter.md)
* [Doctrine DBAL 适配器](./cache/adapters/doctrine_dbal_adapter.md)
* [文件系统适配器](./cache/adapters/filesystem_adapter.md)
* [Memcached 适配器](./cache/adapters/memcached_adapter.md)
* [PDO 适配器](./cache/adapters/pdo_adapter.md)
* [PHP Array 缓存适配器](./cache/adapters/php_array_cache_adapter.md)
* [PHP Files 适配器](./cache/adapters/php_files_adapter.md)
* [代理适配器](./cache/adapters/proxy_adapter.md)
* [Redis 适配器](./cache/adapters/redis_adapter.md)

## 通用缓存（PSR-6）

要使用通用 PSR-6 缓存功能，您需要了解其关键概念：

**Item（项）**  
作为键/值对存储的单个信息单元，其中键是信息的唯一标识符，值是其内容；有关更多详细信息，请参阅[缓存项](./cache/cache_items.md)文章。

**Pool（池）**  
缓存项的逻辑存储库。所有缓存操作（保存项、查找项等）都通过池执行。应用可以根据需要定义任意数量的池。

**Adapter（适配器）**  
它实现实际的缓存机制，以在文件系统、数据库等中存储信息。该组件为常见的缓存后端（Redis、APCu、PDO 等）提供了几个即用型适配器。

## 基本用法（PSR-6）

该组件的这一部分是 [PSR-6][psr-6] 的实现，这意味着其基本 API 与文档中定义的相同。在开始缓存信息之前，使用任何内置适配器创建缓存池。例如，要创建基于文件系统的缓存，请实例化 `Symfony\Component\Cache\Adapter\FilesystemAdapter`：

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new FilesystemAdapter();
```

现在您可以使用此缓存池创建、检索、更新和删除项：

```php
// 通过尝试从缓存中获取来创建新项
$productsCount = $cache->getItem('stats.products_count');

// 为项分配值并保存它
$productsCount->set(4711);
$cache->save($productsCount);

// 检索缓存项
$productsCount = $cache->getItem('stats.products_count');
if (!$productsCount->isHit()) {
    // ... 项在缓存中不存在
}
// 检索项存储的值
$total = $productsCount->get();

// 删除缓存项
$cache->deleteItem('stats.products_count');
```

有关所有支持的适配器的列表，请参阅[缓存池](./cache/cache_pools.md)。

## 编组（序列化）数据

> [!NOTE]
> [编组（Marshalling）][marshalling]和[序列化（serializing）][serializing]是类似的概念。序列化是将对象状态转换为可以存储（例如在文件中）的格式的过程。编组是将对象状态及其代码库转换为可以存储或传输的格式的过程。
> 
> 反编组（Unmarshalling）对象会生成原始对象的副本，可能会自动加载对象的类定义。

Symfony 使用*编组器（marshallers）*（实现 `Symfony\Component\Cache\Marshaller\MarshallerInterface` 的类）在存储缓存项之前处理它们。

`Symfony\Component\Cache\Marshaller\DefaultMarshaller` 默认使用 PHP 的 `serialize()` 函数，但您可以选择使用来自 [Igbinary 扩展][igbinary-extension]的 `igbinary_serialize()` 函数：

```php
use Symfony\Component\Cache\Adapter\RedisAdapter;
use Symfony\Component\Cache\Marshaller\DefaultMarshaller;
use Symfony\Component\Cache\Marshaller\DeflateMarshaller;

$marshaller = new DeflateMarshaller(new DefaultMarshaller());
// 如果已安装 Igbinary 扩展，您可以选择使用它
// $marshaller = new DeflateMarshaller(new DefaultMarshaller(useIgbinarySerialize: true));

$cache = new RedisAdapter(new \Redis(), 'namespace', 0, $marshaller);
```

还有其他*编组器*可以在存储数据之前加密或压缩数据。

## 高级用法

* [缓存失效](./cache/cache_invalidation.md)
* [缓存项](./cache/cache_items.md)
* [缓存池](./cache/cache_pools.md)
* [PSR-6 和 PSR-16 缓存适配器](./cache/psr6_psr16_adapters.md)

[psr-6]: https://www.php-fig.org/psr/psr-6/
[cache-contracts]: https://github.com/symfony/contracts/blob/master/Cache/CacheInterface.php
[stampede-prevention]: https://en.wikipedia.org/wiki/Cache_stampede
[probabilistic-early-expiration]: https://en.wikipedia.org/wiki/Cache_stampede#Probabilistic_early_expiration
[marshalling]: https://en.wikipedia.org/wiki/Marshalling_(computer_science)
[serializing]: https://en.wikipedia.org/wiki/Serialization
[igbinary-extension]: https://github.com/igbinary/igbinary
