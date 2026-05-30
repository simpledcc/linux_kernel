# Linux 文件系统学习 Agent 组织架构

## 目标

本工程使用多 agent 协同学习 Linux 内核文件系统相关代码，沉淀可追溯的中文学习文档，并为后续小范围代码注释和开发实验建立协作流程。

当前阶段目标：

1. 学习 VFS、ext4、page cache/writeback、测试体系等核心模块。
2. 每个 agent 产出独立报告，要求引用源码文件和关键函数。
3. Manager Agent 汇总各 agent 结论，形成统一知识库。
4. 在选定核心源码文件中加入少量中文学习注释，作为代码阅读标记。
5. 创建本地 Git 提交，后续推送到用户指定的 GitHub 仓库。

## Agent 角色

| Agent | 角色 | 主要输入 | 主要输出 |
|---|---|---|---|
| Manager Agent | 总控、任务拆分、整合和提交 | 全部报告、源码树、任务板 | 总结文档、协作记录、Git 提交 |
| VFS Agent | VFS 核心路径研究 | `fs/open.c`、`fs/read_write.c`、`fs/namei.c`、`include/linux/fs.h` | `reports/vfs_agent_report.md` |
| Ext4 Agent | ext4 具体文件系统研究 | `fs/ext4/file.c`、`fs/ext4/inode.c`、`fs/ext4/super.c`、`fs/ext4/ext4.h` | `reports/ext4_agent_report.md` |
| Page Cache Agent | page cache、readahead、writeback 研究 | `mm/filemap.c`、`mm/page-writeback.c`、`mm/readahead.c`、`fs/buffer.c` | `reports/page_cache_agent_report.md` |
| Testing Agent | 内核文件系统测试体系研究 | `fs/ext4/.kunitconfig`、`tools/testing/selftests/filesystems/`、`Documentation/filesystems/` | `reports/testing_agent_report.md` |
| Review Agent | 交叉审查和质量检查 | 各 agent 报告、知识文档、注释 patch | `reports/review_agent_report.md` |

## 协作流程

```text
Manager 分配任务
  -> 多个研究 agent 并行阅读源码并输出报告
  -> Manager 汇总为知识库文档
  -> Review Agent 检查引用、覆盖范围和风险
  -> Manager 根据反馈补充文档和少量中文注释
  -> 创建本地 Git 提交
```

## 质量要求

- 结论必须能追溯到源码文件和函数。
- 不确定点必须明确标出，不能编造。
- 中文注释只用于学习说明，避免大范围污染源码。
- 不修改内核逻辑，不改变编译行为。
- Git 提交应包含文档、协作记录和注释修改。
