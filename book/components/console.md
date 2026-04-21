# Console 组件

Console 组件简化了美观且可测试的命令行界面的创建。

Console 组件允许你创建命令行命令。你的控制台命令可用于任何重复性任务，例如定时任务、数据导入或其他批量处理工作。

## 安装

```terminal
$ composer require symfony/console
```

## 创建控制台应用程序

> **参见：** 本文介绍了如何在任意 PHP 应用程序中将 Console 功能作为独立组件使用。请阅读[控制台](console.md)文章，了解如何在 Symfony 应用程序中使用它。

首先，你需要创建一个 PHP 脚本来定义控制台应用程序：

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

然后，你可以使用 `Application::addCommand` 注册命令：

```php
// ...
$application->addCommand(new GenerateAdminCommand());
```

你也可以注册内联命令，并通过 `Command::setCode()` 方法定义其行为：

```php
// ...
$application->register('generate-admin')
    ->addArgument('username', InputArgument::REQUIRED)
    ->setCode(function (InputInterface $input, OutputInterface $output): int {
        // ...

        return Command::SUCCESS;
    });
```

这在创建[单命令应用程序](console/single_command_tool.md)时非常有用。

有关如何创建命令的信息，请参阅[控制台](console.md)文章。

## 了解更多

