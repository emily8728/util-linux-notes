这个章节主要描述 mount 对于内核的意义，它会在内核中建立两大重要结构体：
# 一、super_block
表示一个已激活的文件系统实例 `super_block`，通过这个结构体，它承载 VFS 与具体文件系统实现之间的连接。
```c
/* 下面只列出与本章讨论相关的关键字段，不是完整定义 */
struct super_block {
    struct file_system_type       *s_type;
    struct dentry                 *s_root;
    const struct super_operations *s_op;
    struct block_device           *s_bdev;
    void                          *s_fs_info;
};
```
## 1. super_block 的本质及与磁盘 super block 的关系
super_block 是 VFS 定义的通用文件系统实例对象。VFS 负责分配 super_block，具体文件系统在挂载回调中完成初始化。
### 1.1 磁盘上的 super block
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

### 1.2 mount 时两者如何关联

```text
读取磁盘 superblock block
    ↓
解析 ext4_super_block
    ↓
分配 VFS super_block
    ↓
填充字段：s_root, s_fs_info
```

其中，填充 `s_fs_info` 的过程见下图：
```text
磁盘
────────────────────
ext4_super_block   (磁盘格式)
ext4_group_desc
journal inode

内存
────────────────────
struct super_block   (VFS对象)
    │
    ▼
sb->s_fs_info
    │
    ▼
struct ext4_sb_info   (ext4私有信息)
    │
    ├── s_es
    │      │
    │      ▼
    │   struct ext4_super_block
    │   (磁盘superblock的内存副本)
    ├── s_groups_count
    ├── s_blocks_per_group
    ├── s_inodes_per_group
    ├── s_inode_size
    ├── s_blocksize
    ├── s_desc_per_block
    ├── s_group_desc[]
    ├── s_journal
    ├── 计数器
    └── mount参数
```

## 2. super_block 的建立和初始化
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
sget_fc(fc, ...)
    ↓
如果不存在已有 super_block:
     alloc_super()
    ↓
设置：
     sb->s_type
     sb->s_fs_info (初始)
     sb->s_id
    ↓
打开 block device
    ↓
sb->s_bdev = block_device
    ↓
调用：
ext4_fill_super(sb, fc)
    ↓
ext4 读取磁盘 superblock
初始化 ext4_sb_info
创建 root inode
创建 root dentry
```

以 ext4 为例的 ext4_fill_super：
```c
/*
 * 示例代码：只展示 ext4 super_block 初始化的关键逻辑顺序，
 * 不是源码原样；为了把关系讲清楚，这里把部分中间 helper 的逻辑直接展开。
 */
static int ext4_fill_super(struct super_block *sb, struct fs_context *fc)
{
     struct ext4_sb_info *sbi;
     struct ext4_super_block *es;
     struct inode *root_inode;
     unsigned long blocksize;
     int ret;

     /*
      * 1. 为 ext4 私有 superblock 运行时结构分配空间
      *
      * 关系：
      *   sb->s_fs_info --> struct ext4_sb_info
      */
     sbi = kzalloc(sizeof(*sbi), GFP_KERNEL);
     if (!sbi)
          return -ENOMEM;
     sb->s_fs_info = sbi;

     /*
      * 2. 从底层块设备读取磁盘 superblock
      *
      * 这里依赖：
      *   sb->s_bdev 已经在更早的 get_tree_bdev()/sget_fc() 阶段绑定好
      */
     es = ext4_read_superblock(sb);
     if (IS_ERR(es)) {
          ret = PTR_ERR(es);
          goto fail_sbi;
     }

     /*
      * 3. 校验磁盘 superblock 的基本合法性
      */
     if (es->s_magic != cpu_to_le16(EXT4_SUPER_MAGIC)) {
          ret = -EINVAL;
          goto fail_sbi;
     }

     /*
      * 4. 把磁盘 superblock 的内存表示保存到 ext4 私有信息中
      *
      * 关系：
      *   sb->s_fs_info --> ext4_sb_info --> s_es --> ext4_super_block
      */
     sbi->s_es = es;

     /*
      * 5. 根据 es 初始化 ext4 运行时参数
      *
      * 这些值有两类：
      *   (1) 直接来自磁盘 superblock 字段
      *   (2) 根据磁盘字段计算出的运行时值
      */

     /* 5.1 block size */
     blocksize = EXT4_MIN_BLOCK_SIZE << le32_to_cpu(es->s_log_block_size);
     if (!sb_set_blocksize(sb, blocksize)) {
          ret = -EINVAL;
          goto fail_sbi;
     }
     sbi->s_blocksize = blocksize;

     /* 5.2 每组 block / inode 数，直接来自磁盘 superblock */
     sbi->s_blocks_per_group = le32_to_cpu(es->s_blocks_per_group);
     sbi->s_inodes_per_group = le32_to_cpu(es->s_inodes_per_group);
     sbi->s_inode_size = EXT4_INODE_SIZE(sb);

     /* 5.3 group descriptor 相关运行时值 */
     sbi->s_desc_per_block = blocksize / sizeof(struct ext4_group_desc);
     sbi->s_groups_count = ext4_count_groups(sb, es);

     /*
      * 5.4 读取/建立 group descriptor 的运行时表示
      *
      * 这里示意为一个 helper；真实源码里会有更细的读取与校验流程
      */
     ret = ext4_load_group_desc(sb);
     if (ret)
          goto fail_sbi;

     /*
      * 5.5 journal 初始化
      *
      * 真实 ext4 里 journal 相关初始化是挂载过程的重要部分，
      * 包括外置 journal / 内置 journal / recovery 等分支。
      */
     ret = ext4_load_and_init_journal(sb, es, fc);
     if (ret)
          goto fail_sbi;

     /*
      * 6. 设置 super_block 级别操作表
      *
      * 从这里开始，这个 VFS super_block 才具备 ext4 的 superblock 级行为。
      */
     sb->s_op = &ext4_sops;

     /*
      * 7. 读取根 inode（ext4 中通常是 inode 2）
      */
     root_inode = ext4_iget(sb, EXT4_ROOT_INO);
     if (IS_ERR(root_inode)) {
          ret = PTR_ERR(root_inode);
          goto fail_sbi;
     }

     /*
      * 8. 用根 inode 创建根 dentry
      */
     sb->s_root = d_make_root(root_inode);
     if (!sb->s_root) {
          ret = -ENOMEM;
          goto fail_sbi;
     }
     return 0;

fail_sbi:
     kfree(sbi);
     sb->s_fs_info = NULL;
     return ret;
}
```

## 3. super_block 中主要变量解析
### 3.1 s_type —— 文件系统类型对象
s_type 表示：这个 super_block 属于哪种文件系统类型，它指向 `struct file_system_type`，后者描述该文件系统类型在 VFS 中的入口与基本属性。简化定义如下：
```c
/* 只列出与本章相关的关键字段，非完整定义 */
struct file_system_type {
    const char *name;
    int (*init_fs_context)(struct fs_context *);
    void (*kill_sb)(struct super_block *);
    struct module *owner;
};
```
执行 `mount -t ext4 /dev/sdb1 /mnt` 时，VFS 会先根据这个名字找到已注册的文件系统类型对象 `ext4_fs_type`，后续在创建 `super_block` 时，再令 `sb->s_type = &ext4_fs_type`。`ext4_fs_type` 如下：
```c
/* 只列出与本章相关的关键字段，非完整定义 */
static struct file_system_type ext4_fs_type = {
    .owner           = THIS_MODULE,
    .name            = "ext4",
    .init_fs_context = ext4_init_fs_context,
    .kill_sb         = kill_block_super,
};
```
变量作用：
- **`name`**：表示文件系统类型名称。
- **`init_fs_context`**：提供挂载初始化入口；在新挂载模型里，VFS 先调用它建立并初始化 `fs_context`，后续再进入该文件系统自己的 get_tree / fill_super 路径（见1.2节中的流程图）。
- **`kill_sb`**：提供卸载入口；卸载时由 VFS 调用它释放该文件系统实例相关资源。
- **`owner`**：记录提供该类型的内核模块；通过它维护引用计数，防止文件系统仍在使用时模块被卸载。

### 3.2 s_root —— 文件系统实例的根对象
```c
struct dentry *s_root;
```
它保存该文件系统实例的根 dentry。这个对象：
- 在 mount 时建立（`s_root` 是该 super_block 对应文件系统实例的根 dentry，第二章中的 `mnt_root` 通常引用它）
- 在整个 mount 生命周期存在
- 被 mount / namespace / path 解析使用

### 3.3 s_op —— 文件系统的操作函数
const struct super_operations *s_op 负责文件系统实例级（super_block级）行为，以及 inode 对象在内存中的生命周期管理和回写控制。
```c
/* 只列出本章关心的典型回调，非完整且非精确逐字定义 */
struct super_operations {
    /* 分配 inode 内存对象（通常分配包含 struct inode 的 FS 私有结构） */
    struct inode *(*alloc_inode)(struct super_block *sb);
    /* 释放 inode 私有结构（extents/xattr/quota 等；通常连同 struct inode 一起 slab 释放） */
    void (*destroy_inode)(struct inode *inode);
    /* 释放 inode 的 in-core 内存对象（通常对应 slab 对象）；只负责内存释放，不涉及 page cache、block 或 orphan 清理（这些由 evict_inode 处理） */
    void (*free_inode)(struct inode *);
    /* 将 inode 元数据写回磁盘（size/mtime/block mapping 等；数据块写回走 address_space_operations） */
    int (*write_inode)(struct inode *inode, struct writeback_control *wbc);
    /* inode 即将回收时调用（释放 page cache、truncate blocks、处理 orphan inode 等） */
    void (*evict_inode)(struct inode *inode);
    /* 卸载文件系统时调用（释放 s_fs_info、关闭 journal、停止后台线程等） */
    void (*put_super)(struct super_block *sb);
    /* 同步整个文件系统（sync/syncfs；wait=1 表示等待写回完成） */
    int (*sync_fs)(struct super_block *sb, int wait);
    /* mount -o remount 时调用（修改 mount 参数，如 journal mode / quota / ro/rw） */
    int (*remount_fs)(struct super_block *sb, int *flags, char *data);
    /* 返回文件系统统计信息（df/statfs：block 数量、剩余空间、inode 数量等） */
    int (*statfs)(struct dentry *dentry, struct kstatfs *buf);
};
```
下面分为三个部分介绍这些 super block 的操作函数：
**3.3.A 与 inode 和 file 的操作函数的区别**

注意与 `inode_operations` 和 `file_operations` 的区别：
- `inode_operations`：负责 inode 语义层操作，包括目录树结构修改（create / unlink / rename 等）以及 inode 元数据修改入口（如 setattr）。
- `file_operations`：负责文件打开后的 I/O 行为，即 read/write/mmap/fsync/ioctl 等运行期操作。

**示例 A：create 文件**

调用链：
```text
open("a", O_CREAT)
    ↓
syscall
    ↓
vfs_create
    ↓
inode->i_op->create              ← inode_operations
        ↓
        分配 inode（内存对象）
            ↓
        sb->s_op->alloc_inode    ← super_operations

        ↓
        初始化 inode（mode/uid/gid/nlink）

        ↓
        修改 metadata（transaction）：
            inode bitmap
            inode table（初始 inode）
            directory block
            group desc
            superblock

        ↓
        dentry 关联 inode
```

这里可以看到：
- `inode_operations`：决定“创建这个名字和目录结构”的语义入口。
- `super_operations`：并不负责目录语义，而是负责 inode 内存对象的分配与后续生命周期管理。

**示例 B：write 文件**

```text
write(fd)
    ↓
syscall
    ↓
vfs_write
    ↓
file->f_op->write_iter           ← file_operations
        ↓
        修改 page cache（文件内容）

        ↓
        更新内存 inode：
            i_size
            mtime
            ctime

        ↓
        mark_inode_dirty
--------------------------------------------        
writeback（稍后）
ext4_writepages                  ← delalloc 真正分配 block
        ↓
        修改 metadata：
            block bitmap
            extent tree
            group desc

        ↓
        submit_bio（数据块写盘）

    ↓
sb->s_op->write_inode            ← super_operations
        ↓
        写 inode table：
            size / mtime / blocks
```

可以看到：
- `file_operations`：决定打开文件之后“怎么写内容”。
- `super_operations`：不负责 write 的语义入口，但负责把 inode 的最终状态写回 inode table，并配合 writeback / journal 完成持久化。

**示例 C：unlink 文件**

调用链：
```text
unlink("a")
    ↓
syscall
    ↓
vfs_unlink
    ↓
inode->i_op->unlink             ← inode_operations
        ↓
        删除目录项：
            directory block 删除 "a"
            inode->i_nlink--

        ↓
        判断 link count：

        ├── i_nlink > 0
        │       ↓
        │   仅更新：
        │       link count
        │       ctime
        │   （结束）
        │
        └── i_nlink == 0
                ↓
            判断是否仍被使用（是否还有打开引用）：

            ├── i_count > 0（文件仍 open）
            │       ↓
            │   加入 orphan 跟踪路径
            │       ↓
            │   等待最后一个引用释放：
            │       ↓
            │   i_count → 0
            │       ↓
            │   sb->s_op->evict_inode    ← super_operations
            │
            └── i_count == 0（无引用）
                    ↓
                直接进入回收：
                    ↓
                sb->s_op->evict_inode    ← super_operations
--------------------------------------------
evict_inode 流程：
evict_inode
    ↓
释放 page cache
    ↓
释放数据块（extent tree / block bitmap）
    ↓
更新 inode bitmap
    ↓
完成 inode 删除
    ↓
sb->s_op->destroy_inode
    （释放 filesystem 私有 inode 资源，如 ext4_inode_info）
    ↓
sb->s_op->free_inode
    （释放 inode 对象本身的内存）
```

这里可以看到：
- `inode->i_op->unlink`：负责目录树语义，删除“名字 → inode”的目录项关系，并减少 `i_nlink`。
- `sb->s_op->evict_inode`：在 inode 生命周期真正结束时，负责回收 page cache、数据块、inode 及相关 metadata。
- `sb->s_op->destroy_inode / free_inode`：负责 inode 内存对象的析构与最终释放。

**示例 D：chmod/chown/touch：i_op->setattr 改语义**

chmod("a", 0644)
chown("a", uid, gid)
touch("a")

```text
syscall
    ↓
vfs_setattr
    ↓
inode->i_op->setattr                  ← inode_operations
        ↓
        修改内存 inode 语义状态：
            i_mode / i_uid / i_gid
            i_atime / i_mtime / i_ctime
            （必要时还有 i_size）

        ↓
        根据修改类型分两种情况：

        A. 当下已经最终确定、且不属于高频写入状态的属性
           例如：
               mode / uid / gid
               touch 直接设定的时间戳
           这类通常会在 syscall 路径中立即进入 transaction
           直接更新 inode table 对应的 metadata buffer

        B. 高频变化、或更适合延迟合并的 inode 状态
           例如某些 write 相关导致的：
               i_size / i_mtime / i_ctime / i_blocks
           这类更常见的是：
               先更新内存 inode
               mark_inode_dirty
               后续 writeback 再由 sb->s_op->write_inode 持久化
--------------------------------------------
如果走延迟写回路径：
writeback
    ↓
sb->s_op->write_inode                 ← super_operations
        ↓
        把当前 inode 状态写入 inode table
```

这里可以看到：
- `inode->i_op->setattr` 是 inode 属性修改的语义入口。
- 但并不是所有 `setattr` 修改都会统一推迟到 `write_inode`。
- 对于 mode / uid / gid / 明确设定的时间戳这类当下已最终确定、且不属于高频连续变化的属性，filesystem 通常会在 syscall 的 transaction 中立即更新 inode metadata。
- 对于 write 路径中频繁变化的 inode 状态（如 `i_size / i_mtime / i_ctime / i_blocks`），更常见的是先修改内存 inode 并 mark dirty，后续再由 `sb->s_op->write_inode` 统一写回 inode table。

**3.3.B 与 file_system_type 的成员函数 kill_sb 的区别**

请见 3.1 节 `file_system_type` 的成员函数 `kill_sb`，它负责把一个 `super_block` 从“正在使用”变成“彻底销毁”。
`put_super` 是具体文件系统释放 `super_block` 私有资源的回调函数。
看 unmount 流程：
```text
umount /mnt
    ↓
VFS 调用 file_system_type->kill_sb(sb)
    ↓
关闭文件系统对底层设备的使用
（例如不再允许新的读写）
    ↓
调用 generic_shutdown_super(sb)
        ↓
        核心清理：
        - 把 super_block 标记为正在关闭
        - 从全局 superblock 链表中摘除
        - 停止 writeback 等后台行为
        - 处理挂载点关系（umount 逻辑）
        - 清理 dentry / inode 引用关系（进入回收路径）
    ↓
调用 sb->s_op->put_super(sb)
        ↓
        文件系统释放私有资源：
        - s_fs_info
        - journal
        - group descriptor 缓存
        - 其它内部结构
    ↓
释放底层设备引用（例如关闭 /dev/sda1 的引用）
    ↓
最终释放 super_block 本身内存
```

**3.3.C 只走 super block 操作函数的流程**

**示例 E：mount：建立 super_block 实例**

```text
mount /dev/sda1 /mnt
    ↓
syscall
    ↓
path_mount
    ↓
vfs_get_tree
    ↓
ext4_fill_super
        ↓
        读取磁盘 superblock

        ↓
        初始化：
            sb->s_op
            sb->s_fs_info
            sb->s_root

        ↓
        创建 root inode：
            sb->s_op->alloc_inode
```

**示例 F：writeback：s_op->write_inode**

Linux内核有后台的 writeback 机制，它的职责是把内存里的脏数据慢慢刷到磁盘，而不是每次写系统调用都立即同步写盘。
三个触发回写的条件：
- A. 后台周期/内存压力触发
- B. 用户显式同步触发（sync/syncfs/fsync）
- C. 卸载、冻结等路径触发
在其中会调用 `s_op->write_inode`。
```text
dirty inode
    ↓
VFS / writeback 发现该 inode 需要回写
    ↓
writeback_sb_inodes()
    ↓
__writeback_single_inode()
    ↓
sb->s_op->write_inode              ← super_operations
        ↓
        文件系统把当前 inode 状态编码到磁盘 inode 对应的 metadata buffer
        ↓
        （ext4/journal 模式下）把该 metadata buffer 纳入当前 journal transaction
        ↓
        后续由 journal commit / checkpoint 推进到真正磁盘位置
```

**示例 G：sync / syncfs：s_op->sync_fs**

sync()的作用：同步系统中所有文件系统的脏数据，这是全局范围。它会唤醒 flusher/writeback 机制，并推动所有 superblock 对应的脏状态同步。
```text
sync()
    ↓
iterate_supers
    ↓
sb->s_op->sync_fs
```

syncfs(fd)的作用：只同步 fd 所在的那个文件系统，不是全局同步。
```text
syncfs(fd)
    ↓
syscall
    ↓
根据 fd 找到对应 file
    ↓
定位到该 file 所属的 super_block
    ↓
sync_filesystem(sb)
    ↓
sb->s_op->sync_fs                ← super_operations
        ↓
        该 filesystem 的脏 inode / 脏 metadata / journal
        被推进到一致、可持久化状态
```

**示例 H：statfs：s_op->statfs**

statfs是汇总整个 filesystem 的容量与资源统计。
```text
stat -f /
    ↓
VFS 统计该路径所在的文件系统
    ↓
sb->s_op->statfs                 ← super_operations
        ↓
        返回该 filesystem 的统计信息：
            blocks
            free blocks
            free inodes
            block size
            filesystem type
```

**示例 I：remount：s_op->remount_fs**

remount 不是重新创建文件系统，而是修改一个已经存在的 super_block 的运行时状态（例如只读/读写、mount 选项等）。
```text
mount -o remount,ro /mnt
    ↓
syscall
    ↓
mount(2) with MS_REMOUNT
    ↓
VFS 识别为 remount 现有挂载
    ↓
拿到对应的 super_block
    ↓
sb->s_op->remount_fs             ← super_operations
        ↓
        校验新的挂载参数
        ↓
        更新 super_block 的挂载状态
        ↓
        更新 filesystem 私有运行时配置
```

### 3.4 s_bdev —— 该实例使用的底层设备对象
struct block_device *s_bdev;

表示这个 filesystem instance 正在使用哪个 block device，例如：
- /dev/sdb1
- loop device
- dm-verity device

对于基于块设备的文件系统（如 ext4/xfs），`s_bdev` 指向该实例所使用的底层 block device；像 proc、tmpfs 这类非块设备文件系统不按这条路径工作。先看端到端主流程：
```text
vfs_read()/vfs_write()
    │
    ▼
filesystem (ext4/xfs/btrfs)
    │ 根据 inode → block mapping 计算要访问的磁盘 block
    ▼
创建 bio (struct bio)
    │ 指定 I/O 目标设备：bio->bi_bdev = sb->s_bdev
    │ 其中 block_device 可表示 /dev/sda1
    │ 并设置：
    │   bio->bi_iter.bi_sector = 起始扇区
    │   bio->bi_opf = READ / WRITE
    │   bio_vec = 内存页
    ▼
submit_bio()
    │
    ▼
┌ - - - - - - - - - - - - - - - - - - - - ┐
| BLOCK LAYER（概要块，细节见下一个流程图）  |
└ - - - - - - - - - - - - - - - - - - - - ┘
    │
    ▼
driver 实现（nvme / sd / virtio_blk / loop）
    │
    ▼
driver 回调（queue_rq，例如 nvme_queue_rq）
    │
    ▼
硬件设备（SSD / HDD / 虚拟磁盘）
```

BLOCK LAYER是内核里把“文件系统的块读写请求”统一转换、调度并提交给具体设备驱动的一层。下面单独展开 BLOCK LAYER 内部流程：
```text
submit_bio()
    │
    ▼
1) generic_make_request()
   入口函数，bio 从这里进入 block layer
    │
    ▼
2) bio 拆分（bio split）
   - 场景：bio 太大或跨越设备边界
   - 例如：1MB bio 拆成多个 bio
    │
    ▼
3) bio 合并（bio merge）
   - 场景：多个连续 bio（如 sector 100-107 与 108-115）
   - 作用：提高吞吐量
    │
    ▼
4) 转换为 request（struct request）
   - request 是真正发送给 driver 的对象
   - 一个 request 可能包含多个 bio
    │
    ▼
5) 放入 request_queue（struct request_queue）
   - 每个 block_device 对应一个 queue
   - 路径：sb->s_bdev → block_device → request_queue
    │
    ▼
6) I/O scheduler 调度
   - 目标：减少寻道 / 提高吞吐量 / 控制延迟
   - 常见算法：mq-deadline / kyber / bfq
    │
    ▼
7) 派发 request 给 driver
   blk_mq_dispatch_rq_list()
```

### 3.5 s_fs_info —— 文件系统私有运行时数据
```c
void *s_fs_info;
```
这是最典型的 runtime state，常见指向：`ext4_sb_info`、`xfs_mount`、`btrfs_fs_info` 等。

例如 ext4：
```c
struct ext4_sb_info {
    struct buffer_head *s_sbh;
    unsigned long s_mount_state;
    unsigned long s_blocksize;
    ...
};
```
这些数据通常包括：
- block size
- 挂载参数解析结果
- journal 状态
- feature flags
- 元数据缓存
- 各种锁
- 统计信息

特点：
- 不直接等同于磁盘 superblock
- 是内核运行时结构
- 生命周期 = mount 生命周期

因此，`sb->s_fs_info` 是 runtime state 的主要承载者。

# 二、mount / vfsmount
本文件系统被挂在 VFS 树的位置结构体。例如执行 `mount /dev/sda1 /mnt`，一个 ext4 文件系统实例被挂在目录 `/mnt` 上，这件事就由 `struct mount` 描述。
```c
// 只列出和文中所讲相关主要字段
struct mount {
    struct hlist_node mnt_hash;     
    struct mount *mnt_parent;         //父mount
    struct dentry *mnt_mountpoint;    //挂载点dentry
    struct vfsmount mnt;              //见下一个结构体
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
## 1. mnt_root
```c
struct dentry *mnt_root;
```
该 mount 在该文件系统中的根目录。通常 `mnt_root = sb->s_root`，但 bind mount 时不同：
```bash
mount --bind /data/log /mnt
```
此时 `mnt_root` 指向 `/data/log` 对应的 dentry，而非文件系统根。
## 2. mnt_mountpoint
```c
struct dentry *mnt_mountpoint;
```
这个文件系统被挂载到哪个目录上。例如：
```bash
mount /dev/sda1 /mnt
```
则 `mnt_mountpoint` 指向 `/mnt` 的 dentry。

## 3. mnt_parent
`mnt_parent` 表示当前这个 mount 是挂在哪个 mount 上面的。也就是说，mount 之间自己也形成一棵树。

所以：
- `mnt_parent` 描述 mount tree 的父子关系
- `mnt_mountpoint` 描述在父 mount 的哪个 dentry 上挂进去
- `mnt.mnt_root` 描述这个子 mount 自己的根 dentry

## 4. mnt_hash
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
## 5. mount 在路径解析中的作用
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
    - 调用 lookup：`child = dir_inode->i_op->lookup(dir_inode, path.dentry, "mnt")`
    - 得到：`child = dentry("mnt")`（属于 rootfs）
    - 更新：`path = (rootfs_mnt, dentry("mnt"))`

- **Step 3：检查 mountpoint（关键）**
    - 检查挂载点：`child_mnt = lookup_mnt(path.mnt, path.dentry)`
    - 查找 key：`(parent mount = rootfs_mnt, mountpoint dentry = dentry("/mnt"))`
    - 命中后 crossing mount：`path.mnt = child_mnt`，`path.dentry = child_mnt->mnt_root`
    - 结果：`path = (ext4_mnt, dentry("/"))`
    - 注意：此时 `dentry` 已不再是 rootfs 的 `"mnt"`，而是 ext4 的 `"/"`；路径字符串逻辑仍是 `"/mnt"`，但 VFS 已切入新的 `super_block`。

- **Step 4：解析 `a.txt` 并得到最终 inode**
    - 当前目录：`path = (ext4_mnt, "/")`
    - 取当前目录 inode：`dir_inode = d_inode(path.dentry)`
    - 在 ext4 根目录查找 `"a.txt"`：`child = dir_inode->i_op->lookup(dir_inode, path.dentry, "a.txt")`
    - 得到：`child = dentry("a.txt")`，且 `child->d_inode = inode_of_a_txt`
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
struct dentry *child;
struct inode *dir_inode;
struct inode *inode;
struct mount *mounted;

/* Step 0: 起点 "/" */
path.mnt = rootfs_mnt;
path.dentry = rootfs_sb->s_root;

/* Step 1: lookup "mnt" */
dir_inode = d_inode(path.dentry);
child = dir_inode->i_op->lookup(dir_inode, path.dentry, "mnt");
path.dentry = child; /* path = (rootfs_mnt, "/mnt") */

/* Step 2: crossing mount */
mounted = lookup_mnt(path.mnt, path.dentry);
if (mounted) {
    path.mnt = &mounted->mnt;
    path.dentry = mounted->mnt.mnt_root; /* path = (ext4_mnt, "/") */
}

/* Step 3: lookup "a.txt" */
dir_inode = d_inode(path.dentry);
child = dir_inode->i_op->lookup(dir_inode, path.dentry, "a.txt");
path.dentry = child; /* path = (ext4_mnt, "a.txt") */

/* Step 4: 得到最终 inode */
inode = d_inode(path.dentry); /* inode 属于 ext4 */
```

## 6. mount namespace
`mount namespace` 用来隔离进程看到的挂载视图：进程访问 `/`、`/proc`、`/mnt`、`/sys` 等路径时，结果不是全局唯一的一份挂载关系，而是由它所在的 `mount namespace` 决定。不同 namespace 可有不同挂载关系，因此两个进程即使访问同一路径字符串，也可能看到不同内容。

例如：

```text
namespace A
/
├── /proc
├── /sys
└── /mnt → ext4

namespace B
/
├── /proc
├── /sys
└── /mnt → overlayfs
```

两个进程都访问 `/mnt`：在 namespace A 中 `/mnt` 可能是 ext4；在 namespace B 中可能是 overlayfs。

因此，`mount namespace` 隔离的不是 inode、dentry、`super_block` 或磁盘本身，而是进程看到的挂载视图。

内核里，一个 `mount namespace` 关联一棵 mount tree。简化示意：

```c
/* 简化：只保留理解 mount namespace 所需的核心字段 */
struct mnt_namespace {
    struct mount *root;   /* 该 namespace 中 mount tree 的根 mount */
};
```

```text
struct mnt_namespace
        │
        ▼
   struct mount *root          ← 该 namespace 内 mount tree 的入口（树顶）
        │
        ├──► struct mount（子节点）
        └──► struct mount（子节点）
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

`struct path` 表示某个 mount 中的某个目录位置：

```c
struct path {
    struct vfsmount *mnt;
    struct dentry   *dentry;
};
```

因此要注意：**路径 lookup 的起点**并不是 `namespace->root`，而是 `fs_struct` 里的 `root` / `pwd` 这一对 `path`（即进程视角下的根与工作目录）。

顺带一提：`mnt_namespace` 所关联的 mount tree 根节点（`mnt_namespace->root` 一类语义）描述的是该 namespace 内的整棵挂载树如何组织与管理，常见用途包括：

- 维护该 namespace 内的挂载拓扑（mount tree 的组织与变更）；
- 在该 namespace 内遍历 mount tree（枚举、查找挂载关系）；
- 处理挂载传播（propagation）相关语义；
- 参与 namespace 的 clone / destroy（与挂载视图的生命周期绑定）；
- 通过 `/proc/.../mountinfo` 等接口向用户态导出当前挂载视图。

# 三. super_block 与 mount 的整体关系
简单来说，mount 命令在内核里做的事情是：先建立文件系统实例 `super_block`，再建立挂载点实例 `mount`，然后把 `super_block` 和 `mount` 关联起来，也就是把文件系统挂载到挂载点上。所以，一个 `super_block` 可以对应多个 `mount`。
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

通常情况下：
- 同一设备 + 相同 mount 参数
    → 复用同一个 `super_block`

但在以下情况下：
- 不同 mount 参数（如 ro/rw、noatime 等）
    → 可能创建新的 `super_block`
- 不同 mount namespace
    → 会产生不同的 `mount` 结构，但通常仍可共享同一个 `super_block`
- 某些特殊挂载方式（如 bind mount、tmpfs、overlayfs 等）
    → 可能只创建新的 `mount`，或创建独立的 `super_block`，行为取决于具体文件系统实现

