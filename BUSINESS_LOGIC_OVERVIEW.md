# Cyclone DDS 业务逻辑完整梳理

## 1. 系统架构概述

Cyclone DDS 是一个符合 OMG DDS (Data Distribution Service) 规范的高性能、开源实现。它实现了一个**分布式数据空间（Shared Data Space）**架构，提供了发布-订阅（Publish-Subscribe）通信机制。

### 1.1 核心模块架构

```
┌─────────────────────────────────────────┐
│          DDS API Layer (ddsc)           │  ← 用户API层
│  - DomainParticipant, Publisher         │
│  - Subscriber, DataWriter, DataReader   │
│  - Topic, QoS, Listener, Waitset        │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│       DDSI-RTPS Layer (ddsi)            │  ← RTPS协议层
│  - 网络发现 (SPDP/SEDP)                 │
│  - 可靠性传输 (Heartbeat/AckNack)       │
│  - 端点匹配 (Reader/Writer Matching)    │
│  - 数据序列化与传输                     │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│    Runtime Layer (ddsrt)                │  ← 运行时抽象层
│  - 线程管理、同步原语                   │
│  - 网络套接字、时间管理                 │
│  - 内存管理、日志系统                   │
└─────────────────────────────────────────┘
```

### 1.2 关键模块说明

1. **ddsc (DDS C API)**: 实现用户可见的 DDS API，提供现代、用户友好的 DDS 规范实现
2. **ddsi (DDS Interoperability)**: 实现 RTPS-DDSI 规范，处理网络协议和互操作性
3. **ddsrt (DDS Runtime)**: 提供跨平台运行时抽象，使上层模块不依赖于特定操作系统
4. **cdr**: CDR 序列化/反序列化支持
5. **security**: DDS Security 插件实现（认证、加密、访问控制）
6. **idl**: IDL 编译器
7. **psmx_iox**: 共享内存传输插件（通过 Eclipse Iceoryx）

---

## 2. 系统初始化流程

### 2.1 库初始化 (`dds_init`)

**文件**: `src/core/ddsc/src/dds_init.c`

**流程**:
1. **调用 `ddsrt_init()`**: 初始化运行时环境
   - 初始化线程系统
   - 初始化同步原语（互斥锁、条件变量）
   - 初始化日志系统

2. **状态管理**: 使用原子变量管理初始化状态
   - `CDDS_STATE_ZERO`: 未初始化
   - `CDDS_STATE_STARTING`: 正在初始化
   - `CDDS_STATE_READY`: 已就绪
   - `CDDS_STATE_STOPPING`: 正在停止

3. **全局资源初始化**:
   - 创建全局互斥锁和条件变量
   - 初始化实例ID生成器 (`ddsi_iid_init`)
   - 初始化线程状态管理 (`ddsi_thread_states_init`)
   - 初始化句柄服务器 (`dds_handle_server_init`)

4. **创建全局伪实体**: 创建 `DDS_CYCLONEDDS_HANDLE` 作为全局实体引用

### 2.2 Domain 初始化 (`ddsi_init`)

**文件**: `src/core/ddsi/src/ddsi_init.c`

**流程**:
1. **配置加载**: 从 XML 配置或环境变量加载配置
2. **传输层初始化**: 
   - 初始化 UDP/TCP 传输工厂
   - 创建发现连接（`disc_conn_uc`, `disc_conn_mc`）
   - 创建数据连接（`data_conn_uc`, `data_conn_mc`）
3. **发现协议启动**:
   - 启动 SPDP (Simple Participant Discovery Protocol)
   - 启动 SEDP (Simple Endpoint Discovery Protocol)
4. **线程启动**: 启动接收线程、发送线程等

---

## 3. 实体生命周期管理

### 3.1 实体层次结构

```
DomainParticipant (域参与者)
├── Publisher (发布者)
│   └── DataWriter (数据写入者)
│       └── Topic (主题)
├── Subscriber (订阅者)
│   └── DataReader (数据读取者)
│       └── Topic (主题)
└── Topic (主题)
```

### 3.2 实体创建流程

#### 3.2.1 DomainParticipant 创建

**文件**: `src/core/ddsc/src/dds_participant.c`

**步骤**:
1. 验证域ID和QoS配置
2. 创建 DDSC 层的 participant 对象 (`dds_participant`)
3. 创建 DDSI 层的 participant 对象 (`ddsi_participant`)
4. 启动 SPDP 协议，开始参与域发现
5. 创建内置主题（Built-in Topics）的订阅者

#### 3.2.2 Topic 创建

**文件**: `src/core/ddsc/src/dds_topic.c`

**步骤**:
1. 在 Participant 的 ktopic 树中查找或创建 `dds_ktopic`
   - `dds_ktopic`: 存储 topic 名称到 QoS 和类型名的映射
   - 相同名称和QoS的 Topic 共享同一个 `dds_ktopic`
2. 创建 DDSI 层的 topic 实体 (`ddsi_topic_definition`)
3. 关联类型信息 (`ddsi_sertype`)
4. 如果启用了主题发现，在网络上通告此 Topic

#### 3.2.3 DataWriter 创建

**文件**: `src/core/ddsc/src/dds_writer.c`

**步骤**:
1. 验证 Topic 和 Publisher
2. 创建 DDSC 层的 writer 对象 (`dds_writer`)
3. 创建 DDSI 层的 writer 对象 (`ddsi_writer`)
   - 初始化 Writer History Cache (WHC)
   - 初始化心跳控制 (`ddsi_hbcontrol`)
   - 初始化发送缓冲区 (`ddsi_xpack`)
4. 注册到 DDSI participant
5. 通过 SEDP 协议发现匹配的 Reader

#### 3.2.4 DataReader 创建

**文件**: `src/core/ddsc/src/dds_reader.c`

**步骤**:
1. 验证 Topic 和 Subscriber
2. 创建 DDSC 层的 reader 对象 (`dds_reader`)
3. 创建 DDSI 层的 reader 对象 (`ddsi_reader`)
   - 初始化 Reader History Cache (RHC)
   - 初始化重排序缓冲区 (`ddsi_reorder`)
   - 初始化分片重组缓冲区 (`ddsi_defrag`)
4. 注册到 DDSI participant
5. 通过 SEDP 协议发现匹配的 Writer

### 3.3 实体删除流程

1. **删除子实体**: 先删除所有子实体（Writer/Reader 必须在 Topic 删除前删除）
2. **资源清理**:
   - 释放历史缓存 (WHC/RHC)
   - 停止相关的网络线程和定时器
   - 通过 SEDP 发送删除通知
3. **引用计数**: 使用引用计数管理实体生命周期
4. **延迟删除**: 可靠的 Writer 可能在删除后"徘徊"（lingering）一段时间，等待所有读者的确认

---

## 4. 发现机制 (Discovery)

### 4.1 SPDP (Simple Participant Discovery Protocol)

**目的**: 发现域中的其他参与者

**文件**: `src/core/ddsi/src/ddsi_discovery_spdp.c`

**流程**:
1. **周期性发送 SPDP 消息**:
   - 默认发送到多播地址 `239.255.0.1`（IPv4）或 `ff02::ffff:239.255.0.1`（IPv6）
   - 端口号根据域ID计算
   - 包含参与者的 GUID、地址信息、QoS 等

2. **接收 SPDP 消息**:
   - 为每个发现的远程参与者创建代理参与者 (`ddsi_proxy_participant`)
   - 建立到远程参与者的连接

3. **租约管理 (Lease Duration)**:
   - 每个参与者声明自己的租约期限
   - 如果超过租约期限未收到 SPDP 消息，则认为参与者已离线
   - 自动清理代理参与者和相关资源

### 4.2 SEDP (Simple Endpoint Discovery Protocol)

**目的**: 在已发现的参与者之间交换端点（Reader/Writer）信息

**文件**: `src/core/ddsi/src/ddsi_discovery_endpoint.c`

**流程**:
1. **端点通告**:
   - Writer 通过 SEDP Writer 通告自己的存在
   - Reader 通过 SEDP Reader 通告自己的存在
   - 包含 Topic 名称、类型名、QoS 等信息

2. **代理端点创建**:
   - 为每个远程 Writer 创建代理 Writer (`ddsi_proxy_writer`)
   - 为每个远程 Reader 创建代理 Reader (`ddsi_proxy_reader`)

3. **端点匹配**:
   - 检查 Topic 名称是否匹配
   - 检查类型名称或类型兼容性
   - 检查 QoS 兼容性（见 5.3 节）

4. **连接建立**:
   - 如果匹配成功，在 Writer 和 Reader 之间建立连接
   - 配置可靠传输的心跳和确认机制（如果是可靠的）

### 4.3 内置端点 (Built-in Endpoints)

DDSI 规范定义了内置端点，用于发现协议：
- **SPDP Writer/Reader**: 用于 SPDP 协议
- **SEDP Writer/Reader**: 用于 SEDP 协议
- **TypeLookup Writer/Reader**: 用于类型发现（如果启用）
- **Security Writer/Reader**: 用于安全握手（如果启用安全）

### 4.4 共享发现信息

**文件**: `docs/manual/config/discovery-behavior.rst`

Cyclone DDS 的一个重要优化是：同一进程内的多个 DomainParticipant **共享发现信息**。
- 新创建的 Participant 可以立即看到已存在的端点
- 减少了网络流量和发现时间
- 可以通过配置模拟每个 Participant 独立发现（用于与其他实现互操作）

---

## 5. 发布订阅数据流

### 5.1 写入数据流程 (`dds_write`)

**文件**: `src/core/ddsc/src/dds_write.c`

**流程**:
```
应用程序调用 dds_write()
    ↓
1. 验证 Writer 状态和参数
    ↓
2. 评估 Topic 过滤器（如果有）
    ↓
3. 序列化数据
   - 如果使用 PSMX (共享内存): 可能需要特殊处理
   - 否则: 使用 CDR 序列化 (ddsi_serdata_from_sample)
    ↓
4. 添加到 Writer History Cache (WHC)
    ↓
5. 传输数据 (transmit_sample)
   a. 如果使用 PSMX: 通过 PSMX 传输
   b. 如果使用网络传输:
      - 分配 xmsg (传输消息)
      - 大样本可能需要多个 DATAFRAG 子消息
      - 添加到发送队列 (xpack)
      - 发送 UDP 数据包 (sendmsg)
    ↓
6. 如果是可靠传输: 等待 Reader 确认
```

**关键数据结构**:
- **`ddsi_serdata`**: 序列化的数据表示
- **`ddsi_xmsg`**: 要发送的 RTPS 消息
- **`ddsi_xpack`**: 发送包，包含多个 xmsg
- **`ddsi_whc`**: Writer History Cache，存储已发送但未确认的样本

### 5.2 读取数据流程 (`dds_read` / `dds_take`)

**文件**: `src/core/ddsc/src/dds_read.c`

**流程**:
```
接收线程接收 UDP 数据包
    ↓
1. 接收数据 (recvmsg)
   - 使用接收缓冲区 (ddsi_rbuf)
   - 创建接收消息 (ddsi_rmsg)
    ↓
2. 解析 RTPS 消息 (handle_submsg_sequence)
   - 解析各种子消息: DATA, DATAFRAG, HEARTBEAT, ACKNACK, GAP
    ↓
3. 处理数据子消息
   a. 如果是分片 (DATAFRAG):
      - 进行分片重组 (ddsi_defrag_rsample)
   b. 如果是完整样本 (DATA):
      - 直接使用
    ↓
4. 可靠性处理（如果是可靠传输）
   a. 检查序列号连续性 (ddsi_reorder_rsample)
   b. 检测丢失的样本（通过 GAP 或 HEARTBEAT）
   c. 请求重传（发送 ACKNACK）
    ↓
5. 数据反序列化
   - 从 rdata 中提取样本数据
   - 反序列化为应用程序类型
    ↓
6. 添加到 Reader History Cache (RHC)
    ↓
7. 通知应用程序（通过 Listener 或 Waitset）
    ↓
8. 应用程序调用 dds_read() / dds_take() 获取数据
```

**关键数据结构**:
- **`ddsi_rbuf`**: 接收缓冲区池
- **`ddsi_rmsg`**: 接收到的 RTPS 消息
- **`ddsi_rdata`**: 表示一个数据子消息
- **`ddsi_defrag`**: 分片重组表
- **`ddsi_reorder`**: 序列号重排序表
- **`ddsi_rhc`**: Reader History Cache

### 5.3 端点匹配 (Endpoint Matching)

**文件**: `src/core/ddsi/src/ddsi_qosmatch.c`, `src/core/ddsi/src/ddsi_endpoint_match.c`

**匹配条件**:
1. **Topic 名称**: 必须完全相同
2. **类型兼容性**:
   - 基本模式: 类型名称必须相同
   - 类型发现模式: 检查类型兼容性（通过类型ID和类型描述符）
3. **QoS 兼容性检查**:
   - **Reliability**: Reader 的可靠性要求不能高于 Writer 提供的
   - **Durability**: Reader 的持久性要求不能高于 Writer 提供的
   - **Partition**: 分区必须重叠（如果使用了分区）
   - **Data Representation**: 数据表示格式必须兼容
   - **Type Consistency**: 类型一致性策略检查
   - 其他 QoS 策略的兼容性检查

**匹配流程**:
```
发现新的远程端点
    ↓
1. 创建代理端点 (proxy_writer 或 proxy_reader)
    ↓
2. 遍历所有本地匹配端点
    ↓
3. 对每对 (本地Reader, 远程Writer) 或 (本地Writer, 远程Reader):
   a. 检查 Topic 名称
   b. 检查类型兼容性
   c. 调用 ddsi_qos_match_p() 检查 QoS 兼容性
    ↓
4. 如果匹配成功:
   a. 建立连接
   b. 配置可靠传输参数（如果需要）
   c. 通知应用程序（通过 Listener）
   d. 开始数据传输
```

---

## 6. 可靠性传输机制

### 6.1 Best-Effort 传输

**特点**:
- 简单的 UDP 封装
- Writer 不维护状态
- 丢失的数据包不会被重传
- 适用于对实时性要求高、可容忍数据丢失的场景

### 6.2 Reliable 传输

**文件**: `src/core/ddsi/src/ddsi_hbcontrol.c`, `src/core/ddsi/src/ddsi_receive.c`

**核心机制**:

#### 6.2.1 Writer History Cache (WHC)

- **作用**: Writer 存储已发送但未确认的样本
- **清理条件**: 当所有匹配的 Reader 都确认收到后，可以从 WHC 中移除样本
- **配置**: 通过 `WriterHistoryCacheQosPolicy` 配置大小

#### 6.2.2 Heartbeat (心跳)

**目的**: 通知 Reader 关于 Writer 中可用的数据范围

**流程**:
1. **周期性发送**: Writer 定期发送 Heartbeat 消息
   - 包含序列号范围: `[firstSN, lastSN]`
   - 标志位: FINAL（不需要响应）、LIVELINESS（活力声明）
2. **条件抑制**: 如果所有 Reader 都已同步（acknowledged），可以抑制 Heartbeat
3. **处理**: Reader 收到 Heartbeat 后检查是否需要发送 ACKNACK

**关键函数**:
- `ddsi_heartbeat_xevent_cb()`: 心跳定时器回调
- `ddsi_add_heartbeat()`: 添加心跳子消息到消息中

#### 6.2.3 AckNack (确认和否定确认)

**目的**: Reader 告诉 Writer 哪些序列号已收到，哪些丢失

**流程**:
1. **触发条件**:
   - 收到 Heartbeat 且需要响应（非 FINAL）
   - 检测到数据丢失
   - 定时器触发（即使没有 Heartbeat）
2. **内容**:
   - 位图 (bitmap) 表示已收到的序列号
   - 序列号范围
3. **处理**: Writer 收到 ACKNACK 后：
   - 检查哪些样本需要重传
   - 标记读者已确认的序列号
   - 清理 WHC 中已确认的样本

**关键函数**:
- `handle_Heartbeat()`: 处理收到的 Heartbeat
- `ddsi_sched_acknack_if_needed()`: 调度 ACKNACK 发送

#### 6.2.4 Gap 消息

**目的**: Writer 通知 Reader 某些序列号永远不会被发送（例如数据被丢弃）

**用途**:
- 当 WHC 满时，Writer 可能需要丢弃旧数据
- 通过 GAP 通知 Reader，避免 Reader 等待重传

#### 6.2.5 分片重组 (Fragmentation)

**场景**: 当样本太大，无法放入单个 UDP 数据包时

**流程**:
1. Writer 将大样本分成多个 `DATAFRAG` 子消息
2. 每个分片包含:
   - 分片序号
   - 分片大小
   - 总样本大小
3. Reader 使用 `ddsi_defrag` 重组分片
4. 只有当所有分片都收到后，才将完整样本传递给应用程序

**关键函数**:
- `ddsi_defrag_rsample()`: 处理分片重组

#### 6.2.6 序列号重排序 (Reordering)

**场景**: 网络数据包可能乱序到达

**流程**:
1. Reader 维护重排序表 (`ddsi_reorder`)
2. 收到样本后，检查序列号是否连续
3. 如果序列号不连续，将样本存储在重排序缓冲区中
4. 当缺失的样本到达后，按顺序传递给应用程序

**关键函数**:
- `ddsi_reorder_rsample()`: 处理序列号重排序

---

## 7. QoS (Quality of Service) 机制

### 7.1 主要 QoS 策略

1. **Reliability** (可靠性)
   - `BEST_EFFORT`: 尽力而为，可能丢失数据
   - `RELIABLE`: 可靠传输，保证数据送达（通过重传）

2. **Durability** (持久性)
   - `VOLATILE`: 不持久化
   - `TRANSIENT_LOCAL`: Writer 关闭后，新 Reader 仍可获取最后的值
   - `TRANSIENT`: 跨进程持久化（需要 Durability Service）
   - `PERSISTENT`: 跨重启持久化（需要 Durability Service）

3. **History** (历史)
   - `KEEP_LAST`: 只保留最后 N 个样本
   - `KEEP_ALL`: 保留所有样本（受资源限制）

4. **Liveliness** (活力)
   - 监控 Writer 是否还在运行
   - 如果 Writer 在一定时间内未声明活力，Reader 会收到 LIVELINESS_CHANGED 状态

5. **Deadline** (截止时间)
   - Writer 必须定期更新数据
   - Reader 期望定期收到新数据
   - 如果违反，会触发 DEADLINE_MISSED 状态

6. **Partition** (分区)
   - 将主题空间划分为多个分区
   - 只有相同分区的 Reader 和 Writer 才能匹配

7. **Data Representation** (数据表示)
   - `XCDR1`, `XCDR2`: CDR 编码版本
   - Reader 和 Writer 必须使用兼容的表示格式

### 7.2 QoS 兼容性匹配

**文件**: `src/core/ddsi/src/ddsi_qosmatch.c`

**规则**:
- Reader 的可靠性要求不能高于 Writer 提供的
- Reader 的持久性要求不能高于 Writer 提供的
- 分区必须重叠（如果使用）
- 数据表示格式必须兼容
- 类型一致性策略必须兼容

**匹配失败时**:
- 设置 `INCOMPATIBLE_QOS` 状态
- 调用 Listener 回调（如果配置了）
- 端点不会建立连接

---

## 8. 安全机制 (DDS Security)

### 8.1 安全插件架构

**文件**: `src/security/`, `docs/dev/modules.md`

**插件类型**:
1. **Authentication Plugin** (认证插件)
   - 参与者之间的相互认证
   - 使用证书颁发机构 (CA)
   - 建立共享密钥（使用 DH 或 ECDH）

2. **Cryptography Plugin** (加密插件)
   - 数据加密/解密（使用 AES-GCM）
   - 消息认证码 (MAC)
   - 支持 128 位和 256 位密钥

3. **Access Control Plugin** (访问控制插件)
   - 权限检查
   - 控制哪些参与者可以访问哪些主题

4. **Logging Plugin** (日志插件)
   - 审计日志

5. **Data Tagging Plugin** (数据标记插件)
   - 为数据添加标记

### 8.2 安全握手流程

1. **SPDP 安全扩展**: 在 SPDP 消息中包含证书信息
2. **相互认证**: 交换和验证证书
3. **密钥协商**: 使用 DH/ECDH 建立共享密钥
4. **安全端点创建**: 为认证的参与者创建安全端点
5. **数据加密**: 后续数据传输使用协商的密钥加密

---

## 9. 共享内存传输 (PSMX / Iceoryx)

### 9.1 PSMX 架构

**文件**: `src/core/ddsc/include/dds/ddsc/dds_psmx.h`

**目的**: 使用共享内存进行进程间通信，避免网络开销

**流程**:
1. **PSMX 实例创建**: 每个 PSMX Domain 创建一个实例
2. **Locator 匹配**: 只有相同 Locator 的实例可以通信
3. **端点映射**: 将 DDS Reader/Writer 映射到 PSMX Endpoint
4. **数据传输**: 使用共享内存传输数据
5. **回退到网络**: 如果 PSMX 不可用，回退到网络传输

### 9.2 零拷贝机制

- 使用 PSMX 时，可以实现零拷贝
- 数据直接写入共享内存，Reader 直接读取
- 避免了序列化/反序列化和网络传输的开销

---

## 10. 主要业务流程总结

### 10.1 典型发布流程

```
1. 初始化库 (dds_init)
2. 创建 DomainParticipant
3. 创建 Topic
4. 创建 Publisher 和 DataWriter
5. 发现匹配的 Reader (通过 SEDP)
6. 循环:
   a. 应用程序准备数据
   b. 调用 dds_write() 写入数据
   c. 数据序列化
   d. 添加到 WHC
   e. 发送到网络（或 PSMX）
   f. 如果是可靠的，等待确认
7. 清理资源 (dds_delete)
```

### 10.2 典型订阅流程

```
1. 初始化库 (dds_init)
2. 创建 DomainParticipant
3. 创建 Topic
4. 创建 Subscriber 和 DataReader
5. 配置 Listener 或 Waitset（可选）
6. 发现匹配的 Writer (通过 SEDP)
7. 循环:
   a. 接收网络数据包
   b. 解析 RTPS 消息
   c. 分片重组和序列号重排序
   d. 反序列化数据
   e. 添加到 RHC
   f. 通知应用程序
   g. 应用程序调用 dds_read() / dds_take() 获取数据
   h. 如果是可靠的，发送 ACKNACK
8. 清理资源 (dds_delete)
```

### 10.3 发现和匹配流程

```
1. SPDP 发现参与者
   → 创建代理参与者
2. SEDP 交换端点信息
   → 创建代理端点
3. 检查匹配条件:
   → Topic 名称匹配
   → 类型兼容
   → QoS 兼容
4. 如果匹配成功:
   → 建立连接
   → 配置可靠传输（如果需要）
   → 开始数据传输
```

### 10.4 可靠传输流程

```
Writer 端:
1. 写入数据 → 添加到 WHC
2. 发送数据包
3. 定期发送 Heartbeat
4. 接收 ACKNACK
5. 重传丢失的数据
6. 清理已确认的样本（从 WHC）

Reader 端:
1. 接收数据包
2. 检查序列号连续性
3. 检测到丢失 → 发送 ACKNACK
4. 接收 Heartbeat → 检查是否需要 ACKNACK
5. 接收重传的数据
6. 按顺序传递给应用程序
```

---

## 11. 关键数据结构总结

### 11.1 DDSC 层 (用户 API 层)

- **`dds_participant`**: 域参与者
- **`dds_publisher`** / **`dds_subscriber`**: 发布者/订阅者
- **`dds_writer`** / **`dds_reader`**: 数据写入者/读取者
- **`dds_topic`**: 主题
- **`dds_ktopic`**: 主题的键值映射（名称 → QoS/类型）

### 11.2 DDSI 层 (协议层)

- **`ddsi_participant`**: DDSI 参与者
- **`ddsi_writer`** / **`ddsi_reader`**: DDSI 端点
- **`ddsi_proxy_writer`** / **`ddsi_proxy_reader`**: 代理端点（远程）
- **`ddsi_topic_definition`**: DDSI 主题定义
- **`ddsi_whc`**: Writer History Cache
- **`ddsi_rhc`**: Reader History Cache
- **`ddsi_serdata`**: 序列化数据
- **`ddsi_xmsg`**: 发送消息
- **`ddsi_rmsg`**: 接收消息

### 11.3 传输相关

- **`ddsi_xpack`**: 发送包
- **`ddsi_rbuf`**: 接收缓冲区
- **`ddsi_defrag`**: 分片重组表
- **`ddsi_reorder`**: 序列号重排序表

---

## 12. 性能优化特性

1. **共享发现信息**: 同进程内多个 Participant 共享发现
2. **零拷贝**: 通过 PSMX 实现共享内存零拷贝传输
3. **异步传输**: 支持异步发送模式，提高吞吐量
4. **批量发送**: 多个样本打包在一个 UDP 数据包中
5. **高效的心跳控制**: 智能抑制不需要的心跳
6. **内存池**: 使用预分配的内存池减少分配开销
7. **多线程**: 接收线程和发送线程分离

---

## 13. 配置和扩展

### 13.1 XML 配置

通过 `CYCLONEDDS_URI` 环境变量指定配置文件：
- 网络接口配置
- 多播设置
- 消息大小限制
- 心跳间隔
- 线程配置
- 等等

### 13.2 插件扩展

- **PSMX 插件**: 自定义传输机制（如共享内存）
- **Security 插件**: 自定义安全实现
- **QoS Provider**: 从外部源加载 QoS 配置

---

## 结语

Cyclone DDS 实现了一个完整的分布式数据空间系统，通过精心设计的模块化架构、高效的网络协议实现和丰富的 QoS 支持，为分布式应用提供了高性能、可靠的通信机制。其核心业务逻辑围绕**发现、匹配、传输**三个关键环节展开，通过标准化的 RTPS 协议确保与其他 DDS 实现的互操作性。
