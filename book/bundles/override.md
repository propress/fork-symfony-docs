# 如何覆盖 Bundle 的任意部分

当您使用第三方 bundle 时，可能希望自定义或覆盖其中的某些功能。本文介绍了覆盖 bundle
中最常见功能的几种方式。

## 模板

第三方 bundle 的模板可以在
``<your-project>/templates/bundles/<bundle-name>/`` 目录中覆盖。新模板必须与原模板
使用相同的名称和路径（相对于 ``<bundle>/templates/``）。

例如，要覆盖 ``AcmeUserBundle`` 中的
``templates/registration/confirmed.html.twig`` 模板，请创建以下模板：
``<your-project>/templates/bundles/AcmeUserBundle/registration/confirmed.html.twig``

> [!WARNING]
> 如果您在一个新位置添加模板，即使处于 debug 模式，也**可能**需要清除缓存
> （``php bin/console cache:clear``）。

有时您不想覆盖整个模板，而只想覆盖一个或多个 block。不过，因为您正在覆盖自己想要继承
的模板，所以会陷入无限循环错误。解决方案是在模板名称前使用特殊的 ``!`` 前缀，告诉
Symfony：您要继承的是原始模板，而不是已经被覆盖的模板：

```twig
{# templates/bundles/AcmeUserBundle/registration/confirmed.html.twig #}
{# 特殊的 '!' 前缀可避免从已覆盖模板继承时出现错误 #}
{% extends "@!AcmeUser/registration/confirmed.html.twig" %}

{% block some_block %}
    ...
{% endblock %}
```

> [!TIP]
> Symfony 内部自身也使用了一些 bundle，因此您也可以用同样的技术覆盖 Symfony
> 核心模板。例如，您可以通过覆盖 TwigBundle 模板来
> [自定义错误页](../controller/error_pages.md)。

## 路由

Symfony 永远不会自动导入路由（routing）。如果您想包含某个 bundle 的路由，
就必须在应用的某处手动导入它们（例如 ``config/routes.yaml``）。

“覆盖”某个 bundle 路由的最简单方式，是根本不要导入它。与其导入第三方 bundle 的路由，
不如把那份路由文件复制到应用中，按需修改后，再导入您自己的副本。

## 控制器

如果控制器本身是服务，请参阅下一节了解如何覆盖它。否则，请定义一个新的路由和控制器，
并让它们使用与目标控制器相同的路径（同时确保这个新路由会在 bundle 的路由之前加载）。

## 服务与配置

如果您想修改某个 bundle 创建的服务，可以使用
[服务装饰（service decoration）](../service_container/service_decoration.md)。

如果您要做更高级的操作，比如删除其他 bundle 创建的服务，就必须在
[compiler pass](../service_container/compiler_passes.md) 中处理
[服务定义](../service_container/definitions.md)。

## 实体与实体映射

只有当某个 bundle 提供了 mapped superclass
（例如 FOSUserBundle 中的 ``User`` 实体）时，才有可能覆盖实体映射。通过这种方式，
您可以覆盖 attributes 和 associations。有关该特性及其限制的更多信息，请参阅
[Doctrine 文档][doctrine-documentation]。

## 表单

现有的表单类型（form types）可以通过定义
[表单类型扩展](../form/create_form_type_extension.md) 来修改。

## 验证元数据

Symfony 会加载每个 bundle 中的所有验证配置文件，并将它们合并为一棵验证元数据树。
这意味着您可以为某个属性新增约束（constraints），但不能覆盖已有约束。

要解决这个问题，第三方 bundle 需要提供
[验证组（validation groups）](../validation/groups.md) 相关配置。
例如 FOSUserBundle 就提供了这类配置。要创建您自己的验证规则，请把约束添加到一个新的
验证组中：

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

现在，请更新 FOSUserBundle 的配置，让它使用您的验证组，而不是原始验证组。

> 📝 译者注：这里的关键点不是“替换”原有约束，而是通过 validation groups 把应用自身
> 的验证逻辑接入到第三方 bundle 的验证流程中。

## 翻译

翻译（translations）与 bundle 无关，而与翻译 domain 有关。因此，只要新文件使用相同
domain，您就可以在主 ``translations/`` 目录中覆盖任意 bundle 的翻译文件。

例如，要覆盖 ``AcmeUserBundle`` 的
``translations/AcmeUserBundle.es.yaml`` 中定义的翻译，请创建
``<your-project>/translations/AcmeUserBundle.es.yaml`` 文件。

[doctrine-documentation]: https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/inheritance-mapping.html#overrides
