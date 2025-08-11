```bash
sudo mount -o loop ext4_htree.img /mnt/ext4_htree
losetup -a
./ext_tool -m -d /dev/loop19
```


## def.h
```c
#ifndef _EXT4_TOOL_LIB_H_
#define _EXT4_TOOL_LIB_H_

#ifndef _GNU_SOURCE
#define _GNU_SOURCE /* for O_DIRECT */
#endif

/* 对超过2GB的大文件进行支持 */
#define __USE_FILE_OFFSET64
#define __USE_LARGEFILE64
#define _LARGEFILE64_SOURCE

#if defined(__APPLE__) || defined(__BSD__)
#include <stdlib.h>
#define memalign(alignment, size) malloc(size)
#elif defined(_WIN32)
#include <malloc.h>
#define memalign(alignment, size) _aligned_malloc(size, alignment)
#else
#include <stdlib.h>
#endif

#include <inttypes.h>
#include <dirent.h>
#include <fcntl.h>
#include <getopt.h>
#include <linux/ioctl.h>
#include <linux/sched.h>
#include <linux/types.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <string.h>
#include <sys/ioctl.h>
#include <sys/mount.h> // mount 命令
#include <sys/reboot.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <time.h>
#include <unistd.h>
#include <malloc.h> // 某些平台需要额外包含
#include <ctype.h>  // 必须包含的头文件

#define MEM_ALIGN_SIZE 512

#define SAFE_CLOSE(fd) \
    do                 \
    {                  \
        if (-1 != fd)  \
        {              \
            close(fd); \
            fd = -1;   \
        }              \
    } while (0) ///< 安全关闭文件描述符

#define TEST_BIT(val, bit) (((val & bit) == bit) && bit != 0)
#define TEST_BIT_Z(val, bit) (((val & bit) == bit))
#define SET_BIT(val, bit) (val |= bit)
#define SET_NBIT(val, bit) (val &= ~bit)

#ifdef offsetof
#undef offsetof
#define offsetof(TYPE, MEMBER) ((size_t)&((TYPE *)0)->MEMBER)
#endif

/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:	the pointer to the member.
 * @type:	the type of the container struct this is embedded in.
 * @member:	the name of the member within the struct.
 *
 */
#define container_of(ptr, type, member)                    \
    ({                                                     \
        const typeof(((type *)0)->member) *__mptr = (ptr); \
        (type *)((char *)__mptr - offsetof(type, member)); \
    })

enum
{
    INFO_LEVEL_1 = 1, /* 重要信息 */
    INFO_LEVEL_2,
    INFO_LEVEL_3, /* 次要信息 */
    INFO_LEVEL_4,
    INFO_LEVEL_5, /* 非常次要信息 */
};

#define EXT4_LVL_SUPER 0x1
#define EXT4_LVL_GDT 0x2
#define EXT4_LVL_BLOCK_MAP 0x4
#define EXT4_LVL_INODE 0x8
#define EXT4_LVL_JNL 0x10
#define EXT4_LVL_JNL_DATA 0x20
#define EXT4_LVL_DATA 0x40

extern int outlvl;
#define pimp(fmt, arg...) printf("[major]:" fmt, ##arg);
#define pinfo(flag, fmt, arg...)                \
    do                                          \
    {                                           \
        if (TEST_BIT(outlvl, flag))             \
            printf("[info]: %-35s" fmt, ##arg); \
    } while (0);
#define pinfo_n(flag, fmt, arg...)  \
    do                              \
    {                               \
        if (TEST_BIT(outlvl, flag)) \
            printf(fmt, ##arg);     \
    } while (0);
#define pwarn(fmt, arg...) printf("[warning]:" fmt "\n", ##arg);
#define perr(fmt, arg...) printf("[error]:%s :" fmt "\n", __func__, ##arg);

/* inode state */
enum
{
    EXT4_INODE_USE = 0,
    EXT4_INODE_UNUSE,
    EXT4_INODE_NOT_INIT,
};

/* OS. One of:
0	Linux
1	Hurd
2	Masix
3	FreeBSD
4	Lites
*/
enum
{
    EXT4_OS_LINUX = 0,
    EXT4_OS_Hurd,
    EXT4_OS_Masix,
    EXT4_OS_FreeBSD,
    EXT4_OS_Lites,
};

/*
Behaviour when detecting errors. One of:
1	Continue
2	Remount read-only
3	Panic
*/
enum
{
    EXT4_SERR_Continue = 1,
    EXT4_SERR_Remount,
    EXT4_SERR_Panic,
};

/*
File system state. Valid values are:
0x0001	Cleanly umounted
0x0002	Errors detected
0x0004	Orphans being recovered
*/
enum
{
    EXT4_STATE_Cleanly = 1,
    EXT4_STATE_Errors = 2,
    EXT4_STATE_Orphans = 4,
};

/*Magic signature, 0xEF53*/
#define EXT4_Magic 0xEF53

/*
Revision level. One of:
0	Original format
1	v2 format w/ dynamic inode sizes
*/
enum
{
    EXT4_REV_LVL_Original = 0,
    EXT4_REV_LVL_v2,
};

/*
Compatible feature set flags. Kernel can still read/write this fs even if it doesn't understand a flag; e2fsck will
not attempt to fix a filesystem with any unknown COMPAT flags. Any of:

0x1	Directory preallocation (COMPAT_DIR_PREALLOC).

0x2	"imagic inodes". Used by AFS to indicate inodes that are not linked into the directory namespace. Inodes marked with
this flag will not be added to lost+found by e2fsck. (COMPAT_IMAGIC_INODES).

0x4	Has a journal (COMPAT_HAS_JOURNAL).

0x8	Supports extended attributes (COMPAT_EXT_ATTR).

0x10 Has reserved GDT blocks for filesystem expansion. Requires RO_COMPAT_SPARSE_SUPER. (COMPAT_RESIZE_INODE).

0x20 Has indexed directories. (COMPAT_DIR_INDEX).

0x40 "Lazy BG". Not in Linux kernel, seems to have been for uninitialized block groups? (COMPAT_LAZY_BG).

0x80 "Exclude inode". Intended for filesystem snapshot feature, but not used. (COMPAT_EXCLUDE_INODE).

0x100 "Exclude bitmap". Seems to be used to indicate the presence of snapshot-related exclude bitmaps? Not defined in
kernel or used in e2fsprogs. (COMPAT_EXCLUDE_BITMAP).

0x200 Sparse Super Block, v2. If this flag is set, the SB field s_backup_bgs points to the two block groups that
contain backup superblocks. (COMPAT_SPARSE_SUPER2).
*/
#define EXT4_FEATURE_COMPAT_DIR_PREALLOC 0x0001
#define EXT4_FEATURE_COMPAT_IMAGIC_INODES 0x0002
#define EXT4_FEATURE_COMPAT_HAS_JOURNAL 0x0004
#define EXT4_FEATURE_COMPAT_EXT_ATTR 0x0008
#define EXT4_FEATURE_COMPAT_RESIZE_INODE 0x0010
#define EXT4_FEATURE_COMPAT_DIR_INDEX 0x0020
#define EXT4_FEATURE_COMPAT_LAZY_BG 0x0040
#define EXT4_FEATURE_COMPAT_EXCLUDE_INODE 0x0080
#define EXT4_FEATURE_COMPAT_EXCLUDE_BITMAP 0x0100
#define EXT4_FEATURE_COMPAT_SPARSE_SUPER2 0x0200

union feature_compat
{
    int feature_compat_dat;
    struct
    {
        int dir_prealloc : 1;
        int imagic_inodes : 1;
        int has_journal : 1;
        int ext_attr : 1;

        int resize_inode : 1;
        int dir_index : 1;
        int lazy_bg : 1;
        int exclude_inode : 1;

        int exclude_bitmap : 1;
        int sparse_super2 : 1;
    };
};

/*
Incompatible feature set. If the kernel or e2fsck doesn't understand one of these bits, it will refuse to mount or
attempt to repair the filesystem. Any of:

0x1	Compression. Not implemented. (INCOMPAT_COMPRESSION).

0x2	Directory entries record the file type. See ext4_dir_entry_2 below. (INCOMPAT_FILETYPE).

0x4	Filesystem needs journal recovery. (INCOMPAT_RECOVER).

0x8	Filesystem has a separate journal device. (INCOMPAT_JOURNAL_DEV).

0x10	Meta block groups. See the earlier discussion of this feature. (INCOMPAT_META_BG).

0x40	Files in this filesystem use extents. (INCOMPAT_EXTENTS).

0x80	Enable a filesystem size over 2^32 blocks. (INCOMPAT_64BIT).

0x100	Multiple mount protection. Prevent multiple hosts from mounting the filesystem concurrently by updating a
reserved block periodically while mounted and checking this at mount time to determine if the filesystem is in use on
another host. (INCOMPAT_MMP).

0x200	Flexible block groups. See the earlier discussion of this feature. (INCOMPAT_FLEX_BG).

0x400	Inodes can be used to store large extended attribute values (INCOMPAT_EA_INODE).

0x1000	Data in directory entry. Allow additional data fields to be stored in each dirent, after struct ext4_dirent. The
presence of extra data is indicated by flags in the high bits of ext4_dirent file type flags (above EXT4_FT_MAX). The
flag EXT4_DIRENT_LUFID = 0x10 is used to store a 128-bit File Identifier for Lustre. The flag EXT4_DIRENT_IO64 = 0x20 is
used to store the high word of 64-bit inode numbers. Feature still in development. (INCOMPAT_DIRDATA).

0x2000	Metadata checksum seed is stored in the superblock. This feature enables the administrator to change the UUID of
a metadata_csum filesystem while the filesystem is mounted; without it, the checksum definition requires all metadata
blocks to be rewritten. (INCOMPAT_CSUM_SEED).

0x4000	Large directory >2GB or 3-level htree. Prior to this feature, directories could not be larger than 4GiB and
could not have an htree more than 2 levels deep. If this feature is enabled, directories can be larger than 4GiB and
have a maximum htree depth of 3. (INCOMPAT_LARGEDIR).

0x8000	Data in inode. Small files or directories are stored directly in the inode i_blocks and/or xattr space.
(INCOMPAT_INLINE_DATA).

0x10000	Encrypted inodes are present on the filesystem (INCOMPAT_ENCRYPT).
*/
#define EXT4_FEATURE_INCOMPAT_COMPRESSION 0x0001
#define EXT4_FEATURE_INCOMPAT_FILETYPE 0x0002
#define EXT4_FEATURE_INCOMPAT_RECOVER 0x0004     /* Needs recovery */
#define EXT4_FEATURE_INCOMPAT_JOURNAL_DEV 0x0008 /* Journal device */
#define EXT4_FEATURE_INCOMPAT_META_BG 0x0010
#define EXT4_FEATURE_INCOMPAT_EXTENTS 0x0040 /* extents support */
#define EXT4_FEATURE_INCOMPAT_64BIT 0x0080
#define EXT4_FEATURE_INCOMPAT_MMP 0x0100
#define EXT4_FEATURE_INCOMPAT_FLEX_BG 0x0200
#define EXT4_FEATURE_INCOMPAT_EA_INODE 0x0400 /* EA in inode */
#define EXT4_FEATURE_INCOMPAT_DIRDATA 0x1000  /* data in dirent */
#define EXT4_FEATURE_INCOMPAT_CSUM_SEED 0x2000
#define EXT4_FEATURE_INCOMPAT_LARGEDIR 0x4000    /* >2GB or 3-lvl htree */
#define EXT4_FEATURE_INCOMPAT_INLINE_DATA 0x8000 /* data in inode */
#define EXT4_FEATURE_INCOMPAT_ENCRYPT 0x10000

union feature_incompat
{
    int feature_incompat_dat;
    struct
    {
        int compression : 1;
        int file_type : 1;
        int journal_recovery : 1;
        int journal_device : 1;

        int meta_bg : 1;
        int nc1 : 1;
        int use_extents : 1;
        int use_64bit : 1;

        int mul_mount_protection : 1;
        int flex_bg : 1;
        int ea_inode : 1;
        int nc2 : 1;

        int dirdata : 1;
        int csum_seed : 1;
        int large_dir : 1;
        int inline_data : 1;

        int encrypt : 1;
    };
};

/*
Readonly-compatible feature set. If the kernel doesn't understand one of these bits, it can still mount read-only, but
e2fsck will refuse to modify the filesystem. Any of:

0x1	Sparse superblocks. See the earlier discussion of this feature. (RO_COMPAT_SPARSE_SUPER).

0x2	Allow storing files larger than 2GiB (RO_COMPAT_LARGE_FILE).

0x4	Was intended for use with htree directories, but was not needed. Not used in kernel or e2fsprogs
(RO_COMPAT_BTREE_DIR).

0x8	This filesystem has files whose space usage is stored in i_blocks in units of filesystem blocks, not 512-byte
sectors. Inodes using this feature will be marked with EXT4_INODE_HUGE_FILE. (RO_COMPAT_HUGE_FILE)

0x10	Group descriptors have checksums. In addition to detecting corruption, this is useful for lazy formatting with
uninitialized groups (RO_COMPAT_GDT_CSUM).

0x20	Indicates that the old ext3 32,000 subdirectory limit no longer applies. A directory's i_links_count will be set
to 1 if it is incremented past 64,999. (RO_COMPAT_DIR_NLINK).

0x40	Indicates that large inodes exist on this filesystem, storing extra fields after EXT2_GOOD_OLD_INODE_SIZE.
(RO_COMPAT_EXTRA_ISIZE).

0x80	This filesystem has a snapshot. Not implemented in ext4. (RO_COMPAT_HAS_SNAPSHOT).

0x100	Quota is handled transactionally with the journal (RO_COMPAT_QUOTA).

0x200	This filesystem supports "bigalloc", which means that filesystem block allocation bitmaps are tracked in units
of clusters (of blocks) instead of blocks (RO_COMPAT_BIGALLOC).

0x400	This filesystem supports metadata checksumming. (RO_COMPAT_METADATA_CSUM; implies RO_COMPAT_GDT_CSUM, though
GDT_CSUM must not be set)

0x800	Filesystem supports replicas. This feature is neither in the kernel nor e2fsprogs. (RO_COMPAT_REPLICA).

0x1000	Read-only filesystem image; the kernel will not mount this image read-write and most tools will refuse to write
to the image. (RO_COMPAT_READONLY).

0x2000	Filesystem tracks project quotas. (RO_COMPAT_PROJECT)
*/
#define EXT4_FEATURE_RO_COMPAT_SPARSE_SUPER 0x0001
#define EXT4_FEATURE_RO_COMPAT_LARGE_FILE 0x0002
#define EXT4_FEATURE_RO_COMPAT_BTREE_DIR 0x0004
#define EXT4_FEATURE_RO_COMPAT_HUGE_FILE 0x0008
#define EXT4_FEATURE_RO_COMPAT_GDT_CSUM 0x0010
#define EXT4_FEATURE_RO_COMPAT_DIR_NLINK 0x0020
#define EXT4_FEATURE_RO_COMPAT_EXTRA_ISIZE 0x0040
#define EXT4_FEATURE_RO_COMPAT_QUOTA 0x0100
#define EXT4_FEATURE_RO_COMPAT_BIGALLOC 0x0200
/*
 * METADATA_CSUM also enables group descriptor checksums (GDT_CSUM).  When
 * METADATA_CSUM is set, group descriptor checksums use the same algorithm as
 * all other data structures' checksums.  However, the METADATA_CSUM and
 * GDT_CSUM bits are mutually exclusive.
 */
#define EXT4_FEATURE_RO_COMPAT_METADATA_CSUM 0x0400
#define EXT4_FEATURE_RO_COMPAT_READONLY 0x1000
#define EXT4_FEATURE_RO_COMPAT_PROJECT 0x2000

union feature_rocompat
{
    int feature_rocompat_dat;
    struct
    {
        int sparse_super : 1;
        int large_file : 1;
        int btree_dir : 1;
        int huge_dir : 1;

        int gdt_csum : 1;
        int dir_nlink : 1;
        int extra_isize : 1;
        int nc1 : 1;

        int quota : 1;
        int bigalloc : 1;
        int metadata_csum : 1;
        int nc2 : 1;

        int readonly : 1;
        int project : 1;
    };
};

/*
Default mount options. Any of:
0x0001	Print debugging info upon (re)mount. (EXT4_DEFM_DEBUG)

0x0002	New files take the gid of the containing directory (instead of the fsgid of the current process).
(EXT4_DEFM_BSDGROUPS)

0x0004	Support userspace-provided extended attributes. (EXT4_DEFM_XATTR_USER)

0x0008	Support POSIX access control lists (ACLs). (EXT4_DEFM_ACL)

0x0010	Do not support 32-bit UIDs. (EXT4_DEFM_UID16)

0x0020	All data and metadata are commited to the journal. (EXT4_DEFM_JMODE_DATA)

0x0040	All data are flushed to the disk before metadata are committed to the journal. (EXT4_DEFM_JMODE_ORDERED)

0x0060	Data ordering is not preserved; data may be written after the metadata has been written. (EXT4_DEFM_JMODE_WBACK)

0x0100	Disable write flushes. (EXT4_DEFM_NOBARRIER)

0x0200	Track which blocks in a filesystem are metadata and therefore should not be used as data blocks. This option
will be enabled by default on 3.18, hopefully. (EXT4_DEFM_BLOCK_VALIDITY)

0x0400	Enable DISCARD support, where the storage device is told about blocks becoming unused. (EXT4_DEFM_DISCARD)

0x0800	Disable delayed allocation. (EXT4_DEFM_NODELALLOC)
*/
/*
 * Default mount options
 */
#define EXT4_DEFM_DEBUG 0x0001
#define EXT4_DEFM_BSDGROUPS 0x0002
#define EXT4_DEFM_XATTR_USER 0x0004
#define EXT4_DEFM_ACL 0x0008
#define EXT4_DEFM_UID16 0x0010
#define EXT4_DEFM_JMODE 0x0060
#define EXT4_DEFM_JMODE_DATA 0x0020
#define EXT4_DEFM_JMODE_ORDERED 0x0040
#define EXT4_DEFM_JMODE_WBACK 0x0060
#define EXT4_DEFM_NOBARRIER 0x0100
#define EXT4_DEFM_BLOCK_VALIDITY 0x0200
#define EXT4_DEFM_DISCARD 0x0400
#define EXT4_DEFM_NODELALLOC 0x0800

/*********************************************superblock相关**********************************************/

/*
 * Revision levels
 */
#define EXT4_GOOD_OLD_REV 0 /* The good old (original) format */
#define EXT4_DYNAMIC_REV 1  /* V2 format w/ dynamic inode sizes */

#define EXT4_CURRENT_REV EXT4_GOOD_OLD_REV
#define EXT4_MAX_SUPP_REV EXT4_DYNAMIC_REV

/*
Miscellaneous flags. Any of:
0x0001	Signed directory hash in use.
0x0002	Unsigned directory hash in use.
0x0004	To test development code.
*/
enum
{
    EXT4_MISC_F_Signed = 1,
    EXT4_MISC_F_Unsigned = 2,
    EXT4_MISC_F_test = 4,
};

struct ext4_super_block
{
    /*00*/ __le32 s_inodes_count;      /* Inodes count */
    __le32 s_blocks_count_lo;          /* Blocks count */
    __le32 s_r_blocks_count_lo;        /* Reserved blocks count */
    __le32 s_free_blocks_count_lo;     /* Free blocks count */
    /*10*/ __le32 s_free_inodes_count; /* Free inodes count */
    __le32 s_first_data_block;         /* First Data Block */
    __le32 s_log_block_size;           /* Block size ,Block size is 2 ^ (10 + s_log_block_size). */
    __le32 s_log_cluster_size;         /* Allocation cluster size */
    /*20*/ __le32 s_blocks_per_group;  /* # Blocks per group */
    __le32 s_clusters_per_group;       /* # Clusters per group */
    __le32 s_inodes_per_group;         /* # Inodes per group */
    __le32 s_mtime;                    /* Mount time */
    /*30*/ __le32 s_wtime;             /* Write time */
    __le16 s_mnt_count;                /* Mount count */
    __le16 s_max_mnt_count;            /* Maximal mount count */
    __le16 s_magic;                    /* Magic signature */
    __le16 s_state;                    /* File system state */
    __le16 s_errors;                   /* Behaviour when detecting errors */
    __le16 s_minor_rev_level;          /* minor revision level */
    /*40*/ __le32 s_lastcheck;         /* time of last check */
    __le32 s_checkinterval;            /* max. time between checks */
    __le32 s_creator_os;               /* OS */
    __le32 s_rev_level;                /* Revision level */
    /*50*/ __le16 s_def_resuid;        /* Default uid for reserved blocks */
    __le16 s_def_resgid;               /* Default gid for reserved blocks */
    /*
     * These fields are for EXT4_DYNAMIC_REV superblocks only.
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
    __le32 s_first_ino;                     /* First non-reserved inode */
    __le16 s_inode_size;                    /* size of inode structure */
    __le16 s_block_group_nr;                /* block group # of this superblock */
    __le32 s_feature_compat;                /* compatible feature set */
    /*60*/ __le32 s_feature_incompat;       /* incompatible feature set */
    __le32 s_feature_ro_compat;             /* readonly-compatible feature set */
    /*68*/ __u8 s_uuid[16];                 /* 128-bit uuid for volume */
    /*78*/ char s_volume_name[16];          /* volume name */
    /*88*/ char s_last_mounted[64];         /* directory where last mounted */
    /*C8*/ __le32 s_algorithm_usage_bitmap; /* For compression */

    /*
     * Performance hints.  Directory preallocation should only
     * happen if the EXT4_FEATURE_COMPAT_DIR_PREALLOC flag is on.
     */
    __u8 s_prealloc_blocks;       /* Nr of blocks to try to preallocate*/
    __u8 s_prealloc_dir_blocks;   /* Nr to preallocate for dirs */
    __le16 s_reserved_gdt_blocks; /* Per group desc for online growth */

    /*
     * Journaling support valid if EXT4_FEATURE_COMPAT_HAS_JOURNAL set.
     */
    /*D0*/ __u8 s_journal_uuid[16]; /* uuid of journal superblock */
    /*E0*/ __le32 s_journal_inum;   /* inode number of journal file */
    __le32 s_journal_dev;           /* device number of journal file */
    __le32 s_last_orphan;           /* start of list of inodes to delete */
    __le32 s_hash_seed[4];          /* HTREE hash seed */
    __u8 s_def_hash_version;        /* Default hash version to use */
    __u8 s_jnl_backup_type;
    __le16 s_desc_size; /* size of group descriptor */
    /*100*/ __le32 s_default_mount_opts;
    __le32 s_first_meta_bg;  /* First metablock block group */
    __le32 s_mkfs_time;      /* When the filesystem was created */
    __le32 s_jnl_blocks[17]; /* Backup of the journal inode */

    /* 64bit support valid if EXT4_FEATURE_COMPAT_64BIT */
    /*150*/ __le32 s_blocks_count_hi; /* Blocks count */
    __le32 s_r_blocks_count_hi;       /* Reserved blocks count */
    __le32 s_free_blocks_count_hi;    /* Free blocks count */
    __le16 s_min_extra_isize;         /* All inodes have at least # bytes */
    __le16 s_want_extra_isize;        /* New inodes should reserve # bytes */
    __le32 s_flags;                   /* Miscellaneous flags */
    __le16 s_raid_stride;             /* RAID stride */
    __le16 s_mmp_update_interval;     /* # seconds to wait in MMP checking */
    __le64 s_mmp_block;               /* Block for multi-mount protection */
    __le32 s_raid_stripe_width;       /* blocks on all data disks (N*stride)*/
    __u8 s_log_groups_per_flex;       /* FLEX_BG group size */
    __u8 s_checksum_type;             /* metadata checksum algorithm used */
    __u8 s_encryption_level;          /* versioning level for encryption */
    __u8 s_reserved_pad;              /* Padding to next 32bits */
    __le64 s_kbytes_written;          /* nr of lifetime kilobytes written */
    __le32 s_snapshot_inum;           /* Inode number of active snapshot */
    __le32 s_snapshot_id;             /* sequential ID of active snapshot */
    __le64 s_snapshot_r_blocks_count; /* reserved blocks for active
                         snapshot's future use */
    __le32 s_snapshot_list;           /* inode number of the head of the
                                       on-disk snapshot list */
#define EXT4_S_ERR_START offsetof(struct ext4_super_block, s_error_count)
    __le32 s_error_count;        /* number of fs errors */
    __le32 s_first_error_time;   /* first time an error happened */
    __le32 s_first_error_ino;    /* inode involved in first error */
    __le64 s_first_error_block;  /* block involved of first error */
    __u8 s_first_error_func[32]; /* function where the error happened */
    __le32 s_first_error_line;   /* line number where error happened */
    __le32 s_last_error_time;    /* most recent time of an error */
    __le32 s_last_error_ino;     /* inode involved in last error */
    __le32 s_last_error_line;    /* line number where error happened */
    __le64 s_last_error_block;   /* block involved of last error */
    __u8 s_last_error_func[32];  /* function where the error happened */
#define EXT4_S_ERR_END offsetof(struct ext4_super_block, s_mount_opts)
    __u8 s_mount_opts[64];
    __le32 s_usr_quota_inum;    /* inode for tracking user quota */
    __le32 s_grp_quota_inum;    /* inode for tracking group quota */
    __le32 s_overhead_clusters; /* overhead blocks/clusters in fs */
    __le32 s_backup_bgs[2];     /* groups with sparse_super2 SBs */
    __u8 s_encrypt_algos[4];    /* Encryption algorithms in use  */
    __u8 s_encrypt_pw_salt[16]; /* Salt used for string2key algorithm */
    __le32 s_lpf_ino;           /* Location of the lost+found inode */
    __le32 s_prj_quota_inum;    /* inode for tracking project quota */
    __le32 s_checksum_seed;     /* crc32c(uuid) if csum_seed set */
    __le32 s_reserved[98];      /* Padding to the end of the block */
    __le32 s_checksum;          /* crc32c(superblock) */
};
#define EXT4_S_ERR_LEN (EXT4_S_ERR_END - EXT4_S_ERR_START)

#define EXT4_SB_OFFSET 1024
#define EXT4_SB_LEN 1024

#define READ_MAX_BUF_LEN 40960

/*********************************************GDT相关**********************************************/

#define EXT4_BG_INODE_UNINIT 0x0001 /* Inode table/bitmap not in use */
#define EXT4_BG_BLOCK_UNINIT 0x0002 /* Block bitmap not in use */
#define EXT4_BG_INODE_ZEROED 0x0004 /* On-disk itable initialized to zero */
union bg_flags_u
{
    uint16_t bg_flags_dat;
    struct
    {
        uint16_t inode_uninit : 1;
        uint16_t block_uninit : 1;
        uint16_t inode_zeroed : 1;
    };
};

/*
 * Structure of a blocks group descriptor Total size is 64 bytes.
 */
struct ext4_group_desc
{
    __le32 bg_block_bitmap_lo;      /* Blocks bitmap block */
    __le32 bg_inode_bitmap_lo;      /* Inodes bitmap block */
    __le32 bg_inode_table_lo;       /* Inodes table block */
    __le16 bg_free_blocks_count_lo; /* Free blocks count */
    __le16 bg_free_inodes_count_lo; /* Free inodes count */
    __le16 bg_used_dirs_count_lo;   /* Directories count */
    __le16 bg_flags;                /* EXT4_BG_flags (INODE_UNINIT, etc) */
    __le32 bg_exclude_bitmap_lo;    /* Exclude bitmap for snapshots */
    __le16 bg_block_bitmap_csum_lo; /* crc32c(s_uuid+grp_num+bbitmap) LE */
    __le16 bg_inode_bitmap_csum_lo; /* crc32c(s_uuid+grp_num+ibitmap) LE */
    __le16 bg_itable_unused_lo;     /* Unused inodes count */
    __le16 bg_checksum;             /* crc16(sb_uuid+group+desc) */

    /* These fields only exist if the 64bit feature is enabled and s_desc_size > 32. */
    /* 只有开启了64bit才会使用以下字节 */
    /*0x20*/ __le32 bg_block_bitmap_hi; /* Blocks bitmap block MSB */
    __le32 bg_inode_bitmap_hi;          /* Inodes bitmap block MSB */
    __le32 bg_inode_table_hi;           /* Inodes table block MSB */
    __le16 bg_free_blocks_count_hi;     /* Free blocks count MSB */
    __le16 bg_free_inodes_count_hi;     /* Free inodes count MSB */
    __le16 bg_used_dirs_count_hi;       /* Directories count MSB */
    __le16 bg_itable_unused_hi;         /* Unused inodes count MSB */
    __le32 bg_exclude_bitmap_hi;        /* Exclude bitmap block MSB */
    __le16 bg_block_bitmap_csum_hi;     /* crc32c(s_uuid+grp_num+bbitmap) BE */
    __le16 bg_inode_bitmap_csum_hi;     /* crc32c(s_uuid+grp_num+ibitmap) BE */
    __u32 bg_reserved;
};

#define EXT4_GROUP_DESC_LEN 32
#define EXT4_GROUP_DESC_64bit_LEN 64

#define EXT4_NDIR_BLOCKS 12
#define EXT4_IND_BLOCK EXT4_NDIR_BLOCKS
#define EXT4_DIND_BLOCK (EXT4_IND_BLOCK + 1)
#define EXT4_TIND_BLOCK (EXT4_DIND_BLOCK + 1)
#define EXT4_N_BLOCKS (EXT4_TIND_BLOCK + 1)

/*********************************************inode相关**********************************************/

/* File mode */
#define EXT4_S_IXOTH 0x1 //(Others may execute)
#define EXT4_S_IWOTH 0x2 //(Others may write)
#define EXT4_S_IROTH 0x4 // Others may read)

#define EXT4_S_IXGRP 0x8  //(Group members may execute)
#define EXT4_S_IWGRP 0x10 //(Group members may write)
#define EXT4_S_IRGRP 0x20 //(Group members may read)

#define EXT4_S_IXUSR 0x40  //(Owner may execute)
#define EXT4_S_IWUSR 0x80  //(Owner may write)
#define EXT4_S_IRUSR 0x100 //(Owner may read)

#define EXT4_S_ISVTX 0x200 //(Sticky bit)
#define EXT4_S_ISGID 0x400 //(Set GID)
#define EXT4_S_ISUID 0x800 //(Set UID)

/* These are mutually-exclusive file types: */
#define EXT4_S_IFIFO 0x1000  //(FIFO)
#define EXT4_S_IFCHR 0x2000  //(Character device)
#define EXT4_S_IFDIR 0x4000  //(Directory)
#define EXT4_S_IFBLK 0x6000  //(Block device)
#define EXT4_S_IFREG 0x8000  //(Regular file)
#define EXT4_S_IFLNK 0xA000  //(Symbolic link)
#define EXT4_S_IFSOCK 0xC000 //(Socket)

/*
 * Inode flags
 */
#define EXT4_SECRM_FL 0x00000001     /* Secure deletion(not implemented) */
#define EXT4_UNRM_FL 0x00000002      /* Undelete(not implemented) */
#define EXT4_COMPR_FL 0x00000004     /* Compress file(not really implemented) */
#define EXT4_SYNC_FL 0x00000008      /* Synchronous updates */
#define EXT4_IMMUTABLE_FL 0x00000010 /* Immutable file */
#define EXT4_APPEND_FL 0x00000020    /* writes to file may only append */
#define EXT4_NODUMP_FL 0x00000040    /* do not dump file */
#define EXT4_NOATIME_FL 0x00000080   /* do not update atime */
/* Reserved for compression usage... */
#define EXT4_DIRTY_FL 0x00000100    /* Dirty compressed file(not used) */
#define EXT4_COMPRBLK_FL 0x00000200 /* One or more compressed clusters(not used) */
#define EXT4_NOCOMPR_FL 0x00000400  /* Don't compress (not used)*/
/* nb: was previously EXT2_ECOMPR_FL */
#define EXT4_ENCRYPT_FL 0x00000800 /* encrypted file(never used) */
/* End compression flags --- maybe not all used */
#define EXT4_INDEX_FL 0x00001000        /* hash-indexed directory */
#define EXT4_IMAGIC_FL 0x00002000       /* AFS directory */
#define EXT4_JOURNAL_DATA_FL 0x00004000 /* file data should be journaled */
#define EXT4_NOTAIL_FL 0x00008000       /* file tail should not be merged */
#define EXT4_DIRSYNC_FL 0x00010000      /* dirsync behaviour (directories only) */
#define EXT4_TOPDIR_FL 0x00020000       /* Top of directory hierarchies*/
#define EXT4_HUGE_FILE_FL 0x00040000    /* Set to each huge file */
#define EXT4_EXTENTS_FL 0x00080000      /* Inode uses extents */
#define EXT4_EA_INODE_FL 0x00200000     /* Inode used for large EA */
#define EXT4_EOFBLOCKS_FL 0x00400000    /* Blocks allocated beyond EOF */
#define EXT4_INLINE_DATA_FL 0x10000000  /* Inode has inline data. */
#define EXT4_PROJINHERIT_FL 0x20000000  /* Create with parents projid */
#define EXT4_RESERVED_FL 0x80000000     /* reserved for ext4 lib */

union inode_flag
{
    int inode_flag_dat;
    struct
    {
        int secrm : 1;
        int unrm : 1;
        int compr : 1;
        int sync : 1;

        int immutable : 1;
        int append : 1;
        int nodump : 1;
        int noatime : 1;

        int dirty : 1;
        int comprblk : 1;
        int nocompr : 1;
        int encrypt : 1;

        int index : 1;
        int imagic : 1;
        int journal : 1;
        int notail : 1;

        int dirsync : 1;
        int topdir : 1;
        int hugefile : 1;
        int extents : 1;

        int nc1 : 1;
        int eainode : 1;
        int eofbolck : 1;
        int nc2 : 1;

        int nc3 : 1;
        int nc4 : 1;
        int nc5 : 1;
        int nc6 : 1;

        int inlinedata : 1;
        int projinherit : 1;
        int nc7 : 1;
        int reserved : 1;
    };
};

/*
File flags. Any of:
0x1	This file requires secure deletion (EXT4_SECRM_FL). (not implemented)
0x2	This file should be preserved, should undeletion be desired (EXT4_UNRM_FL). (not implemented)
0x4	File is compressed (EXT4_COMPR_FL). (not really implemented)
0x8	All writes to the file must be synchronous (EXT4_SYNC_FL).
0x10	File is immutable (EXT4_IMMUTABLE_FL).
0x20	File can only be appended (EXT4_APPEND_FL).
0x40	The dump(1) utility should not dump this file (EXT4_NODUMP_FL).
0x80	Do not update access time (EXT4_NOATIME_FL).
0x100	Dirty compressed file (EXT4_DIRTY_FL). (not used)
0x200	File has one or more compressed clusters (EXT4_COMPRBLK_FL). (not used)
0x400	Do not compress file (EXT4_NOCOMPR_FL). (not used)
0x800	Encrypted inode (EXT4_ENCRYPT_FL). This bit value previously was EXT4_ECOMPR_FL (compression error), which
was never used.
0x1000	Directory has hashed indexes (EXT4_INDEX_FL).
0x2000	AFS magic directory (EXT4_IMAGIC_FL).
0x4000	File data must always be written through the journal (EXT4_JOURNAL_DATA_FL).
0x8000	File tail should not be merged (EXT4_NOTAIL_FL). (not used by ext4)
0x10000	All directory entry data should be written synchronously (see dirsync) (EXT4_DIRSYNC_FL).
0x20000	Top of directory hierarchy (EXT4_TOPDIR_FL).
0x40000	This is a huge file (EXT4_HUGE_FILE_FL).
0x80000	Inode uses extents (EXT4_EXTENTS_FL).
0x200000	Inode stores a large extended attribute value in its data blocks (EXT4_EA_INODE_FL).
0x400000	This file has blocks allocated past EOF (EXT4_EOFBLOCKS_FL). (deprecated)
0x01000000	Inode is a snapshot (EXT4_SNAPFILE_FL). (not in mainline)
0x04000000	Snapshot is being deleted (EXT4_SNAPFILE_DELETED_FL). (not in mainline)
0x08000000	Snapshot shrink has completed (EXT4_SNAPFILE_SHRUNK_FL). (not in mainline)
0x10000000	Inode has inline data (EXT4_INLINE_DATA_FL).
0x20000000	Create children with the same project ID (EXT4_PROJINHERIT_FL).
0x80000000	Reserved for ext4 library (EXT4_RESERVED_FL).
Aggregate flags:
0x4BDFFF	User-visible flags.
0x4B80FF	User-modifiable flags. Note that while EXT4_JOURNAL_DATA_FL and EXT4_EXTENTS_FL can be set with setattr,
they are not in the kernel's EXT4_FL_USER_MODIFIABLE mask, since it needs to handle the setting of these flags in a
special manner and they are masked out of the set of flags that are saved directly to i_flags.

*/

/*
 * Structure of an inode on the disk
 */
struct ext4_inode
{
    /* 看上面的宏定义规定了文件的mode */
    __le16 i_mode;        /* File mode */
    __le16 i_uid;         /* Low 16 bits of Owner Uid */
    __le32 i_size_lo;     /* Size in bytes */
    __le32 i_atime;       /* Access time */
    __le32 i_ctime;       /* Inode Change time */
    __le32 i_mtime;       /* Modification time */
    __le32 i_dtime;       /* Deletion Time */
    __le16 i_gid;         /* Low 16 bits of Group Id */
    __le16 i_links_count; /* Links count */
    __le32 i_blocks_lo;   /* Blocks count */

    /* 看上面宏定义规定的flag状态 */
    __le32 i_flags; /* File flags */
    union
    {
        struct
        {
            __le32 l_i_version;
        } linux1;
        struct
        {
            __u32 h_i_translator;
        } hurd1;
        struct
        {
            __u32 m_i_reserved1;
        } masix1;
    } osd1;                        /* OS dependent 1 */
    __le32 i_block[EXT4_N_BLOCKS]; /* Pointers to blocks */
    __le32 i_generation;           /* File version (for NFS) */
    __le32 i_file_acl_lo;          /* File ACL */
    __le32 i_size_high;
    __le32 i_obso_faddr; /* Obsoleted fragment address */
    union
    {
        struct
        {
            __le16 l_i_blocks_high; /* were l_i_reserved1 */
            __le16 l_i_file_acl_high;
            __le16 l_i_uid_high;    /* these 2 fields */
            __le16 l_i_gid_high;    /* were reserved2[0] */
            __le16 l_i_checksum_lo; /* crc32c(uuid+inum+inode) LE */
            __le16 l_i_reserved;
        } linux2;
        struct
        {
            __le16 h_i_reserved1; /* Obsoleted fragment number/size which are removed in ext4 */
            __u16 h_i_mode_high;
            __u16 h_i_uid_high;
            __u16 h_i_gid_high;
            __u32 h_i_author;
        } hurd2;
        struct
        {
            __le16 h_i_reserved1; /* Obsoleted fragment number/size which are removed in ext4 */
            __le16 m_i_file_acl_high;
            __u32 m_i_reserved2[2];
        } masix2;
    } osd2; /* OS dependent 2 */
    __le16 i_extra_isize;
    __le16 i_checksum_hi;  /* crc32c(uuid+inum+inode) BE */
    __le32 i_ctime_extra;  /* extra Change time      (nsec << 2 | epoch) */
    __le32 i_mtime_extra;  /* extra Modification time(nsec << 2 | epoch) */
    __le32 i_atime_extra;  /* extra Access time      (nsec << 2 | epoch) */
    __le32 i_crtime;       /* File Creation time */
    __le32 i_crtime_extra; /* extra FileCreationtime (nsec << 2 | epoch) */
    __le32 i_version_hi;   /* high 32 bits for 64-bit version */
    __le32 i_projid;       /* Project ID */
};

#define INODE_DATA_LEN_128 128
#define INODE_DATA_LEN_256 256

typedef unsigned long long ext4_fsblk_t;
/*
 * This is the extent on-disk structure.
 * It's used at the bottom of the tree.
 */
struct ext4_extent
{
    __le32 ee_block;    /* first logical block extent covers */
    __le16 ee_len;      /* number of blocks covered by extent */
    __le16 ee_start_hi; /* high 16 bits of physical block */
    __le32 ee_start_lo; /* low 32 bits of physical block */
};

/*
 * This is index on-disk structure.
 * It's used at all the levels except the bottom.
 */
struct ext4_extent_idx
{
    __le32 ei_block;   /* index covers logical blocks from 'block' */
    __le32 ei_leaf_lo; /* pointer to the physical block of the next *
                        * level. leaf or next index could be there */
    __le16 ei_leaf_hi; /* high 16 bits of physical block */
    __u16 ei_unused;
};

/*
 * Each block (leaves and indexes), even inode-stored has header.
 */
struct ext4_extent_header
{
    __le16 eh_magic;      /* probably will support different formats */
    __le16 eh_entries;    /* number of valid entries */
    __le16 eh_max;        /* capacity of store in entries */
    __le16 eh_depth;      /* has tree real underlying blocks? */
    __le32 eh_generation; /* generation of the tree */
};

/*
 * Array of ext4_ext_path contains path to some extent.
 * Creation/lookup routines use it for traversal/splitting/etc.
 * Truncate uses it to simulate recursive walking.
 */
struct ext4_ext_path
{
    ext4_fsblk_t p_block;
    __u16 p_depth;
    __u16 p_maxdepth;
    struct ext4_extent *p_ext;
    struct ext4_extent_idx *p_idx;
    struct ext4_extent_header *p_hdr;
    // struct buffer_head		*p_bh;
};

#define EXTENT_LEN sizeof(struct ext4_extent)

#define EXT_FIRST_EXTENT(__hdr__) ((struct ext4_extent *)(((char *)(__hdr__)) + sizeof(struct ext4_extent_header)))
#define EXT_NEXT_EXTENT(__hdr__, entries) \
    ((struct ext4_extent *)(((char *)(__hdr__)) + ((entries + 1) * sizeof(struct ext4_extent_header))))
#define EXT_FIRST_INDEX(__hdr__) ((struct ext4_extent_idx *)(((char *)(__hdr__)) + sizeof(struct ext4_extent_header)))
#define EXT_HAS_FREE_INDEX(__path__) ((__path__)->p_hdr->eh_entries < (__path__)->p_hdr->eh_max)
#define EXT_LAST_EXTENT(__hdr__) (EXT_FIRST_EXTENT((__hdr__)) + ((__hdr__)->eh_entries) - 1)
#define EXT_LAST_INDEX(__hdr__) (EXT_FIRST_INDEX((__hdr__)) + ((__hdr__)->eh_entries) - 1)
#define EXT_MAX_EXTENT(__hdr__) (EXT_FIRST_EXTENT((__hdr__)) + ((__hdr__)->eh_max) - 1)
#define EXT_MAX_INDEX(__hdr__) (EXT_FIRST_INDEX((__hdr__)) + ((__hdr__)->eh_max) - 1)

#define EXT4_EXT_MAGIC (0xf30a)

/************inline data************/

#define EXT4_XATTR_MAGIC 0xEA020000
#define EXT4_XATTR_SYSTEM_DATA "data"

struct ext4_xattr_ibody_header
{
    __le32 h_magic; /* magic number for identification */
};

struct ext4_xattr_entry
{
    __u8 e_name_len;      /* length of name */
    __u8 e_name_index;    /* attribute name index */
    __le16 e_value_offs;  /* offset in disk block of value */
    __le32 e_value_block; /* disk block attribute is stored on (n/i) */
    __le32 e_value_size;  /* size of attribute value */
    __le32 e_hash;        /* hash value of name and value */
    char e_name[0];       /* attribute name */
};

#define XATTR_SYSTEM_DATA_LEN 4
#define XATTR_MAGIC_LEN 4
#define INLINE_DAT_ATTR (sizeof(struct ext4_xattr_entry) + XATTR_MAGIC_LEN) /* 实际长度应为16 */

/*********************************************目录相关**********************************************/
/*
 * Structure of a directory entry
 */
#define EXT4_NAME_LEN 255

struct ext4_dir_entry
{
    __le32 inode;             /* Inode number */
    __le16 rec_len;           /* Directory entry length */
    __le16 name_len;          /* Name length */
    char name[EXT4_NAME_LEN]; /* File name */
};

/*
 * The new version of the directory entry.  Since EXT4 structures are
 * stored in intel byte order, and the name_len field could never be
 * bigger than 255 chars, it's safe to reclaim the extra byte for the
 * file_type field.
 */
struct ext4_dir_entry_2
{
    __le32 inode;   /* Inode number */
    __le16 rec_len; /* Directory entry length */
    __u8 name_len;  /* Name length */
    __u8 file_type;
    char name[EXT4_NAME_LEN]; /* File name */
};

#define EXT4_FT_UNKNOWN 0
#define EXT4_FT_REG_FILE 1
#define EXT4_FT_DIR 2
#define EXT4_FT_CHRDEV 3
#define EXT4_FT_BLKDEV 4
#define EXT4_FT_FIFO 5
#define EXT4_FT_SOCK 6
#define EXT4_FT_SYMLINK 7

// 链表节点结构（存储目录条目关键信息）
typedef struct dir_entry_node
{
    uint32_t inode; // 主机字节序的inode号
    uint32_t root_dir_inode;
    uint8_t file_type;
    char *name; // 动态分配的文件名
    struct dir_entry_node *next;
} dir_entry_node_t;

/*********************************************jbd2日志相关**********************************************/
#define JBD2_MAGIC_NUMBER 0xc03b3998U

/*
 * Descriptor block types:
 */

#define JBD2_DESCRIPTOR_BLOCK 1
#define JBD2_COMMIT_BLOCK 2
#define JBD2_SUPERBLOCK_V1 3
#define JBD2_SUPERBLOCK_V2 4
#define JBD2_REVOKE_BLOCK 5

/* Definitions for the journal tag flags word: */
#define JBD2_FLAG_ESCAPE 1    /* on-disk block is escaped */
#define JBD2_FLAG_SAME_UUID 2 /* block has same uuid as previous */
#define JBD2_FLAG_DELETED 4   /* block deleted by this transaction */
#define JBD2_FLAG_LAST_TAG 8  /* last tag in this descriptor block */

/*
 * Standard header for all descriptor blocks:
 */
typedef struct journal_header_s
{
    __be32 h_magic;
    __be32 h_blocktype;
    __be32 h_sequence;
} journal_header_t;
/*
 * The journal superblock.  All fields are in big-endian byte order.
 */
typedef struct journal_superblock_s
{
    /* 0x0000 */
    journal_header_t s_header;

    /* 0x000C */
    /* Static information describing the journal */
    __be32 s_blocksize; /* journal device blocksize */
    __be32 s_maxlen;    /* total blocks in journal file */
    __be32 s_first;     /* first block of log information */

    /* 0x0018 */
    /* Dynamic information describing the current state of the log */
    __be32 s_sequence; /* first commit ID expected in log */
    __be32 s_start;    /* blocknr of start of log */

    /* 0x0020 */
    /* Error value, as set by jbd2_journal_abort(). */
    __be32 s_errno;

    /* 0x0024 */
    /* Remaining fields are only valid in a version-2 superblock */
    __be32 s_feature_compat;    /* compatible feature set */
    __be32 s_feature_incompat;  /* incompatible feature set */
    __be32 s_feature_ro_compat; /* readonly-compatible feature set */
    /* 0x0030 */
    __u8 s_uuid[16]; /* 128-bit uuid for journal */

    /* 0x0040 */
    __be32 s_nr_users; /* Nr of filesystems sharing log */

    __be32 s_dynsuper; /* Blocknr of dynamic superblock copy*/

    /* 0x0048 */
    __be32 s_max_transaction; /* Limit of journal blocks per trans.*/
    __be32 s_max_trans_data;  /* Limit of data blocks per trans. */

    /* 0x0050 */
    __u8 s_checksum_type; /* checksum type */
    __u8 s_padding2[3];
    __u32 s_padding[42];
    __be32 s_checksum; /* crc32c(superblock) */

    /* 0x0100 */
    __u8 s_users[16 * 48]; /* ids of all fs'es sharing the log */
                           /* 0x0400 */
} journal_superblock_t;

/* Use the jbd2_{has,set,clear}_feature_* helpers; these will be removed */
#define JBD2_HAS_COMPAT_FEATURE(j, mask) \
    ((j)->s_header.h_blocktype == JBD2_SUPERBLOCK_V2 && TEST_BIT((j)->s_feature_compat, mask))
#define JBD2_HAS_RO_COMPAT_FEATURE(j, mask) \
    ((j)->s_header.h_blocktype == JBD2_SUPERBLOCK_V2 && TEST_BIT((j)->s_feature_ro_compat, mask))
#define JBD2_HAS_INCOMPAT_FEATURE(j, mask) \
    ((j)->s_header.h_blocktype == JBD2_SUPERBLOCK_V2 && TEST_BIT((j)->s_feature_incompat, mask))

#define JBD2_FEATURE_COMPAT_CHECKSUM 0x00000001

#define JBD2_FEATURE_INCOMPAT_REVOKE 0x00000001
#define JBD2_FEATURE_INCOMPAT_64BIT 0x00000002
#define JBD2_FEATURE_INCOMPAT_ASYNC_COMMIT 0x00000004
#define JBD2_FEATURE_INCOMPAT_CSUM_V2 0x00000008
#define JBD2_FEATURE_INCOMPAT_CSUM_V3 0x00000010

/*
 * The block tag: used to describe a single buffer in the journal.
 * t_blocknr_high is only used if INCOMPAT_64BIT is set, so this
 * raw struct shouldn't be used for pointer math or sizeof() - use
 * journal_tag_bytes(journal) instead to compute this.
 */
typedef struct journal_block_tag3_s
{
    __be32 t_blocknr;      /* The on-disk block number */
    __be32 t_flags;        /* See below */
    __be32 t_blocknr_high; /* most-significant high 32bits. */
    __be32 t_checksum;     /* crc32c(uuid+seq+block) */
} journal_block_tag3_t;

typedef struct journal_block_tag_s
{
    __be32 t_blocknr;      /* The on-disk block number */
    __be16 t_checksum;     /* truncated crc32c(uuid+seq+block) */
    __be16 t_flags;        /* See below */
    __be32 t_blocknr_high; /* most-significant high 32bits. */
} journal_block_tag_t;

typedef struct jbd2_journal_revoke_header_s
{
    journal_header_t r_header;
    __be32 r_count; /* Count of bytes used in the block */
} jbd2_journal_revoke_header_t;

#define REVOKE_FLAG 0x7EDABC12
#define COMMIT_FLAG 0x71234567

/* 如果tag太多就不再解析了 */
#define MAX_TAG_NUM 128

static inline bool jbd2_has_feature_csum3(journal_superblock_t *journal)
{
    return JBD2_HAS_INCOMPAT_FEATURE(journal, JBD2_FEATURE_INCOMPAT_CSUM_V3);
}

static inline bool jbd2_has_feature_csum2(journal_superblock_t *journal)
{
    return JBD2_HAS_INCOMPAT_FEATURE(journal, JBD2_FEATURE_INCOMPAT_CSUM_V2);
}

static inline bool jbd2_has_feature_64bit(journal_superblock_t *journal)
{
    return JBD2_HAS_INCOMPAT_FEATURE(journal, JBD2_FEATURE_INCOMPAT_64BIT);
}

/*
 * Checksum types.
 */
#define JBD2_CRC32_CHKSUM 1
#define JBD2_MD5_CHKSUM 2
#define JBD2_SHA1_CHKSUM 3
#define JBD2_CRC32C_CHKSUM 4

#define JBD2_CRC32_CHKSUM_SIZE 4

#define JBD2_CHECKSUM_BYTES (32 / sizeof(uint32_t))
/*
 * Commit block header for storing transactional checksums:
 *
 * NOTE: If FEATURE_COMPAT_CHECKSUM (checksum v1) is set, the h_chksum*
 * fields are used to store a checksum of the descriptor and data blocks.
 *
 * If FEATURE_INCOMPAT_CSUM_V2 (checksum v2) is set, then the h_chksum
 * field is used to store crc32c(uuid+commit_block).  Each journal metadata
 * block gets its own checksum, and data block checksums are stored in
 * journal_block_tag (in the descriptor).  The other h_chksum* fields are
 * not used.
 *
 * If FEATURE_INCOMPAT_CSUM_V3 is set, the descriptor block uses
 * journal_block_tag3_t to store a full 32-bit checksum.  Everything else
 * is the same as v2.
 *
 * Checksum v1, v2, and v3 are mutually exclusive features.
 */
struct commit_header
{
    __be32 h_magic;
    __be32 h_blocktype;
    __be32 h_sequence;
    unsigned char h_chksum_type;
    unsigned char h_chksum_size;
    unsigned char h_padding[2];
    __be32 h_chksum[JBD2_CHECKSUM_BYTES];
    __be64 h_commit_sec;
    __be32 h_commit_nsec;
};

static inline void swfun(uint8_t *buf, uint8_t len)
{
    uint8_t i = 0;
    uint8_t t;
    for (i = 0; i < (len >> 1); i++)
    {
        t = buf[i];
        buf[i] = buf[len - 1 - i];
        buf[len - 1 - i] = t;
    }
}

#define conversion(data)                       \
    {                                          \
        swfun((uint8_t *)&data, sizeof(data)); \
    }

/*********************************************ext4tool**********************************************/
#define JOURNAL_P_WORD "journal"

#define B_ATTR_SB 0x01
#define B_ATTR_GDT 0x02
#define B_ATTR_RES_GDT 0x04
#define B_ATTR_BB 0x08
#define B_ATTR_IB 0x10
#define B_ATTR_IT 0x20
#define B_ATTR_DATA_NOT_USE 0x40
#define B_ATTR_DATA_USE 0x80
#define UNKNOWN 0x00

#define BLOCK_BITMAP 0
#define INODE_BITMAP 1

#define MAX_PATH_LEN 100
#define P_INODE_STATE 0x80000000

#define DEVICE_MODE 0
#define FILE_MODE 1
#define SPUER_MODE 2

#define DUMP_MODE_EXT 1
#define TREE_LIST_MODE_EXT 2

struct ext4_tool
{
    uint8_t mode;     /* 0:设备模式 1:文件模式 2:强制使用超级块的块大小 */
    uint8_t ext_mode; /* 1: dump数据 2:列出树状inode结构 */
    uint32_t dump_bolcknum;
    int fd;
    char path[MAX_PATH_LEN];      /* 要打开的设备路径 */
    uint32_t p_inode;             /* 要打印的inode数量，或者和inode相关的输出，如果=0，则不解析inode */
    uint32_t ssz;                 /* 扇区大小 */
    uint32_t bsz;                 /* 块大小 */
    uint32_t setbsz;              /* 文件模式下有效，通过传参设置，默认4096 */
    unsigned long long bytes;     /* 块设备总大小 */
    uint64_t bolcknum;            /* 块的数量 */
    uint32_t per_block;           /* 每个组块拥有的块数量 */
    uint32_t per_inode;           /* 每个组块拥有的inode数量 */
    uint16_t inode_size;          /* inode的大小 */
    uint32_t groupnum;            /* 组块数量,正常一个块组128M=32768*4K,该值为总大小/128M */
    uint32_t inodeblknum;         /* inode占用的klock的大小 每块中能包含的inode=per_inode/inodeblknum */
    uint32_t group_desc_blk_num;  /* 组描述符所占用的块大小 */
    uint32_t reserved_gdt_blocks; /* 预留块组 */
    uint32_t flex_bg_num;         /* 柔性分组的大小 */
    uint32_t mate_bg_num;         /* 元组块数量，只有开启了mate_bg特性才有用 */
    uint8_t group_desc_len;       /* 组描述符大小 */
    uint8_t *buf;                 /* 按块进行读取 */
    uint8_t *extbuf;              /* 对日志或者目录解析操作时使用 */
    uint8_t *jnl_data_buf;        /* 读取日志块时使用 */
    struct ext4_super_block *sb;
    union feature_compat f_compat;
    union feature_incompat f_incompat;
    union feature_rocompat f_rocompat;
    uint8_t *group_desc_buf; /* 存放组块描述符 */
    uint8_t *block_bit_buf;  /* 存放 blockbitmap */
    uint16_t block_bit_pos;  /* 定位存放 blockbitmap 的位置 */
    uint8_t *inode_bit_buf;  /* 存放 inodebitmap */
    uint16_t inode_bit_pos;  /* 定位存放 inodebitmap 的位置 */
    uint16_t extent_num;
    struct ext4_extent ex[4];
    journal_superblock_t jnlsuper;
    uint64_t *loc_map_buf; /* 叶子节点逻辑映射实际物理块缓存 */
};

#define EXT4_TOOL_LEN sizeof(struct ext4_tool)

struct inode_show_info
{
    int group;
    struct ext4_inode *inode;
    uint32_t inode_num;
    int type;
};

#ifndef BLKROSET
#define BLKROSET _IO(0x12, 93)    /* set device read-only (0 = read-write) */
#define BLKROGET _IO(0x12, 94)    /* get read-only status (0 = read_write) */
#define BLKRRPART _IO(0x12, 95)   /* re-read partition table */
#define BLKGETSIZE _IO(0x12, 96)  /* return device size /512 (long *arg) */
#define BLKFLSBUF _IO(0x12, 97)   /* flush buffer cache */
#define BLKRASET _IO(0x12, 98)    /* set read ahead for block device */
#define BLKRAGET _IO(0x12, 99)    /* get current read ahead setting */
#define BLKFRASET _IO(0x12, 100)  /* set filesystem (mm/filemap.c) read-ahead */
#define BLKFRAGET _IO(0x12, 101)  /* get filesystem (mm/filemap.c) read-ahead */
#define BLKSECTSET _IO(0x12, 102) /* set max sectors per request (ll_rw_blk.c) */
#define BLKSECTGET _IO(0x12, 103) /* get max sectors per request (ll_rw_blk.c) */
#define BLKSSZGET _IO(0x12, 104)  /* get block device sector size */

/* ioctls introduced in 2.2.16, removed in 2.5.58 */
#define BLKELVGET _IOR(0x12, 106, size_t) /* elevator get */
#define BLKELVSET _IOW(0x12, 107, size_t) /* elevator set */

#define BLKBSZGET _IOR(0x12, 112, size_t)
#define BLKBSZSET _IOW(0x12, 113, size_t)
#endif /* !BLKROSET */

#ifndef BLKGETSIZE64
#define BLKGETSIZE64 _IOR(0x12, 114, size_t) /* return device size in bytes (u64 *arg) */
#endif

#define GET_NUN(a, b, c) \
    {                    \
        c = (a) / (b);   \
        if ((a) % (b))   \
            c += 1;      \
    }

static inline void ext4_bitmap_set(uint8_t type, uint8_t attr, struct ext4_tool *dat)
{
    if (BLOCK_BITMAP == type)
    {
        if (dat->block_bit_pos >= dat->per_block)
            dat->block_bit_pos = 0;
        SET_BIT(dat->block_bit_buf[dat->block_bit_pos], attr);
        dat->block_bit_pos++;
    }
    else
    {
        if (dat->inode_bit_pos >= dat->per_inode)
            dat->inode_bit_pos = 0;
        SET_BIT(dat->inode_bit_buf[dat->inode_bit_pos], attr);
        dat->inode_bit_pos++;
    }
}

static inline void ext4_pos_zero(struct ext4_tool *dat)
{
    dat->block_bit_pos = 0;
    dat->inode_bit_pos = 0;
}

#endif

/*
struct feature {
    int		compat;
    unsigned int	mask;
    const char	*string;
};
static struct feature feature_list[] = {
    {	E2P_FEATURE_COMPAT, EXT2_FEATURE_COMPAT_DIR_PREALLOC,
            "dir_prealloc" },
    {	E2P_FEATURE_COMPAT, EXT3_FEATURE_COMPAT_HAS_JOURNAL,
            "has_journal" },
    {	E2P_FEATURE_COMPAT, EXT2_FEATURE_COMPAT_IMAGIC_INODES,
            "imagic_inodes" },
    {	E2P_FEATURE_COMPAT, EXT2_FEATURE_COMPAT_EXT_ATTR,
            "ext_attr" },
    {	E2P_FEATURE_COMPAT, EXT2_FEATURE_COMPAT_DIR_INDEX,
            "dir_index" },
    {	E2P_FEATURE_COMPAT, EXT2_FEATURE_COMPAT_RESIZE_INODE,
            "resize_inode" },
    {	E2P_FEATURE_COMPAT, EXT2_FEATURE_COMPAT_LAZY_BG,
            "lazy_bg" },
    {	E2P_FEATURE_COMPAT, EXT2_FEATURE_COMPAT_EXCLUDE_BITMAP,
            "snapshot_bitmap" },
    {	E2P_FEATURE_COMPAT, EXT4_FEATURE_COMPAT_SPARSE_SUPER2,
            "sparse_super2" },

    {	E2P_FEATURE_RO_INCOMPAT, EXT2_FEATURE_RO_COMPAT_SPARSE_SUPER,
            "sparse_super" },
    {	E2P_FEATURE_RO_INCOMPAT, EXT2_FEATURE_RO_COMPAT_LARGE_FILE,
            "large_file" },
    {	E2P_FEATURE_RO_INCOMPAT, EXT4_FEATURE_RO_COMPAT_HUGE_FILE,
            "huge_file" },
    {	E2P_FEATURE_RO_INCOMPAT, EXT4_FEATURE_RO_COMPAT_GDT_CSUM,
            "uninit_bg" },
    {	E2P_FEATURE_RO_INCOMPAT, EXT4_FEATURE_RO_COMPAT_GDT_CSUM,
            "uninit_groups" },
    {	E2P_FEATURE_RO_INCOMPAT, EXT4_FEATURE_RO_COMPAT_DIR_NLINK,
            "dir_nlink" },
    {	E2P_FEATURE_RO_INCOMPAT, EXT4_FEATURE_RO_COMPAT_EXTRA_ISIZE,
            "extra_isize" },
    {	E2P_FEATURE_RO_INCOMPAT, EXT4_FEATURE_RO_COMPAT_QUOTA,
            "quota" },
    {	E2P_FEATURE_RO_INCOMPAT, EXT4_FEATURE_RO_COMPAT_BIGALLOC,
            "bigalloc"},
    {	E2P_FEATURE_RO_INCOMPAT, EXT4_FEATURE_RO_COMPAT_METADATA_CSUM,
            "metadata_csum"},
    {	E2P_FEATURE_RO_INCOMPAT, EXT4_FEATURE_RO_COMPAT_REPLICA,
            "replica" },
    {	E2P_FEATURE_RO_INCOMPAT, EXT4_FEATURE_RO_COMPAT_READONLY,
            "read-only" },
    {	E2P_FEATURE_RO_INCOMPAT, EXT4_FEATURE_RO_COMPAT_PROJECT,
            "project"},
    {	E2P_FEATURE_RO_INCOMPAT, EXT4_FEATURE_RO_COMPAT_SHARED_BLOCKS,
            "shared_blocks"},
    {	E2P_FEATURE_RO_INCOMPAT, EXT4_FEATURE_RO_COMPAT_VERITY,
            "verity"},

    {	E2P_FEATURE_INCOMPAT, EXT2_FEATURE_INCOMPAT_COMPRESSION,
            "compression" },
    {	E2P_FEATURE_INCOMPAT, EXT2_FEATURE_INCOMPAT_FILETYPE,
            "filetype" },
    {	E2P_FEATURE_INCOMPAT, EXT3_FEATURE_INCOMPAT_RECOVER,
            "needs_recovery" },
    {	E2P_FEATURE_INCOMPAT, EXT3_FEATURE_INCOMPAT_JOURNAL_DEV,
            "journal_dev" },
    {	E2P_FEATURE_INCOMPAT, EXT3_FEATURE_INCOMPAT_EXTENTS,
            "extent" },
    {	E2P_FEATURE_INCOMPAT, EXT3_FEATURE_INCOMPAT_EXTENTS,
            "extents" },
    {	E2P_FEATURE_INCOMPAT, EXT2_FEATURE_INCOMPAT_META_BG,
            "meta_bg" },
    {	E2P_FEATURE_INCOMPAT, EXT4_FEATURE_INCOMPAT_64BIT,
            "64bit" },
    {       E2P_FEATURE_INCOMPAT, EXT4_FEATURE_INCOMPAT_MMP,
            "mmp" },
    {       E2P_FEATURE_INCOMPAT, EXT4_FEATURE_INCOMPAT_FLEX_BG,
            "flex_bg"},
    {       E2P_FEATURE_INCOMPAT, EXT4_FEATURE_INCOMPAT_EA_INODE,
            "ea_inode"},
    {       E2P_FEATURE_INCOMPAT, EXT4_FEATURE_INCOMPAT_DIRDATA,
            "dirdata"},
    {       E2P_FEATURE_INCOMPAT, EXT4_FEATURE_INCOMPAT_CSUM_SEED,
            "metadata_csum_seed"},
    {       E2P_FEATURE_INCOMPAT, EXT4_FEATURE_INCOMPAT_LARGEDIR,
            "large_dir"},
    {       E2P_FEATURE_INCOMPAT, EXT4_FEATURE_INCOMPAT_INLINE_DATA,
            "inline_data"},
    {       E2P_FEATURE_INCOMPAT, EXT4_FEATURE_INCOMPAT_ENCRYPT,
            "encrypt"},
    {	0, 0, 0 },
};
*/
```
## main.c

```c
#include "def.h"

/* 全局变量 */
int outlvl = 0;
dir_entry_node_t *sorted_list = NULL;

#define CHAR_TYPE 0
#define SHORT_TYPE 1
#define INT_TYPE 2
#define BUF_SIZE 4096

static void put_16_data(int type, uint8_t *dat)
{
	int i = 0;
	uint8_t *memdat = dat;
	uint32_t *uip = (void *)dat;
	uint16_t *usp = (void *)dat;
	uint8_t *ucp = (void *)dat;
	if (CHAR_TYPE == type) {
		for (i = 0; i < 16; i++) {
			printf("%02x ", *ucp);
			ucp++;
		}
	} else if (SHORT_TYPE == type) {
		for (i = 0; i < 16; i += 2) {
			printf("%04x ", *usp);
			usp++;
		}
	} else {
		for (i = 0; i < 16; i += 4) {
			printf("%08x ", *uip);
			uip++;
		}
	}
	printf(" | ");
	for (i = 0; i < 16; i++) {
		printf("%c", isprint(*memdat) && (*memdat < 0x80) ? *memdat : '.');
		memdat++;
	}
	printf(" |\n");
}

static int compare(uint8_t *buf1, uint8_t *buf2, uint8_t len)
{
	int i;
	for (i = 0; i < len; i++) {
		if (buf1[i] != buf2[i])
			return -1;
	}
	return 0;
}

static void show_data_all(uint8_t *buf, uint32_t len)
{
	uint16_t dolen = 0;
	uint16_t rlen; /* 一次最大读取长度为 4096+16 */
	uint32_t slen = len; /* 剩余未读取长度 */
	uint32_t tlen = 0; /* 以读取长度 */
	int j;
	uint8_t tbuf[16];
	uint8_t flag = 0, showall = 0;
	uint16_t datlen = 16;

	while (slen) {
		dolen = 0;
		if (slen < BUF_SIZE) {
			rlen = (uint16_t)slen;
			slen = 0;
		} else {
			rlen = BUF_SIZE;
			slen -= BUF_SIZE;
		}

		j = 0;
		while (dolen < rlen) {
			if (!showall) {
				if (j == 0 && flag == 1) {
					/* 说明有上一次遗留 */
					if (0 == compare(tbuf, &buf[j * datlen], datlen))
						goto done;
					else
						flag = 0;
				} else if (j > 0) {
					if (0 == compare(&buf[(j - 1) * datlen], &buf[j * datlen], datlen)) {
						/* 说明缓存里的数据完全一致 */
						if (!flag) {
							printf("*** \n");
							flag = 1;
						}
						goto done;
					} else {
						flag = 0;
					}
				}
			}
			printf("[%08x] ", tlen + dolen);
			put_16_data(INT_TYPE, &buf[j * datlen]);
done:
			dolen += datlen;
			j++;
		}
		tlen += rlen;
		if (flag) {
			if (slen == 0)
				printf("[%08x]\n", tlen);
			else
				memcpy(tbuf, &buf[(j - 1) * datlen], datlen);
		}
	}
}

static const char *get_ext4_os(int type)
{
	switch (type) {
	case EXT4_OS_LINUX:
		return "Linux";
	case EXT4_OS_Hurd:
		return "Hurd";
	case EXT4_OS_Masix:
		return "Masix";
	case EXT4_OS_FreeBSD:
		return "FreeBSD";
	case EXT4_OS_Lites:
		return "Lites";
	default:
		return "unknown";
	}
}

static const char *get_ext4_err_behaviour(int type)
{
	switch (type) {
	case EXT4_SERR_Continue:
		return "Continue";
	case EXT4_SERR_Remount:
		return "Remount read-only";
	case EXT4_SERR_Panic:
		return "Panic";
	default:
		return "unknown";
	}
}

static const char *get_ext4_File_system_state(int type)
{
	switch (type) {
	case EXT4_STATE_Cleanly:
		return "Clean";
	case EXT4_STATE_Errors:
		return "Errors detected";
	case EXT4_STATE_Orphans:
		return "Orphans being recovered";
	default:
		return "unknown";
	}
}

static const char *get_ext4_Magic_chk(struct ext4_tool *dat)
{
	int type = dat->sb->s_magic;
	if (type == EXT4_Magic)
		return "Magic correct";
	else
		return "Magic err";
}

static const char *get_ext4_Revision_level(int type)
{
	switch (type) {
	case EXT4_REV_LVL_Original:
		return "Original format";
	case EXT4_REV_LVL_v2:
		return "v2 format w/ dynamic inode sizes";
	default:
		return "unknown";
	}
}

static const char *get_ext4_Miscellaneous_flags(int type)
{
	switch (type) {
	case EXT4_MISC_F_Signed:
		return "Signed directory hash in use";
	case EXT4_MISC_F_Unsigned:
		return "Unsigned directory hash in use";
	case EXT4_MISC_F_test:
		return "To test development code";
	default:
		return "unknown";
	}
}

static void put_ext4_feature_compat(int lvl, struct ext4_tool *dat)
{
	int type = dat->sb->s_feature_compat;
	pinfo(lvl, "", "Compatible feature: ");
	{
		if (TEST_BIT(type, EXT4_FEATURE_COMPAT_DIR_PREALLOC)) {
			pinfo_n(lvl, "Directory preallocation; ");
			dat->f_compat.dir_prealloc = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_COMPAT_IMAGIC_INODES)) {
			pinfo_n(lvl, "imagic inodes; ");
			dat->f_compat.imagic_inodes = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_COMPAT_HAS_JOURNAL)) {
			pinfo_n(lvl, "has_journal; ");
			dat->f_compat.has_journal = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_COMPAT_EXT_ATTR)) {
			pinfo_n(lvl, "Supports extended attributes(ext_attr); ");
			dat->f_compat.ext_attr = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_COMPAT_RESIZE_INODE)) {
			pinfo_n(lvl, "Has reserved GDT blocks(resize_inode); ");
			dat->f_compat.resize_inode = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_COMPAT_DIR_INDEX)) {
			pinfo_n(lvl, "Has indexed directories(dir_index); ");
			dat->f_compat.dir_index = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_COMPAT_LAZY_BG)) {
			pinfo_n(lvl, "Lazy BG; ");
			dat->f_compat.lazy_bg = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_COMPAT_EXCLUDE_INODE)) {
			pinfo_n(lvl, "Exclude inode; ");
			dat->f_compat.exclude_inode = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_COMPAT_EXCLUDE_BITMAP)) {
			pinfo_n(lvl, "Exclude bitmap; ");
			dat->f_compat.exclude_bitmap = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_COMPAT_SPARSE_SUPER2)) {
			pinfo_n(lvl, "Sparse Super Block; ");
			dat->f_compat.sparse_super2 = 1;
		}

		if (type == 0)
			pinfo_n(lvl, "none; ");
	}
	pinfo_n(lvl, "\n");
}

static void put_ext4_feature_incompat(int lvl, struct ext4_tool *dat)
{
	int type = dat->sb->s_feature_incompat;
	pinfo(lvl, "", "Incompatible feature: ");
	{
		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_COMPRESSION)) {
			pinfo_n(lvl, "Compression. Not implemented; ");
			dat->f_incompat.compression = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_FILETYPE)) {
			/* 将文件类型信息存储在目录条目中 */
			pinfo_n(lvl, "Directory entries record the file type(filetype); ");
			dat->f_incompat.file_type = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_RECOVER)) {
			pinfo_n(lvl, "Filesystem needs journal recovery; ");
			dat->f_incompat.journal_recovery = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_JOURNAL_DEV)) {
			pinfo_n(lvl, "Use separate journal device; ");
			dat->f_incompat.journal_device = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_META_BG)) {
			pinfo_n(lvl, "meta_bg; ");
			dat->f_incompat.meta_bg = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_EXTENTS)) {
			pinfo_n(lvl, "File use extents(extent); ");
			dat->f_incompat.use_extents = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_64BIT)) {
			pinfo_n(lvl, "64BIT; ");
			dat->f_incompat.use_64bit = 1;
			dat->group_desc_len = EXT4_GROUP_DESC_64bit_LEN;
		} else {
			dat->group_desc_len = EXT4_GROUP_DESC_LEN;
		}

		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_MMP)) {
			pinfo_n(lvl, "Multiple mount protection; ");
			dat->f_incompat.mul_mount_protection = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_FLEX_BG)) {
			pinfo_n(lvl, "flex_bg; ");
			dat->f_incompat.flex_bg = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_EA_INODE)) {
			pinfo_n(lvl, "large extended attribute; ");
			dat->f_incompat.ea_inode = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_DIRDATA)) {
			pinfo_n(lvl, "Data in directory entry; ");
			dat->f_incompat.dirdata = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_CSUM_SEED)) {
			pinfo_n(lvl, "Metadata checksum seed is stored in the superblock; ");
			dat->f_incompat.csum_seed = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_LARGEDIR)) {
			pinfo_n(lvl, "Large directory; ");
			dat->f_incompat.large_dir = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_INLINE_DATA)) {
			pinfo_n(lvl, "Data in inode; ");
			dat->f_incompat.inline_data = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_INCOMPAT_ENCRYPT)) {
			pinfo_n(lvl, "Encrypted inodes; ");
			dat->f_incompat.encrypt = 1;
		}

		if (type == 0)
			pinfo_n(lvl, "none; ");
	}
	pinfo_n(lvl, "\n");
}

static void put_ext4_feature_rocompat(int lvl, struct ext4_tool *dat)
{
	int type = dat->sb->s_feature_ro_compat;
	pinfo(lvl, "", "Readonly-compatible feature: ");
	{
		if (TEST_BIT(type, EXT4_FEATURE_RO_COMPAT_SPARSE_SUPER)) {
			pinfo_n(lvl, "sparse_super; ");
			dat->f_rocompat.sparse_super = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_RO_COMPAT_LARGE_FILE)) {
			pinfo_n(lvl, "Allow files larger than 2GiB(large_file); ");
			dat->f_rocompat.large_file = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_RO_COMPAT_HUGE_FILE)) {
			pinfo_n(lvl, "huge_file; ");
			dat->f_rocompat.huge_dir = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_RO_COMPAT_GDT_CSUM)) {
			/* 如果启用了uninit_bg特性，那么mke2fs不会完全初始化inode表 */
			pinfo_n(lvl, "Group descriptors have checksums(uninit_bg); ");
			dat->f_rocompat.gdt_csum = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_RO_COMPAT_DIR_NLINK)) {
			pinfo_n(lvl, "dir_nlink; ");
			dat->f_rocompat.dir_nlink = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_RO_COMPAT_EXTRA_ISIZE)) {
			pinfo_n(lvl, "large inodes(extra_isize); ");
			dat->f_rocompat.extra_isize = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_RO_COMPAT_QUOTA)) {
			pinfo_n(lvl, "Quota is handled transactionally with the journal; ");
			dat->f_rocompat.quota = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_RO_COMPAT_BIGALLOC)) {
			pinfo_n(lvl, "This filesystem supports bigalloc; ");
			dat->f_rocompat.bigalloc = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_RO_COMPAT_METADATA_CSUM)) {
			pinfo_n(lvl, "supports metadata checksumming; ");
			dat->f_rocompat.metadata_csum = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_RO_COMPAT_READONLY)) {
			pinfo_n(lvl, "Read-only filesystem image; ");
			dat->f_rocompat.readonly = 1;
		}

		if (TEST_BIT(type, EXT4_FEATURE_RO_COMPAT_PROJECT)) {
			pinfo_n(lvl, "Filesystem tracks project quotas; ");
			dat->f_rocompat.project = 1;
		}

		if (type == 0)
			pinfo_n(lvl, "none; ");
	}
	pinfo_n(lvl, "\n");
}

static void put_ext4_uuid(int lvl, uint8_t *uid)
{
	/* 18b95711-55c7-4ee3-a1b3-67faa49dd297 */
	pinfo(lvl, "", "uuid for volume:");
	pinfo_n(lvl, "%x%x%x%x-%x%x-%x%x-%x%x-%x%x%x%x%x%x\n", uid[0], uid[1], uid[2], uid[3], uid[4], uid[5], uid[6], uid[7],
		uid[8], uid[9], uid[0], uid[11], uid[12], uid[13], uid[14], uid[15]);
}

static void put_ext4_journal_uuid(int lvl, uint8_t *uid)
{
	/* 18b95711-55c7-4ee3-a1b3-67faa49dd297 */
	pinfo(lvl, "", "uuid of journal superblock:");
	pinfo_n(lvl, "%x%x%x%x-%x%x-%x%x-%x%x-%x%x%x%x%x%x\n", uid[0], uid[1], uid[2], uid[3], uid[4], uid[5], uid[6], uid[7],
		uid[8], uid[9], uid[0], uid[11], uid[12], uid[13], uid[14], uid[15]);
}

static void put_journal_tag_uuid(int lvl, uint8_t *uid)
{
	/* 18b95711-55c7-4ee3-a1b3-67faa49dd297 */
	pinfo_n(lvl, "[%s]:tag uuid:", JOURNAL_P_WORD);
	pinfo_n(lvl, "%x%x%x%x-%x%x-%x%x-%x%x-%x%x%x%x%x%x\n", uid[0], uid[1], uid[2], uid[3], uid[4], uid[5], uid[6], uid[7],
		uid[8], uid[9], uid[0], uid[11], uid[12], uid[13], uid[14], uid[15]);
}

static void put_ext4_default_mount_opts(int lvl, struct ext4_tool *dat)
{
	int type = dat->sb->s_default_mount_opts;
	pinfo(lvl, "0x%x  ", "Default mount options:", dat->sb->s_default_mount_opts);
	{
		if (TEST_BIT(type, EXT4_DEFM_DEBUG))
			pinfo_n(lvl, "Print debugging info upon (re)mount; ");
		if (TEST_BIT(type, EXT4_DEFM_BSDGROUPS))
			pinfo_n(lvl, "New files take the gid of the containing directory; ");
		if (TEST_BIT(type, EXT4_DEFM_XATTR_USER))
			pinfo_n(lvl, "Support userspace-provided extended attributes; ");
		if (TEST_BIT(type, EXT4_DEFM_ACL))
			pinfo_n(lvl, "Support POSIX access control lists; ");
		if (TEST_BIT(type, EXT4_DEFM_UID16))
			pinfo_n(lvl, "Do not support 32-bit UIDs; ");
		if (TEST_BIT(type, EXT4_DEFM_JMODE_DATA))
			pinfo_n(lvl, "JOUTNAL; ");
		if (TEST_BIT(type, EXT4_DEFM_JMODE_ORDERED))
			pinfo_n(lvl, "ORDERED; ");
		if (TEST_BIT(type, EXT4_DEFM_JMODE_WBACK))
			pinfo_n(lvl, "WRITE BACK; ");
		if (TEST_BIT(type, EXT4_DEFM_NOBARRIER))
			pinfo_n(lvl, "Disable write flushes; ");
		if (TEST_BIT(type, EXT4_DEFM_BLOCK_VALIDITY))
			pinfo_n(lvl,
				"Track which blocks in a filesystem are metadata and therefore should not be used as data blocks; ");
		if (TEST_BIT(type, EXT4_DEFM_DISCARD))
			pinfo_n(lvl, "Enable DISCARD support, where the storage device is told about blocks becoming unused; ");
		if (TEST_BIT(type, EXT4_DEFM_NODELALLOC))
			pinfo_n(lvl, "Disable delayed allocation; ");
		if (type == 0)
			pinfo_n(lvl, "none; ");
	}
	pinfo_n(lvl, "\n");
}

static int ext4_super_analysis(struct ext4_tool *dat)
{
	struct ext4_super_block *sb = dat->sb;
	uint32_t superblocksize = 1 << (10 + sb->s_log_block_size);
	int lvl = EXT4_LVL_SUPER;
	time_t t;
	if (sb->s_magic != EXT4_Magic) {
		perr("magic err!");
		return -1;
	}

	if (dat->bsz != superblocksize) {
		pwarn("Block size err! We get:%d,super:%d", dat->bsz, superblocksize);
		dat->bsz = superblocksize;
		pwarn("We change block size to:%d", dat->bsz);
		dat->mode = SPUER_MODE;
		return 1;
	}

	if (dat->per_block != sb->s_blocks_per_group) {
		pwarn("Blocks per group err! We count:%d,super:%d", dat->per_block, sb->s_blocks_per_group);
		dat->per_block = sb->s_blocks_per_group;
		pwarn("We change blocks per group to:%d", dat->per_block);
		dat->mode = SPUER_MODE;
		return 1;
	}

	/*00*/
	pinfo(lvl, "%d\n", "Inodes count:", sb->s_inodes_count);
	pinfo(lvl, "%d\n", "Blocks count:", sb->s_blocks_count_lo);
	dat->bolcknum = sb->s_blocks_count_lo;
	pinfo(lvl, "%d\n", "Reserved blocks count:", sb->s_r_blocks_count_lo);
	pinfo(lvl, "%d\n", "Free blocks count:", sb->s_free_blocks_count_lo);
	/*10*/
	pinfo(lvl, "%d\n", "Free inodes count:", sb->s_free_inodes_count);
	pinfo(lvl, "%d\n", "First Data Block:", sb->s_first_data_block);
	pinfo(lvl, "%d\n", "Log Block size:", superblocksize);
	pinfo(lvl, "%d\n", "Allocation cluster size:", sb->s_log_cluster_size); /* 分配群集大小 */
	/*20*/
	pinfo(lvl, "%d\n", "Blocks per group:", sb->s_blocks_per_group);
	pinfo(lvl, "%d\n", "Clusters per group:", sb->s_clusters_per_group);
	pinfo(lvl, "%d\n", "Inodes per group:", sb->s_inodes_per_group);
	dat->per_inode = sb->s_inodes_per_group;
	t = (time_t)sb->s_mtime;
	pinfo(lvl, "%s", "Mount time:", asctime(localtime(&t)));
	// /*30*/
	t = (time_t)sb->s_wtime;
	pinfo(lvl, "%s", "Write time:", asctime(localtime(&t)));
	pinfo(lvl, "%hd\n", "Mount count:", sb->s_mnt_count);
	pinfo(lvl, "%hd\n", "Maximal mount count:", sb->s_max_mnt_count);
	pinfo(lvl, "0x%x,%s\n", "Magic signature:", sb->s_magic, get_ext4_Magic_chk(dat));
	pinfo(lvl, "%s\n", "File system state:", get_ext4_File_system_state(sb->s_state));
	pinfo(lvl, "%s\n", "Errors behavior:", get_ext4_err_behaviour(sb->s_errors));
	pinfo(lvl, "%hd\n", "minor revision level:", sb->s_minor_rev_level);
	/*40*/
	t = (time_t)sb->s_lastcheck;
	pinfo(lvl, "%s", "Last checked:", asctime(localtime(&t)));
	pinfo(lvl, "%d\n", "Check interval:", sb->s_checkinterval);
	pinfo(lvl, "%s\n", "Filesystem OS type:", get_ext4_os(sb->s_creator_os));
	if (EXT4_OS_LINUX != sb->s_creator_os) {
		perr("This tool only for linux!!!");
		return -1;
	}
	pinfo(lvl, "%s\n", "Filesystem revision:", get_ext4_Revision_level(sb->s_rev_level));
	/*50*/
	pinfo(lvl, "%hd\n", "Reserved blocks uid:", sb->s_def_resuid);
	pinfo(lvl, "%hd\n", "Reserved blocks gid:", sb->s_def_resgid);

	if (sb->s_rev_level != EXT4_DYNAMIC_REV) {
		return -1;
		/* 如果非DYNAMIC，会有很多东西解析不出来，直接pass */
	}
	/* These fields are for EXT4_DYNAMIC_REV superblocks only. */
	pinfo(lvl, "%d\n", "First inode:", sb->s_first_ino);
	pinfo(lvl, "%hd\n", "Inode size:", sb->s_inode_size);
	dat->inode_size = sb->s_inode_size;
	pinfo(lvl, "%hd\n", "block group of this superblock:", sb->s_block_group_nr);
	put_ext4_feature_compat(lvl, dat);
	/*60*/
	put_ext4_feature_incompat(lvl, dat);
	put_ext4_feature_rocompat(lvl, dat);
	if (dat->f_compat.resize_inode && dat->f_incompat.meta_bg) {
		/* 这两个特性是互斥的，如果同时存在肯定有问题 */
		perr("There are incompatible features");
		return -1;
	}
	/*68*/
	put_ext4_uuid(lvl, sb->s_uuid);
	/*78*/
	pinfo(lvl, "%s\n", "volume name:", sb->s_volume_name);
	/*88*/
	pinfo(lvl, "%s\n", "directory where last mounted:", sb->s_last_mounted);
	/*C8 For compression (Not used in e2fsprogs/Linux)*/

	/*CE*/
	pinfo(lvl, "%d\n", "reserved GDT entries", sb->s_reserved_gdt_blocks);
	dat->reserved_gdt_blocks = sb->s_reserved_gdt_blocks;
	if (dat->f_compat.has_journal) {
		/* Journaling support valid if EXT4_FEATURE_COMPAT_HAS_JOURNAL set. */
		/*D0*/
		put_ext4_journal_uuid(lvl, sb->s_journal_uuid);
		/*E0*/
		pinfo(lvl, "%d\n", "Journal inode:", sb->s_journal_inum);
		if (dat->f_incompat.journal_device)
			pinfo(lvl, "%d\n", "Device number of journal:", sb->s_journal_dev);
		pinfo(lvl, "%d\n", "start inodes del:", sb->s_last_orphan);
		pinfo(lvl, "%d\n", "group descriptor size:", sb->s_desc_size);
		/*100*/
		put_ext4_default_mount_opts(lvl, dat);
		if (dat->f_incompat.meta_bg)
			pinfo(lvl, "%d\n", "First metablock block group:", sb->s_first_meta_bg);
		t = (time_t)sb->s_mkfs_time;
		pinfo(lvl, "%s", "Filesystem created: ", asctime(localtime(&t)));
	}
	if (dat->f_incompat.use_64bit) {
		/* 64bit support valid if EXT4_FEATURE_COMPAT_64BIT */
		/*150*/
		pinfo(lvl, "%d\n", "High 32-bits block:", sb->s_blocks_count_hi);
		dat->bolcknum += ((uint64_t)sb->s_blocks_count_hi << 32);
		pinfo(lvl, "%d\n", "High 32-bits reserved block:", sb->s_r_blocks_count_hi);
		pinfo(lvl, "%d\n", "High 32-bits free block:", sb->s_free_blocks_count_hi);
	}
	pinfo(lvl, "%d\n", "All inodes have at least bytes:", sb->s_min_extra_isize);
	pinfo(lvl, "%d\n", "New inodes should reserve bytes:", sb->s_want_extra_isize);

	pinfo(lvl, "%s\n", "Miscellaneous flags:", get_ext4_Miscellaneous_flags(sb->s_flags));
	pinfo(lvl, "%d\n", "RAID stride:", sb->s_raid_stride);
	pinfo(lvl, "%d\n", "seconds to wait in MMP checking:", sb->s_mmp_update_interval);
	pinfo(lvl, "%lld\n", "Block for multi-mount protection:", sb->s_mmp_block);
	pinfo(lvl, "%d\n", "blocks on all data disks:", sb->s_raid_stripe_width);

	pinfo(lvl, "%d\n", "FLEX_BG group size:", 1 << sb->s_log_groups_per_flex);
	dat->flex_bg_num = 1 << sb->s_log_groups_per_flex;
	/* Metadata checksum algorithm type. The only valid value is 1 (crc32c). */
	pinfo(lvl, "%d\n", "metadata checksum algorithm used:", sb->s_checksum_type);
	pinfo(lvl, "%d\n", "versioning level for encryption:", sb->s_encryption_level);
	pinfo(lvl, "%d\n", "Padding to next 32bits:", sb->s_reserved_pad);
	pinfo(lvl, "%lld\n", "nr of lifetime kilobytes written:", sb->s_kbytes_written);

	pinfo(lvl, "%d\n", "Inode number of active snapshot:", sb->s_snapshot_inum);
	pinfo(lvl, "%d\n", "sequential ID of active snapshot:", sb->s_snapshot_id);
	pinfo(lvl, "%lld\n", "reserved blocks active snapshot's:", sb->s_snapshot_r_blocks_count);
	pinfo(lvl, "%d\n", "snapshot head inode number:", sb->s_snapshot_list);

	pinfo(lvl, "%x\n", "crc32c(superblock):", sb->s_checksum);
	return 0;
}

static void put_ext4_EXT4_BG_flags(int lvl, uint16_t type)
{
	pinfo_n(lvl, "    EXT4_BG_flags: ");
	if (TEST_BIT(type, EXT4_BG_INODE_UNINIT))
		pinfo_n(lvl, "Inode table/bitmap not in use; ");
	if (TEST_BIT(type, EXT4_BG_BLOCK_UNINIT))
		pinfo_n(lvl, "Block bitmap not in use; ");
	if (TEST_BIT(type, EXT4_BG_INODE_ZEROED))
		pinfo_n(lvl, "On-disk itable initialized to zero; ");
	if (type == 0)
		pinfo_n(lvl, "none; ");
	pinfo_n(lvl, "\n");
}

static int ext4_bitmap_analysis(int lvl, int group, uint64_t block_num, uint16_t flag, uint8_t type, struct ext4_tool *dat)
{
	/* 位图解析 */
	uint32_t i = 0, j = 0, k;
	uint64_t show_bit0 = 0, show_bit1 = 0;
	uint16_t bit = 0, continuity = 0; /* 用于判断bit(最大32768)是否连续，以及是否是首次不连续 */

	lseek64(dat->fd, (block_num * dat->bsz), SEEK_SET);
	if (read(dat->fd, dat->buf, dat->bsz) != dat->bsz) {
		perr("read data err.\n");
		return -1;
	}

	if (type == BLOCK_BITMAP) {
		/* Free blocks: 87-32767 也有可能不连续 */
		if (TEST_BIT(flag, EXT4_BG_BLOCK_UNINIT)) {
			/* bitmap未初始化 */
			pinfo_n(lvl, "    Free blocks: %" PRIu64 "-%" PRIu64 "(uninit)\n", (uint64_t)(group * dat->per_block),
				(uint64_t)((group + 1) * dat->per_block - 1));
			goto end;
		}
		pinfo_n(lvl, "    Free blocks: ");
		/* 最后一块需要额外判断 */
		if (dat->groupnum == group + 1)
			k = (uint32_t)((dat->bolcknum - dat->per_block * group) >> 3);
		else
			k = dat->bsz;
	} else {
		/* Free inodes: 12-48 inode直接从1开始 */
		if (TEST_BIT(flag, EXT4_BG_INODE_UNINIT)) {
			/* bitmap未初始化 */
			pinfo_n(lvl, "    Free inodes: %d-%d (uninit)\n", (group * dat->per_inode + 1),
				((group + 1) * dat->per_inode));
			goto end;
		}
		pinfo_n(lvl, "    Free inodes: ");
		k = dat->per_inode >> 3;
	}

	for (i = 0; i < k; i++) {
		for (j = 0; j < 8; j++) {
			bit++;
			if (!TEST_BIT(dat->buf[i], 1 << j)) {
				ext4_bitmap_set(type, B_ATTR_DATA_NOT_USE, dat);
				/* 如果为假，表示该bit对应的块没被使用 对应的块=i*8+j */
				if (bit > 1) {
					if (continuity == 0) {
						show_bit0 = (uint64_t)((dat->per_block * group) + (i * 8 + j));
						if (type == INODE_BITMAP)
							show_bit0 += 1; /* 因为inode直接从1开始 */
						continuity = 1;
					} else {
						show_bit1 = (uint64_t)((dat->per_block * group) + (i * 8 + j));
						pinfo_n(lvl, "%" PRIu64 "-%" PRIu64 " ", show_bit0, show_bit1);
						show_bit0 = 0;
						show_bit1 = 0;
						continuity = 0;
					}
				}
				bit = 0;
			} else {
				ext4_bitmap_set(type, B_ATTR_DATA_USE, dat);
				if (continuity) {
					pinfo_n(lvl, "%" PRIu64 " ", show_bit0);
					show_bit0 = 0;
					show_bit1 = 0;
					continuity = 0;
				}
			}
		}
	}

	if (continuity) {
		/* 如果flag一直为1，说明之后一直都是连续的0，即一直未被使用，则将其打印 */
		if (type == BLOCK_BITMAP) {
			show_bit1 = (uint64_t)(dat->per_block * (group + 1) - 1);
			if (show_bit1 > dat->bolcknum)
				show_bit1 = dat->bolcknum - 1;
		} else {
			show_bit1 = (uint64_t)(dat->per_inode * (group + 1));
		}
		pinfo_n(lvl, "%" PRIu64 "-%" PRIu64, show_bit0, show_bit1);
	} else if (bit == 0) {
		/* 说明该组块的所有结点都没被使用 */
		if (type == BLOCK_BITMAP) {
			show_bit0 = (uint64_t)(dat->per_block * group);
			show_bit1 = (uint64_t)(dat->per_block * (group + 1) - 1);
			if (show_bit1 > dat->bolcknum)
				show_bit1 = dat->bolcknum - 1;
		} else {
			show_bit0 = (uint64_t)(dat->per_block * group);
			show_bit1 = (uint64_t)(dat->per_inode * (group + 1));
		}
		pinfo_n(lvl, "%" PRIu64 "-%" PRIu64, show_bit0, show_bit1);
	}

	pinfo_n(lvl, "\n");
end:
	return 0;
}

static int ext4_show_comm(int cnt, int *pos, char *buf, int buflen, const char *str, uint8_t flag, struct ext4_tool *dat)
{
	int memcnt = cnt;
	for (; cnt < dat->per_block; cnt++) {
		if (!TEST_BIT_Z(dat->block_bit_buf[cnt], flag))
			break;
	}
	cnt -= 1;
	if (*pos >= buflen - 30) {
		cnt = dat->per_block;
		*pos += snprintf(buf + (*pos), buflen, "| ... ");
		return cnt;
	}

	if (memcnt != cnt) {
		*pos += snprintf(buf + (*pos), buflen, "| %s[%d-%d] ", str, memcnt, cnt);
	} else {
		*pos += snprintf(buf + (*pos), buflen, "| %s[%d] ", str, memcnt);
	}

	return cnt;
}

static void ext4_show_block_use(int lvl, int group, struct ext4_tool *dat, uint64_t start)
{
	/* 显示ext4块的使用状态图 */
	int i = 0, pos = 0;
	char pbuf[1024] = { 0 };
	int buflen = sizeof(pbuf);
	pinfo_n(lvl, "[gp %-4d pos:%-8" PRIu64 "]: ", group, start);
	for (i = 0; i < dat->per_block; i++) {
		if (TEST_BIT(dat->block_bit_buf[i], B_ATTR_SB)) {
			memset(pbuf, 0, buflen);
			pos += snprintf(pbuf + pos, buflen, "| SB[%d] ", i);
		} else if (TEST_BIT(dat->block_bit_buf[i], B_ATTR_GDT)) {
			i = ext4_show_comm(i, &pos, pbuf, buflen, "GDT", B_ATTR_GDT, dat);
		} else if (TEST_BIT(dat->block_bit_buf[i], B_ATTR_RES_GDT)) {
			i = ext4_show_comm(i, &pos, pbuf, buflen, "RES_GDT", B_ATTR_RES_GDT, dat);
		} else if (TEST_BIT(dat->block_bit_buf[i], B_ATTR_BB)) {
			i = ext4_show_comm(i, &pos, pbuf, buflen, "BB", B_ATTR_BB, dat);
		} else if (TEST_BIT(dat->block_bit_buf[i], B_ATTR_IB)) {
			i = ext4_show_comm(i, &pos, pbuf, buflen, "IB", B_ATTR_IB, dat);
		} else if (TEST_BIT(dat->block_bit_buf[i], B_ATTR_IT)) {
			i = ext4_show_comm(i, &pos, pbuf, buflen, "IT", B_ATTR_IT, dat);
		} else if (TEST_BIT(dat->block_bit_buf[i], B_ATTR_DATA_NOT_USE)) {
			i = ext4_show_comm(i, &pos, pbuf, buflen, "DAT_NU", B_ATTR_DATA_NOT_USE, dat);
		} else if (TEST_BIT(dat->block_bit_buf[i], B_ATTR_DATA_USE)) {
			i = ext4_show_comm(i, &pos, pbuf, buflen, "DAT_U", B_ATTR_DATA_USE, dat);
		} else {
			i = ext4_show_comm(i, &pos, pbuf, buflen, "UNINIT", UNKNOWN, dat);
		}
	}
	pinfo_n(lvl, "%-150s|\n", pbuf);
}

static const char *get_inode_stat(int type)
{
	switch (type) {
	case EXT4_INODE_USE:
		return "U";
	case EXT4_INODE_UNUSE:
		return "N";
	case EXT4_INODE_NOT_INIT:
		return "I";
	default:
		return "E";
	}
}

static void put_ext4_inode_pointers(int lvl, uint32_t *dat)
{
	/* i_block[EXT4_N_BLOCKS=15] */
	pinfo_n(lvl, "[inode]:Pointers to blocks:");
	pinfo_n(lvl, "%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x\n", dat[0], dat[1], dat[2], dat[3], dat[4], dat[5], dat[6],
		dat[7], dat[8], dat[9], dat[10], dat[11], dat[12], dat[13], dat[14]);
}

static void put_file_mode(int lvl, int type)
{
	pinfo_n(lvl, "[inode]:File mode: ");
	/*These are mutually-exclusive file types*/
	if (TEST_BIT(type, EXT4_S_IFIFO)) {
		pinfo_n(lvl, "Fifo ");
	} else if (TEST_BIT(type, EXT4_S_IFCHR)) {
		pinfo_n(lvl, "char ");
	} else if (TEST_BIT(type, EXT4_S_IFDIR)) {
		pinfo_n(lvl, "dir ");
	} else if (TEST_BIT(type, EXT4_S_IFBLK)) {
		pinfo_n(lvl, "blk ");
	} else if (TEST_BIT(type, EXT4_S_IFREG)) {
		pinfo_n(lvl, "file ");
	} else if (TEST_BIT(type, EXT4_S_IFLNK)) {
		pinfo_n(lvl, "link ");
	} else if (TEST_BIT(type, EXT4_S_IFSOCK)) {
		pinfo_n(lvl, "sock ");
	} else {
		pinfo_n(lvl, "unknown ");
	}
	/* Owner */
	pinfo_n(lvl, "%c", TEST_BIT(type, EXT4_S_IRUSR) ? 'r' : '-');
	pinfo_n(lvl, "%c", TEST_BIT(type, EXT4_S_IWUSR) ? 'w' : '-');
	pinfo_n(lvl, "%c", TEST_BIT(type, EXT4_S_IXUSR) ? 'x' : '-');

	/* Group */
	pinfo_n(lvl, "%c", TEST_BIT(type, EXT4_S_IRGRP) ? 'r' : '-');
	pinfo_n(lvl, "%c", TEST_BIT(type, EXT4_S_IWGRP) ? 'w' : '-');
	pinfo_n(lvl, "%c", TEST_BIT(type, EXT4_S_IXGRP) ? 'x' : '-');

	/* Others */
	pinfo_n(lvl, "%c", TEST_BIT(type, EXT4_S_IROTH) ? 'r' : '-');
	pinfo_n(lvl, "%c", TEST_BIT(type, EXT4_S_IWOTH) ? 'w' : '-');
	pinfo_n(lvl, "%c", TEST_BIT(type, EXT4_S_IXOTH) ? 'x' : '-');

	if (TEST_BIT(type, EXT4_S_ISUID)) {
		pinfo_n(lvl, "  Set UID");
	}
	if (TEST_BIT(type, EXT4_S_ISGID)) {
		pinfo_n(lvl, "  Set GID");
	}
	if (TEST_BIT(type, EXT4_S_ISVTX)) {
		pinfo_n(lvl, "  Sticky bit");
	}
	pinfo_n(lvl, "\n");
}

static void put_ext4_file_flags(int lvl, int flag)
{
	pinfo_n(lvl, "[inode]:Inode flags: ");
	{
		if (TEST_BIT(flag, EXT4_SECRM_FL))
			pinfo_n(lvl, "Secure deletion; ");
		if (TEST_BIT(flag, EXT4_UNRM_FL))
			pinfo_n(lvl, "Undelete; ");
		if (TEST_BIT(flag, EXT4_COMPR_FL))
			pinfo_n(lvl, "Compress file; ");
		if (TEST_BIT(flag, EXT4_SYNC_FL))
			pinfo_n(lvl, "Synchronous updates; ");
		if (TEST_BIT(flag, EXT4_IMMUTABLE_FL))
			pinfo_n(lvl, "Immutable file; ");
		if (TEST_BIT(flag, EXT4_APPEND_FL))
			pinfo_n(lvl, "Write only append; ");
		if (TEST_BIT(flag, EXT4_NODUMP_FL))
			pinfo_n(lvl, "Not dump; ");
		if (TEST_BIT(flag, EXT4_NOATIME_FL))
			pinfo_n(lvl, "Not update atime; ");
		if (TEST_BIT(flag, EXT4_DIRTY_FL))
			pinfo_n(lvl, "Dirty compressed file; ");
		if (TEST_BIT(flag, EXT4_COMPRBLK_FL))
			pinfo_n(lvl, "One or more compressed clusters; ");
		if (TEST_BIT(flag, EXT4_NOCOMPR_FL))
			pinfo_n(lvl, "Not compress; ");
		if (TEST_BIT(flag, EXT4_ENCRYPT_FL))
			pinfo_n(lvl, "Encrypted file; ");
		if (TEST_BIT(flag, EXT4_INDEX_FL))
			pinfo_n(lvl, "Hash-indexed directory; ");
		if (TEST_BIT(flag, EXT4_IMAGIC_FL))
			pinfo_n(lvl, "AFS directory; ");
		if (TEST_BIT(flag, EXT4_JOURNAL_DATA_FL))
			pinfo_n(lvl, "Be journaled; ");
		if (TEST_BIT(flag, EXT4_NOTAIL_FL))
			pinfo_n(lvl, "file tail should not be merged; ");
		if (TEST_BIT(flag, EXT4_DIRSYNC_FL))
			pinfo_n(lvl, "Dirsync behaviour; ");
		if (TEST_BIT(flag, EXT4_TOPDIR_FL))
			pinfo_n(lvl, "Top directory; ");
		if (TEST_BIT(flag, EXT4_HUGE_FILE_FL))
			pinfo_n(lvl, "Huge file; ");
		if (TEST_BIT(flag, EXT4_EXTENTS_FL))
			pinfo_n(lvl, "Use extents; ");
		if (TEST_BIT(flag, EXT4_EA_INODE_FL))
			pinfo_n(lvl, "Large EA; ");
		if (TEST_BIT(flag, EXT4_EOFBLOCKS_FL))
			pinfo_n(lvl, "Blocks allocated beyond EOF; ");
		if (TEST_BIT(flag, EXT4_INLINE_DATA_FL))
			pinfo_n(lvl, "Inode has inline data; ");
		if (TEST_BIT(flag, EXT4_PROJINHERIT_FL))
			pinfo_n(lvl, "Create with parents projid; ");
		if (TEST_BIT(flag, EXT4_RESERVED_FL))
			pinfo_n(lvl, "Reserved for ext4 lib; ");
		if (flag == 0)
			pinfo_n(lvl, "none; ");
	}
	pinfo_n(lvl, "\n");
}

static void show_special_inode(int lvl, int inode_num)
{
	/*
        Inode号    用途
        0      不存在0号inode
        1      损坏数据块链表
        2      根目录
        3      User quota. 用户quota索引
        4      Group quota. 组quota索引
        5      Boot loader
        6      Undelete directory. 未删除的目录
        7      预留的块组描述符inode. (用于调整inode数目)
        8      日志inode
        9      The "exclude" inode, for snapshots(?)
        10     Replica inode, used for some non-upstream feature?
        11     第一个非预留的inode，通常是lost+found目录 当然也有可能不是，所以就不在这里列出了
    */
	if (inode_num > 10)
		return;

	switch (inode_num) {
	case 1:
		pinfo_n(lvl, "[inode]:Special Inode:Store damaged data linked list\n");
		break;
	case 2:
		pinfo_n(lvl, "[inode]:Special Inode:Root directory\n");
		break;
	case 3:
		pinfo_n(lvl, "[inode]:Special Inode:User quota\n");
		break;
	case 4:
		pinfo_n(lvl, "[inode]:Special Inode:Group quota\n");
		break;
	case 5:
		pinfo_n(lvl, "[inode]:Special Inode:Boot loader\n");
		break;
	case 6:
		pinfo_n(lvl, "[inode]:Special Inode:Undelete directory\n");
		break;
	case 7:
		pinfo_n(lvl, "[inode]:Special Inode:Reserved block group descriptor inode\n");
		break;
	case 8:
		pinfo_n(lvl, "[inode]:Special Inode:Journal inode\n");
		break;
	case 9:
		pinfo_n(lvl, "[inode]:Special Inode:The exclude inode, for snapshots\n");
		break;
	case 10:
		pinfo_n(lvl, "[inode]:Special Inode:Replica inode\n");
		break;
	// case 11:
	// 	pinfo_n(lvl, "[inode]:Special Inode:lost+found\n");
	// 	break;
	default:
		pinfo_n(lvl, "[inode]:Error Inode 0 not exist\n");
		break;
	}
}

static const char get_char(uint8_t dat)
{
	return (dat <= 32) ? '.' : dat;
}

static void put_10_data(int lvl, uint8_t *dat)
{
	/* 18b95711-55c7-4ee3-a1b3-67faa49dd297 */
	pinfo_n(lvl, "%02x %02x %02x %02x %02x %02x %02x %02x %02x %02x ", dat[0], dat[1], dat[2], dat[3], dat[4], dat[5], dat[6],
		dat[7], dat[8], dat[9]);
	pinfo_n(lvl, "| %c%c%c%c%c%c%c%c%c%c |\n", get_char(dat[0]), get_char(dat[1]), get_char(dat[2]), get_char(dat[3]),
		get_char(dat[4]), get_char(dat[5]), get_char(dat[6]), get_char(dat[7]), get_char(dat[8]), get_char(dat[9]));
}

static int ext4_extent_analysis(int lvl, int flag, struct ext4_inode *inode, struct ext4_tool *dat)
{
	union inode_flag i_flag;
	struct ext4_ext_path path;
	i_flag.inode_flag_dat = flag;
	int entries;
	/* 如果支持 EXT4_EXTENTS_FL 则用该种方法进行解析 */
	if (i_flag.extents) {
		/* 只解析这种方式的，旧版方式先不支持 */
		path.p_hdr = (struct ext4_extent_header *)inode->i_block;
		if (EXT4_EXT_MAGIC != path.p_hdr->eh_magic) {
			perr("inode eh_magic err!!! dat:0x%x", path.p_hdr->eh_magic);
			return -1;
		}
		path.p_depth = path.p_hdr->eh_depth;
		path.p_maxdepth = path.p_hdr->eh_max;
		dat->extent_num = path.p_hdr->eh_entries;
		if (path.p_depth == 0) {
			/* 后面跟着的都是叶子节点 */
			for (entries = 0; entries < path.p_hdr->eh_entries; entries++) {
				path.p_ext = EXT_NEXT_EXTENT(path.p_hdr, entries);
				memcpy(&dat->ex[entries], path.p_ext, sizeof(struct ext4_extent));
				pinfo_n(lvl, "[extent]:Logical block:%d\n", path.p_ext->ee_block);
				/* 范围覆盖的块数。如果此字段的值<=32768，则会初始化区段。
                如果该字段的值大于32768，则区段未初始化，实际区段长度为ee_len-32768。
                因此，初始化区段的最大长度为32768个块，未初始化区段的最大长度为32767个块。
                 */
				pinfo_n(lvl, "[extent]:number of blocks:%d\n", path.p_ext->ee_len);
				pinfo_n(lvl, "[extent]:physical block:%" PRIu64 "\n",
					(uint64_t)(((uint64_t)path.p_ext->ee_start_hi << 32) + path.p_ext->ee_start_lo));
			}
			path.p_ext = EXT_FIRST_EXTENT(path.p_hdr);
		} else {
			/* 后面跟的都是索引节点,只有大于512M的数据这样处理 */
			perr("File size greater than 512M !!!");
			return -1;
		}
	} else if (i_flag.inlinedata) {
		int i = 0;
		uint8_t *tbuf = NULL;

		/* 内联数据，数据直接保存在inode的bolck的60字节中 */
		/* 内联数据实际上最大能存的数量为：256 - 128 - i_extra_isize(32) - INLINE_DAT_ATTR(16) +60 = 140 */
		/* 前60字节数据必定在block中 */
		for (i = 0; i < 6; i++) {
			tbuf = (uint8_t *)inode->i_block;
			pinfo_n(lvl, "[extent]:[%03d] ", i * 10);
			put_10_data(lvl, tbuf + i * 10);
		}
		/* 如果inodelen=256 */
		if (dat->inode_size == INODE_DATA_LEN_256) {
			int magic;
			struct ext4_xattr_entry *xattr;
			char name[XATTR_SYSTEM_DATA_LEN + 1];
			uint8_t *pos;
			uint32_t len;
			uint8_t j;

			tbuf = (uint8_t *)(inode) + INODE_DATA_LEN_128 + inode->i_extra_isize;

			memcpy(&magic, tbuf, XATTR_MAGIC_LEN);
			if (EXT4_XATTR_MAGIC == magic) {
				xattr = (struct ext4_xattr_entry *)(tbuf + XATTR_MAGIC_LEN);
				memcpy(name, tbuf + INLINE_DAT_ATTR, XATTR_SYSTEM_DATA_LEN);
				if ((xattr->e_name_len == 4) &&
				    (strncmp(name, EXT4_XATTR_SYSTEM_DATA, XATTR_SYSTEM_DATA_LEN) == 0)) {
					/* 确保是真的数据 */
					pos = tbuf + XATTR_MAGIC_LEN + xattr->e_value_offs;
					len = xattr->e_value_size;
					do {
						pinfo_n(lvl, "[extent]:[%03d] ", i * 10);
						if (len >= 10) {
							put_10_data(lvl, pos);
							len -= 10;
							pos += 10;
						} else {
							for (j = 0; j < len; j++) {
								pinfo_n(lvl, "%02x ", pos[j]);
							}
							pinfo_n(lvl, "| ");
							for (j = 0; j < len; j++) {
								pinfo_n(lvl, "%c", get_char(pos[j]));
							}
							pinfo_n(lvl, " |\n");
							len = 0;
						}
						i++;
					} while (len > 0);
				}
			}
		}
	}

	return 0;
}

/*
 * helper functions to deal with 32 or 64bit block numbers.
 */
static size_t journal_tag_bytes(journal_superblock_t *journal)
{
	size_t sz;

	if (jbd2_has_feature_csum3(journal))
		return sizeof(journal_block_tag3_t);

	sz = sizeof(journal_block_tag_t);

	if (jbd2_has_feature_csum2(journal))
		sz += sizeof(uint16_t);

	if (jbd2_has_feature_64bit(journal))
		return sz;
	else
		return sz - sizeof(uint32_t);
}

static const char *get_journal_block(int type)
{
	switch (type) {
	case JBD2_DESCRIPTOR_BLOCK:
		return "DESCRIPTOR";
	case JBD2_COMMIT_BLOCK:
		return "COMMIT";
	case JBD2_SUPERBLOCK_V1:
		return "SUPERBLOCK_V1";
	case JBD2_SUPERBLOCK_V2:
		return "SUPERBLOCK_V2";
	case JBD2_REVOKE_BLOCK:
		return "REVOKE";
	default:
		return "ERR";
	}
}

static void put_journal_superblock_uuid(int lvl, uint8_t *uid)
{
	/* 18b95711-55c7-4ee3-a1b3-67faa49dd297 */
	pinfo_n(lvl, "[%s]:uuid:", JOURNAL_P_WORD);
	pinfo_n(lvl, "%x%x%x%x-%x%x-%x%x-%x%x-%x%x%x%x%x%x\n", uid[0], uid[1], uid[2], uid[3], uid[4], uid[5], uid[6], uid[7],
		uid[8], uid[9], uid[0], uid[11], uid[12], uid[13], uid[14], uid[15]);
}

static void put_jnl_incompat_flags(int lvl, int flag)
{
	pinfo_n(lvl, "[%s]:incompatible feature set: ", JOURNAL_P_WORD);
	{
		if (TEST_BIT(flag, JBD2_FEATURE_INCOMPAT_REVOKE))
			pinfo_n(lvl, "REVOKE; ");
		if (TEST_BIT(flag, JBD2_FEATURE_INCOMPAT_64BIT))
			pinfo_n(lvl, "64BIT; ");
		if (TEST_BIT(flag, JBD2_FEATURE_INCOMPAT_ASYNC_COMMIT))
			pinfo_n(lvl, "ASYNC_COMMIT; ");
		if (TEST_BIT(flag, JBD2_FEATURE_INCOMPAT_CSUM_V2))
			pinfo_n(lvl, "CSUM_V2; ");
		if (TEST_BIT(flag, JBD2_FEATURE_INCOMPAT_CSUM_V3))
			pinfo_n(lvl, "CSUM_V3; ");
		if (flag == 0)
			pinfo_n(lvl, "none; ");
	}
	pinfo_n(lvl, "\n");
}

static int journal_super_analysis(int lvl, journal_superblock_t *journal)
{
	conversion(journal->s_header.h_magic);
	if (journal->s_header.h_magic != JBD2_MAGIC_NUMBER) {
		perr("journal magic err.read magic=0x%x\n", journal->s_header.h_magic);
		return -1;
	}

	conversion(journal->s_header.h_blocktype);
	pinfo_n(lvl, "[%s]:blocktype:%s\n", JOURNAL_P_WORD, get_journal_block(journal->s_header.h_blocktype));

	conversion(journal->s_header.h_sequence);
	pinfo_n(lvl, "[%s]:sequence:%d\n", JOURNAL_P_WORD, journal->s_header.h_sequence);

	/* Static information describing the journal */
	conversion(journal->s_blocksize);
	pinfo_n(lvl, "[%s]:journal device blocksize:%d\n", JOURNAL_P_WORD, journal->s_blocksize);

	conversion(journal->s_maxlen);
	pinfo_n(lvl, "[%s]:total blocks in journal file:%d\n", JOURNAL_P_WORD, journal->s_maxlen);

	conversion(journal->s_first);
	pinfo_n(lvl, "[%s]:first block of log information:%d\n", JOURNAL_P_WORD, journal->s_first);

	/* Dynamic information describing the current state of the log */
	conversion(journal->s_sequence);
	pinfo_n(lvl, "[%s]:first commit ID expected in log:%d\n", JOURNAL_P_WORD, journal->s_sequence);

	conversion(journal->s_start);
	pinfo_n(lvl, "[%s]:blocknr of start of log:%d\n", JOURNAL_P_WORD, journal->s_start);

	/* Error value, as set by jbd2_journal_abort(). */
	conversion(journal->s_errno);
	pinfo_n(lvl, "[%s]:Error value:%d\n", JOURNAL_P_WORD, journal->s_errno);

	if (journal->s_header.h_blocktype == JBD2_SUPERBLOCK_V2) {
		/* 以下字段只在超级块v2中有效 */
		conversion(journal->s_feature_compat);
		pinfo_n(lvl, "[%s]:compatible feature set:%d\n", JOURNAL_P_WORD, journal->s_feature_compat);

		conversion(journal->s_feature_incompat);
		put_jnl_incompat_flags(lvl, journal->s_feature_incompat);

		conversion(journal->s_feature_ro_compat);
		pinfo_n(lvl, "[%s]:readonly-compatible feature set:%d\n", JOURNAL_P_WORD, journal->s_feature_ro_compat);

		put_journal_superblock_uuid(lvl, journal->s_uuid);

		conversion(journal->s_nr_users);
		pinfo_n(lvl, "[%s]:Nr of filesystems sharing log:%d\n", JOURNAL_P_WORD, journal->s_nr_users);

		conversion(journal->s_dynsuper);
		pinfo_n(lvl, "[%s]:Blocknr of dynamic superblock copy:%d\n", JOURNAL_P_WORD, journal->s_dynsuper);

		conversion(journal->s_max_transaction);
		pinfo_n(lvl, "[%s]:Limit of journal blocks per trans:%d\n", JOURNAL_P_WORD, journal->s_max_transaction);

		conversion(journal->s_max_trans_data);
		pinfo_n(lvl, "[%s]:Limit of data blocks per trans:%d\n", JOURNAL_P_WORD, journal->s_max_trans_data);

		pinfo_n(lvl, "[%s]:checksum type:%d\n", JOURNAL_P_WORD, journal->s_checksum_type);

		conversion(journal->s_checksum);
		pinfo_n(lvl, "[%s]:crc32c(superblock):%d\n", JOURNAL_P_WORD, journal->s_checksum);
	}
	return 0;
}

#if 0
static const char *get_jdb2_chktype(int type)
{
    switch (type) {
    case JBD2_CRC32_CHKSUM:
        return "CRC32";
    case JBD2_MD5_CHKSUM:
        return "MD5";
    case JBD2_SHA1_CHKSUM:
        return "SHA1";
    case JBD2_CRC32C_CHKSUM:
        return "CRC32C";
    default:
        return "NOCHK";
    }
}
#endif

static inline int tid_geq(__be32 x, __be32 y)
{
	int difference = (x - y);
	return (difference >= 0);
}

static int journal_data_analysis(int lvl, struct ext4_tool *dat, uint32_t offset)
{
	journal_superblock_t *journal = &dat->jnlsuper;
	uint8_t *buf = dat->extbuf;
	journal_header_t *head = (journal_header_t *)buf;
	journal_block_tag_t *tag = (journal_block_tag_t *)(buf + sizeof(journal_header_t));
	// journal_block_tag3_t *tag3 = (journal_block_tag3_t *)tag;
	int i = 0;
	int size = journal_tag_bytes(journal);
	time_t t;

	conversion(head->h_magic);
	if (head->h_magic != JBD2_MAGIC_NUMBER) {
		pinfo_n(lvl, "[error]:journal magic err.read magic=0x%x\n", head->h_magic);
		return -1;
	}

	if (jbd2_has_feature_csum3(journal)) {
		/* 先不考虑使用tag3的情况 */
		pinfo_n(lvl, "[error]:journal use journal_block_tag3_t not support\n");
		return -1;
	}

	conversion(head->h_blocktype);
	conversion(head->h_sequence);
	if (!tid_geq(head->h_sequence, journal->s_sequence)) {
		pinfo_n(lvl, "[info]:sequence err.read sequence=%d, journal_sequence=%d\n", head->h_sequence,
			journal->s_sequence);
		return -1;
	}
	pinfo_n(lvl, "\n========== pos:%" PRIu64 " SEQ:%-3d State:%-11s ==========\n", dat->loc_map_buf[offset], head->h_sequence,
		get_journal_block(head->h_blocktype));
	offset++;
	if (head->h_blocktype == JBD2_REVOKE_BLOCK || head->h_blocktype == JBD2_DESCRIPTOR_BLOCK) {
		if (JBD2_REVOKE_BLOCK == head->h_blocktype) {
			jbd2_journal_revoke_header_t *header = (jbd2_journal_revoke_header_t *)buf;
			uint32_t offset = sizeof(jbd2_journal_revoke_header_t);
			uint32_t rcount, max;
			int record_len = 4;
			uint32_t blocknr32;
			uint64_t blocknr64;

			conversion(header->r_count);
			rcount = header->r_count;
			max = rcount;

			if (jbd2_has_feature_64bit(journal))
				record_len = 8;

			while (offset + record_len <= max) {
				blocknr32 = 0;
				blocknr64 = 0;
				if (record_len == 4) {
					memcpy(&blocknr32, buf + offset, record_len);
					conversion(blocknr32);
					pinfo_n(lvl, "[%s]:revoke block:%d\n", JOURNAL_P_WORD, blocknr32);
				} else {
					memcpy(&blocknr64, buf + offset, record_len);
					conversion(blocknr64);
					pinfo_n(lvl, "[%s]:revoke block:%" PRIu64 "\n", JOURNAL_P_WORD, blocknr64);
				}
				offset += record_len;
			}

			return REVOKE_FLAG;
		}

		for (i = 1; i < MAX_TAG_NUM + 1; i++) {
			conversion(tag->t_blocknr);
			conversion(tag->t_flags);

			if (i == 1)
				put_journal_tag_uuid(lvl, buf + sizeof(journal_header_t));
			if (jbd2_has_feature_64bit(journal)) {
				conversion(tag->t_blocknr_high);
				pinfo_n(lvl, "[%s-64bit]:tag %d,block:%" PRIu64 ",flag:0x%x pos:%" PRIu64 "\n", JOURNAL_P_WORD, i,
					(uint64_t)(((uint64_t)tag->t_blocknr_high << 32) + tag->t_blocknr), tag->t_flags,
					dat->loc_map_buf[offset]);
			} else {
				pinfo_n(lvl, "[%s]:tag %d,block:%d,flag:0x%x pos:%" PRIu64 "\n", JOURNAL_P_WORD, i,
					tag->t_blocknr, tag->t_flags, dat->loc_map_buf[offset]);
			}
			offset++;
			if (TEST_BIT(tag->t_flags, JBD2_FLAG_LAST_TAG)) {
				/* 最后一个tag */
				break;
			}

			/* 跳过tag1后面的16字节uuid */
			tag = (journal_block_tag_t *)(buf + sizeof(journal_header_t) + 16 + i * size);
		}
		if (i >= MAX_TAG_NUM + 1) {
			perr("journal tag too much!!!");
			return -1;
		}

		return i;
	} else {
		perr("journal blocktype err.read blocktype:%s", get_journal_block(head->h_blocktype));
		if (head->h_blocktype == JBD2_COMMIT_BLOCK) {
			struct commit_header *comit = (struct commit_header *)buf;
			if (comit->h_magic != JBD2_MAGIC_NUMBER) {
				perr("comit magic err.read magic=0x%x\n", comit->h_magic);
				return -1;
			}
			t = (time_t)comit->h_commit_sec;
			pinfo_n(lvl, "[%s]:commit time: %s", JOURNAL_P_WORD, asctime(localtime(&t)));
			pinfo_n(lvl, "========== pos:%" PRIu64 " SEQ:%-3d State:%-11s  ==========\n", dat->loc_map_buf[offset],
				comit->h_sequence, get_journal_block(comit->h_blocktype));
			return COMMIT_FLAG;
		}
		return -1;
	}
	return 0;
}

static int journal_commit_analysis(int lvl, struct commit_header *comit, uint64_t pos)
{
	time_t t;
	conversion(comit->h_magic);
	if (comit->h_magic != JBD2_MAGIC_NUMBER) {
		perr("comit magic err.read magic=0x%x\n", comit->h_magic);
		return -1;
	}

	conversion(comit->h_blocktype);
	conversion(comit->h_sequence);
	conversion(comit->h_commit_sec);
	t = (time_t)comit->h_commit_sec;
	pinfo_n(lvl, "[%s]:commit time: %s", JOURNAL_P_WORD, asctime(localtime(&t)));
	pinfo_n(lvl, "========== pos:%" PRIu64 " SEQ:%-3d State:%-11s ==========\n", pos, comit->h_sequence,
		get_journal_block(comit->h_blocktype));

	return 0;
}

/**
 * 填充逻辑块到物理块的映射表
 * @param extents    输入的extent数组 (最多4个)
 * @param num_extents 实际有效的extent数量
 * @param loc        输出数组，loc[逻辑块号] = 物理块号
 * @param max_logical 最大逻辑块号 (避免越界)
 */
void build_block_map(struct ext4_extent *extents, int num_extents, uint64_t *loc, uint32_t max_logical)
{
	for (int i = 0; i < num_extents; i++) {
		struct ext4_extent *ext = &extents[i];
		uint32_t ee_block = ext->ee_block;
		uint16_t ee_len = ext->ee_len;
		uint64_t phys_start = ((uint64_t)(ext->ee_start_hi) << 32) | (ext->ee_start_lo);

		// 遍历该extent覆盖的所有逻辑块
		for (uint16_t j = 0; j < ee_len; j++) {
			uint32_t logical = ee_block + j;
			if (logical > max_logical)
				break; // 防越界

			loc[logical] = phys_start + j;
		}
	}
}

static int journal_analysis(int lvl, struct ext4_tool *dat)
{
	struct commit_header comit;
	uint64_t physical;
	int ret = 0, i;
	uint32_t offset = 0, block_num = 0;

	/* 这里我们解析日志先只考虑使用extent的方案，不考虑inode直接索引的方式 i_block[EXT4_N_BLOCKS]; 
	   因为也不太可能，毕竟日志还是挺大的
	*/
	if (dat->extent_num == 0) {
		pinfo_n(lvl, "[error]:extent_num err. num=%d\n", dat->extent_num);
		return -1;
	}

	/* 通过叶子节点，创建逻辑连续日志分区 */
	for (i = 0; i < dat->extent_num; i++) {
		block_num += dat->ex[i].ee_len;
	}
	dat->loc_map_buf = memalign(MEM_ALIGN_SIZE, block_num * sizeof(uint64_t));
	if (!dat->loc_map_buf) {
		perr("memalign failed. size=%u\n", block_num);
		return -1;
	}
	build_block_map(dat->ex, dat->extent_num, dat->loc_map_buf, block_num);

	/* 日志分析使用额外的 extbuf */
	physical = dat->loc_map_buf[0]; // 获取日志超级块，日志超级块必定在日志逻辑块分区的第一个块
	lseek64(dat->fd, (physical * dat->bsz), SEEK_SET);
	if (read(dat->fd, dat->extbuf, dat->bsz) != dat->bsz) {
		perr("read journal super err.");
		return -1;
	}

	/* 这里读出来的就是 journal 的超级块 */
	memcpy(&dat->jnlsuper, dat->extbuf, sizeof(journal_superblock_t));

	if (journal_super_analysis(lvl, &dat->jnlsuper)) {
		perr("journal super analysis err.");
		return -1;
	}

	if (dat->jnlsuper.s_start == 0) {
		perr("there is no journal.");
		return -1;
	}

	offset += dat->jnlsuper.s_start;
	physical = dat->loc_map_buf[offset];
	do {
		lseek64(dat->fd, (physical * dat->bsz), SEEK_SET);
		if (read(dat->fd, dat->extbuf, dat->bsz) != dat->bsz) {
			perr("read journal data err.");
			break;
		}

		/* ret返回tag的个数，有几个tag就代表有多少个数据块 */
		ret = journal_data_analysis(lvl, dat, offset);
		if (ret == REVOKE_FLAG || ret == COMMIT_FLAG) {
			goto end;
		} else if (ret <= 0) {
			pinfo_n(lvl, "[info]:tag num err!! num=%d,maybe journal end\n", ret);
			break;
		}

		/* 日志data显示 */
		if (TEST_BIT(outlvl, EXT4_LVL_JNL_DATA)) {
			for (i = 0; i < ret; i++) {
				pinfo_n(lvl, "========== pos:%" PRIu64 " ------ State:%-11s  ==========\n",
					dat->loc_map_buf[offset + 1 + i], "DATA_START");
				lseek64(dat->fd, (dat->loc_map_buf[offset + 1 + i] * dat->bsz), SEEK_SET);
				if (read(dat->fd, dat->jnl_data_buf, dat->bsz) != dat->bsz) {
					perr("read journal data err.");
					break;
				}

				show_data_all(dat->jnl_data_buf, dat->bsz);

				pinfo_n(lvl, "========== pos:%" PRIu64 " ------ State:%-11s  ==========\n",
					dat->loc_map_buf[offset + 1 + i], "DATA_END");
			}
		}

		/* 直接跳过日志的数据区块 */
		offset += (ret + 1);
		physical = dat->loc_map_buf[offset];
		lseek64(dat->fd, (physical * dat->bsz), SEEK_SET);
		if (read(dat->fd, dat->extbuf, dat->bsz) != dat->bsz) {
			perr("read journal commit err.");
			ret = -1;
			break;
		}
		memcpy(&comit, dat->extbuf, sizeof(struct commit_header));
		ret = journal_commit_analysis(lvl, &comit, dat->loc_map_buf[offset]);
		if (ret < 0) {
			break;
		}
end:
		offset += 1;
		physical = dat->loc_map_buf[offset];
	} while (offset < dat->jnlsuper.s_maxlen);

	return ret;
}

static const char *get_file_type(int type)
{
	switch (type) {
	case EXT4_FT_UNKNOWN:
		return "UNKNOWN";
	case EXT4_FT_REG_FILE:
		return "FILE";
	case EXT4_FT_DIR:
		return "DIR";
	case EXT4_FT_CHRDEV:
		return "CHRDEV";
	case EXT4_FT_BLKDEV:
		return "BLKDEV";
	case EXT4_FT_FIFO:
		return "FIFO";
	case EXT4_FT_SOCK:
		return "SOCK";
	case EXT4_FT_SYMLINK:
		return "SYMLINK";
	default:
		return "ERR";
	}
}

dir_entry_node_t *create_node(struct ext4_dir_entry_2 *entry)
{
	dir_entry_node_t *node = malloc(sizeof(dir_entry_node_t));
	if (!node)
		return NULL;

	node->inode = entry->inode;
	node->file_type = entry->file_type;

	// 复制文件名（注意：磁盘上的name无终止符）
	node->name = malloc(entry->name_len + 1);
	if (!node->name) {
		free(node);
		return NULL;
	}
	memcpy(node->name, entry->name, entry->name_len);
	node->name[entry->name_len] = '\0';
	node->next = NULL;

	return node;
}

/* 不允许有相同的inode号执行插入操作 */
void insert_sorted(dir_entry_node_t **head, dir_entry_node_t *new_node)
{
	if (!head || !new_node)
		return;

	// 提前检查头节点是否重复
	if (*head && (*head)->inode == new_node->inode) {
		free(new_node->name);
		free(new_node);
		return;
	}

	// 处理插入头部的特殊情况
	if (*head == NULL || new_node->inode < (*head)->inode) {
		new_node->next = *head;
		*head = new_node;
		return;
	}

	dir_entry_node_t *current = *head;
	while (current->next) {
		// 检查下一个节点是否重复
		if (current->next->inode == new_node->inode) {
			free(new_node->name);
			free(new_node);
			return;
		}

		// 找到插入位置
		if (current->next->inode > new_node->inode) {
			break;
		}
		current = current->next;
	}

	// 检查当前节点是否重复（处理插入末尾的情况）
	if (current->inode == new_node->inode) {
		free(new_node->name);
		free(new_node);
		return;
	}

	// 执行插入
	new_node->next = current->next;
	current->next = new_node;
}

void free_list(dir_entry_node_t *head)
{
	while (head) {
		dir_entry_node_t *temp = head;
		head = head->next;
		free(temp->name);
		free(temp);
	}
}

// 归并排序实现示例（时间复杂度O(n log n)）
dir_entry_node_t *merge_sort(dir_entry_node_t *head)
{
	if (!head || !head->next)
		return head;

	// 分割链表（快慢指针法）
	dir_entry_node_t *slow = head, *fast = head->next;
	while (fast && fast->next) {
		slow = slow->next;
		fast = fast->next->next;
	}
	dir_entry_node_t *mid = slow->next;
	slow->next = NULL;

	// 递归排序子链表
	dir_entry_node_t *left = merge_sort(head);
	dir_entry_node_t *right = merge_sort(mid);

	// 合并有序链表
	dir_entry_node_t dummy = { 0 };
	dir_entry_node_t *tail = &dummy;
	while (left && right) {
		if (left->inode <= right->inode) {
			tail->next = left;
			left = left->next;
		} else {
			tail->next = right;
			right = right->next;
		}
		tail = tail->next;
	}
	tail->next = left ? left : right;
	return dummy.next;
}

static int dir_analysis(int lvl, struct ext4_tool *dat, uint32_t inode_num)
{
	struct ext4_dir_entry_2 dir;
	uint64_t physical;
	int i = 0;
	uint32_t offset = 0, block_num = 0;
	dir_entry_node_t *node = NULL;

	if (dat->extent_num == 0) {
		pinfo_n(lvl, "[error]:extent_num err. num=%d\n", dat->extent_num);
		return -1;
	}

	/* 通过叶子节点，创建逻辑连续日志分区 */
	for (i = 0; i < dat->extent_num; i++) {
		block_num += dat->ex[i].ee_len;
	}
	dat->loc_map_buf = memalign(MEM_ALIGN_SIZE, block_num * sizeof(uint64_t));
	if (!dat->loc_map_buf) {
		perr("memalign failed. size=%u\n", block_num);
		return -1;
	}
	build_block_map(dat->ex, dat->extent_num, dat->loc_map_buf, block_num);
	for (i = 0; i < block_num; i++) {
		if (offset >= dat->bsz)
			offset -= dat->bsz;
		physical = dat->loc_map_buf[i]; // 获取根目录第i块
		lseek64(dat->fd, (physical * dat->bsz), SEEK_SET);
		if (read(dat->fd, dat->extbuf, dat->bsz) != dat->bsz) {
			perr("read root dir err.");
			return -1;
		}
		/* 读出来目录索引 */
		do {
			memcpy(&dir, dat->extbuf + offset, sizeof(struct ext4_dir_entry_2));
			if (dir.rec_len == 0 || dir.name_len == 0 || dir.file_type == EXT4_FT_UNKNOWN) {
				break;
			}

			offset += dir.rec_len;
			if (dir.inode != 2) {
				node = create_node(&dir);
				if (node) {
					node->root_dir_inode = inode_num;
					insert_sorted(&sorted_list, node);
				}
			}

		} while (offset <= dat->bsz);
	}
	sorted_list = merge_sort(sorted_list);
	/* 输出排序结果 */
	for (node = sorted_list; node; node = node->next) {
		if (node->root_dir_inode == inode_num)
			pinfo_n(lvl, "[info]:inode num=%d file_type:%s name:%s\n", node->inode, get_file_type(node->file_type),
				node->name);
	}
	return 0;
}

static int inode_analysis(int lvl, uint32_t inode_num, __le16 type, struct ext4_tool *dat)
{
	int ret = 0;
	if (inode_num == 2) {
		/* 根目录,可以根据实际使用物理块解析目录结构 */
		ret = dir_analysis(lvl, dat, inode_num);
	} else if (inode_num == 8) {
		/* 日志索引节点 */
		if (dat->f_compat.has_journal && lvl != 0)
			ret = journal_analysis(EXT4_LVL_JNL, dat);
	} else if (inode_num > 10 && TEST_BIT(type, EXT4_S_IFDIR)) {
		ret = dir_analysis(lvl, dat, inode_num);
	}
	return ret;
}

static void inde_show(int lvl, struct ext4_tool *dat, struct inode_show_info *info)
{
	dir_entry_node_t *node = NULL;
	time_t t;
	int group = info->group;
	struct ext4_inode *inode = info->inode;
	uint32_t inode_num = info->inode_num;
	int type = info->type;

	pinfo_n(lvl, "\n======= GP:%-3d Inode:%-5d State:%s =======\n", group, inode_num, get_inode_stat(type));
	show_special_inode(lvl, inode_num);
	for (node = sorted_list; node; node = node->next) {
		if (node->inode == inode_num) {
			pinfo_n(lvl, "[inode]:Name: %s\n", node->name);
			break;
		}
	}

	put_file_mode(lvl, inode->i_mode);
	put_ext4_file_flags(lvl, inode->i_flags);
	pinfo_n(lvl, "[inode]:Owner UID: 0x%x\n", (uint32_t)(((uint32_t)inode->osd2.linux2.l_i_uid_high << 16) + inode->i_uid));
	pinfo_n(lvl, "[inode]:Group Id: 0x%x\n", (uint32_t)((uint32_t)(inode->osd2.linux2.l_i_gid_high << 16) + inode->i_gid));
	pinfo_n(lvl, "[inode]:Size in bytes: %" PRIu64 "\n", (uint64_t)(((uint64_t)inode->i_size_high << 32) + inode->i_size_lo));
	if (dat->inode_size == INODE_DATA_LEN_256) {
		t = (time_t)inode->i_crtime;
		pinfo_n(lvl, "[inode]:Creation time: %s", asctime(localtime(&t)));
	}

	t = (time_t)inode->i_atime;
	pinfo_n(lvl, "[inode]:Access time: %s", asctime(localtime(&t)));
	t = (time_t)inode->i_ctime;
	pinfo_n(lvl, "[inode]:Inode Change time: %s", asctime(localtime(&t)));
	t = (time_t)inode->i_mtime;
	pinfo_n(lvl, "[inode]:Modification time: %s", asctime(localtime(&t)));
	t = (time_t)inode->i_dtime;
	pinfo_n(lvl, "[inode]:Deletion Time: %s", asctime(localtime(&t)));

	/* 通常，ext4不允许inode有超过65000个硬链接,这适用于文件和目录，这意味着一个目录中不能有超过64998个子目录
                 */
	/* 每个子目录的“..”项和目录本身的“.”项都算作硬链接 */
	/* 启用DIR_NLINK功能后，ext4通过将此字段设置为1来表示硬链接的数量未知，从而支持64998多个子目录。 */
	pinfo_n(lvl, "[inode]:Hard link count: %d\n", inode->i_links_count);
	pinfo_n(lvl, "[inode]:Blocks count: %" PRIu64 "\n",
		(uint64_t)(((uint64_t)inode->osd2.linux2.l_i_blocks_high << 32) + inode->i_blocks_lo));

	pinfo_n(lvl, "[inode]:File version: %u\n", inode->i_generation);
	/* 扩展属性块 */
	pinfo_n(lvl, "[inode]:Extended attribute block: %" PRIu64 "\n",
		(uint64_t)(((uint64_t)inode->osd2.linux2.l_i_file_acl_high << 32) + inode->i_file_acl_lo));

	pinfo_n(lvl, "[inode]:(Obsolete) fragment address: %d\n", inode->i_obso_faddr);
	put_ext4_inode_pointers(lvl, inode->i_block);
	if (dat->inode_size == INODE_DATA_LEN_128) {
		/* dat->f_incompat.ea_inode */
		/* Inode版本。但是，如果设置了EA_INODE
                     * INODE标志，则该INODE存储扩展属性值，该字段包含属性值引用计数的上32位。 */
		pinfo_n(lvl, "[inode]:%s: %d\n", dat->f_incompat.ea_inode ? "extended attribute value" : "Inode version",
			inode->osd1.linux1.l_i_version);

		pinfo_n(lvl, "[inode]:Inode checksum: %d\n", inode->osd2.linux2.l_i_checksum_lo);
	} else {
		pinfo_n(lvl, "[inode]:%s: %" PRIu64 "\n", dat->f_incompat.ea_inode ? "extended attribute value" : "Inode version",
			(uint64_t)(((uint64_t)inode->i_version_hi << 32) + inode->osd1.linux1.l_i_version));

		pinfo_n(lvl, "[inode]:Inode checksum: %" PRIu64 "\n",
			(uint64_t)(((uint64_t)inode->i_checksum_hi << 32) + inode->osd2.linux2.l_i_checksum_lo));

		/* 0x80 */
		pinfo_n(lvl, "[inode]:extra_isize: %d\n", inode->i_extra_isize);
		pinfo_n(lvl, "[inode]:Project ID: 0x%x\n", inode->i_projid);
	}
}

static int ext4_inode_table_analysis(int group, uint64_t block_num, struct ext4_tool *dat)
{
	/* inode表解析 */
	int i = 0, type = 0, cnt = 0;
	struct ext4_inode *inode;
	int bolck_per_inode = dat->per_inode / dat->inodeblknum;
	int pos;
	int lvl = EXT4_LVL_INODE;
	uint32_t inode_num = 0;
	struct inode_show_info info;
	int show_inode_num = -1;

	for (cnt = 0; cnt < dat->inodeblknum; cnt++) {
		lseek64(dat->fd, ((block_num + cnt) * dat->bsz), SEEK_SET);
		if (read(dat->fd, dat->buf, dat->bsz) != dat->bsz) {
			perr("read data err.\n");
			return -1;
		}

		for (i = 0; i < bolck_per_inode; i++) {
			pos = i + cnt * bolck_per_inode;
			if (TEST_BIT(dat->inode_bit_buf[pos], B_ATTR_DATA_USE)) {
				/* 该 inode 正被使用 */
				type = EXT4_INODE_USE;
			} else if (TEST_BIT(dat->inode_bit_buf[pos], B_ATTR_DATA_NOT_USE)) {
				/* 初始化了，但是该 inode 没用 */
				type = EXT4_INODE_UNUSE;
			} else {
				/* 能走到这里说明该 inode 表可能没初始化 */
				type = EXT4_INODE_NOT_INIT;
			}

			if (!TEST_BIT(dat->p_inode, P_INODE_STATE)) {
				if (dat->p_inode) {
					if (show_inode_num == -1)
						show_inode_num = dat->p_inode;
					dat->p_inode--;
					/* 如果需求打印的inode还是大于0，则打印其他inode信息 */
					if ((type == EXT4_INODE_UNUSE) || (type == EXT4_INODE_NOT_INIT))
						goto showinode;
				} else {
					return 0;
				}
			}

			if (type == EXT4_INODE_USE) {
showinode:
				/* 先只打印使用了的inode信息 */
				inode_num = group * dat->per_inode + pos + 1;
				inode = (struct ext4_inode *)(dat->buf + i * dat->inode_size);

				if (inode_num == show_inode_num || show_inode_num == -1) {
					info.group = group;
					info.inode = inode;
					info.inode_num = inode_num;
					info.type = type;
					lvl = EXT4_LVL_INODE;
					inde_show(lvl, dat, &info);
				} else {
					lvl = 0;
				}

				ext4_extent_analysis(lvl, inode->i_flags, inode, dat);
				inode_analysis(lvl, inode_num, inode->i_mode, dat);
			}
		}
	}

	return 0;
}

static int ext4_group_desc_analysis(struct ext4_tool *dat)
{
	/* 组块描述符分析 */
	uint8_t havesuper = 0, flex_group = 0;
	uint32_t i = 0, bgnum = 0;
	uint64_t tmp;
	uint64_t pow_3 = 3, pow_5 = 5, pow_7 = 7;
	uint64_t start_block_bitmap = 0;
	uint64_t start_inode_bitmap = 0;
	uint64_t start_inode_table = 0;

	uint32_t bolck_bitmap_pos = 0;
	uint32_t inode_bitmap_pos = 0;
	uint32_t inode_table_pos = 0;

	uint32_t free_blocks_count = 0;
	uint32_t free_inodes_count = 0;
	uint32_t used_dirs_count = 0;
	uint32_t itable_unused_count = 0;

	struct ext4_group_desc *desc = NULL;
	uint64_t t_block_num = 0;
	uint32_t cur_group_desc_num = dat->group_desc_blk_num;

	int lvl = EXT4_LVL_GDT;

	if (dat->f_incompat.meta_bg) {
		/* 开启元组特性后，一个元组内只包含此元组的组块描述符，只占用1block */
		cur_group_desc_num = 1;
	}

	for (i = 0; i < dat->groupnum; i++) {
		memset(dat->block_bit_buf, 0, dat->per_block);
		memset(dat->inode_bit_buf, 0, dat->per_inode);
		ext4_pos_zero(dat);

		havesuper = 0;

		desc = (struct ext4_group_desc *)(dat->group_desc_buf + i * dat->group_desc_len);

		t_block_num = (uint64_t)(i * dat->per_block);
		if (dat->bsz == 1024)
			t_block_num += 1;

		pinfo_n(lvl, "\nGroup %d:(Blocks %" PRIu64 "-%" PRIu64 ")\n", i, t_block_num,
			((t_block_num + dat->per_block - 1) > dat->bolcknum) ? (dat->bolcknum - 1) :
									       (t_block_num + dat->per_block - 1));

		if (dat->f_rocompat.sparse_super) {
			/* 开启了稀疏超级块属性的话，超级块的备份只出现在0,3,5,7次幂的块数组中 */
			if (i == 0 || i == 1) {
				havesuper = 1;
			} else if (i == pow_3) {
				pow_3 *= 3;
				havesuper = 1;
			} else if (i == pow_5) {
				pow_5 *= 5;
				havesuper = 1;
			} else if (i == pow_7) {
				pow_7 *= 7;
				havesuper = 1;
			}
		} else {
			havesuper = 1;
		}

		if (havesuper)
			ext4_bitmap_set(BLOCK_BITMAP, B_ATTR_SB, dat);

		if (dat->f_incompat.meta_bg) {
			/* 开启了元组特性后,组描述符只存在于第0，1，以及最后一块中 */
			if (((i % dat->mate_bg_num) == 0) || ((i % dat->mate_bg_num) == 1) ||
			    ((i % dat->mate_bg_num) == (dat->mate_bg_num - 1))) {
				if (havesuper)
					pinfo_n(lvl, "    %s at %" PRIu64 "",
						(i == 0) ? "Primary superblock" : "Backup superblock", t_block_num);
				pinfo_n(lvl, "%sGroup descriptors at %" PRIu64 "\n", havesuper ? ", " : "    ",
					havesuper ? (1 + t_block_num) : t_block_num);

				ext4_bitmap_set(BLOCK_BITMAP, B_ATTR_GDT, dat);
			} else {
				if (havesuper)
					pinfo_n(lvl, "    %s at %" PRIu64 "\n",
						(i == 0) ? "Primary superblock" : "Backup superblock", t_block_num);
			}
		} else {
			if (havesuper) {
				pinfo_n(lvl, "    %s at %" PRIu64 ", Group descriptors at %" PRIu64 "-%" PRIu64 "\n",
					(i == 0) ? "Primary superblock" : "Backup superblock", t_block_num, (1 + t_block_num),
					(t_block_num + cur_group_desc_num));
				/* 需要包含到上限 故<= */
				for (tmp = 1 + t_block_num; tmp <= t_block_num + cur_group_desc_num; tmp++) {
					ext4_bitmap_set(BLOCK_BITMAP, B_ATTR_GDT, dat);
				}
			}
		}

		if (dat->f_compat.resize_inode && havesuper) {
			pinfo_n(lvl, "    Reserved GDT blocks at %" PRIu64 "-%" PRIu64 "\n",
				(uint64_t)(t_block_num + cur_group_desc_num + 1),
				(uint64_t)(t_block_num + cur_group_desc_num + dat->reserved_gdt_blocks));
			/* 需要包含到上限 故<= */
			for (tmp = (t_block_num + cur_group_desc_num + 1);
			     tmp <= (t_block_num + cur_group_desc_num + dat->reserved_gdt_blocks); tmp++) {
				ext4_bitmap_set(BLOCK_BITMAP, B_ATTR_RES_GDT, dat);
			}
		}

		if (dat->f_incompat.use_64bit) {
			start_block_bitmap = desc->bg_block_bitmap_lo + ((uint64_t)desc->bg_block_bitmap_hi << 32);
			start_inode_bitmap = desc->bg_inode_bitmap_lo + ((uint64_t)desc->bg_inode_bitmap_hi << 32);
			start_inode_table = desc->bg_inode_table_lo + ((uint64_t)desc->bg_inode_table_hi << 32);
			free_blocks_count = desc->bg_free_blocks_count_lo + ((uint32_t)desc->bg_free_blocks_count_hi << 16);
			free_inodes_count = desc->bg_free_inodes_count_lo + ((uint32_t)desc->bg_free_inodes_count_hi << 16);
			used_dirs_count = desc->bg_used_dirs_count_lo + ((uint32_t)desc->bg_used_dirs_count_hi << 16);
			itable_unused_count = desc->bg_itable_unused_lo + ((uint32_t)desc->bg_itable_unused_hi << 16);
		} else {
			start_block_bitmap = desc->bg_block_bitmap_lo;
			start_inode_bitmap = desc->bg_inode_bitmap_lo;
			start_inode_table = desc->bg_inode_table_lo;
			free_blocks_count = desc->bg_free_blocks_count_lo;
			free_inodes_count = desc->bg_free_inodes_count_lo;
			used_dirs_count = desc->bg_used_dirs_count_lo;
			itable_unused_count = desc->bg_itable_unused_lo;
		}

		flex_group = 0;
		bolck_bitmap_pos++;
		inode_bitmap_pos++;
		inode_table_pos += dat->inodeblknum;

		/* 计算 block_bitmap , inode_bitmap , inode_table 所在块组的块位置 */
		if (dat->f_incompat.flex_bg) {
			if (dat->flex_bg_num) {
				if (i % dat->flex_bg_num == 0) {
					flex_group = 1;
					bgnum = i;
					/* 如果开启了柔性块组的话，多个块组组合成一个大块,大块中的第一组块存放其他块的BB,IB,IT */
					for (tmp = 0; tmp < dat->flex_bg_num; tmp++) {
						ext4_bitmap_set(BLOCK_BITMAP, B_ATTR_BB, dat);
					}
					for (tmp = 0; tmp < dat->flex_bg_num; tmp++) {
						ext4_bitmap_set(BLOCK_BITMAP, B_ATTR_IB, dat);
					}
					for (tmp = 0; tmp < dat->flex_bg_num * dat->inodeblknum; tmp++) {
						ext4_bitmap_set(BLOCK_BITMAP, B_ATTR_IT, dat);
					}
					goto countpos;
				}
			} else {
				dat->f_incompat.flex_bg = 0;
				goto countpos;
			}
		} else {
			ext4_bitmap_set(BLOCK_BITMAP, B_ATTR_BB, dat);
			ext4_bitmap_set(BLOCK_BITMAP, B_ATTR_IB, dat);
			for (tmp = 0; tmp < dat->inodeblknum; tmp++) {
				ext4_bitmap_set(BLOCK_BITMAP, B_ATTR_IT, dat);
			}

countpos:
			/* 非柔性每次都要算 */
			if ((start_block_bitmap < t_block_num) || (start_inode_bitmap < start_block_bitmap) ||
			    (start_inode_table < start_inode_bitmap)) {
				/* 出现这种情况绝对是数据错误了 */
				perr("data err!!! start_block_bitmap:%" PRIu64 ", start_inode_bitmap:%" PRIu64
				     ", start_inode_table:%" PRIu64 ", t_block_num:%" PRIu64 " \n",
				     start_block_bitmap, start_inode_bitmap, start_inode_table, t_block_num);
				return -1;
			}
			/* bolck_bitmap_pos 这里如果出现超级大的文件系统(2^32*4k)可能出现问题，先不考虑 */
			bolck_bitmap_pos = (uint32_t)(start_block_bitmap - t_block_num);
			inode_bitmap_pos = (uint32_t)(start_inode_bitmap - t_block_num);
			inode_table_pos = (uint32_t)(start_inode_table - t_block_num);
		}

		if ((flex_group) || (!dat->f_incompat.flex_bg)) {
			/* Block bitmap at 524288 (+0) */
			pinfo_n(lvl, "    Block bitmap at %" PRIu64 "(+%d)\n", start_block_bitmap, bolck_bitmap_pos);
			/* Inode bitmap at 1041 (+1041) */
			pinfo_n(lvl, "    Inode bitmap at %" PRIu64 "(+%d)\n", start_inode_bitmap, inode_bitmap_pos);
			/* Inode table at 1057-1059 (+1057) */
			pinfo_n(lvl, "    Inode table at %" PRIu64 "-%" PRIu64 " (+%d)\n", start_inode_table,
				start_inode_table + (uint64_t)dat->inodeblknum - 1, inode_table_pos);
		} else {
			/* Block bitmap at 1026 (bg #0 + 1026) */
			pinfo_n(lvl, "    Block bitmap at %" PRIu64 "(bg #%d + %d)\n", start_block_bitmap, bgnum,
				bolck_bitmap_pos);
			/* Inode bitmap at 1042 (bg #0 + 1042) */
			pinfo_n(lvl, "    Inode bitmap at %" PRIu64 "(bg #%d + %d)\n", start_inode_bitmap, bgnum,
				inode_bitmap_pos);
			/* Inode table at 1060-1062 (bg #0 + 1060) */
			pinfo_n(lvl, "    Inode table at %" PRIu64 "-%" PRIu64 "(bg #%d + %d)\n", start_inode_table,
				start_inode_table + (uint64_t)dat->inodeblknum - 1, bgnum, inode_table_pos);
		}

		/* 31657 free blocks, 0 free inodes, 2 directories */
		pinfo_n(lvl, "    %d free blocks, %d free inodes, %d directories\n", free_blocks_count, free_inodes_count,
			used_dirs_count);
		pinfo_n(lvl, "    Itable unused inodes count:%d\n", itable_unused_count);

		put_ext4_EXT4_BG_flags(lvl, desc->bg_flags);
		ext4_pos_zero(dat);
		ext4_bitmap_analysis(lvl, i, start_block_bitmap, desc->bg_flags, BLOCK_BITMAP, dat);
		ext4_bitmap_analysis(lvl, i, start_inode_bitmap, desc->bg_flags, INODE_BITMAP, dat);
		if (dat->f_rocompat.gdt_csum) {
			pinfo_n(lvl, "    csum:0x%x\n", desc->bg_checksum);
		}
		if (TEST_BIT(outlvl, EXT4_LVL_BLOCK_MAP))
			ext4_show_block_use(EXT4_LVL_BLOCK_MAP, i, dat, t_block_num);
		if (dat->p_inode)
			ext4_inode_table_analysis(i, start_inode_table, dat);
	}
	return 0;
}

static int ext4_group_analysis(int group, struct ext4_tool *dat)
{
	int ret = 0, i = 0;
	uint64_t t_block_num = 0; /* 确定下一个的块位置 */
	int lvl = EXT4_LVL_SUPER;

	pimp("analysis group:%d\n", group);

	t_block_num = (uint64_t)(dat->per_block * group);
	if (dat->bsz == 1024)
		t_block_num += 1;

	lseek64(dat->fd, (t_block_num * dat->bsz), SEEK_SET);
	if (read(dat->fd, dat->buf, dat->bsz) != dat->bsz) {
		perr("read data err.\n");
		return -1;
	}

	if (dat->bsz == 1024) {
		memcpy(dat->sb, dat->buf, EXT4_SB_LEN);
	} else {
		if (group == 0) {
			/* 只在第0块组需要偏移 EXT4_SB_OFFSET */
			memcpy(dat->sb, dat->buf + EXT4_SB_OFFSET, EXT4_SB_LEN);
		} else {
			memcpy(dat->sb, dat->buf, EXT4_SB_LEN);
		}
	}

	if ((ret = ext4_super_analysis(dat)) != 0)
		return ret;

	if (dat->ext_mode == DUMP_MODE_EXT) {
		lvl = EXT4_LVL_DATA;
		pinfo_n(lvl, "========== pos:%d ------ State:%-11s  ==========\n", dat->dump_bolcknum, "DATA_START");
		lseek64(dat->fd, (dat->dump_bolcknum * dat->bsz), SEEK_SET);
		if (read(dat->fd, dat->buf, dat->bsz) != dat->bsz) {
			perr("read data err.");
			return -1;
		}
		show_data_all(dat->buf, dat->bsz);
		pinfo_n(lvl, "========== pos:%d ------ State:%-11s  ==========\n", dat->dump_bolcknum, "DATA_END");
		return 0;
	}

	if (dat->inode_bit_buf == NULL)
		dat->inode_bit_buf = memalign(MEM_ALIGN_SIZE, dat->per_inode);
	if (dat->inode_bit_buf == NULL) {
		perr("memalign inode_bit_buf err mabey ram is not enough");
		return -1;
	}

	GET_NUN(dat->per_inode * dat->inode_size, dat->bsz, dat->inodeblknum);
	pinfo(lvl, "%d\n", "Inode blocks per group:", dat->inodeblknum);

	GET_NUN(dat->bolcknum, dat->per_block, dat->groupnum);
	pinfo(lvl, "%d\n", "group num:", dat->groupnum);

	if (dat->f_incompat.meta_bg) {
		/* 元组块特性的话，每一个元组块集合，组块描述符只能有一个 */
		/* 计算出具有几个元组合集,块大小/组块描述符大小,一定是整除的 */
		dat->mate_bg_num = dat->bsz / dat->group_desc_len;
		/* 在计算需要保存的组块描述符的块数 */
		GET_NUN(dat->groupnum, dat->mate_bg_num, dat->group_desc_blk_num);
	} else {
		GET_NUN(dat->group_desc_len * dat->groupnum, dat->bsz, dat->group_desc_blk_num);
		pinfo(lvl, "%d\n", "group desc block num:", dat->group_desc_blk_num);
	}

	if (dat->group_desc_buf == NULL)
		dat->group_desc_buf = memalign(MEM_ALIGN_SIZE, dat->bsz * dat->group_desc_blk_num);
	if (dat->group_desc_buf == NULL) {
		perr("memalign group_desc_buf memory err mabey ram is not enough");
		return -1;
	}

	if (dat->f_incompat.meta_bg) {
		/* 定位保存组块描述符的位置,第0块的直接读取就可以 */
		if (read(dat->fd, dat->buf, dat->bsz) != dat->bsz) {
			perr("read Group Descriptors err.\n");
			return -1;
		}
		memcpy(dat->group_desc_buf, dat->buf, dat->bsz);
		/* 关键是后面的组块描述符 */
		for (i = 1; i < dat->group_desc_blk_num; i++) {
			if (dat->f_rocompat.sparse_super) {
				/* 如果开启了稀疏超级块的话，3，5，7的幂是除了第0，1块组是不可能和其他的具有组块描述符的块重合的 */
				t_block_num = (uint64_t)(dat->mate_bg_num * i) * dat->per_block;
			} else {
				/* 如果没开稀疏超级快，则每个组块都会有超级块，则跳过超级块 */
				t_block_num = (uint64_t)(dat->mate_bg_num * i) * dat->per_block + 1;
			}

			/* 如果是1024大小的话,第一块必须跳过,所以要+1 */
			if (dat->bsz == 1024)
				t_block_num += 1;

			lseek64(dat->fd, (t_block_num * dat->bsz), SEEK_SET);
			if (read(dat->fd, dat->buf, dat->bsz) != dat->bsz) {
				perr("read Group Descriptors err.\n");
				return -1;
			}
			memcpy(dat->group_desc_buf + i * dat->bsz, dat->buf, dat->bsz);
		}
	} else {
		/* 在读取一块数据,根据dat->group_desc_blk_num，可以读取出全部组块描述符 */
		for (i = 0; i < dat->group_desc_blk_num; i++) {
			if (read(dat->fd, dat->buf, dat->bsz) != dat->bsz) {
				perr("read Group Descriptors err.\n");
				return -1;
			}
			memcpy(dat->group_desc_buf + i * dat->bsz, dat->buf, dat->bsz);
		}
	}

	ret = ext4_group_desc_analysis(dat);

	/* 最后无论如何先归位 */
	lseek64(dat->fd, 0L, SEEK_SET);
	return ret;
}

static int ext4_tool_prepare(struct ext4_tool *dat)
{
	/* 对当前mtd设备只读打开 */
	dat->fd = open(dat->path, O_LARGEFILE | O_RDONLY | O_DIRECT);
	if (dat->fd < 0) {
		perr("open %s error!", dat->path);
		return -1;
	}

	dat->sb = memalign(MEM_ALIGN_SIZE, EXT4_SB_LEN);
	if (dat->sb == NULL) {
		perr("memalign ext4_super_block memory err mabey ram is not enough");
		return -1;
	}

	if (FILE_MODE == dat->mode) {
		dat->ssz = 512; /* 写死512大小的扇区 */
		dat->bsz = dat->setbsz;
		dat->bytes = lseek64(dat->fd, 0L, SEEK_END);
		lseek64(dat->fd, 0L, SEEK_SET);
	} else if (SPUER_MODE == dat->mode) {
		pimp("spuer mode");
	} else {
		if ((ioctl(dat->fd, BLKSSZGET, &dat->ssz) != 0) || (ioctl(dat->fd, BLKBSZGET, &dat->bsz) != 0) ||
		    (ioctl(dat->fd, BLKGETSIZE64, &dat->bytes) != 0)) {
			perr("read device err!!");
			return -1;
		}
	}

	pimp("sector size:%d\n", dat->ssz);
	pimp("block size:%d\n", dat->bsz);
	pimp("total size:%lldbytes,%dG\n", dat->bytes, (int)(dat->bytes >> 30));
	/* 块大小明显有问题 */
	if (dat->bsz > READ_MAX_BUF_LEN)
		return -1;
	/* 计算单个块组具有的最大数据块个数=块大小*8bit，根据位图来算的 */
	dat->per_block = dat->bsz * 8;

	dat->buf = memalign(MEM_ALIGN_SIZE, dat->bsz);
	if (dat->buf == NULL) {
		perr("memalign readbuf err mabey ram is not enough");
		return -1;
	}

	dat->block_bit_buf = memalign(MEM_ALIGN_SIZE, dat->per_block);
	if (dat->buf == NULL) {
		perr("memalign block_bit_buf err mabey ram is not enough");
		return -1;
	}

	dat->extbuf = memalign(MEM_ALIGN_SIZE, dat->bsz);
	if (dat->extbuf == NULL) {
		perr("memalign extbuf err mabey ram is not enough");
		return -1;
	}

	dat->jnl_data_buf = memalign(MEM_ALIGN_SIZE, dat->bsz);
	if (dat->jnl_data_buf == NULL) {
		perr("memalign jnl_data_buf err mabey ram is not enough");
		return -1;
	}

	return 0;
}

static void ext4_tool_close(struct ext4_tool *dat)
{
	if (dat->sb)
		free(dat->sb);
	if (dat->buf)
		free(dat->buf);
	if (dat->extbuf)
		free(dat->extbuf);
	if (dat->jnl_data_buf)
		free(dat->jnl_data_buf);
	if (dat->group_desc_buf)
		free(dat->group_desc_buf);
	if (dat->block_bit_buf)
		free(dat->block_bit_buf);
	if (dat->inode_bit_buf)
		free(dat->inode_bit_buf);
	if (dat->loc_map_buf)
		free(dat->loc_map_buf);
	free_list(sorted_list);
	close(dat->fd);
}

static void print_directory_tree(dir_entry_node_t *root, int depth, struct ext4_tool *dat)
{
	char indent[512] = { 0 };
	int i;

	// 创建缩进字符串
	for (i = 0; i < depth; i++) {
		if (i == depth - 1) {
			strcat(indent, "\033[34m |-->\033[0m "); // 蓝色箭头
		} else {
			strcat(indent, "\033[34m |    \033[0m"); // 蓝色竖线
		}
	}

	// 只在第一次调用时打印root:
	if (depth == 0) {
		printf("\n\033[1;33mroot:\033[0m\n"); // 黄色加粗root
	}

	for (dir_entry_node_t *node = root; node != NULL; node = node->next) {
		if (node->root_dir_inode == root->root_dir_inode) {
			const char *color = node->file_type == EXT4_FT_DIR ? "\033[1;32m" : "\033[1;36m";
			// 打印当前条目
			printf("%s%s%s: \033[1;37m%s\033[0m (inode:%d)\n", (depth == 0) ? "" : indent, color,
			       get_file_type(node->file_type), node->name, node->inode);
			// 如果是目录，递归打印其内容
			if (node->file_type == EXT4_FT_DIR && node->inode != 2) { // 跳过根目录自身
				// 查找该目录下的所有条目
				dir_entry_node_t *subdir = NULL;
				for (dir_entry_node_t *n = sorted_list; n != NULL; n = n->next) {
					if (n->root_dir_inode == node->inode) {
						// 添加到子目录列表
						dir_entry_node_t *new_node = malloc(sizeof(dir_entry_node_t));
						memcpy(new_node, n, sizeof(dir_entry_node_t));
						new_node->next = subdir;
						subdir = new_node;
					}
				}

				// 递归打印子目录
				if (subdir != NULL) {
					print_directory_tree(subdir, depth + 1, dat);

					// 释放临时子目录列表
					while (subdir != NULL) {
						dir_entry_node_t *tmp = subdir;
						subdir = subdir->next;
						free(tmp);
					}
				}
			}
		}
	}
}

// 定义长选项命令用到的长选项表
static struct option g_long_options[] = { { "super", no_argument, NULL, 's' }, // 只显示超级块信息
					  { "map", no_argument, NULL, 'm' }, //显示block_map，该功能与其他打印互斥
					  { "inode", no_argument, NULL, 'i' }, //显示inode信息(只显示已使用inode)
					  { "INODE", required_argument, NULL, 'I' }, //显示inode信息(根据参数打印部分)
					  { "journal", no_argument, NULL, 'j' }, //显示日志信息
					  { "journal_all", no_argument, NULL, 'J' }, //显示日志信息+日志数据
					  { "gdt", no_argument, NULL, 'g' }, //显示组描述符信息
					  { "dev_name", required_argument, NULL, 'd' }, //要查看的设备
					  { "dump", required_argument, NULL, 'D' }, //打印出指定块的数据情况
					  { "block_size", required_argument, NULL,
					    'b' }, //设置要读取的文件的块大小(只用于读取文件时)
					  { "file", required_argument, NULL, 'f' }, //要读取的文件
					  { "all", no_argument, NULL, 'a' }, //帮助信息
					  { "tree", no_argument, NULL, 'l' }, // 显示树状目录结构
					  { "help", no_argument, NULL, 'h' }, //帮助信息
					  { 0, 0, 0, 0 } };

const char g_opt_string[] = "asmgijJhd:I:f:b:D:l"; // 合法的选项字符集

int help_print(void)
{
	printf("使用方法: ./ext_tool [options]\n");
	printf("说明: 该程序主要用ext4数据分析,可用选项如下:\n");
	printf("\t -d or --dev_name 需要查看的块设备(含路径,default:/dev/part/emmc)\n");
	printf("\t -a or --all      打印出全部数据(超级块+组块+位图情况+日志+inode)\n");
	printf("\t -D or --dump     打印出指定块的数据情况\n");
	printf("\t -s or --super    只打印超级块信息\n");
	printf("\t -m or --map      打印block_map信息\n");
	printf("\t -i or --inode    打印已使用inode信息\n");
	printf("\t -I or --INODE    显示指定inode号信息\n");
	printf("\t -j or --journal  打印jdb2日志信息\n");
	printf("\t -J or --journal_all  打印jdb2日志信息包括日志数据块\n");
	printf("\t -g or --gdt      打印组描述符信息\n");
	printf("\t -f or --file     打开ext4镜像文件，并对该镜像文件进行解析\n");
	printf("\t -b or --block    只有在 -f(文件模式) 下生效设置文件的块大小,默认4096\n");
	printf("\t -l or --tree     显示文件系统的树状目录结构\n");
	printf("\t -h or --help     查看帮助信息\n");
	printf("\t 例：./ext4tool -m -d /dev/part/emmc\n");
	printf("\t\n");
	return 0;
}

static int cmd_analysis(struct ext4_tool *dat, int argc, char *argv[])
{
	int ret = 0, tmp_len = 0;
	char *tmp_ptr = NULL;
	uint8_t mutexflag = 0;
	uint8_t devmutex = 0; //-d和-f选项必须互斥
	int option_index = 0;
	char *endptr;
	long num;
	/* 默认打印超级块和组描述符信息 */
	outlvl = 0;
	// 循环处理输入的长选项命令行
	while ((ret = getopt_long(argc, argv, g_opt_string, g_long_options, &option_index)) != -1) {
		switch (ret) {
		case 'a':
			outlvl = 0xffff;
			SET_BIT(dat->p_inode, P_INODE_STATE);
			break;
		case 's':
			SET_BIT(outlvl, EXT4_LVL_SUPER);
			break;
		case 'm':
			SET_BIT(outlvl, EXT4_LVL_BLOCK_MAP);
			mutexflag = 1;
			break;
		case 'i':
			if (0 == mutexflag) {
				/* 默认状态下，i选项不在打印gdt */
				SET_BIT(outlvl, EXT4_LVL_INODE);
				SET_BIT(dat->p_inode, P_INODE_STATE);
			}
			break;
		case 'I':
			tmp_ptr = optarg;
			if (0 == mutexflag) {
				/* 默认状态下，i选项不在打印gdt */
				SET_BIT(outlvl, EXT4_LVL_INODE);
				dat->p_inode = atoi(tmp_ptr);
			}
			break;
		case 'j':
			if (0 == mutexflag) {
				SET_BIT(outlvl, EXT4_LVL_JNL);
				dat->p_inode = 8; // 日志只要解析到inode=8就行了
			}
			break;
		case 'J':
			if (0 == mutexflag) {
				SET_BIT(outlvl, EXT4_LVL_JNL);
				SET_BIT(outlvl, EXT4_LVL_JNL_DATA);
				dat->p_inode = 8; // 日志只要解析到inode=8就行了
			}
			break;
		case 'g':
			if (0 == mutexflag) {
				SET_BIT(outlvl, EXT4_LVL_GDT);
			}
			break;
		case 'd':
			if (0 == devmutex) {
				tmp_ptr = optarg;
				tmp_len = strlen(tmp_ptr);
				if (tmp_len >= MAX_PATH_LEN) {
					perr("name len:%d too long!!!", tmp_len);
					return -1;
				}

				memcpy(dat->path, tmp_ptr, strlen(tmp_ptr));
				dat->path[tmp_len] = '\0';
				devmutex = 1; /* 只能设置一次 */
				dat->mode = DEVICE_MODE;
			}
			break;
		case 'D':
			tmp_ptr = optarg;
			num = strtol(tmp_ptr, &endptr, 10);
			if (num < 0) {
				// 处理溢出
				perr("num < 0");
				return -1;
			} else if (*endptr != '\0') {
				// 处理无效字符（如"123abc"中的'a'）
				perr("Invalid number: %s\n", tmp_ptr);
				return -1;
			}
			dat->dump_bolcknum = num;
			dat->ext_mode = DUMP_MODE_EXT;
			SET_BIT(outlvl, EXT4_LVL_DATA);
			break;
		case 'f':
			if (0 == devmutex) {
				tmp_ptr = optarg;
				tmp_len = strlen(tmp_ptr);
				if (tmp_len >= MAX_PATH_LEN) {
					perr("name len:%d too long!!!", tmp_len);
					return -1;
				}

				memcpy(dat->path, tmp_ptr, strlen(tmp_ptr));
				dat->path[tmp_len] = '\0';
				devmutex = 1; /* 只能设置一次 */
				dat->mode = FILE_MODE;
				if (dat->setbsz == 0)
					dat->setbsz = 4096; /* 默认大小4096 */
			}
			break;
		case 'b':
			tmp_ptr = optarg;
			dat->setbsz = atoi(tmp_ptr);
			break;
		case 'l':
			SET_BIT(dat->p_inode, P_INODE_STATE);
			dat->ext_mode = TREE_LIST_MODE_EXT;
			break;
		case 'h': // （-h or --help）查看帮助信息
			help_print();
			return -1;

		default:
			pimp("请输入\"./ext4tool -h\" or \"./ext4tool --help\" 来获取更多的信息！\n");
			return -1;
		}
	} /* end of while*/

	// 提示用户输入的命令中含有无效字符，并列举出所有的无效字符
	if (optind < argc) {
		fprintf(stderr, "无法识别的输入信息: ");
		while (optind < argc) {
			fprintf(stderr, "%s ", argv[optind++]);
		}
		fprintf(stderr, "\n");
		pimp("请输入\"./ext4tool -h\" or \"./ext4tool --help\" 来获取更多的信息！\n ");
		return -1;
	}
	return 0;
}

int main(int argc, char *argv[])
{
	int ret = 0;
	struct ext4_tool dat;
	dir_entry_node_t *root_entries = NULL;
	printf("ext4 tool\n");
	memset(&dat, 0, EXT4_TOOL_LEN);

	if (cmd_analysis(&dat, argc, argv) < 0)
		return -1;
again:
	ret = ext4_tool_prepare(&dat);
	if (ret)
		goto close;
	ret = ext4_group_analysis(0, &dat);
	if (ret > 0) {
		ext4_tool_close(&dat);
		goto again;
	} else if (ret < 0) {
		ret = ext4_group_analysis(1, &dat);
	}

	if (dat.ext_mode == TREE_LIST_MODE_EXT) {
		for (dir_entry_node_t *node = sorted_list; node != NULL; node = node->next) {
			if (node->root_dir_inode == 2) { // 根目录的条目
				dir_entry_node_t *new_node = malloc(sizeof(dir_entry_node_t));
				memcpy(new_node, node, sizeof(dir_entry_node_t));
				new_node->next = root_entries;
				root_entries = new_node;
			}
		}

		if (root_entries != NULL) {
			print_directory_tree(root_entries, 0, &dat);
			// 释放临时列表
			while (root_entries != NULL) {
				dir_entry_node_t *tmp = root_entries;
				root_entries = root_entries->next;
				free(tmp);
			}
		}
	}

close:
	ext4_tool_close(&dat);
	return ret;
}


```