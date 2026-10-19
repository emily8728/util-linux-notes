# Linux mount 内核机制：super_block 与 mount（Linux 6.6）

上一篇介绍了 util-linux 中 `mount` 命令及 libmount 的执行流程。本文接续用户态的挂载过程，聚焦内核中的两个核心对象：`struct super_block` 描述文件系统实例，`struct mount` 描述挂载关系。在此基础上，介绍文件系统实例的创建与复用、挂载树的组织与传播，以及 VFS 如何通过挂载点完成路径解析。

> **版本说明**：上一篇用户态文章基于 util-linux 2.39.3 源码分析；本文主要参考 Linux v6.6 相关的公开技术资料，并结合部分内核源码进行核对。
>
> **代码说明**：文中的结构体仅摘录与主题相关的字段，流程图及标为“示意”的代码用于辅助理解对象关系和执行流程，不代表 Linux v6.6 的完整源码实现。
## 一、super_block
表示一个已激活的文件系统实例 `super_block`，通过这个结构体，它承载 VFS 与具体文件系统实现之间的连接。
```c
/* 下面只列出与本章讨论相关的关键字段，不是完整定义 */
struct super_block {
    struct file_system_type       *s_type;
    struct dentry                 *s_root;
    const struct super_operations *s_op;
    unsigned long                  s_flags;
    struct block_device           *s_bdev;
    void                          *s_fs_info;
};
```
### 1. super_block 的本质及与磁盘 super block 的关系
super_block 是 VFS 定义的通用文件系统实例对象。VFS 负责分配 super_block，具体文件系统在挂载回调中完成初始化。
#### 1.1 磁盘上的 super block
磁盘上的固定位置也有一个 super block，它是整个文件系统的全局磁盘元数据入口。文件系统必须在磁盘上保存一份全局布局描述信息，否则内核无法知道：
- block size
- inode 在哪里
- 元数据结构怎么解释
- 是否有 journal
- 支持哪些特性

以 ext4 为例，磁盘上的 super block 结构为：
```c
/* 只列出与理解 mount / 运行时初始化相关的关键磁盘字段，不是完整定义 */
struct ext4_super_block {
    __le32 s_inodes_count;        /* 磁盘格式：inode 总数 */
    __le32 s_blocks_count_lo;     /* 磁盘格式：低 32 位 block 总数 */
    __le32 s_log_block_size;      /* 磁盘格式：块大小的对数编码 */
    __le32 s_blocks_per_group;    /* 磁盘格式：每个 block group 的 block 数 */
    __le32 s_inodes_per_group;    /* 磁盘格式：每个 block group 的 inode 数 */
    __le32 s_mtime;               /* 磁盘格式：上次挂载时间 */
    __le32 s_wtime;               /* 磁盘格式：上次写入时间 */
    __le16 s_magic;               /* 磁盘格式：ext4 魔数，0xEF53 */
    __le16 s_inode_size;          /* 磁盘格式：磁盘 inode 记录大小 */
    __le32 s_rev_level;           /* 磁盘格式：superblock 修订版本 */
    __le32 s_feature_compat;      /* 磁盘格式：兼容特性位 */
    __le32 s_feature_incompat;    /* 磁盘格式：不兼容特性位 */
    __le32 s_feature_ro_compat;   /* 磁盘格式：只读兼容特性位 */
    /* ... 还有 journal、uuid、挂载计数、错误处理策略等字段 ... */
};
```

ext4 的主 superblock 位于文件系统起始处偏移 1024 字节：
- 若块大小为 1KiB，通常位于 block 1；
- 若块大小大于 1KiB，通常位于 block 0 内部的 1024 字节偏移处。

#### 1.2 mount 时两者如何关联

```text
获取或创建 VFS super_block
    ↓
读取磁盘 ext4_super_block
    ↓
初始化 ext4_sb_info
    ↓
创建 root inode
    ↓
创建 root dentry
    ↓
设置 sb->s_root
```

磁盘上的 ext4 元数据与内存中 `super_block`、`ext4_sb_info` 的对应关系如下：

```text
磁盘
────────────────────────────────────
ext4_super_block
    ├── s_log_block_size
    ├── s_blocks_count_lo
    ├── s_blocks_per_group
    ├── s_inodes_per_group
    ├── s_inode_size
    └── 其它文件系统元数据

ext4_group_desc[]
journal inode


内存
────────────────────────────────────
struct super_block
    │
    ▼
sb->s_fs_info
    │
    ▼
struct ext4_sb_info
    │
    ├── s_es ────────────────► struct ext4_super_block
    │                          (磁盘 superblock 的内存副本)
    │
    ├── s_blocks_count       ← s_es->s_blocks_count_lo/hi
    ├── s_blocks_per_group    ← s_es->s_blocks_per_group
    ├── s_inodes_per_group    ← s_es->s_inodes_per_group
    ├── s_inode_size          ← s_es->s_inode_size
    ├── s_blocksize           ← s_es->s_log_block_size 计算
    ├── s_groups_count        ← 根据磁盘块数和分组参数计算
    ├── s_desc_per_block      ← 根据 block size 计算
    ├── s_group_desc[]        ← 读取 ext4_group_desc[]
    ├── s_journal             ← 加载 journal
    ├── 计数器
    └── mount 参数
```

这里需要区分两类数据：

- **直接读取或转换**：例如 `s_blocks_per_group`、`s_inodes_per_group`、`s_inode_size`，其来源是磁盘 superblock 中的对应字段。
- **计算或初始化**：例如 `s_groups_count`、`s_desc_per_block`、`s_journal`，由磁盘元数据结合文件系统初始化逻辑建立。

### 2. fs_context 与 super_block 的建立、初始化及重新配置
### 2.1 `fs_context`：文件系统挂载与重新配置的上下文

`fs_context` 是 VFS 在创建或重新配置 `super_block` 时使用的上下文，用于保存本次操作的配置参数及相关状态。

对于 ext4，`file_system_type` 中的 `init_fs_context` 指向 `ext4_init_fs_context()`：

```c
static struct file_system_type ext4_fs_type = {
    .owner           = THIS_MODULE,
    .name            = "ext4",
    .init_fs_context = ext4_init_fs_context,
    .kill_sb         = ext4_kill_sb,
    .fs_flags        = FS_REQUIRES_DEV,
};
```

VFS 创建 `fs_context` 后调用 `ext4_init_fs_context()`，该函数主要完成两项工作：

- 分配 `struct ext4_fs_context`，通过 `fc->fs_private` 保存 ext4 专有配置。
- 设置 `fc->ops = &ext4_context_ops`，指定本次配置操作使用的回调函数。

`fs_context` 中与本章有关的主要成员如下：

| 成员 | 作用 |
|---|---|
| `fs_type` | 指向 `ext4_fs_type` |
| `ops` | 指向 `ext4_context_ops` |
| `source` | 文件系统来源，例如 `/dev/sdb1` |
| `sb_flags` | 本次请求的 superblock 标志，例如 `SB_RDONLY` |
| `sb_flags_mask` | 本次明确指定修改的 superblock 标志位 |
| `fs_private` | 指向 `ext4_fs_context`，保存 ext4 专有配置 |
| `root` | 挂载时取得的文件系统根 dentry；重新配置时关联已有文件系统 |

例如，`data=ordered` 等 ext4 专有选项经过解析后，记录在 `ext4_fs_context` 的相关字段中。

**`fs_context` 与 `super_block` 中的flags的区别：** 前者保存本次挂载或重新配置请求，后者保存文件系统实例的实际运行状态。

`fs_context` 的生命周期与 mount API 有关：

- 传统 `mount(2)`：每次 mount 或 remount 创建新的 `fs_context`，操作结束后释放。
- 新 mount API：`fsopen()` 返回文件描述符，通过对应 `struct file->private_data` 引用 `fs_context`，后续 `fsconfig()` 等操作可以继续使用它。一次挂载完成后，其 ext4 私有配置可以先清理，而 `fs_context` 对象仍由文件描述符持有。

### 2.2 `fs_context_operations`：文件系统配置操作函数

`fs_context->ops` 指向 `struct fs_context_operations`，它定义了文件系统如何解析挂载参数、取得文件系统树以及重新配置已有实例。

Linux v6.6 定义了六个回调：

| 回调 | 作用 |
|---|---|
| `free` | 清理文件系统专有的上下文资源 |
| `dup` | 复制文件系统专有上下文 |
| `parse_param` | 解析单个挂载参数 |
| `parse_monolithic` | 解析传统形式的整块挂载参数 |
| `get_tree` | 获取文件系统树，创建或复用 `super_block` |
| `reconfigure` | 重新配置已有的 `super_block` |

ext4 实际注册其中四个：

```c
static const struct fs_context_operations ext4_context_ops = {
    .parse_param = ext4_parse_param,
    .get_tree    = ext4_get_tree,
    .reconfigure = ext4_reconfigure,
    .free        = ext4_fc_free,
};
```

**`ext4_parse_param()`：解析挂载参数**

VFS 将挂载参数传递给该函数，ext4 根据参数名称和值进行解析，并将结果记录到 `ext4_fs_context` 中。

例如，`data=ordered` 会被解析为 ext4 的数据日志模式配置。解析阶段并不意味着配置已经应用到文件系统实例。

**`ext4_get_tree()`：取得文件系统树**

在 mount 路径中，`vfs_get_tree()` 调用 `ext4_get_tree()`，后者通过 `get_tree_bdev()` 获取块设备对应的 `super_block`。

如果需要创建新的 `super_block`，则调用 `ext4_fill_super()` 完成初始化。具体流程见 2.3.1。

**`ext4_reconfigure()`：重新配置已有文件系统**

在文件系统级 remount 中，VFS 通过 `reconfigure_super()` 调用 `ext4_reconfigure()`，将已经解析的配置应用到已有的 ext4 文件系统实例。具体流程见2.3.2。

**`ext4_fc_free()`：清理 ext4 配置上下文**

该函数释放 `fc->fs_private` 中的 ext4 私有配置资源。比如一次挂载完成之后。

### 2.3 mount 与 remount：创建和重新配置 `super_block`

#### 2.3.1 mount：创建和初始化 `super_block`

mount 流程中，VFS 提供通用挂载框架，具体文件系统通过回调函数参与关键步骤的实现。
super_block 在 mount 过程中的创建和初始化过程如图（以ext4为例）：
```text
用户态
────────────────────────
mount -t ext4 /dev/sdb1 /mnt
进入内核
────────────────────────
sys_mount
    ↓
path_mount
    ↓
get_fs_type("ext4") -> ext4_fs_type
    ↓
创建 fs_context
    ↓
ext4_init_fs_context(fc)
    ↓
vfs_get_tree(fc)
    ↓
ext4_get_tree(fc)
    ↓
get_tree_bdev(fc, ext4_fill_super)
    ↓
取得块设备的 dev_t
    ↓
sget_dev(fc, dev_t)
    │
    ├── 找到已有 super_block → 复用
    │
    └── 未找到
          ↓
        alloc_super()
    ↓
setup_bdev_super() → 设置 sb->s_bdev
    ↓
新建 super_block 时：
    ext4_fill_super()
        ├── 分配 ext4_sb_info，设置 sb->s_fs_info
        ├── 设置 sb->s_op
        ├── 创建 root inode
        └── 创建 root dentry，设置 sb->s_root
    ↓
fc->root = dget(sb->s_root)
```

初始化 VFS `super_block`，以 ext4 为例函数为`ext4_fill_super()`。

这里只是展示 `ext4_fill_super()` 及其内部调用函数的关键逻辑。

```c
static int ext4_fill_super(struct super_block *sb,
                           struct fs_context *fc)
{
    struct ext4_sb_info *sbi;
    struct ext4_super_block *es;
    struct inode *root;
    int blocksize, ret;

    /* 1. 分配 ext4 私有信息，设置 sb->s_fs_info */
    sbi = kzalloc(sizeof(*sbi), GFP_KERNEL);
    if (!sbi)
        return -ENOMEM;
    sb->s_fs_info = sbi;

    /*
     * 2. 读取磁盘 superblock，默认位于文件系统起始偏移 1024 字节。
     * 以下直接展开 ext4_load_super() 的关键逻辑。
     */
    es = ...;                  /* 指向从 sb->s_bdev 读入内存的 superblock */
    sbi->s_es = es;

    /* 3. 设置并校验 ext4 魔数 */
    sb->s_magic = le16_to_cpu(es->s_magic);
    if (sb->s_magic != EXT4_SUPER_MAGIC) {
        ret = -EINVAL;
        goto failed;
    }

    /* 4. 计算文件系统 block size，设置 sb->s_blocksize 等字段 */
    blocksize = EXT4_MIN_BLOCK_SIZE << le32_to_cpu(es->s_log_block_size);
    if (!sb_set_blocksize(sb, blocksize)) {
        ret = -EINVAL;
        goto failed;
    }
    /* 如果 block size 改变，实际实现会重新读取磁盘 superblock。 */

    /*
     * 5. 根据磁盘 superblock 初始化运行时参数。
     *
     * sb->s_fs_info ──> ext4_sb_info (sbi)
     *                         |
     *                         +── s_es ──> ext4_super_block (es)
     */

    /* 5.1 初始化元数据校验算法，并验证磁盘 superblock */
    ret = ext4_init_metadata_csum(sb, es);
    if (ret)
        goto failed;

    /* 5.2 应用挂载选项，设置 sb->s_flags */
    ext4_apply_options(fc, sb);

    sb->s_flags = (sb->s_flags & ~SB_POSIXACL) |
                  (test_opt(sb, POSIX_ACL) ? SB_POSIXACL : 0);
    sb->s_flags |= SB_I_VERSION;

    /* 5.3 初始化每组 block / inode 数及相关参数 */
    sbi->s_blocks_per_group = le32_to_cpu(es->s_blocks_per_group);
    sbi->s_inodes_per_group = le32_to_cpu(es->s_inodes_per_group);
    sbi->s_inodes_per_block = sb->s_blocksize / sbi->s_inode_size;

    /* 根据文件系统实际使用的 group descriptor 大小计算 */
    sbi->s_desc_size = ext4_has_feature_64bit(sb)
                     ? le16_to_cpu(es->s_desc_size)
                     : EXT4_MIN_DESC_SIZE;
    sbi->s_desc_per_block = sb->s_blocksize / sbi->s_desc_size;

    /* 计算最大文件大小 */
    sb->s_maxbytes = ext4_max_size(
        sb->s_blocksize_bits, ext4_has_feature_huge_file(sb));

    /* 5.4 根据文件系统总 block 数计算 block group 数量 */
    sbi->s_groups_count =
        (ext4_blocks_count(es) -
         le32_to_cpu(es->s_first_data_block) +
         sbi->s_blocks_per_group - 1) /
         sbi->s_blocks_per_group;

    /* 6. 读取并校验 group descriptors */
    ret = ext4_group_desc_init(sb, es, ...);
    if (ret)
        goto failed;
    /* 成功后 sbi->s_group_desc 关联已读取的块组描述符数据。 */

    /* 7. 设置 VFS 操作函数表 */
    sb->s_op = &ext4_sops;
    sb->s_export_op = &ext4_export_ops;
    sb->s_xattr = ext4_xattr_handlers;

#ifdef CONFIG_FS_ENCRYPTION
    sb->s_cop = &ext4_cryptops;
#endif

#ifdef CONFIG_FS_VERITY
    sb->s_vop = &ext4_verityops;
#endif

#ifdef CONFIG_QUOTA
    sb->dq_op = &ext4_quota_operations;
    sb->s_qcop = ext4_has_feature_quota(sb)
               ? &dquot_quotactl_sysfile_ops
               : &ext4_qctl_operations;
    sb->s_quota_types = QTYPE_MASK_USR | QTYPE_MASK_GRP | QTYPE_MASK_PRJ;
#endif

    /* 8. 从磁盘 superblock 复制文件系统 UUID */
    memcpy(&sb->s_uuid, es->s_uuid, sizeof(es->s_uuid));

    /* 9. 加载 journal，并完成必要的日志恢复 */
    if (!test_opt(sb, NOLOAD) && ext4_has_feature_journal(sb)) {
        ret = ext4_load_and_init_journal(sb, es, ...);
        if (ret)
            goto failed;
    }
    /* 成功后 sbi->s_journal 指向 journal_t 对象。 */

    /* 10. 读取根 inode（EXT4_ROOT_INO = 2） */
    root = ext4_iget(sb, EXT4_ROOT_INO, EXT4_IGET_SPECIAL);
    if (IS_ERR(root)) {
        ret = PTR_ERR(root);
        goto failed;
    }

    /* 校验根 inode 必须是有效目录 */
    if (!S_ISDIR(root->i_mode) || !root->i_blocks || !root->i_size) {
        iput(root);
        ret = -EFSCORRUPTED;
        goto failed;
    }

    /* 11. 根据root创建根 dentry，设置 sb->s_root */
    sb->s_root = d_make_root(root);
    if (!sb->s_root) {
        ret = -ENOMEM;
        goto failed;
    }

    /* 省略后续块分配器、quota、sysfs 等初始化及检查 */
    return 0;

failed:
    /* 示意清理：实际源码还会释放已初始化的相关资源 */
    kfree(sbi);
    sb->s_fs_info = NULL;
    return ret;
}
```

#### 2.3.2 remount：重新配置已有 `super_block`

mount 建立文件系统实例后，仍可以通过 remount 修改部分文件系统配置，而不必重新创建 `super_block`。

例如：

```bash
mount -o remount,ro /mnt
```

Linux v6.6 的传统 remount 主要流程如下：

```text
mount -o remount,ro /mnt
    ↓
mount(2)
    ↓
path_mount()
    ↓
do_remount()                         ← VFS
    ↓
fs_context_for_reconfigure()
    ├── 关联已有 super_block
    └── ext4_init_fs_context()       ← file_system_type->init_fs_context
            ├── 分配 ext4_fs_context
            └── 设置 fc->ops = &ext4_context_ops
    ↓
解析本次 remount 参数
    ├── fc->sb_flags / sb_flags_mask
    └── ext4_parse_param()           ← fs_context_operations->parse_param
    ↓
reconfigure_super()                  ← VFS
    └── ext4_reconfigure()           ← fs_context_operations->reconfigure
            └── __ext4_remount()
                ├── 检查配置与文件系统状态
                ├── 执行必要的 ext4 状态调整
                └── 处理 journal 等相关操作
    ↓
VFS 完成相关挂载状态更新
    ↓
put_fs_context()
    └── ext4_fc_free()               ← fs_context_operations->free
```

其中，`fs_context_for_reconfigure()` 为已有文件系统实例创建临时配置上下文，`ext4_init_fs_context()` 初始化 ext4 专有配置。参数解析完成后，VFS 通过 `reconfigure_super()` 调用 `ext4_reconfigure()`，由 ext4 完成文件系统专有的重新配置操作。

重新配置成功后，原有的 `super_block`、`s_root` 和 `s_fs_info` 仍然存在；本次 remount 创建的 `fs_context` 则通过 `put_fs_context()` 清理。

### 3. super_block 中主要变量解析
#### 3.1 `s_type` —— 文件系统类型对象

`s_type` 表示当前 `super_block` 属于哪种文件系统类型，它指向 `struct file_system_type`。后者描述文件系统类型的名称、挂载配置入口、支持的特性以及文件系统实例的关闭回调。

以 ext4 为例：

```c
static struct file_system_type ext4_fs_type = {
    .owner           = THIS_MODULE,
    .name            = "ext4",
    .init_fs_context = ext4_init_fs_context,
    .parameters      = ext4_param_specs,
    .kill_sb         = ext4_kill_sb,
    .fs_flags        = FS_REQUIRES_DEV | FS_ALLOW_IDMAP,
};
```

执行 `mount -t ext4 /dev/sdb1 /mnt` 时，VFS 首先通过 `get_fs_type("ext4")` 找到已注册的 `ext4_fs_type`，然后创建 `fs_context`，将 `fc->fs_type` 指向 `ext4_fs_type`，再通过 `fc->fs_type->init_fs_context(fc)` 调用 `ext4_init_fs_context()`，初始化 ext4 挂载上下文。见2.3.1小节的流程的开头部分。

后续创建新的 `super_block` 时，VFS 将 `sb->s_type` 设置为 `fc->fs_type`，因此 `sb->s_type` 最终指向 `ext4_fs_type`。如果复用已有的 `super_block`，则不需要重新设置 `s_type`。

各成员作用：

- **`name`**：文件系统类型名称。`get_fs_type("ext4")` 根据该名称查找已注册的文件系统类型；如果尚未注册，还可以尝试通过 `request_module("fs-ext4")` 加载相应模块。

- **`init_fs_context`**：指向 `ext4_init_fs_context()`。VFS 创建 `fs_context` 后调用它，分配 ext4 私有配置上下文，并设置 `fc->ops = &ext4_context_ops`。具体见第 2 节。

- **`parameters`**：指向 ext4 的参数解析规则表 `ext4_param_specs`，用于识别挂载参数的名称、类型及取值。

  例如，`data` 参数涉及以下两个表：

  ```c
  /* 1. data 参数的取值映射表 */
  static const struct constant_table ext4_param_data[] = {
      {"journal",   EXT4_MOUNT_JOURNAL_DATA},
      {"ordered",   EXT4_MOUNT_ORDERED_DATA},
      {"writeback", EXT4_MOUNT_WRITEBACK_DATA},
      {}
  };

  /* 2. ext4_param_specs 参数解析规则表（节选） */
  static const struct fs_parameter_spec ext4_param_specs[] = {
      fsparam_enum("data", Opt_data, ext4_param_data),
      /* 其他参数规则省略 */
  };
  ```

  当用户传入 `data=ordered` 时，`ext4_parse_param()` 调用 VFS 提供的 `fs_parse()`，根据 `ext4_param_specs` 中的规则及 `ext4_param_data` 的映射，将 `ordered` 转换为 `EXT4_MOUNT_ORDERED_DATA`。

  解析结果保存在 `fc->fs_private` 指向的 `ext4_fs_context` 中。

  因此，`ext4_param_specs` 定义**如何解析参数**，`ext4_param_data` 定义 **`data` 参数的字符串取值如何转换**，而 `ext4_fs_context` 保存**本次解析得到的配置结果**。

- **`kill_sb`**：指向 `ext4_kill_sb()`。当文件系统实例的最后一个活动引用被释放时，VFS 调用该回调，关闭文件系统实例并清理其资源。它不一定在每次 `umount` 时执行，因为同一 `super_block` 可能仍被其他 mount 引用。

- **`fs_flags`**：描述文件系统类型的能力，ext4 设置了两个标志：

  - `FS_REQUIRES_DEV`：表示该文件系统需要块设备。对于 `mount -t ext4 /dev/sdb1 /mnt`，`ext4_get_tree()` 通过 `get_tree_bdev()` 取得块设备并创建或复用 `super_block`。`FS_REQUIRES_DEV` 同时用于 VFS 的文件系统类型分类，例如 `/proc/filesystems` 会据此区分需要设备的文件系统与 `nodev` 文件系统。
  - `FS_ALLOW_IDMAP`：表示 ext4 支持 idmapped mount。VFS 在为 mount 设置 ID 映射时检查该标志；如果文件系统类型没有声明支持，则不能为其设置这种 ID 映射。

- **`owner`**：指向提供该文件系统类型的内核模块，ext4 设置为 THIS_MODULE。 VFS 通过 try_module_get()、__module_get() 和 module_put() 管理该模块的引用计数，保证挂载上下文或文件系统实例仍需要 ext4 代码时，模块不能被正常卸载。owner 本身只是模块指针，不是引用计数。

#### 3.2 s_root —— 文件系统实例的根对象
```c
struct dentry *s_root;
```
它保存该文件系统实例的根 dentry，在初始化 `super_block` 时建立。下面的 C 代码是 **`ext4_fill_super()`** 中的示意片段：

```c
//示意，非逐字摘录
root_inode = ext4_iget(sb, EXT4_ROOT_INO);
sb->s_root = d_make_root(root_inode);
if (!sb->s_root)
    return -ENOMEM;
```

对应关系：

1. **`ext4_iget(sb, EXT4_ROOT_INO)`**  
   从磁盘上的 ext4 实例读出根 inode，在内存里得到 `struct inode`。

2. **`d_make_root(root_inode)`**  
   为该 inode 创建根 dentry（`struct dentry`）。

3. **`sb->s_root = …`**  
   把上述根 dentry 记入 `super_block->s_root`，供后续路径解析与 `mnt_root` 等引用。

#### 3.3 s_op —— 文件系统的操作函数
const struct super_operations *s_op 负责文件系统实例级（super_block级）行为，以及 inode 对象在内存中的生命周期管理和回写控制。
```c
/* 只列出本章关心的典型回调，非完整且非精确逐字定义 */
struct super_operations {
    /* 分配 inode 内存对象（通常分配包含 struct inode 的 FS 私有结构） */
    struct inode *(*alloc_inode)(struct super_block *sb);
    /* inode 析构回调；ext4 用于检查 orphan 链表、预留数据块等私有状态是否已清理 */
    void (*destroy_inode)(struct inode *inode);
    /* inode 内存释放回调；ext4 在 RCU 宽限期后释放 ext4_inode_info（包含 struct inode） */
    void (*free_inode)(struct inode *inode);
    /* inode 被标记为 dirty 时通知文件系统；ext4 用于将 inode 元数据修改纳入 journal */
    void (*dirty_inode)(struct inode *inode, int flags);
    /* VFS inode 写回回调；ext4 主要用于同步协调，普通异步写回直接返回 */
    int (*write_inode)(struct inode *inode, struct writeback_control *wbc);
    /* inode 即将回收时调用（释放 page cache、truncate blocks、处理 orphan inode 等） */
    void (*evict_inode)(struct inode *inode);
    /* 卸载文件系统时调用（释放 s_fs_info、关闭 journal、停止后台线程等） */
    void (*put_super)(struct super_block *sb);
    /* 同步整个文件系统（sync/syncfs；wait=1 表示等待写回完成） */
    int (*sync_fs)(struct super_block *sb, int wait);
    /* 返回文件系统统计信息（df/statfs：block 数量、剩余空间、inode 数量等） */
    int (*statfs)(struct dentry *dentry, struct kstatfs *buf);
};
```
下面分为三个部分介绍这些 super block 的操作函数，以ext4作为例子：

##### 3.3.1 与 inode 和 file 的操作函数的区别

注意与 `inode_operations` 和 `file_operations` 的区别：
- `inode_operations`：负责 inode 语义层操作，包括目录树结构修改（create / unlink / rename 等）以及 inode 元数据修改入口（如 setattr）。
- `file_operations`：负责文件打开后的 I/O 行为，即 read/write/mmap/fsync/ioctl 等运行期操作。

**示例 A：create 文件**

调用链：

```text
open("a", O_CREAT)（文件不存在）
    ↓
syscall
    ↓
vfs_create()
    ↓
inode->i_op->create              ← inode_operations
    ↓
ext4_create()
    ↓
分配 inode（内存对象）
    ↓
sb->s_op->alloc_inode            ← super_operations
    ↓
初始化 inode（mode/uid/gid/nlink）
    ↓
修改 metadata（journal transaction）：
    inode bitmap
    inode table（初始 inode）
    directory block
    group descriptor（相关统计信息）
    ↓
dentry 关联 inode
```

这里可以看到：

- `inode_operations`：决定“创建文件名及目录项”的语义入口，ext4 通过 `ext4_create()` 实现。
- `super_operations`：不负责目录操作语义，而是负责 inode 内存对象的分配及后续生命周期管理。

**示例 B：write 文件**

```text

write(fd)
    ↓
syscall
    ↓
vfs_write()                         ← VFS
    ↓
file->f_op->write_iter              ← file_operations
    ↓
ext4_file_write_iter()
    ↓
修改 page cache（文件内容）
    ↓
inode_update_time()                 ← VFS
    │
    ├── inode->i_op->update_time    ← inode_operations
    │   （文件系统实现了该回调时）
    │
    └── generic_update_time()      ← VFS 默认实现
            ↓
        更新 mtime、ctime
            ↓
        __mark_inode_dirty()        ← VFS
            │
            ├── 未启用 lazytime
            │       ↓
            │   I_DIRTY_SYNC
            │       ↓
            │   sb->s_op->dirty_inode  ← super_operations
            │       ↓
            │   ext4_dirty_inode()
            │       ↓
            │   ext4_mark_inode_dirty()
            │       将 inode 修改纳入 journal
            │
            └── 启用 lazytime
                    ↓
                I_DIRTY_TIME
                暂缓时间戳持久化
                此时不调用 dirty_inode

    ↓
文件变长，更新 i_size
    ↓
如果新文件末尾已有有效的磁盘 block 映射：
    更新 i_disksize
    调用 ext4_mark_inode_dirty()
    将 inode 修改纳入 journal
    ↓
write() 返回

--------------------------------------------
后台 writeback（之后执行）

__writeback_single_inode()          ← VFS
    ↓
do_writepages()                     ← VFS
    ↓
mapping->a_ops->writepages          ← address_space_operations
    ↓
ext4_writepages()
    ↓
如果需要为写入的数据分配新 block：
    delalloc 分配 block
    更新 extent、bitmap、inode 块数
    根据 i_size 更新 i_disksize
    调用 ext4_mark_inode_dirty()
    将 inode 修改纳入 journal
    ↓
提交文件数据写盘
    ↓
检查 lazytime 时间戳
    │
    ├── I_DIRTY_TIME 且达到写回条件
    │       ↓
    │   mark_inode_dirty_sync()    ← VFS
    │       ↓
    │   __mark_inode_dirty(I_DIRTY_SYNC)
    │       ↓
    │   sb->s_op->dirty_inode      ← super_operations
    │       ↓
    │   ext4_dirty_inode()
    │       ↓
    │   ext4_mark_inode_dirty()
    │       将时间戳纳入 journal
    │
    └── 未达到条件
            暂不处理延迟时间戳
    ↓
VFS 检查 inode dirty 状态
    │
    ├── 只有 I_DIRTY_PAGES
    │       ↓
    │   不调用 write_inode()
    │
    └── 有 I_DIRTY_SYNC / I_DIRTY_DATASYNC
            ↓
        write_inode()               ← VFS
            ↓
        sb->s_op->write_inode       ← super_operations
            ↓
        ext4_write_inode()
            ↓
        WB_SYNC_NONE
            直接返回

```
先补充一下：Linux内核有后台的 writeback 机制，它的职责是把内存里的脏数据慢慢刷到磁盘，而不是每次写系统调用都立即同步写盘。
三个触发回写的条件：
- A. 后台周期/内存压力触发
- B. 用户显式同步触发（sync/syncfs/fsync）
- C. 卸载、冻结等路径触发

流程图可以看到：

- `file_operations`：决定打开文件之后如何执行写操作。对于 ext4，`write_iter` 对应 `ext4_file_write_iter()`，负责进入文件写入流程。
- `address_space_operations`：负责 page cache 与文件系统之间的数据读写。对于 ext4，`writepages` 对应 `ext4_writepages()`，负责将脏页中的文件数据提交写回。
- `super_operations`：不负责 `write()` 的语义入口，而是提供 inode dirty 通知、inode 写回等文件系统级回调。对于 ext4，inode 元数据主要通过 `ext4_mark_inode_dirty()` 纳入 journal，**不是由 `write_inode()` 直接写入 inode table**。

**1. `mapping->a_ops->writepages` 中的 `mapping` 和 `a_ops`**

- `mapping` 是 `struct address_space *`，通常对应 `inode->i_mapping`，用于管理该文件的 page cache，包括缓存页及其 dirty 状态。
- `a_ops` 是 `mapping` 中的 `struct address_space_operations *`，保存文件系统提供的 page cache 操作函数。
- `mapping->a_ops->writepages` 是 VFS/writeback 调用文件系统写回脏页的接口。对于 ext4，最终进入 `ext4_writepages()`，将文件数据写往磁盘，并在需要时处理 delayed allocation。

**2. `sb->s_op->write_inode` 与 inode dirty 标志的关系**

`I_DIRTY_SYNC`、`I_DIRTY_DATASYNC`、`I_DIRTY_TIME` 都是 VFS 定义的 inode dirty 状态标志，保存在 `inode->i_state` 中。

VFS 的 `__writeback_single_inode()` 在执行 `do_writepages()` 后，根据 inode 的 dirty 状态，只有存在 `I_DIRTY_SYNC` 或 `I_DIRTY_DATASYNC`才调用 `write_inode()`。

**上述 dirty 标志判断属于 VFS 通用机制，不是 ext4 特有的。**

ext4 的特殊之处在于 inode 元数据的持久化方式：

- `sb->s_op->dirty_inode` 对应 `ext4_dirty_inode()`，用于在 VFS 标记 inode 元数据变脏时，将相关修改纳入 journal。
- `ext4_mark_inode_dirty()` 是 ext4 内部函数，负责更新磁盘 inode 对应的元数据 buffer，并将其修改纳入 journal transaction。ext4 在文件大小、块映射等元数据变化时，也会直接调用该函数。
- `sb->s_op->write_inode` 对应 `ext4_write_inode()`。普通异步 writeback（`WB_SYNC_NONE`）调用它时直接返回，不负责再次将 inode 元数据写入 inode table。

因此，**VFS 使用 `I_DIRTY_SYNC`、`I_DIRTY_DATASYNC` 管理 inode 的脏状态和写回调度；ext4 则主要通过 `ext4_mark_inode_dirty()` 配合 JBD2 journal 完成 inode 元数据持久化。** 两者属于不同层次的机制。

**示例 C：unlink 文件**

调用链：
```text

unlink("a")
    ↓
syscall
    ↓
vfs_unlink()
    ↓
dir->i_op->unlink                  ← inode_operations
    ↓
ext4_unlink()
    ↓
ext4_journal_start()
    创建 handle，加入当前 transaction
    ↓
删除目录项 "a"
inode->i_nlink--
    相关元数据修改纳入 transaction
    ↓
判断 link count：

    ├── i_nlink > 0
    │       ↓
    │   更新 inode 元数据
    │   （文件仍有其他硬链接）
    │
    └── i_nlink == 0
            ↓
        ext4_orphan_add()
        登记 orphan，相关修改纳入 transaction

    ↓
ext4_journal_stop()
    结束 handle，不代表 transaction 已提交
    ↓
ext4_unlink() 返回
    ↓
VFS减少inode的内存引用计数：iput()      ← VFS 通用函数
    ↓
判断是否为最后一个引用：

    ├── i_count > 0
    │       ↓
    │   inode 继续保留在内存
    │   （例如文件仍 open）
    │
    └── i_count == 0
            ↓
        iput_final()                 ← VFS 内部函数
            ↓
        判断 i_nlink：

        ├── i_nlink > 0
        │       ↓
        │   通常保留在 inode cache
        │   （需要驱逐时也不会删除磁盘文件）
        │
        └── i_nlink == 0
                ↓
            evict()                  ← VFS 内部函数
                ↓
            sb->s_op->evict_inode    ← super_operations
--------------------------------------------
evict_inode 流程（i_nlink == 0）：

ext4_evict_inode()
    ↓
释放 page cache
    ↓
ext4_journal_start()
    创建 handle，加入当前 transaction
    用于记录 inode 回收的元数据修改
    ↓
ext4_truncate()
    释放文件数据块及 extent 相关资源
    更新 block bitmap 等元数据
    ↓
ext4_xattr_delete_inode()
    清理扩展属性资源
    ↓
ext4_orphan_del()
    移除 orphan 记录
    ↓
ext4_free_inode()
    更新 inode bitmap
    释放磁盘 inode 分配资源
    ↓
ext4_journal_stop()
    结束 handle，不代表 transaction 已提交
    ↓
返回 VFS
    ↓
sb->s_op->destroy_inode            ← super_operations
    ↓
ext4_destroy_inode()
    检查 orphan 链表、预留数据块状态
    不释放 inode 内存
    ↓
call_rcu()
    注册 RCU 延迟释放回调
    ↓
RCU 宽限期结束
    ↓
sb->s_op->free_inode               ← super_operations
    ↓
ext4_free_in_core_inode()
    ↓
kmem_cache_free()
    释放 ext4_inode_info 内存
    （包含 VFS struct inode）
```

这里可以看到：
- `dir->i_op->unlink`：属于 `inode_operations`，负责删除“文件名 → inode”的目录项关联，并减少目标 inode 的 `i_nlink`。
- `sb->s_op->evict_inode`：属于 `super_operations`，负责清理 inode 的 page cache 等内存资源；在本图 `i_nlink == 0` 的删除路径中，还负责释放磁盘上的文件数据块、inode 分配资源及相关元数据资源。
- `sb->s_op->destroy_inode`：检查 ext4 inode 的 orphan 链表、预留数据块等状态是否已正确清理。。
- `sb->s_op->free_inode`：属于 `super_operations`，由 `ext4_free_in_core_inode()` 最终释放内存中的 `ext4_inode_info` 对象，其中包含 VFS 的 `struct inode`。

**示例 D：chmod/chown/touch：通过 `i_op->setattr` 修改 inode 属性**

```text
chmod("a", 0644)
chown("a", uid, gid)
touch("a")
    ↓
syscall
    ↓
notify_change()                      ← VFS
    ↓
inode->i_op->setattr                 ← inode_operations
    ↓
ext4_setattr()
    ↓
修改 inode 属性：
    i_mode / i_uid / i_gid
    i_atime / i_mtime / i_ctime
    ↓
mark_inode_dirty()                   ← VFS
    ↓
__mark_inode_dirty()
    ↓
sb->s_op->dirty_inode                ← super_operations
    ↓
ext4_dirty_inode()
    ↓
ext4_mark_inode_dirty()              ← ext4 内部函数
    将 inode 元数据修改纳入 journal
    ↓
后续 JBD2 commit / checkpoint
    将元数据持久化到 inode table
```

这里可以看到：

- `inode_operations`：提供 inode 属性修改的语义入口，ext4 通过 `ext4_setattr()` 实现。
- `super_operations->dirty_inode`：VFS 标记 inode 元数据变脏时调用，ext4 通过 `ext4_dirty_inode()` 将修改纳入 journal。
- `ext4_mark_inode_dirty()`：负责更新 inode 对应的元数据 buffer，并将修改纳入 journal transaction。ext4 内部已有 journal handle 的路径也会直接调用它。

##### 3.3.2 与 `file_system_type` 的成员函数 `kill_sb` 的区别

请见 3.1 节 `file_system_type` 的成员函数 `kill_sb`。

`kill_sb` 负责关闭不再被使用的文件系统 superblock；`put_super` 则是关闭过程中，由具体文件系统释放自身私有资源的回调。

**两者的区别是：`kill_sb` 负责整个 superblock 的关闭流程，`put_super` 只负责其中的文件系统私有资源清理。**

以 ext4 为例，正常卸载且释放最后一个 superblock 活动引用时：

```text
umount /mnt
    ↓
VFS 解除 mount 与挂载点的关联
    ↓
释放 superblock 的活动引用
    ↓
最后一个活动引用被释放
    ↓
file_system_type->kill_sb           ← file_system_type
    ↓
ext4_kill_sb()
    ↓
kill_block_super()                  ← VFS
    ↓
generic_shutdown_super()            ← VFS
    核心清理：
    - 清理 dentry 缓存
    - 同步文件系统
    - 清除 SB_ACTIVE 标志
    - 回收 inode
    ↓
sb->s_op->put_super                  ← super_operations
    ↓
ext4_put_super()
    释放 ext4 私有资源：
    - 停止并销毁 journal
    - 释放 group descriptor 缓存
    - 释放 s_fs_info 等内部结构
    ↓
返回 kill_block_super()
    同步并释放主块设备引用
    ↓
返回 ext4_kill_sb()
    释放独立 journal 设备引用（如果存在）
    ↓
VFS 完成 superblock 剩余清理
    最终释放 super_block 内存
```

这里可以看到：

- `file_system_type->kill_sb`：负责整个文件系统实例的关闭。ext4 对应 `ext4_kill_sb()`，通过 `kill_block_super()` 完成通用 superblock 关闭及块设备引用释放。
- `generic_shutdown_super()`：负责与具体文件系统无关的 VFS 清理，包括同步文件系统、清理 dentry、回收 inode，以及调用 `put_super()`。
- `super_operations->put_super`：在 `generic_shutdown_super()` 内部调用。ext4 对应 `ext4_put_super()`，负责 journal、group descriptor 缓存、`s_fs_info` 等私有资源的清理。

**注意：** `umount` 只是解除一个挂载关系；只有 superblock 的最后一个活动引用被释放，才会进入 `kill_sb()` 关闭流程。如果仍有其他 mount 使用同一个 superblock，就不会立即调用 `kill_sb()`。

##### 3.3.3 只走 superblock 操作函数的流程

**示例 E：mount：建立 `super_block` 实例**

```text
mount /dev/sda1 /mnt
    ↓
syscall
    ↓
path_mount()                         ← VFS
    ↓
vfs_get_tree()                       ← VFS
    ↓
fc->ops->get_tree                   ← fs_context_operations
    ↓
ext4_get_tree()
    ↓
get_tree_bdev()                      ← VFS
    创建或复用 super_block
    ↓
ext4_fill_super()
    ↓
读取磁盘 superblock
    ↓
初始化：
    sb->s_op
    sb->s_fs_info
    sb->s_root
    ↓
读取 root inode
    ↓
sb->s_op->alloc_inode               ← super_operations
    ↓
ext4_alloc_inode()
```

- `sb->s_op->alloc_inode` 并不是 `ext4_fill_super()` 直接调用的，而是读取 root inode 时，通过 VFS inode 分配流程间接调用的。
- `fc` 是 `struct fs_context *`，表示本次文件系统挂载操作的上下文，保存文件系统类型、挂载参数、待创建的挂载树等信息。
  - `fc->ops` 是 `struct fs_context_operations *`，指向文件系统为该上下文注册的操作函数表。
  - `fc->ops->get_tree` 是其中的回调，用于取得本次挂载所需的文件系统根目录树。对 ext4，该回调对应 `ext4_get_tree()`，随后经 VFS 的 `get_tree_bdev()` 创建或复用 `super_block`，并在需要初始化新实例时调用 `ext4_fill_super()`。

**示例 F：sync / syncfs：`s_op->sync_fs`**

`sync()` 用于同步系统中各文件系统的脏数据和元数据。

```text
sync()
    ↓
ksys_sync()                          ← VFS
    ↓
唤醒 flusher threads
    ↓
iterate_supers()
    同步各 superblock 的 dirty inode
    ↓
iterate_supers()
    ↓
sb->s_op->sync_fs                   ← super_operations
    ↓
ext4_sync_fs()
    提交 journal transaction
    ↓
同步底层块设备
```

`syncfs(fd)` 只同步 fd 所属的文件系统。

```text
syncfs(fd)
    ↓
syscall
    ↓
根据 fd 定位 super_block
    ↓
sync_filesystem(sb)                  ← VFS
    ↓
写回该文件系统的 dirty inode / 数据
    ↓
sb->s_op->sync_fs                   ← super_operations
    ↓
ext4_sync_fs()
    提交 journal transaction
    ↓
等待相关写回完成
    同步底层块设备
```

**示例 G：statfs：`s_op->statfs`**

`statfs` 用于获取整个文件系统的容量和资源统计信息。

```text
stat -f /
    ↓
statfs 系统调用
    ↓
user_statfs()                        ← VFS
    ↓
security_sb_statfs() 等检查
    ↓
sb->s_op->statfs                    ← super_operations
    ↓
ext4_statfs()
    ↓
返回文件系统统计信息：
    blocks
    free blocks
    free inodes
    block size
    filesystem type
```

这里可以看到，`super_operations->statfs` 提供文件系统级统计信息；ext4 通过 `ext4_statfs()` 返回相关统计结果。
#### 3.4 `s_flags` —— super_block 的状态与行为标志

`super_block->s_flags` 是一个位掩码，用于记录文件系统实例的状态及部分通用行为选项。内核通过检查相应标志位，决定是否允许写入、是否采用同步写入方式，以及如何处理文件时间戳等。

这些标志常见的有：

| 标志 | 含义 |
|---|---|
| `SB_RDONLY` | 文件系统实例处于只读状态，不允许普通文件系统写入 |
| `SB_SYNCHRONOUS` | 启用同步写入行为，对应 `sync` 挂载选项 |
| `SB_DIRSYNC` | 目录更新采用同步写入行为，对应 `dirsync` |
| `SB_LAZYTIME` | 延迟将 inode 时间戳更新写入磁盘，对应 `lazytime` |
| `SB_POSIXACL` | 文件系统启用 POSIX ACL 支持 |
| `SB_BORN` | 内核内部状态标志，表示 superblock 已完成初始化并进入可用状态 |
| `SB_ACTIVE` | 内核内部状态标志，表示 superblock 处于活动状态 |

这些标志并非全部直接来自用户的挂载选项。例如，`SB_BORN`、`SB_ACTIVE` 是 VFS 管理 `super_block` 生命周期使用的内部标志。

**`SB_RDONLY` 与只读状态**

`SB_RDONLY` 是 `super_block->s_flags` 中的只读标志，表示整个文件系统实例是否允许普通写入。它可以在首次 mount 时设置，也可以在后续 remount 时改变。

例如：

```bash
mount -t ext4 -o ro /dev/sdb1 /mnt
```

挂载成功后，`sb->s_flags` 包含 `SB_RDONLY`。VFS 可以通过 `sb_rdonly(sb)` 检查该标志：

```c
static inline bool sb_rdonly(const struct super_block *sb)
{
    return sb->s_flags & SB_RDONLY;
}
```

**需要注意**，Linux 的挂载标志分为文件系统实例级和单个 mount 级：

| 保存位置 | 标志示例 | 作用 |
|---|---|---|
| `super_block->s_flags` | `SB_RDONLY`、`SB_SYNCHRONOUS`、`SB_LAZYTIME` | 控制整个文件系统实例的状态和行为 |
| `vfsmount->mnt_flags` | `MNT_READONLY`、`MNT_NOSUID`、`MNT_NODEV`、`MNT_NOEXEC` | 控制单个 mount 的访问限制及行为 |

其中，`nosuid`、`nodev`、`noexec` 只属于单个 mount 的属性，不保存在 `super_block->s_flags` 中。

`ro` 则同时具有两个层次的控制：

- **`SB_RDONLY`**：使整个文件系统实例只读，影响共享该 `super_block` 的所有 mount。
- **`MNT_READONLY`**：仅使当前 mount 只读，不改变整个文件系统实例的读写状态。

VFS 判断某个 mount 是否只读时，会同时检查 `MNT_READONLY` 和 `SB_RDONLY`。只要其中一个设置了只读标志，就不能通过该 mount 进行普通写入。

顺便说明一下，用户传入的挂载参数会根据其作用范围，由 VFS 或具体文件系统分别处理：

- **文件系统实例级参数**：例如 `ro`，VFS 将其记录在 `fs_context->sb_flags`、`sb_flags_mask` 中，并在创建或重新配置 `super_block` 时应用到 `s_flags`。
- **单个 mount 级参数**：例如 `nosuid`、`nodev`、`noexec`，由 VFS 处理，转换为 `MNT_NOSUID`、`MNT_NODEV`、`MNT_NOEXEC`，最终保存在对应 mount 的 `mnt_flags` 中。
- **文件系统专有参数**：例如 ext4 的 `data=ordered`，由 `ext4_parse_param()` 解析，结果保存在 `fs_context->fs_private` 指向的 `ext4_fs_context` 中。首次挂载时，由 `ext4_fill_super()` 将配置应用到 `super_block->s_fs_info` 指向的 `ext4_sb_info`；重新挂载时，则由 `ext4_reconfigure()` 更新已有文件系统实例的配置。

### 3.4 s_bdev —— 该实例使用的底层设备对象

```c
struct block_device *s_bdev;
```

表示这个 filesystem instance 正在使用哪个 block device，例如：

- `/dev/sdb1`
- loop device
- dm-verity device

对于基于块设备的文件系统（如 ext4、XFS），`s_bdev` 指向该实例使用的 `struct block_device`；像 proc、tmpfs 这类非块设备文件系统不按这条路径工作。

先看端到端主流程：

```text
vfs_read()/vfs_write()
    │
    ▼
filesystem (ext4/xfs)
    │ 根据 inode → block mapping 确定要访问的磁盘 block
    │ 普通 buffered I/O 还会经过 page cache
    ▼
创建 bio (struct bio)
    │ 指定 I/O 目标设备：bio->bi_bdev
    │ 例如 ext4 通常使用 sb->s_bdev
    │ 并设置：
    │   bio->bi_iter.bi_sector = 起始扇区
    │   bio->bi_opf = REQ_OP_READ / REQ_OP_WRITE 等
    │   bio_vec = I/O 使用的内存区域
    ▼
submit_bio()
    │
    ▼
┌ - - - - - - - - - - - - - - - - - - - - ┐
| BLOCK LAYER（概要块，细节见下一个流程图）  |
└ - - - - - - - - - - - - - - - - - - - - ┘
    │
    ▼
driver 回调（queue_rq，例如 nvme_queue_rq）
    │
    ▼
driver 实现（nvme / sd / virtio_blk）
    │
    ▼
硬件设备（SSD / HDD / 虚拟磁盘）
```

这里涉及几个主要对象：

**1. `struct block_device`**

表示 I/O 的目标块设备。

例如 ext4 挂载在 `/dev/sdb1` 上，文件系统的 `sb->s_bdev` 就关联这个分区对应的 `block_device`。

**2. `struct bio`**

描述一次块 I/O，包括目标设备、扇区位置、操作类型以及内存数据区域。

一个 `bio` 不一定对应一个最终的 `request`。

**3. `blk-mq`**

即 Multi-Queue Block I/O Queuing Mechanism，是现代 Linux Block Layer 用来管理和派发块 I/O 请求的多队列框架。

**4. `struct request`**

是 blk-mq 用来管理和派发块 I/O 的请求对象。

一个 `request` 可以包含一个或多个 `bio`，驱动通常通过 `queue_rq()` 接收 `request`。

**5. `queue_rq()`**

是设备驱动向 blk-mq 注册的请求提交回调，例如 NVMe 驱动的 `nvme_queue_rq()`。

blk-mq 通过这个回调将 `request` 交给驱动处理。

#### 3.4.2 Block Layer 内部流程

Block Layer 是 Linux 内核负责处理块 I/O 的子系统。它接收文件系统提交的 `bio`，通过 blk-mq 完成请求组织、可选调度及向设备驱动派发。

下面单独展开 BLOCK LAYER 内部流程：

```text
submit_bio()
    │
    ▼
1) submit_bio_noacct()
   - 检查 bio 并处理分区扇区偏移等
   - 普通 blk-mq 设备进入 blk_mq_submit_bio()
    │
    ▼
2) bio 拆分（bio split，按需）
   - 场景：bio 太大，或跨越设备规定的 I/O 边界
   - 例如：1MB bio 超过设备单次 I/O 限制，
     拆成多个较小的 bio
    │
    ▼
3) bio 合并（bio merge，按需）
   - 场景：相邻且兼容的 bio
     （如 sector 100-107 与 108-115）
   - 作用：合并到同一个 request，减少请求数量
    │
    ▼
4) 转换为 request（struct request）
   - 未合并到已有 request 时，分配新 request
   - 一个 request 可以包含一个或多个 bio
    │
    ▼
5) Plug（可选）
   - 暂存当前任务产生的多个 request
   - 尝试进一步合并，随后批量提交
    │
    ▼
6) request_queue（struct request_queue）
   - 一个磁盘设备通常对应一个 request_queue
   - 通过成员指针管理以下对象：
       │
       ├── queue_ctx
       │     └── CPU software queue（struct blk_mq_ctx）
       │
       ├── queue_hw_ctx
       │     └── Hardware dispatch queue（struct blk_mq_hw_ctx）
       │
       └── elevator
             └── I/O scheduler（可选）
    │
    ▼
7) CPU software queue（struct blk_mq_ctx）
   - 每个可能的 CPU 通常对应一个 blk_mq_ctx
   - 保存需要排队的 request
    │
    ▼
8) I/O scheduler（可选）
   - mq-deadline / kyber / bfq
   - 对 request 进行调度
    │
    ▼
9) Hardware dispatch queue（struct blk_mq_hw_ctx）
   - 一个 request_queue 可以有多个 blk_mq_hw_ctx
   - 多个 blk_mq_ctx 可以映射到同一个 blk_mq_hw_ctx
   - 负责协调 request 向驱动派发
    │
    ▼
10) 向 driver 派发 request
    - 调用驱动注册的 queue_rq()
```

上述流程表示主要处理环节，不代表每个请求都必须依次进入这些队列。Plug、软件队列排队和 I/O scheduler 都是条件性路径，blk-mq 也可以直接派发请求。

其中还需要说明几个对象和机制：

**1. `struct request_queue`**

是 Block Layer 为块设备维护的 I/O 请求管理结构体，保存设备 I/O 限制、软件队列、硬件派发队列及 I/O scheduler 等管理信息。

一个磁盘设备通常对应一个 `request_queue`，其分区共享这个队列。例如 `/dev/sda1` 和 `/dev/sda2` 通常共享 `/dev/sda` 的 `request_queue`。

从 `s_bdev` 可以找到对应的 `request_queue`：

```c
struct request_queue *q = sb->s_bdev->bd_disk->queue;
```

它通过以下成员管理队列：

- **`queue_ctx`**：指向 `struct blk_mq_ctx` 数组，包含每个可能的 CPU 对应的软件队列上下文。
- **`queue_hw_ctx`**：指向 `struct blk_mq_hw_ctx *` 数组，数组元素指向各个硬件派发上下文。
- **`nr_hw_queues`**：记录当前硬件派发上下文的数量。
- **`elevator`**：指向当前 I/O scheduler 的 `struct elevator_queue`，未启用调度器时为 `NULL`。

因此，`request_queue` 并不是直接内嵌所有队列结构体，而是通过指针及数组管理它们。

**2. Plug**

是 Block Layer 的批量提交机制。

当当前任务启用了 Plug 时，可以暂存多个 `request`，尝试进一步合并，再集中提交，减少频繁派发的开销。

**3. CPU software queue（`struct blk_mq_ctx`）**

是 blk-mq 按 CPU 建立的软件请求提交上下文。

一个 request_queue 为每个可能的 CPU 分配一个 blk_mq_ctx。例如系统有 8 个可能的 CPU，该 request_queue 就有 8 个这样的上下文。

提交请求时会确定对应的 blk_mq_ctx，但 `request` 不一定经过它的软件等待队列：

- **需要软件排队时**：`request` 进入 `blk_mq_ctx` 的软件等待队列，等待后续派发。
- **可以直接派发时**：`request` 跳过软件等待队列，直接向驱动派发。

**4. Hardware dispatch queue（`struct blk_mq_hw_ctx`）**

是 blk-mq 面向驱动的硬件派发上下文。

一个 `request_queue` 至少关联一个 `blk_mq_hw_ctx`，可以关联多个，其数量主要由驱动通过 `blk_mq_tag_set.nr_hw_queues` 等配置决定。

例如，一个磁盘设备对应：

- 1 个 `request_queue`。
- 8 个 CPU software queue 上下文（假设有 8 个可能的 CPU）。
- 4 个 hardware dispatch queue 上下文（假设驱动配置了 4 个）。

此时，多个 CPU software queue 可以映射到同一个 hardware dispatch queue。

每个普通 blk-mq 请求都需要确定对应的 blk_mq_hw_ctx，但 `request` 不一定经过它的硬件派发等待队列：

- **需要等待派发时**：例如驱动暂时无法接收请求，`request` 可以进入 `blk_mq_hw_ctx` 的 `dispatch` 等待链表，等待后续重新派发。
- **可以直接派发时**：`request` 跳过 `dispatch` 等待链表，通过对应的 `blk_mq_hw_ctx` 直接调用驱动的 `queue_rq()`。

`blk_mq_hw_ctx` 不等同于硬件设备内部真正的命令队列；它是内核中管理请求派发的上下文。

**5. I/O scheduler**

是可选的请求调度组件，用于控制 `request` 的调度顺序、服务分配或并发派发。

常见调度器包括 `mq-deadline`、`kyber`、`bfq`。

一个 `request_queue` 同一时刻最多启用一个 I/O scheduler，也可以不启用（`none`）。启用调度器不代表所有请求都必须经过调度器，例如 passthrough 请求可以绕过调度器。

### 3.5 s_fs_info —— 文件系统私有运行时数据

```c
void *s_fs_info;
```

指向具体文件系统为该 filesystem instance 维护的私有运行时数据（runtime state）。

不同文件系统使用不同的结构体，例如：

- ext4：`struct ext4_sb_info`
- XFS：`struct xfs_mount`
- Btrfs：`struct btrfs_fs_info`

以 ext4 为例，`s_fs_info` 指向 `struct ext4_sb_info`：

```c
struct ext4_sb_info {
    struct buffer_head *s_sbh;
    struct ext4_super_block *s_es;

    unsigned long s_blocks_per_group;
    ext4_group_t s_groups_count;

    unsigned int s_mount_opt;
    unsigned short s_mount_state;
    unsigned long s_ext4_flags;

    struct journal_s *s_journal;

    struct buffer_head * __rcu *s_group_desc;
    struct ext4_group_info ** __rcu *s_group_info;
    struct inode *s_buddy_cache;

    struct blockgroup_lock *s_blockgroup_lock;
    struct mutex s_orphan_lock;

    struct percpu_counter s_freeclusters_counter;
    struct percpu_counter s_freeinodes_counter;

    /* 其他成员 */
};
```

其中：

- `s_sbh`：指向缓存 ext4 磁盘 superblock 所在块的 `buffer_head`。
- `s_es`：指向 `s_sbh` 缓存数据中的 `struct ext4_super_block`，即磁盘 superblock 的内存映像。
- `s_blocks_per_group`：每个 block group 包含的 block 数量。
- `s_groups_count`：文件系统的 block group 总数。
- `s_mount_opt`：保存 ext4 的挂载选项标志。
- `s_mount_state`：保存 ext4 的文件系统状态信息。
- `s_journal`：指向 ext4 使用的 journal 对象。
- `s_group_desc`：指向保存各 block group descriptor 所在缓存块的 `buffer_head` 指针数组。
- `s_group_info`：指向各 block group 的内存分配管理信息，供 ext4 块分配器使用。
- `s_blockgroup_lock`：指向 block group 锁管理结构，用于保护相关元数据操作。
- `s_orphan_lock`：保护 orphan inode 链表相关操作的互斥锁。
- `s_freeinodes_counter`：统计文件系统中的空闲 inode 数量。
- `s_ext4_flags`：保存 ext4 内核运行时状态标志，例如正在调整文件系统大小（RESIZING）、强制关闭（SHUTDOWN）、底层设备支持 DAX（BDEV_IS_DAX）、因错误进入紧急只读状态（EMERGENCY_RO）。磁盘格式的 feature flags 则保存在 `s_es` 指向的磁盘 superblock 中。
- `s_buddy_cache`：指向 ext4 多块分配器使用的缓存 inode，保存各 block group 的 block bitmap 和 buddy 信息。Buddy 用于快速查找连续空闲空间，减少分配时扫描 bitmap 的开销。
- `s_freeclusters_counter`：统计文件系统中的空闲 cluster 数量。Cluster 是 ext4 的磁盘空间分配单位，普通 ext4 中一个 cluster 等于一个 block；启用 `bigalloc` 后，一个 cluster 可以包含多个连续 block。

特点：

- 不直接等同于磁盘 superblock，而是保存文件系统运行时状态，并可引用磁盘 superblock 的内存映像。
- 由具体文件系统负责创建、初始化和释放。
- 生命周期与对应的 VFS `super_block` 绑定，而不是单个 mount。多个 mount 可以共享同一个 `super_block` 及其 `s_fs_info`。

因此，`sb->s_fs_info` 是 **filesystem instance 的文件系统私有运行时数据的主要入口**。

## 二、mount / vfsmount
本文件系统被挂在 VFS 树的位置结构体。例如执行 `mount /dev/sda1 /mnt`，一个 ext4 文件系统实例被挂在目录 `/mnt` 上，这件事就由 `struct mount` 描述。
```c
// 只列出和文中所讲相关主要字段
struct mount {
    struct hlist_node mnt_hash;          // mount hash 节点：用于 (parent_mount, mountpoint_dentry) -> child mount 查找

    struct mount *mnt_parent;            // 父 mount（mount tree 拓扑关系）
    struct dentry *mnt_mountpoint;       // 当前 mount 挂载到父 mount 时对应的 mountpoint dentry

    struct vfsmount mnt;                 // 对外暴露的也就是VFS通用代码使用的挂载对象部分（见下一个结构体）

    /* propagation（挂载传播）相关字段 */
    struct list_head  mnt_share;         // shared peer group 链表（shared mounts 互相连接）
    struct hlist_head mnt_slave_list;    // 当前 mount 作为 master 时，其 slave mounts 链表头
    struct hlist_node mnt_slave;         // 当前 mount 作为 slave 时，挂入 master->mnt_slave_list 的节点
    struct mount      *mnt_master;       // 当前 slave mount 对应的 master mount
    int               mnt_group_id;      // shared peer group 的 group id
    int               mnt_t_flags;       // mount 内部拓扑/propagation 相关 flags

    struct mnt_namespace *mnt_ns;        // 当前 mount 所属的 mount namespace
};
struct vfsmount {
    struct dentry *mnt_root;         //该mount的根dentry
    struct super_block *mnt_sb;      
    int mnt_flags;                   //挂载参数(ro,noexec...)
};
struct dentry {

    struct qstr d_name;      // 当前名字
    struct dentry *d_parent; // 父目录
    struct inode *d_inode;   // 对应 inode
    struct super_block *d_sb;// 所属文件系统
};
```
请注意，这里 `vfsmount` 里面的 `*mnt_sb` 就是指向第一小节说的 `struct super_block`，这就让这两个结构体产生了关联。下面分别介绍其它字段：
### 1. `mnt_root`
```c
struct dentry *mnt_root;
```
`mnt_root` 指向**本次 mount 在该文件系统中的根 dentry**。
挂载整个文件系统时，通常有：
```c
mnt_root = sb->s_root;
```
但 `mnt_root` 并不要求一定等于 `sb->s_root`。如果本次 mount 只暴露文件系统中的某个子树，`mnt_root` 就指向该子树的根 dentry。
例如 bind mount：
```bash
mount --bind /data/log /mnt
```
如果 `/data/log` 不是其所在文件系统的根目录，那么新 mount 的 `mnt_root` 指向 `log` 对应的 dentry，而不是 `sb->s_root`。
Btrfs subvolume mount 也是典型情况。例如：
```bash
mount -o subvol=@home /dev/sda1 /mnt
```
本次 mount 选择 `@home` subvolume 作为挂载内容的根，因此 `mnt_root` 指向所选 subvolume 的根 dentry，而不是简单地认为它一定等于 `sb->s_root`。
因此可以区分：
```text
sb->s_root
    文件系统对应 superblock 的根 dentry

mnt_root
    某一次具体 mount 所使用的根 dentry
```
挂载整个文件系统时两者通常相同；bind mount、Btrfs subvolume mount 等情况下，两者可以不同。

### 2. mnt_mountpoint
```c
struct dentry *mnt_mountpoint;
```
这个文件系统被挂载到哪个目录上。例如：
```bash
mount /dev/sda1 /mnt
```
则 `mnt_mountpoint` 指向 `/mnt` 的 dentry。

### 3. mnt_parent
`mnt_parent` 表示当前这个 mount 是挂在哪个 mount 上面的。也就是说，mount 之间自己也形成一棵树。

所以：
- `mnt_parent` 描述 mount tree 的父子关系
- `mnt_mountpoint` 描述在父 mount 的哪个 dentry 上挂进去
- `mnt.mnt_root` 描述这个子 mount 自己的根 dentry

### 4. mnt_hash
一个系统中可能有几十到几百个 `mount`，容器环境中甚至可能有上千个。如果每次查找都遍历全部 `mount`，效率会很差，所以内核维护了 hash table。`mnt_hash` 就是 `mount` 挂在内核 hash bucket 链表中的节点，用于在路径解析或挂载管理时快速定位对应的 `mount` 对象。它的结构：
```c
struct hlist_node {
    struct hlist_node *next;
    struct hlist_node **pprev;
};
```
特点：有接近双向链表的删除效率，同时节省头节点内存。

既然是 hash table，就可以把它理解成 `key -> bucket -> value`：`key` 主要由 `mount *mnt_parent` 指针和 `dentry *mnt_mountpoint` 指针混合计算得到，经过 hash 后落到某个 bucket；再在 bucket 链表中查找到的 `value`，就是对应的 `struct mount *` 对象。大致代码如下：
```c
unsigned long h = hash(parent, mountpoint);

/* mount_hashtable 是一个数组，每个元素是一个 hlist_head */
struct hlist_head *bucket = &mount_hashtable[h];

hlist_for_each_entry(mnt, bucket, mnt_hash) {
    if (mnt->mnt_parent == parent &&
        mnt->mnt_mountpoint == mountpoint)
        return mnt;
}
```

### 5. mount propagation：挂载传播相关字段

**mount propagation** 描述的是：具有传播关系的 mount 之间，后续新增或移除子挂载等事件如何传播。

典型问题包括：

- clone 出新的 **mount namespace** 之后，宿主机上新增的 mount，要不要反映到新的 namespace 里？
- `bind` / `rbind` 把两棵树拴在一起之后，一侧的 mount / unmount，另一侧是否还要继续跟着变？

内核用 **mount propagation** 这一套策略来回答：哪些 mount 结点之间**互相**、或**单向**同步挂载事件。

#### 5.1 四种传播类型

常用类型如下（可用 `mount --make-*` 调整挂载点或子树的策略）：

| 类型 | 含义 |
|------|------|
| shared | 双向传播 |
| slave | 单向接收传播 |
| private | 完全不传播 |
| unbindable | 不允许再被 bind mount |

```bash
mount --make-shared /path
mount --make-slave /path
mount --make-private /path
mount --make-unbindable /path
```

以下按类型说明。

##### shared

`mount --make-shared /mnt` 之后，`/mnt` 一侧的 mount 成为 **shared mount**：落在同一 **peer group** 里的 mount 之间，会互相转发「子树里新增 / 拆掉 mount」这类变化。

**peer group** 可理解为一组 **mutually shared** 的 mount：`/proc/self/mountinfo` 里形如 `shared:17` 的标记，就与这种分组有关。

示意：

```text
mount A : shared:17
mount B : shared:17
→ A 与 B 在同一 peer group：A 下新增 mount 会传到 B，反之亦然。
```

##### slave

**slave** 只**接收**来自 **master** 的传播，**不**把本侧变化回传。

```text
A (master)
  ↓
B (slave)
```

容器里常见：宿主或上层 mount 树为 master，容器内子树为 slave。

##### private

**private** 不参与传播：既不接收外部事件，也不向其它 mount 播本 subtree 的变化，是最「隔绝」的一种。如果需要将整棵子挂载树都设置为 private，应使用 mount --make-rprivate /mnt。

```bash
mount --make-private /mnt
```

之后 `/mnt` 子树下的 mount 变化不会外传，也不会从外面接收传播。

##### unbindable

**unbindable** 表示该 mount **不能**再作为 bind mount 的源被复制，用于避免某些拓扑下形成难以拆开的绑定关系。执行 recursive bind（`rbind`）时，源树中的 unbindable 子挂载会被排除。与 `T_UNBINDABLE` 等标志对应，详见下一小节。

#### 5.2 与 propagation 相关的 `struct mount` 字段

`struct mount` 中与传播相关的成员可概括为：

1. **`mnt_group_id`**  
   shared **peer group** 的编号；`/proc/self/mountinfo` 里 `shared:X` 一类语义与此对应。

2. **`mnt_share`**  
   把同一 peer group 里的 shared mount 串成链表，是 shared 拓扑的核心链表之一。

3. **`mnt_slave_list` / `mnt_slave` / `mnt_master`**  
   当前 mount 作为 **master** 时，用 `mnt_slave_list` 挂住其 **slave**；当前 mount 作为 slave 时，用 `mnt_slave` 挂入 `master->mnt_slave_list`，并用 `mnt_master` 指回 master。

4. **`mnt_t_flags`**  
   保存传播相关标志（如 `T_SHARED`、`T_UNBINDABLE` 等）。示意（非完整枚举）：

```c
enum {
    T_SHARED           = 1, /* mount is shared */
    T_UNBINDABLE       = 2, /* mount is unbindable */
    T_MARKED           = 4,
    T_UMOUNT_CANDIDATE = 8,
};
```

**与 §5.1 四种传播类型的大致对应：**

- **shared**：`mnt_t_flags` 含 `T_SHARED`，有 `mnt_group_id`，并参与 `mnt_share` 链表。
- **slave**：`mnt_master` 指向 master，`mnt_slave` 挂在 `master->mnt_slave_list` 上。
- **private**：不参与 shared peer group，无上述 slave/master 传播关系。
- **unbindable**：`mnt_t_flags` 含 `T_UNBINDABLE`。


### 6. mnt_ns
**`mnt_ns`**，类型为 `struct mnt_namespace *`，表示**这个 mount 结点登记在哪个 mount namespace 里**（创建/切换时常说 `CLONE_NEWNS`；与 pid、net 等其它 namespace 不是一类东西）。

**mount namespace** 用来隔离进程看到的挂载关系：不同 mount namespace 可有不同挂载关系，因此两个进程即使访问同一路径字符串，也可能看到不同内容。

例如：

```text
mount namespace A
/
├── /proc
├── /sys
└── /mnt → ext4

mount namespace B
/
├── /proc
├── /sys
└── /mnt → overlayfs
```

两个进程都访问 `/mnt`：在 mount namespace A 中 `/mnt` 可能是 ext4；在 mount namespace B 中可能是 overlayfs。

因此，**mount namespace** 隔离的不是 inode、dentry、`super_block` 或磁盘本身，而是进程看到的挂载视图。

内核里，一个 mount namespace（`struct mnt_namespace`）关联一棵 mount tree，对应关系如下（结构体为示意字段）：

```c
/* 简化：只保留理解 mount namespace（struct mnt_namespace）所需的核心字段 */
struct mnt_namespace {
    struct mount *root;   /* 该 mount namespace 中 mount tree 的根 mount */
};
```

```text
struct mnt_namespace
    │
    ▼
struct mount *root   ← 该 mount namespace 内 mount tree 的入口（树顶）
    │
    ├──► struct mount
    └──► struct mount
            └──► …
```

在此基础上，每个进程既关联某个 mount namespace，也持有自己的文件系统上下文 `fs_struct`：

```c
struct task_struct {
    struct nsproxy   *nsproxy;
    struct fs_struct *fs;
};
```

`nsproxy->mnt_ns` 表示当前进程使用哪棵 mount tree（简化，仅保留与 mount namespace 相关部分）：

```c
struct nsproxy {
    struct mnt_namespace *mnt_ns;
};
```

`fs_struct` 给出路径解析起点：

```c
struct fs_struct {
    struct path root;
    struct path pwd;
};
```

- `root`：当前进程视角下的 `/`
- `pwd`：当前工作目录

路径解析时：

```c
if (pathname[0] == '/')
    nd->path = current->fs->root;
else
    nd->path = current->fs->pwd;
```

即：绝对路径从 `current->fs->root` 开始；相对路径从 `current->fs->pwd` 开始。
VFS 在解析路径时，本质始终维护一个当前 `path = (mnt, dentry)`：
```c
struct path {
    struct vfsmount *mnt;
    struct dentry   *dentry;
};
```
假设执行 `mount /dev/sdb1 /mnt`，产生的挂载关系可简化为：
```c
struct mount {
    mnt_parent     = rootfs_mount;
    mnt_mountpoint = dentry("/mnt");  // 位于 rootfs
    mnt.mnt_root   = dentry("/");     // ext4 根目录
    mnt.mnt_sb     = ext4_sb;
};
```
下面看 `cat /mnt/a.txt` 的解析过程：

- **Step 1：初始化起点**（绝对路径从进程 `root` 开始）
    - `path = (rootfs_mnt, dentry("/"))`，即：`mnt = rootfs mount`，`dentry = rootfs` 的 `"/"`。

- **Step 2：解析 `mnt`**
    - 当前：`path = (rootfs_mnt, "/")`
    - 取目录 inode：`dir_inode = d_inode(path.dentry)`
    - 调用 lookup（示意 dcache 未命中，已准备好名称为 `"mnt"` 的 `child_dentry`）：`dir_inode->i_op->lookup(dir_inode, child_dentry, 0)`
    - 得到：`child_dentry = dentry("mnt")`（属于 rootfs）
    - 更新：`path = (rootfs_mnt, dentry("mnt"))`

- **Step 3：检查 mountpoint（关键）**
    - 检查挂载点：`child_mnt = lookup_mnt(&path)`
    - 查找 key：`(parent mount = rootfs_mnt, mountpoint dentry = dentry("/mnt"))`
    - 命中后 crossing mount：`path.mnt = child_mnt`，`path.dentry = child_mnt->mnt_root`
    - 结果：`path = (ext4_mnt, dentry("/"))`
    - 注意：此时 `dentry` 已不再是 rootfs 的 `"mnt"`，而是 ext4 的 `"/"`；路径字符串逻辑仍是 `"/mnt"`，但 VFS 已切入新的 `super_block`。

- **Step 4：解析 `a.txt` 并得到最终 inode**
    - 当前目录：`path = (ext4_mnt, "/")`
    - 取当前目录 inode：`dir_inode = d_inode(path.dentry)`
    - 在 ext4 根目录查找 `"a.txt"`（示意 dcache 未命中，已准备好名称为 `"a.txt"` 的 `child_dentry`）：`dir_inode->i_op->lookup(dir_inode, child_dentry, 0)`
    - 得到：`child_dentry = dentry("a.txt")`，且 `child_dentry->d_inode = inode_of_a_txt`
    - 更新：`path = (ext4_mnt, dentry("a.txt"))`
    - 最终目标：`inode = d_inode(path.dentry)`，即 `inode = inode_of_a_txt`

对应伪代码：

```c
/*
 * 初始状态：
 * rootfs (sb A)
 *   /
 *    └── mnt    (mountpoint)
 * ext4 (sb B)
 *   /
 *    └── a.txt
 */
struct path path;
struct dentry *child_dentry;
struct inode *dir_inode;
struct inode *inode;
struct vfsmount *mounted;

/* Step 0: 起点 "/" */
path.mnt = rootfs_mnt;
path.dentry = rootfs_sb->s_root;

/* Step 1: lookup "mnt" */
dir_inode = d_inode(path.dentry);
child_dentry->d_name = (struct qstr) QSTR_INIT("mnt", 3); /* 示意：VFS 已准备好待查找的 dentry */
dir_inode->i_op->lookup(dir_inode, child_dentry, 0);
path.dentry = child_dentry; /* path = (rootfs_mnt, "/mnt") */

/* Step 2: crossing mount */
mounted = lookup_mnt(&path);
if (mounted) {
    path.mnt = mounted;
    path.dentry = mounted->mnt_root;
    /* path = (ext4_mnt, "/") */
}

/* Step 3: lookup "a.txt" */
dir_inode = d_inode(path.dentry);
child_dentry->d_name = (struct qstr) QSTR_INIT("a.txt", 5); /* 示意：VFS 已准备好待查找的 dentry */
dir_inode->i_op->lookup(dir_inode, child_dentry, 0);
path.dentry = child_dentry; /* path = (ext4_mnt, "a.txt") */

/* Step 4: 得到最终 inode */
inode = d_inode(path.dentry); /* inode 属于 ext4 */
```

这里要注意的是：**路径 lookup 的起点**由 `fs_struct` 里的 `root` / `pwd` 这一对 `path` 决定（进程视角下的根与工作目录），而不是默认从 `mnt_ns->root` 那条挂载树入口出发。

顺带一提：`struct mnt_namespace` 里的 mount tree 根（`mnt_ns->root` 一类语义）描述的是该 mount namespace 内的整棵挂载树如何组织与管理，常见用途包括：

- 作为该 mount namespace 下 mount tree 的拓扑入口；
- 支持对该 mount namespace 下 mount tree 的层级遍历；
- 参与 mount namespace 的 clone / destroy，与挂载视图的生命周期绑定；
- 通过 `/proc/.../mountinfo` 等接口向用户态导出当前挂载视图。

## 三. super_block 与 mount 的整体关系
从内核对象看，挂载需要把一个文件系统实例与挂载关系关联起来：`super_block` 描述文件系统实例，`mount` 引用它并描述本次挂载在挂载树中的位置。并非每次执行 `mount` 都新建 `super_block`：已有实例可被复用，bind mount 也会引用已有实例。因此，一个 `super_block` 可以对应多个 `mount`。
例如：
```bash
mount /dev/sda1 /mnt/a
mount /dev/sda1 /mnt/b
```
内核对象关系：
```text
super_block (ext4 on sda1)
├── mount A
│   └── /mnt/a
└── mount B
    └── /mnt/b
```
即：一个磁盘文件系统可被挂载多次，产生多个 `mount`，但通常共享同一个 `super_block`。但是，多个 mount 是否共享同一个 super_block，取决于文件系统实例是否被复用。

是否复用已有 `super_block`，取决于文件系统类型及其查找、匹配和创建规则。需要特别区分：

- **VFS mount 属性**（例如 `noatime`、`nosuid`、`nodev`、`noexec`）可以因 `mount` 而异，不要求为每个挂载创建新的 `super_block`。
- **文件系统级参数与状态**由具体文件系统管理；重复挂载时能否复用实例、是否接受参数变化，取决于该文件系统实现。
- **不同 mount namespace** 可以拥有不同的 `mount` 拓扑，同时引用相同的 `super_block`。
- **bind mount** 引用已有文件系统实例并创建新的挂载关系，不会因为 bind 本身重新建立磁盘文件系统实例。
- **tmpfs、overlayfs 等** 是否创建独立文件系统实例，应按具体挂载方式与实现判断。

> 本文重点是两个核心对象及其关系，不以完整还原 ext4 和块层所有调用分支为目标。
