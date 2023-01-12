---
title: "Java 中的 charAt()–如何使用 Java charAt() 方法"
author: "axiaoxin"
date: 2023-01-12T12:04:31+08:00
subtitle: ""
image: ""
tags: ["java"]
slug: ""
---

Java 中的 `charAt()` 方法返回字符串中给定或指定索引处字符的 `char` 值。

在本文中，我们将从 `charAt()` 方法的语法开始，然后通过几个示例来了解如何使用它。

## 如何使用 Java 的 `charAt()` 方法

下面是 `charAt()` 方法的语法:

```java
public char charAt(int index)
```

请注意，使用 `charAt()` 方法从字符串返回的字符具有 `char` 数据类型。在本文后面，我们将看到这是如何影响返回值的串联（concatenate）。

现在我们来看一些例子。

```java
public class Main {
  public static void main(String[] args) {

    String greetings = "Hello World";

    System.out.println(greetings.charAt(0));
    // H
  }
}
```

在上面的代码中，我们的字符串“Hello World”存储在一个名为 greetings 的变量中。我们使用 `charAt()` 方法获取索引 0 处的字符，即 `H`。

第一个字符的索引始终为 0，第二个字符的索引始终为 1，依此类推。子字符串之间的空格也算作一个索引。

在下一个示例中，我们将看到当我们尝试连接返回的不同字符时会发生什么。串联（concatenate）意味着将两个或多个值连接在一起（在大多数情况下，此术语用于连接字符串中的字符或子字符串）。

```java
public class Main {
  public static void main(String[] args) {
    String greetings = "Hello World";

    char ch1 = greetings.charAt(0); // H
    char ch2 = greetings.charAt(4); // o
    char ch3 = greetings.charAt(9); // l
    char ch4 = greetings.charAt(10); // d

    System.out.println(ch1 + ch2 + ch3 + ch4);
    // 391
  }
}
```

使用 `charAt()` 方法，我们得到了索引为 0、4、9 和 10 处的字符，分别为 H、o、l 和 d。

然后我们尝试打印并连接这些字符：`System.out.println(ch1 + ch2 + ch3 + ch4);` 。

但是没有输出“Hold”返回给我们，而是输出了 391。发生这种情况是因为返回值不再是字符串，而是数据类型为 `char`。因此，当我们连接它们时，解释器会添加它们的 ASCII 值。

H 的 ASCII 值为 72，o 的值为 111，l 的值为 108，d 的值为 100。当我们将它们相加时，我们得到上一个示例中返回的 391。

## StringIndexOutOfBoundsException 错误

当我们传入的索引号超过字符串中的字符数时，我们会在控制台中收到 StringIndexOutOfBoundsException 错误。

此错误也适用于使用 Java 不支持的负索引。在 Python 等支持负索引的编程语言中，传入 -1 将为您提供数据集中的最后一个字符或值，类似于 0 始终返回第一个字符。

这里有一个例子:

```java
public class Main {
  public static void main(String[] args) {
    String greetings = "Hello World";

    char ch1 = greetings.charAt(20);

    System.out.println(ch1);

    /* Exception in thread "main" java.lang.StringIndexOutOfBoundsException: String index out of range: 20
    */
  }
}
```

在上面的代码中，我们传入了一个 20 的索引： `char ch1 = greetings.charAt(20);` 这超出了 greetings 变量中的字符数，所以我们收到了一个错误。您可以在上面的代码块中看到注释的错误信息。

类似地，如果我们像这样传入一个负值：`char ch1 = greetings.charAt(-1);`，我们会得到一个类似的错误。

## 总结

在本文中，我们学习了如何在 Java 中使用 `charAt()` 方法。我们看到了如何根据索引号返回字符串中的字符，以及连接这些字符时会发生什么。

最后，我们讨论了一些在 Java 中使用 `charAt()` 方法时会收到错误响应的实例。

祝您编码愉快！
