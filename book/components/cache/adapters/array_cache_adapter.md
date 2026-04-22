# Array 缓存适配器

通常，此适配器对于测试目的很有用，因为其内容存储在内存中，不会以任何方式持久化到运行的 PHP 进程之外。由于 `Symfony\Component\Cache\Adapter\ArrayAdapter::getValues` 方法，它在预热缓存时也很有用：

```php
use Symfony\Component\Cache\Adapter\ArrayAdapter;

$cache = new ArrayAdapter(

    // 未定义自己生命周期的缓存项的默认生命周期（以秒为单位），
    // 值为 0 会导致项被无限期存储（即直到当前 PHP 进程完成）
    $defaultLifetime = 0,

    // 如果为 true，则在存储之前序列化保存在缓存中的值
    $storeSerialized = true,

    // 整个缓存的最大生命周期（以秒为单位）（在此时间之后，
    // 删除整个缓存以避免过时数据占用内存）
    $maxLifetime = 0,

    // 可以存储在缓存中的最大项数。当达到限制时，
    // 缓存遵循 LRU 模型（删除最近最少使用的项）
    $maxItems = 0,

    // Psr\Clock\ClockInterface 的可选实现，将用于
    // 计算缓存项的生命周期（例如在测试中获得可预测的生命周期）
    $clock = null,
);
```
