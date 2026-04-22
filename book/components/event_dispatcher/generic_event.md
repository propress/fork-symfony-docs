# 通用事件对象

EventDispatcher 组件提供的基础 `Symfony\Contracts\EventDispatcher\Event` 类故意简洁，以允许通过使用 OOP 继承创建特定于 API 的事件对象。这允许在复杂应用中使用优雅且可读的代码。

`Symfony\Component\EventDispatcher\GenericEvent` 可供那些希望在整个应用中仅使用一个事件对象的人使用。它开箱即用，适合大多数用途，因为它遵循标准观察者模式，其中事件对象封装事件"主题"，但增加了可选的额外参数。

`Symfony\Component\EventDispatcher\GenericEvent` 除了基类 `Symfony\Contracts\EventDispatcher\Event` 之外，还添加了一些更多的方法

* `Symfony\Component\EventDispatcher\GenericEvent::__construct`：构造函数接受事件主题和任何参数；

* `Symfony\Component\EventDispatcher\GenericEvent::getSubject`：获取主题；

* `Symfony\Component\EventDispatcher\GenericEvent::setArgument`：按键设置参数；

* `Symfony\Component\EventDispatcher\GenericEvent::setArguments`：设置参数数组；

* `Symfony\Component\EventDispatcher\GenericEvent::getArgument`：按键获取参数；

* `Symfony\Component\EventDispatcher\GenericEvent::getArguments`：获取所有参数的 getter；

* `Symfony\Component\EventDispatcher\GenericEvent::hasArgument`：如果参数键存在，则返回 true；

`GenericEvent` 还在事件参数上实现了 `ArrayAccess`，这使得传递有关事件主题的额外参数非常方便。

以下示例显示了用例以提供灵活性的总体思路。这些示例假定事件监听器已添加到调度器。

传递主题：

```php
use Symfony\Component\EventDispatcher\GenericEvent;

$event = new GenericEvent($subject);
$dispatcher->dispatch($event, 'foo');

class FooListener
{
    public function handler(GenericEvent $event): void
    {
        if ($event->getSubject() instanceof Foo) {
            // ...
        }
    }
}
```

使用 `ArrayAccess` API 传递和处理参数以访问事件参数：

```php
use Symfony\Component\EventDispatcher\GenericEvent;

$event = new GenericEvent(
    $subject,
    ['type' => 'foo', 'counter' => 0]
);
$dispatcher->dispatch($event, 'foo');

class FooListener
{
    public function handler(GenericEvent $event): void
    {
        if (isset($event['type']) && 'foo' === $event['type']) {
            // ... 做某事
        }

        $event['counter']++;
    }
}
```

过滤数据：

```php
use Symfony\Component\EventDispatcher\GenericEvent;

$event = new GenericEvent($subject, ['data' => 'Foo']);
$dispatcher->dispatch($event, 'foo');

class FooListener
{
    public function filter(GenericEvent $event): void
    {
        $event['data'] = strtolower($event['data']);
    }
}
```
