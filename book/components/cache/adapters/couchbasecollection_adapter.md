# CouchbaseCollection 缓存适配器

此适配器使用一个（或多个）[Couchbase 服务器][couchbase-server]实例将值存储在内存中。与 [APCu 适配器](./apcu_adapter.md)不同，类似于 [Memcached 适配器](./memcached_adapter.md)，它不限于当前服务器的共享内存；您可以独立于 PHP 环境存储内容。还可以利用服务器集群提供冗余和/或故障转移的能力。

> [!WARNING]
> **要求：**必须安装、激活并运行 [Couchbase PHP 扩展][couchbase-php-extension]以及 [Couchbase 服务器][couchbase-server]才能使用此适配器。此适配器需要 [Couchbase PHP 扩展][couchbase-php-extension]的 `3.0` 或更高版本。

此适配器期望将 [Couchbase Collection][couchbase-collection] 实例作为第一个参数传递。可以选择性地将命名空间和默认缓存生命周期作为第二个和第三个参数传递：

```php
use Symfony\Component\Cache\Adapter\CouchbaseCollectionAdapter;

$cache = new CouchbaseCollectionAdapter(
    // 设置选项并添加服务器实例的客户端对象
    $client,

    // 作为存储在此缓存中的项的键的前缀的字符串
    $namespace,

    // 未定义自己生命周期的缓存项的默认生命周期（以秒为单位），
    // 值为 0 会导致项被无限期存储
    $defaultLifetime
);
```

## 配置连接

`Symfony\Component\Cache\Adapter\CouchbaseCollectionAdapter::createConnection` 辅助方法允许使用[数据源名称（DSN）][dsn]或 DSN 数组创建和配置 [Couchbase Collection][couchbase-collection] 类实例：

```php
use Symfony\Component\Cache\Adapter\CouchbaseCollectionAdapter;

// 传递单个 DSN 字符串以向客户端注册单个服务器
$client = CouchbaseCollectionAdapter::createConnection(
    'couchbase://localhost'
    // DSN 可以包含配置选项（作为查询字符串传递）：
    // 'couchbase://localhost:11210?operationTimeout=10'
    // 'couchbase://localhost:11210?operationTimeout=10&configTimout=20'
);

// 传递 DSN 字符串数组以向客户端注册多个服务器
$client = CouchbaseCollectionAdapter::createConnection([
    'couchbase://10.0.0.100',
    'couchbase://10.0.0.101',
    'couchbase://10.0.0.102',
    // 等等...
]);

// 单个 DSN 可以使用以下语法定义多个服务器：
// host[hostname-or-IP:port]（其中 port 是可选的）。套接字必须包含尾随 ':'
$client = CouchbaseCollectionAdapter::createConnection(
    'couchbase:?host[localhost]&host[localhost:12345]'
);
```

## 配置选项

`Symfony\Component\Cache\Adapter\CouchbaseCollectionAdapter::createConnection` 辅助方法还接受选项数组作为其第二个参数。预期格式是表示选项名称及其各自值的 `key => value` 对的关联数组：

```php
use Symfony\Component\Cache\Adapter\CouchbaseCollectionAdapter;

$client = CouchbaseCollectionAdapter::createConnection(
    // DSN 字符串或 DSN 字符串数组
    [],

    // 配置选项的关联数组
    [
        'username' => 'xxxxxx',
        'password' => 'yyyyyy',
        'configTimeout' => '100',
    ]
);
```

### 可用选项

`username`（类型：`string`）  
连接 `CouchbaseCluster` 的用户名。

`password`（类型：`string`）  
连接 `CouchbaseCluster` 的密码。

`operationTimeout`（类型：`int`，默认值：`2500000`）  
操作超时（以微秒为单位）是库在使用失败状态调用其回调之前等待操作接收响应的最长时间。

`configTimeout`（类型：`int`，默认值：`5000000`）  
客户端等待获取初始配置的时间（以微秒为单位）。

`configNodeTimeout`（类型：`int`，默认值：`2000000`）  
每个节点的配置超时（以微秒为单位）。

`viewTimeout`（类型：`int`，默认值：`75000000`）  
对 Couchbase Views API 的 HTTP 请求的 I/O 超时（以微秒为单位）。

`httpTimeout`（类型：`int`，默认值：`75000000`）  
HTTP 查询（管理 API）的 I/O 超时（以微秒为单位）。

`configDelay`（类型：`int`，默认值：`10000`）  
配置刷新限制  
修改配置错误阈值被强制设置为其最大数量以强制配置刷新之前的时间（以微秒为单位）。

`htconfigIdleTimeout`（类型：`int`，默认值：`4294967295`）  
HTTP 引导的空闲/持久性（以微秒为单位）。

`durabilityInterval`（类型：`int`，默认值：`100000`）  
客户端在对给定服务器的重复探测之间等待的时间（以微秒为单位）。

`durabilityTimeout`（类型：`int`，默认值：`5000000`）  
客户端在被视为不满足持久性要求之前，向给定键的 vBucket 主服务器和副本发送重复探测所花费的时间（以微秒为单位）。

> [!TIP]
> 有关可用选项的更多信息，请参阅 [Couchbase Collection][couchbase-collection] 扩展的[预定义常量][predefined-constants]文档。

[couchbase-php-extension]: https://docs.couchbase.com/sdk-api/couchbase-php-client/namespaces/couchbase.html
[predefined-constants]: https://docs.couchbase.com/sdk-api/couchbase-php-client/classes/Couchbase-Bucket.html
[couchbase-server]: https://couchbase.com/
[couchbase-collection]: https://docs.couchbase.com/sdk-api/couchbase-php-client/classes/Couchbase-Collection.html
[dsn]: https://en.wikipedia.org/wiki/Data_source_name
