---
title: "Golang 并发 append slice 时的并发安全问题分析与总结"
author: "axiaoxin"
type: ""
date: 2021-05-13
subtitle: ""
image: ""
tags: ["golang", "并发"]
---

## 背景

导出数据库中的数据，由于数据量巨大且查询复杂，完成导出的时间很长，因此通过将并发查询然后将结果合并到一起。


在导出的数据量只有30多万时，导出的记录数和sql count出的数量一致；
但当导出超过数据量超过百万时，导出的数据量变少了。

此时意识到是忽略了并发问题，将 append 操作加上锁后验证一切正常。

## 代码示例

通常对 map 的并发读写问题很容易发现，因为一旦并发的更新一个 map 时，golang 会 panic ，单纯的并发读没有问题。
对 slice 进行 append 时也有并发安全问题，但是由于不会 panic 因此很容易被忽略，一旦没有对比校验实际数据量，很难发现问题。

在使用上并发更新 map 的安全问题很直观，同时更新相同的 key 结果无法预料，但是并发 append slice 的安全问题相对就没有那么直观了。

回顾一下 map 的并发读写，在并发很小的时候比如10个以内，也不是每次都会 panic，比如：

```golang
package main

import (
    "sync"
    "testing"
)

func TestMap(t *testing.T) {
    m := map[int]int{}
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(i int) {
            if _, exists := m[i]; !exists {
                m[i] = i
            }
            wg.Done()
        }(i)
    }
    wg.Wait()

    t.Log(m)
}
```

执行单元测试，有可能要执行 10 多次才会有 1 次 panic：

```shell
go test -v -run TestMap
=== RUN   TestMap
    map_test.go:23: map[0:0 1:1 2:2 3:3 4:4 5:5 6:6 7:7 8:8 9:9]
--- PASS: TestMap (0.00s)
PASS
```

调大并发数到 1000 以上则几乎 100% panic 。

为方便快速验证，添加 `-race` 参数：

```shell
go test -v -race -run TestMap
=== RUN   TestMap
==================
WARNING: DATA RACE
Read at 0x00c00002c330 by goroutine 9:
    ...
==================
    map_test.go:23: map[0:0 1:1 2:2 3:3 4:4 5:5 6:6 7:7 8:8 9:9]
    testing.go:1093: race detected during execution of test
--- FAIL: TestMap (0.00s)
=== CONT
    testing.go:1093: race detected during execution of test
FAIL
```

golang map 主要是使用 hmap 和 bmap 两个结构实现的哈希表， map 的操作因为考虑到使用场景和性能问题，没有实现为原子操作，参考官方文档 FAQ: <https://golang.org/doc/faq#atomic_maps>

map 底层实现：

```text
map[int]int{} -> hmap -> []bmap -> tophash
                         |           |
                     *overflow     []key
                         |           |
                         []bmap      []value
```

map 实现原理参考:<https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/>

再来看并发 append slice 的情况，使用 10 个 goroutine 并发 append 10000 个数字到 slice s 中，最终 s 正确长度为 100000

测试代码：

```golang
package main

import (
    "sync"
    "testing"
)

func TestSlice(t *testing.T) {
    s := []int{}
    var wg sync.WaitGroup

    // 外部变量记录每个 goroutine append 的数量
    count := 0
    // 10 个 goroutine 并发 append 10000 个数字到 slice s 中，最终 s 正确长度为 10 * 10000 = 100000
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(i, count int) {
            for j := 0; j < 10000; j++ {
                s = append(s, j)
                count++
            }
            t.Logf("G%d append count:%d\n", i, count)
            wg.Done()
        }(i, count)
    }
    wg.Wait()

    if len(s) != 100000 {
        t.Errorf("s.len:%d != 100000", len(s))
    }
}
```

同样，循环次数如果很少时也有可能是正常的结果，数量较大时就能明显观察到错误。

执行测试：

```shell
go test -v -race -run TestSlice
=== RUN   TestSlice
==================
WARNING: DATA RACE
Read at 0x00c00000e048 by goroutine 10:
    ...
...
==================
    slice_test.go:22: G9 append count:10000
    slice_test.go:22: G0 append count:10000
    slice_test.go:22: G3 append count:10000
    slice_test.go:22: G2 append count:10000
    slice_test.go:22: G8 append count:10000
    slice_test.go:22: G4 append count:10000
    slice_test.go:22: G6 append count:10000
    slice_test.go:22: G5 append count:10000
    slice_test.go:22: G1 append count:10000
    slice_test.go:22: G7 append count:10000
    slice_test.go:29: s.len:26101 != 100000
    testing.go:1093: race detected during execution of test
--- FAIL: TestSlice (0.28s)
=== CONT
    testing.go:1093: race detected during execution of test
FAIL
```

在 append 出加锁后，修复问题：

 ```golang
package main

import (
    "sync"
    "testing"
)

func TestSlice(t *testing.T) {
    s := []int{}
    var wg sync.WaitGroup
    // 锁
    var mu sync.Mutex

    // 外部变量记录每个 goroutine append 的数量
    count := 0
    // 10 个 goroutine 并发 append 10000 个数字到 slice s 中，最终 s 正确长度为 10 * 10000 = 100000
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(i, count int) {
            for j := 0; j < 10000; j++ {
                // append slice 加锁解决并发安全问题
                mu.Lock()
                s = append(s, j)
                mu.Unlock()
                count++
            }
            t.Logf("G%d append count:%d\n", i, count)
            wg.Done()
        }(i, count)
    }
    wg.Wait()

    if len(s) != 100000 {
        t.Errorf("s.len:%d != 100000", len(s))
    }
}
 ```

运行测试：

```shell
go test -v -race -run TestSlice
=== RUN   TestSlice
    slice_test.go:25: G6 append count:10000
    slice_test.go:25: G0 append count:10000
    slice_test.go:25: G9 append count:10000
    slice_test.go:25: G5 append count:10000
    slice_test.go:25: G8 append count:10000
    slice_test.go:25: G7 append count:10000
    slice_test.go:25: G2 append count:10000
    slice_test.go:25: G3 append count:10000
    slice_test.go:25: G1 append count:10000
    slice_test.go:25: G4 append count:10000
--- PASS: TestSlice (0.05s)
PASS
```


## 问题分析

先说结论：因为并发的 append 操作的是同一个底层数组，导致同一个数组下标的元素被多次覆盖。

slice 底层是使用数组保存数据，数组是一段连续的内存空间

```text
[]int{} -> *array -> [连续内存空间]
            |
            len
            |
            cap
```


slice 实现原理参考：<https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-array-and-slice/>

append 处理流程：

```golang
// append(slice, 1, 2, 3)
ptr, len, cap := slice
newlen := len + 3
if newlen > cap {
    ptr, len, cap = growslice(slice, newlen)
    newlen = len + 3
}
*(ptr+len) = 1
*(ptr+len+1) = 2
*(ptr+len+2) = 3
return makeslice(ptr, newlen, cap)
```

在程序中先声明了一个 slice s 然后并发的不断向其中添加元素，
append 追加元素实际是将每个元素依次赋值给对应的数组中的内存指针（即将内存空间的指针指向对应的元素），
并发情况下，所有 goroutine 都操作的时同一个数组，同一个指针可能多次指向了不同的元素，最后导致元素个数和预期不一致。

