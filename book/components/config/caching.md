# 基于资源的缓存

当加载所有配置资源时，您可能希望处理配置值并将它们全部组合到一个文件中。此文件充当缓存。其内容不必在应用每次运行时重新生成 - 仅在修改配置资源时重新生成。

例如，Symfony Routing 组件允许您加载所有路由，然后根据这些路由转储 URL 匹配器或 URL 生成器。在这种情况下，当修改其中一个资源时（并且您在开发环境中工作），生成的文件应该失效并重新生成。这可以通过使用 `Symfony\Component\Config\ConfigCache` 类来实现。

下面的示例向您展示如何收集资源，然后基于加载的资源生成一些代码，并将此代码写入缓存。缓存还接收用于生成代码的资源集合。通过查看这些资源的"最后修改"时间戳，缓存可以判断它是否仍然是最新的或者其内容是否应该重新生成：

```php
use Symfony\Component\Config\ConfigCache;
use Symfony\Component\Config\Resource\FileResource;

$cachePath = __DIR__.'/cache/appUserMatcher.php';

// 第二个参数指示您是否要使用调试模式
$userMatcherCache = new ConfigCache($cachePath, true);

if (!$userMatcherCache->isFresh()) {
    // 用 'users.yaml' 文件路径数组填充此
    $yamlUserFiles = ...;

    $resources = [];

    foreach ($yamlUserFiles as $yamlUserFile) {
        // 参阅"加载资源"文章以了解
        // $delegatingLoader 来自哪里
        $delegatingLoader->load($yamlUserFile);
        $resources[] = new FileResource($yamlUserFile);
    }

    // UserMatcher 的代码在其他地方生成
    $code = ...;

    $userMatcherCache->write($code, $resources);
}

// 您可能想要引入缓存的代码：
require $cachePath;
```

在调试模式下，将在缓存文件本身所在的同一目录中创建一个 `.meta` 文件。此 `.meta` 文件包含序列化的资源，其时间戳用于确定缓存是否仍然是最新的。当不在调试模式下时，一旦缓存存在，就认为它是"最新的"，因此不会生成 `.meta` 文件。

您可以显式定义元文件的绝对路径：

```php
use Symfony\Component\Config\ConfigCache;
use Symfony\Component\Config\Resource\FileResource;

$cachePath = __DIR__.'/cache/appUserMatcher.php';

// 第三个可选参数指示元文件的绝对路径
$userMatcherCache = new ConfigCache($cachePath, true, '/my/absolute/path/to/cache.meta');
```
