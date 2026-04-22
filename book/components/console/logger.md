# 使用日志记录器

Console 组件附带一个符合 [PSR-3][psr-3] 标准的独立日志记录器。根据详细程度设置，日志消息将被发送到作为参数传递给构造函数的 `Symfony\Component\Console\Output\OutputInterface` 实例。

日志记录器除了 `psr/log` 之外没有任何外部依赖项。这对于需要轻量级 PSR-3 兼容日志记录器的控制台应用和命令很有用：

```php
namespace Acme;

use Psr\Log\LoggerInterface;

class MyDependency
{
    public function __construct(
        private LoggerInterface $logger,
    ) {
    }

    public function doStuff(): void
    {
        $this->logger->info('I love Tony Vairelles\' hairdresser.');
    }
}
```

您可以依靠日志记录器在命令内部使用此依赖项：

```php
namespace Acme\Console\Command;

use Acme\MyDependency;
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Logger\ConsoleLogger;
use Symfony\Component\Console\Output\OutputInterface;

#[AsCommand(
    name: 'my:command',
    description: 'Use an external dependency requiring a PSR-3 logger'
)]
class MyCommand
{
    public function __invoke(OutputInterface $output): int
    {
        $logger = new ConsoleLogger($output);

        $myDependency = new MyDependency($logger);
        $myDependency->doStuff();

        return Command::SUCCESS;
    }
}
```

依赖项将使用 `Symfony\Component\Console\Logger\ConsoleLogger` 的实例作为日志记录器。发出的日志消息将显示在控制台输出上。

## 详细程度（Verbosity）

根据命令运行的详细程度级别，消息可能会或可能不会被发送到 `Symfony\Component\Console\Output\OutputInterface` 实例。

默认情况下，控制台日志记录器的行为类似于 Monolog 的 Console Handler。日志级别和详细程度之间的关联可以通过 `Symfony\Component\Console\Logger\ConsoleLogger` 构造函数的第二个参数配置：

```php
use Psr\Log\LogLevel;
// ...

$verbosityLevelMap = [
    LogLevel::NOTICE => OutputInterface::VERBOSITY_NORMAL,
    LogLevel::INFO   => OutputInterface::VERBOSITY_NORMAL,
];

$logger = new ConsoleLogger($output, $verbosityLevelMap);
```

## 颜色

日志记录器输出日志消息时使用反映其级别的颜色进行格式化。此行为可通过构造函数的第三个参数配置：

```php
// ...
$formatLevelMap = [
    LogLevel::CRITICAL => ConsoleLogger::ERROR,
    LogLevel::DEBUG    => ConsoleLogger::INFO,
];

$logger = new ConsoleLogger($output, [], $formatLevelMap);
```

## 错误

Console 日志记录器包含一个 `hasErrored()` 方法，只要在命令执行期间记录了任何错误消息，该方法就会返回 `true`。这对于决定作为执行命令的结果返回哪个状态代码很有用。

[psr-3]: https://www.php-fig.org/psr/psr-3/
