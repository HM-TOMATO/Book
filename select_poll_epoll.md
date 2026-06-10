# select / poll / epoll 原理、实践与面试考点

## 一、为什么需要 I/O 多路复用

Linux 网络编程中，一个进程需要同时监视多个文件描述符（socket、pipe、文件等）的可读/可写状态。若对每个 fd 单独使用阻塞 `read()`/`write()`，则必须一个 fd 配一个线程/进程，资源开销巨大。I/O 多路复用（I/O multiplexing）允许单个线程同时监视大量 fd，在任一 fd 就绪时得到通知，从而高效处理并发连接。

`select`、`poll`、`epoll` 是 Linux 提供的三种 I/O 多路复用机制，它们的发展史也是 Linux 高并发网络编程的演进史。

---

## 二、select

### 2.1 基本原理

`select` 是最古老的 I/O 多路复用接口，诞生于 1983 年的 BSD。

```c
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
```

核心机制：

- 用户进程将关心的 fd 集合以 **fd_set**（位图，bitmap）形式传入内核
- 内核遍历所有 fd，检查是否有就绪事件
- 若有就绪 fd，内核修改 fd_set，标记就绪位，返回给用户态
- 用户态再次遍历 fd_set，找出所有就绪 fd 进行处理

### 2.2 fd_set 的限制

`fd_set` 通常定义为长整型数组，大小固定：

```c
#define FD_SETSIZE 1024

typedef struct {
    unsigned long fds_bits[FD_SETSIZE / (8 * sizeof(unsigned long))];
} fd_set;
```

这意味着：

- 单个进程最多监视 **1024 个 fd**（可重新编译内核修改，但无法动态调整）
- 每次调用 `select` 都需要将 fd_set 从用户态拷贝到内核态
- 返回时内核再次修改 fd_set，用户态需要重新设置关心的 fd

### 2.3 时间复杂度

- 内核遍历：O(n)，n 为 `nfds` 参数值（最大 fd 编号 + 1）
- 用户态遍历：O(n)，需要扫描整个 fd_set 找出就绪 fd
- 两次数据拷贝：用户态 -> 内核态 -> 用户态

### 2.4 代码示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/select.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT 8080
#define MAX_CLIENTS 1024

int main() {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(PORT);
    bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(listen_fd, 128);

    fd_set read_fds, all_fds;
    FD_ZERO(&all_fds);
    FD_SET(listen_fd, &all_fds);
    int max_fd = listen_fd;

    while (1) {
        read_fds = all_fds;
        int ret = select(max_fd + 1, &read_fds, NULL, NULL, NULL);
        if (ret < 0) {
            perror("select");
            break;
        }

        for (int fd = 0; fd <= max_fd; fd++) {
            if (!FD_ISSET(fd, &read_fds)) continue;

            if (fd == listen_fd) {
                int client_fd = accept(listen_fd, NULL, NULL);
                FD_SET(client_fd, &all_fds);
                if (client_fd > max_fd) max_fd = client_fd;
            } else {
                char buf[1024];
                int n = read(fd, buf, sizeof(buf));
                if (n <= 0) {
                    close(fd);
                    FD_CLR(fd, &all_fds);
                } else {
                    write(fd, buf, n);  // echo
                }
            }
        }
    }
    return 0;
}
```

### 2.5 select 的缺陷总结

| 缺陷 | 说明 |
|------|------|
| fd 数量限制 | 默认最多 1024，受 FD_SETSIZE 限制 |
| 两次数据拷贝 | 每次调用都要将 fd_set 从用户态拷贝到内核态 |
| 内核遍历 O(n) | 无论多少 fd 就绪，都要遍历全部 fd |
| 用户态遍历 O(n) | 返回后需要扫描整个位图找出就绪 fd |
| 状态丢失 | 返回后 fd_set 被修改，下次调用前必须重新设置 |

---

## 三、poll

### 3.1 基本原理

`poll` 于 1997 年引入，解决了 `select` 的 fd 数量限制问题。

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd {
    int   fd;       /* 文件描述符 */
    short events;   /* 请求监视的事件 */
    short revents;  /* 返回的就绪事件 */
};
```

核心改进：

- 使用 **pollfd 数组** 替代位图，不再受 1024 限制
- 每个 fd 独立记录请求事件 `events` 和返回事件 `revents`
- `revents` 与 `events` 分离，不会破坏用户传入的数据

### 3.2 事件类型

| 事件宏 | 含义 |
|--------|------|
| POLLIN | 数据可读 |
| POLLOUT | 数据可写 |
| POLLERR | 错误条件 |
| POLLHUP | 挂起 |
| POLLNVAL | 无效 fd |

### 3.3 时间复杂度

- 内核遍历：O(n)，n 为传入的 nfds
- 用户态遍历：O(n)，需要扫描整个数组找出 `revents != 0` 的 fd
- 数据拷贝：每次调用都要将 pollfd 数组拷贝到内核

### 3.4 代码示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <poll.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT 8080
#define MAX_CLIENTS 10000

int main() {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(PORT);
    bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(listen_fd, 128);

    struct pollfd fds[MAX_CLIENTS];
    int nfds = 1;
    fds[0].fd = listen_fd;
    fds[0].events = POLLIN;

    while (1) {
        int ret = poll(fds, nfds, -1);
        if (ret < 0) {
            perror("poll");
            break;
        }

        for (int i = 0; i < nfds; i++) {
            if (fds[i].revents == 0) continue;

            if (fds[i].fd == listen_fd) {
                int client_fd = accept(listen_fd, NULL, NULL);
                fds[nfds].fd = client_fd;
                fds[nfds].events = POLLIN;
                nfds++;
            } else {
                char buf[1024];
                int n = read(fds[i].fd, buf, sizeof(buf));
                if (n <= 0) {
                    close(fds[i].fd);
                    fds[i] = fds[nfds - 1];
                    nfds--;
                    i--;
                } else {
                    write(fds[i].fd, buf, n);
                }
            }
        }
    }
    return 0;
}
```

### 3.5 poll 的缺陷

`poll` 虽然突破了 1024 限制，但本质问题未解决：

- 每次调用仍需将 pollfd 数组从用户态拷贝到内核态
- 内核仍需遍历全部 fd，时间复杂度 O(n)
- 用户态返回后仍需遍历全部 fd，时间复杂度 O(n)
- fd 数量很大时，性能线性下降

---

## 四、epoll

### 4.1 基本原理

`epoll` 于 2002 年在 Linux 2.5.44 引入，是 Linux 下最高效的 I/O 多路复用机制。

```c
int epoll_create(int size);        /* 创建 epoll 实例 */
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  /* 增删改 fd */
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);  /* 等待事件 */
```

核心设计思想：

- **事件驱动 + 回调机制**：通过内核回调将就绪 fd 放入就绪链表，避免遍历
- **红黑树管理 fd**：所有监视的 fd 存储在红黑树中，增删查 O(log n)
- **就绪链表**：就绪事件通过回调加入链表，`epoll_wait` 直接返回就绪事件
- **mmap 共享内存**：内核就绪事件通过共享内存映射到用户态，避免数据拷贝

### 4.2 内核数据结构

```c
struct eventpoll {
    struct rb_root rbr;           /* 红黑树根节点，存储所有 epoll_event */
    struct list_head rdllist;     /* 就绪链表，存储就绪的 fd */
    wait_queue_head_t wq;         /* 等待队列，epoll_wait 阻塞在此 */
};

struct epitem {
    struct rb_node rbn;           /* 红黑树节点 */
    struct list_head rdllink;     /* 就绪链表节点 */
    struct epoll_filefd ffd;      /* 被监视的 fd */
    struct eventpoll *ep;         /* 所属 eventpoll */
};
```

### 4.3 工作模式

`epoll` 提供两种触发模式：

#### LT（Level Trigger，水平触发，默认）

- fd 就绪后，如果用户没有处理完数据，下次 `epoll_wait` 仍会返回该 fd
- 与 `select`/`poll` 行为一致，编程简单，不易丢事件
- 支持阻塞 I/O 和非阻塞 I/O

#### ET（Edge Trigger，边缘触发）

- fd 状态变化时（不可读 -> 可读）只通知一次
- 用户必须一次性将所有数据读完，否则不再通知
- **必须使用非阻塞 I/O**，配合循环读到 `EAGAIN`
- 减少 epoll 事件触发次数，性能更高，但编程复杂

### 4.4 代码示例（LT 模式）

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT 8080
#define MAX_EVENTS 1024

int main() {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(PORT);
    bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(listen_fd, 128);

    int epoll_fd = epoll_create1(0);
    struct epoll_event ev, events[MAX_EVENTS];

    ev.events = EPOLLIN;
    ev.data.fd = listen_fd;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev);

    while (1) {
        int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == listen_fd) {
                int client_fd = accept(listen_fd, NULL, NULL);
                ev.events = EPOLLIN;
                ev.data.fd = client_fd;
                epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client_fd, &ev);
            } else {
                char buf[1024];
                int n = read(events[i].data.fd, buf, sizeof(buf));
                if (n <= 0) {
                    close(events[i].data.fd);
                    epoll_ctl(epoll_fd, EPOLL_CTL_DEL, events[i].data.fd, NULL);
                } else {
                    write(events[i].data.fd, buf, n);
                }
            }
        }
    }
    return 0;
}
```

### 4.5 代码示例（ET 模式 + 非阻塞 I/O）

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT 8080
#define MAX_EVENTS 1024
#define BUF_SIZE 1024

int set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    set_nonblocking(listen_fd);
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(PORT);
    bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(listen_fd, 128);

    int epoll_fd = epoll_create1(0);
    struct epoll_event ev, events[MAX_EVENTS];

    ev.events = EPOLLIN | EPOLLET;  // ET 模式
    ev.data.fd = listen_fd;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev);

    while (1) {
        int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == listen_fd) {
                while (1) {  // ET 模式需要循环 accept
                    int client_fd = accept(listen_fd, NULL, NULL);
                    if (client_fd < 0) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK)
                            break;
                        perror("accept");
                        break;
                    }
                    set_nonblocking(client_fd);
                    ev.events = EPOLLIN | EPOLLET;
                    ev.data.fd = client_fd;
                    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client_fd, &ev);
                }
            } else {
                while (1) {  // ET 模式需要循环读到 EAGAIN
                    char buf[BUF_SIZE];
                    int n = read(events[i].data.fd, buf, sizeof(buf));
                    if (n > 0) {
                        write(events[i].data.fd, buf, n);
                    } else if (n == 0) {
                        close(events[i].data.fd);
                        epoll_ctl(epoll_fd, EPOLL_CTL_DEL, events[i].data.fd, NULL);
                        break;
                    } else { // n < 0
                        if (errno == EAGAIN || errno == EWOULDBLOCK) {
                            break;
                        }
                        close(events[i].data.fd);
                        epoll_ctl(epoll_fd, EPOLL_CTL_DEL, events[i].data.fd, NULL);
                        break;
                    }
                }
            }
        }
    }
    return 0;
}
```

### 4.6 epoll 的优势

| 特性 | select/poll | epoll |
|------|-------------|-------|
| fd 数量限制 | 有（1024）/ 无但线性增长 | 无限制，仅受系统内存限制 |
| 数据拷贝 | 每次调用拷贝整个 fd 集合 | 首次 epoll_ctl 拷贝，后续无拷贝 |
| 内核遍历 | O(n) 遍历所有 fd | O(1) 直接返回就绪事件 |
| 用户态遍历 | O(n) 扫描全部 fd | O(m)，m 为就绪 fd 数量 |
| 时间复杂度 | O(n) | O(1)（活跃连接少时） |
| 触发模式 | 仅 LT | LT + ET |

---

## 五、三者对比总结

| 维度 | select | poll | epoll |
|------|--------|------|-------|
| 诞生时间 | 1983 (BSD) | 1997 (POSIX) | 2002 (Linux 2.5.44) |
| 数据结构 | fd_set（位图） | pollfd 数组 | 红黑树 + 就绪链表 |
| 最大 fd 数 | 1024（默认） | 无硬限制 | 无限制 |
| 内核检测方式 | 轮询遍历 O(n) | 轮询遍历 O(n) | 回调通知 O(1) |
| 用户态遍历 | O(n) | O(n) | O(m)，m 为就绪数 |
| 数据拷贝 | 每次调用全量拷贝 | 每次调用全量拷贝 | 首次拷贝，后续共享内存 |
| 触发模式 | LT | LT | LT / ET |
| 跨平台 | POSIX | POSIX | Linux only |
| 适用场景 | 少量连接、跨平台 | 中等连接、跨平台 | 大量连接、Linux 高并发 |

---

## 六、面试高频考点

### 6.1 select 的 fd_set 为什么是值-结果参数？

`select` 返回时会修改传入的 fd_set，将未就绪的 fd 对应的位清零。这导致：

- 每次调用前必须重新设置 fd_set
- 无法保留原始关心的 fd 集合
- 这是早期 API 设计的缺陷，`poll` 通过分离 `events` 和 `revents` 解决了这个问题

### 6.2 为什么 epoll 比 select/poll 高效？

三个核心原因：

1. **避免全量遍历**：epoll 通过内核回调将就绪 fd 加入就绪链表，`epoll_wait` 直接返回就绪事件，无需遍历全部 fd
2. **避免重复数据拷贝**：fd 通过 `epoll_ctl` 一次性注册到内核红黑树，后续 `epoll_wait` 无需再次拷贝；就绪事件通过 mmap 共享内存返回
3. **时间复杂度优化**：epoll 的时间复杂度为 O(就绪事件数)，而 select/poll 为 O(监视 fd 总数)

### 6.3 epoll LT 和 ET 的区别？

| 维度 | LT（水平触发） | ET（边缘触发） |
|------|----------------|----------------|
| 触发时机 | fd 只要可读就一直通知 | fd 状态变化时只通知一次 |
| 编程难度 | 简单 | 复杂，必须非阻塞 + 循环读到 EAGAIN |
| 事件重复 | 可能重复触发 | 不重复触发 |
| 数据读取 | 可以只读一部分 | 必须一次性读完 |
| 性能 | 一般 | 更高，减少 epoll 调用次数 |
| 适用场景 | 通用场景 | 高并发、大流量场景 |

### 6.4 epoll 的惊群问题（Thundering Herd）

**问题描述**：多个进程/线程阻塞在同一个 `epoll_wait` 上，当事件到来时，所有进程都被唤醒，但只有一个能处理该事件，其余进程空转。

**解决方案**：

- `EPOLLEXCLUSIVE` 标志（Linux 4.5+）：确保只有一个进程被唤醒
- 使用 `SO_REUSEPORT` 让多个 listen socket 分散到不同进程
- 单线程 + 线程池模型（如 Nginx、Redis）

### 6.5 epoll 的线程安全性

- `epoll_create` 创建的 epoll 实例是文件描述符，可以被多线程共享
- `epoll_ctl` 是线程安全的，但多线程同时操作同一个 fd 需要应用层加锁
- `epoll_wait` 可以多线程同时调用，但可能触发惊群问题

### 6.6 什么场景下 select/poll 反而更合适？

- 需要跨平台兼容（epoll 仅 Linux）
- 监视的 fd 数量很少（< 100），且连接频繁变化
- 代码简洁性优先于极致性能

### 6.7 epoll 的文件描述符上限

epoll 本身没有 fd 数量上限，但受限于：

- 系统最大打开文件数：`ulimit -n`
- 进程内存限制：每个 fd 占用一定内核内存
- `/proc/sys/fs/file-max` 系统级限制

### 6.8 epoll 内核源码关键路径

```
epoll_create1()
  -> sys_epoll_create1()
    -> ep_alloc()              // 分配 eventpoll 结构体

epoll_ctl()
  -> sys_epoll_ctl()
    -> ep_insert() / ep_remove() / ep_modify()
      -> ep_rbtree_insert()    // 红黑树操作 O(log n)
      -> ep_item_poll()        // 注册回调函数 ep_poll_callback

epoll_wait()
  -> sys_epoll_wait()
    -> ep_poll()
      -> 检查 rdllist 是否为空
      -> 非空则拷贝就绪事件到用户态
      -> 空则加入等待队列，睡眠等待回调唤醒

ep_poll_callback()            // fd 就绪时由内核调用
  -> 将 epitem 加入 rdllist
  -> 唤醒等待队列中的进程
```

### 6.9 为什么 epoll 使用红黑树而不是哈希表？

- 红黑树是有序结构，支持范围查询和顺序遍历
- fd 编号虽然通常是整数，但可能非常稀疏，哈希表空间利用率低
- 红黑树的 O(log n) 增删查在 fd 数量很大时仍然稳定
- 内核中红黑树实现成熟，如 `include/linux/rbtree.h`

### 6.10 一个 fd 可以加入多个 epoll 实例吗？

可以。一个 fd 可以被多个 epoll 实例监视，但每个 epoll 实例对该 fd 是独立的。需要注意：

- 每个 epoll 实例都会收到该 fd 的就绪通知
- 关闭 fd 会自动从所有 epoll 实例中移除
- 多 epoll 实例监听同一 fd 的场景较少见，通常用于特殊架构设计

---

## 七、实践建议

### 7.1 技术选型

| 连接数 | 推荐方案 |
|--------|----------|
| < 100 | select / poll，代码简单 |
| 100 ~ 10000 | poll 或 epoll LT |
| > 10000 | epoll ET + 非阻塞 I/O |
| 跨平台需求 | select / poll（或使用 libevent/libuv 封装） |

### 7.2 高并发服务器设计要点

1. **使用 ET 模式**：减少 epoll 事件触发次数，提升性能
2. **非阻塞 I/O**：ET 模式必须配合非阻塞，LT 模式也建议使用非阻塞
3. **线程模型**：
   - 单线程事件循环：Redis、Nginx（worker 进程）
   - 多线程 + epoll：一个线程 accept，多个线程 epoll_wait（注意惊群）
   - 线程池：主线程 I/O，工作线程处理业务逻辑
4. **连接管理**：使用定时器处理空闲连接超时（如心跳机制）
5. **零拷贝**：大文件传输使用 `sendfile()` 减少数据拷贝

### 7.3 常见陷阱

- **ET 模式下未读完数据**：导致数据滞留，后续不再收到通知
- **LT 模式下未及时处理**：事件重复触发，浪费 CPU
- **忘记设置非阻塞**：ET 模式下阻塞 read 可能导致永久阻塞
- **fd 泄漏**：连接关闭后未从 epoll 中删除，也未 close fd
- **大 fd_set 栈溢出**：select 的 fd_set 放在栈上，大量 fd 可能导致栈溢出

---

## 八、相关系统调用速查

| 系统调用 | 作用 |
|----------|------|
| `select()` | 监视多个 fd 的就绪状态（位图方式） |
| `poll()` | 监视多个 fd 的就绪状态（数组方式） |
| `epoll_create()` / `epoll_create1()` | 创建 epoll 实例 |
| `epoll_ctl()` | 增删改 epoll 监视的 fd |
| `epoll_wait()` / `epoll_pwait()` | 等待事件就绪 |
| `fcntl(fd, F_SETFL, O_NONBLOCK)` | 设置非阻塞模式 |
| `sendfile()` | 零拷贝文件发送 |
| `splice()` / `tee()` | 管道零拷贝 |
