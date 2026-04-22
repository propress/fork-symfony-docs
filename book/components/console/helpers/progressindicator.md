# Progress Indicator

进度指示器对于让用户知道命令没有停滞很有用。与进度条不同，当命令持续时间不确定时（例如，长时间运行的命令、不可量化的任务等），将使用这些指示器。

它们通过实例化 `Symfony\Component\Console\Helper\ProgressIndicator` 类并在命令执行时推进进度来工作：

```php
use Symfony\Component\Console\Helper\ProgressIndicator;

// 创建新的进度指示器
$progressIndicator = new ProgressIndicator($output);

// 使用自定义消息启动并显示进度指示器
$progressIndicator->start('Processing...');

$i = 0;
while ($i++ < 50) {
    // ... 做一些工作

    // 推进进度指示器
    $progressIndicator->advance();
}

// 确保进度指示器显示最终消息
$progressIndicator->finish('Finished');
```

## 自定义进度指示器

### 内置格式

默认情况下，进度指示器上呈现的信息取决于 `OutputInterface` 实例的当前详细级别：

```text
# OutputInterface::VERBOSITY_NORMAL（没有详细标志的 CLI）
 \ Processing...
 | Processing...
 / Processing...
 - Processing...
 ✔ Finished

# OutputInterface::VERBOSITY_VERBOSE (-v)
 \ Processing... (1 sec)
 | Processing... (1 sec)
 / Processing... (1 sec)
 - Processing... (1 sec)
 ✔ Finished (1 sec)

# OutputInterface::VERBOSITY_VERY_VERBOSE (-vv) 和 OutputInterface::VERBOSITY_DEBUG (-vvv)
 \ Processing... (1 sec, 6.0 MiB)
 | Processing... (1 sec, 6.0 MiB)
 / Processing... (1 sec, 6.0 MiB)
 - Processing... (1 sec, 6.0 MiB)
 ✔ Finished (1 sec, 6.0 MiB)
```

> [!TIP]
> 使用 quiet 标志（`-q`）调用命令以不显示任何进度指示器。

您还可以通过 `ProgressIndicator` 构造函数的第二个参数强制使用格式，而不是依赖于当前命令的详细模式：

```php
$progressIndicator = new ProgressIndicator($output, 'verbose');
```

内置格式如下：

* `normal`
* `verbose`
* `very_verbose`

如果您的终端不支持 ANSI，请使用 `no_ansi` 变体：

* `normal_no_ansi`
* `verbose_no_ansi`
* `very_verbose_no_ansi`

### 自定义指示器值

您还可以设置自己的指示器值，而不是使用内置指示器值：

```php
$progressIndicator = new ProgressIndicator($output, 'verbose', 100, ['⠏', '⠛', '⠹', '⢸', '⣰', '⣤', '⣆', '⡇']);
```

进度指示器现在将如下所示：

```text
 ⠏ Processing...
 ⠛ Processing...
 ⠹ Processing...
 ⢸ Processing...
 ✔ Finished
```

一旦进度完成，它会显示一个特殊的完成指示器（默认为 ✔）。您可以用自己的替换它：

```php
$progressIndicator = new ProgressIndicator($output, finishedIndicatorValue: '🎉');

try {
    /* do something */
    $progressIndicator->finish('Finished');
} catch (\Exception) {
    $progressIndicator->finish('Failed', '🚨');
}
```

进度指示器现在将如下所示：

```text
 \ Processing...
 | Processing...
 / Processing...
 - Processing...
 🎉 Finished
```

### 自定义占位符

进度指示器使用占位符（用 `%` 字符括起来的名称）来确定输出格式。以下是内置占位符的列表：

* `indicator`：当前指示器；
* `elapsed`：自进度指示器启动以来经过的时间；
* `memory`：当前内存使用情况；
* `message`：用于在进度指示器中显示任意消息。

例如，这是您可以自定义 `message` 占位符的方式：

```php
ProgressIndicator::setPlaceholderFormatterDefinition(
    'message',
    static function (ProgressIndicator $progressIndicator): string {
        // 返回任意字符串
        return 'My custom message';
    }
);
```

> [!NOTE]
> 占位符自定义是全局应用的，这意味着在 `setPlaceholderFormatterDefinition()` 调用之后显示的任何进度指示器都将受到影响。
