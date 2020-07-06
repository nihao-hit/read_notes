# listen_sock与conn_sock端口关系源码分析

mptcpv0.95

1. 端口、fd、socket、进程

内核为新创建的socket结构分配fd，进程通过fd操作socket，而端口是用于标识socket结构体的数，与进程没有直接联系，进程可以有多个端口，即进程可以持有多个socket结构体；一个socket结构体也可以被多个进程持有（父子进程）。

监听socket通过accept()创建的新socket的端口号是否一致？

一致，tcp连接通过四元组进行区分，由同一监听socket创建的多个连接可通过客户端ip地址和端口号的变化来区分。

监听socket通过accept()在三次握手收到ack时才为新连接创建socket，三次握手阶段使用的是request_sock结构体，它含有创建新连接的最小数据，避免资源浪费。

监听socket的backlog队列缓存的是已被tcp接受（即三次握手已经完成），但还没有被应用层所接受的连接。

服务器调用位于tcp_minisocks.c文件中的tcp_create_openreq_child()完成这一过程。通过调用inet_csk_clone_lock()函数，从监听socket与request_sock拷贝新连接所需资源

```c
struct sock *tcp_create_openreq_child(const struct sock *sk,
				      struct request_sock *req,
				      struct sk_buff *skb)
{
	struct sock *newsk = inet_csk_clone_lock(sk, req, GFP_ATOMIC);
	...
}
```

显然，新连接的源端口号与目的端口号都直接req复制过来，且在以后未改变。

```c
struct sock *inet_csk_clone_lock(const struct sock *sk,
				 const struct request_sock *req,
				 const gfp_t priority)
{
	struct sock *newsk;

	newsk = sk_clone_lock(sk, priority);

	if (newsk) {
		struct inet_connection_sock *newicsk = inet_csk(newsk);
		...
        //inet_dport：目的端口号，inet_num：源端口号
		inet_sk(newsk)->inet_dport = inet_rsk(req)->ir_rmt_port;
		inet_sk(newsk)->inet_num = inet_rsk(req)->ir_num;
		...
	}
	return newsk;
}
```

而req_sock是由监听socket在三次握手收到syn包时创建的，通过tcp_input.c文件里的tcp_conn_request()函数

```c
int tcp_conn_request(struct request_sock_ops *rsk_ops,
		     const struct tcp_request_sock_ops *af_ops,
		     struct sock *sk, struct sk_buff *skb)
{
	...
	req = inet_reqsk_alloc(rsk_ops, sk, !want_cookie);
	...
	tcp_openreq_init(req, &tmp_opt, skb, sk);
    ...
}
```

源端口号和目的端口号在tcp_openreq_init()函数中由收到的syn首部赋值。

```c
static void tcp_openreq_init(struct request_sock *req,
			     const struct tcp_options_received *rx_opt,
			     struct sk_buff *skb, const struct sock *sk)
{
	struct inet_request_sock *ireq = inet_rsk(req);
	...
	ireq->ir_rmt_port = tcp_hdr(skb)->source;
	ireq->ir_num = ntohs(tcp_hdr(skb)->dest);
	...
}
```
