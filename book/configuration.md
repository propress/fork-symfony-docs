# 配置 Symfony

## 配置文件

Symfony 应用通过存储在 `config/` 目录中的文件进行配置，该目录具有以下默认结构：

```text
your-project/
├─ config/
│  ├─ packages/
│  ├─ routes/
│  ├─ bundles.php
│  ├─ preload.php
│  ├─ reference.php
│  ├─ routes.yaml
│  └─ services.yaml
```

- `routes.yaml` 文件定义[路由配置](routing.md)；
- `services.yaml` 文件配置[服务容器](service_container.md)的服务；
- `bundles.php` 文件在应用中启用/禁用包；
- `preload.php` 文件定义[用 OPcache 预加载](performance.md#使用-opcache-预加载)的类；
- `reference.php` 文件由 Symfony 自动生成，包含在使用 PHP 作为配置格式时改善 IDE 自动补全和静态分析的定义；
- `config/packages/` 目录存储应用中安装的每个包的配置；
- `config/routes/` 目录存储已安装包加载的路由配置（例如 `framework.yaml`）。

包（在 Symfony 中也称为"Bundle"，在其他项目中称为"插件/模块"）向你的项目添加即用型功能。

使用 [Symfony Flex](setup.md#symfony-flex)（在 Symfony 应用中默认启用）时，包在安装期间会自动更新 `bundles.php` 文件并在 `config/packages/` 中创建新文件。例如，这是"API Platform" Bundle 创建的默认文件：

```yaml
# config/packages/api_platform.yaml
api_platform:
    mapping:
        paths: ['%kernel.project_dir%/src/Entity']
```

将配置拆分为许多小文件对某些 Symfony 新手来说可能看起来很复杂。但是，你很快就会习惯它们，并且在包安装后很少需要更改这些文件。

> **提示**
>
> 要了解所有可用的配置选项，请查看 [Symfony 配置参考](reference/index.md)或运行 `config:dump-reference` 命令。

### 配置格式 {#configuration-formats}

与其他框架不同，Symfony 不强制你使用特定格式来配置应用，而是让你在 YAML 和 PHP 之间选择。在整个 Symfony 文档中，所有配置示例将以这两种格式显示。

格式之间没有任何实质性区别。实际上，Symfony 在运行应用之前会将它们全部转换为 PHP 并缓存，因此甚至没有性能差异。

安装包时默认使用 YAML，因为它简洁且非常易读。以下是每种格式的主要优缺点：

- **YAML**：简单、清晰且易读，但并非所有 IDE 都支持其自动补全和验证。[了解 YAML 语法](reference/formats/yaml.md)；
- **PHP**：非常强大，允许你用数组创建动态配置，并受益于数组形状的自动补全和静态分析。

### 导入配置文件

Symfony 使用 [Config 组件](components/config.md)加载配置文件，该组件提供了高级功能，如导入其他配置文件（即使它们使用不同格式）：

```yaml
# config/services.yaml
imports:
    - { resource: 'legacy_config.php' }

    # 也支持 glob 表达式以加载多个文件
    - { resource: '/etc/myapp/*.yaml' }

    # ignore_errors: not_found 会在加载的文件不存在时静默丢弃错误
    - { resource: 'my_config_file.php', ignore_errors: not_found }
    # ignore_errors: true 静默丢弃所有错误（包括无效代码和未找到）
    - { resource: 'my_other_config_file.php', ignore_errors: true }

# ...
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'imports' => [
        ['resource' => 'legacy_config.php'],
        // 也支持 glob 表达式以加载多个文件
        ['resource' => '/etc/myapp/*.yaml'],
        // ignore_errors: not_found 会在加载的文件不存在时静默丢弃错误
        ['resource' => 'my_config_file.xml', 'ignore_errors' => 'not_found'],
        // ignore_errors: true 静默丢弃所有错误（包括无效代码和未找到）
        ['resource' => 'my_other_config_file.xml', 'ignore_errors' => true],
    ],
]);
```

---

## 配置参数 {#configuration-parameters}

有时同一配置值被用于多个配置文件。不必重复它，你可以将其定义为"参数"，这就像一个可重用的配置值。按照惯例，参数在 `parameters` 键下定义：

```yaml
# config/services.yaml
parameters:
    # 参数名是任意字符串（推荐使用 'app.' 前缀
    # 以更好地区分你的参数和 Symfony 参数）
    app.admin_email: 'something@example.com'

    # 布尔参数
    app.enable_v2_protocol: true

    # 数组/集合参数
    app.supported_locales: ['en', 'es', 'fr']

    # 二进制内容参数（用 base64_encode() 编码内容）
    app.some_parameter: !!binary VGhpcyBpcyBhIEJlbGwgY2hhciAH

    # PHP 常量作为参数值
    app.some_constant: !php/const GLOBAL_CONSTANT
    app.another_constant: !php/const App\Entity\BlogPost::MAX_ITEMS

    # 枚举案例作为参数值
    app.some_enum: !php/enum App\Enum\PostState::Published

# ...
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Entity\BlogPost;
use App\Enum\PostState;

return App::config([
    'parameters' => [
        // 参数名是任意字符串（推荐使用 'app.' 前缀）
        'app.admin_email' => 'something@example.com',
        // 布尔参数
        'app.enable_v2_protocol' => true,
        // 数组/集合参数
        'app.supported_locales' => ['en', 'es', 'fr'],
        // 二进制内容参数
        'app.some_parameter' => base64_decode('VGhpcyBpcyBhIEJlbGwgY2hhciAA'),
        // PHP 常量作为参数值
        'app.some_constant' => GLOBAL_CONSTANT,
        'app.another_constant' => BlogPost::MAX_ITEMS,
        // 枚举案例作为参数值
        'app.some_enum' => PostState::Published,
    ],
]);
```

定义后，你可以在任何其他配置文件中使用特殊语法引用此参数值：用两个 `%` 包裹参数名（例如 `%app.admin_email%`）：

```yaml
# config/packages/some_package.yaml
some_package:
    # 被两个 % 包围的任何字符串都会被替换为该参数值
    email_address: '%app.admin_email%'
```

```php
// config/packages/some_package.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'some_package' => [
        // 使用 param() 函数时，只需传递参数名...
        'email_address' => param('app.admin_email'),
        // ... 但如果你愿意，也可以传递被两个 % 包围的名称（与 YAML 格式相同）
        // Symfony 会将其替换为该参数值
        'email_address' => '%app.admin_email%',
    ],
]);
```

> **注意**
>
> 如果某个参数值包含 `%` 字符，需要通过再添加一个 `%` 来转义它，以防 Symfony 将其视为对参数名的引用：
>
> ```yaml
> # config/services.yaml
> parameters:
>     # 解析为 'https://symfony.com/?foo=%s&amp;bar=%d'
>     url_pattern: 'https://symfony.com/?foo=%%s&amp;bar=%%d'
> ```

配置参数在 Symfony 应用中非常常见。Symfony 本身定义了几个参数，包括与[内核配置](reference/configuration/kernel.md)相关的参数（例如 `kernel.project_dir`、`kernel.debug`），某些包在安装时也会向 `config/services.yaml` 添加自己的参数。

> **提示**
>
> 按照惯例，名称以点 `.` 开头的参数（例如 `.mailer.transport`）仅在容器编译期间可用。它们在使用[编译器通道](service_container/compiler_passes.md)声明一些后来在应用中不可用的临时参数时很有用。

配置参数通常无需验证，但你可以确保对应用功能至关重要的参数不为空：

```php
/** @var ContainerBuilder $container */
$container->parameterCannotBeEmpty('app.private_key', '你是否忘记为 "app.private_key" 参数设置值？');
```

如果非空参数是 `null`、空字符串 `''` 或空数组 `[]`，Symfony 将抛出异常。此验证**不**在编译时进行，而是在尝试检索参数值时进行。

> **另请参阅**
>
> 本文后面介绍如何[在控制器和服务中获取配置参数](#访问配置参数)。

---

## 配置环境 {#configuration-environments}

你只有一个应用，但无论你是否意识到，你都需要它在不同时间表现不同：

- **开发**时，你希望记录所有内容并公开友好的调试工具；
- 部署到**生产**环境后，你希望同一个应用针对速度进行优化，并只记录错误。

`config/packages/` 中存储的文件被 Symfony 用来配置[应用服务](service_container.md)。换句话说，你可以通过更改加载哪些配置文件来更改应用行为。这就是 Symfony **配置环境**的理念。

典型的 Symfony 应用从三个环境开始：

- `dev`：用于本地开发，
- `prod`：用于生产服务器，
- `test`：用于[自动化测试](testing.md)。

运行应用时，Symfony 按以下顺序加载配置文件（后面的文件可以覆盖前面设置的值）：

1. `config/packages/*.<extension>` 中的文件；
2. `config/packages/<environment-name>/*.<extension>` 中的文件；
3. `config/services.<extension>`；
4. `config/services_<environment-name>.<extension>`。

以默认安装的 `framework` 包为例：

- 首先，`config/packages/framework.yaml` 在所有环境中加载，并用某些选项配置框架；
- 在 **prod** 环境中，不会额外设置任何内容，因为没有 `config/packages/prod/framework.yaml` 文件；
- 在 **dev** 环境中，也没有文件（`config/packages/dev/framework.yaml` 不存在）；
- 在 **test** 环境中，加载 `config/packages/test/framework.yaml` 文件以覆盖之前在 `config/packages/framework.yaml` 中配置的某些设置。

实际上，每个环境只与其他环境略有不同。这意味着所有环境共享大量公共配置，这些配置放在 `config/packages/` 目录中的文件里。

> **提示**
>
> 你也可以使用特殊的 `when` 关键字在单个配置文件中为不同环境定义选项：
>
> ```yaml
> # config/packages/webpack_encore.yaml
> webpack_encore:
>     # ...
>     output_path: '%kernel.project_dir%/public/build'
>     strict_mode: true
>     cache: false
>
> # 仅在 "prod" 环境中启用缓存
> when@prod:
>     webpack_encore:
>         cache: true
>
> # 仅在 "test" 环境中禁用严格模式
> when@test:
>     webpack_encore:
>         strict_mode: false
> ```

### 选择活动环境 {#selecting-the-active-environment}

Symfony 应用附带一个名为 `.env` 的文件，位于项目根目录。此文件用于定义环境变量的值，[本文后面](#在-env-文件中配置环境变量)会详细说明。

打开 `.env` 文件（或者更好的是，如果你创建了 `.env.local` 文件则打开它）并编辑 `APP_ENV` 变量的值以更改应用运行的环境。例如，要在生产环境中运行应用：

```bash
# .env（或 .env.local）
APP_ENV=prod
```

此值既用于 Web 请求也用于控制台命令。但是，你可以通过在运行命令之前设置 `APP_ENV` 值来覆盖命令的环境：

```terminal
# 使用 .env 文件中定义的环境
$ php bin/console command_name

# 忽略 .env 文件并在生产环境中运行此命令
$ APP_ENV=prod php bin/console command_name
```

### 创建新环境

Symfony 提供的默认三个环境对大多数项目来说已经足够，但你也可以定义自己的环境。例如，这是如何定义一个 `staging` 环境，让客户在上线前测试项目：

1. 创建一个与环境同名的配置目录（在本例中为 `config/packages/staging/`）；
2. 在 `config/packages/staging/` 中添加所需的配置文件以定义新环境的行为。Symfony 首先加载 `config/packages/*.yaml` 文件，因此你只需配置与这些文件的差异；
3. 使用 `APP_ENV` 环境变量选择 `staging` 环境，如上一节所述。

> **提示**
>
> 环境之间通常很相似，因此你可以在 `config/packages/<environment-name>/` 目录之间使用[符号链接](https://en.wikipedia.org/wiki/Symbolic_link)来复用相同的配置。

你也可以使用以下部分中说明的环境变量，而不是创建新环境。这样你可以使用相同的应用和环境（例如 `prod`），但通过基于环境变量的配置更改其行为（例如在不同场景中运行应用：staging、质量保证、客户评审等）。

---

## 基于环境变量的配置 {#config-env-vars}

使用[环境变量](https://en.wikipedia.org/wiki/Environment_variable)（简称"env vars"）是一种常见做法，用于：

- 配置依赖于应用运行位置的选项（例如，数据库凭据在生产环境与本地机器上通常不同）；
- 配置可以在生产环境中动态更改的选项（例如，无需重新部署整个应用即可更新过期 API 密钥的值）。

在其他情况下，建议继续使用[配置参数](#配置参数)。

使用特殊语法 `%env(ENV_VAR_NAME)%` 引用环境变量。这些选项的值在运行时解析（每次请求只解析一次，以不影响性能），因此你可以在不清除缓存的情况下更改应用行为。

此示例展示了如何使用环境变量配置应用密钥：

```yaml
# config/packages/framework.yaml
framework:
    # 按照惯例，环境变量名总是大写
    secret: '%env(APP_SECRET)%'
    # ...
```

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        // 按照惯例，环境变量名总是大写
        'secret' => env('APP_SECRET'),
    ],
]);
```

> **注意**
>
> 你的环境变量也可以通过 PHP 超全局变量 `$_ENV` 和 `$_SERVER`（两者等效）访问：
>
> ```php
> $databaseUrl = $_ENV['DATABASE_URL']; // mysql://db_user:db_password@127.0.0.1:3306/db_name
> $env = $_SERVER['APP_ENV']; // prod
> ```
>
> 但是，在 Symfony 应用中不需要这样做，因为配置系统提供了更好的环境变量使用方式。

> **另请参阅**
>
> 环境变量的值只能是字符串，但 Symfony 包含一些[环境变量处理器](configuration/env_var_processors.md)来转换其内容（例如将字符串值转换为整数）。

要定义环境变量的值，你有几个选项：

- [将值添加到 .env 文件](#在-env-文件中配置环境变量)；
- [将值加密为 secret](#加密环境变量secrets)；
- 在你的 shell 或 Web 服务器中将值设置为真实的环境变量。

如果你的应用尝试使用尚未定义的环境变量，你将看到异常。你可以通过为环境变量定义默认值来防止这种情况。为此，使用以下语法定义一个与环境变量同名的参数：

```yaml
# config/services.yaml
parameters:
    # 如果 SECRET 环境变量值未在任何地方定义，Symfony 使用此值
    env(SECRET): 'some_secret'

# ...
```

> **提示**
>
> 某些主机（如 Upsun.com）提供了方便的[环境变量管理工具](https://symfony.com/doc/current/cloud/env.html)用于生产环境。

> **注意**
>
> 某些配置功能与环境变量不兼容。例如，根据另一个配置选项的存在有条件地定义某些容器参数。使用环境变量时，配置选项始终存在，因为当相关环境变量未定义时，其值将为 `null`。

> **危险**
>
> 请注意，转储 `$_SERVER` 和 `$_ENV` 变量的内容或输出 `phpinfo()` 内容将显示环境变量的值，从而暴露敏感信息（如数据库凭据）。
>
> 环境变量的值也会在 [Symfony 分析器](profiler.md)的 Web 界面中公开。实际上这不应该是问题，因为在生产环境中**绝不**能启用 Web 分析器。

### 在 .env 文件中配置环境变量 {#config-dot-env}

Symfony 提供了一种在位于项目根目录的 `.env`（以点开头）文件中定义环境变量的便捷方式，而不是在 shell 或 Web 服务器中定义。

`.env` 文件在每次请求时被读取和解析，其环境变量被添加到 PHP 变量 `$_ENV` 和 `$_SERVER` 中。任何现有的环境变量*永远*不会被 `.env` 中定义的值覆盖，因此你可以将两者结合使用。

例如，要定义本文前面展示的 `DATABASE_URL` 环境变量，你可以添加：

```bash
# .env
DATABASE_URL="mysql://db_user:db_password@127.0.0.1:3306/db_name"
```

此文件应提交到你的仓库，因此（由于这个原因）应只包含适合本地开发的"默认"值。此文件不应包含生产值。

除了你自己的环境变量，`.env` 文件还包含应用中安装的第三方包定义的环境变量（它们在安装包时由 [Symfony Flex](setup.md#symfony-flex) 自动添加）。

> **提示**
>
> 由于 `.env` 文件在每次请求时都会被读取和解析，如果你使用 Docker，无需清除 Symfony 缓存或重启 PHP 容器。

#### .env 文件语法

通过在注释前加 `#` 来添加注释：

```bash
# 数据库凭据
DB_USER=root
DB_PASS=pass # 这是秘密密码
```

在值中使用环境变量，通过在变量前加 `$`：

```bash
DB_USER=root
DB_PASS=${DB_USER}pass # 将用户名作为密码前缀
```

> **警告**
>
> 当某个环境变量依赖于其他环境变量的值时，顺序很重要。在上面的示例中，`DB_PASS` 必须在 `DB_USER` 之后定义。此外，如果你定义多个 `.env` 文件并将 `DB_PASS` 放在前面，其值将依赖于其他文件中定义的 `DB_USER` 值，而不是此文件中定义的值。

在环境变量未设置时定义默认值：

```bash
DB_USER=
DB_PASS=${DB_USER:-root}pass # 结果为 DB_PASS=rootpass
```

用单引号包裹值以将其用作字面字符串，其中 `$`、`#` 和其他特殊字符没有特殊含义：

```bash
DB_PASS='p@ss#w$rd'
```

使用双引号时，变量仍会被插值，但 `#` 和其他字符被视为字面量：

```bash
DB_PASS="p@ss#word"
DB_NAME="my_${DB_USER}_database"
```

通过 `$()` 嵌入命令（Windows 不支持）：

```bash
START_TIME=$(date)
```

> **警告**
>
> 使用 `$()` 可能因你的 shell 而不同。

> **提示**
>
> 由于 `.env` 文件是常规 shell 脚本，你可以在自己的 shell 脚本中 `source` 它：
>
> ```terminal
> $ source .env
> ```

### 通过 .env.local 覆盖环境值 {#configuration-multiple-env-files}

如果你需要覆盖某个环境值（例如，在本地机器上使用不同的值），可以在 `.env.local` 文件中这样做：

```bash
# .env.local
DATABASE_URL="mysql://root:@127.0.0.1:3306/my_database_name"
```

此文件应被 git 忽略，**不应**提交到你的仓库。以下几个 `.env` 文件可用于在恰当情况下设置环境变量：

- `.env`：定义应用所需的环境变量默认值；
- `.env.local`：为所有环境覆盖默认值，但仅在包含该文件的机器上。此文件不应提交到仓库，并在 `test` 环境中被忽略（因为测试应该对所有人产生相同的结果）；
- `.env.<environment>`（例如 `.env.test`）：仅覆盖一个环境的环境变量，但对所有机器有效（这些文件*会*提交）；
- `.env.<environment>.local`（例如 `.env.test.local`）：仅为一个环境定义机器特定的环境变量覆盖。类似于 `.env.local`，但覆盖仅适用于一个环境。

*真实*的环境变量始终优先于任何 `.env` 文件创建的环境变量。请注意，此行为取决于 [variables_order](http://php.net/manual/en/ini.core.php#ini.variables-order) 配置，该配置必须包含 `E` 以公开 `$_ENV` 超全局变量。这是 PHP 的默认配置。

`.env` 和 `.env.<environment>` 文件应提交到仓库，因为它们对所有开发人员和机器都相同。但是，以 `.local` 结尾的环境文件（`.env.local` 和 `.env.<environment>.local`）**不应提交**，因为只有你会使用它们。实际上，Symfony 附带的 `.gitignore` 文件会阻止它们被提交。

### 覆盖系统定义的环境变量

如果需要覆盖系统定义的环境变量，请使用 `Dotenv::loadEnv`、`Dotenv::bootEnv` 和 `Dotenv::populate` 方法定义的 `overrideExistingVars` 参数：

```php
use Symfony\Component\Dotenv\Dotenv;

$dotenv = new Dotenv();
$dotenv->loadEnv(__DIR__.'/.env', overrideExistingVars: true);

// ...
```

这将覆盖系统定义的环境变量，但**不会**覆盖 `.env` 文件中定义的环境变量。

### 在生产环境中配置环境变量 {#configuration-env-var-in-prod}

在生产环境中，`.env` 文件也会在每次请求时被解析和加载。因此，定义环境变量的最简单方法是在生产服务器上创建一个包含生产值的 `.env.local` 文件。

为了提高性能，你可以选择运行 `dump-env` Composer 命令：

```terminal
# 解析所有 .env 文件并将其最终值转储到 .env.local.php
$ composer dump-env prod
```

运行此命令后，Symfony 将加载 `.env.local.php` 文件以获取环境变量，而不会花时间解析 `.env` 文件。

> **提示**
>
> 更新你的部署工具/工作流，在每次部署后运行 `dotenv:dump` 命令以提高应用性能。

### 将环境变量存储在其他文件中

默认情况下，环境变量存储在项目根目录的 `.env` 文件中。但是，你可以用多种方式将它们存储在其他文件中。

如果你使用 [Runtime 组件](components/runtime.md)，dotenv 路径是你可以在 `composer.json` 文件中设置的选项之一：

```json
{
    "extra": {
        "runtime": {
            "dotenv_path": "my/custom/path/to/.env"
        }
    }
}
```

作为替代选项，你可以在 `bootstrap.php` 文件或应用的任何其他文件中直接调用 `Dotenv` 类：

```php
use Symfony\Component\Dotenv\Dotenv;

new Dotenv()->bootEnv(dirname(__DIR__).'my/custom/path/to/.env');
```

然后 Symfony 将在该文件中查找环境变量，也会在本地和特定环境的文件中查找（例如 `.*.local` 和 `.*.<environment>.local`）。

如果你需要知道 Symfony 正在使用的 `.env` 文件路径，可以在应用中读取 `SYMFONY_DOTENV_PATH` 环境变量。

### 加密环境变量（Secrets） {#configuration-secrets}

如果变量的值是敏感的（例如 API 密钥或数据库密码），你可以使用 [Secrets 管理系统](configuration/secrets.md)加密该值，而不是定义真实的环境变量或将其添加到 `.env` 文件。

### 列出环境变量

使用 `debug:dotenv` 命令了解 Symfony 如何解析不同的 `.env` 文件来设置每个环境变量的值：

```terminal
$ php bin/console debug:dotenv

Dotenv Variables & Files
========================

Scanned Files (in descending priority)
---------------------------------------

* ⨯ .env.local.php
* ⨯ .env.dev.local
* ✓ .env.dev
* ⨯ .env.local
* ✓ .env

Variables
---------

---------- ------- ---------- ------
 Variable   Value   .env.dev   .env
---------- ------- ---------- ------
 FOO        BAR     n/a        BAR
 ALICE      BOB     BOB        bob
---------- ------- ---------- ------

# 传递完整或部分名称作为参数来查找特定变量
$ php bin/console debug:dotenv foo
```

此外，无论你如何设置环境变量，都可以查看 Symfony 容器配置中引用的所有环境变量及其值，还可以查看每个环境变量在容器中出现的次数：

```terminal
$ php bin/console debug:container --env-vars

------------ ----------------- ------------------------------------ -------------
 Name         Default value     Real value                           Usage count
------------ ----------------- ------------------------------------ -------------
 APP_SECRET   n/a               "471a62e2d601a8952deb186e44186cb3"   2
 BAR          n/a               n/a                                  1
 BAZ          n/a               "value"                              0
 FOO          "[1, "2.5", 3]"   n/a                                  1
------------ ----------------- ------------------------------------ -------------

# 也可以按名称过滤环境变量列表：
$ php bin/console debug:container --env-vars foo

# 运行此命令以显示特定环境变量的所有详细信息：
$ php bin/console debug:container --env-var=FOO
```

### 创建自己的逻辑加载环境变量

如果默认的 Symfony 行为不符合你的需求，你可以实现自己的逻辑来加载环境变量。为此，创建一个类实现 `EnvVarLoaderInterface` 的服务。

> **注意**
>
> 如果你使用[默认的 services.yaml 配置](service_container.md#服务容器服务加载示例)，自动配置功能将自动启用并标记此服务。否则，你需要注册并[用 `container.env_var_loader` 标签标记你的服务](service_container/tags.md)。

假设你有一个名为 `env.json` 的 JSON 文件，其中包含你的环境变量：

```json
{
    "vars": {
        "APP_ENV": "prod",
        "APP_DEBUG": false
    }
}
```

你可以定义如下 `JsonEnvVarLoader` 类，从该文件填充环境变量：

```php
namespace App\DependencyInjection;

use Symfony\Component\DependencyInjection\EnvVarLoaderInterface;

final class JsonEnvVarLoader implements EnvVarLoaderInterface
{
    private const ENV_VARS_FILE = 'env.json';

    public function loadEnvVars(): array
    {
        $fileName = __DIR__.\DIRECTORY_SEPARATOR.self::ENV_VARS_FILE;
        if (!is_file($fileName)) {
            // 根据需要抛出异常或只是忽略此加载器
        }

        $content = json_decode(file_get_contents($fileName), true);

        return $content['vars'];
    }
}
```

就这样！现在应用将在当前目录中查找 `env.json` 文件以填充环境变量（除了已有的 `.env` 文件）。

### 在 Bundle 配置中使用环境变量

当你在 Bundle 配置中使用 `%env(...)%`（例如 `config/packages/doctrine.yaml`）时，该值**不会**在编译时读取。相反，Symfony 用唯一的占位符替换它。实际的环境变量只在运行时，当使用它的服务被实例化时才会解析。

**Bundle 作者**必须遵循某些规则以确保其 Bundle 正确支持运行时环境变量：

**在 Configuration 类中**（`TreeBuilder`）：

- 不要编写检查或转换配置选项*值*的 `beforeNormalization()` 步骤（仅重新组织键而不读取值的步骤是可以的）；
- 不要编写检查配置选项*值*的 `validate()` 步骤（在编译时，值仍然是占位符字符串，而非真实值）。

**在 DI 扩展中**（`load()` 方法）：

- 不要编写在将处理后的配置选项值注入 DI 参数或服务定义参数之前检查它们的逻辑。在编译时，这些值是占位符字符串，而非实际的环境变量值。

> **提示**
>
> 一般规则是：**将值连接到容器，不要检查它**。将环境变量值传递给服务参数或参数而不解释它们，足以使运行时解析自动工作。

#### 处理需要解析的 DSN 和值

如果服务需要 DSN 的已解析版本（例如从数据库 URL 中提取主机、端口和凭据），不要在容器编译期间在 DI 扩展中解析它。相反，创建一个在运行时解析 DSN 的**工厂服务**：

```php
// src/Factory/ClientFactory.php
namespace App\Factory;

class ClientFactory
{
    public static function create(string $dsn): SomeClient
    {
        $params = parse_url($dsn);

        return new SomeClient(
            host: $params['host'],
            port: $params['port'] ?? 5432,
            username: $params['user'] ?? '',
            password: $params['pass'] ?? '',
        );
    }
}
```

然后在服务配置中注册工厂：

```yaml
# config/services.yaml
services:
    App\SomeClient:
        factory: ['App\Factory\ClientFactory', 'create']
        arguments:
            - '%env(DATABASE_DSN)%'
```

```php
// config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Factory\ClientFactory;
use App\SomeClient;

return App::config([
    'services' => [
        SomeClient::class => service()
            ->factory([ClientFactory::class, 'create'])
            ->args([env('DATABASE_DSN')]),
    ],
]);
```

DoctrineBundle 通过其 `ConnectionFactory` 使用了这种方法，在运行时而非容器编译期间解析数据库 URL。

---

## 访问配置参数 {#configuration-accessing-parameters}

控制器和服务可以访问所有配置参数，包括[你自己定义的参数](#配置参数)和包/Bundle 创建的参数。运行以下命令查看应用中存在的所有参数：

```terminal
$ php bin/console debug:container --parameters
```

在继承自 [AbstractController](controller.md#基础控制器类与服务) 的控制器中，使用 `getParameter()` 辅助方法：

```php
// src/Controller/UserController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;

class UserController extends AbstractController
{
    // ...

    public function index(): Response
    {
        $projectDir = $this->getParameter('kernel.project_dir');
        $adminEmail = $this->getParameter('app.admin_email');

        // ...
    }
}
```

在不继承 `AbstractController` 的服务和控制器中，将参数作为构造函数的参数注入：

```php
// src/Service/MessageGenerator.php（使用 Attribute）
namespace App\Service;

use Symfony\Component\DependencyInjection\Attribute\Autowire;

class MessageGenerator
{
    public function __construct(
        #[Autowire(param: 'app.contents_dir')]
        private string $contentsDir,
    ) {
    }

    // ...
}
```

```yaml
# config/services.yaml（使用 YAML 配置注入）
parameters:
    app.contents_dir: '...'

services:
    App\Service\MessageGenerator:
        arguments:
            $contentsDir: '%app.contents_dir%'
```

推荐方式是使用 [#[Autowire] 属性](service_container/autowiring.md#autowire-attribute)。

如果你需要反复注入相同的参数，请改用 `services._defaults.bind` 选项。在该选项中定义的参数在服务构造函数或控制器动作定义具有确切名称的参数时自动注入。例如，要在服务/控制器定义 `$projectDir` 参数时注入 [kernel.project_dir 参数](reference/configuration/kernel.md)的值：

```yaml
# config/services.yaml
services:
    _defaults:
        bind:
            # 将此值传递给在此文件中创建的任何服务
            # 的任何 $projectDir 参数（包括控制器参数）
            $projectDir: '%kernel.project_dir%'

    # ...
```

> **另请参阅**
>
> 阅读关于[按名称和/或类型绑定参数](service_container/autowiring.md#按名称和类型绑定)的文章，了解更多关于这个强大功能的信息。

最后，如果某个服务需要访问大量参数，与其逐个注入，不如通过将其任何构造函数参数类型提示为 `ContainerBagInterface` 来一次性注入所有应用参数：

```php
// src/Service/MessageGenerator.php
namespace App\Service;

// ...

use Symfony\Component\DependencyInjection\ParameterBag\ContainerBagInterface;

class MessageGenerator
{
    public function __construct(
        private ContainerBagInterface $params,
    ) {
    }

    public function someMethod(): void
    {
        // 从 $this->params 获取任何容器参数，它存储了所有参数
        $sender = $this->params->get('mailer_sender');
        // ...
    }
}
```

---

## 继续学习！

恭喜！你已经掌握了 Symfony 的基础知识。接下来，通过跟随指南逐一学习 Symfony 的*每个*部分：

- [表单](forms.md)
- [Doctrine](doctrine.md)
- [服务容器](service_container.md)
- [安全](security.md)
- [邮件发送](mailer.md)
- [日志](logging.md)

以及所有其他与配置相关的主题：

- [环境变量处理器](configuration/env_var_processors.md)
- [前端控制器与 Kernel](configuration/front_controllers_and_kernel.md)
- [MicroKernel 特性](configuration/micro_kernel_trait.md)
- [多 Kernel](configuration/multiple_kernels.md)
- [覆盖目录结构](configuration/override_dir_structure.md)
- [Secrets 管理](configuration/secrets.md)
- [在 DIC 中使用参数](configuration/using_parameters_in_dic.md)
