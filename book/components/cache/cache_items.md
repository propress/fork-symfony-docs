# 缓存项（Cache Items）

缓存项是作为键/值对存储在缓存中的信息单元。在 Cache 组件中，它们由 `Symfony\Component\Cache\CacheItem` 类表示。它们在 Cache Contracts 和 PSR-6 接口中都使用。

## 缓存项键和值

缓存项的**键（key）**是一个普通字符串，充当其标识符，因此对于每个缓存池必须是唯一的。您可以自由选择键，但它们应仅包含字母（A-Z、a-z）、数字（0-9）以及 `_` 和 `.` 符号。其他常见符号（如 `{ } ( ) / \ @ :`）由 PSR-6 标准保留供将来使用。

缓存项的**值（value）**可以是由 PHP 可序列化的类型表示的任何数据，例如基本类型（字符串、整数、浮点数、布尔值、null）、数组和对象。

## 创建缓存项

创建缓存项的唯一方法是通过缓存池。使用 Cache Contracts 时，它们作为参数传递给重新计算回调：

```php
// $cache 池对象之前已创建
$productsCount = $cache->get('stats.products_count', function (ItemInterface $item) {
    // [...]
});
```

使用 Cache Contracts 时，您还可以在回调内配置缓存项。例如，您可以设置其过期时间：

```php
$productsCount = $cache->get('stats.products_count', function (ItemInterface $item): int {
    $item->expiresAfter(3600); // 缓存 1 小时

    // ... 计算值
    return 4711;
});
```

使用 PSR-6 时，它们通过缓存池的 `getItem($key)` 方法创建：

```php
// $cache 池对象之前已创建
$productsCount = $cache->getItem('stats.products_count');
```

然后，使用 `Psr\Cache\CacheItemInterface::set` 方法设置存储在缓存项中的数据（使用 Cache Contracts 时此步骤会自动完成）：

```php
// 存储简单整数
$productsCount->set(4711);
$cache->save($productsCount);

// 存储数组
$productsCount->set([
    'category1' => 4711,
    'category2' => 2387,
]);
$cache->save($productsCount);
```

任何给定缓存项的键和值都可以通过相应的 *getter* 方法获得：

```php
$cacheItem = $cache->getItem('exchange_rate');
// ...
$key = $cacheItem->getKey();
$value = $cacheItem->get();
```

### 缓存项过期

默认情况下，缓存项是永久存储的。实际上，这种"永久存储"可能会因所使用的缓存类型而大不相同，如[缓存池](./cache_pools.md)文章中所述。

但是，在某些应用中，通常使用寿命较短的缓存项。例如，考虑一个仅缓存最新新闻一分钟的应用。在这些情况下，使用 `expiresAfter()` 方法设置缓存项的秒数：

```php
$latestNews->expiresAfter(60);  // 60 秒 = 1 分钟

// 此方法也接受 \DateInterval 实例
$latestNews->expiresAfter(DateInterval::createFromDateString('1 hour'));
```

缓存项定义了另一个相关方法 `expiresAt()`，用于设置项将过期的确切日期和时间：

```php
$mostPopularNews->expiresAt(new \DateTime('tomorrow'));
```

## 缓存项命中和未命中

使用缓存机制对于提高应用性能很重要，但不应要求使应用工作。实际上，PSR-6 文档明智地指出，缓存错误不应导致应用失败。

实际上，使用 PSR-6，这意味着 `getItem()` 方法始终返回一个实现 `Psr\Cache\CacheItemInterface` 接口的对象，即使缓存项不存在。因此，您不必处理 `null` 返回值，并且可以安全地在缓存中存储 `false` 和 `null` 等值。

为了确定返回的对象是否表示来自存储的值，缓存使用命中和未命中的概念：

* **缓存命中（Cache Hits）**在请求的项在缓存中找到、其值未损坏或无效且尚未过期时发生；
* **缓存未命中（Cache Misses）**与命中相反，因此在缓存中找不到项、其值因任何原因损坏或无效或项已过期时发生。

缓存项对象定义了一个布尔值 `isHit()` 方法，该方法对于缓存命中返回 `true`：

```php
$latestNews = $cache->getItem('latest_news');

if (!$latestNews->isHit()) {
    // 执行一些繁重的计算
    $news = ...;
    $cache->save($latestNews->set($news));
} else {
    $news = $latestNews->get();
}
```
