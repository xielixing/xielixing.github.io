+++
title = 'Agent 沙箱是什么：拆解 Claude Code 的 sandbox-runtime Linux 沙箱机制'
date = 2026-08-10T09:30:00+08:00
draft = false
summary = '梳理 Linux 自带沙箱能力的基础机制，包括 namespace、mount 隔离、capabilities、seccomp、cgroup、Landlock 等，以及它们如何组合成一个可用的命令执行沙箱；最后以 sandbox-runtime 为例拆解"用户感知特性 → 二进制 → 内核接口"的依赖链，并评估自研封装这些接口的工作量。'
tags = ['linux', 'sandbox', 'namespace', 'seccomp', 'landlock']
categories = ['Engineering']
+++

Agent 沙箱是什么？简单说，就是给 AI 代理（Agent）跑代码时准备的隔离环境。

现在的 AI 编码工具（比如 Claude Code）会自己决定执行什么命令、读写哪些文件、装什么依赖。这些动作是代理自主做的，如果直接跑在你的机器上，一次误触发的 `rm -rf`，或者一条被诱导执行的命令，就可能把整个系统搞坏。沙箱的作用，就是把这类自主行为限制在一个笼子里：文件系统、网络、系统调用都受到约束，出了问题也只坏在笼子内部，不会波及其它。

为什么值得关注？因为代理的工作方式和人是两回事。人敲命令之前会先想清楚，代理是"想好了就自己执行"，你拦不住它中途改主意。越是想放手让代理多干活，越得先把边界划清楚，沙箱就是这条边界。

这篇文章要讲的就是 Claude Code 的 Linux 沙箱是怎么实现的，对应的是 anthropic-experimental/sandbox-runtime 这个仓库（npm 包名 `@anthropic-ai/sandbox-runtime`）。前面第 0 到 10 节先把 Linux 自带的机制讲清楚——namespace、mount 隔离、capabilities、seccomp、cgroup、Landlock——第 11 节再拆开看 sandbox-runtime 是怎么把"用户配置的规则"翻译成"内核能理解的系统调用"的。

实验环境是 WSL2 里的 Ubuntu 24.04。WSL2 不等同于完整服务器发行版：它适合观察 namespace、user namespace、mount namespace、cgroup v2、Landlock 等基础机制，但 AppArmor/SELinux 这类 LSM 在这里未必启用，网络隔离行为也可能和标准 Linux 虚拟机不同。

## 0. 普通用户的身份基线

后面的实验默认在 WSL2 Ubuntu 里的普通用户下执行：

```bash
mkdir -p ~/sandbox-lab
cd ~/sandbox-lab
id
pwd
```

实际输出：

```text
uid=1000(sandboxer) gid=1000(sandboxer) groups=1000(sandboxer)
/home/sandboxer/sandbox-lab
```

这里有几个基础事实：

- `uid` 是 user id。Linux 内核真正识别的是数字 ID，不是用户名字符串。`uid=0` 是 root；`uid=1000` 是普通用户。
- `gid` 是当前进程的主用户组 ID。
- `groups` 是当前进程所属的所有用户组列表，通常包含主组，也可能包含附加组。

Linux 最基础的文件权限判断，就是拿当前进程携带的 `uid/gid/groups` 去和文件的 owner/group/others 权限匹配。可以把它想成文件门禁：

```text
owner   文件主人
group   文件授权给的团队
others  其他所有人
```

`owner` 解决“个人所有权”，`group` 解决“团队共享权限”。一个 group 里可以有多个用户；用户也可以属于多个 group。`id` 输出里的 `gid=...` 是主组，`groups=...` 是当前进程所属的完整组列表。内核不是先问“你叫什么名字”，而是看“这个进程带着哪些身份标记”。

至于为什么这里是 `1000`，不是 `1001`：在 Ubuntu/Debian 这类系统里，普通用户 UID 通常从 `1000` 开始分配。这个 WSL 发行版之前没有普通用户，所以我们创建的 `sandboxer` 成了第一个普通用户，自然拿到了 `1000`。如果系统里已经有一个普通用户占用了 `1000`，下一个用户通常才会拿到 `1001`。

后面的命令默认都在这个身份和目录下执行：

```text
用户：sandboxer
目录：/home/sandboxer/sandbox-lab
提示符：sandboxer@DESKTOP-ABLKNN8C:~/sandbox-lab$
```

下一步要观察的第一件事，是普通用户为什么不能读取 root 的私有目录。这会把“uid/gid 决定基础权限”的直觉先钉住，然后再进入 user namespace。

## 1. 普通用户访问 root 私有目录

先看 `/root` 目录本身：

```bash
ls -ld /root
```

输出类似：

```text
drwx------ 10 root root 4096 Aug 9 11:11 /root
```

这里 `ls -ld /root` 的含义是：用详细格式查看 `/root` 这个目录本身，而不是展开它里面的内容。`-l` 是 long format，显示权限、owner、group、大小、时间等信息；`-d` 是 directory itself，只看目录本身。

权限字段可以拆成：

```text
d   rwx   ---   ---
    owner group others
```

后面的 `root root` 分别表示 owner 是 `root` 用户，group 是 `root` 用户组。所以这行的意思是：`/root` 是一个目录，只有 owner `root` 有读、写、进入权限；group 和 others 都没有权限。

再尝试展开目录内容：

```bash
ls /root
```

实际结果：

```text
ls: cannot open directory '/root': Permission denied
```

这一步失败不是 `ls` 程序自己决定的，而是内核根据当前进程的身份和 `/root` 的权限拒绝了它。当前进程是 `uid=1000(sandboxer)`，不是 owner `root`，也没有命中可用的 group 权限，最后只能落到 others；但 `/root` 的 others 权限是 `---`。

目录上的 `rwx` 和普通文件略有不同：

```text
r  可以列出目录里的文件名
w  可以在目录里创建、删除、改名文件
x  可以进入或穿过这个目录，访问里面的路径
```

所以可以把这一步理解成：

```text
ls -ld /root   看门牌，可以
ls /root       进门看房间，不行
```

## 2. User namespace 里的 root

接下来创建一个新的 user namespace：

```bash
unshare -Ur sh
```

`-U` 表示创建 user namespace，`-r` 表示把当前用户映射成 namespace 里的 root。进入新 shell 后执行：

```bash
echo "== inside user namespace =="
id
cat /proc/self/uid_map
cat /proc/self/gid_map
```

实际输出：

```text
== inside user namespace ==
uid=0(root) gid=0(root) groups=0(root)
         0       1000          1
         0       1000          1
```

这两张映射表的意思是：

```text
namespace 里的 uid 0  <->  外面宿主的 uid 1000
namespace 里的 gid 0  <->  外面宿主的 gid 1000
只映射 1 个 ID
```

也就是说，在这个小世界里，当前进程看起来是 root；但在外面的宿主系统里，它仍然只是 `sandboxer`。

再看 `/root`：

```bash
ls -ld /root
ls /root
```

实际输出：

```text
drwx------ 10 nobody nogroup 4096 Aug 9 11:11 /root
ls: cannot open directory '/root': Permission denied
```

这里最有意思的是 owner/group 从外面的 `root root` 变成了里面的 `nobody nogroup`。这不是 `/root` 真的被改了归属，而是当前 user namespace 里没有外部 `uid=0/gid=0` 的映射。namespace 只认识外部 `1000`，并把它显示成内部 `0(root)`；外部真正的 root 无法在这个 namespace 里表示，于是显示成 `nobody/nogroup`。

所以这三件事并不矛盾：

```text
id 显示 uid=0(root)
/root 显示 nobody nogroup
ls /root 仍然 Permission denied
```

一句话直觉：你是“这个小世界里的 root”，不是“外面真实系统的 root”。`unshare -Ur` 不是让普通用户攻破系统，而是创建了一个新的身份空间，把普通用户映射成里面的 root，同时不授予外部 root 的权力。

顺便拆一下 `ls -ld /root` 的完整输出：

```text
drwx------  10  nobody  nogroup  4096  Aug 9 11:11  /root
权限        链接数 owner   group    大小   修改时间       路径
```

其中 `10` 是 link count。对目录来说，可以粗略理解为 `2 + 直接子目录数量`，不是权限。`4096` 是目录本身占用的字节数；目录也是一种特殊文件，里面保存“文件名到 inode”的映射，4096 通常是一个文件系统块大小。当前实验里，真正影响访问判断的是权限和 owner/group。

## 3. Mount namespace 和临时挂载

如果 user namespace 改变的是“进程怎么看用户身份”，mount namespace 改变的就是“进程怎么看文件系统挂载表”。

Linux 的目录树不是单一硬盘目录。很多不同来源的文件系统可以被挂到同一棵目录树上，比如 `/proc` 通常是 procfs，`/tmp` 下面也可以挂一个临时的 tmpfs。`mount` 可以理解成把一个文件系统接到某个目录点上；`findmnt` 用来查看当前进程能看到的挂载关系。

先在外面确认没有挂载：

```bash
findmnt /tmp/sbox-demo || echo "outside: no mount yet"
ls -ld /tmp/sbox-demo 2>/dev/null || echo "outside: no directory yet"
```

实际输出：

```text
outside: no mount yet
outside: no directory yet
```

然后创建新的 user namespace 和 mount namespace：

```bash
unshare -Ur -m sh
```

这里 `-m` 表示创建 mount namespace。进入里面后执行：

```bash
mkdir -p /tmp/sbox-demo
mount -t tmpfs tmpfs /tmp/sbox-demo
touch /tmp/sbox-demo/inside.txt
findmnt /tmp/sbox-demo
ls -la /tmp/sbox-demo
```

实际输出：

```text
TARGET          SOURCE FSTYPE OPTIONS
/tmp/sbox-demo tmpfs  tmpfs  rw,relatime,uid=1000,gid=1000

total 4
drwxrwxrwt 2 root   root      60 Aug 9 13:18 .
drwxrwxrwt 9 nobody nogroup 4096 Aug 9 13:18 ..
-rw-rw-r-- 1 root   root       0 Aug 9 13:18 inside.txt
```

这说明在当前 namespace 的挂载表里，`/tmp/sbox-demo` 已经变成一个 tmpfs 挂载点。`inside.txt` 写进的是这个临时文件系统，而不是外面原来的 `/tmp` 目录。

退出 namespace 后再看：

```bash
exit
findmnt /tmp/sbox-demo || echo "outside: no mount"
ls -la /tmp/sbox-demo 2>/dev/null || echo "outside: no directory"
```

实际输出：

```text
outside: no mount

total 8
drwxr-xr-x 2 sandboxer sandboxer 4096 Aug 9 13:18 .
drwxrwxrwt 9 root      root      4096 Aug 9 13:33 ..
```

这里有一个很好的细节：`/tmp/sbox-demo` 这个目录壳还在，因为 `mkdir -p /tmp/sbox-demo` 创建的是底层 `/tmp` 里的普通目录；但 `inside.txt` 不见了，因为它是在 namespace 里面挂上的 tmpfs 里创建的。退出后，外面的挂载表里没有这个 tmpfs，自然也看不到 tmpfs 里的文件。

所以 mount namespace 的直觉是：

```text
不同进程可以拥有不同的文件系统挂载视图。
```

沙箱会大量利用这一点：给进程一个假的 `/tmp`，把系统目录挂成只读，只挂入允许访问的 workspace，不把真实的 `~/.ssh`、token、配置目录暴露进去。它不只是“改权限”，而是先改变进程能看见哪一棵文件系统树。

### 3.1 挂载问答：从“路径”到 mount

这里用一问一答的方式，把上面几句话背后的概念链条拆开，方便建立直觉。

**Q1：输入 `ls /home/sandboxer/foo.txt` 时，路径让计算机做什么？**

路径就是给内核指路：从 `/` 开始一级一级“走”，每进一个目录就查下一级。关键点：目录本身也是一种文件，但它存的不是内容，而是“名字 → 数据位置”的指针（指向数据在存储里的实际位置）。所以目录树 = 一层层的“名字→指针”表，内容都在别处。

**Q2：数据一定要在硬盘上吗？**

不一定。内存（tmpfs，写在 `/tmp` 的东西可以只存在内存里）、内核虚拟（`/proc`、`/sys`——读的那一刻内核现场生成）、网络共享盘（NFS）都可以当数据源。每个数据源自带一套“目录+指针”结构，这就是“文件系统”的真正含义：一套放在某个来源里的目录树。

**Q3：这么多独立的数据树，怎么合到同一棵 `/` 树里？**

靠 mount（挂载）：把一个文件系统的“根”，接到目录树的一个目录点（挂载点）上。挂上之后，从那个点往下走的路径就进入该数据源的内容；`umount` 就是拔掉。Linux 只有一棵 `/` 树，开机时 `mount /` 挂上根文件系统，之后每个数据源都是这样“接”进去的：`/tmp` 是内存盘接的，`/proc` 是内核虚拟树接的。

**Q4：内核虚拟树就是 namespace 吗？**

不是，两个层面。虚拟文件系统（`/proc` 等）说的是“数据源不在磁盘上、读的时候内核现编”，本质还是文件系统；namespace 说的是“进程的视角隔离”（PID namespace 决定 `/proc` 里列出哪些进程、mount namespace 决定挂载表、network namespace 决定网卡）。关系可以这样记：虚拟文件系统是公告栏（内容实时画），namespace 是房间——不同房间的人看同一块公告栏，但内容按房间过滤。

**Q5：`/proc/cpuinfo` 在两个不同的 PID namespace 里读，内容一样吗？**

一样。CPU 是全局信息，内核实时计算的结果不受 namespace 影响。而 `/proc` 里的进程列表会按 PID namespace 过滤。这正好解释了沙箱为什么要重新挂载 `/proc`：防的不是 `cpuinfo`，而是进程列表和 `/proc/<pid>/mem` 这类暴露。

**Q6：重新挂载是什么意思？**

在同一个路径点上把数据源换掉。创建 mount namespace 时挂载表是继承的（`/proc` 还接着宿主的 procfs）；进入新 PID namespace 后内容不匹配，就在 `/proc` 这个点上再挂一个新的 procfs 实例（`mount("proc", "/proc", "proc", 0, NULL)`），新挂载盖住旧的。类比：同一块公告栏，撕掉旧名单贴上新名单。bwrap 的 `--proc /proc` 和 apply-seccomp 嵌套时重挂私有 `/proc` 都是这个操作。

**Q7：挂载时能把想给的信息算好传进去吗（比如只给 `cpuinfo` 不给别的）？**

不能也不需要。procfs 的内容是“读的那一刻”内核现编的，挂载只声明“这个点接的是 procfs”，选不了内部文件。想“只给某个文件/不给某个文件”要用两种手段：① 让内核按身份换内容（PID namespace 过滤进程列表、`hidepid` 挂载选项隐藏他人进程）；② 挂完之后用 bind mount 单点遮盖，比如 `--ro-bind /tmp/fake-cpuinfo /proc/cpuinfo` 把假文件盖上去，或 `--ro-bind /dev/null /proc/loadavg` 用“无底洞”盖掉不想给的。sandbox-runtime 的 `maskedFileBinds` 就是这个思路：把真实凭据文件换成“哨兵假文件”，让沙箱里读到一份无害版本。

一句话收尾：mount 只负责“哪个数据源接在哪个路标上”，不负责挑内容；内容由数据源按进程身份实时决定，沙箱再用 bind mount 逐点遮盖来微调。

## 4. 外层网络视图：网卡、网段、网关和路由

进入 network namespace 之前，先看外层正常 WSL 环境怎么访问网络：

```bash
echo "== outside network =="
ip addr
ip route
```

输出里有两个主要网络接口：

```text
1: lo
2: eth0
```

可以把网络理解成寄快递：

```text
网卡     = 出入口
IP 地址  = 这个出入口的地址
网段     = 附近这一片地址范围
网关     = 去外地的转运站
路由表   = 遇到不同目的地时该从哪条路走
```

`lo` 是 loopback，本机回环接口：

```text
inet 127.0.0.1/8
inet6 ::1/128
```

它用于本机访问自己。比如一个服务监听 `127.0.0.1:8000`，本机进程访问它时走的就是 `lo`，不需要出外网。

`eth0` 是 WSL 的主要网络接口：

```text
inet 172.22.181.127/20 brd 172.22.191.255 scope global eth0
```

这里 `172.22.181.127` 是 WSL 这张虚拟网卡的 IPv4 地址。`/20` 表示它所在的本地网段，大致是 `172.22.176.0` 到 `172.22.191.255`。同一个网段可以理解成同一个小区里的地址；不在这个范围里的目标，就要交给网关转发。

路由表里最关键的是：

```text
default via 172.22.176.1 dev eth0
172.22.176.0/20 dev eth0 proto kernel scope link src 172.22.181.127
```

第一行表示：不知道怎么走的目标，都交给默认网关 `172.22.176.1`，从 `eth0` 发出去。第二行表示：访问 `172.22.176.0/20` 这个本地网段时，直接从 `eth0` 走，源地址用 `172.22.181.127`。

所以外层 WSL 的网络路径大概是：

```text
访问 127.0.0.1
WSL 进程 -> lo -> 本机服务

访问外部地址
WSL 进程 -> eth0(172.22.181.127) -> gateway(172.22.176.1) -> 外部网络
```

这里的 `eth0` 是虚拟网卡，不是物理网卡。Windows 是宿主系统，WSL2 像一台轻量虚拟机，Windows/WSL 网络层给它提供了一个虚拟网络出口。

如果是在一台真正的 Linux 机器上，拓扑可能不一样：`eth0` 可能对应真实有线网卡，Wi-Fi 可能叫 `wlan0`，服务器上可能有多张物理网卡、bond、bridge、veth、Docker 网桥等。容器里也经常看到虚拟网卡。名字和拓扑会变，但基本概念不变：进程通过网络接口、IP、路由表和网关访问网络。

network namespace 要隔离的，正是这一整套网络视图：进程能看到哪些网卡、哪些 IP、哪些路由、有没有默认网关。一个沙箱如果不给进程 `eth0` 和 default route，它就算调用网络程序，也没有外网出口。

## 5. Network namespace 里的空网络世界

创建新的 network namespace：

```bash
unshare -Urn sh
```

这里 `-n` 表示创建 network namespace。进入里面后执行：

```bash
id
ip -br addr
ip route
```

实际输出：

```text
uid=0(root) gid=0(root) groups=0(root)
lo               DOWN
```

`ip route` 没有任何输出。和外层网络相比，这个 namespace 里没有 `eth0`，没有外部 IP，也没有 default route；只有一个默认关闭的 `lo`。这说明 network namespace 隔离的是整套网络视图：网卡列表、IP 地址、路由表、默认网关。

先把 loopback 启起来：

```bash
ip link set lo up
ip -br addr
ip route
```

实际输出：

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
```

`ip route` 仍然没有输出。此时再测试：

```bash
ping -c 1 127.0.0.1
ping -c 1 8.8.8.8
```

实际结果：

```text
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=1.46 ms

--- 127.0.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms

ping: connect: Network is unreachable
```

这说明里面的“自己访问自己”已经可用，但外网仍然不可达。原因不是 DNS、代理或 ping 程序的问题，而是当前 network namespace 里根本没有外网出口：没有 `eth0`，也没有默认网关。

所以 network namespace 的直觉是：

```text
可以给进程一个独立的网络世界。
这个世界里可以只有 lo，没有 eth0，没有 default route。
```

这就是沙箱断网的基础方式之一。它比“告诉程序不要联网”更硬，因为内核给这个进程的网络视图里压根没有通向外网的路。

### 5.1 网络问答：从“出门”到“断网”

**Q1：数据从哪里“出门”？**

从网卡（network interface）。网卡是网络的出入口，每种连接方式是一块网卡：有线网卡（`eth0`）、无线网卡（`wlan0`），还有一块虚拟网卡 `lo`——它不连任何外部设备，专门用来“自己和自己说话”（本机进程访问本机服务就走它）。

**Q2：出门之后怎么找到对方？**

靠 IP 地址。IP 是每台联网设备的“门牌号”，负责找到“哪台电脑”。（数据包到达后，操作系统再看端口号——端口是“房间号”，决定把数据交给哪个程序：网页通常用 443/80，SSH 用 22，数据库用 3306。IP 找楼，端口找房间。）

**Q3：怎么知道对方是“邻居”还是“远门”？**

靠子网掩码把 IP 拆成“小区号 + 房间号”：前几段是小区号（网段），后几段是房间号。目标在同网段 = 邻居，直接送；不同网段 = 要出远门，交给别人转送。

**Q4：出远门的数据交给谁？**

交给网关（gateway，通常就是家里那台路由器）。网关 = 转运站：所有要出远门的数据先送到它那里，它自己手里也有一张“路怎么走”的表（路由表），一层层转送，最终送到目标所在网段。

**Q5：电脑怎么知道“出远门就交给网关”？**

靠路由表。电脑里存着一张表：遇到不同目标走哪条路。例如：

```text
目标是 172.22.176.0/20（小区内）→ 直接从 eth0 送
其他一切目标（外网）        → 交给网关 172.22.176.1
```

最后那行“其他一切目标都交给网关”的兜底规则叫**默认路由**（default route，Linux 里写成 `default via <网关IP> dev <网卡>`）。补充一个容易混的点：静态路由/动态路由是另一个维度——静态是手动配置的，动态是协议自动学习的；默认路由是“表里的那行兜底”，别把这两个维度混在一起。

**Q6：为什么“断网”的沙箱真的断网？**

`--unshare-net` 创建的空 network namespace 里，网络链路需要的每一样东西都缺：

```text
1. 没有网卡     → 没有出入口，数据根本出不了门
2. 没有路由表   → 就算有网卡，也不知道往哪送
3. 没有默认网关 → 就算知道要出门，也没有转运站接手
```

唯一剩下的 `lo` 只能“自己和自己说话”（启起来后 `ping 127.0.0.1` 能通），对外一个字节都发不出去。所以断网是结构性的：不是程序被禁止联网，而是这个世界里根本没有路——程序再多花招也没用。

### 5.2 代理问答：受控联网是怎么“接回来”的

**Q7：代理是什么？**

代理（proxy）= 中间人。程序不直接联系目标服务器，而是把请求交给代理，代理替它联系目标、拿到结果再带回来。类比代购：你不直接去日本，把需求交给代购，代购替你跑一趟。代购的关键性质是**他能检查你想买什么**——沙箱就靠它做域名白名单：请求到达代理，代理比对 `allowedDomains`/`deniedDomains`，放行或拒绝（拒绝时返回类似 `Connection blocked by network allowlist` 的提示）。

程序怎么找到代理？靠环境变量：沙箱注入 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`，程序一读就知道“去 `127.0.0.1:3128` 找代购”。（局限也在这里：不认这些环境变量的程序会直接连不上网络——失败即拒绝，而不是绕过。）

**Q8：HTTP 代理和 SOCKS5 代理是什么关系？**

都是代理，区别在于“懂多少”：

| | HTTP 代理 | SOCKS5 代理 |
| --- | --- | --- |
| 懂什么 | 只懂 HTTP/HTTPS 协议，能看懂域名、路径、甚至内容 | 不懂任何应用层协议，纯搬运字节流 |
| 能干什么 | 域名过滤、内容检查、缓存、认证 | 任何 TCP 流量都能转：SSH、git、数据库、游戏 |
| 类比 | 懂日语的代购，能看说明书帮你挑 | 纯快递公司，不管箱子里是啥 |

沙箱两个都起：网页/API/npm 这类 HTTP 流量走 HTTP 代理；`git` 走 SSH、数据库连接这类非 HTTP 流量交给 SOCKS5 代理。细节：HTTPS 走 HTTP 代理时其实是 CONNECT 隧道——代理也只“搬箱子”不拆箱，只有开启 TLS 终止（`tlsTerminate`，代理伪装成目标服务器解密流量）后才能真正看见内容。

**Q9：沙箱里的 JS 代理是 sandbox-runtime 自己的代码吗？**

是。它在 sandbox-runtime 仓库的 `src/sandbox/http-proxy.ts` 和 `src/sandbox/socks-proxy.ts`，跑在宿主机上。整个网络链路是三个角色分工：

```text
bwrap（原生 C） 负责“断”：--unshare-net 清空沙箱的网络世界
socat（原生 C） 负责“桥”：Unix socket 和端口之间搬数据（纯搬运，不过问内容）
JS 代理（TS）  负责“查”：域名白名单逻辑全在这里，是 sandbox-runtime 自己写的业务部分
```

沙箱内的传输链路是：

```text
沙箱内程序 → 沙箱内 127.0.0.1:3128/1080（socat 监听）
→ 转成 Unix socket 请求（这个 socket 被 bind mount 进沙箱）
→ 宿主侧 socat 收到 → 转给真正的 JS 代理 → 代理查白名单 → 放行则访问外网
```

**Q10：代理的 IP 是物理机实际的 IP 吗？**

不是。`127.0.0.1` 叫回环地址，含义是“我自己”，不是任何物理机的地址。更妙的是每个 network namespace 都有自己的“自己”（那根 `lo`），所以沙箱里的 `127.0.0.1:3128` 是沙箱自己 lo 上的 socat，宿主代理监听的是宿主自己 lo 上的端口，两边是“各自房间里的自己”。

代理和沙箱物理上是同一台机器，它们之间靠 bind 进沙箱的 Unix socket 通信——Unix socket 走的是文件系统，根本不经过物理网络，所以代理不需要物理 IP。只有代理决定放行、真正替沙箱访问外网时，才从宿主 `eth0` 的真实 IP 出去：

```text
127.0.0.1 = “我家的门”（沙箱 ↔ 宿主内部传递，不出物理网络）
eth0 的真实 IP = “我家的对外地址”（代理真正访问外网时才用）
```

**Q11：代理和 IP 是什么关系？**

IP 是地址系统，代理是角色。代理也是一种网络服务，所以它自己也有一个 IP:端口——本机代理就是 `127.0.0.1:3128`，公司里的远程代理就是一台远程机器的 IP:端口。代理的本质是**插在连接链路中间**：

```text
没有代理：程序 ────────────> 目标服务器（连接目标 IP）
有代理：  程序 ──> 代理 ──────> 目标服务器
          （连接代理的 IP:端口）  （代理替你连接目标 IP）
```

程序甚至不需要知道目标 IP——它只告诉代理“我要去 example.com”，解析和连接都由代理代办（所以叫“代”理）。沙箱里这层意义重大：目标地址的解析和连接全握在代理手里，程序至始至终接触不到目标的真实 IP，这就是代理能“把关”的原因。

## 6. PID namespace 和新的进程树

PID namespace 隔离的是进程编号和进程树。先在外层看当前 shell：

```bash
echo "== outside pid baseline =="
echo "shell pid: $$"
ps -o pid,ppid,comm | head
```

实际输出：

```text
shell pid: 1786

  PID  PPID COMMAND
 1786  1760 bash
27314  1786 ps
27315  1786 head
```

这里当前 shell 的 PID 是 `1786`，它的父进程是 `1760`。`ps` 和 `head` 是刚刚为了显示进程列表临时启动的子进程。

创建新的 PID namespace：

```bash
unshare -Urpf sh
```

参数含义：

```text
-U  新 user namespace
-r  当前用户映射成里面的 root
-p  新 PID namespace
-f  fork 一个子进程进入新 PID namespace
```

进入后执行：

```bash
echo "shell pid: $$"
ps -o pid,ppid,comm
```

实际看到：

```text
shell pid: 1

  PID  PPID COMMAND
```

`shell pid: 1` 说明当前 `sh` 在这个 PID namespace 里是 1 号进程，也就是这个小世界里的第一个进程。`ps` 只有表头，是因为 PID namespace 已经换了，但 `/proc` 还没有换成匹配这个 PID namespace 的 procfs。`ps` 主要靠读取 `/proc` 来列进程；如果 `/proc` 视图不配套，看到的结果就会不完整。

退出后再用 `--mount-proc`：

```bash
unshare -Urpf --mount-proc sh
```

进入后执行：

```bash
echo "shell pid: $$"
ps -o pid,ppid,comm
ls -ld /proc/1
readlink /proc/1/exe
```

实际输出：

```text
shell pid: 1

  PID  PPID COMMAND
    1     0 sh
    2     1 ps

dr-xr-xr-x 9 root root 0 Aug 9 14:57 /proc/1
/usr/bin/dash
```

这次 `ps` 可以正确看到 namespace 里的进程树：`sh` 是 PID 1，`ps` 是它启动的子进程。`/proc/1/exe` 指向 `/usr/bin/dash`，说明这里的 `sh` 实际由 dash 提供。

PID namespace 的直觉是：

```text
同一个进程，在外面有外面的 PID；
在新的 PID namespace 里，又有里面的 PID。
```

沙箱和容器会用它让进程只能看到自己这棵小进程树，而不是宿主机上的所有进程。配套挂载 `/proc` 很重要，因为很多进程工具看到的世界来自 `/proc`。

## 7. Capabilities：root 权限被拆开了

Linux 里的 root 权限不是一个不可分割的整体，而是拆成了一组 capabilities。比如挂载文件系统通常需要 `CAP_SYS_ADMIN`，修改网络配置通常需要 `CAP_NET_ADMIN`，调试别的进程可能需要 `CAP_SYS_PTRACE`。

先看普通用户当前进程的 capability：

```bash
grep -E 'CapInh|CapPrm|CapEff|CapBnd|CapAmb|NoNewPrivs' /proc/self/status
```

实际输出：

```text
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 000001ffffffffff
CapAmb: 0000000000000000
NoNewPrivs: 0
```

这里先看 `CapEff`，它表示当前真正生效的 capabilities。普通用户的 `CapEff` 是 0，所以直接挂载 tmpfs 会失败：

```bash
mkdir -p /tmp/cap-demo
mount -t tmpfs tmpfs /tmp/cap-demo
```

实际输出：

```text
mount: /tmp/cap-demo: must be superuser to use mount.
```

进入 user namespace 后再看：

```bash
unshare -Ur sh
id
grep -E 'CapInh|CapPrm|CapEff|CapBnd|CapAmb|NoNewPrivs' /proc/self/status
```

实际输出：

```text
uid=0(root) gid=0(root) groups=0(root)
CapInh: 0000000000000000
CapPrm: 000001ffffffffff
CapEff: 000001ffffffffff
CapBnd: 000001ffffffffff
CapAmb: 0000000000000000
NoNewPrivs: 0
```

这里 `CapEff` 变成了非 0，说明在这个 user namespace 里确实拥有很多 capability。但如果只创建 user namespace，不创建新的 mount namespace，再执行：

```bash
mount -t tmpfs tmpfs /tmp/cap-demo
```

实际仍然失败：

```text
mount: /tmp/cap-demo: permission denied.
```

这说明 capability 不是“无限 root 权力”。它有 namespace 边界：里面的 `CAP_SYS_ADMIN` 不能直接拿去修改外层真实系统的挂载视图。

把 user namespace 和 mount namespace 配起来：

```bash
unshare -Ur -m sh
id
grep -E 'CapInh|CapPrm|CapEff|CapBnd|CapAmb|NoNewPrivs' /proc/self/status
mkdir -p /tmp/cap-demo
mount -t tmpfs tmpfs /tmp/cap-demo
findmnt /tmp/cap-demo
umount /tmp/cap-demo
```

实际输出：

```text
uid=0(root) gid=0(root) groups=0(root)
CapEff: 000001ffffffffff

TARGET        SOURCE FSTYPE OPTIONS
/tmp/cap-demo tmpfs  tmpfs  rw,relatime,uid=1000,gid=1000
```

这次挂载成功了，因为进程既拥有当前 user namespace 里的 capability，也拥有一个新的 mount namespace 可以被它修改。

所以这一步的直觉是：

```text
普通用户：没有有效 capability，mount 失败。
user namespace root：有里面的 capability，但不能随便改外层资源。
user + mount namespace：有里面的 CAP_SYS_ADMIN，也有自己的挂载视图，mount 成功。
```

沙箱会利用这个组合：在小世界里给进程一些“看起来像 root 才能做”的能力，同时把这些能力限制在 namespace 边界内，不让它变成宿主系统的真实 root。

## 8. no_new_privs：防提权保险丝

`no_new_privs` 是一个单向开关。打开后，当前进程和它的子进程不能再通过 `execve` 获得新的权限。它常和 seccomp 一起使用，是沙箱里很重要的防提权保险丝。

先看外层普通 shell：

```bash
grep -E 'NoNewPrivs|Seccomp|CapEff' /proc/self/status
```

实际输出：

```text
CapEff: 0000000000000000
NoNewPrivs: 0
Seccomp: 0
Seccomp_filters: 0
```

这里 `NoNewPrivs: 0` 表示开关还没打开，`Seccomp: 0` 表示当前也没有 seccomp 过滤器。

用 `setpriv` 打开它，并进入一个新的子 shell：

```bash
setpriv --no-new-privs sh
```

在子 shell 里看：

```bash
grep -E 'NoNewPrivs|Seccomp|CapEff' /proc/self/status
```

实际输出：

```text
CapEff: 0000000000000000
NoNewPrivs: 1
Seccomp: 0
Seccomp_filters: 0
```

再启动一个子进程：

```bash
sh -c 'echo "child:"; grep -E "NoNewPrivs|Seccomp|CapEff" /proc/self/status'
```

实际输出：

```text
child:
CapEff: 0000000000000000
NoNewPrivs: 1
Seccomp: 0
Seccomp_filters: 0
```

这说明 `NoNewPrivs` 会继承给子进程。它一旦在某条进程链上打开，就不能被普通程序改回 0。

退出这个子 shell 后再看外层：

```bash
exit
grep -E 'NoNewPrivs|Seccomp|CapEff' /proc/self/status
```

实际输出又回到：

```text
CapEff: 0000000000000000
NoNewPrivs: 0
Seccomp: 0
Seccomp_filters: 0
```

这不是因为 `no_new_privs` 被关闭了，而是因为刚才打开它的是子 shell。退出子 shell 后，我们回到了原来的外层 shell；这个外层 shell 从未打开过该开关。

直觉上可以这样理解：

```text
外层 shell: NoNewPrivs=0
  └─ setpriv 创建的子 shell: NoNewPrivs=1
       └─ 子进程: 继续继承 NoNewPrivs=1

退出子 shell 后，回到外层 shell: NoNewPrivs=0
```

沙箱会在启动不可信命令前打开它。这样即使子进程执行了 setuid 程序或带文件 capability 的程序，也不能借机获得比当前更多的权限。

## 9. Seccomp：限制系统调用

前面的机制更多是在回答“进程处在哪个世界里”：

```text
user namespace    身份视图
mount namespace   文件系统挂载视图
network namespace 网络视图
PID namespace     进程树视图
capabilities      在 namespace 边界内拥有哪些特权
no_new_privs      子进程不能再获得新权限
```

seccomp 关注的是另一件事：这个进程还能向内核发起哪些 syscall。

普通程序并不是直接操作硬件或文件系统。它要读文件、开 socket、创建进程、挂载文件系统，本质上都要通过 syscall 进入内核。例如：

```text
openat   打开文件
read     读取文件
write    写文件或终端
socket   创建网络 socket
connect  发起网络连接
clone    创建线程或进程
mount    挂载文件系统
ptrace   调试/控制其他进程
```

seccomp 就是在 syscall 入口处加过滤器。过滤器可以决定：

```text
允许这个 syscall
返回 EPERM / ENOSYS 之类的错误
直接杀死进程
记录日志或通知 supervisor
```

当前 shell 的状态可以这样看：

```bash
grep -E 'NoNewPrivs|Seccomp|Seccomp_filters' /proc/self/status
```

实验环境里的基线是：

```text
NoNewPrivs: 0
Seccomp: 0
Seccomp_filters: 0
```

这只表示当前 shell 没有安装 seccomp 过滤器，不表示内核不支持 seccomp。

一个典型的最小实验是：安装一个 seccomp 过滤器，专门拒绝 `getpid` syscall。安装前：

```text
getpid() -> 返回当前进程 PID
```

安装后：

```text
getpid() -> -1, errno = EPERM
```

这个实验的价值在于它足够安全：`getpid` 不会改文件、不联网、不杀进程；但它能证明 seccomp 的控制点不在文件权限、不在 namespace，而在 syscall 入口。

更接近真实沙箱的规则通常不会禁 `getpid`，而是限制危险 syscall，例如：

```text
mount      禁止进程重新挂载文件系统
ptrace     禁止调试/注入其他进程
bpf        禁止加载 eBPF 程序
keyctl     禁止访问内核 keyring
reboot     禁止重启系统
setns      禁止加入其他 namespace
unshare    禁止再创建新的 namespace
socket     禁止创建网络 socket
connect    禁止主动连接外部地址
```

seccomp 和 `no_new_privs` 经常一起出现。原因是：如果一个普通进程想在不提权的前提下安装 seccomp 过滤器，通常需要先打开 `no_new_privs`。这样内核知道这个进程链之后不会通过 setuid 或文件 capability 获得新权限，安装 syscall 限制才不会被绕成提权路径。

所以 seccomp 的直觉是：

```text
namespace 决定进程看到哪个世界；
capabilities 决定进程在这个世界里有什么特权；
seccomp 决定进程还能调用哪些内核入口。
```

如果前面的 network namespace 是“这个世界里没有外网网卡”，那么 seccomp 禁 `socket/connect` 更像是“即使你看得到网络相关对象，也不允许你调用创建连接的内核入口”。真实沙箱通常会组合使用这些机制，而不是只依赖某一个。

## 10. 后续还需要补齐的沙箱能力

到这里已经看过了几块最核心的隔离和权限机制：

```text
身份视图      user namespace
文件系统视图  mount namespace
网络视图      network namespace
进程树视图    PID namespace
特权拆分      capabilities
防提权开关    no_new_privs
系统调用过滤  seccomp
```

但一个可用的命令执行沙箱通常还会继续补几类能力：

| 能力 | 解决的问题 | 后续实验直觉 |
| --- | --- | --- |
| rlimit / `ulimit` | 限制单进程或当前 shell 派生进程的资源 | 把 `open files` 调小，然后打开很多文件，看到 `Too many open files` |
| cgroup v2 | 限制一组进程的 CPU、内存、进程数、IO | 把进程放进自己的 cgroup，观察 `memory.max`、`pids.max` 这类硬限制 |
| `chroot` / `pivot_root` | 改变进程看到的根目录 | 让进程以一个临时目录为 `/`，理解为什么单独 `chroot` 不等于强沙箱 |
| bind mount / readonly mount | 精确决定哪些目录可见、哪些只读 | 只把 workspace 挂进去，把系统目录挂成只读 |
| tmpfs | 提供临时、退出即丢的可写空间 | 给沙箱一个假的 `/tmp` 或临时 home |
| Landlock | 普通进程主动给自己加文件访问限制 | 不靠 root，让进程声明“只能读写这些路径” |
| LSM: AppArmor / SELinux | 系统级强制访问控制 | 在支持的发行版上，用 profile/label 限制文件、网络和能力 |
| 超时和进程回收 | 防止命令无限运行或遗留后台进程 | 给命令加 timeout，结束时清理整个进程组/cgroup |
| 环境变量和 secret 隔离 | 防止 token、SSH key、配置泄漏 | 启动命令前清理环境变量，不挂载敏感目录 |
| 组合策略 | 把单点机制变成真正沙箱 | namespace 改视图，mount 控文件，cgroup 控资源，seccomp 控 syscall，no_new_privs 防提权 |

接下来最适合继续手动实验的是：

```text
1. rlimit / ulimit
2. cgroup v2
3. chroot 和为什么它不够
4. bind mount + readonly mount
5. Landlock
```

这几项都能在 WSL2 里建立比较直观的认识。AppArmor/SELinux 更依赖发行版和内核配置，在当前 WSL 环境里不一定适合做第一轮手动实验。

## 11. Claude Code 沙箱运行时的 Linux 实现

看完这些内核机制之后，再看 [anthropic-experimental/sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime) 的实现，就能更容易分清“产品能力”和“系统能力”的边界。这个仓库发布的是 `@anthropic-ai/sandbox-runtime`，README 里说明它是为 Claude Code 沙箱能力开发的 research preview。

这个运行时没有从零发明沙箱机制，而是把 Linux/macOS 的系统能力包装成一个可复用的 CLI/库。接入方通常主要做三件事：

```text
1. 把用户设置和权限规则转换成 sandbox runtime 配置
2. 决定哪些 Bash 命令需要进沙箱
3. 在执行命令前，让 sandbox runtime 把原始命令包装成 sandboxed command
```

整体链路大概是：

```text
Bash 工具调用
  -> 判断是否启用沙箱
  -> 构造文件/网络/临时目录配置
  -> sandbox runtime 包装命令
  -> Linux: bubblewrap + proxy + seccomp
  -> 执行真正的 bash command
```

它对外暴露的沙箱能力可以分成几类：

| Claude Code 沙箱运行时能力 | 用户看到的效果 | 底层 Linux 能力 | 实际使用的程序/参数或内核接口 |
| --- | --- | --- | --- |
| Bash 命令沙箱 | Bash 命令默认在受限环境中运行 | `bubblewrap` 创建隔离进程环境 | 调用 `bwrap --new-session --die-with-parent ... -- <shell> -c <command>`；`bwrap` 内部再调用 Linux namespace / mount / exec 相关接口 |
| 文件写限制 | 只能写 workspace、临时目录、额外授权目录 | mount namespace、bind mount、readonly mount | 生成 `bwrap --ro-bind / /`、`--bind <path> <path>`、`--ro-bind <path> <path>` 等参数；底层对应 `mount(2)` / bind mount / readonly remount |
| 文件读限制 | 可以 deny 读敏感目录，再 allow 特定路径 | mount 规则和路径策略 | 生成 `bwrap --tmpfs <path>` 或 `--ro-bind /dev/null <path>` 等参数，把路径隐藏、替换或变成只读 |
| 网络默认拒绝 | 没授权域名就不能访问外网 | network namespace + host proxy | 生成 `bwrap --unshare-net`；底层对应 `CLONE_NEWNET` 网络 namespace |
| 域名 allow/deny | 只允许访问配置过的域名，deny 优先 | HTTP/HTTPS proxy + SOCKS5 proxy | 启动 `socat` 桥接 Unix socket 和沙箱内 `127.0.0.1:3128/1080`，并通过 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY` 注入代理配置 |
| Unix socket 阻断 | 防止进程走本地 socket 绕过网络沙箱 | seccomp BPF | 执行 `apply-seccomp <unix-block.bpf> <command>`；内部调用 `prctl(PR_SET_NO_NEW_PRIVS)`、`prctl(PR_SET_SECCOMP)`，过滤 `socket(AF_UNIX, ...)` |
| 临时目录隔离 | 沙箱命令使用受控 `$TMPDIR` | bind mount / tmpdir policy | 创建受控临时目录，再通过 `bwrap --bind` 或 `--tmpfs` 暴露给沙箱进程 |
| 违规提示 | 命令输出里能标记 sandbox violation | runtime violation store + CLI 展示层 | 不是内核能力，来自运行时记录和上层展示 |
| 排除命令 | 某些命令可配置为不进沙箱 | CLI 策略层，不是内核安全边界 | 不是内核能力，属于执行前的策略判断 |

这里要注意一层关系：Claude Code 沙箱运行时本身主要是在拼 `bwrap`、`socat`、`apply-seccomp` 这些命令和配置；真正调用 `mount(2)`、创建 namespace、安装 seccomp 过滤器的是这些底层工具和它们背后的内核接口。

再具体一点，这里分了三层：

```text
sandbox-runtime: TypeScript 代码，负责拼命令参数
bwrap: Linux 上安装的原生可执行程序，负责真正创建沙箱
Linux kernel: 提供 namespace、mount、seccomp、exec 等系统调用
```

也就是说，TypeScript 本身没有直接调用 `clone(2)` / `mount(2)` 这些 C 风格系统调用。它通常只是通过 Node/Bun 的 `child_process.spawn` 启动一个外部命令。以 `linux-sandbox-utils.ts` 里 `wrapCommandWithSandboxLinux` 拼出的强沙箱参数为例，形状大致是：

```ts
spawn("bwrap", [
  "--new-session", "--die-with-parent",
  "--unshare-net",
  "--unshare-pid",
  "--unshare-user", "--cap-drop", "ALL",
  "--ro-bind", "/", "/",
  "--bind", workspace, workspace,
  "--dev", "/dev",
  "--proc", "/proc",
  "--",
  "bash", "-c", command,
])
```

然后 `bwrap` 这个原生程序再去调用 Linux 内核接口。每一组参数背后都是一类系统接口：`--unshare-*` 对应 `clone(2)`/`unshare(2)` 的 namespace 标志，`--ro-bind`/`--bind` 对应 `mount(2)` 的 bind mount，`--proc`/`--dev` 对应挂载 procfs 和最小化的 `/dev`，`--cap-drop ALL` 对应清空 capabilities，`--die-with-parent` 对应 `prctl(2)` 的 parent-death signal。`clone(2)`、`unshare(2)`、`mount(2)`、`execve(2)` 里的 `(2)` 不是函数参数，也不是版本号，而是 Linux man page 的章节编号。常见编号可以这样理解：

| 写法 | 含义 |
| --- | --- |
| `bash(1)` / `mount(8)` | 用户命令或管理员命令 |
| `mount(2)` / `execve(2)` | 系统调用，程序直接向内核发请求 |
| `printf(3)` / `malloc(3)` | C 库函数 |
| `proc(5)` | 文件格式或伪文件系统说明 |
| `namespaces(7)` / `capabilities(7)` | Linux 概念或机制说明 |

所以 `mount(2)` 强调的是“内核系统调用”，不是命令行里的 `mount` 命令；`mount(8)` 才是用户在 shell 里执行的管理命令。

从用户感知到系统接口，可以把 sandbox runtime 的 Linux 路径整理成这样：

| 用户侧能感知到的沙箱特性 | 主要二进制或组件 | 底层接口 / 内核能力 | 这些接口大概提供什么能力 |
| --- | --- | --- | --- |
| 只能访问允许的目录，比如 workspace 可写、系统目录只读 | `bwrap` / `bubblewrap` | `mount(2)`、`MS_BIND`、`MS_REC`、`MS_REMOUNT`、`MS_RDONLY` | `mount(2)` 是重布置文件系统视图的核心接口；`MS_BIND` 把宿主目录接进沙箱，`MS_REC` 递归带上子挂载点，`MS_REMOUNT` 重新设置已有挂载，`MS_RDONLY` 把挂载点改成只读 |
| 看不到或写不了敏感路径，比如 `~/.ssh`、token、配置目录 | `bwrap`，路径扫描可能辅助用 `rg` | mount namespace、bind mount、tmpfs、路径遮蔽 | 不是只靠应用层判断“别访问”，而是让沙箱进程看到一棵被重新布置过的目录树；敏感路径可以不挂入、只读挂入，或用空目录/tmpfs 覆盖 |
| 沙箱里的 `/tmp` 像一次性草稿纸 | `bwrap` | `mount(2)` + `tmpfs` | 挂载一个内存里的临时文件系统。命令可以写临时文件，但这些内容不污染宿主原来的目录 |
| 沙箱有独立的文件系统挂载表 | `bwrap` | `clone(2)` / `unshare(2)` + `CLONE_NEWNS` | 创建 mount namespace。可以把它理解成给进程一份自己的“挂载地图”，之后它看到的挂载关系可以和宿主不同 |
| 沙箱进程没有特权，被攻破后也难以提权 | `bwrap` | `clone(2)` / `unshare(2)` + `CLONE_NEWUSER`，以及 `/proc/<pid>/uid_map`、`gid_map`、`setgroups` | 创建 user namespace 并写入内外 uid/gid 映射（默认 1:1，保持原 uid）。user namespace 的关键是：进程获得的 capabilities 只对这个 namespace 内的资源生效。再配合 `--cap-drop ALL` 清空全部 capability，沙箱进程对宿主资源没有任何特权，`mount`、`ptrace`、改网络配置等操作都会被内核拒绝 |
| 默认断网，命令找不到外网出口 | `bwrap` | `clone(2)` / `unshare(2)` + `CLONE_NEWNET` | 创建 network namespace。沙箱有自己的网卡、IP、路由表；如果不给它 `eth0` 和 default route，它就没有通向外网的路 |
| 只能通过受控代理联网 | `socat` | `socket(2)`、`bind(2)`、`listen(2)`、`accept(2)`、`connect(2)`、`read(2)`、`write(2)` | 和 `--unshare-net` 配合使用：沙箱里根本没有外网网卡，所有流量只能走挂在 Unix socket 上的代理桥。宿主侧 `socat` 监听一个 Unix socket 并转发到代理端口；沙箱内再启动一个 `socat`，把 `127.0.0.1:3128`（HTTP）和 `127.0.0.1:1080`（SOCKS）转回那个 Unix socket。请求经过代理时，由 allow/deny 域名规则决定是否放行 |
| 不能通过本地 Unix socket 绕过网络代理 | `apply-seccomp` + `unix-block.bpf` | `prctl(2)`、`PR_SET_NO_NEW_PRIVS`、`PR_SET_SECCOMP`、`seccomp(2)`、classic BPF filter | seccomp 给进程安装系统调用过滤器。规则让 `socket(AF_UNIX, ...)` 返回 EPERM，避免绕过 HTTP/HTTPS 代理去连 Docker socket、SSH agent 等本地服务；同时挡掉 `io_uring_setup`/`io_uring_enter`/`io_uring_register`，因为 Linux 5.19+ 可以用 `IORING_OP_SOCKET` 绕过 `socket()` 这条规则 |
| 只能看到沙箱内进程 | `bwrap` + `apply-seccomp` | `clone(2)` / `unshare(2)` + `CLONE_NEWPID`，再用 `mount(2)` 挂载匹配的 `procfs` | PID namespace 给进程一棵新的进程树；重新挂 `/proc` 后，`ps` 这类工具看到的就是沙箱里的进程，而不是宿主机全部进程。apply-seccomp 还会再套一层嵌套 PID namespace，让用户命令连 socat、bash 包装进程都看不到，防止通过 ptrace 它们来绕过 seccomp |
| 命令退出后不留后台进程残留 | `bwrap` | `setsid(2)`、`prctl(2)` + `PR_SET_PDEATHSIG` | `--new-session` 让沙箱进程自成会话；`--die-with-parent` 让内核在 bwrap 退出时向沙箱进程发 SIGKILL，命令里用 `&` 拉起的后台进程也会被一并带走，不会逃逸到宿主 |
| 隔离 hostname、IPC、cgroup 视图等其他系统视角 | `bwrap` 或容器运行时 | `CLONE_NEWUTS`、`CLONE_NEWIPC`、`CLONE_NEWCGROUP` | 分别隔离 hostname/domainname、System V IPC / POSIX message queue、进程看到的 cgroup 层级。这些不是每个沙箱配置都必须启用，但属于同一套 namespace 思路 |
| 限制 CPU、内存、进程数、IO 等资源 | runtime、systemd、容器工具，不一定是单个二进制 | cgroup v2、cgroupfs、systemd slice/scope | cgroup 把进程放进资源控制组，内核按组限制资源。直观说，就是给沙箱一个“最多只能用这么多”的额度 |
| 更细粒度限制文件读写 | runtime 或 helper，可能直接调用 syscall | Landlock：`landlock_create_ruleset`、`landlock_add_rule`、`landlock_restrict_self` | Landlock 允许普通进程给自己加文件访问规则。规则生效后，即使进程原本有某些文件权限，也会被这层规则再次收窄 |
| root 权限被拆成小块能力 | `bwrap`、runtime、系统工具 | capabilities、`capget(2)`、`capset(2)`、`prctl(2)` | Linux 把传统 root 权限拆成很多 capability，例如挂载、改网络、调试进程等。沙箱可以丢弃不需要的能力，避免“看起来是 root 就什么都能做” |
| 沙箱环境搭好后启动用户命令 | `bwrap`、`apply-seccomp` | `fork(2)` / `clone(2)`、`execve(2)` | 前面是在搭舞台：隔离视图、布置目录、装过滤器；`execve(2)` 是最后一步，把进程替换成真正要跑的命令，比如 `bash -c <command>` |

所以从依赖角度看，业务代码依赖的是 `bwrap` 这个用户态程序；`bwrap` 再依赖 Linux 内核支持 namespace、mount、procfs、tmpfs 和进程执行这些能力。如果目标系统没有这些内核接口，就不能直接平移 `bwrap` 这套实现，只能找等价的进程隔离、文件视图隔离和网络隔离能力。

调用链可以粗略理解成这样：

```text
Claude Code / sandbox runtime
  -> spawn("bwrap", ["--unshare-net", "--ro-bind", "/", "/", "--bind", workspace, workspace, "--", "bash", "-c", command])
  -> bwrap 进程启动
  -> bwrap 调用 unshare/clone 创建新的 namespace
  -> bwrap 调用 mount 配置只读根目录、可写 workspace、tmpfs、/proc、/dev
  -> bwrap 设置环境变量和工作目录
  -> bwrap 最后 execve("bash", ["bash", "-c", command], env)
```

如果不用 `bwrap`，直接写 C 代码，形状大概会是：

```c
// 非特权进程的第一步：创建 user namespace，拿到只对本 namespace 生效的 CAP_SYS_ADMIN
unshare(CLONE_NEWUSER);
write_file("/proc/self/setgroups", "deny");
write_file("/proc/self/uid_map",   "0 1000 1");
write_file("/proc/self/gid_map",   "0 1000 1");

unshare(CLONE_NEWNS | CLONE_NEWNET);
mount("/", "/", NULL, MS_BIND | MS_REC, NULL);
mount(NULL, "/", NULL, MS_REMOUNT | MS_RDONLY | MS_BIND | MS_REC, NULL);
mount(workspace, workspace, NULL, MS_BIND, NULL);
mount("proc", "/proc", "proc", 0, NULL);
clone(child_fn, child_stack, CLONE_NEWPID | SIGCHLD, NULL);

// child_fn 里再执行真实命令
execve("/bin/bash", argv, envp);
```

这段不是 `bwrap` 的真实源码，只是帮助理解调用顺序：先隔离视图，再重新布置文件系统，再创建要运行命令的子进程，最后执行用户命令。`seccomp` 那部分则是另一条链路：`apply-seccomp` 先调用 `prctl(PR_SET_NO_NEW_PRIVS)`，再调用 `prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog)`，最后 `execve` 用户命令。

Linux 上比较关键的二进制依赖是：

| 二进制 / 文件 | 来源 | sandbox-runtime 怎么用 | 依赖的系统接口 |
| --- | --- | --- | --- |
| `bwrap` / `bubblewrap` | Linux 发行版包，一般通过 `apt install bubblewrap`、`dnf install bubblewrap`、`pacman -S bubblewrap` 安装 | 由 TS 层拼参数并 `spawn("bwrap", args)`；负责创建命令沙箱 | namespace: `clone(2)` / `unshare(2)`；mount: `mount(2)` / bind mount / readonly remount；process: `fork(2)` / `clone(2)` / `execve(2)`；还会使用 procfs、tmpfs、`/dev` 相关挂载能力 |
| `socat` | Linux 发行版包，一般通过 `apt install socat`、`dnf install socat`、`pacman -S socat` 安装 | 在宿主和沙箱之间做 Unix socket / TCP 代理桥接，让沙箱里的 HTTP/SOCKS 代理请求转到宿主侧代理 | socket API: `socket(2)`、`bind(2)`、`listen(2)`、`accept(2)`、`connect(2)`、`read(2)` / `write(2)`；会用到 Unix domain socket 和 TCP socket |
| `rg` / `ripgrep` | Linux 发行版包或已有系统工具 | 辅助扫描 deny path / 危险路径，帮助生成 mount 规则；它不是沙箱边界本身 | 普通文件系统接口，如 `open(2)`、`read(2)`、`stat(2)`、目录遍历等 |
| `apply-seccomp` | `@anthropic-ai/sandbox-runtime` 包内 vendor 自带的原生静态二进制，位于 `vendor/seccomp/<arch>/apply-seccomp`，C 源码在 `vendor/seccomp-src/` | 在执行用户命令前：创建嵌套 user+PID+mount namespace → 挂载私有 `/proc` → 安装 seccomp 过滤器 → `execve` 用户命令 | `unshare(2)` + `CLONE_NEWUSER`/`CLONE_NEWPID`/`CLONE_NEWNS`、写 `uid_map`/`gid_map`/`setgroups`、`mount(2)` 挂 procfs、`prctl(PR_SET_NO_NEW_PRIVS)`、`prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog)`、`prctl(PR_SET_DUMPABLE)`、信号转发与 `waitpid(2)` |
| `unix-block.bpf` | 新版把 BPF 规则直接编译进 `apply-seccomp`（由 `vendor/seccomp-src/seccomp-unix-block.c` 生成）；旧版是独立数据文件 | 作为 seccomp filter 数据安装到进程 | 不是可执行程序；它是 classic BPF 规则，目标是让 `socket(AF_UNIX, ...)` 和 `io_uring_setup`/`io_uring_enter`/`io_uring_register` 返回 `EPERM` |

其中 `bubblewrap` 是最核心的容器化工具。它负责把前面实验过的能力组合起来：创建 namespace、配置 bind mount、把某些路径变成只读、隔离网络视图等。

`socat` 用在网络代理桥接上。这个实现不是简单地把沙箱网络完全打开，而是让沙箱进程通过受控代理访问网络。Linux 上网络请求会通过 Unix domain socket 走到宿主侧代理，再由代理根据 allow/deny 域名规则决定是否放行。

`ripgrep` 不是沙箱内核能力，而是辅助工具。它用于快速扫描一些危险路径或 deny path，帮助 runtime 在构造 mount 规则时找到需要保护的文件。

seccomp 这里主要用于阻断 Unix domain socket 创建。原因是：即使外网被 network namespace 和代理限制了，本地 Unix socket 仍可能成为绕过路径。例如某些系统服务、Docker socket、agent socket 一旦暴露，权限可能比普通网络请求大得多。这个实现会带一个 `apply-seccomp` helper 和一个预生成的 `unix-block.bpf` 过滤器；运行时由 helper 调用内核接口安装 seccomp 规则，再 exec 真正的用户命令。

这两个东西可以这样理解：

```text
apply-seccomp = seccomp 规则安装器 + 嵌套 namespace 搭建者，是可以执行的原生程序
unix-block.bpf = seccomp 规则数据（新版本直接编译进 apply-seccomp，不再单独携带）
```

它们解决的是同一个场景：防止沙箱里的命令通过本地 Unix domain socket 绕过网络沙箱。比如普通外网已经被 network namespace 切掉了，HTTP/HTTPS 访问也必须经过受控代理；但如果进程还能随便创建 Unix socket，它可能去连接 `/var/run/docker.sock`、SSH agent、GPG agent、系统服务 socket 或其他本地 agent socket。这类访问不一定经过域名 allow/deny 代理，因此可能变成绕过路径。

所以 Linux 上会分两阶段处理：

```text
1. bwrap 先创建文件系统、网络、PID 等沙箱
2. 如果需要代理，先在沙箱里启动 socat 桥接
3. apply-seccomp 读取 unix-block.bpf
4. apply-seccomp 把 BPF 规则安装到当前进程
5. apply-seccomp execve 用户命令
6. 用户命令继承 seccomp 限制
7. 用户命令再调用 socket(AF_UNIX, ...) 时返回 EPERM
```

这里不能一开始就安装 `unix-block.bpf`，因为代理桥接用的 `socat` 自己也需要 Unix socket。先让 `socat` 启动，再用 `apply-seccomp` 给真正的用户命令加限制，才能同时保留受控代理能力，并阻断用户命令自己创建新的 Unix socket。

### 11.1 apply-seccomp 为什么还要嵌套 namespace

`apply-seccomp` 不是简单地"安装过滤器然后 exec"，它会在外层 bwrap 沙箱里再创建一层嵌套的 user + PID + mount namespace，并重新挂载 `/proc`。原因：如果用户命令和 `socat`、bash 包装脚本住在同一个 PID namespace 里，用户命令就能看到这些"没有装过滤器"的进程，攻击者可以尝试 `ptrace` 它们或读写 `/proc/<pid>/mem`，从而绕过 seccomp 限制。放进嵌套 PID namespace 后，用户命令视角里的 `/proc` 只剩自己这棵小进程树，bwrap 的 init、bash 包装、socat 都不可见、不可寻址。

嵌套后的进程布局：

```text
bwrap init（外层 PID 1，无 seccomp）
└─ bash 包装 / socat 桥（外层，无 seccomp）
   └─ apply-seccomp（外层，等待内层退出并转发退出码）
      ═══════ 嵌套 PID namespace 边界 ═══════
      └─ apply-seccomp（内层 PID 1，PR_SET_DUMPABLE=0，负责收割子进程、转发信号）
         └─ 用户命令（内层 PID 2，seccomp 已生效）

用户命令的视角：/proc 里只有自己的进程树
```

内层 PID 1 会设置 `PR_SET_DUMPABLE=0`，让自己也不可被 ptrace/转储；它还负责把外部信号转发给用户命令（PID 1 如果不处理信号，外面发来的 `SIGTERM` 会被静默丢弃）。安装顺序是：`unshare(CLONE_NEWPID|CLONE_NEWNS)`（没有特权时先 `unshare(CLONE_NEWUSER)`，再写 `/proc/self/setgroups`、`uid_map`、`gid_map`）→ `fork` → 挂载私有 `/proc` → `prctl(PR_SET_NO_NEW_PRIVS)` → `prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &bpf)` → `execve` 用户命令。任何一步失败都会中止运行，绝不降级成"没有隔离地执行"。

### 11.2 Linux 实现的几个平台细节

**路径配置只支持字面路径，不支持通配符。** macOS 的 Seatbelt profile 支持 git 风格 glob（`**/*.ts` 这类模式）；Linux 的配置会直接翻译成 bwrap 的 bind mount 参数，写什么路径就保护什么路径，没有把 glob 展开成多个挂载点的逻辑，所以同样的配置在 Linux 上要用完整字面路径。

**bind mount 只能遮蔽"已经存在"的文件。** 用 `--ro-bind /dev/null <path>` 去遮一个不存在的路径时，bwrap 会在宿主文件系统上创建一个空的挂载点文件；命令结束后，这个"幽灵文件"会留在工作目录里（比如 `.bashrc`、`.gitconfig`）。sandbox-runtime 为此维护了一套挂载点清理逻辑，并且只在没有其他沙箱同时运行时才删除——删早了会让还在运行的沙箱的遮蔽失效。

**危险路径的强制保护靠 ripgrep 扫描。** 除了用户配置，还有一批"永远禁止写入"的路径：`.bashrc`、`.zshrc`、`.gitconfig`、`.mcp.json`、`.ripgreprc`、`.claude/commands/`、`.claude/agents/`、`.git/hooks/`、`.git/config`、`.vscode/`、`.idea/` 等。Linux 上 bind 规则只对已存在的路径生效，所以运行时用一次 ripgrep 扫描允许写的目录（默认往下 3 层，可配 1-10 层），把扫到的危险文件/目录转成额外的 deny 挂载参数。这就是 `ripgrep` 在这个实现里的角色：不是沙箱边界，而是"把配置翻译成挂载规则"之前的路径扫描器。

**Ubuntu 24.04 有一个需要先处理的内核参数。** Ubuntu 24.04 默认开启 `kernel.apparmor_restrict_unprivileged_userns`：它允许 `unshare(CLONE_NEWUSER)`，但会从新 namespace 里剥掉 capabilities。bwrap 和 apply-seccomp 都需要"带 capabilities 的 user namespace"（bwrap 要靠它完成挂载，apply-seccomp 要靠它创建嵌套 namespace），所以在这个发行版上要先 `sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0`，或者给相关二进制加 AppArmor profile。这个例子也说明：沙箱可用性不仅取决于"系统有没有 namespace 接口"，还取决于"namespace 里保留不保留特权"。

把它和前面的手动实验对应起来：

```text
文件系统可见性  -> mount namespace / bind mount / readonly mount
网络是否可达    -> network namespace / proxy / domain allowlist
本地 IPC 绕过   -> seccomp 阻断 Unix socket
命令是否进沙箱  -> CLI 策略层
路径是否可写    -> allowWrite / denyWrite 转 mount 策略
路径是否可读    -> denyRead / allowRead 转 mount 策略
```

所以真实产品里的 Linux 沙箱并不是单点机制，而是组合拳：

```text
namespace 改变进程看到的世界
mount 策略控制文件系统可见性和可写性
network namespace 切断默认外网
proxy 把允许的网络访问重新接回来
seccomp 堵住 Unix socket 这类本地绕过
CLI 权限层决定什么时候启用、什么时候提示、什么时候允许例外
```

这个分层也解释了为什么“沙箱能力”看起来像产品功能，实际依赖的是操作系统能力。产品代码负责把权限意图翻译成配置；真正执行拒绝和隔离的，还是 Linux 内核和围绕它的工具链。

### 11.3 如果自己封装这些底层接口，工作量有多大

“调用接口”本身并不贵，贵的是接口之间的顺序约束、失败处理和语义映射。可以用 `bwrap` 和 `apply-seccomp` 的真实代码规模来感受一下：

| 组件 | 规模 | 主要工作内容 |
| --- | --- | --- |
| `bubblewrap.c` | 约 100KB（2500+ 行 C） | CLI 参数解析、namespace 编排、mount 视图构建、进程模型、错误降级 |
| `bind-mount.c` | 约 16KB | bind mount 底层封装：目标不存在时的处理、挂载点创建、参数校验 |
| `utils.c` / `network.c` | 约 27KB | 公共工具、/proc 读写、网络相关封装 |
| 合计（不含测试） | 约 150KB，约 4000 行 C | 多年打磨的成熟实现 |
| `apply-seccomp.c` | 约 36KB，约 1000 行 C | 嵌套 namespace 编排、uid/gid 映射、BPF 安装、信号转发与子进程收割 |

对照功能拆开看，各部分的工作量大概是：

| 模块 | 依赖的接口 | 大致工作量 | 难点在哪 |
| --- | --- | --- | --- |
| namespace 编排 | `unshare(2)`/`clone(2)` + 写 `uid_map`/`gid_map`/`setgroups` | 500-800 行 C | 顺序约束（先 user namespace，再 mount/PID）、映射写入时机、`fork` 与 `CLONE_NEWPID` 的配合 |
| mount 视图构建 | `mount(2)` + `MS_BIND`/`MS_REC`/`MS_REMOUNT`/`MS_RDONLY`、tmpfs、procfs、devtmpfs | 1200-1500 行 C | mount propagation 处理、只读 remount 的顺序、目标路径不存在、跨发行版差异 |
| seccomp 过滤 | `prctl(2)`/`seccomp(2)` + BPF | 用 libseccomp 几百行；手写 BPF 更多 | 架构相关（x86-64/arm64 的 syscall 号不同）、x32 ABI、多个过滤器组合、USER_NOTIF 观察 |
| 进程监督 | `fork(2)`/`execve(2)`/`sigaction(2)`/`waitpid(2)`/`prctl(PR_SET_PDEATHSIG)` | 300-500 行 C | 信号转发、僵尸收割、后台进程回收 |
| 代理桥接 | `socket(2)`/`connect(2)` 等 | 约 200 行配置 | 基本无难点，直接复用 `socat` |

经验性的结论：

```text
最小可用版（userns + 只读根 + bind 开放可写路径 + exec）：
  熟悉 Linux 系统编程的人约 2-4 周，500-800 行 C。

bwrap 级别（覆盖所有边界情况、跨发行版、错误降级）：
  数月级，约 4000 行 C，测试量与实现量相当。

seccomp 部分：优先复用 libseccomp，不要手写 BPF。
```

还有一类容易被低估的工作：**配置语义到挂载规则的翻译**。sandbox-runtime 自己的 `linux-sandbox-utils.ts` 有几千行 TypeScript，大部分在算“哪个 deny 路径要 `--ro-bind /dev/null`、哪个目录要 `--tmpfs` 覆盖、allow 路径被 deny 的 tmpfs 覆盖后要不要重新 bind 回来”。如果把 bwrap 换成自研实现，这层翻译逻辑和它的边界情况（符号链接、路径不存在、嵌套目录互相覆盖）都要重新走一遍。

最后，这套实现对系统接口的依赖收敛起来其实很集中，可以按重要程度排个序：

```text
1. user namespace（CLONE_NEWUSER + uid_map/gid_map）：
   非特权进程获得“只对 namespace 内资源生效”的特权的基础。没有它，普通用户连挂载类操作都做不了
2. mount namespace + bind mount（CLONE_NEWNS + mount(2)）：
   文件视图隔离的全部基础：哪些目录可见、哪些可写、哪些被遮蔽
3. network namespace（CLONE_NEWNET）：
   “默认断网”的前提；代理再把允许的流量接回来
4. PID namespace + procfs（CLONE_NEWPID + 挂载 procfs）：
   进程树隔离，以及让 /proc 视图和隔离后的进程树配套
5. seccomp（prctl(2) / seccomp(2)）：
   syscall 级兜底，堵住文件/网络隔离管不到的绕过路径（Unix socket、io_uring）
6. 进程原语（fork/execve/signal/waitpid）+ socket 转发工具：
   搭建和运行沙箱的脚手架，socat 即可
```

前四项都在 `bwrap` 里，第五项是 `apply-seccomp`，第六项是 `socat`。评估另一个系统能不能跑这套沙箱时，逐项确认这些能力是否存在（或有没有等价物），比直接问“能不能装 bwrap”要准确得多；缺了 user namespace 或 syscall 过滤这类关键项，工作量就不是“封装”而是“重新设计”了。

### 11.4 汇总：从用户需求到内核接口

把前几节的内容压成一张表：每一行都是一条"用户感知的需求 → 二进制实现 → 源码 → 封装方式 → 工作量 → 目的 → 内核接口 → 接口通俗解释"的完整链路。

| # | 用户感知的需求 | 调用的二进制 | 开源情况（源码 / 许可证） | 二进制怎么封装 | 工作量 | 达到什么目的 | 底层接口 | 接口功能（通俗版） |
|---|---|---|---|---|---|---|---|---|
| 1 | 系统目录只读，只有 workspace 可写 | `bwrap` | ✅ 开源，https://github.com/containers/bubblewrap ，LGPL-2.0-or-later，约 4000 行 C | 拼 `--ro-bind / /`（整棵只读根）+ `--bind <workspace> <workspace>`（单独开放可写） | mount 视图构建约 1200-1500 行 C | 让沙箱进程“看到”一棵重新布置过的目录树，文件根本写不出/读不进它不该碰的地方 | `mount(2)` + `MS_BIND`/`MS_REC`/`MS_REMOUNT`/`MS_RDONLY` | mount=把文件系统“挂”到目录树上；MS_BIND=把宿主目录原样映射到另一个路径（内核级快捷方式）；MS_REC=连子挂载点一起搬；MS_REMOUNT=改已有挂载的属性；MS_RDONLY=改成只读。合起来=整棵复制一套只读系统树，再把工作目录单独开放 |
| 2 | 敏感路径不可读/写（`~/.ssh`、token） | `bwrap` | ✅ 同 1 | 文件用 `--ro-bind /dev/null <path>` 遮蔽，目录用 `--tmpfs <dir>` 覆盖 | 同 1（mount 模块内）+ 一套“幽灵挂载点”清理逻辑 | deny 优先：让进程在路径层面根本找不到敏感内容，而不是靠程序自觉 | `mount(2)` + `MS_BIND`、`tmpfs` | 把 /dev/null（无底洞）bind 到敏感文件上，读它只得到空；在敏感目录上挂一块“临时黑板”（tmpfs），里面空空如也、写啥都丢 |
| 3 | 沙箱的 `/tmp` 是一次性草稿纸 | `bwrap` | ✅ 同 1 | 创建受控临时目录 + `--bind`/`--tmpfs` 挂入 | 同 1（mount 模块内） | 临时文件不污染宿主目录 | `mount(2)` + `tmpfs` | tmpfs 是内存里的文件系统，卸载即消失，像一次性便签本 |
| 4 | 默认断网，命令找不到外网 | `bwrap` | ✅ 同 1 | `--unshare-net` | namespace 编排共 500-800 行 C（userns/mount/PID/网络全部在这里） | 沙箱默认没有外网出口，任何想联网的程序直接无路可走，比“告诉它别联网”硬得多 | `unshare(2)`/`clone(2)` + `CLONE_NEWNET` | 给进程一个全新的“网络世界”，里面没有网卡、没有路由表，像刚装好的路由器还没插网线 |
| 5 | 只允许访问配置过的域名（网络白名单） | `socat` + JS 代理（HTTP/SOCKS5） | ✅ socat：https://www.dest-unreach.org/socat/ （GitHub 镜像 3hhh/socat），GPL-2.0；JS 代理在 sandbox-runtime 内，Apache-2.0 | 宿主侧 `socat` 监听 Unix socket 转发到代理端口；沙箱内再起一个 `socat` 把 `127.0.0.1:3128/1080` 转回 Unix socket；注入 `HTTP_PROXY` 等环境变量 | 桥接约 200 行配置，代理过滤是 JS 层业务逻辑 | 网络默认断，要联网必须走代理这扇门，代理按 allow/deny 域名规则放行 | `socket(2)`/`bind(2)`/`listen(2)`/`accept(2)`/`connect(2)` | 沙箱里只留一扇“小门”（Unix socket 桥），所有流量必须走这门，门卫（代理）查完域名才放行 |
| 6 | 不能用本地 Unix socket 绕过网络限制 | `apply-seccomp` + `unix-block.bpf` | ✅ 开源，sandbox-runtime 仓库 `vendor/seccomp-src/`（apply-seccomp.c + seccomp-unix-block.c），Apache-2.0，约 1000 行 C | 创建嵌套 user+PID+mount namespace → 重挂 `/proc` → `PR_SET_NO_NEW_PRIVS` → 装 BPF 过滤器 → `execve` | apply-seccomp 约 1000 行 C（BPF 规则若手写还要更多，建议用 libseccomp） | 堵住 Docker socket、SSH agent 等本地 socket 绕过路径；连 io_uring 也一起挡（防 `IORING_OP_SOCKET` 绕过） | `prctl(2)` + `PR_SET_SECCOMP`、`seccomp(2)` + BPF | 给进程装一道“安检门”：每次调系统调用先过安检，发现要创建 AF_UNIX socket 直接拒绝（返回 EPERM） |
| 7 | 进程只能看到沙箱内进程 | `bwrap` + `apply-seccomp` | ✅ 同 1 + 同 6 | `--unshare-pid` + `--proc /proc`；apply-seccomp 再套一层嵌套 PID namespace 并重挂私有 `/proc` | 嵌套编排含在 apply-seccomp 的 1000 行里 | 用户命令连 socat、bash 包装、init 都看不到，无法通过 ptrace/`/proc/N/mem` 攻击它们绕过 seccomp | `unshare(2)`/`clone(2)` + `CLONE_NEWPID`、`mount(2)` 挂 procfs | 给进程一个“只有自己家的小区”，外面的人和房子全看不见；/proc 像户口本，重新挂上配套户口本，`ps` 才显示得对 |
| 8 | 沙箱内无特权，被攻破也难提权 | `bwrap` | ✅ 同 1 | `--unshare-user` + `--cap-drop ALL` | 身份映射逻辑含在 namespace 编排里 | 进程拿不到 CAP_SYS_ADMIN 这类特权钥匙，mount、ptrace、改网络配置全被内核拒绝 | `unshare(2)`/`clone(2)` + `CLONE_NEWUSER`、写 `/proc/<pid>/uid_map`/`gid_map`/`setgroups`、`capset(2)` | 新身份空间里 uid 映射 1:1，但所有“特权钥匙”（capabilities）被收走，进程只能干普通用户能干的事 |
| 9 | 命令退出后不残留后台进程 | `bwrap` | ✅ 同 1 | `--new-session` + `--die-with-parent` | 进程监督共 300-500 行 C | bwrap 一退出，沙箱整体消失，`&` 拉起的后台进程一起带走 | `setsid(2)`、`prctl(2)` + `PR_SET_PDEATHSIG` | 自成一套会话、和终端脱钩；再签一份“老爸死了我就自杀”的协议（PDEATHSIG），bwrap 退出的瞬间内核发 SIGKILL 全家带走 |
| 10 | 最后真正跑用户命令 | `bwrap` / `apply-seccomp` | ✅ 同 1 + 同 6 | 布置完环境后 `-- <shell> -c <command>`；apply-seccomp 里用 `execvp` | 进程原语 | 前面全在搭舞台，这一步把舞台交给用户命令 | `fork(2)`/`clone(2)`、`execve(2)` | fork=生一个子进程，execve=把子进程“身体”换成要跑的程序，像换演员上台 |
| 11 | 危险文件永远禁写（`.bashrc`、`.git/hooks` 等） | `rg`（ripgrep，辅助） | ✅ 开源，https://github.com/BurntSushi/ripgrep ，MIT / Unlicense 双许可 | `rg --files --hidden --max-depth 3` 扫描允许写目录，把命中的危险路径转成 bwrap 的 deny 挂载参数 | 翻译逻辑是 TS 层几千行的一部分（linux-sandbox-utils.ts） | bind 规则只对已存在的文件生效，所以要先把危险文件“找出来”再遮 | `open(2)`/`read(2)`/`stat(2)`/目录遍历 | 翻文件夹找出危险文件，报告给 bwrap 去遮住，本身不是安全边界 |
| 12 | 限制 CPU/内存/进程数（可选） | 未启用（可选项） | —（需要时选 cgroup v2，Linux 内核能力） | 需要时用 cgroup v2 | 另起一套，不算在 bwrap 路径里 | 给沙箱一个“最多只能用这么多”的额度 | cgroup v2 | 给进程组发“粮票”，超了内核直接限流 |

许可证补充说明：bwrap（LGPL-2.0-or-later）、socat（GPL-2.0）是 copyleft，修改后分发需按对应协议开源；apply-seccomp（Apache-2.0）、ripgrep（MIT/Unlicense）宽松，改完可闭源；sandbox-runtime 本体（TS 层翻译逻辑）也是 Apache-2.0。学习参考不受任何限制。
