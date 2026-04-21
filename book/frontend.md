# 前端工具：处理 CSS 和 JavaScript

Symfony 让你可以灵活选择任何你想要的前端工具。通常有两种方式：

1. [使用 PHP 和 Twig 构建 HTML](#使用-php--twig)；
2. [使用 JavaScript 框架构建前端](#使用前端框架reactvuesvelte-等)（如 React、Vue、Svelte 等）。

两种方式都很好用——下面将进行讨论。

---

## 使用 PHP & Twig {#frontend-twig-php}

Symfony 提供了两个强大的选项来帮助你构建现代且快速的前端：

- **AssetMapper**（推荐用于新项目）：完全在 PHP 中运行，不需要任何构建步骤，并利用现代 Web 标准。
- **Webpack Encore**：基于 [Node.js](https://nodejs.org/) 在 [Webpack](https://webpack.js.org/) 之上构建。

|  | AssetMapper | Encore |
|--|-------------|--------|
| 生产就绪？ | 是 | 是 |
| 稳定？ | 是 | 是 |
| 依赖 | 无 | Node.js |
| 需要构建步骤？ | 否 | 是 |
| 所有浏览器兼容？ | 是 | 是 |
| 支持 Stimulus/UX | 是 | 是 |
| 支持 Sass/Tailwind | 是 | 是 |
| 支持 React、Vue、Svelte？ | 是 [1] | 是 |
| 支持 TypeScript | 是 | 是 |
| 从 JavaScript 中删除注释 | 否 [2] | 是 |
| 从 CSS 中删除注释 | 否 [2] | 是 [4] |
| 版本化资产 | 始终 | 可选 |
| 可更新第三方包 | 是 | 否 [3] |

**[1]** 使用 JSX（React）、Vue 等与 AssetMapper 是可能的，但你需要使用它们的原生工具进行预编译。此外，某些功能（如 Vue 单文件组件）无法编译为可由浏览器执行的纯 JavaScript。

**[2]** 你可以安装 [SensioLabs Minify Bundle](https://github.com/sensiolabs/minify-bundle) 在使用 AssetMapper 编译资产时压缩 CSS/JS 代码（并删除所有注释）。

**[3]** 如果你使用 `npm`，有可用的更新检查器（例如 `npm-check`）。

**[4]** 可以使用 [CssMinimizerPlugin](https://webpack.js.org/plugins/css-minimizer-webpack-plugin) 删除 CSS 注释，该插件包含在 Webpack Encore 中，可通过 `Encore.configureCssMinimizerPlugin()` 配置。

### AssetMapper（推荐）{#frontend-asset-mapper}

> 更喜欢视频教程？请查看 [AssetMapper 视频系列](https://symfonycasts.com/screencast/asset-mapper)。

AssetMapper 是处理资产的推荐系统。它完全在 PHP 中运行，没有复杂的构建步骤或依赖。它通过利用浏览器的 `importmap` 功能来实现这一点，该功能在所有浏览器中都可以通过 polyfill 使用。

[阅读 AssetMapper 文档](frontend/asset_mapper.md)

### Webpack Encore {#frontend-webpack-encore}

> 更喜欢视频教程？请查看 [Webpack Encore 视频系列](https://symfonycasts.com/screencast/webpack-encore)。

[Webpack Encore](https://www.npmjs.com/package/@symfony/webpack-encore) 是将 [Webpack](https://webpack.js.org/) 集成到应用中的更简单方法。它包装了 Webpack，为你提供了一个干净而强大的 API，用于打包 JavaScript 模块、预处理 CSS 和 JS 以及编译和压缩资产。

[阅读 Encore 文档](frontend/encore/index.md)

#### 从 AssetMapper 切换

默认情况下，新的 Symfony Web 应用项目（使用 `symfony new --webapp myapp` 创建）使用 AssetMapper。如果你仍然需要使用 Webpack Encore，请使用以下步骤切换。这最好在新项目上进行，并提供与默认 Web 应用相同的功能（Turbo/Stimulus）：

```terminal
# 临时删除 AssetMapper & Turbo/Stimulus
$ composer remove symfony/ux-turbo symfony/asset-mapper symfony/stimulus-bundle

# 重新添加 Webpack Encore & Turbo/Stimulus
$ composer require symfony/webpack-encore-bundle symfony/ux-turbo symfony/stimulus-bundle

# 安装并构建资产
$ npm install
$ npm run dev
```

### Stimulus 和 Symfony UX 组件

一旦你安装了 AssetMapper 或 Webpack Encore，就可以开始构建你的前端了。你可以用任何你想要的方式编写 JavaScript，但我们推荐使用 [Stimulus](https://stimulus.hotwired.dev/)、[Turbo](https://turbo.hotwired.dev/) 和一套称为 [Symfony UX](https://ux.symfony.com) 的工具。

要了解 Stimulus 和 UX 组件，请参阅 [StimulusBundle 文档](https://symfony.com/bundles/StimulusBundle/current/index.html)。

---

## 使用前端框架（React、Vue、Svelte 等）{#frontend-js}

> 更喜欢视频教程？请查看 [API Platform 视频系列](https://symfonycasts.com/screencast/api-platform)。

如果你想使用前端框架（Next.js、React、Vue、Svelte 等），我们建议使用它们的原生工具，并将 Symfony 用作纯 API。[API Platform](https://api-platform.com/) 是一个很好的工具。它的标准发行版附带了一个由 Symfony 驱动的 API 后端，Next.js 中的前端脚手架（也支持其他框架）和 React 管理界面。它完全 Dockerized，甚至包含一个 Web 服务器。

---

## 其他前端文章

- [创建 UX Bundle](frontend/create_ux_bundle.md)
- [自定义版本策略](frontend/custom_version_strategy.md)
- [服务器数据](frontend/server-data.md)
