# 验证

验证是 Web 应用中非常常见的任务。表单中输入的数据需要经过验证，数据在写入数据库或传递给 Web 服务之前也需要进行验证。

Symfony 提供了一个 [Validator](https://github.com/symfony/validator) 组件来帮助你完成这项工作。该组件基于 [JSR303 Bean Validation 规范](https://jcp.org/en/jsr/detail?id=303)。

## 安装

在使用 Symfony Flex 的应用中，运行以下命令来安装验证器：

```bash
$ composer require symfony/validator
```

> **注意：** 如果你的应用没有使用 Symfony Flex，可能需要手动进行一些配置以启用验证功能。请参阅验证配置参考。

## 验证基础

理解验证最好的方式是实际操作。首先，假设你创建了一个普通的 PHP 对象，需要在应用中某处使用：

```php
// src/Entity/Author.php
namespace App\Entity;

class Author
{
    private string $name;
}
```

到目前为止，这是一个在应用中具有某种用途的普通类。验证的目标是告诉你对象的数据是否有效。为此，你需要配置一组规则（称为约束），对象必须遵守这些规则才算有效。这些规则通常使用 PHP 代码或属性来定义，但也可以定义为 `config/validator/` 目录下的 `.yaml` 或 `.xml` 文件。

例如，要指定 `$name` 属性不能为空，请添加以下内容：

```php
// src/Entity/Author.php
namespace App\Entity;

// ...
use Symfony\Component\Validator\Constraints as Assert;

class Author
{
    #[Assert\NotBlank]
    private string $name;
}
```

```yaml
# config/validator/validation.yaml
App\Entity\Author:
    properties:
        name:
            - NotBlank: ~
```

```xml
<!-- config/validator/validation.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<constraint-mapping xmlns="http://symfony.com/schema/dic/constraint-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/constraint-mapping
        https://symfony.com/schema/dic/constraint-mapping/constraint-mapping-1.0.xsd">

    <class name="App\Entity\Author">
        <property name="name">
            <constraint name="NotBlank"/>
        </property>
    </class>
</constraint-mapping>
```

```php
// src/Entity/Author.php
namespace App\Entity;
// ...
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Mapping\ClassMetadata;

class Author
{
    private string $name;

    public static function loadValidatorMetadata(ClassMetadata $metadata): void
    {
        $metadata->addPropertyConstraint('name', new Assert\NotBlank());
    }
}
```

单独添加此配置并不能保证值不为空；如果你愿意，仍然可以将其设置为空值。要真正保证值符合约束，必须将对象传递给验证器服务进行检查。

> **提示：** Symfony 的验证器使用 PHP 反射以及 *"getter"* 方法来获取任意属性的值，因此属性可以是 public、private 或 protected 的（参见约束目标）。

> **提示：** Symfony 为验证映射文件提供了 JSON schema，可以在 PhpStorm 等 IDE 中启用自动补全和验证功能。在 YAML 文件开头添加以下 `$schema` 键以启用此功能：
>
> ```yaml
> # config/validator/validation.yaml
> '$schema': https://symfony.com/schema/dic/constraint-mapping/constraint-mapping-1.0.json
> App\Entity\Author:
>     properties:
>         # 你的 IDE 现在将在这里提供自动补全...
> ```

### 使用验证器服务

接下来，要实际验证一个 `Author` 对象，请在 `validator` 服务（实现了 `ValidatorInterface`）上使用 `validate()` 方法。`validator` 的工作是读取类的约束（即规则），并验证对象上的数据是否满足这些约束。如果验证失败，将返回一个非空的错误列表（`ConstraintViolationList` 类）。下面是控制器中的一个简单示例：

```php
// ...
use App\Entity\Author;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Validator\Validator\ValidatorInterface;

// ...
public function author(ValidatorInterface $validator): Response
{
    $author = new Author();

    // ... do something to the $author object

    $errors = $validator->validate($author);

    if (count($errors) > 0) {
        /*
         * Uses a __toString method on the $errors variable which is a
         * ConstraintViolationList object. This gives us a nice string
         * for debugging.
         */
        $errorsString = (string) $errors;

        return new Response($errorsString);
    }

    return new Response('The author is valid! Yes!');
}
```

如果 `$name` 属性为空，你将看到以下错误消息：

```text
Object(App\Entity\Author).name:
    This value should not be blank.
```

如果你在 `name` 属性中插入一个值，则会显示成功消息。

> **提示：** 大多数情况下，你不会直接与 `validator` 服务交互，也不需要担心打印错误。大多数时候，你会在处理表单提交数据时间接使用验证。更多信息请参阅如何验证 Symfony 表单。

你也可以将错误集合传递给模板：

```php
if (count($errors) > 0) {
    return $this->render('author/validation.html.twig', [
        'errors' => $errors,
    ]);
}
```

在模板中，你可以按需输出错误列表：

```twig
{# templates/author/validation.html.twig #}
<h3>The author has the following errors</h3>
<ul>
{% for error in errors %}
    <li>{{ error.message }}</li>
{% endfor %}
</ul>
```

> **注意：** 每个验证错误（称为"约束违规"）由一个 `ConstraintViolation` 对象表示。该对象允许你通过 `ConstraintViolation::getConstraint()` 方法获取导致此违规的约束等信息。

### 验证可调用对象

`Validation` 还允许你创建一个闭包，用于根据一组约束验证值（例如在验证控制台命令答案或验证 OptionsResolver 值时非常有用）：

`Symfony\Component\Validator\Validation::createCallable`
: 返回一个闭包，当约束不匹配时抛出 `ValidationFailedException`。

`Symfony\Component\Validator\Validation::createIsValidCallable`
: 返回一个闭包，当约束不匹配时返回 `false`。

## 约束

`validator` 被设计为根据*约束*（即规则）来验证对象。要验证一个对象，只需将一个或多个约束映射到其类，然后将其传递给 `validator` 服务。

在内部，约束是一个做出断言声明的 PHP 对象。在现实中，约束可以是：`'蛋糕不能被烤焦'`。在 Symfony 中，约束类似：它们是条件为真的断言。给定一个值，约束将告诉你该值是否符合约束的规则。

### 支持的约束

Symfony 打包了许多最常用的约束。

你也可以创建自己的自定义约束，相关内容在自定义约束一文中有介绍。

### 约束配置

有些约束很简单，例如 `NotBlank`；而其他约束，例如 `Choice` 约束，有多个可用的配置选项。假设 `Author` 类有另一个属性 `genre`，定义与作者最相关的文学类型，可以设置为"fiction"或"non-fiction"：

```php
// src/Entity/Author.php
namespace App\Entity;

// ...
use Symfony\Component\Validator\Constraints as Assert;

class Author
{
    #[Assert\Choice(
        choices: ['fiction', 'non-fiction'],
        message: 'Choose a valid genre.',
    )]
    private string $genre;

    // ...
}
```

```yaml
# config/validator/validation.yaml
App\Entity\Author:
    properties:
        genre:
            - Choice: { choices: [fiction, non-fiction], message: Choose a valid genre. }
        # ...
```

```xml
<!-- config/validator/validation.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<constraint-mapping xmlns="http://symfony.com/schema/dic/constraint-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/constraint-mapping
        https://symfony.com/schema/dic/constraint-mapping/constraint-mapping-1.0.xsd">

    <class name="App\Entity\Author">
        <property name="genre">
            <constraint name="Choice">
                <option name="choices">
                    <value>fiction</value>
                    <value>non-fiction</value>
                </option>
                <option name="message">Choose a valid genre.</option>
            </constraint>
        </property>

        <!-- ... -->
    </class>
</constraint-mapping>
```

```php
// src/Entity/Author.php
namespace App\Entity;

// ...
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Mapping\ClassMetadata;

class Author
{
    private string $genre;

    // ...

    public static function loadValidatorMetadata(ClassMetadata $metadata): void
    {
        // ...

        $metadata->addPropertyConstraint('genre', new Assert\Choice(
            choices: ['fiction', 'non-fiction'],
            message: 'Choose a valid genre.',
        ));
    }
}
```

## 表单类中的约束

可以在构建表单时通过表单字段的 `constraints` 选项来定义约束：

```php
use Symfony\Component\Validator\Constraints as Assert;

public function buildForm(FormBuilderInterface $builder, array $options): void
{
    $builder
        ->add('myField', TextType::class, [
            'required' => true,
            'constraints' => [new Assert\Length(min: 3)],
        ])
    ;
}
```

## 约束目标

约束可以应用于类属性（例如 `name`）、getter 方法（例如 `getFullName()`）或整个类。属性约束是最常见且最易于使用的。Getter 约束允许你指定更复杂的验证规则。最后，类约束适用于希望将类作为整体进行验证的场景。

### 属性

验证类属性是最基本的验证技术。Symfony 允许你验证 private、protected 或 public 属性。以下示例展示了如何将 `Author` 类的 `$firstName` 属性配置为至少 3 个字符。

```php
// src/Entity/Author.php

// ...
use Symfony\Component\Validator\Constraints as Assert;

class Author
{
    #[Assert\NotBlank]
    #[Assert\Length(min: 3)]
    private string $firstName;
}
```

```yaml
# config/validator/validation.yaml
App\Entity\Author:
    properties:
        firstName:
            - NotBlank: ~
            - Length:
                min: 3
```

```xml
<!-- config/validator/validation.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<constraint-mapping xmlns="http://symfony.com/schema/dic/constraint-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/constraint-mapping
        https://symfony.com/schema/dic/constraint-mapping/constraint-mapping-1.0.xsd">

    <class name="App\Entity\Author">
        <property name="firstName">
            <constraint name="NotBlank"/>
            <constraint name="Length">
                <option name="min">3</option>
            </constraint>
        </property>
    </class>
</constraint-mapping>
```

```php
// src/Entity/Author.php
namespace App\Entity;

// ...
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Mapping\ClassMetadata;

class Author
{
    private string $firstName;

    public static function loadValidatorMetadata(ClassMetadata $metadata): void
    {
        $metadata->addPropertyConstraint('firstName', new Assert\NotBlank());
        $metadata->addPropertyConstraint(
            'firstName',
            new Assert\Length(min: 3)
        );
    }
}
```

> **警告：** 如果一个有类型的属性未初始化，验证器将使用 `null` 值。如果属性在初始化时持有某个值，这可能会导致意外行为。为避免这种情况，请确保在验证之前初始化所有属性。

### Getter

约束也可以应用于方法的返回值。Symfony 允许你为任何名称以"get"、"is"或"has"开头的 private、protected 或 public 方法添加约束。在本指南中，这类方法被称为"getter"。

这种技术的好处是它允许你动态地验证对象。例如，假设你想确保密码字段与用户的名字不匹配（出于安全原因）。你可以通过创建一个 `isPasswordSafe()` 方法来实现，然后断言该方法必须返回 `true`：

```php
// src/Entity/Author.php
namespace App\Entity;

// ...
use Symfony\Component\Validator\Constraints as Assert;

class Author
{
    #[Assert\IsTrue(message: 'The password cannot match your first name')]
    public function isPasswordSafe(): bool
    {
        // ... return true or false
    }
}
```

```yaml
# config/validator/validation.yaml
App\Entity\Author:
    getters:
        passwordSafe:
            - 'IsTrue': { message: 'The password cannot match your first name' }
```

```xml
<!-- config/validator/validation.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<constraint-mapping xmlns="http://symfony.com/schema/dic/constraint-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/constraint-mapping
        https://symfony.com/schema/dic/constraint-mapping/constraint-mapping-1.0.xsd">

    <class name="App\Entity\Author">
        <getter property="passwordSafe">
            <constraint name="IsTrue">
                <option name="message">The password cannot match your first name</option>
            </constraint>
        </getter>
    </class>
</constraint-mapping>
```

```php
// src/Entity/Author.php
namespace App\Entity;

// ...
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Mapping\ClassMetadata;

class Author
{
    public static function loadValidatorMetadata(ClassMetadata $metadata): void
    {
        $metadata->addGetterConstraint('passwordSafe', new Assert\IsTrue(
            message: 'The password cannot match your first name',
        ));
    }
}
```

现在，创建 `isPasswordSafe()` 方法并包含所需的逻辑：

```php
public function isPasswordSafe(): bool
{
    return $this->firstName !== $this->password;
}
```

> **注意：** 细心的读者会注意到，在 YAML、XML 和 PHP 格式的映射中，getter 的前缀（"get"、"is"或"has"）被省略了。这样做可以让你在以后将约束移动到同名属性上（反之亦然），而无需更改验证逻辑。

### 类

有些约束应用于被验证的整个类。例如，`Callback` 约束是应用于类本身的通用约束。当该类被验证时，该约束指定的方法会被简单地执行，每个方法都可以提供更多自定义验证。

## 验证具有继承关系的对象

当你验证一个继承自另一个类的对象时，验证器会自动验证父类中定义的约束。

**即使子属性覆盖了那些约束，父属性中定义的约束也会应用于子属性**。Symfony 始终会合并每个属性的父类约束。

你无法更改此行为，但可以通过在不同的验证分组中定义父类和子类约束，然后在验证每个对象时选择合适的分组来解决这个问题。

## 为类扩展验证

有时你可能想要在无法修改的类上添加或覆盖验证约束（例如，来自第三方库或 bundle 的模型）。

假设你使用了一个第三方 `Product` 类，该类以最小长度 2 来验证 `name` 属性，但在你的应用中你希望强制最少 10 个字符。

为此，创建一个单独的类，并使用 `#[ExtendsValidationFor]` 属性告知验证器哪个类应该接收这些约束。你的新类名无关紧要，通常将类设为 `abstract` 以表明它永远不会被实例化：

```php
use Symfony\Component\Validator\Attribute\ExtendsValidationFor;
use Symfony\Component\Validator\Constraints as Assert;

#[ExtendsValidationFor(Product::class)]
abstract class MyProductValidation
{
    #[Assert\NotBlank(groups: ['my_app'])]
    #[Assert\Length(min: 10, groups: ['my_app'])]
    public string $name = '';
}
```

此类中定义的约束将被应用于目标类（`Product`），就像它们在那里定义的一样。

你只能为目标类上存在的属性定义约束，否则将抛出 `MappingException`。

## 调试约束

使用 `debug:validator` 命令列出给定类的验证约束：

```bash
$ php bin/console debug:validator 'App\Entity\SomeClass'

    App\Entity\SomeClass
    -----------------------------------------------------

    +---------------+--------------------------------------------------+---------+------------------------------------------------------------+
    | Property      | Name                                             | Groups  | Options                                                    |
    +---------------+--------------------------------------------------+---------+------------------------------------------------------------+
    | firstArgument | Symfony\Component\Validator\Constraints\NotBlank | Default | [                                                          |
    |               |                                                  |         |   "message" => "This value should not be blank.",          |
    |               |                                                  |         |   "allowNull" => false,                                    |
    |               |                                                  |         |   "normalizer" => null,                                    |
    |               |                                                  |         |   "payload" => null                                        |
    |               |                                                  |         | ]                                                          |
    | firstArgument | Symfony\Component\Validator\Constraints\Email    | Default | [                                                          |
    |               |                                                  |         |   "message" => "This value is not a valid email address.", |
    |               |                                                  |         |   "mode" => null,                                          |
    |               |                                                  |         |   "normalizer" => null,                                    |
    |               |                                                  |         |   "payload" => null                                        |
    |               |                                                  |         | ]                                                          |
    +---------------+--------------------------------------------------+---------+------------------------------------------------------------+
```

你也可以验证存储在给定目录中的所有类：

```bash
$ php bin/console debug:validator src/Entity
```

## 总结

Symfony 的 `validator` 是一个强大的工具，可以用来保证任何对象的数据是"有效的"。验证的力量在于"约束"，即可以应用于对象属性或 getter 方法的规则。虽然你最常在使用表单时间接使用验证框架，但请记住，它可以在任何地方用于验证任何对象。

## 深入了解
