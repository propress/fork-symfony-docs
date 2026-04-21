# Intl 组件

该组件提供对 [ICU 库][ICU library] 本地化数据的访问。

> **参见：**
> 本文介绍如何在任意 PHP 应用程序中将 Intl 功能作为独立组件使用。请阅读 [translation](translation.md) 文章，了解如何在 Symfony 应用程序中实现国际化和管理用户语言环境。

## 安装

```terminal
$ composer require symfony/intl
```

## 访问 ICU 数据

该组件提供以下 ICU 数据：

* 语言和文字名称
* 国家名称
* 语言环境
* 货币
* 时区

### 语言和文字名称

`Symfony\Component\Intl\Languages` 类根据 [ISO 639-1 alpha-2][ISO 639-1 alpha-2] 列表和 [ISO 639-2 alpha-3 (2T)][ISO 639-2 alpha-3 (2T)] 列表提供对所有语言名称的访问：

```php
use Symfony\Component\Intl\Languages;

\Locale::setDefault('en');

$languages = Languages::getNames();
// ('languageCode' => 'languageName')
// => ['ab' => 'Abkhazian', 'ace' => 'Achinese', ...]

$languages = Languages::getAlpha3Names();
// ('languageCode' => 'languageName')
// => ['abk' => 'Abkhazian', 'ace' => 'Achinese', ...]

$language = Languages::getName('fr');
// => 'French'

$language = Languages::getAlpha3Name('fra');
// => 'French'
```

所有方法都接受翻译语言环境作为最后一个可选参数，默认为当前默认语言环境：

```php
$languages = Languages::getNames('de');
// => ['ab' => 'Abchasisch', 'ace' => 'Aceh', ...]

$languages = Languages::getAlpha3Names('de');
// => ['abk' => 'Abchasisch', 'ace' => 'Aceh', ...]

$language = Languages::getName('fr', 'de');
// => 'Französisch'

$language = Languages::getAlpha3Name('fra', 'de');
// => 'Französisch'
```

如果给定的语言环境不存在，这些方法将触发 `Symfony\Component\Intl\Exception\MissingResourceException`。除了捕获异常之外，你还可以检查给定的语言代码是否有效：

```php
$isValidLanguage = Languages::exists($languageCode);
```

如果你有一个 alpha3 语言代码需要检查：

```php
$isValidLanguage = Languages::alpha3CodeExists($alpha3Code);
```

你可以在两字母 alpha2 代码和三字母 alpha3 代码之间进行转换：

```php
$alpha3Code = Languages::getAlpha3Code($alpha2Code);

$alpha2Code = Languages::getAlpha2Code($alpha3Code);
```

`Symfony\Component\Intl\Scripts` 类根据 [Unicode ISO 15924 Registry][Unicode ISO 15924 Registry] 提供对可选四字母文字代码的访问，该代码可以跟在语言代码后面（例如，简体中文 `zh_HANS` 中的 `HANS`，繁体中文 `zh_HANT` 中的 `HANT`）：

```php
use Symfony\Component\Intl\Scripts;

\Locale::setDefault('en');

$scripts = Scripts::getNames();
// ('scriptCode' => 'scriptName')
// => ['Adlm' => 'Adlam', 'Afak' => 'Afaka', ...]

$script = Scripts::getName('Hans');
// => 'Simplified'
```

所有方法都接受翻译语言环境作为最后一个可选参数，默认为当前默认语言环境：

```php
$scripts = Scripts::getNames('de');
// => ['Adlm' => 'Adlam', 'Afak' => 'Afaka', ...]

$script = Scripts::getName('Hans', 'de');
// => 'Vereinfacht'
```

如果给定的文字代码不存在，这些方法将触发 `Symfony\Component\Intl\Exception\MissingResourceException`。除了捕获异常之外，你还可以检查给定的文字代码是否有效：

```php
$isValidScript = Scripts::exists($scriptCode);
```

### 国家名称

`Symfony\Component\Intl\Countries` 类根据官方认可的国家和地区的 [ISO 3166-1 alpha-2][ISO 3166-1 alpha-2] 列表和 [ISO 3166-1 alpha-3][ISO 3166-1 alpha-3] 列表提供对所有国家名称的访问：

```php
use Symfony\Component\Intl\Countries;

\Locale::setDefault('en');

$countries = Countries::getNames();
// ('alpha2Code' => 'countryName')
// => ['AF' => 'Afghanistan', 'AX' => 'Åland Islands', ...]

$countries = Countries::getAlpha3Names();
// ('alpha3Code' => 'countryName')
// => ['AFG' => 'Afghanistan', 'ALA' => 'Åland Islands', ...]

$country = Countries::getName('GB');
// => 'United Kingdom'

$country = Countries::getAlpha3Name('NOR');
// => 'Norway'
```

所有方法都接受翻译语言环境作为最后一个可选参数，默认为当前默认语言环境：

```php
$countries = Countries::getNames('de');
// => ['AF' => 'Afghanistan', 'EG' => 'Ägypten', ...]

$countries = Countries::getAlpha3Names('de');
// => ['AFG' => 'Afghanistan', 'EGY' => 'Ägypten', ...]

$country = Countries::getName('GB', 'de');
// => 'Vereinigtes Königreich'

$country = Countries::getAlpha3Name('GBR', 'de');
// => 'Vereinigtes Königreich'
```

如果给定的国家代码不存在，这些方法将触发 `Symfony\Component\Intl\Exception\MissingResourceException`。除了捕获异常之外，你还可以检查给定的国家代码是否有效：

```php
$isValidCountry = Countries::exists($alpha2Code);
```

如果你有一个 alpha3 国家代码需要检查：

```php
$isValidCountry = Countries::alpha3CodeExists($alpha3Code);
```

你可以在两字母 alpha2 代码和三字母 alpha3 代码之间进行转换：

```php
$alpha3Code = Countries::getAlpha3Code($alpha2Code);

$alpha2Code = Countries::getAlpha2Code($alpha3Code);
```

### 国家数字代码

[ISO 3166-1 numeric][ISO 3166-1 numeric] 标准定义了用于表示国家、附属地区和特殊地理区域的三位数国家代码。

与 ISO 3166-1 字母代码（alpha-2 和 alpha-3）相比，其主要优势在于这些数字代码独立于书写系统。字母代码使用 26 个英文字母，对于使用非拉丁文字（例如阿拉伯语或日语）的人员和系统来说，可能不可用或难以使用。

`Symfony\Component\Intl\Countries` 类提供对这些数字国家代码的访问：

```php
use Symfony\Component\Intl\Countries;

\Locale::setDefault('en');

$numericCodes = Countries::getNumericCodes();
// ('alpha2Code' => 'numericCode')
// => ['AA' => '958', 'AD' => '020', ...]

$numericCode = Countries::getNumericCode('FR');
// => '250'

$alpha2 = Countries::getAlpha2FromNumeric('250');
// => 'FR'

$exists = Countries::numericCodeExists('250');
// => true
```

> **注意：**
> 当设置了 `SYMFONY_INTL_WITH_USER_ASSIGNED` 环境变量时，Symfony Intl 组件还将识别用户分配的代码：`XK`、`XKK` 和 `983`。这允许应用程序处理这些代码，对于需要支持这些区域的应用程序非常有用。

### 语言环境

语言环境是语言、地区和一些定义用户界面偏好参数的组合。例如，"中文"是语言，`zh_Hans_MO` 是"中文"（语言）+ "简体"（文字）+ "中国澳门特别行政区"（地区）的语言环境。`Symfony\Component\Intl\Locales` 类提供对所有语言环境名称的访问：

```php
use Symfony\Component\Intl\Locales;

\Locale::setDefault('en');

$locales = Locales::getNames();
// ('localeCode' => 'localeName')
// => ['af' => 'Afrikaans', 'af_NA' => 'Afrikaans (Namibia)', ...]

$locale = Locales::getName('zh_Hans_MO');
// => 'Chinese (Simplified, Macau SAR China)'
```

所有方法都接受翻译语言环境作为最后一个可选参数，默认为当前默认语言环境：

```php
$locales = Locales::getNames('de');
// => ['af' => 'Afrikaans', 'af_NA' => 'Afrikaans (Namibia)', ...]

$locale = Locales::getName('zh_Hans_MO', 'de');
// => 'Chinesisch (Vereinfacht, Sonderverwaltungsregion Macau)'
```

如果给定的语言环境代码不存在，这些方法将触发 `Symfony\Component\Intl\Exception\MissingResourceException`。除了捕获异常之外，你还可以检查给定的语言环境代码是否有效：

```php
$isValidLocale = Locales::exists($localeCode);
```

### 货币

`Symfony\Component\Intl\Currencies` 类提供对所有货币名称及其部分信息（符号、小数位数等）的访问：

```php
use Symfony\Component\Intl\Currencies;

\Locale::setDefault('en');

$currencies = Currencies::getNames();
// ('currencyCode' => 'currencyName')
// => ['AFN' => 'Afghan Afghani', 'ALL' => 'Albanian Lek', ...]

$currency = Currencies::getName('INR');
// => 'Indian Rupee'

$symbol = Currencies::getSymbol('INR');
// => '₹'
```

小数位数方法返回使用该货币格式化数字时要显示的小数位数。根据货币的不同，如果数字用于现金交易或其他场景（例如会计），该值可能会有所不同：

```php
// Indian rupee defines the same value for both
$fractionDigits = Currencies::getFractionDigits('INR');         // returns: 2
$cashFractionDigits = Currencies::getCashFractionDigits('INR'); // returns: 2

// Swedish krona defines different values
$fractionDigits = Currencies::getFractionDigits('SEK');         // returns: 2
$cashFractionDigits = Currencies::getCashFractionDigits('SEK'); // returns: 0
```

某些货币要求将数字四舍五入到某个值的最近增量（例如 5 美分）。如果数字用于现金交易或其他场景（例如会计），该增量可能会有所不同：

```php
// Indian rupee defines the same value for both
$roundingIncrement = Currencies::getRoundingIncrement('INR');         // returns: 0
$cashRoundingIncrement = Currencies::getCashRoundingIncrement('INR'); // returns: 0

// Canadian dollar defines different values because they have eliminated
// the smaller coins (1-cent and 2-cent) and prices in cash must be rounded to
// 5 cents (e.g. if price is 7.42 you pay 7.40; if price is 7.48 you pay 7.50)
$roundingIncrement = Currencies::getRoundingIncrement('CAD');         // returns: 0
$cashRoundingIncrement = Currencies::getCashRoundingIncrement('CAD'); // returns: 5
```

除 `getFractionDigits()`、`getCashFractionDigits()`、`getRoundingIncrement()` 和 `getCashRoundingIncrement()` 之外，所有方法都接受翻译语言环境作为最后一个可选参数，默认为当前默认语言环境：

```php
$currencies = Currencies::getNames('de');
// => ['AFN' => 'Afghanischer Afghani', 'EGP' => 'Ägyptisches Pfund', ...]

$currency = Currencies::getName('INR', 'de');
// => 'Indische Rupie'
```

如果给定的货币代码不存在，这些方法将触发 `Symfony\Component\Intl\Exception\MissingResourceException`。除了捕获异常之外，你还可以检查给定的货币代码是否有效：

```php
$isValidCurrency = Currencies::exists($currencyCode);
```

默认情况下，上述货币方法返回所有货币，包括不再使用的货币。Symfony 提供了几种过滤货币的方法，以便你只处理对给定国家/地区实际有效且正在使用的货币。

这些方法使用 ICU 元数据（`tender`、`from` 和 `to` 日期）来确定货币是否为[法定货币][legal tender]以及/或者在特定时间点是否有效：

```php
use Symfony\Component\Intl\Currencies;

// get the list of today's legal and active currencies for a country
$codes = Currencies::forCountry('FR');
// ['EUR']

// include non-legal currencies too, and check them at a given date
$codesAll = Currencies::forCountry(
    'ES',
    legalTender: null,
    active: true,
    date: new \DateTimeImmutable('1982-01-01')
);
// ['ESP', 'ESB']

// check if a currency is valid today for a country
$isOk = Currencies::isValidInCountry('CH', 'CHF');
// true

// check if a currency is valid in any country on a specific date
$isGlobal = Currencies::isValidInAnyCountry(
    'USD',
    legalTender: true,
    active: true,
    date: new \DateTimeImmutable('2005-01-01')
);
// true
```

请注意，某些货币（尤其是非法定货币）没有定义有效期范围。在这种情况下，将抛出 `RuntimeException`。此外，如果指定的货币无效，将抛出 `InvalidArgumentException`。

### 时区

`Symfony\Component\Intl\Timezones` 类提供与时区相关的多种实用工具。首先，你可以获取所有语言中所有时区的名称和值：

```php
use Symfony\Component\Intl\Timezones;

\Locale::setDefault('en');

$timezones = Timezones::getNames();
// ('timezoneID' => 'timezoneValue')
// => ['America/Eirunepe' => 'Acre Time (Eirunepe)', 'America/Rio_Branco' => 'Acre Time (Rio Branco)', ...]

$timezone = Timezones::getName('Africa/Nairobi');
// => 'East Africa Time (Nairobi)'
```

所有方法都接受翻译语言环境作为最后一个可选参数，默认为当前默认语言环境：

```php
$timezones = Timezones::getNames('de');
// => ['America/Eirunepe' => 'Acre-Zeit (Eirunepe)', 'America/Rio_Branco' => 'Acre-Zeit (Rio Branco)', ...]

$timezone = Timezones::getName('Africa/Nairobi', 'de');
// => 'Ostafrikanische Zeit (Nairobi)'
```

你还可以获取给定国家/地区中存在的所有时区。`forCountryCode()` 方法返回一个或多个时区 ID，你可以使用前面介绍的 `getName()` 方法将其翻译为任何语言环境：

```php
// unlike language codes, country codes are always uppercase (CL = Chile)
$timezones = Timezones::forCountryCode('CL');
// => ['America/Punta_Arenas', 'America/Santiago', 'Pacific/Easter']
```

反向查找也可以通过 `getCountryCode()` 方法实现，该方法返回给定时区 ID 所属国家/地区的代码：

```php
$countryCode = Timezones::getCountryCode('America/Vancouver');
// => $countryCode = 'CA' (CA = Canada)
```

所有时区的 [UTC/GMT 时间偏移量][UTC/GMT time offsets] 由 `getRawOffset()`（返回表示偏移量秒数的整数）和 `getGmtOffset()`（返回偏移量的字符串表示形式以显示给用户）提供：

```php
$offset = Timezones::getRawOffset('Etc/UTC');              // $offset = 0
$offset = Timezones::getRawOffset('America/Buenos_Aires'); // $offset = -10800
$offset = Timezones::getRawOffset('Asia/Katmandu');        // $offset = 20700

$offset = Timezones::getGmtOffset('Etc/UTC');              // $offset = 'GMT+00:00'
$offset = Timezones::getGmtOffset('America/Buenos_Aires'); // $offset = 'GMT-03:00'
$offset = Timezones::getGmtOffset('Asia/Katmandu');        // $offset = 'GMT+05:45'
```

由于[夏令时（DST）][daylight saving time (DST)]实践，时区偏移量可能随时间变化。默认情况下，这些方法使用 PHP 的 `time()` 函数获取当前时区偏移量值，但你可以将时间戳作为第二个参数传递，以获取任意时间点的偏移量：

```php
// In 2019, the DST period in Madrid (Spain) went from March 31 to October 27
$offset = Timezones::getRawOffset('Europe/Madrid', strtotime('March 31, 2019'));   // $offset = 3600
$offset = Timezones::getRawOffset('Europe/Madrid', strtotime('April 1, 2019'));    // $offset = 7200
$offset = Timezones::getGmtOffset('Europe/Madrid', strtotime('October 27, 2019')); // $offset = 'GMT+02:00'
$offset = Timezones::getGmtOffset('Europe/Madrid', strtotime('October 28, 2019')); // $offset = 'GMT+01:00'
```

GMT 偏移量的字符串表示形式可能因语言环境而异，因此你可以将语言环境作为第三个可选参数传递：

```php
$offset = Timezones::getGmtOffset('Europe/Madrid', strtotime('October 28, 2019'), 'ar'); // $offset = 'غرينتش+01:00'
$offset = Timezones::getGmtOffset('Europe/Madrid', strtotime('October 28, 2019'), 'dz'); // $offset = 'ཇི་ཨེམ་ཏི་+01:00'
```

如果给定的时区 ID 不存在，这些方法将触发 `Symfony\Component\Intl\Exception\MissingResourceException`。除了捕获异常之外，你还可以检查给定的时区 ID 是否有效：

```php
$isValidTimezone = Timezones::exists($timezoneId);
```

### Emoji 音译

Symfony 提供了将表情符号翻译为所有语言文本表示形式的实用工具。请阅读关于 emoji 音译的文档以了解更多有关此功能的信息。

## 磁盘空间

如果你需要节省磁盘空间（例如，因为你部署到有严格大小限制的服务），请运行此命令（例如，作为 `composer install` 后的自动化脚本），使用 PHP `zlib` 扩展压缩内部 Symfony Intl 数据文件：

```terminal
# adjust the path to the 'compress' binary based on your application installation
$ php ./vendor/symfony/intl/Resources/bin/compress
```

[ICU library]: https://icu.unicode.org/
[Unicode ISO 15924 Registry]: https://www.unicode.org/iso15924/iso15924-codes.html
[ISO 3166-1 alpha-2]: https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2
[ISO 3166-1 alpha-3]: https://en.wikipedia.org/wiki/ISO_3166-1_alpha-3
[ISO 3166-1 numeric]: https://en.wikipedia.org/wiki/ISO_3166-1_numeric
[legal tender]: https://en.wikipedia.org/wiki/Legal_tender
[UTC/GMT time offsets]: https://en.wikipedia.org/wiki/List_of_UTC_time_offsets
[daylight saving time (DST)]: https://en.wikipedia.org/wiki/Daylight_saving_time
[ISO 639-1 alpha-2]: https://en.wikipedia.org/wiki/ISO_639-1
[ISO 639-2 alpha-3 (2T)]: https://en.wikipedia.org/wiki/ISO_639-2
