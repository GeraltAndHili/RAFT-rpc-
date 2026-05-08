# RAFT-rpc

这是一个基于 Raft 共识算法的分布式 KV 存储项目，主要用于理解 Raft 在实际工程中的落地方式。

## 环境依赖

建议在 Linux/Ubuntu 环境下运行，项目使用 CMake 构建，依赖 C++20、Muduo、Protobuf 和 Boost Serialization。

Ubuntu 可以先安装这些包：

```bash
sudo apt update
sudo apt install -y build-essential cmake protobuf-compiler libprotobuf-dev libboost-serialization-dev libmuduo-dev
```

如果系统源里没有 `libmuduo-dev`，需要先手动安装 Muduo 库，并保证能链接到 `muduo_net` 和 `muduo_base`。

## 编译运行

在项目根目录执行：

```bash
cmake -S . -B build
cmake --build build -j
```

编译后可执行文件会生成到 `bin/` 目录。

启动一个 3 节点 Raft KV 集群：

```bash
cd bin
./raftCoreRun -n 3 -f test.conf
```

再打开另一个终端，进入同一个 `bin/` 目录运行客户端：

```bash
cd bin
./callerMain
```

`raftCoreRun` 会把各个节点的地址和端口写入 `test.conf`，`callerMain` 会读取这个配置文件并向集群发起 `Put/Get` 请求。

常见问题：

- 如果提示找不到 Muduo/Protobuf/Boost，先确认对应开发包是否安装。
- 如果端口被占用，重新运行 `raftCoreRun` 即可，它会随机选择一组端口写入配置。
- 如果需要重新开始测试，可以先停止旧的 `raftCoreRun` 进程，再重新启动服务端和客户端。

## 项目理解

项目核心思路是：多个节点通过 Raft 完成 leader 选举、日志复制和状态一致性维护；客户端请求先进入 leader，再通过日志同步到多数节点，最后应用到 KV 状态机中。

补充阅读：

- [Raft KV 项目系统组织架构与数据流图](docs/raft_系统组织架构与数据流图.md)：按系统总图、模块职责、读写请求、Snapshot 和 RPC 链路梳理整体数据流。
- [心得总结](心得总结/summary.ipynb)：记录学习过程中的理解、问题和阶段性总结。
- [Raft 总结图](心得总结/raft总结4-22.jpg)：配合心得总结查看的手写/图示笔记。

我对这个项目的要点理解：

- Raft 的重点不是单个函数，而是节点状态、任期、日志和提交索引之间的协作。
- leader 选举解决“谁来接收请求”的问题，日志复制解决“所有节点按同样顺序执行命令”的问题。
- KV 存储本身只是状态机，真正保证一致性的是 Raft 层。
- RPC 是节点之间通信的基础，Raft 的投票、心跳和日志同步都依赖它。
- 持久化很关键，节点宕机重启后必须恢复任期、投票记录和日志，否则容易破坏一致性。

整体来说，这个项目适合用来练习 C++、RPC、协程/线程调度，以及分布式一致性算法的工程实现。
