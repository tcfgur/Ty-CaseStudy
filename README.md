# Ceph Disk Setup

## 1. Disk Kontrolü

Öncelikle mevcut diskleri listeleyerek yeni eklenen diski kontrol ettik:

```bash
lsblk
```

Çıktı:
```plaintext
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0 89.4M  1 loop /snap/lxd/31333
loop1                       7:1    0 63.9M  1 loop /snap/core20/2318
loop2                       7:2    0   87M  1 loop /snap/lxd/29351
loop3                       7:3    0 44.4M  1 loop /snap/snapd/23771
loop4                       7:4    0 38.8M  1 loop /snap/snapd/21759
loop6                       7:6    0 63.7M  1 loop /snap/core20/2496
sda                         8:0    0   80G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   78G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0   78G  0 lvm  /
sdb                         8:16   0    1T  0 disk
sr0                        11:0    1 1024M  0 rom
```

Yeni eklenen disk `sdb` olarak görünüyor.

## 2. Yeni Partition Oluşturma

Diskin içinde bir partition olup olmadığını kontrol ettik ve yeni bir partition oluşturduk:

```bash
sudo fdisk /dev/sdb
```

Fdisk komutları:

```plaintext
Command (m for help): n  # Yeni partition oluştur
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p  # Primary partition seç
Partition number (1-4, default 1):  # Varsayılan 1
First sector (2048-2147483647, default 2048):  # Varsayılanı kullan
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-2147483647, default 2147483647):  # Tüm diski kullan

Created a new partition 1 of type 'Linux' and of size 1024 GiB.

Command (m for help): w  # Değişiklikleri kaydet ve çık
```

Partition işlemi tamamlandıktan sonra tekrar `lsblk` ile kontrol ettik:

```bash
lsblk
```

Çıktı:
```plaintext
sdb                         8:16   0    1T  0 disk
└─sdb1                      8:17   0 1024G  0 part
```

## 3. Dosya Sistemi Oluşturma

Yeni oluşturulan partition'a ext4 dosya sistemi formatladık:

```bash
sudo mkfs.ext4 /dev/sdb1
```

Çıktı:
```plaintext
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 268435200 4k blocks and 67108864 inodes
Filesystem UUID: 7c7e1529-0eec-4161-a99e-1fecf4d88d8c
Writing superblocks and filesystem accounting information: done
```

## 4. Mount Noktası Oluşturma

Bir mount noktası oluşturduk:

```bash
sudo mkdir /ceph_disk
```

Bu adımlardan sonra disk `/ceph_disk` dizinine bağlanarak kullanılabilir hale getirilecektir.
