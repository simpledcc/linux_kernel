# 如何学习 F2FS 源码

## 学习顺序

建议按“从 VFS 入口到 F2FS 内部机制”的顺序读：

1. `Documentation/filesystems/f2fs.rst`
   先理解 F2FS 为什么是 log-structured 文件系统，以及 NAT/SIT/SSA/checkpoint/GC 的概念。

2. `fs/f2fs/super.c`
   读 `init_f2fs_fs()`、`f2fs_get_tree()`、`f2fs_fill_super()`，弄清挂载时如何建立 `f2fs_sb_info`。

3. `fs/f2fs/f2fs.h`
   对照 `f2fs_sb_info`、`f2fs_inode_info`、`f2fs_nm_info`、`f2fs_sm_info`，建立核心结构体地图。

4. `fs/f2fs/namei.c` 与 `fs/f2fs/inode.c`
   读 `f2fs_create()`、`f2fs_lookup()`、`f2fs_iget()`，理解 inode 如何创建、读取和装配 operations。

5. `fs/f2fs/file.c` 与 `fs/f2fs/data.c`
   读 `f2fs_file_read_iter()`、`f2fs_file_write_iter()`、`f2fs_sync_file()`、`f2fs_dblock_aops`。

6. `fs/f2fs/node.c` 与 `fs/f2fs/segment.c`
   读 `get_node_path()`、`f2fs_get_dnode_of_data()`、NAT、node page、SIT、curseg、segment allocation。

7. `fs/f2fs/checkpoint.c` 与 `fs/f2fs/gc.c`
   读 checkpoint pack、prefree/free、victim 选择和 valid block 迁移。

8. `fs/f2fs/recovery.c`、`fs/f2fs/sysfs.c`、`fs/f2fs/debug.c`
   读 roll-forward recovery、状态观测、tracepoints、sysfs/procfs/debugfs 输出和测试入口。

## 建议问题清单

阅读时可以反复问这些问题：

- 一个普通文件 inode 是如何从磁盘 node folio 装配成 VFS inode 的？
- `inode->i_mapping->a_ops = &f2fs_dblock_aops` 后，page cache 如何回调 F2FS？
- `write()` 返回成功时，数据在哪些情况下还只是 dirty folio？
- `fsync()` 什么时候必须 checkpoint，什么时候只写 fsync node？
- `f2fs_allocate_data_block()` 如何更新新旧块的 SIT 状态？
- GC 如何确认 victim block 仍然是有效块？
- checkpoint 之后 prefree segment 如何变成 free segment？

## 推荐画图

至少画四张图：

- 挂载流程图：`f2fs_fill_super()` 的阶段图。
- 对象关系图：`super_block -> f2fs_sb_info -> nm_info/sm_info/ckpt`。
- I/O 流程图：read/write/fsync/writeback。
- 空间管理图：NAT/SIT/SSA/checkpoint/GC 的关系。

## 开发时的风险提醒

- 不要只看 `file.c` 就修改写路径，必须同时看 `data.c`、`node.c`、`segment.c`、`checkpoint.c`。
- 不要只在普通文件上验证，还要考虑 compressed/encrypted/verity/atomic/pinned/zoned 组合。
- 修改 GC、checkpoint、segment allocation 时，必须把崩溃恢复和 fsck 风险放在第一位。
- 当前仓库是源码快照，不是完整 Linux 构建树；真实编译和 xfstests 需要 Linux VM 或完整源码环境。

## 本轮学习产物

- `reports/f2fs_architecture_agent_report.md`
- `reports/f2fs_io_path_agent_report.md`
- `reports/f2fs_segment_gc_agent_report.md`
- `reports/f2fs_testing_agent_report.md`
- `reports/f2fs_review_agent_report.md`
- `knowledge/f2fs_architecture.md`
- `knowledge/f2fs_code_flow.md`
- `knowledge/f2fs_agent_workflow.md`
- `knowledge/f2fs_deep_code_study.md`
- `knowledge/f2fs_observability_and_tests.md`
- `reports/deep_mount_inode_dir_agent_report.md`
- `reports/deep_data_node_agent_report.md`
- `reports/deep_segment_checkpoint_recovery_agent_report.md`
- `reports/deep_testing_features_agent_report.md`
