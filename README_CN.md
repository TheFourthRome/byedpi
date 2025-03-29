# DPI 绕过方法的实现
该程序是一个本地的 SOCKS 代理服务器。

## 使用示例：
```
ciadpi --disorder 1 --auto=torst --tlsrec 1+s
ciadpi --fake -1 --ttl 8
```

## 参数说明：
```
-i, --ip <ip>
    监听的 IP 地址，默认为 0.0.0.0

-p, --port <num>
    监听的端口，默认为 1080

-D, --daemon
    以守护进程模式运行，仅支持 Linux 和 BSD 系统

-w, --pidfile <filename>
    PID 文件存储路径

-E, --transparent
    以透明代理模式运行，SOCKS 不工作

-c, --max-conn <count>
    最大客户端连接数，默认为 512

-I, --conn-ip <ip>
    绑定的外部连接地址，默认为 ::
    如果指定了 IPv4 地址，IPv6 请求将被拒绝

-b, --buf-size <size>
    每次 recv/send 调用接收或发送的数据最大字节数，单位为字节，默认为 16384

-g, --def-ttl <num>
    所有外部连接的 TTL 值
    可用于绕过 TTL 异常或减小 TTL 的检测

-N, --no-domain
    丢弃域名请求
    因为解析是同步执行的，它可能会减慢或冻结程序

-U, --no-udp
    不代理 UDP 请求

-F, --tfo
    启用 TCP 快速打开（TCP Fast Open）
    如果服务器支持，第一个包将与 SYN 一起发送
    仅支持 Linux (4.11+)

-A, --auto <t,r,s,n>
    自动模式
    当检测到类似阻止或失败的事件时，自动应用后续的绕过参数
    可能的事件：
        torst   : 超时或服务器在首次请求后重置连接
        redirect: HTTP 重定向，其 Location 域名与原请求不匹配
        ssl_err : ClientHello 后没有收到 ServerHello 或 session_id 错误
        none    : 跳过前述事件，如由于域名或协议限制

-L, --auto-mode <0|1>
    0: 仅当有重连机会时缓存 IP
    1: 即使在以下情况下也缓存 IP:
        torst - 超时或连接在数据交换期间被重置
        ssl_err - 仅发生了一个数据交换循环

-u, --cache-ttl <sec>
    缓存存活时间，默认为 100800（28 小时）

-T, --timeout <sec>
    等待服务器首个响应的超时时间，单位为秒
    在 Linux 中转换为毫秒，因此可以使用小数

-K, --proto <t,h,u,i>
    允许的协议白名单：tls,http,udp,ipv4

-H, --hosts <file|:string>
    限制操作范围到指定的域名列表
    域名应以换行符或空格分隔

-j, --ipset <file|:str>
    限制特定 IP/子网

-V, --pf <port[-portr]>
    限制端口范围

-R, --round <num[-numr]>
    指定请求的顺序应用扰乱规则
    默认为 1，即应用于首个请求

-s, --split <pos_t>
    按指定位置拆分请求
    格式为 offset[:repeats:skip][+flag1[flag2]]
    示例：
        0+sm - 在 SNI 中拆分请求
        1:3:5 - 在位置 1, 6, 和 11 拆分

-d, --disorder <pos_t>
    与 --split 类似，但数据发送顺序被颠倒

-o, --oob <pos_t>
    类似 --split，但部分数据通过 OOB（超出带外）数据发送

-q, --disoob <pos_t>
    与 --disorder 类似，但部分数据通过 OOB 数据发送

-f, --fake <pos_t>
    类似 --disorder，但在发送第一部分数据之前，先发送部分伪造数据
    伪造数据的字节数与分割部分的字节数相等

-t, --ttl <num>
    伪造数据包的 TTL 值，默认为 8

-S, --md5sig
    启用 TCP MD5 签名选项用于伪造数据包

-O, --fake-offset <pos_t>
    调整伪造数据的开始位置

-l, --fake-data <file|:str>
    使用自定义的伪造数据包

-e, --oob-data <char>
    发送超出带外的数据，默认为 'a'

-n, --fake-sni <str>
    动态修改伪造数据包中的 SNI

-Q, --fake-tls-mod <r,o>
    rand - 使用随机数据填充 SessionID, Random 和 KeyExchange
    orig - 使用原始的 ClientHello 作为伪造数据包

-M, --mod-http <h[,d,r]>
    对 HTTP 包进行修改，可以组合使用

-r, --tlsrec <pos_t>
    将 ClientHello 拆分为多个记录

-a, --udp-fake <count>
    伪造 UDP 包的数量

-Y, --drop-sack
    忽略 TCP SACK 扩展，强制内核重新发送已经传输的数据包

```

## 详细说明
**`--split`**
将请求拆分为多个部分。示例：  
- 参数：`--split 3 --split 7`
- 发送顺序：1-3, 3-7, 7-30  

**`--disorder`**
颠倒部分请求的发送顺序。示例：  
- 参数：`--disorder 7`
- 发送顺序：7-30, 1-7  

**`--fake`**
替换请求的某部分为伪造数据，确保这部分数据经过 DPI 检查，但不被送到服务器。示例：  
- 参数：`--fake 7`
- 发送顺序：1-7 伪造数据，7-30 原始数据，1-7 原始数据  

**`--oob`**
发送带有 OOB（带外数据）标志的数据包，其中最后一个字节是带外字节。示例：  
- 参数：`--oob 3`
- 发送顺序：1-4 使用 URG 标志（1-3 数据 + 4 作为带外字节），3-30  

**`--disoob`**
类似 `--disorder`，但数据通过带外字节发送。示例：  
- 参数：`--disoob 3`
- 发送顺序：3-30, 1-4 使用 URG 标志  

## 编译
需要使用 `make` 和 `gcc/clang`（用于 Linux），`mingw`（用于 Windows）。  
Linux：`make`  
Windows：`make windows CC=x86_64-w64-mingw32-gcc`

## 额外信息
* [DPI 绕过和防火墙规避的源文档](https://github.com/bol-van/zapret/blob/master/docs/readme.md)
* [Geneva: A Practical Framework for Evaluating and Exploiting Deep Packet Inspection](https://geneva.cs.umd.edu/papers/geneva_ccs19.pdf)
* [Habr 文章](https://habr.com/ru/post/335436)

--- 

这就是该项目 README 文件的中文翻译！
