# F2FS 深度源码学习笔记

本文汇总第二轮多 agent 深度学习结果，面向想继续阅读和修改 `fs/f2fs/` 的开发者。阅读时建议把 F2FS 看成四条主线同时运转：

1. VFS 接入：把 Linux 通用文件系统对象接到 F2FS 实现。
2. 数据映射：把文件逻辑块定位到 node tree 中的地址槽。
3. 空间管理：用 active log、SIT、SSA、checkpoint 和 GC 管理物理块生命周期。
4. 恢复语义：用 checkpoint 与 roll-forward recovery 保证崩溃后能回到一致状态。

## 1. 全局对象关系

F2FS 的核心运行时对象集中在 `fs/f2fs/f2fs.h`：

```text
VFS super_block
  -> s_fs_info
    -> struct f2fs_sb_info
      -> raw_super            // 磁盘 super block 的内存副本
      -> ckpt                 // 当前有效 checkpoint
      -> meta_inode           // NAT/SIT/SSA/checkpoint 等元数据页缓存
      -> node_inode           // node block 页缓存
      -> nm_info              // NAT 与 free nid 管理
      -> sm_info              // SIT/free/dirty/curseg 管理
      -> write_io[]           // data/node/meta BIO 合并队列
      -> gc_thread/cprc_info  // GC 和 checkpoint 后台线程状态
```

`struct f2fs_inode_info` 内嵌 VFS `struct inode`。常见入口是：

```text
inode -> F2FS_I(inode) -> f2fs_inode_info
inode -> F2FS_I_SB(inode) -> f2fs_sb_info
sb    -> F2FS_SB(sb) -> f2fs_sb_info
sbi   -> NM_I(sbi) / SM_I(sbi) / SIT_I(sbi)
```

读源码时，先找到当前函数手里的对象是 `inode`、`folio`、`mapping` 还是 `sbi`，再顺着这些宏找到全局状态，会比直接搜索字段更容易。

## 2. 挂载主线

挂载入口从模块注册开始：

```text
init_f2fs_fs()
  -> register_filesystem(&f2fs_fs_type)
mount -t f2fs
  -> f2fs_get_tree()
    -> get_tree_bdev(..., f2fs_fill_super)
      -> f2fs_fill_super()
```

`f2fs_fill_super()` 是挂载主线的核心，可以分成 8 个阶段：

1. 分配 `f2fs_sb_info`，初始化锁、链表、计数器。
2. 读取 raw super block，设置 block size 和 mount options。
3. 把 `super_block` 的 `s_op/s_cop/s_vop/s_xattr` 绑定到 F2FS。
4. 读取 `meta_inode`，选择有效 checkpoint。
5. 初始化 device、post-read workqueue、extent cache、ino 管理结构。
6. 构建 segment manager 和 node manager。
7. 读取 `node_inode` 和 `root inode`。
8. 运行 orphan/recovery、启动 GC/checkpoint/sysfs 等后台机制。

需要注意：`segment manager` 必须早于 `node manager`，因为 node manager 初始化 NAT/free nid 时依赖主区布局和 segment 元数据地址。

## 3. Inode 与目录操作

`f2fs_iget()` 把磁盘 node page 中的 inode 信息装配成 VFS inode。不同文件类型挂接不同操作表：

```text
regular file:
  i_op      = f2fs_file_inode_operations
  i_fop     = f2fs_file_operations
  a_ops     = f2fs_dblock_aops

directory:
  i_op      = f2fs_dir_inode_operations
  i_fop     = f2fs_dir_operations
  a_ops     = f2fs_dblock_aops

symlink:
  i_op      = f2fs_symlink_inode_operations 或 f2fs_encrypted_symlink_inode_operations
  a_ops     = f2fs_dblock_aops
```

目录项主线在 `namei.c` 和 `dir.c`：

```text
lookup:
  f2fs_lookup()
    -> f2fs_prepare_lookup()
    -> __f2fs_find_entry()
      -> inline dentry 或 hash level/bucket/block 扫描
    -> f2fs_iget(target_ino)
    -> d_splice_alias()

create:
  f2fs_create()
    -> f2fs_new_inode()
    -> f2fs_lock_op()
    -> f2fs_add_link()
      -> f2fs_add_dentry()
        -> f2fs_add_inline_entry() 或 f2fs_add_regular_entry()
    -> f2fs_alloc_nid_done()
    -> d_instantiate_new()

unlink:
  f2fs_unlink()
    -> f2fs_find_entry()
    -> f2fs_acquire_orphan_inode()
    -> f2fs_delete_entry()
    -> 后续 evict/truncate/checkpoint 继续回收

rename:
  f2fs_rename2()
    -> f2fs_rename()
      -> 找 old/new dir entry
      -> 处理 whiteout/orphan/父目录 ".."
      -> f2fs_set_link() 或 f2fs_add_link()
      -> f2fs_delete_entry()
```

目录读写本质上仍然使用 F2FS data block，所以目录 inode 的 `a_ops` 也是 `f2fs_dblock_aops`。

## 4. Buffered I/O 与 Page Cache

普通文件写入口在 `file.c:f2fs_file_write_iter()`：

```text
f2fs_file_write_iter()
  -> checkpoint/error/compress/NOWAIT/pinned file 检查
  -> f2fs_write_checks()
  -> f2fs_should_use_dio()
  -> f2fs_preallocate_blocks()
  -> f2fs_dio_write_iter() 或 f2fs_buffered_write_iter()
```

buffered write 最终进入 `data.c` 的 address space 回调：

```text
generic_perform_write()
  -> f2fs_write_begin()
       准备 folio、旧数据、inline/atomic/compress 状态
  -> copy_from_iter()
  -> f2fs_write_end()
       标脏 folio、更新 i_size、处理压缩覆盖写
```

关键点：`write_end()` 通常只是把 folio 标脏，数据并不一定已经拥有最终物理块。真正的物理块分配通常发生在 writeback：

```text
f2fs_write_data_pages()
  -> __f2fs_write_data_pages()
    -> f2fs_write_cache_pages()
      -> f2fs_write_single_data_page()
        -> f2fs_do_write_data_page()
          -> IPU: f2fs_inplace_write_data()
          -> OPU: f2fs_outplace_write_data()
            -> f2fs_allocate_data_block()
```

IPU 是原地写，OPU 是 out-of-place 写。F2FS 的主线是 OPU：写到新块，更新 node 页中的地址槽，再让旧块失效并等待 GC 回收。

## 5. 逻辑块到物理块

F2FS 的映射不是 `inode -> physical block` 的一层表，而是两层：

```text
file logical block
  -> inode/direct/indirect node 中的 data address slot
    -> data physical block

nid
  -> NAT
    -> node physical block
```

`node.c:get_node_path()` 把文件逻辑块号拆成最多 4 层路径：

```text
level 0: inode node 内直接地址
level 1: direct node
level 2: indirect node -> direct node
level 3: double indirect node -> indirect node -> direct node
```

`f2fs_get_dnode_of_data()` 沿这个路径读取或创建 node：

```text
set_new_dnode()
  -> f2fs_get_dnode_of_data(dn, index, mode)
    -> get_node_path()
    -> f2fs_get_inode_folio()
    -> 必要时 f2fs_alloc_nid() + f2fs_new_node_folio()
    -> f2fs_get_node_folio()
    -> dn->node_folio / dn->ofs_in_node / dn->data_blkaddr
```

三种常见模式：

- `LOOKUP_NODE`：只查已有 node，读、writeback、truncate 常见。
- `ALLOC_NODE`：路径缺 node 时创建，扩展写入和预分配常见。
- `LOOKUP_NODE_RA`：查找时带最后一级 node readahead，截断和预读常见。

`data.c:f2fs_map_blocks()` 是统一块映射接口。它会先尝试 read extent cache，再走 dnode 查找，并根据调用 flag 决定是否分配、是否允许 hole、是否为 fiemap/bmap/DIO 特殊处理。

## 6. Extent Cache 与压缩

read extent cache 缓存连续的逻辑块到物理块映射，命中后可以绕过 node 页，提高读路径和 DIO 映射效率。OPU、truncate、punch hole 等路径必须正确更新或失效 extent cache，否则可能读到旧物理块。

压缩文件以 cluster 为单位，不再满足“逻辑连续块数等于物理连续块数”的普通 extent 假设：

```text
cluster slot 0: COMPRESS_ADDR
cluster slot 1..N: 压缩后的物理页
```

压缩读：

```text
f2fs_read_data_folio()
  -> f2fs_mpage_readpages()
    -> f2fs_read_multi_pages()
      -> 读取压缩页
      -> post-read 解压
      -> 填回原 page cache folio
```

压缩写分为两类：

1. 部分覆盖写：`f2fs_prepare_compress_overwrite()` 先读入整个 cluster，`f2fs_compress_write_end()` 再标脏。
2. writeback 压缩：`f2fs_write_cache_pages()` 聚合 cluster，`f2fs_write_multi_pages()` 决定压缩写或 raw write。

## 7. Segment、SIT、SSA 与 Active Logs

F2FS 用 active logs 组织新写位置。常见类型：

```text
CURSEG_HOT_DATA
CURSEG_WARM_DATA
CURSEG_COLD_DATA
CURSEG_HOT_NODE
CURSEG_WARM_NODE
CURSEG_COLD_NODE
CURSEG_ALL_DATA_ATGC
```

`segment.c:f2fs_allocate_data_block()` 是 OPU 分配核心：

```text
f2fs_allocate_data_block()
  -> 取 curseg_lock / curseg_mutex / sentry_lock
  -> new = NEXT_FREE_BLKADDR(curseg)
  -> 写 summary: nid/ofs/version
  -> update_sit_entry(new, +1)
  -> update_sit_entry(old, -1)
  -> segment 满时 new_curseg() 或 change_curseg()
  -> locate_dirty_segment(old/new)
```

SIT 维护两套视图：

- `cur_valid_map/valid_blocks`：当前内存视图，反映最新写入和失效。
- `ckpt_valid_map/ckpt_valid_blocks`：最近一次 checkpoint 视图，用于恢复和 CP-disabled 判断。

SSA summary 记录“这个物理块属于哪个 nid/offset/version”。GC 迁移 victim block 时，需要从 SSA 反查 owner，再用 NAT/node page 验证该块是否仍有效。

## 8. Dirty、Prefree、Free 状态机

segment 释放不是一步完成：

```text
valid blocks 变少
  -> dirty segment
valid blocks == 0
  -> prefree segment
checkpoint 成功
  -> free segment
```

为什么需要 prefree：崩溃后只能回到上一个 checkpoint。如果一个 segment 在当前内存视图里已经空了，但上一个 checkpoint 仍可能引用它，就不能立刻把它当成完全 free。只有 checkpoint 成功后，新的恢复基线不再引用这些旧块，prefree 才能清理并发 discard。

## 9. Checkpoint

`checkpoint.c:f2fs_write_checkpoint()` 是一致性切点外层：

```text
f2fs_write_checkpoint()
  -> cp_global_sem
  -> block_operations()
  -> f2fs_flush_merged_writes()
  -> checkpoint_ver++
  -> f2fs_flush_nat_entries()
  -> f2fs_flush_sit_entries()
  -> f2fs_save_inmem_curseg()
  -> do_checkpoint()
  -> f2fs_clear_prefree_segments()
  -> unblock_operations()
```

`do_checkpoint()` 写入 checkpoint pack：

- 当前 data/node curseg 的 segno、blkoff、alloc_type。
- NAT/SIT bitmap。
- data/node summaries。
- orphan inode blocks。
- checkpoint payload、version、CRC。
- 写屏障和设备 cache flush。

checkpoint 成功后，F2FS 获得新的恢复基线；checkpoint 失败时不能清掉 prefree 和 discard 状态，否则可能丢失仍被旧 checkpoint 引用的块。

## 10. GC

GC 的入口在 `gc.c:f2fs_gc()`：

```text
f2fs_gc()
  -> 空间压力检查，必要时先 checkpoint 回收 prefree
  -> __get_victim()
    -> f2fs_get_victim()
  -> do_garbage_collect()
    -> 读取 victim summary
    -> 对 data 或 node block 做有效性验证
    -> 迁移有效块
    -> 旧块失效，segment 进入 dirty/prefree
```

前台 GC 和后台 GC 的语义不同：

- BG_GC 偏向低干扰，可能只是把页标脏，等待后续 writeback。
- FG_GC 在空间不足时更激进，倾向同步迁移以尽快释放 section。

victim 选择依赖 SIT 的 valid block 数、冷热类型、mtime、age 策略、pinned section、CP disabled、current section 等过滤条件。不能把正在写的 current segment 当 victim。

## 11. Roll-Forward Recovery

`recovery.c:f2fs_recover_fsync_data()` 恢复上次 checkpoint 之后已经 fsync 但尚未 checkpoint 的修改：

```text
f2fs_recover_fsync_data()
  -> cp_global_sem 禁止 recovery 中途 checkpoint
  -> find_fsync_dnodes()
  -> recover_data()
    -> 恢复 inode/dentry/data block 映射
    -> 修补 SIT/summary
  -> f2fs_check_and_fix_write_pointer()
  -> 清 SBI_POR_DOING
  -> f2fs_write_checkpoint(CP_RECOVERY)
```

重点：recovery 不是“重放所有写入”，而是只 replay checkpoint 之后已经通过 fsync 语义确认的 node 链。成功后立即写 `CP_RECOVERY` checkpoint，把临时恢复结果固化成新的稳定状态。

## 12. 修改 F2FS 时的风险清单

- 修改写路径时必须同时检查 `file.c`、`data.c`、`node.c`、`segment.c`、`checkpoint.c`。
- 不要混淆 data block 和 node block。NAT 映射的是 nid 到 node 物理块。
- 不要混淆 `valid_blocks` 和 `ckpt_valid_blocks`。当前视图和 checkpoint 视图语义不同。
- OPU 后必须正确更新 node 地址和 extent cache。
- 压缩文件的 `COMPRESS_ADDR` 不能当成普通数据块。
- rename/unlink/create 涉及 orphan、quota、inline dir、casefold、encryption，不能只验证普通目录。
- GC 迁移必须用 SSA/NAT/node page 验证块仍有效。
- checkpoint 失败时不能清理 prefree/discard 状态。
- CP disabled、pinned file、atomic write、verity、fscrypt、zoned device 都会改变普通路径假设。
