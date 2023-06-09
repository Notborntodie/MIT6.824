# Lab1

[TOC]



##  翻译

### You Job

我们的工作是实现一个分布式的`MapReduce`，由两个程序组成，`coordinator`和`worker`。 将只有一个`coordinator`进程和一个或多个并行执行的`worker`进程。 在真实系统中，`worker` 会在一堆不同的机器上运行，但对于本实验，您将在一台机器上运行它们。==`worker`将通过 RPC 与`coordinator`交谈==。 

1. 每个`worker`进程都会向`coordinator`请求任务，从一个或多个文件中读取任务的输入。
2. `worker`执行任务，并将任务的输出写入一个或多个文件。 
3. `coordinator`应该注意到`worker`是否没有在合理的时间内完成任务（对于本实验，使用十秒），并将相同的任务交给不同的`worker`。

我们已经给了你一些代码来让你开始。`coordinator`和的`worker`主要例程在 `main/mrcoordinator.go`和 `main/mrworker.go` 中； 不要更改这些文件。 您应该将您的实现放在`mr/coordinator.go`、`mr/worker.go `和 `mr/rpc.go`中。

以下是如何在字数统计 `MapReduce` 应用程序上运行您的代码。 首先，确保字数统计plugin是全新构建的：

```shell
$ go build -buildmode=plugin ../mrapps/wc.go
```

*  *为什么要重新构建plugin,这是因为map/reduce可以说是应用层的东西，是调用Mapreduce库的用户要做的事情*

### A few rules:

* `map`阶段应该将中间键划分到nReduce个桶中，其中 nReduce 是`reduce`任务的数量.——`main/mrcoordinator.go`传递给`MakeCoordinator()`的参数。 每个`mapper`都应该创建 nReduce 间文件以供`reduce`任务使用。
* `worker` 实现应该将第 X 个`reduce`任务的输出放在文件 mr-out-X 中。
* mr-out-X 文件应包含一行每个 `reduce `函数输出。该行应使用 Go "%v %v" 格式生成。 查看 `main/mrsequential.go` 中注释为“这是正确格式”的行。 如果您的实现与此格式偏离太多，测试脚本将失败。
* 您可以修改 mr/worker.go、mr/coordinator.go 和 mr/rpc.go。 您可以临时修改其他文件进行测试，但请确保您的代码适用于原始版本； 我们将使用原始版本进行测试。
  worker 应该将中间`map`输出放在当前目录中的文件中，您的 worker 稍后可以在其中读取它们作为 Reduce 任务的输入。
* main/mrcoordinator.go 期望 mr/coordinator.go 实现一个 Done() 方法，该方法在 MapReduce 作业完全完成时返回 true； 那时，mrcoordinator.go 将退出。
* 当作业完全完成时，worker进程应该退出。 
  1. 一个简单的实现方法是使用 `call()` 的返回值：如果 worker 未能联系coordinator，它可以假设coordinator.已经退出，因为工作已完成，因此 worker 也可以终止。 
  2. 您可能还会发现coordinator可以给worker的“请退出”伪任务很有帮助。

### Hints

* [指导](https://pdos.csail.mit.edu/6.824/labs/guidance.html)有一些关于开发和调试的提示。

* 开始的一种方法是修改 mr/worker.go 的 Worker() 以将 RPC 发送给协调器以请求任务。 然后修改协调器以响应尚未启动的map的文件名。 然后修改 worker 以读取该文件并调用应用程序 Map 函数，如 mrsequential.go 中所示。

* 应用程序 Map 和 Reduce 函数在运行时使用 Go 插件包从名称以 .so 结尾的文件加载。
  如果您更改了 mr/ 目录中的任何内容，您可能必须重新构建您使用的任何 MapReduce 插件，例如

  ```shell
   go build -buildmode=plugin ../mrapps/wc.go
  ```

  该Lab依赖于worker共享同一个文件系统 当所有worker都在同一台机器上运行时，这很简单，但如果worker在不同机器上运行，则需要像 ==GFS 这样的全局文件系统==。

  *这==太重要了==，事实上，我一开始在思考被split的文件放在哪里，因为中间放在map的本地。*

  所以事实上来说，现在coordinator的RPC只需要考虑文件名

* 中间文件的合理命名约定是 mr-X-Y，其中 X 是 Map 任务编号，Y 是 reduce 任务编号。
  worker 的 map 任务代码将需要一种方法来将中间键/值对存储在文件中，这种方式可以在 reduce 任务期间正确读回。 一种可能是使用 Go 的 encoding/json 包。 将 JSON 格式的键/值对写入打开的文件：

  ```go
  enc := json.NewEncoder(file)
    for _, kv := ... {
      err := enc.Encode(&kv)
  ```

  ```go
  dec := json.NewDecoder(file)
    for {
      var kv KeyValue
      if err := dec.Decode(&kv); err != nil {
        break
      }
      kva = append(kva, kv)
    }
  ```
  
* worker 的 map 部分可以使用 ihash(key) 函数（在 worker.go 中）为给定的键选择 reduce 任务。
  您可以从 mrsequential.go 中窃取一些代码，用于读取 Map 输入文件、对 Map 和 Reduce 之间的中间键/值对进行排序以及将 Reduce 输出存储在文件中。
* 协调器作为RPC服务器，会并发； 不要忘记锁定共享数据。
* 使用 Go 的竞态检测器和 go run -race。 test-mr.sh 开头有个注释告诉你如何用-race 运行它。 当我们对您的实验室进行评分时，我们不会使用竞争检测器。 然而，如果您的代码存在竞争，即使没有竞争检测器，当我们测试它时它也很有可能会失败。
* worker有时需要等待，==例如在最后一个map完成之前，reduce 无法开始==。 一种可能性是`worker`定期向协调员询问工作，在每次请求之间使用 time.Sleep() 休眠。 另一种可能性是协调器中的相关 RPC 处理程序有一个等待循环，使用 time.Sleep() 或 sync.Cond。 Go 在其自己的线程中为每个 RPC 运行处理程序，因此一个处理程序正在等待的事实不需要阻止协调器处理其他 RPC。
* 协调器无法可靠地区分崩溃的 worker、还活着但由于某种原因停止的 worker 以及正在执行但速度太慢而无用的 worker。 您能做的最好的事情就是让协调员等待一段时间，然后放弃并将任务重新发布给其他`worker`。 对于这个实验，让协调员等待十秒钟； 之后协调器应该假设工人已经死亡（当然，它可能没有）。
  如果您选择实施备份任务（第 3.6 节），请注意我们测试您的代码不会worker执行任务而不会崩溃时安排无关的任务。 备份任务应该只安排在一段相对较长的时间（例如 10 秒）之后。
  要测试崩溃恢复，您可以使用 mrapps/crash.go 应用程序插件。 它在 Map 和 Reduce 函数中随机退出。
* 为了确保在出现崩溃时没有人观察到部分写入的文件，`MapReduce` 论文提到了使用临时文件并在完全写入后自动重命名它的技巧。 您可以使用 ioutil.TempFile 创建临时文件并使用 os.Rename 自动重命名它。
* test-mr.sh 在子目录 mr-tmp 中运行其所有进程，因此如果出现问题并且您想查看中间文件或输出文件，请查看那里。 随意临时修改 test-mr.sh 以在测试失败后退出，这样脚本就不会继续测试（并覆盖输出文件）。
  test-mr-many.sh 连续多次运行 test-mr.sh，您可能希望这样做以发现低概率错误。 它以运行测试的次数作为参数。 您不应该并行运行多个 test-mr.sh 实例，因为协调器将重用同一个套接字，从而导致冲突。
* Go RPC 仅发送名称以大写字母开头的结构字段。 子结构也必须有大写的字段名称。
* 调用 RPC call() 函数时，回复结构应包含所有默认值。 RPC 调用应如下所示：
  
  ```go
    reply := SomeType{}
    call(..., &reply)
  ```
  
  
  不要在`call`之前设置任何回复字段。 如果您传递具有非默认字段的回复结构，RPC 系统可能会静默返回不正确的值。



## 思路和数据结构



首先分为两部分

1. `coordinator`

   *  维护一个taskNode的双向链表，供worker和coordinator建立联系，并尝试申请任务时调度

   * RPC 方法

     1. `GetTask`

     2. `FnishTask`

   * `done`,类似于定时器，定时查看链表是否为空，并将超过10s未完成的task重新分配（即修改taskNode一个字段）

2. `worker`

   1. `map`
   2. `reduce`

## 代码



## 运行与调试

<img src="http://cdn.zhengyanchen.cn/img202303100747904.png" alt="截屏2023-03-10 07.47.25" style="zoom:25%;" />



## Questions an Loggings

* ==论文提到==在reduce函数中，value集合是以迭代器方式提供的，防止中间的value集合过大造成内存溢出。

  (这是如何做到的？在Lab并没有体现)

* ==论文提到==。重新执行(re-execution)用作容错的主要机制

  hint写到

  Backup tasks should only be scheduled after some relatively long period of time (e.g., 10s).

  ==在`done`函数里写`backup task`==, 但是不能直接将所有的`taskNode`的状态粗暴地由true转化为false,所以为每个节点安排了一个time字段。由于`done`函数每1s执行一次

* 第一次写的时候，果然还是出现了没加锁的情况。使用go run -race 发现存在race condition

* 写的双向循环链表存在问题，所以直接就使用双向链表就好了

* Go RPC is a simple form of "at-most-once"

  1. open TCP connection
  2. write request to TCP connection
  3. Go RPC never re-sends a request

  策略就是worker如果调用`call`失败，那么就结束进程

* ==在最后一个map完成之前，reduce 无法开始==,那么就安排两个task链表

* NReduce和gaptime的大小很玄妙
