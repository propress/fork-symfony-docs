# 缓存失效（Cache Invalidation）

缓存失效是删除与模型状态更改相关的所有缓存项的过程。最基本的失效类型是直接删除项。但是，当主资源的状态已分布在多个缓存项中时，保持它们同步可能很困难。

Symfony Cache 组件提供了两种机制来帮助解决此问题：

* 用于管理数据依赖关系的[基于标签的失效](#使用缓存标签)；
* 用于时间相关依赖关系的[基于过期的失效](#使用缓存过期)。

## 使用缓存标签

要从基于标签的失效中受益，您需要为每个缓存项附加适当的标签。每个标签都是一个普通的字符串标识符，您可以随时使用它来触发删除与此标签关联的所有项。

要将标签附加到缓存项，您需要使用由缓存项实现的 `Symfony\Contracts\Cache\ItemInterface::tag` 方法：

```php
$item = $cache->get('cache_key', function (ItemInterface $item): string {
    // [...]
    // 添加一个或多个标签
    $item->tag('tag_1');
    $item->tag(['tag_2', 'tag_3']);

    return $cachedValue;
});
```

如果 `$cache` 实现了 `Symfony\Contracts\Cache\TagAwareCacheInterface`，您可以通过调用 `Symfony\Contracts\Cache\TagAwareCacheInterface::invalidateTags` 使缓存项失效：

```php
// 使与 `tag_1` 或 `tag_3` 相关的所有项失效
$cache->invalidateTags(['tag_1', 'tag_3']);

// 如果您知道缓存键，也可以直接删除该项
$cache->delete('cache_key');
```

当跟踪缓存键变得困难时，使用标签失效非常有用。

### Tag Aware 适配器

要存储标签，您需要使用 `Symfony\Component\Cache\Adapter\TagAwareAdapter` 类包装缓存适配器，或实现 `Symfony\Contracts\Cache\TagAwareCacheInterface` 及其 `Symfony\Component\Cache\Adapter\TagAwareAdapterInterface::invalidateTags` 方法。

> [!NOTE]
> 当使用 Redis 后端时，考虑使用为此目的优化的 `RedisTagAwareAdapter`。当使用文件系统时，同样考虑使用 `FilesystemTagAwareAdapter`。

`Symfony\Component\Cache\Adapter\TagAwareAdapter` 类实现了即时失效（时间复杂度为 `O(N)`，其中 `N` 是失效标签的数量）。它需要一个或两个缓存适配器：第一个必需的适配器用于存储缓存项；第二个可选的适配器用于存储标签及其失效版本号（在概念上类似于它们的最新失效日期）。当仅使用一个适配器时，项和标签都存储在同一位置。通过使用两个适配器，您可以例如将一些大型缓存项存储在文件系统或数据库中，并将标签保存在 Redis 数据库中，以同步所有前端并进行非常快速的失效检查：

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;
use Symfony\Component\Cache\Adapter\RedisAdapter;
use Symfony\Component\Cache\Adapter\TagAwareAdapter;

$cache = new TagAwareAdapter(
    // 用于缓存项的适配器
    new FilesystemAdapter(),
    // 用于标签的适配器
    new RedisAdapter('redis://localhost')
);
```

> [!NOTE]
> `Symfony\Component\Cache\Adapter\TagAwareAdapter` 实现了 `Symfony\Component\Cache\PruneableInterface`，允许通过调用其 `Symfony\Component\Cache\Adapter\TagAwareAdapter::prune` 方法手动修剪过期的缓存条目（假设包装的适配器本身实现了 `Symfony\Component\Cache\PruneableInterface`）。

## 使用缓存过期

如果您的数据仅在有限的时间段内有效，您可以使用 PSR-6 接口指定它们的生命周期或过期日期，如[缓存项](./cache_items.md)文章中所述。
