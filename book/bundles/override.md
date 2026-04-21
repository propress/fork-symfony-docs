# 如何覆盖 Bundle 的任意部分

在使用第三方 Bundle 时，你可能希望自定义或覆盖其某些功能。本文介绍覆盖 Bundle 最常见功能的方法。

## 模板

第三方 Bundle 的模板可以在 `<your-project>/templates/bundles/<bundle-name>/` 目录中被覆盖。新模板必须使用与原始模板相同的名称和路径（相对于 `<bundle>/templates/`）。

例如，要覆盖 AcmeUserBundle 中的 `templates/registration/confirmed.html.twig` 模板，请创建以下模板：
`<your-project>/templates/bundles/AcmeUserBundle/registration/confirmed.html.twig`

> **警告：**
> 如果你在新位置添加了模板，即使处于调试模式，你*也可能*需要清除缓存（`php bin/console cache:clear`）。

你可能只想覆盖一个或多个区块，而不是覆盖整个模板。但是，由于你要覆盖的是你想要继承的模板，这会导致无限循环错误。解决方法是在模板名称中使用特殊的 `!` 前缀，告诉 Symfony 你想从原始模板继承，而不是从被覆盖的模板继承：

```twig
{# templates/bundles/AcmeUserBundle/registration/confirmed.html.twig #}
{# 特殊的 '!' 前缀可避免从被覆盖的模板继承时出现错误 #}
{% extends "@!AcmeUser/registration/confirmed.html.twig" %}

{% block some_block %}
    ...
{% endblock %}
```

> **提示：**
> Symfony 内部也使用了一些 Bundle，因此你可以对核心 Symfony 模板应用同样的技术。例如，你可以通过覆盖 TwigBundle 模板来[自定义错误页面](../controller/error_pages.md)。

## 路由

路由在 Symfony 中不会自动导入。如果你想包含任何 Bundle 的路由，则必须从应用的某处手动导入它们（例如 `config/routes.yaml`）。

"覆盖" Bundle 路由最简单的方法是根本不导入它。与其导入第三方 Bundle 的路由，不如将该路由文件复制到你的应用中，根据需要进行修改，然后导入你的副本。

## 控制器

如果控制器是一个服务，请参阅下一节关于如何覆盖它的内容。否则，定义一个新的路由 + 控制器，使其路径与你想覆盖的控制器相同（并确保新路由在 Bundle 路由之前加载）。

## 服务与配置

如果你想修改 Bundle 创建的服务，可以使用[服务装饰](../service_container/service_decoration.md)。

如果你想进行更高级的操作，例如删除其他 Bundle 创建的服务，你必须在[编译器传递](../service_container/compiler_passes.md)中使用[服务定义](../service_container/definitions.md)。

## 实体与实体映射

只有当 Bundle 提供映射的超类（例如 FOSUserBundle 中的 `User` 实体）时，才能覆盖实体映射。以这种方式覆盖属性和关联是可能的。在 [Doctrine 文档](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/inheritance-mapping.html#overrides) 中了解更多关于此特性及其限制的内容。

## 表单

现有的表单类型可以通过定义[表单类型扩展](../form/create_form_type_extension.md)来修改。

## 验证元数据

Symfony 从每个 Bundle 加载所有验证配置文件，并将它们合并为一个验证元数据树。这意味着你可以向属性添加新约束，但无法覆盖已有的约束。

要解决此问题，第三方 Bundle 需要提供[验证组](../validation/groups.md)的配置。例如，FOSUserBundle 具有此配置。要创建你自己的验证，请将约束添加到新的验证组：

```yaml
# config/validator/validation.yaml
FOS\UserBundle\Model\User:
    properties:
        plainPassword:
            - NotBlank:
                groups: [AcmeValidation]
            - Length:
                min: 6
                minMessage: fos_user.password.short
                groups: [AcmeValidation]
```

```xml
<!-- config/validator/validation.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<constraint-mapping xmlns="http://symfony.com/schema/dic/constraint-mapping"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/constraint-mapping
        https://symfony.com/schema/dic/constraint-mapping/constraint-mapping-1.0.xsd"
>
    <class name="FOS\UserBundle\Model\User">
        <property name="plainPassword">
            <constraint name="NotBlank">
                <option name="groups">
                    <value>AcmeValidation</value>
                </option>
            </constraint>

            <constraint name="Length">
                <option name="min">6</option>
                <option name="minMessage">fos_user.password.short</option>
                <option name="groups">
                    <value>AcmeValidation</value>
                </option>
            </constraint>
        </property>
    </class>
</constraint-mapping>
```

现在，更新 FOSUserBundle 的配置，使其使用你的验证组而不是原来的验证组。

## 翻译

翻译与 Bundle 无关，而是与翻译域相关。因此，只要新文件使用相同的域，你就可以从主 `translations/` 目录覆盖任何 Bundle 的翻译文件。

例如，要覆盖 AcmeUserBundle 的 `translations/AcmeUserBundle.es.yaml` 文件中定义的翻译，请创建 `<your-project>/translations/AcmeUserBundle.es.yaml` 文件。
