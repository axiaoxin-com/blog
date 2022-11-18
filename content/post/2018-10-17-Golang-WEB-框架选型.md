---
title: "Golang WEB 框架选型"
author: "axiaoxin"
date: 2018-10-17T17:22:52+08:00
subtitle: ""
image: ""
tags: ["golang"]
slug: ""
---

Go 本身默认就已经具备了 WEB 开发的特性，`net/http` 包 + httprouter 开发一个 apiserver 已经足够了，
写好包含 `http.ResponseWriter` 和 `*http.Request` 参数的业务处理函数，通过 `http.HandleFunc` 注册路由就可以了。

近期做项目需要选一个 Go 的 WEB 框架来业务统一使用，所以记录一下选择理由。

当前流行的框架：结合 github 关键字 `go web` `go web framework` 搜索和 `awesome-go` 中提及的框架，选出了当前比较 start 数较多且最近一直在更新的框架

按start数排名：

1. gin 21.2k
2. beego 17.4k
3. iris 12.1k
4. echo 11.9k
5. revel 10.4k

性能上 gin、iris、echo 网上是给的数据都是五星，beego 三星，revel 两星。

echo 最新的测试数据显示它比 gin 更快一点，gin 在编译时可以指定使用 jsoniter 来处理 json，echo 不支持，在处理 json 的服务中，性能猜测是 gin 更好。

在 helloworld 的写法上 gin、iris、echo 写法几乎都一模一样。

iris 从 readme 看更加花哨，功能要支持的要多一点，但是这些功能个人认为没有也不影响。

beego和revel都比较类似于 Python 的 Django，个人不太喜欢这种较重的框架，学习成本相对较高，而且集成一堆永远用不到的东西。

beego是国产，有中文文档；gin的文档几乎是没有，全靠readme；echo有专门的文档网站。

据说 iris 作者由于使用别人代码删 license，awesome-go 上被除名，看了几篇国外博客在比较这些框架是特意指出不要使用 iris，这样可能会导致给它贡献代码的人可能会比较少，iris 用到的依赖库很多，网上看到很多人在编译的时候遇到这种错误；gin 主要是两个大学生写的，主要活跃在寒暑假期间。

个人比较喜欢 echo 和 gin 这两个框架，都比较轻量，开发快速，拼装能力强。据说 echo 在路由的报错处理上并不是很友好，有冲突也没有报错，排查困难，中间件不少。gin 出错信息详尽，中间件也比较丰富，但是路由匹配有一些限制。

所以，**根据 start 数，性能，易用程度，社区活跃度和具体应用场景来选择** 的话，当前我更加倾向于使用 gin 作为基础开发框架，可以获得比使用原生包更高的开发效率和执行效率，组织好代码结构，按需加入依赖包即可。

