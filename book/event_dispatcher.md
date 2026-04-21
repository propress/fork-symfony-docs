# 事件与事件监听器

在 Symfony 应用执行期间，会触发大量事件通知。你的应用可以监听这些通知，并通过执行任意代码来响应它们。

Symfony 在处理 HTTP 请求时触发多个[与内核相关的事件](reference/events.md)。第三方 Bundle 也可以分发事件，你甚至可以从自己的代码中分发[自定义事件](components/event_dispatcher.md)。

本文中所有示例都使用相同的 `KernelEvents::EXCEPTION` 事件（为了保持一致性）。在你自己的应用中，你可以使用任何事件，甚至在同一个订阅者中混合使用多个事件。

---

## 创建事件监听器

监听事件最常见的方式是注册一个**事件监听器**：

```php
// src/EventListener/ExceptionListener.php
namespace App\EventListener;

use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;
use Symfony\Component\HttpKernel\Exception\HttpExceptionInterface;

class ExceptionListener
{
    public function __invoke(ExceptionEvent $event): void
    {
        // 从接收到的事件中获取异常对象
        $exception = $event->getThrowable();
        $message = sprintf(
            'My Error says: %s with code: %s',
            $exception->getMessage(),
            $exception->getCode()
        );

        // 自定义响应对象以显示异常详情
        $response = new Response();
        $response->setContent($message);
        // 异常消息可能包含未过滤的用户输入；
        // 将 content-type 设置为 text 以避免 XSS 问题
        $response->headers->set('Content-Type', 'text/plain; charset=utf-8');

        // HttpExceptionInterface 是一种特殊类型的异常，
        // 保存状态码和头信息详情
        if ($exception instanceof HttpExceptionInterface) {
            $response->setStatusCode($exception->getStatusCode());
            $response->headers->replace($exception->getHeaders());
        } else {
            $response->setStatusCode(Response::HTTP_INTERNAL_SERVER_ERROR);
        }

        // 向事件发送修改后的响应对象
        $event->setResponse($response);
    }
}
```

现在类已创建，你需要将其注册为服务，并通过使用特殊"标签"通知 Symfony 它是事件监听器：

```yaml
# config/services.yaml
services:
    App\EventListener\ExceptionListener:
        tags: [kernel.event_listener]
```

Symfony 遵循以下逻辑决定在事件监听器类中调用哪个方法：

1. 如果 `kernel.event_listener` 标签定义了 `method` 属性，则这就是要调用的方法名；
2. 如果未定义 `method` 属性，则尝试调用 `__invoke()` 魔术方法（使事件监听器可调用）；
3. 如果也未定义 `__invoke()` 方法，则抛出异常。

> **注意**
>
> `kernel.event_listener` 标签有一个名为 `priority` 的可选属性，它是一个正整数或负整数，默认为 `0`，它控制监听器的执行顺序（数字越大，监听器越早执行）。当你需要保证一个监听器在另一个之前执行时，这很有用。Symfony 内部监听器的优先级通常在 `-256` 到 `256` 之间，但你自己的监听器可以使用任何正整数或负整数。

> **注意**
>
> `kernel.event_listener` 标签有一个名为 `event` 的可选属性，当监听器 `$event` 参数没有类型提示时很有用。如果你配置了它，它将更改 `$event` 对象的类型。对于 `kernel.exception` 事件，它是 `Symfony\Component\HttpKernel\Event\ExceptionEvent`。查看 [Symfony 事件参考](reference/events.md)以查看每个事件提供的对象类型。
>
> 使用此属性，Symfony 遵循以下逻辑决定在事件监听器类中调用哪个方法：
>
> 1. 如果 `kernel.event_listener` 标签定义了 `method` 属性，则这就是要调用的方法名；
> 2. 如果未定义 `method` 属性，则尝试调用名为 `on` + "帕斯卡命名事件名"的方法（例如 `kernel.exception` 事件的 `onKernelException()` 方法）；
> 3. 如果该方法也未定义，则尝试调用 `__invoke()` 魔术方法；
> 4. 如果 `__invoke()` 方法也未定义，则抛出异常。

### 使用 PHP 属性定义事件监听器 {#event-dispatcher_event-listener-attributes}

定义事件监听器的另一种方式是使用 `#[AsEventListener]` PHP 属性。这允许你在类内部配置监听器，而无需在外部文件中添加任何配置：

```php
namespace App\EventListener;

use App\Event\CustomEvent;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener]
final class MyListener
{
    public function __invoke(CustomEvent $event): void
    {
        // ...
    }
}
```

你可以添加多个 `#[AsEventListener]` 属性来配置不同的方法。`method` 属性是可选的，未定义时默认为 `on` + 事件名首字母大写。在下面的示例中，`'foo'` 事件监听器没有明确定义其方法，因此将调用 `onFoo()` 方法：

```php
namespace App\EventListener;

use App\Event\CustomEvent;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener(event: CustomEvent::class, method: 'onCustomEvent')]
#[AsEventListener(event: 'foo', priority: 42)]
#[AsEventListener(event: 'bar', method: 'onBarEvent')]
final class MyMultiListener
{
    public function onCustomEvent(CustomEvent $event): void
    {
        // ...
    }

    public function onFoo(): void
    {
        // ...
    }

    public function onBarEvent(): void
    {
        // ...
    }
}
```

`#[AsEventListener]` 也可以直接应用于方法：

```php
namespace App\EventListener;

use App\Event\CustomEvent;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

final class MyMultiListener
{
    #[AsEventListener]
    public function onCustomEvent(CustomEvent $event): void
    {
        // ...
    }

    #[AsEventListener]
    public function onMultipleCustomEvent(CustomEvent|AnotherCustomEvent $event): void
    {
        // ...
    }

    #[AsEventListener(event: 'foo', priority: 42)]
    public function onFoo(): void
    {
        // ...
    }

    #[AsEventListener(event: 'bar')]
    public function onBarEvent(): void
    {
        // ...
    }
}
```

> **注意**
>
> 如果方法已经对预期事件进行了类型提示，则该属性不要求设置其 `event` 参数。

---

## 创建事件订阅者 {#events-subscriber}

监听事件的另一种方式是通过**事件订阅者**——这是一个定义一个或多个监听一个或多个事件的方法的类。与事件监听器的主要区别在于，订阅者始终知道它们正在监听哪些事件。

如果不同的事件订阅者方法监听同一个事件，它们的顺序由 `priority` 参数定义。此值是一个正整数或负整数，默认为 `0`。数字越大，方法越早被调用。**优先级对所有监听器和订阅者是聚合的**，因此你的方法可能在其他监听器和订阅者中定义的方法之前或之后被调用。

以下示例显示了一个事件订阅者，它定义了多个方法，通过其 `ExceptionEvent` 类监听同一个 `kernel.exception` 事件：

```php
// src/EventSubscriber/ExceptionSubscriber.php
namespace App\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;

class ExceptionSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        // 返回订阅的事件、它们的方法和优先级
        return [
            ExceptionEvent::class => [
                ['processException', 10],
                ['logException', 0],
                ['notifyException', -10],
            ],
        ];
    }

    public function processException(ExceptionEvent $event): void
    {
        // ...
    }

    public function logException(ExceptionEvent $event): void
    {
        // ...
    }

    public function notifyException(ExceptionEvent $event): void
    {
        // ...
    }
}
```

你的 `services.yaml` 文件应该已经设置为从 `EventSubscriber` 目录加载服务。Symfony 会处理其余的事情。

> **提示**
>
> 如果抛出异常时没有调用你的方法，请仔细检查你是否从 `EventSubscriber` 目录[加载服务](service_container.md#服务容器服务加载示例)并启用了[自动配置](service_container.md#服务自动配置)。你也可以手动添加 `kernel.event_subscriber` 标签。

---

## 请求事件与检查类型

单个页面可以发出多个请求（一个主请求，然后是多个子请求——通常是在[模板中嵌入控制器](templates.md#在模板中嵌入控制器)时）。对于核心 Symfony 事件，你可能需要检查事件是针对"主"请求还是"子请求"：

```php
// src/EventListener/RequestListener.php
namespace App\EventListener;

use Symfony\Component\HttpKernel\Event\RequestEvent;

class RequestListener
{
    public function onKernelRequest(RequestEvent $event): void
    {
        if (!$event->isMainRequest()) {
            // 如果不是主请求则不做任何事情
            return;
        }

        // ...
    }
}
```

某些事情（如检查*真实*请求的信息）可能不需要在子请求监听器上完成。

---

## 监听器还是订阅者 {#events-or-subscribers}

监听器和订阅者可以在同一应用中无差别地使用。决定使用哪个通常是个人偏好问题。但是，每个都有一些小的优势：

- **订阅者更容易复用**，因为事件的知识保存在类中，而不是服务定义中。这就是 Symfony 在内部使用订阅者的原因；
- **监听器更灵活**，因为 Bundle 可以根据某些配置值有条件地启用或禁用每个监听器。

---

## 事件别名

通过依赖注入配置事件监听器和订阅者时，Symfony 的核心事件也可以通过相应事件类的完全限定类名（FQCN）来引用：

```php
// src/EventSubscriber/RequestSubscriber.php
namespace App\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;

class RequestSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            RequestEvent::class => 'onKernelRequest',
        ];
    }

    public function onKernelRequest(RequestEvent $event): void
    {
        // ...
    }
}
```

在内部，事件 FQCN 被视为原始事件名的别名。由于映射在编译服务容器时已经发生，使用 FQCN 代替事件名的事件监听器和订阅者在检查事件分发器时将显示在原始事件名下。

可以通过注册编译器通道 `AddEventAliasesPass` 为自定义事件扩展此别名映射：

```php
// src/Kernel.php
namespace App;

use App\Event\MyCustomEvent;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\EventDispatcher\DependencyInjection\AddEventAliasesPass;
use Symfony\Component\HttpKernel\Kernel as BaseKernel;

class Kernel extends BaseKernel
{
    protected function build(ContainerBuilder $container): void
    {
        $container->addCompilerPass(new AddEventAliasesPass([
            MyCustomEvent::class => 'my_custom_event',
        ]));
    }
}
```

---

## 调试事件监听器

你可以使用控制台查看事件分发器中注册了哪些监听器。要显示所有事件及其监听器，请运行：

```terminal
$ php bin/console debug:event-dispatcher
```

你可以通过指定名称来获取特定事件的已注册监听器：

```terminal
$ php bin/console debug:event-dispatcher kernel.exception
```

或者获取所有与事件名部分匹配的内容：

```terminal
$ php bin/console debug:event-dispatcher kernel // 匹配 "kernel.exception"、"kernel.response" 等
$ php bin/console debug:event-dispatcher Security // 匹配 "Symfony\Component\Security\Http\Event\CheckPassportEvent"
```

[安全](security.md)系统每个防火墙使用一个事件分发器。使用 `--dispatcher` 选项获取特定事件分发器的已注册监听器：

```terminal
$ php bin/console debug:event-dispatcher --dispatcher=security.event_dispatcher.main
```

---

## 如何设置前置和后置过滤器 {#event-dispatcher-before-after-filters}

在 Web 应用开发中，经常需要在控制器动作之前或之后立即执行一些逻辑，充当过滤器或钩子。

一些 Web 框架定义了 `preExecute()` 和 `postExecute()` 等方法，但在 Symfony 中没有这样的东西。好消息是，使用 [EventDispatcher 组件](components/event_dispatcher.md)有一种更好的方式来干预 Request -> Response 过程。

### Token 验证示例

假设你需要开发一个 API，其中一些控制器是公开的，但另一些只限于一个或多个客户端。对于这些私有功能，你可能会向客户端提供一个 Token 来标识自己。

因此，在执行控制器动作之前，你需要检查该动作是否受限。如果受限，你需要验证提供的 Token。

> **注意**
>
> 请注意，为了简单起见，Token 将在配置中定义，不会使用数据库设置或通过安全组件进行身份验证。

### 使用 `kernel.controller` 事件的前置过滤器

首先，将一些 Token 配置定义为参数：

```yaml
# config/services.yaml
parameters:
    tokens:
        client1: pass1
        client2: pass2
```

#### 标记要检查的控制器

`kernel.controller`（即 `KernelEvents::CONTROLLER`）监听器会在*每次*请求时收到通知，就在控制器执行之前。因此，首先你需要某种方法来识别与请求匹配的控制器是否需要 Token 验证。

一种干净简单的方法是创建一个空接口，让控制器实现它：

```php
namespace App\Controller;

interface TokenAuthenticatedController
{
    // ...
}
```

实现此接口的控制器如下所示：

```php
namespace App\Controller;

use App\Controller\TokenAuthenticatedController;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;

class FooController extends AbstractController implements TokenAuthenticatedController
{
    // 需要身份验证的动作
    public function bar(): Response
    {
        // ...
    }
}
```

#### 创建事件订阅者

接下来，你需要创建一个事件订阅者，它将保存你希望在控制器之前执行的逻辑：

```php
// src/EventSubscriber/TokenSubscriber.php
namespace App\EventSubscriber;

use App\Controller\TokenAuthenticatedController;
use Symfony\Component\DependencyInjection\Attribute\Autowire;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ControllerEvent;
use Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException;
use Symfony\Component\HttpKernel\KernelEvents;

class TokenSubscriber implements EventSubscriberInterface
{
    public function __construct(
        #[Autowire(param: 'tokens')]
        private array $tokens
    ) {
    }

    public function onKernelController(ControllerEvent $event): void
    {
        $controller = $event->getController();

        // 当控制器类定义多个动作方法时，控制器
        // 以 [$controllerInstance, 'methodName'] 的形式返回
        if (is_array($controller)) {
            $controller = $controller[0];
        }

        if ($controller instanceof TokenAuthenticatedController) {
            $token = $event->getRequest()->query->get('token');
            if (!in_array($token, $this->tokens)) {
                throw new AccessDeniedHttpException('This action needs a valid token!');
            }
        }
    }

    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::CONTROLLER => 'onKernelController',
        ];
    }
}
```

你的 `services.yaml` 文件应该已经设置为从 `EventSubscriber` 目录加载服务。Symfony 会处理其余的事情。`TokenSubscriber` 的 `onKernelController()` 方法将在每次请求时执行。如果即将执行的控制器实现了 `TokenAuthenticatedController`，则应用 Token 身份验证。这让你可以在任何你想要的控制器上拥有"前置"过滤器。

### 使用 `kernel.response` 事件的后置过滤器

除了在控制器执行*之前*有一个"钩子"，你还可以添加一个在控制器执行*之后*执行的钩子。例如，假设你想为所有通过 Token 身份验证的响应添加一个 `sha1` 哈希（使用该 Token 作为盐）。

另一个核心 Symfony 事件——称为 `kernel.response`（即 `KernelEvents::RESPONSE`）——在每次请求时通知，但在控制器返回 Response 对象之后。

例如，修改上一个示例中的 `TokenSubscriber`，首先在请求属性中记录身份验证 Token。这将作为此请求通过了 Token 身份验证的基本标志：

```php
public function onKernelController(ControllerEvent $event): void
{
    // ...

    if ($controller instanceof TokenAuthenticatedController) {
        $token = $event->getRequest()->query->get('token');
        if (!in_array($token, $this->tokens)) {
            throw new AccessDeniedHttpException('This action needs a valid token!');
        }

        // 标记请求已通过 Token 身份验证
        $event->getRequest()->attributes->set('auth_token', $token);
    }
}
```

现在，配置订阅者监听另一个事件并添加 `onKernelResponse()`。这将在请求对象上查找 `auth_token` 标志，如果找到则在响应中设置自定义头：

```php
// 在文件顶部添加新的 use 语句
use Symfony\Component\HttpKernel\Event\ResponseEvent;

public function onKernelResponse(ResponseEvent $event): void
{
    // 检查 onKernelController 是否将此标记为 Token 身份验证的请求
    if (!$token = $event->getRequest()->attributes->get('auth_token')) {
        return;
    }

    $response = $event->getResponse();

    // 创建哈希并将其设置为响应头
    $hash = sha1($response->getContent().$token);
    $response->headers->set('X-CONTENT-HASH', $hash);
}

public static function getSubscribedEvents(): array
{
    return [
        KernelEvents::CONTROLLER => 'onKernelController',
        KernelEvents::RESPONSE => 'onKernelResponse',
    ];
}
```

`TokenSubscriber` 现在在每次控制器执行之前（`onKernelController()`）和每次控制器返回响应之后（`onKernelResponse()`）都会收到通知。

---

## 如何不使用继承来自定义方法行为 {#event-dispatcher-method-behavior}

如果你想在方法调用之前或之后立即执行某些操作，可以分别在方法开始或结束时分发事件：

```php
use Symfony\Contracts\EventDispatcher\EventDispatcherInterface;

class CustomMailer
{
    public function __construct(
        private EventDispatcherInterface $dispatcher,
    ) {
    }

    public function send(string $subject, string $message): mixed
    {
        // 在方法之前分发事件
        $event = new BeforeSendMailEvent($subject, $message);
        $this->dispatcher->dispatch($event, 'mailer.pre_send');

        // 从事件获取 $subject 和 $message，它们可能已被修改
        $subject = $event->getSubject();
        $message = $event->getMessage();

        // 这里是真正的方法实现
        $returnValue = ...;

        // 在方法之后执行某些操作
        $event = new AfterSendMailEvent($returnValue);
        $this->dispatcher->dispatch($event, 'mailer.post_send');

        return $event->getReturnValue();
    }
}
```

在此示例中，分发了两个事件：

1. `mailer.pre_send`，在调用方法之前；
2. `mailer.post_send`，在调用方法之后。

> **提示**
>
> 注入事件分发器时，对其接口进行类型提示，而不是具体的 `EventDispatcher` 类。仅需要分发事件时使用 `Symfony\Contracts\EventDispatcher\EventDispatcherInterface`，如果你还需要检查或管理监听器（例如 `addListener()`、`removeListener()`），则使用 `Symfony\Component\EventDispatcher\EventDispatcherInterface`。

每个事件使用自定义事件类向两个事件的监听器传递信息。例如，`BeforeSendMailEvent` 可能如下所示：

```php
// src/Event/BeforeSendMailEvent.php
namespace App\Event;

use Symfony\Contracts\EventDispatcher\Event;

class BeforeSendMailEvent extends Event
{
    public function __construct(
        private string $subject,
        private string $message,
    ) {
    }

    public function getSubject(): string
    {
        return $this->subject;
    }

    public function setSubject(string $subject): void
    {
        $this->subject = $subject;
    }

    public function getMessage(): string
    {
        return $this->message;
    }

    public function setMessage(string $message): void
    {
        $this->message = $message;
    }
}
```

`AfterSendMailEvent` 如下所示：

```php
// src/Event/AfterSendMailEvent.php
namespace App\Event;

use Symfony\Contracts\EventDispatcher\Event;

class AfterSendMailEvent extends Event
{
    public function __construct(
        private mixed $returnValue,
    ) {
    }

    public function getReturnValue(): mixed
    {
        return $this->returnValue;
    }

    public function setReturnValue(mixed $returnValue): void
    {
        $this->returnValue = $returnValue;
    }
}
```

两个事件都允许你获取某些信息（例如 `getMessage()`）甚至更改该信息（例如 `setMessage()`）。

现在，你可以创建一个事件订阅者来挂接此事件。例如，你可以监听 `mailer.post_send` 事件并更改方法的返回值：

```php
// src/EventSubscriber/MailPostSendSubscriber.php
namespace App\EventSubscriber;

use App\Event\AfterSendMailEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class MailPostSendSubscriber implements EventSubscriberInterface
{
    public function onMailerPostSend(AfterSendMailEvent $event): void
    {
        $returnValue = $event->getReturnValue();
        // 修改原始 $returnValue 值

        $event->setReturnValue($returnValue);
    }

    public static function getSubscribedEvents(): array
    {
        return [
            'mailer.post_send' => 'onMailerPostSend',
        ];
    }
}
```

---

## 延伸阅读

- [请求-响应生命周期](controller.md#request-和-response-对象)
- [Symfony 事件参考](reference/events.md)
- [安全相关事件](security.md#安全事件)
- [EventDispatcher 组件](components/event_dispatcher.md)
