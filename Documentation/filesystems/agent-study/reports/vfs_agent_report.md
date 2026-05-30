# VFS Agent 报告：核心路径与核心对象关系

本报告基于当前分支 `agent-fs-study-cn` 的源码阅读，重点文件为 `fs/open.c`、`fs/read_write.c`、`fs/namei.c`、`fs/file_table.c`、`include/linux/fs.h`，并为说明 VFS 与 ext4 的衔接补充阅读了 `fs/ext4/` 下的相关实现。

## 1. VFS 在文件系统栈中的位置

VFS（Virtual File System）位于系统调用/文件描述符层和具体文件系统实现之间。它向上承接 `open(2)`、`read(2)`、`write(2)` 等系统调用，向下通过统一的对象和操作表调用 ext4、xfs、tmpfs、procfs 等具体文件系统。它不是一种磁盘格式，而是一组统一的内核对象模型、路径解析逻辑、权限检查逻辑、页缓存/地址空间接口和操作分派机制。

简化位置如下：

```text
用户态程序
  -> 系统调用入口（open/read/write）
  -> VFS：fd table、path/namei、dcache、inode、file、super_block、权限/LSM/fsnotify
  -> 具体文件系统：ext4_file_operations、ext4_dir_inode_operations、ext4_sops 等
  -> 页缓存/块层/设备驱动
  -> 存储介质
```

VFS 的核心价值是把“路径名解析、文件描述符、打开文件实例、目录项缓存、inode 元数据、挂载实例、权限检查”等通用部分抽象出来，具体文件系统只需要填充对应的操作表和私有数据。

## 2. open/read/write 的关键调用链

### open 路径

主要源码：`fs/open.c`、`fs/namei.c`、`fs/file_table.c`、`include/linux/file.h`、`fs/file.c`。

关键调用链：

```text
fs/open.c:SYSCALL_DEFINE3(open)
fs/open.c:SYSCALL_DEFINE4(openat)
fs/open.c:SYSCALL_DEFINE4(openat2)
  -> fs/open.c:do_sys_open()
  -> fs/open.c:do_sys_openat2()
       build_open_flags()
       CLASS(filename, name)(filename)
       FD_ADD(how->flags, do_file_open(dfd, name, &op))
  -> fs/namei.c:do_file_open()
  -> fs/namei.c:path_openat()
       fs/file_table.c:alloc_empty_file()
       path_init()
       link_path_walk()
       open_last_lookups()
       do_open()
  -> fs/namei.c:open_last_lookups()
       lookup_fast_for_open()
       lookup_open()
       可能调用 dir_inode->i_op->atomic_open()
       否则可能调用 dir_inode->i_op->lookup()
       O_CREAT 时可能调用 dir_inode->i_op->create()
  -> fs/namei.c:do_open()
       complete_walk()
       may_open()
       fs/open.c:vfs_open()
       security_file_post_open()
       O_TRUNC 时 handle_truncate()
  -> fs/open.c:vfs_open()
  -> fs/open.c:do_dentry_open()
       file->f_inode = dentry->d_inode
       file->f_mapping = inode->i_mapping
       file->f_op = fops_get(inode->i_fop)
       调用 file->f_op->open(inode, file)（如果存在）
       设置 FMODE_OPENED / FMODE_CAN_READ / FMODE_CAN_WRITE 等
  -> include/linux/file.h:FD_ADD()
  -> fs/file.c:fd_install()
       把 struct file 安装进当前进程 fd table
```

`fs/file_table.c:alloc_empty_file()` 只分配并初始化一个“空的” `struct file`，其中 `init_file()` 设置 `f_cred`、`f_flags`、`f_mode`、`f_pos`、引用计数等，但此时还没有绑定具体路径和 inode。真正把 `file` 绑定到 `dentry/inode/file_operations` 的动作发生在 `fs/open.c:do_dentry_open()`。

`fs/namei.c` 是 open 路径中最复杂的部分：`link_path_walk()` 负责逐级解析路径组件；`open_last_lookups()` 处理最后一个路径组件；`lookup_open()` 在需要时调用具体文件系统的 `inode_operations`，例如 `lookup/create/atomic_open`；`do_open()` 完成权限检查和真正打开。

### read 路径

主要源码：`fs/read_write.c`、`include/linux/fs.h`。

关键调用链：

```text
fs/read_write.c:SYSCALL_DEFINE3(read)
  -> fs/read_write.c:ksys_read()
       CLASS(fd_pos, f)(fd)
       file_ppos()
       vfs_read(fd_file(f), buf, count, ppos)
  -> fs/read_write.c:vfs_read()
       检查 FMODE_READ / FMODE_CAN_READ
       access_ok()
       rw_verify_area(READ, file, pos, count)
       如果 file->f_op->read 存在，调用 ->read()
       否则如果 file->f_op->read_iter 存在，调用 new_sync_read()
  -> fs/read_write.c:new_sync_read()
       init_sync_kiocb()
       iov_iter_ubuf()
       file->f_op->read_iter(&kiocb, &iter)
```

对 ext4 普通文件而言，`file->f_op` 来自 `fs/ext4/file.c:ext4_file_operations`，读接口是 `.read_iter = ext4_file_read_iter`。`ext4_file_read_iter()` 根据 DAX、Direct I/O、普通 buffered I/O 分支进一步走 `dax_iomap_rw()`、`iomap_dio_rw()` 或 `generic_file_read_iter()`。

### write 路径

主要源码：`fs/read_write.c`、`include/linux/fs.h`。

关键调用链：

```text
fs/read_write.c:SYSCALL_DEFINE3(write)
  -> fs/read_write.c:ksys_write()
       CLASS(fd_pos, f)(fd)
       file_ppos()
       vfs_write(fd_file(f), buf, count, ppos)
  -> fs/read_write.c:vfs_write()
       检查 FMODE_WRITE / FMODE_CAN_WRITE
       access_ok()
       rw_verify_area(WRITE, file, pos, count)
       file_start_write(file)
       如果 file->f_op->write 存在，调用 ->write()
       否则如果 file->f_op->write_iter 存在，调用 new_sync_write()
       file_end_write(file)
  -> fs/read_write.c:new_sync_write()
       init_sync_kiocb()
       iov_iter_ubuf()
       file->f_op->write_iter(&kiocb, &iter)
```

`include/linux/fs.h:file_start_write()` 对普通文件调用 `sb_start_write(file_inode(file)->i_sb)`，用于与 filesystem freeze 机制配合；`file_end_write()` 对应 `sb_end_write()`。

对 ext4 普通文件而言，写接口是 `fs/ext4/file.c:ext4_file_operations` 中的 `.write_iter = ext4_file_write_iter`。`ext4_file_write_iter()` 会根据 DAX、Direct I/O、atomic write、buffered write 等条件分派到 ext4 自己的写路径，并可能进入 iomap、页缓存、日志和块分配相关逻辑。

## 3. 核心对象关系

### struct super_block

`struct super_block` 表示一个已挂载文件系统实例。当前源码树中它的定义在 `include/linux/fs/super_types.h`，由 `include/linux/fs.h -> include/linux/fs/super.h -> include/linux/fs/super_types.h` 引入。

关键字段包括：

- `s_type`：指向 `struct file_system_type`，表示文件系统类型。
- `s_op`：指向 `struct super_operations`，提供 inode 分配、写回、卸载、statfs、freeze/thaw 等超级块级操作。
- `s_root`：该文件系统根 dentry。
- `s_bdev` / `s_bdev_file`：块设备相关信息。
- `s_fs_info`：具体文件系统私有信息，ext4 中通常通过 `EXT4_SB(sb)` 访问。

### struct inode

`include/linux/fs.h:struct inode` 表示文件系统对象的元数据和身份，不等同于一次打开的文件。关键字段包括：

- `i_op`：`const struct inode_operations *`，用于 lookup、create、mkdir、rename、permission、getattr、setattr 等命名空间和元数据操作。
- `i_fop`：`const struct file_operations *`，该 inode 被打开成 `struct file` 时默认使用的文件操作表。
- `i_sb`：所属 `super_block`。
- `i_mapping` / `i_data`：地址空间和页缓存相关对象。
- `i_private`：具体文件系统或驱动私有数据。

一个 inode 可以对应多个 dentry，例如硬链接或别名场景。

### struct dentry

`include/linux/dcache.h:struct dentry` 表示目录项缓存中的“名字到 inode”的关系。关键字段包括：

- `d_parent`、`d_name`：父目录项和当前名字。
- `d_inode`：该名字对应的 inode；为 `NULL` 时是 negative dentry。
- `d_sb`：所属 dentry 树的 super_block。
- `d_op`：`struct dentry_operations`，用于 revalidate、hash、compare、delete 等 dcache 行为。

路径解析时，VFS 先尽量通过 dcache 快速找到 dentry；miss 或需要 revalidate 时再调用具体文件系统的 `inode_operations.lookup` 等方法。

### struct file

`include/linux/fs.h:struct file` 表示一次打开后的文件实例，和用户态 fd 对应但不是 fd 本身。关键字段包括：

- `f_path`：`struct path`，包含 `vfsmount` 和 `dentry`。
- `f_inode`：缓存的 inode，`file_inode(file)` 返回它。
- `f_op`：打开后使用的 `file_operations`。
- `f_mapping`：通常来自 `inode->i_mapping`。
- `f_pos`：当前文件偏移。
- `f_flags` / `f_mode`：打开标志和 VFS 模式位。
- `private_data`：具体文件系统或驱动可用的 per-open 私有数据。

`fs/open.c:do_dentry_open()` 会从 `file->f_path.dentry->d_inode` 得到 inode，再把 `inode->i_fop` 复制/引用到 `file->f_op`。因此读写路径只需要看 `file->f_op`，不必重新根据文件系统类型判断。

### file_operations 与 inode_operations

`include/linux/fs.h:struct file_operations` 面向“已经打开的文件实例”，典型成员有：

- `open`
- `read` / `read_iter`
- `write` / `write_iter`
- `llseek`
- `mmap`
- `fsync`
- `release`
- `iterate_shared`

`include/linux/fs.h:struct inode_operations` 面向“文件系统命名空间和 inode 元数据”，典型成员有：

- `lookup`
- `create`
- `link` / `unlink`
- `mkdir` / `rmdir`
- `rename`
- `permission`
- `getattr` / `setattr`
- `atomic_open`
- `tmpfile`

粗略理解：`inode_operations` 解决“这个名字是什么、能否创建/删除/改属性”；`file_operations` 解决“已经打开后如何读写、seek、mmap、fsync、关闭”。

对象关系可简化为：

```text
task fd table
  -> fd number
  -> struct file
       -> f_path.mnt -> struct vfsmount -> mnt_sb -> struct super_block
       -> f_path.dentry -> struct dentry -> d_inode -> struct inode
       -> f_inode ------------------------------^
       -> f_op：来自 inode->i_fop

struct inode
  -> i_sb：所属 super_block
  -> i_op：命名空间/元数据操作
  -> i_fop：默认 file_operations
  -> i_mapping：页缓存/address_space

struct super_block
  -> s_root：根 dentry
  -> s_op：super_operations
  -> s_fs_info：具体文件系统私有数据
```

## 4. VFS 与 ext4 的衔接点

### 挂载和 super_block

`fs/ext4/super.c:ext4_sops` 填充 `struct super_operations`，包括：

- `.alloc_inode = ext4_alloc_inode`
- `.write_inode = ext4_write_inode`
- `.evict_inode = ext4_evict_inode`
- `.put_super = ext4_put_super`
- `.sync_fs = ext4_sync_fs`
- `.statfs = ext4_statfs`

`fs/ext4/super.c:ext4_fill_super()` 中将 VFS super_block 接到 ext4：

```text
sb->s_op = &ext4_sops
sb->s_export_op = &ext4_export_ops
sb->s_xattr = ext4_xattr_handlers
root = ext4_iget(sb, EXT4_ROOT_INO, EXT4_IGET_SPECIAL)
sb->s_root = d_make_root(root)
```

这一步完成了“这个挂载实例由 ext4 管理”的绑定。

### inode 装配 i_op / i_fop

`fs/ext4/inode.c:__ext4_iget()` 从磁盘读取 inode 后，根据 `i_mode` 设置 VFS inode 的操作表：

- 普通文件：`inode->i_op = &ext4_file_inode_operations`，`inode->i_fop = &ext4_file_operations`，并调用 `ext4_set_aops(inode)`。
- 目录：`inode->i_op = &ext4_dir_inode_operations`，`inode->i_fop = &ext4_dir_operations`。
- 符号链接：设置到 ext4 的 symlink inode operations。
- 特殊文件：设置到 `ext4_special_inode_operations`。

这一步是 VFS 后续 open/read/write 能分派到 ext4 的关键。

### 目录 lookup/create

`fs/ext4/namei.c:ext4_dir_inode_operations` 填充目录 inode 的命名空间操作：

```text
.create = ext4_create
.lookup = ext4_lookup
.link   = ext4_link
.unlink = ext4_unlink
.mkdir  = ext4_mkdir
.rename = ext4_rename2
.tmpfile = ext4_tmpfile
```

`fs/namei.c:lookup_open()` 在 O_CREAT 或 lookup miss 等场景会调用这些方法。对 ext4：

- `fs/ext4/namei.c:ext4_lookup()` 读取 ext4 目录项，找到 inode 号后调用 `ext4_iget()`，最后用 `d_splice_alias(inode, dentry)` 把 inode 接入 dentry。
- `fs/ext4/namei.c:ext4_create()` 调用 `ext4_new_inode_start_handle()` 分配新 inode，设置 `inode->i_op` 和 `inode->i_fop`，再通过 `ext4_add_nondir()` 添加目录项。
- `fs/ext4/namei.c:ext4_add_nondir()` 成功后调用 `d_instantiate_new(dentry, inode)`，把原本 negative 的 dentry 变为 positive dentry。

当前源码树里的 `ext4_dir_inode_operations` 没有设置 `.atomic_open`，所以普通 ext4 创建/查找路径主要走 `lookup/create`，不是 `atomic_open`。

### 打开和读写

`fs/open.c:do_dentry_open()` 从 ext4 inode 上取得 `inode->i_fop`，因此普通 ext4 文件打开后：

```text
file->f_op = &ext4_file_operations
```

`fs/ext4/file.c:ext4_file_operations` 中关键成员包括：

- `.read_iter = ext4_file_read_iter`
- `.write_iter = ext4_file_write_iter`
- `.open = ext4_file_open`
- `.release = ext4_release_file`
- `.fsync = ext4_sync_file`
- `.llseek = ext4_llseek`
- `.mmap_prepare = ext4_file_mmap_prepare`

因此 VFS 的 `vfs_read()`/`vfs_write()` 最终通过 `file->f_op->read_iter/write_iter` 进入 ext4 文件读写实现。

## 5. 学习难点和后续建议

学习难点：

- `fs/namei.c` 分支很多：RCU walk/ref walk、符号链接、挂载点、`O_CREAT`、`O_EXCL`、`O_TRUNC`、`O_PATH`、`O_TMPFILE` 都交织在路径解析里。
- `struct file` 和 fd 不是一回事：fd 是进程 fd table 的索引，`struct file` 是打开文件对象，二者通过 `fd_install()` 连接。
- `dentry` 和 `inode` 的关系容易混淆：dentry 是名字缓存，可以为 negative；inode 是文件对象元数据；一个 inode 可有多个 dentry 别名。
- `file_operations` 和 `inode_operations` 的边界需要反复结合调用点理解：open 的最后一跳同时涉及两者，路径解析阶段偏 `inode_operations`，打开后的读写偏 `file_operations`。
- ext4 读写并不止 VFS 到 ext4 一跳，后面还有 page cache、iomap、DAX、Direct I/O、jbd2 日志、块分配、writeback 等多层路径。
- 锁和生命周期复杂：`i_rwsem`、RCU path walk、引用计数、`fput()` 延迟释放、freeze protection 等都影响真实行为。

后续建议：

- 先固定一个最小场景跟踪：`open("a", O_RDONLY)`、`read(fd)`、`write(fd)`，再逐步加入 `O_CREAT/O_TRUNC/O_DIRECT/O_PATH`。
- 配合 `Documentation/filesystems/path-lookup.rst`、`Documentation/filesystems/vfs.rst`、`Documentation/filesystems/locking.rst` 阅读 `fs/namei.c`。
- 用 ftrace/bpftrace 或内核 tracepoints 观察 `do_sys_openat2`、`path_openat`、`vfs_open`、`vfs_read`、`vfs_write`、`ext4_file_read_iter`、`ext4_file_write_iter` 的真实运行顺序。
- 画对象生命周期图：fd table 引用 `file`，`file` 引用 `path`，`path` 引用 `vfsmount+dentry`，`dentry` 引用 `inode`，`inode` 引用 `super_block`。
- 对 ext4 建议继续追 `ext4_file_write_iter()` 后面的 buffered write、direct I/O、journal transaction 和 writeback 路径。

## 6. 不确定点

- 本报告只梳理 open/read/write 主干，没有完整覆盖 `readv/writev/pread/pwrite/io_uring/aio/splice/mmap` 等变体路径。
- 没有完整展开 `openat2` 的所有 `RESOLVE_*` 语义和 scoped lookup 安全细节，只确认它们进入 `do_sys_openat2()` 和 namei lookup flags 相关逻辑。
- 对 RCU path walk、rename 并发、mount namespace 穿越、automount、网络文件系统 revalidate 等并发细节仍需继续阅读，不能仅凭本报告下结论。
- ext4 写路径后半段涉及 jbd2、delayed allocation、iomap、page cache、writeback 和块层，本报告只说明 VFS 到 ext4 的衔接点，没有声称完整解释数据落盘顺序。
- 当前源码树中 `struct super_block` 不直接定义在 `include/linux/fs.h`，而是在 `include/linux/fs/super_types.h`，由 `include/linux/fs.h` 间接包含；如果其他内核版本布局不同，需要按对应版本源码确认。
- 不同具体文件系统是否实现 `.atomic_open`、`.read`、`.read_iter`、`.write`、`.write_iter` 差异较大，不能把 ext4 的实现方式直接推广到所有文件系统。
