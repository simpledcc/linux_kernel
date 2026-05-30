# Linux 文件系统多 Agent 学习产物

本分支是 `D:\demo\kernel\linux-src` 中多 agent 协同学习 Linux 文件系统代码后的发布产物。

## 重要说明

当前 GitHub 仓库 `simpledcc/linux_kernel` 不是 `torvalds/linux` 的 fork，远程 `main` 目前只包含一个轻量 README。为了避免向空仓库上传完整 Linux 内核历史，本分支只发布学习产物、补丁文件和少量带中文注释的源码副本。

如果后续希望把提交保持在真正 Linux 内核历史上，建议把 GitHub 仓库改为 `torvalds/linux` 的 fork，或新建一个 fork 仓库，然后推送本地分支 `agent-fs-study-cn`。

## 来源

- 本地源码目录：`D:\demo\kernel\linux-src`
- 本地 Linux 分支：`agent-fs-study-cn`
- Linux 基础提交：`f5e5d3509bffb95c6648eb9795f7f236852ae62d`
- 本地学习提交：`f93bfff4d91a51a4664951ffc78c973e34310fa1`

## 内容

- `Documentation/filesystems/agent-study/`：多 agent 组织架构、任务板、运行记录、知识库文档和各 agent 报告。
- `patches/0001-docs-add-multi-agent-filesystem-study-notes.patch`：可应用到 Linux 源码树的完整补丁。
- `annotated-source/`：本轮加入中文学习注释的源码文件副本，便于在 GitHub 上直接浏览。

## 本轮 Agent 分工

- VFS Agent：研究 VFS open/read/write 和核心对象关系。
- Ext4 Agent：研究 ext4 注册、读写入口、inode、extent、journal。
- Page Cache Agent：研究 page cache、readahead、dirty page、writeback。
- Testing Agent：研究 KUnit、selftests、xfstests 和 QEMU 测试路线。
- Review Agent：审查文档、注释 patch 和未验证项。
- Manager Agent：拆分任务、汇总报告、添加学习注释、创建提交。

## 应用补丁

在完整 Linux 源码树中：

```sh
git checkout f5e5d3509bffb95c6648eb9795f7f236852ae62d
git am /path/to/patches/0001-docs-add-multi-agent-filesystem-study-notes.patch
```

本轮只做学习文档和中文注释，不改变内核逻辑。
