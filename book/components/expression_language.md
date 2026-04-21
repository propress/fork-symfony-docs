# ExpressionLanguage 组件

ExpressionLanguage 组件提供了一个可以编译和求值表达式的引擎。表达式是一行代码，返回一个值（大多数情况下，但不限于，是布尔值）。

## 安装

```terminal
$ composer require symfony/expression-language
```

## 表达式语言如何帮助我？

该组件的目的是让用户在配置中使用表达式来实现更复杂的逻辑。例如，Symfony 框架在安全性、验证规则和路由匹配中使用表达式。

除了在框架本身中使用该组件外，ExpressionLanguage 组件是构建*业务规则引擎*的完美候选。其思路是让网站管理员以动态方式配置内容，而无需使用 PHP 且不引入安全问题：

```text
# Get the special price if
user.getGroup() in ['good_customers', 'collaborator']

# Promote article to the homepage when
article.commentCount > 100 and article.category not in ["misc"]

# Send an alert when
product.stock < 15
```

表达式可以被视为一个非常受限的 PHP 沙箱，对外部注入的脆弱性更低，因为你必须明确声明表达式中可用的变量（但你仍然应该对最终用户提供并传递给表达式的任何数据进行清理）。

## 使用

ExpressionLanguage 组件可以编译和求值表达式。表达式是通常返回布尔值的单行代码，可被执行表达式的代码在 `if` 语句中使用。一个简单的表达式示例是 `1 + 2`。你也可以使用更复杂的表达式，例如 `someArray[3].someMethod('bar')`。

该组件提供了两种处理表达式的方式：

* **求值（evaluation）**：表达式在不编译为 PHP 的情况下被求值；
* **编译（compile）**：表达式被编译为 PHP，因此可以被缓存和求值。

该组件的主类是 `Symfony\Component\ExpressionLanguage\ExpressionLanguage`：

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

$expressionLanguage = new ExpressionLanguage();

var_dump($expressionLanguage->evaluate('1 + 2')); // displays 3

var_dump($expressionLanguage->compile('1 + 2')); // displays (1 + 2)
```

> **提示：**
> 参阅 [/reference/formats/expression_language](expression_language.md) 以了解 ExpressionLanguage 组件的语法。

#### 空值合并运算符

> **注意：**
> 此内容已移至 ExpressionLanguage 语法参考页面的空值合并运算符部分。

#### 解析和校验表达式

ExpressionLanguage 组件提供了解析和校验表达式的方法。`Symfony\Component\ExpressionLanguage\ExpressionLanguage::parse` 方法返回一个 `Symfony\Component\ExpressionLanguage\ParsedExpression` 实例，可用于检查和操作表达式。而 `Symfony\Component\ExpressionLanguage\ExpressionLanguage::lint` 方法则在表达式无效时抛出 `Symfony\Component\ExpressionLanguage\SyntaxError`：

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

$expressionLanguage = new ExpressionLanguage();

var_dump($expressionLanguage->parse('1 + 2', []));
// displays the AST nodes of the expression which can be
// inspected and manipulated

$expressionLanguage->lint('1 + 2', []); // doesn't throw anything

$expressionLanguage->lint('1 + a', []);
// throws a SyntaxError exception:
// "Variable "a" is not valid around position 5 for expression `1 + a`."
```

这些方法的行为可以通过 `Symfony\Component\ExpressionLanguage\Parser` 类中定义的一些标志进行配置：

* `IGNORE_UNKNOWN_VARIABLES`：如果表达式中未定义某个变量，不抛出异常；
* `IGNORE_UNKNOWN_FUNCTIONS`：如果表达式中未定义某个函数，不抛出异常。

以下是使用这些标志的方式：

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;
use Symfony\Component\ExpressionLanguage\Parser;

$expressionLanguage = new ExpressionLanguage();

// does not throw a SyntaxError because the unknown variables and functions are ignored
$expressionLanguage->lint('unknown_var + unknown_function()', [], Parser::IGNORE_UNKNOWN_VARIABLES | Parser::IGNORE_UNKNOWN_FUNCTIONS);
```

## 传入变量

你也可以向表达式传入变量，变量可以是任何有效的 PHP 类型（包括对象）：

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

$expressionLanguage = new ExpressionLanguage();

class Apple
{
    public string $variety;
}

$apple = new Apple();
$apple->variety = 'Honeycrisp';

var_dump($expressionLanguage->evaluate(
    'fruit.variety',
    [
        'fruit' => $apple,
    ]
)); // displays "Honeycrisp"
```

在 Symfony 应用程序中使用此组件时，Symfony 会自动注入某些对象和变量，以便你在表达式中使用它们（例如请求、当前用户等）：

* 安全表达式中可用的变量；
* 服务容器表达式中可用的变量；
* 路由表达式中可用的变量。

## 缓存

ExpressionLanguage 组件提供了一个 `Symfony\Component\ExpressionLanguage\ExpressionLanguage::compile` 方法，可以将表达式缓存为纯 PHP。但在内部，该组件也会缓存已解析的表达式，以便重复出现的表达式可以更快地被编译/求值。

### 工作流程

`Symfony\Component\ExpressionLanguage\ExpressionLanguage::evaluate` 和 `compile()` 在提供返回值之前都需要做一些工作。对于 `evaluate()`，这个开销甚至更大。

这两个方法都需要对表达式进行词法分析和解析。这由 `Symfony\Component\ExpressionLanguage\ExpressionLanguage::parse` 方法完成。它返回一个 `Symfony\Component\ExpressionLanguage\ParsedExpression`。现在，`compile()` 方法只返回此对象的字符串转换。`evaluate()` 方法需要遍历"节点"（保存在 `ParsedExpression` 中的表达式片段）并动态求值。

为了节省时间，`ExpressionLanguage` 缓存 `ParsedExpression`，以便它可以跳过重复表达式的词法分析和解析步骤。缓存由 PSR-6 [CacheItemPoolInterface] 实例完成（默认使用 `Symfony\Component\Cache\Adapter\ArrayAdapter`）。你可以通过创建自定义缓存池或使用可用的其中一个来自定义，并通过构造函数注入：

```php
use Symfony\Component\Cache\Adapter\RedisAdapter;
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

$cache = new RedisAdapter(...);
$expressionLanguage = new ExpressionLanguage($cache);
```

> **参见：**
> 有关可用缓存适配器的更多信息，请参阅 [/components/cache](cache.md) 文档。

### 使用已解析和序列化的表达式

`evaluate()` 和 `compile()` 都可以处理 `ParsedExpression` 和 `SerializedParsedExpression`：

```php
// ...

// the parse() method returns a ParsedExpression
$expression = $expressionLanguage->parse('1 + 4', []);

var_dump($expressionLanguage->evaluate($expression)); // prints 5
```

```php
use Symfony\Component\ExpressionLanguage\SerializedParsedExpression;
// ...

$expression = new SerializedParsedExpression(
    '1 + 4',
    serialize($expressionLanguage->parse('1 + 4', [])->getNodes())
);

var_dump($expressionLanguage->evaluate($expression)); // prints 5
```

## AST 转储和编辑

使用 ExpressionLanguage 组件创建的表达式很难被操作或检查，因为表达式是纯字符串。更好的方法是将这些表达式转换为 AST。在计算机科学中，[AST]（*抽象语法树*）是*"用编程语言编写的源代码结构的树形表示"*。在 Symfony 中，ExpressionLanguage AST 是一组包含表示给定表达式的 PHP 类的节点。

### 转储 AST

在解析任何表达式后调用 `Symfony\Component\ExpressionLanguage\ExpressionLanguage::getNodes` 方法以获取其 AST：

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

$ast = new ExpressionLanguage()
    ->parse('1 + 2', [])
    ->getNodes()
;

// dump the AST nodes for inspection
var_dump($ast);

// dump the AST nodes as a string representation
$astAsString = $ast->dump();
```

### 操作 AST

AST 的节点也可以转储为 PHP 节点数组以允许操作它们。调用 `Symfony\Component\ExpressionLanguage\ExpressionLanguage::toArray` 方法将 AST 转换为数组：

```php
// ...

$astAsArray = new ExpressionLanguage()
    ->parse('1 + 2', [])
    ->getNodes()
    ->toArray()
;
```

## 扩展 ExpressionLanguage

ExpressionLanguage 可以通过添加自定义函数来扩展。例如，在 Symfony 框架中，安全性有自定义函数来检查用户角色。

> **注意：**
> 如果你想了解如何在表达式中使用函数，请阅读"component-expression-functions"。

### 注册函数

函数在每个特定的 `ExpressionLanguage` 实例上注册。这意味着函数可以在该实例执行的任何表达式中使用。

要注册函数，请使用 `Symfony\Component\ExpressionLanguage\ExpressionLanguage::register`。此方法有 3 个参数：

* **名称（name）** - 表达式中函数的名称；
* **编译器（compiler）** - 使用该函数编译表达式时执行的函数；
* **求值器（evaluator）** - 求值表达式时执行的函数。

示例：

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

$expressionLanguage = new ExpressionLanguage();
$expressionLanguage->register('lowercase', function ($str): string {
    return sprintf('(is_string(%1$s) ? strtolower(%1$s) : %1$s)', $str);
}, function ($arguments, $str): string {
    if (!is_string($str)) {
        return $str;
    }

    return strtolower($str);
});

var_dump($expressionLanguage->evaluate('lowercase("HELLO")'));
// this will print: hello
```

除了自定义函数参数外，**求值器**还将 `arguments` 变量作为其第一个参数传入，该变量等于 `evaluate()` 的第二个参数（例如，求值表达式时的"值"）。

### 使用表达式提供者

在库中使用 `ExpressionLanguage` 类时，通常需要添加自定义函数。为此，你可以通过创建一个实现 `Symfony\Component\ExpressionLanguage\ExpressionFunctionProviderInterface` 的类来创建新的表达式提供者。

此接口需要一个方法：`Symfony\Component\ExpressionLanguage\ExpressionFunctionProviderInterface::getFunctions`，它返回要注册的表达式函数（`Symfony\Component\ExpressionLanguage\ExpressionFunction` 的实例）数组：

```php
use Symfony\Component\ExpressionLanguage\ExpressionFunction;
use Symfony\Component\ExpressionLanguage\ExpressionFunctionProviderInterface;

class StringExpressionLanguageProvider implements ExpressionFunctionProviderInterface
{
    public function getFunctions(): array
    {
        return [
            new ExpressionFunction('lowercase', function ($str): string {
                return sprintf('(is_string(%1$s) ? strtolower(%1$s) : %1$s)', $str);
            }, function ($arguments, $str): string {
                if (!is_string($str)) {
                    return $str;
                }

                return strtolower($str);
            }),
        ];
    }
}
```

> **提示：**
> 使用 `Symfony\Component\ExpressionLanguage\ExpressionFunction::fromPhp` 静态方法从 PHP 函数创建表达式函数：
>
> ```php
> ExpressionFunction::fromPhp('strtoupper');
> ```
>
> 支持命名空间函数，但需要第二个参数来定义表达式的名称：
>
> ```php
> ExpressionFunction::fromPhp('My\strtoupper', 'my_strtoupper');
> ```

你可以使用 `Symfony\Component\ExpressionLanguage\ExpressionLanguage::registerProvider` 注册提供者，或使用构造函数的第二个参数：

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

// using the constructor
$expressionLanguage = new ExpressionLanguage(null, [
    new StringExpressionLanguageProvider(),
    // ...
]);

// using registerProvider()
$expressionLanguage->registerProvider(new StringExpressionLanguageProvider());
```

> **提示：**
> 建议在你的库中创建自己的 `ExpressionLanguage` 类。现在你可以通过覆盖构造函数来添加扩展：
>
> ```php
> use Psr\Cache\CacheItemPoolInterface;
> use Symfony\Component\ExpressionLanguage\ExpressionLanguage as BaseExpressionLanguage;
>
> class ExpressionLanguage extends BaseExpressionLanguage
> {
>     public function __construct(?CacheItemPoolInterface $cache = null, array $providers = [])
>     {
>         // prepends the default provider to let users override it
>         array_unshift($providers, new StringExpressionLanguageProvider());
>
>         parent::__construct($cache, $providers);
>     }
> }
> ```

[AST]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
[CacheItemPoolInterface]: https://github.com/php-fig/cache/blob/master/src/CacheItemPoolInterface.php
