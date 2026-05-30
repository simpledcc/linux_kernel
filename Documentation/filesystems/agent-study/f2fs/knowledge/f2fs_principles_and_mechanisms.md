# F2FS 工作底层原理和机制

本文从整个 F2FS 文件系统视角解释它的底层工作原理。目标不是逐行解释源码，而是建立一个能帮助阅读 `fs/f2fs/` 的机制模型。

一句话概括：

```text
F2FS 是面向闪存设备的日志式文件系统。
它尽量把新数据顺序写到新的 segment，
通过 NAT 间接定位 node，
通过 SIT 记录 segment 有效块，
通过 SSA 支持 GC 反查 block owner，
通过 checkpoint 固化一致性，
通过 roll-forward recovery 恢复已 fsync 但未 checkpoint 的修改。
```

## 1. 设计动机

F2FS 面向 NAND Flash、SSD、eMMC、UFS 这类存储设备。它们通常有这些特点：

- 随机小写代价高。
- 擦除粒度大于写入粒度。
- 频繁原地覆盖不友好。
- 写放大会影响性能和寿命。
- 设备内部还有 FTL、wear leveling、discard/trim 等机制。

因此 F2FS 避免把所有更新都做成原地覆盖，而是采用 log-structured 思路：

```text
旧块不直接覆盖
  -> 新内容写到新块
  -> 更新元数据指向新块
  -> 旧块变成无效块
  -> GC 后续回收旧块所在 segment
```

这也是理解 F2FS 的第一原则：写入路径服务于顺序追加和冷热分离，空间回收由 GC 和 checkpoint 配合完成。

## 2. 磁盘区域

F2FS 的磁盘布局可以简化为：

```text
Superblock
Checkpoint Area
Segment Information Table, SIT
Node Address Table, NAT
Segment Summary Area, SSA
Main Area
```

各区域作用如下。

| 区域 | 作用 |
|---|---|
| Superblock | 保存文件系统几何参数、feature、uuid、版本等基本信息 |
| Checkpoint Area | 保存最近一次一致性切点 |
| SIT | 记录每个 segment 的有效块数量、有效 bitmap、冷热类型、mtime |
| NAT | 记录 `nid -> node block address` |
| SSA | 保存 segment summary，用于从物理块反查 owner |
| Main Area | 保存真正的 data block 和 node block |

源码入口：

- `fs/f2fs/super.c`
- `fs/f2fs/checkpoint.c`
- `fs/f2fs/segment.c`
- `fs/f2fs/node.c`
- `fs/f2fs/f2fs.h`

## 3. 核心内存对象

挂载后，VFS 的 `super_block` 会通过 `s_fs_info` 指向 F2FS 的总控对象：

```text
struct super_block
  -> s_fs_info
    -> struct f2fs_sb_info
```

`f2fs_sb_info` 管理整个挂载实例：

```text
f2fs_sb_info
  -> raw_super       // superblock 的内存副本
  -> ckpt            // 当前有效 checkpoint
  -> meta_inode      // 元数据页缓存
  -> node_inode      // node 页缓存
  -> nm_info         // node manager, NAT/free nid
  -> sm_info         // segment manager, SIT/free/dirty/curseg
  -> write_io[]      // data/node/meta BIO 合并队列
  -> gc_thread       // GC 线程
  -> cprc_info       // checkpoint request 控制
```

常见导航宏：

```text
inode -> F2FS_I(inode)       -> f2fs_inode_info
inode -> F2FS_I_SB(inode)    -> f2fs_sb_info
sb    -> F2FS_SB(sb)         -> f2fs_sb_info
sbi   -> NM_I(sbi)           -> f2fs_nm_info
sbi   -> SM_I(sbi)           -> f2fs_sm_info
sbi   -> SIT_I(sbi)          -> sit_info
```

阅读源码时，先判断当前函数手里是 `inode`、`folio`、`mapping` 还是 `sbi`，再顺着这些宏找全局状态。

## 4. 挂载流程

挂载主线：

```text
init_f2fs_fs()
  -> register_filesystem(&f2fs_fs_type)

mount -t f2fs
  -> f2fs_get_tree()
    -> get_tree_bdev(..., f2fs_fill_super)
      -> f2fs_fill_super()
```

`f2fs_fill_super()` 可以分成这些阶段：

1. 分配并初始化 `f2fs_sb_info`。
2. 读取 raw superblock。
3. 设置 VFS super 操作表，如 `s_op`、`s_cop`、`s_vop`、`s_xattr`。
4. 读取 `meta_inode`。
5. 选择有效 checkpoint。
6. 初始化 device、post-read workqueue、extent cache、ino 管理结构。
7. 构建 segment manager。
8. 构建 node manager。
9. 读取 `node_inode` 和 root inode。
10. 处理 orphan/recovery，启动 GC/checkpoint/sysfs 等机制。

关键点：F2FS 挂载不是只读 superblock。它要恢复 checkpoint 视图，重建 NAT/SIT/free/dirty/curseg 等多个管理器，才能开始服务 VFS 请求。

## 5. 文件如何定位数据

F2FS 的文件映射有两层。

第一层：文件逻辑块到 data block。

```text
file logical block
  -> inode node / direct node / indirect node 中的 data address slot
    -> data physical block
```

第二层：node id 到 node block。

```text
nid
  -> NAT
    -> node physical block
```

因此要区分两件事：

- data block 地址存在 node page 的地址槽里。
- node page 自己的物理地址由 NAT 管。

典型路径：

```text
f2fs_map_blocks()
  -> set_new_dnode()
  -> f2fs_get_dnode_of_data()
    -> get_node_path()
    -> f2fs_get_inode_folio()
    -> f2fs_get_node_folio()
  -> dn->data_blkaddr
```

`f2fs_get_dnode_of_data()` 返回的 `dnode_of_data` 是很多路径的共同上下文：

```text
dnode_of_data
  -> inode
  -> inode_folio
  -> node_folio
  -> nid
  -> ofs_in_node
  -> data_blkaddr
```

写入、截断、fiemap、DIO、GC 迁移都会复用这套定位结果。

## 6. Page Cache 和普通写入

普通 buffered write 的主线：

```text
write()
  -> f2fs_file_write_iter()
    -> f2fs_write_checks()
    -> f2fs_should_use_dio()
    -> f2fs_preallocate_blocks()
    -> f2fs_buffered_write_iter()
      -> generic_perform_write()
        -> f2fs_write_begin()
        -> copy_from_iter()
        -> f2fs_write_end()
```

`write_begin()` 负责准备 folio、旧数据、inline/atomic/compress 状态。`write_end()` 负责把 folio 标脏、更新 i_size、处理压缩覆盖写。

重要点：

```text
write() 返回成功
  不等于数据已经写到最终物理块
```

buffered write 多数情况下只是进入 page cache。真正的物理块分配和 BIO 提交通常发生在 writeback：

```text
f2fs_write_data_pages()
  -> __f2fs_write_data_pages()
    -> f2fs_write_cache_pages()
      -> f2fs_write_single_data_page()
        -> f2fs_do_write_data_page()
          -> IPU 或 OPU
```

## 7. OPU 和 IPU

F2FS 有两种数据更新策略。

| 策略 | 含义 | 特点 |
|---|---|---|
| OPU | Out-Place Update | 写到新块，旧块失效，是 F2FS 主线 |
| IPU | In-Place Update | 特定条件下原地写旧块 |

OPU 的简化流程：

```text
dirty folio writeback
  -> f2fs_do_write_data_page()
    -> f2fs_get_dnode_of_data()
    -> f2fs_get_node_info()
    -> f2fs_outplace_write_data()
      -> f2fs_allocate_data_block()
        -> 从 curseg 取新块
        -> 写 summary
        -> update_sit_entry(new, +1)
        -> update_sit_entry(old, -1)
        -> 更新 node 地址槽
```

IPU 适合一些希望直接更新旧块的场景，例如空间压力、SSR 策略、某些同步写和旧块稳定的情况。但从整体设计看，F2FS 更偏向 OPU。

## 8. Segment 和 Active Logs

F2FS 把 Main Area 划分为 segment，多个 segment 可以组成 section。

写入不是随便找块，而是写入当前 active log：

```text
CURSEG_HOT_DATA
CURSEG_WARM_DATA
CURSEG_COLD_DATA
CURSEG_HOT_NODE
CURSEG_WARM_NODE
CURSEG_COLD_NODE
CURSEG_ALL_DATA_ATGC
```

冷热分离的目标是让生命周期相近的数据放在一起：

- hot 数据频繁更新。
- warm 数据普通更新。
- cold 数据更新少。
- data 和 node 分开。
- GC 迁移数据可以进入专门路径。

这样 GC 更容易找到无效块较多的 victim segment。

## 9. SIT 状态机

SIT 是 Segment Information Table，记录每个 segment 的有效块状态。

SIT 维护两套视图：

| 视图 | 含义 |
|---|---|
| `cur_valid_map/valid_blocks` | 当前内存视图 |
| `ckpt_valid_map/ckpt_valid_blocks` | 最近一次 checkpoint 视图 |

OPU 后的状态变化：

```text
新块 valid +1
旧块 valid -1
旧 segment 变 dirty
valid_blocks == 0 时进入 prefree
checkpoint 成功后 prefree 变 free
```

segment 生命周期：

```text
free
  -> in-use / active
  -> dirty
  -> prefree
  -> checkpoint
  -> free
```

`prefree` 非常重要。它表示当前内存视图里 segment 已经没有有效块，但上一次 checkpoint 可能仍然引用它，所以还不能立刻复用。只有 checkpoint 成功后，它才可以变成真正 free。

## 10. SSA 和 GC 反查

SSA 是 Segment Summary Area。它让 F2FS 可以从物理块反查这个块属于谁。

GC 迁移 victim block 时，需要做这样的验证：

```text
physical block
  -> SSA summary
    -> nid / ofs / version
      -> NAT 找 node
        -> node page 检查当前 data address
          -> 如果仍指向该物理块，说明有效
          -> 否则说明是旧块，不迁移
```

这能避免 GC 搬运已经失效的旧块。

## 11. Checkpoint

checkpoint 是 F2FS 的一致性基线。

`f2fs_write_checkpoint()` 外层负责冻结关键修改、刷写 dirty 数据和 NAT/SIT；`do_checkpoint()` 负责组装并写出 checkpoint pack。

checkpoint 写入内容包括：

- 当前 active data/node log 的 segment 和 offset。
- NAT/SIT bitmap。
- data/node summaries。
- orphan inode blocks。
- free segment count。
- checkpoint version。
- CRC。

简化流程：

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

checkpoint 成功后：

```text
当前 NAT/SIT/summary/curseg 状态成为新的恢复基线
prefree segment 可以变成真正 free
```

checkpoint 失败时，不能清理 prefree/discard 状态，否则可能丢失仍被旧 checkpoint 引用的块。

## 12. fsync 和 Roll-Forward

F2FS 的 `fsync()` 不一定每次都做完整 checkpoint。

它会先判断是否必须 checkpoint：

```text
need_do_checkpoint()
  -> hardlink?
  -> compressed?
  -> pino 异常?
  -> 空间不足?
  -> 特殊日志配置?
  -> strict fsync 目录恢复需求?
```

如果必须 checkpoint：

```text
f2fs_do_sync_file()
  -> f2fs_sync_fs()
```

如果不需要完整 checkpoint：

```text
f2fs_do_sync_file()
  -> f2fs_fsync_node_pages()
  -> 写 fsync node 链
```

这样可以降低 fsync 成本。崩溃后，F2FS 通过 roll-forward recovery 沿 fsync node 链恢复 checkpoint 之后已经 fsync 的修改。

## 13. GC 回收机制

因为 OPU 会留下旧块，F2FS 必须通过 GC 回收空间。

GC 主线：

```text
f2fs_gc()
  -> 空间压力检查
  -> 必要时先 checkpoint 回收 prefree
  -> __get_victim()
    -> f2fs_get_victim()
  -> do_garbage_collect()
    -> 读取 victim summary
    -> 通过 SSA/NAT/node 验证有效块
    -> 搬迁仍有效的数据或 node
    -> 旧块失效
```

GC 类型：

| 类型 | 作用 |
|---|---|
| BG_GC | 后台 GC，低干扰 |
| FG_GC | 前台 GC，空间不足时救急 |

victim 选择会考虑：

- segment 有效块数量。
- 冷热类型。
- mtime/age。
- current section 过滤。
- pinned section。
- CP disabled。
- foreground/background GC 策略。

GC 不是简单搬块。它必须验证每个块仍然有效，否则会把旧版本数据重新写活。

## 14. 崩溃恢复

F2FS 恢复分两层。

第一层：checkpoint recovery。

```text
读取最近一次有效 checkpoint
  -> 恢复 NAT/SIT/curseg/free/dirty 等状态
```

第二层：roll-forward recovery。

```text
f2fs_recover_fsync_data()
  -> find_fsync_dnodes()
  -> recover_data()
  -> 修复 inode/dentry/data block 映射
  -> f2fs_check_and_fix_write_pointer()
  -> f2fs_write_checkpoint(CP_RECOVERY)
```

roll-forward 只恢复 checkpoint 之后已经 fsync 的 node 链，不是重放所有写入。

挂载时的分支：

```text
可写设备且允许 recovery
  -> 真正执行 f2fs_recover_fsync_data(false)

只读设备或 norecovery
  -> f2fs_recover_fsync_data(true)
  -> 只检查是否有必须恢复的数据
  -> 必要时拒绝挂载或丢弃 fsynced data
```

## 15. 整体工作闭环

把所有机制串起来：

```text
mount
  -> 读取 superblock
  -> 选择有效 checkpoint
  -> 初始化 NAT/SIT/SSA/segment/node manager
  -> 必要时 roll-forward recovery

write
  -> 数据进入 page cache 或 DIO
  -> 查找/创建 node 映射
  -> 分配新 data block
  -> 写 summary
  -> 更新 node 地址
  -> 更新 SIT
  -> 旧块失效

fsync / checkpoint
  -> 刷数据和 node
  -> 必要时写 checkpoint
  -> 固化 NAT/SIT/summary/curseg

GC
  -> 选择 victim segment
  -> 通过 SSA/NAT/node 验证有效块
  -> 迁移有效块
  -> 旧 segment 进入 dirty/prefree/free 状态机

crash recovery
  -> 回到 checkpoint
  -> replay 已 fsync 的 node 链
  -> 成功后写 CP_RECOVERY checkpoint
```

## 16. 一个简化模型

可以把 F2FS 理解成四个协同系统：

```text
写入系统
  -> 尽量顺序写新块

映射系统
  -> node tree + NAT 找到 data 和 node

空间系统
  -> SIT + active logs + dirty/prefree/free 管理 segment

一致性系统
  -> checkpoint + fsync node + recovery 保证崩溃后可解释
```

这四个系统互相约束：

- 写入会改变 node、SIT、summary。
- checkpoint 会固化 node/NAT/SIT/summary/curseg。
- GC 依赖 SSA/NAT/node 验证有效块。
- recovery 依赖 checkpoint 和 fsync node 链恢复一致性。

因此修改 F2FS 时不能只看单个函数。尤其是写路径、GC、checkpoint、recovery，必须一起看。

## 17. 推荐源码阅读顺序

建议按下面顺序读：

1. `fs/f2fs/f2fs.h`：先建立结构体地图。
2. `fs/f2fs/super.c`：理解挂载和恢复入口。
3. `fs/f2fs/inode.c`、`fs/f2fs/namei.c`、`fs/f2fs/dir.c`：理解 VFS 对象和目录操作。
4. `fs/f2fs/file.c`：理解 read/write/fsync 入口。
5. `fs/f2fs/data.c`：理解 page cache、block mapping、writeback。
6. `fs/f2fs/node.c`、`fs/f2fs/node.h`：理解 node tree 和 NAT。
7. `fs/f2fs/segment.c`、`fs/f2fs/segment.h`：理解 active logs、SIT、SSR、block allocation。
8. `fs/f2fs/checkpoint.c`：理解一致性基线。
9. `fs/f2fs/gc.c`：理解空间回收。
10. `fs/f2fs/recovery.c`：理解 roll-forward recovery。

配套文档：

- `knowledge/f2fs_architecture.md`
- `knowledge/f2fs_code_flow.md`
- `knowledge/f2fs_deep_code_study.md`
- `knowledge/f2fs_observability_and_tests.md`
