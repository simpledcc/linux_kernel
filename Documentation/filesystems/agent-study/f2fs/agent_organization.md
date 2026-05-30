# F2FS 多 Agent 学习组织架构

## 目标

本轮多 agent 协作专注学习 Linux 内核 `fs/f2fs/` 文件系统代码，输出可追溯的中文学习文档，并在少量关键入口添加中文学习注释。

本轮目标包括：

1. 梳理 f2fs 代码架构、核心数据结构和模块边界。
2. 梳理 mount/open/read/write/fsync/checkpoint/GC 等关键流程。
3. 输出适合后续继续学习的阅读路线和方法。
4. 记录 agent 的分工、学习过程、检验过程和配合方式。
5. 添加少量中文学习注释，不改变内核逻辑。

## Agent 角色

| Agent | 职责 | 重点文件 | 输出 |
|---|---|---|---|
| Manager Agent | 拆分任务、整合文档、添加注释、提交推送 | 全部 f2fs 报告和源码 | `knowledge/*.md`、提交记录 |
| F2FS Architecture Agent | 研究整体架构、挂载、VFS 接入和核心结构体 | `super.c`、`f2fs.h`、`inode.c`、`namei.c`、`file.c` | `reports/f2fs_architecture_agent_report.md` |
| F2FS IO Path Agent | 研究 read/write/fsync、page cache、aops、direct I/O | `file.c`、`data.c`、`node.c`、`checkpoint.c` | `reports/f2fs_io_path_agent_report.md` |
| F2FS Segment/GC Agent | 研究 segment、SIT/NAT、checkpoint、GC、cleaner 机制 | `segment.c`、`gc.c`、`node.c`、`checkpoint.c` | `reports/f2fs_segment_gc_agent_report.md` |
| F2FS Testing Agent | 研究 f2fs 测试、验证和后续开发检查路线 | `Kconfig`、`Makefile`、`Documentation/filesystems/f2fs.rst`、selftests | `reports/f2fs_testing_agent_report.md` |
| Review Agent | 交叉审查文档、注释和未验证项 | f2fs 文档和 patch | `reports/f2fs_review_agent_report.md` |

## 协作方式

```text
Manager Agent 建立任务板
  -> 多个研究 agent 并行阅读源码
  -> 各 agent 写入独立报告
  -> Manager 汇总为知识库文档
  -> Manager 添加少量中文学习注释
  -> Review Agent 审查报告和注释 patch
  -> Manager 修正问题、提交并推送
```

## 质量要求

- 每个结论必须引用源码文件和函数名。
- 不确定点必须单独列出，不能编造。
- 中文注释只用于学习说明，不改变逻辑。
- Review Agent 必须明确哪些检查实际运行、哪些只是建议路线。
