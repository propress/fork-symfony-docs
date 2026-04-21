# 工作流（Workflow）

在 Symfony 应用程序中使用 Workflow 组件之前，需要先了解关于工作流和状态机的一些基本理论和概念。[阅读此文章](../workflow/workflow-and-state-machine.md)可快速了解概述。

## 安装

在使用 Symfony Flex 的应用程序中，在使用工作流功能之前，运行以下命令安装：

```terminal
$ composer require symfony/workflow
```

## 配置

如需查看所有配置选项，在 Symfony 项目中运行以下命令：

```terminal
$ php bin/console config:dump-reference framework workflows
```

## 创建工作流

工作流是对象所经历的流程或生命周期。流程中的每个步骤或阶段称为一个*位置（place）*。你还需要定义*转换（transition）*，用于描述从一个位置到另一个位置所需的操作。

![工作流示例状态图，展示转换和位置。](/_images/components/workflow/states_transitions.png)

一组位置和转换构成一个**定义（definition）**。工作流需要一个 `Definition` 以及一种将状态写入对象的方式（即 `MarkingStoreInterface` 的一个实例）。

以下是一个博客文章的示例。一篇文章可以有这些位置：`draft`、`reviewed`、`rejected`、`published`。你可以按如下方式定义工作流：

```yaml
# config/packages/workflow.yaml
framework:
    workflows:
        blog_publishing:
            type: 'workflow' # or 'state_machine'
            audit_trail:
                enabled: true
            marking_store:
                type: 'method'
                property: 'currentPlace'
            supports:
                - App\Entity\BlogPost
            initial_marking: draft
            places:          # defining places manually is optional
                - draft
                - reviewed
                - rejected
                - published
            transitions:
                to_review:
                    from: draft
                    to:   reviewed
                publish:
                    from: reviewed
                    to:   published
                reject:
                    from: reviewed
                    to:   rejected
```

```php
// config/packages/workflow.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Entity\BlogPost;

return App::config([
    'framework' => [
        'workflows' => [
            'blog_publishing' => [
                'type' => 'workflow', // or 'state_machine'
                'audit_trail' => [
                    'enabled' => true,
                ],
                'marking_store' => [
                    'type' => 'method',
                    'property' => 'currentPlace',
                ],
                'supports' => [BlogPost::class],
                'initial_marking' => 'draft',
                'places' => [
                    'draft',
                    'reviewed',
                    'rejected',
                    'published',
                ],
                'transitions' => [
                    'to_review' => [
                        'from' => 'draft',
                        'to' => 'reviewed',
                    ],
                    'publish' => [
                        'from' => 'reviewed',
                        'to' => 'published',
                    ],
                    'reject' => [
                        'from' => 'reviewed',
                        'to' => 'rejected',
                    ],
                ],
            ],
        ],
    ],
]);
```

> **提示：** 如果你正在创建第一个工作流，可以考虑使用 `workflow:dump` 命令来[调试工作流内容](../workflow/dumping-workflows.md)。

> **提示：** 你可以在 YAML 文件中通过 `!php/const ` 符号使用 PHP 常量。例如，你可以使用 `!php/const App\Entity\BlogPost::STATE_DRAFT` 代替 `'draft'`，或者使用 `!php/const App\Entity\BlogPost::TRANSITION_TO_REVIEW` 代替 `'to_review'`。

> **提示：** 如果你的转换定义了工作流中使用的所有位置，可以省略 `places` 选项。Symfony 将自动从转换中提取位置。

配置的属性将通过其实现的 getter/setter 方法被标记存储（marking store）使用：

```php
// src/Entity/BlogPost.php
namespace App\Entity;

class BlogPost
{
    // the configured marking store property must be declared
    private string $currentPlace;
    private string $title;
    private string $content;

    // getter/setter methods must exist for property access by the marking store
    public function getCurrentPlace(): string
    {
        return $this->currentPlace;
    }

    public function setCurrentPlace(string $currentPlace, array $context = []): void
    {
        $this->currentPlace = $currentPlace;
    }

    // you don't need to set the initial marking in the constructor or any other method;
    // this is configured in the workflow with the 'initial_marking' option
}
```

也可以为标记存储使用公共属性。上面的类可以改写为：

```php
// src/Entity/BlogPost.php
namespace App\Entity;

class BlogPost
{
    // the configured marking store property must be declared
    public string $currentPlace;
    public string $title;
    public string $content;
}
```

使用公共属性时不支持上下文（context）。如果需要支持上下文，必须声明一个 setter 来写入属性：

```php
// src/Entity/BlogPost.php
namespace App\Entity;

class BlogPost
{
    public string $currentPlace;
    // ...

    public function setCurrentPlace(string $currentPlace, array $context = []): void
    {
        // assign the property and do something with the context
    }
}
```

> **注意：** 标记存储类型可以是 "multiple_state" 或 "single_state"。单状态标记存储不支持对象同时处于多个位置。这意味着 "workflow" 类型必须使用 "multiple_state" 标记存储，"state_machine" 类型必须使用 "single_state" 标记存储。Symfony 默认会根据 "type" 配置标记存储，因此建议不要单独配置它。
>
> 单状态标记存储使用 `string` 存储数据。多状态标记存储使用 `array` 存储数据。如果未定义状态标记存储，则两种情况下都必须返回 `null`（例如，上面的示例应该将返回类型定义为 `App\Entity\BlogPost::getCurrentPlace(): ?array` 或 `App\Entity\BlogPost::getCurrentPlace(): ?string`）。

> **提示：** `marking_store` 选项的 `marking_store.type`（默认值取决于 `type` 值）和 `property`（默认值 `['marking']`）属性是可选的。如果省略，将使用其默认值。强烈建议使用默认值。

> **提示：** 将 `audit_trail.enabled` 选项设置为 `true` 会使应用程序为工作流活动生成详细的日志消息。

有了名为 `blog_publishing` 的工作流，你可以获得帮助来决定对博客文章允许哪些操作：

```php
use App\Entity\BlogPost;
use Symfony\Component\Workflow\Exception\LogicException;

$post = new BlogPost();
// you don't need to set the initial marking with code; this is configured
// in the workflow with the 'initial_marking' option

$workflow = $this->container->get('workflow.blog_publishing');
$workflow->can($post, 'publish'); // False
$workflow->can($post, 'to_review'); // True

// Update the currentState on the post
try {
    $workflow->apply($post, 'to_review');
} catch (LogicException $exception) {
    // ...
}

// See all the available transitions for the post in the current state
$transitions = $workflow->getEnabledTransitions($post);
// See a specific available transition for the post in the current state
$transition = $workflow->getEnabledTransition($post, 'publish');
```

## 在工作流中使用枚举

### 在工作流定义中使用枚举

使用状态机时，你可以将 PHP 后端枚举用作工作流中的位置。首先，用支持的值定义你的枚举：

```php
// src/Enumeration/BlogPostStatus.php
namespace App\Enumeration;

enum BlogPostStatus: string
{
    case Draft = 'draft';
    case Reviewed = 'reviewed';
    case Published = 'published';
    case Rejected = 'rejected';
}
```

然后将枚举 case 作为位置、初始标记和转换来配置工作流：

```yaml
# config/packages/workflow.yaml
framework:
    workflows:
        blog_publishing:
            type: 'workflow'
            marking_store:
                type: 'method'
                property: 'status'
            supports:
                - App\Entity\BlogPost
            initial_marking: !php/enum App\Enumeration\BlogPostStatus::Draft
            places: !php/enum App\Enumeration\BlogPostStatus
            transitions:
                to_review:
                    from: !php/enum App\Enumeration\BlogPostStatus::Draft
                    to:   !php/enum App\Enumeration\BlogPostStatus::Reviewed
                publish:
                    from: !php/enum App\Enumeration\BlogPostStatus::Reviewed
                    to:   !php/enum App\Enumeration\BlogPostStatus::Published
                reject:
                    from: !php/enum App\Enumeration\BlogPostStatus::Reviewed
                    to:   !php/enum App\Enumeration\BlogPostStatus::Rejected
```

```php
// config/packages/workflow.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Entity\BlogPost;
use App\Enumeration\BlogPostStatus;

return App::config([
    'framework' => [
        'workflows' => [
            'blog_publishing' => [
                'type' => 'workflow',
                'marking_store' => [
                    'type' => 'method',
                    'property' => 'status',
                ],
                'supports' => [BlogPost::class],
                'initial_marking' => BlogPostStatus::Draft,
                'places' => BlogPostStatus::cases(),
                'transitions' => [
                    'to_review' => [
                        'from' => BlogPostStatus::Draft,
                        'to' => BlogPostStatus::Reviewed,
                    ],
                    'publish' => [
                        'from' => BlogPostStatus::Reviewed,
                        'to' => BlogPostStatus::Published,
                    ],
                    'reject' => [
                        'from' => BlogPostStatus::Reviewed,
                        'to' => BlogPostStatus::Rejected,
                    ],
                ],
            ],
        ],
    ],
]);
```

该组件现在将在需要时透明地将枚举转换为其支持的值，反之亦然：

```php
// src/Entity/BlogPost.php
namespace App\Entity;

class BlogPost
{
    private BlogPostStatus $status;

    public function getStatus(): BlogPostStatus
    {
        return $this->status;
    }

    public function setStatus(BlogPostStatus $status): void
    {
        $this->status = $status;
    }
}
```

> **提示：** 你还可以使用 PHP 常量和枚举的 [glob 模式](https://php.net/glob)来列出位置：
>
> ```yaml
> # config/packages/workflow.yaml
> framework:
>     workflows:
>         my_workflow_name:
>             # with constants:
>             places: 'App\Workflow\MyWorkflow::PLACE_*'
>
>             # with enums:
>             places: !php/enum App\Workflow\Places
>
>             # ...
> ```
>
> ```php
> // config/packages/workflow.php
> namespace Symfony\Component\DependencyInjection\Loader\Configurator;
>
> use App\Enumeration\BlogPostStatus;
>
> return App::config([
>     'framework' => [
>         'workflows' => [
>             'my_workflow_name' => [
>                 // with constants:
>                 'places' => 'App\Workflow\MyWorkflow::PLACE_*',
>                 // with enums:
>                 'places' => BlogPostStatus::cases(),
>             ],
>         ],
>     ],
> ]);
> ```

### 在标记存储中使用枚举

使用单状态标记存储时，你可以用 `BackedEnum` 而不是字符串来类型提示属性。`MethodMarkingStore` 将自动在枚举和其支持值之间转换：

```php
// src/Entity/Status.php
namespace App\Entity;

enum Status: string
{
    case Draft = 'draft';
    case Reviewed = 'reviewed';
    case Published = 'published';
}

// src/Entity/BlogPost.php
namespace App\Entity;

class BlogPost
{
    public ?Status $currentPlace = null;

    public function getCurrentPlace(): ?Status
    {
        return $this->currentPlace;
    }

    public function setCurrentPlace(Status $currentPlace, array $context = []): void
    {
        $this->currentPlace = $currentPlace;
    }
}
```

## 使用加权转换

工作流（与状态机相比）的一个关键特性是对象可以同时处于多个位置。例如，在制造产品时，你可能会并行组装多个组件。然而，在前面的示例中，每个位置只能记录对象是否在那里，就像一个二进制标志。

**加权转换**引入了多重性：一个位置现在可以追踪对象在该位置的次数。从技术上说，加权转换允许你定义从位置消耗或向位置产生多个令牌（实例）的转换。这对于建模复杂的工作流（如制造流程、资源分配或任何需要产生或消耗多个实例的场景）非常有用。

例如，假设有一个制作桌子的工作流，你需要创建 4 条腿、1 个桌面，并用秒表跟踪整个过程。你可以使用加权转换来建模：

```yaml
# config/packages/workflow.yaml
framework:
    workflows:
        make_table:
            type: 'workflow'
            marking_store:
                type: 'method'
                property: 'marking'
            supports:
                - App\Entity\TableProject
            initial_marking: init
            places:
                - init
                - prepare_leg
                - prepare_top
                - stopwatch_running
                - leg_created
                - top_created
                - finished
            transitions:
                start:
                    from: init
                    to:
                        - place: prepare_leg
                          weight: 4
                        - place: prepare_top
                          weight: 1
                        - place: stopwatch_running
                          weight: 1
                build_leg:
                    from: prepare_leg
                    to: leg_created
                build_top:
                    from: prepare_top
                    to: top_created
                join:
                    from:
                        - place: leg_created
                          weight: 4
                        - top_created  # weight defaults to 1
                        - stopwatch_running
                    to: finished
```

```php
// config/packages/workflow.php
use App\Entity\TableProject;
use Symfony\Config\FrameworkConfig;

return static function (FrameworkConfig $framework): void {
    $makeTable = $framework->workflows()->workflows('make_table');
    $makeTable
        ->type('workflow')
        ->supports([TableProject::class])
        ->initialMarking(['init']);

    $makeTable->markingStore()
        ->type('method')
        ->property('marking');

    $makeTable->place()->name('init');
    $makeTable->place()->name('prepare_leg');
    $makeTable->place()->name('prepare_top');
    $makeTable->place()->name('stopwatch_running');
    $makeTable->place()->name('leg_created');
    $makeTable->place()->name('top_created');
    $makeTable->place()->name('finished');

    $makeTable->transition()
        ->name('start')
            ->from(['init'])
            ->to([
                ['place' => 'prepare_leg', 'weight' => 4],
                ['place' => 'prepare_top', 'weight' => 1],
                ['place' => 'stopwatch_running', 'weight' => 1],
            ]);

    $makeTable->transition()
        ->name('build_leg')
            ->from(['prepare_leg'])
            ->to(['leg_created']);

    $makeTable->transition()
        ->name('build_top')
            ->from(['prepare_top'])
            ->to(['top_created']);

    $makeTable->transition()
        ->name('join')
            ->from([
                ['place' => 'leg_created', 'weight' => 4],
                'top_created',  // weight defaults to 1
                'stopwatch_running',
            ])
            ->to(['finished']);
};
```

在这个例子中，当应用 `start` 转换时，它会在 `prepare_leg` 位置创建 4 个令牌，在 `prepare_top` 中创建 1 个，在 `stopwatch_running` 中创建 1 个。然后，`build_leg` 转换必须被应用 4 次（每个令牌一次），`build_top` 转换一次。最后，只有当所有 4 条腿创建完成、桌面创建完成且秒表仍在运行时，才能应用 `join` 转换。

加权转换也可以通过 `Arc` 类以编程方式定义：

```php
use Symfony\Component\Workflow\Arc;
use Symfony\Component\Workflow\Definition;
use Symfony\Component\Workflow\Transition;
use Symfony\Component\Workflow\Workflow;

$definition = new Definition(
    ['init', 'prepare_leg', 'prepare_top', 'stopwatch_running', 'leg_created', 'top_created', 'finished'],
    [
        new Transition('start', 'init', [
            new Arc('prepare_leg', 4),
            new Arc('prepare_top', 1),
            'stopwatch_running',  // defaults to weight 1
        ]),
        new Transition('build_leg', 'prepare_leg', 'leg_created'),
        new Transition('build_top', 'prepare_top', 'top_created'),
        new Transition('join', [
            new Arc('leg_created', 4),
            'top_created',
            'stopwatch_running',
        ], 'finished'),
    ]
);

$workflow = new Workflow($definition);
$workflow->apply($subject, 'start');

// Build each leg (4 times)
$workflow->apply($subject, 'build_leg');
$workflow->apply($subject, 'build_leg');
$workflow->apply($subject, 'build_leg');
$workflow->apply($subject, 'build_leg');

// Build the top
$workflow->apply($subject, 'build_top');

// Now we can join all parts
$workflow->apply($subject, 'join');
```

`Arc` 类接受两个参数：位置名称和权重（必须大于或等于 1）。当位置以简单字符串而不是 `Arc` 对象指定时，其默认权重为 1。

## 使用多状态标记存储

如果你在创建[工作流](../workflow/workflow-and-state-machine.md)，你的标记存储可能需要同时包含多个位置。因此，如果你在使用 Doctrine，对应的列定义应该使用 `json` 类型：

```php
// src/Entity/BlogPost.php
namespace App\Entity;

use Doctrine\DBAL\Types\Types;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class BlogPost
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private int $id;

    #[ORM\Column(type: Types::JSON)]
    private array $currentPlaces;

    // ...
}
```

> **警告：** 不应该将 `simple_array` 类型用于你的标记存储。在多状态标记存储中，位置以键的形式存储，值为 1，例如 `['draft' => 1]`。如果标记存储中只有一个位置，Doctrine 的这种类型只会将其值存储为字符串，从而导致对象当前位置的丢失。

## 在类中访问工作流

Symfony 为你定义的每个工作流创建一个服务。你有两种方式将每个工作流注入到任何服务或控制器中：

**(1) 使用特定的参数名**

用 `WorkflowInterface` 对构造函数/方法参数进行类型提示，并按照此模式命名参数："工作流名称（驼峰格式）" + `Workflow` 后缀。如果是状态机类型，则使用 `StateMachine` 后缀。

例如，要注入之前定义的 `blog_publishing` 工作流：

```php
use App\Entity\BlogPost;
use Symfony\Component\Workflow\WorkflowInterface;

class MyClass
{
    public function __construct(
        private WorkflowInterface $blogPublishingWorkflow,
    ) {
    }

    public function toReview(BlogPost $post): void
    {
        try {
            // update the currentState on the post
            $this->blogPublishingWorkflow->apply($post, 'to_review');
        } catch (LogicException $exception) {
            // ...
        }
        // ...
    }
}
```

**(2) 使用 `#[Target]` 属性**

当处理同一类型的多个实现时，`#[Target]` 属性帮助你选择要注入哪一个。Symfony 为每个工作流创建一个与工作流同名的目标（target）。

例如，要选择之前定义的 `blog_publishing` 工作流：

```php
use Symfony\Component\DependencyInjection\Attribute\Target;
use Symfony\Component\Workflow\WorkflowInterface;

class MyClass
{
    public function __construct(
        #[Target('blog_publishing')] private WorkflowInterface $workflow,
    ) {
    }

    // ...
}
```

要获取工作流的已启用转换，你可以使用 `WorkflowInterface::getEnabledTransition` 方法。

> **提示：** 如果你想检索所有工作流（例如用于文档目的），可以使用以下标签[注入所有服务](../service_container/service_subscribers_locators.md)：
>
> * `workflow`：所有工作流和所有状态机；
> * `workflow.workflow`：所有工作流；
> * `workflow.state_machine`：所有状态机。
>
> 注意，工作流元数据附加在标签的 `metadata` 键下，为你提供关于工作流的更多上下文和信息。了解更多关于标签属性和存储工作流元数据的内容。

> **提示：** 你可以使用 `php bin/console debug:autowiring workflow` 命令查找可用的工作流服务列表。

### 注入多个工作流

使用 `AutowireLocator` 属性来延迟加载所有工作流并获取你需要的那个：

```php
use Symfony\Component\DependencyInjection\Attribute\AutowireLocator;
use Symfony\Component\DependencyInjection\ServiceLocator;

class MyClass
{
    public function __construct(
        // 'workflow' is the service tag name and injects both workflows and state machines;
        // 'name' tells Symfony to index services using that tag property
        #[AutowireLocator('workflow', 'name')]
        private ServiceLocator $workflows,
    ) {
    }

    public function someMethod(): void
    {
        // if you use the 'name' tag property to index services (see constructor above),
        // you can get workflows by their name; otherwise, you must use the full
        // service name with the 'workflow.' prefix (e.g. 'workflow.user_registration')
        $workflow = $this->workflows->get('user_registration');

        // ...
    }
}
```

> **提示：** 你也可以只注入工作流或只注入状态机：
>
> ```php
> public function __construct(
>     #[AutowireLocator('workflow.workflow', 'name')]
>     private ServiceLocator $workflows,
>     #[AutowireLocator('workflow.state_machine', 'name')]
>     private ServiceLocator $stateMachines,
> ) {
> }
> ```

## 使用事件

为了使你的工作流更加灵活，你可以使用 `EventDispatcher` 构造 `Workflow` 对象。你现在可以创建事件监听器来阻止转换（例如根据博客文章中的数据）并在工作流操作发生时执行额外的操作（例如发送通知）。

每个步骤都有三个按顺序触发的事件：

* 针对每个工作流的事件；
* 针对相关工作流的事件；
* 针对相关工作流中特定转换或位置名称的事件。

当状态转换被触发时，事件按以下顺序分发：

`workflow.guard`
    验证转换是否被阻止（参见守卫事件和阻止转换）。

    分发的三个事件是：

    * `workflow.guard`
    * `workflow.[workflow name].guard`
    * `workflow.[workflow name].guard.[transition name]`

`workflow.leave`
    对象即将离开某个位置。

    分发的三个事件是：

    * `workflow.leave`
    * `workflow.[workflow name].leave`
    * `workflow.[workflow name].leave.[place name]`

`workflow.transition`
    对象正在经历此转换。

    分发的三个事件是：

    * `workflow.transition`
    * `workflow.[workflow name].transition`
    * `workflow.[workflow name].transition.[transition name]`

`workflow.enter`
    对象即将进入新位置。此事件在对象位置更新之前触发，这意味着对象的标记尚未更新为新位置。

    分发的三个事件是：

    * `workflow.enter`
    * `workflow.[workflow name].enter`
    * `workflow.[workflow name].enter.[place name]`

`workflow.entered`
    对象已进入位置且标记已更新。

    分发的三个事件是：

    * `workflow.entered`
    * `workflow.[workflow name].entered`
    * `workflow.[workflow name].entered.[place name]`

`workflow.completed`
    对象已完成此转换。

    分发的三个事件是：

    * `workflow.completed`
    * `workflow.[workflow name].completed`
    * `workflow.[workflow name].completed.[transition name]`

`workflow.announce`
    针对对象现在可访问的每个转换触发。

    分发的三个事件是：

    * `workflow.announce`
    * `workflow.[workflow name].announce`
    * `workflow.[workflow name].announce.[transition name]`

    应用转换后，announce 事件会测试所有可用的转换。这将再次触发所有守卫事件，如果它们包含密集的 CPU 或数据库工作，可能会影响性能。

    如果你不需要 announce 事件，可以使用上下文禁用它：

    ```php
    $workflow->apply($subject, $transitionName, [Workflow::DISABLE_ANNOUNCE_EVENT => true]);
    ```

> **注意：** 即使对于停留在同一位置的转换，也会触发离开和进入事件。

> **注意：** 如果通过调用 `$workflow->getMarking($object);` 初始化标记，则将使用默认上下文（`Workflow::DEFAULT_INITIAL_CONTEXT`）调用 `workflow.[workflow_name].entered.[initial_place_name]` 事件。

以下是一个示例，展示如何在 "blog_publishing" 工作流每次离开某个位置时启用日志记录：

```php
// src/EventSubscriber/WorkflowLoggerSubscriber.php
namespace App\EventSubscriber;

use Psr\Log\LoggerInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\Workflow\Event\Event;
use Symfony\Component\Workflow\Event\LeaveEvent;

class WorkflowLoggerSubscriber implements EventSubscriberInterface
{
    public function __construct(
        private LoggerInterface $logger,
    ) {
    }

    public function onLeave(Event $event): void
    {
        $this->logger->alert(sprintf(
            'Blog post (id: "%s") performed transition "%s" from "%s" to "%s"',
            $event->getSubject()->getId(),
            $event->getTransition()->getName(),
            implode(', ', array_keys($event->getMarking()->getPlaces())),
            implode(', ', $event->getTransition()->getTos())
        ));
    }

    public static function getSubscribedEvents(): array
    {
        return [
            LeaveEvent::getName('blog_publishing') => 'onLeave',
            // if you prefer, you can write the event name manually like this:
            // 'workflow.blog_publishing.leave' => 'onLeave',
        ];
    }
}
```

> **提示：** 所有内置工作流事件都定义了 `getName(?string $workflowName, ?string $transitionOrPlaceName)` 方法，以便无需处理字符串即可构建完整的事件名称。你也可以通过 `EventNameTrait` 在自定义事件中使用此方法。

如果某些监听器在转换期间更新了上下文，你可以通过标记来检索它：

```php
$marking = $workflow->apply($post, 'to_review');

// contains the new value
$marking->getContext();
```

也可以使用以下属性声明事件监听器来监听这些事件：

* `AsAnnounceListener`
* `AsCompletedListener`
* `AsEnterListener`
* `AsEnteredListener`
* `AsGuardListener`
* `AsLeaveListener`
* `AsTransitionListener`

这些属性的工作方式类似于 `AsEventListener` 属性：

```php
class ArticleWorkflowEventListener
{
    #[AsTransitionListener(workflow: 'my-workflow', transition: 'published')]
    public function onPublishedTransition(TransitionEvent $event): void
    {
        // ...
    }

    // ...
}
```

你可以参阅关于[使用 PHP 属性定义事件监听器](../event_dispatcher.md)的文档以获取进一步的用法。

### 守卫事件

有一种称为"守卫事件"的特殊事件类型。每次调用 `Workflow::can()`、`Workflow::apply()` 或 `Workflow::getEnabledTransitions()` 时，都会调用其事件监听器。通过守卫事件，你可以添加自定义逻辑来决定哪些转换应该被阻止或允许。以下是守卫事件名称列表：

* `workflow.guard`
* `workflow.[workflow name].guard`
* `workflow.[workflow name].guard.[transition name]`

此示例阻止任何缺少标题的博客文章转换到 "reviewed" 状态：

```php
// src/EventSubscriber/BlogPostReviewSubscriber.php
namespace App\EventSubscriber;

use App\Entity\BlogPost;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\Workflow\Event\GuardEvent;

class BlogPostReviewSubscriber implements EventSubscriberInterface
{
    public function guardReview(GuardEvent $event): void
    {
        /** @var BlogPost $post */
        $post = $event->getSubject();
        $title = $post->title;

        if (empty($title)) {
            $event->setBlocked(true, 'This blog post cannot be marked as reviewed because it has no title.');
        }
    }

    public static function getSubscribedEvents(): array
    {
        return [
            'workflow.blog_publishing.guard.to_review' => ['guardReview'],
        ];
    }
}
```

### 选择要分发的事件

如果你希望控制执行每个转换时触发哪些事件，请使用 `events_to_dispatch` 配置选项。此选项不适用于守卫事件，守卫事件始终会触发：

```yaml
# config/packages/workflow.yaml
framework:
    workflows:
        blog_publishing:
            # you can pass one or more event names
            events_to_dispatch: ['workflow.leave', 'workflow.completed']

            # pass an empty array to not dispatch any event
            events_to_dispatch: []

            # ...
```

```php
// config/packages/workflow.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'workflows' => [
            'blog_publishing' => [
                // you can pass one or more event names
                'events_to_dispatch' => ['workflow.leave', 'workflow.completed'],
                // pass an empty array to not dispatch any event
                'events_to_dispatch' => [],
                // ...
            ],
        ],
    ],
]);
```

你还可以禁用在应用转换时触发的特定事件：

```php
use App\Entity\BlogPost;
use Symfony\Component\Workflow\Exception\LogicException;

$post = new BlogPost();

try {
    $blogPublishingWorkflow->apply($post, 'to_review', [
        Workflow::DISABLE_ANNOUNCE_EVENT => true,
        Workflow::DISABLE_LEAVE_EVENT => true,
    ]);
} catch (LogicException $exception) {
    // ...
}
```

为特定转换禁用事件的优先级高于工作流配置中指定的任何事件。在上面的示例中，即使在工作流配置中将 `workflow.leave` 事件指定为所有转换都要分发的事件，它也不会被触发。

以下是所有可用的常量：

* `Workflow::DISABLE_LEAVE_EVENT`
* `Workflow::DISABLE_TRANSITION_EVENT`
* `Workflow::DISABLE_ENTER_EVENT`
* `Workflow::DISABLE_ENTERED_EVENT`
* `Workflow::DISABLE_COMPLETED_EVENT`

### 事件方法

每个工作流事件都是 `Event` 的一个实例。这意味着每个事件都可以访问以下信息：

`Event::getMarking()`
    返回工作流的 `Marking`。

`Event::getSubject()`
    返回分发事件的对象。

`Event::getTransition()`
    返回分发事件的 `Transition`。

`Event::getWorkflowName()`
    返回触发事件的工作流名称字符串。

`Event::getMetadata()`
    返回元数据。

对于守卫事件，有一个扩展的 `GuardEvent` 类。该类具有以下额外方法：

`GuardEvent::isBlocked()`
    返回转换是否被阻止。

`GuardEvent::setBlocked()`
    设置阻止值。

`GuardEvent::getTransitionBlockerList()`
    返回事件的 `TransitionBlockerList`。参见阻止转换。

`GuardEvent::addTransitionBlocker()`
    添加一个 `TransitionBlocker` 实例。

## 阻止转换

可以通过在应用转换之前调用自定义逻辑来控制工作流的执行，以决定当前转换是否被阻止或允许。此功能由"守卫"提供，有两种使用方式。

首先，你可以监听守卫事件。或者，你可以为转换定义一个 `guard` 配置选项。此选项的值是使用 ExpressionLanguage 组件创建的任何有效表达式：

```yaml
# config/packages/workflow.yaml
framework:
    workflows:
        blog_publishing:
            # previous configuration
            transitions:
                to_review:
                    # the transition is allowed only if the current user has the ROLE_REVIEWER role.
                    guard: "is_granted('ROLE_REVIEWER')"
                    from: draft
                    to:   reviewed
                publish:
                    # or "is_remember_me", "is_fully_authenticated", "is_granted", "is_valid"
                    guard: "is_authenticated"
                    from: reviewed
                    to:   published
                reject:
                    # or any valid expression language with "subject" referring to the supported object
                    guard: "is_granted('ROLE_ADMIN') and subject.isRejectable()"
                    from: reviewed
                    to:   rejected
```

```php
// config/packages/workflow.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'workflows' => [
            'blog_publishing' => [
                'transitions' => [
                    'to_review' => [
                        'guard' => 'is_granted("ROLE_REVIEWER")',
                        'from' => ['draft'],
                        'to' => ['reviewed'],
                    ],
                    'publish' => [
                        // or "is_remember_me", "is_fully_authenticated", "is_granted"
                        'guard' => 'is_authenticated',
                        'from' => ['reviewed'],
                        'to' => ['published'],
                    ],
                    'reject' => [
                        // or any valid expression language with "subject" referring to the post
                        'guard' => 'is_granted("ROLE_ADMIN") and subject.isStatusReviewed()',
                        'from' => ['reviewed'],
                        'to' => ['rejected'],
                    ],
                ],
            ],
        ],
    ],
]);
```

你还可以使用转换阻止器（transition blockers）来阻止转换并在阻止转换发生时返回用户友好的错误消息。在示例中，我们从 `Event` 的元数据中获取此消息，为你提供一个集中管理文本的地方。

此示例已简化；在生产环境中，你可能更希望使用 [Translation](../translation.md) 组件在一个地方管理消息：

```php
// src/EventSubscriber/BlogPostPublishSubscriber.php
namespace App\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\Workflow\Event\GuardEvent;
use Symfony\Component\Workflow\TransitionBlocker;

class BlogPostPublishSubscriber implements EventSubscriberInterface
{
    public function guardPublish(GuardEvent $event): void
    {
        $eventTransition = $event->getTransition();
        $hourLimit = $event->getMetadata('hour_limit', $eventTransition);

        if (date('H') <= $hourLimit) {
            return;
        }

        // Block the transition "publish" if it is more than 8 PM
        // with the message for end user
        $explanation = $event->getMetadata('explanation', $eventTransition);
        $event->addTransitionBlocker(new TransitionBlocker($explanation , '0'));
    }

    public static function getSubscribedEvents(): array
    {
        return [
            'workflow.blog_publishing.guard.publish' => ['guardPublish'],
        ];
    }
}
```

## 创建自定义标记存储

你可能需要实现自己的存储来在标记更新时执行一些额外的逻辑。例如，你可能对某些工作流的标记存储有特殊需求。为此，你需要实现 `MarkingStoreInterface`：

```php
namespace App\Workflow\MarkingStore;

use Symfony\Component\Workflow\Marking;
use Symfony\Component\Workflow\MarkingStore\MarkingStoreInterface;

final class BlogPostMarkingStore implements MarkingStoreInterface
{
    /**
     * @param BlogPost $subject
     */
    public function getMarking(object $subject): Marking
    {
        return new Marking([$subject->getCurrentPlace() => 1]);
    }

    /**
     * @param BlogPost $subject
     */
    public function setMarking(object $subject, Marking $marking, array $context = []): void
    {
        $marking = key($marking->getPlaces());
        $subject->setCurrentPlace($marking);
    }
}
```

实现标记存储后，你可以配置工作流来使用它：

```yaml
# config/packages/workflow.yaml
framework:
    workflows:
        blog_publishing:
            # ...
            marking_store:
                service: 'App\Workflow\MarkingStore\BlogPostMarkingStore'
```

```php
// config/packages/workflow.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Workflow\MarkingStore\BlogPostMarkingStore;

return App::config([
    'framework' => [
        'workflows' => [
            'blog_publishing' => [
                'marking_store' => [
                    'service' => BlogPostMarkingStore::class,
                ],
            ],
        ],
    ],
]);
```

## 在 Twig 中使用

Symfony 定义了几个 Twig 函数来管理工作流，减少模板中对领域逻辑的需求：

`workflow_can(object $subject, string $transitionName, ?string $name = null)`
    如果给定对象可以进行给定的转换，则返回 `true`。

`workflow_transitions(object $subject, ?string $name = null)`
    返回包含给定对象所有已启用转换的数组。

`workflow_transition(object $subject, string $transition, ?string $name = null)`
    返回给定对象和转换名称的特定已启用转换。

`workflow_marked_places(object $subject, bool $placesNameOnly = true, ?string $name = null)`
    返回包含给定标记的位置名称的数组。

`workflow_has_marked_place(object $subject, string $placeName, ?string $name = null)`
    如果给定对象的标记具有给定状态，则返回 `true`。

`workflow_transition_blockers(object $subject, string $transitionName, ?string $name = null)`
    返回给定转换的 `TransitionBlockerList`。

所有工作流函数都接受一个可选的 `name` 参数（工作流名称）作为最后一个参数。只有当对象与多个工作流关联时才需要此参数：

```html+twig
<h3>Actions on Blog Post</h3>
{% if workflow_can(post, 'publish', 'blog_publishing') %}
    <a href="...">Publish</a>
{% endif %}
{% if workflow_can(post, 'to_review') %}
    <a href="...">Submit to review</a>
{% endif %}
{% if workflow_can(post, 'reject') %}
    <a href="...">Reject</a>
{% endif %}

{# Or loop through the enabled transitions #}
{% for transition in workflow_transitions(post) %}
    <a href="...">{{ transition.name }}</a>
{% else %}
    No actions available.
{% endfor %}

{# Check if the object is in some specific place #}
{% if workflow_has_marked_place(post, 'reviewed') %}
    <p>This post is ready for review.</p>
{% endif %}

{# Check if some place has been marked on the object #}
{% if 'reviewed' in workflow_marked_places(post) %}
    <span class="label">Reviewed</span>
{% endif %}

{# Loop through the transition blockers #}
{% for blocker in workflow_transition_blockers(post, 'publish') %}
    <span class="error">{{ blocker.message }}</span>
{% endfor %}
```

## 存储元数据

如果需要，你可以使用 `metadata` 选项在工作流、其位置和转换中存储任意元数据。这些元数据可以只是工作流的标题，也可以是非常复杂的对象：

```yaml
# config/packages/workflow.yaml
framework:
    workflows:
        blog_publishing:
            metadata:
                title: 'Blog Publishing Workflow'
            # ...
            places:
                draft:
                    metadata:
                        max_num_of_words: 500
                # ...
            transitions:
                to_review:
                    from: draft
                    to:   review
                    metadata:
                        priority: 0.5
                publish:
                    from: reviewed
                    to:   published
                    metadata:
                        hour_limit: 20
                        explanation: 'You can not publish after 8 PM.'
```

```php
// config/packages/workflow.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'workflows' => [
            'blog_publishing' => [
                // ... previous configuration
                'metadata' => [
                    'title' => 'Blog Publishing Workflow'
                ],
                // ...
                'places' => [
                    [
                        'name' => 'draft',
                        'metadata' => [
                            'max_num_of_words' => 500,
                        ],
                    ],
                    // ...
                ],
                'transitions' => [
                    'to_review' => [
                        'from' => 'draft',
                        'to' => 'reviewed',
                        'metadata' => [
                            'priority' => 0.5,
                        ],
                    ],
                    'publish' => [
                        'from' => 'reviewed',
                        'to' => 'published',
                        'metadata' => [
                            'hour_limit' => 20,
                            'explanation' => 'You can not publish after 8 PM.',
                        ],
                    ],
                ],
            ],
        ],
    ],
]);
```

然后你可以在控制器中访问此元数据：

```php
// src/App/Controller/BlogPostController.php
use App\Entity\BlogPost;
use Symfony\Component\Workflow\WorkflowInterface;
// ...

public function myAction(WorkflowInterface $blogPublishingWorkflow, BlogPost $post): Response
{
    $title = $blogPublishingWorkflow
        ->getMetadataStore()
        ->getWorkflowMetadata()['title'] ?? 'Default title'
    ;

    $maxNumOfWords = $blogPublishingWorkflow
        ->getMetadataStore()
        ->getPlaceMetadata('draft')['max_num_of_words'] ?? 500
    ;

    $aTransition = $blogPublishingWorkflow->getDefinition()->getTransitions()[0];
    $priority = $blogPublishingWorkflow
        ->getMetadataStore()
        ->getTransitionMetadata($aTransition)['priority'] ?? 0
    ;

    // ...
}
```

有一个 `getMetadata()` 方法可以处理所有类型的元数据：

```php
// get "workflow metadata" passing the metadata key as argument
$title = $workflow->getMetadataStore()->getMetadata('title');

// get "place metadata" passing the metadata key as the first argument and the place name as the second argument
$maxNumOfWords = $workflow->getMetadataStore()->getMetadata('max_num_of_words', 'draft');

// get "transition metadata" passing the metadata key as the first argument and a Transition object as the second argument
$priority = $workflow->getMetadataStore()->getMetadata('priority', $aTransition);
```

在控制器中的闪现消息中：

```php
// $transition = ...; (an instance of Transition)

// $workflow is an injected Workflow instance
$title = $workflow->getMetadataStore()->getMetadata('title', $transition);
$this->addFlash('info', "You have successfully applied the transition with title: '$title'");
```

元数据也可以在监听器中，通过 `Event` 对象访问。

在 Twig 模板中，元数据可通过 `workflow_metadata()` 函数获取：

```html+twig
<h2>Metadata of Blog Post</h2>
<p>
    <strong>Workflow</strong>:<br>
    <code>{{ workflow_metadata(blog_post, 'title') }}</code>
</p>
<p>
    <strong>Current place(s)</strong>
    <ul>
        {% for place in workflow_marked_places(blog_post) %}
            <li>
                {{ place }}:
                <code>{{ workflow_metadata(blog_post, 'max_num_of_words', place) ?: 'Unlimited'}}</code>
            </li>
        {% endfor %}
    </ul>
</p>
<p>
    <strong>Enabled transition(s)</strong>
    <ul>
        {% for transition in workflow_transitions(blog_post) %}
            <li>
                {{ transition.name }}:
                <code>{{ workflow_metadata(blog_post, 'priority', transition) ?: 0 }}</code>
            </li>
        {% endfor %}
    </ul>
</p>
<p>
    <strong>to_review Priority</strong>
    <ul>
        <li>
            to_review:
            <code>{{ workflow_metadata(blog_post, 'priority', workflow_transition(blog_post, 'to_review')) }}</code>
        </li>
    </ul>
</p>
```

## 验证工作流定义

Symfony 允许你使用自己的自定义逻辑来验证工作流定义。为此，创建一个实现 `DefinitionValidatorInterface` 的类：

```php
namespace App\Workflow\Validator;

use Symfony\Component\Workflow\Definition;
use Symfony\Component\Workflow\Exception\InvalidDefinitionException;
use Symfony\Component\Workflow\Validator\DefinitionValidatorInterface;

final class BlogPublishingValidator implements DefinitionValidatorInterface
{
    public function validate(Definition $definition, string $name): void
    {
        if (!$definition->getMetadataStore()->getMetadata('title')) {
            throw new InvalidDefinitionException(sprintf('The workflow metadata title is missing in Workflow "%s".', $name));
        }

        // ...
    }
}
```

实现验证器后，配置你的工作流来使用它：

```yaml
# config/packages/workflow.yaml
framework:
    workflows:
        blog_publishing:
            # ...

            definition_validators:
                - App\Workflow\Validator\BlogPublishingValidator
```

```php
// config/packages/workflow.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Workflow\Validator\BlogPublishingValidator;

return App::config([
    'framework' => [
        'workflows' => [
            'blog_publishing' => [
                // ...
                'definition_validators' => [
                    BlogPublishingValidator::class,
                ],
            ],
        ],
    ],
]);
```

`BlogPublishingValidator` 将在容器编译期间执行，以验证工作流定义。

## 延伸阅读

* [工作流与状态机](../workflow/workflow-and-state-machine.md)
* [导出工作流](../workflow/dumping-workflows.md)
