# 控制器

控制器是你创建的 PHP 函数，它从 `Request` 对象中读取信息，并创建和返回 `Response` 对象。响应可以是 HTML 页面、JSON、XML、文件下载、重定向、404 错误或任何其他内容。控制器运行*你的应用*渲染页面内容所需的任意逻辑。

> **提示**
>
> 如果你还没有创建第一个页面，请先查看[创建页面](page_creation.md)，然后再回来！

---

## 基本控制器

虽然控制器可以是任何 PHP 可调用对象（函数、对象上的方法或 `Closure`），但控制器通常是控制器类中的方法：

```php
// src/Controller/LuckyController.php
namespace App\Controller;

use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class LuckyController
{
    #[Route('/lucky/number/{max}', name: 'app_lucky_number')]
    public function number(int $max): Response
    {
        $number = random_int(0, $max);

        return new Response(
            '<html><body>Lucky number: '.$number.'</body></html>'
        );
    }
}
```

控制器是 `number()` 方法，它位于控制器类 `LuckyController` 中。

这个控制器非常简单：

- *第 2 行*：Symfony 利用 PHP 的命名空间功能为整个控制器类设置命名空间。
- *第 4 行*：Symfony 再次利用 PHP 的命名空间功能：`use` 关键字导入了控制器必须返回的 `Response` 类。
- *第 7 行*：该类可以从技术上叫任何名称，但按惯例以 `Controller` 为后缀。
- *第 10 行*：由于路由中的 `{max}` [通配符](routing.md)，Action 方法允许有一个 `$max` 参数。
- *第 14 行*：控制器创建并返回一个 `Response` 对象。

### 将 URL 映射到控制器

要*查看*此控制器的结果，需要通过路由将 URL 映射到它。上面通过 `#[Route('/lucky/number/{max}')]` [路由属性](routing.md#属性路由)完成了这一操作。

要查看你的页面，请在浏览器中访问此 URL：http://localhost:8000/lucky/number/100

更多路由信息，请参阅[路由](routing.md)。

---

## 基础控制器类与服务 {#the-base-controller-classes-services}

为了帮助开发，Symfony 提供了一个可选的基础控制器类，称为 `AbstractController`。可以继承它以获得辅助方法的访问权限。

在控制器类顶部添加 `use` 语句，然后修改 `LuckyController` 以继承它：

```diff
  // src/Controller/LuckyController.php
  namespace App\Controller;

+ use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;

- class LuckyController
+ class LuckyController extends AbstractController
  {
      // ...
  }
```

就这样！现在你可以访问 `$this->render()` 等方法以及接下来你将了解的许多其他方法。

### 生成 URL

`generateUrl()` 方法只是为给定路由生成 URL 的辅助方法：

```php
$url = $this->generateUrl('app_lucky_number', ['max' => 10]);
```

### 重定向 {#controller-redirect}

如果你想将用户重定向到另一个页面，请使用 `redirectToRoute()` 和 `redirect()` 方法：

```php
use Symfony\Component\HttpFoundation\RedirectResponse;
use Symfony\Component\HttpFoundation\Response;

// ...
public function index(): RedirectResponse
{
    // 重定向到 "homepage" 路由
    return $this->redirectToRoute('homepage');

    // redirectToRoute 是以下的快捷方式：
    // return new RedirectResponse($this->generateUrl('homepage'));

    // 执行永久 HTTP 301 重定向
    return $this->redirectToRoute('homepage', [], 301);
    // 如果你愿意，可以使用 PHP 常量代替硬编码的数字
    return $this->redirectToRoute('homepage', [], Response::HTTP_MOVED_PERMANENTLY);

    // 重定向到带参数的路由
    return $this->redirectToRoute('app_lucky_number', ['max' => 10]);

    // 重定向到路由并保持原始查询字符串参数
    return $this->redirectToRoute('blog_show', $request->query->all());

    // 重定向到当前路由（例如用于 Post/Redirect/Get 模式）：
    return $this->redirectToRoute($request->attributes->get('_route'));

    // 外部重定向
    return $this->redirect('http://symfony.com/doc');
}
```

> **危险**
>
> `redirect()` 方法不会以任何方式检查其目标。如果你重定向到终端用户提供的 URL，你的应用可能面临[未经验证的重定向安全漏洞](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html)。

### 渲染模板 {#controller-rendering-templates}

如果你提供 HTML，你需要渲染模板。`render()` 方法渲染模板**并**将该内容放入 `Response` 对象：

```php
// 渲染 templates/lucky/number.html.twig
return $this->render('lucky/number.html.twig', ['number' => $number]);
```

模板和 Twig 在[创建和使用模板](templates.md)文章中有更多说明。

### 获取服务 {#controller-accessing-services}

Symfony 内置了大量有用的类和功能，称为[服务](service_container.md)。这些服务用于渲染模板、发送电子邮件、查询数据库以及你能想到的任何其他"工作"。

如果你的控制器中需要某个服务，用其类（或接口）名称类型提示一个参数，Symfony 会自动注入它。这要求你的[控制器注册为服务](controller/service.md)：

```php
use Psr\Log\LoggerInterface;
use Symfony\Component\HttpFoundation\Response;
// ...

#[Route('/lucky/number/{max}')]
public function number(int $max, LoggerInterface $logger): Response
{
    $logger->info('We are logging!');
    // ...
}
```

你还可以使用哪些其他服务？使用 `debug:autowiring` 控制台命令查看：

```terminal
$ php bin/console debug:autowiring
```

> **提示**
>
> 如果需要精确控制参数的值，或需要一个参数，可以使用 `#[Autowire]` 属性：
>
> ```php
> // ...
> use Psr\Log\LoggerInterface;
> use Symfony\Component\DependencyInjection\Attribute\Autowire;
> use Symfony\Component\HttpFoundation\Response;
>
> class LuckyController extends AbstractController
> {
>     public function number(
>         int $max,
>
>         // 注入特定的日志服务
>         #[Autowire(service: 'monolog.logger.request')]
>         LoggerInterface $logger,
>
>         // 或注入参数值
>         #[Autowire('%kernel.project_dir%')]
>         string $projectDir
>     ): Response
>     {
>         $logger->info('We are logging!');
>         // ...
>     }
> }
> ```

与所有服务一样，你也可以在控制器中使用常规的[构造函数注入](service_container.md#构造函数注入)。

更多关于服务的信息，请参阅[服务容器](service_container.md)文章。

---

## 生成控制器

为了节省时间，你可以安装 [Symfony Maker](https://symfony.com/doc/current/bundles/SymfonyMakerBundle/index.html) 并告诉 Symfony 生成一个新的控制器类：

```terminal
$ php bin/console make:controller BrandNewController

created: src/Controller/BrandNewController.php
created: templates/brandnew/index.html.twig
```

如果你想从 Doctrine [实体](doctrine.md)生成完整的 CRUD，请使用：

```terminal
$ php bin/console make:crud Product

created: src/Controller/ProductController.php
created: src/Form/ProductType.php
created: templates/product/_delete_form.html.twig
created: templates/product/_form.html.twig
created: templates/product/edit.html.twig
created: templates/product/index.html.twig
created: templates/product/new.html.twig
created: templates/product/show.html.twig
```

---

## 管理错误和 404 页面

当找不到内容时，你应该返回 404 响应。为此，抛出一个特殊类型的异常：

```php
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

// ...
public function index(): Response
{
    // 从数据库获取对象
    $product = ...;
    if (!$product) {
        throw $this->createNotFoundException('The product does not exist');

        // 以上只是以下的快捷方式：
        // throw new NotFoundHttpException('The product does not exist');
    }

    return $this->render(/* ... */);
}
```

`createNotFoundException()` 方法只是创建一个特殊的 `NotFoundHttpException` 对象的快捷方式，该对象最终在 Symfony 内部触发 404 HTTP 响应。

如果你抛出的异常继承自或是 `HttpException` 的实例，Symfony 将使用适当的 HTTP 状态码。否则，响应将具有 500 HTTP 状态码：

```php
// 此异常最终生成 500 状态错误
throw new \Exception('Something went wrong!');
```

在每种情况下，都会向最终用户显示错误页面，并向开发者显示完整的调试错误页面（即当你处于"Debug"模式时——参阅[配置环境](configuration.md#配置环境)）。

要自定义向用户显示的错误页面，请参阅[错误页面](controller/error_pages.md)文章。

---

## Request 对象作为控制器参数 {#controller-request-argument}

如果你需要读取查询参数、获取请求头或访问上传的文件怎么办？这些信息存储在 Symfony 的 `Request` 对象中。要在控制器中访问它，添加一个参数并**用 Request 类对其进行类型提示**：

```php
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
// ...

public function index(Request $request): Response
{
    $page = $request->query->get('page', 1);

    // ...
}
```

[继续阅读](#request-和-response-对象)以了解更多关于使用 Request 对象的信息。

---

## 自动映射请求 {#controller_map-request}

可以使用属性将请求的载荷和/或查询参数自动映射到控制器的 Action 参数。

### 逐个映射查询参数

假设用户向你发送带有以下查询字符串的请求：`https://example.com/dashboard?firstName=John&lastName=Smith&age=27`。借助 `#[MapQueryParameter]` 属性，控制器 Action 的参数可以自动填充：

```php
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;

// ...

public function dashboard(
    #[MapQueryParameter] string $firstName,
    #[MapQueryParameter] string $lastName,
    #[MapQueryParameter] int $age,
): Response
{
    // ...
}
```

`MapQueryParameter` 属性支持以下参数类型：

- `\BackedEnum`
- `array`
- `bool`
- `float`
- `int`
- `string`
- 继承 `AbstractUid` 的对象

`#[MapQueryParameter]` 可以接受一个可选参数 `filter`，可以使用 PHP 定义的[验证过滤器](https://www.php.net/manual/en/filter.constants.php)常量：

```php
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;

// ...

public function dashboard(
    #[MapQueryParameter(filter: \FILTER_VALIDATE_REGEXP, options: ['regexp' => '/^\w+$/'])] string $firstName,
    #[MapQueryParameter] string $lastName,
    #[MapQueryParameter(filter: \FILTER_VALIDATE_INT)] int $age,
): Response
{
    // ...
}
```

### 映射整个查询字符串 {#controller-mapping-query-string}

另一种可能是将整个查询字符串映射到一个包含可用查询参数的对象。假设你声明了以下带有可选验证约束的 DTO：

```php
namespace App\Model;

use Symfony\Component\Validator\Constraints as Assert;

class UserDto
{
    public function __construct(
        #[Assert\NotBlank]
        public string $firstName,

        #[Assert\NotBlank]
        public string $lastName,

        #[Assert\GreaterThan(18)]
        public int $age,
    ) {
    }
}
```

然后你可以在控制器中使用 `#[MapQueryString]` 属性：

```php
use App\Model\UserDto;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Attribute\MapQueryString;

// ...

public function dashboard(
    #[MapQueryString] UserDto $userDto
): Response
{
    // ...
}
```

你可以自定义映射期间使用的验证组，以及验证失败时返回的 HTTP 状态码：

```php
use Symfony\Component\HttpFoundation\Response;

// ...

public function dashboard(
    #[MapQueryString(
        validationGroups: ['strict', 'edit'],
        validationFailedStatusCode: Response::HTTP_UNPROCESSABLE_ENTITY
    )] UserDto $userDto
): Response
{
    // ...
}
```

验证失败时默认返回的状态码为 404。

如果你想使用特定键将对象映射到查询中的嵌套数组，请在 `#[MapQueryString]` 属性中设置 `key` 选项：

```php
use App\Model\SearchDto;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Attribute\MapQueryString;

// ...

public function dashboard(
    #[MapQueryString(key: 'search')] SearchDto $searchDto
): Response
{
    // ...
}
```

如果即使请求查询字符串为空也需要有效的 DTO，请为控制器参数设置默认值：

```php
use App\Model\UserDto;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Attribute\MapQueryString;

// ...

public function dashboard(
    #[MapQueryString] UserDto $userDto = new UserDto()
): Response
{
    // ...
}
```

### 映射请求载荷 {#controller-mapping-request-payload}

创建 API 并处理 `GET` 以外的其他 HTTP 方法（如 `POST` 或 `PUT`）时，用户数据不存储在查询字符串中，而是直接存储在请求载荷中：

```json
{
    "firstName": "John",
    "lastName": "Smith",
    "age": 28
}
```

在这种情况下，也可以使用 `#[MapRequestPayload]` 属性将此载荷直接映射到你的 DTO：

```php
use App\Model\UserDto;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Attribute\MapRequestPayload;

// ...

public function dashboard(
    #[MapRequestPayload] UserDto $userDto
): Response
{
    // ...
}
```

此属性允许你自定义序列化上下文以及负责在请求和 DTO 之间进行映射的类：

```php
public function dashboard(
    #[MapRequestPayload(
        serializationContext: ['...'],
        resolver: App\Resolver\UserDtoResolver
    )]
    UserDto $userDto
): Response
{
    // ...
}
```

你还可以自定义使用的验证组、验证失败时返回的状态码以及支持的载荷格式：

```php
use Symfony\Component\HttpFoundation\Response;

// ...

public function dashboard(
    #[MapRequestPayload(
        acceptFormat: 'json',
        validationGroups: ['strict', 'read'],
        validationFailedStatusCode: Response::HTTP_NOT_FOUND
    )] UserDto $userDto
): Response
{
    // ...
}
```

验证失败时默认返回的状态码为 422。

> **提示**
>
> 如果你构建 JSON API，请确保将路由声明为使用 JSON [格式](routing.md#格式参数)。这将使错误处理在验证错误时输出 JSON 响应，而不是 HTML 页面：
>
> ```php
> #[Route('/dashboard', name: 'dashboard', format: 'json')]
> ```

### 映射上传的文件 {#controller_map-uploaded-file}

Symfony 提供了一个名为 `#[MapUploadedFile]` 的属性，用于将一个或多个 `UploadedFile` 对象映射到控制器参数：

```php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\File\UploadedFile;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Attribute\MapUploadedFile;
use Symfony\Component\Routing\Attribute\Route;

class UserController extends AbstractController
{
    #[Route('/user/picture', methods: ['PUT'])]
    public function changePicture(
        #[MapUploadedFile] UploadedFile $picture,
    ): Response {
        // ...
    }
}
```

在此示例中，关联的[参数解析器](controller/value_resolver.md)根据参数名称（`$picture`）获取 `UploadedFile`。如果未提交文件，则抛出 `HttpException`。你可以通过使控制器参数可为空来改变这一行为：

```php
#[MapUploadedFile]
?UploadedFile $document
```

`#[MapUploadedFile]` 属性还允许你传递要应用于上传文件的约束列表：

```php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\File\UploadedFile;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Attribute\MapUploadedFile;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Validator\Constraints as Assert;

class UserController extends AbstractController
{
    #[Route('/user/picture', methods: ['PUT'])]
    public function changePicture(
        #[MapUploadedFile([
            new Assert\File(mimeTypes: ['image/png', 'image/jpeg']),
            new Assert\Image(maxWidth: 3840, maxHeight: 2160),
        ])]
        UploadedFile $picture,
    ): Response {
        // ...
    }
}
```

验证约束在将 `UploadedFile` 注入控制器参数之前进行检查。如果存在约束违规，则抛出 `HttpException`，控制器的 Action 不会执行。

如果需要上传文件集合，将它们映射到数组或可变参数。给定的约束将应用于所有文件，如果任何文件失败，则抛出 `HttpException`：

```php
#[MapUploadedFile(new Assert\File(mimeTypes: ['application/pdf']))]
array $documents

#[MapUploadedFile(new Assert\File(mimeTypes: ['application/pdf']))]
UploadedFile ...$documents
```

使用 `name` 选项将上传的文件重命名为自定义值：

```php
#[MapUploadedFile(name: 'something-else')]
UploadedFile $document
```

此外，你可以更改存在约束违规时抛出的 HTTP 异常的状态码：

```php
#[MapUploadedFile(
    constraints: new Assert\File(maxSize: '2M'),
    validationFailedStatusCode: Response::HTTP_REQUEST_ENTITY_TOO_LARGE
)]
UploadedFile $document
```

---

## 管理会话

Symfony 提供了一个会话服务，用于在请求之间存储用户信息。你可以通过 `Request` 对象访问会话（在服务中，[注入 RequestStack 服务](service_container/request.md)）：

```php
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

public function index(Request $request): Response
{
    $session = $request->getSession();

    // 存储属性以在稍后的用户请求中复用
    $session->set('user_id', 42);

    // 使用可选默认值检索属性
    $userId = $session->get('user_id', 0);

    // ...
}
```

阅读[会话文档](session.md)以了解更多关于配置和使用会话的详细信息。

### 快闪消息

快闪消息是只使用一次的特殊会话消息：一旦你获取它们，它们就会自动从会话中消失。这使得它们非常适合存储用户通知：

```php
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
// ...

public function update(Request $request): Response
{
    // ... 做一些数据处理

    $this->addFlash('notice', 'Your changes were saved!');
    // $this->addFlash() 等同于 $request->getSession()->getFlashBag()->add()

    return $this->redirectToRoute(/* ... */);
}
```

---

## Request 和 Response 对象 {#request-object-info}

如前所述，Symfony 会将 `Request` 对象传递给任何用 `Request` 类进行类型提示的控制器参数：

```php
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

public function index(Request $request): Response
{
    $request->isXmlHttpRequest(); // 这是 Ajax 请求吗？

    $request->getPreferredLanguage(['en', 'fr']);

    // 分别获取 GET 和 POST 变量
    $request->query->get('page');
    $request->getPayload()->get('page');

    // 获取 SERVER 变量
    $request->server->get('HTTP_HOST');

    // 获取由 foo 标识的 UploadedFile 实例
    $request->files->get('foo');

    // 获取 COOKIE 值
    $request->cookies->get('PHPSESSID');

    // 获取 HTTP 请求头，键为标准化的小写形式
    $request->headers->get('host');
    $request->headers->get('content-type');
}
```

`Request` 类有几个公共属性和方法，可以返回你需要的任何请求信息。

与 `Request` 一样，`Response` 对象也有一个公共 `headers` 属性。此对象类型为 `ResponseHeaderBag`，提供获取和设置响应头的方法。头名称是标准化的，因此名称 `Content-Type` 等同于名称 `content-type` 或 `content_type`。

在 Symfony 中，控制器需要返回一个 `Response` 对象：

```php
use Symfony\Component\HttpFoundation\Response;

// 创建一个带有 200 状态码（默认）的简单响应
$response = new Response('Hello '.$name, Response::HTTP_OK);

// 创建带有 200 状态码的 CSS 响应
$response = new Response('<style> ... </style>');
$response->headers->set('Content-Type', 'text/css');
```

### 访问配置值

要从控制器获取任何[配置参数](configuration.md#配置参数)的值，请使用 `getParameter()` 辅助方法：

```php
// ...
public function index(): Response
{
    $contentsDir = $this->getParameter('kernel.project_dir').'/contents';
    // ...
}
```

### 返回 JSON 响应

要从控制器返回 JSON，使用 `json()` 辅助方法。这将返回一个自动编码数据的 `JsonResponse` 对象：

```php
use Symfony\Component\HttpFoundation\JsonResponse;
// ...

public function index(): JsonResponse
{
    // 返回 '{"username":"jane.doe"}' 并设置正确的 Content-Type 头
    return $this->json(['username' => 'jane.doe']);

    // 快捷方式定义了三个可选参数
    // return $this->json($data, $status = 200, $headers = [], $context = []);
}
```

如果应用中启用了[序列化器服务](serializer.md)，它将用于将数据序列化为 JSON。否则，使用 `json_encode` 函数。

### 流式传输文件响应

你可以使用 `file()` 辅助方法从控制器内部提供文件：

```php
use Symfony\Component\HttpFoundation\BinaryFileResponse;
// ...

public function download(): BinaryFileResponse
{
    // 发送文件内容并强制浏览器下载
    return $this->file('/path/to/some_file.pdf');
}
```

`file()` 辅助方法提供了一些参数来配置其行为：

```php
use Symfony\Component\HttpFoundation\File\File;
use Symfony\Component\HttpFoundation\ResponseHeaderBag;
// ...

public function download(): BinaryFileResponse
{
    // 从文件系统加载文件
    $file = new File('/path/to/some_file.pdf');

    return $this->file($file);

    // 重命名下载的文件
    return $this->file($file, 'custom_name.pdf');

    // 在浏览器中显示文件内容而不是下载
    return $this->file('invoice_3241.pdf', 'my_invoice.pdf', ResponseHeaderBag::DISPOSITION_INLINE);
}
```

### 发送早期提示

你可以通过发送 `103` 早期提示响应来提高性能，让浏览器在完整响应准备好之前开始下载资源。详情参阅[早期提示](http_cache.md#早期提示)。

### 流式传输服务器发送事件 {#controller-server-sent-events}

[服务器发送事件（SSE）](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)是一种标准，允许服务器通过单个 HTTP 连接向客户端推送更新。它提供了一种高效的方式，将实时更新从服务器发送到浏览器，例如实时通知、进度更新或数据流。

`EventStreamResponse` 类允许你使用 SSE 协议向客户端流式传输事件。它自动设置所需的头（`Content-Type: text/event-stream`、`Cache-Control: no-cache`、`Connection: keep-alive`）并提供发送事件的 API：

```php
use Symfony\Component\HttpFoundation\EventStreamResponse;
use Symfony\Component\HttpFoundation\ServerEvent;

// ...

public function liveNotifications(): EventStreamResponse
{
    return new EventStreamResponse(function (): iterable {
        foreach ($this->getNotifications() as $notification) {
            yield new ServerEvent($notification->toJson());

            sleep(1); // 模拟事件之间的延迟
        }
    });
}
```

`ServerEvent` 类是一个 DTO，表示遵循 [WHATWG SSE 规范](https://html.spec.whatwg.org/multipage/server-sent-events.html)的 SSE 事件。你可以使用其构造函数参数自定义每个事件：

```php
// 仅包含数据的基本事件
yield new ServerEvent('Some message');

// 带有自定义类型的事件（客户端通过 addEventListener('my-event', ...) 监听）
yield new ServerEvent(
    data: json_encode(['status' => 'completed']),
    type: 'my-event'
);

// 带有 ID 的事件（对于使用 Last-Event-ID 头恢复流很有用）
yield new ServerEvent(
    data: 'Update content',
    id: 'event-123'
);

// 告诉客户端在特定时间后重试的事件（毫秒）
yield new ServerEvent(
    data: 'Retry info',
    retry: 5000
);

// 带有注释的事件（可用于保持连接）
yield new ServerEvent(comment: 'keep-alive');
```

对于生成器不实用的用例，你可以使用 `EventStreamResponse::sendEvent` 方法进行手动控制：

```php
use Symfony\Component\HttpFoundation\EventStreamResponse;
use Symfony\Component\HttpFoundation\ServerEvent;

// ...

public function liveProgress(): EventStreamResponse
{
    return new EventStreamResponse(function (EventStreamResponse $response): void {
        $redis = new \Redis();
        $redis->connect('127.0.0.1');
        $redis->subscribe(['message'], function (/* ... */, string $message) use ($response): void {
            $response->sendEvent(new ServerEvent($message));
        });
    });
}
```

在客户端，你可以使用原生 `EventSource` API 监听事件：

```javascript
const eventSource = new EventSource('/live-notifications');

// 监听所有事件（没有特定类型）
eventSource.onmessage = (event) => {
    console.log('Received:', event.data);
};

// 监听特定类型的事件
eventSource.addEventListener('my-event', (event) => {
    console.log('My event:', JSON.parse(event.data));
});

// 处理连接错误
eventSource.onerror = (error) => {
    console.error('SSE error:', error);
    eventSource.close();
};
```

> **警告**
>
> `EventStreamResponse` 是为并发连接有限的应用设计的。由于 SSE 保持 HTTP 连接打开，它为每个连接的客户端消耗服务器资源（内存和连接限制）。
>
> 对于需要同时向许多客户端广播更新的高流量应用，请考虑使用 [Mercure](mercure.md)，它建立在 SSE 之上，但使用专用的 hub 来高效管理连接。

---

## 将控制器与 Symfony 解耦

继承 [AbstractController 基类](#基础控制器类与服务)可以简化控制器开发，**对大多数应用都推荐**。但是，某些高级用户更希望将控制器与 Symfony 完全解耦（例如，为了提高可测试性或遵循更框架无关的设计）。Symfony 提供了工具来帮助你做到这一点。

为了解耦控制器，Symfony 通过另一个名为 `ControllerHelper` 的类公开了 `AbstractController` 中的所有辅助方法，其中每个辅助方法都作为公共方法可用：

```php
use Symfony\Bundle\FrameworkBundle\Controller\ControllerHelper;
use Symfony\Component\DependencyInjection\Attribute\AutowireMethodOf;
use Symfony\Component\HttpFoundation\Response;

class MyController
{
    public function __construct(
        #[AutowireMethodOf(ControllerHelper::class)]
        private \Closure $render,
        #[AutowireMethodOf(ControllerHelper::class)]
        private \Closure $redirectToRoute,
    ) {
    }

    public function showProduct(int $id): Response
    {
        if (!$id) {
            return ($this->redirectToRoute)('product_list');
        }

        return ($this->render)('product/show.html.twig', ['product_id' => $id]);
    }
}
```

如果你愿意，可以注入整个 `ControllerHelper` 类，但像上面示例那样使用 `#[AutowireMethodOf]` 属性，只注入你需要的确切辅助方法，使代码更高效。

由于 `#[AutowireMethodOf]` 也适用于接口，你可以为这些辅助方法定义接口：

```php
interface RenderInterface
{
    // 这是 render() 辅助方法的签名
    public function __invoke(string $view, array $parameters = [], ?Response $response = null): Response;
}
```

然后，更新你的控制器以使用接口而不是闭包：

```php
use Symfony\Bundle\FrameworkBundle\Controller\ControllerHelper;
use Symfony\Component\DependencyInjection\Attribute\AutowireMethodOf;

class MyController
{
    public function __construct(
        #[AutowireMethodOf(ControllerHelper::class)]
        private RenderInterface $render,
    ) {
    }

    // ...
}
```

像上面示例那样使用接口提供了完整的静态分析和自动补全优势，而无需额外的样板代码。

---

## 总结

在 Symfony 中，控制器通常是一个类方法，用于接受请求并返回 `Response` 对象。当映射到 URL 时，控制器变得可访问，其响应可以被查看。

为了便于控制器开发，Symfony 提供了 `AbstractController`。可以使用它扩展控制器类，以访问一些常用工具，如 `render()` 和 `redirectToRoute()`。`AbstractController` 还提供了 `createNotFoundException()` 工具，用于返回页面未找到响应。

在其他文章中，你将学习如何在控制器内部使用特定服务，这些服务将帮助你从数据库中持久化和获取对象、处理表单提交、处理缓存等。

---

## 继续学习！

接下来，学习所有关于[使用 Twig 渲染模板](templates.md)的内容。

## 延伸阅读

- [将控制器注册为服务](controller/service.md)
- [错误页面](controller/error_pages.md)
- [上传文件](controller/upload_file.md)
- [内置 Symfony 控制器类的参数解析器](controller/value_resolver.md)
- [CSRF 保护](security/csrf.md)
- [控制器中的参数](controller/argument_value_resolver.md)
