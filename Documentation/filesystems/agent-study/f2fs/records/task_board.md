# F2FS Agent 协作任务板

## 运行信息

- 工作目录：`D:\demo\kernel\github-publish`
- Git 分支：`main`
- 目标模块：`fs/f2fs/`

## 任务列表

| ID | 任务 | 负责人 | 状态 | 输出 |
|---|---|---|---|---|
| F2FS-T001 | 研究 f2fs 整体架构、挂载和 VFS 接入 | F2FS Architecture Agent | done | `reports/f2fs_architecture_agent_report.md` |
| F2FS-T002 | 研究 f2fs read/write/fsync 和 page cache 路径 | F2FS IO Path Agent | done | `reports/f2fs_io_path_agent_report.md` |
| F2FS-T003 | 研究 f2fs segment、checkpoint、GC 和 cleaner | F2FS Segment/GC Agent | done | `reports/f2fs_segment_gc_agent_report.md` |
| F2FS-T004 | 研究 f2fs 测试与验证路线 | F2FS Testing Agent | done | `reports/f2fs_testing_agent_report.md` |
| F2FS-T005 | 汇总 f2fs 知识库和学习指南 | Manager Agent | done | `knowledge/*.md` |
| F2FS-T006 | 添加少量中文学习注释 | Manager Agent | done | `fs/f2fs/*.c` |
| F2FS-T007 | 审查 f2fs 文档和注释 patch | Review Agent | done | `reports/f2fs_review_agent_report.md` |
| F2FS-T008 | 创建并推送 Git 提交 | Manager Agent | done | Git commit |

## 写入约束

- 各研究 agent 只写自己的报告文件。
- Manager 负责汇总文档和源码注释。
- Review Agent 只写审查报告，不直接修改源码。
