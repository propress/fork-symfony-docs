# 在 Symfony 中创建你的第一个页面

创建新页面（无论是 HTML 页面还是 JSON 端点）是一个两步过程：

1. **创建控制器**：控制器是你编写的用于构建页面的 PHP 函数。它接收传入的请求信息并使用它创建 Symfony 的 `Response` 对象。
2. **创建路由**：路由是页面的 URL（例如 `/about`），指向一个控制器。

---

## 创建页面：路由和控制器

假设你想创建一个页面 `/lucky/number`，生成一个幸运（随机）数字并打印出来。为此，创建一个控制器类和方法：

```php
<?php
// src/Controller/LuckyController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class LuckyController
{
    #[Route('/lucky/number')]
    public function number(): Response
    {
        $number = random_int(0, 100);

        return new Response(
            '<html><body>Lucky number: '.$number.'</body></html>'
        );
    }
}
```

如果你使用 Symfony web server，在浏览器访问：http://localhost:8000/lucky/number

创建页面的两个步骤：

1. **创建控制器和方法**：这是你构建页面并最终返回 `Response` 对象的函数。
2. **创建路由**：路由定义了页面的 URL 路径以及要调用的控制器方法。

---

## bin/console 命令

你的项目已经内置了一个强大的调试工具：`bin/console` 命令。

```terminal
$ php bin/console
```

列出系统中所有路由：

```terminal
$ php bin/console debug:router
```

---

## Web 调试工具栏：调试利器

Symfony 的一个神奇功能是 Web 调试工具栏：在开发时显示在页面底部的信息栏，包含大量调试信息。

悬停并点击不同图标，可以获取关于路由、性能、日志等方面的信息。

---

## 渲染模板

如果要从控制器返回 HTML，可以渲染 Twig 模板：

```terminal
$ composer require twig
```

```php
// src/Controller/LuckyController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;

class LuckyController extends AbstractController
{
    #[Route('/lucky/number')]
    public function number(): Response
    {
        $number = random_int(0, 100);

        return $this->render('lucky/number.html.twig', [
            'number' => $number,
        ]);
    }
}
```

创建模板文件 `templates/lucky/number.html.twig`：

```twig
{# templates/lucky/number.html.twig #}
<h1>Your lucky number is {{ number }}</h1>
```

---

## 查看项目结构

主要目录：

- `config/` - 配置文件。包含路由、服务和包的配置。
- `src/` - 所有 PHP 代码
- `templates/` - 所有 Twig 模板

其他目录：

- `bin/` - `bin/console` 文件等可执行文件
- `var/` - 自动创建的文件，如缓存（`var/cache/`）和日志（`var/log/`）
- `vendor/` - 通过 Composer 下载的第三方库
- `public/` - 项目的文档根目录，公开可访问的文件放在这里

---

## 下一步

掌握基础知识后，可以继续学习：

- [路由](routing.md)
- [控制器](controller.md)
- [模板](templates.md)
- [前端](frontend.md)
- [配置](configuration.md)

以及其他重要主题，如[服务容器](service_container.md)、[表单系统](forms.md)、[Doctrine](doctrine.md) 等。
