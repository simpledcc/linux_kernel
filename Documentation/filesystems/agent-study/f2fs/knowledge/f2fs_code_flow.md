# F2FS 关键代码流程

## Mount 流程

```text
register_filesystem(&f2fs_fs_type)
  -> VFS mount
  -> f2fs_init_fs_context()
  -> f2fs_get_tree()
  -> get_tree_bdev(..., f2fs_fill_super)
  -> f2fs_fill_super()
      -> read_raw_super_block()
      -> f2fs_get_valid_checkpoint()
      -> f2fs_build_segment_manager()
      -> f2fs_build_node_manager()
      -> f2fs_build_gc_manager()
      -> f2fs_iget(F2FS_ROOT_INO)
      -> d_make_root(root)
```

关键点：`f2fs_fill_super()` 是挂载主流程的中心，里面同时初始化 VFS superblock、F2FS 私有状态、checkpoint、segment/node manager、GC 和 root inode。

## 普通文件创建流程

```text
VFS create
  -> f2fs_create()
      -> f2fs_new_inode()
      -> inode->i_op = f2fs_file_inode_operations
      -> inode->i_fop = f2fs_file_operations
      -> inode->i_mapping->a_ops = f2fs_dblock_aops
      -> f2fs_add_link()
      -> f2fs_init_inode_metadata()
      -> d_instantiate_new()
```

已存在文件的读取则通常经过 `f2fs_lookup()` -> `f2fs_iget()`，由 `f2fs_iget()` 根据 inode 类型重新装配 operations。

## Read 流程

```text
VFS read_iter
  -> f2fs_file_read_iter()
      -> direct I/O: f2fs_dio_read_iter()
      -> buffered read: filemap_read()
          -> page cache miss
          -> f2fs_read_data_folio()
          -> f2fs_mpage_readpages()
```

关键点：buffered read 的数据面在 page cache 中；F2FS 通过 `f2fs_dblock_aops.read_folio` 和 `readahead` 参与缺页读和预读。

## Write 流程

```text
VFS write_iter
  -> f2fs_file_write_iter()
      -> f2fs_write_checks()
      -> f2fs_preallocate_blocks()
      -> direct I/O: f2fs_dio_write_iter()
      -> buffered write: f2fs_buffered_write_iter()
          -> generic_perform_write()
              -> f2fs_write_begin()
              -> copy to page cache
              -> f2fs_write_end()
      -> generic_write_sync()
```

关键点：buffered write 先写 page cache，不等于马上落盘。真正的数据写出通常发生在 writeback、fsync 或 checkpoint 相关路径中。

## Writeback 流程

```text
dirty folio
  -> f2fs_dirty_data_folio()
  -> writeback
  -> f2fs_write_data_pages()
  -> f2fs_write_cache_pages()
  -> f2fs_write_single_data_page()
  -> f2fs_do_write_data_page()
      -> f2fs_inplace_write_data() 或 f2fs_outplace_write_data()
      -> f2fs_allocate_data_block()
```

关键点：F2FS 的 out-of-place 更新会分配新物理块、更新 SIT，并通过 node/NAT/checkpoint 机制把逻辑地址和恢复路径串起来。

## Fsync 与 Checkpoint

```text
VFS fsync
  -> f2fs_sync_file()
  -> f2fs_do_sync_file()
      -> file_write_and_wait_range()
      -> need_do_checkpoint()
      -> yes: f2fs_sync_fs() -> f2fs_write_checkpoint()
      -> no: f2fs_fsync_node_pages()
```

checkpoint 主流程：

```text
f2fs_write_checkpoint()
  -> block_operations()
  -> f2fs_flush_nat_entries()
  -> f2fs_flush_sit_entries()
  -> do_checkpoint()
  -> commit_checkpoint()
```

关键点：fsync 先确保目标范围数据写回，然后决定是走完整 checkpoint，还是写 fsync node 支持 roll-forward recovery。

## GC 流程

```text
gc_thread_func() 或前台空间压力
  -> f2fs_gc()
      -> __get_victim()
      -> do_garbage_collect()
          -> gc_node_segment()
          -> gc_data_segment()
              -> is_alive()
              -> move_data_page() 或 move_data_block()
      -> 必要时 f2fs_write_checkpoint()
```

关键点：GC 不只是搬块。它必须通过 SSA 反查 owner，通过 NAT/node page 验证 victim block 仍然有效，并在迁移后让 SIT/NAT/checkpoint 关系保持一致。
