# Asset 组件

> Asset 组件管理 Web 资源（如 CSS 样式表、JavaScript 文件和图像文件）的 URL 生成和版本控制。

过去，Web 应用程序通常会硬编码 Web 资源的 URL，例如：

```html
<link rel="stylesheet" type="text/css" href="/css/main.css">

<!-- ... -->

<a href="/"><img src="/images/logo.png" alt="logo"></a>
```

除非 Web 应用程序极其简单，否则不再推荐这种做法。硬编码 URL 可能有以下缺点：

* **模板变得冗长**：你必须为每个资源写出完整路径。使用 Asset 组件，你可以将资源分组到包中，以避免重复其路径的公共部分；
* **版本控制困难**：必须为每个应用程序单独管理。为资源 URL 添加版本（例如 `main.css?v=5`）对于某些应用程序至关重要，因为它允许你控制资源的缓存方式。Asset 组件允许你为每个包定义不同的版本控制策略；
* **移动资源位置**繁琐且容易出错：它要求你仔细更新所有模板中包含的所有资源的 URL。Asset 组件只需更改与资源包关联的基础路径值，即可轻松移动资源；
* **几乎不可能使用多个 CDN**：这种技术要求你在每次请求时随机更改资源的 URL。Asset 组件为任意数量的多个 CDN 提供开箱即用的支持，包括普通（`http://`）和安全（`https://`）协议。

## 安装

```terminal
$ composer require symfony/asset
```

## 使用

### 资源包

Asset 组件通过包来管理资源。一个包将所有共享相同属性的资源分组在一起：版本控制策略、基础路径、CDN 主机等。在以下基本示例中，创建了一个不带任何版本控制的包来管理资源：

```php
use Symfony\Component\Asset\Package;
use Symfony\Component\Asset\VersionStrategy\EmptyVersionStrategy;

$package = new Package(new EmptyVersionStrategy());

// 绝对路径
echo $package->getUrl('/image.png');
// 结果：/image.png

// 相对路径
echo $package->getUrl('image.png');
// 结果：image.png
```

包实现了 `Symfony\Component\Asset\PackageInterface`，该接口定义了以下两个方法：

`Symfony\Component\Asset\PackageInterface::getVersion`
    返回资源的资源版本。

`Symfony\Component\Asset\PackageInterface::getUrl`
    返回绝对路径或根相对的公共路径。

使用包，你可以：

A) 对资源进行版本控制；
B) 为资源设置公共基础路径（例如 `/css`）；
C) 为资源配置 CDN

### 版本化资源

Asset 组件的主要功能之一是能够管理应用程序资源的版本控制。资源版本通常用于控制这些资源的缓存方式。

Asset 组件不依赖简单的版本机制，而是允许你通过 PHP 类定义高级版本控制策略。两种内置策略是 `Symfony\Component\Asset\VersionStrategy\EmptyVersionStrategy`（不向资源添加任何版本）和 `Symfony\Component\Asset\VersionStrategy\StaticVersionStrategy`（允许你使用格式字符串设置版本）。

在此示例中，`StaticVersionStrategy` 用于将 `v1` 后缀附加到任何资源路径：

```php
use Symfony\Component\Asset\Package;
use Symfony\Component\Asset\VersionStrategy\StaticVersionStrategy;

$package = new Package(new StaticVersionStrategy('v1'));

// 绝对路径
echo $package->getUrl('/image.png');
// 结果：/image.png?v1

// 相对路径
echo $package->getUrl('image.png');
// 结果：image.png?v1
```

如果你想修改版本格式，可以将兼容 `sprintf` 的格式字符串作为 `StaticVersionStrategy` 构造函数的第二个参数传入：

```php
// 在版本值之前加上 'version' 单词
$package = new Package(new StaticVersionStrategy('v1', '%s?version=%s'));

echo $package->getUrl('/image.png');
// 结果：/image.png?version=v1

// 将资源版本放在其路径之前
$package = new Package(new StaticVersionStrategy('v1', '%2$s/%1$s'));

echo $package->getUrl('/image.png');
// 结果：/v1/image.png

echo $package->getUrl('image.png');
// 结果：v1/image.png
```

#### JSON 文件清单

由 [Webpack] 等工具使用的一种流行资源版本控制管理策略，是生成一个 JSON 文件，将所有源文件名映射到其对应的输出文件：

```json
{
    "css/app.css": "build/css/app.b916426ea1d10021f3f17ce8031f93c2.css",
    "js/app.js": "build/js/app.13630905267b809161e71d0f8a0c017b.js",
    "...": "..."
}
```

在这种情况下，使用 `Symfony\Component\Asset\VersionStrategy\JsonManifestVersionStrategy`：

```php
use Symfony\Component\Asset\Package;
use Symfony\Component\Asset\VersionStrategy\JsonManifestVersionStrategy;

// 假设上面的 JSON 文件名为 "rev-manifest.json"
$package = new Package(new JsonManifestVersionStrategy(__DIR__.'/rev-manifest.json'));

echo $package->getUrl('css/app.css');
// 结果：build/css/app.b916426ea1d10021f3f17ce8031f93c2.css
```

如果请求的资源在 `rev-manifest.json` 文件中*未找到*，将返回原始的*未修改*资源路径。`$strictMode` 参数有助于调试问题，因为当资源未在清单中列出时会抛出异常：

```php
use Symfony\Component\Asset\Package;
use Symfony\Component\Asset\VersionStrategy\JsonManifestVersionStrategy;

// $strictMode 的值可以根据环境不同，调试时为 "true"，稳定性要求时为 "false"。
$strictMode = true;
// 假设上面的 JSON 文件名为 "rev-manifest.json"
$package = new Package(new JsonManifestVersionStrategy(__DIR__.'/rev-manifest.json', null, $strictMode));

echo $package->getUrl('not-found.css');
// 错误：
```

如果你的 JSON 文件不在本地文件系统上，但可以通过 HTTP 访问，请使用 `Symfony\Component\Asset\VersionStrategy\JsonManifestVersionStrategy` 与 [HttpClient 组件](http_client.md)：

```php
use Symfony\Component\Asset\Package;
use Symfony\Component\Asset\VersionStrategy\JsonManifestVersionStrategy;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$manifestUrl = 'https://cdn.example.com/rev-manifest.json';
$package = new Package(new JsonManifestVersionStrategy($manifestUrl, $httpClient));
```

#### 自定义版本策略

使用 `Symfony\Component\Asset\VersionStrategy\VersionStrategyInterface` 定义你自己的版本控制策略。例如，你的应用程序可能需要将当前日期附加到所有 Web 资源，以便每天清除缓存：

```php
use Symfony\Component\Asset\VersionStrategy\VersionStrategyInterface;

class DateVersionStrategy implements VersionStrategyInterface
{
    private string $version;

    public function __construct()
    {
        $this->version = date('Ymd');
    }

    public function getVersion(string $path): string
    {
        return $this->version;
    }

    public function applyVersion(string $path): string
    {
        return sprintf('%s?v=%s', $path, $this->getVersion($path));
    }
}
```

### 分组资源

通常，许多资源都位于一个公共路径下（例如 `/static/images`）。如果是这种情况，请将默认的 `Symfony\Component\Asset\Package` 类替换为 `Symfony\Component\Asset\PathPackage`，以避免重复该路径：

```php
use Symfony\Component\Asset\PathPackage;
// ...

$pathPackage = new PathPackage('/static/images', new StaticVersionStrategy('v1'));

echo $pathPackage->getUrl('logo.png');
// 结果：/static/images/logo.png?v1

// 使用绝对路径时，基础路径被忽略
echo $pathPackage->getUrl('/logo.png');
// 结果：/logo.png?v1
```

#### 感知请求上下文的资源

如果你在项目中也使用了 [HttpFoundation](components/http_foundation.md) 组件（例如在 Symfony 应用中），`PathPackage` 类可以考虑当前请求的上下文：

```php
use Symfony\Component\Asset\Context\RequestStackContext;
use Symfony\Component\Asset\PathPackage;
// ...

$pathPackage = new PathPackage(
    '/static/images',
    new StaticVersionStrategy('v1'),
    new RequestStackContext($requestStack)
);

echo $pathPackage->getUrl('logo.png');
// 结果：/somewhere/static/images/logo.png?v1

// 对于资源使用绝对路径时，"基础路径"和"基础 URL"都被忽略
echo $pathPackage->getUrl('/logo.png');
// 结果：/logo.png?v1
```

现在请求上下文已设置，`PathPackage` 将在前面加上当前请求的基础 URL。例如，如果你的整个站点托管在 Web 服务器根目录的 `/somewhere` 目录下，并且配置的基础路径是 `/static/images`，则所有路径都将以 `/somewhere/static/images` 为前缀。

### 绝对资源和 CDN

将资源托管在不同域名和 CDN（*内容分发网络*）上的应用程序应使用 `Symfony\Component\Asset\UrlPackage` 类来为其资源生成绝对 URL：

```php
use Symfony\Component\Asset\UrlPackage;
// ...

$urlPackage = new UrlPackage(
    'https://static.example.com/images/',
    new StaticVersionStrategy('v1')
);

echo $urlPackage->getUrl('/logo.png');
// 结果：https://static.example.com/images/logo.png?v1
```

你也可以传递与协议无关的 URL：

```php
use Symfony\Component\Asset\UrlPackage;
// ...

$urlPackage = new UrlPackage(
    '//static.example.com/images/',
    new StaticVersionStrategy('v1')
);

echo $urlPackage->getUrl('/logo.png');
// 结果：//static.example.com/images/logo.png?v1
```

这很有用，因为如果访问者通过 HTTPS 浏览你的站点，资源将自动通过 HTTPS 请求。如果你想使用这个功能，请确保你的 CDN 主机支持 HTTPS。

如果你从多个域名提供资源以提高应用程序性能，可以将 URL 数组作为第一个参数传递给 `UrlPackage` 构造函数：

```php
use Symfony\Component\Asset\UrlPackage;
// ...

$urls = [
    'https://static1.example.com/images/',
    'https://static2.example.com/images/',
];
$urlPackage = new UrlPackage($urls, new StaticVersionStrategy('v1'));

echo $urlPackage->getUrl('/logo.png');
// 结果：https://static1.example.com/images/logo.png?v1
echo $urlPackage->getUrl('/icon.png');
// 结果：https://static2.example.com/images/icon.png?v1
```

对于每个资源，将随机使用其中一个 URL。但是，选择是确定性的，这意味着每个资源将始终由同一个域名提供。这种行为简化了 HTTP 缓存的管理。

#### 感知请求上下文的资源

与应用程序相对资源类似，绝对资源也可以考虑当前请求的上下文。在这种情况下，只考虑请求方案，以便选择适当的基础 URL（HTTPS 请求使用 HTTPS 或协议相对 URL，HTTP 请求使用任何基础 URL）：

```php
use Symfony\Component\Asset\Context\RequestStackContext;
use Symfony\Component\Asset\UrlPackage;
// ...

$urlPackage = new UrlPackage(
    ['http://example.com/', 'https://example.com/'],
    new StaticVersionStrategy('v1'),
    new RequestStackContext($requestStack)
);

echo $urlPackage->getUrl('/logo.png');
// 假设 RequestStackContext 指示我们在安全主机上
// 结果：https://example.com/logo.png?v1
```

### 命名包

管理大量不同资源的应用程序可能需要将它们分组到具有相同版本控制策略和基础路径的包中。Asset 组件包含一个 `Symfony\Component\Asset\Packages` 类来简化多个包的管理。

在以下示例中，所有包使用相同的版本控制策略，但它们都有不同的基础路径：

```php
use Symfony\Component\Asset\Package;
use Symfony\Component\Asset\Packages;
use Symfony\Component\Asset\PathPackage;
use Symfony\Component\Asset\UrlPackage;
// ...

$versionStrategy = new StaticVersionStrategy('v1');

$defaultPackage = new Package($versionStrategy);

$namedPackages = [
    'img' => new UrlPackage('https://img.example.com/', $versionStrategy),
    'doc' => new PathPackage('/somewhere/deep/for/documents', $versionStrategy),
];

$packages = new Packages($defaultPackage, $namedPackages);
```

`Packages` 类允许你定义一个默认包，该包将应用于未指定要使用的包名称的资源。此外，该应用程序定义了一个名为 `img` 的包，用于从外部域名提供图像，以及一个 `doc` 包，以避免在模板中链接文档时重复长路径：

```php
echo $packages->getUrl('/main.css');
// 结果：/main.css?v1

echo $packages->getUrl('/logo.png', 'img');
// 结果：https://img.example.com/logo.png?v1

echo $packages->getUrl('resume.pdf', 'doc');
// 结果：/somewhere/deep/for/documents/resume.pdf?v1
```

### 本地文件和其他协议

除了 HTTP，该组件还支持其他协议（例如 `file://` 和 `ftp://`）。这允许例如提供本地文件以提高性能：

```php
use Symfony\Component\Asset\UrlPackage;
// ...

$localPackage = new UrlPackage(
    'file:///path/to/images/',
    new EmptyVersionStrategy()
);

$ftpPackage = new UrlPackage(
    'ftp://example.com/images/',
    new EmptyVersionStrategy()
);

echo $localPackage->getUrl('/logo.png');
// 结果：file:///path/to/images/logo.png

echo $ftpPackage->getUrl('/logo.png');
// 结果：ftp://example.com/images/logo.png
```

## 了解更多

* [如何在 Symfony 应用程序中管理 CSS 和 JavaScript 资源](frontend.md)
* [WebLink 组件](web_link.md) 用于预加载资源和发送早期提示。

[Webpack]: https://webpack.js.org/
