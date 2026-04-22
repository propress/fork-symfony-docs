# Semaphore 组件

Semaphore 组件管理[信号量][semaphores]，这是一种提供对共享资源的独占访问的机制。

## 安装

```bash
$ composer require symfony/semaphore
```

如果您在 Symfony 应用之外使用此组件，则必须在代码中引入 Composer 生成的 `vendor/autoload.php` 文件来启用类自动加载机制。更多信息请阅读[此文章](./using_components.md)。

## 用法

在计算机科学中，信号量是一个变量或抽象数据类型，用于控制并发系统（例如多任务操作系统）中多个进程对公共资源的访问。与锁的主要区别在于，信号量允许多个进程访问资源，而锁只允许一个进程。

使用 `Symfony\Component\Semaphore\SemaphoreFactory` 类创建信号量，而该类又需要另一个类来管理存储：

```php
use Symfony\Component\Semaphore\SemaphoreFactory;
use Symfony\Component\Semaphore\Store\RedisStore;

$redis = new Redis();
$redis->connect('172.17.0.2');

$store = new RedisStore($redis);
$factory = new SemaphoreFactory($store);
```

通过调用 `Symfony\Component\Semaphore\SemaphoreFactory::createSemaphore` 方法创建信号量。它的第一个参数是表示锁定资源的任意字符串。它的第二个参数是允许的最大进程数。然后，调用 `Symfony\Component\Semaphore\SemaphoreInterface::acquire` 方法将尝试获取信号量：

```php
// ...
$semaphore = $factory->createSemaphore('pdf-invoice-generation', 2);

if ($semaphore->acquire()) {
    // 资源 "pdf-invoice-generation" 已锁定。
    // 在这里您可以安全地计算和生成发票。

    $semaphore->release();
}
```

如果无法获取信号量，该方法将返回 `false`。即使已经获取了信号量，也可以安全地重复调用 `acquire()` 方法。

> [!NOTE]
> 与其他实现不同，Semaphore 组件即使为同一资源创建信号量实例，也会区分它们。如果信号量必须由多个服务使用，它们应该共享 `SemaphoreFactory::createSemaphore` 方法返回的相同 `Semaphore` 实例。

> [!TIP]
> 如果您不显式释放信号量，它将在实例销毁时自动释放。在某些情况下，跨多个请求锁定资源可能很有用。要禁用自动释放行为，请将 `createSemaphore()` 方法的第五个参数设置为 `false`。

[semaphores]: https://en.wikipedia.org/wiki/Semaphore_(programming)
