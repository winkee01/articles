# Title: pkg_config 原理和使用详解

## 简介

当一个应用程序在编译时，如果依赖了某个第三方库，那么它需要知道去哪里找到所有与这个库相关的文件（比如 `.h`, `.so` 等文件）。这个库就提供一个 `.pc` 文件，方便这个应用程序能够在编译时正确的访问到头文件和其他所有依赖的文件。
`pkg-config` 工具就是用来查看某个库的 `.pc` 信息的。


比如，我现在想编译一个 C++ 程序，这个程序依赖一个第三方库 Gtk，那么我们不用自己去找 Gtk 的头文件还有库文件在哪，而是直接执行 `pkg-config gtk+-3.0` 就可以。我们可以执行下面的命令来编译：

```
gcc myprogram.c $(pkg-config --cflags --libs gtk+-3.0) -o myprogram
```

当然，要注意，这个 `.pc` 文件要在 `pkg-config` 的搜索路径内才行。

搜索路径是保存在环境变量 `PKG_CONFIG_PATH` 中的，它的默认值通常是 `/usr/lib/pkgconfig` 或 `/usr/local/lib/pkgconfig`


## 配置文件

一个典型的 `.pc` 文件如下：

```toml
prefix=/usr/local
exec_prefix=${prefix}
includedir=${prefix}/include
libdir=${exec_prefix}/lib

Name: foo
Description: The foo library
Version: 1.0.0
Cflags: -I${includedir}/foo
Libs: -L${libdir} -lfoo
```

各个选项的解释如下：

- **Name**：库或软件包的人类可读名称。这不会影响 pkg-config 工具的使用，该工具使用 .pc 文件的名称。

- **Description**：对该软件包的简要描述。

- **URL**：一个 URL，用户可以在此获取有关该软件包的更多信息并下载该软件包。

- **Version**：一个字符串，专门定义该软件包的版本。

- **Requires**：该软件包所需的包的列表。这些包的版本可以使用比较运算符 =、<、>、<= 或 >= 来指定。

- **Requires.private**：该软件包所需的私有包的列表，但不对应用程序公开。来自依赖字段的版本特定规则也适用于此。

- **Conflicts**：一个可选字段，描述与此包冲突的包。来自依赖字段的版本特定规则也适用于此。此字段也可以包含同一包的多个实例。例如：冲突：`bar < 1.2.3, bar >= 1.3.0`。

- **Cflags**：特定于该软件包和任何不支持 pkg-config 的所需库的编译器标志。如果所需的库支持 pkg-config，则应将其添加到依赖或私有依赖中。

- **Libs**：特定于该软件包和任何不支持 pkg-config 的所需库的链接标志。Cflags 的相同规则适用于此。

- **Libs.private**：该软件包所需的私有库的链接标志，但不对应用程序公开。Cflags 的相同规则适用于此。


来看一个具体的例子，这个是 grpc 提供的 `pkgconfig/grpc.pc` 文件

```toml
prefix=/usr/local
exec_prefix=${prefix}
includedir=${prefix}/include
libdir=${exec_prefix}/lib


Name: gRPC
Description: high performance general RPC framework
Version: 4.0.0-dev
Cflags: -I${includedir}
Requires.private:  zlib libcares openssl
Libs: -L${libdir} -lgrpc
Libs.private:
```

有了该文件之后，确保这个 `.pc` 文件在 `PKG_CONFIG_PATH` 环境变量中。这样，我们就可以调用 pkg-config 工具来获取 grpc 这个库的各种信息了，比如库所在目录，头文件目录，依赖的其他库等。


我们可以用下列命令来找到 pkg-config 的默认搜索路径:

```
$ pkg-config --variable pc_path pkg-config
/usr/lib64/pkgconfig:/usr/share/pkgconfig
```

Linux 中，pkg-config 的默认搜索路径：

```
/usr/lib/pkgconfig
/usr/share/pkgconfig
```

有时有，下列路径也在 pkg-config 搜索范围:

```
/usr/local/lib/pkgconfig/
```


我们可以把 *.pc 文件拷贝到这里

```
cp /usr/local/opt/openssl@1.1/lib/pkgconfig/*.pc /usr/local/lib/pkgconfig/
$ pkg-config --libs grpc
-L/usr/local/lib -lgrpc
```

查看

```
$ pkg-config --cflags grpc
-I/usr/local/inclue


$pkg-config --print-variables grpc
exec_prefix
includedir
libdir
pcfiledir
prefix



$ pkg-config --list-all grpc
...


pkg-config --libs-only-L libssl
-L/usr/local/Cellar/openssl@1.1/1.1.1g/lib

```


#### 编译依赖的库

要编译一个使用 Protocol Buffers 的软件包，你需要向编译器和链接器传递各种标志。从 2.2.0 版本开始，Protocol Buffers 与 pkg-config 集成以管理这些标志。如果你已经安装了 pkg-config，那么你可以通过以下方式调用它以获取标志列表：

```
pkg-config --cflags protobuf         # print compiler flags
pkg-config --libs protobuf           # print linker flags
pkg-config --cflags --libs protobuf  # print both
```


例子：

```
c++ my_program.cc my_proto.pb.cc `pkg-config --cflags --libs protobuf`
```

请注意，在 Protocol Buffers 的 2.2.0 版本发布之前编写的软件包可能尚未与 pkg-config 集成以获取标志，并且可能不会传递正确的标志以正确链接 libprotobuf。如果相关软件包使用 autoconf，通常可以通过以下方式调用其配置脚本来修复此问题：

```
configure CXXFLAGS="$(pkg-config --cflags protobuf)" \
          LIBS="$(pkg-config --libs protobuf)"

```

这样可以强制使用正确的 flags。


全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 deliverit。




参考来源：
https://people.freedesktop.org/~dbn/pkg-config-guide.html
https://github.com/google/protobuf/tree/master/src 

