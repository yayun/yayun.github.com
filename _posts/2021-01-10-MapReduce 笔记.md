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
3. 被分配 Map 任务的 worker 会从输入分片中读取内容，解析输入数据，将输入数据传给用户自定义的 Map 函数。Map 函数生成的中间结果会缓存在内存中。
4. Map 函数放在内存中的中间结果会周期性的写入本地磁盘。通过 partitioning 函数分成 R 个部分。这些数据在磁盘上的地址会回传给 Master，master 将这些地址转发给 reduce worker。
5. 当 Master 开始通知 reduce worker 这些文件的地址之后，reduce worker 使用 rpc 来从 map worker 的本地磁盘读取这些中间数据。当 reduce worker 把这些数据都读取完成之后，会**先把这些中间结果的 key 排序，让相同的 key 的数据组合在一起**。如果中间结果数据量太大内存中装不下，还需要依赖外部排序来实现。
6. 



几个月的功夫，mrmaster 就被改成了 mrcoordinator .... 太政治正确了。

The issues of how to parallelize the computation, distribute the data, and handle
failures conspire to obscure the original simple computation with large amounts of complex code to deal with
these issues.

we were trying to perform but hides the messy details of parallelization, fault-tolerance, data distribution
and load balancing in a library. 

When a reduce worker has read all intermediate data, it sorts it by the intermediate keys
so that all occurrences of the same key are grouped
together

After successful completion, the output of the mapreduce execution is available in the R output files (one per
reduce task, with file names as specified by the user).
Typically, users do not need to combine these R output
files into one file – they often pass these files as input to
another MapReduce call, or use them from another distributed application that is able to deal with input that is
partitioned into multiple files


The master is the conduit through which the location
of intermediate file regions is propagated from map tasks
to reduce tasks.

### Fault Tolerance

#### Worker Failure

The master pings every worker periodically. If no response is received from a worker in a certain amount of
time, the master marks the worker as failed. Any map
tasks completed by the worker are reset back to their initial idle state, and therefore become eligible for scheduling on other workers. Similarly, any map task or reduce
task in progress on a failed worker is also reset to idle
and becomes eligible for rescheduling.

#### Master Failure

It is easy to make the master write periodic checkpoints
of the master data structures described above. If the master task dies, a new copy can be started from the last
checkpointed state

#### Semantics in the presence of Failures

Each in-progress task
writes its output to private temporary files. A reduce task
produces one such file, and a map task produces R such
files (one per reduce task).

When running large
MapReduce operations on a significant fraction of the
workers in a cluster, most input data is read locally and
consumes no network bandwidth.

### bakup tasks

 When a MapReduce operation is close
to completion, the master schedules backup executions
of the remaining in-progress tasks. The task is marked
as completed whenever either the primary or the backup
execution completes.


### 参考

1. [https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)