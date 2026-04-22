# Memcached 缓存适配器

此适配器使用一个（或多个）[Memcached 服务器][memcached-server]实例将值存储在内存中。与 [APCu 适配器](./apcu_adapter.md)不同，类似于 [Redis 适配器](./redis_adapter.md)，它不限于当前服务器的共享内存；您可以独立于 PHP 环境存储内容。还可以利用服务器集群提供冗余和/或故障转移的能力。

> [!WARNING]
> **要求：**必须安装、激活并运行 [Memcached PHP 扩展][memcached-php-extension]以及 [Memcached 服务器][memcached-server]才能使用此适配器。此适配器需要 [Memcached PHP 扩展][memcached-php-extension]的 `2.2` 或更高版本。

此适配器期望将 [Memcached][memcached] 实例作为第一个参数传递。可以选择性地将命名空间和默认缓存生命周期作为第二个和第三个参数传递：

```php
use Symfony\Component\Cache\Adapter\MemcachedAdapter;

$cache = new MemcachedAdapter(
    // 设置选项并添加服务器实例的客户端对象
    \Memcached $client,

    // 作为存储在此缓存中的项的键的前缀的字符串
    $namespace = '',

    // 未定义自己生命周期的缓存项的默认生命周期（以秒为单位），
    // 值为 0 会导致项被无限期存储（即直到调用 MemcachedAdapter::clear() 或服务器重启）
    $defaultLifetime = 0
);
```

## 配置连接

`Symfony\Component\Cache\Adapter\MemcachedAdapter::createConnection` 辅助方法允许使用[数据源名称（DSN）][data-source-name]或 DSN 数组创建和配置 [Memcached][memcached] 类实例：

```php
use Symfony\Component\Cache\Adapter\MemcachedAdapter;

// 传递单个 DSN 字符串以向客户端注册单个服务器
$client = MemcachedAdapter::createConnection(
    'memcached://localhost'
    // DSN 可以包含配置选项（作为查询字符串传递）：
    // 'memcached://localhost:11222?retry_timeout=10'
    // 'memcached://localhost:11222?socket_recv_size=1&socket_send_size=2'
);

// 传递 DSN 字符串数组以向客户端注册多个服务器
$client = MemcachedAdapter::createConnection([
    'memcached://10.0.0.100',
    'memcached://10.0.0.101',
    'memcached://10.0.0.102',
    // 等等...
]);

// 单个 DSN 可以使用以下语法定义多个服务器：
// host[hostname-or-IP:port]（其中 port 是可选的）。套接字必须包含尾随 ':'
$client = MemcachedAdapter::createConnection(
    'memcached:?host[localhost]&host[localhost:12345]&host[/some/memcached.sock:]=3'
);
```

此适配器的[数据源名称（DSN）][data-source-name]必须使用以下格式：

```text
memcached://[user:pass@][ip|host|socket[:port]][?weight=int]
```

DSN 必须包含 IP/主机（和可选端口）或套接字路径、可选的用户名和密码（用于 SASL 身份验证；这要求 memcached 扩展使用 `--enable-memcached-sasl` 编译）以及可选的权重（用于确定集群中服务器的优先级；其值是介于 `0` 和 `100` 之间的整数，默认为 `null`；值越高表示优先级越高）。

以下是显示可用值组合的有效 DSN 的常见示例：

```php
use Symfony\Component\Cache\Adapter\MemcachedAdapter;

$client = MemcachedAdapter::createConnection([
    // 主机名 + 端口
    'memcached://my.server.com:11211'

    // 主机名无端口 + SASL 用户名和密码
    'memcached://rmf:abcdef@localhost'

    // IP 地址而不是主机名 + 权重
    'memcached://127.0.0.1?weight=50'

    // 套接字而不是主机名/IP + SASL 用户名和密码
    'memcached://janesmith:mypassword@/var/run/memcached.sock'

    // 套接字而不是主机名/IP + 权重
    'memcached:///var/run/memcached.sock?weight=20'
]);
```

## 配置选项

`Symfony\Component\Cache\Adapter\MemcachedAdapter::createConnection` 辅助方法还接受选项数组作为其第二个参数。预期格式是表示选项名称及其各自值的 `key => value` 对的关联数组：

```php
use Symfony\Component\Cache\Adapter\MemcachedAdapter;

$client = MemcachedAdapter::createConnection(
    // DSN 字符串或 DSN 字符串数组
    [],

    // 配置选项的关联数组
    [
        'libketama_compatible' => true,
        'serializer' => 'igbinary',
    ]
);
```

### 可用选项

`auto_eject_hosts`（类型：`bool`，默认值：`false`）  
启用或禁用通过自动弹出超过配置的 `server_failure_limit` 的主机来持续、自动地重新平衡集群。

`buffer_writes`（类型：`bool`，默认值：`false`）  
启用或禁用缓冲输入/输出操作，导致存储命令被缓冲而不是立即发送到远程服务器。任何检索数据、退出连接或关闭连接的操作都将导致缓冲区被提交。

`connect_timeout`（类型：`int`，默认值：`1000`）  
指定启用 `no_block` 选项时套接字连接操作的超时时间（以毫秒为单位）。

有效的选项值包括*任何正整数*。

`distribution`（类型：`string`，默认值：`consistent`）  
指定服务器之间的项键分布方法。一致性哈希提供更好的分布，并允许以最小的缓存损失将服务器添加到集群。

有效的选项值包括 `modula`、`consistent` 和 `virtual_bucket`。

`hash`（类型：`string`，默认值：`md5`）  
指定用于项键的哈希算法。每种哈希算法都有其优点和缺点。建议使用默认值以与其他客户端兼容。

有效的选项值包括 `default`、`md5`、`crc`、`fnv1_64`、`fnv1a_64`、`fnv1_32`、`fnv1a_32`、`hsieh` 和 `murmur`。

`libketama_compatible`（类型：`bool`，默认值：`true`）  
启用或禁用"libketama"兼容行为，使其他基于 libketama 的客户端能够透明地访问客户端实例存储的键（如 Python 和 Ruby）。启用此选项会将 `hash` 选项设置为 `md5`，将 `distribution` 选项设置为 `consistent`。

`no_block`（类型：`bool`，默认值：`true`）  
启用或禁用异步输入和输出操作。这是存储函数可用的最快传输选项。

`number_of_replicas`（类型：`int`，默认值：`0`）  
指定应为每个项存储的副本数（在不同的服务器上）。这不会将某些 memcached 服务器专用于存储副本，而是将副本与所有其他对象一起存储（在接下来注册的"n"个服务器上）。

有效的选项值包括*任何正整数*。

`prefix_key`（类型：`string`，默认值：空字符串）  
指定附加到您的键的"域"（或"命名空间"）。它不能超过 128 个字符，并会减少最大键大小。

有效的选项值包括*任何字母数字字符串*。

`poll_timeout`（类型：`int`，默认值：`1000`）  
指定在套接字轮询操作期间超时之前的时间量（以秒为单位）。

有效的选项值包括*任何正整数*。

`randomize_replica_read`（类型：`bool`，默认值：`false`）  
启用或禁用副本读取起始点的随机化。通常，读取从主服务器完成，如果未命中，则从"主服务器+1"读取，然后是"主服务器+2"，一直到"n"个副本。此选项将副本读取设置为在所有可用服务器之间随机化；它允许将读取负载分配到多个服务器，但代价是更多的写入流量。

`recv_timeout`（类型：`int`，默认值：`0`）  
指定在传出套接字（读取）操作期间超时之前的时间量（以微秒为单位）。当未启用 `no_block` 选项时，这将允许您在读取数据时仍然有超时。

有效的选项值包括 `0` 或*任何正整数*。

`retry_timeout`（类型：`int`，默认值：`0`）  
指定在超时并重试连接尝试之前的时间量（以秒为单位）。

有效的选项值包括*任何正整数*。

`send_timeout`（类型：`int`，默认值：`0`）  
指定在传入套接字（发送）操作期间超时之前的时间量（以微秒为单位）。当未启用 `no_block` 选项时，这将允许您在发送数据时仍然有超时。

有效的选项值包括 `0` 或*任何正整数*。

`serializer`（类型：`string`，默认值：`php`）  
指定用于序列化非标量值的序列化器。`igbinary` 选项要求启用 igbinary PHP 扩展，以及 memcached 扩展已编译支持它。

有效的选项值包括 `php` 和 `igbinary`。

`server_failure_limit`（类型：`int`，默认值：`0`）  
指定在将服务器标记为"死亡"之前的服务器连接尝试失败限制。除非启用 `auto_eject_hosts`，否则服务器将保留在服务器池中。

有效的选项值包括*任何正整数*。

`socket_recv_size`（类型：`int`）  
指定在传入（接收）套接字连接数据的上下文中的最大缓冲区大小（以字节为单位）。

有效的选项值包括*任何正整数*，默认值*因平台和内核配置而异*。

`socket_send_size`（类型：`int`）  
指定在传出（发送）套接字连接数据的上下文中的最大缓冲区大小（以字节为单位）。

有效的选项值包括*任何正整数*，默认值*因平台和内核配置而异*。

`tcp_keepalive`（类型：`bool`，默认值：`false`）  
启用或禁用"[keep-alive][keep-alive]"[传输控制协议（TCP）][tcp]功能，这是一种通过在空闲期后向网络对等方发送探测并根据响应（或缺少响应）关闭或持久化套接字来帮助确定另一端是否已停止响应的功能。

`tcp_nodelay`（类型：`bool`，默认值：`false`）  
启用或禁用"[no-delay][no-delay]"（Nagle 算法）[传输控制协议（TCP）][tcp]算法，这是一种旨在通过减少 TCP 头的开销来提高网络效率的机制，方法是组合多个小的传出消息并一次性发送它们。

`use_udp`（类型：`bool`，默认值：`false`）  
启用或禁用使用[用户数据报协议（UDP）][udp]模式（而不是[传输控制协议（TCP）][tcp]模式），其中所有操作都以"即发即弃"方式执行；一旦客户端执行了操作，就不会尝试确保操作已被接收或执行。

> [!WARNING]
> 并非所有库操作都在此模式下进行了测试。不允许混合使用 TCP 和 UDP 服务器。

`verify_key`（类型：`bool`，默认值：`false`）  
启用或禁用对所有使用的键的测试和验证，以确保它们有效并符合所使用协议的设计。

> [!TIP]
> 有关可用选项的更多信息，请参阅 [Memcached][memcached] 扩展的[预定义常量][predefined-constants]文档。

[tcp]: https://en.wikipedia.org/wiki/Transmission_Control_Protocol
[udp]: https://en.wikipedia.org/wiki/User_Datagram_Protocol
[no-delay]: https://en.wikipedia.org/wiki/TCP_NODELAY
[keep-alive]: https://en.wikipedia.org/wiki/Keepalive
[memcached-php-extension]: https://www.php.net/manual/en/book.memcached.php
[predefined-constants]: https://www.php.net/manual/en/memcached.constants.php
[memcached-server]: https://memcached.org/
[memcached]: https://www.php.net/manual/en/class.memcached.php
[data-source-name]: https://en.wikipedia.org/wiki/Data_source_name
