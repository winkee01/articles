下面是 Lua 的基础语法介绍，适合用来配置 Neovim时掌握：

---

### **1. 变量和数据类型**
Lua 是动态类型语言，可以直接声明变量，不需要指定类型。

#### 变量定义
```lua
local x = 10        -- 局部变量
y = 20              -- 全局变量（不推荐）
```

#### 常见数据类型
- **`nil`**：表示“没有值”。
- **`boolean`**：`true` 或 `false`。
- **`number`**：整数和浮点数。
- **`string`**：字符串。

```lua
local a = nil       -- 无值
local b = true      -- 布尔值
local c = 3.14      -- 数字
local d = "hello"   -- 字符串
```

---

### **2. 表（Table）**
表是 Lua 的核心数据结构，可以用作数组、字典或对象。

#### 定义表
```lua
local t = {1, 2, 3}           -- 数组形式
local t = {key1 = "value1", key2 = "value2"}  -- 字典形式
```

#### 访问表
```lua
print(t[1])       -- 数组访问，输出 1
print(t.key1)     -- 字典访问，输出 "value1"
```

#### 修改表
```lua
t[2] = 100
t.new_key = "new_value"
```

---

### **3. 条件语句**
Lua 使用 `if` 语句进行条件判断。

```lua
local x = 10
if x > 5 then
    print("x > 5")
elseif x == 5 then
    print("x == 5")
else
    print("x < 5")
end
```

---

### **4. 循环**
#### **`for` 循环**
```lua
for i = 1, 5 do
    print(i)  -- 输出 1 到 5
end
```

#### **`while` 循环**
```lua
local x = 0
while x < 5 do
    x = x + 1
    print(x)
end
```

#### **遍历表**
```lua
local t = {key1 = "value1", key2 = "value2"}
for k, v in pairs(t) do
    print(k, v) -- 遍历字典
end
```

---

### **5. 函数**
函数是 Lua 的核心，支持局部和全局定义。

#### 定义函数
```lua
local function add(a, b)
    return a + b
end
```

#### 调用函数
```lua
local result = add(10, 20)
print(result)  -- 输出 30
```

---

### **6. 冒号语法与面向对象**
冒号语法常用于表示方法调用。

#### 定义方法
```lua
local obj = {}
function obj:print_name(name)
    print("Name: " .. name)
end
```

#### 调用方法
```lua
obj:print_name("Lua")  -- 输出 "Name: Lua"
```

---

### **7. 模块**
Lua 支持将代码组织为模块，Neovim 配置中经常用到。

#### 定义模块
```lua
local M = {}

function M.say_hello()
    print("Hello, Neovim!")
end

return M
```

#### 使用模块
```lua
local my_module = require('my_module')
my_module.say_hello()
```

---

### **8. Neovim 常用 Lua 函数**
在配置 Neovim 时，你会经常用到一些特殊的函数。

#### 设置选项
```lua
vim.o.number = true          -- 全局设置
vim.bo.expandtab = true      -- 针对缓冲区设置
vim.wo.relativenumber = true -- 针对窗口设置
```

#### 创建快捷键
```lua
vim.api.nvim_set_keymap('n', '<leader>ff', ':Telescope find_files<CR>', { noremap = true, silent = true })
```

#### 执行命令
```lua
vim.cmd('colorscheme gruvbox')  -- 运行 Vim 命令
```

#### 使用插件
```lua
require('telescope').setup{}
```

---

### **9. 重要函数**
- **`print`**：输出调试信息。
- **`type`**：获取变量类型。
- **`os.getenv`**：获取环境变量。
- **`vim.fn`**：访问 Vim 的函数（如 `vim.fn.expand()`）。

---

### 学习重点
1. 理解 Lua 的表（Table），因为 Neovim 配置大量使用表。
2. 熟悉 `vim.*` API（如 `vim.o`、`vim.api`、`vim.cmd`）。
3. 学习如何创建模块和方法，组织配置代码。

---

