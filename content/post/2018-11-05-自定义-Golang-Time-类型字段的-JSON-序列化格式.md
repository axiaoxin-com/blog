---
title: "自定义 Golang Time 类型字段的 JSON 序列化格式"
author: "axiaoxin"
date: 2018-11-05T19:18:32+08:00
subtitle: ""
image: ""
tags: ["golang"]
slug: ""
---

## Golang 中自定义 time.Time 类型的 JSON 序列化格式

**Golang 中时间格式定义**：

```golang
const (
    ANSIC       = "Mon Jan _2 15:04:05 2006"
    UnixDate    = "Mon Jan _2 15:04:05 MST 2006"
    RubyDate    = "Mon Jan 02 15:04:05 -0700 2006"
    RFC822      = "02 Jan 06 15:04 MST"
    RFC822Z     = "02 Jan 06 15:04 -0700" // RFC822 with numeric zone
    RFC850      = "Monday, 02-Jan-06 15:04:05 MST"
    RFC1123     = "Mon, 02 Jan 2006 15:04:05 MST"
    RFC1123Z    = "Mon, 02 Jan 2006 15:04:05 -0700" // RFC1123 with numeric zone
    RFC3339     = "2006-01-02T15:04:05Z07:00"
    RFC3339Nano = "2006-01-02T15:04:05.999999999Z07:00"
    Kitchen     = "3:04PM"
    // Handy time stamps.
    Stamp      = "Jan _2 15:04:05"
    StampMilli = "Jan _2 15:04:05.000"
    StampMicro = "Jan _2 15:04:05.000000"
    StampNano  = "Jan _2 15:04:05.000000000"
)
```

当我们的 Golang 代码将一个包含 time.Time 类型字段的结构体序列化为 JSON 时，默认是以 [RFC3339](https://github.com/golang/go/blob/master/src/time/format.go#L78) 的格式输出。

序列化后的 JSON 中，所有的 time 类型的字段都是 `2006-01-02T15:04:05Z07:00` 这种格式。

究其原因是 go 在 time 包中实现 [`json.Marshaler`](https://github.com/golang/go/blob/master/src/time/time.go#L1252) 接口时，指定了使用 RFC3339Nano 这种格式

```golang
// MarshalJSON implements the json.Marshaler interface.
// The time is a quoted string in RFC 3339 format, with sub-second precision added if present.
func (t Time) MarshalJSON() ([]byte, error) {
    if y := t.Year(); y < 0 || y >= 10000 {
        // RFC 3339 is clear that years are 4 digits exactly.
        // See golang.org/issue/4556#c15 for more discussion.
        return nil, errors.New("Time.MarshalJSON: year outside of range [0,9999]")
    }

    b := make([]byte, 0, len(RFC3339Nano)+2)
    b = append(b, '"')
    b = t.AppendFormat(b, RFC3339Nano)
    b = append(b, '"')
    return b, nil
}
```

当 json 进行 Marshal 时，time.Time 类型的字段会使用这个 Time.MarshalJSON 进行序列化。

要想自定义 time.Time 类型字段的 JSON 序列化格式需要重写这个接口方法。

> 参考：[如何向 Go 中的现有类型添加新方法？](https://stackoverflow.com/questions/28800672/how-to-add-new-methods-to-an-existing-type-in-go)

由于 go 中不允许在包外新增或重写方法，否则可能会报错：

```text
cannot define new methods on non-local type[s]
```

因此，我们需要通过在外部定义别名或者内嵌结构体，再在这上面新加或重写方法。

别名方式：

```golang
type JSONTime time.Time
```

内嵌方式（推荐）：

```golang
type JSONTime struct {
    time.Time
}
```

需要注意别名方式只能使用原始类型的字段，不能使用其方法，只重写字段的时候可以考虑使用该方式。推荐使用内嵌方式，原始类型的字段和方法都可以被使用。

因此，我们只需定义一个内嵌 time.Time 的结构体，并重写 MarshalJSON 方法，然后在定义结构体时，把 time.Time 类型替换为我们自己的类型即可。

参考代码：

```golang
package main

import (
	"encoding/json"
	"fmt"
	"time"
)

type MyStruct struct {
	CreatedAt time.Time
}

type JSONTime struct {
	time.Time
}

// MarshalJSON 覆盖 time.Time 的 MarshalJSON
func (t JSONTime) MarshalJSON() ([]byte, error) {
	formatted := fmt.Sprintf("\"%s\"", t.Format("2006-01-02 15:04:05"))
	return []byte(formatted), nil
}

type MyStruct2 struct {
	CreatedAt JSONTime
}

func main() {

	ms := MyStruct{
		CreatedAt: time.Now(),
	}
	m, _ := json.Marshal(ms)
	fmt.Println("time.Time: ", string(m))

	ms2 := MyStruct2{
		CreatedAt: JSONTime{time.Now()},
	}
	m2, _ := json.Marshal(ms2)
	fmt.Println("JSONTime: ", string(m2))
}
```

输出结果：

```text
time.Time:  {"CreatedAt":"2009-11-10T23:00:00Z"}
JSONTime:  {"CreatedAt":"2009-11-10 23:00:00"}
```

## 自定义 Gorm Model 中 time.Time 类型字段的 JSON 序列化格式

在 gorm 中使用覆盖 MarshalJSON 的方式来自定义时间字段的格式时，只重写 MarshalJSON 是不够的，只写这个方法会在写数据库的时候会提示 delete_at 字段不存在，还需要加上对 database/sql 中的 Value 和 Scan 方法的实现。


实现自定义格式化的 time 类型 JSONTime 为：

```golang
package main
import (
    "database/sql/driver"
    "fmt"
    "time"
)

// JSONTime format json time field by myself
type JSONTime struct {
    time.Time
}

// MarshalJSON on JSONTime format Time field with %Y-%m-%d %H:%M:%S
func (t JSONTime) MarshalJSON() ([]byte, error) {
    formatted := fmt.Sprintf("\"%s\"", t.Format("2006-01-02 15:04:05"))
    return []byte(formatted), nil
}

// Value insert timestamp into mysql need this function.
func (t JSONTime) Value() (driver.Value, error) {
    var zeroTime time.Time
    if t.Time.UnixNano() == zeroTime.UnixNano() {
        return nil, nil
    }
    return t.Time, nil
}

// Scan valueof time.Time
func (t *JSONTime) Scan(v interface{}) error {
    value, ok := v.(time.Time)
    if ok {
        *t = JSONTime{Time: value}
        return nil
    }
    return fmt.Errorf("can not convert %v to timestamp", v)
}
```

> 参考 gorm issue：[我想自定义模型json输出时间格式](https://github.com/go-gorm/gorm/issues/1611)

在定义 gorm model 时，时间类型的字段使用我们自己实现的 JSONTime 类型声明即可。

也可以重新定义 gorm 的 BaseModel：

```golang
package main

type BaseModel struct {
    ID        int            `gorm:"primary_key" json:"id"`
    CreatedAt JSONTime  `json:"createdAt"`
    UpdatedAt JSONTime  `json:"updatedAt"`
    DeletedAt JSONTime `sql:"index" json:"-"`
}
```

在定义 gorm model 时，不再内嵌 gorm.Model，而是使用我们自己的 BaseModel，这样在接口中返回的 JSON 中时间类型的字段就是我们自定义的格式了。
