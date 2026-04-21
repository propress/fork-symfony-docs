# Webhook

Webhook 是一种在系统之间发送事件通知的机制，通常通过 HTTP POST 请求传递。

Webhook 组件提供两项主要功能：

1. **消费（Consuming）**：接收并处理来自远程系统的 webhook 调用；
2. **发送（Sending）**：当事件发生时，向已注册的端点分发 webhook 回调。

## 安装

```bash
$ composer require symfony/webhook
```

## 消费 Webhook

Webhook 组件结合 RemoteEvent，通过三个阶段让你接收并处理 webhook：

1. 通过专用端点接收 webhook
2. 验证 webhook 并将其转换为 RemoteEvent 对象
3. 在应用程序逻辑中消费该事件

### 统一的 Webhook 端点

`Symfony\Component\Webhook\Controller\WebhookController` 为接收所有传入 webhook 提供单一入口点，无论其来源（第三方服务、自定义 API 等）。

默认情况下，以 `/webhook` 为前缀的所有 URL 都会路由到此控制器。你可以在路由配置中自定义该前缀：

```yaml
# config/routes/webhook.yaml
webhook:
    resource: '@FrameworkBundle/Resources/config/routing/webhook.php'
    prefix: /webhook  # 按需自定义
```

```php
// config/routes/webhook.php
use Symfony\Component\Routing\Loader\Configurator\RoutingConfigurator;

return static function (RoutingConfigurator $routes): void {
    $routes->import('@FrameworkBundle/Resources/config/routing/webhook.php')
        ->prefix('/webhook');
};
```

接下来，配置用于处理传入 webhook 的解析器服务。控制器使用路由机制将传入请求映射到对应的解析器：

```yaml
# config/packages/webhook.yaml
framework:
    webhook:
        routing:
            acme_webhook:  # 路由名称，映射到 /webhook/acme_webhook
                service: App\Webhook\AcmeWebhookRequestParser
                secret: '%env(WEBHOOK_SECRET)%'  # 可选
```

```php
// config/packages/framework.php
use Symfony\Config\FrameworkConfig;

return static function (FrameworkConfig $config): void {
    $config->webhook()
        ->routing('acme_webhook')
        ->service('App\Webhook\AcmeWebhookRequestParser')
        ->secret('%env(WEBHOOK_SECRET)%');
};
```

路由名称将成为 webhook URL 的一部分（例如 `https://example.com/webhook/acme_webhook`）。每个路由名称必须唯一，因为它将 webhook 来源与你的消费者代码相关联。

所有解析器都会自动注入到 WebhookController 中。

### 解析 Webhook 请求

一旦 webhook 请求到达你的端点，在应用程序处理之前必须对其进行解析和验证。解析包括验证请求的真实性（通常通过签名验证）、提取负载，并将其转换为 `Symfony\Component\RemoteEvent\RemoteEvent` 对象。

Symfony 提供两种处理解析的方式：

* **内置解析器**：对来自其他 Symfony 应用程序的 webhook，使用标准的 `Symfony\Component\Webhook\Client\RequestParser`；
* **自定义解析器**：为来自第三方服务或自定义 API 的 webhook 创建自己的解析器。

#### 使用内置解析器

对于来自其他 Symfony 应用程序的 webhook，你可以使用内置的 `Symfony\Component\Webhook\Client\RequestParser`，而无需创建自定义解析器。该解析器处理标准的 Symfony webhook 请求格式：

```yaml
# config/packages/framework.yaml
framework:
    webhook:
        routing:
            acme_webhook:
                service: Symfony\Component\Webhook\Client\RequestParser
                secret: '%env(WEBHOOK_SECRET)%'
```

```php
// config/packages/framework.php
use Symfony\Config\FrameworkConfig;

return static function (FrameworkConfig $config): void {
    $config->webhook()
        ->routing('acme_webhook')
        ->service(Symfony\Component\Webhook\Client\RequestParser::class)
        ->secret('%env(WEBHOOK_SECRET)%');
};
```

内置解析器自动处理请求验证和签名校验，让你专注于在应用程序逻辑中消费 RemoteEvent。

#### 创建自定义解析器

对于来自自定义 API 的 webhook，可实现 `Symfony\Component\Webhook\Client\RequestParserInterface` 或继承 `Symfony\Component\Webhook\Client\AbstractRequestParser`。

最简单的方式是使用 maker 命令：

```bash
$ php bin/console make:webhook
```

> **提示：** `make:webhook` 命令会自动生成解析器和消费者类，并更新你的配置。

继承 `Symfony\Component\Webhook\Client\AbstractRequestParser` 时，需要实现两个方法：

* `Symfony\Component\Webhook\Client\AbstractRequestParser::getRequestMatcher`：验证传入请求的格式；
* `Symfony\Component\Webhook\Client\AbstractRequestParser::doParse`：验证 webhook 并将其解析为 RemoteEvent。

```php
// src/Webhook/AcmeWebhookRequestParser.php
namespace App\Webhook;

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\RequestMatcher\ChainRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcher\IsJsonRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcher\MethodRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcher\RequestMatcherInterface;
use Symfony\Component\RemoteEvent\RemoteEvent;
use Symfony\Component\Webhook\Client\AbstractRequestParser;

final class AcmeWebhookRequestParser extends AbstractRequestParser
{
    protected function getRequestMatcher(): RequestMatcherInterface
    {
        return new ChainRequestMatcher([
            new IsJsonRequestMatcher(),
            new MethodRequestMatcher('POST'),
        ]);
    }

    protected function doParse(
        Request $request,
        #[\SensitiveParameter] string $secret
    ): ?RemoteEvent {
        $payload = $request->toArray();
        return new RemoteEvent(
            $payload['event_type'],
            $payload['event_id'],
            $payload,
        );
    }
}
```

`doParse()` 方法接收请求和密钥。你应该：

* 验证请求签名（通常为 HMAC-SHA256）
* 解析并验证负载
* 对无效请求抛出 `Symfony\Component\Webhook\Exception\RejectWebhookException`
* 成功时返回 `Symfony\Component\RemoteEvent\RemoteEvent`

#### 测试你的解析器

通过继承 `Symfony\Component\Webhook\Test\AbstractRequestParserTestCase` 来测试你的自定义解析器。该基类使用来自 `Symfony\Component\Webhook\Test\AbstractRequestParserTestCase::getPayloads` 的数据运行 `Symfony\Component\Webhook\Test\AbstractRequestParserTestCase::testParse`，从 `Fixtures/*.json` 加载文件，并将每个文件与对应的 `.php` 期望文件配对：

```php
// tests/Webhook/AcmeWebhookRequestParserTest.php
namespace App\Tests\Webhook;

use App\Webhook\AcmeWebhookRequestParser;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Webhook\Test\AbstractRequestParserTestCase;

class AcmeWebhookRequestParserTest extends AbstractRequestParserTestCase
{
    protected function createRequestParser(): AcmeWebhookRequestParser
    {
        return new AcmeWebhookRequestParser();
    }

    // 默认 createRequest() 构建一个 Content-Type: application/json 的 POST 请求
    // 如需添加特定于提供商的请求头（如 webhook 签名）或更改方法，请覆盖此方法
    protected function createRequest(string $payload): Request
    {
        return Request::create('/', 'POST', [], [], [], // 实际上不测试路由
            [
                'CONTENT_TYPE' => 'application/json', // 按需添加请求头
            ],
            $payload
        );
    }
}
```

创建基础测试所需的 fixture 文件（例如 `tests/Webhook/Fixtures/resource.created.json`）：

```json
{
    "event_type": "resource.created",
    "event_id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com"
}
```

以及：

```php
// tests/Webhook/Fixtures/resource.created.php
use Symfony\Component\RemoteEvent\RemoteEvent;

return new RemoteEvent(
    name: 'resource.created',
    id: '550e8400-e29b-41d4-a716-446655440000',
    payload: [
        'event_type' => 'resource.created',
        'event_id' => '550e8400-e29b-41d4-a716-446655440000',
        'email' => 'user@example.com',
    ]
);
```

你的测试必须实现 `Symfony\Component\Webhook\Test\AbstractRequestParserTestCase::createRequestParser` 来返回你的 `Symfony\Component\Webhook\Client\RequestParserInterface` 实现的实例。

你还可以在测试中覆盖以下方法：

* `Symfony\Component\Webhook\Test\AbstractRequestParserTestCase::getSecret`：如果你的解析器需要验证签名
* `Symfony\Component\Webhook\Test\AbstractRequestParserTestCase::getFixtureExtension`：如果你的 fixture 不是 `.json`（例如，表单编码负载使用 `.txt`）

#### 处理复杂的负载转换

对于复杂的 webhook 负载，使用 `Symfony\Component\RemoteEvent\PayloadConverterInterface` 来封装转换逻辑：

```php
// src/RemoteEvent/AcmeWebhookPayloadConverter.php
namespace App\RemoteEvent;

use Symfony\Component\RemoteEvent\PayloadConverterInterface;
use Symfony\Component\RemoteEvent\RemoteEvent;

final class AcmeWebhookPayloadConverter implements PayloadConverterInterface
{
    public function convert(array $payload): RemoteEvent
    {
        // 将外部事件名称映射到你的领域事件
        $eventName = match ($payload['event_type']) {
            'resource.created' => 'acme.resource_created',
            'resource.updated' => 'acme.resource_updated',
            'resource.deleted' => 'acme.resource_deleted',
            default => 'acme.unknown_event',
        };

        return new RemoteEvent($eventName, $payload['event_id'], $payload);
    }
}
```

然后将其注入到你的解析器中：

```php
// src/Webhook/AcmeWebhookRequestParser.php
namespace App\Webhook;

use App\RemoteEvent\AcmeWebhookPayloadConverter;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\RemoteEvent\PayloadConverterInterface;
use Symfony\Component\RemoteEvent\RemoteEvent;
use Symfony\Component\Webhook\Client\AbstractRequestParser;
use Symfony\Component\Webhook\Exception\RejectWebhookException;
use Symfony\DependencyInjection\Attribute\Autowire;

final class AcmeWebhookRequestParser extends AbstractRequestParser
{
    public function __construct(
        #[Autowire(service: AcmeWebhookPayloadConverter::class)]
        private readonly PayloadConverterInterface $converter,
    ) {
    }

    // ... getRequestMatcher() 同前

    protected function doParse(
        Request $request,
        #[\SensitiveParameter] string $secret,
    ): ?RemoteEvent {
        try {
            return $this->converter->convert($request->toArray());
        } catch (ParseException|\JsonException $e) {
            throw new RejectWebhookException(406, $e->getMessage(), $e);
        }
    }
}
```

> **提示：** 可以参考内置的 `Symfony\Component\Mailer\Bridge\Mailgun\RemoteEvent\MailgunPayloadConverter` 作为灵感。

### 消费 RemoteEvent

无论是同步处理还是异步处理（通过 Messenger），你都需要一个实现了 `Symfony\Component\RemoteEvent\Consumer\ConsumerInterface` 的消费者。

`make:webhook` 命令会自动生成一个。否则，使用 `Symfony\Component\RemoteEvent\Attribute\AsRemoteEventConsumer` 属性手动创建：

```php
// src/RemoteEvent/AcmeWebhookConsumer.php
namespace App\RemoteEvent;

use Symfony\Component\RemoteEvent\Attribute\AsRemoteEventConsumer;
use Symfony\Component\RemoteEvent\Consumer\ConsumerInterface;
use Symfony\Component\RemoteEvent\RemoteEvent;

#[AsRemoteEventConsumer('acme_webhook')]  // 必须与路由名称匹配
final class AcmeWebhookConsumer implements ConsumerInterface
{
    public function consume(RemoteEvent $event): void
    {
        // 根据业务逻辑处理事件
    }
}
```

传递给 `AsRemoteEventConsumer` 属性的名称必须与 webhook 配置中定义的路由名称匹配。

#### 异步消费

默认情况下，webhook 消费者在 RemoteEvent 被分发时同步调用。要异步处理 webhook，需为 `Symfony\Component\RemoteEvent\Messenger\ConsumeRemoteEventMessage` 配置 Messenger 路由：

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        routing:
            'Symfony\Component\RemoteEvent\Messenger\ConsumeRemoteEventMessage': async
```

```php
// config/packages/messenger.php
use Symfony\Component\RemoteEvent\Messenger\ConsumeRemoteEventMessage;
use Symfony\Config\FrameworkConfig;

return static function (FrameworkConfig $config): void {
    $config->messenger()
        ->routing(ConsumeRemoteEventMessage::class)
        ->senders(['async']);
};
```

使用此配置，消费者通过消息总线异步调用。不使用时，消费者在 webhook 请求期间同步处理。

### 内置集成

Symfony 为常见服务提供了预构建的解析器，因此你无需为它们创建自定义解析器。你仍然需要创建自己的消费者，根据业务逻辑处理 RemoteEvent。

#### 邮件 Webhook

从第三方邮件服务接收投递和互动通知：

| 邮件服务      | 解析器服务名称                                    |
|--------------|--------------------------------------------------|
| AhaSend      | `mailer.webhook.request_parser.ahasend`          |
| Brevo        | `mailer.webhook.request_parser.brevo`            |
| Mandrill     | `mailer.webhook.request_parser.mailchimp`        |
| MailerSend   | `mailer.webhook.request_parser.mailersend`       |
| Mailgun      | `mailer.webhook.request_parser.mailgun`          |
| Mailjet      | `mailer.webhook.request_parser.mailjet`          |
| Mailomat     | `mailer.webhook.request_parser.mailomat`         |
| Mailtrap     | `mailer.webhook.request_parser.mailtrap`         |
| Postmark     | `mailer.webhook.request_parser.postmark`         |
| Resend       | `mailer.webhook.request_parser.resend`           |
| Sendgrid     | `mailer.webhook.request_parser.sendgrid`         |
| Sweego       | `mailer.webhook.request_parser.sweego`           |

> **注意：** 请按照 Mailer 组件文档中的说明安装你想要使用的第三方邮件提供商。本文档以 Mailgun 作为提供商示例。

配置路由：

```yaml
# config/packages/framework.yaml
framework:
    webhook:
        routing:
            mailer_mailgun:
                service: 'mailer.webhook.request_parser.mailgun'
                secret: '%env(MAILER_MAILGUN_SECRET)%'
```

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'webhook' => [
            'routing' => [
                'mailer_mailgun' => [
                    'service' => 'mailer.webhook.request_parser.mailgun',
                    'secret' => env('MAILER_MAILGUN_SECRET'),
                ],
            ],
        ],
    ],
]);
```

路由名称将成为你的 webhook URL 的一部分（例如 `https://example.com/webhook/mailer_mailgun`）。在你的邮件提供商处配置该 URL，并通过密钥管理系统或 `.env` 文件将 webhook 密钥存储到你的环境变量中。

然后创建一个消费者来处理投递和互动事件：

```php
// src/RemoteEvent/MailerWebhookConsumer.php
namespace App\RemoteEvent;

use Symfony\Component\RemoteEvent\Attribute\AsRemoteEventConsumer;
use Symfony\Component\RemoteEvent\Consumer\ConsumerInterface;
use Symfony\Component\RemoteEvent\Event\Mailer\MailerDeliveryEvent;
use Symfony\Component\RemoteEvent\Event\Mailer\MailerEngagementEvent;
use Symfony\Component\RemoteEvent\RemoteEvent;

#[AsRemoteEventConsumer('mailer_mailgun')]
final class MailerWebhookConsumer implements ConsumerInterface
{
    public function consume(RemoteEvent $event): void
    {
        if ($event instanceof MailerDeliveryEvent) {
            $this->handleDelivery($event);
        } elseif ($event instanceof MailerEngagementEvent) {
            $this->handleEngagement($event);
        }
    }

    private function handleDelivery(MailerDeliveryEvent $event): void
    {
        // 在数据库中更新消息状态、记录投递日志等
    }

    private function handleEngagement(MailerEngagementEvent $event): void
    {
        // 处理打开、点击、退信等
    }
}
```

#### 通知 Webhook

从提供商接收短信状态通知：

| 短信服务  | 解析器服务名称                                      |
|----------|---------------------------------------------------|
| LOX24    | `notifier.webhook.request_parser.lox24`           |
| Smsbox   | `notifier.webhook.request_parser.smsbox`          |
| Sweego   | `notifier.webhook.request_parser.sweego`          |
| Twilio   | `notifier.webhook.request_parser.twilio`          |
| Vonage   | `notifier.webhook.request_parser.vonage`          |

配置方式与邮件类似，然后消费 `Symfony\Component\RemoteEvent\Event\Sms\SmsEvent`：

```php
// src/RemoteEvent/SmsWebhookConsumer.php
namespace App\RemoteEvent;

use Symfony\Component\RemoteEvent\Attribute\AsRemoteEventConsumer;
use Symfony\Component\RemoteEvent\Consumer\ConsumerInterface;
use Symfony\Component\RemoteEvent\Event\Sms\SmsEvent;
use Symfony\Component\RemoteEvent\RemoteEvent;

#[AsRemoteEventConsumer('notifier_twilio')]
final class SmsWebhookConsumer implements ConsumerInterface
{
    public function consume(RemoteEvent $event): void
    {
        if ($event instanceof SmsEvent) {
            $this->handleSms($event);
        }
    }

    private function handleSms(SmsEvent $event): void
    {
        // 在数据库中更新短信投递状态等
    }
}
```

## 发送 Webhook

Webhook 组件还允许你的应用程序向远程端点分发 webhook 回调。当你构建需要通知订阅者重要事件的 API 时，这非常有用。

要发送 webhook，请确保已安装 HttpClient 和 Serializer 组件：

```bash
$ composer require symfony/http-client symfony/serializer
```

### 基本用法

要发送 webhook，通过 Messenger 组件分发 `Symfony\Component\Webhook\Messenger\SendWebhookMessage`：

```php
use Symfony\Component\Messenger\MessageBusInterface;
use Symfony\Component\RemoteEvent\RemoteEvent;
use Symfony\Component\Webhook\Messenger\SendWebhookMessage;
use Symfony\Component\Webhook\Subscriber;

class StockNotifier
{
    public function __construct(
        private readonly MessageBusInterface $messageBus,
    ) {
    }

    public function notifyOutOfStock(int $productId): void
    {
        $subscriber = new Subscriber(
            url: 'https://example.com/webhook/stock',
            secret: 'your-shared-secret',
        );

        $event = new RemoteEvent(
            name: 'resource.created',
            id: '550e8400-e29b-41d4-a716-446655440000',
            payload: [
                'resource_id' => 12345,
                'email' => 'user@example.com',
                'created_at' => time(),
            ]
        );

        $this->messageBus->dispatch(
            new SendWebhookMessage($subscriber, $event)
        );
    }
}
```

该消息由 `Symfony\Component\Webhook\Messenger\SendWebhookHandler` 处理，它会：

1. 构造 HTTP 请求体（JSON 编码的负载）
2. 添加标准请求头：`Webhook-Event`（事件名称）、`Webhook-Id`（事件 ID）、`Webhook-Signature`（事件名称、ID 和请求体拼接后的 HMAC-SHA256 签名）以及 `Content-Type: application/json`
3. 使用订阅者的密钥对请求进行签名
4. 使用 Symfony HttpClient 组件发送 HTTP 请求

### 发送的 HTTP 请求

发送 webhook 时，会生成如下格式的 HTTP POST 请求：

```text
POST /webhook/symfony HTTP/1.1
Host: example.com
Content-Type: application/json
Webhook-Event: resource.created
Webhook-Id: 550e8400-e29b-41d4-a716-446655440000
Webhook-Signature: sha256=9f86d081884c7d6d9ffd60bb51d3263112c4b2486f80fa12ab5807265dc789d6

{
    "resource_id": 12345,
    "email": "user@example.com",
    "created_at": 1234567890
}
```

默认情况下，签名使用事件名称、事件 ID 和 JSON 请求体拼接后的 HMAC-SHA256。接收端点应使用共享密钥验证此签名，以确保 webhook 的真实性。

### 自定义发送逻辑

对于高级用例，你可以实现 `Symfony\Component\Webhook\Server\TransportInterface` 来自定义发送逻辑，以控制请求头生成、签名和 HTTP 传输。
