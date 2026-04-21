# 如何保护敏感信息的安全

[环境变量](../configuration.md#使用环境变量)是存储依赖于应用运行位置的配置的最佳方式——例如，某个 API 密钥在本地开发时可能设置为一个值，在生产环境中设置为另一个值。

当这些值*敏感*且需要保密时，你可以使用 Symfony 的密钥管理系统（有时称为"vault"）安全地存储它们。

> **注意：** 密钥系统需要 Sodium PHP 扩展。

## 生成加密密钥

为了加密和解密**密钥**，Symfony 需要**加密密钥**。可以通过运行以下命令生成一对密钥：

```terminal
$ php bin/console secrets:generate-keys
```

这将生成一对非对称**加密密钥**。每个[环境](../configuration.md#配置环境)都有自己的一组密钥。假设你在 `dev` 环境中进行本地编码，这将创建：

`config/secrets/dev/dev.encrypt.public.php`
用于加密/向 vault 添加密钥。可以安全地提交。

`config/secrets/dev/dev.decrypt.private.php`
用于解密/从 vault 读取密钥。`dev` 解密密钥可以提交（假设 dev vault 中没有存储高度敏感的密钥），但 `prod` 解密密钥**绝不**应该提交。

你可以通过运行以下命令为 `prod` 环境生成一对加密密钥：

```terminal
$ APP_RUNTIME_ENV=prod php bin/console secrets:generate-keys
```

这将生成 `config/secrets/prod/prod.encrypt.public.php` 和 `config/secrets/prod/prod.decrypt.private.php`。

> **危险：** `prod.decrypt.private.php` 文件高度敏感。你的开发团队甚至持续集成服务都不需要该密钥。如果**解密密钥**已泄露（例如前员工离职），你应该考虑通过运行以下命令生成新密钥：`secrets:generate-keys --rotate`。

## 创建或更新密钥

假设你想将数据库密码存储为密钥。使用 `secrets:set` 命令，你应该将此密钥添加到 `dev` *和* `prod` vault 中：

```terminal
# 出于安全原因，输入时隐藏

# 设置默认开发值（可以在本地覆盖）
$ php bin/console secrets:set DATABASE_PASSWORD

# 设置生产值
$ APP_RUNTIME_ENV=prod php bin/console secrets:set DATABASE_PASSWORD
```

这将在 `config/secrets/dev` 中为密钥创建一个新文件，在 `config/secrets/prod` 中创建另一个文件。你也可以通过其他几种方式设置密钥：

```terminal
# 提供要从中读取密钥的文件
$ php bin/console secrets:set DATABASE_PASSWORD ~/Download/password.json

# 或传递给 STDIN 的内容
$ echo -n "$DB_PASS" | php bin/console secrets:set DATABASE_PASSWORD -

# 或让 Symfony 为你生成随机值
$ php bin/console secrets:set REMEMBER_ME --random
```

> **注意：** 没有重命名密钥的命令，因此你需要创建一个新密钥并删除旧密钥。

## 在配置文件中引用密钥

密钥值可以用与[环境变量](../configuration.md#使用环境变量)相同的方式引用。注意不要意外地使用相同名称定义密钥*和*环境变量：**环境变量覆盖密钥**。

如果你存储了 `DATABASE_PASSWORD` 密钥，可以通过以下方式引用它：

```yaml
# config/packages/doctrine.yaml
doctrine:
    dbal:
        password: '%env(DATABASE_PASSWORD)%'
```

```php
// config/packages/doctrine.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'doctrine' => [
        'dbal' => [
            'password' => env('DATABASE_PASSWORD'),
        ],
    ],
]);
```

实际值将在运行时解析：容器编译和缓存预热不需要**解密密钥**。

## 列出现有密钥

所有人都可以使用 `secrets:list` 命令列出密钥名称。如果你有**解密密钥**，也可以通过传递 `--reveal` 选项来显示密钥的值：

```terminal
$ php bin/console secrets:list --reveal

 ------------------- ------------ -------------
  Name                Value        Local Value
 ------------------- ------------ -------------
  DATABASE_PASSWORD   "my secret"
 ------------------- ------------ -------------
```

## 显示现有密钥

如果你有**解密密钥**，`secrets:reveal` 命令允许你显示单个密钥的值：

```terminal
$ php bin/console secrets:reveal DATABASE_PASSWORD

 my secret
```

## 删除密钥

Symfony 提供了一个方便的命令来删除密钥：

```terminal
$ php bin/console secrets:remove DATABASE_PASSWORD
```

## 本地密钥：在本地覆盖密钥

`dev` 环境密钥应该包含用于开发的良好默认值。但有时开发者在开发时*仍然*需要在本地覆盖密钥值。

大多数 `secrets` 命令——包括 `secrets:set`——都有一个 `--local` 选项，它将"密钥"存储在 `.env.{env}.local` 文件中作为标准环境变量。要在本地覆盖 `DATABASE_PASSWORD` 密钥，请运行：

```terminal
$ php bin/console secrets:set DATABASE_PASSWORD --local
```

如果你输入了 `root`，你现在会在 `.env.dev.local` 文件中看到：

```bash
DATABASE_PASSWORD=root
```

这将*覆盖* `DATABASE_PASSWORD` 密钥，因为环境变量始终优先于密钥。

列出密钥现在也会显示本地变量：

```terminal
$ php bin/console secrets:list --reveal
 ------------------- ------------- -------------
  Name                Value         Local Value
 ------------------- ------------- -------------
  DATABASE_PASSWORD   "dev value"   "root"
 ------------------- ------------- -------------
```

Symfony 还提供了 `secrets:decrypt-to-local` 命令，它解密所有密钥并将它们存储在本地 vault 中，以及 `secrets:encrypt-from-local` 命令，用于将所有本地密钥加密到 vault 中。

## test 环境中的密钥

如果你在 `dev` 和 `prod` 环境中添加密钥，它将在 `test` 环境中缺失。你*可以*为 `test` 环境创建一个"vault"并在那里定义密钥。但一个更简单的方法是通过 `.env.test` 文件设置测试值：

```bash
# .env.test
DATABASE_PASSWORD="testing"
```

## 将密钥部署到生产环境

由于解密密钥绝不应该提交，你需要手动将此文件存储在某处并部署它。有两种方法：

**(1) 上传文件**

第一个选项是将**生产解密密钥**——`config/secrets/prod/prod.decrypt.private.php`——复制到你的服务器。

**(2) 使用环境变量**

第二种方法是将 `SYMFONY_DECRYPTION_SECRET` 环境变量设置为**生产解密密钥**的 base64 编码值。获取密钥值的便捷方式是：

```terminal
   # 此命令仅获取密钥的值；你还必须在系统中
   # 使用此值设置环境变量（例如 `export SYMFONY_DECRYPTION_SECRET=...`）
   $ php -r 'echo base64_encode(require "config/secrets/prod/prod.decrypt.private.php");'
```

为了提高性能（即避免在运行时解密密钥），你可以在部署期间将密钥解密到"本地"vault：

```terminal
   $ APP_RUNTIME_ENV=prod php bin/console secrets:decrypt-to-local --force
```

这将把所有解密后的密钥写入 `%kernel.project_dir%/.env.<environment>.local` 文件（在此示例中为 `.env.prod.local`）。完成此操作后，解密密钥**不**需要保留在服务器上。

> **提示：** 如果你的应用将环境文件存储在自定义目录中，请覆盖 `secrets.local_vault` 服务以指向正确的位置：
>
> ```yaml
> # config/services.yaml
> services:
>     secrets.local_vault:
>         class: Symfony\Bundle\FrameworkBundle\Secrets\DotenvVault
>         arguments:
>             - '%kernel.project_dir%/env/.env.%kernel.environment%.local'
> ```

> **提示：** 当设置了 `SYMFONY_DECRYPTION_SECRET` 且未定义 `APP_SECRET` 时，`kernel.secret` 参数会自动从解密密钥派生。这意味着如果你已经在使用密钥 vault，就不需要定义单独的 `APP_SECRET` 环境变量。

## 轮换密钥

`secrets:generate-keys` 命令提供 `--rotate` 选项以重新生成**加密密钥**。Symfony 将使用旧密钥解密现有密钥，生成新的**加密密钥**，并使用新密钥重新加密密钥。为了解密之前的密钥，开发者必须拥有**解密密钥**。

## 配置

密钥系统默认启用，其行为的某些方面可以配置：

```yaml
# config/packages/framework.yaml
framework:
    secrets:
        #vault_directory: '%kernel.project_dir%/config/secrets/%kernel.environment%'
        #local_dotenv_file: '%kernel.project_dir%/.env.%kernel.environment%.local'
        #decryption_env_var: 'base64:default::SYMFONY_DECRYPTION_SECRET'
```

```php
// config/packages/framework.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return App::config([
    'framework' => [
        'secrets' => [
            // 'vault_directory' => '%kernel.project_dir%/config/secrets/%kernel.environment%',
            // 'local_dotenv_file' => '%kernel.project_dir%/.env.%kernel.environment%.local',
            // 'decryption_env_var' => 'base64:default::SYMFONY_DECRYPTION_SECRET',
        ],
    ],
]);
```
