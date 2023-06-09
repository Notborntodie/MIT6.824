## Lab2A：领导者选举

实现 Raft 领导者选举和心跳（没有日志条目的 `AppendEntries RPC`）。

2A 部分的目标是

1. 选举出一个领导者，如果没有故障，领导者保持领导者身份；

2. 如果原领导者故障或丢失与/来自原领导者的数据包，新领导者接管。运行 go test -run 2A 来测试您的 2A 代码。

   

* 您不能轻易直接运行您的 Raft 实现；相反，您应该通过测试程序运行它，即 go test -run 2A。

- 遵循论文的图 2。目前，您关心的是==发送和接收 RequestVote RPC==，与==选举相关的服务器规则==以及与领导者选举相关的状态。

- 将图 2 中的领导者选举状态添加到 raft.go 中的 Raft 结构中。您还需要定义一个结构来保存有关每个日志条目的信息。

  

- 填充 RequestVoteArgs 和 RequestVoteReply 结构。修改 Make() 以创建一个后台 goroutine，该 goroutine 在一段时间内没有收到其他节点的消息时定期通过发送 RequestVote RPC 来发起领导者选举。这样，节点将了解谁是领导者（如果已经有领导者）或者自己成为领导者。实现 RequestVote() RPC 处理程序，以便服务器互相投票。

- 要实现心跳，定义一个 AppendEntries RPC 结构（尽管您可能还不需要所有参数），并让领导者定期发送它们。编写一个 AppendEntries RPC 处理程序方法，重置选举超时，以便在已经选举出领导者时，其他服务器不会向前迈出领导者角色。

- ==确保不同节点中的选举超时不会始终同时触发== （随机），否则所有节点都只会为自己投票，没有人会成为领导者。

- 测试程序要求领导者每秒不超过十次地发送心跳 RPC。

- 测试程序要求在旧领导者失败后的五秒内选举出新领导者（如果多数节点仍然可以通信）。但请记住，如果出现分裂投票（由于数据包丢失或候选人不幸选择相同的随机回退时间），选举可能需要多轮。您必须选择足够短的选举超时（因此心跳间隔），以便即使需要多轮，选举也很可能在五秒内完成。

- 该论文的第5.2节提到选举超时范围为150至300毫秒。只有当领导者发送心跳频率远远超过每150毫秒一次（例如，每10毫秒一次）时，这样的范围才有意义。因为测试器每秒只能限制你发送十几个心跳，所以你将不得不使用比论文中150到300毫秒更长的选举超时时间，但也不能太长，否则你可能无法在五秒钟内选举出一位领导者。

* 您可能会发现 Go 的 rand 函数很有用。 
* 您需要编写能在一段时间后或周期性地执行操作的代码。最简单的方法是创建一个带有循环并调用 time.Sleep() 的 goroutine；请参考 Make() 函数中为此目的创建的 ticker() goroutine。不要使用 Go 的 time.Timer 或 time.Ticker，因为它们很难正确使用。

* 实验指导页面上有一些关于如何开发和调试代码的提示。

* 如果您的代码无法通过测试，请再次阅读论文中的图 2；领导者选举的完整逻辑分布在图中的多个部分。

* 不要忘记实现 GetState() 函数。

* 当测试程序永久关闭 Raft 实例时，它会调用您的 Raft 的 rf.Kill() 方法。您可以使用 rf.killed() 函数检查是否调用了 Kill()。您可能希望在所有循环中执行此操作，以避免已关闭的 Raft 实例打印出令人困惑的信息。

* Go RPC 仅发送以大写字母开头的结构体字段。子结构的字段名称也必须以大写字母开头（例如，数组中日志记录的字段）。labgob 包会对此发出警告；不要忽略这些警告。

## raft设计

### 状态设计

在Raft中，服务器的状态被分为持久状态和易失状态。这里是它们的设计：

**所有服务器上的持久状态**（在响应RPC之前更新到稳定存储）：

- `currentTerm`：服务器已知的最新任期（首次启动时初始化为0，单调递增）。
- `votedFor`：在当前任期中获得投票的候选者ID（如果没有，则为null）。
- `log[]`：日志条目数组；每个条目包含状态机的命令以及领导者接收条目时的任期（第一个索引为1）。

**所有服务器上的易失状态**：

- `commitIndex`：已知提交的最高日志条目的索引（初始化为0，单调递增）。
- `lastApplied`：应用到状态机的最高日志条目的索引（初始化为0，单调递增）。

==`commitIndex` 是一个全局的概念，表示在整个集群中已知已提交的最高日志条目。相比之下，`lastApplied` 是一个本地概念，表示在某个特定服务器上已经应用到状态机的最高日志条目==

**领导者上的易失状态**（在选举后重新初始化）：

- `nextIndex[]`：针对每个服务器，要发送给该服务器的下一个日志条目的索引（初始化为领导者最后一个日志条目的索引+1）。
- `matchIndex[]`：针对每个服务器，已知在服务器上复制的最高日志条目的索引（初始化为0，单调递增）。

### RequestVote RPC

被候选人调用以收集选票（§5.2）。

* 参数
  1. term：候选人的任期 
  2. candidateId：请求投票的候选人 
  3. lastLogIndex：候选人的最后日志条目索引（§5.4）
  4.  lastLogTerm：候选人的最后日志条目的任期（§5.4）

* 结果：
  1. term：当前任期，供候选人更新自身 
  2. voteGranted：真值表示候选人收到了选票

接收者实现：

1. 如果 term < currentTerm，则回复 false（§5.1）
2. 如果 votedFor 为空或等于 candidateId，且候选人的日志至少与接收者的日志一样新，则授予选票（§5.2, §5.4）

### 服务器规则rules：

所有服务器：

- 如果 commitIndex > lastApplied：递增 lastApplied，将 log[lastApplied] 应用于状态机（§5.3）
- 如果 RPC 请求或响应包含任期 T > currentTerm：设置 currentTerm = T，转换为跟随者（§5.1）

跟随者（§5.2）：

- 响应来自候选人和领导者的 RPC
- 如果在没有收到来自当前领导者的 AppendEntries RPC 或给候选人投票的情况下，选举超时：转换为候选人

候选人（§5.2）：

- 转换为候选人时，开始选举：
  - 递增 currentTerm
  - 为自己投票
  - 重置选举定时器
  - 向所有其他服务器发送 RequestVote RPC
  - 如果从大多数服务器获得选票：成为领导者
  - 如果从新领导者收到 AppendEntries RPC：转换为跟随者
  - 如果选举超时：开始新选举

领导者：

- 当选后：向每个服务器发送初始空的 AppendEntries RPC（心跳）；在空闲期间重复发送，以防止选举超时（§5.2）
- 如果从客户端收到命令：将条目附加到本地日志，在条目应用于状态机后响应（§5.3）
- 如果最后一个日志索引 ≥ 跟随者的 nextIndex：发送从 nextIndex 开始的包含日志条目的 AppendEntries RPC
- 如果成功：更新跟随者的 nextIndex 和 matchIndex（§5.3）
- 如果 AppendEntries 失败，因为日志不一致：递减 nextIndex 并重试（§5.3）
- 如果存在一个 N，使得 N > commitIndex，大多数 matchIndex[i] ≥ N，且 log[N].term == currentTerm：设置 commitIndex = N（§5.3, §5.4）。

### Append RPC

参数：

- term：任期
- leaderId：领导者 ID
- prevLogIndex：前一个日志条目的索引
- prevLogTerm：前一个日志条目的任期
- entries[]：要存储的日志条目（对于心跳为空；为提高效率，可以发送多个）
- leaderCommit：领导者的 commitIndex

结果：

- term：领导者的任期
- success：如果跟随者包含与 prevLogIndex 和 prevLogTerm 匹配的条目，则返回 true

接收器实现：

1. 如果 term < currentTerm，则回复 false（见论文第 5.1 节）
2. 如果日志中没有在 prevLogIndex 位置且任期与 prevLogTerm 匹配的条目，则回复 false（见论文第 5.3 节）
3. 如果现有条目与新条目冲突（索引相同但任期不同），则删除现有条目及其后续所有条目（见论文第 5.3 节）
4. 将所有尚未在日志中的新条目追加到日志中
5. 如果 leaderCommit > commitIndex，则设置 commitIndex = min(leaderCommit, 最后一个新条目的索引)





## 测试的函数









## chatGPT

### 收到 RequestVote RPC的跟随者应该需要对votedFor字段做类似test_and_set的原子操作？

是的，处理 RequestVote RPC 时，跟随者需要对 `votedFor` 字段进行类似于原子操作的处理，以确保在给候选人投票时避免数据竞争和不一致的情况。在 Go 语言中，您可以使用互斥锁（Mutex）或读写锁（RWMutex）来实现这种原子操作。

例如，在处理 RequestVote RPC 时，您可以这样实现：

```go
goCopy code
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
    rf.mu.Lock()
    defer rf.mu.Unlock()

    // ... 处理 RequestVote RPC ...
}
```

在这个例子中，`rf.mu` 是一个互斥锁，它可以确保在处理 RequestVote RPC 时，同一时刻只有一个 goroutine 可以访问和修改 `votedFor` 字段。

### t *testing.T这是什么？

`testing.T` 是 Go 语言标准库 `testing` 包中的一个类型。它提供了一些方法，用于编写测试用例，报告测试过程中的错误和失败，以及获取测试相关的信息。在 Go 的测试函数中，通常会将 `*testing.T` 类型的参数传递给测试函数，使得测试函数可以使用 `testing.T` 的方法来报告测试结果。

例如，下面是一个简单的 Go 测试用例：

```go
goCopy code
package mypkg

import "testing"

func TestMyFunction(t *testing.T) {
    result := MyFunction()
    expected := 42
    if result != expected {
        t.Errorf("MyFunction() returned %d, expected %d", result, expected)
    }
}
```

在这个测试用例中，我们调用 `MyFunction()` 函数并检查其返回值是否与预期值相等。如果不相等，我们使用 `t.Errorf()` 方法来报告错误。这个方法会记录错误信息，并在测试结束后将错误信息输出到控制台。

### `checkOneLeader()` 函数？

`checkOneLeader()` 函数的目的是检查在配置（config）中是否只有一个领导者。这个函数的主要逻辑如下：

1. 对于 10 次迭代：
   - 等待一个随机的时间（450ms 到 550ms 之间）。
   - 对于每个 Raft 节点 `i`，检查它是否被认为是领导者。如果是，将其添加到一个名为 `leaders` 的 map 中，其中 key 是节点的任期（term）。
2. 然后，遍历 `leaders` map，检查是否存在一个任期有多个领导者的情况。如果有，使用 `t.Fatalf()` 方法报告错误，因为这违反了 Raft 协议。
3. 如果在迭代过程中找到了领导者，返回具有最高任期的领导者。
4. 如果 10 次迭代都没有找到领导者，使用 `t.Fatalf()` 方法报告错误，因为预期至少有一个领导者。

通过这个方法，`checkOneLeader()` 函数确保在给定的 Raft 集群中，只有一个节点被认为是领导者。

### 这里提到The paper's Section 5.2 mentions election timeouts in the range of 150 to 300 milliseconds. Such a range only makes sense if the leader sends heartbeats considerably more often than once per 150 milliseconds (e.g., once per 10 milliseconds). Because the tester limits you tens of heartbeats per second, you will have to use an election timeout larger than the paper's 150 to 300 milliseconds, but not too large, because then you may fail to elect a leader within five seconds. 选举时间应该在5秒以内，但是为什么在测试函数里面 “time.Sleep(50 * time.Millisecond)”，等了50m就开始查看是否已经有领导者当选？

这里的 `time.Sleep(50 * time.Millisecond)` 是给 Raft 服务器留出一定时间来进行选举并选出领导者。根据 Raft 论文的建议，选举超时通常设置在 150 到 300 毫秒之间，以确保在合理的时间内选出领导者。然而，由于测试环境可能会对心跳次数施加一定限制，实际的选举超时可能需要设置得更大一些，但不能太大，否则可能无法在 5 秒内选出领导者。

在这个测试用例中，`time.Sleep(50 * time.Millisecond)` 是用来确保在检查领导者之前，各个 Raft 服务器有足够的时间进行选举。这个等待时间不是固定的，可以根据实际情况进行调整。它的目的是确保在检查领导者之前，各个服务器有足够的时间进行通信和选举。注意，这个 50 毫秒的等待时间不是选举超时时间，而是一个独立于选举超时时间的等待时间。
