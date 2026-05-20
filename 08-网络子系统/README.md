# 08 — 网络子系统

> Linux 网络子系统实现了完整的 TCP/IP 协议栈，是现代互联网的基石。
> 本章从 **Socket 接口 → 协议栈层次 → 数据包接收/发送路径 → 关键数据结构**
> 逐层解析，并对照 Linux 2.6.0 源码（0.11 网络功能不完整，跳过）。

---

## 1. 网络子系统层次架构

```
┌─────────────────────────────────────────────────────────────┐
│                    用户空间                                   │
│    socket() bind() listen() accept() send() recv()          │
└──────────────────────┬──────────────────────────────────────┘
                       │ 系统调用
┌──────────────────────▼──────────────────────────────────────┐
│                  Socket 层（BSD Socket API）                  │
│   sock_create → inet_create → tcp_v4_connect ...            │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                 传输层（Transport Layer）                     │
│   TCP（tcp.c）          UDP（udp.c）                         │
│   可靠/有序/流式        不可靠/无序/报文                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                 网络层（Network Layer）                       │
│   IP（ip_input.c, ip_output.c）                             │
│   路由（route.c）    ARP（arp.c）    ICMP（icmp.c）          │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                 链路层（Link Layer）                          │
│   以太网帧（eth.c）   网络设备接口（dev.c）                   │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                     网卡驱动                                  │
│   e100 / e1000 / virtio_net ...                             │
└──────────────────────┬──────────────────────────────────────┘
                       │ DMA
                    物理网线
```

---

## 2. 核心数据结构

### 2.1 sk_buff（套接字缓冲区）

`sk_buff` 是 Linux 网络子系统中最重要的数据结构，
代表**网络中的一个数据包**（从用户数据到最终的以太网帧）：

```
sk_buff 内存布局：

  head ──► ┌────────────────────────────────────────┐
           │      headroom（为协议头预留空间）         │
  data ──► ├────────────────────────────────────────┤
           │  以太网头（14字节）  ← 由驱动填充         │
           │  IP 头（20字节）    ← 由 IP 层填充        │
           │  TCP 头（20字节）   ← 由 TCP 层填充       │
           │  用户数据            ← 应用程序提供        │
  tail ──► ├────────────────────────────────────────┤
           │      tailroom（为尾部预留空间）           │
  end  ──► └────────────────────────────────────────┘

关键指针：
  skb->head: 缓冲区起始
  skb->data: 当前数据起始（随协议处理向上移动）
  skb->tail: 数据结束
  skb->end:  缓冲区结束
  skb->len:  data 到 tail 的长度（当前数据长度）
```

```c
/* include/linux/skbuff.h（精简）*/
struct sk_buff {
    struct sk_buff *next, *prev;    /* 链表 */
    struct sock *sk;                /* 关联的 socket */
    struct net_device *dev;         /* 网络设备 */
    
    unsigned char *head, *data, *tail, *end;
    unsigned int len;               /* 数据长度 */
    unsigned int data_len;          /* 分散/聚集 IO 的额外数据 */
    
    __u16 protocol;                 /* 协议类型（ETH_P_IP/ETH_P_ARP...）*/
    __u8  pkt_type;                 /* 包类型（PACKET_HOST/BROADCAST/...）*/
    
    /* 协议头指针（在 skb->data 移动时保存各层头部位置）*/
    union { ... } h;    /* 传输层头（tcp_header/udp_header）*/
    union { ... } nh;   /* 网络层头（iphdr）*/
    union { ... } mac;  /* 链路层头（ethhdr）*/
    
    struct dst_entry *dst;          /* 路由信息 */
};

/* sk_buff 操作 API */
skb_push(skb, len)   /* 在 data 前添加 len 字节（添加协议头）*/
skb_pull(skb, len)   /* 从 data 移除 len 字节（去掉协议头）*/
skb_put(skb, len)    /* 在 tail 后添加 len 字节（添加数据）*/
skb_reserve(skb, len)/* 在 head 处预留 len 字节（为头部预留空间）*/
```

### 2.2 sock / socket 结构

```
用户空间 fd    内核空间
──────────     ─────────────────────────────────────
    fd  ──►  file ──► socket（VFS 文件）
                         │
                         └─► sock（协议相关，如 tcp_sock）

struct socket {             /* BSD socket（VFS 层）*/
    socket_state state;     /* SS_UNCONNECTED/SS_CONNECTED/... */
    struct proto_ops *ops;  /* inet_stream_ops / inet_dgram_ops */
    struct sock *sk;        /* → 协议实现层 */
    struct file *file;      /* → VFS 文件 */
};

struct sock {               /* 协议无关的 sock 基类 */
    __u32 rcv_saddr;        /* 本地 IP */
    __u16 num;              /* 本地端口 */
    __u32 daddr;            /* 对端 IP */
    __u16 dport;            /* 对端端口 */
    struct sk_buff_head sk_receive_queue;  /* 接收队列 */
    struct sk_buff_head sk_write_queue;    /* 发送队列 */
    struct proto *prot;     /* → TCP/UDP 协议操作 */
    ...
};

struct tcp_sock {           /* TCP 专有状态 */
    struct sock sk;         /* ← 必须是第一个字段 */
    __u32 snd_nxt;          /* 下一个要发送的序号 */
    __u32 rcv_nxt;          /* 期望收到的下一个序号 */
    __u32 snd_una;          /* 最后一个未确认的序号 */
    __u16 mss_cache;        /* 最大段大小（MSS）*/
    struct tcp_options_received rx_opt;
    ...
};
```

---

## 3. 数据包接收路径（RX Path）

```
网卡收到数据帧
      │
      │ （DMA 将数据写入 ring buffer）
      ▼
网卡中断（netif_rx() 或 NAPI: netif_receive_skb()）
      │
      ▼
net/core/dev.c: netif_receive_skb(skb)
      │
      ├─ 根据 skb->protocol 分发：
      │   ETH_P_IP  → ip_rcv()
      │   ETH_P_ARP → arp_rcv()
      │   ...
      ▼
net/ipv4/ip_input.c: ip_rcv(skb, ...)
      │
      ├─ IP 校验和检查
      ├─ 路由决策（ip_route_input）：
      │   · 本机目标 → ip_local_deliver()
      │   · 转发     → ip_forward()
      ▼
ip_local_deliver → ip_local_deliver_finish
      │
      ├─ 根据 IP 头的 protocol 字段分发：
      │   IPPROTO_TCP → tcp_v4_rcv()
      │   IPPROTO_UDP → udp_rcv()
      │   IPPROTO_ICMP→ icmp_rcv()
      ▼
net/ipv4/tcp_ipv4.c: tcp_v4_rcv(skb)
      │
      ├─ 查找 socket（根据 src_ip:src_port:dst_ip:dst_port）
      ├─ 调用 TCP 状态机处理（tcp_rcv_state_process）
      ├─ 将数据放入 socket 接收队列（sk_receive_queue）
      └─ 唤醒等待数据的进程（sk->sk_data_ready）
              │
              ▼
用户进程从 recv() 系统调用中醒来，读取数据
```

---

## 4. 数据包发送路径（TX Path）

```
用户程序调用 send(fd, buf, len, 0)
      │
      ▼
sys_send → sys_sendto → sock_sendmsg
      │
      ▼
inet_sendmsg → tcp_sendmsg
      │
      ├─ 将用户数据拷贝到 sk_buff（sk_stream_alloc_skb）
      ├─ 更新 TCP 序号（snd_nxt）
      ├─ 放入发送队列（sk->sk_write_queue）
      └─ 调用 tcp_push_one() 或 tcp_write_xmit()
              │
              ▼
tcp_transmit_skb(sk, skb, ...)
      │
      ├─ 构建 TCP 头（填充序号/确认号/标志）
      └─ 调用 ip_queue_xmit()
              │
              ▼
net/ipv4/ip_output.c: ip_queue_xmit
      │
      ├─ 路由查找（ip_route_output）
      ├─ 构建 IP 头（填充 src_ip/dst_ip/ttl）
      └─ ip_output → ip_finish_output
              │
              ▼
dev_queue_xmit(skb)
      │
      ├─ 流量控制（qdisc）
      └─ dev->hard_start_xmit()  ← 调用网卡驱动发送
              │
              ▼
网卡硬件发送数据帧（DMA → 物理网线）
```

---

## 5. TCP 三次握手（内核视角）

```
客户端                                    服务端（已 listen）
─────                                     ─────────────────
connect()                                 已调用 listen()

tcp_v4_connect()                          (等待中)
  sk->state = TCP_SYN_SENT
  发送 SYN 包 ──────────────────────────►
                                          tcp_v4_rcv()
                                          tcp_rcv_state_process()
                                            SYN: sk->state = TCP_SYN_RECV
                                          ◄────────────── 发送 SYN+ACK

tcp_rcv_state_process()
  SYN+ACK: sk->state = TCP_ESTABLISHED
  发送 ACK ──────────────────────────────►
                                          tcp_rcv_state_process()
                                            ACK: sk->state = TCP_ESTABLISHED
                                            将连接移到 accept 队列
                                            唤醒 accept() 中的进程

连接建立完成！
send() / recv() 可以开始工作
```

---

## 6. Socket 编程与内核对应关系

```c
/* 用户程序                           内核函数 */
socket(AF_INET, SOCK_STREAM, 0)  → sock_create → inet_create → tcp_v4_init_sock
bind(sockfd, &addr, ...)         → inet_bind
listen(sockfd, backlog)          → inet_listen → tcp_listen_start
accept(sockfd, ...)              → inet_accept → inet_csk_accept（阻塞等待）
connect(sockfd, &server_addr, .) → tcp_v4_connect → 发送 SYN

send(sockfd, buf, len, 0)        → tcp_sendmsg
recv(sockfd, buf, len, 0)        → tcp_recvmsg（阻塞等待数据）

close(sockfd)                    → tcp_close → 发送 FIN
```

---

## 7. 实验

```bash
# 跟踪 TCP 连接建立（使用 ss）
ss -tn state established

# 查看路由表
ip route show
cat /proc/net/route

# 查看 socket 统计
cat /proc/net/sockstat
cat /proc/net/tcp    # 所有 TCP 连接

# 用 tcpdump 抓包（同时观察内核行为）
sudo tcpdump -i eth0 -n tcp port 80

# GDB 跟踪 TCP 状态机（在 2.6.0 内核）
(gdb) break tcp_rcv_state_process
(gdb) commands
> printf "TCP state: %d\n", sk->sk_state
> continue
> end
```

---

## 8. 网络性能优化关键点

| 优化技术 | 引入版本 | 说明 |
|---------|---------|------|
| NAPI | 2.4.20+ | 轮询 + 中断混合，减少中断开销 |
| TCP Offload (TSO/GSO) | 2.6.18+ | 大包分片由硬件完成 |
| Zero Copy (sendfile) | 2.2 | 跳过用户空间拷贝 |
| epoll | 2.5.44 | O(1) IO 多路复用 |
| RSS/RPS | 2.6.35+ | 多队列网卡，多核并行接收 |
| XDP/eBPF | 4.8+ | 在驱动层直接处理包，旁路协议栈 |
