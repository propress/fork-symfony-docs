# 不可变事件调度器

`Symfony\Component\EventDispatcher\ImmutableEventDispatcher` 是一个锁定或冻结的事件调度器。该调度器无法注册新的监听器或订阅者。

`ImmutableEventDispatcher` 接受另一个包含所有监听器和订阅者的事件调度器。不可变调度器只是此原始调度器的代理。

要使用它，首先创建一个普通的 `EventDispatcher` 调度器并注册一些监听器或订阅者：

```php
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Contracts\EventDispatcher\Event;

$dispatcher = new EventDispatcher();
$dispatcher->addListener('foo.action', function (Event $event): void {
    // ...
});

// ...
```

现在，将其注入到 `ImmutableEventDispatcher` 中：

```php
use Symfony\Component\EventDispatcher\ImmutableEventDispatcher;
// ...

$immutableDispatcher = new ImmutableEventDispatcher($dispatcher);
```

您需要在项目中使用这个新的调度器。

如果您尝试执行修改调度器的方法之一（例如 `addListener()`），则会抛出 `BadMethodCallException`。
