# SSM (Source-Specific Multicast) 源特定多播详解

## 1. 什么是 SSM？

**SSM (Source-Specific Multicast)** 是**源特定多播**，是多播的一个变种，允许接收者**指定发送源的地址**。

### 1.1 多播类型对比

#### ASM (Any-Source Multicast) - 任意源多播

**特点**:
- 接收者加入多播组后，可以接收来自**任何源**的数据
- 只需要指定多播组地址（如 `239.255.0.1`）
- 传统多播方式

**示例**:
```
Reader 加入多播组: 239.255.0.1
  ↓
接收来自任何 Writer 的数据:
  - Writer A -> 239.255.0.1 ✓
  - Writer B -> 239.255.0.1 ✓
  - Writer C -> 239.255.0.1 ✓
```

#### SSM (Source-Specific Multicast) - 源特定多播

**特点**:
- 接收者加入多播组时，必须**同时指定源地址**
- 只能接收来自指定源的数据
- 需要同时指定：**(源地址, 多播组地址)**

**示例**:
```
Reader 加入 SSM 组: (192.168.1.100, 232.1.1.1)
  ↓
只接收来自指定 Writer 的数据:
  - Writer A (192.168.1.100) -> 232.1.1.1 ✓
  - Writer B (192.168.1.200) -> 232.1.1.1 ✗ (忽略)
  - Writer C (192.168.1.300) -> 232.1.1.1 ✗ (忽略)
```

### 1.2 SSM 地址范围

**IPv4**:
- SSM 多播地址范围: `232.0.0.0/8` (232.x.x.x)
- ASM 多播地址范围: `224.0.0.0` 到 `231.255.255.255` 和 `233.0.0.0` 到 `239.255.255.255`

**IPv6**:
- SSM 多播地址: `ff3x::/96` (其中 x 是范围标识符)

**关键区别**: SSM 地址本身就标识了它是源特定的，而不是通过协议参数区分。

---

## 2. SSM 的优势

### 2.1 安全性

- **过滤恶意源**: 只接收来自可信源的数据
- **减少攻击面**: 即使有人发送到多播组，但源地址不匹配也会被过滤

### 2.2 网络效率

- **减少流量**: 路由器知道特定的源，可以进行更精确的路由
- **更好的 QoS**: 可以为特定的源-组组合提供服务质量保证
- **减少拥塞**: 不需要处理所有源的流量

### 2.3 精确控制

- **选择性接收**: 只为匹配的 writer 加入多播组
- **动态管理**: 当 writer 匹配/不匹配时，动态加入/离开多播组

---

## 3. SSM 在 Cyclone DDS 中的应用

### 3.1 工作流程

**文件**: `src/core/ddsi/src/ddsi_endpoint_match.c`

**当 Reader 和 Writer 匹配时**:

```c
// 如果 Reader 支持 SSM 且 Writer 支持 SSM
if (rd->favours_ssm && pwr->supports_ssm) {
    // 1. 获取 Writer 的单播源地址
    ddsi_addrset_any_uc(pwr->c.as, &ssm_src_loc);
    
    // 2. 获取 SSM 多播组地址
    ddsi_addrset_any_ssm(rd->e.gv, pwr->c.as, &ssm_mc_loc);
    
    // 3. 加入 SSM 多播组 (源地址, 多播组地址)
    ddsi_join_mc(rd->e.gv, rd->e.gv->data_conn_mc, 
                 &ssm_src_loc.c, &ssm_mc_loc.c);
}
```

**当 Reader 和 Writer 不匹配时**:
```c
// 离开 SSM 多播组
ddsi_leave_mc(rd->e.gv, rd->e.gv->data_conn_mc,
              &ssm_src_loc.c, &ssm_mc_loc.c);
```

### 3.2 系统调用

**文件**: `src/core/ddsi/src/ddsi_udp.c`

**IPv4**:
```c
struct ip_mreq_source mreq;
mreq.imr_sourceaddr = srcip;      // 源地址（Writer 的地址）
mreq.imr_multiaddr = mcip;        // 多播组地址（SSM 地址）
mreq.imr_interface = interface;   // 网络接口

// 加入 SSM 组
setsockopt(socket, IPPROTO_IP, IP_ADD_SOURCE_MEMBERSHIP, &mreq, sizeof(mreq));

// 离开 SSM 组
setsockopt(socket, IPPROTO_IP, IP_DROP_SOURCE_MEMBERSHIP, &mreq, sizeof(mreq));
```

**IPv6**:
```c
struct group_source_req gsr;
gsr.gsr_interface = interface;
gsr.gsr_group = mcip;             // SSM 多播组地址
gsr.gsr_source = srcip;           // 源地址

// 加入 SSM 组
setsockopt(socket, IPPROTO_IPV6, MCAST_JOIN_SOURCE_GROUP, &gsr, sizeof(gsr));

// 离开 SSM 组
setsockopt(socket, IPPROTO_IPV6, MCAST_LEAVE_SOURCE_GROUP, &gsr, sizeof(gsr));
```

### 3.3 配置 SSM

**XML 配置**:
```xml
<CycloneDDS>
  <Domain>
    <General>
      <!-- 启用 SSM -->
      <AllowMulticast>ssm</AllowMulticast>
      <!-- 或同时启用 ASM 和 SSM -->
      <AllowMulticast>asm,ssm</AllowMulticast>
    </General>
    <Networking>
      <!-- 配置 SSM 地址 -->
      <NetworkPartition Name="partition1">
        <Addresses>
          <Address>232.1.1.1</Address>  <!-- SSM 地址 -->
        </Addresses>
      </NetworkPartition>
    </Networking>
  </Domain>
</CycloneDDS>
```

---

## 4. 为什么 SSM 不创建专门的 recvMC 线程？

**代码**: `src/core/ddsi/src/ddsi_init.c:889-891`

```c
if (ddsi_is_mcaddr(gv, &gv->loc_default_mc) && 
    !ddsi_is_ssm_mcaddr(gv, &gv->loc_default_mc) &&  // 关键：不是 SSM
    allow_asm_mc) {
    // 只为 ASM 创建 recvMC 线程
    gv->recv_threads[n].name = "recvMC";
    ...
}
```

### 4.1 原因分析

**注释说明**（代码第 891 行）:
> "the trouble with SSM addresses is that we only join matching writers, which our own sockets typically would not be"

**关键点**:

1. **动态加入/离开**:
   - SSM 只为**匹配的 writer** 加入多播组
   - 当没有匹配的 writer 时，不会加入任何 SSM 组
   - 本地 Writer 的 socket 通常不会加入自己的 SSM 组

2. **本地回环问题**:
   - 本地 Writer 发送的数据通过**单播回环**或**共享 socket** 接收
   - 不需要在多播 socket 上接收本地数据
   - 创建专门的 recvMC 线程意义不大

3. **数据流特点**:
   - ASM: 一个固定的多播 socket 持续接收来自多个源的数据
   - SSM: 多播组是动态的，只有匹配时才接收数据

### 4.2 数据接收路径对比

**ASM 模式**:
```
Writer A (远程) -> 239.255.0.1 -> recvMC 线程接收 ✓
Writer B (远程) -> 239.255.0.1 -> recvMC 线程接收 ✓
Writer C (本地) -> 239.255.0.1 -> 单播回环 ✓
```

**SSM 模式**:
```
Writer A (远程, 已匹配) -> (A的IP, 232.1.1.1) 
  -> recv 线程接收（通过 data_conn_mc）✓

Writer B (远程, 未匹配) -> (B的IP, 232.1.1.1)
  -> 未加入组，不接收 ✗

Writer C (本地) -> 232.1.1.1
  -> 单播回环或共享 socket ✓
```

**关键**: SSM 的数据流量是**动态的**，而 ASM 是**持续稳定的**。

---

## 5. SSM vs ASM 在 Cyclone DDS 中的对比

### 5.1 配置对比

| 特性 | ASM | SSM |
|------|-----|-----|
| 地址范围 | 239.255.0.1 (或其他) | 232.0.0.0/8 |
| 加入方式 | `IP_ADD_MEMBERSHIP` | `IP_ADD_SOURCE_MEMBERSHIP` |
| 需要源地址 | ❌ | ✅ |
| recvMC 线程 | ✅ (通常) | ❌ |
| 本地回环 | 需要 | 通常不需要 |

### 5.2 使用场景

**ASM 适合**:
- 简单的广播场景
- 所有参与者都在同一子网
- 不需要源过滤
- 大量 writer 发送数据

**SSM 适合**:
- 需要安全性的场景
- 精确控制数据源
- 跨越路由器边界（更好的路由支持）
- 每个 reader 只匹配少量 writer

### 5.3 Reader 如何选择

**Reader 的 `favours_ssm` 标志**:

```c
// Reader 声明是否偏好 SSM
rd->favours_ssm = 1;  // 偏好 SSM

// 在发现消息中通告
ps.present |= PP_READER_FAVOURS_SSM;
ps.reader_favours_ssm.state = 1u;
```

**匹配规则**:
- 如果 Reader `favours_ssm` 且 Writer `supports_ssm` → 使用 SSM
- 否则 → 使用 ASM 或单播

---

## 6. SSM 地址判断

**文件**: `src/core/ddsi/src/ddsi_udp.c`

```c
static int ddsi_udp_is_ssm_mcaddr(const struct ddsi_tran_factory *tran, 
                                   const ddsi_locator_t *loc)
{
    if (loc->kind == DDSI_LOCATOR_KIND_UDPv4) {
        // IPv4 SSM 地址范围: 232.0.0.0/8
        return (loc->address[12] == 232);
    }
    else if (loc->kind == DDSI_LOCATOR_KIND_UDPv6) {
        // IPv6 SSM 地址: ff3x::/96
        return (loc->address[0] == 0xff && 
                (loc->address[1] & 0x0f) == 0x03);
    }
    return 0;
}
```

---

## 7. 实际应用示例

### 7.1 配置 SSM

```xml
<CycloneDDS>
  <Domain Id="0">
    <General>
      <!-- 启用 SSM -->
      <AllowMulticast>ssm</AllowMulticast>
    </General>
    <Networking>
      <NetworkPartition Name="SecurePartition">
        <Addresses>
          <!-- 使用 SSM 地址范围 -->
          <Address>232.1.1.1</Address>
        </Addresses>
      </NetworkPartition>
    </Networking>
  </Domain>
</CycloneDDS>
```

### 7.2 工作流程示例

```
场景: Reader 匹配远程 Writer

1. Writer 通告（SEDP）:
   - 包含 SSM 地址: 232.1.1.1
   - 单播地址: 192.168.1.100

2. Reader 匹配:
   - 检查 Topic 和 QoS
   - 匹配成功

3. Reader 加入 SSM 组:
   - 源地址: 192.168.1.100
   - 多播组: 232.1.1.1
   - 调用: IP_ADD_SOURCE_MEMBERSHIP

4. Writer 发送数据:
   - 发送到: 232.1.1.1
   - 源地址: 192.168.1.100
   - 只有加入该 (源,组) 的 Reader 能收到

5. Reader 接收数据:
   - 在 data_conn_mc socket 上接收
   - 由 recv 线程处理（不是 recvMC）
```

---

## 8. 总结

### 8.1 SSM 核心概念

1. **源特定**: 必须指定源地址才能接收数据
2. **地址范围**: IPv4 使用 232.x.x.x，IPv6 使用 ff3x::
3. **动态性**: 只为匹配的 writer 加入/离开组
4. **安全性**: 只接收来自指定源的数据

### 8.2 为什么 SSM 不需要 recvMC 线程？

1. **动态加入**: 只为匹配的 writer 加入，不是持续的多播流
2. **本地处理**: 本地 writer 的数据通过单播回环
3. **流量特点**: 不像 ASM 那样有稳定的多播流量

### 8.3 使用建议

- **需要安全性**: 使用 SSM
- **简单场景**: 使用 ASM
- **WiFi 网络**: 通常只使用 SPDP 多播，数据用单播
- **有线网络**: 可以同时使用 ASM 和 SSM

SSM 提供了更精细的控制和更好的安全性，但增加了配置和管理的复杂性。在 Cyclone DDS 中，SSM 是一个可选特性，默认情况下使用 ASM 或单播。
