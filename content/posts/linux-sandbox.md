+++
title = 'Agent 沙箱介绍'
date = 2026-08-10T09:30:00+08:00
draft = false
summary = '从功能维度拆解 Agent 沙箱：文件怎么防护、网络怎么防护、进程怎么隔离、特权怎么收走、资源怎么限制，再以 Claude Code 的 sandbox-runtime 为例，看这些机制如何组合成一套可用的 Linux 沙箱。'
tags = ['linux', 'sandbox', 'namespace', 'seccomp', 'landlock']
categories = ['Engineering']
+++

Agent 沙箱是 Agent 跑代码时准备的隔离环境。Claude Code 会自己决定执行什么命令、读写哪些文件、装什么依赖。这些动作是 Agent 自主做的，由于模型是概率输出的，因此有可能会执行 `rm -rf` 这样的把整个系统文件删掉的危险操作。沙箱的作用，就是即使模型真的尝试去执行这样的危险命令，利用沙箱的安全机制，让这个命令不能执行成功。

这篇文章以 Claude Code 的 Linux 沙箱机制 [sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime) 为例子。不讲"Linux 有哪些隔离名词"，而是按功能维度讲：**文件怎么防护、网络怎么防护、进程怎么隔离、特权怎么收走、资源怎么限制**。每个维度先把必要的基础背景融入进来，再看内核和工具链具体怎么做。

## 1. 一张维度地图

先把整篇文章的骨架画出来。Agent 沙箱要防的事情，可以按五个维度拆：

| 维度 | 要防的威胁 | 核心机制 |
| --- | --- | --- |
| 文件 | 读到不该读的（`~/.ssh`、token）、写坏系统文件（`rm -rf`） | mount namespace、bind mount、只读挂载、路径遮蔽、tmpfs |
| 网络 | 偷偷外联、访问未授权域名、本地 socket 绕过 | network namespace 断网、代理 + 域名白名单、seccomp 堵 Unix socket |
| 进程 | 看到/操控宿主进程、留下后台残留 | PID namespace、重新挂载 `/proc`、退出即回收 |
| 特权 | 被攻破后提权成 root | user namespace、capabilities、no_new_privs、seccomp |
| 资源 | 命令无限运行、吃光 CPU/内存 | cgroup、rlimit、超时 |

一个真实的沙箱不是某一个机制，而是这五类机制的组合拳。下面逐维度展开。

## 2. 文件防护

### 2.1 背景：进程凭什么访问文件

进程的身份由三组数字组成：`uid`（用户 ID，进程以谁的身份运行）、`gid`（主组 ID）、`groups`（附加组列表，一个用户可同时属于多个组）。在 shell 里敲一下 `id` 就能看到当前进程的这三组数字。

每个文件有三档权限：owner（文件主人）、group（授权给的团队）、others（其他所有人），每档是一组读/写/执行位。进程要读或写一个文件时，内核拿进程的 uid/gid/groups 去匹配：命中 owner 就看 owner 那档，命中 group 就看 group 那档，都不是就看 others。整个过程在内核里完成，程序只能拿到"允许"或"拒绝"，绕不过去。

一个例子：普通用户读 `/root`。进程 uid 是 1000，不是 `/root` 的 owner（root）；gid/groups 也不在 root 组里；只能落到 others，而 `/root` 的 others 没有读权限，内核直接返回 Permission denied。

一句话直觉：**uid/gid/groups 是进程的身份，文件的三档权限是门禁规则，内核拿身份对规则，决定读写是否放行。**

### 2.2 背景：目录树是怎么拼出来的

进程访问文件走的是路径：从 `/` 开始一级一级"走"，每进一个目录就查下一级。目录本身也是一种文件，但它存的不是内容，而是"名字 → 数据位置"的指针，内容都在别处。

数据不一定在硬盘上：tmpfs 在内存里（`/tmp` 可以只存在内存中）、procfs/sysfs 是内核虚拟的（读的那一刻现场生成）、NFS 在网络上。每个数据源自带一套目录结构，这就是"文件系统"的真正含义：一套放在某个来源里的目录树。

这么多独立的树，怎么合成一棵 `/` 树？靠 mount（挂载）：把一个文件系统的"根"接到目录树的一个目录点（挂载点）上。挂上之后，从那个点往下走的路径就进入该数据源的内容；`umount` 就是拔掉。Linux 只有一棵 `/` 树，每个数据源都是这样"接"进去的：`/tmp` 是内存盘接的，`/proc` 是内核虚拟树接的。

两个容易混的概念：

- **虚拟文件系统**（`/proc` 等）说的是"数据源不在磁盘上、读的时候内核现编"，本质还是文件系统。
- **namespace** 说的是"进程的视角隔离"。可以这样记：虚拟文件系统是公告栏（内容实时画），namespace 是房间——不同房间的人看同一块公告栏，但内容按房间过滤。PID namespace 决定 `/proc` 里列出哪些进程，mount namespace 决定挂载表，network namespace 决定网卡。

### 2.3 机制：给进程一棵重新布置的目录树

**mount namespace** 让每个进程拥有一份独立的挂载视图。沙箱利用这一点，给进程一棵"重新布置过的目录树"，而不是靠程序自觉不去碰。sandbox-runtime 在 Linux 上生成的挂载参数长这样：

- `--ro-bind / /`：把整棵宿主根目录 bind 进沙箱，并挂成**只读**。进程能读到系统文件，但写不进去——`rm -rf` 到系统目录这一步就会被内核拒绝。
- `--bind <workspace> <workspace>`：把 workspace 单独挂回**可写**，命令只在这一块地上有写权限。
- `--ro-bind /dev/null <敏感文件>`：用"无底洞"遮住单个敏感文件（token、`~/.ssh` 里的 key），读它只能得到空。
- `--tmpfs <敏感目录>`：在敏感目录上盖一块"临时黑板"，里面空空如也、写啥都丢。
- 受控的 `$TMPDIR`：把临时目录也隔离出来，命令可以写临时文件，但退出即消失，不污染宿主。

这些参数背后是内核的 `mount(2)`：bind mount（`MS_BIND`）把宿主目录原样映射到另一个路径，`MS_RDONLY` 把挂载点改成只读，tmpfs 挂载内存里的文件系统。mount namespace（`CLONE_NEWNS`）则是给进程一份自己的"挂载地图"，之后它看到的挂载关系可以和宿主不同。

一个细节：bind mount 只能遮蔽"已经存在"的文件。用 `--ro-bind /dev/null <path>` 遮一个不存在的路径时，bwrap 会在宿主文件系统上创建一个空的挂载点文件；命令结束后，这个"幽灵文件"会留在工作目录里（比如 `.bashrc`、`.gitconfig`）。sandbox-runtime 为此维护了一套挂载点清理逻辑，并且只在没有其他沙箱同时运行时才删除——删早了会让还在运行的沙箱的遮蔽失效。

（普通用户本来没有 `CAP_SYS_ADMIN` 不能挂载，这里能挂载是因为 user namespace 在 namespace 内提供了这份特权，详见第 5.2 节。）

### 2.4 机制：危险文件强制保护

除了用户配置，还有一批"永远禁止写入"的路径：`.bashrc`、`.zshrc`、`.gitconfig`、`.mcp.json`、`.ripgreprc`、`.claude/commands/`、`.claude/agents/`、`.git/hooks/`、`.git/config`、`.vscode/`、`.idea/` 等。

但 bind 规则只对已存在的路径生效，所以运行时用一次 ripgrep 扫描允许写的目录（默认往下 3 层，可配 1-10 层），把扫到的危险文件/目录转成额外的 deny 挂载参数。这就是 ripgrep 在实现里的角色：不是沙箱边界，而是"把配置翻译成挂载规则"之前的路径扫描器。

### 2.5 文件维度小结

文件防护的核心不是"告诉程序别碰"，而是让进程**在路径层面根本找不到、写不进**它不该碰的内容：整棵系统目录只读，workspace 单独可写，敏感路径被遮蔽，危险文件被扫描后逐一钉死。

## 3. 网络防护

### 3.1 背景：网络是怎么走的

把网络理解成寄快递：

- **网卡** = 出入口。有线网卡 `eth0`、无线网卡 `wlan0`；`lo` 是虚拟回环口，专门"自己和自己说话"（本机进程访问本机服务走它）
- **IP 地址** = 门牌号，找到"哪台电脑"。数据包到达后，操作系统再看端口号——端口是"房间号"，决定交给哪个程序（网页 443/80、SSH 22、数据库 3306）。IP 找楼，端口找房间
- **网段** = 小区。子网掩码把 IP 拆成"小区号 + 房间号"，目标在同网段 = 邻居，直接送；不同网段 = 出远门，交给别人转送
- **网关** = 转运站。所有要出远门的数据先送到它那里，它自己手里也有一张路由表，一层层转送
- **路由表** = 遇到不同目的地走哪条路。最后那行兜底叫**默认路由**（Linux 里写成 `default via <网关IP> dev <网卡>`）：不知道怎么走的目标，都交给默认网关

进程就是通过网络接口、IP、路由表和网关访问网络的。要断一个进程的网，就从这四样东西下手。

### 3.2 机制：默认断网

network namespace 隔离的是整套网络视图：进程能看到哪些网卡、哪些 IP、哪些路由、有没有默认网关。`bwrap --unshare-net` 创建的空 network namespace 里，网络链路需要的每一样东西都缺：

1. 没有网卡 → 没有出入口，数据根本出不了门
2. 没有路由表 → 就算有网卡，也不知道往哪送
3. 没有默认网关 → 就算知道要出门，也没有转运站接手

唯一剩下的 `lo` 只能"自己和自己说话"（启起来后 `ping 127.0.0.1` 能通），对外一个字节都发不出去。所以断网是**结构性**的：不是程序被禁止联网，而是这个世界里根本没有路——程序再多花招也没用。

### 3.3 机制：受控联网——代理与域名白名单

默认断网之后，允许的流量怎么"接回来"？靠代理（proxy）= 中间人。程序不直接联系目标服务器，而是把请求交给代理，代理替它联系目标、拿到结果再带回来，像代购。代购的关键性质是**他能检查你想买什么**——沙箱就靠它做域名白名单：请求到达代理，代理比对 `allowedDomains`/`deniedDomains`，放行或拒绝（拒绝时返回类似 `Connection blocked by network allowlist` 的提示）。

程序怎么找到代理？靠环境变量：沙箱注入 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`，程序一读就知道"去 `127.0.0.1:3128` 找代购"。（不认这些环境变量的程序会直接连不上网络——失败即拒绝，而不是绕过。）

HTTP 代理和 SOCKS5 代理的区别，在于"懂多少"：

| | HTTP 代理 | SOCKS5 代理 |
| --- | --- | --- |
| 懂什么 | 只懂 HTTP/HTTPS 协议，能看懂域名、路径、甚至内容 | 不懂任何应用层协议，纯搬运字节流 |
| 能干什么 | 域名过滤、内容检查、缓存、认证 | 任何 TCP 流量都能转：SSH、git、数据库 |
| 类比 | 懂日语的代购，能看说明书帮你挑 | 纯快递公司，不管箱子里是啥 |

沙箱两个都起：网页/API/npm 这类 HTTP 流量走 HTTP 代理；git 走 SSH、数据库连接这类非 HTTP 流量交给 SOCKS5。细节：HTTPS 走 HTTP 代理时其实是 CONNECT 隧道——代理也只"搬箱子"不拆箱，只有开启 TLS 终止（`tlsTerminate`，代理伪装成目标服务器解密流量）后才能真正看见内容。

整个网络链路是三个角色分工：

- `bwrap`（原生 C）负责"断"：`--unshare-net` 清空沙箱的网络世界
- `socat`（原生 C）负责"桥"：Unix socket 和端口之间搬数据（纯搬运，不过问内容）
- JS 代理（TS）负责"查"：域名白名单逻辑全在这里，是 sandbox-runtime 自己写的业务部分（`src/sandbox/http-proxy.ts` 和 `src/sandbox/socks-proxy.ts`，跑在宿主机上）

沙箱内的传输链路：沙箱内程序 → 沙箱内 `127.0.0.1:3128/1080`（socat 监听）→ 转成 Unix socket 请求（这个 socket 被 bind mount 进沙箱）→ 宿主侧 socat 收到 → 转给真正的 JS 代理 → 代理查白名单 → 放行则访问外网。

这里有个容易混的点：`127.0.0.1` 叫回环地址，含义是"我自己"，不是任何物理机的地址。每个 network namespace 都有自己的"自己"（那根 `lo`），所以沙箱里的 `127.0.0.1:3128` 是沙箱自己 lo 上的 socat，宿主代理监听的是宿主自己 lo 上的端口，两边是"各自房间里的自己"。代理和沙箱物理上是同一台机器，它们之间靠 bind 进沙箱的 Unix socket 通信——Unix socket 走的是文件系统，根本不经过物理网络，所以代理不需要物理 IP。只有代理决定放行、真正替沙箱访问外网时，才从宿主 `eth0` 的真实 IP 出去。

### 3.4 机制：堵住本地绕过

外网被 network namespace 切掉、HTTP/HTTPS 必须经过代理之后，还有一个绕过路径：**本地 Unix socket**。如果进程还能随便创建 Unix socket，它可能去连接 `/var/run/docker.sock`、SSH agent、GPG agent、系统服务 socket 或其他本地 agent socket。这类访问不一定经过域名白名单代理，而且本地服务的权限可能比普通网络请求大得多。

sandbox-runtime 用 seccomp 堵这条路径：`apply-seccomp` 给用户命令安装 BPF 过滤器，让 `socket(AF_UNIX, ...)` 返回 `EPERM`；同时挡掉 `io_uring_setup`/`io_uring_enter`/`io_uring_register`，因为 Linux 5.19+ 可以用 `IORING_OP_SOCKET` 绕过 `socket()` 这条规则。

时序上有讲究：不能一开始就安装过滤器，因为代理桥接用的 `socat` 自己也需要 Unix socket。所以先让 `socat` 启动，再用 `apply-seccomp` 给真正的用户命令加限制——既保留受控代理能力，又阻断用户命令自己创建新的 Unix socket。

### 3.5 网络维度小结

网络防护是三段式：**network namespace 结构性断网 → 代理按域名白名单把允许的流量接回来 → seccomp 堵住 Unix socket 这类本地绕过**。

## 4. 进程隔离

### 4.1 机制：PID namespace 和 /proc

PID namespace 隔离的是进程编号和进程树。进入新的 PID namespace 后，当前进程变成这个世界的 1 号进程，子进程从这里开始编号；同一个进程在外面依然保留它原来的 PID。

配套要做的是重新挂载 `/proc`。`ps`、`top` 这类工具看到的世界来自 `/proc`；PID namespace 换掉之后，如果 `/proc` 还连着宿主，进程列表就会穿帮——这正是沙箱要防止的。bwrap 的 `--proc /proc` 干的就是这件事。

### 4.2 机制：嵌套一层，连"看守"也藏起来

为什么 `apply-seccomp` 不直接"安装过滤器然后 exec"，而是要在外层 bwrap 沙箱里再创建一层嵌套的 user + PID + mount namespace？

原因：如果用户命令和 `socat`、bash 包装脚本住在同一个 PID namespace 里，用户命令就能看到这些"没有装过滤器"的进程，攻击者可以尝试 `ptrace` 它们或读写 `/proc/<pid>/mem`，从而绕过 seccomp 限制。放进嵌套 PID namespace 后，用户命令视角里的 `/proc` 只剩自己这棵小进程树，bwrap 的 init、bash 包装、socat 都不可见、不可寻址。

嵌套后的进程布局：

- bwrap init（外层 PID 1，无 seccomp）
  - bash 包装 / socat 桥（外层，无 seccomp）
    - apply-seccomp（外层，等待内层退出并转发退出码）
      - ═══ 嵌套 PID namespace 边界 ═══
        - apply-seccomp（内层 PID 1，`PR_SET_DUMPABLE=0`，负责收割子进程、转发信号）
          - 用户命令（内层 PID 2，seccomp 已生效）

用户命令的视角：`/proc` 里只有自己的进程树。内层 PID 1 还设置 `PR_SET_DUMPABLE=0` 让自己也不可被 ptrace/转储，并负责把外部信号转发给用户命令（PID 1 如果不处理信号，外面发来的 `SIGTERM` 会被静默丢弃）。

### 4.3 机制：退出不留后台残留

命令里用 `&` 拉起的后台进程，如果不处理，会逃逸到宿主。bwrap 用两个参数解决：

- `--new-session`：让沙箱进程自成会话、和终端脱钩（对应 `setsid(2)`）
- `--die-with-parent`：内核在 bwrap 退出时向沙箱进程发 SIGKILL（对应 `prctl(2)` + `PR_SET_PDEATHSIG`），后台进程被一并带走

### 4.4 进程维度小结

进程隔离的目标：让沙箱内的进程**只看得到自己这棵小进程树**，连"看守它的那些进程"都看不到、够不着；命令一结束，整棵进程树连同后台进程一起消失。

## 5. 特权与提权防护

### 5.1 capabilities：root 权限被拆开了

Linux 里的 root 权限不是一个不可分割的整体，而是拆成了一组 capabilities：挂载文件系统需要 `CAP_SYS_ADMIN`，修改网络配置需要 `CAP_NET_ADMIN`，调试别的进程需要 `CAP_SYS_PTRACE`。普通用户真正生效的能力集（`CapEff`）是空的，所以直接挂载 tmpfs 会失败。

沙箱的做法是 `--cap-drop ALL`：把新身份空间里的全部 capability 收走。这样即使进程"看起来是 root"，也没有任何特权钥匙，`mount`、`ptrace`、改网络配置都会被内核拒绝。

### 5.2 user namespace：小世界里的 root

那普通用户怎么获得挂载文件系统这类"看起来要 root"的能力？靠 user namespace：`unshare(CLONE_NEWUSER)` 创建一个新的身份空间，把当前用户映射成里面的 root（`/proc/self/uid_map` 写着"namespace 里的 0 ↔ 外面的 1000"，默认 1:1，保持原 uid）。

关键性质：这个 root 的 capabilities **只对这个 namespace 内的资源生效**。里面有 `CAP_SYS_ADMIN` 不等于能改外层真实系统——所以只有 user namespace 时想挂载仍然会被拒（没有属于自己的挂载视图可以改）；配上 mount namespace 之后，挂载就成功了。组合关系：

- 普通用户：没有有效 capability，挂载失败
- user namespace root：有里面的 capability，但不能随便改外层资源
- user + mount namespace：有里面的 `CAP_SYS_ADMIN`，也有自己的挂载视图，挂载成功

### 5.3 no_new_privs：防提权保险丝

`no_new_privs` 是一个单向开关。打开后，当前进程和它的子进程不能再通过 `execve` 获得新的权限——即使去执行 setuid 程序或带文件 capability 的程序，也不能借机提权。它有两个关键性质：一旦打开就无法由普通程序关回；会沿进程链继承给所有子进程。

隔离机制负责限制"能碰什么"，但如果进程还能借 setuid 提权，隔离就被绕过了；no_new_privs 就是把这条提权路径直接焊死。它也是安装 seccomp 过滤器的前提——普通进程要在不提权的前提下安装过滤器，通常需要先打开它。

### 5.4 seccomp：syscall 级兜底

前面的机制回答的是"进程处在哪个世界里"，seccomp 回答的是另一件事：**这个进程还能向内核发起哪些系统调用**。普通程序读文件、开 socket、创建进程、挂载文件系统，本质上都要通过 syscall 进入内核；seccomp 就是在 syscall 入口处加过滤器，可以决定允许、返回 `EPERM`/`ENOSYS`、直接杀死进程、或记录日志。

真实沙箱不会去禁 `getpid` 这类无害调用，而是限制危险 syscall：

- `mount`：禁止重新挂载文件系统
- `ptrace`：禁止调试/注入其他进程
- `bpf`：禁止加载 eBPF 程序
- `keyctl`：禁止访问内核 keyring
- `reboot`：禁止重启系统
- `setns` / `unshare`：禁止加入或创建新的 namespace
- `socket` / `connect`：禁止创建网络 socket、主动连接外部地址

如果说 network namespace 是"这个世界里没有外网网卡"，那么 seccomp 禁 `socket/connect` 更像是"即使你看得到网络相关对象，也不允许你调用创建连接的内核入口"。

### 5.5 特权维度小结

- namespace 决定进程看到哪个世界
- capabilities 决定进程在这个世界里有什么特权（沙箱直接收走全部）
- no_new_privs 保证子进程不能再获得新权限
- seccomp 决定进程还能调用哪些内核入口

## 6. 资源限制

前面几类机制划的是"边界"，资源限制管的是"额度"：

| 能力 | 解决的问题 | 直观理解 |
| --- | --- | --- |
| rlimit / `ulimit` | 限制单进程资源 | 给进程"最多只能用这么多"的额度，超了报错（如 `Too many open files`） |
| cgroup v2 | 限制一组进程的 CPU、内存、进程数、IO | 把整组进程放进"资源盒子"，内核按组硬性限流 |
| 超时 / 进程回收 | 防止命令无限运行 | 跑超时就杀掉，退出时整棵进程树一起清理 |

顺带两个相关的文件访问机制：**Landlock** 让普通进程不靠 root 就能自己声明"只能读写这些路径"，内核强制执行；**AppArmor / SELinux** 是发行版级的强制访问控制。另外，`chroot` 只改进程看到的"根目录指针"，不隔离进程树、网络、挂载，单独用很容易逃出，所以现代实现用 mount namespace + bind mount 而不是 chroot。这些不是每套沙箱都必需，但属于同一类"定细节"的能力。

## 7. 落地：sandbox-runtime 怎么把维度组合起来

### 7.1 一条完整的 bwrap 命令

把上面五个维度的机制，落到 sandbox-runtime 拼出的一条命令上（`linux-sandbox-utils.ts` 里 `wrapCommandWithSandboxLinux` 的强沙箱参数，形状大致是）：

```ts
spawn("bwrap", [
  "--new-session", "--die-with-parent",     // 进程：退出不留后台残留
  "--unshare-net",                          // 网络：结构性断网
  "--unshare-pid",                          // 进程：新的进程树
  "--unshare-user", "--cap-drop", "ALL",    // 特权：小世界 root + 收走全部 capability
  "--ro-bind", "/", "/",                    // 文件：整棵系统目录只读
  "--bind", workspace, workspace,           // 文件：workspace 单独可写
  "--dev", "/dev",                          // 设备：最小化的 /dev
  "--proc", "/proc",                        // 进程：重新挂载配套的 /proc
  "--",
  "bash", "-c", command,
])
```

每一组参数背后都是一类系统接口：`--unshare-*` 对应 `clone(2)`/`unshare(2)` 的 namespace 标志，`--ro-bind`/`--bind` 对应 `mount(2)` 的 bind mount，`--proc`/`--dev` 对应挂载 procfs 和最小化的 `/dev`，`--cap-drop ALL` 对应清空 capabilities，`--die-with-parent` 对应 `prctl(2)` 的 parent-death signal。

（`mount(2)` 里的 `(2)` 不是版本号，是 Linux man page 的章节编号：`(1)` 用户命令、`(2)` 系统调用、`(3)` C 库函数、`(5)` 文件格式、`(7)` 概念说明、`(8)` 管理员命令。`mount(2)` 强调的是"内核系统调用"，`mount(8)` 才是 shell 里执行的管理命令。）

整个调用链是：

1. Claude Code / sandbox runtime 通过 Node/Bun 的 `child_process.spawn` 启动 `bwrap`
2. bwrap 调用 `unshare`/`clone` 创建新的 namespace
3. bwrap 调用 `mount` 配置只读根目录、可写 workspace、tmpfs、`/proc`、`/dev`
4. bwrap 设置环境变量和工作目录
5. bwrap 最后 `execve("bash", ["bash", "-c", command], env)`；如果是网络白名单模式，前面还有 socat 桥接和 apply-seccomp 两步

TypeScript 本身没有直接调用 `clone(2)`/`mount(2)` 这些 C 风格系统调用，它只负责拼参数；真正调用内核接口的是 bwrap、socat、apply-seccomp 这些原生程序。

### 7.2 二进制依赖

| 二进制 | 来源 | 角色 | 依赖的系统接口 |
| --- | --- | --- | --- |
| `bwrap` / `bubblewrap` | 发行版包（`apt install bubblewrap`、`dnf install bubblewrap`、`pacman -S bubblewrap`） | 核心容器化工具：创建 namespace、配置 bind mount、把某些路径变只读、隔离网络视图 | `unshare(2)`/`clone(2)`、`mount(2)`、`execve(2)`、procfs、tmpfs |
| `socat` | 发行版包（`apt install socat` 等） | 宿主与沙箱之间的 Unix socket / TCP 代理桥 | `socket(2)`、`bind(2)`、`listen(2)`、`accept(2)`、`connect(2)` |
| `rg` / `ripgrep` | 发行版包 | 扫描危险路径，辅助生成 mount 规则；不是沙箱边界 | 普通文件系统接口（`open(2)`/`read(2)`/`stat(2)`、目录遍历） |
| `apply-seccomp` | sandbox-runtime 包内 vendor 自带的原生二进制（`vendor/seccomp/<arch>/apply-seccomp`） | 创建嵌套 user+PID+mount namespace、挂私有 `/proc`、安装 seccomp 过滤器、exec 用户命令 | `unshare(2)`、写 `uid_map`/`gid_map`、`mount(2)`、`prctl(PR_SET_NO_NEW_PRIVS)`、`prctl(PR_SET_SECCOMP, ...)`、`execve(2)` |

### 7.3 平台细节

**Ubuntu 24.04 有一个需要先处理的内核参数。** Ubuntu 24.04 默认开启 `kernel.apparmor_restrict_unprivileged_userns`：它允许 `unshare(CLONE_NEWUSER)`，但会从新 namespace 里剥掉 capabilities。bwrap 和 apply-seccomp 都需要"带 capabilities 的 user namespace"（bwrap 要靠它完成挂载，apply-seccomp 要靠它创建嵌套 namespace），所以在这个发行版上要先 `sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0`，或者给相关二进制加 AppArmor profile。这也说明：沙箱可用性不仅取决于"系统有没有 namespace 接口"，还取决于"namespace 里保留不保留特权"。

**Linux 路径配置只支持字面路径。** macOS 的 Seatbelt profile 支持 git 风格 glob（`**/*.ts` 这类模式）；Linux 的配置直接翻译成 bwrap 的 bind mount 参数，写什么路径就保护什么路径，没有把 glob 展开成多个挂载点的逻辑。

### 7.4 如果自己封装，工作量有多大

"调用接口"本身并不贵，贵的是接口之间的顺序约束、失败处理和语义映射。参考两个成熟实现的规模：`bubblewrap.c` 约 4000 行 C，`apply-seccomp.c` 约 1000 行 C，都是多年打磨。经验性的结论：熟悉 Linux 系统编程的人做**最小可用版**（userns + 只读根 + bind 开放可写路径 + exec）约 2-4 周、500-800 行 C；做到 **bwrap 级别**（覆盖所有边界情况、跨发行版、错误降级）是数月级。还有一类容易被低估的工作：**配置语义到挂载规则的翻译**——sandbox-runtime 自己的 `linux-sandbox-utils.ts` 有几千行 TypeScript，大部分在算"哪个 deny 路径要 `--ro-bind /dev/null`、哪个目录要 `--tmpfs` 覆盖、allow 路径被 deny 的 tmpfs 覆盖后要不要重新 bind 回来"。

### 7.5 一张表回顾：从需求到内核接口

| 需求（用户视角） | 二进制 | 底层接口 | 接口通俗解释 |
| --- | --- | --- | --- |
| 系统目录只读，只有 workspace 可写 | `bwrap` | `mount(2)` + `MS_BIND`/`MS_RDONLY` | 把宿主目录映射进沙箱，整棵系统树只读、工作目录单独开放 |
| 敏感路径不可读/写（`~/.ssh`、token） | `bwrap` | `mount(2)` + `MS_BIND`、`tmpfs` | 把 /dev/null（无底洞）盖到敏感文件上、在敏感目录上挂空黑板 |
| 沙箱 `/tmp` 是一次性草稿纸 | `bwrap` | `mount(2)` + `tmpfs` | 内存里的文件系统，卸载即消失 |
| 默认断网 | `bwrap` | `unshare(2)` + `CLONE_NEWNET` | 新网络世界，没有网卡、没有路由表，像刚装好的路由器还没插网线 |
| 只允许访问白名单域名 | `socat` + JS 代理 | `socket(2)`/`connect(2)` 等 | 沙箱里只留一扇"小门"，门卫（代理）查完域名才放行 |
| 不能用本地 Unix socket 绕过 | `apply-seccomp` | `prctl(2)` + `PR_SET_SECCOMP`、`seccomp(2)` + BPF | 每次系统调用先过安检，发现 AF_UNIX socket 直接拒绝 |
| 只能看到沙箱内进程 | `bwrap` + `apply-seccomp` | `unshare(2)` + `CLONE_NEWPID`、挂 procfs | 只有自己家的小区，外面的人和房子全看不见；/proc 像户口本，重新挂上配套户口本 `ps` 才显示得对 |
| 沙箱内无特权 | `bwrap` | `unshare(2)` + `CLONE_NEWUSER`、`capset(2)` | 新身份空间，但特权钥匙（capabilities）全部收走 |
| 退出不留后台进程 | `bwrap` | `setsid(2)`、`prctl(2)` + `PR_SET_PDEATHSIG` | 自成会话 + "老爸死了我就自杀"，bwrap 退出瞬间 SIGKILL 全家带走 |
| 危险文件永远禁写（`.bashrc`、`.git/hooks` 等） | `rg`（辅助） | `open(2)`/`read(2)`/`stat(2)`/目录遍历 | 翻文件夹找出危险文件，报告给 bwrap 去遮住，本身不是安全边界 |

## 8. 收尾：为什么说沙箱是组合拳

把五个维度串起来：

- namespace 改变进程看到的世界（文件挂载表、网络、进程树、身份）
- mount 策略控制文件系统可见性和可写性
- network namespace 切断默认外网，proxy 把允许的网络访问重新接回来
- seccomp 堵住 Unix socket 这类本地绕过
- capabilities / no_new_privs 收走特权、焊死提权路径
- CLI 权限层决定什么时候启用、什么时候提示、什么时候允许例外

这个分层也解释了为什么"沙箱能力"看起来像产品功能，实际依赖的是操作系统能力：产品代码负责把权限意图翻译成配置；真正执行拒绝和隔离的，还是 Linux 内核和围绕它的工具链。评估另一个系统能不能跑这套沙箱时，逐项确认 user namespace、mount namespace + bind mount、network namespace、PID namespace、seccomp 这些能力是否存在（或有没有等价物），比直接问"能不能装 bwrap"要准确得多。
