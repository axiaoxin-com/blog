---
title: "Golang JSON 序列化时 HTML 特殊字符转义问题分析"
author: "axiaoxin"
date: 2021-11-21T20:19:45+08:00
subtitle: ""
image: ""
tags: ["golang"]
slug: ""
---

## 场景复现

在 API 实现中返回一个 JSON 结果，其中有一个字段为 URL 链接，客户端拿到该链接后做请求，URL 链接中存在多个使用 `&` 连接的 querystring 参数。

服务端实现时，通过构造结构体后返回对应的 JSON 序列化结果。

但是请求接口时发现 URL 链接中的 `&` 符号被自动转义为 `\u0026`，导致历史版本的客户端无法解析 URL 中的参数。

一段代码模拟该场景：

```golang
package main

import (
    "encoding/json"
    "fmt"
)

type Data struct {
    Link string
}

func main() {

    link := `https://mbti.axiaoxin.com/?lang=ko&from=article`
    data := Data{
        Link: link,
    }
    b, _ := json.Marshal(data)
    fmt.Println("data json:", string(b))
    fmt.Println("---")
}
```


代码执行结果输出：

```text
data json: {"Link":"https://mbti.axiaoxin.com/?lang=ko\u0026from=article"}
---
```

其中的 `&` 变成了 `\u0026`。

## 原因分析

Golang 在 json 序列化时，默认会对特殊的 html 字符进行转义处理。

`json.Marshal` 的源码实现：

```golang
func Marshal(v interface{}) ([]byte, error) {
    e := newEncodeState()

    err := e.marshal(v, encOpts{escapeHTML: true})
    if err != nil {
        return nil, err
    }
    buf := append([]byte(nil), e.Bytes()...)

    encodeStatePool.Put(e)

    return buf, nil
}
```

其中 `e.marshal` 通过 `escapeHTML: true` 指定了 encode 时需要做 HTMLEscape 处理，即对 HTML 特殊字符进行转义。

该方法中通过判断需要序列化的对象类型，来使用对应的 encoderFunc 来做序列化。

这里我们序列化的对象是结构体，因此会调用 `structEncoder` 的 encode 方法来处理：

![Golang JSON Marshal 源码分析](https://user-images.githubusercontent.com/2876405/142756709-51e495ff-a2b3-4a1e-8130-4a0b858ce704.png)

可以看到 encode 方法除了会对结构体字段的值做 escape 处理外，对结构体的 json tag 名也会做 escape 处理，实验一下：

```golang
package main

import (
    "encoding/json"
    "fmt"
)

type Data struct {
    Link string `json:"Li&nk"`
}

func main() {

    link := `https://mbti.axiaoxin.com/?lang=ko&from=article`
    data := Data{
        Link: link,
    }
    b, _ := json.Marshal(data)
    fmt.Println("data json:", string(b))
    fmt.Println("---")
}
```

运行输出：

```text
data json: {"Li\u0026nk":"https://mbti.axiaoxin.com/?lang=ko\u0026from=article"}
---
```

对于结构体的每个字段，structEncoder 都会将其转换为 `field` 对象：


```golang
// A field represents a single field found in a struct.
type field struct {
    name      string
    nameBytes []byte                 // []byte(name)
    equalFold func(s, t []byte) bool // bytes.EqualFold or equivalent

    nameNonEsc  string // `"` + name + `":`
    nameEscHTML string // `"` + HTMLEscape(name) + `":`

    tag       bool
    index     []int
    typ       reflect.Type
    omitEmpty bool
    quoted    bool

    encoder encoderFunc
}
```

结构体字段值通过自身类型的 `encoderFunc` 进行处理，这里我们是 string 类型，因此将调用 `stringEncoder` 对值进行处理：

![Golang JSON Marshal 源码分析](https://user-images.githubusercontent.com/2876405/142756952-6d035cb3-44d5-4d94-b9e1-b11c77850b0d.png)

stringEncoder 中 调用 string 方法对字符串中 `RuneSelf` 以下的字符（ASCII）进行 escape 处理，其中就对 `&` 进行了替换：

![Golang JSON Marshal 源码分析](https://user-images.githubusercontent.com/2876405/142757132-67a856e4-9820-4e87-8293-de8f1a9f4aca.png)

`htmlSafeSet`是一个 html 特殊符号是否安全的 map，`&` 符号返回 false, 因此进行了 hex 的运算被替换。

## 解决方案

解决方法 golang 在源码中已说明白：

```golang
// String values encode as JSON strings coerced to valid UTF-8,
// replacing invalid bytes with the Unicode replacement rune.
// So that the JSON will be safe to embed inside HTML <script> tags,
// the string is encoded using HTMLEscape,
// which replaces "<", ">", "&", U+2028, and U+2029 are escaped
// to "\u003c","\u003e", "\u0026", "\u2028", and "\u2029".
// This replacement can be disabled when using an Encoder,
// by calling SetEscapeHTML(false).
```

就是说，可以通过 `json.NewEncoder` 创建一个新的 encoder，再对该 encoder 设置 `SetEscapeHTML(false)` 封装除自定义的 encoder。

```golang
// NewEncoder returns a new encoder that writes to w.
func NewEncoder(w io.Writer) *Encoder {
    return &Encoder{w: w, escapeHTML: true}
}
```

可见 NewEncoder 默认也是设置 `escapeHTML: true`，但是它提供了 SetEscapeHTML 方法来修改这个设置，然后调用他的 `Encode` 方法进行序列化。

这里需要注意 NewEncoder 的 Encode 方法会在 JSON 序列化结果后添加一个 `\n`:

![Golang JSON Marshal 源码分析](https://user-images.githubusercontent.com/2876405/142757522-2a50c315-e9b3-4de1-9771-62b796d09732.png)

代码实现：

```golang
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
)

type Data struct {
    Link string `json:"Li&nk"`
}

func Encoding(t interface{}) ([]byte, error) {
    buffer := &bytes.Buffer{}

    encoder := json.NewEncoder(buffer)
    encoder.SetEscapeHTML(false)
    err := encoder.Encode(t)
    return buffer.Bytes(), err
}

func main() {

    link := `https://mbti.axiaoxin.com/?lang=ko&from=article`
    data := Data{
        Link: link,
    }
    b, _ := json.Marshal(data)
    fmt.Println("data json:", string(b))
    fmt.Println("---")

    b, _ = Encoding(data)
    fmt.Println("data json:", string(b))
    fmt.Println("---")
}
```

执行结果：

```text
data json: {"Li\u0026nk":"https://mbti.axiaoxin.com/?lang=ko\u0026from=article"}
---
data json: {"Li&nk":"https://mbti.axiaoxin.com/?lang=ko&from=article"}

---
```

注意，第二个打印多了一个空行。

## echo 框架中自定义 JSON 序列化方法

在 web 框架中，比如 echo，其 New 方法实现：

![echo 框架中自定义 JSON 序列化方法](https://user-images.githubusercontent.com/2876405/142757711-37dac28c-e8fe-4172-a766-14e4538c6aca.png)

其中的 JSONSerializer 没有设置 escape 为 false，因此使用 echo 做开发，返回的 json 都会出现这种问题：

![echo 框架中自定义 JSON 序列化方法](https://user-images.githubusercontent.com/2876405/142757753-53bb6e8a-f910-4d6d-8dc7-0c612c32add2.png)

复制 JSONSerializer 的代码，改为自己的 MyJSONSerializer ，在 `Serialize` 方法中添加 `enc.SetEscapeHTML(false)` 来实现不 escape。

在调用 echo 的 New 方法后，再把他的JSONSerializer用我们新的 JSONSerializer 覆盖即可。

代码示例：

```golang
e := echo.New()
e.JSONSerializer = &MyJSONSerializer{}
```
