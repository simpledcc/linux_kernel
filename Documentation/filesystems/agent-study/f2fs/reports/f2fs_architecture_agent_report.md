# F2FS Architecture Agent Report

> 研究对象：当前工作区 `D:\demo\kernel\github-publish`，分支 `main`，读取到的提交为 `ae29b2f`。
> 重点源码：`fs/f2fs/super.c`、`fs/f2fs/f2fs.h`、`fs/f2fs/inode.c`、`fs/f2fs/namei.c`、`fs/f2fs/file.c`、`Documentation/filesystems/f2fs.rst`。

## 1. 设计目标和整体分层

F2FS 是面向 NAND flash/SSD/eMMC/SD 等闪存类设备的文件系统。`Documentation/filesystems/f2fs.rst` 的 Overview 和 Key Features 表明，它基于 Log-structured File System (LFS)，核心目标是适应闪存设备与传统旋转磁盘不同的写入和擦除特性，同时缓解经典 LFS 的两个问题：

- wandering tree：数据块 out-of-place 更新后会导致直接/间接索引、inode map、checkpoint 递归更新。F2FS 引入 node 概念，把 inode 和索引块统一为 node，并用 NAT(Node Address Table) 记录 node 的当前位置，降低地址变更向上传播。
- cleaning overhead：LFS 会产生大量失效块，需要清理回收。F2FS 用 SIT/SSA、后台 GC、greedy/cost-benefit victim 选择、多头日志、冷热数据分离、自适应 logging 等机制降低清理成本和前台延迟。

从代码结构看，F2FS 的整体分层可以按职责理解：

- VFS 接入层：`fs/f2fs/super.c` 注册文件系统、解析挂载上下文、填充 `super_block`；`fs/f2fs/namei.c`/`file.c`/`dir.c` 提供 inode/file/directory operations。
- inode 与页缓存层：`fs/f2fs/inode.c` 负责 VFS inode 和磁盘 inode/node 的装配、回写、淘汰；`fs/f2fs/data.c` 提供普通数据页的 address_space_operations。
- node 管理层：`fs/f2fs/node.c` 维护 NAT、nid 分配、node page 读写；入口结构是 `struct f2fs_nm_info`。
- segment 管理层：`fs/f2fs/segment.c` 维护 SIT/free/dirty/curseg、discard/flush、冷热日志与空间分配；入口结构是 `struct f2fs_sm_info`。
- checkpoint/recovery/GC 层：`checkpoint.c`、`recovery.c`、`gc.c` 等围绕一致性点、roll-forward recovery、orphan inode、清理回收工作。
- 可选能力层：加密、verity、quota、compression、casefold、zoned block device 等通过 super/inode/mapping 的条件字段和挂载选项接入。

## 2. mount/register/fill_super 关键调用链

### register/module 初始化路径

源码入口在 `fs/f2fs/super.c`：

1. `module_init(init_f2fs_fs)` 注册模块初始化入口。
2. `init_f2fs_fs()` 先初始化全局缓存和子系统：`init_inodecache()`、`f2fs_create_node_manager_caches()`、`f2fs_create_segment_manager_caches()`、`f2fs_create_checkpoint_caches()`、`f2fs_create_recovery_cache()`、`f2fs_create_extent_cache()`、`f2fs_create_garbage_collection_cache()`、sysfs/shrinker/post-read/iostat/bio/compress/casefold/xattr 等。
3. 初始化成功后调用 `register_filesystem(&f2fs_fs_type)`。
4. `f2fs_fs_type` 的关键字段为 `.name = "f2fs"`、`.init_fs_context = f2fs_init_fs_context`、`.kill_sb = kill_f2fs_super`、`.fs_flags = FS_REQUIRES_DEV | FS_ALLOW_IDMAP`。

卸载模块路径是 `module_exit(exit_f2fs_fs)`，由 `exit_f2fs_fs()` 调用 `unregister_filesystem(&f2fs_fs_type)` 并销毁各类缓存。

### mount/fs_context 路径

挂载时 VFS 通过 `f2fs_fs_type.init_fs_context` 进入：

1. `f2fs_init_fs_context(struct fs_context *fc)` 分配 `struct f2fs_fs_context`，挂到 `fc->fs_private`，并设置 `fc->ops = &f2fs_context_ops`。
2. `f2fs_context_ops` 中 `.parse_param = f2fs_parse_param`，`.get_tree = f2fs_get_tree`，`.reconfigure = f2fs_reconfigure`，`.free = f2fs_fc_free`。
3. `f2fs_get_tree()` 调用 `get_tree_bdev(fc, f2fs_fill_super)`，说明 F2FS 是块设备型文件系统，真正填充 superblock 的函数是 `f2fs_fill_super()`。

### f2fs_fill_super 主流程

`fs/f2fs/super.c::f2fs_fill_super()` 是挂载主干，关键阶段如下：

1. 分配并初始化 `struct f2fs_sb_info`，设置 `sbi->sb = sb`，初始化 GC/CP/node/flush/quota/inode list 等锁和链表。
2. `sb_set_blocksize(sb, F2FS_BLKSIZE)` 设置块大小。
3. `read_raw_super_block()` 读取并校验磁盘 super block，随后 `sb->s_fs_info = sbi`、`sbi->raw_super = raw_super`。
4. `default_options()`、`f2fs_check_opt_consistency()`、`f2fs_apply_options()`、`f2fs_sanity_check_options()` 处理挂载选项。
5. 设置 VFS superblock 能力：`sb->s_maxbytes`、`sb->s_max_links`、quota ops、`sb->s_op = &f2fs_sops`、`sb->s_cop`、`sb->s_vop`、`sb->s_xattr`、`sb->s_export_op`、`sb->s_magic`、`sb->s_time_gran`、`SB_POSIXACL/SB_INLINECRYPT/SB_LAZYTIME` 等。
6. `f2fs_init_write_merge_io()`、`init_sb_info()`、`f2fs_init_iostat()`、`init_percpu_info()`、`f2fs_init_page_array_cache()` 初始化内存状态。
7. 读取内部 meta inode：`sbi->meta_inode = f2fs_iget(sb, F2FS_META_INO(sbi))`。
8. `f2fs_get_valid_checkpoint()` 读取有效 checkpoint，并据此恢复 `sbi` 中的计数、flag、用户块数量等。
9. `f2fs_scan_devices()`、`f2fs_init_post_read_wq()`、extent cache、ino entry、fsync node、checkpoint request control 初始化。
10. `f2fs_build_segment_manager(sbi)` 建立 segment manager，再 `f2fs_build_node_manager(sbi)` 建立 node manager。
11. `f2fs_build_gc_manager()`、`f2fs_build_stats()` 后读取 node inode：`sbi->node_inode = f2fs_iget(sb, F2FS_NODE_INO(sbi))`。
12. 读取 root inode：`root = f2fs_iget(sb, F2FS_ROOT_INO(sbi))`，校验它是有效目录，然后 `generic_set_sb_d_ops(sb)`、`sb->s_root = d_make_root(root)`。
13. 后续还会初始化 compression inode、注册 sysfs、启用 quota、恢复 orphan inode、根据挂载选项执行 roll-forward recovery 等。

## 3. 核心结构体关系

### f2fs_sb_info

`fs/f2fs/f2fs.h::struct f2fs_sb_info` 是每个挂载实例的 F2FS 私有总控结构，通过 `sb->s_fs_info` 挂在 VFS `struct super_block` 上，并由 `F2FS_SB(sb)` 取回。它包含：

- VFS/磁盘 superblock 指针：`struct super_block *sb`、`struct f2fs_super_block *raw_super`。
- node 管理入口：`struct f2fs_nm_info *nm_info`、内部 `node_inode`。
- segment 管理入口：`struct f2fs_sm_info *sm_info`。
- checkpoint/meta：`struct f2fs_checkpoint *ckpt`、`meta_inode`、CP 锁和请求控制。
- inode 管理、dirty list、orphan/ino entry、extent cache、GC、mount options、计数器、write IO、zoned/device 信息等。

常用访问宏：

- `F2FS_SB(sb)`：从 VFS superblock 取 `sbi`。
- `F2FS_I_SB(inode)`：从 VFS inode 取其所在 `sbi`。
- `NM_I(sbi)`：取 `sbi->nm_info`。
- `SM_I(sbi)`：取 `sbi->sm_info`。
- `SIT_I(sbi)`、`FREE_I(sbi)`、`DIRTY_I(sbi)`：从 `SM_I(sbi)` 继续取 segment 子结构。
- `META_MAPPING(sbi)`、`NODE_MAPPING(sbi)`：分别取内部 meta/node inode 的 address_space。

### f2fs_inode_info

`fs/f2fs/f2fs.h::struct f2fs_inode_info` 内嵌 `struct inode vfs_inode`，是 VFS inode 的外层私有对象。`F2FS_I(inode)` 通过 `container_of(inode, struct f2fs_inode_info, vfs_inode)` 找回私有字段。它保存：

- 磁盘/语义属性：`i_flags`、`i_advise`、`i_dir_level`、`i_current_depth` 或 `i_gc_failures`、`i_pino`、`i_xattr_nid`、project quota、inline xattr/data/dentry 状态。
- 并发与脏页状态：`i_sem`、`dirty_pages`、`dirty_list`、`gdirty_list`、`i_gc_rwsem`、`i_xattr_sem`。
- extent cache、atomic write/COW、compression 参数和压缩块计数等。

`fs/f2fs/super.c::f2fs_alloc_inode()` 从 `f2fs_inode_cachep` 分配该结构，初始化私有锁、链表、计数器；`f2fs_sops.alloc_inode` 指向它。

### f2fs_nm_info

`fs/f2fs/f2fs.h::struct f2fs_nm_info` 是 node manager 状态，由 `fs/f2fs/node.c::f2fs_build_node_manager()` 分配并挂到 `sbi->nm_info`。它管理：

- NAT 基址、NAT block 数量、最大 nid、可用 nid、next scan nid。
- NAT cache：`nat_root`、`nat_set_root`、`nat_entries`、`nat_tree_lock`。
- free nid cache：`free_nid_root`、`free_nid_list`、`nid_cnt`、`nid_list_lock`、`free_nid_bitmap`。
- checkpoint 相关 NAT bitmap、nat_bits/full/empty bitmaps。

inode、目录项和 node page 都以 nid 为核心连接；`f2fs_new_inode()` 分配 nid，`f2fs_iget()` 根据 ino/nid 读取 inode node folio。

### f2fs_sm_info

`fs/f2fs/f2fs.h::struct f2fs_sm_info` 是 segment manager 状态，由 `fs/f2fs/segment.c::f2fs_build_segment_manager()` 分配并挂到 `sbi->sm_info`。它管理：

- `sit_info`：segment usage table 相关状态。
- `free_info`：空闲 segment bitmap。
- `dirty_info`：dirty/pre-free segment 信息。
- `curseg_array`：当前活跃日志段，包括 hot/warm/cold data/node 等。
- 基址与规模：`seg0_blkaddr`、`main_blkaddr`、`ssa_blkaddr`、`segment_count`、`main_segments`、`reserved_segments`、`ovp_segments`。
- IPU/SSR 策略、flush/discard 控制器。

因此核心对象关系可以概括为：

```text
VFS super_block
  -> s_fs_info: f2fs_sb_info
       -> raw_super / ckpt
       -> meta_inode -> META_MAPPING
       -> node_inode -> NODE_MAPPING
       -> nm_info: f2fs_nm_info -> NAT / nid / node cache
       -> sm_info: f2fs_sm_info -> SIT / free / dirty / curseg / discard

VFS inode
  embedded in f2fs_inode_info
  -> inode->i_sb -> super_block -> f2fs_sb_info
```

## 4. VFS operations 和 address_space_operations 设置

### super_operations

`fs/f2fs/super.c::f2fs_sops` 设置为 `sb->s_op`，主要成员包括：

- `.alloc_inode = f2fs_alloc_inode`
- `.free_inode = f2fs_free_inode`
- `.drop_inode = f2fs_drop_inode`
- `.write_inode = f2fs_write_inode`
- `.dirty_inode = f2fs_dirty_inode`
- `.show_options = f2fs_show_options`
- quota 条件成员：`.quota_read`、`.quota_write`、`.get_dquots`
- `.evict_inode = f2fs_evict_inode`
- `.put_super = f2fs_put_super`
- `.sync_fs = f2fs_sync_fs`
- `.freeze_fs = f2fs_freeze`
- `.unfreeze_fs = f2fs_unfreeze`
- `.statfs = f2fs_statfs`
- `.shutdown = f2fs_shutdown`

### inode_operations

不同 inode 类型在新建或读取时设置不同 `i_op`：

- 普通文件：`fs/f2fs/file.c::f2fs_file_inode_operations`，提供 `getattr`、`setattr`、ACL、xattr、`fiemap`、fileattr get/set。
- 目录：`fs/f2fs/namei.c::f2fs_dir_inode_operations`，提供 `create`、`lookup`、`link`、`unlink`、`symlink`、`mkdir`、`rmdir`、`mknod`、`rename`、`tmpfile` 等。
- symlink：`f2fs_symlink_inode_operations` 或 `f2fs_encrypted_symlink_inode_operations`。
- 特殊文件：`f2fs_special_inode_operations`。

### file_operations

- 普通文件：`fs/f2fs/file.c::f2fs_file_operations`，包括 `llseek`、`read_iter`、`write_iter`、`open`、`release`、`mmap_prepare`、`flush`、`fsync`、`fallocate`、`ioctl`、`splice_read/write`、`fadvise`、`setlease` 等。
- 目录文件：`fs/f2fs/dir.c::f2fs_dir_operations`，包括 `llseek`、`generic_read_dir`、`iterate_shared = f2fs_readdir`、`fsync`、`ioctl`、`setlease`。

### address_space_operations

- 普通数据页：`fs/f2fs/data.c::f2fs_dblock_aops`，包括 `read_folio`、`readahead`、`writepages`、`write_begin`、`write_end`、`dirty_folio`、`invalidate_folio`、`release_folio`、`bmap`、swap activate/deactivate 等。
- node inode：`fs/f2fs/node.c::f2fs_node_aops`，主要是 node page writeback/dirty/invalidate/release/migrate。
- meta inode：`fs/f2fs/checkpoint.c::f2fs_meta_aops`，主要是 meta page writeback/dirty/invalidate/release/migrate。
- compression 内部 inode：`fs/f2fs/compress.c::f2fs_compress_aops`，主要用于压缩块缓存页的 release/invalidate/migrate。

ops 设置路径有两类：

- 挂载时内部 inode：`f2fs_fill_super()` 调用 `f2fs_iget()` 读取 `F2FS_META_INO` 和 `F2FS_NODE_INO`，`fs/f2fs/inode.c::f2fs_iget()` 根据 ino 设置 `f2fs_meta_aops` 或 `f2fs_node_aops`。
- 用户 inode：`f2fs_create()`、`f2fs_mkdir()`、`f2fs_symlink()`、`f2fs_mknod()`、`f2fs_tmpfile()` 在新建 inode 后立即设置对应 `i_op/i_fop/a_ops`；从磁盘读取已有 inode 时，`f2fs_iget()` 根据 `inode->i_mode` 再设置。

## 5. 目录、普通文件、inode 装配路径

### 已存在 inode 读取路径

典型 lookup 路径：

1. VFS 调目录 `i_op->lookup`，进入 `fs/f2fs/namei.c::f2fs_lookup()`。
2. `f2fs_prepare_lookup()` 处理文件名、加密/casefold 等前置状态。
3. `__f2fs_find_entry()` 在目录数据中查找 dentry，拿到 `struct f2fs_dir_entry`。
4. 从 `de->ino` 取 ino，调用 `f2fs_iget(dir->i_sb, ino)`。
5. `fs/f2fs/inode.c::f2fs_iget()` 先 `iget_locked()` 查 VFS inode cache；若是新 inode，则调用 `do_read_inode()`。
6. `do_read_inode()` 通过 `f2fs_get_inode_folio(sbi, inode->i_ino)` 从 node mapping 读取 inode node folio，`F2FS_INODE(node_folio)` 得到磁盘 inode，填充 VFS inode 和 `f2fs_inode_info` 私有字段，校验 inode、恢复 inline 状态、初始化 extent tree。
7. 回到 `f2fs_iget()`，根据 ino 或 `i_mode` 设置 node/meta/compress/regular/dir/symlink/special 的 ops，最后 `unlock_new_inode()`。
8. `f2fs_lookup()` 用 `d_splice_alias(inode, dentry)` 连接 dentry 和 inode。

### 普通文件创建路径

`fs/f2fs/namei.c::f2fs_create()`：

1. 检查 checkpoint 错误和空间状态，初始化父目录 quota。
2. 调 `f2fs_new_inode(idmap, dir, mode, dentry->d_name.name)`。
3. `f2fs_new_inode()` 调 `new_inode()` 分配 VFS inode/f2fs_inode_info，再用 `f2fs_alloc_nid()` 从 node manager 分配 nid，设置 owner、ino、时间、generation、project quota、加密、inline、压缩、冷热文件温度、extent tree 等。
4. `f2fs_create()` 设置 `inode->i_op = &f2fs_file_inode_operations`、`inode->i_fop = &f2fs_file_operations`、`inode->i_mapping->a_ops = &f2fs_dblock_aops`。
5. 加 `f2fs_lock_op()`，调用 `f2fs_add_link(dentry, inode)`。
6. `f2fs_add_link()` 是 `f2fs_do_add_link()` 的 inline 包装；后者设置文件名并最终进入 `f2fs_add_dentry()`。
7. `f2fs_add_dentry()` 优先尝试 inline dentry，必要时走 `f2fs_add_regular_entry()`。
8. `f2fs_add_regular_entry()` 找目录块空槽，调用 `f2fs_init_inode_metadata()`；新 inode 时它会 `f2fs_new_inode_folio()` 分配 inode node folio，初始化 ACL/security/encryption context，并写入 dentry/inode 元数据。
9. 成功后 `f2fs_alloc_nid_done()` 确认 nid，`d_instantiate_new(dentry, inode)` 实例化 dentry。

### 目录创建路径

`fs/f2fs/namei.c::f2fs_mkdir()` 和普通文件类似，但差异是：

- `f2fs_new_inode()` 的 mode 带 `S_IFDIR`，并把目录 `i_current_depth` 初始化为 1。
- 设置 `inode->i_op = &f2fs_dir_inode_operations`、`inode->i_fop = &f2fs_dir_operations`、`inode->i_mapping->a_ops = &f2fs_dblock_aops`，并对 mapping 设置 `GFP_NOFS`。
- 设置 `FI_INC_LINK`，在 `f2fs_init_inode_metadata()` 中若是目录，会调用 `make_empty_dir()` 创建 `.` 和 `..` 基本目录项。
- `f2fs_update_parent_metadata()` 会更新父目录链接计数、mtime/ctime、目录深度等。

### 特殊 inode/符号链接/tmpfile

- `f2fs_symlink()` 新建 inode 后根据是否加密设置 symlink inode ops，设置 `f2fs_dblock_aops`，再走 `f2fs_add_link()`。
- `f2fs_mknod()` 用 `init_special_inode()` 装配设备/FIFO/socket inode，并设置 `f2fs_special_inode_operations`。
- `f2fs_tmpfile()` 通过 `__f2fs_tmpfile()` 新建 inode，普通 tmpfile 设置 file inode/file ops/aops；后续 `f2fs_do_tmpfile()` 初始化 inode metadata，但不通过普通 dentry 名字暴露。

## 6. 架构学习难点和后续建议

学习难点：

- `f2fs_fill_super()` 同时处理挂载选项、磁盘 super/checkpoint、内部 inode、segment/node manager、recovery、quota、compression、sysfs，失败回滚标签很多，需要画阶段图才能不迷路。
- `f2fs_inode_info`、磁盘 `struct f2fs_inode`、node folio、VFS `struct inode` 之间不是一一文件这么简单：F2FS 把 inode 本身也作为 node 管理，读 inode 要经过 node cache/NAT。
- NAT/SIT/SSA/curseg/checkpoint 的关系跨 `node.c`、`segment.c`、`checkpoint.c`，仅看本任务列出的五个 `.c` 文件会缺少部分因果。
- ops 设置分散在新建路径和读取路径：新建时 `namei.c` 主动设置；已存在 inode 从 `inode.c::f2fs_iget()` 根据 mode 设置；内部 meta/node inode 又按特殊 ino 设置。
- 可选特性较多，`CONFIG_*` 和 mount option 组合会改变挂载行为和 ops 字段，例如 quota、fscrypt、verity、compression、zoned device、casefold。

后续建议：

- 画一张从 `mount -t f2fs` 到 `sb->s_root` 的时序图，单独标注失败回滚路径。
- 对 `f2fs_iget()` 做专题：比较 `F2FS_META_INO`、`F2FS_NODE_INO`、`F2FS_ROOT_INO`、普通文件、目录、symlink 的装配差异。
- 继续阅读 `fs/f2fs/node.c` 的 NAT/free nid、`fs/f2fs/segment.c` 的 SIT/free/dirty/curseg 和 `fs/f2fs/checkpoint.c` 的 checkpoint pack，补齐“写入后如何落盘并被 checkpoint 固化”的链路。
- 做一个 VFS ops 对照表：VFS 回调 -> F2FS 函数 -> 主要内部模块，这对理解 namei/file/data 的职责边界很有效。
- 选一个具体 syscall 走读，例如 `open(O_CREAT)`、`mkdir`、`read`、`write`、`fsync`，分别串起 namei/inode/data/node/segment/checkpoint。

## 7. 不确定点

- 本报告只基于静态源码阅读，没有实际格式化并挂载一个 F2FS 镜像验证调用路径。
- recovery、quota、compression、fscrypt、verity、zoned block device 的条件路径很多；报告只列出主干和已读到的设置点，没有穷举所有配置组合。
- `Documentation/filesystems/f2fs.rst` 在本地 PowerShell 输出中部分标点/引号显示为编码异常；我没有直接复制这些异常文本，只依据其段落含义总结设计目标。
- `fs/f2fs/segment.c`、`fs/f2fs/node.c`、`fs/f2fs/data.c`、`fs/f2fs/checkpoint.c` 虽为理解结构体关系和 aops 必须少量阅读，但不是本任务的重点文件；更深入的数据分配、GC victim 选择、checkpoint pack 格式仍需后续专题确认。
- 具体函数行号会随内核版本变化；报告引用以当前工作区的源码文件名和函数名为准。
