# F2FS segment / checkpoint / GC / recovery 深度学习报告

本文只覆盖本 agent 负责的 segment manager、SIT、checkpoint、GC、roll-forward recovery 路径，主要阅读文件为 `fs/f2fs/segment.c`、`fs/f2fs/segment.h`、`fs/f2fs/checkpoint.c`、`fs/f2fs/gc.c`、`fs/f2fs/recovery.c`、`fs/f2fs/f2fs.h`。

## 1. block allocation 与 active logs

### 1.1 active log 类型

`fs/f2fs/f2fs.h` 定义了 6 个持久 active logs：

- `CURSEG_HOT_DATA`：目录项等热 data block。
- `CURSEG_WARM_DATA`：普通 data block。
- `CURSEG_COLD_DATA`：多媒体、冷数据、GC 后数据。
- `CURSEG_HOT_NODE`：目录文件 direct node。
- `CURSEG_WARM_NODE`：普通文件 direct node。
- `CURSEG_COLD_NODE`：indirect node。

另外有 2 个 in-memory log：

- `CURSEG_COLD_DATA_PINNED`：为 pinned file 分配连续块。
- `CURSEG_ALL_DATA_ATGC`：ATGC/AT_SSR 下的数据迁移目标。

`struct curseg_info` 是 active log 的内存状态，关键字段是 `segno`、`next_blkoff`、`alloc_type`、`seg_type`、`sum_blk`、`journal`。它同时承担三个职责：记录当前 segment 和下一个写入 offset；缓存 summary block；在 checkpoint 时承载 NAT/SIT journal。

### 1.2 active_logs=2/4/6 的分流

写入分流由 `__get_segment_type()` 统一决定：

- `active_logs=2`：data 全部进 `HOT_DATA`，node 全部进 `HOT_NODE`，冷热分离最弱。
- `active_logs=4`：目录 data 走 `HOT_DATA`，普通 data 走 `COLD_DATA`；dnode 根据 `is_cold_node()` 分到 `WARM_NODE` 或 `COLD_NODE`。
- `active_logs=6`：数据路径结合 pinned、GC、cold/compress file、age extent cache、hot inode 标志、write hint 分配到 hot/warm/cold data；node 仍按 dnode/cold node 分流。

`f2fs_get_segment_temp()` 将 curseg 的 `seg_type` 映射为 block layer write hint 的温度，`do_write_page()` 在真正分配前完成类型选择。

### 1.3 out-of-place block allocation 主路径

普通 data/node out-of-place 写入最终走：

```text
f2fs_outplace_write_data() / f2fs_do_write_node_page()
  -> do_write_page()
    -> __get_segment_type()
    -> f2fs_allocate_data_block()
    -> f2fs_submit_page_write()
```

`f2fs_allocate_data_block()` 是核心：

1. 取得 `SM_I(sbi)->curseg_lock` 读锁、`curseg->curseg_mutex`、`SIT_I(sbi)->sentry_lock` 写锁。
2. 用 `NEXT_FREE_BLKADDR(sbi, curseg)` 计算新块地址。
3. 把 `struct f2fs_summary` 写入 `curseg->sum_blk[next_blkoff]`。
4. 如果当前 segment 是 SSR，则调用 `f2fs_find_next_ssr_block()` 在 `ckpt_valid_map | cur_valid_map` 的反向 bit 序中找下一个空洞；否则 `next_blkoff++`。
5. 更新 mtime。GC 迁移时把源 segment 的年龄带到目标块；普通写入则按当前时间更新旧块和新块。
6. 先 `update_sit_entry(new, +1)`，再 `update_sit_entry(old, -1)`。源码中特别强调：SIT 要先于 segment 切换更新，因为 SSR 依赖最新 valid block 信息。
7. 如果当前 segment 满了，选择 `new_curseg()` 或 `change_curseg()`；ATGC 迁移则用 `get_atssr_segment()`。
8. 最后 `locate_dirty_segment(old_segno)` 和 `locate_dirty_segment(new_segno)`，延后 dirty 状态更新以避免关闭旧 segment 时重复统计。

### 1.4 LFS、SSR、AT_SSR

`new_curseg()` 永远从 free bitmap 中按 LFS 方式取新 segment。它先把旧 summary 写回 SSA，再通过 `get_new_segment()` 从 `free_secmap/free_segmap` 找空闲 section/segment，调用 `__set_inuse()` 标记为已使用，最后 `reset_curseg()` 初始化 summary footer 和 SIT type。

`change_curseg()` 则复用 dirty segment 做 SSR：先把目标 segment 标为 in-use，从 dirty/prefree 位图中移除，然后恢复该 segment 的历史 summary，并把 `alloc_type` 设置为 `SSR`。SSR 的可写位置不是线性尾部，而是 `ckpt_valid_map | cur_valid_map` 共同认为空闲的 hole。

`need_new_seg()` 决定 segment 满后是继续 LFS 取新段，还是在空间紧张时转 SSR。`f2fs_need_SSR()` 的触发条件包括非 LFS 模式、urgent GC、CP disabled，以及 free sections 低于 node/dentry/imeta 需求加保留阈值。

## 2. SIT / dirty / prefree / free segment 状态变化

### 2.1 SIT entry 双视图

`struct seg_entry` 中最容易混淆的是两套 valid 信息：

- `cur_valid_map` / `valid_blocks`：当前运行时视图，分配和失效立即更新。
- `ckpt_valid_map` / `ckpt_valid_blocks`：最近一次 checkpoint 已持久化的视图，SSR 和 CP-disabled 判断会使用它。

`seg_info_from_raw_sit()` 从磁盘 SIT 初始化时，两套视图相同。`seg_info_to_raw_sit()` 在 flush SIT 时把当前视图写入 raw SIT，同时把 `ckpt_valid_map` 和 `ckpt_valid_blocks` 更新为当前值。因此 checkpoint 是两套视图重新对齐的边界。

`update_sit_entry()` 是 SIT 状态变化中心：

- 分配新块：设置 `cur_valid_map`，增加 `valid_blocks`，必要时增加 `ckpt_valid_blocks`，更新 `written_valid_blocks`。
- 释放旧块：清除 `cur_valid_map`，减少 `valid_blocks`；如果 CP disabled 且旧块属于 checkpoint 视图，会增加 `unusable_block_count`，防止重用 checkpointed data。
- 大 section 模式下同步维护 `sec_entry.valid_blocks/ckpt_valid_blocks`。
- 每次变化通过 `__mark_sit_entry_dirty()` 设置 `dirty_sentries_bitmap`，等待 checkpoint flush。

### 2.2 dirty 与 prefree 的转换

`locate_dirty_segment()` 按 valid block 数给 segment 分类：

- `valid_blocks == 0` 且 checkpoint 条件允许：加入 `dirty_segmap[PRE]`，从 `DIRTY` 删除。PRE 表示“逻辑上已经无有效块，但还没有经过 checkpoint 完成最终释放”。
- `0 < valid_blocks < usable_blocks`：加入 `dirty_segmap[DIRTY]`，并按 segment type 同步加入 `DIRTY_HOT_DATA` 等细分类。
- `valid_blocks == usable_blocks`：从 `DIRTY` 删除，恢复 full segment 状态。

`__locate_dirty_segment()` 不会把当前 active segment 加进 dirty list，这是一个重要保护：active log 还在追加或 SSR 写入，不应被 GC 当 victim。大 section 模式还维护 `dirty_secmap`，但当前 section 也会被排除。

`f2fs_dirty_to_prefree()` 在 DIRTY 中扫描 valid_blocks 为 0 的 segment，把它们搬到 PRE。这个转换常见于 truncate、GC 迁移或 recovery 后旧块全部失效。

### 2.3 prefree 到 free 的两阶段释放

F2FS 不会在 segment 一变空就立刻成为 free。原因是最近 checkpoint 仍可能引用旧位置；崩溃后只能回到上一个 checkpoint。因此释放是两阶段：

```text
valid blocks 归零
  -> locate_dirty_segment(): DIRTY -> PRE
  -> checkpoint flush SIT/NAT/summary/curseg
  -> f2fs_flush_sit_entries(): set_prefree_as_free_segments()
  -> do_checkpoint() 成功
  -> f2fs_clear_prefree_segments(): 清 PRE bitmap + issue discard
```

`set_prefree_as_free_segments()` 在 checkpoint 写 SIT 时就调用 `__set_test_and_free()`，让 free bitmap 看到这些 segment。真正清理 PRE bitmap 和发 discard 在 `f2fs_write_checkpoint()` 的 `do_checkpoint()` 成功之后由 `f2fs_clear_prefree_segments()` 完成。若 checkpoint 失败，会 `f2fs_release_discard_addrs()` 而不是清掉 PRE，避免丢失待释放状态。

### 2.4 free bitmap 语义

`free_segmap` / `free_secmap` 的 bit 语义是：1 表示 in-use，0 表示 free。构建阶段 `build_free_segmap()` 先全置 1，再由 `init_free_segmap()` 根据 SIT 中 `valid_blocks == 0` 的 segment 调 `__set_free()` 清 bit。

几个 helper 的职责：

- `__set_inuse()`：不带检查地把 free bit 置 1，减少 free 计数。
- `__set_test_and_inuse()`：只在原来 free 时扣减计数。
- `__set_free()`：初始化阶段标 free。
- `__set_test_and_free()`：运行时释放，处理 section 计数、current section 排除、next victim 清理。

## 3. checkpoint 与 roll-forward recovery 的关系

### 3.1 checkpoint 写入内容和顺序

`f2fs_write_checkpoint()` 是一致性切点。主流程：

1. 通过 `cp_global_sem` 序列化 checkpoint。
2. `block_operations()` 阻塞新写相关操作，flush quota、dirty node、dirty meta，准备 CP block。
3. 增加 `checkpoint_ver`。
4. `f2fs_flush_nat_entries()`、`f2fs_flush_sit_entries()` 持久化 NAT/SIT 和 SIT journal。
5. `f2fs_save_inmem_curseg()` 暂存 in-memory curseg。
6. `do_checkpoint()` 写 checkpoint pack：curseg segno/blkoff/alloc_type、free segment count、SIT/NAT bitmap、data summaries、node summaries、orphan blocks、CRC 等。
7. `commit_checkpoint()` 写 cp pack 2，并通过 META_FLUSH 提交屏障。
8. 成功后清 fsync node info、清 `SBI_IS_DIRTY/SBI_NEED_CP`，切换 CP pack。
9. `f2fs_clear_prefree_segments()` 清 PRE 和发 discard。

`update_ckpt_flags()` 每次都会设置 `CP_CRC_RECOVERY_FLAG` 并清 `CP_NOCRC_RECOVERY_FLAG`，这也是 roll-forward recovery 识别 fsync node 链和校验版本的重要前提。

### 3.2 roll-forward recovery 扫描起点

`f2fs_recover_fsync_data()` 在 mount/POR 场景下执行。它先拿 `cp_global_sem`，防止 recovery 中途 checkpoint，然后：

```text
find_fsync_dnodes()
  -> 从 CURSEG_WARM_NODE 的 NEXT_FREE_BLKADDR 开始扫描
  -> 沿 node footer 的 next_blkaddr 链前进
  -> 收集带 fsync mark 的 dnode/inode

recover_data()
  -> 再扫一遍 recoverable dnode
  -> recover_inode()
  -> recover_dentry()
  -> do_recover_data()
```

这里的起点不是 checkpoint pack 自身，而是 checkpoint 恢复出的 warm node curseg 的尾部。F2FS 利用 fsync 后追加的 node log 形成 roll-forward 链，把 checkpoint 之后已经 fsync 的修改补回来。

### 3.3 数据块恢复如何修补 SIT/summary

`do_recover_data()` 对 recoverable dnode 中的每个 data index 比较：

- `src`：当前 dnode 中记录的旧地址。
- `dest`：崩溃前 fsync dnode 中记录的新地址。

关键分支：

- `dest == NULL_ADDR`：truncate 当前 `src`。
- `dest == NEW_ADDR`：释放当前块并重新 reserve 一个新块。
- `dest` 是有效地址：先 `check_index_in_prev_nodes()` 清理可能仍引用 `dest` 的旧 dnode index，再 `f2fs_replace_block(..., recover_curseg=false, recover_newaddr=false)` 把 inode 的 data address 改到 `dest`。

`f2fs_replace_block()` 进入 `f2fs_do_replace_block()`，会根据 `recover_curseg` 决定是否临时切换 curseg。recovery 场景 `recover_curseg=false` 时，它只修补 summary、SIT、dirty 状态和 dnode 地址；并且 `recover_newaddr=false` 表示新地址 `dest` 已经是 crash 前写下的块，不需要再次把它当“新分配块”增加 SIT。这个参数组合是 recovery 避免重复计数的核心。

### 3.4 recovery 结束后的 checkpoint

`recover_data()` 成功后调用 `f2fs_allocate_new_segments()`，强制 data curseg 切到新 segment，避免继续写入 recovery 扫描/修补过的旧日志区域。`f2fs_recover_fsync_data()` 清 `SBI_POR_DOING` 后，如果确实做了恢复，则设置 `SBI_IS_RECOVERED` 并以 `CP_RECOVERY` 写一次 checkpoint。这样 roll-forward 的结果成为新的基线，下次 mount 不需要重复 replay。

## 4. GC victim 选择和迁移路径

### 4.1 victim 选择策略

GC 和 SSR 共用 `f2fs_get_victim()`，差别由 `alloc_mode` 和 `gc_type` 控制：

- GC 路径使用 `alloc_mode=LFS`，目标是找需要清理的 dirty section/segment。
- SSR/AT_SSR 使用 `alloc_mode=SSR/AT_SSR`，目标是找可以作为写入目标的 dirty segment。

`select_policy()` 决定搜索位图：

- SSR/AT_SSR：按指定冷热类型搜索 `dirty_segmap[type]`，粒度为 segment。
- LFS GC：小 section 搜 `dirty_segmap[DIRTY]`，大 section 搜 `dirty_secmap`，粒度为 section。

成本函数：

- `GC_GREEDY`：成本等于 valid block 数，越少越优。
- `GC_CB`：cost-benefit，综合 section 利用率和 age，越老且越空越优。
- `GC_AT`：先把候选按 mtime 放入红黑树，再选 older + lower utilization 的 section。
- `SSR`：成本使用 `ckpt_valid_blocks`，确保不会复用 checkpointed 或新近失效仍不安全的块。
- `AT_SSR`：在 age 接近源 segment 的候选中找空洞，降低冷热混杂。

特殊过滤包括：跳过 current section、CP disabled 下不碰 checkpointed data、BG_GC 跳过已登记 victim section、FG_GC 跳过 pinned section。

### 4.2 f2fs_gc 主循环

`f2fs_gc()` 的主循环先判断空间压力。如果 free sections 不足且存在 prefree segments，会先写 checkpoint，把 PRE 转为 free，可能避免昂贵的 foreground GC。

随后：

```text
__get_victim()
  -> f2fs_get_victim(..., LFS)
do_garbage_collect()
  -> 读取 victim SSA summary
  -> 校验 SIT type 与 SSA footer type
  -> node segment: gc_node_segment()
  -> data segment: gc_data_segment()
  -> 迁移完成后提交合并写
```

大 section 下 `next_victim_seg[gc_type]` 允许一次只迁移窗口内的 segment，下次从同一 section 的后续 segment 继续，避免单次 BG_GC 迁移过重。

### 4.3 data 迁移路径

`gc_data_segment()` 分 5 个 phase 扫描 victim summary：

1. phase 0：根据 summary.nid 做 NAT readahead。
2. phase 1：node page readahead。
3. phase 2：通过 `is_alive()` 校验 summary、NAT、node 中的 data blkaddr 是否仍指向 victim block。
4. phase 3：iget inode，检查 pinned、inline data、GC rwsem，预读数据页或 meta inode 临时页。
5. phase 4：真正迁移，普通文件走 `move_data_page()`，需要 meta inode GC 的加密/verity/compressed 路径走 `move_data_block()`。

`move_data_page()`：

- BG_GC 只把 folio 标脏并设置 `folio_set_f2fs_gcing()`，让后续 writeback 走正常 out-of-place 路径。
- FG_GC 同步调用 `f2fs_do_write_data_page()`，旧地址由写路径失效，新地址由 `f2fs_allocate_data_block()` 分配。

`move_data_block()`：

- 直接通过 `META_MAPPING` 读取源块，再调用 `f2fs_allocate_data_block()` 分配目标块。
- 把源块内容 memcpy 到目标 meta folio 并提交写。
- 更新 dnode 指向新地址。
- 若中途失败，调用 `f2fs_do_replace_block(..., from_gc=true)` 回滚 SIT/summary，把块关系恢复到旧地址。

### 4.4 node 迁移要点

虽然本报告重点在 segment/checkpoint/recovery，GC node 路径也要注意：`do_garbage_collect()` 对 node segment 调 `gc_node_segment()`，迁移时需要校验 node footer、NAT 信息和 writeback 状态。node 迁移最终也会走 node page 写入路径，由 `f2fs_do_write_node_page()` 生成 summary 并分配到 node curseg。node segment 与 data segment 共用 victim 选择和 dirty/free 状态机，但 summary 的含义不同：data summary 记录 nid/ofs/version，node summary 主要记录 nid。

## 5. 推荐源码注释点和易错点

### 5.1 推荐注释点

1. `f2fs_allocate_data_block()` 中 `update_sit_entry(new,+1)` 早于 segment 切换的原因：SSR 需要最新 valid/hole 视图，否则可能复用刚分配或刚失效但仍不安全的块。
2. `locate_dirty_segment()` 中 PRE 的条件：`valid_blocks == 0` 并不总能立即 free，必须区分 checkpoint 是否仍引用该 segment。
3. `set_prefree_as_free_segments()` 与 `f2fs_clear_prefree_segments()` 的分工：前者让 checkpoint 中的 free count/free bitmap 生效，后者只在 checkpoint 成功后清 PRE 和发 discard。
4. `f2fs_get_victim()` 的双用途：GC 只选择 victim，不从 dirty list 删除；SSR 选择目标 segment 后由 `change_curseg()` 从 dirty/prefree 中移除。
5. `f2fs_do_replace_block()` 的 `recover_curseg/recover_newaddr/from_gc` 三个布尔参数：不同组合分别服务 recovery、atomic replace、GC rollback，容易误改。
6. `recover_data()` 结束后调用 `f2fs_allocate_new_segments()`：这是为了避开 roll-forward 扫描过的日志尾部，不是单纯的空间整理。
7. `CP_CRC_RECOVERY_FLAG` 在 `update_ckpt_flags()` 中每次设置：它让后续 recovery 可以依赖 CRC/CP version 判断 roll-forward 链。

### 5.2 易错点

- free bitmap bit 语义反直觉：1 是 in-use，0 是 free。
- `valid_blocks` 与 `ckpt_valid_blocks` 不能混用。GC 看当前有效块，SSR/CP-disabled 还必须考虑 checkpoint 视图。
- current segment/current section 不能加入 dirty victim。`is_curseg()` 和 `is_cursec()` 的过滤一旦漏掉，会出现 GC 清理正在写的日志。
- large section 模式下 segment 级和 section 级计数必须同步，尤其是 `sec_entry.valid_blocks`、`dirty_secmap`、`free_secmap`。
- CP disabled 下，释放 checkpointed block 会增加 `unusable_block_count`，不能把这类 hole 当成普通可复用空间。
- recovery 中 `recover_newaddr=false` 不是“不恢复新地址”，而是“新地址已经存在，不再重复增加 SIT 计数”。
- BG_GC 与 FG_GC 的迁移语义不同：BG_GC 可能只是标脏等待回写，FG_GC 更倾向同步迁移以尽快释放空间。
- ATGC/AT_SSR 使用 mtime 和 age threshold，系统时间异常或 mtime 范围变化会影响 victim 排序，代码中多处维护 min/max mtime。
- summary 与 SIT type 必须一致。`do_garbage_collect()` 检查 SSA footer type 与 SIT type，不一致会 stop checkpoint 并要求修复。

## 6. 一句话模型

F2FS 的 segment manager 可以理解为三套状态的同步机：active logs 决定新写落点，SIT 记录块级当前/checkpoint 有效性，dirty/prefree/free bitmap 决定 GC 和空间回收。checkpoint 把这三套状态固化成新的恢复基线；roll-forward recovery 只补 replay checkpoint 之后已 fsync 的 node 链，并在成功后立即 checkpoint，把临时修补转成稳定状态。
