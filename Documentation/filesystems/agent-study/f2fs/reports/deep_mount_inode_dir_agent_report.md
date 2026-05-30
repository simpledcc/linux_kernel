# F2FS 深度学习报告：mount / inode / directory / namei

本报告面向 `fs/f2fs/super.c`、`fs/f2fs/inode.c`、`fs/f2fs/namei.c`、`fs/f2fs/dir.c`、`fs/f2fs/f2fs.h` 中与挂载、superblock、inode、目录项和 VFS namei 路径相关的代码。分析只基于源码阅读，未修改 F2FS 源码。

## 1. 关键结构关系

### 1.1 VFS super_block 与 F2FS 私有 super 信息

核心关系：

```text
struct super_block
  -> s_fs_info
       -> struct f2fs_sb_info
            -> sb                 回指 VFS super_block
            -> raw_super          磁盘 super block 的内存副本
            -> ckpt               checkpoint 内存副本
            -> meta_inode         元数据地址空间 inode
            -> node_inode         node 地址空间 inode
            -> nm_info            node manager
            -> sm_info            segment manager
            -> mount_opt          mount options
            -> root_ino_num/node_ino_num/meta_ino_num
```

`struct f2fs_sb_info` 是 F2FS 运行时的中心对象。`super.c::f2fs_fill_super()` 分配它并挂到 `sb->s_fs_info`，后续大多数代码通过 `F2FS_SB(sb)`、`F2FS_I_SB(inode)` 取回它。它同时连接 VFS、磁盘 superblock、checkpoint、NAT/SIT/segment/node 管理器、内部 inode、锁和计数器。

`raw_super` 是磁盘 `struct f2fs_super_block` 的拷贝；`init_sb_info()` 会把其中的块大小、segment/section 配置、root/node/meta inode 编号等字段转换到 `f2fs_sb_info`。因此学习挂载路径时要区分：

- 磁盘格式字段：`raw_super->root_ino`、`raw_super->node_ino`、`raw_super->meta_ino` 等。
- 运行时字段：`sbi->root_ino_num`、`sbi->node_ino_num`、`sbi->meta_ino_num` 等。

### 1.2 VFS inode 与 F2FS inode 私有信息

核心关系：

```text
struct inode
  嵌入在
struct f2fs_inode_info
  -> vfs_inode
  -> i_flags / flags[]        F2FS inode 标志
  -> i_current_depth          目录 hash 层级深度
  -> i_pino                   记录父 inode，用于 fsck / recovery 语义
  -> chash / clevel / task    lookup-create 优化与一致性辅助
  -> i_sem / i_xattr_sem      F2FS 私有同步
  -> extent_tree[]            extent cache
```

`F2FS_I(inode)` 使用 `container_of()` 从 VFS inode 得到 `struct f2fs_inode_info`。F2FS 的目录深度、父 inode、inline xattr/data/dentry、压缩、project quota、extent cache 等状态都在这里维护。

`inode.c::f2fs_iget()` 是从 inode number 获得 VFS inode 的关键函数。它走 `iget_locked()`，如果 inode 已在缓存中且不是内部 meta inode，则直接返回；如果是新 inode，则读磁盘 node page、填充 VFS inode 和 F2FS 私有字段，最后按文件类型安装对应的 operation table：

- regular file：`f2fs_file_inode_operations`、`f2fs_file_operations`、`f2fs_dblock_aops`
- directory：`f2fs_dir_inode_operations`、`f2fs_dir_operations`、`f2fs_dblock_aops`
- symlink：普通或加密 symlink inode operations
- special file：`f2fs_special_inode_operations`
- meta/node/compress 内部 inode：使用专门的 address_space operations

### 1.3 目录项、目录页与 namei

核心关系：

```text
VFS dentry name
  -> f2fs_filename
       -> usr_fname / disk_name / hash / cf_name
  -> f2fs_dir_entry
       -> hash_code / name_len / ino / file_type
  -> bitmap slots + filename array
       -> struct f2fs_dentry_block 或 inline dentry 区域
```

F2FS 目录不是简单线性数组。普通目录项按 hash level / bucket / block 查找：

```text
__f2fs_find_entry()
  -> inline dentry: f2fs_find_in_inline_dir()
  -> regular dentry:
       for level in [0, i_current_depth)
         find_in_level()
           -> 根据 fname->hash 选 bucket
           -> f2fs_find_data_folio()
           -> find_in_block()
           -> f2fs_find_target_dentry()
```

`f2fs_dir_entry` 只保存 `ino`、类型、hash 和名字长度；实际名字存放在 dentry block 的 filename 区域，bitmap 表示 slot 占用情况。长文件名可能占用多个 slot，`GET_DENTRY_SLOTS()` 和 `f2fs_room_for_filename()` 是理解新增目录项的关键。

## 2. mount / fill_super 到 root inode 的流程

入口：

```text
mount syscall / fs_context
  -> f2fs_get_tree()
  -> get_tree_bdev(fc, f2fs_fill_super)
  -> f2fs_fill_super()
```

`f2fs_fill_super()` 主流程可以拆成以下阶段。

### 2.1 分配并初始化基础 super 运行时对象

1. 分配 `struct f2fs_sb_info`。
2. `sbi->sb = sb`，初始化 checkpoint、GC、node、quota、flush、inode list 等锁和链表。
3. 设置 VFS block size 为 `F2FS_BLKSIZE`。
4. `read_raw_super_block()` 读取两个 superblock 副本，逐个做 `sanity_check_raw_super()`，选择有效 superblock，把磁盘 superblock 拷贝到 `sbi->raw_super`。
5. `sb->s_fs_info = sbi`。

这里的关键点是：只有读到有效 raw super 后，`sb->s_fs_info` 才正式指向 `sbi`。后续宏 `F2FS_SB(sb)` 才能工作。

### 2.2 设置 VFS super 操作表与 mount options

1. `default_options()` 建立默认挂载选项。
2. `f2fs_check_opt_consistency()` 和 `f2fs_apply_options()` 处理 fs_context 中的挂载参数。
3. `f2fs_sanity_check_options()` 校验组合是否合法。
4. 设置 `sb->s_op = &f2fs_sops`，以及 encryption、verity、xattr、export、magic、ACL、lazytime、inlinecrypt 等 VFS super 属性。
5. `init_sb_info()` 根据 raw super 填充运行时参数，包括 `F2FS_ROOT_INO(sbi)`、`F2FS_NODE_INO(sbi)`、`F2FS_META_INO(sbi)`。

### 2.3 建立内部 inode、checkpoint 与管理器

顺序很重要：

1. `sbi->meta_inode = f2fs_iget(sb, F2FS_META_INO(sbi))`
2. `f2fs_get_valid_checkpoint(sbi)`
3. `f2fs_scan_devices(sbi)`
4. 初始化 post-read workqueue、计数器、extent cache、ino entry、fsync node 信息、checkpoint request control。
5. `f2fs_build_segment_manager(sbi)`
6. `f2fs_build_node_manager(sbi)`
7. `f2fs_build_gc_manager(sbi)` 和 stats。
8. `sbi->node_inode = f2fs_iget(sb, F2FS_NODE_INO(sbi))`

`meta_inode` 早于 checkpoint 读取，因为元数据地址空间需要先可用；`node_inode` 在 node manager 建立后再读取。二者都是 F2FS 内部 inode，不代表用户目录树里的普通文件。

### 2.4 读取 root inode 并构造根 dentry

root 阶段代码集中在 `f2fs_fill_super()` 中：

```text
root = f2fs_iget(sb, F2FS_ROOT_INO(sbi))
  -> iget_locked()
  -> do_read_inode()
  -> f2fs_set_inode_flags()
  -> S_ISDIR(root->i_mode) 分支安装 f2fs_dir_inode_operations

校验 root:
  S_ISDIR(root->i_mode)
  root->i_blocks != 0
  root->i_size != 0
  root->i_nlink != 0

generic_set_sb_d_ops(sb)
sb->s_root = d_make_root(root)
```

`f2fs_iget()` 读出的 root inode 必须是目录，并且 block、size、link count 都不能为 0。通过校验后，`d_make_root(root)` 将 root inode 包装成 VFS 根 dentry，挂载点才真正有了可遍历的目录树入口。

### 2.5 root 之后的恢复和后台机制

root dentry 建立后，挂载还会继续做 quota、orphan inode recovery、roll-forward fsync data recovery、checkpoint reset、inmem curseg、GC thread、损坏 superblock 修复等。不要误以为 `sb->s_root` 设置完成就等价于整个 mount 成功；后续任一恢复或初始化失败仍可能走错误释放路径。

## 3. lookup / create / link / unlink / rename 的主要调用链

### 3.1 VFS operation table

目录 inode 在 `f2fs_iget()` 中安装：

```text
inode->i_op = &f2fs_dir_inode_operations
inode->i_fop = &f2fs_dir_operations
inode->i_mapping->a_ops = &f2fs_dblock_aops
```

`f2fs_dir_inode_operations` 中相关入口：

```text
.create = f2fs_create
.lookup = f2fs_lookup
.link   = f2fs_link
.unlink = f2fs_unlink
.rename = f2fs_rename2
```

### 3.2 lookup 调用链

```text
VFS lookup
  -> f2fs_lookup(dir, dentry, flags)
       -> 检查 name 长度
       -> f2fs_prepare_lookup(dir, dentry, &fname)
       -> __f2fs_find_entry(dir, &fname, &folio)
            -> inline: f2fs_find_in_inline_dir()
            -> regular:
                 for each level:
                   find_in_level()
                     -> f2fs_find_data_folio()
                     -> find_in_block()
                     -> f2fs_find_target_dentry()
                          -> hash_code 初筛
                          -> f2fs_match_name()
       -> de->ino
       -> f2fs_iget(dir->i_sb, ino)
       -> 校验 i_nlink 和加密上下文
       -> d_splice_alias(inode, dentry)
```

关键语义：

- `f2fs_prepare_lookup()` / `f2fs_setup_filename()` 负责加密、casefold、hash 等文件名准备。
- `__f2fs_find_entry()` 查找失败时会设置 `F2FS_I(dir)->task = current`，用于 create 路径减少重复查找，但 create 仍可能在必要时重新验证。
- casefold 目录在某些 negative dentry 场景下不会缓存负 dentry。

### 3.3 create 调用链

```text
VFS create
  -> f2fs_create(idmap, dir, dentry, mode, excl)
       -> f2fs_cp_error() / f2fs_is_checkpoint_ready()
       -> f2fs_dquot_initialize(dir)
       -> f2fs_new_inode(idmap, dir, mode, name)
            -> new_inode()
            -> f2fs_alloc_nid()
            -> inode_init_owner()
            -> insert_inode_locked()
            -> fscrypt_prepare_new_inode()
            -> quota 初始化
            -> 设置 FI_NEW_INODE、inline/compress/project/temperature 等状态
            -> f2fs_set_inode_flags()
            -> f2fs_init_extent_tree()
       -> 给普通文件安装 i_op/i_fop/a_ops
       -> f2fs_lock_op()
       -> f2fs_add_link(dentry, inode)
            -> f2fs_do_add_link(parent, name, inode, ino, mode)
                 -> f2fs_setup_filename()
                 -> 可能 __f2fs_find_entry() 重查防重
                 -> f2fs_add_dentry()
                      -> inline: f2fs_add_inline_entry()
                      -> regular: f2fs_add_regular_entry()
                           -> f2fs_get_new_data_folio()
                           -> f2fs_init_inode_metadata()
                                -> f2fs_new_inode_folio()
                                -> 目录则 make_empty_dir()
                                -> ACL/security/encryption context
                                -> init_dent_inode()
                           -> f2fs_update_dentry()
                           -> f2fs_i_pino_write()
                           -> f2fs_update_inode()
                           -> f2fs_update_parent_metadata()
       -> f2fs_unlock_op()
       -> f2fs_alloc_nid_done()
       -> d_instantiate_new()
       -> dirsync 时 f2fs_sync_fs()
       -> f2fs_balance_fs()
```

创建路径的核心不是单纯“分配 inode”，而是同时完成：

- NID 分配和 inode cache 插入。
- 新 inode node page 初始化。
- 目录项写入。
- 父目录 mtime/ctime、目录深度、link count 更新。
- VFS dentry 与 inode 绑定。

### 3.4 link 调用链

```text
VFS link
  -> f2fs_link(old_dentry, dir, dentry)
       -> checkpoint 状态检查
       -> fscrypt_prepare_link()
       -> project quota 继承检查
       -> f2fs_dquot_initialize(dir)
       -> f2fs_balance_fs()
       -> inode_set_ctime_current(old_inode)
       -> ihold(old_inode)
       -> set FI_INC_LINK
       -> f2fs_lock_op()
       -> f2fs_add_link(dentry, old_inode)
            -> f2fs_do_add_link()
            -> f2fs_add_dentry()
            -> f2fs_init_inode_metadata()
                 -> 已有 inode 走 f2fs_get_inode_folio()
                 -> FI_INC_LINK 时 f2fs_i_links_write(inode, true)
       -> f2fs_unlock_op()
       -> d_instantiate()
       -> dirsync 时 f2fs_sync_fs()
```

`FI_INC_LINK` 是 link 路径的重要状态，它让 `f2fs_init_inode_metadata()` 在已有 inode 的 inode page 上更新 link count，并处理 tmpfile linkat 到可见名字时的 orphan list 清理。

### 3.5 unlink 调用链

```text
VFS unlink
  -> f2fs_unlink(dir, dentry)
       -> f2fs_cp_error()
       -> f2fs_dquot_initialize(dir)
       -> f2fs_dquot_initialize(inode)
       -> f2fs_find_entry(dir, name, &folio)
            -> f2fs_setup_filename()
            -> __f2fs_find_entry()
       -> 校验 inode->i_nlink
       -> f2fs_balance_fs()
       -> f2fs_lock_op()
       -> f2fs_acquire_orphan_inode()
       -> f2fs_delete_entry(de, folio, dir, inode)
            -> strict fsync 模式记录 TRANS_DIR_INO
            -> inline: f2fs_delete_inline_entry()
            -> regular:
                 清 dentry bitmap slots
                 空 dentry page 可 f2fs_truncate_hole()
                 更新父目录时间并 mark dirty
                 f2fs_drop_nlink(dir, inode)
                      -> 目录还会减少父目录 nlink
                      -> 减 inode nlink，目录再额外减一次并 size=0
                      -> nlink 到 0 则 f2fs_add_orphan_inode()
       -> f2fs_unlock_op()
       -> casefold 目录可能 d_invalidate()
       -> dirsync 时 f2fs_sync_fs()
```

`unlink` 并不一定立刻释放 inode 数据。它先删除目录项并减少 link count；如果 link count 到 0，会加入 orphan inode，真正清理由后续 truncate / eviction / checkpoint recovery 语义保证。

### 3.6 rename 调用链

入口：

```text
VFS rename
  -> f2fs_rename2(idmap, old_dir, old_dentry, new_dir, new_dentry, flags)
       -> flags 校验
       -> fscrypt_prepare_rename()
       -> RENAME_EXCHANGE ? f2fs_cross_rename() : f2fs_rename()
```

普通 rename：

```text
f2fs_rename()
  -> checkpoint/project quota/whiteout/quota 检查
  -> old_entry = f2fs_find_entry(old_dir, old_name)
  -> 如果移动目录且跨父目录:
       old_dir_entry = f2fs_parent_dir(old_inode)   // 找 old_inode 的 ".."
  -> 如果 new_inode 存在:
       若 old 是目录，要求 new_inode 为空目录
       new_entry = f2fs_find_entry(new_dir, new_name)
       f2fs_lock_op()
       f2fs_acquire_orphan_inode()
       f2fs_set_link(new_dir, new_entry, new_folio, old_inode)
       降低被覆盖 new_inode 的 nlink，必要时加入 orphan
     否则:
       f2fs_lock_op()
       f2fs_add_link(new_dentry, old_inode)
       若 old 是目录，增加 new_dir nlink
  -> 更新 old_inode 的 pino 或 lost_pino
  -> f2fs_delete_entry(old_entry, old_folio, old_dir, NULL)
  -> whiteout 场景在 old_dentry 位置补 whiteout
  -> 如果跨目录移动目录，f2fs_set_link(old_inode, old_dir_entry, ..., new_dir) 更新 ".."
  -> 若 old 是目录，减少 old_dir nlink
  -> strict fsync 模式记录 TRANS_DIR_INO
  -> f2fs_unlock_op()
  -> dirsync 时 f2fs_sync_fs()
```

`RENAME_EXCHANGE`：

```text
f2fs_cross_rename()
  -> 查 old_entry/new_entry
  -> 跨父目录且对象为目录时，分别查各自 ".."
  -> 检查父目录 link count 上限
  -> f2fs_lock_op()
  -> 必要时交换/更新两个目录的 ".."
  -> f2fs_set_link(old_dir, old_entry, ..., new_inode)
  -> 更新 old_inode pino/lost_pino 和 old_dir nlink
  -> f2fs_set_link(new_dir, new_entry, ..., old_inode)
  -> 更新 new_inode pino/lost_pino 和 new_dir nlink
  -> strict fsync 记录两个目录
  -> f2fs_unlock_op()
```

rename 的难点在于它同时修改多个对象：旧父目录项、新父目录项、被移动 inode、被覆盖 inode、目录对象的 `..`、父目录 nlink、orphan inode 和 strict fsync 追踪。

## 4. 推荐补充中文学习注释的精确函数/位置与理由

以下建议均为“学习注释”位置，不建议修改逻辑。注释应尽量短，避免把调用链全文塞进源码。

1. `fs/f2fs/super.c::f2fs_fill_super()`，`read_raw_super_block()` 成功后、`sb->s_fs_info = sbi` 附近。
   - 理由：这是 VFS `super_block` 与 F2FS 私有 `f2fs_sb_info` 正式绑定的位置，后续 `F2FS_SB()` 宏依赖它。

2. `fs/f2fs/super.c::f2fs_fill_super()`，`init_sb_info(sbi)` 调用附近。
   - 理由：`init_sb_info()` 把 raw super 中的 root/node/meta inode 编号和文件系统几何参数转成运行时字段，是理解 mount 后续流程的分水岭。

3. `fs/f2fs/super.c::f2fs_fill_super()`，`sbi->meta_inode = f2fs_iget(...)` 与 `sbi->node_inode = f2fs_iget(...)` 附近。
   - 理由：这两个 inode 是内部地址空间载体，不是用户可见文件。初学者容易把它们与 root inode 混淆。

4. `fs/f2fs/super.c::f2fs_fill_super()`，`root = f2fs_iget(sb, F2FS_ROOT_INO(sbi))` 到 `d_make_root(root)` 附近。
   - 理由：这是从磁盘 root inode 进入 VFS dentry 树的关键桥接点，建议说明校验条件与 `sb->s_root` 的意义。

5. `fs/f2fs/inode.c::f2fs_iget()`，`iget_locked()` 后的 `I_NEW` 判断附近。
   - 理由：说明 inode cache 命中与首次从磁盘读取的分叉；尤其 meta inode 在缓存命中时被拒绝访问的特殊处理。

6. `fs/f2fs/inode.c::f2fs_iget()`，按 `S_ISREG/S_ISDIR/S_ISLNK/...` 分派 operation table 的分支附近。
   - 理由：这是 VFS 后续 file/namei/readdir 调用如何落到 F2FS 实现的入口表。

7. `fs/f2fs/namei.c::f2fs_lookup()`，`f2fs_prepare_lookup()` 到 `__f2fs_find_entry()` 附近。
   - 理由：说明 lookup 不直接比较 `dentry->d_name`，而先转换为处理加密、casefold、hash 后的 `f2fs_filename`。

8. `fs/f2fs/dir.c::__f2fs_find_entry()`，inline dentry 分支与 level 循环附近。
   - 理由：这是理解 F2FS 目录 hash、多层 bucket 和 inline dentry 双路径的最佳位置。

9. `fs/f2fs/dir.c::find_in_level()`，`bucket_no = hash % nbucket` 和 `room/chash/clevel` 逻辑附近。
   - 理由：说明查找失败也会记录可用空间提示，服务后续 create 加速。

10. `fs/f2fs/dir.c::f2fs_do_add_link()`，`current != F2FS_I(dir)->task` 的重查逻辑附近。
    - 理由：这里解释 lookup-create 竞态防御最合适：同一 task 的 lookup miss 可以优化，不同上下文仍需重新查磁盘目录项。

11. `fs/f2fs/dir.c::f2fs_add_regular_entry()`，`f2fs_init_inode_metadata()` 与 `f2fs_update_dentry()` 之间。
    - 理由：新增名字不是只写目录项，还可能先初始化 inode node page、目录的 `.`/`..`、ACL/security/encryption context。

12. `fs/f2fs/dir.c::f2fs_delete_entry()`，清 bitmap 和 `f2fs_drop_nlink()` 附近。
    - 理由：说明删除目录项与减少 inode link count 是两个相邻但不同的动作，空目录项页还可能被 punch hole。

13. `fs/f2fs/namei.c::f2fs_rename()`，`new_inode` 存在和不存在两个大分支开始处。
    - 理由：rename 覆盖目标与 rename 到空目标的行为差异很大，覆盖路径需要 orphan 处理和替换已有 dentry。

14. `fs/f2fs/namei.c::f2fs_cross_rename()`，更新两个对象 `..` 与两个父目录 nlink 的位置。
    - 理由：`RENAME_EXCHANGE` 是 rename 中最容易读乱的路径，注释应强调“交换目录项指向”和“维护目录父子关系”是两件事。

15. `fs/f2fs/f2fs.h::struct f2fs_inode_info`，`chash/clevel/task` 字段附近。
    - 理由：这三个字段很不起眼，但直接参与目录 create 的性能优化和一致性兜底，适合补一句说明。

## 5. 容易误解点

1. `meta_inode`、`node_inode` 不是普通文件。
   - 它们是 F2FS 内部地址空间载体，用来缓存 meta block 和 node block。root inode 才是用户目录树入口。

2. `f2fs_iget()` 不只是“读 inode”。
   - 它还负责从 inode cache 复用对象、读取磁盘 node page、做 sanity check、设置 F2FS inode flags、安装 VFS operation table。

3. root dentry 建立不代表 mount 已经完全成功。
   - `d_make_root()` 后还有 quota、orphan recovery、fsync data recovery、checkpoint/GC 等步骤，后续失败仍会回滚释放。

4. lookup 失败不一定只是返回 negative dentry。
   - casefold/Unicode 场景下，F2FS 会避免缓存某些 negative dentry；另外 lookup miss 会记录 `F2FS_I(dir)->task`，影响 create 的重查策略。

5. 目录查找不是简单全目录线性扫描。
   - 普通目录按 hash level、bucket、block 查找；只有 casefold fallback 等场景可能走非 hash 匹配。inline dentry 又是独立路径。

6. `f2fs_add_link()` 需要外层持有 `f2fs_lock_op()`。
   - `dir.c` 注释明确要求调用者持有并释放该锁。`create/link/rename` 都在关键修改区间加锁，不能只看 `f2fs_add_link()` 本身。

7. `create` 的目录项写入与 inode 初始化是交织的。
   - `f2fs_add_regular_entry()` 在写 dentry 前会通过 `f2fs_init_inode_metadata()` 初始化新 inode 的 node page；目录 inode 还要创建 `.` 和 `..`。

8. `unlink` 删除的是名字，inode 生命周期由 link count 和 orphan 机制继续管理。
   - `f2fs_delete_entry()` 清目录项 bitmap 后调用 `f2fs_drop_nlink()`；nlink 到 0 时加入 orphan inode，保证崩溃恢复时能继续清理。

9. directory link count 会被特殊处理。
   - 删除目录时 inode 自身 link count 可能减两次；跨目录 rename 目录时还要更新旧父/新父目录 nlink 和被移动目录的 `..`。

10. rename 覆盖目标与 rename 到不存在目标不是同一路径。
    - 覆盖目标时 `f2fs_set_link(new_dir, new_entry, old_inode)` 替换已有目录项并减少 `new_inode` link count；不存在目标时先 `f2fs_add_link(new_dentry, old_inode)` 新增目录项。

11. `i_pino` 不是 VFS 的权威父指针。
    - 它主要服务 F2FS recovery/fsck 语义。rename 时普通文件可能 `file_lost_pino()`，目录则尽量更新 `i_pino` 和 `..` 以通过一致性检查。

12. `f2fs_dir_entry` 本身不保存完整名字数组。
    - 名字在 dentry block 的 filename 区域，bitmap 控制 slot 占用。阅读 `f2fs_update_dentry()` 时要同时看 `d->dentry`、`d->filename` 和 `d->bitmap`。

13. 加密和 casefold 影响 namei 的每一步。
    - lookup、create、link、rename 都有 fscrypt 准备或上下文校验；目录查找比较也可能走 `generic_ci_match()` 或 `fscrypt_match_name()`。

14. checkpoint 错误会让很多 namei 操作直接失败。
    - `create/link/unlink/rename` 都会检查 `f2fs_cp_error()` 或 checkpoint ready 状态，这是 F2FS 元数据一致性保护的一部分。

15. `f2fs_balance_fs()` 不只是后台细节。
    - create/link/unlink/rename 等前台操作会主动触发平衡，防止空间、segment、GC 压力无限后移。
