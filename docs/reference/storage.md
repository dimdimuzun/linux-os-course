# Сховище

Докладніше: [модуль 13](../modules/13-storage.md).

```bash
lsblk
df -h
du -sh .
mount
findmnt
```

Перед операціями з дисками завжди перевіряйте пристрій і точку монтування.

## Перегляд

```bash
lsblk -f
sudo blkid
sudo parted -l
df -hT
df -i
findmnt -T /
ls -l /dev/disk/by-uuid
cat /proc/partitions
```

| Команда | Дія |
|---|---|
| `lsblk -f` | Диски, розділи, тип ФС, мітка, `UUID`, точки монтування |
| `blkid` | `UUID`, `LABEL`, `PARTUUID`, тип (`sudo`) |
| `parted -l`, `fdisk -l` | Таблиці розділів (MBR чи GPT) |
| `df -hT` | Заповнення ФС із типом; `df -i` — inode |
| `findmnt -T шлях` | ФС, якій належить шлях |

`df -h` і `lsblk` показують двійкові одиниці (КіБ, МіБ), `df -H` — десяткові.

## Рівні й іменування

| Рівень | Приклад |
|---|---|
| Пристрій | `/dev/sda`, `/dev/vda`, `/dev/nvme0n1` |
| Розділ | `/dev/sda2`, `/dev/nvme0n1p1` |
| Файлова система | `ext4`, `xfs`, `vfat` |
| Точка монтування | `/`, `/boot/efi`, `/mnt/data` |

| Ідентифікатор | Що ідентифікує |
|---|---|
| `UUID` | Файлову систему (змінюється при `mkfs`) |
| `LABEL` | Файлову систему (зручна назва) |
| `PARTUUID` | Розділ GPT |

**У `fstab` використовуйте `UUID`**: імена `/dev/sdX` можуть змінюватися.

| Таблиця розділів | Особливості |
|---|---|
| MBR | До 2 ТіБ, 4 основні розділи, BIOS |
| GPT | Практично необмежений розмір, 128 розділів, UEFI, резервна копія таблиці |

## Файлові системи

| Тип | Особливості |
|---|---|
| `ext4` | Журнальна, стандарт Ubuntu, права Linux |
| `xfs` | Великі файли, не зменшується |
| `btrfs` | Знімки, контрольні суми |
| `vfat`, `exfat`, `ntfs` | **Без прав Linux**: `uid=`, `gid=`, `umask=` під час монтування |
| `tmpfs` | У пам'яті, зникає після перезавантаження |
| `squashfs` | Стиснена, лише читання (snap) |

Inode — запис із метаданими файла; їх кількість обмежена: `No space left on device` може означати нестачу inode (`df -i`).

## Безпечна практика: loop-образ

Тренуйтеся на образі, а не на реальних дисках.

```bash
mkdir -p ~/storage-practice && cd ~/storage-practice
truncate -s 200M disk.img
LOOP="$(sudo losetup --find --show -P disk.img)"
[[ "$LOOP" == /dev/loop* ]] && echo "OK: $LOOP" || echo "СТОП"
sudo parted -s "$LOOP" mklabel gpt mkpart data ext4 1MiB 120MiB mkpart fat fat32 120MiB 100%
sudo mkfs.ext4 -L LAB_DATA "${LOOP}p1"
sudo mkfs.vfat -F 32 -n LAB_FAT "${LOOP}p2"
sudo mkdir -p /mnt/storage-practice/data
sudo mount "${LOOP}p1" /mnt/storage-practice/data
sudo chown "$(id -u):$(id -g)" /mnt/storage-practice/data
```

Прибирання:

```bash
sudo umount /mnt/storage-practice/data
sudo losetup -d "$LOOP"
sudo rmdir /mnt/storage-practice/data /mnt/storage-practice
rm -ri ~/storage-practice
```

!!! danger

    `mkfs`, `parted`, `fdisk`, `dd`, `wipefs` знищують дані. Перед ними перевіряйте `lsblk -f` і змінну `$LOOP`; не застосовуйте до реальних дисків без плану й знімка.

## Монтування

```bash
sudo mount ПРИСТРІЙ ТОЧКА
sudo mount -o ro ПРИСТРІЙ ТОЧКА
sudo mount -o remount,rw ТОЧКА
sudo umount ТОЧКА
findmnt -o TARGET,FSTYPE,OPTIONS ТОЧКА
```

| Параметр | Дія |
|---|---|
| `ro` / `rw` | Лише читання / читання й запис |
| `noexec` | Заборонити запуск програм (`bash script.sh` усе одно читає скрипт) |
| `nosuid`, `nodev` | Ігнорувати `setuid`/`setgid` і файли пристроїв |
| `nofail` | Не вважати помилкою відсутність пристрою під час завантаження |
| `remount` | Змінити параметри змонтованої ФС |
| `uid=`, `gid=`, `umask=` | Власник і права для `vfat` |

Після `mkfs.ext4` корінь ФС належить `root` — потрібен `chown`. `target is busy` — файлову систему використовує процес: `ls -l /proc/PID/cwd`, `fuser -vm ТОЧКА`; не користуйтеся `umount -f` без потреби.

## `/etc/fstab`

```text
# <пристрій>         <точка>    <тип>  <параметри>  <dump> <pass>
UUID=8c0a…           /          ext4   defaults     0      1
UUID=1A2B-3C4D       /boot/efi  vfat   umask=0077   0      1
/swap.img            none       swap   sw           0      0
```

`pass`: `1` — корінь, `2` — решта, `0` — не перевіряти.

```bash
findmnt --verify
findmnt --verify --tab-file файл
sudo mount -a
sudo systemctl daemon-reload
```

**Помилка в `fstab` може зупинити завантаження.** Резервна копія, `UUID`, `nofail`, перевірка `findmnt --verify` до перезавантаження; під час навчання перевіряйте тестовий файл, а не реальний.

## Swap

```bash
swapon --show
free -h
sudo fallocate -l 64M /swapfile-practice
sudo chmod 600 /swapfile-practice
sudo mkswap /swapfile-practice
sudo swapon /swapfile-practice
sudo swapoff /swapfile-practice
sudo rm /swapfile-practice
```

Файл swap має мати права `600` (інакше `mkswap` і `swapon` попереджають). `vm.swappiness` (зазвичай 60) — активність використання.

## Обслуговування

```bash
sudo e2fsck -f /dev/loop0p1
sudo tune2fs -l /dev/loop0p1 | head
sudo resize2fs /dev/loop0p1
```

`fsck`/`e2fsck`/`xfs_repair` — **лише для відмонтованих** файлових систем.

## Нестача місця

```bash
df -hT
df -i
sudo du -h -d 1 /var | sort -h | tail -n 5
find / -xdev -type f -size +100M 2>/dev/null
journalctl --disk-usage
sudo apt clean
sudo lsof +L1
```

| Симптом | Причина |
|---|---|
| `df` майже 100% | Місце заповнене; шукайте `du`, `find -size` |
| `df -i` 100% | Закінчились inode (мільйони дрібних файлів) |
| `df` більше, ніж `du` | Видалені, але відкриті файли (`lsof +L1`) |
| `Read-only file system` | Змонтовано `ro` або помилки ФС |

Типові «пожирачі»: журнали, кеш `apt`, старі ревізії snap, резервні копії.

## LVM, RAID, шифрування, копії

| Технологія | Суть |
|---|---|
| LVM (`pvs`, `vgs`, `lvs`) | PV → VG → LV: змінні за розміром томи |
| RAID (`mdadm`) | Надійність і швидкість; **не є резервною копією** |
| LUKS (`cryptsetup`) | Шифрування розділу |
| Копії | Правило 3-2-1; `tar`, `rsync`; копію перевіряють відновленням |
