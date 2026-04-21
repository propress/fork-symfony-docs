# Workflow 组件

Workflow 组件提供了用于管理工作流或有限状态机的工具。

## 安装

```terminal
$ composer require symfony/workflow
```

## 创建工作流

Workflow 组件为你提供了一种面向对象的方式来定义对象所经历的流程或生命周期。流程中的每个步骤或阶段称为一个*场所（place）*。你还需要定义*转换（transitions）*，描述从一个场所到另一个场所的动作。

![工作流状态图示例，展示转换和场所。](/_images/components/workflow/states_transitions.png)

一组场所和转换构成一个**定义（definition）**。工作流需要一个 `Definition` 以及一种将状态写入对象的方式（即 `Symfony\Component\Workflow\MarkingStore\MarkingStoreInterface` 的实例）。

考虑以下博客文章的示例。一篇文章可以有多个预定义状态之一（`draft`、`reviewed`、`rejected`、`published`）。在工作流中，这些状态称为**场所**。你可以这样定义工作流：

```php
use Symfony\Component\Workflow\DefinitionBuilder;
use Symfony\Component\Workflow\MarkingStore\MethodMarkingStore;
use Symfony\Component\Workflow\Transition;
use Symfony\Component\Workflow\Workflow;

$definitionBuilder = new DefinitionBuilder();
$definition = $definitionBuilder->addPlaces(['draft', 'reviewed', 'rejected', 'published'])
    // 转换由唯一名称、起始场所和目标场所定义
    ->addTransition(new Transition('to_review', 'draft', 'reviewed'))
    ->addTransition(new Transition('publish', 'reviewed', 'published'))
    ->addTransition(new Transition('reject', 'reviewed', 'rejected'))
    ->build()
;

$singleState = true; // 如果主体在某一时刻只能处于一个状态，则为 true
$property = 'currentState'; // 存储状态的主体属性名称
$marking = new MethodMarkingStore($singleState, $property);
$workflow = new Workflow($definition, $marking);
```

`Workflow` 现在可以帮助你根据博客文章所处的*场所*（状态）来决定允许哪些*转换*（动作）。这将使你的领域逻辑集中在一处，而不是分散在整个应用程序中。

## 使用

以下是使用上面定义的工作流的示例：

```php
// ...
// 假设 $blogPost 默认处于 "draft" 场所
$blogPost = new BlogPost();

$workflow->can($blogPost, 'publish'); // False
$workflow->can($blogPost, 'to_review'); // True

$workflow->apply($blogPost, 'to_review'); // $blogPost 现在处于 "reviewed" 场所

$workflow->can($blogPost, 'publish'); // True
$workflow->getEnabledTransitions($blogPost); // $blogPost 可以执行 "publish" 或 "reject" 转换
```

## 初始化

如果你的对象的标记属性为 `null`，并且你想用配置中的 `initial_marking` 来设置它，可以调用 `getMarking()` 方法来初始化对象属性：

```php
// ...
$blogPost = new BlogPost();

// 初始化工作流
$workflow->getMarking($blogPost);
```

## 了解更多

请阅读更多关于在 Symfony 应用程序中使用 [Workflow 组件](workflow.md)的内容。
