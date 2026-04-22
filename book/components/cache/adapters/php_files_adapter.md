# PHP Files 缓存适配器

与 [Filesystem 适配器](./filesystem_adapter.md)类似，此缓存实现将缓存条目写出到磁盘，但与 Filesystem 缓存适配器不同，PHP Files 缓存适配器将这些缓存文件*作为原生 PHP 代码*写入和读回。例如，缓存值 `['my', 'cached', 'array']` 将写出类似于以下内容的缓存文件：

```php
<?php return [

    // 缓存项过期时间
    0 => 9223372036854775807,

    // 缓存项内容
    1 => [
        0 => 'my',
        1 => 'cached',
        2 => 'array',
    ],

];
```

> [!NOTE]
> 此适配器需要打开 `opcache.enable` php.ini 设置。由于缓存项作为原生 PHP 代码包含和解析，并且由于 [OPcache][opcache] 处理文件包含的方式，此适配器有可能比其他基于文件系统的缓存快得多。

> [!WARNING]
> 虽然它支持更新，并且因为它使用 OPcache 作为后端，但此适配器更适合仅追加（append-mostly）的需求。在其他场景中使用它可能会导致 OPcache 内存的周期性重置，可能导致性能下降。

PhpFilesAdapter 可以选择性地在构造函数参数中提供命名空间、默认缓存生命周期和缓存目录路径：

```php
use Symfony\Component\Cache\Adapter\PhpFilesAdapter;

$cache = new PhpFilesAdapter(

    // 用作根缓存目录的子目录的字符串，缓存项将存储在其中
    $namespace = '',

    // 未定义自己生命周期的缓存项的默认生命周期（以秒为单位），
    // 值为 0 会导致项被无限期存储（即直到文件被删除）
    $defaultLifetime = 0,

    // 主缓存目录（应用需要对其具有读写权限）
    // 如果未指定，则在系统临时目录中创建一个目录
    $directory = null
);
```

> [!NOTE]
> 此适配器实现了 `Symfony\Component\Cache\PruneableInterface`，允许通过调用其 `prune()` 方法手动修剪过期的缓存条目。

[opcache]: https://www.php.net/manual/en/book.opcache.php
