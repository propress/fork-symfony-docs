# 使用事件

Console 组件的 Application 类允许您通过事件可选地挂钩到控制台应用的生命周期。它没有重新发明轮子，而是使用 Symfony EventDispatcher 组件来完成这项工作：

```php
use Symfony\Component\Console\Application;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();

$application = new Application();
$application->setDispatcher($dispatcher);
$application->run();
```

> [!WARNING]
> 控制台事件仅由正在执行的主命令触发。主命令调用的命令不会触发任何事件，除非由应用本身运行，请参阅调用命令。

## `ConsoleEvents::COMMAND` 事件

**典型用途：** 在运行任何命令之前执行某些操作（例如记录将要执行的命令），或显示有关要执行的事件的某些内容。

就在执行任何命令之前，会调度 `ConsoleEvents::COMMAND` 事件。监听器接收一个 `Symfony\Component\Console\Event\ConsoleCommandEvent` 事件：

```php
use Symfony\Component\Console\ConsoleEvents;
use Symfony\Component\Console\Event\ConsoleCommandEvent;

$dispatcher->addListener(ConsoleEvents::COMMAND, function (ConsoleCommandEvent $event): void {
    // 获取输入实例
    $input = $event->getInput();

    // 获取输出实例
    $output = $event->getOutput();

    // 获取要执行的命令
    $command = $event->getCommand();

    // 写入有关命令的内容
    $output->writeln(sprintf('Before running command <info>%s</info>', $command->getName()));

    // 获取应用
    $application = $command->getApplication();
});
```

### 在监听器内禁用命令

使用 `Symfony\Component\Console\Event\ConsoleCommandEvent::disableCommand` 方法，您可以在监听器内禁用命令。然后应用将*不*执行该命令，而是返回代码 `113`（在 `ConsoleCommandEvent::RETURN_CODE_DISABLED` 中定义）。此代码是符合 C/C++ 标准的控制台命令的[保留退出代码][reserved-exit-codes]之一：

```php
use Symfony\Component\Console\ConsoleEvents;
use Symfony\Component\Console\Event\ConsoleCommandEvent;

$dispatcher->addListener(ConsoleEvents::COMMAND, function (ConsoleCommandEvent $event): void {
    // 获取要执行的命令
    $command = $event->getCommand();

    // ... 检查命令是否可以执行

    // 禁用命令，这将导致跳过命令
    // 并从应用返回代码 113
    $event->disableCommand();

    // 可以在以后的监听器中启用命令
    if (!$event->commandShouldRun()) {
        $event->enableCommand();
    }
});
```

## `ConsoleEvents::ERROR` 事件

**典型用途：** 处理命令执行期间抛出的异常。

每当命令抛出异常时（包括由事件监听器触发的异常），都会调度 `ConsoleEvents::ERROR` 事件。监听器可以包装或更改异常，或在应用抛出异常之前执行任何有用的操作。

监听器接收一个 `Symfony\Component\Console\Event\ConsoleErrorEvent` 事件：

```php
use Symfony\Component\Console\ConsoleEvents;
use Symfony\Component\Console\Event\ConsoleErrorEvent;

$dispatcher->addListener(ConsoleEvents::ERROR, function (ConsoleErrorEvent $event): void {
    $output = $event->getOutput();

    $command = $event->getCommand();

    $output->writeln(sprintf('Oops, exception thrown while running command <info>%s</info>', $command->getName()));

    // 获取当前退出代码（异常代码）
    $exitCode = $event->getExitCode();

    // 将异常更改为另一个
    $event->setError(new \LogicException('Caught exception', $exitCode, $event->getError()));
});
```

## `ConsoleEvents::TERMINATE` 事件

**典型用途：** 在命令执行后执行一些清理操作。

命令执行后，会调度 `ConsoleEvents::TERMINATE` 事件。它可用于执行需要为所有命令执行的任何操作，或清理您在 `ConsoleEvents::COMMAND` 监听器中启动的内容（例如发送日志、关闭数据库连接、发送电子邮件...）。监听器也可能更改退出代码。

监听器接收一个 `Symfony\Component\Console\Event\ConsoleTerminateEvent` 事件：

```php
use Symfony\Component\Console\ConsoleEvents;
use Symfony\Component\Console\Event\ConsoleTerminateEvent;

$dispatcher->addListener(ConsoleEvents::TERMINATE, function (ConsoleTerminateEvent $event): void {
    // 获取输出
    $output = $event->getOutput();

    // 获取已执行的命令
    $command = $event->getCommand();

    // 显示给定内容
    $output->writeln(sprintf('After running command <info>%s</info>', $command->getName()));

    // 更改退出代码
    $event->setExitCode(128);
});
```

> [!TIP]
> 当命令抛出异常时，也会调度此事件。然后在 `ConsoleEvents::ERROR` 事件之后立即调度它。在这种情况下接收的退出代码是异常代码。
> 
> 此外，当命令在信号上退出时，也会调度该事件。您可以在专用部分中了解有关信号的更多信息。

## `ConsoleEvents::SIGNAL` 事件

**典型用途：** 在命令执行被中断后执行一些操作。

[信号][signals]是发送到进程的异步通知，以便通知它发生的事件。例如，当您在命令中按 `Ctrl + C` 时，操作系统会向其发送 `SIGINT` 信号。

当命令被中断时，Symfony 会调度 `ConsoleEvents::SIGNAL` 事件。监听此事件，以便在完成命令执行之前执行某些操作（例如记录某些结果、清理某些临时文件等）。

监听器接收一个 `Symfony\Component\Console\Event\ConsoleSignalEvent` 事件：

```php
use Symfony\Component\Console\ConsoleEvents;
use Symfony\Component\Console\Event\ConsoleSignalEvent;

$dispatcher->addListener(ConsoleEvents::SIGNAL, function (ConsoleSignalEvent $event): void {

    // 获取信号编号
    $signal = $event->getHandlingSignal();

    // 设置退出代码
    $event->setExitCode(0);

    if (\SIGINT === $signal) {
        echo "bye bye!";
    }
});
```

如果您希望命令在调度事件后继续执行，也可以使用 `Symfony\Component\Console\Event\ConsoleSignalEvent::abortExit` 方法中止退出：

```php
use Symfony\Component\Console\ConsoleEvents;
use Symfony\Component\Console\Event\ConsoleSignalEvent;

$dispatcher->addListener(ConsoleEvents::SIGNAL, function (ConsoleSignalEvent $event) {
    $event->abortExit();
});
```

> [!TIP]
> 所有可用的信号（`SIGINT`、`SIGQUIT` 等）都定义为 PCNTL PHP 扩展的常量。必须安装该扩展才能使这些常量可用。

如果您在 Symfony 应用中使用 Console 组件，命令可以通过订阅 `Symfony\Component\Console\Event\ConsoleSignalEvent` 事件来自己处理信号：

```php
// src/Command/MyCommand.php
namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsCommand(name: 'app:my-command')]
class MyCommand
{
    // ...

    #[AsEventListener(ConsoleSignalEvent::class)]
    public function handleSignal(ConsoleSignalEvent $event): void
    {
        // 在此处设置 PCNTL 扩展定义的任何常量
        if (in_array($event->getHandlingSignal(), [\SIGINT, \SIGTERM], true)) {
            // ...
        }

        // ...

        // 设置整数退出代码，或
        // false 以继续正常执行
        $event->setExitCode(0);
    }
}
```

Symfony 不处理命令接收的任何信号（甚至 `SIGKILL`、`SIGTERM` 等）。此行为是有意的，因为它为您提供了处理所有信号的灵活性，例如在终止命令之前执行某些任务。

> [!TIP]
> 如果您需要从其整数值获取信号名称（例如用于日志记录），可以使用 `Symfony\Component\Console\SignalRegistry\SignalMap::getSignalName` 方法。

## `ConsoleAlarmEvent`

**典型用途：** 在长时间运行的命令期间执行定期任务（例如数据库连接保持活动、扩展锁 TTL、心跳检查）。

当命令调用 `Symfony\Component\Console\Application::setAlarmInterval` 时，应用会设置一个循环警报间隔，导致操作系统以给定间隔（以秒为单位）发送 `SIGALRM` 信号。每次信号触发时，都会调度一个 `ConsoleAlarmEvent`，允许监听器运行定期逻辑：

```php
// src/Command/LongRunningCommand.php
namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[AsCommand(name: 'app:long-running')]
class LongRunningCommand extends Command
{
    protected function initialize(InputInterface $input, OutputInterface $output): void
    {
        // 每 10 秒触发一次警报
        $this->getApplication()->setAlarmInterval(10);
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        // 长时间运行的处理...

        return Command::SUCCESS;
    }
}
```

然后，您可以监听 `Symfony\Component\Console\Event\ConsoleAlarmEvent` 以在每个警报上执行操作：

```php
// src/EventListener/ConsoleAlarmListener.php
namespace App\EventListener;

use Symfony\Component\Console\Event\ConsoleAlarmEvent;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener]
class ConsoleAlarmListener
{
    public function __invoke(ConsoleAlarmEvent $event): void
    {
        // 例如 ping 数据库以保持连接活动
    }
}
```

> [!NOTE]
> 警报功能需要 `pcntl` PHP 扩展，并且在 Windows 上不可用。

[reserved-exit-codes]: https://www.tldp.org/LDP/abs/html/exitcodes.html
[signals]: https://en.wikipedia.org/wiki/Signal_(IPC)
