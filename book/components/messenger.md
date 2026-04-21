# Messenger 组件

Messenger 组件帮助应用程序向其他应用程序或通过消息队列发送和接收消息。

该组件深受 Matthias Noback 的[关于命令总线的博客文章]系列和 [SimpleBus 项目]的启发。

> **参见：**
> 本文介绍了如何在任意 PHP 应用程序中将 Messenger 功能作为独立组件使用。阅读 [/messenger](messenger.md) 文章以了解如何在 Symfony 应用程序中使用它。

## 安装

```terminal
$ composer require symfony/messenger
```

## 概念

**发送者（Sender）**：
负责序列化消息并将其发送到某处。这个"某处"可以是消息代理或第三方 API 等。

**接收者（Receiver）**：
负责检索、反序列化并将消息转发给处理器。这可以是消息队列拉取器或 API 端点等。

**处理器（Handler）**：
负责使用适用于消息的业务逻辑来处理消息。处理器由 `HandleMessageMiddleware` 中间件调用。

**中间件（Middleware）**：
中间件可以在消息通过总线分发时访问消息及其包装器（信封）。字面意思是*"中间的软件"*，这些不涉及应用程序的核心关注点（业务逻辑）。相反，它们是适用于整个应用程序并影响整个消息总线的横切关注点。例如：日志记录、验证消息、开始事务等。它们还负责调用链中的下一个中间件，这意味着它们可以通过添加印记或甚至替换信封来调整信封，以及中断中间件链。中间件在消息最初分发时和从传输接收消息时都会被调用。

**信封（Envelope）**：
Messenger 特有的概念，通过将消息包装在信封中来提供消息总线内部的完全灵活性，允许通过*信封印记*在内部添加有用的信息。

**信封印记（Envelope Stamps）**：
你需要附加到消息上的信息片段：用于传输的序列化器上下文、标识已接收消息的标记，或你的中间件或传输层可能使用的任何元数据。

## 总线

总线用于分发消息。总线的行为由其有序的中间件栈决定。该组件附带了一组可以使用的中间件。

在与 Symfony 的 FrameworkBundle 一起使用消息总线时，以下中间件会为你配置：

1. `Symfony\Component\Messenger\Middleware\SendMessageMiddleware`（启用异步处理，如果你提供了日志记录器则记录消息处理）
2. `Symfony\Component\Messenger\Middleware\HandleMessageMiddleware`（调用注册的处理器）

示例：

```php
use App\Message\MyMessage;
use App\MessageHandler\MyMessageHandler;
use Symfony\Component\Messenger\Handler\HandlersLocator;
use Symfony\Component\Messenger\MessageBus;
use Symfony\Component\Messenger\Middleware\HandleMessageMiddleware;

$handler = new MyMessageHandler();

$bus = new MessageBus([
    new HandleMessageMiddleware(new HandlersLocator([
        MyMessage::class => [$handler],
    ])),
]);

$bus->dispatch(new MyMessage(/* ... */));
```

> **注意：**
> 每个中间件都需要实现 `Symfony\Component\Messenger\Middleware\MiddlewareInterface`。

## 处理器

分发到总线后，消息将由"消息处理器"处理。消息处理器是一个 PHP 可调用对象（即函数或类的实例），它将为你的消息执行所需的处理：

```php
namespace App\MessageHandler;

use App\Message\MyMessage;

class MyMessageHandler
{
    public function __invoke(MyMessage $message): void
    {
        // Message processing...
    }
}
```

## 向消息添加元数据（信封）

如果你需要向消息添加元数据或某些配置，请用 `Symfony\Component\Messenger\Envelope` 类包装它并添加印记。例如，要设置消息通过传输层时使用的序列化分组，请使用 `SerializerStamp` 印记：

```php
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Stamp\SerializerStamp;

$bus->dispatch(
    new Envelope($message)->with(new SerializerStamp([
        // groups are applied to the whole message, so make sure
        // to define the group for every embedded object
        'groups' => ['my_serialization_groups'],
    ]))
);
```

以下是 Symfony Messenger 附带的一些重要信封印记：

* `Symfony\Component\Messenger\Stamp\DelayStamp`，用于延迟处理异步消息。
* `Symfony\Component\Messenger\Stamp\DispatchAfterCurrentBusStamp`，使消息在当前总线执行完毕后才被处理。详见事务性消息相关文档。
* `Symfony\Component\Messenger\Stamp\HandledStamp`，标记消息已被特定处理器处理的印记。允许访问处理器返回值和处理器名称。
* `Symfony\Component\Messenger\Stamp\ReceivedStamp`，标记消息已从传输接收的内部印记。
* `Symfony\Component\Messenger\Stamp\SentStamp`，标记消息已被特定发送者发送的印记。允许访问发送者的完全限定类名以及来自 `Symfony\Component\Messenger\Transport\Sender\SendersLocator` 的别名（如果可用）。
* `Symfony\Component\Messenger\Stamp\SerializerStamp`，用于配置传输使用的序列化分组。
* `Symfony\Component\Messenger\Stamp\ValidationStamp`，用于配置启用验证中间件时使用的验证分组。
* `Symfony\Component\Messenger\Stamp\ErrorDetailsStamp`，当消息因处理器中的异常而失败时的内部印记。
* `Symfony\Component\Scheduler\Messenger\ScheduledStamp`，标记消息由调度器生成的印记。这有助于将其与"手动"创建的消息区分开来。你可以在 [调度器文档](scheduler.md) 中了解更多信息。

> **注意：**
> `Symfony\Component\Messenger\Stamp\ErrorDetailsStamp` 印记包含一个 `Symfony\Component\ErrorHandler\Exception\FlattenException`，它是导致消息失败的异常的表示。你可以使用 `Symfony\Component\Messenger\Stamp\ErrorDetailsStamp::getFlattenException` 方法获取此异常。由于 `Symfony\Component\Messenger\Transport\Serialization\Normalizer\FlattenExceptionNormalizer` 的规范化处理，该异常有助于 Messenger 上下文中的错误报告。

不同于在中间件中直接处理消息，你接收到的是信封。因此，你可以检查信封内容及其印记，或添加任何印记：

```php
use App\Message\Stamp\AnotherStamp;
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Middleware\MiddlewareInterface;
use Symfony\Component\Messenger\Middleware\StackInterface;
use Symfony\Component\Messenger\Stamp\ReceivedStamp;

class MyOwnMiddleware implements MiddlewareInterface
{
    public function handle(Envelope $envelope, StackInterface $stack): Envelope
    {
        if (null !== $envelope->last(ReceivedStamp::class)) {
            // Message just has been received...

            // You could for example add another stamp.
            $envelope = $envelope->with(new AnotherStamp(/* ... */));
        } else {
            // Message was just originally dispatched
        }

        return $stack->next()->handle($envelope, $stack);
    }
}
```

如果消息刚刚被接收（即至少有一个 `ReceivedStamp` 印记），上述示例将把消息转发到下一个中间件，并附加一个额外的印记。你可以通过实现 `Symfony\Component\Messenger\Stamp\StampInterface` 来创建自己的印记。

如果你想检查信封上的所有印记，请使用 `$envelope->all()` 方法，该方法返回按类型（完全限定类名）分组的所有印记。或者，你可以通过使用完全限定类名作为该方法的第一个参数来遍历特定类型的所有印记（例如 `$envelope->all(ReceivedStamp::class)`）。

> **注意：**
> 如果通过使用 `Symfony\Component\Messenger\Transport\Serialization\Serializer` 基础序列化器的传输，任何印记都必须可以使用 Symfony Serializer 组件序列化。

## 传输

为了发送和接收消息，你需要配置传输。传输负责与你的消息代理或第三方通信。

### 自定义发送者

假设你已经有一个通过消息总线并由处理器处理的 `ImportantAction` 消息。现在你还想将此消息作为电子邮件发送（使用 [Mime 组件](mime.md) 和 [Mailer 组件](mailer.md)）。

使用 `Symfony\Component\Messenger\Transport\Sender\SenderInterface`，你可以创建自己的消息发送者：

```php
namespace App\MessageSender;

use App\Message\ImportantAction;
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Transport\Sender\SenderInterface;
use Symfony\Component\Mime\Email;

class ImportantActionToEmailSender implements SenderInterface
{
    public function __construct(
        private MailerInterface $mailer,
        private string $toEmail,
    ) {
    }

    public function send(Envelope $envelope): Envelope
    {
        $message = $envelope->getMessage();

        if (!$message instanceof ImportantAction) {
            throw new \InvalidArgumentException(sprintf('This transport only supports "%s" messages.', ImportantAction::class));
        }

        $this->mailer->send(
            new Email()
                ->to($this->toEmail)
                ->subject('Important action made')
                ->html('<h1>Important action</h1><p>Made by '.$message->getUsername().'</p>')
        );

        return $envelope;
    }
}
```

### 自定义接收者

接收者负责从源获取消息并将其分发给应用程序。

假设你已经在应用程序中使用 `NewOrder` 消息处理了一些"订单"。现在你想与第三方或遗留应用程序集成，但无法使用 API，需要使用包含新订单的共享 CSV 文件。

你将读取此 CSV 文件并分发 `NewOrder` 消息。你只需编写自己的 CSV 接收者：

```php
namespace App\MessageReceiver;

use App\Message\NewOrder;
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Exception\MessageDecodingFailedException;
use Symfony\Component\Messenger\Transport\Receiver\ReceiverInterface;
use Symfony\Component\Serializer\SerializerInterface;

class NewOrdersFromCsvFileReceiver implements ReceiverInterface
{
    private $connection;

    public function __construct(
        private SerializerInterface $serializer,
        private string $filePath,
    ) {
        // Available connection bundled with the Messenger component
        // can be found in "Symfony\Component\Messenger\Bridge\*\Transport\Connection".
        $this->connection = /* create your connection */;
    }

    public function get(): iterable
    {
        // Receive the envelope according to your transport ($yourEnvelope here),
        // in most cases, using a connection is the easiest solution.
        $yourEnvelope = $this->connection->get();
        if (null === $yourEnvelope) {
            return [];
        }

        try {
            $envelope = $this->serializer->decode([
                'body' => $yourEnvelope['body'],
                'headers' => $yourEnvelope['headers'],
            ]);
        } catch (MessageDecodingFailedException $exception) {
            $this->connection->reject($yourEnvelope['id']);
            throw $exception;
        }

        return [$envelope->with(new CustomStamp($yourEnvelope['id']))];
    }

    public function ack(Envelope $envelope): void
    {
        // Add information about the handled message
    }

    public function reject(Envelope $envelope): void
    {
        // In the case of a custom connection
        $id = /* get the message id thanks to information or stamps present in the envelope */;

        $this->connection->reject($id);
    }
}
```

### 同一总线上的接收者和发送者

为了允许在同一总线上发送和接收消息并防止无限循环，消息总线将向消息信封添加 `Symfony\Component\Messenger\Stamp\ReceivedStamp` 印记，`Symfony\Component\Messenger\Middleware\SendMessageMiddleware` 中间件将知道不应再次将这些消息路由到传输。

[关于命令总线的博客文章]: https://matthiasnoback.nl/tags/command-bus/
[SimpleBus 项目]: https://docs.simplebus.io/en/latest/
