

## 简介
VPS 中的 openssl 版本可能老旧，可能存在已知的高危漏洞，这可能导致服务器经常给你发警告邮件。
所以编译安装较新版本的 openssl 是运维的一项基本技能。这篇文章介绍在编译安装中需要注意的一些细节。


####  1. 编译安装



#### 步骤一：官网下载后的源码压缩包进行解压缩

```
cd /usr/local/src
tar xf openssl-1.1.1i.tar.gz
cd openssl-1.1.1i
```

在编译安装之前，确定所有的编译参数

先查看旧版本的编译参数

```
openssl version -f
```

输出结果：

```
CC=clang CFLAGS="-fPIC -arch arm64 -O3 -Wall -DL_ENDIAN -DOPENSSL_PIC -D_REENTRANT -DNDEBUG"
```

我们在编译时就可以带上

```
./config darwin64-arm64-cc --prefix=/usr/local \
no-zlib no-ssl3 no-ssl3-method \
CC=clang CFLAGS="-fPIC -arch arm64 -O3 -Wall -DL_ENDIAN -DOPENSSL_PIC -D_REENTRANT -DNDEBUG" 
```

执行 `./Configure help` 或 `./config help` 来查看 Usage 帮助信息。

Usage: Configure [no-<feature> ...] [enable-<feature> ...] [-Dxxx] [-lxxx] [-Lxxx] [-fxxx] [-Kxxx] [no-hw-xxx|no-hw] [[no-]threads] [[no-]thread-pool] [[no-]default-thread-pool] [[no-]shared] [[no-]zlib|zlib-dynamic] [no-asm] [no-egd] [sctp] [386] [--prefix=DIR] [--openssldir=OPENSSLDIR]
 [--with-xxx[=vvv]] [--config=FILE] os/compiler[:flags]



命令之后，会在当前目录生成下面三个文件：

```
Makefile
Makefile.in
configdata.pm
```

所有的编译参数都在 configdata.pm 里面。我们现在只需要执行 `make` 命令即可进行编译。


#### 步骤二：执行编译

```
./config --prefix=/usr/local/openssl --openssldir=
./config -t
make && make install
```

perl ./Configure --prefix=/usr/local --openssldir=/usr/local/openssl darwin64-x86_64-cc no-ssl3 no-ssl3-method no-zlib  enable-ec_nistp_64_gcc_128
./Configure darwin64-arm64-cc -shared -mmacosx-version-min=10.10
configure Cygwin-x86_64 --prefix="C:/OpenSSL/x64/" no-shared -static enable-scrypt

需要注意我们编译的是静态库还是动态库，默认是动态库，则添加选项 -shared；如果是静态库，则添加选项 `no-shared` 和 `-static`。


```
./config darwin64-arm64-cc --prefix=/Users/wforthewin/Downloads/tmp/openssl \
no-zlib no-ssl3 no-ssl3-method enable-scrypt \
CC=clang CFLAGS="-fPIC -arch arm64 -O3 -Wall -DL_ENDIAN -DOPENSSL_PIC -D_REENTRANT -DNDEBUG" 
```


Build for ARM by running the configure script, followed by make and make install. This will build for arm64, and place the results in /tmp/openssl-arm

```
./Configure darwin64-arm64-cc --prefix="/tmp/openssl-arm" no-asm
make
make install
```


编译完成后，会在当前目录多出如下文件：

```
OpenSSLConfig.cmake
OpenSSLConfigVersion.cmake
builddata.pm
installdata.pm

libcrypto.3.dylib
libcrypto.a
libcrypto.dylib
libcrypto.pc

libssl.3.dylib
libssl.a
libssl.dylib
libssl.pc

openssl.pc
```

可以看到，默认情况下，动态库和静态库都生成了。


The no-asm option tells the build system to avoid using assembly language routines, instead falling back to C routines. This prevents errors during build on ARM. The same option is not necessary for the x86 build. I’m sure there’s a way to fix building the ASM routines for arm64 (they build fine for iOS), but it’s beyond me right now.



lipo /tmp/openssl-arm/lib/libssl.a /tmp/openssl-x86/lib/libssl.a -create -output libopenssl/lib/libssl.a
lipo /tmp/openssl-arm/lib/libcrypto.a /tmp/openssl-x86/lib/libcrypto.a -create -output libopenssl/lib/libcrypto.a


#### 查看编译选项

```
openssl version -f
```

输出结果：

```
compiler: clang -fPIC -arch arm64 -O3 -Wall -DL_ENDIAN -DOPENSSL_PIC -D_REENTRANT -DOPENSSL_BUILDING_OPENSSL -DNDEBUG
```




查看符号表

Linux
```
objdump -f libssl.so -x | grep -i SRTP
objdump -f libcrypto.a 2>/dev/null | grep -i scrypt
```



MacOS

```
nm -gU /opt/homebrew/Cellar/openssl@3/3.3.2/lib/libssl.3.dylib | grep -i SRTP
```


#### 步骤三：检查

```
openssl version -a
```

```
openssl list -cipher-algorithms
```


至此，安装已经完成了。


但是，我们还需要做以下几件事来确保新装的 openssl 能够适配系统并正常工作。

#### 一：查询旧版并卸载之

```
rpm -qa | grep openssl        # 查询所有
rpm -e openssl --nodeps       # -e 卸载  --nodeps 忽略依赖
```

#### 二：链接新版

```
ln -s /usr/local/openssl/bin/openssl /usr/bin/openssl
```

#### 三：更新库函数
系统中有许多软件可能依赖 openssl 的库文件，所以我们需要更新使用新的库文件

```
echo "/usr/local/openssl/lib" >> /etc/ld.so.conf
ldconfig -v
```

#### 四：检查

```
openssl -a
```

#### 五：其他注意事项

- 系统旧版 openssl 软件包可以卸载，openssl 其他附属软件包不建议卸载

- 系统默认安装的 openssl-libs 不能卸载，否则系统可能会崩溃


全文完！




如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad 。





参考：

https://github.com/jo-tools/macos-openssl-byo
https://blog.andrewmadsen.com/2020/06/22/building-openssl-for.html
