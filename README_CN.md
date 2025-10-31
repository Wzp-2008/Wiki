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

## 登录 Login

登录过程如下：

1. C→S：握手 Handshake，意图设置为2（登录 login）
2. C→S：登录开始 Login Start
3. S→C：加密请求 Encryption Request
4. 客户端身份验证 Client auth（如果启用）
5. C→S：加密响应 Encryption Response
6. 服务器身份验证 Server auth（如果启用）
7. 双方启用加密 Both enable encryption
8. S→C：设置压缩 Set Compression（可选 optional）
9. S→C：登录成功 Login Success
10. C→S：登录确认 Login Acknowledged

设置压缩 Set Compression（如果存在）必须在登录成功 Login Success 之前发送。请注意，在设置压缩之后发送的任何内容都必须使用压缩后数据包格式 Post Compression packet format。

根据数据包的发送方式，可能有三种操作模式：
- 带加密的在线模式 Online-mode with encryption
- 带加密的离线模式 Offline-mode with encryption
- 不带加密的离线模式 Offline-mode without encryption

对于在线模式服务器（启用了身份验证的服务器），加密始终是强制性的，需要遵循上述整个过程。

对于离线模式服务器（禁用身份验证的服务器），加密是可选的，可以跳过部分过程。在这种情况下，登录开始 Login Start 直接后跟登录成功 Login Success。原版服务器仅将UUID v3用于离线玩家UUID，从字符串 `OfflinePlayer:<player's name>` 派生。例如，Notch的离线UUID将从字符串 `OfflinePlayer:Notch` 中选择。但是，这不是要求，UUID可以设置为任何内容。

从1.21开始，原版服务器在离线模式下从不使用加密。

有关详细信息，请参阅协议加密 protocol encryption。

### 客户端绑定 Clientbound

#### 断开连接（登录） Disconnect (login)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x00`<br/>`login_disconnect` | 登录 Login | 客户端 Client | 原因 Reason | JSON文本组件 JSON Text Component | 玩家被断开连接的原因。 |

#### 加密请求 Encryption Request

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x01`<br/>`hello` | 登录 Login | 客户端 Client | 服务器ID Server ID | 字符串 String (20) | 原版服务器发送时始终为空。 |
| `0x01`<br/>`hello` | 登录 Login | 客户端 Client | 公钥 Public Key | 字节前缀数组 Prefixed Array of Byte | 服务器的公钥（以字节为单位）。 |
| `0x01`<br/>`hello` | 登录 Login | 客户端 Client | 验证令牌 Verify Token | 字节前缀数组 Prefixed Array of Byte | 服务器生成的随机字节序列。 |
| `0x01`<br/>`hello` | 登录 Login | 客户端 Client | 应该身份验证 Should authenticate | 布尔值 Boolean | 客户端是否应该尝试通过mojang服务器进行身份验证。 |

有关详细信息，请参阅协议加密 protocol encryption。

#### 登录成功 Login Success

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x02`<br/>`login_finished` | 登录 Login | 客户端 Client | 个人资料 Profile | 游戏配置文件 Game Profile | |

#### 设置压缩 Set Compression

启用压缩。如果启用了压缩，则所有后续数据包都以压缩数据包格式 compressed packet format 编码。负值将禁用压缩，这意味着数据包格式应保持未压缩数据包格式 uncompressed packet format。但是，此数据包是完全可选的，如果不发送，也不会启用压缩（当禁用压缩时，原版服务器不会发送数据包）。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x03`<br/>`login_compression` | 登录 Login | 客户端 Client | 阈值 Threshold | VarInt | 压缩前数据包的最大大小。 |

#### 登录插件请求 Login Plugin Request

与登录插件响应 Login Plugin Response 一起用于实现自定义握手流程。

与"游戏 play"模式下的插件消息不同，这些消息遵循锁步请求/响应方案，其中客户端应该响应请求，指示它是否理解。原版客户端总是响应它没有理解并发送一个空的有效载荷。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x04`<br/>`custom_query` | 登录 Login | 客户端 Client | 消息ID Message ID | VarInt | 由服务器生成 - 对连接应该是唯一的。 |
| `0x04`<br/>`custom_query` | 登录 Login | 客户端 Client | 频道 Channel | 标识符 Identifier | 用于发送数据的插件频道 plugin channel 的名称。 |
| `0x04`<br/>`custom_query` | 登录 Login | 客户端 Client | 数据 Data | 字节数组 Byte Array (1048576) | 任何数据，取决于频道。必须从数据包长度推断此数组的长度。 |

#### Cookie请求（登录） Cookie Request (login)

请求先前存储的cookie。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x05`<br/>`cookie_request` | 登录 Login | 客户端 Client | 键 Key | 标识符 Identifier | cookie的标识符。 |

### 服务器绑定 Serverbound

#### 登录开始 Login Start

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x00`<br/>`hello` | 登录 Login | 服务器 Server | 名称 Name | 字符串 String (16) | 玩家的用户名 Player's Username。 |
| `0x00`<br/>`hello` | 登录 Login | 服务器 Server | 玩家UUID Player UUID | UUID | 登录玩家的UUID。原版服务器未使用。 |

#### 加密响应 Encryption Response

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x01`<br/>`key` | 登录 Login | 服务器 Server | 共享密钥 Shared Secret | 字节前缀数组 Prefixed Array of Byte | 使用服务器的公钥加密的共享密钥值。 |
| `0x01`<br/>`key` | 登录 Login | 服务器 Server | 验证令牌 Verify Token | 字节前缀数组 Prefixed Array of Byte | 使用与共享密钥相同的公钥加密的验证令牌值。 |

有关详细信息，请参阅协议加密 protocol encryption。

#### 登录插件响应 Login Plugin Response

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x02`<br/>`custom_query_answer` | 登录 Login | 服务器 Server | 消息ID Message ID | VarInt | 应该与服务器的ID匹配。 |
| `0x02`<br/>`custom_query_answer` | 登录 Login | 服务器 Server | 数据 Data | 可选字节前缀数组 Prefixed Optional Byte Array (1048576) | 任何数据，取决于频道。必须从数据包长度推断此数组的长度。仅当客户端理解请求时才存在。 |

#### 登录确认 Login Acknowledged

对服务器发送的登录成功 Login Success 数据包的确认。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x03`<br/>`login_acknowledged` | 登录 Login | 服务器 Server | | | 无字段 no fields |

此数据包将连接状态切换到配置 configuration。

#### Cookie响应（登录） Cookie Response (login)

对服务器的Cookie请求（登录） Cookie Request (login) 的响应。原版服务器仅接受最大5 kiB的响应。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x04`<br/>`cookie_response` | 登录 Login | 服务器 Server | 键 Key | 标识符 Identifier | cookie的标识符。 |
| `0x04`<br/>`cookie_response` | 登录 Login | 服务器 Server | 有效载荷 Payload | 可选字节前缀数组 Prefixed Optional Prefixed Array (5120) of Byte | cookie的数据。 |

## 配置 Configuration

### 客户端绑定 Clientbound

#### Cookie请求（配置） Cookie Request (configuration)

请求先前存储的cookie。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x00`<br/>`cookie_request` | 配置 Configuration | 客户端 Client | 键 Key | 标识符 Identifier | cookie的标识符。 |

#### 客户端绑定插件消息（配置） Clientbound Plugin Message (configuration)

模组和插件可以使用它来发送他们的数据。Minecraft本身使用几个插件频道 plugin channels。这些内部频道位于 `minecraft` 命名空间中。

有关其工作原理的更多信息，请访问Dinnerbone的博客。有关内部和流行注册频道的更多文档在这里。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x01`<br/>`custom_payload` | 配置 Configuration | 客户端 Client | 频道 Channel | 标识符 Identifier | 用于发送数据的插件频道 plugin channel 的名称。 |
| `0x01`<br/>`custom_payload` | 配置 Configuration | 客户端 Client | 数据 Data | 字节数组 Byte Array (1048576) | 任何数据。必须从数据包长度推断此数组的长度。 |

在原版客户端中，最大数据长度为1048576字节。

#### 断开连接（配置） Disconnect (configuration)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x02`<br/>`disconnect` | 配置 Configuration | 客户端 Client | 原因 Reason | 文本组件 Text Component | 玩家被断开连接的原因。 |

#### 完成配置 Finish Configuration

由服务器发送以通知客户端配置过程已完成。客户端在准备好继续时会回复确认完成配置 Acknowledge Finish Configuration。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x03`<br/>`finish_configuration` | 配置 Configuration | 客户端 Client | | | 无字段 no fields |

此数据包将连接状态切换到游戏 play。

#### 客户端绑定保持连接（配置） Clientbound Keep Alive (configuration)

服务器将经常发送保持连接 keep-alive，每个包含一个随机ID。客户端必须以相同的有效载荷 payload 响应（请参阅服务器绑定保持连接 Serverbound Keep Alive）。如果客户端在发送后15秒内未响应保持连接数据包，则服务器会踢出客户端。相反，如果服务器在20秒内不发送任何保持连接，客户端将断开连接并产生"超时 Timed out"异常。

原版服务器使用系统相关的时间（以毫秒为单位）来生成保持连接ID值。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x04`<br/>`keep_alive` | 配置 Configuration | 客户端 Client | 保持连接ID Keep Alive ID | 长整型 Long | |

#### Ping（配置） Ping (configuration)

原版服务器不使用此数据包。当发送到客户端时，客户端会用具有相同ID的Pong数据包响应。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x05`<br/>`ping` | 配置 Configuration | 客户端 Client | ID | 整型 Int | |

#### 重置聊天 Reset Chat

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x06`<br/>`reset_chat` | 配置 Configuration | 客户端 Client | | | 无字段 no fields |

#### 注册表数据 Registry Data

表示从服务器发送并应用于客户端的某些注册表 registries。

有关详细信息，请参阅注册表数据 Registry Data。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x07`<br/>`registry_data` | 配置 Configuration | 客户端 Client | 注册表ID Registry ID | 标识符 Identifier | |
| `0x07`<br/>`registry_data` | 配置 Configuration | 客户端 Client | 条目 Entries - 条目ID Entry ID | 前缀数组 Prefixed Array - 标识符 Identifier | |
| `0x07`<br/>`registry_data` | 配置 Configuration | 客户端 Client | 条目 Entries - 数据 Data | 前缀数组 Prefixed Array - 可选NBT Prefixed Optional NBT | 条目数据 Entry data。 |

#### 移除资源包（配置） Remove Resource Pack (configuration)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x08`<br/>`resource_pack_pop` | 配置 Configuration | 客户端 Client | UUID | 可选UUID Prefixed Optional UUID | 要移除的资源包的UUID。如果不存在，将移除每个资源包。 |

#### 添加资源包（配置） Add Resource Pack (configuration)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x09`<br/>`resource_pack_push` | 配置 Configuration | 客户端 Client | UUID | UUID | 资源包的唯一标识符。 |
| `0x09`<br/>`resource_pack_push` | 配置 Configuration | 客户端 Client | URL | 字符串 String (32767) | 资源包的URL。 |
| `0x09`<br/>`resource_pack_push` | 配置 Configuration | 客户端 Client | 哈希 Hash | 字符串 String (40) | 资源包文件的40个字符的十六进制、不区分大小写的SHA-1哈希。如果不是40个字符的十六进制字符串，客户端将不会将其用于哈希验证，并且可能会浪费带宽。 |
| `0x09`<br/>`resource_pack_push` | 配置 Configuration | 客户端 Client | 强制 Forced | 布尔值 Boolean | 原版客户端将被强制使用服务器的资源包。如果他们拒绝，他们将被踢出服务器。 |
| `0x09`<br/>`resource_pack_push` | 配置 Configuration | 客户端 Client | 提示消息 Prompt Message | 可选文本组件 Prefixed Optional Text Component | 这显示在提示中，使客户端接受或拒绝资源包（仅在存在时）。 |

#### 存储Cookie（配置） Store Cookie (configuration)

在客户端上存储一些任意数据，这些数据在服务器传输之间保持不变。原版客户端仅接受最大5 kiB的cookie。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x0A`<br/>`store_cookie` | 配置 Configuration | 客户端 Client | 键 Key | 标识符 Identifier | cookie的标识符。 |
| `0x0A`<br/>`store_cookie` | 配置 Configuration | 客户端 Client | 有效载荷 Payload | 字节前缀数组 Prefixed Array (5120) of Byte | cookie的数据。 |

#### 传输（配置） Transfer (configuration)

通知客户端应该传输到给定的服务器。先前存储的Cookie在服务器传输之间保留。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x0B`<br/>`transfer` | 配置 Configuration | 客户端 Client | 主机 Host | 字符串 String (32767) | 服务器的主机名或IP。 |
| `0x0B`<br/>`transfer` | 配置 Configuration | 客户端 Client | 端口 Port | VarInt | 服务器的端口。 |

#### 功能标志 Feature Flags

用于在客户端上启用和禁用功能，通常是实验性功能。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x0C`<br/>`update_enabled_features` | 配置 Configuration | 客户端 Client | 功能标志 Feature Flags | 标识符前缀数组 Prefixed Array of Identifier | |

有一个特殊的功能标志，在大多数版本中都存在：
- minecraft:vanilla - 启用原版功能

对于其他可能在版本之间更改的功能标志，请参阅实验#Java版 Experiments#Java_Edition。

#### 更新标签（配置） Update Tags (configuration)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x0D`<br/>`update_tags` | 配置 Configuration | 客户端 Client | 标签数组 Array of tags - 注册表 Registry | 前缀数组 Prefixed Array - 标识符 Identifier | 注册表标识符（原版期望 `minecraft:block`、`minecraft:item`、`minecraft:fluid`、`minecraft:entity_type` 和 `minecraft:game_event` 注册表的标签） |
| `0x0D`<br/>`update_tags` | 配置 Configuration | 客户端 Client | 标签数组 Array of tags - 标签数组 Array of Tag | 前缀数组 Prefixed Array | （见下文） |

标签数组 Tag arrays 如下所示：

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|----------|------|
| 标签 Tags - 标签名称 Tag name | 前缀数组 Prefixed Array - 标识符 Identifier | |
| 标签 Tags - 条目 Entries | 前缀数组 Prefixed Array - VarInt前缀数组 Prefixed Array of VarInt | 给定类型的数字ID（方块 block、物品 item等）。此列表替换给定标签的先前ID列表。如果一些预先存在的标签未提及，则会打印警告。 |

有关更多信息（包括原版标签列表），请参阅Minecraft Wiki上的标签 Tag。

#### 客户端绑定已知包 Clientbound Known Packs

通知客户端服务器上存在哪些数据包 data packs。客户端应该用自己的服务器绑定已知包 Serverbound Known Packs 数据包响应。原版服务器在收到响应之前不会继续配置。

原版客户端需要版本为 `1.21.8` 的 `minecraft:core` 包才能进行正常的登录序列。此数据包必须在注册表数据 Registry Data 数据包之前发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x0E`<br/>`select_known_packs` | 配置 Configuration | 客户端 Client | 已知包 Known Packs - 命名空间 Namespace | 前缀数组 Prefixed Array - 字符串 String (32767) | |
| `0x0E`<br/>`select_known_packs` | 配置 Configuration | 客户端 Client | 已知包 Known Packs - ID | 前缀数组 Prefixed Array - 字符串 String (32767) | |
| `0x0E`<br/>`select_known_packs` | 配置 Configuration | 客户端 Client | 已知包 Known Packs - 版本 Version | 前缀数组 Prefixed Array - 字符串 String (32767) | |

#### 自定义报告详细信息（配置） Custom Report Details (configuration)

包含在连接到服务器期间生成的任何崩溃或断开连接报告中包含的键值文本条目列表。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x0F`<br/>`custom_report_details` | 配置 Configuration | 客户端 Client | 详细信息 Details - 标题 Title | 前缀数组 Prefixed Array (32) - 字符串 String (128) | |
| `0x0F`<br/>`custom_report_details` | 配置 Configuration | 客户端 Client | 详细信息 Details - 描述 Description | 前缀数组 Prefixed Array (32) - 字符串 String (4096) | |

#### 服务器链接（配置） Server Links (configuration)

此数据包包含原版客户端将在暂停菜单中显示的链接列表。链接标签可以是内置的或自定义的（即任何文本）。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x10`<br/>`server_links` | 配置 Configuration | 客户端 Client | 链接 Links - 标签 Label | 前缀数组 Prefixed Array - VarInt枚举 VarInt Enum 或 or 文本组件 Text Component | 枚举用于内置标签（见下文），文本组件用于自定义标签。 |
| `0x10`<br/>`server_links` | 配置 Configuration | 客户端 Client | 链接 Links - URL | 前缀数组 Prefixed Array - 字符串 String | 有效的URL。 |

| ID | 名称 Name | 说明 Notes |
|----|------|------|
| 0 | 错误报告 Bug Report | 在连接错误屏幕上显示；作为断开连接报告中的注释包含。 |
| 1 | 社区准则 Community Guidelines | |
| 2 | 支持 Support | |
| 3 | 状态 Status | |
| 4 | 反馈 Feedback | |
| 5 | 社区 Community | |
| 6 | 网站 Website | |
| 7 | 论坛 Forums | |
| 8 | 新闻 News | |
| 9 | 公告 Announcements | |

#### 清除对话框（配置） Clear Dialog (configuration)

如果我们当前在对话框屏幕中，则这将删除当前屏幕并切换回上一个屏幕。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x11`<br/>`clear_dialog` | 配置 Configuration | 客户端 Client | | | 无字段 no fields |

#### 显示对话框（配置） Show Dialog (configuration)

向客户端显示自定义对话框屏幕。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x12`<br/>`show_dialog` | 配置 Configuration | 客户端 Client | 对话框 Dialog | NBT | 内联定义，如注册表数据#对话框 Registry_data#Dialog 中所述。 |

### 服务器绑定 Serverbound

#### 客户端信息（配置） Client Information (configuration)

在玩家连接或更改设置时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x00`<br/>`client_information` | 配置 Configuration | 服务器 Server | 区域设置 Locale | 字符串 String (16) | 例如 `en_GB`。 |
| `0x00`<br/>`client_information` | 配置 Configuration | 服务器 Server | 视距 View Distance | 字节 Byte | 客户端渲染距离，以区块 chunks 为单位。 |
| `0x00`<br/>`client_information` | 配置 Configuration | 服务器 Server | 聊天模式 Chat Mode | VarInt枚举 VarInt Enum | 0：启用，1：仅命令，2：隐藏。有关更多信息，请参阅聊天#客户端聊天模式 Chat#Client chat mode。 |
| `0x00`<br/>`client_information` | 配置 Configuration | 服务器 Server | 聊天颜色 Chat Colors | 布尔值 Boolean | "颜色 Colors"多人游戏设置。原版服务器存储此值但不执行任何操作（请参阅MC-64867）。当它为false时，某些第三方服务器会禁用聊天和系统消息中的所有着色。 |
| `0x00`<br/>`client_information` | 配置 Configuration | 服务器 Server | 显示的皮肤部位 Displayed Skin Parts | 无符号字节 Unsigned Byte | 位掩码 Bit mask，见下文。 |
| `0x00`<br/>`client_information` | 配置 Configuration | 服务器 Server | 主手 Main Hand | VarInt枚举 VarInt Enum | 0：左，1：右。 |
| `0x00`<br/>`client_information` | 配置 Configuration | 服务器 Server | 启用文本过滤 Enable text filtering | 布尔值 Boolean | 启用对标牌和书面书籍标题上文本的过滤。原版客户端根据Mojang API的 `/player/attributes` 端点指示的 `profanityFilterPreferences.profanityFilterOn` 帐户属性设置此项。在离线模式下，它始终为false。 |
| `0x00`<br/>`client_information` | 配置 Configuration | 服务器 Server | 允许服务器列表 Allow server listings | 布尔值 Boolean | 服务器通常列出在线玩家；此选项应该让您不出现在该列表中。 |
| `0x00`<br/>`client_information` | 配置 Configuration | 服务器 Server | 粒子状态 Particle Status | VarInt枚举 VarInt Enum | 0：全部 all，1：减少 decreased，2：最小 minimal |

显示的皮肤部位 Displayed Skin Parts 标志：

- 位0 Bit 0（0x01）：披风启用 Cape enabled
- 位1 Bit 1（0x02）：夹克启用 Jacket enabled
- 位2 Bit 2（0x04）：左袖启用 Left Sleeve enabled
- 位3 Bit 3（0x08）：右袖启用 Right Sleeve enabled
- 位4 Bit 4（0x10）：左裤腿启用 Left Pants Leg enabled
- 位5 Bit 5（0x20）：右裤腿启用 Right Pants Leg enabled
- 位6 Bit 6（0x40）：帽子启用 Hat enabled

最高有效位（位7 bit 7，0x80）似乎未使用。

#### Cookie响应（配置） Cookie Response (configuration)

对服务器的Cookie请求（配置） Cookie Request (configuration) 的响应。原版服务器仅接受最大5 kiB的响应。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x01`<br/>`cookie_response` | 配置 Configuration | 服务器 Server | 键 Key | 标识符 Identifier | cookie的标识符。 |
| `0x01`<br/>`cookie_response` | 配置 Configuration | 服务器 Server | 有效载荷 Payload | 可选字节前缀数组 Prefixed Optional Prefixed Array (5120) of Byte | cookie的数据。 |

#### 服务器绑定插件消息（配置） Serverbound Plugin Message (configuration)

模组和插件可以使用它来发送他们的数据。Minecraft本身使用一些插件频道 plugin channels。这些内部频道位于 `minecraft` 命名空间中。

有关更多文档，请参阅此处。

请注意，数据 Data 的长度仅从数据包长度已知，因为数据包没有任何类型的长度字段。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x02`<br/>`custom_payload` | 配置 Configuration | 服务器 Server | 频道 Channel | 标识符 Identifier | 用于发送数据的插件频道 plugin channel 的名称。 |
| `0x02`<br/>`custom_payload` | 配置 Configuration | 服务器 Server | 数据 Data | 字节数组 Byte Array (32767) | 任何数据，取决于频道。 `minecraft:` 频道在此处记录。必须从数据包长度推断此数组的长度。 |

在原版服务器中，最大数据长度为32767字节。

#### 确认完成配置 Acknowledge Finish Configuration

由客户端发送以通知服务器配置过程已完成。它是对服务器的完成配置 Finish Configuration 的响应。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x03`<br/>`finish_configuration` | 配置 Configuration | 服务器 Server | | | 无字段 no fields |

此数据包将连接状态切换到游戏 play。

#### 服务器绑定保持连接（配置） Serverbound Keep Alive (configuration)

服务器将经常发送保持连接 keep-alive（请参阅客户端绑定保持连接 Clientbound Keep Alive），每个包含一个随机ID。客户端必须以相同的数据包响应。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x04`<br/>`keep_alive` | 配置 Configuration | 服务器 Server | 保持连接ID Keep Alive ID | 长整型 Long | |

#### Pong（配置） Pong (configuration)

对客户端绑定数据包（Ping）的响应，ID相同。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x05`<br/>`pong` | 配置 Configuration | 服务器 Server | ID | 整型 Int | |

#### 资源包响应（配置） Resource Pack Response (configuration)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x06`<br/>`resource_pack` | 配置 Configuration | 服务器 Server | UUID | UUID | 在添加资源包（配置） Add Resource Pack (configuration) 请求中收到的资源包的唯一标识符。 |
| `0x06`<br/>`resource_pack` | 配置 Configuration | 服务器 Server | 结果 Result | VarInt枚举 VarInt Enum | 结果ID（见下文）。 |

结果 Result 可以是以下值之一：

| ID | 结果 Result |
|----|------|
| 0 | 成功下载 Successfully downloaded |
| 1 | 拒绝 Declined |
| 2 | 下载失败 Failed to download |
| 3 | 接受 Accepted |
| 4 | 已下载 Downloaded |
| 5 | 无效的URL Invalid URL |
| 6 | 重新加载失败 Failed to reload |
| 7 | 已丢弃 Discarded |

#### 服务器绑定已知包 Serverbound Known Packs

通知服务器客户端上存在哪些数据包 data packs。客户端发送此响应客户端绑定已知包 Clientbound Known Packs。

如果客户端在此数据包中指定了一个包，则服务器应从注册表数据 Registry Data 数据包中省略其包含的数据。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x07`<br/>`select_known_packs` | 配置 Configuration | 服务器 Server | 已知包 Known Packs - 命名空间 Namespace | 前缀数组 Prefixed Array - 字符串 String | |
| `0x07`<br/>`select_known_packs` | 配置 Configuration | 服务器 Server | 已知包 Known Packs - ID | 前缀数组 Prefixed Array - 字符串 String | |
| `0x07`<br/>`select_known_packs` | 配置 Configuration | 服务器 Server | 已知包 Known Packs - 版本 Version | 前缀数组 Prefixed Array - 字符串 String | |

#### 自定义点击操作（配置） Custom Click Action (configuration)

当客户端点击具有 `minecraft:custom` 点击操作的文本组件 Text Component 时发送。这意味着作为运行命令的替代方案，但不会对原版服务器产生任何影响。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x08`<br/>`custom_click_action` | 配置 Configuration | 服务器 Server | ID | 标识符 Identifier | 点击操作的标识符。 |
| `0x08`<br/>`custom_click_action` | 配置 Configuration | 服务器 Server | 有效载荷 Payload | NBT | 要与点击操作一起发送的数据。可以是TAG_END（0）。 |

## 游戏 Play

### 客户端绑定 Clientbound

#### 捆绑分隔符 Bundle Delimiter

数据包捆绑的分隔符。收到后，客户端应该存储它收到的每个后续数据包，并等待收到另一个分隔符。一旦发生这种情况，客户端就可以保证在同一刻度 tick 上处理捆绑中的每个数据包，并且客户端应该停止存储数据包。

从1.20.6开始，原版服务器仅使用它来确保生成实体 Spawn Entity 和用于配置实体的关联数据包在同一刻度上发生。每个实体都有一个单独的捆绑。

原版客户端不允许在同一捆绑中超过4096个数据包。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x00`<br/>`bundle_delimiter` | 游戏 Play | 客户端 Client | | | 无字段 no fields |

#### 生成实体 Spawn Entity

由服务器发送以在客户端上创建实体，通常是在实体在玩家视野范围内生成或进入时。

本地玩家实体由客户端自动创建，不得使用此数据包显式创建。在原版客户端上这样做会产生奇怪的后果。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x01`<br/>`add_entity` | 游戏 Play | 客户端 Client | 实体ID Entity ID | VarInt | 一个唯一的整数ID，主要用于协议中识别实体。如果客户端上已存在具有相同ID的实体，则会自动删除它并替换为新实体。在原版服务器上，实体ID在所有维度中全局唯一，并且在服务器运行时永不重用，但不会在服务器重启之间保留。 |
| `0x01`<br/>`add_entity` | 游戏 Play | 客户端 Client | 实体UUID Entity UUID | UUID | 一个唯一的标识符，主要用于持久性和唯一性更重要的地方。可以在原版客户端上创建具有相同UUID的多个实体，但会记录警告，并且依赖于UUID的功能可能会忽略实体或以其他方式出现异常。 |
| `0x01`<br/>`add_entity` | 游戏 Play | 客户端 Client | 类型 Type | VarInt | `minecraft:entity_type` 注册表中的ID（请参阅实体元数据#实体 Entity metadata#Entities 中的"type"字段）。 |
| `0x01`<br/>`add_entity` | 游戏 Play | 客户端 Client | X | 双精度浮点型 Double | |
| `0x01`<br/>`add_entity` | 游戏 Play | 客户端 Client | Y | 双精度浮点型 Double | |
| `0x01`<br/>`add_entity` | 游戏 Play | 客户端 Client | Z | 双精度浮点型 Double | |
| `0x01`<br/>`add_entity` | 游戏 Play | 客户端 Client | 俯仰角 Pitch | 角度 Angle | |
| `0x01`<br/>`add_entity` | 游戏 Play | 客户端 Client | 偏航角 Yaw | 角度 Angle | |
| `0x01`<br/>`add_entity` | 游戏 Play | 客户端 Client | 头部偏航角 Head Yaw | 角度 Angle | 仅由生物实体 living entities 使用，其中实体的头部可能与一般身体旋转不同。 |
| `0x01`<br/>`add_entity` | 游戏 Play | 客户端 Client | 数据 Data | VarInt | 含义取决于类型 Type 字段的值，有关详细信息，请参阅对象数据 Object Data。 |
| `0x01`<br/>`add_entity` | 游戏 Play | 客户端 Client | 速度X Velocity X | 短整型 Short | 与设置实体速度 Set Entity Velocity 相同的单位。 |
| `0x01`<br/>`add_entity` | 游戏 Play | 客户端 Client | 速度Y Velocity Y | 短整型 Short | 与设置实体速度 Set Entity Velocity 相同的单位。 |
| `0x01`<br/>`add_entity` | 游戏 Play | 客户端 Client | 速度Z Velocity Z | 短整型 Short | 与设置实体速度 Set Entity Velocity 相同的单位。 |

**警告：** 当此数据包用于生成玩家实体时，应考虑以下几点。

在在线模式 online mode 下，UUID必须有效并具有有效的皮肤数据。在离线模式下，原版服务器使用UUID v3，并通过使用字符串 `OfflinePlayer:<player name>`（以UTF-8编码且区分大小写）选择玩家的UUID，然后使用 `UUID.nameUUIDFromBytes` 处理它。

对于NPC应使用UUID v2。注意：

> <+Grum> i will never confirm this as a feature you know that :)

在示例UUID `xxxxxxxx-xxxx-Yxxx-xxxx-xxxxxxxxxxxx` 中，UUID版本由 `Y` 指定。因此，对于UUID v3，`Y` 将始终为 `3`，对于UUID v2，`Y` 将始终为 `2`。

#### 实体动画 Entity Animation

在实体应该更改动画时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x02`<br/>`animate` | 游戏 Play | 客户端 Client | 实体ID Entity ID | VarInt | 玩家ID Player ID。 |
| `0x02`<br/>`animate` | 游戏 Play | 客户端 Client | 动画 Animation | 无符号字节 Unsigned Byte | 动画ID Animation ID（见下文）。 |

动画 Animation 可以是以下值之一：

| ID | 动画 Animation |
|----|------|
| 0 | 挥动主手 Swing main arm |
| 2 | 离开床 Leave bed |
| 3 | 挥动副手 Swing offhand |
| 4 | 暴击效果 Critical effect |
| 5 | 魔法暴击效果 Magic critical effect |

#### 授予统计信息 Award Statistics

作为对客户端状态 Client Status（id 1）的响应发送。如果之前请求过，只会发送更改的值。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x03`<br/>`award_stats` | 游戏 Play | 客户端 Client | 统计信息 Statistics - 类别ID Category ID | 前缀数组 Prefixed Array - VarInt | `minecraft:stat_type` 注册表中的ID；见下文。 |
| `0x03`<br/>`award_stats` | 游戏 Play | 客户端 Client | 统计信息 Statistics - 统计ID Statistic ID | 前缀数组 Prefixed Array - VarInt | 见下文。 |
| `0x03`<br/>`award_stats` | 游戏 Play | 客户端 Client | 统计信息 Statistics - 值 Value | 前缀数组 Prefixed Array - VarInt | 要设置的数量。 |

类别 Categories（在 `minecraft:stat_type` 注册表中定义）：

| 名称 Name | ID | 注册表 Registry |
|------|------|------|
| `minecraft:mined` | 0 | `minecraft:block` |
| `minecraft:crafted` | 1 | `minecraft:item` |
| `minecraft:used` | 2 | `minecraft:item` |
| `minecraft:broken` | 3 | `minecraft:item` |
| `minecraft:picked_up` | 4 | `minecraft:item` |
| `minecraft:dropped` | 5 | `minecraft:item` |
| `minecraft:killed` | 6 | `minecraft:entity_type` |
| `minecraft:killed_by` | 7 | `minecraft:entity_type` |
| `minecraft:custom` | 8 | `minecraft:custom_stat` |

方块 Blocks、物品 Items 和实体 Entities 使用方块（而不是方块状态 block state）、物品和实体ID。

自定义 Custom 使用 `minecraft:custom_stat` 注册表中的ID：

| 名称 Name | ID | 单位 Unit |
|------|------|------|
| `minecraft:leave_game` | 0 | 无 None |
| `minecraft:play_time` | 1 | 时间 Time |
| `minecraft:total_world_time` | 2 | 时间 Time |
| `minecraft:time_since_death` | 3 | 时间 Time |
| `minecraft:time_since_rest` | 4 | 时间 Time |
| `minecraft:sneak_time` | 5 | 时间 Time |
| `minecraft:walk_one_cm` | 6 | 距离 Distance |
| `minecraft:crouch_one_cm` | 7 | 距离 Distance |
| `minecraft:sprint_one_cm` | 8 | 距离 Distance |
| `minecraft:walk_on_water_one_cm` | 9 | 距离 Distance |
| `minecraft:fall_one_cm` | 10 | 距离 Distance |
| `minecraft:climb_one_cm` | 11 | 距离 Distance |
| `minecraft:fly_one_cm` | 12 | 距离 Distance |
| `minecraft:walk_under_water_one_cm` | 13 | 距离 Distance |
| `minecraft:minecart_one_cm` | 14 | 距离 Distance |
| `minecraft:boat_one_cm` | 15 | 距离 Distance |
| `minecraft:pig_one_cm` | 16 | 距离 Distance |
| `minecraft:horse_one_cm` | 18 | 距离 Distance |
| `minecraft:aviate_one_cm` | 19 | 距离 Distance |
| `minecraft:swim_one_cm` | 20 | 距离 Distance |
| `minecraft:strider_one_cm` | 21 | 距离 Distance |
| `minecraft:jump` | 22 | 无 None |
| `minecraft:drop` | 23 | 无 None |
| `minecraft:damage_dealt` | 24 | 伤害 Damage |
| `minecraft:damage_dealt_absorbed` | 25 | 伤害 Damage |
| `minecraft:damage_dealt_resisted` | 26 | 伤害 Damage |
| `minecraft:damage_taken` | 27 | 伤害 Damage |
| `minecraft:damage_blocked_by_shield` | 28 | 伤害 Damage |
| `minecraft:damage_absorbed` | 29 | 伤害 Damage |
| `minecraft:damage_resisted` | 30 | 伤害 Damage |
| `minecraft:deaths` | 31 | 无 None |
| `minecraft:mob_kills` | 32 | 无 None |
| `minecraft:animals_bred` | 33 | 无 None |
| `minecraft:player_kills` | 34 | 无 None |
| `minecraft:fish_caught` | 35 | 无 None |
| `minecraft:talked_to_villager` | 36 | 无 None |
| `minecraft:traded_with_villager` | 37 | 无 None |
| `minecraft:eat_cake_slice` | 38 | 无 None |
| `minecraft:fill_cauldron` | 39 | 无 None |
| `minecraft:use_cauldron` | 40 | 无 None |
| `minecraft:clean_armor` | 41 | 无 None |
| `minecraft:clean_banner` | 42 | 无 None |
| `minecraft:clean_shulker_box` | 43 | 无 None |
| `minecraft:interact_with_brewingstand` | 44 | 无 None |
| `minecraft:interact_with_beacon` | 45 | 无 None |
| `minecraft:inspect_dropper` | 46 | 无 None |
| `minecraft:inspect_hopper` | 47 | 无 None |
| `minecraft:inspect_dispenser` | 48 | 无 None |
| `minecraft:play_noteblock` | 49 | 无 None |
| `minecraft:tune_noteblock` | 50 | 无 None |
| `minecraft:pot_flower` | 51 | 无 None |
| `minecraft:trigger_trapped_chest` | 52 | 无 None |
| `minecraft:open_enderchest` | 53 | 无 None |
| `minecraft:enchant_item` | 54 | 无 None |
| `minecraft:play_record` | 55 | 无 None |
| `minecraft:interact_with_furnace` | 56 | 无 None |
| `minecraft:interact_with_crafting_table` | 57 | 无 None |
| `minecraft:open_chest` | 58 | 无 None |
| `minecraft:sleep_in_bed` | 59 | 无 None |
| `minecraft:open_shulker_box` | 60 | 无 None |
| `minecraft:open_barrel` | 61 | 无 None |
| `minecraft:interact_with_blast_furnace` | 62 | 无 None |
| `minecraft:interact_with_smoker` | 63 | 无 None |
| `minecraft:interact_with_lectern` | 64 | 无 None |
| `minecraft:interact_with_campfire` | 65 | 无 None |
| `minecraft:interact_with_cartography_table` | 66 | 无 None |
| `minecraft:interact_with_loom` | 67 | 无 None |
| `minecraft:interact_with_stonecutter` | 68 | 无 None |
| `minecraft:bell_ring` | 69 | 无 None |
| `minecraft:raid_trigger` | 70 | 无 None |
| `minecraft:raid_win` | 71 | 无 None |
| `minecraft:interact_with_anvil` | 72 | 无 None |
| `minecraft:interact_with_grindstone` | 73 | 无 None |
| `minecraft:target_hit` | 74 | 无 None |
| `minecraft:interact_with_smithing_table` | 75 | 无 None |

单位 Units：

- 无 None：只是一个普通数字（格式化为0位小数）
- 伤害 Damage：值是正常数量的10倍
- 距离 Distance：以厘米为单位的距离（方块的百分之一）
- 时间 Time：以刻 ticks 为单位的时间跨度

#### 确认方块更改 Acknowledge Block Change

确认用户发起的方块更改。收到此数据包后，客户端将显示服务器发送的方块状态，而不是客户端预测的方块状态。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x04`<br/>`block_changed_ack` | 游戏 Play | 客户端 Client | 序列ID Sequence ID | VarInt | 表示要确认的序列；这用于在交互后正确同步方块更改到客户端。 |

#### 设置方块破坏阶段 Set Block Destroy Stage

0-9是可显示的破坏阶段，任何其他数字意味着此坐标上没有动画。

方块破坏动画仍然可以应用于空气；动画将保持可见，尽管没有方块被破坏。但是，如果应用于透明方块，可能会发生奇怪的图形效果，包括水失去透明度。（在正常游戏中破坏冰块时可以看到类似的效果）

如果需要同时显示多个破坏动画，则必须为每个动画提供唯一的实体ID。实体ID不需要对应于客户端上的实际实体。使用随机生成的数字是有效的。

移除破坏动画时，必须使用设置它的实体的ID。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x05`<br/>`block_destruction` | 游戏 Play | 客户端 Client | 实体ID Entity ID | VarInt | 正在破坏方块的实体的ID。 |
| `0x05`<br/>`block_destruction` | 游戏 Play | 客户端 Client | 位置 Location | 位置 Position | 方块位置 Block Position。 |
| `0x05`<br/>`block_destruction` | 游戏 Play | 客户端 Client | 破坏阶段 Destroy Stage | 无符号字节 Unsigned Byte | 0-9设置它，任何其他值移除它。 |

#### 方块实体数据 Block Entity Data

设置与给定位置的方块关联的方块实体 block entity。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x06`<br/>`block_entity_data` | 游戏 Play | 客户端 Client | 位置 Location | 位置 Position | |
| `0x06`<br/>`block_entity_data` | 游戏 Play | 客户端 Client | 类型 Type | VarInt | `minecraft:block_entity_type` 注册表中的ID |
| `0x06`<br/>`block_entity_data` | 游戏 Play | 客户端 Client | NBT数据 NBT Data | NBT | 要设置的数据。 |

#### 方块动作 Block Action

此数据包用于方块执行的许多动作和动画，通常是非持久性的。客户端忽略提供的方块类型，而是使用其世界中的方块状态。

有关值列表，请参阅方块动作 Block Actions。

**警告：** 此数据包使用来自 `minecraft:block` 注册表的方块ID，而不是方块状态。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x07`<br/>`block_event` | 游戏 Play | 客户端 Client | 位置 Location | 位置 Position | 方块坐标 Block coordinates。 |
| `0x07`<br/>`block_event` | 游戏 Play | 客户端 Client | 动作ID（字节1） Action ID (Byte 1) | 无符号字节 Unsigned Byte | 根据方块而变化 - 请参阅方块动作 Block Actions。 |
| `0x07`<br/>`block_event` | 游戏 Play | 客户端 Client | 动作参数（字节2） Action Parameter (Byte 2) | 无符号字节 Unsigned Byte | 根据方块而变化 - 请参阅方块动作 Block Actions。 |
| `0x07`<br/>`block_event` | 游戏 Play | 客户端 Client | 方块类型 Block Type | VarInt | `minecraft:block` 注册表中的ID。原版客户端不使用此值，因为它将根据给定位置推断方块类型。 |

#### 方块更新 Block Update

在渲染距离内更改方块时触发。

**警告：** 在未加载的区块中更改方块不是稳定的操作。原版客户端当前使用共享的空区块，该区块针对未加载区块中的所有方块更改进行修改；虽然在1.9中此区块从不渲染，但在旧版本中，更改的方块将出现在空区块的所有副本中。服务器应避免在未加载的区块中发送方块更改，客户端应忽略此类数据包。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x08`<br/>`block_update` | 游戏 Play | 客户端 Client | 位置 Location | 位置 Position | 方块坐标 Block Coordinates。 |
| `0x08`<br/>`block_update` | 游戏 Play | 客户端 Client | 方块ID Block ID | VarInt | 方块状态注册表 block state registry 中给出的方块的新方块状态ID。 |

#### Boss栏 Boss Bar

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x09`<br/>`boss_event` | 游戏 Play | 客户端 Client | UUID | UUID | 此栏的唯一ID Unique ID for this bar。 |
| `0x09`<br/>`boss_event` | 游戏 Play | 客户端 Client | 动作 Action | VarInt枚举 VarInt Enum | 确定剩余数据包的布局。 |

Boss栏动作：

| 动作 Action | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|------|----------|----------|------|
| 0: 添加 add | 标题 Title | 文本组件 Text Component | |
| 0: 添加 add | 生命值 Health | 浮点型 Float | 从0到1。大于1的值不会使原版客户端崩溃，并在约1.5时开始渲染第二条生命条的一部分。 |
| 0: 添加 add | 颜色 Color | VarInt枚举 VarInt Enum | 颜色ID Color ID（见下文）。 |
| 0: 添加 add | 分段 Division | VarInt枚举 VarInt Enum | 分段类型 Type of division（见下文）。 |
| 0: 添加 add | 标志 Flags | 无符号字节 Unsigned Byte | 位掩码 Bit mask。0x01：应该使天空变暗，0x02：是龙栏（用于播放末地音乐），0x04：创建雾（以前也由0x02控制）。 |
| 1: 移除 remove | 无字段 no fields | 无字段 no fields | 移除此Boss栏。 |
| 2: 更新生命值 update health | 生命值 Health | 浮点型 Float | 如上所述 as above |
| 3: 更新标题 update title | 标题 Title | 文本组件 Text Component | |
| 4: 更新样式 update style | 颜色 Color | VarInt枚举 VarInt Enum | 颜色ID Color ID（见下文）。 |
| 4: 更新样式 update style | 分隔符 Dividers | VarInt枚举 VarInt Enum | 如上所述 as above |
| 5: 更新标志 update flags | 标志 Flags | 无符号字节 Unsigned Byte | 如上所述 as above |

颜色 Color：

| ID | 颜色 Color |
|----|------|
| 0 | 粉红色 Pink |
| 1 | 蓝色 Blue |
| 2 | 红色 Red |
| 3 | 绿色 Green |
| 4 | 黄色 Yellow |
| 5 | 紫色 Purple |
| 6 | 白色 White |

分段类型 Type of division：

| ID | 分段类型 Type of division |
|----|------|
| 0 | 无分段 No division |
| 1 | 6个刻度 6 notches |
| 2 | 10个刻度 10 notches |
| 3 | 12个刻度 12 notches |
| 4 | 20个刻度 20 notches |

#### 更改难度 Change Difficulty

更改客户端选项菜单中的难度设置。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x0A`<br/>`change_difficulty` | 游戏 Play | 客户端 Client | 难度 Difficulty | 无符号字节枚举 Unsigned Byte Enum | 0：和平 peaceful，1：简单 easy，2：普通 normal，3：困难 hard。 |
| `0x0A`<br/>`change_difficulty` | 游戏 Play | 客户端 Client | 难度锁定？ Difficulty locked? | 布尔值 Boolean | |

#### 区块批次完成 Chunk Batch Finished

标记区块批次的结束。原版客户端标记接收此数据包的时间，并计算自区块批次开始 Chunk Batch Start 以来经过的持续时间。服务器使用此持续时间和此数据包中接收的批次大小来估计每个接收区块所经过的毫秒数。然后，此值用于通过公式 `25 / millisPerChunk` 计算每刻 tick 所需的区块数，该数据通过区块批次已接收 Chunk Batch Received 报告给服务器。这可能使用 `25` 而不是正常刻持续时间 `50`，因此区块处理仅使用客户端和网络带宽的一半。

原版客户端使用最近15个批次的样本来估计每个区块的毫秒数。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x0B`<br/>`chunk_batch_finished` | 游戏 Play | 客户端 Client | 批次大小 Batch size | VarInt | 区块数量 Number of chunks。 |

#### 区块批次开始 Chunk Batch Start

标记区块批次的开始。原版客户端标记并存储接收此数据包的时间。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x0C`<br/>`chunk_batch_start` | 游戏 Play | 客户端 Client | | | 无字段 no fields |

#### 区块生物群系 Chunk Biomes

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x0D`<br/>`chunks_biomes` | 游戏 Play | 客户端 Client | 区块生物群系数据 Chunk biome data - 区块Z Chunk Z | 前缀数组 Prefixed Array - 整型 Int | 区块坐标（方块坐标除以16，向下取整）Chunk coordinate (block coordinate divided by 16, rounded down) |
| `0x0D`<br/>`chunks_biomes` | 游戏 Play | 客户端 Client | 区块生物群系数据 Chunk biome data - 区块X Chunk X | 前缀数组 Prefixed Array - 整型 Int | 区块坐标（方块坐标除以16，向下取整）Chunk coordinate (block coordinate divided by 16, rounded down) |
| `0x0D`<br/>`chunks_biomes` | 游戏 Play | 客户端 Client | 区块生物群系数据 Chunk biome data - 数据 Data | 前缀数组 Prefixed Array - 字节前缀数组 Prefixed Array of Byte | 区块数据结构 Chunk data structure，其中区块节 sections 仅包含 `Biomes` 字段 |

注意：X和Z的顺序是倒置的，因为客户端将它们读取为一个大端 big-endian 长整型 Long，其中Z是高32位。

#### 清除标题 Clear Titles

清除客户端当前的标题信息，可选择重置它。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x0E`<br/>`clear_titles` | 游戏 Play | 客户端 Client | 重置 Reset | 布尔值 Boolean | |

#### 命令建议响应 Command Suggestions Response

服务器响应发送给它的最后一个单词的自动补全列表。在普通聊天的情况下，这是一个玩家用户名。命令名称和参数也受支持。客户端在列出它们之前按字母顺序对它们进行排序。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x0F`<br/>`command_suggestions` | 游戏 Play | 客户端 Client | ID | VarInt | 事务ID Transaction ID。 |
| `0x0F`<br/>`command_suggestions` | 游戏 Play | 客户端 Client | 开始 Start | VarInt | 要替换的文本的开始位置。 |
| `0x0F`<br/>`command_suggestions` | 游戏 Play | 客户端 Client | 长度 Length | VarInt | 要替换的文本的长度。 |
| `0x0F`<br/>`command_suggestions` | 游戏 Play | 客户端 Client | 匹配 Matches - 匹配 Match | 前缀数组 Prefixed Array - 字符串 String (32767) | 一个要插入的合格值，请注意，每个命令都是单独发送的，而不是在单个字符串中，因此需要计数 Count。 |
| `0x0F`<br/>`command_suggestions` | 游戏 Play | 客户端 Client | 匹配 Matches - 工具提示 Tooltip | 前缀数组 Prefixed Array - 可选文本组件 Prefixed Optional Text Component | 要显示的工具提示。 |

#### 命令 Commands

列出服务器上的所有命令以及如何解析它们。

这是一个有向图，有一个根节点。每个重定向或子节点必须仅引用已声明的节点。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x10`<br/>`commands` | 游戏 Play | 客户端 Client | 节点 Nodes | 节点前缀数组 Prefixed Array of Node | 节点数组 An array of nodes。 |
| `0x10`<br/>`commands` | 游戏 Play | 客户端 Client | 根索引 Root index | VarInt | 前一个数组中 `root` 节点的索引。 |

有关此数据包的更多信息，请参阅命令数据 Command Data 文章。

#### 关闭容器 Close Container

当窗口被强制关闭时，例如当箱子在打开时被摧毁时，此数据包从服务器发送到客户端。原版客户端忽略提供的窗口ID并关闭任何活动窗口。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x11`<br/>`container_close` | 游戏 Play | 客户端 Client | 窗口ID Window ID | VarInt | 这是已关闭窗口的ID。0表示物品栏 inventory。 |

#### 设置容器内容 Set Container Content

替换容器窗口的内容。在初始化容器窗口或玩家物品栏时由服务器发送，并响应状态ID不匹配（请参阅点击容器 Click Container）。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x12`<br/>`container_set_content` | 游戏 Play | 客户端 Client | 窗口ID Window ID | VarInt | 要发送物品的窗口ID。0表示玩家物品栏。客户端忽略针对当前窗口ID以外的任何数据包。但是，玩家物品栏可以随时作为目标，这是一个例外。（原版服务器似乎不使用这种特殊情况。） |
| `0x12`<br/>`container_set_content` | 游戏 Play | 客户端 Client | 状态ID State ID | VarInt | 服务器管理的序列号，用于避免不同步；请参阅点击容器 Click Container。 |
| `0x12`<br/>`container_set_content` | 游戏 Play | 客户端 Client | 槽位数据 Slot Data | 槽位前缀数组 Prefixed Array of Slot | |
| `0x12`<br/>`container_set_content` | 游戏 Play | 客户端 Client | 携带的物品 Carried Item | 槽位 Slot | 用鼠标拖动的物品。 |

有关槽位如何索引的更多信息，请参阅物品栏窗口 inventory windows。使用打开屏幕 Open Screen 在客户端上打开容器。

#### 设置容器属性 Set Container Property

此数据包用于通知客户端GUI窗口的一部分应该更新。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x13`<br/>`container_set_data` | 游戏 Play | 客户端 Client | 窗口ID Window ID | VarInt | |
| `0x13`<br/>`container_set_data` | 游戏 Play | 客户端 Client | 属性 Property | 短整型 Short | 要更新的属性，见下文。 |
| `0x13`<br/>`container_set_data` | 游戏 Play | 客户端 Client | 值 Value | 短整型 Short | 属性的新值，见下文。 |

属性 Property 字段的含义取决于窗口的类型。下表显示了窗口类型和属性的已知组合，以及如何解释值。

| 窗口类型 Window type | 属性 Property | 值 Value |
|------|------|------|
| 熔炉 Furnace | 0：火焰图标（剩余燃料） Fire icon (fuel left) | 从燃料燃烧时间倒计时到0（游戏刻 in-game ticks） |
| 熔炉 Furnace | 1：最大燃料燃烧时间 Maximum fuel burn time | 燃料燃烧时间或0（游戏刻 in-game ticks） |
| 熔炉 Furnace | 2：进度箭头 Progress arrow | 从0计数到最大进度（游戏刻 in-game ticks） |
| 熔炉 Furnace | 3：最大进度 Maximum progress | 原版服务器上始终为200 |
| 附魔台 Enchantment Table | 0：顶部附魔槽的等级要求 Level requirement for top enchantment slot | 附魔的经验等级要求 The enchantment's xp level requirement |
| 附魔台 Enchantment Table | 1：中部附魔槽的等级要求 Level requirement for middle enchantment slot | 附魔的经验等级要求 The enchantment's xp level requirement |
| 附魔台 Enchantment Table | 2：底部附魔槽的等级要求 Level requirement for bottom enchantment slot | 附魔的经验等级要求 The enchantment's xp level requirement |
| 附魔台 Enchantment Table | 3：附魔种子 The enchantment seed | 用于在客户端绘制附魔名称（在标准银河字母 SGA中）。相同的种子用于计算附魔，但某些数据不会发送到客户端，以防止轻松猜测整个列表（此处的种子值是常规种子按位与 `0xFFFFFFF0`）。 |
| 附魔台 Enchantment Table | 4：鼠标悬停在顶部附魔槽上显示的附魔ID Enchantment ID shown on mouse hover over top enchantment slot | 附魔ID（设置为-1隐藏它），见下文的值 The enchantment ID (set to -1 to hide it) |
| 附魔台 Enchantment Table | 5：鼠标悬停在中部附魔槽上显示的附魔ID Enchantment ID shown on mouse hover over middle enchantment slot | 附魔ID（设置为-1隐藏它） The enchantment ID (set to -1 to hide it) |
| 附魔台 Enchantment Table | 6：鼠标悬停在底部附魔槽上显示的附魔ID Enchantment ID shown on mouse hover over bottom enchantment slot | 附魔ID（设置为-1隐藏它） The enchantment ID (set to -1 to hide it) |
| 附魔台 Enchantment Table | 7：鼠标悬停在顶部槽上显示的附魔等级 Enchantment level shown on mouse hover over the top slot | 附魔等级（1=I，2=II，6=VI等），如果没有附魔则为-1 The enchantment level (1 = I, 2 = II, 6 = VI, etc.), or -1 if no enchant |
| 附魔台 Enchantment Table | 8：鼠标悬停在中部槽上显示的附魔等级 Enchantment level shown on mouse hover over the middle slot | 附魔等级（1=I，2=II，6=VI等），如果没有附魔则为-1 |
| 附魔台 Enchantment Table | 9：鼠标悬停在底部槽上显示的附魔等级 Enchantment level shown on mouse hover over the bottom slot | 附魔等级（1=I，2=II，6=VI等），如果没有附魔则为-1 |
| 信标 Beacon | 0：能量等级 Power level | 0-4，控制启用哪些效果按钮 |
| 信标 Beacon | 1：第一个药水效果 First potion effect | 第一个效果的药水效果ID Potion effect ID，如果没有效果则为-1 |
| 信标 Beacon | 2：第二个药水效果 Second potion effect | 第二个效果的药水效果ID Potion effect ID，如果没有效果则为-1 |
| 铁砧 Anvil | 0：修复成本 Repair cost | 以经验等级为单位的修复成本 The repair's cost in XP levels |
| 酿造台 Brewing Stand | 0：酿造时间 Brew time | 0-400，400使箭头为空，0使箭头为满 |
| 酿造台 Brewing Stand | 1：燃料时间 Fuel time | 0-20，0使箭头为空，20使箭头为满 |
| 切石机 Stonecutter | 0：选定的配方 Selected recipe | 选定配方的索引。-1表示未选择。 |
| 织布机 Loom | 0：选定的图案 Selected pattern | 选定图案的索引。0表示未选择，0也是"基础"图案的内部ID。 |
| 讲台 Lectern | 0：页码 Page number | 当前页码，从0开始。 |
| 锻造台 Smithing Table | 0：有配方错误 Has recipe error | 如果大于零则为真 True if greater than zero。 |

对于附魔台，使用以下数字ID：

| 数字ID Numerical ID | 附魔ID Enchantment ID | 附魔名称 Enchantment Name |
|------|------|------|
| 0 | minecraft:protection | 保护 Protection |
| 1 | minecraft:fire_protection | 火焰保护 Fire Protection |
| 2 | minecraft:feather_falling | 摔落保护 Feather Falling |
| 3 | minecraft:blast_protection | 爆炸保护 Blast Protection |
| 4 | minecraft:projectile_protection | 弹射物保护 Projectile Protection |
| 5 | minecraft:respiration | 水下呼吸 Respiration |
| 6 | minecraft:aqua_affinity | 水下速掘 Aqua Affinity |
| 7 | minecraft:thorns | 荆棘 Thorns |
| 8 | minecraft:depth_strider | 深海探索者 Depth Strider |
| 9 | minecraft:frost_walker | 冰霜行者 Frost Walker |
| 10 | minecraft:binding_curse | 绑定诅咒 Curse of Binding |
| 11 | minecraft:soul_speed | 灵魂疾行 Soul Speed |
| 12 | minecraft:swift_sneak | 迅捷潜行 Swift Sneak |
| 13 | minecraft:sharpness | 锋利 Sharpness |
| 14 | minecraft:smite | 亡灵杀手 Smite |
| 15 | minecraft:bane_of_arthropods | 节肢杀手 Bane of Arthropods |
| 16 | minecraft:knockback | 击退 Knockback |
| 17 | minecraft:fire_aspect | 火焰附加 Fire Aspect |
| 18 | minecraft:looting | 抢夺 Looting |
| 19 | minecraft:sweeping_edge | 横扫之刃 Sweeping Edge |
| 20 | minecraft:efficiency | 效率 Efficiency |
| 21 | minecraft:silk_touch | 精准采集 Silk Touch |
| 22 | minecraft:unbreaking | 耐久 Unbreaking |
| 23 | minecraft:fortune | 时运 Fortune |
| 24 | minecraft:power | 力量 Power |
| 25 | minecraft:punch | 冲击 Punch |
| 26 | minecraft:flame | 火矢 Flame |
| 27 | minecraft:infinity | 无限 Infinity |
| 28 | minecraft:luck_of_the_sea | 海之眷顾 Luck of the Sea |
| 29 | minecraft:lure | 饵钓 Lure |
| 30 | minecraft:loyalty | 忠诚 Loyalty |
| 31 | minecraft:impaling | 穿刺 Impaling |
| 32 | minecraft:riptide | 激流 Riptide |
| 33 | minecraft:channeling | 引雷 Channeling |
| 34 | minecraft:multishot | 多重射击 Multishot |
| 35 | minecraft:quick_charge | 快速装填 Quick Charge |
| 36 | minecraft:piercing | 穿透 Piercing |
| 37 | minecraft:density | 密度 Density |
| 38 | minecraft:breach | 破甲 Breach |
| 39 | minecraft:wind_burst | 风爆 Wind Burst |
| 40 | minecraft:mending | 经验修补 Mending |
| 41 | minecraft:vanishing_curse | 消失诅咒 Curse of Vanishing |

#### 设置容器槽位 Set Container Slot

当槽位中的物品（在窗口中）添加/移除时由服务器发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x14`<br/>`container_set_slot` | 游戏 Play | 客户端 Client | 窗口ID Window ID | VarInt | 正在更新的窗口。0表示玩家物品栏。客户端忽略针对当前窗口ID以外的任何数据包；有关例外情况，请参见下文。 |
| `0x14`<br/>`container_set_slot` | 游戏 Play | 客户端 Client | 状态ID State ID | VarInt | 服务器管理的序列号，用于避免不同步；请参阅点击容器 Click Container。 |
| `0x14`<br/>`container_set_slot` | 游戏 Play | 客户端 Client | 槽位 Slot | 短整型 Short | 应该更新的槽位。 |
| `0x14`<br/>`container_set_slot` | 游戏 Play | 客户端 Client | 槽位数据 Slot Data | 槽位 Slot | |

如果窗口ID为0，即使打开了不同的容器窗口，热键栏和副手槽位（槽位36到45）也可能会更新。（原版服务器似乎不使用这种特殊情况。）当玩家查看除生存物品栏以外的创造模式物品栏选项卡时，更新也仅限于这些槽位。（原版服务器不以任何方式处理此限制，导致MC-242392。）

当容器窗口打开时，服务器从不发送针对窗口ID 0的更新——所有窗口类型 window types 都包括玩家物品栏的槽位。客户端必须自动将针对容器窗口的物品栏部分的更改应用于主物品栏；服务器在窗口关闭时不会为ID 0重新发送它们。但是，由于护甲和副手槽位仅存在于ID 0上，因此在窗口打开时对这些槽位的更新必须由服务器推迟到窗口关闭为止。

#### Cookie请求（游戏） Cookie Request (play)

请求先前存储的cookie。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x15`<br/>`cookie_request` | 游戏 Play | 客户端 Client | 键 Key | 标识符 Identifier | cookie的标识符。 |

#### 设置冷却 Set Cooldown

对具有给定类型的所有物品应用冷却期。原版服务器将其用于末影珍珠。应在冷却开始时和冷却结束时发送此数据包（以补偿延迟），尽管客户端会自动结束冷却。可以应用于任何物品，请注意，交互仍然会与物品一起发送到服务器，但客户端不会播放动画，也不会尝试预测结果（即方块放置）。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x16`<br/>`cooldown` | 游戏 Play | 客户端 Client | 冷却组 Cooldown Group | 标识符 Identifier | 物品的标识符（minecraft:stone）或冷却组（"use_cooldown"物品组件）Identifier of the item or the cooldown group |
| `0x16`<br/>`cooldown` | 游戏 Play | 客户端 Client | 冷却刻 Cooldown Ticks | VarInt | 应用冷却的刻数，或0以清除冷却。 |

#### 聊天建议 Chat Suggestions

原版服务器未使用。可能是为自定义服务器提供的，用于向客户端发送聊天消息补全。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x17`<br/>`custom_chat_completions` | 游戏 Play | 客户端 Client | 动作 Action | VarInt枚举 VarInt Enum | 0：添加 Add，1：移除 Remove，2：设置 Set |
| `0x17`<br/>`custom_chat_completions` | 游戏 Play | 客户端 Client | 条目 Entries | 字符串前缀数组 Prefixed Array of String (32767) | |

#### 客户端绑定插件消息（游戏） Clientbound Plugin Message (play)

模组和插件可以使用它来发送他们的数据。Minecraft本身使用几个插件频道 plugin channels。这些内部频道位于 `minecraft` 命名空间中。

有关其工作原理的更多信息，请访问Dinnerbone的博客。有关内部和流行注册频道的更多文档在这里。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x18`<br/>`custom_payload` | 游戏 Play | 客户端 Client | 频道 Channel | 标识符 Identifier | 用于发送数据的插件频道 plugin channel 的名称。 |
| `0x18`<br/>`custom_payload` | 游戏 Play | 客户端 Client | 数据 Data | 字节数组 Byte Array (1048576) | 任何数据。必须从数据包长度推断此数组的长度。 |

在原版客户端中，最大数据长度为1048576字节。

#### 伤害事件 Damage Event

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x19`<br/>`damage_event` | 游戏 Play | 客户端 Client | 实体ID Entity ID | VarInt | 受到伤害的实体的ID The ID of the entity taking damage |
| `0x19`<br/>`damage_event` | 游戏 Play | 客户端 Client | 源类型ID Source Type ID | VarInt | `minecraft:damage_type` 注册表中的伤害类型，由注册表数据 Registry Data 数据包定义。 |
| `0x19`<br/>`damage_event` | 游戏 Play | 客户端 Client | 源原因ID Source Cause ID | VarInt | 负责伤害的实体的ID+1（如果存在）。如果不存在，则值为0 |
| `0x19`<br/>`damage_event` | 游戏 Play | 客户端 Client | 源直接ID Source Direct ID | VarInt | 直接造成伤害的实体的ID+1（如果存在）。如果不存在，则值为0。如果此字段存在：<br/>- 如果伤害是间接造成的，例如使用弹射物，此字段将包含该弹射物的ID；<br/>- 如果伤害是直接造成的，例如手动攻击，此字段将包含与源原因ID Source Cause ID相同的值。 |
| `0x19`<br/>`damage_event` | 游戏 Play | 客户端 Client | 源位置 Source Position - X | 可选前缀 Prefixed Optional - 双精度 Double | 当伤害由/damage命令造成且指定了位置时，原版服务器发送源位置 Source Position |
| `0x19`<br/>`damage_event` | 游戏 Play | 客户端 Client | 源位置 Source Position - Y | 可选前缀 Prefixed Optional - 双精度 Double | 同上 |
| `0x19`<br/>`damage_event` | 游戏 Play | 客户端 Client | 源位置 Source Position - Z | 可选前缀 Prefixed Optional - 双精度 Double | 同上 |

#### 调试样本 Debug Sample

在客户端订阅调试样本订阅 Debug Sample Subscription 后定期发送的样本数据。

原版服务器仅向服务器操作员的玩家发送调试样本。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x1A`<br/>`debug_sample` | 游戏 Play | 客户端 Client | 样本 Sample | 长整型前缀数组 Prefixed Array of Long | 类型相关的样本数组 Array of type-dependent samples。 |
| `0x1A`<br/>`debug_sample` | 游戏 Play | 客户端 Client | 样本类型 Sample Type | VarInt枚举 VarInt Enum | 见下文。 |

类型 Types：

| ID | 名称 Name | 描述 Description |
|----|------|------|
| 0 | 刻时间 Tick time | 四个不同的刻相关指标，每个都由数组中的一个长整型表示。它们以纳秒为单位测量，如下所示：<br/>- 0：完整刻时间 Full tick time：以下三个时间的总和；<br/>- 1：服务器刻时间 Server tick time：主服务器刻逻辑；<br/>- 2：任务时间 Tasks time：计划在主逻辑后执行的任务；<br/>- 3：空闲时间 Idle time：完成完整50ms刻周期的空闲时间。<br/>请注意，原版客户端通过从完整刻时间中减去空闲时间来计算用于最小/最大/平均显示的时序。如果空闲时间（荒谬地）大于完整刻时间，这可能导致显示的值变为负数。 |

#### 删除消息 Delete Message

从客户端的聊天中删除消息。这仅适用于带签名的消息；无法使用此数据包删除系统消息。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x1B`<br/>`delete_chat` | 游戏 Play | 客户端 Client | 消息ID Message ID | VarInt | 消息ID+1，用于验证消息签名。仅当此字段的值等于0时，下一个字段才存在。 |
| `0x1B`<br/>`delete_chat` | 游戏 Play | 客户端 Client | 签名 Signature | 可选字节数组 Optional Byte Array (256) | 前一条消息的签名。始终为256字节，且不带长度前缀。 |

#### 断开连接（游戏） Disconnect (play)

在断开客户端连接之前由服务器发送。客户端假设服务器在数据包到达时已经关闭了连接。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x1C`<br/>`disconnect` | 游戏 Play | 客户端 Client | 原因 Reason | 文本组件 Text Component | 连接终止时显示给客户端。 |

#### 伪装的聊天消息 Disguised Chat Message

向客户端发送聊天消息，但不包含任何消息签名信息。

原版服务器在控制台通过命令与玩家通信时使用此数据包，例如 `/say`、`/tell`、`/me` 等。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x1D`<br/>`disguised_chat` | 游戏 Play | 客户端 Client | 消息 Message | 文本组件 Text Component | 在客户端格式化消息时，这用作 `content` 参数。 |
| `0x1D`<br/>`disguised_chat` | 游戏 Play | 客户端 Client | 聊天类型 Chat Type | ID或聊天类型 ID or Chat Type | 由注册表数据 Registry Data 数据包定义的 `minecraft:chat_type` 注册表中的聊天类型，或内联定义。 |
| `0x1D`<br/>`disguised_chat` | 游戏 Play | 客户端 Client | 发送者名称 Sender Name | 文本组件 Text Component | 发送消息的人的名称，通常是发送者的显示名称。<br/>在客户端格式化消息时，这用作 `sender` 参数。 |
| `0x1D`<br/>`disguised_chat` | 游戏 Play | 客户端 Client | 目标名称 Target Name | 可选文本组件前缀 Prefixed Optional Text Component | 接收消息的人的名称，通常是接收者的显示名称。<br/>在客户端格式化消息时，这用作 `target` 参数。 |

#### 实体事件 Entity Event

实体状态通常会触发实体的动画。可用状态因实体类型而异（并且对该类型的子类也可用）。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x1E`<br/>`entity_event` | 游戏 Play | 客户端 Client | 实体ID Entity ID | 整型 Int | |
| `0x1E`<br/>`entity_event` | 游戏 Play | 客户端 Client | 实体状态 Entity Status | 字节枚举 Byte Enum | 有关哪些状态对每种类型的实体有效的列表，请参阅实体状态 Entity statuses。 |

#### 传送实体 Teleport Entity

**警告：** 此数据包的Mojang指定名称在1.21.2中从 `teleport_entity` 更改为 `entity_position_sync`。有一个新的 `teleport_entity`，本文档更恰当地称为同步载具位置 Synchronize Vehicle Position。该数据包具有不同的功能，如果用来代替此数据包将导致混乱的结果。

当实体移动超过8个方块时，服务器发送此数据包。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x1F`<br/>`entity_position_sync` | 游戏 Play | 客户端 Client | 实体ID Entity ID | VarInt | |
| `0x1F`<br/>`entity_position_sync` | 游戏 Play | 客户端 Client | X | 双精度 Double | |
| `0x1F`<br/>`entity_position_sync` | 游戏 Play | 客户端 Client | Y | 双精度 Double | |
| `0x1F`<br/>`entity_position_sync` | 游戏 Play | 客户端 Client | Z | 双精度 Double | |
| `0x1F`<br/>`entity_position_sync` | 游戏 Play | 客户端 Client | 速度X Velocity X | 双精度 Double | |
| `0x1F`<br/>`entity_position_sync` | 游戏 Play | 客户端 Client | 速度Y Velocity Y | 双精度 Double | |
| `0x1F`<br/>`entity_position_sync` | 游戏 Play | 客户端 Client | 速度Z Velocity Z | 双精度 Double | |
| `0x1F`<br/>`entity_position_sync` | 游戏 Play | 客户端 Client | 偏航角 Yaw | 浮点型 Float | X轴上的旋转，以度为单位 Rotation on the X axis, in degrees。 |
| `0x1F`<br/>`entity_position_sync` | 游戏 Play | 客户端 Client | 俯仰角 Pitch | 浮点型 Float | Y轴上的旋转，以度为单位 Rotation on the Y axis, in degrees。 |
| `0x1F`<br/>`entity_position_sync` | 游戏 Play | 客户端 Client | 在地面上 On Ground | 布尔值 Boolean | |

#### 爆炸 Explosion

当发生爆炸时发送（苦力怕、TNT和恶魂火球）。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x20`<br/>`explode` | 游戏 Play | 客户端 Client | X | 双精度 Double | |
| `0x20`<br/>`explode` | 游戏 Play | 客户端 Client | Y | 双精度 Double | |
| `0x20`<br/>`explode` | 游戏 Play | 客户端 Client | Z | 双精度 Double | |
| `0x20`<br/>`explode` | 游戏 Play | 客户端 Client | 玩家速度增量 Player Delta Velocity - X | 可选前缀 Prefixed Optional - 双精度 Double | 被爆炸推动的玩家的速度差 Velocity difference of the player being pushed by the explosion。 |
| `0x20`<br/>`explode` | 游戏 Play | 客户端 Client | 玩家速度增量 Player Delta Velocity - Y | 可选前缀 Prefixed Optional - 双精度 Double | 同上 |
| `0x20`<br/>`explode` | 游戏 Play | 客户端 Client | 玩家速度增量 Player Delta Velocity - Z | 可选前缀 Prefixed Optional - 双精度 Double | 同上 |
| `0x20`<br/>`explode` | 游戏 Play | 客户端 Client | 爆炸粒子ID Explosion Particle ID | VarInt | `minecraft:particle_type` 注册表中的ID。 |
| `0x20`<br/>`explode` | 游戏 Play | 客户端 Client | 爆炸粒子数据 Explosion Particle Data | 可变 Varies | 粒子 Particles 中指定的粒子数据。 |
| `0x20`<br/>`explode` | 游戏 Play | 客户端 Client | 爆炸声音 Explosion Sound | ID或声音事件 ID or Sound Event | `minecraft:sound_event` 注册表中的ID，或内联定义。 |

#### 卸载区块 Unload Chunk

告诉客户端卸载区块列。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x21`<br/>`forget_level_chunk` | 游戏 Play | 客户端 Client | 区块Z Chunk Z | 整型 Int | 方块坐标除以16，向下取整 Block coordinate divided by 16, rounded down。 |
| `0x21`<br/>`forget_level_chunk` | 游戏 Play | 客户端 Client | 区块X Chunk X | 整型 Int | 方块坐标除以16，向下取整 Block coordinate divided by 16, rounded down。 |

注意：顺序是倒置的，因为客户端将此数据包读取为一个大端 big-endian 长整型 Long，其中Z是高32位。

即使给定的区块当前未加载，发送此数据包也是合法的。

#### 游戏事件 Game Event

用于各种游戏事件，例如天气、重生可用性（来自床 bed 和重生锚 respawn anchor）、游戏模式、某些游戏规则和演示 demo 消息。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x22`<br/>`game_event` | 游戏 Play | 客户端 Client | 事件 Event | 无符号字节 Unsigned Byte | 见下文。 |
| `0x22`<br/>`game_event` | 游戏 Play | 客户端 Client | 值 Value | 浮点型 Float | 取决于事件 Depends on Event。 |

事件 Events：

| 事件 Event | 效果 Effect | 值 Value |
|------|------|------|
| 0 | 无重生方块可用 No respawn block available | 注意：向玩家显示消息'block.minecraft.spawn.not_valid'（您没有家床或充能重生锚，或它被阻挡了）Note: Displays message to the player。 |
| 1 | 开始下雨 Begin raining | |
| 2 | 结束下雨 End raining | |
| 3 | 更改游戏模式 Change game mode | 0：生存 Survival，1：创造 Creative，2：冒险 Adventure，3：旁观 Spectator。 |
| 4 | 获胜游戏 Win game | 0：只是重生玩家 Just respawn player。<br/>1：滚动制作人员名单并重生玩家 Roll the credits and respawn player。<br/>注意：当玩家尚未获得进度"结束了？"时，原版服务器仅发送1，否则发送0。 |
| 5 | 演示事件 Demo event | 0：显示演示欢迎屏幕 Show welcome to demo screen。<br/>101：告知移动控制 Tell movement controls。<br/>102：告知跳跃控制 Tell jump control。<br/>103：告知物品栏控制 Tell inventory control。<br/>104：告知演示结束并打印有关如何截图的消息 Tell that the demo is over and print a message about how to take a screenshot。 |
| 6 | 箭击中玩家 Arrow hit player | 注意：当任何玩家被箭击中时发送 Sent when any player is struck by an arrow。 |
| 7 | 降雨等级变化 Rain level change | 注意：似乎会改变天空颜色和光照 Seems to change both sky color and lighting。<br/>降雨等级范围从0到1 Rain level ranging from 0 to 1。 |
| 8 | 雷暴等级变化 Thunder level change | 注意：似乎会改变天空颜色和光照（与降雨等级变化相同，但不会开始下雨）。原版客户端也需要下雨才能渲染 It also requires rain to render by vanilla client。<br/>雷暴等级范围从0到1 Thunder level ranging from 0 to 1。 |
| 9 | 播放河豚刺痛声音 Play pufferfish sting sound | |
| 10 | 播放远古守卫者生物出现（效果和声音） Play elder guardian mob appearance (effect and sound) | |
| 11 | 启用重生屏幕 Enable respawn screen | 0：启用重生屏幕 Enable respawn screen。<br/>1：立即重生 Immediately respawn（当`doImmediateRespawn`游戏规则更改时发送）。 |
| 12 | 限制合成 Limited crafting | 0：禁用限制合成 Disable limited crafting。<br/>1：启用限制合成 Enable limited crafting（当`doLimitedCrafting`游戏规则更改时发送）。 |
| 13 | 开始等待关卡区块 Start waiting for level chunks | 指示客户端开始关卡区块的等待过程 Instructs the client to begin the waiting process for the level chunks。<br/>在客户端上清除关卡并重新发送后（在第一次或后续重新配置期间），由服务器发送 Sent by the server after the level is cleared on the client and is being re-sent。 |

#### 打开马屏幕 Open Horse Screen

此数据包专门用于打开马GUI。打开屏幕 Open Screen 用于所有其他GUI。如果实体ID不指向类马动物，客户端将不会打开物品栏。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x23`<br/>`horse_screen_open` | 游戏 Play | 客户端 Client | 窗口ID Window ID | VarInt | 与打开屏幕 Open Screen 的字段相同。 |
| `0x23`<br/>`horse_screen_open` | 游戏 Play | 客户端 Client | 物品栏列数 Inventory columns count | VarInt | GUI中存在多少列马物品栏槽位，每列3个槽位 How many columns of horse inventory slots exist in the GUI, 3 slots per column。 |
| `0x23`<br/>`horse_screen_open` | 游戏 Play | 客户端 Client | 实体ID Entity ID | 整型 Int | GUI的"所有者"实体。如果所有者实体死亡或被清除，客户端应关闭GUI The "owner" entity of the GUI。 |

#### 受伤动画 Hurt Animation

为受到伤害的实体播放摇晃动画 Plays a bobbing animation for the entity receiving damage。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x24`<br/>`hurt_animation` | 游戏 Play | 客户端 Client | 实体ID Entity ID | VarInt | 受到伤害的实体的ID The ID of the entity taking damage |
| `0x24`<br/>`hurt_animation` | 游戏 Play | 客户端 Client | 偏航角 Yaw | 浮点型 Float | 相对于实体，伤害来自的方向 The direction the damage is coming from in relation to the entity |

#### 初始化世界边界 Initialize World Border

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x25`<br/>`initialize_border` | 游戏 Play | 客户端 Client | X | 双精度 Double | |
| `0x25`<br/>`initialize_border` | 游戏 Play | 客户端 Client | Z | 双精度 Double | |
| `0x25`<br/>`initialize_border` | 游戏 Play | 客户端 Client | 旧直径 Old Diameter | 双精度 Double | 世界边界单边的当前长度，以米为单位 Current length of a single side of the world border, in meters。 |
| `0x25`<br/>`initialize_border` | 游戏 Play | 客户端 Client | 新直径 New Diameter | 双精度 Double | 世界边界单边的目标长度，以米为单位 Target length of a single side of the world border, in meters。 |
| `0x25`<br/>`initialize_border` | 游戏 Play | 客户端 Client | 速度 Speed | VarLong | 达到新直径的实时毫秒数 Number of real-time milliseconds until New Diameter is reached。原版服务器似乎不将世界边界速度同步到游戏刻，因此它会与服务器延迟不同步。如果世界边界未移动，则设置为0。 |
| `0x25`<br/>`initialize_border` | 游戏 Play | 客户端 Client | 传送门传送边界 Portal Teleport Boundary | VarInt | 传送门传送产生的坐标限制为±值 Resulting coordinates from a portal teleport are limited to ±value。通常为29999984。 |
| `0x25`<br/>`initialize_border` | 游戏 Play | 客户端 Client | 警告方块 Warning Blocks | VarInt | 以米为单位 In meters。 |
| `0x25`<br/>`initialize_border` | 游戏 Play | 客户端 Client | 警告时间 Warning Time | VarInt | 以秒为单位，由`/worldborder warning time`设置 In seconds as set by `/worldborder warning time`。 |

原版客户端通过比较警告距离或以下两者中较低者（当前直径到目标直径的距离或边界在warningTime秒后将到达的位置）中较高者来确定警告的显示强度。伪代码：

```java
distance = max(min(resizeSpeed * 1000 * warningTime, abs(targetDiameter - currentDiameter)), warningDistance);
if (playerDistance < distance) {
    warning = 1.0 - playerDistance / distance;
} else {
    warning = 0.0;
}
```

#### 客户端绑定保持连接（游戏） Clientbound Keep Alive (play)

服务器将经常发出保持连接，每个都包含一个随机ID。客户端必须使用相同的有效载荷进行响应（请参阅服务器绑定保持连接 Serverbound Keep Alive）。如果客户端在发送后15秒内未响应保持连接数据包，服务器将踢出客户端。反之，如果服务器在20秒内未发送任何保持连接，客户端将断开连接并产生"Timed out"异常。

原版服务器使用与系统相关的时间（以毫秒为单位）生成保持连接ID值。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x26`<br/>`keep_alive` | 游戏 Play | 客户端 Client | 保持连接ID Keep Alive ID | 长整型 Long | |

#### 区块数据和更新光照 Chunk Data and Update Light

当区块进入客户端的视距时发送，指定其地形、光照和方块实体。

区块必须在之前使用设置中心区块 Set Center Chunk 指定的视图区域内；有关详细信息，请参阅该数据包。

在此数据包中发送所有方块实体不是严格必要的；稍后使用方块实体数据 Block Entity Data 发送它们仍然是合法的。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x27`<br/>`level_chunk_with_light` | 游戏 Play | 客户端 Client | 区块X Chunk X | 整型 Int | 区块坐标（方块坐标除以16，向下取整） Chunk coordinate (block coordinate divided by 16, rounded down) |
| `0x27`<br/>`level_chunk_with_light` | 游戏 Play | 客户端 Client | 区块Z Chunk Z | 整型 Int | 区块坐标（方块坐标除以16，向下取整） Chunk coordinate (block coordinate divided by 16, rounded down) |
| `0x27`<br/>`level_chunk_with_light` | 游戏 Play | 客户端 Client | 数据 Data | 区块数据 Chunk Data | |
| `0x27`<br/>`level_chunk_with_light` | 游戏 Play | 客户端 Client | 光照 Light | 光照数据 Light Data | |

与使用相同格式的更新光照 Update Light 数据包不同，在方块光照或天空光照掩码中将对应于某个部分的位设置为0似乎并不有用，测试结果高度不一致。

#### 世界事件 World Event

当客户端播放声音或粒子效果时发送。

默认情况下，Minecraft客户端根据距离调整音效的音量。最后的布尔字段用于禁用此功能，而是从正确方向的2个方块外播放效果。目前，这仅用于效果1023（凋灵生成）、效果1028（末影龙死亡）和效果1038（末地传送门打开）；在其他效果上被忽略。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|----------|------|--------|----------|----------|------|
| `0x28`<br/>`level_event` | 游戏 Play | 客户端 Client | 事件 Event | 整型 Int | 事件，见下文 The event, see below。 |
| `0x28`<br/>`level_event` | 游戏 Play | 客户端 Client | 位置 Location | 位置 Position | 事件的位置 The location of the event。 |
| `0x28`<br/>`level_event` | 游戏 Play | 客户端 Client | 数据 Data | 整型 Int | 某些事件的额外数据，见下文 Extra data for certain events, see below。 |
| `0x28`<br/>`level_event` | 游戏 Play | 客户端 Client | 禁用相对音量 Disable Relative Volume | 布尔值 Boolean | 见上文 See above。 |

事件 Events：

| ID | 名称 Name | 数据 Data |
|----|------|------|
| **声音 Sound** | | |
| 1000 | 发射器发射 Dispenser dispenses | |
| 1001 | 发射器发射失败 Dispenser fails to dispense | |
| 1002 | 发射器射击 Dispenser shoots | |
| 1004 | 烟花发射 Firework shot | |
| 1009 | 火熄灭 Fire extinguished | |
| 1010 | 播放唱片 Play record | `minecraft:item` 注册表中的ID，对应于唱片物品 record item。如果ID不对应唱片，则忽略数据包。在给定位置播放的任何唱片都会被覆盖。 |
| 1011 | 停止唱片 Stop record | |
| 1015 | 恶魂警告 Ghast warns | |
| 1016 | 恶魂射击 Ghast shoots | |
| 1017 | 末影龙射击 Ender dragon shoots | |
| 1018 | 烈焰人射击 Blaze shoots | |
| 1019 | 僵尸攻击木门 Zombie attacks wooden door | |
| 1020 | 僵尸攻击铁门 Zombie attacks iron door | |
| 1021 | 僵尸破坏木门 Zombie breaks wooden door | |
| 1022 | 凋灵破坏方块 Wither breaks block | |
| 1023 | 凋灵生成 Wither spawned | |
| 1024 | 凋灵射击 Wither shoots | |
| 1025 | 蝙蝠起飞 Bat takes off | |
| 1026 | 僵尸感染 Zombie infects | |
| 1027 | 僵尸村民转化 Zombie villager converted | |
| 1028 | 末影龙死亡 Ender dragon dies | |
| 1029 | 铁砧损坏 Anvil destroyed | |
| 1030 | 铁砧使用 Anvil used | |
| 1031 | 铁砧落地 Anvil lands | |
| 1032 | 传送门旅行 Portal travel | |
| 1033 | 紫颂花生长 Chorus flower grows | |
| 1034 | 紫颂花死亡 Chorus flower dies | |
| 1035 | 酿造台酿造 Brewing stand brews | |
| 1038 | 末地传送门创建 End portal created | |

---

**翻译进度：第1-5部分完成，第6部分（游戏 Play）进行中**

**已翻译：**
- 介绍、定义和数据包格式 ✅
- 握手 Handshaking ✅
- 状态 Status ✅
- 登录 Login ✅
- 配置 Configuration ✅
- 游戏 Play（进行中）
  - 捆绑分隔符 Bundle Delimiter ✅
  - 生成实体 Spawn Entity ✅
  - 实体动画 Entity Animation ✅
  - 授予统计信息 Award Statistics ✅（完整统计表）
  - 确认方块更改 Acknowledge Block Change ✅
  - 设置方块破坏阶段 Set Block Destroy Stage ✅
  - 方块实体数据 Block Entity Data ✅
  - 方块动作 Block Action ✅
  - 方块更新 Block Update ✅
  - Boss栏 Boss Bar ✅（开始）

**待继续：**
- 游戏（Play）剩余数据包 - 约8200行
- 导航（Navigation）
| 1039 | 幻翼咬 Phantom bites | |
| 1040 | 僵尸转化为溺尸 Zombie converts to drowned | |
| 1041 | 尸壳溺水转化为僵尸 Husk converts to zombie by drowning | |
| 1042 | 砂轮使用 Grindstone used | |
| 1043 | 书页翻页 Book page turned | |
| 1044 | 锻造台使用 Smithing table used | |
| 1045 | 滴水石锥落地 Pointed dripstone landing | |
| 1046 | 熔岩从滴水石滴到炼药锅 Lava dripping on cauldron from dripstone | |
| 1047 | 水从滴水石滴到炼药锅 Water dripping on cauldron from dripstone | |
| 1048 | 骷髅转化为流浪者 Skeleton converts to stray | |
| 1049 | 合成器成功合成物品 Crafter successfully crafts item | |
| 1050 | 合成器合成物品失败 Crafter fails to craft item | |

**粒子 Particle**

| ID | 名称 Name | 数据 Data |
|---|---|---|
| 1500 | 堆肥桶堆肥 Composter composts | |
| 1501 | 熔岩转化方块 Lava converts block | 将水转化为石头，或移除现有方块（如火把） |
| 1502 | 红石火把熄灭 Redstone torch burns out | |
| 1503 | 末影之眼放置在末地传送门框架 Ender eye placed in end portal frame | |
| 1504 | 流体从滴水石滴下 Fluid drips from dripstone | |
| 1505 | 骨粉粒子和声音 Bone meal particles and sound | 生成多少粒子 How many particles to spawn |
| 2000 | 发射器激活烟雾 Dispenser activation smoke | 方向 Direction，见下文 |
| 2001 | 方块破坏 + 方块破坏声音 Block break + block break sound | 方块状态ID Block state ID |
| 2002 | 喷溅药水 Splash potion | RGB颜色整数值 RGB color as an integer（例如 #7FA1FF 为 8364543） |
| 2003 | 末影之眼实体破坏动画 Eye of ender entity break animation | 粒子和声音 particles and sound |
| 2004 | 刷怪笼生成生物 Spawner spawns mob | 烟雾 + 火焰 smoke + flames |
| 2006 | 龙息 Dragon breath | |
| 2007 | 瞬间喷溅药水 Instant splash potion | RGB颜色整数值 RGB color as an integer（例如 #7FA1FF 为 8364543） |
| 2008 | 末影龙破坏方块 Ender dragon destroys block | |
| 2009 | 湿海绵蒸发 Wet sponge vaporizes | |
| 2010 | 合成器激活烟雾 Crafter activation smoke | 方向 Direction，见下文 |
| 2011 | 蜜蜂给植物施肥 Bee fertilizes plant | 生成多少粒子 How many particles to spawn |
| 2012 | 海龟蛋放置 Turtle egg placed | 生成多少粒子 How many particles to spawn |
| 2013 | 重击攻击（锤） Smash attack (mace) | 生成多少粒子 How many particles to spawn |
| 3000 | 末地折跃门生成 End gateway spawns | |
| 3001 | 末影龙复活 Ender dragon resurrected | |
| 3002 | 电火花 Electric spark | |
| 3003 | 铜涂蜡 Copper apply wax | |
| 3004 | 铜去蜡 Copper remove wax | |
| 3005 | 铜刮除氧化 Copper scrape oxidation | |
| 3006 | 幽匿充能 Sculk charge | |
| 3007 | 幽匿尖啸体尖叫 Sculk shrieker shriek | |
| 3008 | 方块完成刷洗 Block finished brushing | 方块状态ID Block state ID |
| 3009 | 嗅探兽蛋破裂 Sniffer egg cracks | 如果为1，3-6；如果为其他数字，1-3个粒子将被生成 |
| 3011 | 试炼刷怪笼生成生物（在刷怪笼处） Trial spawner spawns mob (at spawner) | |
| 3012 | 试炼刷怪笼生成生物（在生成位置） Trial spawner spawns mob (at spawn location) | |
| 3013 | 试炼刷怪笼检测到玩家 Trial spawner detects player | 附近玩家数量 Number of players nearby |
| 3014 | 试炼刷怪笼弹出物品 Trial spawner ejects item | |
| 3015 | 宝库激活 Vault activates | |
| 3016 | 宝库停用 Vault deactivates | |
| 3017 | 宝库弹出物品 Vault ejects item | |
| 3018 | 蜘蛛网编织 Cobweb weaved | |
| 3019 | 不祥试炼刷怪笼检测到玩家 Ominous trial spawner detects player | 附近玩家数量 Number of players nearby |
| 3020 | 试炼刷怪笼变为不祥 Trial spawner turns ominous | 如果为0，声音将以0.3音量播放。否则以全音量播放 |
| 3021 | 不祥物品刷怪笼生成物品 Ominous item spawner spawns item | |

**烟雾方向 Smoke directions:**

| ID | 方向 Direction |
|---|---|
| 0 | 下 Down |
| 1 | 上 Up |
| 2 | 北 North |
| 3 | 南 South |
| 4 | 西 West |
| 5 | 东 East |

### 粒子 Particle

显示指定的粒子效果 Displays the named particle。

**数据包ID Packet ID:** `0x29`  
**状态 State:** Play  
**绑定到 Bound To:** Client 客户端

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| 远距离 Long Distance | Boolean | 如果为true，粒子距离从256增加到65536 If true, particle distance increases from 256 to 65536 |
| 始终可见 Always Visible | Boolean | 此粒子是否应始终可见 Whether this particle should always be visible |
| X | Double | 粒子的X位置 X position of the particle |
| Y | Double | 粒子的Y位置 Y position of the particle |
| Z | Double | 粒子的Z位置 Z position of the particle |
| X偏移量 Offset X | Float | 乘以 `random.nextGaussian()` 后加到X位置 This is added to the X position after being multiplied by random.nextGaussian() |
| Y偏移量 Offset Y | Float | 乘以 `random.nextGaussian()` 后加到Y位置 This is added to the Y position after being multiplied by random.nextGaussian() |
| Z偏移量 Offset Z | Float | 乘以 `random.nextGaussian()` 后加到Z位置 This is added to the Z position after being multiplied by random.nextGaussian() |
| 最大速度 Max Speed | Float | |
| 粒子数量 Particle Count | Int | 要创建的粒子数量 The number of particles to create |
| 粒子ID Particle ID | VarInt | `minecraft:particle_type` 注册表中的ID ID in the minecraft:particle_type registry |
| 数据 Data | Varies | 粒子数据，详见粒子页面 Particle data as specified in Particles |

### 更新光照 Update Light

更新区块的光照级别 Updates light levels for a chunk。

**数据包ID Packet ID:** `0x2A`  
**状态 State:** Play  
**绑定到 Bound To:** Client 客户端

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| 区块X Chunk X | VarInt | 区块坐标（方块坐标除以16，向下取整） Chunk coordinate (block coordinate divided by 16, rounded down) |
| 区块Z Chunk Z | VarInt | 区块坐标（方块坐标除以16，向下取整） Chunk coordinate (block coordinate divided by 16, rounded down) |
| 数据 Data | Light Data | |

一个位永远不会同时在方块光遮罩和空方块光遮罩中设置，尽管它可能在两者中都不存在（如果对应的区块部分不需要更新方块光）。天空光遮罩和空天空光遮罩也是如此 A bit will never be set in both the block light mask and the empty block light mask, though it may be present in neither of them. The same applies to the sky light mask and the empty sky light mask。

### 登录（游戏）Login (play)

**数据包ID Packet ID:** `0x2B`  
**状态 State:** Play  
**绑定到 Bound To:** Client 客户端

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| 实体ID Entity ID | Int | 玩家的实体ID The player's Entity ID (EID) |
| 是否为硬核 Is hardcore | Boolean | |
| 维度名称 Dimension Names | Array of Identifier | 服务器上所有维度的标识符 Identifiers for all dimensions on the server |
| 最大玩家数 Max Players | VarInt | 曾用于客户端绘制玩家列表，现在被忽略 Was once used by the client to draw the player list, but now it is ignored |
| 视距 View Distance | VarInt | 渲染距离（2-32） Render distance (2-32) |
| 模拟距离 Simulation Distance | VarInt | 客户端将处理特定事物（如实体）的距离 The distance that the client will process specific things, such as entities |
| 减少调试信息 Reduced Debug Info | Boolean | 如果为true，原版客户端在调试屏幕上显示减少的信息 If true, a vanilla client shows reduced information on the debug screen |
| 启用重生屏幕 Enable respawn screen | Boolean | 当doImmediateRespawn游戏规则为true时设置为false Set to false when the doImmediateRespawn gamerule is true |
| 限制合成 Do limited crafting | Boolean | 玩家是否只能合成已解锁的配方。目前客户端未使用 Whether players can only craft recipes they have already unlocked. Currently unused by the client |
| 维度类型 Dimension Type | VarInt | `minecraft:dimension_type` 注册表中的维度类型ID，由注册表数据包定义 The ID of the type of dimension in the minecraft:dimension_type registry, defined by the Registry Data packet |
| 维度名称 Dimension Name | Identifier | 正在生成到的维度的名称 Name of the dimension being spawned into |
| 哈希种子 Hashed seed | Long | 世界种子的SHA-256哈希的前8个字节。客户端用于生物群系噪声 First 8 bytes of the SHA-256 hash of the world's seed. Used client-side for biome noise |
| 游戏模式 Game mode | Unsigned Byte | 0: 生存 Survival, 1: 创造 Creative, 2: 冒险 Adventure, 3: 旁观 Spectator |
| 上一个游戏模式 Previous Game mode | Byte | -1: 未定义 Undefined（可能永远不会使用），0: 生存 Survival, 1: 创造 Creative, 2: 冒险 Adventure, 3: 旁观 Spectator |

| 是否为调试 Is Debug | Boolean | 如果世界是调试模式世界则为true；调试模式世界无法修改并具有预定义的方块 True if the world is a debug mode world; debug mode worlds cannot be modified and have predefined blocks |
| 是否为平坦 Is Flat | Boolean | 如果世界是超平坦世界则为true；平坦世界具有不同的虚空迷雾，地平线在y=0而不是y=63 True if the world is a superflat world; flat worlds have different void fog and a horizon at y=0 instead of y=63 |
| 有死亡位置 Has death location | Boolean | 如果为true，则存在接下来的两个字段 If true, then the next two fields are present |
| 死亡维度名称 Death dimension name | Optional Identifier | 玩家死亡所在维度的名称 Name of the dimension the player died in |
| 死亡位置 Death location | Optional Position | 玩家死亡的位置 The location that the player died at |
| 传送门冷却 Portal cooldown | VarInt | 玩家可以再次使用上次使用的传送门之前的刻数。看起来是修复MC-180的尝试 The number of ticks until the player can use the last used portal again. Looks like it's an attempt to fix MC-180 |
| 海平面 Sea level | VarInt | |
| 强制安全聊天 Enforces Secure Chat | Boolean | |

### 地图数据 Map Data

更新地图物品上的矩形区域 Updates a rectangular area on a map item。

**数据包ID Packet ID:** `0x2C`  
**状态 State:** Play  
**绑定到 Bound To:** Client 客户端

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| 地图ID Map ID | VarInt | 正在修改的地图的地图ID Map ID of the map being modified |
| 缩放级别 Scale | Byte | 从0表示完全放大的地图（每像素1个方块）到4表示完全缩小的地图（每像素16个方块） From 0 for a fully zoomed-in map (1 block per pixel) to 4 for a fully zoomed-out map (16 blocks per pixel) |
| 已锁定 Locked | Boolean | 如果地图已在制图台中锁定则为true True if the map has been locked in a cartography table |
| 图标 Icons | Prefixed Optional Prefixed Array | |
| - 类型 Type | VarInt Enum | 见下文 See below |
| - X | Byte | 地图坐标：-128表示最左侧，+127表示最右侧 Map coordinates: -128 for furthest left, +127 for furthest right |
| - Z | Byte | 地图坐标：-128表示最上方，+127表示最下方 Map coordinates: -128 for highest, +127 for lowest |
| - 方向 Direction | Byte | 0-15 |
| - 显示名称 Display Name | Prefixed Optional Text Component | |
| 颜色补丁 Color Patch | | |
| - 列数 Columns | Unsigned Byte | 更新的列数 Number of columns updated |
| - 行数 Rows | Optional Unsigned Byte | 仅当列数大于0时；更新的行数 Only if Columns is more than 0; number of rows updated |
| - X | Optional Unsigned Byte | 仅当列数大于0时；最西列的x偏移 Only if Columns is more than 0; x offset of the westernmost column |
| - Z | Optional Unsigned Byte | 仅当列数大于0时；最北行的z偏移 Only if Columns is more than 0; z offset of the northernmost row |
| - 长度 Length | Optional VarInt | 仅当列数大于0时；以下数组的长度 Only if Columns is more than 0; length of the following array |
| - 数据 Data | Optional Array of Unsigned Byte | 仅当列数大于0时；详见地图物品格式 Only if Columns is more than 0; see Map item format |

对于图标类型 For Icon Type：

| 图标类型 Icon type | 名称 Name |
|---|---|
| 0 | 白色箭头（玩家） White arrow (player) |
| 1 | 绿色箭头（物品展示框） Green arrow (item frame) |
| 2 | 红色箭头 Red arrow |
| 3 | 蓝色箭头 Blue arrow |
| 4 | 白色十字 White cross |
| 5 | 红色指针 Red pointer |
| 6 | 白色圆圈（玩家在附近） White circle (player off map) |
| 7 | 小白色圆圈（远离地图的玩家） Small white circle (far player off map) |
| 8 | 宅邸 Mansion |
| 9 | 纪念碑 Monument |
| 10 | 白色旗帜 White banner |
| 11 | 橙色旗帜 Orange banner |
| 12 | 品红色旗帜 Magenta banner |
| 13 | 淡蓝色旗帜 Light blue banner |
| 14 | 黄色旗帜 Yellow banner |
| 15 | 黄绿色旗帜 Lime banner |
| 16 | 粉红色旗帜 Pink banner |
| 17 | 灰色旗帜 Gray banner |
| 18 | 淡灰色旗帜 Light gray banner |
| 19 | 青色旗帜 Cyan banner |
| 20 | 紫色旗帜 Purple banner |
| 21 | 蓝色旗帜 Blue banner |
| 22 | 棕色旗帜 Brown banner |
| 23 | 绿色旗帜 Green banner |
| 24 | 红色旗帜 Red banner |
| 25 | 黑色旗帜 Black banner |
| 26 | 藏宝点 Treasure marker |
| 27 | 红色X Red X |
| 28 | 村庄沙漠房屋 Village desert house |
| 29 | 村庄平原房屋 Village plains house |
| 30 | 村庄热带草原房屋 Village savanna house |
| 31 | 村庄针叶林房屋 Village taiga house |
| 32 | 村庄针叶林房屋 Village tundra house |
| 33 | 丛林神庙 Jungle temple |
| 34 | 沼泽小屋 Swamp hut |
| 35 | 试炼密室 Trial chambers |

### 商户交易 Merchant Offers

更新村民和流浪商人的交易列表 Updates the trades available from a villager or wandering trader。

**数据包ID Packet ID:** `0x2D`  
**状态 State:** Play  
**绑定到 Bound To:** Client 客户端

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| 窗口ID Window ID | VarInt | 打开的窗口的ID；这是一个int而不是byte The ID of the window that is open; this is an int rather than a byte |
| 交易 Trades | Prefixed Array | |
| - 输入物品1 Input item 1 | Trade Item | 见下文。玩家必须为此村民交易提供的第一个物品。物品堆栈的数量是此交易的默认"价格" See below. The first item the player has to supply for this villager trade. The count of the item stack is the default "price" of this trade |
| - 输出物品 Output item | Slot | 玩家将从此村民交易中获得的物品 The item the player will receive from this villager trade |
| - 输入物品2 Input item 2 | Prefixed Optional Trade Item | 玩家必须为此村民交易提供的第二个物品 The second item the player has to supply for this villager trade |
| - 交易已禁用 Trade disabled | Boolean | 如果交易被禁用则为true；如果交易被启用则为false True if the trade is disabled; false if the trade is enabled |
| - 交易使用次数 Number of trade uses | Int | 到目前为止交易已被使用的次数。如果等于最大交易次数，客户端将显示红色X Number of times the trade has been used so far. If equal to the maximum number of trades, the client will display a red X |
| - 最大交易使用次数 Maximum number of trade uses | Int | 此交易在耗尽之前可以使用的次数 Number of times this trade can be used before it's exhausted |
| - 经验值 XP | Int | 每次使用交易时村民将获得的经验值数量 Amount of XP the villager will earn each time the trade is used |
| - 特殊价格 Special Price | Int | 可以为零或负数。当物品因玩家声望或其他效果而打折时，该数字被添加到价格中 Can be zero or negative. The number is added to the price when an item is discounted due to player reputation or other effects |
| - 价格乘数 Price Multiplier | Float | 可以很低（0.05）或很高（0.2）。决定需求、玩家声望和临时效果将如何调整价格 Can be low (0.05) or high (0.2). Determines how much demand, player reputation, and temporary effects will adjust the price |
| - 需求 Demand | Int | 如果为正数，会导致价格上涨。负值似乎被视为与零相同 If positive, causes the price to increase. Negative values seem to be treated the same as zero |
| 村民等级 Villager level | VarInt | 显示在交易GUI上；含义来自翻译键 merchant.level. + 等级。1: 新手Novice, 2: 学徒Apprentice, 3: 熟练工Journeyman, 4: 专家Expert, 5: 大师Master |
| 经验 Experience | VarInt | 此村民的总经验（对于流浪商人始终为0） Total experience for this villager (always 0 for the wandering trader) |
| 是常规村民 Is regular villager | Boolean | 如果这是常规村民则为true；对于流浪商人为false。当为false时，隐藏村民等级和一些其他GUI元素 True if this is a regular villager; false for the wandering trader. When false, hides the villager level and some other GUI elements |
| 可以补货 Can restock | Boolean | 对于常规村民为true，对于流浪商人为false。如果为true，当悬停在禁用的交易上时会显示"村民每天最多补货两次"的消息 True for regular villagers and false for the wandering trader. If true, the "Villagers restock up to two times per day." message is displayed when hovering over disabled trades |

### 更新实体位置 Update Entity Position

此数据包由服务器发送，当实体移动少于8个方块时 This packet is sent by the server when an entity moves less than 8 blocks。

**数据包ID Packet ID:** `0x2E`  
**状态 State:** Play  
**绑定到 Bound To:** Client 客户端

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| 实体ID Entity ID | VarInt | |
| Delta X | Short | 以（当前X * 32 - 上一个X * 32）* 128的形式改变X坐标 Change in X position as (currentX * 32 - prevX * 32) * 128 |
| Delta Y | Short | 以（当前Y * 32 - 上一个Y * 32）* 128的形式改变Y坐标 Change in Y position as (currentY * 32 - prevY * 32) * 128 |
| Delta Z | Short | 以（当前Z * 32 - 上一个Z * 32）* 128的形式改变Z坐标 Change in Z position as (currentZ * 32 - prevZ * 32) * 128 |
| 在地面上 On Ground | Boolean | |

### 更新实体位置和旋转 Update Entity Position and Rotation

此数据包由服务器发送，当实体旋转并移动时 This packet is sent by the server when an entity rotates and moves。

**数据包ID Packet ID:** `0x2F`  
**状态 State:** Play  
**绑定到 Bound To:** Client 客户端

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| 实体ID Entity ID | VarInt | |
| Delta X | Short | 以（当前X * 32 - 上一个X * 32）* 128的形式改变X坐标 Change in X position as (currentX * 32 - prevX * 32) * 128 |
| Delta Y | Short | 以（当前Y * 32 - 上一个Y * 32）* 128的形式改变Y坐标 Change in Y position as (currentY * 32 - prevY * 32) * 128 |
| Delta Z | Short | 以（当前Z * 32 - 上一个Z * 32）* 128的形式改变Z坐标 Change in Z position as (currentZ * 32 - prevZ * 32) * 128 |
| 偏航角 Yaw | Angle | 新角度，而不是增量 New angle, not a delta |
| 俯仰角 Pitch | Angle | 新角度，而不是增量 New angle, not a delta |
| 在地面上 On Ground | Boolean | |

### 更新实体旋转 Update Entity Rotation

此数据包由服务器发送，当实体旋转时 This packet is sent by the server when an entity rotates。

**数据包ID Packet ID:** `0x30`  
**状态 State:** Play  
**绑定到 Bound To:** Client 客户端

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| 实体ID Entity ID | VarInt | |
| 偏航角 Yaw | Angle | 新角度，而不是增量 New angle, not a delta |
| 俯仰角 Pitch | Angle | 新角度，而不是增量 New angle, not a delta |
| 在地面上 On Ground | Boolean | |

### 移动载具 Move Vehicle

注意，所有字段都使用绝对位置和角度，而不是相对位置 Note that all fields use absolute positioning and do not allow for relative positioning。

**数据包ID Packet ID:** `0x31`  
**状态 State:** Play  
**绑定到 Bound To:** Client 客户端

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| X | Double | 绝对位置（X坐标） Absolute position (X coordinate) |
| Y | Double | 绝对位置（Y坐标） Absolute position (Y coordinate) |
| Z | Double | 绝对位置（Z坐标） Absolute position (Z coordinate) |
| 偏航角 Yaw | Float | 绝对旋转，角度 Absolute rotation on the vertical axis, in degrees |
| 俯仰角 Pitch | Float | 绝对旋转，角度 Absolute rotation on the horizontal axis, in degrees |


#### 打开书 Open Book

由服务器发送以打开客户端当前持有的书。用于提供更好的作弊检测。

| 数据包ID Packet ID | 状态 State | 绑定至 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| *协议 protocol:*<br/>`0x30`<br/><br/>*资源 resource:*<br/>`open_book` | 游戏 Play | 客户端 Client | 手 Hand | VarInt 枚举 Enum | 0: 主手 Main hand, 1: 副手 Off hand |

#### 打开签名编辑器 Open Sign Editor

由服务器发送以打开客户端的告示牌编辑器。仅在玩家放置告示牌时发送，不用于现有告示牌。

| 数据包ID Packet ID | 状态 State | 绑定至 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| rowspan="2" \| *协议 protocol:*<br/>`0x31`<br/><br/>*资源 resource:*<br/>`open_sign_editor` | rowspan="2" \| 游戏 Play | rowspan="2" \| 客户端 Client | 位置 Location | 位置 Position | |
| 是前面 Is Front Text | 布尔值 Boolean | 客户端是否应编辑告示牌的前面（true）还是背面（false）。 Whether the player should edit the front (true) or back (false) of the sign. |

#### Ping（游戏） Ping (play)

数据包不用于保持连接。服务器发送数据包，客户端回复一个相同ID的数据包。如果客户端在30秒内不回复，服务器将断开连接。

| 数据包ID Packet ID | 状态 State | 绑定至 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| *协议 protocol:*<br/>`0x32`<br/><br/>*资源 resource:*<br/>`ping` | 游戏 Play | 客户端 Client | ID | 整数 Int | |

#### Ping 响应（游戏） Ping Response (play)

| 数据包ID Packet ID | 状态 State | 绑定至 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| *协议 protocol:*<br/>`0x33`<br/><br/>*资源 resource:*<br/>`pong_response` | 游戏 Play | 客户端 Client | ID | 整数 Int | ID由服务器传递 ID passed from the server |

#### 放置幽灵方块 Place Ghost Block

| 数据包ID Packet ID | 状态 State | 绑定至 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| rowspan="2" \| *协议 protocol:*<br/>`0x34`<br/><br/>*资源 resource:*<br/>`place_ghost_recipe` | rowspan="2" \| 游戏 Play | rowspan="2" \| 客户端 Client | 窗口ID Window ID | 字节 Byte | |
| 配方 Recipe | 标识符 Identifier | 配方ID Recipe ID |

响应是将配方的输出放入合成网格。

#### 玩家能力（客户端绑定） Player Abilities (clientbound)

后两个字段的vanilla客户端在创造模式下分别使用0.05和0.1，在其他模式下使用0.02和0.08。

| 数据包ID Packet ID | 状态 State | 绑定至 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| rowspan="3" \| *协议 protocol:*<br/>`0x35`<br/><br/>*资源 resource:*<br/>`player_abilities` | rowspan="3" \| 游戏 Play | rowspan="3" \| 客户端 Client | 标志 Flags | 字节 Byte | 位掩码。0x01: 无敌 Invulnerable, 0x02: 飞行 Flying, 0x04: 允许飞行 Allow Flying, 0x08: 即时破坏 Instant Break |
| 飞行速度 Flying Speed | 浮点数 Float | 0.05是默认值 0.05 is default |
| 视野修改器 Field of View Modifier | 浮点数 Float | 根据玩家的速度修改视野。0.1是默认值 Modifies the field of view, like a speed potion. 0.1 is default |

#### 玩家聊天消息 Player Chat Message

从其他玩家发送并由客户端显示在聊天框中。

| 数据包ID Packet ID | 状态 State | 绑定至 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| rowspan="9" \| *协议 protocol:*<br/>`0x36`<br/><br/>*资源 resource:*<br/>`player_chat` | rowspan="9" \| 游戏 Play | rowspan="9" \| 客户端 Client | 发送者 Sender | UUID | 发送此消息的玩家使用 Used by the Notchian client for the disableChat launch option. |
| 索引 Index | VarInt | |
| 消息签名存在 Message Signature Present | 布尔值 Boolean | 指示下一个字段是否存在 States if a message signature is present |
| 消息签名字节 Message Signature bytes | 可选 Optional 字节数组 Byte Array (256) | 仅当消息签名存在为true时存在。签名由发送玩家的会话密钥计算得出 Only present if Message Signature Present is true. Cryptography, the signature consists of the Sender UUID, Session UUID from the Player Session packet, Index, Salt, Timestamp in epoch seconds, the length of the original chat content, the original content itself, the length of Previous Messages, and all of the Previous Message signatures. |
| 正文 Body | 字符串 String (256) | |
| 时间戳 Timestamp | 长整数 Long | 表示消息发送时间（自Unix纪元以来的毫秒数） Represents the time the message was sent as milliseconds since the Unix epoch |
| 盐 Salt | 长整数 Long | 用于验证消息签名的加密盐 Cryptography, used for validating the message signature |
| rowspan="2" \| 先前消息 Previous Messages | 总计 Total | rowspan="2" \| 前置数组 Prefixed Array | VarInt | 最多20条先前消息的签名。消息按发送时间最旧优先的顺序排列 The maximum length is 20 in Notchian client. |
| 消息ID Message ID | VarInt | 签名消息数组中此消息的签名的索引 The message Id + 1, used for validating message signature. |
| 消息签名 Message Signature | 字节数组 Byte Array (256) | 引用消息的签名 The previous message's signature. |
| 无符号内容存在 Unsigned Content Present | 布尔值 Boolean | 如果客户端没有安全聊天，服务器可能会发送没有签名的消息。客户端可以选择不显示它们（此数据包的客户端设置） True if the next field is present |
| 无符号内容 Unsigned Content | 可选 Optional 文本组件 Text Component | |
| 过滤类型 Filter Type | VarInt 枚举 Enum | 如果值为PASS_THROUGH，则不应渲染过滤类型位掩码。如果值为FULLY_FILTERED，则不应渲染消息，仅渲染过滤类型位掩码。如果值为PARTIALLY_FILTERED，则应对应用了过滤类型位掩码的消息进行过滤。 |
| 过滤类型位 Filter Type Bits | 可选 Optional 位集合 BitSet | 仅当前一个字段的值为PARTIALLY_FILTERED时发送。对于每个聊天消息中被审查的字符，此位集合中对应的位被设置。 |
| 聊天类型 Chat Type | VarInt | 注册表中聊天类型的ID The type of chat in the <code>minecraft:chat_type</code> registry, defined by the Registry Data packet. |
| 发送者名称 Sender Name | 文本组件 Text Component | 用于填充聊天类型的sender字段的组件 The component to display as sender in the chat type |
| 目标名称存在 Has Target Name | 布尔值 Boolean | 如果为true，则下一个字段存在 True if the next field is present |
| 目标名称 Target Name | 文本组件 Text Component | 用于填充聊天类型的target字段的组件 |

过滤类型 Filter Type 可以是以下值之一：

| ID | 名称 Name | 说明 Notes |
|---|---|---|
| 0 | PASS_THROUGH | 消息未被过滤 Message is not filtered at all |
| 1 | FULLY_FILTERED | 消息完全被过滤 Message is fully filtered |
| 2 | PARTIALLY_FILTERED | 仅消息的某些部分被过滤 Only some characters in the message are filtered |

#### 结束战斗 End Combat

由服务器发送以指示战斗事件已结束。

| 数据包ID Packet ID | 状态 State | 绑定至 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| *协议 protocol:*<br/>`0x37`<br/><br/>*资源 resource:*<br/>`player_combat_end` | 游戏 Play | 客户端 Client | 持续时间 Duration | VarInt | 战斗持续时间的长度（以刻为单位）Length of the combat in ticks. |

#### 进入战斗 Enter Combat

由服务器发送以指示战斗事件已开始。

| 数据包ID Packet ID | 状态 State | 绑定至 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| *协议 protocol:*<br/>`0x38`<br/><br/>*资源 resource:*<br/>`player_combat_enter` | 游戏 Play | 客户端 Client | 无字段 no fields | | |

#### 战斗死亡 Combat Death

在玩家死亡或观看到玩家因其他原因死亡时从服务器发送（例如，因服务器命令导致的死亡）。

| 数据包ID Packet ID | 状态 State | 绑定至 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| rowspan="2" \| *协议 protocol:*<br/>`0x39`<br/><br/>*资源 resource:*<br/>`player_combat_kill` | rowspan="2" \| 游戏 Play | rowspan="2" \| 客户端 Client | 玩家ID Player ID | VarInt | 死亡玩家的实体ID Entity ID of the player that died (should match the client's entity ID). |
| 消息 Message | 文本组件 Text Component | 死亡消息 The death message |

#### 玩家信息移除 Player Info Remove

由服务器用于从客户端的Tab列表中移除玩家。服务器可以随意添加/移除玩家，但客户端在断开连接时会移除玩家。

| 数据包ID Packet ID | 状态 State | 绑定至 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| *协议 protocol:*<br/>`0x3A`<br/><br/>*资源 resource:*<br/>`player_info_remove` | 游戏 Play | 客户端 Client | 玩家数量 Number of Players | VarInt | 要从玩家列表中移除的玩家数 Number of elements in the following array. |
| 玩家 Player | UUID数组 Array of UUID | 要移除的玩家的UUID UUIDs of players to remove. |

#### 玩家信息更新 Player Info Update

由服务器发送以更新客户端的Tab列表中的玩家信息。

| 数据包ID Packet ID | 状态 State | 绑定至 Bound To | colspan="2" \| 字段名称 Field Name | colspan="2" \| 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|---|
| rowspan="4" \| *协议 protocol:*<br/>`0x3B`<br/><br/>*资源 resource:*<br/>`player_info_update` | rowspan="4" \| 游戏 Play | rowspan="4" \| 客户端 Client | colspan="2" \| 操作 Actions | colspan="2" \| 字节 Byte | 确定要更新的字段的位掩码。详见下文 Determines what actions are present. |
| colspan="2" \| 玩家数量 Number of Players | colspan="2" \| VarInt | 要更新的玩家数 Number of elements in the following array. |
| rowspan="2" \| 玩家 Players | UUID | rowspan="2" \| 数组 Array | UUID | 玩家的UUID The player UUID |
| 玩家操作 Player Actions | 玩家操作 Player Actions | 根据Actions字段确定的操作集合。详见下方的Player Actions。 The set of actions to apply, see below for details. |

操作 Actions 字段确定在数据包中包含哪些Player Actions。它是以下值的位掩码：

| 操作 Action | 位掩码 Bitmask | 说明 Notes |
|---|---|---|
| 添加玩家 ADD PLAYER | 0x01 | 添加玩家；如果玩家列表中已存在此UUID，则用提供的数据替换所有数据 |
| 初始化聊天 INITIALIZE CHAT | 0x02 | |
| 更新游戏模式 UPDATE GAME MODE | 0x04 | |
| 更新列出状态 UPDATE LISTED | 0x08 | |
| 更新延迟 UPDATE LATENCY | 0x10 | |
| 更新显示名称 UPDATE DISPLAY NAME | 0x20 | |

玩家操作 Player Actions 结构取决于操作字段中设置的标志：

| 操作 Action | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|
| 添加玩家 ADD PLAYER | 名称 Name | 字符串 String (16) | |
| 属性数量 Number Of Properties | VarInt | 玩家属性数量 Number of elements in the following array. |
| 属性 Property | 数组 Array | 名称 Name | 字符串 String (32767) | |
| | 值 Value | 字符串 String (32767) | |
| | 是签名 Is Signed | 布尔值 Boolean | |
| | 签名 Signature | 可选 Optional 字符串 String (32767) | 仅当Is Signed为true时存在 Only if Is Signed is true. |
| 初始化聊天 INITIALIZE CHAT | 有签名数据 Has Signature Data | 布尔值 Boolean | |
| 聊天会话ID Chat Session ID | UUID | 仅当Has Signature Data为true时存在 Only sent if Has Signature Data is true. |
| 公钥过期时间 Public key expiry time | 长整数 Long | 密钥过期时间，以纪元毫秒为单位。仅当Has Signature Data为true时发送 Key expiry time, as a UNIX timestamp in milliseconds. Only sent if Has Signature Data is true. |
| 编码的公钥大小 Encoded public key size | VarInt | 以字节为单位的公钥大小。仅当Has Signature Data为true时发送 Size of the following array. Only sent if Has Signature Data is true. Maximum length is 512 bytes. |
| 编码的公钥 Encoded public key | 字节数组 Byte Array (512) | X.509编码的公钥。仅当Has Signature Data为true时发送 The player's public key, in X.509 format. Only sent if Has Signature Data is true. |
| 公钥签名大小 Public key signature size | VarInt | 以字节为单位的公钥签名大小。仅当Has Signature Data为true时发送 Size of the following array. Only sent if Has Signature Data is true. Maximum length is 4096 bytes. |
| 公钥签名 Public key signature | 字节数组 Byte Array (4096) | 公钥数据的签名。仅当Has Signature Data为true时发送 The public key's digital signature. Only sent if Has Signature Data is true. |
| 更新游戏模式 UPDATE GAME MODE | 游戏模式 Game Mode | VarInt | |
| 更新列出状态 UPDATE LISTED | 已列出 Listed | 布尔值 Boolean | 玩家是否应该在Tab列表中显示 Whether the player should be listed on the tab list. |
| 更新延迟 UPDATE LATENCY | Ping | VarInt | 以毫秒为单位测量 Measured in milliseconds. |
| 更新显示名称 UPDATE DISPLAY NAME | 有显示名称 Has Display Name | 布尔值 Boolean | |
| 显示名称 Display Name | 可选 Optional 文本组件 Text Component | 仅当Has Display Name为true时发送 Only sent if Has Display Name is true. |

在游戏模式字段中：生存模式为0，创造模式为1，冒险模式为2，旁观模式为3。

#### 看向 Look At

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| rowspan="8" 0x3B | rowspan="8" Play | rowspan="8" Client | 来自X From X | Double | |
| 来自Y From Y | Double | |
| 来自Z From Z | Double | |
| 目标X At X | Double | |
| 目标Y At Y | Double | |
| 目标Z At Z | Double | |
| 瞄准类型 Aim With | VarInt 枚举 Enum | 0: 脚 feet, 1: 眼睛 eyes |
| 实体ID Entity ID | 可选 Optional VarInt | 要看向的实体 The entity to look at. Only if Aim With is eyes. |

#### 同步玩家位置 Synchronize Player Position

传送玩家的位置和视角。服务器可以根据需要更新位置、视角或两者。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| rowspan="8" 0x3C | rowspan="8" Play | rowspan="8" Client | X | Double | 绝对或相对位置，取决于Flags Absolute or relative position, depending on Flags. |
| Y | Double | 绝对或相对位置，取决于Flags Absolute or relative position, depending on Flags. |
| Z | Double | 绝对或相对位置，取决于Flags Absolute or relative position, depending on Flags. |
| 偏航角 Yaw | Float | 绝对或相对旋转，取决于Flags Absolute or relative rotation, depending on Flags. |
| 俯仰角 Pitch | Float | 绝对或相对旋转，取决于Flags Absolute or relative rotation, depending on Flags. |
| 标志 Flags | Byte | 位掩码。0x01: X是相对的, 0x02: Y是相对的, 0x04: Z是相对的, 0x08: Yaw是相对的, 0x10: Pitch是相对的 Bit mask. 0x01: X is relative, 0x02: Y is relative, 0x04: Z is relative, 0x08: Yaw is relative, 0x10: Pitch is relative. |
| 传送ID Teleport ID | VarInt | 客户端应在传送确认中回复此ID Client should reply with Teleport Confirm containing the same Teleport ID. |

#### 更新配方书 Update Recipe Book

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| rowspan="8" 0x43 | rowspan="8" Play | rowspan="8" Client | 配方数量 Recipe Count | VarInt | 配方数组中的元素数量 Number of elements in the following array. |
| 配方 Recipes | 配方ID Recipe ID | rowspan="6" 前缀数组 Prefixed Array | VarInt | 分配给配方的ID ID to assign to the recipe. |
| 显示 Display | 配方显示 Recipe Display | |
| 组ID Group ID | VarInt | |
| 类别ID Category ID | VarInt | minecraft:recipe_book_category注册表中的ID ID in the minecraft:recipe_book_category registry. |
| 材料 Ingredients | 前缀可选 Prefixed Optional 前缀数组 Prefixed Array of ID集 ID Set | minecraft:item注册表中的ID或内联定义 IDs in the minecraft:item registry, or an inline definition. |
| 标志 Flags | Byte | 0x01: 显示通知 show notification; 0x02: 高亮为新 highlight as new |
| colspan="2" 替换 Replace | colspan="2" 布尔值 Boolean | 替换或添加到已知配方 Replace or Add to known recipes |

#### 配方书移除 Recipe Book Remove

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 0x44 | Play | Client | 配方 Recipes | 前缀数组 Prefixed Array of VarInt | 要移除的配方ID IDs of recipes to remove. |

#### 配方书设置 Recipe Book Settings

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| rowspan="8" 0x45 | rowspan="8" Play | rowspan="8" Client | 工作台配方书打开 Crafting Recipe Book Open | 布尔值 Boolean | 如果为true，当玩家打开背包时工作台配方书将打开 If true, then the crafting recipe book will be open when the player opens its inventory. |
| 工作台配方书过滤器激活 Crafting Recipe Book Filter Active | 布尔值 Boolean | 如果为true，当玩家打开背包时过滤选项将激活 If true, then the filtering option is active when the player opens its inventory. |
| 熔炉配方书打开 Smelting Recipe Book Open | 布尔值 Boolean | 如果为true，当玩家打开熔炉时熔炉配方书将打开 If true, then the smelting recipe book will be open when the player opens its inventory. |
| 熔炉配方书过滤器激活 Smelting Recipe Book Filter Active | 布尔值 Boolean | 如果为true，当玩家打开熔炉时过滤选项将激活 If true, then the filtering option is active when the player opens its inventory. |
| 高炉配方书打开 Blast Furnace Recipe Book Open | 布尔值 Boolean | 如果为true，当玩家打开高炉时高炉配方书将打开 If true, then the blast furnace recipe book will be open when the player opens its inventory. |
| 高炉配方书过滤器激活 Blast Furnace Recipe Book Filter Active | 布尔值 Boolean | 如果为true，当玩家打开高炉时过滤选项将激活 If true, then the filtering option is active when the player opens its inventory. |
| 烟熏炉配方书打开 Smoker Recipe Book Open | 布尔值 Boolean | 如果为true，当玩家打开烟熏炉时烟熏炉配方书将打开 If true, then the smoker recipe book will be open when the player opens its inventory. |
| 烟熏炉配方书过滤器激活 Smoker Recipe Book Filter Active | 布尔值 Boolean | 如果为true，当玩家打开烟熏炉时过滤选项将激活 If true, then the filtering option is active when the player opens its inventory. |

#### 移除实体 Remove Entities

当实体需要在客户端销毁时由服务器发送 Sent by the server when an entity is to be destroyed on the client.

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 0x46 | Play | Client | 实体ID Entity IDs | 前缀数组 Prefixed Array of VarInt | 要销毁的实体列表 The list of entities to destroy. |

#### 移除实体效果 Remove Entity Effect

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| rowspan="2" 0x47 | rowspan="2" Play | rowspan="2" Client | 实体ID Entity ID | VarInt | |
| 效果ID Effect ID | VarInt | 参见状态效果列表 See status effect table. |

#### 重置分数 Reset Score

当客户端应移除记分板项目时发送此数据包 This is sent to the client when it should remove a scoreboard item.

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| rowspan="2" 0x48 | rowspan="2" Play | rowspan="2" Client | 实体名称 Entity Name | 字符串 String (32767) | 拥有此分数的实体。对于玩家，这是他们的用户名；对于其他实体，这是他们的UUID The entity whose score this is. For players, this is their username; for other entities, it is their UUID. |
| 目标名称 Objective Name | 前缀可选 Prefixed Optional 字符串 String (32767) | 分数所属目标的名称 The name of the objective the score belongs to. |

#### 移除资源包（游戏） Remove Resource Pack (play)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 0x49 | Play | Client | UUID | 可选 Optional UUID | 要移除的资源包的UUID The UUID of the resource pack to be removed. |

#### 添加资源包（游戏） Add Resource Pack (play)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| rowspan="5" 0x4A | rowspan="5" Play | rowspan="5" Client | UUID | UUID | 资源包的唯一标识符 The unique identifier of the resource pack. |
| URL | 字符串 String (32767) | 资源包的URL The URL to the resource pack. |
| 哈希 Hash | 字符串 String (40) | 40字符的十六进制，不区分大小写的SHA-1哈希 A 40 character hexadecimal, case-insensitive SHA-1 hash of the resource pack file. |
| 强制 Forced | 布尔值 Boolean | 原版客户端将被强制使用服务器的资源包。如果他们拒绝，将被踢出服务器 The vanilla client will be forced to use the resource pack from the server. If they decline, they will be kicked from the server. |
| 提示消息 Prompt Message | 前缀可选 Prefixed Optional 文本组件 Text Component | 这将显示在提示中，让客户端接受或拒绝资源包 This is shown in the prompt making the client accept or decline the resource pack. |

#### 重生 Respawn

要更改玩家的维度（主世界/下界/末地），向他们发送包含适当维度的重生数据包，然后是新维度的预区块/区块，最后是位置和视角数据包 To change the player's dimension (overworld/nether/end), send them a respawn packet with the appropriate dimension, followed by prechunks/chunks for the new dimension, and finally a position and look packet.

加载屏幕的背景根据此数据包中指定的维度名称和之前登录或重生数据包中指定的维度名称确定 The background of the loading screen is determined based on the Dimension Name specified in this packet and the one specified in the previous Login or Respawn packet.

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| rowspan="13" 0x4B | rowspan="13" Play | rowspan="13" Client | 维度类型 Dimension Type | VarInt | minecraft:dimension_type注册表中的维度类型ID The ID of type of dimension in the minecraft:dimension_type registry. |
| 维度名称 Dimension Name | 标识符 Identifier | 正在生成到的维度名称 Name of the dimension being spawned into. |
| 哈希种子 Hashed seed | Long | 世界种子SHA-256哈希的前8个字节 First 8 bytes of the SHA-256 hash of the world's seed. |
| 游戏模式 Game mode | 无符号字节 Unsigned Byte | 0: 生存 Survival, 1: 创造 Creative, 2: 冒险 Adventure, 3: 旁观 Spectator. |
| 之前的游戏模式 Previous Game mode | 字节 Byte | -1: 未定义 Undefined, 0: 生存 Survival, 1: 创造 Creative, 2: 冒险 Adventure, 3: 旁观 Spectator. |
| 是调试 Is Debug | 布尔值 Boolean | 如果世界是调试模式世界则为true True if the world is a debug mode world. |
| 是平坦 Is Flat | 布尔值 Boolean | 如果世界是超平坦世界则为true True if the world is a superflat world. |
| 有死亡位置 Has death location | 布尔值 Boolean | 如果玩家有死亡位置则为true If true, then the next two fields are present. |
| 死亡维度名称 Death dimension Name | 可选 Optional 标识符 Identifier | 玩家死亡的维度名称 Name of the dimension the player died in. |
| 死亡位置 Death location | 可选 Optional 位置 Position | 玩家死亡的坐标 The location that the player died at. |
| 传送门冷却 Portal cooldown | VarInt | 玩家在使用传送门后无法再次使用传送门的刻数 The number of ticks until the player can use the portal again. |
| 海平面变化 Sea level | VarInt | |
| 数据保持 Data kept | Byte | 位掩码。0x01: 保留属性 Keep attributes, 0x02: 保留元数据 Keep metadata. |


#### 选择进度标签页 Select Advancements Tab

由服务器发送以指示客户端应切换进度标签页。在客户端在GUI中切换标签页或在另一个标签页中取得进度时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x4E`<br/>资源 resource: `select_advancements_tab` | 游戏 Play | 客户端 Client | 标识符 Identifier | 前缀可选 Prefixed Optional 标识符 Identifier | 见下文 See below. |

如果没有加载自定义数据包，标识符必须是以下之一：

| 标识符 Identifier |
|---|
| minecraft:story/root |
| minecraft:nether/root |
| minecraft:end/root |
| minecraft:adventure/root |
| minecraft:husbandry/root |

如果发送了无效的标识符或未发送标识符，客户端将切换到GUI中的第一个标签页。

#### 服务器数据 Server Data

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x4F`<br/>资源 resource: `server_data` | 游戏 Play | 客户端 Client | MOTD | 文本组件 Text Component | |
| | | | 图标 Icon | 前缀可选 Prefixed Optional 前缀数组 Prefixed Array of 字节 Byte | PNG格式的图标字节 Icon bytes in the PNG format. |

#### 设置动作栏文本 Set Action Bar Text

在快捷栏上方显示消息。等同于叠加层设置为true的系统聊天消息，但不执行聊天消息阻止。仅由原版服务器用于实现 `/title` 命令。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x50`<br/>资源 resource: `set_action_bar_text` | 游戏 Play | 客户端 Client | 动作栏文本 Action bar text | 文本组件 Text Component | |

#### 设置边界中心 Set Border Center

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x51`<br/>资源 resource: `set_border_center` | 游戏 Play | 客户端 Client | X | 双精度浮点数 Double | |
| | | | Z | 双精度浮点数 Double | |

#### 设置边界插值大小 Set Border Lerp Size

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x52`<br/>资源 resource: `set_border_lerp_size` | 游戏 Play | 客户端 Client | 旧直径 Old Diameter | 双精度浮点数 Double | 世界边界单边的当前长度（以米为单位） Current length of a single side of the world border, in meters. |
| | | | 新直径 New Diameter | 双精度浮点数 Double | 世界边界单边的目标长度（以米为单位） Target length of a single side of the world border, in meters. |
| | | | 速度 Speed | VarLong | 到达新直径所需的实时毫秒数 Number of real-time milliseconds until New Diameter is reached. 原版服务器似乎不会将世界边界速度同步到游戏刻，因此它会与服务器延迟不同步。如果世界边界没有移动，则设置为0。 |

#### 设置边界大小 Set Border Size

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x53`<br/>资源 resource: `set_border_size` | 游戏 Play | 客户端 Client | 直径 Diameter | 双精度浮点数 Double | 世界边界单边的长度（以米为单位） Length of a single side of the world border, in meters. |

#### 设置边界警告延迟 Set Border Warning Delay

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x54`<br/>资源 resource: `set_border_warning_delay` | 游戏 Play | 客户端 Client | 警告时间 Warning Time | VarInt | 以秒为单位，由 `/worldborder warning time` 设置 In seconds as set by `/worldborder warning time`. |

#### 设置边界警告距离 Set Border Warning Distance

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x55`<br/>资源 resource: `set_border_warning_distance` | 游戏 Play | 客户端 Client | 警告方块数 Warning Blocks | VarInt | 以米为单位 In meters. |

#### 设置摄像机 Set Camera

设置玩家渲染视角所在的实体。通常在玩家在旁观模式下左键单击实体时使用。

玩家的摄像机将随实体移动并查看它所看的方向。该实体通常是另一个玩家，但可以是任何类型的实体。玩家无法移动该实体（移动数据包将被视为来自另一个实体）。

如果给定的实体未被玩家加载，则忽略此数据包。要将控制权返回给玩家，请使用其实体ID发送此数据包。

原版服务器会在旁观的实体被杀死或玩家潜行时重置此设置（将其发送回默认实体），但仅当他们正在旁观实体时才会这样做。它还会在玩家切换出旁观模式时发送此数据包（即使他们没有旁观实体）。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x56`<br/>资源 resource: `set_camera` | 游戏 Play | 客户端 Client | 摄像机ID Camera ID | VarInt | 要将客户端摄像机设置到的实体ID ID of the entity to set the client's camera to. |

原版客户端还会为给定实体加载某些着色器：

- 苦力怕 Creeper → `shaders/post/creeper.json`
- 蜘蛛 Spider（和洞穴蜘蛛 cave spider）→ `shaders/post/spider.json`
- 末影人 Enderman → `shaders/post/invert.json`
- 其他任何东西 → 当前着色器被卸载

#### 设置中心区块 Set Center Chunk

设置客户端区块加载区域的中心位置。该区域为正方形，在两个轴上跨越 2 × 服务器视距 + 7 个区块（宽度，而非半径！）。由于区域的宽度始终为奇数，因此对于哪个区块是中心没有歧义。

原版客户端永远不会渲染或模拟位于加载区域外的区块，但会将它们保留在内存中（除非在仍在范围内时被服务器明确卸载），并且仅当另一个区块以与旧区块坐标模 (2 × 服务器视距 + 7) 全等的坐标加载时才会自动卸载区块。这意味着区块可能在离开并随后通过连续使用此数据包重新进入加载区域后重新出现，除非在此期间被同一"槽位"中的不同区块替换。

原版客户端会忽略尝试加载或卸载位于加载区域外的区块。即使针对仍然加载但当前位于加载区域外的区块的卸载操作也适用（按照上一段）。

原版服务器不依赖于离开加载区域的区块的任何特定行为，自定义客户端无需完全复制上述内容。客户端可以选择立即卸载加载区域外的任何区块，使用不同的模数，或完全忽略加载区域并保持区块加载，无论其位置如何，直到服务器请求卸载它们。以最大互操作性为目标的服务器应始终在区块超出加载区域之前明确卸载任何已加载的区块。

中心区块通常是玩家所在的区块，但除了对区块加载的影响外，（原版）客户端对此不是这种情况不会有问题。实际上，只要区块仅在以世界原点为中心的默认加载区域内发送，就根本不需要发送此数据包。这对于具有小型有界世界的服务器（例如小游戏）可能很有用，因为它确保在客户端加入后永远不需要重新发送区块，从而节省带宽。

每当玩家水平移动穿过区块边界时，原版服务器都会发送此数据包，并且（根据测试）对于垂直轴的任何整数变化也是如此，即使它不跨越区块截面边界。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x57`<br/>资源 resource: `set_chunk_cache_center` | 游戏 Play | 客户端 Client | 区块X Chunk X | VarInt | 加载区域中心的区块X坐标 Chunk X coordinate of the loading area center. |
| | | | 区块Z Chunk Z | VarInt | 加载区域中心的区块Z坐标 Chunk Z coordinate of the loading area center. |

#### 设置渲染距离 Set Render Distance

在更改渲染距离时由集成的单人服务器发送。当客户端离开末地后重新出现在主世界时，服务器会发送此数据包。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x58`<br/>资源 resource: `set_chunk_cache_radius` | 游戏 Play | 客户端 Client | 视距 View Distance | VarInt | 渲染距离 (2-32) Render distance (2-32). |

#### 设置光标物品 Set Cursor Item

替换或设置用鼠标拖动的物品栏物品。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x59`<br/>资源 resource: `set_cursor_item` | 游戏 Play | 客户端 Client | 槽位数据 Slot Data | 槽位 Slot | |

#### 玩家旋转 Player Rotation

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x42`<br/>资源 resource: `player_rotation` | 游戏 Play | 客户端 Client | 偏航角 Yaw | Float | X轴上的旋转，以度为单位 Rotation on the X axis, in degrees. |
| | | | 俯仰角 Pitch | Float | Y轴上的旋转，以度为单位 Rotation on the Y axis, in degrees. |

#### 配方书添加 Recipe Book Add

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x43`<br/>资源 resource: `recipe_book_add` | 游戏 Play | 客户端 Client | 配方 Recipes | 前缀数组 Prefixed Array | |
| | | | - 配方ID Recipe ID | VarInt | 要分配给配方的ID ID to assign to the recipe. |
| | | | - 显示 Display | Recipe Display | |
| | | | - 组ID Group ID | VarInt | |
| | | | - 类别ID Category ID | VarInt | `minecraft:recipe_book_category` 注册表中的ID ID in the `minecraft:recipe_book_category` registry. |
| | | | - 材料 Ingredients | 前缀可选 Prefixed Optional 前缀数组 Prefixed Array of ID Set | `minecraft:item` 注册表中的ID，或内联定义 IDs in the `minecraft:item` registry, or an inline definition. |
| | | | - 标志 Flags | Byte | 0x01: 显示通知 show notification; 0x02: 高亮为新 highlight as new |
| | | | 替换 Replace | Boolean | 替换或添加到已知配方 Replace or Add to known recipes |

#### 设置默认重生位置 Set Default Spawn Position

设置玩家死亡后重生的坐标。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x5A`<br/>资源 resource: `set_default_spawn_position` | 游戏 Play | 客户端 Client | 位置 Location | Position | 重生位置 Spawn location. |
| | | | 角度 Angle | Float | 重生时面向的角度 The angle at which to respawn at. |

#### 显示死亡画面 Set Display Death Screen

当玩家死亡时不显示死亡画面（客户端会重生到其床/重生锚）。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x5B`<br/>资源 resource: `set_display_objective` | 游戏 Play | 客户端 Client | 位置 Position | VarInt | 计分板位置 The position of the scoreboard. 0: 列表 list, 1: 侧边栏 sidebar, 2: 名字下方 below name, 3 - 18: 团队特定侧边栏 team specific sidebar (计算为 3 + 团队颜色ID 3 + team color id). |
| | | | 计分板名称 Score Name | String (32767) | 要显示的唯一计分板名称 The unique name for the scoreboard to be displayed. |

#### 设置实体元数据 Set Entity Metadata

更新一个或多个实体的元数据字段。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x5C`<br/>资源 resource: `set_entity_data` | 游戏 Play | 客户端 Client | 实体ID Entity ID | VarInt | |
| | | | 元数据 Metadata | 实体元数据 Entity Metadata | |

#### 链接实体 Link Entities

此数据包从服务器发送到客户端时，客户端将绘制链接实体的牵引绳/约束线。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x5D`<br/>资源 resource: `set_entity_link` | 游戏 Play | 客户端 Client | 被链接实体ID Attached Entity ID | Int | 被链接实体的EID Attached entity's EID. |
| | | | 链接对象ID Holding Entity ID | Int | 持有牵引绳的实体的EID EID of the entity holding the lead. 设置为-1以分离 Set to -1 to detach. |

#### 设置实体速度 Set Entity Velocity

实体的速度以每刻 1/8000 方块为单位。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x5E`<br/>资源 resource: `set_entity_motion` | 游戏 Play | 客户端 Client | 实体ID Entity ID | VarInt | |
| | | | 速度X Velocity X | Short | X轴上的速度 Velocity on the X axis. |
| | | | 速度Y Velocity Y | Short | Y轴上的速度 Velocity on the Y axis. |
| | | | 速度Z Velocity Z | Short | Z轴上的速度 Velocity on the Z axis. |

#### 设置装备 Set Equipment

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x5F`<br/>资源 resource: `set_equipment` | 游戏 Play | 客户端 Client | 实体ID Entity ID | VarInt | 装备所有者的实体ID Entity ID of the entity. |
| | | | 装备 Equipment | 数组 Array | 装备槽位和物品数据 Equipment slot and item. 数组以槽位字节为键，其最高有效位（0x80）设置为表示后面跟着另一个槽位 The array is keyed by the slot byte, with its top significant bit (0x80) set to indicate that there is another slot following. 最后一个槽位的该位未设置，表示这是最后一个槽位 On the last slot, that bit is not set, indicating that this is the final slot. |
| | | | - 槽位 Slot | Byte | 装备槽位 Equipment slot. 0: 主手 main hand, 1: 副手 off hand, 2-5: 盔甲槽位（2: 靴子 boots, 3: 护腿 leggings, 4: 胸甲 chestplate, 5: 头盔 helmet），6: 身体 body. 见下文 See below. 还要注意最高有效位，如上所述 Also note the top significant bit as described above. |
| | | | - 物品 Item | 槽位 Slot | |

装备槽位值 Equipment slot values:

| ID | 装备槽位 Equipment Slot |
|---|---|
| 0 | 主手 Main hand |
| 1 | 副手 Off hand |
| 2 | 靴子 Boots |
| 3 | 护腿 Leggings |
| 4 | 胸甲 Chestplate |
| 5 | 头盔 Helmet |
| 6 | 身体 Body |

#### 设置经验 Set Experience

发送给玩家每当经验球被收集或总经验以任何方式改变时。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x60`<br/>资源 resource: `set_experience` | 游戏 Play | 客户端 Client | 经验值 Experience Bar | Float | 介于0和1之间 Between 0 and 1. |
| | | | 等级 Level | VarInt | |
| | | | 总经验 Total Experience | VarInt | 见经验条目 See Experience. |

#### 设置生命值 Set Health

发送给客户端以设置或更新玩家的生命值。

食物饱和度充当食物等级的缓冲区。食物饱和度不会显示给客户端，并且通常仅稍高于食物等级。如果在一项动作需要食物时食物饱和度大于零，食物饱和度将减少而不是食物等级。有关详细信息，请参阅 Minecraft Wiki 文章。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x61`<br/>资源 resource: `set_health` | 游戏 Play | 客户端 Client | 生命值 Health | Float | 0或更少 = 死亡 0 or less = dead, 20 = 满血 full HP. |
| | | | 食物 Food | VarInt | 0-20. |
| | | | 食物饱和度 Food Saturation | Float | 似乎可以变为负数并大于5 Seems to vary from 0.0 to 5.0 in integer increments. |

#### 更新目标 Update Objectives

这通过计分板系统发送给客户端。它通常用于在客户端创建计分板或进行内容更改。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x62`<br/>资源 resource: `set_objective` | 游戏 Play | 客户端 Client | 目标名称 Objective Name | String (32767) | 一个唯一的名称 A unique name for the objective. |
| | | | 模式 Mode | Byte | 0: 创建计分板 create the scoreboard. 1: 移除计分板 remove the scoreboard. 2: 更新显示文本 update the display text. |
| | | | 目标值 Objective Value | 可选 Optional Text Component | 仅当模式为0或2时 Only if mode is 0 or 2. 要显示的文本 The text to be displayed for the score. |
| | | | 类型 Type | 可选 Optional VarInt Enum | 仅当模式为0或2时 Only if mode is 0 or 2. 0 = "整数" "integer", 1 = "心形" "hearts". |
| | | | 数字格式 Number Format | 可选 Optional VarInt Enum | 仅当模式为0或2时 Only if mode is 0 or 2. 0 = 空白 blank, 1 = 样式化 styled, 2 = 固定 fixed. |
| | | | 数字格式字段 Number Format Field | 变化 Varies | 取决于数字格式类型，见下文 Depends on number format type, see below. |

数字格式 Number Format：空白显示无内容，样式化使用计分板的默认样式，固定显示给定文本。

#### 设置乘客 Set Passengers

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x63`<br/>资源 resource: `set_passengers` | 游戏 Play | 客户端 Client | 实体ID Entity ID | VarInt | 载具的实体ID Vehicle's EID. |
| | | | 乘客 Passengers | 前缀数组 Prefixed Array of VarInt | 乘客的实体ID Passenger EIDs. |

#### 更新团队 Update Teams

创建和更新团队。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x64`<br/>资源 resource: `set_player_team` | 游戏 Play | 客户端 Client | 团队名称 Team Name | String (32767) | 一个唯一的名称 A unique name for the team. (在计分板实现中与目标名称共享命名空间；建议使用绝对或相对命名空间限定名称来避免冲突 Shared with objectives; command suggestions will likely show an overlap if you don't use a unique namespace.) |
| | | | 模式 Mode | Byte | 决定其余数据包的布局 Determines the layout of the remaining packet. 0: 创建团队 create team. 1: 移除团队 remove team. 2: 更新团队信息 update team info. 3: 添加实体到团队 add entities to team. 4: 从团队移除实体 remove entities from team. |
| | | | 团队显示名称 Team Display Name | 可选 Optional Text Component | 仅当模式为0或2时 Only if Mode is 0 or 2. |
| | | | 友好标志 Friendly Flags | 可选 Optional Byte | 仅当模式为0或2时 Only if Mode is 0 or 2. 位掩码 Bit mask. 0x01: 允许友军伤害 Allow friendly fire, 0x02: 可以看到隐形队友 Can see invisible teammates. |
| | | | 名称标签可见性 Name Tag Visibility | 可选 Optional String Enum (40) | 仅当模式为0或2时 Only if Mode is 0 or 2. `always`, `hideForOtherTeams`, `hideForOwnTeam`, `never`. |
| | | | 碰撞规则 Collision Rule | 可选 Optional String Enum (40) | 仅当模式为0或2时 Only if Mode is 0 or 2. `always`, `pushOtherTeams`, `pushOwnTeam`, `never`. |
| | | | 团队颜色 Team Color | 可选 Optional VarInt Enum | 仅当模式为0或2时 Only if Mode is 0 or 2. 用于为团队和玩家着色 Used to color the name of players on the team; see below for values. |
| | | | 团队前缀 Team Prefix | 可选 Optional Text Component | 仅当模式为0或2时 Only if Mode is 0 or 2. 显示在团队成员名称之前 Displayed before the names of entities on the team. |
| | | | 团队后缀 Team Suffix | 可选 Optional Text Component | 仅当模式为0或2时 Only if Mode is 0 or 2. 显示在团队成员名称之后 Displayed after the names of entities on the team. |
| | | | 实体 Entities | 可选 Optional 前缀数组 Prefixed Array of String (32767) | 仅当模式为0、3或4时 Only if Mode is 0, 3 or 4. 要添加/移除的实体的标识符 Identifiers for the entities in this team. 对于玩家，这是他们的用户名；对于其他实体，这是他们的UUID For players, this is their username; for other entities, it is their UUID. |

团队颜色：与聊天颜色的ID相同。

#### 更新分数 Update Score

当项目的分数更新或移除时发送到客户端。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x65`<br/>资源 resource: `set_score` | 游戏 Play | 客户端 Client | 实体名称 Entity Name | String (32767) | 此分数所属的实体 The entity whose score this is. 对于玩家，这是他们的用户名；对于其他实体，这是他们的UUID For players, this is their username; for other entities, it is their UUID. |
| | | | 目标名称 Objective Name | String (32767) | 分数所属的目标的名称 The name of the objective the score belongs to. |
| | | | 值 Value | VarInt | 要显示的分数 The score to be displayed next to the entry. |
| | | | 显示名称 Display Name | 可选 Optional Text Component | 用于自定义条目的显示名称 The custom display name. |
| | | | 数字格式 Number Format | 可选 Optional VarInt Enum | 与 Update Objectives 中的相同 Same as Update Objectives. |

#### 设置模拟距离 Set Simulation Distance

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x66`<br/>资源 resource: `set_simulation_distance` | 游戏 Play | 客户端 Client | 模拟距离 Simulation Distance | VarInt | 服务器模拟的距离，以区块为单位 The distance that the client will process specific things, such as entities. |

#### 设置副标题文本 Set Subtitle Text

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x67`<br/>资源 resource: `set_subtitle_text` | 游戏 Play | 客户端 Client | 副标题文本 Subtitle Text | Text Component | |

#### 更新时间 Update Time

时间以刻为单位；一天有20刻/秒。世界年龄以刻为单位；没有相关计算，只是累加器。

世界年龄不与日间循环挂钩，而是每刻递增，无论时间是否被冻结。时间表示是日间循环。0是日出，6000是中午，12000是日落，18000是午夜，23999是日出的一刻前。负值时间将冻结循环。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x68`<br/>资源 resource: `set_time` | 游戏 Play | 客户端 Client | 世界年龄 World Age | Long | 以刻为单位；不会被服务器命令修改 In ticks; not changed by server commands. |
| | | | 时间 Time of day | Long | 世界时间，以刻为单位 The world (or region) time, in ticks. 如果为负，太阳将停在天空中的绝对值 If negative the sun will stop moving at the Math.abs of the time. |

#### 设置标题文本 Set Title Text

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x69`<br/>资源 resource: `set_title_text` | 游戏 Play | 客户端 Client | 标题文本 Title Text | Text Component | |

#### 设置标题动画时间 Set Title Animation Times

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x6A`<br/>资源 resource: `set_titles_animation` | 游戏 Play | 客户端 Client | 淡入 Fade In | Int | 以刻为单位的淡入时间 Ticks to spend fading in. |
| | | | 停留 Stay | Int | 以刻为单位的停留时间 Ticks to keep the title displayed. |
| | | | 淡出 Fade Out | Int | 以刻为单位的淡出时间 Ticks to spend fading out, not when to start fading out. |

#### 实体声音效果 Entity Sound Effect

播放与实体绑定的声音效果。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x6B`<br/>资源 resource: `sound_entity` | 游戏 Play | 客户端 Client | 声音ID Sound ID | ID或 Sound Event | `minecraft:sound_event` 注册表中的声音事件ID，或内联定义 ID in the `minecraft:sound_event` registry, or an inline definition. |
| | | | 声音类别 Sound Category | VarInt Enum | 声音的类别，见下文 The category that this sound will be played from. |
| | | | 实体ID Entity ID | VarInt | |
| | | | 音量 Volume | Float | 1.0是100%，可以更多 1.0 is 100%, capped between 0.0 and 1.0 by vanilla clients. |
| | | | 音调 Pitch | Float | 音调的浮点表示形式 Float between 0.5 and 2.0 by Vanilla clients. |
| | | | 种子 Seed | Long | 用于为客户端播放声音时提供种子 Seed used to pick sound variant. |

声音类别 Sound Categories：

| ID | 名称 Name |
|---|---|
| 0 | 主音量 master |
| 1 | 音乐 music |
| 2 | 唱片 record |
| 3 | 天气 weather |
| 4 | 方块 block |
| 5 | 敌对生物 hostile |
| 6 | 友好生物 neutral |
| 7 | 玩家 player |
| 8 | 环境 ambient |
| 9 | 语音 voice |

#### 声音效果 Sound Effect

用于播放声音效果。所有已知的声音效果位置在原版客户端的源代码中的 sounds.json 中定义（修改此文件或使用资源包覆盖它可以添加自定义声音；未来的版本可能会为此目的添加注册表）。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x6C`<br/>资源 resource: `sound` | 游戏 Play | 客户端 Client | 声音ID Sound ID | ID或 Sound Event | `minecraft:sound_event` 注册表中的声音事件ID，或内联定义 ID in the `minecraft:sound_event` registry, or an inline definition. |
| | | | 声音类别 Sound Category | VarInt Enum | 声音的类别，见上文 The category that this sound will be played from (see Sound Categories). |
| | | | 效果位置X Effect Position X | Int | 效果位置乘以8（固定点数，与32有关的分数）Effect position (X coordinate) multiplied by 8 (fixed-point number with only 3 bits dedicated to the fractional part). |
| | | | 效果位置Y Effect Position Y | Int | 效果位置乘以8（固定点数，与32有关的分数）Effect position (Y coordinate) multiplied by 8 (fixed-point number with only 3 bits dedicated to the fractional part). |
| | | | 效果位置Z Effect Position Z | Int | 效果位置乘以8（固定点数，与32有关的分数）Effect position (Z coordinate) multiplied by 8 (fixed-point number with only 3 bits dedicated to the fractional part). |
| | | | 音量 Volume | Float | 1.0是100%，可以更多 1.0 is 100%, capped between 0.0 and 1.0 by Vanilla clients. |
| | | | 音调 Pitch | Float | 音调的浮点表示形式 Float between 0.5 and 2.0 by Vanilla clients. |
| | | | 种子 Seed | Long | 用于为客户端播放声音时提供种子 Seed used to pick sound variant. |

#### 开始配置 Start Configuration

由服务器发送以从游戏状态切换到配置状态。

原版客户端会发送 Acknowledge Finish Configuration 数据包来确认配置开始。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x6D`<br/>资源 resource: `start_configuration` | 游戏 Play | 客户端 Client | 无字段 no fields | | |

#### 停止声音 Stop Sound

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x6E`<br/>资源 resource: `stop_sound` | 游戏 Play | 客户端 Client | 标志 Flags | Byte | 控制后续字段 Controls the following two fields. 0x01: 是否存在声音 Should Source be included, 0x02: 是否存在声音名称 Should Sound be included. |
| | | | 来源 Source | 可选 Optional VarInt Enum | 仅当标志设置了相应位时 Only if flags is 3 or 1 (bit 0 set). 见 Sound Categories. |
| | | | 声音 Sound | 可选 Optional Identifier | 仅当标志设置了相应位时 Only if flags is 2 or 3 (bit 1 set). 声音的名称 A sound effect name. |

#### 存储Cookie Store Cookie

由服务器发送以存储客户端上的数据。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x6F`<br/>资源 resource: `store_cookie` | 游戏 Play | 客户端 Client | 键 Key | Identifier | 要存储的cookie的标识符 The identifier of the cookie. |
| | | | 载荷长度 Payload Length | VarInt | 以下载荷的长度，以字节为单位 Length of the following byte array. |
| | | | 载荷 Payload | Byte Array (5120) | 要存储的cookie的数据 The data of the cookie. |

#### 系统聊天消息 System Chat Message

向客户端发送系统聊天消息。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x70`<br/>资源 resource: `system_chat` | 游戏 Play | 客户端 Client | 内容 Content | Text Component | 有限的聊天格式化 Limited to 262144 bytes. |
| | | | 覆盖 Overlay | Boolean | 是否将消息显示在游戏上的覆盖层（hotbar上方）中 Whether the message is an actionbar or chat message. true: 游戏信息（hotbar）actionbar, false: 系统消息 system message. |

#### 设置Tab列表页眉和页脚 Set Tab List Header And Footer

此数据包可以由服务器发送以设置显示在玩家列表（通过按Tab键访问）上方和下方的文本。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x71`<br/>资源 resource: `tab_list` | 游戏 Play | 客户端 Client | 页眉 Header | Text Component | 要显示在玩家列表上方 To display above the player list. |
| | | | 页脚 Footer | Text Component | 要显示在玩家列表下方 To display below the player list. |

#### 标签查询响应 Tag Query Response

由服务器发送以响应 Query Block Entity Tag 或 Query Entity Tag。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x72`<br/>资源 resource: `tag_query` | 游戏 Play | 客户端 Client | 事务ID Transaction ID | VarInt | 可以与发送的查询数据包进行比较 Can be compared to the one sent in the original query packet. |
| | | | NBT | NBT | 数据 The NBT of the block or entity. 如果没有可用的标签，则可能为空 May be a TAG_END (0) in which case no NBT is present. |

#### 拾取物品 Pickup Item

由服务器发送当有人拾取地面上的物品时。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x73`<br/>资源 resource: `take_item_entity` | 游戏 Play | 客户端 Client | 收集的实体ID Collected Entity ID | VarInt | |
| | | | 收集者实体ID Collector Entity ID | VarInt | |
| | | | 拾取物品数量 Pickup Item Count | VarInt | 似乎不被原版客户端使用 Seems to be 1 for XP orbs, otherwise the number of items in the stack. |

#### 传送实体 Teleport Entity

此数据包由服务器发送以将实体传送到新位置（尽管名称如此）。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x74`<br/>资源 resource: `teleport_entity` | 游戏 Play | 客户端 Client | 实体ID Entity ID | VarInt | |
| | | | X | Double | |
| | | | Y | Double | |
| | | | Z | Double | |
| | | | 偏航角 Yaw | Angle | 新角度，而不是增量 New angle, not a delta. |
| | | | 俯仰角 Pitch | Angle | 新角度，而不是增量 New angle, not a delta. |
| | | | 在地面上 On Ground | Boolean | |

#### 设置刻状态 Set Ticking State

用于调整客户端刻的速率。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x75`<br/>资源 resource: `set_ticking_state` | 游戏 Play | 客户端 Client | 刻率 Tick Rate | Float | |
| | | | 是否冻结 Is Frozen | Boolean | |

#### 步进刻 Step Tick

将世界时间提前给定数量的刻。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x76`<br/>资源 resource: `tick_step` | 游戏 Play | 客户端 Client | 刻步数 Tick Steps | VarInt | |

#### 传送实体移动 Transfer (play)

通知客户端连接到另一台服务器。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x77`<br/>资源 resource: `transfer` | 游戏 Play | 客户端 Client | 主机 Host | String | 要连接到的主机名或IP The hostname of IP of the server. |
| | | | 端口 Port | VarInt | 要连接到的端口 The port of the server. |

#### 更新进度 Update Advancements

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x78`<br/>资源 resource: `update_advancements` | 游戏 Play | 客户端 Client | 是否重置/清除 Reset/Clear | Boolean | 是否重置/清除现有的进度数据 Whether to reset/clear the current advancements. |
| | | | 进度映射 Advancement mapping | 前缀数组 Prefixed Array | 进度的键值对 Key-value pairs of advancements. |
| | | | - 键 Key | Identifier | 进度标识符 The identifier of the advancement. |
| | | | - 值 Value | 进度 Advancement | 进度数据 Advancement data (see below). |
| | | | 已移除标识符列表 List of identifiers | 前缀数组 Prefixed Array of Identifier | 要移除的进度标识符 The identifiers of the advancements that should be removed. |
| | | | 进度列表 Progress | 前缀数组 Prefixed Array | 进度进度数据 Progress on the advancements. |
| | | | - 键 Key | Identifier | 进度的标识符 The identifier of the advancement. |
| | | | - 值 Value | 进度进度 Advancement progress | 进度数据（见下文）Progress data (see below). |

进度结构 Advancement structure：

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| 是否有父级 Has parent | Boolean | 指示接下来是否有父级ID Indicates whether the next field exists. |
| 父级ID Parent id | 可选 Optional Identifier | 此进度的父级标识符 The identifier of the parent advancement. |
| 是否有显示 Has display | Boolean | 指示接下来是否有显示数据 Indicates whether the next field exists. |
| 显示数据 Display data | 可选 Optional Advancement Display | 见下文 (see below). |
| 数组长度 Array length | VarInt | 奖励数组中的元素数 Number of elements in the following array. |
| 奖励 Rewards | Identifier数组 Array of Identifier | 使用此进度获得的战利品表 The loot tables that are rewarded with this advancement. |
| 是否发送遥测事件 Send telemetry event | Boolean | |

进度显示 Advancement Display：

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| 标题 Title | Text Component | |
| 描述 Description | Text Component | |
| 图标 Icon | 槽位 Slot | |
| 框架类型 Frame type | VarInt Enum | 0 = `task`, 1 = `challenge`, 2 = `goal`. |
| 标志 Flags | Int | 0x01: 有背景纹理 has background texture; 0x02: `show_toast`; 0x04: `hidden`. |
| 背景纹理 Background texture | 可选 Optional Identifier | 仅当标志指示存在背景纹理时 Only if flags indicates it. |
| X坐标 X coord | Float | |
| Y坐标 Y coord | Float | |

进度进度 Advancement Progress：

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| 大小 Size | VarInt | 标准数组长度 Size of the following array. |
| 标准 Criteria | 标准数组 Array of Criterion | 每个标准的进度 Progress on each individual criterion. |

标准 Criterion：

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| 标准标识符 Criterion identifier | Identifier | 标准的标识符 The identifier of the criterion. |
| 标准进度 Criterion progress | 标准进度 Criterion Progress | 此特定标准的进度 The progress on this particular criterion. |

标准进度 Criterion Progress：

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| 是否已达成 Achieved | Boolean | 如果为真，则下一个字段存在 If true, next field is present. |
| 达成日期 Date of achieving | 可选 Optional Long | 作为Unix时间戳的达成日期 As returned by Date.getTime. |

#### 更新属性 Update Attributes

设置给定实体的属性。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x79`<br/>资源 resource: `update_attributes` | 游戏 Play | 客户端 Client | 实体ID Entity ID | VarInt | |
| | | | 属性 Attributes | 前缀数组 Prefixed Array | 属性和相关修改器的列表 List of attributes and their modifiers. |
| | | | - 属性ID Attribute ID | ID或 Attribute | `minecraft:attribute` 注册表中的属性ID，或内联定义 ID in the `minecraft:attribute` registry, or an inline definition. |
| | | | - 值 Value | Double | |
| | | | - 修改器数量 Number of Modifiers | VarInt | 后续数组的长度 Length of the following array. |
| | | | - 修改器 Modifiers | 属性修改器数组 Array of Attribute Modifier | 见下文 (see below). |

属性修改器 Attribute Modifier structure：

| 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|
| ID | Identifier | |
| 数量 Amount | Double | 可以是负数 May be negative. |
| 操作 Operation | Byte | 见下文 (see below). |

操作字段 Operation field：

| ID | 操作 Operation |
|---|---|
| 0 | 加值到总和 Add value to sum |
| 1 | 乘值到总和 Multiply sum by (1 + value) |
| 2 | 乘值到积 Multiply product by (1 + value) |

#### 实体效果 Entity Effect

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x7A`<br/>资源 resource: `update_mob_effect` | 游戏 Play | 客户端 Client | 实体ID Entity ID | VarInt | |
| | | | 效果ID Effect ID | VarInt | 见状态效果列表 See Status Effect. |
| | | | 放大器 Amplifier | VarInt | 药水效果等级减1 Notchian client displays effect level as Amplifier + 1. |
| | | | 持续时间 Duration | VarInt | 持续时间，以刻为单位 Duration in ticks. (-1 for infinite) |
| | | | 标志 Flags | Byte | 位字段 Bit field, see below. |
| | | | 是否有因子数据 Has Factor Data | Boolean | 当 Effect ID 为 32 (RAID_OMEN) 时使用 Used for effect ID 32 (RAID_OMEN). |
| | | | 因子计算数据 Factor Codec | 可选 Optional NBT | 见下文 See below. |

标志 Flags：

| 位 Bit | 效果 Effect |
|---|---|
| 0x01 | 是环境效果 Is ambient |
| 0x02 | 显示粒子 Show particles |
| 0x04 | 显示图标 Show icon |
| 0x08 | 混合 Blend |

因子数据 Factor Data（NBT标签）：

```
TAG_Compound: none
  TAG_Int: padding_duration
  TAG_Float: factor_start
  TAG_Float: factor_target
  TAG_Float: factor_current
  TAG_Int: effect_changed_timestamp
  TAG_Float: factor_previous_frame
  TAG_Boolean: had_effect_last_tick
```

#### 更新配方 Update Recipes

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x7B`<br/>资源 resource: `update_recipes` | 游戏 Play | 客户端 Client | 配方数量 Num Recipes | VarInt | 后续数组中的元素数 Number of elements in the following array. |
| | | | 配方 Recipes | 配方数组 Array of Recipe | |
| | | | - 配方ID Recipe ID | VarInt | |
| | | | - 配方数据 Recipe Data | Recipe | |

#### 更新标签 Update Tags (play)

与配置阶段的 Update Tags 数据包相同。

---

以上为客户端绑定 Play 数据包的完整翻译。

### 服务器绑定 Serverbound

#### 确认传送 Confirm Teleportation

由客户端发送，作为同步玩家位置 Synchronize Player Position 数据包的确认。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x00`<br/>资源 resource: `accept_teleportation` | 游戏 Play | 服务器 Server | 传送ID Teleport ID | VarInt | 由同步玩家位置 Synchronize Player Position 数据包给出的ID。 |

#### 查询方块实体标签 Query Block Entity Tag

在查看方块时按下 F3+I 时使用。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x01`<br/>资源 resource: `block_entity_tag_query` | 游戏 Play | 服务器 Server | 事务ID Transaction ID | VarInt | 递增ID，以便客户端可以验证响应是否匹配。 |
| | | | 位置 Location | Position | 要检查的方块的位置。 |

#### 捆绑物品已选择 Bundle Item Selected

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x02`<br/>资源 resource: `bundle_item_selected` | 游戏 Play | 服务器 Server | 槽位ID Slot ID | VarInt | 捆绑中的槽位。 |
| | | | 选中的物品ID Selected Item ID | VarInt | 捆绑中的物品。 |

#### 更改难度 Change Difficulty

必须启用难度锁定才会有效。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x03`<br/>资源 resource: `change_difficulty` | 游戏 Play | 服务器 Server | 新难度 New difficulty | Byte | 0: 和平 peaceful, 1: 简单 easy, 2: 普通 normal, 3: 困难 hard。 |

#### 确认消息 Acknowledge Message

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x04`<br/>资源 resource: `chat_ack` | 游戏 Play | 服务器 Server | 消息计数 Message Count | VarInt | |

#### 聊天命令 Chat Command

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x05`<br/>资源 resource: `chat_command` | 游戏 Play | 服务器 Server | 命令 Command | String (256) | 命令（不含斜杠 `/`）。 |

#### 已签名聊天命令 Signed Chat Command

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x06`<br/>资源 resource: `signed_chat_command` | 游戏 Play | 服务器 Server | 命令 Command | String (256) | 命令（不含斜杠 `/`）。 |
| | | | 时间戳 Timestamp | Long | 自Unix纪元以来的毫秒数。 |
| | | | 盐 Salt | Long | 用于验证签名哈希的盐。 |
| | | | 参数签名数组长度 Array Length | VarInt | 参数签名的最大数量为8。 |
| | | | 参数名称 Argument Name | String (16) | 命令的参数名称。 |
| | | | 签名 Signature | Byte Array (256) | 参数值的加密签名。 |
| | | | 已确认消息计数 Message Count | VarInt | |
| | | | 已确认 Acknowledged | Fixed BitSet (20) | |

#### 聊天消息 Chat Message

用于发送客户端聊天消息。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x07`<br/>资源 resource: `chat` | 游戏 Play | 服务器 Server | 消息 Message | String (256) | |

#### 玩家会话 Player Session

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x08`<br/>资源 resource: `chat_session_update` | 游戏 Play | 服务器 Server | 会话ID Session ID | UUID | |
| | | | 公钥过期时间 Public Key Expiry Time | Long | Unix时间戳（毫秒）。 |
| | | | 编码公钥大小 Encoded Public Key Size | VarInt | 以下字节数组的大小。最大长度为512字节。 |
| | | | 编码公钥 Encoded Public Key | Byte Array (512) | X.509编码的公钥。 |
| | | | 公钥签名大小 Public Key Signature Size | VarInt | 以下字节数组的大小。最大长度为4096字节。 |
| | | | 公钥签名 Public Key Signature | Byte Array (4096) | 由Mojang的YggdrasilSessionPublicKey签名的公钥数据和过期时间。 |

#### 区块批次已接收 Chunk Batch Received

通知服务器客户端已完成接收上一个区块批次。服务器使用此信息来调整未来批次发送的速率。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x09`<br/>资源 resource: `chunk_batch_received` | 游戏 Play | 服务器 Server | 批次中区块数 Chunks per tick | Float | 期望的区块每刻发送速率。 |

#### 客户端状态 Client Status

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x0A`<br/>资源 resource: `client_command` | 游戏 Play | 服务器 Server | 动作ID Action ID | VarInt Enum | 0: 执行重生 perform respawn, 1: 请求统计信息 request stats。 |

#### 客户端信息(游戏) Client Information (play)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x0B`<br/>资源 resource: `client_information` | 游戏 Play | 服务器 Server | 语言 Locale | String (16) | 例如 en_GB。 |
| | | | 视距 View Distance | Byte | 客户端渲染距离，以区块为单位。 |
| | | | 聊天模式 Chat Mode | VarInt Enum | 0: 启用 enabled, 1: 仅命令 commands only, 2: 隐藏 hidden。 |
| | | | 聊天颜色 Chat Colors | Boolean | 客户端是否有聊天颜色。 |
| | | | 显示的皮肤部件 Displayed Skin Parts | Unsigned Byte | 位掩码，详见下文。 |
| | | | 主手 Main Hand | VarInt Enum | 0: 左手 Left, 1: 右手 Right。 |
| | | | 启用文本过滤 Enable text filtering | Boolean | 在服务器上启用文本过滤。 |
| | | | 允许服务器列表 Allow server listings | Boolean | 服务器列表是否在多人游戏菜单中显示服务器。 |

显示的皮肤部件 Displayed Skin Parts 字段的位标志如下：

- 位0(0x01): 披风 Cape enabled
- 位1(0x02): 夹克 Jacket enabled  
- 位2(0x04): 左袖子 Left Sleeve enabled
- 位3(0x08): 右袖子 Right Sleeve enabled
- 位4(0x10): 左裤腿 Left Pants Leg enabled
- 位5(0x20): 右裤腿 Right Pants Leg enabled
- 位6(0x40): 帽子 Hat enabled

#### 命令建议请求 Command Suggestions Request

tab自动补全命令时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x0C`<br/>资源 resource: `command_suggestion` | 游戏 Play | 服务器 Server | 事务ID Transaction Id | VarInt | 递增计数器。 |
| | | | 文本 Text | String (32500) | 正在编辑的所有文本。 |

#### 确认配置 Acknowledge Configuration

由客户端发送，以响应开始配置 Start Configuration。此数据包切换连接状态到配置 configuration。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x0D`<br/>资源 resource: `configuration_acknowledged` | 游戏 Play | 服务器 Server | 无字段 no fields | | |

#### 点击容器按钮 Click Container Button

在附魔台、信标、织布机、切石机、锻造台、酿造台、铁砧或讲台中使用。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x0E`<br/>资源 resource: `container_button_click` | 游戏 Play | 服务器 Server | 窗口ID Window ID | Byte | 打开窗口的ID。 |
| | | | 按钮ID Button ID | Byte | 含义取决于容器类型。 |

#### 点击容器 Click Container

此数据包在玩家与容器的槽位进行交互时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x0F`<br/>资源 resource: `container_click` | 游戏 Play | 服务器 Server | 窗口ID Window ID | Unsigned Byte | 窗口的ID。 |
| | | | 状态ID State ID | VarInt | 服务器发送的最后一个Set Container Content的ID。 |
| | | | 槽位 Slot | Short | 被点击的槽位。 |
| | | | 按钮 Button | Byte | 使用的鼠标按钮。 |
| | | | 模式 Mode | VarInt Enum | 点击类型。 |
| | | | 改变的槽位长度 Length of the array | VarInt | 后续数组的最大值为128。 |
| | | | - 槽位编号 Slot number | Short | |
| | | | - 槽位数据 Slot data | Slot | 客户端假设此槽位的新数据。 |
| | | | 携带的物品 Carried item | Slot | 鼠标携带的物品。必须为空（物品ID = -1）对于创造模式。 |

#### 关闭容器(服务器绑定) Close Container (serverbound)

当玩家关闭容器时，服务器会发送此数据包。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x10`<br/>资源 resource: `container_close` | 游戏 Play | 服务器 Server | 窗口ID Window ID | Unsigned Byte | 这是服务器发送的窗口ID。 |

#### 更改容器槽位状态 Change Container Slot State

此数据包在玩家更改容器槽位的状态时发送（在crafter中）。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x11`<br/>资源 resource: `container_slot_state_changed` | 游戏 Play | 服务器 Server | 槽位ID Slot ID | VarInt | 槽位的ID。 |
| | | | 窗口ID Window ID | VarInt | 窗口的ID。 |
| | | | 状态 State | Boolean | 新状态：true为启用，false为禁用。 |

#### Cookie响应(游戏) Cookie Response (play)

对Cookie请求(游戏) Cookie Request (play)的响应。响应可能会在以后的时间点发送。响应必须有相同的标识符作为请求。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x12`<br/>资源 resource: `cookie_response` | 游戏 Play | 服务器 Server | 键 Key | Identifier | Cookie的标识符。 |
| | | | 有效载荷长度 Has Payload | Boolean | Cookie的有效载荷是否存在。 |
| | | | 有效载荷长度 Payload Length | Optional VarInt | 有效载荷的长度（字节）。 |
| | | | 有效载荷 Payload | Optional Byte Array (5120) | Cookie的有效载荷，如果存在。 |

#### 服务器绑定插件消息(游戏) Serverbound Plugin Message (play)

主要文章: 插件通道 Plugin channels

允许向服务器发送可选信息。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x13`<br/>资源 resource: `custom_payload` | 游戏 Play | 服务器 Server | 通道 Channel | Identifier | 用于发送数据的通道名称。 |
| | | | 数据 Data | Byte Array (32767) | 任何数据。 |

#### 调试样本订阅 Debug Sample Subscription

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x14`<br/>资源 resource: `debug_sample_subscription` | 游戏 Play | 服务器 Server | 样本类型 Sample Type | VarInt Enum | 要订阅的调试样本类型。0: 刻时间 Tick time。 |

#### 编辑书 Edit Book

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x15`<br/>资源 resource: `edit_book` | 游戏 Play | 服务器 Server | 槽位 Slot | VarInt | 拿着书的槽位编号。0表示主手，1表示副手。 |
| | | | 条目数量 Count | VarInt | 后续数组的元素数。最多为200。 |
| | | | 条目 Entries | String (8192) 数组 Array of String (8192) | 文本从每页。 |
| | | | 有标题 Has title | Boolean | 如果为true，则有下一个字段。 |
| | | | 标题 Title | Optional String (128) | 新书的标题。 |

#### 查询实体标签 Query Entity Tag

在对实体按下F3+I时使用。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x16`<br/>资源 resource: `entity_tag_query` | 游戏 Play | 服务器 Server | 事务ID Transaction ID | VarInt | 递增整数，以便客户端可以验证响应是否匹配。 |
| | | | 实体ID Entity ID | VarInt | 要查询的实体的ID。 |

#### 交互 Interact

当玩家与实体交互时发送此数据包。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x17`<br/>资源 resource: `interact` | 游戏 Play | 服务器 Server | 实体ID Entity ID | VarInt | 交互的实体ID。 |
| | | | 类型 Type | VarInt Enum | 0: 交互 interact, 1: 攻击 attack, 2: 在特定位置交互 interact at。 |
| | | | 目标X Target X | Optional Float | 仅当Type为在特定位置交互时。 |
| | | | 目标Y Target Y | Optional Float | 仅当Type为在特定位置交互时。 |
| | | | 目标Z Target Z | Optional Float | 仅当Type为在特定位置交互时。 |
| | | | 手 Hand | Optional VarInt Enum | 仅当Type为交互或在特定位置交互时。0: 主手 main hand, 1: 副手 off hand。 |
| | | | 潜行 Sneaking | Boolean | 玩家是否潜行。 |

#### 拼图生成 Jigsaw Generate

服务器发送生成的完成结构后，通过更新拼图方块 Update Jigsaw Block 来回复。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x18`<br/>资源 resource: `jigsaw_generate` | 游戏 Play | 服务器 Server | 位置 Location | Position | 拼图方块位置。 |
| | | | 级别 Levels | VarInt | 要生成的级别数。 |
| | | | 保留拼图 Keep Jigsaws | Boolean | |

#### 服务器绑定保持连接(游戏) Serverbound Keep Alive (play)

服务器在客户端通过超时断开连接之前会等待客户端发送相同的keep alive包。服务器将每30秒尝试发送一次Keep Alive包。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x19`<br/>资源 resource: `keep_alive` | 游戏 Play | 服务器 Server | Keep Alive ID | Long | |

#### 锁定难度 Lock Difficulty

必须启用难度锁定才会生效。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x1A`<br/>资源 resource: `lock_difficulty` | 游戏 Play | 服务器 Server | 已锁定 Locked | Boolean | |

#### 设置玩家位置 Set Player Position

对x, y或z值的改变大于8个区块时，服务器会认为玩家在作弊并踢出玩家。同时，如果y或z的固定点移动距离超过3.2个区块，则服务器也会认为玩家在作弊并踢出玩家。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x1B`<br/>资源 resource: `move_player_pos` | 游戏 Play | 服务器 Server | X | Double | 玩家的绝对位置。 |
| | | | 脚Y Feet Y | Double | 玩家的脚的绝对位置，通常是头Y - 1.62。 |
| | | | Z | Double | 玩家的绝对位置。 |
| | | | 在地面上 On Ground | Boolean | 玩家是否在地面上。 |

#### 设置玩家位置和旋转 Set Player Position and Rotation

对x, y或z值的改变大于8个区块时，服务器会认为玩家在作弊并踢出玩家。同时，如果y或z的固定点移动距离超过3.2个区块，则服务器也会认为玩家在作弊并踢出玩家。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x1C`<br/>资源 resource: `move_player_pos_rot` | 游戏 Play | 服务器 Server | X | Double | 玩家的绝对位置。 |
| | | | 脚Y Feet Y | Double | 玩家的脚的绝对位置，通常是头Y - 1.62。 |
| | | | Z | Double | 玩家的绝对位置。 |
| | | | 偏航角 Yaw | Float | 玩家左右转头的绝对旋转角度，以度为单位。 |
| | | | 俯仰角 Pitch | Float | 玩家上下转头的绝对旋转角度，以度为单位。 |
| | | | 在地面上 On Ground | Boolean | 玩家是否在地面上。 |

#### 设置玩家旋转 Set Player Rotation

更新玩家视角方向。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x1D`<br/>资源 resource: `move_player_rot` | 游戏 Play | 服务器 Server | 偏航角 Yaw | Float | 玩家左右转头的绝对旋转角度，以度为单位。 |
| | | | 俯仰角 Pitch | Float | 玩家上下转头的绝对旋转角度，以度为单位。 |
| | | | 在地面上 On Ground | Boolean | 玩家是否在地面上。 |

#### 设置玩家在地面 Set Player On Ground

此数据包以及设置玩家位置 Set Player Position, 设置玩家旋转 Set Player Rotation, 和设置玩家位置和旋转 Set Player Position and Rotation 被称为"移动数据包"。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x1E`<br/>资源 resource: `move_player_status_only` | 游戏 Play | 服务器 Server | 在地面上 On Ground | Boolean | 玩家是否在地面上。 |

#### 移动载具(服务器绑定) Move Vehicle (serverbound)

当玩家移动载具时发送；另见移动载具(客户端绑定) Move Vehicle (clientbound)。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x1F`<br/>资源 resource: `move_vehicle` | 游戏 Play | 服务器 Server | X | Double | 载具的绝对位置。 |
| | | | Y | Double | 载具的绝对位置。 |
| | | | Z | Double | 载具的绝对位置。 |
| | | | 偏航角 Yaw | Float | 载具的绝对旋转角度，以度为单位。 |
| | | | 俯仰角 Pitch | Float | 载具的绝对旋转角度，以度为单位。 |

#### 划船 Paddle Boat

用于在没有控制杆的情况下挥舞手臂/移动船桨。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x20`<br/>资源 resource: `paddle_boat` | 游戏 Play | 服务器 Server | 左桨转动 Left paddle turning | Boolean | |
| | | | 右桨转动 Right paddle turning | Boolean | |

#### 拾取物品 Pick Item

用于在创造模式物品栏中交换主手和副手槽位中的物品。在创造模式物品栏界面中使用选取方块(中键)时使用。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x21`<br/>资源 resource: `pick_item` | 游戏 Play | 服务器 Server | 槽位待使用 Slot to use | VarInt | 另请参阅物品栏 Inventory。 |

#### Ping请求(游戏) Ping Request (play)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x22`<br/>资源 resource: `ping_request` | 游戏 Play | 服务器 Server | 时间 Time | Long | 毫秒时间戳。 |

#### 放置配方 Place Recipe

当玩家在带有配方书的GUI(例如合成台)中按下绿色配方按钮时，会发送此数据包。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x23`<br/>资源 resource: `place_recipe` | 游戏 Play | 服务器 Server | 窗口ID Window ID | Byte | |
| | | | 配方 Recipe | Identifier | 配方ID。 |
| | | | 制作所有 Make all | Boolean | 尽可能多地制作配方。 |

#### 玩家能力(服务器绑定) Player Abilities (serverbound)

玩家能力的可修改字段是飞行标志。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x24`<br/>资源 resource: `player_abilities` | 游戏 Play | 服务器 Server | 标志 Flags | Byte | 位字段，见下文。 |

标志 Flags:

| 字段 Field | 位 Bit |
|---|---|
| 飞行中 Flying | 0x02 |

#### 玩家动作 Player Action

当玩家挖掘、取消挖掘或完成挖掘时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x25`<br/>资源 resource: `player_action` | 游戏 Play | 服务器 Server | 状态 Status | VarInt Enum | 要执行的操作，见下文。 |
| | | | 位置 Location | Position | 方块位置。 |
| | | | 面 Face | Byte Enum | 被挖掘的方块的面。 |
| | | | 序列 Sequence | VarInt | |

状态 Status可以是以下之一:

| 值 Value | 含义 Meaning | 说明 Notes |
|---|---|---|
| 0 | 开始挖掘方块 Started digging | 玩家开始挖掘。 |
| 1 | 取消挖掘 Cancelled digging | 玩家在完成挖掘之前取消。 |
| 2 | 完成挖掘方块 Finished digging | 告诉服务器完成了挖掘。 |
| 3 | 丢弃物品堆 Drop item stack | 丢弃整个物品堆。 |
| 4 | 丢弃物品 Drop item | 从物品堆丢弃一个物品。 |
| 5 | 释放使用物品 Release use item | 标示玩家完成使用物品（例如弓箭）。 |
| 6 | 交换手中物品 Swap item in hand | 主手和副手交换。 |

面 Face是玩家挖掘的方块的面，可以是以下之一:

| 值 Value | 偏移量 Offset | 面 Face |
|---|---|---|
| 0 | -Y | 底部 Bottom |
| 1 | +Y | 顶部 Top |
| 2 | -Z | 北 North |
| 3 | +Z | 南 South |
| 4 | -X | 西 West |
| 5 | +X | 东 East |

#### 玩家命令 Player Command

当玩家开始/停止飞行或潜行时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x26`<br/>资源 resource: `player_command` | 游戏 Play | 服务器 Server | 实体ID Entity ID | VarInt | 玩家ID。 |
| | | | 动作ID Action ID | VarInt Enum | 要执行的动作，见下文。 |
| | | | 跳跃加成 Jump Boost | VarInt | 仅在骑马时使用，范围从0到100。 |

动作ID Action ID可以是以下之一:

| ID | 动作 Action |
|---|---|
| 0 | 开始潜行 Start sneaking |
| 1 | 停止潜行 Stop sneaking |
| 2 | 离开床 Leave bed |
| 3 | 开始疾跑 Start sprinting |
| 4 | 停止疾跑 Stop sprinting |
| 5 | 开始骑马跳跃 Start jump with horse |
| 6 | 停止骑马跳跃 Stop jump with horse |
| 7 | 打开载具物品栏 Open vehicle inventory |
| 8 | 开始飞行鞘翅 Start flying with elytra |

#### 玩家输入 Player Input

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x27`<br/>资源 resource: `player_input` | 游戏 Play | 服务器 Server | 侧向 Sideways | Float | 正值表示向左。 |
| | | | 前进 Forward | Float | 正值表示向前。 |
| | | | 标志 Flags | Unsigned Byte | 位掩码。0x1: 跳跃 jump, 0x2: 卸载 unmount。 |

#### Pong(游戏) Pong (play)

对Ping Request(play) Ping Request(play)的响应。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x28`<br/>资源 resource: `pong` | 游戏 Play | 服务器 Server | ID | Int | 来自请求的相应ID。 |

#### 更改配方书设置 Change Recipe Book Settings

替换已弃用的配方书数据 Recipe Book Data，没有菜单。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x29`<br/>资源 resource: `recipe_book_change_settings` | 游戏 Play | 服务器 Server | 书本ID Book ID | VarInt Enum | 0: 合成 crafting, 1: 熔炉 furnace, 2: 高炉 blast furnace, 3: 烟熏炉 smoker。 |
| | | | 书本打开 Book Open | Boolean | |
| | | | 过滤开启 Filter Active | Boolean | |

#### 设置已查看配方 Set Seen Recipe

在配方书中点击配方时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x2A`<br/>资源 resource: `recipe_book_seen_recipe` | 游戏 Play | 服务器 Server | 配方ID Recipe ID | Identifier | |

#### 重命名物品 Rename Item

当在铁砧UI中重命名物品时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x2B`<br/>资源 resource: `rename_item` | 游戏 Play | 服务器 Server | 物品名称 Item name | String (32767) | 铁砧中物品的新名称。 |

#### 资源包响应(服务器绑定) Resource Pack Response (serverbound)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x2C`<br/>资源 resource: `resource_pack` | 游戏 Play | 服务器 Server | UUID | UUID | 资源包的唯一标识符。 |
| | | | 结果 Result | VarInt Enum | 0: 成功加载 successfully downloaded, 1: 拒绝 declined, 2: 下载失败 failed to download, 3: 接受 accepted, 4: 无效URL invalid URL, 5: 重新加载失败 failed to reload, 6: 丢弃 discarded。 |

#### 已查看进度 Seen Advancements

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x2D`<br/>资源 resource: `seen_advancements` | 游戏 Play | 服务器 Server | 动作 Action | VarInt Enum | 0: 打开的标签 Opened tab, 1: 关闭屏幕 Closed screen。 |
| | | | 标签ID Tab ID | Optional Identifier | 仅当动作 Action 为打开的标签 Opened tab 时存在。 |

#### 选择交易 Select Trade

当玩家选择商户的特定交易选项时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x2E`<br/>资源 resource: `select_trade` | 游戏 Play | 服务器 Server | 选中的槽位 Selected slot | VarInt | 选中的交易，从0开始。 |

#### 设置信标效果 Set Beacon Effect

修改信标 Beacon 提供的效果。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x2F`<br/>资源 resource: `set_beacon` | 游戏 Play | 服务器 Server | 有主效果 Has Primary Effect | Boolean | |
| | | | 主效果 Primary Effect | Optional VarInt | 主要效果的ID；如果Has Primary Effect为false，则不发送。 |
| | | | 有次效果 Has Secondary Effect | Boolean | |
| | | | 次效果 Secondary Effect | Optional VarInt | 次要效果的ID；如果Has Secondary Effect为false，则不发送。 |

#### 设置手持物品(服务器绑定) Set Held Item (serverbound)

在玩家更改物品栏中选中的槽位时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x30`<br/>资源 resource: `set_carried_item` | 游戏 Play | 服务器 Server | 槽位 Slot | Short | 玩家选中的槽位(0-8)。 |

#### 编程命令方块 Program Command Block

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x31`<br/>资源 resource: `set_command_block` | 游戏 Play | 服务器 Server | 位置 Location | Position | |
| | | | 命令 Command | String (32767) | |
| | | | 模式 Mode | VarInt Enum | 0: 序列 SEQUENCE, 1: 自动 AUTO, 2: 红石 REDSTONE。 |
| | | | 标志 Flags | Byte | 0x01: 跟踪输出 Track Output (如果为false, 命令方块输出将被清除); 0x02: 条件 Is conditional; 0x04: 自动 Automatic。 |

#### 编程命令方块矿车 Program Command Block Minecart

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x32`<br/>资源 resource: `set_command_minecart` | 游戏 Play | 服务器 Server | 实体ID Entity ID | VarInt | |
| | | | 命令 Command | String (32767) | |
| | | | 跟踪输出 Track Output | Boolean | 如果为false, 命令方块输出将被清除。 |

#### 设置创造模式槽位 Set Creative Mode Slot

当玩家处于创造模式并点击物品栏上的槽位时，服务器可能会发送此数据包以更新物品栏。服务器将仅为创造模式物品栏的槽位(1-45)发送此数据包。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x33`<br/>资源 resource: `set_creative_mode_slot` | 游戏 Play | 服务器 Server | 槽位 Slot | Short | 物品栏槽位。 |
| | | | 点击的物品 Clicked Item | Slot | |

#### 编程拼图方块 Program Jigsaw Block

由客户端发送以更新拼图方块 Jigsaw Block 的字段。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x34`<br/>资源 resource: `set_jigsaw_block` | 游戏 Play | 服务器 Server | 位置 Location | Position | 正在更新的拼图方块的位置。 |
| | | | 名称 Name | Identifier | |
| | | | 目标 Target | Identifier | |
| | | | 池 Pool | Identifier | |
| | | | 最终状态 Final state | String (32767) | "minecraft:air"表示删除拼图。 |
| | | | 关节类型 Joint type | String (32767) | "rollable"表示拼图可以旋转；"aligned"表示拼图必须对齐。 |
| | | | 选择优先级 Selection priority | VarInt | |
| | | | 放置优先级 Placement priority | VarInt | |

#### 编程结构方块 Program Structure Block

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x35`<br/>资源 resource: `set_structure_block` | 游戏 Play | 服务器 Server | 位置 Location | Position | 正在更新的结构方块的位置。 |
| | | | 动作 Action | VarInt Enum | 更新动作: 0表示更新数据, 1表示保存结构, 2表示加载结构, 3表示检测大小。 |
| | | | 模式 Mode | VarInt Enum | 0: 保存 SAVE, 1: 加载 LOAD, 2: 角落 CORNER, 3: 数据 DATA。 |
| | | | 名称 Name | String (32767) | |
| | | | 偏移X Offset X | Byte | -48到48之间。 |
| | | | 偏移Y Offset Y | Byte | -48到48之间。 |
| | | | 偏移Z Offset Z | Byte | -48到48之间。 |
| | | | 大小X Size X | Byte | 0到48之间。 |
| | | | 大小Y Size Y | Byte | 0到48之间。 |
| | | | 大小Z Size Z | Byte | 0到48之间。 |
| | | | 镜像 Mirror | VarInt Enum | 0: 无 NONE, 1: 左右 LEFT_RIGHT, 2: 前后 FRONT_BACK。 |
| | | | 旋转 Rotation | VarInt Enum | 0: 无 NONE, 1: 顺时针90 CLOCKWISE_90, 2: 顺时针180 CLOCKWISE_180, 3: 逆时针90 COUNTERCLOCKWISE_90。 |
| | | | 元数据 Metadata | String (128) | |
| | | | 完整性 Integrity | Float | 0到1之间。 |
| | | | 种子 Seed | VarLong | |
| | | | 标志 Flags | Byte | 0x01: 忽略实体 Ignore entities; 0x02: 显示空气 Show air; 0x04: 显示边界框 Show bounding box。 |

#### 更新告示牌 Update Sign

此消息由客户端发送以更新告示牌的文本，或者在按下放置告示牌的"完成"按钮后发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x36`<br/>资源 resource: `sign_update` | 游戏 Play | 服务器 Server | 位置 Location | Position | 正在更新的告示牌的方块坐标。 |
| | | | 是前面 Is Front Text | Boolean | 正在更新的文字是否在告示牌的前面。 |
| | | | 第1行 Line 1 | String (384) | 第一行文字。 |
| | | | 第2行 Line 2 | String (384) | 第二行文字。 |
| | | | 第3行 Line 3 | String (384) | 第三行文字。 |
| | | | 第4行 Line 4 | String (384) | 第四行文字。 |

#### 挥动手臂 Swing Arm

当玩家的手臂动画时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x37`<br/>资源 resource: `swing` | 游戏 Play | 服务器 Server | 手 Hand | VarInt Enum | 手: 0表示主手, 1表示副手。 |

#### 传送到实体 Teleport To Entity

在观察者模式下传送到另一个实体。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x38`<br/>资源 resource: `teleport_to_entity` | 游戏 Play | 服务器 Server | 目标玩家 Target Player | UUID | 要传送到的玩家的UUID(也可以是实体)。 |

#### 在...上使用物品 Use Item On

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x39`<br/>资源 resource: `use_item_on` | 游戏 Play | 服务器 Server | 手 Hand | VarInt Enum | 用于放置方块的手: 0表示主手, 1表示副手。 |
| | | | 位置 Location | Position | 方块位置。 |
| | | | 面 Face | VarInt Enum | 光标所在的方块面。 |
| | | | 光标位置X Cursor Position X | Float | 光标在方块上的位置，从0到1。 |
| | | | 光标位置Y Cursor Position Y | Float | 光标在方块上的位置，从0到1。 |
| | | | 光标位置Z Cursor Position Z | Float | 光标在方块上的位置，从0到1。 |
| | | | 方块内部 Inside block | Boolean | 如果为true, 光标在方块的边界框内。 |
| | | | 世界边界 World border hit | Boolean | 如果放置会与世界边界碰撞则为true。在这种情况下，不会放置任何方块。 |
| | | | 序列 Sequence | VarInt | |

#### 使用物品 Use Item

由客户端发送以告诉服务器玩家使用了物品。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x3A`<br/>资源 resource: `use_item` | 游戏 Play | 服务器 Server | 手 Hand | VarInt Enum | 手: 0表示主手, 1表示副手。 |
| | | | 序列 Sequence | VarInt | |
| | | | 偏航角 Yaw | Float | 玩家的偏航角（以度为单位）。 |
| | | | 俯仰角 Pitch | Float | 玩家的俯仰角（以度为单位）。 |

---

**完整的协议文档翻译已完成！Minecraft Java版协议的所有核心部分（握手、状态、登录、配置、游戏的客户端绑定和服务器绑定数据包）已全部翻译为中文，并保持双语格式。**

### 服务器绑定 Serverbound

#### 确认传送 Confirm Teleportation

客户端发送此数据包作为对同步玩家位置 Synchronize Player Position 的确认。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x00`<br/>资源 resource: `accept_teleportation` | 游戏 Play | 服务器 Server | 传送ID Teleport ID | VarInt | 由同步玩家位置数据包给出的ID The ID given by the Synchronize Player Position packet. |

#### 查询方块实体标签 Query Block Entity Tag

当按下F3+I键并看着一个方块时使用。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x01`<br/>资源 resource: `block_entity_tag_query` | 游戏 Play | 服务器 Server | 事务ID Transaction ID | VarInt | 一个递增的ID，以便客户端可以验证响应是否匹配 An incremental ID so that the client can verify that the response matches. |
| | | | 位置 Location | Position | 要检查的方块的位置 The location of the block to check. |

#### 捆绑物品已选择 Bundle Item Selected

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x02`<br/>资源 resource: `bundle_item_selected` | 游戏 Play | 服务器 Server | 捆绑的槽位 Slot of Bundle | VarInt | |
| | | | 捆绑中的槽位 Slot in Bundle | VarInt | |

#### 更改难度 Change Difficulty

必须启用作弊才能工作。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x03`<br/>资源 resource: `change_difficulty` | 游戏 Play | 服务器 Server | 新难度 New difficulty | Byte | 0: 和平 peaceful, 1: 简单 easy, 2: 普通 normal, 3: 困难 hard. |

#### 确认消息 Acknowledge Message

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x04`<br/>资源 resource: `chat_ack` | 游戏 Play | 服务器 Server | 消息计数 Message Count | VarInt | |

#### 聊天命令 Chat Command

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x05`<br/>资源 resource: `chat_command` | 游戏 Play | 服务器 Server | 命令 Command | String (256) | 不包含前导斜杠 The command typed by the client (without the leading slash). |
| | | | 时间戳 Timestamp | Long | 命令执行的时间戳 The timestamp that the command was executed. |
| | | | Salt | Long | 用于生成以下签名的salt The salt for the following signatures. |
| | | | 参数签名数组长度 Array length | VarInt | 参数签名数组中的元素数量 Number of entries in the following array. 最大为8 Maximum of 8. |
| | | | 参数签名 Array of Argument Signatures | 参数名 Argument name | String (16) | 被签名的参数名称 The name of the argument that is signed by the following signature. |
| | | | | 签名 Signature | Byte Array (256) | 参数对应的签名。长度为256或0字节 The signature that verifies the argument. Always 256 bytes and is not length-prefixed. |
| | | | 消息计数 Message Count | VarInt | |
| | | | 确认位字段 Acknowledged | Fixed BitSet (20) | |

#### 已签名聊天命令 Signed Chat Command

与聊天命令 Chat Command 相同，但用于防止服务器拦截客户端命令。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x06`<br/>资源 resource: `signed_chat_command` | 游戏 Play | 服务器 Server | 命令 Command | String (256) | |
| | | | 时间戳 Timestamp | Long | |
| | | | Salt | Long | |
| | | | 参数签名数组长度 Array length | VarInt | 最大为8 Maximum of 8. |
| | | | 参数签名 Array of Argument Signatures | 参数名 Argument name | String (16) | |
| | | | | 签名 Signature | Byte Array (256) | |
| | | | 消息计数 Message Count | VarInt | |
| | | | 确认位字段 Acknowledged | Fixed BitSet (20) | |

#### 聊天消息 Chat Message

用于向服务器发送聊天消息。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x07`<br/>资源 resource: `chat` | 游戏 Play | 服务器 Server | 消息 Message | String (256) | |
| | | | 时间戳 Timestamp | Long | |
| | | | Salt | Long | |
| | | | 是否有签名 Has Signature | Boolean | |
| | | | 签名 Signature | Optional Byte Array (256) | 仅当有签名为真时存在 Only present if Has Signature is true. |
| | | | 消息计数 Message Count | VarInt | |
| | | | 确认位字段 Acknowledged | Fixed BitSet (20) | |

#### 玩家会话 Player Session

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x08`<br/>资源 resource: `chat_session_update` | 游戏 Play | 服务器 Server | 会话ID Session ID | UUID | |
| | | | 公钥过期时间 Public Key Expires at | Long | 公钥到期的Unix时间（毫秒） The time the play session key expires in Unix epoch milliseconds. |
| | | | 公钥长度 Public key length | VarInt | 公钥字节长度。最大为512字节 Length of the proceeding public key. Maximum length in Notchian server is 512 bytes. |
| | | | 公钥 Public Key | Byte Array | 编码的公钥 The public key. |
| | | | 密钥签名长度 Key Signature length | VarInt | 密钥签名字节长度。最大为4096字节 Length of the proceeding key signature. Maximum length in Notchian server is 4096 bytes. |
| | | | 密钥签名 Key Signature | Byte Array | 密钥签名字节 The key signature bytes. |

#### 区块批次已接收 Chunk Batch Received

通知服务器客户端何时接收到整个区块批次。服务器使用信息来调整发送的未完成批次的数量。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x09`<br/>资源 resource: `chunk_batch_received` | 游戏 Play | 服务器 Server | 每个批次区块数量 Chunks per tick | Float | 所需的每刻接收的区块数量 Desired chunks per tick. |

#### 客户端状态 Client Status

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x0A`<br/>资源 resource: `client_command` | 游戏 Play | 服务器 Server | 动作ID Action ID | VarInt Enum | 见下文 See below. |

动作ID Action ID 值：

| 动作ID Action ID | 动作 Action |
|---|---|
| 0 | 执行重生 Perform respawn |
| 1 | 请求统计 Request stats |

#### 客户端信息（游戏）Client Information (play)

与配置阶段的客户端信息数据包相同。

#### 命令建议请求 Command Suggestions Request

当按下Tab键时发送，以请求可用的自动完成选项。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x0C`<br/>资源 resource: `command_suggestion` | 游戏 Play | 服务器 Server | 事务ID Transaction Id | VarInt | 唯一标识符 Unique identifier. |
| | | | 文本 Text | String (32500) | 命令或命令的一部分。不包含前导斜杠 All text behind the cursor without the / (e.g. if the user typed /foo bar<cursor> then text would be foo bar). |

#### 确认配置 Acknowledge Configuration

从游戏状态发送以确认配置状态。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x0D`<br/>资源 resource: `configuration_acknowledged` | 游戏 Play | 服务器 Server | 无字段 no fields | |

此数据包将客户端的状态切换到配置。

#### 点击容器按钮 Click Container Button

当在容器中按下可见按钮/附魔/信标交易选项时使用。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x0E`<br/>资源 resource: `container_button_click` | 游戏 Play | 服务器 Server | 窗口ID Window ID | Byte | 打开的窗口的ID The ID of the window. |
| | | | 按钮ID Button ID | Byte | 按钮的唯一ID Unique ID depending on the type of container. |

#### 点击容器 Click Container

当玩家在容器窗口中点击槽位时使用。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x0F`<br/>资源 resource: `container_click` | 游戏 Play | 服务器 Server | 窗口ID Window ID | Unsigned Byte | 窗口的ID，由服务器在打开容器 Open Screen 中提供 The ID of the window which was clicked. 0 for player inventory. |
| | | | 状态ID State ID | VarInt | 发送动作的最后一个同步容器状态 Set Container Slot State ID packet的状态ID The last received State ID from either a Set Container Slot or a Set Container Content packet. |
| | | | 槽位 Slot | Short | 被点击的槽位 The clicked slot number, see below. |
| | | | 按钮 Button | Byte | 用于按下的按钮 The button used in the click, see below. |
| | | | 模式 Mode | VarInt Enum | 点击模式，见下文 Inventory operation mode, see below. |
| | | | 槽位长度 Length of the array | VarInt | 最大为128 Maximum value for Notchian server is 128 slots. |
| | | | 数组槽位 Array of Slot | 槽位编号 Slot number | Short | |
| | | | | 槽位数据 Slot data | Slot | 更改后该槽位的新数据 New data for this slot, in the client's opinion; see below. |
| | | | 携带物品 Carried item | Slot | 鼠标携带的物品（或当前物品），如果玩家没有携带任何物品则为空 Item carried by the cursor. Has to be empty (item ID = -1) for drop mode, otherwise nothing will happen. |

#### 关闭容器（服务器绑定）Close Container (serverbound)

当玩家关闭窗口时从客户端发送此数据包。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x10`<br/>资源 resource: `container_close` | 游戏 Play | 服务器 Server | 窗口ID Window ID | Unsigned Byte | 这是服务器在打开窗口时发送的窗口ID，如果是玩家物品栏则为0 This is the ID of the window that was closed. 0 for player inventory. |

#### 更改容器槽位状态 Change Container Slot State

此数据包在创造模式物品栏屏幕中点击物品槽位时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x11`<br/>资源 resource: `container_slot_state_changed` | 游戏 Play | 服务器 Server | 槽位ID Slot ID | VarInt | |
| | | | 窗口ID Window ID | VarInt | |
| | | | 状态 State | Boolean | |

#### Cookie响应（游戏）Cookie Response (play)

响应从服务器接收的Cookie请求（游戏）数据包。Cookie存储在客户端，并且仅在从同一服务器IP地址加入服务器时进行验证。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x12`<br/>资源 resource: `cookie_response` | 游戏 Play | 服务器 Server | 密钥 Key | Identifier | Cookie的标识符 The identifier of the cookie. |
| | | | 是否有载荷 Has Payload | Boolean | Cookie是否有载荷 The payload is only present if the cookie exists on the client. |
| | | | 载荷长度 Payload Length | Optional VarInt | Cookie载荷的长度 Length of the following byte array. |
| | | | 载荷 Payload | Optional Byte Array (5120) | Cookie的数据。可能为空 The data of the cookie, if it exists. |

#### 服务器绑定插件消息（游戏）Serverbound Plugin Message (play)

主要命名空间文章：插件通道 Plugin channels

将插件消息发送到服务器。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x13`<br/>资源 resource: `custom_payload` | 游戏 Play | 服务器 Server | 通道 Channel | Identifier | 使用的通道名称 Name of the plugin channel used to send the data. |
| | | | 数据 Data | Byte Array (32767) | 通道的任意数据。如果通道不被识别，则忽略 Any data. The length of this array must be inferred from the packet length. |

#### 调试样本订阅 Debug Sample Subscription

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x14`<br/>资源 resource: `debug_sample_subscription` | 游戏 Play | 服务器 Server | 样本类型 Sample Type | VarInt Enum | 0: 刻时间 Tick time |

#### 编辑书 Edit Book

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x15`<br/>资源 resource: `edit_book` | 游戏 Play | 服务器 Server | 槽位 Slot | VarInt | 修改的物品栏槽位。0表示背包中的物品 The hotbar slot where the written book is located. |
| | | | 计数 Count | VarInt | 以下数组中的元素数量 Number of elements in the following array. Maximum array size is 200. |
| | | | 条目 Entries | Array (200) of String (8192) | 书的每一页文本 Text from each page. Maximum string length is 8192 chars. |
| | | | 是否有标题 Has title | Boolean | 如果为真，后面跟标题 If true, the next field is present. |
| | | | 标题 Title | Optional String (128) | 书的标题 Title of book. |

#### 查询实体标签 Query Entity Tag

当按下F3+I时使用。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x16`<br/>资源 resource: `entity_tag_query` | 游戏 Play | 服务器 Server | 事务ID Transaction ID | VarInt | |
| | | | 实体ID Entity ID | VarInt | |

#### 交互 Interact

当玩家与实体交互时发送此数据包。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x17`<br/>资源 resource: `interact` | 游戏 Play | 服务器 Server | 实体ID Entity ID | VarInt | 要交互的实体的ID The ID of the entity to interact with. |
| | | | 类型 Type | VarInt Enum | 0: 交互 interact, 1: 攻击 attack, 2: 交互目标位置 interact at. |
| | | | 目标X Target X | Optional Float | 仅当类型为交互目标位置 interact at. |
| | | | 目标Y Target Y | Optional Float | 仅当类型为交互目标位置 interact at. |
| | | | 目标Z Target Z | Optional Float | 仅当类型为交互目标位置 interact at. |
| | | | 手 Hand | Optional VarInt Enum | 仅当类型为交互或交互目标位置。0: 主手 main hand, 1: 副手 off hand. |
| | | | 是否潜行 Sneaking | Boolean | 玩家是否正在潜行 If the client is sneaking. |

#### 拼图生成 Jigsaw Generate

从拼图方块UI发送以生成拼图结构。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x18`<br/>资源 resource: `jigsaw_generate` | 游戏 Play | 服务器 Server | 位置 Location | Position | 拼图方块位置 Block entity location. |
| | | | 等级 Levels | VarInt | 要生成的等级数 Value of the levels slider/max depth to generate. |
| | | | 保留接头 Keep Jigsaws | Boolean | |

#### 服务器绑定保持连接（游戏）Serverbound Keep Alive (play)

服务器会经常发送保持连接数据包，每个数据包都包含一个随机ID。客户端必须使用相同的载荷响应。如果客户端在数据包发送后的15秒内没有响应保持连接数据包，服务器将踢出客户端。反之，如果服务器在20秒内没有发送任何保持连接数据包，客户端将断开连接并抛出超时异常。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x19`<br/>资源 resource: `keep_alive` | 游戏 Play | 服务器 Server | 保持连接ID Keep Alive ID | Long | |

#### 锁定难度 Lock Difficulty

必须启用作弊才能工作。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x1A`<br/>资源 resource: `lock_difficulty` | 游戏 Play | 服务器 Server | 已锁定 Locked | Boolean | |

#### 设置玩家位置 Set Player Position

更新玩家在服务器上的X、Y和Z位置的更新。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x1B`<br/>资源 resource: `move_player_pos` | 游戏 Play | 服务器 Server | X | Double | 玩家脚的绝对位置 Absolute position. |
| | | | 脚Y Feet Y | Double | 玩家脚的绝对位置 Absolute feet position, normally Head Y - 1.62. |
| | | | Z | Double | 玩家脚的绝对位置 Absolute position. |
| | | | 在地面上 On Ground | Boolean | 如果玩家接触地面（包括水或熔岩上方的空气块）则为真 True if the client is on the ground, false otherwise. |

#### 设置玩家位置和旋转 Set Player Position and Rotation

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x1C`<br/>资源 resource: `move_player_pos_rot` | 游戏 Play | 服务器 Server | X | Double | 玩家脚的绝对位置 Absolute position. |
| | | | 脚Y Feet Y | Double | 玩家脚的绝对位置 Absolute feet position, normally Head Y - 1.62. |
| | | | Z | Double | 玩家脚的绝对位置 Absolute position. |
| | | | 偏航 Yaw | Float | 旋转的绝对旋转角度（度），以度数表示 Absolute rotation on the X Axis, in degrees. |
| | | | 俯仰 Pitch | Float | 旋转的绝对旋转角度（度），以度数表示 Absolute rotation on the Y Axis, in degrees. |
| | | | 在地面上 On Ground | Boolean | 如果玩家接触地面则为真 True if the client is on the ground, false otherwise. |

#### 设置玩家旋转 Set Player Rotation

更新玩家看的方向。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x1D`<br/>资源 resource: `move_player_rot` | 游戏 Play | 服务器 Server | 偏航 Yaw | Float | 旋转的绝对旋转角度（度） Absolute rotation on the X Axis, in degrees. |
| | | | 俯仰 Pitch | Float | 旋转的绝对旋转角度（度） Absolute rotation on the Y Axis, in degrees. |
| | | | 在地面上 On Ground | Boolean | 如果玩家接触地面则为真 True if the client is on the ground, false otherwise. |

#### 设置玩家在地面 Set Player On Ground

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x1E`<br/>资源 resource: `move_player_status_only` | 游戏 Play | 服务器 Server | 在地面上 On Ground | Boolean | 如果玩家接触地面则为真 True if the client is on the ground, false otherwise. |

#### 移动载具（服务器绑定）Move Vehicle (serverbound)

当玩家移动载具时（如船）发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x1F`<br/>资源 resource: `move_vehicle` | 游戏 Play | 服务器 Server | X | Double | 绝对位置（X坐标） Absolute position (X coordinate). |
| | | | Y | Double | 绝对位置（Y坐标） Absolute position (Y coordinate). |
| | | | Z | Double | 绝对位置（Z坐标） Absolute position (Z coordinate). |
| | | | 偏航 Yaw | Float | 旋转的绝对旋转角度（度） Absolute rotation on the vertical axis, in degrees. |
| | | | 俯仰 Pitch | Float | 旋转的绝对旋转角度（度） Absolute rotation on the horizontal axis, in degrees. |

#### 划船 Paddle Boat

由客户端使用以通知服务器玩家开始或停止使用桨。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x20`<br/>资源 resource: `paddle_boat` | 游戏 Play | 服务器 Server | 左桨转动 Left paddle turning | Boolean | |
| | | | 右桨转动 Right paddle turning | Boolean | |

#### 拾取物品 Pick Item

用于在创造模式中交换物品并当Ctrl+Middle Click方块时拾取创造物品。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x21`<br/>资源 resource: `pick_item` | 游戏 Play | 服务器 Server | 槽位使用 Slot to use | VarInt | 参见物品栏 See Inventory. |

#### Ping请求（游戏）Ping Request (play)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x22`<br/>资源 resource: `ping_request` | 游戏 Play | 服务器 Server | 载荷 Payload | Long | 由响应返回 May be any number. Notchian clients use a system-dependent time value which is counted in milliseconds. |

#### 放置配方 Place Recipe

当玩家在合成台或熔炉UI中点击配方时发送此数据包。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x23`<br/>资源 resource: `place_recipe` | 游戏 Play | 服务器 Server | 窗口ID Window ID | Byte | |
| | | | 配方 Recipe | Identifier | 一个配方ID A recipe ID. |
| | | | 全部制作 Make all | Boolean | 影响Shift点击的数量 Affects the amount of items processed; true if shift is down when clicked. |

#### 玩家能力（服务器绑定）Player Abilities (serverbound)

先前称为飞行 Flying。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x24`<br/>资源 resource: `player_abilities` | 游戏 Play | 服务器 Server | 标志 Flags | Byte | 位字段。0x02：正在飞行 is flying. |

#### 玩家动作 Player Action

当玩家挖掘、破坏或射击方块时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x25`<br/>资源 resource: `player_action` | 游戏 Play | 服务器 Server | 状态 Status | VarInt Enum | 玩家执行的动作 See below. |
| | | | 位置 Location | Position | 方块位置 Block position. |
| | | | 面 Face | Byte Enum | 被击中的方块的面 The face being hit (see below). |
| | | | 序列 Sequence | VarInt | |

状态 Status 可以是以下之一：

| 值 Value | 含义 Meaning | 备注 Notes |
|---|---|---|
| 0 | 开始挖掘 Started digging | 仅在创造模式下破坏方块 Sent when the player starts digging a block. If the block was instamined or the player is in creative mode, the client will not send Stop Digging; it will only send this packet. |
| 1 | 取消挖掘 Cancelled digging | 仅当方块未被破坏时发送 Sent when the player lets go of the Mine Block key (default: left click). Face is always set to -Y. |
| 2 | 完成挖掘 Finished digging | 当方块破坏动画完成时发送 Sent when the client thinks it is finished. |
| 3 | 丢弃物品栈 Drop item stack | 触发当玩家按下丢弃键（Q）时丢弃当前手中的整个物品栈 Triggered by using the Drop Item key (default: Q) with the modifier to drop the entire selected stack (default: Control or Command, depending on OS). Location is always set to 0/0/0, Face is always set to -Y. Sequence is always set to 0. |
| 4 | 丢弃物品 Drop item | 触发当玩家按下丢弃键（Q）时丢弃当前手中的单个物品 Triggered by using the Drop Item key (default: Q). Location is always set to 0/0/0, Face is always set to -Y. Sequence is always set to 0. |
| 5 | 释放使用物品 Shoot arrow / finish eating | 表示物品已被使用；BOW和CROSSBOW游戏模式在此发送。Location is always set to 0/0/0, Face is always set to -Y. Sequence is always set to 0. |
| 6 | 交换手中物品 Swap item in hand | 用于交换或放置手持物品 Used to swap or assign an item to the second hand. Location is always set to 0/0/0, Face is always set to -Y. Sequence is always set to 0. |

面 Face 是以下之一：

| 值 Value | 朝向 Offset | 面 Face |
|---|---|---|
| 0 | -Y | 底部 Bottom |
| 1 | +Y | 顶部 Top |
| 2 | -Z | 北 North |
| 3 | +Z | 南 South |
| 4 | -X | 西 West |
| 5 | +X | 东 East |

#### 玩家命令 Player Command

当玩家开始/停止飞行、潜行、疾跑或从睡眠中醒来时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x26`<br/>资源 resource: `player_command` | 游戏 Play | 服务器 Server | 实体ID Entity ID | VarInt | 玩家ID Player ID. |
| | | | 动作ID Action ID | VarInt Enum | 动作ID The type of action, see below. |
| | | | 跳跃提升 Jump Boost | VarInt | 仅用于开始跳跃马 Only used by the "start jump with horse" action, in which case it ranges from 0 to 100. In all other cases it is 0. |

动作ID Action ID 可以是以下之一：

| ID | 动作 Action |
|---|---|
| 0 | 开始潜行 Start sneaking |
| 1 | 停止潜行 Stop sneaking |
| 2 | 离开床 Leave bed |
| 3 | 开始疾跑 Start sprinting |
| 4 | 停止疾跑 Stop sprinting |
| 5 | 开始马跳跃 Start jump with horse |
| 6 | 停止马跳跃 Stop jump with horse |
| 7 | 打开载具物品栏 Open vehicle inventory |
| 8 | 开始潜行飞行 Start flying with elytra |

#### 玩家输入 Player Input

当玩家按下移动或跳跃键时发送此数据包。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x27`<br/>资源 resource: `player_input` | 游戏 Play | 服务器 Server | 横向移动 Sideways | Float | 正值表示向左移动 Positive to the left of the player. |
| | | | 前进移动 Forward | Float | 正值表示向前移动 Positive forward. |
| | | | 标志 Flags | Unsigned Byte | 位字段：0x1: 跳跃 jump, 0x2: 取消挂载 unmount. |

#### Pong（游戏）Pong (play)

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x28`<br/>资源 resource: `pong` | 游戏 Play | 服务器 Server | ID | Int | Ping请求中的ID The id of the ping packet. |

#### 更改配方书设置 Change Recipe Book Settings

替换先前分开的配方书数据、工艺配方显示、熔炉配方显示和爆破熔炉/烟熏器配方显示数据包，并使用不同的ID。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x29`<br/>资源 resource: `recipe_book_change_settings` | 游戏 Play | 服务器 Server | 书ID Book ID | VarInt Enum | 0: 合成台 crafting, 1: 熔炉 furnace, 2: 爆破熔炉 blast furnace, 3: 烟熏器 smoker. |
| | | | 书打开 Book Open | Boolean | |
| | | | 过滤可合成 Filter Craftable | Boolean | |

#### 设置已查看配方 Set Seen Recipe

当玩家在配方书中选择配方以合成时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x2A`<br/>资源 resource: `recipe_book_seen_recipe` | 游戏 Play | 服务器 Server | 配方ID Recipe ID | Identifier | |

#### 选择交易 Select Trade

当玩家在商人交易UI中选择特定交易时发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x2B`<br/>资源 resource: `select_trade` | 游戏 Play | 服务器 Server | 选择的槽位 Selected slot | VarInt | 选择的交易 The selected trade. |

#### 设置信标效果 Set Beacon Effect

更改信标的效果。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x2C`<br/>资源 resource: `set_beacon` | 游戏 Play | 服务器 Server | 是否有主效果 Has Primary Effect | Boolean | |
| | | | 主效果 Primary Effect | Optional VarInt | 主效果的ID A Potion ID. |
| | | | 是否有次效果 Has Secondary Effect | Boolean | |
| | | | 次效果 Secondary Effect | Optional VarInt | 次效果的ID A Potion ID. |

#### 设置手持物品（服务器绑定）Set Held Item (serverbound)

当玩家改变活动物品栏槽位时从客户端发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x2D`<br/>资源 resource: `set_carried_item` | 游戏 Play | 服务器 Server | 槽位 Slot | Short | 玩家已选择的槽位（0-8） The slot which the player has selected (0–8). |

#### 编程命令方块 Program Command Block

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x2E`<br/>资源 resource: `set_command_block` | 游戏 Play | 服务器 Server | 位置 Location | Position | |
| | | | 命令 Command | String (32767) | |
| | | | 模式 Mode | VarInt Enum | 0: 序列 SEQUENCE, 1: 自动 AUTO, 2: 红石 REDSTONE. |
| | | | 标志 Flags | Byte | 0x01: 追踪输出 Track Output (if false, the output of the previous command will not be stored within the command block); 0x02: 条件 Is conditional; 0x04: 总是活动 Automatic. |

#### 编程命令方块矿车 Program Command Block Minecart

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x2F`<br/>资源 resource: `set_command_minecart` | 游戏 Play | 服务器 Server | 实体ID Entity ID | VarInt | |
| | | | 命令 Command | String (32767) | |
| | | | 追踪输出 Track Output | Boolean | 如果为假，上一个命令的输出将不会存储在命令方块中 If false, the output of the previous command will not be stored within the command block. |

#### 设置创造模式槽位 Set Creative Mode Slot

当玩家在创造模式物品栏中点击物品槽位时，客户端会发送此数据包到服务器以设置物品栏中该槽位的内容。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x30`<br/>资源 resource: `set_creative_mode_slot` | 游戏 Play | 服务器 Server | 槽位 Slot | Short | 物品栏槽位 Inventory slot. |
| | | | 点击的物品 Clicked Item | Slot | |

#### 编程拼图方块 Program Jigsaw Block

从拼图方块UI发送以更新方块。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x31`<br/>资源 resource: `set_jigsaw_block` | 游戏 Play | 服务器 Server | 位置 Location | Position | 方块实体位置 Block entity location. |
| | | | 名称 Name | Identifier | |
| | | | 目标 Target | Identifier | |
| | | | 池 Pool | Identifier | |
| | | | 最终状态 Final state | String (32767) | 变为... Turns into. |
| | | | 接头类型 Joint type | String (32767) | 翻滚、对齐 rollable if the attached piece can be rotated, else aligned. |
| | | | 选择优先级 Selection priority | VarInt | |
| | | | 位置优先级 Placement priority | VarInt | |

#### 编程结构方块 Program Structure Block

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x32`<br/>资源 resource: `set_structure_block` | 游戏 Play | 服务器 Server | 位置 Location | Position | 方块实体位置 Block entity location. |
| | | | 动作 Action | VarInt Enum | 更新动作 An additional action to perform beyond simply saving the given data; see below. |
| | | | 模式 Mode | VarInt Enum | 0: 保存 SAVE, 1: 加载 LOAD, 2: 角落 CORNER, 3: 数据 DATA. |
| | | | 名称 Name | String (32767) | |
| | | | 偏移X Offset X | Byte | 介于-48和48之间 Between -48 and 48. |
| | | | 偏移Y Offset Y | Byte | 介于-48和48之间 Between -48 and 48. |
| | | | 偏移Z Offset Z | Byte | 介于-48和48之间 Between -48 and 48. |
| | | | 尺寸X Size X | Byte | 介于0和48之间 Between 0 and 48. |
| | | | 尺寸Y Size Y | Byte | 介于0和48之间 Between 0 and 48. |
| | | | 尺寸Z Size Z | Byte | 介于0和48之间 Between 0 and 48. |
| | | | 镜像 Mirror | VarInt Enum | 0: 无 NONE, 1: 左右 LEFT_RIGHT, 2: 前后 FRONT_BACK. |
| | | | 旋转 Rotation | VarInt Enum | 0: 无 NONE, 1: 顺时针90 CLOCKWISE_90, 2: 顺时针180 CLOCKWISE_180, 3: 逆时针90 COUNTERCLOCKWISE_90. |
| | | | 元数据 Metadata | String (128) | |
| | | | 完整性 Integrity | Float | 介于0和1之间 Between 0 and 1. |
| | | | 种子 Seed | VarLong | |
| | | | 标志 Flags | Byte | 0x01: 忽略实体 Ignore entities; 0x02: 显示空气 Show air; 0x04: 显示边界框 Show bounding box; 0x08: 需要红石 Requires power. |

可能的动作 action 值：

| 值 Value | 动作 Action |
|---|---|
| 0 | 更新数据 Update data |
| 1 | 保存结构 Save structure |
| 2 | 加载结构 Load structure |
| 3 | 检测尺寸 Detect size |

#### 更新告示牌 Update Sign

当玩家点击完成按钮完成告示牌编辑时从客户端发送此数据包，或在打开书并静默签名后（即不由打开签名编辑器 Open Sign Editor 打开）。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x33`<br/>资源 resource: `sign_update` | 游戏 Play | 服务器 Server | 位置 Location | Position | 方块坐标 Block Coordinates. |
| | | | 是前面 Is Front Text | Boolean | 文本是否在告示牌的前面 Whether the updated text is in front or on the back of the sign. |
| | | | 行1 Line 1 | String (384) | 告示牌的第一行 First line of text in the sign. |
| | | | 行2 Line 2 | String (384) | 告示牌的第二行 Second line of text in the sign. |
| | | | 行3 Line 3 | String (384) | 告示牌的第三行 Third line of text in the sign. |
| | | | 行4 Line 4 | String (384) | 告示牌的第四行 Fourth line of text in the sign. |

#### 挥动手臂 Swing Arm

当玩家的手臂摆动时从客户端发送。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x34`<br/>资源 resource: `swing` | 游戏 Play | 服务器 Server | 手 Hand | VarInt Enum | 0: 主手 main hand, 1: 副手 off hand. |

#### 传送到实体 Teleport To Entity

传送到给定实体。观察者模式专用。

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x35`<br/>资源 resource: `teleport_to_entity` | 游戏 Play | 服务器 Server | 目标玩家 Target Player | UUID | 要传送到的玩家的UUID UUID of the player to teleport to (can also be an entity UUID). |

#### 在...上使用物品 Use Item On

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x36`<br/>资源 resource: `use_item_on` | 游戏 Play | 服务器 Server | 手 Hand | VarInt Enum | 用于放置方块的手；0: 主手 main hand, 1: 副手 off hand. |
| | | | 位置 Location | Position | 方块位置 Block position. |
| | | | 面 Face | VarInt Enum | 光标所在的方块的面 The face on which the block is placed (as documented at Player Action). |
| | | | 光标位置X Cursor Position X | Float | 光标在方块面上点击的位置，范围为0到1 The position of the crosshair on the block, from 0 to 1 increasing from west to east. |
| | | | 光标位置Y Cursor Position Y | Float | 光标在方块面上点击的位置，范围为0到1 The position of the crosshair on the block, from 0 to 1 increasing from bottom to top. |
| | | | 光标位置Z Cursor Position Z | Float | 光标在方块面上点击的位置，范围为0到1 The position of the crosshair on the block, from 0 to 1 increasing from north to south. |
| | | | 在方块内部 Inside block | Boolean | 如果玩家头部在方块内部则为真 True when the player's head is inside of a block. |
| | | | 序列 Sequence | VarInt | |

#### 使用物品 Use Item

当玩家在未指向方块或实体的情况下右键点击时发送。（例如，射击弓箭）

| 数据包ID Packet ID | 状态 State | 绑定到 Bound To | 字段名称 Field Name | 字段类型 Field Type | 说明 Notes |
|---|---|---|---|---|---|
| 协议 protocol: `0x38`<br/>资源 resource: `use_item` | 游戏 Play | 服务器 Server | 手 Hand | VarInt Enum | 0: 主手 main hand, 1: 副手 off hand. |
| | | | 序列 Sequence | VarInt | |

---

## 游戏 Play 章节翻译完成

所有游戏 Play 状态的客户端绑定和服务器绑定数据包已全部翻译完成！

