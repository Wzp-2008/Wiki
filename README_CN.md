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
