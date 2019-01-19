---
layout: post
title: Расширение файловой системы с использованием механизмов LVM.
tags: linux lvm hdd
---


![LVM In Linux](/images/Configure-lvm-linux.png)

#### В этом посте будет описана процедура увеличения размера файловой системы с использованием LVM.

Сначала посмотрим исходный раздел файловой системы, который необходимо расширить. В моем случае это корневой раздел (_/dev/mapper/centos_sonarqube-root   20G  3.4G   17G  17% /_)
```shell
# df -h
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/centos_sonarqube-root   20G  3.4G   17G  17% /
devtmpfs                           3.9G     0  3.9G   0% /dev
tmpfs                              3.9G     0  3.9G   0% /dev/shm
tmpfs                              3.9G   26M  3.8G   1% /run
tmpfs                              3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda1                          497M  217M  280M  44% /boot
/dev/mapper/centos_sonarqube-tmp   2.0G  202M  1.8G  10% /tmp
/dev/mapper/centos_sonarqube-var   5.1G  3.7G  1.5G  72% /var
```

* Расширяем текущий HDD через оснастку консоли виртуализации. Тут все зависит от системы виртуализации, не буду описывать этот шаг подробно, предполагаю что это может сделать каждый.

* Что бы ОС увидела добавленное место, выполняем следующую команду:

```shell 
echo 1 > /sys/class/block/sda/device/rescan 
```

* Проверяем что место добавилось, для этого используем команду *__parted__*, и в появившемся приглашении вводим *__print free__.*

```shell
# parted
GNU Parted 3.1
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print free
Model: VMware Virtual disk (scsi)
Disk /dev/sda: 42.9GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:
Number  Start   End     Size    Type     File system  Flags
        32.3kB  1049kB  1016kB           Free Space
 1      1049kB  525MB   524MB   primary  xfs          boot
 2      525MB   29.6GB  29.1GB  primary               lvm
        29.6GB  42.9GB  13.4GB           Free Space
```

4. Далее необходимо разметить свободное место, для этого используем команду *__cfdisk__*.
В появившемся меню выбираем _пункт с пометкой Free space -> New -> Primary -> Вводим размер создаваемого раздела -> Write -> вводим  'yes' для подтверждения_.

* Перечитаем таблицу разделов с помощью команды *__partprobe__* 

```shell
# partprobe
```

* Проверяем, что раздел появился.

```shell
# fdisk -l
Disk /dev/sda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x00027b36
   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     1026047      512000   83  Linux
/dev/sda2         1026048    57794559    28384256   8e  Linux LVM
/dev/sda3        57794560    83886079    13045760   83  Linux
```

* Создаем Physical Volume (PV). Имя устройства, указываемое в качестве параметра, берем из вывод предыдущей команды.

```shell 
# pvcreate /dev/sda3
  Physical volume "/dev/sda3" successfully created.

# pvdisplay
  --- Physical volume ---
  PV Name               /dev/sda2
  VG Name               centos_sonarqube
  PV Size               <27.07 GiB / not usable 3.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              6929
  Free PE               0
  Allocated PE          6929
  PV UUID               3TtA5K-Wyjr-8D1L-nFpD-lV37-Dir0-oQHuNR
  "/dev/sda3" is a new physical volume of "12.44 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sda3
  VG Name
  PV Size               12.44 GiB
  Allocatable           NO
  PE Size               0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               2apyrl-QeZg-FLkU-Sjbb-02dC-yj70-jmYX3E
```

* Отобразим имеющиеся Volume Group (VG)и расширим нужный.

```shell
# vgdisplay
  --- Volume group ---
  VG Name               centos_sonarqube
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  5
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                4
  Open LV               4
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <27.07 GiB
  PE Size               4.00 MiB
  Total PE              6929
  Alloc PE / Size       6929 / <27.07 GiB
  Free  PE / Size       0 / 0
  VG UUID               qrvMPo-yHMc-fU5w-Y0zR-cCJA-a6s2-sQxeeQ

# vgextend centos_sonarqube /dev/sda3
  Volume group "centos_sonarqube" successfully extended
```
Проверим что размер VG centos_sonarqube дейтвительно увеличился.

```shell
# vgdisplay
  --- Volume group ---
  VG Name               centos_sonarqube
  System ID
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  6
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                4
  Open LV               4
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               39.50 GiB
  PE Size               4.00 MiB
  Total PE              10113
  Alloc PE / Size       6929 / <27.07 GiB
  Free  PE / Size       3184 / <12.44 GiB
  VG UUID               qrvMPo-yHMc-fU5w-Y0zR-cCJA-a6s2-sQxeeQ
```

Видно, что значение Free PE увеличилось на размер добавленного PV (~12 Gb).

* Посмотрим существующие Logical Volume (LV) и расширим требуемый.

```shell
# lvdisplay
--- Logical volume ---
  LV Path                /dev/centos_sonarqube/root
  LV Name                root
  VG Name                centos_sonarqube
  LV UUID                rfl0V1-Gu0G-xsYv-F6rA-Qns3-gOe0-LK1ZQm
  LV Write Access        read/write
  LV Creation host, time sonarqube.domain.ru, 2018-07-06 12:07:13 +0300
  LV Status              available
  # open                 1
  LV Size                19.76 GiB
  Current LE             5059
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:0

# lvextend -l+100%FREE /dev/centos_sonarqube/root
  Size of logical volume centos_sonarqube/root changed from 19.76 GiB (5059 extents) to <32.20 GiB (8243 extents).
  Logical volume centos_sonarqube/root successfully resized.
```
Убедимся, что размер LV centos_sonarqube/root действительно  увеличился:

```shell
# lvdisplay
  --- Logical volume ---
  LV Path                /dev/centos_sonarqube/root
  LV Name                root
  VG Name                centos_sonarqube
  LV UUID                rfl0V1-Gu0G-xsYv-F6rA-Qns3-gOe0-LK1ZQm
  LV Write Access        read/write
  LV Creation host, time sonarqube.parma.ru, 2018-07-06 12:07:13 +0300
  LV Status              available
  # open                 1
  LV Size                <32.20 GiB
  Current LE             8243
  Segments               2
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:0
```

* И наконец, расширим файловую систему. В случае с CentOS команда resize2fs была заменена на xfs_growfs.

```shell
# resize2fs /dev/mapper/centos_sonarqube-root
resize2fs 1.42.9 (28-Dec-2013)
resize2fs: Bad magic number in super-block while trying to open /dev/mapper/centos_sonarqube-root
Couldn't find valid filesystem superblock.
```
```shell
# xfs_growfs /dev/mapper/centos_sonarqube-root
meta-data=/dev/mapper/centos_sonarqube-root isize=512    agcount=4, agsize=1295104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=5180416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 5180416 to 8440832
```

* Проверяем что размер файловой системы действительно увеличился:

```shell
# df -h
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/centos_sonarqube-root   33G  3.4G   29G  11% /
devtmpfs                           3.9G     0  3.9G   0% /dev
tmpfs                              3.9G     0  3.9G   0% /dev/shm
tmpfs                              3.9G   26M  3.8G   1% /run
tmpfs                              3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda1                          497M  217M  280M  44% /boot
/dev/mapper/centos_sonarqube-tmp   2.0G  202M  1.8G  10% /tmp
/dev/mapper/centos_sonarqube-var   5.1G  3.7G  1.5G  72% /var
```

На этой увеличение раздела файловой системы можно считать завершенным.

