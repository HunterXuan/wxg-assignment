# 用 PHP 语言实现 n 的阶乘

## 递归实现

递归实现的优势是代码简洁，和递归的通用公式 `fact(n) = n * fact(n-1)` 形式一致，易于理解；缺点在于需要频繁地入栈出栈，效率不高。

```php
function fact(int $n): int
{
    if ($n == 0) {
        return 1;
    }

    return $n * fact($n - 1);
}
```

## 循环实现

通过循环实现的优缺点刚好与递归的相反，循环实现没有频繁的入栈出栈过程，效率相对高一些；而相对于递归实现，算法实现不够直观。

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

## 大整数阶乘

由于 PHP 中 `int` 的范围限制，上述实现方法最大只可以计算到 20 的阶乘。为了能够计算更大的数，同时得到精确结果，可以对「循环实现」中的算法做改进：使用数组存储计算结果的每一位，通过相乘进位的方式依次计算每一位的结果。

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

## 使用 BCMath 扩展

BCMath 是 PHP 的一个数学扩展，支持对于用字符串表示的任意大小和精度的数字的二进制计算。由于是使用 C 实现的扩展，计算速度会比上面用 PHP 自行实现的快些。在我的笔记本上，同样是计算 2000 的阶乘，自行实现的需要平均 0.5-0.6 秒，使用 BCMath 耗时 0.18-0.19 秒。

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

## 减少乘法运算

乘法的计算速度通常要低于加减法运算，通过减少乘法的运算次数可以提高整体运算速度。对于 n 的阶乘，可以依次求出比 (n/2)^2 小 1、1+3、1+3+5... 的数值，再依次相乘得到目标值。

该算法的优势是计算速度快，缺点在于实现过程不直观，不易理解。经测试以下代码计算 2000 的阶乘平均时间为 0.11 秒，大约是普通循环方法的一半耗时。

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
