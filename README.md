# vma-tool

Optimize and manage VMA (virtual machine archive) files created by Proxmox backup.


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

Solution: *vma-tool* repacks the VMA file using the same default uuid
(`12345678-aabb-ccdd-eeff-aabbccddeeff`) for every file. The fact that the uuid
is no longer random seems to have no adverse effects.


### The big problem: The order of blocks

Far worse with regard to effective deduplication is that the order in which the
64 KiB data clusters are stored is non-deterministic and therefore different
every time . This destroys any attempt at deduplication entirely.

Solution: *vma-tool* repacks the VMA file storing all data blocks in the
correct order.

