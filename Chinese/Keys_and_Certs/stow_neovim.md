#### 背景介绍

一个全量的 Neovim 配置通常比较复杂，比如网上流行的 **LazyVim**, **AstroVim** 或 **LunarVim**等，它们都是比较 Full-fledged 的配置项目，有时候我们想把它们都试用一下，看看哪个更顺眼或顺手再决定自己喜欢哪个。

但我们都知道，想要应用一个第三方的配置工程，操作起来还是比较麻烦的。

首先我们得备份原来的根配置目录 `~/.config/nvim`，然后还要插件目录 `~/.local/share/nvim/`；然后下载第三方项目，移动目录为 `~/,config/nvim`。如果我们还想试试别的配置工程，我们又得重复这个步骤，还是相当繁琐的。

那么，有没有一种快速简便还丝毫不影响本地环境的方法呢？

答案是：有的！

我花了大量时间构思，经过不断试验和验证，终于证明了可行性。

今天，我决定把这个方法分享出来，为更多的 Neovim 新人造福。

##### 思路

首先，我的核心思想就是为每个项目都配置一个单独的环境（单独的目录），这样它们互不影响。然后，我使用软链接和别名的方式让 Neovim 启动不同的环境，独立访问各自目录中的配置项目。

软链接由 stow 工具完成任务，而启动不同的环境则用 Shell 中的别名（alias）完成。

关于别名，这是我们简化任务的一大利器。我这里要实现的就是，输入 `nvz` 可以启用 **LazyVim** 的配置，而输入 `nva` 则可以启用 **AstroVim** 的配置，输入 `nvl` 又可以启用 **LunaVim** 的配置，想用哪个配置，直接换个别名启用 Neovim，是不是超级方便？

我们把上述功能都在一个 Shell 脚本中实现（比如脚本命名为 `stow.fish`），最后，再实现一些管理别名的命令，达到类似如下的一些功能，岂不美哉？


![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/stow_neovim.png)



#### 原理解析

接下来，我先仔细解析一下原理，**没耐心的可以直接滑到最后看代码**。


想要 Neovim 加载不同的配置项目，我们首先就要理解它的 RuntimePath (rtp) 是怎么工作的。
rtp 的值由 `stdpath(‘config’)` 确定，而它默认的值来自于 `$XDG_CONFIG_HOME` 环境变量，而 `XDG_CONFIG_HOME` 的值默认是 `~/.config/nvim`，也就是为什么 Neovim 默认的初始配置目录是它的原因。

除了 `stdpah(‘config’)`，Neovim 中还关联一些其他的目录，比如 Data 目录为 `stdpah('data')`，Cache 目录为 `stdpah(‘cache’)` 等等。这些属性的值都跟 `XDG_` 打头的环境变量关联。

如下：

- **\$XDG\_DATA\_HOME** 默认值为 `$HOME/.local/share`.

- **\$XDG\_CONFIG\_HOME** 默认值为 `$HOME/.config` 

- **\$XDG\_DATA\_DIRS** 定义了目录的搜索优先级，用冒号 `:` 分开。那么默认值为：`/usr/local/share/:/usr/share/`。

想在 Neovim 中查看 `stdpath(‘config’)` 的值，Ex 模式下输入：

```
:echo stdpath(‘config’)
```


根据构想，我们想使用第三方的配置文件的路径 `~/downloads/nvim-starter` 作为 `stdpath(‘config’)`，怎么办？


有**三种方式**：

##### 一、直接把 `~/downloads/nvim-starter` 目录改成 `~/.config/nvim`

```
move ~/.config/nvim-starter/nvim ~/.config/nvim
```

##### 二、把 `~/.config/nvim` 设置为软链，目标是 `~/.config/nvim-starter/nvim`

```
ln -sf ~/.config/nvim-starter/nvim ~/.config/nvim
```

或者，如果 `nvim-starter/nvim` 是在远程主机上的，先把改远程目录映射到本地（比如：`/var/sftp/nvim-starter/nvim`)，

然后再用同样的方式进行软链。

##### 三、修改 XDG\_CONFIG\_HOME 的值

在系统环境变量中把 `XDG_CONFIG_HOME` 的值直接修改为 `~/.config/nvim-starter/nvim`，让 nvim 使用 `~/.config/nvim-starter/nvim` 作为默认配置目录，而不是 `~/.config/nvim`。（为了更加灵活，我们使用 `alias nvm=` 的方式） 。不过这种方式操作太麻烦，非常不推荐。


显然，根据我们文章开头的分析，第二种方式是最好的，因为有 stow 工具（接下来会讲怎么使用）帮助我们完成设软链的任务。其他两种方式都涉及比较麻烦的手工改来改去的操作。


现在假设我们同时下载了 **LazyVim**, **AstroVim** 或 **LunarVim**等好几个配置项目。本地还有我自己的项目名叫 `nvim-starter`。

现在，为了避免都使用默认的 `~/.config/nvim` 目录，我们使用具体的项目名作为父目录命，比如 `~/.config/nvim-starter/nvim`，来替代 `~/.config/nvim`，并且把 `stdpad(‘config’)` 修改为 `~/.config/nvim-starter/nvim` 即可。

具体方法就是设置环境变量 `XDG_CONFIG_HOME` ，我们在执行 nvim 命令之前加上 `XDG_CONFIG_HOME=~/.config/nvim-starter/nvim` 配置。

目录的效果大致如下：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/stow_multi_dirsx.png)

但是，仅这样做还不够完美，原因在于：

`stdpah('data')` 的默认值是 `~/.local/share/nvim`，所有项目都共享使用这个目录，这会导致各项目所配置的插件都保存在这个共享目录中，没有物理隔离，因此必然会导致插件之间的冲突。


**经过优化，我们可以这样做：**

为每个不同的项目都创建一个对应的实际目录，假如项目所在目录为 `~/Dev/nvim-all/nvim-starter`，那我们就创建 `~/.config/nvim-starter`（非软链），并在该目录中依次创建 nvim, cache 和 `share/nvim/site/pack/packer/{opt,start}` 等子目录。

这样，我们相当于隔离了每个不同的项目所会下载的 plugins 的所在目录。

然后，我们创建把 `~/.config/nvim-starter/nvim` 目录软链到 `~/Dev/nvim-all/nvim-starter`。注意，这里我们使用 stow 工具来完成，它的好处是可以对整个目录中所有文件创建软链，而不需要我们手工一个个执行 `ln -s`

简单来说，我们的目的就是把 `~/Dev/nvim-all/nvim-starter/` 中所有的文件都软链到 `~/.config/nvim-starter/nvim/` 中去，下面是具体的命令：

```
cd ~/Dev/nvim-all/nvim-starter
stow -t ~/.config/nvim-starter/nvim .
```

注：我们通常还会加上 `-R ( --restow)` 选项，这个选项的作用是先 unstow, 再 stow，当我们增加或删除过目录中的文件时，这个选项很有用。

最终目录中的效果如下：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/stow_multi_dirs1.png)


下面就具体来看怎么通过修改环境变量的值来改变 `stdpath` 的值。

#### 修改 stdpath(‘data’)

我们把 `~/.config/nvim` 软链到目标目录后，还是不够的，因为这仅是修改了 `stdpath(“config”)`；我们还需要设置插件的下载目录，即 `stdpath(“data”)`。它是 plugin 的 user path。这个 `stdpah(“data”)` 的值来自于 `$XDG_DATA_HOME`， 它默认是 `~/.local/share/`，具体来说，packpath 会在 `~/.local/share/nvim/site` 中搜索 plugin。如果多个不同的配置项目都是用这个目录来管理 plugins 的话，显然就会相互冲突，所以我们还需要修改成一个第三方目录，方便互不打扰。

注：`stdpath(“data”)` 默认值是来自 `$XDG_DATA_HOME`，它默认是  `$HOME/.local/share`.


#### 具体步骤：

设置 `~/.config/nvim` 软链具体链接到的目录，比如这里是 `~/downloads/nvim-starter`。

在操作之前，我们安装一下 stow 这个工具先，方便后面我们软链时用到它。

#### 安装 stow

MacOS 可以通过包管理器 homebrew 来安装：

```
brew install stow
```

Linux 各种发行版也可以通过各自的包管理工具安装，比如：

```
sudo apt install stow
yum install -y stow
```


设置 `~/.config/nvim` 软链具体链接到的目录，比如这里是 `~/downloads/nvim-starter`。

这个步骤包含了一些创建目录和子目录，设置环境变量等操作，我们编写一个脚本来完成这一系列动作：

`stow.sh`

```
#!/usr/bin/env fish
NVIM_STARTER=~/.config/nvim-starter
export NVIM_STARTER
rm -rf $NVIM_STARTER
mkdir -p $NVIM_STARTER/nvim
mkdir -p $NVIM_STARTER/share/nvim/site/pack/packer/{opt,start}
mkdir -p $NVIM_STARTER/cache
stow --restow --target=$NVIM_STARTER/nvim .
```

当然，这只是一个简化的脚本，主要是为了展示一下流程。在后面的完整源码中，我们会更加严谨并加入其他一些语句来使得整个操作更安全且方便。

**解释：**

`stow.fish` 位于 `~/downloads/nvim-starter/nvim/` 目录中

我们下载到的第三方配置放在 `~/downloads/nvim-starter`，然后，我们创建一个对应的目录 `~/.config/nvim-starter`，这个目录是一个 symlink，指向 `~/downloads/nvim-starter`，在这个 symlink 目录中，创建 3 个子目录，用来分别存放 plugin 相关的 data。

然后我们 cd 进入 `~/downloads/nvim-starter`，执行 stow 命令，创建 symlinks。

**注意：**

- stow 是 Linux/MacOS 中默认就自带的命令行工具，使用 stow 时， `$NVIM_STARTER/nvim` 必须是一个实际目录（不能是 symlink） ，`$NVIM_STARTER/nvim/` 内的所有文件是 symlinks。

注意：

- stow 是 Linux/MacOS 中默认就自带的命令行工具，使用 stow 时， `$NVIM_STARTER/nvim` 必须是一个实际目录（不能是 symlink） ，`$NVIM_STARTER/nvim/` 内的所有文件是 symlinks。

- stow.sh 这个脚本要放在项目根目录中，比如 `~/Downloads/nvim-starter/stow.sh`

另外，我们也可以不用 stow，而是直接手动创建一个 symlink `$NVIM_STARTER/nvim` 指向 `~/downloads/nvim-starter`；

**但区别在于：** 此时的 `$NVIM_STARTER/nvim` 本身必须是一个 symlink，而使用 stow 的话， `$NVIM_STARTER/nvim` 必须是一个实际目录，`$NVIM_STARTER/nvim/` 内的所有文件是 symlinks。

使用 stow 的好处是：如果后续在 `~/downloads/nvim-starter` 中添加了更多文件，我们可以再手动执行 `—-restow`，如下：

```
stow --restow --target=$NVIM_STARTER/nvim .
```

完成上述设置之后，我们还需要修改 XDG_DATA_HOME，它是所有 Neovim 插件的下载目录。这一步可别漏了。

这时候，我们就可以发挥 alias 的作用了，执行 alias nvb= 来设置一个别名，这样，我们只需执行 nvb 时就可以启动 Neovim 并加载上述环境了，不仅方便，而且完全不影响现有的环境。

```
export NVIM_STARTER=~/.config/nvim-starter
alias nvb='XDG_DATA_HOME=$NVIM_STARTER/share XDG_CACHE_HOME=$NVIM_STARTER/cache XDG_CONFIG_HOME=$NVIM_STARTER nvim'
```

启动 Neovim 之后，输入下面的指令来验证环境：

```
:echo stdpath('config')
:echo stdpath('data')
```

最后，要记得：在修改、新增或删除原项目中的文件之后，一定要执行 --restow 命令更新链接！

```
cd ~/Download/nvim-starter/
stow --restow --target=$NVIM_STARTER/nvim .
```

最后，提供一个我实现了完整功能的 Fish 源码：

```fish
#!/usr/bin/env fish

# Colors
set RED (tput setaf 1)
set YELLOW (tput setaf 3)
set GREEN (tput setaf 2)
set BLUE (tput setaf 4)
set BOLD (tput bold)
set RESET (tput sgr0)

set -g FUNC_DIR ~/.config/fish/functions


set -g CURRENT_DIR (dirname (realpath (status --current-filename)))
set -g PARENT_CURRENT_DIR (dirname $CURRENT_DIR)

set -g TARGET_PATH ~/.config/(basename $PWD)
set -g TARGET_PATH_FULL (eval echo $TARGET_PATH)
set -g TARGET_PATH_FULL_NVIM $TARGET_PATH_FULL"/nvim"


set -g TARGET_PATH_EXIST true
if not test -d $TARGET_PATH_FULL 
    set TARGET_PATH_EXIST false
end

# Define the default alias based on the current working directory
set -g DEFAULT_ALIAS_NAME "v"(string match -r '^[^-]+' (basename $PWD))
set -g DEFAULT_ALIAS_FILE $FUNC_DIR/$DEFAULT_ALIAS_NAME.fish

function add_alias
    set alias_name
    if test (count $argv) = 0
        set alias_name $DEFAULT_ALIAS_NAME
    end

    set alias_name $argv[1]
    set alias_file $FUNC_DIR/$alias_name.fish
    set alias_list_file $TARGET_PATH/alias.txt  # Path to alias.txt

    # Define the function content with environment variables
    set function_content "
function $alias_name
    XDG_DATA_HOME=$TARGET_PATH_FULL/share XDG_CACHE_HOME=$TARGET_PATH_FULL XDG_CONFIG_HOME=$TARGET_PATH_FULL nvim \$argv
end
"

    # Check if the alias file already exists
    if test -f $alias_file
        echo $RED"Alias '$alias_name' already exists!" $RESET
        
        # Ask for confirmation to override
        echo "Do you want to override the existing alias? (y/n): "
        set user_input
        read user_input
        if test $user_input != "y"
            echo "Aborting alias addition."
            return 1
        end
    end

    # Write the function content to the alias file
    echo $function_content > $alias_file

    if not test $status -eq 0
        echo "$BOLD$RED""Failed to create fish function!"
        exit 1
    end

    # Append the alias name to alias.txt (create the file if it doesn't exist)
    if not test -f $alias_list_file
        touch $alias_list_file
    end
    echo $alias_name >> $alias_list_file

    if not test $status -eq 0
        echo "$BOLD$RED""Failed to update alias list!"
        exit 1
    end

    echo "Alias '$alias_name' created in $FUNC_DIR/$alias_name.fish"
    source $target_func_dir

end

function list_alias
    set alias_name $DEFAULT_ALIAS_NAME
    set alias_file $FUNC_DIR/$alias_name.fish
    set alias_list_file $TARGET_PATH_FULL/alias.txt

    set -l alias_list
    if test -f $alias_file
        set alias_list $alias_list alias_name
    end

    if test -f $alias_list_file
        # echo "found $alias_list_file"
        for line in (cat $alias_list_file)
            set alias_list $alias_list $line
        end
    end

    if test -z "$alias_list"
        echo "No alias is found."
    else
        echo Available aliases are: $GREEN$alias_list$RESET
    end
end

function show_alias
    # if no argument is given, show default alias info (or all aliases?)
    # else show the specified alias info
    if test (count $argv) -eq 0
        if test -f $FUNC_DIR/$DEFAULT_ALIAS_NAME.fish
            echo $DEFAULT_ALIAS_NAME is set as:
            echo (cat $FUNC_DIR/$DEFAULT_ALIAS_NAME.fish)
        else
            echo "No default alias is found"
        end
    else
        set alias_name $argv[1]
        set alias_list_file $TARGET_PATH_FULL/alias.txt

        if test -f $alias_list_file
            set found_alias 0
            for line in (cat $alias_list_file)
                if test -f $FUNC_DIR/$alias_name.fish
                    cat $FUNC_DIR/$alias_name.fish
                    set found_alias (math $found_alias + 1) 
                    break
                end
            end
            if test $found_alias -eq 0
                echo $alias_name is not found
            end
        end
    end
end

function remove_alias
    set alias_list_file $TARGET_PATH_FULL/alias.txt

    if test (count $argv) -eq 0
        # No alias name provided: remove default alias files and aliases in alias_list_file

        set alias_name $DEFAULT_ALIAS_NAME
        set alias_file $FUNC_DIR/$alias_name.fish

        # Remove the default alias file
        if test -f $alias_file
            echo "Default alias $alias_file is found."
            echo "Are you sure you want to $YELLOW remove$RESET this alias? (y/n): "
            set user_input
            read user_input
            if test $user_input = "y"
                rm $alias_file
                echo "Default alias $alias_name removed successfully."
            else
                echo "Aborting removal of default alias."
            end
        else
            echo "Default alias $alias_name does not exist. Ignored."
        end

        if not test -d $TARGET_PATH_FULL
            echo "$TARGET_PATH_FULL directory does not exist, Ignored"
        end

        # 2. Remove all aliases listed in alias.txt
        if test -f $alias_list_file
            echo "$alias_list_file is found."
            echo "Are you sure you want to $YELLOW remove$RESET all aliases listed in $alias_list_file? (y/n): "
            set user_input
            read user_input
            if test $user_input = "y"
                # Loop through all entries in alias.txt and remove corresponding .fish files
                for alias_name in (cat $alias_list_file)
                    set alias_file $FUNC_DIR/$alias_name.fish

                    # Remove the corresponding alias file if it exists
                    if test -f $alias_file
                        rm $alias_file
                        echo "$alias_file has been removed."
                    else
                        echo "$alias_file does not exist. Ignored."
                    end
                end

                # 3. Finally, remove alias.txt
                rm $alias_list_file
                echo "$alias_list_file has been removed."
            else
                echo "Aborted removal of entries from $alias_list_file."
            end
        end

    else
        # Alias name provided: remove the specified alias and update alias_list_file
        set alias_name $argv[1]
        set alias_file $FUNC_DIR/$alias_name.fish

        # 1. Remove the specified alias file if it exists
        if test -f $alias_file
            echo "$alias_file is found."
            echo "Are you sure you want to $YELLOW remove$RESET this alias? (y/n): "
            set user_input
            read user_input
            if test $user_input = "y"
                rm $alias_file
                echo "$alias_file removed successfully."
            else
                echo "Aborted removal."
                return
            end

        else
            echo "Alias $alias_file not found."
        end

        # 2. Remove the alias from alias.txt
        if test -f $alias_list_file
            # Read alias.txt into an array
            set -l lines (cat $alias_list_file)

            # Create a temporary list to hold updated content
            set -l updated_lines
            set -l found_alias 0  # Flag to track if alias was found

            # Iterate through the lines and skip the one matching the alias name
            for line in $lines
                if test "$line" != "$alias_name"
                    set updated_lines $updated_lines $line
                else
                    set found_alias 1  # Mark that we found the alias
                    set found_alias (math $found_alias + 1)
                end
            end

            # Overwrite alias.txt with updated content
            printf "%s\n" $updated_lines > $alias_list_file

            # Check if alias was found and removed
            if test $found_alias -ge 1
                echo "Alias $alias_name has been removed from $alias_list_file."
            else
                echo "Alias $alias_name not found in $alias_list_file."
            end
        end
    end
end

function clean_config
    # Check if the alias file to clean exists
    if $TARGET_PATH_EXIST
        echo "Are you sure you want to $YELLOW clean$RESET $TARGET_PATH (y/n): "
        set user_input
        read user_input
        if test $user_input = "y"
            # 1. Remove default alias file
            if test -f $DEFAULT_ALIAS_FILE
                rm $DEFAULT_ALIAS_FILE
                echo "Default alias $DEFAULT_ALIAS_NAME removed successfully."
            end

            # 2. Remove alias file that are in $alias_list_file
            if test -f $TARGET_PATH_FULL/alias.txt
                for alias_name in (cat $TARGET_PATH_FULL/alias.txt)
                    set alias_file $FUNC_DIR/$alias_name.fish

                    # Check if the corresponding .fish file exists
                    if test -f $alias_file
                        rm $alias_file
                        echo "$alias_file has been removed."
                    else
                        echo "$alias_file does not exist. Ignored."
                    end
                end
            end

            # 3. Finally, remove $TARGET_PATH_FULL
            rm -rf $TARGET_PATH_FULL
            echo "$TARGET_PATH_FULL has been removed."

        else
            echo "Clean operation aborted."
        end
    else
        echo "The directory does not exist, nothing to clean!"
    end
end


function do_stow
    if $TARGET_PATH_EXIST
        # If it exists and is a directory, ask whether to proceed (remove/exit/restow)
        echo "$GREEN""$TARGET_PATH_FULL already exists, do you still want to proceed (y/n/r) ?"
        # echo "Type 'yes'/'y' to confirm, 'no'/'n' to exit, or 'r' to restow."
        read confirmation
        switch "$confirmation"
        case "yes" "y" "Y"
            # remove $TARGET_PATH_FULL
            rm -rf "$TARGET_PATH_FULL"; echo "$TARGET_PATH_FULL removed"
            # set TARGET_PATH_EXIST false
        case "no" "n" "N"
            echo "Operation cancelled."
            exit 1
        case "r" "R"
            do_restow
        case ""
            echo "No input provided. Operation cancelled."
            exit 1
        case "*"
            echo -e $BOLD$RED"Invalid input. Operation cancelled."
            exit 1
        end
    # else
    #     echo "$TARGET_PATH_FULL does not exist, created!"
    #     set TARGET_PATH_EXIST false
    end

    mkdir "$TARGET_PATH_FULL"; echo "$TARGET_PATH_FULL is created."
    mkdir -p "$TARGET_PATH_FULL/nvim"; echo "$TARGET_PATH_FULL/nvim is created."
    mkdir -p "$TARGET_PATH_FULL/share/nvim/site/pack/packer/opt"; echo "$TARGET_PATH_FULL/share/nvim/site/pack/packer/opt is created."
    mkdir -p "$TARGET_PATH_FULL/share/nvim/site/pack/packer/start"; echo "$TARGET_PATH_FULL/share/nvim/site/pack/packer/start is created."
    mkdir -p "$TARGET_PATH_FULL/cache"; echo "$TARGET_PATH_FULL/cache is created"

    stow -R -t "$TARGET_PATH_FULL/nvim" .


    # user can set a customed alias name
    echo "$GREEN""Please type an alias name you like, default ($DEFAULT_ALIAS_NAME):"
    set alias_name
    read alias_name
    if test -z $alias_name; set alias_name $DEFAULT_ALIAS_NAME; end
    # echo "alias name is set as: "$alias_name
    add_alias $alias_name
end

function do_restow
    if test -d $TARGET_PATH_FULL_NVIM
        stow -R -t $TARGET_PATH_FULL_NVIM -d $PARENT_CURRENT_DIR/nvim (basename $CURRENT_DIR)
        echo stow -R -t $TARGET_PATH_FULL_NVIM -d $PARENT_CURRENT_DIR/nvim (basename $CURRENT_DIR)
        return 0
    else
        echo -e $BOLD$RED$TARGET_PATH_FULL_NVIM is not a valid directory, failed to restow!
        return 1
    end
end

# Main script logic
set -l arg_count (count $argv)

if test $arg_count -eq 0
    do_stow

else if test $arg_count -gt 2
    # More than two arguments provided
    echo "The script accepts two arguments at most."
    exit 1

else
    # Handle regular cases with 1 or 2 arguments
    switch $argv[1]
        case "add"
            if test (count $argv) -lt 2
                add_alias
            else
                add_alias $argv[2]
            end
        case "remove"
            if test (count $argv) -lt 2
                remove_alias
            else
                remove_alias $argv[2]
            end
        case "list"
                list_alias
        case "show"
            if test (count $argv) -lt 2
                show_alias
            else
                show_alias $argv[2]
            end
        case "clean"
            clean_config
        case "stow"
            do_stow
        case "restow"
            do_restow
        case '*'
            echo "Unknown command! Usage: add <alias_name>, remove <alias_name>, or clean"
    end
end

# echo "$GREEN""Completed!"
```

#### 脚本使用方法：

*   把这个脚本放到你下载的某个第三方 Neovim 配置的**根目录**中去，比如你下载的是 kickstart.nvim 项目代码，放在 `~/downloads/kickstart.nvim`，那么你把 [stow.fish](http://stow.fish/) 脚本拷贝到这个目录中即可。
*   然后 source 这个脚本（`source stow.fish`）按照提示进行就可以。也可以 `chmod +x stow.fish` 改成可执行。
*   脚本执行完毕之后，会创建一个 alias，比如 nvk。然后我们执行 nvk，就会自动加载 `kickstart.nvim` 的配置来启动 Neovim 了。
*   如果我们需要测试其他 Neovim 配置，我们按照上述同样的操作即可。不同的配置会生成不同的 alias，完全互不影响，也不会影响 `~/.config/nvim`，因为我们根本不需要这个目录中的配置。如果想使用 `~/.config/nvim` 的配置信息，我们直接以 nvim 命令启动即可。


命令详解：

(1) 创建项目：

```
source stow.fish 
```

(2) 添加自定义别名:  

```
source stow.fish add <alias_name>
```

添加默认别名：

```
source stow.fish add
```

(3) 删除别名：

```
source stow.fish remove <alias_name>
```

删除所有别名：

```
source stow.fish remove
```

(4) 列出所有别名：

```
source stow.fish list
```

(5) 查看别名信息：

```
source stow.fish show <alias_name>
```

查看默认别名信息：

```
source stow.fish show
```

(5) 清空配置：

```
source stow.fish clean
```


注：

由于我常年习惯了 Fish shell，所以这里提供一个 Fish 写的源码。如果大家更习惯使用 bash，可以私信找我要，我也实现了一份。



