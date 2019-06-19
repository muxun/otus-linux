# otus-linux
<details>
<summary> Домашнее задание №1 Kernel </summary>
<p>

При выполненни домашнего задания использовались два способа сборки ядра - oldconfig и menuconfig <br>

Cборка ядра menuconfig: <br>
* обновлена система и установлены необходимые пакеты

```
# yum update
# yum install -y ncurses-devel make gcc bc bison flex elfutils-libelf-devel openssl-devel grub2

```

* скачано ядро 5.1

```
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.1.tar.xz

```
 
* скачана текущая конфигруация ядра в папку новго ядра

```
cp -v /boot/config-3.10.0-693.5.2.el7.x86_64 /usr/src/linux-5*/.config

```  
* с помощью псевдо графического интерфейса выбраны необходимые модули (оставил без практически без изменений, так как не совсем понимаю что нужно а что нет)

```
make menuconfig

```
*  запустил сборку ядра

```
# make bzImage
# make modules
# make
# make install
# make modules_install

```

Сборка ядра с помощью oldconfig

* проделаны первые шаги из предыдущего этапа и использованы команды из пособия к лекции

```
cp /boot/config* .config && make oldconfig && make && make install &&make modules_install

```

в результате в grub можно загрузиться с новым ядром

<img src="https://raw.githubusercontent.com/muxun/otus-linux/kernel/img/kerner.png"></img>
</p>
</details>


<details>
<summary> Домашнее задание №2 Raid </summary>
<p>
  Расширил количество дисков до 6
  
```
[vagrant@otuslinux ~]$ lsscsi
[0:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sda 
[3:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdb 
[4:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdc 
[5:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdd 
[6:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sde 
[7:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdf 
[8:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdg 
[vagrant@otuslinux ~]$ sudo lshw -short | grep disk 
/0/100/1.1/0.0.0    /dev/sda  disk        42GB VBOX HARDDISK
/0/100/d/0          /dev/sdb  disk        262MB VBOX HARDDISK
/0/100/d/1          /dev/sdc  disk        262MB VBOX HARDDISK
/0/100/d/2          /dev/sdd  disk        262MB VBOX HARDDISK
/0/100/d/3          /dev/sde  disk        262MB VBOX HARDDISK
/0/100/d/4          /dev/sdf  disk        262MB VBOX HARDDISK
/0/100/d/5          /dev/sdg  disk        262MB VBOX HARDDISK
```
Обнулил суперблоки и создал рэйд 1

```
[vagrant@otuslinux ~]$ sudo mdadm --zero-superblock --force /dev/sd{b,c,d,e,f,g}
mdadm: Unrecognised md component device - /dev/sdb
mdadm: Unrecognised md component device - /dev/sdc
mdadm: Unrecognised md component device - /dev/sdd
mdadm: Unrecognised md component device - /dev/sde
mdadm: Unrecognised md component device - /dev/sdf
mdadm: Unrecognised md component device - /dev/sdg
[vagrant@otuslinux ~]$ sudo mdadm --create --verbose /dev/md0 -l 1 -n 6 /dev/sd{b,c,d,e,f,g}
mdadm: Note: this array has metadata at the start and
    may not be suitable as a boot device.  If you plan to
    store '/boot' on this device please ensure that
    your boot-loader understands md/v1.x metadata, or use
    --metadata=0.90
mdadm: size set to 254976K
Continue creating array? y
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```
Проверим корректность рейда

```
[vagrant@otuslinux ~]$ cat /proc/mdstat
Personalities : [raid1] 
md0 : active raid1 sdg[5] sdf[4] sde[3] sdd[2] sdc[1] sdb[0]
      254976 blocks super 1.2 [6/6] [UUUUUU]
```
Создадим mdadm.conf 

```
[vagrant@otuslinux etc]$ sudo -s                                          
[root@otuslinux etc]# echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
[root@otuslinux etc]# mdadm --detail --scan --verbose | awk '/ARRAY/{print}' >> /etc/mdadm/mdadm.conf
[root@otuslinux etc]# cat /etc/mdadm/mdadm.conf 
DEVICE partitions
ARRAY /dev/md0 level=raid1 num-devices=6 metadata=1.2 name=otuslinux:0 UUID=b34fcf7c:3cd7e117:8363f585:59a5d5e5

```

Упражнение на поломку raid 

```
[root@otuslinux etc]# mdadm /dev/md0 --fail /dev/sde
mdadm: set /dev/sde faulty in /dev/md0
[root@otuslinux etc]# cat /proc/mdstat
Personalities : [raid1] 
md0 : active raid1 sdg[5] sdf[4] sde[3](F) sdd[2] sdc[1] sdb[0]
      254976 blocks super 1.2 [6/5] [UUU_UU]
      
unused devices: <none>

##############################################################
    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       -       0        0        3      removed
       4       8       80        4      active sync   /dev/sdf
       5       8       96        5      active sync   /dev/sdg

       3       8       64        -      faulty   /dev/sde
```

Эмулируем замену физического диска 

```
[root@otuslinux etc]# mdadm /dev/md0 --remove /dev/sde
mdadm: hot removed /dev/sde from /dev/md0

[root@otuslinux etc]# mdadm /dev/md0 --add /dev/sde
mdadm: added /dev/sde

[root@otuslinux etc]# cat /proc/mdstat
Personalities : [raid1] 
md0 : active raid1 sde[6] sdg[5] sdf[4] sdd[2] sdc[1] sdb[0]
      254976 blocks super 1.2 [6/6] [UUUUUU]
      
unused devices: <none>

[root@otuslinux etc]# mdadm -D /dev/md0
/dev/md0:
    ..................
    ..................
    ..................
    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       6       8       64        3      active sync   /dev/sde
       4       8       80        4      active sync   /dev/sdf
       5       8       96        5      active sync   /dev/sdg

```

Напилим партиций и смонтируем на диск 

```
[root@otuslinux etc]# lsblk
NAME      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda         8:0    0   40G  0 disk  
`-sda1      8:1    0   40G  0 part  /
sdb         8:16   0  250M  0 disk  
`-md0       9:0    0  249M  0 raid1 
  |-md0p1 259:0    0   49M  0 md    /raid/part1
  |-md0p2 259:1    0   50M  0 md    /raid/part2
  |-md0p3 259:2    0   49M  0 md    /raid/part3
  |-md0p4 259:3    0   50M  0 md    /raid/part4
  `-md0p5 259:4    0   49M  0 md    /raid/part5
sdc         8:32   0  250M  0 disk  
`-md0       9:0    0  249M  0 raid1 
  |-md0p1 259:0    0   49M  0 md    /raid/part1
  |-md0p2 259:1    0   50M  0 md    /raid/part2
  |-md0p3 259:2    0   49M  0 md    /raid/part3
  |-md0p4 259:3    0   50M  0 md    /raid/part4
  `-md0p5 259:4    0   49M  0 md    /raid/part5
sdd         8:48   0  250M  0 disk  
`-md0       9:0    0  249M  0 raid1 
  |-md0p1 259:0    0   49M  0 md    /raid/part1
  |-md0p2 259:1    0   50M  0 md    /raid/part2
  |-md0p3 259:2    0   49M  0 md    /raid/part3
  |-md0p4 259:3    0   50M  0 md    /raid/part4
  `-md0p5 259:4    0   49M  0 md    /raid/part5
sde         8:64   0  250M  0 disk  
`-md0       9:0    0  249M  0 raid1 
  |-md0p1 259:0    0   49M  0 md    /raid/part1
  |-md0p2 259:1    0   50M  0 md    /raid/part2
  |-md0p3 259:2    0   49M  0 md    /raid/part3
  |-md0p4 259:3    0   50M  0 md    /raid/part4
  `-md0p5 259:4    0   49M  0 md    /raid/part5
sdf         8:80   0  250M  0 disk  
`-md0       9:0    0  249M  0 raid1 
  |-md0p1 259:0    0   49M  0 md    /raid/part1
  |-md0p2 259:1    0   50M  0 md    /raid/part2
  |-md0p3 259:2    0   49M  0 md    /raid/part3
  |-md0p4 259:3    0   50M  0 md    /raid/part4
  `-md0p5 259:4    0   49M  0 md    /raid/part5
sdg         8:96   0  250M  0 disk  
`-md0       9:0    0  249M  0 raid1 
  |-md0p1 259:0    0   49M  0 md    /raid/part1
  |-md0p2 259:1    0   50M  0 md    /raid/part2
  |-md0p3 259:2    0   49M  0 md    /raid/part3
  |-md0p4 259:3    0   50M  0 md    /raid/part4
  `-md0p5 259:4    0   49M  0 md    /raid/part5
```
</p>
</details>
