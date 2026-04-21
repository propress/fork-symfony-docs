# 控制台命令

Symfony 框架通过 `bin/console` 脚本提供了大量命令（例如广为人知的 `bin/console cache:clear` 命令）。这些命令是使用 [Console 组件](components/console.md)创建的。你也可以使用它来创建自己的命令。

---

## 运行命令

每个 Symfony 应用都附带了大量命令。你可以使用 `list` 命令查看应用中所有可用的命令：

```terminal
$ php bin/console list
...

Available commands:
  about             Display information about the current project
  completion        Dump the shell completion script
  help              Display help for a command
  list              List commands
 assets
  assets:install    Install bundle's web assets under a public directory
 cache
  cache:clear       Clear the cache
...
```

> **注意**
>
> `list` 是默认命令，因此运行 `php bin/console` 效果相同。

找到所需命令后，可以使用 `--help` 选项运行它以查看命令文档：

```terminal
$ php bin/console assets:install --help
```

> **注意**
>
> `--help` 是 Console 组件内置的全局选项之一，对所有命令（包括你创建的命令）都可用。要了解更多，请参阅[控制台全局选项](#全局选项)。

### APP_ENV 和 APP_DEBUG

控制台命令在 `.env` 文件的 `APP_ENV` 变量定义的[环境](configuration.md#配置环境)中运行，默认为 `dev`。它还读取 `APP_DEBUG` 值以开启或关闭"调试"模式（默认为 `1`，即开启）。

要在另一个环境或调试模式下运行命令，请编辑 `APP_ENV` 和 `APP_DEBUG` 的值。你也可以在运行命令时定义这些环境变量，例如：

```terminal
# 为 prod 环境清除缓存
$ APP_ENV=prod php bin/console cache:clear
```

### 控制台补全 {#console-completion-setup}

如果你使用 Bash、Zsh 或 Fish shell，可以安装 Symfony 的补全脚本，以便在终端中输入命令时获得自动补全。所有命令支持名称和选项补全，某些命令甚至可以补全值。

![终端补全命令名称 "secrets:remove" 和参数 "SOME_OTHER_SECRET"。](../_images/components/console/completion.gif)

首先，你需要*一次性*安装补全脚本。运行 `bin/console completion --help` 获取适用于你的 shell 的安装说明。

> **注意**
>
> 使用 Bash 时，请确保已为你的操作系统安装并设置了"bash completion"包（通常名为 `bash-completion`）。

安装并重启终端后，你就可以使用补全了（默认情况下按 Tab 键）。

> **提示**
>
> 许多 PHP 工具都是使用 Symfony Console 组件构建的（例如 Composer、PHPStan 和 Behat）。如果它们使用的是 5.4 或更高版本，你也可以安装它们的补全脚本来启用控制台补全：
>
> ```terminal
> $ php vendor/bin/phpstan completion --help
> $ composer completion --help
> ```

> **提示**
>
> 如果你使用 [Symfony CLI](setup/symfony_cli.md) 工具，请按照[这些说明](setup/symfony_cli.md#启用自动补全)启用自动补全。

---

## 创建命令 {#console_creating-command}

命令在类中定义，并使用 `#[AsCommand]` 属性自动注册。例如，你可能想要一个创建用户的命令：

```php
// src/Command/CreateUserCommand.php
namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;

// 命令名称是用户在 "php bin/console" 之后输入的内容
#[AsCommand(name: 'app:create-user')]
class CreateUserCommand
{
    public function __invoke(): int
    {
        // ... 在这里放置创建用户的代码

        // 此方法必须返回一个整数，作为命令的"退出状态码"
        // 你也可以使用这些常量使代码更易读

        // 如果命令运行没有问题则返回此值
        // （等同于返回 int(0)）
        return Command::SUCCESS;

        // 或者如果执行期间发生错误则返回此值
        // （等同于返回 int(1)）
        // return Command::FAILURE;

        // 或者返回此值以表示命令使用不正确；例如无效的选项或缺少参数
        // （等同于返回 int(2)）
        // return Command::INVALID
    }
}
```

如果无法使用 PHP 属性，请将命令注册为服务并用 `console.command` [标签](service_container/tags.md)标记。如果你使用[默认的 services.yaml 配置](service_container.md#服务容器服务加载示例)，由于[自动配置](service_container.md#服务自动配置)，这已经为你完成了。

你还可以使用 `#[AsCommand]` 添加描述、使用示例和更长的帮助文本：

```php
#[AsCommand(
    name: 'app:create-user',
    // 运行 "php bin/console list" 时显示此简短描述
    description: 'Creates a new user.',
    // 运行带 "--help" 选项的命令时显示此内容
    help: 'This command allows you to create a user...',
    // 允许你显示一个或多个使用示例（无需添加命令名）
    usages: ['bob', 'alice --as-admin'],
)]
class CreateUserCommand
{
    public function __invoke(): int
    {
        // ...
    }
}
```

此外，你可以继承 `Command` 类来利用高级功能，如生命周期钩子（例如 `initialize` 和 `interact`）：

```php
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[AsCommand(name: 'app:create-user')]
class CreateUserCommand extends Command
{
    public function initialize(InputInterface $input, OutputInterface $output): void
    {
        // ...
    }

    public function interact(InputInterface $input, OutputInterface $output): void
    {
        // ...
    }

    public function __invoke(): int
    {
        // ...
    }
}
```

### 运行命令

配置并注册命令后，可以在终端中运行它：

```terminal
$ php bin/console app:create-user
```

由于你还没有编写任何逻辑，此命令不会做任何事情。在 `__invoke()` 方法中添加你自己的逻辑。

### 命令别名 {#command-aliases}

你可以使用管道（`|`）分隔符直接在名称中为命令定义备用名称（别名）。列表中的第一个名称成为实际的命令名称；其他名称是也可用于运行命令的别名：

```php
// src/Command/CreateUserCommand.php
namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;

#[AsCommand(
    name: 'app:create-user|app:add-user|app:new-user',
    description: 'Creates a new user.',
)]
class CreateUserCommand extends Command
{
    // ...
}
```

---

## 控制台输出

`__invoke()` 方法可以访问输出流以向控制台写入消息：

```php
// ...
public function __invoke(OutputInterface $output): int
{
    // 向控制台输出多行（每行末尾添加 "\n"）
    $output->writeln([
        'User Creator',
        '============',
        '',
    ]);

    // someMethod() 返回的值可以是生成并返回带 'yield' 关键字的消息的迭代器
    $output->writeln($this->someMethod());

    // 输出一条消息后跟一个 "\n"
    $output->writeln('Whoa!');

    // 输出一条消息，行末不添加 "\n"
    $output->write('You are about to ');
    $output->write('create a user.');

    return Command::SUCCESS;
}
```

现在，尝试执行命令：

```terminal
$ php bin/console app:create-user
User Creator
============

Whoa!
You are about to create a user.
```

### 输出部分 {#console-output-sections}

常规控制台输出可以分为多个独立区域，称为"输出部分"。当需要清除和覆盖输出信息时，可以创建一个或多个这些部分。

部分通过 `ConsoleOutput::section()` 方法创建，该方法返回 `ConsoleSectionOutput` 的实例：

```php
// ...
use Symfony\Component\Console\Output\ConsoleOutputInterface;

#[AsCommand(name: 'app:my-command')]
class MyCommand
{
    public function __invoke(OutputInterface $output): int
    {
        if (!$output instanceof ConsoleOutputInterface) {
            throw new \LogicException('This command accepts only an instance of "ConsoleOutputInterface".');
        }

        $section1 = $output->section();
        $section2 = $output->section();

        $section1->writeln('Hello');
        $section2->writeln('World!');
        sleep(1);
        // 输出显示 "Hello\nWorld!\n"

        // overwrite() 将所有现有的部分内容替换为给定的内容
        $section1->overwrite('Goodbye');
        sleep(1);
        // 输出现在显示 "Goodbye\nWorld!\n"

        // clear() 删除所有部分内容...
        $section2->clear();
        sleep(1);
        // 输出现在显示 "Goodbye\n"

        // ...但你也可以删除给定数量的行
        // （此示例删除部分的最后两行）
        $section1->clear(2);
        sleep(1);
        // 输出现在完全为空！

        // 设置部分的最大高度将使新行替换旧行
        $section1->setMaxHeight(2);
        $section1->writeln('Line1');
        $section1->writeln('Line2');
        $section1->writeln('Line3');

        return Command::SUCCESS;
    }
}
```

> **注意**
>
> 在部分中显示信息时会自动追加新行。

输出部分允许你以高级方式操作控制台输出，例如[显示多个独立更新的进度条](components/console/helpers/progressbar.md)和[向已渲染的表格追加行](components/console/helpers/table.md)。

> **警告**
>
> 终端只允许覆盖可见内容，因此在尝试写入/覆盖部分内容时，必须考虑到控制台的高度。

---

## 控制台输入

使用输入选项或参数向命令传递信息：

```php
use Symfony\Component\Console\Attribute\Argument;

// #[Argument] 属性将 $username 配置为必填的输入参数，
// 其值会自动传递给此参数
public function __invoke(#[Argument('The username of the user.')] string $username, OutputInterface $output): int
{
    $output->writeln([
        'User Creator',
        '============',
        '',
    ]);

    $output->writeln('Username: '.$username);

    return Command::SUCCESS;
}
```

现在，你可以将用户名传递给命令：

```terminal
$ php bin/console app:create-user Wouter
User Creator
============

Username: Wouter
```

> **另请参阅**
>
> 阅读[控制台输入](console/input.md)以了解更多关于控制台选项和参数的信息。

---

## 从服务容器获取服务

要实际创建新用户，命令需要访问某些[服务](service_container.md)。由于你的命令已注册为服务，你可以使用普通的依赖注入。假设你有一个想要访问的 `App\Service\UserManager` 服务：

```php
// ...
use App\Service\UserManager;
use Symfony\Component\Console\Attribute\Argument;
use Symfony\Component\Console\Attribute\AsCommand;

#[AsCommand(name: 'app:create-user')]
class CreateUserCommand
{
    public function __construct(
        private UserManager $userManager
    ) {
    }

    public function __invoke(#[Argument] string $username, OutputInterface $output): int
    {
        // ...

        $this->userManager->create($username);

        $output->writeln('User successfully generated!');

        return Command::SUCCESS;
    }
}
```

---

## 命令生命周期

命令有三个生命周期方法，在运行命令时被调用：

**`initialize`**（可选）
此方法在 `interact()` 和 `execute()` 方法之前执行。其主要目的是初始化在命令方法其余部分使用的变量。

**`interact`**（可选）
此方法在 `initialize()` 之后、`execute()` 之前执行。其目的是检查某些选项/参数是否缺失，并以交互方式向用户询问这些值。这是你可以询问缺少的必填选项/参数的最后地方。此方法在验证输入之前调用。请注意，当命令在无交互模式下运行时（例如传递 `--no-interaction` 全局选项标志时），不会调用此方法。

**`__invoke()`（或 `execute`）**（必填）
此方法在 `interact()` 和 `initialize()` 之后执行。它包含你希望命令执行的逻辑，必须返回一个整数，该整数将用作命令的[退出状态码](https://en.wikipedia.org/wiki/Exit_status)。

---

## 测试命令 {#console-testing-commands}

Symfony 提供了多种工具来帮助你测试命令。最有用的是 `CommandTester` 类。它使用特殊的输入和输出类，便于在没有真实控制台的情况下进行测试：

```php
// tests/Command/CreateUserCommandTest.php
namespace App\Tests\Command;

use Symfony\Bundle\FrameworkBundle\Console\Application;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;
use Symfony\Component\Console\Tester\CommandTester;

class CreateUserCommandTest extends KernelTestCase
{
    public function testExecute(): void
    {
        self::bootKernel();
        $application = new Application(self::$kernel);

        $command = $application->find('app:create-user');
        $commandTester = new CommandTester($command);
        $commandTester->execute([
            // 向辅助工具传递参数
            'username' => 'Wouter',

            // 传递选项时在键前加两个横线，
            // 例如：'--some-option' => 'option_value',
            // 测试数组值时使用括号，
            // 例如：'--some-option' => ['option_value'],
        ]);

        $commandTester->assertCommandIsSuccessful();

        // 控制台中命令的输出
        $output = $commandTester->getDisplay();
        $this->assertStringContainsString('Username: Wouter', $output);

        // ...
    }
}
```

> **注意**
>
> 如果你使用[单命令应用](components/console/single_command_tool.md)，请在应用上调用 `setAutoExit(false)` 以在 `CommandTester` 中获取命令结果。

> **警告**
>
> 使用 `CommandTester` 类测试命令时，不会分发控制台事件。如果需要测试这些事件，请改用 `ApplicationTester`。

> **警告**
>
> 测试 `InputOption::VALUE_NONE` 命令选项时，必须向其传递 `true`：
>
> ```php
> $commandTester = new CommandTester($command);
> $commandTester->execute(['--some-option' => true]);
> ```

测试命令时，了解命令在不同设置下（如终端的宽度和高度，甚至使用的颜色模式）的反应会很有用。你可以通过 `Terminal` 类访问此类信息：

```php
use Symfony\Component\Console\Terminal;

$terminal = new Terminal();

// 获取可用行数
$height = $terminal->getHeight();

// 获取可用列数
$width = $terminal->getWidth();

// 获取颜色模式
$colorMode = $terminal->getColorMode();

// 更改颜色模式
$colorMode = $terminal->setColorMode(AnsiColorMode::Ansi24);
```

### 测试控制台应用

除了测试单个命令，你还可以使用 `ApplicationTester` 类测试完整的控制台应用：

```php
use Symfony\Bundle\FrameworkBundle\Console\Application;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;
use Symfony\Component\Console\Tester\ApplicationTester;

class WelcomeCommandTest extends KernelTestCase
{
    public function testPerson(): void
    {
        self::bootKernel();
        $application = new Application(self::$kernel);
        // 这是关键：运行命令后不终止 PHP 进程
        $application->setAutoExit(false);

        $applicationTester = new ApplicationTester($application);
        $applicationTester->run([
            'command' => 'app:welcome-person',
            'firstName' => 'Jane',
            'lastName' => 'Smith',
            'hobbies' => ['reading', 'dancing']
        ]);

        $applicationTester->assertCommandIsSuccessful();

        $output = $applicationTester->getDisplay();
        $this->assertStringContainsString('Jane Smith', $output);
        $this->assertStringContainsString('reading and dancing', $output);
    }
}
```

> **注意**
>
> 不要忘记在将应用传递给 `ApplicationTester` 之前调用 `setAutoExit(false)`。没有它，应用会在运行命令后调用 `exit()`，这会终止 PHPUnit 进程。

---

## 记录命令错误

每当运行命令时抛出异常，Symfony 都会为其添加一条日志消息，包括完整的失败命令。此外，Symfony 注册了一个[事件订阅者](event_dispatcher.md)来监听 [ConsoleEvents::TERMINATE 事件](components/console/events.md)，并在命令没有以 `0` [退出状态码](https://en.wikipedia.org/wiki/Exit_status)结束时添加日志消息。

---

## 使用事件和处理信号

当命令运行时，会分发许多事件，其中一个允许你响应信号，在[这一部分](components/console/events.md)阅读更多内容。

---

## 分析命令

Symfony 允许你分析任何命令（包括你自己的）的执行情况。首先，确保启用了[调试模式](configuration.md#配置环境)和[分析器](profiler.md)。然后，运行命令时添加 `--profile` 选项：

```terminal
$ php bin/console --profile app:my-command
```

Symfony 现在将收集关于命令执行的数据，这有助于调试错误或检查其他问题。命令执行结束后，可以通过分析器的 Web 页面访问配置文件。

> **提示**
>
> 如果你在详细模式下运行命令（添加 `-v` 选项），Symfony 将在输出中显示一个可点击的命令配置文件链接（如果你的终端支持链接）。如果你以调试详细程度运行（`-vvv`），你还将看到命令消耗的时间和内存。

> **警告**
>
> 分析 [Messenger](messenger.md) 组件的 `messenger:consume` 命令时，请在命令中添加 `--no-reset` 选项，否则你不会获得任何配置文件。此外，考虑使用 `--limit` 选项仅处理几条消息，以使配置文件在分析器中更易读。

---

## 定义命令的旧语法

你也可以通过继承 `Command` 类来定义命令，而不是使用可调用命令。两种语法都受支持，但推荐使用可调用命令：

```php
// src/Command/CreateUserCommand.php
namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

// 命令名称是用户在 "php bin/console" 之后输入的内容
#[AsCommand(name: 'app:create-user')]
class CreateUserCommand extends Command
{
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        // ... 在这里放置创建用户的代码
    }
}
```

你可以选择通过覆盖 `configure()` 方法来定义描述、帮助消息以及输入选项和参数：

```php
// src/Command/CreateUserCommand.php

// ...
class CreateUserCommand extends Command
{
    // ...
    protected function configure(): void
    {
        $this
            // 运行 "php bin/console list" 时显示的命令描述
            ->setDescription('Creates a new user.')
            // 运行带 "--help" 选项的命令时显示的命令帮助
            ->setHelp('This command allows you to create a user...')
            // 为你的命令添加必填参数
            ->addArgument('username', InputArgument::REQUIRED, 'How the user should be named?')
        ;
    }
}
```

> **提示**
>
> 使用 `#[AsCommand]` 属性定义描述，而不是 `setDescription()` 方法，可以在不实例化其类的情况下检索命令描述，这使 `php bin/console list` 命令运行快得多。

`configure()` 方法在命令构造函数的末尾自动调用。如果你的命令定义了自己的构造函数，请先设置属性，然后调用父构造函数，以使这些属性在 `configure()` 方法中可用。

---

## 延伸阅读

控制台组件还包含一组"辅助工具"——能够帮助你完成不同任务的小工具：

- [问题辅助工具](components/console/helpers/questionhelper.md)：以交互方式向用户询问信息
- [格式化辅助工具](components/console/helpers/formatterhelper.md)：自定义输出着色
- [进度条](components/console/helpers/progressbar.md)：显示进度条
- [进度指示器](components/console/helpers/progressindicator.md)：显示进度指示器
- [表格](components/console/helpers/table.md)：以表格形式显示表格数据
- [调试格式化工具](components/console/helpers/debug_formatter.md)：提供在运行外部程序时输出调试信息的函数
- [进程辅助工具](components/console/helpers/processhelper.md)：允许你使用 `DebugFormatterHelper` 运行进程
- [光标](components/console/helpers/cursor.md)：允许你在终端中操作光标
- [树形结构](components/console/helpers/tree.md)：显示树形结构

更多控制台相关文档：

- [调用命令](console/calling_commands.md)
- [在控制器中运行命令](console/command_in_controller.md)
- [将命令注册为服务](console/commands_as_services.md)
- [隐藏命令](console/hide_commands.md)
- [控制台输入（参数和选项）](console/input.md)
- [延迟命令加载](console/lazy_commands.md)
- [可锁定命令特性](console/lockable_trait.md)
- [控制台样式](console/style.md)
- [输出详细程度](console/verbosity.md)
