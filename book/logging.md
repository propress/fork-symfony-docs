# 日志

Symfony 附带两个最简化的 [PSR-3](https://www.php-fig.org/psr/psr-3/) 日志记录器：`Logger`（用于 HTTP 上下文）和 `ConsoleLogger`（用于 CLI 上下文）。按照[十二因素应用方法论](https://12factor.net/logs)，它们将从 `WARNING` 级别开始的消息发送到 [stderr](https://en.wikipedia.org/wiki/Standard_streams#Standard_error_(stderr))。

最小日志级别可以通过设置 `SHELL_VERBOSITY` 环境变量来更改：

| `SHELL_VERBOSITY` 值 | 最小日志级别 |
|-----------------------|-------------|
| `-1` | `ERROR` |
| `1` | `NOTICE` |
| `2` | `INFO` |
| `3` | `DEBUG` |

---

## 记录消息

要记录消息，在你的控制器或服务中注入默认日志记录器：

```php
use Psr\Log\LoggerInterface;
// ...

public function index(LoggerInterface $logger): Response
{
    $logger->info('I just got the logger');
    $logger->error('An error occurred');

    // 日志消息还可以包含占位符，这些是用花括号括起来的变量名
    // 其值作为第二个参数传递
    $logger->debug('User {userId} has logged in', [
        'userId' => $this->getUserId(),
    ]);

    $logger->critical('I left the oven on!', [
        // 在日志中包含额外的"上下文"信息
        'cause' => 'in_hurry',
    ]);

    // ...
}
```

建议在日志消息中添加占位符，因为：

- 更容易检查日志消息，因为许多日志工具会将除某些变量值之外相同的日志消息分组；
- 翻译这些日志消息更容易；
- 安全性更好，因为转义可以由实现以上下文感知的方式完成。

`logger` 服务对不同的日志级别/优先级有不同的方法。请参阅 [LoggerInterface](https://github.com/php-fig/log/blob/master/src/LoggerInterface.php) 获取日志记录器上所有方法的列表。

---

## Monolog

Symfony 与 [Monolog](https://github.com/Seldaek/monolog)（最流行的 PHP 日志库）无缝集成，在不同的地方创建和存储日志消息，并触发各种操作。

运行此命令在使用前安装基于 Monolog 的日志记录器：

```terminal
$ composer require symfony/monolog-bundle
```

---

## 日志存储位置

默认情况下，当你处于 `dev` 环境时，日志条目被写入 `var/log/dev.log` 文件。

在 `prod` 环境中，日志被写入 STDERR PHP 流，这在部署到没有磁盘写权限的服务器上的现代容器化应用中效果最好。

---

## 处理器：将日志写入不同位置

日志记录器有一个*处理器*栈，每个处理器可用于将日志条目写入不同位置（例如文件、数据库、Slack 等）。

> **提示**
>
> 你也可以配置日志"频道"，类似于分类。每个频道可以有其自己的处理器，这意味着你可以将不同的日志消息存储在不同的地方。请参阅 [logging/channels_handlers](logging/channels_handlers.md)。

此示例使用*两个*处理器：`stream`（写入文件）和 `syslog`（使用 `syslog` 函数写入日志）：

```yaml
# config/packages/prod/monolog.yaml
monolog:
    handlers:
        # "file_log" 键可以是任何名称
        file_log:
            type: stream
            # 记录到 var/log/(environment).log
            path: "%kernel.logs_dir%/%kernel.environment%.log"
            # 记录*所有*消息（debug 是最低级别）
            level: debug

        syslog_handler:
            type: syslog
            # 记录 error 级别及更高级别的消息
            level: error
```

这定义了一个处理器栈。每个处理器可以定义一个 `priority`（默认 `0`）来控制其在栈中的位置。优先级更高的处理器首先被调用，而具有相同优先级的处理器保持它们定义的顺序。

### 修改日志条目的处理器 {#logging-handler-fingers_crossed}

某些处理器不是将日志文件写入某处，而是用于在将日志条目发送到*其他*处理器之前对其进行过滤或修改。一个强大的内置处理器 `fingers_crossed` 在 `prod` 环境中默认使用。它在请求期间存储*所有*日志消息，但*仅在*其中一条消息达到 `action_level` 时才将它们传递给第二个处理器：

```yaml
# config/packages/prod/monolog.yaml
monolog:
    handlers:
        filter_for_errors:
            type: fingers_crossed
            # 如果*有一条*日志是 error 或更高，则将*所有*传递给 file_log
            action_level: error
            handler: file_log

        # 现在传递了*所有*日志，但仅当有一条日志是 error 或更高
        file_log:
            type: stream
            path: "%kernel.logs_dir%/%kernel.environment%.log"

        # 仍然传递*所有*日志，仍然只记录 error 或更高
        syslog_handler:
            type: syslog
            level: error
```

现在，如果即使有一个日志条目的 `LogLevel::ERROR` 级别或更高，则该请求的*所有*日志条目都通过 `file_log` 处理器保存到文件中。这意味着你的日志文件将包含有关问题请求的*所有*详细信息——使调试更加容易！

---

## 所有内置处理器

Monolog 附带了*许多*内置处理器，用于发送日志电子邮件、将其发送到 Loggly 或在 Slack 中通知你。这些都记录在 MonologBundle 本身中。完整列表，请参阅 [Monolog 配置](https://github.com/symfony/monolog-bundle/blob/4.x/src/DependencyInjection/Configuration.php)。

---

## 如何轮换你的日志文件

随着时间的推移，日志文件可能会变得很大。一个最佳实践解决方案是使用类似 [logrotate](https://github.com/logrotate/logrotate) 的 Linux 命令工具在日志文件变得太大之前轮换它们。

另一个选项是使用 `rotating_file` 处理器让 Monolog 为你轮换文件。此处理器每天创建一个新的日志文件，还可以自动删除旧文件：

```yaml
# config/packages/prod/monolog.yaml
monolog:
    handlers:
        main:
            type:  rotating_file
            path:  '%kernel.logs_dir%/%kernel.environment%.log'
            level: debug
            # 要保留的最大日志文件数
            # 默认值为零，表示无限文件
            max_files: 10
```

---

## 在服务中使用日志记录器

如果你的应用使用[服务自动配置](service_container.md#自动配置)，任何类实现 `Psr\Log\LoggerAwareInterface` 的服务都将收到对其 `setLogger()` 方法的调用，其中传递了默认日志记录器服务。

---

## 向每个日志添加额外数据（例如唯一请求令牌）

Monolog 还支持*处理器*：可以动态向你的日志条目添加额外信息的函数。

更多详情，请参阅 [logging/processors](logging/processors.md)。

---

## 处理长时间运行进程中的日志

在长时间运行的进程中，日志可以积累到 Monolog 中并导致一些缓冲区溢出、内存增加甚至不合逻辑的日志。可以使用 `Monolog\Logger` 实例上的 `reset()` 方法清除 Monolog 的内存数据。这通常应在长时间运行的进程处理的每个作业或任务之间调用。

---

## 延伸阅读

- [Monolog 发送错误邮件](logging/monolog_email.md)
- [日志频道和处理器](logging/channels_handlers.md)
- [日志格式化器](logging/formatter.md)
- [日志处理器](logging/processors.md)
- [日志处理器参考](logging/handlers.md)
- [排除 HTTP 状态码](logging/monolog_exclude_http_codes.md)
- [控制台日志](logging/monolog_console.md)
