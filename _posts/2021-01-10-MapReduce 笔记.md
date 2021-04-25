---
layout: post
title: "MapReduce 笔记"
---

### 简介

很长一段时间内，Google 的工程师编写了许多特定目的的计算程序，来处理大规模的原始数据：从网页上爬取的数据，web 请求日志.., 来生成他们想要的结果。大部分计算从概念上来说都很简单。但是这些计算的输入数据都非常大，为了在一定的时间内完成计算，需要在数百台甚至上千台的机器上并行计算。Google 工程师设计了 MapReduce ，我们只需要关心上层计算，而不需要关心怎样并行计算，容错，数据分配负载等涉及分布式计算的问题。

MapReduce 是由 Google 提出的程序模型，用于处理和生成大规模的数据集，可以并行计算。

用户只需要编写两个函数: map function 来生成多组键值对的中间结果; reduce function 来合并这些中间结果生成最终结果。

###  Programming Model


如果想使用 MapReduce library，我们只需要编写两个函数 Map 和 Reduce。

Map 函数，输入 key/value pairs，输出中间结果集 key/value pairs。MapReduce将所有的中间结果排序之后传给 reduce 函数。Reduce 函数接收 key/value pairs，处理数据生成结果。

### Implementation

Google 的 MapReduce 运行环境：

1. X86 双核处理器，Linux 环境，2G内存
2. 100M/1000M 带宽网卡
3. 一个集群包含上百台或者上千台机器，其中有机器宕机是常有的事
4. 机器上使用的是便宜的 IDE 硬盘，分布式文件系统（GFS）来管理这些磁盘上的存储空间。
5. 用户提交 Job 给调度系统。每个 job 包含一系列的 task，调度系统将这些 task 分配给集群中多台可用的机器。

#### Execution Overview

<img src="/images/MapReduceOverview.png">

1. MapReduce Library 将输入的文件划分成 M 个 16M到64M（大小可以通过参数调节） 大小的数据分片，然后在一组集群中启动多个程序
2. 这些程序中只有一个程序比较特殊：Master，其他程序都是 worker。这些 worker 会被 master 分配 map/reduce 任务。M 个 map 任务，R 个 reduce 任务。master 会将任务分派给空闲的 worker。
3. 被分配 Map 任务的 worker 会从输入分片中读取内容，解析输入数据，将输入数据传给用户自定义的 Map 函数。Map 函数生成的中间结果会缓存在内存中。一个 map task 会产生 N 个中间文件（hash(key) mod R），相同 key 的数据分配给同一个 reduce task。
4. Map 函数放在内存中的中间结果会周期性的写入本地磁盘。通过 partitioning 函数分成 R 个部分。数据在磁盘上的地址会回传给 Master，master 将地址转发给 reduce worker。
5. 当 Master 通知 reduce worker 这些文件的地址之后，reduce worker 使用 rpc 来从 map worker 的本地磁盘读取这些中间数据。当 reduce worker 把这些数据都读取完成之后，会**先把这些中间结果的 key 排序，让相同 key 的数据组合在一起**。如果中间结果数据量太大内存中装不下，还需要依赖外部排序来实现。
6. Reduce worker 迭代排序的中间数据，将它遇到的 key 和 与 key 相匹配的 values 传给 reduce 函数。reduce 函数的输出会 append 到结果文件中。每个 reduce partition 都有一个结果文件。
7. 当所有的 map task 和 reduce task 都执行完成之后，master 会唤醒用户程序，将计算结果返回给用户。

成功执行之后，MapReduce 的输出通常会放在 R 个文件中（每个 reduce task 产生一个文件），用户不需要合并这些文件，因为通常这些文件会作为下一个 MapReduce 的输入或者传给另外的可以处理这些输入的分布式程序。

#### Master Data Structures

Master 数据结构中存储的内容：
task 的状态：idle，in-progress，completed；非空闲的 task 还需要存储跑当前 task 的机器 id。
对于每个结束的 map task，Master 存储其输出的 R 个中间文件的地址和大小。

### Fault Tolerance

#### Worker Failure

Master 会周期性的 ping 每个 worker，如果在一定的时间内没有收到响应，master 会标记这个 worker 宕机。同时在这台 worker 执行的 map/reduce task 也会被置为空闲状态，重新等待 master 分配执行。
每个完成任务的 worker 会被 master 标记为空闲状态。

#### Master Failure

可以通过定期的 checkpoint 来保存 master 的状态。master 挂掉之后重启可以回到 checkpoint 的点

#### Semantics in the presence of Failures

原子提交：每个正在执行的 task 都会将他的输出写入到本地临时文件中。reduce task 输出一个文件，map task 输出 R 个文件（每个文件对应一个 reduce task）。当一个 map task 执行完之后，worker 发消息给 master，将当前 task 输出的 R 个临时文件地址传给 master。如果 master 收到已经完成的 map 任务的消息，会忽略这条消息。reduce task 执行完，reduce worker 将当前 task 输出的临时文件 rename 到最终的输出文件上，如果相同的 reduce task 在多台不同的机器上执行，多台机器都做 rename 操作，atomic rename，最终的输出文件只是其中一个 reduce task 输出的。

#### Locality
因为 MapReduce 的输入数据是放在 GFS 上面的，所以采用了以下策略来减少网络通信：

1. GFS 将每个文件分成 64MB 大小的块，然后将每块数据的副本（一般 copy 3份数据）放到多台机器上
2. master 会记住这些文件的存储信息，master 会尝试给本地包含输入文件的 worker 分配 map task，如果失败则会尝试给本地有其他副本的 worker 分配。所以，map task 读取文件都是从本地读取的。


#### Task Granularity

将 map 任务分成 M 份，将 reduce 任务分成 R 份。M 和 R 需要远远大于 worker 机器数量。让一个 worker 做多个不同的任务可以提高负载均衡，也有助于崩溃恢复：map 任务失败后可以分配给其他 worker 执行。

具体实现中 M 和 R 的个数是有限制的：master 需要 O(M+R) 的空间来做调度决策，需要 O(M*R) 空间来存储状态信息。

通常 R 由 用户自定义。M 的个数需要让每个 task 的输入数据在 16M 到 64M 之间，R 的个数是 worker 机器数的小倍数。eg：worker 机器数量是 2000 的时候，R 设置为 5000， M 设置为 200000。

#### Bakup tasks

straggler：执行 task 过程中，可能由少数最后几个 map/reduce task 执行特别慢，导致 mapreduce 任务无法完成。master 会把多个 task 放到一个 machine 上执行，这些 task 共享这台机器的 CPU，内存，本地磁盘，网络带宽。
解决办法：当 mapreduce 任务快执行完时，master 给正在执行的 task 启动备份任务，两者只要有一个执行完成那么 master 将此任务标记为执行完成。

### Refinements

#### Partitioning Function

默认的 partitioning function： hash(key) mod R

有些特殊情况，比方说 key 是 Url，我们想要让 host 相同的 key 输出到一个文件中，mapreduce library 提供了一个 partitioning function：hash(Hostname(urlkey)) mod R。

#### Ordering Guarantees

使用给定的 partition 函数，保证中间结果 key 输出是按照顺序输出的

#### Combiner Function

通常情况下，map task 产生的中间结果的重复度很高，reduce task 从多台机器上获取这些中间结果要耗费大量的网络带宽。eg： wordCount 任务中，产生大量像 <the, 1> 这样的数据，reduce task 从多台机器上拿到这些数据最后聚合只生成一条数据 <the, count>。 用户可以自定义 Combiner 函数，在 reduce task 获取数据之前，Combiner 先做一个数据聚合。

Combiner 函数的输出到中间文件，之后 reduce 函数再读取。

####  Input and Output Types

input output 支持多种格式，用户也可以自定义格式，只需要实现相应的接口

#### Side-effects 

mapreduce 允许用户生成辅助输出文件

#### Skipping Bad Records

有时用户程序有可能有 bug，导致程序在处理某些记录时崩溃，导致 mapreduce 任务无法完成。通常我们需要修复这些 bug，但是有些时候 bug 来自三方库，我们无法修改其源码。如果我们对大数据量做数据分析，这些记录我们也可以忽略。mapreduce 提供了一些来跳过这些记录的方法。

每个 worker 进程安装一个信号处理程序来捕捉 segementation violation 和 bus errors 信号。在启动 map/reduce 任务之前，mapreduce library 在全局变量里面存储一个序列号。一旦收到信号，则将该序列号发送给 master。如果 master 发现某条记录失败的次数大于 1，则会在下次执行时跳过该记录。

#### Local Execution

mapreduce 任务在分布式环境下执行，调试起来很困难。mapreduce 提供了在本地串行执行的方式，方便用户调试

#### Status Information

master 内部跑了一个 HTTP 服务，可以将当前 master 执行状态打印出来。包含多少 task 执行完成，多少 task 正在执行，输入多少字节，输出多少字节，中间数据多少字节，处理速度等。

#### Counters

mapreduce library 提供了一个计数功能来记录各种事件发生的次数。eg: 用户有可能像知道总共处理的单词总数等信息。

使用这个功能：定义一个 counter 对象，在 map/reduce task 中递增它

eg: 用户想知道总共处理的大写单词数：

	Counter* uppercase;
	uppercase = GetCounter("uppercase");
	map(String name, String contents):
		for each word w in contents:
			if (IsCapitalized(w)):
				uppercase->Increment();
		EmitIntermediate(w, "1");
		
worker 中的 count 数据会周期性的发给 master （piggybacked on the ping response， master 会周期性的 ping worker 来探活)。最终 master 这些数据（执行成功的 task 发过来的数据）做聚合返回给用户。重复执行的 task count 只会记一次，保证结果的准确性。

mapreduce library 会自动 count 一些数据：输入的 k/v pairs 数量，输出 k/v pairs 数量。

counter 机制在某些情况下很有用：用户通过检查数据对和输出对的数量来判断结果的准确性。

### 参考

1. [https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)