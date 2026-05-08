# Raft KV 项目系统组织架构与数据流图

这份文档适合用支持 Mermaid 的 Markdown 查看器打开。

## 1. 系统总图

```mermaid
flowchart LR
    A[Client / Clerk\nsrc/raftClerk/clerk.cpp] --> B[KvServer RPC服务\nGet / PutAppend]
    B --> C[KvServer前台RPC线程\n等待waitApplyCh]
    C --> D[Raft Core\nsrc/raftCore/raft.cpp]
    D --> E[Raft日志\ncurrentTerm/votedFor/logs\ncommitIndex/lastApplied]
    D --> F[Raft节点间RPC\nAppendEntries / RequestVote / InstallSnapshot]
    F --> G[Other Raft Nodes]

    D --> H[applyChan\nRaft -> KvServer]
    H --> I[KvServer后台apply线程\nReadRaftApplyCommandLoop]
    I --> J[KV状态机\nSkipList]
    I --> K[lastRequestId\n去重表]

    J --> L[MakeSnapShot]
    K --> L
    L --> M[Raft::Snapshot]
    M --> N[Persister\nraft state + snapshot]

    N --> O[leaderSendSnapShot]
    O --> P[Follower InstallSnapshot]
    P --> H

    Q[RPC框架\nmprpcchannel/rpcprovider] --> B
    Q --> F
```

## 2. 模块职责图

```mermaid
flowchart TB
    subgraph ClientSide[客户端侧]
        C1[Clerk\n维护leader猜测\n失败重试]
        C2[raftServerRpcUtil\nKv RPC stub]
        C1 --> C2
    end

    subgraph ServerSide[单个KvServer节点]
        S1[RpcProvider\n收包/解包/分发]
        S2[KvServer\nGet/PutAppend]
        S3[waitApplyCh\n前台等待某个raftIndex结果]
        S4[Raft\n一致性核心]
        S5[applyChan]
        S6[ReadRaftApplyCommandLoop]
        S7[SkipList状态机]
        S8[lastRequestId]
        S9[Persister]

        S1 --> S2
        S2 --> S3
        S2 --> S4
        S4 --> S5
        S5 --> S6
        S6 --> S7
        S6 --> S8
        S6 --> S3
        S6 --> S9
        S4 --> S9
    end

    subgraph Cluster[其他节点]
        R1[Raft Peer]
        R2[Raft Peer]
        R3[Raft Peer]
    end

    C2 --> S1
    S4 --> R1
    S4 --> R2
    S4 --> R3
```

## 3. 写请求主链路

```mermaid
sequenceDiagram
    participant Client
    participant Clerk
    participant KvRPC as KvServer::PutAppend
    participant Raft
    participant Peers
    participant ApplyLoop as KvServer ApplyLoop
    participant KV as SkipList

    Client->>Clerk: Put/Append
    Clerk->>KvRPC: RPC请求
    KvRPC->>Raft: Start(op)
    Raft-->>KvRPC: raftIndex, isLeader
    KvRPC->>KvRPC: 创建/等待 waitApplyCh[raftIndex]

    Raft->>Peers: AppendEntries复制日志
    Peers-->>Raft: 多数派成功
    Raft->>Raft: commitIndex推进
    Raft->>ApplyLoop: applyChan.Push(ApplyMsg)

    ApplyLoop->>KV: 执行Put/Append
    ApplyLoop->>ApplyLoop: 更新lastRequestId
    ApplyLoop->>KvRPC: waitApplyCh[raftIndex].Push(op)

    KvRPC-->>Clerk: OK
    Clerk-->>Client: 成功
```

## 4. 读请求链路

```mermaid
sequenceDiagram
    participant Client
    participant Clerk
    participant KvRPC as KvServer::Get
    participant Raft
    participant Peers
    participant ApplyLoop as KvServer ApplyLoop
    participant KV as SkipList

    Client->>Clerk: Get(key)
    Clerk->>KvRPC: RPC请求
    KvRPC->>Raft: Start(GetOp)
    Raft-->>KvRPC: raftIndex, isLeader
    KvRPC->>KvRPC: 等待 waitApplyCh[raftIndex]

    Raft->>Peers: AppendEntries复制GetOp
    Peers-->>Raft: 多数派成功
    Raft->>Raft: commitIndex推进
    Raft->>ApplyLoop: applyChan.Push(ApplyMsg)

    ApplyLoop->>KV: 读取当前值
    ApplyLoop->>KvRPC: waitApplyCh[raftIndex].Push(op/result)

    KvRPC-->>Clerk: value / ErrNoKey / ErrWrongLeader
    Clerk-->>Client: 返回读结果
```

## 5. Snapshot 链路

```mermaid
sequenceDiagram
    participant ApplyLoop as KvServer ApplyLoop
    participant KV as KV状态机
    participant Raft
    participant Persister
    participant Follower

    ApplyLoop->>KV: 已提交命令执行到状态机
    ApplyLoop->>ApplyLoop: 判断RaftStateSize是否过大
    ApplyLoop->>KV: MakeSnapShot()
    KV-->>ApplyLoop: snapshot bytes
    ApplyLoop->>Raft: Snapshot(raftIndex, snapshot)
    Raft->>Raft: 截断已提交日志前缀
    Raft->>Persister: Save(raftState, snapshot)

    Raft->>Follower: InstallSnapshot RPC
    Follower->>Follower: 截断日志/推进状态
    Follower->>Follower: Persister.Save(...)
    Follower->>Follower: applyChan.Push(snapshot ApplyMsg)
```

## 6. RPC 框架链路

```mermaid
sequenceDiagram
    participant Stub
    participant Channel as MprpcChannel
    participant Net as TCP Socket
    participant Provider as RpcProvider
    participant Service as 业务Service

    Stub->>Channel: CallMethod(method, request)
    Channel->>Channel: request序列化为args_str
    Channel->>Channel: 组装RpcHeader(service_name, method_name, args_size)
    Channel->>Net: 发送 [header_size][rpc_header_str][args_str]

    Net->>Provider: 收到字节流
    Provider->>Provider: 解析header_size
    Provider->>Provider: 解析rpc_header_str
    Provider->>Provider: 得到service_name/method_name/args_size
    Provider->>Provider: 读取args_str
    Provider->>Service: service->CallMethod(...)
    Service-->>Provider: response
    Provider-->>Net: 序列化并回包
    Net-->>Channel: response bytes
    Channel-->>Stub: 反序列化response
```

## 7. 最短总纲

项目可以压缩成 6 层：

1. `raftClerk`
   作用：客户端重试、记 leader、发 RPC
2. `rpc`
   作用：底层 RPC 打包、收包、分发，不关心 Raft 语义
3. `KvServer`
   作用：对外提供 KV 服务；把请求送入 Raft；接收 apply 后真正操作状态机；维护幂等
4. `Raft`
   作用：选举、日志复制、提交、apply、snapshot 元信息管理
5. `Persister`
   作用：保存 `raft state + snapshot`
6. `skipList`
   作用：KV 状态机容器，本身不是一致性逻辑重点

## 8. 四条必须背下来的数据流

1. 写请求：`Clerk -> KvServer::PutAppend -> Raft::Start -> AppendEntries -> commit -> applyChan -> KV -> waitApplyCh -> reply`
2. 读请求：`Clerk -> KvServer::Get -> Raft::Start(GetOp) -> commit -> applyChan -> KV读 -> reply`
3. 快照：`KV状态机 -> MakeSnapShot -> Raft::Snapshot -> Persister -> InstallSnapshot -> follower恢复`
4. RPC：`stub -> CallMethod -> RpcHeader封包 -> socket -> RpcProvider解包 -> CallMethod分发 -> 回包`
