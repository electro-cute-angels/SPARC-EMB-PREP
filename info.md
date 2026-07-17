riccardo@mvc-tower:~$ sudo blkid
/dev/nvme2n1p2: UUID="1b519fec-58d9-4cfe-b850-c35824e099cd" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="dd1a2c66-987d-4330-a26d-937655a7d1af"
/dev/loop1: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop29: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop19: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/nvme0n1p1: PARTLABEL="Microsoft reserved partition" PARTUUID="eadb6fab-5a49-4490-b57a-b99e17f0640c"
/dev/nvme0n1p2: LABEL="DATA" BLOCK_SIZE="512" UUID="B60A9EA40A9E60E3" TYPE="ntfs" PARTLABEL="Basic data partition" PARTUUID="418e017b-3b68-4d02-8d5c-e8c905d506b1"
/dev/nvme3n1p1: PARTLABEL="Microsoft reserved partition" PARTUUID="8e726a2d-3d5b-46f2-a3c9-e94aad3b1040"
/dev/nvme3n1p2: LABEL="DATA" BLOCK_SIZE="512" UUID="108EA1EB8EA1C994" TYPE="ntfs" PARTLABEL="Basic data partition" PARTUUID="fa33dbd4-7a4a-444d-85bf-86545c0e25cc"
/dev/loop17: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop8: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop25: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop15: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop6: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop23: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop13: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop4: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop31: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop21: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop11: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/nvme2n1p1: UUID="C849-FF30" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="43df556d-2e57-4888-8636-fd95c02511f0"
/dev/loop2: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop0: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop28: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop18: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop9: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop26: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop16: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop7: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/nvme1n1p4: LABEL="WINRETOOLS" BLOCK_SIZE="512" UUID="F4DCB7FDDCB7B866" TYPE="ntfs" PARTUUID="2bb615d6-3a1d-4d90-a34f-0531a639d788"
/dev/nvme1n1p2: PARTLABEL="Microsoft reserved partition" PARTUUID="8d445ecf-9507-433b-9790-55186eb3f8bd"
/dev/nvme1n1p5: LABEL="DELLSUPPORT" BLOCK_SIZE="512" UUID="7EB8B2F2B8B2A7D3" TYPE="ntfs" PARTUUID="e8451b40-f7db-486e-ae3b-2ecc1e86671e"
/dev/nvme1n1p3: LABEL="OS" BLOCK_SIZE="512" UUID="B49E9B3C9E9AF5D8" TYPE="ntfs" PARTLABEL="Basic data partition" PARTUUID="65d32f7d-8a54-4fd0-83cb-5ea0dc6abe1e"
/dev/nvme1n1p1: LABEL="ESP" UUID="DC72-813C" BLOCK_SIZE="512" TYPE="vfat" PARTLABEL="EFI system partition" PARTUUID="8f46c2e4-d3e1-47ce-be6d-74bc70813672"
/dev/loop24: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop14: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop5: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop22: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop12: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop3: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop30: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop20: BLOCK_SIZE="131072" TYPE="squashfs"
/dev/loop10: BLOCK_SIZE="131072" TYPE="squashfs"
riccardo@mvc-tower:~$ sudo parted -l
Model: PC801 NVMe SED SK hynix 1TB (nvme)
Disk /dev/nvme0n1: 1024GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name                          Flags
 1      1049kB  135MB   134MB                Microsoft reserved partition  msftres
 2      135MB   1024GB  1024GB  ntfs         Basic data partition          msftdata


Model: PC801 NVMe SED SK hynix 1TB (nvme)
Disk /dev/nvme3n1: 1024GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name                          Flags
 1      1049kB  135MB   134MB                Microsoft reserved partition  msftres
 2      135MB   1024GB  1024GB  ntfs         Basic data partition          msftdata


Model: PC801 NVMe SED SK hynix 1TB (nvme)
Disk /dev/nvme2n1: 1024GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  1128MB  1127MB  fat32              boot, esp
 2      1128MB  1024GB  1023GB  ext4


Model: PC801 NVMe SED SK hynix 1TB (nvme)
Disk /dev/nvme1n1: 1024GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name                          Flags
 1      1049kB  456MB   455MB   fat32        EFI system partition          boot, esp
 2      456MB   590MB   134MB                Microsoft reserved partition  msftres
 3      590MB   1022GB  1021GB  ntfs         Basic data partition          msftdata
 4      1022GB  1023GB  1038MB  ntfs                                       hidden, diag, no_automount
 5      1023GB  1024GB  1603MB  ntfs                                       hidden, diag, no_automount


riccardo@mvc-tower:~$ 








































NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0         7:0    0     4K  1 loop /snap/bare/5
loop1         7:1    0  55.5M  1 loop /snap/core18/2999
loop2         7:2    0  55.5M  1 loop /snap/core18/2979
loop3         7:3    0    74M  1 loop /snap/core22/2411
loop4         7:4    0    74M  1 loop /snap/core22/2339
loop5         7:5    0  66.8M  1 loop /snap/core24/1587
loop6         7:6    0  66.8M  1 loop /snap/core24/1643
loop7         7:7    0  50.1M  1 loop /snap/snapd/27406
loop8         7:8    0   828K  1 loop /snap/snapd-desktop-integration/391
loop9         7:9    0  16.4M  1 loop /snap/firmware-updater/224
loop10        7:10   0  16.5M  1 loop /snap/firmware-updater/226
loop11        7:11   0 531.4M  1 loop /snap/gnome-42-2204/247
loop12        7:12   0 531.5M  1 loop /snap/gnome-42-2204/263
loop13        7:13   0 606.1M  1 loop /snap/gnome-46-2404/153
loop14        7:14   0  91.7M  1 loop /snap/gtk-common-themes/1535
loop15        7:15   0  49.3M  1 loop /snap/gwenview/151
loop16        7:16   0  43.4M  1 loop /snap/gwenview/153
loop17        7:17   0   1.4G  1 loop /snap/kf6-core24/36
loop18        7:18   0   1.5G  1 loop /snap/kf6-core24/58
loop19        7:19   0   1.2G  1 loop /snap/libreoffice/373
loop20        7:20   0   1.2G  1 loop /snap/libreoffice/374
loop21        7:21   0   4.3M  1 loop /snap/lxqt-support-core24/16
loop22        7:22   0   395M  1 loop /snap/mesa-2404/1165
loop23        7:23   0  15.6M  1 loop /snap/snap-store/1338
loop24        7:24   0  15.7M  1 loop /snap/snap-store/1367
loop25        7:25   0 252.8M  1 loop /snap/firefox/8595
loop26        7:26   0  49.3M  1 loop /snap/snapd/26865
loop28        7:28   0   828K  1 loop /snap/snapd-desktop-integration/387
loop29        7:29   0 321.1M  1 loop /snap/vlc/3777
loop30        7:30   0 252.7M  1 loop /snap/firefox/8585
loop31        7:31   0   402M  1 loop /snap/mesa-2404/1839
nvme0n1     259:0    0 953.9G  0 disk 
├─nvme0n1p1 259:1    0   128M  0 part 
└─nvme0n1p2 259:2    0 953.7G  0 part 
nvme1n1     259:3    0 953.9G  0 disk 
├─nvme1n1p1 259:5    0   434M  0 part 
├─nvme1n1p2 259:6    0   128M  0 part 
├─nvme1n1p3 259:7    0 950.9G  0 part /media/riccardo/OS
├─nvme1n1p4 259:8    0   990M  0 part 
└─nvme1n1p5 259:9    0   1.5G  0 part 
nvme3n1     259:4    0 953.9G  0 disk 
├─nvme3n1p1 259:10   0   128M  0 part 
└─nvme3n1p2 259:11   0 953.7G  0 part /media/riccardo/DATA1
nvme2n1     259:12   0 953.9G  0 disk 
├─nvme2n1p1 259:13   0     1G  0 part /boot/efi
└─nvme2n1p2 259:14   0 952.8G  0 part /
