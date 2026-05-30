# Page Cache Agent 报告：page cache、readahead、writeback 与文件系统

## 阅读范围

本报告基于以下文件的代码阅读：`mm/filemap.c`、`mm/readahead.c`、`mm/page-writeback.c`、`mm/truncate.c`、`fs/buffer.c`、`include/linux/pagemap.h`、`include/linux/writeback.h`，并补充核对了 ext4 的 `fs/ext4/file.c`、`fs/ext4/inode.c`、`fs/ext4/readpage.c`。

## 1. page cache 在 read/write 路径中的位置

### buffered read

典型 buffered read 路径是：

```text
file_operations->read_iter
  -> generic_file_read_iter()
     -> filemap_read()
        -> filemap_get_pages()
           -> mapping->i_pages XArray 查找 page-cache folio
           -> miss: page_cache_sync_ra()/filemap_create_folio()
           -> not uptodate: mapping->a_ops->read_folio()
        -> copy_folio_to_iter()
```

page cache 位于 `struct address_space` 的 `i_pages` XArray 中，索引是文件偏移转换成的 page/folio index。`filemap_read()` 不直接理解磁盘布局；它负责查找、锁定、等待、触发 readahead/read_folio，并把 uptodate folio 的内容复制到用户 iov_iter。真正把文件逻辑块映射到底层块设备并提交 I/O 的工作由文件系统的 `address_space_operations` 完成。

以 ext4 为例，`ext4_file_read_iter()` 在普通 buffered read 时调用 `generic_file_read_iter()`；后续 miss 或 readahead 会进入 `ext4_read_folio()` / `ext4_readahead()`，再由 `ext4_mpage_readpages()` 组装 BIO。异常情况会退回 `block_read_full_folio()` 这种 buffer_head 路径。

### buffered write

典型 buffered write 路径是：

```text
file_operations->write_iter
  -> 文件系统 write checks / inode lock
  -> generic_perform_write()
     -> balance_dirty_pages_ratelimited(mapping)
     -> mapping->a_ops->write_begin()
        -> 找到或创建 page-cache folio，必要时分配/映射块
     -> copy_folio_from_iter_atomic()
     -> mapping->a_ops->write_end()
        -> 提交本次修改，标记 buffer/folio dirty，更新 i_size
```

buffered write 先写入 page cache，不一定立即落盘。dirty 状态通过 folio、buffer_head、page-cache tag 和 inode dirty 状态串起来：`mark_buffer_dirty()` 会设置 buffer dirty、folio dirty、`PAGECACHE_TAG_DIRTY`，并把 inode 标成 `I_DIRTY_PAGES`；不使用 buffer_head 的文件系统通常走 `filemap_dirty_folio()`。

ext4 的 buffered write 入口是 `ext4_buffered_write_iter()`，它做 ext4 特有检查后调用 `generic_perform_write()`。是否使用 delayed allocation、data=journal 或普通模式，由 `ext4_set_aops()` 给 inode 选择不同的 aops：

```text
ext4_da_aops          -> ext4_da_write_begin()/ext4_da_write_end()
ext4_aops             -> ext4_write_begin()/ext4_write_end()
ext4_journalled_aops  -> ext4_write_begin()/ext4_journalled_write_end()
ext4_dax_aops         -> DAX 专用，不走普通 page-cache read/write folio
```

## 2. buffered read、buffered write、direct I/O 的边界

buffered read/write 的共同点是以 `address_space->i_pages` 为中心，读 miss 会填充 page cache，写会先修改 page cache 并产生 dirty folio。读写的数据一致性依赖 folio uptodate、dirty/writeback tag、`invalidate_lock`、i_size 检查以及文件系统 aops。

direct I/O 的目标是绕过 page cache 直接对存储提交 I/O，但它不能忽略 page cache 的一致性边界：

- generic direct read 路径在 `generic_file_read_iter()` 中会先调用 `kiocb_write_and_wait()`，等待相关范围的 dirty/writeback page cache，再调用 `mapping->a_ops->direct_IO()`。如果 direct read 短读且仍有剩余，通用代码可能回落到 buffered read；DAX 文件不会这样回落。
- generic direct write 路径 `generic_file_direct_write()` 会在写前调用 `kiocb_invalidate_pages()`，必要时等待并失效对应 page-cache 范围；写后还会尝试 `kiocb_invalidate_post_direct_write()`，避免 clean cached pages 或 mmap/GUP 引入的页留下旧数据。
- 当前 ext4 direct I/O 主要在 `fs/ext4/file.c` 的 `ext4_dio_read_iter()` / `ext4_dio_write_iter()` 中通过 `iomap_dio_rw()` 实现，而不是依赖 ext4 aops 填 `direct_IO`。如果 ext4 判断该 inode/请求不支持 DIO，会回落到 buffered I/O；direct write 部分完成后若还有剩余 buffered fallback，ext4 会对 fallback 范围做 `filemap_write_and_wait_range()` 和 `invalidate_mapping_pages()` 来尽量保留 direct I/O 语义。

因此边界可以概括为：buffered I/O 以 page cache 为数据面；direct I/O 以块映射/iomap/BIO 为数据面，但进出 direct I/O 前后必须处理 page cache 中同一范围的 dirty、writeback 和 stale cache。

## 3. readahead、dirty page、writeback 的关键函数和调用关系

### readahead

关键结构是每个 `struct file` 内的 `file_ra_state f_ra`，由 `file_ra_state_init()` 根据 backing device 的 `ra_pages` 初始化。

主要调用关系：

```text
filemap_read()
  -> filemap_get_pages()
     -> filemap_get_read_batch()
     -> batch empty:
        DEFINE_READAHEAD(...)
        page_cache_sync_ra()
          -> 计算窗口 start/size/async_size
          -> page_cache_ra_order() 或 do_page_cache_ra()
             -> filemap_add_folio()
             -> read_pages()
                -> mapping->a_ops->readahead(rac)
                -> fallback: mapping->a_ops->read_folio()
     -> folio_test_readahead(last folio):
        page_cache_async_ra()
```

`mm/readahead.c` 的注释说明了两个触发点：cache miss 触发同步 readahead；访问到带 readahead 标记的 folio 触发异步 readahead。`read_pages()` 是从 MM readahead 框架进入文件系统的关键边界。

ext4 的对应路径：

```text
read_pages()
  -> ext4_readahead()
     -> ext4_mpage_readpages()
        -> 映射 ext4 blocks，合并 BIO，设置 REQ_RAHEAD
        -> inline data: 不做 readahead
        -> 特殊/复杂情况: block_read_full_folio()
```

### dirty page

buffered write 和 mmap shared write 都会把 folio 变 dirty，但入口略有不同：

```text
generic_perform_write()
  -> mapping->a_ops->write_begin()
  -> copy data into folio
  -> mapping->a_ops->write_end()
     -> block_write_end()/block_commit_write()
        -> mark_buffer_dirty()
           -> __folio_mark_dirty()
           -> __mark_inode_dirty(..., I_DIRTY_PAGES)
```

对 buffer_head 文件系统，`block_dirty_folio()` 是常见的 `->dirty_folio` 实现；它先同步 buffer dirty 状态，再设置 folio dirty，最后标记 inode。对非 buffer_head 文件系统，`filemap_dirty_folio()` 是更直接的 folio dirty 实现。

`balance_dirty_pages_ratelimited(mapping)` 在 `generic_perform_write()` 每轮写入前调用，用于周期性检查 dirty 限额并触发/等待 writeback，防止写入者无限堆积 dirty page。

### writeback

关键调用关系：

```text
filemap_fdatawrite_range()/filemap_flush_range()
  -> filemap_writeback()
     -> do_writepages(mapping, wbc)
        -> mapping->a_ops->writepages(mapping, wbc)

文件系统 writepages 实现
  -> writeback_iter(mapping, wbc, folio, &error)
     -> 按 PAGECACHE_TAG_DIRTY/PAGECACHE_TAG_TOWRITE 扫描
     -> 锁 folio
     -> folio_prepare_writeback()
        -> 等待已有 writeback
        -> folio_clear_dirty_for_io()
     -> 文件系统提交 I/O
```

`writeback_control` 描述本轮 writeback：`nr_to_write`、`range_start`、`range_end`、`sync_mode`、`tagged_writepages`、`range_cyclic` 等。`WB_SYNC_ALL` 或 tagged writepages 会先用 `tag_pages_for_writeback()` 把 dirty tag 转为 `PAGECACHE_TAG_TOWRITE`，减少边写边脏导致的 livelock。

ext4 的 writeback 入口是 `ext4_writepages()`，核心是 `ext4_do_writepages()`。它需要处理 delayed allocation、data=journal、dioread_nolock、unwritten extent 转换、journal credits、BIO submit 和 `mapping->writeback_index` 更新。普通 mpage 类文件系统可用 `mpage_writepages()`，其内部也是围绕 `writeback_iter()` 遍历 dirty folio。

## 4. address_space、address_space_operations 与文件系统的衔接

`struct address_space` 是 page cache 与具体文件对象之间的桥：

- `host` 指向拥有者 inode 或 block_device。
- `i_pages` 保存 page-cache folio。
- `invalidate_lock` 保护 page cache 内容和文件 offset 到磁盘块映射之间的一致性，truncate/hole punch/direct I/O/read fault 等路径都需要考虑它。
- `i_mmap` / `i_mmap_writable` 追踪文件映射到用户态的 VMA。
- `nrpages`、`writeback_index`、`flags`、`wb_err` 记录 cache/writeback 状态。
- `a_ops` 是文件系统参与 page-cache 生命周期的回调表。

`struct address_space_operations` 中与本主题最相关的回调：

- `read_folio()`：同步读取单个 folio。
- `readahead()`：批量预读 folio。
- `write_begin()` / `write_end()`：buffered write 的文件系统准备和提交阶段。
- `dirty_folio()`：mmap/shared dirty 或其他 set_page_dirty 场景下同步文件系统元数据。
- `writepages()`：writeback 入口。
- `invalidate_folio()` / `release_folio()`：truncate、invalidate、回收时释放文件系统私有状态。
- `is_partially_uptodate()`：支持部分 uptodate 的 buffer_head folio，避免不必要整页读取。
- `migrate_folio()`、`bmap()`、`swap_activate()` 等是迁移、FIBMAP/swap 等辅助接口。

通用 MM 层只管理 folio 生命周期、tag、锁、LRU、dirty accounting 和 writeback 调度；文件系统通过 aops 决定如何把 folio 中的字节映射到块、如何与日志/延迟分配/加密/verity/DAX 等特性协调。

## 5. 与 ext4 开发相关的风险点

1. direct I/O 与 buffered I/O/mmap 混用的一致性风险。direct write 前后 invalidate 失败时可能留下 stale page cache；通用层甚至有 “possible data corruption due to collision with buffered I/O” 的告警路径。ext4 DIO fallback 到 buffered I/O 后还要 flush/invalidate fallback 范围，这部分不能省。

2. delayed allocation 把 ENOSPC 和块分配推迟到 writeback。`ext4_da_write_begin()` / `ext4_da_write_end()` 只是准备 delalloc 状态，真正分配可能在 `ext4_writepages()`。修改 i_size、i_disksize、orphan list 或错误恢复时，必须考虑崩溃一致性和 late ENOSPC。

3. data=journal 模式的 dirty 语义不同。`ext4_journalled_dirty_folio()` 对 DMA pinned folio 使用 pending dirty/check 标记；不能随意把 folio dirty、buffer dirty、jbd dirty 当作同一件事。buffer 状态在 journalled data 下是权威信息之一。

4. lock ordering 容易踩雷。`mm/filemap.c` 明确列出 `i_rwsem`、`invalidate_lock`、`mmap_lock`、folio lock、`i_pages` lock 等顺序；ext4 又叠加 journal handle。`ext4_write_begin()` 先 grab folio 再启动 transaction，是为了避免内存压力下持有 journal handle 等待 folio。`ext4_do_writepages()` 也避免在未提交 I/O、未释放 folio/io_end 时停止同步 handle。

5. truncate/hole punch 与 page cache 的同步必须完整。`truncate_inode_pages_range()` 会处理 partial folio、等待 writeback、调用 `invalidate_folio()` 并从 page cache 删除 folio。文件系统释放块前后如果没有正确持有 `invalidate_lock` 或清理 folio 私有状态，可能出现旧块数据、SIGBUS 语义或 buffer_head 悬挂问题。

6. buffer_head 与 folio dirty 状态必须一致。`block_dirty_folio()` 和 `try_to_free_buffers()` 通过 `mapping->i_private_lock` 协调；`mark_buffer_dirty()` 同时推进 buffer、folio、page-cache tag 和 inode dirty 状态。修改 ext4 buffer_head 路径时，最危险的是留下 “dirty buffer / clean folio” 或 “clean buffer / dirty folio” 的不一致。

7. large folio、partial uptodate 和 blocksize < pagesize 场景不能按单页假设写代码。`mapping_max_folio_size()`、`write_begin_get_folio()`、`block_is_partially_uptodate()`、`truncate_inode_partial_folio()` 都说明现在 page cache 路径已经是 folio 语义。

8. ext4 read path 还叠加 inline data、fscrypt、fsverity、DAX 等分支。`ext4_readahead()` 对 inline data 直接跳过；`ext4_mpage_readpages()` 在复杂布局下会 fallback 到 buffer_head read。优化 readahead 时不能只看普通 extent 连续映射场景。

## 6. 不确定点

- 本次没有完整展开 `fs/fs-writeback.c` 中 flusher thread、dirty inode list、superblock writeback 与 `do_writepages()` 之间的全部调度链；这里只确认了 page-cache 层和 aops `writepages()` 边界。
- 本次没有深入 `fs/iomap/` 的 direct I/O 内部实现，因此 ext4 DIO 的 BIO 提交、完成回调、并发细节只记录到 `iomap_dio_rw()` 边界。
- ext4 的 inline data、fscrypt、fsverity、bigalloc、DAX、journal fast commit 等特性没有逐项完整验证；报告中的风险点只基于本次阅读到的 page-cache 相关路径。
- 没有对具体内核配置组合做运行验证，例如 DAX、不同 block size、large folio 策略、cgroup writeback、memcg foreign dirty accounting 等；这些会影响部分分支是否可达。
