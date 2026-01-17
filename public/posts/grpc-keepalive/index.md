# Grpc Keepalive


## TCP KeepAlive

[TCP KeepAlive机制理解与实践小结](https://www.cnblogs.com/hueyxu/p/15759819.html)中详细介绍了TCP KeepAlive的机制，这里重点提一下涉及到的参数:

- `tcp_keepalive_time` 在TCP保活打开的情况下，最后一次数据交换到TCP发送第一个保活探测包的间隔，即允许的空闲时长，默认为2h
- `tcp_keepalive_probes` 最大允许发送保活探测包的次数，达到此次数后直接放弃尝试，并关闭连接，默认为9次
- `tcp_keepalive_intvl` 发送保活探测包的间隔，默认为75s

所以如果开启了TCP KeepAlive并保持默认参数，则空闲连接会在大约 2h 11min之后被断开

## TCP_USER_TIMEOUT

`TCP_USER_TIMEOUT`表示等待传输数据未被`ack`或者由于没有发送窗口导致未被传输的数据的最长时间，超过该时间，TCP连接会被强制关闭，并且返回`ETIMEDOUT`给应用。

如果将`TCP_USER_TIMEOUT`和 TCP KeepAlive同时使用，根据cloudflare的文章[When TCP sockets refuse to die](https://blog.cloudflare.com/when-tcp-sockets-refuse-to-die/)，`tcp_keepalive_probes`会被忽略，每次尝试发送keepAlive报文时，都会检查是否超过了`TCP_USER_TIMEOUT`，如果超过则直接关闭连接。

类似的，如果发生超时重传，`TCP_USER_TIMEOUT`决定了最终的超时关闭时间，`tcp_retries2`会被忽略，对于零发送窗口的连接，发送端会定期发送窗口探测报文，在发送之前之后检查超过了`TCP_USER_TIMEOUT`。

## TCP超时重传

tcp超时重试包含很多机制，这里只讨论最基础的情况，某个TCP报文发送后一直没有收到响应导致重传，重传会采用指数退避的方式，也就是每次重传的间隔都是上次的2倍，重传间隔的下限为200ms，上限为120s，主要有以下参数控制重传过程：

- ` `/proc/sys/net/ipv4/tcp_retries1``，默认为3，在3次重传后，会尝试通知ip层更改路由
- `/proc/sys/net/ipv4/tcp_retries2`，默认为15，用来计算重传超时时间，超过重传超时时间，大约对应13~30min，连接会被关闭
- ` tcp_syn_retries`，默认为6，初始SYN报文被重传的最大次数，由于RTO默认为1s，对应127s左右的重传耗时
- `tcp_syncack_retries`，默认为5，初始SYN/ACK报文被重传的最大次数

这里多解释一下`tcp_retries`的意义，根据 [Linux TCP_RTO_MIN, TCP_RTO_MAX and the tcp_retries2 sysctl](https://pracucci.com/linux-tcp-rto-min-max-and-tcp-retries2.html)，初始的重试时间间隔就是RTO (Retransmission TimeOut)，对于正在建立的连接( [TCP Retransmission May Be Misleading (2023)](http://arthurchiao.art/blog/tcp-retransmission-may-be-misleading/) )，RTO的初始值为1s，对于已经建立完成的连接，RTO会通过RTT的历史信息计算得到，对于内网环境，可以认为初始的RTO为200ms，最终重传超时总共需要15min左右，如果网络环境很差，最高可能需要30min左右。显然对于客户端而言，这个超时时间太长了，如果指定`TCP_USER_TIMEOUT`则可以在更短的时间内检测到网络不可达，从而快速关闭连接并重连。

todo:
https://codearcana.com/posts/2015/08/28/tcp-keepalive-is-a-lie.html

https://groups.google.com/g/grpc-io/c/6VZYCFZpyTI

https://github.com/grpc/grpc-java/issues/11517

https://github.com/apache/brpc/issues/1154

https://lukexng.medium.com/grpc-keepalive-maxconnectionage-maxconnectionagegrace-6352909c57b8

https://pandaychen.github.io/2020/09/01/GRPC-CLIENT-CONN-LASTING/




