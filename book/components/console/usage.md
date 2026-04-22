# 使用控制台命令、快捷方式和内置命令

除了您为命令指定的选项外，还有一些内置选项以及 Console 组件的一些内置命令。

> [!NOTE]
> 这些示例假设您已添加一个文件 `application.php` 在 CLI 上运行：
> 
> ```php
> #!/usr/bin/env php
> <?php
> // application.php
> 
> require __DIR__.'/vendor/autoload.php';
> 
> use Symfony\Component\Console\Application;
> 
> $application = new Application();
> // ...
> $application->run();
> ```

## 内置命令

有一个内置命令 `list`，它输出所有标准选项和已注册的命令：

```bash
$ php application.php list
```

不运行任何命令也可以获得相同的输出：

```bash
$ php application.php
```

`help` 命令列出指定命令的帮助信息。例如，要获取 `list` 命令的帮助：

```bash
$ php application.php help list
```

运行 `help` 而不指定命令将列出全局选项：

```bash
$ php application.php help
```

## 全局选项

您可以使用 `--help` 选项获取任何命令的帮助信息。要获取 list 命令的帮助：

```bash
$ php application.php list --help
$ php application.php list -h
```

您可以使用以下方式抑制输出：

```bash
# 抑制所有输出，包括错误
$ php application.php list --silent

# 抑制所有输出除了错误
$ php application.php list --quiet
$ php application.php list -q
```

您可以使用以下方式获取更详细的消息（如果命令支持）：

```bash
$ php application.php list --verbose
$ php application.php list -v
```

要输出更详细的消息，您可以使用这些选项：

```bash
$ php application.php list -vv
$ php application.php list -vvv
```

如果您设置可选参数来为应用命名和版本：

```php
$application = new Application('Acme Console Application', '1.2');
```

那么您可以使用：

```bash
$ php application.php list --version
$ php application.php list -V
```

获取此信息输出：

```text
Acme Console Application version 1.2
```

如果您不提供控制台名称，那么它将只输出：

```text
Console Tool
```

您可以使用以下方式强制启用 ANSI 输出着色：

```bash
$ php application.php list --ansi
```

或使用以下方式关闭它：

```bash
$ php application.php list --no-ansi
```

您可以使用以下方式抑制您正在运行的命令中的任何交互式问题：

```bash
$ php application.php list --no-interaction
$ php application.php list -n
```

## 快捷语法

您不必输入完整的命令名称。您只需输入最短的明确名称即可运行命令。因此，如果没有冲突的命令，那么您可以像这样运行 `help`：

```bash
$ php application.php h
```

如果您有使用 `:` 来命名空间命令的命令，那么您只需要为每个部分输入最短的明确文本。如果您已创建如 [Console 组件](../console.md)中所示的 `demo:greet`，那么您可以使用以下方式运行它：

```bash
$ php application.php d:g Fabien

# 只要明确，您还可以混合使用大小写
# php application.php Demo:g Fabien
# php application.php de:Gr Fabien
# php application.php DE:Gre Fabien
```

如果您输入的短命令是模糊的（即有多个命令匹配），那么将不会运行任何命令，并且将输出可能选择的命令的一些建议。
