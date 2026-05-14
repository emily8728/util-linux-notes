# util-linux `mount` 代码阅读笔记
就 `mount` 命令而言，`util-linux` 的核心作用，是将用户输入与系统配置转化为可执行的挂载参数并交由内核执行，同时完成用户态结果记录。源码里面分为main函数（在mount.c中）和libmount库两层。

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

#### option list 与 mount flags

**要点：`main()` 里大多数 option 并不会立刻等于内核 `mount` flags。**

例如：

```bash
mount -o bind,ro /src /mnt
```

在 `main()` 这一层，`bind` 与 `ro` 通常仍是 **option string**，经 `append_option` 进入 **`optlist`**；后续 libmount 再区分：

- 哪些映射为 **`MS_*`**；
- 哪些属于 **文件系统私有** 参数；
- 哪些只在 **用户态**（helper、loop、`x-*` 等）处理。

| option 类型 | 示例 | 作用位置 |
|-------------|------|----------|
| VFS mount flags | `ro`、`bind`、`slave`、`remount` | 后续映射为 `MS_RDONLY`、`MS_BIND`、`MS_SLAVE`、`MS_REMOUNT` 等 |
| filesystem-specific | `data=ordered`、`compress=zstd` | 交给具体文件系统（如 ext4、btrfs）解析 |
| userspace | `user`/`users`、`loop`、`X-mount.*`、helper 相关 | libmount、loop、systemd 或 `mount.<type>` 在用户态处理 |

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

不会作为普通 option 留在 optlist 中，而是由 main() 单独记录，最后通过：

mnt_context_set_mflags(cxt, MS_MOVE);

直接设置对应的 mount flag。

```c
mnt_context_set_mflags(cxt, MS_MOVE);
```

### 2. restricted 与 setuid 安全模型

当前很多 Linux 发行版里的 `/bin/mount` 仍然可能带有 setuid root。

因此普通用户执行 mount 时，进程一开始可能会拥有 `effective uid = root`。

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

> “当前 euid=root，  
> 就完全按真正 root 用户处理”

而是引入 restricted 语义。

它的本质是：

> 虽然 mount 当前暂时带有 setuid root 权限，  
> 但这次挂载仍然被视为：  
> “普通用户发起的受限挂载”

因此在 restricted 模式下，libmount 会额外做一层用户态限制与检查。

典型包括：

| restricted 下的行为 | 目的 |
|---------------------|------|
| 某些 option 或参数组合会触发 `suid_drop()` | 不允许普通用户继续借 setuid mount 执行高风险挂载 |
| 普通用户显式指定完整 source + target 时可能 `suid_drop()` | 避免普通用户继续不通过 fstab 借 setuid mount 直接发起完整挂载 |
| `sanitize_paths()` 按真实用户权限 canonicalize 路径 | 防止借 root 权限解析本来无权访问的路径 |
| 后续 libmount 会检查 user/users/owner/group 等 fstab 授权 | 普通用户不能自由挂载任意对象 |
| 某些 helper/FUSE 路径会放弃 restricted 会话 | 改由 helper 自己的权限模型继续处理 |

这里最关键的一点是：`suid_drop()` 并不等于 mount 立即失败退出。它真正表示的是：放弃 setuid mount 的特权代理身份。之后程序仍然会继续执行，只是后续已经不再依赖 mount 自己的 setuid 权限。

因此，传统 filesystem 与 FUSE / userspace helper 在 `suid_drop()` 之后的表现会不同。

例如：

- ext4
- xfs
- btrfs

最终通常仍需要 `mount(2)` + `CAP_SYS_ADMIN`。因此一旦 `suid_drop()`，普通用户后续大概率无法真正完成挂载。

而：

- FUSE
- sshfs
- 部分 userspace helper

本来就支持普通用户挂载。因此即使已经 `suid_drop()`，后续仍可能成功，因为真正继续工作的已经不是 setuid mount，而是 helper 自己的权限模型。例如 `fusermount3` 本身就支持普通用户挂载。

### 3. 参数解析：option 与位置参数是两层

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
    -> mount_all()
    -> 若带 remount：remount_all()

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

这里有一个容易忽略的限制：位置参数阶段可以补全尚未指定的 source 或 target，但不能重复指定已经存在的 source 或 target。源码并不会先判断两个值是否相同；只要同一个语义位置被指定了两次，就会按用法错误处理。例如：mount -L DATA UUID=xxxx或者mount --source /dev/sda1 /dev/sda1 /mnt都被认为是错误。

#### restricted下显式给出source和target

restricted 对这段逻辑的影响不是“改变 source/target 的含义”，而是：当受限用户显式给出了完整 source + target 时，main() 会倾向于 suid_drop()，不再让这次命令继续借 setuid mount 代理身份执行。

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

在 restricted 场景下会 suid_drop()。

#### restricted下真实用户能否解析路径

此后如果当前会话仍处于 `restricted` 状态，`main()` 还会调用：

```c
sanitize_paths(cxt)
```

这是因为历史上的 `mount` 程序通常带有 `setuid root` 属性，普通用户执行时，进程一度会拥有 `root` 的 `effective uid`。这样如果不额外检查，用户就可能借助 `mount` 的特权去解析自己本来无权访问的路径，例如：

- 指向他人目录的符号链接
- 无法遍历的目录
- 无权限访问的挂载点路径

本质上相当于：

> “借 setuid mount 的权限，  
> 偷看路径最终解析到了哪里”

因此，在 `restricted` 会话下，`sanitize_paths()` 会调用：

```c
canonicalize_path_restricted()
```

按调用者的真实权限，对 `target` 与 `srcpath` 做 `canonicalize` 和路径检查，而不是直接沿用 `setuid root` 的能力。

如果 `canonicalize` 失败，说明：

> 在不借助 setuid 权限的情况下，  
> 当前用户其实无法合法解析这条路径。

这时 `main()` 不会立刻报错，而是再次：

```c
suid_drop(cxt);
```

放弃 `setuid mount` 的代理身份。之后这次 `mount` 是否还能继续成功，就不再依赖用户态的 `setuid` 代理，而由后续真实用户权限以及内核策略决定。


