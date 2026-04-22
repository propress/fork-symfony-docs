# 如何安装和使用 Symfony 组件

如果您正在启动一个新项目（或已经有一个项目），该项目将使用一个或多个组件，集成所有内容的最简单方法是使用 [Composer][composer]。Composer 足够智能，可以下载您需要的组件并负责自动加载，以便您可以立即开始使用这些库。

本文将带您完成使用 Finder 组件的过程，尽管这适用于使用任何组件。

## 使用 Finder 组件

**1.** 如果您正在创建一个新项目，请为其创建一个新的空目录。

**2.** 打开终端，进入此目录并使用 Composer 获取库。

```bash
$ composer require symfony/finder
```

名称 `symfony/finder` 写在您想要的任何组件的文档顶部。

> [!TIP]
> 如果您的系统上还没有 Composer，请[安装 Composer][install-composer]。根据您的安装方式，您最终可能会在目录中找到一个 `composer.phar` 文件。在这种情况下，没问题！在这种情况下，您的命令行是 `php composer.phar require symfony/finder`。

**3.** 编写您的代码！

一旦 Composer 下载了组件，您所需要做的就是引入 Composer 生成的 `vendor/autoload.php` 文件。此文件负责自动加载所有库，以便您可以立即使用它们：

```php
// 项目结构示例：
// my_project/
//     data/
//         ...              # 一些项目数据
//     src/
//         my_script.php    # 主入口点
//     vendor/
//         autoload.php     # Composer 生成的自动加载器
//         ...              # Composer 下载的包

// 文件示例：src/my_script.php
// 相对于此 PHP 文件的自动加载器路径
require_once __DIR__.'/../vendor/autoload.php';

use Symfony\Component\Finder\Finder;

$finder = new Finder();
$finder->in('../data/');

// 其余的 PHP 代码...
```

## 接下来做什么？

现在，组件已安装并自动加载。阅读特定组件的文档以了解有关如何使用它的更多信息。

祝您玩得开心！

[composer]: https://getcomposer.org
[install-composer]: https://getcomposer.org/download/
