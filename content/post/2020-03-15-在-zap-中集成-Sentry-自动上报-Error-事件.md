---
title: "在 Zap 中集成 Sentry 自动上报 Error 事件"
author: "axiaoxin"
date: 2020-03-15T10:03:10+08:00
subtitle: ""
image: ""
tags: ["golang"]
slug: ""
---

在项目中发生了错误时我们都会打印 Error 级别的日志，但是即使有日志采集，在对发生 Error 时的告警通知和信息采集都不一定能快速且完善，目前对日志的告警也只是限于一些指标上的阈值告警，对于一些偶发或者非必现的 Error 其实我们还是很难及时发现。

使用 [Sentry](https://sentry.io) 可以解决这类痛点，在可能产生 Error 的地方我们除了打印 Error 日志，同时还会对严重的 Error 事件进行 Sentry 上报，Sentry 会记录详细的相关 issue 并告警通知。

在 Golang 的项目中我对易用性和性能及流行程度的综合考虑下选择使用 [zap](https://godoc.org/go.uber.org/zap) 作为日志组件来打印日志。

项目的 Error 日志必然是散落到各个角落，对于新写的代码，你可以在打 Error 的时候在加上一行 Sentry 的上报，对于已有的项目你可能需要全局搜索 Error 日志的打印再去人工添加代码，这样的操作显然是合适的。

经过对 [zap源码阅读](/post/2020-01-16-zap-源码阅读笔记/) 可以知道 zap 支持大多数其他日志组件都支持的 Hooks 回调和自定义 Core 的方式添加日志打印后的额外操作。那么我们可以使用这个机制，在打印日志时检测日志级别判断符合我们需要的 Error 以上级别的错误都自动上报的到 Sentry。

在 zap 的 Hooks 中有一个限制，就是我们在 zap 的 logger 中通常会添加一些 Fields 来固定打印一些额外的信息，比如 requestid，用户当前请求的 id，服务进程 id 等。

在 Hooks 的回调实现中，阅读源码会发现回调函数中不支持获取这些 Fields，只涉及对 Entry 的读取，Entry 即一条日志的基本信息包括Level、Time、LoggerName、 Message、Caller、Stack 等信息，
而 Fields 中的信息也都是我们所需要的，所以这里我不使用 Hooks 的方法来实现自动上报，而是使用 zap 的 Core 机制。

如果不需要 Fields 信息也可以使用 Hooks 来实现。Hook 函数签名为：`func(Entry) error`， 即接收一个 zap 的 Entry 作为参数，你可以在这个函数里面做你需要完成的任务后返回 error 即可，然后将 Hook 函数注册到 Logger 上即可。Hook 函数最终也是通过实现一个自定义 Core 的方式来把这些注册的 Hooks 都执行。

这里贴一下 Hooks 在 zap 中的用法：

```golang
package main

import (
	"fmt"

	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

// DemoHook 用于生成一个示例Hook函数，Hook函数实现当前日志大于Error级别时打印Message的长度
// 这里使用函数的方法返回一个Hook函数的好处时外层函数可以根据需要传入一些参数供Hook函数使用
func DemoHook() func(zapcore.Entry) error {
	return func(e zapcore.Entry) error {
		if e.Level < zapcore.ErrorLevel {
			return nil
		}

		fmt.Printf("message length:%d\n", len(e.Message))
		return nil
	}
}

func main() {
	logger := zap.NewExample()  // 生成一个logger
	logger = logger.WithOptions(zap.Hooks(DemoHook()))  // 将回调函数注册到logger生成一个新的logger
	logger.Info("123456789")  // 不打印长度信息
	logger.Error("123456789")  // 打印长度信息
}
```

执行结果：

```text
{"level":"info","msg":"123456789"}
{"level":"error","msg":"123456789"}
message length:9
```

可以看到定义 Hook 函数的时候无法接收到 Fields 的相关参数，所以需要 Fields 参数时，需要自定义Core。

zap 在执行最终日志打印的时候会将其 logger 中的所有 core 中的 Write 方法都遍历执行，因此我们只需要在自定义的 Core 中实现我们自己的操作即可，在这里需要实现对 Sentry 客户端的调用上报信息到 Sentry，最后将 core 添加到 logger 中即可。

在 zap 中 core 是一个接口，所以自定义 core 需要实现其定义的 5 个方法，相对于 Hooks 的实现较为复杂。

以下是一个 Sentry Core 的实现和使用示例：

```golang
// Package main is a demo for zapcore for sentry capture error

package main

import (
	"errors"
	"fmt"
	"time"

	"github.com/getsentry/sentry-go"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

// 将zap的Level转换为sentry的Level
func sentryLevel(lvl zapcore.Level) sentry.Level {
	switch lvl {
	case zapcore.DebugLevel:
		return sentry.LevelDebug
	case zapcore.InfoLevel:
		return sentry.LevelInfo
	case zapcore.WarnLevel:
		return sentry.LevelWarning
	case zapcore.ErrorLevel:
		return sentry.LevelError
	case zapcore.DPanicLevel:
		return sentry.LevelFatal
	case zapcore.PanicLevel:
		return sentry.LevelFatal
	case zapcore.FatalLevel:
		return sentry.LevelFatal
	default:
		return sentry.LevelFatal
	}
}

// SentryCoreConfig 定义 Sentry Core 的配置参数.
type SentryCoreConfig struct {
	Tags              map[string]string
	DisableStacktrace bool
	Level             zapcore.Level
	FlushTimeout      time.Duration
	Hub               *sentry.Hub
}

// sentryCore sentrycore的Core结构体，用于实现Core接口
type sentryCore struct {
	client               *sentry.Client    // sentry客户端
	cfg                  *SentryCoreConfig // core配置
	zapcore.LevelEnabler                   // LevelEnabler接口
	flushTimeout         time.Duration     // sentry上报的flush时间

	fields map[string]interface{} // 保存Fields
}

// With接口方法的实际实现，对传入fields进行设置日志打印时的打印解析方式并添加到已有的fields中
func (c *sentryCore) with(fs []zapcore.Field) *sentryCore {
	// Copy our map.
	m := make(map[string]interface{}, len(c.fields))
	for k, v := range c.fields {
		m[k] = v
	}

	// Add fields to an in-memory encoder.
	enc := zapcore.NewMapObjectEncoder()
	for _, f := range fs {
		f.AddTo(enc)
	}

	// Merge the two maps.
	for k, v := range enc.Fields {
		m[k] = v
	}

	return &sentryCore{
		client:       c.client,
		cfg:          c.cfg,
		fields:       m,
		LevelEnabler: c.LevelEnabler,
	}
}

// With 实现Core接口的With方法
func (c *sentryCore) With(fs []zapcore.Field) zapcore.Core {
	return c.with(fs)
}

// Check 实现Core接口的Check方法，只有大于在core配置中的的Level才会被打印
func (c *sentryCore) Check(ent zapcore.Entry, ce *zapcore.CheckedEntry) *zapcore.CheckedEntry {
	if c.cfg.Level.Enabled(ent.Level) {
		return ce.AddCore(ent, c)
	}
	return ce
}

// Write 实现Core接口的Write方法，对sentry进行上报，Fields作为Extra信息上报
func (c *sentryCore) Write(ent zapcore.Entry, fs []zapcore.Field) error {
	clone := c.with(fs)

	event := sentry.NewEvent()
	event.Message = ent.Message
	event.Timestamp = ent.Time.Unix()
	event.Level = sentryLevel(ent.Level)
	event.Platform = "demo"
	event.Extra = clone.fields
	event.Tags = c.cfg.Tags

	if !c.cfg.DisableStacktrace {
		trace := sentry.NewStacktrace()
		if trace != nil {
			event.Exception = []sentry.Exception{{
				Type:       ent.Message,
				Value:      ent.Caller.TrimmedPath(),
				Stacktrace: trace,
			}}
		}
	}

	hub := c.cfg.Hub
	if hub == nil {
		hub = sentry.CurrentHub()
	}
	_ = c.client.CaptureEvent(event, nil, hub.Scope())

	if ent.Level > zapcore.ErrorLevel {
		c.client.Flush(c.flushTimeout)
	}
	return nil
}

// Sync 实现Core接口的Sync方法
func (c *sentryCore) Sync() error {
	c.client.Flush(c.flushTimeout)
	return nil
}

// NewSentryCore 生成Core对象
func NewSentryCore(cfg SentryCoreConfig, sentryClient *sentry.Client) zapcore.Core {

	core := sentryCore{
		client:       sentryClient,
		cfg:          &cfg,
		LevelEnabler: cfg.Level,
		flushTimeout: 3 * time.Second,
		fields:       make(map[string]interface{}),
	}

	if cfg.FlushTimeout > 0 {
		core.flushTimeout = cfg.FlushTimeout
	}

	return &core
}

//  生成SentryCore对象并添加到Logger中
func main() {
	// 默认logger
	logger := zap.NewExample()

	// sentrycore配置
	cfg := SentryCoreConfig{
		Level: zap.ErrorLevel,
		Tags: map[string]string{
			"source": "demo",
		},
	}
	// 生成sentry客户端
	sentryClient, err := sentry.NewClient(sentry.ClientOptions{
		Dsn:              "http://abc:xyz@sentry.host.com/id",
		Debug:            true,
		AttachStacktrace: true,
	})
	if err != nil {
		fmt.Println(err)
	}
	// 生成sentryCore
	sCore := NewSentryCore(cfg, sentryClient)
	// 添加sentryCore到默认logger产生新的logger，使用该logger即可自动上报sentry
	logger = logger.WithOptions(zap.WrapCore(func(core zapcore.Core) zapcore.Core {
		return zapcore.NewTee(core, sCore)
	}))

	logger.Info("info log")
	logger.Error("this log will be auto captured by sentry", zap.String("f1", "v1"), zap.Error(errors.New("this ia an error")))
	time.Sleep(2 * time.Second) // sleep避免程序结束太快而导致上报失败
}
```

运行结果：

```text
[Sentry] 2020/03/15 17:46:10 Integration installed: ContextifyFrames
[Sentry] 2020/03/15 17:46:10 Integration installed: Environment
[Sentry] 2020/03/15 17:46:10 Integration installed: Modules
[Sentry] 2020/03/15 17:46:10 Integration installed: IgnoreErrors
{"level":"info","msg":"info log"}
{"level":"error","msg":"this log will be auto captured by sentry","f1":"v1","error":"this ia an error"}
[Sentry] 2020/03/15 17:46:10 Sending error event [81e966c6573e4134b3d8b842ea0fb738] to sentry.host.com project: id
```

sentry页面信息：

![sentry 页面截图](https://user-images.githubusercontent.com/2876405/76699191-f60fdc80-66e5-11ea-8591-3b06d37ff89c.png)
