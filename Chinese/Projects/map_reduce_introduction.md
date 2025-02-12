MapReduce 模型实战


### MapReduce 简介

MapReduce 是一种编程模型，用于在集群上使用并行分布式算法处理和生成大数据集。其基本思想是将输入数据拆分成更小、可管理的块，以便可以并行处理（Map 阶段），然后将结果组合以生成最终输出（Reduce 阶段）。

### MapReduce 的工作原理

#### 1. Map 阶段（Mapper）：

输入数据被划分为小块，并分配到多个“mapper ”上。
每个mapper 独立处理其数据块，并输出键值对（`<key, value>`）。

#### 2. Shuffle 和 Order：

在所有 mapper 生成键值对后，系统根据键对所有值进行分组（洗牌和排序阶段），确保相同键的所有值都发送到同一个 reducer 。

#### 3. Reduce 阶段（Reducer）：

每个reducer 处理一组具有相同键的键值对。它对每个键的值执行某种操作以聚合或总结这些值，并生成最终结果。

###### 总结分析：

其实 mapper 并没有什么特殊，它只不过担任一个 worker 的角色，接收数据（比如来自 Stdin），处理数据（既可以自己处理，也可以发给 Server 进行处理）

**唯一的特殊就是：** 有一个 Master 程序负责动态的生成多个 workers，每个 worker 都是一个独立的执行线程，去执行 mapper 所实现的功能，mapper 只需处理一小部分数据。然后把处理后的结果发送给 Master 程序进行 reduce 或 combine。当然，也可以发送给专门的 reducers 来整合，最后再发给 Master 汇总。

我绘制了一个大致的流程图，如下：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/map_reduce_intro.jpg)


接下来我们使用 MapReduce 的原理来实战一个 word count 程序

```go
package main

import (
    "fmt"
    "strings"
    "sync"
)

// Mapper function: splits the input text into words and emits <word, 1> pairs
func mapper(text string) map[string]int {
    words := strings.Fields(text)
    result := make(map[string]int)
    for _, word := range words {
        result[word]++
    }
    return result
}

// Reducer function: combines the results from multiple mappers by summing up the values for each word
func reducer(mappedData []map[string]int) map[string]int {
    finalResult := make(map[string]int)
    for _, partialResult := range mappedData {
        for word, count := range partialResult {
            finalResult[word] += count
        }
    }
    return finalResult
}

func main() {
    // Input data: a list of text blocks
    textBlocks := []string{
        "hello world",
        "world of code",
        "hello gophers",
        "hello world of gophers",
    }

    // Step 1: Map Phase
    var wg sync.WaitGroup
    var mu sync.Mutex
    mappedData := make([]map[string]int, len(textBlocks))

    for i, text := range textBlocks {
        wg.Add(1)
        go func(i int, text string) {
            defer wg.Done()
            result := mapper(text)

            mu.Lock()
            mappedData[i] = result
            mu.Unlock()
        }(i, text)
    }

    wg.Wait()

    // Step 2: Reduce Phase
    result := reducer(mappedData)

    // Output the final word count result
    for word, count := range result {
        fmt.Printf("%s: %d\n", word, count)
    }
}
```

##### 输出结果：

    hello: 3
    world: 2
    of: 2
    code: 1
    gophers: 2

##### 代码解释：

*   **Mapper**：mapper 函数接收一段文本并将其拆分为单词。对于每个单词，它发出一个键值对，其中键是单词，值是数字 1。结果是该文本块的单词计数键值对（`<word: count>`)。

*   **Reducer**：reducer 函数接收多个 mapper 的结果（映射列表）并将它们合并。它对所有 mapper 结果中的每个单词的计数进行累加，生成每个单词的最终计数。

*   **并行执行**：主函数使用 goroutines 并行处理每个文本块。sync.WaitGroup 确保程序等待所有goroutines 完成，而 sync.Mutex 确保对共享的 mappedData 切片的线程安全写入。

*   **最终输出**：最终结果是所有输入文本块中每个单词的计数。

#### 升级

这个 word count 是最典型的使用 MapReduce 思想的应用。程序非常简单，接下来我们增加一点复杂度。
下面是新的要求：

1.  程序从磁盘文件读取数据（假设文件名为 `data.txt`）。

2.  `data.txt` 包含大量用空格分隔的单词，可以根据测试数据量的需要来控制行数。

3.  实现一个生成器程序来生成 `data.txt`，该程序应从字典文件 `dict.txt` 中读取单词，以随机生成单词。

4.  主程序只需计数预定义的单词列表，忽略其他单词。假设它只关注五个单词：Happy、Sex、Food、Bank 和 Physics。

5.  忽略单词的大小写。

6.  主程序需要决定实际需要多少个 mapper ，以避免给系统带来过重的负担。换句话说，它应该限制生成的最大 goroutines 数量，可能需要实现一个 goroutine 池。

为了方便理解，我绘制了一个流程图：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/mapred_channel01.jpg)

## 实现

我们需要实现两个程序，一个是 data generator 用于生成测试数据，另一个是主程序，使用 mapreduce 思想来统计单词数量。这里省去 data generator。

###### MapReduce 主程序:

`mapred_wc1.go`

```go
    package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
    "sync"
    "time"
)

// Words of interest to count (all lowercase now)
var wordsOfInterest = map[string]bool{
    "happy":   true,
    "sex":     true,
    "food":    true,
    "bank":    true,
    "physics": true,
}

// Mapper function: counts only the words of interest in a given line (case insensitive)
func mapper(line string) map[string]int {
    result := make(map[string]int)
    words := strings.Fields(line)
    for _, word := range words {
        normalizedWord := strings.ToLower(word) // Normalize to lowercase
        if wordsOfInterest[normalizedWord] {
            result[normalizedWord]++
        }
    }
    return result
}

// Reducer function: combines the results from multiple mappers
func reducer(mappedData []map[string]int) map[string]int {
    finalResult := make(map[string]int)
    for _, partialResult := range mappedData {
        for word, count := range partialResult {
            finalResult[word] += count
        }
    }
    return finalResult
}

// Goroutine worker pool to process lines with a limited number of workers
func workerPool(jobs <-chan string, results chan<- map[string]int, wg *sync.WaitGroup) {
    defer wg.Done()
    for line := range jobs {
        results <- mapper(line)
    }
}

func main() {
    startTime := time.Now()

    // Open file for reading
    file, err := os.Open("data100k.txt")
    if err != nil {
        panic(err)
    }
    defer file.Close()

    // Channel for job distribution and result collection
    jobs := make(chan string, 100)        // Buffer size 100 for incoming lines
    results := make(chan map[string]int)  // For the mapper's output

    // Restrict goroutines to a maximum of 10 workers
    const maxWorkers = 15
    var wg sync.WaitGroup

    // Start worker pool
    for i := 0; i < maxWorkers; i++ {
        wg.Add(1)
        go workerPool(jobs, results, &wg)
    }

    // Collect results asynchronously
    var resultWg sync.WaitGroup
    resultWg.Add(1)
    go func() {
        defer resultWg.Done()
        var allResults []map[string]int
        for res := range results {
            allResults = append(allResults, res)
        }
        // Final reduction after all results are gathered
        finalResult := reducer(allResults)
        for word, count := range finalResult {
            fmt.Printf("%s: %d\n", word, count)
        }
    }()

    // Read file line by line and send to jobs channel
    scanner := bufio.NewScanner(file)
    for scanner.Scan() {
        line := scanner.Text()
        jobs <- line
    }
    close(jobs)

    // Wait for all workers to finish
    wg.Wait()
    close(results)

    // Wait for results collection and reduction to finish
    resultWg.Wait()

    fmt.Printf("Execution Time: %s\n", time.Since(startTime))
}
```

##### 点评

这个程序存在 goroutine 的一些竞争问题，效率并不算高，再加上 goroutine 的创建销毁等开销，增加 goroutine 的数量反而可能会伤害性能。

为了进一步提升性能，我再次设计了一个优化的流程，使用多个 reducer。

下面是具体的要求：

1.  **生成多个工作线程（最多10个）以从 `data.txt` 中读取单词行。**

2.  **每个工作线程查找感兴趣的 word，并将其计数发送到相应的 word channel。**

3.  **word channel 的数量与预定义的感兴趣的 word 数量相同。**

4.  **reducer 工作线程的数量与 word channel 的数量相同。**

5.  **每个 reducer 工作线程统计其 word chennel 的总数，并将结果发送到 combine channel。**

6.  **主程序从 combine channel 读取数据并打印结果。**

同样，为了方便理解，我绘制了程序的执行流程图。

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/mapred_channel02.jpg)

主程序

`mapred_wc2.go`

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
    "sync"
    "time"
)

// Words of interest
var wordsOfInterest = []string{"happy", "sex", "food", "bank", "physics"}

// Mapper function: Counts occurrences of words of interest in a line (case-insensitive)
func mapper(line string) map[string]int {
    result := make(map[string]int)
    words := strings.Fields(line)
    for _, word := range words {
        normalizedWord := strings.ToLower(word) // Normalize to lowercase
        for _, interestWord := range wordsOfInterest {
            if normalizedWord == interestWord {
                result[normalizedWord]++
            }
        }
    }
    return result
}

// Worker function to process lines and send word counts to designated channels
func worker(jobs <-chan string, wordChans map[string]chan int, wg *sync.WaitGroup) {
    defer wg.Done()
    for line := range jobs {
        counts := mapper(line) // Map the line
        for word, count := range counts {
            wordChans[word] <- count // Send counts to corresponding channels
        }
    }
}

// Reducer function to aggregate word counts from its designated channel
func reducer(word string, wordChan <-chan int, combineChan chan<- map[string]int, wg *sync.WaitGroup) {
    defer wg.Done()
    totalCount := 0
    for count := range wordChan {
        totalCount += count
    }
    // Send final count to the combine channel
    combineChan <- map[string]int{word: totalCount}
}

func main() {
    startTime := time.Now()

    // Open file for reading
    file, err := os.Open("data100m.txt")
    if err != nil {
        panic(err)
    }
    defer file.Close()

    // Channels
    jobs := make(chan string, 100)                        // Channel for distributing lines to workers
    wordChans := make(map[string]chan int)                // Channels for each word of interest
    combineChan := make(chan map[string]int, len(wordsOfInterest)) // Channel to collect final results

    // Initialize word-specific channels
    for _, word := range wordsOfInterest {
        wordChans[word] = make(chan int, 100) // Buffer size of 100 for each word channel
    }

    // Workers
    var workerWg sync.WaitGroup
    maxWorkers := 5
    for i := 0; i < maxWorkers; i++ {
        workerWg.Add(1)
        go worker(jobs, wordChans, &workerWg)
    }

    // Reducers for each word
    var reducerWg sync.WaitGroup
    for _, word := range wordsOfInterest {
        reducerWg.Add(1)
        go reducer(word, wordChans[word], combineChan, &reducerWg)
    }

    // Read lines from file and distribute them to workers
    go func() {
        scanner := bufio.NewScanner(file)
        for scanner.Scan() {
            line := scanner.Text()
            jobs <- line
        }
        close(jobs)
        workerWg.Wait() // Wait for all workers to finish
        for _, ch := range wordChans {
            close(ch) // Close all word-specific channels after workers are done
        }
    }()

    // Wait for all reducers to finish
    go func() {
        reducerWg.Wait()
        close(combineChan) // Close combine channel after all reducers are done
    }()

    // Print final results from combine channel
    finalResults := make(map[string]int)
    for result := range combineChan {
        for word, count := range result {
            finalResults[word] = count
        }
    }

    // Output the results
    fmt.Println("Final word counts:")
    for word, count := range finalResults {
        fmt.Printf("%s: %d\n", word, count)
    }

    fmt.Printf("Execution Time: %s\n", time.Since(startTime))
}
```

#### 测试效果：

(1) 100万数据：

> mapred\_wc1: 677.334292ms
>
> mapred\_wc2: 561.095166ms

(2) 1000万数据：

> mapred\_wc1: 6.7s
>
> mapred\_wc2: 3.9s

(3) 1亿数据：

> mapred\_wc1: 1m18.787939375s
>
> mapred\_wc2: 43.700050417s

#### 点评与分析

数据量越大的时候，`mapred_wc2` 的优势越明显。

注意到，无论是 wc1 还是 wc2，mapper 都不做很复杂的工作。worker 线程主要的开销可能来源于创建销毁，以及访问 channel 时的竞争。

因为我们可以看到，在 wc1 中，worker 线程在从 input channel 中取到数据之后，它很快就能处理完并写入 result channel，当 worker 数量增加之后，多个 worker 会同时向同一个 channel 中写数据，这会导致可能的竞争问题，从而拖累性能。另外，多个 worker 从 input channel 中取数据也可能存在竞争。

而在 wc2 中，把 result channel 细化成了 5 个不同的 word channel，worker 往 word channel
中写数据，虽然也存在多个 workers 同时向同一个 word channel 写数据而导致竞争的情况，但是几率会小很多。其他方面，worker 的开销是与 wc1 中差不多的。总体来说，性能会有不错的提高。

全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad。
