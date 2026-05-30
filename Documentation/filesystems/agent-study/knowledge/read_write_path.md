# VFS、ext4 与 Page Cache 读写路径

## 读路径

```text
fs/read_write.c::SYSCALL_DEFINE3(read)
  -> ksys_read()
  -> vfs_read()
      -> 权限、access_ok、rw_verify_area
      -> new_sync_read()
          -> file->f_op->read_iter()
             ext4: fs/ext4/file.c::ext4_file_read_iter()
               -> DAX: ext4_dax_read_iter()
               -> Direct I/O: ext4_dio_read_iter()
               -> Buffered: generic_file_read_iter()
                   -> mm/filemap.c::filemap_read()
                      -> page cache 命中：copy_folio_to_iter()
                      -> page cache miss：read_folio/readahead
                         ext4: ext4_read_folio()/ext4_readahead()
```

关键理解点：

- `vfs_read()` 只做通用检查和分派，不理解 ext4 磁盘布局。
- `ext4_file_read_iter()` 决定 DAX、Direct I/O、buffered read 的分支。
- `generic_file_read_iter()` 是大多数可直接使用 page cache 的文件系统的通用读入口。
- `filemap_read()` 管理 folio 查找、等待、预读和复制；真正读盘由 `mapping->a_ops` 交给文件系统。

## 写路径

```text
fs/read_write.c::SYSCALL_DEFINE3(write)
  -> ksys_write()
  -> vfs_write()
      -> 权限、access_ok、rw_verify_area
      -> file_start_write()
      -> new_sync_write()
          -> file->f_op->write_iter()
             ext4: fs/ext4/file.c::ext4_file_write_iter()
               -> DAX: ext4_dax_write_iter()
               -> Direct I/O: ext4_dio_write_iter()
               -> Buffered: ext4_buffered_write_iter()
                    -> generic_perform_write()
                       -> mapping->a_ops->write_begin()
                       -> copy_folio_from_iter_atomic()
                       -> mapping->a_ops->write_end()
                    -> generic_write_sync()
```

关键理解点：

- `file_start_write()` 和 `file_end_write()` 参与 freeze 保护。
- buffered write 写入 page cache，不代表马上写入磁盘。
- ext4 的 delayed allocation 可能把真正物理块分配推迟到 `ext4_writepages()`。
- Direct I/O 绕过 page cache 数据面，但仍需要等待或失效同范围 page cache，避免陈旧数据。

## writeback 路径

```text
dirty folio
  -> writeback 触发
  -> do_writepages()
  -> mapping->a_ops->writepages()
     ext4: ext4_writepages()
       -> ext4_do_writepages()
       -> ext4_map_blocks()
       -> 提交 BIO
       -> 处理 journal、unwritten extent、io_end
```

关键理解点：

- writeback 是 buffered write 后半段的重要部分。
- `writeback_control` 控制本轮写回范围、同步模式、写回数量。
- ext4 writeback 同时处理延迟分配、extent 映射、BIO 合并和 journal 一致性。

## fsync 路径

```text
fsync/fdatasync
  -> file->f_op->fsync()
     ext4: ext4_sync_file()
       -> file_write_and_wait_range()
       -> ext4_fsync_journal()
       -> ext4_fc_commit() 或 full commit
       -> 必要时 blkdev_issue_flush()
```

关键理解点：

- fsync 不只是“写数据”，还要确保相关 journal transaction 达到一致性要求。
- `fdatasync` 和 `fsync` 等待的元数据范围可能不同。
- ordered/journalled/nojournal 模式会影响具体路径。

## 学习用追踪点

可以用 ftrace/bpftrace 或 printk 风格实验观察这些函数：

- `do_sys_openat2`
- `path_openat`
- `do_dentry_open`
- `vfs_read`
- `vfs_write`
- `ext4_file_open`
- `ext4_file_read_iter`
- `ext4_file_write_iter`
- `generic_file_read_iter`
- `filemap_read`
- `generic_perform_write`
- `ext4_writepages`
- `ext4_sync_file`
