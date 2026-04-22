# 可追踪事件调度器

`Symfony\Component\EventDispatcher\Debug\TraceableEventDispatcher` 是一个事件调度器，它包装任何其他事件调度器，然后可用于确定调度器调用了哪些事件监听器。将要包装的事件调度器和 `Symfony\Component\Stopwatch\Stopwatch` 的实例传递给其构造函数：

```php
use Symfony\Component\EventDispatcher\Debug\TraceableEventDispatcher;
use Symfony\Component\Stopwatch\Stopwatch;

// 要调试的事件调度器
$dispatcher = ...;

$traceableEventDispatcher = new TraceableEventDispatcher(
    $dispatcher,
    new Stopwatch()
);
```

现在，`TraceableEventDispatcher` 可以像任何其他事件调度器一样用于注册事件监听器和调度事件：

```php
// ...

// 注册事件监听器
$eventListener = ...;
$priority = ...;
$traceableEventDispatcher->addListener(
    'event.the_name',
    $eventListener,
    $priority
);

// 调度事件
$event = ...;
$traceableEventDispatcher->dispatch($event, 'event.the_name');
```

在您的应用处理完毕后，您可以使用 `Symfony\Component\EventDispatcher\Debug\TraceableEventDispatcher::getCalledListeners` 方法检索应用中已调用的事件监听器数组。类似地，`Symfony\Component\EventDispatcher\Debug\TraceableEventDispatcher::getNotCalledListeners` 方法返回尚未调用的事件监听器数组：

```php
// ...

$calledListeners = $traceableEventDispatcher->getCalledListeners();
$notCalledListeners = $traceableEventDispatcher->getNotCalledListeners();
```
