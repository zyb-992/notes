# GRPC

> http2 https://www.cncf.io/blog/2018/07/03/http-2-smarter-at-scale/

## 基于Http2实现

### Compare http1.1 with http2

#### http1.1

##### 缺点

- 队头阻塞
- 一个连接一次只能处理一个请求(同步)
- HTTP头部没有经过压缩，增加了请求字节数，并且不可重用
- 每个请求之间没有优先次序

#### http2

> https://datatracker.ietf.org/doc/html/rfc7540#autoid-12

- Binary message framing layer
  - - 
- Request-response multiplexing
- Efficient header compressions
- Request prioritization 
- Server push

## 消息结构

- 标签
  - 组成
    - 字段索引: proto文件中定义message的字段的索引
    - 线路类型
  - 值
    - value = (field_index << 3) | wire_type



## Channelz

