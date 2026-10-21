# util-linux `mount` 代码阅读笔记

就 `mount` 命令而言，util-linux 的核心作用，是将用户输入与系统配置转化为可执行的挂载参数并交由内核执行，同时完成用户态结果记录。源码主要分为 `mount.c` 中的 `main()` 入口层和 libmount 库两层。

本文基于 util-linux-2.39.3 版本编写，文中写法沿用 man page 节号：`mount(2)` 指内核 `mount` 系统调用（第 2 节）；`mount(8)` 指用户态挂载管理命令（第 8 节）。后文 `umount(8)`、`fsopen(2)` 等同理。

这是主要流程：

```text
用户输入 mount 命令
        │
        ▼
main：接收并解析用户输入，把源/目标/类型/选项/挂载标志等填入 `libmnt_context`，作为 libmount 的输入
----------------------------------------------------------
进入 libmount
        ▼
结合 fstab/默认规则做补全与校验，生成最终执行参数
        │
        ├── 需要外部程序时：调用 mount.<fstype> helper 执行挂载
        └── 否则：在 libmount 内部直接执行挂载流程
        │
        ▼
调用内核挂载相关接口（按路径分两类）
        │
        ├── 新接口路径（Linux 6.17 常见）
        │      1) `fsopen`：创建文件系统配置上下文
        │      2) `fsconfig`：把源设备与挂载选项写入配置上下文；若上下文表明需建新实例或无实例可复用，则据此新建/初始化文件系统实例（含 super_block/root）
        │      3) `fsmount`：基于该文件系统实例生成“尚未挂到目标目录”的挂载对象（vfsmount/struct mount）
        │      4) `move_mount`：把挂载对象挂到目标目录
        │      5) （按场景）`mount_setattr`：设置传播/属性
        └── 旧接口路径（legacy）
               1) `mount(2)`：一次调用提交源、目标、类型、flags、data
        │
        ▼
更新用户态记录：按条件写入 utab 等补充信息
```
## 一、入口层：`main()` 命令变成上下文
### 1. 主要功能

`mount.c` 里的 `main()` 本质上是在做一件事：把 shell 里输入的 `mount` 命令，转换成一次完整的 **`libmnt_context` 挂载会话**；后续 libmount 再基于这个 context 完成真正的挂载流程。

因此，在 `main()` 这一层，重点并不是「真正执行挂载」，而是：

- 解析命令行；
- 构建 `libmnt_context`；
- 保存 source、target、options 等状态；
- 做第一层用户态语义与安全检查；
- 决定后续 libmount 该如何处理这次挂载。

`main()` 创建的核心对象是：

```c
struct libmnt_context *cxt;

cxt = mnt_new_context();
```

可以把它理解成：**一次 `mount` 命令在用户态的上下文对象**。后面的命令行解析过程，本质上就是不断把用户输入写进这个 `cxt`。

`cxt` 里比较关键的状态包括：

**表 1-1** `main()` 写入 context 的主要状态

| context 中的状态 | 命令行来源 | `main()` 中的设置方式 | 含义 |
|------------------|------------|----------------------|------|
| source | `/dev/sdb1`、`UUID=…`、`LABEL=…`、`--source`、`-L`、`-U` | `mnt_context_set_source()` | 挂载来源；常见为块设备，也可以是 bind/move 的源路径 |
| target | `/mnt/u`、`--target` | `mnt_context_set_target()` | 挂载点目录 |
| fstype | `-t ext4` | `mnt_context_set_fstype()` | 单一文件系统类型 |
| fstype pattern | `-t ext4,xfs`、`-t nofuse`、`mount -a -t …` | `mnt_context_set_fstype_pattern()` | 类型过滤模式，不一定对应单一类型 |
| option list（optlist） | `-o ro,noatime`、`-r`、`-w`、`--bind`、`--make-slave` 等 | `append_option()` → `mnt_context_append_options()` | 挂载选项字符串；先进入 optlist |
| mount flags（`MS_*`） | `ro`、`bind`、`remount`、`shared`/`slave` 等解析结果 | 多数由 optlist 经 libmount 映射；**`move` 例外**见下文 | 对应 VFS 语义与传播语义，如 `MS_RDONLY`、`MS_BIND`、`MS_SHARED`、`MS_SLAVE`、`MS_MOVE` |
| fstab | `-T <path>` | `append_fstab()` → `mnt_context_set_fstab()` | 使用指定 fstab，而非默认 `/etc/fstab` |
| options mode（optsmode） | `--options-mode`、`--options-source*` | `mnt_context_set_optsmode()` | 控制命令行 options 与 fstab/mountinfo 的合并方式 |
| cache | 隐式创建；`-T` 时与表绑定 | `mnt_table_set_cache()` | 路径规范化、标签解析等结果的缓存 |
| context flags | `-f`、`-i`、`-n`、`-v`、`--onlyonce` 等 | `mnt_context_enable_*()` / `mnt_context_disable_*()` | fake、helper、mtab、verbose 等行为开关 |
| namespace | `-N <ns>`（pid 或路径） | `mnt_context_set_target_ns()` | 在指定 mount namespace 中执行挂载 |
| restricted | `mnt_new_context()` 初始化；`suid_drop()` 等参与 | `mnt_context_force_unrestricted()`（在 `suid_drop` 内） | 是否仍按「非真 root / setuid 会话代理」一类规则处理 |

#### optlist 与选项三分

`-o`、`-r`、`-w`、`--bind` 等在 `main()` 里经 `append_option()` 进入 **`optlist`**（此时仍是选项名/字符串，见上表 option list 行）。后续 libmount 再按 map 把它们分成三类：

| 类型 | 示例 | 去向 |
|------|------|------|
| **VFS / 传播**（映射为 `MS_*` 或 `MOUNT_ATTR_*`） | `ro`、`bind`、`slave`、`remount` | `MS_RDONLY`、`MS_BIND`、`MS_SLAVE`、`MS_REMOUNT` 等；新 API 下经 `mount_setattr` |
| **文件系统私有** | `data=ordered`、`compress=zstd` | 具体 FS 驱动 / `fsconfig` 的 superblock 参数 |
| **用户态** | `user`/`users`、`loop`、`X-mount.*`、helper 相关 | libmount 用户态处理（loop、utab、idmap 等），不进内核 VFS flag |

例如 `mount -o bind,ro /src /mnt`：`bind`、`ro` 先进 optlist，prepare 阶段再分别归入 VFS 的 `MS_BIND`、`MS_RDONLY` 与（若有）其余两类。

#### `move` 的例外

源码注释写得很明确：

```c
/* "move" is not supported as option string in libmount
 * to avoid use in fstab */
```

因此：

```text
mount -o move ...
```

不会作为普通 option 留在 optlist 中，而是由 `main()` 单独记录，最后通过：

```c
mnt_context_set_mflags(cxt, MS_MOVE);
```

直接设置对应的 mount flag。

### 2. restricted 与 setuid 安全模型

当前很多 Linux 发行版里的 `/bin/mount` 仍然可能带有 setuid root。

因此普通用户执行 `mount` 时，进程一开始可能会拥有 effective uid = root。

否则很多挂载相关操作无法继续，例如：

- `mount(2)`
- loop device 设置
- propagation 修改
- namespace 操作
- 部分 helper 调用

但与此同时，mount 又是一个高风险入口。

它不仅涉及 VFS 和块设备，还涉及：

- bind / move / remount
- propagation
- loop image
- network filesystem
- helper
- 路径解析

因此 libmount 并不会简单认为：

> “当前 euid=root，就完全按真正 root 用户处理”

而是引入 restricted 语义。

它的本质是：

> 虽然 mount 当前暂时带有 setuid root 权限，但这次挂载仍然被视为：“普通用户发起的受限挂载”

因此在 restricted 模式下，libmount 会额外做一层用户态限制与检查。

典型包括：

| restricted 下的行为 | 目的 |
|---------------------|------|
| parse 阶段用了非白名单选项会 `suid_drop()` | 白名单仅 `hlLUVvrist` 与 `--source`/`--target`；会 drop 的例如 **`-o`、`-a`、`-B`、`-f`、`-n`、`-i`、`-O`、`-T`、`-N`、`--make-*`** 等 |
| 普通用户显式指定完整 source + target 时可能 `suid_drop()` | 避免普通用户继续不通过 fstab 借 setuid mount 直接发起完整挂载 |
| `sanitize_paths()` 按真实用户权限 canonicalize 路径 | 防止借 root 权限解析本来无权访问的路径 |
| 后续 libmount 会检查 user/users/owner/group 等 fstab 授权 | 普通用户不能自由挂载任意对象 |

这里最关键的一点是：`suid_drop()` 并不等于 mount 立即失败退出。它真正表示的是：放弃 setuid mount 的特权代理身份（`mnt_context_force_unrestricted()`），**不再做上表那套 restricted 授权检查**。之后程序仍然会继续执行，只是后续已经不再依赖 mount 自己的 setuid 权限。drop 之后主进程无 `CAP_SYS_ADMIN`，**`fsopen` / `mount(2)` / `move_mount` 等常规挂载 syscall 以及 fstab `user` 块设备挂载都做不了**。

### 3. 参数解析

getopt_long() 阶段处理的不只是 -o，还包括 --source、--target、-L、-U、-t、-T、-N 等选项。不同选项写入 cxt 的位置不同：

```text
-o / -r / -w / --bind / --make-*  -> append_option()，进入 option list
--source / -L / -U               -> mnt_context_set_source()
--target                         -> mnt_context_set_target()
-t                               -> 暂存到 types，后面再决定 fstype / fstype pattern
-T                               -> append_fstab()
-N                               -> mnt_context_set_target_ns()
```

所以这里真正要区分的不是“选项”和“source/target”，而是：命令行里的不同语义，会被写入 cxt 的不同位置。

循环结束后：

```text
argc -= optind;
argv += optind;
```

剩下的才是位置参数。随后 `main()` 按下述分支处理（与「其他组合」互斥）：

```text
无 source、无 target、argc == 0、非 -a，且既未使用 -B/-M/-R，也未通过 -o 留下挂载选项
    -> print_all()（只列挂载）

-a
    -> 若带 remount（has_remount_flag）：remount_all()
    -> 否则：mount_all()

argc == 0，option 阶段已设 source 或 target
    -> 沿用已有 source / target
    -> restricted 下如果 source 和 target 都已经有了：suid_drop()

argc == 1，仍缺 source 或 target
    -> 如果 argv[0] 是 LABEL=/UUID= 等 tag：
           设置为 source
       否则如果 target 已经存在：
           设置为 source
       否则：
           设置为 target

    -> restricted 下如果补完后 source 和 target 都有了，则 suid_drop()

argc == 2，option 阶段未写 source/target
    -> restricted：先 suid_drop()
    -> argv[0] -> source，argv[1] -> target

`mnt_context_mount` 之前（argv 分派之后），仍 restricted，且 sanitize_paths 失败
    -> suid_drop()

其他组合
    -> bad usage
```

这里有一个容易忽略的限制：位置参数阶段可以补全尚未指定的 source 或 target，但不能重复指定已经存在的 source 或 target。源码并不会先判断两个值是否相同；只要同一个语义位置被指定了两次，就会按用法错误处理。例如：`mount -L DATA UUID=xxxx` 或 `mount --source /dev/sda1 /dev/sda1 /mnt` 都会被认为是用法错误。

#### restricted 下显式给出 source 和 target

restricted 对这段逻辑的影响不是“改变 source/target 的含义”，而是：当受限用户显式给出了完整 source + target 时，`main()` 会倾向于 `suid_drop()`，不再让这次命令继续借 setuid mount 代理身份执行。

比如：

```text
mount /mnt
```

结果通常是：

```text
target = /mnt
source = NULL
```

还可以保留 restricted 语义，让后续 libmount 去 fstab 里查授权项。

而：

```text
mount /dev/sdb1 /mnt
```

结果是：

```text
source = /dev/sdb1
target = /mnt
```

在 restricted 场景下会 `suid_drop()`。

#### restricted 下真实用户能否解析路径

此后如果当前会话仍处于 `restricted` 状态，`main()` 还会调用：

```c
sanitize_paths(cxt)
```

这是因为历史上的 `mount` 程序通常带有 setuid root 属性，普通用户执行时，进程一度会拥有 root 的 effective uid。这样如果不额外检查，用户就可能借助 `mount` 的特权去解析自己本来无权访问的路径，例如：

- 指向他人目录的符号链接
- 无法遍历的目录
- 无权限访问的挂载点路径

本质上相当于：

> “借 setuid mount 的权限，偷看路径最终解析到了哪里”

因此，在 `restricted` 会话下，`sanitize_paths()` 会调用：

```c
canonicalize_path_restricted()
```

按调用者的真实权限，对 `target` 与 `srcpath` 做 `canonicalize` 和路径检查，而不是直接沿用 `setuid root` 的能力。

如果 `canonicalize` 失败，说明：

> 在不借助 setuid 权限的情况下，当前用户其实无法合法解析这条路径。

这时 `main()` 不会立刻报错，而是再次：

```c
suid_drop(cxt);
```

放弃 `setuid mount` 的代理身份。之后这次 `mount` 是否还能继续成功，就不再依赖用户态的 `setuid` 代理，而由后续真实用户权限以及内核策略决定。

## 二、进入 libmount：`mnt_context_mount()` 执行链

第一章在 `mount.c` 中完成命令行解析、restricted 检查，并将 source、target、options 等初始挂载信息写入 `libmnt_context`。本章继续进入 libmount，分析 `mnt_context_mount()` 如何准备挂载参数、执行实际挂载并更新 utab。

### 1. 从 `main()` 到 libmount 的交接

第一章已经创建了本次操作使用的 context：

```c
struct libmnt_context *cxt;

cxt = mnt_new_context();
```

`cxt->fs`、`cxt->optlist` 已包含第一章建立的初始挂载请求：

| 对象             | 作用                                                           |
| -------------- | ------------------------------------------------------------ |
| `cxt->fs`      | 本次挂载对应的 `struct libmnt_fs`，保存 source、target、fstype 等文件系统描述信息 |
| `cxt->optlist` | 本次挂载的 `struct libmnt_optlist`，保存结构化的 mount options           |

此时这两个对象中主要还是 `mount.c` 根据命令行建立的初始信息。进入 libmount 后，还会继续根据 fstab、路径解析和权限处理等结果对它们进行补全和整理。

相比之下，用于更新 utab 的 `cxt->update` 此时尚未建立。

#### 1.1 `-EPERM` 后的第二次整轮重试

第一次 `mnt_context_mount()` 在 restricted 状态下可能于 mount syscall 之前返回 `-EPERM`。`main()` 检查到当前仍为 restricted，并且 libmount 尚未调用 mount syscall 时，会先执行 `suid_drop()`，然后重新调用一次 `mnt_context_mount()`：

```c
rc = mnt_context_mount(cxt);
if (rc == -EPERM && mnt_context_is_restricted(cxt)
    && !mnt_context_syscall_called(cxt)) {
        suid_drop(cxt);
        rc = mnt_context_mount(cxt);
}
```

`suid_drop()` 通过 `drop_permissions()` 将 euid 降回真实 uid，并调用 `mnt_context_force_unrestricted(cxt)`。因此第二次进入 libmount 时，不再执行 restricted 模式下的 userspace 限制。

普通用户并不会因此获得内核的 mount 权限：普通文件系统最终调用 `fsopen` / `move_mount` / legacy `mount(2)` 时仍可能得到 `EPERM`。这次重试主要为能够通过 helper 等用户空间机制完成挂载的情况保留机会，例如 FUSE。

`!mnt_context_syscall_called(cxt)` 则保证：如果第一次的 `EPERM` 已经来自实际的 mount syscall，就不会再进行这种重试。

### 2. `mnt_context_mount()` 总控

`main()` 在 argv 分派、`sanitize_paths()`、`MS_MOVE` 等处理完成后调用：

```c
rc = mnt_context_mount(cxt);
```

从这里开始，挂载操作由 libmount 接管。`mnt_context_mount()` 按以下顺序组织整个挂载过程：

```c
again:
    rc = mnt_context_prepare_mount(cxt);
    if (!rc)
        rc = mnt_context_prepare_update(cxt);
    if (!rc)
        rc = mnt_context_do_mount(cxt);
    if (!rc)
        rc = mnt_context_update_tabs(cxt);

    /* EROFS/EACCES/EBUSY → 加 MS_RDONLY → goto again */

    if (rc == 0)
        rc = mnt_context_call_hooks(cxt, MNT_STAGE_POST);
```

因此整个过程可以分为 **四个主体阶段 + 一个 POST 阶段**（后面的小节就按这个顺序展开）：

| 阶段               | 主要工作                                                                                           |
| ---------------- | ---------------------------------------------------------------------------------------------- |
| `prepare_mount`  | 补全 source/target/fstype，合并 fstab 与命令行 options，检查普通用户权限，并准备各类 mount hooks                       |
| `prepare_update` | 判断是否需要更新 utab，并准备 `cxt->update` 及 `update->fs`                                                 |
| `do_mount`       | 调用 filesystem helper，或通过 new mount API / legacy `mount(2)` 执行实际挂载；内部还包含 `MNT_STAGE_MOUNT_POST` |
| `update_tabs`    | 挂载成功后，将准备好的记录提交到 `/run/mount/utab`                                                             |
| `MNT_STAGE_POST` | 整个挂载事务完成后的最终 hooks，例如处理 `X-mount.owner/group/mode`                                             |

其中，`MNT_STAGE_MOUNT_POST` 与最后的 `MNT_STAGE_POST` 层次不同：前者属于 `do_mount()` 内部的挂载收尾阶段；后者由 `mnt_context_mount()` 在 `update_tabs` 完成后调用。

随着这几个阶段推进，还会出现另外两个与 utab 更新有关的对象：

| 对象                | 类型                        | 核心作用                                                   | 主要阶段                                                                      |
| ----------------- | ------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------- |
| `cxt->fs`         | `struct libmnt_fs *`      | 本次挂载对象：source、target、fstype、root 等                     | `prepare_mount` 补全、规范化；`do_mount` 使用并可回填 mount ID                         |
| `cxt->optlist`    | `struct libmnt_optlist *` | 本次挂载的结构化 options，是后续 flags、attrs、helper options 等处理的基础 | `prepare_mount` 合并、整理；`do_mount` 各 hooks 继续读取                             |
| `cxt->update`     | `struct libmnt_update *`  | 管理本次 utab 更新，包括目标文件及更新状态                               | `prepare_update` 创建和准备；`update_tabs` 提交                                   |
| `cxt->update->fs` | `struct libmnt_fs *`      | 本次准备写入 utab 的 entry                                    | `prepare_update` 从 `cxt->fs` 拷贝形成；`do_mount` 回填 mount ID、成功的 `try_type` 等 |

这里需要区分 `cxt->update` 和 `cxt->update->fs`：

```text
cxt->update
    → 管理这次 utab 更新

cxt->update->fs
    → 保存准备写入 utab 的具体 entry
```

`prepare_update` 创建更新对象并准备 `update->fs`；`do_mount` 执行真正的挂载，其中 new mount API 路径可在 mount 建立后取得 mount ID，而 filesystem type 需要逐项试挂载时，成功的 `try_type` 也会写入 `update->fs->fstype`；最后 `update_tabs` 将准备好的 entry 提交到 utab。

### 3. `prepare_mount`：按源码调用顺序展开

`mnt_context_prepare_mount()` 在 `mnt_context_mount()` 内首先执行，把 `main()` 传入的 context 补全、校验，编译成可执行的挂载计划。子节按源码调用顺序编号。

总流程：

```c
rc = mnt_context_apply_fstab(cxt);
if (!rc) rc = mnt_context_merge_mflags(cxt);
if (!rc && cxt->fs && cxt->optlist)
	rc = mnt_fs_follow_optlist(cxt->fs, cxt->optlist);
if (!rc) rc = evaluate_permissions(cxt);
if (!rc) rc = fix_optstr(cxt);                    /* → PREP_OPTIONS */
if (!rc) rc = mnt_context_prepare_srcpath(cxt);   /* → PREP_SOURCE */
if (!rc) rc = mnt_context_guess_fstype(cxt);
if (!rc) rc = prepare_target(cxt);                /* → PREP_TARGET */
if (!rc) rc = mnt_context_prepare_helper(cxt, "mount", NULL);
if (!rc && mnt_context_is_onlyonce(cxt)) {
	int mounted = 0;
	rc = mnt_context_is_fs_mounted(cxt, cxt->fs, &mounted);
	if (rc == 0 && mounted == 1)
		rc = -MNT_ERR_ONLYONCE;
}
if (!rc) rc = mnt_context_call_hooks(cxt, MNT_STAGE_PREP);
```

#### 3.1 `mnt_context_apply_fstab`：补全 source/target/options

`mnt_context_apply_fstab` 是 `prepare_mount` 的第一步：根据当前 mount context 的 source、target 和 optsmode，
判断是否需要从 fstab 或 mountinfo 中查找已有记录，
然后把找到的表项合并到当前 cxt->fs，
用于补全 source、target、fstype、options 等信息，并将该行 options 按 optsmode 并入 `cxt->optlist`。

```text
mnt_context_apply_fstab(cxt)
│
├─ 参数检查
│   └─ 已应用过？
│         └─ 是 → return
│
├─ 确定 optsmode（见 3.1.1）
│
├─ 是否需要查表？（见 3.1.2）
│   ├─ 不需要
│   │     └─ return
│   │
│   └─ 需要
│
├─ 切换到目标 namespace
│
├─ 优先查 fstab
│   │
│   ├─ 命中
│   │     └─ apply_table()（见 3.1.3）
│   │
│   └─ 未命中
│
├─ 特定场景退回 mountinfo
│   │
│   └─ remount / umount
│         └─ apply_table()
│
├─ 恢复原 namespace
│
├─ 结果修正
│   ├─ 查表失败处理
│   └─ remount bind 修正
│
└─ return rc
```
##### 3.1.1 确定 optsmode

对应流程「确定 optsmode」一步；在 `mnt_context_apply_fstab()` 开头，若 `cxt->optsmode` 尚未设定则写入默认值，并处理 `MNT_OMODE_NOTAB` 对 FSTAB/MTAB/FORCE 位的清理。

```c
if (mnt_context_is_restricted(cxt)) {
	cxt->optsmode = MNT_OMODE_USER;
} else if (cxt->optsmode == 0) {
	cxt->optsmode = MNT_OMODE_AUTO;
} else if (cxt->optsmode & MNT_OMODE_NOTAB) {
	cxt->optsmode &= ~MNT_OMODE_FSTAB;
	cxt->optsmode &= ~MNT_OMODE_MTAB;
	cxt->optsmode &= ~MNT_OMODE_FORCE;
}
```

| 模式 | 含义 |
|------|------|
| `MNT_OMODE_USER` | restricted 强制；fstab 的 options **覆盖** 命令行 `-o` |
| `MNT_OMODE_AUTO` | root 默认；source+target 都有时可 skip fstab（见 3.1.2） |
| `MNT_OMODE_FORCE` | 必须查表 |
| `MNT_OMODE_FSTAB` / `MNT_OMODE_MTAB` | 控制查 fstab 还是 mountinfo |

**默认赋值**：非 root → `MNT_OMODE_USER`（= REPLACE \| FORCE \| FSTAB）；root 且命令行未指定 optsmode → `MNT_OMODE_AUTO`（= PREPEND \| FSTAB \| MTAB）。

##### 3.1.2 是否查表

本节判断 `mnt_context_apply_fstab()` 是否要提前 `return`（不查任何表）。若不 skip，则按 `optsmode` 里的 **FSTAB / MTAB** 位依次查 **fstab**、必要时再查 **mountinfo**（仅 remount/umount）；skip时 **fstab 与 mountinfo 都不查**。

```c
/* fstab is not required if source and target are specified */
if (src && tgt && !(cxt->optsmode & MNT_OMODE_FORCE)) {
	return 0;   /* skip */
}

if (!src && tgt
    && !(cxt->optsmode & MNT_OMODE_FSTAB)
    && !(cxt->optsmode & MNT_OMODE_MTAB)) {
	return 0;   /* skip, probably MS_PROPAGATION */
}
```

满足下列之一则 **不查表**（`return 0`，不标 `MNT_FL_TAB_APPLIED`）：

- `src && tgt && !FORCE`：root 显式 `mount /dev/sda1 /mnt` 不必查 fstab。
- 仅 `target`、且无 FSTAB/MTAB 位：常见于 `mount --make-*` 传播类操作。

**不查表时（skip）**，`fstype` 不从 fstab 来：命令行已带 `-t` 则写入 `cxt->fs`；否则在 `prepare_mount` 后续 `mnt_context_guess_fstype()` 根据 source 路径用 blkid 探测 `TYPE`。

##### 3.1.3 查找逻辑：`apply_table()`

```text
apply_table(cxt, tab, …)
│
├─ 按 cxt->fs 已有字段 find_pair / find_source / find_target
│   └─ swapmatch：首次未命中可交换 src/tgt 列再查（见下）
├─ 未命中 → -MNT_ERR_NOFSTAB
└─ 命中   → apply_fs()（见 3.1.4）
```

- 同时有 source+target → `mnt_table_find_pair`
- 只有 source → `mnt_table_find_source`
- 只有 target → `mnt_table_find_target`
- `swapmatch` 开启时还会交换 source/target 再试（bind、符号链接设备等）

查找使用 `cxt->fstab`（懒加载 `/etc/fstab`）。若 fstab 未命中且为 remount/umount，还会尝试 **mountinfo**（`apply_table(..., MNT_ITER_BACKWARD)`）。

##### 3.1.4 `apply_fs`

`apply_fs()` 把 fstab/mountinfo 里**匹配到的行**的字段**抄进** `cxt->fs`，并合并 options 到 `optlist`：

```c
/* 字段拷贝 */
rc = mnt_fs_set_source(cxt->fs, mnt_fs_get_source(fs));
rc = mnt_fs_set_target(cxt->fs, mnt_fs_get_target(fs));
if (!mnt_fs_get_fstype(cxt->fs))
	rc = mnt_fs_set_fstype(cxt->fs, mnt_fs_get_fstype(fs));
if (!mnt_fs_get_root(cxt->fs) && mnt_fs_get_root(fs))
	rc = mnt_fs_set_root(cxt->fs, mnt_fs_get_root(fs));

/* options：按 optsmode 分支（§3.1.1） */
if (cxt->optsmode & MNT_OMODE_IGNORE)
	;
else if (cxt->optsmode & MNT_OMODE_REPLACE)
	rc = mnt_optlist_set_optstr(ls, mnt_fs_get_options(fs), NULL);
else if (cxt->optsmode & MNT_OMODE_APPEND)
	rc = mnt_optlist_append_optstr(ls, mnt_fs_get_options(fs), NULL);
else if (cxt->optsmode & MNT_OMODE_PREPEND)
	rc = mnt_optlist_prepend_optstr(ls, mnt_fs_get_options(fs), NULL);

if (!rc)
	cxt->flags |= MNT_FL_TAB_APPLIED;
```

| optsmode | 调用 | 默认场景 | 表项 options 写入方式 | 相对命令行 `-o` |
|----------|------|----------|----------------------|-----------------|
| **REPLACE** (`MNT_OMODE_REPLACE`) | `mnt_optlist_set_optstr` | restricted（`MNT_OMODE_USER`） | 先删 optlist 里已有**字符串来源**项，再写入表项 | 命令行 `-o` 字符串项被清掉，**以表为准** |
| **PREPEND** (`MNT_OMODE_PREPEND`) | `mnt_optlist_prepend_optstr` | root（`MNT_OMODE_AUTO`） | 表项 **插到链表头** | 命令行 `-o` 仍在后面 |
| **APPEND** (`MNT_OMODE_APPEND`) | `mnt_optlist_append_optstr` | 显式 append 模式 | 表项 **接到链表尾** | 命令行 `-o` 仍在前面 |
| **IGNORE** (`MNT_OMODE_IGNORE`) | — | — | 不写表项 | 只用命令行已有 optlist |

**同名选项冲突**：表项与命令行 `-o` 都进 optlist 后，最终谁生效由 **`§3.2.2` `merge_mflags`** 统一去重（不是 `apply_fs` 里 prepend/append 时决定）。规则与示例见 §3.2。

注意：**`cxt->fs` 与表 `cxt->fstab` 的行不是同一个对象**：`cxt->fstab` 是只读的共享视图，`cxt->fs` 是这次挂载可能会改来改去的选项集。

##### 3.1.5 `mount -a` 不走 `apply_fstab`

`mount -a` 在 `main()` 调用的 `mount_all()` 里循环 `mnt_context_next_mount()`：

```text
mount -a 每行：
  mnt_context_next_mount() 取 fstab 当前行
  mnt_reset_context()               /* 在 next_mount() 内部 */
  mnt_context_apply_fs(cxt, *fs)    /* 直调 apply_fs()，不经 apply_table() 查表 */
  mnt_context_mount(cxt)            /* prepare_mount 仍调 apply_fstab，但 apply_fs 已设 MNT_FL_TAB_APPLIED → 查表 skip；其余与单次 mount 相同 */
```

- 迭代器从 `cxt->fstab` 取当前行
- 过滤 noauto、根、` -t`/`-O` pattern、已挂载等
- `-F` 时对每行 `fork`，父进程按顺序 `waitpid`（并行发起、顺序完成）

##### 3.1.6 bind / move 与 fstab 的关系

`linux_flags_map` 里：

- `bind` / `rbind`：正常 map 项，**可以**出现在 fstab（持久 bind 挂载）
- `move`：带 `MNT_NOFSTAB | MNT_NOHLPS | MNT_NOMTAB`，**不应**进 fstab；`mount.c` 解析 `-o move` 时会剥离该选项并设 `is_move`，最终在调用 `mnt_context_mount()` 前执行 `mnt_context_set_mflags(cxt, MS_MOVE)`（与 `-M` / `--move` 等价）

原因：`move` 是一次性**搬迁**已有挂载树，不是「声明式持久配置」；`bind` 则是可重复的挂载关系描述。

##### 3.1.7 cache 与 canonicalize（`mount -c`）

**`mount -c`（`--no-canonicalize`）的含义**

平时 `mount` 会帮你「收拾」路径字符串：把 `UUID=...` / `LABEL=...` 查成 `/dev/...`，把相对路径、符号链接收成绝对路径，查 fstab 时也会用规范化后的路径去对挂载点。加 **`-c`** 就是告诉 libmount：**这些自动整理统统不要做**——`source` / `target` 保持你（或 fstab）写进去的原样，不查 UUID/LABEL，不把路径 canonicalize（规范化）。

适合的场景：chroot 里路径和宿主机不一致、调试想看 libmount 实际拿了什么字符串、或调用方已经给好绝对路径、希望自己完全控制传给 `mount(2)` 的字符串。man page 写明：**正常手工挂载不建议用 `-c`**。它主要给 **`/sbin/mount.<type>` helper 的二次回调**用：helper 自己解析好 source/target 后，再 `exec mount -i -c ...`——**`-i`** 表示不再调 helper（`--internal-only`），**`-c`** 表示别再 canonicalize 一遍；两个选项各管一件事，不要混成一个「给 helper 用」。

**代价**：`source` 若是 `UUID=` / `LABEL=`，`prepare_srcpath` 不会解析，会 **`MNT_ERR_NOSOURCE` 失败**——`-c` 下应自己写 `/dev/...` 等真实路径。

**实现**：`mount -c` → `mnt_context_disable_canonicalize` → `mnt_context_get_cache()` 返回 NULL。context 主路径里（`apply_fstab` 查表、`prepare_srcpath` / `prepare_target` 准备 source/target）**不再**走 TAG 解析和路径 canonicalize，也**不再**往 `libmnt_cache` 里缓存结果。

**`libmnt_cache` 里的值**（`cache.c`）

| 成员 | 作用 |
|------|------|
| `ents[]` | key→value：路径 canonicalize 结果；UUID/LABEL → 设备路径 |
| `bc` | libblkid 句柄，解析 TAG 时复用 |
| `mountinfo` | 可挂一张当前已挂载表；若目标路径已是表中的挂载点，路径解析时可不再做 canonicalize。普通 `mount` 主流程一般不设；`findmnt` 等会用到 |

`cxt->cache` 与 `cxt->fstab->cache`、`cxt->mountinfo->cache` 懒创建后指向**同一份** `libmnt_cache`，多张表共享 `ents[]` 命中。

**两层机制**

| 层 | `-c` 的影响 |
|----|-------------|
| **Context 层**（`get_cache()`、`prepare_srcpath` / `prepare_target`、查表挂 cache 等） | `get_cache()` 返回 NULL；prepare 阶段**跳过** TAG 解析和路径 canonicalize，也不往 cache 里记结果 |
| **底层 API**（`mnt_resolve_path` / `mnt_resolve_tag` / `mnt_resolve_spec`） | **不读** `-c` 开关；别处若仍直接调用这些函数，**仍会**做解析，只是不走 context 那套 cache |

因此 `-c` **不是**「整个 libmount 都不 canonicalize」；只是 **mount context 这条挂载准备链路**不再自动整理字符串。例如 `evaluate_permissions` 里 `owner`/`group` 分支仍可能调 `mnt_resolve_spec` 做 `stat`，与 `prepare_srcpath` 策略并不完全一致。

#### 3.2 `mnt_context_merge_mflags`：折叠 optlist

在 `apply_fstab`（§3.1.4 把表项插/接到 optlist）之后、`evaluate_permissions` 之前执行。第一章已说明命令行 `-o` 先进 **optlist**；本节先交代 prepare 阶段要用的结构，再说本步做什么。

##### 3.2.1 `optlist` 里有什么

`cxt->optlist`（`struct libmnt_optlist`）是本次挂载**选项的权威来源**：命令行 `-o`、fstab 合并进来的选项串、以及代码里插入的 flag（如 `MS_SECURE`），都会先被解析成链表上的 `libmnt_opt` 节点挂在这个结构里；后面 `merge_mflags`、`get_flags`、各 hook 和挂载 syscall **都从optlist取**。

| 主要成员 | 含义 |
|----------|------|
| `opts` | 链表头，挂着多个 `struct libmnt_opt`（顺序有意义：prepend/append 决定先后） |
| `linux_map` / `maps[]` | 已注册的选项对照表：`linux_flags_map`（`ro`→`MS_RDONLY`…）、`userspace_opts_map`（`user`、`loop`…）等 （见3.2.4节）|
| `merged` | `mnt_optlist_merge_opts()` 之后置 1，表示已完成折叠去重 |
| `cache_*` | 按 map 懒缓存解析后的 `MS_*` 位或 optstr 片段 |

**`opts` 链表**（见上表）上的每一项 **`libmnt_opt`**：

| 字段 | 含义 |
|------|------|
| `name` / `value` | 如 `ro`；或 `data=ordered` 的 name/value |
| `map` / `ent` | 若名字命中某张 optmap，指向表项（含 `MS_*` id、mask） |
| `src` | `MNT_OPTSRC_STRING`（来自 `-o`/fstab 字符串）或 `MNT_OPTSRC_FLAG`（代码直接插 flag） |
| `external` | 标记为仅给外部 helper 可见的项 |

**merge 之前**，链表上可以同时存在重复或互斥项（例如 fstab 的 `ro` 与命令行的 `rw`）；此时只是「都插进去了」，**尚未**决定最终生效哪一个。

##### 3.2.2 本步：`merge_mflags` 做什么

```c
int mnt_context_merge_mflags(struct libmnt_context *cxt)
{
	...
	cxt->flags |= MNT_FL_MOUNTFLAGS_MERGED;
	return mnt_optlist_merge_opts(ls);
}
```

**本步变动**：调用 `mnt_optlist_merge_opts()`——对 `opts` 链表**去重、处理互斥**（如 `ro`/`rw`），规则是 **keep last**（同名及互斥项只保留链表中最后一项）；置 `MNT_FL_MOUNTFLAGS_MERGED`。此后 `evaluate_permissions` 等步骤都断言该标志已置位。

与 §3.1.4 optsmode 的配合：

| optsmode | merge 前链表顺序（简记） | keep last 后谁赢 |
|----------|--------------------------|------------------|
| PREPEND | 表项在前，命令行 `-o` 在后 | 命令行覆盖表项 |
| APPEND | 命令行在前，表项在后 | 表项覆盖命令行 |
| REPLACE | 表项已 `set_optstr` 清掉命令行字符串项 | 以表项为准 |

例：root、fstab `defaults,ro`、命令行 `-o rw`、PREPEND → merge 后倾向 `rw`。

##### 3.2.3 选项从哪来、怎么分类

`main()` 里 `-o` 只是字符串，**分类发生在** `mnt_context_append_options()` 解析时（见表 1-1 的 option list 行）：靠 `optlist->maps[]` 里注册的 `linux_flags_map` / `userspace_opts_map` 做 name 匹配（`optmap.c`）。map 表项类型如下：

```c
struct libmnt_optmap {
	const char *name;
	int         id;
	int         mask;
};
```

- **`name`**：用来匹配选项字符串
- **`id`**：对应的 `MS_*`（见表 1-1 的 mount flags 行）
- **`mask`**：附加语义，取值见下表

| mask | 作用 | 例子 |
|------|------|------|
| `MNT_SUPERBLOCK` | 超级块侧参数（`configure_superblock` → `fsconfig`） | `sync`、`lazytime` |
| `MNT_NOHLPS` | 不传给外部 mount helper（如 `/sbin/mount.<type>`） | `bind`、`shared`、`move` |
| `MNT_NOMTAB` | 不进 utab **`OPTS=`**（linux map 上还有传播、`remount` 等，见 optmap） | userspace 例：`noauto`、`users`、`offset=`；linux 例：`shared`、`remount` |
| `MNT_NOFSTAB` | 不应出现在 fstab | `move` |
| `MNT_INVERT` | 否定形式选项：与正向选项共用同一 `id`，但表示**清除**该 flag | `rw` 清除 `MS_RDONLY`（相对 `ro`）；`noexec` 清除 `MS_NOEXEC`（相对 `exec`） |
| `MNT_PREFIX` | 前缀匹配 | userspace 的 `x-` → `X-mount.*` |

选项按下表分类：

| 类别 | 识别方式 | 示例 | 主要消费者 |
|------|----------|------|------------|
| **VFS** | linux map 中无 `MNT_SUPERBLOCK`，且 **`get_attrs` 认得出**（能转成 `MOUNT_ATTR_*`；完整列表见 3.2.4） | `ro`、`nosuid`、`noatime` | map 里 id 仍是 `MS_*`（新旧共用）。新 API：`hook_set_vfsflags` 调 `get_attrs`，写入 `mount_setattr` 的 `attr_set`/`attr_clr`；legacy：`get_flags` → `mount(2)` 的 flags |
| **传播** | linux map，`id` 带传播位（`MS_SHARED` / `MS_SLAVE` 等） | `shared`/`rshared`、`slave`/`rslave`、`private`、`unbindable` | **`get_attrs` 认不出**。新 API：`mount_setattr` 的 `propagation` 字段；legacy：额外的 `mount(2)`。常带 `MNT_NOHLPS`/`MNT_NOMTAB` |
| **Superblock** | linux map + `MNT_SUPERBLOCK` | `sync`、`lazytime`、`dirsync` | map 里 id 也是 `MS_*`。新 API：`configure_superblock` → `fsconfig`；legacy：同样进 `mount(2)` 的 **flags** |
| **FS 专有** | **不属于两张 builtin map，且不是 libmount 特殊识别的安全选项；由具体 filesystem 定义和解析** | ext4 的 `data=ordered`、`journal_checksum`；cifs 的 `vers=`、`sec=` | 新 API：`configure_superblock` 中按 name/value 调 `fsconfig`；legacy：拼成字符串作为 `mount(2)` 第 5 参，最终由具体 filesystem 解析 |
| **安全** | **不属于两张 builtin map，但被 libmount 按 option name 专门识别** | SELinux：`context=`、`fscontext=`、`defcontext=`、`rootcontext=`、`seclabel`；Smack：`smackfsdef=` 等 | SELinux 由 `hook_selinux`（`PREP_OPTIONS`）做 context 翻译等预处理，再进入内核挂载流程；Smack 选项在 `fix_optstr` 里按名字表识别（未启用 Smack 时删除） |
| **Userspace** | 命中 userspace 专用那张选项表 | `user`、`loop`、`X-mount.*`、`helper=`、`verity.*` 等 | 由 libmount 用户态逻辑消费：如 `loop` 准备 loop device、`verity.*` 准备 dm-verity device、`X-mount.idmap` 做 idmap、`user` 等用于 utab |
| **操作类** | linux map 里 `id` 为 `MS_BIND` / `MS_MOVE` / `MS_REMOUNT` | `bind`/`rbind`、`move`、`remount` | 不设普通 VFS 属性，也不走超级块参数；而是决定 libmount 运行 bind、move 或 remount 的相应流程。可带 `MNT_NOMTAB` / `MNT_NOHLPS` |

##### 3.2.4 `get_attrs()`：VFS 选项 → `MOUNT_ATTR_*`

新 API 下设置 VFS 选项的过程：`mnt_optlist_get_attrs()`（`optlist.c`）遍历 optlist 的 linux map 项，把能识别的选项聚成两个 `uint64_t` 位图 **`set` / `clr`**，再写入 `mount_setattr` 的 **`attr_set` / `attr_clr`**。

**能识别的映射（共 9 项）：**

| 选项（map 名） | `MS_*`（optmap `id`） | `MOUNT_ATTR_*` |
|----------------|----------------------|----------------|
| `ro` / `rw`（`rw` 为 invert） | `MS_RDONLY` | `MOUNT_ATTR_RDONLY` |
| `nosuid` / `suid` | `MS_NOSUID` | `MOUNT_ATTR_NOSUID` |
| `nodev` / `dev` | `MS_NODEV` | `MOUNT_ATTR_NODEV` |
| `noexec` / `exec` | `MS_NOEXEC` | `MOUNT_ATTR_NOEXEC` |
| `nodiratime` / `diratime` | `MS_NODIRATIME` | `MOUNT_ATTR_NODIRATIME` |
| `relatime` / `norelatime` | `MS_RELATIME` | `MOUNT_ATTR_RELATIME`（值为 0） |
| `noatime` / `atime` | `MS_NOATIME` | `MOUNT_ATTR_NOATIME` |
| `strictatime` / `nostrictatime` | `MS_STRICTATIME` | `MOUNT_ATTR_STRICTATIME` |
| `nosymfollow` / `symfollow` | `MS_NOSYMFOLLOW` | `MOUNT_ATTR_NOSYMFOLLOW` |

上表即 `get_attrs` 的全部映射。**`MOUNT_ATTR_IDMAP` 不在其中**：它不是 linux map 的 `MS_*` 项，由 userspace 选项 `X-mount.idmap` 发起最终走 `mount_setattr`，不经 `get_attrs`。

**atime 三者互斥：** `relatime` / `noatime` / `strictatime` 同时只能生效一种。设或清其中任一者时，还会向 `clr` 并入掩码 **`MOUNT_ATTR__ATIME`**（UAPI 名即两个下划线，表示 atime 模式位段的整段掩码），先清掉旧 atime 模式再写入新模式。`MOUNT_ATTR_RELATIME` 本身为 0，常靠「清 `MOUNT_ATTR__ATIME` + set 为 0」表示切回 relatime。

**`nodiratime`：** 只抑制**目录**的 atime 更新，与上面三种 atime 策略独立，不走 `MOUNT_ATTR__ATIME` 互斥。

#### 3.3 `mnt_fs_follow_optlist`：让 `fs` 跟随 context 的 `optlist`

`mnt_context_merge_mflags()` 完成后，`prepare_mount()` 调用：

```c
mnt_fs_follow_optlist(cxt->fs, cxt->optlist);
```

这个函数并不复制选项，而是让：

```text
cxt->fs->optlist ──┐
                   ├──> 同一个 libmnt_optlist
cxt->optlist ──────┘
```

同时增加该 `optlist` 的引用计数。此后对 `cxt->optlist` 的修改，例如 `fix_optstr()` 或后续 hook 对选项的调整，也会直接反映在 `cxt->fs->optlist` 所指向的对象中。

这里需要区分 `libmnt_fs` 中的两种选项表示：

```text
libmnt_fs
├── optstr / vfs_optstr / fs_optstr / user_optstr   字符串形式
└── optlist                                          结构化选项列表
```

`libmnt_fs` 本身可以脱离 `libmnt_context` 独立使用。例如解析 `/etc/fstab`、`/proc/self/mountinfo` 等表时，每一行都表示为一个 `libmnt_fs`，这类 `fs` 通常直接保存解析得到的选项字符串。

而在一次具体的 mount 操作中，选项需要经历命令行与 fstab 合并、权限处理以及 hook 修改等过程，因此 `cxt->optlist` 才是本次操作中持续维护的结构化选项列表。`mnt_fs_follow_optlist()` 让 `cxt->fs` 也引用这份列表，避免 `fs` 中保存的选项表示与 context 当前的选项状态脱节。

绑定之后，`mnt_fs_get_options()`、`mnt_fs_get_vfs_options()`、`mnt_fs_get_fs_options()`、`mnt_fs_get_user_options()` 等接口在需要返回字符串时，会根据 `fs->optlist` 的变化按需更新相应的字符串缓存；`mnt_fs_follow_optlist()` 本身并不会立即重建这些字符串。

因此，在 mount 的执行流程中可以简单区分为：

* 对本次挂载选项进行查找、增删、过滤或计算 flags/attributes，主要操作 `cxt->optlist`；
* 需要通过 `libmnt_fs` 接口取得选项字符串时，使用 `mnt_fs_get_*options()`。如果这个 `fs` 已经 follow `cxt->optlist`，得到的字符串就是当前 optlist 的字符串视图；如果它只是从 fstab、mountinfo 等表中解析得到的普通 `fs`，则读取其自身保存的选项字符串。

所以这一步的核心作用可以概括为：

> **`mnt_fs_follow_optlist()` 不生成第二份选项，而是让 `cxt->fs` 与 `cxt` 共享同一个 `optlist`；`cxt->optlist` 负责 mount 过程中的结构化选项处理，`fs` 则可以在需要时从这份 optlist 得到字符串形式的选项。**

#### 3.4 `evaluate_permissions`：处理用户挂载权限

`evaluate_permissions()` 主要处理 `user`、`users`、`owner`、`group` 这些 userspace 选项所表达的普通用户挂载权限，并补充它们隐含的安全挂载标志：

| userspace 选项 | 隐含的 Linux flags |
| --- | --- |
| `user` / `users` | `MS_SECURE`：`noexec,nosuid,nodev` |
| `owner` / `group` | `MS_OWNERSECURE`：`nosuid,nodev` |

其中 `user/users` 隐含 `noexec,nosuid,nodev`，`owner/group` 隐含 `nosuid,nodev`。

##### 3.4.1 非 restricted 用户

对于非 restricted 用户，`user/users/owner/group` 不再用于判断调用者是否具有 mount 权限，因此不会因为 `owner/group` 再去检查 source 设备的属主或属组。

不过，这些选项隐含的安全挂载标志仍然需要处理，如下（后续说到处理隐含标志都类似）：

```c
if (!rc && (user_flags & MNT_MS_OWNER))
	rc = mnt_optlist_insert_flags(ol,
			MS_OWNERSECURE, cxt->map_linux,
			MNT_MS_OWNER, cxt->map_userspace);
```

##### 3.4.2 restricted 用户

对于 restricted 用户，首先要求当前请求已经成功应用了表项：

```c
if (!mnt_context_tab_applied(cxt))
    return -EPERM;
```

也就是说，普通用户不能仅凭自己提供的 source、target 和 options 获得 mount 权限，必须由匹配到的 fstab 条目提供授权。

在此基础上，`user/users` 与 `owner/group` 是两种不同的授权方式：前者直接允许普通用户挂载，后者则需要进一步检查 source 设备节点的属主或属组。

###### 3.4.2.1 `user/users`：直接授权

`user` 和 `users` 本身就表示 fstab 允许普通用户挂载，因此不需要检查 source 设备节点的 uid/gid。

它们同时隐含：

```text
MS_SECURE
    ↓
noexec,nosuid,nodev
```

两者的区别主要体现在后续卸载权限：

* `user`：允许普通用户挂载，并保留实际执行挂载的用户信息，供之后判断谁可以卸载；
* `users`：允许普通用户挂载，同时允许其他普通用户卸载。

因此 `user/users` 属于**直接授权**。

###### 3.4.2.2 `owner/group`：检查 source 设备

`owner` 和 `group` 属于**条件授权**：

```text
owner → 当前 uid 必须等于设备节点的 st_uid
group → 当前用户必须属于设备节点的 st_gid
```

因此 libmount 必须取得实际的 source 设备路径，再执行 `stat()`。

如果 fstab 直接使用设备路径：

```fstab
/dev/sdb1  /mnt  ext4  owner  0  0
```

`mnt_fs_get_srcpath(cxt->fs)` 可以直接得到：

```text
/dev/sdb1
```

如果使用 TAG：

```fstab
UUID=xxxx-xxxx  /mnt  ext4  owner  0  0
```

则需要调用 `mnt_resolve_spec()` 将其解析为实际设备路径：

```text
UUID=xxxx-xxxx
       ↓
mnt_resolve_spec()
       ↓
/dev/sdb1
       ↓
stat()
       ↓
st_uid / st_gid
```

`UUID=`、`LABEL=` 等设备标识的解析使用 libblkid 提供的能力：已有缓存或 `/dev/disk/by-*` 信息能够完成解析时直接使用，必要时还可以 probe 已存在的块设备，从文件系统 metadata 中识别 UUID、LABEL 等信息。

`owner/group` 还隐含：

```text
MS_OWNERSECURE
    ↓
nosuid,nodev
```

如果设备属主或属组检查失败，则当前用户没有通过这种方式获得挂载权限。

###### 3.4.2.3 `owner/group` 验证成功后转换为 `user`

`owner/group` 只负责决定当前用户**如何取得挂载资格**。一旦验证成功，libmount 不再维持独立的 “owner mount” 或 “group mount” 状态，而是统一转换为普通的 `user` mount：

```c
mnt_optlist_append_flags(... MNT_MS_USER ...);
```

则把这个结果写入 `optlist`，使后续流程能够继续按照标准的 `user` mount 处理。

这个转换也为后续记录实际挂载用户提供了基础。加入 optlist 的裸 `user` 后续可以变成：

```text
user=<实际挂载用户名>
```

并记录到 utab。之后执行 `umount` 时，libmount 可以利用这个信息判断卸载者是否是当初执行挂载的用户。

###### 3.4.2.4 已有 `user=<name>`

如果 optlist 中已经存在：

```text
user=bo
```

`evaluate_permissions()` 会：

```c
cxt->flags |= MNT_FL_SAVED_USER;
mnt_opt_set_external(opt, 1);
user_flags &= ~MNT_MS_USER;
```

这里有两个目的。

首先清除 `MNT_MS_USER`，表示 `user=bo` **不能作为当前 restricted 用户获得 mount 权限的依据**。它只是已经保存下来的挂载用户名，不等同于 fstab 中用于授权的裸 `user`。

其次将该 option 标记为 `external`。这样：

```text
user=bo
```

仍然保留在 optlist 中，后续可以继续作为**挂载用户信息**使用，如果后续调用 /sbin/mount.<type> helper，该选项仍可通过 -o 传给 helper，由具体文件系统自行解释；但 libmount 在生成内核 mount attributes 或 filesystem 配置时会跳过它，不会把 `user=bo` 当成需要传给内核的挂载选项。`MNT_FL_SAVED_USER` 用来记录：这个 `user=<name>` 是原本就带有 value 的 option。

这与 restricted 用户的裸 `user` 不同。裸 `user` 后面会在 `fix_optstr()` 中临时补成 `user=<当前用户名>`,见下一节。

#### 3.5 `fix_optstr`：修正挂载选项

前面的 `mnt_context_apply_fstab()`、`mnt_context_merge_mflags()` 等步骤已经把命令行、fstab 等来源的挂载选项汇总到了 `cxt->optlist` 中，`evaluate_permissions()` 也已经完成了 restricted user 相关的权限检查。
但此时 `optlist` 中的选项还不一定是后续挂载阶段真正需要的最终形式。例如：
```text
user
uid=bo
gid=developers
uid=useruid
X-mount.owner=bo
X-mount.idmap=...
rootcontext=@target
```
这里面混着两类东西：

1. 有些选项本身还需要补全或转换，例如 `user`、`uid=`、`gid=`；
2. 有些 `X-mount.*`、SELinux 等选项不是简单地转换成一个 mount flag，而是要求 libmount 在挂载流程的某个阶段执行额外操作。

`fix_optstr()` 正是在这里处理这两类事情：
```text
fix_optstr()
    │
    ├── 直接修正 optlist
    │      ├── user → user=<username>
    │      ├── uid=<username> → uid=<number>
    │      ├── gid=<groupname> → gid=<number>
    │      └── 清理不可用的 Smack 选项
    │
    └── mnt_context_call_hooks(MNT_STAGE_PREP_OPTIONS)
           │
           ├── owner hook
           ├── idmap hook
           ├── selinux hook
           └── ...
```
到这里，options 相关的主体处理基本完成。`fix_optstr()` 最后调用 `mnt_context_call_hooks(cxt, MNT_STAGE_PREP_OPTIONS)`，让注册在 `PREP_OPTIONS` 阶段的各个 hook 完成与 options 有关的扩展处理，作为这一阶段的收尾。

##### 3.5.1 为什么还要修正 `user`、`uid=` 和 `gid=`

先看 restricted mount 下的 `user`。
如果选项只有：
```text
user
```
`fix_optstr()` 会取得当前用户的信息，把它补成：
```text
user=<username>
```
原来的裸 `user` 只表示允许普通用户执行挂载；`fix_optstr()` 将其补成 `user=<当前用户名>`，记录实际执行挂载的用户，后续写入 **utab**，用于识别该挂载属于哪个普通用户，并据此支持该用户后续执行相应的卸载操作。

`uid=`、`gid=` 解决的是另一个问题。`uid=` / `gid=` 用来为文件系统中的文件指定或呈现 UID/GID，也就是决定这些文件在 VFS inode 中看到的 `i_uid`、`i_gid`。
对于 ext4 这类原生支持 Unix ownership 的文件系统，每个磁盘 inode 本身就保存自己的 UID/GID，读取 inode 时直接恢复到内存 inode 的 `i_uid`、`i_gid`，因此不需要通过普通的 `uid=` / `gid=` mount option 统一指定文件属主，ext4 也不支持这种用法。
而 FAT 等本身没有完整 Unix UID/GID 信息的文件系统，则需要通过 mount option 补充这类属性。例如挂载时指定：
```text
uid=1000,gid=1000
```
文件系统解析这些参数后，可以将其保存在自己的文件系统私有状态中；之后创建或读取 inode 时，再据此设置内存 inode 的 `i_uid`、`i_gid`。因此，同一次挂载下的文件就可以统一呈现为指定的 UID/GID。
libmount 在这里并不负责给 inode 赋值，它只负责把 `uid=` / `gid=` 整理成数字 ID，真正如何保存和使用这些值由具体文件系统实现。
`uid=` / `gid=` 的值可以分为三种情况，`fix_optstr()` 会分别处理：

1. **已经是数字 ID**
   例如：
   ```text
   uid=1000
   gid=1000
   ```
   这种形式已经可以直接使用，`fix_optstr()` 不再处理。

2. **特殊值 `useruid` / `usergid`**
   ```text
   uid=useruid  → getuid() → 当前调用者的 real UID
   gid=usergid  → getgid() → 当前调用者的 real GID
   ```
   `useruid`、`usergid` 是 libmount 认识的特殊值，表示使用实际调用 mount 的用户的 UID/GID。因此这里使用 `getuid()` / `getgid()`，而不是 effective UID/GID。

3. **用户名或组名**
   例如：
   ```text
   uid=bo
       ↓
   mnt_get_uid("bo")
       ↓
   getpwnam_r("bo", ...)
       ↓
   系统 passwd database
       ↓
   struct passwd::pw_uid
       ↓
   uid=1000
   ```
   `getpwnam_r()` 查询系统的 passwd database。在 glibc 系统上，实际数据可以通过 NSS 来自本地 `/etc/passwd`，也可以来自 LDAP、NIS、SSSD 等其他用户信息源。`gid=<groupname>` 同理，通过 group database 查询对应的数字 GID。

因此最终都是把 `uid=` / `gid=` 整理成数字形式：
```text
uid=1000        → uid=1000
uid=useruid     → uid=<当前 real UID>
uid=bo          → uid=<bo 对应的 UID>

gid=1000        → gid=1000
gid=usergid     → gid=<当前 real GID>
gid=developers  → gid=<developers 对应的 GID>
```
如果编译时启用了 Smack 支持，`fix_optstr()` 还会检查 `/sys/fs/smackfs`。系统实际没有 Smack 环境时，会从 `optlist` 中删除 `smackfsdef`、`smackfsfloor`、`smackfshat`、`smackfsroot`、`smackfstransmute` 等 Smack 选项。
这些直接的 option 修正完成以后，`fix_optstr()` 还没有结束。
它最后还有一句非常重要的调用：
```c
rc = mnt_context_call_hooks(cxt, MNT_STAGE_PREP_OPTIONS);
```
这就进入了 libmount 的 hook 机制。

##### 3.5.2 先认识 hook：把特殊挂载操作挂到正确的阶段

这是本文第一次遇到 hook，因此先解释这个机制。
有些选项要求在挂载前处理，有些要求 target 准备好以后处理，有些必须在 mount syscall 完成以后才能处理。
例如：
```text
X-mount.owner=
```
要求最终修改挂载点的 owner；
```text
X-mount.idmap=
```
要求对得到的 mount 执行 `mount_setattr(MOUNT_ATTR_IDMAP)`；
```text
rootcontext=@target
```
又必须等 target 已经准备好以后才能读取 target 的 SELinux context。
如果把这些特殊逻辑全部硬编码进 `mnt_context_prepare_mount()`、`mnt_context_do_mount()`，主流程很快就会变成大量：
```text
if (idmap) ...
if (selinux) ...
if (owner) ...
if (loop) ...
...
```
libmount 因此使用了 **hook** 机制。
可以把 hook 理解成：

> 某项功能先向 mount context 登记一个回调函数，并指定“这个回调应该在哪个 mount stage 执行”；主流程运行到该 stage 时，由 `mnt_context_call_hooks()` 统一调用这些回调。这些stage如下：

因此 mount 流程中存在一系列 stage，例如：
```text
PREP_OPTIONS
     │
     │ options 相关处理基本完成
     ▼
PREP_SOURCE
     │
     │ source 已准备到相应阶段
     ▼
PREP_TARGET
     │
     │ target 已准备到相应阶段
     ▼
PREP
     │
     │ 整个 mount prepare 阶段基本完成
     ▼
MOUNT_PRE
     │
     ▼
MOUNT
     │
     │ 执行实际的挂载操作
     ▼
MOUNT_POST
     │
     │ mount 已成功，可继续处理得到的 mount
     ▼
POST
        整次挂载事务后期处理
```
一个 stage 的 hook 在执行过程中，还可以为后面的 stage 注册新的 callback。
例如现在：
```c
mnt_context_call_hooks(cxt, MNT_STAGE_PREP_OPTIONS);
```
表示：

> 现在已经走到 `PREP_OPTIONS`，执行所有属于这个 stage 的 hook。

但其中某个 `PREP_OPTIONS` hook 执行以后，还可以再登记：
```text
MNT_STAGE_MOUNT_POST
```
或者：
```text
MNT_STAGE_POST
```
的回调。
于是就形成：
```text
现在
PREP_OPTIONS
     │
     ├── owner hook
     │      └── 登记 POST 回调
     │
     ├── idmap hook
     │      └── 登记 MOUNT_POST 回调
     │
     └── selinux hook
            └── 必要时登记 PREP_TARGET 回调

                     ↓ 后面的主流程继续运行 ↓

PREP_TARGET ───────────────→ 执行相应回调
MOUNT
MOUNT_POST ────────────────→ 执行相应回调
POST ──────────────────────→ 执行相应回调
```
**本文对 hook 的命名约定：** 下文为方便区分不同的 hook 功能，统一使用 `hook_owner`、`hook_idmap`、`hook_selinux` 等名称。这些名称取自对应的源码文件名（如 `hook_owner.c`、`hook_idmap.c`），本文用它们作为各 hook 功能模块的简称，并非 libmount 中实际存在的同名函数。实际注册到 hook 框架中的回调函数通常是各模块内部的 `hook_*()` 函数，并通过 `struct libmnt_hookset`、执行阶段（`MNT_STAGE_*`）等信息组织和调用。

下面分别看这个阶段几个重要的 hook。

##### 3.5.3 `hook_owner`：处理 `X-mount.owner/group/mode`

`X-mount.owner/group/mode` 要修改的是**挂载完成后的 mountpoint**，所以真正的 owner/group/mode 修改必须等到后面的 `POST` 阶段执行。
但 `POST` 阶段执行这些操作之前，需要先把用户写在 option 中的值解析成可以直接使用的数据。例如：
```text
X-mount.owner=bo
X-mount.group=developers
X-mount.mode=0755
```
在 `PREP_OPTIONS` 阶段，`hook_owner` 先完成这些转换：
```text
bo          → uid_t
developers  → gid_t
0755        → mode_t
```
并把得到的 UID、GID、mode 保存在 `hook_data` 中：
```text
PREP_OPTIONS
     │
     │ hook_owner
     ▼
解析 X-mount.owner/group/mode
     │
     ├── owner → uid
     ├── group → gid
     └── mode  → mode_t
     │
     ▼
保存到 hook_data
     │
     └── 注册 POST → hook_post()
                       │
                       ▼
                    …… mount ……
                       │
                       ▼
                     POST
                       │
                       ▼
             使用准备好的 uid/gid/mode
             修改最终 mountpoint
```
因此这里需要 `PREP_OPTIONS` 和 `POST` 两个阶段：**`PREP_OPTIONS` 负责把 option 解析成后续操作需要的数据并保存下来，`POST` 再使用这些数据完成真正的 mountpoint 属性修改。**

放到`POST`的原因：这些操作针对的是**最终挂载点路径上的文件系统对象**，而不是用于创建 mount 本身的参数。必须等挂载以及相关的 mount 后处理完成以后，再去修改最终 target 的属性。

##### 3.5.4 `hook_idmap`：`X-mount.idmap` 在这里准备，在 `MOUNT_POST` 真正应用

`hook_idmap` 同样首先在：
```text
MNT_STAGE_PREP_OPTIONS
```
执行。
它处理：
```text
X-mount.idmap=
```
这个阶段的主要任务不是立即对 mount 做 idmap，因为 mount 本身可能还没有准备出来，而是**先准备 user namespace，并登记后续操作**。
`X-mount.idmap` 可以直接指定已有 user namespace 的路径。例如 value 以 `/` 开头时，会打开该 user namespace：
```text
X-mount.idmap=/proc/<pid>/ns/user
                  │
                  ▼
             open_userns()
                  │
                  ▼
             userns_fd
```
另一种形式是直接给 ID mapping，例如：
```text
u:nsid:hostid:range
g:nsid:hostid:range
b:nsid:hostid:range
```
其中 `b` 同时建立 UID 和 GID mapping。
这种情况下，libmount 会创建新的 user namespace，并建立相应的 `uid_map` / `gid_map`，最终同样得到一个：
```text
userns_fd
```
准备好 user namespace 后，`hook_idmap` 保存得到的 `userns_fd`，并登记 `MNT_STAGE_MOUNT_POST` 回调。
这里有一个重要限制：**已经 attached 到 mount namespace 中的 mount 不能直接设置 ID mapping**。因此，`MOUNT_POST` 阶段执行的 `hook_mount_post()` 必须先取得一个 detached mount tree，再通过 `mount_setattr()` 设置 `MOUNT_ATTR_IDMAP`。

`hook_mount_post()` 根据是否已有可复用的 `api->fd_tree`，分为两种处理方式：

- **`is_private = 0`**：新 mount API 已经准备好 `api->fd_tree`，直接对它执行 `mount_setattr()`。本函数不负责最终挂载，后续由 `hook_attach_target()` 执行 `move_mount()`。
- **`is_private = 1`**：没有可复用的 `api->fd_tree`，本函数通过 `open_tree(target, OPEN_TREE_CLONE)` 克隆当前已经挂载到 target 的 mount，得到 detached tree。设置 idmap 后，先通过 `umount2(target, MNT_DETACH)` 卸载旧 mount，再通过 `move_mount()` 将 idmapped clone 挂载回 target。

整个流程如下：

```text
MOUNT_POST
    │
    ▼
hook_idmap::hook_mount_post()
    │
    ▼
是否已有 api->fd_tree？
    │
    ├── 有：is_private = 0
    │       └── 复用 api->fd_tree
    │
    └── 无：is_private = 1
            └── open_tree(target, OPEN_TREE_CLONE)
                    │
                    ▼
                detached clone
    │
    ▼
mount_setattr(fd_tree, MOUNT_ATTR_IDMAP, userns_fd)
    │
    ▼
is_private？
    │
    ├── 0：本函数结束
    │       └── 后续 hook_attach_target()
    │             ├── 替换已有 mount 时：
    │             │     umount2(target, MNT_DETACH)
    │             └── move_mount(fd_tree, target)
    │
    └── 1：本函数自行完成替换
            ├── umount2(target, MNT_DETACH)
            └── move_mount(fd_tree, target)
```

这里的 `clone` 并不是复制文件系统数据，而是克隆 mount tree，以便在尚未 attached 的状态下设置 ID mapping。

另外，**`is_private = 0` 时，`hook_attach_target()` 并不总是执行 `umount2()`**。只有满足以下条件时才卸载旧 mount：

```c
if (cxt->force_clone &&
    !api->is_new_fs &&
    !mnt_optlist_is_bind(cxt->optlist))
    umount2(target, MNT_DETACH);
```

即本次操作使用克隆的 mount tree 替换 target 上已有的 mount。新建 filesystem 挂载和 bind 挂载都不需要卸载 target 上原来的 mount。

##### 3.5.5 `hook_selinux`：先处理 context，必要时等 target 准备好

SELinux hook 也从：
```text
MNT_STAGE_PREP_OPTIONS
```
开始。
它关注：
```text
context=
fscontext=
defcontext=
rootcontext=
seclabel
```
这里首先判断 SELinux 当前是否可用。
如果 SELinux 没有启用，这些 option 对本次挂载没有意义，因此从 `optlist` 中删除。
对于正常 mount，SELinux context 还可能需要从用户看到的 translated context 转换成内核使用的 raw context，然后把转换后的值重新写回 option。
因此这里仍然属于：
```text
读取 option
      ↓
解释/转换
      ↓
修改 optlist
```
但：
```text
rootcontext=@target
```
是一个特殊情况。
`@target` 的意思不是一个固定的 SELinux context 字符串，而是：

> 使用 target 当前的 SELinux context。

问题在于当前还处于：
```text
fix_optstr()
    ↓
PREP_OPTIONS
```
而真正的 target 准备发生在后面：
```text
prepare_target()
    ↓
MNT_STAGE_PREP_TARGET
```
所以此时不能贸然读取最终 target 的 context。
SELinux hook 的处理方式正好体现了 hook stage 的用途：
```text
PREP_OPTIONS
     │
     │ 发现 rootcontext=@target
     │
     └── 登记 PREP_TARGET hook
                    │
                    ▼
              prepare_target()
                    │
                    ▼
               PREP_TARGET
                    │
                    ├── getfilecon_raw(target)
                    │
                    └── @target → 实际 raw context
```
也就是说：

> `PREP_OPTIONS` 发现“这件事以后需要做”，但当前条件还没准备好，于是把真正的工作挂到 `PREP_TARGET`。

此外，SELinux hook 会设置：
```text
cxt->has_selinux_opt = 1
```
后面的挂载实现可以据此知道本次挂载包含 SELinux context option。一个具体用途是：**当文件系统为 btrfs 且带有 SELinux context option 时，libmount 不使用 `fsopen()` / `fsconfig()` / `fsmount()` 这套新的 mount API，而是退回传统的 `mount(2)` 路径**，因为 btrfs 内核驱动尚不支持通过新 mount API 处理这一组合。

##### 3.5.6 `fix_optstr()` 完成后，`optlist` 到了什么状态

因此，`fix_optstr()` 的职责不能简单概括成“补全 option string”。
更准确地说，它完成了两层工作：
```text
                    fix_optstr()
                         │
          ┌──────────────┴──────────────┐
          │                             │
          ▼                             ▼
   基础 option 修正                PREP_OPTIONS hooks
          │                             │
   user → user=name               owner / idmap /
   uid=name → uid=N              selinux / ...
   gid=name → gid=N                    │
   清理 Smack option                    │
                                        ▼
                              修正特殊 option，或者
                              登记以后阶段的 callback
```
成功完成以后设置：
```text
MNT_FL_MOUNTOPTS_FIXED
```
此时可以把 `optlist` 理解为已经完成了**挂载选项准备阶段的第一轮最终整理**：

* 普通的 `user`、`uid=`、`gid=` 等选项已经规范化；
* 当前环境不适用的部分选项已经删除；
* `X-mount.owner/group/mode`、`X-mount.idmap`、SELinux 等特殊功能已经完成 `PREP_OPTIONS` 阶段的解析；
* 需要等到 `PREP_TARGET`、`MOUNT_POST`、`POST` 才能执行的工作，已经通过 hook 登记到相应阶段。

#### 3.6 `mnt_context_prepare_srcpath`：准备真正用于挂载的 source

`mnt_context_prepare_srcpath()` 负责把 `cxt->fs` 中的 source 解析、规范化成后续挂载可以使用的路径：
- `LABEL=`/`UUID=` → 经 cache/blkid 解析为 `/dev/...`
- 相对路径、symlink 等规范化

并在末尾调用：

```text
mnt_context_call_hooks(cxt, MNT_STAGE_PREP_SOURCE)
```

让需要进一步准备 source 的功能继续处理，例如 loop 和 dm-verity：

```text
用户指定的 source
        │
        ▼
mnt_context_prepare_srcpath()
        │
        ├── LABEL/UUID → /dev/...
        ├── 路径解析、规范化
        │
        ▼
   PREP_SOURCE hooks
        │
        ├── 普通文件 + loop
        │        ↓
        │    /dev/loopN
        │
        └── verity.*
                 ↓
           /dev/mapper/...
                 │
                 ▼
        后续真正用于 mount 的 source
```
##### 3.6.1 `hook_loopdev`：把普通文件变成 loop block device

例如：

```text
mount -o loop disk.img /mnt
```

用户给出的 source 是：

```text
disk.img
```

但镜像文件只是普通文件，而文件系统挂载通常需要操作块设备。因此 `hook_loopdev` 会为这个文件准备 loop device：

```text
disk.img
   │
   │ attach
   ▼
/dev/loopN
   │
   ▼
mnt_fs_set_source()
   │
   ▼
本次 mount 的 source 改为 /dev/loopN
```

这样后面的挂载流程看到的就不再是原来的 `disk.img`，而是准备好的 loop block device。

如果同时指定：

```text
offset=
sizelimit=
```

这些参数也是在建立 loop device 时使用。

其中 `offset=` 表示从 backing file 的哪个**字节偏移**开始映射。例如一个镜像文件前面还有其他数据，而真正需要挂载的文件系统从某个偏移位置开始，就可以让 loop device 从那里开始映射：

```text
backing file
┌─────────────────────────────────────┐
│ 前面的数据 │      filesystem        │
└────────────┴────────────────────────┘
             ↑
           offset
             │
             ▼
        /dev/loopN
```

它的含义与 `losetup -o` 的 offset 相同。`sizelimit=` 则可以进一步限制映射的长度。

这些参数在 loop device setup 时已经被消化，不需要再作为 filesystem mount option 传下去。

loop setup 成功后，如果 loop device 启用了 `LO_FLAGS_AUTOCLEAR`，或者本次挂载复用了已有的 loop device，libmount 会从 `optlist` 中删除 `loop=`。因此后续写入 utab 时，不会出现：

```text
OPTS=loop
```

`OPTS=loop` 主要用于表明该挂载的 loop device 需要由后续卸载流程处理。启用 autoclear 时，内核会在最后的相关引用释放后自动 detach；复用已有 loop device 时，该设备也不应由本次挂载负责清理。因此这两种情况下都不需要保留 `loop=`。

hook已经把 mount source 改成了 loop device，所以 utab 中记录的是最终 source，例如：

```text
SRC=/dev/loop3
```

也就是说：

```text
原始 source： /home/bo/disk.img
                    │
                    ▼
             loop setup
                    │
                    ▼
最终 source： /dev/loop3
                    │
                    ▼
utab：        SRC=/dev/loop3
```

##### 3.6.2 `hook_veritydev`：把 source 转换成 dm-verity device

`verity.*` 处理的是另一种 source 准备。

例如挂载时指定：

```text
verity.hashdevice=
verity.roothash=
verity.roothashfile=
verity.hashoffset=
verity.fecdevice=
...
```

这些不是交给 ext4、squashfs 等文件系统驱动解析的普通 filesystem option，而是 libmount 在 userspace 中准备 dm-verity device 所需要的参数。

其流程可以概括为：

原始 source
      +
verity.hashdevice=
verity.roothash=
...
      │
      ▼
libmount 的 verity hook
      │
      ▼
调用 libcryptsetup
      │
      ▼
配置并激活 dm-verity mapping
      │
      ▼
得到 /dev/mapper/<verity-device>
      │
      ▼
libmount 把 mount 的 source
替换成这个 dm-verity device

这里并不会在挂载时校验整个 filesystem。libmount 先根据 `verity.*` 参数建立只读的 dm-verity 映射设备，再将该设备作为真正的 source 挂载。挂载完成后，每次从该设备读取数据时，内核 dm-verity 才会利用预先建立的 hash tree 对读取的数据块进行完整性校验。

可以看到，verity 处理 source 的方式与 loop 其实很相似：

```text
loop:
普通文件
   ↓
准备 loop device
   ↓
/dev/loopN
   ↓
mount

verity:
原始块设备 + hash 信息
   ↓
准备 dm-verity device
   ↓
/dev/mapper/...
   ↓
mount
```

因此二者都放在 `MNT_STAGE_PREP_SOURCE` 很自然：这个阶段就是在真正挂载之前，把 source 准备成后续 mount 可以直接使用的形式。

`verity.*` 这些 setup 参数在 userspace map 中标有 `MNT_NOMTAB`，因为它们只用于建立 dm-verity device，不需要作为最终挂载记录中的 `OPTS=` 保存。设备准备完成以后，真正需要记录的是已经替换后的 source，例如：

```text
SRC=/dev/mapper/<verity-device>
```

另外，loop 和 verity 相关 userspace option 还使用了 `MNT_NOHLPS`，表示这些 option 由 libmount 自己的 userspace 逻辑处理，不作为相应 option 交给外部 mount helper。

#### 3.7 `mnt_context_guess_fstype`：确定挂载使用的文件系统类型

`mnt_context_guess_fstype()` 的作用是确定本次挂载应使用的 filesystem type。前面的 `mnt_context_prepare_srcpath()` 已经把 source 准备成实际可用的路径，例如 `/dev/sda1`、`/dev/loop0` 或 `/dev/mapper/...`；这一步则在没有明确类型时，根据这个最终 source 探测 filesystem type，并在探测成功后写入：

```c
cxt->fs->fstype
```

此时 `cxt->update` 尚未创建，因此这里只修改本次挂载使用的 `cxt->fs`。

##### 3.7.1 此函数的处理流程
函数首先处理几种不需要探测 filesystem type 的情况：

```text
bind / move / propagation-only
        ↓
fstype = "none"

remount（如果 fstype 没有指定）
        ↓
fstype = "none"
```

如果 `cxt->fs->fstype` 已经有明确值，例如：

```text
ext4
xfs
nfs
ext4,xfs
```

则直接保留，不再探测。

如果原来的类型是：

```text
auto
```

则先将其清空，再按照“未指定类型”处理。

如果设置了 `cxt->fstype_pattern`，这里同样不进行 source 探测，而是把类型选择留到后面的 `do_mount_by_pattern()`。

真正需要自动识别时，调用：

```c
mnt_context_guess_srcpath_fstype(cxt, &type)
```

对于存在的本地 source，通过 `mnt_get_fstype()` 获取设备的 filesystem `TYPE`：有 libmount cache 时优先利用 cache 中的 `TYPE` 信息；没有 cache 时则使用 libblkid 对设备进行 superblock probe，根据磁盘上的 filesystem 特征识别 `ext4`、`xfs`、`vfat` 等类型。

例如：

```text
/dev/loop0
    │
    ▼
mnt_get_fstype()
    │
    ▼
libblkid probe
    │
    │ 读取并识别 filesystem superblock
    ▼
"ext4"
    │
    ▼
cxt->fs->fstype = "ext4"
```

如果 source 本身不是可访问的本地路径，`mnt_context_guess_srcpath_fstype()` 还会识别两种典型的网络 source：

```text
server:/path   → nfs
//server/share → cifs
```

探测得到类型后，还会检查 `X-mount.auto-fstypes=`。如果该选项限制了自动探测允许使用的 filesystem type，而实际探测结果不在允许范围内，则返回 `MNT_ERR_NOFSTYPE`。

##### 3.7.2 后续的 fstype 处理

`mnt_context_guess_fstype()` 并不保证一定能探测出 filesystem type。源码明确允许探测结果为空，因为后面的挂载阶段还可以尝试其他类型。

挂载阶段，`mnt_context_do_mount()` 根据最终的 `cxt->fs->fstype` 选择不同路径：

| `cxt->fs->fstype`      | 后续处理                                              |
| ---------------------- | ------------------------------------------------- |
| 单一类型，如 `ext4`          | `do_mount(cxt, NULL)`，直接按该类型挂载                    |
| 逗号分隔的类型列表，如 `ext4,xfs` | `do_mount_by_types()`，按顺序逐个尝试                     |
| `NULL`                 | `do_mount_by_pattern()`，从 filesystem type 列表中逐个尝试 |

###### 3.7.2.1 `do_mount_by_types`

对于逗号列表：

```text
ext4,xfs,btrfs
      ↓
do_mount_by_types()
      ↓
ext4 → xfs → btrfs
```

每次尝试都会调用：

```c
do_mount(cxt, try_type)
```

此时 `try_type` 会临时写入 `cxt->fs->fstype`，并加入 `MS_SILENT`，避免每次失败尝试都产生普通 mount 的错误输出。失败后恢复原来的 `fstype`；某个类型挂载成功后则保留成功的类型。

列表中还允许出现：

```text
udf,iso9660,auto
```

遇到 `auto` 时不会继续把字符串 `"auto"` 直接交给内核，而是再次调用：

```c
mnt_context_guess_srcpath_fstype()
```

先根据 source 探测出实际类型，再用探测结果进行这一次挂载尝试。

###### 3.7.2.2 `do_mount_by_pattern`

如果 `cxt->fs->fstype == NULL`，则进入：

```c
do_mount_by_pattern(cxt, cxt->fstype_pattern)
```

`do_mount_by_pattern()` 再根据 `fstype_pattern` 分为三种情况。

1. 如果 `pattern` 是正向类型列表，例如：

   ```text
   ext4,xfs,btrfs
   ```

   则不读取 `/etc/filesystems` 和 `/proc/filesystems`，而是直接把这个 pattern 当作候选 filesystem type 列表：

   ```c
   do_mount_by_types(cxt, pattern)
   ```

   按 `ext4 → xfs → btrfs` 的顺序逐个尝试。

2. 如果 `pattern` 是以 `no` 开头的否定 pattern，例如：

   ```text
   noext4,xfs
   ```

   则调用 `mnt_get_filesystems()` 从 `/etc/filesystems`、`/proc/filesystems` 获取候选 filesystem type，同时排除 pattern 指定的 `ext4`、`xfs`，再对剩余类型逐个调用 `do_mount()`。

3. 如果 `pattern == NULL`，同样通过 `mnt_get_filesystems()` 获取 `/etc/filesystems`、`/proc/filesystems` 中的候选类型，只是不进行 pattern 过滤，然后逐个尝试挂载。

因此，`do_mount_by_pattern()` 并不一定会扫描 `/etc/filesystems`、`/proc/filesystems`：

```text
pattern = "ext4,xfs"
        ↓
do_mount_by_types()
        ↓
直接逐个尝试指定类型

pattern = "noext4,xfs"
        ↓
/etc/filesystems、/proc/filesystems
        ↓
排除 ext4、xfs
        ↓
逐个尝试剩余类型

pattern = NULL
        ↓
/etc/filesystems、/proc/filesystems
        ↓
逐个尝试其中的候选类型
```

此外，`prepare_mount()` 后面会调用（见 3.9 小节）：

```c
mnt_context_prepare_helper(cxt, "mount", NULL)
```

如果 `fstype` 是单一类型，它可以据此查找相应的 `mount.<type>` helper；但如果 `fstype` 仍是带逗号的类型列表，`prepare_helper()` 不会直接查找 helper。等挂载阶段的`mnt_context_do_mount()` 里`do_mount_by_types()` 逐项取出具体的 `try_type` 后，`do_mount(cxt, try_type)` 才会针对该具体类型再次调用 `mnt_context_prepare_helper()`。

#### 3.8 `prepare_target`：准备挂载目标

前面的 `mnt_context_prepare_srcpath()` 解决的是“**从哪里挂载**”，`prepare_target()` 处理的则是“**挂到哪里**”：它准备 `cxt->fs->target`，使 target 成为后续 mount 操作可以直接使用的最终路径。

整体流程为：

```text
cxt->fs->target
      │
      ├── 应用 target prefix
      │
      ├── canonicalize target
      │
      ▼
PREP_TARGET hooks
      │
      ├── X-mount.mkdir
      ├── SELinux rootcontext=@target（在 `__mkdir` 之后）
      └── X-mount.subdir
```

##### 3.8.1 target 路径处理

首先取得：

```c
tgt = mnt_fs_get_target(cxt->fs);
```

如果设置了 target prefix，则先把 prefix 加到 target 前面。

例如：

```text
target        = /etc
target-prefix = /mnt/root
```

最终得到：

```text
/mnt/root/etc
```

接着通过 cache 和 `mnt_resolve_path()` 对 target 做 canonicalize：

```text
/mnt/a/../data
        ↓
mnt_resolve_path()
        ↓
/mnt/data
```

这里不仅会处理 `.`、`..`，也会解析实际路径中的符号链接等路径关系。若解析后的路径发生变化，则重新写入：

```c
cxt->fs->target
```

target 路径准备完成后，进入：

```c
mnt_context_call_hooks(cxt, MNT_STAGE_PREP_TARGET);
```

这一阶段主要处理那些**必须等 target 确定以后才能完成**的 mount option。

##### 3.8.2 `hook_subdir`：处理 `X-mount.subdir`

`X-mount.subdir=<path>` 用于只挂载 filesystem 中的某个子目录，而不是整个 filesystem root。

例如 filesystem 内容为：

```text
/
├── etc/
├── data/
└── foo/
    └── bar/
        ├── 1.txt
        └── 2.txt
```

执行：

```bash
mount /dev/sda1 /mnt -o X-mount.subdir=foo/bar
```

最终：

```text
filesystem 中的 /foo/bar  →  /mnt

/mnt/
├── 1.txt
└── 2.txt
```

也就是说，最终 `/mnt` 的根对应 filesystem 内部的 `/foo/bar`。

在 `PREP_TARGET` 阶段，`hook_subdir` 解析 `X-mount.subdir`，保存 `subdir` 和原来的 target，并登记后续 `MOUNT_PRE`、`MOUNT_POST` 阶段的处理；此时还不会真正挂载 subdir。

这里的关键问题是：新 mount API 可以先创建一个 **detached mount tree**，但内核此时还不能直接以这个 detached tree 为起点解析并取得其中的 `foo/bar`。因此不能直接完成：

```text
detached filesystem
        │
        └── foo/bar
               ↓
             /mnt
```

libmount 的处理方法是：**先创建一个临时 mount namespace，并将整个 filesystem 临时挂载到其中的临时 target，使 filesystem 内部的 subdir 可以通过正常路径取得；随后取得该 subdir 对应的 mount tree，切回原 mount namespace 将它 attach 到真正的 target；最后回到临时 namespace，清理临时挂载并恢复原 mount namespace。**

整体过程为：

```text
原 target = /mnt
subdir     = foo/bar
        │
        ▼
PREP_TARGET
保存 subdir
登记 MOUNT_PRE
        │
        ▼
MOUNT_PRE
保存原 target /mnt
保存当前 mount namespace
        │
        ▼
unshare(CLONE_NEWNS)
创建并进入临时 mount namespace
        │
        ▼
将临时挂载区域设为 private
保存临时 namespace
        │
        ▼
创建临时 target
/run/mount/tmptgt
        │
        ▼
将 cxt->fs->target
临时改为 /run/mount/tmptgt
登记 MOUNT_POST
        │
        ▼
MOUNT
将整个 filesystem
挂载到 /run/mount/tmptgt
        │
        ▼
MOUNT_POST
恢复 cxt->fs->target = /mnt
        │
        ▼
open_tree(fd_tree, "foo/bar",
          OPEN_TREE_CLONE)
取得 subdir 对应的 mount tree
        │
        ▼
setns(old_ns_fd)
切回原 mount namespace
        │
        ▼
move_mount(subdir tree → /mnt)
        │
        ▼
setns(new_ns_fd)
切回临时 mount namespace
        │
        ▼
umount(/run/mount/tmptgt)
        │
        ▼
tmptgt_cleanup()
清理临时 target 和 namespace 资源
恢复原 mount namespace
```

因此，这套处理的核心就是：

```text
detached filesystem
        ↓
暂时无法直接取得其中的 subdir
        ↓
先临时 attach 整个 filesystem
        ↓
通过正常路径取得 subdir
        ↓
将 subdir 放到真正 target
```

较新的内核与更新版 util-linux 可以简化这条路径：直接从 detached mount tree 取得 `foo/bar` 并 attach 到最终 target。本文基于 util-linux 2.39.3，其中无论新/旧 mount API，subdir 仍先临时挂到 `/run/mount/tmptgt`，再 `open_tree()` 取得子目录。

##### 3.8.3 `hook_mkdir`：处理 `X-mount.mkdir`

普通 mount 要求 target 已经存在，例如：

```bash
mount /dev/sda1 /mnt/data
```

指定：

```text
X-mount.mkdir
```

或：

```text
x-mount.mkdir
```

后，libmount 可以在 `PREP_TARGET` 阶段自动创建 target。

例如：

```bash
mount /dev/sda1 /mnt/data -o X-mount.mkdir
```

如果：

```text
/mnt/data
```

不存在，则 `hook_mkdir` 调用：

```c
ul_mkdir_p(tgt, mode);
```

创建完整目录路径。

默认 mode 为：

```text
0755
```

也可以显式指定，例如：

```text
X-mount.mkdir=0700
```

目录创建成功后，libmount 会再次通过 `mnt_resolve_path()` 解析 target，并在路径发生变化时更新：

```c
cxt->fs->target
```

该功能只允许 root 或非 restricted 的 mount 使用；restricted mount 会返回 `EPERM`。

##### 3.8.4 `hook_selinux_target`：解析 `rootcontext=@target`

SELinux 的：

```text
rootcontext=@target
```

含义是：

> 使用当前 target mountpoint 的 SELinux context，作为新 filesystem root inode 的 `rootcontext`。

这里有一个时间顺序问题。

SELinux hook 最早在：

```text
MNT_STAGE_PREP_OPTIONS
```

处理 options，但此时 target 可能还不存在，例如同时指定：

```text
X-mount.mkdir,rootcontext=@target
```

因此 `PREP_OPTIONS` 阶段不能立即执行：

```c
getfilecon_raw(target, ...)
```

遇到 `rootcontext=@target` 时，SELinux hook 会把真正的解析操作延迟到：

```text
MNT_STAGE_PREP_TARGET
```

而且这个 callback 被安排在 `__mkdir` hook **之后**。

因此顺序是：

```text
PREP_OPTIONS
      │
      │ 发现 rootcontext=@target
      │
      └── 登记 PREP_TARGET callback
                    │
                    ▼
             prepare_target()
                    │
                    ▼
             PREP_TARGET
                    │
             X-mount.mkdir
                    │
              target 已存在
                    │
                    ▼
          hook_selinux_target()
                    │
                    ├── getfilecon_raw(target)
                    │
                    ▼
          得到 target 当前 SELinux context
                    │
                    ▼
      将 rootcontext=@target 中的 @target
             替换为实际 context
```

因此这里把 SELinux 处理放在 `PREP_TARGET`，不是因为所有 SELinux option 都在这个阶段处理，而是因为 **`rootcontext=@target` 必须等 target 的最终路径确定、并且必要时已经由 `X-mount.mkdir` 创建之后，才能取得 target 的实际 SELinux context。**

#### 3.9 `mnt_context_prepare_helper`：选择外部 mount helper

前面的步骤已经基本确定了本次挂载的 source、target、fstype 和 options。接下来还有一个问题需要确定：

> **这次挂载由 libmount 自己调用内核 mount 接口完成，还是交给文件系统专用的外部程序 `mount.<type>` 完成**

对于 ext4、xfs 等普通文件系统，通常不需要专门的用户态挂载程序。libmount 可以根据已经整理好的 fstype 和 options，继续进入自己的挂载流程，最终通过 `mount(2)`，或者新 mount API 的 `fsopen()`、`fsconfig()`、`fsmount()`、`move_mount()` 等接口请求内核完成挂载。

但有些文件系统在真正调用内核接口之前，还需要由专门的用户态程序完成协议协商、参数处理或其他文件系统特有的准备工作。典型的就是 NFS、CIFS、FUSE 等。它们通常提供：

```text
/sbin/mount.nfs
/sbin/mount.cifs
/sbin/mount.fuse
```

这样的程序。此时 libmount 不再自己完成实际挂载，而是执行相应的 `mount.<type>`，将 source、target、fstype 和经过筛选的 options 交给它，由这个 helper 负责完成后续挂载。

因此，从这里开始，后面的实际挂载存在两条路线：

```text
                         prepare_mount
                              │
                              ▼
                mnt_context_prepare_helper()
                              │
                  查找 mount.<type>
                              │
                 ┌────────────┴────────────┐
                 │                         │
             找到 helper                没有 helper
                 │                         │
                 ▼                         ▼
        cxt->helper = 路径             cxt->helper = NULL
                 │                         │
                 ▼                         ▼
             do_mount()                do_mount()
                 │                         │
                 ▼                         ▼
           exec_helper()          libmount 自己执行挂载
                 │                         │
                 ▼                         ▼
          mount.nfs 等             mount(2)
                                   或新 mount API
```

例如：

```text
mount -t ext4 /dev/sda1 /mnt
```

如果系统中没有 `mount.ext4`，`cxt->helper` 保持为空，后面继续走 libmount 自己的挂载路径。

而：

```text
mount -t nfs server:/share /mnt
```

如果找到：

```text
/sbin/mount.nfs
```

则记录：

```text
cxt->helper = "/sbin/mount.nfs"
```

后面的 `do_mount()` 看到 `cxt->helper` 已设置，就转入 `exec_helper()` 执行 `mount.nfs`，而不再由 libmount 自己完成这次实际挂载。

所以 `mnt_context_prepare_helper()` 本身并不执行 helper，也不执行任何内核挂载操作。它在 `prepare_mount` 阶段所做的是一次**执行路径选择**：

```c
rc = mnt_context_prepare_helper(cxt, "mount", NULL);
```

根据已经确定的 fstype 查找对应的 `mount.<type>`；找到后把路径保存到 `cxt->helper`，供后面的 `do_mount()` 使用。

本步对 context 最主要的变化因此只有：

```text
cxt->helper:
    NULL
      ↓
    "/sbin/mount.<type>"      // 找到 helper 时
```

##### 3.9.1 根据 fstype 查找 `mount.<type>`

普通情况下，`mnt_context_prepare_helper()` 使用当前文件系统类型：

```c
type = mnt_fs_get_fstype(cxt->fs);
```

例如：

```text
cxt->fs->fstype = "nfs"
```

则在 `FS_SEARCH_PATH` 指定的目录中寻找：

```text
mount.nfs
```

典型结果可能是：

```text
/sbin/mount.nfs
```

找到后保存为：

```text
cxt->helper = "/sbin/mount.nfs"
```

这样，后面的 `do_mount()` 就知道本次挂载应交给外部 helper，而不是由 libmount 自己调用内核 mount 接口。

有几种情况不会选择 helper：

| fstype / 状态              | 处理                                   |
| ------------------------ | ------------------------------------ |
| `ext4`，且不存在 `mount.ext4` | `cxt->helper` 保持为空，后面由 libmount 直接挂载 |
| `nfs`，找到 `mount.nfs`     | 设置 `cxt->helper=/sbin/mount.nfs`     |
| fstype 为 `none`          | 不查找 helper                           |
| fstype 中含 `,`            | 此处不按整个字符串查 helper                    |
| `mount -i` 禁用了 helper    | 不查找                                  |
| type 中存在危险的 `/..` 路径形式   | 拒绝查找                                 |
| swap 条目                  | 不使用 mount helper                     |

其中 fstype 含逗号时尤其需要区分。例如：

```text
-t udf,iso9660
```

此时：

```text
fstype = "udf,iso9660"
```

不能去寻找：

```text
mount.udf,iso9660
```

因为这不是一个真正的文件系统类型。

后面的 `do_mount_by_types()` 会拆开这个列表：

```text
udf
iso9660
```

然后分别调用：

```c
do_mount(cxt, "udf");
do_mount(cxt, "iso9660");
```

而 `do_mount()` 在收到 `try_type` 时，又会重新调用：

```c
mnt_context_prepare_helper(cxt, "mount", try_type);
```

所以 helper 的判断最终是针对**当前正在尝试的单个 fstype**完成的。

##### 3.9.2 `.subtype`：兼容基础类型的 helper

FUSE 文件系统的 fstype 可以采用 `type.subtype` 形式，例如：

```text
fuse.sshfs
fuse.rclone
```

查找 helper 时，libmount 首先按完整 fstype 查找。以：

```text
fstype = fuse.sshfs
```

为例，先查找：

```text
mount.fuse.sshfs
```

如果不存在，并且 fstype 中含有 `.`，则去掉最后的 `.subtype`，再查找基础类型对应的 helper：

```text
mount.fuse.sshfs
        ↓ 不存在
mount.fuse
        ↓
cxt->helper = "/sbin/mount.fuse"
```

这里去掉 `.sshfs` **只是为了匹配 helper 的文件名**，后面 `exec_helper()` 执行 `mount.fuse` 时，由于 helper 名称只对应基础类型 `fuse`，还会通过 `-t` 把完整的 fstype 传给它：

```text
/sbin/mount.fuse <source> <target> ... -t fuse.sshfs
```

因此这个机制可以概括为：

```text
fuse.sshfs
    │
    ├── helper 名称：mount.fuse
    │
    └── 完整类型：   -t fuse.sshfs
```

也就是说，`.subtype` 可以不体现在 helper 的文件名中，但不会丢失；它仍作为完整 fstype 传给基础类型的 helper。

##### 3.9.3 `do_mount`：进入 helper 路径

`mnt_context_prepare_helper()` 只是找到 helper 并设置 `cxt->helper`，真正的分流发生在后面挂载阶段的 `do_mount()`。

如果当前正在试探某个具体 fstype，`do_mount()` 会先针对这个 `try_type` 再准备一次 helper：

```c
if (try_type) {
        rc = mnt_context_prepare_helper(cxt, "mount", try_type);
        if (rc)
                return rc;
}
```

随后检查：

```c
if (cxt->helper)
        return exec_helper(cxt);
```

因此这里形成两条不同的实际挂载路径：

```text
                         do_mount()
                             │
              ┌──────────────┴──────────────┐
              │                             │
        cxt->helper != NULL           cxt->helper == NULL
              │                             │
              ▼                             ▼
        exec_helper()                libmount 自己挂载
              │                             │
              ▼                             ▼
       /sbin/mount.<type>        hook_create_mount 等
                                            │
                                            ▼
                                mount(2) / 新 mount API
```

例如：

```bash
mount -t nfs server:/share /mnt
```

如果前面找到：

```text
cxt->helper = /sbin/mount.nfs
```

那么 `do_mount()` 到这里直接：

```text
exec_helper()
    ↓
/sbin/mount.nfs server:/share /mnt ...
```

并从这条路径返回。

而：

```bash
mount -t ext4 /dev/sda1 /mnt
```

如果没有 `mount.ext4`，则：

```text
cxt->helper = NULL
```

`do_mount()` 继续执行 libmount 自己的挂载代码，最终由 libmount 调用内核 mount 接口。

所以 helper 不是在 libmount 自己挂载之前额外执行的一步，而是**替代了 `do_mount()` 中 libmount 自己完成实际挂载的那条路径**。

##### 3.9.4 `exec_helper`：将 context 中的参数重新传给 helper

进入 helper 路径后，真正执行挂载的程序从 libmount 变成了独立的 `mount.<type>`。前面 `mount` 已经解析过的参数保存在 `struct libmnt_context` 和 `optlist` 中，而新的 helper 程序无法直接读取这些数据，因此 `exec_helper()` 需要重新构造 helper 的命令行参数。

source 和 target 直接从当前 `cxt->fs` 取得：

```c
args[i++] = cxt->helper;
args[i++] = mnt_fs_get_srcpath(cxt->fs);
args[i++] = mnt_fs_get_target(cxt->fs);
```

例如：

```text
cxt->helper = /sbin/mount.nfs
source      = server:/share
target      = /mnt
```

形成：

```text
/sbin/mount.nfs server:/share /mnt ...
```

source 和 target 在这里没有特殊转换。真正需要注意的是后面的三类参数。

**第一类是已经解析成 context 状态的独立参数。**

例如用户执行：

```text
mount -n -v -t nfs server:/share /mnt
```

前面解析命令行后，`-n` 和 `-v` 已经成为 `cxt` 中的状态。现在实际挂载改由 `mount.nfs` 完成，`exec_helper()` 再根据这些状态重新生成 helper 的参数：

```c
if (mnt_context_is_nomtab(cxt))
        args[i++] = "-n";

if (mnt_context_is_verbose(cxt))
        args[i++] = "-v";
```

因此发生的是：

```text
原始 mount 命令：

mount -n -v -t nfs server:/share /mnt
       │  │
       │  └──────→ cxt: verbose
       └─────────→ cxt: nomtab

                 ...

发现 mount.nfs
       │
       ▼
exec_helper()
       │
       ├── nomtab  → "-n"
       └── verbose → "-v"
       │
       ▼
/sbin/mount.nfs server:/share /mnt -n -v ...
```

其中：

* `-n`：不进行相应的用户空间 mount table 更新；
* `-v`：verbose，让 helper 输出更详细的执行信息。

同一段 helper 执行代码中还能看到：

```c
if (mnt_context_is_lazy(cxt))
        args[i++] = "-l";

if (mnt_context_is_force(cxt))
        args[i++] = "-f";

if (mnt_context_is_rdonly_umount(cxt))
        args[i++] = "-r";
```

其中 `-l`（lazy unmount）、`-f`（force unmount）和 `-r`（unmount 失败后尝试 remount read-only）主要服务于 `umount.<type>`，不是这里 `mount.<type>` 路径需要展开的内容。

**第二类是从当前 cxt->optlist 重新生成 helper 的 -o 参数。**

`-o` 与 `-n/-v` 不同，它不是根据一个 context 状态简单恢复，首先，对 `cxt->optlist` 进行 user option 整理，如果存在 user option，但没有 `MNT_FL_SAVED_USER`，则清除这个 option 当前的 value（可以参考 3.4.2.4 小节有 `MNT_FL_SAVED_USER` 的来源）。例如：

user=bo
   ↓
mnt_opt_set_value(opt, NULL)
   ↓
user

已经整理好的 `cxt->optlist` **重新生成一份专门交给 helper 的 option string**：

```c
mnt_optlist_get_optstr(ol, &o, NULL, MNT_OL_FLTR_HELPERS);
```

也就是带有 `MNT_NOHLPS` 属性的选项，生成 helper option string 时会被排除。

例如：

```text
bind
move
shared
private
```
而文件系统需要处理的选项则继续保留。例如 NFS：

```text
vers=4.2
proto=tcp
```

可以生成：

```text
-o vers=4.2,proto=tcp
```

传给 `mount.nfs`。

**第三类是 target namespace。**

如果本次挂载指定了目标 mount namespace，前面的处理已经将该 namespace 保存在 `cxt->ns_tgt` 中。libmount 自己执行挂载时可以直接使用这个 namespace 状态；但 helper 是随后执行的独立程序，不能直接访问 `struct libmnt_context`，因此需要将 target namespace 转换成 helper 可以重新取得的形式。

`exec_helper()` 为此构造：

```text
-N /proc/<pid>/fd/<fd>
```

其中 `<fd>` 是 libmount 已经打开的 target namespace fd，`/proc/<pid>/fd/<fd>` 则把这个 fd 表示成一个可重新打开的 procfs 路径。

例如 libmount 中保存的 target namespace fd 为 `7`，当前进程 PID 为 `1234`：

```text
cxt->ns_tgt
    │
    └── fd = 7
          │
          ▼
/proc/1234/fd/7
          │
          ▼
args[i++] = "-N";
args[i++] = "/proc/1234/fd/7";
```

最终 helper 的 argv 中就包含：

```text
mount.<type> ... -N /proc/1234/fd/7
```

helper 根据 `-N` 重新取得这个 target namespace，并在相应 namespace 中完成挂载。

因此，`exec_helper()` 中需要特别处理的参数主要有三类：

```text
                    libmount 已准备好的 context
                               │
             ┌─────────────────┼──────────────────┐
             │                 │                  │
      独立 context 状态       optlist        target namespace
             │                 │                  │
      nomtab / verbose    处理 user value          namespace fd
             │                 │                  │
             │          MNT_OL_FLTR_HELPERS        │
             │                 │                  │
             ▼                 ▼                  ▼
          -n / -v      -o <filtered-options>  /proc/<pid>/fd/<fd>
             │                 │                  │
             │                 │                  ▼
             │                 │             -N <ns-path>
             │                 │                  │
             └─────────────────┼──────────────────┘
                               ▼
                      mount.<type> argv
```

##### 3.9.5 helper 子进程：放弃 mount 程序的额外权限

`exec_helper()` fork 出子进程后，在真正执行：

```c
execv(cxt->helper, args);
```

之前还会处理权限和 namespace。

其中权限方面会调用：

```c
drop_permissions();
```

也就是说，外部 `mount.<type>` 不直接继承 `mount` 程序可能由于 setuid 等机制获得的额外有效权限，而是在 exec helper 前将这些权限放弃，按调用者身份执行 helper。

随后还会：

```c
mnt_context_switch_origin_ns(cxt);
```

切回 origin namespace，再执行：

```c
execv(cxt->helper, args);
```

这里和上一节的 target namespace 要分开：

```text
exec 前：
    mount 子进程切回 origin namespace

target namespace：
    通过 -N /proc/<pid>/fd/<fd>
    告诉 helper 本次挂载的目标 namespace
```

也就是说，libmount **不是先进入 target namespace，再在里面 exec `mount.<type>`**；它回到 origin namespace 执行 helper，而需要操作的 target namespace则作为 `-N` 参数交给 helper。

##### 3.9.6 helper 与后续 utab 更新

`exec_helper()` 返回后，流程回到 `mnt_context_do_mount()`去做挂载，随后外层 `mnt_context_mount()` 调用去更新utab（见第6节）：

```c
mnt_context_update_tabs(cxt);
```

`cxt->update` 已在执行挂载之前由 `mnt_context_prepare_update()`（见第4节）准备完成。

普通 mount 路径中，libmount 根据 `syscall_status` 确认挂载成功后，直接调用：

```c
mnt_update_table(cxt->update, cxt->lock);
```

更新 utab。

helper 路径有所不同。helper 成功退出后，`mnt_context_update_tabs()` 会先调用：

```c
mnt_update_already_done(cxt->update, cxt->lock);
```

检查 external helper 是否已经完成了相同的 utab 更新。已经更新则直接跳过；尚未更新才由外层 libmount 调用 `mnt_update_table()` 补写。

```text
普通 mount                         helper mount

mount syscall                     mount.<type>
     │                                │
     ▼                                ▼
syscall_status == 0               helper_status == 0
     │                                │
     │                                ▼
     │                       mnt_update_already_done()
     │                           │              │
     │                         已更新          未更新
     │                           │              │
     │                           ▼              │
     │                          跳过            │
     │                                          │
     └──────────────────┐          ┌────────────┘
                        ▼          ▼
                    mnt_update_table()
                           │
                           ▼
                       更新 utab
```

因此，helper 路径的特殊之处是增加了一次重复更新检查，避免 helper 已经写过 utab 后，外层 libmount 再次写入。

#### 3.10 `onlyonce` 检查与 `MNT_STAGE_PREP`

执行到这里，source、target、fstype、options 和 helper 都已经确定。`prepare_mount()` 最后还要完成两件事：

1. onlyonce 检查：如果设置了 onlyonce flag，就要确认本次文件系统是否已经挂载，`mount -a` 默认启用 onlyonce（已在挂载则跳过）；单次 mount 一般不设。
2. 调用 `MNT_STAGE_PREP` hooks，向 hook 链表**注册**后续 `MOUNT` / `MOUNT_POST` 回调。下面先说新的 mount API 和 legacy 的 API `mount(2)` 的区别。

##### 3.10.1 新 mount API 与 legacy `mount(2)` 的区别

之前有提到过新的 mount API 和 legacy 的 API，这一节详细介绍这两者的区别，
下面这些：

```text
fsopen
fsconfig
fsmount
open_tree
fspick
mount_setattr
move_mount
```

都属于新的 fd-based mount API。

它的特点是把过去一个 `mount(2)` 中混在一起的工作拆成多个步骤。

以普通新文件系统为例：

```text
新的 API

fsopen(type)
    ↓
创建 filesystem context

fsconfig(...)
    ↓
配置 source / filesystem options

fsconfig(CREATE)
    ↓
创建 filesystem

fsmount()
    ↓
得到 detached mount tree

mount_setattr()
    ↓
设置 VFS mount attributes

move_mount()
    ↓
attach 到 target
```

传统 API 则主要把这些信息一次交给：

```c
mount(source, target, fstype, flags, data);
```

即：

```text
旧 API

source
target
fstype
MS_* flags
filesystem-specific data
   │
   ▼
mount(2)
   │
   ▼
内核完成创建/修改并直接作用到 target
```

因此新 API 的一个重要结构变化是：

> **filesystem 的创建/重配置、mount tree 的取得、VFS 属性修改以及最终 attach 被拆成了独立操作。**

和legacy路径的区别在于：注册到这些 stage 中的具体操作不同。

新 API 路径大致为：

```text
PREP
  │
  ├── fsopen() / open_tree()
  └── 注册后续 hooks
            │
            ▼
MOUNT
  ├── fsconfig() / fsmount()
  ├── fsconfig(RECONFIGURE)
  └── mount_setattr()
            │
            ▼
MOUNT_POST
  ├── move_mount()
  └── propagation 设置
```

传统路径则以 `mount(2)` 为核心。主体挂载在 MOUNT 阶段执行：

```c
mount(src, target, type, flags, options);
```

但传统 `mount(2)` 的限制决定了某些操作不能全部在这一次调用中完成，因此 `hook_mount_legacy` 还会根据挂载类型注册额外的 MOUNT_POST hook。

**第一种是 bind mount 后修改 VFS flags。**

例如：

```text
mount -o bind,ro /foo /bar
```

传统 API 不能在第一次 `MS_BIND` 调用中同时设置新 bind mount 的 `ro` 等 VFS 属性。因此，`hook_mount_legacy` 在 PREP 阶段通过 `prepare_bindremount()` 注册 `hook_bindremount()`，将操作分为两步：

```text
MOUNT
  │
  └── mount(source, target, NULL,
            MS_BIND, NULL)
             │
             ▼
        创建 bind mount
             │
             ▼
MOUNT_POST
  │
  └── hook_bindremount()
             │
             ▼
      mount("none", target, NULL,
            MS_REMOUNT | MS_BIND | MS_RDONLY,
            NULL)
             │
             ▼
      将新 bind mount 设置为只读
```

第二次 `mount(2)` 中，`MS_REMOUNT | MS_BIND` 表示对已有 bind mount 执行 **VFS 属性限定的 remount**，`MS_RDONLY` 则指定将其设置为只读。

因此，这次调用不会重新创建 bind mount，也不会修改 filesystem/superblock 的只读状态，只修改刚创建的 bind mount 自身的 VFS 属性。

**第二种是 propagation 设置。**

`MS_SHARED`、`MS_PRIVATE`、`MS_SLAVE`、`MS_UNBINDABLE` 等 propagation 操作也不会混在主体挂载中一起提交，而是在 MOUNT_POST 阶段单独执行相应的 `mount(2)`：

```text
MOUNT
  │
  └── mount(source, target, type,
            <主体 flags>, data)
             │
             ▼
       主体挂载完成
             │
             ▼
MOUNT_POST
  │
  └── propagation hook
             │
             ▼
      mount("none", target, NULL,
            <propagation flags>,
            NULL)
```
因此，新旧两套 API 都使用 libmount 的 PREP / MOUNT / MOUNT_POST 框架，只是各阶段执行的内核操作不同：

| 阶段         | 新 mount API                                        | legacy `mount(2)`                                        |
| ---------- | -------------------------------------------------- | -------------------------------------------------------- |
| PREP       | `fsopen()` / `open_tree()`，准备 fd 并注册 hooks         | 准备主体 `mount(2)`，并按需要注册 bind-remount、propagation 等后续 hook |
| MOUNT      | `fsconfig`、`fsmount`、`mount_setattr`、reconfigure 等 | 主体 `mount(source, target, type, flags, data)`            |
| MOUNT_POST | `move_mount()` attach、propagation 等                | bind-remount、propagation 等需要在主体挂载之后执行的附加 `mount(2)`      |

##### 3.10.2 `onlyonce`：判断同一项挂载是否已经存在

启用 `onlyonce` 时：

```c
if (!rc && mnt_context_is_onlyonce(cxt)) {
        rc = mnt_context_is_fs_mounted(cxt, cxt->fs, &mounted);
        if (mounted == 1)
                rc = -MNT_ERR_ONLYONCE;
}
```

`mnt_context_is_fs_mounted()` 取得当前 target namespace 的 mountinfo，然后调用：

```c
__mnt_table_is_fs_mounted(mountinfo, fs, target_prefix);
```

这里并不是简单判断“target 上有没有 mount”，而是判断 mountinfo 中是否已经存在与本次 `cxt->fs` 对应的**同一项挂载**。

其主要比较过程为：

```text
本次 cxt->fs
     │
     ├── 确定 source
     │      ├── 普通情况：本次 source
     │      ├── bind：根据已有 mount tree 确定实际 filesystem source/root
     │      └── loop：必要时根据 backing file 匹配 loop device（例如 mount -a 按 fstab 查、尚未改成 loop 设备）
     │
     ├── 确定 filesystem root
     │
     ▼
遍历 mountinfo
     │
     ├── source 是否对应
     │      └── 不同 → 下一项
     │
     ├── root 是否相同
     │      └── 不同 → 下一项
     │
     └── target 是否相同
            └── 相同 → mounted = 1
```

因此，一般情况下可以把判断条件概括为：

```text
source 对应
   &&
filesystem root 相同
   &&
target 相同
        ↓
认为这项 mount 已经存在
```

source 的比较还处理了设备号、loop backing file 等特殊情况；NFS、CIFS、btrfs 以及 bind mount 的 filesystem root 也有相应处理。

##### 3.10.3 `MNT_STAGE_PREP`：选择实际挂载路径

`onlyonce` 通过以后：

```c
mnt_context_call_hooks(cxt, MNT_STAGE_PREP);
```

这里，如果支持新的 API，也就是打开了 `USE_LIBMOUNT_MOUNTFD_SUPPORT`（从版本 2.39 开始）宏，`MNT_STAGE_PREP` 里面注册有两个 hook：一个 `hook_mount` 使用新的 mount API；一个 `hook_mount_legacy` 使用 legacy 的 API。

`MNT_STAGE_PREP` 执行时，先执行 `hook_mount`；本次挂载的选项已经整理在 `optlist` 中。`hook_mount` 从中取得两类已经归并好的标志：

```text
MS_*          → 描述本次挂载操作及传统 mount flags
                如 MS_BIND、MS_MOVE、MS_REMOUNT

MOUNT_ATTR_*  → 描述需要设置的 VFS mount attributes
                如 RDONLY、NOSUID、NOEXEC（只影响是否登记setattr的hook）
```

其中，`MS_BIND`、`MS_MOVE`、`MS_REMOUNT` 等 `MS_*` 标志决定了这次是在**新建 filesystem、操作 source 上已有的 mount tree，还是修改 target 上已有的 mount**。

`hook_mount` 会在`init_sysapi()` 里据此为后续的新 mount API 操作准备相应的 fd：

```text
                         init_sysapi()
                              │
          ┌───────────────────┼────────────────────┐
          │                   │                    │
       普通 mount          bind / move       remount / propagation
          │                   │                    │
          ▼                   ▼                    ▼
     fsopen(fstype)      open_tree(source)     open_tree(target)
          │                   │                    │
          ▼                   ▼                    ▼
        fd_fs              fd_tree              fd_tree
```

三条路径准备的对象不同：

* **普通 mount** 尚不存在可操作的 mount tree，因此通过 `fsopen()` 创建 filesystem context，得到 `fd_fs`；后续再通过 `fsconfig()`、`fsmount()` 创建新的 mount tree。
* **bind / move** 操作的是 source 上已经存在的 mount tree，因此从 source 执行 `open_tree()`，得到 `fd_tree`。
* **remount / propagation** 修改的是 target 上已经存在的 mount，因此从 target 执行 `open_tree()`，得到 `fd_tree`。

`init_sysapi()` 完成这些基础 fd 的准备后，`hook_mount` 再根据具体操作和 flags 注册后续 MOUNT / MOUNT_POST 阶段需要执行的 hooks。

如果在上面的过程中，新 API 不可用，例如 fsopen() / mount_setattr() 返回 ENOSYS，`hook_mount` 会删掉已经注册的新 API hooks，使后面的 `hook_mount_legacy` 接管。当然，若一开始就是带 SELinux mount options 的 btrfs（`cxt->has_selinux_opt`），也会直接走传统 `hook_mount_legacy`，因为当时 btrfs 的 fsconfig() 对该情况支持不完整。当走到传统 `hook_mount_legacy`，一开始就会检查是否存在新API注册的hook，如果存在，直接返回。

下面重点介绍`hook_mount`的几条路径：

###### 3.10.3.1 普通 mount：创建新的 filesystem + mount tree

例如：

```text
mount /dev/sdb1 /mnt
```

它不是从已有 mount tree 中取得 mount tree，而是先创建新的 filesystem context，再创建新的 mount tree：

```text
PREP
 │
 └── init_sysapi()
        │
        └── fsopen(type)
                ↓
             fd_fs

MOUNT
 │
 └── hook_create_mount()
        │
        ├── fsconfig(fd_fs, "source", source)
        ├── fsconfig(...)             配置 filesystem options
        ├── FSCONFIG_CMD_CREATE
        │
        └── fsmount(fd_fs)
                ↓
             fd_tree
          detached mount tree

MOUNT
 │
 └── hook_set_vfsflags()
        ↓
     mount_setattr()

MOUNT_POST
 │
 ├── hook_attach_target()
 │      └── move_mount(fd_tree → target)
 │
 └── hook_set_propagation()           若需要
```

这里有两个不同层次的配置：

```text
filesystem / superblock 配置
        → fsconfig()

VFS mount 属性
        → mount_setattr()
```

最后才通过 `move_mount()` 把 detached mount tree attach 到 target。

###### 3.10.3.2 bind / rbind 与 move：从 source 取得 mount tree

bind / rbind：从 source 克隆已有 mount tree

例如：

```text
mount --bind /mnt/a /mnt/b
```
move 的含义就是：把一个已经存在的 mount 从原挂载点整体移动到另一个挂载点，不创建新的 mount，也不复制 mount。

例如原来：

/dev/sdb1 mounted on /mnt/a

执行：

mount --move /mnt/a /mnt/b
之后：
/dev/sdb1 → /mnt/b

bind 和 move 都不创建新的 filesystem，而是操作 **source 上已经存在的 mount tree**。因此它们不走普通 mount 的：

```text
fsopen → fsconfig → fsmount
```

`init_sysapi()` 对二者都先取得 source：

```c
path = mnt_fs_get_srcpath(cxt->fs);
```

然后通过 `open_tree(source, ...)` 取得 mount tree。bind 与 move 的区别从这里开始：bind 使用 `OPEN_TREE_CLONE` 克隆 source 上的 mount tree，而 move 不使用 `OPEN_TREE_CLONE`，直接取得已有 mount tree。

```text
                               PREP
                                │
                          init_sysapi()
                                │
                 ┌──────────────┴──────────────┐
                 │                             │
            bind / rbind                     move
                 │                             │
                 ▼                             ▼
     open_tree(source, CLONE)          open_tree(source)
                 │                             │
                 ▼                             ▼
       cloned detached tree               fd_tree
                 │                             │
                 ▼                             │
              MOUNT                            │
                 │                             │
                 ▼                             │
       mount_setattr()                         │
       （需要修改 VFS 属性时）                    │
                 │                             │
                 └──────────────┬──────────────┘
                                ▼
                           MOUNT_POST
                                │
                                ▼
                           move_mount(fd_tree → target)
```

对于 **bind / rbind**，`OPEN_TREE_CLONE` 使 `open_tree()` 克隆 source 上已有的 mount tree，因此 source 上原来的 mount 保留；克隆得到的 detached tree 可以先通过 `mount_setattr()` 设置所需的 VFS 属性，再由 `move_mount()` attach 到 target。`rbind` 在此基础上递归处理 source 下的子 mount。

对于 **move**，`open_tree()` 不使用 `OPEN_TREE_CLONE`，因此不会克隆出另一个 mount tree；取得已有 mount tree 的 `fd_tree` 后，直接通过 `move_mount()` 将其移动到 target，原来的挂载位置不再保留该 mount。

源码中二者正是通过是否加入 `OPEN_TREE_CLONE` 区分：

```c
if ((flags & MS_BIND) && !(flags & MS_REMOUNT))
        oflags |= OPEN_TREE_CLONE;
```

###### 3.10.3.3 remount：通过 target 修改已有 mount

remount 修改的是 **target 上已经存在的 mount**，包括普通 `remount` 和 `remount,bind`。

在 PREP 阶段，`init_sysapi()` 对 `MS_REMOUNT` 使用：

```c
path = mnt_fs_get_target(cxt->fs);
```

随后通过 `open_tree(target)` 取得已有 mount 的 `fd_tree`。

这与前两节介绍的普通 mount、bind 不同：

```text
普通 mount    → fsopen(fstype)，创建新的 filesystem context
bind / rbind   → open_tree(source, CLONE)，克隆已有 mount tree
普通 remount   → open_tree(target)，取得已有 mount，重新配置 filesystem 和 VFS 属性
remount,bind   → open_tree(target)，取得已有 mount，只修改 VFS 属性
```

因此，**两种 remount 都由 target 确定操作对象，区别在于是否重新配置 filesystem/superblock。**

例如：

```bash
mount -o remount,ro /mnt
```

`/mnt` 就是需要修改的已有 mount。libmount 可以根据 mountinfo 等挂载表信息补全 source、fstype 和相关 options。

即使显式指定 source：

```bash
mount -o remount,ro /dev/sda1 /mnt
```

新 API 仍然通过 `open_tree("/mnt")` 定位操作对象，而不是通过 `/dev/sda1` 查找 mount。source 可以参与 libmount 的用户空间参数及挂载表处理，但不是 `open_tree()` 定位已有 mount 的参数。

**两种 remount 在新 API 中的执行流程如下：**

```text
                          PREP
                           │
                           ▼
                     init_sysapi()
                           │
                           ▼
                 open_tree(target) → fd_tree
                           │
                           ▼
                          MOUNT
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
         普通 remount              remount,bind
              │                         │
              ▼                         │
    hook_reconfigure_mount()            │
              │                         │
              ▼                         │
      fspick(fd_tree) → fd_fs           │
              │                         │
              ▼                         │
   configure_superblock(fd_fs, 1)       │
              │                         │
              ▼                         │
   fsconfig(FSCONFIG_CMD_RECONFIGURE)   │
     （filesystem / superblock）         │
              │                         │
              └────────────┬────────────┘
                           │
                           ▼
                  hook_set_vfsflags()
                           │
                           ▼
                 mount_setattr(fd_tree)
                    （VFS 属性）
                           │
                           ▼
                       remount 完成
```

其中，`hook_set_vfsflags()` 根据本次 VFS 属性的 `set/clr` 决定是否调用 `mount_setattr()`；图中展示的是需要修改 VFS 属性的情况。

**1. 普通 remount：filesystem reconfigure + VFS 属性修改**

例如：

```bash
mount -o remount,ro /mnt
```

`hook_reconfigure_mount()` 通过 `fspick(fd_tree)` 取得已有 filesystem 的配置上下文，再调用：

```c
configure_superblock(cxt, hs, fd_fs, 1);
fsconfig(fd_fs, FSCONFIG_CMD_RECONFIGURE, NULL, NULL, 0);
```

其中 `configure_superblock()` 的最后一个参数 `1` 表示 `force_rwro`：如果没有显式指定 `ro/rw`，则补充 `rw`，以兼容传统 remount 的语义。

随后，`hook_set_vfsflags()` 通过 `mount_setattr()` 修改当前 mount 的 VFS 属性。

因此，普通 `remount,ro` 同时涉及 filesystem/superblock 层和 VFS mount 层的只读状态。

**2. remount,bind：只修改 VFS 属性**

例如：

```bash
mount -o remount,bind,ro /mnt
```

这里的 `bind` **不是重新执行 bind，也不要求 `/mnt` 原本由 bind 创建**，而是限定本次 remount 只修改 target mount 的 VFS 属性。

源码通过以下条件决定是否注册 filesystem reconfigure hook：

```c
(flags & MS_REMOUNT) && !(flags & MS_BIND)
```

因此，`remount,bind` 不执行 `hook_reconfigure_mount()`，也不调用 `fspick()` 或 `fsconfig(FSCONFIG_CMD_RECONFIGURE)`，而是通过 `hook_set_vfsflags()`、`mount_setattr()` 修改已有 mount 的 VFS 属性。

两者的区别可以概括为：

```text
remount,ro
    → filesystem/superblock reconfigure
    → 修改当前 mount 的 VFS ro 属性

remount,bind,ro
    → 不重新配置 filesystem/superblock
    → 只修改当前 mount 的 VFS ro 属性
```

这保留了传统 `mount(2)` 对普通 remount 与 bind-remount 的语义区别，但新 API 将 filesystem 重配置和 VFS 属性修改拆成了独立操作。

**两种 remount 都不创建新的 mount tree，也不需要在 MOUNT_POST 阶段重新 attach 到 target。**

###### 3.10.3.4 propagation-only：取得 target 后修改 propagation

只修改：

```text
shared
private
slave
unbindable
```

等 propagation 属性时，也不需要创建 filesystem。

它和 remount 一样，从 target 取得当前 mount tree：

```text
                         PREP
                          │
                    init_sysapi()
                          │
                          ▼
              open_tree(target) → fd_tree
                          │
                          ▼
                     MOUNT_POST
                          │
                          ▼
      mount_setattr(fd_tree, ... propagation ...)
                          │
                          ▼
                 propagation 完成
```

因此，新 mount API 下各类挂载在 `PREP`、`MOUNT` 和 `MOUNT_POST` 阶段的执行路径可以总结为：

| 挂载类型             | `init_sysapi()` 路径         | MOUNT 阶段                                                                            | MOUNT_POST                                                   |
| ---------------- | -------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| 普通 mount         | `fsopen(type)`             | `hook_create_mount`（`fsconfig` → `fsmount`）；按需 `hook_set_vfsflags`                  | `hook_attach_target`（`move_mount`）；按需 `hook_set_propagation` |
| bind / rbind     | `open_tree(source, CLONE)` | 按需 `hook_set_vfsflags`（`mount_setattr`）                                             | `hook_attach_target`（`move_mount`）；按需 `hook_set_propagation` |
| move             | `open_tree(source)`        | —                                                                                   | `hook_attach_target`（`move_mount`）；按需 `hook_set_propagation` |
| remount          | `open_tree(target)`        | `hook_reconfigure_mount`（`fspick` → `fsconfig(RECONFIGURE)`）；按需 `hook_set_vfsflags` | 按需 `hook_set_propagation`                                    |
| `remount,bind`   | `open_tree(target)`        | 按需 `hook_set_vfsflags`（`mount_setattr`），不执行 filesystem reconfigure                  | 按需 `hook_set_propagation`                                    |
| propagation-only | `open_tree(target)`        | —                                                                                   | `hook_set_propagation`                                       |

##### 3.10.4 各 PREP stage 在 `prepare_mount()` 中的位置

前面出现的几个 PREP stage 分别处理不同对象：

| hook stage     | 所在步骤                 | 主要作用                                                                      |
| -------------- | -------------------- | ------------------------------------------------------------------------- |
| `PREP_OPTIONS` | `fix_optstr` 末尾      | 根据最终 options 准备 idmap、owner、SELinux 等数据                                   |
| `PREP_SOURCE`  | `prepare_srcpath` 末尾 | 处理 source，如 loop、verity                                                   |
| `PREP_TARGET`  | `prepare_target` 末尾  | 处理 target，如 subdir、mkdir、SELinux target                                   |
| `PREP`         | `prepare_mount` 最后   | 根据最终 source、target 和 options 选择 mount API 路径，并注册 MOUNT / MOUNT_POST hooks |

因此前三个阶段是在分别把 **options、source、target 准备完整**；最后的 `MNT_STAGE_PREP` 才根据这些最终信息，组织接下来真正要执行的挂载动作。

`MNT_STAGE_PREP` 成功完成后设置：

```text
MNT_FL_PREPARED
```

后面的 `mnt_context_do_mount()` 才进入 MOUNT 阶段，执行这里已经确定并注册好的挂载操作。

### 4. `prepare_update`：准备 utab 更新信息

`mnt_context_prepare_update()` 负责判断本次操作是否需要更新 userspace mount table；如果需要，则创建 `cxt->update`，并通过 `mnt_update_set_fs()` 准备后续更新 utab 所需的信息。

这里**只做准备，不写 utab**。真正向 `/run/mount/utab` 提交更新是在后面的 `update_tabs` 阶段。

对于这里讨论的 mount 路径，主要流程是：

```text
检查本次操作是否需要更新 utab
        ↓
找到 writable utab 路径
        ↓
创建 cxt->update
        ↓
取得本次 mount flags
        ↓
mnt_update_set_fs(cxt->update, flags, NULL, cxt->fs)
```

其中有两个不同层次的对象：

```text
cxt->update
    │
    └── update->fs
```

* **`cxt->update`**：utab 更新操作的 handler，保存 utab 文件名以及本次更新所需的状态；
* **`update->fs`**：根据当前 `cxt->fs` 准备出来的 utab 条目，后续 `update_tabs` 会以它为基础更新 utab。

#### 4.1 为什么在 `do_mount` 之前准备

`prepare_update` 放在实际挂载之前，并不是提前写 utab，而是提前准备**内存中的更新信息**。

这样做主要有三个作用。

##### 1. 提前发现无法准备更新项的错误

mount 路径最终调用：

```c
rc = mnt_update_set_fs(cxt->update, flags,
                        NULL, cxt->fs);
```

`mnt_update_set_fs()` 会检查本次更新所需的信息。

例如 move mount 要求 `fs` 中存在 source path；如果缺失，会直接返回负 errno。

因此：

```text
prepare_update
    ↓
mnt_update_set_fs()
    ↓
发现更新信息本身不合法
    ↓
返回负 errno
```

上层调用链因此停止，不必等到实际 mount syscall 后才发现问题。

##### 2. 将 utab 信息与后续的临时挂载改动隔离

这是 `prepare_update` 放在 `do_mount` 前面更重要的原因之一。

此时 `prepare_mount` 已经完成，`cxt->fs` 中的 source、target 等信息已经准备好，因此可以先据此建立 utab 更新项。

后面进入实际挂载阶段时，mount hooks 可能为了完成挂载而**临时修改 `cxt->fs`**。

例如 subdir 处理过程中，MOUNT_PRE 阶段可能暂时把：

```text
cxt->fs->target
```

改成内部使用的临时挂载点：

```text
/run/mount/...
```

这些修改服务于实际挂载过程，并不代表用户最终要求的 mount target。

而 `update->fs` 已经在此前根据正常的 `cxt->fs` 准备完成，因此不会因为后面对 `cxt->fs` 的这种临时修改而丢失原来的 utab 信息。

可以理解为：

```text
prepare_mount 完成

cxt->fs
   │
   │ prepare_update
   ▼
update->fs
   │
   └── 保存 utab 所需的信息

随后进入 do_mount：

cxt->fs
   │
   └── 可以因 mount hook 等需要发生临时变化
```

因此 `update->fs` 与实际挂载过程中继续使用的 `cxt->fs` 是分开的。

##### 3. 挂载过程中仍可回填最终结果

提前建立 `update->fs` 并不意味着它此后完全不能变化。

有些信息只有实际挂载过程中才能确定，因此成功后还会有针对性地同步到 `update->fs`。

例如 new mount API 的成功路径中，`hook_create_mount` 可以通过 `statx()` 得到 mount ID：

```text
statx()
   ↓
得到 mnt_id
   ↓
写入 cxt->fs->id
   ↓
如果 update->fs 存在
   ↓
同步到 update->fs
```

另一个例子是 filesystem type 自动探测。

如果 mount 最初没有确定唯一 fstype，而是在 `do_mount` 中通过 `try_type` 尝试多个类型，那么某个类型成功后，会把最终成功的类型同步给：

```c
mnt_fs_set_fstype(mnt_update_get_fs(cxt->update), try_type);
```

因此整体关系是：

```text
prepare_update
      ↓
先建立 utab 更新项
      ↓
do_mount
      ↓
得到只有实际挂载后才能确定的信息
      ↓
必要时回填 update->fs
      ↓
update_tabs 最终提交
```

所以 `update->fs` 更准确地说，是**挂载前建立、挂载过程中允许补充最终结果的 utab 更新项**。

#### 4.2 哪些情况会跳过 utab 更新准备

`mnt_context_prepare_update()` 并不是每次都会创建 `cxt->update`。

函数前半部分首先判断本次操作是否需要维护 userspace mount table。

##### 1. propagation-only

```c
if (mnt_context_propagation_only(cxt)) {
        ...
        return 0;
}
```

如果本次操作只是：

```text
--make-shared
--make-private
--make-slave
--make-unbindable
```

这一类 propagation 修改，就直接返回。

因为这里只是在修改已有 mount tree 的传播属性，不需要为此准备新的 utab 更新项。

##### 2. NOMTAB

随后检查：

```c
if (mnt_context_is_nomtab(cxt)) {
        ...
        return 0;
}
```

例如使用 `mount -n` 时，libmount 被明确要求不要维护 userspace mount table，因此不需要创建 `cxt->update`。

这里还有一个 umount 特例：

```c
if (cxt->action == MNT_ACT_UMOUNT &&
    target && !strcmp(target, "/"))
        mnt_context_disable_mtab(cxt, TRUE);
```

即卸载 `/` 时会主动设置 NOMTAB。

对于本章讨论的 mount 路径，主要关注前面的 `mnt_context_is_nomtab()` 判断即可。

##### 3. 没有 writable table

接着：

```c
name = mnt_context_get_writable_tabpath(cxt);

if (!name)
        return 0;
```

libmount 会先确定是否存在可以维护的 userspace mount table。

如果没有 writable destination，就不再准备 update。

因此：

```text
没有 writable utab
        ↓
不创建 cxt->update
        ↓
不影响后面实际 mount
```

utab 是 userspace 的补充记录，并不是内核执行 mount 所必需的。

##### 4. syscall 已经失败

接下来是：

```c
/* 0 = success, 1 = not called yet */
if (cxt->syscall_status != 1 &&
    cxt->syscall_status != 0) {
        ...
        return 0;
}
```

这里需要注意 `syscall_status` 的含义：

```text
1   = syscall 尚未执行
0   = syscall 已成功
其他 = syscall 已失败
```

在正常的 mount 流程中，`prepare_update` 第一次执行时还没有进入 `do_mount`，因此通常是：

```text
syscall_status == 1
```

这是允许继续的。

代码同时接受：

```text
syscall_status == 0
```
#### 4.3 创建 `cxt->update`

通过前面的检查后，如果 update handler 尚不存在：

```c
if (!cxt->update) {
        cxt->update = mnt_new_update();
        if (!cxt->update)
                return -ENOMEM;

        mnt_update_set_filename(cxt->update, name);
}
```

这里完成两件事：

```text
mnt_new_update()
        ↓
创建 cxt->update

mnt_update_set_filename()
        ↓
记录要更新的 userspace table 路径
```

注意此时只是创建 **update handler**。

真正代表某一条 mount 记录的 `update->fs`，是在下面的 `mnt_update_set_fs()` 中进一步准备的。

#### 4.4 `mnt_update_set_fs()`：准备本次 mount 的 utab 条目

创建 update handler 后，`prepare_update()` 首先取得已经合并好的 mount flags：

```c
mnt_context_get_mflags(cxt, &flags);
```

对于 mount 路径，随后调用：

```c
rc = mnt_update_set_fs(cxt->update, flags,
                       NULL, cxt->fs);
```

这里传入的 `cxt->fs`，就是经过 `prepare_mount` 整理后的本次挂载描述。

`mnt_update_set_fs()` 根据本次操作类型及 mount flags（如 move、remount、bind），决定如何准备更新信息。对于普通 mount，最终会进入 `utab_new_entry()`，根据 `cxt->fs` 准备用于 utab 的 `update->fs`。

整体流程如下：

```text
mnt_context_prepare_update(cxt)
        │
        ▼
mnt_update_set_fs(cxt->update, flags, NULL, cxt->fs)
        │
        ▼
utab_new_entry()
        │
        ├── 取得并过滤 user options（排除 MNT_NOMTAB）
        │
        ├── 取得 fs->attrs
        │
        ▼
判断过滤后的 OPTS 和 ATTRS 是否均为空？
        │
        ├── 是 → return 1（不创建 update->fs）
        │
        └── 否
             │
             ▼
         mnt_copy_fs()
             │
             ▼
       生成独立的 update->fs
```

**1. 取得并过滤 `OPTS`**

utab 中的 `OPTS=` 保存的是经过过滤后，仍需要持久化的 **userspace options**。

`utab_new_entry()` 首先通过：

```c
mnt_fs_get_user_options(fs)
```

取得当前 `fs` 的 userspace options，再根据 `userspace_opts_map` 中的 `MNT_NOMTAB` 标记进行过滤。

`MNT_NOMTAB` 是 libmount 选项映射表中的一个标记，表示对应选项不需要写入用户空间挂载表。它并不表示该选项在本次 mount 中无效，只是表示不需要在 utab 中继续保存。

因此，`cxt->fs` 中原有的 userspace options，并不一定全部进入最终的 `OPTS=`。

典型情况如下：

| userspace option | 是否写入 `OPTS=` | 原因 |
|---|:---:|---|
| `user` / `user=name` | 是 | 保留用户挂载信息 |
| `_netdev` | 是 | 保留网络挂载语义 |
| `helper=` / `uhelper=` | 是 | 未标记 `MNT_NOMTAB` |
| `x-...` | 是 | 小写 `x-` 允许持久化 |
| `noauto` / `auto` | 否 | `MNT_NOMTAB` |
| `users` / `owner` / `group` | 否 | `MNT_NOMTAB` |
| `nofail` | 否 | `MNT_NOMTAB` |
| `offset=` / `sizelimit=` / `encryption=` | 否 | loop setup 信息，`MNT_NOMTAB` |
| `verity.*` | 否 | verity setup 信息，`MNT_NOMTAB` |
| `X-...` / `X-mount.*` | 否 | `X-` 前缀带有 `MNT_NOMTAB` 标记 |

需要注意小写 `x-` 与大写 `X-` 的区别：前者可以保留在 utab 中，后者不会写入 `OPTS=`。

**2. 取得 `ATTRS`**

处理完 userspace options 后，`utab_new_entry()` 继续通过：

```c
mnt_fs_get_attributes(fs)
```

取得 `fs->attrs`，用于准备utab中的`ATTRS`字段。

这里的 `ATTRS` 是 libmount 为挂载条目额外保存的**用户空间属性信息**，供后续用户空间程序读取和使用。这些信息不属于内核 `mountinfo` 记录的挂载状态，因此需要由 utab 补充保存，对应字段为 `ATTRS=`。

需要将它与 **3.2.4 节介绍的 VFS mount attributes** 区分开来：

- `mnt_optlist_get_attrs()`：从结构化 VFS options 中取得 `MOUNT_ATTR_*` 标志，如 `MOUNT_ATTR_RDONLY`、`MOUNT_ATTR_NOSUID`，用于后续通过 `mount_setattr()` 设置内核 mount 属性。
- `mnt_fs_get_attributes()`：取得 `fs->attrs` 中的用户空间属性信息，用于准备 utab 的 `ATTRS=` 字段。

两者虽然都使用 attributes 这一名称，但前者用于设置内核挂载属性，后者用于保存用户空间的补充信息，不能混为一谈。

**3. 判断是否需要创建 utab 条目**

取得过滤后的 `OPTS` 和 `ATTRS` 后，`utab_new_entry()` 会检查两者是否均为空。

如果：

```text
过滤后的 OPTS 为空
并且
ATTRS 为空
```

说明本次 mount 没有需要由 utab 补充保存的信息，函数直接返回 `1`，不再执行后面的 `mnt_copy_fs()`，也不会创建 `update->fs`。

因此，可能出现：

```c
cxt->update != NULL
cxt->update->fs == NULL
```

这不是错误，而是表示 update handler 已经存在，但本次挂载没有需要实际写入 utab 的条目。

对应地，`mnt_update_set_fs()` 的返回值含义为：

| 返回值 | 含义 |
|---|---|
| `rc < 0` | 准备更新信息时发生错误 |
| `rc == 0` | 已成功准备 `update->fs` |
| `rc == 1` | 不需要创建 utab entry |

在 `prepare_update()` 中，最终通过：

```c
return rc < 0 ? rc : 0;
```

将 `rc == 1` 也作为成功处理。

**4. 调用 `mnt_copy_fs()` 创建独立的 `update->fs`**

如果过滤后的 `OPTS` 或 `ATTRS` 至少有一项需要保存，`utab_new_entry()` 才继续调用 `mnt_copy_fs()`，根据当前 `cxt->fs` 创建独立的 `libmnt_fs` 对象，并保存为 `update->fs`。

这里需要明确：`update->fs` 是复制出来的独立对象，而不是简单保存一个指向 `cxt->fs` 的指针。

后续实际挂载仍然使用 `cxt->fs`；`update->fs` 则用于保存本次 utab 更新信息，并可在挂载过程中补充 mount ID、最终 fstype 等结果。

```text
update->fs
    │
    ▼
必要时回填 mount ID / 最终 fstype 等结果
    │
    ▼
update_tabs
    │
    ▼
/run/mount/utab
```

至此，`prepare_update()` 完成的是 utab 更新条目的准备工作，实际写入仍由后续 `update_tabs` （见第6节）完成。

#### 4.5 最终准备出的 utab 信息

`prepare_update()` 负责准备 utab 更新条目，实际写入由后续的 `mnt_context_update_tabs()`（见第6节） 完成。

对于普通 mount，只有过滤后的 `OPTS` 或 `ATTRS` 非空时，`utab_new_entry()` 才创建 `update->fs`，后续根据该对象生成 utab 记录。

utab 的字段及输出条件如下：

| 字段 | 内容及输出条件 |
|---|---|
| `ID=` | mount ID；`update->fs` 中已设置有效 mount ID 时输出 |
| `SRC=` | 本次挂载最终记录的 source；字段非空时输出 |
| `TARGET=` | 实际挂载点；字段非空时输出 |
| `ROOT=` | 该 mount 在 filesystem 中对应的 root；字段非空时输出 |
| `BINDSRC=` | 原始 bind source；设置了该字段时输出 |
| `ATTRS=` | libmount 单独维护的用户空间属性信息；非空时输出 |
| `OPTS=` | 经过 `MNT_NOMTAB` 过滤后保留的 userspace options；非空时输出 |

**`SRC`、`TARGET`、`ROOT`、`ID` 等字段用于标识挂载条目，本身不会触发 utab 条目的创建。** 如果过滤后的 `OPTS` 和 `ATTRS` 均为空，`utab_new_entry()` 直接返回 `1`，不创建 `update->fs`，因此也不会单独写入上述标识字段。

### 5. `do_mount`：执行挂载

`mnt_context_do_mount()` 负责执行已经准备好的挂载操作。本阶段不仅包括实际的 mount syscall，还包括挂载前的 hook、挂载后的 attach，以及部分特殊挂载所需的收尾操作。

**本阶段涉及的主要对象变动：**

| 对象 | 变动内容 |
|---|---|
| `cxt->fs` | subdir 临时修改 target；fstype 试探时临时修改 fstype；成功挂载后补充 mount ID |
| `cxt->optlist` | fstype 试探期间临时加入 `MS_SILENT`，供 legacy `mount(2)` 抑制内核诊断信息；新 mount API 不使用该标志，离开 `do_mount()` 前将其移除。 |
| `cxt->update->fs` | 挂载成功后回填 mount ID、最终 fstype 等信息 |

#### 5.1 `mnt_context_do_mount()` 的内部执行结构

`mnt_context_do_mount()` 按以下顺序执行三个部分：

1. `MNT_STAGE_MOUNT_PRE`：执行实际挂载前需要完成的 hook。
2. 实际挂载：根据 helper、fstype 和 mount flags，选择对应的执行路径。
3. `MNT_STAGE_MOUNT_POST`：执行挂载后的 hook，包括 attach target、subdir 处理和 propagation 设置等。

对应的核心调用关系为：

```c
rc = mnt_context_call_hooks(cxt, MNT_STAGE_MOUNT_PRE);

if (!rc) {
    if (/* 单一 fstype */)
        res = do_mount(cxt, NULL);
    else if (/* 逗号分隔的 fstype 列表 */)
        res = do_mount_by_types(cxt);
    else
        res = do_mount_by_pattern(cxt);
}

if (res == 0)
    rc = mnt_context_call_hooks(cxt, MNT_STAGE_MOUNT_POST);
```

其中，`MNT_STAGE_MOUNT_POST` 只有在实际挂载成功后才执行。

#### 5.2 `MNT_STAGE_MOUNT_PRE`：subdir 的临时 target

完整的`X-mount.subdir` 挂载流程参考3.8.2小节准备挂载目标。
对于使用 `X-mount.subdir` 的挂载，`hook_subdir` 注册的 `hook_mount_pre()` 会在实际挂载前就是MNT_STAGE_MOUNT_PRE阶段准备临时环境。

其执行过程为：

```text
保存真实 target → org_target
        │
        ▼
tmptgt_unshare()
        │
        ├── 创建 /run/mount/tmptgt
        ├── unshare(CLONE_NEWNS)
        └── 设置临时环境所需的 private mount
        │
        ▼
cxt->fs->target = MNT_PATH_TMPTGT
        │
        ▼
注册 MOUNT_POST → hook_mount_post
```

`MNT_PATH_TMPTGT` 对应的路径为 `/run/mount/tmptgt`。

此时，`cxt->fs->target` 已从真实 target 临时修改为 tmptgt，因此接下来的实际挂载会先发生在临时 target 上。

需要注意，`cxt->update->fs` 已在 `prepare_update()` 中根据原始 `cxt->fs` 独立创建，其 target 仍保存真实挂载点，不受这里临时修改的影响。

subdir 的子目录提取、真实 target 恢复及临时资源清理，在后面的 `MNT_STAGE_MOUNT_POST` 中完成。

#### 5.3 实际挂载：helper、fstype 与 `do_mount()`

完成 `MOUNT_PRE` 后，libmount 进入实际挂载部分。

##### 5.3.1 helper 路径

如果 `prepare_mount()` 已经为本次操作设置了 `cxt->helper`，则执行 `exec_helper()`，由外部 helper 完成挂载。

helper 的选择条件、参数构造和执行过程已在 **3.9 节**介绍，这里不再展开。

需要注意，helper 执行后，libmount 内部 mount hook 会根据 `helper_executed` 状态跳过不应重复执行的内部挂载操作。

##### 5.3.2 fstype 分支与 `do_mount()`

没有使用外部 helper 时，libmount 根据 fstype 的具体形式，选择 `do_mount()`、`do_mount_by_types()` 或 `do_mount_by_pattern()`。

fstype 的完整选择规则见 **3.7 节**。这里重点说明最终进入的核心函数：

`do_mount(cxt, try_type)`

其中：

- `try_type == NULL`：直接使用 `cxt->fs->fstype`。
- `try_type != NULL`：使用传入的候选 fstype 赋值给`cxt->fs->fstype`进行一次挂载试探。

第二种情况是当 fstype 为逗号列表或需要逐项试探的 pattern 时进入的。如果本次试探失败，则把 `cxt->fs->fstype` 恢复为试探前的值；如果成功，则保留此时的 `cxt->fs->fstype`，并通过 `mnt_update_get_fs()` 将其同步到已经创建的 `update->fs`（如果存在）。

另外，第二种情况一次失败并不代表整个挂载操作失败，因此需要避免每次试探都产生不必要的内核诊断信息。`do_mount()` 会临时设置 `MS_SILENT`到`cxt->optlist`，`MS_SILENT` 是 legacy `mount(2)` 使用的挂载标志，要求文件系统抑制部分内核诊断输出；它并不负责屏蔽 libmount 自身的 stderr 错误报告。新 mount API 不使用这一 `mount(2)` 标志。离开 `do_mount()` 前，还会移除本次试探临时加入的 `MS_SILENT`。

#### 5.4 `MNT_STAGE_MOUNT`：执行主体挂载操作

完成 `MOUNT_PRE` 后，`mnt_context_do_mount()` 进入主体挂载流程，其中会调用：

```c
mnt_context_call_hooks(cxt, MNT_STAGE_MOUNT);
```

执行此前（见 3.10 的 `hookset_mount`）注册到 `MNT_STAGE_MOUNT` 的 hook。

对于新 mount API，这些 hook 根据本次操作类型完成 filesystem 的创建或重新配置，以及 VFS mount 属性的设置。不同操作注册的 hook 不完全相同，例如普通新建 filesystem 会执行 `hook_create_mount()`，需要设置 VFS mount 属性时还会执行 `hook_set_vfsflags()`。

新 mount API 的各个 hook 通过 `get_sysapi()` 取得当前 `cxt` 中 `hookset_mount` 的共享数据 `struct libmnt_sysapi`，由 3.10.3 `MNT_STAGE_PREP` 阶段的 `init_sysapi()` 函数创建，下文简称 `api`。其中：

- `api->fd_fs` 保存 `fsopen()` / `fspick()` 取得的 filesystem FD；
- `api->fd_tree` 保存 `fsmount()` / `open_tree()` 取得的 mount tree FD。

这些 FD 可以由后续 hook 继续使用。

##### 5.4.1 普通新建 filesystem：`hook_create_mount()`

`hook_create_mount()` 是前面 3.10 节介绍的 `hook_mount()` 在普通新建 filesystem 场景下注册到 `MNT_STAGE_MOUNT` 的 hook。完整的创建流程已经在 3.10.3.1 说明，这里只看 `hook_create_mount()` 自身完成的工作。

进入 `hook_create_mount()` 时，PREP 阶段的 `init_sysapi()` 已经执行（见 3.10.3 节）：

```c
fsopen(fstype, ...)
```

并将返回的 filesystem context FD 保存到 `api->fd_fs`。

因此，`hook_create_mount()` 不再创建 filesystem context，而是使用这个已经存在的 `fd_fs` 完成 **配置 filesystem context → 创建 filesystem → 创建 mount tree** 的过程：

```text
                    hook_create_mount()
                            │
                            ▼
       fsconfig(fd_fs, FSCONFIG_SET_STRING,
                "source", source, ...)
                            │
                            ▼
              configure_superblock()
                            │
                            ├── fsconfig(fd_fs,
                            │            FSCONFIG_SET_STRING, ...)
                            │
                            ├── fsconfig(fd_fs,
                            │            FSCONFIG_SET_FLAG, ...)
                            │
                            └── ...
                            │
                            ▼
       fsconfig(fd_fs, FSCONFIG_CMD_CREATE,
                NULL, NULL, 0)
                            │
                            ▼
          fsmount(fd_fs, ...) → api->fd_tree
                    （detached mount tree）
                            │
                            ▼
             statx(api->fd_tree, ...)
                            │
                 取得新 mount 的 mount ID
                            │
             ┌──────────────┴──────────────┐
             │                             │
             ▼                             ▼
       cxt->fs->id                 cxt->update->fs->id
                                   （若 update->fs 存在）
```

这里要注意 `FSCONFIG_SET_*` 命令，作用是**向 `fd_fs` 所代表的 filesystem context 写入配置**。

例如 source 会通过类似：

```c
fsconfig(fd_fs, FSCONFIG_SET_STRING,
         "source", source, 0);
```

传入；`configure_superblock()` 则遍历需要交给 filesystem / superblock 的选项，根据选项是否带 value 等情况使用相应的 `FSCONFIG_SET_*` 命令继续配置这个 filesystem context。
这里说一下`configure_superblock()`并不是将整串 `-o` 原样传给 `fsconfig()`，而是逐项检查 optlist去遍历，规则如下：
- `ro/rw`：对 `MS_RDONLY` 保留 superblock 层面的兼容处理。
- 标记为 `MNT_SUPERBLOCK` 的 Linux options：转换为对应的 superblock 配置。
- 没有匹配 option map、也不是 external option 的 FS 专有选项：作为 `name/value` 传给 `fsconfig()`。
- 属于 VFS map、userspace map 或 external 的选项：不在这里作为普通 FS 专有参数提交。

因此，到这里为止：

```text
fsopen()
   ↓
fd_fs
   ↓
FSCONFIG_SET_STRING / FSCONFIG_SET_FLAG / ...
```

都还处于 **filesystem context 的准备和配置阶段**。

只有这些配置完成之后，`hook_create_mount()` 才调用：

```c
fsconfig(fd_fs, FSCONFIG_CMD_CREATE, NULL, NULL, 0);
```

`FSCONFIG_CMD_CREATE` 与前面的 `FSCONFIG_SET_*` 不同，它不是再添加一个配置项，而是告诉内核：

> 前面的 filesystem context 已经配置完成，现在根据这些配置创建 filesystem。

创建完成后，再调用：

```c
fsmount(fd_fs, ...)
```

根据这个 filesystem 创建一个 mount tree。`fsmount()` 返回的 mount FD 保存到：

```c
api->fd_tree
```

此时得到的是 **detached mount tree**，它已经是一个 mount，但还没有 attach 到最终的 target。

随后 `hook_create_mount()` 对这个 mount FD 调用 `statx()`，取得刚创建的 mount 的 **mount ID**。这个 ID 写入：

```text
cxt->fs->id
```

如果已经存在用于更新 utab 的 `cxt->update->fs`，则同时把这个 mount ID 同步到：

```text
cxt->update->fs->id
```
到这里，`hook_create_mount()` 的任务结束。

##### 5.4.2 remount：`hook_reconfigure_mount()`

普通 remount（这里指不带 bind 的）因为不创建新的 filesystem，所以不会注册普通新建 filesystem：`hook_create_mount()`，而会注册重新配置 target 上已经存在的 mount 的函数：`hook_reconfigure_mount()`。普通 remount 的流程见 3.10.3.3 小节，这里不再详细说明，主要只讲 `hook_reconfigure_mount()`。

前面的 PREP 阶段中，`init_sysapi()` 已经通过 `open_tree(target)` 取得现有 mount，并保存到 `api->fd_tree`。进入 MOUNT 阶段后，`hook_reconfigure_mount()` 首先通过 `fspick()` 从这个 mount tree 取得 filesystem configuration context：

```text
                 hook_reconfigure_mount()
                           │
                           ▼
       fspick(api->fd_tree, ...) → api->fd_fs
                           │
                           ▼
        configure_superblock(..., fd_fs, 1)
                           │
                           ▼
 fsconfig(fd_fs, FSCONFIG_CMD_RECONFIGURE, ...)
```

这里与普通新建 filesystem 调用 `configure_superblock()` 有一个重要区别：

```c
/* 普通新建 filesystem */
configure_superblock(cxt, hs, api->fd_fs, 0);

/* remount */
configure_superblock(cxt, hs, api->fd_fs, 1);
```

最后一个参数是 `force_rwro`。

`configure_superblock()` 在遍历 optlist 时，如果遇到 `ro` 或 `rw`，会设置：

```c
has_rwro = 1;
```

而 remount 传入 `force_rwro = 1` 后，如果整个 optlist 中没有 `ro/rw`，函数最后会主动补一个：

```c
if (force_rwro && !has_rwro)
    fsconfig_set_value(cxt, hs, fd, "rw", NULL);
```

因此，remount 的 filesystem/superblock 配置并不是简单地“只把命令行显式写出的 filesystem options 重新提交”。为了兼容传统 `mount(2)` 的 remount 语义，当没有显式指定 `ro/rw` 时，libmount 会向 filesystem configuration context 补入 `rw`。

完成这些 filesystem / superblock 配置后：

```c
fsconfig(api->fd_fs, FSCONFIG_CMD_RECONFIGURE, NULL, NULL, 0);
```

让内核按照新的 configuration 重新配置已有 filesystem。

##### 5.4.3 设置 VFS mount 属性：`hook_set_vfsflags()`

`hook_set_vfsflags()` 同样是注册到 `MNT_STAGE_MOUNT` 的 hook，但它与 `hook_create_mount()` 的职责不同：

```text
hook_create_mount()
        → 创建 filesystem / mount tree

hook_set_vfsflags()
        → 设置 mount tree 的 VFS mount 属性
```

对于普通新建 filesystem，`hook_create_mount()` 先通过 `fsmount()` 创建出 `api->fd_tree`，随后 `hook_set_vfsflags()` 使用这个 `fd_tree`，从 `cxt->optlist` 中取得需要设置和清除的 VFS mount attributes，最终通过 `mount_setattr()` 应用到这个 mount tree：

```text
                hook_set_vfsflags()
                         │
                         ▼
          从 optlist 取得 VFS attributes
                         │
                         ▼
              生成 set / clr mask
                         │
                         ▼
        mount_setattr(api->fd_tree, ...)
                         │
                         ▼
          修改该 mount 的 VFS 属性
```

这里处理的是 **VFS mount 层面的属性**，例如 `ro/rw`、`nosuid/suid`、`nodev/dev`、`noexec/exec` 以及 atime 相关属性。

`hook_set_vfsflags()` 不只用于普通新建 filesystem，也用于 **bind / rbind**。

bind / rbind 不需要在 MOUNT 阶段通过 `fsopen()`、`fsconfig()`、`fsmount()` 创建新的 filesystem 和 mount tree。它们在前面的 PREP 阶段已经由 `init_sysapi()` 从 source 通过 `open_tree(source, OPEN_TREE_CLONE)` 取得相应的 mount tree，并保存到 `api->fd_tree`，具体见 **3.10.3.2** 小节。

所以，对于 bind/rbind，如果本次 bind 指定了需要改变的 VFS attributes，`hook_set_vfsflags()` 再从 optlist 取得相应的 set/clr attributes，并通过 `mount_setattr()` 修改克隆出的 `api->fd_tree`；未要求改变的属性保持从 source 继承的状态。

`hook_set_vfsflags()` 也用于 remount。 remount 在处理 VFS attributes 时，通过 `remount_reset` 兼容传统 `mount(2)` 的属性复位语义如下：

**1. 非递归属性（`MNT_OL_NOREC`）**

传统 `mount(2)` 的普通 remount 会复位部分未显式指定的 VFS flags。因此，对于非 bind 的普通 remount，使用 `remount_reset` 模拟这一行为：

```c
remount_reset =
      MOUNT_ATTR_RDONLY
    | MOUNT_ATTR_NOSUID
    | MOUNT_ATTR_NODEV
    | MOUNT_ATTR_NOEXEC
    | MOUNT_ATTR_NOSYMFOLLOW;
```

遍历 `optlist` 时，显式指定的属性按要求加入 `set/clr`，并从 `remount_reset` 中移除；未显式指定的上述属性则加入 `clr`，最后统一通过 `mount_setattr()` 修改。

这里并不是清除所有旧属性，而是**复位指定范围内未显式保留的属性**。atime 属性不参与这套复位。

**2. 递归属性（`MNT_OL_REC`）**

传统 `mount(2)` 不支持递归修改 VFS mount attributes，这一功能由新 `mount_setattr()` API 提供，因此**不需要兼容传统 remount 的隐式复位语义**。

递归处理只将显式指定的属性加入 `set/clr`，并通过 `mount_setattr(..., AT_RECURSIVE, ...)` 应用到整个 mount tree，其余属性保持不变。

例如 `ro=recursive` 只递归设置 `RDONLY`，不会清除各个子 mount 原有的 `NOSUID`、`NODEV` 等属性。

因此，**非递归的普通 remount 为兼容旧 API 而复位部分未指定属性；递归操作采用新 API 的显式修改语义，只改变用户指定的属性。** `remount,bind` 同样不执行 `remount_reset`。

##### 5.4.4 传统 `mount(2)`：属于 `hook_mount_legacy()`

PREP 阶段会依次处理 `hookset_mount`（新 API）和 `hookset_mount_legacy`（旧 API）。如果新 API 初始化成功，就使用新 API hooks；如果初始化失败且允许回退，则清理新 API hooks，由后续的 legacy hookset 注册 `mount(2)` 操作。

回退的情况主要包括：

- 所需的新 API syscall 不受支持，例如返回 `ENOSYS`。
- btrfs 挂载包含 SELinux 选项，需要使用兼容的 legacy 路径。
- 内核版本低于 5.14，且本次 remount 涉及新 API 的 VFS 属性设置兼容问题。

与新 API 分阶段调用 `fsconfig()`、`fsmount()`、`mount_setattr()` 的方式不同，legacy `mount(2)` 在其接口语义下集中提交这些挂载参数，具体见 3.10.1 小节。

#### 5.5 `MNT_STAGE_MOUNT_POST`：attach 与挂载后处理

实际挂载成功后，libmount 执行 `MNT_STAGE_MOUNT_POST` 中已注册的 hook。

这一阶段的核心工作，是将前面准备好的 mount tree 放到最终 target，并完成依赖挂载结果的特殊处理。

##### 5.5.1 `hook_attach_target()`：将 mount tree 挂载到 target

对于使用新 mount API、需要 attach 的路径，`hook_attach_target()` 通过 `move_mount()` 将 `api->fd_tree` 对应的 detached mount tree 挂载到最终 target。

`force_clone` 目前用于支持 `X-mount.idmap`，要求对已有 mount tree 进行克隆，以便在 detached 状态下设置 ID mapping。

其中，`hook_attach_target()` 包含以下处理：

```c
if (cxt->force_clone &&
    !api->is_new_fs &&
    !mnt_optlist_is_bind(cxt->optlist))
    umount2(target, MNT_DETACH);
```

三个条件分别表示：

- `cxt->force_clone`：要求使用克隆的 mount tree，目前用于 `X-mount.idmap`。
- `!api->is_new_fs`：本次不是通过 `fsopen()`、`fsmount()` 创建新的 filesystem mount。
- `!mnt_optlist_is_bind()`：本次不是 bind/rbind 挂载。

**三个条件同时成立时，表示本次需要用克隆的 mount tree 替换 target 上已有的 mount。** 因此先执行 `umount2(target, MNT_DETACH)`，再通过 `move_mount()` 将准备好的 tree 挂载回 target。

各场景的区别如下：

| 场景 | 处理方式 |
|---|---|
| 普通新建 filesystem（包括带 idmap） | `fsmount()` 创建 detached mount tree，直接通过 `move_mount()` 挂载到 target，即使 target 上已有 mount，也按普通挂载语义进行叠加 |
| bind/rbind（包括带 idmap） | `open_tree(source, CLONE)` 创建 detached mount tree，直接通过 `move_mount()` 挂载到 target，不卸载 target 上原有的 mount |
| 对已有 mount 进行 idmap 转换（如 `remount,X-mount.idmap`） | `open_tree(target, CLONE)` 克隆已有 mount，设置 ID mapping 后，先卸载旧 mount，再通过 `move_mount()` 将克隆挂载回 target，避免额外增加一层挂载 |

这里需要结合 **remount 的使用场景** 来理解最后一种处理方式。

普通 remount，例如：

```bash
mount -o remount,ro /mnt/data
```

修改的是 `/mnt/data` 上已经存在的 mount，并不创建新的挂载层。

而当 remount 同时指定 `X-mount.idmap` 时：

```bash
mount -o remount,X-mount.idmap=b:0:1000:1 /mnt/data
```

由于已经 attached 的 mount 不能直接设置 ID mapping，libmount 必须先通过 `open_tree(target, CLONE)` 得到 detached clone，再对它执行 `mount_setattr(MOUNT_ATTR_IDMAP)`。

此时如果直接执行 `move_mount()`，新的 idmapped mount 就会叠加在原 mount 上，原 mount 仍保留在下层。这与 remount 修改已有挂载的语义不符。

因此，libmount 先通过 `umount2(target, MNT_DETACH)` 卸载旧 mount，再将设置好 ID mapping 的 clone 挂载回原位置。

**所以，是否执行 `umount2()`，并不是由是否指定 idmap 单独决定，而是取决于本次操作是新增一个挂载层，还是用克隆的 mount tree 替换已有挂载。**

##### 5.5.2 subdir：恢复真实 target 并提取子目录

对于前面在 `MOUNT_PRE` 中建立了 tmptgt 的 subdir 挂载，注册的 `hook_mount_post()` 会继续完成子目录挂载。注意 `hook_mount_post()` 由于是在 `MOUNT_PRE` 中建立的，比 `MNT_STAGE_PREP` 阶段要晚，所以会在 `MNT_STAGE_PREP` 阶段挂载的 `hook_attach_target()` 后面执行，此时 tmptgt 已经挂载。

subdir 的全过程见 3.8.2 小节，这里只是说明其在 `MNT_STAGE_MOUNT_POST` 的执行过程，如下：

```text
mnt_fs_set_target(cxt->fs, org_target)
        │
        ▼
恢复真实 target
        │
        ▼
do_mount_subdir()
        │
        ├── 新 API
        │     open_tree(fd_tree, subdir, CLONE)
        │              │
        │              ▼
        │     切回原 mount namespace
        │              │
        │              ▼
        │     move_mount(子树 → 真实 target)
        │              │
        │              ▼
        │     切回临时 mount namespace
        │
        └── legacy
              mount(tmptgt/subdir, target, MS_BIND)
        │
        ▼
        umount(tmptgt)
        │
        ▼
tmptgt_cleanup()
        │
        ├── 若尚未 umount：umount(tmptgt)
        │     （成功路径上 do_mount_subdir 通常已卸过）
        ├── close(new_ns_fd)
        ├── setns(old_ns_fd) → 回到原 mount namespace
        └── close(old_ns_fd)
```

这里的重点是：把subdir从临时 mount namespace 的 tmptgt放到真实 target，然后unmount tmptgt和close mount namespace。

##### 5.5.3 idmap：设置 ID mapping 并完成挂载

对于前面在 `MNT_STAGE_PREP_OPTIONS` 中处理了 `X-mount.idmap` 的挂载，注册的 `hook_idmap::hook_mount_post()` 会在 `MNT_STAGE_MOUNT_POST` 阶段真正设置 ID mapping。

idmap 的准备过程见 3.5.4 小节，这里只说明其在 `MNT_STAGE_MOUNT_POST` 阶段的执行过程。

`hook_mount_post()` 首先检查是否已有可复用的 `api->fd_tree`：

- **`is_private = 0`**：直接使用 `api->fd_tree`，通过 `mount_setattr()` 设置 ID mapping。随后由 `hook_attach_target()` 完成最终挂载。
- **`is_private = 1`**：没有可复用的 `api->fd_tree`，通过 `open_tree(target, OPEN_TREE_CLONE)` 克隆已挂载的 mount。设置 ID mapping 后，由本函数自行卸载旧 mount，并将克隆的 mount 挂载回 target。

执行过程如下：

```text
hook_idmap::hook_mount_post()
        │
        ▼
是否已有 api->fd_tree？
        │
        ├── 有：is_private = 0
        │     复用 api->fd_tree
        │
        └── 无：is_private = 1
              open_tree(target, CLONE)
        │
        ▼
mount_setattr(fd_tree, MOUNT_ATTR_IDMAP)
        │
        ▼
is_private？
        │
        ├── 0
        │    本函数结束
        │    后续由 hook_attach_target()
        │    执行 move_mount()
        │
        └── 1
             umount2(target, MNT_DETACH)
                    │
                    ▼
             move_mount(fd_tree, target)
```

这里需要注意：**`is_private = 0` 时，idmap hook 只设置 ID mapping，不执行 `move_mount()`**。最终挂载由前面5.5.1节介绍的 `hook_attach_target()` 完成，由该函数判断是否执行 `umount2(target, MNT_DETACH)`。

而 `is_private = 1` 时，原 mount 已经挂载到 target，因此 idmap hook 需要自行克隆、设置 ID mapping，并用克隆后的 mount 替换原 mount。`is_private = 1` 主要用于兼容 legacy `mount(2)` 挂载路径。由于 legacy `mount(2)` 会直接将 mount 挂载到 target，不会产生可供 idmap hook 复用的 `api->fd_tree`，因此 `hook_mount_post()` 需要自行通过 `open_tree(target, OPEN_TREE_CLONE)` 创建 detached clone，再设置 ID mapping，并通过 `umount2()` 和 `move_mount()` 替换原 mount。

##### 5.5.4 propagation：attach 后设置传播属性

`shared`、`slave` 等 propagation 属性由 `hook_set_propagation` 处理。

对于需要 attach 的新 API 路径，传播属性在 mount tree 放到 target 后设置：

```text
move_mount()
      │
      ▼
mount tree 已 attach 到 target
      │
      ▼
hook_set_propagation
      │
      ▼
mount_setattr(propagation = ...)
```

这里需要特别注意：propagation hook 位于 `MNT_STAGE_MOUNT_POST`，与前面 MOUNT 阶段的普通 VFS 属性设置不属于同一个执行位置。

#### 5.6 与事务级 `MNT_STAGE_POST` 的边界

前面介绍的 `MNT_STAGE_MOUNT_POST` 属于 `mnt_context_do_mount()` 内部。

而 `hook_owner` 注册的 `MNT_STAGE_POST` 是更外层的事务级 hook，不属于 `mnt_context_do_mount()` 的内部三个部分。

当整次挂载流程成功完成后，`hook_owner` 才对最终 target 执行相应的 `lchown()`、`chmod()`。

此时 subdir 已恢复真实 target，idmap 和 attach 等挂载处理也已完成。

`X-mount.owner/group/mode` 的参数解析和 hook 注册过程已在前面的 PREP_OPTIONS 章节3.5.3小节中介绍，这里只明确其最终执行时机。

### 6. `update_tabs()`：将挂载信息写入 utab

**本步变动**：`mnt_context_update_tabs()` 根据挂载执行结果，决定是否将 `cxt->update` 中准备好的信息提交到 `/run/mount/utab`。实际更新由 `mnt_update_table(cxt->update, cxt->lock)` 完成，本阶段不再重新生成 `update->fs`。

在第 4 章 `prepare_update()` 中，libmount 已经准备好待更新的 `cxt->update`。因此，本阶段主要负责检查挂载是否成功、避免与外部 helper 重复更新，并完成 utab 写盘。

#### 6.1 检查更新条件

`mnt_context_update_tabs()` 首先检查 NOMTAB 标志，以及 `cxt->update` 是否存在且已准备完成。未满足条件则直接返回。

随后检查实际挂载结果：

- `cxt->syscall_status == 0`：内核挂载调用成功。
- helper 已执行且退出状态为 `0`：外部 helper 挂载成功。

满足其中一个条件，才允许提交更新；否则跳过。

这里与 `prepare_update()` 的区别在于：**`prepare_update()` 允许 syscall 尚未执行时提前准备更新，而 `update_tabs()` 必须确认挂载已经成功，才能写入 utab。**

#### 6.2 避免外部 helper 重复更新

如果外部 helper 已成功执行，并且 utab 可写，libmount 会调用：

`mnt_update_already_done(cxt->update, cxt->lock)`

该函数对 utab 加锁并读取已有记录，然后通过 `mnt_table_find_pair()` 查找是否存在与 `update->fs` 对应的 source/target 记录。

其中，bind mount 优先使用 `BINDSRC` 作为 source 进行匹配。

- 找到匹配记录：认为 utab 已经更新，跳过写盘。
- 未找到匹配记录：由 libmount 继续提交更新。
- 检查失败：返回错误，调用方同样跳过更新。

该机制只判断挂载记录是否存在，不逐项比较 `OPTS`、`ATTRS` 等内容。

#### 6.3 提交 utab 更新

通过上述检查后，libmount 调用：

```c
rc = mnt_update_table(cxt->update, cxt->lock);
```

该函数根据 `cxt->update` 保存的操作类型，执行相应的 utab 更新：

- 普通 mount：添加记录。
- remount：更新已有记录的选项和属性；如果目标记录不存在，则添加记录。
- move mount：修改相关记录的 target。
- umount：删除对应记录。

实际写盘时，libmount 先读取现有 utab，将更新应用到内存中的 `libmnt_table`，然后写入临时文件，最后通过 `rename()` 替换原 utab 文件。

这里采用临时文件和 `rename()`，是为了保证文件替换的原子可见性：其他进程通过 utab 路径打开文件时，只会得到旧版本或新版本，不会读到正在重写的中间内容。如果直接截断并重写原 utab，其他进程就可能读到空文件、部分记录或不完整的记录。

此外，libmount 使用文件锁协调并发更新，防止多个进程同时读取旧 utab、各自修改后相互覆盖。**`rename()` 保证文件替换的原子性，文件锁保证并发读—改—写操作的协调，两者作用不同。**

普通 mount 新增记录的内容来自 `update->fs`，其中 `SRC`、`TARGET`、`ROOT`、`BINDSRC`、`ATTRS`、`OPTS` 等字段的准备过程已在第 4 章介绍，此处不再重复。

