---

### 实现一些DPI绕过方法
该程序是一个本地SOCKS代理服务器。

用法示例：
```
ciadpi --disorder 1 --auto=torst --tlsrec 1+s
ciadpi --fake -1 --ttl 8
```

------
### 参数说明

```
-i, --ip <ip>
    监听的IP地址，默认值为 0.0.0.0

-p, --port <num>
    监听的端口，默认值为 1080

-D, --daemon
    以守护进程模式启动
    仅支持 Linux 和 BSD 系统

-w, --pidfile <filename>
    PID文件的位置

-E, --transparent
    以透明代理模式启动，SOCKS 将不会工作

-c, --max-conn <count>
    最大客户端连接数，默认值为 512

-I, --conn-ip <ip>
    绑定的出站连接IP地址，默认值为 ::
    如果指定了IPv4地址，则会拒绝IPv6请求

-b, --buf-size <size>
    每次接收/发送的最大数据大小，单位字节，默认值为 16384

-g, --def-ttl <num>
    所有出站连接的TTL值
    可能对绕过非标准/缩小TTL的检测有用

-N, --no-domain
    如果地址是域名，则丢弃请求
    由于域名解析是同步进行的，这可能会减慢甚至冻结程序的运行

-U, --no-udp
    不代理UDP

-F, --tfo
    启用TCP快速打开
    如果服务器支持，首个数据包会与SYN一起发送
    仅在Linux（4.11+）中支持

-A, --auto <t,r,s,n>
    自动模式
    如果发生类似于阻断或故障的事件，将应用该选项后面的绕过参数
    可能的事件：
        torst   : 超时或服务器在第一次请求后重置连接
        redirect: HTTP重定向，且Location中的域名与请求域名不匹配
        ssl_err : 响应ClientHello时没有收到ServerHello，或ServerHello中包含无效的session_id
        none    : 上述事件未触发，例如由于域名或协议限制

-L, --auto-mode <0|1>
    0: 只有在可以重新连接的情况下才缓存IP
    1: 即使在以下情况下也缓存IP：
        torst - 在数据交换过程中（即在收到服务器的第一批数据之后）发生超时/连接重置
        ssl_err - 只进行了一轮数据交换（请求-响应/请求-响应-请求）

-u, --cache-ttl <sec>
    缓存值的生存时间，默认值为 100800 秒（28小时）

-T, --timeout <sec>
    等待服务器第一个响应的超时时间（秒）
    在Linux中会转换为毫秒，因此可以指定小数值

-K, --proto <t,h,u,i>
    允许的协议白名单：tls,http,udp,ipv4

-H, --hosts <file|:string>
    限制参数作用范围，仅对指定的域名列表生效
    域名应当以换行符或空格分隔

-j, --ipset <file|:str>
    限制特定IP/子网

-V, --pf <port[-portr]>
    限制特定端口

-R, --round <num[-numr]>
    对哪个/哪些请求应用混淆
    默认值为 1，即应用于第一个请求

-s, --split <pos_t>
    根据指定位置拆分请求
    位置格式为 offset[:repeats:skip][+flag1[flag2]]
    标志：
        +s: 添加SNI偏移
        +h: 添加Host偏移
        +n: 零偏移
    附加标志：
        +e: 结束；+m: 中间
    示例：
        0+sm - 拆分请求于SNI中间
        1:3:5 - 拆分于位置1, 6, 11
    可多次使用此参数，以便在多个位置拆分请求
    如果offset为负且没有标志，则偏移量将加上数据包的大小

-d, --disorder <pos_t>
    类似于 --split，但发送的部分按相反顺序发送

-o, --oob <pos_t>
    类似于 --split，但将部分数据作为OOB（带外）数据发送

-q, --disoob <pos_t>
    类似于 --disorder，但部分数据作为OOB数据发送

-f, --fake <pos_t>
    类似于 --disorder，但在发送第一部分之前，先发送一部分伪造数据
    伪造数据的字节数等于拆分部分的大小
    ! 在Windows上可能不稳定工作

-t, --ttl <num>
    伪造数据包的TTL，默认值为 8
    需要选择一个TTL值，使得数据包无法到达服务器，但会被DPI处理

-S, --md5sig
    为伪造数据包设置TCP MD5签名选项
    大多数服务器（尤其是Linux服务器）会丢弃带有此选项的数据包
    仅支持Linux，某些内核版本（<3.9，Android）可能会禁用此选项

-O, --fake-offset <pos_t>
    偏移伪造数据的开始位置
    带有标志的偏移是相对于原始请求计算的

-l, --fake-data <file|:str>
    指定自定义伪造数据包
    字符串可以包含转义字符（\n,\0,\0x10）

-e, --oob-data <char>
    发送带外数据的字节，默认值为 'a'
    可以指定ASCII字符或转义字符

-n, --fake-sni <str>
    动态更改伪造数据包中的SNI
    如果伪造数据包的大小大于请求的大小，伪造数据包会被缩小（修改Padding、ECH或移除某些扩展）
    "?" 替换为随机字母，"#" 替换为数字，"*" 替换为字母或数字
    可以多次使用该选项，每个请求会从指定的SNI列表中随机选择一个

-Q, --fake-tls-mod <r,o>
    rand - 使用随机数据填充 SessionID、Random 和 KeyExchange 字段
    orig - 使用原始的ClientHello作为伪造数据包

-M, --mod-http <h[,d,r]>
    对HTTP数据包进行各种操作，可以组合使用
    hcsmix:
        "Host: name" -> "hOsT: name"
    dcsmix:
        "Host: name" -> "Host: NaMe"
    rmspace:
        "Host: name" -> "Host:name\t"

-r, --tlsrec <pos_t>
    根据指定的偏移拆分ClientHello数据
    可以多次使用该选项

-a, --udp-fake <count>
    伪造UDP数据包的数量

-Y, --drop-sack
    忽略SACK，迫使内核重新发送已传送的数据包
    仅在Linux中支持
```
### 更多信息  
`--split`

将请求拆分为多个部分。以30字节的请求为例：
- 参数：`--split 3 --split 7`
- 发送顺序：1-3, 3-7, 7-30  

位置应该按升序指定。

------
`--disorder`

进入`disorder`的部分将被发送时TTL=1，也就是说，实际上数据不会传递到任何地方。
操作系统只有在发送下一部分时才能意识到这一点，服务器会通过SACK报告丢失。
系统将不得不重新发送之前的包，从而打乱正常的顺序。
- 参数：`--disorder 7`
- 发送顺序：7-30, 1-7  

上述内容仅适用于Linux。
在Windows中，重传从丢失位置开始（从服务器接收到的最大ACK）：
- 参数：`--disorder 7`
- 发送顺序：7-30, 1-30  

因此，建议同时使用`split`：
- 参数：`--split 7 --disorder 23`
- 发送顺序：1-7, 23-30, 7-30  

实际中，最优使用方式是：  
* Linux：`--disorder 1`
* Windows：`--split 1+s --disorder 3+s`

------
`--fake`

- 参数：`--fake 7`
- 发送顺序：1-7 假数据, 7-30 原始数据, 1-7 原始数据

请求的第一部分数据会被伪造数据替换。  
这一部分需要通过DPI，但不能到达服务器。  
由于部分数据没有到达，操作系统会重新发送它，从而改变顺序，就像`disorder`一样。
为了避免伪造数据到达服务器，可以使用`ttl`和`md5sig`选项。

TTL值需要调整，使得数据包可以通过所有DPI，但不会到达服务器。  
对于Linux，可以使用md5sig。它设置了TCP MD5签名选项，防止许多服务器接受这个包。  
遗憾的是，md5sig并不是所有版本的内核都支持。

对于Windows，避免伪造数据被服务器处理的另一种方法是将`fake`与`disorder`结合：
- 参数：`--disorder 1 --fake 7`
- 发送顺序：2-7 假数据, 7-30 原始数据, 1-30 原始数据  

如果伪造数据包到达服务器，它将因为完全重传而被覆盖。

实际中，最优使用方式是：  
* Linux：`--fake -1 --md5sig`
* Windows：`--disorder 1 --fake -1`

------
`--oob`

TCP可以使用URG标志将数据发送到主数据流之外，但每个包仅能发送1个字节数据。  
该包中的所有数据都将被应用程序接收，除了最后一个字节，它将作为OOB数据：
- 参数：`--oob 3`
- 发送顺序：1-4 使用URG标志（1-3是数据，4是会被截断的字节），3-30  

这个字节最好放到SNI中：`--oob 3+s`

------
`--disoob`

与`--disorder`类似，但其中一部分会被发送为OOB字节：
- 参数：`--disoob 3`
- 发送顺序：3-30, 1-4 使用URG标志（1-3是数据，4是会被截断的字节）

将`--fake`或`--disorder`与此一起使用时，可以使包在拆分位置插入OOB字节：
- 参数：`--disoob 3 --disorder 7`
- 发送顺序：3-30, 1-8 使用URG标志（1-3是数据 + 被截断的字节 + 4-8）

------
`--tlsrec`

可以将一个TLS记录拆分成多个，通过稍微修改其头部。
在拆分的位置插入一个新的头部，增加请求大小5字节。

这个头部可以放到SNI的中间，从而使DPI无法正确读取：
`--tlsrec 3+s`

虽然`tlsrec`和`oob`可以干扰DPI，但它们也可能干扰不支持完整TCP/TLS栈的中间件。
因此，应与`--auto`一起使用：
`--auto=torst --timeout 3 --tlsrec 3+s`  
在这个例子中，`tlsrec`只有在连接被重置或超时时才会应用，即当可能发生封锁时。  
也可以反过来，在服务器重置连接或丢弃数据包时禁用`tlsrec`：
`--tlsrec 3+s --auto=torst --timeout 3`

------
`-Y, --drop-sack`

让内核忽略带有TCP SACK扩展的包。
该扩展允许确认单独数据段的接收。
如果请求的第一部分丢失，而第二部分到达服务器，服务器可以通过此扩展通知客户端。客户端知道第二部分到达后，将只发送第一部分。  
为什么要忽略这个扩展？第二部分可能是伪造数据。如果它到达服务器，但客户端没有意识到，它将尝试重新发送。然而，这个数据包将包含原始数据，覆盖伪造数据，从而防止协议的崩溃。  
由于快速确认将无法工作，这将打乱`disorder`，并且会在重传之前引入延迟（约200ms）。

------
`--auto`, `--hosts`

`auto`参数将选项分为几个组。
每个请求按顺序检查这些选项。
首先检查`auto`中指定的触发器，然后是`pf`、`ipset`、`proto`和`hosts`。

可以指定多个选项组，用`auto`参数进行分隔。  
`--timeout`下的参数可以提取到单独的组中。

#### 示例：
```
--fake -1 --ttl 10 --auto=ssl_err --fake -1 --ttl 5
```
默认使用`fake`和ttl=10，在发生错误时使用`fake`和ttl=5

```
--hosts list.txt --disorder 3 --auto=none
```
仅对`list.txt`中的域应用扰乱

```
--hosts list.txt --auto=none --disorder 3
```
不对`list.txt`中的域应用扰乱

```
--auto=torst --hosts list.txt --disorder 3
```
默认不做任何事，在发生封锁并且域名在`list.txt`中的情况下使用`disorder`。

```
--proto=http,tls --disorder 3 --auto=none
```
仅对HTTP和TLS进行扰乱

```
--proto=http --fake -1 --fake-data=':GET /...' --auto=none --fake -1
```
重写HTTP的伪造数据包

------
### 构建
构建所需：  
`make`，`gcc/clang`用于Linux，`mingw`用于Windows

* Linux: `make`
* Windows: `make windows CC=x86_64-w64-mingw32-gcc`

------
### 其他DPI信息，灵感来源  
* https://github.com/bol-van/zapret/blob/master/docs/readme.md  
* https://geneva.cs.umd.edu/papers/geneva_ccs19.pdf  
* https://habr.com/ru/post/335436  
