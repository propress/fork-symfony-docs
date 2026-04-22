# 加载资源

加载器从不同的来源（如 YAML 文件）填充应用的配置。Config 组件定义了此类加载器的接口。[依赖注入](../dependency_injection.md)和 [Routing][routing] 组件附带了针对不同文件格式的专用加载器。

## 定位资源

加载配置通常从搜索资源（主要是文件）开始。这可以通过 `Symfony\Component\Config\FileLocator` 来完成：

```php
use Symfony\Component\Config\FileLocator;

$configDirectories = [__DIR__.'/config'];

$fileLocator = new FileLocator($configDirectories);
$yamlUserFiles = $fileLocator->locate('users.yaml', null, false);
```

定位器接收一个位置集合，它应该在其中查找文件。`locate()` 的第一个参数是要查找的文件名。第二个参数可以是当前路径，当提供时，定位器将首先在此目录中查找。第三个参数指示定位器是否应返回它找到的第一个文件或包含所有匹配项的数组。

## 资源加载器

对于每种类型的资源（YAML、XML、属性等），必须定义一个加载器。每个加载器都应该实现 `Symfony\Component\Config\Loader\LoaderInterface` 或扩展抽象 `Symfony\Component\Config\Loader\FileLoader` 类，该类允许递归导入其他资源：

```php
namespace Acme\Config\Loader;

use Symfony\Component\Config\Loader\FileLoader;
use Symfony\Component\Yaml\Yaml;

class YamlUserLoader extends FileLoader
{
    public function load($resource, $type = null): void
    {
        $configValues = Yaml::parse(file_get_contents($resource));

        // ... 处理配置值

        // 也许导入其他资源：

        // $this->import('extra_users.yaml');
    }

    public function supports($resource, $type = null): bool
    {
        return is_string($resource) && 'yaml' === pathinfo(
            $resource,
            PATHINFO_EXTENSION
        );
    }
}
```

## 找到正确的加载器

`Symfony\Component\Config\Loader\LoaderResolver` 在其第一个构造函数参数中接收加载器的集合。当应加载资源（例如 XML 文件）时，它会循环遍历此加载器集合，并返回支持此特定资源类型的加载器。

`Symfony\Component\Config\Loader\DelegatingLoader` 使用 `Symfony\Component\Config\Loader\LoaderResolver`。当要求它加载资源时，它将此问题委托给 `Symfony\Component\Config\Loader\LoaderResolver`。如果解析器找到了合适的加载器，则将要求此加载器加载资源：

```php
use Acme\Config\Loader\YamlUserLoader;
use Symfony\Component\Config\Loader\DelegatingLoader;
use Symfony\Component\Config\Loader\LoaderResolver;

$loaderResolver = new LoaderResolver([new YamlUserLoader($fileLocator)]);
$delegatingLoader = new DelegatingLoader($loaderResolver);

// YamlUserLoader 用于加载此资源，因为它支持扩展名为 '.yaml' 的文件
$delegatingLoader->load(__DIR__.'/users.yaml');
```

[routing]: https://github.com/symfony/routing
