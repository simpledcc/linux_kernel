# Linux 文件系统源码快照

这是从本地目录 `D:\demo\kernel\linux-src` 导出的普通文件快照，不包含 Linux 上游 Git 历史。

## 快照来源

- 上游源码：`https://github.com/torvalds/linux.git`
- 本地源码分支：`agent-fs-study-cn`
- 基础提交：`f5e5d3509bffb95c6648eb9795f7f236852ae62d`
- 本地学习提交：`f93bfff4d91a51a4664951ffc78c973e34310fa1`

## 包含范围

当前仓库保留的是本轮 sparse checkout 下载到本地的代码和文档：

- `fs/`
- `include/linux/`
- `mm/`
- `Documentation/filesystems/`
- `tools/testing/selftests/filesystems/`
- Linux 顶层构建和说明文件，例如 `Makefile`、`Kconfig`、`MAINTAINERS`、`COPYING`

这不是完整 Linux 内核源码树，但足够作为文件系统学习、注释和后续小范围实验的第一版基础。

## Agent 学习产物

多 agent 协作学习记录位于：

```text
Documentation/filesystems/agent-study/
```

其中包含：

- agent 组织架构
- 任务板
- 协作运行记录
- VFS/ext4/page cache/testing/review 报告
- Manager 汇总知识库

## 后续建议

后续可以直接基于本仓库继续提交学习笔记、中文注释和实验 patch。如果需要真正编译内核或运行 KUnit/xfstests，建议重新使用完整 Linux 源码树或 Linux VM。
