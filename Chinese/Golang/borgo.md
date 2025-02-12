## Borgo：像写 typescript 一样来写 Go，爽飞了

最近，我发现了一个有趣的新编程语言——Borgo。如果你是一个对编程语言感兴趣的开发者，或者在使用 Go 开发应用时曾想过“如果 Go 能多点现代语言特性会怎样”，那这篇文章一定值得你读下去！
Borgo 的定位非常清晰：它是一种以简洁和高效为核心的语言，最终编译成 Go 代码。这意味着什么？你不仅能获得 Go 语言的性能优势，还能享受到现代编程语言带来的灵活性和便捷性。今天我们来聊聊 Borgo 的设计亮点、语法，以及它能解决哪些实际问题。

Borgo 的特别之处

1. 性能与效率
Borgo 的核心设计是将代码编译成 Go，这直接继承了 Go 在高并发、高性能领域的能力，比如用来构建 Web 服务、后台任务处理等。
2. 类型安全
Borgo 支持静态类型检查，这意味着在你代码运行前，编译器就能帮你发现类型相关的问题，减少那些“生产环境突然炸了”的风险。
3. 语法简洁
上手快！语法像极了现代主流语言（比如 TypeScript、Rust），不用花费太多精力去适应新规则。
4. 支持泛型
是不是总觉得 Go 的泛型用起来有点拗口？Borgo 提供了更自然的方式编写泛型代码，不仅能提升代码复用率，还能让逻辑变得更清晰。

语法体验：像写 TypeScript 一样轻松
来看看 Borgo 的代码长什么样，顺便感受一下它的优雅和实用性。
变量声明
csharp 代码解读复制代码let x = 10  // 类型自动推断为 int

枚举类型
Borgo 的枚举还支持泛型，这让状态管理变得非常方便：

```
enum NetworkState<T> {
    Loading,
    Failed(int),
    Success(T),
}
```
结构体
用起来就像 TypeScript 的对象类型：
```
struct Response {
    title: string,
    duration: int,
}
```

函数
用 fn 定义函数，逻辑清晰。
fn main() {
    let res = Response { title: "Hello world", duration: 42 }
    fmt.Println(res.title)
}

模式匹配
这是我最喜欢的功能之一！相比 Go 的 switch，Borgo 的模式匹配更灵活。

```
let msg = match state {
    NetworkState.Loading => "still loading",
    NetworkState.Failed(code) => fmt.Sprintf("Error code: %d", code),
    NetworkState.Success(res) => res.title,
}
```

动手试试：如何运行 Borgo 项目？
如果你想自己跑一个 Borgo 项目，可以按照以下步骤来：
1. 安装依赖
首先确保你的系统已经安装了 Go。
2. 克隆仓库
从 Borgo 官方 GitHub[1] 上克隆项目：
bash 代码解读复制代码git clone https://github.com/borgo-lang/borgo.git

3. 构建项目
进入项目目录后运行：
go 代码解读复制代码go build

4. 运行示例
用以下命令跑一个简单的 Borgo 程序：
bash 代码解读复制代码./borgo run examples/main.borgo


一个完整示例：小而美的代码片段
以下是一个完整的 Borgo 程序示例，展示了变量、结构体、枚举、模式匹配等特性是如何组合在一起的：
```
use fmt
    
enum NetworkState<T> {
    Loading,
    Failed(int),
    Success(T),
}

struct Response {
    title: string,
    duration: int,
}

fn main() {
    let res = Response { title: "Hello Borgo", duration: 42 }
    let state = NetworkState.Success(res)

    let msg = match state {
        NetworkState.Loading => "Loading...",
        NetworkState.Failed(code) => fmt.Sprintf("Failed with code %d", code),
        NetworkState.Success(res) => fmt.Sprintf("Success: %s", res.title),
    }

    fmt.Println(msg)
}
```

运行后，你会看到输出：
Success: Hello Borgo


Borgo 的适用场景
Borgo 的设计使它适合多种开发场景：



场景优势Web 开发高并发性能，代码简洁，适合构建 RESTful API数据处理类型安全的同时性能不打折，处理大数据也很流畅系统工具开发靠近底层的特性，让它开发 CLI 工具得心应手
比如，想象你要写一个爬虫应用，Borgo 的类型安全和高性能编译结果（Go）可以确保任务顺利完成，同时避免一堆潜在的运行时问题。

和其他语言比一比：为什么选 Borgo？


引用链接
[1] Borgo 官方 GitHub: borgo-lang.github.io/


本文链接：
https://juejin.cn/post/7442713189698273306


