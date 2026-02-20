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
   - S = ((LBA % n) % m) + 1

> 操作系统只关心 LBA 地址，转换的工作就交给硬件啦
## 2. 文件系统前置
### 2.1 块
文件系统是操作系统关于对磁盘/硬盘管理工作的部分
硬盘与磁盘是典型的 块 设备，操作系统不用扇区作为基本单位，这样效率也太低，操作系统通常一个一个块地读取，一个块的扇区数量是由 格式化 决定的，我们之后会谈到这个，最常见 8 个扇区组成一块，正好 8 * 512 = 4 KB
> 每个扇区拥有 LBA ，所以自然能简单地得到块的逻辑地址
- 块号 = LBA // 8
- 块内偏移 = LBA % 8
- LBA = 块号 * 8 + 块内偏移

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
> 最后还填充了190个32位无符号整数以凑满1KB，类似的操作很多

超级块存放着整个分区的文件系统信息
- s_inodes_count 此分区的所有块组的 inode 总量 
- s_blocks_count 此分区所有块组中的块总量
- s_free_blocks_count 此分区空闲的块数量
- s_free_inodes_count 此分区空闲 inode 的数量
- s_blocks_per_group 此分区每个块组的块总量
- s_inodes_per_group 此分区每个块组的 inode 总量
- s_inode_size 此分区 inode 结构体的大小
- s_first_data_block 超级块所在块的块号
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

> [!TIP]
> **Bitmap 是如何映射的？**
> 我们知道全局有一个庞大的 inode 编号和 Block 编号，但 Bitmap 是按照`块组`分别存放的。这就涉及到一个**全局编号到局部比特位**的映射转换。
> 
> 还记得前面提过的 超级块 吗？里面有两个关键字段：
> * `s_inodes_per_group`：每个块组包含的 inode 数量。
> * `s_blocks_per_group`：每个块组包含的 Block 数量。
> 
> **以 inode 为例（1-base）：**
> 假设我们要找 inode 编号为 `I` 的状态：
> 1. **它在哪一个块组？** >    `Group_Index = (I - 1) / s_inodes_per_group`
> 2. **它是该块组 inode Bitmap 里的第几个比特？**
>    `Bit_Index = (I - 1) % s_inodes_per_group`
> 
> **以 Data Block 为例（通常 Block 是 0-base，但要减去全局的起始数据块偏移）：**
> 假设要找 Block 编号为 `B` 的状态：
> 1. **它在哪一个块组？**
>    `Group_Index = (B - s_first_data_block) / s_blocks_per_group`
> 2. **它是该块组 Block Bitmap 里的第几个比特？**
>    `Bit_Index = (B - s_first_data_block) % s_blocks_per_group`
### i_block[EXT2_N_BLOCKS] - 索引映射
这个 inode 结构体成员用来帮助一个文件找到它的数据都分别在哪些块里被存储，可以跨越块组存储，虽然操作系统会尽量把同一个文件的数据和属性存在一个块组里
数组大小这个宏一般是15，其中包含：
- 12个直接块指针
- 1个一级块索引表指针
- 1个二级块索引表指针
- 1个三级块索引表指针
- 索引表本身存在于数据区中
每个指针占 4 字节（__le32）
一级索引表能存多少个块号？ 假设块大小 4KB，则一个索引块能存 4096 / 4 = 1024 个块号
最大文件大小计算：
直接块：12 * 4KB = 48KB
一级索引：1024 * 4KB = 4MB
二级索引：1024 * 1024 * 4KB = 4GB
三级索引：1024³ * 4KB = 4TB

### 格式化
> [!TIP]
> **知道 inode 号的情况下，对文件进行增、删、查、改到底是在做什么？**
> - **格式化的本质：** 分区之后的格式化操作，就是对物理分区进行“逻辑分组”，并在每个分组中写入 Superblock 、GDT、Block Bitmap、Inode Bitmap 等管理信息。这些管理信息的集合，就叫作**文件系统**。
> - **O(1) 的寻址：** 只要知道了文件的 inode 号，内核就能在指定分区中直接计算出它属于**哪一个分组**，进而算出它在该分组 inode Table 中的**确切下标位置**。
> - 只要拿到了 inode 结构体，文件的所有属性和数据块指针就全部都有了！

## 5. 文件路径
### 5.1 路径解析
我们平时使用系统调用或者命令行工具操作目标文件时，都要使用绝对路径或者相对路径
这是因为，要访问目标文件，必须要获得那个 inode 结构体，这样就能获取文件的属性以及数据，而用户层面的目标文件是文件名而不是 inode，之前我们提到过，ext2_dir_entry_2 结构体数组中存放着文件名与 对应 inode 编号的映射关系，这个数组则存在于直接父级目录的数据块中，从根节点开始进行的路径解析是操作目标文件的必需步骤，需要沿途检查目录权限，路径正确性等等
### 5.2 路径缓存 - 内存中的 dentry
每次对文件操作都需要路径解析，如果不断地对磁盘进行缓慢的IO操作，读取目录文件的数据块，这非常笨重。
操作系统会在内存中动态维护一颗路径缓存树，这样每个路径都只需要从磁盘里读一次了，树的节点结构体如下：
```cpp
#include <linux/dcache.h>

struct dentry {
    struct inode *d_inode;      /* 指向对应的 inode 结构体 */
    struct dentry *d_parent;    /* 指向父目录的 dentry (parent directory) */
    struct qstr d_name;         /* 目录项的名字，例如 "edu.txt" */
    struct list_head d_subdirs; /* 子目录项的链表头 (our children) */
    ......
};
现在，操作系统开始从根目录进行路径解析的时候，优先使用本目录的dentry，内存读写速度可比磁盘快多了
路径尽头的文件就是路径缓存树的叶子节点，也存在对应的dentry
```
### 5.3 挂载分区 
目录文件存储着 inode 编号与文件名的映射，但 inode 编号不能跨分区，如何确定文件属于哪个分区？这就涉及到挂载机制。

Linux 没有 Windows 的盘符概念。分区是对硬件的抽象，优秀的操作系统应该让这层抽象具有普适性，而不是再抽象回硬件形态（如 C:、D:）造成冗余。Linux 通过挂载点动态关联分区，对用户透明。

挂载机制和路径解析关联很大，当一个分区中建立文件系统，也就是格式化之后，他还无法直接使用，必须得和指定的目录进行关联，让操作系统知道，哦哦，原来这个目录为根的子树往后是个新分区了呀
操作系统一开始只有与根目录相关联的一个分区，挂载机制就相当于把其他的分区挂在了其他已经存在的目录树上，接下来我们来谈谈具体的过程

挂载操作可以用 mount 指令，当使用挂载指令把一块分区与一个目录进行关联，那么当然要对这个目录做路径解析，验证这个目录是否有足够的权限，是否存在，不合法直接报错就不会挂载了
若挂载成功，这时缓存目录树中一定已经有了那个被挂载目录的 dentry，我们叫它old_dentry
内核会在 old_dentry 中的 d_falgs 标志位设置已挂载，这个标志位一旦被设置成这样就会告知操作系统接下来的路径解析不要走原先的 dentry 树路径了，我们来走跨纬度传送门吧，lookup_mnt() 这个函数就是传送门，他会让操作系统去查全局挂载表
```cpp
struct vfsmount {
    struct dentry *mnt_root;        /* 新盘的根 Dentry */
    struct super_block *mnt_sb;     /* 新盘的超级块 (Superblock) */
    int mnt_flags;                  /* 挂载权限标志 (如 MS_RDONLY 只读) */
}; //这个结构体可以被看作一个分区实例
```
全局挂载表是个哈希，它的 key 是一个 <旧分区的 vfsmount, old_dentry>，value 是新分区的 vfsmount
这样，操作系统就不会管 old_dentry了，直接用上新分区里的 mnt_root 就行，mnt_root 是被挂载的分区的根目录 dentry

这种遮蔽行为不需要删除 old_dentry 及其子树结构，很优雅
卸载分区同样只需要把那个标志位重置就行了

### 5.3 总结
**一次跨分区的路径解析全流程：**

1. 读取挂载在 /mnt/usb 下的 a.txt：
2. 系统从根目录 / 开始查，找到 mnt 的 dentry。
3. 继续往下，找到 usb 的 dentry（即 old_dentry）。
4. 发现 usb 的 dentry 被打上了已挂载的标签。
5. 触发传送门，用当前的上下文去查挂载哈希表。
6. 查出 U 盘文件系统的 vfsmount 实例。
7. 提取出 U 盘实例里的 mnt_root（U 盘的根 dentry）。
8. 从 U 盘的根 dentry 出发，在 U 盘的目录树中找到了 a.txt 的 dentry。
9. 拿到对应的 inode，顺利读取数据。

## 6. 软硬链接
### 6.1 软链接
当你创建一个软链接时，文件系统会为它分配一个全新的 inode 和一个全新的 dentry，它是一个彻头彻尾的新文件。
- 软链接文件的 data block 里存的是目标文件的绝对/相对路径字符串
- inode 结构体中的标志位被设置为`S_IFLNK`以标识此文件是一个软链接
- 路径解析如果碰到软链接，会接着其存储的路径继续往下解析，继续去寻找真正的目标文件
- 支持跨分区，反正他就是存着路径字符串而已
- 允许软链接连接目录
- 会有失效，悬空引用的问题 - `Dangling Link`
> 创建命令 - ln -s  /path/to/target_file   /path/to/soft_link_name
### 6.2 硬链接
硬链接并没有创建新的文件实体
当创建一个硬链接时，内核做的动作就只是创建一个 新的 dentry，并把其中的 inode 指针替换为目标文件 inode 的指针
**也就是说，多个 dentry 指向同一个 inode结构体**
- 引用计数保活 有点像 shared智能指针，inode 结构体 中有一个`i_nlink`成员，他记录目前有几个 dentry 指向此 inode，只有这个成员变为 0，且没有一个进程打开这个 inode 对应的文件，这个文件才会真正被回收
- 不可跨分区，inode 不可跨分区
- 严禁用户自己硬链接目录，那样你将永远无法到达目标文件的真实...
- 虽然  . 和 .. 就是操作系统自己设立的目录硬链接，这就是为什么一个空目录的 `i_nlink` 是 2
> 创建命令 - ln /path/to/target_file   /path/to/hard_link_name
> 不加参数默认就是硬链接