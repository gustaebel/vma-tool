# vma-tool

Optimize and manage VMA (virtual machine archive) files created by Proxmox backup.

> [!WARNING]
> *vma-tool* is in the early stages of development. Use at your own risk! VMA
> files written with *vma-tool* may not be readable by Proxmox backup. Always
> check the results of *vma-tool* with Proxmox's `vma` program:
> ```sh
> vma verify -v test.vma
> ```


## VMA backups don't dedup well

By nature, Proxmox device images, both raw and qcow2 images, are well suited
for deduplication as they are made up of contiguous fixed-size data blocks.
Unfortunately, VMA archive files that are created by Proxmox's backup service
do not play at all well with the deduplication feature of existing backup
software, e.g. [restic](https://restic.net/) or
[librsync](https://github.com/librsync/librsync)-based solutions.

*vma-tool* repacks and reorganizes existing VMA files in a compatible way that
makes them optimal for deduplication.


### The small problem: The file format

Basically, a VMA file is made up of a VMA header that contains all the
information about what is stored in the archive, followed by a sequence of
extents that store the actual data of the included device images. Each data
extent consists of an extent header followed by a variable-length sequence of
4096-byte blocks (organized in 64 KiB clusters) and can store up to 3776 KiB in
data. The specification is
[here](https://git.proxmox.com/?p=pve-qemu.git&a=blob&f=vma_spec.txt).

The Problem: Every VMA file contains a header field with a randomly-generated
uuid which is referenced again in all the data extent headers. Unfortunately,
this uuid is different between each VMA file, even if they were created for the
same guest VM. So, even if two VMA files contain the exact same data, they are
guaranteed to differ after every 3776 KiB of data, i.e. at least 277 times per
1 GiB.

**Solution:** *vma-tool* repacks the VMA file using the same default uuid
`12345678-aabb-ccdd-eeff-aabbccddeeff` for every file. The fact that the uuid
is no longer random seems to have no adverse effects.


### The big problem: The order of blocks

Far worse with regard to effective deduplication is that the order in which the
64 KiB data clusters are stored is non-deterministic and therefore different
every time . This destroys any attempt at deduplication entirely.

**Solution:** *vma-tool* repacks the VMA file storing all data blocks in the
correct order.


## TL;DR

#### Install vma-tool

*vma-tool* is a single Python 3 script with no external dependencies, just
download it and make it executable.

#### Optimize a VMA file

```sh
vma-tool optimize source.vma destination.vma
```

If `destination.vma` is omitted `source.vma` will be overwritten with the
optimized version.

#### Unpack a VMA file

```sh
vma-tool unpack source.vma destination/
```

```sh
zstdcat source.vma.zst | vma-tool unpack - destination/
```

#### Pack a VMA file

```sh
vma-tool pack source/ destination.vma
```


## Tests and Numbers

I created a small makeshift test setup to demonstrate the effects with some
numbers. I used two test VMA files (let us call them `a.vma` and `b.vma`) which
both were **19125 MiB** in size.

For both *restic* and *librsync*, I did three passes with three different
versions of the same VMA files:

- original: The unaltered original VMA files as Proxmox backup produced them.
- ordered data: Repacked versions of the VMA files with the data blocks in the
  right order.
- optimized: Repacked versions with ordered data blocks *and* the uuid "hack".

#### restic

The *restic* test creates an empty repository with compression switched off to
test the isolated effects of deduplication. The *Delta Size* column displays
the size difference of the restic repository between before and after adding
`b.vma` to it.

```sh
restic init

restic backup a.vma
size1 = get-size a.vma

restic backup b.vma
size2 = get-size b.vma

delta-size = size2 - size1
```

| Test          | Delta Size    | Space Saved   |
| ------------- | ------------- | ------------- |
| original      | 19125 MiB     | 0.0 %         |
| ordered data  | 7579 MiB      | 60.4 %        |
| optimized     | 48 MiB        | 99.7 %        |


#### librsync

The *librsync* test is similar. `rdiff` is used to create the delta
between `a.vma` and `b.vma`. The size of the resulting delta file is shown in
the column *Delta Size*.

```sh
rdiff signature a.vma sig
rdiff delta sig b.vma delta-file

delta-size = get-size delta-file
```

The column *Space Saved* compares the *Delta Size* to the original file size of
`b.vma`, i.e. 19125 MiB.

| Test          | Delta Size    | Space Saved   |
| ------------- | ------------- | ------------- |
| original      | 18772 MiB     | 1.8 %         |
| ordered data  | 820 MiB       | 95.7 %        |
| optimized     | 25 MiB        | 99.9 %        |
