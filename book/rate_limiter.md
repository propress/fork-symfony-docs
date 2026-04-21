# 速率限制器

"速率限制器"控制某些事件（如 HTTP 请求或登录尝试）允许发生的频率。速率限制通常用作防御措施，以保护服务免受过度使用（有意或无意），并保持其可用性。

> **危险**：Symfony 速率限制器需要在 PHP 进程中启动 Symfony，因此它们无法用于防范 DoS 攻击。这类保护必须消耗尽可能少的资源。考虑使用 Apache mod_ratelimit、NGINX 速率限制、Caddy HTTP 速率限制模块或代理（如 AWS 或 Cloudflare）来防止服务器被压垮。

---

## 速率限制策略

### 固定窗口速率限制器

最简单的技术，基于为给定时间间隔设置限制（例如每小时 5,000 个请求，或每 15 分钟 3 次登录尝试）。主要缺点是资源使用在时间上分布不均匀，可能在窗口边缘使服务器过载。

### 滑动窗口速率限制器

固定窗口算法的替代方案，旨在减少突发请求。速率限制基于当前窗口和前一个窗口的近似值来计算。

### 令牌桶速率限制器

实现令牌桶算法，连续更新资源使用预算：

1. 创建一个包含初始令牌集的桶
2. 以预定义频率（例如每秒）向桶中添加新令牌
3. 允许事件会消耗一个或多个令牌
4. 如果桶中仍有令牌，则允许该事件；否则拒绝
5. 如果桶已满，新令牌将被丢弃

---

## 安装

```terminal
$ composer require symfony/rate-limiter
```

---

## 配置

```yaml
# config/packages/rate_limiter.yaml
framework:
    rate_limiter:
        anonymous_api:
            policy: 'fixed_window'
            limit: 100
            interval: '60 minutes'
        authenticated_api:
            policy: 'token_bucket'
            limit: 5000
            rate: { interval: '15 minutes', amount: 500 }
```

---

## 使用速率限制器

### 注入速率限制器服务

**方式 1：使用特定参数名**

```php
// src/Controller/ApiController.php
use Symfony\Component\RateLimiter\RateLimiterFactoryInterface;

class ApiController extends AbstractController
{
    // 参数名 $anonymousApiLimiter 对应 'anonymous_api' 限制器
    public function index(RateLimiterFactoryInterface $anonymousApiLimiter): Response
    {
        // ...
    }
}
```

**方式 2：使用 `#[Target]` 属性**

```php
use Symfony\Component\DependencyInjection\Attribute\Target;

class ApiController extends AbstractController
{
    public function index(
        #[Target('anonymous_api')] RateLimiterFactoryInterface $rateLimiter
    ): Response {
        // ...
    }
}
```

### 使用速率限制器服务

```php
class ApiController extends AbstractController
{
    public function index(Request $request, RateLimiterFactoryInterface $anonymousApiLimiter): Response
    {
        // 基于客户端唯一标识符创建限制器（如 IP 地址）
        $limiter = $anonymousApiLimiter->create($request->getClientIp());

        // consume() 的参数是要消耗的令牌数
        if (false === $limiter->consume(1)->isAccepted()) {
            throw new TooManyRequestsHttpException();
        }

        // 也可以使用 ensureAccepted() 方法
        // $limiter->consume(1)->ensureAccepted();

        // 重置计数器
        // $limiter->reset();

        // ...
    }
}
```

### 等待令牌可用

```php
// 阻塞应用，直到可以消耗给定数量的令牌
$limiter->reserve(1)->wait();

// 可选：传递最大等待时间（秒）
// $limiter->reserve(1, 20)->wait();
```

### 公开速率限制器状态

```php
$limiter = $anonymousApiLimiter->create($request->getClientIp());
$limit = $limiter->consume();
$headers = [
    'X-RateLimit-Remaining' => $limit->getRemainingTokens(),
    'X-RateLimit-Retry-After' => $limit->getRetryAfter()->getTimestamp() - time(),
    'X-RateLimit-Limit' => $limit->getLimit(),
];

if (false === $limit->isAccepted()) {
    return new Response(null, Response::HTTP_TOO_MANY_REQUESTS, $headers);
}
```

---

## 存储速率限制器状态

默认情况下，所有限制器使用缓存存储状态（`cache.rate_limiter` 缓存池）。可以使用 `cache_pool` 选项覆盖：

```yaml
# config/packages/rate_limiter.yaml
framework:
    rate_limiter:
        anonymous_api:
            cache_pool: 'cache.anonymous_rate_limiter'
```
