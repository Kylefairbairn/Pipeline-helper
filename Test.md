Expand an SD card root filesystem so /opt has more space

Your situation

• SD card: approximately 25 GB.
• Current root filesystem: approximately 10 GB.
• /opt is a directory inside /, not a separate filesystem.
• Goal: add at least 5 GB of usable capacity.
• Board has fdisk, but does not have resize2fs and cannot download tools.

Recommended for your board: flash the card, then resize it on a Linux computer with GParted before booting the board. Alternatively, build the image with a larger root partition from the start.

Increasing root capacity makes the space available to /opt and every other directory on root. It does not reserve 5 GB exclusively for /opt.

Choose a method

|Method                               |Where                              |Required tools                                        |Fits your current board?               |
|-------------------------------------|-----------------------------------|------------------------------------------------------|---------------------------------------|
|1. GParted after flashing            |Linux PC or GParted Live           |GParted with ext4 support                             |Yes; no board resize tools needed      |
|2. Command line after flashing       |Linux PC                           |growpart or fdisk, e2fsck, resize2fs                  |Yes; tools run on PC                   |
|3. Resize the running root           |Board                              |growpart or fdisk, resize2fs                          |Only after supplying resize2fs         |
|4. Transfer offline tools            |Board                              |Compatible executable and libraries                   |Possible without board internet        |
|5. Build a larger image              |Build computer                     |Image build tools, such as Yocto/Wic                  |Yes, if you control the image build    |
|6. Enlarge an existing image file    |Linux PC                           |truncate, losetup, partition editor, e2fsck, resize2fs|Yes; flash the modified image afterward|
|7. Separate partition mounted at /opt|PC for preparation, board for mount|fdisk, mkfs.ext4, copy/mount tools                    |Yes; avoids resizing root              |

Before changing anything

Back up the card or important files. Partition editing can make the board unbootable if the wrong device, start sector, or partition identity is used.

Examples assume the board’s disk is /dev/mmcblk0 and root is partition 1, /dev/mmcblk0p1. Substitute your actual device. On a PC, a USB card reader often appears as /dev/sdb, with /dev/sdb1; the same card can have different names on different machines.

Inspect the board:

df -hT / /opt
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
sudo fdisk -l /dev/mmcblk0
command -v resize2fs
ls /sbin/resize2fs /usr/sbin/resize2fs

If lsblk is unavailable, use fdisk -l and cat /proc/mounts. If resize2fs exists outside PATH, invoke its full path.

Check these conditions:

1. / and /opt are on the same filesystem.
2. The filesystem is ext4 for the resize2fs instructions below.
3. There is unused space immediately after the root partition. Total free space elsewhere on the card is insufficient for a simple expansion.
4. The card has enough actual capacity for all partitions. GB and GiB differ; +15G in fdisk is approximately 15 GiB. Root filesystem usable capacity is smaller due to metadata and reserved blocks.

Never run mkfs on your existing root partition. That formats it and destroys its contents.

Method 1 — GParted after flashing (recommended)

1. Flash the original image using your usual flashing tool.
2. Keep the card attached to the Linux computer. If you only have Windows or macOS, boot a computer into GParted Live or use another Linux computer. Ordinary Windows/macOS disk tools generally do not resize ext4.
3. Open GParted and identify the SD card by its capacity and device name.
4. Unmount any mounted card partitions through GParted. Do not select your PC’s system disk.
5. Select the 10 GB root partition and choose Resize/Move.
6. Keep its starting position unchanged. Increase the end to give at least 5 GiB more capacity; 16 GiB total gives headroom over an approximately 10 GiB original. Alternatively, use all contiguous unallocated space.
7. Apply the queued operation and wait for success. GParted resizes both the partition and its supported filesystem using the host’s tools.
8. Safely eject the card, insert it into the board, and boot.
9. Verify:

df -hT / /opt

If another partition sits after root, GParted may need an offline move of that partition before root can grow. Back up first and check board-specific boot requirements before moving boot partitions. Method 7 may avoid moving anything.

Method 2 — Command line on a Linux PC after flashing

Shut down the board before removing its card. These examples use a PC card device of /dev/sdb and root /dev/sdb1. Verify these paths before running any command.

2A. With growpart

Unmount root if the PC mounted it:

sudo umount /dev/sdb1
sudo growpart /dev/sdb 1
sudo e2fsck -f /dev/sdb1
sudo resize2fs /dev/sdb1

If e2fsck repairs errors, rerun it until it reports a clean filesystem before resizing. Do not proceed after unresolved errors. growpart expands to the next partition or disk boundary, rather than adding exactly 5 GB.

2B. With fdisk instead of growpart

Use this only for a normal partition you understand, not an LVM, RAID, or extended/logical layout. Prefer GParted if partition identities or boot requirements are unclear.

1. Unmount root and record the original layout:

sudo umount /dev/sdb1
sudo fdisk -l /dev/sdb
sudo fdisk /dev/sdb

2. In fdisk, enter p. Record partition 1’s exact Start sector, type, and boot flag. Record any PARTUUID used by the bootloader or /etc/fstab before proceeding.
3. Enter d and select partition 1 if asked. This removes the table entry, not the filesystem contents.
4. Enter n and recreate partition 1 in the same slot and with the same partition kind. On an MBR disk this may involve choosing p for primary. GPT prompts differ.
5. Enter the exact original Start sector. Never assume the default is correct.
6. Set the last sector using +16G for roughly 16 GiB total, provided enough space is available, or accept the available end for maximum expansion. This value is the new total partition size, not the amount added. Do not make it smaller than its original size.
7. If asked to remove an existing filesystem signature, answer N.
8. Restore the original partition type and boot flag if needed; use m to see the commands your fdisk version supports.
9. Enter p and verify unchanged start, correct partition number/type, a larger end, and no overlap.
10. For GPT, deleting/recreating can change the partition’s unique GUID/PARTUUID. Preserve or restore it with a capable editor, or update every boot/fstab reference before booting. If you cannot ensure this, enter q and use GParted or growpart instead.
11. Enter w only after all checks pass. Before writing, q exits without saving.

If the host cannot reread the table, safely remove/reinsert the card and recheck the layout. Then, with root still unmounted:

sudo e2fsck -f /dev/sdb1
sudo resize2fs /dev/sdb1

Resolve filesystem check errors first. Safely eject the card and boot the board. Verify using df -hT / /opt.

Method 3 — Resize the running board

Your board cannot finish an ext4 resize with fdisk alone. First make resize2fs available using Method 4, or use a PC instead.

If both tools exist:

sudo growpart /dev/mmcblk0 1
sudo resize2fs /dev/mmcblk0p1
df -hT / /opt

Run resize2fs only after the partition change succeeds and the kernel sees its larger size. If the partition table cannot be reread because root is busy, reboot before resize2fs.

If growpart is missing but resize2fs is available, use Method 2B’s fdisk procedure with the board’s paths, preserving the start and partition identity. Reboot after writing the new partition table, then run:

sudo resize2fs /dev/mmcblk0p1
df -hT / /opt

Modern ext4 can usually grow while mounted, but an embedded kernel may lack online resize support. If it fails for that reason, shut down and use the offline PC procedure. Never run e2fsck on a mounted root filesystem.

Method 4 — Supply resize2fs without downloading on the board

Find the board’s architecture:

uname -m

Obtain a trusted resize2fs build for that architecture and the board’s userspace ABI. A binary from an x86 Linux PC will not run on an ARM board. Dynamic binaries also need compatible shared libraries and a matching loader; copying just one executable may not work.

Options:

• Copy the matching tool and dependencies from your embedded SDK or a compatible system.
• Transfer a suitable offline package and install it through the board’s package manager, supplying dependencies too.
• Build a statically linked resize2fs for the board using your cross-compilation toolchain.
• Add the tool to the next firmware image.

Example transfer from another computer (replace the address and file):

scp ./resize2fs root@BOARD_IP:/tmp/resize2fs

On the board:

chmod +x /tmp/resize2fs
/tmp/resize2fs -V

After enlarging the partition and making sure the kernel sees it:

sudo /tmp/resize2fs /dev/mmcblk0p1

If you see Exec format error, check architecture. If an existing executable reports not found, its dynamic loader may be absent. If libraries are missing, use compatible dependencies or a trusted static build. Do not overwrite system libraries to force compatibility.

Method 5 — Build a larger root filesystem into the flashed image

This is useful when repeatedly flashing multiple boards. Raw flashing normally reproduces the image’s existing partition sizes; a larger SD card alone does not increase root capacity. An image-specific first-boot expansion service can do so only if that feature and its required tools are included.

Yocto / Wic example

Edit the root partition line in your existing board-specific .wks or .wks.in. Keep the BSP’s bootloader, offsets, disk naming, partition order, and other options intact.

For example, add a minimum size:

part / --source rootfs --ondisk mmcblk0 --fstype=ext4 --label rootfs --align 1024 --size 16384

Here 16384 means 16384 MiB (16 GiB). This is an illustrative root line, not a complete board image definition. --size is a minimum; content and sizing settings can make it larger. For an exact partition size, use --fixed-size 16384 instead, never both. A fixed-size build fails if the root contents cannot fit. Ensure all partitions fit the actual SD card.

Rebuild using your existing image target, then flash the resulting Wic image:

bitbake YOUR_IMAGE_TARGET

If your goal is extra free filesystem space rather than a fixed total partition size, your Wic version may support --extra-filesystem-space. Use your version’s documentation. --extra-partition-space leaves room outside the filesystem and does not by itself create usable free space inside /opt.

To include a resizer in future Yocto images, a common package is:

IMAGE_INSTALL:append = " e2fsprogs-resize2fs"

Verify that package name against your release’s e2fsprogs recipe/package output. Older Yocto releases use different override syntax. Keep the existing BSP image configuration rather than replacing it with a generic layout.

Method 6 — Enlarge an existing raw image before flashing

Use a Linux PC and work on a copy of an uncompressed, partitioned raw .img or .wic image. This is not a procedure for a compressed archive, Android sparse image, or signed firmware bundle.

cp original.wic expanded.wic
truncate -s 20G expanded.wic
sudo losetup --find --show --partscan expanded.wic

20G is an example whole-image size. Only use it if it is larger than the original file and smaller than the target card’s actual capacity. truncate can destroy the end of an image if given a smaller size.

Record the loop device returned, for example /dev/loop7. Verify:

lsblk /dev/loop7
sudo fdisk -l /dev/loop7

Assuming root is partition 1 and unused space follows it:

sudo growpart /dev/loop7 1

If partition nodes do not reflect the new size, detach and reattach the loop device with --partscan, then use the newly returned loop device name. GPT images may need their backup GPT header relocated to the new end; use a GPT-aware tool such as GParted/sgdisk and follow its repair prompt before growing the partition.

With the correct loop root partition unmounted:

sudo e2fsck -f /dev/loop7p1
sudo resize2fs /dev/loop7p1
sudo losetup -d /dev/loop7

Resolve e2fsck errors before resizing. Flash expanded.wic using your usual tool. The board now receives the already enlarged filesystem. Space beyond the image’s size remains unused unless you resize it afterward.

Method 7 — Create a separate partition for /opt

This gives /opt its own capacity without enlarging root. You need a new filesystem, so fdisk alone is still insufficient: run mkfs.ext4 on a PC if it is missing on the board.

1. Shut down the board and put the card into a Linux PC.
2. In GParted, create an ext4 partition of at least 5 GiB in unallocated space. Do not format root or overwrite a bootloader area. Alternatively, use fdisk’s n command, write the new entry, then format only the confirmed new partition with mkfs.ext4.
3. Boot the board. Identify the new partition. The example below assumes /dev/mmcblk0p2; substitute the actual new partition number. It may be p3 or another number if a boot partition already exists.
4. Stop every service/process using /opt, and ensure no subordinate mounts exist there. Work during maintenance with console access.
5. Mount the new filesystem temporarily:

sudo mkdir -p /mnt/new-opt
sudo mount /dev/mmcblk0p2 /mnt/new-opt

6. Copy all /opt contents, including dotfiles, preserving ownership, permissions, links, ACLs, and extended attributes. With a host/board rsync supporting these options:

sudo rsync -aHAX --numeric-ids /opt/ /mnt/new-opt/
sudo rsync -aHAXn --numeric-ids --checksum /opt/ /mnt/new-opt/

Inspect the dry-run output; it should show no required changes. If rsync is unavailable, copy offline on a Linux PC with suitable tools. A basic BusyBox cp may not preserve all required metadata.

7. Unmount the temporary mount, retain the original directory for rollback, and create an empty mountpoint:

sudo umount /mnt/new-opt
sudo mv /opt /opt.backup
sudo mkdir /opt

8. Find the new filesystem UUID using blkid /dev/mmcblk0p2. Back up /etc/fstab, then add an entry using the actual UUID:

UUID=ACTUAL_NEW_FILESYSTEM_UUID /opt ext4 defaults 0 2

On a minimal embedded image without boot-time fsck tools, use the BSP’s supported mount/check configuration; the final field may need to be 0 instead of 2. Ensure /opt mounts before services that need it.

9. Mount and verify before restarting services:

sudo mount /opt
df -hT /opt
ls -la /opt

10. Reboot and test applications. Keep /opt.backup until successful validation. It still uses root space; removing it later frees that original space. Do not remove it until you confirm /opt is the new mounted filesystem and the copied data is complete.

To roll back, stop services, unmount /opt, remove the added fstab entry, remove the empty mountpoint using rmdir, and rename /opt.backup back to /opt.

Mounting an empty partition over the original /opt hides its contents. Copy first; mounting does not migrate files automatically.

If root is not ext4

These commands are alternatives for the filesystem-growth step only; first enlarge the backing partition correctly:

|Filesystem      |Growth tool / approach                                                           |
|----------------|---------------------------------------------------------------------------------|
|ext3/ext4       |resize2fs; online growth depends on kernel/features                              |
|ext2            |resize2fs while unmounted                                                        |
|XFS             |xfs_growfs / on the mounted root                                                 |
|Btrfs           |btrfs filesystem resize max / for a simple single-device root                    |
|SquashFS / EROFS|Read-only images; rebuild or provide a separate writable /opt or writable overlay|

For LVM or encrypted/multilayer storage, intermediate layers also need resizing. Do not apply the simple partition recipe directly.

Troubleshooting

|Symptom                                                |Meaning / next step                                                                             |
|-------------------------------------------------------|------------------------------------------------------------------------------------------------|
|fdisk shows a bigger partition, df still shows 10 GB   |Filesystem has not been expanded; run its growth tool                                           |
|growpart reports NOCHANGE                              |Partition already reaches its boundary, or another partition blocks it; inspect layout          |
|resize2fs says filesystem is already the requested size|Kernel still sees old partition size, or partition was not enlarged; reboot/reinsert and inspect|
|Kernel cannot reread partition table                   |Reboot board, or reconnect card/loop device on PC before resizing                               |
|Bad magic number from resize2fs                        |Wrong partition or non-ext filesystem; check type, do not force                                 |
|Online resizing unsupported                            |Use offline host procedure with partition unmounted                                             |
|Board stops booting after fdisk                        |Check original start sector, type, flags, and boot references such as PARTUUID; use backup      |
|Root card is read-only or has I/O errors               |Resolve media/filesystem problems before attempting expansion                                   |
|Reflashing returns root to 10 GB                       |Original image contains a 10 GB layout; resize after every flash or prepare a larger image      |

Final verification

On the booted board:

df -hT / /opt
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS

For root expansion, both paths should show the same enlarged filesystem. For a separate /opt partition, they should show different filesystems and /opt should have the new capacity.

References

• resize2fs manual: partition versus filesystem, offline and online growth
• fdisk manual: partition editing and signature handling
• growpart source/manual
• GParted manual
• Yocto Wic partition sizing reference
• Yocto Wic image creation guide

Use the documentation matching your installed tools and Yocto release. Examples are instructions for your hardware; they have not been executed against your SD card.
