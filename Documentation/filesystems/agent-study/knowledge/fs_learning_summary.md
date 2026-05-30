# Linux 文件系统源码学习总览

## 本轮学习范围

本轮多 agent 协同学习基于 Linux 源码提交 `f5e5d35`，重点覆盖以下目录和文件：

- VFS：`fs/open.c`、`fs/read_write.c`、`fs/namei.c`、`fs/file_table.c`、`include/linux/fs.h`
- ext4：`fs/ext4/file.c`、`fs/ext4/inode.c`、`fs/ext4/super.c`、`fs/ext4/namei.c`、`fs/ext4/ext4.h`
- page cache/writeback：`mm/filemap.c`、`mm/readahead.c`、`mm/page-writeback.c`、`mm/truncate.c`、`fs/buffer.c`
- 测试：`fs/ext4/.kunitconfig`、`fs/ext4/*-test.c`、`tools/testing/selftests/filesystems/`、`Documentation/filesystems/`

本目录下的报告由多个子 agent 分工完成，再由 Manager Agent 汇总。每份报告都要求引用具体源码文件和函数，并明确标注不确定点。

## 文件系统栈的分层

```text
用户态程序
  -> open/read/write/fsync 等系统调用
  -> VFS：fd table、path lookup、dentry、inode、file、super_block
  -> 具体文件系统：ext4_file_operations、ext4_sops、ext4 address_space_operations
  -> page cache / iomap / buffer_head / writeback
  -> block layer / device
```

VFS 不代表某种磁盘格式，它负责统一对象模型和调用分派。ext4 是具体文件系统实现，通过 VFS 的操作表接入系统调用路径。page cache 位于通用 MM 层与具体文件系统之间，用 `struct address_space` 和 `address_space_operations` 把缓存、预读、脏页、写回连接起来。

## 核心对象关系

- `struct super_block`：一次挂载实例，保存 `s_op`、`s_root`、`s_fs_info` 等。
- `struct inode`：文件对象的元数据，保存 `i_op`、`i_fop`、`i_mapping`、`i_sb`。
- `struct dentry`：目录项缓存，表示“名字到 inode”的关系，可为 negative dentry。
- `struct file`：一次打开后的文件实例，保存 `f_path`、`f_inode`、`f_op`、`f_mapping`、`f_pos`。
- `struct address_space`：文件 page cache 的宿主，保存 `i_pages`、`a_ops`、`host`、dirty/writeback 状态。

`fs/open.c::do_dentry_open()` 是理解 VFS 分派的关键点：它从 `dentry` 找到 `inode`，再把 `inode->i_fop` 绑定到本次打开的 `struct file`。后续 `vfs_read()`、`vfs_write()` 就通过 `file->f_op` 调用具体文件系统。

## 主调用链

### open

```text
open/openat/openat2
  -> do_sys_open()
  -> do_sys_openat2()
  -> do_file_open()
  -> path_openat()
  -> open_last_lookups()
  -> do_open()
  -> vfs_open()
  -> do_dentry_open()
  -> file->f_op->open()，例如 ext4_file_open()
```

### read

```text
read()
  -> ksys_read()
  -> vfs_read()
  -> new_sync_read()
  -> file->f_op->read_iter()
  -> ext4_file_read_iter()
  -> generic_file_read_iter()
  -> filemap_read()
  -> mapping->a_ops->read_folio()/readahead()
```

Direct I/O 和 DAX 会在 ext4 层分流到 `iomap_dio_rw()` 或 `dax_iomap_rw()`。

### write

```text
write()
  -> ksys_write()
  -> vfs_write()
  -> new_sync_write()
  -> file->f_op->write_iter()
  -> ext4_file_write_iter()
  -> ext4_dio_write_iter() 或 ext4_buffered_write_iter()
  -> generic_perform_write()
  -> mapping->a_ops->write_begin()/write_end()
  -> 后续 writeback 进入 ext4_writepages()
```

buffered write 先写 page cache，真正块分配和落盘可能被 delayed allocation 推迟到 writeback。Direct I/O 绕过 page cache 的数据面，但进入和退出时仍要处理 page cache 一致性。

## ext4 的接入点

- `fs/ext4/super.c::ext4_init_fs()` 调用 `register_filesystem()` 注册 ext4。
- `fs/ext4/super.c::__ext4_fill_super()` 设置 `sb->s_op = &ext4_sops`。
- `fs/ext4/inode.c::__ext4_iget()` 根据 inode 类型设置 `i_op`、`i_fop` 和 `a_ops`。
- `fs/ext4/namei.c::ext4_create()` 创建普通文件时设置 `ext4_file_inode_operations`、`ext4_file_operations` 和 `ext4_set_aops()`。
- `fs/ext4/file.c::ext4_file_operations` 提供 read/write/open/fsync 等入口。

## 学习结论

1. 学 VFS 时先抓住 `struct file` 的形成过程，再看 read/write 如何通过 `file_operations` 分派。
2. 学 ext4 时要把 `file_operations`、`inode_operations`、`super_operations` 和 `address_space_operations` 分开看。
3. 学 write path 时不能只看 `write_iter()`，还必须追 page cache、delalloc、writeback、journal 和 extent 映射。
4. ext4 patch 风险主要来自锁顺序、journal credit、崩溃一致性、page cache/DIO 混用、特性组合和错误路径。
5. 对学习型中文注释 patch，最重要的验证是确认 diff 只改注释和文档，不改变语义。

## 后续阅读路线

1. 先读 `Documentation/filesystems/vfs.rst`、`path-lookup.rst`、`locking.rst`。
2. 跟踪最小路径：`open("a", O_RDONLY)`、`read(fd)`、`write(fd)`。
3. 再加入 ext4：`ext4_file_open()`、`ext4_file_read_iter()`、`ext4_file_write_iter()`。
4. 然后深入 page cache：`generic_file_read_iter()`、`filemap_read()`、`generic_perform_write()`、`ext4_writepages()`。
5. 最后进入复杂专题：DIO、DAX、fsync、journal、extent、delalloc、truncate、rename、crash consistency。
