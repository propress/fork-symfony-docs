# Cursor Helper

`Symfony\Component\Console\Cursor` 允许您在控制台命令中更改光标位置。这允许您在输出的任何位置写入：

![Cursor Helper 动画](/_images/components/console/cursor.gif)

```php
// src/Command/MyCommand.php
namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Cursor;
use Symfony\Component\Console\Output\OutputInterface;

#[AsCommand(name: 'app:my-command')]
class MyCommand
{
    // ...

    public function __invoke(Cursor $cursor, OutputInterface $output): int
    {
        // ...

        // 将光标移动到特定列（第1个参数）和行（第2个参数）位置
        $cursor->moveToPosition(7, 11);

        // 并使用输出在此位置写入文本
        $output->write('My text');

        // ...
    }
}
```

## 使用光标

### 移动光标

有几种方法可以控制移动命令光标：

```php
// 将光标从其当前位置向上移动 1 行
$cursor->moveUp();

// 将光标从其当前位置向上移动 3 行
$cursor->moveUp(3);

// 向下同样
$cursor->moveDown();

// 将光标从其当前位置向右移动 1 列
$cursor->moveRight();

// 将光标从其当前位置向右移动 3 列
$cursor->moveRight(3);

// 向左同样
$cursor->moveLeft();

// 将光标移动到从终端左上位置的特定（列，行）位置
$cursor->moveToPosition(7, 11);
```

您可以使用以下方式获取当前命令的光标位置：

```php
$position = $cursor->getCurrentPosition();
// $position[0] // 列（即 x 坐标）
// $position[1] // 行（即 y 坐标）
```

### 清除输出

光标还可以清除屏幕上的一些输出：

```php
// 清除当前行的所有输出
$cursor->clearLine();

// 清除当前位置之后当前行的所有输出
$cursor->clearLineAfter();

// 清除从光标当前位置到屏幕末尾的所有输出
$cursor->clearOutput();

// 清除整个屏幕
$cursor->clearScreen();
```

您还可以在光标上利用 `Symfony\Component\Console\Cursor::show` 和 `Symfony\Component\Console\Cursor::hide` 方法。
