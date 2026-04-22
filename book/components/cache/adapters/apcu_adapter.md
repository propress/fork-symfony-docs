# APCu 缓存适配器

此适配器是一个高性能的共享内存缓存。它可以*显著*提高应用的性能，因为其缓存内容存储在共享内存中，这一组件明显比许多其他组件（如文件系统）更快。

> [!WARNING]
> **要求：**必须安装并激活 [APCu 扩展][apcu-extension]才能使用此适配器。

ApcuAdapter 可以选择性地在构造函数参数中提供命名空间、默认缓存生命周期和缓存项版本字符串：

```php
use Symfony\Component\Cache\Adapter\ApcuAdapter;

$cache = new ApcuAdapter(

    // 作为存储在此缓存中的项的键的前缀的字符串
    $namespace = '',

    // 未定义自己生命周期的缓存项的默认生命周期（以秒为单位），
    // 值为 0 会导致项被无限期存储（即直到 APCu 内存被清除）
    $defaultLifetime = 0,

    // 设置后，可以通过更改此 $version 字符串
    // 使所有以 $namespace 为前缀的键失效
    $version = null
);
```

> [!WARNING]
> 在写入/删除密集型工作负载中不建议使用此适配器，因为这些操作会导致内存碎片，从而导致性能显著下降。

> [!TIP]
> 此适配器的 CRUD 操作特定于它所运行的 PHP SAPI。这意味着使用 CLI 的缓存操作（如添加、删除等）在 FPM 或 CGI SAPI 下将不可用。

[apcu-extension]: https://pecl.php.net/package/APCu
