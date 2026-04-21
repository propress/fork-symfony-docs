# Finder 组件

> Finder 组件通过直观的流式接口，根据不同条件（名称、文件大小、修改时间等）查找文件和目录。

## 安装

```terminal
$ composer require symfony/finder
```

## 使用

`Symfony\Component\Finder\Finder` 类可以查找文件和/或目录：

```php
use Symfony\Component\Finder\Finder;

$finder = new Finder();
// 查找当前目录中的所有文件
$finder->files()->in(__DIR__);

// 检查是否有任何搜索结果
if ($finder->hasResults()) {
    // ...
}

foreach ($finder as $file) {
    $absoluteFilePath = $file->getRealPath();
    $fileNameWithExtension = $file->getRelativePathname();

    // ...
}
```

`$file` 变量是 `Symfony\Component\Finder\SplFileInfo` 的实例，它继承自 PHP 自己的 `SplFileInfo`，提供处理相对路径的方法。

> **警告：** `Finder` 对象是有状态的。任何方法调用都会改变其所有实例。如果需要执行多次搜索，请创建多个 `Finder` 实例，或在设置公共配置后克隆它：
>
> ```php
> $finder = new Finder();
> // 首先，为以下搜索配置公共选项
> $finder->files()->in('./templates');
>
> // 然后，克隆 finder 以在搜索文件之前创建一个新实例
> foreach ((clone $finder)->name('partial_*') as $file) {
>     echo('Processing partials config: ' . $file->getRelativePathname());
> }
>
> // 如果不克隆，之前的 ->name() 调用将影响此搜索
> foreach ((clone $finder)->name('partial_*')->name('plugin_*') as $file) {
>     echo('Processing plugin config: ' . $file->getRelativePathname());
> }
> ```

## 搜索文件和目录

该组件提供了很多方法来定义搜索条件。由于它们实现了流式接口，所有方法都可以链式调用。

### 位置

位置是唯一的必填条件。它告诉 finder 使用哪个目录进行搜索：

```php
$finder->in(__DIR__);
```

通过链式调用 `Symfony\Component\Finder\Finder::in` 在多个位置搜索：

```php
// 在*两个*目录中搜索
$finder->in([__DIR__, '/elsewhere']);

// 与上面相同
$finder->in(__DIR__)->in('/elsewhere');
```

使用 `*` 作为通配符，在匹配模式的目录中搜索（每个模式必须解析为至少一个目录路径）：

```php
$finder->in('src/Symfony/*/*/Resources');
```

使用 `Symfony\Component\Finder\Finder::exclude` 方法从匹配中排除目录：

```php
// 作为参数传递的目录必须相对于用 in() 方法定义的目录
$finder->in(__DIR__)->exclude('ruby');
```

也可以忽略没有读权限的目录：

```php
$finder->ignoreUnreadableDirs()->in(__DIR__);
```

由于 Finder 使用 PHP 迭代器，你可以传递任何带有支持的 [PHP URL 风格协议包装器][PHP wrapper for URL-style protocols]（`ftp://`、`zlib://` 等）的 URL：

```php
// 在 FTP 根目录中查找时，始终添加尾部斜杠
$finder->in('ftp://example.com/');

// 你也可以在 FTP 目录中查找
$finder->in('ftp://example.com/pub/');
```

它也适用于用户自定义的流：

```php
use Symfony\Component\Finder\Finder;

// 使用官方 AWS SDK 注册 's3://' 包装器
$s3Client = new Aws\S3\S3Client([/* 配置选项 */]);
$s3Client->registerStreamWrapper();

$finder = new Finder();
$finder->name('photos*')->size('< 100K')->date('since 1 hour ago');
foreach ($finder->in('s3://bucket-name') as $file) {
    // ... 对文件执行某些操作
}
```

> **参见：** 请阅读 [PHP streams] 文档，了解如何创建自己的流。

### 文件或目录

默认情况下，Finder 同时返回文件和目录。如果只需要查找文件或目录，请使用 `Symfony\Component\Finder\Finder::files` 和 `Symfony\Component\Finder\Finder::directories` 方法：

```php
// 仅查找文件；忽略目录
$finder->files();

// 仅查找目录；忽略文件
$finder->directories();
```

如果想跟踪[符号链接][symbolic links]，请使用 `followLinks()` 方法：

```php
$finder->files()->followLinks();
```

注意，此方法跟踪链接但不解析它们。考虑以下文件和目录的结构：

```text
├── folder1/
│   ├──file1.txt
│   ├── file2link（指向 folder2/file2.txt 文件的符号链接）
│   └── folder3link（指向 folder3/ 目录的符号链接）
├── folder2/
│   └── file2.txt
└── folder3/
    └── file3.txt
```

如果你尝试通过 `$finder->files()->in('/path/to/folder1/')` 查找 `folder1/` 中的所有文件，你将得到以下结果：

* **不**使用 `followLinks()` 方法时：`file1.txt` 和 `file2link`（此链接未被解析）。`folder3link` 不会出现在结果中，因为它既没有被跟踪也没有被解析；
* 使用 `followLinks()` 方法时：`file1.txt`、`file2link`（此链接仍未被解析）和 `folder3/file3.txt`（此文件出现在结果中是因为 `folder1/folder3link` 链接被跟踪了）。

### 版本控制文件

[版本控制系统][Version Control Systems]（简称 "VCS"），如 Git 和 Mercurial，会创建一些特殊文件来存储其元数据。在查找文件和目录时，默认情况下会忽略这些文件，但你可以使用 `ignoreVCS()` 方法更改此行为：

```php
$finder->ignoreVCS(false);
```

如果搜索目录及其子目录包含 `.gitignore` 文件，可以使用 `Symfony\Component\Finder\Finder::ignoreVCSIgnored` 方法重用这些规则来从结果中排除文件和目录：

```php
// 排除匹配 .gitignore 模式的文件/目录
$finder->ignoreVCSIgnored(true);
```

目录的规则总是覆盖其父目录的规则。

> **注意：** Git 从存储库根目录开始查找 `.gitignore` 文件。Symfony 的 Finder 行为不同，它从用于搜索文件/目录的目录开始查找 `.gitignore` 文件。为了与 Git 行为保持一致，你应该明确地从 Git 存储库根目录开始搜索。

### 文件名

使用 `Symfony\Component\Finder\Finder::name` 方法按名称查找文件：

```php
$finder->files()->name('*.php');
```

`name()` 方法接受 glob、字符串、正则表达式或 glob、字符串或正则表达式的数组：

```php
$finder->files()->name('/\.php$/');
```

可以通过链式调用或传递数组来定义多个文件名：

```php
$finder->files()->name('*.php')->name('*.twig');

// 与上面相同
$finder->files()->name(['*.php', '*.twig']);
```

`notName()` 方法排除匹配某个模式的文件：

```php
$finder->files()->notName('*.rb');
```

可以通过链式调用或传递数组来排除多个文件名：

```php
$finder->files()->notName('*.rb')->notName('*.py');

// 与上面相同
$finder->files()->notName(['*.rb', '*.py']);
```

### 文件内容

使用 `Symfony\Component\Finder\Finder::contains` 方法按内容查找文件：

```php
$finder->files()->contains('lorem ipsum');
```

`contains()` 方法接受字符串或正则表达式：

```php
$finder->files()->contains('/lorem\s+ipsum$/i');
```

`notContains()` 方法排除包含给定模式的文件：

```php
$finder->files()->notContains('dolor sit amet');
```

### 路径

使用 `Symfony\Component\Finder\Finder::path` 方法按路径查找文件和目录：

```php
// 匹配路径中任何位置包含 "data" 的文件（文件或目录）
$finder->path('data');
// 例如，如果它们存在，这将匹配 data/*.xml 和 data.xml
$finder->path('data')->name('*.xml');
```

在所有平台（包括 Windows）上，使用正斜杠（即 `/`）作为目录分隔符。该组件在内部进行必要的转换。

`path()` 方法接受字符串、正则表达式或字符串或正则表达式的数组：

```php
$finder->path('foo/bar');
$finder->path('/^foo\/bar/');
```

可以通过链式调用或传递数组来定义多个路径：

```php
$finder->path('data')->path('foo/bar');

// 与上面相同
$finder->path(['data', 'foo/bar']);
```

在内部，字符串通过转义斜杠和添加分隔符转换为正则表达式：

| 原始给定字符串 | 使用的正则表达式 |
|---|---|
| `dirname` | `/dirname/` |
| `a/b/c` | `/a\/b\/c/` |

`Symfony\Component\Finder\Finder::notPath` 方法按路径排除文件：

```php
$finder->notPath('other/dir');
```

可以通过链式调用或传递数组来排除多个路径：

```php
$finder->notPath('first/dir')->notPath('other/dir');

// 与上面相同
$finder->notPath(['first/dir', 'other/dir']);
```

### 文件大小

使用 `Symfony\Component\Finder\Finder::size` 方法按大小查找文件：

```php
$finder->files()->size('< 1.5K');
```

通过链式调用或传递数组来限制大小范围：

```php
$finder->files()->size('>= 1K')->size('<= 2K');

// 与上面相同
$finder->files()->size(['>= 1K', '<= 2K']);
```

比较运算符可以是以下任意一个：`>`、`>=`、`<`、`<=`、`==`、`!=`。

目标值可以使用千字节（`k`、`ki`）、兆字节（`m`、`mi`）或吉字节（`g`、`gi`）的量级。带有 `i` 后缀的那些根据 [IEC 标准][IEC standard] 使用适当的 `2**n` 版本。

### 文件日期

使用 `Symfony\Component\Finder\Finder::date` 方法按最后修改日期查找文件：

```php
$finder->date('since yesterday');
```

通过链式调用或传递数组来限制日期范围：

```php
$finder->date('>= 2018-01-01')->date('<= 2018-12-31');

// 与上面相同
$finder->date(['>= 2018-01-01', '<= 2018-12-31']);
```

比较运算符可以是以下任意一个：`>`、`>=`、`<`、`<=`、`==`。你也可以使用 `since` 或 `after` 作为 `>` 的别名，以及使用 `until` 或 `before` 作为 `<` 的别名。

目标值可以是 `strtotime` 支持的任何日期。

### 目录深度

默认情况下，Finder 递归地遍历目录。使用 `Symfony\Component\Finder\Finder::depth` 限制遍历的深度：

```php
// 这将只考虑作为直接子级的文件/目录
$finder->depth('== 0');
$finder->depth('< 3');
```

通过链式调用或传递数组来限制深度范围：

```php
$finder->depth('> 2')->depth('< 5');

// 与上面相同
$finder->depth(['> 2', '< 5']);
```

### 自定义过滤

使用 `Symfony\Component\Finder\Finder::filter` 使用自己的策略过滤结果：

```php
$filter = function (\SplFileInfo $file)
{
    if (strlen($file) > 10) {
        return false;
    }
};

$finder->files()->filter($filter);
```

`filter()` 方法接受一个闭包作为参数。对于每个匹配的文件，它会以 `Symfony\Component\Finder\SplFileInfo` 实例的形式调用该文件。如果闭包返回 `false`，则文件将从结果集中排除。

`filter()` 方法包含一个第二个可选参数用于修剪目录。如果设置为 `true`，此方法完全跳过被排除的目录，而不是遍历整个文件/目录结构后再排除它们。在使用闭包时，对于你想修剪的目录返回 `false`。

提前修剪目录可以显著提高性能，这取决于文件/目录层次的复杂性和被排除目录的数量。

## 排序结果

按名称、扩展名、大小或类型（先目录，后文件）排序结果：

```php
$finder->sortByName();
$finder->sortByCaseInsensitiveName();
$finder->sortByExtension();
$finder->sortBySize();
$finder->sortByType();
```

> **提示：** 默认情况下，`sortByName()` 方法使用 `strcmp` PHP 函数（例如 `file1.txt`、`file10.txt`、`file2.txt`）。将 `true` 作为参数传入，以使用 PHP 的[自然排序][natural sort order]算法（例如 `file1.txt`、`file2.txt`、`file10.txt`）。
>
> `sortByCaseInsensitiveName()` 方法使用不区分大小写的 `strcasecmp` PHP 函数。将 `true` 作为参数传入，以使用 PHP 的不区分大小写的[自然排序][natural sort order]算法（即 `strnatcasecmp` PHP 函数）。

按最后访问、更改或修改时间对文件和目录进行排序：

```php
$finder->sortByAccessedTime();

$finder->sortByChangedTime();

$finder->sortByModifiedTime();
```

你还可以使用 `sort()` 方法定义自己的排序算法：

```php
$finder->sort(function (\SplFileInfo $a, \SplFileInfo $b): int {
    return strcmp($a->getRealPath(), $b->getRealPath());
});
```

你可以使用 `reverseSorting()` 方法反转任何排序：

```php
// 结果将以 "Z 到 A" 而不是默认的 "A 到 Z" 排序
$finder->sortByName()->reverseSorting();
```

> **注意：** 注意，`sort*` 方法需要获取所有匹配的元素才能完成其工作。对于大型迭代器，这会很慢。

## 将结果转换为数组

Finder 实例是一个 `IteratorAggregate` PHP 类。因此，除了使用 `foreach` 遍历 Finder 结果之外，你还可以使用 `iterator_to_array` 函数将其转换为数组，或使用 `iterator_count` 获取条目数量。

如果你多次调用 `Symfony\Component\Finder\Finder::in` 方法在多个位置搜索，请将 `false` 作为第二个参数传递给 `iterator_to_array`，以避免问题（为每个位置创建一个单独的迭代器，如果不传递 `false` 给 `iterator_to_array`，结果集的键会被使用，其中一些可能会重复，其值会被覆盖）。

## 读取返回文件的内容

可以使用 `Symfony\Component\Finder\SplFileInfo::getContents` 读取返回文件的内容：

```php
use Symfony\Component\Finder\Finder;

$finder = new Finder();
$finder->files()->in(__DIR__);

foreach ($finder as $file) {
    $contents = $file->getContents();

    // ...
}
```

[fluent interface]: https://en.wikipedia.org/wiki/Fluent_interface
[symbolic links]: https://en.wikipedia.org/wiki/Symbolic_link
[Version Control Systems]: https://en.wikipedia.org/wiki/Version_control
[PHP wrapper for URL-style protocols]: https://www.php.net/manual/en/wrappers.php
[PHP streams]: https://www.php.net/streams
[IEC standard]: https://physics.nist.gov/cuu/Units/binary.html
[natural sort order]: https://en.wikipedia.org/wiki/Natural_sort_order
