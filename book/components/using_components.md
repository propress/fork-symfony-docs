# 如何安装和使用 Symfony 组件

如果你正在创建一个新项目（或已有一个项目），并需要使用一个或多个组件，最简单的集成方式是使用 [Composer][]。Composer 足够智能，可以自动下载你所需的组件并处理自动加载，让你可以立即开始使用这些库。

本文将以使用 [Finder 组件](finder.md) 为例进行说明，但同样适用于任何其他组件的使用。

## 使用 Finder 组件

**1.** 如果你正在创建新项目，请为其创建一个新的空目录。

**2.** 打开终端，进入该目录，并使用 Composer 获取库。

```terminal
$ composer require symfony/finder
```

`symfony/finder` 这个名称写在你所需组件文档的顶部。

> **提示：** 如果你的系统上尚未安装 Composer，请[安装 Composer][Install Composer]。根据安装方式的不同，你的目录中可能会有一个 `composer.phar` 文件。在这种情况下，请不要担心！你的命令行应使用 `php composer.phar require symfony/finder`。

**3.** 编写你的代码！

一旦 Composer 下载了组件，你只需包含由 Composer 生成的 `vendor/autoload.php` 文件。该文件负责自动加载所有库，让你可以立即使用它们：

```php
// 项目结构示例：
// my_project/
//     data/
//         ...              # 一些项目数据
//     src/
//         my_script.php    # 主入口点
//     vendor/
//         autoload.php     # 由 Composer 生成的自动加载器
//         ...              # 由 Composer 下载的包

// 文件示例：src/my_script.php
// 相对于此 PHP 文件的自动加载器路径
require_once __DIR__.'/../vendor/autoload.php';

use Symfony\Component\Finder\Finder;

$finder = new Finder();
$finder->in('../data/');

// 其余的 PHP 代码...
```

## 接下来呢？

现在，组件已安装并自动加载。请阅读具体组件的文档，了解更多关于如何使用它的信息。

祝使用愉快！

[Composer]: https://getcomposer.org
[Install Composer]: https://getcomposer.org/download/
