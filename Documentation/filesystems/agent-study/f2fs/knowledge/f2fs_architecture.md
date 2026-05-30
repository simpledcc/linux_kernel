# F2FS 代码架构学习总结

## 总体定位

F2FS 是面向闪存设备的 log-structured 文件系统。它把普通文件数据、node、segment、checkpoint、GC 等机制组合起来，目标是在闪存设备上减少随机覆盖写、降低清理成本，并保持崩溃后一致性。

代码可以按以下层次理解：

```text
VFS 接入层
  -> super.c / namei.c / file.c / dir.c
  -> super_operations / inode_operations / file_operations

Page Cache 与 I/O 层
  -> data.c
  -> f2fs_dblock_aops / read_folio / write_begin / write_end / writepages

Node 管理层
  -> node.c / node.h
  -> NAT / nid / node folio / inode node

Segment 管理层
  -> segment.c / segment.h
  -> SIT / free segment / dirty segment / curseg / discard

Checkpoint、恢复和 GC
  -> checkpoint.c / recovery.c / gc.c
  -> checkpoint pack / orphan / fsync node / valid block migration
```

## 核心入口

- `fs/f2fs/super.c::init_f2fs_fs()`：初始化全局缓存和子系统，调用 `register_filesystem(&f2fs_fs_type)`。
- `fs/f2fs/super.c::f2fs_get_tree()`：通过 `get_tree_bdev(fc, f2fs_fill_super)` 进入块设备挂载。
- `fs/f2fs/super.c::f2fs_fill_super()`：读取 super block/checkpoint，初始化 `f2fs_sb_info`、segment manager、node manager、GC、root inode。
- `fs/f2fs/inode.c::f2fs_iget()`：读取 inode node，并根据 inode 类型设置 VFS operations。
- `fs/f2fs/namei.c::f2fs_create()`：新建普通文件，设置 `f2fs_file_inode_operations`、`f2fs_file_operations` 和 `f2fs_dblock_aops`。
- `fs/f2fs/data.c::f2fs_dblock_aops`：普通文件 page cache 与 F2FS 的接口。
- `fs/f2fs/gc.c::f2fs_gc()`：GC 主流程，选择 victim 并迁移有效块。
- `fs/f2fs/checkpoint.c::f2fs_write_checkpoint()`：写 checkpoint pack，建立可恢复一致点。

## 核心结构体

- `struct f2fs_sb_info`：每个挂载实例的总控结构，通过 `sb->s_fs_info` 挂到 VFS superblock。
- `struct f2fs_inode_info`：包裹 VFS `struct inode`，保存 f2fs 私有 inode 状态。
- `struct f2fs_nm_info`：node manager，维护 NAT、nid 分配、free nid cache。
- `struct f2fs_sm_info`：segment manager，维护 SIT、free/dirty segment、current segment、discard。
- `struct curseg_info`：当前活跃日志段，区分 hot/warm/cold data/node。
- `struct f2fs_checkpoint`：磁盘 checkpoint pack 的内存视图，记录一致性点。

## 关键关系

```text
super_block
  -> s_fs_info: f2fs_sb_info
       -> nm_info: f2fs_nm_info
       -> sm_info: f2fs_sm_info
       -> ckpt: f2fs_checkpoint
       -> meta_inode / node_inode

inode
  -> f2fs_inode_info
  -> i_fop: f2fs_file_operations
  -> i_mapping->a_ops: f2fs_dblock_aops
```

理解 F2FS 时，建议始终把 VFS 对象和 F2FS 私有对象对应起来看：VFS 负责通用抽象，F2FS 私有结构负责闪存友好的地址映射、日志写、checkpoint 和 GC。
