好的，这个实验是关于Cisco Packet Tracer中的NAT（网络地址转换）配置。我们将根据提供的拓扑图和实验要求，一步步完成配置并解释原因。

**实验目标：**

1. 搭建拓扑图并完成基础配置。
2. 配置动态NAT，并指定NAT接口。
3. 配置默认路由和静态路由，以实现全网互通。
4. 测试连通性。
5. 查看地址转换结果和地址池配置。

------

### 实验拓扑图设备列表：

- **PC:** A, B, C (连接到SwitchA)
- **交换机:** SwitchA
- **路由器:** NATDynamic, ISP (路由器之间通过串口连接)
- **服务器:** S (连接到ISP路由器)

### 第一部分：拓扑搭建与基础配置

**1. 搭建拓扑图**

- 在Cisco Packet Tracer中，拖拽出所有需要的设备，并按照拓扑图连接网线。
  - PC与SwitchA之间使用直通线。
  - SwitchA与NATDynamic路由器之间使用直通线（NATDynamic的Fa0/0口连接SwitchA）。
  - NATDynamic路由器与ISP路由器之间使用串行线（NATDynamic的Se0/0/0口连接ISP的Se0/0/0口）。
  - ISP路由器与Server S之间使用直通线（ISP的Fa0/0口连接Server S）。

**2. 配置IP地址和子网掩码**

- **PC A:**
  - IP地址：10.136.10.1
  - 子网掩码：255.255.255.0
  - 默认网关：10.136.10.254 (NATDynamic的Fa0/0口IP)
- **PC B:**
  - IP地址：10.136.10.2
  - 子网掩码：255.255.255.0
  - 默认网关：10.136.10.254
- **PC C:**
  - IP地址：10.136.10.3
  - 子网掩码：255.255.255.0
  - 默认网关：10.136.10.254
- **SwitchA:** (本实验中交换机默认无需配置，因为只涉及二层转发)
- **NATDynamic 路由器:**
  - Fa0/0 口 (连接内部网络):
    - IP地址：10.136.10.254
    - 子网掩码：255.255.255.0
  - Se0/0/0 口 (连接ISP):
    - IP地址：125.77.120.61
    - 子网掩码：255.255.255.252 (请注意这是/30子网掩码)
    - 时钟速率：64000 (通常在DCE端配置，这里可以是NATDynamic或ISP，取决于谁是DCE)
- **ISP 路由器:**
  - Se0/0/0 口 (连接NATDynamic):
    - IP地址：125.77.120.62
    - 子网掩码：255.255.255.252
  - Fa0/0 口 (连接Server S):
    - IP地址：121.41.74.254
    - 子网掩码：255.255.255.0
- **Server S:**
  - IP地址：121.41.74.100
  - 子网掩码：255.255.255.0
  - 默认网关：121.41.74.254 (ISP的Fa0/0口IP)

------

### 第二部分：配置动态NAT

动态NAT允许内部网络中的多台设备共享一个或多个公共IP地址。当内部设备发起连接时，路由器会从一个公共IP地址池中分配一个IP地址给它。

**在NATDynamic路由器上配置：**

```
NATDynamic>enable
NATDynamic#configure terminal
NATDynamic(config)#ip nat pool NAT_POOL 125.77.120.65 125.77.120.70 netmask 255.255.255.248
// 定义一个名为NAT_POOL的地址池，包含IP地址范围125.77.120.65到125.77.120.70。
// 为什么这样做：这些IP地址是公共IP，用于内部网络设备访问外部网络时进行地址转换。
// 注意：这个地址范围需要是你可用的公共IP地址，并且不与现有接口IP冲突。这里我假设125.77.120.64/29这个网络段是可用的。

NATDynamic(config)#access-list 1 permit 10.136.10.0 0.0.0.255
// 定义一个标准ACL，匹配内部网络10.136.10.0/24的所有IP地址。
// 为什么这样做：ACL用于指定哪些内部IP地址允许进行NAT转换。这里我们允许整个内部子网进行NAT。

NATDynamic(config)#ip nat inside source list 1 pool NAT_POOL
// 将ACL 1中匹配的内部源地址，转换为NAT_POOL中分配的公共IP地址。
// 为什么这样做：这是动态NAT的关键命令，它将内部私有IP地址映射到NAT地址池中的公共IP地址。

NATDynamic(config)#interface Fa0/0
NATDynamic(config-if)#ip nat inside
// 将Fa0/0接口（连接内部网络）定义为NAT的内部接口。
// 为什么这样做：路由器需要知道哪些接口是内部的，其流量需要进行NAT转换。

NATDynamic(config-if)#interface Se0/0/0
NATDynamic(config-if)#ip nat outside
// 将Se0/0/0接口（连接外部ISP）定义为NAT的外部接口。
// 为什么这样做：路由器需要知道哪些接口是外部的，其流量是从外部进入内部或从内部出去的。

NATDynamic(config-if)#exit
NATDynamic(config)#exit
NATDynamic#write memory
// 保存配置
```

------

### 第三部分：配置路由

为了实现全网互通，我们需要在路由器上配置适当的路由。

**1. NATDynamic 路由器路由配置：**

```
NATDynamic>enable
NATDynamic#configure terminal
NATDynamic(config)#ip route 0.0.0.0 0.0.0.0 Se0/0/0
// 配置默认路由，所有未知目的地的流量都从Se0/0/0接口发送。
// 为什么这样做：对于内部网络来说，除了自己的私有网络外，所有外部网络的流量都应该发给ISP路由器，默认路由是最简单高效的方式。

NATDynamic(config)#exit
NATDynamic#write memory
```

**2. ISP 路由器路由配置：**

```
ISP>enable
ISP#configure terminal
ISP(config)#ip route 10.136.10.0 255.255.255.0 125.77.120.61
// 配置静态路由，指向内部网络10.136.10.0/24，下一跳是NATDynamic的Se0/0/0接口IP地址。
// 为什么这样做：ISP路由器需要知道如何将流量路由回内部私有网络，否则它无法将外部响应发送回内部设备。

ISP(config)#exit
ISP#write memory
```

------

### 第四部分：测试连通性

1. **从PC A/B/C Ping Server S：**
   - 在PC A/B/C的命令提示符中，尝试ping Server S的IP地址：`ping 121.41.74.100`
   - 如果NAT和路由配置正确，你应该能看到成功的回复。
2. **从Server S Ping PC A/B/C：**
   - 在Server S的命令提示符中，尝试ping PC A/B/C的内部IP地址：`ping 10.136.10.1` (这个应该会失败，因为Server S不知道私有地址)
   - **重要提示：** 从Server S是无法直接ping到内部私有IP地址的，因为NAT是单向的（从内到外）。如果想实现从外部访问内部服务，需要配置静态NAT或端口映射。这个实验只涉及动态NAT。
3. **从NATDynamic路由器Ping各接口和对端：**
   - `ping 10.136.10.1` (应该通)
   - `ping 125.77.120.62` (应该通)
   - `ping 121.41.74.254` (应该通)
   - `ping 121.41.74.100` (应该通)

------

### 第五部分：查看地址转换结果及地址池配置

**在NATDynamic路由器上查看：**

1. **查看NAT转换表：**

   ```
   NATDynamic#show ip nat translations
   ```

   - 预期输出：

      当PC A/B/C发起与Server S的连接后，你会看到类似以下内容的条目：

     ```
     Pro Inside global      Inside local       Outside local      Outside global
     icmp  125.77.120.65    10.136.10.1        121.41.74.100      121.41.74.100
     ```

     - `Inside global`: 这是NAT转换后分配的公共IP地址 (从NAT_POOL中分配)。
     - `Inside local`: 这是内部私有IP地址。
     - `Outside local`: 这是外部目的地的私有IP地址（如果外部也用了NAT）。
     - `Outside global`: 这是外部目的地的公共IP地址。
     - **为什么这样做：** 这条命令让你能够实时监控NAT的转换情况，看到哪个内部IP被转换成了哪个公共IP。

2. **查看NAT统计信息和地址池：**

   ```
   NATDynamic#show ip nat statistics
   ```

   - 预期输出：

      会显示NAT的详细统计信息，包括配置的地址池（如

     ```
     NAT_POOL
     ```

     ）以及当前使用情况。

     ```
     Total translations: 1 (1 static, 0 dynamic, 0 extended)
     ...
     NAT Pool NAT_POOL:
       Start IP: 125.77.120.65, End IP: 125.77.120.70
       Total addresses: 6, Allocated addresses: 1, Misses: 0
     ```

     - **为什么这样做：** 这条命令提供了NAT配置的概览，包括地址池的大小、有多少地址正在被使用，以及NAT的总翻译次数等，有助于排查问题和了解NAT的运行状态。

------

**总结**

这个实验通过配置动态NAT，实现了内部私有网络与外部公共网络之间的通信。关键在于：

- **私有地址与公共地址的区分：** 内部使用私有地址，外部使用公共地址。
- **NAT的inside和outside接口：** 明确指定哪些接口是内部的，哪些是外部的，路由器才能正确进行地址转换。
- **NAT地址池：** 为内部设备提供公共IP地址进行转换。
- **ACL：** 决定哪些内部流量可以进行NAT。
- **路由配置：** 确保内外网络之间的流量能够正确地转发。

通过`show ip nat translations`和`show ip nat statistics`命令，我们可以验证NAT配置是否生效以及地址池的使用情况。