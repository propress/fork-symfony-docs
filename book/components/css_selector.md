# CssSelector 组件

CssSelector 组件将 CSS 选择器转换为 [XPath][xpath] 表达式。

## 安装

```bash
$ composer require symfony/css-selector
```

如果您在 Symfony 应用之外使用此组件，则必须在代码中引入 Composer 生成的 `vendor/autoload.php` 文件来启用类自动加载机制。更多信息请阅读[此文章](./using_components.md)。

## 用法

> [!NOTE]
> 本文介绍如何在任何 PHP 应用中将 CssSelector 功能用作独立组件。阅读 [Symfony 功能测试](../../testing.md)文章以了解如何在创建 Symfony 测试时使用它。

### 为什么使用 CSS 选择器？

当您解析 HTML 或 XML 文档时，到目前为止最强大的方法是 [XPath][xpath]。

XPath 表达式非常灵活，因此几乎总有一个 XPath 表达式可以找到您需要的元素。不幸的是，它们也可能变得非常复杂，学习曲线很陡峭。即使是常见操作（例如查找具有特定类的元素）也可能需要冗长且笨拙的表达式。

许多开发人员 - 特别是 Web 开发人员 - 更习惯于使用 CSS 选择器来查找元素。除了在样式表中工作外，CSS 选择器还在 JavaScript 中与 `querySelectorAll()` 函数以及流行的 JavaScript 库（如 jQuery）一起使用。

CSS 选择器不如 XPath 强大，但更易于编写、阅读和理解。由于它们不那么强大，几乎所有 CSS 选择器都可以转换为等效的 XPath。然后可以将此 XPath 表达式与使用 XPath 在文档中查找元素的其他函数和类一起使用。

### CssSelector 组件

该组件的唯一目标是使用 `Symfony\Component\CssSelector\CssSelectorConverter::toXPath` 将 CSS 选择器转换为其 XPath 等效项：

```php
use Symfony\Component\CssSelector\CssSelectorConverter;

$converter = new CssSelectorConverter();
var_dump($converter->toXPath('div.item > h4 > a'));
```

这给出以下输出：

```text
descendant-or-self::div[@class and contains(concat(' ',normalize-space(@class), ' '), ' item ')]/h4/a
```

您可以将此表达式与例如 `DOMXPath` 或 `SimpleXMLElement` 一起使用以在文档中查找元素。

> [!TIP]
> `Crawler::filter()` 方法使用 CssSelector 组件根据 CSS 选择器字符串查找元素。有关更多详细信息，请参阅 [DomCrawler 组件](./dom_crawler.md)。

### CssSelector 组件的限制

并非所有 CSS 选择器都可以转换为 [XPath][xpath] 等效项。

有几个 CSS 选择器仅在 Web 浏览器的上下文中才有意义。

* 链接状态选择器：`:link`、`:visited`、`:target`
* 基于用户操作的选择器：`:hover`、`:focus`、`:active`
* UI 状态选择器：`:invalid`、`:indeterminate`（但是，`:enabled`、`:disabled`、`:checked` 和 `:unchecked` 是可用的）

不支持伪元素（`:before`、`:after`、`:first-line`、`:first-letter`），因为它们选择的是文本部分而不是元素。

部分支持伪类：

* 不支持：`*:first-of-type`、`*:last-of-type`、`*:nth-of-type` 和 `*:nth-last-of-type`（所有这些都适用于元素名称（例如 `li:first-of-type`）但不适用于 `*` 选择器）。
* 支持：`*:only-of-type`、`*:scope`、`*:is` 和 `*:where`。

## 了解更多

* [测试](../../testing.md)
* [DomCrawler 组件](./dom_crawler.md)

[xpath]: https://en.wikipedia.org/wiki/XPath
