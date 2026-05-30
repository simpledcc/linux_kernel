# F2FS 数据 I/O、Node/NAT、Extent Cache 与压缩路径深度学习报告

> 子 agent 范围：`fs/f2fs/data.c`、`file.c`、`node.c`、`node.h`、`extent_cache.c`、`compress.c`、`f2fs.h` 中的数据 I/O、page cache、writeback、node/NAT、extent cache、压缩路径。本文只做学习分析，不修改源码。

## 0. 总览：从 VFS 页缓存到 F2FS 物理块

F2FS 的普通文件数据路径可以拆成三层：

1. **VFS/page cache 层**：用户态 `read/write` 进入 `filemap` 或 `generic_perform_write()`，F2FS 通过 `f2fs_dblock_aops` 接管 `.read_folio`、`.readahead`、`.write_begin`、`.write_end`、`.writepages`。
2. **逻辑块映射层**：`f2fs_map_blocks()` 或 `f2fs_get_dnode_of_data()` 把文件逻辑页号 `fofs/lblk` 映射到 dnode 中的地址槽，再得到 `NULL_ADDR`、`NEW_ADDR`、`COMPRESS_ADDR` 或真实物理块。
3. **落盘与元数据层**：数据块写入由 `f2fs_do_write_data_page()` 决定 IPU/OPU；node 页写回由 `f2fs_sync_node_pages()`、`__write_node_folio()` 更新 NAT 中的 node 物理地址。数据块地址写进 node 页，node 页地址写进 NAT，这是 F2FS 两级映射的核心。

关键特殊值：

- `NULL_ADDR`：逻辑块为空洞，没有数据块。
- `NEW_ADDR`：已预留逻辑块/额度，但尚未分配真实物理块，典型于 buffered write 的延迟分配阶段。
- `COMPRESS_ADDR`：压缩 cluster 的头部标记，第 0 个槽不是普通数据块，后续槽记录压缩页物理块。
- valid blkaddr：真实物理地址，可提交 BIO。

## 1. Buffered Read / Write / Writeback 路径

### 1.1 buffered read

入口在 `f2fs_dblock_aops`：

```text
VFS/filemap
  -> f2fs_read_data_folio() 或 f2fs_readahead()
    -> f2fs_mpage_readpages()
      -> 普通文件: f2fs_read_single_page()
      -> 压缩文件: f2fs_read_multi_pages()
        -> f2fs_map_blocks() / f2fs_get_dnode_of_data()
        -> f2fs_submit_page_read() / f2fs_grab_read_bio()
        -> f2fs_read_end_io()
```

`f2fs_read_data_folio()` 先检查压缩后端是否可用，再处理 inline data；inline data 命中时不进入块映射。普通块读走 `f2fs_mpage_readpages()`，它复用 `struct f2fs_map_blocks` 的上一次结果以减少重复查 node。`f2fs_read_single_page()` 的关键逻辑是：

- 若页号越过 EOF，直接 zero + uptodate。
- 若已有连续映射，按 `map->m_pblk + index - map->m_lblk` 得物理块。
- 映射为空洞时 zero + uptodate。
- 映射有效时检查 `f2fs_is_valid_blkaddr()`，等待同一物理块 writeback，合并 BIO 后提交读。

读路径的性能优化点有两个：

- `f2fs_map_blocks()` 先查 read extent cache，命中后不必读 node 页。
- readahead 会把多个 folio 聚合成连续 BIO；压缩文件则以 cluster 为单位攒页，避免对每个页重复解压。

### 1.2 buffered write

文件写入口在 `f2fs_file_write_iter()`：

```text
f2fs_file_write_iter()
  -> f2fs_write_checks()
  -> f2fs_preallocate_blocks(..., dio=false)
       -> f2fs_map_blocks(..., F2FS_GET_BLOCK_PRE_AIO)
  -> f2fs_buffered_write_iter()
       -> generic_perform_write()
            -> f2fs_write_begin()
            -> copy_from_iter()
            -> f2fs_write_end()
```

`f2fs_file_write_iter()` 负责全局检查：checkpoint error、压缩后端、NOWAIT、pinned file 覆盖写限制、通用写检查、是否走 DIO、是否需要预分配。buffered write 最终进入 `generic_perform_write()`，由 address_space_operations 回调 F2FS。

`f2fs_write_begin()` 的职责：

- 确认 checkpoint 状态，必要时转换 inline inode。
- 对压缩文件的部分覆盖写调用 `f2fs_prepare_compress_overwrite()`，先把整个 cluster 读入页缓存再改。
- 获取/创建目标 folio。
- `prepare_write_begin()` 或 atomic 写路径准备映射，必要时得到旧块地址。
- 若写入不覆盖整页且 folio 不是 uptodate，需要从旧物理块读入后再局部修改；若块是 `NEW_ADDR`，直接 zero。

`f2fs_write_end()` 的职责：

- 确认 folio uptodate。
- 压缩覆盖写通过 `f2fs_compress_write_end()` 标记整个 cluster dirty。
- 普通写则 `folio_mark_dirty()`，必要时标记 atomic，更新 `i_size` 和 COW inode size。

注意：buffered write 的用户数据通常先停在 page cache，`write_end()` 只标脏，不一定马上分配最终物理块。真正物理分配和 node 地址更新多发生在 writeback。

### 1.3 data writeback

数据写回入口：

```text
writeback
  -> f2fs_write_data_pages()
    -> __f2fs_write_data_pages()
      -> f2fs_write_cache_pages()
        -> 普通文件: f2fs_write_single_data_page()
             -> f2fs_do_write_data_page()
        -> 压缩文件: f2fs_write_multi_pages()
             -> f2fs_write_compressed_pages() 或 f2fs_write_raw_pages()
```

`__f2fs_write_data_pages()` 先按 dirty 页数量、POR、目录/配额、`FI_SKIP_WRITES`、同步写优先级等条件决定是否跳过；对普通文件还可能通过 `sbi->writepages` 串行化，降低混合 writeback 的碎片和死锁风险。

`f2fs_write_cache_pages()` 类似内核通用 `write_cache_pages()`，但 F2FS 做了冷热数据、压缩 cluster、EAGAIN retry、WB_SYNC 优先级等定制。它按 dirty tag 扫 page cache，锁 folio，等待 writeback，清 dirty，然后进入：

- 普通文件：`f2fs_write_single_data_page()`。
- 压缩文件：先按 cluster 收集页，最后 `f2fs_write_multi_pages()`。

`f2fs_write_single_data_page()` 做 EOF 裁剪、inline data 写入、目录/配额特殊锁，然后调用 `f2fs_do_write_data_page()`。

`f2fs_do_write_data_page()` 是普通数据写回的核心：

1. 构造 `dnode_of_data`，atomic commit 时使用 COW inode。
2. 若需要 IPU 且 read extent cache 命中旧块，则可跳过 dnode 查询，强制原地写。
3. 否则 `f2fs_get_dnode_of_data()` 找到 node 页和旧数据块地址。
4. 若旧地址为空，说明页被截断，清 uptodate/gcing 后退出。
5. 根据 `need_inplace_update()` 决定：
   - **IPU**：加密后 `folio_start_writeback()`，调用 `f2fs_inplace_write_data()`，node 地址不变。
   - **OPU/LFS**：获取 node version，加密后 `f2fs_outplace_write_data()`，分配新物理块并更新 node 页地址，设置 `FI_APPEND_WRITE`。

data writeback 完成后，F2FS 会提交合并 BIO：普通 OPU 走 `f2fs_submit_merged_write_cond(..., DATA)`，IPU 缓存 BIO 走 `f2fs_submit_merged_ipu_write()`。

### 1.4 node writeback 与数据写回的关系

数据块地址记录在 node 页里，因此 data writeback 可能把 node 页弄脏；node 页本身的物理位置再由 NAT 记录。node 写回入口是 `f2fs_node_aops.writepages = f2fs_write_node_pages()`：

```text
f2fs_write_node_pages()
  -> f2fs_sync_node_pages()
    -> __write_node_folio()
      -> f2fs_get_node_info()      // 从 NAT 得旧 node 物理地址
      -> f2fs_do_write_node_page() // 写新 node 块
      -> set_node_addr()           // 更新 NAT cache/journal
```

`f2fs_sync_node_pages()` 分三步刷 node：

- step 0：indirect nodes；
- step 1：dentry dnodes；
- step 2：file dnodes。

这样做是为了控制元数据依赖和恢复语义。`__write_node_folio()` 会设置 fsync/dentry mark，写 node 页后调用 `set_node_addr()` 更新 NAT 中该 nid 的物理地址。也就是说：

- data writeback：更新“文件逻辑块 -> 数据物理块”的地址槽。
- node writeback：更新“nid -> node 物理块”的 NAT 映射。

## 2. Logical Block 到 Node/NAT 的映射路径

### 2.1 基本结构

F2FS 文件逻辑块并不直接映射到数据物理块，而是先映射到 inode/direct node/indirect node 中的地址槽。`f2fs.h` 中：

- `blkaddr_in_node()` 根据 node 是否 inode node 选择 `node->i.i_addr` 或 `node->dn.addr`。
- `get_dnode_addr()` 在 inode node 中跳过 extra attr / inline xattr 区域。
- `addrs_per_page()` 对压缩文件按 cluster size 对齐，避免一个 dnode 中出现跨 cluster 的地址尾巴。

### 2.2 `get_node_path()`：逻辑块定位到 node 层级

`node.c:get_node_path()` 把文件逻辑块号拆成最多 4 层路径：

- inode node 内直接地址；
- 两个 direct node；
- 两个 indirect node；
- 一个 double indirect node。

输出：

- `offset[]`：每层在父 node 中的 nid 或 block 地址槽偏移。
- `noffset[]`：node footer 中记录的 node offset。
- 返回 `level`：0 表示 inode node 地址槽，1 表示 direct node，2/3 表示 indirect/double-indirect。

### 2.3 `f2fs_get_dnode_of_data()`：沿路径取/建 dnode

调用链典型形态：

```text
f2fs_map_blocks() / f2fs_do_write_data_page() / truncate / fiemap
  -> set_new_dnode()
  -> f2fs_get_dnode_of_data(dn, index, LOOKUP_NODE/ALLOC_NODE/LOOKUP_NODE_RA)
       -> get_node_path()
       -> f2fs_get_inode_folio()
       -> 逐层 get_nid()
       -> 必要时 f2fs_alloc_nid() + f2fs_new_node_folio()
       -> f2fs_get_node_folio()
       -> dn->node_folio / dn->ofs_in_node / dn->data_blkaddr
```

不同 mode 的语义：

- `LOOKUP_NODE`：只查已有 node；缺失返回 `-ENOENT`，读空洞、fiemap、writeback 常用。
- `ALLOC_NODE`：路径上缺 node 时分配 nid 并创建 node 页；预留/写入扩展路径常用。
- `LOOKUP_NODE_RA`：查最后一级 node 时可做 sibling readahead，截断/预读映射等路径使用。

`f2fs_get_dnode_of_data()` 返回后，`dn->data_blkaddr` 是目标逻辑块当前记录值。后续：

- `f2fs_data_blkaddr(dn)` / `data_blkaddr()` 读取槽。
- `f2fs_set_data_blkaddr()` 写槽并 dirty node 页。
- `f2fs_update_data_blkaddr()` 写槽后同步更新 read extent cache。

### 2.4 NAT：nid 到 node 物理块

NAT 只管理 node 的物理位置，不直接管理普通数据块。关键函数是 `f2fs_get_node_info()`：

```text
f2fs_get_node_info(sbi, nid, &ni, checkpoint_context)
  -> nat cache
  -> current segment NAT journal
  -> NAT block page
  -> sanity check
  -> cache_nat_entry()
```

它返回 `struct node_info`：

- `nid`：node id；
- `ino`：所属 inode；
- `blk_addr`：该 node 页当前物理地址；
- `version`：用于 summary 和恢复校验。

数据写 OPU 时也会调用 `f2fs_get_node_info(sbi, dn.nid, &ni, false)`，但目的不是找数据块，而是拿当前 dnode 的 NAT version，写入 data summary。node 写回时 `__write_node_folio()` 用 `f2fs_get_node_info()` 得旧 node 块地址，然后写出新 node 块，并用 `set_node_addr()` 更新 NAT。

### 2.5 `f2fs_map_blocks()`：统一映射接口

`f2fs_map_blocks()` 是 DIO、read、fiemap、bmap、预分配、预缓存的统一逻辑块映射接口。核心行为：

- 非创建路径先查 read extent cache；命中可直接填 `m_pblk/m_len/m_flags`。
- 按 dnode 批量扫描地址槽，合并连续物理块。
- `map->m_may_create` 为真时可分配或预留：
  - `F2FS_GET_BLOCK_PRE_AIO`：批量把 `NULL_ADDR` 预留成 `NEW_ADDR`。
  - `F2FS_GET_BLOCK_PRE_DIO` / `F2FS_GET_BLOCK_DIO`：调用 `__allocate_data_block()` 分配真实物理块。
- 空洞在不同 flag 下语义不同：
  - read/default：返回未映射，调用者 zero-fill。
  - bmap：返回 0。
  - fiemap：可报告 delalloc 或跳过空洞。
  - DIO read：可用 `m_next_pgofs` 通知下一个可能非空洞位置。

因此，`f2fs_map_blocks()` 不是简单查表；它同时承载 extent cache、node 查找、预分配、DIO 多设备映射、压缩 cluster sanity check、物理块合法性检查等职责。

## 3. Extent Cache / 压缩与普通 I/O 的关系

### 3.1 read extent cache

read extent cache 用 rb-tree 缓存连续的“文件逻辑块区间 -> 物理块区间”。结构在 `f2fs.h`：

- `extent_tree`：每 inode 每类型一棵树，带最近访问 `cached_en` 和最大 extent `largest`。
- `extent_node`：rb-tree 节点，同时挂到全局 LRU。
- `extent_info`：`fofs/len/blk`，压缩启用时还有 `c_len`。

查找路径：

```text
f2fs_map_blocks()
  -> f2fs_map_blocks_cached()
    -> f2fs_lookup_read_extent_cache()
      -> __lookup_extent_tree()
```

更新路径：

- OPU/IPU 改变地址后：`f2fs_update_data_blkaddr()` -> `f2fs_update_read_extent_cache()`。
- precache/fiemap 类扫描：`f2fs_update_read_extent_cache_range()`。
- 只读压缩镜像：`f2fs_update_read_extent_tree_range_compressed()`。

extent cache 对普通读和 DIO 映射很重要：命中后可以绕过 node 页读取。但写路径必须谨慎失效/更新，否则会读到旧物理块。

### 3.2 block age extent cache

`EX_BLOCK_AGE` 与读映射不同，它记录块年龄和最近分配计数，用于冷热数据/段选择策略。它不直接参与 `f2fs_map_blocks()` 返回物理映射，但在数据块更新时经由 `__update_extent_cache(..., EX_BLOCK_AGE)` 维护。压缩文件、cold file 通常不启用该类型，以免年龄统计失真。

### 3.3 压缩文件对 extent cache 的限制

`__may_extent_tree()` 对压缩文件有特殊限制：

- 普通可读写压缩文件通常不创建 read extent cache。
- 只读压缩场景可以维护压缩 extent。
- block age extent 对压缩文件禁用。

原因是压缩 cluster 的逻辑长度和物理长度不一致，普通 extent 的“逻辑连续 == 物理连续且等长”假设容易被破坏。压缩 extent 使用 `c_len` 表示压缩后的物理块数，并且 `__is_extent_mergeable()` 对压缩 extent 合并非常保守：如果 `c_len` 存在且不等于逻辑长度，就不与相邻 extent 合并。

### 3.4 压缩读路径

压缩读在 `f2fs_mpage_readpages()` 中分流：

```text
f2fs_mpage_readpages()
  -> f2fs_is_compressed_cluster()
  -> f2fs_init_compress_ctx()
  -> f2fs_compress_ctx_add_page()
  -> f2fs_read_multi_pages()
       -> 读取 COMPRESS_ADDR 后续压缩块
       -> f2fs_alloc_dic()
       -> BIO post-read STEP_DECOMPRESS
       -> f2fs_decompress_end_io()
```

cluster 第 0 个槽必须是 `COMPRESS_ADDR`。`f2fs_read_multi_pages()` 从 dnode 或 read extent cache 找到压缩块地址，读取压缩页到临时 page，然后 post-read 解压，把解压后的内容填回原 page cache folio。普通页缓存看到的仍是解压后的文件页。

### 3.5 压缩写路径

压缩写有两类：

1. **部分覆盖写**：`f2fs_write_begin()` 调 `f2fs_prepare_compress_overwrite()`，若目标 cluster 已压缩，则先读入并解压整个 cluster，锁住所有 cluster 页。`f2fs_write_end()` 再通过 `f2fs_compress_write_end()` 把整个 cluster 标脏。
2. **writeback 压缩**：`f2fs_write_cache_pages()` 按 cluster 聚集 dirty 页，`f2fs_write_multi_pages()` 决定是否压缩：
   - `cluster_may_compress()` 为真时，先 `f2fs_compress_pages()`。
   - 压缩成功走 `f2fs_write_compressed_pages()`。
   - 压缩失败或收益不足走 `f2fs_write_raw_pages()`。

`f2fs_write_compressed_pages()` 的关键动作：

- 锁操作区，拿起始 dnode。
- 确认 cluster 所有槽非 `NULL_ADDR`。
- 第 0 个槽写成 `COMPRESS_ADDR`。
- 第 1..N 个槽写压缩页物理块。
- 多余旧数据块失效并写成 `NEW_ADDR`。
- 更新 inode 的 compressed block 计数。
- 原始 page cache 页在压缩 BIO 完成后统一 end writeback。

### 3.6 压缩与普通 I/O 的边界

- 普通读写以单页/连续物理块为单位；压缩读写以 cluster 为单位。
- 普通 extent cache 假设逻辑长度和物理长度相同；压缩 extent 必须带 `c_len`，且只在受限场景启用。
- 普通 overwrite 可以只改一页；压缩 overwrite 要读-改-写整个 cluster，否则会破坏压缩流。
- DIO、swap、clone、pinned file、atomic write 与压缩经常互斥或受限，因为这些功能要求稳定、可直接定位、页级粒度的物理块。

## 4. 推荐源码中文学习注释位置

建议只加“解释分叉原因和不变量”的注释，避免逐行翻译。以下是高价值位置：

1. `fs/f2fs/data.c:f2fs_read_data_folio()`
   说明 buffered read 缺页入口、inline data、verity、压缩后端检查，以及最终进入 `f2fs_mpage_readpages()`。

2. `fs/f2fs/data.c:f2fs_mpage_readpages()`
   说明普通 read 与压缩 read 的分流，`map` 复用、readahead BIO 聚合、cluster 聚合的差异。

3. `fs/f2fs/data.c:f2fs_read_single_page()`
   说明空洞 zero-fill、extent/node 映射、等待同物理块 writeback、防止读旧数据的原因。

4. `fs/f2fs/file.c:f2fs_file_write_iter()`
   说明用户写入口中 checkpoint、NOWAIT、DIO/buffered 判定、预分配与 truncate 清理的顺序。

5. `fs/f2fs/data.c:f2fs_write_begin()` / `f2fs_write_end()`
   说明 page cache 写入的两阶段协议：`write_begin` 准备 folio 和旧数据，`write_end` 标脏和更新 i_size；压缩覆盖写为什么要整 cluster 准备。

6. `fs/f2fs/data.c:f2fs_write_cache_pages()`
   说明它为何复制并改造通用 write_cache_pages：冷热数据、WB_SYNC 优先级、压缩 cluster 聚合、EAGAIN 重试。

7. `fs/f2fs/data.c:f2fs_do_write_data_page()`
   说明 IPU/OPU 决策、read extent cache 快速路径、node version 用于 summary、OPU 后 node 地址被更新。

8. `fs/f2fs/data.c:f2fs_map_blocks()`
   说明它是统一映射接口，不只是查表；不同 flag 的空洞、预分配、DIO、fiemap 行为不同。

9. `fs/f2fs/node.c:get_node_path()`
   说明 F2FS 文件逻辑块如何分布到 inode/direct/indirect/double-indirect node。

10. `fs/f2fs/node.c:f2fs_get_dnode_of_data()`
    说明 LOOKUP/ALLOC/RA 三种模式、何时分配 nid、返回的 `dn->node_folio/ofs_in_node/data_blkaddr` 是后续 I/O 的核心上下文。

11. `fs/f2fs/node.c:f2fs_get_node_info()`
    说明 NAT 查询顺序：nat cache -> journal -> NAT block；以及它返回的是 node 位置，不是数据块位置。

12. `fs/f2fs/node.c:__write_node_folio()`
    说明 node 写回后为什么要 `set_node_addr()` 更新 NAT，以及 fsync/dentry mark 对恢复的意义。

13. `fs/f2fs/extent_cache.c:__may_extent_tree()`
    说明压缩文件、只读镜像、block age cache 的启用/禁用条件。

14. `fs/f2fs/extent_cache.c:__update_extent_tree_range()`
    说明 extent 更新时的覆盖、分裂、合并、largest extent 维护。

15. `fs/f2fs/compress.c:f2fs_read_multi_pages()`
    说明 `COMPRESS_ADDR`、压缩页读取、post-read 解压、page cache 原页解锁的关系。

16. `fs/f2fs/compress.c:f2fs_write_multi_pages()` / `f2fs_write_compressed_pages()`
    说明压缩收益不足时回退 raw write，以及压缩 cluster 的地址槽布局。

17. `fs/f2fs/f2fs.h:struct f2fs_map_blocks`、`struct extent_info`、`f2fs_compressed_file()`、`addrs_per_page()`
    这些结构/辅助函数是理解 data/node/compress 交叉关系的入口，适合加字段级中文注释。

注意：如果在 PowerShell 默认编码下直接读取源码，中文学习注释可能显示为乱码；当前文件内容用 UTF-8 显式读取是可读的。后续继续补注释时建议统一保持 UTF-8，避免在审查或网页展示时产生误判。

## 5. 常见风险与排查要点

### 5.1 空洞、`NEW_ADDR`、真实物理块语义混淆

`NULL_ADDR` 是空洞，`NEW_ADDR` 是预留未落盘，真实物理块才可读。读路径遇 `NEW_ADDR` 有时 zero-fill，有时说明元数据状态特殊；fiemap/bmap/DIO 对空洞的处理也不同。修改 `f2fs_map_blocks()` 时最容易把这些语义混掉。

### 5.2 extent cache 陈旧导致读旧块

OPU 会把逻辑块从旧物理块迁到新物理块；如果更新 node 地址后没有同步失效/更新 read extent cache，后续读可能绕过 node 页命中旧映射。相关路径要重点看 `f2fs_update_data_blkaddr()`、`f2fs_update_read_extent_cache()`、truncate/punch hole 的 extent invalidation。

### 5.3 IPU/OPU 决策与写回竞争

IPU 依赖旧块稳定，OPU 依赖 node 地址更新。`f2fs_do_write_data_page()` 中有 checkpoint、GC、atomic、compressed、LFS、SSR、pinned file 等条件。任何新增条件都可能影响恢复语义、冷热分离、discard 或加密 BIO 合并。

### 5.4 page lock、node lock、operation lock 死锁

源码中明确提示了 data block address 修改的锁顺序：data page -> node folio -> 更新 node 地址。`f2fs_write_begin()` 也避免 inline inode 转换时 inode page 与 page #0 死锁。涉及 `f2fs_lock_op()`、`node_write`、`i_gc_rwsem`、folio lock 的改动必须重新审视锁顺序。

### 5.5 压缩 cluster 被当成普通页处理

压缩 cluster 的第 0 槽是 `COMPRESS_ADDR`，后续槽是压缩物理块；逻辑页和物理页数量不一致。若 truncate、fiemap、bmap、DIO、swap、clone、writeback 中误按普通连续块处理，可能造成数据损坏或错误暴露物理映射。

### 5.6 部分覆盖压缩写的读-改-写风险

压缩文件局部写必须准备整个 cluster。`f2fs_prepare_compress_overwrite()` 需要读入缺失页、等待 writeback、锁住 cluster 页。若中途失败没有正确 unlock/put/destroy ctx，容易造成页锁泄漏、脏页丢失或重复 writeback。

### 5.7 NAT 与 data mapping 概念混淆

NAT 映射的是 nid -> node 物理块。文件数据块地址存在 node 页的地址数组里。`f2fs_get_node_info()` 在 data OPU 中出现，是为了拿 dnode 的 version 写 summary，不是为了查数据块。学习或修改代码时应始终区分：

- logical file block -> dnode address slot -> data block；
- nid -> NAT -> node block。

### 5.8 writeback 跳过与 fsync 语义

`__f2fs_write_data_pages()`、`f2fs_write_node_pages()` 都会在 WB_SYNC_NONE、POR、checkpoint error、dirty 数量不足时跳过或 redirty。fsync/atomic write 依赖 node/data 的特定刷盘顺序和 mark；修改跳过条件可能造成 fsync 漏刷或恢复后元数据不一致。

### 5.9 多设备、加密、verity 与 BIO 合并

读写合并不仅要求物理块连续，还要满足 multi-device 边界、fscrypt mergeability、verity post-read、压缩 post-read。`f2fs_map_blocks_cached()` 在 DIO 下还会等待 block writeback。任何“优化合并”的改动都必须检查这些条件。

### 5.10 大 folio 与压缩/不可变文件限制

`f2fs_read_data_large_folio()` 当前只支持不可变且非压缩的场景；压缩文件和普通可写文件会回退。若未来扩展 large folio，需要同时处理 per-page pending、映射复用、verity blocks、writeback 与压缩 cluster 边界。

## 6. 学习路线建议

推荐按以下顺序读源码：

1. `f2fs.h`：先看 `f2fs_map_blocks`、`extent_info`、压缩 inode 字段、`blkaddr_in_node()`。
2. `node.c:get_node_path()` 和 `f2fs_get_dnode_of_data()`：建立逻辑块到 dnode 地址槽的模型。
3. `data.c:f2fs_map_blocks()`：理解统一映射接口。
4. `data.c:f2fs_read_single_page()`、`f2fs_write_begin()`、`f2fs_do_write_data_page()`、`f2fs_write_cache_pages()`：串起 buffered I/O 与 writeback。
5. `node.c:f2fs_get_node_info()`、`__write_node_folio()`、`f2fs_sync_node_pages()`：理解 NAT 与 node 写回。
6. `extent_cache.c`：理解 read extent cache 如何加速读、如何在写时失效/更新。
7. `compress.c`：最后看压缩，因为它复用普通 I/O 但把粒度提升到 cluster，且引入 `COMPRESS_ADDR/c_len` 等额外约束。
