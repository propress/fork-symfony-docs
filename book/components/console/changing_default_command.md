# 更改默认命令

当没有传递命令名称时，Console 组件将始终运行 `ListCommand`。为了更改默认命令，您需要将命令名称传递给 `setDefaultCommand()` 方法：

```php
namespace Acme\Console\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(name: 'hello:world', description: 'Outputs "Hello World"')]
class HelloWorldCommand extends Command
{
    public function __invoke(SymfonyStyle $io): int
    {
        $io->writeln('Hello World');

        return Command::SUCCESS;
    }
}
```

执行应用并更改默认命令：

```php
// application.php
use Acme\Console\Command\HelloWorldCommand;
use Symfony\Component\Console\Application;

$command = new HelloWorldCommand();
$application = new Application();
$application->add($command);
$application->setDefaultCommand($command->getName());
$application->run();
```

通过运行以下命令测试新的默认控制台命令：

```bash
$ php application.php
```

这将在命令行中打印以下内容：

```text
Hello World
```

> [!WARNING]
> 此功能有一个限制：您不能向默认命令传递任何参数或选项，因为它们会被忽略。

## 了解更多！

* [单命令工具](./single_command_tool.md)
