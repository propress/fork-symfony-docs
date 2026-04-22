# Debug Formatter Helper

`Symfony\Component\Console\Helper\DebugFormatterHelper` 提供了在运行外部程序（例如进程或 HTTP 请求）时输出调试信息的函数。例如，如果您使用它来输出运行 `figlet symfony` 的结果，它可能会输出如下内容：

![Debug Formatter 输出](/_images/components/console/debug_formatter.png)

## 使用 Debug Formatter

可以像这样直接实例化调试格式化辅助工具：

```php
$debugFormatter = new DebugFormatterHelper();
```

它接受字符串并返回格式化的字符串，然后您可以将其输出到控制台（甚至记录信息或执行其他任何操作）。

此辅助工具的所有方法都将标识符作为第一个参数。这是每个程序的唯一值。这样，辅助工具可以同时为多个程序调试信息。使用 Process 组件时，您可能希望使用 `spl_object_hash`。

> [!TIP]
> 此信息通常太详细，无法默认显示。您可以使用详细级别仅在调试模式（`-vvv`）下显示它。

## 启动程序

一旦启动程序，您可以使用 `Symfony\Component\Console\Helper\DebugFormatterHelper::start` 来显示程序已启动的信息：

```php
// ...
$process = new Process(...);

$output->writeln($debugFormatter->start(
    spl_object_hash($process),
    'Some process description'
));

$process->run();
```

这将输出：

```text
 RUN Some process description
```

您可以使用第三个参数调整前缀：

```php
$output->writeln($debugFormatter->start(
    spl_object_hash($process),
    'Some process description',
    'STARTED'
));
// 将输出：
//  STARTED Some process description
```

## 输出进度信息

某些程序在运行时给出输出。可以使用 `Symfony\Component\Console\Helper\DebugFormatterHelper::progress` 显示此信息：

```php
use Symfony\Component\Process\Process;

// ...
$process = new Process(...);

$process->run(function (string $type, string $buffer) use ($output, $debugFormatter, $process): void {
    $output->writeln(
        $debugFormatter->progress(
            spl_object_hash($process),
            $buffer,
            Process::ERR === $type
        )
    );
});
// ...
```

在成功的情况下，这将输出：

```text
OUT The output of the process
```

在失败的情况下：

```text
ERR The output of the process
```

第三个参数是一个布尔值，告诉函数输出是否为错误输出。当为 `true` 时，输出被视为错误输出。

第四个和第五个参数允许您分别覆盖正常输出和错误输出的前缀。

## 停止程序

当程序停止时，您可以使用 `Symfony\Component\Console\Helper\DebugFormatterHelper::stop` 向用户通知此情况：

```php
// ...
$output->writeln(
    $debugFormatter->stop(
        spl_object_hash($process),
        'Some command description',
        $process->isSuccessful()
    )
);
```

这将输出：

```text
RES Some command description
```

在失败的情况下，这将显示为红色，在成功的情况下显示为绿色。

## 使用多个程序

如前所述，您也可以使用辅助工具同时显示多个程序。有关不同程序的信息将以不同的颜色显示，以明确哪个输出属于哪个命令。
