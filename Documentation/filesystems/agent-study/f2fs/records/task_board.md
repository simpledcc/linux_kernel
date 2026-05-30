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
| F2FS-T009 | 深入研究挂载、inode、目录和 namei | Deep Mount/Inode/Dir Agent | done | `reports/deep_mount_inode_dir_agent_report.md` |
| F2FS-T010 | 深入研究数据 I/O、node/NAT、extent cache 和压缩 | Deep Data/Node Agent | done | `reports/deep_data_node_agent_report.md` |
| F2FS-T011 | 深入研究 segment、checkpoint、GC 和 recovery | Deep Segment/CP/Recovery Agent | done | `reports/deep_segment_checkpoint_recovery_agent_report.md` |
| F2FS-T012 | 深入研究测试、观测和特性边界 | Deep Testing/Features Agent | done | `reports/deep_testing_features_agent_report.md` |
| F2FS-T013 | 增加更详细源码学习注释 | Manager Agent | done | `fs/f2fs/*.c` |
| F2FS-T014 | 新增深度源码学习与观测测试文档 | Manager Agent | done | `knowledge/f2fs_deep_code_study.md`、`knowledge/f2fs_observability_and_tests.md` |
| F2FS-T015 | 第二轮审查和提交推送 | Review Agent / Manager Agent | done | `reports/deep_review_agent_report.md` 与 Git commit |
| F2FS-T016 | 在关键执行路径内部补充代码逻辑注释 | Manager Agent | done | `fs/f2fs/{file,data,node,namei,dir,segment,checkpoint,gc,recovery,super}.c` |

## 写入约束

- 各研究 agent 只写自己的报告文件。
- Manager 负责汇总文档和源码注释。
- Review Agent 只写审查报告，不直接修改源码。
