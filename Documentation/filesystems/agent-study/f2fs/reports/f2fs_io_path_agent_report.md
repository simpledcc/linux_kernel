# F2FS IO Path Agent 报告

本报告基于当前仓库 `main` 分支源码阅读，重点文件包括 `fs/f2fs/file.c`、`fs/f2fs/data.c`、`fs/f2fs/node.c`、`fs/f2fs/checkpoint.c`、`include/linux/fs.h`、`mm/filemap.c`。结论只覆盖本次已阅读到的路径，不把未确认的行为写成事实。

## 1. 普通文件 read/write/fsync 关键入口与调用链

普通文件 inode 在创建或读取时接入 F2FS 的文件与页缓存操作：

- `fs/f2fs/namei.c:f2fs_create()` 为新普通文件设置 `inode->i_fop = &f2fs_file_operations`、`inode->i_mapping->a_ops = &f2fs_dblock_aops`。
- `fs/f2fs/inode.c:f2fs_iget()` 对已存在普通文件做同样绑定。
- `include/linux/fs.h` 中 `struct file_operations` 提供 VFS 调用的 `read_iter`、`write_iter`、`fsync` 等入口，`struct address_space_operations` 提供 page cache 对文件系统的 `read_folio`、`write_begin`、`write_end`、`writepages`、`dirty_folio` 等回调。

读路径：

1. VFS 根据 `file->f_op->read_iter` 调到 `f2fs_file_read_iter()`。
2. `f2fs_file_read_iter()` 先检查压缩后端、trace，再用 `f2fs_should_use_dio()` 判定是否走 direct I/O。
3. Direct read：`f2fs_dio_read_iter()` 获取 `i_gc_rwsem[READ]`，拒绝 atomic file，增加 `F2FS_DIO_READ` 计数，然后调用 `__iomap_dio_rw(..., &f2fs_iomap_ops, &f2fs_iomap_dio_read_ops, ...)`；完成时 `f2fs_dio_read_end_io()` 更新统计并递减计数。
4. Buffered read：直接调用 `filemap_read()`。`mm/filemap.c:filemap_read()` 从 page cache 拿 folio；缺页时通过 `mapping->a_ops->read_folio` 调到 `f2fs_read_data_folio()`，readahead 则通过 `f2fs_readahead()`。

写路径：

1. VFS 根据 `file->f_op->write_iter` 调到 `f2fs_file_write_iter()`。
2. `f2fs_file_write_iter()` 检查 `cp_error`、压缩后端、获取 inode 锁，检查 pinned file overwrite、调用 `f2fs_write_checks()`，再用 `f2fs_should_use_dio()` 决定 DIO 或 buffered。
3. 写前可能调用 `f2fs_preallocate_blocks()`，为合适场景预分配块并设置 `FI_PREALLOCATED_ALL`。
4. Direct write：`f2fs_dio_write_iter()` 转换 inline inode、持有 GC 读写相关锁，增加 `F2FS_DIO_WRITE` 计数，用 `__iomap_dio_rw(..., &f2fs_iomap_ops, &f2fs_iomap_dio_write_ops, ...)` 提交。若 DIO 短写后 `iov_iter` 仍有剩余，会回退到 `f2fs_buffered_write_iter()`，然后 `f2fs_flush_buffered_write()` 写回并失效该范围 page cache，以维持 `O_DIRECT` 语义。
5. Buffered write：`f2fs_buffered_write_iter()` 调 `generic_perform_write()`。`mm/filemap.c:generic_perform_write()` 循环调用 `mapping->a_ops->write_begin`、拷贝用户数据到 folio、再调用 `mapping->a_ops->write_end`。在 F2FS 中这两个回调分别是 `f2fs_write_begin()` 和 `f2fs_write_end()`。
6. 若返回正数且 `may_need_sync` 为真，`f2fs_file_write_iter()` 调 `generic_write_sync()` 处理同步写语义。

fsync 路径：

1. VFS 根据 `file->f_op->fsync` 调到 `f2fs_sync_file()`。
2. `f2fs_sync_file()` 检查 `cp_error` 后进入 `f2fs_do_sync_file()`。
3. 普通文件先调用 `file_write_and_wait_range()`，这会通过 `filemap_fdatawrite_range()` 触发脏数据页 writeback，并等待范围内 writeback 结束，再检查并推进 `file->f_wb_err`。
4. 若需要 checkpoint，`f2fs_do_sync_file()` 调 `f2fs_sync_fs(sb, 1)`，后者经 `f2fs_issue_checkpoint()` 到 `f2fs_write_checkpoint()`。
5. 若不需要 checkpoint，则走 roll-forward fsync：`f2fs_fsync_node_pages()` 写出该 inode 对应的 dirty node pages，必要时等待 node writeback，并在非 `NOBARRIER` 模式下 `f2fs_issue_flush()`。

## 2. `f2fs_file_operations` 与 `f2fs_dblock_aops` 如何衔接 VFS/page cache

`f2fs_file_operations` 是普通文件面向 VFS 的第一层入口，关键成员为：

- `.read_iter = f2fs_file_read_iter`
- `.write_iter = f2fs_file_write_iter`
- `.fsync = f2fs_sync_file`
- `.splice_read = f2fs_file_splice_read`
- `.splice_write = iter_file_splice_write`
- `.mmap_prepare = f2fs_file_mmap_prepare`

`f2fs_dblock_aops` 是该 inode 的 `address_space` 面向 page cache/writeback 的回调表，关键成员为：

- `.read_folio = f2fs_read_data_folio`
- `.readahead = f2fs_readahead`
- `.writepages = f2fs_write_data_pages`
- `.write_begin = f2fs_write_begin`
- `.write_end = f2fs_write_end`
- `.dirty_folio = f2fs_dirty_data_folio`
- `.invalidate_folio = f2fs_invalidate_folio`
- `.release_folio = f2fs_release_folio`
- `.bmap = f2fs_bmap`

二者的衔接关系是：

- VFS 先进入 `f2fs_file_operations` 的 `read_iter/write_iter/fsync`。
- Buffered read/write 在内部交给 `mm/filemap.c` 的 page cache 通用代码；通用代码再通过 `file->f_mapping->a_ops` 回调 F2FS。
- Writeback 不从 `file_operations` 进入，而是由内核回写框架对 `address_space` 调 `.writepages = f2fs_write_data_pages`。
- Direct I/O 是例外：当前 `f2fs_dblock_aops` 没有设置 `.direct_IO`，F2FS 在 `f2fs_file_read_iter()` / `f2fs_file_write_iter()` 中自行选择 DIO，并通过 iomap 的 `f2fs_iomap_ops` 完成映射和提交。

## 3. Buffered write、direct I/O、writeback 的路径边界

Buffered write 边界：

- 入口在 `f2fs_file_write_iter()` 中选择 `dio == false` 后进入 `f2fs_buffered_write_iter()`。
- 核心拷贝和 page cache 驱动由 `generic_perform_write()` 完成。
- `f2fs_write_begin()` 负责准备/锁定 folio、准备块映射、处理 inline data、压缩覆盖、atomic/COW、旧块读入或新块清零。
- `f2fs_write_end()` 负责根据实际 copied 字节把 folio 置 uptodate/dirty，更新 i_size，并更新 F2FS 请求时间。
- 数据此时通常只是 page cache 脏页；真正下盘发生在后续 writeback 或 fsync/write-and-wait。

Direct I/O 边界：

- 由 `f2fs_should_use_dio()` 判定，必须有 `IOCB_DIRECT`，且不能被 `f2fs_force_buffered_io()` 强制回退。
- 会强制 buffered 的条件包括：fscrypt 不支持 DIO、fsverity active、压缩文件、inline data 的 direct read、多设备块大小不对齐、非 pinned 的 zoned direct write、checkpoint disabled 等。
- 对齐规则上，F2FS 要求文件系统 block size 对齐；若只满足设备 logical block size 而不满足 fs block size，会按传统行为回退 buffered。
- DIO 映射由 `fs/f2fs/data.c:f2fs_iomap_begin()` 调 `f2fs_map_blocks(..., F2FS_GET_BLOCK_DIO)` 提供。写入 hole 时若需要创建映射，`map.m_may_create = true`；若写映射不到块，返回 `-ENOTBLK`，上层可能短写并回退 buffered。

Writeback 边界：

- Page cache 脏页由 `folio_mark_dirty()` 进入 `f2fs_dirty_data_folio()`，后者调用 `filemap_dirty_folio()` 并用 `f2fs_update_dirty_folio()` 增加 inode dirty page 与全局 `F2FS_DIRTY_DATA/F2FS_DIRTY_DENTS` 计数。
- 回写框架调用 `f2fs_write_data_pages()`，进入 `__f2fs_write_data_pages()`，再通过 `f2fs_write_cache_pages()` 按 tag 扫描 dirty folio。
- 每个数据 folio 进入 `f2fs_write_single_data_page()`，进一步调用 `f2fs_do_write_data_page()`。这里根据 IPU/OPU 策略选择 `f2fs_inplace_write_data()` 或 `f2fs_outplace_write_data()`。
- 成功提交后，`f2fs_write_cache_pages()` 合并提交 data/IPU bio；`f2fs_write_single_data_page()` 在出页时递减 dirty page 计数。

## 4. 四个关键函数的作用

`f2fs_read_data_folio()`：

- 是 buffered read 缺页时的 `.read_folio`。
- 先检查压缩后端是否就绪。
- inline data 文件优先尝试 `f2fs_read_inline_data()`。
- 若需要 fs-verity，先触发 `fsverity_readahead()`。
- 最后调用 `f2fs_mpage_readpages(inode, vi, NULL, folio)` 建立并提交实际读 bio；压缩文件会在更深层处理 cluster 读。

`f2fs_write_begin()`：

- 是 buffered write 拷贝前的准备阶段。
- 检查 checkpoint 是否可写；必要时转换 inline inode。
- 对压缩文件处理 overwrite 准备。
- 获取并锁定目标 folio，普通写调用 `prepare_write_begin()`，atomic write 调 `prepare_atomic_write_begin()`。
- 等待 folio 写回结束；如果是新块则清零并标记 uptodate，否则读入旧块，保证部分页写不会破坏未覆盖数据。

`f2fs_write_end()`：

- 是 buffered write 拷贝后的提交阶段。
- 若 folio 还不是 uptodate，根据 copied 情况决定重试或标记 uptodate。
- 压缩覆盖写走 `f2fs_compress_write_end()`。
- 普通情况将 folio 标 dirty；atomic file 额外设置 atomic 标记。
- 若写过 EOF，更新 inode i_size；最后释放 folio 并更新请求时间。

`f2fs_write_data_pages()`：

- 是 `.writepages` 回调，是 writeback 子系统写出 dirty page cache 的入口。
- 包装 `__f2fs_write_data_pages()`，并根据当前任务是否为 checkpoint 任务选择 `FS_CP_DATA_IO` 或 `FS_DATA_IO` 统计类型。
- 内部会跳过无 dirty page、POR、defrag skip 等场景；对同步/异步 writeback 做优先级和序列化控制。
- 最终调用 `f2fs_write_cache_pages()` 扫描 dirty folio，并通过 `f2fs_write_single_data_page()` / `f2fs_do_write_data_page()` 转换成实际 data bio。

## 5. fsync、checkpoint、dirty page 的关系

Dirty data page 与 inode dirty metadata 是两套相关但不同的状态：

- `f2fs_write_end()` 标脏数据 folio；`f2fs_dirty_data_folio()` 增加 per-inode `dirty_pages` 与全局 dirty data/dentry 计数。
- inode 元数据脏由 `f2fs_mark_inode_dirty_sync()`、`f2fs_dirty_inode()` 等路径维护，表现为 `FI_DIRTY_INODE`、`DIRTY_META`、`F2FS_DIRTY_IMETA` 等。
- `file_write_and_wait_range()` 是 fsync 的第一道数据保证：它触发并等待目标范围内 data page writeback，并上报此前 writeback 错误。

`f2fs_do_sync_file()` 在数据页写完后决定是 checkpoint 还是 roll-forward：

- `need_do_checkpoint()` 返回非零时走完整 checkpoint，原因包括非普通文件、压缩文件、硬链接、超级块需要 CP、父目录/节点状态不适合 roll-forward、roll-forward 空间不足、strict fsync 下目录恢复要求等。
- 需要 checkpoint 时调用 `f2fs_sync_fs(sb, 1)`，最终进入 `f2fs_write_checkpoint()`。
- 不需要 checkpoint 时，写当前 inode 的 fsync node pages，清除 `APPEND_INO/UPDATE_INO` 等恢复相关 ino entry，并按配置发 flush。

Checkpoint 的核心目标是建立全文件系统一致点：

- `f2fs_write_checkpoint()` 持有 `cp_global_sem`，调用 `block_operations()` 阻塞/冻结会影响 CP 的操作。
- `block_operations()` 会 flush inline data、写 dirty dentry pages、同步 dirty inode metadata、同步 dirty node pages，并准备 checkpoint block。
- `do_checkpoint()` 写 NAT/SIT、summary、orphan、checkpoint pack，等待 dirty meta 和 CP data writeback，flush 设备缓存，并提交最后一个 checkpoint pack。
- 普通 regular file 的 dirty data page 不等同于 checkpoint 本身一定会全部写出；fsync 通过 `file_write_and_wait_range()` 保证该文件目标范围的数据先落盘，再用 checkpoint 或 fsync node 链保证恢复路径。

## 6. 修改 I/O 路径的主要风险

- Page cache 与块映射一致性：`write_begin()` 的旧块读入、新块清零、truncate/hole punch 的 `invalidate_lock` 规则若破坏，会出现 stale data、越 EOF 脏数据或数据泄露。
- fsync 持久化顺序：F2FS 依赖 data writeback、node page、fsync mark、flush/checkpoint 的顺序。改动 `FI_APPEND_WRITE`、`FI_UPDATE_WRITE`、ino entry、barrier/flush 可能导致断电恢复丢数据或恢复到错误版本。
- DIO 与 buffered 混用：DIO 短写回退 buffered 后必须写回并 invalidate；否则 `O_DIRECT` 语义、page cache coherency 和后续 read 结果会错。
- IPU/OPU 策略：`f2fs_do_write_data_page()` 选择原地更新或异地更新会影响 NAT/SIT、segment summary、roll-forward 恢复与 GC。错误选择可能破坏 LFS/SSR、zoned device 或 pinned file 约束。
- Dirty page 计数：`inode_inc_dirty_pages()` / `inode_dec_dirty_pages()` 和 `F2FS_DIRTY_*` 计数若失衡，会导致 checkpoint/writeback 等待条件错误、后台回写不触发或无限等待。
- 压缩、加密、verity、atomic write 是高风险交叉路径。它们在 read/write_begin/writeback/DIO 判定中都有特殊分支，不能只按普通未压缩文件推断。
- 锁顺序风险：`i_rwsem`、`i_gc_rwsem`、`cp_global_sem`、`node_write`、`node_change`、folio lock、`f2fs_lock_op` 等顺序被多处注释强调，随意调整可能造成 fsync/checkpoint/writeback/GC 死锁。

## 7. 不确定点

- 本次没有完整追踪 `f2fs_map_blocks()`、`f2fs_outplace_write_data()`、`f2fs_inplace_write_data()` 到 segment allocator/NAT/SIT 的所有细节，因此不在报告中断言具体物理分配策略的全部条件。
- 压缩文件的 cluster 读写路径只阅读了与 `read_folio/write_begin/writeback` 相邻的分支，没有完整展开 `f2fs_read_multi_pages()`、`f2fs_write_multi_pages()` 的所有错误处理。
- Atomic write/COW inode 路径只确认了 `write_begin/write_end/writeback` 中的入口和标记，不确认用户态 ioctl 生命周期的全部状态机。
- Checkpoint 与普通文件 dirty data 的关系在本报告中按已读源码描述：fsync 先写等待目标数据范围，checkpoint 核心流程主要同步 dentry/node/meta/summary/CP pack；后台 `DATA_FLUSH` 会额外从 `segment.c` 同步 FILE_INODE dirty data。没有继续展开所有后台线程触发条件。
