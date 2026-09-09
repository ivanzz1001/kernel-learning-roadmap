# AI 时代 Linux 内核学习资料

这是一套与 [《AI 时代 Linux 内核学习路线图》](../AI时代Linux内核学习路线图.md) 配套的实践教材。目标不是“读完”，而是在约 40 周内形成完整工程闭环：**提出问题 → 定位源码 → 动态观察 → 修改代码 → 测试验证 → 清晰表达**。

## 适用基础与学习周期

- 已能在 Linux 命令行中编辑、编译和运行普通程序。
- 至少掌握一门编程语言；C 基础薄弱可从第 1 章开始补齐。
- 标准节奏为每周 10～12 小时，共 40 周。
- 每周 5～6 小时时，将每个单元拆成两周。
- 实验默认使用 x86-64 Linux 开发机和 QEMU 虚拟机。

## 资料结构

| 文件 | 对应周次 | 核心产出 |
|---|---:|---|
| [00 实验环境](00_实验环境.md) | 第 0 周 | 可重复构建、启动、调试的 QEMU 内核实验室 |
| [01 C 与系统编程](01_C与系统编程.md) | 第 1～4 周 | ELF 分析、进程实验、mmap 实验、mini shell |
| [02 启动与源码导航](02_启动与源码导航.md) | 第 5～8 周 | 启动地图、系统调用路径、最小配置 |
| [03 模块与字符设备](03_模块与字符设备.md) | 第 9～13 周 | 可并发、可 poll、可测试的字符设备 |
| [04 调度、并发与中断](04_调度并发与中断.md) | 第 14～18 周 | 调度延迟报告、死锁复现与修复 |
| [05 内存管理](05_内存管理.md) | 第 19～23 周 | 缺页/回收观测器、OOM 分析报告 |
| [06 VFS、文件系统与块 I/O](06_VFS文件系统与块IO.md) | 第 24～28 周 | I/O 路径报告、最小文件系统 |
| [07 网络与 eBPF](07_网络与eBPF.md) | 第 29～32 周 | 隔离网络实验、内核延迟诊断工具 |
| [08 调试、测试、安全与性能](08_调试测试安全与性能.md) | 第 33～36 周 | 五类故障档案和回归测试 |
| [09 上游协作与毕业项目](09_上游协作与毕业项目.md) | 第 37～40 周 | 可审查 patch series 和毕业项目报告 |
| [10 AI 学习工作流](10_AI学习工作流.md) | 全程 | 可核验的 AI 提问、审查和复盘方法 |
| [实验报告模板](templates/实验报告模板.md) | 全程 | 可复现的实验记录 |
| [周报模板](templates/周报模板.md) | 全程 | 每周知识与证据闭环 |

## 40 周课程表

| 周 | 核心问题 | 必做实验 | 验收证据 |
|---:|---|---|---|
| 0 | 如何得到可重复的实验环境？ | 编译 LTS 内核并进入 QEMU shell | 构建日志、`uname -a`、启动脚本 |
| 1 | C 对象如何落到内存？ | 指针/结构体/位域/宏实验 | GDB 内存与反汇编截图或文本 |
| 2 | 源码如何变成 ELF？ | `readelf/nm/objdump` 解剖程序 | 符号、section、重定位说明 |
| 3 | 进程和地址空间如何工作？ | `fork/exec/wait/mmap/COW` | page fault 与 maps 数据 |
| 4 | 如何写可靠的系统程序？ | mini shell + ASan/TSan + Git bisect | 测试脚本和缺陷定位记录 |
| 5 | 内核从哪里开始执行？ | 跟踪 `start_kernel` | 启动时间线 |
| 6 | 系统调用怎样穿越权限边界？ | 跟踪 `getpid` 或 `openat` | 静态调用链 + 动态证据 |
| 7 | Kconfig/Kbuild 如何控制产物？ | 新增配置项和 built-in 代码 | `y/n` 两组构建结果 |
| 8 | 如何高效阅读陌生源码？ | 建立一个子系统源码地图 | 入口、对象、锁、生命周期 |
| 9 | 模块如何装载和卸载？ | hello 模块、参数、日志 | 反复装卸记录 |
| 10 | 字符设备如何连接用户态？ | read/write/ioctl | 用户态测试结果 |
| 11 | 阻塞 I/O 如何唤醒？ | wait queue + poll | 阻塞/非阻塞行为对照 |
| 12 | 多进程访问如何保证正确？ | mutex/spinlock 对照、压力测试 | 竞态复现与修复 |
| 13 | 设备模型如何组织对象？ | class/sysfs/platform device | sysfs 树与生命周期图 |
| 14 | 进程何时变为可运行？ | sched wakeup trace | wakeup 到 switch 的事件链 |
| 15 | 调度器怎样选择任务？ | 绑核、nice、负载对照 | 延迟分布而非单一平均值 |
| 16 | 不同执行上下文能做什么？ | hrtimer + workqueue | 上下文与允许操作表 |
| 17 | 锁为什么会死锁？ | lockdep 锁顺序反转 | 报告解读与修复 patch |
| 18 | RCU 解决什么问题？ | 读多写少数据结构实验 | grace period 与生命周期说明 |
| 19 | 一次缺页发生了什么？ | 匿名页/file-backed page fault | fault trace |
| 20 | 物理页如何分配？ | buddy 与 slab 观测 | `/proc/buddyinfo`、slab 数据 |
| 21 | 内存不足时如何回收？ | 受控 memory pressure | reclaim/swap 指标 |
| 22 | OOM 为什么选择这个进程？ | VM/cgroup 内 OOM | 完整 OOM 日志解读 |
| 23 | 内存错误如何被发现？ | KASAN/KFENCE/kmemleak | 错误分配/释放栈 |
| 24 | 路径名如何变成 inode？ | `openat` 路径 trace | VFS 对象关系图 |
| 25 | 读写何时进入页缓存？ | buffered/direct/mmap I/O | 三种方式对比 |
| 26 | 脏页何时落盘？ | writeback trace | 数据持久化边界说明 |
| 27 | bio/request 如何到设备？ | loop 设备块 I/O trace | 块层事件链 |
| 28 | 文件系统最小实现是什么？ | 只读 pseudo filesystem | mount/read/unmount 测试 |
| 29 | 网络命名空间如何隔离？ | netns + veth + route | 两节点拓扑与抓包 |
| 30 | TCP 数据如何穿过协议栈？ | connect/send/receive trace | 收发关键对象与函数 |
| 31 | BPF verifier 在证明什么？ | 一个成功和一个被拒程序 | verifier 日志解释 |
| 32 | 如何构建低开销观测器？ | libbpf CO-RE 延迟直方图 | 工具、README、测试负载 |
| 33 | 如何停在内核启动代码？ | QEMU `-s -S` + GDB | 断点、变量、调用栈 |
| 34 | 内核测试如何分层？ | KUnit + kselftest | 修复前失败/修复后通过 |
| 35 | 如何从 oops 找到根因？ | panic/UAF/deadlock 档案 | 五类故障证据链 |
| 36 | 如何证明性能变化？ | 固定环境 A/B benchmark | 分位数、方差、配置 |
| 37 | 怎样选择有价值的小问题？ | 阅读 MAINTAINERS 和近期历史 | 候选问题说明 |
| 38 | patch 如何拆分？ | 形成 3～8 个 commits | 每个 commit 单一目的 |
| 39 | 上游前如何自审？ | checkpatch/build/test/sparse | 提交检查单 |
| 40 | 如何完成技术交付？ | 整理毕业项目与 patch series | 陌生人可复现的文档 |

## 每周使用方法

1. 周初只选一个“核心问题”，把自己的初始答案写下来。
2. 阅读本章“概念地图”，再进入官方文档和源码，不先搜零散博客。
3. 先跑基线，再做改动；基线和改动使用相同 `.config`、启动参数和负载。
4. 所有实验均填写[实验报告模板](templates/实验报告模板.md)。
5. AI 只参与提出假设、解释局部代码和审查证据，结论以本地源码和运行结果为准。
6. 周末用[周报模板](templates/周报模板.md)做一次口述复盘：不看资料讲清楚，才算掌握。

## 学习仓库建议结构

```text
kernel-lab/
├── linux/             # 固定版本的内核源码
├── build/             # O= 输出，禁止与源码混放
├── rootfs/            # BusyBox initramfs
├── scripts/           # build/run/debug/test
├── modules/           # 自己编写的模块
├── userspace/         # 测试程序、负载生成器
├── experiments/       # 每个实验一目录
├── patches/           # format-patch 输出
└── notes/             # 源码地图、周报、故障档案
```

## 总验收规则

每个阶段至少满足以下六条，才进入下一阶段：

- **可重复**：另一台干净环境能按 README 复现。
- **可观察**：结论有日志、trace、计数器、调用栈或测试结果支持。
- **可解释**：能说明执行上下文、对象生命周期、关键锁和失败路径。
- **可回归**：至少有一个自动化测试。
- **可审查**：改动被拆成逻辑完整的小 commits。
- **可质疑**：明确记录实验限制、替代解释和未解决问题。

## 权威资料入口

- [Linux Kernel Documentation](https://docs.kernel.org/)
- [Kernel subsystem documentation](https://docs.kernel.org/subsystem-apis.html)
- [Kernel development process](https://docs.kernel.org/process/development-process.html)
- [Submitting patches](https://docs.kernel.org/process/submitting-patches.html)
- [QEMU GDB usage](https://qemu.readthedocs.io/en/master/system/gdb.html)
- [KUnit](https://docs.kernel.org/dev-tools/kunit/start.html)
- [kselftest](https://docs.kernel.org/dev-tools/kselftest.html)
- [BPF/libbpf](https://docs.kernel.org/bpf/libbpf/index.html)
- [syzkaller Linux setup](https://github.com/google/syzkaller/blob/master/docs/linux/setup.md)

