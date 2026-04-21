# PSR-7 桥接

PSR-7 桥接将 [HttpFoundation](http_foundation.md) 对象与实现 [PSR-7][] 所定义 HTTP 消息接口的对象相互转换。

## 安装

```terminal
$ composer require symfony/psr-http-message-bridge
```

该桥接还需要 PSR-7 和 [PSR-17][] 的实现才能将 HttpFoundation 对象转换为 PSR-7 对象。以下命令安装 `nyholm/psr7` 库——一个轻量且快速的 PSR-7 实现，但你也可以使用任何[实现 psr/http-factory-implementation 的库][libraries that implement psr/http-factory-implementation]：

```terminal
$ composer require nyholm/psr7
```

## 使用

### 将 HttpFoundation 对象转换为 PSR-7

桥接提供了名为 [HttpMessageFactoryInterface][] 的工厂接口，用于从 HttpFoundation 对象构建实现 PSR-7 接口的对象。

以下代码片段说明了如何将 `Symfony\Component\HttpFoundation\Request` 转换为实现 `Psr\Http\Message\ServerRequestInterface` 接口的 `Nyholm\Psr7\ServerRequest` 类：

```php
use Nyholm\Psr7\Factory\Psr17Factory;
use Symfony\Bridge\PsrHttpMessage\Factory\PsrHttpFactory;
use Symfony\Component\HttpFoundation\Request;

$symfonyRequest = new Request([], [], [], [], [], ['HTTP_HOST' => 'dunglas.fr'], 'Content');
// 必须设置 HTTP_HOST 服务器键以避免意外错误

$psr17Factory = new Psr17Factory();
$psrHttpFactory = new PsrHttpFactory($psr17Factory, $psr17Factory, $psr17Factory, $psr17Factory);
$psrRequest = $psrHttpFactory->createRequest($symfonyRequest);
```

以及从 `Symfony\Component\HttpFoundation\Response` 转换为实现 `Psr\Http\Message\ResponseInterface` 接口的 `Nyholm\Psr7\Response` 类：

```php
use Nyholm\Psr7\Factory\Psr17Factory;
use Symfony\Bridge\PsrHttpMessage\Factory\PsrHttpFactory;
use Symfony\Component\HttpFoundation\Response;

$symfonyResponse = new Response('Content');

$psr17Factory = new Psr17Factory();
$psrHttpFactory = new PsrHttpFactory($psr17Factory, $psr17Factory, $psr17Factory, $psr17Factory);
$psrResponse = $psrHttpFactory->createResponse($symfonyResponse);
```

### 将实现 PSR-7 接口的对象转换为 HttpFoundation

另一方面，桥接提供了名为 [HttpFoundationFactoryInterface][] 的工厂接口，用于从实现 PSR-7 接口的对象构建 HttpFoundation 对象。

以下代码片段说明了如何将实现 `Psr\Http\Message\ServerRequestInterface` 接口的对象转换为 `Symfony\Component\HttpFoundation\Request` 实例：

```php
use Symfony\Bridge\PsrHttpMessage\Factory\HttpFoundationFactory;

// $psrRequest 是 Psr\Http\Message\ServerRequestInterface 的实例

$httpFoundationFactory = new HttpFoundationFactory();
$symfonyRequest = $httpFoundationFactory->createRequest($psrRequest);
```

从实现 `Psr\Http\Message\ResponseInterface` 的对象转换为 `Symfony\Component\HttpFoundation\Response` 实例：

```php
use Symfony\Bridge\PsrHttpMessage\Factory\HttpFoundationFactory;

// $psrResponse 是 Psr\Http\Message\ResponseInterface 的实例

$httpFoundationFactory = new HttpFoundationFactory();
$symfonyResponse = $httpFoundationFactory->createResponse($psrResponse);
```

[PSR-7]: https://www.php-fig.org/psr/psr-7/
[PSR-17]: https://www.php-fig.org/psr/psr-17/
[libraries that implement psr/http-factory-implementation]: https://packagist.org/providers/psr/http-factory-implementation
[HttpMessageFactoryInterface]: https://github.com/symfony/psr-http-message-bridge/blob/main/HttpMessageFactoryInterface.php
[HttpFoundationFactoryInterface]: https://github.com/symfony/psr-http-message-bridge/blob/main/HttpFoundationFactoryInterface.php
