# Asset 组件

Asset 组件管理 Web 资源（如 CSS 样式表、JavaScript 文件和图片文件）的 URL 生成和版本控制。

过去，Web 应用通常会硬编码 Web 资源的 URL。例如：

```html
<link rel="stylesheet" type="text/css" href="/css/main.css">

<!-- ... -->

<a href="/"><img src="/images/logo.png" alt="logo"></a>
```

除非 Web 应用极其简单，否则不再推荐这种做法。硬编码 URL 有以下缺点：

* **模板变得冗长**：您必须为每个资源编写完整路径。使用 Asset 组件时，可以将资源分组到包（packages）中，以避免重复其路径的公共部分；
* **版本控制困难**：每个应用都必须自定义管理。为资源 URL 添加版本（例如 `main.css?v=5`）对某些应用至关重要，因为它允许您控制资源的缓存方式。Asset 组件允许您为每个包定义不同的版本控制策略；
* **移动资源位置**繁琐且容易出错：需要仔细更新所有模板中包含的所有资源的 URL。Asset 组件允许您只需更改与资源包关联的基本路径值，即可轻松移动资源；
* **几乎不可能使用多个 CDN**：此技术要求您为每个请求随机更改资源的 URL。Asset 组件为任意数量的多个 CDN 提供开箱即用的支持，包括常规的（`http://`）和安全的（`https://`）。

## 安装

```bash
$ composer require symfony/asset
```

如果您在 Symfony 应用之外使用此组件，则必须在代码中引入 Composer 生成的 `vendor/autoload.php` 文件来启用类自动加载机制。更多信息请阅读[此文章](../components/using_components.md)。

## 用法

### 资源包（Asset Packages）

Asset 组件通过包（packages）管理资源。一个包将所有共享相同属性的资源分组：版本控制策略、基本路径、CDN 主机等。在以下基本示例中，创建了一个包来管理没有任何版本控制的资源：

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
返回资源的版本。

`Symfony\Component\Asset\PackageInterface::getUrl`  
返回绝对路径或根相对公共路径。

使用包，您可以：

A) 为资源[添加版本](#版本化资源)；  
B) 为资源设置[通用基本路径](#分组资源)（例如 `/css`）；  
C) 为资源[配置 CDN](#绝对资源和-cdn)。

### 版本化资源

Asset 组件的主要功能之一是能够管理应用资源的版本控制。资源版本通常用于控制这些资源的缓存方式。

Asset 组件不是依赖简单的版本机制，而是允许您通过 PHP 类定义高级版本控制策略。两个内置策略是 `Symfony\Component\Asset\VersionStrategy\EmptyVersionStrategy`（不向资源添加任何版本）和 `Symfony\Component\Asset\VersionStrategy\StaticVersionStrategy`（允许您使用格式字符串设置版本）。

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

如果您想修改版本格式，将 `sprintf` 兼容的格式字符串作为 `StaticVersionStrategy` 构造函数的第二个参数传递：

```php
// 在版本值之前放置 'version' 词
$package = new Package(new StaticVersionStrategy('v1', '%s?version=%s'));

echo $package->getUrl('/image.png');
// 结果：/image.png?version=v1

// 在路径之前放置资源版本
$package = new Package(new StaticVersionStrategy('v1', '%2$s/%1$s'));

echo $package->getUrl('/image.png');
// 结果：/v1/image.png

echo $package->getUrl('image.png');
// 结果：v1/image.png
```

#### JSON 文件清单（Manifest）

管理资源版本控制的一种流行策略是生成一个 JSON 文件，将所有源文件名映射到其相应的输出文件，这被 [Webpack][webpack] 等工具使用：

```json
{
    "css/app.css": "build/css/app.b916426ea1d10021f3f17ce8031f93c2.css",
    "js/app.js": "build/js/app.13630905267b809161e71d0f8a0c017b.js",
    "...": "..."
}
```

在这些情况下，使用 `Symfony\Component\Asset\VersionStrategy\JsonManifestVersionStrategy`：

```php
use Symfony\Component\Asset\Package;
use Symfony\Component\Asset\VersionStrategy\JsonManifestVersionStrategy;

// 假设上面的 JSON 文件称为 "rev-manifest.json"
$package = new Package(new JsonManifestVersionStrategy(__DIR__.'/rev-manifest.json'));

echo $package->getUrl('css/app.css');
// 结果：build/css/app.b916426ea1d10021f3f17ce8031f93c2.css
```

如果您请求的资源在 `rev-manifest.json` 文件中*未找到*，将返回原始的 - *未修改的* - 资源路径。`$strictMode` 参数有助于调试问题，因为当资源未在清单中列出时它会抛出异常：

```php
use Symfony\Component\Asset\Package;
use Symfony\Component\Asset\VersionStrategy\JsonManifestVersionStrategy;

// $strictMode 的值可以针对每个环境特定，调试时为 "true"，稳定时为 "false"。
$strictMode = true;
// 假设上面的 JSON 文件称为 "rev-manifest.json"
$package = new Package(new JsonManifestVersionStrategy(__DIR__.'/rev-manifest.json', null, $strictMode));

echo $package->getUrl('not-found.css');
// 错误：
```

如果您的 JSON 文件不在本地文件系统上，而是通过 HTTP 可访问，请将 `Symfony\Component\Asset\VersionStrategy\JsonManifestVersionStrategy` 与 [HttpClient 组件](../http_client.md)一起使用：

```php
use Symfony\Component\Asset\Package;
use Symfony\Component\Asset\VersionStrategy\JsonManifestVersionStrategy;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$manifestUrl = 'https://cdn.example.com/rev-manifest.json';
$package = new Package(new JsonManifestVersionStrategy($manifestUrl, $httpClient));
```

#### 自定义版本策略

使用 `Symfony\Component\Asset\VersionStrategy\VersionStrategyInterface` 定义您自己的版本控制策略。例如，您的应用可能需要将当前日期附加到所有 Web 资源，以便每天清除缓存：

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

通常，许多资源位于一个通用路径下（例如 `/static/images`）。如果是这种情况，请使用 `Symfony\Component\Asset\PathPackage` 类替换默认的 `Symfony\Component\Asset\Package` 类，以避免一遍又一遍地重复该路径：

```php
use Symfony\Component\Asset\PathPackage;
// ...

$pathPackage = new PathPackage('/static/images', new StaticVersionStrategy('v1'));

echo $pathPackage->getUrl('logo.png');
// 结果：/static/images/logo.png?v1

// 使用绝对路径时，基本路径被忽略
echo $pathPackage->getUrl('/logo.png');
// 结果：/logo.png?v1
```

#### 请求上下文感知资源

如果您在项目中也使用 [HttpFoundation](./http_foundation.md) 组件（例如，在 Symfony 应用中），`PathPackage` 类可以考虑当前请求的上下文：

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

// 使用资源的绝对路径时，"基本路径"和"基本 URL"都被忽略
echo $pathPackage->getUrl('/logo.png');
// 结果：/logo.png?v1
```

现在设置了请求上下文，`PathPackage` 将在前面添加当前请求的基本 URL。因此，例如，如果您的整个站点托管在 Web 服务器根目录的 `/somewhere` 目录下，并且配置的基本路径是 `/static/images`，则所有路径都将以 `/somewhere/static/images` 为前缀。

### 绝对资源和 CDN

在不同域和 CDN（*内容分发网络*）上托管资源的应用应使用 `Symfony\Component\Asset\UrlPackage` 类为其资源生成绝对 URL：

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

您还可以传递与协议无关的 URL：

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

这很有用，因为如果访问者通过 HTTPS 查看您的站点，资源将自动通过 HTTPS 请求。如果要使用此功能，请确保您的 CDN 主机支持 HTTPS。

如果您从多个域提供资源以提高应用性能，请将 URL 数组作为第一个参数传递给 `UrlPackage` 构造函数：

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

对于每个资源，将随机使用其中一个 URL。但是，选择是确定性的，这意味着每个资源将始终由同一域提供。此行为简化了 HTTP 缓存的管理。

#### 请求上下文感知资源

与应用相对资源类似，绝对资源也可以考虑当前请求的上下文。在这种情况下，只考虑请求协议，以便选择适当的基本 URL（HTTPS 请求使用 HTTPS 或协议相对 URL，HTTP 请求使用任何基本 URL）：

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
// 假设 RequestStackContext 表示我们在安全主机上
// 结果：https://example.com/logo.png?v1
```

### 命名包（Named Packages）

管理大量不同资源的应用可能需要将它们分组到具有相同版本控制策略和基本路径的包中。Asset 组件包含一个 `Symfony\Component\Asset\Packages` 类来简化多个包的管理。

在以下示例中，所有包使用相同的版本控制策略，但它们都有不同的基本路径：

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

`Packages` 类允许您定义一个默认包，该包将应用于未定义要使用的包名称的资源。此外，此应用定义了一个名为 `img` 的包，用于从外部域提供图像，以及一个 `doc` 包，以避免在模板中链接到文档时重复长路径：

```php
echo $packages->getUrl('/main.css');
// 结果：/main.css?v1

echo $packages->getUrl('/logo.png', 'img');
// 结果：https://img.example.com/logo.png?v1

echo $packages->getUrl('resume.pdf', 'doc');
// 结果：/somewhere/deep/for/documents/resume.pdf?v1
```

### 本地文件和其他协议

除了 HTTP，此组件还支持其他协议（如 `file://` 和 `ftp://`）。例如，这允许提供本地文件以提高性能：

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

* [如何在 Symfony 应用中管理 CSS 和 JavaScript 资源](../frontend.md)
* [WebLink 组件](../web_link.md)用于预加载资源和发送早期提示。

[webpack]: https://webpack.js.org/
