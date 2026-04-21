# HTTP 客户端

## 安装

HttpClient 组件是一个低级别的 HTTP 客户端，支持 PHP 流包装器和 cURL。它提供了使用 API 的实用工具，并支持同步和异步操作。你可以通过以下命令安装：

```terminal
$ composer require symfony/http-client
```

---

## 基本用法

使用 `HttpClient` 类来发出请求。在 Symfony 框架中，此类可作为 `http_client` 服务使用。当对 `HttpClientInterface` 进行类型提示时，此服务将自动[自动装配](service_container/autowiring.md)：

**Symfony 应用：**

```php
use Symfony\Contracts\HttpClient\HttpClientInterface;

class SymfonyDocs
{
    public function __construct(
        private HttpClientInterface $client,
    ) {
    }

    public function fetchGitHubInformation(): array
    {
        $response = $this->client->request(
            'GET',
            'https://api.github.com/repos/symfony/symfony-docs'
        );

        $statusCode = $response->getStatusCode();
        // $statusCode = 200
        $contentType = $response->getHeaders()['content-type'][0];
        // $contentType = 'application/json'
        $content = $response->getContent();
        // $content = '{"id":521583, "name":"symfony-docs", ...}'
        $content = $response->toArray();
        // $content = ['id' => 521583, 'name' => 'symfony-docs', ...]

        return $content;
    }
}
```

**独立 PHP 应用：**

```php
use Symfony\Component\HttpClient\HttpClient;

$client = HttpClient::create();
$response = $client->request(
    'GET',
    'https://api.github.com/repos/symfony/symfony-docs'
);

$statusCode = $response->getStatusCode();
$contentType = $response->getHeaders()['content-type'][0];
$content = $response->getContent();
$content = $response->toArray();
```

---

## 配置

HTTP 客户端包含许多选项，你可能需要这些选项来完全控制执行请求的方式，包括 DNS 预解析、SSL 参数、公钥固定等。

使用 `default_options` 选项配置全局选项：

```yaml
# config/packages/framework.yaml
framework:
    http_client:
        default_options:
            max_redirects: 7
```

你也可以使用 `withOptions()` 方法获取具有新默认选项的客户端新实例：

```php
$this->client = $client->withOptions([
    'base_uri' => 'https://...',
    'headers' => ['header-name' => 'header-value'],
    'extra' => ['my-key' => 'my-value'],
]);
```

### 范围客户端

当某些 HTTP 客户端选项取决于请求的 URL 时（例如，向 GitHub API 发出请求时必须设置某些头，但不适用于其他主机），该组件提供范围客户端（使用 `ScopingHttpClient`）来根据请求的 URL 自动配置 HTTP 客户端：

```yaml
# config/packages/framework.yaml
framework:
    http_client:
        scoped_clients:
            # 只有匹配范围的请求才会使用这些选项
            github.client:
                scope: 'https://api\.github\.com'
                headers:
                    Accept: 'application/vnd.github.v3+json'
                    Authorization: 'token %env(GITHUB_API_TOKEN)%'
                # ...

            # 使用 base_uri，相对 URL（例如 request("GET", "/repos/symfony/symfony-docs")）
            # 将默认使用这些选项
            github.client:
                base_uri: 'https://api.github.com'
                headers:
                    Accept: 'application/vnd.github.v3+json'
                    Authorization: 'token %env(GITHUB_API_TOKEN)%'
                # ...
```

---

## 发出请求

HTTP 客户端提供了一个单一的 `request()` 方法来执行所有类型的 HTTP 请求：

```php
$response = $client->request('GET', 'https://...');
$response = $client->request('POST', 'https://...');
$response = $client->request('PUT', 'https://...');
// ...

// 你可以使用第三个参数添加请求选项（或覆盖全局选项）
$response = $client->request('GET', 'https://...', [
    'headers' => [
        'Accept' => 'application/json',
    ],
]);
```

Symfony 的 HTTP 客户端默认是异步的。当你调用 `request()` 时，HTTP 请求立即开始，但该方法不等待响应就返回。只有在实际需要响应数据时，你的代码才会阻塞：

```php
// 请求开始，但继续执行而不等待
$response = $client->request('GET', 'http://releases.ubuntu.com/18.04.2/ubuntu-18.04.2-desktop-amd64.iso');

// 这会阻塞直到收到响应头
$contentType = $response->getHeaders()['content-type'][0];

// 这会阻塞直到收到完整的响应体
$content = $response->getContent();
```

### 认证

HTTP 客户端支持不同的认证机制：

```yaml
# config/packages/framework.yaml
framework:
    http_client:
        scoped_clients:
            example_api:
                base_uri: 'https://example.com/'

                # HTTP Basic 认证
                auth_basic: 'the-username:the-password'

                # HTTP Bearer 认证（也称为令牌认证）
                auth_bearer: the-bearer-token

                # Microsoft NTLM 认证
                auth_ntlm: 'the-username:the-password'
```

### 查询字符串参数

你可以将它们手动附加到请求的 URL，或通过 `query` 选项将它们定义为关联数组：

```php
// 向 https://httpbin.org/get?token=...&name=... 发出 HTTP GET 请求
$response = $client->request('GET', 'https://httpbin.org/get', [
    // 这些值在包含到 URL 之前会自动编码
    'query' => [
        'token' => '...',
        'name' => '...',
    ],
]);
```

### 请求头

使用 `headers` 选项定义添加到所有请求的默认头：

```yaml
# config/packages/framework.yaml
framework:
    http_client:
        default_options:
            headers:
                'User-Agent': 'My Fancy App'
```

### 上传数据

此组件提供了多种使用 `body` 选项上传数据的方法：

```php
$response = $client->request('POST', 'https://...', [
    // 使用普通字符串定义数据
    'body' => 'raw data',

    // 使用参数数组定义数据
    'body' => ['parameter1' => 'value1', '...'],

    // 使用闭包生成上传数据
    'body' => function (int $size): string {
        // ...
    },

    // 使用资源从中获取数据
    'body' => fopen('/path/to/file', 'r'),
]);
```

> **提示**
>
> 上传 JSON 负载时，使用 `json` 选项代替 `body`。给定的内容将自动进行 JSON 编码，请求也会自动添加 `Content-Type: application/json`：
> ```php
> $response = $client->request('POST', 'https://...', [
>     'json' => ['param1' => 'value1', '...'],
> ]);
> ```

### Cookie

此组件提供的 HTTP 客户端是无状态的，但处理 cookie 需要状态存储（因为响应可以更新 cookie，并且必须用于后续请求）。你可以通过手动设置 `Cookie` HTTP 请求头来实现：

```php
$client = HttpClient::create([
    'headers' => [
        // 以 name=value 对的形式设置一个 cookie
        'Cookie' => 'flavor=chocolate',

        // 可以一次设置多个 cookie，用 ; 分隔
        'Cookie' => 'flavor=chocolate; size=medium',
    ],
]);
```

### 重定向

默认情况下，HTTP 客户端在发出请求时跟随重定向，最多 20 次。使用 `max_redirects` 设置来配置此行为：

```php
$response = $client->request('GET', 'https://...', [
    // 0 表示不跟随任何重定向
    'max_redirects' => 0,
]);
```

### 重试失败的请求 {#http-client-retry-failed-requests}

有时，请求会因为网络问题或临时服务器错误而失败。Symfony 的 HttpClient 允许使用 `retry_failed` 选项自动重试失败的请求。

默认情况下，失败的请求最多重试 3 次，重试之间有指数延迟（第一次重试 = 1 秒；第三次重试：4 秒），并且仅针对以下 HTTP 状态码：`423`、`425`、`429`、`502` 和 `503`（使用任何 HTTP 方法）以及 `500`、`504`、`507` 和 `510`（使用 HTTP 幂等方法）。

### HTTP 代理

默认情况下，此组件遵循你的操作系统定义的标准环境变量，通过本地代理引导 HTTP 流量。你也可以使用 `proxy` 和 `no_proxy` 选项来设置或覆盖这些设置。

### 进度回调

通过向 `on_progress` 选项提供可调用项，可以在上传/下载完成时跟踪它们：

```php
$response = $client->request('GET', 'https://...', [
    'on_progress' => function (int $dlNow, int $dlSize, array $info): void {
        // $dlNow 是目前下载的字节数
        // $dlSize 是要下载的总大小，如果未知则为 -1
        // $info 是 $response->getInfo() 此时返回的内容
    },
]);
```

### HTTPS 证书

HttpClient 使用系统的证书存储来验证 SSL 证书（而浏览器使用它们自己的存储）。在开发期间使用自签名证书时，建议创建自己的证书颁发机构（CA）并将其添加到系统存储中。

### SSRF（服务器端请求伪造）处理

如果你将 `HttpClient` 与用户提供的 URI 一起使用，最好用 `NoPrivateNetworkHttpClient` 来装饰它。这将确保本地网络对 HTTP 客户端不可访问：

```php
use Symfony\Component\HttpClient\HttpClient;
use Symfony\Component\HttpClient\NoPrivateNetworkHttpClient;

$client = new NoPrivateNetworkHttpClient(HttpClient::create());
// 请求公共网络时没有变化
$client->request('GET', 'https://example.com/');

// 但是，所有对私有网络的请求现在默认被阻止
$client->request('GET', 'http://localhost/');
```

### 使用 URI 模板

`UriTemplateHttpClient` 提供了一个客户端，可以按照 [RFC 6570](https://www.rfc-editor.org/rfc/rfc6570) 的描述轻松使用 URI 模板：

```php
$client = new UriTemplateHttpClient();

// 这将向 URL http://example.org/users?page=1 发出请求
$client->request('GET', 'http://example.org/{resource}{?page}', [
    'vars' => [
        'resource' => 'users',
        'page' => 1,
    ],
]);
```

---

## 性能

### 启用 cURL 支持

此组件可以使用原生 PHP 流、`amphp/http-client` 和 cURL 库发出 HTTP 请求。`HttpClient::create()` 方法在启用 cURL PHP 扩展时选择 cURL 传输。如果找不到 cURL 或版本太旧，则回退到 `AmpHttpClient`，最后回退到 PHP 流。

### HTTP/2 支持

请求 `https` URL 时，如果安装了以下工具之一，HTTP/2 默认启用：

- `libcurl` 包版本 7.36 或更高，与 PHP >= 7.2.17/7.3.4 一起使用；
- `amphp/http-client` Packagist 包版本 4.2 或更高。

---

## 处理响应

所有 HTTP 客户端返回的响应是 `ResponseInterface` 类型的对象：

```php
$response = $client->request('GET', 'https://...');

// 获取响应的 HTTP 状态码
$statusCode = $response->getStatusCode();

// 获取头名称小写的 HTTP 头作为 string[][]
$headers = $response->getHeaders();

// 获取响应体作为字符串
$content = $response->getContent();

// 将响应 JSON 内容转换为 PHP 数组
$content = $response->toArray();

// 将响应内容转换为 PHP 流资源
$content = $response->toStream();

// 取消请求/响应
$response->cancel();

// 返回来自传输层的信息
$httpInfo = $response->getInfo();
```

### 流式响应 {#http-client-streaming-responses}

调用 `stream()` 方法顺序获取响应的*块*，而不是等待整个响应：

```php
$url = 'https://releases.ubuntu.com/18.04.1/ubuntu-18.04.1-desktop-amd64.iso';
$response = $client->request('GET', $url);

if (200 !== $response->getStatusCode()) {
    throw new \Exception('...');
}

// 以块的形式获取响应内容并将其保存到文件中
$fileHandler = fopen('/ubuntu.iso', 'w');
foreach ($client->stream($response) as $chunk) {
    fwrite($fileHandler, $chunk->getContent());
}
```

### 处理异常

有三种类型的异常，所有这些异常都实现 `ExceptionInterface`：

- 实现 `HttpExceptionInterface` 的异常：当你的代码不处理 300-599 范围内的状态码时抛出；
- 实现 `TransportExceptionInterface` 的异常：当发生较低级别的问题时抛出；
- 实现 `DecodingExceptionInterface` 的异常：当内容类型无法解码为预期表示时抛出。

---

## 并发请求 {#http-client-concurrent-requests}

Symfony 的 HTTP 客户端默认发出异步 HTTP 请求。这意味着你不需要配置任何特殊内容即可并行发送多个请求：

```php
$packages = ['console', 'http-kernel', '...', 'routing', 'yaml'];
$responses = [];
foreach ($packages as $package) {
    $uri = sprintf('https://repo.packagist.org/p2/symfony/%s.json', $package);
    // 并发发送所有请求（在读取响应内容之前它们不会阻塞）
    $responses[$package] = $client->request('GET', $uri);
}

$results = [];
// 遍历响应并读取它们的内容
foreach ($responses as $package => $response) {
    $results[$package] = $response->toArray();
}
```

### 多路复用响应

`stream()` 方法可以用于监视响应列表，允许在响应到达时立即处理它们：

```php
foreach ($client->stream($responses) as $response => $chunk) {
    if ($chunk->isFirst()) {
        // $response 头刚刚到达
    } elseif ($chunk->isLast()) {
        // 已收到完整的 $response 体
    } else {
        // $chunk->getContent() 返回刚到达的一段体
    }
}
```

### 处理网络超时

使用 `timeout` 请求选项配置超时：

```php
$response = $client->request('GET', 'https://...', ['timeout' => 2.5]);
```

---

## 缓存请求和响应 {#http-client_caching}

此组件提供了 `CachingHttpClient` 装饰器，可按照 RFC 9111 的描述启用 HTTP 响应缓存：

```yaml
# config/packages/framework.yaml
framework:
    http_client:
        scoped_clients:
            example.client:
                base_uri: 'https://example.com'
                caching:
                    cache_pool: example_cache_pool

    cache:
        pools:
            example_cache_pool:
                adapter: cache.adapter.redis_tag_aware
                tags: true
```

---

## 限制请求数量

此组件提供了 `ThrottlingHttpClient` 装饰器，允许你在特定时期内限制请求数量：

```yaml
# config/packages/framework.yaml
framework:
    http_client:
        scoped_clients:
            example.client:
                base_uri: 'https://example.com'
                rate_limiter: 'http_example_limiter'

    rate_limiter:
        # 在 5 秒内不发送超过 10 个请求
        http_example_limiter:
            policy: 'token_bucket'
            limit: 10
            rate: { interval: '5 seconds', amount: 10 }
```

---

## 消费服务器发送事件

[服务器发送事件](https://html.spec.whatwg.org/multipage/server-sent-events.html)是用于向网页推送数据的互联网标准。Symfony 的 HTTP 客户端提供了 `EventSourceHttpClient` 实现来消费这些服务器发送事件：

```php
use Symfony\Component\HttpClient\Chunk\ServerSentEvent;
use Symfony\Component\HttpClient\EventSourceHttpClient;

// 第二个可选参数是重新连接时间（以秒为单位，默认 = 10）
$client = new EventSourceHttpClient($client, 10);
$source = $client->connect('https://localhost:8080/events');
while ($source) {
    foreach ($client->stream($source, 2) as $r => $chunk) {
        if ($chunk->isTimeout()) {
            continue;
        }

        if ($chunk->isLast()) {
            return;
        }

        // 这是一个持有推送消息的特殊 ServerSentEvent 块
        if ($chunk instanceof ServerSentEvent) {
            // 对服务器事件做一些处理...
        }
    }
}
```

---

## 互操作性

该组件与四种不同的 HTTP 客户端抽象互操作：Symfony Contracts、PSR-18、HTTPlug v1/v2 和原生 PHP 流。

### PSR-18 和 PSR-17

此组件通过 `Psr18Client` 类实现 PSR-18（HTTP 客户端）规范：

```terminal
$ composer require psr/http-client
$ composer require nyholm/psr7
```

### 原生 PHP 流

实现 `ResponseInterface` 的响应可以使用 `StreamWrapper::createResource()` 转换为原生 PHP 流：

```php
use Symfony\Component\HttpClient\HttpClient;
use Symfony\Component\HttpClient\Response\StreamWrapper;

$client = HttpClient::create();
$response = $client->request('GET', 'https://symfony.com/versions.json');

$streamResource = StreamWrapper::createResource($response, $client);

echo stream_get_contents($streamResource);
```

---

## 测试

此组件包含 `MockHttpClient` 和 `MockResponse` 类，用于不应发出实际 HTTP 请求的测试：

```php
use Symfony\Component\HttpClient\MockHttpClient;
use Symfony\Component\HttpClient\Response\MockResponse;

$responses = [
    new MockResponse($body1, $info1),
    new MockResponse($body2, $info2),
];

$client = new MockHttpClient($responses);
$response1 = $client->request('...'); // 返回 $responses[0]
$response2 = $client->request('...'); // 返回 $responses[1]
```

也可以传递动态生成响应的回调：

```php
$callback = function ($method, $url, $options): MockResponse {
    return new MockResponse('...');
};

$client = new MockHttpClient($callback);
```

### 测试请求数据

`MockResponse` 类提供了一些辅助方法来测试请求：

```php
$mockResponse = new MockResponse('', ['http_code' => 204]);
$httpClient = new MockHttpClient($mockResponse, 'https://example.com');

$response = $httpClient->request('DELETE', 'api/article/1337');

$mockResponse->getRequestMethod();   // 返回 "DELETE"
$mockResponse->getRequestUrl();      // 返回 "https://example.com/api/article/1337"
$mockResponse->getRequestOptions();  // 返回包含头、查询参数、体内容等的数组
```

### 使用 HAR 文件测试

现代浏览器（通过其网络选项卡）和 HTTP 客户端允许你使用 [HAR](https://w3c.github.io/web-performance/specs/HAR/Overview.html)（HTTP 存档）格式导出一个或多个 HTTP 请求的信息。你可以使用那些 `.har` 文件来使用 Symfony 的 HTTP 客户端执行测试。
