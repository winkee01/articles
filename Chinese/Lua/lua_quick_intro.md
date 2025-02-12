## 简介
Lua 是一个小巧的脚本语言，是巴西里约热内卢天主教大学里的一个研究小组于 1993 年开发的。

Lua 使用标准 C 语言编写并以源代码形式开放，它体积小、启动速度快，一个完整的 Lua 解释器不过 200k，在所有脚本引擎中，Lua 的速可以于说是最快的。所以 Lua 是作为嵌入式脚本的最佳选择。这也就是我们为什么要学习 Lua 这门语言。

### 1. 安装 Lua 并理解 Lua 运行环境
MacOS 系统中使用 homebrew 安装很方便：

```
brew install lua
```

默认是安装的 lua 5.4 版本。由于 5.1 版本已经被 homebrew 禁用，所以如果实在想安装，可以源码编译安装。

如果在其他 OS 中，使用相应的包管理工具安装即可，比如：

```
apt-get install lua
yum install -y lua 
```


MacOS 中 lua 安装完成之后，相关文件会默认安装在 `/opt/homebrew` 目录下，具体来说

- Lua binary: `/opt/homebrew/bin/lua`
- headers: `/usr/local/include/lua5.x`
- Libraries: `/usr/local/lib/lua/5.x`


安装完成后执行 `lua -v` 来检查版本。


注：如果是 Linux，那么目录是在 `/usr/local` 而不是 `/opt/homebrew`


### 2. 安装 luarocks

luarocks 是 lua 的包管理工具（类似于 npm 之于 Nodejs），有了它，在开发 lua 的时候会非常方便。

```
brew install luarocks
```

同样，luarocks 的相关文件也会默认安装在 `/opt/homebrew/luarocks` 目录下。

另外，libraries 在 luarocks 中也叫 modules，它们的配置信息会在 `~/.luarocks` 目录下。这个目录叫 user 目录；而 `/usr/local/` 则叫 system 目录。

直接执行 luarocks 查看所有可用命令。

常见命令：

```
luarocks install dkjson
luarocks remove dkjson
luarocks show dkjson
luarocks path --bin
```


注：执行 `luarocks install dkjson` 时，module 默认是被安装在 `/opt/homebrew/share/lua/5.x/<package_name>` 中的


### 3. LUA\_PATH and LUA\_CPATH

lua 脚本写好之后，lua 是怎么执行或加载的呢？

**LUA_PATH**: 这个环境变量定义了 Lua 会去哪里查找 `.lua` 脚本并加载
比如，我们写了一个 `hello.lua` 脚本，这个脚本使用了 `require(“lyaml")` 语句，那么 lua 就会通过 `LUA_PATH` 中所定义的路径去搜索 lyaml 这个 module。如果 lyaml 不在 `LUA_PATH` 中，require 就会提示找不到 module。
除了 `.lua` 后缀的 module，还有 `.so` 后缀的 module（C 语言编写），后者在 `LUA_CPATH` 的路径中搜索。


LUA_PATH 默认值为：

```
./?.lua;/usr/local/share/lua/5.x/?.lua;/usr/local/share/lua/5.x/?/init.lua
```

或者

```
./?.lua;
./?/init.lua
/opt/homebrew/share/lua/5.4/?.lua;
/opt/homebrew/share/lua/5.4/?/init.lua;
/opt/homebrew/lib/lua/5.4/?.lua;
/opt/homebrew/lib/lua/5.4/?/init.lua;
./?.lua: Lua looks for files in the current directory.
/usr/local/share/lua/5.x/?.lua: Lua looks for modules in /usr/local/share/lua/5.x/
```

LUA_CPATH 默认值为：

```
./?.so;/usr/local/lib/lua/5.x/?.so
```

或

```
./?.so
/opt/homebrew/lib/lua/5.4/?.so;
/opt/homebrew/lib/lua/5.4/loadall.so;
```

我们可以通过 `print(package.path)` 和 `print(package.cpath)` 来查看 `LUA_PATH` 和 `LUA_CPATH` 的当前值。


修改 `LUA_PATH` 和 `LUA_CPATH`
直接在 `~/.bashrc` 或 `~/.zshrc` 等 shell 配置文件中导出即可，如下：

```
export LUA_PATH="./?.lua;/usr/local/share/lua/5.x/?.lua"
export LUA_CPATH="./?.so;/usr/local/lib/lua/5.x/?.so"
```


#### init.lua
这是一个比较特殊的文件，因为，如果当它所在的文件夹在 `LUA_PATH` 中时，这个文件会被自动加载。
如果是在 `vim/neovim` 中，那么所有在 runtimepath 中 plugin 下的 `.vim` 或 `.lua` 文件也会被自动加载。


### 4. Lua 语法

##### （1）导入包 require

```
require("something")
require"something"
require "something"
```


上述三种用法是等效的：
Lua allows both parentheses () and quotes "" for function calls when there’s a single string or table as an argument.


##### （2）table 用法
table 是 lua 中唯一的复合结构，可以是 array 型，也可以是 dictionary 型，也可以两种类型都包含。
array 型时，index 是从 1 开始（而不是 0），统计长度时，也只统计 array 元素，不统计 dict 元素。
dict 型时，遍历元素不保证顺序。

```
local t = {} -- Empty table
local t = {1, 2, 3} -- Array-like table
local t = {name="Lua", year=1993} -- Key-value table (dictionary-like)

t[1] = "hello" -- For array-style indexing
t.name = "Lua Programming" -- For key-value pairs (acts like syntactic sugar for t["name"])
```

#### 深度理解 array 型和 dict 型的区别：
当使用 array 型时，比如 `local t = {"one", "two", "three"}`，Lua 实际上会隐式的创建一个 numeric key，其效果类似：
`t = { [1] = "one", [2] = "two", [3] = "three" }`，这样，我们就可以通过 `t[1]` 访问到 "one" 这个值。而对于 dict 型，则是没有 numeric key 的。

当我们遍历的时候，也就存在使用 `ipairs()` 和 `pairs()` 的区别了。
`ipairs()` 用来遍历 array-like table，pairs() 用来遍历 dict-like table。

例子：

```
> u = {}
> u[-1] = "y"
> u[0] = "z"
> u[1] = "a"
> u[3] = "b"
> u[2] = "c"
> u[4] = "d"
> u[6] = "e"
> u["hello"] = "world"
>
> for key, value in ipairs(u) do print(key, value) end
1       a
2       c
3       b
4       d
>
> for key, value in pairs(u) do print(key, value) end
1       a
2       c
3       b
4       d
6       e
0       z
hello   world
-1      y
> 
```

注意：
尽管 0 和 6 都是 numeric，但是 0 小于 1，而 lua 的 index 从 1 开始，所以不打印出来；6 不打出来是因为 `t[5]` 是 nil（因为这里没设置 `t[5]`），而 `ipairs()` 函数遇到 value 为 nil 时就停止。


Table constructors

从上面可以看到，我们初始化一个 dict 类型的 table 时，使用了这种语法：

```
local tb = {
  name = "john", 
  ["5"] = "tom"
}

```

注意：name = "john" 不能写成 "name" = "john"
由于使用 name = "john" 这种方式有许多限制，比如如果 key 不是一个有效的 indentifier 或者不是 string 类型的（比如 + 就不能作为 key 使用）。还有许多特殊符号(+, -, /, * 等等）都无法作为 key。为了让它们也可以作为 key，lua 就扩展了 ["xxx"] = "tom" 这种语法。

这样，任何符号（unicode 符号除外，因为 lua不支持）都可以放左边了。

```
opnames = {
    ["+"] = "add", 
    ["-"] = "sub",
    ["*"] = "mul", 
    ["/"] = "div"
}
```


table 的 overide 和 extend 经常使用，

对于 override 很好理解，直接 `t["key"]` 赋值的方式覆盖即可。


extend 的话，有如下：

```
table.insert(t, "new value") -- Insert into array-style table
t["newkey"] = "new value" -- Insert key-value pair

local array_tbl = {3, 5, 7}
table.insert(array_tbl, 9)  -- {3, 5, 7, 9}
table.remove(array_tbl, 1)  -- {5, 7, 9}
```

不过，根据 extend 级别，还分为 shallow 和 deep 两种方式，在 `vim_tbl_extend` 中有介绍，如下：


neovim 中提供的 built-in 函数 `vim.tbl_extend({behavir}, {...})` 和 `vim.tbl_deep_extend({behavir}, {...})`


这里的 behavior 是 3 种：

```
"error": raise an error
"keep": use value from the leftmost map
"force": use value from the rightmost map
```


`tbl_extend` 和 `tbl_deep_extend` 的区别，看一个例子：

```
tb_1 = {a = true, b = { c = true }}
tb_2 = {b = { c = true, d = true}

print(vim.inspect(vim.tbl_deep_extend('keep', tb_1, tb_2)))

{
  a = true,
  b = {
    c = true,
    d = true
  }
}

print(vim.inspect(vim.tbl_extend('keep', tb_1, tb_2)))

{
  a = true,
  b = {
    c = true
  }
}

meta table
local mt = {
    __index = function(t, key)
        return "default value"
    end
}
setmetatable(t, mt)
```


####（3）函数

定义函数

```
function myFunc(arg1, arg2)
    return arg1 + arg2
end
```

匿名函数

```
local add = function(a, b) return a + b end
```

####（4）字符串

```
path = "/lua"..config
pattern match
s = string.gsub("Lua is cute", "cute", "great")
print(s)         --> Lua is great
s = string.gsub("all lii", "l", "x")
print(s)         --> axx xii
s = string.gsub("Lua is great", "perl", "tcl")
print(s)         --> Lua is great
```


####（5）vim.lua
所有在 runtimepath 下的 `lua/` 目录下的 `.lua` 脚本，都可以直接通过 `require()` 函数手动加载。`init.lua` 则是自动加载。

比如，`~/.config/nvim` 中的结构如下：

```
~/.config/nvim
|-- after/
|-- ftplugin/
|-- lua/
|   |-- myluamodule.lua
|   |-- other_modules/
|       |-- anothermodule.lua
|       |-- init.lua
|-- plugin/
|-- syntax/
|-- init.vim
```

这里，`require("myluamodule")` 语句就会加载 `myluamodule.lua` 脚本内容。如果想要加载更下一层的脚本，则使用 `require("other_modules.anothermodule")` 或 `require("other_modules/anothermodule")`

当文件夹中有 `init.lua` 文件时，我们直接加载文件夹即可，`init.lua` 会自动加载。比如 `require("other_modules")` 就可以自动加载 `init.lua`。

注意⚠️：
当我们加载一个并不存在的 module 时，lua 会报错并退出执行当前的脚本，为了避免直接退出，我们可以使用 `pcall()` 来捕获异常。
假设 `module_with_error` 是一个错误的 module，我们可以这样：

```
local ok, mymod = pcall(require, 'module_with_error')
if not ok then
  print("Module had an error")
else
  mymod.function()
end
```

这里，只有 require 成功时，我们才调用 `mymod.function()`。
提示🔔：
Lua 会把成功 `require` 过的 module 进行 cache，这样，当再次 `require` 时，就不会重复执行该 module 了。
如果你确实想再次执行该 module，那么可以这样：

```
package.loaded['myluamodule'] = nil
require('myluamodule')    -- read and execute the module again from disk
```


想要执行 vimscript 中的命令时，我们可以用 `vim.cmd()` 来执行，比如：

```
vim.cmd("colorscheme habamax")
vim.fn 可以用来调用 vimscript 中的函数，比如：
print(vim.fn.printf('Hello from %s', 'Lua'))
local reversed_list = vim.fn.reverse({ 'a', 'b', 'c' })
vim.print(reversed_list) -- { "c", "b", "a" }
local function print_stdout(chan_id, data, name)
  print(data[1])
end
vim.fn.jobstart('ls', { on_stdout = print_stdout })

```



参考：
https://stackoverflow.com/questions/55108794/what-is-the-difference-between-pairs-and-ipairs-in-lua
https://neovim.io/doc/user/lua-guide.html#lua-guide



全文完！



如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad。


