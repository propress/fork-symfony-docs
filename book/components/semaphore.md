# Semaphore 组件

Semaphore 组件管理[信号量][semaphores]——一种为共享资源提供独占访问的机制。

## 安装

```terminal
$ composer require symfony/semaphore
```

## 使用

在计算机科学中，信号量是一种变量或抽象数据类型，用于在多任务操作系统等并发系统中控制多个进程对公共资源的访问。与[锁](lock.md)的主要区别在于，信号量允许多个进程访问资源，而锁只允许一个进程访问。

使用 `SemaphoreFactory` 类创建信号量，该类需要另一个类来管理存储：

```php
use Symfony\Component\Semaphore\SemaphoreFactory;
use Symfony\Component\Semaphore\Store\RedisStore;

$redis = new Redis();
$redis->connect('172.17.0.2');

$store = new RedisStore($redis);
$factory = new SemaphoreFactory($store);
```

通过调用 `SemaphoreFactory::createSemaphore` 方法创建信号量。其第一个参数是表示被锁定资源的任意字符串，第二个参数是允许的最大进程数。然后，调用 `SemaphoreInterface::acquire` 方法将尝试获取信号量：

```php
// ...
$semaphore = $factory->createSemaphore('pdf-invoice-generation', 2);

if ($semaphore->acquire()) {
    // 资源 "pdf-invoice-generation" 已被锁定。
    // 在这里你可以安全地计算并生成发票。

    $semaphore->release();
}
```

如果无法获取信号量，该方法返回 `false`。即使信号量已被获取，也可以安全地多次调用 `acquire()` 方法。

> **注意：** 与其他实现不同，Semaphore 组件即使对同一资源创建的信号量实例也加以区分。如果一个信号量需要被多个服务使用，它们应该共享由 `SemaphoreFactory::createSemaphore` 方法返回的同一个 `Semaphore` 实例。

> **提示：** 如果你没有显式释放信号量，它将在实例销毁时自动释放。在某些情况下，跨多个请求锁定资源可能很有用。要禁用自动释放行为，请将 `createSemaphore()` 方法的第五个参数设置为 `false`。

[semaphores]: https://en.wikipedia.org/wiki/Semaphore_(programming)
