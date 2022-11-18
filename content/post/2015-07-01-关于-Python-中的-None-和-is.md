---
title: "关于 Python 中的 None 和 is"
author: "axiaoxin"
date: 2015-07-01T11:46:27+08:00
subtitle: ""
image: ""
tags: ["python"]
slug: ""
---

## Python 中的 None

`x is None` 比 `x == None` 更快。

`is` 是用 C 实现的, 简单的比较两个对象的 ID，不需要调用任何方法函数, `is` 是比较 id，`==` 是比较值。

创建一个 None 实例：

```python
Python 2.7.6 (default, Sep  9 2014, 15:04:36)
[GCC 4.2.1 Compatible Apple LLVM 6.0 (clang-600.0.39)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> x = type(None)()
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
TypeError: cannot create 'NoneType' instances

Python 3.4.3 (default, Mar 17 2015, 18:12:35)
[GCC 4.2.1 Compatible Apple LLVM 6.0 (clang-600.0.56)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> x = type(None)()
>>> type(x)
<class 'NoneType'>
>>> x is None
True
```

无论如何, `id(None)` 总是返回相同的值. 因为 None 在 Python 中是一个单例：

```python
>>> x = type(None)()
>>> id(x)
4489363032
>>> y = type(None)()
>>> id(y)
4489363032
```


## Python 中的 is

CPython 对 is 的优化：

参考代码：
- <https://github.com/python/cpython/blob/829b49cbd2e4b1d573470da79ca844b730120f3d/Python/peephole.c#L223>
- <https://gist.github.com/mrluanma/9118bf2ada09ebd017d3>

演示代码：

```python
>>> x = 'a' * 20
>>> y = 'a' * 20
>>> x is y
True
>>> x = 'a' * 21
>>> y = 'a' * 21
>>> x is y
False
>>> x = 'a' * 21; y = 'a' * 21
>>> x is y
False
>>> x = 'aaaaaaaaaaaaaaaaaaaaa'
>>> len(x)
21
>>> y = 'aaaaaaaaaaaaaaaaaaaaa'
>>> x is y
True
>>> x = intern('a' * 21)
>>> y = intern('a' * 21)
>>> x is y
True
```

intern 是 Python2 的内建方法：接受一个字符串作为参数(并且只有一个字符串)。

它返回一个引用——要么创建一个新的字符串,或一个已经分配好内存的字符串。

“intern”保证每一个独一无二的字符串只分配一次。

如果你使用“intern”第二次传入相同的字符串, Python 返回一个对第一个字符串的引用。

所以在需要使用重复的长字符串时，可以考虑使用intern减少内存分配。

Python2:

```python
Python 2.7.6 (default, Sep  9 2014, 15:04:36)
[GCC 4.2.1 Compatible Apple LLVM 6.0 (clang-600.0.39)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> for i in range(25):
...   x = 'a' * i
...   y = 'a' * i
...   print("[{}]: x is y == {}".format(i, x is y))
...
[0]: x is y == False
[1]: x is y == True
[2]: x is y == False
[3]: x is y == False
[4]: x is y == False
[5]: x is y == False
[6]: x is y == False
[7]: x is y == False
[8]: x is y == False
[9]: x is y == False
[10]: x is y == False
[11]: x is y == False
[12]: x is y == False
[13]: x is y == False
[14]: x is y == False
[15]: x is y == False
[16]: x is y == False
[17]: x is y == False
[18]: x is y == False
[19]: x is y == False
[20]: x is y == False
[21]: x is y == False
[22]: x is y == False
[23]: x is y == False
[24]: x is y == False
```

Python3:

```python
Python 3.4.3 (default, Mar 17 2015, 18:12:35)
[GCC 4.2.1 Compatible Apple LLVM 6.0 (clang-600.0.56)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> for i in range(25):
...   x = 'a' * i
...   y = 'a' * i
...   print("[{}]: x is y == {}".format(i, x is y))
...
[0]: x is y == True
[1]: x is y == True
[2]: x is y == False
[3]: x is y == False
[4]: x is y == False
[5]: x is y == False
[6]: x is y == False
[7]: x is y == False
[8]: x is y == False
[9]: x is y == False
[10]: x is y == False
[11]: x is y == False
[12]: x is y == False
[13]: x is y == False
[14]: x is y == False
[15]: x is y == False
[16]: x is y == False
[17]: x is y == False
[18]: x is y == False
[19]: x is y == False
[20]: x is y == False
[21]: x is y == False
[22]: x is y == False
[23]: x is y == False
[24]: x is y == False
```

当整数在某个范围内时，他们使用同一个对象，超出范围就会分为两个不同对象，在同一行代码里面分配他们却使用的是同一个对象，在 for 循环里面都使用同一个对象

```python
>>> x = 20
>>> y = 20
>>> x is y
True
>>> x = 20000
>>> y = 20000
>>> x is y
False
>>> x = 20000; y = 20000;
>>> x is y
True
>>> for i in range(2000):
...   x = i
...   y = i
...   if not x is y:
...     print i
...
```
