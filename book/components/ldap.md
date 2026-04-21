# Ldap 组件

Ldap 组件提供了连接到 LDAP 服务器（OpenLDAP 或 Active Directory）的方法。

## 安装

```terminal
$ composer require symfony/ldap
```

## 使用

`Symfony\Component\Ldap\Ldap` 类提供了针对 LDAP 服务器进行身份验证和查询的方法。

`Ldap` 类使用 `Symfony\Component\Ldap\Adapter\AdapterInterface` 与 LDAP 服务器通信。例如，PHP 内置 LDAP 扩展的适配器 `Symfony\Component\Ldap\Adapter\ExtLdap\Adapter` 可以通过以下选项进行配置：

`host`
: LDAP 服务器的 IP 或主机名

`port`
: 访问 LDAP 服务器所用的端口

`version`
: 使用的 LDAP 协议版本

`encryption`
: 加密协议：`ssl`、`tls` 或 `none`（默认）

`connection_string`
: 可以使用此选项代替 `host` 和 `port` 来连接 LDAP 服务器

`optReferrals`
: 指定是否自动跟随 LDAP 服务器返回的引用

`options`
: LDAP 服务器的选项，定义在
  `Symfony\Component\Ldap\Adapter\ExtLdap\ConnectionOptions` 中

例如，连接到使用 start-TLS 加密的 LDAP 服务器：

```php
use Symfony\Component\Ldap\Ldap;

$ldap = Ldap::create('ext_ldap', [
    'host' => 'my-server',
    'encryption' => 'ssl',
]);
```

或者直接指定连接字符串：

```php
use Symfony\Component\Ldap\Ldap;

$ldap = Ldap::create('ext_ldap', ['connection_string' => 'ldaps://my-server:636']);
```

`Symfony\Component\Ldap\Ldap::bind` 方法使用用户的可分辨名称（DN）和密码对已配置的连接进行身份验证：

```php
use Symfony\Component\Ldap\Ldap;
// ...

$ldap->bind($dn, $password);
```

> **警告：**
> 当 LDAP 服务器允许未经身份验证的绑定时，空密码将始终有效。

你也可以使用 `Symfony\Component\Ldap\Ldap::saslBind` 方法通过 [SASL] 绑定到 LDAP 服务器：

```php
// this method defines other optional arguments like $mech, $realm, $authcId, etc.
$ldap->saslBind($dn, $password);
```

绑定到 LDAP 服务器后，可以使用 `Symfony\Component\Ldap\Ldap::whoami` 方法获取已验证和已授权用户的可分辨名称（DN）。

绑定后（或启用了匿名身份验证），可以使用 `Symfony\Component\Ldap\Ldap::query` 方法查询 LDAP 服务器：

```php
use Symfony\Component\Ldap\Ldap;
// ...

$query = $ldap->query('dc=symfony,dc=com', '(&(objectclass=person)(ou=Maintainers))');
$results = $query->execute();

foreach ($results as $entry) {
    // Do something with the results
}
```

默认情况下，LDAP 条目是延迟加载的。如果希望在单次调用中获取所有条目并对结果数组进行操作，可以使用 `Symfony\Component\Ldap\Adapter\ExtLdap\Collection::toArray` 方法：

```php
use Symfony\Component\Ldap\Ldap;
// ...

$query = $ldap->query('dc=symfony,dc=com', '(&(objectclass=person)(ou=Maintainers))');
$results = $query->execute()->toArray();

// Do something with the results array
```

默认情况下，LDAP 查询使用 `Symfony\Component\Ldap\Adapter\QueryInterface::SCOPE_SUB` 作用域，对应 `ldap_search` 函数的 `LDAP_SCOPE_SUBTREE` 作用域。也可以使用 `SCOPE_BASE`（对应 `ldap_read` 的 `LDAP_SCOPE_BASE`）和 `SCOPE_ONE`（对应 `ldap_list` 的 `LDAP_SCOPE_ONELEVEL`）：

```php
use Symfony\Component\Ldap\Adapter\QueryInterface;

$query = $ldap->query('dc=symfony,dc=com', '...', ['scope' => QueryInterface::SCOPE_ONE]);
```

使用 `filter` 选项只检索特定属性：

```php
$query = $ldap->query('dc=symfony,dc=com', '...', ['filter' => ['cn', 'mail']);
```

## 创建或更新条目

Ldap 组件提供了创建新 LDAP 条目、更新或删除现有条目的功能：

```php
use Symfony\Component\Ldap\Entry;
use Symfony\Component\Ldap\Ldap;
// ...

$entry = new Entry('cn=Fabien Potencier,dc=symfony,dc=com', [
    'sn' => ['fabpot'],
    'objectClass' => ['inetOrgPerson'],
]);

$entryManager = $ldap->getEntryManager();

// Creating a new entry
$entryManager->add($entry);

// Finding and updating an existing entry
$query = $ldap->query('dc=symfony,dc=com', '(&(objectclass=person)(ou=Maintainers))');
$result = $query->execute();
$entry = $result[0];

$phoneNumber = $entry->getAttribute('phoneNumber');
$isContractor = $entry->hasAttribute('contractorCompany');
// attribute names in getAttribute() and hasAttribute() methods are case-sensitive
// pass FALSE as the second method argument to make them case-insensitive
$isContractor = $entry->hasAttribute('contractorCompany', false);

$entry->setAttribute('email', ['fabpot@symfony.com']);
$entryManager->update($entry);

// Adding or removing values to a multi-valued attribute is more efficient than using update()
$entryManager->addAttributeValues($entry, 'telephoneNumber', ['+1.111.222.3333', '+1.222.333.4444']);
$entryManager->removeAttributeValues($entry, 'telephoneNumber', ['+1.111.222.3333', '+1.222.333.4444']);

// Removing an existing entry
$entryManager->remove(new Entry('cn=Test User,dc=symfony,dc=com'));
```

### 批量更新

使用条目管理器的 `Symfony\Component\Ldap\Adapter\ExtLdap\EntryManager::applyOperations` 方法一次更新多个属性：

```php
use Symfony\Component\Ldap\Entry;
use Symfony\Component\Ldap\Ldap;
// ...

$entry = new Entry('cn=Fabien Potencier,dc=symfony,dc=com', [
    'sn' => ['fabpot'],
    'objectClass' => ['inetOrgPerson'],
]);

$entryManager = $ldap->getEntryManager();

// Adding multiple email addresses at once
$entryManager->applyOperations($entry->getDn(), [
    new UpdateOperation(LDAP_MODIFY_BATCH_ADD, 'mail', 'new1@example.com'),
    new UpdateOperation(LDAP_MODIFY_BATCH_ADD, 'mail', 'new2@example.com'),
]);
```

可能的操作类型有 `LDAP_MODIFY_BATCH_ADD`、`LDAP_MODIFY_BATCH_REMOVE`、`LDAP_MODIFY_BATCH_REMOVE_ALL`、`LDAP_MODIFY_BATCH_REPLACE`。使用 `LDAP_MODIFY_BATCH_REMOVE_ALL` 操作类型时，`$values` 参数必须为 `NULL`。

[SASL]: https://en.wikipedia.org/wiki/Simple_Authentication_and_Security_Layer
