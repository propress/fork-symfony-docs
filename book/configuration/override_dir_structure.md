# 如何覆盖 Symfony 的默认目录结构

Symfony 应用具有以下默认目录结构，但你可以覆盖它以创建自己的结构：

```text
your-project/
├─ assets/
├─ bin/
│  └─ console
├─ config/
├─ public/
│  └─ index.php
├─ src/
│  └─ ...
├─ templates/
├─ tests/
├─ translations/
├─ var/
│  ├─ cache/
│  ├─ log/
│  └─ ...
├─ vendor/
└─ .env
```

## 覆盖环境（DotEnv）文件目录

默认情况下，[.env 配置文件](../configuration.md#.env-文件)位于项目的根目录。如果你将其存储在不同的位置，请在 `composer.json` 文件中定义 `runtime.dotenv_path` 选项：

```json
{
    "...": "...",
    "extra": {
        "...": "...",
        "runtime": {
            "dotenv_path": "my/custom/path/to/.env"
        }
    }
}
```

然后，更新你的 Composer 文件（例如运行 `composer dump-autoload`），以便使用新的 `.env` 路径重新生成 `vendor/autoload_runtime.php` 文件。

你还可以为 console 和 Web 服务器调用设置不同的 `.env` 路径。编辑 `public/index.php` 和/或 `bin/console` 文件以定义新的文件路径。

Console 脚本：

```php
// bin/console

// ...
$_SERVER['APP_RUNTIME_OPTIONS']['dotenv_path'] = 'some/custom/path/to/.env';

require_once dirname(__DIR__).'/vendor/autoload_runtime.php';
// ...
```

Web 前端控制器：

```php
// public/index.php

// ...
$_SERVER['APP_RUNTIME_OPTIONS']['dotenv_path'] = 'another/custom/path/to/.env';

require_once dirname(__DIR__).'/vendor/autoload_runtime.php';
// ...
```

## 覆盖二进制文件目录

你可以通过在 `composer.json` 文件中添加 `extra.bin-dir` 选项来更改二进制文件目录：

```json
{
    "...": "...",
    "extra": {
        "...": "...",
        "bin-dir": "my_new_bin_dir"
    }
}
```

## 覆盖配置目录

你可以通过在 `composer.json` 文件中添加 `extra.config-dir` 选项来更改配置目录：

```json
{
    "...": "...",
    "extra": {
        "...": "...",
        "config-dir": "my_new_config_dir"
    }
}
```

## 覆盖缓存目录

更改缓存目录可以通过在应用的 `Kernel` 类中覆盖 `getCacheDir()` 方法来实现：

```php
// src/Kernel.php

// ...
class Kernel extends BaseKernel
{
    // ...

    public function getCacheDir(): string
    {
        return dirname(__DIR__).'/var/'.$this->environment.'/cache';
    }
}
```

在此代码中，`$this->environment` 是当前环境（即 `dev`）。在这种情况下，你已将缓存目录的位置更改为 `var/{environment}/cache/`。

你也可以通过定义名为 `APP_CACHE_DIR` 的环境变量来更改缓存目录，其值为缓存文件夹的完整路径。

> **警告：** 你应该为每个环境保持不同的缓存目录，否则可能会发生一些意外行为。每个环境生成自己的缓存配置文件，因此每个环境都需要自己的目录来存储这些缓存文件。

如果你有多个前端服务器使用同一个共享文件系统，可以使用 `Symfony\Component\HttpKernel\Kernel::getShareDir` 方法获取用于缓存和共享数据的共享目录。可以通过覆盖名为 `APP_SHARE_DIR` 的环境变量来设置共享目录，其值为共享文件夹的完整路径。该目录也可作为名为 `%kernel.share_dir%` 的容器参数访问。

## 覆盖日志目录

覆盖 `var/log/` 目录与覆盖 `var/cache/` 目录几乎相同。

你可以通过在应用的 `Kernel` 类中覆盖 `getLogDir()` 方法来实现：

```php
// src/Kernel.php

// ...
class Kernel extends BaseKernel
{
    // ...

    public function getLogDir(): string
    {
        return dirname(__DIR__).'/var/'.$this->environment.'/log';
    }
}
```

这里你已将目录的位置更改为 `var/{environment}/log/`。

你也可以通过定义名为 `APP_LOG_DIR` 的环境变量来更改日志目录，其值为日志文件夹的完整路径。

## 覆盖源文件目录

你可以通过在 `composer.json` 文件中添加 `extra.src-dir` 选项并更新 `autoload.psr-4` 选项来更改源文件目录：

```json
{
    "...": "...",
    "autoload": {
        "psr-4": {
            "App\\": "my_new_src_dir/"
        }
    },
    "extra": {
        "...": "...",
        "src-dir": "my_new_src_dir"
    }
}
```

> **提示：** 更改 `autoload.psr-4` 后，不要忘记运行 `composer dump-autoload` 命令。

## 覆盖模板目录

如果你的模板不存储在默认的 `templates/` 目录中，请使用 `twig.default_path` 配置选项定义自己的模板目录（对于多个目录，使用 `twig.paths`）：

```yaml
# config/packages/twig.yaml
twig:
    default_path: "%kernel.project_dir%/resources/views"
```

```php
// config/packages/twig.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'twig' => [
        'default_path' => '%kernel.project_dir%/resources/views',
    ],
]);
```

## 覆盖翻译目录

如果你的翻译文件不存储在默认的 `translations/` 目录中，请使用 `framework.translator.default_path` 配置选项定义自己的翻译目录（对于多个目录，使用 `framework.translator.paths`）：

```yaml
# config/packages/translation.yaml
framework:
    translator:
        # ...
        default_path: "%kernel.project_dir%/i18n"
```

```php
// config/packages/translation.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'translator' => [
            'default_path' => '%kernel.project_dir%/i18n',
        ],
    ],
]);
```

## 覆盖公共目录

如果你需要重命名或移动 `public/` 目录，唯一需要保证的是 `index.php` 前端控制器中指向 `vendor/` 目录的路径仍然正确。如果你重命名了目录，一切都没问题。但如果你以某种方式移动了它，可能需要修改这些文件中的路径：

```php
require_once __DIR__.'/../path/to/vendor/autoload_runtime.php';
```

你还需要在 `composer.json` 文件中更改 `extra.public-dir` 选项：

```json
{
    "...": "...",
    "extra": {
        "...": "...",
        "public-dir": "my_new_public_dir"
    }
}
```

> **提示：** 某些共享主机有一个 `public_html/` Web 目录根目录。将 Web 目录从 `public/` 重命名为 `public_html/` 是让你的 Symfony 项目在共享主机上工作的一种方式。另一种方式是将应用部署到 Web 根目录之外的目录，删除 `public_html/` 目录，然后用指向项目中 `public/` 目录的符号链接替换它。

## 覆盖 Vendor 目录

要覆盖 `vendor/` 目录，你需要在 `composer.json` 文件中定义 `vendor-dir` 选项，如下所示：

```json
{
    "config": {
        "bin-dir": "bin",
        "vendor-dir": "/some/dir/vendor"
    }
}
```

> **提示：** 如果你在虚拟环境中工作且无法使用 NFS，此修改可能会很有用——例如，在 guest 操作系统中使用 Vagrant/VirtualBox 运行 Symfony 应用时。
