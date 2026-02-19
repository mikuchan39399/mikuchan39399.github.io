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

