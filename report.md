# Краткий отчёт: команды по снапшотам и ⭐ btrfs/zfs

ВМ **Irida** (Ubuntu 24.04.3, Proxmox `Hecate`). Итог: `/`=8G, `/home`=XFS LV,
`/var`=ext4 на RAID1-mirror, `/opt`=ZFS с L2ARC-кэшем.

## 1. Общая настройка LVM

```bash
# уменьшение / до 8G (offline: qemu-nbd на гипервизоре)
e2fsck -fy /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv 8G
lvreduce -L 8G -y /dev/ubuntu-vg/ubuntu-lv

# /home (XFS, том для снапшотов)
lvcreate -n home -L 10G ubuntu-vg
mkfs.xfs -f -L home /dev/ubuntu-vg/home

# /var — зеркало
pvcreate /dev/sdb /dev/sdc
vgcreate vg_var /dev/sdb /dev/sdc
lvcreate -m1 -L 4G -n var vg_var
mkfs.ext4 -L var /dev/vg_var/var
```

## 2. Снапшоты LVM (`/home`) — создание и восстановление

```bash
# 1) сгенерировать файлы
mkdir -p /home/hwfiles && for i in $(seq 1 20); do echo "content $i" > /home/hwfiles/file$i; done

# 2) снять снапшот
lvcreate -L 200M -s -n home_snap /dev/ubuntu-vg/home

# 3) удалить часть файлов
rm -f /home/hwfiles/file{11..20}

# 4) восстановиться
#    вариант A — смонтировать снапшот и скопировать файлы обратно
mount -o ro,nouuid /dev/ubuntu-vg/home_snap /mnt/snap
cp -a /mnt/snap/hwfiles/. /home/hwfiles/
umount /mnt/snap

#    вариант B — откат всего тома на снапшот (нужно отмонтировать /home)
umount /home
lvconvert --merge /dev/ubuntu-vg/home_snap
mount /dev/ubuntu-vg/home /home

# вспомогательные команды
lvs -a -o lv_name,vg_name,lv_size,lv_attr,origin    # список LV и снапшотов
lvremove -f /dev/ubuntu-vg/home_snap                # удалить снапшот
```

Лог выполнения: `logs/lvm_home_snap.log`.

## 3. ⭐ ZFS: `/opt` с кэшем и снапшотами

```bash
# разметка диска sdd: данные + L2ARC-кэш
sgdisk -n1:0:+6144M -t1:bf00 -c1:zfspool  /dev/sdd
sgdisk -n2:0:0      -t2:bf00 -c2:zfscache /dev/sdd

# пул с кэш-устройством
zpool create -o ashift=12 -O atime=off -O compression=lz4 -O xattr=sa \
      optpool /dev/sdd1 cache /dev/sdd2
zfs create optpool/opt
zfs set mountpoint=/opt optpool/opt
zfs set recordsize=128k optpool/opt

# снапшот и откат
zfs snapshot optpool/opt@snap1
rm -f /opt/appdata/important.txt
zfs rollback optpool/opt@snap1

# диагностика
zpool status
zpool list
zfs list -t snapshot
zfs get compression,atime,recordsize optpool/opt
```

Лог выполнения: `logs/lvm_opt_zfs.log`.

## 4. Опции монтирования (fstab)

Проверены разные наборы опций (лог `logs/lvm_fs_options.log`):

| Точка | ФС | Опции |
|---|---|---|
| `/` | ext4 | `defaults,noatime` |
| `/home` | xfs | `defaults,noatime` (пробовали `nosuid,nodev,noatime`) |
| `/var` | ext4 | `defaults,noatime,commit=60` |
| `/opt` | zfs | `mountpoint=/opt`, `atime=off`, `compression=lz4` |

## 5. Примечание об уменьшении `/`

ext4 не поддерживает уменьшение на смонтированной ФС, поэтому корневой том
уменьшался offline (диски подключены на гипервизоре через `qemu-nbd`, команды
`e2fsck`/`resize2fs`/`lvreduce` из раздела 1). Альтернативный способ —
через временный корень и перезагрузку (см. README методички).
