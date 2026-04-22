# Redis 缓存适配器

> [!NOTE]
> 本文介绍如何在任何 PHP 应用中将 Redis 适配器作为独立组件使用时配置它。如果您在 Symfony 应用中使用它，请阅读 Symfony 缓存配置文章。

此适配器使用一个（或多个）[Redis 服务器][redis-server]或 [Valkey][valkey] 服务器实例将值存储在内存中。

与 [APCu 适配器](./apcu_adapter.md)不同，类似于 [Memcached 适配器](./memcached_adapter.md)，它不限于当前服务器的共享内存；您可以独立于 PHP 环境存储内容。还可以利用服务器集群提供冗余和/或故障转移的能力。

> [!WARNING]
> **要求：**至少必须安装并运行一个 [Redis 服务器][redis-server]才能使用此适配器。此外，此适配器需要实现 `\Redis`、`\RedisArray`、`RedisCluster`、`\Relay\Relay`、`\Relay\Cluster` 或 `\Predis` 的兼容扩展或库。

此适配器期望将 [Redis][redis]、[RedisArray][redis-array]、[RedisCluster][redis-cluster]、[Relay][relay]、[RelayCluster][relay-cluster] 或 [Predis][predis] 实例作为第一个参数传递。可以选择性地将命名空间和默认缓存生命周期作为第二个和第三个参数传递：

```php
use Symfony\Component\Cache\Adapter\RedisAdapter;

$cache = new RedisAdapter(

    // 存储到 Redis 系统的有效连接的对象
    \Redis $redisConnection,

    // 作为存储在此缓存中的项的键的前缀的字符串
    $namespace = '',

    // 未定义自己生命周期的缓存项的默认生命周期（以秒为单位），
    // 值为 0 会导致项被无限期存储（即直到调用 RedisAdapter::clear() 或清除服务器）
    $defaultLifetime = 0,

    // $marshaller（可选）MarshallerInterface 的实例，用于控制缓存项的序列化和反序列化。
    // 默认情况下，使用原生 PHP 序列化。
    // 这对于压缩数据、应用自定义序列化逻辑或优化缓存项的大小和性能很有用
    ?MarshallerInterface $marshaller = null
);
```

## 配置连接

`Symfony\Component\Cache\Traits\RedisTrait::createConnection` 辅助方法允许使用[数据源名称（DSN）][data-source-name]创建和配置 Redis 客户端类实例：

```php
use Symfony\Component\Cache\Adapter\RedisAdapter;

// 传递单个 DSN 字符串以向客户端注册单个服务器
$client = RedisAdapter::createConnection(
    'redis://localhost'
);
```

DSN 可以指定 IP/主机（和可选端口）或套接字路径，以及用户名、密码和数据库索引。要为连接启用 TLS，必须将方案 `redis` 替换为 `rediss`（第二个 `s` 表示"安全"）。

> [!NOTE]
> 此适配器的[数据源名称（DSN）][data-source-name]必须使用以下格式之一。
> 
> ```text
> redis[s]://[pass@][ip|host|socket[:port]][/db-index]
> ```
> 
> ```text
> redis[s]:[[user]:pass@]?[ip|host|socket[:port]][&params]
> ```
> 
> 占位符 `[user]`、`[:port]`、`[/db-index]` 和 `[&params]` 的值是可选的。

以下是显示可用值组合的有效 DSN 的常见示例：

```php
use Symfony\Component\Cache\Adapter\RedisAdapter;

// 主机 "my.server.com" 和端口 "6379"
RedisAdapter::createConnection('redis://my.server.com:6379');

// 主机 "my.server.com" 和端口 "6379" 和数据库索引 "20"
RedisAdapter::createConnection('redis://my.server.com:6379/20');

// 主机 "localhost"，auth "abcdef" 和超时 5 秒
RedisAdapter::createConnection('redis://abcdef@localhost?timeout=5');

// 套接字 "/var/run/redis.sock" 和 auth "bad-pass"
RedisAdapter::createConnection('redis://bad-pass@/var/run/redis.sock');

// 主机 "redis1"（docker 容器），使用替代 DSN 语法并选择数据库索引 "3"
RedisAdapter::createConnection('redis:?host[redis1:6379]&dbindex=3');

// 使用替代 DSN 语法提供凭据
RedisAdapter::createConnection('redis:myusername:verysecurepassword@?host[redis1:6379]&dbindex=3');

// 使用 auth 参数提供凭据
RedisAdapter::createConnection('redis://host?auth[]=myusername&auth[]=verysecurepassword');

// 单个 DSN 也可以定义多个服务器
RedisAdapter::createConnection(
    'redis:?host[localhost]&host[localhost:6379]&host[/var/run/redis.sock:]&auth=my-password&redis_cluster=1'
);
```

[Redis Sentinel][redis-sentinel] 为 Redis 提供高可用性，在使用 PHP Redis 扩展 v5.2+ 或 Predis 库时也受支持。使用 `redis_sentinel` 参数设置服务组的名称：

```php
RedisAdapter::createConnection(
    'redis:?host[redis1:26379]&host[redis2:26379]&host[redis3:26379]&redis_sentinel=mymaster'
);

// 提供凭据
RedisAdapter::createConnection(
    'redis:default:verysecurepassword@?host[redis1:26379]&host[redis2:26379]&host[redis3:26379]&redis_sentinel=mymaster'
);

// 提供凭据并选择数据库索引 "3"
RedisAdapter::createConnection(
    'redis:default:verysecurepassword@?host[redis1:26379]&host[redis2:26379]&host[redis3:26379]&redis_sentinel=mymaster&dbindex=3'
);
```

当 Redis 主服务器和哨兵使用不同的凭据时，在 DSN userinfo 中提供主服务器凭据，并通过 `auth` 查询参数或选项提供哨兵凭据：

```php
// 主服务器密码在 userinfo 中，哨兵密码在 auth 查询参数中
RedisAdapter::createConnection(
    'redis://master-password@?host[redis1:26379]&host[redis2:26379]&redis_sentinel=mymaster&auth=sentinel-password'
);

// 主服务器用户+密码在 userinfo 中，哨兵用户+密码在 auth 查询参数中（ACL）
RedisAdapter::createConnection(
    'redis://master-user:master-pass@?host[redis1:26379]&host[redis2:26379]&redis_sentinel=mymaster&auth[]=sentinel-user&auth[]=sentinel-pass'
);

// 或使用 auth 选项
RedisAdapter::createConnection(
    'redis://master-password@?host[redis1:26379]&host[redis2:26379]&redis_sentinel=mymaster',
    ['auth' => ['sentinel-user', 'sentinel-pass']]
);
```

> [!NOTE]
> 有关可以作为 DSN 参数传递的更多选项，请参阅 `Symfony\Component\Cache\Traits\RedisTrait`。

## 配置选项

`Symfony\Component\Cache\Adapter\RedisAdapter::createConnection` 辅助方法还接受选项数组作为其第二个参数。预期格式是表示选项名称及其各自值的 `key => value` 对的关联数组：

```php
use Symfony\Component\Cache\Adapter\RedisAdapter;

$client = RedisAdapter::createConnection(

    // 提供字符串 dsn
    'redis://localhost:6379',

    // 配置选项的关联数组
    [
        'class' => null,
        'auth' => null,
        'persistent' => 0,
        'persistent_id' => null,
        'timeout' => 30,
        'read_timeout' => 0,
        'retry_interval' => 0,
        'tcp_keepalive' => 0,
        'lazy' => null,
        'redis_cluster' => false,
        'redis_sentinel' => null,
        'dbindex' => 0,
        'failover' => 'none',
        'ssl' => null,
    ]

);
```

### 可用选项

`class`（类型：`string`，默认值：`null`）  
指定要返回的连接库，可以是 `\Redis`、`\Relay\Relay` 或 `\Predis\Client`。如果未指定，则按以下顺序回退，取决于哪个首先可用：`\Redis`、`\Relay\Relay`、`\Predis\Client`。如果在检索主服务器信息时遇到问题，请将其显式设置为 `\Predis\Client` 以用于 Sentinel。

`auth`（类型：`string|string[]`，默认值：`null`）  
指定 Redis 连接的身份验证凭据。对于仅密码身份验证使用字符串，或对于基于 ACL 的身份验证使用包含两个元素 `[username, password]` 的数组。当为 `null` 时，凭据从 DSN 中提取（例如 `redis://user:password@host`）。

当使用 Redis Sentinel 时，此选项用于**哨兵**凭据。**主服务器**凭据应通过 DSN userinfo 提供（例如 `redis://master-pass@host`）。

> [!NOTE]
> 使用 [Predis][predis] 时，此选项将被忽略。请通过 DSN 提供凭据。

`persistent`（类型：`int`，默认值：`0`）  
启用或禁用持久连接的使用。值为 `0` 禁用持久连接，值为 `1` 启用它们。

`persistent_id`（类型：`string|null`，默认值：`null`）  
指定用于持久连接的持久 ID 字符串。

`timeout`（类型：`int`，默认值：`30`）  
指定在连接尝试超时之前用于连接到 Redis 服务器的时间（以秒为单位）。

`read_timeout`（类型：`int`，默认值：`0`）  
指定在操作超时之前在底层网络资源上执行读取操作时使用的时间（以秒为单位）。

`retry_interval`（类型：`int`，默认值：`0`）  
指定在客户端失去与服务器的连接时重新连接尝试之间的延迟（以毫秒为单位）。

`tcp_keepalive`（类型：`int`，默认值：`0`）  
指定连接的 [TCP-keepalive][tcp-keepalive] 超时时间（以秒为单位）。这需要 phpredis v4 或更高版本以及启用了 TCP-keepalive 的服务器。

`lazy`（类型：`bool`，默认值：`null`）  
启用或禁用到后端的延迟连接。在将其作为独立组件使用时，默认为 `false`，在 Symfony 应用中使用时默认为 `true`。

`redis_cluster`（类型：`bool`，默认值：`false`）  
启用或禁用 redis 集群。只要传递的实际值通过松散比较检查，它就是无关紧要的：`redis_cluster=1` 就足够了。

`redis_sentinel`（类型：`string`，默认值：`null`）  
指定连接到哨兵的主服务器名称。

`sentinel_master`（类型：`string`，默认值：`null`）  
`redis_sentinel` 选项的别名。

`dbindex`（类型：`int`，默认值：`0`）  
指定要选择的数据库索引。

`failover`（类型：`string`，默认值：`none`）  
指定集群实现的故障转移。对于 `\RedisCluster`，有效选项为 `none`（默认）、`error`、`distribute` 或 `slaves`。对于 `\Predis\ClientInterface`，有效选项为 `slaves` 或 `distribute`。

`ssl`（类型：`array`，默认值：`null`）  
SSL 上下文选项。有关更多信息，请参阅 [php.net/context.ssl][php-net-context-ssl]。

`relay_cluster_context`（类型：`array`，默认值：`[]`）  
定义特定于 `\Relay\Cluster` 的配置选项。例如，在本地环境中使用自签名证书进行测试：

```php
$options = [
    // ...
    'relay_cluster_context' => [
        // ...
        'stream' => [
            'verify_peer' => false,
            'verify_peer_name' => false,
            'allow_self_signed' => true,
            'local_cert' => '/valkey.crt',
            'local_pk' => '/valkey.key',
            'cafile' => '/valkey.crt',
        ],
    ],
];
```

> [!NOTE]
> 使用 [Predis][predis] 库时，还有一些额外的 Predis 特定选项可用。有关更多信息，请参阅 [Predis 连接参数][predis-connection-parameters]文档。

## 配置 Redis

将 Redis 用作缓存时，应配置 `maxmemory` 和 `maxmemory-policy` 设置。通过设置 `maxmemory`，您限制了 Redis 允许消耗的内存量。如果数量太低，Redis 将丢弃仍然有用的条目，您从缓存中获得的好处就会减少。将 `maxmemory-policy` 设置为 `allkeys-lru` 告诉 Redis 当内存不足时可以丢弃数据，并首先丢弃最旧的条目（最近最少使用）。如果您不允许 Redis 丢弃条目，当没有内存可用时尝试添加数据时，它将返回错误。示例设置可能如下所示：

```ini
maxmemory 100mb
maxmemory-policy allkeys-lru
```

## 使用标签

为了使用基于标签的失效，您可以将适配器包装在 `Symfony\Component\Cache\Adapter\TagAwareAdapter` 中。但是，当使用 Redis 作为后端时，使用专用的 `Symfony\Component\Cache\Adapter\RedisTagAwareAdapter` 通常更有趣。由于标签失效逻辑在 Redis 本身中实现，因此当使用基于标签的失效时，此适配器提供更好的性能：

```php
use Symfony\Component\Cache\Adapter\RedisAdapter;
use Symfony\Component\Cache\Adapter\RedisTagAwareAdapter;

$client = RedisAdapter::createConnection('redis://localhost');
$cache = new RedisTagAwareAdapter($client);
```

> [!NOTE]
> 使用 RedisTagAwareAdapter 时，为了维护标签和缓存项之间的关系，您必须在 Redis `maxmemory-policy` 驱逐策略中使用 `noeviction` 或 `volatile-*`。

在官方 [Redis LRU 缓存文档][redis-lru-cache-documentation]中阅读有关此主题的更多信息。

## 使用 Marshaller

### TagAwareMarshaller 用于基于标签的缓存

优化基于标签的检索的缓存，允许高效管理相关项：

```php
$marshaller = new TagAwareMarshaller();

$cache = new RedisAdapter($redis, 'tagged_namespace', 3600, $marshaller);

$item = $cache->getItem('tagged_key');
$item->set(['value' => 'some_data', 'tags' => ['tag1', 'tag2']]);
$cache->save($item);
```

### SodiumMarshaller 用于加密缓存

使用 Sodium 加密缓存数据以增强安全性：

```php
$encryptionKeys = [sodium_crypto_box_keypair()];
$marshaller = new SodiumMarshaller($encryptionKeys);

$cache = new RedisAdapter($redis, 'secure_namespace', 3600, $marshaller);

$item = $cache->getItem('secure_key');
$item->set('confidential_data');
$cache->save($item);
```

### DefaultMarshaller 使用 igbinary 序列化

在可用时使用 `igbinary` 进行更快、更高效的序列化：

```php
$marshaller = new DefaultMarshaller(true);

$cache = new RedisAdapter($redis, 'optimized_namespace', 3600, $marshaller);

$item = $cache->getItem('optimized_key');
$item->set(['data' => 'optimized_data']);
$cache->save($item);
```

### DefaultMarshaller 在失败时抛出异常

如果序列化失败，抛出异常，便于错误处理：

```php
$marshaller = new DefaultMarshaller(false, true);

$cache = new RedisAdapter($redis, 'error_namespace', 3600, $marshaller);

try {
    $item = $cache->getItem('error_key');
    $item->set('data');
    $cache->save($item);
} catch (\ValueError $e) {
    echo 'Serialization failed: '.$e->getMessage();
}
```

### SodiumMarshaller 使用密钥轮换

支持密钥轮换，确保使用新旧密钥安全解密：

```php
$keys = [sodium_crypto_box_keypair(), sodium_crypto_box_keypair()];
$marshaller = new SodiumMarshaller($keys);

$cache = new RedisAdapter($redis, 'rotated_namespace', 3600, $marshaller);

$item = $cache->getItem('rotated_key');
$item->set('data_to_encrypt');
$cache->save($item);
```

[data-source-name]: https://en.wikipedia.org/wiki/Data_source_name
[redis-server]: https://redis.io/
[valkey]: https://valkey.io/
[redis]: https://github.com/phpredis/phpredis
[redis-array]: https://github.com/phpredis/phpredis/blob/develop/arrays.md
[redis-cluster]: https://github.com/phpredis/phpredis/blob/develop/cluster.md
[relay]: https://relay.so/
[relay-cluster]: https://relay.so/docs/1.x/connections#cluster
[predis]: https://packagist.org/packages/predis/predis
[predis-connection-parameters]: https://github.com/nrk/predis/wiki/Connection-Parameters#list-of-connection-parameters
[tcp-keepalive]: https://redis.io/topics/clients#tcp-keepalive
[redis-sentinel]: https://redis.io/topics/sentinel
[redis-lru-cache-documentation]: https://redis.io/topics/lru-cache
[php-net-context-ssl]: https://php.net/context.ssl
