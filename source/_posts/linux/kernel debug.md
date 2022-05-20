---
title: kernel debug 环境配置（100%可靠）
categories: [linux, debug]
date: 2022-5-7 21:30:00
---

## 1. kernel 编译

1. `make menuconfig`

需要打开下面这个选项：
`Kernel hacking —> Compile-time checks and compiler options —> [ ] Compile the kernel with debug info`（对应 `.config` 文件中的 `CONFIG_DEBUG_INFO`）。

否则虽然可以进行汇编级别的调试，但是因为缺少 dwarf 信息而不能与源码对应起来。

2. `make -j$(nproc)`

多线程编译内核，最终会得到 `vmlinux` 这个可执行的内核。

## 2. rootfs 制作

有了内核还不够，加下来需要制作文件系统，虽然可以通过 debootstrap 手动创建，但稍嫌麻烦。因此这里用到了 buildroot 这个项目，可以通过 `git clone git://git.buildroot.net/buildroot` 下载到本地，然后执行如下操作

1. `make menuconfig`，并进行如下操作

- Target options -> Target architecture, select x86_64
- Filesystem images -> ext2/3/4 root filesystem; then choose the ext4 variant
- Target packages -> Network applications -> openssh [*], or any tools you need

2. `make -j$(nproc)`

编译过程会下载很多文件，请保证与 github 连接畅通，或者更换下载过程中 wget 使用的 github 链接。

## 3. qemu 虚拟机网卡配置

上一步结束之后，应该已经得到了 vmlinux 和 rootfs.ext4 这两个文件，为了保证 qemu 启动之后，虚拟机可以连接外网，需要分别在宿主机和 qemu 虚拟机上分配网络环境。

这里的思路是，创建一个 tap 设备并挂载到网桥上，qemu 启动时将指定该 tap 设备虚拟为 eth0 网卡，同时宿主机中打开 ip 转发功能，通过 iptables 进行 NAT 地址转换。

1. apt install bridge-utils && apt install uml-utilities
2. 
   - brctl addbr br0
   - ifconfig br0 192.168.2.1
   - ip tuntap add tap0 mode tap
   - ifconfig tap0 0.0.0.0 promisc up
   - brctl addif br0 tap0
   - echo 1 > /proc/sys/net/ipv4/ip_forward
   - iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

    注意到我们没有将物理网卡也添加到 bridge 中，所以需要通过 iptables 做一层 proxy 才能访问外网
3. qemu-system-x86_64 -kernel ./arch/x86_64/boot/bzImage \
    -nographic \
    -hda /path/to/buildroot/output/image/rootfs.ext4 \
    -append "nokaslr root=/dev/sda console=ttyS0"  \
    -smp 4 \
    -netdev tap,id=mynet0,ifname=tap0,script=no,downscript=no -device e1000,netdev=mynet0,mac=52:55:00:d1:55:01 \
    -s -S

    这里要注意的是，如果不指定 nokaslr，会由于内核地址随机初始化这个特性，导致无法命中断点

4. 在内核源码目录，执行 `gdb ./vmlinux`，在 gdb 命令行中，执行 `target remote :1234` `continue` 即可
   
5. gdb 中放行，qemu 会初始化 vm，登录之后，需要执行 `ifconfig eth0 192.168.2.2 && route add default gw 192.168.2.1` 来配置网卡的 ip 地址，此时已经可以访问 host 和通过 ip 访问外网

6. 拷贝宿主机 `/etc/resolv.conf` 到 qemu 虚拟机中，尝试 `ping baidu.com`

7. 在 vm 中，修改 `/etc/ssh/sshd_config`，将 `PermitRootLogin` 设置为 `yes`，将 `PermitEmptyPasswords` 设置为 `yes`，然后执行 `/etc/init.d/S50sshd restart` 来重启 sshd 服务；然后就可以在宿主机中，执行 `ssh root@192.168.2.2` 远程连接虚拟机了，接下来就可以随心所欲进行调试了（其实调试和最后这步远程连接关系不大）。