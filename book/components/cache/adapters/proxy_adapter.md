# Proxy 缓存适配器

此适配器包装了符合 [PSR-6][psr-6] 的[缓存项池接口][cache-item-pool-interface]。它用于通过使用任何 `Psr\Cache\CacheItemPoolInterface` 的实现，将应用的缓存项池实现与 Symfony Cache 组件集成。

它还可以用于在将项存储到装饰的池中之前自动为所有键添加前缀，有效地允许从单个池创建多个命名空间池。

此适配器期望将 `Psr\Cache\CacheItemPoolInterface` 实例作为其第一个参数，并可选择将命名空间和默认缓存生命周期作为其第二个和第三个参数：

```php
use Psr\Cache\CacheItemPoolInterface;
use Symfony\Component\Cache\Adapter\ProxyAdapter;

// 创建您自己的实现 PSR-6 CacheItemPoolInterface 的缓存池实例
$psr6CachePool = ...

$cache = new ProxyAdapter(

    // 缓存池实例
    CacheItemPoolInterface $psr6CachePool,

    // 作为存储在此缓存中的项的键的前缀的字符串
    $namespace = '',

    // 未定义自己生命周期的缓存项的默认生命周期（以秒为单位），
    // 值为 0 会导致项被无限期存储（即直到缓存被清除）
    $defaultLifetime = 0
);
```

[psr-6]: https://www.php-fig.org/psr/psr-6/
[cache-item-pool-interface]: https://www.php-fig.org/psr/psr-6/#cacheitempoolinterface
