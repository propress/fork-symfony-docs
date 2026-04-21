# HTML 清理器

HTML 清理器组件旨在将不受信任的 HTML 代码（例如由浏览器中的 WYSIWYG 编辑器创建的）清理/净化为可信任的 HTML。它基于 [HTML 清理器 W3C 标准提案](https://wicg.github.io/sanitizer-api/)。

HTML 清理器从头开始创建新的 HTML 结构，只采用配置允许的元素和属性。这意味着返回的 HTML 是非常可预测的（它只包含允许的元素），但它对格式错误的输入（例如无效的 HTML）处理不好。清理器针对两个用例：

- 防止基于 [XSS](#xss-attacks) 或其他依赖在访客浏览器上执行恶意代码的技术的安全攻击；
- 生成始终遵循特定格式（仅某些标签、属性、主机等）的 HTML，以便能够一致地使用 CSS 样式化结果输出。这也保护你的应用免受例如更改整个页面 CSS 的攻击。

---

## 安装 {#html-sanitizer-installation}

你可以使用以下命令安装 HTML 清理器组件：

```terminal
$ composer require symfony/html-sanitizer
```

---

## 基本用法

使用 `HtmlSanitizer` 类来清理 HTML。在 Symfony 框架中，此类可作为 `html_sanitizer` 服务使用。当对 `HtmlSanitizerInterface` 进行类型提示时，此服务将自动[自动装配](service_container/autowiring.md)：

**Symfony 应用：**

```php
// src/Controller/BlogPostController.php
namespace App\Controller;

// ...
use Symfony\Component\HtmlSanitizer\HtmlSanitizerInterface;

class BlogPostController extends AbstractController
{
    public function createAction(HtmlSanitizerInterface $htmlSanitizer, Request $request): Response
    {
        $unsafeContents = $request->getPayload()->get('post_contents');

        $safeContents = $htmlSanitizer->sanitize($unsafeContents);
        // ... 继续使用安全的 HTML
    }
}
```

**独立 PHP 应用：**

```php
use Symfony\Component\HtmlSanitizer\HtmlSanitizer;
use Symfony\Component\HtmlSanitizer\HtmlSanitizerConfig;

$htmlSanitizer = new HtmlSanitizer(
    new HtmlSanitizerConfig()->allowSafeElements()
);

// 不安全的 HTML（例如来自浏览器中的 WYSIWYG 编辑器）
$unsafePostContents = ...;

$safePostContents = $htmlSanitizer->sanitize($unsafePostContents);
// ... 继续使用安全的 HTML
```

> **注意**
>
> HTML 清理器的默认配置允许所有"安全"元素和属性，如 [W3C 标准提案](https://wicg.github.io/sanitizer-api/)所定义。在实践中，这意味着结果代码将不包含任何脚本、样式或其他可能导致网站行为或外观不同的元素。

---

## 为特定上下文清理 HTML

默认的 `sanitize()` 方法清理 HTML 代码以在 `<body>` 元素中使用。使用 `sanitizeFor()` 方法，你可以指示 HTML 清理器针对 `<head>` 或更特定的 HTML 标签进行自定义：

```php
// <head> 中不允许的标签将被删除
$safeInput = $htmlSanitizer->sanitizeFor('head', $userInput);

// 使用 HTML 实体对返回的 HTML 进行编码
$safeInput = $htmlSanitizer->sanitizeFor('title', $userInput);
$safeInput = $htmlSanitizer->sanitizeFor('textarea', $userInput);

// 使用 <body> 上下文，删除只在 <head> 中允许的标签
$safeInput = $htmlSanitizer->sanitizeFor('body', $userInput);
$safeInput = $htmlSanitizer->sanitizeFor('section', $userInput);
```

---

## 从表单输入清理 HTML

HTML 清理器组件直接与 Symfony 表单集成，以在你的应用处理表单输入之前对其进行清理。

你可以使用 `sanitize_html` 选项在 `TextType` 表单或扩展此类型的任何表单（如 `TextareaType`）中启用清理器：

```php
// src/Form/BlogPostType.php
namespace App\Form;

// ...
class BlogPostType extends AbstractType
{
    // ...

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'sanitize_html' => true,
            // 使用 "sanitizer" 选项来使用自定义清理器（见下文）
            //'sanitizer' => 'app.post_sanitizer',
        ]);
    }
}
```

---

## 在 Twig 模板中清理 HTML {#html-sanitizer-twig}

除了清理用户输入外，你还可以使用 `sanitize_html()` 过滤器在 Twig 模板中输出 HTML 代码之前对其进行清理：

```twig
{{ post.body|sanitize_html }}

{# 你也可以使用自定义清理器（见下文）#}
{{ post.body|sanitize_html('app.post_sanitizer') }}
```

---

## 配置 {#html-sanitizer-configuration}

HTML 清理器的行为可以完全自定义。这允许你明确声明允许哪些元素、属性甚至属性值。

你可以通过在配置中定义新的 HTML 清理器来实现：

```yaml
# config/packages/html_sanitizer.yaml
framework:
    html_sanitizer:
        sanitizers:
            app.post_sanitizer:
                block_elements:
                    - h1
```

此配置定义了一个新的 `html_sanitizer.sanitizer.app.post_sanitizer` 服务。现在你有两种方式在任何服务或控制器中注入它：

**（1）使用特定的参数名称**

将你的构造函数/方法参数类型提示为 `HtmlSanitizerInterface`，并使用此模式命名参数："camelCase 的 HTML 清理器名称"。例如，要注入之前定义的 `app.post_sanitizer`，使用名为 `$appPostSanitizer` 的参数：

```php
// src/Controller/ApiController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HtmlSanitizer\HtmlSanitizerInterface;

class BlogController extends AbstractController
{
    public function __construct(
        private HtmlSanitizerInterface $appPostSanitizer,
    ) {
    }

    // ...
}
```

**（2）使用 `#[Target]` 属性**

当处理同类型的多个实现时，`#[Target]` 属性帮助你选择要注入的那个。Symfony 创建一个与 HTML 清理器同名的目标：

```php
// ...
use Symfony\Component\DependencyInjection\Attribute\Target;

class BlogController extends AbstractController
{
    public function __construct(
        #[Target('app.post_sanitizer')]
        private HtmlSanitizerInterface $sanitizer,
    ) {
    }

    // ...
}
```

### 允许元素基准

你可以使用两个基准之一来启动自定义 HTML 清理器：

- **静态元素**：[W3C 标准提案](https://wicg.github.io/sanitizer-api/)允许列表中的所有元素和属性（不包括脚本）；
- **安全元素**：来自"静态元素"列表的所有元素和属性，不包括也可能导致 CSS 注入/点击劫持的元素和属性。

```yaml
# config/packages/html_sanitizer.yaml
framework:
    html_sanitizer:
        sanitizers:
            app.post_sanitizer:
                # 启用其中之一
                allow_safe_elements: true
                allow_static_elements: true
```

### 允许元素

这将元素添加到允许列表中。对于每个元素，你还可以指定该元素上允许的属性。如果未给出，则允许 [W3C 标准提案](https://wicg.github.io/sanitizer-api/)中的所有允许属性：

```yaml
# config/packages/html_sanitizer.yaml
framework:
    html_sanitizer:
        sanitizers:
            app.post_sanitizer:
                # ...
                allow_elements:
                    # 允许 <article> 元素和 2 个属性
                    article: ['class', 'data-attr']
                    # 允许 <img> 元素并保留 src 属性
                    img: 'src'
                    # 允许 <h1> 元素和所有安全属性
                    h1: '*'
                    # 允许没有属性的 <div> 元素
                    div: []
```

### 阻止和删除元素

你也可以阻止元素（元素将被删除，但其子元素将被保留）或删除元素（元素及其子元素将被删除）：

```yaml
# config/packages/html_sanitizer.yaml
framework:
    html_sanitizer:
        sanitizers:
            app.post_sanitizer:
                # ...

                # 删除 <div>，但处理子元素
                block_elements: ['div']
                # 删除 <figure> 及其子元素
                drop_elements: ['figure']
```

### 允许属性

使用此选项，你可以指定哪些属性将保留在返回的 HTML 中：

```yaml
# config/packages/html_sanitizer.yaml
framework:
    html_sanitizer:
        sanitizers:
            app.post_sanitizer:
                # ...
                allow_attributes:
                    # 在 <iframe> 元素上允许 "src"
                    src: ['iframe']

                    # 在当前允许的所有元素上允许 "data-attr"
                    data-attr: '*'
```

### 删除属性

此选项允许你禁止之前允许的属性：

```yaml
# config/packages/html_sanitizer.yaml
framework:
    html_sanitizer:
        sanitizers:
            app.post_sanitizer:
                # ...
                allow_attributes:
                    # 在所有安全元素上允许 "data-attr"...
                    data-attr: '*'

                drop_attributes:
                    # ...除了 <section> 元素
                    data-attr: ['section']
                    # 禁止在任何允许的元素上使用 "style"
                    style: '*'
```

### 强制属性值

使用此选项，你可以在元素上强制使用给定值的属性。例如，使用以下配置始终在每个 `<a>` 元素上设置 `rel="noopener noreferrer"`（即使原始元素不包含 `rel` 属性）：

```yaml
# config/packages/html_sanitizer.yaml
framework:
    html_sanitizer:
        sanitizers:
            app.post_sanitizer:
                # ...
                force_attributes:
                    a:
                        rel: noopener noreferrer
```

### 强制/允许链接 URL {#html-sanitizer-link-url}

除了允许/阻止元素和属性外，你还可以控制 `<a>` 元素的 URL：

```yaml
# config/packages/html_sanitizer.yaml
framework:
    html_sanitizer:
        sanitizers:
            app.post_sanitizer:
                # ...

                # 如果为 `true`，使用 `http://` 方案的所有 URL 将被转换为使用 `https://` 方案
                # `http` 仍需要在 `allowed_link_schemes` 中允许
                force_https_urls: true

                # 指定允许的 URL 方案。如果 URL 具有不同的方案，则该属性将被删除
                allowed_link_schemes: ['http', 'https', 'mailto']

                # 指定允许的主机，如果 URL 包含不同的主机，则该属性将被删除
                # 子域名是允许的：例如以下配置也允许 'www.symfony.com'、'live.symfony.com' 等
                allowed_link_hosts: ['symfony.com']

                # 是否允许相对链接（即没有方案和主机的 URL）
                allow_relative_links: true
```

### 强制/允许媒体 URL

与[链接 URL](#html-sanitizer-link-url) 类似，你也可以控制 HTML 中其他媒体的 URL。HTML 清理器检查以下属性：`src`、`href`、`lowsrc`、`background` 和 `ping`：

```yaml
# config/packages/html_sanitizer.yaml
framework:
    html_sanitizer:
        sanitizers:
            app.post_sanitizer:
                # ...
                force_https_urls: true
                allowed_media_schemes: ['http', 'https', 'mailto']
                allowed_media_hosts: ['symfony.com']
                allow_relative_medias: true
```

### 最大输入长度

为了防止 [DoS 攻击](https://en.wikipedia.org/wiki/Denial-of-service_attack)，默认情况下 HTML 清理器将输入长度限制为 `20000` 个字符（由 `strlen($input)` 测量）。超过该长度的所有内容将被截断。使用此选项增加或减少此限制：

```yaml
# config/packages/html_sanitizer.yaml
framework:
    html_sanitizer:
        sanitizers:
            app.post_sanitizer:
                # ...

                # 输入长度（以字符为单位）超过此值将被截断
                max_input_length: 30000 # 默认值：20000
```

通过将最大输入长度设置为 `-1` 可以禁用此长度限制。请注意，这可能会使你的应用暴露于 DoS 攻击。

### 自定义属性清理器

控制链接和媒体 URL 由 `UrlAttributeSanitizer` 完成。你也可以实现自己的属性清理器，来控制 HTML 中其他属性的值。创建一个实现 `AttributeSanitizerInterface` 的类并将其注册为服务。然后，使用 `with_attribute_sanitizers` 为 HTML 清理器启用它：

```yaml
# config/packages/html_sanitizer.yaml
framework:
    html_sanitizer:
        sanitizers:
            app.post_sanitizer:
                # ...
                with_attribute_sanitizers:
                    - App\Sanitizer\CustomAttributeSanitizer

                # 你也可以禁用之前启用的自定义属性清理器
                #without_attribute_sanitizers:
                #    - App\Sanitizer\CustomAttributeSanitizer
```
