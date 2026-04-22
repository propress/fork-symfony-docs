# Formatter Helper

`Symfony\Component\Console\Helper\FormatterHelper` 辅助工具提供了使用颜色格式化输出的函数。您可以使用此辅助工具做比基本颜色和样式更高级的事情：

```php
$formatter = new FormatterHelper();
```

这些方法返回一个字符串，您通常会通过将其传递给 `OutputInterface::writeln` 方法将其呈现到控制台。

> [!NOTE]
> 作为替代方案，考虑使用 SymfonyStyle 来显示样式化的块。

## 在节中打印消息

Symfony 在打印属于某个"节"的消息时提供了定义的样式。它以颜色打印节，并在其周围加上方括号，实际消息位于此右侧。减去颜色，它看起来像这样：

```text
[SomeSection] Here is some message related to that section
```

要重现此样式，您可以使用 `Symfony\Component\Console\Helper\FormatterHelper::formatSection` 方法：

```php
$formattedLine = $formatter->formatSection(
    'SomeSection',
    'Here is some message related to that section'
);
$output->writeln($formattedLine);
```

## 在块中打印消息

有时您希望能够打印带有背景颜色的整个文本块。Symfony 在打印错误消息时使用此功能。

如果您手动在多行上打印错误消息，您会注意到背景只与每个单独的行一样长。使用 `Symfony\Component\Console\Helper\FormatterHelper::formatBlock` 生成块输出：

```php
$errorMessages = ['Error!', 'Something went wrong'];
$formattedBlock = $formatter->formatBlock($errorMessages, 'error');
$output->writeln($formattedBlock);
```

如您所见，将消息数组传递给 `Symfony\Component\Console\Helper\FormatterHelper::formatBlock` 方法会创建所需的输出。如果您传递 `true` 作为第三个参数，块将使用更多填充进行格式化（消息上方和下方一个空行，左侧和右侧 2 个空格）。

您在块中使用的确切"样式"取决于您。在这种情况下，您使用预定义的 `error` 样式，但还有其他样式（`info`、`comment`、`question`），或者您可以创建自己的样式。请参阅控制台样式。

## 打印截断消息

有时您希望打印截断到显式字符长度的消息。这可以通过 `Symfony\Component\Console\Helper\FormatterHelper::truncate` 方法实现。

如果您想截断一个非常长的消息，例如截断到 7 个字符，您可以写：

```php
$message = "This is a very long message, which should be truncated";
$truncatedMessage = $formatter->truncate($message, 7);
$output->writeln($truncatedMessage);
```

输出将是：

```text
This is...
```

消息被截断到给定长度，然后后缀被附加到该字符串的末尾。

### 负字符串长度

如果长度为负，则从字符串末尾开始计算要截断的字符数：

```php
$truncatedMessage = $formatter->truncate($message, -5);
```

这将导致：

```text
This is a very long message, which should be trun...
```

### 自定义后缀

默认情况下，使用 `...` 后缀。如果您希望使用不同的后缀，请将其作为第三个参数传递给方法。后缀始终被附加，除非截断长度大于消息和后缀长度。如果您根本不想使用后缀，请传递空字符串：

```php
$truncatedMessage = $formatter->truncate($message, 7, '!!'); // 结果：This is!!
$truncatedMessage = $formatter->truncate($message, 7, '');   // 结果：This is

$truncatedMessage = $formatter->truncate('test', 10);
// 结果：test
// 因为 "test..." 字符串的长度短于 10
```

## 格式化时间

有时您希望将秒格式化为时间。这可以通过 `Symfony\Component\Console\Helper\Helper::formatTime` 方法实现。第一个参数是要格式化的秒数，第二个参数是结果的精度（默认为 `1`）：

```php
Helper::formatTime(0.001);         // 1 ms
Helper::formatTime(42);            // 42 s
Helper::formatTime(125);           // 2 min
Helper::formatTime(125, 2);        // 2 min, 5 s
Helper::formatTime(172799, 4);     // 1 d, 23 h, 59 min, 59 s
Helper::formatTime(172799.056, 5); // 1 d, 23 h, 59 min, 59 s, 56 ms
```

## 格式化内存

有时您希望将内存格式化为 GiB、MiB、KiB 和 B。这可以通过 `Symfony\Component\Console\Helper\Helper::formatMemory` 方法实现。唯一的参数是要格式化的内存大小：

```php
Helper::formatMemory(512);                // 512 B
Helper::formatMemory(1024);               // 1 KiB
Helper::formatMemory(1024 * 1024);        // 1.0 MiB
Helper::formatMemory(1024 * 1024 * 1024); // 1 GiB
```
