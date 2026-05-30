# F2FS Segment/GC Agent 报告

本报告基于当前仓库源码阅读，重点覆盖 `fs/f2fs/segment.c`、`fs/f2fs/segment.h`、`fs/f2fs/gc.c`、`fs/f2fs/gc.h`、`fs/f2fs/node.c`、`fs/f2fs/node.h`、`fs/f2fs/checkpoint.c`、`fs/f2fs/f2fs.h`；为理解磁盘格式，少量参考了 `include/linux/f2fs_fs.h` 中的 on-disk 结构定义。

## 1. Log-Structured 写入与 segment/section/zone

F2FS 的主区写入总体是 log-structured/out-of-place update：新数据或新 node 通常写到当前 active log 的下一个空块，旧物理块通过 SIT bitmap 失效，后续由 GC 回收。active log 由 `struct curseg_info` 表示，记录 `segno`、`next_blkoff`、`alloc_type`、`seg_type`、summary block、journal 等。默认有 6 个持久 active log：hot/warm/cold data 与 hot/warm/cold node，对应 `CURSEG_HOT_DATA` 到 `CURSEG_COLD_NODE`。

基本空间层级是：

- block：F2FS 的基本 I/O/地址单位。
- segment：连续 block 的集合，`BLKS_PER_SEG(sbi)` 表示每个 segment 的 block 数；常见 4KB block 下，一个 segment 是 512 个 block，即 2MB，但代码使用宏而不是硬编码。
- section：由 `SEGS_PER_SEC(sbi)` 个 segment 组成，GC 在大 section 配置下按 section 选择和迁移。
- zone：由 `secs_per_zone` 个 section 组成，`GET_ZONE_FROM_SEC()`/`GET_ZONE_FROM_SEG()` 用于 zone 定位；在 zoned block device 上，还要考虑 zone capacity，`f2fs_usable_blks_in_seg()` 可能小于完整 segment 大小。

F2FS 有两类主要分配行为：

- LFS：`new_curseg()` 从 free segment/section 中分配新的 current segment，顺序写新块。
- SSR：`change_curseg()` 复用 dirty segment 中的空洞，`get_ssr_segment()` 借用 `f2fs_get_victim()` 找候选 segment，并加载原 summary，避免覆盖仍有效或 checkpoint 保护的块。

## 2. SIT、NAT、SSA、Checkpoint 的角色和结构关系

SIT（Segment Information Table）描述 main area 每个 segment 的有效块状态。磁盘结构 `struct f2fs_sit_entry` 包含 `vblocks`、`valid_map`、`mtime`：`vblocks` 高位保存 segment 类型，低位保存有效块数；`valid_map` 是 segment 内 block 有效性 bitmap；`mtime` 用于 GC age/cost-benefit。内存中 `struct sit_info` 持有 `seg_entry` 数组、`sec_entry` 数组、SIT version bitmap、dirty sentry bitmap、mtime 范围、last victim 等。`update_sit_entry()` 是写入和失效路径更新 SIT 的核心。

NAT（Node Address Table）维护 nid 到 node 物理地址的映射。磁盘结构 `struct f2fs_nat_entry` 包含 `ino`、`block_addr`、`version`；内存中 `struct nat_entry` 包装 `struct node_info`，`struct f2fs_nm_info` 维护 NAT cache、dirty NAT set、free nid cache、NAT version bitmap 和 nat_bits。`f2fs_get_node_info()` 的查找顺序是 NAT cache、current hot data summary 中的 NAT journal、NAT page。dirty NAT 在 checkpoint 时由 `f2fs_flush_nat_entries()` 写入 journal 或 NAT block。

SSA（Segment Summary Area）保存每个 segment 的 summary。`struct f2fs_summary` 对 data block 记录父 dnode 的 nid、`ofs_in_node` 和 node version；对 node block 记录 node 的 nid。summary footer 标记 DATA/NODE。GC 依赖 SSA 找到 victim block 的逻辑归属，再用 NAT 和 node page 交叉验证。当前 summary block 的 spare/journal 区还能暂存 NAT/SIT journal，因此 SSA 同时连接写入日志、GC 反查和 checkpoint metadata flush。

Checkpoint 是一致性切点。`struct f2fs_checkpoint` 记录 checkpoint version、有效 block/node/inode 计数、free segment 计数、当前 data/node curseg 的 `segno` 与 `blkoff`、各 curseg 的 `alloc_type`、SIT/NAT version bitmap、summary 起点、flags 等。挂载时 `f2fs_get_valid_checkpoint()` 校验两个 checkpoint pack 的 CRC 和 version，选择较新的有效 pack；随后 segment manager 和 node manager 用 checkpoint 中的 bitmap、curseg、summary 信息重建 SIT/NAT/free/dirty 状态。

结构关系可以简化为：

```text
checkpoint pack
  -> 当前 curseg 位置、alloc_type、summary 位置
  -> SIT/NAT version bitmap
  -> data/node summaries、orphan、flags、计数

SIT
  -> segment/block 有效性、类型、mtime
  -> free/dirty/prefree 判断和 GC victim cost

NAT
  -> nid -> node block 地址/version
  -> GC 验证 node 是否仍指向 victim block

SSA
  -> segment block -> owner nid/ofs/version
  -> GC 从物理块反推逻辑块
```

## 3. Segment 分配和写入路径关键函数

初始化路径：

- `f2fs_build_segment_manager()` 分配 `f2fs_sm_info`，设置 `seg0_blkaddr`、`main_blkaddr`、`ssa_blkaddr`、reserved/overprovision 等参数。
- `build_sit_info()` 分配 SIT cache、valid bitmap、dirty bitmap、section cache，并从 checkpoint 取得 SIT bitmap。
- `build_curseg()` 分配各 active log 的 `curseg_info`，恢复 summary。
- `build_sit_entries()` 从当前 SIT block 和 cold data curseg 的 SIT journal 重建 `seg_entry`，并校验 valid node/data 计数。
- `init_free_segmap()`、`build_dirty_segmap()` 根据 SIT 建立 free、dirty、prefree 位图。

普通 out-place 写入路径：

- data：`f2fs_outplace_write_data()` 生成 data summary，然后调用 `do_write_page()`，最后更新 dnode 中的数据块地址。
- node：`f2fs_do_write_node_page()` 生成 node summary，然后调用 `do_write_page()`；NAT 地址变更由 node 写回路径中的 `set_node_addr()`/相关 node manager 逻辑维护。
- `do_write_page()` 调用 `f2fs_allocate_data_block()` 取得新物理地址，然后提交 page write。

`f2fs_allocate_data_block()` 是 segment 分配和 SIT 更新的中心：

- 锁定 `curseg_lock`、`curseg_mutex` 和 `sit_i->sentry_lock`。
- 用 `NEXT_FREE_BLKADDR()` 计算新地址，把 summary 写入当前 curseg 的 `sum_blk[next_blkoff]`。
- LFS 下递增 `next_blkoff`；SSR 下用 `f2fs_find_next_ssr_block()` 找下一个可复用空洞。
- 更新新旧地址的 mtime 和 SIT：`update_sit_entry(new, +1)`、`update_sit_entry(old, -1)`。
- 如果当前 segment 满了，非 GC 写入通过 `need_new_seg()` 决定 `new_curseg()` 或 `change_curseg()`；ATGC/GC 写入走 `get_atssr_segment()` 或新 cold segment。
- 最后 `locate_dirty_segment()` 更新 dirty/prefree 状态。

segment 选择相关函数：

- `get_new_segment()` 从 free section/segment bitmap 中找新 segment，并处理 section hint、zone 互斥、zoned device 和 pinned section 等情况。
- `new_curseg()` 写出旧 summary，调用 `get_new_segment()`，`reset_curseg()` 后设为 LFS。
- `change_curseg()` 用 SSR 复用 dirty segment，移除 dirty/prefree 标记并从 SSA 读取旧 summary。
- `get_ssr_segment()` 通过 `f2fs_get_victim()` 找可 SSR 的 segment，优先同类型，失败后尝试邻近 hot/warm/cold 类型。

## 4. GC 线程、f2fs_gc、victim 选择和 valid block 迁移

后台 GC 由 `f2fs_start_gc_thread()` 创建 `gc_thread_func()`。线程按 sleep interval 唤醒，综合 urgent GC、`GC_MERGE`、zoned device free space、IO idle、dirty segment 和 free section 情况决定是否执行。它设置 `f2fs_gc_control`，以 `BG_GC` 或 `FG_GC` 调用 `f2fs_gc()`。前台 GC 通常来自空间不足或 `f2fs_balance_fs()` 触发，优先保障可用空间。

`f2fs_gc()` 的主流程：

- 检查文件系统 active、checkpoint error、free section 是否不足。
- 如果 free section 不足且存在 prefree segment，先调用 `f2fs_write_checkpoint()`，因为 prefree 只有 checkpoint 后才能真正成为 free。
- 通过 `__get_victim()` 获取 victim segment/section。
- 调用 `do_garbage_collect()` 读取 SSA summary，迁移有效块。
- 如果仍未达到 free section 目标，可能继续 GC；在 free section 接近 checkpoint 所需余量时再次 checkpoint。

victim 选择由 `f2fs_get_victim()` 完成：

- `select_policy()` 根据 GC 类型和 alloc mode 选择策略。SSR/AT_SSR 固定用 greedy；后台 GC 通常用 GC_CB，启用 ATGC 时用 GC_AT；前台 GC 用 GC_GREEDY。
- dirty bitmap 来源取决于粒度：普通 segment 用 `dirty_segmap[DIRTY]`，large section 用 `dirty_secmap`；SSR 按具体 hot/warm/cold 类型 dirty map 找 segment。
- cost 函数：GC_GREEDY 选择有效块少的 victim；GC_CB 使用 utilization 与 age；GC_AT 建立按 mtime 排序的 victim rb-tree，再按 age/valid ratio 取候选。
- 会跳过 current section、pinned section、已被 BG GC 标记的 section、invalid_segmap，以及 CP_DISABLED 下仍有 checkpointed data 的 section。

`do_garbage_collect()` 先按 victim section 范围读取 SSA summary page，检查 victim 不是 current section，并确认 SSA summary footer 类型和 SIT 类型一致。之后按 segment 类型分流：

- node segment：`gc_node_segment()` 分三阶段处理。先看 SIT valid map，再预读 NAT，再预读 node page；最终读取 node page 和 NAT，确认 `ni.blk_addr == victim_addr`，有效则调用 `f2fs_move_node_folio()` 以 cold/GC 语义重写 node。
- data segment：`gc_data_segment()` 分五阶段处理。先 valid map，再 NAT 预读，再 node 预读；`is_alive()` 通过 summary 找到 dnode，检查 node version、nid、`ofs_in_node` 和 node page 中的数据块地址是否仍等于 victim block；随后读取 inode/data page；最后 `move_data_page()` 或 `move_data_block()` 迁移。

data 迁移有两条路径：

- `move_data_page()`：普通文件数据。BG_GC 多数情况下标脏并设置 gcing，让后续写回迁移；FG_GC 构造同步 `f2fs_io_info` 调用数据写路径。
- `move_data_block()`：用于需要通过 `META_MAPPING` 搬迁物理块内容的场景，例如加密、verity 或压缩相关路径。它读取旧块，调用 `f2fs_allocate_data_block()` 分配新块，写新块，再更新 dnode；失败时用 `f2fs_do_replace_block()` 回滚块映射。

## 5. Checkpoint 与 GC、恢复、一致性的关系

Checkpoint 是 F2FS 把“内存中的 log/SIT/NAT/summary 状态”固化成可恢复一致点的过程。`f2fs_write_checkpoint()` 先拿 `cp_global_sem`，`block_operations()` 冻结关键 FS 操作并同步 inline data、dirty dentry、inode meta、dirty node；随后 flush merged writes，递增 checkpoint version，再依次 `f2fs_flush_nat_entries()`、`f2fs_flush_sit_entries()`、`f2fs_save_inmem_curseg()`、`do_checkpoint()`。

`do_checkpoint()` 会：

- 同步已有 NAT/SIT metadata page。
- 更新 elapsed time、free segment count、当前 curseg 的 segno/blkoff/alloc_type。
- 计算 data summary block 数、orphan block 数和 checkpoint pack 总长度。
- 更新 CP flags、SIT/NAT bitmap、checkpoint checksum。
- 写 checkpoint pack：CP block、payload、orphan blocks、data summaries、必要时 node summaries。
- flush meta、等待 dirty meta/CP data、flush device cache。
- 最后通过 `commit_checkpoint()` 写 pack 末尾 checkpoint block，完成 barrier/flush 语义；成功后切换下一 checkpoint pack。

GC 和 checkpoint 的关系很紧：

- GC 迁移有效块后，旧 segment 的 valid block 变为 0 时会进入 PRE/prefree，而不是立即 free。
- `f2fs_clear_prefree_segments()` 在 checkpoint 成功后清理 PRE bitmap，必要时发 discard，使这些 segment 真正回到 free pool。
- 因此 `f2fs_gc()` 在空间不足、prefree segment 存在或 checkpoint 所需空间紧张时，会主动调用 `f2fs_write_checkpoint()`。
- CP_DISABLED 模式下，GC/victim/SSR 对 checkpointed data 有额外保护，代码明确避免复用旧 checkpoint 中仍需恢复引用的块，并用 `unusable_block_count` 跟踪不可复用空间。

恢复角度上，挂载时从两个 checkpoint pack 中选择 CRC 和 version 都有效的最新 pack；然后根据 CP 中的 SIT/NAT bitmap、summary、curseg 位置重建 segment 和 node manager。`CP_CRC_RECOVERY_FLAG` 在 checkpoint 中被设置，用于恢复路径识别带 CRC/version 的信息。更完整的 roll-forward 细节需要结合 `fs/f2fs/recovery.c` 继续阅读，本报告只基于 checkpoint、segment、node、GC 文件给出关系。

## 6. 修改 segment/GC 代码的主要风险

- SIT 计数或 bitmap 错误：`valid_blocks`、`ckpt_valid_blocks`、`cur_valid_map`、`ckpt_valid_map` 不一致会导致误判 free/dirty/prefree，轻则空间泄漏，重则覆盖有效数据。
- NAT 更新错误：nid 到 node 地址/version 不一致会让 GC 迁移错误 node，或恢复后找不到正确 node。
- SSA 与 SIT 类型不一致：GC 用 SSA 反查 owner，如果 summary footer 或 entry 错误，会把物理块归属解析错；代码对此会 stop checkpoint 或标记 fsck。
- SSR 复用风险：SSR 必须避开 checkpointed 或 newly invalidated block。改动 `change_curseg()`、`__next_free_blkoff()`、`ckpt_valid_map` 更新逻辑很容易破坏恢复语义。
- checkpoint 顺序风险：NAT/SIT flush、summary 写入、bitmap 更新、checksum、device flush 和最后 CP block 的顺序共同定义 crash consistency，不能随意调换。
- GC 并发风险：GC 与 checkpoint、writeback、truncate、direct I/O、node write 共享多把锁；改动 `gc_lock`、`cp_global_sem`、`sentry_lock`、`curseg_mutex`、`i_gc_rwsem` 顺序可能引入死锁或竞态。
- zoned device 风险：zone write pointer、zone capacity、section alignment、discard/reset 行为都参与分配和 GC；普通块设备上通过的改动可能在 zoned 设备上破坏顺序写约束。
- pinned/compressed/encrypted/verity 文件风险：GC 有特殊迁移路径和跳过逻辑，错误处理会造成迁移失败循环、数据块内容丢失或需要 fsck。
- prefree/free/discard 风险：GC 后的 segment 不能在 checkpoint 前随意变 free；discard map 错误可能误丢还需要恢复的数据。
- 性能和空间抖动风险：victim cost、`max_victim_search`、ATGC age 参数、BG/FG GC 切换和 checkpoint 触发阈值会影响写放大、延迟和 ENOSPC 行为。

## 7. 不确定点和后续需要确认的内容

- 本次没有系统阅读 `fs/f2fs/recovery.c`，所以 roll-forward recovery 如何消费 CP flags、fsync node list 和 NAT/SIT 状态仍需单独确认。
- `__get_segment_type()` 的冷热分类策略只在写入路径中粗略梳理，具体与文件属性、写回上下文、temperature policy 的交互还需要结合 data/file 路径继续看。
- packed SSA、large NAT bitmap、nat_bits 在不同 mkfs feature 组合下的精确磁盘布局，需要结合格式化参数和实际镜像验证。
- ATGC 的运行时调参、sysfs knob 与 workload 的实际效果未做实验，本报告只描述源码中的选择机制。
- CP_DISABLED 模式涉及 enable/disable checkpoint、unusable space、快速禁用等复杂状态，本报告只覆盖 segment/GC 中直接出现的保护逻辑，不能视为完整语义说明。
