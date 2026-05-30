# F2FS 观测与测试路线

本文整理第二轮 Testing/Features Agent 的学习结果，说明后续真正开发 F2FS 时如何观察状态、设计测试和验证风险。

## 1. 可观测性入口

F2FS 的运行状态可以从四类接口观察。

### sysfs

代码入口主要在 `fs/f2fs/sysfs.c`。初始化时创建 `/sys/fs/f2fs`，每个挂载实例按设备名创建目录：

```text
/sys/fs/f2fs/
  features
  tuning
  <dev>/
    feature_list
    stat/
    gc_urgent
    iostat_enable
    ...
```

用途：

- 查看编译和磁盘特性是否启用。
- 调整 GC、discard、iostat、fault injection 等运行时参数。
- 为性能回归收集 workload 前后的状态快照。

### procfs

`f2fs_register_sysfs()` 同时为每个挂载实例创建 `/proc/fs/f2fs/<dev>` 下的 seq 文件。常见用途是读取 iostat、segment、victim、discard、压缩、GC 等内部统计。

### debugfs

如果启用 `CONFIG_F2FS_STAT_FS`，`fs/f2fs/debug.c` 会提供 `/sys/kernel/debug/f2fs/status`。它适合做整体体检，尤其是 free/dirty/prefree segment、GC 次数、脏页、内存占用和写入统计。

### tracepoints

F2FS 提供大量 tracepoint，例如 writeback、GC、checkpoint、iostat。典型使用路线：

```text
mount -t tracefs none /sys/kernel/tracing
echo 1 > /sys/kernel/tracing/events/f2fs/enable
cat /sys/kernel/tracing/trace_pipe
```

## 2. 用户态工具

真正动态验证需要配合 f2fs-tools：

```text
mkfs.f2fs      // 创建测试镜像
fsck.f2fs      // 一致性检查
dump.f2fs      // 查看 inode、SIT、SSA 等元数据
resize.f2fs    // resize 场景
defrag.f2fs    // 碎片整理场景
```

这些工具最适合与内核侧 `/proc/fs/f2fs`、debugfs status、tracepoints 交叉验证。

## 3. 静态验证

本仓库当前是源码快照，不能在 Windows 环境直接完成完整内核构建。对注释和文档类修改，可先做以下静态检查：

```text
git diff --check -- fs/f2fs Documentation/filesystems/agent-study/f2fs
rg -n "学习注释" fs/f2fs
rg -n "TODO|FIXME|不确定" Documentation/filesystems/agent-study/f2fs
```

如果放到完整 Linux 树中，建议继续运行：

```text
scripts/checkpatch.pl --strict --file fs/f2fs/<changed-file>
make LLVM=1 W=1 fs/f2fs/
make LLVM=1 W=1 M=fs/f2fs
```

## 4. 动态测试分层

### 冒烟测试

目标是确认基本挂载和读写没有破坏：

```text
truncate -s 2G f2fs.img
mkfs.f2fs -f f2fs.img
mount -o loop f2fs.img /mnt/f2fs
echo hello > /mnt/f2fs/a
sync
cat /mnt/f2fs/a
umount /mnt/f2fs
fsck.f2fs -f f2fs.img
```

### xfstests

建议以 xfstests 作为主回归套件：

```text
FSTYP=f2fs
TEST_DEV=/dev/loopX
SCRATCH_DEV=/dev/loopY
./check generic/001 generic/013 generic/231 f2fs/
```

重点覆盖：

- `generic` 基础读写、rename、unlink、fsync、truncate。
- `generic/encrypt` 或 fscrypt 相关组。
- verity、xattr、ACL、quota、fallocate、DIO、mmap。
- f2fs 专项测试，如果当前 xfstests 版本包含对应分组。

### 崩溃恢复测试

F2FS 的 checkpoint/roll-forward/GC 修改必须做断电类测试：

1. QEMU 或 NBD/loop 环境创建 F2FS 镜像。
2. 执行 create/write/fsync/rename/unlink/truncate/GC 压力 workload。
3. 在关键点强制断电或杀 VM。
4. 重新挂载，运行 `fsck.f2fs`，验证文件内容和元数据。
5. 对比 `dump.f2fs`、debugfs status 和 tracepoint 记录。

## 5. 特性测试矩阵

开发 F2FS 时不要只测默认普通文件。至少按下面维度组合：

| 维度 | 需要覆盖的情况 |
|---|---|
| 文件类型 | regular、directory、symlink、special file |
| 写入方式 | buffered、direct I/O、mmap、writeback、fsync |
| 空间状态 | 空闲充足、空间紧张、prefree 多、GC 压力 |
| 数据特性 | inline data、compressed、atomic、pinned |
| 安全特性 | fscrypt、fsverity、xattr、POSIX ACL |
| 设备特性 | 单设备、多设备、zoned device、discard |
| 恢复语义 | clean mount、unclean mount、roll-forward recovery |
| 配置状态 | checkpoint enabled、checkpoint disabled、fault injection |

## 6. 修改注释后的本轮验证边界

本轮只新增中文学习注释和学习文档，没有修改执行逻辑。已适合做静态检查：

- 确认 diff 只包含注释和文档。
- 确认没有空白错误。
- 确认多 agent 报告、汇总文档和源码注释引用的函数名一致。

未在当前 Windows 源码快照环境执行的项目：

- 内核编译。
- KUnit。
- xfstests。
- f2fs-tools 动态挂载、fsck、dump。
- 断电恢复测试。

这些未执行项不是因为不重要，而是因为需要完整 Linux 构建和运行环境。
