## Loop device

A loop device is a dummy device which does not have a phyiscal storage medium.

It is file-based, where each loop device is backed by a disk image file that sits atop other actual (i.e., physical) block devices such as a disk, eMMC, sd card.

And each rw command to the loop device is translated corresponding operation done on the backing file (i.e., disk image).

It is best shown by the following code snippet, where ``file`` points to the disk image.

```c

iov_iter_bvec(&i, ITER_BVEC, bvec, 1, bvec->bv_len);

file_start_write(file);
bw = vfs_iter_write(file, &i, ppos);
file_end_write(file);

```

## From FS layer to BIO layer

Whenever a file system submits its block io requests to ask for data from a certain block number, this block number will be translated to a ``sector`` number.

To understand this, it is important to note that physical block device acts upon ``sector-addressed`` instead of ``block-addressed``.

Therefore, each ``logical`` block number corresponding to the file system is only meaningful to that file system and thus must be translated into a ``sector number`` that makes sense to the block device.

Consider we have an ext2 fs, whose ``block size = 1KB``.

When mounted on a block device that has a ``sector size`` of ``512B``, the block number requested must be multipled by ``1024B/512B = 2``.
That is, when file system submits a request asking for a block ``2`` (i.e., super block), block layer will pass a sector number ``4`` to the device instead of ``2``.

