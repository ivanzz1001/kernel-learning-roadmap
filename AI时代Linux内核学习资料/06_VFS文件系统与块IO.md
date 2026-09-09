# 第 6 章：VFS、文件系统与块 I/O

## 本章目标

能够从一个文件描述符追到 `file/dentry/inode/super_block`，通过 trace 观察 buffered I/O、writeback 和块层请求，并实现一个可挂载、可测试的只读 pseudo filesystem。

## 第 24 周：VFS 对象与路径解析

### 核心对象关系

```text
进程 files_struct
  → fd table
  → struct file（一次打开实例、位置与 flags）
  → path = mount + dentry（名称与层级）
  → inode（文件对象元数据与操作）
  → super_block（一个挂载文件系统实例）
```

同一 inode 可由多个 dentry/硬链接引用，同一路径多次 open 会产生不同 `struct file`。把这些对象混同会导致生命周期和缓存理解错误。

### 路径解析需要考虑

- 绝对/相对路径与 cwd/root；
- symlink 与解析次数限制；
- mount namespace 和跨挂载点；
- dcache 命中/未命中；
- rename/unlink 并发；
- 权限、LSM 与 id mapping；
- `openat/openat2` 的 dirfd 和解析约束。

### 实验：追踪 openat

准备三种场景：已缓存普通文件、首次访问文件、包含 symlink/挂载点的路径。使用 syscall tracepoint 加适量函数跟踪，比较共同路径和分叉。

限制 ftrace 范围：先设置 PID/函数 filter 再启用 tracer，避免全函数跟踪导致巨大开销。

## 第 25 周：page cache 与三种 I/O

### 三条路径的直觉

- buffered read/write：用户缓冲区与 page cache 之间复制，写完成不等于持久化。
- `mmap`：用户页表映射文件页，访问通过 fault 建立；脏页仍需 writeback。
- direct I/O：尽量绕过 page cache，但有对齐、文件系统和设备限制，也不自动等同持久化。

### 对照实验

对同一个临时块设备上的测试文件执行：

1. sequential buffered read/write；
2. random buffered read/write；
3. `mmap` read/write；
4. `O_DIRECT`（满足对齐约束）；
5. 每组分别有/无 `fsync`。

采集 syscall 次数、page fault、block events、耗时分位数、CPU 时间。不要使用宿主重要文件系统，也不要把 VM 的虚拟磁盘缓存行为直接推广到真实硬件。

### 持久性问题

回答：

- `write` 返回说明了什么？
- `fsync` 针对哪个文件/元数据？
- rename-based 原子更新为什么还需考虑目录 fsync？
- 存储设备 volatile cache 和 QEMU cache mode 会怎样影响结论？

## 第 26 周：writeback

### 状态变化

```text
clean cached page/folio
  → 用户写入
dirty
  → writeback 选择并提交 I/O
writeback
  → I/O 完成
clean 或 error
```

观察 dirty 并不代表马上提交；提交也不代表物理介质已持久。需要结合文件系统、块层、设备 flush/FUA 语义。

### 实验

使用受控写入程序每秒写固定数据并可选 `fsync`。同时采集 `/proc/meminfo` 的 Dirty/Writeback、vmstat、writeback 和 block tracepoints。比较：

- 小文件 vs 大文件；
- 持续写 vs burst；
- 每次 fsync vs 最后 fsync；
- VM 内存大小变化。

报告中标出应用看到的完成时间和内核/设备各层完成时间。

## 第 27 周：块层

### 对象层次

```text
文件系统提交 I/O
  → bio（描述块范围和页）
  → request queue / blk-mq
  → driver
  → 虚拟或真实设备
  → completion 向上返回
```

现代内核具体函数和字段变化较快；通过 `Documentation/block/`、tracepoint 格式、目标版本源码确认。

### loop 设备实验

仅对新建镜像操作：

```bash
truncate -s 1G /tmp/kernel-lab.img
mkfs.ext4 /tmp/kernel-lab.img
sudo mkdir -p /mnt/kernel-lab
sudo mount -o loop /tmp/kernel-lab.img /mnt/kernel-lab
```

卸载后再删除镜像。实验前用 `findmnt`/`losetup` 明确目标，严禁对不确定设备运行 `mkfs`。

写负载期间记录 block tracepoints，关联 sector、bytes、提交与完成。分析合并、队列和延迟时要说明 loop 文件本身仍依赖下层宿主文件系统。

## 第 28 周：最小文件系统

### 第一版范围

实现内存中的只读 pseudo filesystem：

- 可注册与注销；
- 可 mount/unmount；
- 根目录包含一个只读文件；
- 文件内容由内核生成；
- 正确填充 superblock、inode、dentry 与 file operations；
- 支持重复挂载和卸载；
- 初始化失败无泄漏。

第一版不做磁盘格式、崩溃一致性和并发写入。范围小才能真正理解 VFS 契约。

### 实现顺序

1. 注册 `file_system_type`。
2. 实现 mount/get_tree（按目标内核 API）。
3. 构造 superblock 和 root inode/dentry。
4. 构造一个 regular file inode。
5. 用 `simple_read_from_buffer` 或 seq_file 提供内容。
6. 实现 kill_sb/退出清理。
7. 编写 mount/read/stat/多次卸载测试。

### 测试清单

- [ ] mount 成功且文件内容正确。
- [ ] 同一 mountpoint 重复 mount 的错误清晰。
- [ ] 普通用户权限行为符合设计。
- [ ] 并发 100 个 reader 数据一致。
- [ ] mount/unmount 循环 100 次。
- [ ] KASAN/kmemleak 无已知报告。
- [ ] 模块使用中卸载被安全阻止或正确处理。

## 故障与一致性扩展

完成只读版本后再选一个：

- 加可写内存文件并定义锁与 size 更新不变量；
- 用 fault injection 覆盖 inode/dentry 分配失败；
- 分析 ext4 某个小修复 commit；
- 设计用户态崩溃一致性实验，比较 `write/fsync/rename` 顺序。

破坏文件系统实验只操作可恢复镜像副本，操作前记录镜像路径、loop 设备和挂载点，确认未指向真实数据盘。

## AI 提示词

```text
请基于目标内核 commit 为这个 VFS 路径建立对象生命周期表：
file、path、mount、dentry、inode、super_block 分别由谁取得/释放引用，哪些锁或 RCU 保护查找，失败 goto 路径释放了什么。
不要根据旧版 API 猜测；列出我需要在当前源码核对的 helper 契约。
```

## 本章验收

- [ ] 能解释 fd/file/dentry/inode/superblock 的区别和引用关系。
- [ ] 能通过 trace 区分应用写完成、writeback 和块 I/O 完成。
- [ ] 能安全使用临时镜像与 loop 设备，明确目标后再格式化。
- [ ] 最小文件系统通过并发、循环挂载和检测器测试。

## 官方延伸阅读

- [Filesystems in the Linux kernel](https://docs.kernel.org/filesystems/index.html)
- [The Linux VFS](https://docs.kernel.org/filesystems/vfs.html)
- [Block layer](https://docs.kernel.org/block/index.html)
- [Fault injection](https://docs.kernel.org/fault-injection/index.html)

