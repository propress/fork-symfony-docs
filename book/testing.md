# 测试

每当你写一行新代码，你也可能引入新的 bug。为了构建更好、更可靠的应用，你应该使用功能测试和单元测试来测试你的代码。

Symfony 与独立库 [PHPUnit](https://phpunit.de/) 集成，为你提供丰富的测试框架。本文介绍你编写 Symfony 测试所需的 PHPUnit 基础知识。要了解 PHPUnit 及其功能的所有内容，请阅读[官方 PHPUnit 文档](https://docs.phpunit.de/)。

## 测试类型

自动化测试有很多类型，精确的定义因项目而异。在 Symfony 中，使用以下定义。如果你学到了不同的内容，那不一定是错的，只是与 Symfony 文档所使用的不同。

**单元测试（Unit Tests）**
: 这些测试确保源代码的*单个*单元（例如单个类）按预期运行。

**集成测试（Integration Tests）**
: 这些测试测试类的组合，并通常与 Symfony 的服务容器交互。这些测试尚未覆盖完整运行的应用，那些被称为*应用测试*。

**应用测试（Application Tests）**
: 应用测试（也称为功能测试）测试完整应用的行为。它们发出 HTTP 请求（真实的和模拟的），并测试响应是否符合预期。

## 安装

在创建第一个测试之前，安装 `symfony/test-pack`，它会安装测试所需的其他包（如 `phpunit/phpunit`）：

```terminal
$ composer require --dev symfony/test-pack
```

安装库后，尝试运行 PHPUnit：

```terminal
$ php bin/phpunit
```

此命令自动运行你的应用测试。每个测试都是一个以"Test"结尾的 PHP 类（例如 `BlogControllerTest`），位于应用的 `tests/` 目录中。

PHPUnit 由应用根目录中的 `phpunit.dist.xml` 文件配置（在 PHPUnit 10 之前的版本中，文件名为 `phpunit.xml.dist`）。Symfony Flex 提供的默认配置在大多数情况下已经足够。阅读 [PHPUnit 文档](https://docs.phpunit.de/en/13.1/configuration.html)以了解所有可能的配置选项（例如启用代码覆盖率或将测试分成多个"测试套件"）。

> **注意：** [Symfony Flex](/setup#symfony-flex) 会自动创建 `phpunit.dist.xml` 和 `tests/bootstrap.php`。如果这些文件丢失，你可以尝试使用 `composer recipes:install phpunit/phpunit --force -v` 重新运行配方。

## 单元测试

[单元测试](https://en.wikipedia.org/wiki/Unit_testing)确保源代码的单个单元（例如单个类或某个类中的特定方法）满足其设计并按预期运行。在 Symfony 应用中编写单元测试与编写标准 PHPUnit 单元测试没有区别。你可以在 PHPUnit 文档中了解相关内容：[为 PHPUnit 编写测试](https://docs.phpunit.de/en/13.1/writing-tests-for-phpunit.html)。

按照惯例，`tests/` 目录应该复制应用的目录结构。因此，如果你正在测试 `src/Form/` 目录中的类，请将测试放在 `tests/Form/` 目录中。通过 `vendor/autoload.php` 文件自动启用自动加载（在 `phpunit.dist.xml` 文件中默认配置）。

你可以使用 `bin/phpunit` 命令运行测试：

```terminal
# 运行应用的所有测试
$ php bin/phpunit

# 运行 Form/ 目录中的所有测试
$ php bin/phpunit tests/Form

# 运行 UserType 类的测试
$ php bin/phpunit tests/Form/UserTypeTest.php
```

> **提示：** 在大型测试套件中，为每种类型的测试创建子目录（`tests/Unit/`、`tests/Integration/`、`tests/Application/` 等）是有意义的。

## 集成测试

与单元测试相比，集成测试将测试应用更大的部分（例如服务的组合）。集成测试可能需要使用 Symfony Kernel 从依赖注入容器中获取服务。

Symfony 提供了 `Symfony\Bundle\FrameworkBundle\Test\KernelTestCase` 类，帮助你在测试中使用 `bootKernel()` 创建和启动内核：

```php
// tests/Service/NewsletterGeneratorTest.php
namespace App\Tests\Service;

use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class NewsletterGeneratorTest extends KernelTestCase
{
    public function testSomething(): void
    {
        self::bootKernel();

        // ...
    }
}
```

`KernelTestCase` 还确保你的内核在每次测试时重新启动。这保证了每个测试都独立于彼此运行。

要运行你的应用测试，`KernelTestCase` 类需要找到应用内核来进行初始化。内核类通常在 `KERNEL_CLASS` 环境变量中定义（包含在 Symfony Flex 提供的默认 `.env.test` 文件中）：

```env
# .env.test
KERNEL_CLASS=App\Kernel
```

> **注意：** 如果你的用例更复杂，你也可以覆盖功能测试的 `getKernelClass()` 或 `createKernel()` 方法，这优先于 `KERNEL_CLASS` 环境变量。

### 设置测试环境

测试会创建一个在 `test` [环境](/configuration#configuration-environments)中运行的内核。这允许你在 `config/packages/test/` 中或使用 `when@test` 键为测试提供特殊设置。

如果你安装了 Symfony Flex，一些已安装的包已经配置了一些有用的测试配置。例如，默认情况下，Twig bundle 被配置为特别严格，以便在将代码部署到生产环境之前捕获错误：

```yaml
# config/packages/twig.yaml
when@test:
    twig:
        strict_variables: true
```

```php
// config/packages/twig.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'when@test' => [
        'twig' => [
            'strict_variables' => true,
        ],
    ],
]);
```

你也可以使用完全不同的环境，或通过将每个选项作为 `bootKernel()` 方法的选项来覆盖默认调试模式（`true`）：

```php
self::bootKernel([
    'environment' => 'my_test_env',
    'debug'       => false,
]);
```

> **提示：** 建议在 CI 服务器上将 `debug` 设置为 `false` 运行测试，因为这可以显著提高测试性能。这会禁用清除缓存。如果你的测试每次都不在干净的环境中运行，你必须手动清除它，例如在 `tests/bootstrap.php` 中使用此代码：
>
> ```php
> // ...
>
> // 当调试模式被禁用时确保缓存是新的
> new \Symfony\Component\Filesystem\Filesystem()->remove(__DIR__.'/../var/cache/test');
> ```

### 自定义环境变量

如果你需要为测试自定义某些环境变量（例如 Doctrine 使用的 `DATABASE_URL`），你可以通过在 `.env.test` 文件中覆盖任何你需要的内容来实现：

```env
# .env.test

# ...
DATABASE_URL="mysql://db_user:db_password@127.0.0.1:3306/db_name_test?serverVersion=8.0.37"
```

在测试环境中，这些 env 文件会被读取（如果变量在其中重复，列表中靠后的文件会覆盖之前的项目）：

1. `.env`：包含具有应用默认值的环境变量；
2. `.env.test`：覆盖/设置特定的测试值或变量；
3. `.env.test.local`：覆盖特定于此机器的设置。

> **警告：** `.env.local` 文件**不**在测试环境中使用，以使每个测试设置尽可能一致。

### 在测试中获取服务

在集成测试中，你通常需要从服务容器中获取服务来调用特定方法。启动内核后，容器由 `static::getContainer()` 返回：

```php
// tests/Service/NewsletterGeneratorTest.php
namespace App\Tests\Service;

use App\Service\NewsletterGenerator;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class NewsletterGeneratorTest extends KernelTestCase
{
    public function testSomething(): void
    {
        // (1) 启动 Symfony 内核
        self::bootKernel();

        // (2) 使用 static::getContainer() 访问服务容器
        $container = static::getContainer();

        // (3) 运行某个服务并测试结果
        $newsletterGenerator = $container->get(NewsletterGenerator::class);
        $newsletter = $newsletterGenerator->generateMonthlyNews(/* ... */);

        $this->assertEquals('...', $newsletter->getContent());
    }
}
```

`static::getContainer()` 返回的容器实际上是一个特殊的测试容器。它允许你访问公共服务和未被移除的[私有服务](/service_container#container-public)。

> **注意：** 如果你需要测试已被移除的私有服务（那些没有被任何其他服务使用的服务），你需要在 `config/services_test.yaml` 文件中将这些私有服务声明为公共的。

## 模拟依赖

有时，模拟被测服务的依赖项很有用。从上一节的示例出发，假设 `NewsletterGenerator` 依赖于一个指向私有 `NewsRepository` 服务的私有别名 `NewsRepositoryInterface`，你想使用模拟的 `NewsRepositoryInterface` 而不是具体的实现：

```php
// ...
use App\Contracts\Repository\NewsRepositoryInterface;

class NewsletterGeneratorTest extends KernelTestCase
{
    public function testSomething(): void
    {
        // ... 与上一节相同的引导程序

        $newsRepository = $this->createMock(NewsRepositoryInterface::class);
        $newsRepository->expects(self::once())
            ->method('findNewsFromLastMonth')
            ->willReturn([
                new News('some news'),
                new News('some other news'),
            ])
        ;

        $container->set(NewsRepositoryInterface::class, $newsRepository);

        // 将注入模拟的仓库
        $newsletterGenerator = $container->get(NewsletterGenerator::class);

        // ...
    }
}
```

不需要额外配置，因为测试服务容器是一个特殊的容器，允许你与私有服务和别名交互。

### 为测试配置数据库

与数据库交互的测试应该使用各自独立的数据库，以免与其他[配置环境](/configuration#configuration-environments)中使用的数据库混淆。

为此，编辑或创建项目根目录下的 `.env.test.local` 文件，并为 `DATABASE_URL` 环境变量定义新值：

```env
# .env.test.local
DATABASE_URL="mysql://USERNAME:PASSWORD@127.0.0.1:3306/DB_NAME?serverVersion=8.0.37"
```

这假设每个开发者/机器对测试使用不同的数据库。如果每台机器上的测试设置相同，请使用 `.env.test` 文件并将其提交到共享仓库。了解更多关于[在 Symfony 应用中使用多个 .env 文件](/configuration#configuration-multiple-env-files)的内容。

之后，你可以使用以下命令创建测试数据库和所有表：

```terminal
# 创建测试数据库
$ php bin/console --env=test doctrine:database:create

# 在测试数据库中创建表/列
$ php bin/console --env=test doctrine:schema:create
```

> **提示：** 你可以在[测试引导过程](/testing/bootstrap)中运行这些命令来创建数据库。

> **提示：** 一个常见的做法是在测试中为原始数据库名称追加 `_test` 后缀。如果生产中的数据库名称为 `project_acme`，测试数据库的名称可以是 `project_acme_test`。

#### 在每次测试前自动重置数据库

测试之间应该相互独立，以避免副作用。例如，如果某个测试修改了数据库（通过添加或删除实体），它可能会改变其他测试的结果。

[DAMADoctrineTestBundle](https://github.com/dmaicher/doctrine-test-bundle) 使用 Doctrine 事务让每个测试与未修改的数据库进行交互。使用以下命令安装它：

```terminal
$ composer require --dev dama/doctrine-test-bundle
```

现在，将其作为 PHPUnit 扩展启用：

```xml
<!-- phpunit.dist.xml -->
<phpunit>
    <!-- ... -->

    <extensions>
        <!-- 用于 PHPUnit 10 或更新版本 -->
        <bootstrap class="DAMA\DoctrineTestBundle\PHPUnit\PHPUnitExtension"/>
        <!-- 用于早于 10 的旧版 PHPUnit -->
        <extension class="DAMA\DoctrineTestBundle\PHPUnit\PHPUnitExtension"/>
    </extensions>
</phpunit>
```

就是这样！该 bundle 使用了一个巧妙的技巧：它在每次测试之前开始一个数据库事务，并在测试完成后自动回滚以撤销所有更改。在 [DAMADoctrineTestBundle](https://github.com/dmaicher/doctrine-test-bundle) 的文档中阅读更多内容。

#### 加载测试数据夹具

不使用生产数据库中的真实数据，而是在测试数据库中使用假数据或测试数据是很常见的做法。这通常称为*"夹具数据（fixtures data）"*，Doctrine 提供了一个库来创建和加载它们。使用以下命令安装：

```terminal
$ composer require --dev doctrine/doctrine-fixtures-bundle
```

然后，使用 [SymfonyMakerBundle](https://symfony.com/doc/current/bundles/SymfonyMakerBundle/index.html) 的 `make:fixtures` 命令生成一个空的夹具类：

```terminal
$ php bin/console make:fixtures

The class name of the fixtures to create (e.g. AppFixtures):
> ProductFixture
```

然后修改并使用该类在数据库中加载新实体。例如，要将 `Product` 对象加载到 Doctrine 中，使用：

```php
// src/DataFixtures/ProductFixture.php
namespace App\DataFixtures;

use App\Entity\Product;
use Doctrine\Bundle\FixturesBundle\Fixture;
use Doctrine\Persistence\ObjectManager;

class ProductFixture extends Fixture
{
    public function load(ObjectManager $manager): void
    {
        $product = new Product();
        $product->setName('Priceless widget');
        $product->setPrice(14.50);
        $product->setDescription('Ok, I guess it *does* have a price');
        $manager->persist($product);

        // 添加更多产品

        $manager->flush();
    }
}
```

清空数据库并使用以下命令重新加载*所有*夹具类：

```terminal
$ php bin/console --env=test doctrine:fixtures:load
```

更多信息，请阅读 [DoctrineFixturesBundle 文档](https://symfony.com/doc/current/bundles/DoctrineFixturesBundle/index.html)。

## 应用测试

应用测试检查应用所有不同层次的集成（从路由到视图）。就 PHPUnit 而言，它们与单元测试或集成测试没有区别，但有一个非常特定的工作流程：

1. [发起请求](#making-requests)；
2. [与页面交互](#interacting-with-the-response)（例如点击链接或提交表单）；
3. [测试响应](#test-assertions)；
4. 重复以上步骤。

> **注意：** 本节中使用的工具可以通过 `symfony/test-pack` 安装，如果你还没有安装，请使用 `composer require symfony/test-pack`。

### 编写第一个应用测试

应用测试是通常位于应用 `tests/Controller/` 目录中的 PHP 文件。它们通常继承 `Symfony\Bundle\FrameworkBundle\Test\WebTestCase`。此类在 `KernelTestCase` 之上添加了特殊逻辑。

如果你想测试由 `PostController` 类处理的页面，请先使用 [SymfonyMakerBundle](https://symfony.com/doc/current/bundles/SymfonyMakerBundle/index.html) 的 `make:test` 命令创建一个新的 `PostControllerTest`：

```terminal
$ php bin/console make:test

 Which test type would you like?:
 > WebTestCase

 The name of the test class (e.g. BlogPostTest):
 > Controller\PostControllerTest
```

这会创建以下测试类：

```php
// tests/Controller/PostControllerTest.php
namespace App\Tests\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class PostControllerTest extends WebTestCase
{
    public function testSomething(): void
    {
        // 这会调用 KernelTestCase::bootKernel()，并创建一个
        // 充当浏览器的"客户端"
        $client = static::createClient();

        // 请求特定页面
        $crawler = $client->request('GET', '/');

        // 验证成功响应和一些内容
        $this->assertResponseIsSuccessful();
        $this->assertSelectorTextContains('h1', 'Hello World');
    }
}
```

在上面的示例中，测试验证 HTTP 响应是否成功，以及请求体是否包含带有 `"Hello world"` 的 `<h1>` 标签。

`request()` 方法还返回一个 crawler，你可以使用它在测试中创建更复杂的断言（例如计算匹配给定 CSS 选择器的页面元素数量）：

```php
$crawler = $client->request('GET', '/post/hello-world');
$this->assertCount(4, $crawler->filter('.comment'));
```

你可以在 [/testing/dom_crawler](/testing/dom_crawler) 中了解更多关于 crawler 的内容。

### 发起请求

测试客户端模拟像浏览器一样的 HTTP 客户端，并向 Symfony 应用发出请求：

```php
$crawler = $client->request('GET', '/post/hello-world');
```

`request()` 方法以 HTTP 方法和 URL 作为参数，并返回一个 `Crawler` 实例。

> **提示：** 在应用测试中硬编码请求 URL 是最佳实践。如果测试使用 Symfony 路由器生成 URL，它将不会检测到对可能影响最终用户的应用 URL 所做的任何更改。

`request()` 方法的完整签名是：

```php
public function request(
    string $method,
    string $uri,
    array $parameters = [],
    array $files = [],
    array $server = [],
    ?string $content = null,
    bool $changeHistory = true
): Crawler
```

#### 在一个测试中发起多个请求

发起请求后，后续请求将使客户端重新启动内核。这会从头开始重新创建容器，以确保请求是隔离的，并且每次都使用新的服务对象。此行为可能会有一些意外后果：例如，安全令牌将被清除，Doctrine 实体将被分离等。

首先，你可以调用客户端的 `disableReboot()` 方法来重置内核而不是重新启动它。实际上，Symfony 将调用每个标记有 `kernel.reset` 的服务的 `reset()` 方法。但是，这**也**会清除安全令牌、分离 Doctrine 实体等。

为了解决这个问题，创建一个[编译器通道](/service_container/compiler_passes)来在你的测试环境中从某些服务中删除 `kernel.reset` 标签：

```php
// src/Kernel.php
namespace App;

use Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait;
use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\HttpKernel\Kernel as BaseKernel;

class Kernel extends BaseKernel implements CompilerPassInterface
{
    use MicroKernelTrait;

    // ...

    public function process(ContainerBuilder $container): void
    {
        if ('test' === $this->environment) {
            // 防止安全令牌被清除
            $container->getDefinition('security.token_storage')->clearTag('kernel.reset');

            // 防止 Doctrine 实体被分离
            $container->getDefinition('doctrine')->clearTag('kernel.reset');

            // ...
        }
    }
}
```

#### 浏览站点

客户端支持在真实浏览器中可以完成的许多操作：

```php
$client->back();
$client->forward();
$client->reload();

// 清除所有 Cookie 和历史记录
$client->restart();
```

> **注意：** `back()` 和 `forward()` 方法会跳过请求 URL 时可能发生的重定向，就像普通浏览器一样。

#### 重定向

当请求返回重定向响应时，客户端不会自动跟随它。你可以检查响应，然后使用 `followRedirect()` 方法强制重定向：

```php
$crawler = $client->followRedirect();
```

如果你希望客户端自动跟随所有重定向，可以在执行请求之前调用 `followRedirects()` 方法来强制执行：

```php
$client->followRedirects();
```

如果你向 `followRedirects()` 方法传入 `false`，重定向将不再被跟随：

```php
$client->followRedirects(false);
```

#### 登录用户（身份验证）

当你想要为受保护页面添加应用测试时，你必须首先以用户身份"登录"。重现实际步骤（如提交登录表单）会使测试非常缓慢。因此，Symfony 提供了 `loginUser()` 方法来在功能测试中模拟登录。

建议不要使用真实用户登录，而是创建一个仅用于测试的用户。你可以使用 [Doctrine 数据夹具](https://symfony.com/doc/current/bundles/DoctrineFixturesBundle/index.html)仅在测试数据库中加载测试用户。

在数据库中加载用户后，使用用户仓库获取该用户，并使用 `$client->loginUser()` 来模拟登录请求：

```php
// tests/Controller/ProfileControllerTest.php
namespace App\Tests\Controller;

use App\Repository\UserRepository;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class ProfileControllerTest extends WebTestCase
{
    // ...

    public function testVisitingWhileLoggedIn(): void
    {
        $client = static::createClient();
        $userRepository = static::getContainer()->get(UserRepository::class);

        // 获取测试用户
        $testUser = $userRepository->findOneByEmail('john.doe@example.com');

        // 模拟 $testUser 已登录
        $client->loginUser($testUser);

        // 测试例如个人资料页面
        $client->request('GET', '/profile');
        $this->assertResponseIsSuccessful();
        $this->assertSelectorTextContains('h1', 'Hello John!');
    }
}
```

你也可以在测试中使用[内存用户](/security#security-memory-user-provider)，直接实例化 `Symfony\Component\Security\Core\User\InMemoryUser`：

```php
// tests/Controller/ProfileControllerTest.php
use Symfony\Component\Security\Core\User\InMemoryUser;

$client = static::createClient();
$testUser = new InMemoryUser('admin', 'password', ['ROLE_ADMIN']);
$client->loginUser($testUser);
```

在执行此操作之前，你必须在测试环境配置中定义内存用户，以确保它存在并可以通过身份验证：

```yaml
# config/packages/security.yaml
when@test:
    security:
        providers:
            users_in_memory:
                memory:
                    users:
                        admin: { password: password, roles: ROLE_ADMIN }
```

要设置特定的防火墙（默认设置为 `main`）：

```php
$client->loginUser($testUser, 'my_firewall');
```

> **注意：** 根据设计，`loginUser()` 方法在使用无状态防火墙时不起作用。请在每个 `request()` 调用中添加适当的令牌/头。

#### 发起 AJAX 请求

客户端提供了一个 `xmlHttpRequest()` 方法，它与 `request()` 方法具有相同的参数，是发起 AJAX 请求的快捷方式：

```php
// 所需的 HTTP_X_REQUESTED_WITH 头会自动添加
$client->xmlHttpRequest('POST', '/submit', ['name' => 'Fabien']);
```

#### 发送自定义 HTTP 头

如果你的应用根据某些 HTTP 头的行为，将它们作为 `createClient()` 的第二个参数传递：

```php
$client = static::createClient([], [
    'HTTP_HOST'       => 'en.example.com',
    'HTTP_USER_AGENT' => 'MySuperBrowser/1.0',
]);
```

你也可以按请求覆盖 HTTP 头：

```php
$client->request('GET', '/', [], [], [
    'HTTP_HOST'       => 'en.example.com',
    'HTTP_USER_AGENT' => 'MySuperBrowser/1.0',
]);
```

> **警告：** 你的自定义头的名称必须遵循 [RFC 3875 第 4.1.18 节](https://tools.ietf.org/html/rfc3875#section-4.1.18)中定义的语法：将 `-` 替换为 `_`，转换为大写，并在结果前加 `HTTP_`。例如，如果你的头名称是 `X-Session-Token`，请传递 `HTTP_X_SESSION_TOKEN`。

### 与响应交互

像真实浏览器一样，Client 和 Crawler 对象可以用于与所提供的页面进行交互。

#### 点击链接

使用 `clickLink()` 方法点击包含给定文本（或具有该 `alt` 属性的第一个可点击图像）的第一个链接：

```php
$client = static::createClient();
$client->request('GET', '/post/hello-world');

$client->clickLink('Click here');
```

如果你需要访问提供特定于链接的辅助方法（如 `getMethod()` 和 `getUri()`）的 `Symfony\Component\DomCrawler\Link` 对象，请使用 `Crawler::selectLink()` 方法：

```php
$client = static::createClient();
$crawler = $client->request('GET', '/post/hello-world');

$link = $crawler->selectLink('Click here')->link();
// ...

// 如果你想点击选定的链接，请使用 click()
$client->click($link);
```

#### 提交表单

使用 `submitForm()` 方法提交包含给定按钮的表单：

```php
$client = static::createClient();
$client->request('GET', '/post/hello-world');

$crawler = $client->submitForm('Add comment', [
    'comment_form[content]' => '...',
]);
```

`submitForm()` 的第一个参数是表单中任何 `<button>` 或 `<input type="submit">` 的文本内容、`id` 或 `name`。第二个可选参数用于覆盖默认表单字段值。

根据表单类型，你可以使用不同的方法来填写输入：

```php
// 选择一个选项或单选按钮
$form['my_form[country]']->select('France');

// 勾选复选框
$form['my_form[like_symfony]']->tick();

// 上传文件
$form['my_form[photo]']->upload('/path/to/lucas.jpg');

// 多文件上传的情况
$form['my_form[field][0]']->upload('/path/to/lucas.jpg');
$form['my_form[field][1]']->upload('/path/to/lisa.jpg');
```

### 端到端测试（E2E）

如果你需要测试整个应用，包括其 JavaScript 代码，你可以使用真实浏览器而不是测试客户端。这称为**端到端测试**，它是测试应用的有效方式。

你可以使用 Panther 组件实现这一点。了解更多关于 [Symfony 中的 E2E 测试](/testing/end_to_end)的内容。

## Symfony 定义的测试断言

如果你的测试基于 PHPUnit，你可以在测试中使用任何 [PHPUnit 断言](https://docs.phpunit.de/en/13.1/assertions.html)。Symfony 还提供了许多额外的断言。

### 响应断言

`assertResponseIsSuccessful(string $message = '', ?bool $verbose = null)`
: 断言响应是成功的（HTTP 状态码为 2xx）。

`assertResponseStatusCodeSame(int $expectedCode, string $message = '', ?bool $verbose = null)`
: 断言特定的 HTTP 状态码。

`assertResponseRedirects(?string $expectedLocation = null, ?int $expectedCode = null, string $message = '', ?bool $verbose = null)`
: 断言响应是重定向响应（可选地，你可以检查目标位置和状态码）。预期位置可以是绝对路径或相对路径。

`assertResponseHasHeader(string $headerName, string $message = '')` / `assertResponseNotHasHeader(string $headerName, string $message = '')`
: 断言给定的头（不）在响应中可用，例如 `assertResponseHasHeader('content-type')`。

`assertResponseHeaderSame(string $headerName, string $expectedValue, string $message = '')` / `assertResponseHeaderNotSame(string $headerName, string $expectedValue, string $message = '')`
: 断言给定的头在响应中（不）包含预期值，例如 `assertResponseHeaderSame('content-type', 'application/octet-stream')`。

`assertResponseHasCookie(string $name, string $path = '/', ?string $domain = null, string $message = '')` / `assertResponseNotHasCookie(...)`
: 断言给定的 Cookie 在响应中存在（可选地检查特定的 Cookie 路径或域）。

`assertResponseCookieValueSame(string $name, string $expectedValue, string $path = '/', ?string $domain = null, string $message = '')`
: 断言给定的 Cookie 存在并设置为预期值。

`assertResponseIsUnprocessable(string $message = '', bool ?$verbose = null)`
: 断言响应是不可处理的（HTTP 状态码为 422）。

### 请求断言

`assertRequestAttributeValueSame(string $name, string $expectedValue, string $message = '')`
: 断言给定的请求属性被设置为预期值。

`assertRouteSame($expectedRoute, array $parameters = [], string $message = '')`
: 断言请求匹配给定的路由和可选的路由参数。

### 浏览器断言

`assertBrowserHasCookie(string $name, ...)` / `assertBrowserNotHasCookie(...)`
: 断言测试客户端（不）设置了给定的 Cookie（即该 Cookie 由测试中的任何响应设置）。

`assertBrowserCookieValueSame(string $name, string $expectedValue, ...)`
: 断言测试客户端中的给定 Cookie 被设置为预期值。

### Crawler 断言

`assertSelectorExists(string $selector, ...)` / `assertSelectorNotExists(...)`
: 断言给定的选择器（不）匹配响应中至少一个元素。

`assertSelectorCount(int $expectedCount, string $selector, string $message = '')`
: 断言响应中预期数量的选择器元素。

`assertSelectorTextContains(string $selector, string $text, ...)` / `assertSelectorTextNotContains(...)`
: 断言匹配给定选择器的第一个元素（不）包含预期文本。

`assertPageTitleSame(string $expectedTitle, string $message = '')`
: 断言 `<title>` 元素等于给定标题。

`assertPageTitleContains(string $expectedTitle, string $message = '')`
: 断言 `<title>` 元素包含给定标题。

`assertInputValueSame(string $fieldName, string $expectedValue, ...)` / `assertInputValueNotSame(...)`
: 断言具有给定名称的表单输入的值（不）等于预期值。

`assertCheckboxChecked(string $fieldName, ...)` / `assertCheckboxNotChecked(...)`
: 断言具有给定名称的复选框（未）被选中。

### 邮件断言

`assertEmailCount(int $count, ?string $transport = null, string $message = '')`
: 断言发送了预期数量的邮件。

`assertEmailTextBodyContains(RawMessage $email, string $text, ...)` / `assertEmailTextBodyNotContains(...)`
: 断言给定邮件的文本正文（不）包含预期文本。

`assertEmailHtmlBodyContains(RawMessage $email, string $text, ...)` / `assertEmailHtmlBodyNotContains(...)`
: 断言给定邮件的 HTML 正文（不）包含预期文本。

`assertEmailHasHeader(RawMessage $email, string $headerName, ...)` / `assertEmailNotHasHeader(...)`
: 断言给定邮件（不）设置了预期的头。

`assertEmailSubjectContains(RawMessage $email, string $expectedValue, ...)` / `assertEmailSubjectNotContains(...)`
: 断言给定邮件的主题（不）包含预期主题。

### 通知断言

`assertNotificationCount(int $count, ?string $transportName = null, string $message = '')`
: 断言已创建给定数量的通知（总计或对于给定传输）。

`assertNotificationSubjectContains(MessageInterface $notification, string $text, string $message = '')`
: 断言给定文本包含在给定通知的主题中。

### HttpClient 断言

> **提示：** 对于以下所有断言，必须在将触发 HTTP 请求的代码之前调用 `$client->enableProfiler()`。

`assertHttpClientRequest(string $expectedUrl, string $expectedMethod = 'GET', ...)`
: 断言给定的 URL 已被调用，如果指定了方法、正文和头。

`assertNotHttpClientRequest(string $unexpectedUrl, string $expectedMethod = 'GET', ...)`
: 断言给定的 URL 未使用 GET 或指定方法被调用。

`assertHttpClientRequestCount(int $count, string $httpClientId = 'http_client')`
: 断言在 HttpClient 上已发出给定数量的请求。

## 深入学习

* [testing/*](/testing/)
* [/components/dom_crawler](/components/dom_crawler)
* [/components/css_selector](/components/css_selector)
