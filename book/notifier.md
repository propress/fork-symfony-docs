# 创建和发送通知

## 安装

```terminal
$ composer require symfony/notifier
```

---

## 通知渠道

Notifier 组件支持以下渠道：

- **短信渠道**：通过短信向手机发送通知
- **聊天渠道**：向 Slack 和 Telegram 等聊天服务发送通知
- **电子邮件渠道**：集成 Symfony Mailer
- **浏览器渠道**：使用 flash 消息
- **推送渠道**：通过推送通知向手机和浏览器发送通知
- **桌面渠道**：在同一台主机上显示桌面通知

---

## 短信渠道 {#notifier-sms-channel}

短信渠道使用 Texter 类向手机发送短信。主要提供商包括：

| 服务 | 安装 |
|------|------|
| Twilio | `composer require symfony/twilio-notifier` |
| AmazonSns | `composer require symfony/amazon-sns-notifier` |
| Brevo | `composer require symfony/brevo-notifier` |
| Vonage | `composer require symfony/vonage-notifier` |
| OvhCloud | `composer require symfony/ovh-cloud-notifier` |

> **提示**：使用 [Symfony 配置 Secrets](configuration/secrets.md) 安全地存储你的 API 令牌。

配置短信传输：

```env
# .env
TWILIO_DSN=twilio://SID:TOKEN@default?from=FROM
```

```yaml
# config/packages/notifier.yaml
framework:
    notifier:
        texter_transports:
            twilio: '%env(TWILIO_DSN)%'
```

发送短信消息：

```php
// src/Controller/SecurityController.php
use Symfony\Component\Notifier\Message\SmsMessage;
use Symfony\Component\Notifier\TexterInterface;

class SecurityController
{
    #[Route('/login/success')]
    public function loginSuccess(TexterInterface $texter): Response
    {
        $sms = new SmsMessage(
            '+1411111111',
            'A new login was detected!',
        );

        $sentMessage = $texter->send($sms);
        // ...
    }
}
```

---

## 聊天渠道 {#notifier-chat-channel}

主要聊天服务集成：

| 服务 | 安装 |
|------|------|
| Slack | `composer require symfony/slack-notifier` |
| Telegram | `composer require symfony/telegram-notifier` |
| Discord | `composer require symfony/discord-notifier` |
| GoogleChat | `composer require symfony/google-chat-notifier` |
| MicrosoftTeams | `composer require symfony/microsoft-teams-notifier` |

```env
# .env
SLACK_DSN=slack://TOKEN@default?channel=CHANNEL
```

```yaml
# config/packages/notifier.yaml
framework:
    notifier:
        chatter_transports:
            slack: '%env(SLACK_DSN)%'
```

发送聊天消息：

```php
use Symfony\Component\Notifier\ChatterInterface;
use Symfony\Component\Notifier\Message\ChatMessage;

class CheckoutController extends AbstractController
{
    #[Route('/checkout/thankyou')]
    public function thankyou(ChatterInterface $chatter): Response
    {
        $message = new ChatMessage('You got a new invoice for 15 EUR.')
            ->transport('slack');

        $sentMessage = $chatter->send($message);
        // ...
    }
}
```

> **警告**：如果安装了 Messenger 组件，通知将通过消息总线发送。如果没有运行消息消费者，消息将永远不会发送。要直接发送消息，在配置中添加 `message_bus: false`。

---

## 电子邮件渠道 {#notifier-email-channel}

```terminal
$ composer require symfony/twig-pack twig/cssinliner-extra twig/inky-extra
```

```yaml
# config/packages/mailer.yaml
framework:
    mailer:
        dsn: '%env(MAILER_DSN)%'
        envelope:
            sender: 'notifications@example.com'
```

---

## 推送渠道 {#notifier-push-channel}

主要推送服务：

| 服务 | 安装 |
|------|------|
| Expo | `composer require symfony/expo-notifier` |
| OneSignal | `composer require symfony/one-signal-notifier` |
| Pushover | `composer require symfony/pushover-notifier` |

---

## 桌面渠道 {#notifier-desktop-channel}

```terminal
$ composer require symfony/joli-notif-notifier
```

```php
use Symfony\Component\Notifier\Message\DesktopMessage;

$message = new DesktopMessage(
    'New subscription! 🎉',
    sprintf('%s is a new subscriber', $user->getFullName())
);
$this->texter->send($message);
```

---

## 配置故障转移或轮询传输

使用 `||` 和 `&&` 实现故障转移或轮询传输：

```yaml
# config/packages/notifier.yaml
framework:
    notifier:
        chatter_transports:
            # 发送到 Slack，如果 Slack 出错则使用 Telegram
            main: '%env(SLACK_DSN)% || %env(TELEGRAM_DSN)%'

            # 按轮询顺序发送到下一个传输
            roundrobin: '%env(SLACK_DSN)% && %env(TELEGRAM_DSN)%'
```

---

## 创建和发送通知

注入 `NotifierInterface` 来发送通知：

```php
// src/Controller/InvoiceController.php
use Symfony\Component\Notifier\Notification\Notification;
use Symfony\Component\Notifier\NotifierInterface;
use Symfony\Component\Notifier\Recipient\Recipient;

class InvoiceController extends AbstractController
{
    #[Route('/invoice/create')]
    public function create(NotifierInterface $notifier): Response
    {
        // 创建要通过 "email" 渠道发送的通知
        $notification = new Notification('New Invoice', ['email'])
            ->content('You got a new invoice for 15 EUR.');

        // 通知的接收者
        $recipient = new Recipient(
            $user->getEmail(),
            $user->getPhonenumber()
        );

        // 向接收者发送通知
        $notifier->send($notification, $recipient);
        // ...
    }
}
```

---

## 配置渠道策略

使用重要性级别替代在创建时指定目标渠道：

```yaml
# config/packages/notifier.yaml
framework:
    notifier:
        channel_policy:
            # 对紧急通知使用短信、Slack 和电子邮件
            urgent: ['sms', 'chat/slack', 'email']

            # 对高重要性通知使用 Slack
            high: ['chat/slack']

            # 对中低重要性通知使用浏览器
            medium: ['browser']
            low: ['browser']
```

```php
$notification = new Notification('New Invoice')
    ->content('You got a new invoice for 15 EUR.')
    ->importance(Notification::IMPORTANCE_HIGH);

$notifier->send($notification, new Recipient('user@example.com'));
```

---

## 自定义通知

扩展 `Notification` 类以自定义其行为：

```php
namespace App\Notifier;

use Symfony\Component\Notifier\Notification\Notification;
use Symfony\Component\Notifier\Recipient\RecipientInterface;
use Symfony\Component\Notifier\Recipient\SmsRecipientInterface;

class InvoiceNotification extends Notification
{
    public function __construct(
        private int $price,
    ) {
    }

    public function getChannels(RecipientInterface $recipient): array
    {
        if ($this->price > 10000 && $recipient instanceof SmsRecipientInterface) {
            return ['sms'];
        }

        return ['email'];
    }
}
```

实现 `ChatNotificationInterface` 自定义聊天消息：

```php
class InvoiceNotification extends Notification implements ChatNotificationInterface
{
    public function asChatMessage(RecipientInterface $recipient, ?string $transport = null): ?ChatMessage
    {
        if ('slack' === $transport) {
            $this->subject('You\'re invoiced '.strval($this->price).' EUR.');
            $this->emoji("money");
            return ChatMessage::fromNotification($this);
        }

        return null;
    }
}
```

---

## 禁用通知投递（开发环境）

```yaml
# config/packages/notifier.yaml
when@dev:
    framework:
        notifier:
            texter_transports:
                twilio: 'null://null'
            chatter_transports:
                slack: 'null://null'
```

---

## 使用事件

Notifier 传输类在消息生命周期中调度以下事件：

- **`MessageEvent`**：发送消息之前（用于日志记录等）
- **`FailedMessageEvent`**：发送消息失败时（用于重试或额外日志）
- **`SentMessageEvent`**：消息成功发送后（用于检索消息 ID）

```php
use Symfony\Component\Notifier\Event\SentMessageEvent;

$dispatcher->addListener(SentMessageEvent::class, function (SentMessageEvent $event): void {
    $message = $event->getMessage();
    // 消息 ID 等信息
});
```
