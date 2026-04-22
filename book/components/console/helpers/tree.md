# Tree Helper

Tree Helper 允许您在控制台中构建和显示树结构。它通常用于呈现目录层次结构，但您也可以使用它来呈现任何树状内容，例如组织结构图、产品类别树、分类法等。

## 渲染树

`Symfony\Component\Console\Helper\TreeHelper::createTree` 方法从数组创建树结构，并返回可以在控制台中呈现的 `Symfony\Component\Console\Helper\Tree` 对象。

### 从数组渲染树

您可以通过在控制台命令内将数组传递给 `Symfony\Component\Console\Helper\TreeHelper::createTree` 方法来从数组构建树：

```php
namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Helper\TreeHelper;
use Symfony\Component\Console\Helper\TreeNode;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(name: 'app:my-command', description: '...')]
class MyCommand
{
    // ...

    public function __invoke(SymfonyStyle $io): int
    {
        $node = TreeNode::fromValues([
            'config/',
            'public/',
            'src/',
            'templates/',
            'tests/',
        ]);

        $tree = TreeHelper::createTree($io, $node);
        $tree->render();

        // ...
    }
}
```

此示例将输出以下内容：

```bash
├── config/
├── public/
├── src/
├── templates/
└── tests/
```

给定内容可以在多维数组中定义：

```php
$tree = TreeHelper::createTree($io, null, [
    'src' =>  [
        'Command',
        'Controller' => [
            'DefaultController.php',
        ],
        'Kernel.php',
    ],
    'templates' => [
        'base.html.twig',
    ],
]);

$tree->render();
```

上面的代码将输出以下树：

```bash
├── src
│   ├── Command
│   ├── Controller
│   │   └── DefaultController.php
│   └── Kernel.php
└── templates
    └── base.html.twig
```

### 以编程方式构建树

如果您事先不知道树元素，可以通过创建 `Symfony\Component\Console\Helper\Tree` 类的新实例并向其添加节点来以编程方式构建树：

```php
use Symfony\Component\Console\Helper\TreeHelper;
use Symfony\Component\Console\Helper\TreeNode;

$root = new TreeNode('my-project/');
// 您可以直接传递字符串或创建 TreeNode 对象
$root->addChild('src/');
$root->addChild(new TreeNode('templates/'));

// 通过向其他节点添加子节点来创建嵌套结构
$testsNode = new TreeNode('tests/');
$functionalTestsNode = new TreeNode('Functional/');
$testsNode->addChild($functionalTestsNode);
$root->addChild($testsNode);

$tree = TreeHelper::createTree($io, $root);
$tree->render();
```

此示例输出：

```bash
my-project/
├── src/
├── templates/
└── tests/
    └── Functional/
```

如果您愿意，可以以编程方式构建元素数组，然后像这样创建和呈现树：

```php
$tree = TreeHelper::createTree($io, null, $array);
$tree->render();
```

您还可以从数组构建树的一部分，然后添加其他节点：

```php
$node = TreeNode::fromValues($array);
$node->addChild('templates');
// ...
$tree = TreeHelper::createTree($io, $node);
$tree->render();
```

## 自定义树样式

### 内置树样式

树辅助工具提供了几种内置样式，您可以使用它们来自定义树的输出。

**默认**：

```php
TreeHelper::createTree($io, $node, [], TreeStyle::default());
```

**Box**：

```php
TreeHelper::createTree($io, $node, [], TreeStyle::box());
```

**Double box**：

```php
TreeHelper::createTree($io, $node, [], TreeStyle::doubleBox());
```

**Compact**：

```php
TreeHelper::createTree($io, $node, [], TreeStyle::compact());
```

**Light**：

```php
TreeHelper::createTree($io, $node, [], TreeStyle::light());
```

**Minimal**：

```php
TreeHelper::createTree($io, $node, [], TreeStyle::minimal());
```

**Rounded**：

```php
TreeHelper::createTree($io, $node, [], TreeStyle::rounded());
```

### 制作自定义树样式

您可以通过将字符传递给 `Symfony\Component\Console\Helper\TreeStyle` 类的构造函数来创建自己的树样式：

```php
use Symfony\Component\Console\Helper\TreeHelper;
use Symfony\Component\Console\Helper\TreeStyle;

$customStyle = new TreeStyle('🟣 ', '🟠 ', '🔵 ', '🟢 ', '🔴 ', '🟡 ');

// 将自定义样式传递给 createTree 方法

$tree = TreeHelper::createTree($io, null, [
    'src' =>  [
        'Command',
        'Controller' => [
            'DefaultController.php',
        ],
        'Kernel.php',
    ],
    'templates' => [
        'base.html.twig',
    ],
], $customStyle);

$tree->render();
```
