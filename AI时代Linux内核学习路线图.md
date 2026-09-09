# AI 时代 Linux 内核学习路线图

> 目标：用约 9 个月建立“能编译、能观察、能调试、能修改、能测试、能提交”的 Linux 内核能力，而不是只记概念。  
> 建议投入：每周 10～12 小时。若每周只能投入 5～6 小时，将每个阶段时长翻倍。  
> 适合对象：会使用 Linux，掌握一门编程语言，希望系统学习内核的开发者。

---

## 一、学习原则

### 1. 双轨内核版本

- **实验主线**：选择 kernel.org 当前仍活跃的 LTS 版本，固定小版本，保存 `.config` 和启动脚本，保证实验可重复。
- **阅读主线**：保留 Linus 主线仓库，用于查看最新代码、提交历史和社区变化。
- 不要同时追多个发行版内核；发行版补丁会增加初学阶段的认知负担。

截至 2026 年，6.18、6.12、6.6、6.1 等仍属于活跃 longterm 分支，具体状态以 [kernel.org Active Kernel Releases](https://www.kernel.org/releases.html) 为准。

### 2. 每个知识点执行“六步闭环”

1. **提问**：它解决什么问题？
2. **定位**：入口函数、核心结构体、关键文件在哪里？
3. **追踪**：用日志、trace、perf 或 GDB 观察一次真实执行。
4. **修改**：做一个最小、可撤销的代码变化。
5. **验证**：用测试或可观测指标证明修改生效且没有破坏其他行为。
6. **表达**：写一页实验报告和一条合格的 commit message。

### 3. AI 是副驾驶，不是答案生成器

AI 适合做：解释局部代码、生成检索线索、比较调用路径、设计实验、补测试、审查补丁、整理日志。  
AI 不应替代：源码核对、运行验证、并发正确性判断、API 版本确认、补丁署名与最终责任。

每次使用 AI 都要求它给出以下四项：

- 结论对应的文件、符号或 commit；
- 不确定点和适用内核版本；
- 可执行的验证方法；
- 至少一种反例或失败路径。

---

## 二、先搭建贯穿全程的实验室（第 0 周）

### 推荐环境

- 主机：Linux x86-64，推荐 Ubuntu/Debian/Fedora；Windows/macOS 用户使用一台 Linux 主机或 Linux VM 作为开发机。
- 客体：QEMU + BusyBox initramfs；有 KVM 时启用加速，无 KVM 也可使用 TCG。
- 工具：Git、GCC/Clang、make、binutils、GDB、QEMU、bc、bison、flex、pahole、cpio、ncurses 开发包。
- 编辑器：支持 C、compile_commands.json、跳转和调用层级即可；不要把时间耗在插件配置上。

### 建议目录

```text
kernel-lab/
├── linux/                 # 内核源码
├── build/                 # O= 输出目录
├── rootfs/                # initramfs 根目录
├── scripts/               # build/run/debug 脚本
├── modules/               # 练习模块
├── experiments/           # 每个实验一个目录
└── notes/                 # 源码地图、故障记录、周报
```

### 最小构建闭环

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux
git checkout <选定的-LTS-标签>
make O=../build x86_64_defconfig
make O=../build -j"$(nproc)"
```

QEMU 启动脚本至少保留串口输出，并让 panic 后停住：

```bash
qemu-system-x86_64 \
  -m 2G -smp 2 -nographic \
  -kernel build/arch/x86/boot/bzImage \
  -initrd rootfs.cpio.gz \
  -append "console=ttyS0 rdinit=/init nokaslr panic=-1"
```

### 第 0 周验收

- [ ] 从零开始在 30 分钟内完成一次内核编译。
- [ ] QEMU 能进入 shell，`uname -a` 显示自己编译的版本。
- [ ] 修改 `LOCALVERSION`、重新编译并确认变化。
- [ ] 保存 `build.sh`、`run.sh`、`.config` 和实验说明。
- [ ] 能从 `dmesg` 找到启动命令行、CPU、内存和 init 进程信息。

> 安全规则：所有可能导致崩溃、死锁、内存破坏和 fuzzing 的练习先放在可快照、无重要数据、网络隔离的 VM 中。

---

## 三、9 个月循序渐进路线

## 阶段 1：Linux 与 C 的系统编程地基（第 1～4 周）

### 学习主题

- C：指针、数组与字符串、结构体、位运算、宏、函数指针、生命周期、未定义行为。
- 编译链接：预处理、ELF、符号表、静态/动态链接、反汇编、DWARF 调试信息。
- Linux 用户态：进程、线程、文件描述符、虚拟内存、信号、管道、socket、`mmap`、`ioctl`。
- Git：branch、rebase、bisect、format-patch、send-email 的概念与基本操作。
- 基础数据结构：链表、哈希、红黑树；基础并发：原子操作、锁、内存可见性。

### 实操任务

1. 写 `minish`：支持启动程序、重定向、管道和信号处理。
2. 写 `mmap-lab`：比较匿名映射、文件映射、缺页和写时复制。
3. 用 `readelf`、`nm`、`objdump`、GDB 解剖一个 C 程序。
4. 制造一次 use-after-free 和数据竞争，分别用 AddressSanitizer、ThreadSanitizer 定位。
5. 对一个小型 C 仓库执行 `git bisect`，自动找到引入错误的提交。

### 验收标准

- [ ] 能解释 `fork → execve → wait` 的职责边界。
- [ ] 能从汇编指出参数传递、栈帧和返回值。
- [ ] 能区分虚拟地址、物理地址、文件偏移和页缓存。
- [ ] 能写出有错误处理、资源回收和测试的 C 程序。

---

## 阶段 2：认识内核全貌与源码导航（第 5～8 周）

### 学习主题

- 用户态/内核态、系统调用、异常、中断、上下文切换。
- 启动路径：固件/bootloader → 解压 → 架构初始化 → `start_kernel` → init。
- 源码目录：`arch/`、`kernel/`、`mm/`、`fs/`、`net/`、`drivers/`、`include/`、`tools/`。
- Kbuild、Kconfig、built-in 与 module、内核参数和 sysctl。
- `task_struct`、`mm_struct`、`file`、`inode`、`dentry` 等核心对象只先建立“关系地图”，不要背字段。

### 实操任务

1. 为 `getpid()` 画出“用户库 → 指令入口 → 系统调用处理 → 返回”的调用路径。
2. 从 `start_kernel()` 选择 10 个关键调用，用 Git 查每个函数最近的演进历史。
3. 编译最小内核：从 `tinyconfig` 或 `defconfig` 出发，逐项补齐到能启动 BusyBox。
4. 修改一个启动日志并观察；再新增一个只读 kernel parameter。
5. 使用 `scripts/config` 自动生成可复现实验配置。

### 阶段项目：内核启动地图

产出一份包含时间线、关键函数、源码位置、对应日志的启动报告。每个箭头必须能通过源码或 trace 证实。

### 验收标准

- [ ] 看到一个内核符号时，能用 `rg`、Git 和文档在 10 分钟内定位定义、调用者、配置依赖。
- [ ] 能解释为什么 `vmlinux`、`bzImage`、`.ko` 用途不同。
- [ ] 能自行增加 Kconfig 项并证明 `y/m/n` 行为正确。

---

## 阶段 3：模块、驱动模型与字符设备（第 9～13 周）

### 学习主题

- 模块装载、符号导出、参数、引用计数、taint。
- 字符设备、`file_operations`、主次设备号、`copy_to_user/copy_from_user`。
- 设备模型：bus、device、driver、class、sysfs、udev。
- platform driver、设备树基本概念；无嵌入式目标时先用 QEMU/虚拟设备。
- 错误码、资源管理、锁与用户输入边界。

### 实操任务

1. `hello.ko`：参数、日志级别、加载/卸载。
2. `counter.ko`：实现 open/read/write/ioctl/poll，支持多个进程访问。
3. 为模块增加 sysfs 属性，并处理并发读写。
4. 故意植入越界、泄漏和睡眠上下文错误，用 KASAN、kmemleak、lockdep 分别发现。
5. 写用户态测试程序和一键加载/测试/卸载脚本。

### 阶段项目：可测试字符设备

设备应至少具备：并发安全、阻塞/非阻塞 I/O、poll、统计信息、错误注入、用户态回归测试和设计文档。

### 验收标准

- [ ] 所有用户指针都通过正确的 uaccess API 访问。
- [ ] 每次申请资源都有清晰释放路径，模块可反复装卸 100 次。
- [ ] 能解释函数是否允许睡眠、运行在哪种上下文、用哪类锁以及原因。

---

## 阶段 4：进程、调度、时间与同步（第 14～18 周）

### 学习主题

- 进程创建、执行、退出、等待；线程与内核线程。
- 调度类、runqueue、抢占、CPU affinity、上下文切换。
- 中断、softirq、tasklet（理解遗留使用）、workqueue、timer、hrtimer。
- spinlock、mutex、rwsem、completion、wait queue、atomic/refcount、RCU。
- 内存屏障与锁无关代码只做基础理解，初学阶段不要随意设计 lock-free 结构。

### 实操任务

1. 用 tracepoint/ftrace 记录一个进程从 wakeup 到被调度的路径与延迟。
2. 写 workqueue + hrtimer 模块，对比硬中断上下文与进程上下文允许的操作。
3. 植入锁顺序反转，让 lockdep 报告潜在死锁，再修复。
4. 用 `perf sched` 分析 CPU 密集型和 I/O 密集型负载。
5. 阅读一个小型并发修复 commit，复现旧问题并验证补丁。

### 阶段项目：调度延迟实验

构造可控负载，收集 wakeup latency 分布，比较 CPU 数、优先级、绑核和抢占配置的影响；报告必须区分事实、推断和噪声。

### 验收标准

- [ ] 能从调用上下文决定使用 mutex 还是 spinlock。
- [ ] 能解释“原子上下文不可睡眠”的实际后果。
- [ ] 能用 trace 数据定位一次异常调度延迟，而不是只看平均值。

---

## 阶段 5：内存管理（第 19～23 周）

### 学习主题

- 页表、TLB、缺页异常、VMA、`mmap`、COW。
- 伙伴系统、SLUB、页分配标志、内存回收、LRU/folio 基本概念。
- 页缓存、匿名页、swap、OOM、内存控制组。
- NUMA、THP 作为进阶观察项。

### 实操任务

1. 写用户程序触发 minor/major page fault，用 `perf stat` 与 tracepoint 观察。
2. 用 `/proc/<pid>/maps`、`smaps`、`pagemap`（受权限限制时记录限制）分析布局。
3. 模块中比较 `kmalloc`、`vmalloc`、page allocator 的限制与地址特征。
4. 制造受控内存压力，观察 reclaim、swap 和 OOM；只在 VM 内执行。
5. 使用 KASAN/KFENCE 检测错误，理解报告中的分配栈和释放栈。

### 阶段项目：缺页与回收观测器

用 trace/perf/BPF 任一方式采集缺页、分配和回收事件，输出按进程聚合的指标，并用两个负载验证解释是否成立。

### 验收标准

- [ ] 能画出一次匿名页缺页的主要路径。
- [ ] 能区分虚拟内存占用、RSS、PSS、页缓存与 slab。
- [ ] 遇到 OOM 时能从日志解释约束来源，而不只说“内存不够”。

---

## 阶段 6：VFS、文件系统与块 I/O（第 24～28 周）

### 学习主题

- VFS 对象关系、路径解析、挂载命名空间、权限检查。
- page cache、writeback、direct I/O 的基本边界。
- bio、request、块层与设备的概念链路。
- 日志、崩溃一致性和 fsck 的核心思想。

### 实操任务

1. 跟踪一次 `openat → read → close` 的 VFS 路径。
2. 写一个只读 pseudo filesystem 或基于 `libfuse` 先验证设计，再尝试内核实现。
3. 用 loop device 创建 ext4 镜像，观察创建、挂载、写回、卸载；破坏实验仅针对镜像副本。
4. 比较 buffered I/O、direct I/O 和 `mmap` I/O 的行为。
5. 用 fault injection 测试分配失败或 I/O 错误路径。

### 阶段项目：最小文件系统

实现挂载、目录、文件读取和统计信息；若实现写入，必须写清崩溃一致性假设。配套 mount/read/unmount 回归测试。

### 验收标准

- [ ] 能解释 `fd → file → dentry → inode → super_block` 的关系。
- [ ] 能用 trace 证明一次数据何时进入页缓存、何时写入块设备。
- [ ] 能处理部分读写、并发访问、卸载与失败清理。

---

## 阶段 7：网络栈与 eBPF 可观测性（第 29～32 周）

### 学习主题

- socket API、sk_buff、收发路径、NAPI、路由、netfilter。
- network namespace、veth、bridge，使用容器概念但不依赖复杂编排系统。
- BPF 程序类型、map、verifier、BTF、CO-RE、libbpf 基础。
- eBPF 先作为“观察内核”的工具，再学习改变数据路径。

### 实操任务

1. 用 network namespace + veth 搭建两节点网络。
2. 跟踪一次 TCP connect、发送、接收、关闭的关键 tracepoints。
3. 写一个 libbpf CO-RE 程序，按 PID/进程统计系统调用或网络延迟。
4. 阅读 verifier 拒绝日志，修复一次越界或不可证明安全的 BPF 程序。
5. 用 `perf`/ftrace/BPF 对同一问题测量，比较侵入性与信息粒度。

### 阶段项目：内核延迟诊断工具

做一个 CLI：输入进程或 cgroup，输出 syscall、调度或网络延迟的直方图和异常样本；提供 README、权限说明、内核配置要求和测试负载。

### 验收标准

- [ ] 能解释一个包从 socket 到驱动的主要层次。
- [ ] 能说明 BPF verifier 在拒绝什么，而不是反复碰运气修改代码。
- [ ] 工具能在干净环境按 README 复现结果。

---

## 阶段 8：调试、测试、安全与性能（第 33～36 周）

### 学习主题

- GDB + QEMU、crash dump 基本分析、ftrace、perf、dynamic debug。
- KUnit、kselftest、fault injection。
- KASAN、KMSAN、UBSAN、KFENCE、lockdep、kmemleak、KCOV。
- 静态检查：compiler warnings、sparse、Coccinelle、Smatch（按子系统需要选择）。
- fuzzing：理解 syzkaller 架构、最小复现和报告分析。
- 性能方法：先建立基线和假设，再采样、定位、修改、复测。

### 实操任务

1. QEMU 使用 `-s -S` 停在启动阶段，通过 `gdb vmlinux` 连接并单步关键函数。
2. 为之前的字符设备写 KUnit（适合纯函数/组件）与 kselftest（用户接口）。
3. 开启 KASAN + lockdep 跑全部自建测试并修复问题。
4. 选择一个公开 syzbot 报告：读懂堆栈、定位子系统、尝试最小化复现；不要求发现新漏洞。
5. 对一个真实性能问题完成 A/B 测量，记录平均值、分位数、方差和环境。

官方文档给出的 QEMU/GDB 基本方式是让 QEMU 以 `-s -S` 启动，然后在 GDB 中加载 `vmlinux` 并连接 `target remote :1234`。KUnit 可直接通过 `tools/testing/kunit/kunit.py run` 启动；kselftest 位于 `tools/testing/selftests/`。

### 阶段项目：故障诊断档案

建立 5 类故障样本：panic、内存越界、泄漏、死锁/竞态、性能退化。每类包含触发器、原始日志、定位过程、根因、修复补丁和回归测试。

### 验收标准

- [ ] 面对 kernel oops 能读出异常类型、faulting address、调用栈和疑似首个异常帧。
- [ ] 每个修复至少有一个“修复前失败、修复后通过”的测试。
- [ ] 能区分症状、相关性和根因。

---

## 四、毕业项目与上游协作（第 37～40 周）

### 选题方向（只选一个）

- **驱动方向**：QEMU 虚拟设备 + platform/PCI 驱动 + 中断/DMA 的受控实验。
- **内存方向**：内存压力与 reclaim 诊断工具，附真实负载分析。
- **文件系统方向**：完善最小文件系统或为现有文件系统补测试/修 bug。
- **网络/BPF 方向**：生产可用的低开销延迟观测工具。
- **质量与安全方向**：为一个较小子系统补 KUnit/kselftest、静态检查规则或 fuzz 描述。

### 毕业项目必须包含

1. 问题定义与不做什么；
2. 设计与关键数据结构；
3. 可重复构建和启动环境；
4. 正常、边界、错误、并发测试；
5. 动态检测与静态检查结果；
6. 性能基线和回归对比；
7. 3～8 个逻辑独立、可审查的 commits；
8. 一篇面向陌生开发者的复现文档。

### 上游实践顺序

1. 先读目标目录文档、`MAINTAINERS` 和近期邮件/提交。
2. 从文档修正、测试增强、真实小 bug 开始，不做无价值的空白/拼写清理。
3. 一个 patch 只解决一个问题，说明问题、原因、方案、影响和测试。
4. 执行 `scripts/checkpatch.pl`，并按变更范围运行 build、sparse、KUnit/kselftest。
5. 使用 `scripts/get_maintainer.pl` 找到维护者和邮件列表。
6. 本地先生成 `git format-patch` 并自审；只有完全理解内容后才签署 `Signed-off-by`。
7. 根据评审意见修改，保留版本变化说明，不把评审当作一次性考试。

Linux 官方提交指南要求使用 Git 准备补丁、运行 `checkpatch.pl`、按 `MAINTAINERS`/`get_maintainer.pl` 选择收件人，并强调一个 patch 解决一个问题；提交前还应参考官方 [patch checklist](https://docs.kernel.org/process/submit-checklist.html)。

---

## 五、每周可直接照抄的节奏

| 时间 | 内容 | 产出 |
|---|---|---|
| 周一 1h | 读官方文档，列 5 个问题 | 问题清单 |
| 周二 1.5h | 源码定位与 Git 历史 | 调用图、关键 commit |
| 周三 1.5h | 做基线实验 | 命令、原始日志、环境信息 |
| 周四 1.5h | 修改或植入故障 | 最小 patch |
| 周五 1.5h | trace/debug | 证据链 |
| 周六 3h | 完成项目增量、自动化测试 | 可运行版本 |
| 周日 1h | 复盘与提交 | 周报、commit、下周问题 |

每周只要求一个闭环，不追求阅读页数。一个能重复运行的小实验，价值高于看完一章却没有验证。

---

## 六、AI 协作工作流

### 1. 阅读源码

推荐提示模板：

```text
我在阅读 Linux <版本> 的 <函数/文件>。请不要泛泛解释：
1. 先列入口、核心结构体、锁、执行上下文和可能睡眠点；
2. 给出完整调用链中需要我亲自核对的符号；
3. 标记你不确定或可能随版本变化的内容；
4. 设计一个 ftrace/GDB 实验验证调用链；
5. 最后给我 3 个检查理解的问题。
```

### 2. 调试故障

```text
以下是内核配置、复现步骤和完整日志。请分成：已证实事实、候选根因、排除实验。
不要直接给补丁；先给最小化复现方案和每个假设对应的观测信号。
```

### 3. 审查补丁

```text
请按正确性、并发、生命周期、错误路径、ABI、性能、风格、测试八个维度审查。
每条意见必须引用具体代码并给出能证伪该意见的测试；不要替我添加 Signed-off-by。
```

### 4. 防止 AI 幻觉的核验规则

- AI 提到的函数：用 `rg` 确认存在，用 Git 确认目标版本。
- AI 提到的调用关系：用静态搜索加一次动态 trace 双重确认。
- AI 生成的 patch：逐行解释后再编译；不能解释的代码不合入。
- AI 给出的锁建议：核对上下文、锁顺序、对象生命周期和官方 locking 文档。
- AI 给出的性能结论：必须有固定环境下的重复实验和统计量。
- AI 声称“测试通过”：只认可本机/CI 的真实命令输出。

---

## 七、学习资料优先级

### 第一优先级：随源码维护的官方材料

- [Linux Kernel Documentation](https://docs.kernel.org/)
- [Kernel subsystem documentation](https://docs.kernel.org/subsystem-apis.html)
- 源码树中的 `Documentation/`、目标目录 README、Kconfig 帮助文本
- `MAINTAINERS`、`git log -- <path>`、`git blame`
- [Submitting patches](https://docs.kernel.org/process/submitting-patches.html)

### 第二优先级：工具官方文档

- [QEMU GDB usage](https://qemu.readthedocs.io/en/master/system/gdb.html)
- [Kernel GDB debugging](https://docs.kernel.org/dev-tools/gdb-kernel-debugging.html)
- [ftrace](https://docs.kernel.org/trace/ftrace.html)
- [KUnit](https://docs.kernel.org/dev-tools/kunit/start.html)
- [kselftest](https://docs.kernel.org/dev-tools/kselftest.html)
- [BPF / libbpf](https://docs.kernel.org/bpf/libbpf/index.html)
- [syzkaller Linux setup](https://github.com/google/syzkaller/blob/master/docs/linux/setup.md)

### 第三优先级：经典教材与课程

教材用于构建概念框架，源码用于核对当前实现。阅读旧教材时，重点保留设计思想，不照搬具体函数名、结构体字段和机制细节。

建议按需选择：

- 《Operating Systems: Three Easy Pieces》：进程、虚拟内存、并发、持久化。
- 《Computer Systems: A Programmer's Perspective》：机器级程序、链接、异常、虚拟内存。
- 《Linux Kernel Development》：建立内核整体地图；实现细节需和当前源码对照。
- 《Understanding the Linux Kernel》：适合深入历史设计，同样不可当作当前源码手册。
- xv6 及其课程实验：适合理解原理，但不能替代 Linux 实操。

---

## 八、里程碑与能力自测

| 时间 | 应达到的能力 | 可展示成果 |
|---|---|---|
| 1 个月 | 扎实的 C/系统调用/调试基础 | minish、mmap 实验 |
| 2 个月 | 能编译启动并导航内核源码 | 启动地图、最小配置 |
| 3 个月 | 能写安全可测的内核模块 | 字符设备与回归测试 |
| 5 个月 | 能分析调度、并发、内存 | 调度报告、内存观测器 |
| 7 个月 | 能跟踪 VFS/网络真实路径 | 最小文件系统、网络实验 |
| 8 个月 | 能用 BPF/GDB/检测器诊断 | 延迟工具、故障档案 |
| 9 个月 | 能完成可审查项目和规范 patch | 毕业项目、patch series |

最终自测：任选一个陌生但较小的内核 bug，能否在不依赖 AI 最终结论的前提下，完成复现、定位、修复、测试、性能/回归评估和清晰表达？如果可以，这条路线就已经从“学过内核”走到了“具备内核工程能力”。

---

## 九、如果时间有限：12 周压缩版

- 第 1～2 周：C/ELF/系统调用/Git。
- 第 3 周：编译、QEMU 启动、源码导航。
- 第 4～5 周：模块、字符设备、用户态测试。
- 第 6 周：调度、锁、workqueue、lockdep。
- 第 7 周：虚拟内存、页分配、KASAN。
- 第 8 周：VFS 和一次完整 I/O 跟踪。
- 第 9 周：网络栈与 namespace 实验。
- 第 10 周：ftrace/perf/BPF 观测工具。
- 第 11 周：GDB、KUnit、kselftest、故障诊断。
- 第 12 周：完成一个小项目并整理为 patch series。

压缩版只能建立“可继续深入的骨架”，不能替代并发、内存管理和子系统项目所需的大量实践。
