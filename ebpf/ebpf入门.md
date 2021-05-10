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
eBPF是Linux内核近几年最引人注目的特性之一，通过一个内核内置的字节码虚拟机，完成数据包过滤、调用栈跟踪、耗时统计、热点分析等高级功能，是Linux系统和Linux应用的功能/性能分析利器。本文将介绍eBPF的技术特点，及eBPF在网易杭研轻舟系统探测和网络性能优化方面的应用。

### 1 技术浅析-eBPF好在哪里
eBPF是Linux kernel 3.15中引入的全新设计，将原先的BPF发展成一个指令集更复杂、应用范围更广的"内核虚拟机".

eBPF支持在用户态将C语音编写的一小段"内核代码"注入带内核中运行，注入时要先用llvm编译得到使用BPF指令集的ELF文件，然后从ELF文件中解析出可以注入内核的部分，最后用bpf_load_program()方法完成注入。用户态程序和注入到内核中的程序通过共用一个位于内核中的eBPF MAP实现通信。为了防止注入的代码导致内核崩溃，eBPF会对注入的代码进行严格检查，拒绝不合格的代码的注入。

![](https://static001.infoq.cn/resource/image/d6/fe/d6ffc0eyy17785f654037cbc989931fe.jpg)

#### 1.1 eBPF的技术优势
eBPF具备一些非常棒的特性，使得我们在内核层面的监控变得更加便捷、搞笑、并且非常安全：
- 平台无关，有JIT负责将BPF代码翻译成最终的处理器指令
- 内核无关，辅助函数(Helper functions)使得BPF能够通过一组内核定义的函数调用(Function call)来从内核中查询数据，或将数据推送到内核。不同类型的BPF程序能够使用的辅助函数可能是不同的
- 安全检查，与内核模块不同，BPF程序会被一个位于内核中的校验器(in-kernel verifier)进行校验，以且薄它们不会造成内核崩溃、程序永远能够终止等
- 方便升级，对于网络场景(例如TC和XDP)，BPF程序可以在无需重启内核、系统服务或容器的情况下实现院子更新，并不会导致网络中断
- 通信方式，eBPF MAP，跟Ftrace提供的ring buffer相比就是数据常驻内存，不会读取之后就消失
- 事件驱动，BPF程序在内核中的执行总是事件驱动的

#### 1.2 eBPF和其他trace工具的对比
在eBPF出现之前，Linux已经存在多种成熟的trace机制和相应的上层工具：

![](https://static001.infoq.cn/resource/image/29/e7/29e2b0044905186a766fe1243e6c71e7.jpg)

如上图所示，eBPF借助perf_event和trace_event几乎支持对目前所有已知trace功能的支持，唯一与传统trace工具不同的是，attach到每个探测点的probe函数是运行在JIT虚拟机上的eBPF程序，具备上面提到平台无关、内核无关、安全等一些列更优的特性。

### 2 热点追踪--eBPF有哪些应用场景
eBPF应用主要分为两个场景，一是系统探测；二是网络方面性能优化(包括数据面传输和规则匹配等场景)

#### 2.1 在系统探测方面
在国外，Google已经开始用BPF做profiling，找出在分布式系统中应用消耗多少CPU。而且，他们也开始将BPF的使用范围扩展到流量优化和网络安全。

在国内，字节跳动利用eBPF技术开发了一款名为sysprobe的监控工具，用来定位分析线上业务的性能瓶颈等。

基于eBPF开源的探测工具推荐两个：
- BCC，是eBPF的一个外围工具集，使得编写BPF代码--->编译成字节码--->注入内核--->获取结果--->展示整个过程更加便捷。此外BCC项目还有一个非常肺腑的探测工具集合
- bpftrace，可以理解为是eBPF的高级追踪语音，动态的翻译这些语言，生产eBPF后端程序并通过BCC工具实现和Linux BPF系统进行交互。

#### 2.2 在网络性能优化方面
Facebook用BPF重写了他们的大部分基础设施。例如使用BPF替换了iptables和network filter，并且Facebook基本上已经将他们的负载均衡器从IPVS换成了BPF，此外他们还将BPF用在流量优化(traffic optimization)、网络安全等方面。

Redhat则正在开发一个叫bpffilter的上游项目，将来会替换掉内核里的iptables，也就是说，内核里基于iptables做包过滤的功能，以后都会用BPF替换。

国内方面腾讯云也对外宣称利用eBPF替换conntrack模块是k8s service短链接性能提升40%

### 3 牛刀小试--eBPF在网易轻舟的应用及落地
eBPF在网易轻舟的应用也分为系统探测和网络性能优化两个方面。

#### 3.1 系统探测
我们现有监控工具只能监控内核主动暴露的数据，存在监控盲点。通过修改内核或开发内核模块的方式进行监控，上线周期长、风险高，针对不同版本内核的适配费时费力还容易出错。所以我们在开源项目ebpf_exporter的基础上完成了ebpf监控子系统的开发。

**eBPF 监控子系统架构**

![](https://static001.infoq.cn/resource/image/5b/8f/5bdf40e1cb065babffde38cb36ab778f.jpg)

eBPF 监控子系统采用容器化部署，支持通过 K8s 部署和管理，监控数据通过 Prometheus 进行持久化存储并通过 Grafana 进行展示。eBPF 监控子系统具备如下特性：

- 平台无关且内核无关；
- 安全检查以保证绝对安全；
- 可对内核进行任意监控；
- 支持 pid/container/pod 级别的细粒度监控；
- 支持监控项动态开启、关闭；
- 支持监控项在线升级；
- 提供统一的监控项开发接口；
- 提供 metrics/json 格式的监控数据及统一的数据获取接口;
- 支持 K8s 部署、管理；

**关于 eBPF“内核无关”特性的说明**


