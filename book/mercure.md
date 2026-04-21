# 使用 Mercure 协议向客户端推送数据

在服务器向客户端实时广播数据的能力是许多现代 Web 和移动应用的需求。

Symfony 提供了一个简单的组件，基于 [Mercure 协议](https://mercure.rocks/spec)，专为此类用例设计。

Mercure 是一个从头开始设计的开放协议，用于从服务器向客户端发布更新。它是基于定时轮询和 WebSocket 的现代高效替代方案。

由于它建立在[服务器发送事件（SSE）](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)之上，Mercure 在现代浏览器中原生支持，并在许多编程语言中有高级实现。

Mercure 附带授权机制、在网络问题时自动重新连接并检索丢失的更新、存在 API、智能手机的"无连接"推送和自动发现。

---

## 安装

### 安装 Symfony Bundle

运行此命令安装 Mercure 支持：

```terminal
$ composer require mercure
```

### 运行 Mercure Hub

要管理持久连接，Mercure 依赖 Hub：一个处理与客户端持久 SSE 连接的专用服务器。Symfony 应用将更新发布到 Hub，Hub 将其广播给客户端。

在生产中，你必须自己安装 Mercure Hub。可以从 [Mercure.rocks](https://mercure.rocks) 下载基于 Caddy Web 服务器的官方开源（AGPL）Hub 的静态二进制文件。

---

## 配置

配置 MercureBundle 的首选方式是使用[环境变量](configuration.md)。

将 Hub 的 URL 设置为 `MERCURE_URL` 和 `MERCURE_PUBLIC_URL` 环境变量的值。客户端还必须向 Mercure Hub 携带 [JSON Web Token（JWT）](https://tools.ietf.org/html/rfc7519)以获得授权发布和订阅更新。

```yaml
# config/packages/mercure.yaml
mercure:
    hubs:
        default:
            url: '%env(MERCURE_URL)%'
            public_url: '%env(MERCURE_PUBLIC_URL)%'
            jwt:
                secret: '%env(MERCURE_JWT_SECRET)%'
                publish: ['https://example.com/foo1', 'https://example.com/foo2']
                subscribe: ['https://example.com/bar1', 'https://example.com/bar2']
                algorithm: 'hmac.sha256'
```

---

## 基本用法

### 发布

Mercure 组件提供了一个表示要发布的更新的 `Update` 值对象，以及一个将更新分发到 Hub 的 `Publisher` 服务：

```php
// src/Controller/PublishController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Mercure\HubInterface;
use Symfony\Component\Mercure\Update;

class PublishController extends AbstractController
{
    public function publish(HubInterface $hub): Response
    {
        $update = new Update(
            'https://example.com/books/1',
            json_encode(['status' => 'OutOfStock'])
        );

        $hub->publish($update);

        return new Response('published!');
    }
}
```

传递给 `Update` 构造函数的第一个参数是正在更新的**主题**。此主题应该是一个 [IRI](https://tools.ietf.org/html/rfc3987)（国际化资源标识符，RFC 3987）：被分发的资源的唯一标识符。

### 订阅

从 Twig 模板中用 JavaScript 订阅更新非常简单：

```html+twig
<script>
const eventSource = new EventSource("{{ mercure('https://example.com/books/1')|escape('js') }}");
eventSource.onmessage = event => {
    // 每次服务器发布更新时都会被调用
    console.log(JSON.parse(event.data));
}
</script>
```

`mercure()` Twig 函数根据配置生成 Mercure Hub 的 URL。

Mercure 还允许订阅多个主题，并使用 URI 模板或特殊值 `*`（匹配所有主题）作为模式：

```html+twig
<script>
{# 订阅多个 Book 资源和所有匹配给定模式的 Review 资源的更新 #}
const eventSource = new EventSource("{{ mercure([
    'https://example.com/books/1',
    'https://example.com/books/2',
    'https://example.com/reviews/{id}'
])|escape('js') }}");

eventSource.onmessage = event => {
    console.log(JSON.parse(event.data));
}
</script>
```

---

## 发现

Mercure 协议带有发现机制。为了利用它，Symfony 应用必须在 `Link` HTTP 头中公开 Mercure Hub 的 URL：

```php
// src/Controller/DiscoverController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Mercure\Discovery;

class DiscoverController extends AbstractController
{
    public function discover(Request $request, Discovery $discovery): JsonResponse
    {
        // Link: <https://hub.example.com/.well-known/mercure>; rel="mercure"
        $discovery->addLink($request);

        return $this->json([
            '@id' => '/books/1',
            'availability' => 'https://schema.org/InStock',
        ]);
    }
}
```

---

## 授权

Mercure 还允许仅向授权客户端分发更新。为此，通过将 `Update` 构造函数的第三个参数设置为 `true` 将更新标记为**私有**：

```php
$update = new Update(
    'https://example.com/books/1',
    json_encode(['status' => 'OutOfStock']),
    true // private
);

$hub->publish($update);
```

要订阅私有更新，订阅者必须向 Hub 提供包含与更新主题匹配的主题选择器的 JWT。

Cookie 可以通过将适当的选项传递给 `mercure()` Twig 函数由 Symfony 自动设置：

```html+twig
<script>
const eventSource = new EventSource("{{ mercure('https://example.com/books/1', { subscribe: 'https://example.com/books/1' })|escape('js') }}", {
    withCredentials: true
});
</script>
```

---

## 测试

在单元测试期间，通常不需要向 Mercure 发送更新。你可以改用 `MockHub` 类：

```php
use Symfony\Component\Mercure\HubInterface;
use Symfony\Component\Mercure\JWT\StaticTokenProvider;
use Symfony\Component\Mercure\MockHub;
use Symfony\Component\Mercure\Update;

$hub = new MockHub('https://internal/.well-known/mercure', new StaticTokenProvider('foo'), function(Update $update): string {
    return 'id';
});
```

---

## 异步分发

你也可以通过提供的 Messenger 组件集成，让 Symfony 异步分发更新：

```php
use Symfony\Component\Mercure\Update;
use Symfony\Component\Messenger\MessageBusInterface;

class PublishController extends AbstractController
{
    public function publish(MessageBusInterface $bus): Response
    {
        $update = new Update(
            'https://example.com/books/1',
            json_encode(['status' => 'OutOfStock'])
        );

        // 同步或异步（Doctrine、RabbitMQ、Kafka...）
        $bus->dispatch($update);

        return new Response('published!');
    }
}
```
