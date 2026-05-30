# F2FS 测试与特性边界深度学习报告

本文面向 F2FS 的测试、可观测性、sysfs/debugfs、ACL/xattr/verity/fscrypt 边界学习。阅读范围包括 `fs/f2fs/sysfs.c`、`debug.c`、`iostat.c`、`acl.c`、`xattr.c`、`verity.c`、`Kconfig`、`Makefile`、`Documentation/filesystems/f2fs.rst` 以及 `tools/testing/selftests/filesystems`。

## 1. 如何观察 F2FS 状态

### 1.1 sysfs：挂载实例、特性与运行时调参

F2FS 在初始化时注册 `/sys/fs/f2fs` kset，并创建全局 `features`、`tuning` 目录；每个挂载实例按 `sb->s_id` 创建 `/sys/fs/f2fs/<dev>`，其下还有 `stat` 和 `feature_list` 子目录。代码入口在 `f2fs_init_sysfs()` 与 `f2fs_register_sysfs()`。

重点观察路径：

- `/sys/fs/f2fs/features/*`：表示当前内核编译支持的 F2FS 能力，例如 `encryption`、`verity`、`compression`、`casefold`、`atomic_write`、`pin_file`、`linear_lookup`、`packed_ssa` 等。
- `/sys/fs/f2fs/<dev>/feature_list/*`：表示该挂载实例/超级块实际具备的 on-disk feature，例如 `sb_encryption`、`sb_verity`、`sb_compression`、`sb_casefold`、`sb_readonly`、`sb_device_alias` 等。
- `/sys/fs/f2fs/<dev>/stat/*`：小粒度统计计数，受 `CONFIG_F2FS_STAT_FS` 影响，例如 checkpoint、GC 调用次数、移动块数、平均 valid blocks、碎片整理块数等。
- `/sys/fs/f2fs/<dev>/*`：运行时状态与调参入口，覆盖 GC、discard、checkpoint、readahead、atomic write、compression、fragment 实验、age threshold、extent cache 等。

可优先查看的只读状态：

- 空间与生命周期：`dirty_segments`、`free_segments`、`ovp_segments`、`unusable`、`lifetime_write_kbytes`、`reserved_blocks`、`current_reserved_blocks`。
- checkpoint/错误状态：`sb_status`、`cp_status`、`mounted_time_sec`。
- discard：`pending_discard`、`issued_discard`、`queued_discard`、`undiscard_blks`。
- 特性与编码：`features`、`encoding`、`encoding_flags`、`effective_lookup_mode`。
- 统计：`cp_foreground_calls`、`cp_background_calls`、`gc_foreground_calls`、`gc_background_calls`、`moved_blocks_foreground`、`moved_blocks_background`、`avg_vblocks`、`defrag_blocks`。

可优先验证的可写调参：

- GC：`gc_urgent`、`gc_idle`、`gc_*_sleep_time`、`gc_valid_thresh_ratio`、`gc_reclaimed_segments`。
- discard：`max_small_discards`、`max_discard_request`、`discard_granularity`、`discard_io_aware`、`discard_urgent_util`、`max_ordered_discard`。
- checkpoint/后台线程：`cp_interval`、`idle_interval`、`ckpt_thread_ioprio`。
- iostat：`iostat_enable`、`iostat_period_ms`，仅在 `CONFIG_F2FS_IOSTAT` 下存在。
- fault injection：`inject_rate`、`inject_type`、`inject_lock_timeout`，仅在 `CONFIG_F2FS_FAULT_INJECTION` 下存在。
- fragmentation 实验：`max_fragment_chunk`、`max_fragment_hole`，配合挂载参数 `mode=fragment:block`。
- 压缩：`compr_written_block`、`compr_saved_block`、`compr_new_inode`、`compress_percent`、`compress_watermark`，受 `CONFIG_F2FS_FS_COMPRESSION` 影响。

观察原则：先确认全局 `features` 是否由内核支持，再确认 `<dev>/feature_list` 是否由格式化/超级块启用，最后看 `<dev>` 根目录下的运行时状态与可写调参是否存在。

### 1.2 procfs：按挂载实例输出结构化内部视图

`f2fs_register_sysfs()` 同时在 `/proc/fs/f2fs/<dev>` 创建多个 seq_file：

- `segment_info`：segment/SIT 级别信息，适合观察 segment valid block 分布。
- `segment_bits`：segment bitmap 视图，适合定位脏段、预释放段、有效块图。
- `victim_bits`：GC victim 相关位图。
- `discard_plist_info`：discard pending list。
- `disk_map`：多设备或设备区间映射。
- `donation_list`：donation 文件列表和状态。
- `iostat_info`：仅 `CONFIG_F2FS_IOSTAT`，且 `iostat_enable=1` 后输出读写/flush/discard/zone reset 统计。
- `inject_stats`：仅 `CONFIG_F2FS_FAULT_INJECTION`，输出各 fault type 的 injected count。

`iostat_info` 的读法比较直接：WRITE、READ、OTHER 三段分别给出 `io_bytes`、`count`、`avg_bytes`，另外有 `fs read folio order` 统计。注意关闭 `iostat_enable` 会重置累计数据。

### 1.3 debugfs：全局 status 快照

`CONFIG_F2FS_STAT_FS` 编译 `debug.o`，并在 debugfs 创建 `/sys/kernel/debug/f2fs/status`。它按已挂载 F2FS 分区逐个输出综合状态：

- 分区读写状态、CP 状态、SBI flag。
- SB/CP/SIT/NAT/SSA/MAIN 区域段数。
- utilization、valid node/data、inline xattr/data/dentry、compressed inode/block、orphan/append/update inode。
- 当前 curseg、dirty/full/valid block 分布。
- multi-device stats。
- checkpoint、GC、GC skip、reclaimed segment、moved block。
- extent cache 命中率、内部节点数量。
- IO/dirty pages/NAT/SIT/free nid、内存占用。

debugfs status 适合“整体体检”，sysfs/procfs 适合“单项观察与调参”。如果要做回归验证，建议把 workload 前后的 `status`、`/sys/fs/f2fs/<dev>` 关键节点、`/proc/fs/f2fs/<dev>/iostat_info` 一起采样。

### 1.4 tracepoints 与用户态工具

`iostat.c` 周期性触发 `trace_f2fs_iostat` 与 `trace_f2fs_iostat_latency`，适合用 ftrace/perf/trace-cmd 做动态观察。典型路线：

```sh
echo 1 > /sys/kernel/tracing/events/f2fs/f2fs_iostat/enable
echo 1 > /sys/kernel/tracing/events/f2fs/f2fs_iostat_latency/enable
cat /sys/kernel/tracing/trace_pipe
```

用户态工具侧：

- `mkfs.f2fs`：构造 on-disk feature 与 layout。
- `fsck.f2fs`：做一致性检查，适合故障注入、崩溃恢复、xattr/verity 元数据异常后的离线验证。
- `dump.f2fs`：按 inode 或 SIT/SSA dump 元数据，适合与内核 proc/debugfs 输出交叉验证。

## 2. 可配置特性与测试路线

### 2.1 Kconfig/Makefile 边界

核心编译开关：

- `CONFIG_F2FS_FS`：F2FS 主体，依赖 BLOCK，选择 BUFFER_HEAD、NLS、CRC32、FS_IOMAP 等。
- `CONFIG_F2FS_STAT_FS`：编译 `debug.o`，提供 debugfs status 与部分 sysfs stat。
- `CONFIG_F2FS_FS_XATTR`：编译 `xattr.o`，默认 y；若启用 `FS_ENCRYPTION`，F2FS 会选择 xattr。
- `CONFIG_F2FS_FS_POSIX_ACL`：编译 `acl.o`，依赖 xattr，并选择 `FS_POSIX_ACL`。
- `CONFIG_F2FS_FS_SECURITY`：启用 `security.*` xattr handler。
- `CONFIG_FS_VERITY`：编译 `verity.o`，不是 F2FS 自有 Kconfig 项，但 F2FS Makefile 受它控制。
- `CONFIG_F2FS_FS_COMPRESSION` 及 LZO/LZ4/ZSTD 子项：编译 `compress.o`。
- `CONFIG_F2FS_IOSTAT`：编译 `iostat.o`，提供 sysfs 开关、proc 统计和 tracepoint。
- `CONFIG_F2FS_FAULT_INJECTION`：提供 fault injection 挂载参数、sysfs 注入参数和 proc 统计。
- `CONFIG_F2FS_CHECK_FS`：运行时一致性检查 BUG_ON，适合开发内核，不适合性能基准。

测试矩阵应至少覆盖“编译有/无”和“挂载启/禁”两个层次。例如 xattr 支持已编译，但挂载 `nouser_xattr` 后 user.* 应返回不支持；ACL 支持已编译，但挂载 `noacl` 后 POSIX ACL 行为应回退到普通 mode/umask。

### 2.2 xattr 测试路线

`xattr.c` 支持 user、trusted、security、system advise，以及 ACL 使用的 internal xattr index。关键边界：

- `user.*` 受挂载选项 `XATTR_USER` 控制，`nouser_xattr` 下 get/set/list 应不可用。
- `trusted.*` list 需要 `CAP_SYS_ADMIN`。
- `security.*` 受 `CONFIG_F2FS_FS_SECURITY` 控制，用于 LSM label、file capability 等。
- xattr 名称长度超过 `F2FS_NAME_LEN` 返回 `-ERANGE`。
- value 超过 `MAX_VALUE_LEN(inode)` 返回 `-E2BIG`。
- `XATTR_CREATE` 已存在时返回 `-EEXIST`，`XATTR_REPLACE` 不存在时返回 `-ENODATA`。
- xattr 结构损坏会设置 `SBI_NEED_FSCK` 并调用 `f2fs_handle_error(ERROR_CORRUPTED_XATTR)`。
- 加密上下文 xattr 设置后会标记 encrypted inode。

建议测试：

1. 基础 CRUD：`setfattr/getfattr/listfattr` 覆盖 user.*、trusted.*、security.*。
2. 挂载参数：同一镜像分别以默认、`nouser_xattr` 挂载，对比 user.* 行为。
3. 边界大小：名称长度、value 长度、多个 xattr 填满 inline/block xattr 空间。
4. symlink/special file：确认 VFS xattr 行为与 F2FS handler 一致。
5. crash/fsck：设置/删除目录 xattr 后强制断电或 fault injection，观察 `XATTR_DIR_INO`、checkpoint 与 fsck 结果。
6. 与 fscrypt/verity 交叉：加密目录中设置 user.*，启用 verity 后确认 verity xattr 只保存 descriptor location，不直接保存大 Merkle tree。

现有 `tools/testing/selftests/filesystems/xattr` 是 sockfs/socket xattr 测试，不是 F2FS 专项；可借用 kselftest 风格和断言模式，但 F2FS 需要 loop image + mkfs.f2fs + mount namespace 的自建 fixture。

### 2.3 POSIX ACL 测试路线

`acl.c` 将 POSIX ACL 编码进 F2FS xattr。关键边界：

- ACL on-disk header version 必须为 `F2FS_ACL_VERSION`。
- access ACL 使用 `F2FS_XATTR_INDEX_POSIX_ACL_ACCESS`，default ACL 使用 `F2FS_XATTR_INDEX_POSIX_ACL_DEFAULT`。
- default ACL 只能用于目录；对非目录设置 default ACL，acl 非空时返回 `-EACCES`。
- symlink 或父目录未启用 POSIX ACL 时，新建 inode 走 umask。
- access ACL 可等价折叠为 mode 时，不保留冗余 ACL。
- 设置 ACL 会更新 inode mode，并通过 `FI_ACL_MODE` 在 xattr 写入成功后同步 mode 状态。

建议测试：

1. `setfacl/getfacl` 基础读写、删除、replace。
2. 默认 ACL 继承：父目录设置 default ACL 后创建文件/目录，确认新 inode 的 access/default ACL 与 mode。
3. `noacl` 挂载：确认 `setfacl` 返回不支持或按 VFS 预期失败，新建文件仅受 umask。
4. 非目录 default ACL：应失败。
5. chmod 交互：chmod 后 ACL mask/mode 是否按 POSIX 语义变化。
6. xattr 空间压力：ACL 条目较多时与其他 xattr 竞争空间，观察 `-E2BIG`。

### 2.4 fs-verity 测试路线

`verity.c` 的核心设计是：Merkle tree 与 fsverity descriptor 不放在 xattr value 中，而是从 `round_up(i_size, 65536)` 之后存储在文件数据地址空间；xattr 只保存 descriptor 的 `{version, size, pos}`。原因是 F2FS xattr 总空间有限，且 fscrypt 不加密 xattr，而 verity metadata 对加密文件必须加密。

关键边界：

- 正在启用 verity 时设置 `FI_VERITY_IN_PROGRESS`，重复启用返回 `-EBUSY`。
- atomic file 不支持 enable verity，返回 `-EOPNOTSUPP`。
- enable 前会初始化 quota 并转换 inline inode。
- 写入 descriptor、等待 writeback、设置 verity xattr、最后设置 inode verity flag；顺序关系是崩溃一致性重点。
- 失败路径会截断 i_size 之后的 verity metadata，并在 truncate 失败时设置 `SBI_NEED_FSCK`。
- descriptor xattr 格式错误返回 `-EINVAL`，位置/大小越界返回 `-EFSCORRUPTED` 并记录 `ERROR_CORRUPTED_VERITY_XATTR`。

建议测试：

1. 基础 fsverity：`fsverity enable`、`fsverity measure`、读文件校验。
2. 修改数据块或 Merkle tree 后读文件，确认 EIO/校验失败。
3. 小文件/空文件/大文件、i_size 接近 64K 边界，验证 metadata 起始位置。
4. inline data 文件启用 verity，确认转换路径。
5. atomic write 文件启用 verity，应失败。
6. fscrypt + verity：在加密目录中创建文件并启用 verity，确认 descriptor/Merkle tree 通过文件数据路径加密，xattr 仅保存 location。
7. fault injection：在 descriptor 写入、writeback、xattr 创建、truncate cleanup 附近注入错误，观察 fsck 与 `SBI_NEED_FSCK`。

### 2.5 fscrypt 测试路线

本轮未深入阅读 `fs/f2fs/super.c`、`file.c`、`namei.c` 中 fscrypt 主路径，但从文档和 xattr/verity 可确认这些边界：

- 编译 `FS_ENCRYPTION` 时 F2FS 自动选择 `F2FS_FS_XATTR` 和 `FS_ENCRYPTION_ALGS`。
- 挂载 `test_dummy_encryption` 或 `test_dummy_encryption=v1/v2` 用于 xfstests。
- `inlinecrypt` 可请求 blk-crypto 路径，不改变磁盘格式。
- 加密上下文存储在 xattr；`f2fs_setxattr()` 检测 encryption context 后标记 encrypted inode。
- verity metadata 不能只放 xattr，因为 xattr 不被 fscrypt 加密。

建议测试：

1. xfstests generic/encrypt 相关用例，分别跑真实 policy 与 `test_dummy_encryption=v1/v2`。
2. 加密目录下 xattr/ACL 操作，确认权限与可见性不泄露 plaintext 文件名/内容。
3. fscrypt + casefold/encrypted_casefold，配合 `lookup_mode=perf|compat|auto`。
4. fscrypt + inlinecrypt，分别在支持和不支持 blk-crypto 的设备上确认 fallback。
5. fscrypt + verity，重点看启用顺序、读校验和 crash recovery。

### 2.6 iostat、fault injection 与观测测试

iostat：

- 启用：`echo 1 > /sys/fs/f2fs/<dev>/iostat_enable`。
- 调周期：`echo 1000 > /sys/fs/f2fs/<dev>/iostat_period_ms`，代码限制在最小/最大周期内。
- 读取：`cat /proc/fs/f2fs/<dev>/iostat_info`。
- 追踪：打开 f2fs iostat tracepoints。
- 关闭：`echo 0 > .../iostat_enable` 会调用 `f2fs_reset_iostat()` 清零。

fault injection：

- 编译 `CONFIG_F2FS_FAULT_INJECTION`。
- 挂载时可用 `fault_injection=<rate>,fault_type=<mask>`。
- 运行时可调 `inject_rate`、`inject_type`、`inject_lock_timeout`。
- 结果读 `/proc/fs/f2fs/<dev>/inject_stats`。
- 适合覆盖 xattr ENOMEM/ENOSPC、verity 写入失败、checkpoint/IO 错误、锁超时等边界。

推荐 workload：

- `fsstress`/xfstests：覆盖通用 VFS、xattr、ACL、fscrypt、fsverity。
- `fio`：顺序/随机 buffered/direct/mmap 读写，用于 iostat 和 GC 观察。
- 自制 shell/kselftest：专门校验 sysfs 文件存在性、读写范围和错误码。
- 崩溃恢复测试：loop/NBD/QEMU 中制造断电，重挂载 + fsck.f2fs + debugfs/procfs 采样。

## 3. 修改注释后可做的静态/动态验证

假设只修改注释或文档，不改变 C 语义，验证目标是确认没有误改源码、没有格式/文档构建问题，并能用运行时观察证明理解没有偏离。

### 3.1 静态验证

- `git diff -- fs/f2fs Documentation/filesystems/agent-study/f2fs/reports/deep_testing_features_agent_report.md`：确认只有预期文档/注释变化。
- `git diff --check`：检查 trailing whitespace、空白错误。
- `scripts/checkpatch.pl --strict --file fs/f2fs/sysfs.c` 或对实际改动文件运行 checkpatch：注释风格、行宽、拼写风险。
- `make htmldocs` 或至少 `make SPHINXDIRS=filesystems htmldocs`：如果修改 RST 文档，确认引用和表格格式正确。
- `make fs/f2fs/` 或目标架构增量编译：注释理论上不影响编译，但能捕获误删括号/宏续行等问题。
- `rg` 复核：对新增术语、sysfs 节点名、Kconfig 名称做全文搜索，避免报告或注释中写错接口名。

### 3.2 动态验证

基础挂载：

```sh
truncate -s 2G /tmp/f2fs.img
mkfs.f2fs -f /tmp/f2fs.img
mkdir -p /mnt/f2fs
mount -o loop -t f2fs /tmp/f2fs.img /mnt/f2fs
DEV=$(basename "$(findmnt -no SOURCE /mnt/f2fs)")
ls /sys/fs/f2fs
```

可观测性冒烟：

```sh
cat /sys/fs/f2fs/$DEV/features
cat /sys/fs/f2fs/$DEV/dirty_segments
cat /sys/fs/f2fs/$DEV/free_segments
cat /proc/fs/f2fs/$DEV/segment_info | head
cat /sys/kernel/debug/f2fs/status
```

iostat 冒烟：

```sh
echo 1 > /sys/fs/f2fs/$DEV/iostat_enable
dd if=/dev/zero of=/mnt/f2fs/io.bin bs=4K count=4096 oflag=sync
cat /proc/fs/f2fs/$DEV/iostat_info
echo 0 > /sys/fs/f2fs/$DEV/iostat_enable
```

xattr/ACL 冒烟：

```sh
touch /mnt/f2fs/a
setfattr -n user.demo -v value /mnt/f2fs/a
getfattr -d /mnt/f2fs/a
mkdir /mnt/f2fs/d
setfacl -m u:$(id -u):rwX /mnt/f2fs/d
getfacl /mnt/f2fs/d
```

verity/fscrypt 冒烟：

```sh
# 需要 fsverity-utils，且文件写完后再 enable verity
echo payload > /mnt/f2fs/v
fsverity enable /mnt/f2fs/v
fsverity measure /mnt/f2fs/v

# fscrypt 建议优先跑 xfstests generic/encrypt 或使用 test_dummy_encryption=v2 镜像。
```

离线一致性：

```sh
umount /mnt/f2fs
fsck.f2fs -f /tmp/f2fs.img
dump.f2fs -d 1 /tmp/f2fs.img
```

## 4. 推荐学习文档结构

建议把 F2FS 测试与特性边界整理成如下文档树：

```text
Documentation/filesystems/agent-study/f2fs/
  knowledge/
    f2fs_observability.md          # sysfs/procfs/debugfs/tracepoints/f2fs-tools
    f2fs_feature_matrix.md         # Kconfig、mkfs feature、mount option、sysfs feature_list
    f2fs_xattr_acl.md              # xattr layout、handler、ACL 编码与 POSIX 语义
    f2fs_fscrypt_verity.md         # 加密上下文、verity metadata、组合边界
    f2fs_fault_iostat_testing.md   # iostat、fault injection、workload 与采样
  tests/
    f2fs_loop_fixture.md           # loop image、mount namespace、清理流程
    f2fs_xfstests_plan.md          # xfstests 分组、配置、已知跳过项
    f2fs_kselftest_plan.md         # 可新增 selftests 的结构
    f2fs_crash_recovery_plan.md    # QEMU/断电/fsck/dump 方案
  reports/
    deep_testing_features_agent_report.md
```

每篇文档建议固定五段：

1. 入口：相关源码、Kconfig、文档路径。
2. 状态面：如何观察、读哪些文件、采样命令。
3. 控制面：Kconfig/mkfs/mount/sysfs/ioctl 如何启停。
4. 测试面：正向、错误码、并发、崩溃恢复、组合特性。
5. 未决问题：需要源码复读或实机验证的点。

## 5. 不确定点

- `Documentation/ABI/testing/sysfs-fs-f2fs` 在当前工作树中不存在，但 `Documentation/filesystems/f2fs.rst` 仍引用它；需要确认该仓库是否裁剪了 ABI 文档，还是文档引用滞后。
- 本轮未完整阅读 `fs/f2fs/super.c` 的 mount option 解析，因此 `noacl`、`nouser_xattr`、`test_dummy_encryption`、`inlinecrypt` 的具体错误码和默认值仍应以实测/源码复读确认。
- fscrypt 主路径分散在通用 fscrypt 层和 F2FS inode/namei/file 路径，本报告只从 xattr 与 verity 交叉处归纳，不能替代 fscrypt 专项分析。
- `tools/testing/selftests/filesystems/xattr` 目前偏 socket/sockfs，不是 F2FS 专用；若要加入 F2FS selftest，需要评估 kselftest 是否适合依赖 root、loop、mkfs.f2fs、mount namespace。
- sysfs 可写节点很多，部分写入有范围限制、只读挂载限制、checkpoint error 限制、GC 线程存在性限制；本报告只做路线归类，具体每个节点的返回码需要按 `f2fs_sbi_store()` 逐项表格化。
- fs-verity 与 fscrypt 组合需要真实内核、fsverity-utils、fscrypt 工具以及支持的 page size/块大小环境验证；尤其是 64K metadata 边界、inline data 转换和崩溃恢复路径，应以 xfstests/QEMU 结果为准。
