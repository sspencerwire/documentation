---
title: 2 Налаштування ZFS
author: Steven Spencer
contributors: Ezequiel Bruni, Ganna Zhyrnova
tested_with: 9.4
tags:
  - incus
  - enterprise
  - incus zfs
---

# Розділ 2: Налаштування ZFS

У цьому розділі ви повинні бути користувачем root або вміти `sudo`, щоб стати root.

Якщо ви вже інсталювали ZFS, цей розділ проведе вас через налаштування ZFS.

## Увімкнення ZFS і налаштування пулу

Спочатку введіть цю команду:

```bash
/sbin/modprobe zfs
```

Якщо помилок немає, він повернеться до підказки та нічого не повторить. Якщо ви отримали повідомлення про помилку, ви можете зупинитися зараз і почати усунення несправностей. Знову переконайтеся, що безпечне завантаження вимкнено. Це буде найімовірнішим винуватцем.

Далі вам потрібно перевірити диски в нашій системі, знайти операційну систему та визначити, що доступно для пулу ZFS. Ви зробите це за допомогою `lsblk`:

```bash
lsblk
```

Що має повернути щось на зразок цього (ваша система буде іншою!):

```bash
AME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0    7:0    0  32.3M  1 loop /var/lib/snapd/snap/snapd/11588
loop1    7:1    0  55.5M  1 loop /var/lib/snapd/snap/core18/1997
loop2    7:2    0  68.8M  1 loop /var/lib/snapd/snap/lxd/20037
sda      8:0    0 119.2G  0 disk
├─sda1   8:1    0   600M  0 part /boot/efi
├─sda2   8:2    0     1G  0 part /boot
├─sda3   8:3    0  11.9G  0 part [SWAP]
├─sda4   8:4    0     2G  0 part /home
└─sda5   8:5    0 103.7G  0 part /
sdb      8:16   0 119.2G  0 disk
├─sdb1   8:17   0 119.2G  0 part
└─sdb9   8:25   0     8M  0 part
sdc      8:32   0 149.1G  0 disk
└─sdc1   8:33   0 149.1G  0 part
```

Цей список показує, що операційна система використовує _/dev/sda_. Ви будете використовувати _/dev/sdb_ для нашого zpool. Зверніть увагу: якщо у вас багато доступних жорстких дисків, ви можете розглянути можливість використання raidz (програмний рейд спеціально для ZFS).

Це виходить за рамки цього документа, але розглядається для виробництва. Він пропонує кращу продуктивність і резервування. Наразі створіть свій пул на одному пристрої, який ви визначили:

```bash
zpool create storage /dev/sdb
```

Це говорить про створення пулу під назвою «сховище», тобто ZFS на пристрої _/dev/sdb_.

Після створення пулу знову перезавантажте сервер.
