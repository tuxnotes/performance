# BCC安装
仓库地址：https://github.com/iovisor/bcc/blob/master/INSTALL.md
## 内核要求
通常要求内核版本在4.1及以上版本，且内核编译时有如下设置：
```
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
# [optional, for tc filters]
CONFIG_NET_CLS_BPF=m
# [optional, for tc actions]
CONFIG_NET_ACT_BPF=m
CONFIG_BPF_JIT=y
# [for Linux kernel versions 4.1 through 4.6]
CONFIG_HAVE_BPF_JIT=y
# [for Linux kernel versions 4.7 and later]
CONFIG_HAVE_EBPF_JIT=y
# [optional, for kprobes]
CONFIG_BPF_EVENTS=y
# Need kernel headers through /sys/kernel/kheaders.tar.xz
CONFIG_IKHEADERS=y
```
内核编译的flag设置可以通过以下两个文件检查：
```
/proc/config.gz
/boot/config-<kernel-version>
```
## 安装
考虑到不同的Linux发行版，如果是处于学习的目的，推荐采用Ubuntu或Debian等发行版，因为其内核版本较高。而centos7因为内核版本的问题，需要升级内核，且bcc的版本较低。这里以Debian10为例进行安装，详细信息参考官方文档。安装命令很简单，如下:
```bash
# apt install bpfcc-tools
```
centos从7.6版本开始，官方仓库就内置了bcc，安装命令如下：
```bash
# yum -y install bcc
```
centos7通过上述方法安装完成后，命令工具位于`/usr/share/bcc/tools`,而Debian10通过apt安装后可执行的命令工具位于目录`/usr/sbin`下。且可执行文件的名称不同：
- Debian10：`/usr/sbin/execsnoop-bpfcc`
- Centos7: `/usr/share/bcc/tools/execsnoop`