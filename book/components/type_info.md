# TypeInfo 组件

TypeInfo 组件从属性、参数和返回类型等 PHP 元素中提取类型信息。

该组件提供：

* 一个强大的 `Type` 定义，可以处理联合类型、交叉类型和泛型（未来可以扩展以支持更多类型）；
* 一种从属性、方法参数、返回类型和原始字符串等 PHP 元素获取类型的方法。

## 安装

```terminal
$ composer require symfony/type-info
```

## 使用

该组件提供了一个 `Symfony\Component\TypeInfo\Type` 对象，表示你构建或要求解析的任何内容的 PHP 类型。

使用此组件有两种方式。第一种是借助 `Symfony\Component\TypeInfo\Type` 的静态方法手动创建类型：

```php
use Symfony\Component\TypeInfo\Type;

Type::int();
Type::nullable(Type::string());
Type::generic(Type::object(Collection::class), Type::int());
Type::list(Type::bool());
Type::intersection(Type::object(\Stringable::class), Type::object(\Iterator::class));
```

更多方法可以在 `Symfony\Component\TypeInfo\TypeFactoryTrait` 中找到。

你也可以使用通用方法自动检测类型：

```php
Type::fromValue(1.1);   // same as Type::float()
Type::fromValue('...'); // same as Type::string()
Type::fromValue(false); // same as Type::false()
```

### 解析器

使用该组件的第二种方式是使用 `TypeInfo` 基于反射或简单字符串解析类型。这种方法专为需要简单方式描述类或任何具有类型内容的库而设计：

```php
use Symfony\Component\TypeInfo\Type;
use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;

class Dummy
{
    public function __construct(
        public int $id,
    ) {
    }
}

// Instantiate a new resolver
$typeResolver = TypeResolver::create();

// Then resolve types for any subject
$typeResolver->resolve(new \ReflectionProperty(Dummy::class, 'id')); // returns an "int" Type instance
$typeResolver->resolve('bool'); // returns a "bool" Type instance
$typeResolver->resolve('array{id: int, name?: string}'); // returns an array shape type instance where 'id' is required and 'name' is optional


// Types can be instantiated thanks to static factories
$type = Type::list(Type::nullable(Type::bool()));

// Type instances have several helper methods

// for collections, it returns the type of the item used as the key;
// in this example, the collection is a list, so it returns an "int" Type instance
$keyType = $type->getCollectionKeyType();

// you can chain the utility methods (e.g. to introspect the values of the collection)
// the following code will return true
$isValueNullable = $type->getCollectionValueType()->isNullable();
```

这些调用中的每一个都将返回对应于所用静态方法的 `Type` 实例。你也可以从字符串解析类型（如上面示例中的 `bool` 参数所示）。

### PHPDoc 解析

在许多情况下，你可能没有清晰类型化的属性，或者需要由高级 PHPDoc 提供的更精确的类型定义。为此，你可以使用基于 PHPDoc 注解的字符串解析器。

首先，运行命令 `composer require phpstan/phpdoc-parser` 安装字符串解析所需的 PHP 包。然后，按照以下步骤操作：

```php
use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;

class Dummy
{
    public function __construct(
        public int $id,
        /** @var string[] $tags */
        public array $tags,
    ) {
    }
}

$typeResolver = TypeResolver::create();
$typeResolver->resolve(new \ReflectionProperty(Dummy::class, 'id')); // returns an "int" Type
$typeResolver->resolve(new \ReflectionProperty(Dummy::class, 'tags')); // returns a collection with "int" as key and "string" as values Type
```

### 类型别名

TypeInfo 组件支持通过 PHPDoc 注解定义的类型别名。这允许你一次定义复杂类型并在代码库中复用它们：

```php
/**
 * @phpstan-type UserData = array{name: string, email: string, age: int}
 */
class UserService
{
    /**
     * @var UserData
     */
    public mixed $userData;

    /**
     * @param UserData $data
     */
    public function process(mixed $data): void
    {
        // ...
    }
}

$typeResolver = TypeResolver::create();
$typeResolver->resolve(new \ReflectionProperty(UserService::class, 'userData'));
// returns an array Type with the shape defined in UserData
```

该组件支持 PHPStan 和 Psalm 注解格式：

* `@phpstan-type` 和 `@psalm-type` 用于定义类型别名
* `@phpstan-import-type` 和 `@psalm-import-type` 用于从其他类导入类型别名

你也可以导入在其他类中定义的类型别名：

```php
/**
 * @phpstan-type Address = array{street: string, city: string, zip: string}
 */
class Location
{
}

/**
 * @phpstan-import-type Address from Location
 */
class Company
{
    /**
     * @var Address
     */
    public mixed $headquarters;
}
```

> **注意：**
> 两种语法变体都受支持：带等号（`@phpstan-type TypeAlias = Type`）或不带等号（`@phpstan-type TypeAlias Type`）。

你也可以通过框架配置全局定义类型别名。这些别名在类型解析器中处处可用，无需 `@phpstan-type` 注解：

```yaml
# config/packages/framework.yaml
framework:
    type_info:
        aliases:
            MoneyAmount: int
            UserData: 'array{name: string, email: string, age: int}'
```

```php
// config/packages/cache.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'type_info' => [
            'aliases' => [
                'MoneyAmount' => 'int',
                'UserData' => 'array{name: string, email: string, age: int}',
            ],
        ],
    ],
]);
```

配置完成后，这些别名可以在 PHPDoc 注解中使用，并由类型解析器解析：

```php
class Product
{
    /** @var MoneyAmount */
    public mixed $price;
}

$typeResolver = TypeResolver::create();
$typeResolver->resolve(new \ReflectionProperty(Product::class, 'price'));
// returns an "int" Type instance

// aliases can also be resolved directly from strings
$typeResolver->resolve('MoneyAmount'); // returns an "int" Type instance
```

> **注意：**
> 当 PHPDoc `@phpstan-type` 注解定义的别名与配置别名同名时，PHPDoc 注解优先。

### 数组形状

TypeInfo 可以解析数组形状，它描述具有特定键值类型关系的数组结构。在 PHPDoc 注解中使用 `array{...}` 语法：

```php
use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;

class Dummy
{
    /**
     * @var array{name: string, age: int, email?: string}
     */
    public array $person;
}

$typeResolver = TypeResolver::create();
$type = $typeResolver->resolve(new \ReflectionProperty(Dummy::class, 'person'));
// returns an ArrayShapeType with "name" (string), "age" (int), and optional "email" (string)
```

`?` 后缀将键标记为可选（例如 `email?`）。

数组形状默认是**封闭**的，意味着它们拒绝超出明确定义的额外条目。使用 `...` 创建接受额外条目的**非封闭**形状：

```
// sealed: only accepts "id" key
// @var array{id: int}

// unsealed: accepts "id" and any extra entries
// @var array{id: int, ...}

// unsealed but extra entries must use strings as keys and booleans as values
// @var array{id: int, ...<string, bool>}
```

你也可以使用 `Type::arrayShape()` 方法手动创建数组形状：

```php
use Symfony\Component\TypeInfo\Type;

// simple array shape (sealed by default)
$type = Type::arrayShape([
    'name' => Type::string(),
    'age' => Type::int()
]);

// with optional keys (denoted by "?" suffix)
$type = Type::arrayShape([
    'required_id' => Type::int(),
    'optional_name' => ['type' => Type::string(), 'optional' => true],
]);

// unsealed: allow extra entries (sealed = false)
$type = Type::arrayShape([
    'id' => Type::int(),
], false);

// unsealed with typed extra keys and values (extraKeyType=string, extraValueType=bool)
// equivalent to: array{id: int, ...<string, bool>}
$type = Type::arrayShape([
    'id' => Type::int(),
], false, Type::string(), Type::bool());
```

### 高级用法

TypeInfo 组件提供了多种方法来根据你的需要操作和检查类型。

**识别**类型：

```php
// define a simple integer type
$type = Type::int();
// check if the type matches a specific identifier
$type->isIdentifiedBy(TypeIdentifier::INT);    // true
$type->isIdentifiedBy(TypeIdentifier::STRING); // false

// define a union type (equivalent to PHP's int|string)
$type = Type::union(Type::string(), Type::int());
// now the second check is true because the union type contains the string type
$type->isIdentifiedBy(TypeIdentifier::INT);    // true
$type->isIdentifiedBy(TypeIdentifier::STRING); // true

class DummyParent {}
class Dummy extends DummyParent implements DummyInterface {}

// define an object type
$type = Type::object(Dummy::class);

// check if the type is an object or matches a specific class
$type->isIdentifiedBy(TypeIdentifier::OBJECT); // true
$type->isIdentifiedBy(Dummy::class);           // true
// check if it inherits/implements something
$type->isIdentifiedBy(DummyParent::class);     // true
$type->isIdentifiedBy(DummyInterface::class);  // true
```

检查类型是否**接受某个值**：

```php
$type = Type::int();
// check if the type accepts a given value
$type->accepts(123); // true
$type->accepts('z'); // false

$type = Type::union(Type::string(), Type::int());
// now the second check is true because the union type accepts either an int or a string value
$type->accepts(123); // true
$type->accepts('z'); // true
```

使用可调用对象进行**复杂检查**：

```php
class Foo
{
    private int $integer;
    private string $string;
    private ?float $float;
}

$reflClass = new \ReflectionClass(Foo::class);

$resolver = TypeResolver::create();
$integerType = $resolver->resolve($reflClass->getProperty('integer'));
$stringType = $resolver->resolve($reflClass->getProperty('string'));
$floatType = $resolver->resolve($reflClass->getProperty('float'));

// define a callable to validate non-nullable number types
$isNonNullableNumber = function (Type $type): bool {
    if ($type->isNullable()) {
        return false;
    }

    if ($type->isIdentifiedBy(TypeIdentifier::INT) || $type->isIdentifiedBy(TypeIdentifier::FLOAT)) {
        return true;
    }

    return false;
};

$integerType->isSatisfiedBy($isNonNullableNumber); // true
$stringType->isSatisfiedBy($isNonNullableNumber);  // false
$floatType->isSatisfiedBy($isNonNullableNumber);   // false
```
