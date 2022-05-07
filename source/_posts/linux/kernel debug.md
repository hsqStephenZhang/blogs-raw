---
title: kernel debug 环境配置（100%可靠）
categories: [linux, debug]
date: 2022-5-7 21:30:00
---

## 1. kernel 编译

1. make menuconfig

需要打开下面这个选项：
`Kernel hacking —> Compile-time checks and compiler options —> [ ] Compile the kernel with debug info`（对应 `.config` 文件中的 `CONFIG_DEBUG_INFO`）。

否则，在后续的调试时，无法将指令和源码文件一一对应。

2. make -j$(nproc)

多线程编译内核，最终会得到 `vmlinux` 和 `arch/x86_64/boot/bzImage` 两个文件。

## 2. rootfs 制作

这里用到了 buildroot 这个项目，可以通过 `git clone git://git.buildroot.net/buildroot` 指令下载到本地，然后执行下面操作

1. `make menuconfig`，并进行如下操作

- Target options -> Target architecture, select x86_64
- Filesystem images -> ext2/3/4 root filesystem; then choose the ext4 variant
- Target packages -> Network applications -> openssh [*], or any tools you need

2. make -j$(nproc)

编译过程会下载很多文件，请保证与 github 连接畅通，否则自行 hack 下载网址

## 3. qemu 虚拟机网卡配置

1. apt install bridge-utils && apt install uml-utilities
2. 
   - brctl addbr br0
   - ifconfig br0 192.168.2.1
   - ip tuntap add tap0 mode tap
   - ifconfig tap0 0.0.0.0 promisc up
   - brctl addif br0 tap0
   - echo 1 > /proc/sys/net/ipv4/ip_forward
   - iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
3. qemu-system-x86_64 -kernel ./arch/x86_64/boot/bzImage \
    -nographic \
    -hda /path/to/buildroot/output/image/rootfs.ext4 \
    -append "nokaslr root=/dev/sda console=ttyS0"  \
    -smp 4 \
    -netdev tap,id=mynet0,ifname=tap0,script=no,downscript=no -device e1000,netdev=mynet0,mac=52:55:00:d1:55:01 \
    -s -S

4. 启动之后，需要执行 `ifconfig eth0 192.168.2.2 && route add default gw 192.168.2.1` 来设置网卡的 ip 地址，此时已经可以访问 host 和通过 ip 访问外网
5. 拷贝宿主机 `/etc/resolv.conf` 到 qemu 虚拟机中，尝试 `ping baidu.com`