# 使用 Mailer 发送电子邮件

## 安装

Symfony 的 Mailer 与 Mime 组件共同构成了一个用于创建和发送电子邮件的**强大**系统——完整支持多部分消息、Twig 集成、CSS 内联、文件附件等。通过以下命令安装：

```terminal
$ composer require symfony/mailer
```

---

## 传输设置 {#mailer-transport-setup}

电子邮件通过"传输"投递。默认情况下，你可以通过在 `.env` 文件中配置 DSN 来通过 SMTP 投递电子邮件：

```env
# .env
MAILER_DSN=smtp://user:pass@smtp.example.com:port
```

```yaml
# config/packages/mailer.yaml
framework:
    mailer:
        dsn: '%env(MAILER_DSN)%'
```

### 使用内置传输

| DSN 协议 | 示例 | 描述 |
|---------|------|------|
| smtp | `smtp://user:pass@smtp.example.com:25` | Mailer 使用 SMTP 服务器发送电子邮件 |
| sendmail | `sendmail://default` | Mailer 使用本地 sendmail 二进制文件发送电子邮件 |
| native | `native://default` | Mailer 使用 `php.ini` 中配置的 sendmail 二进制文件和选项 |

### 使用第三方传输 {#mailer_3rd_party_transport}

你可以通过第三方提供商发送电子邮件，而不是使用自己的 SMTP 服务器：

| 服务 | 安装命令 | 支持 Webhook |
|-----|---------|------------|
| Amazon SES | `composer require symfony/amazon-mailer` | |
| Brevo | `composer require symfony/brevo-mailer` | 是 |
| Mailgun | `composer require symfony/mailgun-mailer` | 是 |
| Postmark | `composer require symfony/postmark-mailer` | 是 |
| SendGrid | `composer require symfony/sendgrid-mailer` | 是 |
| Mailtrap | `composer require symfony/mailtrap-mailer` | 是 |
| 等等... | | |

每个库都包含一个 Symfony Flex recipe，它将在你的 `.env` 文件中添加配置示例。例如，安装 SendGrid：

```terminal
$ composer require symfony/sendgrid-mailer
```

然后取消注释 `.env` 文件中的新行：

```env
# .env
MAILER_DSN=sendgrid://KEY@default
```

### 高可用性

Symfony 的 mailer 通过"故障转移"技术支持高可用性：

```env
MAILER_DSN="failover(postmark+api://ID@default sendgrid+smtp://KEY@default)"
```

### 负载均衡

通过"轮询"技术支持负载均衡：

```env
MAILER_DSN="roundrobin(postmark+api://ID@default sendgrid+smtp://KEY@default)"
```

### TLS 对等验证

默认情况下，SMTP 传输执行 TLS 对等验证。此行为可通过 `verify_peer` 选项配置：

```php
$dsn = 'smtp://user:pass@smtp.example.com?verify_peer=0';
```

---

## 创建和发送消息

要发送电子邮件，通过类型提示 `MailerInterface` 获取 `Mailer` 实例，并创建一个 `Email` 对象：

```php
// src/Controller/MailerController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Mime\Email;
use Symfony\Component\Routing\Attribute\Route;

class MailerController extends AbstractController
{
    #[Route('/email')]
    public function sendEmail(MailerInterface $mailer): Response
    {
        $email = new Email()
            ->from('hello@example.com')
            ->to('you@example.com')
            //->cc('cc@example.com')
            //->bcc('bcc@example.com')
            //->replyTo('fabien@example.com')
            //->priority(Email::PRIORITY_HIGH)
            ->subject('Time for Symfony Mailer!')
            ->text('Sending emails is fun again!')
            ->html('<p>See Twig integration for better HTML integration!</p>');

        $mailer->send($email);

        // ...
    }
}
```

### 电子邮件地址

所有需要电子邮件地址的方法（`from()`、`to()` 等）既接受字符串也接受地址对象：

```php
use Symfony\Component\Mime\Address;

$email = new Email()
    ->from('fabien@example.com')
    ->from(new Address('fabien@example.com'))
    ->from(new Address('fabien@example.com', 'Fabien'))
    ->from(Address::create('Fabien Potencier <fabien@example.com>'))
    // ...
;
```

### 消息头

消息包含许多描述其内容的头字段。Symfony 自动设置所有必需的头，但你也可以设置自己的头：

```php
$email = new Email()
    ->getHeaders()
        ->addTextHeader('X-Auto-Response-Suppress', 'OOF, DR, RN, NRN, AutoReply')
        ->addIdHeader('References', ['123@example.com', '456@example.com'])
;
```

### 消息内容

电子邮件消息的文本和 HTML 内容可以是字符串或 PHP 资源：

```php
$email = new Email()
    // ...
    ->text('Lorem ipsum...')
    ->html('<p>Lorem ipsum...</p>')

    // 附加文件流
    ->text(fopen('/path/to/emails/user_signup.txt', 'r'))
    ->html(fopen('/path/to/emails/user_signup.html', 'r'))
;
```

### 文件附件

使用 `addPart()` 方法附加文件：

```php
use Symfony\Component\Mime\Part\DataPart;
use Symfony\Component\Mime\Part\File;

$email = new Email()
    // ...
    ->addPart(new DataPart(new File('/path/to/documents/terms-of-use.pdf')))
    ->addPart(new DataPart(new File('/path/to/documents/privacy.pdf'), 'Privacy Policy'))
    ->addPart(new DataPart(new File('/path/to/documents/contract.doc'), 'Contract', 'application/msword'))
;
```

### 嵌入图片

如果想在电子邮件中显示图片，必须嵌入它们而不是作为附件添加：

```php
$email = new Email()
    // ...
    ->addPart(new DataPart(fopen('/path/to/images/logo.png', 'r'), 'logo', 'image/png')->asInline())
    ->addPart(new DataPart(new File('/path/to/images/signature.gif'), 'footer-signature', 'image/gif')->asInline())

    // 使用 'cid:' + "图片嵌入名称" 语法引用图片
    ->html('<img src="cid:logo"> ... <img src="cid:footer-signature"> ...')
;
```

---

## 全局配置电子邮件 {#mailer-configure-email-globally}

你可以全局配置 `from`、`to` 和 `headers` 值，而不是在每个 Email 上调用它们：

```yaml
# config/packages/mailer.yaml
framework:
    mailer:
        envelope:
            sender: 'fabien@example.com'
            recipients: ['foo@example.com', 'bar@example.com']
        headers:
            From: 'Fabien <fabien@example.com>'
            Bcc: 'baz@example.com'
            X-Custom-Header: 'foobar'
```

---

## 处理发送失败

如果将电子邮件交给传输时出错，Symfony 会抛出 `TransportExceptionInterface`：

```php
use Symfony\Component\Mailer\Exception\TransportExceptionInterface;

$email = new Email();
// ...
try {
    $mailer->send($email);
} catch (TransportExceptionInterface $e) {
    // 某些错误阻止了电子邮件发送；显示错误消息或尝试重新发送
}
```

---

## Twig：HTML 和 CSS {#mailer-twig}

### HTML 内容

使用 `TemplatedEmail` 类用 Twig 定义电子邮件内容：

```php
use Symfony\Bridge\Twig\Mime\TemplatedEmail;

$email = new TemplatedEmail()
    ->from('fabien@example.com')
    ->to(new Address('ryan@example.com'))
    ->subject('Thanks for signing up!')

    // Twig 模板的路径
    ->htmlTemplate('emails/signup.html.twig')

    // 传递变量（name => value）给模板
    ->context([
        'expiration_date' => new \DateTime('+7 days'),
        'username' => 'foo',
    ])
;
```

然后，创建模板：

```html+twig
{# templates/emails/signup.html.twig #}
<h1>Welcome {{ email.toName }}!</h1>

<p>
    You signed up as {{ username }} the following email:
</p>
<p><code>{{ email.to[0].address }}</code></p>

<p>
    <a href="#">Activate your account</a>
    (this link is valid until {{ expiration_date|date('F jS') }})
</p>
```

### 嵌入图片 {#mailer-twig-embedding-images}

在 Twig 模板中，使用特殊的 `email.image()` Twig 辅助函数嵌入图片：

```html+twig
{# '@images/' 是指之前定义的 Twig 命名空间 #}
<img src="{{ email.image('@images/logo.png') }}" alt="Logo">
```

### 内联 CSS 样式 {#mailer-inline-css}

许多电子邮件客户端不支持在 `<style>...</style>` 部分定义样式，你必须**内联所有 CSS 样式**。使用 `CssInlinerExtension`：

```terminal
$ composer require twig/extra-bundle twig/cssinliner-extra
```

```html+twig
{% apply inline_css %}
    <style>
        h1 {
            color: #333;
        }
    </style>

    <h1>Welcome {{ email.toName }}!</h1>
    {# ... #}
{% endapply %}
```

### 渲染 Markdown 内容 {#mailer-markdown}

```terminal
$ composer require twig/extra-bundle twig/markdown-extra league/commonmark
```

```twig
{% apply markdown_to_html %}
    Welcome {{ email.toName }}!
    ===========================

    You signed up to our site using the following email:
    `{{ email.to[0].address }}`

    [Activate your account]({{ url('...') }})
{% endapply %}
```

---

## 签名和加密消息 {#signing-and-encrypting-messages}

### 签名消息

**S/MIME 签名：**

```php
use Symfony\Component\Mime\Crypto\SMimeSigner;

$signer = new SMimeSigner('/path/to/certificate.crt', '/path/to/certificate-private-key.key');
$signedEmail = $signer->sign($email);
```

**DKIM 签名：**

```php
use Symfony\Component\Mime\Crypto\DkimSigner;

$signer = new DkimSigner('file:///path/to/private-key.key', 'example.com', 'sf');
$signedEmail = $signer->sign($email);
```

### 加密消息

```php
use Symfony\Component\Mime\Crypto\SMimeEncrypter;

$encrypter = new SMimeEncrypter('/path/to/certificate.crt');
$encryptedEmail = $encrypter->encrypt($email);
```

---

## 多个电子邮件传输 {#multiple-email-transports}

```yaml
# config/packages/mailer.yaml
framework:
    mailer:
        transports:
            main: '%env(MAILER_DSN)%'
            alternative: '%env(MAILER_DSN_IMPORTANT)%'
```

通过添加 `X-Transport` 头来选择其他传输：

```php
// 使用第一个传输（"main"）发送：
$mailer->send($email);

// 使用传输 "alternative"：
$email->getHeaders()->addTextHeader('X-Transport', 'alternative');
$mailer->send($email);
```

---

## 异步发送消息 {#mailer-sending-messages-async}

调用 `$mailer->send($email)` 时，电子邮件立即发送到传输。要提高性能，你可以利用 Messenger 稍后通过 Messenger 传输发送消息：

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            async: "%env(MESSENGER_TRANSPORT_DSN)%"

        routing:
            'Symfony\Component\Mailer\Messenger\SendEmailMessage': async
```

---

## 添加标签和元数据

```php
use Symfony\Component\Mailer\Header\MetadataHeader;
use Symfony\Component\Mailer\Header\TagHeader;

$email->getHeaders()->add(new TagHeader('password-reset'));
$email->getHeaders()->add(new MetadataHeader('Color', 'blue'));
$email->getHeaders()->add(new MetadataHeader('Client-ID', '12345'));
```

---

## Mailer 事件

### MessageEvent

`MessageEvent` 允许在发送电子邮件之前更改 Mailer 消息和信封：

```php
use Symfony\Component\Mailer\Event\MessageEvent;
use Symfony\Component\Mime\Email;

public function onMessage(MessageEvent $event): void
{
    $message = $event->getMessage();
    if (!$message instanceof Email) {
        return;
    }
    // 对消息做一些处理（日志记录等）

    // 和/或添加一些 Messenger 标记
    $event->addStamp(new SomeMessengerStamp());
}
```

### SentMessageEvent {#mailer-sent-message-event}

`SentMessageEvent` 允许访问 `SentMessage` 类，获取原始消息和调试信息：

```php
use Symfony\Component\Mailer\Event\SentMessageEvent;

public function onMessage(SentMessageEvent $event): void
{
    $message = $event->getMessage();
    // 对消息做一些处理（例如获取其 id）
}
```

### FailedMessageEvent {#mailer-failed-message-event}

`FailedMessageEvent` 允许在失败时对初始消息进行操作：

```php
use Symfony\Component\Mailer\Event\FailedMessageEvent;
use Symfony\Component\Mailer\Exception\TransportExceptionInterface;

public function onMessage(FailedMessageEvent $event): void
{
    $error = $event->getError();
    if ($error instanceof TransportExceptionInterface) {
        $error->getDebug();
    }
}
```

---

## 开发和调试

### 启用电子邮件捕获器 {#mail-catcher}

在本地开发时，建议使用电子邮件捕获器。发送测试电子邮件：

```terminal
$ php bin/console mailer:test someone@example.com
```

### 禁用投递

在开发（或测试）时，你可能想要完全禁用消息投递：

```yaml
# config/packages/mailer.yaml
when@dev:
    framework:
        mailer:
            dsn: 'null://null'
```

### 始终发送到相同地址

在开发中，始终发送到特定地址而不是真实地址：

```yaml
# config/packages/mailer.yaml
when@dev:
    framework:
        mailer:
            envelope:
                recipients: ['youremail@example.com']
```

### 编写功能测试

Symfony 提供了许多内置的 mailer 断言来功能性测试电子邮件是否发送：

```php
class MailControllerTest extends WebTestCase
{
    public function testMailIsSentAndContentIsOk(): void
    {
        $client = static::createClient();
        $client->request('GET', '/mail/send');
        self::assertResponseIsSuccessful();

        self::assertEmailCount(1);

        $email = self::getMailerMessage();

        self::assertEmailHtmlBodyContains($email, 'Welcome');
        self::assertEmailTextBodyContains($email, 'Welcome');
    }
}
```
