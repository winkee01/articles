## 简介

从前一篇文章介绍的 MapReduce 的概念，我们知道 mapper 是用来处理每一个数据的，

一堆数据，分成一小份一小份，不断的发给 mapper，mapper 处理完就发给 reducer 继续处理，我们可以把这个过程称为 emit 或 feed。

reducer 每收到一个 result，就在之前的结果上加一（即 `result += 1`），则不断的累加，最终得到总的统计结果。

Python 中的 map 函数原型是 `map(mapper_func, iterable, ...)`

这个 map() 跟我们另一篇文章中介绍的 Master Program（也即 mapred 程序）的作用非常相似，其第一个参数 `mapper_func` 跟 MapReduce 概念中的 mapper 作用类似，后面的 iterable 则是相当于输入数据（比如 Stdin）

这个 iterable 是可以是单个元素本身，也可以是可遍历的列表类元素，比如 list, dictionary, 或者 Go 中的 channel 之类的。

只要它是 iterable，那么就可以每次 feed 一个元素给 `mapper_func`。

我们来看一个简单的例子：

```python
def square(x):
    return x * x

numbers = [1, 2, 3, 4, 5]

# Apply the square function to each number in the list
squared_numbers = map(square, numbers)

# Convert map object to list to see results
print(list(squared_numbers))
```

输出结果： \[1, 4, 9, 16, 25]

解释：
map() 函数把 square 当作 mapper，把 numbers 列表中的每个元素会被一个个 feed 给 `square` 函数进行处理，并最终返回了一个列表。这里没有进行 reduce 处理，而是由 mapper 处理完直接输出。


lambda 语法更简洁：

```python
numbers = [1, 2, 3, 4, 5]
squared_numbers = map(lambda x: x * x, numbers)
print(list(squared_numbers))
```

mapper 还可以同时应用到多个列表

```python
numbers1 = [1, 2, 3]
numbers2 = [10, 20, 30]

result = map(lambda x, y: x * y, numbers1, numbers2)

print(list(result))
```

输出结果：\[10, 40, 90]

这里是把多个列表中的元素各选一个一起 feed 给 mapper 处理，最终返回一个列表作为结果。

#### reduce 函数

前面的 map() 函数是在应用 mapper 对每个元素处理完后直接输出结果，并没有进一步 reduce 操作（比如把结果再加总之类的）。如果我们想要进一步执行 reduce 操作，我们需要使用标准库 functools 中的函数。

`reduce` 函数则是来自标准库 functools。它实现了一个叫 folding 或 reduction 的功能。（Tip: `map()` 函数是 Python 语言原生自带的）

**reduce 函数** 的目的就是不断的累加，直到最终返回一个累加后的总值。**reduce** 非常有 functional programming 的味道。

`reduce()` is useful when you need to apply a function to an iterable and **reduce it to a single cumulative value**.

###### reduce 函数原型：

    functools.reduce(reducer_func, iterable[, initializer])

它的内部实现可能是这样：

```python
def reduce(reducer_func, iterable, initializer=None):
    it = iter(iterable)
    if initializer is None:
        value = next(it)
    else:
        value = initializer
    for element in it:
        value = function(value, element)
    return value
```

`initializer` 就是初始值，假如我们要处理的 iterable 是一组 integer，那么 `initializer` 默认就是 0。

function 的作用就是接收一个 new\_value，把这个 new\_value 累加到 old\_value 上，并返回累加后的结果。这个累加后的结果又作为下一次的 old\_value。而 reduce 的作用就是把 iterable 中的元素一个个的发给 function 去累加。

接下来我们把 map 和 reduce 两个函数结合起来，实现一个 MapReduce 效果：

```python
from functools import reduce

def add(x, y):
    return x + y

numbers = [1, 2, 3, 4, 5]

squared_numbers = map(lambda x: x * x, numbers)

sum_of_squares = reduce(add, squared_numbers)

print(sum_of_squares)
```

输出结果：55


#### Word Counter

Word Counter 是 MapReduce 世界中最常见的实战案例，我们接下来实现一个简单的 word counter

```python
from functools import reduce
from collections import Counter

text = ["hello world", "hello", "world hello"]

# Map phase: Split each line into words
word_lists = map(lambda line: line.split(), text)

# Flatten the list of lists into a single list of words
words = reduce(lambda x, y: x + y, word_lists)

# Reduce phase: Count the occurrences of each word
word_counts = Counter(words)

# Output the word count
print(word_counts)
```

输出结果：Counter({'hello': 3, 'world': 2})

##### 点评与分析：

**mapper** 决定了数据以什么样的形式发给 **reducer**，比如是把输入的每个数据处理成 `<hello, 2>, <world, 1>` 这样的键值对，还是 `text.split()` 这样的只是把 string 分割成一个个 word, 发给 **reducer**。

**reducer** 则决定了怎么把一个个收到的数据 **“累加”** 成一个最终的结果。比如，上述的程序里的 reducer 的作用就是把收到的 word 不断的拼接，组成一个长串。

Counter 这个函数对一个字符串中的所有 word 统计数量。简单来说就是，前面的 map, reduce 都是为了把数据整合成 Counter 所能接收的格式。

###### reduce 再度思考：

其实 reduce 不仅可以累加，还可以进行任意操作，我们只需要记住它是一个二元操作，第一个操作数是 `base_value` （或 `old_value`），第二个数是 `new_value`，`new_value` 与 `base_value` 进行一个操作（运算或比较大小）后形成一个新的 `base_value`，然后不断重复这个过程，直到完成处理完所有元素为止。

```python
>>> from functools import reduce

>>> numbers = [3, 5, 2, 4, 7, 1]

>>> # Minimum
>>> reduce(lambda a, b: a if a < b else b, numbers)
1

>>> # Maximum
>>> reduce(lambda a, b: a if a > b else b, numbers)
7
```

#### 警告：

从上述程序的工作原理我们可以知道，reduce 的性能好不了，因为它需要不断的 reducer （也即 lambda 函数）。而且，它的可读性也不好。

如果我们想要它所能实现的上述功能，我们应该考虑使用其他效率更高的库函数，比如：`sum()`, `all()`, `any()`, `max()`, `min()`, `len()`, `math.prod()`。尤其要注意，即便我们一定要使用 reduce 模型，也要避免很复杂的自定义函数。

最后，我们再来看一个 Python 的 MapReduce 例子加深理解：

```python
from collections import defaultdict

def mapper(word):
    return word, 1

def reducer(kv_pair):
    key, values = kv_pair
    return key, sum(values)

def map_reduce(input_list, mapper, reducer):
    map_results = map(mapper, input_list)
    shuffler = defaultdict(list)
    for key, value in map_results:
        shuffler[key].append(value)
    return map(reducer, shuffler.items())

if __name__ == "__main__":
    words = "hello how are you and you how old are you".split(" ")
    result = list(map_reduce(words, mapper, reducer))
    print(result)
```

我画了一个示意图来展示整个流程：

![](https://us-article-images.oss-cn-shanghai.aliyuncs.com/screenshots/map_red_python.jpg)

全文完！

如果你喜欢我的文章，欢迎关注我的微信公众号 codeandroad。
