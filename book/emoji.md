# 使用 Emoji

Symfony 提供了多种实用工具，用于处理来自 [Unicode CLDR 数据集](https://github.com/unicode-org/cldr)的 Emoji 字符和序列。它们通过 Emoji 组件提供，你必须首先在应用中安装它：

```terminal
$ composer require symfony/emoji
```

存储所有 Emoji（约 5,000 个）的音译数据到所有语言需要相当大的磁盘空间。

如果你需要节省磁盘空间（例如因为你部署到某些对大小有严格限制的服务），请运行此命令（例如作为 `composer install` 后的自动化脚本），使用 PHP `zlib` 扩展压缩内部 Symfony Emoji 数据文件：

```terminal
# 根据你的应用安装调整 'compress' 二进制文件的路径
$ php ./vendor/symfony/emoji/Resources/bin/compress
```

---

## Emoji 音译 {#emoji-transliteration}

`EmojiTransliterator` 类提供了一种基于 [Unicode CLDR 数据集](https://github.com/unicode-org/cldr)将 Emoji 翻译为所有语言文字表示的方式：

```php
use Symfony\Component\Emoji\EmojiTransliterator;

// 用英语描述 Emoji
$transliterator = EmojiTransliterator::create('en');
$transliterator->transliterate('Menus with 🍕 or 🍝');
// => 'Menus with pizza or spaghetti'

// 用乌克兰语描述 Emoji
$transliterator = EmojiTransliterator::create('uk');
$transliterator->transliterate('Menus with 🍕 or 🍝');
// => 'Menus with піца or спагеті'
```

> **提示**
>
> 使用 String 组件中的 [slug 生成器](string.md#字符串-slug-生成器)时，你可以将其与 `EmojiTransliterator` 结合使用来对 Emoji 生成 slug。

---

## 音译 Emoji 文本短代码

GitHub 和 Slack 等服务允许使用文本短代码在消息中包含 Emoji（例如，你可以添加 `:+1:` 代码来渲染 👍 Emoji）。

Symfony 还提供了将 Emoji 音译为短代码（反之亦然）的功能。每个服务的短代码略有不同，因此创建音译器时必须将服务名称作为参数传递。

### GitHub Emoji 短代码音译

使用 `emoji-github` 语言环境将 Emoji 转换为 GitHub 短代码：

```php
$transliterator = EmojiTransliterator::create('emoji-github');
$transliterator->transliterate('Teenage 🐢 really love 🍕');
// => 'Teenage :turtle: really love :pizza:'
```

使用 `github-emoji` 语言环境将 GitHub 短代码转换为 Emoji：

```php
$transliterator = EmojiTransliterator::create('github-emoji');
$transliterator->transliterate('Teenage :turtle: really love :pizza:');
// => 'Teenage 🐢 really love 🍕'
```

### Gitlab Emoji 短代码音译

使用 `emoji-gitlab` 语言环境将 Emoji 转换为 Gitlab 短代码：

```php
$transliterator = EmojiTransliterator::create('emoji-gitlab');
$transliterator->transliterate('Breakfast with 🥝 or 🥛');
// => 'Breakfast with :kiwi: or :milk:'
```

使用 `gitlab-emoji` 语言环境将 Gitlab 短代码转换为 Emoji：

```php
$transliterator = EmojiTransliterator::create('gitlab-emoji');
$transliterator->transliterate('Breakfast with :kiwi: or :milk:');
// => 'Breakfast with 🥝 or 🥛'
```

### Slack Emoji 短代码音译

使用 `emoji-slack` 语言环境将 Emoji 转换为 Slack 短代码：

```php
$transliterator = EmojiTransliterator::create('emoji-slack');
$transliterator->transliterate('Menus with 🥗 or 🧆');
// => 'Menus with :green_salad: or :falafel:'
```

使用 `slack-emoji` 语言环境将 Slack 短代码转换为 Emoji：

```php
$transliterator = EmojiTransliterator::create('slack-emoji');
$transliterator->transliterate('Menus with :green_salad: or :falafel:');
// => 'Menus with 🥗 or 🧆'
```

### 通用 Emoji 短代码音译 {#text-emoji}

如果你不知道生成短代码使用的是哪个服务，可以使用 `text-emoji` 语言环境，它结合了所有服务的所有代码：

```php
$transliterator = EmojiTransliterator::create('text-emoji');

// Github 短代码
$transliterator->transliterate('Breakfast with :kiwi-fruit: or :milk-glass:');
// Gitlab 短代码
$transliterator->transliterate('Breakfast with :kiwi: or :milk:');
// Slack 短代码
$transliterator->transliterate('Breakfast with :kiwifruit: or :glass-of-milk:');

// 以上所有示例都产生相同的结果：
// => 'Breakfast with 🥝 or 🥛'
```

你可以使用 `emoji-text` 语言环境将 Emoji 转换为短代码：

```php
$transliterator = EmojiTransliterator::create('emoji-text');
$transliterator->transliterate('Breakfast with 🥝 or 🥛');
// => 'Breakfast with :kiwifruit: or :milk-glass:'
```

---

## 反向 Emoji 音译

给定 Emoji 的文字表示，你可以通过 `emojify` 过滤器将其反向恢复为实际的 Emoji：

```twig
{{ 'I like :kiwi-fruit:'|emojify }} {# 渲染为：I like 🥝 #}
{{ 'I like :kiwi:'|emojify }}       {# 渲染为：I like 🥝 #}
{{ 'I like :kiwifruit:'|emojify }}  {# 渲染为：I like 🥝 #}
```

默认情况下，`emojify` 使用[文本目录](#text-emoji)，它合并了所有服务的 Emoji 文本代码。如果你愿意，可以选择使用特定的目录：

```twig
{{ 'I :green-heart: this'|emojify }}                  {# 渲染为：I 💚 this #}
{{ ':green_salad: is nice'|emojify('slack') }}        {# 渲染为：🥗 is nice #}
{{ 'My :turtle: has no name yet'|emojify('github') }} {# 渲染为：My 🐢 has no name yet #}
{{ ':kiwi: is a great fruit'|emojify('gitlab') }}     {# 渲染为：🥝 is a great fruit #}
```

---

## 删除 Emoji

`EmojiTransliterator` 也可以通过特殊的 `strip` 语言环境从字符串中删除所有 Emoji：

```php
use Symfony\Component\Emoji\EmojiTransliterator;

$transliterator = EmojiTransliterator::create('strip');
$transliterator->transliterate('🎉Hey!🥳 🎁Happy Birthday!🎁');
// => 'Hey! Happy Birthday!'
```
