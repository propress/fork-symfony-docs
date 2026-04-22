# Console 组件

Console 组件简化了美观且可测试的命令行接口的创建。

Console 组件允许您创建命令行命令。您的控制台命令可用于任何重复性任务，例如 cron 作业、导入或其他批处理作业。

## 安装

```bash
$ composer require symfony/console
```

如果您在 Symfony 应用之外使用此组件，则必须在代码中引入 Composer 生成的 `vendor/autoload.php` 文件来启用类自动加载机制。更多信息请阅读[此文章](./using_components.md)。

## 创建控制台应用

> [!NOTE]
> 本文介绍如何在任何 PHP 应用中将 Console 功能用作独立组件。阅读控制台文章以了解如何在 Symfony 应用中使用它。

首先，您需要创建一个 PHP 脚本来定义控制台应用：

```php
#!/usr/bin/env php
<?php
// application.php

require __DIR__.'/vendor/autoload.php';

use Symfony\Component\Console\Application;

$application = new Application();

// ... 注册命令

$application->run();
```

然后，您可以使用 `Symfony\Component\Console\Application::addCommand` 注册命令：

```php
// ...
$application->addCommand(new GenerateAdminCommand());
```

您还可以注册内联命令并使用 `Command::setCode()` 方法定义它们的行为：

```php
// ...
$application->register('generate-admin')
    ->addArgument('username', InputArgument::REQUIRED)
    ->setCode(function (InputInterface $input, OutputInterface $output): int {
        // ...

        return Command::SUCCESS;
    });
```

这在创建[单命令应用](./console/single_command_tool.md)时很有用。

有关如何创建命令的信息，请参阅控制台文章。

## 了解更多

* [控制台命令](../../console.md)
* [更改默认命令](./console/changing_default_command.md)
* [控制台参数](./console/console_arguments.md)
* [控制台事件](./console/events.md)
* [控制台辅助工具](./console/helpers/index.md)
* [日志记录器](./console/logger.md)
* [单命令工具](./console/single_command_tool.md)
* [控制台使用](./console/usage.md)
