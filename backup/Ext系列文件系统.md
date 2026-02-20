## 1. 磁盘
### 1.1 磁盘特性
磁盘是计算机中唯一的机械设备
很慢，但是造价便宜，容量大，常被各大互联网公司用来存储数据
### 1.2磁盘结构

<img width="707" height="550" alt="Image" src="https://github.com/user-attachments/assets/18370cbb-be17-4cbd-b773-4f7c1c569642" />

这里的磁头臂就属于机械设备，把这块圆盘当作多个半径不同的同心圆（磁道）来看待，磁头臂左右摆动以让磁头对齐某个同心圆
同时，这里的磁头臂就和立体停车场一样，拥有Z轴，<mark>磁头臂上的所有磁头是同步行动的</mark>。
可以把这里的所有圆面看成离散化的圆柱，可得磁盘寻址最外层单位为柱面，也就是在每个圆面上半径相同的圆。
然后在确定地址在哪个Z轴的圆面上，最后，一个圆可以被分成均匀的扇形块，被称作 sector 扇区，圆本身可以转动从而让磁头对准它想要的那个扇区。一个扇区一般 512 字节。
> 这里稍微简化一下，认为每个磁道的扇区大小与数量都相同好了
### 1.3 CHS && LBA地址
CHS 是磁盘的物理地址，对应 Cylinder - Header - Sector，是磁盘物理上的地址。
操作系统才不需要这个！在它看来，物理地址就该是线性的
操作系统需要 名叫 LBA - Logical Block Address 的下标映射。
他们的相互转换十分简单，稍微想想就能明白...
需要注意：
- 扇区编号通常为 1 - base
- 柱面编号与磁道编号为 0 - base
- LBA 使用 0 -base
所以...
- CHS -> LBA：
   - 单个柱面的扇区数 = 磁道扇区数(m) * 磁头数 = n
   - LBA = 柱⾯号 * 单个柱⾯的扇区总数 + 磁头号 * 磁道扇区数 + 扇区号 - 1
   - 也就是 LBA = C * n + H * m + S - 1
- LBA -> CHS：
   - C = LBA // n
   - H = (LBA % n) // m
   - S = (LBA % m) + 1
> 操作系统只关心 LBA 地址，转换的工作就交给硬件啦
## 2. 文件系统前置
### 2.1 块
文件系统是操作系统关于对磁盘/硬盘管理工作的部分
硬盘与磁盘是典型的 块 设备，操作系统不用扇区作为基本单位，这样效率也太低，操作系统通常一个一个块地读取，一个块的扇区数量是由 格式化 决定的，我们之后会谈到这个，最常见 8 个扇区组成一块，正好 8 * 512 = 4 KB
> 每个扇区拥有 LBA ，所以自然能简单地得到块的逻辑地址
- 块号 = LBA / 8
- LBA = 块号 * 8 + S - 1
### 2.2 分区
我们平时用的 Windows 系统有盘符的概念，C 盘，D盘之类的，可以想到这些盘内部相对独立，成为了存储的单位，可是就算笔记本上只有一块固态硬盘还是能一直分区，很容易想到这种其实是操作系统主动做的一层。
分区最小单位为一个柱面，也就是说任何一个柱面最多属于一个分区。
> 其实 Linux 不这么干，没有盘符的概念
### 2.3 inode！
Linux 一切皆文件，文件 = 文件数据 + 文件属性
一个普通文件的属性包括且不仅限于...
- 模式
- 硬链接数
- 文件所有者
- 组
- 大小
- 最后修改时间
> 这些属性能用 ls 命令与 stat 命令查询
Linux中的 文件数据与文件属性分开存储，存储文件属性的被叫做 inode 索引节点
```cpp
/*
 * Structure of an inode on the disk
 */
struct ext2_inode {
    __le16  i_mode;         /* File mode */
    __le16  i_uid;          /* Low 16 bits of Owner Uid */
    __le32  i_size;         /* Size in bytes */
    __le32  i_atime;        /* Access time */
    __le32  i_ctime;        /* Creation time */
    __le32  i_mtime;        /* Modification time */
    __le32  i_dtime;        /* Deletion Time */
    __le16  i_gid;          /* Low 16 bits of Group Id */
    __le16  i_links_count;  /* Links count */
    __le32  i_blocks;       /* Blocks count */
    __le32  i_flags;        /* File flags */
    union {
        struct {
            __le32  l_i_reserved1;
        } linux1;
        struct {
            __le32  h_i_translator;
        } hurd1;
        struct {
            __le32  m_i_reserved1;
        } masix1;
    } osd1;                         /* OS dependent 1 */
    __le32  i_block[EXT2_N_BLOCKS]; /* Pointers to blocks */
    __le32  i_generation;           /* File version (for NFS) */
    __le32  i_file_acl;             /* File ACL */
    __le32  i_dir_acl;              /* Directory ACL */
    __le32  i_faddr;                /* Fragment address */
    union {
        struct {
            __u8    l_i_frag;       /* Fragment number */
            __u8    l_i_fsize;      /* Fragment size */
            __u16   i_pad1;
            __le16  l_i_uid_high;   /* these 2 fields */
            __le16  l_i_gid_high;   /* were reserved2[0] */
            __u32   l_i_reserved2;
        } linux2;
        struct {
            __u8    h_i_frag;       /* Fragment number */
            __u8    h_i_fsize;      /* Fragment size */
            __le16  h_i_mode_high;
            __le16  h_i_uid_high;
            __le16  h_i_gid_high;
            __le32  h_i_author;
        } hurd2;
        struct {
            __u8    m_i_frag;       /* Fragment number */
            __u8    m_i_fsize;      /* Fragment size */
            __u16   m_pad1;
            __u32   m_i_reserved2[2];
        } masix2;
    } osd2;                         /* OS dependent 2 */
};

/*
 * Constants relative to the data blocks
 */
#define EXT2_NDIR_BLOCKS                12
#define EXT2_IND_BLOCK                  EXT2_NDIR_BLOCKS
#define EXT2_DIND_BLOCK                 (EXT2_IND_BLOCK + 1)
#define EXT2_TIND_BLOCK                 (EXT2_DIND_BLOCK + 1)
#define EXT2_N_BLOCKS                   (EXT2_TIND_BLOCK + 1)

/* 备注：EXT2_N_BLOCKS = 15 */
```
这里特别提一嘴，目录在Linux中也是文件，如果要在某个目录创建文件则需要目标目录的写权限
文件名并不属于文件本身的属性，它被归类于目录文件的文件数据中
## 3. ext2 文件系统
### 3.1 MBR 主引导记录
磁盘的第 0 号扇区，也就是第一块扇区，被用作 MBR
- 前 446 字节，这里存放着引导加载程序，主板 BIOS 亮机后的第一件事就是把这块区域从磁盘拉到内存里去执行
- 中间 64 字节，分区表，记录这块磁盘被分作几块分区
- 最后 2 字节，正常情况下永远是魔法数字 0x55AA，如果 BIOS 读到其他数字就会报错 No bootable device
MBR 之外的所有区域用作分区
### 3.2 Boot Sector 引导扇区
MBR 是整个磁盘的起点，引导扇区就是每个分区的起点，占 1024 字节，两块扇区
MBR 可以通过查询引导扇区来读取每个分区的具体文件系统，这里的代码更加复杂
> [!NOTE]
> 不管这个分区是不是纯粹的数据区，甚至装没装操作系统，每个分区的引导扇区在ext系列文件系统中永久存在

每个分区除了引导扇区之外的区域就是ext系列文件系统了！
### 3.3 块组
ext系列文件系统由多个块组构成，分而治之
接下来的内容就算是重难点了
## 4. 块组内部结构
按照先后顺序...
### 4.1 超级块（Super Block）
```cpp
/*
 * Structure of the super block
 */
struct ext2_super_block {
    __le32  s_inodes_count;        /* Inodes count */
    __le32  s_blocks_count;        /* Blocks count */
    __le32  s_r_blocks_count;      /* Reserved blocks count */
    __le32  s_free_blocks_count;   /* Free blocks count */
    __le32  s_free_inodes_count;   /* Free inodes count */
    __le32  s_first_data_block;    /* First Data Block */
    __le32  s_log_block_size;      /* Block size */
    __le32  s_log_frag_size;       /* Fragment size */
    __le32  s_blocks_per_group;    /* # Blocks per group */
    __le32  s_frags_per_group;     /* # Fragments per group */
    __le32  s_inodes_per_group;    /* # Inodes per group */
    __le32  s_mtime;               /* Mount time */
    __le32  s_wtime;               /* Write time */
    __le16  s_mnt_count;           /* Mount count */
    __le16  s_max_mnt_count;       /* Maximal mount count */
    __le16  s_magic;               /* Magic signature */
    __le16  s_state;               /* File system state */
    __le16  s_errors;              /* Behaviour when detecting errors */
    __le16  s_minor_rev_level;     /* minor revision level */
    __le32  s_lastcheck;           /* time of last check */
    __le32  s_checkinterval;       /* max. time between checks */
    __le32  s_creator_os;          /* OS */
    __le32  s_rev_level;           /* Revision level */
    __le16  s_def_resuid;          /* Default uid for reserved blocks */
    __le16  s_def_resgid;          /* Default gid for reserved blocks */

    /*
     * These fields are for EXT2_DYNAMIC_REV superblocks only.
     *
     * Note: the difference between the compatible feature set and
     * the incompatible feature set is that if there is a bit set
     * in the incompatible feature set that the kernel doesn't
     * know about, it should refuse to mount the filesystem.
     *
     * e2fsck's requirements are more strict; if it doesn't know
     * about a feature in either the compatible or incompatible
     * feature set, it must abort and not try to meddle with
     * things it doesn't understand...
     */
    __le32  s_first_ino;           /* First non-reserved inode */
    __le16  s_inode_size;          /* size of inode structure */
    __le16  s_block_group_nr;      /* block group # of this superblock */
    __le32  s_feature_compat;      /* compatible feature set */
    __le32  s_feature_incompat;    /* incompatible feature set */
    __le32  s_feature_ro_compat;   /* readonly-compatible feature set */
    __u8    s_uuid[16];            /* 128-bit uuid for volume */
    char    s_volume_name[16];     /* volume name */
    char    s_last_mounted[64];    /* directory where last mounted */
    __le32  s_algorithm_usage_bitmap; /* For compression */

    /*
     * Performance hints. Directory preallocation should only
     * happen if the EXT2_COMPAT_PREALLOC flag is on.
     */
    __u8    s_prealloc_blocks;     /* Nr of blocks to try to preallocate*/
    __u8    s_prealloc_dir_blocks; /* Nr to preallocate for dirs */
    __u16   s_padding1;

    /*
     * Journaling support valid if EXT3_FEATURE_COMPAT_HAS_JOURNAL set.
     */
    __u8    s_journal_uuid[16];    /* uuid of journal superblock */
    __u32   s_journal_inum;        /* inode number of journal file */
    __u32   s_journal_dev;         /* device number of journal file */
    __u32   s_last_orphan;         /* start of list of inodes to delete */
    __u32   s_hash_seed[4];        /* HTREE hash seed */
    __u8    s_def_hash_version;    /* Default hash version to use */
    __u8    s_reserved_char_pad;
    __u16   s_reserved_word_pad;
    __le32  s_default_mount_opts;
    __le32  s_first_meta_bg;       /* First metablock block group */
    __u32   s_reserved[190];       /* Padding to the end of the block */
};
```
> 最后还填充了190个字节以凑满1KB，类似的操作很多

超级块存放着整个分区的文件系统信息
- s_inodes_count 此分区的所有块组的 inode 总量 
- s_blocks_count 此分区所有块组中的块总量
- s_free_blocks_count 此分区空闲的块数量
- s_free_inodes_count 此分区空闲 inode 的数量
- s_blocks_per_group 此分区每个块组的块总量
- s_inodes_per_group 此分区每个块组的 inode 总量
- s_inode_size 此分区 inode 结构体的大小
等等...这些超级块中的信息管理和描述着一个分区的文件系统，破坏了超级块的信息等同于破坏了文件系统
> [!NOTE]
> 超级块有可能存在于一个分区的多个块组中，为了备份


### 4.2 GDT（Group Descriptor Table）
```cpp
/*
* Structure of a blocks group descriptor
*/
struct ext2_group_desc
{
    __le32 bg_block_bitmap; /* Blocks bitmap block */
    __le32 bg_inode_bitmap; /* Inodes bitmap */
    __le32 bg_inode_table; /* Inodes table block*/
    __le16 bg_free_blocks_count; /* Free blocks count */
    __le16 bg_free_inodes_count; /* Free inodes count */
    __le16 bg_used_dirs_count; /* Directories count */
    __le16 bg_pad;
    __le32 bg_reserved[3];
};
```
> [!TIP] 
> GDT 就是磁盘版的 VMA
> 如果你学过 Linux 的内存管理，不妨把 `ext2_group_desc` 类比成描述虚拟内存区域的 `struct vm_area_struct` (VMA)。
> VMA 告诉操作系统：“这段内存是代码段，那段内存是堆栈”；
> 而 GDT 则是告诉文件系统：“在这个 Block Group 里，哪几个块是 Block Bitmap，哪几个块是 Inode Table”。它们本质上都是
> 为了划分和管理一段连续的存储空间而存在的**元数据索引**。
### 4.3 块位图 （Block Bitmap）
Block Bitmap中记录着 Data Block 中哪个数据块已经被占⽤，哪个数据块没有被占⽤
> Block 号按照分区划分，不可跨分区

### 4.4 inode 位图（inode Bitmap）
每个比特位表⽰⼀个inode是否空闲可⽤。
### 4.5 inode Table (inode 表)
还记得我们刚才在那一大串代码里贴的 `struct ext2_inode` 吗？
**inode Table 本质上就是一个巨大的、由 `ext2_inode` 结构体组成的连续数组**

**为什么 Linux 找文件元数据那么快？**
因为只要文件系统知道了文件的 `inode 号`，结合当前 Block Group 的起始地址，直接用 $O(1)$ 的时间复杂度做一次 `数组下标寻址`，就能把这个文件的所有属性全部从磁盘加载到内存

> [!NOTE]
> inode 号并不是全局随机分配的，它是基于这个巨大的 Table 的 Index 算出来的。
> inode 号 并不存在于 inode 结构体内部，并且不可跨分区

### 4.6 Data Block (数据块区)
终于来到磁盘面积最大、也是我们存放学习资料（和Miku酱的美图）的真正位置了！

这里是被 inode 中的 `i_block[15]` 数组所指向的物理区域。但是，千万不要以为数据块里存的都是普通的文本或视频二进制！

在 Linux 的哲学里：“一切皆文件”。因此，**Data Block 具有“双重人格”**，这取决于指向它的那个 inode 到底是个什么玩意儿：

1. **如果 inode 是个普通文件 (`-`)：**
   那 Data Block 里存的就是老老实实的文件内容（比如你写的代码，或者一首 Miku 的 mp3）。

2. **如果 inode 是个目录 (`d`)：**                                       
   那 Data Block 里存的内容就非常有趣了！它存的是一个叫做 **`dentry` (Directory Entry, 目录项)** 的结构体数组。
   这个数组里记录着该目录下所有文件的**“文件名”**和**“对应的 inode 号”**。
```cpp
struct ext2_dir_entry_2 {
    __le32  inode;          /* 这个文件对应的 inode 编号 */
    __le16  rec_len;        /* 这个目录项的长度 */
    __u8    name_len;       /* 文件名的实际长度 */
    __u8    file_type;      /* 文件类型（普通文件、目录、软链接等） */
    char    name[];         /* 文件名字符串 (没有 \0 结尾) */
};
```
> [!NOTE]
> **磁盘上的目录项（如 `ext2_dir_entry_2`）：** 它是物理存在的，躺在目录的 Data Block 里，仅仅记录了 `<文件名, inode号>` 的映射。它是持久化的。

