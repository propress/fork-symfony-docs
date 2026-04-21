# Clock 组件

Clock 组件将应用程序与系统时钟解耦。这允许你固定时间以提高时间敏感逻辑的可测试性。

该组件提供了一个 `ClockInterface`，并针对不同使用场景提供以下实现：

`Symfony\Component\Clock\NativeClock`
: 提供与系统时钟交互的方式，与执行 `new \DateTimeImmutable()` 相同。

`Symfony\Component\Clock\MockClock`
: 通常在测试中用作 `NativeClock` 的替代品，以便使用 `sleep()` 或 `modify()` 冻结和更改当前时间。

`Symfony\Component\Clock\MonotonicClock`
: 依赖 `hrtime()` 并提供高分辨率的单调时钟，当你需要精确的秒表时使用。

## 安装

```terminal
$ composer require symfony/clock
```

## 使用

`Symfony\Component\Clock\Clock` 类返回当前时间，并允许你在应用程序中使用任何兼容 [PSR-20] 的实现作为全局时钟：

```php
use Symfony\Component\Clock\Clock;
use Symfony\Component\Clock\MockClock;

// by default, Clock uses the NativeClock implementation, but you can change
// this by setting any other implementation
Clock::set(new MockClock());

// Then, you can get the clock instance
$clock = Clock::get();

// Additionally, you can set a timezone
$clock->withTimeZone('Europe/Paris');

// From here, you can get the current time
$now = $clock->now();

// And sleep for any number of seconds
$clock->sleep(2.5);
```

Clock 组件还提供了 `now()` 函数：

```php
use function Symfony\Component\Clock\now;

// Get the current time as a DatePoint instance
$now = now();
```

`now()` 函数接受一个可选的 `modifier` 参数，该参数将应用于当前时间：

```php
$later = now('+3 hours');

$yesterday = now('-1 day');
```

你可以使用 [DateTime 构造函数接受]的任何字符串。

在本页后面，你可以了解如何在服务和测试中使用此时钟。使用 Clock 组件时，你将操作 `Symfony\Component\Clock\DatePoint` 实例。你可以在专用部分中了解更多信息。

## 可用的时钟实现

Clock 组件提供了一些 `Symfony\Component\Clock\ClockInterface` 的即用实现，你可以根据需要在应用程序中将其用作全局时钟。

### NativeClock

时钟服务替代了为当前时间创建新的 `DateTime` 或 `DateTimeImmutable` 对象的做法。你注入 `ClockInterface` 并调用 `now()` 来代替。默认情况下，你的应用程序可能会使用 `NativeClock`，它始终返回当前系统时间。在测试中，它会被替换为 `MockClock`。

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

`MockClock` 用某个时间实例化，不会自行向前移动。时间是固定的，直到调用 `sleep()` 或 `modify()`。这让你对代码假定的当前时间拥有完全控制。

在为此服务编写测试时，你可以通过修改时钟的时间来检查某物过期和未过期的两种情况：

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

        // $validUntil is in the future, so it is not expired
        $this->assertFalse($expirationChecker->isExpired($validUntil));

        // Clock sleeps for 10 minutes, so now is '2022-11-16 15:30:00'
        $clock->sleep(600); // Instantly changes time as if we waited for 10 minutes (600 seconds)

        $this->assertTrue($expirationChecker->isExpired($validUntil));

        // modify the clock, accepts all formats supported by DateTimeImmutable::modify()
        $clock->modify('2022-11-16 15:00:00');

        // $validUntil is in the future again, so it is no longer expired
        $this->assertFalse($expirationChecker->isExpired($validUntil));
    }
}
```

### Monotonic Clock

`MonotonicClock` 允许你实现精确的秒表；根据系统的不同，精度可达纳秒级别。它可以用于测量两次调用之间的经过时间，而不受系统时钟有时引入的不一致性（例如更新系统时钟）的影响。相反，它会持续增加时间，使其特别适合用于性能测量。

## 在服务中使用时钟

在服务中使用 Clock 组件检索当前时间使它们更容易测试。例如，在测试期间使用 `MockClock` 实现作为默认实现，你将对将"当前时间"设置为任意日期/时间拥有完全控制。

要在服务中使用此组件，请让类使用 `Symfony\Component\Clock\ClockAwareTrait`。得益于服务自动配置，该 trait 的 `setClock()` 方法将由服务容器自动调用。

现在你可以调用 `$this->now()` 方法来获取当前时间：

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

得益于 `ClockAwareTrait` 以及使用 `MockClock` 实现，你可以任意设置当前时间而无需更改服务代码。这将帮助你测试方法的每种情况，而无需实际处于某个月份。

## `DatePoint` 类

Clock 组件使用一个特殊的 `Symfony\Component\Clock\DatePoint` 类。这是 PHP 的 `DateTimeImmutable` 的一个小型包装器。你可以在任何期望 `DateTimeImmutable` 或 `DateTimeInterface` 的地方无缝使用它。`DatePoint` 对象从 `Symfony\Component\Clock\Clock` 类获取日期和时间。这意味着如果你按照使用部分所述对时钟进行了任何更改，创建新的 `DatePoint` 时将会反映这些更改。你也可以直接创建新的 `DatePoint` 实例，例如将其用作默认值时：

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
// you can specify a timezone
$withTimezone = new DatePoint(timezone: new \DateTimezone('UTC'));

// you can also create a DatePoint from a reference date
$referenceDate = new \DateTimeImmutable();
$relativeDate = new DatePoint('+1month', reference: $referenceDate);
```

`DatePoint` 类还提供了一个命名构造函数，可以从时间戳创建日期：

```php
$dateOfFirstCommitToSymfonyProject = DatePoint::createFromTimestamp(1129645656);
// equivalent to:
// $dateOfFirstCommitToSymfonyProject = (new \DateTimeImmutable())->setTimestamp(1129645656);

// negative timestamps (for dates before January 1, 1970) and float timestamps
// (for high precision sub-second datetimes) are also supported
$dateOfFirstMoonLanding = DatePoint::createFromTimestamp(-14182940);
```

> **注意：**
> 此外，`DatePoint` 提供了更严格的返回类型，并通过填充 [PHP 8.3 的行为]提供了跨 PHP 版本一致的错误处理。

`DatePoint` 还允许你设置和获取日期时间的微秒部分：

```php
$datePoint = new DatePoint();
$datePoint->setMicrosecond(345);
$microseconds = $datePoint->getMicrosecond();
```

> **注意：**
> 此功能填充了 PHP 8.4 的行为，因为之前版本的 PHP 不支持微秒操作。

### 在数据库中存储 DatePoint

如果你[使用 Doctrine](/doctrine) 处理数据库，请考虑使用新的 Doctrine 类型：

| DatePoint Doctrine 类型 | 扩展 Doctrine 类型 | 类 |
|---|---|---|
| `date_point` | `datetime_immutable` | `Symfony\Bridge\Doctrine\Types\DatePointType` |
| `day_point` | `date_immutable` | `Symfony\Bridge\Doctrine\Types\DayPointType` |
| `time_point` | `time_immutable` | `Symfony\Bridge\Doctrine\Types\TimePointType` |

它们自动与 `DatePoint` 对象之间进行转换：

```php
// src/Entity/Product.php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Clock\DatePoint;

#[ORM\Entity]
class Product
{
    // if you don't define the Doctrine type explicitly, Symfony will autodetect 'date_point':
    #[ORM\Column]
    private DatePoint $createdAt;

    // if you prefer to define the Doctrine type explicitly:
    #[ORM\Column(type: 'date_point')]
    private DatePoint $updatedAt;

    #[ORM\Column(type: 'day_point')]
    public DatePoint $birthday;

    #[ORM\Column(type: 'time_point')]
    public DatePoint $openAt;

    // ...
}
```

## 编写时间敏感的测试

Clock 组件提供了另一个 trait，称为 `Symfony\Component\Clock\Test\ClockSensitiveTrait`，帮助你编写时间敏感的测试。该 trait 提供了冻结时间和在每次测试后恢复全局时钟的方法。

使用 `ClockSensitiveTrait::mockTime()` 方法在测试中与模拟时钟交互。该方法接受不同类型作为其唯一参数：

* 字符串，可以是要将时钟设置到的日期（例如 `1996-07-01`）或修改时钟的间隔（例如 `+2 days`）；
* `DateTimeImmutable`，将时钟设置到该时间；
* 布尔值，用于冻结或恢复全局时钟。

假设你想测试上面示例中 `MonthSensitive::isWinterMonth()` 方法。你可以这样编写该测试：

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

无论你在一年中的什么时候运行，此测试的行为都是相同的。通过结合 `Symfony\Component\Clock\ClockAwareTrait` 和 `Symfony\Component\Clock\Test\ClockSensitiveTrait`，你对时间敏感代码的行为拥有完全控制。

## 异常管理

Clock 组件充分利用了一些 [PHP DateTime 异常]。如果你向时钟传递无效字符串（例如在创建时钟或修改 `MockClock` 时），你会得到 `DateMalformedStringException`。如果你传递无效的时区，你会得到 `DateInvalidTimeZoneException`：

```php
$userInput = 'invalid timezone';

try {
    $clock = Clock::get()->withTimeZone($userInput);
} catch (\DateInvalidTimeZoneException $exception) {
    // ...
}
```

这些异常从 PHP 8.3 开始可用。但是，得益于 Clock 组件所需的 [symfony/polyfill-php83] 依赖，即使你的项目尚未使用 PHP 8.3，你也可以使用它们。

[PSR-20]: https://www.php-fig.org/psr/psr-20/
[DateTime 构造函数接受]: https://www.php.net/manual/en/datetime.formats.php
[PHP DateTime 异常]: https://wiki.php.net/rfc/datetime-exceptions
[symfony/polyfill-php83]: https://github.com/symfony/polyfill-php83
[PHP 8.3 的行为]: https://wiki.php.net/rfc/datetime-exceptions
