# 安装与配置 Symfony 框架

> **提示：** 更喜欢视频教程？请查看 [Cosmic Coding with Symfony](https://symfonycasts.com/screencast/symfony) 系列视频课程。

## 技术要求

在创建第一个 Symfony 应用之前，你必须：

* 安装 PHP 8.4 或更高版本，并启用以下 PHP 扩展（这些扩展在大多数 PHP 8 安装中默认已安装并启用）：`Ctype`、`iconv`、`PCRE`、`Session`、`SimpleXML` 和 `Tokenizer`；
* [安装 Composer](https://getcomposer.org/download/)，用于安装 PHP 包。

此外，还可以[安装 Symfony CLI](https://symfony.com/download)。这是可选的，但它提供了一个名为 `symfony` 的二进制工具，包含了本地开发和运行 Symfony 应用所需的所有工具。

`symfony` 二进制工具还提供了一个命令来检查你的计算机是否满足所有要求。打开终端并运行以下命令：

```terminal
$ symfony check:requirements
```

> **注意：** Symfony CLI 是开源的，你可以在 [symfony-cli/symfony-cli GitHub 仓库](https://github.com/symfony-cli/symfony-cli)中参与贡献。

## 创建 Symfony 应用

打开终端，运行以下任一命令来创建新的 Symfony 应用：

```terminal
# 如果你在构建传统的 Web 应用，运行此命令
$ symfony new my_project_directory --version="8.0.*" --webapp

# 如果你在构建微服务、控制台应用或 API，运行此命令
$ symfony new my_project_directory --version="8.0.*"
```

这两个命令的唯一区别是默认安装的包数量。`--webapp` 选项会安装额外的包，为你提供构建 Web 应用所需的一切。

如果你没有使用 Symfony 二进制工具，可以使用 Composer 运行以下命令来创建新的 Symfony 应用：

```terminal
# 如果你在构建传统的 Web 应用，运行此命令
$ composer create-project symfony/skeleton:"8.0.*" my_project_directory
$ cd my_project_directory
$ composer require webapp

# 如果你在构建微服务、控制台应用或 API，运行此命令
$ composer create-project symfony/skeleton:"8.0.*" my_project_directory
```

无论你运行哪个命令来创建 Symfony 应用，它们都会创建一个新的 `my_project_directory/` 目录，下载一些依赖项，甚至生成你开始所需的基本目录和文件。换句话说，你的新应用已经准备好了！

> **注意：** 项目的缓存和日志目录（默认为 `<project>/var/cache/` 和 `<project>/var/log/`）必须对 Web 服务器可写。如果遇到问题，请阅读如何[为 Symfony 应用设置权限](/setup/file_permissions)。

## 配置现有 Symfony 项目

除了创建新的 Symfony 项目外，你还会参与其他开发者已创建的项目。在这种情况下，你只需要获取项目代码并使用 Composer 安装依赖项。假设你的团队使用 Git，使用以下命令来设置项目：

```terminal
# 克隆项目以下载其内容
$ cd projects/
$ git clone ...

# 让 Composer 将项目的依赖项安装到 vendor/ 中
$ cd my-project/
$ composer install
```

你可能还需要自定义 [.env 文件](/configuration#config-dot-env) 并执行一些项目特定的任务（例如创建数据库）。在第一次使用现有 Symfony 应用时，运行此命令可能会很有帮助，它会显示项目的相关信息：

```terminal
$ php bin/console about
```

## 运行 Symfony 应用

在生产环境中，你应该安装 Nginx 或 Apache 等 Web 服务器，并[配置它来运行 Symfony](/setup/web_server_configuration)。如果你不使用 Symfony 本地 Web 服务器进行开发，也可以使用此方法。

然而，在本地开发中，运行 Symfony 最便捷的方式是使用 Symfony CLI 工具提供的[本地 Web 服务器](/setup/symfony_server)。该本地服务器支持 HTTP/2、并发请求、TLS/SSL 以及自动生成安全证书等功能。

打开终端，进入新项目目录，按如下方式启动本地 Web 服务器：

```terminal
$ cd my-project/
$ symfony server:start
```

打开浏览器并访问 `http://localhost:8000/`。如果一切正常，你将看到一个欢迎页面。完成工作后，在终端按 `Ctrl+C` 停止服务器。

> **提示：** 该 Web 服务器适用于任何 PHP 应用，而不仅仅是 Symfony 项目，因此它是一个非常有用的通用开发工具。

### Symfony Docker 集成

如果你想在 Symfony 中使用 Docker，请参阅 [/setup/docker](/setup/docker)。

## 安装包

开发 Symfony 应用时，一个常见做法是安装提供开箱即用功能的包（Symfony 称之为 [bundles](/bundles)）。包在使用之前通常需要一些配置（编辑某些文件以启用 bundle、创建某些文件以添加初始配置等）。

大多数情况下，这些配置可以自动完成，这就是为什么 Symfony 包含了 [Symfony Flex](https://github.com/symfony/flex)——一个简化 Symfony 应用中包的安装/移除的工具。从技术上讲，Symfony Flex 是一个 Composer 插件，在创建新 Symfony 应用时默认安装，它**自动化了 Symfony 应用中最常见的任务**。

> **提示：** 你也可以[将 Symfony Flex 添加到现有项目](/setup/flex)。

Symfony Flex 修改了 `require`、`update` 和 `remove` Composer 命令的行为以提供高级功能。考虑以下示例：

```terminal
$ cd my-project/
$ composer require logger
```

如果你在没有使用 Flex 的 Symfony 应用中运行该命令，你会看到 Composer 错误，提示 `logger` 不是一个有效的包名。但是，如果应用安装了 Symfony Flex，该命令将安装并启用使用官方 Symfony 日志记录器所需的所有包。

之所以如此，是因为许多 Symfony 包/bundle 定义了**"配方（recipes）"**，这是一套自动化的指令，用于安装并启用 Symfony 应用中的包。Flex 会在 `symfony.lock` 文件中跟踪已安装的配方，该文件必须提交到你的代码仓库。

Symfony Flex 配方由社区贡献，存储在两个公共仓库中：

* [主配方仓库](https://github.com/symfony/recipes)，是高质量且维护良好的包的精选配方列表。Symfony Flex 默认只在此仓库中查找。

* [社区配方仓库](https://github.com/symfony/recipes-contrib)，包含所有社区创建的配方。所有配方都保证可以工作，但其关联的包可能没有维护。Symfony Flex 在安装任何这些配方之前会征求你的许可。

阅读 [Symfony 配方文档](https://github.com/symfony/recipes/blob/master/README.rst)以了解如何为你自己的包创建配方。

### Symfony Packs

有时，单个功能需要安装多个包和 bundle。为此，Symfony 提供了 **packs**，它们是包含多个依赖项的 Composer 元包。

例如，要在应用中添加调试功能，你可以运行 `composer require --dev debug` 命令。这会安装 `symfony/debug-pack`，进而安装多个包，如 `symfony/debug-bundle`、`symfony/monolog-bundle`、`symfony/var-dumper` 等。

你不会在 `composer.json` 中看到 `symfony/debug-pack` 依赖项，因为 Flex 会自动解包。这意味着它只将真正的包添加为依赖项（例如，你会在 `require-dev` 中看到新的 `symfony/var-dumper`）。

## 检查安全漏洞

安装 [Symfony CLI](#setup-symfony-cli) 时创建的 `symfony` 二进制工具提供了一个命令，用于检查项目的依赖项是否包含已知的安全漏洞：

```terminal
$ symfony check:security
```

定期执行此命令是一个良好的安全实践，以便尽快更新或替换受影响的依赖项。安全检查通过获取公共 [PHP 安全公告数据库](https://github.com/FriendsOfPHP/security-advisories)在本地完成，因此你的 `composer.lock` 文件不会通过网络发送。

`check:security` 命令在任何依赖项受已知安全漏洞影响时，以非零退出代码终止。这样你就可以将其添加到项目构建流程和持续集成工作流中，以便在存在漏洞时使它们失败。

> **提示：** 在持续集成服务中，你可以通过运行 `composer audit` 命令来检查安全漏洞。它在内部使用与 `check:security` 相同的数据，但不需要在 CI 或 CI 工作器上安装整个 Symfony CLI。

## Symfony LTS 版本

根据 [Symfony 发布流程](/contributing/community/releases)，"长期支持"（简称 LTS）版本每两年发布一次。查看 [Symfony 版本](https://symfony.com/releases)以了解最新的 LTS 版本。

默认情况下，创建新 Symfony 应用的命令使用最新的稳定版本。如果你想使用 LTS 版本，添加 `--version` 选项：

```terminal
# 使用最新的 LTS 版本
$ symfony new my_project_directory --version=lts

# 使用即将发布的"下一个" Symfony 版本（仍在开发中）
$ symfony new my_project_directory --version=next

# 你也可以选择一个精确的特定 Symfony 版本
$ symfony new my_project_directory --version="7.4.*"
```

`lts` 和 `next` 快捷方式只在使用 Symfony 创建新项目时可用。如果你使用 Composer，需要指定确切的版本：

```terminal
$ composer create-project symfony/skeleton:"7.4.*" my_project_directory
```

## Symfony Demo 应用

[Symfony Demo 应用](https://github.com/symfony/demo)是一个功能完整的应用，展示了开发 Symfony 应用的推荐方式。它是 Symfony 新手的极佳学习工具，其代码包含大量注释和有用说明。

运行此命令来创建基于 Symfony Demo 应用的新项目：

```terminal
$ symfony new my_project_directory --demo
```

## 开始编码！

配置完成后，是时候[在 Symfony 中创建你的第一个页面](/page_creation)了。

## 深入学习

* [setup/docker](/setup/docker)
* [setup/homestead](/setup/homestead)
* [setup/web_server_configuration](/setup/web_server_configuration)
* [setup/*](/setup/)
