## Домашнее задание.

Работа с LVM

Что нужно сделать?

уменьшить том под / до 8G
выделить том под /home
выделить том под /var (/var - сделать в mirror)
для /home - сделать том для снэпшотов
прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)
Работа со снапшотами:
сгенерировать файлы в /home/
снять снэпшот
удалить часть файлов
восстановиться со снэпшота

(залоггировать работу можно утилитой script, скриншотами и т.п.)

Задание со звездочкой*
на нашей куче дисков попробовать поставить btrfs/zfs:
с кешем и снэпшотами
разметить здесь каталог /opt

### ***************************************************

[root@lvm ~]# lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0  63.4M  1 loop /snap/core20/1974
loop1                       7:1    0  53.3M  1 loop /snap/snapd/19457
loop2                       7:2    0 111.9M  1 loop /snap/lxd/24322
loop3                       7:3    0  63.9M  1 loop /snap/core20/2182
loop4                       7:4    0    87M  1 loop /snap/lxd/27948
loop5                       7:5    0  39.1M  1 loop /snap/snapd/21184
sda                         8:0    0   128G  0 disk 
├─sda1                      8:1    0     1M  0 part 
├─sda2                      8:2    0     2G  0 part /boot
└─sda3                      8:3    0   126G  0 part 
  └─ubuntu--vg-ubuntu--lv 253:0    0    63G  0 lvm  /
sdb                         8:16   0    10G  0 disk 
sdc                         8:32   0     2G  0 disk 
sdd                         8:48   0     1G  0 disk 
sde                         8:64   0     1G  0 disk 

### Подготовим временный том для / раздела:

[root@lvm ~]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.

[root@lvm ~]# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created

[root@lvm ~]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
WARNING: ext4 signature detected on /dev/vg_root/lv_root at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/vg_root/lv_root.
  Logical volume "lv_root" created.

### Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные:

root@lvm:/home/vagrant# mkfs.ext4 /dev/vg_root/lv_root
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2620416 4k blocks and 655360 inodes
Filesystem UUID: b0198511-a23c-43c4-a5af-98e89ecaa145
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Progress: [ 20%] [###################...............................................................................]
Progress: [ 40%] [#######################################...........................................................]
Progress: [ 60%] [##########################################################........................................]
Progress: [ 80%] [##############################################################################....................]


[root@lvm ~]# mount /dev/vg_root/lv_root /mnt

### Ставим программу для переноса и командой копируем все данные с / раздела в /mnt:

[root@lvm ~]# yum install dump

[root@lvm ~]# dump -0f /mnt/root.img /

[root@lvm ~]# move /mnt/root.img /tmp
[root@lvm ~]# cd /mnt
[root@lvm ~]# restore -rf /tmp/root.img

### Проверить что скопировалось можно командой ls /mnt.


### Сымитируем текущий root, сделаем в него chroot и обновим grub:

[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/;  do mount --bind $i /mnt/$i; done

[root@lvm ~]# chroot /mnt/

### Затем сконфигурируем grub для того, чтобы при старте перейти в новый /

[root@lvm /]# grub-mkconfig -o /boot/grub/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-1160.102.1.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-1160.102.1.el7.x86_64.img
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done


### Проверяем заменну в файле 
/boot/grub/grub.cfg заменить ubuntu_vg/ubuntu_lv на vg_root/lv_root

выходим  Ctrl+d 

Перезагружаемся и проверяем перенос 

root@lvm:/home/vagrant# lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0  63.9M  1 loop /snap/core20/2182
loop1                       7:1    0  63.4M  1 loop /snap/core20/1974
loop2                       7:2    0    87M  1 loop /snap/lxd/27948
loop3                       7:3    0 111.9M  1 loop /snap/lxd/24322
loop4                       7:4    0  39.1M  1 loop /snap/snapd/21184
loop5                       7:5    0  53.3M  1 loop /snap/snapd/19457
sda                         8:0    0   128G  0 disk
├─sda1                      8:1    0     1M  0 part
├─sda2                      8:2    0     2G  0 part /boot
└─sda3                      8:3    0   126G  0 part
  └─ubuntu--vg-ubuntu--lv 253:1    0    63G  0 lvm
sdb                         8:16   0    10G  0 disk
└─vg_root-lv_root         253:0    0    10G  0 lvm  /
sdc                         8:32   0     2G  0 disk
sdd                         8:48   0     1G  0 disk
sde                         8:64   0     1G  0 disk


### Теперь нам нужно изменить размер старой VG и вернуть на него рут. Изменяем размер на 8G:


root@lvm:/home/vagrant# resize2fs /dev/ubuntu-vg/ubuntu-lv 8G
resize2fs 1.46.5 (30-Dec-2021)
Please run 'e2fsck -f /dev/ubuntu-vg/ubuntu-lv' first.

root@lvm:/home/vagrant# e2fsck -f /dev/ubuntu-vg/ubuntu-lv
e2fsck 1.46.5 (30-Dec-2021)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/ubuntu-vg/ubuntu-lv: 84150/4128768 files (0.2% non-contiguous), 1687404/16514048 blocks
root@lvm:/home/vagrant# resize2fs /dev/ubuntu-vg/ubuntu-lv 8G
resize2fs 1.46.5 (30-Dec-2021)
Resizing the filesystem on /dev/ubuntu-vg/ubuntu-lv to 2097152 (4k) blocks.
The filesystem on /dev/ubuntu-vg/ubuntu-lv is now 2097152 (4k) blocks long.

root@lvm:/home/vagrant# lvreduce -L 8G /dev/ubuntu-vg/ubuntu-lv
  WARNING: Reducing active logical volume to 8.00 GiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce ubuntu-vg/ubuntu-lv? [y/n]: n
  Logical volume ubuntu-vg/ubuntu-lv NOT reduced.
root@lvm:/home/vagrant# lvreduce -L 8G /dev/ubuntu-vg/ubuntu-lv -r

### Размер LVM 8G

root@lvm:/home/vagrant# lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0  63.9M  1 loop /snap/core20/2182
loop1                       7:1    0  63.4M  1 loop /snap/core20/1974
loop2                       7:2    0    87M  1 loop /snap/lxd/27948
loop3                       7:3    0 111.9M  1 loop /snap/lxd/24322
loop4                       7:4    0  39.1M  1 loop /snap/snapd/21184
loop5                       7:5    0  53.3M  1 loop /snap/snapd/19457
sda                         8:0    0   128G  0 disk
├─sda1                      8:1    0     1M  0 part
├─sda2                      8:2    0     2G  0 part /boot
└─sda3                      8:3    0   126G  0 part
  └─ubuntu--vg-ubuntu--lv 253:1    0     8G  0 lvm
sdb                         8:16   0    10G  0 disk
└─vg_root-lv_root         253:0    0    10G  0 lvm  /
sdc                         8:32   0     2G  0 disk
sdd                         8:48   0     1G  0 disk
sde                         8:64   0     1G  0 disk

root@lvm:/mnt# df -hT
Filesystem                        Type   Size  Used Avail Use% Mounted on
tmpfs                             tmpfs  197M  976K  196M   1% /run
/dev/mapper/vg_root-lv_root       ext4   9.8G  5.4G  3.9G  59% /
tmpfs                             tmpfs  982M     0  982M   0% /dev/shm
tmpfs                             tmpfs  5.0M     0  5.0M   0% /run/lock
/dev/sda2                         ext4   2.0G  234M  1.6G  13% /boot
tmpfs                             tmpfs  197M  4.0K  197M   1% /run/user/1000
/dev/mapper/ubuntu--vg-ubuntu--lv ext4   7.6G  5.2G  2.1G  72% /mnt

### Возврящаем диск обратно, можно с переносом данных, но мы ничего не меняли, поэтому переносим корень и загрузчик




### Сымитируем текущий root, сделаем в него chroot и обновим grub:

[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/;  do mount --bind $i /mnt/$i; done

[root@lvm ~]# chroot /mnt/

Затем сконфигурируем grub для того, чтобы при старте перейти в новый /

[root@lvm /]# grub-mkconfig -o /boot/grub/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-1160.102.1.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-1160.102.1.el7.x86_64.img
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done


### Проверяем заменну в файле 
/boot/grub/grub.cfg заменить  vg_root/lv_root на  ubuntu_vg/ubuntu_lv

выходим  Ctrl+d 

Перезагружаемся

### Раздел перенесен

root@lvm:/# lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0  63.9M  1 loop /snap/core20/2182
loop1                       7:1    0 111.9M  1 loop /snap/lxd/24322
loop2                       7:2    0  63.4M  1 loop /snap/core20/1974
loop3                       7:3    0    87M  1 loop /snap/lxd/27948
loop4                       7:4    0  53.3M  1 loop /snap/snapd/19457
loop5                       7:5    0  39.1M  1 loop /snap/snapd/21184
sda                         8:0    0   128G  0 disk
├─sda1                      8:1    0     1M  0 part
├─sda2                      8:2    0     2G  0 part /boot
└─sda3                      8:3    0   126G  0 part
  ├─ubuntu--vg-ubuntu--lv 253:1    0     8G  0 lvm  /
  └─ubuntu--vg-ubuntu_lv2 253:2    0    10G  0 lvm
sdb                         8:16   0    10G  0 disk
└─vg_root-lv_root         253:0    0    10G  0 lvm
sdc                         8:32   0     2G  0 disk
sdd                         8:48   0     1G  0 disk
sde                         8:64   0     1G  0 disk


##  Выделить том под /home.
Создаем его

root@lvm:/# lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0  63.9M  1 loop /snap/core20/2182
loop1                       7:1    0 111.9M  1 loop /snap/lxd/24322
loop2                       7:2    0  63.4M  1 loop /snap/core20/1974
loop3                       7:3    0    87M  1 loop /snap/lxd/27948
loop4                       7:4    0  53.3M  1 loop /snap/snapd/19457
loop5                       7:5    0  39.1M  1 loop /snap/snapd/21184
sda                         8:0    0   128G  0 disk
├─sda1                      8:1    0     1M  0 part
├─sda2                      8:2    0     2G  0 part /boot
└─sda3                      8:3    0   126G  0 part
  └─ubuntu--vg-ubuntu--lv 253:1    0     8G  0 lvm  /
sdb                         8:16   0    10G  0 disk
└─vg_root-lv_root         253:0    0    10G  0 lvm
sdc                         8:32   0     2G  0 disk
sdd                         8:48   0     1G  0 disk
sde                         8:64   0     1G  0 disk
root@lvm:/# vgs
  VG        #PV #LV #SN Attr   VSize    VFree
  ubuntu-vg   1   1   0 wz--n- <126.00g <118.00g
  vg_root     1   1   0 wz--n-  <10.00g       0
root@lvm:/# lvcreate --size 1G -n ubuntu_lv_home ubuntu-vg
WARNING: ext4 signature detected on /dev/ubuntu-vg/ubuntu_lv_home at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/ubuntu-vg/ubuntu_lv_home.
  Logical volume "ubuntu_lv_home" created.
root@lvm:/# vgs
  VG        #PV #LV #SN Attr   VSize    VFree
  ubuntu-vg   1   2   0 wz--n- <126.00g <117.00g
  vg_root     1   1   0 wz--n-  <10.00g       0
root@lvm:/# lvs
  LV             VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  ubuntu-lv      ubuntu-vg -wi-ao----   8.00g
  ubuntu_lv_home ubuntu-vg -wi-a-----   1.00g
  lv_root        vg_root   -wi-a----- <10.00g

root@lvm:/# lsblk
NAME                          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                           7:0    0  63.9M  1 loop /snap/core20/2182
loop1                           7:1    0 111.9M  1 loop /snap/lxd/24322
loop2                           7:2    0  63.4M  1 loop /snap/core20/1974
loop3                           7:3    0    87M  1 loop /snap/lxd/27948
loop4                           7:4    0  53.3M  1 loop /snap/snapd/19457
loop5                           7:5    0  39.1M  1 loop /snap/snapd/21184
sda                             8:0    0   128G  0 disk
├─sda1                          8:1    0     1M  0 part
├─sda2                          8:2    0     2G  0 part /boot
└─sda3                          8:3    0   126G  0 part
  ├─ubuntu--vg-ubuntu--lv     253:1    0     8G  0 lvm  /
  └─ubuntu--vg-ubuntu_lv_home 253:2    0     1G  0 lvm
sdb                             8:16   0    10G  0 disk
└─vg_root-lv_root             253:0    0    10G  0 lvm
sdc                             8:32   0     2G  0 disk
sdd                             8:48   0     1G  0 disk
sde                             8:64   0     1G  0 disk


Выделить том под /var (/var - сделать в mirror) 

root@lvm:/# pvs
  PV         VG        Fmt  Attr PSize    PFree
  /dev/sda3  ubuntu-vg lvm2 a--  <126.00g <117.00g
  /dev/sdb   vg_root   lvm2 a--   <10.00g       0
root@lvm:/# vgex
vgexport  vgextend
root@lvm:/# pvcreate /dev/sdd /dev/sde
  Physical volume "/dev/sdd" successfully created.
  Physical volume "/dev/sde" successfully created.
root@lvm:/# pvs
  PV         VG        Fmt  Attr PSize    PFree
  /dev/sda3  ubuntu-vg lvm2 a--  <126.00g <117.00g
  /dev/sdb   vg_root   lvm2 a--   <10.00g       0
  /dev/sdd             lvm2 ---     1.00g    1.00g
  /dev/sde             lvm2 ---     1.00g    1.00g
root@lvm:/# vgcreate vg_var /dev/sdd /dev/sde
  Volume group "vg_var" successfully created
root@lvm:/# vgs
  VG        #PV #LV #SN Attr   VSize    VFree
  ubuntu-vg   1   2   0 wz--n- <126.00g <117.00g
  vg_root     1   1   0 wz--n-  <10.00g       0
  vg_var      2   0   0 wz--n-    1.99g    1.99g
root@lvm:/# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
root@lvm:/# lvs
  LV             VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  ubuntu-lv      ubuntu-vg -wi-ao----   8.00g
  ubuntu_lv_home ubuntu-vg -wi-a-----   1.00g
  lv_root        vg_root   -wi-a----- <10.00g
  lv_var         vg_var    rwi-a-r--- 952.00m                                    68.82
root@lvm:/# lsblk
NAME                          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                           7:0    0  63.9M  1 loop /snap/core20/2182
loop1                           7:1    0 111.9M  1 loop /snap/lxd/24322
loop2                           7:2    0  63.4M  1 loop /snap/core20/1974
loop3                           7:3    0    87M  1 loop /snap/lxd/27948
loop4                           7:4    0  53.3M  1 loop /snap/snapd/19457
loop5                           7:5    0  39.1M  1 loop /snap/snapd/21184
sda                             8:0    0   128G  0 disk
├─sda1                          8:1    0     1M  0 part
├─sda2                          8:2    0     2G  0 part /boot
└─sda3                          8:3    0   126G  0 part
  ├─ubuntu--vg-ubuntu--lv     253:1    0     8G  0 lvm  /
  └─ubuntu--vg-ubuntu_lv_home 253:2    0     1G  0 lvm
sdb                             8:16   0    10G  0 disk
└─vg_root-lv_root             253:0    0    10G  0 lvm
sdc                             8:32   0     2G  0 disk
sdd                             8:48   0     1G  0 disk
├─vg_var-lv_var_rmeta_0       253:3    0     4M  0 lvm
│ └─vg_var-lv_var             253:7    0   952M  0 lvm
└─vg_var-lv_var_rimage_0      253:4    0   952M  0 lvm
  └─vg_var-lv_var             253:7    0   952M  0 lvm


Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

Скопируем и смонтируем /home /var 

root@lvm:/# mount /dev/ubuntu-vg/ubuntu_lv_home /mnt
root@lvm:/# cp -aR /home/ /mnt/
root@lvm:/# ls /mnt
home  lost+found
root@lvm:/# ls /home/
vagrant
root@lvm:/# ls /mnt/home/
vagrant
root@lvm:/# mount /dev/ubuntu-vg/ubuntu_lv_home /home/
root@lvm:/# lsblk
NAME                          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                           7:0    0  63.9M  1 loop /snap/core20/2182
loop1                           7:1    0 111.9M  1 loop /snap/lxd/24322
loop2                           7:2    0  63.4M  1 loop /snap/core20/1974
loop3                           7:3    0    87M  1 loop /snap/lxd/27948
loop4                           7:4    0  53.3M  1 loop /snap/snapd/19457
loop5                           7:5    0  39.1M  1 loop /snap/snapd/21184
sda                             8:0    0   128G  0 disk
├─sda1                          8:1    0     1M  0 part
├─sda2                          8:2    0     2G  0 part /boot
└─sda3                          8:3    0   126G  0 part
  ├─ubuntu--vg-ubuntu--lv     253:1    0     8G  0 lvm  /
  └─ubuntu--vg-ubuntu_lv_home 253:2    0     1G  0 lvm  /home
                                                        /mnt
sdb                             8:16   0    10G  0 disk
└─vg_root-lv_root             253:0    0    10G  0 lvm
sdc                             8:32   0     2G  0 disk
sdd                             8:48   0     1G  0 disk
├─vg_var-lv_var_rmeta_0       253:3    0     4M  0 lvm
│ └─vg_var-lv_var             253:7    0   952M  0 lvm
└─vg_var-lv_var_rimage_0      253:4    0   952M  0 lvm
  └─vg_var-lv_var             253:7    0   952M  0 lvm
sde                             8:64   0     1G  0 disk
├─vg_var-lv_var_rmeta_1       253:5    0     4M  0 lvm
│ └─vg_var-lv_var             253:7    0   952M  0 lvm
└─vg_var-lv_var_rimage_1      253:6    0   952M  0 lvm
  └─vg_var-lv_var             253:7    0   952M  0 lvm
root@lvm:/# umount /mnt
root@lvm:/# lsblk
NAME                          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                           7:0    0  63.9M  1 loop /snap/core20/2182
loop1                           7:1    0 111.9M  1 loop /snap/lxd/24322
loop2                           7:2    0  63.4M  1 loop /snap/core20/1974
loop3                           7:3    0    87M  1 loop /snap/lxd/27948
loop4                           7:4    0  53.3M  1 loop /snap/snapd/19457
loop5                           7:5    0  39.1M  1 loop /snap/snapd/21184
sda                             8:0    0   128G  0 disk
├─sda1                          8:1    0     1M  0 part
├─sda2                          8:2    0     2G  0 part /boot
└─sda3                          8:3    0   126G  0 part
  ├─ubuntu--vg-ubuntu--lv     253:1    0     8G  0 lvm  /
  └─ubuntu--vg-ubuntu_lv_home 253:2    0     1G  0 lvm  /home
sdb                             8:16   0    10G  0 disk
└─vg_root-lv_root             253:0    0    10G  0 lvm
sdc                             8:32   0     2G  0 disk
sdd                             8:48   0     1G  0 disk
├─vg_var-lv_var_rmeta_0       253:3    0     4M  0 lvm
│ └─vg_var-lv_var             253:7    0   952M  0 lvm
└─vg_var-lv_var_rimage_0      253:4    0   952M  0 lvm
  └─vg_var-lv_var             253:7    0   952M  0 lvm
sde                             8:64   0     1G  0 disk
├─vg_var-lv_var_rmeta_1       253:5    0     4M  0 lvm
│ └─vg_var-lv_var             253:7    0   952M  0 lvm
└─vg_var-lv_var_rimage_1      253:6    0   952M  0 lvm
  └─vg_var-lv_var             253:7    0   952M  0 lvm
root@lvm:/# ls /home/
home  lost+found
root@lvm:/# echo "`blkid | grep home: | awk '{print $2}'`
> ^C
root@lvm:/# echo "`blkid | grep home: | awk '{print $2}'`/home ext4 defaults 0 0"
UUID="3394250b-fa10-4018-a374-e3a5ee71c6fd"/home ext4 defaults 0 0
root@lvm:/# cat /etc/fstab
'# /etc/fstab: static file system information.
'#
'# Use 'blkid' to print the universally unique identifier for a
'# device; this may be used with UUID= as a more robust way to name devices
'# that works even if disks are added and removed. See fstab(5).
'#
'# <file system> <mount point>   <type>  <options>       <dump>  <pass>
'# / was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
/dev/disk/by-id/dm-uuid-LVM-Jmvz2VciZHoChGPtTlFvr6Yh4aU13UY3oP3CvPJI9d8yWARQF51QbgDpPCi0PNJx / ext4 defaults 0 1
'# /boot was on /dev/sda2 during curtin installation
/dev/disk/by-uuid/da3abab5-0da8-40d6-bac7-fa94844ca836 /boot ext4 defaults 0 1
/swap.img       none    swap    sw      0       0
'#VAGRANT-BEGIN
'# The contents below are automatically generated by Vagrant. Do not modify.
'#VAGRANT-END
root@lvm:/# echo "`blkid | grep home: | awk '{print $2}'`/home ext4 defaults 0 0"  >> /etc/fstab

Правим руками fstab
root@lvm:/# cat /etc/fstab
'#/etc/fstab: static file system information.
'#
'#Use 'blkid' to print the universally unique identifier for a
'#device; this may be used with UUID= as a more robust way to name devices
'#that works even if disks are added and removed. See fstab(5).
'#
'#<file system> <mount point>   <type>  <options>       <dump>  <pass>
'#/ was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
/dev/disk/by-id/dm-uuid-LVM-Jmvz2VciZHoChGPtTlFvr6Yh4aU13UY3oP3CvPJI9d8yWARQF51QbgDpPCi0PNJx / ext4 defaults 0 1
'#/boot was on /dev/sda2 during curtin installation
/dev/disk/by-uuid/da3abab5-0da8-40d6-bac7-fa94844ca836 /boot ext4 defaults 0 1
/swap.img       none    swap    sw      0       0
'#VAGRANT-BEGIN
'#The contents below are automatically generated by Vagrant. Do not modify.
'#VAGRANT-END
/dev/disk/by-uuid/3394250b-fa10-4018-a374-e3a5ee71c6fd /home ext4 defaults 0 0



root@lvm:/# lsblk
NAME                          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                           7:0    0    87M  1 loop /snap/lxd/27948
loop1                           7:1    0  53.3M  1 loop /snap/snapd/19457
loop2                           7:2    0 111.9M  1 loop /snap/lxd/24322
loop3                           7:3    0  39.1M  1 loop /snap/snapd/21184
loop4                           7:4    0  63.9M  1 loop /snap/core20/2182
loop5                           7:5    0  63.4M  1 loop /snap/core20/1974
sda                             8:0    0   128G  0 disk
├─sda1                          8:1    0     1M  0 part
├─sda2                          8:2    0     2G  0 part /boot
└─sda3                          8:3    0   126G  0 part
  ├─ubuntu--vg-ubuntu--lv     253:1    0     8G  0 lvm  /
  └─ubuntu--vg-ubuntu_lv_home 253:2    0     1G  0 lvm  /home
sdb                             8:16   0    10G  0 disk
└─vg_root-lv_root             253:0    0    10G  0 lvm
sdc                             8:32   0     2G  0 disk
sdd                             8:48   0     1G  0 disk
├─vg_var-lv_var_rmeta_0       253:3    0     4M  0 lvm
│ └─vg_var-lv_var             253:7    0   952M  0 lvm
└─vg_var-lv_var_rimage_0      253:4    0   952M  0 lvm
  └─vg_var-lv_var             253:7    0   952M  0 lvm
sde                             8:64   0     1G  0 disk
├─vg_var-lv_var_rmeta_1       253:5    0     4M  0 lvm
│ └─vg_var-lv_var             253:7    0   952M  0 lvm
└─vg_var-lv_var_rimage_1      253:6    0   952M  0 lvm
  └─vg_var-lv_var             253:7    0   952M  0 lvm

### Проверяем /home 

C:\vagrant1>vagrant ssh
Last login: Thu Apr  4 19:21:40 2024 from 10.0.2.2
vagrant@lvm:~$ sudo su
root@lvm:/home/vagrant# ls
terminal2.log  terminal.log  terminat3.log
root@lvm:/home/vagrant# lsblk
NAME                          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                           7:0    0  63.9M  1 loop /snap/core20/2182
loop1                           7:1    0  63.4M  1 loop /snap/core20/1974
loop2                           7:2    0 111.9M  1 loop /snap/lxd/24322
loop3                           7:3    0  53.3M  1 loop /snap/snapd/19457
loop4                           7:4    0  39.1M  1 loop /snap/snapd/21184
loop5                           7:5    0    87M  1 loop /snap/lxd/27948
sda                             8:0    0   128G  0 disk
├─sda1                          8:1    0     1M  0 part
├─sda2                          8:2    0     2G  0 part /boot
└─sda3                          8:3    0   126G  0 part
  ├─ubuntu--vg-ubuntu--lv     253:1    0     8G  0 lvm  /
  └─ubuntu--vg-ubuntu_lv_home 253:2    0     1G  0 lvm  /home
sdb                             8:16   0    10G  0 disk
└─vg_root-lv_root             253:0    0    10G  0 lvm
sdc                             8:32   0     2G  0 disk
sdd                             8:48   0     1G  0 disk
├─vg_var-lv_var_rmeta_0       253:3    0     4M  0 lvm
│ └─vg_var-lv_var             253:7    0   952M  0 lvm
└─vg_var-lv_var_rimage_0      253:4    0   952M  0 lvm
  └─vg_var-lv_var             253:7    0   952M  0 lvm
sde                             8:64   0     1G  0 disk
├─vg_var-lv_var_rmeta_1       253:5    0     4M  0 lvm
│ └─vg_var-lv_var             253:7    0   952M  0 lvm
└─vg_var-lv_var_rimage_1      253:6    0   952M  0 lvm
  └─vg_var-lv_var             253:7    0   952M  0 lvm
  
###  аналогичное действие с /var
  
root@lvm:/home/vagrant#  echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
root@lvm:/home/vagrant# vi /etc/fstab
root@lvm:/home/vagrant# cat /etc/fstab
'#/etc/fstab: static file system information.
'#
'#Use 'blkid' to print the universally unique identifier for a
'#device; this may be used with UUID= as a more robust way to name devices
'#that works even if disks are added and removed. See fstab(5).
'#
'#<file system> <mount point>   <type>  <options>       <dump>  <pass>
'#/ was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
/dev/disk/by-id/dm-uuid-LVM-Jmvz2VciZHoChGPtTlFvr6Yh4aU13UY3oP3CvPJI9d8yWARQF51QbgDpPCi0PNJx / ext4 defaults 0 1
'#/boot was on /dev/sda2 during curtin installation
/dev/disk/by-uuid/da3abab5-0da8-40d6-bac7-fa94844ca836 /boot ext4 defaults 0 1
/swap.img       none    swap    sw      0       0
'#VAGRANT-BEGIN
'#The contents below are automatically generated by Vagrant. Do not modify.
'#VAGRANT-END
/dev/disk/by-uuid/3394250b-fa10-4018-a374-e3a5ee71c6fd /home ext4 defaults 0 0
/dev/disk/by-uuid/e86628d4-25ad-4e2d-9d24-d8f6dd446d1b /var ext4 defaults 0 0
root@lvm:/home/vagrant#


Last login: Thu Apr  4 19:27:06 2024 from 192.168.1.62
vagrant@lvm:~$ sudo su
root@lvm:/home/vagrant# lsblk
NAME                          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                           7:0    0  63.4M  1 loop /snap/core20/1974
loop1                           7:1    0  63.9M  1 loop /snap/core20/2182
loop2                           7:2    0 111.9M  1 loop /snap/lxd/24322
loop3                           7:3    0    87M  1 loop /snap/lxd/27948
loop4                           7:4    0  53.3M  1 loop /snap/snapd/19457
loop5                           7:5    0  39.1M  1 loop /snap/snapd/21184
sda                             8:0    0   128G  0 disk
├─sda1                          8:1    0     1M  0 part
├─sda2                          8:2    0     2G  0 part /boot
└─sda3                          8:3    0   126G  0 part
  ├─ubuntu--vg-ubuntu--lv     253:1    0     8G  0 lvm  /
  └─ubuntu--vg-ubuntu_lv_home 253:2    0     1G  0 lvm  /home
sdb                             8:16   0    10G  0 disk
└─vg_root-lv_root             253:0    0    10G  0 lvm
sdc                             8:32   0     2G  0 disk
sdd                             8:48   0     1G  0 disk
├─vg_var-lv_var_rmeta_0       253:3    0     4M  0 lvm
│ └─vg_var-lv_var             253:7    0   952M  0 lvm  /var
└─vg_var-lv_var_rimage_0      253:4    0   952M  0 lvm
  └─vg_var-lv_var             253:7    0   952M  0 lvm  /var
sde                             8:64   0     1G  0 disk
├─vg_var-lv_var_rmeta_1       253:5    0     4M  0 lvm
│ └─vg_var-lv_var             253:7    0   952M  0 lvm  /var
└─vg_var-lv_var_rimage_1      253:6    0   952M  0 lvm
  └─vg_var-lv_var             253:7    0   952M  0 lvm  /var


### Cгенерировать файлы в /home/

root@lvm:/home/vagrant# touch /home/vagrant/ﬁle{1..20}
root@lvm:/home/vagrant# ls -al
total 76
drwxr-x--- 4 vagrant vagrant  4096 Apr  4 19:36 .
drwxr-xr-x 5 root    root     4096 Apr  4 19:23 ..
-rw------- 1 vagrant vagrant    27 Apr  4 19:34 .bash_history
-rw-r--r-- 1 vagrant vagrant   220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 vagrant vagrant  3775 Jan 11 00:05 .bashrc
drwxr-xr-x 2 vagrant vagrant  4096 Jan 11 00:05 .cache
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle1
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle10
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle11
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle12
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle13
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle14
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle15
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle16
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle17
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle18
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle19
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle2
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle20
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle3
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle4
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle5
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle6
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle7
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle8
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle9
-rw-r--r-- 1 vagrant vagrant   807 Jan  6  2022 .profile
drwx------ 2 vagrant vagrant  4096 Apr  3 16:44 .ssh
-rw-r--r-- 1 root    root    16384 Apr  4 18:19 terminal2.log
-rw-r--r-- 1 root    root    20480 Apr  4 17:39 terminal.log
-rw-r--r-- 1 root    root      796 Apr  4 18:41 terminat3.log
-rw-r--r-- 1 vagrant vagrant    13 Jan 11 00:05 .vimrc

### снять снэпшот


root@lvm:/home/vagrant# lvcreate -L 100MB -s -n home_snap /dev/ubuntu-vg/ubuntu_lv_home
  Logical volume "home_snap" created.

### Удаляем часть файлов
root@lvm:/home/vagrant# rm /home/vagrant/ﬁle{10..20}


root@lvm:/tmp# lsblc
Command 'lsblc' not found, did you mean:
  command 'lsblk' from deb util-linux (2.37.2-4ubuntu3)
Try: apt install <deb name>
root@lvm:/tmp# lsblk
NAME                               MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                                7:0    0  63.4M  1 loop /snap/core20/1974
loop1                                7:1    0  63.9M  1 loop /snap/core20/2182
loop2                                7:2    0 111.9M  1 loop /snap/lxd/24322
loop3                                7:3    0    87M  1 loop /snap/lxd/27948
loop4                                7:4    0  53.3M  1 loop /snap/snapd/19457
loop5                                7:5    0  39.1M  1 loop /snap/snapd/21184
sda                                  8:0    0   128G  0 disk
├─sda1                               8:1    0     1M  0 part
├─sda2                               8:2    0     2G  0 part /boot
└─sda3                               8:3    0   126G  0 part
  ├─ubuntu--vg-ubuntu--lv          253:1    0     8G  0 lvm  /
  ├─ubuntu--vg-ubuntu_lv_home-real 253:8    0     1G  0 lvm
  │ ├─ubuntu--vg-ubuntu_lv_home    253:2    0     1G  0 lvm  /home
  │ └─ubuntu--vg-home_snap         253:10   0     1G  0 lvm
  └─ubuntu--vg-home_snap-cow       253:9    0   100M  0 lvm
    └─ubuntu--vg-home_snap         253:10   0     1G  0 lvm
sdb                                  8:16   0    10G  0 disk
└─vg_root-lv_root                  253:0    0    10G  0 lvm
sdc                                  8:32   0     2G  0 disk
sdd                                  8:48   0     1G  0 disk
├─vg_var-lv_var_rmeta_0            253:3    0     4M  0 lvm
│ └─vg_var-lv_var                  253:7    0   952M  0 lvm  /var
└─vg_var-lv_var_rimage_0           253:4    0   952M  0 lvm
  └─vg_var-lv_var                  253:7    0   952M  0 lvm  /var
sde                                  8:64   0     1G  0 disk
├─vg_var-lv_var_rmeta_1            253:5    0     4M  0 lvm
│ └─vg_var-lv_var                  253:7    0   952M  0 lvm  /var
└─vg_var-lv_var_rimage_1           253:6    0   952M  0 lvm
  └─vg_var-lv_var                  253:7    0   952M  0 lvm  /var


root@lvm:/tmp# lvconvert --merge /dev/ubuntu-vg/home_snap
  Delaying merge since origin is open.
  Merging of snapshot ubuntu-vg/home_snap will occur on next activation of ubuntu-vg/ubuntu_lv_home.
root@lvm:/tmp# umount /home
umount: /home: target is busy.
root@lvm:/tmp# ls /home/vagrant/
ﬁle16  ﬁle17  ﬁle18  ﬁle19  ﬁle20  terminal2.log  terminal.log  terminat3.log
root@lvm:/tmp# ls /dev/ubuntu-vg/
home_snap  ubuntu-lv  ubuntu_lv_home
root@lvm:/tmp# umount /dev/ubuntu-vg/ubuntu_lv_home
umount: /home: target is busy.
root@lvm:/tmp# umount /dev/ubuntu-vg/ubuntu_lv_home
umount: /home: target is busy.
root@lvm:/tmp# reboot

### Файлы восстановлены
vagrant@lvm:~$ cd /home/
vagrant@lvm:/home$ cd vagrant/
vagrant@lvm:~$ ls -al
total 76
drwxr-x--- 4 vagrant vagrant  4096 Apr  4 19:36 .
drwxr-xr-x 5 root    root     4096 Apr  4 19:23 ..
-rw------- 1 vagrant vagrant    27 Apr  4 19:34 .bash_history
-rw-r--r-- 1 vagrant vagrant   220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 vagrant vagrant  3775 Jan 11 00:05 .bashrc
drwxr-xr-x 2 vagrant vagrant  4096 Jan 11 00:05 .cache
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle1
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle10
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle11
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle12
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle13
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle14
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle15
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle16
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle17
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle18
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle19
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle2
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle20
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle3
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle4
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle5
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle6
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle7
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle8
-rw-r--r-- 1 root    root        0 Apr  4 19:36 ﬁle9
-rw-r--r-- 1 vagrant vagrant   807 Jan  6  2022 .profile
drwx------ 2 vagrant vagrant  4096 Apr  3 16:44 .ssh
-rw-r--r-- 1 root    root    16384 Apr  4 18:19 terminal2.log
-rw-r--r-- 1 root    root    20480 Apr  4 17:39 terminal.log
-rw-r--r-- 1 root    root      796 Apr  4 18:41 terminat3.log
-rw-r--r-- 1 vagrant vagrant    13 Jan 11 00:05 .vimrc
