# 翻译

"国际化"（通常缩写为 `i18n`）这个术语指的是将应用程序中的字符串和其他与语言环境相关的内容抽象出来，放到一个层中，以便根据用户的语言环境（即语言和国家）进行翻译和转换。对于文本而言，这意味着将每段文字用一个能够将文本（即"消息"）翻译成用户语言的函数包裹起来：

```php
// 文本将 *始终* 以英文输出
echo 'Hello World';

// 文本可以被翻译成终端用户的语言，
// 或默认为英文
echo $translator->trans('Hello World');
```

> **注意：** 术语"语言环境"大致指用户的语言和国家。它可以是应用程序用于管理翻译和其他格式差异（例如货币格式）的任意字符串。推荐使用 `ISO 639-1` 语言代码，加下划线（`_`），再加 `ISO 3166-1 alpha-2` 国家代码（例如 `fr_FR` 表示法语/法国）。

翻译可以组织成组，称为**域（domains）**。默认情况下，所有消息使用默认的 `messages` 域：

```php
echo $translator->trans('Hello World', domain: 'messages');
```

翻译过程包含以下几个步骤：

1. 启用并配置 Symfony 的翻译服务（参见"配置"部分）；
2. 通过将字符串（即"消息"）包裹在对 `Translator` 的调用中来抽象它们（参见"基本翻译"部分）；
3. 为每种支持的语言环境创建翻译资源/文件，翻译应用程序中的每条消息（参见"翻译资源"部分）；
4. 确定、设置并管理请求的用户语言环境，以及可选地在用户的整个会话中持久化语言环境（参见"处理用户语言环境"部分）。

## 安装

首先，在使用翻译器之前运行以下命令安装它：

```terminal
$ composer require symfony/translation
```

Symfony 包含了若干国际化 polyfill（`symfony/polyfill-intl-icu`、`symfony/polyfill-intl-messageformatter` 等），即使没有 PHP intl 扩展也能使用翻译功能。然而，这些 polyfill 仅支持英文翻译，因此在翻译成其他语言时，必须安装 PHP `intl` 扩展。

## 配置

上述命令会创建一个初始配置文件，你可以在其中定义应用程序的默认语言环境以及翻译文件所在的目录：

**YAML 格式：**

```yaml
# config/packages/translation.yaml
framework:
    default_locale: 'en'
    translator:
        default_path: '%kernel.project_dir%/translations'
```

**PHP 格式：**

```php
// config/packages/translation.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'default_locale' => 'en',
        'translator' => [
            'default_path' => '%kernel.project_dir%/translations',
        ],
    ],
]);
```

> **提示：** 你还可以定义 `enabled_locales` 选项来限制应用程序支持的语言环境。

## 基本翻译

文本的翻译通过 `translator` 服务（`Symfony\Component\Translation\Translator`）完成。要翻译一段文本（称为"消息"），使用 `Symfony\Component\Translation\Translator::trans` 方法。假设你要在控制器内部翻译一条静态消息：

```php
// ...
use Symfony\Contracts\Translation\TranslatorInterface;

public function index(TranslatorInterface $translator): Response
{
    $translated = $translator->trans('Symfony is great');

    // ...
}
```

当这段代码运行时，Symfony 会根据用户的 `locale` 尝试翻译消息"Symfony is great"。为此，你需要通过"翻译资源"告诉 Symfony 如何翻译该消息，翻译资源通常是包含给定语言环境所有翻译的文件集合。这个翻译"字典"可以用多种不同格式创建：

**YAML 格式：**

```yaml
# translations/messages.fr.yaml
Symfony is great: Symfony est génial
```

**XML 格式：**

```xml
<!-- translations/messages.fr.xlf -->
<?xml version="1.0" encoding="UTF-8" ?>
<xliff version="1.2" xmlns="urn:oasis:names:tc:xliff:document:1.2">
    <file source-language="en" datatype="plaintext" original="file.ext">
        <body>
            <trans-unit id="symfony_is_great">
                <source>Symfony is great</source>
                <target>Symfony est génial</target>
            </trans-unit>
        </body>
    </file>
</xliff>
```

**PHP 格式：**

```php
// translations/messages.fr.php
return [
    'Symfony is great' => 'Symfony est génial',
];
```

关于这些文件应存放位置的更多信息，请参见"翻译资源/文件名称和位置"部分。

现在，如果用户的语言环境是法语（例如 `fr_FR` 或 `fr_BE`），消息将被翻译成 `Symfony est génial`。你也可以在模板中翻译消息（参见"模板中的翻译"部分）。

### 使用真实消息还是关键字消息

这个例子展示了创建待翻译消息时的两种不同理念：

```php
$translator->trans('Symfony is great');

$translator->trans('symfony.great');
```

第一种方法，消息用默认语言环境（本例中为英文）的语言书写。该消息随后作为创建翻译时的"id"使用。

第二种方法，消息实际上是传达消息含义的"关键字"。关键字消息随后作为所有翻译的"id"使用。在这种情况下，必须为默认语言环境创建翻译（即将 `symfony.great` 翻译为 `Symfony is great`）。

第二种方法的好处是：如果你决定在默认语言环境中将消息改为"Symfony is really great"，则不需要在每个翻译文件中都修改消息键。

选择使用哪种方法完全取决于你，但"关键字"格式通常推荐用于多语言应用程序。而对于包含翻译资源的共享包，我们推荐使用真实消息，这样你的应用程序可以选择禁用翻译层，并且你会看到可读的消息。

此外，`php` 和 `yaml` 文件格式支持嵌套 id，以避免在将关键字而非真实文本用作 id 时出现重复：

**YAML 格式：**

```yaml
symfony:
    is:
        # id 是 symfony.is.great
        great: Symfony is great
        # id 是 symfony.is.amazing
        amazing: Symfony is amazing
    has:
        # id 是 symfony.has.bundles
        bundles: Symfony has bundles
user:
    # id 是 user.login
    login: Login
```

**PHP 格式：**

```php
[
    'symfony' => [
        'is' => [
            // id 是 symfony.is.great
            'great'   => 'Symfony is great',
            // id 是 symfony.is.amazing
            'amazing' => 'Symfony is amazing',
        ],
        'has' => [
            // id 是 symfony.has.bundles
            'bundles' => 'Symfony has bundles',
        ],
    ],
    'user' => [
        // id 是 user.login
        'login' => 'Login',
    ],
];
```

### 翻译过程

在使用 `trans()` 方法时，Symfony 实际上使用以下流程来翻译消息：

1. 获取当前用户的 `locale`，它存储在请求中；通常通过路由上的 `_locale` 属性来设置；
2. 从为该 `locale`（例如 `fr_FR`）定义的翻译资源中加载已翻译消息的目录。如果回退语言环境和已启用语言环境的消息不存在，也会被加载并添加到目录中。最终结果是一个大型翻译"字典"；
3. 如果消息在目录中找到，则返回翻译。如果没找到，翻译器返回原始消息。

## 消息格式

有时，需要翻译包含变量的消息：

```php
// ...
$translated = $translator->trans('Hello '.$name);
```

然而，无法为这个字符串创建翻译，因为翻译器会尝试查找包含变量部分的消息（例如"Hello Ryan"或"Hello Fabien"）。

你可以用 `%` 字符包裹的**占位符**来替换变量部分：

**YAML 格式：**

```yaml
# translations/messages.en.yaml
say_hello: 'Hello %name%!'
```

**XML 格式：**

```xml
<!-- translations/messages.en.xlf -->
<?xml version="1.0" encoding="UTF-8" ?>
<xliff version="1.2" xmlns="urn:oasis:names:tc:xliff:document:1.2"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="urn:oasis:names:tc:xliff:document:1.2
    https://docs.oasis-open.org/xliff/v1.2/os/xliff-core-1.2-strict.xsd">
    <file source-language="en" datatype="plaintext" original="file.ext">
        <body>
            <trans-unit id="say_hello">
                <source>say_hello</source>
                <target>Hello %name%!</target>
            </trans-unit>
        </body>
    </file>
</xliff>
```

**PHP 格式：**

```php
// translations/messages.en.php
return [
    'say_hello' => 'Hello %name%!',
];
```

然后，将占位符的值作为 `trans()` 方法的第二个参数传入：

```php
// ...
$translated = $translator->trans('say_hello', ['%name%' => 'Fabien']);
// $translated = 'Hello Fabien!'
```

Symfony 使用 PHP 的 `strtr` 函数将占位符替换为给定的值。用 `%` 包裹是一种使占位符易于识别的约定，但这不是必须的。你可以使用任何包裹方式（例如 `#name#` 或 `{name}`），只要参数数组中的键与消息中的占位符完全匹配即可。这种基本机制适用于所有翻译格式，并涵盖大多数使用场景。

当你的翻译需要处理更复杂的场景时，基本占位符就不够用了。例如，你可能需要根据以下情况调整消息：

* 根据数量："There is one apple" 对比 "There are 5 apples"
* 根据性别："He accepted the invitation" 对比 "She accepted the invitation" 对比 "They accepted the invitation"
* 根据用户的语言环境：例如不同的日期格式，如 "Published on January 25, 2026"（英文）对比 "Publicado el 25 de Enero de 2026"（西班牙文）

Symfony 通过 [ICU MessageFormat](../reference/formats/message_format.md) 语法处理所有这些情况，使用 PHP 的 `MessageFormatter` 类。ICU 占位符使用 `{name}` 而非 `%name%`，并且需要在翻译文件名中加上 `+intl-icu` 后缀（例如 `messages+intl-icu.en.yaml`）。更多内容请参见 [/reference/formats/message_format](../reference/formats/message_format.md)。

## 可翻译对象

在许多应用程序中，可翻译文本不只存在于 Twig 模板和控制器中。枚举、服务、值对象和表单类型通常需要生成稍后将被翻译的消息。在创建时翻译这些消息会迫使你在各处注入 `translator` 服务，并在每个测试中进行模拟。

**可翻译对象**通过存储将来翻译所需的所有信息（消息 ID、参数和域），而不实际执行任何翻译，来解决这个问题。当该对象最终到达 Twig 模板或任何其他感知翻译的层时，它会被自动翻译。

Symfony 附带了 `Symfony\Component\Translation\TranslatableMessage`，它实现了 `Symfony\Contracts\Translation\TranslatableInterface`。你可以直接使用它，也可以创建自己对该接口的实现。

### 基本用法

在代码的任意位置创建 `TranslatableMessage`。翻译发生在对象被渲染时：

```php
use Symfony\Component\Translation\TranslatableMessage;

// 使用默认域（"messages"）的可翻译消息
$message = new TranslatableMessage('notification.welcome');

// 带参数和自定义域
$status = new TranslatableMessage(
    'order.status',
    ['%status%' => $order->getStatus()],
    'store'
);
```

在 Twig 中，将可翻译对象像普通字符串一样传给 `trans` 过滤器：

```twig
<h1>{{ message|trans }}</h1>
<p>{{ status|trans }}</p>
```

> **提示：** 使用 `t()` 快捷函数在 Twig 和 PHP 中创建可翻译对象，减少样板代码：
>
> ```php
> use function Symfony\Component\Translation\t;
>
> $message = t('notification.welcome');
> $status  = t('order.status', ['%status%' => $order->getStatus()], 'store');
> ```

> **提示：** `TranslatableMessage` 的翻译参数本身也可以是 `Symfony\Component\Translation\TranslatableMessage` 实例。

### 可翻译对象的实际应用

枚举是用户界面文本的常见来源。考虑一个需要在 UI 中显示翻译标签的 `UserRole` 枚举：

```php
enum UserRole: string
{
    case User  = 'ROLE_USER';
    case Admin = 'ROLE_ADMIN';
}
```

第一种方法是从方法中返回普通字符串或翻译键。然而，这有两个重大缺点：

1. 无法附加翻译参数（例如用于复数化）或指定翻译域。
2. `translation:extract` 命令无法检测这些键，因此不会自动更新翻译文件，而且 `--clean` 选项会错误地将这些键标记为未使用。

使用 `TranslatableMessage` 可以解决这两个问题：

```php
use Symfony\Component\Translation\TranslatableMessage;

enum UserRole: string
{
    case User  = 'ROLE_USER';
    case Admin = 'ROLE_ADMIN';

    public function label(): TranslatableMessage
    {
        return match ($this) {
            self::User  => new TranslatableMessage('user_role.user'),
            self::Admin => new TranslatableMessage('user_role.admin'),
        };
    }
}
```

在 Twig 中，用 `trans` 过滤器渲染标签：

```twig
<span>{{ role.label|trans }}</span>
```

`translation:extract` 命令会检测这些 `TranslatableMessage` 构造函数并保持你的翻译目录是最新的。

### 自定义 TranslatableInterface 实现

如果你需要更多控制，可以直接在任何类上实现 `Symfony\Contracts\Translation\TranslatableInterface`。该接口只需要一个 `trans()` 方法：

```php
use Symfony\Contracts\Translation\TranslatableInterface;
use Symfony\Contracts\Translation\TranslatorInterface;

enum UserRole: string implements TranslatableInterface
{
    case User  = 'ROLE_USER';
    case Admin = 'ROLE_ADMIN';

    public function trans(TranslatorInterface $translator, ?string $locale = null): string
    {
        return $translator->trans('user_role.'.$this->name, locale: $locale);
    }
}
```

任何实现了 `TranslatableInterface` 的对象都可以传给 Twig 的 `trans` 过滤器，并会被自动翻译。当翻译逻辑比静态消息 ID 更复杂时，例如消息依赖于运行时条件时，这种方法非常有用。

### 不可翻译的消息

在某些情况下，你可能希望明确阻止某条消息被翻译。你可以通过使用 `Symfony\Component\Translation\StaticMessage` 类来确保这种行为：

```php
use Symfony\Component\Translation\StaticMessage;

$message = new StaticMessage('This message will never be translated.');
```

这在渲染用户定义的内容或其他必须保持原样的字符串时非常有用。

## 模板中的翻译

大多数时候，翻译发生在模板中。Symfony 为 Twig 和 PHP 模板提供了原生支持。

### 使用 Twig 过滤器

`trans` 过滤器可用于翻译*变量文本*和复杂表达式：

```twig
{{ message|trans }}

{{ message|trans({'%name%': 'Fabien'}, 'app') }}
```

> **提示：** 你可以用一个标签为整个 Twig 模板设置翻译域：
>
> ```twig
> {% trans_default_domain 'app' %}
> ```
>
> 注意，这只影响当前模板，不影响任何"包含"的模板（以避免副作用）。

默认情况下，翻译后的消息会进行输出转义；在翻译过滤器之后应用 `raw` 过滤器可以避免自动转义：

```twig
{% set message = '<h3>foo</h3>' %}

{# 通过过滤器翻译的字符串和变量默认会被转义 #}
{{ message|trans|raw }}
{{ '<h3>bar</h3>'|trans|raw }}
```

### 使用 Twig 标签

Symfony 提供了专门的 Twig 标签 `trans` 来帮助翻译*静态文本块*：

```twig
{% trans %}Hello %name%{% endtrans %}
```

> **警告：** 在 Twig 模板中使用标签进行翻译时，必须使用占位符的 `%var%` 格式。

> **提示：** 如果需要在字符串中使用百分号（`%`），可以将其加倍来转义：`{% trans %}Percent: %percent%%%{% endtrans %}`

你还可以指定消息域并传入一些额外的变量：

```twig
{% trans with {'%name%': 'Fabien'} from 'app' %}Hello %name%{% endtrans %}

{% trans with {'%name%': 'Fabien'} from 'app' into 'fr' %}Hello %name%{% endtrans %}
```

> **警告：** 使用翻译标签与使用过滤器效果相同，但有一个重要区别：使用标签进行翻译时**不会**应用自动输出转义。

## 全局翻译参数

如果某个翻译参数的内容在多个翻译消息中重复出现（例如公司名称或版本号），你可以将其定义为全局翻译参数。这有助于避免在每条消息中手动重复相同的值。

你可以在主配置文件的 `translations.globals` 选项中配置这些全局参数，使用 `%...%` 或 `{...}` 语法：

**YAML 格式：**

```yaml
# config/packages/translator.yaml
translator:
    # ...
    globals:
        # 使用 '%' 包裹字符时，必须对其转义
        '%%app_name%%': 'My application'
        '{app_version}': '1.2.3'
        '{url}': { message: 'url', parameters: { scheme: 'https://' }, domain: 'global' }
```

**PHP 格式：**

```php
// config/packages/translator.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        // ...
        'translator' => [
            'globals' => [
                // 使用 '%' 包裹字符时，必须对其转义
                '%%app_name%%' => 'My application',
                '{app_version}' => '1.2.3',
                '{url}' => ['message' => 'url', 'parameters' => ['scheme' => 'https://']],
            ],
        ],
    ],
]);
```

定义后，你可以在应用程序中的翻译消息中使用这些参数：

```twig
{{ 'Application version: {app_version}'|trans }}
{# 输出："Application version: 1.2.3" #}

{# 传给消息的参数会覆盖全局参数 #}
{{ 'Package version: {app_version}'|trans({'{app_version}': '2.3.4'}) }}
# 显示 "Package version: 2.3.4"
```

## 强制指定翻译器语言环境

翻译消息时，翻译器使用指定的语言环境，必要时使用 `fallback` 语言环境。你也可以手动指定用于翻译的语言环境：

```php
$translator->trans('Symfony is great', locale: 'fr_FR');
```

## 自动提取翻译内容并更新目录

翻译应用程序时最耗时的任务是提取所有需要翻译的模板内容，并保持所有翻译文件同步。Symfony 包含了一个叫做 `translation:extract` 的命令来帮你完成这些任务：

```terminal
# 显示法语需要翻译的所有消息
$ php bin/console translation:extract --dump-messages fr

# 使用该语言环境缺少的字符串更新法语翻译文件
$ php bin/console translation:extract --force fr

# 查看命令帮助，了解其选项（前缀、输出格式、域、排序等）
$ php bin/console translation:extract --help
```

`translation:extract` 命令在以下位置查找缺失的翻译：

* 存储在 `templates/` 目录（或 `twig.default_path` 和 `twig.paths` 配置选项中定义的其他目录）中的模板；
* 任何注入或自动装配了 `translator` 服务并调用 `trans()` 方法的 PHP 文件/类；
* 存储在 `src/` 目录中的任何 PHP 文件/类，这些文件通过构造函数或 `t()` 方法创建可翻译对象，或调用 `trans()` 方法；
* 存储在 `src/` 目录中的任何 PHP 文件/类，这些文件使用带有 `*message` 命名参数的约束属性。

> **提示：** 在项目中安装 `nikic/php-parser` 包可以改进 `translation:extract` 命令的结果。该包启用了一个 AST 解析器，可以找到更多可翻译项：
>
> ```terminal
> $ composer require nikic/php-parser
> ```

默认情况下，当 `translation:extract` 命令在翻译文件中创建新条目时，它将使用相同内容作为源和待翻译内容。唯一的区别是待翻译内容前面加了 `__` 前缀。你可以使用 `--prefix` 选项自定义此前缀：

```terminal
$ php bin/console translation:extract --force --prefix="NEW_" fr
```

或者，你可以使用 `--no-fill` 选项在翻译目录中创建新条目时将待翻译内容完全留空。这在使用外部翻译工具时特别有用，因为它使未翻译的字符串更容易被发现：

```terminal
# 使用 --no-fill 选项时，--prefix 选项被忽略
$ php bin/console translation:extract --force --no-fill fr
```

## 翻译资源/文件名称和位置

Symfony 在以下默认位置查找消息文件（即翻译）：

* 项目根目录下的 `translations/` 目录；
* 任何包内的 `translations/` 目录（以及它们的 `Resources/translations/` 目录，不再推荐用于包）。

这些位置按优先级从高到低列出。也就是说，你可以在第一个目录中覆盖包的翻译消息。包按照 `config/bundles.php` 文件中列出的顺序处理，因此排在前面的包具有更高优先级。

覆盖机制在键级别起作用：只有被覆盖的键需要在更高优先级的消息文件中列出。当在消息文件中找不到某个键时，翻译器会自动回退到较低优先级的消息文件。

翻译文件的文件名也很重要：每个消息文件必须按照以下路径命名：`domain.locale.loader`：

* **domain**：翻译域；
* **locale**：翻译所针对的语言环境（例如 `en_GB`、`en` 等）；
* **loader**：Symfony 应如何加载和解析文件（例如 `xlf`、`php`、`yaml` 等）。

加载器可以是任何已注册加载器的名称。默认情况下，Symfony 提供了许多加载器，根据以下文件扩展名选择：

* `.yaml`：YAML 文件（也可以使用 `.yml` 文件扩展名）；
* `.xlf`：XLIFF 文件（也可以使用 `.xliff` 文件扩展名）；
* `.php`：返回翻译数组的 PHP 文件；
* `.csv`：CSV 文件；
* `.json`：JSON 文件；
* `.ini`：INI 文件；
* `.dat`、`.res`：ICU 资源包；
* `.mo`：机器对象格式；
* `.po`：可移植对象格式；
* `.qt`：QT 翻译 TS XML 文件。

选择使用哪种加载器完全取决于你的喜好。推荐的选项是对简单项目使用 YAML，如果你使用专业程序或团队生成翻译，则使用 XLIFF。

> **警告：** 每次创建*新的*消息目录（或安装包含翻译目录的包）时，请务必清除缓存，以便 Symfony 能够发现新的翻译资源：
>
> ```terminal
> $ php bin/console cache:clear
> ```

> **注意：** 你可以在配置中使用 `paths` 选项添加其他目录：
>
> **YAML 格式：**
>
> ```yaml
> # config/packages/translation.yaml
> framework:
>     translator:
>         paths:
>             - '%kernel.project_dir%/custom/path/to/translations'
> ```
>
> **PHP 格式：**
>
> ```php
> // config/packages/translation.php
> namespace Symfony\Component\DependencyInjection\Loader\Configurator;
>
> return App::config([
>     'framework' => [
>         'translator' => [
>             'paths' => ['%kernel.project_dir%/custom/path/to/translations'],
>         ],
>     ],
> ]);
> ```

### Doctrine 实体的翻译

与模板内容不同，使用翻译目录来翻译存储在 Doctrine 实体中的内容并不实际。应使用 Doctrine 的 Translatable Extension。

### 自定义翻译资源

如果你的翻译使用 Symfony 不支持的格式，或者你以特殊方式存储它们（例如不使用文件或 Doctrine 实体），你需要提供一个实现 `Symfony\Component\Translation\Loader\LoaderInterface` 接口的自定义类。更多信息请参见 `translation.loader` 标签。

## 翻译提供商

使用外部翻译器翻译应用程序时，你必须频繁地将新内容发送给他们进行翻译，并将结果合并回应用程序。

Symfony 不必手动完成这些操作，而是提供了与多个第三方翻译服务的集成。你可以向这些服务上传（称为"推送"）和下载（称为"拉取"）翻译，并自动将结果合并到应用程序中。

### 安装和配置第三方提供商

在推送/拉取翻译到第三方提供商之前，必须安装提供与该提供商集成的包：

| 提供商 | 安装命令 |
|--------|----------|
| Crowdin | `composer require symfony/crowdin-translation-provider` |
| Loco (localise.biz) | `composer require symfony/loco-translation-provider` |
| Lokalise | `composer require symfony/lokalise-translation-provider` |
| Phrase | `composer require symfony/phrase-translation-provider` |

每个库都包含一个 Symfony Flex 配方，会在你的 `.env` 文件中添加一个配置示例。例如，假设你想使用 Loco。首先安装它：

```terminal
$ composer require symfony/loco-translation-provider
```

你的 `.env` 文件中会有一行新内容，你可以取消注释：

```env
# .env
LOCO_DSN=loco://API_KEY@default
```

`LOCO_DSN` 不是一个*真实*地址：它是一种方便的格式，将大部分配置工作转交给 Symfony。`loco` 方案激活了你安装的 Loco 提供商，它了解如何通过 Loco 推送和拉取翻译。你唯一需要更改的是 `API_KEY` 占位符。

下表显示每个提供商的完整 DSN 格式列表：

| 提供商 | DSN |
|--------|-----|
| Crowdin | `crowdin://PROJECT_ID:API_TOKEN@ORGANIZATION_DOMAIN.default` |
| Loco (localise.biz) | `loco://API_KEY@default` |
| Lokalise | `lokalise://PROJECT_ID:API_KEY@default` |
| Phrase | `phrase://PROJECT_ID:API_TOKEN@default?userAgent=myProject` |

要启用翻译提供商，请在 `.env` 文件中自定义 DSN，并配置 `providers` 选项：

**YAML 格式：**

```yaml
# config/packages/translation.yaml
framework:
    translator:
        providers:
            loco:
                dsn: '%env(LOCO_DSN)%'
                domains: ['messages']
                locales: ['en', 'fr']
```

**PHP 格式：**

```php
# config/packages/translation.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'translator' => [
            'providers' => [
                'loco' => [
                    'dsn' => env('LOCO_DSN'),
                    'domains' => ['messages'],
                    'locales' => ['en', 'fr'],
                ],
            ],
        ],
    ],
]);
```

> **注意（谨慎）：** 如果你使用 Phrase 作为提供商，必须在 DSN 中配置 user agent。请参见 Identification via User-Agent 了解原因和示例。
>
> 另外，请确保 Phrase 中的语言环境*名称*符合 RFC4646 的定义（例如使用 pt-BR 而非 pt_BR）。不这样做会导致 Phrase 为导入的键创建新的语言环境。

> **提示：** 如果你使用 Crowdin 作为提供商，而某些语言环境与 Crowdin 语言代码不同，你必须在 Crowdin 项目中为每个语言环境设置自定义语言代码，以覆盖默认值。你需要选择"locale"占位符并在"Custom Code"字段中指定自定义代码。

> **提示：** 如果你使用 Lokalise 作为提供商，且语言环境格式遵循 ISO 639-1（例如"en"或"fr"），你必须在 Lokalise 中为每个语言环境设置自定义语言名称设置，以覆盖默认值（默认值遵循 ISO 639-1，后跟指定国家变体的大写子代码，例如根据 ISO 3166-1 alpha-2 的"GB"或"US"）。

> **提示：** Phrase 提供商使用 Phrase 的标签功能将翻译映射到 Symfony 的翻译域。如果你需要帮助组织 Phrase 中的标签，可以考虑 Phrase Tag Bundle，它提供了一些命令来帮助你。

### 推送和拉取翻译

配置好访问翻译提供商的凭据后，你现在可以使用以下命令来推送（上传）和拉取（下载）翻译：

```terminal
# 将所有本地翻译推送到 Loco 提供商，使用 config/packages/translation.yaml
# 文件中配置的语言环境和域。
# 这将更新提供商上已有的翻译。
$ php bin/console translation:push loco --force

# 将新的本地翻译推送到 Loco 提供商，针对法语语言环境
# 和 validators 域。
# 这**不会**更新提供商上已有的翻译。
$ php bin/console translation:push loco --locales fr --domains validators

# 推送新的本地翻译，并删除提供商上不再存在于本地文件中的翻译
# 针对法语语言环境和 validators 域。
# 这**不会**更新提供商上已有的翻译。
$ php bin/console translation:push loco --delete-missing --locales fr --domains validators

# 查看命令帮助，了解其选项（格式、域、语言环境等）
$ php bin/console translation:push --help
```

```terminal
# 将提供商上的所有翻译拉取到本地文件，使用 config/packages/translation.yaml
# 文件中配置的语言环境和域。
# 这将完全覆盖你的本地文件。
$ php bin/console translation:pull loco --force

# 从 Loco 提供商拉取新翻译到本地文件，针对法语语言环境
# 和 validators 域。
# 这**不会**覆盖你的本地文件，只会添加新翻译。
$ php bin/console translation:pull loco --locales fr --domains validators

# 查看命令帮助，了解其选项（格式、域、语言环境、intl-icu 等）
$ php bin/console translation:pull --help

# "--as-tree" 选项将以树形结构而不是扁平键的形式写入 YAML 消息
$ php bin/console translation:pull loco --force --as-tree
```

### 创建自定义提供商

除了使用 Symfony 内置的翻译提供商之外，你还可以创建自己的提供商。为此，你需要创建两个类：

1. 第一个类必须实现 `Symfony\Component\Translation\Provider\ProviderInterface`；
2. 第二个类需要是一个工厂，用于创建第一个类的实例。它必须实现 `Symfony\Component\Translation\Provider\ProviderFactoryInterface`（你可以扩展 `Symfony\Component\Translation\Provider\AbstractProviderFactory` 来简化其创建）。

创建好这两个类之后，你需要将工厂注册为服务，并用 `translation.provider_factory` 标签标记它。

## 处理用户的语言环境

翻译基于用户的语言环境进行。当前用户的语言环境存储在请求中，可以通过 `Request` 对象访问：

```php
use Symfony\Component\HttpFoundation\Request;

public function index(Request $request): void
{
    $locale = $request->getLocale();
}
```

要设置用户的语言环境，你可能需要创建一个自定义事件监听器，以便在任何其他系统部分（即翻译器）需要之前就将其设置好：

```php
public function onKernelRequest(RequestEvent $event): void
{
    $request = $event->getRequest();

    // 确定 $locale 的一些逻辑
    $request->setLocale($locale);
}
```

> **注意：** 自定义监听器必须在 `LocaleListener` **之前**被调用，后者根据当前请求初始化语言环境。为此，请将你的监听器优先级设置为高于 `LocaleListener` 优先级的值（可以通过运行 `debug:event kernel.request` 命令获得该值）。

关于让用户的语言环境在会话中持久化的更多信息，请参见"粘性会话语言环境"部分。

> **注意：** 在控制器中使用 `$request->setLocale()` 设置语言环境太晚了，无法影响翻译器。请通过监听器（如上所述）、URL（见下文）设置语言环境，或直接在 `translator` 服务上调用 `setLocale()`。

关于通过路由设置语言环境的信息，请参见下面的"语言环境与 URL"部分。

### 语言环境与 URL

由于你可以将用户的语言环境存储在会话中，可能会想使用同一 URL 根据用户的语言环境以不同语言显示资源。例如，`http://www.example.com/contact` 可能对一个用户显示英文内容，对另一个用户显示法文内容。但不幸的是，这违反了 Web 的一个基本规则：特定 URL 无论用户是谁都应返回相同的资源。此外，搜索引擎会索引哪个版本的内容也成问题。

更好的策略是使用特殊的 `_locale` 参数将语言环境包含在 URL 中：

**属性格式：**

```php
// src/Controller/ContactController.php
namespace App\Controller;

// ...
class ContactController extends AbstractController
{
    #[Route(
        path: '/{_locale}/contact',
        name: 'contact',
        requirements: [
            '_locale' => 'en|fr|de',
        ],
    )]
    public function contact(): Response
    {
        // ...
    }
}
```

**YAML 格式：**

```yaml
# config/routes.yaml
contact:
    path:       /{_locale}/contact
    controller: App\Controller\ContactController::index
    requirements:
        _locale: en|fr|de
```

**PHP 格式：**

```php
// config/routes.php
namespace Symfony\Component\Routing\Loader\Configurator;

use App\Controller\ContactController;

return Routes::config([
    'contact' => [
        'path' => '/{_locale}/contact',
        'controller' => [ContactController::class, 'index'],
        'requirements' => [
            '_locale' => 'en|fr|de',
        ],
    ],
]);
```

在路由中使用特殊的 `_locale` 参数时，匹配的语言环境会*自动设置在 Request 上*，可以通过 `Symfony\Component\HttpFoundation\Request::getLocale` 方法获取。换句话说，如果用户访问 URI `/fr/contact`，语言环境 `fr` 将自动设置为当前请求的语言环境。

你现在可以使用该语言环境来创建指向应用程序中其他翻译页面的路由。

> **提示：** 将语言环境要求定义为容器参数，以避免在所有路由中硬编码其值。

### 设置默认语言环境

如果用户的语言环境尚未确定怎么办？你可以通过为框架定义 `default_locale` 来保证每个用户请求都设置了语言环境：

**YAML 格式：**

```yaml
# config/packages/translation.yaml
framework:
    default_locale: en
```

**PHP 格式：**

```php
// config/packages/translation.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'default_locale' => 'en',
    ],
]);
```

如下一节所示，这个 `default_locale` 对翻译器也很重要。

### 选择用户偏好的语言

如果你的应用程序支持多种语言，用户第一次访问网站时，通常会根据其偏好将其重定向到最合适的语言。这通过请求对象的 `getPreferredLanguage()` 方法实现：

```php
// 以某种方式获取 Request 对象（例如作为控制器参数）
$request = ...
// 传入应用程序支持的语言环境数组（脚本和地区部分是可选的），
// 该方法返回当前用户的最佳语言环境
$locale = $request->getPreferredLanguage(['pt', 'fr_Latn_CH', 'en_US'] );
```

Symfony 根据传入的语言环境和 `Accept-Language` HTTP 头的值找到最佳语言。如果找不到完全匹配项，Symfony 会尝试根据语言进行部分匹配（例如 `fr_CA` 会匹配 `fr_Latn_CH`，因为它们的语言相同）。如果既没有完全匹配也没有部分匹配，该方法返回第一个传入的语言环境（这就是传入语言环境的顺序很重要的原因）。

## 回退翻译语言环境

假设用户的语言环境是 `es_AR`，你要翻译键 `Symfony is great`。为了找到西班牙语翻译，Symfony 实际上会检查多个语言环境的翻译资源：

1. 首先，Symfony 在 `es_AR`（阿根廷西班牙语）翻译资源（例如 `messages.es_AR.yaml`）中查找翻译；
2. 如果未找到，Symfony 在父语言环境中查找翻译，父语言环境只会为某些语言环境自动定义。在本例中，父语言环境是 `es_419`（拉丁美洲西班牙语）；
3. 如果未找到，Symfony 在 `es`（西班牙语）翻译资源（例如 `messages.es.yaml`）中查找翻译；
4. 如果翻译仍未找到，Symfony 使用 `fallbacks` 选项，可以按如下方式配置。未定义此选项时，默认为上一节中提到的 `default_locale` 设置。

**YAML 格式：**

```yaml
# config/packages/translation.yaml
framework:
    translator:
        fallbacks: ['en']
        # ...
```

**PHP 格式：**

```php
// config/packages/translation.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

 return [
     'framework' => [
     'translator' => [
         'fallbacks' => ['en'],
     ],
 ],
];
```

> **注意：** 当 Symfony 在给定的语言环境中找不到翻译时，会将缺失的翻译添加到日志文件中。有关详情，请参见 `reference-framework-translator-logging`。

## 以编程方式切换语言环境

有时你需要在运行某些代码时动态更改应用程序的语言环境。例如，一个以不同语言渲染电子邮件模板的控制台命令。在这种情况下，你只需要临时切换语言环境。

`LocaleSwitcher` 类允许你这样做：

```php
use Symfony\Component\Translation\LocaleSwitcher;

class SomeService
{
    public function __construct(
        private LocaleSwitcher $localeSwitcher,
    ) {
    }

    public function someMethod(): void
    {
        $currentLocale = $this->localeSwitcher->getLocale();

        // 以编程方式将应用程序语言环境设置为 'fr'（法语）：
        // 这会影响翻译、URL 生成等。
        $this->localeSwitcher->setLocale('fr');

        // 将语言环境重置为通过 config/packages/translation.yaml
        // 中的 'default_locale' 选项配置的默认语言环境
        $this->localeSwitcher->reset();

        // 使用特定语言环境临时运行某些代码，
        // 而不更改应用程序其余部分的语言环境
        $this->localeSwitcher->runWithLocale('es', function() {
            // 例如，使用 'es'（西班牙语）语言环境渲染模板、发送电子邮件等
        });

        // 可选地，将当前语言环境作为参数接收：
        $this->localeSwitcher->runWithLocale('es', function(string $locale) {

            // 在这里，$locale 参数将被设置为 'es'

        });

        // ...
    }
}
```

`LocaleSwitcher` 类更改以下内容的语言环境：

* 所有标记了 `kernel.locale_aware` 的服务；
* 通过 `\Locale::setDefault()` 设置的默认语言环境；
* `RequestContext` 服务（如果可用）的 `_locale` 参数，使生成的 URL 反映新的语言环境。

> **注意：** LocaleSwitcher 仅对当前请求应用新的语言环境，其效果在后续请求（例如重定向后）中会丢失。
>
> 参见"如何让语言环境在请求之间持久化"部分。

使用自动装配时，用 `Symfony\Component\Translation\LocaleSwitcher` 类对任何控制器或服务参数进行类型提示，以注入语言环境切换器服务。否则，请手动配置你的服务并注入 `translation.locale_switcher` 服务。

## 如何查找缺失或未使用的翻译消息

当你处理多种语言的许多翻译消息时，很难追踪哪些翻译缺失以及哪些不再使用。`debug:translation` 命令帮助你在模板中找到这些缺失或未使用的翻译消息：

```twig
{# 使用 trans 过滤器和标签时可以找到消息 #}
{% trans %}Symfony is great{% endtrans %}

{{ 'Symfony is great'|trans }}
```

> **警告：** 提取器无法找到在模板之外翻译的消息（例如表单标签或控制器），除非使用可翻译对象或在翻译器上调用 `trans()` 方法（自 Symfony 5.3 起）。模板中使用变量或表达式的动态翻译也不会被检测到：
>
> ```twig
> {# 这个翻译使用了 Twig 变量，因此不会被检测到 #}
> {% set message = 'Symfony is great' %}
> {{ message|trans }}
> ```

假设你的应用程序的 `default_locale` 是 `fr`，你将 `en` 配置为回退语言环境（参见配置和回退部分了解如何配置这些）。并假设你已经为 `fr` 语言环境设置了一些翻译：

**XML 格式：**

```xml
<!-- translations/messages.fr.xlf -->
<?xml version="1.0" encoding="UTF-8" ?>
<xliff version="1.2" xmlns="urn:oasis:names:tc:xliff:document:1.2">
    <file source-language="en" datatype="plaintext" original="file.ext">
        <body>
            <trans-unit id="1">
                <source>Symfony is great</source>
                <target>Symfony est génial</target>
            </trans-unit>
        </body>
    </file>
</xliff>
```

**YAML 格式：**

```yaml
# translations/messages.fr.yaml
Symfony is great: Symfony est génial
```

**PHP 格式：**

```php
// translations/messages.fr.php
return [
    'Symfony is great' => 'Symfony est génial',
];
```

以及 `en` 语言环境的翻译：

**XML 格式：**

```xml
<!-- translations/messages.en.xlf -->
<?xml version="1.0" encoding="UTF-8" ?>
<xliff version="1.2" xmlns="urn:oasis:names:tc:xliff:document:1.2">
    <file source-language="en" datatype="plaintext" original="file.ext">
        <body>
            <trans-unit id="1">
                <source>Symfony is great</source>
                <target>Symfony is great</target>
            </trans-unit>
        </body>
    </file>
</xliff>
```

**YAML 格式：**

```yaml
# translations/messages.en.yaml
Symfony is great: Symfony is great
```

**PHP 格式：**

```php
// translations/messages.en.php
return [
    'Symfony is great' => 'Symfony is great',
];
```

要检查应用程序中 `fr` 语言环境的所有消息，运行：

```terminal
$ php bin/console debug:translation fr

---------  ------------------  ----------------------  -------------------------------
 State      Id                  Message Preview (fr)    Fallback Message Preview (en)
---------  ------------------  ----------------------  -------------------------------
 unused     Symfony is great    Symfony est génial      Symfony is great
---------  ------------------  ----------------------  -------------------------------
```

它会显示一个表格，显示在 `fr` 语言环境中翻译消息的结果，以及使用回退语言环境 `en` 时的结果。此外，当翻译与回退翻译相同时，它也会提示（这可能表明消息未被正确翻译）。此外，它还会显示消息 `Symfony is great` 是未使用的，因为它虽然被翻译了，但你还没有在任何地方使用它。

现在，如果你在某个模板中翻译了该消息，你将得到以下输出：

```terminal
$ php bin/console debug:translation fr

---------  ------------------  ----------------------  -------------------------------
 State      Id                  Message Preview (fr)    Fallback Message Preview (en)
---------  ------------------  ----------------------  -------------------------------
            Symfony is great    Symfony est génial      Symfony is great
---------  ------------------  ----------------------  -------------------------------
```

状态为空，表示消息已在 `fr` 语言环境中翻译，并在一个或多个模板中使用。

如果你从 `fr` 语言环境的翻译文件中删除消息 `Symfony is great` 并运行该命令，你将得到：

```terminal
$ php bin/console debug:translation fr

---------  ------------------  ----------------------  -------------------------------
 State      Id                  Message Preview (fr)    Fallback Message Preview (en)
---------  ------------------  ----------------------  -------------------------------
 missing    Symfony is great    Symfony is great        Symfony is great
---------  ------------------  ----------------------  -------------------------------
```

状态表示消息缺失，因为它在 `fr` 语言环境中没有被翻译，但仍在模板中使用。此外，`fr` 语言环境中的消息等于 `en` 语言环境中的消息。这是一种特殊情况，因为未翻译的消息 id 等于其在 `en` 语言环境中的翻译。

如果你将 `en` 语言环境翻译文件的内容复制到 `fr` 语言环境翻译文件，并运行该命令，你将得到：

```terminal
$ php bin/console debug:translation fr

----------  ------------------  ----------------------  -------------------------------
 State       Id                  Message Preview (fr)    Fallback Message Preview (en)
----------  ------------------  ----------------------  -------------------------------
 fallback    Symfony is great    Symfony is great        Symfony is great
----------  ------------------  ----------------------  -------------------------------
```

你可以看到 `fr` 和 `en` 语言环境中该消息的翻译是相同的，这意味着这条消息可能是从英语复制到法语的，你可能忘记翻译它了。

默认情况下，所有域都会被检查，但可以指定单个域：

```terminal
$ php bin/console debug:translation en --domain=messages
```

当应用程序有许多消息时，使用 `--only-unused` 或 `--only-missing` 选项只显示未使用或缺失的消息会很有用：

```terminal
$ php bin/console debug:translation en --only-unused
$ php bin/console debug:translation en --only-missing
```

### debug 命令退出码

`debug:translation` 命令的退出码根据翻译的状态而变化。使用以下公共常量来检查它：

```php
use Symfony\Bundle\FrameworkBundle\Command\TranslationDebugCommand;

// 一般失败（例如没有翻译）
TranslationDebugCommand::EXIT_CODE_GENERAL_ERROR;

// 存在缺失的翻译
TranslationDebugCommand::EXIT_CODE_MISSING;

// 存在未使用的翻译
TranslationDebugCommand::EXIT_CODE_UNUSED;

// 某些翻译使用了回退翻译
TranslationDebugCommand::EXIT_CODE_FALLBACK;
```

这些常量被定义为"位掩码"，因此你可以如下组合使用：

```php
if (TranslationDebugCommand::EXIT_CODE_MISSING | TranslationDebugCommand::EXIT_CODE_UNUSED) {
    // ... 存在缺失和/或未使用的翻译
}
```

## 如何在翻译文件中查找错误

Symfony 在执行应用程序代码之前，会处理所有应用程序翻译文件作为编译过程的一部分。如果任何翻译文件有错误，你会看到一条解释问题的错误消息。

如果你愿意，还可以使用 `lint:yaml` 和 `lint:xliff` 命令验证任何 YAML 和 XLIFF 翻译文件的语法：

```terminal
# 检查单个文件
$ php bin/console lint:yaml translations/messages.en.yaml
$ php bin/console lint:xliff translations/messages.en.xlf

# 检查整个目录
$ php bin/console lint:yaml translations
$ php bin/console lint:xliff translations

# 检查多个文件或目录
$ php bin/console lint:yaml translations path/to/trans
$ php bin/console lint:xliff translations/messages.en.xlf translations/messages.es.xlf
```

可以使用 `--format` 选项将检查结果导出为 JSON：

```terminal
$ php bin/console lint:yaml translations/ --format=json
$ php bin/console lint:xliff translations/ --format=json
```

在 GitHub Actions 中运行这些检查器时，输出会自动适配为 GitHub 所需的格式，但你也可以强制使用该格式：

```terminal
$ php bin/console lint:yaml translations/ --format=github
$ php bin/console lint:xliff translations/ --format=github
```

> **提示：** Yaml 组件提供了一个独立的 `yaml-lint` 二进制文件，允许你在不创建控制台应用程序的情况下检查 YAML 文件：
>
> ```terminal
> $ php vendor/bin/yaml-lint translations/
> ```

`lint:yaml` 和 `lint:xliff` 命令验证翻译文件的 YAML 和 XML 语法，但不验证其内容。使用以下命令检查翻译内容是否也正确：

```terminal
# 检查所有语言环境中所有翻译目录的内容
$ php bin/console lint:translations

# 检查意大利语（it）和日语（ja）语言环境的翻译目录内容
$ php bin/console lint:translations --locale=it --locale=ja
```

## 测试翻译

Symfony 提供了简化翻译相关代码和功能测试的工具。

### 身份翻译器

测试使用翻译服务的功能时，通常不需要验证实际翻译的内容。你可以使用 `Symfony\Component\Translation\IdentityTranslator`，而无需模拟 `Symfony\Contracts\Translation\TranslatorInterface`，它实现了该接口而不加载任何翻译目录。

`IdentityTranslator` 不查找翻译，而是在应用参数替换和消息选择（例如复数化）之后始终返回原始消息：

```php
use Symfony\Component\Translation\IdentityTranslator;

$translator = new IdentityTranslator();

// 使用关键字键时，返回键本身
$translator->trans('app.greeting');
// => "app.greeting"
$translator->trans('app.greeting', ['%name%' => 'Fabien']);
// => "app.greeting"

// 使用真实消息作为键时，参数会替换键中的占位符
$translator->trans('Hello %name%!', ['%name%' => 'Fabien']);
// => "Hello Fabien!"

// 消息选择（包括复数化）仍然适用
$translator->trans('{0} No results|one result|%count% results', ['%count%' => 3]);
// => "3 results"
```

语言环境默认为 `\Locale::getDefault()`（或在 `intl` 扩展不可用时为 `en`），可以使用 `setLocale()` 更改。语言环境只影响消息选择；不会使用任何翻译目录。

### 伪本地化翻译器

> **注意：** 伪本地化翻译器仅用于开发阶段。

下图显示了一个典型的网页菜单：

![菜单显示多个整齐排列的项目。](/_images/translation/pseudolocalization-interface-original.png)

另一张图显示了当用户将语言切换为西班牙语时的同一菜单。出乎意料的是，有些文字被截断，其他内容因为太长而溢出，无法看到：

![西班牙语中，某些菜单项包含更多字母，导致它们被截断。](/_images/translation/pseudolocalization-interface-translated.png)

这类错误非常常见，因为不同语言的文本可能比原始应用程序语言更长或更短。另一个常见问题是只检查应用程序在使用基本重音字母时是否正常工作，而不检查更复杂的字符，例如波兰语、捷克语等中的字符。

这些问题可以通过伪本地化来解决，伪本地化是一种用于测试国际化的软件测试方法。在这种方法中，不是将软件的文本翻译成外语，而是用原始语言的修改版本替换应用程序的文本元素。

例如，`Account Settings` 被"翻译"为 `[!!! Àççôûñţ Šéţţîñĝš !!!]`。首先，原始文本用 `[!!! !!!]` 等字符在长度上进行扩展，以测试使用比原始语言更冗长的语言时的应用程序。这解决了第一个问题。

此外，原始字符被替换为类似但带重音的字符。这使文本高度可读，同时允许测试各种重音和特殊字符的应用程序。这解决了第二个问题。

已添加对伪本地化的完整支持，以帮助你调试应用程序中的国际化问题。你可以在翻译器配置中启用和配置它：

**YAML 格式：**

```yaml
# config/packages/translation.yaml
framework:
    translator:
        pseudo_localization:
            # 将字符替换为其带重音的版本
            accents: true
            # 用括号包裹字符串
            brackets: true
            # 控制添加多少额外字符以使文本更长
            expansion_factor: 1.4
            # 保留翻译内容的原始 HTML 标签
            parse_html: true
            # 还翻译这些 HTML 属性的内容
            localizable_html_attributes: ['title']
```

**PHP 格式：**

```php
// config/packages/translation.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'translator' => [
            'pseudo_localization' => [
                // 将字符替换为其带重音的版本
                'accents' => true,
                // 用括号包裹字符串
                'brackets' => true,
                // 控制添加多少额外字符以使文本更长
                'expansion_factor' => 1.4,
                // 保留翻译内容的原始 HTML 标签
                'parse_html' => true,
                // 还翻译这些 HTML 属性的内容
                'localizable_html_attributes' => ['title'],
            ],
        ],
    ],
]);
```

就这些。应用程序现在将开始显示这些奇怪但可读的内容，以帮助你将其国际化。例如，可以在 Symfony Demo 应用程序中看到差异。这是原始页面：

![Symfony demo 登录页面。](/_images/translation/pseudolocalization-symfony-demo-disabled.png)

这是启用伪本地化后的同一页面：

![启用伪本地化的 Symfony demo 登录页面。](/_images/translation/pseudolocalization-symfony-demo-enabled.png)

## 总结

借助 Symfony 翻译组件，创建国际化应用程序不再是一个痛苦的过程，可以归结为以下几个步骤：

* 通过将应用程序中的每条消息包裹在 `Symfony\Component\Translation\Translator::trans` 方法中来抽象消息；
* 通过创建翻译消息文件将每条消息翻译成多种语言环境。Symfony 会发现并处理每个文件，因为其名称遵循特定的约定；
* 管理用户的语言环境，它存储在请求中，但也可以在用户的会话中设置。

## 了解更多

* [消息格式](../reference/formats/message_format.md)
* [XLIFF 格式](../reference/formats/xliff.md)
