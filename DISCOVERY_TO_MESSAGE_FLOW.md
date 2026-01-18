# Cyclone DDS: 从发现过程到报文发送接收的详细流程

## 目录

1. [发现过程详解](#1-发现过程详解)
   - [1.1 SPDP 发现流程](#11-spdp-发现流程)
   - [1.2 SEDP 端点发现流程](#12-sedp-端点发现流程)
2. [报文发送流程详解](#2-报文发送流程详解)
   - [2.1 应用程序写入数据](#21-应用程序写入数据)
   - [2.2 数据序列化](#22-数据序列化)
   - [2.3 RTPS 消息构建](#23-rtps-消息构建)
   - [2.4 网络发送](#24-网络发送)
3. [报文接收流程详解](#3-报文接收流程详解)
   - [3.1 接收线程和缓冲区管理](#31-接收线程和缓冲区管理)
   - [3.2 RTPS 消息解析](#32-rtps-消息解析)
   - [3.3 分片重组和序列号重排序](#33-分片重组和序列号重排序)
   - [3.4 数据传递到应用程序](#34-数据传递到应用程序)

---

## 1. 发现过程详解

### 1.1 SPDP 发现流程

**目的**: 在域中发现其他参与者 (Participants)

**文件**: `src/core/ddsi/src/ddsi_discovery_spdp.c`

#### 1.1.1 SPDP 消息发送

**触发时机**:
1. Participant 创建时立即发送
2. 周期性定时发送（默认间隔由 `spdp_interval` 配置）
3. 收到新的 SPDP 消息后，响应发送（避免冲突使用随机延迟）

**发送流程**:

```
ddsi_spdp_write() 或定时器回调
    ↓
1. 构建 SPDP 数据 (ddsi_get_participant_builtin_topic_data)
   - 参与者 GUID
   - 内置端点集合 (Built-in Endpoint Set)
   - 单播地址 (Unicast Locators)
   - 多播地址 (Multicast Locators)
   - QoS 信息
   - 协议版本、厂商ID等
    ↓
2. 创建序列化数据 (ddsi_serdata_from_sample)
   - 使用 CDR 序列化 plist 数据
    ↓
3. 写入 SPDP Writer (ddsi_write_sample_nogc)
   - SPDP 使用 best-effort 传输（不需要 WHC）
   - 直接创建 RTPS 消息
    ↓
4. 构建 RTPS 消息 (ddsi_create_fragment_message)
   - RTPS Header:
     * Protocol Version
     * Vendor ID
     * GUID Prefix
   - DATA Submessage:
     * Writer ID: DDSI_ENTITYID_SPDP_BUILTIN_PARTICIPANT_WRITER
     * Reader ID: DDSI_ENTITYID_SPDP_BUILTIN_PARTICIPANT_READER (或 UNKNOWN)
     * Sequence Number
     * Serialized Payload (plist)
    ↓
5. 设置目标地址
   - 多播地址: 239.255.0.1 (IPv4) 或 ff02::ffff:239.255.0.1 (IPv6)
   - 单播地址: 配置的对等节点列表 (Peers)
    ↓
6. 添加到发送队列 (ddsi_xpack_add)
   - 将 xmsg 添加到 xpack
   - 当 xpack 满或 flush 时发送
    ↓
7. 网络发送 (ddsi_xpack_send -> ddsi_conn_write)
   - UDP sendmsg() 发送数据包
```

**关键代码位置**:
- `ddsi_get_participant_builtin_topic_data()`: 构建 SPDP 数据
- `ddsi_spdp_write()`: 写入 SPDP 消息
- `ddsi_write_sample_nogc()`: 写入样本（不需要 GC）

#### 1.1.2 SPDP 消息接收

**接收流程**:

```
接收线程 (do_packet)
    ↓
1. 接收 UDP 数据包 (recvmsg)
   - 使用接收缓冲区 (ddsi_rbuf)
   - 创建接收消息 (ddsi_rmsg)
    ↓
2. 解析 RTPS 消息 (handle_rtps_message)
   - 验证 RTPS Header
   - 检查 Vendor ID、协议版本
    ↓
3. 遍历子消息 (handle_submsg_sequence)
   - 对于 DATA 子消息:
     * 检查 Writer ID 是否为 SPDP Writer
     * 检查 Reader ID 是否匹配
    ↓
4. 验证和解析数据 (validate_Data)
   - 检查子消息大小
   - 字节序转换（如果需要）
   - 提取序列号、时间戳
    ↓
5. 处理 SPDP 数据 (handle_Data -> ddsi_handle_spdp)
   a. 反序列化 plist (ddsi_serdata_to_sample)
   b. 提取参与者信息:
      * participant_guid
      * builtin_endpoint_set
      * unicast_locators / multicast_locators
      * QoS 信息
   c. 检查是否为已知参与者:
      * 本地参与者: 忽略（可能是环路）
      * 已存在的代理参与者: 更新信息
      * 新的参与者: 创建代理参与者
    ↓
6. 创建/更新代理参与者 (ddsi_new_proxy_participant)
   a. 检查参与者是否已存在
   b. 创建代理参与者对象:
      * 存储 GUID
      * 存储地址信息
      * 初始化租约 (Lease)
      * 创建代理内置端点
   c. 更新序列号和租约时间
   d. 触发响应（如果是新发现的）
    ↓
7. 响应新发现的参与者 (respond_to_spdp)
   - 随机延迟避免冲突
   - 发送定向 SPDP 消息给新发现的参与者
    ↓
8. 启动 SEDP 发现
   - 当双方都匹配了内置端点后
   - 开始交换端点信息
```

**关键代码位置**:
- `handle_rtps_message()`: 处理 RTPS 消息
- `ddsi_handle_spdp()`: 处理 SPDP 数据
- `ddsi_new_proxy_participant()`: 创建代理参与者

**消息格式**:

```
RTPS Message:
  +---------------------------+
  | RTPS Header               |
  | - Protocol Version (1)    |
  | - Vendor ID              |
  | - GUID Prefix            |
  +---------------------------+
  | INFO_TS Submessage        |  (可选，时间戳)
  +---------------------------+
  | DATA Submessage           |
  | - Submessage Header       |
  | - Writer ID              |
  | - Reader ID              |
  | - Sequence Number        |
  | - Serialized Payload:    |
  |   * Participant GUID     |
  |   * Built-in Endpoint Set|
  |   * Unicast Locators     |
  |   * Multicast Locators   |
  |   * QoS (User Data, etc) |
  +---------------------------+
```

### 1.2 SEDP 端点发现流程

**目的**: 在已发现的参与者之间交换端点（Reader/Writer）信息

**文件**: `src/core/ddsi/src/ddsi_discovery_endpoint.c`

#### 1.2.1 SEDP 消息发送

**触发时机**:
1. Writer/Reader 创建时立即发送
2. Writer/Reader 删除时发送 DISPOSE/UNREGISTER 消息
3. QoS 更改时重新发送

**发送流程**:

```
端点创建时 (ddsi_reader_or_writer_create)
    ↓
1. 构建端点数据 (sedp_write_endpoint_impl)
   - Endpoint GUID
   - Topic 名称
   - 类型名称
   - QoS 信息
   - Unicast/Multicast Locators
   - Security Info (如果启用)
    ↓
2. 创建序列化数据
   - 使用 CDR 序列化 plist
   - 标记为 reliable + transient-local
    ↓
3. 写入 SEDP Writer
   - SEDP Writer 根据端点类型选择:
     * DataWriter -> SEDP Publication Writer
     * DataReader -> SEDP Subscription Writer
   - 使用 ddsi_write_sample_nogc() 写入
    ↓
4. 构建 RTPS 消息
   - DATA Submessage:
     * Writer ID: SEDP Publication/Subscription Writer
     * Reader ID: SEDP Publication/Subscription Reader
     * Sequence Number
     * Serialized Payload (端点 plist)
    ↓
5. 可靠传输处理
   - 添加到 SEDP Writer 的 WHC
   - 发送 Heartbeat
   - 等待 AckNack 确认
    ↓
6. 网络发送
   - 发送到所有已知的代理参与者
```

**关键代码位置**:
- `sedp_write_endpoint_impl()`: 写入端点信息
- `ddsi_write_and_fini_plist()`: 写入 plist 数据

#### 1.2.2 SEDP 消息接收

**接收流程**:

```
接收线程接收 SEDP 消息
    ↓
1. 解析 RTPS 消息
   - 识别 DATA 子消息
   - 检查 Writer ID 是否为 SEDP Writer
    ↓
2. 反序列化数据 (ddsi_handle_sedp)
   - 解析端点 plist
   - 提取端点 GUID、Topic、类型、QoS
    ↓
3. 检查状态信息 (statusinfo)
   a. statusinfo == 0 (ALIVE):
      - 端点存活，创建或更新代理端点
   b. statusinfo & DISPOSE/UNREGISTER:
      - 端点已删除，清理代理端点
    ↓
4. 处理 ALIVE 端点 (ddsi_handle_sedp_alive_endpoint)
   a. 验证端点 GUID
   b. 查找或创建代理端点:
      - 代理 Writer (ddsi_new_proxy_writer)
      - 代理 Reader (ddsi_new_proxy_reader)
   c. 存储端点信息:
      * Topic 名称
      * 类型信息
      * QoS
      * 地址信息
   d. 触发端点匹配
    ↓
5. 端点匹配 (ddsi_endpoint_match)
   a. 遍历所有本地端点
   b. 对每对 (本地Reader, 远程Writer) 或 (本地Writer, 远程Reader):
      - 检查 Topic 名称匹配
      - 检查类型兼容性
      - 检查 QoS 兼容性 (ddsi_qos_match_p)
   c. 如果匹配成功:
      - 建立连接
      - 配置可靠传输（如果需要）
      - 通知应用程序
    ↓
6. 处理已删除端点 (ddsi_handle_sedp_dead_endpoint)
   - 删除代理端点
   - 清理连接
   - 通知应用程序
```

**关键代码位置**:
- `ddsi_handle_sedp()`: 处理 SEDP 消息
- `ddsi_handle_sedp_alive_endpoint()`: 处理存活端点
- `ddsi_endpoint_match()`: 端点匹配

**消息格式**:

```
RTPS Message:
  +---------------------------+
  | RTPS Header               |
  +---------------------------+
  | INFO_TS Submessage        |  (时间戳)
  +---------------------------+
  | DATA Submessage           |
  | - Writer ID: SEDP Writer  |
  | - Reader ID: SEDP Reader  |
  | - Sequence Number        |
  | - Serialized Payload:    |
  |   * Endpoint GUID        |
  |   * Topic Name           |
  |   * Type Name            |
  |   * QoS                  |
  |   * Locators             |
  +---------------------------+
```

---

## 2. 报文发送流程详解

### 2.1 应用程序写入数据

**文件**: `src/core/ddsc/src/dds_write.c`

**流程**:

```
应用程序调用 dds_write(writer, data, timestamp)
    ↓
dds_write_impl()
    ↓
1. 参数验证
   - 验证 Writer 状态
   - 验证时间戳
   - 评估 Topic 过滤器（如果有）
    ↓
2. 确定传输方式
   - 检查是否有 PSMX (共享内存) 传输
   - 检查是否有网络传输
   - 确定是否需要序列化
    ↓
3. 序列化数据 (ddsi_serdata_from_sample)
   - 详见 2.2 节
    ↓
4. 传输数据
   - 如果是 PSMX: 通过共享内存传输
   - 如果是网络: 调用 deliver_data_network()
    ↓
5. 网络传输路径
   - ddsi_write_sample_gc()
   - 详见后续步骤
```

### 2.2 数据序列化

**文件**: `src/core/ddsi/src/ddsi_serdata.c`, `src/core/cdr/`

**流程**:

```
ddsi_serdata_from_sample()
    ↓
1. 根据类型选择序列化方法
   - 内置类型: 使用内置序列化
   - 用户类型: 使用 CDR 序列化
    ↓
2. CDR 序列化 (dds_cdrstream_write)
   a. 计算序列化大小 (get_serialized_size)
      - 遍历所有字段
      - 计算每个字段的大小（包括对齐）
   b. 分配序列化缓冲区
   c. 写入 CDR Header:
      * CDR 编码版本 (XCDR1 或 XCDR2)
      * 字节序标志
   d. 按字段顺序序列化:
      * 基本类型: 直接写入（注意字节序）
      * 字符串/序列: 长度 + 数据
      * 结构体: 递归序列化成员
      * 联合体: discriminator + 选中的成员
   e. 处理对齐 (padding)
    ↓
3. 创建 serdata 对象
   - 存储序列化后的数据
   - 存储元数据（类型、大小、时间戳等）
   - 设置引用计数
```

**CDR 格式示例**:

```
CDR Serialized Data:
  +---------------------------+
  | CDR Header (4 bytes)      |
  | - Encoding Version        |
  | - Endianness Flag         |
  +---------------------------+
  | Field 1 (aligned)         |
  +---------------------------+
  | Field 2 (aligned)         |
  +---------------------------+
  | ...                       |
  +---------------------------+
```

### 2.3 RTPS 消息构建

**文件**: `src/core/ddsi/src/ddsi_transmit.c`, `src/core/ddsi/src/ddsi_xmsg.c`

**流程**:

```
ddsi_write_sample_gc()
    ↓
1. Writer 状态检查
   - 检查 Writer 是否存活
   - 检查样本大小限制
   - 更新活力租约（如果启用）
    ↓
2. 分配序列号 (seq)
   - wr->seq++ (原子操作)
    ↓
3. 添加到 WHC (ddsi_whc_insert) - 仅可靠传输
   - 创建 whc_node
   - 插入序列号哈希表
   - 插入序列号区间树
   - 检查 WHC 高水位，可能需要阻塞
    ↓
4. 判断是否需要分片
   - 如果样本大小 <= fragment_size:
     使用 DATA Submessage
   - 如果样本大小 > fragment_size:
     使用 DATAFRAG Submessage (多个)
    ↓
5. 创建 RTPS 消息 (ddsi_create_fragment_message)
   
   5.1 创建 xmsg 对象
       - 分配消息缓冲区
       - 设置 Writer GUID
       - 设置目标地址集合
    ↓
   5.2 添加 INFO_TS Submessage (如果是第一个分片)
       - 时间戳 (serdata->timestamp)
    ↓
   5.3 添加 DATA 或 DATAFRAG Submessage
       
       对于 DATA (小样本):
       +---------------------------+
       | DATA Submessage Header    |
       | - Submessage ID           |
       | - Flags (KEY/DATA)        |
       | - Octets to Next          |
       +---------------------------+
       | Reader ID                 |
       | Writer ID                 |
       | Sequence Number           |
       | Octets to Inline QoS      |
       +---------------------------+
       | Inline QoS (可选)         |
       | - StatusInfo              |
       | - KeyHash                 |
       +---------------------------+
       | Serialized Payload        |
       | (CDR data)                |
       +---------------------------+
       
       对于 DATAFRAG (大样本):
       +---------------------------+
       | DATAFRAG Submessage Header|
       +---------------------------+
       | Reader ID                 |
       | Writer ID                 |
       | Writer SN                 |
       | Fragment Starting Number  |
       | Fragments in Submessage   |
       | Fragment Size             |
       | Sample Size               |
       +---------------------------+
       | Inline QoS (可选)         |
       +---------------------------+
       | Fragment Data             |
       +---------------------------+
    ↓
   5.4 添加其他子消息 (如果需要)
       - INFO_DST: 目标 GUID Prefix
       - INFO_SRC: 源 GUID Prefix
    ↓
6. 添加到发送包 (ddsi_xpack_add)
   - 将 xmsg 添加到 xpack
   - xpack 是一个 scatter/gather 列表
   - 当 xpack 满或 flush 时准备发送
```

**关键代码位置**:
- `ddsi_write_sample_gc()`: 写入样本
- `ddsi_create_fragment_message()`: 创建分片消息
- `ddsi_xmsg_new()`: 创建新消息
- `ddsi_xpack_add()`: 添加到发送包

### 2.4 网络发送

**文件**: `src/core/ddsi/src/ddsi_xpack.c`, `src/core/ddsi/src/ddsi_udp.c`

**流程**:

```
ddsi_xpack_send()
    ↓
1. 检查发送模式
   a. 同步模式:
      - 立即发送
   b. 异步模式:
      - 添加到发送队列
      - 由发送线程处理
    ↓
2. 构建发送列表
   - 遍历 xpack 中的所有 xmsg
   - 构建 scatter/gather 列表 (msgfrags)
   - 计算总大小
    ↓
3. 安全编码 (如果启用 DDS Security)
   - 加密数据负载
   - 添加安全前缀
    ↓
4. 遍历目标地址集合
   对于每个目标地址:
    ↓
5. 发送 UDP 数据包 (ddsi_conn_write -> ddsi_udp_conn_write)
   a. 构建 msghdr:
      - msg_name: 目标地址 (sockaddr)
      - msg_iov: scatter/gather 列表
      - msg_iovlen: 列表长度
   b. 调用 sendmsg()
    ↓
6. 清理
   - 释放 xmsg 对象（如果引用计数为 0）
   - 更新发送统计
```

**UDP 数据包结构**:

```
UDP Packet:
  +---------------------------+
  | IP Header                 |
  +---------------------------+
  | UDP Header                |
  | - Source Port             |
  | - Dest Port               |
  | - Length                  |
  +---------------------------+
  | RTPS Message              |
  | (见 2.3 节)               |
  +---------------------------+
```

---

## 3. 报文接收流程详解

### 3.1 接收线程和缓冲区管理

**文件**: `src/core/ddsi/src/ddsi_receive.c`, `src/core/ddsi/src/ddsi_radmin.c`

**接收线程主循环**:

```
接收线程主循环 (do_packet)
    ↓
while (running)
{
    1. 分配接收缓冲区 (ddsi_rmsg_new)
       - 从 rbufpool 分配
       - rbuf 是大块缓冲区 (默认 1MB)
       - 可以容纳多个 UDP 数据包
    ↓
    2. 接收 UDP 数据包 (ddsi_conn_read -> recvmsg)
       - 使用接收缓冲区的一部分
       - 最大大小: 64KB (UDP 限制)
    ↓
    3. 处理数据包 (handle_rtps_message)
       - 详见 3.2 节
    ↓
    4. 提交缓冲区 (ddsi_rmsg_commit)
       - 如果没有引用，释放缓冲区
       - 如果有引用（分片/重排序），保留缓冲区
}
```

**缓冲区管理**:

```
ddsi_rbufpool (接收缓冲区池)
  └── ddsi_rbuf (大块缓冲区，1MB)
       └── ddsi_rmsg (UDP 数据包)
            └── ddsi_rdata (RTPS 子消息)
                 └── 分片重组状态
                 └── 重排序状态
```

### 3.2 RTPS 消息解析

**文件**: `src/core/ddsi/src/ddsi_receive.c`

**流程**:

```
handle_rtps_message()
    ↓
1. 解析 RTPS Header
   - 协议版本
   - Vendor ID
   - GUID Prefix (源)
    ↓
2. 验证消息
   - 检查协议版本兼容性
   - 检查消息大小
    ↓
3. 初始化接收状态 (ddsi_receiver_state)
   - 存储连接信息
   - 存储源/目标 GUID Prefix
   - 存储时间戳
    ↓
4. 遍历子消息 (handle_submsg_sequence)
   
   while (submsg < end)
   {
       a. 解析子消息头 (ddsi_rtps_submessage_header)
          - Submessage ID
          - Flags
          - Octets to Next Header
       ↓
       b. 字节序转换 (如果需要)
       ↓
       c. 根据 Submessage ID 分发:
          
          DATA / DATAFRAG:
          → handle_Data() / handle_DataFrag()
          
          HEARTBEAT:
          → handle_Heartbeat()
          
          ACKNACK:
          → handle_AckNack()
          
          GAP:
          → handle_Gap()
          
          INFO_TS:
          → handle_InfoTS() (提取时间戳)
          
          INFO_SRC / INFO_DST:
          → handle_InfoSRC() / handle_InfoDST()
       ↓
       d. 移动到下一个子消息
   }
```

**子消息处理 - DATA/DATAFRAG**:

```
handle_Data() / handle_DataFrag()
    ↓
1. 验证子消息 (validate_Data / validate_DataFrag)
   - 检查大小
   - 提取序列号、时间戳
   - 提取 KeyHash (如果有)
    ↓
2. 安全解码 (如果启用 DDS Security)
   - 解密数据负载
    ↓
3. 查找目标 Writer
   - 根据 Writer ID 查找代理 Writer
   - 检查是否匹配
    ↓
4. 创建 rdata 对象
   - 指向接收缓冲区中的数据
   - 存储序列号、时间戳等信息
    ↓
5. 分片重组 (如果是 DATAFRAG)
   - 详见 3.3 节
    ↓
6. 序列号重排序 (如果是可靠传输)
   - 详见 3.3 节
    ↓
7. 传递数据
   - 添加到 Reader History Cache
   - 通知应用程序
```

### 3.3 分片重组和序列号重排序

**文件**: `src/core/ddsi/src/ddsi_defrag.c`, `src/core/ddsi/src/ddsi_reorder.c`

#### 3.3.1 分片重组

**流程**:

```
handle_DataFrag()
    ↓
1. 查找或创建分片状态 (ddsi_defrag_rsample)
   - 为每个 Writer + Sequence Number 维护分片状态
   - 状态包括:
     * 已收到的分片位图
     * 总样本大小
     * 分片数量
    ↓
2. 记录收到的分片
   - 更新位图
   - 存储分片数据指针（在接收缓冲区中）
    ↓
3. 检查是否所有分片都收到
   - 如果所有分片都收到:
     a. 创建完整样本的 rdata
     b. 返回给调用者
   - 如果还有缺失分片:
     a. 保留分片状态
     b. 返回 NULL (等待更多分片)
    ↓
4. 超时处理
   - 如果超过一定时间未收到所有分片
   - 清理分片状态
   - 丢弃数据
```

#### 3.3.2 序列号重排序

**流程**:

```
handle_Data/DataFrag -> ddsi_reorder_rsample()
    ↓
1. 检查序列号连续性
   - 期望序列号 = 上次收到的序列号 + 1
   - 如果序列号 == 期望序列号:
     → 立即传递 (DELIVER)
   - 如果序列号 > 期望序列号:
     → 数据乱序，需要缓冲 (BUFFER)
   - 如果序列号 < 期望序列号:
     → 重复数据，丢弃 (DISCARD)
    ↓
2. 缓冲乱序数据
   - 插入重排序缓冲区（按序列号排序）
   - 更新期望序列号
    ↓
3. 检查是否可以传递连续数据
   - 从重排序缓冲区检查:
     * 如果下一个期望的序列号在缓冲区中
     * 连续传递所有可用的数据
   - 更新期望序列号
    ↓
4. 检测丢失数据 (通过 GAP 或 HEARTBEAT)
   - 如果收到 GAP [a, b]:
     * 标记序列号 a 到 b 为丢失
     * 发送 AckNack 请求重传
   - 如果收到 Heartbeat [first, last]:
     * 检查 first 到当前期望序列号之间的缺失
     * 发送 AckNack
```

### 3.4 数据传递到应用程序

**文件**: `src/core/ddsi/src/ddsi_receive.c`, `src/core/ddsc/src/dds_read.c`

**流程**:

```
传递数据 (deliver_user_data_synchronously)
    ↓
1. 反序列化数据 (ddsi_serdata_from_ser)
   - 从接收缓冲区的 CDR 数据创建 serdata
   - 验证数据有效性
   - 存储到 serdata 对象
    ↓
2. 实例处理 (ddsi_tkmap_lookup_instance)
   - 对于有键的主题:
     * 从样本中提取键
     * 查找或创建实例句柄
     * 更新实例映射
    ↓
3. 添加到 Reader History Cache (rhc_store)
   - 根据历史策略:
     * KEEP_LAST: 只保留最后 N 个样本
     * KEEP_ALL: 保留所有样本（受资源限制）
   - 更新实例状态
   - 触发条件通知（如果有等待的 WaitSet）
    ↓
4. 通知应用程序
   a. 通过 Listener (如果配置):
      - 调用 on_data_available 回调
   b. 通过 WaitSet (如果等待):
      - 触发条件
      - 唤醒等待的线程
    ↓
5. 应用程序读取数据 (dds_read / dds_take)
   a. 从 RHC 获取样本列表
   b. 反序列化到应用程序类型 (ddsi_serdata_to_sample)
   c. 返回给应用程序
   d. 如果是 dds_take:
      - 从 RHC 中移除样本
      - 释放 serdata
```

**关键代码位置**:
- `deliver_user_data_synchronously()`: 同步传递数据
- `rhc_store()`: 存储到 Reader History Cache
- `dds_read()` / `dds_take()`: 应用程序读取接口

---

## 4. 完整流程时序图

### 4.1 发现流程时序

```
Participant A              Participant B
     |                          |
     |--SPDP (多播)------------>|
     |<--SPDP (多播)------------|
     |                          |
     |--SEDP Writer------------>|
     |<--SEDP Reader------------|
     |                          |
     |--SEDP Reader------------>|
     |<--SEDP Writer------------|
     |                          |
   端点匹配成功                  端点匹配成功
     |                          |
   开始数据传输                 开始数据传输
```

### 4.2 数据传输时序（可靠传输）

```
Writer                      Reader
  |                           |
  |--DATA (seq=1)----------->|
  |                           |
  |--DATA (seq=2)----------->|
  |                           |
  |--HEARTBEAT [1,2]--------->|
  |<--ACKNACK [1,2]-----------|
  |                           |
  |--DATA (seq=3)----------->|
  |--HEARTBEAT [1,3]--------->|
  |<--ACKNACK [1,3]-----------|
  |                           |
  |--DATA (seq=4) (丢失)     |
  |--HEARTBEAT [1,5]--------->|
  |<--ACKNACK [1,3],[5,5]-----| (请求重传 seq=4)
  |                           |
  |--DATA (seq=4, 重传)------>|
  |<--ACKNACK [1,5]-----------|
  |                           |
```

---

## 5. 关键数据结构总结

### 5.1 发送相关

- **`ddsi_serdata`**: 序列化数据对象
- **`ddsi_xmsg`**: RTPS 消息对象
- **`ddsi_xpack`**: 发送包，包含多个 xmsg
- **`ddsi_whc`**: Writer History Cache

### 5.2 接收相关

- **`ddsi_rbufpool`**: 接收缓冲区池
- **`ddsi_rbuf`**: 接收缓冲区（大块）
- **`ddsi_rmsg`**: 接收消息（UDP 数据包）
- **`ddsi_rdata`**: 接收数据（RTPS 子消息）
- **`ddsi_defrag`**: 分片重组表
- **`ddsi_reorder`**: 序列号重排序表
- **`ddsi_rhc`**: Reader History Cache

### 5.3 发现相关

- **`ddsi_proxy_participant`**: 代理参与者
- **`ddsi_proxy_writer`**: 代理 Writer
- **`ddsi_proxy_reader`**: 代理 Reader
- **`ddsi_plist`**: 参数列表（发现数据）

---

## 6. 性能优化要点

1. **零拷贝**: 接收缓冲区中的数据直接传递给应用程序（未完全实现）
2. **批量发送**: 多个样本打包在一个 UDP 数据包中
3. **异步发送**: 发送队列由独立线程处理
4. **分片优化**: 只在需要时进行分片重组
5. **重排序优化**: 只在乱序时缓冲数据
6. **内存池**: 预分配常用对象（xmsg、whc_node 等）

---

## 结语

本文档详细描述了 Cyclone DDS 从发现过程到报文发送接收的完整流程。理解这些流程对于：

- 调试网络问题
- 性能优化
- 扩展功能
- 理解 DDS 协议实现

都至关重要。每个步骤都涉及复杂的同步、内存管理和协议处理，但整体架构设计使得系统既高效又可靠。
