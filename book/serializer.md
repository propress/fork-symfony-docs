# 如何使用序列化器

Symfony 提供了一个序列化器，用于在不同格式的数据结构与 PHP 对象之间进行相互转换。

这最常见于构建 API 或与第三方 API 进行通信时。序列化器可以将传入的 JSON 请求负载转换为应用程序所使用的 PHP 对象。然后，在生成响应时，可以使用序列化器将 PHP 对象转换回 JSON 响应。

它还可用于，例如，将 CSV 配置数据作为 PHP 对象加载，甚至在格式之间进行转换（例如 YAML 转 XML）。

## 安装

在使用 [Symfony Flex](https://symfony.com/doc/current/setup.html#symfony-flex) 的应用程序中，运行以下命令安装序列化器 [Symfony pack](https://symfony.com/doc/current/setup.html#symfony-packs)：

```bash
$ composer require symfony/serializer-pack
```

> **注意：** 序列化器 pack 还会安装序列化器组件的一些常用可选依赖项。在 Symfony 框架之外使用该组件时，你可能希望从 `symfony/serializer` 包开始，并在需要时安装可选依赖项。

> **另请参阅：** Symfony 序列化器组件的一个流行替代方案是第三方库 [JMS serializer](https://github.com/schmittjoh/serializer)。

## 序列化对象

对于此示例，假设项目中存在以下类：

```php
// src/Model/Person.php
namespace App\Model;

class Person
{
    public function __construct(
        private int $age,
        private string $name,
        private bool $sportsperson
    ) {
    }

    public function getAge(): int
    {
        return $this->age;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function isSportsperson(): bool
    {
        return $this->sportsperson;
    }
}
```

如果你想将此类型的对象转换为 JSON 结构（例如通过 API 响应发送），可以使用 `Symfony\Component\Serializer\SerializerInterface` 参数类型获取 `serializer` 服务：

**Symfony 框架方式：**

```php
// src/Controller/PersonController.php
namespace App\Controller;

use App\Model\Person;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Serializer\SerializerInterface;

class PersonController extends AbstractController
{
    public function index(SerializerInterface $serializer): Response
    {
        $person = new Person('Jane Doe', 39, false);

        $jsonContent = $serializer->serialize($person, 'json');
        // $jsonContent contains {"name":"Jane Doe","age":39,"sportsperson":false}

        return JsonResponse::fromJsonString($jsonContent);
    }
}
```

**独立使用方式：**

```php
use App\Model\Person;
use Symfony\Component\Serializer\Encoder\JsonEncoder;
use Symfony\Component\Serializer\Normalizer\ObjectNormalizer;
use Symfony\Component\Serializer\Serializer;

$encoders = [new JsonEncoder()];
$normalizers = [new ObjectNormalizer()];
$serializer = new Serializer($normalizers, $encoders);

$person = new Person('Jane Done', 39, false);

$jsonContent = $serializer->serialize($person, 'json');
// $jsonContent contains {"name":"Jane Doe","age":39,"sportsperson":false}
```

`Symfony\Component\Serializer\Serializer::serialize` 的第一个参数是要序列化的对象，第二个参数用于选择合适的编码器（即格式），在本例中为 `Symfony\Component\Serializer\Encoder\JsonEncoder`。

> **提示：** 当你的控制器类继承 `AbstractController`（如上面的示例），可以使用 `Symfony\Bundle\FrameworkBundle\Controller\AbstractController::json` 方法，通过序列化器从对象创建 JSON 响应，从而简化控制器：
>
> ```php
> class PersonController extends AbstractController
> {
>     public function index(): Response
>     {
>         $person = new Person('Jane Doe', 39, false);
>
>         // when the Serializer is not available, this will use json_encode()
>         return $this->json($person);
>     }
> }
> ```

### 在 Twig 模板中使用序列化器

你也可以在任何 Twig 模板中使用 `serialize` 过滤器来序列化对象：

```twig
{{ person|serialize(format = 'json') }}
```

更多信息请参阅 [Twig 参考文档](https://symfony.com/doc/current/reference/twig_reference.html#serialize)。

## 反序列化对象

API 通常也需要将格式化的请求体（例如 JSON）转换为 PHP 对象。这个过程称为*反序列化*（也称为"水化"）：

**Symfony 框架方式：**

```php
// src/Controller/PersonController.php
namespace App\Controller;

// ...
use Symfony\Component\HttpFoundation\Exception\BadRequestException;
use Symfony\Component\HttpFoundation\Request;

class PersonController extends AbstractController
{
    // ...

    public function create(Request $request, SerializerInterface $serializer): Response
    {
        if ('json' !== $request->getContentTypeFormat()) {
            throw new BadRequestException('Unsupported content format');
        }

        $jsonData = $request->getContent();
        $person = $serializer->deserialize($jsonData, Person::class, 'json');

        // ... do something with $person and return a response
    }
}
```

**独立使用方式：**

```php
use App\Model\Person;
use Symfony\Component\Serializer\Encoder\JsonEncoder;
use Symfony\Component\Serializer\Normalizer\ObjectNormalizer;
use Symfony\Component\Serializer\Serializer;

// ...
$jsonData = ...; // fetch JSON from the request
$person = $serializer->deserialize($jsonData, Person::class, 'json');
```

在本例中，`Symfony\Component\Serializer\Serializer::deserialize` 需要三个参数：

1. 要解码的数据
2. 将该信息解码到的类名
3. 用于将数据转换为数组的编码器名称（即输入格式）

向该控制器发送请求时（例如 `{"first_name":"John Doe","age":54,"sportsperson":true}`），序列化器会创建一个新的 `Person` 实例，并将属性设置为给定 JSON 中的值。

> **注意：** 默认情况下，序列化器组件会忽略未映射到反规范化对象的额外属性。例如，如果向上述控制器的请求中包含 `{..., "city": "Paris"}`，则 `city` 字段将被忽略。你也可以在这些情况下使用稍后将介绍的[序列化器上下文](#序列化器上下文)抛出异常。

> **另请参阅：** 你也可以将数据反序列化到现有对象实例中（例如更新数据时）。请参阅[在现有对象中反序列化](#在现有对象中反序列化)。

## 序列化过程：规范化器与编码器

序列化器在（反）序列化对象时使用两步过程：

<object data="_images/serializer/serializer_workflow.svg" type="image/svg+xml"
    alt="A flow diagram showing how objects are serialized/deserialized. This is described in the subsequent paragraph."
></object>

在两个方向上，数据始终首先被转换为数组。这将过程分成两个独立的职责：

**规范化器（Normalizers）**
这些类将**对象**转换为**数组**，反之亦然。它们承担着繁重的工作：确定要序列化哪些类属性、它们持有什么值以及应使用什么名称。

**编码器（Encoders）**
编码器将**数组**转换为特定的**格式**，反之亦然。每个编码器都确切地知道如何解析和生成特定格式，例如 JSON 或 XML。

在内部，`Serializer` 类在（反）序列化对象时使用规范化器的有序列表和特定格式的一个编码器。

默认的 `serializer` 服务中配置了几个规范化器。最重要的规范化器是 `Symfony\Component\Serializer\Normalizer\ObjectNormalizer`。该规范化器使用反射和 [PropertyAccess 组件](https://symfony.com/doc/current/components/property_access.html) 在任何对象和数组之间进行转换。稍后你将了解更多关于[此规范化器及其他规范化器](#序列化器规范化器)的内容。

默认序列化器还配置了一些编码器，涵盖 HTTP 应用程序常用的格式：

* `Symfony\Component\Serializer\Encoder\JsonEncoder`
* `Symfony\Component\Serializer\Encoder\XmlEncoder`
* `Symfony\Component\Serializer\Encoder\CsvEncoder`
* `Symfony\Component\Serializer\Encoder\YamlEncoder`

更多关于这些编码器及其配置的信息，请参阅 [/serializer/encoders](https://symfony.com/doc/current/serializer/encoders.html)。

> **提示：** [API Platform](https://api-platform.com) 项目提供了更高级格式的编码器：
>
> * [JSON-LD](https://json-ld.org) 以及 [Hydra Core Vocabulary](https://www.hydra-cg.com/)
> * [OpenAPI](https://www.openapis.org) v2（前身为 Swagger）和 v3
> * [GraphQL](https://graphql.org)
> * [JSON:API](https://jsonapi.org)
> * [HAL](https://stateless.group/hal_specification.html)

### 序列化器上下文

序列化器及其规范化器和编码器通过*序列化器上下文*进行配置。此上下文可以在多个位置进行配置：

* [通过框架配置全局设置](#配置默认上下文)
* [在序列化/反序列化时设置](#在序列化反序列化时传递上下文)
* [针对特定属性设置](#在特定属性上配置上下文)

你可以同时使用这三个选项。当同一设置在多个位置配置时，列表中靠后的位置会覆盖前面的位置（例如，特定属性上的设置会覆盖全局配置的设置）。

#### 配置默认上下文

你可以在框架配置中配置默认上下文，例如在反序列化时禁止额外字段：

**YAML 配置：**

```yaml
# config/packages/serializer.yaml
framework:
    serializer:
        default_context:
            allow_extra_attributes: false
```

**PHP 配置：**

```php
// config/packages/serializer.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'serializer' => [
            'default_context' => [
                'allow_extra_attributes' => false,
            ],
        ],
    ],
]);
```

**独立使用方式：**

```php
use Symfony\Component\Serializer\NameConverter\CamelCaseToSnakeCaseNameConverter;
use Symfony\Component\Serializer\Normalizer\ObjectNormalizer;

// ...
$normalizers = [
    new ObjectNormalizer(null, null, null, null, null, null, [
        'allow_extra_attributes' => false,
    ]),
];
$serializer = new Serializer($normalizers, $encoders);
```

#### 在序列化/反序列化时传递上下文

你也可以为单次 `serialize()`/`deserialize()` 调用配置上下文。例如，只对一次序列化调用跳过具有 `null` 值的属性：

```php
use Symfony\Component\Serializer\Normalizer\AbstractObjectNormalizer;

// ...
$serializer->serialize($person, 'json', [
    AbstractObjectNormalizer::SKIP_NULL_VALUES => true
]);

// next calls to serialize() will NOT skip null values
```

##### 使用上下文构建器

你可以使用"上下文构建器"来帮助定义（反）序列化上下文。上下文构建器是 PHP 对象，提供上下文选项的自动补全、验证和文档：

```php
use Symfony\Component\Serializer\Context\Normalizer\DateTimeNormalizerContextBuilder;

$contextBuilder = new DateTimeNormalizerContextBuilder()
    ->withFormat('Y-m-d H:i:s');
$serializer->serialize($something, 'json', $contextBuilder->toArray());
```

每个规范化器/编码器都有其相关的上下文构建器。要创建更复杂的（反）序列化上下文，可以使用 `withContext()` 方法将它们链接起来：

```php
use Symfony\Component\Serializer\Context\Encoder\CsvEncoderContextBuilder;
use Symfony\Component\Serializer\Context\Normalizer\ObjectNormalizerContextBuilder;

$initialContext = [
    'custom_key' => 'custom_value',
];

$contextBuilder = new ObjectNormalizerContextBuilder()
    ->withContext($initialContext)
    ->withGroups(['group1', 'group2']);

$contextBuilder = new CsvEncoderContextBuilder()
    ->withContext($contextBuilder)
    ->withDelimiter(';');

$serializer->serialize($something, 'csv', $contextBuilder->toArray());
```

> **另请参阅：** 你还可以[创建自己的上下文构建器](https://symfony.com/doc/current/serializer/custom_context_builders.html)，为自定义上下文值提供自动补全、验证和文档。

#### 在特定属性上配置上下文

最后，你还可以在特定对象属性上配置上下文值。例如，配置日期时间格式：

**PHP 属性方式：**

```php
// src/Model/Person.php

// ...
use Symfony\Component\Serializer\Attribute\Context;
use Symfony\Component\Serializer\Normalizer\DateTimeNormalizer;

class Person
{
    #[Context([DateTimeNormalizer::FORMAT_KEY => 'Y-m-d'])]
    public \DateTimeImmutable $createdAt;

    // ...
}
```

**YAML 配置：**

```yaml
# config/serializer/person.yaml
App\Model\Person:
    attributes:
        createdAt:
            contexts:
                - context: { datetime_format: 'Y-m-d' }
```

**XML 配置：**

```xml
<!-- config/serializer/person.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<serializer xmlns="http://symfony.com/schema/dic/serializer-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/serializer-mapping
        https://symfony.com/schema/dic/serializer-mapping/serializer-mapping-1.0.xsd"
>
    <class name="App\Model\Person">
        <attribute name="createdAt">
            <context>
                <entry name="datetime_format">Y-m-d</entry>
            </context>
        </attribute>
    </class>
</serializer>
```

> **注意：** 使用 YAML 或 XML 时，映射文件必须放在以下位置之一：
>
> * `config/serializer/` 目录中的所有 `*.yaml` 和 `*.xml` 文件。
> * bundle 的 `Resources/config/` 目录中的 `serialization.yaml` 或 `serialization.xml` 文件；
> * bundle 的 `Resources/config/serialization/` 目录中的所有 `*.yaml` 和 `*.xml` 文件。

> **提示：** Symfony 为序列化器映射文件提供了 JSON Schema，可以在 PhpStorm 等 IDE 中启用自动补全和验证。在 YAML 文件的开头添加以下 `$schema` 键来启用此功能：
>
> ```yaml
> # config/serializer/person.yaml
> '$schema': https://symfony.com/schema/dic/serializer-mapping/serializer-mapping-1.0.json
> App\Model\Person:
>     attributes:
>         # your IDE will now provide autocompletion here...
> ```

你还可以指定特定于规范化或反规范化的上下文：

**PHP 属性方式：**

```php
// src/Model/Person.php

// ...
use Symfony\Component\Serializer\Attribute\Context;
use Symfony\Component\Serializer\Normalizer\DateTimeNormalizer;

class Person
{
    #[Context(
        normalizationContext: [DateTimeNormalizer::FORMAT_KEY => 'Y-m-d'],
        denormalizationContext: [DateTimeNormalizer::FORMAT_KEY => \DateTime::RFC3339],
    )]
    public \DateTimeImmutable $createdAt;

    // ...
}
```

**YAML 配置：**

```yaml
# config/serializer/person.yaml
App\Model\Person:
    attributes:
        createdAt:
            contexts:
                - normalization_context: { datetime_format: 'Y-m-d' }
                  denormalization_context: { datetime_format: !php/const \DateTime::RFC3339 }
```

**XML 配置：**

```xml
<!-- config/serializer/person.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<serializer xmlns="http://symfony.com/schema/dic/serializer-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/serializer-mapping
        https://symfony.com/schema/dic/serializer-mapping/serializer-mapping-1.0.xsd"
>
    <class name="App\Model\Person">
        <attribute name="createdAt">
            <normalization-context>
                <entry name="datetime_format">Y-m-d</entry>
            </normalization-context>

            <denormalization-context>
                <entry name="datetime_format">Y-m-d\TH:i:sP</entry>
            </denormalization-context>
        </attribute>
    </class>
</serializer>
```

你还可以将上下文的使用限制到某些[分组](#选择特定属性)：

**PHP 属性方式：**

```php
// src/Model/Person.php

// ...
use Symfony\Component\Serializer\Attribute\Context;
use Symfony\Component\Serializer\Attribute\Groups;
use Symfony\Component\Serializer\Normalizer\DateTimeNormalizer;

class Person
{
    #[Groups(['extended'])]
    #[Context([DateTimeNormalizer::FORMAT_KEY => \DateTime::RFC3339])]
    #[Context(
        context: [DateTimeNormalizer::FORMAT_KEY => \DateTime::RFC3339_EXTENDED],
        groups: ['extended'],
    )]
    public \DateTimeImmutable $createdAt;

    // ...
}
```

**YAML 配置：**

```yaml
# config/serializer/person.yaml
App\Model\Person:
    attributes:
        createdAt:
            groups: [extended]
            contexts:
                - context: { datetime_format: !php/const \DateTime::RFC3339 }
                - context: { datetime_format: !php/const \DateTime::RFC3339_EXTENDED }
                  groups: [extended]
```

**XML 配置：**

```xml
<!-- config/serializer/person.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<serializer xmlns="http://symfony.com/schema/dic/serializer-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/serializer-mapping
        https://symfony.com/schema/dic/serializer-mapping/serializer-mapping-1.0.xsd"
>
    <class name="App\Model\Person">
        <attribute name="createdAt">
            <group>extended</group>

            <context>
                <entry name="datetime_format">Y-m-d\TH:i:sP</entry>
            </context>
            <context>
                <entry name="datetime_format">Y-m-d\TH:i:s.vP</entry>
                <group>extended</group>
            </context>
        </attribute>
    </class>
</serializer>
```

该属性可以在单个属性上根据需要重复使用。没有分组的上下文始终首先应用。然后按提供的顺序合并匹配分组的上下文。

如果你在多个属性中重复相同的上下文，考虑在类上使用 `#[Context]` 属性，将该上下文配置应用于类的所有属性：

```php
namespace App\Model;

use Symfony\Component\Serializer\Attribute\Context;
use Symfony\Component\Serializer\Normalizer\DateTimeNormalizer;

#[Context([DateTimeNormalizer::FORMAT_KEY => \DateTime::RFC3339])]
#[Context(
    context: [DateTimeNormalizer::FORMAT_KEY => \DateTime::RFC3339_EXTENDED],
    groups: ['extended'],
)]
class Person
{
    // ...
}
```

## 使用流式序列化 JSON

Symfony 可以将 PHP 数据结构编码为 JSON 流，并将 JSON 流解码回 PHP 数据结构。

为此，它依赖于 [JsonStreamer 组件](https://symfony.com/doc/current/serializer/streaming_json.html)，该组件专为高效率设计，可以增量处理大型 JSON 数据，无需将整个内容加载到内存中。

在序列化器组件和 JsonStreamer 组件之间做选择时，请考虑以下几点：

* **序列化器组件**：最适合需要灵活性的场景，例如使用规范化器和反规范化器动态操作对象结构，或处理具有多种序列化格式的复杂对象。它还支持 JSON 以外的输出格式（包括你自己的自定义格式）。
* **JsonStreamer 组件**：最适合简单对象以及需要高性能和低内存使用的场景。它特别适合处理非常大的 JSON 数据集，或在不将整个数据集加载到内存的情况下实时流式传输 JSON。

选择取决于你的具体用例。JsonStreamer 组件专为性能和内存效率而设计，而序列化器组件提供更大的灵活性和更广泛的格式支持。

更多信息请参阅[流式 JSON](https://symfony.com/doc/current/serializer/streaming_json.html)。

## 序列化到 PHP 数组或从 PHP 数组反序列化

默认的 `Symfony\Component\Serializer\Serializer` 也可以通过使用相应的接口，仅执行[两步序列化过程](#序列化过程规范化器与编码器)中的一步：

**Symfony 框架方式：**

```php
use Symfony\Component\Serializer\Encoder\DecoderInterface;
use Symfony\Component\Serializer\Encoder\EncoderInterface;
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;
// ...

class PersonController extends AbstractController
{
    public function index(DenormalizerInterface&NormalizerInterface $serializer): Response
    {
        $person = new Person('Jane Doe', 39, false);

        // use normalize() to convert a PHP object to an array
        $personArray = $serializer->normalize($person, 'json');

        // ...and denormalize() to convert an array back to a PHP object
        $personCopy = $serializer->denormalize($personArray, Person::class);

        // ...
    }

    public function json(DecoderInterface&EncoderInterface $serializer): Response
    {
        $data = ['name' => 'Jane Doe'];

        // use encode() to transform PHP arrays into another format
        $json = $serializer->encode($data, 'json');

        // ...and decode() to transform any format to just PHP arrays (instead of objects)
        $data = $serializer->decode('{"name":"Charlie Doe"}', 'json');
        // $data contains ['name' => 'Charlie Doe']
    }
}
```

**独立使用方式：**

```php
use App\Model\Person;
use Symfony\Component\Serializer\Encoder\JsonEncoder;
use Symfony\Component\Serializer\Normalizer\ObjectNormalizer;
use Symfony\Component\Serializer\Serializer;

$encoders = [new JsonEncoder()];
$normalizers = [new ObjectNormalizer()];
$serializer = new Serializer($normalizers, $encoders);

// use normalize() to convert a PHP object to an array
$personArray = $serializer->normalize($person, 'json');

// ...and denormalize() to convert an array back to a PHP object
$personCopy = $serializer->denormalize($personArray, Person::class);

$data = ['name' => 'Jane Doe'];

// use encode() to transform PHP arrays into another format
$json = $serializer->encode($data, 'json');

// ...and decode() to transform any format to just PHP arrays (instead of objects)
$data = $serializer->decode('{"name":"Charlie Doe"}', 'json');
// $data contains ['name' => 'Charlie Doe']
```

## 忽略属性

`ObjectNormalizer` 规范化对象的*所有*属性以及所有以 `get*()`、`has*()`、`is*()` 和 `can*()` 开头的方法。某些属性或方法不应被序列化，可以使用 `#[Ignore]` 属性将其排除：

**PHP 属性方式：**

```php
// src/Model/Person.php
namespace App\Model;

use Symfony\Component\Serializer\Attribute\Ignore;

class Person
{
    // ...

    #[Ignore]
    public function isPotentiallySpamUser(): bool
    {
        // ...
    }
}
```

**YAML 配置：**

```yaml
App\Model\Person:
    attributes:
        potentiallySpamUser:
            ignore: true
```

**XML 配置：**

```xml
<?xml version="1.0" ?>
<serializer xmlns="http://symfony.com/schema/dic/serializer-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/serializer-mapping
        https://symfony.com/schema/dic/serializer-mapping/serializer-mapping-1.0.xsd"
>
    <class name="App\Model\Person">
        <attribute name="potentiallySpamUser" ignore="true"/>
    </class>
</serializer>
```

`potentiallySpamUser` 属性现在将永远不会被序列化：

**Symfony 框架方式：**

```php
use App\Model\Person;

// ...
$person = new Person('Jane Doe', 32, false);
$json = $serializer->serialize($person, 'json');
// $json contains {"name":"Jane Doe","age":32,"sportsperson":false}

$person1 = $serializer->deserialize(
    '{"name":"Jane Doe","age":32,"sportsperson":false","potentiallySpamUser":false}',
    Person::class,
    'json'
);
// the "potentiallySpamUser" value is ignored
```

**独立使用方式：**

```php
use App\Model\Person;
use Symfony\Component\Serializer\Mapping\Factory\ClassMetadataFactory;
use Symfony\Component\Serializer\Mapping\Loader\AttributeLoader;
use Symfony\Component\Serializer\Normalizer\ObjectNormalizer;
use Symfony\Component\Serializer\Serializer;

// ...

// you need to pass a class metadata factory with a loader to the
// ObjectNormalizer when reading mapping information like Ignore or Groups.
// E.g. when using PHP attributes:
$classMetadataFactory = new ClassMetadataFactory(new AttributeLoader());
$normalizers = [new ObjectNormalizer($classMetadataFactory)];

$serializer = new Serializer($normalizers, $encoders);

$person = new Person('Jane Doe', 32, false);
$json = $serializer->serialize($person, 'json');
// $json contains {"name":"Jane Doe","age":32,"sportsperson":false}

$person1 = $serializer->deserialize(
    '{"name":"Jane Doe","age":32,"sportsperson":false","potentiallySpamUser":false}',
    Person::class,
    'json'
);
// the "potentiallySpamUser" value is ignored
```

### 使用上下文忽略属性

你也可以在运行时通过 `ignored_attributes` 上下文选项传递要忽略的属性名称数组：

```php
use Symfony\Component\Serializer\Normalizer\AbstractNormalizer;

// ...
$person = new Person('Jane Doe', 32, false);
$json = $serializer->serialize($person, 'json',
[
    AbstractNormalizer::IGNORED_ATTRIBUTES => ['age'],
]);
// $json contains {"name":"Jane Doe","sportsperson":false}
```

但是，如果过度使用这种方式，可能很快变得难以维护。请参阅下一节关于*序列化分组*的内容，以获得更好的解决方案。

## 选择特定属性

与其在所有情况下排除某个属性或方法，不如在某个地方排除某些属性，同时在另一个地方序列化它们。分组是实现这一点的便捷方法。

你可以将 `#[Groups]` 属性添加到你的类中：

**PHP 属性方式：**

```php
// src/Model/Person.php
namespace App\Model;

use Symfony\Component\Serializer\Attribute\Groups;

class Person
{
    #[Groups(["admin-view"])]
    private int $age;

    #[Groups(["public-view"])]
    private string $name;

    #[Groups(["public-view"])]
    private bool $sportsperson;

    private string $email;

    // ...
}
```

**YAML 配置：**

```yaml
# config/serializer/person.yaml
App\Model\Person:
    attributes:
        age:
            groups: ['admin-view']
        name:
            groups: ['public-view']
        sportsperson:
            groups: ['public-view']
        # email has no groups defined
```

**XML 配置：**

```xml
<!-- config/serializer/person.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<serializer xmlns="http://symfony.com/schema/dic/serializer-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/serializer-mapping
        https://symfony.com/schema/dic/serializer-mapping/serializer-mapping-1.0.xsd"
>
    <class name="App\Model\Person">
        <attribute name="age">
            <group>admin-view</group>
        </attribute>
        <attribute name="name">
            <group>public-view</group>
        </attribute>
        <attribute name="sportsperson">
            <group>public-view</group>
        </attribute>
        <!-- email has no groups defined -->
    </class>
</serializer>
```

现在你可以选择序列化时使用哪些分组：

```php
$json = $serializer->serialize(
    $person,
    'json',
    ['groups' => 'public-view']
);
// $json contains {"name":"Jane Doe","sportsperson":false}

// you can also pass an array of groups
$json = $serializer->serialize(
    $person,
    'json',
    ['groups' => ['public-view', 'admin-view']]
);
// $json contains {"name":"Jane Doe","age":32,"sportsperson":false}

// or use the special "*" value to serialize all properties
// (including those without any groups)
$json = $serializer->serialize(
    $person,
    'json',
    ['groups' => '*']
);
// $json contains {"name":"Jane Doe","age":32,"sportsperson":false,"email":"jane.doe@exemple.com"}
```

### 使用序列化上下文

最后，你还可以在运行时使用 `attributes` 上下文选项来选择属性：

```php
use Symfony\Component\Serializer\Normalizer\AbstractNormalizer;
// ...

$json = $serializer->serialize($person, 'json', [
    AbstractNormalizer::ATTRIBUTES => ['name', 'company' => ['name']]
]);
// $json contains {"name":"Dunglas","company":{"name":"Les-Tilleuls.coop"}}
```

只有[未被忽略](#忽略属性)的属性才可用。如果设置了序列化分组，则只能使用这些分组允许的属性。

## 处理数组

序列化器能够处理对象数组。序列化数组的方式与序列化单个对象相同：

```php
use App\Model\Person;

// ...
$person1 = new Person('Jane Doe', 39, false);
$person2 = new Person('John Smith', 52, true);

$persons = [$person1, $person2];
$jsonContent = $serializer->serialize($persons, 'json');

// $jsonContent contains [{"name":"Jane Doe","age":39,"sportsman":false},{"name":"John Smith","age":52,"sportsman":true}]
```

要反序列化对象列表，必须在类型参数后附加 `[]`：

```php
// ...

$jsonData = ...; // the serialized JSON data from the previous example
$persons = $serializer->deserialize($JsonData, Person::class.'[]', 'json');
```

对于嵌套类，必须在属性、构造函数或 setter 中添加 PHPDoc 类型：

```php
// src/Model/UserGroup.php
namespace App\Model;

class UserGroup
{
    /**
     * @param Person[] $members
     */
    public function __construct(
        private array $members,
    ) {
    }

    // or if you're using a setter

    /**
     * @param Person[] $members
     */
    public function setMembers(array $members): void
    {
        $this->members = $members;
    }

    // ...
}
```

> **提示：** 序列化器还支持静态分析中使用的数组类型，如 `list<Person>` 和 `array<Person>`。请确保安装了 `phpstan/phpdoc-parser` 和 `phpdocumentor/reflection-docblock` 包（这些都是 `symfony/serializer-pack` 的一部分）。

## 反序列化嵌套结构

某些 API 可能会提供你希望在 PHP 对象中扁平化的冗长嵌套结构。例如，想象这样一个 JSON 响应：

```json
{
    "id": "123",
    "profile": {
        "username": "jdoe",
        "personal_information": {
            "full_name": "Jane Doe"
        }
    }
}
```

你可能希望将此信息序列化为单个 PHP 对象：

```php
class Person
{
    private int $id;
    private string $username;
    private string $fullName;
}
```

使用 `#[SerializedPath]` 使用[有效的 PropertyAccess 语法](https://symfony.com/doc/current/components/property_access.html)指定嵌套属性的路径：

**PHP 属性方式：**

```php
namespace App\Model;

use Symfony\Component\Serializer\Attribute\SerializedPath;

class Person
{
    private int $id;

    #[SerializedPath('[profile][username]')]
    private string $username;

    #[SerializedPath('[profile][personal_information][full_name]')]
    private string $fullName;
}
```

**YAML 配置：**

```yaml
App\Model\Person:
    attributes:
        username:
            serialized_path: '[profile][username]'
        fullName:
            serialized_path: '[profile][personal_information][full_name]'
```

**XML 配置：**

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<serializer xmlns="http://symfony.com/schema/dic/serializer-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/serializer-mapping
        https://symfony.com/schema/dic/serializer-mapping/serializer-mapping-1.0.xsd"
>
    <class name="App\Model\Person">
        <attribute name="username" serialized-path="[profile][username]"/>
        <attribute name="fullName" serialized-path="[profile][personal_information][full_name]"/>
    </class>
</serializer>
```

> **警告：** `SerializedPath` 不能与同一属性的 `SerializedName` 结合使用。

`#[SerializedPath]` 属性也适用于 PHP 对象的序列化：

```php
use App\Model\Person;
// ...

$person = new Person(123, 'jdoe', 'Jane Doe');
$jsonContent = $serializer->serialize($person, 'json');
// $jsonContent contains {"id":123,"profile":{"username":"jdoe","personal_information":{"full_name":"Jane Doe"}}}
```

## 在序列化和反序列化时转换属性名称

有时序列化属性的名称必须与 PHP 类的属性或 getter/setter 方法不同。这可以通过名称转换器来实现。

序列化器服务使用 `Symfony\Component\Serializer\NameConverter\MetadataAwareNameConverter`。使用此名称转换器，你可以使用 `#[SerializedName]` 属性更改属性的名称：

**PHP 属性方式：**

```php
// src/Model/Person.php
namespace App\Model;

use Symfony\Component\Serializer\Attribute\SerializedName;

class Person
{
    #[SerializedName('customer_name')]
    private string $name;

    // ...
}
```

**YAML 配置：**

```yaml
# config/serializer/person.yaml
App\Entity\Person:
    attributes:
        name:
            serialized_name: customer_name
```

**XML 配置：**

```xml
<!-- config/serializer/person.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<serializer xmlns="http://symfony.com/schema/dic/serializer-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/serializer-mapping
        https://symfony.com/schema/dic/serializer-mapping/serializer-mapping-1.0.xsd"
>
    <class name="App\Entity\Person">
        <attribute name="name" serialized-name="customer_name"/>
    </class>
</serializer>
```

此自定义映射用于在序列化和反序列化对象时转换属性名称：

**Symfony 框架方式：**

```php
// ...

$json = $serializer->serialize($person, 'json');
// $json contains {"customer_name":"Jane Doe", ...}
```

**独立使用方式：**

```php
use App\Model\Person;
use Symfony\Component\Serializer\Mapping\Factory\ClassMetadataFactory;
use Symfony\Component\Serializer\Mapping\Loader\AttributeLoader;
use Symfony\Component\Serializer\NameConverter\MetadataAwareNameConverter;
use Symfony\Component\Serializer\Normalizer\ObjectNormalizer;
use Symfony\Component\Serializer\Serializer;

// ...

// Configure a loader to retrieve mapping information like SerializedName.
// E.g. when using PHP attributes:
$classMetadataFactory = new ClassMetadataFactory(new AttributeLoader());
$nameConverter = new MetadataAwareNameConverter($classMetadataFactory);
$normalizers = [
    new ObjectNormalizer($classMetadataFactory, $nameConverter),
];

$serializer = new Serializer($normalizers, $encoders);

$person = new Person('Jane Doe', 32, false);
$json = $serializer->serialize($person, 'json');
// $json contains {"customer_name":"Jane Doe", ...}
```

> **另请参阅：** 你也可以创建自定义名称转换器类。更多信息请参阅 [/serializer/custom_name_converter](https://symfony.com/doc/current/serializer/custom_name_converter.html)。

### CamelCase 转 snake_case

在许多格式中，通常使用下划线分隔单词（也称为 snake_case）。然而，在 Symfony 应用程序中，通常使用 camelCase 来命名属性。

Symfony 提供了一个内置的名称转换器，专门用于在序列化和反序列化过程中在 snake_case 和 CamelCase 风格之间进行转换。你可以通过将 `name_converter` 设置设为 `serializer.name_converter.camel_case_to_snake_case` 来使用它，而不是元数据感知名称转换器：

**YAML 配置：**

```yaml
# config/packages/serializer.yaml
framework:
    serializer:
        name_converter: 'serializer.name_converter.camel_case_to_snake_case'
```

**PHP 配置：**

```php
// config/packages/serializer.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'serializer' => [
            'name_converter' => 'serializer.name_converter.camel_case_to_snake_case',
        ],
    ],
]);
```

**独立使用方式：**

```php
use Symfony\Component\Serializer\NameConverter\CamelCaseToSnakeCaseNameConverter;
use Symfony\Component\Serializer\Normalizer\ObjectNormalizer;

// ...
$normalizers = [
    new ObjectNormalizer(null, new CamelCaseToSnakeCaseNameConverter()),
];
$serializer = new Serializer($normalizers, $encoders);
```

### snake_case 转 CamelCase

在 Symfony 应用程序中，通常使用 camelCase 命名属性。但某些包可能遵循 snake_case 约定。

Symfony 提供了一个内置的名称转换器，专门用于在序列化和反序列化过程中在 CamelCase 和 snake_case 风格之间进行转换。你可以通过将 `name_converter` 设置设为 `serializer.name_converter.snake_case_to_camel_case` 来使用它，而不是元数据感知名称转换器：

**YAML 配置：**

```yaml
# config/packages/serializer.yaml
framework:
    serializer:
        name_converter: 'serializer.name_converter.snake_case_to_camel_case'
```

**PHP 配置：**

```php
// config/packages/serializer.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'serializer' => [
            'name_converter' => 'serializer.name_converter.snake_case_to_camel_case',
        ],
    ],
]);
```

**独立使用方式：**

```php
use Symfony\Component\Serializer\NameConverter\SnakeCaseToCamelCaseNameConverter;
use Symfony\Component\Serializer\Normalizer\ObjectNormalizer;

// ...
$normalizers = [
    new ObjectNormalizer(null, new SnakeCaseToCamelCaseNameConverter()),
];
$serializer = new Serializer($normalizers, $encoders);
```

## 序列化器规范化器

默认情况下，序列化器服务按以下优先级顺序配置了以下规范化器：

**`Symfony\Component\Serializer\Normalizer\UnwrappingDenormalizer`**
可用于仅反规范化输入的一部分，稍后在本文中[详细介绍](#部分反序列化输入解包)。

**`Symfony\Component\Serializer\Normalizer\ProblemNormalizer`**
根据 API Problem 规范 [RFC 7807](https://tools.ietf.org/html/rfc7807) 规范化 `Symfony\Component\ErrorHandler\Exception\FlattenException` 错误。

**`Symfony\Component\Serializer\Normalizer\UidNormalizer`**
规范化扩展 `Symfony\Component\Uid\AbstractUid` 的对象。

实现 `Symfony\Component\Uid\Uuid` 的对象的默认规范化格式为 [RFC 4122](https://tools.ietf.org/html/rfc4122) 格式（例如：`d9e7a184-5d5b-11ea-a62a-3499710062d0`）。实现 `Symfony\Component\Uid\Ulid` 的对象的默认规范化格式为 Base 32 格式（例如：`01E439TP9XJZ9RPFH3T1PYBCR8`）。你可以通过将序列化器上下文选项 `UidNormalizer::NORMALIZATION_FORMAT_KEY` 设置为 `UidNormalizer::NORMALIZATION_FORMAT_BASE58`、`UidNormalizer::NORMALIZATION_FORMAT_BASE32` 或 `UidNormalizer::NORMALIZATION_FORMAT_RFC4122` 来更改字符串格式。

它还可以将 `uuid` 或 `ulid` 字符串反规范化为 `Symfony\Component\Uid\Uuid` 或 `Symfony\Component\Uid\Ulid`。格式不重要。

**`Symfony\Component\Serializer\Normalizer\DateTimeNormalizer`**
在 `DateTimeInterface` 对象（例如 `DateTime` 和 `DateTimeImmutable`）与字符串、整数或浮点数之间进行规范化。

`DateTime` 和 `DateTimeImmutable` 默认使用 [RFC 3339](https://tools.ietf.org/html/rfc3339#section-5.8) 格式转换为字符串。使用 `DateTimeNormalizer::FORMAT_KEY` 和 `DateTimeNormalizer::TIMEZONE_KEY` 来更改格式。

要始终使用上下文中指定的时区创建 `DateTime` 和 `DateTimeImmutable` 对象，将 `DateTimeNormalizer::FORCE_TIMEZONE_KEY` 上下文选项设置为 `true`。这会强制使用上下文时区并忽略输入中提供的任何时区。

要将对象转换为整数或浮点数，将序列化器上下文选项 `DateTimeNormalizer::CAST_KEY` 设置为 `int` 或 `float`。

**`Symfony\Component\Serializer\Normalizer\ConstraintViolationListNormalizer`**
根据 [RFC 7807](https://tools.ietf.org/html/rfc7807) 标准，将实现 `Symfony\Component\Validator\ConstraintViolationListInterface` 的对象转换为错误列表。

**`Symfony\Component\Serializer\Normalizer\DateTimeZoneNormalizer`**
在 `DateTimeZone` 对象与根据 [PHP 时区列表](https://www.php.net/manual/en/timezones.php) 表示时区名称的字符串之间进行转换。

**`Symfony\Component\Serializer\Normalizer\DateIntervalNormalizer`**
在 `DateInterval` 对象与字符串之间进行规范化。默认使用 `P%yY%mM%dDT%hH%iM%sS` 格式。使用 `DateIntervalNormalizer::FORMAT_KEY` 选项可以更改此设置。

**`Symfony\Component\Serializer\Normalizer\FormErrorNormalizer`**
适用于实现 `Symfony\Component\Form\FormInterface` 的类。

它将从表单中获取错误，并根据 API Problem 规范 [RFC 7807](https://tools.ietf.org/html/rfc7807) 对其进行规范化。

**`Symfony\Component\Serializer\Normalizer\TranslatableNormalizer`**
使用[翻译器](https://symfony.com/doc/current/translation.html)将实现 `Symfony\Contracts\Translation\TranslatableInterface` 的对象转换为已翻译的字符串。

你可以通过设置 `TranslatableNormalizer::NORMALIZATION_LOCALE_KEY` 上下文选项来定义翻译对象所使用的语言环境。

**`Symfony\Component\Serializer\Normalizer\BackedEnumNormalizer`**
在 `BackedEnum` 枚举与字符串或整数之间进行转换。

默认情况下，当数据不是有效的 backed 枚举时会抛出异常。如果你想要 `null` 而不是异常，可以设置 `BackedEnumNormalizer::ALLOW_INVALID_VALUES` 选项。

**`Symfony\Component\Serializer\Normalizer\NumberNormalizer`**
在 `BcMath\Number` 或 `GMP` 对象与字符串或整数之间进行转换。

**`Symfony\Component\Serializer\Normalizer\DataUriNormalizer`**
在 `SplFileInfo` 对象与 [data URI](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs) 字符串（`data:...`）之间进行转换，使文件可以嵌入到序列化数据中。

**`Symfony\Component\Serializer\Normalizer\JsonSerializableNormalizer`**
适用于实现 `JsonSerializable` 的类。

它会调用 `JsonSerializable::jsonSerialize` 方法，然后进一步规范化结果。这意味着嵌套的 `JsonSerializable` 类也将被规范化。

当你想要从使用简单 `json_encode` 的现有代码库逐步迁移到 Symfony 序列化器时，此规范化器特别有用，因为它允许你混合使用哪些规范化器用于哪些类。

与 `json_encode` 不同，可以处理循环引用。

**`Symfony\Component\Serializer\Normalizer\ArrayDenormalizer`**
将数组的数组转换为对象数组（具有给定类型）。请参阅[处理数组](#处理数组)。

使用 `Symfony\Component\PropertyInfo\PropertyInfoExtractor` 提供带有 `@var Person[]` 等注解的提示：

```php
use Symfony\Component\PropertyInfo\Extractor\PhpDocExtractor;
use Symfony\Component\PropertyInfo\Extractor\ReflectionExtractor;
use Symfony\Component\PropertyInfo\PropertyInfoExtractor;
use Symfony\Component\Serializer\Encoder\JsonEncoder;
use Symfony\Component\Serializer\Mapping\Factory\ClassMetadataFactory;
use Symfony\Component\Serializer\Mapping\Loader\AttributeLoader;
use Symfony\Component\Serializer\Normalizer\ArrayDenormalizer;
use Symfony\Component\Serializer\Normalizer\ObjectNormalizer;
use Symfony\Component\Serializer\Serializer;

$propertyInfo = new PropertyInfoExtractor([], [new PhpDocExtractor(), new ReflectionExtractor()]);
$normalizers = [new ObjectNormalizer(new ClassMetadataFactory(new AttributeLoader()), null, null, $propertyInfo), new ArrayDenormalizer()];

$this->serializer = new Serializer($normalizers, [new JsonEncoder()]);
```

**`Symfony\Component\Serializer\Normalizer\ObjectNormalizer`**
这是最强大的默认规范化器，用于其他规范化器无法规范化的任何对象。

它利用 [PropertyAccess 组件](https://symfony.com/doc/current/components/property_access.html) 在对象中读取和写入。这允许它直接访问属性，或者通过 getters、setters、hassers、issers、canners、adders 和 removers 访问。名称通过从方法名称中删除 `get`、`set`、`has`、`is`、`can`、`add` 或 `remove` 前缀并将第一个字母转换为小写来生成（例如 `getFirstName()` -> `firstName`）。

在反规范化期间，它支持使用构造函数以及发现的方法。

> **危险：** 在序列化 `DateTime` 或 `DateTimeImmutable` 类时，请始终确保注册了 `DateTimeNormalizer`，以避免过多的内存使用和暴露内部细节。

### 内置规范化器

除了默认注册的规范化器（见上一节）之外，序列化器组件还提供了一些额外的规范化器。你可以通过定义服务并使用 [`serializer.normalizer`](https://symfony.com/doc/current/reference/dic_tags.html#serializer-normalizer) 标签来注册这些规范化器。例如，要使用 `CustomNormalizer`，你必须定义如下服务：

**YAML 配置：**

```yaml
# config/services.yaml
services:
    # ...

    # if you're using autoconfigure, the tag will be automatically applied
    Symfony\Component\Serializer\Normalizer\CustomNormalizer:
        tags:
            # register the normalizer with a high priority (called earlier)
            - { name: 'serializer.normalizer', priority: 500 }
```

**PHP 配置：**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\Serializer\Normalizer\CustomNormalizer;

return App::config([
    'services' => [
        // if you're using autoconfigure, the tag will be automatically applied
        CustomNormalizer::class => [
            'tags' => [
                // register the normalizer with a high priority (called earlier)
                ['serializer.normalizer' => ['priority' => 500]],
            ],
        ],
    ],
]);
```

**`Symfony\Component\Serializer\Normalizer\CustomNormalizer`**
在规范化时调用 PHP 对象上的方法。PHP 对象必须实现 `Symfony\Component\Serializer\Normalizer\NormalizableInterface` 和/或 `Symfony\Component\Serializer\Normalizer\DenormalizableInterface`。

**`Symfony\Component\Serializer\Normalizer\GetSetMethodNormalizer`**
这是默认 `ObjectNormalizer` 的替代方案。它通过调用"getters"（以 `get`、`has`、`is` 或 `can` 开头的公共方法）来读取类的内容。它将通过调用构造函数和"setters"（以 `set` 开头的公共方法）来反规范化数据。

对象被规范化为属性名称和值的映射（名称通过从方法名称中删除 `get` 前缀并将第一个字母转换为小写来生成；例如 `getFirstName()` -> `firstName`）。

**`Symfony\Component\Serializer\Normalizer\PropertyNormalizer`**
这是 `ObjectNormalizer` 的另一个替代方案。此规范化器使用 [PHP 反射](https://php.net/manual/en/book.reflection.php) 直接读取和写入公共属性以及**私有和受保护**属性（来自类及其所有父类）。它支持在反规范化过程中调用构造函数。

对象被规范化为属性名称到属性值的映射。

你还可以使用 `PropertyNormalizer::NORMALIZE_VISIBILITY` 上下文选项限制规范化器仅使用具有特定可见性的属性（例如只有公共属性）。你可以将其设置为 `PropertyNormalizer::NORMALIZE_PUBLIC`、`PropertyNormalizer::NORMALIZE_PROTECTED` 和 `PropertyNormalizer::NORMALIZE_PRIVATE` 常量的任意组合：

```php
use Symfony\Component\Serializer\Normalizer\PropertyNormalizer;
// ...

$json = $serializer->serialize($person, 'json', [
    // only serialize public properties
    PropertyNormalizer::NORMALIZE_VISIBILITY => PropertyNormalizer::NORMALIZE_PUBLIC,

    // serialize public and protected properties
    PropertyNormalizer::NORMALIZE_VISIBILITY => PropertyNormalizer::NORMALIZE_PUBLIC | PropertyNormalizer::NORMALIZE_PROTECTED,
]);
```

## 命名序列化器

有时，你可能需要为序列化器提供多种配置，例如不同的默认上下文、名称转换器或规范化器和编码器集，具体取决于用例。例如，当你的应用程序与多个 API 通信时，每个 API 都遵循自己的序列化规则集。

你可以通过使用 `named_serializers` 选项配置多个序列化器实例来实现：

**YAML 配置：**

```yaml
# config/packages/serializer.yaml
framework:
    serializer:
        named_serializers:
            api_client1:
                name_converter: 'serializer.name_converter.camel_case_to_snake_case'
                default_context:
                    enable_max_depth: true
            api_client2:
                default_context:
                    enable_max_depth: false
```

**PHP 配置：**

```php
// config/packages/serializer.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'serializer' => [
            'named_serializers' => [
                'api_client1' => [
                    'name_converter' => 'serializer.name_converter.camel_case_to_snake_case',
                    'default_context' => [
                        'enable_max_depth' => true,
                    ],
                ],
                'api_client2' => [
                    'default_context' => [
                        'enable_max_depth' => false,
                    ],
                ],
            ],
        ],
    ],
]);
```

你可以使用[命名别名](https://symfony.com/doc/current/service_container/autowiring.html#autowiring-multiple-implementations-same-type)注入这些不同的序列化器实例：

```php
namespace App\Controller;

// ...
use Symfony\Component\DependencyInjection\Attribute\Target;

class PersonController extends AbstractController
{
    public function index(
        SerializerInterface $serializer,           // default serializer
        SerializerInterface $apiClient1Serializer, // api_client1 serializer
        #[Target('apiClient2.serializer')]         // api_client2 serializer
        SerializerInterface $customName,
    ) {
        // ...
    }
}
```

默认情况下，命名序列化器使用与主序列化器服务相同的内置规范化器和编码器集。但是，你可以通过为特定命名序列化器注册额外的规范化器或编码器来自定义它们。为此，在 [`serializer.normalizer`](https://symfony.com/doc/current/reference/dic_tags.html#serializer-normalizer) 或 [`serializer.encoder`](https://symfony.com/doc/current/reference/dic_tags.html#serializer-encoder) 标签中添加 `serializer` 属性：

**YAML 配置：**

```yaml
# config/services.yaml
services:
    # ...

    Symfony\Component\Serializer\Normalizer\CustomNormalizer:
        tags:
            # add this normalizer only to a specific named serializer
            - serializer.normalizer: { serializer: 'api_client1' }
            # add this normalizer to several named serializers
            - serializer.normalizer: { serializer: [ 'api_client1', 'api_client2' ] }
            # add this normalizer to all serializers, including the default one
            - serializer.normalizer: { serializer: '*' }
```

**PHP 配置：**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\Serializer\Normalizer\CustomNormalizer;

return App::config([
    'services' => [
        CustomNormalizer::class => [
            // prevent this normalizer from being automatically added to the default serializer
            'autoconfigure' => false,
            'tags' => [
                // add this normalizer only to a specific named serializer
                ['serializer.normalizer' => ['serializer' => 'api_client1']],
                // add this normalizer to several named serializers
                ['serializer.normalizer' => ['serializer' => ['api_client1', 'api_client2']]],
                // add this normalizer to all serializers, including the default one
                ['serializer.normalizer' => ['serializer' => '*']],
            ],
        ],
    ],
]);
```

当未设置 `serializer` 属性时，该服务仅注册到默认序列化器。

命名序列化器中使用的每个规范化器或编码器都带有 `serializer.normalizer.<name>` 或 `serializer.encoder.<name>` 标签。你可以使用以下命令检查它们的优先级：

```bash
$ php bin/console debug:container --tag serializer.<normalizer|encoder>.<name>
```

此外，你可以通过将 `include_built_in_normalizers` 和 `include_built_in_encoders` 选项设置为 `false`，从命名序列化器中排除默认的规范化器和编码器集：

**YAML 配置：**

```yaml
# config/packages/serializer.yaml
framework:
    serializer:
        named_serializers:
            api_client1:
                include_built_in_normalizers: false
                include_built_in_encoders: true
```

**PHP 配置：**

```php
// config/packages/serializer.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'serializer' => [
            'named_serializers' => [
                'api_client1' => [
                    'include_built_in_normalizers' => false,
                    'include_built_in_encoders' => true,
                ],
            ],
        ],
    ],
]);
```

## 调试序列化器

使用 `debug:serializer` 命令转储给定类的序列化器元数据：

```bash
$ php bin/console debug:serializer 'App\Entity\Book'

    App\Entity\Book
    ---------------

    +----------+------------------------------------------------------------+
    | Property | Options                                                    |
    +----------+------------------------------------------------------------+
    | name     | [                                                          |
    |          |   "groups" => [                                            |
    |          |       "book:read",                                         |
    |          |       "book:write",                                        |
    |          |   ],                                                       |
    |          |   "maxDepth" => 1,                                         |
    |          |   "serializedName" => "book_name",                         |
    |          |   "serializedPath" => null,                                |
    |          |   "ignore" => false,                                       |
    |          |   "normalizationContexts" => [],                           |
    |          |   "denormalizationContexts" => []                          |
    |          | ]                                                          |
    | isbn     | [                                                          |
    |          |   "groups" => [                                            |
    |          |       "book:read",                                         |
    |          |   ],                                                       |
    |          |   "maxDepth" => null,                                      |
    |          |   "serializedName" => null,                                |
    |          |   "serializedPath" => "[data][isbn]",                      |
    |          |   "ignore" => false,                                       |
    |          |   "normalizationContexts" => [],                           |
    |          |   "denormalizationContexts" => []                          |
    |          | ]                                                          |
    +----------+------------------------------------------------------------+
```

## 高级序列化

### 跳过 `null` 值

默认情况下，序列化器会保留包含 `null` 值的属性。你可以通过将 `AbstractObjectNormalizer::SKIP_NULL_VALUES` 上下文选项设置为 `true` 来更改此行为：

```php
class Person
{
    public string $name = 'Jane Doe';
    public ?string $gender = null;
}

$jsonContent = $serializer->serialize(new Person(), 'json', [
    AbstractObjectNormalizer::SKIP_NULL_VALUES => true,
]);
// $jsonContent contains {"name":"Jane Doe"}
```

### 保留空对象

默认情况下，序列化器将空数组转换为 `[]`。你可以通过将 `AbstractObjectNormalizer::PRESERVE_EMPTY_OBJECTS` 上下文选项设置为 `true` 来更改此行为。当值是 `\ArrayObject()` 的实例时，序列化后的数据将为 `{}`。

### 处理未初始化的属性

在 PHP 中，类型化属性具有 `uninitialized` 状态，这与非类型化属性的默认 `null` 不同。当你在给类型化属性显式赋值之前尝试访问它时，会得到一个错误。

为了避免序列化器在序列化或规范化具有未初始化属性的对象时抛出错误，默认情况下 `ObjectNormalizer` 会捕获这些错误并忽略此类属性。

你可以通过将 `AbstractObjectNormalizer::SKIP_UNINITIALIZED_VALUES` 上下文选项设置为 `false` 来禁用此行为：

```php
class Person {
    public string $name = 'Jane Doe';
    public string $phoneNumber; // uninitialized
}

$jsonContent = $normalizer->serialize(new Dummy(), 'json', [
    AbstractObjectNormalizer::SKIP_UNINITIALIZED_VALUES => false,
]);
// throws Symfony\Component\PropertyAccess\Exception\UninitializedPropertyException
// as the ObjectNormalizer cannot read uninitialized properties
```

> **注意：** 当 `AbstractObjectNormalizer::SKIP_UNINITIALIZED_VALUES` 上下文选项设置为 `false` 时，将 `Symfony\Component\Serializer\Normalizer\PropertyNormalizer` 或 `Symfony\Component\Serializer\Normalizer\GetSetMethodNormalizer` 与具有未初始化属性的对象一起使用，将抛出 `\Error` 实例，因为规范化器无法读取未初始化的属性（直接或通过 getter/isser 方法）。

### 处理循环引用

处理关联对象时，循环引用很常见：

```php
class Organization
{
    public function __construct(
        private string $name,
        private array $members = []
    ) {
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function addMember(Member $member): void
    {
        $this->members[] = $member;
    }

    public function getMembers(): array
    {
        return $this->members;
    }
}

class Member
{
    private Organization $organization;

    public function __construct(
        private string $name
    ) {
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function setOrganization(Organization $organization): void
    {
        $this->organization = $organization;
    }

    public function getOrganization(): Organization
    {
        return $this->organization;
    }
}
```

为了避免无限循环，规范化器在遇到此类情况时会抛出 `Symfony\Component\Serializer\Exception\CircularReferenceException`：

```php
$organization = new Organization('Les-Tilleuls.coop');
$member = new Member('Kévin');

$organization->addMember($member);
$member->setOrganization($organization);

$jsonContent = $serializer->serialize($organization, 'json');
// throws a CircularReferenceException
```

上下文中的 `circular_reference_limit` 键设置在将同一对象视为循环引用之前序列化该对象的次数。默认值为 `1`。

除了抛出异常之外，循环引用也可以通过自定义可调用函数来处理。当序列化具有唯一标识符的实体时，这特别有用：

```php
use Symfony\Component\Serializer\Exception\CircularReferenceException;
use Symfony\Component\Serializer\Normalizer\AbstractNormalizer;

$context = [
    AbstractNormalizer::CIRCULAR_REFERENCE_HANDLER => function (object $object, ?string $format, array $context): string {
        if (!$object instanceof Organization) {
            throw new CircularReferenceException('A circular reference has been detected when serializing the object of class "'.get_debug_type($object).'".');
        }

        // serialize the nested Organization with only the name (and not the members)
        return $object->getName();
    },
];

$jsonContent = $serializer->serialize($organization, 'json', $context);
// $jsonContent contains {"name":"Les-Tilleuls.coop","members":[{"name":"K\u00e9vin", organization: "Les-Tilleuls.coop"}]}
```

### 处理序列化深度

序列化器还可以检测同一类的嵌套对象并限制序列化深度。这对于树形结构很有用，其中同一对象嵌套了多次。

例如，假设有一个家族树的数据结构：

```php
// ...
class Person
{
    // ...

    public function __construct(
        private string $name,
        private ?self $mother
    ) {
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function getMother(): ?self
    {
        return $this->mother;
    }

    // ...
}

// ...
$greatGrandmother = new Person('Elizabeth', null);
$grandmother = new Person('Jane', $greatGrandmother);
$mother = new Person('Sophie', $grandmother);
$child = new Person('Joe', $mother);
```

你可以为给定属性指定最大深度。例如，你可以将最大深度设置为 `1`，以始终只序列化某人的母亲（而不是祖母等）：

**PHP 属性方式：**

```php
// src/Model/Person.php
namespace App\Model;

use Symfony\Component\Serializer\Attribute\MaxDepth;

class Person
{
    #[MaxDepth(1)]
    private ?self $mother;

    // ...
}
```

**YAML 配置：**

```yaml
# config/serializer/person.yaml
App\Model\Person:
    attributes:
        mother:
            max_depth: 1
```

**XML 配置：**

```xml
<!-- config/serializer/person.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<serializer xmlns="http://symfony.com/schema/dic/serializer-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/serializer-mapping
        https://symfony.com/schema/dic/serializer-mapping/serializer-mapping-1.0.xsd"
>
    <class name="App\Model\Person">
        <attribute name="mother" max-depth="1"/>
    </class>
</serializer>
```

要限制序列化深度，必须在上下文中（或在 `framework.yaml` 中指定的默认上下文中）将 `AbstractObjectNormalizer::ENABLE_MAX_DEPTH` 键设置为 `true`：

```php
// ...
$greatGrandmother = new Person('Elizabeth', null);
$grandmother = new Person('Jane', $greatGrandmother);
$mother = new Person('Sophie', $grandmother);
$child = new Person('Joe', $mother);

$jsonContent = $serializer->serialize($child, null, [
    AbstractObjectNormalizer::ENABLE_MAX_DEPTH => true
]);
// $jsonContent contains {"name":"Joe","mother":{"name":"Sophie"}}
```

你还可以配置自定义可调用函数，该函数在达到最大深度时使用。例如，可以用于返回下一个嵌套对象的唯一标识符，而不是省略该属性：

```php
use Symfony\Component\Serializer\Normalizer\AbstractObjectNormalizer;
// ...

$greatGrandmother = new Person('Elizabeth', null);
$grandmother = new Person('Jane', $greatGrandmother);
$mother = new Person('Sophie', $grandmother);
$child = new Person('Joe', $mother);

// all callback parameters are optional (you can omit the ones you don't use)
$maxDepthHandler = function (object $innerObject, object $outerObject, string $attributeName, ?string $format = null, array $context = []): ?string {
    // return only the name of the next person in the tree
    return $innerObject instanceof Person ? $innerObject->getName() : null;
};

$jsonContent = $serializer->serialize($child, null, [
    AbstractObjectNormalizer::ENABLE_MAX_DEPTH => true,
    AbstractObjectNormalizer::MAX_DEPTH_HANDLER => $maxDepthHandler,
]);
// $jsonContent contains {"name":"Joe","mother":{"name":"Sophie","mother":"Jane"}}
```

### 使用回调函数序列化具有对象实例的属性

在序列化时，可以设置回调函数来格式化特定的对象属性。这可以代替[为分组定义上下文](#在特定属性上配置上下文)：

```php
$person = new Person('cordoval', 34);
$person->setCreatedAt(new \DateTime('now'));

$context = [
    AbstractNormalizer::CALLBACKS => [
        // all callback parameters are optional (you can omit the ones you don't use)
        'createdAt' => function (object $attributeValue, object $object, string $attributeName, ?string $format = null, array $context = []) {
            return $attributeValue instanceof \DateTime ? $attributeValue->format(\DateTime::ATOM) : '';
        },
    ],
];
$jsonContent = $serializer->serialize($person, 'json', $context);
// $jsonContent contains {"name":"cordoval","age":34,"createdAt":"2014-03-22T09:43:12-0500"}
```

## 高级反序列化

### 要求所有属性

默认情况下，当未提供可空属性的参数时，序列化器会将 `null` 添加到这些属性。你可以通过将 `AbstractNormalizer::REQUIRE_ALL_PROPERTIES` 上下文选项设置为 `true` 来更改此行为：

```php
class Person
{
    public function __construct(
        public string $firstName,
        public ?string $lastName,
    ) {
    }
}

// ...
$data = ['firstName' => 'John'];
$person = $serializer->deserialize($data, Person::class, 'json', [
    AbstractNormalizer::REQUIRE_ALL_PROPERTIES => true,
]);
// throws Symfony\Component\Serializer\Exception\MissingConstructorArgumentException
```

### 在反规范化时收集类型错误

当将负载反规范化为具有类型化属性的对象时，如果负载包含与对象类型不同的属性，你将得到一个异常。

使用 `COLLECT_DENORMALIZATION_ERRORS` 选项一次性收集所有异常，并获取部分反规范化的对象：

```php
try {
    $person = $serializer->deserialize($jsonString, Person::class, 'json', [
        DenormalizerInterface::COLLECT_DENORMALIZATION_ERRORS => true,
    ]);
} catch (PartialDenormalizationException $e) {
    $violations = new ConstraintViolationList();

    /** @var NotNormalizableValueException $exception */
    foreach ($e->getErrors() as $exception) {
        $message = sprintf('The type must be one of "%s" ("%s" given).', implode(', ', $exception->getExpectedTypes()), $exception->getCurrentType());
        $parameters = [];
        if ($exception->canUseMessageForUser()) {
            $parameters['hint'] = $exception->getMessage();
        }
        $violations->add(new ConstraintViolation($message, '', $parameters, null, $exception->getPath(), null));
    }

    // ... return violation list to the user
}
```

### 在现有对象中反序列化

序列化器也可以用于更新现有对象。你可以通过配置 `object_to_populate` 序列化器上下文选项来实现：

```php
use Symfony\Component\Serializer\Normalizer\AbstractNormalizer;

// ...
$person = new Person('Jane Doe', 59);

$serializer->deserialize($jsonData, Person::class, 'json', [
    AbstractNormalizer::OBJECT_TO_POPULATE => $person,
]);
// instead of returning a new object, $person is updated instead
```

> **注意：** `AbstractNormalizer::OBJECT_TO_POPULATE` 选项仅用于顶级对象。如果该对象是树形结构的根，则规范化数据中存在的所有子元素将使用新实例重新创建。
>
> 当 `AbstractObjectNormalizer::DEEP_OBJECT_TO_POPULATE` 上下文选项设置为 `true` 时，根 `OBJECT_TO_POPULATE` 的现有子级将从规范化数据中更新，而不是由反规范化器重新创建它们。这仅适用于单个子对象，不适用于对象数组。当规范化数据中存在对象数组时，这些数组仍将被替换。

### 反序列化接口和抽象类

在处理关联对象时，属性有时会引用接口或抽象类。在反序列化这些属性时，序列化器必须知道要初始化哪个具体类。这是使用*鉴别器类映射*来完成的。

假设有一个 `InvoiceItemInterface`，它由 `Product` 和 `Shipping` 对象实现。在序列化对象时，序列化器将添加一个额外的"鉴别器属性"。这包含 `product` 或 `shipping`。鉴别器类映射在反序列化时将这些类型名称映射到真实的 PHP 类名：

**PHP 属性方式：**

```php
namespace App\Model;

use Symfony\Component\Serializer\Attribute\DiscriminatorMap;

#[DiscriminatorMap(
    typeProperty: 'type',
    mapping: [
        'product' => Product::class,
        'shipping' => Shipping::class,
    ]
)]
interface InvoiceItemInterface
{
    // ...
}
```

**YAML 配置：**

```yaml
App\Model\InvoiceItemInterface:
    discriminator_map:
        type_property: type
        mapping:
            product: 'App\Model\Product'
            shipping: 'App\Model\Shipping'
```

**XML 配置：**

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<serializer xmlns="http://symfony.com/schema/dic/serializer-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/serializer-mapping
        https://symfony.com/schema/dic/serializer-mapping/serializer-mapping-1.0.xsd"
>
    <class name="App\Model\InvoiceItemInterface">
        <discriminator-map type-property="type">
            <mapping type="product" class="App\Model\Product"/>
            <mapping type="shipping" class="App\Model\Shipping"/>
        </discriminator-map>
    </class>
</serializer>
```

配置好鉴别器映射后，序列化器现在可以为类型为 `InvoiceItemInterface` 的属性选择正确的类：

**Symfony 框架方式：**

```php
class InvoiceLine
{
    public function __construct(
        private InvoiceItemInterface $invoiceItem
    ) {
        $this->invoiceItem = $invoiceItem;
    }

    public function getInvoiceItem(): InvoiceItemInterface
    {
        return $this->invoiceItem;
    }

    // ...
}

// ...
$invoiceLine = new InvoiceLine(new Product());

$jsonString = $serializer->serialize($invoiceLine, 'json');
// $jsonString contains {"type":"product",...}

$invoiceLine = $serializer->deserialize($jsonString, InvoiceLine::class, 'json');
// $invoiceLine contains new InvoiceLine(new Product(...))
```

**独立使用方式：**

```php
// ...
use Symfony\Component\Serializer\Mapping\ClassDiscriminatorFromClassMetadata;
use Symfony\Component\Serializer\Mapping\Factory\ClassMetadataFactory;
use Symfony\Component\Serializer\Mapping\Loader\AttributeLoader;
use Symfony\Component\Serializer\Normalizer\ObjectNormalizer;
use Symfony\Component\Serializer\Serializer;

class InvoiceLine
{
    public function __construct(
        private InvoiceItemInterface $invoiceItem
    ) {
        $this->invoiceItem = $invoiceItem;
    }

    public function getInvoiceItem(): InvoiceItemInterface
    {
        return $this->invoiceItem;
    }

    // ...
}

// ...

// Configure a loader to retrieve mapping information like DiscriminatorMap.
// E.g. when using PHP attributes:
$classMetadataFactory = new ClassMetadataFactory(new AttributeLoader());
$discriminator = new ClassDiscriminatorFromClassMetadata($classMetadataFactory);
$normalizers = [
    new ObjectNormalizer($classMetadataFactory, null, null, null, $discriminator),
];

$serializer = new Serializer($normalizers, $encoders);

$invoiceLine = new InvoiceLine(new Product());

$jsonString = $serializer->serialize($invoiceLine, 'json');
// $jsonString contains {"type":"product",...}

$invoiceLine = $serializer->deserialize($jsonString, InvoiceLine::class, 'json');
// $invoiceLine contains new InvoiceLine(new Product(...))
```

你可以添加默认类型，以避免在反序列化时需要添加 type 属性：

**PHP 属性方式：**

```php
namespace App\Model;

use Symfony\Component\Serializer\Attribute\DiscriminatorMap;

#[DiscriminatorMap(
    typeProperty: 'type',
    mapping: [
        'product' => Product::class,
        'shipping' => Shipping::class,
    ],
    defaultType: 'product',
)]
interface InvoiceItemInterface
{
    // ...
}
```

**YAML 配置：**

```yaml
App\Model\InvoiceItemInterface:
    discriminator_map:
        type_property: type
        mapping:
            product: 'App\Model\Product'
            shipping: 'App\Model\Shipping'
        default_type: product
```

**XML 配置：**

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<serializer xmlns="http://symfony.com/schema/dic/serializer-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/serializer-mapping
        https://symfony.com/schema/dic/serializer-mapping/serializer-mapping-1.0.xsd"
>
    <class name="App\Model\InvoiceItemInterface">
        <discriminator-map type-property="type" default-type="product">
            <mapping type="product" class="App\Model\Product"/>
            <mapping type="shipping" class="App\Model\Shipping"/>
        </discriminator-map>
    </class>
</serializer>
```

现在反序列化如下：

```php
// $jsonString does NOT contain "type" in "invoiceItem"
$invoiceLine = $serializer->deserialize('{"invoiceItem":{...},...}', InvoiceLine::class, 'json');
// $invoiceLine contains new InvoiceLine(new Product(...))
```

### 部分反序列化输入（解包）

序列化器始终将完整的输入字符串反序列化为 PHP 值。当连接第三方 API 时，你通常只需要返回响应的特定部分。

为了避免反序列化整个响应，你可以使用 `Symfony\Component\Serializer\Normalizer\UnwrappingDenormalizer` 来"解包"输入数据：

```php
$jsonData = '{"result":"success","data":{"person":{"name": "Jane Doe","age":57}}}';
$data = $serialiser->deserialize($jsonData, Object::class, 'json', [
    UnwrappingDenormalizer::UNWRAP_PATH => '[data][person]',
]);
// $data is Person(name: 'Jane Doe', age: 57)
```

`unwrap_path` 是 PropertyAccess 组件的[属性路径](https://symfony.com/doc/current/components/property_access.html#reading-from-arrays)，应用于反规范化的数组。

### 处理构造函数参数

如果类构造函数定义了参数（通常在[值对象](https://en.wikipedia.org/wiki/Value_object)中发生），序列化器会将参数名称与反序列化的属性进行匹配。如果缺少某些参数，则会抛出 `Symfony\Component\Serializer\Exception\MissingConstructorArgumentsException`。

在这些情况下，使用 `default_constructor_arguments` 上下文选项为缺少的参数定义默认值：

```php
use App\Model\Person;
use Symfony\Component\Serializer\Normalizer\AbstractNormalizer;
// ...

$jsonData = '{"age":39,"name":"Jane Doe"}';
$person = $serializer->deserialize($jsonData, Person::class, 'json', [
    AbstractNormalizer::DEFAULT_CONSTRUCTOR_ARGUMENTS => [
        Person::class => ['sportsperson' => true],
    ],
]);
// $person is Person(name: 'Jane Doe', age: 39, sportsperson: true);
```

### 递归反规范化和类型安全

当 `PropertyTypeExtractor` 可用时，规范化器还会检查要反规范化的数据是否与属性类型匹配（即使对于原始类型）。例如，如果提供了 `string`，但属性类型为 `int`，则会抛出 `Symfony\Component\Serializer\Exception\UnexpectedValueException`。可以通过将序列化器上下文选项 `ObjectNormalizer::DISABLE_TYPE_ENFORCEMENT` 设置为 `true` 来禁用属性的类型强制执行。

### 处理布尔值

PHP 将许多不同的值视为 true 或 false。例如，字符串 `true`、`1` 和 `yes` 被视为 true，而 `false`、`0` 和 `no` 被视为 false。

在反序列化时，序列化器组件可以自动处理这一问题。这可以通过使用 `AbstractNormalizer::FILTER_BOOL` 上下文选项来完成：

```php
use Symfony\Component\Serializer\Normalizer\AbstractNormalizer;
// ...

$person = $serializer->denormalize(['sportsperson' => 'yes'], Person::class, context: [
    AbstractNormalizer::FILTER_BOOL => true
]);
// $person contains a Person instance with sportsperson set to true
```

此上下文使反序列化过程的行为类似于带有 `FILTER_VALIDATE_BOOL` 标志的 `filter_var` 函数。

## 为类扩展序列化

有时你可能希望在无法修改的类上添加或覆盖序列化元数据，例如来自第三方库或供应商包的模型。传统上，你必须创建 YAML 或 XML 映射文件来配置这些类的序列化。`#[ExtendsSerializationFor]` 属性提供了一种更方便的替代方案。

假设你使用第三方 `Product` 类，并且希望在不修改原始类的情况下公开不同的序列化字段名或分组。

为此，创建一个单独的类，并使用 `#[ExtendsSerializationFor]` 属性告诉序列化器哪个类应接收此元数据。你的新类名无关紧要，通常将该类设为 `abstract`，以明确表示它永远不会被实例化：

```php
// src/Serializer/VendorProductExtension.php
namespace App\Serializer;

use Symfony\Component\Serializer\Attribute\ExtendsSerializationFor;
use Symfony\Component\Serializer\Attribute\Groups;
use Symfony\Component\Serializer\Attribute\MaxDepth;
use Symfony\Component\Serializer\Attribute\SerializedName;
use Vendor\Library\Product;

#[ExtendsSerializationFor(Product::class)]
abstract class MyProductSerialization
{
    #[Groups(['api'])]
    #[SerializedName('product_name')]
    public string $name = '';

    #[Groups(['api', 'admin'])]
    public float $price = 0;

    #[Groups(['admin'])]
    #[MaxDepth(1)]
    public $category;
}
```

此类中定义的序列化元数据将应用于目标类（`Product`），就好像它直接定义在该类上一样。

你只能为目标类上存在的属性定义元数据。否则，在容器编译期间会抛出 `MappingException`。

你可以在源类属性上使用任何序列化属性，包括 `#[Groups]`、`#[SerializedName]`、`#[MaxDepth]`、`#[Ignore]` 等。

### 编译时属性元数据

在使用带有[自动配置](https://symfony.com/doc/current/service_container.html#services-autoconfigure)的 Symfony 框架时，使用序列化器属性（如 `#[Groups]`、`#[SerializedName]`、`#[MaxDepth]`、`#[Ignore]`、`#[Context]`、`#[SerializedPath]` 或 `#[DiscriminatorMap]`）的类将在编译时自动发现。这使属性加载器只需处理已知具有序列化器属性的类，从而提高生产环境的性能。

如果你需要显式注册使用序列化器属性的类（例如来自不属于你的服务定义的第三方库），请使用 `serializer.attribute_metadata` 和 `container.excluded` 标签：

```yaml
# config/services.yaml
services:
    Vendor\Library\SomeModel:
        tags:
            - { name: container.excluded }
            - { name: serializer.attribute_metadata }
```

## 配置元数据缓存

序列化器的元数据会自动缓存以提高应用程序性能。默认情况下，序列化器使用 `cache.system` 缓存池，该缓存池使用 [`cache.system`](https://symfony.com/doc/current/reference/configuration/framework.html#reference-cache-system) 选项进行配置。

## 深入了解序列化器

更多相关内容请参阅：

* [序列化器编码器](https://symfony.com/doc/current/serializer/encoders.html)
* [自定义名称转换器](https://symfony.com/doc/current/serializer/custom_name_converter.html)
* [自定义上下文构建器](https://symfony.com/doc/current/serializer/custom_context_builders.html)
* [流式 JSON](https://symfony.com/doc/current/serializer/streaming_json.html)
