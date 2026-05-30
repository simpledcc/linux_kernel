# F2FS Testing Agent 测试路线报告

## 0. 范围说明

本报告基于当前快照仓库 `D:\demo\kernel\github-publish` 的只读梳理结果。当前仓库是文件系统学习用 sparse snapshot，能看到 `fs/f2fs/`、`Documentation/filesystems/`、`tools/testing/selftests/filesystems/`、顶层 `Makefile/Kconfig/MAINTAINERS` 等，但没有完整上游内核树里的 `scripts/` 目录和许多非文件系统子系统。因此本文把“当前快照可做的轻量检查”和“需要 Linux/完整源码/外部测试套件才能运行的验证”分开列出。

## 1. 当前快照仓库里能做的轻量检查

- 工作树与协作边界检查：
  - `git status --short --branch`
  - `Get-ChildItem Documentation\filesystems\agent-study\f2fs -Force`
  - 目的：确认当前分支、未跟踪报告目录，以及避免误改其他 agent 的文件。
- f2fs 入口静态检索：
  - `rg -n -i "f2fs|xfstests|fstests|fsck\.f2fs|mkfs\.f2fs|kunit" fs\f2fs Documentation\filesystems tools\testing\selftests\filesystems`
  - 当前观察：`fs/f2fs` 内没有 KUnit 命名入口；`tools/testing/selftests/filesystems` 仅在 `statmount_test.c` 的 known_fs 列表中出现 `f2fs`，未见 f2fs 专用 selftest 目录。
- 配置/构建依赖核对：
  - `Get-Content fs\f2fs\Kconfig`
  - `Get-Content fs\f2fs\Makefile`
  - `rg -n -i "F2FS_FS|F2FS_CHECK_FS|F2FS_FAULT_INJECTION|F2FS_FS_COMPRESSION|FS_VERITY|FS_ENCRYPTION" fs\f2fs\Kconfig fs\f2fs\Makefile`
- 文档线索核对：
  - `rg -n -i "mkfs\.f2fs|fsck\.f2fs|dump\.f2fs|f2fs-tools|test_dummy_encryption|checkpoint|compress|background_gc|gc_merge|fault_injection" Documentation\filesystems\f2fs.rst`
  - `rg -n -i "kvm-xfstests|xfstests|f2fs|inlinecrypt|verity" Documentation\filesystems\fscrypt.rst Documentation\filesystems\fsverity.rst`
- 报告/注释 patch 的纯文本检查：
  - `git diff --check -- Documentation/filesystems/agent-study/f2fs/reports/f2fs_testing_agent_report.md`
  - 后续若只改中文注释，可用 `git diff --check -- fs/f2fs Documentation/filesystems/f2fs.rst` 做空白错误检查。

## 2. f2fs 相关 Kconfig/Makefile/test 线索

- `fs/f2fs/Kconfig`
  - `CONFIG_F2FS_FS` 依赖 `BLOCK`，并选择 `BUFFER_HEAD`、`NLS`、`CRC32`、`FS_IOMAP`。
  - 若启用 `CONFIG_FS_ENCRYPTION`，f2fs 会选择 `F2FS_FS_XATTR` 和 `FS_ENCRYPTION_ALGS`，说明 fscrypt 路线要同时关注 xattr。
  - `CONFIG_F2FS_CHECK_FS` 打开运行时一致性 BUG_ON 检查，适合作为开发/测试内核的强检查配置，不适合作为性能基线。
  - `CONFIG_F2FS_FAULT_INJECTION` 对应文档中的 `fault_injection=` 和 `fault_type=` 挂载选项，可覆盖 ENOMEM、ENOSPC、checkpoint、read/write I/O、block validity 等失败路径。
  - `CONFIG_F2FS_FS_COMPRESSION` 及 `F2FS_FS_LZO/LZORLE/LZ4/LZ4HC/ZSTD` 对应压缩算法后端。
  - `CONFIG_F2FS_STAT_FS`、`CONFIG_F2FS_IOSTAT` 对应 debugfs/sysfs 观测能力，适合测试时抓状态。
- `fs/f2fs/Makefile`
  - 核心对象包括 `checkpoint.o`、`gc.o`、`data.o`、`node.o`、`segment.o`、`recovery.o`、`super.o` 等。
  - 条件对象：`debug.o` 受 `F2FS_STAT_FS` 控制；`verity.o` 受 `FS_VERITY` 控制；`compress.o` 受 `F2FS_FS_COMPRESSION` 控制；`iostat.o` 受 `F2FS_IOSTAT` 控制。
- 文档和测试入口线索
  - `Documentation/filesystems/f2fs.rst` 明确外部工具链来自 f2fs-tools：`mkfs.f2fs`、`fsck.f2fs`、`dump.f2fs`，并提到 `f2fs_io` 对 QA 测试很有用。
  - f2fs 挂载选项中有测试价值的开关包括：`test_dummy_encryption`、`inlinecrypt`、`checkpoint=disable/enable`、`checkpoint_merge`、`background_gc=on/off/sync`、`gc_merge/nogc_merge`、`atgc`、`compress_*`、`fault_injection`、`fault_type`、`fsync_mode`。
  - `tools/testing/selftests/filesystems/Makefile` 只生成通用程序 `devpts_pts file_stressor anon_inode_test kernfs_test fclog`，没有 f2fs 专项 target。
  - `tools/testing/selftests/filesystems/statmount/statmount_test.c` 把 `f2fs` 列为 known filesystem name；这只能验证 statmount/listmount 这类通用接口识别，不等价于 f2fs 功能测试。
  - 当前快照中 `rg -n -i "kunit" fs\f2fs ...` 无匹配；不能声称 f2fs 已有 KUnit 测试。

## 3. xfstests/fstests 对 f2fs 的适用范围

- `fscrypt.rst` 明确说 fscrypt 应使用 xfstests，并给出 f2fs 示例：
  - `kvm-xfstests -c ext4,f2fs -g encrypt`
  - `kvm-xfstests -c ext4,f2fs -g encrypt -m inlinecrypt`
  - `kvm-xfstests -c ext4/encrypt,f2fs/encrypt -g auto`
  - `gce-xfstests -c ext4/encrypt,f2fs/encrypt -g auto`
- `fsverity.rst` 明确说 fs-verity 应使用 xfstests，并给出：
  - `kvm-xfstests -c ext4,f2fs,btrfs -g verity`
- 适用范围判断：
  - fstests/xfstests 的 generic 用例适合覆盖 VFS 语义、rename/link/fallocate/fsync/direct I/O、崩溃恢复类基础行为；f2fs 作为 FSTYP 通常可运行大量 generic 组。
  - encrypt/verity 组是当前文档直接确认适用于 f2fs 的专项路线。
  - f2fs 的 GC、checkpoint disable、compression、fault injection、zoned/multi-device 等内部策略，不能只依赖 generic 用例；需要组合挂载选项、f2fs-tools、debugfs/sysfs 指标和定制压力负载。
  - 当前 Linux 源码快照不包含 xfstests/fstests 仓库；是否有 `tests/f2fs/` 专项目录、具体 group 名称和跳过列表，必须以实际安装的 xfstests-dev/fstests 版本为准。

## 4. fscrypt/fsverity/compress/checkpoint/GC 场景测试建议

- fscrypt
  - 内核配置：`F2FS_FS=y/m`、`FS_ENCRYPTION=y`，按需要加 `FS_ENCRYPTION_INLINE_CRYPT` 和块层 inline encryption/fallback。
  - 基础组：运行 `-g encrypt`；再以 `inlinecrypt` 跑一遍。
  - 扩展组：用 `test_dummy_encryption` 跑 `auto`/generic 类用例，覆盖更多加密 I/O 路径。
  - 场景：v1/v2 policy、key add/remove、无 key 访问、rename/link 策略、direct I/O 限制、长文件名、xattr/ACL/security label、加密目录下新建/删除/恢复。
- fsverity
  - 格式化：f2fs 文档要求创建 verity 文件的文件系统需 `mkfs.f2fs -O verity`。
  - 基础组：运行 `-g verity`。
  - 场景：`FS_IOC_ENABLE_VERITY` 成功后只读、写打开失败、direct I/O 不支持、digest/metadata 读取、损坏数据读失败、加密+verity 的 post-read 解密后校验路径。
  - f2fs 特点：verity inode flag 只能由 `FS_IOC_ENABLE_VERITY` 设置且不能清除；verity metadata 存在文件尾后 64K 边界外；f2fs 不支持在 atomic/volatile writes pending 的文件上启用 verity。
- compress
  - 内核配置：`F2FS_FS_COMPRESSION=y`，按算法打开 LZO/LZORLE/LZ4/LZ4HC/ZSTD。
  - 挂载矩阵：`compress_algorithm=lzo/lz4/zstd/lzo-rle`、`compress_log_size=`、`compress_extension=`、`nocompress_extension=`、`compress_chksum`、`compress_mode=fs/user`、`compress_cache`。
  - 场景：扩展名自动压缩、目录 `chattr +c/-c` 继承、用户态 ioctl `F2FS_IOC_COMPRESS_FILE`/`F2FS_IOC_DECOMPRESS_FILE`、`F2FS_IOC_RELEASE_COMPRESS_BLOCKS`/`RESERVE_COMPRESS_BLOCKS`、压缩文件 truncate/fallocate/mmap/read/write/fsync。
  - 组合：压缩+加密时文档说明压缩后的数据再加密；建议用 fscrypt 场景叠加压缩文件，并比较 mount/unmount 后读回和 fsck 结果。
- checkpoint
  - 场景：默认 checkpoint、`checkpoint=disable`、`checkpoint=disable:N%`、`checkpoint=enable`、`checkpoint_merge/nocheckpoint_merge`。
  - 验证：写入/rename/fsync 后 remount，观察 `/sys/fs/f2fs/<dev>/unusable`，切回 `checkpoint=enable` 后检查空间回收，卸载后跑 `fsck.f2fs`。
  - 故障：结合 `fault_injection` 的 `FAULT_CHECKPOINT`、`FAULT_WRITE_IO`、`FAULT_SKIP_WRITE` 路线观察是否正确进入只读/错误处理。
- GC
  - 挂载矩阵：`background_gc=on/off/sync`、`gc_merge/nogc_merge`、`atgc`、`discard_unit=`，必要时加入 fragmentation 模式。
  - 负载：循环填满、删除、覆盖、fsync、小文件/大文件混合、pin file、压缩文件、加密文件，制造 invalid segments 后触发前台/后台 GC。
  - 观测：`/sys/kernel/debug/f2fs/status`、`/sys/fs/f2fs/<dev>/` 下 GC/checkpoint/iostat 相关节点、`F2FS_IOC_GARBAGE_COLLECT` 和 `F2FS_IOC_GARBAGE_COLLECT_RANGE` 路径、dmesg 错误。

## 5. 后续中文注释 patch 推荐验证命令

当前快照/Windows 可先做静态验证：

```powershell
git status --short --branch
git diff --check -- fs/f2fs Documentation/filesystems/f2fs.rst
rg -n -i "TODO|FIXME|f2fs|checkpoint|compress|gc|fscrypt|fsverity" fs/f2fs Documentation/filesystems/f2fs.rst
rg -n -i "kunit|xfstests|fstests|mkfs\.f2fs|fsck\.f2fs" fs/f2fs Documentation/filesystems tools/testing/selftests/filesystems
```

完整 Linux 内核源码树/Linux VM 中再做构建和运行验证：

```sh
make ARCH=x86_64 defconfig
./scripts/config -e F2FS_FS -e F2FS_STAT_FS -e F2FS_CHECK_FS \
  -e F2FS_FAULT_INJECTION -e F2FS_FS_COMPRESSION \
  -e FS_ENCRYPTION -e FS_VERITY
make ARCH=x86_64 olddefconfig
make ARCH=x86_64 -j"$(nproc)" W=1 fs/f2fs/
make ARCH=x86_64 htmldocs SPHINXDIRS=filesystems
make -C tools/testing/selftests TARGETS=filesystems
./tools/testing/selftests/run_kselftest.sh -c filesystems
kvm-xfstests -c ext4,f2fs -g encrypt
kvm-xfstests -c ext4,f2fs -g encrypt -m inlinecrypt
kvm-xfstests -c ext4,f2fs,btrfs -g verity
kvm-xfstests -c ext4/encrypt,f2fs/encrypt -g auto
```

如果完整树里存在 `scripts/checkpatch.pl`，中文注释 patch 还建议跑：

```sh
git diff --check
perl scripts/checkpatch.pl --strict --codespell -g HEAD
```

## 6. 当前 Windows/快照环境下不能实际运行的测试

- 不能实际格式化、挂载、卸载 f2fs：需要 Linux 内核、root 权限、loop/block device，以及 `mkfs.f2fs`。
- 不能实际运行 `fsck.f2fs`、`dump.f2fs`、`defrag.f2fs`、`resize.f2fs`、`sload.f2fs`、`f2fs_io`：这些来自外部 f2fs-tools，当前快照没有提供。
- 不能实际运行 xfstests/fstests：当前仓库不包含 xfstests-dev/fstests；还需要 Linux VM、TEST_DEV/SCRATCH_DEV、root 权限和 f2fs-tools。
- 不能实际跑 fscrypt/fsverity 场景：需要启用相应内核配置、f2fs 格式化特性、用户态 fscrypt/fsverity-utils 或 fstests 用例。
- 不能实际跑 KUnit：当前 `fs/f2fs` 没有 KUnit 入口，且当前快照缺少完整 KUnit/构建环境。
- 不能可靠构建内核或运行 `scripts/checkpatch.pl`：当前快照没有 `scripts/` 目录，且是在 Windows/PowerShell 环境，不是完整 Linux 构建环境。
- 不能运行当前 kselftest：虽然存在 `tools/testing/selftests/run_kselftest.sh` 和 filesystem 子目录，但该脚本依赖 Linux shell/runner、编译产物和内核运行环境；当前快照还缺少 `tools/testing/selftests/kselftest/runner.sh`。

## 7. 不确定点

- 当前报告没有运行任何真实 f2fs 挂载、fsck、xfstests、KUnit 或 kselftest；所有运行路线都是基于当前源码/文档线索提出的验证建议。
- 当前快照不是完整上游 Linux 源码树；缺失文件可能导致某些构建、脚本、KUnit 或 selftest 入口在完整树中存在但本快照不可见。
- xfstests/fstests 的 f2fs 专项目录、group 名称、skip 列表和配置名需要以实际安装版本为准；本快照中只确认了文档明确给出的 encrypt/verity 示例。
- f2fs-tools 的具体版本、manpage 选项和 `f2fs_io` 命令集未在当前仓库中验证。
- fault injection、checkpoint=disable、GC、compression 与 fscrypt/fsverity 的组合预期返回码，需要在真实内核和设备上确认，不能仅凭文档推断。
