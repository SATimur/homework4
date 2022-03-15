# Домашнее задание по ZFS

1. Определить алгоритм с наилучшим сжатием
2. Определить настройки пула
3. Работа со снапшотами

#### Подготовка Vagrantfile и запуск тестового стенда

> Редактируем Vagrantfile следующим образом
```
    MACHINES = {
         :zfs => {
               :box_name => "centos/7",
               :box_version => "2004.01",
               :disks => {
         :sata1 => {
               :dfile => './sata1.vdi',
               :size => 512,
               :port => 1
         },
         :sata2 => {
               :dfile => './sata2.vdi',
               :size => 512, # Megabytes
               :port => 2
         },
         :sata3 => {
               :dfile => './sata3.vdi',
               :size => 512,
               :port => 3
         },
         :sata4 => {
               :dfile => './sata4.vdi',
               :size => 512,
               :port => 4
         },
         :sata5 => {
               :dfile => './sata5.vdi',
               :size => 512,
               :port => 5
         },
         :sata6 => {
               :dfile => './sata6.vdi',
               :size => 512,
               :port => 6
         },
         :sata7 => {
               :dfile => './sata7.vdi',
               :size => 512,
               :port => 7
         },
         :sata8 => {
               :dfile => './sata8.vdi',
               :size => 512,
               :port => 8
         },
     }
   },
 }
```
```
   box.vm.provision "shell", inline: <<-SHELL
   #install zfs repo
   yum install -y
   http://download.zfsonlinux.org/epel/zfs-release.el7_8.noarch.rpm
   #import gpg key
   rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-zfsonlinux
   #install DKMS style packages for correct work ZFS
   yum install -y epel-release kernel-devel zfs
   #change ZFS repo
   yum-config-manager --disable zfs
   yum-config-manager --enable zfs-kmod
   yum install -y zfs
   #Add kernel module zfs
   modprobe zfs
   #install wget
   yum install -y wget
SHELL
```

Далее запускаем Vagrantfile командой
>vagrant up
>vagrant ssh

и выводим команду lsblk:
```
   [vagrant@zfs ~]$ lsblk
   NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
   sda      8:0    0   40G  0 disk
   `-sda1   8:1    0   40G  0 part /
   sdb      8:16   0  512M  0 disk
   sdc      8:32   0  512M  0 disk
   sdd      8:48   0  512M  0 disk
   sde      8:64   0  512M  0 disk
   sdf      8:80   0  512M  0 disk
   sdg      8:96   0  512M  0 disk
   sdh      8:112  0  512M  0 disk
   sdi      8:128  0  512M  0 disk
```
Переключаемся на root, т.к. все остальные манипуляции потребуют прав супер пользователя
>sudo su

### Определение алгоритма с наилучшим сжатием

1. Создаём пул из двух дисков в режиме RAID 1, но перед этис установим ZFS
```  
    yum install update;
    yum install http://download.zfsonlinux.org/epel/zfs-release.el7_3.noarch.rpm
    yum install zfs
    modprobe zfs
```
2. Создаём 4-е пула по два диска в режиме RAID 1
```
   [root@zfs vagrant]# modprobe zfs
   [root@zfs vagrant]# zpool create otus1 mirror /dev/sdb /dev/sdc
   [root@zfs vagrant]# zpool create otus2 mirror /dev/sdd /dev/sde
   [root@zfs vagrant]# zpool create otus3 mirror /dev/sdf /dev/sdg
   [root@zfs vagrant]# zpool create otus4 mirror /dev/sdh /dev/sdi
   [root@zfs vagrant]# zpool list
   NAME    SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
   otus1   496M   272K   496M         -     0%     0%  1.00x  ONLINE  -
   otus2   496M   272K   496M         -     0%     0%  1.00x  ONLINE  -
   otus3   496M   272K   496M         -     0%     0%  1.00x  ONLINE  -
   otus4   496M   272K   496M         -     0%     0%  1.00x  ONLINE  -
```
3. Добавим разные алгоритмы сжатия
```
   [root@zfs vagrant]# zfs set compression=lzjb otus1
   [root@zfs vagrant]# zfs set compression=lz4 otus2
   [root@zfs vagrant]# zfs set compression=gzip-9 otus3
   [root@zfs vagrant]# zfs set compression=zle otus4
   [root@zfs vagrant]# zfs get all | grep compression
   otus1  compression           lzjb                   local
   otus2  compression           lz4                    local
   otus3  compression           gzip-9                 local
   otus4  compression           zle                    local
```
4. Зальем файлов во все пулы
```
   [root@zfs vagrant]# for i in {1..20}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
   --2022-03-10 07:18:40--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
   Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
   Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
   HTTP request sent, awaiting response... 200 OK
   Length: 40792737 (39M) [text/plain]
   Saving to: '/otus1/pg2600.converter.log'

   100%[===========================================================================>] 40,792,737  14.2MB/s   in 2.7s
  
   Все вставлять не буду, выхлоп большой получился
```
```
   [root@zfs vagrant]# zfs list
   NAME    USED  AVAIL  REFER  MOUNTPOINT
   otus1  21.6M   346M  21.5M  /otus1
   otus2  17.6M   350M  17.6M  /otus2
   otus3  10.8M   357M  10.7M  /otus3  -- самое эффективное сжатие
   otus4  39.0M   329M  39.0M  /otus4
```
```
   [root@zfs vagrant]#  zfs get all | grep compressratio | grep -v ref
   otus1  compressratio         1.81x                  -
   otus2  compressratio         2.22x                  -
   otus3  compressratio         3.64x                  -
   otus4  compressratio         1.00x                  -
```
5. Скачиваем архив в домашний каталог:
```
   [root@zfs vagrant]# wget -O archive.tar.gz --no-check-certificate 'https://drive.google.com/u/0/uc?  id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download'
   --2022-03-10 10:12:49--  https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download
   Resolving drive.google.com (drive.google.com)... 172.217.168.238, 2a00:1450:400e:80d::200e
   Connecting to drive.google.com (drive.google.com)|172.217.168.238|:443... connected.
   HTTP request sent, awaiting response... 302 Found
   Location: https://drive.google.com/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download [following]
   --2022-03-10 10:12:49--  https://drive.google.com/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download
   Reusing existing connection to drive.google.com:443.
   HTTP request sent, awaiting response... 303 See Other
   Location: https://doc-0c-bo-   docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/fdmjl82hnpl3j8qtbq7tqat7i6peccio/1646907150000/16189157874053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg?e=download [following]
   Warning: wildcards not supported in HTTP.
   --2022-03-10 10:12:54--  https://doc-0c-bo-    docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/fdmjl82hnpl3j8qtbq7tqat7i6peccio/1646907150000/161891578 74053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg?e=download
   Resolving doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)... 142.250.179.193,  2a00:1450:400e:801::2001
   Connecting to doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)|142.250.179.193|:443... connected.
   HTTP request sent, awaiting response... 200 OK
   Length: 7275140 (6.9M) [application/x-gzip]
   Saving to: 'archive.tar.gz'

   100%[===========================================================================>] 7,275,140   28.2MB/s   in 0.2s

   2022-03-10 10:12:55 (28.2 MB/s) - 'archive.tar.gz' saved [7275140/7275140]
```
6. Разархивируем архив
```
   [root@zfs vagrant]# tar -xzvf archive.tar.gz
   zpoolexport/
   zpoolexport/filea
   zpoolexport/fileb
```
7. Проверяем на возможность импорта данного каталога
```
   [root@zfs vagrant]# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

        otus                                 ONLINE
          mirror-0                           ONLINE
            /home/vagrant/zpoolexport/filea  ONLINE
            /home/vagrant/zpoolexport/fileb  ONLINE
```
8. Делаем импорт данного пула к нам в ОС:
```
   [root@zfs vagrant]# zpool import -d zpoolexport/ otus
   [root@zfs vagrant]# zpool status
      pool: otus
     state: ONLINE
      scan: none requested
   config:

            NAME                                 STATE     READ WRITE CKSUM
           otus                                 ONLINE       0     0     0
             mirror-0                           ONLINE       0     0     0
               /home/vagrant/zpoolexport/filea  ONLINE       0     0     0
               /home/vagrant/zpoolexport/fileb  ONLINE       0     0     0

   errors: No known data errors

      pool: otus1
     state: ONLINE
      scan: none requested
   config:

            NAME        STATE     READ WRITE CKSUM
          otus1       ONLINE       0     0     0
            mirror-0  ONLINE       0     0     0
              sdb     ONLINE       0     0     0
              sdc     ONLINE       0     0     0

   errors: No known data errors

      pool: otus2
     state: ONLINE
      scan: none requested
   config:

            NAME        STATE     READ WRITE CKSUM
          otus2       ONLINE       0     0     0
            mirror-0  ONLINE       0     0     0
              sdf     ONLINE       0     0     0
              sdg     ONLINE       0     0     0

   errors: No known data errors

      pool: otus3
     state: ONLINE
      scan: none requested
   config:

            NAME        STATE     READ WRITE CKSUM
           otus3       ONLINE       0     0     0
             mirror-0  ONLINE       0     0     0
               sdd     ONLINE       0     0     0
               sde     ONLINE       0     0     0

   errors: No known data errors

      pool: otus4
     state: ONLINE
      scan: none requested
   config:

            NAME        STATE     READ WRITE CKSUM
          otus4       ONLINE       0     0     0
            mirror-0  ONLINE       0     0     0
              sdh     ONLINE       0     0     0
              sdi     ONLINE       0     0     0

   errors: No known data errors 
```
9. Параметры пула:
```
   [root@zfs vagrant]# zpool get all otus
   NAME  PROPERTY                       VALUE                          SOURCE
   otus  size                           480M                           -
   otus  capacity                       0%                             -
   otus  altroot                        -                              default
   otus  health                         ONLINE                         -
   otus  guid                           6554193320433390805            -
   otus  version                        -                              default
   otus  bootfs                         -                              default
   otus  delegation                     on                             default
   otus  autoreplace                    off                            default
   otus  cachefile                      -                              default
   otus  failmode                       wait                           default
   otus  listsnapshots                  off                            default
   otus  autoexpand                     off                            default
   otus  dedupditto                     0                              default
   otus  dedupratio                     1.00x                          -
   otus  free                           478M                           -
   otus  allocated                      2.09M                          -
   otus  readonly                       off                            -
   otus  ashift                         0                              default 
   otus  comment                        -                              default
   otus  expandsize                     -                              -
   otus  freeing                        0                              -
   otus  fragmentation                  0%                             -
   otus  leaked                         0                              -
   otus  multihost                      off                            default
   otus  checkpoint                     -                              -
   otus  load_guid                      17316409608158661891           -
   otus  autotrim                       off                            default
   otus  feature@async_destroy          enabled                        local 
   otus  feature@empty_bpobj            active                         local
   otus  feature@lz4_compress           active                         local
   otus  feature@multi_vdev_crash_dump  enabled                        local
   otus  feature@spacemap_histogram     active                         local
   otus  feature@enabled_txg            active                         local
   otus  feature@hole_birth             active                         local
   otus  feature@extensible_dataset     active                         local
   otus  feature@embedded_data          active                         local
   otus  feature@bookmarks              enabled                        local
   otus  feature@filesystem_limits      enabled                        local
   otus  feature@large_blocks           enabled                        local
   otus  feature@large_dnode            enabled                        local
   otus  feature@sha512                 enabled                        local
   otus  feature@skein                  enabled                        local
   otus  feature@edonr                  enabled                        local
   otus  feature@userobj_accounting     active                         local
   otus  feature@encryption             enabled                        local
   otus  feature@project_quota          active                         local
   otus  feature@device_removal         enabled                        local
   otus  feature@obsolete_counts        enabled                        local
   otus  feature@zpool_checkpoint       enabled                        local
   otus  feature@spacemap_v2            active                         local
   otus  feature@allocation_classes     enabled                        local
   otus  feature@resilver_defer         enabled                        local
   otus  feature@bookmark_v2            enabled                        local
```
10. Параметры файловой системы:
```
   [root@zfs vagrant]# zfs get all otus
   NAME  PROPERTY              VALUE                  SOURCE
   otus  type                  filesystem             -
   otus  creation              Fri May 15  4:00 2020  -
   otus  used                  2.04M                  -
   otus  available             350M                   -
   otus  referenced            24K                    -
   otus  compressratio         1.00x                  -
   otus  mounted               yes                    -
   otus  quota                 none                   default
   otus  reservation           none                   default
   otus  recordsize            128K                   local
   otus  mountpoint            /otus                  default
   otus  sharenfs              off                    default
   otus  checksum              sha256                 local
   otus  compression           zle                    local
   otus  atime                 on                     default
   otus  devices               on                     default
   otus  exec                  on                     default
   otus  setuid                on                     default
   otus  readonly              off                    default
   otus  zoned                 off                    default
   otus  snapdir               hidden                 default
   otus  aclinherit            restricted             default
   otus  createtxg             1                      -
   otus  canmount              on                     default
   otus  xattr                 on                     default
   otus  copies                1                      default
   otus  version               5                      - 
   otus  utf8only              off                    -
   otus  normalization         none                   -
   otus  casesensitivity       sensitive              -
   otus  vscan                 off                    default
   otus  nbmand                off                    default
   otus  sharesmb              off                    default
   otus  refquota              none                   default
   otus  refreservation        none                   default
   otus  guid                  14592242904030363272   -
   otus  primarycache          all                    default
   otus  secondarycache        all                    default
   otus  usedbysnapshots       0B                     -
   otus  usedbydataset         24K                    -
   otus  usedbychildren        2.01M                  -
   otus  usedbyrefreservation  0B                     -
   otus  logbias               latency                default
   otus  objsetid              54                     -
   otus  dedup                 off                    default
   otus  mlslabel              none                   default
   otus  sync                  standard               default
   otus  dnodesize             legacy                 default
   otus  refcompressratio      1.00x                  -
   otus  written               24K                    -
   otus  logicalused           1020K                  -
   otus  logicalreferenced     12K                    -
   otus  volmode               default                default
   otus  filesystem_limit      none                   default
   otus  snapshot_limit        none                   default
   otus  filesystem_count      none                   default
   otus  snapshot_count        none                   default
   otus  snapdev               hidden                 default
   otus  acltype               off                    default
   otus  context               none                   default
   otus  fscontext             none                   default
   otus  defcontext            none                   default
   otus  rootcontext           none                   default
   otus  relatime              off                    default
   otus  redundant_metadata    all                    default
   otus  overlay               off                    default
   otus  encryption            off                    default
   otus  keylocation           none                   default
   otus  keyformat             none                   default
   otus  pbkdf2iters           0                      default
   otus  special_small_blocks  0                      default
```
### Работа со снапшотом, поиск сообщения от преподавателя

1. Скачиваем файл
```
   [root@zfs vagrant]# wget -O otus_task2.file --no-check-certificate 'https://drive.google.com/u/0/uc?       id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download'
   --2022-03-15 11:54:27--  https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download
   Resolving drive.google.com (drive.google.com)... 172.217.168.238, 2a00:1450:400e:80d::200e
   Connecting to drive.google.com (drive.google.com)|172.217.168.238|:443... connected.
   HTTP request sent, awaiting response... 302 Found
   Location: https://drive.google.com/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download [following]
   --2022-03-15 11:54:27--  https://drive.google.com/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download
   Reusing existing connection to drive.google.com:443.
   HTTP request sent, awaiting response... 303 See Other
   Location: https://doc-00-bo-   docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/dbn9h10te75q55m966qbi436eup35mf4/1647345225000/16189157874053420 687/*/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG?e=download [following]
   Warning: wildcards not supported in HTTP.
   --2022-03-15 11:54:30--  https://doc-00-bo- docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/dbn9h10te75q55m966qbi436eup35mf4/1647345225000/16189157874053420 687/*/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG?e=download
   Resolving doc-00-bo-docs.googleusercontent.com (doc-00-bo-docs.googleusercontent.com)... 142.250.179.193, 2a00:1450:400e:803::2001
   Connecting to doc-00-bo-docs.googleusercontent.com (doc-00-bo-docs.googleusercontent.com)|142.250.179.193|:443... connected.
   HTTP request sent, awaiting response... 200 OK
   Length: 5432736 (5.2M) [application/octet-stream]
   Saving to: 'otus_task2.file'

   100%[=================================================================>] 5,432,736   19.5MB/s   in 0.3s

   2022-03-15 11:54:31 (19.5 MB/s) - 'otus_task2.file' saved [5432736/5432736] 
```
2. Восстановим файловую систему из снапшота:
```
   [root@zfs vagrant]# zfs receive otus/test@today <otus_task2.file
```
3. Смотрим содержимое найденного файла:
```
   [root@zfs vagrant]# find /otus/test -name "secret_message"
   /otus/test/task1/file_mess/secret_message
   [root@zfs vagrant]# cat /otus/test/task1/file_mess/secret_message
   https://github.com/sindresorhus/awesome
```
![Screenshot_3](https://user-images.githubusercontent.com/98743534/158374106-d73e717e-9112-4bf9-8828-bfa8b19b56a0.png)



