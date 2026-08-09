+++
title = 'Linux 沙箱机制：从 Namespace 到 Seccomp'
date = 2026-08-09T11:15:00+08:00
draft = false
summary = '梳理 Linux 自带沙箱能力的基础机制，包括 namespace、mount 隔离、capabilities、seccomp、cgroup、Landlock 等，以及它们如何组合成一个可用的命令执行沙箱。'
tags = ['linux', 'sandbox', 'namespace', 'seccomp', 'landlock']
categories = ['Engineering']
+++

这篇文章先不急着从定义开始。Linux 的沙箱能力很容易被讲成一串名词：namespace、capabilities、seccomp、cgroup、Landlock。真正建立直觉的方式，是从一个普通进程开始，一步一步看内核到底根据什么放行、拒绝、隔离和收紧权限。

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

也就是说，TypeScript 本身没有直接调用 `clone(2)` / `mount(2)` 这些 C 风格系统调用。它通常只是通过 Node/Bun 的 `child_process.spawn` 启动一个外部命令：

```ts
spawn("bwrap", [
  "--unshare-net",
  "--ro-bind", "/", "/",
  "--bind", workspace, workspace,
  "--",
  "bash", "-c", command,
])
```

然后 `bwrap` 这个原生程序再去调用 Linux 内核接口。`clone(2)`、`unshare(2)`、`mount(2)`、`execve(2)` 里的 `(2)` 是 man page 的分区编号，意思是“系统调用”，不是函数参数。

`bwrap` 背后大概会落到几类 Linux 接口：

| 类型 | 典型接口 | 在这里的作用 |
| --- | --- | --- |
| namespace | `clone(2)` / `unshare(2)`，配合 `CLONE_NEWNS`、`CLONE_NEWNET`、`CLONE_NEWPID` 等 flag | 创建新的 mount、network、PID 视图，让沙箱进程看到的文件挂载、网络、进程树和宿主不同 |
| mount | `mount(2)` / bind mount / remount readonly，必要时配合 `pivot_root(2)` 或类似根目录切换手段 | 把宿主路径按规则重新挂进沙箱：有的可写，有的只读，有的用 tmpfs 或 `/dev/null` 替换 |
| exec | `fork(2)` / `clone(2)` 创建子进程，最后 `execve(2)` 执行 shell 或真实命令 | 沙箱环境准备好之后，把当前进程替换成用户要跑的 Bash 命令 |
| proc/dev | `mount(2)` 挂载新的 `/proc`，`bwrap --dev /dev` 准备受控设备目录 | 避免沙箱进程直接看到宿主完整 `/proc`，同时提供基本设备文件 |

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
| `apply-seccomp` | `@anthropic-ai/sandbox-runtime` 包内 vendor 自带的原生二进制，位于 `vendor/seccomp/<arch>/apply-seccomp` | 在执行用户命令前安装 seccomp 过滤器，然后再 `execve` 用户命令 | `prctl(PR_SET_NO_NEW_PRIVS)`、`prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog)`、`execve(2)` |
| `unix-block.bpf` | `@anthropic-ai/sandbox-runtime` 包内 vendor 自带的预生成 BPF 过滤器，位于 `vendor/seccomp/<arch>/unix-block.bpf` | 作为 seccomp filter 数据传给 `apply-seccomp`，用于阻断 Unix socket 创建 | 不是可执行程序；它是给 seccomp 使用的 BPF 规则，目标是让 `socket(AF_UNIX, ...)` 返回 `EPERM` |

其中 `bubblewrap` 是最核心的容器化工具。它负责把前面实验过的能力组合起来：创建 namespace、配置 bind mount、把某些路径变成只读、隔离网络视图等。

`socat` 用在网络代理桥接上。这个实现不是简单地把沙箱网络完全打开，而是让沙箱进程通过受控代理访问网络。Linux 上网络请求会通过 Unix domain socket 走到宿主侧代理，再由代理根据 allow/deny 域名规则决定是否放行。

`ripgrep` 不是沙箱内核能力，而是辅助工具。它用于快速扫描一些危险路径或 deny path，帮助 runtime 在构造 mount 规则时找到需要保护的文件。

seccomp 这里主要用于阻断 Unix domain socket 创建。原因是：即使外网被 network namespace 和代理限制了，本地 Unix socket 仍可能成为绕过路径。例如某些系统服务、Docker socket、agent socket 一旦暴露，权限可能比普通网络请求大得多。这个实现会带一个 `apply-seccomp` helper 和一个预生成的 `unix-block.bpf` 过滤器；运行时由 helper 调用内核接口安装 seccomp 规则，再 exec 真正的用户命令。

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
