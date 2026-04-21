# CssSelector 组件

CssSelector 组件将 CSS 选择器转换为 [XPath][] 表达式。

## 安装

```terminal
$ composer require symfony/css-selector
```

## 使用

> **参见：** 本文介绍了如何在任意 PHP 应用程序中将 CssSelector 功能作为独立组件使用。请阅读 Symfony 功能测试文章，了解如何在创建 Symfony 测试时使用它。

### 为什么使用 CSS 选择器？

在解析 HTML 或 XML 文档时，最强大的方法是 [XPath][]。

XPath 表达式极为灵活，几乎总能找到你所需元素的 XPath 表达式。但不幸的是，它们也可能变得非常复杂，学习曲线较陡。即使是常见操作（例如查找具有特定类的元素）也可能需要冗长而笨拙的表达式。

许多开发者——尤其是 Web 开发者——更习惯使用 CSS 选择器来查找元素。除了在样式表中使用外，CSS 选择器还在 JavaScript 中通过 `querySelectorAll()` 函数以及 jQuery 等流行 JavaScript 库中使用。

CSS 选择器没有 XPath 那么强大，但编写、阅读和理解起来更容易得多。由于它们功能较少，几乎所有 CSS 选择器都可以转换为等效的 XPath 表达式。然后，该 XPath 表达式可与其他使用 XPath 在文档中查找元素的函数和类一起使用。

### CssSelector 组件

该组件的唯一目标是使用 `CssSelectorConverter::toXPath` 将 CSS 选择器转换为其等效的 XPath 表达式：

```php
use Symfony\Component\CssSelector\CssSelectorConverter;

$converter = new CssSelectorConverter();
var_dump($converter->toXPath('div.item > h4 > a'));
```

这将给出以下输出：

```text
descendant-or-self::div[@class and contains(concat(' ',normalize-space(@class), ' '), ' item ')]/h4/a
```

你可以将此表达式与 `DOMXPath` 或 `SimpleXMLElement` 等类一起使用，以在文档中查找元素。

> **提示：** `Crawler::filter()` 方法使用 CssSelector 组件根据 CSS 选择器字符串查找元素。有关更多详细信息，请参阅 [dom_crawler](dom_crawler.md) 文档。

### CssSelector 组件的限制

并非所有 CSS 选择器都能转换为等效的 [XPath][] 表达式。

有几种 CSS 选择器仅在 Web 浏览器的上下文中有意义。

* 链接状态选择器：`:link`、`:visited`、`:target`
* 基于用户操作的选择器：`:hover`、`:focus`、`:active`
* UI 状态选择器：`:invalid`、`:indeterminate`（但 `:enabled`、`:disabled`、`:checked` 和 `:unchecked` 是可用的）

伪元素（`:before`、`:after`、`:first-line`、`:first-letter`）不受支持，因为它们选择的是文本片段而非元素。

伪类部分受支持：

* 不支持：`*:first-of-type`、`*:last-of-type`、`*:nth-of-type` 和 `*:nth-last-of-type`（这些在使用元素名称时有效，例如 `li:first-of-type`，但不支持 `*` 选择器）。
* 支持：`*:only-of-type`、`*:scope`、`*:is` 和 `*:where`。

## 了解更多

* [测试](testing.md)
* [DomCrawler 组件](dom_crawler.md)

[XPath]: https://en.wikipedia.org/wiki/XPath
