Методическое пособие к занятию 
“Файловые системы и LVM-1” курса 
Administrator Linux.Professional 
Базовый стенд 
Для быстрого запуска окружения и работы с данными задачами рекомендуется 
использовать Vagrant-стенд из репозитория: https://github.com/Nickmob/vagrant_lvm1 
Стенд протестирован на VirtualBox 7.0.12, Vagrant 2.4, хостовая система: Ubuntu 22.04. 
LVM - начало работы 
Почти все команды требуют прав суперпользователя, поэтому сразу переходим в root: 
sudo -i 
Для начала необходимо определиться какие устройства мы хотим использовать в 
качестве Physical Volumes (далее - PV) для наших будущих Volume Groups (далее - VG). 
Для этого можно воспользоваться lsblk: 

[root@lvm ~]# lsblk 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT 
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot 
└─sda3                    8:3    0   39G  0 part 
├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  / 
└─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP] 
sdb                       8:16   0   10G  0 disk 
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 

На выделенных дисках будем экспериментировать. Диски sdb, sdc будем использовать 
для базовых вещей и снапшотов. На дисках sdd,sde создадим lvm mirror. 
Также можно воспользоваться утилитой lvmdiskscan: 

[root@lvm ~]# lvmdiskscan 
/dev/VolGroup00/LogVol00 [     <37.47 GiB] 
/dev/VolGroup00/LogVol01 [       1.50 GiB] 


/dev/sda2                [       1.00 GiB] 
/dev/sda3                [     <39.00 GiB] LVM physical volume 
/dev/sdb                 [      10.00 GiB] 
/dev/sdc                 [       2.00 GiB] 
/dev/sdd                 [       1.00 GiB] 
/dev/sde                 [       1.00 GiB] 
4 disks 
3 partitions 
0 LVM physical volume whole disks 
1 LVM physical volume 

Для начала разметим диск для будущего использования LVM - создадим PV: 
[root@otuslinux ~]# pvcreate /dev/sdb 
Physical volume "/dev/sdb" successfully created. 

Затем можно создавать первый уровень абстракции - VG: 

[root@otuslinux ~]# vgcreate otus /dev/sdb 
Volume group "otus" successfully created 

И в итоге создать Logical Volume (далее - LV): 

[root@otuslinux ~]# lvcreate -l+80%FREE -n test otus 
Logical volume "test" created. 

Посмотреть информацию о только что созданном Volume Group: 

[root@lvm ~]# vgdisplay otus 
--- Volume group --- 
VG Name               otus 
System ID 
Format                lvm2 
Metadata Areas        1 
Metadata Sequence No  2 
VG Access             read/write 
VG Status             resizable 
MAX LV                0 
Cur LV                1 
Open LV               0 
Max PV                0 
Cur PV                1 
Act PV                1 
VG Size               <10.00 GiB 
PE Size               4.00 MiB 
Total PE              2559 
Alloc PE / Size       2047 / <8.00 GiB 


Free  PE / Size       512 / 2.00 GiB 
VG UUID               DfTABD-Wwwz-ol8M-xzN9-YxkA-wN89-apFpPN 

Так, например, можно посмотреть информацию о том, какие диски входит в VG: 

[root@lvm ~]# vgdisplay -v otus | grep 'PV Name' 
PV Name               /dev/sdb 

На примере с расширением VG мы увидим, что сюда добавится еще один диск. 
Детальную информацию о LV получим командой: 

[root@lvm ~]# lvdisplay /dev/otus/test 
--- Logical volume --- 
LV Path                /dev/otus/test 
LV Name                test 
VG Name                otus 
LV UUID                6sDlQJ-oCtV-1RpY-do3t-0EwY-nab1-4OffOF 
LV Write Access        read/write 
LV Creation host, time lvm, 2023-12-13 14:28:36 +0000 
LV Status              available 
# open                 0 
LV Size                <8.00 GiB 
Current LE             2047 
Segments               1 
Allocation             inherit 
Read ahead sectors     auto 
- currently set to     8192 
Block device           253:2 

В сжатом виде информацию можно получить командами vgs и lvs: 

[root@lvm ~]# vgs 
VG         #PV #LV #SN Attr   VSize   VFree 
VolGroup00   1   2   0 wz--n- <38.97g    0 
otus         1   1   0 wz--n- <10.00g 2.00g 
[root@lvm ~]# lvs 
LV       VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert 
LogVol00 VolGroup00 -wi-ao---- <37.47g 
LogVol01 VolGroup00 -wi-ao----   1.50g 
test     otus       -wi-a-----  <8.00g 

Мы можем создать еще один LV из свободного места. На этот раз создадим не 
экстентами, а абсолютным значением в мегабайтах: 

[root@lvm ~]# lvcreate -L100M -n small otus 
Logical volume "small" created. 


[root@lvm ~]# lvs 
LV       VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert 
LogVol00 VolGroup00 -wi-ao---- <37.47g 
LogVol01 VolGroup00 -wi-ao----   1.50g 
small    otus       -wi-a----- 100.00m 
test     otus       -wi-a-----  <8.00g 

Создадим на LV файловую систему и смонтируем его 

[root@lvm ~]# mkfs.ext4 /dev/otus/test 
… 
Allocating group tables: done 
Writing inode tables: done 
Creating journal (32768 blocks): done 
Writing superblocks and ﬁlesystem accounting information: done 

[root@lvm ~]# mkdir /data 
[root@lvm ~]# mount /dev/otus/test /data/ 
[root@lvm ~]# mount | grep /data 
/dev/mapper/otus-test on /data type ext4 (rw,relatime,seclabel,data=ordered) 
Расширение LVM 
Допустим, перед нами встала проблема нехватки свободного места в директории /data. 
Мы можем расширить файловую систему на LV /dev/otus/test за счет нового блочного 
устройства /dev/sdc. 
Для начала так же необходимо создать PV: 

[root@lvm ~]# pvcreate /dev/sdc 
Physical volume "/dev/sdc" successfully created. 

Далее необходимо расширить VG добавив в него этот диск. 

[root@lvm ~]# vgextend otus /dev/sdc 
Volume group "otus" successfully extended 

Убедимся что новый диск присутствует в новой VG: 

[root@lvm ~]# vgdisplay -v otus | grep 'PV Name' 
PV Name               /dev/sdb 
PV Name               /dev/sdc 

И что места в VG прибавилось: 


[root@lvm ~]# vgs 
VG         #PV #LV #SN Attr   VSize   VFree 
VolGroup00   1   2   0 wz--n- <38.97g     0 
otus         2   2   0 wz--n-  11.99g <3.90g 

Сымитируем занятое место с помощью команды dd для большей наглядности: 

[root@lvm ~]# dd if=/dev/zero of=/data/test.log bs=1M \ 
count=8000 status=progress 
7071596544 bytes (7.1 GB) copied, 4.007152 s, 1.8 GB/s 
dd: error writing ‘/data/test.log’: No space left on device 
7880+0 records in 
7879+0 records out 
8262189056 bytes (8.3 GB) copied, 4.71664 s, 1.8 GB/s 

Теперь у нас занято 100% дискового пространства: 

[root@lvm ~]# df -Th /data/ 
Filesystem            Type  Size  Used Avail Use% Mounted on 
/dev/mapper/otus-test ext4  7.8G  7.8G     0 100% /data 

Увеличиваем LV за счет появившегося свободного места. Возьмем не все место - это 
для того, чтобы осталось место для демонстрации снапшотов: 

[root@lvm ~]# lvextend -l+80%FREE /dev/otus/test 
Size of logical volume otus/test changed from <8.00 GiB (2047 extents) to <11.12 GiB (2846 
extents). 
Logical volume otus/test successfully resized. 
Наблюдаем, что LV расширен до 11.12g: 

[root@lvm ~]# lvs /dev/otus/test 
LV   VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert 
test otus -wi-ao---- <11.12g 

Но файловая система при этом осталась прежнего размера: 

[root@lvm ~]# df -Th /data 
Filesystem            Type  Size  Used Avail Use% Mounted on 
/dev/mapper/otus-test ext4  7.8G  7.8G     0 100% /data 

Произведем resize файловой системы: 

[root@lvm ~]# resize2fs /dev/otus/test 
resize2fs 1.42.9 (28-Dec-2013) 
Filesystem at /dev/otus/test is mounted on /data; on-line resizing required 
old_desc_blocks = 1, new_desc_blocks = 2 


The ﬁlesystem on /dev/otus/test is now 2914304 blocks long. 

[root@lvm ~]# df -Th /data 
Filesystem            Type  Size  Used Avail Use% Mounted on 
/dev/mapper/otus-test ext4   11G  7.8G  2.6G  76% /data 

Допустим Вы забыли оставить место на снапшоты. Можно уменьшить существующий 
LV с помощью команды lvreduce, но перед этим необходимо отмонтировать файловую 
систему, проверить её на ошибки и уменьшить ее размер: 

[root@lvm ~]# umount /data/ 

[root@lvm ~]# e2fsck -fy /dev/otus/test 
e2fsck 1.42.9 (28-Dec-2013) 
Pass 1: Checking inodes, blocks, and sizes 
Pass 2: Checking directory structure 
Pass 3: Checking directory connectivity 
Pass 4: Checking reference counts 
Pass 5: Checking group summary information 
/dev/otus/test: 12/729088 ﬁles (0.0% non-contiguous), 2105907/2914304 blocks 

[root@lvm ~]# resize2fs /dev/otus/test 10G 
resize2fs 1.42.9 (28-Dec-2013) 
Resizing the ﬁlesystem on /dev/otus/test to 2621440 (4k) blocks. 
The ﬁlesystem on /dev/otus/test is now 2621440 blocks long. 

[root@lvm ~]# lvreduce /dev/otus/test -L 10G 
WARNING: Reducing active logical volume to 10.00 GiB. 
THIS MAY DESTROY YOUR DATA (ﬁlesystem etc.) 
Do you really want to reduce otus/test? [y/n]: y 
Size of logical volume otus/test changed from <11.12 GiB (2846 extents) to 10.00 GiB (2560 
extents). 
Logical volume otus/test successfully resized. 
[root@lvm ~]# mount /dev/otus/test /data/ 

Убедимся, что ФС и lvm необходимого размера: 

[root@lvm ~]# df -Th /data/ 
Filesystem            Type  Size  Used Avail Use% Mounted on 
/dev/mapper/otus-test ext4  9.8G  7.8G  1.6G  84% /data 

[root@lvm ~]# lvs /dev/otus/test 
LV   VG   Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert 
test otus -wi-ao---- 10.00g 


Работа со снапшотами 
Снапшот создается командой lvcreate, только с флагом -s, который указывает на то, что 
это снимок: 

[root@lvm ~]# lvcreate -L 500M -s -n test-snap /dev/otus/test 
Logical volume "test-snap" created. 

Проверим с помощью vgs: 

[root@lvm ~]# vgs -o +lv_size,lv_name | grep test 
otus         2   3   1 wz--n-  11.99g <1.41g  10.00g test 
otus         2   3   1 wz--n-  11.99g <1.41g 500.00m test-snap 

Команда lsblk, например, нам наглядно покажет, что произошло: 

[root@lvm ~]# lsblk 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT 
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot 
└─sda3                    8:3    0   39G  0 part 
├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  / 
└─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP] 
sdb                       8:16   0   10G  0 disk 
├─otus-small            253:3    0  100M  0 lvm 
└─otus-test-real        253:4    0   10G  0 lvm 
├─otus-test           253:2    0   10G  0 lvm  /data 
└─otus-test--snap     253:6    0   10G  0 lvm 
sdc                       8:32   0    2G  0 disk 
├─otus-test-real        253:4    0   10G  0 lvm 
│ ├─otus-test           253:2    0   10G  0 lvm  /data 
│ └─otus-test--snap     253:6    0   10G  0 lvm 
└─otus-test--snap-cow   253:5    0  500M  0 lvm 
└─otus-test--snap     253:6    0   10G  0 lvm 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 

Здесь otus-test-real — оригинальный LV, otus-test--snap — снапшот, а otus-test--snap-cow 
— copy-on-write, сюда пишутся изменения. 
Снапшот можно смонтировать как и любой другой LV: 

[root@lvm ~]# mkdir /data-snap 

[root@lvm ~]# mount /dev/otus/test-snap /data-snap/ 


[root@lvm ~]# ll /data-snap/ 
total 8068564 
drwx------. 2 root root      16384 Dec 13 14:34 lost+found 
-rw-r--r--. 1 root root 8262189056 Dec 13 14:40 test.log 

[root@lvm ~]# umount /data-snap 

Можно также восстановить предыдущее состояние. “Откатиться” на снапшот. Для 
этого сначала для большей наглядности удалим наш log файл: 

[root@lvm ~]# rm /data/test.log 
rm: remove regular ﬁle ‘/data/test.log’? y 

[root@lvm ~]# ll /data 
total 16 
drwx------. 2 root root 16384 Dec 13 14:34 lost+found 

[root@lvm ~]# umount /data 

[root@lvm ~]# lvconvert --merge /dev/otus/test-snap 
Merging of volume otus/test-snap started. 
otus/test: Merged: 99.94% 
otus/test: Merged: 100.00% 

[root@lvm ~]# mount /dev/otus/test /data 
[root@lvm ~]# ll /data 
total 8068564 
drwx------. 2 root root      16384 Dec 13 14:34 lost+found 
-rw-r--r--. 1 root root 8262189056 Dec 13 14:40 test.log 

Работа с LVM-RAID 
[root@lvm ~]# pvcreate /dev/sd{d,e} 
Physical volume "/dev/sdd" successfully created. 
Physical volume "/dev/sde" successfully created. 

[root@lvm ~]# vgcreate vg0 /dev/sd{d,e} 
Volume group "vg0" successfully created 

[root@lvm ~]# lvcreate -l+80%FREE -m1 -n mirror vg0 
Logical volume "mirror" created. 

[root@lvm ~]# lvs 
LV       VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert 


LogVol00 VolGroup00 -wi-ao---- <37.47g 
LogVol01 VolGroup00 -wi-ao----   1.50g 
small    otus       -wi-a----- 100.00m 
test     otus       -wi-ao----  10.00g 
mirror   vg0        rwi-a-r--- 816.00m                                    100.00 
Домашнее задание 
На имеющемся образе centos/7 - v. 1804.2 
1. Уменьшить том под / до 8G. 
2. Выделить том под /home. 
3. Выделить том под /var - сделать в mirror. 
4. /home - сделать том для снапшотов. 
5. Прописать монтирование в fstab. Попробовать с разными опциями и разными 
файловыми системами (на выбор). 
6. Работа со снапшотами: 
a. сгенерить файлы в /home/; 
b. снять снапшот; 
c. удалить часть файлов; 
d. восстановится со снапшота. 
7. * На дисках попробовать поставить btrfs/zfs — с кешем, снапшотами и 
разметить там каталог /opt. 
Логировать работу можно с помощью утилиты script. 
Уменьшить том под / до 8G 
Эту часть можно выполнить разными способами, в данном примере мы будем 
уменьшать / до 8G без использования LiveCD. 
Если вы оставили том /dev/sdb из прошлых примеров заполненным, очистите его (или 
создайте чистый стенд). 
Перед началом работы поставьте пакет xfsdump - он будет необходим для снятия копии 
/ тома. 
Подготовим временный том для / раздела: 

[root@lvm ~]# pvcreate /dev/sdb 
Physical volume "/dev/sdb" successfully created. 

[root@lvm ~]# vgcreate vg_root /dev/sdb 
Volume group "vg_root" successfully created 

[root@lvm ~]# lvcreate -n lv_root -l +100%FREE /dev/vg_root 
WARNING: ext4 signature detected on /dev/vg_root/lv_root at offset 1080. Wipe it? [y/n]: y 
Wiping ext4 signature on /dev/vg_root/lv_root. 
Logical volume "lv_root" created. 


Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные: 

[root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root 
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks 
=                       sectsz=512   attr=2, projid32bit=1 
=                       crc=1        ﬁnobt=0, sparse=0 
data     =                       bsize=4096   blocks=2620416, imaxpct=25 
=                       sunit=0      swidth=0 blks 
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1 
log      =internal log           bsize=4096   blocks=2560, version=2 
=                       sectsz=512   sunit=0 blks, lazy-count=1 
realtime =none                   extsz=4096   blocks=0, rtextents=0 

[root@lvm ~]# mount /dev/vg_root/lv_root /mnt 

Этой командой копируем все данные с / раздела в /mnt: 

[root@lvm ~]# yum install xfsdump 

[root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt 
… 
xfsrestore: Restore Status: SUCCESS 
Тут вывод большой, но в итоге вы должны увидеть SUCCESS. Проверить что 
скопировалось можно командой ls /mnt. 

Затем сконфигурируем grub для того, чтобы при старте перейти в новый /. 
Сымитируем текущий root, сделаем в него chroot и обновим grub: 

[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; \ 
do mount --bind $i /mnt/$i; done 

[root@lvm ~]# chroot /mnt/ 

[root@lvm /]# grub2-mkconﬁg -o /boot/grub2/grub.cfg 
Generating grub conﬁguration ﬁle ... 
Found linux image: /boot/vmlinuz-3.10.0-1160.102.1.el7.x86_64 
Found initrd image: /boot/initramfs-3.10.0-1160.102.1.el7.x86_64.img 
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64 
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img 
done 

Обновим образ initrd. Что это такое и зачем нужно вы узнаете из следующей лекции. 

[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; \ 
do dracut -v $i `echo $i|sed "s/initramfs-//g; \ 


> s/.img//g"` --force; done 

Ну и для того, чтобы при загрузке был смонтирован нужны root нужно в файле 
/boot/grub2/grub.cfg заменить rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root 

[root@lvm ~]# lsblk 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT 
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot 
└─sda3                    8:3    0   39G  0 part 
├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm 
└─VolGroup00-LogVol01 253:2    0  1.5G  0 lvm  [SWAP] 
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:1    0   10G  0 lvm  / 
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
├─vg0-mirror_rmeta_0    253:3    0    4M  0 lvm 
│ └─vg0-mirror          253:7    0  816M  0 lvm 
└─vg0-mirror_rimage_0   253:4    0  816M  0 lvm 
└─vg0-mirror          253:7    0  816M  0 lvm 
sde                       8:64   0    1G  0 disk 
├─vg0-mirror_rmeta_1    253:5    0    4M  0 lvm 
│ └─vg0-mirror          253:7    0  816M  0 lvm 
└─vg0-mirror_rimage_1   253:6    0  816M  0 lvm 
└─vg0-mirror          253:7    0  816M  0 lvm 

Теперь нам нужно изменить размер старой VG и вернуть на него рут. Для этого 
удаляем старый LV размером в 40G и создаём новый на 8G: 

[root@lvm ~]# lvremove /dev/VolGroup00/LogVol00 
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y 
Logical volume "LogVol00" successfully removed 

[root@lvm ~]# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00 
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y 
Wiping xfs signature on /dev/VolGroup00/LogVol00. 
Logical volume "LogVol00" created. 

Проделываем на нем те же операции, что и в первый раз: 

[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol00 

[root@lvm ~]# mount /dev/VolGroup00/LogVol00 /mnt 

[root@lvm ~]# xfsdump -J - /dev/vg_root/lv_root | \ 


xfsrestore -J - /mnt 
xfsrestore: using ﬁle dump (drive_simple) strategy 
… 
xfsrestore: Restore Status: SUCCESS 

Так же как в первый раз cконфигурируем grub, за исключением правки 
/etc/grub2/grub.cfg 

[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; \ 
do mount --bind $i /mnt/$i; done 
[root@lvm ~]# chroot /mnt/ 
[root@lvm /]# grub2-mkconﬁg -o /boot/grub2/grub.cfg 
Generating grub conﬁguration ﬁle ... 
Found linux image: /boot/vmlinuz-3.10.0-1160.102.1.el7.x86_64 
Found initrd image: /boot/initramfs-3.10.0-1160.102.1.el7.x86_64.img 
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64 
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img 
done 

[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; \ 
do dracut -v $i `echo $i|sed "s/initramfs-//g; \ 
> s/.img//g"` --force; done 
Executing: /sbin/dracut -v initramfs-3.10.0-1160.102.1.el7.x86_64.img 
3.10.0-1160.102.1.el7.x86_64 --force 
… 
*** Creating initramfs image ﬁle '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done *** 

Пока не перезагружаемся и не выходим из под chroot - мы можем заодно перенести 
/var. 
Выделить том под /var в зеркало 
На свободных дисках создаем зеркало: 

[root@lvm boot]# pvcreate /dev/sdc /dev/sdd 
Physical volume "/dev/sdc" successfully created. 
Physical volume "/dev/sdd" successfully created. 

[root@lvm boot]# vgcreate vg_var /dev/sdc /dev/sdd 
Volume group "vg_var" successfully created 

[root@lvm boot]# lvcreate -L 950M -m1 -n lv_var vg_var 
Rounding up size to full physical extent 952.00 MiB 
Logical volume "lv_var" created. 

Создаем на нем ФС и перемещаем туда /var: 


[root@lvm boot]# mkfs.ext4 /dev/vg_var/lv_var 
mke2fs 1.42.9 (28-Dec-2013) 
Writing superblocks and ﬁlesystem accounting information: done 

[root@lvm boot]# mount /dev/vg_var/lv_var /mnt 
[root@lvm boot]# cp -aR /var/* /mnt/ 

На всякий случай сохраняем содержимое старого var (или же можно его просто 
удалить): 

[root@lvm boot]# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar 

Ну и монтируем новый var в каталог /var: 

[root@lvm boot]# umount /mnt 
[root@lvm boot]# mount /dev/vg_var/lv_var /var 

Правим fstab для автоматического монтирования /var: 

[root@lvm boot]# echo "`blkid | grep var: | awk '{print $2}'` \ 
/var ext4 defaults 0 0" >> /etc/fstab 

После чего можно успешно перезагружаться в новый (уменьшенный root) и удалять 
временную Volume Group: 

[root@lvm ~]# lvremove /dev/vg_root/lv_root 
Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y 
Logical volume "lv_root" successfully removed 

[root@lvm ~]# vgremove /dev/vg_root 
Volume group "vg_root" successfully removed 

[root@lvm ~]# pvremove /dev/sdb 
Labels on physical volume "/dev/sdb" successfully wiped. 

Выделить том под /home 
Выделяем том под /home по тому же принципу что делали для /var: 

[root@lvm ~]# lvcreate -n LogVol_Home -L 2G /dev/VolGroup00 
Logical volume "LogVol_Home" created. 

[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol_Home 
[root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /mnt/ 


[root@lvm ~]# cp -aR /home/* /mnt/ 
[root@lvm ~]# rm -rf /home/* 
[root@lvm ~]# umount /mnt 
[root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /home/ 

Правим fstab для автоматического монтирования /home: 

[root@lvm ~]# echo "`blkid | grep Home | awk '{print $2}'` \ 
/home xfs defaults 0 0" >> /etc/fstab 

Работа со снапшотами 
Генерируем файлы в /home/: 

[root@lvm ~]# touch /home/ﬁle{1..20} 

Снять снапшот: 

[root@lvm ~]# lvcreate -L 100MB -s -n home_snap \ 
dev/VolGroup00/LogVol_Home 

Удалить часть файлов: 

[root@lvm ~]# rm -f /home/ﬁle{11..20} 

Процесс восстановления из снапшота: 

[root@lvm ~]# umount /home 
[root@lvm ~]# lvconvert --merge /dev/VolGroup00/home_snap 
Merging of volume VolGroup00/home_snap started. 
VolGroup00/LogVol_Home: Merged: 100.00% 
[root@lvm ~]# mount /home 
[root@lvm ~]# ls -al /home 
total 0 
drwxr-xr-x.  3 root    root    292 Dec 14 09:51 . 
drwxr-xr-x. 20 root    root    268 Dec 14 09:31 .. 
-rw-r--r--.  1 root    root      0 Dec 14 09:51 ﬁle1 
-rw-r--r--.  1 root    root      0 Dec 14 09:51 ﬁle10 
-rw-r--r--.  1 root    root      0 Dec 14 09:51 ﬁle11 
… 
