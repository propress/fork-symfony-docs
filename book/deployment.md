# 如何部署 Symfony 应用

部署 Symfony 应用可能是一项复杂且多样化的任务，具体取决于你的设置和应用的需求。本文并非分步指南，而是部署中最常见需求和想法的通用列表。

---

## Symfony 部署基础 {#symfony2-deployment-basics}

部署 Symfony 应用时通常需要执行以下步骤：

1. 将代码上传到生产服务器；
2. 安装供应商依赖项（通常通过 Composer 完成，可以在上传之前完成）；
3. 运行数据库迁移或类似任务以更新任何已更改的数据结构；
4. 清除（以及可选地，预热）缓存。

部署还可能包括其他任务，例如：

- 在源代码控制仓库中将特定版本的代码标记为发布版本；
- 创建临时暂存区域以"离线"构建更新后的设置；
- 运行任何可用的测试并[检查 Twig 模板](templates.md)以确保代码和/或服务器稳定性；
- 从 `public/` 目录中删除任何不必要的文件以保持生产环境整洁；
- 清除外部缓存系统（如 [Memcached](https://memcached.org/) 或 [Redis](https://redis.io/)）。

---

## 如何部署 Symfony 应用

有几种方法可以部署 Symfony 应用。从一些基本的部署策略开始，然后在此基础上构建。

### 基本文件传输

部署应用最基本的方式是通过 FTP/SCP（或类似方法）手动复制文件。这有其缺点，因为在升级过程中你缺乏对系统的控制。此方法还需要你在传输文件后执行一些手动步骤（参阅[常见部署任务](#常见部署任务)）。

### 使用源代码控制

如果你使用源代码控制（例如 Git 或 SVN），可以通过将线上安装也作为仓库的副本来简化部署。当你准备好升级时，从源代码控制系统中获取最新更新。使用 Git 时，常见的方法是为每个发布版本创建标签，并在部署时检出适当的标签（参阅 [Git 标签](https://git-scm.com/book/en/v2/Git-Basics-Tagging)）。

这使得更新文件*更容易*，但你仍然需要担心手动执行其他步骤（参阅[常见部署任务](#常见部署任务)）。

### 使用平台即服务（PaaS）

使用平台即服务（PaaS）可以是快速部署 Symfony 应用的好方法。有许多 PaaS，但我们推荐 [Upsun](https://symfony.com/cloud)，因为它提供专用的 Symfony 集成并有助于资助 Symfony 开发。

### 使用构建脚本和其他工具

还有一些工具可以帮助减轻部署的痛苦。其中一些是专门针对 Symfony 的需求量身定制的：

**[Deployer](https://deployer.org/)**
这是 Capistrano 的另一个原生 PHP 重写版本，带有一些适用于 Symfony 的现成配方。

**[Ansistrano](https://ansistrano.com/)**
一个 Ansible 角色，允许你通过 YAML 文件配置强大的部署。

**[Magallanes](https://github.com/andres-montanez/Magallanes)**
这个类似 Capistrano 的部署工具是用 PHP 构建的，对于 PHP 开发人员来说可能更容易根据自己的需求进行扩展。

**[Fabric](https://www.fabfile.org/)**
这个基于 Python 的库提供了一套基本的操作，用于执行本地或远程 shell 命令以及上传/下载文件。

**[Capistrano](https://capistranorb.com/) 与 [Symfony 插件](https://github.com/capistrano/symfony/)**
Capistrano 是一个用 Ruby 编写的远程服务器自动化和部署工具。Symfony 插件是一个简化 Symfony 相关任务的插件，灵感来自 [Capifony](https://github.com/everzet/capifony)（仅适用于 Capistrano 2）。

---

## 常见部署任务 {#common-post-deployment-tasks}

在部署实际源代码之前和之后，有许多常见的事情需要做：

### A) 检查需求

运行 Symfony 应用有一些[技术要求](setup.md#技术要求)。在你的开发机器上，检查这些需求的推荐方式是使用 [Symfony CLI](https://symfony.com/download)。但是，在生产服务器上，你可能不希望安装 Symfony CLI 工具。在这种情况下，在你的应用中安装以下包：

```terminal
$ composer require symfony/requirements-checker
```

然后，确保检查器包含在你的 Composer 脚本中：

```json
{
    "...": "...",

    "scripts": {
        "auto-scripts": {
            "vendor/bin/requirements-checker": "php-script",
            "...": "..."
        },

        "...": "..."
    }
}
```

### B) 配置你的环境变量 {#b-configure-your-app-config-parameters-yml-file}

大多数 Symfony 应用从环境变量中读取配置。在本地开发时，你通常将这些变量存储在 [.env 文件](configuration.md#在-env-文件中配置环境变量)中。在生产环境中，你有两个选择：

1. 创建"真实"的环境变量。如何设置环境变量取决于你的设置：可以在命令行中设置、在 Nginx 配置中设置，或者通过托管服务提供的其他方法设置；

2. 或者，创建一个包含特定于你的生产环境值的 `.env.prod.local` 文件。

两种选择没有显著优势区别：使用最适合你的托管环境的那种。

> **提示**
>
> 你可能不希望你的应用在每次请求时都处理 `.env.*` 文件。你可以生成一个优化的 `.env.local.php` 文件，它覆盖所有其他配置文件：
>
> ```terminal
> $ composer dump-env prod
> ```
>
> 生成的文件将包含存储在 `.env` 中的所有配置。如果你想仅依赖环境变量，请生成一个没有任何值的文件：
>
> ```terminal
> $ composer dump-env prod --empty
> ```
>
> 如果生产服务器上没有安装 Composer，请改用 [dotenv:dump Symfony 命令](configuration.md#在生产环境中配置环境变量)。

### C) 安装/更新供应商

你的供应商可以在传输源代码之前更新（即更新 `vendor/` 目录，然后与源代码一起传输）或之后在服务器上更新。无论哪种方式，像平时一样更新你的供应商：

```terminal
$ composer install --no-dev --optimize-autoloader
```

> **提示**
>
> `--optimize-autoloader` 标志通过构建"类映射"显著提高 Composer 的自动加载器性能。`--no-dev` 标志确保开发包不会安装在生产环境中。

> **警告**
>
> 如果在此步骤中出现"class not found"错误，你可能需要在运行此命令之前运行 `export APP_ENV=prod`（如果你不使用 [Symfony Flex](setup.md#symfony-flex) 则使用 `export SYMFONY_ENV=prod`），以便 `post-install-cmd` 脚本在 `prod` 环境中运行。

### D) 清除 Symfony 缓存

确保清除并预热 Symfony 缓存：

```terminal
$ APP_ENV=prod APP_DEBUG=0 php bin/console cache:clear
```

### E) 其他事项！

根据你的设置，可能还有很多其他事情需要做：

- 运行任何数据库迁移
- 清除 APCu 缓存
- 添加/编辑 CRON 任务
- 重启工作进程
- 使用 Webpack Encore 构建和压缩资源
- 如果你使用 AssetMapper 组件，则编译资源
- 将资源推送到 CDN
- 将错误页面转储为静态 HTML 文件
- 在使用 Apache Web 服务器的共享托管平台上，你可能需要安装 [symfony/apache-pack](https://packagist.org/packages/symfony/apache-pack) 包
- 等等

---

## 应用生命周期：持续集成、QA 等

虽然本文涵盖了部署的技术细节，但从开发到生产的完整代码生命周期可能有更多步骤：部署到暂存环境、QA（质量保证）、运行测试等。

强烈建议使用暂存、测试、QA、持续集成、数据库迁移以及在失败时回滚的能力。有简单和更复杂的工具，可以根据你的环境需求使你的部署尽可能简单（或复杂）。

不要忘记，部署应用还涉及更新任何依赖项（通常通过 Composer）、迁移数据库、清除缓存以及其他潜在事项，如将资源推送到 CDN（参阅[常见部署任务](#常见部署任务)）。

---

## 故障排除

### 不使用 `composer.json` 文件的部署

[项目根目录](configuration.md#内核配置)（其值通过 `kernel.project_dir` 参数和 `Kernel::getProjectDir()` 方法使用）由 Symfony 自动计算为存储主 `composer.json` 文件的目录。

在不使用 `composer.json` 文件的部署中，你需要覆盖 `Kernel::getProjectDir()` 方法，如[该部分](configuration.md#内核配置)所述。

---

## 延伸阅读

- [在反向代理后部署](deployment/proxies.md)
