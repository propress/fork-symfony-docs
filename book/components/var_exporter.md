# VarExporter 组件

VarExporter 组件可以将任何可序列化的 PHP 数据结构导出为纯 PHP 代码，并允许你在不调用构造函数的情况下实例化和填充对象。

## 安装

```terminal
$ composer require --dev symfony/var-exporter
```

## 导出/序列化变量

此组件的主要功能是将 PHP 数据结构序列化为纯 PHP 代码，类似于 PHP 的 `var_export` 函数：

```php
use Symfony\Component\VarExporter\VarExporter;

$exported = VarExporter::export($someVariable);
// store the $exported data in some file or cache system for later reuse
$data = file_put_contents('exported.php', '<?php return '.$exported.';');

// later, regenerate the original variable when you need it
$regeneratedVariable = require 'exported.php';
```

使用此组件而非 `serialize()` 或 `igbinary` 的原因在于性能：得益于 [OPcache]，生成的代码比使用 `unserialize()` 或 `igbinary_unserialize()` 要快得多，内存效率也更高。

此外，还有一些细微差异：

* 如果原始变量定义了这些语义，与 `serialize()` 相关的所有语义（如 `__wakeup()`、`__sleep()` 和 `Serializable`）都会被保留（`var_export()` 会忽略它们）；
* 涉及 `SplObjectStorage`、`ArrayObject` 或 `ArrayIterator` 实例的引用会被保留；
* 缺失的类会抛出 `ClassNotFoundException`，而不是被反序列化为 `PHP_Incomplete_Class` 对象；
* `Reflection*`、`IteratorIterator` 和 `RecursiveIteratorIterator` 类在序列化时会抛出异常。

导出的数据是符合 [PSR-2] 规范的 PHP 文件。例如，考虑以下类层次结构：

```php
abstract class AbstractClass
{
    protected int $foo;
    private int $bar;

    protected function setBar($bar): void
    {
        $this->bar = $bar;
    }
}

class ConcreteClass extends AbstractClass
{
    public function __construct()
    {
        $this->foo = 123;
        $this->setBar(234);
    }
}
```

使用 VarExporter 导出 `ConcreteClass` 数据后，生成的 PHP 文件如下所示：

```php
return \Symfony\Component\VarExporter\Internal\Hydrator::hydrate(
    $o = [
        clone (\Symfony\Component\VarExporter\Internal\Registry::$prototypes['Symfony\\Component\\VarExporter\\Tests\\ConcreteClass'] ?? \Symfony\Component\VarExporter\Internal\Registry::p('Symfony\\Component\\VarExporter\\Tests\\ConcreteClass')),
    ],
    null,
    [
        'Symfony\\Component\\VarExporter\\Tests\\AbstractClass' => [
            'foo' => [
                123,
            ],
            'bar' => [
                234,
            ],
        ],
    ],
    $o[0],
    []
);
```

## 实例化和填充 PHP 类

### Instantiator

此组件提供了一个实例化器，可以创建对象并设置其属性，而无需调用构造函数或任何其他方法：

```php
use Symfony\Component\VarExporter\Instantiator;

// creates an empty instance of Foo
$fooObject = Instantiator::instantiate(Foo::class);

// creates a Foo instance and sets one of its properties
$fooObject = Instantiator::instantiate(Foo::class, ['propertyName' => $propertyValue]);
```

实例化器还可以填充父类的属性。假设 `Bar` 是 `Foo` 的父类，并定义了 `privateBarProperty` 属性：

```php
use Symfony\Component\VarExporter\Instantiator;

// creates a Foo instance and sets a private property defined on its parent Bar class
$fooObject = Instantiator::instantiate(Foo::class, [], [
    Bar::class => ['privateBarProperty' => $propertyValue],
]);
```

可以使用特殊属性名 `"\0"` 来定义 `ArrayObject`、`ArrayIterator` 和 `SplObjectHash` 实例的内部值：

```php
use Symfony\Component\VarExporter\Instantiator;

// creates an SplObjectStorage where $info1 is associated with $object1, etc.
$theObject = Instantiator::instantiate(SplObjectStorage::class, [
    "\0" => [$object1, $info1, $object2, $info2...],
]);

// creates an ArrayObject populated with $inputArray
$theObject = Instantiator::instantiate(ArrayObject::class, [
    "\0" => [$inputArray],
]);
```

### Hydrator

与实例化器（用于填充尚不存在的对象）不同，有时你需要填充已存在对象的属性。`Symfony\Component\VarExporter\Hydrator` 正是为此设计的。以下是填充对象属性的基本用法：

```php
use Symfony\Component\VarExporter\Hydrator;

$object = new Foo();
Hydrator::hydrate($object, ['propertyName' => $propertyValue]);
```

Hydrator 也可以填充父类的属性。假设 `Bar` 是 `Foo` 的父类，并定义了 `privateBarProperty` 属性：

```php
use Symfony\Component\VarExporter\Hydrator;

$object = new Foo();
Hydrator::hydrate($object, [], [
    Bar::class => ['privateBarProperty' => $propertyValue],
]);

// alternatively, you can use the special "\0" syntax
Hydrator::hydrate($object, ["\0Bar\0privateBarProperty" => $propertyValue]);
```

可以使用特殊属性名 `"\0"` 来填充 `ArrayObject`、`ArrayIterator` 和 `SplObjectHash` 实例的内部值：

```php
use Symfony\Component\VarExporter\Hydrator;

// creates an SplObjectHash where $info1 is associated with $object1, etc.
$storage = new SplObjectStorage();
Hydrator::hydrate($storage, [
    "\0" => [$object1, $info1, $object2, $info2...],
]);

// creates an ArrayObject populated with $inputArray
$arrayObject = new ArrayObject();
Hydrator::hydrate($arrayObject, [
    "\0" => [$inputArray],
]);
```

## 创建惰性对象

惰性对象是空实例化后按需填充的对象。当类的某些属性需要大量计算才能确定其值时，这特别有用。在这种情况下，你可能希望只有在实际访问该属性时才触发计算，从而在属性从未被使用时完全避免昂贵的处理。

自 PHP 8.4 起，PHP 通过反射 API 提供了对惰性对象的原生支持。该原生 API 适用于具体类，但不适用于抽象类或内置类。此组件提供了使用装饰器模式生成惰性对象的辅助方法，也适用于抽象类、内置类和接口：

```php
$proxyCode = ProxyHelper::generateLazyProxy(new \ReflectionClass(SomeInterface::class));
// $proxyCode should be dumped into a file in production environments
eval('class ProxyDecorator'.$proxyCode);

$proxy = ProxyDecorator::createLazyProxy(initializer: function (): SomeInterface {
    // use whatever heavy logic you need here
    // to compute the $dependencies of the proxied class
    $instance = new SomeHeavyClass(...$dependencies);
    // call setters, etc. if needed

    return $instance;
});
```

仅在无法使用原生惰性对象时才使用此机制（否则你会收到废弃通知）。

[OPcache]: https://www.php.net/opcache
[PSR-2]: https://www.php-fig.org/psr/psr-2/
