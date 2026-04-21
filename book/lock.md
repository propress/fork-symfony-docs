# 使用锁处理并发

当程序并发运行时，修改共享资源的某些代码段不应被多个进程同时访问。Symfony 的 [Lock 组件](components/lock.md)提供了一种锁定机制，以确保在任何时间点只有一个进程运行代码的关键部分，从而防止竞争条件的发生。

以下示例显示了锁的典型用法：

```php
$lock = $lockFactory->createLock('pdf-creation');
if (!$lock->acquire()) {
    return;
}

// 代码的关键部分
$service->method();

$lock->release();
```

---

## 安装

在使用 [Symfony Flex](setup.md#symfony-flex) 的应用中，运行此命令安装 Lock 组件：

```terminal
$ composer require symfony/lock
```

---

## 配置

默认情况下，Symfony 在可用时提供 Semaphore，否则提供 Flock。你可以使用 `lock` 键配置此行为：

```yaml
# config/packages/lock.yaml
framework:
    lock: ~
    lock: 'flock'
    lock: 'flock:///path/to/file'
    lock: 'semaphore'
    lock: 'memcached://m1.docker'
    lock: ['memcached://m1.docker', 'memcached://m2.docker']
    lock: 'redis://r1.docker'
    lock: ['redis://r1.docker', 'redis://r2.docker']
    lock: 'sqlite:///%kernel.project_dir%/var/lock.db'
    lock: 'mysql:host=127.0.0.1;dbname=app'
    lock: 'pgsql:host=127.0.0.1;dbname=app'
    lock: 'mongodb://127.0.0.1/app?collection=lock'
    lock: '%env(LOCK_DSN)%'
    # 使用现有服务
    lock: 'snc_redis.default'

    # 命名锁
    lock:
        invoice: ['semaphore', 'redis://r2.docker']
        report: 'semaphore'
```

---

## 锁定资源

要锁定默认资源，使用 `LockFactory` 自动装配锁工厂：

```php
// src/Controller/PdfController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Lock\LockFactory;

class PdfController extends AbstractController
{
    #[Route('/download/terms-of-use.pdf')]
    public function downloadPdf(LockFactory $factory, MyPdfGeneratorService $pdf): Response
    {
        $lock = $factory->createLock('pdf-creation');
        $lock->acquire(true);

        // 大量计算
        $myPdf = $pdf->getOrCreatePdf();

        $lock->release();

        // ...
    }
}
```

> **警告**
>
> 在同一进程内多次调用 `acquire` 时，`LockInterface` 的同一实例不会阻塞。当多个服务使用同一个锁时，改为注入 `LockFactory` 以为每个服务创建单独的锁实例。

---

## 锁定动态资源

有时应用能够将资源切割成小块，以便只锁定一小部分进程并让其他进程通过。上面的示例展示了如何为每个人锁定 `$pdf->getOrCreatePdf()` 调用，现在让我们看看如何只为请求同一 `$version` 的进程锁定 `$pdf->getOrCreatePdf($version)` 调用：

```php
// src/Controller/PdfController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Lock\LockFactory;

class PdfController extends AbstractController
{
    #[Route('/download/{version}/terms-of-use.pdf')]
    public function downloadPdf($version, LockFactory $lockFactory, MyPdfGeneratorService $pdf): Response
    {
        $lock = $lockFactory->createLock('pdf-creation-'.$version);
        $lock->acquire(true);

        // 大量计算
        $myPdf = $pdf->getOrCreatePdf($version);

        $lock->release();

        // ...
    }
}
```

---

## 命名锁 {#lock-named-locks}

如果应用需要不同种类的存储并排使用，Symfony 提供命名锁：

```yaml
# config/packages/lock.yaml
framework:
    lock:
        invoice: ['semaphore', 'redis://r2.docker']
        report: 'semaphore'
```

配置一个或多个命名锁后，你有两种方式在任何服务或控制器中注入它们：

**（1）使用特定的参数名称**

将你的构造函数/方法参数类型提示为 `LockFactory`，并使用此模式命名参数："camelCase 的锁名称" + `LockFactory` 后缀。例如，要注入之前定义的 `invoice` 包：

```php
use Symfony\Component\Lock\LockFactory;

class SomeService
{
    public function __construct(
        private LockFactory $invoiceLockFactory
    ) {
        // ...
    }
}
```

**（2）使用 `#[Target]` 属性**

当处理同类型的多个实现时，`#[Target]` 属性帮助你选择要注入的那个。Symfony 创建一个与锁同名的目标。

例如，要选择之前定义的 `invoice` 锁：

```php
// ...
use Symfony\Component\DependencyInjection\Attribute\Target;

class SomeService
{
    public function __construct(
        #[Target('invoice')] private LockFactory $lockFactory
    ) {
        // ...
    }
}
```
