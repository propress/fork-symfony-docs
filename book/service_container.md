# 服务容器

> **提示：**
> 更喜欢视频教程？请查看 [Symfony Fundamentals 视频系列](https://symfonycasts.com/screencast/symfony-fundamentals)。

你的应用程序中充满了有用的对象：一个"Mailer"对象可以帮助你发送电子邮件，另一个对象可以帮助你将数据保存到数据库。应用程序"做"的几乎所有事情，实际上都是由这些对象之一完成的。每次安装新的 Bundle 时，你都可以访问更多对象！

在 Symfony 中，这些有用的对象被称为**服务**，每个服务都存在于一个非常特殊的对象中，称为**服务容器**。容器允许你集中管理对象的构建方式。它让你的开发工作更轻松，促进了良好的架构，而且速度超快！

## 获取和使用服务

当你启动一个 Symfony 应用程序时，你的容器*已经*包含了许多服务。这些服务就像*工具*一样：等待你去使用它们。在你的控制器中，你可以通过将参数的类型提示设置为服务的类或接口名称，从容器中"请求"一个服务。想要记录日志吗？没问题：

```php
// src/Controller/ProductController.php
namespace App\Controller;

use Psr\Log\LoggerInterface;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class ProductController extends AbstractController
{
    #[Route('/products')]
    public function list(LoggerInterface $logger): Response
    {
        $logger->info('Look, I just used a service!');

        // ...
    }
}
```

还有哪些其他服务可用？运行以下命令查看：

```bash
$ php bin/console debug:autowiring

  # this is just a *small* sample of the output...

  Autowirable Types
  =================

   The following classes & interfaces can be used as type-hints when autowiring:

   Describes a logger instance.
   Psr\Log\LoggerInterface - alias:logger

   Request stack that controls the lifecycle of requests.
   Symfony\Component\HttpFoundation\RequestStack - alias:request_stack

   RouterInterface is the interface that all Router classes must implement.
   Symfony\Component\Routing\RouterInterface - alias:router.default

   [...]
```

当你在控制器方法或自己的服务中使用这些类型提示时，Symfony 会自动将与该类型匹配的服务对象传递给你。

在整个文档中，你将看到如何使用容器中存在的许多不同服务。

> **提示：**
> 容器中实际上有*更多*服务，每个服务在容器中都有一个唯一的 ID，如 `request_stack` 或 `router.default`。要获取完整列表，可以运行 `php bin/console debug:container`。但大多数情况下，你不需要担心这个。参阅如何选择特定服务以及 `/service_container/debug`。

## 在容器中创建/配置服务

你也可以将自己的代码组织成服务。例如，假设你需要向用户显示一条随机的开心消息。如果将这段代码放在控制器中，它将无法重用。因此，你决定创建一个新类：

```php
// src/Service/MessageGenerator.php
namespace App\Service;

class MessageGenerator
{
    public function getHappyMessage(): string
    {
        $messages = [
            'You did it! You updated the system! Amazing!',
            'That was one of the coolest updates I\'ve seen all day!',
            'Great work! Keep going!',
        ];

        $index = array_rand($messages);

        return $messages[$index];
    }
}
```

恭喜！你创建了第一个服务类！你可以立即在控制器中使用它：

```php
// src/Controller/ProductController.php
use App\Service\MessageGenerator;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class ProductController extends AbstractController
{
    #[Route('/products/new')]
    public function new(MessageGenerator $messageGenerator): Response
    {
        // thanks to the type-hint, the container will instantiate a
        // new MessageGenerator and pass it to you!
        // ...

        $message = $messageGenerator->getHappyMessage();
        $this->addFlash('success', $message);
        // ...
    }
}
```

当你请求 `MessageGenerator` 服务时，容器会构建一个新的 `MessageGenerator` 对象并返回它（见下面的侧边栏）。但如果你从不请求该服务，它就*永远*不会被构建：节省了内存和速度。另外，`MessageGenerator` 服务只创建*一次*：每次请求时都返回同一个实例。

> **自动在 services.yaml 中加载服务**
>
> 本文档假设你使用以下服务配置，这是新项目的默认配置：
>
> **YAML**
>
> ```yaml
> # config/services.yaml
> services:
>     # default configuration for services in *this* file
>     _defaults:
>         autowire: true      # Automatically injects dependencies in your services.
>         autoconfigure: true # Automatically registers your services as commands, event subscribers, etc.
>
>     # makes classes in src/ available to be used as services
>     # this creates a service per class whose id is the fully-qualified class name
>     App\:
>         resource: '../src/'
>
>     # order is important in this file because service definitions
>     # always *replace* previous ones; add your own service configuration below
>
>     # ...
> ```
>
> **PHP**
>
> ```php
> // config/services.php
> namespace Symfony\Component\DependencyInjection\Loader\Configurator;
>
> return App::config([
>     'services' => [
>         // Autowiring and autoconfiguration are enabled by default when using App::config()
>         // '_defaults' => [
>         //     'autowire' => true,      // Automatically injects dependencies in your services.
>         //     'autoconfigure' => true, // Automatically registers your services as commands, event subscribers, etc.
>         // ],
>         'App\\' => [
>             'resource' => '../src/',
>         ],
>         // order is important in this file because service definitions
>         // always *replace* previous ones; add your own service configuration below
>     ],
> ]);
> ```
>
> **提示：** `resource` 选项的值可以是任何有效的 [glob 模式](https://en.wikipedia.org/wiki/Glob_(programming))。
>
> 得益于此配置，你可以自动使用 `src/` 目录中的任何类作为服务，无需手动配置它。之后，你将学习如何使用 resource 一次性导入多个服务。
>
> 如果项目中的某些文件或目录不应成为服务，可以使用 `exclude` 选项排除它们：
>
> **YAML**
>
> ```yaml
> # config/services.yaml
> services:
>     # ...
>     App\:
>         resource: '../src/'
>         exclude:
>             - '../src/SomeDirectory/'
>             - '../src/AnotherDirectory/'
>             - '../src/SomeFile.php'
> ```
>
> **PHP**
>
> ```php
> // config/services.php
> namespace Symfony\Component\DependencyInjection\Loader\Configurator;
>
> return App::config([
>     'services' => [
>         'App\\' => [
>             'resource' => '../src/',
>             'exclude' => '../src/{SomeDirectory,AnotherDirectory,Kernel.php}',
>         ],
>     ],
> ]);
> ```
>
> 如果你更喜欢手动装配服务，可以使用显式配置。

### 将服务限制在特定 Symfony 环境中

你可以按如下方式将服务注册限制在特定环境中：

**PHP 属性**

```php
use Symfony\Component\DependencyInjection\Attribute\When;

// SomeClass is only registered in the "dev" environment

#[When(env: 'dev')]
class SomeClass
{
    // ...
}

// you can also apply more than one When attribute to the same class

#[When(env: 'dev')]
#[When(env: 'test')]
class AnotherClass
{
    // ...
}
```

**YAML**

```yaml
# config/services.yaml
services:
    App\Service\SomeClass: ~

when@dev:
    services:
        App\Service\AnotherClass: ~
```

**PHP**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return static function (ContainerConfigurator $container): void {
    $services = $container->services();

    $services->set(App\Service\SomeClass::class);

    if ('dev' === $container->env()) {
        $services->set(App\Service\AnotherClass::class);
    }
};
```

> **警告：**
> `_defaults` 部分仅适用于同一 `services` 块中定义的服务。每个 `when@<env>` 块都有自己的作用域，不继承主 `services` 部分的 `_defaults`。在每个需要它的 `when@<env>` 块中重新定义 `_defaults`：
>
> ```yaml
> # config/services.yaml
> services:
>     _defaults:
>         autowire: true
>         autoconfigure: true
>     # ...
>
> when@prod:
>     services:
>         _defaults:
>             autowire: true
>             autoconfigure: true
>         # ...
> ```

如果你想从特定环境的注册中排除某个服务，可以使用 `#[WhenNot]` 属性：

```php
use Symfony\Component\DependencyInjection\Attribute\WhenNot;

// SomeClass is registered in all environments except "dev"

#[WhenNot(env: 'dev')]
class SomeClass
{
    // ...
}

// you can apply more than one WhenNot attribute to the same class

#[WhenNot(env: 'dev')]
#[WhenNot(env: 'test')]
class AnotherClass
{
    // ...
}
```

## 向服务注入服务/配置

如果你需要在 `MessageGenerator` 中访问 `logger` 服务怎么办？没问题！创建一个 `__construct()` 方法，其中包含一个具有 `LoggerInterface` 类型提示的 `$logger` 参数。将其设置到一个新的 `$logger` 属性上，然后在之后使用它：

```php
// src/Service/MessageGenerator.php
namespace App\Service;

use Psr\Log\LoggerInterface;

class MessageGenerator
{
    public function __construct(
        private LoggerInterface $logger,
    ) {
    }

    public function getHappyMessage(): string
    {
        $this->logger->info('About to find a happy message!');
        // ...
    }
}
```

就是这样！容器在实例化 `MessageGenerator` 时会*自动*知道要传入 `logger` 服务。它是怎么知道的？靠的是**自动装配（Autowiring）**。关键是你在 `__construct()` 方法中使用了 `LoggerInterface` 类型提示，以及 `services.yaml` 中的 `autowire: true` 配置。当你对一个参数进行类型提示时，容器会自动找到匹配的服务。如果找不到，你会看到一个清晰的异常信息和有用的建议。

顺便说一句，这种向 `__construct()` 方法添加依赖项的方式叫做*依赖注入*。

如何知道对类型提示使用 `LoggerInterface`？你可以阅读你所使用功能的文档，或者使用之前展示的 `debug:autowiring` 命令来获取应用程序中所有可自动装配的类型提示列表。

除了注入服务外，你还可以将标量值和集合作为其他服务的参数传入：

**YAML**

```yaml
# config/services.yaml
services:
    App\Service\SomeService:
        arguments:
            # string, numeric and boolean arguments can be passed "as is"
            - 'Foo'
            - true
            - 7
            - 3.14

            # constants can be built-in, user-defined, or Enums
            - !php/const E_ALL
            - !php/const PDO::FETCH_NUM
            - !php/const Symfony\Component\HttpKernel\Kernel::VERSION
            - !php/const App\Config\SomeEnum::SomeCase

            # when not using autowiring, you can pass service arguments explicitly
            - '@some-service-id'  # the leading '@' tells this is a service ID, not a string
            - '@?some-service-id' # using '?' means to pass null if service doesn't exist

            # binary contents are passed encoded as base64 strings
            - !!binary VGhpcyBpcyBhIEJlbGwgY2hhciAH

            # collections (arrays) can include any type of argument
            -
                first: !php/const true
                second: 'Foo'
```

**PHP**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'services' => [
        App\Service\SomeService::class => [
            'arguments' => [
                'Foo',
                true,
                7,
                3.14,
                E_ALL,
                \PDO::FETCH_NUM,
                Symfony\Component\HttpKernel\Kernel::VERSION,
                App\Config\SomeEnum::SomeCase,
                service('some-service-id'),
                service('some-service-id')->nullOnInvalid(),
                [
                    'first' => true,
                    'second' => 'Foo',
                ],
            ],
        ],
    ],
]);
```

### 处理多个服务

假设你还想在每次站点更新时向站点管理员发送一封电子邮件。为此，你创建一个新类：

```php
// src/Service/SiteUpdateManager.php
namespace App\Service;

use App\Service\MessageGenerator;
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Mime\Email;

class SiteUpdateManager
{
    public function __construct(
        private MessageGenerator $messageGenerator,
        private MailerInterface $mailer,
    ) {
    }

    public function notifyOfSiteUpdate(): bool
    {
        $happyMessage = $this->messageGenerator->getHappyMessage();

        $email = new Email()
            ->from('admin@example.com')
            ->to('manager@example.com')
            ->subject('Site update just happened!')
            ->text('Someone just updated the site. We told them: '.$happyMessage);

        $this->mailer->send($email);

        // ...

        return true;
    }
}
```

这个类需要 `MessageGenerator` *和* `Mailer` 服务。没问题，通过对其类和接口名称进行类型提示来请求它们！现在，这个新服务已经可以使用了。例如，在控制器中，你可以对新的 `SiteUpdateManager` 类进行类型提示并使用它：

```php
// src/Controller/SiteController.php
namespace App\Controller;

use App\Service\SiteUpdateManager;
// ...

class SiteController extends AbstractController
{
    public function new(SiteUpdateManager $siteUpdateManager): Response
    {
        // ...

        if ($siteUpdateManager->notifyOfSiteUpdate()) {
            $this->addFlash('success', 'Notification mail was sent successfully.');
        }

        // ...
    }
}
```

得益于自动装配和 `__construct()` 中的类型提示，容器会创建 `SiteUpdateManager` 对象并向其传递正确的参数。在大多数情况下，这可以完美运行。

### 手动装配参数

但有些情况下，服务的某个参数无法被自动装配。例如，假设你想让管理员电子邮件可配置：

```diff
  // src/Service/SiteUpdateManager.php
  // ...

  class SiteUpdateManager
  {
      // ...

      public function __construct(
          private MessageGenerator $messageGenerator,
          private MailerInterface $mailer,
+         private string $adminEmail
      ) {
      }

      public function notifyOfSiteUpdate(): bool
      {
          // ...

          $email = new Email()
              // ...
-            ->to('manager@example.com')
+            ->to($this->adminEmail)
              // ...
          ;
          // ...
      }
  }
```

如果你做了这个更改并刷新页面，你会看到一个错误：

```
Cannot autowire service "App\\Service\\SiteUpdateManager": argument "$adminEmail"
of method "__construct()" must have a type-hint or be given a value explicitly.
```

这是有道理的！容器无法知道你想在这里传递什么值。没问题！在配置中，你可以显式设置这个参数：

**YAML**

```yaml
# config/services.yaml
services:
    # ... same as before

    # same as before
    App\:
        resource: '../src/'
        exclude: '../src/{DependencyInjection,Entity,Kernel.php}'

    # explicitly configure the service
    App\Service\SiteUpdateManager:
        arguments:
            $adminEmail: 'manager@example.com'
```

**PHP**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Service\SiteUpdateManager;

return App::config([
    'services' => [
        // ...
        SiteUpdateManager::class => [
            'arguments' => [
                '$adminEmail' => 'manager@example.com',
            ],
        ],
    ],
]);
```

得益于此，容器在创建 `SiteUpdateManager` 服务时，会将 `manager@example.com` 传递给 `__construct` 的 `$adminEmail` 参数。其他参数仍会被自动装配。

但是，这会不会很脆弱？幸运的是，不会！如果你将 `$adminEmail` 参数重命名为其他名称——例如 `$mainEmail`——在重新加载下一个页面时，你会得到一个清晰的异常（即使该页面不使用这个服务）。

> **提示：**
> 除了在 YAML 中配置标量参数外，你还可以使用 `#[Autowire]` 属性直接将它们注入到需要它们的服务中。

## 服务参数

除了存储服务对象外，容器还存储配置，称为**参数**。关于 Symfony 配置的主要文章详细解释了配置参数，并展示了所有类型（字符串、布尔值、数组、二进制和 PHP 常量参数）。

然而，还有另一种与服务相关的参数类型。在 YAML 配置中，任何以 `@` 开头的字符串都被视为服务的 ID，而不是普通字符串。在 PHP 配置中使用 `service()` 函数：

**YAML**

```yaml
# config/services.yaml
services:
    App\Service\MessageGenerator:
        arguments:
            # this is not a string, but a reference to a service called 'logger'
            - '@logger'

            # if the value of a string argument starts with '@', you need to escape
            # it by adding another '@' so Symfony doesn't consider it a service
            # the following example would be parsed as the string '@securepassword'
            # - '@@securepassword'
```

使用容器的参数访问器方法来处理容器参数非常简单：

```php
// checks if a parameter is defined (parameter names are case-sensitive)
$container->hasParameter('app.admin_email');

// gets value of a parameter
$container->getParameter('app.admin_email');

// adds a new parameter
$container->setParameter('app.admin_email', 'admin@example.com');
```

> **警告：**
> 所使用的 `.` 符号是一种 Symfony 约定，使参数更易于阅读。参数是扁平的键值元素，不能组织成嵌套数组。

> **注意：**
> 你只能在容器编译之前设置参数，而不能在运行时设置。要了解更多关于编译容器的信息，请参阅 `/components/dependency_injection/compilation`。

## 选择特定服务

前面创建的 `MessageGenerator` 服务需要一个 `LoggerInterface` 参数：

```php
// src/Service/MessageGenerator.php
namespace App\Service;

use Psr\Log\LoggerInterface;

class MessageGenerator
{
    public function __construct(
        private LoggerInterface $logger,
    ) {
    }
    // ...
}
```

然而，容器中有*多个*实现了 `LoggerInterface` 的服务，例如 `logger`、`monolog.logger.request`、`monolog.logger.php` 等。容器如何知道使用哪一个？

在这种情况下，容器通常被配置为自动选择其中一个服务——在本例中是 `logger`（在服务自动装配别名中了解更多原因）。但你可以控制这一点，并传入不同的 logger：

**YAML**

```yaml
# config/services.yaml
services:
    # ... same code as before

    # explicitly configure the service
    App\Service\MessageGenerator:
        arguments:
            # the '@' symbol is important: that's what tells the container
            # you want to pass the *service* whose id is 'monolog.logger.request',
            # and not just the *string* 'monolog.logger.request'
            $logger: '@monolog.logger.request'
```

**PHP**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Service\MessageGenerator;

return App::config([
    'services' => [
        // ...
        MessageGenerator::class => [
        'arguments' => [
                '$logger' => service('monolog.logger.request'),
            ],
        ],
    ],
]);
```

这告诉容器，`__construct` 的 `$logger` 参数应该使用 ID 为 `monolog.logger.request` 的服务。

> **提示：**
> 如果你需要在整个应用程序中在同一类型的多个实现之间进行选择，可以使用命名自动装配别名，而不是手动装配每个注入点。

要获取可与自动装配一起使用的 logger 服务列表，运行：

```bash
$ php bin/console debug:autowiring logger
```

要获取容器中*所有*可能服务的完整列表，运行：

```bash
$ php bin/console debug:container
```

## 移除服务

如果需要，可以从服务容器中移除一个服务。例如，这对于在某些配置环境中使服务不可用非常有用（例如在 `test` 环境中）：

```php
// config/services_test.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\RemovedService;

return function(ContainerConfigurator $containerConfigurator) {
    $services = $containerConfigurator->services();

    $services->remove(RemovedService::class);
};
```

现在，容器在 `test` 环境中将不包含 `App\RemovedService`。

## 将闭包作为参数注入

可以将可调用对象作为服务的参数注入。让我们在 `MessageGenerator` 的构造函数中添加一个参数：

```php
// src/Service/MessageGenerator.php
namespace App\Service;

use Psr\Log\LoggerInterface;

class MessageGenerator
{
    private string $messageHash;

    public function __construct(
        private LoggerInterface $logger,
        callable $generateMessageHash,
    ) {
        $this->messageHash = $generateMessageHash();
    }
    // ...
}
```

现在，我们添加一个新的可调用服务来生成消息哈希：

```php
// src/Hash/MessageHashGenerator.php
namespace App\Hash;

class MessageHashGenerator
{
    public function __invoke(): string
    {
        // Compute and return a message hash
    }
}
```

我们的配置如下所示：

**YAML**

```yaml
# config/services.yaml
services:
    # ... same code as before

    # explicitly configure the service
    App\Service\MessageGenerator:
        arguments:
            $logger: '@monolog.logger.request'
            $generateMessageHash: !closure '@App\Hash\MessageHashGenerator'
```

**PHP**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Service\MessageGenerator;

return App::config([
    'services' => [
        // ... same code as before

        // explicitly configure the service
        MessageGenerator::class => [
            'arguments' => [
                '$logger' => service('monolog.logger.request'),
                '$generateMessageHash' => closure('App\Hash\MessageHashGenerator'),
            ],
        ],
    ],
]);
```

> **另请参阅：**
> 闭包也可以通过使用自动装配及其专用属性来注入。

## 按名称或类型绑定参数

你也可以使用 `bind` 关键字按名称或类型绑定特定参数：

**YAML**

```yaml
# config/services.yaml
services:
    _defaults:
        bind:
            # pass this value to any $adminEmail argument for any service
            # that's defined in this file (including controller arguments)
            $adminEmail: 'manager@example.com'

            # pass this service to any $requestLogger argument for any
            # service that's defined in this file
            $requestLogger: '@monolog.logger.request'

            # pass this service for any LoggerInterface type-hint for any
            # service that's defined in this file
            Psr\Log\LoggerInterface: '@monolog.logger.request'

            # optionally you can define both the name and type of the argument to match
            string $adminEmail: 'manager@example.com'
            Psr\Log\LoggerInterface $requestLogger: '@monolog.logger.request'
            iterable $rules: !tagged_iterator app.foo.rule

    # ...
```

**PHP**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Psr\Log\LoggerInterface;

return App::config([
    'services' => [
        '_defaults' => [
            'bind' => [
                // pass this value to any $adminEmail argument for any service
                // that's defined in this file (including controller arguments)
                '$adminEmail' => 'manager@example.com',

                // pass this service to any $requestLogger argument for any
                // service that's defined in this file
                '$requestLogger' => service('monolog.logger.request'),

                // pass this service for any LoggerInterface type-hint for any
                // service that's defined in this file
                LoggerInterface::class => service('monolog.logger.request'),

                // optionally you can define both the name and type of the argument to match
                'string $adminEmail' => 'manager@example.com',
                LoggerInterface::class.' $requestLogger' => service('monolog.logger.request'),
                'iterable $rules' => tagged_iterator('app.foo.rule'),
            ],
        ],
    ],
]);
```

通过将 `bind` 键放在 `_defaults` 下，你可以为此文件中定义的*任何*服务的*任何*参数指定值！你可以按名称（例如 `$adminEmail`）、按类型（例如 `Psr\Log\LoggerInterface`）或两者都用（例如 `Psr\Log\LoggerInterface $requestLogger`）来绑定参数。

`bind` 配置也可以应用于特定服务或一次性加载多个服务时。

## 抽象服务参数

有时，某些服务参数的值无法在配置文件中定义，因为它们是在运行时使用编译器通道或 Bundle 扩展计算的。

在这种情况下，你可以使用 `abstract` 参数类型来至少定义参数的名称和关于其用途的简短描述：

**YAML**

```yaml
# config/services.yaml
services:
    # ...

    App\Service\MyService:
        arguments:
            $rootNamespace: !abstract 'should be defined by Pass'

    # ...
```

**PHP**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Service\MyService;

return App::config([
    'services' => [
        MyService::class => [
            'arguments' => [
                '$rootNamespace' => abstract_arg('should be defined by Pass'),
            ],
        ],
    ],
]);
```

如果在运行时没有替换抽象参数的值，将会抛出 `RuntimeException`，消息类似于 `Argument "$rootNamespace" of service "App\Service\MyService" is abstract: should be defined by Pass.`

## autowire 选项

上面，`services.yaml` 文件在 `_defaults` 部分有 `autowire: true`，以便它适用于该文件中定义的所有服务。通过此设置，你可以在服务的 `__construct()` 方法中对参数进行类型提示，容器会自动向你传递正确的参数。本文整篇都是围绕自动装配写的。

有关更多详细信息，请查看服务自动装配文档，其中涵盖了自动装配的工作原理、如何处理同一类型的多个实现、用于非服务参数的 `#[Autowire]` 属性以及从服务生成闭包。

## autoconfigure 选项

上面，`services.yaml` 文件在 `_defaults` 部分有 `autoconfigure: true`，以便它适用于该文件中定义的所有服务。通过此设置，容器会根据服务的*类*自动对你的服务应用某些配置。这主要用于*自动标记*你的服务。

例如，要创建一个 Twig 扩展，你需要创建一个类，将其注册为服务，并用 `twig.extension` 标记它。

但是，使用 `autoconfigure: true`，你不需要这个标记。实际上，如果你使用默认的 services.yaml 配置，你不需要做*任何事情*：服务将被自动加载。然后，`autoconfigure` 会*为你*添加 `twig.extension` 标记，因为你的类实现了 `Twig\Extension\ExtensionInterface`。得益于 `autowire`，你甚至可以添加构造函数参数而无需任何配置。

自动配置也适用于属性。一些属性如 `AsMessageHandler`、`AsEventListener` 和 `AsCommand` 被注册为自动配置。使用这些属性的任何类都会应用相应的标记。

## 检查服务定义

`lint:container` 命令会执行额外的检查，以确保容器被正确配置。在将应用程序部署到生产环境之前运行此命令很有用（例如在持续集成服务器中）：

```bash
$ php bin/console lint:container

# optionally, you can force the resolution of environment variables;
# the command will fail if any of those environment variables are missing
$ php bin/console lint:container --resolve-env-vars
```

每次编译容器时执行这些检查可能会影响性能。这就是为什么它们在名为 `CheckTypeDeclarationsPass` 和 `CheckAliasValidityPass` 的编译器通道中实现，默认情况下禁用，仅在执行 `lint:container` 命令时启用。如果你不介意性能损失，可以在应用程序中启用这些编译器通道。

## 公共服务与私有服务

每个定义的服务默认都是私有的。当服务是私有的时，你不能使用 `$container->get()` 直接从容器访问它。作为最佳实践，你应该只创建*私有*服务，并且应该使用依赖注入而不是 `$container->get()` 来获取服务。

如果你需要延迟获取服务，应该考虑使用服务定位器，而不是使用公共服务。

但是，如果你*确实*需要将服务设为公共，可以覆盖 `public` 设置：

**YAML**

```yaml
# config/services.yaml
services:
    # ... same code as before

    # explicitly configure the service
    App\Service\PublicService:
        public: true
```

**PHP**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Service\PublicService;

return App::config([
    'services' => [
        PublicService::class => [
            'public' => true,
        ],
    ],
]);
```

也可以通过 `#[Autoconfigure]` 属性将服务定义为公共的。此属性必须直接在你要配置的服务类上使用：

```php
// src/Service/PublicService.php
namespace App\Service;

use Symfony\Component\DependencyInjection\Attribute\Autoconfigure;

#[Autoconfigure(public: true)]
class PublicService
{
    // ...
}
```

## 使用 resource 一次导入多个服务

你已经看到可以使用 `resource` 键一次导入多个服务。例如，默认的 Symfony 配置包含以下内容：

**YAML**

```yaml
# config/services.yaml
services:
    # ... same as before

    # makes classes in src/ available to be used as services
    # this creates a service per class whose id is the fully-qualified class name
    App\:
        resource: '../src/'
        exclude: '../src/{DependencyInjection,Entity,Kernel.php}'
```

**PHP**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'services' => [
        // makes classes in src/ available to be used as services
        // this creates a service per class whose id is the fully-qualified class name
        'App\\' => [
            'resource' => '../src/',
            'exclude' => '../src/{DependencyInjection,Entity,Kernel.php}',
        ],
    ],
]);
```

> **提示：**
> `resource` 和 `exclude` 选项的值可以是任何有效的 [glob 模式](https://en.wikipedia.org/wiki/Glob_(programming))。如果你只想排除少数几个服务，可以直接在类上使用 `Exclude` 属性来排除它：
>
> ```php
> // src/Service/SomeService.php
> namespace App\Service;
>
> #[Exclude]
> class SomeService
> {
>     // ...
> }
> ```

这可以用来快速使许多类作为服务可用，并应用一些默认配置。每个服务的 `id` 是其完全限定的类名。你可以通过在下面使用其 id（类名）来覆盖任何导入的服务（例如参见如何手动装配参数）。如果你覆盖了一个服务，导入中的任何选项（例如 `public`）都不会被继承（但被覆盖的服务*确实*仍然从 `_defaults` 继承）。

你也可以 `exclude` 某些路径。这是可选的，但会略微提高 `dev` 环境中的性能：被排除的路径不会被跟踪，因此修改它们不会导致容器重建。

> **注意：**
> 等等，这是否意味着 `src/` 中的*每个*类都被注册为服务？甚至包括模型类？实际上，不是。只要你将导入的服务保持为私有，`src/` 中所有*没有*被明确用作服务的类都会从最终容器中自动删除。实际上，导入意味着所有类都"可以*被用作*服务"，而无需手动配置。

### 使用相同命名空间的多个服务定义

如果你使用 YAML 配置格式定义服务，PHP 命名空间被用作每个配置的键，因此你不能为同一命名空间下的类定义不同的服务配置。

为了有多个定义，添加 `namespace` 选项，并使用任何唯一字符串作为每个服务配置的键：

```yaml
# config/services.yaml
services:
    command_handlers:
        namespace: App\Domain\
        resource: '../src/Domain/*/CommandHandler'
        tags: [command_handler]

    event_subscribers:
        namespace: App\Domain\
        resource: '../src/Domain/*/EventSubscriber'
        tags: [event_subscriber]
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'services' => [
        'command_handlers' => [
            'namespace' => 'App\Domain\\',
            'resource' => '../src/Domain/*/CommandHandler',
            'tags' => ['command_handler'],
        ],
        'event_subscribers' => [
            'namespace' => 'App\Domain\\',
            'resource' => '../src/Domain/*/EventSubscriber',
            'tags' => ['event_subscriber'],
        ],
    ],
]);
```

## 显式配置服务和参数

自动加载服务和自动装配都是可选的。即使你使用它们，也可能有些情况下你想手动装配一个服务。例如，假设你想为 `SiteUpdateManager` 类注册*2*个服务——每个服务使用不同的管理员电子邮件。在这种情况下，每个服务都需要有一个唯一的服务 ID：

**YAML**

```yaml
# config/services.yaml
services:
    # ...

    # this is the service's id
    site_update_manager.superadmin:
        class: App\Service\SiteUpdateManager
        # you CAN still use autowiring: we just want to show what it looks like without
        autowire: false
        # manually wire all arguments
        arguments:
            - '@App\Service\MessageGenerator'
            - '@mailer'
            - 'superadmin@example.com'

    site_update_manager.normal_users:
        class: App\Service\SiteUpdateManager
        autowire: false
        arguments:
            - '@App\Service\MessageGenerator'
            - '@mailer'
            - 'contact@example.com'

    # Create an alias, so that - by default - if you type-hint SiteUpdateManager,
    # the site_update_manager.superadmin will be used
    App\Service\SiteUpdateManager: '@site_update_manager.superadmin'
```

**PHP**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Service\MessageGenerator;
use App\Service\SiteUpdateManager;

return App::config([
    'services' => [
        'site_update_manager.superadmin' => [
            'class' => SiteUpdateManager::class,
            // you CAN still use autowiring: we just want to show what it looks like without
            'autowire' => false,
            // manually wire all arguments
            'arguments' => [
                service(MessageGenerator::class),
                service('mailer'),
                'superadmin@example.com',
            ],
        ],
        'site_update_manager.normal_users' => [
            'class' => SiteUpdateManager::class,
            'autowire' => false,
            'arguments' => [
                service(MessageGenerator::class),
                service('mailer'),
                'contact@example.com',
            ],
        ],
        // Create an alias, so that - by default - if you type-hint SiteUpdateManager,
        // the site_update_manager.superadmin will be used
        SiteUpdateManager::class => service('site_update_manager.superadmin'),
    ],
]);
```

在这种情况下，注册了*两个*服务：`site_update_manager.superadmin` 和 `site_update_manager.normal_users`。得益于别名，如果你对 `SiteUpdateManager` 进行类型提示，第一个（`site_update_manager.superadmin`）会被传入。

如果你想传入第二个，你需要手动装配服务或创建一个命名的自动装配别名。

> **警告：**
> 如果你*没有*创建别名，并且正在从 `src/` 加载所有服务，那么*三个*服务已经被创建（自动服务 + 你的两个服务），自动加载的服务将在你对 `SiteUpdateManager` 进行类型提示时——默认情况下——被传入。这就是为什么创建别名是个好主意。

使用 PHP 闭包配置服务时，可以通过向闭包添加一个名为 `$env` 的字符串参数来自动注入当前环境值：

```php
// config/packages/my_config.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return function(ContainerConfigurator $containerConfigurator, string $env): void {
    // `$env` is automatically filled in, so you can configure your
    // services depending on which environment you're on
};
```

## 为函数式接口生成适配器

函数式接口是只有一个方法的接口。它们在概念上与闭包非常相似，只是它们唯一的方法有一个名称。此外，它们可以在代码中用作类型提示。

`AutowireCallable` 属性可用于为函数式接口生成适配器。假设你有以下函数式接口：

```php
// src/Service/MessageFormatterInterface.php
namespace App\Service;

interface MessageFormatterInterface
{
    public function format(string $message, array $parameters): string;
}
```

你还有一个定义了许多方法的服务，其中一个方法与前面接口的 `format()` 方法相同：

```php
// src/Service/MessageUtils.php
namespace App\Service;

class MessageUtils
{
    // other methods...

    public function format(string $message, array $parameters): string
    {
        // ...
    }
}
```

得益于 `#[AutowireCallable]` 属性，你现在可以将这个 `MessageUtils` 服务作为函数式接口实现注入：

```php
namespace App\Service\Mail;

use App\Service\MessageFormatterInterface;
use App\Service\MessageUtils;
use Symfony\Component\DependencyInjection\Attribute\AutowireCallable;

class Mailer
{
    public function __construct(
        #[AutowireCallable(service: MessageUtils::class, method: 'format')]
        private MessageFormatterInterface $formatter
    ) {
    }

    public function sendMail(string $message, array $parameters): string
    {
        $formattedMessage = $this->formatter->format($message, $parameters);

        // ...
    }
}
```

除了使用 `#[AutowireCallable]` 属性外，你还可以通过配置为函数式接口生成适配器：

**YAML**

```yaml
# config/services.yaml
services:

    # ...

    app.message_formatter:
        class: App\Service\MessageFormatterInterface
        from_callable: [!service {class: 'App\Service\MessageUtils'}, 'format']
```

**PHP**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Service\MessageFormatterInterface;
use App\Service\MessageUtils;

return App::config([
    'services' => [
        // ...
        'app.message_formatter' => [
            'class' => MessageFormatterInterface::class,
            'from_callable' => [inline_service(MessageUtils::class), 'format'],
        ],
    ],
]);
```

这样做，Symfony 将生成一个类（也称为*适配器*），实现 `MessageFormatterInterface`，该类将 `MessageFormatterInterface::format()` 的调用转发给底层服务的方法 `MessageUtils::format()`，并传入所有参数。

## 深入了解

- `/service_container/` 下的所有相关文档
