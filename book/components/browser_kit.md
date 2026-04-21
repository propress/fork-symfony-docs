# BrowserKit 组件

> BrowserKit 组件模拟 Web 浏览器的行为，允许你以编程方式发出请求、点击链接和提交表单。

## 安装

```terminal
$ composer require symfony/browser-kit
```

## 基本用法

> **参见：** 本文介绍如何在任何 PHP 应用程序中将 BrowserKit 功能用作独立组件。请阅读 Symfony 功能测试文章，了解如何在 Symfony 应用程序中使用它。

### 创建客户端

该组件仅提供一个抽象客户端，不提供任何可用于 HTTP 层的后端。要创建自己的客户端，必须继承 `AbstractBrowser` 类并实现 `Symfony\Component\BrowserKit\AbstractBrowser::doRequest` 方法。该方法接受一个请求并应返回一个响应：

```php
namespace Acme;

use Symfony\Component\BrowserKit\AbstractBrowser;
use Symfony\Component\BrowserKit\Response;

class Client extends AbstractBrowser
{
    protected function doRequest($request): Response
    {
        // ... 将请求转换为响应

        return new Response($content, $status, $headers);
    }
}
```

对于基于 HTTP 层的简单浏览器实现，请查看该组件提供的 `Symfony\Component\BrowserKit\HttpBrowser`。对于基于 `HttpKernelInterface` 的实现，请查看 [HttpKernel 组件](components/http_kernel.md) 提供的 `Symfony\Component\HttpKernel\HttpClientKernel`。

### 发出请求

使用 `Symfony\Component\BrowserKit\AbstractBrowser::request` 方法发出 HTTP 请求。前两个参数是 HTTP 方法和请求的 URL：

```php
use Acme\Client;

$client = new Client();
$crawler = $client->request('GET', '/');
```

`request()` 方法返回的值是 `Symfony\Component\DomCrawler\Crawler` 类的实例，由 [DomCrawler 组件](components/dom_crawler.md) 提供，它允许以编程方式访问和遍历 HTML 元素。

> **注意：** 发出请求后，后续请求会使客户端重新启动内核。这会清除安全令牌、分离 Doctrine 实体等。详细了解在同一测试中发出多个请求。

`Symfony\Component\BrowserKit\AbstractBrowser::jsonRequest` 方法（与 `request()` 方法参数相同）是一个快捷方式，可以将请求参数编码为 JSON 字符串并设置所需的 HTTP 标头：

```php
use Acme\Client;

$client = new Client();
// 此方法将参数编码为 JSON，并设置所需的 CONTENT_TYPE 和 HTTP_ACCEPT 标头
$crawler = $client->jsonRequest('GET', '/', ['some_parameter' => 'some_value']);
```

`Symfony\Component\BrowserKit\AbstractBrowser::xmlHttpRequest` 方法（与 `request()` 方法参数相同）是发出 AJAX 请求的快捷方式：

```php
use Acme\Client;

$client = new Client();
// 自动添加所需的 HTTP_X_REQUESTED_WITH 标头
$crawler = $client->xmlHttpRequest('GET', '/');
```

### 包装响应内容

当获取不是有效独立文档的 HTML 片段时（例如 `<tr>` 或 `<td>` 元素），原生 HTML5 解析器可能会丢弃或重写这些元素，因为它们缺少适当的上下文。

为了解决这个问题，BrowserKit 允许在解析响应内容之前将其包装，使用 `Symfony\Component\BrowserKit\AbstractBrowser::wrapContent` 方法。

例如，当控制器返回如下 HTML 片段时：

```php
<tr><td>Cell content</td></tr>
```

直接解析可能会删除 `<tr>` 和 `<td>` 元素。你可以将片段包装在有效的父结构中：

```php
use Acme\Client;

$client = new Client();
$client->wrapContent('<table>%s</table>');

$crawler = $client->xmlHttpRequest('GET', '/fragment');

echo $crawler->html();
```

> **注意：** 包装模式必须包含一个 `%s` 占位符，该占位符将被原始响应内容替换。包装内容仅用于解析，不会修改实际的 HTTP 响应。

### 点击链接

`AbstractBrowser` 能够模拟点击链接。传递链接的文本内容，客户端将执行所需的 HTTP GET 请求来模拟链接点击：

```php
use Acme\Client;

$client = new Client();
$client->request('GET', '/product/123');

$crawler = $client->clickLink('Go elsewhere...');
```

如果你需要 `Symfony\Component\DomCrawler\Link` 对象（它提供对链接属性的访问，例如 `$link->getMethod()`、`$link->getUri()`），请使用另一个方法：

```php
// ...
$crawler = $client->request('GET', '/product/123');
$link = $crawler->selectLink('Go elsewhere...')->link();
$client->click($link);
```

`Symfony\Component\BrowserKit\AbstractBrowser::click` 和 `Symfony\Component\BrowserKit\AbstractBrowser::clickLink` 方法可以接受一个可选的 `serverParameters` 参数。该参数允许在点击链接时发送附加信息（如标头）：

```php
use Acme\Client;

$client = new Client();
$client->request('GET', '/product/123');

// 适用于 `click()`...
$link = $crawler->selectLink('Go elsewhere...')->link();
$client->click($link, ['X-Custom-Header' => 'Some data']);

// ... 以及 `clickLink()`
$crawler = $client->clickLink('Go elsewhere...', ['X-Custom-Header' => 'Some data']);
```

### 提交表单

`AbstractBrowser` 也能够提交表单。首先，使用任何一个按钮选择表单，然后在提交之前覆盖其任何属性（方法、字段值等）：

```php
use Acme\Client;

$client = new Client();
$crawler = $client->request('GET', 'https://github.com/login');

// 找到带有 'Log in' 按钮的表单并提交它
// 'Log in' 可以是 <button> 或 <input type="submit"> 的文本内容、id 或 name
$client->submitForm('Log in');

// 第二个可选参数允许覆盖默认表单字段值
$client->submitForm('Log in', [
    'login' => 'my_user',
    'password' => 'my_pass',
    // 要上传文件，值必须是绝对文件路径
    'file' => __FILE__,
]);

// 你也可以覆盖其他表单选项
$client->submitForm(
    'Log in',
    ['login' => 'my_user', 'password' => 'my_pass'],
    // 覆盖默认表单 HTTP 方法
    'PUT',
    // 覆盖某些 $_SERVER 参数（例如 HTTP 标头）
    ['HTTP_ACCEPT_LANGUAGE' => 'es']
);
```

如果你需要 `Symfony\Component\DomCrawler\Form` 对象（它提供对表单属性的访问，例如 `$form->getUri()`、`$form->getValues()`、`$form->getFields()`），请使用另一个方法：

```php
// ...

// 选择表单并填写一些值
$form = $crawler->selectButton('Log in')->form();
$form['login'] = 'symfonyfan';
$form['password'] = 'anypass';

// 提交该表单
$crawler = $client->submit($form);
```

### 自定义标头处理

传递给 `request()` 方法的可选 HTTP 标头遵循 FastCGI 请求格式（大写、使用下划线代替破折号，并以 `HTTP_` 为前缀）。在将这些标头保存到请求之前，它们会被小写化，去掉 `HTTP_` 前缀，并将下划线转换为破折号。

如果你向一个对标头大小写或标点符号有特殊规则的应用程序发出请求，请覆盖 `getHeaders()` 方法，该方法必须返回一个关联标头数组：

```php
protected function getHeaders(Request $request): array
{
    $headers = parent::getHeaders($request);
    if (isset($request->getServer()['api_key'])) {
        $headers['api_key'] = $request->getServer()['api_key'];
    }

    return $headers;
}
```

## Cookie

### 获取 Cookie

`AbstractBrowser` 实现通过 `Symfony\Component\BrowserKit\CookieJar` 公开 Cookie（如果有的话），它允许你在用客户端发出请求时存储和检索任何 Cookie：

```php
use Acme\Client;

// 发出请求
$client = new Client();
$crawler = $client->request('GET', '/');

// 获取 Cookie Jar
$cookieJar = $client->getCookieJar();

// 通过名称获取 Cookie
$cookie = $cookieJar->get('name_of_the_cookie');

// 获取 Cookie 数据
$name       = $cookie->getName();
$value      = $cookie->getValue();
$rawValue   = $cookie->getRawValue();
$isSecure   = $cookie->isSecure();
$isHttpOnly = $cookie->isHttpOnly();
$isExpired  = $cookie->isExpired();
$expires    = $cookie->getExpiresTime();
$path       = $cookie->getPath();
$domain     = $cookie->getDomain();
$sameSite   = $cookie->getSameSite();
```

> **注意：** 这些方法仅返回未过期的 Cookie。

### 遍历 Cookie

```php
use Acme\Client;

// 发出请求
$client = new Client();
$crawler = $client->request('GET', '/');

// 获取 Cookie Jar
$cookieJar = $client->getCookieJar();

// 获取包含所有 Cookie 的数组
$cookies = $cookieJar->all();
foreach ($cookies as $cookie) {
    // ...
}

// 获取所有值
$values = $cookieJar->allValues('http://symfony.com');
foreach ($values as $value) {
    // ...
}

// 获取所有原始值
$rawValues = $cookieJar->allRawValues('http://symfony.com');
foreach ($rawValues as $rawValue) {
    // ...
}
```

### 设置 Cookie

你还可以创建 Cookie 并将其添加到 Cookie Jar 中，然后注入到客户端构造函数中：

```php
use Acme\Client;

// 创建 Cookie 并添加到 Cookie Jar
$cookie = new Cookie('flavor', 'chocolate', strtotime('+1 day'));
$cookieJar = new CookieJar();
$cookieJar->set($cookie);

// 创建客户端并设置 Cookie
$client = new Client([], null, $cookieJar);
// ...
```

### 发送 Cookie

请求可以包含 Cookie。为此，使用 `Symfony\Component\BrowserKit\AbstractBrowser::request` 方法的 `serverParameters` 参数来设置 `Cookie` 标头值：

```php
$client->request('GET', '/', [], [], [
    'HTTP_COOKIE' => new Cookie('flavor', 'chocolate', strtotime('+1 day')),

    // 你也可以将 Cookie 内容作为字符串传递
    'HTTP_COOKIE' => 'flavor=chocolate; expires=Sat, 11 Feb 2023 12:18:13 GMT; Max-Age=86400; path=/'
]);
```

> **注意：** 使用 `serverParameters` 参数设置的所有 HTTP 标头必须以 `HTTP_` 为前缀。

## 历史记录

客户端存储所有请求，允许你在历史记录中前进和后退：

```php
use Acme\Client;

$client = new Client();
$client->request('GET', '/');

// 选择并点击链接
$link = $crawler->selectLink('Documentation')->link();
$client->click($link);

// 返回主页
$crawler = $client->back();

// 前进到文档页面
$crawler = $client->forward();

// 检查历史记录位置是否在第一页
if (!$client->getHistory()->isFirstPage()) {
    $crawler = $client->back();
}

// 检查历史记录位置是否在最后一页
if (!$client->getHistory()->isLastPage()) {
    $crawler = $client->forward();
}
```

你可以使用 `restart()` 方法删除客户端的历史记录。这也会删除所有 Cookie：

```php
use Acme\Client;

$client = new Client();
$client->request('GET', '/');

// 重置客户端（历史记录和 Cookie 也会被清除）
$client->restart();
```

## 发出外部 HTTP 请求

到目前为止，本文中的所有示例都假设你对自己的应用程序发出内部请求。但是，在向外部网站和应用程序发出 HTTP 请求时，你也可以运行完全相同的示例。

首先，安装并配置 [HttpClient 组件](http_client.md)。然后，使用 `Symfony\Component\BrowserKit\HttpBrowser` 创建将发出外部 HTTP 请求的客户端：

```php
use Symfony\Component\BrowserKit\HttpBrowser;
use Symfony\Component\HttpClient\HttpClient;

$browser = new HttpBrowser(HttpClient::create());
```

现在，你可以使用本文中所示的任何方法来提取信息、点击链接、提交表单等。这意味着你不再需要使用专用的 Web 爬虫或抓取器（如 [Goutte]）：

```php
$browser = new HttpBrowser(HttpClient::create());

$browser->request('GET', 'https://github.com');
$browser->clickLink('Sign in');
$browser->submitForm('Sign in', ['login' => '...', 'password' => '...']);
$openPullRequests = trim($browser->clickLink('Pull requests')->filter(
    '.table-list-header-toggle a:nth-child(1)'
)->text());
```

> **提示：** 你还可以使用 HTTP 客户端选项，如 `ciphers`、`auth_basic` 和 `query`。它们必须作为默认选项参数传递给 HTTP 浏览器使用的客户端。

### 处理 HTTP 响应

使用 BrowserKit 组件时，你可能需要处理所发出请求的响应。为此，请调用 `HttpBrowser` 对象的 `getResponse()` 方法。该方法返回浏览器收到的最后一个响应：

```php
$browser = new HttpBrowser(HttpClient::create());

$browser->request('GET', 'https://foo.com');
$response = $browser->getResponse();
```

如果你发出的请求产生 JSON 响应，可以使用 `toArray()` 方法将 JSON 文档转换为 PHP 数组，而无需显式调用 `json_decode()`：

```php
$browser = new HttpBrowser(HttpClient::create());

$browser->request('GET', 'https://api.foo.com');
$response = $browser->getResponse()->toArray();
// $response 是解码后 JSON 内容的 PHP 数组
```

## 了解更多

* [测试](testing.md)
* [CssSelector 组件](components/css_selector.md)
* [DomCrawler 组件](components/dom_crawler.md)

[Goutte]: https://github.com/FriendsOfPHP/Goutte
