# Network



## HTTP

### HTTP2

#### Frame

##### Format

所有帧(`frame`)都以固定的9个8位字节开头作为固定帧

- Frame Header 
  - Length(24 bit): 表示Frame Payload的长度(并不包括Header)，最大不能超过2^24，除非设置`SETTINGS_MAX_FRAME_SIZE`
  - Type(8 bit): 帧类型，决定了该帧Payload的组成格式和语义
  - Flags(8 bit):
  - R(1 bit): 保留位
  - Stream Identifier(31 bit)
- Frame Payload: 

##### Frame Size

默认`Frame Payload`的最大大小为2^14，除非请求对方主动设置`SETTINGS_MAX_FRAME_SIZE` ，该变量区间范围2^14 ~ 2^24-1，`Frame Header`的大小并不计入至Frame Size的计算当中

##### 头部压缩与解压缩(Header Compression/Decompression)

通过`HPACK`(RFC )

- **静态表**：HTTP2协议预定义在应用层代码中
- **动态表**：每个HTTP2连接单独维护一个动态表

##### Frame Type

- `HEADER`
- `DATA`
- `PUSH_PROMISE`
- `WIN_UPDATE`
- `END_STREAM`
- `RST_STREAM`

#### Stream

##### State

状态枚举

- `idle`
- `reserved`
- `open`
- `half closed`
- `closed`

[状态机](https://datatracker.ietf.org/doc/html/rfc7540#section-5.1)

流(`stream`)建立过程

1. client execute dial operation
2. server accept
3. client/server create bidirectional tcp connection
4. client send multiple requests(req1, req2, req3)
5. at HTTP2 layer, the code will create a unique stream for every request, and assign a identifier for every stream, call `stream identifier` , like 
   - req1 -> stream1 
   - req2 -> stream2
   - req3 -> steam3

**优先级**

客户端可以在发送HEADER帧时为stream设置优先级(`priotity`)

**流量控制**

通过在应用层层面，做到以每个流(`stream`)的粒度来让接收端控制流量窗口大小，每个流都有独立的窗口，发送端发送的数据量不允许超过接收端的窗口容量。

接收端通过发送`WINDOW_UPDATE帧`告诉发送端其容许的窗口大小**增量**，在建立连接时，双方通过`SETTINGS_INITIAL_WINDOW_SIZE帧`定义了初始流量控制窗口的默认大小