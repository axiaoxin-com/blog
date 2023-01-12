---
title: "双斜杠“//”在 Python 中是什么意思？Python 中的运算符"
author: "axiaoxin"
date: 2023-01-12T10:18:47+08:00
subtitle: ""
image: ""
tags: ["python"]
slug: ""
---

在 Python 中，您使用双斜杠 `//` 运算符来执行地板除法（floor division）。此 `//` 运算符将第一个数字除以第二个数字，并将结果向下取整。

在本文中，我将向您展示如何使用 `//` 运算符，并将其与常规除法进行比较，以便您了解它的工作原理。

除此之外——您还将了解与双斜杠 `//` 运算符同义的 Python 数学方法。


## `//` 运算符的基本语法

要使用双斜杠 `//` 操作符，您的操作几乎类似于常规除法。唯一的区别是使用双斜杠 `//` 而不是单斜杠 `/`:

```python
first_num // second_num
```

## 地板除法示例

在下面的例子中，12 除以 5 的地板除法结果是 2:

```python
num1 = 12
num2 = 5
num3 = num1 // num2

print("floor division of", num1, "by", num2, "=", num3)
# Output: floor division of 12 by 5 = 2
```

而 12 除以 5 的常规除法等于 2.4。也就是说结果为 2， 余数为 4：

```python
num2 = 5
num3 = num1 / num2

print("normal division of", num1, "by", num2, "=", num3)
# Output: normal division of 12 by 5 = 2.4
```

这表明 `//` 运算符将两个数字的除法结果向下取整为最接近的整数。

即使小数位是 9，`//` 运算符仍会将结果向下取整。

```python
num1 = 29
num2 = 10
num3 = num1 / num2
num4 = num1 // num2

print("normal division of", num1, "by", num2, "=", num3)
print("but floor division of", num1, "by", num2, "=", num4)

"""
Output:
normal division of 29 by 10 = 2.9
but floor division of 29 by 10 = 2
"""
```

如果用负数进行地板除法，结果仍然会向下取整。

为了让你对结果有心理准备，向下舍入一个负数意味着远离 0。因此，-12 除以 5 结果为 -3。不要感到困惑——尽管乍一看数字似乎在变“大”，但实际上它在变小（离零更远/一个更大的负数）。


```python
num1 = -12
num2 = 5
num3 = num1 // num2

print("floor division of", num1, "by", num2, "=", num3)

# floor division of -12 by 5 = -3

```


## 双斜杠 `//` 运算符的工作原理与 `math.floor()` 类似

在 Python 中，math.floor() 将数字向下取整，就像双斜杠 `//` 运算符一样。

因此，`math.floor()` 是 `//` 运算符的替代方法，因为它们在底层是执行相同的操作。

下面是一个示例：

```python
import math

num1 = 12
num2 = 5
num3 = num1 // num2
num4 = math.floor(num1 / num2)

print("floor division of", num1, "by", num2, "=", num3)
print("math.floor of", num1, "divided by", num2, "=", num4)

"""
Output:
floor division of 12 by 5 = 2
math.floor of 12 divided by 5 = 2
"""
```

您可以看到 `math.floor()` 与 `//` 运算符执行的结果是相同的。


## 双斜杠 `//` 运算符在幕后的工作原理

当您使用 `//` 运算符将两个数字相除时，在底层调用的方法是 `__floordiv__()`。

您也可以直接使用此 `__floordiv__()` 方法来代替 `//` 运算符：

```python
num1 = 12
num2 = 5
num3 = num1 // num2
num4 = num1.__floordiv__(num2)

print("floor division of", num1, "by", num2, "=", num3)
print("using the floordiv method gets us the same value of", num4)

"""
Output:
floor division of 12 by 5 = 2
using the floordiv method gets us the same value of 2
"""
```

## 总结

在本文中，您了解了如何使用双斜杠 `//` 运算符以及它在底层的工作原理。

此外，您还了解了 `//` 运算符的两种替代方法 – `math.floor()` 和 `__floordiv__()` 方法。

不用纠结应该使用哪一种方法。地板除法的三种方法是工作原理都是一样的。但是我建议你使用双斜杠 `//` 操作符，因为使用用它可以减少输入。

感谢您的阅读。
