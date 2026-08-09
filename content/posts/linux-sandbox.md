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
