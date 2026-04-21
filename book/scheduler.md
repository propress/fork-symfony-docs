# 调度器（Scheduler）

> **提示：** 喜欢视频教程？请查看 [Scheduler 快速入门视频教程](https://symfonycasts.com/screencast/mailtrap/bonus-symfony-scheduler)。

调度器组件用于管理 PHP 应用程序中的任务调度，例如每晚凌晨 3 点执行一个任务、每两周执行一次（节假日除外），或任何其他你需要的自定义调度。

该组件适用于调度以下类型的任务：维护任务（数据库清理、缓存清除等）、后台处理（队列处理、数据同步等）、定期数据更新、定时通知（邮件、告警）等。

本文档重点介绍在完整 Symfony 应用程序中使用调度器组件。

## 安装

运行以下命令安装调度器组件：

```bash
$ composer require symfony/scheduler
```

> **注意：** 在使用 [Symfony Flex](https://symfony.com/doc/current/setup.html#symfony-flex) 的应用程序中，安装该组件还会创建一个初始调度，可以直接开始添加任务。

## Symfony 调度器基础

使用该组件的主要好处是，自动化由你的应用程序来管理，这给了你极大的灵活性，而这是 cron 作业无法实现的（例如，基于某些条件的动态调度）。

从本质上讲，调度器组件允许你创建一个由服务执行并按某种调度重复运行的任务（称为消息）。它与 [Symfony Messenger](/components/messenger) 组件有一些相似之处（如消息、处理器、总线、传输等），但主要区别在于 Messenger 无法处理按固定间隔重复执行的任务。

以下面这个应用程序为例，它按计划向客户发送报告。首先，创建一个代表生成报告任务的调度器消息：

```php
// src/Scheduler/Message/SendDailySalesReports.php
namespace App\Scheduler\Message;

class SendDailySalesReports
{
    public function __construct(private int $id) {}

    public function getId(): int
    {
        return $this->id;
    }
}
```

接下来，创建处理该类型消息的处理器：

```php
// src/Scheduler/Handler/SendDailySalesReportsHandler.php
namespace App\Scheduler\Handler;

use App\Scheduler\Message\SendDailySalesReports;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
class SendDailySalesReportsHandler
{
    public function __invoke(SendDailySalesReports $message)
    {
        // ... 执行一些工作，将报告发送给客户
    }
}
```

与 Messenger 组件立即发送这些消息不同，这里的目标是根据预定义的频率来创建它们。这得益于 `Symfony\Component\Scheduler\Messenger\SchedulerTransport`，这是一个专用于调度器消息的特殊传输。

该传输会根据分配的频率自主生成各种消息。以下图片展示了 Messenger 和 Scheduler 组件在消息处理上的区别：

在 Messenger 中：

![Symfony Messenger 基本循环](/_images/components/messenger/basic_cycle.png)

在 Scheduler 中：

![Symfony Scheduler 基本循环](/_images/components/scheduler/scheduler_cycle.png)

另一个重要区别是，调度器组件中的消息是循环的。它们通过 `Symfony\Component\Scheduler\RecurringMessage` 类来表示。

## 将循环消息附加到调度

消息频率的配置存储在实现了 `Symfony\Component\Scheduler\ScheduleProviderInterface` 接口的类中。该提供者使用 `Symfony\Component\Scheduler\ScheduleProviderInterface::getSchedule` 方法返回一个包含各种循环消息的调度对象。

`Symfony\Component\Scheduler\Attribute\AsSchedule` 特性（attribute）默认引用名为 `default` 的调度，允许你注册到特定调度上：

```php
// src/Scheduler/SaleTaskProvider.php
namespace App\Scheduler;

use Symfony\Component\Scheduler\Attribute\AsSchedule;
use Symfony\Component\Scheduler\Schedule;
use Symfony\Component\Scheduler\ScheduleProviderInterface;

#[AsSchedule]
class SaleTaskProvider implements ScheduleProviderInterface
{
    public function getSchedule(): Schedule
    {
        // ...
    }
}
```

> **提示：** 调度名称必须唯一，默认值为 `default`。传输名称的语法为：`scheduler_你的调度名称`（例如 `scheduler_default`）。

> **提示：** [记忆化（Memoizing）](https://en.wikipedia.org/wiki/Memoization) 你的调度是个好习惯，可以避免在 `getSchedule()` 方法被其他服务检查时进行不必要的重建。

## 调度循环消息

`RecurringMessage` 是一个与触发器关联的消息，触发器配置了消息的频率。Symfony 提供了不同类型的触发器：

**`Symfony\Component\Scheduler\Trigger\CronExpressionTrigger`**
使用与 [cron 命令行工具](https://en.wikipedia.org/wiki/Cron) 相同语法的触发器。

**`Symfony\Component\Scheduler\Trigger\CallbackTrigger`**
使用回调函数来确定下次运行日期的触发器。

**`Symfony\Component\Scheduler\Trigger\ExcludeTimeTrigger`**
从给定触发器中排除某些时间的触发器。

**`Symfony\Component\Scheduler\Trigger\JitterTrigger`**
为给定触发器添加随机抖动的触发器。抖动是添加到原始触发日期/时间的一段时间，这允许分散计划任务的负载，而不是在完全相同的时间运行所有任务。

**`Symfony\Component\Scheduler\Trigger\PeriodicalTrigger`**
使用 `DateInterval` 来确定下次运行日期的触发器。

`Symfony\Component\Scheduler\Trigger\JitterTrigger` 和 `Symfony\Component\Scheduler\Trigger\ExcludeTimeTrigger` 是装饰器，用于修改它们所包装的触发器的行为。你可以通过调用 `Symfony\Component\Scheduler\Trigger\AbstractDecoratedTrigger::inner` 和 `Symfony\Component\Scheduler\Trigger\AbstractDecoratedTrigger::decorators` 方法来获取被装饰的触发器以及各个装饰器：

```php
$trigger = new ExcludeTimeTrigger(new JitterTrigger(CronExpressionTrigger::fromSpec('#midnight', new MyMessage()));

$trigger->inner(); // CronExpressionTrigger
$trigger->decorators(); // [ExcludeTimeTrigger, JitterTrigger]
```

其中大多数可以通过 `Symfony\Component\Scheduler\RecurringMessage` 类来创建，如以下示例所示。

### Cron 表达式触发器

在使用 cron 触发器之前，你需要安装以下依赖：

```bash
$ composer require dragonmantank/cron-expression
```

然后，使用与 [cron 命令行工具](https://en.wikipedia.org/wiki/Cron) 相同的语法定义触发器的日期/时间：

```php
RecurringMessage::cron('* * * * *', new Message());

// 你也可以定义 cron 表达式使用的时区
RecurringMessage::cron('* * * * *', new Message(), new \DateTimeZone('Africa/Malabo'));
```

> **提示：** 如果你需要帮助构建或理解 cron 表达式，请查看 [crontab.guru 网站](https://crontab.guru/)。

你也可以使用一些代表常见 cron 表达式的特殊值：

* `@yearly`、`@annually` - 每年运行一次，1 月 1 日午夜 - `0 0 1 1 *`
* `@monthly` - 每月运行一次，每月第一天午夜 - `0 0 1 * *`
* `@weekly` - 每周运行一次，周日午夜 - `0 0 * * 0`
* `@daily`、`@midnight` - 每天运行一次，午夜 - `0 0 * * *`
* `@hourly` - 每小时运行一次，第一分钟 - `0 * * * *`

例如：

```php
RecurringMessage::cron('@daily', new Message());
```

> **提示：** 你也可以使用 [AsCronTask 特性](#ascrontask-示例) 来定义 cron 任务。

#### 哈希 Cron 表达式

如果你有许多触发器被安排在同一时间执行（例如，在午夜，`0 0 * * *`），这将在那个确切的时间创建一个非常长的调度运行列表。如果某个任务存在内存泄漏，这可能会引发问题。

你可以在表达式中添加哈希符号（`#`）来生成随机值。虽然这些值是随机的，但它们是可预测且一致的，因为它们基于消息生成。一个字符串表示为 `my task`、频率定义为 `# # * * *` 的消息，将会有一个幂等的频率 `56 20 * * *`（每天晚上 8:56）。

你也可以使用哈希范围（`#(x-y)`）来定义该随机部分的可能值列表。例如，`# #(0-7) * * *` 表示每天在午夜到凌晨 7 点之间的某个时间。不带范围的 `#` 会为该字段创建任何有效值的范围。`# # # # #` 是 `#(0-59) #(0-23) #(1-28) #(1-12) #(0-6)` 的简写。

你也可以使用一些代表常见哈希 cron 表达式的特殊值：

| 别名 | 转换为 |
|------|--------|
| `#hourly` | `# * * * *`（每小时的某分钟） |
| `#daily` | `# # * * *`（每天某个时间） |
| `#weekly` | `# # * * #`（每周某个时间） |
| `#weekly@midnight` | `# #(0-2) * * #`（每周某天的 `#midnight`） |
| `#monthly` | `# # # * *`（每月某天某时间） |
| `#monthly@midnight` | `# #(0-2) # * *`（每月某天的 `#midnight`） |
| `#annually` | `# # # # *`（每年某天某时间） |
| `#annually@midnight` | `# #(0-2) # # *`（每年某天的 `#midnight`） |
| `#yearly` | `# # # # *`（`#annually` 的别名） |
| `#yearly@midnight` | `# #(0-2) # # *`（`#annually@midnight` 的别名） |
| `#midnight` | `# #(0-2) * * *`（每天午夜到凌晨 2:59 之间的某个时间） |

例如：

```php
RecurringMessage::cron('#midnight', new Message());
```

> **注意：** 月中天数的范围是 `1-28`，这是为了考虑到最少有 28 天的二月。

### 周期性触发器

这些触发器允许你使用不同的数据类型（`string`、`integer`、`DateInterval`）来配置频率。它们也支持 PHP datetime 函数定义的[相对格式](https://www.php.net/manual/en/datetime.formats.php#datetime.formats.relative)：

```php
RecurringMessage::every('10 seconds', new Message());
RecurringMessage::every('3 weeks', new Message());
RecurringMessage::every('first Monday of next month', new Message());
```

> **注意：** `every()` 方法不支持以逗号分隔的星期（例如，`'Monday, Thursday, Saturday'`）。对于多个星期，请改用 cron 表达式：
>
> ```diff
> - RecurringMessage::every('Monday, Thursday, Saturday', new Message());
> + RecurringMessage::cron('5 12 * * 1,4,6', new Message());
> ```

> **提示：** 你也可以使用 [AsPeriodicTask 特性](#asperiodictask-示例) 来定义周期性任务。

你也可以为你的调度定义 `from` 和 `until` 时间：

```php
// 每天 13:00 创建一条消息
$from = new \DateTimeImmutable('13:00', new \DateTimeZone('Europe/Paris'));
RecurringMessage::every('1 day', new Message(), $from);

// 每天创建一条消息，直到特定日期
$until = '2023-06-12';
RecurringMessage::every('1 day', new Message(), null, $until);

// 结合 from 和 until 以进行更精确的控制
$from = new \DateTimeImmutable('2023-01-01 13:47', new \DateTimeZone('Europe/Paris'));
$until = '2023-06-12';
RecurringMessage::every('first Monday of next month', new Message(), $from, $until);
```

当启动调度器时，消息不会立即发送给 messenger。如果你不设置 `from` 参数，第一个频率周期从调度器运行的那一刻开始计算。例如，如果你在 8:33 启动它，并且消息被安排为每小时执行，那么它将在 9:33、10:33、11:33 等时间运行。

### 自定义触发器

自定义触发器允许你动态配置任意频率。它们被创建为实现了 `Symfony\Component\Scheduler\Trigger\TriggerInterface` 接口的服务。

例如，如果你想在节假日期间跳过每日发送客户报告：

```php
// src/Scheduler/Trigger/NewUserWelcomeEmailHandler.php
namespace App\Scheduler\Trigger;

class ExcludeHolidaysTrigger implements TriggerInterface
{
    public function __construct(private TriggerInterface $inner)
    {
    }

    // 使用此方法提供一个可显示的好名称，
    // 用于标识你的触发器（便于调试）
    public function __toString(): string
    {
        return $this->inner.' (except holidays)';
    }

    public function getNextRunDate(\DateTimeImmutable $run): ?\DateTimeImmutable
    {
        if (!$nextRun = $this->inner->getNextRunDate($run)) {
            return null;
        }

        // 循环直到找到不是假日的下次运行日期
        while ($this->isHoliday($nextRun)) {
            $nextRun = $this->inner->getNextRunDate($nextRun);
        }

        return $nextRun;
    }

    private function isHoliday(\DateTimeImmutable $timestamp): bool
    {
        // 添加一些逻辑来判断给定的 $timestamp 是否为假日
        // 如果是假日返回 true，否则返回 false
    }
}
```

然后，定义你的循环消息：

```php
RecurringMessage::trigger(
    new ExcludeHolidaysTrigger(
        CronExpressionTrigger::fromSpec('@daily'),
    ),
    new SendDailySalesReports('...'),
);
```

最后，循环消息需要附加到一个调度上：

```php
// src/Scheduler/SaleTaskProvider.php
namespace App\Scheduler;

#[AsSchedule('uptoyou')]
class SaleTaskProvider implements ScheduleProviderInterface
{
    public function getSchedule(): Schedule
    {
        return $this->schedule ??= new Schedule()
            ->with(
                RecurringMessage::trigger(
                    new ExcludeHolidaysTrigger(
                        CronExpressionTrigger::fromSpec('@daily'),
                    ),
                    new SendDailySalesReports()
                ),
                RecurringMessage::cron('3 8 * * 1', new CleanUpOldSalesReport())
            );
    }
}
```

因此，这个 `RecurringMessage` 将同时包含触发器（定义消息的生成频率）和消息本身（由特定处理器处理的消息）。

但有趣的是，它还提供了动态生成消息的能力。

### 动态生成消息

当消息依赖于存储在数据库或第三方服务中的数据时，这一特性尤为有用。

延续之前报告生成的示例：报告的生成依赖于客户请求。根据具体需求，可能需要按定义的频率生成任意数量的报告。对于这些动态场景，它让你能够动态地定义消息，而不是静态地定义。这是通过定义一个 `Symfony\Component\Scheduler\Trigger\CallbackMessageProvider` 来实现的。

本质上，这意味着你可以在运行时通过一个回调函数动态定义消息，该回调函数在每次调度器传输检查要生成的消息时执行：

```php
// src/Scheduler/SaleTaskProvider.php
namespace App\Scheduler;

#[AsSchedule('uptoyou')]
class SaleTaskProvider implements ScheduleProviderInterface
{
    public function getSchedule(): Schedule
    {
        return $this->schedule ??= new Schedule()
            ->with(
                RecurringMessage::trigger(
                    new ExcludeHolidaysTrigger(
                        CronExpressionTrigger::fromSpec('@daily'),
                    ),
                    // 与前面示例中的静态方式不同
                    new CallbackMessageProvider([$this, 'generateReports'], 'foo')
                ),
                RecurringMessage::cron('3 8 * * 1', new CleanUpOldSalesReport())
            );
    }

    public function generateReports(MessageContext $context)
    {
        // ...
        yield new SendDailySalesReports();
        yield new ReportSomethingReportSomethingElse();
    }
}
```

### 构建循环消息的其他方式

还有另一种构建 `RecurringMessage` 的方式，可以通过向服务或命令添加以下特性之一来实现：`Symfony\Component\Scheduler\Attribute\AsPeriodicTask` 和 `Symfony\Component\Scheduler\Attribute\AsCronTask`。

对于这两个特性，你可以通过 `schedule` 选项来定义要使用的调度。默认情况下，将使用名为 `default` 的调度。同样，默认情况下会调用服务的 `__invoke` 方法，但也可以通过 `method` 选项指定要调用的方法，并可以通过 `arguments` 选项定义参数（如有必要）。

#### `AsCronTask` 示例

这是使用该特性定义 cron 触发器的最基本方式：

```php
// src/Scheduler/Task/SendDailySalesReports.php
namespace App\Scheduler\Task;

use Symfony\Component\Scheduler\Attribute\AsCronTask;

#[AsCronTask('0 0 * * *')]
class SendDailySalesReports
{
    public function __invoke()
    {
        // ...
    }
}
```

该特性支持更多参数来自定义触发器：

```php
// 向触发时间随机添加最多 6 秒，以避免负载峰值
#[AsCronTask('0 0 * * *', jitter: 6)]

// 定义要调用的方法名以及要传递给它的参数
#[AsCronTask('0 0 * * *', method: 'sendEmail', arguments: ['email' => 'admin@example.com'])]

// 定义要使用的时区
#[AsCronTask('0 0 * * *', timezone: 'Africa/Malabo')]

// 当将此特性应用于 Symfony 控制台命令时，可以使用 'arguments' 选项
// 传递参数和选项给命令：
#[AsCronTask('0 0 * * *', arguments: 'some_argument --some-option --another-option=some_value')]
#[AsCommand(name: 'app:my-command')]
class MyCommand
```

#### `AsPeriodicTask` 示例

这是使用该特性定义周期性触发器的最基本方式：

```php
// src/Scheduler/Task/SendDailySalesReports.php
namespace App\Scheduler\Task;

use Symfony\Component\Scheduler\Attribute\AsPeriodicTask;

#[AsPeriodicTask(frequency: '1 day', from: '2022-01-01', until: '2023-06-12')]
class SendDailySalesReports
{
    public function __invoke()
    {
        // ...
    }
}
```

> **注意：** `from` 和 `until` 选项是可选的。如果未定义，该任务将无限期执行。

`#[AsPeriodicTask]` 特性支持许多参数来自定义触发器：

```php
// 频率可以定义为代表秒数的整数
#[AsPeriodicTask(frequency: 86400)]

// 向触发时间随机添加最多 6 秒，以避免负载峰值
#[AsPeriodicTask(frequency: '1 day', jitter: 6)]

// 定义要调用的方法名以及要传递给它的参数
#[AsPeriodicTask(frequency: '1 day', method: 'sendEmail', arguments: ['email' => 'admin@symfony.com'])]
class SendDailySalesReports
{
    public function sendEmail(string $email): void
    {
        // ...
    }
}

// 当将此特性应用于 Symfony 控制台命令时，可以使用 'arguments' 选项
// 传递参数和选项给命令：
#[AsPeriodicTask(frequency: '1 day', arguments: 'some_argument --some-option --another-option=some_value')]
#[AsCommand(name: 'app:my-command')]
class MyCommand
```

## 管理计划消息

### 实时修改计划消息

虽然提前规划调度是有益的，但调度很少会长期保持静态。在一定时间后，某些 `RecurringMessages` 可能会变得过时，而其他的可能需要被整合到计划中。

作为一般实践，为了减轻繁重的工作负载，调度中的循环消息被存储在内存中，以避免每次调度器传输生成消息时重新计算。然而，这种方法也有其不足之处。

延续上面相同的报告生成示例，公司可能在特定时期进行促销活动（需要在给定时间段内重复传达），或者在某些情况下需要停止删除旧报告。

这就是为什么 `Scheduler` 内置了一种动态修改调度并实时考虑所有更改的机制。

### 在调度中添加、删除和修改条目的策略

调度提供了 `Symfony\Component\Scheduler\Schedule::add`、`Symfony\Component\Scheduler\Schedule::remove` 或 `Symfony\Component\Scheduler\Schedule::clear` 方法，用于操作所有关联的循环消息，从而重置并重新计算内存中的循环消息堆栈。

例如，出于各种原因，如果不需要生成报告，可以使用回调来有条件地跳过部分或全部报告的生成。

然而，如果目的是完全删除一条循环消息及其重复，`Symfony\Component\Scheduler\Schedule` 提供了 `Symfony\Component\Scheduler\Schedule::remove` 或 `Symfony\Component\Scheduler\Schedule::removeById` 方法。这在你的场景中特别有用，尤其是当你需要停止循环消息的生成（即停止删除旧报告）时。

在处理器中，你可以检查某个条件，如果条件成立，则访问 `Symfony\Component\Scheduler\Schedule` 并调用该方法：

```php
// src/Scheduler/SaleTaskProvider.php
namespace App\Scheduler;

#[AsSchedule('uptoyou')]
class SaleTaskProvider implements ScheduleProviderInterface
{
    public function getSchedule(): Schedule
    {
        $this->removeOldReports = RecurringMessage::cron('3 8 * * 1', new CleanUpOldSalesReport());

        return $this->schedule ??= new Schedule()
            ->with(
                // ...
                $this->removeOldReports;
            );
    }

    // ...

    public function removeCleanUpMessage()
    {
        $this->getSchedule()->getSchedule()->remove($this->removeOldReports);
    }
}

// src/Scheduler/Handler/CleanUpOldSalesReportHandler.php
namespace App\Scheduler\Handler;

#[AsMessageHandler]
class CleanUpOldSalesReportHandler
{
    public function __invoke(CleanUpOldSalesReport $cleanUpOldSalesReport): void
    {
        // 在这里做一些工作...

        if ($isFinished) {
            $this->mySchedule->removeCleanUpMessage();
        }
    }
}
```

不过，这种系统可能并不适合所有场景。此外，处理器理想情况下应该被设计为处理它所针对的消息类型，而不必决定是否添加或删除新的循环消息。

例如，如果由于外部事件需要添加一条旨在删除报告的循环消息，在处理器内部实现可能很有挑战性。这是因为一旦没有该类型的消息，处理器将不再被调用或执行。

然而，调度器还有一个事件系统，它通过嫁接到 Symfony Messenger 事件上集成到 Symfony 完整堆栈应用程序中。这些事件通过监听器分发，提供了一种方便的响应手段。

## 通过事件管理计划消息

### 策略性事件处理

其目标是在保持解耦的同时，提供决定何时采取行动的灵活性。引入了三种主要事件类型：

* `PRE_RUN_EVENT`
* `POST_RUN_EVENT`
* `FAILURE_EVENT`

访问调度是一个关键特性，允许轻松添加或删除消息类型。此外，还可以访问当前处理的消息及其消息上下文。

考虑到我们的场景，你可以监听 `PRE_RUN_EVENT` 并检查某个条件是否满足。例如，你可能决定以相同或不同的配置再次添加一条用于清理旧报告的循环消息，或添加任何其他循环消息。

如果你选择在此处理循环消息的删除，可以在该事件的监听器中完成。重要的是，它揭示了一个特定的特性 `Symfony\Component\Scheduler\Event\PreRunEvent::shouldCancel`，它允许你阻止被删除的循环消息的消息被传输并由其处理器处理：

```php
// src/Scheduler/SaleTaskProvider.php
namespace App\Scheduler;

#[AsSchedule('uptoyou')]
class SaleTaskProvider implements ScheduleProviderInterface
{
    public function __construct(private EventDispatcherInterface $dispatcher)
    {
    }

    public function getSchedule(): Schedule
    {
        $this->removeOldReports = RecurringMessage::cron('3 8 * * 1', new CleanUpOldSalesReport());

        return $this->schedule ??= new Schedule($this->dispatcher)
            ->with(
                // ...
            )
            ->before(function(PreRunEvent $event) {
                $message = $event->getMessage();
                $messageContext = $event->getMessageContext();

                // 可以访问调度
                $schedule = $event->getSchedule()->getSchedule();

                // 可以直接定位正在处理的 RecurringMessage
                $schedule->removeById($messageContext->id);

                // 允许调用 ShouldCancel() 来避免处理消息
                $event->shouldCancel(true);
            })
            ->after(function(PostRunEvent $event) {
                // 做你想做的事
            })
            ->onFailure(function(FailureEvent $event) {
                // 做你想做的事
            });
    }
}
```

### 调度器事件

#### PreRunEvent

**事件类**：`Symfony\Component\Scheduler\Event\PreRunEvent`

`PreRunEvent` 允许你修改 `Symfony\Component\Scheduler\Schedule` 或在消息被消费之前取消它：

```php
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\Scheduler\Event\PreRunEvent;

public function onMessage(PreRunEvent $event): void
{
    $schedule = $event->getSchedule();
    $context = $event->getMessageContext();
    $message = $event->getMessage();

    // 对调度、上下文或消息做一些处理

    // 并/或取消消息
    $event->shouldCancel(true);
}
```

执行此命令以查看为该事件注册的监听器及其优先级：

```bash
$ php bin/console debug:event-dispatcher "Symfony\Component\Scheduler\Event\PreRunEvent"
```

#### PostRunEvent

**事件类**：`Symfony\Component\Scheduler\Event\PostRunEvent`

`PostRunEvent` 允许你在消息被消费后修改 `Symfony\Component\Scheduler\Schedule`：

```php
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\Scheduler\Event\PostRunEvent;

public function onMessage(PostRunEvent $event): void
{
    $schedule = $event->getSchedule();
    $context = $event->getMessageContext();
    $message = $event->getMessage();
    $result = $event->getResult();

    // 对调度、上下文、消息或结果做一些处理
}
```

执行此命令以查看为该事件注册的监听器及其优先级：

```bash
$ php bin/console debug:event-dispatcher "Symfony\Component\Scheduler\Event\PostRunEvent"
```

#### FailureEvent

**事件类**：`Symfony\Component\Scheduler\Event\FailureEvent`

`FailureEvent` 允许你在消息消费抛出异常时修改 `Symfony\Component\Scheduler\Schedule`：

```php
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\Scheduler\Event\FailureEvent;

public function onMessage(FailureEvent $event): void
{
    $schedule = $event->getSchedule();
    $context = $event->getMessageContext();
    $message = $event->getMessage();

    $error = $event->getError();

    // 对调度、上下文、消息或错误做一些处理（日志记录等）

    // 并/或忽略失败事件
    $event->shouldIgnore(true);
}
```

执行此命令以查看为该事件注册的监听器及其优先级：

```bash
$ php bin/console debug:event-dispatcher "Symfony\Component\Scheduler\Event\FailureEvent"
```

## 消费消息

调度器组件提供两种消费消息的方式，具体取决于你的需求：使用 `messenger:consume` 命令或以编程方式创建 worker。第一种方案是在完整 Symfony 应用程序中使用调度器组件时推荐的方式，第二种方案更适合将调度器组件作为独立组件使用时。

### 运行 Worker

在定义并将循环消息附加到调度之后，你需要一种机制来根据其定义的频率生成和消费消息。为此，调度器组件使用 Messenger 组件中的 `messenger:consume` 命令：

```bash
$ php bin/console messenger:consume scheduler_nameofyourschedule

# 如果你需要了解发生了什么，请使用 -vv
$ php bin/console messenger:consume scheduler_nameofyourschedule -vv
```

![Symfony Scheduler - 生成和消费](/_images/components/scheduler/generate_consume.png)

> **提示：** 根据你的部署场景，你可能更喜欢使用 cron、Supervisor 或 systemd 等工具自动执行 Messenger worker 进程。这可以确保 worker 持续运行。更多详情，请参阅 Messenger 组件文档的[部署到生产环境](https://symfony.com/doc/current/messenger.html#deploying-to-production)章节。

### 以编程方式创建消费者

前一种方案的替代方式是创建并调用一个 worker 来消费消息。该组件附带了一个名为 `Symfony\Component\Scheduler\Scheduler` 的开箱即用的 worker，你可以在代码中使用它：

```php
use Symfony\Component\Scheduler\Scheduler;

$schedule = new Schedule()
    ->with(
        RecurringMessage::trigger(
            new ExcludeHolidaysTrigger(
                CronExpressionTrigger::fromSpec('@daily'),
            ),
            new SendDailySalesReports()
        ),
    );

$scheduler = new Scheduler(handlers: [
    SendDailySalesReports::class => new SendDailySalesReportsHandler(),
    // 如果你有更多消息类型，请添加更多处理器
], schedules: [
    $schedule,
    // 调度器可以接受任意数量的调度
]);

// 最后，在调度器准备好后运行它
$scheduler->run();
```

> **注意：** 当将调度器组件作为独立组件使用时，可以使用 `Symfony\Component\Scheduler\Scheduler`。如果你在框架上下文中使用它，强烈建议使用上一节中介绍的 `messenger:consume` 命令。

## 在运行时修改调度

当在调度中添加或删除循环消息时，调度器会自动重启并重新计算内部触发器堆。这使得在运行时动态控制计划任务成为可能：

```php
// src/Scheduler/DynamicScheduleProvider.php
namespace App\Scheduler;

#[AsSchedule('uptoyou')]
class DynamicScheduleProvider implements ScheduleProviderInterface
{
    private ?Schedule $schedule = null;

    public function getSchedule(): Schedule
    {
        return $this->schedule ??= new Schedule()
            ->with(
                // ...
            )
        ;
    }

    public function clearAndAddMessages(): void
    {
        // 清除当前调度并添加新的循环消息
        $this->schedule?->clear();
        $this->schedule?->add(
            RecurringMessage::cron('@hourly', new DoActionMessage()),
            RecurringMessage::cron('@daily', new DoAnotherActionMessage()),
        );
    }
}
```

## 调试调度

`debug:scheduler` 命令提供了调度及其循环消息的列表。你可以将列表缩小到特定的调度：

```bash
$ php bin/console debug:scheduler

  Scheduler
  =========

  default
  -------

    ------------------- ------------------------- ----------------------
    Trigger             Provider                  Next Run
    ------------------- ------------------------- ----------------------
    every 2 days        App\Messenger\Foo(0:17..)  Sun, 03 Dec 2023 ...
    15 4 */3 * *        App\Messenger\Foo(0:17..)  Mon, 18 Dec 2023 ...
   -------------------- -------------------------- ---------------------

# 你也可以指定用于下次运行日期的日期：
$ php bin/console debug:scheduler --date=2025-10-18

# 你也可以指定用于某个调度的下次运行日期：
$ php bin/console debug:scheduler name_of_schedule --date=2025-10-18

# 使用 --all 选项还可以显示已终止的循环消息
$ php bin/console debug:scheduler --all
```

## 使用 Symfony Scheduler 进行高效管理

当 worker 重启或关闭一段时间时，调度器传输将无法生成消息（因为它们是由调度器传输动态创建的）。这意味着在 worker 非活动期间计划发送的任何消息都不会被发送，调度器将丢失对上次处理消息的跟踪。在重启时，它将从那一刻起重新计算要生成的消息。

为了说明这一点，考虑一条每 3 天发送一次的循环消息。如果 worker 在第 2 天重启，该消息将在重启后 3 天，即第 5 天发送。

虽然这种行为不一定会造成问题，但它可能与你的预期不符。

这就是为什么调度器允许你通过 `stateful` 选项（以及[缓存组件](/components/cache)）记住消息的上次执行日期。这允许系统保留调度的状态，确保当 worker 重启时，它能从上次停止的地方继续：

```php
// src/Scheduler/SaleTaskProvider.php
namespace App\Scheduler;

#[AsSchedule('uptoyou')]
class SaleTaskProvider implements ScheduleProviderInterface
{
    public function getSchedule(): Schedule
    {
        $this->removeOldReports = RecurringMessage::cron('3 8 * * 1', new CleanUpOldSalesReport());

        return $this->schedule ??= new Schedule()
            ->with(
                // ...
            )
            ->stateful($this->cache)
    }
}
```

使用 `stateful` 选项后，所有未处理的消息都会被处理。如果你只需要处理一次消息，可以使用 `processOnlyLastMissedRun` 选项：

```php
// src/Scheduler/SaleTaskProvider.php
namespace App\Scheduler;

#[AsSchedule('uptoyou')]
class SaleTaskProvider implements ScheduleProviderInterface
{
    public function getSchedule(): Schedule
    {
        $this->removeOldReports = RecurringMessage::cron('3 8 * * 1', new CleanUpOldSalesReport());

        return $this->schedule ??= new Schedule()
            ->with(
                // ...
            )
            ->stateful($this->cache)
            ->processOnlyLastMissedRun(true)
    }
}
```

为了更有效地扩展你的调度，你可以使用多个 worker。在这种情况下，一个好的做法是添加一个[锁（lock）](/components/lock)来防止同一任务运行多次：

```php
// src/Scheduler/SaleTaskProvider.php
namespace App\Scheduler;

#[AsSchedule('uptoyou')]
class SaleTaskProvider implements ScheduleProviderInterface
{
    public function getSchedule(): Schedule
    {
        $this->removeOldReports = RecurringMessage::cron('3 8 * * 1', new CleanUpOldSalesReport());

        return $this->schedule ??= new Schedule()
            ->with(
                // ...
            )
            ->lock($this->lockFactory->createLock('my-lock'));
    }
}
```

> **提示：** 消息的处理时间很重要。如果处理时间很长，后续所有消息的处理都可能延迟。因此，一个好的做法是预估这一点，并将频率设置为大于消息处理时间的值。

此外，为了更好地扩展你的调度，你可以选择将你的消息包装在 `Symfony\Component\Messenger\Message\RedispatchMessage` 中。这允许你指定一个传输，你的消息将在该传输上被重新分发，然后再进一步分发到其对应的处理器：

```php
// src/Scheduler/SaleTaskProvider.php
namespace App\Scheduler;

#[AsSchedule('uptoyou')]
class SaleTaskProvider implements ScheduleProviderInterface
{
    public function getSchedule(): Schedule
    {
        return $this->schedule ??= new Schedule()
            ->with(
                RecurringMessage::every('5 seconds', new RedispatchMessage(new Message(), 'async'))
            );
    }
}
```

使用 `RedispatchMessage` 时，Symfony 会将 `Symfony\Component\Scheduler\Messenger\ScheduledStamp` 附加到消息上，帮助你在需要时识别这些消息。
