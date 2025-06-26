# 将 libtcpx 适配到 iperf

本文档描述了将 `libtcpx` 集成到 `iperf` 中以实现零拷贝 TCP 接收与发送的步骤。

## 总体思路

仅仅在测速时用 `libtcpx` 管理我们的流量，对于 iperf 的控制连接不使用 tcpx。

**重要约束**：
- 只有在 TCP 协议模式下才启用 `libtcpx` 功能。
- 控制连接保持使用标准 socket API，只有数据连接使用 `libtcpx`。

---

## 详细步骤

### 第 1 步: 修改构建系统

为了让 `iperf` 能够找到并链接 `libtcpx`，我们需要修改其 `autotools` 构建系统。

#### 1.1 修改 `configure.ac`

在 `configure.ac` 文件中，我们需要添加使用 `pkg-config` 来检查 `libtcpx` 是否存在的逻辑。

```diff
--- a/configure.ac
+++ b/configure.ac
@@ -39,6 +39,9 @@
 AC_PROG_RANLIB
 AC_PROG_INSTALL
 
+# TCPX
+PKG_CHECK_MODULES([TCPX], [libtcpx], [use_tcpx=yes], [use_tcpx=no])
+
 # Checks for header files.
 AC_HEADER_STDC
 AC_CHECK_HEADERS([arpa/inet.h fcntl.h netdb.h netinet/in.h netinet/tcp.h stdlib.h string.h strings.h sys/param.h sys/socket.h sys/time.h syslog.h unistd.h linux/tcp.h linux/tls.h sched.h sys/resource.h sys/sendfile.h])

```
同时，可以添加一个 `--with-tcpx` 编译选项，以便在构建时控制是否启用 `libtcpx` 支持。

#### 1.2 修改 `src/Makefile.am`

在 `src/Makefile.am` 中，将 `libtcpx` 的编译和链接选项加入到 iperf 的构建目标中。

```diff
--- a/src/Makefile.am
+++ b/src/Makefile.am
@@ -1,6 +1,8 @@
 AM_CFLAGS = -D_GNU_SOURCE
 AM_CPPFLAGS = -I$(top_srcdir)/src
 
+AM_CFLAGS += $(TCPX_CFLAGS)
+
 bin_PROGRAMS = iperf3
 
 noinst_LIBRARIES = libiperf.a
@@ -18,7 +20,7 @@
 libiperf_a_SOURCES = \
 	$(iperf3_SOURCES)
 
-iperf3_LDADD = libiperf.a $(iperf3_socket_lib) $(iperf3_thread_lib)
+iperf3_LDADD = libiperf.a $(iperf3_socket_lib) $(iperf3_thread_lib) $(TCPX_LIBS)
 
 # Tests
 # XXX: none yet

```

#### 1.3 验证修改

完成对 `configure.ac` 和 `src/Makefile.am` 的修改后，我们需要重新生成构建系统脚本并验证 `libtcpx` 是否被正确识别。

1.  **重新生成 `configure` 脚本**:
    首先，如果项目中有 `bootstrap.sh` 脚本，运行它来更新 `autotools` 配置。

    ```bash
    ./bootstrap.sh
    ```

2.  **运行 `configure` 脚本**:
    接下来，运行 `configure` 脚本。它会检查系统环境，并根据 `configure.ac` 中的设置生成 `Makefile`。

    ```bash
    # 设置 PKG_CONFIG_PATH 以确保能找到 libtcpx.pc
    PKG_CONFIG_PATH=/path/to/libtcpx:$PKG_CONFIG_PATH ./configure
    ```

    **验证输出**：在 `configure` 的输出中，仔细检查关于 `libtcpx` (即 `TCPX`) 的部分。如果 `pkg-config` 成功找到了库，你应该能看到类似 "checking for TCPX... yes" 的信息。

3.  **编译检查**:
    最后，运行 `make`，确认项目可以成功编译。

    ```bash
    make
    ```
    **验证结果**：如果在链接阶段没有出现关于 `libtcpx` 未定义的引用的错误，说明构建系统已成功配置。

---

### 第 2 步: 修改核心数据结构

根据规划，我们需要在 `iperf` 的核心数据结构中添加 `libtcpx` 的上下文指针。

#### 2.1 修改 `iperf.h`

编辑 `src/iperf.h` 文件，在 `iperf_test` 和 `iperf_stream` 两个结构体中添加相应的 `ctx` 成员。

**重要**：确保在文件顶部包含 `libtcpx` 的头文件。

```c
// iperf.h

#ifdef HAVE_LIBTCPX
#include <libtcpx.h> // <--- 包含 libtcpx 头文件
#endif

// ...

struct iperf_test
{
    // ... 现有成员 ...
    
#ifdef HAVE_LIBTCPX
    // TCPX 相关上下文
    struct rxnic_ctx* rxnic_ctx;         // 用于网卡 RX 管理
    struct txnic_ctx* txnic_ctx;         // 用于网卡 TX 管理  
    struct tcpx_server_ctx* server_ctx;  // 服务器预连接上下文
    
    // TCPX 配置参数
    char* tcpx_rxdev;                    // RX 网络设备名称
    char* tcpx_txdev;                    // TX 网络设备名称
    int tcpx_rxmem_type;                 // RX 内存类型
    int tcpx_txmem_type;                 // TX 内存类型
    size_t tcpx_rxmem_size;              // RX 内存缓冲区大小
    size_t tcpx_txmem_size;              // TX 内存缓冲区大小
    int start_queue;                     // 起始队列号
    int num_queue;                       // 队列数量
    int tcpx_debug_level;                // TCPX 调试级别
    size_t iobuf_size;                   // 连接的iobuf大小
    int max_tokens;                      // 连接的最大tokens
    int tx_pipeline;                     // TX pipeline深度
#endif
};

// ...

struct iperf_stream
{
    // ... 现有成员 ...
    
#ifdef HAVE_LIBTCPX
    struct tcpx_client_ctx* client_ctx;  // 客户端预连接上下文
    struct connection_ctx* conn_ctx;     // 活动连接上下文
#endif
};
```

**验证**：确保修改后代码能够编译通过，即使在没有 `libtcpx` 的环境中也能正常工作。

---

### 第 3 步: 集成 libtcpx 上下文管理

#### 3.1 添加命令行参数

为了在运行时动态配置 `libtcpx`，我们需要在 `iperf_parse_arguments()` (`src/iperf_api.c`) 中添加新的命令行选项。

**新增参数**：
*   `--tcpx-rxdev <ifname>`: 指定 `libtcpx` 绑定的 RX 网络设备。
*   `--tcpx-txdev <ifname>`: 指定 `libtcpx` 绑定的 TX 网络设备。
*   `--tcpx-rxmem <type>`: 指定用于 RX 的内存类型（如 `host` 或 `cuda`），`--tcpx-rxdev` 必须同时指定, 默认host。
*   `--tcpx-txmem <type>`: 指定用于 TX 的内存类型（如 `host` 或 `cuda`），`--tcpx-txdev` 必须同时指定，默认host。
*   `--tcpx-rxmemsz <int>`: 指定用于 RX 的缓冲区大小，`--tcpx-rxdev` 必须同时指定，默认16000 * 4096B。
*   `--tcpx-txmemsz <int>`: 指定用于 TX 的缓冲区大小，`--tcpx-txdev` 必须同时指定，默认16000 * 4096B。
*   `--start-queue <int>`: 指定用于 RX 的起始队列号，`--tcpx-rxdev` 必须同时指定，默认为1。
*   `--num-queue <int>`: 指定用于 RX 的队列数量，`--tcpx-rxdev` 必须同时指定，默认为16。
*   `--iobuf-sz <int>`: 指定用于初始化connection的iobuf_size，`--tcpx-rxdev` 必须同时指定，默认为819200，可以通过KB，MB来指定。
*   `--max-tokens <int>`: 指定用于初始化connection的max_tokens，`--tcpx-rxdev` 必须同时指定，默认为512，范围为(0, 1024]。
*   `--tx-pipeline`: 指定用于初始化connection的pipeline，`--tcpx-txdev` 必须同时指定，默认32，要求大于0。
*   `--tcpx-debug <type>`: 指定用于tcpx-debug的等级，默认为INFO。

**调试级别说明**：
- `NONE` 或 `0`: 无调试输出
- `ERROR` 或 `1`: 仅输出错误信息
- `WARN` 或 `2`: 输出警告和错误信息
- `INFO` 或 `3`: 输出一般信息（默认）
- `VERBOSE` 或 `4`: 输出详细调试信息
- `ALL` 或 `5`: 输出所有调试信息包括跟踪

**重要约束检查**：

#### 3.2 网卡上下文的创建

**位置**：在 `iperf_parse_arguments()` 解析完 TCPX 相关参数后立即执行。

**创建逻辑**：
```c
#ifdef HAVE_LIBTCPX
// 在参数解析完成后创建网卡上下文
if (test->tcpx_rxdev) {
    // 设置 TCPX 调试级别
    tcpx_set_debug_level(test->tcpx_debug_level);
    if (test->debug) {
        iperf_printf(test, "TCPX 调试级别已设置为: %d\n", test->tcpx_debug_level);
    }
    
    // 创建 RX NIC 上下文
    test->rxnic_ctx = tcpx_rxnic_ctx_init(
        test->tcpx_rxdev, 
        test->tcpx_rxmem_type,
        test->tcpx_rxmem_size,
        test->start_queue,
        test->num_queue
    );
    if (!test->rxnic_ctx) {
        iperf_err(test, "无法初始化 RX NIC 上下文 for %s", test->tcpx_rxdev);
        return -1;
    }
    iperf_printf(test, "TCPX 已启用：rx设备=%s, RX队列=%d-%d, 内存大小=%zu\n", 
                 test->tcpx_rxdev, test->start_queue, 
                 test->start_queue + test->num_queue - 1, test->tcpx_rxmem_size);
}

if (test->tcpx_txdev){
    // 如果之前没有设置过调试级别（即没有 rxdev），在这里设置
    if (!test->tcpx_rxdev) {
        tcpx_set_debug_level(test->tcpx_debug_level);
        if (test->debug) {
            iperf_printf(test, "TCPX 调试级别已设置为: %d\n", test->tcpx_debug_level);
        }
    }
    
    // 创建 TX NIC 上下文
    test->txnic_ctx = tcpx_txnic_ctx_init(
        test->tcpx_txdev,
        test->tcpx_txmem_type,
        test->tcpx_txmem_size
    );
    if (!test->txnic_ctx) {
        iperf_err(test, "无法初始化 TX NIC 上下文 for %s", test->tcpx_txdev);
        // 如果 RX 也创建了，需要销毁
        if (test->rxnic_ctx) {
            tcpx_rxnic_ctx_destroy(test->rxnic_ctx);
            test->rxnic_ctx = NULL;
        }
        return -1;
    }
    iperf_printf(test, "TCPX 已启用：tx设备=%s, 内存大小=%zu\n", test->tcpx_txdev, test->tcpx_txmem_size);
}
#endif
```

#### 3.3 运行前重置流控规则

在开始任何测试之前（无论是客户端还是服务器模式），都应该重置网卡上的流控规则，以确保从干净的状态开始，避免之前测试留下的规则干扰当前测试。

**位置**: `iperf_run_server()` 和 `iperf_run_client()` (`src/iperf_api.c`)

**重置逻辑**:
```c
// 在 iperf_run_server() 和 iperf_run_client() 函数的开头

#ifdef HAVE_LIBTCPX
if (test->tcpx_rxdev) {
    // 重置流控规则，确保从干净状态开始
    if (tcpx_reset_flow_steering(test->tcpx_rxdev) != 0) {
        iperf_err(test, "无法重置设备 %s 上的流控规则", test->tcpx_rxdev);
        return -1;
    } else if (test->debug) {
        iperf_printf(test, "设备 %s 上的流控规则已重置，准备开始测试\n", test->tcpx_rxdev);
    }
}
#endif
```

#### 3.4 控制socket流规则设置

在服务器模式下，当监听套接字创建完成后，需要为将控制socket流量引导至不相关的队列。这个操作应在 `iperf_server_listen()` 函数 (`src/iperf_api.c`) 中，在 `netannounce()` 调用之后执行。

**位置**: `iperf_server_listen()` (`src/iperf_api.c`)

**逻辑**:
控制连接使用一个特殊的队列和规则ID配置。队列ID设置为 `start_queue + num_queue`，规则ID设置为0，这样可以避免与后续的客户端连接产生冲突。

```c
// 在 iperf_server_listen() 中，netannounce() 调用之后

#ifdef HAVE_LIBTCPX
if (test->tcpx_rxdev) {
    // 为控制连接创建特殊的队列ID和规则ID
    int server_queue_id = test->start_queue + test->num_queue;
    int server_rule_id = 0;
    
    // 获取本地地址信息
    struct sockaddr_in6 local_addr;
    socklen_t addr_len = sizeof(local_addr);
    if (getsockname(test->listener, (struct sockaddr*)&local_addr, &addr_len) != 0) {
        iperf_err(test, "无法获取服务器监听地址信息");
        return -1;
    }
    
    // 引导控制连接流量到特殊队列
    if (tcpx_open_flow_steering(
        test->tcpx_rxdev,
        &local_addr,
        NULL,
        server_queue_id,
        server_rule_id
    ) != 0) {
        iperf_err(test, "无法设置控制连接的流控规则");
        return -1;
    } else if (test->debug) {
        iperf_printf(test, "控制连接流控规则已设置 (队列 %d, 规则 %d)\n", 
                     server_queue_id, server_rule_id);
    }
}
#endif
```

### 第 3.5 步: 连接上下文的创建

当每个数据流连接建立时，我们需要为其初始化 `libtcpx` 的连接上下文 (`connection_ctx`)。此操作已移至 `iperf_add_stream()` 函数 (`src/iperf_api.c`) 中，以确保在流被赋予唯一 ID 后再进行初始化。

**位置**: `iperf_add_stream()` (`src/iperf_api.c`)

**创建逻辑**:
对于每个新的 TCP 流，我们利用 `iperf_stream` 结构中的 `id` 成员。由于 `id` 是 1-based 的，我们使用 `id - 1` 作为 0-based 数组索引，为流分配一个专用的 RX 队列。

```c
// iperf_add_stream() - 在流被添加到列表并分配 ID 之后
void iperf_add_stream(struct iperf_test *test, struct iperf_stream *stream)
{
    // ... 原有的 iperf_add_stream 逻辑，例如：
    // SLIST_INSERT_HEAD(&test->streams, stream, streams);
    // stream->id = ++test->stream_list_size;

#ifdef HAVE_LIBTCPX
    // 确保是 TCP 协议且已启用 TCPX
    if (test->tcpx_dev && test->protocol->id == Ptcp) {
        // stream->id 是 1-based, 因此用 stream->id - 1 作为数组索引
        int stream_index = stream->id - 1;
        int rule_id = stream->id;

        if (stream_index < test->num_queue) {
            int queue_id = test->rxnic_ctx->queue_array[stream_index];
            
            // 初始化连接上下文
            stream->conn_ctx = tcpx_connection_init_simple(
                stream->socket,
                test->rxnic_ctx,
                test->txnic_ctx,
                queue_id,
                rule_id,
                TCPX_FLOW_STEERING_ENABLED,
                test->iobuf_size,
                test->max_tokens,
                test->tx_pipeline
            );

            if (!stream->conn_ctx) {
                iperf_err(test, "stream %d: 无法初始化 TCPX 连接上下文", stream->socket);
            } else if (test->debug) {
                iperf_printf(test, "stream %d: TCPX 连接上下文已初始化 (队列 %d)\n", stream->socket, queue_id);
            }
        } else {
            iperf_err(test, "stream %d: 没有足够的 TCPX 队列 (需求 %d, 可用 %d)", stream->socket, stream->id, test->num_queue);
        }
    }
#endif
}
```

### 第 3.6 步: 连接上下文的销毁

在流被销毁时，也需要释放对应的 `connection_ctx`。

**位置**: `iperf_free_stream()` (`src/iperf_api.c`)

**销毁逻辑**:
```c
// 在 iperf_free_stream() 的开头
#ifdef HAVE_LIBTCPX
if (sp->conn_ctx) {
    tcpx_connection_destroy(sp->conn_ctx);
    sp->conn_ctx = NULL;
}
#endif
```

### 第 3.7 步: 网卡资源的最终销毁

**位置**：`iperf_free_test()` (`src/iperf_api.c`)

**销毁逻辑**：
```c
void iperf_free_test(struct iperf_test *test) {
    // ... 现有释放代码 ...
    
#ifdef HAVE_LIBTCPX
    // 先重置流规则（包括控制连接的流规则）
    if (test->tcpx_rxdev) {
        if (tcpx_reset_flow_steering(test->tcpx_rxdev) != 0) {
            iperf_err(test, "无法重置设备 %s 上的流控规则", test->tcpx_rxdev);
        } else if (test->debug) {
            iperf_printf(test, "设备 %s 上的流控规则已成功重置\n", test->tcpx_rxdev);
        }
    }
    
    // 销毁网卡上下文
    if (test->txnic_ctx) {
        tcpx_txnic_ctx_destroy(test->txnic_ctx);
        test->txnic_ctx = NULL;
        if (test->debug) {
            iperf_printf(test, "TX NIC 上下文已销毁\n");
        }
    }
    
    if (test->rxnic_ctx) {
        tcpx_rxnic_ctx_destroy(test->rxnic_ctx);
        test->rxnic_ctx = NULL;
        if (test->debug) {
            iperf_printf(test, "RX NIC 上下文已销毁\n");
        }
    }
    
    // 释放设备名称字符串
    if (test->tcpx_rxdev) {
        free(test->tcpx_rxdev);
        test->tcpx_rxdev = NULL;
    }
    if (test->tcpx_txdev) {
        free(test->tcpx_txdev);
        test->tcpx_txdev = NULL;
    }
#endif
    
    // ... 继续现有释放代码 ...
}
```

---

### 第 4 步: 替换数据传输函数

在数据传输阶段，我们需要有条件地使用 `tcpx_recv` 和 `tcpx_send` 替换标准的 socket 操作。

#### 4.1 替换接收函数

**位置**：`iperf_tcp_recv()` (`src/iperf_tcp.c`)

**替换逻辑**：
```c
int iperf_tcp_recv(struct iperf_stream *sp)
{
    int r;
    
#ifdef HAVE_LIBTCPX
    // 检查是否使用 TCPX 接收
    if (sp->conn_ctx && sp->conn_ctx->rxnic_ctx) {
        // 使用 TCPX 零拷贝接收
        r = tcpx_recv(sp->conn_ctx, sp->buffer, sp->settings->blksize, 0);
        
        if (r < 0 && sp->test->debug) {
            iperf_printf(sp->test, "TCPX recv 失败，回退到标准 socket\n");
            // 可以选择回退到标准 socket 或直接返回错误
        }
    } else {
#endif
        // 使用标准 socket 接收
        int sock_opt;
#if defined(HAVE_MSG_TRUNC)
        sock_opt = sp->test->settings->skip_rx_copy ? MSG_TRUNC : 0;
#else
        sock_opt = 0;
#endif
        r = Nrecv_no_select(sp->socket, sp->buffer, sp->settings->blksize, Ptcp, sock_opt);
        
#ifdef HAVE_LIBTCPX
    }
#endif
    
    if (r < 0)
        return r;

    // 统计逻辑保持不变
    if (sp->test->state == TEST_RUNNING) {
        sp->result->bytes_received += r;
        sp->result->bytes_received_this_interval += r;
    } else {
        if (sp->test->debug)
            printf("Late receive, state = %d-%s\n", sp->test->state, state_to_text(sp->test->state));
    }

    return r;
}
```

#### 4.2 替换发送函数

**位置**：`iperf_tcp_send()` (`src/iperf_tcp.c`)

**替换逻辑**：
```c
int iperf_tcp_send(struct iperf_stream *sp)
{
    int r;

    if (!sp->pending_size)
        sp->pending_size = sp->settings->blksize;

#ifdef HAVE_LIBTCPX
    // 检查是否使用 TCPX 发送
    if (sp->conn_ctx && sp->conn_ctx->txnic_ctx) {
        // 使用 TCPX 零拷贝发送
        r = tcpx_send(sp->conn_ctx, sp->buffer, sp->pending_size, 0);
        
        if (r < 0 && sp->test->debug) {
            iperf_printf(sp->test, "TCPX send 失败，回退到标准 socket\n");
            // 可以选择回退到标准 socket 或直接返回错误
        }
    } else {
#endif
        // 使用标准 socket 发送
        if (sp->test->zerocopy)
            r = Nsendfile(sp->buffer_fd, sp->socket, sp->buffer, sp->pending_size);
        else
            r = Nwrite(sp->socket, sp->buffer, sp->pending_size, Ptcp);
            
#ifdef HAVE_LIBTCPX
    }
#endif

    if (r < 0)
        return r;

    // 统计逻辑保持不变
    sp->pending_size -= r;
    sp->result->bytes_sent += r;
    sp->result->bytes_sent_this_interval += r;

    if (sp->test->debug_level >= DEBUG_LEVEL_DEBUG)
        printf("sent %d bytes of %d, pending %d, total %" PRIu64 "\n",
            r, sp->settings->blksize, sp->pending_size, sp->result->bytes_sent);

    return r;
}
```

---

## 验证与测试

### 编译验证

1. **检查宏定义**：确保 `HAVE_LIBTCPX` 宏在配置时正确定义
2. **条件编译**：验证在没有 `libtcpx` 的环境中仍能正常编译
3. **链接测试**：确保所有 TCPX 函数都能正确链接

### 运行时验证

1. **参数验证**：测试各种 TCPX 参数组合
2. **错误处理**：验证各种错误情况下的行为
3. **性能测试**：对比启用/禁用 TCPX 时的性能差异
4. **资源清理**：确保所有上下文都能正确释放

### 调试建议

- 使用 `--debug` 参数获取详细的 TCPX 操作日志
- 监控 TCPX 相关的系统资源使用情况
- 验证流控规则的正确配置和重置