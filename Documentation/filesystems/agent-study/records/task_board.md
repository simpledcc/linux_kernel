# Agent 协作任务板

## 运行信息

- 工作目录：`D:\demo\kernel\linux-src`
- Git 分支：`agent-fs-study-cn`
- 源码来源：`https://github.com/torvalds/linux.git`
- 当前源码提交：由 Manager 在最终记录中填写。

## 任务列表

| ID | 任务 | 负责人 | 状态 | 输出 |
|---|---|---|---|---|
| T001 | 研究 VFS open/read/write 和核心对象关系 | VFS Agent | done | `reports/vfs_agent_report.md` |
| T002 | 研究 ext4 注册、读写入口、inode 和 journaling 关系 | Ext4 Agent | done | `reports/ext4_agent_report.md` |
| T003 | 研究 page cache、readahead、writeback 路径 | Page Cache Agent | done | `reports/page_cache_agent_report.md` |
| T004 | 研究文件系统相关测试和验证方式 | Testing Agent | done | `reports/testing_agent_report.md` |
| T005 | 汇总知识库和协作记录 | Manager Agent | done | `knowledge/*.md`、`records/*.md` |
| T006 | 交叉审查文档和注释 | Review Agent | done | `reports/review_agent_report.md` |
| T007 | 添加少量中文学习注释 | Manager Agent | done | 源码注释 patch |
| T008 | 创建本地 Git 提交 | Manager Agent | done | Git commit |

## 协作原则

- 各 agent 只负责自己的报告文件，避免写入冲突。
- Manager 负责汇总和最终提交。
- Review Agent 不直接修改文档，只提出阻塞问题和改进建议。
- 任何代码注释都必须保持 C 语法正确，不改变运行逻辑。
