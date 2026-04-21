# Profiler（性能分析器）

Profiler 是一个强大的**开发工具**，提供有关任何请求执行的详细信息。

> **危险**：**绝不要**在生产环境中启用 Profiler，因为这会在你的项目中造成重大安全漏洞。

## 安装

```terminal
$ composer require --dev symfony/profiler-pack
```

安装后，在开发环境中浏览应用的任何页面，让 Profiler 收集信息。然后点击注入到页面底部的调试工具栏中的任何元素，即可打开 Symfony Profiler 的 Web 界面。

> **注意**：调试工具栏只注入 HTML 响应。对于其他类型的内容（如 API 请求的 JSON 响应），Profiler URL 在 `X-Debug-Token-Link` HTTP 响应头中可用。访问 `/_profiler` 查看所有分析记录。

---

## 以编程方式访问分析数据

当响应对象可用时，使用 `loadProfileFromResponse` 方法访问关联的分析记录：

```php
// $profiler 是 'profiler' 服务
$profile = $profiler->loadProfileFromResponse($response);
```

使用令牌访问任何过去响应的分析记录：

```php
$token = $response->headers->get('X-Debug-Token');
$profile = $profiler->loadProfile($token);
```

查找令牌：

```php
// 获取最新的 10 个令牌
$tokens = $profiler->find('', '', 10, '', '', '');

// 获取所有包含 /admin/ 的 URL 的最新 10 个令牌
$tokens = $profiler->find('', '/admin/', 10, '', '', '');

// 获取本地 POST 请求的最新 10 个令牌
$tokens = $profiler->find('127.0.0.1', '', 10, 'POST', '', '');
```

---

## 数据收集器

Profiler 使用称为"数据收集器"的服务获取信息。列出应用中启用的所有收集器：

```terminal
$ php bin/console debug:container --tag=data_collector
```

---

## 以编程方式或有条件地启用 Profiler

```php
use Symfony\Component\HttpKernel\Profiler\Profiler;

class DefaultController
{
    public function someMethod(?Profiler $profiler): Response
    {
        if (null !== $profiler) {
            // 为此特定控制器操作禁用 Profiler
            $profiler->disable();
        }

        // ...
    }
}
```

### 条件性启用 Profiler

```yaml
# config/packages/web_profiler.yaml
when@dev:
    framework:
        profiler:
            collect: false
            collect_parameter: 'profile'
```

此配置默认禁用 Profiler，但对包含名为 `profile` 的查询参数的请求启用它。

---

## AJAX 请求后更新 Web 调试工具栏

```yaml
# config/packages/web_profiler.yaml
web_profiler:
    toolbar:
        ajax_replace: true
```

---

## 创建数据收集器

实现 `DataCollectorInterface` 来创建自定义数据收集器，存储应用生成的任何数据并在调试工具栏和 Profiler Web 界面中显示。
