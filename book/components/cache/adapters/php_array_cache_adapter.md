# PHP Array 缓存适配器

此适配器是用于静态数据（例如应用配置）的高性能缓存，经过优化并预加载到 OPcache 内存存储中。它适用于预热后大部分为只读的任何数据：

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;
use Symfony\Component\Cache\Adapter\PhpArrayAdapter;

// 以某种方式，决定是时候预热缓存了！
if ($needsWarmup) {
    // 一些静态值
    $values = [
        'stats.products_count' => 4711,
        'stats.users_count' => 1356,
    ];

    $cache = new PhpArrayAdapter(
        // 缓存值的单个文件
        __DIR__ . '/somefile.cache',
        // 备份适配器，如果在预热后设置值
        new FilesystemAdapter()
    );
    $cache->warmUp($values);
}

// ... 然后，使用缓存！
$cacheItem = $cache->getItem('stats.users_count');
echo $cacheItem->get();
```

> [!NOTE]
> 此适配器需要打开 `opcache.enable` php.ini 设置。
