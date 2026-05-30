# F2FS Agent 知识与协作流程

## Agent 知识分区

本轮将 F2FS 学习拆成四个知识分区：

- 架构知识：挂载、VFS 接入、核心结构体、inode 装配。
- I/O 知识：read/write/fsync、page cache、direct I/O、writeback。
- 空间管理知识：segment、SIT、NAT、SSA、checkpoint、GC。
- 测试知识：轻量检查、f2fs-tools、xfstests、fscrypt/fsverity/compress/GC 场景。

每个 agent 只负责自己的知识分区，并把结论写入独立报告。Manager Agent 负责把这些知识合并为统一视图。

## 本轮组织结构

```text
Manager Agent
  -> F2FS Architecture Agent
  -> F2FS IO Path Agent
  -> F2FS Segment/GC Agent
  -> F2FS Testing Agent
  -> Review Agent
```

## 学习过程

1. Manager Agent 建立组织架构和任务板。
2. 研究 agent 并行阅读源码，各自输出报告。
3. Manager Agent 读取所有报告，抽取共同主线。
4. Manager Agent 汇总架构、流程、学习指南和协作说明。
5. Manager Agent 在关键源码入口添加中文学习注释。
6. Review Agent 检查报告、注释和未验证项。
7. Manager Agent 根据 review 修正，提交并推送。

## 检验过程

本轮实际能做的检验：

- 确认每个研究 agent 只写自己的报告。
- 检查报告是否引用源码文件和函数名。
- 检查中文注释是否只增加 `/* ... */` 注释块。
- 对本轮修改运行 `git diff --check`。
- 由 Review Agent 明确记录未运行的测试。

当前环境不能直接证明通过：

- Linux 内核编译。
- F2FS KUnit。
- f2fs-tools 格式化、挂载、fsck。
- xfstests/fstests。
- QEMU/VM 崩溃恢复测试。

这些验证需要 Linux VM、完整源码树、f2fs-tools 和测试设备。

## 后续协作方式

后续开发任务建议使用同样的任务板格式：

```text
1. Manager Agent 写清任务和验收标准
2. Architecture/IO/GC/Testing Agent 分头分析影响面
3. Dev Agent 修改代码
4. Review Agent 检查 patch
5. Testing Agent 给出可运行验证路线
6. Manager Agent 合并、提交、推送
```

对 F2FS 来说，不建议让多个开发 agent 同时修改 `data.c`、`segment.c`、`checkpoint.c` 这类高耦合文件。更稳的方式是一个 agent 改代码，其他 agent 做影响面分析和审查。
