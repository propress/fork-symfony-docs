# 使用 WebLink 预加载资源与资源提示

Symfony 通过 [WebLink](https://github.com/symfony/web-link) 组件提供对 `Link` HTTP 头的原生支持，这是利用现代浏览器预加载能力提升应用性能的关键。

`Link` 头用于在客户端尚未意识到需要某些资源（例如 CSS 和 JavaScript 文件）之前，提前提示这些资源。WebLink 支持以下几种优化：

* 告知浏览器预加载当前页面所需的资源；
* 发送 [103 Early Hints](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/103) 响应，使浏览器在完整响应准备好之前就开始下载资源（参见[早期提示](#发送早期提示)）；
* 提前进行 DNS 查找、TCP 握手或 TLS 协商。

> **注意：** 其中一些特性（如 Early Hints 或资源提示）在 HTTPS 安全连接下效果最佳。主流 Web 服务器（Apache、nginx、Caddy 等）均支持此功能，你也可以使用由 Symfony 社区成员 Kévin Dunglas 创建的 [Docker installer and runtime for Symfony](https://github.com/dunglas/symfony-docker)。

## 安装

在使用 Symfony Flex 的应用中，运行以下命令来安装 WebLink：

```terminal
$ composer require symfony/web-link
```

## 预加载资源

假设你的应用包含如下网页：

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>My Application</title>
    <link rel="stylesheet" href="/app.css">
</head>
<body>
    <main role="main" class="container">
        <!-- ... -->
    </main>
</body>
</html>
```

在传统 HTTP 工作流中，加载此页面时浏览器会分别发送一个 HTML 文档请求和一个 CSS 文件请求。通过 `Link` HTTP 头，你的应用可以在处理 HTML 时提示浏览器预加载 CSS 文件。

这对于未直接链接到 HTML 但需要尽早加载的资源（例如 CSS 样式表中引用的字体文件）非常有用。

要预加载资源，请使用 WebLink 提供的 `preload()` Twig 函数。需要指定 [`"as"` 属性](https://w3c.github.io/preload/#as-attribute)，浏览器使用它来正确排列资源优先级并符合内容安全策略：

```html+twig
<head>
    <!-- ... -->
    <link rel="preload" href="{{ preload('/fonts/myfont.woff2', {as: 'font'}) }}">
    <!-- 可以选择性地添加更多属性 -->
    <!-- <link rel="preload" href="{{ preload('/fonts/myfont.woff2', {as: 'font', type: 'font/woff2', crossorigin: 'anonymous'}) }}"> -->

    <link rel="stylesheet" href="/app.css">
</head>
```

`preload()` 函数会在响应中添加 `Link` HTTP 头（例如 `Link: </fonts/myfont.woff2>; rel="preload"; as="font"`），告知浏览器（或兼容 HTTP/2 的服务器或 CDN）尽早开始获取资源。你也可以将其与 `asset()` 函数结合使用：

```html+twig
<link rel="preload" href="{{ preload(asset('build/app.css'), {as: 'style'}) }}" as="style">
<link rel="stylesheet" href="{{ asset('build/app.css') }}">
```

重新加载页面后，感知性能会提升，因为浏览器在收到 `Link` 头后就立即开始下载 CSS 文件，无需等待解析完整的 HTML。

> **提示：** 使用 AssetMapper 组件（例如 `importmap('app')`）时，无需手动添加 `<link rel="preload">` 标签。当 WebLink 组件可用时，`importmap()` Twig 函数会自动为你添加 `Link` HTTP 头。

此外，根据[优先级提示规范](https://wicg.github.io/priority-hints/)，你可以使用 `importance` 属性来指定资源下载的优先级：

```html+twig
<head>
    <!-- ... -->
    <link rel="preload" href="{{ preload('/app.css', {as: 'style', importance: 'low'}) }}" as="style">
    <!-- ... -->
</head>
```

### 工作原理

WebLink 组件管理添加到响应中的 `Link` HTTP 头。使用 `preload()` 函数时，会向响应中添加如下头信息：`Link </fonts/myfont.woff2>; rel="preload"; as="font"`。

浏览器收到此头信息后，会立即开始下载资源，而无需等待在 HTML 中遇到对应标签。

[Cloudflare](https://blog.cloudflare.com/announcing-support-for-http-2-server-push-2/)、[Fastly](https://docs.fastly.com/en/guides/http2-server-push) 和 [Akamai](https://http2.akamai.com/) 等流行的代理服务和 CDN 也利用 `Link` 头来优化资源传输并提升你的应用在生产环境中的性能。

## 发送早期提示

默认情况下，`Link` 头随最终响应一起发送。但是，你可以通过在完整响应准备好之前发送这些头信息来进一步提升性能，即使用 [103 Early Hints](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/103) 响应。这会告知浏览器在服务器仍在准备页面时就开始下载资源。

> **注意：** 要使此功能正常工作，你使用的 [SAPI](https://www.php.net/manual/en/function.header.php) 必须支持此特性，例如 [FrankenPHP](https://frankenphp.dev)。

发送早期提示的最简单方式是使用 `preload()` Twig 函数。当你的 Web 服务器支持早期提示时，通过 `preload()` 添加的 `Link` 头会自动作为 `103` 响应发送：

```html+twig
<head>
    <!-- ... -->
    <link rel="preload" href="{{ preload('/app.css', {as: 'style'}) }}" as="style">
    <!-- ... -->
</head>
```

如需更多控制，你可以通过 `sendEarlyHints()` 方法在控制器动作中显式发送早期提示：

```php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\WebLink\Link;

class HomepageController extends AbstractController
{
    #[Route("/", name: "homepage")]
    public function index(): Response
    {
        $response = $this->sendEarlyHints([
            new Link(rel: 'preconnect', href: 'https://fonts.google.com'),
            new Link(href: '/style.css')->withAttribute('as', 'style'),
            new Link(href: '/script.js')->withAttribute('as', 'script'),
        ]);

        // prepare the contents of the response...

        return $this->render('homepage/index.html.twig', response: $response);
    }
}
```

从技术上讲，Early Hints 是状态码为 `103` 的信息性 HTTP 响应。`sendEarlyHints()` 方法创建一个该状态码的 `Response` 对象，并立即发送其头信息。

这样，浏览器可以立即开始下载资源，例如上面示例中的 `style.css` 和 `script.js` 文件。`sendEarlyHints()` 方法还会返回该 `Response` 对象，你必须使用它来创建控制器动作发送的完整响应。

> **提示：** 使用 AssetMapper 组件时，资源文件名包含版本哈希（例如 `styles-3c16d9220694c0e56d8648f25e6035e9.css`）。要在早期提示中引用正确的带版本 URL，请使用 `AssetMapperInterface` 服务：
>
> ```php
> use Symfony\Component\AssetMapper\AssetMapperInterface;
>
> class HomepageController extends AbstractController
> {
>     public function index(AssetMapperInterface $assetMapper): Response
>     {
>         $response = $this->sendEarlyHints([
>             new Link(href: $assetMapper->getAsset('styles/app.css')->publicPath)
>                 ->withAttribute('as', 'style'),
>         ]);
>
>         return $this->render('homepage/index.html.twig', response: $response);
>     }
> }
> ```

## 资源提示

[Resource Hints](https://www.w3.org/TR/resource-hints/) 供应用程序帮助浏览器决定应该优先下载、预处理或连接哪些资源。

WebLink 组件提供以下 Twig 函数来发送这些提示：

* `dns_prefetch()`："表示一个来源（例如 `https://foo.cloudfront.net`），该来源将用于获取所需资源，用户代理应尽早解析它"。
* `preconnect()`："表示一个来源（例如 `https://www.google-analytics.com`），该来源将用于获取所需资源。提前建立连接（包括 DNS 查找、TCP 握手和可选的 TLS 协商）可以让用户代理掩盖建立连接的高延迟成本"。
* `prefetch()`："标识下一次导航可能需要的资源，用户代理*应当*获取它，以便在将来请求该资源时，用户代理可以提供更快的响应"。
* `prerender()`："**已废弃**，由 [Speculation Rules API](https://developer.mozilla.org/docs/Web/API/Speculation_Rules_API) 取代，标识下一次导航可能需要的资源，用户代理*应当*获取并执行它，以便在之后请求该资源时，用户代理可以提供更快的响应"。

该组件还支持发送与性能无关的 HTTP 链接，以及实现 [PSR-13](https://www.php-fig.org/psr/psr-13/) 标准的任何链接。例如，[HTML 规范中定义的任何链接](https://html.spec.whatwg.org/dev/links.html#linkTypes)：

```html+twig
<head>
    <!-- ... -->
    <link rel="alternate" href="{{ link('/index.jsonld', 'alternate') }}">
    <link rel="preload" href="{{ preload('/app.css', {as: 'style', nopush: true}) }}" as="style">
    <!-- ... -->
</head>
```

上面的代码片段会将以下 HTTP 头发送到客户端：`Link: </index.jsonld>; rel="alternate",</app.css>; rel="preload"; nopush`

你也可以直接在控制器和服务中向 HTTP 响应添加链接：

```php
// src/Controller/BlogController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\WebLink\GenericLinkProvider;
use Symfony\Component\WebLink\Link;

class BlogController extends AbstractController
{
    public function index(Request $request): Response
    {
        // using the addLink() shortcut provided by AbstractController
        $this->addLink($request, new Link('preload', '/app.css')->withAttribute('as', 'style'));

        // alternative if you don't want to use the addLink() shortcut
        $linkProvider = $request->attributes->get('_links', new GenericLinkProvider());
        $request->attributes->set('_links', $linkProvider->withLink(
            new Link('preload', '/app.css')->withAttribute('as', 'style')
        ));

        return $this->render('...');
    }
}
```

> **提示：** 链接关系的可能值（`'preload'`、`'preconnect'` 等）也被定义为 `Symfony\Component\WebLink\Link` 类中的常量（例如 `Link::REL_PRELOAD`、`Link::REL_PRECONNECT` 等）。

## 解析 Link 头

一些第三方 API 使用 `Link` HTTP 头提供资源（例如分页 URL）。WebLink 组件提供 `Symfony\Component\WebLink\HttpHeaderParser` 工具类来解析这些头信息，并将其转换为 `Symfony\Component\WebLink\Link` 实例：

```php
use Symfony\Component\WebLink\HttpHeaderParser;

$parser = new HttpHeaderParser();
// get the value of the Link header from the Request
$linkHeader = '</foo.css>; rel="prerender",</bar.otf>; rel="dns-prefetch"; pr="0.7",</baz.js>; rel="preload"; as="script"';

$links = $parser->parse($linkHeader)->getLinks();
$links[0]->getRels();       // ['prerender']
$links[1]->getAttributes(); // ['pr' => '0.7']
$links[2]->getHref();       // '/baz.js'
```
