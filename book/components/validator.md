# Validator 组件

Validator 组件提供了按照 [JSR-303 Bean Validation 规范][JSR-303 Bean Validation specification]验证值的工具。

## 安装

```terminal
$ composer require symfony/validator
```

## 使用

> **参见：** 本文介绍了如何在任意 PHP 应用程序中将 Validator 功能作为独立组件使用。请阅读[验证](validation.md)文章，了解如何在 Symfony 应用程序中验证数据和实体。

Validator 组件的行为基于两个概念：

* 约束（Constraints），用于定义需要验证的规则；
* 验证器（Validators），包含实际验证逻辑的类。

以下示例展示了如何验证一个字符串是否至少有 10 个字符：

```php
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Validation;

$validator = Validation::createValidator();
$violations = $validator->validate('Bernhard', [
    new Assert\Length(min: 10),
    new Assert\NotBlank(),
]);

if (0 !== count($violations)) {
    // 存在错误，现在可以显示它们
    foreach ($violations as $violation) {
        echo $violation->getMessage().'<br>';
    }
}
```

`validate()` 方法以实现 `Symfony\Component\Validator\ConstraintViolationListInterface` 的对象形式返回违规列表。如果有大量验证错误，你可以按错误代码进行过滤：

```php
use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

$violations = $validator->validate(/* ... */);
if (0 !== count($violations->findByCodes(UniqueEntity::NOT_UNIQUE_ERROR))) {
    // 处理此特定错误（显示某些消息、发送电子邮件等）
}
```

## 获取验证器实例

Validator 对象（实现 `Symfony\Component\Validator\Validator\ValidatorInterface`）是 Validator 组件的主要访问点。要创建其新实例，建议使用 `Symfony\Component\Validator\Validation` 类：

```php
use Symfony\Component\Validator\Validation;

$validator = Validation::createValidator();
```

此 `$validator` 对象可以验证字符串、数字和数组等简单变量，但无法验证对象。若要验证对象，请按照后续章节中的说明配置 `Validator`。

## 了解更多

[JSR-303 Bean Validation specification]: https://jcp.org/en/jsr/detail?id=303
