# File Systems: The Semester Project – Part III

> This twentieth article, which is part of the series on Linux device drivers, completes the basic simulation of a file system in user space.

Till now, Shweta had implemented 4 basic functionalities of the simulated file system (sfs) browser, namely quit (browser), list (files), create (an empty file), remove (a file). Here she adds 3 little advanced functionalities to get a feel of a complete basic file system:

- Changing permissions of a file
- Reading from a file
- Writing into a file

Here’s a sneak peek into her thinking process as how she came up with her implementations.

For the various command implementations, two very common requirements keep popping quite often:

- Getting the index of the entry for a particular filename
- Updating the entry for a given filename

Hence these two requirements, captured as the functions `sfs_lookup()` and `sfs_update()`:

```C
int sfs_lookup(int sfs_handle, char *fn, sfs_file_entry_t *fe)
{
	int i;

	lseek(sfs_handle, sb.entry_table_block_start * sb.block_size, SEEK_SET);
	for (i = 0; i < sb.entry_count; i++)
	{   
		read(sfs_handle, fe, sizeof(sfs_file_entry_t));
		if (!fe->name[0]) continue;
		if (strcmp(fe->name, fn) == 0) return i;
	}   

	return -1; 
}

void sfs_update(int sfs_handle, char *fn, int *size, int update_ts, int *perms)
{
	int i;
	sfs_file_entry_t fe;

	if ((i = sfs_lookup(sfs_handle, fn, &fe)) == -1)
	{
		printf("File %s doesn't exist\n", fn);
		return;
	}
	if (size) fe.size = *size;
	if (update_ts) fe.timestamp = time(NULL);
	if (perms && (*perms <= 07)) fe.perms = *perms;
	lseek(sfs_handle, sb.entry_table_block_start * sb.block_size +
						i * sb.entry_size, SEEK_SET);
	write(sfs_handle, &fe, sizeof(sfs_file_entry_t));
}
```

`sfs_lookup()` traverses through all the entries (skipping the invalid i.e. empty filename entries), till it finds the filename match and then returns the index and the entry of the matched entry in the function’s third parameter. It returns -1 in case of no match is found.

`sfs_update()` uses `sfs_lookup()` to get the entry and its index for the given filename. Then, it updates it back into the filesystem with: a) size, if passed (i.e. non-NULL), b) current timestamp if update_ts is set, c) permissions, if passed (i.e. non-NULL).

## Changing file permissions

In the Simula file system, file permissions are basically a combination ‘rwx‘ for the user only, stored as an integer, with Linux like notation of 4 for ‘r‘, 2 for ‘w‘, 1 for ‘x‘. For changing the permissions of a given filename, sfs_update() can be used best by passing NULL pointer for size, zero for update_ts, and the pointer to permissions to change for perms. So sfs_chperm() would be as follows:

```C
void sfs_chperm(int sfs_handle, char *fn, int perm)
{
	sfs_update(sfs_handle, fn, NULL, 0, &perm);
}
```

## Reading a file

Reading a file is basically sequentially reading & displaying the contents of the data blocks indicated by their position from the blocks array of file’s entry and displaying that on stdout’s file descriptor 1. A couple of things to be taken care of:

- File is assumed to be without holes, i.e. block position of 0 in the blocks array indicates no more data block’s for the file
- Reading should not go beyond the file size. Special care to be taken while reading the last block with data, as it may be partially valid.

Here’s the complete read function, keeping track of valid data using bytes left to read:

```C
uint1_t block[SIMULA_FS_BLOCK_SIZE]; // Shared as global with sfs_write

void sfs_read(int sfs_handle, char *fn)
{
	int i, block_i, already_read, rem_to_read, to_read;
	sfs_file_entry_t fe;

	if ((i = sfs_lookup(sfs_handle, fn, &fe)) == -1)
	{
		printf("File %s doesn't exist\n", fn);
		return;
	}
	already_read = 0;
	rem_to_read = fe.size;
	for (block_i = 0; block_i < SIMULA_FS_DATA_BLOCK_CNT; block_i++)
	{
		if (!fe.blocks[block_i]) break;
		to_read = (rem_to_read >= sb.block_size) ?
						sb.block_size : rem_to_read;
		lseek(sfs_handle, fe.blocks[block_i] * sb.block_size, SEEK_SET);
		read(sfs_handle, block, to_read);
		write(1, block, to_read);
		already_read += to_read;
		rem_to_read -= to_read;
		if (!rem_to_read) break;
	}
}
```

## Writing a file

Interestingly, write is not a trivial function. Getting data from the user through browser is okay. But based on that, free available blocks has to be obtained, filled and then their position be noted sequentially in the blocks array of the file’s entry. Typically, we do this whenever we have received a block full data, except the last block. That’s tricky – how do we know the last block? So, we read till end of input, marked by Control-D on its own line from the user – and that is indicated by a return of 0 from read. And in that case, we check if any non-full block of data is left to be written, and if yes follow the same procedure of obtaining a free available block, filling it up (with the partial data), and updating its position in the blocks array.

After all this, we have finally written the file data, along with the data block positions in the blocks array of the file’s entry. And now it’s time to update file’s entry with the total size of data written, as well as timestamp to currently modified. Once done, this entry has to be updated back into the entry table, which is the last step. And in this flow, the following shouldn’t be missed out during getting & filling up free blocks:

- Check for no more block positions available in blocks array of the file’s entry
- Check for no more free blocks available in the file system.

In either of the 2 cases, the thought is to do a graceful stop with data being written upto the maximum possible, and discarding the rest.

Once again all of these put together are in the function below:

```C
uint1_t block[SIMULA_FS_BLOCK_SIZE]; // Shared as global with sfs_read

void sfs_write(int sfs_handle, char *fn)
{
	int i, cur_read_i, to_read, cur_read, total_size, block_i, free_i;
	sfs_file_entry_t fe; 

	if ((i = sfs_lookup(sfs_handle, fn, &fe)) == -1) 
	{   
		printf("File %s doesn't exist\n", fn);
		return;
	}   
	cur_read_i = 0;
	to_read = sb.block_size;
	total_size = 0;
	block_i = 0;
	while ((cur_read = read(0, block + cur_read_i, to_read)) > 0)
	{   
		if (cur_read == to_read)
		{   
			/* Write this block */
			if (block_i == SIMULA_FS_DATA_BLOCK_CNT)
				break; /* File size limit */
			if ((free_i = get_data_block(sfs_handle)) == -1) 
				break; /* File system full */
			lseek(sfs_handle, free_i * sb.block_size, SEEK_SET);
			write(sfs_handle, block, sb.block_size);
			fe.blocks[block_i] = free_i;
			block_i++;
			total_size += sb.block_size;
			/* Reset various variables */
			cur_read_i = 0;
			to_read = sb.block_size;
		}
		else
		{
			cur_read_i += cur_read;
			to_read -= cur_read;
		}
	}
	if ((cur_read <= 0) && (cur_read_i))
	{
		/* Write this partial block */
		if ((block_i != SIMULA_FS_DATA_BLOCK_CNT) &&
			((fe.blocks[block_i] = get_data_block(sfs_handle)) != -1))
		{
			lseek(sfs_handle, fe.blocks[block_i] * sb.block_size,
									SEEK_SET);
			write(sfs_handle, block, cur_read_i);
			total_size += cur_read_i;
		}
	}

	fe.size = total_size;
	fe.timestamp = time(NULL);
	lseek(sfs_handle, sb.entry_table_block_start * sb.block_size +
						i * sb.entry_size, SEEK_SET);
	write(sfs_handle, &fe, sizeof(sfs_file_entry_t));
}
```

## The last stride

With the above 3 sfs command functions, the final change to the `browse_sfs()` function in (previous article’s) `browse_sfs.c` would be to add the cases for handling these 3 new commands of chperm, write, and read.

One of the daunting questions, if it has not yet bothered you, is how do you find the free available blocks. Notice that in sfs_write(), we just called a function `get_data_block()` and everything went smooth. But think through how would that be implemented. Do you need to traverse all the file entry’s every time to figure out which all has been used and the remaining are free. That would be killing. Instead, an easier technique would be to get the used ones by parsing all the file entry’s, only initially once, and then keep track of them whenever more entries are used or freed up. But for that a complete framework needs to be put in place, which includes:

- uint1_t typedef (in sfs_ds.h)
- used_blocks dynamic array to keep track of used blocks (in browse_sfs.c)
- Function init_browsing() to initialize the dynamic array used_blocks, i.e. allocate and mark the initial used blocks (in browse_sfs.c)
- Correspondingly the inverse function shut_browsing() to cleanup the same (in browse_sfs.c)
- And definitely the functions get_data_block() and put_data_block() to respectively get and put back the free data blocks based on the dynamic array used_blocks (in browse_sfs.c)
- All these thoughts, incorporated in the earlier sfs_ds.h and browse_sfs.c files, along with a Makefile and the earlier formatter application format_sfs.c, are available from sfs_code.tar.bz2. Once compiled into browse_sfs and executed as ./browse_sfs, it shows up as something like in Figure 35.

![Figure 35](/Images/Part20/figure_35_simula_file_system_browser_demo-1024x549.png)

## Summing up

As Shweta, showed the above working demo to her project-mate, he observed some miss-outs, and challenged her to find them out on her own. He hinted them to be related to the newly added functionality and ‘getting free block’ framework – some even visible from the demo, i.e. Figure 35. Can you help Shweta, find them out? If yes, post them in the comments below.

