# 创建和操作字符串

Symfony 提供了一套面向对象的 API 来处理 Unicode 字符串（以字节、码位和字素簇的形式）。这套 API 通过 String 组件提供，你必须先在应用中安装它：

```terminal
$ composer require symfony/string
```

## 什么是字符串？

如果你已经了解在处理字符串时什么是"码位"或"字素簇"，可以跳过本节。否则，请阅读本节以了解该组件使用的术语。

像英语这样的语言只需要非常有限的字符和符号集即可显示任何内容。每个字符串都是一系列字符（字母或符号），甚至可以用最有限的标准（如 [ASCII](https://en.wikipedia.org/wiki/ASCII)）进行编码。

然而，其他语言需要数千个符号来显示其内容。它们需要复杂的编码标准（如 [Unicode](https://en.wikipedia.org/wiki/Unicode)），"字符"的概念不再有意义。相反，你必须处理以下术语：

* [码位（Code points）](https://en.wikipedia.org/wiki/Code_point)：它们是信息的原子单位。字符串是一系列码位。每个码位都是一个数字，其含义由 [Unicode](https://en.wikipedia.org/wiki/Unicode) 标准给出。例如，英文字母 `A` 是 `U+0041` 码位，日语假名 `の` 是 `U+306E` 码位。
* [字素簇（Grapheme clusters）](https://en.wikipedia.org/wiki/Grapheme)：它们是显示为单个图形单元的一个或多个码位的序列。例如，西班牙字母 `ñ` 是一个包含两个码位的字素簇：`U+006E` = `n`（"拉丁小写字母 N"）+ `U+0303` = `◌̃`（"组合波浪符"）。
* 字节：它们是为字符串内容实际存储的信息。每个码位根据所使用的标准（UTF-8、UTF-16 等）可能需要一个或多个字节的存储。

下图显示了同一单词用英语（`hello`）和印地语（`नमस्ते`）书写时的字节、码位和字素簇：

!["hello" 中的每个字母由一个字节、一个码位和一个字素簇组成。在印地语翻译中，前两个字母（"नम"）占用三个字节、一个码位和一个字素簇。最后的字母（"स्ते"）各占六个字节、两个码位和一个字素簇。](/_images/components/string/bytes-points-graphemes.png)

## 用法

创建 `Symfony\Component\String\ByteString`、`Symfony\Component\String\CodePointString` 或 `Symfony\Component\String\UnicodeString` 类型的新对象，将字符串内容作为参数传入，然后使用面向对象的 API 处理这些字符串：

```php
use Symfony\Component\String\UnicodeString;

$text = new UnicodeString('This is a déjà-vu situation.')
    ->trimEnd('.')
    ->replace('déjà-vu', 'jamais-vu')
    ->append('!');
// $text = 'This is a jamais-vu situation!'

$content = new UnicodeString('नमस्ते दुनिया');
if ($content->ignoreCase()->startsWith('नमस्ते')) {
    // ...
}
```

## 方法参考

### 创建字符串对象的方法

首先，你可以使用以下类创建准备以字节、码位和字素簇存储字符串的对象：

```php
use Symfony\Component\String\ByteString;
use Symfony\Component\String\CodePointString;
use Symfony\Component\String\UnicodeString;

$foo = new ByteString('hello');
$bar = new CodePointString('hello');
// UnicodeString 是最常用的类
$baz = new UnicodeString('hello');
```

使用 `wrap()` 静态方法一次实例化多个字符串对象：

```php
$contents = ByteString::wrap(['hello', 'world']);        // $contents = ByteString[]
$contents = UnicodeString::wrap(['I', '❤️', 'Symfony']); // $contents = UnicodeString[]

// 使用 unwrap 方法进行逆向转换
$contents = UnicodeString::unwrap([
    new UnicodeString('hello'), new UnicodeString('world'),
]); // $contents = ['hello', 'world']
```

如果你要处理大量字符串对象，可以考虑使用快捷函数使代码更简洁：

```php
// b() 函数创建字节字符串
use function Symfony\Component\String\b;

// 以下两行等价
$foo = new ByteString('hello');
$foo = b('hello');

// u() 函数创建 Unicode 字符串
use function Symfony\Component\String\u;

// 以下两行等价
$foo = new UnicodeString('hello');
$foo = u('hello');

// s() 函数根据给定内容创建字节字符串或 Unicode 字符串
use function Symfony\Component\String\s;

// 创建 ByteString 对象
$foo = s("\xfe\xff");
// 创建 UnicodeString 对象
$foo = s('अनुच्छेद');
```

还有一些专门的构造方法：

```php
// ByteString 可以创建给定长度的随机字符串
$foo = ByteString::fromRandom(12);
// 默认情况下，随机字符串使用 base58 字符；你可以用第二个可选参数设置要使用的字符
$foo = ByteString::fromRandom(6, 'AEIOU0123456789');
$foo = ByteString::fromRandom(10, 'qwertyuiop');

// CodePointString 和 UnicodeString 可以从码位创建字符串
$foo = UnicodeString::fromCodePoints(0x928, 0x92E, 0x938, 0x94D, 0x924, 0x947);
// 等价于：$foo = new UnicodeString('नमस्ते');
```

### 转换字符串对象的方法

每个字符串对象都可以转换为其他两种类型的对象：

```php
$foo = ByteString::fromRandom(12)->toCodePointString();
$foo = new CodePointString('hello')->toUnicodeString();
$foo = UnicodeString::fromCodePoints(0x68, 0x65, 0x6C, 0x6C, 0x6F)->toByteString();

// 可选的 $toEncoding 参数定义目标字符串的编码
$foo = new CodePointString('hello')->toByteString('Windows-1252');
// 可选的 $fromEncoding 参数定义原始字符串的编码
$foo = new ByteString('さよなら')->toCodePointString('ISO-2022-JP');
```

如果出于任何原因无法进行转换，你将收到一个 `Symfony\Component\String\Exception\InvalidArgumentException`。

还有一个方法可以获取某个位置存储的字节：

```php
// ('नमस्ते' 字节 = [224, 164, 168, 224, 164, 174, 224, 164, 184,
//                  224, 165, 141, 224, 164, 164, 224, 165, 135])
b('नमस्ते')->bytesAt(0);   // [224]
u('नमस्ते')->bytesAt(0);   // [224, 164, 168]

b('नमस्ते')->bytesAt(1);   // [164]
u('नमस्ते')->bytesAt(1);   // [224, 164, 174]
```

### 与长度和空白字符相关的方法

```php
// 返回给定字符串的字素、码位或字节数
$word = 'नमस्ते';
new ByteString($word)->length();      // 18（字节）
new CodePointString($word)->length(); // 6（码位）
new UnicodeString($word)->length();   // 4（字素）

// 某些符号在使用等宽字体（如控制台）表示时需要两倍宽度。
// 此方法返回表示整个单词所需的总宽度
$word = 'नमस्ते';
new ByteString($word)->width();      // 18
new CodePointString($word)->width(); // 4
new UnicodeString($word)->width();   // 4
// 如果文本包含多行，则返回所有行的最大宽度
$text = "<<<END
This is a
multiline text
END";
u($text)->width(); // 14

// 仅当字符串恰好是空字符串时返回 TRUE（连空白字符都不算）
u('hello world')->isEmpty();  // false
u('     ')->isEmpty();        // false
u('')->isEmpty();             // true

// 移除字符串开头和结尾的所有空白字符（' \n\r\t\x0C'），并将
// 两个或多个连续空白字符替换为单个空格（' '）字符
u("  \n\n   hello \t   \n\r   world \n    \n")->collapseWhitespace(); // 'hello world'
```

### 大小写转换方法

```php
// 将所有字素/码位转换为小写
u('FOO Bar Brİan')->lower();  // 'foo bar bri̇an'
// 根据特定语言环境的大小写映射将所有字素/码位转换为小写
u('FOO Bar Brİan')->localeLower('en');  // 'foo bar bri̇an'
u('FOO Bar Brİan')->localeLower('lt');  // 'foo bar bri̇̇an'

// 处理不同语言时，大写/小写还不够，有三种大小写（小写、大写、标题大小写），
// 某些字符没有大小写，大小写还与上下文和语言环境相关等。
// 此方法返回一个可用于不区分大小写比较的字符串
u('FOO Bar')->folded();             // 'foo bar'
u('Die O\'Brian Straße')->folded(); // "die o'brian strasse"

// 将所有字素/码位转换为大写
u('foo BAR bάz')->upper(); // 'FOO BAR BΆZ'
// 根据特定语言环境的大小写映射将所有字素/码位转换为大写
u('foo BAR bάz')->localeUpper('en'); // 'FOO BAR BΆZ'
u('foo BAR bάz')->localeUpper('el'); // 'FOO BAR BAZ'

// 将所有字素/码位转换为"标题大小写"
u('foo ijssel')->title();               // 'Foo ijssel'
u('foo ijssel')->title(allWords: true); // 'Foo Ijssel'
// 根据特定语言环境的大小写映射将所有字素/码位转换为"标题大小写"
u('foo ijssel')->localeTitle('en'); // 'Foo ijssel'
u('foo ijssel')->localeTitle('nl'); // 'Foo IJssel'

// 将所有字素/码位转换为驼峰命名法（camelCase）
u('Foo: Bar-baz.')->camel(); // 'fooBarBaz'
// 将所有字素/码位转换为蛇形命名法（snake_case）
u('Foo: Bar-baz.')->snake(); // 'foo_bar_baz'
// 将所有字素/码位转换为短横线命名法（kebab-case）
u('Foo: Bar-baz.')->kebab(); // 'foo-bar-baz'
// 将所有字素/码位转换为帕斯卡命名法（PascalCase）
u('Foo: Bar-baz.')->pascal(); // 'FooBarBaz'
// 其他大小写可以通过链式方法实现，例如：
u('Foo: Bar-baz.')->camel()->upper(); // 'FOOBARBAZ'
```

所有字符串类的方法默认区分大小写。你可以使用 `ignoreCase()` 方法执行不区分大小写的操作：

```php
u('abc')->indexOf('B');               // null
u('abc')->ignoreCase()->indexOf('B'); // 1
```

### 追加和前置方法

```php
// 在字符串的开头/结尾添加给定内容（一个或多个字符串）
u('world')->prepend('hello');      // 'helloworld'
u('world')->prepend('hello', ' '); // 'hello world'

u('hello')->append('world');      // 'helloworld'
u('hello')->append(' ', 'world'); // 'hello world'

// 在字符串开头添加给定内容（或删除它），以确保内容恰好以该内容开始
u('Name')->ensureStart('get');       // 'getName'
u('getName')->ensureStart('get');    // 'getName'
u('getgetName')->ensureStart('get'); // 'getName'
// 此方法类似，但作用于内容的结尾而不是开头
u('User')->ensureEnd('Controller');           // 'UserController'
u('UserController')->ensureEnd('Controller'); // 'UserController'
u('UserControllerController')->ensureEnd('Controller'); // 'UserController'

// 返回在给定字符串第一次出现之前/之后的内容
u('hello world')->before('world');                  // 'hello '
u('hello world')->before('o');                      // 'hell'
u('hello world')->before('o', includeNeedle: true); // 'hello'

u('hello world')->after('hello');                  // ' world'
u('hello world')->after('o');                      // ' world'
u('hello world')->after('o', includeNeedle: true); // 'o world'

// 返回在给定字符串最后一次出现之前/之后的内容
u('hello world')->beforeLast('o');                      // 'hello w'
u('hello world')->beforeLast('o', includeNeedle: true); // 'hello wo'

u('hello world')->afterLast('o');                      // 'rld'
u('hello world')->afterLast('o', includeNeedle: true); // 'orld'
```

### 填充和修剪方法

```php
// 通过在字符串的开头、结尾或两侧添加给定字符串，使字符串达到第一个参数指定的长度
u(' Lorem Ipsum ')->padBoth(20, '-'); // '--- Lorem Ipsum ----'
u(' Lorem Ipsum')->padStart(20, '-'); // '-------- Lorem Ipsum'
u('Lorem Ipsum ')->padEnd(20, '-');   // 'Lorem Ipsum --------'

// 将给定字符串重复指定次数
u('_.')->repeat(10); // '_._._._._._._._._._.'

// 从字符串的开头和结尾删除给定字符（默认为空白字符）
u('   Lorem Ipsum   ')->trim(); // 'Lorem Ipsum'
u('Lorem Ipsum   ')->trim('m'); // 'Lorem Ipsum   '
u('Lorem Ipsum')->trim('m');    // 'Lorem Ipsu'

u('   Lorem Ipsum   ')->trimStart(); // 'Lorem Ipsum   '
u('   Lorem Ipsum   ')->trimEnd();   // '   Lorem Ipsum'

// 从字符串的开头/结尾删除给定内容
u('file-image-0001.png')->trimPrefix('file-');           // 'image-0001.png'
u('file-image-0001.png')->trimPrefix('image-');          // 'file-image-0001.png'
u('file-image-0001.png')->trimPrefix('file-image-');     // '0001.png'
u('template.html.twig')->trimSuffix('.html');            // 'template.html.twig'
u('template.html.twig')->trimSuffix('.twig');            // 'template.html'
u('template.html.twig')->trimSuffix('.html.twig');       // 'template'
// 当传入前缀/后缀数组时，只修剪找到的第一个
u('file-image-0001.png')->trimPrefix(['file-', 'image-']); // 'image-0001.png'
u('template.html.twig')->trimSuffix(['.twig', '.html']);   // 'template.html'
```

### 搜索和替换方法

```php
// 检查字符串是否以给定字符串开始/结束
u('https://symfony.com')->startsWith('https'); // true
u('report-1234.pdf')->endsWith('.pdf');        // true

// 检查字符串内容是否与给定内容完全相同
u('foo')->equalsTo('foo'); // true

// 检查字符串内容是否匹配给定的正则表达式
u('avatar-73647.png')->match('/avatar-(\d+)\.png/');
// 结果 = ['avatar-73647.png', '73647', null]

// 你可以将 preg_match() 的标志作为第二个参数传入。如果传入 PREG_PATTERN_ORDER
// 或 PREG_SET_ORDER，则会使用 preg_match_all()。
u('206-555-0100 and 800-555-1212')->match('/\d{3}-\d{3}-\d{4}/', \PREG_PATTERN_ORDER);
// 结果 = [['206-555-0100', '800-555-1212']]

// 检查字符串是否包含任意给定字符串
u('aeiou')->containsAny('a');                 // true
u('aeiou')->containsAny(['ab', 'efg']);       // false
u('aeiou')->containsAny(['eio', 'foo', 'z']); // true

// 查找给定字符串第一次出现的位置
// （第二个参数是搜索开始的位置，负值与 PHP 函数中的含义相同）
u('abcdeabcde')->indexOf('c');     // 2
u('abcdeabcde')->indexOf('c', 2);  // 2
u('abcdeabcde')->indexOf('c', -4); // 7
u('abcdeabcde')->indexOf('eab');   // 4
u('abcdeabcde')->indexOf('k');     // null

// 查找给定字符串最后一次出现的位置
// （第二个参数是搜索开始的位置，负值与 PHP 函数中的含义相同）
u('abcdeabcde')->indexOfLast('c');     // 7
u('abcdeabcde')->indexOfLast('c', 2);  // 7
u('abcdeabcde')->indexOfLast('c', -4); // 2
u('abcdeabcde')->indexOfLast('eab');   // 4
u('abcdeabcde')->indexOfLast('k');     // null

// 替换所有出现的给定字符串
u('http://symfony.com')->replace('http://', 'https://'); // 'https://symfony.com'
// 替换所有匹配给定正则表达式的内容
u('(+1) 206-555-0100')->replaceMatches('/[^A-Za-z0-9]++/', ''); // '12065550100'
// 你可以将可调用对象作为第二个参数来执行高级替换
u('123')->replaceMatches('/\d/', function (string $match): string {
    return '['.$match[0].']';
}); // 结果 = '[1][2][3]'
```

### 连接、分割、截断和反转方法

```php
// 使用字符串作为"粘合剂"合并所有给定字符串
u(', ')->join(['foo', 'bar']); // 'foo, bar'

// 使用给定分隔符将字符串分割成片段
u('template_name.html.twig')->split('.');    // ['template_name', 'html', 'twig']
// 你可以将最大片段数设置为第二个参数
u('template_name.html.twig')->split('.', 2); // ['template_name', 'html.twig']

// 返回从第一个参数位置开始、具有第二个可选参数长度的子字符串
// （负值与 PHP 函数中的含义相同）
u('Symfony is great')->slice(0, 7);  // 'Symfony'
u('Symfony is great')->slice(0, -6); // 'Symfony is'
u('Symfony is great')->slice(11);    // 'great'
u('Symfony is great')->slice(-5);    // 'great'

// 将字符串缩短到给定长度（如果更长的话）
u('Lorem Ipsum')->truncate(3);             // 'Lor'
u('Lorem Ipsum')->truncate(80);            // 'Lorem Ipsum'
// 第二个参数是字符串被截断时添加的字符（总长度包含这些字符的长度）
// （注意 '…' 是包含三个点的单个字符，不是 '...'）
u('Lorem Ipsum')->truncate(8, '…');        // 'Lorem I…'
// 第三个可选参数定义超出长度时如何截断单词
// 默认值是 TruncateMode::Char，它在给定的精确长度处截断字符串
u('Lorem ipsum dolor sit amet')->truncate(8, cut: TruncateMode::Char);       // 'Lorem ip'
// 返回不超过给定长度的最后完整单词
u('Lorem ipsum dolor sit amet')->truncate(8, cut: TruncateMode::WordBefore); // 'Lorem'
// 返回不超过给定长度的最后完整单词，必要时超出长度
u('Lorem ipsum dolor sit amet')->truncate(8, cut: TruncateMode::WordAfter);   // 'Lorem ipsum'
```

```php
// 将字符串按给定长度断行
u('Lorem Ipsum')->wordwrap(4);                  // 'Lorem\nIpsum'
// 默认按空白字符断行；传入 TRUE 可无条件断行
u('Lorem Ipsum')->wordwrap(4, "\n", cut: true); // 'Lore\nm\nIpsu\nm'

// 用给定内容替换字符串的一部分：
// 第二个参数是替换开始的位置；
// 第三个参数是从字符串中删除的字素/码位数
u('0123456789')->splice('xxx');       // 'xxx'
u('0123456789')->splice('xxx', 0, 2); // 'xxx23456789'
u('0123456789')->splice('xxx', 0, 6); // 'xxx6789'
u('0123456789')->splice('xxx', 6);    // '012345xxx'

// 将字符串按给定参数指定的长度分割成片段
u('0123456789')->chunk(3);  // ['012', '345', '678', '9']

// 反转字符串内容的顺序
u('foo bar')->reverse();  // 'rab oof'
u('さよなら')->reverse(); // 'らなよさ'
```

### ByteString 新增的方法

这些方法只适用于 `ByteString` 对象：

```php
// 如果字符串内容是有效的 UTF-8 内容则返回 TRUE
b('Lorem Ipsum')->isUtf8(); // true
b("\xc3\x28")->isUtf8();    // false
```

### CodePointString 和 UnicodeString 新增的方法

这些方法只适用于 `CodePointString` 和 `UnicodeString` 对象：

```php
// 将任何字符串音译为 ASCII 编码定义的拉丁字母
// （不要使用此方法来构建 slug，因为该组件已经提供了一个 slugger，如本文后面所述）
u('नमस्ते')->ascii();    // 'namaste'
u('さよなら')->ascii(); // 'sayonara'
u('спасибо')->ascii(); // 'spasibo'

// 返回存储在给定位置的码位或码位数组
// ('नमस्ते' 字素的码位 = [2344, 2350, 2360, 2340]
u('नमस्ते')->codePointsAt(0); // [2344]
u('नमस्ते')->codePointsAt(2); // [2360]
```

[Unicode 等价性](https://en.wikipedia.org/wiki/Unicode_equivalence)是 Unicode 标准关于不同码位序列表示相同字符的规范。例如，瑞典字母 `å` 可以是单个码位（`U+00E5` = "带上圆圈的拉丁小写字母 A"）或两个码位的序列（`U+0061` = "拉丁小写字母 A" + `U+030A` = "组合上圆圈"）。`normalize()` 方法允许你选择规范化模式：

```php
// 这些将字母编码为单个码位：U+00E5
u('å')->normalize(UnicodeString::NFC);
u('å')->normalize(UnicodeString::NFKC);
// 这些将字母编码为两个码位：U+0061 + U+030A
u('å')->normalize(UnicodeString::NFD);
u('å')->normalize(UnicodeString::NFKD);
```

## 惰性加载字符串

有时，使用前面章节介绍的方法创建字符串不是最优的。例如，考虑一个需要一定计算才能获得的哈希值，而你最终可能不会用到它。

在这种情况下，最好使用 `Symfony\Component\String\LazyString` 类，它允许存储一个字符串，其值只在你需要时才生成：

```php
use Symfony\Component\String\LazyString;

$lazyString = LazyString::fromCallable(function (): string {
    // 计算字符串值...
    $value = ...;

    // 然后返回最终值
    return $value;
});
```

只有在程序执行过程中请求惰性字符串的值时，才会执行该回调。你也可以从 `Stringable` 对象创建惰性字符串：

```php
class Hash implements \Stringable
{
    public function __toString(): string
    {
        return $this->computeHash();
    }

    private function computeHash(): string
    {
        // 通过可能较重的处理计算哈希值
        $hash = ...;

        return $hash;
    }
}

// 然后从此哈希创建一个惰性字符串，只有在需要时才会触发哈希计算
$lazyHash = LazyString::fromStringable(new Hash());
```

## 处理表情符号

这些内容已移至 [Emoji 组件文档](/emoji)。

## Slugger

在某些上下文中，如 URL 和文件/目录名称，使用任何 Unicode 字符是不安全的。*Slugger* 将给定字符串转换为只包含安全 ASCII 字符的另一个字符串：

```php
use Symfony\Component\String\Slugger\AsciiSlugger;

$slugger = new AsciiSlugger();
$slug = $slugger->slug('Wôrķšƥáçè ~~sèťtïñğš~~');
// $slug = 'Workspace-settings'

// 你也可以传入包含附加字符替换的数组
$slugger = new AsciiSlugger('en', ['en' => ['%' => 'percent', '€' => 'euro']]);
$slug = $slugger->slug('10% or 5€');
// $slug = '10-percent-or-5-euro'

// 如果你的语言环境没有符号映射（如 'en_GB'），则会使用父语言环境的符号映射（即 'en'）
$slugger = new AsciiSlugger('en_GB', ['en' => ['%' => 'percent', '€' => 'euro']]);
$slug = $slugger->slug('10% or 5€');
// $slug = '10-percent-or-5-euro'

// 对于更动态的替换，可以传入 PHP 闭包而不是数组
$slugger = new AsciiSlugger('en', function (string $string, string $locale): string {
    return str_replace('❤️', 'love', $string);
});
```

单词之间的分隔符默认为短横线（`-`），但你可以将另一个分隔符定义为第二个参数：

```php
$slug = $slugger->slug('Wôrķšƥáçè ~~sèťtïñğš~~', '/');
// $slug = 'Workspace/settings'
```

Slugger 在应用其他转换之前将原始字符串音译为拉丁字母。原始字符串的语言环境会自动检测，但你也可以明确定义：

```php
// 这告诉 slugger 从韩语（'ko'）进行音译
$slugger = new AsciiSlugger('ko');

// 你可以将语言环境作为 slug() 的第三个可选参数覆盖
// 例如，此 slugger 从波斯语（'fa'）进行音译
$slug = $slugger->slug('...', '-', 'fa');
```

在 Symfony 应用中，你不需要自己创建 slugger。得益于[服务自动装配](/service_container/autowiring)，你可以通过在服务构造函数参数中类型提示 `Symfony\Component\String\Slugger\SluggerInterface` 来注入 slugger。注入的 slugger 的语言环境与请求语言环境相同：

```php
use Symfony\Component\String\Slugger\SluggerInterface;

class MyService
{
    public function __construct(
        private SluggerInterface $slugger,
    ) {
    }

    public function someMethod(): void
    {
        $slug = $this->slugger->slug('...');
    }
}
```

### 表情符号 Slug

你也可以将[表情符号音译器](/emoji#emoji-transliteration)与 slugger 结合使用，将任何表情符号转换为其文字表示：

```php
use Symfony\Component\String\Slugger\AsciiSlugger;

$slugger = new AsciiSlugger();
$slugger = $slugger->withEmoji();

$slug = $slugger->slug('a 😺, 🐈‍⬛, and a 🦁 go to 🏞️', '-', 'en');
// $slug = 'a-grinning-cat-black-cat-and-a-lion-go-to-national-park';

$slug = $slugger->slug('un 😺, 🐈‍⬛, et un 🦁 vont au 🏞️', '-', 'fr');
// $slug = 'un-chat-qui-sourit-chat-noir-et-un-tete-de-lion-vont-au-parc-national';
```

如果你想为表情符号使用特定语言环境，或使用来自 GitHub、GitLab 或 Slack 的短代码，请使用 `withEmoji()` 方法的第一个参数：

```php
use Symfony\Component\String\Slugger\AsciiSlugger;

$slugger = new AsciiSlugger();
$slugger = $slugger->withEmoji('github'); // 或 "en"、"fr" 等

$slug = $slugger->slug('a 😺, 🐈‍⬛, and a 🦁');
// $slug = 'a-smiley-cat-black-cat-and-a-lion';
```

## 变形器（Inflector）

在代码生成和代码内省等场景中，你需要将单词从单数/复数形式互相转换。例如，要了解与 *adder* 方法关联的属性，你必须从复数形式（`addStories()` 方法）转换为单数形式（`$story` 属性）。

大多数人类语言有简单的复数化规则，但同时也定义了许多例外。例如，英语的一般规则是在单词末尾添加 `s`（`book` → `books`），但即使是常见单词也有很多例外（`woman` → `women`、`life` → `lives`、`news` → `news`、`radius` → `radii` 等）。

该组件提供了一个 `Symfony\Component\String\Inflector\EnglishInflector` 类，以可信赖的方式将英语单词从/转换到单数/复数形式：

```php
use Symfony\Component\String\Inflector\EnglishInflector;

$inflector = new EnglishInflector();

$result = $inflector->singularize('teeth');   // ['tooth']
$result = $inflector->singularize('radii');   // ['radius']
$result = $inflector->singularize('leaves');  // ['leaf', 'leave', 'leaff']

$result = $inflector->pluralize('bacterium'); // ['bacteria']
$result = $inflector->pluralize('news');      // ['news']
$result = $inflector->pluralize('person');    // ['persons', 'people']
```

两个方法的返回值始终是一个数组，因为有时无法为给定单词确定唯一的单数/复数形式。

Symfony 还为其他语言提供变形器：

```php
use Symfony\Component\String\Inflector\FrenchInflector;

$inflector = new FrenchInflector();
$result = $inflector->singularize('souris'); // ['souris']
$result = $inflector->pluralize('hôpital');  // ['hôpitaux']

use Symfony\Component\String\Inflector\SpanishInflector;

$inflector = new SpanishInflector();
$result = $inflector->singularize('aviones'); // ['avión']
$result = $inflector->pluralize('miércoles'); // ['miércoles']
```

> **注意：** Symfony 提供了一个 `Symfony\Component\String\Inflector\InflectorInterface`，以防你需要实现自己的变形器。
