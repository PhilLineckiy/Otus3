# Otus3
## LVM
на имеющемся образе /dev/mapper/VolGroup00-LogVol00 38G 738M 37G 2% / сделать следующее:

-уменьшить том под / до 8G

- выделить том под /home

- выделить том под /var

- /var - сделать в mirror

- /home - сделать том для снэпшотов

- прописать монтирование в fstab

- попробовать с разными опциями и разными файловыми системами (на выбор)

- сгенерить файлы в /home/

- снять снэпшот

- удалить часть файлов

- восстановится со снэпшота

- залоггировать работу можно с помощью утилиты script

1. Уменьшить том под / до 8G.

Выбираем диск /dev/sdb как наиболее крупный. Создаем physical volume.

```
sudo pvcreate /dev/sdb
```

Создаем Volume group.

```
sudo vgcreate vg_new_root /dev/sdb
```
Создаем Logical volume.

```
sudo lvcreate -l+100%FREE -n lv0 vg_new_root
```

Создаем файловую систему на logical volume.

```
sudo mkfs.xfs /dev/vg_new_root/lv0
```

Монтируем lvm в /mnt

```
sudo mount /dev/vg_new_root/lv0 /mnt
```

Скачиваем утилиту xfsdump.

```
sudo yum -y install xfsdump
```

Этой командой скопируем все данные с / раздела в /mnt:

```
sudo xfsdump -J - /dev/VolGroup00/LogVol00 | sudo xfsrestore -J - /mnt
```
Затем переконфигурируем grub длā того, чтобý при старте перейти в новýю /
Сымитируем текущий root -> сделаем в него chroot и обновим grub:

```
for i in /proc/ /sys/ /dev/ /run/ /boot/; do sudo mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
```

Обновим образ initrd.


```
cd /boot ; for i in `ls initramfs-*img`; do sudo dracut -v $i `echo $i|sed "s/initramfs-//g;
s/.img//g"` --force; done
```

Изменим файл /etc/fstab для маунта раздела в /

```
/dev/vg_new_root/lv0  / ext4  defaults  0 0
```

Изменим файл /boot/grub2/grub.cfg для того, чтобы при загрузке был смонтирован нужный root. Изменим строку с содержанием root:

```
root=/dev/vg_new_root/lv0  rd.lvm.lv=/dev/vg_new_root/lv0
```
Перезагружаемся. Удаляем старый LVM и создаем новый размером 8GB:

```
sudo lvremove /dev/VolGroup00/LogVol00
sudo lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
```
Создаем файловую систему и монтируем LVM:

```
sudo mkfs.xfs /dev/VolGroup00/LogVol00
sudo mount /dev/VolGroup00/LogVol00 /mnt
```
2. Выделить том под /var и сделать его mirror.

Выбираем диск /dev/sdс и /dev/sdd как наиболее крупный. Создаем physical volume.

```
sudo pvcreate /dev/sdc /dev/sdd
```

Создаем Volume group.

```
sudo vgcreate var-lvm /dev/sdc /dev/sdd
```
Создаем Logical volume.

```
sudo lvcreate -L 950M -n -m1 var-lvm var-lvm
```

Создаем файловую систему на logical volume.

```
sudo mkfs.ext4 /dev/var-lvm/var-lvm
```

Создаем папку для маунта:

```
sudo mkdir /var-lvm
```

Монтируем:

```
sudo mount /dev/var-lvm/var-lvm /var-lvm
```

Копируем содержимое /var в /var-lvm:

```
sudo rsync -a /var /var-lvm
```

Демонтируем /var-lvm:

```
sudo umount /var-lvm
```

Монтируем новый var в каталог /var:

```
sudo mount /dev/var-lvm/var-lvm /var
```

Прописываем это в /etc/fstab:

```
/dev/var-lvm/var-lvm  /var  ext4  defaults  0 0
```

Исправляем ошибку с polkit. Меняем симлинк /var/run на новый:

```
cd /var
sudo rm run
sudo ln -s /run run
```

3. Выделяем LVM под /home с LVM для snapshot

```
sudo pvcreate /dev/sde
sudo vgcreate home-lvm /dev/sde
sudo lvcreate -l+75%FREE -n home-lvm home-lvm
sudo lvcreate -l+20%FREE -n home-lvm-snap /dev/home-lvm/home-lvm
sudo mkfs.ext4 /dev/home-lvm/home-lvm
sudo mkdir /home-lvm
sudo mount /dev/home-lvm/home-lvm /home-lvm
sudo rsync -a /home/ /home-lvm/
sudo umount /home-lvm
sudo mount /dev/home-lvm/home-lvm /home
```

Корректируем /etc/fstab:

```
/dev/home-lvm/home-lvm  /home ext4  defaults  0 0
```
4. Генерируем файл в /home/:

```
sudo fallocate -l 100M test.txt
```

5. Снимаем snapshot:

```
sudo lvcreate -L 100MB -s -n home_snap /dev/home-lvm/home-lvm-snap
```

6. Удаляем файл:

```
sudo  rm -f /home/test.txt
```

7. Восстанавливаемся из snapshot:

```
sudo umount /home
sudo lvconvert --merge /dev/home-lvm/home-lvm-snap
sudo mount /home
```
