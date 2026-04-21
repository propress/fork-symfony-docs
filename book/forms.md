# 表单

> 更喜欢视频教程？请查看 [Symfony 表单视频系列](https://symfonycasts.com/screencast/symfony-forms)。

创建和处理 HTML 表单是困难且重复的工作。你需要处理渲染 HTML 表单字段、验证提交的数据、将表单数据映射到对象等许多工作。Symfony 包含一个强大的表单功能，为真正复杂的场景提供所有这些功能以及更多功能。

---

## 安装

在使用 [Symfony Flex](setup.md#symfony-flex) 的应用中，在使用前运行此命令安装表单功能：

```terminal
$ composer require symfony/form
```

---

## 理解表单的工作原理

在深入代码之前，了解 Symfony 表单背后的思维模型很有帮助。将表单视为 PHP 对象（或数组）与 HTML 表单之间的**双向映射层**。

此映射工作方向：

1. **对象到 HTML**：渲染表单时，Symfony 从你的对象中读取数据，并将其转换为用户可以编辑的 HTML 字段；
2. **HTML 到对象**：处理提交时，Symfony 从 HTTP 请求中获取原始值（通常是字符串），并将其转换回对象上适当的 PHP 类型。

此流程是表单组件的核心。从简单的文本字段到复杂的嵌套集合，一切都遵循相同的模式。

### 数据转换生命周期 {#form-data-lifecycle}

表单中的数据经过三种表示，通常称为**数据层**：

**模型数据（Model Data）**
应用程序使用的格式的数据。例如，`DateTime` 对象、Doctrine 实体或自定义值对象。这是你传递给 `createForm()` 的内容，也是成功提交后通过 `getData()` 获得的内容。

**规范化数据（Normalized Data）**
对模型数据进行规范化的中间表示。对于大多数字段类型，这与模型数据相同。但对于某些类型，则不同。例如，`DateType` 规范化为带有 `year`、`month` 和 `day` 键的数组。

**视图数据（View Data）**
用于填充 HTML 表单字段并从用户提交接收的格式。在大多数情况下，这是基于字符串的（或字符串数组），因为浏览器提交文本。某些字段可能使用其他表示，或出于安全原因保持为空（例如，文件输入）。

**表单渲染流程**

1. 从你的对象开始使用模型数据；
2. 模型转换器将其转换为规范化数据；
3. 视图转换器将其转换为视图数据（通常是字符串）；
4. Symfony 渲染相应的 HTML 小部件。

**表单提交流程**

1. Symfony 从 HTTP 请求中读取原始值（通常是字符串）；
2. 视图转换器将数据反向转换为规范化数据；
3. 模型转换器将数据反向转换为模型数据；
4. 数据被写回基础对象或数组。

对于配置为渲染为三个 `<select>` 元素的 `DateType` 字段：

- **模型数据**：`DateTime` 对象；
- **规范化数据**：类似 `['year' => 2026, 'month' => 10, 'day' => 18]` 的数组（值是整数）；
- **视图数据**：类似 `['year' => '2026', 'month' => '10', 'day' => '18']` 的数组（值是字符串，由浏览器提交）。

大多数时候你不需要考虑这些层。当调试字段为何不正确显示或提交时，或创建自定义[数据转换器](form/data_transformers.md)时，它们就变得相关了。

---

## 用法

使用 Symfony 表单时的推荐工作流如下：

1. 在 Symfony 控制器中或使用专用表单类**构建表单**；
2. 在模板中**渲染表单**，以便用户可以编辑和提交它；
3. **处理表单**以验证提交的数据、将其转换为 PHP 数据，并对其执行某些操作（例如将其持久化到数据库中）。

这些步骤中的每一个都在接下来的部分中详细说明。为了使示例更易于理解，所有示例都假设你正在构建一个显示"任务"的小型待办事项列表应用。

用户使用 Symfony 表单创建和编辑任务。每个任务都是以下 `Task` 类的实例：

```php
// src/Entity/Task.php
namespace App\Entity;

class Task
{
    protected string $task;

    protected ?\DateTimeInterface $dueDate;

    public function getTask(): string
    {
        return $this->task;
    }

    public function setTask(string $task): void
    {
        $this->task = $task;
    }

    public function getDueDate(): ?\DateTimeInterface
    {
        return $this->dueDate;
    }

    public function setDueDate(?\DateTimeInterface $dueDate): void
    {
        $this->dueDate = $dueDate;
    }
}
```

这个类是一个"普通 PHP 对象"，因为到目前为止，它与 Symfony 或任何其他库无关。它是一个直接解决*你的*应用中某个问题的普通 PHP 对象（即在应用中表示任务的需求）。但你也可以用同样的方式编辑 [Doctrine 实体](doctrine.md)。

### 表单类型 {#form-types}

在创建第一个 Symfony 表单之前，了解"表单类型"的概念很重要。在其他项目中，通常会区分"表单"和"表单字段"。在 Symfony 中，所有这些都是"表单类型"：

- 单个 `<input type="text">` 表单字段是一个"表单类型"（例如 `TextType`）；
- 用于输入邮政地址的几个 HTML 字段的组合是一个"表单类型"（例如 `PostalAddressType`）；
- 带有多个字段用于编辑用户配置文件的整个 `<form>` 是一个"表单类型"（例如 `UserProfileType`）。

这种统一的概念使表单组件更加**灵活**。你可以由更简单的类型组合复杂的表单，在表单中嵌入表单，并在整个应用中复用相同的类型定义。

**表单类型层次结构**

每种表单类型都有一个父类型。父类型决定了你的类型继承的基本行为、选项和渲染。以下是简化视图：

```text
FormType        （所有类型的根父类型）
├─ TextType    （渲染文本输入）
│  ├─ EmailType
│  ├─ PasswordType
│  ├─ ...
│  └─ UrlType
├─ ChoiceType  （渲染选择框、单选框或复选框）
│  ├─ CountryType
│  ├─ EntityType
│  └─ ...
├─ DateType    （渲染日期输入的单个或多个字段）
│  └─ ...
└─ ...
```

Symfony 提供了几十种[表单类型](reference/forms/types.md)，你也可以[创建自己的表单类型](form/create_custom_field_type.md)。

> **提示**
>
> 你可以使用 `debug:form` 列出应用中所有可用的类型、类型扩展和类型猜测器：
>
> ```terminal
> $ php bin/console debug:form
>
> # 传递表单类型 FQCN 只显示该类型、其父类型和扩展的选项
> # 对于内置类型，你可以传递简短的类名而不是 FQCN
> $ php bin/console debug:form BirthdayType
>
> # 也传递选项名称以仅显示该选项的完整定义
> $ php bin/console debug:form BirthdayType label_attr
> ```

---

## 构建表单

Symfony 提供了一个"表单构建器"对象，允许你使用流畅的接口描述表单字段。之后，此构建器创建用于渲染和处理内容的实际表单对象。

### 在控制器中创建表单 {#creating-forms-in-controllers}

如果你的控制器扩展自 [AbstractController](controller.md#基础控制器类和服务)，请使用 `createFormBuilder()` 辅助方法：

```php
// src/Controller/TaskController.php
namespace App\Controller;

use App\Entity\Task;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Form\Extension\Core\Type\DateType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

class TaskController extends AbstractController
{
    public function new(Request $request): Response
    {
        // 创建一个 task 对象并为此示例初始化一些数据
        $task = new Task();
        $task->setTask('Write a blog post');
        $task->setDueDate(new \DateTimeImmutable('tomorrow'));

        $form = $this->createFormBuilder($task)
            ->add('task', TextType::class)
            ->add('dueDate', DateType::class)
            ->add('save', SubmitType::class, ['label' => 'Create Task'])
            ->getForm();

        // ...
    }
}
```

在此示例中，你向表单添加了两个字段——`task` 和 `dueDate`——对应于 `Task` 类的 `task` 和 `dueDate` 属性。你还为每个字段分配了一个[表单类型](#form-types)（例如 `TextType` 和 `DateType`），由其完全限定的类名表示。最后，你添加了一个带有自定义标签的提交按钮，用于将表单提交到服务器。

### 创建表单类 {#creating-forms-in-classes}

Symfony 建议在控制器中尽量少放逻辑。这就是为什么最好将复杂表单移到专用类，而不是在控制器动作中定义它们。此外，在类中定义的表单可以在多个动作和服务中复用。

表单类是实现 `FormTypeInterface` 的[表单类型](#form-types)。但是，更好的做法是从 `AbstractType` 扩展，它已经实现了接口并提供了一些实用工具：

```php
// src/Form/Type/TaskType.php
namespace App\Form\Type;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\DateType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;

class TaskType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('task', TextType::class)
            ->add('dueDate', DateType::class)
            ->add('save', SubmitType::class)
        ;
    }
}
```

> **提示**
>
> 在你的项目中安装 [MakerBundle](https://symfony.com/doc/current/bundles/SymfonyMakerBundle/index.html)，使用 `make:form` 和 `make:registration-form` 命令生成表单类。

在扩展自 [AbstractController](controller.md#基础控制器类和服务) 的控制器中，使用 `createForm()` 辅助方法：

```php
// src/Controller/TaskController.php
namespace App\Controller;

use App\Form\Type\TaskType;
// ...

class TaskController extends AbstractController
{
    public function new(): Response
    {
        // 创建一个 task 对象并为此示例初始化一些数据
        $task = new Task();
        $task->setTask('Write a blog post');
        $task->setDueDate(new \DateTimeImmutable('tomorrow'));

        $form = $this->createForm(TaskType::class, $task);

        // ...
    }
}
```

每个表单都需要知道保存基础数据的类的名称（例如 `App\Entity\Task`）。通常，这是根据传递给 `createForm()` 第二个参数的对象猜测的（即 `$task`）。

因此，虽然不总是必要的，但通常最好通过在表单类型类中添加以下内容来明确指定 `data_class` 选项：

```php
// src/Form/Type/TaskType.php
namespace App\Form\Type;

use App\Entity\Task;
use Symfony\Component\OptionsResolver\OptionsResolver;
// ...

class TaskType extends AbstractType
{
    // ...

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'data_class' => Task::class,
        ]);
    }
}
```

### 将字段映射到对象属性 {#form-property-path}

默认情况下，名为 `dueDate` 的表单字段在你的对象上读取和写入 `dueDate` 属性。这使用 [PropertyAccess 组件](components/property_access.md)，它可以与公共属性和常见的访问器名称（`get*()`、`is*()`、`has*()`、`set*()`）一起工作。

`property_path` 选项允许你自定义此映射。

**映射到不同属性**

如果你的表单字段名与对象属性不匹配：

```php
$builder->add('deadline', DateType::class, [
    // 此字段将读/写 'dueDate' 属性
    'property_path' => 'dueDate',
]);
```

**映射到嵌套属性**

你可以使用点表示法访问嵌套的对象属性：

```php
// 假设 Task::getCategory() 返回一个具有 getName()/setName() 的 Category 对象
$builder->add('categoryName', TextType::class, [
    'property_path' => 'category.name',
]);
```

### 在表单类中注入服务

表单类是常规服务，这意味着你可以使用[自动装配](service_container/autowiring.md)注入其他服务：

```php
// src/Form/Type/TaskType.php
namespace App\Form\Type;

use App\Repository\CategoryRepository;
use Symfony\Component\Form\AbstractType;
// ...

class TaskType extends AbstractType
{
    public function __construct(
        private CategoryRepository $categoryRepository,
    ) {
    }

    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        // 使用 $this->categoryRepository 访问仓库
    }
}
```

---

## 渲染表单 {#rendering-forms}

现在表单已创建，下一步是渲染它：

```php
// src/Controller/TaskController.php
namespace App\Controller;

use App\Entity\Task;
use App\Form\Type\TaskType;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

class TaskController extends AbstractController
{
    public function new(Request $request): Response
    {
        $task = new Task();
        // ...

        $form = $this->createForm(TaskType::class, $task);

        return $this->render('task/new.html.twig', [
            'form' => $form,
        ]);
    }
}
```

然后，使用一些[表单辅助函数](reference/forms/twig_reference.md)来渲染表单内容：

```twig
{# templates/task/new.html.twig #}
{{ form(form) }}
```

[`form()` 函数](reference/forms/twig_reference.md)渲染所有字段*以及* `<form>` 开始和结束标签。默认情况下，表单方法是 `POST`，目标 URL 与显示表单的相同，但[你可以更改两者](#更改-action-和-http-方法)。

注意渲染的 `task` 输入字段如何具有 `$task` 对象的 `task` 属性的值（即"Write a blog post"）。这是表单的第一个工作：从对象获取数据，并将其转换为适合在 HTML 表单中渲染的格式。

> **提示**
>
> 表单系统足够智能，可以通过 `Task` 类上的 `getTask()` 和 `setTask()` 方法访问受保护的 `task` 属性的值。除非属性是公共的，否则它*必须*有一个"getter"和"setter"方法，以便 Symfony 可以获取和放置属性上的数据。对于布尔属性，你可以使用"isser"或"hasser"方法（例如 `isPublished()` 或 `hasReminder()`）代替 getter（例如 `getPublished()` 或 `getReminder()`）。

虽然这种渲染很简短，但不是很灵活。通常，你需要对整个表单或其某些字段的外观有更多控制。例如，由于[Bootstrap 5 与 Symfony 表单集成](form/bootstrap5.md)，你可以设置此选项来生成与 Bootstrap 5 CSS 框架兼容的表单：

```yaml
# config/packages/twig.yaml
twig:
    form_themes: ['bootstrap_5_layout.html.twig']
```

[Symfony 内置表单主题](form/form_themes.md)包括 Bootstrap 3、4 和 5，Foundation 5 和 6，以及 Tailwind 2。你也可以[创建自己的 Symfony 表单主题](form/form_themes.md#创建自己的表单主题)。

除了表单主题，Symfony 还允许你通过多个函数[自定义字段的渲染方式](form/form_customization.md)，分别渲染每个字段部分（小部件、标签、错误、帮助消息等）。

---

## 处理表单 {#processing-forms}

[处理表单的推荐方式](best_practices.md#处理表单)是使用单个动作来渲染表单和处理表单提交。

处理表单意味着将用户提交的数据转换回对象的属性。为此，用户提交的数据必须写入表单对象：

```php
// src/Controller/TaskController.php

// ...
use Symfony\Component\HttpFoundation\Request;

class TaskController extends AbstractController
{
    public function new(Request $request): Response
    {
        // 设置一个新的 $task 对象（删除示例数据）
        $task = new Task();

        $form = $this->createForm(TaskType::class, $task);

        $form->handleRequest($request);
        if ($form->isSubmitted() && $form->isValid()) {
            // $form->getData() 保存提交的值
            // 但是，原始的 `$task` 变量也已更新
            $task = $form->getData();

            // ... 执行某些操作，例如将任务保存到数据库

            return $this->redirectToRoute('task_success');
        }

        return $this->render('task/new.html.twig', [
            'form' => $form,
        ]);
    }
}
```

此控制器遵循处理表单的常见模式，有三条可能的路径：

1. 在浏览器中初始加载页面时，表单尚未提交，`$form->isSubmitted()` 返回 `false`。因此，表单被创建和渲染；

2. 当用户提交表单时，`handleRequest()` 识别到这一点，并立即将提交的数据写回 `$task` 对象的 `task` 和 `dueDate` 属性。然后验证此对象。如果无效，`isValid()` 返回 `false`，表单再次渲染，但现在带有验证错误；

3. 当用户提交带有有效数据的表单时，提交的数据再次写入表单，但这次 `isValid()` 返回 `true`。现在你有机会使用 `$task` 对象执行某些操作（例如将其持久化到数据库），然后将用户重定向到某个其他页面。

> **注意**
>
> 在成功提交表单后重定向用户是一个最佳实践，它可以防止用户点击浏览器的"刷新"按钮并重新提交数据。

### 访问表单数据

你最常使用 `getData()` 方法来访问表单的数据，但 Symfony 表单也提供了在[每一层](#数据转换生命周期)访问数据的方法：

**`getData()`**
返回**模型数据**。这是你最常使用的方法。提交后，它返回填充了所有提交值（已转换为适当的 PHP 类型）的对象（或数组）。

**`getNormData()`**
返回**规范化数据**。在调试转换器问题或需要中间表示时很有用。

**`getViewData()`**
返回**视图数据**。这是渲染到 HTML 字段的内容以及来自用户提交的内容（在转换之前）。

### 使用 submit() 方法 {#processing-forms-submit-method}

`handleRequest()` 方法是处理表单的推荐方式。但是，你也可以使用 `submit()` 方法对表单提交的时机和传递给它的数据有更精细的控制：

```php
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
// ...

public function new(Request $request): Response
{
    $task = new Task();
    $form = $this->createForm(TaskType::class, $task);

    if ($request->isMethod('POST')) {
        $form->submit($request->getPayload()->get($form->getName()));

        if ($form->isSubmitted() && $form->isValid()) {
            // 执行某些操作...

            return $this->redirectToRoute('task_success');
        }
    }

    return $this->render('task/new.html.twig', [
        'form' => $form,
    ]);
}
```

### 处理多个提交按钮 {#processing-forms-multiple-buttons}

当你的表单包含多个提交按钮时，你需要检查点击了哪个按钮以调整控制器中的程序流程。例如，如果你向表单添加了第二个带有"Save and Add"标题的按钮：

```php
$form = $this->createFormBuilder($task)
    ->add('task', TextType::class)
    ->add('dueDate', DateType::class)
    ->add('save', SubmitType::class, ['label' => 'Create Task'])
    ->add('saveAndAdd', SubmitType::class, ['label' => 'Save and Add'])
    ->getForm();
```

在控制器中，使用按钮的 `isClicked()` 方法查询是否点击了"Save and Add"按钮：

```php
if ($form->isSubmitted() && $form->isValid()) {
    // ... 执行某些操作，例如将任务保存到数据库

    $nextAction = $form->get('saveAndAdd')->isClicked()
        ? 'task_new'
        : 'task_success';

    return $this->redirectToRoute($nextAction);
}
```

---

## 验证表单 {#validating-forms}

在 Symfony 中，问题不是"表单"是否有效，而是应用提交数据后底层对象（此示例中的 `$task`）是否有效。调用 `$form->isValid()` 是一个快捷方式，询问 `$task` 对象是否具有有效数据。

在使用验证之前，先在你的应用中添加对它的支持：

```terminal
$ composer require symfony/validator
```

验证是通过向类添加一组规则（称为验证约束）来完成的。你可以将它们添加到实体类或使用表单类型的 `constraints` 选项：

```php-attributes
// src/Entity/Task.php
namespace App\Entity;

use Symfony\Component\Validator\Constraints as Assert;

class Task
{
    #[Assert\NotBlank]
    public string $task;

    #[Assert\NotBlank]
    #[Assert\Type(\DateTimeInterface::class)]
    protected \DateTimeInterface $dueDate;
}
```

### 禁用验证 {#form-disabling-validation}

有时完全禁止表单验证很有用。对于这些情况，将 `validation_groups` 选项设置为 `false`：

```php
use Symfony\Component\OptionsResolver\OptionsResolver;

public function configureOptions(OptionsResolver $resolver): void
{
    $resolver->setDefaults([
        'validation_groups' => false,
    ]);
}
```

---

## 其他常见表单功能

### 向表单传递选项

如果你[在类中创建表单](#creating-forms-in-classes)，在控制器中构建表单时，可以将自定义选项作为 `createForm()` 的第三个可选参数传递：

```php
// src/Controller/TaskController.php
namespace App\Controller;

use App\Form\Type\TaskType;
// ...

class TaskController extends AbstractController
{
    public function new(): Response
    {
        $task = new Task();
        // 使用一些 PHP 逻辑决定此表单字段是否必填
        $dueDateIsRequired = ...;

        $form = $this->createForm(TaskType::class, $task, [
            'require_due_date' => $dueDateIsRequired,
        ]);

        // ...
    }
}
```

表单必须使用 `configureOptions()` 方法声明它们接受的所有选项：

```php
// src/Form/Type/TaskType.php
namespace App\Form\Type;

use Symfony\Component\OptionsResolver\OptionsResolver;
// ...

class TaskType extends AbstractType
{
    // ...

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            // ...,
            'require_due_date' => false,
        ]);

        $resolver->setAllowedTypes('require_due_date', 'bool');
    }
}
```

### 表单类型选项

每种[表单类型](#form-types)都有许多配置选项，如 [Symfony 表单类型参考](reference/forms/types.md)所述。两个常用选项是 `required` 和 `label`。

**`required` 选项**

最常见的选项是 `required` 选项，它可以应用于任何字段。默认情况下，此选项设置为 `true`，意味着支持 HTML5 的浏览器在提交表单之前要求你填写所有字段。

**`label` 选项**

默认情况下，表单字段的标签是属性名的*人性化*版本（`user` -> `User`；`postalAddress` -> `Postal Address`）。在字段上设置 `label` 选项以显式定义其标签：

```php
->add('dueDate', DateType::class, [
    // 设置为 FALSE 不显示此字段的标签
    'label' => 'To Be Completed Before',
])
```

### 更改 Action 和 HTTP 方法 {#forms-change-action-method}

默认情况下，`<form>` 标签以 `method="post"` 属性渲染，没有 `action` 属性。在构建表单时，使用 `setAction()` 和 `setMethod()` 方法更改此设置：

```php
$form = $this->createFormBuilder($task)
    ->setAction($this->generateUrl('target_route'))
    ->setMethod('GET')
    // ...
    ->getForm();
```

### 更改表单字段名称和 ID {#changing-the-form-name}

当 Symfony 渲染表单时，它按照特定的约定为每个字段生成 HTML `name` 和 `id` 属性。

字段名遵循模式：`formName[fieldName]`。对于嵌套表单，名称进一步嵌套：`formName[childForm][fieldName]`。

`id` 属性遵循类似的模式，但使用下划线而不是括号：`formName_fieldName`。

### 客户端 HTML 验证 {#forms-html5-validation-disable}

由于 HTML5，许多浏览器可以在客户端本机强制执行某些验证约束。可以通过向 `<form>` 标签添加 `novalidate` 属性或向提交标签添加 `formnovalidate` 来禁用客户端验证：

```twig
{# templates/task/new.html.twig #}
{{ form_start(form, {'attr': {'novalidate': 'novalidate'}}) }}
    {{ form_widget(form) }}
{{ form_end(form) }}
```

### 表单类型猜测 {#form-type-guessing}

如果表单处理的对象包含验证约束，Symfony 可以自省该元数据以猜测你字段的类型。

要启用 Symfony 的"猜测机制"，省略 `add()` 方法的第二个参数，或向其传递 `null`：

```php
$builder
    // 如果不定义字段选项，你可以省略第二个参数
    ->add('task')
    // 如果定义字段选项，将 NULL 作为第二个参数传递
    ->add('dueDate', null, ['required' => false])
    ->add('save', SubmitType::class)
;
```

### 未映射字段 {#form-unmapped-fields}

如果你需要在表单中添加不会存储在对象中的额外字段（例如添加*"我同意这些条款"*复选框），在这些字段上将 `mapped` 选项设置为 `false`：

```php
$builder
    ->add('task')
    ->add('dueDate')
    ->add('agreeTerms', CheckboxType::class, ['mapped' => false])
    ->add('save', SubmitType::class)
;
```

这些"未映射字段"可以在控制器中设置和访问：

```php
$form->get('agreeTerms')->getData();
$form->get('agreeTerms')->setData(true);
```

### 额外字段 {#form-extra-fields}

默认情况下，Symfony 期望每个提交的字段都在表单中定义。任何额外的提交字段都被视为"额外字段"。你可以通过 `getExtraData()` 方法访问它们。

---

## 不使用数据类的表单 {#forms-without-class}

在大多数应用中，表单与对象绑定，表单的字段从该对象的属性获取和存储数据。

但是，如果你*不*将表单绑定到对象，表单将以数组的形式返回数据：

```php
$defaultData = ['message' => 'Type your message here'];
$form = $this->createFormBuilder($defaultData)
    ->add('name', TextType::class)
    ->add('email', EmailType::class)
    ->add('message', TextareaType::class)
    ->add('send', SubmitType::class)
    ->getForm();

$form->handleRequest($request);

if ($form->isSubmitted() && $form->isValid()) {
    // data 是一个包含 "name"、"email" 和 "message" 键的数组
    $data = $form->getData();
}
```

### 在字段级别添加约束 {#form-option-constraints}

你可以将约束附加到各个字段：

```php
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\Validator\Constraints as Assert;

public function buildForm(FormBuilderInterface $builder, array $options): void
{
    $builder
        ->add('firstName', TextType::class, [
            'constraints' => new Assert\Length(['min' => 3]),
        ])
        ->add('lastName', TextType::class, [
            'constraints' => [
                new Assert\NotBlank(),
                new Assert\Length(['min' => 3]),
            ],
        ])
    ;
}
```

### 在类级别添加约束

你也可以在类级别添加约束：

```php
public function configureOptions(OptionsResolver $resolver): void
{
    $resolver->setDefaults([
        'data_class' => null,
        'constraints' => new Assert\Collection([
            'firstName' => new Assert\Length(['min' => 3]),
            'lastName' => [
                new Assert\NotBlank(),
                new Assert\Length(['min' => 3]),
            ],
        ]),
    ]);
}
```

### 条件约束

可以定义依赖于其他字段值的字段约束。为此，使用 [When 约束](reference/constraints/When.md)的 `expression` 选项来引用其他字段：

```php
use Symfony\Component\Validator\Constraints as Assert;

$builder
    ->add('how_did_you_hear', ChoiceType::class, [
        'required' => true,
        'label' => 'How did you hear about us?',
        'choices' => [
            'Search engine' => 'search_engine',
            'Friends' => 'friends',
            'Other' => 'other',
        ],
        'expanded' => true,
        'constraints' => [
            new Assert\NotBlank(),
        ]
    ])

    // 此字段仅在 'how_did_you_hear' 为 'other' 时才必填
    ->add('other_text', TextType::class, [
        'required' => false,
        'label' => 'Please specify',
        'constraints' => [
            new Assert\When(
                expression: 'this.getParent().get("how_did_you_hear").getData() == "other"',
                constraints: [
                    new Assert\NotBlank(),
                ],
            )
        ],
    ])
;
```

---

## 故障排除

### 为什么我的字段值不显示？

**问题**：表单渲染了，但即使基础数据有值，字段也是空的。

**常见原因**：

1. 属性不可读（缺少访问器、名称错误或不公开）。对于布尔值，Symfony 还查找 `is*()` 和 `has*()` 访问器。
2. 字段名与属性名不匹配。如果需要映射到不同的属性，使用 `property_path`。
3. 数据在创建表单后设置。在将对象传递给 `createForm()` 之前填充它。

### 为什么我的提交数据没有保存到对象？

**问题**：表单提交了，但对象属性保持不变。

**常见原因**：

1. 属性不可写（缺少 setter、名称错误或不公开）。
2. 它是一个[未映射字段](#form-unmapped-fields)。
3. 由于转换失败，表单未同步。检查 `isSynchronized()` 并检查字段错误。

### 为什么 `getData()` 在提交后返回 `null`？

**问题**：处理请求后 `$form->getData()` 是 `null`。

**常见原因**：

1. 没有提供初始对象（或默认数据），表单也不创建一个。检查表单的 `data_class` 和 `empty_data` 选项。
2. 转换失败，表单未同步。检查 `isSynchronized()` 和字段错误。
3. 表单未提交或无效。在使用数据之前检查 `isSubmitted()` 和 `isValid()`。

---

## 延伸阅读

- **参考**：[Symfony 表单类型](reference/forms/types.md)

- **高级功能**：
  - [上传文件](controller/upload_file.md)
  - [CSRF 保护](security/csrf.md)
  - [创建自定义字段类型](form/create_custom_field_type.md)
  - [数据转换器](form/data_transformers.md)
  - [数据映射器](form/data_mappers.md)
  - [创建表单类型扩展](form/create_form_type_extension.md)
  - [类型猜测器](form/type_guesser.md)

- **表单主题和自定义**：
  - [Bootstrap 4](form/bootstrap4.md)
  - [Bootstrap 5](form/bootstrap5.md)
  - [Tailwind CSS](form/tailwindcss.md)
  - [自定义表单渲染](form/form_customization.md)
  - [表单主题](form/form_themes.md)

- **事件**：
  - [表单事件](form/events.md)
  - [动态表单修改](form/dynamic_form_modification.md)

- **验证**：[验证组](form/validation_groups.md)

- **其他**：
  - [嵌入表单](form/embedded.md)
  - [表单集合](form/form_collections.md)
  - [表单流程](form/form_flow.md)
  - [继承数据选项](form/inherit_data_option.md)
  - [单元测试](form/unit_testing.md)
  - [使用空数据](form/use_empty_data.md)
