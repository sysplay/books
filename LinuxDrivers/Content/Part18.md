# File Systems: The Semester Project

> This eighteenth article, which is part of the series on Linux device drivers, kick starts with introducing the concept of a file system, by simulating one in user space.

If you have not yet guessed the semester project topic, then you should have a careful look at the table of contents of the various books on Linux device drivers. You’ll find that no book lists a chapter on file systems. Not because they are not used or not useful. In fact, there are close to 100 different file systems existing, as of today, and possibly most of them in active use. Then, why are they not talked in general? And are so many file systems required? Which one of them is the best? Let’s understand these one by one.

In reality, there is no best file system, and in fact there may not be one to come, as well. This is because a particular file system suits best only for a particular set of requirements. Thus, different requirements ask for different best file systems, leading to so many active file systems and many more to come. So, basically they are not generic enough to be talked generically, rather more of a research topic. And the reason for them being the toughest of all drivers is simple: Least availability of generic written material – the best documentation being the code of various file systems. Hmmm! Sounds perfect for a semester project, right?

In order to get to do such a research project, a lot of smaller steps has to be taken. The first one being understanding the concept of a file system itself. That could be done best by simulating one in user space. Shweta took the ownership of getting this first step done, as follows:

- Use a regular file for a partition, that is for the actual data storage of the file system (Hardware space equivalent)
- Design a basic file system structure (Kernel space equivalent) and implement the format/mkfs command for it, over the regular file
- Provide an interface/shell to type commands and operate on the file system, similar to the usual bash shell. In general, this step is achieved by the shell along with the corresponding file system driver in kernel space. But here that translation is embodied/simulated into the user space interface, itself. (Next article shall discuss this in detail)

The default regular file chosen is `.sfsf` (simula-ting file system file) in the current directory. Starting `.` (dot) is for it to be hidden. The basic file system design mainly contains two structures, the super block containing the info about the file system, and the file entry structure containing the info about each file in the file system. Here are Shweta’s defines and structures defined in `sfs_ds.h`

```C
#ifndef SFS_DS_H
#define SFS_DS_H

#define SIMULA_FS_TYPE 0x13090D15 /* Magic Number for our file system */
#define SIMULA_FS_BLOCK_SIZE 512 /* in bytes */
#define SIMULA_FS_ENTRY_SIZE 64 /* in bytes */
#define SIMULA_FS_DATA_BLOCK_CNT ((SIMULA_FS_ENTRY_SIZE - (16 + 3 * 4)) / 4)

#define SIMULA_DEFAULT_FILE ".sfsf"

typedef unsigned int uint4_t;

typedef struct sfs_super_block
{
	uint4_t type; /* Magic number to identify the file system */
	uint4_t block_size; /* Unit of allocation */
	uint4_t partition_size; /* in blocks */
	uint4_t entry_size; /* in bytes */ 
	uint4_t entry_table_size; /* in blocks */
	uint4_t entry_table_block_start; /* in blocks */
	uint4_t entry_count; /* Total entries in the file system */
	uint4_t data_block_start; /* in blocks */
	uint4_t reserved[SIMULA_FS_BLOCK_SIZE / 4 - 8];
} sfs_super_block_t; /* Making it of SIMULA_FS_BLOCK_SIZE */

typedef struct sfs_file_entry
{
	char name[16];
	uint4_t size; /* in bytes */
	uint4_t timestamp; /* Seconds since Epoch */
	uint4_t perms; /* Permissions for user */
	uint4_t blocks[SIMULA_FS_DATA_BLOCK_CNT];
} sfs_file_entry_t;

#endif
```

Note that Shweta has put some redundant fields in the `sfs_super_block_t` rather than computing them every time. And practically that is not space inefficient, as any way lot of empty reserved space is available in the super block, which is expected to be the complete first block of a partition. For example, `entry_count` is a redundant field as it is same as the entry table’s size (in bytes) divided by the entry’s size, which both are part of the super block structure. Moreover, the `data_block_start` number could also be computed by how many blocks have been used before it, but again not preferred.

Also, note the hard coded assumptions made about the block size of 512 bytes, and some random magic number for the simula file system. This magic number is used to verify that we are dealing with the right file system, when operating on it.

Coming to the file entry, it contains the following for every file:
- Name of up to 15 characters (1 byte for ”)
- Size in bytes
- Time stamp (creation or modification? Not yet decided by Shweta)
- Permissions (just one set, for the user)
- Array of block numbers for up to 9 data blocks. Why 9? To make the complete entry of `SIMULA_FS_ENTRY_SIZE (64)`.

With the file system design ready, the make file system `(mkfs)` or more commonly known format application is implemented next, as `format_sfs.c`:

```C
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

#include "sfs_ds.h"

#define SFS_ENTRY_RATIO 0.10 /* 10% of all blocks */
#define SFS_ENTRY_TABLE_BLOCK_START 1

sfs_super_block_t sb =
{
	.type = SIMULA_FS_TYPE,
	.block_size = SIMULA_FS_BLOCK_SIZE,
	.entry_size = SIMULA_FS_ENTRY_SIZE,
	.entry_table_block_start = SFS_ENTRY_TABLE_BLOCK_START
};
sfs_file_entry_t fe; /* All 0's */

void write_super_block(int sfs_handle, sfs_super_block_t *sb)
{
	write(sfs_handle, sb, sizeof(sfs_super_block_t));
}
void clear_file_entries(int sfs_handle, sfs_super_block_t *sb)
{
	int i;

	for (i = 0; i < sb->entry_count; i++)
	{
		write(sfs_handle, &fe, sizeof(fe));
	}
}
void mark_data_blocks(int sfs_handle, sfs_super_block_t *sb)
{
	char c = 0;

	lseek(sfs_handle, sb->partition_size * sb->block_size - 1, SEEK_SET);
	write(sfs_handle, &c, 1); /* To make the file size to partition size */
}

int main(int argc, char *argv[])
{
	int sfs_handle;

	if (argc != 2)
	{
		fprintf(stderr, "Usage: %s <partition size in 512-byte blocks>\n",
			argv[0]);
		return 1;
	}
	sb.partition_size = atoi(argv[1]);
	sb.entry_table_size = sb.partition_size * SFS_ENTRY_RATIO;
	sb.entry_count = sb.entry_table_size * sb.block_size / sb.entry_size;
	sb.data_block_start = SFS_ENTRY_TABLE_BLOCK_START + sb.entry_table_size;

	sfs_handle = creat(SIMULA_DEFAULT_FILE,
		S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);
	if (sfs_handle == -1)
	{
		perror("No permissions to format");
		return 2;
	}
	write_super_block(sfs_handle, &sb);
	clear_file_entries(sfs_handle, &sb);
	mark_data_blocks(sfs_handle, &sb);
	close(sfs_handle);
	return 0;
}
```

Note the heuristic of 10% of total space to be used for the file entries, defined by SFS_ENTRY_RATIO. Apart from that, this format application takes the partition size as input and accordingly creates the default file `.sfsf`, with:

- Its first block as the super block, using `write_super_block()`
- The next 10% of total blocks as the file entry’s zeroed out table, using `clear_file_entries()`
- And the remaining as the data blocks, using `mark_data_blocks()`. This basically writes a zero at the end, to actually extend the underlying file `.sfsf` to the partition size
As any other `format/mkfs` application, `format_sfs.c` is compiled using gcc, and executed on the shell as follows:

```shell
$ gcc format_sfs.c -o format_sfs
$ ./format_sfs 1024 # Partition size in blocks of 512-bytes
$ ls -al # List the .sfsf created with a size of 512 KiBytes
```

Figure 33 shows the `.sfsf` pictorially for the partition size of 1024 blocks of 512 bytes each.

![Figure 33](/Images/Part18/figure_33_simula_file_system.png)

## Summing up

With the above design of simula file system (`sfs_ds.h`), along with the implementation for its format command (`format_sfs.c`), Shweta has thus created the empty file system over the simulated partition `.sfsf`. Now, as listed earlier, the final step in simulating the user space file system is to create the interface/shell to type commands and operate on the empty file system just created on `.sfsf`. Let’s watch out for Shweta coding that as well, completing the first small step towards understanding their project.

