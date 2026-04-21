# 性能优化

Symfony 本身速度很快。但你可以通过优化服务器和应用程序进一步提速。

## 性能检查清单

### Symfony 应用程序清单

1. **限制应用中启用的语言区域数量**

   使用 `framework.enabled_locales` 选项，只生成应用实际使用的翻译文件。

### 生产服务器清单

1. **将服务容器转储为单个文件**

   ```yaml
   # config/services.yaml
   parameters:
       .container.dumper.inline_factories: true
   ```

2. **使用 OPcache 字节码缓存**

   OPcache 缓存 PHP 脚本的编译字节码，避免在每次请求时重新编译。

3. **使用 OPcache 类预加载**

   ```ini
   ; php.ini
   opcache.preload=/path/to/project/config/preload.php
   opcache.preload_user=www-data
   ```

4. **为最大性能配置 OPcache**

   ```ini
   ; php.ini
   opcache.memory_consumption=256
   opcache.max_accelerated_files=32531
   opcache.interned_strings_buffer=32
   ```

5. **不检查 PHP 文件时间戳**

   ```ini
   ; php.ini
   opcache.validate_timestamps=0
   ```

   每次部署后，必须清空并重新生成 OPcache 的缓存。

6. **配置 PHP realpath 缓存**

   ```ini
   ; php.ini
   realpath_cache_size=4096K
   realpath_cache_ttl=600
   ```

7. **优化 Composer 自动加载器**

   ```terminal
   $ composer dump-autoload --no-dev --classmap-authoritative
   ```

   - `--no-dev` 排除开发环境所需的类
   - `--classmap-authoritative` 创建类映射，防止 Composer 扫描文件系统

8. **在调试模式下禁用 XML 容器转储**

   ```yaml
   # config/services.yaml
   parameters:
       debug.container.dump: false
   ```

---

## 分析 Symfony 应用

### 使用 Blackfire 分析

[Blackfire](https://blackfire.io/) 是在开发、测试和生产中分析和优化 Symfony 应用性能的最佳工具。

### 使用 Symfony Stopwatch 分析

Symfony 在开发配置环境中提供基本性能分析器。点击 Web 调试工具栏中的"时间面板"可以查看 Symfony 在各任务（如数据库查询、模板渲染）上花费的时间。

通过类型提示注入 `Stopwatch` 类来测量代码执行时间：

```php
use Symfony\Component\Stopwatch\Stopwatch;

class MyService
{
    public function __construct(private Stopwatch $stopwatch)
    {
    }

    public function doSomething(): void
    {
        $this->stopwatch->start('myMethod');

        // ... 执行某些操作

        $event = $this->stopwatch->stop('myMethod');
        // $event->getDuration()、$event->getMemory()
    }
}
```
