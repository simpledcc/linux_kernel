# 面向文件系统开发的 Agent 协作流程

## 推荐工作模式

```text
Manager Agent
  -> 拆任务、限定文件范围、写验收标准
  -> 启动 VFS/Ext4/Page Cache/Testing/Review 等子 agent
  -> 收集报告和补丁
  -> 合并知识库或代码修改
  -> 触发 Review/Test
  -> 失败则派回对应 agent 返工
```

文件系统开发不适合让多个 agent 同时修改同一批源码。更稳的模式是：

- 研究 agent 只写自己的报告。
- 开发 agent 只改被分配的文件。
- Review agent 默认只读，不直接修代码。
- Test agent 输出命令和测试报告。
- Manager 负责最终整合、提交和记录。

## 任务模板

```yaml
id: fs-task-001
title: "研究 ext4 buffered write 路径"
owner: ext4_agent
allowed_write_paths:
  - Documentation/filesystems/agent-study/reports/ext4_agent_report.md
inputs:
  - fs/ext4/file.c
  - fs/ext4/inode.c
  - mm/filemap.c
acceptance:
  - "必须引用源码文件和函数名"
  - "必须区分 buffered I/O 与 direct I/O"
  - "必须列出不确定点"
```

## 质量门禁

对文档和学习注释 patch，推荐最小检查：

```sh
git diff --check
git diff --stat
git diff -- Documentation/filesystems/agent-study fs/open.c fs/read_write.c fs/ext4/file.c mm/filemap.c
```

如果是完整 Linux 构建环境，可进一步运行：

```sh
scripts/checkpatch.pl --strict --git HEAD
make O=build olddefconfig
make O=build fs/ext4/ W=1
tools/testing/kunit/kunit.py run --kunitconfig=fs/ext4/.kunitconfig
```

在当前 Windows/PowerShell 环境和 sparse checkout 中，不能声称这些 Linux 构建与 KUnit 命令已经通过。它们是后续 Linux VM 或完整源码树中的验证路线。

## 本轮注释策略

本轮只添加少量中文学习注释，目标是帮助阅读关键分派点：

- `fs/open.c::do_dentry_open()`：说明 `inode->i_fop` 如何绑定到 `file->f_op`。
- `fs/read_write.c::vfs_read()`：说明 VFS read 如何分派到具体文件系统。
- `fs/read_write.c::vfs_write()`：说明 VFS write 如何进入 `write_iter()` 并受 freeze 保护。
- `fs/ext4/file.c::ext4_file_write_iter()`：说明 ext4 写入口如何分流 DAX/DIO/buffered。
- `mm/filemap.c::generic_file_read_iter()`：说明通用 read 最终进入 page cache。

这些注释用于个人学习分支，不建议直接作为上游内核提交风格。
