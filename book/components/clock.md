# Clock 组件

Clock 组件将应用与系统时钟解耦。这允许您固定时间以提高时间敏感逻辑的可测试性。

该组件为不同的用例提供了一个 `ClockInterface` 及以下实现：

`Symfony\Component\Clock\NativeClock`  
提供与系统时钟交互的方式，这与执行 `new \DateTimeImmutable()` 相同。

`Symfony\Component\Clock\MockClock`  
通常在测试中用作 `NativeClock` 的替代品，能够使用 `sleep()` 或 `modify()` 冻结和更改当前时间。

`Symfony\Component\Clock\MonotonicClock`  
依赖于 `hrtime()` 并提供高分辨率的单调时钟，当您需要精确的秒表时使用。

## 安装

```bash
$ composer require symfony/clock
```

如果您在 Symfony 应用之外使用此组件，则必须在代码中引入 Composer 生成的 `vendor/autoload.php` 文件来启用类自动加载机制。更多信息请阅读[此文章](./using_components.md)。

## 用法

`Symfony\Component\Clock\Clock` 类返回当前时间，并允许您在应用中使用任何兼容 [PSR-20][psr-20] 的实现作为全局时钟：

```php
use Symfony\Component\Clock\Clock;
use Symfony\Component\Clock\MockClock;

// 默认情况下，Clock 使用 NativeClock 实现，但您可以通过设置任何其他实现来更改它
Clock::set(new MockClock());

// 然后，您可以获取时钟实例
$clock = Clock::get();

// 此外，您可以设置时区
$clock->withTimeZone('Europe/Paris');

// 从这里，您可以获取当前时间
$now = $clock->now();

// 并休眠任意秒数
$clock->sleep(2.5);
```

Clock 组件还提供了 `now()` 函数：

```php
use function Symfony\Component\Clock\now;

// 获取当前时间作为 DatePoint 实例
$now = now();
```

`now()` 函数采用可选的 `modifier` 参数，该参数将应用于当前时间：

```php
$later = now('+3 hours');

$yesterday = now('-1 day');
```

您可以使用 [DateTime 构造函数接受的任何字符串][datetime-formats]。

在本页稍后，您可以了解如何在服务和测试中使用此时钟。使用 Clock 组件时，您操作 `Symfony\Component\Clock\DatePoint` 实例。您可以在[专门的章节](#datepoint-类)中了解更多信息。

## 可用的时钟实现

Clock 组件提供了一些 `Symfony\Component\Clock\ClockInterface` 的即用型实现，您可以根据需要在应用中将它们用作全局时钟。

### NativeClock

时钟服务替代为当前时间创建新的 `DateTime` 或 `DateTimeImmutable` 对象。相反，您注入 `ClockInterface` 并调用 `now()`。默认情况下，您的应用可能会使用 `NativeClock`，它始终返回当前系统时间。在测试中，它被 `MockClock` 替换。

以下示例介绍了一个利用 Clock 组件确定当前时间的服务：

```php
use Symfony\Component\Clock\ClockInterface;

class ExpirationChecker
{
    public function __construct(
        private ClockInterface $clock
    ) {}

    public function isExpired(DateTimeInterface $validUntil): bool
    {
        return $this->clock->now() > $validUntil;
    }
}
```

### MockClock

`MockClock` 使用时间实例化，并且不会自行向前移动。时间是固定的，直到调用 `sleep()` 或 `modify()`。这使您可以完全控制代码假定的当前时间。

在为此服务编写测试时，您可以通过修改时钟的时间来检查某物是否过期或未过期的两种情况：

```php
use PHPUnit\Framework\TestCase;
use Symfony\Component\Clock\MockClock;

class ExpirationCheckerTest extends TestCase
{
    public function testIsExpired(): void
    {
        $clock = new MockClock('2022-11-16 15:20:00');
        $expirationChecker = new ExpirationChecker($clock);
        $validUntil = new DateTimeImmutable('2022-11-16 15:25:00');

        // $validUntil 在未来，所以它没有过期
        $this->assertFalse($expirationChecker->isExpired($validUntil));

        // Clock 休眠 10 分钟，所以现在是 '2022-11-16 15:30:00'
        $clock->sleep(600); // 立即更改时间，就像我们等待了 10 分钟（600 秒）

        $this->assertTrue($expirationChecker->isExpired($validUntil));

        // 修改时钟，接受 DateTimeImmutable::modify() 支持的所有格式
        $clock->modify('2022-11-16 15:00:00');

        // $validUntil 再次在未来，所以它不再过期
        $this->assertFalse($expirationChecker->isExpired($validUntil));
    }
}
```

### 单调时钟（Monotonic Clock）

`MonotonicClock` 允许您实现精确的秒表；根据系统最高可达纳秒精度。它可用于测量两次调用之间的经过时间，而不受系统时钟有时引入的不一致性的影响，例如通过更新它。相反，它始终如一地增加时间，使其特别适用于测量性能。

## 在服务中使用时钟

在服务中使用 Clock 组件检索当前时间使它们更易于测试。例如，通过在测试期间使用 `MockClock` 实现作为默认实现，您将完全控制将"当前时间"设置为任意日期/时间。

为了在服务中使用此组件，使您的类使用 `Symfony\Component\Clock\ClockAwareTrait`。由于服务自动配置，trait 的 `setClock()` 方法将由服务容器自动调用。

现在您可以调用 `$this->now()` 方法来获取当前时间：

```php
namespace App\TimeUtils;

use Symfony\Component\Clock\ClockAwareTrait;

class MonthSensitive
{
    use ClockAwareTrait;

    public function isWinterMonth(): bool
    {
        $now = $this->now();

        return match ($now->format('F')) {
            'December', 'January', 'February', 'March' => true,
            default => false,
        };
    }
}
```

由于 `ClockAwareTrait`，并通过使用 `MockClock` 实现，您可以任意设置当前时间，而无需更改服务代码。这将帮助您测试方法的每种情况，而无需实际处于某个月或另一个月。

## `DatePoint` 类

Clock 组件使用特殊的 `Symfony\Component\Clock\DatePoint` 类。这是 PHP 的 `DateTimeImmutable` 之上的一个小包装器。您可以在期望 `DateTimeImmutable` 或 `DateTimeInterface` 的任何地方无缝使用它。`DatePoint` 对象从 `Symfony\Component\Clock\Clock` 类获取日期和时间。这意味着如果您按照[用法部分](#用法)中所述对时钟进行了任何更改，它将在创建新的 `DatePoint` 时反映出来。您也可以直接创建新的 `DatePoint` 实例，例如在将其用作默认值时：

```php
use Symfony\Component\Clock\DatePoint;

class Post
{
    public function __construct(
        // ...
        private \DateTimeImmutable $createdAt = new DatePoint(),
    ) {
    }
}
```

构造函数还允许设置时区或自定义参考日期：

```php
// 您可以指定时区
$withTimezone = new DatePoint(timezone: new \DateTimezone('UTC'));

// 您也可以从参考日期创建 DatePoint
$referenceDate = new \DateTimeImmutable();
$relativeDate = new DatePoint('+1month', reference: $referenceDate);
```

`DatePoint` 类还提供了一个命名构造函数来从时间戳创建日期：

```php
$dateOfFirstCommitToSymfonyProject = DatePoint::createFromTimestamp(1129645656);
// 等同于：
// $dateOfFirstCommitToSymfonyProject = (new \DateTimeImmutable())->setTimestamp(1129645656);

// 也支持负时间戳（用于 1970 年 1 月 1 日之前的日期）和浮点时间戳（用于高精度亚秒级日期时间）
$dateOfFirstMoonLanding = DatePoint::createFromTimestamp(-14182940);
```

> [!NOTE]
> 此外，`DatePoint` 提供了更严格的返回类型，并通过填充 [PHP 8.3 的行为][php-83-behavior]在该主题上提供了跨 PHP 版本的一致错误处理。

`DatePoint` 还允许您设置和获取日期和时间的微秒部分：

```php
$datePoint = new DatePoint();
$datePoint->setMicrosecond(345);
$microseconds = $datePoint->getMicrosecond();
```

> [!NOTE]
> 此功能填充了 PHP 8.4 在该主题上的行为，因为微秒操作在以前版本的 PHP 中不可用。

### 在数据库中存储 DatePoints

如果您使用 Doctrine 来处理数据库，请考虑使用新的 Doctrine 类型：

| DatePoint Doctrine 类型 | 扩展 Doctrine 类型 | 类 |
|---|---|---|
| `date_point` | `datetime_immutable` | `Symfony\Bridge\Doctrine\Types\DatePointType` |
| `day_point` | `date_immutable` | `Symfony\Bridge\Doctrine\Types\DayPointType` |
| `time_point` | `time_immutable` | `Symfony\Bridge\Doctrine\Types\TimePointType` |

它们自动转换为/从 `DatePoint` 对象：

```php
// src/Entity/Product.php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Clock\DatePoint;

#[ORM\Entity]
class Product
{
    // 如果您不显式定义 Doctrine 类型，Symfony 将自动检测 'date_point'：
    #[ORM\Column]
    private DatePoint $createdAt;

    // 如果您更喜欢显式定义 Doctrine 类型：
    #[ORM\Column(type: 'date_point')]
    private DatePoint $updatedAt;

    #[ORM\Column(type: 'day_point')]
    public DatePoint $birthday;

    #[ORM\Column(type: 'time_point')]
    public DatePoint $openAt;

    // ...
}
```

## 编写时间敏感测试

Clock 组件提供另一个 trait，称为 `Symfony\Component\Clock\Test\ClockSensitiveTrait`，以帮助您编写时间敏感测试。此 trait 提供了冻结时间和在每次测试后恢复全局时钟的方法。

使用 `ClockSensitiveTrait::mockTime()` 方法在测试中与模拟时钟交互。此方法接受不同类型作为其唯一参数：

* 字符串，可以是设置时钟的日期（例如 `1996-07-01`）或修改时钟的间隔（例如 `+2 days`）；
* `DateTimeImmutable` 以设置时钟；
* 布尔值，以冻结或恢复全局时钟。

假设您想测试上述示例的 `MonthSensitive::isWinterMonth()` 方法。以下是您可以编写该测试的方式：

```php
namespace App\Tests\TimeUtils;

use App\TimeUtils\MonthSensitive;
use PHPUnit\Framework\TestCase;
use Symfony\Component\Clock\Test\ClockSensitiveTrait;

class MonthSensitiveTest extends TestCase
{
    use ClockSensitiveTrait;

    public function testIsWinterMonth(): void
    {
        $clock = static::mockTime(new \DateTimeImmutable('2022-03-02'));

        $monthSensitive = new MonthSensitive();
        $monthSensitive->setClock($clock);

        $this->assertTrue($monthSensitive->isWinterMonth());
    }

    public function testIsNotWinterMonth(): void
    {
        $clock = static::mockTime(new \DateTimeImmutable('2023-06-02'));

        $monthSensitive = new MonthSensitive();
        $monthSensitive->setClock($clock);

        $this->assertFalse($monthSensitive->isWinterMonth());
    }
}
```

无论您在一年中的哪个时间运行此测试，它都将表现相同。通过结合 `Symfony\Component\Clock\ClockAwareTrait` 和 `Symfony\Component\Clock\Test\ClockSensitiveTrait`，您可以完全控制时间敏感代码的行为。

## 异常管理

Clock 组件充分利用了一些 [PHP DateTime 异常][php-datetime-exceptions]。如果您向时钟传递无效字符串（例如在创建时钟或修改 `MockClock` 时），您将得到 `DateMalformedStringException`。如果您传递无效时区，您将得到 `DateInvalidTimeZoneException`：

```php
$userInput = 'invalid timezone';

try {
    $clock = Clock::get()->withTimeZone($userInput);
} catch (\DateInvalidTimeZoneException $exception) {
    // ...
}
```

这些异常从 PHP 8.3 开始可用。但是，由于 Clock 组件所需的 [symfony/polyfill-php83][polyfill-php83] 依赖项，即使您的项目尚未使用 PHP 8.3，您也可以使用它们。

[psr-20]: https://www.php-fig.org/psr/psr-20/
[datetime-formats]: https://www.php.net/manual/en/datetime.formats.php
[php-datetime-exceptions]: https://wiki.php.net/rfc/datetime-exceptions
[polyfill-php83]: https://github.com/symfony/polyfill-php83
[php-83-behavior]: https://wiki.php.net/rfc/datetime-exceptions
