# 用 PHP 语言实现 n 的阶乘

据本人了解，阶乘的实现方法一般可以分为三种，通常意义下的递归和循环各算一种，还有一大类通过一些巧妙的数学方法减少运算次数（尤其是乘法运算次数），进而优化计算效率。

如果要考虑到高精度、大整数的阶乘，对于 PHP 语言而言，情况会更复杂一些，比如使用 BCMath 扩展提供的一些方法时，显式的数字与字符串转换操作比较频繁。

本文在只考虑 n 为整数的情况下，分别尝试实现上述的几种情况，每种情况给出可用的代码示例，并在文末附上几种方法的综合对比情况。

## 普通递归实现

首先是普通递归实现，根据递归的通用公式 `fact(n) = n * fact(n-1)` 很容易写出阶乘的计算代码。普通递归实现的优点在于代码比较简洁，和通用公式一样的过程使得代码容易理解。缺点则在于由于需要频繁地调用自身，需要大量的入栈出栈操作，整体的计算效率不高。

```php
function fact(int $n): int
{
    if ($n == 0) {
        return 1;
    }

    return $n * fact($n - 1);
}
```

## 普通循环实现

普通循环实现有些「动态规划」的味道，但由于中间态变量使用频率低，不需要额外存储空间，所以要比一般的动态规划算法简单。普通递归方法是自顶向下（由 n 到 1）的计算过程，而普通循环是自底向上进行计算。

因为，相对而言，代码没有上述方法直观，但由于少了频繁的入栈出栈过程，计算效率会高一些。

```php
function fact(int $n): int
{
    $result = 1;
    $num = 1;
    while ($num <= $n) {
        $result = $result * $num;
        $num = $num + 1;
    }

    return $result;
}
```

## 自行实现的大整数阶乘

由于 PHP 中 `int` 类型的范围限制，上述两种方法最多只能**精确**计算到 20 的阶乘，如果说只是考虑到 20 的阶乘的情况，那么用**查表法**实现会更快：事先计算好 0-20 的阶乘并存储到一个数组中，需要用时查询一次便可。

为了能够适应大数的阶乘，得到精确的计算结果，本文基于「普通循环方法」进行改进，使用数组存储计算结果中的每一位（由低到高位），通过相乘进位的方式依次计算每一位的结果。

不言而喻，本方法的优点在于可以适用于**高精度的大数阶乘**场合，缺点就是对于小数阶乘而言，计算过程复杂且速度慢。

```php
function fact(int $n): array
{
    $result = [1];

    $num = 1;
    while ($num <= $n) {
        $carry = 0;

        for ($index = 0; $index < count($result); $index++) {
            $tmp = $result[$index] * $num + $carry;
            $result[$index] = $tmp % 10;
            $carry = floor($tmp / 10);
        }

        while ($carry > 0) {
            $result[] = $carry % 10;
            $carry = floor($carry / 10);
        }

        $num = $num + 1;
    }

    return $result;
}
```

## BCMath 扩展方法

BCMath 是 PHP 的一个数学扩展，用于处理字符串表示的数字（任意大小和精度）的数值计算。由于是使用 C 语言实现的扩展，计算速度会比上述自行实现的快。

在本人的笔记本上，同样是计算 2000 的阶乘，自行实现的需要平均 0.5-0.6 秒，使用 BCMath 耗时 0.18-0.19 秒。该方法的缺点主要在于需要安装相应的扩展，属于非代码层面的改动，对于环境管理升级不便的应用而言，可实践性有待商榷。

```php
function fact(int $n): string
{
    $result = '1';
    $num = '1';
    while ($num <= $n) {
        $result = bcmul($result, $num);
        $num = bcadd($num, '1');
    }

    return $result;
}
```

## 优化算法

在本文开头有提到，优化算法尝试尽可能地减少运算次数（尤其是乘法的运算次数）来实现快速阶乘。考虑到对于小整数阶乘而言，最快的算法应该是**查表法**，时间复杂度为 O(1)，所以本小节主要针对大整数的精确阶乘进行讨论和测试。

据了解，目前阶乘优化比较常见的是通过 `n! = C(n, n/2) * (n/2)! * (n/2)!` 式子进行复杂度优化，而该式子中的亮点主要在于 C(n, n/2) 的优化。考虑到大整数情况下，PHP 语言实现 C(n, n/2) 的效率不高，而且实现的代码可读性比较差（频繁的数字与字符串的显式转换），所以本文用的是另外一种比较巧妙的方法。

乘法的计算速度通常要低于加减法运算，通过减少乘法的运算次数可以提高整体运算速度。通过数学归纳可以发现，对于 n 的阶乘，可以依次求出比 (n/2)^2 小 1、1+3、1+3+5... 的数值，再依次相乘得到目标值。

该算法的优点在于计算速度较快，而缺点就是实现过程不直观、不易理解。经测试，以下代码计算 2000 的阶乘平均时间为 0.11 秒，大约是普通循环方法的一半耗时。

除了这种方法优化，也有看到其它的类似的思路，比如对 1...n 中的数反复检验是否被 2 整除，记录下被 2 整除的次数 x，并尝试归纳出共同的奇数相乘式，最后乘以 2^x 得到结果。

```php
function fact(int $n): string
{
    $middleSquare = pow(floor($n / 2), 2);
    $result = $n & 1 == 1 ? 2 * $middleSquare * $n : 2 * $middleSquare;
    $result = (string)$result;
    for ($num = 1; $num < $n - 2; $num = $num + 2) {
        $middleSquare = $middleSquare - $num;
        $result = bcmul($result, (string)$middleSquare);
    }

    return $result;
}
```

## 综合对比

| 方法                 | T(n) | S(n) | 测试耗时                            | 优点                                     | 缺点                             |
| -------------------- | :--: | :--: | :---------------------------------- | ---------------------------------------- | -------------------------------- |
| 普通递归实现         | O(n) | O(1) | 10 万次 20 的阶乘平均耗时 0.1728 秒 | 代码直观易理解                           | 多次出入栈操作，效率较低         |
| 普通循环实现         | O(n) | O(1) | 10 万次 20 的阶乘平均耗时 0.0991 秒 | 计算效率较高                             | 代码实现不够直观                 |
| 自行实现的大整数阶乘 |  -   |  -   | 2000 的阶乘平均耗时 0.5-0.6 秒      | 能够处理大整数的高精度阶乘               | 对于小整数计算效率不高           |
| BCMath 扩展方法      |  -   |  -   | 2000 的阶乘平均耗时 0.18-0.19 秒    | C 语言实现的扩展，比自行实现的效率高     | 非代码层的依赖，可实践性有待商榷 |
| 优化算法             | O(n) | O(1) | 2000 的阶乘平均耗时 0.11 秒         | 通过减少乘法运算次数，提高了整体计算速度 | 实现方式比较另类，不易理解       |
| 查表法（文中提到）   | O(1) |  -   | -                                   |                                          |                                  |

