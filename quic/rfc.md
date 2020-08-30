
....前面不重要...
#  简介

## 术语和定义

* QUIC: 这个协议的名字

* QUIC packet: 一个完整的可处理的 quic 单元, 可以被压缩到一个 udp 报文中, 多个 QUIC packet 可以被压缩进一个 udp 报文

* Ack-eliciting Packet(需要 ack 的包): 一个包含了非 ack, padding, connection_close 的数据包, 会让接受者给一个响应.

* Out-of-order packet(失序包): 一个没有增加最大接受包号的包.

* EndPoint (终端): 一个可以通过 生成, 处理, 接受包参与 quic 链接的东西. 包括客户端服务端.

* Client, Sverer 略

* Address: 一个 (ip_version, ip, udp protocol, port) 元组

* Connection id: 一个用于在一个终端 标记 链接的序号

* Stream: 一个单向 or 双向发送数据的频道, 一个连接可以有多个频道

* Applaication 略


## 标记

\[X]: X是可选的

X(A): X 是 A bit long

X(A/B/C): X长度在 ABC 之中

X(i): X使用长度可变编码2.

X(*): X长度可变
...



# 流

流对应用提供了一个轻量的, 字节序的抽象. 流可以单向或者双向, 比如单向流可以看做一个无限长的消息抽象.


流可以通过发送数据创建, 其他相关过程如流的管理- 如结束, 取消, 管理流控制, 都会竟可能的最小化抽象. 例如, 单个流的帧, 可以打开, 携带数据和关闭流. 流也可以长期生存完整个链接的生命周期.

流也可以被任意一个终端创建, 会和其他的流同时工作, 并可以取消. QUIC 不保证不同流的顺序.

QUIC 允许有任意多的流, 每个流也有任意多的数量的数据, 这个会在 第四节 流程控制 和 流的限制中讨论

## 流的类型和标识

流可以是单向的或者是双向的.

流在一个连接内被数字的编号, 作为流 id. 一个流 id 是 62bit 的整数.流 id 会被以 长度可变编码 方式编码. 每个 id 在一个连接范围内不会被重用.

最后一位是用来表示流的类型.

```

+------+----------------------------------+
| Bits | Stream Type                      |
+======+==================================+
| 0x0  | 客户端发起, 双向  |
+------+----------------------------------+
| 0x1  | 服务端发起, 双向  |
+------+----------------------------------+
| 0x2  | 客户端发起, 单向 |
+------+----------------------------------+
| 0x3  | 服务端发起,单向 |
+------+----------------------------------+

            Table 1: Stream ID Types

```
在一个类型内, 流id会递增的被创建.一个用过的大的 流 ID, 比他小的流 id一定是用过的.

第一个客户端发起的打开的流的 id 为 0

## 发送接收数据

应用数据会被封装在流的帧里.终端会用流 id 和 偏移量 去定位这个消息的顺序.

终端必须可以以有序字节流的方式发送流数据. 按序发送需要终端在公告流程控制范围内缓存失序数据.

QUIC 没有限制发送方字节的顺序, 具体的实现可以提供这个功能.

一个终端可以接受一个流多个相同偏移的数据. 已经接受的会被丢弃, 如果数据不一致会报 PROTOCOL_VIOLATION.

流是一个对 QUIC 没有其他可见结构的有序字节流的抽象, 流的每一帧的边界, 在丢包或者重传的时候, 可以不被保留.(换句话说可以合并)

## 流的优先级

流的多路进行会对性能产生比较大的影响, 要按照正确的优先级来分配资源.

QUIC 没有提供交换优先级信息的机制. 他取决续 QUIC 应用程序的接受优先级.

一个 QUIC 实现 **应该** 提供一个让应用标识流的相对优先级的机制. 当决定流使用那些资源的时候, 实现应该用应用提供的信息.

## 流的必要操作

有一些应用和流交互的时候的必要操作. 本文不提供具体 api, 但是 所有的 quic 实现必须暴露实现以下描述的操作:

发送端:

* write data(写数据), 了解合适进行流量控制保证保存的数据都发送出去
* end the stream (clean termination 安全关闭):由含有 fin标志的流帧触发
*  reset the stream (abrupt termination重置流): 当流没有关闭态, 由RESET_STREAM 帧触发,

接收端:

* 读数据
* 放弃读数据并且请求关闭 一般用 STOP_SENDING 帧触发

应用要知道这些操作的情况, 包括 对端打开或者重置流, 对端放弃读这个流, 新的数据可用, 数据因为流控制可写或者不可写等情况.

# 流的状态

本节用流的发送和接受组件来描述. 有两个状态机:

1. 终端传输数据
2. 终端接收数据

单向流会直接使用合适的状态机. 双向留两个都会用. 大部分时候, 无论单向双向状态机都一样. 双向流的打开条件会复杂一些, 因为两个方向都有打开.

> note: 这些状态信息量很大. 此文档用这些状态来描述 这些不同的帧 如何发送和交互. 尽管这些对实现 QUIC很有用,  但是不会限制具体的实现. 具体的实现可以自己定义状态机..

## 发送时流状态

```

          o
          | 创建流 (发送)
          | 对端创建双向流
          v
      +-------+
      | 就绪   | 发送 RESET_STREAM
      |       |-----------------------.
      +-------+                       |
          |                           |
          | 发送 STREAM /              |
          |      STREAM_DATA_BLOCKED  |
          |                           |
          | 对端创建双向流               |
          |                           |
          v                           |
      +-------+                       |
      | 发送   | 发送 RESET_STREAM     |
      |       |---------------------->|
      +-------+                       |
          |                           |
          | 发送 STREAM + FIN         |
          v                           v
      +-------+                   +-------+
      |  数据  | 发送 RESET_STREAM | 重置   |
      |  发送  |----------------->| 发送   |
      +-------+                   +-------+
          |                           |
          | 接收 All ACKs             | Recv ACK
          v                           v
      +-------+                   +-------+
      | 数据  |                    | 重置   |
      | 接收  |                    | 接收   |
      +-------+                   +-------+

```

应用打开终端初始化流的发送部分(0和 2 类型是客户端, 1 和 3 类型的服务端). "就绪" 状态代表了一个新创建的可以接受从应用接受数据的流. 流的数据会被缓存然后等待发送.

发送第一个 STREAM 或者 STREAM_DATA_BLOCKED 帧会导致 发送部分进入 "发送" 态. 一个实现可以选择在进入这个状态的时候申请流 id, 这样能更好的按优先级处理(没太理解).

双向流的发送部分是由对端初始化(0 是服务端, 1 是客户端). 当接收端创建完成后会进入"就绪".

在"发送"态. 终端会发送和重发(如果有需求)流的数据. 终端尊重对端设置的的流控制配置. 会接收和好处理 MAX_STREAM_DATA 帧. 一个终端在发送态被流控制立宪制的时候会发送 STREAM_DATA_BLOCKED帧.

在应用表示 所有的流数据都发送了, 并且一个带有 fin 的 STREAM 也发送了. 流会进入"数据发送"状态. 在这个状态, 终端只会在需要的时候进行重传.
终端不会检查流控制的限制和发送STREAM_DATA_BLOCKED帧. 当对端收完了所有的数据后,终端才会接受MAX_STREAM_DATA帧. 期间终端会无视所有的MAX_STREAM_DATA.

一旦所有的流数据都被确认了. 流的发送方会进入"数据接受"状态, 这是终止状态.

当终端在 "就绪", "发送"或者"数据发送"状态中, 一个应用可以通知他放弃传输. 或者,终端也可能收到对端的STOP_SENDING.  这些情况下, 终端会发送一个RESET_STREAM 并进入 "重置发送" 态.

一些终端会发送一个 RESET_STREAM 作为流的第一帧.  这会让流的发送部分打开并进入 "重置发送" 状态.

一旦发送的RESET_STREAM帧被确认. 流的发送部分会进入"重置接收"状态, 这是一个最终态.

## 接收流状态

下图展示了流从对端接收数据的状态. 流的接收的状态 只是一个对对端发送态的一个镜像. 流的接受部分不会跟踪发送流不可观察的部分. 类似 "就绪" 态.取而代之的, 流的接受部分会 最终一些不会被发送端感知的 把数据给应用的过程.


```
          o
          | Recv STREAM / STREAM_DATA_BLOCKED / RESET_STREAM
          | Create Bidirectional Stream (Sending)
          | Recv MAX_STREAM_DATA / STOP_SENDING (Bidirectional)
          | Create Higher-Numbered Stream
          v
      +-------+
      | 接受   | Recv RESET_STREAM
      |       |-----------------------.
      +-------+                       |
          |                           |
          | Recv STREAM + FIN         |
          v                           |
      +-------+                       |
      | 了解   | Recv RESET_STREAM     |
      | 大小   |---------------------->|
      +-------+                       |
          |                           |
          | Recv All Data             |
          v                           v
      +-------+ Recv RESET_STREAM +-------+
      | 数据   |--- (optional) --->| 重置   |
      | 接受   |  Recv All Data    | 发送 |
      +-------+<-- (optional) ----+-------+
          |                           |
          | App Read All Data         | App Read RST
          v                           v
      +-------+                   +-------+
      | 数据   |                  | 重置  |
      | 读    |                   | 读    |
      +-------+                   +-------+

              Figure 3: States for Receiving Parts of Streams
```

当收到第一个STREAM,STREAM_DATA_BLOCKED,RESET_STREAM  接受部分的流会被对端初始化(1,3 客户端初始化, 0,2 服务端初始化). 对于被对端初始化的双向流来说, 收到了 MAX_STREAM_DATA 和 STOP_SENDING 也会创建接受部分. 初始化状态是 "接受".
