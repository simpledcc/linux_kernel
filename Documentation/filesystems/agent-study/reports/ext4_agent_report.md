# Ext4 Agent 代码阅读报告

本报告基于当前分支 `agent-fs-study-cn` 中的 ext4 源码阅读，重点文件包括 `fs/ext4/file.c`、`fs/ext4/inode.c`、`fs/ext4/super.c`、`fs/ext4/namei.c`、`fs/ext4/ext4.h`、`fs/ext4/ext4_jbd2.h`。为说明 `fsync` 调用链，也额外查阅了 `fs/ext4/fsync.c`。

## 1. super_operations、file_operations、inode_operations 的注册

### 文件系统类型与 super_operations

ext4 作为 VFS 文件系统类型注册在 `fs/ext4/super.c`：

- `static struct file_system_type ext4_fs_type` 定义 `.name = "ext4"`、`.init_fs_context = ext4_init_fs_context`、`.kill_sb = ext4_kill_sb` 等字段。
- `ext4_init_fs()` 初始化 ext4 的缓存、sysfs、mballoc、fast commit dentry cache 等内部资源后，调用 `register_filesystem(&ext4_fs_type)`。
- 挂载时 VFS 通过 `ext4_init_fs_context()` 设置 `fs_context_operations`，其中 `.get_tree = ext4_get_tree`。
- `ext4_get_tree()` 调用 `get_tree_bdev(fc, ext4_fill_super)`，进入块设备挂载路径。
- `ext4_fill_super()` 进一步调用 `__ext4_fill_super()`。
- `__ext4_fill_super()` 在已经建立足够的 superblock 上下文后执行：
  - `sb->s_op = &ext4_sops`
  - `sb->s_export_op = &ext4_export_ops`
  - `sb->s_xattr = ext4_xattr_handlers`
  - 在配置启用时设置 `s_cop`、`s_vop`、quota 相关操作。

`ext4_sops` 是 ext4 对 VFS superblock 生命周期和同步接口的实现，主要字段包括：

- `.alloc_inode = ext4_alloc_inode`
- `.free_inode = ext4_free_in_core_inode`
- `.destroy_inode = ext4_destroy_inode`
- `.write_inode = ext4_write_inode`
- `.dirty_inode = ext4_dirty_inode`
- `.drop_inode = ext4_drop_inode`
- `.evict_inode = ext4_evict_inode`
- `.put_super = ext4_put_super`
- `.sync_fs = ext4_sync_fs`
- `.freeze_fs = ext4_freeze`
- `.unfreeze_fs = ext4_unfreeze`
- `.statfs = ext4_statfs`
- `.show_options = ext4_show_options`
- `.shutdown = ext4_shutdown`
- quota 配置下还会设置 `.quota_read`、`.quota_write`、`.get_dquots`。

### 普通文件 file_operations 与 inode_operations

普通文件的操作表定义在 `fs/ext4/file.c`：

- `const struct file_operations ext4_file_operations`
  - `.llseek = ext4_llseek`
  - `.read_iter = ext4_file_read_iter`
  - `.write_iter = ext4_file_write_iter`
  - `.iopoll = iocb_bio_iopoll`
  - `.unlocked_ioctl = ext4_ioctl`
  - `.mmap_prepare = ext4_file_mmap_prepare`
  - `.open = ext4_file_open`
  - `.release = ext4_release_file`
  - `.fsync = ext4_sync_file`
  - `.splice_read = ext4_file_splice_read`
  - `.splice_write = iter_file_splice_write`
  - `.fallocate = ext4_fallocate`
  - `.setlease = generic_setlease`

- `const struct inode_operations ext4_file_inode_operations`
  - `.setattr = ext4_setattr`
  - `.getattr = ext4_file_getattr`
  - `.listxattr = ext4_listxattr`
  - `.get_inode_acl = ext4_get_acl`
  - `.set_acl = ext4_set_acl`
  - `.fiemap = ext4_fiemap`
  - `.fileattr_get = ext4_fileattr_get`
  - `.fileattr_set = ext4_fileattr_set`

普通文件 inode 获得这些操作表有两条关键路径：

1. 新建文件：`fs/ext4/namei.c::ext4_create()` 调用 `ext4_new_inode_start_handle()` 创建 inode 后，设置：
   - `inode->i_op = &ext4_file_inode_operations`
   - `inode->i_fop = &ext4_file_operations`
   - `ext4_set_aops(inode)`
   然后通过 `ext4_add_nondir()` 把 inode 连接到 dentry。

2. 从磁盘读 inode：`fs/ext4/inode.c::__ext4_iget()` 检查 inode 类型。对 `S_ISREG(inode->i_mode)` 的普通文件，同样设置：
   - `inode->i_op = &ext4_file_inode_operations`
   - `inode->i_fop = &ext4_file_operations`
   - `ext4_set_aops(inode)`

### 目录、特殊文件、符号链接的 inode_operations

`fs/ext4/namei.c` 中定义目录和特殊文件的 inode 操作：

- `const struct inode_operations ext4_dir_inode_operations`
  - `.create = ext4_create`
  - `.lookup = ext4_lookup`
  - `.link = ext4_link`
  - `.unlink = ext4_unlink`
  - `.symlink = ext4_symlink`
  - `.mkdir = ext4_mkdir`
  - `.rmdir = ext4_rmdir`
  - `.mknod = ext4_mknod`
  - `.tmpfile = ext4_tmpfile`
  - `.rename = ext4_rename2`
  - 以及 `setattr/getattr/xattr/acl/fiemap/fileattr` 等。

- `const struct inode_operations ext4_special_inode_operations`
  - 主要提供 `setattr/getattr/listxattr/acl` 等元数据操作。

目录的 `file_operations` 在 `fs/ext4/dir.c` 中定义为 `ext4_dir_operations`，包括：

- `.open = ext4_dir_open`
- `.llseek = ext4_dir_llseek`
- `.read = generic_read_dir`
- `.iterate_shared = ext4_readdir`
- `.fsync = ext4_sync_file`
- `.release = ext4_release_dir`

新建目录时，`ext4_mkdir()` 设置：

- `inode->i_op = &ext4_dir_inode_operations`
- `inode->i_fop = &ext4_dir_operations`

特殊文件由 `ext4_mknod()` 设置 `inode->i_op = &ext4_special_inode_operations`。符号链接操作表包括 `ext4_symlink_inode_operations`、`ext4_fast_symlink_inode_operations`、`ext4_encrypted_symlink_inode_operations`，具体选择发生在创建符号链接和 `__ext4_iget()` 的 inode 类型分支中。

## 2. open/read/write/fsync 的关键入口和调用链

### open

普通文件 open 的 VFS 入口来自 `ext4_file_operations.open = ext4_file_open`。

关键路径：

```text
VFS open
  -> file->f_op->open()
  -> ext4_file_open(inode, filp)
      -> 写打开: ext4_emergency_state()
         读打开: ext4_forced_shutdown() 检查
      -> ext4_sample_last_mounted()
      -> fscrypt_file_open()
      -> fsverity_file_open()
      -> 写打开且有 journal: ext4_inode_attach_jinode()
      -> 根据 inode 能力设置 FMODE_CAN_ATOMIC_WRITE
      -> 设置 FMODE_NOWAIT | FMODE_CAN_ODIRECT
      -> dquot_file_open()
```

`ext4_inode_attach_jinode()` 会为带 journal 的写打开 inode 分配并初始化 `struct jbd2_inode`，挂到 `EXT4_I(inode)->jinode`。后续 ordered/journalled 数据模式和 fsync 需要依赖它追踪 inode 的数据范围与事务关系。

### read

普通文件 read 的 VFS 入口是 `ext4_file_operations.read_iter = ext4_file_read_iter`。

关键路径：

```text
VFS read/readv/io_uring read
  -> file->f_op->read_iter()
  -> ext4_file_read_iter(iocb, to)
      -> ext4_forced_shutdown() 检查
      -> 空 iov: 返回 0
      -> DAX inode: ext4_dax_read_iter()
          -> dax_iomap_rw(..., &ext4_iomap_ops)
      -> IOCB_DIRECT: ext4_dio_read_iter()
          -> inode_lock_shared()
          -> ext4_should_use_dio()
          -> iomap_dio_rw(..., &ext4_iomap_ops)
          -> file_accessed()
      -> buffered read:
          -> generic_file_read_iter()
              -> page cache / readahead
              -> mapping->a_ops->read_folio / readahead
              -> ext4_read_folio() / ext4_readahead()
              -> ext4_mpage_readpages()
              -> ext4_map_blocks(NULL, inode, &map, 0)
              -> bio 提交，必要时 fscrypt/fsverity 后处理
```

buffered read 的 ext4 特定衔接点主要是 `address_space_operations` 中的 `.read_folio = ext4_read_folio` 和 `.readahead = ext4_readahead`。它们在 `fs/ext4/readpage.c` 中通过 `ext4_mpage_readpages()` 批量查找逻辑块到物理块映射，并组装 BIO。

### write

普通文件 write 的 VFS 入口是 `ext4_file_operations.write_iter = ext4_file_write_iter`。

总入口逻辑：

```text
VFS write/writev/io_uring write
  -> file->f_op->write_iter()
  -> ext4_file_write_iter(iocb, from)
      -> ext4_emergency_state()
      -> DAX inode: ext4_dax_write_iter()
      -> IOCB_ATOMIC: 检查 s_awu_min/s_awu_max 和 generic_atomic_write_valid()
      -> IOCB_DIRECT: ext4_dio_write_iter()
      -> 否则: ext4_buffered_write_iter()
```

buffered write 关键路径：

```text
ext4_buffered_write_iter()
  -> 拒绝 NOWAIT buffered write: -EOPNOTSUPP
  -> inode_lock()
  -> ext4_write_checks()
  -> generic_perform_write()
      -> mapping->a_ops->write_begin()
      -> 用户数据 copy 到 page cache folio
      -> mapping->a_ops->write_end()
  -> inode_unlock()
  -> generic_write_sync()   // O_SYNC/IOCB_DSYNC 等同步语义
```

`ext4_set_aops()` 决定 buffered write 具体走哪组 `address_space_operations`：

- `data=journal`：`ext4_journalled_aops`
  - `.write_begin = ext4_write_begin`
  - `.write_end = ext4_journalled_write_end`
- 非 journal-data 且 DAX：`ext4_dax_aops`
- 挂载启用 delalloc：`ext4_da_aops`
  - `.write_begin = ext4_da_write_begin`
  - `.write_end = ext4_da_write_end`
- 非 delalloc：`ext4_aops`
  - `.write_begin = ext4_write_begin`
  - `.write_end = ext4_write_end`

非 delalloc buffered write 中，`ext4_write_begin()` 会在准备 folio 和 buffer head 后启动 journal handle：

```text
ext4_write_begin()
  -> write_begin_get_folio()
  -> create_empty_buffers()
  -> ext4_journal_start(..., EXT4_HT_WRITE_PAGE, needed_blocks)
  -> ext4_block_write_begin(..., ext4_get_block 或 ext4_get_block_unwritten)
      -> ext4_get_block()
          -> _ext4_get_block()
          -> ext4_map_blocks(ext4_journal_current_handle(), ...)
```

`ext4_write_end()` 则通过 `block_write_end()` 完成 page cache/buffer 状态更新，必要时更新 `i_size`、调用 `ext4_mark_inode_dirty()`，最后 `ext4_journal_stop(handle)`。

delalloc buffered write 中，`ext4_da_write_begin()` 通常不立即分配真实磁盘块，而是通过 `ext4_da_get_block_prep()` 建立延迟分配状态并保留空间；`ext4_da_write_end()` 更新 page cache、`i_size`，需要时启动小事务更新 `i_disksize`。真正分配物理块通常推迟到 writeback：

```text
writeback
  -> mapping->a_ops->writepages = ext4_writepages()
  -> ext4_do_writepages()
  -> mpage_prepare_extent_to_map()
  -> mpage_map_and_submit_extent()
  -> mpage_map_one_extent()
  -> ext4_map_blocks(handle, ..., EXT4_GET_BLOCKS_CREATE | EXT4_GET_BLOCKS_IO_SUBMIT | ...)
  -> mpage_map_and_submit_buffers()
  -> ext4_io_submit()
```

direct I/O write 关键路径：

```text
ext4_dio_write_iter()
  -> 根据是否扩展文件选择 inode_lock_shared() 或 inode_lock()
  -> ext4_should_use_dio(); 不支持则回退 ext4_buffered_write_iter()
  -> ext4_dio_write_checks()
  -> 扩展写时:
       ext4_journal_start(... EXT4_HT_INODE ...)
       ext4_orphan_add()
       ext4_journal_stop()
  -> iomap_dio_rw(..., &ext4_iomap_ops, &ext4_dio_write_ops, ...)
       -> ext4_iomap_begin()
       -> 已映射覆盖写可直接 ext4_map_blocks(NULL, ..., 0)
       -> 需要分配/转换时 ext4_iomap_alloc()
            -> ext4_journal_start(... EXT4_HT_MAP_BLOCKS ...)
            -> ext4_map_blocks(handle, ..., CREATE/UNWRIT/CONVERT flags)
            -> ext4_journal_stop()
  -> 必要时 ext4_handle_inode_extension()
  -> 必要时 ext4_inode_extension_cleanup()
  -> 部分 DIO 回退 buffered write 时，调用 filemap_write_and_wait_range() 和 invalidate_mapping_pages()
```

DAX write 与 DIO 类似使用 iomap，但调用 `dax_iomap_rw()`，扩展写同样需要 journal/orphan 保护和 inode size 更新。

### fsync

普通文件和目录都把 `.fsync` 指向 `ext4_sync_file`，实现位于 `fs/ext4/fsync.c`。

关键路径：

```text
VFS fsync/fdatasync/msync
  -> file->f_op->fsync()
  -> ext4_sync_file(file, start, end, datasync)
      -> ext4_emergency_state()
      -> ASSERT 当前任务没有打开的 ext4 journal handle
      -> 只读 superblock: 直接退出到错误检查
      -> 无 journal:
           ext4_fsync_nojournal()
             -> mmb_fsync_noflush()
             -> ext4_write_inode()
             -> ext4_sync_parent()
             -> 必要时请求 flush barrier
      -> 有 journal:
           file_write_and_wait_range(file, start, end)
           ext4_fsync_journal(inode, datasync, &needs_barrier)
             -> 目录/特殊文件: ext4_force_commit()
             -> 普通文件: commit_tid = datasync ? i_datasync_tid : i_sync_tid
             -> ext4_fc_commit(journal, commit_tid)
      -> needs_barrier 时 blkdev_issue_flush()
      -> file_check_and_advance_wb_err()
```

这里的核心点是：普通有 journal 的 fsync 先确保数据页写回，再等待对应事务提交。事务号由 `EXT4_I(inode)->i_sync_tid` 和 `i_datasync_tid` 记录，更新点通常在 inode/metadata 被 journal handle 修改时通过 `ext4_update_inode_fsync_trans()` 或相关路径维护。

## 3. inode、extent、journal/jbd2 在读写路径中的作用

### inode

ext4 的内存 inode 是 `struct ext4_inode_info`，其中嵌入 VFS inode：

- `struct inode vfs_inode`
- `__le32 i_data[15]`：传统块映射或 extent root 的存储区域。
- `loff_t i_disksize`：磁盘上记录的文件大小，可能与 VFS `i_size` 暂时不同。truncate、延迟分配、writeback、崩溃恢复都依赖它区分内存大小和已持久化大小。
- `struct rw_semaphore i_data_sem`：保护 extent/indirect block 映射树，防止 truncate 与 block mapping/分配并发破坏一致性。
- `struct jbd2_inode *jinode`：jbd2 用来跟踪 inode 相关数据范围和事务。
- `struct ext4_es_tree i_es_tree`：extent status cache，缓存 written/unwritten/delayed/hole 状态，`ext4_map_blocks()` 会优先查它。
- `i_reserved_data_blocks`、prealloc 结构：delalloc 和预分配路径使用。
- fast commit 相关字段：`i_fc_*` 用于记录需要 fast commit 的 inode/range。

读路径中，inode 提供文件大小、块大小、加密/verity/DAX/inline data 状态，以及逻辑块到物理块映射的根。写路径中，inode 还承载锁、延迟分配状态、磁盘大小、orphan 保护和 journal 跟踪信息。

### extent 与块映射

extent 格式定义在 `fs/ext4/ext4_extents.h`：

- `struct ext4_extent_header`：extent tree header，记录 magic、entries、max、depth、generation。
- `struct ext4_extent`：叶子节点，记录第一个逻辑块、长度、物理起始块高低位。
- `struct ext4_extent_idx`：非叶子索引节点，指向下一层 extent block。

`EXT4_I(inode)->i_data` 中可以直接容纳 extent root。`ext_inode_hdr(inode)` 把 `i_data` 解释为 extent header。

读写路径统一通过 `struct ext4_map_blocks` 表达一次逻辑块映射请求：

- `m_lblk`：起始逻辑块。
- `m_len`：请求/返回的块数。
- `m_pblk`：物理块。
- `m_flags`：`EXT4_MAP_MAPPED`、`EXT4_MAP_NEW`、`EXT4_MAP_UNWRITTEN`、`EXT4_MAP_DELAYED`、`EXT4_MAP_BOUNDARY` 等。

核心函数是 `ext4_map_blocks(handle, inode, &map, flags)`：

1. 先查 extent status tree。
2. 如果缓存没有满足请求，则持有 `i_data_sem` 读锁调用 `ext4_map_query_blocks()`。
3. extent inode 走 `ext4_ext_map_blocks()`，非 extent inode 走 `ext4_ind_map_blocks()`。
4. 只查询时，没有映射会返回 hole 信息。
5. 带 `EXT4_GET_BLOCKS_CREATE` 时，持有 `i_data_sem` 写锁调用 `ext4_map_create_blocks()` 分配或转换块。
6. 新分配且 ordered data 模式下，会把数据范围加入 jbd2 inode 的 ordered data 列表。
7. 记录 fast commit range：`ext4_fc_track_range()`。

因此 extent 层既是读路径查找物理块的位置，也是写路径分配、延迟分配落盘、unwritten extent 转 written extent、truncate/punch hole 修改空间布局的核心。

### journal/jbd2

`fs/ext4/ext4_jbd2.h` 是 ext4 对 jbd2 的包装层。常用模式是：

```text
handle = ext4_journal_start(inode 或 sb, type, credits)
  -> 修改 inode、目录块、位图、extent tree、superblock 等 metadata
  -> ext4_journal_get_write_access()
  -> ext4_handle_dirty_metadata()
  -> ext4_mark_inode_dirty()
ext4_journal_stop(handle)
```

关键包装包括：

- `ext4_journal_start()`、`ext4_journal_start_sb()`、`ext4_journal_stop()`。
- `ext4_journal_extend()`、`ext4_journal_restart()`、`ext4_journal_ensure_credits()`：处理事务 credit 不足。
- `ext4_handle_valid()`：无 journal 模式下 handle 可以是特殊非 jbd2 值。
- `ext4_jbd2_inode_add_write()`、`ext4_jbd2_inode_add_wait()`：把 ordered data 范围登记到 jbd2 inode。
- `ext4_update_inode_fsync_trans()`：记录后续 fsync 需要等待的事务号。
- `ext4_journal_force_commit()`/`ext4_force_commit()`：强制提交事务。

在写路径中的典型作用：

- buffered 非 delalloc 写：`ext4_write_begin()` 在 block instantiation 和 `write_end` 之间持有同一个 transaction handle，避免分配块与提交数据之间崩溃导致不一致。
- delalloc writeback：`ext4_writepages()` 启动 journal handle 后调用 `ext4_map_blocks()` 真正分配块，并把映射后的页提交 BIO。
- DIO/DAX：`ext4_iomap_alloc()` 在需要分配或转换 extent 时启动 journal handle。
- extending write：通过 orphan list 保护，防止崩溃后留下超过 inode size 的已分配块或未完成扩展。
- fsync：根据 inode 记录的事务号等待 fast commit/full commit，并在需要时发设备 flush。

## 4. ext4 和 VFS/page cache 的衔接点

### VFS 对象上的衔接

- `struct super_block.s_op = &ext4_sops`：superblock 生命周期、sync、freeze、statfs、inode 写回等。
- `struct inode.i_op`：文件名空间和 inode 元数据操作，如 create、lookup、setattr、getattr、fiemap。
- `struct inode.i_fop`：打开文件后的读写、mmap、fsync、ioctl、splice、fallocate。
- `struct address_space.i_mapping->a_ops`：page cache 与文件系统块映射、写回、失效、迁移的接口。

### address_space_operations

`ext4_set_aops()` 根据 inode 的数据模式和挂载选项选择：

- `ext4_aops`：普通非 delalloc buffered I/O。
- `ext4_da_aops`：delalloc buffered I/O。
- `ext4_journalled_aops`：data=journal 模式。
- `ext4_dax_aops`：DAX inode。

几个核心回调：

- `.read_folio = ext4_read_folio`
- `.readahead = ext4_readahead`
- `.writepages = ext4_writepages`
- `.write_begin = ext4_write_begin` 或 `ext4_da_write_begin`
- `.write_end = ext4_write_end`、`ext4_da_write_end` 或 `ext4_journalled_write_end`
- `.dirty_folio = ext4_dirty_folio` 或 `ext4_journalled_dirty_folio`
- `.invalidate_folio = ext4_invalidate_folio` 或 `ext4_journalled_invalidate_folio`
- `.release_folio = ext4_release_folio`
- `.bmap = ext4_bmap`

### page cache 读写关系

buffered read：

- VFS/generic 层优先从 page cache 满足读。
- 缺页或 readahead 触发 `ext4_read_folio()`/`ext4_readahead()`。
- ext4 通过 `ext4_map_blocks()` 获取物理块，组 BIO 读入 folio。
- hole 部分会 zero fill。
- fscrypt/fsverity 通过 BIO 后处理完成解密和验证。

buffered write：

- `generic_perform_write()` 负责通用的 page cache 写入循环。
- ext4 的 `write_begin` 负责准备 folio、buffer heads、块映射或延迟分配、journal handle。
- 用户数据 copy 到 page cache 后，ext4 的 `write_end` 更新 buffer/folio dirty 状态、`i_size`/`i_disksize` 和 inode dirty 状态。
- 后续 writeback 调用 `ext4_writepages()`，delalloc 模式下这一步才真正分配物理块。

direct I/O 和 DAX：

- 绕过 page cache 的数据拷贝路径，但仍需要与 page cache 保持一致。
- DIO 通过 `iomap_dio_rw()` 使用 `ext4_iomap_ops` 获取映射。
- DAX 通过 `dax_iomap_rw()` 和 `dax_iomap_fault()` 使用同一套 iomap 映射。
- DIO 回退 buffered 或覆盖已有 page cache 时，会使用 `filemap_write_and_wait_range()`、`invalidate_mapping_pages()` 等同步/失效 page cache。

## 5. 开发 ext4 patch 时的主要风险

1. journal credit 估算错误
   ext4 很多路径在持有 handle 时修改多个 metadata：inode、extent tree、block bitmap、group descriptor、quota、目录块、orphan 信息等。credit 少了可能导致事务重启时机错误、死锁或失败路径复杂化；credit 多了会影响性能和事务大小。

2. 崩溃一致性和 orphan list 处理错误
   extending write、truncate、punch hole、rename、unlink 等路径都依赖 journal 与 orphan 机制保证 replay 后一致。漏掉 `ext4_orphan_add()`/`ext4_orphan_del()` 或错误更新 `i_disksize`，可能造成空间泄漏、旧数据暴露或文件大小不一致。

3. page cache、buffer head、extent 状态不一致
   buffered write、writeback、truncate、invalidate、DIO fallback 都会同时接触 page cache、buffer head 状态和 extent status tree。错误设置 `BH_New/BH_Unwritten/BH_Delay` 或 `EXT4_MAP_*` 标志，可能导致 stale data exposure、重复写回、hole 误判。

4. delalloc 与 writeback 交互复杂
   delalloc 将分配推迟到 `ext4_writepages()`，错误处理发生在回写上下文，用户态 write 可能早已返回成功。ENOSPC、journal abort、reserved blocks、quota 和 writeback retry 都要谨慎处理。

5. unwritten extent 转换风险
   DIO、dioread_nolock、预分配和 delayed allocation 都可能使用 unwritten extent。转换时机如果早于数据落盘或遗漏失败处理，会造成读到未初始化数据或崩溃后 extent 状态错误。

6. 锁顺序和并发风险
   ext4 源码在 `super.c` 顶部明确列出多条锁顺序，典型涉及 `sb_start_write`、`i_rwsem`、`invalidate_lock`、journal transaction、folio lock、`i_data_sem`。新增路径如果调整锁顺序，容易触发 lockdep 或真实死锁。

7. fscrypt/fsverity/DAX/inline data/quota/fast commit 特性组合
   很多路径都有特性分支。只在默认挂载选项下测试 patch 容易漏掉 data=journal、nojournal、DAX、inline data、encrypted、verity、bigalloc、quota、fast commit 等组合。

8. VFS 语义回归
   `O_SYNC`、`fdatasync`、`IOCB_NOWAIT`、`IOCB_DIRECT`、atomic write、splice、mmap/page fault、truncate 和 seek hole/data 都有 VFS 可见语义。改 ext4 内部实现时需要确认这些语义没有被破坏。

9. 错误路径和 abort 状态处理
   ext4 有 `ext4_emergency_state()`、`ext4_forced_shutdown()`、journal abort、只读 remount 等状态。错误路径如果继续写 metadata 或不传播错误，可能扩大损坏。

10. 性能回退
    读写路径高度优化，例如 extent status cache、delalloc、mballoc、BIO 合并、iomap direct I/O、fast commit。看似安全的同步、flush、事务提交或锁粒度调整，可能造成明显性能退化。

## 6. 不确定点与未深入内容

- 本报告只按任务重点阅读了 ext4 的核心入口和局部相关文件；没有完整展开 `fs/ext4/extents.c`、`fs/ext4/mballoc.c`、`fs/ext4/fast_commit.c`、`fs/jbd2/*` 的内部实现，因此 extent 分裂/合并、mballoc 策略、fast commit record 格式、jbd2 transaction/checkpoint 细节没有细讲。
- `fsync` 的入口在 `fs/ext4/fsync.c`，不在用户列出的重点文件中；报告中关于 fsync 的调用链来自额外阅读该文件。
- direct I/O completion、unwritten extent end-I/O 转换、atomic write 的完整失败恢复路径没有逐行验证，只基于 `file.c`、`inode.c` 中入口和 iomap 分配路径总结。
- 不同内核配置会影响实际路径，例如 `CONFIG_FS_DAX`、`CONFIG_FS_ENCRYPTION`、`CONFIG_FS_VERITY`、quota、fast commit。报告描述的是源码中可见的条件分支，不代表所有运行环境都会启用。
- 没有运行 fstests、xfstests 或崩溃恢复测试；本文是代码阅读报告，不是行为验证报告。
