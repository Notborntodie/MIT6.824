#  Lecture 4: Primary/Backup Replication

[TOC]

Today
 Primary/Backup Replication for Fault Tolerance
Case study of VMware FT (2010), an extreme version of the idea

## Plan

* 使用Primary/Backup Replication可以做到哪些容错
* 面临的挑战 challenges
*  Replication的两种方法
* study case: VM FT  



## 复制可以对付什么

### What kinds of failures can replication deal with?

Replication is good for "fail-stop" failure of a single replica

复制适用于与fail-stop的场景

1. fan stops working, CPU overheats and shuts itself down

2. someone trips over replica's power cord or network cable

3. software notices it is out of disk space and stops

   **复制无法处理很多其他错误**，Replication may not help with bugs or operator error

**但是可以把一些故障转变为fail-stop failure**

  How about earthquake or city-wide power failure?
    Only if replicas are physically separated

即便这样，也很难建立容错系统



## 挑战

* Has primary failed ?

  在分布式系统中，无法区分a network partition and a machine failure

  我们需要避免陷入split-brain的情况(==多个primary==)

*  Keep primary/backup in sync（如何保持同步） 
  1. apply changes in order
  2. **avoid non-determinism**，避免非确定性的

* Fail over,即当故障发生时，如何从primary切换到backup



## 两种方法



1. State transfer **一台机器做**，发生备份快照
       Primary executes the service
       Primary sends state snapshots over network to a storage system
       On failure:
         Find a spare machine (or maybe there's a dedicated backup waiting)
         Load software, load saved state, execute

   发生快照

2. Replicated state machine(RSM) **所有机器做**， 发送操作
       Clients send operations to primary,
         primary sequences and sends to backups
       All replicas execute all operations
       If same start state,
         same operations,
         same order,
         deterministic,
         then same end state

两种方法的优缺点

1. * State transfer is conceptually **simple**

   * But state may be **large**, **slow** to transfer over network

2. * Replicated state machine often generates less network traffic
     Operations are often **small** compared to state
   * But **complex** to get right

**VM-FT uses replicated state machine**, as do Labs 2/3/4

## Today's paper (VMware FT)

复制操作的等级，分为两种

* Application state  ：Application must understand fault tolerance

   GFS works this way
    Efficient; primary only sends high-level operations to backup

   GFS采取这种方式

* Machine-level state  e.g. registers and RAM content?

   VMware FT则采取这种
  
  Today's paper (VMware FT) replicates machine-level state
  Transparent: can run any existing O/S and server software!
  
  Appears like a single server to clients

具体方法就是：如果做到通过**虚拟化**来做到 machine-level 的复制，paper只支持one-core

虚拟机监视器：**hypervisor == monitor == VMM (virtual machine monitor)**

物理机一个主机，一个备份机器  O/S+app is the "guest" running inside a virtual machine

<img src="http://cdn.zhengyanchen.cn/img202303161003158.png" style="zoom:67%;" />

Vmm会处理外部的中断，

1. 一方面送给本地虚拟机
2. 另一方面送入logging channel to a backup 



## 基本思路：

 Primary和backup最初具有相同的内存和寄存器 包括相同的软件（操作系统和应用程序） 大多数指令在primary和backup上执行时是相同的 例如，ADD指令 因此，在大多数情况下，不需要做任何工作就可以使它们保持相同！

<img src="http://cdn.zhengyanchen.cn/img202303181035855.png" alt="截屏2023-03-18 10.35.16" style="zoom:50%;" />

这些具有确定性的指令并不需要主机发送到备份机上

什么时候主机必须向备份发送信息？ 

* 任何可能导致它们执行不同的情况，执行指令的==非确定性后果==。

即FT需要消除分歧

FT必须消除哪些来源的分歧？ 主要以下三种

* 例如读取当前时间non-deterministic 

* 外部世界的输入——网络数据包和磁盘读取。 这些出现为DMA的数据加上中断。

* 定时器

  多核并发的情况也是不确定的，比如多个线程访问同一个数据，我们也需要保证线程的调度顺序是相同的

  （==单核可以通过定时器保证，因为这里不考虑多核==）

我们将针对这三种情况保证确定性

### 为什么分歧会导致灾难？ 

因为备份的状态会与主机的状态不同， 如果主机随后失败，客户端将看到不一致性。 例如：6.824作业提交服务器 对实验室强制执行午夜截止时间。 一个硬件计时器在午夜时分响起。 让我们使用一个*错误的*FT来复制提交服务器。 在主服务器上，我的作业数据包中断恰好在计时器响起之前到达。 主机会告诉我我会得到全部的作业积分。 在备份上，我的作业数据包在计时器响起之后到达，因此备份认为它已经迟到了。 主机和备份现在具有分歧状态。 目前，由于主机回答所有请求，没有人注意到。 然后主机失败，备份接管，并且课程工作人员看到 备份状态，其中表示我提交晚了！ 因此：备份必须看到相同的事件， 在相同的指令流中，以相同的顺序出现。

需要通过The logging channel来实现

###  The logging channel

  primary sends all events to backup over network
    "logging channel", 

carrying log entries格式 : instruction #, type, data.

发送的事件主要有

1. interrupts, 
2. incoming network packets
3.  data read from shared disk

* FT provides backup's input (interrupts &c) from log entries

* FT suppresses backup's network output

  虽然备份机屏蔽自己的中断，处理这些log来的中断，但它并不发output

  1. shared disk
  2. client

都不会 

以下是处理的两个例子

### FT's handling of timer interrupts

定时器中断的处理和其他中断一样

* Goal: primary and backup should see interrupt at exactly
   **the same point in the instruction stream**

*  Primary:

  1. FT fields the timer interrupt
         FT reads instruction number from CPU
         FT sends "timer interrupt at instruction # X" on logging channel 

  ​	发送指令#X处的定时器中断的log entry

  2. 本地

     FT delivers interrupt to primary, and resumes it
     (==relies on CPU support to direct interrupts to FT software==)

* Backup:
      ==ignores its own timer hardware==忽略自己的定时器
      FT sees log entry *before* backup gets to instruction # X
      FT tells CPU to transfer control to FT at instruction # X
      FT mimics a timer interrupt that backup guest sees
      (relies on CPU support to jump to FT after the X'th instruction)

### FT's handling of network packet arrival (input)

1. Primary


   ​    FT configures NIC to write packet data into FT's private "bounce buffer"

   ​	VMM会有自己的缓冲区

   ​    At some point a packet arrives, NIC does DMA, then interrupts

   ​	发生中断之后，处理就类似

   ​    FT gets the interrupt, reads instruction # from CPU
   ​    FT pauses the primary
   ​    FT copies the bounce buffer into the primary's memory

   ​	网络包传输进内存会早于内存

   ​    FT simulates a NIC interrupt in primary

   FT sends the packet data and the instruction # to the backup

   处理网络包和中断不同在于还需要发送数据

2. Backup:
       FT gets data and instruction # from log stream
       FT tells CPU to interrupt (to FT) at instruction # X
       FT copies the data to guest memory, simulates NIC interrupt in backup

#### Why the bounce buffer?
  We want the data to appear in memory at exactly the same point in
  execution of the primary and backup.
  So they see **the same thing** if they read packet memory before interrupt.
  Otherwise they may diverge.

总之就是保证一个同步，所以需要先把数据写进内存，避免I/O的不同步

## 存储服务器也是冲裁者

 FT VMM emulates a local disk interface

模拟一个本地文件接口，事实上文件会发送到存储服务器上

<img src="http://cdn.zhengyanchen.cn/img202303161203561.png" alt="截屏2023-03-16 12.03.41" style="zoom:55%;" />

 but actual storage is ==on a network server== -- the "shared disk"
 all files/directories are in the shared storage; no local disks，
 **only primary** talks to the shared disk

实际的存储设备是一个网络服务器，被称为共享磁盘。所有的文件和目录都存储在共享磁盘上，没有本地磁盘。**只有主服务器与共享磁盘进行通信**，备份服务器不直接访问共享磁盘，而是通过主服务器进行间接访问。

primary forwards blocks it reads to the backup
  backup's FT ignores backup app's writes, serves reads from primary's data

备份系统的故障转移模块会忽略备份应用程序的写入，从主系统的数据中提供读取服务

 shared disk makes creating a new backup much faster
    don't have to copy primary's disk

### Storage server还扮演一个特殊的角色，

那就是有一个flag=0位，来冲裁失败后谁将primary

### 让我们来考虑log channel破坏

两个VMFT之间会保持通讯，发出心跳确定彼此的存在

如果二者无法通讯，这时候存储服务器和主系统与备份系统依然保持着联系

<img src="http://cdn.zhengyanchen.cn/img202303181002750.png" alt="截屏2023-03-18 10.02.17" style="zoom:50%;" />



 如果其中任何一个停止能够通过网络相互通信 ，他们就“go lives”并提供唯一的服务 如果主节点go lives，则停止将日志条目发送到备份节点

为了能成为唯一的主机，而不至于出现split-brain的情况，那么primary/backup都会去对原子地`test-and-set`设置flag为1(flag位原为0)

1. 如果backup获得并修改了flag，那么就升级为primary
2. 如果primary获得并修改了flag，则继续
3. 如果primary/backup没有获得，则终止自己
4. 至于如何恢复状态，和恢复flag位，那就是另外一回事



## VMFT的中断设计

<img src="http://cdn.zhengyanchen.cn/img202303181058852.png" alt="截屏2023-03-18 10.58.21" style="zoom:50%;" />

比如第一个中断发送包含指令100的log之后，backup就会停止执行，把权限叫给虚拟机管理器，知道下一个包含指令200的log

所以The backup must **lag by** one log entry

==备份机一定会慢一步==

  Suppose primary gets an interrupt at instruction # X
  If backup has already executed past X, it is too late!
  So backup FT can't execute unless at least one log entry is waiting
    Then it executes just to the instruction # in that log entry
    And waits for the next log entry before resuming 

## Non-deterministic instructions

(对于这些非确定性的指令，VMM会在开始就坚实它们)

鉴于备份机比主机要慢一个log

所以backup并不会立刻执行非确定性的指令

(由虚拟机监视器监控)

<img src="http://cdn.zhengyanchen.cn/img202303181109331.png" alt="截屏2023-03-18 11.09.35" style="zoom:50%;" />

  some instructions yield different results even if primary/backup have same state
  e.g. reading the current time or processor serial #

* 主机Primary:
  
  ​    FT sets up the CPU to interrupt if primary executes such an instruction（==还是通过间接地控制中断来保证同步==）
  
  ​    FT executes the instruction and records the result
  ​    sends result and instruction # to backup
  
* Backup:
      FT reads log entry, sets up for interrupt at instruction #
      FT then supplies value that the primary got, does not execute instruction

## Failure Over

  Primary and backup both execute instructions for output
  Primary's FT actually does the output
  Backup's FT discards the output

Output example: DB server
  clients can send "increment" request
    DB increments stored value, replies with new value
### 正常情况下

​    suppose the server's value starts out at 10
​    network delivers client request to FT on primary

primary's FT sends on logging channel to backup(==同时日志和发给主机==)

FTs deliver request packet to primary and backup
primary executes, sets value to 11, sends "11" reply, FT really sends reply
backup executes, sets value to 11, sends "11" reply, and FT discards
the client gets one "11" response, as expected

### 如果主机crash了

同时发生

1. suppose primary 已经输出了reply=11 ，然后crash

2. 同时the logging channel discards the log entry w/ client request通道也崩溃了

   备份并没有在内存增加这个数据

      但是backup会 goes live

<img src="http://cdn.zhengyanchen.cn/img202303181128977.png" alt="截屏2023-03-18 11.28.33" style="zoom:50%;" />

   但是backup has value "10" in its memory!

出现了不一致 。 now a client sends another increment request。这时候client继续加
  it will get "11" again, not "12"
  oops

### 解决方案: the Output Rule (Section 2.2)

在主机输出之前，必须要确认backup已经全部收到了这些输出log

  before primary sends output (e.g. to a client, or shared disk),
  must wait for backup to acknowledge all previous log entries

==所以如果主机crash了==，client不会收到回应，会重新发出请求。



## Performance (table 1)
  FT/Non-FT: impressive!
    little slow down
  Logging bandwidth
    Directly reflects disk read rate + network input rate
    18 Mbit/s is the max
  The logging channel traffic numbers seem low to me
    Applications can read a disk at a few 100 megabits/second
    So their applications may not be very disk-intensive

### When might FT be attractive?
  Critical but low-intensity services, e.g. name server.
  Services whose software is not convenient to modify.

应用级别的速度会更

### What about replication for high-throughput services?

  People use application-level replicated state machines for e.g. databases.
    The state is just the DB, not all of memory+disk.
    The events are DB commands (put or get), not packets and interrupts.
    Can have short-cuts for e.g. read-only operations.
  Result: less logging traffic, fewer Output Rule pauses.
  GFS use application-level replication, as do Lab 2 &c

## 问题和我的回答

1. Does primary have to wait for backup? 主节点是否必须等待备份节点？

   在输出数据时，必须等

2. How does backup decide to take over?备份节点如何决定接管操作？

   通过log channel的心跳检测来和primary的,由储存器服务器来仲裁

3. Are anomalies visible at cut-over?切换时是否可见异常？

   不可见

4. How to bring a replacement backup up to speed and resume replication? 如何使替代备份节点达到同步并恢复复制？

   

## 思考

Replicated state machine要保证的就是一个state完全相同，要保证两点

1. 执行指令的顺序完全相同

2. 机器资源如寄存器，内存完全同步，这并不是意味着同一个时间上的完全同步，而是在第一点的基础上，执行到指令流的某个点时的完全同步

在我看来主要就是通过两点来保证

1. Log channel,这是通道
2. 控制一切异步中断

*您的理解是正确的。Replicated state machine需要保证每个副本在执行相同的命令序列时产生完全相同的状态，因此需要确保两个关键点：命令执行顺序和状态同步。为了满足这些要求，一种常用的方法是通过日志复制和异步中断来实现。副本之间的命令序列通过日志通道进行传输和复制，这样可以确保每个副本按照相同的顺序执行命令。同时，异步中断机制用于确保在任何时间点上，每个副本的状态都能够完全同步。这些中断可以是周期性的，也可以是由其他事件触发的*



## Q&A with chatGPT

### 这里的network partition 指什么？

在分布式系统中，"network partition" 指的是网络分区，也称为网络分裂或网络隔离。这种情况发生在分布式系统中的网络连接出现问题，导致系统中的节点之间无法相互通信。当网络分区发生时，系统中的节点被分成两个或更多个不同的部分，这些部分之间无法相互通信，这可能导致系统的一些功能或服务无法正常运行。

### State transfer is conceptually simple  But state may be large, slow to transfer over network，所以快照定时保存对吧？

Yes, that is correct. State transfer refers to the process of transferring the current state of an application or system from one location to another. While the concept of state transfer is simple, the size of the state data can be large, making it slow to transfer over a network.

To address this issue, it is common to use snapshotting, which involves periodically taking a snapshot of the current state and saving it to a storage location. This way, if there is a need to transfer the state,**only the changes made since the last snapshot need to be transferred,**（==快照只保存变化==） which can significantly reduce the amount of data that needs to be transferred over the network. Additionally, having multiple snapshots allows for point-in-time recovery, which can be useful in the event of a failure or data corruption.

### 所以是类似日志？

Yes, snapshotting can be thought of as similar to logging in that it captures a snapshot of the current state at a particular point in time. However, while logging typically captures every change made to the system, **snapshotting only captures the state at specific intervals.** (==快照只会定时保存==) This makes snapshotting more efficient for transferring and storing state data than logging, as it reduces the amount of data that needs to be transferred and stored. Additionally, snapshotting can provide a more comprehensive and reliable means of restoring the system to a previous state in the event of failure or data corruption.

### 为什么client不直接把操作发给backup

需要让所以的操作时有确定性

在Primary/Backup Replication system中，客户端不直接把操作的数据直接发送给Backup的原因是为了确保数据的一致性和可靠性。

### 但是GFS中客户端是直接给所有chunkserer发送了数据，只给primary发生了append的操作指令

您提到的情况是正确的。在Google文件系统（GFS）中，客户端会直接向所有存储块（chunk）服务器发送数据。这是因为GFS采用了一种称为“租约”的机制来管理文件的写入，这种机制允许多个客户端并发地写入同一文件。在这个机制中，每个文件有一个主chunkserver作为“租赁持有者”（lease holder），主chunkserver负责管理文件的写入。

当一个客户端向GFS写入数据时，它会直接向所有存储块（chunk）服务器发送数据，并向主chunkserver发送一个append指令来请求将数据附加到文件的末尾。主chunkserver将这个append操作的结果反馈给客户端，客户端就可以继续进行下一个写操作了。

这种设计的优点是可以提高写入性能，因为客户端可以并行地向多个存储块服务器发送数据。此外，通过将append操作的请求发送给主chunkserver来进行协调，可以保证写入的一致性和可靠性。

### 分布式系统中的 Transparent是什么意思

具体来说，"Transparent" 是指在分布式系统中，应用程序或用户不需要知道系统的分布式特性，也不需要进行任何特殊的配置或编程。分布式系统应该提供透明的接口和服务，使得应用程序和用户可以像使用单个计算机一样使用分布式系统。这包括透明地处理故障恢复、负载平衡、数据复制和分布式协作等任务。

== ==



### 难点似乎在于如果一台机器发生了非同步的中断，那么另一台机器并不能知道？

您提到的确实是分布式系统中的一个难点，即处理机器之间的异步中断或故障。由于网络的不可靠性和延迟，一个机器可能无法立即获知其他机器发生的故障或状态变化，这可能导致系统的不一致性和错误。因此，分布式系统需要设计和实现相应的机制来处理这些问题。

例如，分布式系统可以使用心跳机制来检测节点的健康状况。每个节点定期向其他节点发送心跳信号，以表示自己的状态和可用性。如果某个节点连续若干个心跳信号没有回应，则认为该节点已经故障，需要进行故障恢复。另外，分布式系统还可以使用分布式一致性协议，例如Paxos、Raft等，来确保数据的一致性和节点之间的同步。

处理分布式系统中的异步中断和故障需要设计和实现复杂的算法和机制，这也是分布式系统设计和实现的一个重要挑战。同时，分布式系统的可靠性和性能也取决于这些算法和机制的质量和效率。

### 但也可以在操作系统和硬件之间建立起以一个虚拟机器监视器？

您说得对，确实可以在操作系统和硬件之间建立一个虚拟机器监视器来处理分布式系统中的故障和异步中断。这种方法通常被称为虚拟化技术，它通过在操作系统和硬件之间引入一个虚拟化层，来提供更高的可靠性、可用性和安全性。

在分布式系统中，虚拟化技术可以用来隔离不同的应用程序和服务，以避免它们之间的干扰和冲突。此外，虚拟化技术还可以用来动态分配和管理计算资源，以提高系统的效率和可扩展性。最重要的是，虚拟化技术可以提供强大的故障隔离和恢复能力，以防止故障的扩散和影响。

虚拟化技术通常使用虚拟机监视器（VMM）或称为虚拟机管理器（VMM）来实现。VMM是一个软件程序，它可以模拟硬件平台并在其上运行多个虚拟机。每个虚拟机都运行在独立的虚拟化环境中，可以拥有自己的操作系统和应用程序。VMM可以对每个虚拟机进行监视和管理，以提供更高的可靠性和安全性。

总的来说，虚拟化技术是分布式系统设计中重要的一环，它可以提供更高的可靠性、可用性和安全性，也可以改善系统的效率和可扩展性。

### 也就是说vmm本质还是无法直接在物理机的硬件上运行的

是的，VMM本质上是一个软件程序，它模拟硬件平台并在其上运行多个虚拟机。在物理机上，VMM运行在操作系统之上，并使用操作系统提供的系统资源来模拟硬件平台。因此，VMM并不能直接在物理机的硬件上运行，而是依赖于操作系统和底层硬件的支持。同时，VMM还需要和物理机的硬件进行交互，以实现对虚拟机的监视、管理和控制。

### 那么对于使用虚拟化技术的分布式系统，本质依然无法处理物理机的异步中断，只能保证物理机正常的运行情况，处理分布式的虚拟机的异步中断？

您说的没错，虚拟化技术本质上只能处理虚拟机的异步中断，无法直接处理物理机的异步中断。虚拟化技术是在物理机上模拟多个虚拟机，每个虚拟机都运行在独立的虚拟化环境中，并且由VMM对其进行监视、管理和控制。当物理机发生故障或异步中断时，VMM可以控制虚拟机的状态，以避免故障的扩散和影响。

然而，如果物理机本身发生故障或异步中断，虚拟化技术并不能保证虚拟机的可用性和可靠性。在这种情况下，VMM可能无法控制虚拟机的状态，导致虚拟机的故障或不可用。因此，在使用虚拟化技术的分布式系统中，仍然需要采取其他措施来处理物理机的故障和异步中断，以保证系统的可用性和可靠性。这可能涉及到使用冗余机制、备份和恢复策略等技术手段，以确保系统可以在物理机故障的情况下继续运行。

### 这和2010年VMware FT的paper一致嘛？

是的，VMware FT的原理与我所述的类似，也是通过虚拟化技术实现系统的容错和高可用。VMware FT使用了一种称为“record/replay”的技术，它可以记录主虚拟机的执行过程，并将这些记录传输到备份虚拟机中进行回放，以保证备份虚拟机的状态和主虚拟机一致。当主虚拟机发生故障或异步中断时，备份虚拟机可以立即接管其运行，并继续提供服务，从而实现系统的容错和高可用。

需要注意的是，VMware FT是一种针对虚拟化环境的容错技术，它只能处理虚拟机的故障或异步中断，无法直接处理物理机的故障或异步中断。因此，在实际应用中，需要综合考虑物理机、虚拟机和容错技术等因素，来保证系统的可用性和可靠性。

### 使用虚拟机虽然可以保证了分布式系统中异步中断的处理，但似乎也使用了更多的资源？

是的，使用虚拟化技术确实会增加系统的资源开销。在虚拟化环境中，每个虚拟机都需要占用一定的计算、存储和网络资源，同时还需要额外的虚拟化管理软件和硬件支持。这些开销会导致虚拟化环境中的系统性能略微降低，同时也会增加系统的成本。

因此，在实际应用中，需要综合考虑虚拟化带来的好处和开销，来选择合适的方案。对于一些对性能要求较高的应用场景，可能需要尽量避免使用虚拟化技术；而对于一些对可靠性和可扩展性要求较高的场景，虚拟化技术可以带来较大的优势，但需要注意适当调整虚拟化环境的配置，以达到最优的性价比。

### FT告诉CPU在指令＃X处将控制转移到FT，这里指的是发生该指令的时候发生中断，那么指令和中断谁先执行？

在FT中，当主备节点同时执行指令时，大多数指令都是相同的，所以不需要执行任何额外的操作使它们保持相同。但是，在发生可能导致它们执行分歧的事件时，主节点必须将此信息记录在日志通道中，并在需要时将信息发送给备份节点。

在FT的处理过程中，主节点和备份节点的指令和中断的执行顺序是非常重要的。当处理器执行到FT记录的某个指令时，FT将通过向处理器发送一个中断请求来暂停正在执行的指令，处理器会响应中断请求并转移控制权到FT代码中，执行FT指定的中断处理程序，然后再恢复执行中断前正在执行的指令。这样，处理器将按照FT指定的方式执行指令和中断，并确保主节点和备份节点执行的指令和中断顺序相同，从而保证它们始终处于相同的状态。

### 那么备份节点也会在执行相同的指令的时候发生中断？

是的，备份节点也会在执行与主节点相同的指令时发生中断。备份节点通过监视主节点的执行日志来了解主节点的执行状态，并在日志中发现需要复制的输入事件时立即处理它们。在处理这些事件时，备份节点会暂停自己的执行，将事件复制到自己的内存中，并在处理完成后继续执行。因此，备份节点会在与主节点相同的指令处发生中断，以模拟主节点的执行。

### 那么备份指令会恢复执行中断前正在执行的指令嘛？

是的，备份节点会恢复执行中断前正在执行的指令。当主节点和备份节点执行的指令序列相同时，备份节点会通过日志记录接收到的中断和数据，并在需要时通过日志重放来保持状态同步。当主节点发生故障时，备份节点将根据日志中记录的事件继续运行，包括执行备份节点中断前正在执行的指令。这确保了备份节点的状态与主节点的状态完全相同。

### 如果忽略了自己的定时器，操作系统不就不会调度了？

确实，如果忽略了自己的定时器，操作系统将无法进行调度，无法执行其他进程或线程。但在FT中，备份节点仍然可以处理来自网络或磁盘的数据输入，因此可以保持和主节点相同的状态，直到恢复成为主节点或者主节点重新与备份节点进行同步。在FT中，主节点和备份节点会使用日志通道来同步状态，以确保它们始终保持一致。



### 因此client如果不等待响应只是一味发请求，无法保证每次增量都请求成功？

是的，如果client不等待主服务器的响应，而是一味地发送增量请求，那么无法保证每个增量请求都能够成功执行。这样可能会导致数据不一致的问题，因为client发送的请求可能会被主服务器和备份服务器执行多次，从而导致数据的不一致性。因此，为了确保数据的一致性，client需要使用一些机制来保证每个增量请求的成功，例如等待主服务器的响应、确认机制、超时重传机制等。

### 因此比如说采取 before primary sends output (e.g. to a client, or shared disk),  must wait for backup to acknowledge all previous log entries这样的策略是保证了分布式系统像一个系统？

是的，采取这样的策略可以保证分布式系统的数据一致性，从而让整个分布式系统更像一个系统。在这种策略下，主服务器在向客户端或共享磁盘输出数据之前，必须等待备份服务器确认所有先前的日志条目。这样可以确保主服务器和备份服务器上的数据一致，并避免出现数据不一致的情况。

采用这种策略的一个实现方式是使用两个阶段提交协议（Two-Phase Commit Protocol），它在主服务器和备份服务器之间协调提交或回滚一个事务。在第一个阶段，主服务器向备份服务器发送一个提交请求，并等待所有备份服务器响应。在第二个阶段，如果所有备份服务器都确认提交，则主服务器提交事务并向客户端或共享磁盘输出数据，否则主服务器回滚事务。通过这种方式，主服务器和备份服务器可以协调提交或回滚一个事务，并确保数据一致性。

### 在VM-FT中，VMM将中断发送给虚拟机和写中断到log里并发送到channel是同时发生的嘛

在VM-FT中，VMM将中断发送给虚拟机和写中断到log里并发送到channel是同时发生的。当VMM接收到来自虚拟机的中断请求时，它将中断信息记录到一个本地日志文件中，并将中断信息通过共享存储通道发送到备份节点。这样，在主节点发生故障时，备份节点就能够获取到主节点的中断日志并从上一个已知状态恢复虚拟机的运行。因此，在VM-FT中，VMM将中断发送给虚拟机和写中断到log里并发送到channel是同时发生的，以确保主备节点之间的数据同步和一致性。

### 如何使替代备份节点达到同步并恢复复制？

在Replicated State Machine中，如果备份节点失效，需要替换一个新的备份节点并使其达到与旧备份节点相同的状态，以确保系统的高可用性和容错性。这可以通过以下步骤实现：

1. 将新的备份节点连接到主节点和其他备份节点，使其加入系统。
2. 将最新的快照从主节点复制到新的备份节点。这可以通过在主节点上创建一个快照，然后将其复制到新的备份节点来完成。
3. 将从旧备份节点复制到主节点的所有日志条目复制到新的备份节点。这可以通过在主节点上查找最后一个与旧备份节点相同的日志条目并将其后面的所有日志条目复制到新的备份节点来完成。
4. 将新的备份节点设置为准备就绪状态，并将其添加到备份节点列表中，以确保以后的所有写操作都复制到新的备份节点。

这些步骤可以确保新的备份节点与旧的备份节点具有相同的状态，并且可以恢复复制操作以确保系统的高可用性和容错性。
