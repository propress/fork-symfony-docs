# 容器构建工作流程

与 Dependency Injection 组件相关的文件和类的位置取决于您想要使用容器的应用、库或框架。查看在 Symfony 全栈框架中如何配置和构建容器将帮助您了解这一切是如何结合在一起的，无论您是使用全栈框架还是希望在另一个应用中使用服务容器。

全栈框架使用 HttpKernel 组件来管理从应用和 bundle 加载服务容器配置，还处理编译和缓存。即使您不使用 HttpKernel，它也应该为您提供一种在模块化应用中组织配置的方式的思路。

## 使用缓存的容器

在构建它之前，内核会检查容器的缓存版本是否存在。内核有一个调试设置，如果为 false，则使用缓存版本（如果存在）。如果调试为 true，则内核检查配置是否最新，如果是，则使用容器的缓存版本。如果不是，则从应用级配置和 bundle 的扩展配置构建容器。

有关更多详细信息，请阅读转储配置以提高性能。

## 应用级配置

应用级配置从 `config` 目录加载。加载多个文件，然后在处理扩展时合并这些文件。这允许不同环境的不同配置，例如 dev、prod。

这些文件包含直接加载到容器中的参数和服务，如使用配置文件设置容器中所述。它们还包含由扩展处理的配置，如使用扩展管理配置中所述。这些被视为 bundle 配置，因为每个 bundle 都包含一个 Extension 类。

## 使用扩展的 Bundle 级配置

按照惯例，每个 bundle 都包含一个 Extension 类，该类位于 bundle 的 `DependencyInjection` 目录中。当内核启动时，这些会向 `ContainerBuilder` 注册。当 `ContainerBuilder` 被编译时，与 bundle 扩展相关的应用级配置会传递给 Extension，该扩展通常还会加载自己的配置文件，通常来自 bundle 的 `Resources/config` 目录。应用级配置通常使用 Configuration 对象处理，该对象也存储在 bundle 的 `DependencyInjection` 目录中。

## 允许 Bundle 之间交互的编译器传递

编译器传递用于允许不同 bundle 之间的交互，因为它们不能在扩展类中影响彼此的配置。主要用途之一是处理标记的服务，允许 bundle 注册要由其他 bundle 拾取的服务，例如 Monolog 记录器、Twig 扩展和 Web Profiler 的数据收集器。编译器传递通常放置在 bundle 的 `DependencyInjection/Compiler` 目录中。

## 编译和缓存

在编译过程从配置、扩展和编译器传递加载服务后，它被转储，以便下次可以使用缓存。然后在后续请求期间使用转储的版本，因为它更有效。
