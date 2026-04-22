# PDO 缓存适配器

PDO 适配器将缓存项存储在 SQL 数据库的表中。

> [!NOTE]
> 此适配器实现了 `Symfony\Component\Cache\PruneableInterface`，允许通过调用 `prune()` 方法手动修剪过期的缓存条目。

`Symfony\Component\Cache\Adapter\PdoAdapter` 需要一个 PDO 或 [DSN][dsn-drivers] 作为其第一个参数。您可以将命名空间、默认缓存生命周期和选项数组作为其他可选参数传递：

```php
use Symfony\Component\Cache\Adapter\PdoAdapter;

$cache = new PdoAdapter(

    // PDO 连接或用于通过 PDO 延迟连接的 DSN
    $databaseConnectionOrDSN,

    // 作为存储在此缓存中的项的键的前缀的字符串
    $namespace = '',

    // 未定义自己生命周期的缓存项的默认生命周期（以秒为单位），
    // 值为 0 会导致项被无限期存储（即直到数据库表被截断或其行被删除）
    $defaultLifetime = 0,

    // 用于配置数据库表和连接的选项数组
    $options = []
);
```

存储值的表在第一次调用 `Symfony\Component\Cache\Adapter\PdoAdapter::save` 方法时自动创建。您还可以通过在代码中调用 `Symfony\Component\Cache\Adapter\PdoAdapter::createTable` 方法来显式创建此表。

> [!TIP]
> 当传递[数据源名称（DSN）][data-source-name]字符串（而不是数据库连接类实例）时，连接将在需要时延迟加载。

[dsn-drivers]: https://php.net/manual/pdo.drivers.php
[data-source-name]: https://en.wikipedia.org/wiki/Data_source_name
