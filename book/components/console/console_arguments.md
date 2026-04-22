# 理解控制台参数和选项的处理方式

Symfony Console 应用遵循大多数 CLI 实用工具使用的相同 [docopt][docopt] 标准。本文解释如何处理命令定义具有必需值、无值等选项时的边缘情况。阅读[另一篇文章](../../../console/input.md)以了解如何在 Symfony Console 命令中使用参数和选项。

看看下面这个有三个选项的命令：

```php
namespace Acme\Console\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Attribute\Option;

#[AsCommand(name: 'demo:args', description: 'Describe args behaviors')]
class DemoArgsCommand
{
    public function __invoke(
        #[Option(shortcut: 'f')] bool $foo = false,
        #[Option(shortcut: 'b')] string $bar = '',
        #[Option(shortcut: 'c')] string|bool $cat = false,
    ): int {
        // ...
    }
}
```

此示例使用带有 `#[Option]` 属性的可调用命令。如果您更喜欢经典方法：

```php
namespace Acme\Console\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputDefinition;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;

#[AsCommand(name: 'demo:args', description: 'Describe args behaviors')]
class DemoArgsCommand extends Command
{
    protected function configure(): void
    {
        $this
            ->setDefinition(
                new InputDefinition([
                    new InputOption('foo', 'f'),
                    new InputOption('bar', 'b', InputOption::VALUE_REQUIRED),
                    new InputOption('cat', 'c', InputOption::VALUE_OPTIONAL),
                ])
            );
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        // ...
    }
}
```

由于 `foo` 选项不接受值，它将是 `false`（当未传递给命令时）或 `true`（当用户传递 `--foo` 时）。`bar` 选项（及其 `b` 快捷方式）的值是必需的。它可以通过空格或 `=` 字符与选项名称分隔。`cat` 选项（及其 `c` 快捷方式）的行为类似，只是它不需要值。查看下表以获取传递选项的可能方式的概述：

| Input | `foo` | `bar` | `cat` |
|---|---|---|---|
| `--bar=Hello` | `false` | `"Hello"` | `null` |
| `--bar Hello` | `false` | `"Hello"` | `null` |
| `-b=Hello` | `false` | `"=Hello"` | `null` |
| `-b Hello` | `false` | `"Hello"` | `null` |
| `-bHello` | `false` | `"Hello"` | `null` |
| `-fcWorld -b Hello` | `true` | `"Hello"` | `"World"` |
| `-cfWorld -b Hello` | `false` | `"Hello"` | `"fWorld"` |
| `-cbWorld` | `false` | `null` | `"bWorld"` |

当命令还接受可选参数时，事情会变得更复杂：

```php
// ...

new InputDefinition([
    // ...
    new InputArgument('arg', InputArgument::OPTIONAL),
]);
```

您可能必须使用特殊的 `--` 分隔符将选项与参数分隔。查看下表中的第五个示例，其中使用它来告诉命令 `World` 是 `arg` 的值，而不是可选 `cat` 选项的值：

| Input | `bar` | `cat` | `arg` |
|---|---|---|---|
| `--bar Hello` | `"Hello"` | `null` | `null` |
| `--bar Hello World` | `"Hello"` | `null` | `"World"` |
| `--bar "Hello World"` | `"Hello World"` | `null` | `null` |
| `--bar Hello --cat World` | `"Hello"` | `"World"` | `null` |
| `--bar Hello --cat -- World` | `"Hello"` | `null` | `"World"` |
| `-b Hello -c World` | `"Hello"` | `"World"` | `null` |

## 选项属性约束

在可调用命令中使用 `#[Option]` 属性时，会强制执行以下规则以确保一致的行为：

* 选项**必须始终具有默认值**。与参数不同，选项不能是必需的，因为用户可能根本不提供它们；
* 可为空的布尔选项（`?bool`）不能具有 `true` 或 `false` 默认值。使用 `null` 作为默认值以启用可否定行为；
* 可为空的非布尔选项（例如 `?string`）必须具有 `null` 作为默认值；
* 仅允许 `string|bool`、`int|bool` 和 `float|bool` 的联合类型，并且必须具有 `false` 作为默认值。

有效选项定义的示例：

```php
#[Option] bool $verbose = false           // VALUE_NONE
#[Option] bool $colors = true             // VALUE_NEGATABLE (--colors 或 --no-colors)
#[Option] ?bool $debug = null             // VALUE_NEGATABLE (--debug 或 --no-debug)
#[Option] string $format = 'json'         // VALUE_REQUIRED
#[Option] ?string $filter = null          // VALUE_REQUIRED（可选值）
#[Option] int $limit = 10                 // VALUE_REQUIRED
#[Option] array $roles = []               // VALUE_IS_ARRAY
#[Option] string|bool $output = false     // VALUE_OPTIONAL (--output 或 --output=file.txt)
```

**无效**选项定义的示例：

```php
#[Option] string $format                  // 错误：没有默认值
#[Option] ?bool $debug = true             // 错误：可为空的布尔值具有 true 默认值
#[Option] ?string $filter = 'default'     // 错误：可为空但具有非空默认值
#[Option] string|bool $output = true      // 错误：联合类型具有 true 默认值
#[Option] array|bool $items = false       // 错误：不支持的联合类型
```

[docopt]: http://docopt.org/
