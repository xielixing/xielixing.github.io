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
- `groups` 是当前进程额外所属的用户组列表。

Linux 最基础的文件权限判断，就是拿当前进程携带的 `uid/gid/groups` 去和文件的 owner/group/others 权限匹配。也就是说，内核不是先问“你叫什么名字”，而是看“这个进程带着哪些身份标记”。

至于为什么这里是 `1000`，不是 `1001`：在 Ubuntu/Debian 这类系统里，普通用户 UID 通常从 `1000` 开始分配。这个 WSL 发行版之前没有普通用户，所以我们创建的 `sandboxer` 成了第一个普通用户，自然拿到了 `1000`。如果系统里已经有一个普通用户占用了 `1000`，下一个用户通常才会拿到 `1001`。

后面的命令默认都在这个身份和目录下执行：

```text
用户：sandboxer
目录：/home/sandboxer/sandbox-lab
提示符：sandboxer@DESKTOP-ABLKNN8C:~/sandbox-lab$
```

下一步要观察的第一件事，是普通用户为什么不能读取 root 的私有目录。这会把“uid/gid 决定基础权限”的直觉先钉住，然后再进入 user namespace。
