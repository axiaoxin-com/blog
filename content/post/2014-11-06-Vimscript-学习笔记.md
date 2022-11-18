---
title: "Vimscript 学习笔记"
author: "axiaoxin"
date: 2014-11-06
subtitle: ""
image: ""
tags: ["vim"]
slug: ""
---

在 VIM 中输入`:help <CMD>` 可以查看该命令详细的帮助信息，比如`:help echo`、`:help python`。

VIM 脚本中的注释是以`"`进行注释的。

## VIM 打印信息

打印信息即可以使用`:echo "hi, axiaoxin."`，也可以使用`:echom "hi, Mr.axiaoxin"`，运行时都会输出后面的字符串，
他们的区别在于`:echom`会在运行后将其打印的信息保存在`messages`的列表中，而`:echo`则不会保存在内，
运行`:messages`可以查看 Vim 所有历史输出的信息，用`echom`可以更方便的进行调试信息的输出。

## 基本数据类型及强制类型转换

**数字**包括整型，浮点型，基本的数字运算和 python 都差不多

```vim
:echom 100
:echom 0xff "16进制
:echom 010  "8进制
:echom 019  "不合法8进制，自动被视为十进制
```

**字符串**不能使用`+`连接，`+`只用于数字，当用`+`连接字符串时，会强制将字符串转为开头的数字再相加。正确的连接符号为`.`

```vim
:echom "Hello, " + "world"  " return 0
:echom "3 mice" + "2 cats"  " return 5
:echom "3 mice" + "cats 2"  " return 3
:echom 10 + "10.1"          " return 10
:echom 10 + 10.1            " return 20.1

:echom "Hello, " . "world"  " return hello, world
```

当整数和字符串用`.`连接时，会将整数转换成字符串，但是不能将浮点数和字符串连接，会报错。

当字符串用单引号包住时，里面的特殊字符只会显示为原来的样子

```vim
:echo "foo\nbar"    "显示为两行
:echom "foo\nbar"   "显示为foo^@bar，^@表示换行字符
:echom '\n\\'       "显示为\n\\，失去本身的意义
:echom 'That''s enough.'    "显示为That's enough. 单引号需要用'来转义，用\是错误语法
```

字符串也可以像 python 一样切片`:echo "abcd"[0:2]`

用下标取字符串中的字符不能使用负数下标，
`:echo "abcd"[-1]`返回为空，但是可以使用负数进行切片`:echo "abcd"[-2:]`

**String 函数**

```vim
strlen(str)
len(str)
split(str[, spliter])
join(list[, joiner])
tolower(str)
toupper(str)
```

**列表**下标从 0 开始，比如`:echo [0, [1, 2]][1]`输出`[1, 2]`，`:echo [0, [1, 2]][-2]`输出倒数第二个元素`0`，同样支持 python 风格的切片操作

列表可以相加`:echo ['a', 'b'] + ['c']`返回`['a', 'b', 'c']`

**列表函数**

```vim
add(list, item)
len(list)
get(list, index, default)   获取给定index的item，index超过range返回default
Index(list, item)   返回-1表示不存在该item
join(list[, joiner])
reverse(list)
sort(list)

function! Sorted(l)
    let new_list = deepcopy(a:l)
    call sort(new_list)
    return new_list
endfunction
```

**字典**

取值：

```vim
:echo {'a': 1, 100: 'foo',}['a']
:echo {'a': 1, 100: 'foo',}[100]
:echo {'a': 1, 100: 'foo',}.a
:echo {'a': 1, 100: 'foo',}.100
```

删除：

删除有两种方式，一种是`:let a = remove(dict, key)`，另一种是`unlet dict.key`，区别在于第一种方式有返回值（即被删除 key 的 value），第二种没有，如果删除一个不存在的 key 会报错。

```vim
:let foo = {'a': 1, 'b':2}
:let test = remove(foo, 'a')
:unlet foo.b
:echo foo
:echo test
```

赋值：

```vim
:let foo.x = 11
```

字典方法：

```vim
get(dict, key, default)
has_key(dict, key)  返回0/1
items(dict) 返回list
```

`:echo items({'a': 100, 'b': 200})`返回`[['a', 100], ['b', 200]]`

## 变量

变量赋值

```vim
:let foo = "bar"
:echo foo
```

使用`let`关键字，等号左右可以有空格也可以没有，直接使用变量名。

使用变量 scope 指定为当前 buff 的 local 变量

```vim
:let b:foo = "bar"
:echo b:foo
```

`g:`代表 global

选项变量

```vim
:set textwidth=80
:echo &textwidth
```

使用`set`关键字，当变量为设置选项时，等号左右不能有空格，而且 echo 时必须加`&`

也可以使用`let`赋值，但是前面要加`&`： `:let &textwidth = 100`

local 选项变量

在同一窗口 split 多个窗口，在不同的窗口设置自己的 local 变量：
`:let &l:number = 1`

注册变量:`:let @h = 'hi axiaoxin'`，
现在按`"hp`就会把 h 中保存的值粘贴到当前位置

选中文字按 y 复制后，执行`:echo @"`便可以打印出按 y 复制的值

`:echo @/`可以查看查找过的文字，如果没有查找过则返回 noh，noh 就是在你查找高亮后取消高亮的命令`:noh`

## if 语句

1 代表 True，0 代表 False，非空字符串如果不是数字开头被强制转换为 0（False），数字字符串或数字开头的字符串为 True

```vim
:if 1
:    echom "ONE"    "display
:endif

:if 10 > 1
:   echom "foo"     "display
:elseif 100 == 10
:   echom "bar"
:else
:   echom "ooxx"
:endif

:if "9024"
:    echom "WHAT?!" "display
:endif

:if "something"
:    echom "INDEED" "no display
:endif
```

当比较字符串的时候不要使用`==`来判断相等，因为有`:set ignorecase`和`:set noignorecase`来修改是否区分大小写，如果`:if "foo" == "FOO"`前被添加了`:set ignorecase`，该判断依旧为 True。

正确的方式应该总是使用`==?`和`==#`来判断，无论是否使用 set ignorecase or noignorecase，`?`总是不区分大小写，而`#`总是区分大小写，`<` `>`同样适用。

```vim
:if "foo" ==# "FOO"
:   echom "no, it couldn't be"
:elseif "foo" ==# "foo"
:   echom "this must be the one"
:endif
```

## 布尔设置开关

所有布尔设置都是通过`:set <name>`来开启，通过`:set no<name>`来关闭

也可以用`toggle`来设置自动开或关，`:set <name>!`

检查某个布尔设置的开关情况可以使用 `:set <name>?`，比如查看是否显示行号用`:set number?`，如果是显示行号则显示`number`，不显示行号显示为`nonumber`

特别笔记下`:set shiftround`命令，其作用是在你按`>`或`<`或者是在插入模式下按`C-T`或`C-D`时的缩进取整对齐，
比如我设置了`:set shiftwidth=4`，然后有如下缩进不规则的文本：

```text
  asdasd
   asd
```

先`:set noshiftround`，然后光标移动到第一行按`>>`或`<<`，会将该行移动固定的 4 个宽度，变成没有对齐的形式，如下：

```text
    asdasd
   asd
```

```text
asdasd
   asd
```

如果是`:set shiftround`，同样移到第一行按`>>`或`<<`，则往后移动到于其他行对齐的形式，如下

```text
    asdasd
    asd
```

```text
asdasd
    asd
```

除了`<<`可以将当前行左移外，也可以使用`==`，=只能移到与上一行对齐，`gg=G`可以全文左对齐。visual 模式按=直接将选中部分对齐上一行。

## 赋值设置

修改行号宽度：`:set numberwidth=4`会将行号宽度变为 4（行号显示的情况下，没有行号无效）

可以一次设置多个选项：`:set number numberwidth=10`

## 循环语句

```vim
:let c = 0

:for i in [1, 2, 3, 4]
:  let c += i
:endfor

:echom c

:let c = 1
:let total = 0

:while c <= 4
:  let total += c
:  let c += 1
:endwhile

:echom total
```

## 按键映射

normal 模式：`map` `nmap`

visual 模式：`vmap`

insert 模式：`imap`

各个模式的按键映射只能在该模式下才有效，比如`:vmap \ U`只有在 visual 模式下按`\`将选中的字符执行`U`将其变成大写。
而其他模式下该按键无效，如果在其他模式配置了其他映射，则执行不同的效果。

普通键映射：`:map <KEY> <CMD>`

特殊映射：`:map <keyname> <CMD>`, 比如`:map <space> viw "按空格选中单词`

删除映射：`:unmap <KEY>` `:nunmap <KEY>`

在配置 insert 模式的映射时需要注意添加`<esc>`按键，
比如在 insert 模式下我想`ctrl+d`删除当前行，如果你配置为`:imap <c-d> dd`则会无效，
其效果是在光标处插入 dd 字符，而不是执行 normal 下的 dd 命令，
有效的配置命令为`:imap <c-d> <esc>dd`

`<esc>`是告诉 vim 按下 ESC 键退出 insert 模式，
这样删除当前行后就不再是 insert 模式了，所以正确的命令应该是`:imap <c-d> <esc>ddi`,
加上 i 表示在 normal 下按 i，再次进入 insert 模式。

**递归**

默认的映射是递归映射，即键 a 被映射成了 b，c 又被映射成了 a，那么 c 就被映射成了 b。每一个`*map`映射都有他的非递归映射`*noremap`

比如想修改 dd 命令，默认 dd 会删除当前行，后面的行会移上来，现在我们需要 dd 删除这个行后，这个行保持为空行，后面的行位置不动，你也许会配置映射为`:nmap dd O<esc>jddk`，
但是这样就进入一个死循环卡死你 vim，你本想的是先在当前行的上一行新开一行，然后移动光标到下一行即你想删除的行然后删除掉再将光标移回新开的行，但是这里的 dd 和按键 dd 导致了递归，当执行完 j 时，遇到 dd，又回到了 O，变成了无休止的新开上一行。

所以正确的配置应该使用`:noremap dd O<esc>jddk`

**leader 键**

leader 键是个前缀，比如`nnoremap -d dd`表示按`-d`映射 dd，-就可以看作是一个前缀，如果很多地方用到可以将他指定为 mapleader 的值，然后他会替换<leader>为该值

```vim
:let mapleader = '-'
:nnoremap <leader>d dd
```

默认的 mapleader 是`\\`，即`\`，第一个为转义符

这些映射都可以写在`.vimrc`中，Vim 可以通过`:sp $MYVIMRC`打开该文件，`:source $MYVIMRC`使其生效。

**缩写替换**

`abbrev`是一个类似`imap`的命令。比如`:iabbrev @@ axiaoxin.com`，这样当你在输入`@@`后按空格会将其替换为后面的网址。

## autocmd

```text
Vim在匹配到B模式的文件执行A事件时运行C命令
:autocmd BufNewFile *.txt :write
                ^     ^     ^
                |     |     |
                |     |     |
                |     |     需要运行的命令       C
                |     过滤事件的pattern          B
                watch的事件(可以是多个，用,分割) A
```

该命令是当格式为`txt`的文件在新建时就先自动写入一次，即保存空白新文件。

删除所有 autocmd：`:autocmd!`

**Reading**

```text
|BufNewFile|		starting to edit a file that doesn't exist
|BufReadPre|		starting to edit a new buffer, before reading the file
|BufRead|		    starting to edit a new buffer, after reading the file
|BufReadPost|		starting to edit a new buffer, after reading the file
|BufReadCmd|		before starting to edit a new buffer |Cmd-event|

|FileReadPre|		before reading a file with a ":read" command
|FileReadPost|		after reading a file with a ":read" command
|FileReadCmd|		before reading a file with a ":read" command |Cmd-event|

|FilterReadPre|		before reading a file from a filter command
|FilterReadPost|	after reading a file from a filter command

|StdinReadPre|		before reading from stdin into the buffer
|StdinReadPost|		After reading from the stdin into the buffer
```

**Writing**

```text
|BufWrite|		    starting to write the whole buffer to a file
|BufWritePre|		starting to write the whole buffer to a file
|BufWritePost|		after writing the whole buffer to a file
|BufWriteCmd|		before writing the whole buffer to a file |Cmd-event|

|FileWritePre|		starting to write part of a buffer to a file
|FileWritePost|		after writing part of a buffer to a file
|FileWriteCmd|		before writing part of a buffer to a file |Cmd-event|

|FileAppendPre|		starting to append to a file
|FileAppendPost|	after appending to a file
|FileAppendCmd|		before appending to a file |Cmd-event|

|FilterWritePre|	starting to write a file for a filter command or diff
|FilterWritePost|	after writing a file for a filter command or diff
```

**Buffers**

```text
|BufAdd|		    just after adding a buffer to the buffer list
|BufCreate|		    just after adding a buffer to the buffer list
|BufDelete|		    before deleting a buffer from the buffer list
|BufWipeout|		before completely deleting a buffer

|BufFilePre|		before changing the name of the current buffer
|BufFilePost|		after changing the name of the current buffer

|BufEnter|		    after entering a buffer
|BufLeave|		    before leaving to another buffer
|BufWinEnter|		after a buffer is displayed in a window
|BufWinLeave|		before a buffer is removed from a window

|BufUnload|		    before unloading a buffer
|BufHidden|		    just after a buffer has become hidden
|BufNew|		    just after creating a new buffer

|SwapExists|		detected an existing swap file
```

**Options**

```text
|FileType|		    when the 'filetype' option has been set
|Syntax|		    when the 'syntax' option has been set
|EncodingChanged|	after the 'encoding' option has been changed
|TermChanged|		after the value of 'term' has changed
```

**Startup and exit**

```text
|VimEnter|		    after doing all the startup stuff
|GUIEnter|		    after starting the GUI successfully
|GUIFailed|		    after starting the GUI failed
|TermResponse|		after the terminal response to |t_RV| is received

|QuitPre|		    when using `:quit`, before deciding whether to quit
|VimLeavePre|		before exiting Vim, before writing the viminfo file
|VimLeave|		    before exiting Vim, after writing the viminfo file
```

**Various**

```text
|FileChangedShell|	    Vim notices that a file changed since editing started
|FileChangedShellPost|	After handling a file changed since editing started
|FileChangedRO|		    before making the first change to a read-only file

|ShellCmdPost|		    after executing a shell command
|ShellFilterPost|	    after filtering with a shell command

|FuncUndefined|		    a user function is used but it isn't defined
|SpellFileMissing|	    a spell file is used but it can't be found
|SourcePre|		        before sourcing a Vim script
|SourceCmd|		        before sourcing a Vim script |Cmd-event|

|VimResized|		    after the Vim window size changed
|FocusGained|		    Vim got input focus
|FocusLost|		        Vim lost input focus
|CursorHold|		    the user doesn't press a key for a while
|CursorHoldI|		    the user doesn't press a key for a while in Insert mode
|CursorMoved|		    the cursor was moved in Normal mode
|CursorMovedI|		    the cursor was moved in Insert mode

|WinEnter|		        after entering another window
|WinLeave|		        before leaving a window
|TabEnter|		        after entering another tab page
|TabLeave|		        before leaving a tab page
|CmdwinEnter|		    after entering the command-line window
|CmdwinLeave|		    before leaving the command-line window

|InsertEnter|		    starting Insert mode
|InsertChange|		    when typing <Insert> while in Insert or Replace mode
|InsertLeave|		    when leaving Insert mode
|InsertCharPre|		    when a character was typed in Insert mode, before inserting it

|ColorScheme|		    after loading a color scheme

|RemoteReply|		    a reply from a server Vim was received

|QuickFixCmdPre|	    before a quickfix command is run
|QuickFixCmdPost|	    after a quickfix command is run

|SessionLoadPost|	    after loading a session file

|MenuPopup|		        just before showing the popup menu
|CompleteDone|		    after Insert mode completion is done

|User|			        to be used in combination with ":doautocmd"
```

## 函数

函数名必须使用大写字母开头，默认返回值为 0。

无返回值函数：

```vim
:function Meow()
:  echom "Meow!"
:endfunction
```

有返回值函数：

```vim
:function GetMeow()
:  return "Meow String!"
:endfunction
```

使用`:call <FUNC_NAME>()`进行调用。直接`:echo <FUNC_NAME>()`，如果没有返回值的函数会在正确执行后打印默认返回值 0，有返回值则打印返回值。

**函数参数**

函数内使用参数必须使用`a:`前缀

```vim
:function DisplayName(name)
:  echom "Hello!  My name is:"
:  echom a:name
:endfunction

:call DisplayName("axiaoxin")
```

不定参数

```vim
:function Varg(...)
:  echom a:0
:  echom a:1
:  echo a:000
:endfunction

:call Varg("a", "b")

:function Varg2(foo, ...)
:  echom a:foo
:  echom a:0
:  echom a:1
:  echo a:000
:endfunction

:call Varg2("a", "b", "c")
```

`a:0`表示参数个数

`a:1`表示传入的第一个参数

`a:000`表示所有传入参数的列表，`echom`不能打印列表，只能使用`echo`

参数不能在函数内进行赋值操作，即在函数内执行`let a:foo = "Nope"`是会报错的

函数可以赋值给变量

```vim
function! Append(l, val)
    let new_list = deepcopy(a:l)
    call add(new_list, a:val)
    return new_list
endfunction

function! Pop(l, i)
    let new_list = deepcopy(a:l)
    call remove(new_list, a:i)
    return new_list
endfunction

:let Myfunc = function("Append")
:echo Myfunc([1, 2], 3)
:let funcs = [function("Append"), function("Pop")]
:echo funcs[1](['a', 'b', 'c'], 1)
```

## execute 命令

`execute`用来执行字符串形式的 Vimscript 命令，
比如`:execute "echom 'Hello, world!'"`

在 execute 中执行 normal 命令时`:execute "normal! gg/print\<cr>"`在 normal 后加`!`表示 nore 非递归，特殊按键用`\`转义。

被查找的字符中可能包含有正则的特殊字符，需要进行转义：`:execute "normal! gg/for .\\+ in .\\+:\<cr>"` 或者`:execute 'normal! gg/for .\+ in .\+:\<cr>'`

## filter & map

filter:

```vim
function! Filtered(fn, l)
    let new_list = deepcopy(a:l)
    call filter(new_list, string(a:fn) . '(v:val)')
    return new_list
endfunction
:let mylist = [[1, 2], [], ['foo'], []]
:echo Filtered(function('len'), mylist)

"Vim displays [[1, 2], ['foo']].

function! Removed(fn, l)
    let new_list = deepcopy(a:l)
    call filter(new_list, '!' . string(a:fn) . '(v:val)')
    return new_list
endfunction
:let mylist = [[1, 2], [], ['foo'], []]
:echo Removed(function('len'), mylist)

"Vim displays [[], []]
```

map：

```vim
function! Mapped(fn, l)
    let new_list = deepcopy(a:l)
    call map(new_list, string(a:fn) . '(v:val)')
    return new_list
endfunction
:let mylist = [[1, 2], [3, 4]]
:echo Mapped(function("Reversed"), mylist)

"Vim displays [[2, 1], [4, 3]]
```

## 路径

显示当前文件的相对路径 `:echom expand('%')`

显示当前文件的绝对路径 `:echom expand('%:p')`

显示给定文件在当前目录的绝对路径（不论文件是否存在）`:echom fnamemodify('foo.txt', ':p')`

列出给定目录文件：`:echo globpath('.', '*')`， 使用`**`可以递归列出所有子文件下的文件
