---
layout: article
title: pgbouncer学习
tags: pg postgres postgresql pgbouncer
key: 2023-03-13-pgbouncer-learning
---

## 背景
[pgbouncer](https://github.com/pgbouncer/pgbouncer)是一个PostgreSQL的轻量级线程池，支持客户端多种连接方式。  

## 本地编译部署
```shell
# 依赖安装
brew install autoconf automake libtool

# 编译
sh autogen.sh

./configure --enable-debug --prefix=/usr/local --with-openssl=/usr/local/opt/openssl

make clean; make

# 部署到/usr/local/bin
make install
```

## 使用方式
### 连接用户数据库
使用pgbouncer需要配置增加配置文件:
```ini
; pgbouncer.ini
[databases]
db1 = host=localhost port=5432 dbname=postgres

[pgbouncer]
listen_port=6432
listen_addr=localhost
auth_type=md5
auth_file=userlist.txt
logfile=pgbouncer.log
pidfile=pgbouncer.pid
admin_users=pgbouncer
; session,transaction,statement
pool_mode=transaction
```

userlist.txt配置如下:
```
"postgres" "1"
"pgbouncer" "pgbouncer"
```
执行指令为: `pgbouncer pgbouncer.ini [-dvuR]`，具体可查阅[官方文档](https://www.pgbouncer.org/usage.html)。  
在上述配置的场景下，直连postgres和通过pgbouncer代理连接的命令分别为:
```shell
# 直连postgres
psql -U postgres -d postgres -h localhost -p 5432
# 通过pgbouncer连接
psql -U postgres -d db1 -h localhost -p 6432
```

### 连接pgbouncer后台
pgbouncer本身会提供一个假的数据库，可以查看pgbouncer整体的状态，连接方式是按上面的配置文件配置`admin_users`。  
在上述配置的场景下，连接pgbouncer后台假数据库的方式为:  
```shell
~  $ psql -h 127.0.0.1 -p 6432 -U pgbouncer -d pgbouncer
pgbouncer=# show mem;
     name     | size | used | free | memtotal
--------------+------+------+------+----------
 user_cache   | 2312 |    3 |   47 |   115600
 db_cache     |  216 |    2 |   73 |    16200
 pool_cache   |  552 |    1 |   49 |    27600
 server_cache |  608 |    0 |    0 |        0
 client_cache |  608 |    1 |   49 |    30400
 iobuf_cache  | 4112 |    1 |   49 |   205600
```

### 区分连接pgbouncer密码与连接数据库密码
需要区分密码的话，需要在`databases`的配置中指定用户名和密码，这样的话pgbouncer连接pg服务端时，就只会按照`databases`中的配置来。
```ini
; pgbouncer.ini
[databases]
db1 = host=localhost port=5432 dbname=postgres user=postgres password=diffpw
```

### pgbouncer连接池复用等级
pgbouncer连接池的复用等级有`SESSION`、`TRANSACTION`、`STATEMENT`。  
- `SESSION`模式支持postgres所有特性，连接池的连接会被客户端一直占有直到客户端退出
- `TRANSACTION`模式下，当客户端有请求时，从连接池取出连接提供给客户端执行，执行完成之后放回连接池，不被客户端占有。支持事务，但无法支持一些全局变量的设置
- `STATEMENT`模式下，功能与`TRANSACTION`模式类似，但是不支持事务，只支持单语句提交

`SESSION`与`TRANSACTION`在postgres上的[特性支持对比](https://www.pgbouncer.org/features.html)：

| Feature                          | Session pooling | Transaction pooling |
| -------------------------------- | --------------- | ------------------- |
| Startup parameters               | Yes             | Yes                 |
| SET/RESET                        | Yes             | Never               |
| LISTEN/NOTIFY                    | Yes             | Never               |
| WITHOUT HOLD CURSOR              | Yes             | Yes                 |
| WITH HOLD CURSOR                 | Yes             | Never               |
| Protocol-level prepared plans    | Yes             | No                  |
| PREPARE / DEALLOCATE             | Yes             | Never               |
| ON COMMIT DROP temp tables       | Yes             | Yes                 |
| PRESERVE/DELETE ROWS temp tables | Yes             | Never               |
| Cached plan reset                | Yes             | Yes                 |
| LOAD statement                   | Yes             | Never               |
| Session-level advisory locks     | Yes             | Never               |

## 代码逻辑

### 一些重要的结构体
```c
struct List {
	/** Pointer to next node or head. */
	struct List *next;
	/** Pointer to previous node or head. */
	struct List *prev;
};
/**
 * Header structure for StatList.
 */
struct StatList {
	/** Actual list head */
	struct List head;
	/** Count of objects currently in list */
	int cur_count;
};
/*
 * Store for pre-initialized objects of one type.
 */
struct Slab {
	struct List head;
	struct StatList freelist;
	struct StatList fraglist;
	char name[32];
	unsigned final_size;
	unsigned total_count;
	slab_init_fn  init_func;
	CxMem *cx;
};
/*
 * Stream Buffer.
 *
 * Stream is divided to packets.  On each packet start
 * protocol handler is called that decides what to do.
 */
struct SBuf {
	struct event ev;	/* libevent handle */

	uint8_t wait_type;	/* track wait state */
	uint8_t pkt_action;	/* method for handling current pkt */
	uint8_t tls_state;	/* progress of tls */

	int sock;		/* fd for this socket */

	unsigned pkt_remain;	/* total packet length remaining */

	sbuf_cb_t proto_cb;	/* protocol callback */

	SBuf *dst;		/* target SBuf for current packet */

	IOBuf *io;		/* data buffer, lazily allocated */

	const SBufIO *ops;	/* normal vs. TLS */
	struct tls *tls;	/* TLS context */
	const char *tls_host;	/* target hostname */
};

struct ListenSocket {
	struct List node;
	int fd;
	bool active;
	struct event ev;
	PgAddr addr;
};

/*
 * parsed packet header, plus whatever data is
 * available in SBuf for this packet.
 *
 * if (pkt->len == mbuf_avail(&pkt->data))
 *     packet is fully in buffer
 *
 * get_header() points pkt->data.pos after header.
 * to packet body.
 */
struct PktHdr {
	unsigned type;
	unsigned len;
	struct MBuf data;
};

const struct CfLookup pool_mode_map[] = {
	{ "session", POOL_SESSION },
	{ "transaction", POOL_TX },
	{ "statement", POOL_STMT },
	{ NULL }
};
```

### 框架
pgbouncer是通过[libevent](https://github.com/libevent/libevent)实现的reactor框架，向event_loop中添加或删除socket的回调函数，通过回调函数处理数据。  
pgbouncer起了一个代理的效果，它对与用户的请求交互通过client socket，与pg server交互通过server socket。client socket会模拟真实的pg server，返回给用户自己伪造的返回包或者透传server socket的返回包。  
![pgbouncer框架]({{ site.baseurl }}/assets/images/article_imgs/pgbouncer/framework.png)

### 连接池
pgbouncer连接池实例中有5个服务端链表，这里维护的都是与pg server的真实连接，通过链表之间的移动代表一个真实连接在连接池中的状态。
```c
struct StatList active_server_list;
struct StatList idle_server_list;
struct StatList used_server_list; 
struct StatList tested_server_list;
struct StatList new_server_list;
```
根据连接池模式(`session/transaction/statement`)设置的不同，链表之间的移动逻辑也对应有改变。例如：  
1. `session`模式下，当客户端查询完成时，server的连接会保持在`active_server_list`中。
2. 在`transaction`和`statement`模式中，server的连接会回到`idle_server_list`中，等待其他客户端的调用。在这种情况下，实际上用户是不持有pg server的连接的，但是由于pgbouncer fake server的存在，让他误以为自己依旧占有连接，并可以继续进行sql操作。因此这两种模式能高效利用与pg server的连接，提高并发量。


### 重要函数
#### 数据接受与处理回调函数(sbuf_main_loop)
![socket处理buffer数据流程]({{ site.baseurl }}/assets/images/article_imgs/pgbouncer/socket_buffer_main_loop.png)

#### 客户端回调函数(client_proto)
![客户端回调函数]({{ site.baseurl }}/assets/images/article_imgs/pgbouncer/client_callback(event_read).png)

#### 服务端回调函数(server_proto)
![服务端回调函数]({{ site.baseurl }}/assets/images/article_imgs/pgbouncer/server_callback.png)


## 调用流程比较
### 普通pg server连接流程
![pg startup流程]({{ site.baseurl }}/assets/images/article_imgs/pgbouncer/pg_startup_proto.png)

### pgbouncer首次连接及查询流程(transaction mode)
![连接pgbouncer并查询流程]({{ site.baseurl }}/assets/images/article_imgs/pgbouncer/client-bouncer-pg-seq.png)
