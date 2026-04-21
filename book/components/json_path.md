# JsonPath 组件

JsonPath 组件允许你查询和提取 JSON 结构中的数据。它实现了 [RFC 9535 – JSONPath] 标准，允许你导航复杂的 JSON 数据。

类似于允许你使用 XPath 导航和查询 HTML 或 XML 文档的 DomCrawler 组件，JsonPath 组件通过 JSONPath 表达式为遍历和搜索 JSON 结构提供了同等的便利。该组件还提供了数据提取的抽象层。

## 安装

你可以使用 Composer 在项目中安装该组件：

```terminal
$ composer require symfony/json-path
```

## 使用

要开始查询 JSON 文档，首先从 JSON 字符串创建一个 `Symfony\Component\JsonPath\JsonCrawler` 对象。以下示例使用这个示例"书店"JSON 数据：

```php
use Symfony\Component\JsonPath\JsonCrawler;

$json = <<<'JSON'
{
    "store": {
        "book": [
            {
                "category": "reference",
                "author": "Nigel Rees",
                "title": "Sayings of the Century",
                "price": 8.95
            },
            {
                "category": "fiction",
                "author": "Evelyn Waugh",
                "title": "Sword of Honour",
                "price": 12.99
            },
            {
                "category": "fiction",
                "author": "Herman Melville",
                "title": "Moby Dick",
                "isbn": "0-553-21311-3",
                "price": 8.99
            },
            {
                "category": "fiction",
                "author": "John Ronald Reuel Tolkien",
                "title": "The Lord of the Rings",
                "isbn": "0-395-19395-8",
                "price": 22.99
            }
        ],
        "bicycle": {
            "color": "red",
            "price": 399
        }
    }
}
JSON;

$crawler = new JsonCrawler($json);
```

获得爬取器实例后，使用其 `Symfony\Component\JsonPath\JsonCrawler::find` 方法开始查询数据。该方法返回匹配值的数组。

## 使用表达式查询

查询 JSON 的主要方式是将 JSONPath 表达式字符串传递给 `Symfony\Component\JsonPath\JsonCrawler::find` 方法。

### 访问特定属性

对对象键使用点表示法，对数组索引使用方括号。文档的根用 `$` 表示：

```php
// get the title of the first book in the store
$titles = $crawler->find('$.store.book[0].title');

// $titles is ['Sayings of the Century']
```

点表示法是默认的，但 JSONPath 为其不适用的情况提供了其他语法。当键包含空格或特殊字符时，使用方括号表示法（`['...']`）：

```php
// this is equivalent to the previous example
$titles = $crawler->find('$["store"]["book"][0]["title"]');

// this expression requires brackets because some keys use dots or spaces
$titles = $crawler->find('$["store"]["book collection"][0]["title.original"]');

// you can combine both notations
$titles = $crawler->find('$["store"].book[0].title');
```

### 使用后代运算符搜索

后代运算符（`..`）递归搜索给定键，允许你在不指定完整路径的情况下查找值：

```php
// get all authors from anywhere in the document
$authors = $crawler->find('$..author');

// $authors is ['Nigel Rees', 'Evelyn Waugh', 'Herman Melville', 'John Ronald Reuel Tolkien']
```

### 过滤结果

JSONPath 包括一个过滤语法（`?(expression)`），用于根据条件选择项目。过滤器中的当前项目由 `@` 引用：

```php
// get all books with a price less than 10
$cheapBooks = $crawler->find('$.store.book[?(@.price < 10)]');
```

## 以编程方式构建查询

对于更动态或复杂的查询构建，请使用 `Symfony\Component\JsonPath\JsonPath` 类提供的流式 API。这允许你逐步构建查询对象。然后可以将 `JsonPath` 对象传递给爬取器的 `Symfony\Component\JsonPath\JsonCrawler::find` 方法。

以编程方式构建的主要优势是它会自动处理键和值的转义，防止语法错误：

```php
use Symfony\Component\JsonPath\JsonPath;

$path = new JsonPath()
    ->key('store') // selects the 'store' key
    ->key('book')  // then the 'book' key
    ->index(1);    // then the second item (indexes start at 0)

// the created $path object is equivalent to the string '$["store"]["book"][1]'
$book = $crawler->find($path);

// $book contains the book object for "Sword of Honour"
```

`Symfony\Component\JsonPath\JsonPath` 类提供了几种方法来构建查询：

* `Symfony\Component\JsonPath\JsonPath::key`
  添加键选择器。键名将被正确转义：

  ```php
  // creates the path '$["key\"with\"quotes"]'
  $path = new JsonPath()->key('key"with"quotes');
  ```

* `Symfony\Component\JsonPath\JsonPath::deepScan`
  添加后代运算符 `..` 以从路径中的当前点执行递归搜索：

  ```php
  // get all prices in the store: '$["store"]..["price"]'
  $path = new JsonPath()->key('store')->deepScan()->key('price');
  ```

* `Symfony\Component\JsonPath\JsonPath::all`
  添加通配符运算符 `[*]` 以选择数组或对象中的所有项目：

  ```php
  // creates the path '$["store"]["book"][*]'
  $path = new JsonPath()->key('store')->key('book')->all();
  ```

* `Symfony\Component\JsonPath\JsonPath::index`
  添加数组索引选择器。索引号从 `0` 开始。

* `Symfony\Component\JsonPath\JsonPath::first` /
  `Symfony\Component\JsonPath\JsonPath::last`
  分别是 `index(0)` 和 `index(-1)` 的快捷方式：

  ```php
  // get the last book: '$["store"]["book"][-1]'
  $path = new JsonPath()->key('store')->key('book')->last();
  ```

* `Symfony\Component\JsonPath\JsonPath::slice`
  添加数组切片选择器 `[start:end:step]`：

  ```php
  // get books from index 1 up to (but not including) index 3
  // creates the path '$["store"]["book"][1:3]'
  $path = new JsonPath()->key('store')->key('book')->slice(1, 3);

  // get every second book from the first four books
  // creates the path '$["store"]["book"][0:4:2]'
  $path = new JsonPath()->key('store')->key('book')->slice(0, 4, 2);
  ```

* `Symfony\Component\JsonPath\JsonPath::filter`
  添加过滤器表达式。表达式字符串是放在 `?()` 语法内的部分：

  ```php
  // get expensive books: '$["store"]["book"][?(@.price > 20)]'
  $path = new JsonPath()
      ->key('store')
      ->key('book')
      ->filter('@.price > 20');
  ```

## 高级查询

有关通配符和过滤器中函数等高级运算符的完整概述，请参阅上面的"使用表达式查询"部分。所有这些功能都受支持，并且在适当的情况下可以与以编程方式构建的查询结合使用（例如，在 `filter()` 表达式内部）。

## 使用 JSON 断言进行测试

该组件提供了一组 PHPUnit 断言，使测试 JSON 数据更加方便。在测试类中使用 `Symfony\Component\JsonPath\Test\JsonPathAssertionsTrait`：

```php
use PHPUnit\Framework\TestCase;
use Symfony\Component\JsonPath\Test\JsonPathAssertionsTrait;

class MyTest extends TestCase
{
    use JsonPathAssertionsTrait;

    public function testSomething(): void
    {
        $json = '{"books": [{"title": "A"}, {"title": "B"}]}';

        self::assertJsonPathCount(2, '$.books[*]', $json);
    }
}
```

该 trait 提供以下断言方法：

* `Symfony\Component\JsonPath\Test\JsonPathAssertionsTrait::assertJsonPathCount`
  断言 JSONPath 表达式找到的元素数量与预期数量匹配：

  ```php
  $json = '{"a": [1, 2, 3]}';
  self::assertJsonPathCount(3, '$.a[*]', $json);
  ```

* `Symfony\Component\JsonPath\Test\JsonPathAssertionsTrait::assertJsonPathEquals`
  断言 JSONPath 表达式的结果等于预期值。比较使用 `==`（类型强制转换）而非 `===`：

  ```php
  $json = '{"a": [1, 2, 3]}';

  // passes because "1" == 1
  self::assertJsonPathEquals(['1'], '$.a[0]', $json);
  ```

* `Symfony\Component\JsonPath\Test\JsonPathAssertionsTrait::assertJsonPathNotEquals`
  断言 JSONPath 表达式的结果不等于预期值。比较使用 `!=`（类型强制转换）而非 `!==`：

  ```php
  $json = '{"a": [1, 2, 3]}';
  self::assertJsonPathNotEquals([42], '$.a[0]', $json);
  ```

* `Symfony\Component\JsonPath\Test\JsonPathAssertionsTrait::assertJsonPathSame`
  断言 JSONPath 表达式的结果与预期值完全相同（`===`）。这是严格比较，不执行类型强制转换：

  ```php
  $json = '{"a": [1, 2, 3]}';

  // fails because "1" !== 1
  // self::assertJsonPathSame(['1'], '$.a[0]', $json);

  self::assertJsonPathSame([1], '$.a[0]', $json);
  ```

* `Symfony\Component\JsonPath\Test\JsonPathAssertionsTrait::assertJsonPathNotSame`
  断言 JSONPath 表达式的结果与预期值不完全相同（`!==`）：

  ```php
  $json = '{"a": [1, 2, 3]}';
  self::assertJsonPathNotSame(['1'], '$.a[0]', $json);
  ```

* `Symfony\Component\JsonPath\Test\JsonPathAssertionsTrait::assertJsonPathContains`
  断言在 JSONPath 表达式的结果数组中找到了给定值：

  ```php
  $json = '{"tags": ["php", "symfony", "json"]}';
  self::assertJsonPathContains('symfony', '$.tags[*]', $json);
  ```

* `Symfony\Component\JsonPath\Test\JsonPathAssertionsTrait::assertJsonPathNotContains`
  断言在 JSONPath 表达式的结果数组中**未**找到给定值：

  ```php
  $json = '{"tags": ["php", "symfony", "json"]}';
  self::assertJsonPathNotContains('java', '$.tags[*]', $json);
  ```

## 错误处理

该组件针对无效输入或查询抛出特定异常：

* `Symfony\Component\JsonPath\Exception\InvalidArgumentException`：如果 `JsonCrawler` 构造函数的输入不是有效的 JSON 字符串，则抛出此异常；
* `Symfony\Component\JsonPath\Exception\InvalidJsonStringInputException`：如果 JSON 字符串格式不正确（例如语法错误），在 `find()` 调用期间抛出此异常；
* `Symfony\Component\JsonPath\Exception\JsonCrawlerException`：针对 JsonPath 表达式本身的错误（例如使用未知函数）抛出此异常。

错误处理示例：

```php
use Symfony\Component\JsonPath\Exception\InvalidJsonStringInputException;
use Symfony\Component\JsonPath\Exception\JsonCrawlerException;

try {
    // the following line contains malformed JSON
    $crawler = new JsonCrawler('{"store": }');
    $crawler->find('$..*');
} catch (InvalidJsonStringInputException $e) {
    // ... handle error
}

try {
    // the following line contains an invalid query
    $crawler->find('$.store.book[?unknown_function(@.price)]');
} catch (JsonCrawlerException $e) {
    // ... handle error
}
```

[RFC 9535 – JSONPath]: https://datatracker.ietf.org/doc/html/rfc9535
