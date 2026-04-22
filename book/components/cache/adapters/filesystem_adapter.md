# Filesystem 缓存适配器

此适配器为那些无法在其环境中安装 [APCu](./apcu_adapter.md) 或 [Redis](./redis_adapter.md) 等工具的用户提供了改进的应用性能。它将缓存项过期和内容作为常规文件存储在本地挂载文件系统上的目录集合中。

> [!TIP]
> 通过使用临时的内存文件系统，如 Linux 上的 [tmpfs][tmpfs]，或许多其他可用的 [RAM 磁盘解决方案][ram-disk-solutions]之一，可以大大提高此适配器的性能。

FilesystemAdapter 可以选择性地在构造函数参数中提供命名空间、默认缓存生命周期和缓存根路径：

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new FilesystemAdapter(

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

> [!WARNING]
> 文件系统 IO 的开销通常使此适配器成为*较慢*的选择之一。如果吞吐量至关重要，建议使用内存适配器（[Apcu](./apcu_adapter.md)、[Memcached](./memcached_adapter.md) 和 [Redis](./redis_adapter.md)）或数据库适配器（[Doctrine DBAL](./doctrine_dbal_adapter.md)、[PDO](./pdo_adapter.md)）。

> [!NOTE]
> 此适配器实现了 `Symfony\Component\Cache\PruneableInterface`，通过调用其 `prune()` 方法可以手动修剪过期的缓存项。

## 使用标签

为了使用基于标签的失效，您可以将适配器包装在 `Symfony\Component\Cache\Adapter\TagAwareAdapter` 中，但使用专用的 `Symfony\Component\Cache\Adapter\FilesystemTagAwareAdapter` 通常更有趣。由于标签失效逻辑是使用文件系统上的链接实现的，因此当使用基于标签的失效时，此适配器提供更好的读取性能：

```php
use Symfony\Component\Cache\Adapter\FilesystemTagAwareAdapter;

$cache = new FilesystemTagAwareAdapter();
```

[tmpfs]: https://wiki.archlinux.org/index.php/tmpfs
[ram-disk-solutions]: https://en.wikipedia.org/wiki/List_of_RAM_drive_software
