# 构建单命令应用

在构建命令行工具时，您可能不需要提供多个命令。在这种情况下，每次都必须传递命令名称会很烦人。幸运的是，可以通过声明单命令应用来消除这种需要：

```php
#!/usr/bin/env php
<?php
require __DIR__.'/vendor/autoload.php';

use Symfony\Component\Console\Attribute\Argument;
use Symfony\Component\Console\Attribute\Option;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\SingleCommandApplication;

new SingleCommandApplication()
    ->setName('My Super Command') // 可选
    ->setVersion('1.0.0') // 可选
    ->setCode(function (OutputInterface $output, #[Argument] string $foo = 'The directory', #[Option] string $bar = ''): int {
        // 输出参数和选项

        return 0;
    })
    ->run();
```

您仍然可以像往常一样注册命令：

```php
#!/usr/bin/env php
<?php
require __DIR__.'/vendor/autoload.php';

use Acme\Command\DefaultCommand;
use Symfony\Component\Console\Application;

$application = new Application('echo', '1.0.0');
$command = new DefaultCommand();

$application->addCommand($command);

$application->setDefaultCommand($command->getName(), true);
$application->run();
```

`Symfony\Component\Console\Application::setDefaultCommand` 方法接受布尔值作为第二个参数。如果为 true，则命令 `echo` 将始终被使用，而无需传递其名称。
