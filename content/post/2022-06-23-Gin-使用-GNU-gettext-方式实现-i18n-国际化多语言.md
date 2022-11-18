---
title: "Gin 使用 GNU gettext 方式实现 i18n 国际化多语言"
author: "axiaoxin"
date: 2022-06-23T10:50:19+08:00
subtitle: ""
image: ""
tags: ["golang"]
slug: ""
---

i18n 国际化多语言本质上就是先写好一堆映射，在根据想要的语言取对应的文字。

Golang 的 i18n 多语言方案网上查了一下，文章都讲的不太细致，而且代码看起来也不太好理解。

之前写 Python 代码时有使用过 [pybabel](https://babel.pocoo.org/en/latest/cmdline.html) 做多语言集成，通过命令生成 pot、po、mo 等文件，然后代码中通过函数取msgid就能动态获取到指定的语言，pybabel 实现原理其实就是对 [GNU gettext](https://www.gnu.org/software/gettext/manual/gettext.html) 进行了封装。

gettext 是专门用来做多语言的，它由一系列命令行工具，比如 xgettext、msginit、msgmerge、msgfmt 等，像 linux 和 mac 默认都自带安装好了。

pybabel 使用流程主要是先提取代码中需要设置多语言的文字生成一个 .pot 格式的模板文件，然后根据这个模板创建对应语言的 .po 翻译文件，然后把 .po 文件编译成 .mo 文件就可以被函数动态读取。

当代码中的多语言文字修改或新增了，就需要再次生成一下新的 pot 模板文件，把新增的多语言文字加到模板中，然后讲新模板和之前翻译好的 po 文件进行合并，合并后原来的 po 文件会将本次新增的多语言加入进来，然后对其进行翻译，完成后编译新的 mo 文件。

以上这个流程在 gettext 命令工具中分别需要使用 xgettext 提取代码中的多语言生成模板、msginit 创建 po 文件、msgfmt 编译为 mo 文件、msgmerge 合并新的 po 文件。当拥有 po、mo 文件时，就能使用多种支持的语言进行读取了。

由于比较熟悉这一套流程，因此研究了一下在 golang 和其 web 框架比如 gin 中如何使用 gettext 实现多语言集成，这里记录一下要点。

golang 的 gettext 库也有好几个，这里我选择使用 [gettext-go](https://github.com/chai2010/gettext-go) ，代码比较简洁而且支持 embed。唯一折腾了半天的是 mo 文件的路径问题，必要按固定规则才行，尝试了很多遍才成功。


## 在 Golang 中使用 gettext-go 实现 i18n 多语言

大致步骤如下：

1、 生成你的 po、mo 文件。

2、 必须先绑定你的 mo 翻译文件：`gettext.BindLocale(gettext.New("domain", "path"))`

这里的 domain 和 path 一定要注意命名，domain 你可以记忆为 mo 文件的名称，path 是存放你所有多语言的父目录路径，整个路径为 `path/xx/LC_MESSSAGES/yy.mo`，

其中 path 就是 New 方法中的参数路径，xx 为任意名字标识语言，LC_MESSSAGES 路径必须要，yy 为 domain 参数值，只有这样才能读取到。

3、 完成绑定后调用`gettext.SetLanguage("xx")`指定你要取的目标语言

4、 再调用`gettext.Gettext` 传入 po 文件中 msgid 的值就能返回对应语言的 msgstr

## 在 Gin 中使用 gettext-go 实现 i18n 多语言


golang中获取多语言成功了，在gin中就容易了。可以使用中间件获取用户指定的获取可能的语言来设置目标语言，代码中的返回都使用gettext.Gettext获取即可，需要注意的时gettext.Gettext的参数必须有对应的翻译文字才行。

获取用户的语言首先用户可以通过url参数指定语言，一旦指定语言，就将该语言保存到 cookie 中，下次从 cookie 获取，如果 cookie 没有则获取用户请求头中的 Accept-Language，可以使用 `golang.org/x/text/language` 这个包提供的 `ParseAcceptLanguage` 方法获取最匹配的语言作为用户语言。

中间件实现可参考 [pink-lady GinSetLanguage中间件设置i18n语言](https://github.com/axiaoxin-com/pink-lady/blob/master/webserver/gin_middlewares.go#L148)。

如果要在html模板中替换多语言，可以将gettext.Gettext注册到gin的自定义模板方法中，然后在html中所有需要被翻译的地方都使用这个模板方法处理一下，这样就能得到多语言。

模板注册参考 [pink-lady模板方法注册](https://github.com/axiaoxin-com/pink-lady/blob/master/webserver/gin.go#L51)

**可能存在的问题**：

gettext-go 中默认是使用了一个全局变量来处理逻辑，一次在我们的实现中可能会有并发问题，当一个请求处理耗时较长时，可能导致语言错乱，比如，当请求为英语，执行代码过程中有人请求中文，原本英文的请求使用了该全局变量就变成了中文。

由于我当前的应用场景可以接受这种情况，因此暂未处理该问题，可以通过将 gettext-go 使用的全局变量想办法放入到请求 context 中，通过使用 context 中的对象进行语言获取。

## xgettext 自动提取代码中的待翻译的文字生成 pot 模板

通过 xgettext 无法直接提取 golang 代码和 golang html 模板中的翻译文字，解决办法是 sed 替换为 xgettext 能识别的字符，然后再提取。

比如 html 模板中的模板方法写法 `{{ _text "翻译我"}}` 无法被识别，可以替换为 c 语言版本的 `gettext("翻译我")` 后在提取为html.pot：


```sh
find . -name "*.html"
  | xargs perl -pe "s/{{\s*_text [\"\`](.+?)[\"\`]\s*}}/{{ gettext(\"\1\") }}/g"
    | xgettext --no-wrap --no-location --language=
      │ c --from-code=UTF-8 --output=html.pot -
```

go 文件中也无法识别，同样操作替换后再提取为 go.pot

```sh
find . -name "*.go"
  | xargs perl -pe "s/gettext.Gettext/gettext/g"
    | xgettext --no-wrap --no-location --language=c --from-code=UTF-8 --output=go.pot -
```

然后再把两个 pot 合并为一个 pot 文件（messages.pot）：

```sh
xgettext --no-wrap --no-location *.pot -o messages.pot
```

最后我们统一使用这个 pot 生成各种语言的翻译文件后编译即可。
