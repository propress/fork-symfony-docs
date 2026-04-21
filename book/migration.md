# 将现有应用迁移到 Symfony

当你有一个不是用 Symfony 构建的现有应用时，你可能想要迁移该应用的某些部分，而无需完全重写现有逻辑。对于这些情况，有一种称为[绞杀者无花果应用（Strangler Fig Application）](https://martinfowler.com/bliki/StranglerFigApplication.html)的模式。这种模式的基本思想是创建一个新应用，逐渐接管现有应用的功能。

---

## 前提条件

在开始将 Symfony 引入现有应用之前，你必须确保现有应用和环境满足某些要求。

### 选择目标 Symfony 版本

你需要决定目标迁移到哪个版本，无论是当前稳定版还是长期支持版（LTS）。使用 `check:requirements` 命令检查你的服务器是否满足运行 Symfony 应用的[技术要求](setup.md)。

### 设置 Composer

你需要注意两个应用之间的依赖冲突。一种确保兼容性的好方法是为两个项目使用相同的 `composer.json`：

```php
require __DIR__.'/vendor/autoload.php';
```

### 从旧应用中删除全局状态

在较旧的 PHP 应用中，依赖全局状态很常见，这可能对新引入的 Symfony 应用产生副作用。依赖全局变量的旧应用代码应该重构，以允许两个系统同时工作。

---

## 建立回归安全网

在安全地对现有代码进行更改之前，你必须确保不会破坏任何内容。最好的方法是建立自动化测试。

端到端测试（因为它们覆盖从用户在浏览器中看到的整个应用到正在运行的代码和连接的服务）比单元测试更适合迁移场景。可以使用 [Symfony Panther](https://github.com/symfony/panther) 等工具，或者在新 Symfony 应用设置完成后编写[功能测试](testing.md)。

---

## 将 Symfony 引入现有应用

### 在前端控制器中启动 Symfony

大多数现代框架提供所谓的前端控制器，作为每个请求的入口点。Symfony 也采用这种方式，`public/index.php` 作为前端控制器。

你需要更新你的 Web 服务器（例如 Apache 或 Nginx）以始终使用此前端控制器。使用 Apache 重写规则示例：

```apache
RewriteEngine On

RewriteCond %{REQUEST_URI}::$1 ^(/.+)/(.*)::\2$
RewriteRule ^(.*) - [E=BASE:%1]

RewriteCond %{ENV:REDIRECT_STATUS} ^$
RewriteRule ^index\.php(?:/(.*)|$) %{ENV:BASE}/$1 [R=301,L]

RewriteRule ^index\.php - [L]

RewriteCond %{REQUEST_FILENAME} -f
RewriteCond %{REQUEST_FILENAME} !^.+\.php$
RewriteRule ^ - [L]

RewriteRule ^ %{ENV:BASE}/index.php [L]
```

有两种常用的迁移方法：

- **带旧桥的前端控制器**：旧应用保持不变，允许分阶段将其迁移到 Symfony 应用。
- **旧路由加载器**：旧应用分阶段集成到 Symfony 中，最终结果是完全集成。

### 带旧桥的前端控制器

一旦你有了接管所有请求的 Symfony 应用，通过扩展原始前端控制器脚本即可回退到旧应用：

```php
// public/index.php
use App\Kernel;
use App\LegacyBridge;
use Symfony\Component\HttpFoundation\Request;

require dirname(__DIR__).'/vendor/autoload.php';

global $kernel;

$kernel = new Kernel($_SERVER['APP_ENV'], (bool) $_SERVER['APP_DEBUG']);
$request = Request::createFromGlobals();
$response = $kernel->handle($request);

if (false === $response->isNotFound()) {
    // Symfony 成功处理了路由
    $response->send();
} else {
    LegacyBridge::handleRequest($request, $response, __DIR__);
}

$kernel->terminate($request, $response);
```

旧桥的基本实现：

```php
// src/LegacyBridge.php
namespace App;

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

class LegacyBridge
{
    public static function getLegacyScript(Request $request): string
    {
        $requestPathInfo = $request->getPathInfo();
        $legacyRoot = __DIR__ . '/../';

        // 将路由映射到旧脚本：
        if ($requestPathInfo == '/customer/') {
            return "{$legacyRoot}src/customers/list.php";
        }

        // ...

        throw new \Exception("Unhandled legacy mapping for $requestPathInfo");
    }

    public static function handleRequest(Request $request, Response $response, string $publicDirectory): void
    {
        $legacyScriptFilename = LegacyBridge::getLegacyScript($request);

        $_SERVER['PHP_SELF'] = $request->getPathInfo();
        $_SERVER['SCRIPT_NAME'] = $request->getPathInfo();
        $_SERVER['SCRIPT_FILENAME'] = $legacyScriptFilename;

        require $legacyScriptFilename;
    }
}
```

### 旧路由加载器

旧路由加载器是一个[自定义路由加载器](routing/custom_route_loader.md)，在 Symfony 的路由组件中注册：

```php
// src/Legacy/LegacyRouteLoader.php
namespace App\Legacy;

use Symfony\Component\Config\Loader\Loader;
use Symfony\Component\Routing\Route;
use Symfony\Component\Routing\RouteCollection;

class LegacyRouteLoader extends Loader
{
    public function load($resource, $type = null): RouteCollection
    {
        $collection = new RouteCollection();
        // ... 从旧脚本文件生成路由
        return $collection;
    }
}
```

处理这些旧路由的控制器：

```php
// src/Controller/LegacyController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\StreamedResponse;

class LegacyController
{
    public function loadLegacyScript(string $requestPath, string $legacyScript): StreamedResponse
    {
        return new StreamedResponse(
            function () use ($requestPath, $legacyScript): void {
                $_SERVER['PHP_SELF'] = $requestPath;
                $_SERVER['SCRIPT_NAME'] = $requestPath;
                $_SERVER['SCRIPT_FILENAME'] = $legacyScript;

                chdir(dirname($legacyScript));

                require $legacyScript;
            }
        );
    }
}
```

由于旧代码现在在控制器动作内部运行，你可以访问新 Symfony 应用的许多功能，包括使用 Symfony 的事件生命周期。例如，这允许你使用安全组件及其防火墙将旧应用的身份验证和授权过渡到 Symfony 应用。
