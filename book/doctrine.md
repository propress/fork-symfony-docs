# 数据库与 Doctrine ORM

> 更喜欢视频教程？请查看 [Doctrine 视频系列](https://symfonycasts.com/screencast/symfony-doctrine)。

Symfony 提供了你在应用中使用数据库所需的所有工具，这要归功于 [Doctrine](https://www.doctrine-project.org/)——这是一套用于处理数据库的最佳 PHP 库。这些工具支持关系型数据库（如 MySQL 和 PostgreSQL）以及 NoSQL 数据库（如 MongoDB）。

数据库是一个广泛的主题，因此文档分为三篇文章：

- 本文介绍在 Symfony 应用中处理**关系型数据库**的推荐方式；
- 如果你需要对关系型数据库执行原始 SQL 查询的**低级访问**（类似于 PHP 的 PDO），请阅读[这篇文章](doctrine/dbal.md)；
- 如果你使用 **MongoDB 数据库**，请阅读 [DoctrineMongoDBBundle 文档](https://symfony.com/doc/current/bundles/DoctrineMongoDBBundle/index.html)。

---

## 安装 Doctrine

首先，通过 `orm` [Symfony 包](setup.md#symfony-packs)安装 Doctrine 支持，以及帮助生成代码的 [MakerBundle](https://symfony.com/doc/current/bundles/SymfonyMakerBundle/index.html)：

```terminal
$ composer require symfony/orm-pack
$ composer require --dev symfony/maker-bundle
```

### 配置数据库

数据库连接信息存储为名为 `DATABASE_URL` 的环境变量。在开发时，你可以在 `.env` 中找到并自定义它：

```text
# .env（或在 .env.local 中覆盖 DATABASE_URL 以避免提交更改）

# 自定义这一行！
DATABASE_URL="mysql://db_user:db_password@127.0.0.1:3306/db_name?serverVersion=8.0.37"

# 使用 mariadb：
# DATABASE_URL="mysql://db_user:db_password@127.0.0.1:3306/db_name?serverVersion=10.5.8-MariaDB"

# 使用 sqlite：
# DATABASE_URL="sqlite:///%kernel.project_dir%/var/app.db"

# 使用 postgresql：
# DATABASE_URL="postgresql://db_user:db_password@127.0.0.1:5432/db_name?serverVersion=12.19 (Debian 12.19-1.pgdg120+1)&charset=utf8"

# 使用 oracle：
# DATABASE_URL="oci8://db_user:db_password@127.0.0.1:1521/db_name"
```

> **警告**
>
> 如果用户名、密码、主机或数据库名包含在 URI 中被视为特殊的字符（如 `: / ? # [ ] @ ! $ & ' ( ) * + , ; =`），你必须对其进行编码。有关保留字符的完整列表，请参阅 [RFC 3986](https://www.ietf.org/rfc/rfc3986.txt)。你可以使用 `urlencode` 函数对其进行编码，或使用 [urlencode 环境变量处理器](configuration/env_var_processors.md)。在这种情况下，你需要删除 `config/packages/doctrine.yaml` 中的 `resolve:` 前缀以避免错误：`url: '%env(DATABASE_URL)%'`

> **提示**
>
> 为了避免凭据中特殊字符导致的 URL 编码问题，你可以使用单独的连接参数代替 URL 格式。将每个值定义为自己的环境变量，并在 `.env` 文件中用单引号括起来，以防止 `$` 和 `#` 等字符被解释：
>
> ```text
> # .env
> DATABASE_PASSWORD='p@ss$wo#rd'
> ```
>
> 然后配置 Doctrine 使用各个参数：
>
> ```yaml
> # config/packages/doctrine.yaml
> doctrine:
>     dbal:
>         user:     '%env(DATABASE_USER)%'
>         password: '%env(DATABASE_PASSWORD)%'
>         host:     '%env(DATABASE_HOST)%'
>         port:     '%env(DATABASE_PORT)%'
>         dbname:   '%env(DATABASE_NAME)%'
>         driver:   pdo_mysql
> ```

现在你的连接参数已设置好，Doctrine 可以为你创建 `db_name` 数据库：

```terminal
$ php bin/console doctrine:database:create
```

`config/packages/doctrine.yaml` 中还有更多可以配置的选项，包括你的 `server_version`（例如，如果你使用 MySQL 8.0.37，则为 8.0.37），这可能会影响 Doctrine 的运行方式。

> **提示**
>
> 还有许多其他 Doctrine 命令。运行 `php bin/console list doctrine` 查看完整列表。

---

## 创建实体类 {#doctrine-adding-mapping}

假设你在构建一个需要显示产品的应用。不用考虑 Doctrine 或数据库，你已经知道需要一个 `Product` 对象来表示这些产品。

你可以使用 `make:entity` 命令创建此类以及你需要的任何字段。该命令会问你一些问题——按如下方式回答：

```bash
$ php bin/console make:entity

Class name of the entity to create or update:
> Product

New property name (press <return> to stop adding fields):
> name

Field type (enter ? to see all types) [string]:
> string

Field length [255]:
> 255

Can this field be null in the database (nullable) (yes/no) [no]:
> no

New property name (press <return> to stop adding fields):
> price

Field type (enter ? to see all types) [string]:
> integer

Can this field be null in the database (nullable) (yes/no) [no]:
> no

New property name (press <return> to stop adding fields):
>
（再次按 Enter 结束）
```

现在你有了一个新的 `src/Entity/Product.php` 文件：

```php
// src/Entity/Product.php
namespace App\Entity;

use App\Repository\ProductRepository;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: ProductRepository::class)]
class Product
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 255)]
    private ?string $name = null;

    #[ORM\Column]
    private ?int $price = null;

    public function getId(): ?int
    {
        return $this->id;
    }

    // ... getter 和 setter 方法
}
```

> **提示**
>
> 你可以向 `make:entity` 传递 `--with-uuid` 或 `--with-ulid`。利用 Symfony 的 [Uid 组件](components/uid.md)，这会生成一个以 [Uuid](components/uid.md#uuid) 或 [Ulid](components/uid.md#ulid) 类型（而非 `int`）作为 `id` 的实体。

> **注意**
>
> 不理解为什么价格是整数？别担心：这只是一个示例。但是，以整数存储价格（例如 100 = 1 美元）可以避免舍入问题。

这个类被称为"实体"。很快，你就能够将 Product 对象保存和查询到数据库中的 `product` 表。`Product` 实体中的每个属性都可以映射到该表中的一列。这通常通过属性完成：你在每个属性上方看到的 `#[ORM\Column(...)]` 注解。

`make:entity` 命令是一个让生活更轻松的工具。但这是*你的*代码：添加/删除字段、添加/删除方法或更新配置。

> **警告**
>
> 注意不要将 SQL 保留关键字用作表或列名（例如 `GROUP` 或 `USER`）。有关如何转义这些词的详细信息，请参阅 Doctrine 的[保留 SQL 关键字文档](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/basic-mapping.html#quoting-reserved-words)。或者，用类上方的 `#[ORM\Table(name: 'groups')]` 更改表名，或用 `name: 'group_name'` 选项配置列名。

### 实体字段类型

Doctrine 支持各种各样的**字段类型**（数字、字符串、枚举、二进制、日期、JSON 等），每种类型都有自己的选项。请查看 Doctrine 文档中的 [Doctrine 映射类型列表](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/basic-mapping.html#reference-mapping-types)。

Symfony 还提供以下**额外字段类型**：

#### uuid

**类：** `Symfony\Bridge\Doctrine\Types\UuidType`

将 [UUID](components/uid.md) 存储为原生 GUID 类型（如果可用），否则存储为 16 字节二进制：

```php
// src/Entity/Product.php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Bridge\Doctrine\Types\UuidType;
use Symfony\Component\Uid\Uuid;

#[ORM\Entity]
class Product
{
    #[ORM\Column(type: UuidType::NAME)]
    private Uuid $sku;

    // ...
}
```

#### ulid

**类：** `Symfony\Bridge\Doctrine\Types\UlidType`

将 [ULID](components/uid.md#ulid) 存储为原生 GUID 类型（如果可用），否则存储为 16 字节二进制：

```php
// src/Entity/Product.php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Bridge\Doctrine\Types\UlidType;
use Symfony\Component\Uid\Ulid;

#[ORM\Entity]
class Product
{
    #[ORM\Column(type: UlidType::NAME)]
    private Ulid $identifier;

    // ...
}
```

#### DatePoint 类型

这些类型允许存储来自 [Clock 组件](components/clock.md)的 `DatePoint` 对象。它们会自动与 `DatePoint` 对象相互转换：

| 类型 | 继承 Doctrine 类型 | 类 |
|------|-------------------|-----|
| `date_point` | `datetime_immutable` | `Symfony\Bridge\Doctrine\Types\DatePointType` |
| `day_point` | `date_immutable` | `Symfony\Bridge\Doctrine\Types\DayPointType` |
| `time_point` | `time_immutable` | `Symfony\Bridge\Doctrine\Types\TimePointType` |

示例用法：

```php
// src/Entity/Product.php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Clock\DatePoint;

#[ORM\Entity]
class Product
{
    // 使用 DatePoint 类型提示时，Symfony 自动检测 'date_point' 类型
    #[ORM\Column]
    private DatePoint $createdAt;

    // 你也可以显式设置类型
    #[ORM\Column(type: 'date_point')]
    private DatePoint $updatedAt;

    #[ORM\Column(type: 'day_point')]
    public DatePoint $releaseDate;

    #[ORM\Column(type: 'time_point')]
    public DatePoint $openingTime;

    // ...
}
```

---

## 迁移：创建数据库表/模式 {#doctrine-creating-the-database-tables-schema}

`Product` 类已完全配置，可以保存到 `product` 表中。如果你刚刚定义了此类，你的数据库实际上还没有 `product` 表。要添加它，可以利用已安装的 [DoctrineMigrationsBundle](https://github.com/doctrine/DoctrineMigrationsBundle)：

```terminal
$ php bin/console make:migration
```

> **提示**
>
> 向 `make:migration` 传递 `--formatted` 可生成整洁的迁移文件。

如果一切正常，你应该看到如下内容：

```text
SUCCESS!

Next: Review the new migration "migrations/Version20211116204726.php"
Then: Run the migration with php bin/console doctrine:migrations:migrate
```

打开这个文件，它包含更新数据库所需的 SQL！要运行该 SQL，执行你的迁移：

```terminal
$ php bin/console doctrine:migrations:migrate
```

此命令执行所有尚未针对你的数据库运行的迁移文件。你应该在部署到生产时运行此命令，以保持生产数据库最新。

---

## 迁移与添加更多字段 {#doctrine-add-more-fields}

但是，如果你需要向 `Product` 添加新字段属性（如 `description`）怎么办？你可以编辑类以添加新属性。但是，你也可以再次使用 `make:entity`：

```bash
$ php bin/console make:entity

Class name of the entity to create or update
> Product

New property name (press <return> to stop adding fields):
> description

Field type (enter ? to see all types) [string]:
> text

Can this field be null in the database (nullable) (yes/no) [no]:
> no

New property name (press <return> to stop adding fields):
>
（再次按 Enter 结束）
```

这会添加新的 `description` 属性以及 `getDescription()` 和 `setDescription()` 方法：

```diff
  // src/Entity/Product.php
  // ...
+  use Doctrine\DBAL\Types\Types;

  class Product
  {
      // ...

+     #[ORM\Column(type: Types::TEXT)]
+     private string $description;

      // getDescription() & setDescription() 也已添加
  }
```

新属性已映射，但 `product` 表中还不存在它。没问题！生成新迁移：

```terminal
$ php bin/console make:migration
```

这次，生成文件中的 SQL 将如下所示：

```sql
ALTER TABLE product ADD description LONGTEXT NOT NULL
```

迁移系统很*智能*。它将所有实体与当前数据库状态进行比较，并生成同步它们所需的 SQL！像之前一样，执行你的迁移：

```terminal
$ php bin/console doctrine:migrations:migrate
```

每次更改模式时，运行这两个命令来生成迁移，然后执行它。确保提交迁移文件并在部署时执行它们。

> **提示**
>
> 如果你喜欢手动添加新属性，`make:entity` 命令可以为你生成 getter 和 setter 方法：
>
> ```terminal
> $ php bin/console make:entity --regenerate
> ```
>
> 如果你进行了一些更改并想要重新生成*所有* getter/setter 方法，也传递 `--overwrite`。

---

## 将对象持久化到数据库

是时候将 `Product` 对象保存到数据库了！让我们创建一个新控制器来实验：

```terminal
$ php bin/console make:controller ProductController
```

在控制器中，你可以创建一个新的 `Product` 对象，在其上设置数据，然后保存它：

```php
// src/Controller/ProductController.php
namespace App\Controller;

// ...
use App\Entity\Product;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class ProductController extends AbstractController
{
    #[Route('/product', name: 'create_product')]
    public function createProduct(EntityManagerInterface $entityManager): Response
    {
        $product = new Product();
        $product->setName('Keyboard');
        $product->setPrice(1999);
        $product->setDescription('Ergonomic and stylish!');

        // 告诉 Doctrine 你想要（最终）保存 Product（还没有查询）
        $entityManager->persist($product);

        // 实际执行查询（即 INSERT 查询）
        $entityManager->flush();

        return new Response('Saved new product with id '.$product->getId());
    }
}
```

试一试：http://localhost:8000/product

恭喜！你刚刚在 `product` 表中创建了第一行。可以直接查询数据库来证明：

```terminal
$ php bin/console dbal:run-sql 'SELECT * FROM product'

# 在不使用 Powershell 的 Windows 系统上，运行此命令：
# php bin/console dbal:run-sql "SELECT * FROM product"
```

更仔细地看上面的示例：

- **第 13 行** `EntityManagerInterface $entityManager` 参数告诉 Symfony [注入 Entity Manager 服务](service_container.md#构造函数注入)到控制器方法中。此对象负责将对象保存到数据库以及从数据库获取对象。
- **第 15-18 行** 在这一部分，你像任何其他普通 PHP 对象一样实例化并使用 `$product` 对象。
- **第 21 行** `persist($product)` 调用告诉 Doctrine "管理" `$product` 对象。这**不会**导致向数据库发出查询。
- **第 24 行** 调用 `flush()` 方法时，Doctrine 会查看它管理的所有对象，看它们是否需要持久化到数据库。在此示例中，`$product` 对象的数据在数据库中不存在，因此实体管理器执行一个 `INSERT` 查询，在 `product` 表中创建一个新行。

> **注意**
>
> 如果 `flush()` 调用失败，将抛出 `Doctrine\ORM\ORMException` 异常。参阅[事务和并发](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/transactions-and-concurrency.html)。

无论你是在创建还是更新对象，工作流程总是相同的：Doctrine 足够智能，能够知道是否应该 INSERT 还是 UPDATE 你的实体。

---

## 验证对象 {#automatic_object_validation}

[Symfony 验证器](validation.md)可以重用 Doctrine 元数据来执行一些基本的验证任务。首先，添加或配置 [auto_mapping 选项](reference/configuration/validator.md)以定义哪些实体应该由 Symfony 自省以添加自动验证约束。

考虑以下控制器代码：

```php
// src/Controller/ProductController.php
namespace App\Controller;

use App\Entity\Product;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Validator\Validator\ValidatorInterface;
// ...

class ProductController extends AbstractController
{
    #[Route('/product', name: 'create_product')]
    public function createProduct(ValidatorInterface $validator): Response
    {
        $product = new Product();

        // ... 以某种方式更新产品数据（例如使用表单）...

        $errors = $validator->validate($product);
        if (count($errors) > 0) {
            return new Response((string) $errors, 400);
        }

        // ...
    }
}
```

尽管 `Product` 实体没有定义任何明确的[验证配置](validation.md)，但如果 `auto_mapping` 选项将其包含在要自省的实体列表中，Symfony 会为其推断一些验证规则并应用它们。

例如，给定 `name` 属性在数据库中不能为 `null`，一个 [NotNull 约束](reference/constraints/NotNull.md)会自动添加到该属性（如果它尚未包含该约束）。

下表总结了 Doctrine 元数据与 Symfony 自动添加的相应验证约束之间的映射：

| Doctrine 属性 | 验证约束 | 注意 |
|--------------|---------|------|
| `nullable=false` | NotNull | 需要安装 PropertyInfo 组件 |
| `type` | Type | 需要安装 PropertyInfo 组件 |
| `unique=true` | UniqueEntity | |
| `length` | Length | |

因为[表单组件](forms.md)以及 [API Platform](https://api-platform.com/docs/core/validation/) 在内部使用 Validator 组件，你所有的表单和 Web API 也将自动受益于这些自动验证约束。

这种自动验证是提高你生产力的一个很好的功能，但它并不完全取代验证配置。你仍然需要添加一些[验证约束](reference/constraints.md)以确保用户提供的数据是正确的。

---

## 从数据库获取对象

从数据库中获取对象甚至更简单。假设你想要访问 `/product/1` 来查看你的新产品：

```php
// src/Controller/ProductController.php
namespace App\Controller;

use App\Entity\Product;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
// ...

class ProductController extends AbstractController
{
    #[Route('/product/{id}', name: 'product_show')]
    public function show(EntityManagerInterface $entityManager, int $id): Response
    {
        $product = $entityManager->getRepository(Product::class)->find($id);

        if (!$product) {
            throw $this->createNotFoundException(
                'No product found for id '.$id
            );
        }

        return new Response('Check out this great product: '.$product->getName());

        // 或渲染模板
        // 在模板中，用 {{ product.name }} 打印内容
        // return $this->render('product/show.html.twig', ['product' => $product]);
    }
}
```

另一种可能是使用 Symfony 自动装配并由依赖注入容器注入的 `ProductRepository`：

```php
// src/Controller/ProductController.php
namespace App\Controller;

use App\Entity\Product;
use App\Repository\ProductRepository;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
// ...

class ProductController extends AbstractController
{
    #[Route('/product/{id}', name: 'product_show')]
    public function show(ProductRepository $productRepository, int $id): Response
    {
        $product = $productRepository->find($id);

        // ...
    }
}
```

试一试：http://localhost:8000/product/1

当你查询特定类型的对象时，你总是使用所谓的"仓库"。你可以将仓库视为一个 PHP 类，其唯一工作是帮助你获取特定类的实体。

一旦你有了仓库对象，你就有了许多辅助方法：

```php
$repository = $entityManager->getRepository(Product::class);

// 通过主键（通常是 "id"）查找单个 Product
$product = $repository->find($id);

// 通过名称查找单个 Product
$product = $repository->findOneBy(['name' => 'Keyboard']);
// 或通过名称和价格查找
$product = $repository->findOneBy([
    'name' => 'Keyboard',
    'price' => 1999,
]);

// 查找与名称匹配的多个 Product 对象，按价格排序
$products = $repository->findBy(
    ['name' => 'Keyboard'],
    ['price' => 'ASC']
);

// 查找*所有* Product 对象
$products = $repository->findAll();
```

你还可以添加*自定义*方法来进行更复杂的查询！在[查询对象：仓库](#doctrine-queries)部分会有更多介绍。

---

## 自动获取对象（EntityValueResolver） {#doctrine-entity-value-resolver}

在许多情况下，你可以使用 `EntityValueResolver` 自动为你执行查询！你可以将控制器简化为：

```php
// src/Controller/ProductController.php
namespace App\Controller;

use App\Entity\Product;
use App\Repository\ProductRepository;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
// ...

class ProductController extends AbstractController
{
    #[Route('/product/{id}')]
    public function show(Product $product): Response
    {
        // 使用 Product！
        // ...
    }
}
```

就这样！属性使用路由中的 `{id}` 通过 `id` 列查询 `Product`。如果未找到，则抛出 404 错误。

你可以通过使控制器参数可选来更改此行为。在这种情况下，不会自动抛出 404，你可以自由地处理缺少的实体：

```php
#[Route('/product/{id}')]
public function show(?Product $product): Response
{
    if (null === $product) {
        // 运行你自己的逻辑以返回自定义响应
    }

    // ...
}
```

### 自动获取

默认情况下，自动获取仅在你的路由包含 `{id}` 通配符时有效。解析器使用它通过 `find()` 方法按主键获取实体：

```php
// 执行 find($id) 查询以找到 $product 对象
#[Route('/product/{id}')]
public function show(Product $product): Response
{
    // ...
}
```

要通过其他属性获取实体，使用 `{param:argument}` 路由语法。这将路由参数映射到控制器参数，并告诉解析器使用该属性查询数据库：

```php
// 执行 findOneBy(['slug' => $slug]) 查询以找到 $product 对象
#[Route('/product/{slug:product}')]
public function show(Product $product): Response
{
    // ...
}
```

你还可以使用 `MapEntity` 属性为任何控制器参数显式配置映射，并使用 [MapEntity 选项](#mapentity-选项)控制 `EntityValueResolver` 的行为：

```php
// src/Controller/ProductController.php
namespace App\Controller;

use App\Entity\Product;
use Symfony\Bridge\Doctrine\Attribute\MapEntity;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
// ...

class ProductController extends AbstractController
{
    #[Route('/product/{slug}')]
    public function show(
        #[MapEntity(mapping: ['slug' => 'slug'])]
        Product $product
    ): Response {
        // 使用 Product！
        // ...
    }
}
```

### 通过表达式获取

如果自动获取不适合你的用例，你可以使用 [ExpressionLanguage 组件](components/expression_language.md)编写表达式：

```php
#[Route('/product/{product_id}')]
public function show(
    #[MapEntity(expr: 'repository.find(product_id)')]
    Product $product
): Response {
}
```

在表达式中，`repository` 变量将是实体的 Repository 类，任何路由通配符（如 `{product_id}`）都可以作为变量使用。

### 通过接口获取

假设你的 `Product` 类实现了一个名为 `ProductInterface` 的接口。如果你想将控制器与具体的实体实现解耦，可以通过其接口引用实体。

为此，首先配置 [resolve_target_entities 选项](doctrine/resolve_target_entity.md)。然后，你的控制器可以对接口进行类型提示，实体将自动解析：

```php
public function show(
    #[MapEntity]
    ProductInterface $product
): Response {
    // ...
}
```

### MapEntity 选项 {#mapentity-选项}

`MapEntity` 属性上有许多选项可用于控制行为：

**`id`**
如果配置了 `id` 选项且与路由参数匹配，则解析器将按主键查找：

```php
#[Route('/product/{product_id}')]
public function show(
    #[MapEntity(id: 'product_id')]
    Product $product
): Response {
}
```

**`mapping`**
配置与 `findOneBy()` 方法一起使用的属性和值：键是路由占位符名称，值是 Doctrine 属性名称：

```php
#[Route('/product/{category}/{slug}/comments/{comment_slug}')]
public function show(
    #[MapEntity(mapping: ['category' => 'category', 'slug' => 'slug'])]
    Product $product,
    #[MapEntity(mapping: ['comment_slug' => 'slug'])]
    Comment $comment
): Response {
}
```

**`stripNull`**
如果为 true，则当使用 `findOneBy()` 时，任何 `null` 值都不会用于查询。

**`objectManager`**
默认情况下，`EntityValueResolver` 使用*默认*对象管理器，但你可以对此进行配置：

```php
#[Route('/product/{id}')]
public function show(
    #[MapEntity(objectManager: 'foo')]
    Product $product
): Response {
}
```

**`evictCache`**
如果为 true，强制 Doctrine 始终从数据库而不是缓存获取实体。

**`disabled`**
如果为 true，`EntityValueResolver` 不会尝试替换该参数。

**`message`**
当存在 `NotFoundHttpException` 时显示的可选自定义消息，但**仅在开发环境中**（你在生产中不会看到此消息）：

```php
#[Route('/product/{product_id}')]
public function show(
    #[MapEntity(id: 'product_id', message: 'The product does not exist')]
    Product $product
): Response {
}
```

---

## 更新对象

从 Doctrine 获取对象后，你与任何 PHP 模型的交互方式相同：

```php
// src/Controller/ProductController.php
namespace App\Controller;

use App\Entity\Product;
use App\Repository\ProductRepository;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
// ...

class ProductController extends AbstractController
{
    #[Route('/product/edit/{id}', name: 'product_edit')]
    public function update(EntityManagerInterface $entityManager, int $id): Response
    {
        $product = $entityManager->getRepository(Product::class)->find($id);

        if (!$product) {
            throw $this->createNotFoundException(
                'No product found for id '.$id
            );
        }

        $product->setName('New product name!');
        $entityManager->flush();

        return $this->redirectToRoute('product_show', [
            'id' => $product->getId()
        ]);
    }
}
```

使用 Doctrine 编辑现有产品包括三个步骤：

1. 从 Doctrine 获取对象；
2. 修改对象；
3. 在实体管理器上调用 `flush()`。

你*可以*调用 `$entityManager->persist($product)`，但这没有必要：Doctrine 已经在"监视"你的对象的变化。

---

## 删除对象

删除对象非常相似，但需要调用实体管理器的 `remove()` 方法：

```php
$entityManager->remove($product);
$entityManager->flush();
```

正如你所期望的，`remove()` 方法通知 Doctrine 你想要从数据库中删除给定的对象。`DELETE` 查询实际上不会执行，直到调用 `flush()` 方法。

---

## 查询对象：仓库 {#doctrine-queries}

你已经看到仓库对象如何允许你运行基本查询而无需任何工作：

```php
// 从控制器内部
$repository = $entityManager->getRepository(Product::class);
$product = $repository->find($id);
```

但是，如果你需要更复杂的查询怎么办？当你使用 `make:entity` 生成实体时，命令*也*生成了一个 `ProductRepository` 类：

```php
// src/Repository/ProductRepository.php
namespace App\Repository;

use App\Entity\Product;
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\ManagerRegistry;

class ProductRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, Product::class);
    }
}
```

当你获取你的仓库（即 `->getRepository(Product::class)`）时，它实际上是*这个*对象的实例！这是因为在 `Product` 实体类顶部生成的 `repositoryClass` 配置。

假设你想查询所有价格大于某个值的 Product 对象。为此向你的仓库添加一个新方法：

```php
// src/Repository/ProductRepository.php

// ...
class ProductRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, Product::class);
    }

    /**
     * @return Product[]
     */
    public function findAllGreaterThanPrice(int $price): array
    {
        $entityManager = $this->getEntityManager();

        $query = $entityManager->createQuery(
            'SELECT p
            FROM App\Entity\Product p
            WHERE p.price > :price
            ORDER BY p.price ASC'
        )->setParameter('price', $price);

        // 返回 Product 对象的数组
        return $query->getResult();
    }
}
```

传递给 `createQuery()` 的字符串看起来像 SQL，但它是 [Doctrine 查询语言（DQL）](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/dql-doctrine-query-language.html)。这允许你使用常见的查询语言，但引用 PHP 对象而不是数据库表（即在 `FROM` 语句中）。

现在，你可以在仓库上调用此方法：

```php
// 从控制器内部
$minPrice = 1000;

$products = $entityManager->getRepository(Product::class)->findAllGreaterThanPrice($minPrice);

// ...
```

参阅[如何将仓库注入任何服务](service_container.md#构造函数注入)。

### 使用查询构建器查询

Doctrine 还提供了一个[查询构建器](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/query-builder.html)——一种面向对象的编写查询方式。建议在动态构建查询时使用（即基于 PHP 条件）：

```php
// src/Repository/ProductRepository.php

// ...
class ProductRepository extends ServiceEntityRepository
{
    public function findAllGreaterThanPrice(int $price, bool $includeUnavailableProducts = false): array
    {
        // 自动知道要选择 Products
        // "p" 是你在查询其余部分中使用的别名
        $qb = $this->createQueryBuilder('p')
            ->where('p.price > :price')
            ->setParameter('price', $price)
            ->orderBy('p.price', 'ASC');

        if (!$includeUnavailableProducts) {
            $qb->andWhere('p.available = TRUE');
        }

        $query = $qb->getQuery();

        return $query->execute();

        // 只获取一个结果：
        // $product = $query->setMaxResults(1)->getOneOrNullResult();
    }
}
```

### 使用 SQL 查询

此外，如果需要，你可以直接使用 SQL 查询：

```php
// src/Repository/ProductRepository.php

// ...
class ProductRepository extends ServiceEntityRepository
{
    public function findAllGreaterThanPrice(int $price): array
    {
        $conn = $this->getEntityManager()->getConnection();

        $sql = '
            SELECT * FROM product p
            WHERE p.price > :price
            ORDER BY p.price ASC
            ';

        $resultSet = $conn->executeQuery($sql, ['price' => $price]);

        // 返回数组的数组（即原始数据集）
        return $resultSet->fetchAllAssociative();
    }
}
```

使用 SQL，你将获得原始数据，而不是对象（除非你使用 [NativeQuery](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/native-sql.html) 功能）。

---

## 配置

参阅 [Doctrine 配置参考](reference/configuration/doctrine.md)。

---

## 关系和关联

Doctrine 提供了管理数据库关系（也称为关联）所需的所有功能，包括 ManyToOne、OneToMany、OneToOne 和 ManyToMany 关系。

更多信息，请参阅[关联](doctrine/associations.md)。

---

## 数据库测试

阅读关于[测试与数据库交互的代码](testing/database.md)的文章。

---

## Doctrine 扩展（Timestampable、Translatable 等）

Doctrine 社区创建了一些扩展来实现常见需求，例如*"创建实体时自动设置 createdAt 属性的值"*。阅读更多关于[可用 Doctrine 扩展](https://github.com/doctrine-extensions/DoctrineExtensions)的内容，并使用 [StofDoctrineExtensionsBundle](https://github.com/stof/StofDoctrineExtensionsBundle) 将它们集成到你的应用中。

---

## 延伸阅读

- [关联](doctrine/associations.md)
- [事件](doctrine/events.md)
- [自定义 DQL 函数](doctrine/custom_dql_functions.md)
- [DBAL](doctrine/dbal.md)
- [多实体管理器](doctrine/multiple_entity_managers.md)
- [解析目标实体](doctrine/resolve_target_entity.md)
- [测试数据库](testing/database.md)
