# Mime 组件

Mime 组件允许操作用于发送电子邮件的 MIME 消息，并提供与 MIME 类型相关的实用工具。

## 安装

```terminal
$ composer require symfony/mime
```

## 简介

[MIME]（多用途互联网邮件扩展）是一种互联网标准，它扩展了电子邮件的原始基本格式，以支持以下功能：

* 使用非 ASCII 字符的标头和文本内容；
* 具有多个部分的消息正文（例如 HTML 和纯文本内容）；
* 非文本附件：音频、视频、图像、PDF 等。

整个 MIME 标准复杂而庞大，但 Symfony 将所有复杂性抽象化，提供了两种创建 MIME 消息的方式：

* 基于 `Symfony\Component\Mime\Email` 类的高级 API，可快速创建包含所有常用功能的电子邮件消息；
* 基于 `Symfony\Component\Mime\Message` 类的低级 API，可对电子邮件消息的每个部分进行绝对控制。

## 使用

使用 `Symfony\Component\Mime\Email` 类及其*可链式调用*的方法来组合整个电子邮件消息：

```php
use Symfony\Component\Mime\Email;

$email = new Email()
    ->from('fabien@symfony.com')
    ->to('foo@example.com')
    ->cc('bar@example.com')
    ->bcc('baz@example.com')
    ->replyTo('fabien@symfony.com')
    ->priority(Email::PRIORITY_HIGH)
    ->subject('Important Notification')
    ->text('Lorem ipsum...')
    ->html('<h1>Lorem ipsum</h1> <p>...</p>')
;
```

此组件的唯一目的是创建电子邮件消息。使用 [Mailer 组件](mailer.md) 来实际发送它们。

## Twig 集成

Mime 组件与 Twig 有出色的集成，允许你从 Twig 模板创建消息、嵌入图像、内联 CSS 等。有关如何使用这些功能的详细信息，请参阅 Mailer 文档。

但如果你在没有 Symfony 框架的情况下使用 Mime 组件，则需要处理一些设置细节。

### Twig 设置

要与 Twig 集成，请使用 `Symfony\Bridge\Twig\Mime\BodyRenderer` 类来渲染模板并用结果更新电子邮件消息内容：

```php
// ...
use Symfony\Bridge\Twig\Mime\BodyRenderer;
use Twig\Environment;
use Twig\Loader\FilesystemLoader;

// when using the Mime component inside a full-stack Symfony application, you
// don't need to do this Twig setup. You only have to inject the 'twig' service
$loader = new FilesystemLoader(__DIR__.'/templates');
$twig = new Environment($loader);

$renderer = new BodyRenderer($twig);
// this updates the $email object contents with the result of rendering
// the template defined earlier with the given context
$renderer->render($email);
```

### 内联 CSS 样式（及其他扩展）

要使用 `inline_css` 过滤器，首先安装 Twig 扩展：

```terminal
$ composer require twig/cssinliner-extra
```

然后启用该扩展：

```php
// ...
use Twig\Extra\CssInliner\CssInlinerExtension;

$loader = new FilesystemLoader(__DIR__.'/templates');
$twig = new Environment($loader);
$twig->addExtension(new CssInlinerExtension());
```

启用其他扩展（如 MarkdownExtension 和 InkyExtension）也应使用相同的流程。

## 创建原始电子邮件消息

这适用于需要对每个电子邮件部分进行绝对控制的高级应用程序。对于有常规电子邮件需求的应用程序，不建议使用此方式，因为它增加了复杂性而没有实际收益。

在继续之前，了解电子邮件消息的低级结构非常重要。考虑一个包含文本和 HTML 两种格式内容、嵌入在内容中的单个 PNG 图像以及附加的 PDF 文件的消息。MIME 标准允许以不同方式构建此消息，但以下树形结构是在大多数电子邮件客户端上都能正常工作的结构：

```text
multipart/mixed
├── multipart/related
│   ├── multipart/alternative
│   │   ├── text/plain
│   │   └── text/html
│   └── image/png
└── application/pdf
```

以下是每个 MIME 消息部分的用途：

* `multipart/alternative`：当两个或多个部分是同一内容（或非常相似内容）的替代时使用。首选格式必须最后添加。
* `multipart/mixed`：用于在同一消息中发送不同类型的内容，例如附加文件时。
* `multipart/related`：用于指示每个消息部分是整体的一个组成部分。最常见的用法是显示嵌入在消息内容中的图像。

使用低级 `Symfony\Component\Mime\Message` 类创建电子邮件消息时，必须牢记以上所有内容，手动定义电子邮件的不同部分：

```php
use Symfony\Component\Mime\Header\Headers;
use Symfony\Component\Mime\Message;
use Symfony\Component\Mime\Part\Multipart\AlternativePart;
use Symfony\Component\Mime\Part\TextPart;

$headers = new Headers()
    ->addMailboxListHeader('From', ['fabien@symfony.com'])
    ->addMailboxListHeader('To', ['foo@example.com'])
    ->addTextHeader('Subject', 'Important Notification')
;

$textContent = new TextPart('Lorem ipsum...');
$htmlContent = new TextPart('<h1>Lorem ipsum</h1> <p>...</p>', null, 'html');
$body = new AlternativePart($textContent, $htmlContent);

$email = new Message($headers, $body);
```

嵌入图像和附加文件可以通过创建适当的电子邮件多部分来实现：

```php
// ...
use Symfony\Component\Mime\Part\DataPart;
use Symfony\Component\Mime\Part\Multipart\MixedPart;
use Symfony\Component\Mime\Part\Multipart\RelatedPart;

// ...
$embeddedImage = new DataPart(fopen('/path/to/images/logo.png', 'r'), null, 'image/png');
$imageCid = $embeddedImage->getContentId();

$attachedFile = new DataPart(fopen('/path/to/documents/terms-of-use.pdf', 'r'), null, 'application/pdf');

$textContent = new TextPart('Lorem ipsum...');
$htmlContent = new TextPart(sprintf(
    '<img src="cid:%s"/> <h1>Lorem ipsum</h1> <p>...</p>', $imageCid
), null, 'html');
$bodyContent = new AlternativePart($textContent, $htmlContent);
$body = new RelatedPart($bodyContent, $embeddedImage);

$messageParts = new MixedPart($body, $attachedFile);

$email = new Message($headers, $messageParts);
```

## 序列化电子邮件消息

使用 `Email` 或 `Message` 类创建的电子邮件消息可以序列化，因为它们是简单的数据对象：

```php
$email = new Email()
    ->from('fabien@symfony.com')
    // ...
;

$serializedEmail = serialize($email);
```

一个常见的使用场景是存储序列化的电子邮件消息，将其包含在通过 [Messenger 组件](messenger.md) 发送的消息中，并在稍后发送时重新创建它们。使用 `Symfony\Component\Mime\RawMessage` 类从序列化内容重新创建电子邮件消息：

```php
use Symfony\Component\Mime\RawMessage;

// ...
$serializedEmail = serialize($email);

// later, recreate the original message to actually send it
$message = new RawMessage(unserialize($serializedEmail));
```

## MIME 类型实用工具

尽管 MIME 主要是为创建电子邮件而设计的，但 MIME 标准定义的内容类型（也称为 [MIME 类型]和"媒体类型"）在电子邮件之外的通信协议（如 HTTP）中也非常重要。因此，此组件还提供了处理 MIME 类型的实用工具。

`Symfony\Component\Mime\MimeTypes` 类可在 MIME 类型和文件扩展名之间进行转换：

```php
use Symfony\Component\Mime\MimeTypes;

$mimeTypes = new MimeTypes();
$exts = $mimeTypes->getExtensions('application/javascript');
// $exts = ['js', 'jsm', 'mjs']
$exts = $mimeTypes->getExtensions('image/jpeg');
// $exts = ['jpeg', 'jpg', 'jpe']

$types = $mimeTypes->getMimeTypes('js');
// $types = ['application/javascript', 'application/x-javascript', 'text/javascript']
$types = $mimeTypes->getMimeTypes('apk');
// $types = ['application/vnd.android.package-archive']
```

这些方法返回包含一个或多个元素的数组。元素位置表示其优先级，因此第一个返回的扩展名是首选扩展名。

### 猜测 MIME 类型

另一个有用的实用工具允许你猜测任意给定文件的 MIME 类型：

```php
use Symfony\Component\Mime\MimeTypes;

$mimeTypes = new MimeTypes();
$mimeType = $mimeTypes->guessMimeType('/some/path/to/image.gif');
// Guessing is not based on the file name, so $mimeType will be 'image/gif'
// only if the given file is truly a GIF image
```

猜测 MIME 类型是一个耗时的过程，需要检查文件内容的一部分。Symfony 应用了多种猜测机制，其中一种基于 PHP 的 [fileinfo 扩展]。建议安装该扩展以提高猜测性能。

#### 添加 MIME 类型猜测器

你可以通过创建一个实现 `Symfony\Component\Mime\MimeTypeGuesserInterface` 的类来添加自己的 MIME 类型猜测器：

```php
namespace App;

use Symfony\Component\Mime\MimeTypeGuesserInterface;

class SomeMimeTypeGuesser implements MimeTypeGuesserInterface
{
    public function isGuesserSupported(): bool
    {
        // return true when the guesser is supported (might depend on the OS for instance)
        return true;
    }

    public function guessMimeType(string $path): ?string
    {
        // inspect the contents of the file stored in $path to guess its
        // type and return a valid MIME type ... or null if unknown

        return '...';
    }
}
```

MIME 类型猜测器必须注册为服务并使用 `mime.mime_type_guesser` 标签进行标记。如果你使用的是默认的 services.yaml 配置，由于自动配置的缘故，这已经为你完成了。

[MIME]: https://en.wikipedia.org/wiki/MIME
[MIME 类型]: https://en.wikipedia.org/wiki/Media_type
[fileinfo 扩展]: https://www.php.net/fileinfo
