# 缓存池和支持的适配器

缓存池是缓存项的逻辑存储库。它们对项执行所有常见操作，例如保存或查找它们。缓存池独立于实际的缓存实现。因此，即使底层缓存机制从基于文件系统的缓存更改为基于 Redis 或数据库的缓存，应用也可以继续使用相同的缓存池。

## 创建缓存池

缓存池通过**缓存适配器**创建，缓存适配器是同时实现 `Symfony\Contracts\Cache\CacheInterface` 和 `Psr\Cache\CacheItemPoolInterface` 的类。此组件提供了几个可在应用中使用的适配器。

可用的缓存适配器：
* [APCu 缓存适配器](./adapters/apcu_adapter.md)
* [Array 缓存适配器](./adapters/array_cache_adapter.md)
* [Chain 适配器](./adapters/chain_adapter.md)
* [CouchbaseCollection 适配器](./adapters/couchbasecollection_adapter.md)
* [Doctrine DBAL 适配器](./adapters/doctrine_dbal_adapter.md)
* [Filesystem 适配器](./adapters/filesystem_adapter.md)
* [Memcached 适配器](./adapters/memcached_adapter.md)
* [PDO 适配器](./adapters/pdo_adapter.md)
* [PHP Array 缓存适配器](./adapters/php_array_cache_adapter.md)
* [PHP Files 适配器](./adapters/php_files_adapter.md)
* [Proxy 适配器](./adapters/proxy_adapter.md)
* [Redis 适配器](./adapters/redis_adapter.md)

## 使用 Cache Contracts

`Symfony\Contracts\Cache\CacheInterface` 允许仅使用两个方法和一个回调来获取、存储和删除缓存项：

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;
use Symfony\Contracts\Cache\ItemInterface;

$cache = new FilesystemAdapter();

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

使用此接口通过锁定和早期过期提供自动雪崩保护。早期过期可以通过 `Symfony\Contracts\Cache\CacheInterface::get` 方法的第三个"beta"参数控制。有关更多信息，请参阅 [Cache 组件](../cache.md)文章。

可以通过调用 `Symfony\Contracts\Cache\ItemInterface::isHit` 方法在回调内检测早期过期：如果返回 `true`，则意味着我们当前正在提前重新计算其过期日期之前的值。

对于高级用例，回调可以接受通过引用传递的第二个 `bool &$save` 参数。通过在回调内将 `$save` 设置为 `false`，您可以指示缓存池返回的值*不应*存储在后端。

## 使用 PSR-6

### 查找缓存项

缓存池定义了三个用于查找缓存项的方法。最常见的方法是 `getItem($key)`，它返回由给定键标识的缓存项：

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new FilesystemAdapter('app.cache');
$latestNews = $cache->getItem('latest_news');
```

如果没有为给定键定义项，该方法不会返回 `null` 值，而是返回实现 `Symfony\Component\Cache\CacheItem` 类的空对象。

如果您需要同时获取多个缓存项，请改用 `getItems([$key1, $key2, ...])` 方法：

```php
// ...
$stocks = $cache->getItems(['AAPL', 'FB', 'GOOGL', 'MSFT']);
```

同样，如果任何键不代表有效的缓存项，您不会得到 `null` 值，而是一个空的 `CacheItem` 对象。

与获取缓存项相关的最后一个方法是 `hasItem($key)`，如果存在由给定键标识的缓存项，则返回 `true`：

```php
// ...
$hasBadges = $cache->hasItem('user_'.$userId.'_badges');
```

### 保存缓存项

保存缓存项最常见的方法是 `Psr\Cache\CacheItemPoolInterface::save`，它立即将项存储在缓存中（如果保存了项，则返回 `true`；如果发生某些错误，则返回 `false`）：

```php
// ...
$userFriends = $cache->getItem('user_'.$userId.'_friends');
$userFriends->set($user->getFriends());
$isSaved = $cache->save($userFriends);
```

有时您可能更喜欢不立即保存对象以提高应用性能。在这些情况下，使用 `Psr\Cache\CacheItemPoolInterface::saveDeferred` 方法将缓存项标记为"准备持久化"，然后在准备好持久化所有项时调用 `Psr\Cache\CacheItemPoolInterface::commit` 方法：

```php
// ...
$isQueued = $cache->saveDeferred($userFriends);
// ...
$isQueued = $cache->saveDeferred($userPreferences);
// ...
$isQueued = $cache->saveDeferred($userRecentProducts);
// ...
$isSaved = $cache->commit();
```

`saveDeferred()` 方法在缓存项成功添加到"持久化队列"时返回 `true`，否则返回 `false`。`commit()` 方法在所有待处理项成功保存时返回 `true`，否则返回 `false`。

### 删除缓存项

缓存池包括删除一个缓存项、其中一些或所有缓存项的方法。最常见的是 `Psr\Cache\CacheItemPoolInterface::deleteItem`，它删除由给定键标识的缓存项（当项成功删除或不存在时返回 `true`，否则返回 `false`）：

```php
// ...
$isDeleted = $cache->deleteItem('user_'.$userId);
```

使用 `Psr\Cache\CacheItemPoolInterface::deleteItems` 方法同时删除多个缓存项（仅当所有项都已删除时才返回 `true`，即使任何或某些项不存在）：

```php
// ...
$areDeleted = $cache->deleteItems(['category1', 'category2']);
```

最后，要删除存储在池中的所有缓存项，请使用 `Psr\Cache\CacheItemPoolInterface::clear` 方法（当所有项成功删除时返回 `true`）：

```php
// ...
$cacheIsEmpty = $cache->clear();
```

> [!TIP]
> 如果缓存组件在 Symfony 应用中使用，您可以使用以下命令从缓存池中删除项（这些命令位于框架包中）：
> 
> 要从*给定池*中删除*一个特定项*：
> 
> ```bash
> $ php bin/console cache:pool:delete <cache-pool-name> <cache-key-name>
> 
> # 从 "cache.app" 池中删除 "cache_key" 项
> $ php bin/console cache:pool:delete cache.app cache_key
> ```
> 
> 您还可以从*给定池*中删除*所有项*：
> 
> ```bash
> $ php bin/console cache:pool:clear <cache-pool-name>
> 
> # 清除 "cache.app" 池
> $ php bin/console cache:pool:clear cache.app
> 
> # 清除 "cache.validation" 和 "cache.app" 池
> $ php bin/console cache:pool:clear cache.validation cache.app
> ```

## 修剪缓存项

某些缓存池不包括修剪过期缓存项的自动化机制。例如，[FilesystemAdapter](./adapters/filesystem_adapter.md) 缓存不会删除过期的缓存项，*直到显式请求某个项并确定其已过期*，例如，通过调用 `Psr\Cache\CacheItemPoolInterface::getItem`。在某些工作负载下，这可能导致过时的缓存条目在其过期时间之后持续存在，导致过多过期缓存项浪费大量磁盘或内存空间。

此缺陷已通过引入 `Symfony\Component\Cache\PruneableInterface` 解决，该接口定义了抽象方法 `Symfony\Component\Cache\PruneableInterface::prune`。[ChainAdapter](./adapters/chain_adapter.md)、[DoctrineDbalAdapter](./adapters/doctrine_dbal_adapter.md)、[FilesystemAdapter](./adapters/filesystem_adapter.md)、[PdoAdapter](./adapters/pdo_adapter.md) 和 [PhpFilesAdapter](./adapters/php_files_adapter.md) 都实现了此新接口，允许手动删除过时的缓存项：

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new FilesystemAdapter('app.cache');
// ... 执行一些 set 和 get 操作
$cache->prune();
```

[ChainAdapter](./adapters/chain_adapter.md) 实现本身不直接包含任何修剪逻辑。相反，当调用链适配器的 `Symfony\Component\Cache\Adapter\ChainAdapter::prune` 方法时，调用会委托给其所有兼容的缓存适配器（那些不实现 `PruneableInterface` 的会被静默忽略）：

```php
use Symfony\Component\Cache\Adapter\ApcuAdapter;
use Symfony\Component\Cache\Adapter\ChainAdapter;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;
use Symfony\Component\Cache\Adapter\PdoAdapter;
use Symfony\Component\Cache\Adapter\PhpFilesAdapter;

$cache = new ChainAdapter([
    new ApcuAdapter(),       // 不实现 PruneableInterface
    new FilesystemAdapter(), // 实现 PruneableInterface
    new PdoAdapter(),        // 实现 PruneableInterface
    new PhpFilesAdapter(),   // 实现 PruneableInterface
    // ...
]);

// prune 会将调用代理到 PdoAdapter、FilesystemAdapter 和 PhpFilesAdapter，
// 同时静默跳过 ApcuAdapter
$cache->prune();
```

> [!TIP]
> 如果缓存组件在 Symfony 应用中使用，您可以使用以下命令从*所有池*中修剪*所有项*（该命令位于框架包中）：
> 
> ```bash
> $ php bin/console cache:pool:prune
> ```
