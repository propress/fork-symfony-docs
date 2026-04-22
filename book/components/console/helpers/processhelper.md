# Process Helper

Process Helper 在进程运行时显示进程，并报告有关进程状态的有用信息。

要显示进程详细信息，请使用 `Symfony\Component\Console\Helper\ProcessHelper` 并使用详细程度运行命令。例如，使用非常详细的详细程度（例如 `-vv`）运行以下代码：

```php
use Symfony\Component\Process\Process;

$helper = new ProcessHelper();
$process = new Process(['figlet', 'Symfony']);

$helper->run($output, $process);
```

将导致此输出：

![Process Helper 详细输出](/_images/components/console/process-helper-verbose.png)

使用调试详细程度（例如 `-vvv`）将导致更详细的输出：

![Process Helper 调试输出](/_images/components/console/process-helper-debug.png)

如果进程失败，调试会更容易：

![Process Helper 错误调试输出](/_images/components/console/process-helper-error-debug.png)

> [!NOTE]
> 默认情况下，进程辅助工具使用错误输出（`stderr`）作为其默认输出。可以通过将 `Symfony\Component\Console\Output\StreamOutput` 的实例传递给 `Symfony\Component\Console\Helper\ProcessHelper::run` 方法来更改此行为。

## 参数

有两种方法可以使用进程辅助工具：

* 参数数组：

```php
// ...
$helper->run($output, ['figlet', 'Symfony']);
```

> [!NOTE]
> 当针对参数数组运行辅助工具时，请注意这些参数将自动转义。

* 传递 `Symfony\Component\Process\Process` 实例：

```php
use Symfony\Component\Process\Process;

// ...
$process = new Process(['figlet', 'Symfony']);

$helper->run($output, $process);
```

## 自定义显示

您可以使用 `Symfony\Component\Console\Helper\ProcessHelper::run` 方法的第三个参数显示自定义错误消息：

```php
$helper->run($output, $process, 'The process failed :(');
```

可以将自定义进程回调作为第四个参数传递。有关回调文档，请参阅 Process 组件：

```php
use Symfony\Component\Process\Process;

$helper->run($output, $process, 'The process failed :(', function (string $type, string $data): void {
    if (Process::ERR === $type) {
        // ... 使用 stderr 输出做某事
    } else {
        // ... 使用 stdout 做某事
    }
});
```
