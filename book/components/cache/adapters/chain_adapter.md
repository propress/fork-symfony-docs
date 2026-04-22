# Chain 缓存适配器

此适配器允许组合任意数量的其他可用缓存适配器。从包含它们的第一个适配器中获取缓存项，并将缓存项保存到所有给定的适配器。这提供了一种创建分层缓存的简单而高效的方法。

ChainAdapter 必须在其构造函数参数中提供一个适配器数组和可选的默认缓存生命周期：

```php
use Symfony\Component\Cache\Adapter\ChainAdapter;

$cache = new ChainAdapter(
    // 用于获取缓存项的适配器的有序列表
    array $adapters,

    // 从较低适配器传播到较高适配器的项的默认生命周期
    $defaultLifetime = 0
);
```

> [!NOTE]
> 当在第一个适配器中找不到项但在下一个适配器中找到时，此适配器确保将获取的项保存到之前缺失它的所有适配器。

以下示例展示如何使用最快和最慢的存储引擎（分别是 `Symfony\Component\Cache\Adapter\ApcuAdapter` 和 `Symfony\Component\Cache\Adapter\FilesystemAdapter`）创建链式适配器实例：

```php
use Symfony\Component\Cache\Adapter\ApcuAdapter;
use Symfony\Component\Cache\Adapter\ChainAdapter;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new ChainAdapter([
    new ApcuAdapter(),
    new FilesystemAdapter(),
]);
```

调用此适配器的 `Symfony\Component\Cache\Adapter\ChainAdapter::prune` 方法时，调用会委托给其所有兼容的缓存适配器。混合*实现*和*不*实现 `Symfony\Component\Cache\PruneableInterface` 的适配器是安全的，因为不兼容的适配器会被静默忽略：

```php
use Symfony\Component\Cache\Adapter\ApcuAdapter;
use Symfony\Component\Cache\Adapter\ChainAdapter;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new ChainAdapter([
    new ApcuAdapter(),        // 不实现 PruneableInterface
    new FilesystemAdapter(),  // 实现 PruneableInterface
]);

// prune 会将调用代理到 FilesystemAdapter，同时静默跳过 ApcuAdapter
$cache->prune();
```
