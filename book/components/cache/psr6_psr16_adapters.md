# PSR-6 和 PSR-16 缓存之间的互操作性适配器

有时，您可能有一个实现 [PSR-16][psr-16] 标准的 Cache 对象，但需要将其传递给期望 PSR-6 缓存适配器的对象。或者，您可能遇到相反的情况。缓存组件包含两个用于 PSR-6 和 PSR-16 缓存之间双向互操作性的类。

## 将 PSR-16 缓存对象用作 PSR-6 缓存

假设您想使用需要 PSR-6 缓存池对象的类。例如：

```php
use Psr\Cache\CacheItemPoolInterface;

// 仅为示例编造的类
class GitHubApiClient
{
    // ...

    // 这需要一个 PSR-6 缓存对象
    public function __construct(CacheItemPoolInterface $cachePool)
    {
        // ...
    }
}
```

但是，您已经有一个 PSR-16 缓存对象，并且您想将其传递给该类。没问题！缓存组件为此用例提供了 `Symfony\Component\Cache\Adapter\Psr16Adapter` 类：

```php
use Symfony\Component\Cache\Adapter\Psr16Adapter;

// $psr16Cache 是您想用作 PSR-6 的 PSR-16 对象

// 内部使用您的缓存的 PSR-6 缓存！
$psr6Cache = new Psr16Adapter($psr16Cache);

// 现在可以在任何地方使用它
$githubApiClient = new GitHubApiClient($psr6Cache);
```

## 将 PSR-6 缓存对象用作 PSR-16 缓存

假设您想使用需要 PSR-16 缓存对象的类。例如：

```php
use Psr\SimpleCache\CacheInterface;

// 仅为示例编造的类
class GitHubApiClient
{
    // ...

    // 这需要一个 PSR-16 缓存对象
    public function __construct(CacheInterface $cache)
    {
        // ...
    }
}
```

但是，您已经有一个 PSR-6 缓存池对象，并且您想将其传递给该类。没问题！缓存组件为此用例提供了 `Symfony\Component\Cache\Psr16Cache` 类：

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;
use Symfony\Component\Cache\Psr16Cache;

// 您想使用的 PSR-6 缓存对象
$psr6Cache = new FilesystemAdapter();

// 内部使用您的缓存的 PSR-16 缓存！
$psr16Cache = new Psr16Cache($psr6Cache);

// 现在可以在任何地方使用它
$githubApiClient = new GitHubApiClient($psr16Cache);
```

[psr-16]: https://www.php-fig.org/psr/psr-16/
