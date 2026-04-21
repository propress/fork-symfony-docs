# 安全

Symfony 提供了许多工具来保护你的应用程序。某些与 HTTP 相关的安全工具（如安全的会话 Cookie 和 CSRF 保护）默认就已提供。你将在本指南中了解的 SecurityBundle，提供了保护应用程序所需的所有身份认证和授权功能。

要开始使用，请安装 SecurityBundle：

```bash
$ composer require symfony/security-bundle
```

如果你已安装了 Symfony Flex，这还会为你创建一个 `security.yaml` 配置文件：

```yaml
# config/packages/security.yaml
security:
    # https://symfony.com/doc/current/security.html#registering-the-user-hashing-passwords
    password_hashers:
        Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface: 'auto'
    # https://symfony.com/doc/current/security.html#where-do-users-come-from-user-providers
    providers:
        users_in_memory: { memory: null }
    firewalls:
        dev:
            # 'assets/' is for AssetMapper, 'build/' for Webpack Encore.
            # (Note: no regex delimiters needed; Symfony adds `{}` automatically.)
            pattern: ^/(_profiler|_wdt|assets|build)/
            security: false
        main:
            lazy: true
            provider: users_in_memory

            # activate different ways to authenticate
            # https://symfony.com/doc/current/security.html#firewalls-authentication

            # https://symfony.com/doc/current/security/impersonating_user.html
            # switch_user: true

    # An easy way to control access for large sections of your site
    # Note: Only the *first* access control that matches will be used
    access_control:
        # - { path: ^/admin, roles: ROLE_ADMIN }
        # - { path: ^/profile, roles: ROLE_USER }
```

这个配置内容很多！接下来的章节将讨论三个主要元素：

**用户**（`providers`）
> 应用程序中任何受保护的部分都需要某种用户概念。用户提供者根据"用户标识符"（例如用户的电子邮件地址）从任何存储（如数据库）中加载用户；

**防火墙**与**用户认证**（`firewalls`）
> 防火墙是保护应用程序的核心。防火墙内的每个请求都会检查是否需要已认证的用户。防火墙还负责对该用户进行身份认证（例如使用登录表单）；

**访问控制（授权）**（`access_control`）
> 使用访问控制和授权检查器，你可以控制执行特定操作或访问特定 URL 所需的权限。

## 用户

Symfony 中的权限始终与用户对象关联。如果你需要保护（部分）应用程序，则需要创建一个用户类。这个类实现 `Symfony\Component\Security\Core\User\UserInterface`。它通常是一个 Doctrine 实体，但你也可以使用专用的 Security 用户类。

生成用户类的最简单方法是使用 MakerBundle 的 `make:user` 命令：

```bash
$ php bin/console make:user
 The name of the security user class (e.g. User) [User]:
 > User

 Do you want to store user data in the database (via Doctrine)? (yes/no) [yes]:
 > yes

 Enter a property name that will be the unique "display" name for the user (e.g. email, username, uuid) [email]:
 > email

 Will this app need to hash/check user passwords? Choose No if passwords are not needed or will be checked/hashed by some other system (e.g. a single sign-on server).

 Does this app need to hash/check user passwords? (yes/no) [yes]:
 > yes

 created: src/Entity/User.php
 created: src/Repository/UserRepository.php
 updated: src/Entity/User.php
 updated: config/packages/security.yaml
```

```php
// src/Entity/User.php
namespace App\Entity;

use App\Repository\UserRepository;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface;
use Symfony\Component\Security\Core\User\UserInterface;

#[ORM\Entity(repositoryClass: UserRepository::class)]
#[ORM\Table(name: '`user`')]
#[ORM\UniqueConstraint(name: 'UNIQ_IDENTIFIER_EMAIL', fields: ['email'])]
class User implements UserInterface, PasswordAuthenticatedUserInterface
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 180)]
    private ?string $email = null;

    /**
     * @var list<string> The user roles
     */
    #[ORM\Column]
    private array $roles = [];

    /**
     * @var string The hashed password
     */
    #[ORM\Column]
    private ?string $password = null;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getEmail(): ?string
    {
        return $this->email;
    }

    public function setEmail(string $email): static
    {
        $this->email = $email;

        return $this;
    }

    /**
     * A visual identifier that represents this user.
     *
     * @see UserInterface
     */
    public function getUserIdentifier(): string
    {
        return (string) $this->email;
    }

    /**
     * @see UserInterface
     */
    public function getRoles(): array
    {
        $roles = $this->roles;
        // guarantee every user at least has ROLE_USER
        $roles[] = 'ROLE_USER';

        return array_unique($roles);
    }

    /**
     * @param list<string> $roles
     */
    public function setRoles(array $roles): static
    {
        $this->roles = $roles;

        return $this;
    }

    /**
     * @see PasswordAuthenticatedUserInterface
     */
    public function getPassword(): ?string
    {
        return $this->password;
    }

    public function setPassword(string $password): static
    {
        $this->password = $password;

        return $this;
    }

    // [...]
}
```

> **提示：** 你可以向 `make:user` 传入 `--with-uuid` 或 `--with-ulid`。借助 Symfony 的 Uid 组件，这会生成一个 `User` 实体，其 `id` 类型为 `Uuid` 或 `Ulid`，而不是 `int`。

如果你的用户是 Doctrine 实体（如上例所示），不要忘记通过创建并运行迁移来创建数据库表：

```bash
$ php bin/console make:migration
$ php bin/console doctrine:migrations:migrate
```

> **提示：** 向 `make:migration` 传入 `--formatted` 可以生成整洁的迁移文件。

### 加载用户：用户提供者

除了创建实体外，`make:user` 命令还会在你的安全配置中添加用户提供者的配置：

```yaml
# config/packages/security.yaml
security:
    # ...

    providers:
        app_user_provider:
            entity:
                class: App\Entity\User
                property: email
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Entity\User;

return App::config([
    'security' => [
        // ...

        'providers' => [
            'app_user_provider' => [
                'entity' => [
                    'class' => User::class,
                    'property' => 'email',
                ],
            ],
        ],
    ],
]);
```

此用户提供者知道如何根据"用户标识符"（例如用户的电子邮件地址或用户名）从存储（如数据库）中（重新）加载用户。上述配置使用 Doctrine，以 `email` 属性作为"用户标识符"加载 `User` 实体。

用户提供者在安全生命周期中的几个地方被使用：

**根据标识符加载用户**
> 在登录（或任何其他认证器）期间，提供者根据用户标识符加载用户。某些其他功能（如用户模拟和记住我）也会使用此功能。

**从会话中重新加载用户**
> 在每次请求开始时，用户从会话中加载（除非你的防火墙是 `stateless`）。提供者会"刷新"用户（例如再次查询数据库以获取最新数据），以确保所有用户信息是最新的（如有必要，如果某些内容发生了变化，用户会被取消认证/登出）。有关此过程的更多信息，请参见用户会话刷新部分。

Symfony 提供了几个内置用户提供者：

- **实体用户提供者**：使用 Doctrine 从数据库加载用户；
- **LDAP 用户提供者**：从 LDAP 服务器加载用户；
- **内存用户提供者**：从配置文件加载用户；
- **链式用户提供者**：将两个或多个用户提供者合并成一个新的用户提供者。由于每个防火墙只有一个用户提供者，你可以使用此功能将多个提供者链接在一起。

内置用户提供者满足大多数应用程序的常见需求，但你也可以创建自己的自定义用户提供者。

> **注意：** 有时，你需要在另一个类中注入用户提供者（例如在你的自定义认证器中）。所有用户提供者遵循以下服务 ID 模式：`security.user.provider.concrete.<your-provider-name>`（其中 `<your-provider-name>` 是配置键，例如 `app_user_provider`）。如果你只有一个用户提供者，可以使用 `Symfony\Component\Security\Core\User\UserProviderInterface` 类型提示进行自动装配。

### 注册用户：哈希密码

许多应用程序要求用户使用密码登录。对于这些应用程序，SecurityBundle 提供了密码哈希和验证功能。

首先，确保你的用户类实现 `Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface`：

```php
// src/Entity/User.php

// ...
use Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface;

class User implements UserInterface, PasswordAuthenticatedUserInterface
{
    // ...

    /**
     * @see PasswordAuthenticatedUserInterface
     */
    public function getPassword(): ?string
    {
        return $this->password;
    }

    // ...
}
```

然后，配置应为此类使用的密码哈希器。如果你的 `security.yaml` 文件还未预配置，`make:user` 应该已经为你做了这件事：

```yaml
# config/packages/security.yaml
security:
    # ...
    password_hashers:
        # Use native password hasher, which auto-selects and migrates the best
        # possible hashing algorithm (which currently is "bcrypt")
        Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface: 'auto'
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface;

return App::config([
    'security' => [
        // ...

        // Use native password hasher, which auto-selects and migrates the best
        // possible hashing algorithm (currently this is "bcrypt")
        'password_hashers' => [
            PasswordAuthenticatedUserInterface::class => 'auto',
        ],
    ],
]);
```

现在 Symfony 知道了你想如何哈希密码，你可以在将用户保存到数据库之前使用 `UserPasswordHasherInterface` 服务来执行此操作：

```php
// src/Controller/RegistrationController.php
namespace App\Controller;

// ...
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;

class RegistrationController extends AbstractController
{
    public function index(UserPasswordHasherInterface $passwordHasher): Response
    {
        // ... e.g. get the user data from a registration form
        $user = new User(...);
        $plaintextPassword = ...;

        // hash the password (based on the security.yaml config for the $user class)
        $hashedPassword = $passwordHasher->hashPassword(
            $user,
            $plaintextPassword
        );
        $user->setPassword($hashedPassword);

        // ...
    }
}
```

> **注意：** 如果你的用户类是 Doctrine 实体且你对用户密码进行哈希，则与用户类相关的 Doctrine 仓库类必须实现 `Symfony\Component\Security\Core\User\PasswordUpgraderInterface`。

> **提示：** `make:registration-form` maker 命令可以帮助你设置注册控制器，并添加使用 SymfonyCastsVerifyEmailBundle 进行电子邮件地址验证等功能。
>
> ```bash
> $ composer require symfonycasts/verify-email-bundle
> $ php bin/console make:registration-form
> ```

你也可以通过运行以下命令手动哈希密码：

```bash
$ php bin/console security:hash-password
```

在密码文档中阅读有关所有可用哈希器（包括特定哈希器）和密码迁移的更多信息。

## 防火墙

`config/packages/security.yaml` 的 `firewalls` 部分是**最重要**的部分。"防火墙"是你的认证系统：防火墙定义了你的应用程序的哪些部分是受保护的，以及你的用户将如何进行身份认证（例如登录表单、API 令牌等）。

```yaml
# config/packages/security.yaml
security:
    # ...
    firewalls:
        # the order in which firewalls are defined is very important, as the
        # request will be handled by the first firewall whose pattern matches
        dev:
            # Ensure dev tools and static assets are always allowed
            pattern: ^/(_profiler|_wdt|assets|build)/
            security: false
        # a firewall with no pattern should be defined last because it will match all requests
        main:
            lazy: true
            # provider that you set earlier inside providers
            provider: app_user_provider

            # activate different ways to authenticate
            # https://symfony.com/doc/current/security.html#firewalls-authentication

            # https://symfony.com/doc/current/security/impersonating_user.html
            # switch_user: true
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'security' => [
        // ...

        'firewalls' => [
            // the order in which firewalls are defined is very important, as the
            // request will be handled by the first firewall whose pattern matches
            'dev' => [
                // Ensure dev tools and static assets are always allowed
                'pattern' => '^/(_profiler|_wdt|assets|build)/',
                'security' => false,
            ],

            // a firewall with no pattern should be defined last because it will match all requests
            'main' => [
                'lazy' => true,
                // provider that you set earlier inside providers
                'provider' => 'app_user_provider',

                // activate different ways to authenticate
                // https://symfony.com/doc/current/security.html#firewalls-authentication

                // https://symfony.com/doc/current/security/impersonating_user.html
                // 'switch_user' => true,
            ],
        ],
    ],
]);
```

每次请求只有一个防火墙处于活跃状态：Symfony 使用 `pattern` 键来查找第一个匹配项（你也可以通过主机或其他条件进行匹配）。这里，所有真实 URL 由 `main` 防火墙处理（没有 `pattern` 键意味着它匹配*所有* URL）。

`dev` 防火墙实际上是一个假防火墙：它确保你不会意外阻止 Symfony 的开发工具——这些工具位于 `/_profiler` 和 `/_wdt` 等 URL 下。

> **提示：** 在匹配多个路由时，除了创建一个长正则表达式，你也可以使用更简单的正则表达式数组来匹配每个路由：
>
> ```yaml
> # config/packages/security.yaml
> security:
>     # ...
>     firewalls:
>         dev:
>             pattern:
>                 - ^/_profiler/
>                 - ^/_wdt/
>                 - ^/assets/
>                 - ^/build/
> # ...
> ```
>
> ```php
> // config/packages/security.php
> namespace Symfony\Component\DependencyInjection\Loader\Configurator;
>
> return App::config([
>     'security' => [
>         // ...
>         'firewalls' => [
>             'dev' => [
>                 'pattern' => [
>                     '^/_profiler/',
>                     '^/_wdt/',
>                     '^/assets/',
>                     '^/build/',
>                 ],
>             ],
>         ],
>     ],
> ]);
> ```

防火墙可以有多种认证模式，换句话说，它支持多种提问"你是谁？"的方式。通常，当用户首次访问你的网站时，他们是未知的（即未登录的）。如果你现在访问主页，你*将*能够访问，并且你会在工具栏中看到你正在访问防火墙后面的页面：

![Symfony profiler 工具栏，安全信息显示"Authenticated: no"和"Firewall name: main"](/_images/security/anonymous_wdt.png)

访问防火墙下的 URL 不一定要求你进行身份认证（例如登录表单必须是可访问的，或者应用程序的某些部分是公开的）。另一方面，所有你希望*感知*已登录用户的页面都必须在同一个防火墙下。因此，如果你想在每个页面上显示"你已以...身份登录"的消息，它们都必须包含在同一个防火墙中。

你将在访问控制部分了解如何在防火墙内限制对 URL、控制器或其他任何内容的访问。

> **提示：** `lazy` 匿名模式可防止在不需要授权时（即没有明确检查用户权限时）启动会话。这对于保持请求可缓存性很重要（参见 HTTP 缓存文档）。

> **注意：** 如果你看不到工具栏，请使用以下命令安装分析器：
>
> ```bash
> $ composer require --dev symfony/profiler-pack
> ```

### 获取请求的防火墙配置

如果你需要获取匹配给定请求的防火墙配置，请使用 `Symfony\Bundle\SecurityBundle\Security` 服务：

```php
// src/Service/ExampleService.php
// ...

use Symfony\Bundle\SecurityBundle\Security;
use Symfony\Component\HttpFoundation\RequestStack;

class ExampleService
{
    public function __construct(
        // Avoid calling getFirewallConfig() in the constructor: auth may not
        // be complete yet. Instead, store the entire Security object.
        private Security $security,
        private RequestStack $requestStack,
    ) {
    }

    public function someMethod(): void
    {
        $request = $this->requestStack->getCurrentRequest();
        $firewallName = $this->security->getFirewallConfig($request)?->getName();

        // ...
    }
}
```

## 用户认证

在身份认证期间，系统会尝试为网页访问者找到匹配的用户。传统上，这是通过登录表单或浏览器中的 HTTP 基本对话框来完成的。但是，SecurityBundle 提供了许多其他认证器：

- 表单登录
- JSON 登录
- HTTP 基本认证
- 登录链接
- X.509 客户端证书
- 远程用户
- 自定义认证器

> **提示：** 如果你的应用程序通过第三方服务（如 Google、Facebook 或 Twitter）进行社交登录，请查看 HWIOAuthBundle 社区包或 Oauth2-client 包。

### 表单登录

大多数网站都有一个登录表单，用户使用标识符（例如电子邮件地址或用户名）和密码进行身份认证。此功能由内置的 `Symfony\Component\Security\Http\Authenticator\FormLoginAuthenticator` 提供。

你可以运行以下命令来创建在应用程序中添加登录表单所需的一切：

```bash
$ php bin/console make:security:form-login
```

此命令将创建所需的控制器和模板，并更新安全配置。或者，如果你更喜欢手动进行这些更改，请按照以下步骤操作。

首先，为登录表单创建一个控制器：

```bash
$ php bin/console make:controller Login

 created: src/Controller/LoginController.php
 created: templates/login/index.html.twig
```

```php
// src/Controller/LoginController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class LoginController extends AbstractController
{
    #[Route('/login', name: 'app_login')]
    public function index(): Response
    {
        return $this->render('login/index.html.twig', [
            'controller_name' => 'LoginController',
        ]);
    }
}
```

然后，使用 `form_login` 设置启用 `FormLoginAuthenticator`：

```yaml
# config/packages/security.yaml
security:
    # ...

    firewalls:
        main:
            # ...
            form_login:
                # "app_login" is the name of the route created previously
                login_path: app_login
                check_path: app_login
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'security' => [
        // ...

        'firewalls' => [
            'main' => [
                'form_login' => [
                    // "app_login" is the name of the route created previously
                    'login_path' => 'app_login',
                    'check_path' => 'app_login',
                ],
            ],
        ],
    ],
]);
```

> **注意：** `login_path` 和 `check_path` 支持 URL 和路由名称（但不能有强制通配符——例如 `foo` 没有默认值的 `/login/{foo}`）。

一旦启用，当未认证的访问者尝试访问受保护区域时，安全系统会将其重定向到 `login_path`（此行为可以使用认证入口点进行自定义）。

编辑登录控制器以渲染登录表单：

```diff
  // ...
+ use Symfony\Component\Security\Http\Authentication\AuthenticationUtils;

  class LoginController extends AbstractController
  {
      #[Route('/login', name: 'app_login')]
-     public function index(): Response
+     public function index(AuthenticationUtils $authenticationUtils): Response
      {
+         // get the login error if there is one
+         $error = $authenticationUtils->getLastAuthenticationError();
+
+         // last username entered by the user
+         $lastUsername = $authenticationUtils->getLastUsername();
+
          return $this->render('login/index.html.twig', [
-             'controller_name' => 'LoginController',
+             'last_username' => $lastUsername,
+             'error'         => $error,
          ]);
      }
  }
```

不要被这个控制器搞混。它的工作只是*渲染*表单。`FormLoginAuthenticator` 将自动处理表单*提交*。如果用户提交了无效的电子邮件或密码，该认证器将存储错误并重定向回此控制器，在这里我们读取错误（使用 `AuthenticationUtils`）以便将其显示给用户。

最后，创建或更新模板：

```twig
{# templates/login/index.html.twig #}
{% extends 'base.html.twig' %}

{# ... #}

{% block body %}
    {% if error %}
        <div>{{ error.messageKey|trans(error.messageData, 'security') }}</div>
    {% endif %}

    <form action="{{ path('app_login') }}" method="post">
        <label for="username">Email:</label>
        <input type="text" id="username" name="_username" value="{{ last_username }}" required>

        <label for="password">Password:</label>
        <input type="password" id="password" name="_password" required>

        {# If you want to control the URL the user is redirected to on success
        <input type="hidden" name="_target_path" value="/account"> #}

        <button type="submit">login</button>
    </form>
{% endblock %}
```

> **警告：** 传递到模板的 `error` 变量是 `Symfony\Component\Security\Core\Exception\AuthenticationException` 的实例。它可能包含有关认证失败的敏感信息。**永远不要**使用 `error.message`：改用 `messageKey` 属性，如示例所示。此消息始终可以安全显示。

表单可以是任何样子，但它通常遵循一些约定：

- `<form>` 元素向 `app_login` 路由发送 `POST` 请求，因为这是你在 `security.yaml` 的 `form_login` 键下配置的 `check_path`；
- 用户名（或任何你的用户"标识符"，例如电子邮件）字段的名称为 `_username`，密码字段的名称为 `_password`。

> **提示：** 实际上，所有这些都可以在 `form_login` 键下进行配置。有关更多详细信息，请参阅 `reference-security-firewall-form-login`。

> **注意：** 此登录表单目前未受到 CSRF 攻击的保护。请阅读下面关于如何保护登录表单的内容。

就这样！当你提交表单时，安全系统会自动读取 `_username` 和 `_password` POST 参数，通过用户提供者加载用户，检查用户的凭据，并对用户进行身份认证，或将其发送回登录表单（可以显示错误）。

整个流程回顾：

1. 用户尝试访问受保护的资源（例如 `/admin`）；
2. 防火墙通过将用户重定向到登录表单（`/login`）来启动身份认证过程；
3. `/login` 页面通过本示例中创建的路由和控制器渲染登录表单；
4. 用户将登录表单提交到 `/login`；
5. 安全系统（即 `FormLoginAuthenticator`）拦截请求，检查用户提交的凭据，如果凭据正确则对用户进行身份认证，否则将用户发送回登录表单。

> **参见：** 你可以自定义登录成功或失败时的响应。请参阅表单登录文档。

#### CSRF 保护登录表单

可以使用向登录表单中添加隐藏 CSRF 令牌的相同技术来防止登录 CSRF 攻击。Security 组件已经提供了 CSRF 保护，但你需要在使用之前配置一些选项。

首先，你需要在表单登录上启用 CSRF：

```yaml
# config/packages/security.yaml
security:
    # ...

    firewalls:
        secured_area:
            # ...
            form_login:
                # ...
                enable_csrf: true
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'security' => [
        'firewalls' => [
            'secured_area' => [
                'form_login' => [
                    'enable_csrf' => true,
                ],
            ],
        ],
    ],
]);
```

然后，在 Twig 模板中使用 `csrf_token()` 函数生成 CSRF 令牌，并将其存储为表单的隐藏字段。默认情况下，HTML 字段必须命名为 `_csrf_token`，用于生成值的字符串必须为 `authenticate`：

```twig
{# templates/login/index.html.twig #}

{# ... #}
<form action="{{ path('app_login') }}" method="post">
    {# ... the login fields #}

    <input type="hidden" name="_csrf_token" data-controller="csrf-protection" value="{{ csrf_token('authenticate') }}">

    <button type="submit">login</button>
</form>
```

此后，你已经保护了登录表单免受 CSRF 攻击。

> **提示：** 你可以通过设置 `csrf_parameter` 来更改字段名称，并通过在配置中设置 `csrf_token_id` 来更改令牌 ID。有关更多详细信息，请参阅 `reference-security-firewall-form-login`。

### JSON 登录

某些应用程序提供使用令牌保护的 API。这些应用程序可能使用端点根据用户名（或电子邮件）和密码提供这些令牌。JSON 登录认证器帮助你创建此功能。

使用 `json_login` 设置启用认证器：

```yaml
# config/packages/security.yaml
security:
    # ...

    firewalls:
        main:
            # ...
            json_login:
                # api_login is a route we will create below
                check_path: api_login
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'security' => [
        'firewalls' => [
            'main' => [
                'json_login' => [
                    // api_login is a route we will create below
                    'check_path' => 'api_login',
                ],
            ],
        ],
    ],
]);
```

> **注意：** `check_path` 支持 URL 和路由名称（但不能有强制通配符——例如 `foo` 没有默认值的 `/login/{foo}`）。

认证器在客户端请求 `check_path` 时运行。首先，为此路径创建一个控制器：

```bash
$ php bin/console make:controller --no-template ApiLogin

 created: src/Controller/ApiLoginController.php
```

```php
// src/Controller/ApiLoginController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class ApiLoginController extends AbstractController
{
    #[Route('/api/login', name: 'api_login')]
    public function index(): Response
    {
        return $this->json([
            'message' => 'Welcome to your new controller!',
            'path' => 'src/Controller/ApiLoginController.php',
        ]);
    }
}
```

认证器成功对用户进行身份认证后，将调用此登录控制器。你可以获取已认证的用户，生成令牌（或需要返回的任何内容）并返回 JSON 响应：

```diff
  // ...
+ use App\Entity\User;
+ use Symfony\Component\Security\Http\Attribute\CurrentUser;

  class ApiLoginController extends AbstractController
  {
-     #[Route('/api/login', name: 'api_login')]
+     #[Route('/api/login', name: 'api_login', methods: ['POST'])]
-     public function index(): Response
+     public function index(#[CurrentUser] ?User $user): Response
      {
+         if (null === $user) {
+             return $this->json([
+                 'message' => 'missing credentials',
+             ], Response::HTTP_UNAUTHORIZED);
+         }
+
+         $token = ...; // somehow create an API token for $user
+
          return $this->json([
-             'message' => 'Welcome to your new controller!',
-             'path' => 'src/Controller/ApiLoginController.php',
+             'user'  => $user->getUserIdentifier(),
+             'token' => $token,
          ]);
      }
  }
```

> **注意：** `#[CurrentUser]` 只能用于控制器参数来检索已认证的用户。在服务中，你应该使用 `Symfony\Bundle\SecurityBundle\Security::getUser`。

就这样！总结流程：

1. 客户端（如前端）向 `/api/login` 发送带有 `Content-Type: application/json` 头的 *POST 请求*，包含 `username`（即使你的标识符实际上是电子邮件）和 `password` 键：

```json
{
    "username": "dunglas@example.com",
    "password": "MyPassword"
}
```

2. 安全系统拦截请求，检查用户提交的凭据并对用户进行身份认证。如果凭据不正确，则返回 HTTP 401 未授权 JSON 响应，否则运行你的控制器；

3. 你的控制器创建正确的响应：

```json
{
    "user": "dunglas@example.com",
    "token": "45be42..."
}
```

> **提示：** JSON 请求格式可以在 `json_login` 键下进行配置。有关更多详细信息，请参阅 `reference-security-firewall-json-login`。

### HTTP 基本认证

HTTP 基本认证是一种标准化的 HTTP 认证框架。它使用浏览器中的对话框要求凭据（用户名和密码），Symfony 的 HTTP 基本认证器将验证这些凭据。

在防火墙中添加 `http_basic` 键以启用 HTTP 基本认证：

```yaml
# config/packages/security.yaml
security:
    # ...

    firewalls:
        main:
            # ...
            http_basic:
                realm: Secured Area
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'security' => [
        'firewalls' => [
            'main' => [
                'http_basic' => [
                    'realm' => 'Secured Area',
                ],
            ],
        ],
    ],
]);
```

就这样！每当未认证的用户尝试访问受保护的页面时，Symfony 将通知浏览器它需要启动 HTTP 基本认证（使用 `WWW-Authenticate` 响应头）。然后，认证器验证凭据并对用户进行身份认证。

> **注意：** 你不能将注销功能与 HTTP 基本认证器一起使用。即使你从 Symfony 注销，你的浏览器也会"记住"你的凭据，并会在每次请求时发送它们。

### 登录链接

登录链接是一种无密码认证机制。用户将收到一个短期链接（例如通过电子邮件），该链接将对其进行网站身份认证。

你可以在登录链接文档中了解有关此认证器的所有信息。

### 访问令牌

访问令牌通常用于 API 上下文。用户从授权服务器接收令牌，该令牌对其进行身份认证。

你可以在访问令牌文档中了解有关此认证器的所有信息。

### X.509 客户端证书

使用客户端证书时，你的 Web 服务器本身完成所有身份认证。Symfony 提供的 X.509 认证器从客户端证书的"可分辨名称"（DN）中提取电子邮件。然后，它将此电子邮件用作用户提供者中的用户标识符。

首先，配置你的 Web 服务器以启用客户端证书验证，并将证书的 DN 暴露给 Symfony 应用程序：

```nginx
server {
    # ...

    ssl_client_certificate /path/to/my-custom-CA.pem;

    # enable client certificate verification
    ssl_verify_client optional;
    ssl_verify_depth 1;

    location / {
        # pass the DN as "SSL_CLIENT_S_DN" to the application
        fastcgi_param SSL_CLIENT_S_DN $ssl_client_s_dn;

        # ...
    }
}
```

```apache
# ...
SSLCACertificateFile "/path/to/my-custom-CA.pem"
SSLVerifyClient optional
SSLVerifyDepth 1

# pass the DN to the application
SSLOptions +StdEnvVars
```

```caddy
tls {
    client_auth {
        mode verify_if_given # check the Caddy documentation for more information
        trusted_ca_cert_file /path/to/my-custom-CA.pem
    }
}

route {
    # Other configuration options go here

    php_fastcgi unix//var/run/php/php-fpm.sock {
        env SSL_CLIENT_S_DN {tls_client_subject}

        # Environment variables for other certificate fields that you might need.
        # They are not used by Symfony, but you can use them in your application.
        # See all placeholders: https://caddyserver.com/docs/caddyfile/concepts#placeholders
        env SSL_CLIENT_S_FINGERPRINT {tls_client_fingerprint}
        env SSL_CLIENT_S_CERTIFICATE {tls_client_certificate_der_base64}
        env SSL_CLIENT_S_ISSUER {tls_client_issuer}
        env SSL_CLIENT_S_SERIAL {tls_client_serial}
        env SSL_CLIENT_S_VERSION {tls_version}
    }
}
```

然后，在防火墙上使用 `x509` 启用 X.509 认证器：

```yaml
# config/packages/security.yaml
security:
    # ...

    firewalls:
        main:
            # ...
            x509:
                provider: your_user_provider
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'security' => [
        'firewalls' => [
            'main' => [
                'x509' => [
                    'provider' => 'your_user_provider',
                ],
            ],
        ],
    ],
]);
```

默认情况下，Symfony 以两种不同方式从 DN 中提取电子邮件地址：

1. 首先，它尝试 `SSL_CLIENT_S_DN_Email` 服务器参数，该参数由 Apache 暴露；
2. 如果未设置（例如使用 Nginx 时），它使用 `SSL_CLIENT_S_DN` 并匹配 `emailAddress` 后面的值。

你可以在 `x509` 键下自定义某些参数的名称。有关更多详细信息，请参阅 x509 配置参考。

### 远程用户

除了客户端证书认证之外，还有更多 Web 服务器模块可以预认证用户（例如 kerberos）。远程用户认证器为这些服务提供基本集成。

这些模块通常在 `REMOTE_USER` 环境变量中暴露已认证的用户。远程用户认证器将此值作为用户标识符来加载相应的用户。

使用 `remote_user` 键启用远程用户认证：

```yaml
# config/packages/security.yaml
security:
    firewalls:
        main:
            # ...
            remote_user:
                provider: your_user_provider
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'security' => [
        'firewalls' => [
            'main' => [
                'remote_user' => [
                    'provider' => 'your_user_provider',
                ],
            ],
        ],
    ],
]);
```

> **提示：** 你可以在 `remote_user` 键下自定义此服务器变量的名称。有关更多详细信息，请参阅配置参考。

### 限制登录尝试次数

Symfony 借助速率限制器组件提供针对暴力破解登录攻击的基本保护。如果你还没有在应用程序中使用此组件，请在使用此功能之前安装它：

```bash
$ composer require symfony/rate-limiter
```

然后，使用 `login_throttling` 设置启用此功能：

```yaml
# config/packages/security.yaml
security:

    firewalls:
        # ...

        main:
            # ...

            # by default, the feature allows 5 login attempts per minute
            login_throttling: null

            # configure the maximum login attempts
            login_throttling:
                max_attempts: 3          # per minute ...
                # interval: '15 minutes' # ... or in a custom period

            # use a custom rate limiter via its service ID
            login_throttling:
                limiter: app.my_login_rate_limiter
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'security' => [
        'firewalls' => [
            'main' => [
                // by default, the feature allows 5 login attempts per minute
                'login_throttling' => null,

                // configure the maximum login attempts
                'login_throttling' => [
                    'max_attempts' => 3, // per minute ...
                    'interval' => '15 minutes', // ... or in a custom period
                ],

                // use a custom rate limiter via its service ID
                'login_throttling' => [
                    'limiter' => 'app.my_login_rate_limiter',
                ],
            ],
        ],
    ],
]);
```

> **注意：** `interval` 选项的值必须是一个数字，后跟 PHP 日期相对格式接受的任何单位（例如 `3 seconds`、`10 hours`、`1 day` 等）。

在内部，Symfony 使用速率限制器组件，默认使用 Symfony 的缓存来存储之前的登录尝试。你可以配置缓存池或提供自定义存储服务：

```yaml
# config/packages/security.yaml
security:
    firewalls:
        main:
            login_throttling:
                # use a specific cache pool for storing limiter state
                cache_pool: 'cache.rate_limiter'
                # or use a custom storage service (takes precedence over cache_pool)
                # storage_service: 'app.my_custom_storage'
```

```php
// config/packages/security.php
use Symfony\Config\SecurityConfig;

return static function (SecurityConfig $security): void {
    $mainFirewall = $security->firewall('main');

    $mainFirewall->loginThrottling()
        // use a specific cache pool for storing limiter state
        ->cachePool('cache.rate_limiter')
        // or use a custom storage service (takes precedence over cache_pool)
        // ->storageService('app.my_custom_storage')
    ;
};
```

登录尝试次数受 `max_attempts`（默认值：5）限制，`IP 地址 + 用户名` 的失败请求次数和 `5 * max_attempts` 限制 `IP 地址` 的失败请求次数。第二个限制可防止攻击者使用多个用户名绕过第一个限制，同时不会干扰大型网络（例如办公室）上的普通用户。

> **提示：** 限制失败的登录尝试只是对暴力破解攻击的一种基本保护。OWASP 暴力破解攻击指南提到了几种其他保护措施，你应该根据所需的保护级别加以考虑。

如果你需要更复杂的限制算法，请创建一个实现 `Symfony\Component\HttpFoundation\RateLimiter\RequestRateLimiterInterface` 的类（或使用 `Symfony\Component\Security\Http\RateLimiter\DefaultLoginRateLimiter`），并将 `limiter` 选项设置为其服务 ID：

```yaml
# config/packages/security.yaml
framework:
    rate_limiter:
        # define 2 rate limiters (one for username+IP, the other for IP)
        username_ip_login:
            policy: token_bucket
            limit: 5
            rate: { interval: '5 minutes' }

        ip_login:
            policy: sliding_window
            limit: 50
            interval: '15 minutes'

services:
    # our custom login rate limiter
    app.login_rate_limiter:
        class: Symfony\Component\Security\Http\RateLimiter\DefaultLoginRateLimiter
        arguments:
            # globalFactory is the limiter for IP
            $globalFactory: '@limiter.ip_login'
            # localFactory is the limiter for username+IP
            $localFactory: '@limiter.username_ip_login'
            $secret: '%kernel.secret%'

security:
    firewalls:
        main:
            # use a custom rate limiter via its service ID
            login_throttling:
                limiter: app.login_rate_limiter
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\Security\Http\RateLimiter\DefaultLoginRateLimiter;

return App::config([
    'framework' => [
        'rate_limiter' => [
            // define 2 rate limiters (one for username+IP, the other for IP)
            'username_ip_login' => [
                'policy' => 'token_bucket',
                'limit' => 5,
                'rate' => ['interval' => '5 minutes'],
            ],
            'ip_login' => [
                'policy' => 'sliding_window',
                'limit' => 50,
                'interval' => '15 minutes',
            ],
        ],
    ],
    new ServicesConfig(
        services: [
            // our custom login rate limiter
            'app.login_rate_limiter' => [
                'class' => DefaultLoginRateLimiter::class,
                'arguments' => [
                    // globalFactory is the limiter for IP
                    '$globalFactory' => service('limiter.ip_login'),
                    // localFactory is the limiter for username+IP
                    '$localFactory' => service('limiter.username_ip_login'),
                    // secret is the app secret
                    '$secret' => param('kernel.secret'),
                ],
            ],
        ],
    ),
    'security' => [
        'firewalls' => [
            'main' => [
                // use a custom rate limiter via its service ID
                'login_throttling' => [
                    'limiter' => 'app.login_rate_limiter',
                ],
            ],
        ],
    ],
]);
```

### 自定义认证成功和失败行为

如果你想自定义认证成功或失败过程的处理方式，不必全局覆盖相应的监听器。相反，你可以通过实现 `Symfony\Component\Security\Http\Authentication\AuthenticationSuccessHandlerInterface` 或 `Symfony\Component\Security\Http\Authentication\AuthenticationFailureHandlerInterface` 来设置自定义成功/失败处理器。

有关如何自定义成功处理器的更多信息，请阅读相关文档。

## 编程式登录

你可以使用 `Symfony\Bundle\SecurityBundle\Security` 助手的 `login()` 方法编程式地登录用户：

```php
// src/Controller/SecurityController.php
namespace App\Controller;

use App\Security\Authenticator\ExampleAuthenticator;
use Symfony\Bundle\SecurityBundle\Security;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\RememberMeBadge;

class SecurityController
{
    public function someAction(Security $security): Response
    {
        // get the user to be authenticated
        $user = ...;

        // log the user in on the current firewall
        $security->login($user);

        // if the firewall has more than one authenticator, you must pass it explicitly
        // by using the name of built-in authenticators...
        $security->login($user, 'form_login');
        // ...or the service id of custom authenticators
        $security->login($user, ExampleAuthenticator::class);

        // you can also log in on a different firewall...
        $security->login($user, 'form_login', 'other_firewall');

        // ... add badges...
        $security->login($user, 'form_login', 'other_firewall', [new RememberMeBadge()->enable()]);

        // ... and also add passport attributes
        $security->login($user, 'form_login', 'other_firewall', [new RememberMeBadge()->enable()], ['referer' => 'https://oauth.example.com']);

        // use the redirection logic applied to regular login
        $redirectResponse = $security->login($user);
        return $redirectResponse;

        // or use a custom redirection logic (e.g. redirect users to their account page)
        // return new RedirectResponse('...');
    }
}
```

## 登出

要启用登出功能，请在防火墙下激活 `logout` 配置参数：

```yaml
# config/packages/security.yaml
security:
    # ...

    firewalls:
        main:
            # ...
            logout:
                path: /logout

                # where to redirect after logout
                # target: app_any_route
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'security' => [
        'firewalls' => [
            'main' => [
                'logout' => [
                    'path' => '/logout',

                    // where to redirect after logout
                    // 'target' => 'app_any_route',
                ],
            ],
        ],
    ],
]);
```

Symfony 随后会对导航到已配置 `path` 的用户取消认证，并将其重定向到已配置的 `target`。

> **提示：** 如果你需要引用登出路径，可以使用 `_logout_<firewallname>` 路由名称（例如 `_logout_main`）。

如果你的项目不使用 Symfony Flex，请确保你已在路由中导入了登出路由加载器：

```yaml
# config/routes/security.yaml
_symfony_logout:
    resource: security.route_loader.logout
    type: service
```

```php
// config/routes/security.php
namespace Symfony\Component\Routing\Loader\Configurator;

return Routes::config([
    '_symfony_logout' => [
        'resource' => 'security.route_loader.logout',
        'type' => 'service',
    ],
]);
```

### 编程式登出

你可以使用 `Symfony\Bundle\SecurityBundle\Security` 助手的 `logout()` 方法编程式地登出用户：

```php
// src/Controller/SecurityController.php
namespace App\Controller;

use Symfony\Bundle\SecurityBundle\Security;

class SecurityController
{
    public function someAction(Security $security): Response
    {
        // logout the user in on the current firewall
        $response = $security->logout();

        // you can also disable the csrf logout
        $response = $security->logout(false);

        // ... return $response (if set) or e.g. redirect to the homepage
    }
}
```

用户将从请求的防火墙中登出。如果请求不在防火墙后面，则会抛出 `\LogicException`。

### 自定义登出

在某些情况下，你需要在登出时运行额外的逻辑（例如使某些令牌失效），或者想自定义登出后发生的事情。在登出期间，会分发一个 `Symfony\Component\Security\Http\Event\LogoutEvent` 事件。注册一个事件监听器或订阅者来执行自定义逻辑：

```php
// src/EventListener/LogoutSubscriber.php
namespace App\EventListener;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\RedirectResponse;
use Symfony\Component\Routing\Generator\UrlGeneratorInterface;
use Symfony\Component\Security\Http\Event\LogoutEvent;

class LogoutSubscriber implements EventSubscriberInterface
{
    public function __construct(
        private UrlGeneratorInterface $urlGenerator
    ) {
    }

    public static function getSubscribedEvents(): array
    {
        return [LogoutEvent::class => 'onLogout'];
    }

    public function onLogout(LogoutEvent $event): void
    {
        // get the security token of the session that is about to be logged out
        $token = $event->getToken();

        // get the current request
        $request = $event->getRequest();

        // get the current response, if it is already set by another listener
        $response = $event->getResponse();

        // configure a custom logout response to the homepage
        $response = new RedirectResponse(
            $this->urlGenerator->generate('homepage'),
            RedirectResponse::HTTP_SEE_OTHER
        );
        $event->setResponse($response);
    }
}
```

### 自定义登出路径

另一个选项是将 `path` 配置为路由名称。如果你想让登出 URI 是动态的（例如根据当前语言环境进行翻译），这会很有用。在这种情况下，你必须自己创建此路由：

```yaml
# config/routes.yaml
app_logout:
    path:
        en: /logout
        fr: /deconnexion
    methods: GET
```

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

return Routes::config([
    'app_logout' => [
        'path' => [
            'en' => '/logout',
            'fr' => '/deconnexion',
        ],
        'methods' => ['GET'],
    ],
]);
```

然后，将路由名称传递给 `path` 选项：

```yaml
# config/packages/security.yaml
security:
    # ...

    firewalls:
        main:
            # ...
            logout:
                path: app_logout
```

```php
// config/packages/security.php
namespace Symfony\Component\Routing\Loader\Configurator;

return [
    Routes::config([
        'firewalls' => [
            'main' => [
                'logout' => [
                    'path' => 'app_logout',
                ],
            ],
        ],
    ]),
];
```

## 获取用户对象

### 从控制器中获取用户

要在控制器中获取已认证的用户，向控制器参数添加 `#[CurrentUser]` 属性，并使用代表你的用户的类（通常是 `User`）进行类型提示。将参数设为可空以允许匿名访问，或设为非可空以在没有用户通过身份认证时自动拒绝访问（Symfony 将抛出 `403` 错误）。

基础控制器的 `getUser()` 快捷方式也有效，但 `#[CurrentUser]` 更为推荐，因为它提供了正确的类型提示（无需 `@var` 注解），适用于任何控制器（不仅仅是继承 `AbstractController` 的控制器），并使方法签名中对已认证用户的依赖变得明确：

```php
// src/Controller/ProfileController.php
namespace App\Controller;

use App\Entity\User;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Security\Http\Attribute\CurrentUser;

class ProfileController
{
    // usually you'll want to make sure the user is authenticated first,
    // see "Authorization" below
    #[IsGranted('IS_AUTHENTICATED_FULLY')]
    public function index(#[CurrentUser] User $user): Response
    {
        // ... call here any methods you've added to your User class
        return new Response('Well hi there '.$user->getFirstName());
    }
}
```

```php
// src/Controller/ProfileController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;

class ProfileController extends AbstractController
{
    public function index(): Response
    {
        // usually you'll want to make sure the user is authenticated first,
        // see "Authorization" below
        $this->denyAccessUnlessGranted('IS_AUTHENTICATED_FULLY');

        /** @var \App\Entity\User $user */
        $user = $this->getUser();

        // ... call here any methods you've added to your User class
        return new Response('Well hi there '.$user->getFirstName());
    }
}
```

> **提示：** 你可以将 `#[CurrentUser]` 属性应用于不同用户类的联合类型：
>
> ```php
> #[CurrentUser] Admin|Customer|User $user
> ```

### 从服务中获取用户

如果你需要从服务中获取已登录的用户，请使用 `Symfony\Bundle\SecurityBundle\Security` 服务：

```php
// src/Service/ExampleService.php
// ...

use Symfony\Bundle\SecurityBundle\Security;

class ExampleService
{
    // avoid calling getUser() in the constructor: auth may not
    // be complete yet. Instead, inject the entire Security object.
    public function __construct(
        private Security $security,
    ){
    }

    public function someMethod(): void
    {
        // returns User object or null if not authenticated
        $user = $this->security->getUser();

        // ...
    }
}
```

### 在模板中获取用户

在 Twig 模板中，用户对象通过 Twig 全局应用变量的 `app.user` 变量可用：

```twig
{% if is_granted('IS_AUTHENTICATED_FULLY') %}
    <p>Email: {{ app.user.email }}</p>
{% endif %}
```

## 访问控制（授权）

用户现在可以使用你的登录表单登录到你的应用程序。很好！现在，你需要学习如何拒绝访问并使用用户对象。这称为**授权**，其工作是决定用户是否可以访问某些资源（URL、模型对象、方法调用等）。

授权过程有两个不同的方面：

1. 用户在登录时收到特定角色（例如 `ROLE_ADMIN`）。
2. 你添加代码，使资源（例如 URL、控制器）需要特定的"属性"（例如 `ROLE_ADMIN` 等角色）才能被访问。

### 角色

当用户登录时，Symfony 调用 `User` 对象上的 `getRoles()` 方法来确定该用户拥有哪些角色。在之前生成的 `User` 类中，角色是存储在数据库中的数组，每个用户*始终*至少有一个角色：`ROLE_USER`：

```php
// src/Entity/User.php

// ...
class User implements UserInterface, PasswordAuthenticatedUserInterface
{
    /**
     * @var list<string> The user roles
     */
    #[ORM\Column]
    private array $roles = [];

    // ...
    public function getRoles(): array
    {
        $roles = $this->roles;
        // guarantee every user at least has ROLE_USER
        $roles[] = 'ROLE_USER';

        return array_unique($roles);
    }
}
```

这是一个很好的默认值，但你可以做*任何*你想做的事情来确定用户应该拥有哪些角色。唯一的规则是每个角色**必须以** `ROLE_` 前缀开头——否则，事情将不会按预期工作。除此之外，角色只是一个字符串，你可以发明任何你需要的角色（例如 `ROLE_PRODUCT_ADMIN`）。

你将使用这些角色来授予对网站特定部分的访问权限。

#### 层级角色

你可以通过创建角色层级来定义角色继承规则，而不是给每个用户分配许多角色：

```yaml
# config/packages/security.yaml
security:
    # ...

    role_hierarchy:
        ROLE_ADMIN:       ROLE_USER
        ROLE_SUPER_ADMIN: [ROLE_ADMIN, ROLE_ALLOWED_TO_SWITCH]
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'security' => [
        'role_hierarchy' => [
            'ROLE_ADMIN' => ['ROLE_USER'],
            'ROLE_SUPER_ADMIN' => ['ROLE_ADMIN', 'ROLE_ALLOWED_TO_SWITCH'],
        ],
    ],
]);
```

拥有 `ROLE_ADMIN` 角色的用户也将拥有 `ROLE_USER` 角色。拥有 `ROLE_SUPER_ADMIN` 的用户将自动拥有 `ROLE_ADMIN`、`ROLE_ALLOWED_TO_SWITCH` 和 `ROLE_USER`（继承自 `ROLE_ADMIN`）。

> **警告：** 要使角色层级正常工作，不要手动使用 `$user->getRoles()`。例如，在继承自基础控制器的控制器中：
>
> ```php
> // BAD - $user->getRoles() will not know about the role hierarchy
> $hasAccess = in_array('ROLE_ADMIN', $user->getRoles());
>
> // GOOD - use of the normal security methods
> $hasAccess = $this->isGranted('ROLE_ADMIN');
> $this->denyAccessUnlessGranted('ROLE_ADMIN');
> ```

> **注意：** `role_hierarchy` 值是静态的——例如，你不能将角色层级存储在数据库中。如果你需要这样做，请创建一个自定义安全投票器，在数据库中查找用户角色。

> **提示：** 为了帮助调试角色层级，你可以将其生成为 SVG 或 PNG 图像的可视化表示。首先，安装免费开源的 Mermaid CLI（提供 `mmdc` 命令），然后运行：
>
> ```bash
> $ php bin/console debug:security:role-hierarchy | mmdc -o roles.svg
> ```
>
> 然后，你可以打开 `roles.svg` 文件查看生成的图形。

### 添加代码以拒绝访问

有**两种**方式拒绝对某些内容的访问：

1. **security.yaml 中的 access_control**：允许你保护 URL 模式（例如 `/admin/*`）。更简单，但灵活性较低；
2. **在控制器（或其他代码）中**。

#### 保护 URL 模式（access_control）

在 `security.yaml` 中保护应用程序部分的最基本方法是保护整个 URL 模式。例如，要对所有以 `/admin` 开头的 URL 要求 `ROLE_ADMIN`，你可以：

```yaml
# config/packages/security.yaml
security:
    # ...

    firewalls:
        # ...
        main:
            # ...

    access_control:
        # require ROLE_ADMIN for /admin*
        - { path: '^/admin', roles: ROLE_ADMIN }

        # or require ROLE_ADMIN or IS_AUTHENTICATED_FULLY for /admin*
        - { path: '^/admin', roles: [IS_AUTHENTICATED_FULLY, ROLE_ADMIN] }

        # the 'path' value can be any valid regular expression
        # (this one will match URLs like /api/post/7298 and /api/comment/528491)
        - { path: ^/api/(post|comment)/\d+$, roles: ROLE_USER }
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'security' => [
        'firewalls' => [
            'main' => [
                // ...
            ],
        ],
        'access_control' => [
            // require ROLE_ADMIN for /admin*
            ['path' => '^/admin', 'roles' => 'ROLE_ADMIN'],

            // or require ROLE_ADMIN or IS_AUTHENTICATED_FULLY for /admin*
            ['path' => '^/admin', 'roles' => ['IS_AUTHENTICATED_FULLY', 'ROLE_ADMIN']],

            // the 'path' value can be any valid regular expression
            // (this one will match URLs like /api/post/7298 and /api/comment/528491)
            ['path' => '^/api/(post|comment)/\d+$', 'roles' => 'ROLE_USER'],
        ],
    ],
]);
```

你可以根据需要定义任意多个 URL 模式——每个都是正则表达式。**但是**，每个请求只会匹配**一个**：Symfony 从列表顶部开始，找到第一个匹配项时停止：

```yaml
# config/packages/security.yaml
security:
    # ...

    access_control:
        # matches /admin/users/*
        - { path: '^/admin/users', roles: ROLE_SUPER_ADMIN }

        # matches /admin/* except for anything matching the above rule
        - { path: '^/admin', roles: ROLE_ADMIN }
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'security' => [
        // ...

        'access_control' => [
            // matches /admin/users/*
            ['path' => '^/admin/users', 'roles' => 'ROLE_SUPER_ADMIN'],

            // matches /admin/* except for anything matching the above rule
            ['path' => '^/admin', 'roles' => 'ROLE_ADMIN'],
        ],
    ],
]);
```

在路径前加 `^` 意味着只匹配以该模式*开头*的 URL。例如，路径 `/admin`（没有 `^`）将匹配 `/admin/foo`，但也会匹配 `/foo/admin` 之类的 URL。

每个 `access_control` 还可以在 IP 地址、主机名和 HTTP 方法上进行匹配。它还可以用于将用户重定向到 URL 模式的 `https` 版本。对于更复杂的需求，你也可以使用实现 `RequestMatcherInterface` 的服务。

有关更多信息，请参阅访问控制文档。

#### 保护控制器和其他代码

你可以从控制器内部拒绝访问：

```php
// src/Controller/AdminController.php
// ...

public function adminDashboard(): Response
{
    $this->denyAccessUnlessGranted('ROLE_ADMIN');

    // or add an optional message - seen by developers
    $this->denyAccessUnlessGranted('ROLE_ADMIN', null, 'User tried to access a page without having ROLE_ADMIN');
}
```

就这样！如果访问未被授权，则会抛出一个特殊的 `Symfony\Component\Security\Core\Exception\AccessDeniedException`，并且控制器中不会再执行任何代码。然后，会发生以下两种情况之一：

1. 如果用户尚未登录，他们将被要求登录（例如重定向到登录页面）。

2. 如果用户*已*登录，但*没有* `ROLE_ADMIN` 角色，他们将看到 403 访问被拒绝页面（你可以自定义）。

保护一个或多个控制器操作的另一种方法是使用 `#[IsGranted]` 属性。在以下示例中，所有控制器操作都需要 `ROLE_ADMIN` 权限，除了 `adminDashboard()`，它需要 `ROLE_SUPER_ADMIN` 权限：

```php
// src/Controller/AdminController.php
// ...

use Symfony\Component\Security\Http\Attribute\IsGranted;

#[IsGranted('ROLE_ADMIN')]
class AdminController extends AbstractController
{
    // Optionally, you can set a custom message that will be displayed to the user
    #[IsGranted('ROLE_SUPER_ADMIN', message: 'You are not allowed to access the admin dashboard.')]
    public function adminDashboard(): Response
    {
        // ...
    }
}
```

你可以通过引用控制器参数的名称将任何控制器参数作为投票器主题传递。Symfony 会从控制器方法签名中自动解析它：

```php
// src/Controller/PostController.php
// ...

use App\Entity\Post;
use Symfony\Component\Security\Http\Attribute\IsGranted;

class PostController extends AbstractController
{
    #[Route('/posts/{id}/edit', name: 'post_edit')]
    // 'post' refers to the $post parameter of the controller method
    #[IsGranted('edit', 'post')]
    public function edit(Post $post): Response
    {
        // ...
    }
}
```

如果你想使用自定义状态码而不是默认的（403），可以通过设置 `statusCode` 参数来实现：

```php
// src/Controller/AdminController.php
// ...

use Symfony\Component\Security\Http\Attribute\IsGranted;

#[IsGranted('ROLE_ADMIN', statusCode: 423)]
class AdminController extends AbstractController
{
    // ...
}
```

你还可以设置抛出的 `Symfony\Component\Security\Core\Exception\AccessDeniedException` 的内部异常代码，使用 `exceptionCode` 参数：

```php
// src/Controller/AdminController.php
// ...

use Symfony\Component\Security\Http\Attribute\IsGranted;

#[IsGranted('ROLE_ADMIN', statusCode: 403, exceptionCode: 10010)]
class AdminController extends AbstractController
{
    // ...
}
```

你还可以扩展 `IsGranted` 属性以创建有意义的快捷方式：

```php
// src/Security/Attribute/IsAdmin.php
// ...

use Symfony\Component\Security\Http\Attribute\IsGranted;

class IsAdmin extends IsGranted
{
    public function __construct()
    {
        return parent::__construct('ROLE_ADMIN');
    }
}
```

你可以使用 `methods` 参数将访问验证限制为特定的 HTTP 方法：

```php
// src/Controller/AdminController.php
// ...

use Symfony\Component\Security\Http\Attribute\IsGranted;

#[IsGranted('ROLE_ADMIN', methods: 'POST')]
class AdminController extends AbstractController
{
    // You can also specify an array of methods
    #[IsGranted('ROLE_SUPER_ADMIN', methods: ['GET', 'PUT'])]
    public function adminDashboard(): Response
    {
        // ...
    }
}
```

#### 模板中的访问控制

如果你想检查当前用户是否具有某个角色，可以在任何 Twig 模板中使用内置的 `is_granted()` 辅助函数：

```twig
{% if is_granted('ROLE_ADMIN') %}
    <a href="...">Delete</a>
{% endif %}
```

类似地，如果你想检查特定用户是否具有某个角色，可以使用内置的 `is_granted_for_user()` 辅助函数：

```twig
{% if is_granted_for_user(user, 'ROLE_ADMIN') %}
    <a href="...">Delete</a>
{% endif %}
```

Symfony 还提供了 `access_decision()` 和 `access_decision_for_user()` Twig 函数，用于检查授权并在自定义安全投票器中检索拒绝权限的原因：

```twig
{% set voter_decision = access_decision('post_edit', post) %}
{% if voter_decision.isGranted %}
    {# ... #}
{% else %}
    {# before showing voter messages to end users, make sure it's safe to do so #}
    <p>{{ voter_decision.message }}</p>
{% endif %}

{% set voter_decision = access_decision('post_edit', post, anotherUser) %}
{% if voter_decision.isGranted %}
    {# ... #}
{% else %}
    <p>The {{ anotherUser.name }} user doesn't have sufficient permission:</p>
    {# before showing voter messages to end users, make sure it's safe to do so #}
    <p>{{ voter_decision.message }}</p>
{% endif %}
```

#### 保护其他服务

你可以通过注入 `Security` 服务在代码的*任何地方*检查访问权限。例如，假设你有一个 `SalesReportManager` 服务，你只想为拥有 `ROLE_SALES_ADMIN` 角色的用户包含额外的详细信息：

```diff
  // src/SalesReport/SalesReportManager.php

  // ...
  use Symfony\Component\Security\Core\Exception\AccessDeniedException;
+ use Symfony\Bundle\SecurityBundle\Security;

  class SalesReportManager
  {
+     public function __construct(
+         private Security $security,
+     ) {
+     }

      public function generateReport(): void
      {
          $salesData = [];

+         if ($this->security->isGranted('ROLE_SALES_ADMIN')) {
+             $salesData['top_secret_numbers'] = rand();
+         }

          // ...
      }

      // ...
  }
```

> **提示：** `isGranted()` 方法检查当前已登录用户的授权。如果你需要检查其他用户的授权，或者在用户会话不可用时（例如，在消息队列或 cron 作业等 CLI 上下文中），可以使用 `isGrantedForUser()` 方法显式设置目标用户。

你还可以使用 `getAccessDecision()` 和 `getAccessDecisionForUser()` 方法检查授权，并在自定义安全投票器中获取拒绝权限的原因：

```php
// src/SalesReport/SalesReportManager.php

// ...
use Symfony\Bundle\SecurityBundle\Security;

class SalesReportManager
{
    public function __construct(
        private Security $security,
    ) {
    }

    public function generateReport(): void
    {
        $voterDecision = $this->security->getAccessDecision('ROLE_SALES_ADMIN');
        if ($voterDecision->isGranted) {
            // ...
        } else {
            // do something with $voterDecision->getMessage()
        }

        // ...
    }

    // ...
}
```

如果你使用默认的 `services.yaml` 配置，Symfony 将通过自动装配和 `Security` 类型提示自动将 `security.helper` 传递给你的服务。

你还可以使用更底层的 `Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface` 服务。它与 `Security` 做相同的事情，但允许你对更具体的接口进行类型提示。

### 允许未保护访问（即匿名用户）

当访问者尚未登录你的网站时，他们被视为"未认证"，没有任何角色。如果你定义了 `access_control` 规则，这将阻止他们访问你的页面。

在 `access_control` 配置中，你可以使用 `PUBLIC_ACCESS` 安全属性来排除某些路由的未认证访问（例如登录页面）：

```yaml
# config/packages/security.yaml
security:

    # ...
    access_control:
        # allow unauthenticated users to access the login form
        - { path: ^/admin/login, roles: PUBLIC_ACCESS }

        # but require authentication for all other admin routes
        - { path: ^/admin, roles: ROLE_ADMIN }
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'security' => [
        // ...
        'access_control' => [
            // allow unauthenticated users to access the login form
            ['path' => '^/admin/login', 'roles' => 'PUBLIC_ACCESS'],

            // but require authentication for all other admin routes
            ['path' => '^/admin', 'roles' => 'ROLE_ADMIN'],
        ],
    ],
]);
```

### 在自定义投票器中授予匿名用户访问权限

如果你使用自定义投票器，可以通过检查令牌上是否未设置用户来允许匿名用户访问：

```php
// src/Security/PostVoter.php
namespace App\Security;

// ...
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authentication\User\UserInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Vote;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

class PostVoter extends Voter
{
    // ...

    protected function voteOnAttribute(string $attribute, $subject, TokenInterface $token, ?Vote $vote = null): bool
    {
        // ...

        if (!$token->getUser() instanceof UserInterface) {
            // the user is not authenticated, e.g. only allow them to
            // see public posts
            return $subject->isPublic();
        }
    }
}
```

### 设置个人用户权限

大多数应用程序需要更具体的访问规则。例如，用户应该只能编辑博客上*自己的*评论。投票器允许你编写任何业务逻辑来确定访问权限。使用这些投票器类似于前几章中实现的基于角色的访问检查。阅读投票器文档以了解如何实现自己的投票器。

### 检查用户是否已登录

如果你*只*想检查用户是否已登录（你不关心角色），有以下两个选项。

首先，如果你给*每个*用户都分配了 `ROLE_USER`，你可以检查该角色。

其次，你可以使用特殊的"属性" `IS_AUTHENTICATED` 代替角色：

```php
// ...

public function adminDashboard(): Response
{
    $this->denyAccessUnlessGranted('IS_AUTHENTICATED');

    // ...
}
```

你可以在使用角色的任何地方使用 `IS_AUTHENTICATED`：如 `access_control` 或 Twig 中。

`IS_AUTHENTICATED` 不是角色，但它有点像角色，每个已登录的用户都会拥有它。实际上，有一些类似的特殊属性：

- `IS_AUTHENTICATED_FULLY`：这类似于 `IS_AUTHENTICATED_REMEMBERED`，但更强。仅因为"记住我 Cookie"而登录的用户将拥有 `IS_AUTHENTICATED_REMEMBERED`，但不会拥有 `IS_AUTHENTICATED_FULLY`。

- `IS_REMEMBERED`：*仅*使用记住我功能进行身份认证的用户（即记住我 Cookie）。

- `IS_IMPERSONATOR`：当当前用户在此会话中模拟另一个用户时，此属性将匹配。

## 了解如何从会话中刷新用户

在每次请求结束时（除非你的防火墙是 `stateless`），你的 `User` 对象会被序列化到会话中。在下一次请求开始时，它会被反序列化，然后传递给你的用户提供者以"刷新"它（例如 Doctrine 查询获取最新用户）。

然后，两个用户对象（来自会话的原始对象和刷新后的用户对象）会被"比较"以查看它们是否"相等"。默认情况下，核心 `AbstractToken` 类比较 `getPassword()`、`getSalt()` 和 `getUserIdentifier()` 方法的返回值。如果其中任何一个不同，你的用户将被登出。这是一种安全措施，以确保如果核心用户数据发生变化，恶意用户可以被取消认证。

在会话中存储（明文或哈希后的）密码可能是一种安全风险。为了缓解这一问题，在用户类中实现 `__serialize()` 魔术方法，以便在将序列化的用户对象存储到会话之前排除或转换密码。

支持两种策略：

1. 完全删除密码。反序列化后，`getPassword()` 返回 `null`，Symfony 在不检查密码的情况下刷新用户。仅在你存储明文密码时使用（不推荐）。
2. 使用 `crc32c` 算法哈希密码。Symfony 将对刷新后用户的密码进行哈希，并将其与会话值进行比较。这种方法避免了存储真实哈希，并允许你在密码更改时使会话失效。

示例（假设密码存储在名为 `password` 的私有属性中）：

```php
public function __serialize(): array
{
    $data = (array) $this;
    $data["\0".self::class."\0password"] = hash('crc32c', $this->password);

    return $data;
}
```

如果你在身份认证方面遇到问题，可能是你*确实*成功地进行了身份认证，但在第一次重定向后立即失去了身份认证。

在这种情况下，请检查用户类上的序列化逻辑（例如 `__serialize()` 或 `serialize()` 方法，如果有的话），以确保所有必要的字段都被序列化，并且排除所有不需要序列化的字段（例如 Doctrine 关系）。

### 使用 EquatableInterface 手动比较用户

或者，如果你需要对"比较用户"过程有更多控制，请让你的用户类实现 `Symfony\Component\Security\Core\User\EquatableInterface`。然后，在比较用户时将调用你的 `isEqualTo()` 方法，而不是核心逻辑。

## 安全事件

在身份认证过程中，会分发多个事件，允许你钩入该过程或自定义发送给用户的响应。你可以通过为这些事件创建事件监听器或订阅者来实现。

> **提示：** 每个安全防火墙都有自己的事件分发器（`security.event_dispatcher.FIREWALLNAME`）。事件在全局和防火墙特定的分发器上都会分发。如果你只希望监听器在特定防火墙中被调用，可以在防火墙分发器上注册。例如，如果你有 `api` 和 `main` 防火墙，使用以下配置仅在 `main` 防火墙的登出事件上注册：
>
> ```yaml
> # config/services.yaml
> services:
>     # ...
>
>     App\EventListener\LogoutSubscriber:
>         tags:
>             - name: kernel.event_subscriber
>               dispatcher: security.event_dispatcher.main
> ```
>
> ```php
> // config/services.php
> namespace Symfony\Component\DependencyInjection\Loader\Configurator;
>
> use App\EventListener\LogoutSubscriber;
>
> return [
>     'services' => [
>         LogoutSubscriber::class => [
>             'tags' => [
>                 ['kernel.event_subscriber' => ['dispatcher' => 'security.event_dispatcher.main']],
>             ],
>         ],
>     ],
> ]);
> ```

### 认证事件

**`Symfony\Component\Security\Http\Event\CheckPassportEvent`**
> 在认证器创建安全通行证后分发。此事件的监听器执行实际的认证检查（如检查通行证、验证 CSRF 令牌等）。

**`Symfony\Component\Security\Http\Event\AuthenticationTokenCreatedEvent`**
> 在通行证验证完成且认证器创建了安全令牌（和用户）后分发。这可用于需要修改创建的令牌的高级用例（例如用于多因素认证）。

**`Symfony\Component\Security\Core\Event\AuthenticationSuccessEvent`**
> 在认证接近成功时分发。这是可以通过抛出 `AuthenticationException` 使认证失败的最后一个事件。

**`Symfony\Component\Security\Http\Event\LoginSuccessEvent`**
> 在身份认证完全成功后分发。此事件的监听器可以修改发送给用户的响应。

**`Symfony\Component\Security\Http\Event\LoginFailureEvent`**
> 在身份认证期间抛出 `AuthenticationException` 后分发。此事件的监听器可以修改发送给用户的错误响应。

### 其他事件

**`Symfony\Component\Security\Http\Event\InteractiveLoginEvent`**
> 仅当认证器实现 `Symfony\Component\Security\Http\Authenticator\InteractiveAuthenticatorInterface`（表示登录需要显式用户操作，例如登录表单）时，在身份认证完全成功后分发。此事件的监听器可以修改发送给用户的响应。

**`Symfony\Component\Security\Http\Event\LogoutEvent`**
> 在用户从应用程序注销前分发。请参阅登出部分。

**`Symfony\Component\Security\Http\Event\TokenDeauthenticatedEvent`**
> 在用户被取消认证时分发，例如因为密码已更改。请参阅用户会话刷新部分。

**`Symfony\Component\Security\Http\Event\SwitchUserEvent`**
> 在模拟完成后分发。请参阅用户模拟文档。

## 常见问题

**我可以有多个防火墙吗？**
> 可以！但是，每个防火墙就像一个独立的安全系统：在一个防火墙中通过身份认证并不意味着在另一个防火墙中也通过了认证。每个防火墙可以有多种允许身份认证的方式（例如表单登录和 API 密钥身份认证）。如果你想在防火墙之间共享身份认证，则必须为不同的防火墙显式指定相同的 `reference-security-firewall-context`。

**安全似乎不适用于我的错误页面**
> 由于路由在安全之前完成，404 错误页面不受任何防火墙覆盖。这意味着你无法检查安全或甚至在这些页面上访问用户对象。有关更多详细信息，请参阅错误页面文档。

**我的身份认证似乎不起作用：没有错误，但我从未登录**
> 有时身份认证可能成功，但在重定向后，由于从会话加载 `User` 时出现问题，你立即被登出。要查看是否存在这个问题，请检查日志文件（`var/log/dev.log`）中的日志消息。

**无法刷新令牌，因为用户已更改**
> 如果你看到此消息，可能有两种原因。首先，可能存在从会话加载用户的问题，请参阅用户会话刷新部分。其次，如果自上次页面刷新以来数据库中的某些用户信息已更改，Symfony 出于安全原因会故意将用户登出。

## 了解更多

### 认证（识别/登录用户）

- security/passwords
- security/ldap
- security/remember_me
- security/impersonating_user
- security/user_checkers
- security/firewall_restriction
- security/csrf
- security/form_login
- security/custom_authenticator
- security/entry_point

### 授权（拒绝访问）

- security/voters
- security/access_control
- security/expressions
- security/access_denied_handler
- security/force_https
