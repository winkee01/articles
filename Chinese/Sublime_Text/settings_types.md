Title：理解 SublimeText 中不同的 settings 类型


SublimeText 的配置文件有三个类型：


- Default Settings (Sublime Text > Preferences > Settings)
- User Settings
- Project Settings


![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/st_settings.png)


最后一个 Project Settings，我们在点击 Save As Project 时，会有这个配置，比如：

```json
{
    "folders":
    [
        {
            "path": "."
        }
    ],
    "settings":
    {
        "SublimeLinter.linters.flake8.disable": true
    }
}
```

注意上述 Project 的配置文件中的 settings 字段

其中 `SublimeLinter.linters.flake8.disable` 这个字段就表示在 SbulimeLinter 这个模块中的配置，
如果我们把这个字段展开来理解的话就是：

SublimeLinter.sublime-settings

```json
{
    "linters": {
        "flake8": {
            "disable": true,
        }
    }
}
```

xxx.sublime-project

```json
{
    "folders":
    [
        {
            "path": "neovim-config/nvim"
        }
    ],
    "settings":
    {
        "LSP":
        {
            "LSP-lua":
            {
                "settings":
                {
                    "Lua.diagnostics.globals":
                    [
                        "vim"
                    ]
                }
            }
        }
    }
}
```

参考来源：

http://www.sublimelinter.com/en/stable/settings.html#project
http://www.sublimelinter.com/en/stable/settings.html





