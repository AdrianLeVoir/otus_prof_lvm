# Домашнее задание: Файловые системы и LVM

## 1. Стенд

| Параметр | Значение |
|---|---|
| Гипервизор | Proxmox VE 9.1.6 |
| ВМ | 4 vCPU / 6 GB RAM |
| ОС | Ubuntu 24.04.3 LTS, ядро 6.8.0-142-generic |

Диски ВМ:

```
sda   40G  ├─sda1 1M (BIOS boot)
           ├─sda2 2G ext4  /boot
           └─sda3 38G LVM  (ubuntu-vg)
sdb    5G  свободный  → /var (mirror)
sdc    5G  свободный  → /var (mirror)
sdd    8G  свободный  → ZFS /opt (data + cache)
```

## 2. Итоговая схема

| Точка монтирования | Устройство | Размер | ФС | Примечание |
|---|---|---|---|---|
| `/` | `/dev/ubuntu-vg/ubuntu-lv` | **8.00G** | ext4 | уменьшено с 19G |
| `/boot` | `/dev/sda2` | 2G | ext4 | без изменений |
| `/home` | `/dev/ubuntu-vg/home` | 10G | **XFS** | LV для снапшотов |
| `/var` | `/dev/vg_var/var` | 4G | ext4 | **RAID1 mirror** (sdb+sdc) |
| `/opt` | `optpool/opt` | 5.3G | **ZFS** | L2ARC-кэш + снапшоты |

Состояние LVM:

```
  PV         VG        PSize    PFree
  /dev/sda3  ubuntu-vg <38.00g  19.80g
  /dev/sdb   vg_var     <5.00g  1016m
  /dev/sdc   vg_var     <5.00g  1016m

  LV        VG        LSize   Attr       Origin
  ubuntu-lv ubuntu-vg   8.00g -wi-ao----
  home      ubuntu-vg  10.00g owi-aos---
  home_snap ubuntu-vg 200.00m swi-a-s--- home
  var       vg_var      4.00g rwi-aor---      (var_rimage_0, var_rimage_1)
```

## 3. Ход выполнения

### 3.1 Уменьшение тома `/` до 8G

ext4 **нельзя** уменьшать на смонтированной ФС, поэтому том уменьшался в offline-режиме:
диски ВМ подключены на гипервизоре через `qemu-nbd`, VG активирована вне ВМ.

```bash
# на гипервизоре
qm shutdown 103
modprobe nbd max_part=16
qemu-nbd -c /dev/nbd0 -f raw /mnt/purple/images/103/vm-103-disk-0.raw
vgscan && vgchange -ay ubuntu-vg

# проверка и уменьшение ФС
e2fsck -fy /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv 8G
lvreduce -L 8G -y /dev/ubuntu-vg/ubuntu-lv
e2fsck -fy /dev/ubuntu-vg/ubuntu-lv
```

Результат: `ubuntu-lv` уменьшен с `19.00g` до `8.00g`, fs — 7.8G, ошибок нет.

> Альтернатива из методички (без LiveCD): создать временный `vg_root/lv_root` на
> свободном диске, `rsync -avxHAX / /mnt/`, chroot + `grub-mkconfig` +
> `update-initramfs -u`, перезагрузиться, удалить старый LV, создать новый на 8G
> и скопировать данные обратно.

### 3.2 Том под `/home` (LV + том для снапшотов)

```bash
lvcreate -n home -L 10G ubuntu-vg
mkfs.xfs -f -L home /dev/ubuntu-vg/home      # XFS — "другая ФС"
mount /dev/ubuntu-vg/home /mnt
rsync -aHAX /home/ /mnt/                     # перенос данных пользователя
```
LV `/home` оставлен как **том для снапшотов** (в VG `ubuntu-vg` свободно ~19.8G под COW).

### 3.3 Том под `/var` — в зеркале (mirror)

```bash
pvcreate /dev/sdb /dev/sdc
vgcreate vg_var /dev/sdb /dev/sdc
lvcreate -m1 -L 4G -n var vg_var             # RAID1 (mirror)
mkfs.ext4 -L var /dev/vg_var/var
rsync -aHAX /var/ /mnt_var/
```

`lvs`: `var vg_var 4.00g rwi-aor--- var_rimage_0(0),var_rimage_1(0)` — зеркало из двух
сегментов на `sdb` и `sdc` (+ `var_rmeta_0/1`).

### 3.4 Монтирование в fstab (разные опции и разные ФС)

`/etc/fstab` после правок:

```
UUID=2edc6498-69b1-48fa-b062-f2f3dc38a486   /       ext4   defaults,noatime            0 1
UUID=4265b2ea-2632-4645-aeae-253c596d2b93   /boot   ext4   defaults                    0 2
UUID=5482bf3b-c967-4c17-806d-705aa846a5f7   /home   xfs    defaults,noatime            0 2
UUID=52e33189-d842-4868-be44-8e77ef2e4084   /var    ext4   defaults,noatime,commit=60  0 2
/swap.img                                   none    swap   sw                          0 0
```

Разные ФС: **ext4** (`/`, `/var`, `/boot`), **XFS** (`/home`), **ZFS** (`/opt`).
Разные опции: `noatime`, `commit=60`, для XFS — свои параметры; см. лог
`logs/lvm_fs_options.log` (пробное перемонтирование `/home` с `nosuid,nodev,noatime`,
`/var` с `commit=60`). Для `/opt` (ZFS) монтирование выполняет
`zfs-mount.service` по свойству `mountpoint=/opt`.

### 3.5 Работа со снапшотами `/home`

```bash
mkdir -p /home/hwfiles && for i in $(seq 1 20); do echo "content $i" > /home/hwfiles/file$i; done
lvcreate -L 200M -s -n home_snap /dev/ubuntu-vg/home        # снапшот
rm -f /home/hwfiles/file{11..20}                            # удалили часть файлов
mount -o ro,nouuid /dev/ubuntu-vg/home_snap /mnt/snap       # монтируем снапшот
cp -a /mnt/snap/hwfiles/. /home/hwfiles/                    # восстанавливаем
umount /mnt/snap
```

Файлы `file1..file20` успешно восстановлены. Полный вывод — `logs/lvm_home_snap.log`.
Альтернативный способ восстановления: `lvconvert --merge /dev/ubuntu-vg/home_snap`.

### 3.6 ⭐ `/opt` на ZFS — с кэшем и снапшотами

```bash
# sdd: sdd1 (6G, данные) + sdd2 (2G, L2ARC-кэш)
sgdisk -n1:0:+6144M -t1:bf00 -c1:zfspool /dev/sdd
sgdisk -n2:0:0       -t2:bf00 -c2:zfscache /dev/sdd

zpool create -o ashift=12 -O atime=off -O compression=lz4 -O xattr=sa \
      optpool /dev/sdd1 cache /dev/sdd2
zfs create optpool/opt
zfs set mountpoint=/opt optpool/opt
zfs set recordsize=128k optpool/opt

zfs snapshot optpool/opt@snap1        # снапшот
rm /opt/appdata/important.txt         # удаление данных
zfs rollback optpool/opt@snap1        # восстановление
```

`zpool status` — `ONLINE`, устройства `sdd1` (данные) и `sdd2` (cache).
Полный вывод — `logs/lvm_opt_zfs.log`.

## 4. Логирование через `script`

Все ключевые шаги писались утилитой `script` (файлы в каталоге `logs/`):

| Файл | Содержимое |
|---|---|
| `logs/lvm_final_state.log` | итоговое состояние (`lsblk`, `pvs`, `vgs`, `lvs`, `df`, `fstab`, `zpool`) |
| `logs/lvm_home_snap.log` | генерация файлов, снапшот, удаление, восстановление `/home` |
| `logs/lvm_opt_zfs.log` | ZFS `/opt`: данные, снапшот, rollback |
| `logs/lvm_fs_options.log` | пробные опции монтирования |

## 5. Проверка после перезагрузки

После `reboot` все тома поднялись автоматически (fstab + `zfs-import`/`zfs-mount`),
`systemctl --failed` — пусто:

```
/        ext4  7.8G (8G LV)
/var     ext4  3.9G  mirror on sdb+sdc
/home    xfs   9.9G  + home_snap
/opt     zfs   5.4G  optpool (cache: sdd2)
```
