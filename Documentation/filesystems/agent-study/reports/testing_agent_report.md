# Testing Agent 报告：文件系统测试与验证路线

## 范围与阅读入口

本报告面向后续 agent 的小范围文件系统 patch，尤其是中文学习注释 patch 的验证路线。已阅读的主要入口包括：

- `fs/ext4/.kunitconfig`：ext4 KUnit 的最小配置，包含 `CONFIG_KUNIT=y`、`CONFIG_EXT4_FS=y`、`CONFIG_EXT4_KUNIT_TESTS=y`。
- `fs/ext4/Kconfig`：`EXT4_KUNIT_TESTS` 依赖 `EXT4_FS && KUNIT`，默认跟随 `KUNIT_ALL_TESTS`，说明测试在启动时输出 TAP 格式日志。
- `fs/ext4/Makefile`：`inode-test.o`、`mballoc-test.o`、`extents-test.o` 被合入 `ext4-test.o`，由 `CONFIG_EXT4_KUNIT_TESTS` 控制。
- `tools/testing/selftests/filesystems/` 及其子目录 Makefile。
- `Documentation/filesystems/` 中与 fstests/xfstests 有关的文档线索，主要见 `fscrypt.rst`、`fsverity.rst`、`iomap/porting.rst`。

## 1. 本地能做的轻量检查

这些检查适合在 patch 很小、风险低、主要修改注释或文档时先跑一轮：

```sh
git diff --check
git diff --stat
git diff -- fs/ext4/ Documentation/filesystems/
rg -n "TODO|FIXME|XXX|中文|学习|注释" fs/ext4 Documentation/filesystems
```

建议的静态阅读重点：

- 确认 diff 只改变预期文件和预期行，尤其中文注释 patch 不应改动条件判断、函数调用、结构体字段、宏值、Makefile/Kconfig。
- 对 ext4 源码注释，重点看注释是否贴近局部代码语义，避免把高层推测写成确定结论。
- 对文档 patch，检查标题层级、列表缩进和代码块是否符合现有 RST/Markdown 风格。

如果在完整内核树中有 `scripts/checkpatch.pl`，建议跑：

```sh
scripts/checkpatch.pl --strict --no-tree --file <changed-file>
# 或检查提交/补丁：
scripts/checkpatch.pl --strict --git HEAD
```

当前工作树是裁剪树，未看到 `scripts/checkpatch.pl` 和 `Documentation/dev-tools/`，所以上述 checkpatch 和完整文档构建命令依赖完整源码树。

如果需要语法/编译级别的轻量检查，可在完整构建环境中优先使用独立输出目录：

```sh
make O=build olddefconfig
make O=build fs/ext4/ W=1
make O=build C=1 fs/ext4/
```

其中 `C=1` 调用 Makefile 中的静态检查入口，默认 checker 是 sparse；`W=1` 打开更多编译警告。若只是改文档或注释，这属于增强验证，不是每次都必须。

## 2. KUnit/ext4 相关测试入口

ext4 的本地 KUnit 入口集中在 `fs/ext4/`：

- `.kunitconfig` 打开 `CONFIG_KUNIT`、`CONFIG_EXT4_FS`、`CONFIG_EXT4_KUNIT_TESTS`。
- `Kconfig` 中 `EXT4_KUNIT_TESTS` 说明这些测试面向内核开发者，不应进入生产配置。
- `Makefile` 将三个测试源文件编成 `ext4-test.o`。

当前看到的 ext4 KUnit suite：

- `fs/ext4/inode-test.c`：suite 名为 `ext4_inode_test`，测试 `ext4_decode_extra_time()` 对 ext4 inode 额外时间字段的解码，测试数据来自 `Documentation/filesystems/ext4/inodes.rst` 的时间戳表。
- `fs/ext4/mballoc-test.c`：suite 名为 `ext4_mballoc_test`，覆盖简单分配/释放、多块分配 buddy 生成、mark used/free、mark diskspace used，并有一个标记为 `KUNIT_SPEED_SLOW` 的成本估算用例。
- `fs/ext4/extents-test.c`：suite 名为 `ext4_extents_test`，覆盖 extent split/convert、initialized/unwritten 转换、zeroout fallback，以及 extent status cache 同步检查。

测试代码还使用了 `kunit/static_stub.h`。例如 `mballoc-test.c` stub 掉 block bitmap、group desc、mark context；`extents-test.c` stub 掉 dirty、zeroout、insert extent 等路径。`fs/ext4/ext4.h` 中 `EXPORT_SYMBOL_FOR_EXT4_TEST()` 只在 `CONFIG_EXT4_KUNIT_TESTS` 打开时把符号导出给 `ext4-test`，这说明这些测试入口不是普通运行时 ABI。

推荐命令如下，前提是完整内核树包含 `tools/testing/kunit/kunit.py`：

```sh
tools/testing/kunit/kunit.py run --kunitconfig=fs/ext4/.kunitconfig
```

若只想定位单个 suite，先用本树对应版本的 `tools/testing/kunit/kunit.py run --help` 确认过滤参数；不同内核版本的过滤语法可能有差异，不能在当前裁剪树里实际确认。

## 3. selftests 和 xfstests 的适用范围

### selftests

`tools/testing/selftests/filesystems/Makefile` 构建的主集合包括：

- `devpts_pts`
- `file_stressor`
- `anon_inode_test`
- `kernfs_test`
- `fclog`
- 扩展构建项 `dnotify_test`

顶层 `tools/testing/selftests/Makefile` 中列入了多个文件系统子目标，例如 `filesystems`、`filesystems/binderfs`、`filesystems/epoll`、`filesystems/fat`、`filesystems/overlayfs`、`filesystems/statmount`、`filesystems/mount-notify`、`filesystems/fuse`、`filesystems/move_mount`、`filesystems/empty_mntns`、`filesystems/fsmount_ns`。这些测试适合验证 VFS syscall、mount API、namespace、特定伪文件系统或用户可见行为。

常用命令：

```sh
make kselftest TARGETS=filesystems
make -C tools/testing/selftests/filesystems run_tests
make -C tools/testing/selftests/filesystems/fat run_tests
make -C tools/testing/selftests/filesystems/overlayfs run_tests
```

注意事项：

- 许多测试涉及 `mount()`、`unshare()`、`chroot()`、loop 设备或 `CAP_SYS_ADMIN`，更适合在 Linux VM 内以 root 或具备相应 capability 的用户运行。
- `fat/run_fat_tests.sh` 会创建 vfat 镜像、调用 `mkfs.vfat`，并用 `sudo mount -o loop` 挂载，不能视为普通无权限单元测试。
- 若直接在子目录运行，需确保内核 UAPI headers 已准备好；顶层 `kselftest` 目标会先跑 `headers`。
- 当前树中存在 `filesystems/eventfd`、`filesystems/open_tree_ns`、`filesystems/xattr`、`filesystems/nsfs` 等子目录，但顶层 TARGETS 列表并未全部以 `filesystems/<name>` 形式列入；这是当前裁剪树/分支中需要后续确认的点。

### xfstests/fstests

xfstests 不在当前内核树内，是外部的文件系统回归测试套件。它适合验证真正的磁盘格式、挂载选项、崩溃恢复、fsync、rename、truncate、fiemap、xattr、quota、encryption、verity 等用户可见语义。

本树文档中的线索：

- `Documentation/filesystems/fscrypt.rst` 明确说 fscrypt 使用 xfstests 测试，并给出 `kvm-xfstests -c ext4,f2fs -g encrypt`、`kvm-xfstests -c ext4/encrypt,f2fs/encrypt -g auto` 等示例。
- `Documentation/filesystems/fsverity.rst` 建议用 xfstests 测 fs-verity，例如 `kvm-xfstests -c ext4,f2fs,btrfs -g verity`。
- `Documentation/filesystems/iomap/porting.rst` 建议构建 kernel 后，用 `fstests -g all` 在多种配置上建立回归基线。

适用边界：

- 注释-only patch 通常不需要跑完整 xfstests。
- ext4 行为、I/O、分配、日志、挂载选项、磁盘格式相关 patch，应至少跑 ext4 的 quick/auto 子集。
- 涉及 fscrypt/fsverity 的 patch，应跑对应 group，例如 encrypt 或 verity。
- xfstests 需要独立 TEST_DEV 和 SCRATCH_DEV，可能会格式化设备；必须在虚拟机或专用测试盘上运行。

## 4. QEMU/虚拟机测试建议

建议把会格式化设备、需要 root/capability、可能影响宿主挂载状态的测试放到 VM：

1. 构建待测内核，准备一个最小 rootfs。
2. QEMU/KVM 启动时挂载只用于测试的 virtio-blk/virtio-scsi 磁盘，至少区分系统盘、TEST_DEV、SCRATCH_DEV。
3. VM 内安装 `xfstests`、`e2fsprogs`、`util-linux`、`acl`、`attr`、`fio`、`fsverity-utils`、`keyutils` 等按需依赖。
4. 先跑 `dmesg -w` 或保存 `dmesg`，测试后检查 WARN/OOPS/EXT4-fs error。
5. 对 ext4 行为 patch，优先用 `kvm-xfstests` 简化 VM、磁盘和配置矩阵；没有该工具时，再手工配置 fstests。

示例路线：

```sh
kvm-xfstests -c ext4 -g quick
kvm-xfstests -c ext4 -g auto
kvm-xfstests -c ext4/encrypt -g encrypt
kvm-xfstests -c ext4 -g verity
```

手工 fstests 的思路是准备 `/etc/fstests/local.config`，设置 `FSTYP=ext4`、`TEST_DEV`、`TEST_DIR`、`SCRATCH_DEV`、`SCRATCH_MNT`，再运行类似：

```sh
./check -g quick
./check -g auto
```

具体配置文件格式和 group 名称以所用 xfstests 版本为准。

## 5. 中文注释 patch 不改变逻辑时的推荐验证命令

后续如果只是给 ext4/VFS/page cache 等源码加少量中文学习注释，推荐从低成本到高成本分层验证：

```sh
git diff --check
git diff --stat
git diff --word-diff -- <changed-files>
rg -n "TODO|FIXME|XXX|不确定|可能|应该" <changed-files>
```

如果完整源码树有 checkpatch：

```sh
scripts/checkpatch.pl --strict --no-tree --file <changed-files>
```

如果注释落在 `fs/ext4/` 关键路径，且本地有可用构建环境：

```sh
make O=build olddefconfig
make O=build fs/ext4/ W=1
tools/testing/kunit/kunit.py run --kunitconfig=fs/ext4/.kunitconfig
```

如果 patch 确认只改注释，不改变编译输入、宏、结构体、函数、文档索引，通常不需要跑 selftests 或 xfstests；但合入前可由 review agent 再看一遍 diff，确认没有误改代码行。

如果 patch 虽然声称是注释，但触碰了 `Makefile`、`Kconfig`、测试代码、宏定义、trace/debug 输出或文档索引，应升级到对应构建或测试：

```sh
make kselftest TARGETS=filesystems
make -C tools/testing/selftests/filesystems run_tests
```

涉及实际行为变更时，再进入 VM 跑 xfstests quick/auto 子集。

## 6. 不确定点与不能编造的事项

- 当前工作树是裁剪过的：根目录有 `fs/`、`mm/`、`include/`、`tools/`、`Documentation/filesystems/`，但未看到 `scripts/checkpatch.pl`、`tools/testing/kunit/kunit.py`、`Documentation/dev-tools/kunit/`。因此 KUnit/checkpatch 具体命令未在本地实际执行。
- `fs/ext4/Kconfig` 引用了 `Documentation/dev-tools/kunit/`，但该目录在当前树缺失；这应理解为完整内核树中的文档入口，而不是当前本地已存在的文件。
- 顶层 selftests TARGETS 与 `tools/testing/selftests/filesystems/` 下实际子目录不完全对应，例如 `eventfd`、`open_tree_ns`、`xattr` 在子目录中存在，但当前顶层列表没有全部列出为 `filesystems/<name>`；原因可能是裁剪树、分支状态或待整理项，不能擅自下结论。
- 没有在当前环境运行 Linux 内核构建、KUnit、selftests 或 xfstests。当前宿主上下文是 Windows/PowerShell，文件系统 selftests 和 xfstests 应在 Linux 环境或 VM 中验证。
- xfstests/kvm-xfstests 是外部项目，具体 group、配置名、跳过项和依赖会随版本变化；报告中的命令是测试路线，不等于已在本机验证通过。
