# Cache 组件

Cache 组件提供了覆盖从简单到高级缓存需求的功能。它原生实现了 [PSR-6] 和 [Cache Contracts]，以获得最大的互操作性。它以性能和弹性为设计目标，提供了适用于最常见缓存后端的即用适配器，支持基于标签的失效和通过锁定及提前过期实现的缓存踩踏保护。

> **提示：**
> 该组件还包含用于在 PSR-6 和 PSR-16 之间转换的适配器。详见 [/components/cache/psr6_psr16_adapters](psr6_psr16_adapters.md)。

## 安装

```terminal
$ composer require symfony/cache
```

## Cache Contracts 与 PSR-6

此组件包含两种不同的缓存方式：

**PSR-6 缓存**：
一个通用的缓存系统，涉及缓存"池"和缓存"条目"。

**Cache Contracts**：
一种更简单但更强大的基于重新计算回调的缓存值的方式。

> **提示：**
> 推荐使用 Cache Contracts 方式：它需要的样板代码更少，并且默认提供缓存踩踏保护。

## Cache Contracts

所有适配器都支持 Cache Contracts。它们只包含两个方法：`get()` 和 `delete()`。没有 `set()` 方法，因为 `get()` 方法同时完成获取和设置缓存值的功能。

首先需要实例化一个缓存适配器。本例中使用 `Symfony\Component\Cache\Adapter\FilesystemAdapter`：

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new FilesystemAdapter();
```

现在可以使用此对象检索和删除缓存数据。`get()` 方法的第一个参数是键，一个与缓存值关联的任意字符串，以便以后检索。第二个参数是一个 PHP 可调用对象，当在缓存中找不到该键时执行，用于生成并返回值：

```php
use Symfony\Contracts\Cache\ItemInterface;

// The callable will only be executed on a cache miss.
$value = $cache->get('my_cache_key', function (ItemInterface $item): string {
    $item->expiresAfter(3600);

    // ... do some HTTP request or heavy computations
    $computedValue = 'foobar';

    return $computedValue;
});

echo $value; // 'foobar'

// ... and to remove the cache key
$cache->delete('my_cache_key');
```

> **注意：**
> 使用缓存标签可以一次删除多个键。详见 [/components/cache/cache_invalidation](cache_invalidation.md)。

## 创建子命名空间

有时你需要创建应缓存的数据的上下文相关变体。例如，用于渲染仪表板页面的数据可能生成代价高昂且每个用户都是唯一的，因此你不能为所有人缓存相同的数据。

在这种情况下，Symfony 允许你使用命名空间创建不同的缓存上下文。缓存命名空间是一个任意字符串，用于标识一组相关的缓存条目。组件提供的所有缓存适配器都实现了 `Symfony\Contracts\Cache\NamespacedPoolInterface`，它提供了 `Symfony\Contracts\Cache\NamespacedPoolInterface::withSubNamespace` 方法。

此方法允许你通过透明地为键添加前缀来对缓存条目进行命名空间化：

```php
$userCache = $cache->withSubNamespace(sprintf('user-%d', $user->getId()));

$userCache->get('dashboard_data', function (ItemInterface $item): string {
    $item->expiresAfter(3600);

    return '...';
});
```

在此示例中，缓存条目使用 `dashboard_data` 键，但它将在内部以基于当前用户 ID 的命名空间存储。这是自动处理的，因此你**不**需要手动添加 `user-27.dashboard_data` 这样的前缀。

关于如何定义缓存命名空间没有指导原则或限制。你可以根据应用程序的需要将其做得细粒度或通用：

```php
$localeCache = $cache->withSubNamespace($request->getLocale());

$flagCache = $cache->withSubNamespace(
    $featureToggle->isEnabled('new_checkout') ? 'checkout-v2' : 'checkout-v1'
);

$channel = $request->attributes->get('_route')?->startsWith('api_') ? 'api' : 'web';
$channelCache = $cache->withSubNamespace($channel);
```

> **提示：**
> 你可以将缓存命名空间与缓存标签结合使用以满足更高级的需求。

没有内置的方法按命名空间使缓存失效。推荐的方法是更改命名空间本身。因此，在缓存命名空间中包含静态或动态版本数据是很常见的：

```php
// for simple applications, an incrementing static version number may be enough
$userCache = $cache->withSubNamespace(sprintf('v1-user-%d', $user->getId()));

// other applications may use dynamic versioning based on the date (e.g. monthly)
$userCache = $cache->withSubNamespace(sprintf('%s-user-%d', date('Ym'), $user->getId()));

// or even invalidate the cache when the user data changes
$checksum = hash('xxh128', $user->getUpdatedAt()->format(DATE_ATOM));
$userCache = $cache->withSubNamespace(sprintf('user-%d-%s', $user->getId(), $checksum));
```

### 踩踏预防

Cache Contracts 还内置了[踩踏预防]功能。这将消除缓存冷却时刻的 CPU 峰值。如果一个示例应用程序花费 5 秒来计算缓存 1 小时的数据，而该数据每秒被访问 10 次，这意味着你大部分时候都是缓存命中，一切正常。但 1 小时后，有 10 个新请求到达冷缓存。所以数据被重新计算。下一秒同样的事情发生了。因此，在缓存再次预热之前，数据大约被计算了 50 次。这就是你需要踩踏预防的地方。

第一种解决方案是使用锁定：每次只允许一个 PHP 进程（基于每台主机）计算特定的键。锁定默认内置，所以你不需要做任何超出使用 Cache Contracts 的事情。

第二种解决方案在使用 Cache Contracts 时也内置了：不是等到完全延迟后才使值过期，而是在其过期日期之前重新计算它。[概率性提前过期]算法随机地对一个用户模拟缓存未命中，而其他用户仍然获得缓存的值。你可以通过 `Symfony\Contracts\Cache\CacheInterface::get` 的第三个可选参数（一个称为"beta"的浮点值）来控制其行为。

默认情况下 beta 为 `1.0`，值越高意味着越早重新计算。将其设置为 `0` 以禁用提前重新计算，将其设置为 `INF` 以强制立即重新计算：

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

以下缓存适配器可用（详见各适配器文档）。

## 通用缓存（PSR-6）

要使用通用 PSR-6 缓存能力，你需要学习其关键概念：

**条目（Item）**：
以键/值对形式存储的单个信息单元，其中键是信息的唯一标识符，值是其内容。详见 [/components/cache/cache_items](cache_items.md) 文章。

**池（Pool）**：
缓存条目的逻辑存储库。所有缓存操作（保存条目、查找条目等）都通过池执行。应用程序可以根据需要定义任意数量的池。

**适配器（Adapter）**：
实现实际的缓存机制，将信息存储在文件系统、数据库等中。该组件提供了多个适用于常见缓存后端（Redis、APCu、PDO 等）的即用适配器。

## 基本用法（PSR-6）

此组件的这一部分是 [PSR-6] 的实现，这意味着其基本 API 与文档中定义的相同。在开始缓存信息之前，使用任何内置适配器创建缓存池。例如，要创建基于文件系统的缓存，实例化 `Symfony\Component\Cache\Adapter\FilesystemAdapter`：

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new FilesystemAdapter();
```

现在可以使用此缓存池创建、检索、更新和删除条目：

```php
// create a new item by trying to get it from the cache
$productsCount = $cache->getItem('stats.products_count');

// assign a value to the item and save it
$productsCount->set(4711);
$cache->save($productsCount);

// retrieve the cache item
$productsCount = $cache->getItem('stats.products_count');
if (!$productsCount->isHit()) {
    // ... item does not exist in the cache
}
// retrieve the value stored by the item
$total = $productsCount->get();

// remove the cache item
$cache->deleteItem('stats.products_count');
```

有关所有支持的适配器列表，请参阅 [/components/cache/cache_pools](cache_pools.md)。

## 编组（序列化）数据

> **注意：**
> [编组]和[序列化]是类似的概念。序列化是将对象状态转换为可存储格式（例如文件）的过程。编组是将对象状态及其代码库转换为可存储或传输格式的过程。
>
> 反编组对象会生成原始对象的副本，可能通过自动加载对象的类定义来实现。

Symfony 使用*编组器*（实现 `Symfony\Component\Cache\Marshaller\MarshallerInterface` 的类）在存储缓存条目之前处理它们。

`Symfony\Component\Cache\Marshaller\DefaultMarshaller` 默认使用 PHP 的 `serialize()` 函数，但你可以选择使用来自 [Igbinary 扩展]的 `igbinary_serialize()` 函数：

```php
use Symfony\Component\Cache\Adapter\RedisAdapter;
use Symfony\Component\Cache\Marshaller\DefaultMarshaller;
use Symfony\Component\Cache\Marshaller\DeflateMarshaller;

$marshaller = new DeflateMarshaller(new DefaultMarshaller());
// you can optionally use the Igbinary extension if you have it installed
// $marshaller = new DeflateMarshaller(new DefaultMarshaller(useIgbinarySerialize: true));

$cache = new RedisAdapter(new \Redis(), 'namespace', 0, $marshaller);
```

还有其他*编组器*可以在存储数据之前对其进行加密或压缩。

## 高级用法

详见各缓存子主题文档。

[PSR-6]: https://www.php-fig.org/psr/psr-6/
[Cache Contracts]: https://github.com/symfony/contracts/blob/master/Cache/CacheInterface.php
[踩踏预防]: https://en.wikipedia.org/wiki/Cache_stampede
[概率性提前过期]: https://en.wikipedia.org/wiki/Cache_stampede#Probabilistic_early_expiration
[编组]: https://en.wikipedia.org/wiki/Marshalling_(computer_science)
[序列化]: https://en.wikipedia.org/wiki/Serialization
[Igbinary 扩展]: https://github.com/igbinary/igbinary
