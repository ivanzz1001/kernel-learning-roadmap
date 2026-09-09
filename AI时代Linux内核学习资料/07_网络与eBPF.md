# 第 7 章：网络栈与 eBPF 可观测性

## 本章目标

用 network namespace 构造完全受控的网络，理解一次 TCP 收发的主要层次；再用 BPF/libbpf 开发一个有边界、有丢失统计、能与传统工具交叉验证的延迟观测器。

## 第 29 周：namespace、veth 与路由

### 两节点实验拓扑

```text
namespace A                     namespace B
10.10.0.1/24 -- veth-a ===== veth-b -- 10.10.0.2/24
```

创建前先确认名称未被使用；删除 namespace 会移除其内接口：

```bash
sudo ip netns add ns-a
sudo ip netns add ns-b
sudo ip link add veth-a type veth peer name veth-b
sudo ip link set veth-a netns ns-a
sudo ip link set veth-b netns ns-b
sudo ip -n ns-a addr add 10.10.0.1/24 dev veth-a
sudo ip -n ns-b addr add 10.10.0.2/24 dev veth-b
sudo ip -n ns-a link set lo up
sudo ip -n ns-b link set lo up
sudo ip -n ns-a link set veth-a up
sudo ip -n ns-b link set veth-b up
sudo ip netns exec ns-a ping -c 3 10.10.0.2
```

清理前用 `ip netns list` 核对目标，再删除 `ns-a/ns-b`。这些命令改变主机网络状态，优先在实验 VM 中执行。

### 观察矩阵

- `ip -n ns-a addr/link/route/neigh`：控制面状态。
- `tcpdump -ni veth-a`：线上包。
- socket syscall trace：应用请求。
- 网络 tracepoint/ftrace：内核路径。
- `/proc/net/*` 或 `ss -tin`：连接状态与统计。

从 ping 开始，再做 UDP echo，最后 TCP。每增加一层都保留可工作的基线。

## 第 30 周：TCP 收发路径

### 分层地图

发送侧大致经历：

```text
用户 send/write
  → socket 层
  → TCP
  → IP/路由/netfilter
  → qdisc/device
  → 驱动/虚拟设备
```

接收侧大致反向进入，涉及 NAPI/softirq、协议分发、socket receive queue，再唤醒用户进程。具体函数随配置、offload 和版本变化。

### 实验：一次小连接

1. ns-b 启动只接受一次连接的 server。
2. ns-a connect，发送带 sequence 的 10 条消息，等待 echo。
3. 同时保存 tcpdump、syscall trace、sched wakeup/switch 和少量网络 tracepoint。
4. 用四个时间轴关联：应用、socket、包、调度。
5. 分别加入 50ms netem delay 和少量 loss，观察 retransmission 与应用延迟。

不要把不同时间源/CPU 的时间戳未经校准直接相减。先确认 trace_clock 和抓包时间源。

### sk_buff 阅读任务

不要背全部字段。只追踪一个包在关键层次的：data/head/tail、长度、协议元数据、设备、所有权与释放。重点观察 clone、linear/non-linear data 和 checksum/offload 会怎样改变直觉。

## 第 31 周：BPF 基础与 verifier

### BPF 程序的生命周期

```text
C 源码
  → clang 编译 BPF ELF
  → libbpf 解析、重定位、加载
  → verifier 证明程序满足安全约束
  → attach 到 hook
  → 事件触发执行
  → map/ring buffer 向用户态输出
```

BTF 描述类型，CO-RE 借助目标内核 BTF 进行字段重定位。CO-RE 提高可移植性，但不意味着任意内核/配置都能运行；程序类型、helper、字段和权限仍需探测。

### 从 tracepoint 开始

首个项目选择稳定 tracepoint，不先依赖 kprobe 内部函数。实现：按 TGID 统计 syscall 次数，map 存聚合数据，用户态每秒读取并显示。

### verifier 练习

准备两个被拒案例：

- 未经边界证明访问 packet/ring buffer 数据；
- 可能为 NULL 的 map lookup 结果直接解引用。

逐行解释 verifier 的寄存器状态和控制流。修复的关键是让所有路径上的范围/空值可被静态证明，而不是随机加判断。

### 输出通道选择

- map polling：简单，适合聚合指标。
- perf buffer/ring buffer：适合事件流；必须处理满、丢事件和用户态消费速度。
- `bpf_printk`：仅调试，开销和限制不适合正式输出。

## 第 32 周：libbpf CO-RE 延迟工具

### 项目选题

任选一个，第一版只支持一个目标内核/架构，之后再谈可移植：

- syscall duration histogram；
- wakeup-to-run latency；
- TCP connect latency；
- block I/O request latency；
- memory reclaim stall。

### 设计要求

1. 入口事件保存 key 与开始时间。
2. 出口事件取回并删除状态。
3. key 必须处理嵌套、PID 重用或并发语义。
4. 丢失入口/出口、map 满、负 duration 都要计数。
5. 用户态支持 PID/cgroup 过滤、持续时间、输出周期。
6. 输出直方图或 p50/p95/p99，不只平均值。
7. README 写明 CAP/权限、内核配置、BTF 要求和已知限制。

### 正确性验证

- 用可控 sleep/delay 负载制造已知数量级。
- 与 tracefs 原始事件抽样对照。
- 与 `perf` 或用户态时间测量交叉验证。
- 故意降低 map 容量或停止消费者，验证丢失计数。
- 多 CPU、多线程、进程退出时压力运行。

### 性能验证

比较启用/禁用工具时负载吞吐和延迟。固定 CPU、运行时长、预热和采样频率。不要声称“零开销”；报告检测下限和实验噪声。

## 网络问题的证据层次

| 层次 | 能回答 | 不能单独回答 |
|---|---|---|
| 应用日志 | 请求语义、端到端结果 | 内核何处耗时 |
| syscall trace | API 调用与阻塞时间 | 包是否上网线 |
| socket/TCP trace | 协议状态与队列 | 设备真实发送完成 |
| tcpdump | 观察点经过的包 | 所有内核内部状态 |
| driver/硬件统计 | 丢包、队列、offload | 应用级请求因果 |

诊断时至少组合相邻两层，不用单一工具解释整条链路。

## AI 提示词

```text
我要在 Linux <commit> 用 libbpf CO-RE 观测 <延迟>。
请先给设计审查：选择的 hook 是否稳定、key 如何处理并发/嵌套/PID 重用、map 生命周期、丢事件、时间源、过滤位置、verifier 风险和开销。
然后给最小验证矩阵。不要假定目标内核存在某 helper/tracepoint；列出探测命令。
```

## 本章验收

- [ ] 能独立搭建并清理两 namespace 网络。
- [ ] 能用多层证据解释一次 TCP 收发，而不把抓包等同全部事实。
- [ ] 能解释 verifier 拒绝案例中的范围或空值问题。
- [ ] BPF 工具有丢失统计、过滤、可复现负载和开销对比。

## 官方延伸阅读

- [Linux networking documentation](https://docs.kernel.org/networking/index.html)
- [BPF documentation](https://docs.kernel.org/bpf/index.html)
- [libbpf overview](https://docs.kernel.org/bpf/libbpf/libbpf_overview.html)
- [BPF testing and debugging](https://docs.kernel.org/bpf/bpf_devel_QA.html)

