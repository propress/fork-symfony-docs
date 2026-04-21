# 会话（Sessions）

Symfony HttpFoundation 组件拥有一个功能强大且灵活的会话子系统，其设计目标是通过清晰的面向对象接口并利用多种会话存储驱动，在请求之间存储用户信息，从而提供完善的会话管理。

Symfony 会话旨在替代 `$_SESSION` 超全局变量以及与操作会话相关的原生 PHP 函数，例如 `session_start()`、`session_regenerate_id()`、`session_id()`、`session_name()` 和 `session_destroy()`。

> **注意：** 会话仅在你读取或写入时才会启动。

## 安装

你需要安装 HttpFoundation 组件来处理会话：

```bash
$ composer require symfony/http-foundation
```

## 基本用法

会话可通过 `Request` 对象和 `RequestStack` 服务访问。如果你为参数添加 `Symfony\Component\HttpFoundation\RequestStack` 类型提示，Symfony 会在服务和控制器中自动注入 `request_stack` 服务：

**Symfony 应用：**

```php
use Symfony\Component\HttpFoundation\RequestStack;

class SomeService
{
    public function __construct(
        private RequestStack $requestStack,
    ) {
        // 不建议在构造函数中访问会话，因为此时会话可能尚未可用，
        // 或者会导致意外的副作用
        // $this->session = $requestStack->getSession();
    }

    public function someMethod(): void
    {
        $session = $this->requestStack->getSession();

        // ...
    }
}
```

**独立应用：**

```php
use Symfony\Component\HttpFoundation\Session\Session;

$session = new Session();
$session->start();
```

在 Symfony 控制器中，你也可以为参数添加 `Symfony\Component\HttpFoundation\Request` 类型提示：

```php
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

public function index(Request $request): Response
{
    $session = $request->getSession();

    // ...
}
```

## 会话属性

PHP 的会话管理需要使用 `$_SESSION` 超全局变量。然而，这会干扰面向对象范式中代码的可测试性和封装性。为了解决这个问题，Symfony 使用与会话关联的*会话包（session bags）*来封装特定的**属性**数据集。

这种方法可以减少 `$_SESSION` 超全局变量中的命名空间污染，因为每个包都将其所有数据存储在唯一的命名空间下。这使得 Symfony 能够与其他可能使用 `$_SESSION` 超全局变量的应用程序或库和平共存，并且所有数据与 Symfony 的会话管理完全兼容。

会话包是一个行为类似数组的 PHP 对象：

```php
// 存储属性以供用户后续请求时复用
$session->set('attribute-name', 'attribute-value');

// 通过名称获取属性
$foo = $session->get('foo');

// 第二个参数是属性不存在时的返回值
$filters = $session->get('filters', []);
```

存储的属性在用户会话的剩余时间内保持有效。默认情况下，会话属性是由 `Symfony\Component\HttpFoundation\Session\Attribute\AttributeBag` 类管理的键值对。

每当你读取、写入或检查会话中的数据是否存在时，会话都会自动启动。这可能会影响你的应用程序性能，因为所有用户都会收到一个会话 cookie。为了避免为匿名用户启动会话，你必须*完全*避免访问会话。

> **注意：** 当使用内部依赖会话的功能时，例如表单中的有状态 CSRF 保护，会话也会被启动。

## 闪存消息

你可以在用户的会话中存储称为"闪存"消息的特殊消息。从设计上看，闪存消息仅使用一次：一旦你检索它们，它们就会自动从会话中消失。这一特性使得"闪存"消息特别适合用于存储用户通知。

例如，假设你正在处理一个表单提交：

**Symfony 应用：**

```php
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
// ...

public function update(Request $request): Response
{
    // ...

    if ($form->isSubmitted() && $form->isValid()) {
        // 进行某种处理

        $this->addFlash(
            'notice',
            'Your changes were saved!'
        );
        // $this->addFlash() 等同于 $request->getSession()->getFlashBag()->add()

        return $this->redirectToRoute(/* ... */);
    }

    return $this->render(/* ... */);
}
```

**独立应用：**

```php
use Symfony\Component\HttpFoundation\Session\Session;

$session = new Session();
$session->start();

// 获取闪存消息包
$flashes = $session->getFlashBag();

// 添加闪存消息
$flashes->add(
    'notice',
    'Your changes were saved'
);
```

处理完请求后，控制器在会话中设置一条闪存消息，然后重定向。消息键（本示例中为 `notice`）可以是任意内容，你将使用该键来检索消息。

在下一个页面的模板中（或者更好的是，在你的基础布局模板中），使用 Twig 全局 `app` 变量提供的 `flashes()` 方法从会话中读取所有闪存消息。或者，你也可以使用 `Symfony\Component\HttpFoundation\Session\Flash\FlashBagInterface::peek` 方法检索消息，同时将其保留在包中：

**Twig 模板：**

```twig
{# templates/base.html.twig #}

{# 读取并显示单一类型的闪存消息 #}
{% for message in app.flashes('notice') %}
    <div class="flash-notice">
        {{ message }}
    </div>
{% endfor %}

{# 相同操作，但不从闪存包中清除消息 #}
{% for message in app.session.flashbag.peek('notice') %}
    <div class="flash-notice">
        {{ message }}
    </div>
{% endfor %}

{# 读取并显示多种类型的闪存消息 #}
{% for label, messages in app.flashes(['success', 'warning']) %}
    {% for message in messages %}
        <div class="flash-{{ label }}">
            {{ message }}
        </div>
    {% endfor %}
{% endfor %}

{# 读取并显示所有闪存消息 #}
{% for label, messages in app.flashes %}
    {% for message in messages %}
        <div class="flash-{{ label }}">
            {{ message }}
        </div>
    {% endfor %}
{% endfor %}

{# 或者不清除闪存包 #}
{% for label, messages in app.session.flashbag.peekAll() %}
    {% for message in messages %}
        <div class="flash-{{ label }}">
            {{ message }}
        </div>
    {% endfor %}
{% endfor %}
```

**独立应用：**

```php
// 显示警告
foreach ($session->getFlashBag()->get('warning', []) as $message) {
    echo '<div class="flash-warning">'.$message.'</div>';
}

// 显示警告，但不从闪存包中清除
foreach ($session->getFlashBag()->peek('warning', []) as $message) {
    echo '<div class="flash-warning">'.$message.'</div>';
}

// 显示错误
foreach ($session->getFlashBag()->get('error', []) as $message) {
    echo '<div class="flash-error">'.$message.'</div>';
}

// 一次性显示所有闪存消息
foreach ($session->getFlashBag()->all() as $type => $messages) {
    foreach ($messages as $message) {
        echo '<div class="flash-'.$type.'">'.$message.'</div>';
    }
}

// 一次性显示所有闪存消息，但不清除闪存包
foreach ($session->getFlashBag()->peekAll() as $type => $messages) {
    foreach ($messages as $message) {
        echo '<div class="flash-'.$type.'">'.$message.'</div>';
    }
}
```

通常使用 `notice`、`warning` 和 `error` 作为不同类型闪存消息的键，但你可以使用任何适合你需求的键。

> **提示：** 访问闪存消息需要启动会话，这反过来会导致 Symfony 将响应标记为 `private`。一般来说，由于闪存消息只显示一次，可能显示这些消息的页面无法被 HTTP 缓存合理地缓存。
>
> 作为替代方案，你可以通过另一个 HTTP 请求（例如使用 [Twig Live Component](https://symfony.com/bundles/ux-live-component/current/index.html)）异步加载闪存消息，从而使原始页面完全可缓存。

## 配置

在 Symfony 框架中，会话默认已启用。会话存储和其他配置可以通过 `config/packages/framework.yaml` 中的 `framework.session` 配置来控制：

**YAML：**

```yaml
# config/packages/framework.yaml
framework:
    # 启用会话支持。注意，仅当你读取或写入会话时，会话才会启动。
    # 删除或注释此部分可以明确禁用会话支持。
    session:
        # 用于会话存储的服务 ID
        # NULL 表示 Symfony 使用 PHP 默认的会话机制
        handler_id: null
        # 提高会话所用 cookie 的安全性
        cookie_secure: auto
        cookie_samesite: lax
        storage_factory_id: session.storage.factory.native
```

**PHP：**

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\HttpFoundation\Cookie;

return App::config([
    'framework' => [
        'session' => [
            // 启用会话支持。注意，仅当你读取或写入会话时，会话才会启动。
            // 删除或注释此部分可以明确禁用会话支持。
            'enabled' => true,
            // 用于会话存储的服务 ID
            // NULL 表示 Symfony 使用 PHP 默认的会话机制
            'handler_id' => null,
            // 提高会话所用 cookie 的安全性
            'cookie_secure' => 'auto',
            'cookie_samesite' => Cookie::SAMESITE_LAX,
            'storage_factory_id' => 'session.storage.factory.native',
        ],
    ],
]);
```

**独立应用：**

```php
use Symfony\Component\HttpFoundation\Cookie;
use Symfony\Component\HttpFoundation\Session\Attribute\AttributeBag;
use Symfony\Component\HttpFoundation\Session\Session;
use Symfony\Component\HttpFoundation\Session\Storage\NativeSessionStorage;

$storage = new NativeSessionStorage([
    'cookie_secure' => 'auto',
    'cookie_samesite' => Cookie::SAMESITE_LAX,
]);
$session = new Session($storage);
```

将 `handler_id` 配置选项设置为 `null` 意味着 Symfony 将使用原生 PHP 会话机制。会话元数据文件将存储在 Symfony 应用程序外部、由 PHP 控制的目录中。虽然这通常会简化事情，但如果其他写入同一目录的应用程序有较短的最大生命周期设置，某些与会话过期相关的选项可能无法按预期工作。

如果你愿意，可以使用 `session.handler.native_file` 服务作为 `handler_id`，让 Symfony 自行管理会话。另一个有用的选项是 `save_path`，它定义了 Symfony 存储会话元数据文件的目录：

**YAML：**

```yaml
# config/packages/framework.yaml
framework:
    session:
        # ...
        handler_id: 'session.handler.native_file'
        save_path: '%kernel.project_dir%/var/sessions/%kernel.environment%'
```

**PHP：**

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'session' => [
            // ...
            'handler_id' => 'session.handler.native_file',
            'save_path' => '%kernel.project_dir%/var/sessions/%kernel.environment%',
        ],
    ],
]);
```

**独立应用：**

```php
use Symfony\Component\HttpFoundation\Cookie;
use Symfony\Component\HttpFoundation\Session\Attribute\AttributeBag;
use Symfony\Component\HttpFoundation\Session\Session;
use Symfony\Component\HttpFoundation\Session\Storage\Handler\NativeFileSessionHandler;
use Symfony\Component\HttpFoundation\Session\Storage\NativeSessionStorage;

$handler = new NativeFileSessionHandler('/var/sessions');
$storage = new NativeSessionStorage([], $handler);
$session = new Session($storage);
```

查阅 Symfony 配置参考，了解更多其他可用的会话配置选项。

> **警告：** Symfony 会话与 `php.ini` 指令 `session.auto_start = 1` 不兼容。应在 `php.ini`、Web 服务器指令或 `.htaccess` 中关闭此指令。

会话 cookie 也可以在响应对象中获取。这在 CLI 上下文或使用 Roadrunner、Swoole 等 PHP 运行器时非常有用。

### 会话空闲时间/保活

在某些情况下，你可能需要在用户登录后离开终端时通过在一段空闲时间后销毁会话来保护或最小化会话的未授权使用。例如，银行应用程序通常在仅 5 到 10 分钟不活动后注销用户。在这里设置 cookie 生命周期是不合适的，因为这可以被客户端操纵，所以我们必须在服务器端处理过期。最简单的方法是通过会话垃圾回收来实现，它运行得足够频繁。`cookie_lifetime` 会被设置为一个相对较高的值，而垃圾回收 `gc_maxlifetime` 会被设置为在所需的空闲时间后销毁会话。

另一个选项是在会话启动后专门检查会话是否已过期，然后根据需要销毁会话。这种处理方式可以将会话过期集成到用户体验中，例如，通过显示一条消息。

Symfony 记录每个会话的一些元数据，以便对安全设置进行精细控制：

```php
$session->getMetadataBag()->getCreated();
$session->getMetadataBag()->getLastUsed();
```

这两个方法都返回 Unix 时间戳（相对于服务器时间）。

此元数据可用于在访问时明确使会话过期：

```php
$session->start();
if (time() - $session->getMetadataBag()->getLastUsed() > $maxIdleTime) {
    $session->invalidate();
    throw new SessionExpired(); // 重定向到会话过期页面
}
```

也可以通过读取 `getLifetime()` 方法来了解特定 cookie 的 `cookie_lifetime` 设置：

```php
$session->getMetadataBag()->getLifetime();
```

cookie 的过期时间可以通过将创建时间戳与生命周期相加来确定。

### 配置垃圾回收

当会话打开时，PHP 会根据 `session.gc_probability` / `session.gc_divisor` 设置的概率随机调用 `gc` 处理器。例如，如果分别设置为 `5/100`，则意味着概率为 5%。类似地，`3/4` 意味着 4 次中有 3 次被调用，即 75%。

如果调用了垃圾回收处理器，PHP 将传递 `php.ini` 指令 `session.gc_maxlifetime` 中存储的值。在此上下文中，意思是任何保存时间超过 `gc_maxlifetime` 的存储会话都应该被删除。这允许根据空闲时间使记录过期。

然而，某些操作系统（例如 Debian）以不同方式管理会话处理，并将 `session.gc_probability` 变量设置为 `0` 以防止 PHP 执行垃圾回收。默认情况下，Symfony 使用 `php.ini` 文件中设置的 `gc_probability` 指令的值。如果你无法修改此 PHP 设置，可以直接在 Symfony 中配置它：

```yaml
# config/packages/framework.yaml
framework:
    session:
        # ...
        gc_probability: 1
```

或者，你也可以通过将 `gc_probability`、`gc_divisor` 和 `gc_maxlifetime` 作为数组传递给 `Symfony\Component\HttpFoundation\Session\Storage\NativeSessionStorage` 的构造函数或 `Symfony\Component\HttpFoundation\Session\Storage\NativeSessionStorage::setOptions` 方法来配置这些设置。

## 在数据库中存储会话

Symfony 默认将会话存储在文件中。如果你的应用程序由多台服务器提供服务，你需要使用数据库来使会话在不同服务器之间正常工作。

Symfony 可以将会话存储在各种数据库中（关系型数据库、NoSQL 数据库和键值数据库），但推荐使用 Redis 等键值数据库以获得最佳性能。

### 在键值数据库（Redis）中存储会话

本节假设你已有一个完全正常运行的 Redis 服务器，并且已安装和配置了 [phpredis 扩展](https://github.com/phpredis/phpredis)。

你有两种不同的选项来使用 Redis 存储会话：

第一种基于 PHP 的选项是直接在服务器 `php.ini` 文件中配置 Redis 会话处理器：

```ini
; php.ini
session.save_handler = redis
session.save_path = "tcp://192.168.0.178:6379?auth=REDIS_PASSWORD"
```

第二种选项是在 Symfony 中配置 Redis 会话。首先，为连接 Redis 服务器定义一个 Symfony 服务：

**YAML：**

```yaml
# config/services.yaml
services:
    # ...
    Symfony\Component\HttpFoundation\Session\Storage\Handler\RedisSessionHandler:
        arguments:
            - '@Redis'
            # 你可以选择性地传递一个选项数组。唯一的选项是 'prefix' 和 'ttl'，
            # 它们定义了用于避免 Redis 服务器上键冲突的前缀
            # 以及任意给定条目的过期时间（以秒为单位），默认值分别为 'sf_s' 和 null：
            # - { 'prefix': 'my_prefix', 'ttl': 600 }

    Redis:
        # 你也可以使用 \RedisArray、\RedisCluster、\Relay\Relay 或 \Predis\Client 类
        class: \Redis
        calls:
            - connect:
                - '%env(REDIS_HOST)%'
                - '%env(int:REDIS_PORT)%'

            # 如果你的 Redis 服务器需要密码，请取消注释以下内容
            # - auth:
            #     - '%env(REDIS_PASSWORD)%'

            # 如果你的 Redis 服务器需要用户名和密码（当用户不是 default 时），请取消注释以下内容
            # - auth:
            #     - ['%env(REDIS_USER)%','%env(REDIS_PASSWORD)%']
```

**PHP：**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\HttpFoundation\Session\Storage\Handler\RedisSessionHandler;

return App::config([
    'services' => [
        // ...
        RedisSessionHandler::class => [
            'arguments' => [
                service('Redis'),
                // 你可以选择性地传递一个选项数组。唯一的选项是 'prefix' 和 'ttl'，
                // 它们定义了用于避免 Redis 服务器上键冲突的前缀
                // 以及任意给定条目的过期时间（以秒为单位），默认值分别为 'sf_s' 和 null：
                // ['prefix' => 'my_prefix', 'ttl' => 600],
            ],
        ],
        \Redis::class => [
            // 你也可以使用 \RedisArray、\RedisCluster、\Relay\Relay 或 \Predis\Client 类
            'class' => \Redis::class,
            'calls' => [
                'connect' => [env('REDIS_HOST'), env('REDIS_PORT')->int()],
                // 如果你的 Redis 服务器需要密码，请取消注释以下内容：
                // 'auth' => [env('REDIS_PASSWORD')],
                // 如果你的 Redis 服务器需要用户名和密码（当用户不是 default 时），请取消注释以下内容：
                // 'auth' => [env('REDIS_USER'), env('REDIS_PASSWORD')],
            ],
        ],
    ],
]);
```

接下来，使用 `handler_id` 配置选项告知 Symfony 使用此服务作为会话处理器：

**YAML：**

```yaml
# config/packages/framework.yaml
framework:
    # ...
    session:
        handler_id: Symfony\Component\HttpFoundation\Session\Storage\Handler\RedisSessionHandler
```

**PHP：**

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\HttpFoundation\Session\Storage\Handler\RedisSessionHandler;

return App::config([
    'framework' => [
        // ...
        'session' => [
            'handler_id' => RedisSessionHandler::class,
        ],
    ],
]);
```

Symfony 现在将使用你的 Redis 服务器来读取和写入会话数据。此方案的主要缺点是 Redis 不执行会话锁定，因此在访问会话时可能会面临*竞态条件*。例如，你可能会看到"无效的 CSRF 令牌"错误，因为两个请求并行发出，而只有第一个请求将 CSRF 令牌存储到了会话中。

> **另请参阅：** 如果你使用 Memcached 而不是 Redis，请遵循类似的方法，但将 `RedisSessionHandler` 替换为 `Symfony\Component\HttpFoundation\Session\Storage\Handler\MemcachedSessionHandler`。

> **提示：** 在 `handler_id` 配置选项中使用带有 DSN 的 Redis 时，你可以在 DSN 中以查询字符串参数的形式添加 `prefix` 和 `ttl` 选项。

> **提示：** 使用 [Valkey](https://valkey.io/) 服务器时，你可以在会话处理器配置中使用 `valkey:` 或 `valkeys:` DSN 方案，而不是 `redis:` 或 `rediss:`。

### 在关系型数据库（MariaDB、MySQL、PostgreSQL）中存储会话

Symfony 包含一个 `Symfony\Component\HttpFoundation\Session\Storage\Handler\PdoSessionHandler` 来将会话存储在 MariaDB、MySQL 和 PostgreSQL 等关系型数据库中。要使用它，首先使用你的数据库凭据注册一个新的处理器服务：

**YAML：**

```yaml
# config/services.yaml
services:
    # ...

    Symfony\Component\HttpFoundation\Session\Storage\Handler\PdoSessionHandler:
        arguments:
            - '%env(DATABASE_URL)%'

            # 你也可以使用 PDO 配置，但需要传递两个参数
            # - 'mysql:dbname=mydatabase; host=myhost; port=myport'
            # - { db_username: myuser, db_password: mypassword }
```

**PHP：**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\HttpFoundation\Session\Storage\Handler\PdoSessionHandler;

return App::config([
    'services' => [
        // ...
        PdoSessionHandler::class => [
            'arguments' => [
                env('DATABASE_URL'),
                // 你也可以使用 PDO 配置，但需要传递两个参数
                // 'mysql:dbname=mydatabase; host=myhost; port=myport',
                // ['db_username' => 'myuser', 'db_password' => 'mypassword'],
            ],
        ],
    ],
]);
```

> **提示：** 当使用 MySQL 作为数据库时，`DATABASE_URL` 中定义的 DSN 可以包含 `charset` 和 `unix_socket` 选项作为查询字符串参数。

接下来，使用 `handler_id` 配置选项告知 Symfony 使用此服务作为会话处理器：

**YAML：**

```yaml
# config/packages/framework.yaml
framework:
    session:
        # ...
        handler_id: Symfony\Component\HttpFoundation\Session\Storage\Handler\PdoSessionHandler
```

**PHP：**

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\HttpFoundation\Session\Storage\Handler\PdoSessionHandler;

return App::config([
    'framework' => [
    // ...
        'session' => [
            'handler_id' => PdoSessionHandler::class,
        ],
    ],
]);
```

#### 配置会话表和列名

用于存储会话的表默认称为 `sessions`，并定义了特定的列名。你可以通过传递给 `PdoSessionHandler` 服务的第二个参数来配置这些值：

**YAML：**

```yaml
# config/services.yaml
services:
    # ...

    Symfony\Component\HttpFoundation\Session\Storage\Handler\PdoSessionHandler:
        arguments:
            - '%env(DATABASE_URL)%'
            - { db_table: 'customer_session', db_id_col: 'guid' }
```

**PHP：**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\HttpFoundation\Session\Storage\Handler\PdoSessionHandler;

return App::config([
    'services' => [
        // ...
        PdoSessionHandler::class => [
            'arguments' => [
                env('DATABASE_URL'),
                ['db_table' => 'customer_session', 'db_id_col' => 'guid'],
            ],
        ],
    ],
]);
```

以下是你可以配置的参数：

**`db_table`**（默认值 `sessions`）：数据库中会话表的名称。

**`db_username`**（默认值：`''`）：使用 PDO 配置连接时的用户名（当使用基于 `DATABASE_URL` 环境变量的连接时，它会覆盖环境变量中定义的用户名）。

**`db_password`**（默认值：`''`）：使用 PDO 配置连接时的密码（当使用基于 `DATABASE_URL` 环境变量的连接时，它会覆盖环境变量中定义的密码）。

**`db_id_col`**（默认值 `sess_id`）：用于存储会话 ID 的列名（列类型：`VARCHAR(128)`）。

**`db_data_col`**（默认值 `sess_data`）：用于存储会话数据的列名（列类型：`BLOB`）。

**`db_time_col`**（默认值 `sess_time`）：用于存储会话创建时间戳的列名（列类型：`INTEGER`）。

**`db_lifetime_col`**（默认值 `sess_lifetime`）：用于存储会话生命周期的列名（列类型：`INTEGER`）。

**`db_connection_options`**（默认值：`[]`）：驱动程序特定的连接选项数组。

**`lock_mode`**（默认值：`LOCK_TRANSACTIONAL`）：锁定数据库以避免*竞态条件*的策略。可能的值为 `LOCK_NONE`（无锁定）、`LOCK_ADVISORY`（应用程序级锁定）和 `LOCK_TRANSACTIONAL`（行级锁定）。

#### 准备数据库以存储会话

在数据库中存储会话之前，你必须创建存储信息的表。

安装 Doctrine 后，如果 doctrine 所针对的数据库与此组件使用的数据库相同，当你运行 `make:migration` 命令时，会话表将自动生成。

或者，如果你希望自己创建表且该表尚未创建，会话处理器提供了一个名为 `Symfony\Component\HttpFoundation\Session\Storage\Handler\PdoSessionHandler::createTable` 的方法，可根据所使用的数据库引擎为你设置此表：

```php
try {
    $sessionHandlerService->createTable();
} catch (\PDOException $exception) {
    // 由于某种原因无法创建表
}
```

如果表已存在，则会抛出异常。

如果你宁愿自己设置表，建议使用以下命令生成一个空的数据库迁移：

```bash
$ php bin/console doctrine:migrations:generate
```

然后，在下面找到适合你的数据库的 SQL，将其添加到迁移文件中，并使用以下命令运行迁移：

```bash
$ php bin/console doctrine:migrations:migrate
```

如果需要，你也可以通过在代码中调用 `Symfony\Component\HttpFoundation\Session\Storage\Handler\PdoSessionHandler::configureSchema` 方法将此表添加到你的 schema 中。

##### MariaDB/MySQL

```sql
CREATE TABLE `sessions` (
    `sess_id` VARBINARY(128) NOT NULL PRIMARY KEY,
    `sess_data` BLOB NOT NULL,
    `sess_lifetime` INTEGER UNSIGNED NOT NULL,
    `sess_time` INTEGER UNSIGNED NOT NULL,
    INDEX `sess_lifetime_idx` (`sess_lifetime`)
) COLLATE utf8mb4_bin, ENGINE = InnoDB;
```

> **注意：** `BLOB` 列类型（`createTable()` 默认使用的类型）最多存储 64 KB。如果用户会话数据超过此大小，可能会抛出异常或会话将被静默重置。如果你需要更多空间，请考虑使用 `MEDIUMBLOB`。

##### PostgreSQL

```sql
CREATE TABLE sessions (
    sess_id VARCHAR(128) NOT NULL PRIMARY KEY,
    sess_data BYTEA NOT NULL,
    sess_lifetime INTEGER NOT NULL,
    sess_time INTEGER NOT NULL
);
CREATE INDEX sess_lifetime_idx ON sessions (sess_lifetime);
```

##### Microsoft SQL Server

```sql
CREATE TABLE sessions (
    sess_id VARCHAR(128) NOT NULL PRIMARY KEY,
    sess_data NVARCHAR(MAX) NOT NULL,
    sess_lifetime INTEGER NOT NULL,
    sess_time INTEGER NOT NULL,
    INDEX sess_lifetime_idx (sess_lifetime)
);
```

### 在 NoSQL 数据库（MongoDB）中存储会话

Symfony 包含一个 `Symfony\Component\HttpFoundation\Session\Storage\Handler\MongoDbSessionHandler` 来将会话存储在 MongoDB NoSQL 数据库中。首先，确保在你的 Symfony 应用程序中有一个正常工作的 MongoDB 连接，具体说明参见 [DoctrineMongoDBBundle 配置](https://symfony.com/doc/master/bundles/DoctrineMongoDBBundle/config.html)文章。

然后，为 `MongoDbSessionHandler` 注册一个新的处理器服务，并将 MongoDB 连接作为参数传递，以及所需的参数：

**`database`**：数据库的名称。

**`collection`**：集合的名称。

**YAML：**

```yaml
# config/services.yaml
services:
    # ...

    Symfony\Component\HttpFoundation\Session\Storage\Handler\MongoDbSessionHandler:
        arguments:
            - '@doctrine_mongodb.odm.default_connection'
            - { database: '%env(MONGODB_DB)%', collection: 'sessions' }
```

**PHP：**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\HttpFoundation\Session\Storage\Handler\MongoDbSessionHandler;

return App::config([
    'services' => [
        // ...
        MongoDbSessionHandler::class => [
            'arguments' => [
                service('doctrine_mongodb.odm.default_connection'),
                ['database' => env('MONGODB_DB'), 'collection' => 'sessions'],
            ],
        ],
    ],
]);
```

接下来，使用 `handler_id` 配置选项告知 Symfony 使用此服务作为会话处理器：

**YAML：**

```yaml
# config/packages/framework.yaml
framework:
    session:
        # ...
        handler_id: Symfony\Component\HttpFoundation\Session\Storage\Handler\MongoDbSessionHandler
```

**PHP：**

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\HttpFoundation\Session\Storage\Handler\MongoDbSessionHandler;

return App::config([
    'framework' => [
        // ...
        'session' => [
            'handler_id' => MongoDbSessionHandler::class,
        ],
    ],
]);
```

完成了！Symfony 现在将使用你的 MongoDB 服务器来读取和写入会话数据。你不需要做任何事情来初始化会话集合。但是，你可能需要添加一个索引来提高垃圾回收性能。在 [MongoDB shell](https://docs.mongodb.com/manual/mongo/) 中运行以下命令：

```javascript
use session_db
db.session.createIndex( { "expires_at": 1 }, { expireAfterSeconds: 0 } )
```

#### 配置会话字段名

用于存储会话的集合定义了特定的字段名。你可以通过传递给 `MongoDbSessionHandler` 服务的第二个参数来配置这些值：

**YAML：**

```yaml
# config/services.yaml
services:
    # ...

    Symfony\Component\HttpFoundation\Session\Storage\Handler\MongoDbSessionHandler:
        arguments:
            - '@doctrine_mongodb.odm.default_connection'
            -
                database: '%env(MONGODB_DB)%'
                collection: 'sessions'
                id_field: '_guid'
                expiry_field: 'eol'
```

**PHP：**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\HttpFoundation\Session\Storage\Handler\MongoDbSessionHandler;

return App::config([
    'services' => [
        // ...
        MongoDbSessionHandler::class => [
            'arguments' => [
                service('doctrine_mongodb.odm.default_connection'),
                [
                    'database' => env('MONGODB_DB'),
                    'collection' => 'sessions',
                    'id_field' => '_guid',
                    'expiry_field' => 'eol',
                ],
            ],
        ],
    ],
]);
```

以下是你可以配置的参数：

**`id_field`**（默认值 `_id`）：用于存储会话 ID 的字段名。

**`data_field`**（默认值 `data`）：用于存储会话数据的字段名。

**`time_field`**（默认值 `time`）：用于存储会话创建时间戳的字段名。

**`expiry_field`**（默认值 `expires_at`）：用于存储会话生命周期的字段名。

### 在会话处理器之间迁移

如果你的应用程序更改了会话的存储方式，请使用 `Symfony\Component\HttpFoundation\Session\Storage\Handler\MigratingSessionHandler` 在旧的和新的保存处理器之间迁移，而不会丢失会话数据。

以下是推荐的迁移工作流：

1. 切换到迁移处理器，将新处理器设置为只写处理器。旧处理器照常工作，会话也会被写入新处理器：

    ```php
    $sessionStorage = new MigratingSessionHandler($oldSessionStorage, $newSessionStorage);
    ```

2. 经过会话垃圾回收周期后，验证新处理器中的数据是否正确。

3. 更新迁移处理器，将旧处理器设置为只写处理器，这样会话现在将从新处理器读取。此步骤可以更轻松地回滚：

    ```php
    $sessionStorage = new MigratingSessionHandler($newSessionStorage, $oldSessionStorage);
    ```

4. 验证应用程序中的会话正常工作后，从迁移处理器切换到新处理器。

### 配置会话 TTL

默认情况下，Symfony 使用 PHP 的 ini 设置 `session.gc_maxlifetime` 作为会话生命周期。当你将会话存储在数据库中时，也可以在框架配置中或在运行时配置自己的 TTL。

> **注意：** 会话启动后就无法更改 ini 设置，所以如果你想根据登录用户使用不同的 TTL，必须在运行时使用下面的回调方法来实现。

#### 配置 TTL

你需要在你使用的会话处理器的选项数组中传递 TTL：

**YAML：**

```yaml
# config/services.yaml
services:
    # ...
    Symfony\Component\HttpFoundation\Session\Storage\Handler\RedisSessionHandler:
        arguments:
            - '@Redis'
            - { 'ttl': 600 }
```

**PHP：**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\HttpFoundation\Session\Storage\Handler\RedisSessionHandler;

return App::config([
    'services' => [
        // ...
        RedisSessionHandler::class => [
            'arguments' => [
                service('Redis'),
                ['ttl' => 600],
            ],
        ],
    ],
]);
```

#### 在运行时动态配置 TTL

如果你出于某种原因想要为不同的用户或会话设置不同的 TTL，也可以通过将回调作为 TTL 值来实现。该回调将在会话写入之前调用，并且必须返回一个整数，该整数将用作 TTL。

**YAML：**

```yaml
# config/services.yaml
services:
    # ...
    Symfony\Component\HttpFoundation\Session\Storage\Handler\RedisSessionHandler:
        arguments:
            - '@Redis'
            - { 'ttl': !closure '@my.ttl.handler' }

    my.ttl.handler:
        class: Some\InvokableClass # 某个带有 __invoke() 方法的类
        arguments:
            # 注入解析当前会话 TTL 所需的任何依赖项
            - '@security'
```

**PHP：**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Some\InvokableClass;
use Symfony\Component\HttpFoundation\Session\Storage\Handler\RedisSessionHandler;

return App::config([
    'services' => [
        // ...
        RedisSessionHandler::class => [
            'arguments' => [
                service('Redis'),
                ['ttl' => closure(service('my.ttl.handler'))],
            ],
        ],
        'my.ttl.handler' => [
            'class' => InvokableClass::class,
            // 注入解析当前会话 TTL 所需的任何依赖项
            'arguments' => [service('security')],
        ],
    ],
]);
```

## 在用户会话期间使区域设置"持久"

Symfony 将区域设置存储在 Request 中，这意味着该设置不会在请求之间自动保存（"持久"）。但是，你*可以*将区域设置存储在会话中，以便在后续请求中使用它。

### 创建 LocaleSubscriber

创建一个新的事件订阅器。通常，`_locale` 被用作路由参数来表示区域设置，但你可以通过任何方式确定正确的区域设置：

```php
// src/EventSubscriber/LocaleSubscriber.php
namespace App\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\KernelEvents;

class LocaleSubscriber implements EventSubscriberInterface
{
    public function __construct(
        private string $defaultLocale = 'en',
    ) {
    }

    public function onKernelRequest(RequestEvent $event): void
    {
        $request = $event->getRequest();
        if (!$request->hasPreviousSession()) {
            return;
        }

        // 尝试查看区域设置是否已设置为 _locale 路由参数
        if ($locale = $request->attributes->get('_locale')) {
            $request->getSession()->set('_locale', $locale);
        } else {
            // 如果此请求没有设置明确的区域设置，则使用会话中的区域设置
            $request->setLocale($request->getSession()->get('_locale', $this->defaultLocale));
        }
    }

    public static function getSubscribedEvents(): array
    {
        return [
            // 必须在默认 Locale 监听器之前注册（即优先级更高）
            KernelEvents::REQUEST => [['onKernelRequest', 20]],
        ];
    }
}
```

如果你使用默认的 `services.yaml` 配置，就完成了！Symfony 将自动识别事件订阅器，并在每次请求时调用 `onKernelRequest` 方法。

要查看其效果，可以手动在会话中设置 `_locale` 键（例如，通过某个"更改区域设置"路由和控制器），或者使用 `_locale` 默认值创建路由。

> **显式配置订阅器**
>
> 你也可以显式配置它，以便传入 `default_locale`：
>
> **YAML：**
>
> ```yaml
> # config/services.yaml
> services:
>     # ...
>
>     App\EventSubscriber\LocaleSubscriber:
>         arguments: ['%kernel.default_locale%']
>         # 如果你不使用 autoconfigure，请取消注释下一行
>         # tags: [kernel.event_subscriber]
> ```
>
> **PHP：**
>
> ```php
> // config/services.php
> namespace Symfony\Component\DependencyInjection\Loader\Configurator;
>
> use App\EventSubscriber\LocaleSubscriber;
>
> return App::config([
>     'services' => [
>         // ...
>         LocaleSubscriber::class => [
>             'arguments' => [param('kernel.default_locale')],
>             // 如果你不使用 autoconfigure，请取消注释下一行
>             // 'tags' => ['kernel.event_subscriber'],
>         ],
>     ],
> ]);
> ```

现在，通过更改用户的区域设置，你会看到它在请求过程中是持久的。

记住，要获取用户的区域设置，始终使用 `Request::getLocale` 方法：

```php
// 在控制器中...
use Symfony\Component\HttpFoundation\Request;

public function index(Request $request): void
{
    $locale = $request->getLocale();
}
```

### 根据用户偏好设置区域设置

你可能希望进一步改进此技术，根据已登录用户的用户实体来定义区域设置。但是，由于 `LocaleSubscriber` 在负责处理身份验证并在 `TokenStorage` 上设置用户令牌的 `FirewallListener` 之前被调用，因此你无法访问已登录的用户。

假设你的 `User` 实体上有一个 `locale` 属性，并且想将其用作给定用户的区域设置。为此，你可以挂钩到登录过程，并在用户被重定向到第一个页面之前用此区域设置值更新用户的会话。

为此，你需要在 `LoginSuccessEvent::class` 事件上注册一个事件订阅器：

```php
// src/EventSubscriber/UserLocaleSubscriber.php
namespace App\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\RequestStack;
use Symfony\Component\Security\Http\Event\LoginSuccessEvent;

/**
 * 登录后将用户的区域设置存储在会话中。
 * 之后可以被 LocaleSubscriber 使用。
 */
class UserLocaleSubscriber implements EventSubscriberInterface
{
    public function __construct(
        private RequestStack $requestStack,
    ) {
    }

    public function onLoginSuccess(LoginSuccessEvent $event): void
    {
        $user = $event->getUser();

        if (null !== $user->getLocale()) {
            $this->requestStack->getSession()->set('_locale', $user->getLocale());
        }
    }

    public static function getSubscribedEvents(): array
    {
        return [
            LoginSuccessEvent::class => 'onLoginSuccess',
        ];
    }
}
```

> **警告：** 为了在用户更改语言偏好后立即更新语言，你还需要在更改 `User` 实体时更新会话。

## 会话代理

会话代理机制有多种用途，本文演示了两种常见用法。你可以通过定义一个继承 `Symfony\Component\HttpFoundation\Session\Storage\Proxy\SessionHandlerProxy` 类的类来创建自定义保存处理器，而不是使用常规的会话处理器。

然后，将该类定义为服务。如果你使用默认的 `services.yaml` 配置，这将自动发生。

最后，使用 `framework.session.handler_id` 配置选项告知 Symfony 使用你的会话处理器而不是默认的：

**YAML：**

```yaml
# config/packages/framework.yaml
framework:
    session:
        # ...
        handler_id: App\Session\CustomSessionHandler
```

**PHP：**

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Session\CustomSessionHandler;

return App::config([
    'framework' => [
    // ...
    'session' => [
            'handler_id' => CustomSessionHandler::class,
        ],
    ],
]);
```

继续阅读以下各节，了解如何在实践中使用会话处理器来解决两个常见用例：加密会话信息和定义只读访客会话。

### 会话数据加密

如果你想加密会话数据，可以使用代理在需要时加密和解密会话。以下示例使用 [php-encryption](https://github.com/defuse/php-encryption) 库，但你可以将其调整为你可能正在使用的任何其他库：

```php
// src/Session/EncryptedSessionProxy.php
namespace App\Session;

use Defuse\Crypto\Crypto;
use Defuse\Crypto\Key;
use Symfony\Component\HttpFoundation\Session\Storage\Proxy\SessionHandlerProxy;

class EncryptedSessionProxy extends SessionHandlerProxy
{
    public function __construct(
        private \SessionHandlerInterface $handler,
        private Key $key
    ) {
        parent::__construct($handler);
    }

    public function read($id): string
    {
        $data = parent::read($id);

        return Crypto::decrypt($data, $this->key);
    }

    public function write($id, $data): string
    {
        $data = Crypto::encrypt($data, $this->key);

        return parent::write($id, $data);
    }
}
```

另一种加密会话数据的方式是装饰 `session.marshaller` 服务，该服务指向 `Symfony\Component\HttpFoundation\Session\Storage\Handler\MarshallingSessionHandler`。你可以使用使用加密的 marshaller 来装饰此处理器，例如 `Symfony\Component\Cache\Marshaller\SodiumMarshaller`。

首先，你需要生成一个安全密钥，并将其作为 `SESSION_DECRYPTION_FILE` 添加到你的密钥存储中：

```bash
$ php -r 'echo base64_encode(sodium_crypto_box_keypair());'
```

然后，使用此密钥注册 `SodiumMarshaller` 服务：

**YAML：**

```yaml
# config/services.yaml
services:

    # ...
    Symfony\Component\Cache\Marshaller\SodiumMarshaller:
        decorates: 'session.marshaller'
        arguments:
            - ['%env(file:resolve:SESSION_DECRYPTION_FILE)%']
            - '@.inner'
```

**PHP：**

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use Symfony\Component\Cache\Marshaller\SodiumMarshaller;

return App::config([
    'services' => [
        // ...
        SodiumMarshaller::class => [
            'decorates' => 'session.marshaller',
            'arguments' => [
                [env('SESSION_DECRYPTION_FILE')->resolve()->file()],
                service('.inner'),
            ],
        ],
    ],
]);
```

> **注意：** 这将加密缓存项的值，但不会加密缓存键。注意不要在键中泄露敏感数据。

### 只读访客会话

在某些应用程序中，访客用户需要会话，但没有持久化会话的特别需要。在这种情况下，你可以在写入会话之前拦截它：

```php
// src/Session/ReadOnlySessionProxy.php
namespace App\Session;

use App\Entity\User;
use Symfony\Bundle\SecurityBundle\Security;
use Symfony\Component\HttpFoundation\Session\Storage\Proxy\SessionHandlerProxy;

class ReadOnlySessionProxy extends SessionHandlerProxy
{
    public function __construct(
        private \SessionHandlerInterface $handler,
        private Security $security
    ) {
        parent::__construct($handler);
    }

    public function write($id, $data): string
    {
        if ($this->getUser() && $this->getUser()->isGuest()) {
            return;
        }

        return parent::write($id, $data);
    }

    private function getUser(): ?User
    {
        $user = $this->security->getUser();
        if (is_object($user)) {
            return $user;
        }

        return null;
    }
}
```

## 与旧版应用程序集成

如果你将 Symfony 全栈框架集成到一个使用 `session_start()` 启动会话的旧版应用程序中，你仍然可以通过使用 PHP Bridge 会话来使用 Symfony 的会话管理。

如果应用程序有自己的 PHP 保存处理器，你可以为 `handler_id` 指定 `null`：

**YAML：**

```yaml
# config/packages/framework.yaml
framework:
    session:
        storage_factory_id: session.storage.factory.php_bridge
        handler_id: ~
```

**PHP：**

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'session' => [
            'storage_factory_id' => 'session.storage.factory.php_bridge',
            'handler_id' => null,
        ],
    ],
]);
```

**独立应用：**

```php
use Symfony\Component\HttpFoundation\Session\Session;
use Symfony\Component\HttpFoundation\Session\Storage\PhpBridgeSessionStorage;

// 旧版应用程序配置会话
ini_set('session.save_handler', 'files');
ini_set('session.save_path', '/tmp');
session_start();

// 让 Symfony 与此现有会话对接
$session = new Session(new PhpBridgeSessionStorage());

// symfony 现在将与现有的 PHP 会话对接
$session->start();
```

否则，如果问题是你无法避免应用程序使用 `session_start()` 启动会话，你仍然可以通过按以下示例指定保存处理器来使用基于 Symfony 的会话保存处理器：

**YAML：**

```yaml
# config/packages/framework.yaml
framework:
    session:
        storage_factory_id: session.storage.factory.php_bridge
        handler_id: session.handler.native_file
```

**PHP：**

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'session' => [
            'storage_factory_id' => 'session.storage.factory.php_bridge',
            'handler_id' => 'session.handler.native_file',
        ],
    ],
]);
```

> **注意：** 如果旧版应用程序需要自己的会话保存处理器，请不要覆盖它。而是设置 `handler_id: ~`。注意，一旦会话启动，就无法更改保存处理器。如果应用程序在 Symfony 初始化之前启动会话，保存处理器将已经设置好了。在这种情况下，你需要 `handler_id: ~`。只有在你确定旧版应用程序可以使用 Symfony 保存处理器而没有副作用，并且会话在 Symfony 初始化之前没有被启动的情况下，才覆盖保存处理器。
