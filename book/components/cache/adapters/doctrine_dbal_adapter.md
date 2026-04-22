# Doctrine DBAL 缓存适配器

Doctrine DBAL 适配器将缓存项存储在 SQL 数据库的表中。

> [!NOTE]
> 此适配器实现了 `Symfony\Component\Cache\PruneableInterface`，允许通过调用 `prune()` 方法手动修剪过期的缓存条目。

`Symfony\Component\Cache\Adapter\DoctrineDbalAdapter` 需要一个 [Doctrine DBAL Connection][doctrine-dbal-connection] 或 [Doctrine DBAL URL][doctrine-dbal-url] 作为其第一个参数。您可以将命名空间、默认缓存生命周期和选项数组作为其他可选参数传递：

```php
use Symfony\Component\Cache\Adapter\DoctrineDbalAdapter;

$cache = new DoctrineDbalAdapter(

    // Doctrine DBAL 连接或 DBAL URL
    $databaseConnectionOrURL,

    // 作为存储在此缓存中的项的键的前缀的字符串
    $namespace = '',

    // 未定义自己生命周期的缓存项的默认生命周期（以秒为单位），
    // 值为 0 会导致项被无限期存储（即直到数据库表被截断或其行被删除）
    $defaultLifetime = 0,

    // 用于配置数据库表和连接的选项数组
    $options = []
);
```

> [!NOTE]
> 默认情况下，DBAL 连接是延迟加载的；可能需要一些额外的选项来检测数据库引擎和版本，而无需打开连接。

适配器使用针对所连接的数据库服务器优化的 SQL 语法。已知以下数据库服务器兼容：

* MySQL 5.7 及更高版本
* MariaDB 10.2 及更高版本
* Oracle 10g 及更高版本
* SQL Server 2012 及更高版本
* SQLite 3.24 或更高版本
* PostgreSQL 9.5 或更高版本

> [!NOTE]
> Doctrine DBAL 的较新版本可能会提高这些最低版本。如果您的数据库服务器与已安装的 Doctrine DBAL 版本兼容，请查看 [Doctrine DBAL 平台][doctrine-dbal-platforms]手册页。

[doctrine-dbal-connection]: https://github.com/doctrine/dbal/blob/master/src/Connection.php
[doctrine-dbal-url]: https://www.doctrine-project.org/projects/doctrine-dbal/en/current/reference/configuration.html#connecting-using-a-url
[doctrine-dbal-platforms]: https://www.doctrine-project.org/projects/doctrine-dbal/en/current/reference/platforms.html
