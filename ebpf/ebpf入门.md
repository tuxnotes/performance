# ebpf入门
声明本文整理于网络，仅用于跟人学习。
## 入门介绍1
本部分整理于知乎，连接：https://zhuanlan.zhihu.com/p/108060469
### 1 bpftrace介绍
bpftrace是基于ebpf内核vm货站出来的trace工具。学习资料可参考仓库地址：https://link.zhihu.com/?target=https%3A//github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md
eBPF是BPF的扩展，核心是一个vm，可以执行自己的指令集合，抽象上基于一组map来进行存储数据，提供了安全性的检查和限制(比如bounded-loop)避免内核态的instrument带来低dissaster.
关于bpf的工作原理可参考man手册：`man 2 bpf`，或者网页https://link.zhihu.com/?target=http%3A//man7.org/linux/man-pages/man2/bpf.2.html

BPF目前有2个主要的maintainer，其中之一是cilium公司的Founder主导，他们基于BPF做了网络层L3/L4的安全控制，集成在编排服务中。详情见[文档](https://docs.cilium.io/en/stable/bpf/#bpf-architecture)

bcc是bpf compiler collection,是一个不断丰富的基于eBPF的工具结合，bcc对Linux内核的各个子系统(file, block, network, cpu, mem, security)都有一系列的工具支持。bcc基本上用Python编写，通过BPF syscall注入字节码给eBPF内核虚拟机。

基于BPF的功能在eBPF出现后不断进行丰富，最近的tcp流控制算法就改进为bpf中运行。作者抽象了一个framework，让人们看到，BPF可以做很多除了最初出发点的网络filter之外的能力，具备了通用处理能力的运行时，未来Linux可能时微内核+BPF运行时上运行其他能力的架构。应用中的比如BPF可以做网络交换机。

bpftrace是类似system-tap,perf一样的trace工具，定位在及时交互的进行trace。

### 2 优势
- 在内核内过滤，不像perf那样拉出到用户态再过滤，效率高，overhead低
- 安全。与system-tap对比
- 好用。类似awk+c的语法，几乎所有内核函数都能trace，未来提供类型信息，可以不依赖内核源码，对系统依赖小，避免各种不兼容，跑不起来。

### 3 tracepoint分类
- 内核tracepoint，这是内核开发者埋好的点，expose到`/sys/kernel/debug/tracing`目录，也可以被trace使用
- 内核函数kprobe+kretprobe，就是对函数入口和函数返回进行hook，实现方式是对应入口出指令替换，CPU执行到此处跳到bpf代码执行，然后再返回带原代码处执行。kretprobe实现稍微复杂一些
- 用户态uprobe+uretprobe，用户态对函数进行hook，实现与kprobe类似的功能
- USDT：用户态的动态tracepoint，需要用户态程序自己埋点tracepoint
- perf-event，PMU等硬件的性能指标

更多请参考：https://github.com/iovisor/bpftrace/blob/master/README.md#probe-types

### 4 注意事项
- trace操作注意overhead，一般对于trace都是线上系统进行，注意对workload的影响，执行前想清楚
- 先用coarse工具和方法确定范围，再用bpftrace进行精确定位。工具是工具，思路最重要。
- 了解系统比了解工具更重要

### 5 issues
- 找不到类型，一般是bpftrace内部调用clang编译的时候，没有include到头文件之类的，可在bpftrace要执行的代码前面自己写类似c的：`#include <sys/socket.h>`。可通过-l指定include寻找目录
- 如何查看tracepoint的参数？或者`bpftrace -vl syscall:sys_enter_io_uring_enter`

```bash
# 这里有字段，在bpftrace程序中通过 args->xxx 可引用
cat /sys/kernel/debug/traceing/event/syscall/sys_enter_io_uring_enter/format 
```
## 入门介绍2
**eBPF 在网易轻舟云原生的应用实践**
https://www.infoq.cn/article/ovcvwqijzta7jlexgdoc
