# Object Mapper（对象映射器）

该组件将一个对象转换为另一个对象，简化了将 DTO（数据传输对象）转换为实体或反之亦然等任务。在将 API 输入/输出与内部模型解耦时也很有帮助。

## 安装

```terminal
$ composer require symfony/object-mapper
```

---

## 基本用法

注入 `ObjectMapperInterface` 来使用对象映射器：

```php
// src/Controller/UserController.php
namespace App\Controller;

use App\Dto\UserInput;
use App\Entity\User;
use Symfony\Component\ObjectMapper\ObjectMapperInterface;

class UserController extends AbstractController
{
    public function updateUser(UserInput $userInput, ObjectMapperInterface $objectMapper): Response
    {
        $user = new User();
        // 将属性从 UserInput 映射到 User
        $objectMapper->map($userInput, $user);

        return new Response('User updated!');
    }
}
```

### 映射到新对象

```php
$productInput = new ProductInput();
$productInput->name = 'Wireless Mouse';
$productInput->sku = 'WM-1024';

$mapper = new ObjectMapper();
// 创建新 Product 实例并映射属性
$product = $mapper->map($productInput, Product::class);
```

### 映射到现有对象

```php
$product = $productRepository->find(1);
$updateInput = new ProductUpdateInput();
$updateInput->price = 99.99;

$mapper = new ObjectMapper();
// 更新现有对象
$mapper->map($updateInput, $product);
```

---

## 使用属性配置映射

### 定义默认目标类

在源类上应用 `#[Map]` 来定义其默认映射目标：

```php
// src/Dto/ProductInput.php
use App\Entity\Product;
use Symfony\Component\ObjectMapper\Attribute\Map;

#[Map(target: Product::class)]
class ProductInput
{
    public string $name = '';
    public string $sku = '';
}

// 现在可以不带第二个参数调用 map()
$product = $mapper->map($productInput); // 自动映射到 Product
```

### 配置属性映射

在属性上使用 `#[Map]` 属性：

- `target`：指定目标对象中属性的名称
- `source`：指定源对象中属性的名称
- `if`：定义映射属性的条件
- `transform`：在映射前对值应用转换

```php
// src/Dto/OrderInput.php
use App\Entity\Order;
use Symfony\Component\ObjectMapper\Attribute\Map;

#[Map(target: Order::class)]
class OrderInput
{
    // 将 'customerEmail' 从源映射到目标中的 'email'
    #[Map(target: 'email')]
    public string $customerEmail = '';

    // 完全不映射此属性
    #[Map(if: false)]
    public string $internalNotes = '';

    // 只有当折扣码不为空时才映射
    #[Map(if: 'strlen')]
    public ?string $discountCode = null;
}
```

### 使用服务进行条件映射

```php
// src/ObjectMapper/IsShippableCondition.php
use Symfony\Component\ObjectMapper\ConditionCallableInterface;

final class IsShippableCondition implements ConditionCallableInterface
{
    public function __invoke(mixed $value, object $source, ?object $target): bool
    {
        return $source->total > 50;
    }
}

// 在属性上使用
#[Map(if: IsShippableCondition::class)]
public ?string $shippingAddress = null;
```

### 基于目标类的条件属性映射

当源类映射到多个目标时，可以根据使用哪个目标来包含或排除某些属性：

```php
use Symfony\Component\ObjectMapper\Condition\TargetClass;

#[Map(target: PublicUserProfile::class)]
#[Map(target: AdminUserProfile::class)]
class User
{
    // 仅当目标是 AdminUserProfile 时才映射 'lastLoginIp' 到 'ipAddress'
    #[Map(target: 'ipAddress', if: new TargetClass(AdminUserProfile::class))]
    public ?string $lastLoginIp = '192.168.1.100';

    // 对两个目标都映射 'registrationDate' 到 'memberSince'
    #[Map(target: 'memberSince')]
    public \DateTimeImmutable $registrationDate;
}
```

---

## 转换值

### 使用可调用对象

```php
use Symfony\Component\ObjectMapper\Attribute\Map;

#[Map(target: Product::class)]
class ProductInput
{
    // 使用另一个类的静态方法进行格式化
    #[Map(target: 'displayPrice', transform: [PriceFormatter::class, 'format'])]
    public float $price = 0.0;

    // 也可以使用内置 PHP 函数
    #[Map(transform: 'intval')]
    public string $stockLevel = '100';
}
```

### 使用转换器服务

```php
final class FullNameTransformer implements TransformCallableInterface
{
    public function __invoke(mixed $value, object $source, ?object $target): mixed
    {
        return trim($source->firstName . ' ' . $source->lastName);
    }
}

// 在属性上使用
#[Map(target: 'fullName', transform: FullNameTransformer::class)]
public string $firstName = '';
```

---

## 映射集合

默认情况下，ObjectMapper 不映射数组或可遍历集合。使用 `MapCollection` 转换器：

```php
use Symfony\Component\ObjectMapper\Attribute\Map;
use Symfony\Component\ObjectMapper\Transform\MapCollection;

class ProductListInput
{
    #[Map(transform: new MapCollection())]
    /** @var ProductInput[] */
    public array $products;
}
```

---

## 映射多个目标

```php
#[Map(target: OnlineEvent::class, if: [self::class, 'isOnline'])]
#[Map(target: PhysicalEvent::class, if: [self::class, 'isPhysical'])]
class EventInput
{
    public string $type = 'online';

    public static function isOnline(?mixed $value, object $source): bool
    {
        return 'online' === $source->type;
    }

    public static function isPhysical(?mixed $value, object $source): bool
    {
        return 'physical' === $source->type;
    }
}

// 自动映射到 PhysicalEvent
$event = $mapper->map($eventInput);
```

> **注意**：没有条件时，ObjectMapper 无法确定使用哪个目标并会抛出"Ambiguous mapping"异常。

---

## 基于目标属性的映射（源映射）

使用目标类属性上的 `source` 参数来定义目标应如何从源获取值：

```php
// src/Api/Payload.php - 外部 API 数据（snake_case 属性）
class Payload
{
    public string $product_name = '';
    public float $price_amount = 0.0;
}

// src/Entity/Product.php - 应用内部实体（camelCase 属性）
#[Map(source: Payload::class)]
class Product
{
    #[Map(source: 'product_name')]
    public string $name = '';

    #[Map(source: 'price_amount')]
    public float $price = 0.0;
}
```

---

## 处理递归

ObjectMapper 自动检测并处理对象之间的递归关系，防止无限循环：

```php
$manager = new User();
$manager->name = 'Alice';
$employee = new User();
$employee->name = 'Bob';
$employee->manager = $manager;
$manager->manager = $employee; // 循环引用

$mapper = new ObjectMapper();
$employeeDto = $mapper->map($employee, UserDto::class);

// 正确处理循环：
// $employeeDto->name === 'Bob'
// $employeeDto->manager->name === 'Alice'
// $employeeDto->manager->manager === $employeeDto
```
