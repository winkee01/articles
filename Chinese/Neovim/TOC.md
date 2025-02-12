1、基础篇 Level 0
1.1 Neovim 的下载与安装
1.2 Vim 的神奇力量：基础操作
1.3 从入口文件理解 runtime path
1.4 Lua 基础语法，自定义配置不再繁琐
1.5 基础配置（options），从零起步

2、基础篇 Level 1
2.1 快捷键 keymap 加快操作
2.2 自执行命令 autocmds 小试牛刀
2.3 自定义 Command
2.4 再聊 runtime
2.5 编写简单的 helper，试水自定义配置

3、基础篇 Level 3

3.1 第一印象 colorscheme + base16 真彩色
3.2 语法高亮 syntax highlight，不再枯燥
3.3 状态栏 status line，抓住眼球
3.4 NerdFont，让图标不再缺失
3.5 更多 Key Mapping，让操作更顺手
3.7 clipboard 设置
3.6 更多模式探索

4、基础篇 Level 4
实战操作

4.1 搜索与替换
4.2 grep, rg 
4.3 Fuzzy find: fzf 与 telescope
4.3 代码折叠 fold
4.4 buffer vs args
4.5 register 和 marks
4.6 侧边栏文件浏览器 netrw
4.7 split 与 tagpage 操作
4.8 changelist 和 jumplist

Vim Cheatsheet

代码相关
（1） 代码诊断：quickfix
（2）括号匹配（auto pairs) 只能通过插件
（2） 代码补全（auto completion）自带
（3） 代码片段（code snippets）
（4） 代码跳转（ctags）
传统方式：ctags, cscope
lsp 方式：通过 vim.lsp 实现 Go to definition 等功能


5、提高篇 Level 5
5.1 Neovim 的自动加载系统探究（ftdetect, plugin, ftplugin, after/）
5.2 编写后加载程序
5.3 插件管理工具 Lazyvim
5.4 开源配置工程推荐（初始配置）
5.5 which-key 入门介绍（或者 cheatsheet）

6、提高篇 Level 6
插件篇

6.1 外观美化
Appearance 部件主要有：
- colorscheme/theme
- column (number, sign)
- statusline
- winbar
- notify
- scroll bar (效果)
- indent
- register & mark
- TODO 高亮
- sidebar （侧边栏）
- outline （侧边栏）


UI 部件（vim.ui）
主要涉及一些 floating windows, popup menu 的设置。
Neovim 提供了两个 API：vim.ui.select 和 vim.ui.input，外部插件通过重载它们来实现定制化。

插件分类之 –– 外观类插件（appearance, UI components）

(1) colorscheme

```
tokyonight
gruvbox.nvim
catppuccin/nvim
EdenEast/nightfox.nvim
navarasu/onedark.nvim
monokai-pro
```

(2) column

```
set signcolumn=yes
set numberwidth=2
```

(3) statusline

```
nvim-lualine/lualine.nvim
echasnovski/mini.statusline
famiu/feline.nvim (deprecated)
nvimdev/galaxyline.nvim (deprecated)
tjdevries/express_line.nvim (deprecated)
rebelot/heirline.nvim
```

(4) winbar（非核心）

```
SmiteshP/nvim-navic
SmiteshP/nvim-navbuddy
Bekaboo/dropbar.nvim
```

(5) notify

```
rcarriga/nvim-notify
```

(6) Scroll bar
```
lewis6991/satellite.nvim
nvim-scrollview
nvim-scrollbar
```

(7) UI

```
stevearc/dressing.nvim
folke/noice.nvim
MunifTanjim/nui.nvim (核心组件)
```


(8) Edit area

```
lukas-reineke/indent-blankline.nvim
tversteeg/registers.nvim
```

(9) sidebar （非核心）

```
sidebar-nvim/sidebar.nvim
``


(10) outline （非核心）

```
perservim/tagbar
stevearc/aerial.nvim
vista.vim (vimscript)
simrat39/symbol-outline.nvim
```

(11) register & marks

```
tversteeg/registers.nvim
```

(12) TODO

```
folke/todo-comments.nvim
```


6.2 功能增强（Edit, Navigation，语言相关)

功能增强可以包括：编辑区的显示增强，光标跳转的增强，代码块的缩进显示，代码的层次图 outline/minimap。
有一些功能是跟文件类型（语言）相关的需要通过 LSP 才能实现的增强，比如 inlay hints 等。
以及编辑功能的增强

细分为下面几个方面：
一、Display
二、Movement
三、Edit
四、Search & Replace


普通编辑
- autopair
- surround
- Esc 键替换（快速从 Insert 切换到 Normal 模式）
- symbol toggle
- repeat

多文件/跨文件编辑

- jump cursor
- tagpages 使用 tag 页打开多文件
- layout （开启多个 split 窗口时，保持各窗口比例）
- file explorer

语言相关
- treesitter（提供词法分析）基于 token 或 text object 来行动
- LSP config and server management
- LSP UI
- LSP linter
- LSP formatter
- LSP auto-complete
- LSP code snippets
- LSP Inlay Hints
- LSP Signature
- LSP code actions
- LSP codelens
- LSP diagnostics



persistent undo
cursor back to last position
modeline
clipboard
session
gui settings (guifont)
terminal
autopairs


