# Messenger：同步与队列消息处理

Messenger 提供了一个消息总线，具有发送消息然后在应用程序中立即处理它们或通过传输（如队列）发送以供稍后处理的能力。

## 安装

```terminal
$ composer require symfony/messenger
```

---

## 创建消息和处理器

Messenger 围绕两种不同的类：(1) 保存数据的消息类和 (2) 调度消息时将被调用的处理器类。

**消息类**（对消息类没有特定要求，只要它可以被序列化）：

```php
// src/Message/SmsNotification.php
namespace App\Message;

class SmsNotification
{
    public function __construct(
        private string $content,
    ) {
    }

    public function getContent(): string
    {
        return $this->content;
    }
}
```

**消息处理器**（使用 `#[AsMessageHandler]` 属性）：

```php
// src/MessageHandler/SmsNotificationHandler.php
namespace App\MessageHandler;

use App\Message\SmsNotification;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
class SmsNotificationHandler
{
    public function __invoke(SmsNotification $message)
    {
        // ... 执行一些工作 - 比如发送短信！
    }
}
```

> **提示**：你也可以在单个类的多个方法上使用 `#[AsMessageHandler]` 属性，允许你在一个类中分组处理多个相关类型的消息。

查看所有已配置的处理器：

```terminal
$ php bin/console debug:messenger
```

---

## 调度消息

注入 `MessageBusInterface` 来调度消息：

```php
// src/Controller/DefaultController.php
namespace App\Controller;

use App\Message\SmsNotification;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Messenger\MessageBusInterface;

class DefaultController extends AbstractController
{
    public function index(MessageBusInterface $bus): Response
    {
        // 将导致 SmsNotificationHandler 被调用
        $bus->dispatch(new SmsNotification('Look! I created a message!'));

        // ...
    }
}
```

---

## 传输：异步/队列消息 {#transports-async-queued-messages}

默认情况下，消息在调度时立即处理。如果你想异步处理消息，可以配置传输。

在 `.env` 文件中，取消注释你想使用的传输：

```env
# MESSENGER_TRANSPORT_DSN=amqp://guest:guest@localhost:5672/%2f/messages
# MESSENGER_TRANSPORT_DSN=doctrine://default
# MESSENGER_TRANSPORT_DSN=redis://localhost:6379/messages
```

然后在配置中定义传输：

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            async: "%env(MESSENGER_TRANSPORT_DSN)%"
```

### 将消息路由到传输 {#messenger-routing}

配置了传输后，你可以配置消息发送到传输：

```php
// src/Message/SmsNotification.php
use Symfony\Component\Messenger\Attribute\AsMessage;

#[AsMessage('async')]
class SmsNotification
{
    // ...
}
```

或通过配置：

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            async: "%env(MESSENGER_TRANSPORT_DSN)%"

        routing:
            'App\Message\SmsNotification': async
```

也可以将消息路由到多个传输：

```yaml
routing:
    'My\Message\ToBeSentToTwoSenders': [async, audit]
```

### Doctrine 实体在消息中

如果需要在消息中传递 Doctrine 实体，最好传递实体的主键而不是对象：

```php
// src/Message/NewUserWelcomeEmail.php
class NewUserWelcomeEmail
{
    public function __construct(
        private int $userId,
    ) {
    }
}
```

### 消息类的版本控制 {#messenger-message-versioning}

对于**次要更改**，通过使新的构造函数参数可选来保持向后兼容性。

对于更改消息含义的变更，创建**消息类的新版本**：

```php
final class SendInvoiceV2
{
    public function __construct(
        public readonly int $orderId,
        public readonly string $locale,
        public readonly string $templateId,
    ) {
    }
}
```

---

## 消费消息（运行 Worker）{#messenger-worker}

使用 `messenger:consume` 命令消费消息：

```terminal
$ php bin/console messenger:consume async

# 使用 -vv 查看详情
$ php bin/console messenger:consume async -vv
```

或消费所有可用的接收器：

```terminal
$ php bin/console messenger:consume --all
```

### 生产部署

**使用 Supervisor 或 systemd 保持 worker 运行：**

```ini
;/etc/supervisor/conf.d/messenger-worker.conf
[program:messenger-consume]
command=php /path/to/your/app/bin/console messenger:consume async --time-limit=3600
user=ubuntu
numprocs=2
autostart=true
autorestart=true
```

**部署后重启 worker：**

```terminal
$ php bin/console messenger:stop-workers
```

### 优先传输 {#prioritized-transports}

为具有不同延迟要求的消息类型使用单独的传输：

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            async_priority_high:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                options:
                    queue_name: high
            async_priority_low:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                options:
                    queue_name: low

        routing:
            'App\Message\SmsNotification': async_priority_low
            'App\Message\NewUserWelcomeEmail': async_priority_high
```

按优先顺序运行 worker：

```terminal
$ php bin/console messenger:consume async_priority_high async_priority_low
```

---

## 重试和失败 {#messenger-retries-failures}

如果在从传输消费消息时抛出异常，消息将自动重新发送。默认情况下，消息将重试 3 次，然后被丢弃。

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            async_priority_high:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                retry_strategy:
                    max_retries: 3
                    delay: 1000
                    multiplier: 2
                    max_delay: 10000
                    jitter: 0.1
```

### 避免重试

抛出 `UnrecoverableMessageHandlingException` 使消息不会被重试：

```php
use Symfony\Component\Messenger\Exception\UnrecoverableMessageHandlingException;

throw new UnrecoverableMessageHandlingException('Permanent failure');
```

### 强制重试

抛出 `RecoverableMessageHandlingException` 使消息始终被无限重试：

```php
use Symfony\Component\Messenger\Exception\RecoverableMessageHandlingException;

throw new RecoverableMessageHandlingException('Temporary failure');
```

### 保存和重试失败的消息 {#messenger-failure-transport}

配置 `failure_transport` 以保存失败的消息：

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        failure_transport: failed

        transports:
            failed: 'doctrine://default?queue_name=failed'
```

管理失败的消息：

```terminal
# 查看失败传输中的所有消息
$ php bin/console messenger:failed:show

# 重试特定消息
$ php bin/console messenger:failed:retry 20 30 --force

# 删除消息而不重试
$ php bin/console messenger:failed:remove 20

# 删除所有失败的消息
$ php bin/console messenger:failed:remove --all
```

### 编写幂等处理器 {#messenger-handler-idempotency}

消息可以被投递多次。尽可能设计幂等的处理器（多次运行产生相同结果）。

对于非幂等操作（如支付处理），在消息中包含稳定的幂等键：

```php
final class ProcessPayment
{
    public function __construct(
        public readonly int $orderId,
        public readonly string $idempotencyKey,
    ) {
    }
}
```

---

## 传输配置 {#messenger-transports-config}

### AMQP 传输

```terminal
$ composer require symfony/amqp-messenger
```

```env
MESSENGER_TRANSPORT_DSN=amqp://guest:guest@localhost:5672/%2f/messages
```

### Doctrine 传输

```terminal
$ composer require symfony/doctrine-messenger
```

```env
MESSENGER_TRANSPORT_DSN=doctrine://default
```

### Redis 传输

```terminal
$ composer require symfony/redis-messenger
```

```env
MESSENGER_TRANSPORT_DSN=redis://localhost:6379/messages
```

### 内存传输（用于测试）

```yaml
# config/packages/messenger.yaml
when@test:
    framework:
        messenger:
            transports:
                async_priority_normal: 'in-memory://'
```

在测试中验证消息已调度：

```php
/** @var InMemoryTransport $transport */
$transport = $this->getContainer()->get('messenger.transport.async_priority_normal');
$this->assertCount(1, $transport->getSent());
```

### Amazon SQS 传输

```terminal
$ composer require symfony/amazon-sqs-messenger
```

```env
MESSENGER_TRANSPORT_DSN=https://sqs.eu-west-3.amazonaws.com/123456789012/messages?access_key=KEY&secret_key=SECRET
```

---

## 运行命令和外部进程

调度 `RunCommandMessage` 来触发任何 Symfony 控制台命令：

```php
use Symfony\Component\Console\Messenger\RunCommandMessage;

$this->bus->dispatch(new RunCommandMessage('app:my-cache:clean-up --dir=var/temp'));
$this->bus->dispatch(new RunCommandMessage('cache:clear'));
```

调度 `RunProcessMessage` 来运行外部进程：

```php
use Symfony\Component\Process\Messenger\RunProcessMessage;

$this->bus->dispatch(new RunProcessMessage(['rm', '-rf', 'var/log/temp/*'], cwd: '/my/custom/working-dir'));
```

---

## 从处理器获取结果 {#messenger-getting-handler-results}

```php
use Symfony\Component\Messenger\Stamp\HandledStamp;

$envelope = $messageBus->dispatch(new SomeMessage());
$handledStamp = $envelope->last(HandledStamp::class);
$handledStamp->getResult();
```

使用 `HandleTrait` 获取同步处理器的结果：

```php
class ListItems
{
    use HandleTrait;

    public function __construct(MessageBusInterface $queryBus)
    {
        $this->messageBus = $queryBus;
    }

    public function __invoke(): void
    {
        $result = $this->handle(new ListItemsQuery(/* ... */));
    }
}
```

---

## 扩展 Messenger

### 信封和印章 {#envelopes-stamps}

使用"印章"为消息配置额外信息：

```php
use Symfony\Component\Messenger\Stamp\DelayStamp;

$bus->dispatch(new SmsNotification('...'), [
    new DelayStamp(5000), // 等待 5 秒后处理
]);
```

### 中间件 {#messenger_middleware}

默认中间件包括：

1. `add_bus_name_stamp_middleware` - 添加标记记录消息调度到哪个总线
2. `dispatch_after_current_bus` - 事务性消息
3. `failed_message_processing_middleware` - 处理通过失败传输重试的消息
4. `send_message` - 如果配置了路由则发送消息到传输
5. `handle_message` - 调用消息处理器

自定义中间件：

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        buses:
            messenger.bus.default:
                middleware:
                    - 'App\Middleware\MyMiddleware'
                    - 'App\Middleware\AnotherMiddleware'
```

### Doctrine 中间件

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        buses:
            command_bus:
                middleware:
                    - doctrine_ping_connection
                    - doctrine_close_connection
                    - doctrine_transaction
```

### Messenger 事件

Messenger 还调度几个事件：

- `SendMessageToTransportsEvent`
- `WorkerMessageFailedEvent`
- `WorkerMessageHandledEvent`
- `WorkerMessageReceivedEvent`
- `WorkerMessageRetriedEvent`
- `WorkerStartedEvent`
- `WorkerStoppedEvent`

---

## 多总线、命令和事件总线 {#messenger-multiple-buses}

Messenger 默认提供单个消息总线，但你可以配置多个，创建命令、查询或事件总线：

```yaml
framework:
    messenger:
        default_bus: command.bus
        buses:
            command.bus:
                middleware:
                    - validation
                    - doctrine_transaction
            query.bus:
                middleware:
                    - validation
            event.bus:
                default_middleware:
                    enabled: true
                    allow_no_handlers: false
                middleware:
                    - validation
```

这将创建三个新服务，可通过类型提示自动装配：

- `command.bus`: `MessageBusInterface`（默认总线）
- `query.bus`: `MessageBusInterface $queryBus`
- `event.bus`: `MessageBusInterface $eventBus`

### 限制处理器到特定总线

```yaml
# config/services.yaml
services:
    App\MessageHandler\SomeCommandHandler:
        tags: [{ name: messenger.message_handler, bus: command.bus }]
```

### 调试总线

```terminal
$ php bin/console debug:messenger
```

---

## 重新调度消息

```php
use Symfony\Component\Messenger\Message\RedispatchMessage;

$this->bus->dispatch(new RedispatchMessage($message));
```
