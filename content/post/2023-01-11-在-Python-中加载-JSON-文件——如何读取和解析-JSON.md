---
title: "在 Python 中加载 JSON 文件——如何读取和解析 JSON"
author: "axiaoxin"
date: 2023-01-11T16:41:55+08:00
subtitle: ""
image: ""
tags: ["python"]
slug: ""
---

在本文中，您将学习如何在 Python 中读取和解析 JSON。

## 什么是 JSON？

JSON 是 JavaScript Object Notation 的缩写。这是一种将数据存储在“名称-值”对中的简单语法。只要值有效，它们可以是不同的数据类型。

JSON不可接受的类型包括函数、日期和 `undefined` 。

JSON 文件是以 `.json` 作为后缀名，存储了有效的 JSON 结构的文件。

JSON 文件的结构如下所示：

```json
{
  "name": "axiaoxin",
  "age": 16,
  "is_married": false,
  "profession": null,
  "hobbies": ["traveling", "coding"]
}
```

在 web 应用程序中，您经常使用 JSON 发送和接收来自服务器的数据。当接收到数据时，程序读取并解析JSON以提取特定数据。不同的语言有自己的方法。我们将在这里看看如何在 Python 中完成这些操作。


## 如何读取 JSON 文件

假设上面代码块中的 JSON 存储在 `user.json` 文件中。使用 Python 中的 `open()` 内置函数，我们可以读取该文件并将内容赋值给变量。方法如下：

```python
with open('user.json') as user_file:
    file_contents = user_file.read()

print(file_contents)
# {
#   "name": "axiaoxin",
#   "age": 16,
#   "is_married": false,
#   "profession": null,
#   "hobbies": ["travelling", "coding"]
# }
```

将文件路径传递给打开文件的 `open` 方法，并将文件中的流数据赋值给 `user_file` 变量。使用 `read` 方法，可以将文件的文本内容传给 `file_contents` 变量。

上面代码块的开头使用了 `with` ，这样在读取文件内容后，Python 就可以关闭该文件。

`file_contents` 现在是字符串形式的 JSON。下一步，您就可以开始解析 JSON 了。

## 如何解析 JSON

Python 拥有用于各种操作的内置模块。为了管理 JSON 文件，Python 有一个 json 模块。

这个模块有很多方法。其中之一是用于解析 JSON 字符串的 `loads()` 方法。可以将解析后的数据赋值给如下这样的变量：

```python
import json

with open('user.json') as user_file:
    file_contents = user_file.read()

print(file_contents)

parsed_json = json.loads(file_contents)
# {
#   'name': 'axiaoxin',
#   'age': 16,
#   'is_married': False,
#   'profession': None,
#   'hobbies': ['travelling', 'coding']
# }
```

使用 `loads()` 方法，可以看到 `parsed_json` 变量现在有一个有效的字典。从这个字典中，可以访问其中的键和值。

还要注意 JSON 中的 `null` 在 python 中转换为了 `None` 。这是因为 `null` 在 `Python` 中无效。


## 如何使用 `json.load()` 读取和解析 JSON 文件

`json` 模块还有 `load` 方法，你可以使用它来读取文件对象并同时解析它。使用此方法可以将前面的代码更新为：

```python
import json

with open('user.json') as user_file:
    parsed_json = json.load(user_file)

print(parsed_json)
# {
#   'name': 'axiaoxin',
#   'age': 16,
#   'is_married': False,
#   'profession': None,
#   'hobbies': ['travelling', 'coding']
# }
```

可以直接使用 `load` 方法进行读取文件对象的同时进行 JSON 解析来代替先使用文件对象的 `read` 方法然后再使用 `json` 模块的 `loads` 方法。

## 总结

JSON 数据通常以其简单的结构而闻名，并且在服务器和客户端之间的信息交换中非常流行（在大多数情况下是一种标准）。

不同的语言和技术可以以不同的方式读取和解析 JSON 文件。在本文中，我们学习了如何使用文件对象的 `read` 方法以及 `json` 模块的 `loads` 和 `load` 方法来读取 JSON 文件并解析此类文件。
