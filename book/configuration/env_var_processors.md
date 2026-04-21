# 环境变量处理器

使用[环境变量配置 Symfony 应用](../configuration.md#使用环境变量)是让应用真正动态化的常见做法。

环境变量的主要问题是其值只能是字符串，而你的应用可能需要其他数据类型（整数、布尔值等）。Symfony 通过"环境变量处理器"解决了这个问题，处理器会转换给定环境变量的原始内容。以下示例使用整数处理器将 `HTTP_PORT` 环境变量的值转换为整数：

```yaml
# config/packages/framework.yaml
framework:
    router:
        http_port: '%env(int:HTTP_PORT)%'
```

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'router' => [
            'http_port' => env('HTTP_PORT')->int(),
        ],
    ],
]);
```

## 内置环境变量处理器

Symfony 提供了以下环境变量处理器：

`env(string:FOO)`
将 `FOO` 转换为字符串：

```yaml
# config/packages/framework.yaml
parameters:
    env(SECRET): 'some_secret'
framework:
    secret: '%env(string:SECRET)%'
```

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'env(SECRET)' => 'some_secret',
    ],
    'framework' => [
        'secret' => env('SECRET')->string(),
    ],
]);
```

`env(bool:FOO)`
将 `FOO` 转换为布尔值（`true` 值为 `'true'`、`'on'`、`'yes'`、除 `0` 和 `0.0` 以外的所有数字，以及除 `'0'` 和 `'0.0'` 以外的所有数字字符串；其他所有值为 `false`）：

```yaml
# config/packages/framework.yaml
parameters:
    env(HTTP_METHOD_OVERRIDE): 'true'
framework:
    http_method_override: '%env(bool:HTTP_METHOD_OVERRIDE)%'
```

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'env(HTTP_METHOD_OVERRIDE)' => 'true',
    ],
    'framework' => [
        'http_method_override' => env('HTTP_METHOD_OVERRIDE')->bool(),
    ],
]);
```

`env(not:FOO)`
将 `FOO` 转换为布尔值（与 `env(bool:...)` 相同），但返回取反的值（假值返回 `true`，真值返回 `false`）：

```yaml
# config/services.yaml
parameters:
    safe_for_production: '%env(not:APP_DEBUG)%'
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'safe_for_production' => env('APP_DEBUG')->not(),
    ],
]);
```

`env(int:FOO)`
将 `FOO` 转换为整数。

`env(float:FOO)`
将 `FOO` 转换为浮点数。

`env(const:FOO)`
查找 `FOO` 中命名的常量值：

```yaml
# config/packages/security.yaml
parameters:
    env(HEALTH_CHECK_METHOD): 'Symfony\Component\HttpFoundation\Request::METHOD_HEAD'
security:
    access_control:
        - { path: '^/health-check$', methods: '%env(const:HEALTH_CHECK_METHOD)%' }
```

```php
// config/packages/security.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'env(HEALTH_CHECK_METHOD)' => 'Symfony\Component\HttpFoundation\Request::METHOD_HEAD',
    ],
    'security' => [
        'access_control' => [
            ['path' => '^/health-check$', 'methods' => env('HEALTH_CHECK_METHOD')->const()],
        ],
    ],
]);
```

`env(base64:FOO)`
解码 `FOO` 的内容，该内容是一个 base64 编码的字符串。

`env(json:FOO)`
解码 `FOO` 的内容，该内容是一个 JSON 编码的字符串。它返回一个数组或 `null`：

```yaml
# config/services.yaml
parameters:
    env(ALLOWED_LANGUAGES): '["en","de","es"]'
    app_allowed_languages: '%env(json:ALLOWED_LANGUAGES)%'
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'env(ALLOWED_LANGUAGES)' => '["en","de","es"]',
        'app_allowed_languages' => env('ALLOWED_LANGUAGES')->json(),
    ],
]);
```

`env(resolve:FOO)`
如果 `FOO` 的内容包含容器参数（语法为 `%parameter_name%`），则将参数替换为其对应的值：

```yaml
# config/packages/sentry.yaml
parameters:
    sentry_host: '10.0.0.1'
    env(SENTRY_DSN): 'http://%sentry_host%/project'
sentry:
    dsn: '%env(resolve:SENTRY_DSN)%'
```

```php
// config/packages/sentry.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'sentry_host' => '10.0.0.1',
        'env(SENTRY_DSN)' => 'http://%sentry_host%/project',
    ],
    'sentry' => [
        'dsn' => env('SENTRY_DSN')->resolve(),
    ],
]);
```

`env(csv:FOO)`
解码 `FOO` 的内容，该内容是一个 CSV 编码的字符串：

```yaml
# config/services.yaml
parameters:
    env(ALLOWED_LANGUAGES): "en,de,es"
    app_allowed_languages: '%env(csv:ALLOWED_LANGUAGES)%'
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'env(ALLOWED_LANGUAGES)' => 'en,de,es',
        'app_allowed_languages' => env('ALLOWED_LANGUAGES')->csv(),
    ],
]);
```

`env(shuffle:FOO)`
随机打乱 `FOO` 环境变量的值，该值必须是一个数组：

```yaml
# config/services.yaml
parameters:
    env(REDIS_NODES): "127.0.0.1:6380,127.0.0.1:6381"
services:
    RedisCluster:
        class: RedisCluster
        arguments: [null, "%env(shuffle:csv:REDIS_NODES)%"]
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'env(REDIS_NODES)' => '127.0.0.1:6380,127.0.0.1:6381',
    ],
    'services' => [
        \RedisCluster::class => [
            'class' => \RedisCluster::class,
            'arguments' => [null, env('REDIS_NODES')->csv()->shuffle()],
        ],
    ],
]);
```

`env(file:FOO)`
返回路径为 `FOO` 环境变量值的文件内容：

```yaml
# config/packages/google.yaml
parameters:
    env(AUTH_FILE): '%kernel.project_dir%/config/auth.json'
google:
    auth: '%env(file:AUTH_FILE)%'
```

```php
// config/packages/google.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'env(AUTH_FILE)' => '../config/auth.json',
    ],
    'google' => [
        'auth' => env('AUTH_FILE')->file(),
    ],
]);
```

`env(require:FOO)`
`require()` 路径为 `FOO` 环境变量值的 PHP 文件，并返回该文件的返回值：

```yaml
# config/packages/google.yaml
parameters:
    env(PHP_FILE): '%kernel.project_dir%/config/.runtime-evaluated.php'
google:
    auth: '%env(require:PHP_FILE)%'
```

```php
// config/packages/google.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'env(PHP_FILE)' => '../config/.runtime-evaluated.php',
    ],
    'google' => [
        'auth' => env('PHP_FILE')->require(),
    ],
]);
```

`env(trim:FOO)`
修剪 `FOO` 环境变量的内容，删除字符串开头和末尾的空白。这在与 `file` 处理器结合使用时特别有用，因为它会删除文件末尾的换行符：

```yaml
# config/packages/google.yaml
parameters:
    env(AUTH_FILE): '%kernel.project_dir%/config/auth.json'
google:
    auth: '%env(trim:file:AUTH_FILE)%'
```

```php
// config/packages/google.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'env(AUTH_FILE)' => '../config/auth.json',
    ],
    'google' => [
        'auth' => env('AUTH_FILE')->file()->trim(),
    ],
]);
```

`env(key:FOO:BAR)`
从存储在 `BAR` 环境变量中的数组里，检索与键 `FOO` 关联的值：

```yaml
# config/services.yaml
parameters:
    env(SECRETS_FILE): '/opt/application/.secrets.json'
    database_password: '%env(key:database_password:json:file:SECRETS_FILE)%'
    # 如果 SECRETS_FILE 内容为：{"database_password": "secret"}，则返回 "secret"
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'env(SECRETS_FILE)' => '/opt/application/.secrets.json',
        'database_password' => env('SECRETS_FILE')->file()->json()->key('database_password'),
        // 如果 SECRETS_FILE 内容为：{"database_password": "secret"}，则返回 "secret"
    ],
]);
```

`env(default:fallback_param:BAR)`
当 `BAR` 环境变量不可用时，检索参数 `fallback_param` 的值：

```yaml
# config/services.yaml
parameters:
    # 如果 PRIVATE_KEY 不是有效的文件路径，则返回 raw_key 的内容
    private_key: '%env(default:raw_key:file:PRIVATE_KEY)%'
    raw_key: '%env(PRIVATE_KEY)%'
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        // 如果 PRIVATE_KEY 不是有效的文件路径，则返回 raw_key 的内容
        'private_key' => env('PRIVATE_KEY')->file()->default('raw_key'),
        'raw_key' => env('PRIVATE_KEY'),
    ],
]);
```

当省略回退参数时（例如 `env(default::API_KEY)`），返回值为 `null`。

`env(url:FOO)`
解析绝对 URL，并以关联数组的形式返回其组成部分：

```bash
# .env
DATABASE_URL="postgresql://db_user:db_password@127.0.0.1:5432/db_name"
```

```yaml
# config/packages/doctrine_mongodb.yaml
doctrine_mongodb:
    clients:
        default:
            hosts:
                - { host: '%env(string:key:host:url:MONGODB_URL)%', port: '%env(int:key:port:url:MONGODB_URL)%' }
            username: '%env(string:key:user:url:MONGODB_URL)%'
            password: '%env(string:key:pass:url:MONGODB_URL)%'
    connections:
        default:
            database_name: '%env(key:path:url:MONGODB_URL)%'
```

```php
// config/packages/doctrine_mongodb.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'doctrine_mongodb' => [
        'clients' => [
            'default' => [
                'hosts' => [
                    [
                        'host' => env('MONGODB_URL')->url()->key('host')->string(),
                        'port' => env('MONGODB_URL')->url()->key('port')->int(),
                    ],
                ],
                'username' => env('MONGODB_URL')->url()->key('user')->string(),
                'password' => env('MONGODB_URL')->url()->key('pass')->string(),
            ],
        ],
        'connections' => [
            'default' => [
                'database_name' => env('MONGODB_URL')->url()->key('path'),
            ],
        ],
    ],
]);
```

> **警告：** 为了便于从 URL 中提取资源，`path` 组件开头的 `/` 会被去除。

`env(query_string:FOO)`
解析给定 URL 的查询字符串部分，并以关联数组的形式返回其组成部分：

```bash
# .env
DATABASE_URL="postgresql://db_user:db_password@127.0.0.1:5432/db_name?serverVersion=12.19&charset=utf8"
```

```yaml
# config/packages/doctrine_mongodb.yaml
doctrine_mongodb:
    clients:
        default:
            # ...
            connectTimeoutMS: '%env(int:key:timeout:query_string:MONGODB_URL)%'
```

```php
// config/packages/doctrine_mongodb.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'doctrine_mongodb' => [
        'clients' => [
            'default' => [
                // ...
                'connectTimeoutMS' => env('MONGODB_URL')->queryString()->key('timeout')->int(),
            ],
        ],
    ],
]);
```

`env(enum:FooEnum:BAR)`
尝试将环境变量转换为实际的 `\BackedEnum` 值。此处理器将 `\BackedEnum` 的完全限定名称作为参数：

```php
// App\Enum\Suit.php
enum Suit: string
{
    case Clubs = 'clubs';
    case Spades = 'spades';
    case Diamonds = 'diamonds';
    case Hearts = 'hearts';
}
```

```yaml
# config/services.yaml
parameters:
    suit: '%env(enum:App\Enum\Suit:CARD_SUIT)%'
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Enum\Suit;

return App::config([
    'parameters' => [
        'suit' => env('CARD_SUIT')->enum(Suit::class),
    ],
]);
```

存储在 `CARD_SUIT` 环境变量中的值是字符串（例如 `'spades'`），但应用将使用枚举值（例如 `Suit::Spades`）。

`env(defined:NO_FOO)`
如果环境变量存在且其值既不是 `''`（空字符串）也不是 `null`，则求值为 `true`；否则返回 `false`：

```yaml
# config/services.yaml
parameters:
    typed_env: '%env(defined:FOO)%'
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'typed_env' => env('FOO')->defined(),
    ],
]);
```

`env(urlencode:FOO)`
使用 PHP 的 `urlencode` 函数对 `FOO` 环境变量的内容进行编码。当 `FOO` 的值与 DSN 语法不兼容时，这尤为有用：

```yaml
# config/services.yaml
parameters:
    env(DATABASE_URL): 'mysql://db_user:foo@b$r@127.0.0.1:3306/db_name'
    encoded_database_url: '%env(urlencode:DATABASE_URL)%'
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'env(DATABASE_URL)' => 'mysql://db_user:foo@b$r@127.0.0.1:3306/db_name',
        'encoded_database_url' => env('DATABASE_URL')->urlencode(),
    ],
]);
```

还可以将任意数量的处理器组合使用：

```yaml
# config/packages/google.yaml
parameters:
    env(AUTH_FILE): "%kernel.project_dir%/config/auth.json"
google:
    # 1. 获取 AUTH_FILE 环境变量的值
    # 2. 替换任何配置参数的值以获取配置路径
    # 3. 获取存储在该路径的文件内容
    # 4. 对文件内容进行 JSON 解码并返回
    auth: '%env(json:file:resolve:AUTH_FILE)%'
```

```php
// config/packages/google.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'parameters' => [
        'env(AUTH_FILE)' => '%kernel.project_dir%/config/auth.json',
    ],
    'google' => [
        // 1. 获取 AUTH_FILE 环境变量的值
        // 2. 替换任何配置参数的值以获取配置路径
        // 3. 获取存储在该路径的文件内容
        // 4. 对文件内容进行 JSON 解码并返回
        'auth' => env('AUTH_FILE')->resolve()->file()->json(),
    ],
]);
```

## 自定义环境变量处理器

你也可以为环境变量添加自定义处理器。首先，创建一个实现 `Symfony\Component\DependencyInjection\EnvVarProcessorInterface` 的类：

```php
use Symfony\Component\DependencyInjection\EnvVarProcessorInterface;

class LowercasingEnvVarProcessor implements EnvVarProcessorInterface
{
    public function getEnv(string $prefix, string $name, \Closure $getEnv): string
    {
        $env = $getEnv($name);

        return strtolower($env);
    }

    public static function getProvidedTypes(): array
    {
        return [
            'lowercase' => 'string',
        ];
    }
}
```

要在应用中启用新处理器，将其注册为服务并使用 `container.env_var_processor` 标签进行标记。如果你使用的是默认的 `services.yaml` 配置，由于自动配置的支持，这一步会自动完成。

## 在编译时解析环境变量

环境变量在运行时解析，但你也可以在编译时解析它们。
