# HTTP 缓存

富 Web 应用的本质意味着它们是动态的。无论你的应用效率多高，每个请求总是比提供静态文件包含更多开销。通常这没问题。但当你需要请求速度极快时，你需要 HTTP 缓存。

---

## 站在巨人肩膀上的缓存

使用 HTTP 缓存，你可以缓存页面的完整输出（即响应），并在后续请求时完全绕过你的应用。缓存整个响应对于高度动态的网站并不总是可行的，或者真的不行吗？使用[边缘端包含（ESI）](http_cache/esi.md)，你可以在你的网站的*部分*上使用 HTTP 缓存的强大功能。

Symfony 缓存系统的不同之处在于它依赖于 [RFC 7234 - Caching](https://tools.ietf.org/html/rfc7234) 中定义的 HTTP 缓存的简单性和强大性。Symfony 不是重新发明缓存方法，而是拥抱定义 Web 基本通信的标准。

---

## 使用网关缓存进行缓存 {#gateway-caches}

使用 HTTP 进行缓存时，*缓存*与你的应用完全分离，位于你的应用和发出请求的客户端之间。

缓存的工作是接受来自客户端的请求，并将它们传回给你的应用。缓存还将接收来自你的应用的响应，并将它们转发给客户端。缓存是客户端和你的应用之间请求-响应通信的"中间人"。

在此过程中，缓存将存储每个被认为是"可缓存的"响应。如果再次请求同一个资源，缓存会将缓存的响应发送给客户端，完全忽略你的应用。

这种类型的缓存称为 HTTP 网关缓存，存在许多这样的缓存，如 [Varnish](https://varnish-cache.org/)、[Squid 反向代理模式](https://wiki.squid-cache.org/SquidFaq/ReverseProxy)和 Symfony 反向代理。

> **提示**
>
> 网关缓存有时被称为反向代理缓存、代理缓存，甚至 HTTP 加速器。

### Symfony 反向代理 {#symfony2-reverse-proxy}

Symfony 附带了一个用 PHP 编写的反向代理（即网关缓存）。它不是像 Varnish 那样功能齐全的反向代理缓存，但它是一个很好的起点。

使用 `framework.http_cache` 选项为[生产环境](configuration.md#配置环境)启用代理：

```yaml
# config/packages/framework.yaml
when@prod:
    framework:
        http_cache: true
```

内核将立即充当反向代理：缓存来自你的应用的响应并将其返回给客户端。

> **提示**
>
> 如果你在[调试模式](configuration.md#调试模式)下，Symfony 会自动向响应添加一个 `X-Symfony-Cache` 头。你也可以使用 `trace_level` 配置选项，将其设置为 `none`、`short` 或 `full` 来添加此信息。

---

## 使响应 HTTP 可缓存 {#http-cache-introduction}

一旦你添加了反向代理缓存（例如 Symfony 反向代理或 Varnish），你就准备好缓存你的响应了。要做到这一点，你需要向你的缓存*通知*哪些响应是可缓存的以及缓存多长时间。这通过在响应上设置 HTTP 缓存头来完成。

HTTP 指定了四个响应缓存头，你可以设置它们来启用缓存：

- `Cache-Control`
- `Expires`
- `ETag`
- `Last-Modified`

这四个头用于通过*两种*不同模型帮助缓存你的响应：

1. **过期缓存**：用于在特定时间内缓存你的整个响应（例如 24 小时）。简单，但缓存失效更困难；
2. **验证缓存**：更复杂：用于缓存你的响应，但允许你在内容更改时立即动态使其失效。

### 过期缓存 {#http-cache-expiration-intro}

缓存响应最*简单*的方法是将其缓存特定时间：

```php-attributes
// src/Controller/BlogController.php
use Symfony\Component\HttpKernel\Attribute\Cache;
// ...

#[Cache(public: true, maxage: 3600, mustRevalidate: true)]
public function index(): Response
{
    return $this->render('blog/index.html.twig', []);
}
```

```php
// src/Controller/BlogController.php
use Symfony\Component\HttpFoundation\Response;

public function index(): Response
{
    // 以某种方式创建 Response 对象，例如渲染模板
    $response = $this->render('blog/index.html.twig', []);

    // 公开缓存 3600 秒
    $response->setPublic();
    $response->setMaxAge(3600);

    // （可选）设置自定义 Cache-Control 指令
    $response->headers->addCacheControlDirective('must-revalidate', true);

    return $response;
}
```

由于此新代码，你的 HTTP 响应将具有以下头：

```text
Cache-Control: public, maxage=3600, must-revalidate
```

这告诉你的 HTTP 反向代理将此响应缓存 3600 秒。如果任何人在 3600 秒之前再次请求此 URL，你的应用将*完全*不会被访问。

此方法性能很好且使用简单。但是，不支持缓存*失效*。如果你的内容更改，你需要等到缓存过期才能更新页面。

更多关于过期缓存的信息，请参阅 [http_cache/expiration](http_cache/expiration.md)。

### 验证缓存 {#http-cache-validation-intro}

使用过期缓存时，你说"缓存 3600 秒！"。但是，当有人更新缓存的内容时，你不会在缓存过期之前在你的网站上看到该内容。

如果你需要*立即*看到更新的内容，你需要[使你的缓存失效](#http-cache-invalidation)或使用验证缓存模型。

更多详情，请参阅 [http_cache/validation](http_cache/validation.md)。

### 安全方法：仅缓存 GET 或 HEAD 请求

HTTP 缓存只适用于"安全"的 HTTP 方法（如 GET 和 HEAD）。这意味着三点：

- 不要尝试缓存 PUT 或 DELETE 请求；
- POST 请求通常被认为是不可缓存的；
- 在响应 GET 或 HEAD 请求时，你*永远*不应该更改你的应用状态。

### 更多响应方法

Response 类提供了更多与缓存相关的方法：

```php
// 将响应标记为过时
$response->expire();

// 强制响应返回没有内容的适当 304 响应
$response->setNotModified();
```

此外，大多数与缓存相关的 HTTP 头可以通过单一的 `setCache()` 方法设置：

```php
$response->setCache([
    'must_revalidate'  => false,
    'no_cache'         => false,
    'no_store'         => false,
    'no_transform'     => false,
    'public'           => true,
    'private'          => false,
    'proxy_revalidate' => false,
    'max_age'          => 600,
    's_maxage'         => 600,
    'immutable'        => true,
    'last_modified'    => new \DateTime(),
    'etag'             => 'abcdef'
]);
```

---

## 缓存失效 {#http-cache-invalidation}

缓存失效*不是* HTTP 规范的一部分。但它可以非常有用，当你的网站上的某些内容更新时，立即删除各种 HTTP 缓存条目。

更多详情，请参阅 [http_cache/cache_invalidation](http_cache/cache_invalidation.md)。

---

## 使用边缘端包含

当页面包含动态部分时，你可能无法缓存整个页面，而只能缓存其中的一部分。阅读 [http_cache/esi](http_cache/esi.md) 以了解如何为页面的特定部分配置不同的缓存策略。

---

## HTTP 缓存与用户会话

每当在请求期间启动会话时，Symfony 会将响应转换为私有不可缓存的响应。这是不缓存私人用户信息（例如购物车、用户配置文件详情等）并将其暴露给其他访客的最佳默认行为。

要禁用 Symfony 使使用会话的请求不可缓存的默认行为，可以向你的响应添加以下内部头：

```php
use Symfony\Component\HttpKernel\EventListener\AbstractSessionListener;

$response->headers->set(AbstractSessionListener::NO_AUTO_CACHE_CONTROL_HEADER, 'true');
```

---

## 总结

Symfony 被设计为遵循 HTTP 的既定规则。缓存也不例外。掌握 Symfony 缓存系统意味着熟悉 HTTP 缓存模型并有效使用它们。

---

## 延伸阅读

- [过期缓存](http_cache/expiration.md)
- [验证缓存](http_cache/validation.md)
- [缓存失效](http_cache/cache_invalidation.md)
- [边缘端包含（ESI）](http_cache/esi.md)
- [Varnish 配置](http_cache/varnish.md)
- [缓存变化](http_cache/cache_vary.md)
