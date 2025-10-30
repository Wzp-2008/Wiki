# Java版协议/数据包

本文详细剖析了当前Java版Minecraft协议，对应1.21.8版本，协议版本号为772。

不同版本之间的协议变化可以在协议历史中查看。

## 定义

Minecraft服务器接受来自TCP客户端的连接，并使用数据包与它们通信。数据包是通过TCP连接发送的字节序列。数据包的含义取决于其数据包ID和连接的当前状态。每个连接的初始状态是握手，状态通过握手和登录成功数据包进行切换。

### 数据类型

（数据类型的内容在此处转录——如需编辑请前往该页面）

### 其他定义

| 术语 Term | 定义 Definition |
|------|------|
| 玩家 Player | 当单独使用时，玩家总是指连接到服务器的客户端。 |
| 实体 Entity | 实体指任何物品、玩家、生物、矿车或船等。完整列表请参阅Minecraft Wiki文章。 |
| EID | EID（或实体ID Entity ID）是用于识别特定实体的4字节序列。实体的EID在整个服务器上是唯一的。 |
| XYZ | 在本文档中，轴名称与调试屏幕（F3）中显示的相同。Y指向上方，X指向东方，Z指向南方。 |
| 米 Meter | 米是Minecraft的基本长度单位，等于实心方块顶点的长度。术语"方块 block"可用于表示"米"或"立方米"。 |
| 注册表 Registry | 描述某种静态的、与游戏玩法相关的对象的表，例如实体类型、方块状态或生物群系。注册表的条目通常与文本或数字标识符相关联，或两者兼有。<br/><br/>Minecraft有一个统一的注册表系统，用于实现大多数注册表，包括方块、物品、实体、生物群系和维度。这些"普通"注册表将条目与命名空间文本标识符和有符号（正）32位数字标识符相关联。还有一个注册表的注册表，列出注册表系统中的所有注册表。其他一些注册表，最著名的是方块状态注册表 block state registry，以更临时的方式实现。<br/><br/>某些注册表（如生物群系和维度）可以由服务器在运行时自定义（参见注册表数据 Registry Data），而其他注册表（如方块、物品和实体）是硬编码的。硬编码注册表的内容可以通过内置的数据生成器 Data Generators 系统提取。 |
| 方块状态 Block state | Minecraft中的每个方块都有0个或多个属性 properties，这些属性可以有任意数量的可能值。这些表示例如方块的方向、红石组件的通电状态等。方块的属性值的每个可能排列都是一个不同的方块状态。方块状态注册表为每个方块的每个方块状态分配一个数字标识符。<br/><br/>当前的属性和状态ID范围列表可以在burger上找到。<br/><br/>或者，原版服务器现在包含一个选项，通过运行 `java -DbundlerMainClass=net.minecraft.data.Main -jar minecraft_server.jar --reports` 来导出当前的方块状态ID映射。有关更多信息，请参阅数据生成器 Data Generators。 |
| 原版 Vanilla | 由Mojang开发和发布的Minecraft官方实现。 |
| 序列 Sequence | 本地方块更改的动作编号计数器，当用手点击方块、右键点击物品或开始或完成挖掘方块时递增1。计数器处理延迟以避免将过时的方块更改应用于本地世界。它还用于恢复放置方块、使用桶或破坏方块时创建的幽灵方块 ghost blocks。 |

## 数据包格式

数据包不能大于2²¹ − 1或2097151字节（可以在3字节VarInt中发送的最大值）。此外，长度字段不得超过3字节，即使编码值在限制范围内。仍然允许3字节或更少的不必要的长编码。对于压缩的数据包，这适用于数据包长度字段，即压缩长度。

### 无压缩

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|----------|------|
| 长度 Length | VarInt | 数据包ID Packet ID + 数据 Data 的长度 |
| 数据包ID Packet ID | VarInt | 对应于服务器数据包报告中的 `protocol_id` |
| 数据 Data | 字节数组 Byte Array | 取决于连接状态和数据包ID，请参阅下面的部分 |

### 有压缩

一旦发送设置压缩 Set Compression 数据包（具有非负阈值 threshold），就会为所有后续数据包启用zlib压缩。数据包的格式稍有变化，以包括未压缩数据包的大小。

| 存在？ Present? | 压缩？ Compressed? | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|--------|--------|----------|----------|------|
| 总是 always | 否 No | 数据包长度 Packet Length | VarInt | （数据长度 Data Length）的长度 + 压缩的（数据包ID Packet ID + 数据 Data）的长度 |
| 如果大小 >= 阈值 if size >= threshold | 否 No | 数据长度 Data Length | VarInt | 未压缩的（数据包ID Packet ID + 数据 Data）的长度 |
| 如果大小 >= 阈值 if size >= threshold | 是 Yes | 数据包ID Packet ID | VarInt | zlib压缩的数据包ID（请参阅下面的部分） |
| 如果大小 >= 阈值 if size >= threshold | 是 Yes | 数据 Data | 字节数组 Byte Array | zlib压缩的数据包数据（请参阅下面的部分） |
| 如果大小 < 阈值 if size < threshold | 否 No | 数据长度 Data Length | VarInt | 0表示未压缩 |
| 如果大小 < 阈值 if size < threshold | 否 No | 数据包ID Packet ID | VarInt | 数据包ID（请参阅下面的部分） |
| 如果大小 < 阈值 if size < threshold | 否 No | 数据 Data | 字节数组 Byte Array | 数据包数据（请参阅下面的部分） |

对于服务器绑定数据包，（数据包ID + 数据）的未压缩长度不得大于2²³或8388608字节。请注意，允许长度等于2²³，这与压缩长度限制不同。另一方面，原版客户端对传入压缩数据包的未压缩长度没有限制。

如果包含数据包数据和ID（作为VarInt）的缓冲区大小小于数据包设置压缩中指定的阈值，它将作为未压缩发送。这是通过将数据长度设置为0来完成的（可与在长度和数据包数据之间发送额外的0的非压缩格式相比）。

如果它大于或等于阈值，则遵循常规压缩协议格式。

原版服务器（但不是客户端）拒绝小于阈值的压缩数据包。但是，接受超过阈值的未压缩数据包。

可以通过发送具有负阈值的数据包设置压缩来禁用压缩，或者根本不发送设置压缩数据包。

## 握手 Handshaking

### 客户端绑定 Clientbound

握手状态中没有客户端绑定数据包，因为在客户端发送第一个数据包后，协议立即切换到不同的状态。

### 服务器绑定 Serverbound

#### 握手 Handshake

此数据包使服务器切换到目标状态。在打开TCP连接后应立即发送它，以防止服务器断开连接。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x00`<br/>`intention` | 握手 Handshaking | 服务器 Server | 协议版本 Protocol Version | VarInt | 请参阅协议版本号（当前在Minecraft 1.21.8中为772）。 |
| `0x00`<br/>`intention` | 握手 Handshaking | 服务器 Server | 服务器地址 Server Address | 字符串 String (255) | 用于连接的主机名或IP，例如localhost或127.0.0.1。原版服务器不使用此信息。请注意，SRV记录是一个简单的重定向，例如，如果_minecraft._tcp.example.com指向mc.example.org，则连接到example.com的用户除了连接到它之外，还将提供example.org作为服务器地址。 |
| `0x00`<br/>`intention` | 握手 Handshaking | 服务器 Server | 服务器端口 Server Port | 无符号短整型 Unsigned Short | 默认值为25565。原版服务器不使用此信息。 |
| `0x00`<br/>`intention` | 握手 Handshaking | 服务器 Server | 意图 Intent | VarInt 枚举 Enum | 1表示状态 Status，2表示登录 Login，3表示传输 Transfer。 |

#### 旧版服务器列表Ping Legacy Server List Ping

**警告：** 此数据包使用非标准格式。它从不带长度前缀，并且数据包ID是无符号字节 Unsigned Byte 而不是VarInt。

虽然从技术上讲不是当前协议的一部分，但（旧版）客户端可能会发送此数据包以启动服务器列表Ping，现代服务器应正确处理它。此数据包的格式是pre-Netty时代的遗留物，在1.7中切换到Netty之前，带来了现在认可的标准格式。此数据包仅用于通知旧版客户端它们无法加入我们的现代服务器。

现代客户端（在1.21.5 + 1.21.4中测试）在服务器在30秒时间窗口内不发送任何响应或连接立即关闭时也会发送此数据包。

**警告：** 客户端不会自行关闭与旧版数据包的连接！它只会在Minecraft客户端关闭时关闭。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| 0xFE | 握手 Handshaking | 服务器 Server | 有效载荷 Payload | 无符号字节 Unsigned Byte | 总是1（`0x01`）。 |

有关此数据包之后的协议详细信息，请参阅服务器列表Ping#1.6。

## 状态 Status

### 客户端绑定 Clientbound

#### 状态响应 Status Response

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x00`<br/>`status_response` | 状态 Status | 客户端 Client | JSON响应 JSON Response | 字符串 String (32767) | 请参阅服务器列表Ping#状态响应；与所有字符串一样，此字符串的前缀是其长度作为VarInt。 |

#### Pong响应（状态） Pong Response (status)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x01`<br/>`pong_response` | 状态 Status | 客户端 Client | 时间戳 Timestamp | 长整型 Long | 应该与客户端发送的匹配。 |

### 服务器绑定 Serverbound

#### 状态请求 Status Request

只能在握手后立即请求一次状态，在任何ping之前。否则服务器不会响应。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x00`<br/>`status_request` | 状态 Status | 服务器 Server | | | 无字段 no fields |

#### Ping请求（状态） Ping Request (status)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x01`<br/>`ping_request` | 状态 Status | 服务器 Server | 时间戳 Timestamp | 长整型 Long | 可以是任何数字，但原版客户端将始终使用以毫秒为单位的时间戳。 |

---

**翻译进度：第1-3部分（介绍、定义、数据包格式、握手、状态）- 已完成**

**待翻译章节：**
- 登录（Login）
- 配置（Configuration）
- 游戏（Play）
- 导航（Navigation）
