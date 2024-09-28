---
title: 9 Сервер snapshot
author: Steven Spencer
contributors: Ezequiel Bruni, Ganna Zhyrnova
tested_with: 9.4
tags:
  - incus
  - enterprise
  - incus snapshot server
---

У цій главі використовується комбінація привілейованого (root) користувача та непривілейованого (incusadmin) користувача на основі завдань, які ви виконуєте.

Як зазначалося на початку, сервер моментальних знімків Incus повинен усіма можливими способами віддзеркалювати робочий сервер. Причина полягає в тому, що вам може знадобитися перенести його на робочий стан у разі апаратної несправності, а наявність не лише резервних копій, але й швидкого способу запуску робочих контейнерів зводить до мінімуму панічні телефонні дзвінки та текстові повідомлення системних адміністраторів. ЦЕ ЗАВЖДИ ДОБРЕ!

Таким чином, процес створення snapshot сервера точно схожий на робочий сервер. Щоб повністю імітувати налаштування робочого сервера, повторіть **Розділи 1-4** на сервері знімків і поверніться до цього місця після завершення.

Ви повернулись!! Вітаємо, це має означати, що ви успішно завершили базову інсталяцію snapshot сервера.

## Налаштування зв’язку між основним і snapshot сервером

Перш ніж продовжити, потрібно трохи прибрати. По-перше, якщо ви працюєте у робочому середовищі, ви, ймовірно, маєте доступ до DNS-сервера, який можна використовувати для налаштування IP-адреси для розпізнавання імен.

У вашій лабораторії ви не маєте такої розкоші. Можливо, у вас працює той самий сценарій. З цієї причини ви додасте IP-адреси та імена серверів до файлу `/etc/hosts` на основному сервері та серверах знімків. Ви повинні зробити це як ваш root (або _sudo_) користувач.

У вашій лабораторії основний сервер Incus працює на 192.168.1.106, а знімок сервера Incus працює на 192.168.1.141. SSH на кожному сервері та додайте наступне до файлу `/etc/hosts`:

```bash
192.168.1.106 incus-primary
192.168.1.141 incus-snapshot
```

Далі вам потрібно дозволити весь трафік між двома серверами. Для цього потрібно змінити правила `firewalld`. Спочатку на сервері incus-primary додайте цей рядок:

```bash
firewall-cmd zone=trusted add-source=192.168.1.141 --permanent
```

а на сервері знімків додайте це правило:

```bash
firewall-cmd zone=trusted add-source=192.168.1.106 --permanent
```

потім перезавантажте:

```bash
firewall-cmd reload
```

Далі, як непривілейований користувач (incusadmin), ви повинні встановити довірчі відносини між двома машинами. Це робиться за допомогою наступного виконання на incus-primary:

```bash
incus remote add incus-snapshot
```

Це відображає сертифікат, який потрібно прийняти. Прийміть його, і з’явиться запит на введення пароля. Це «пароль довіри», який ви встановили під час кроку ініціалізації Incus. Сподіваємось, ви надійно зберігаєте всі ці паролі. Коли ви введете пароль, ви отримаєте це:

```bash
Client certificate stored at server:  incus-snapshot
```

Не завадить мати це і в зворотному порядку. Наприклад, довірчі відносини можна встановити на сервері incus-snapshot. За потреби сервер incus-snapshot може надсилати знімки назад на основний сервер incus-primary. Повторіть кроки та замініть "incus-primary" на "incus-snapshot."

### Перенесення вашого першого snapshot

Перш ніж ви зможете перенести свій перший знімок, вам потрібно створити будь-які профілі на знімку incus-snapshot, який ви створили на incus-primary. В даному випадку це профіль "macvlan".

Вам потрібно буде створити це для incus-snapshot. Поверніться до [розділу 6](06-profiles.md) і створіть профіль "macvlan" у incus-snapshot, якщо потрібно. Якщо ваші два сервери мають однакові назви батьківського інтерфейсу (наприклад, "enp3s0"), ви можете скопіювати профіль "macvlan" у incus-snapshot, не створюючи його повторно:

```bash
incus profile copy macvlan incus-snapshot
```

Коли всі зв’язки та профілі налаштовано, наступним кроком є ​​надсилання знімка з incus-primary до incus-snapshot. Якщо ви точно слідкували за цим, ви, ймовірно, видалили всі свої знімки. Створіть інший знімок:

```bash
incus snapshot rockylinux-test-9 rockylinux-test-9-snap1
```

Якщо ви запустите команду "info" для `incus`, ви побачите знімок у нижній частині свого списку:

```bash
incus info rockylinux-test-9
```

Унизу буде показано щось на зразок цього:

```bash
rockylinux-test-9-snap1 at 2021/05/13 16:34 UTC) (stateless)
```

Тож, тримаємо пальці! Давайте спробуємо перенести ваш snapshot:

```bash
incus copy rockylinux-test-9/rockylinux-test-9-snap1 incus-snapshot:rockylinux-test-9
```

Ця команда говорить, що всередині контейнера rockylinux-test-9 ви хочете надіслати snapshot rockylinux-test-9-snap1 до incus-snapshot і назвати його rockylinux-test-9.

Через короткий час копіювання буде завершено. Хочете дізнатися напевно? Створіть `incus list` на сервері incus-snapshot. Що має повернути наступне:

```bash
+-------------------+---------+------+------+-----------+-----------+
|    NAME           |  STATE  | IPV4 | IPV6 |   TYPE    | SNAPSHOTS |
+-------------------+---------+------+------+-----------+-----------+
| rockylinux-test-9 | STOPPED |      |      | CONTAINER | 0         |
+-------------------+---------+------+------+-----------+-----------+
```

Success! Спробуйте запустити його. Оскільки ви запускаєте його на сервері incus-snapshot, вам потрібно спершу зупинити його на сервері incus-primary, щоб уникнути конфлікту IP-адрес:

```bash
incus stop rockylinux-test-9
```

А на сервері incus-snapshot:

```bash
incus start rockylinux-test-9
```

Якщо припустити, що все це працює без помилок, зупиніть контейнер на incus-snapshot і запустіть його знову на incus-primary.

## Вимкнення `boot.autostart` для контейнерів

Снепшоти, скопійовані в incus-snapshot, не працюватимуть під час міграції, але якщо у вас виникне подія живлення або потрібно перезавантажити сервер через оновлення чи щось інше, у вас виникне проблема. Ці контейнери намагатимуться запуститися на сервері знімків, створюючи потенційний конфлікт IP-адрес.

Щоб усунути це, потрібно налаштувати переміщені контейнери так, щоб вони не запускалися після перезавантаження сервера. Для вашого щойно скопійованого контейнера rockylinux-test-9 ви зробите це за допомогою:

```bash
incus config set rockylinux-test-9 boot.autostart 0
```

Зробіть це для кожного snapshot на сервері incus-snapshot. "0" у команді гарантує, що `boot.autostart` вимкнено.

## Автоматизація snapshot процесу

Чудово, що ви можете створювати миттєві знімки за потреби, а іноді вам _до_ потрібно створити миттєвий знімок вручну. Ви навіть можете скопіювати його в incus-snapshot вручну. Але в інших випадках, особливо для багатьох контейнерів, які працюють на вашому первинному сервері incus, **останнє**, що ви хочете зробити, це витратити півдня на видалення знімків на сервері знімків, створення нових знімків і надсилання їх до сервер знімків. Для основної частини ваших операцій ви захочете автоматизувати процес.

Перше, що вам потрібно зробити, це запланувати процес автоматизації створення знімка на incus-primary. Ви зробите це для кожного контейнера на сервері incus-primary. Після завершення він подбає про це в майбутньому. Це можна зробити за допомогою наступного синтаксису. Зверніть увагу на схожість із записом crontab для позначки часу:

```bash
incus config set [container_name] snapshots.schedule "50 20 * * *"
```

Це означає, що щодня о 20:50 робіть знімок назви контейнера.

Щоб застосувати це до контейнера rockylinux-test-9:

```bash
incus config set rockylinux-test-9 snapshots.schedule "50 20 * * *"
```

Ви також хочете налаштувати ім’я snapshot, щоб мати значення для вашої дати. Incus скрізь використовує UTC, тому найкраще, щоб відстежувати речі, це встановити ім’я знімка з датою та міткою часу в більш зрозумілому форматі:

```bash
incus config set rockylinux-test-9 snapshots.pattern "rockylinux-test-9{{ creation_date|date:'2006-01-02_15-04-05' }}"
```

ЧУДОВО, але ви точно не хочете отримувати новий знімок щодня, не позбувшись старого, чи не так? Ви б заповнили диск знімками. Щоб виправити це, виконайте:

```bash
incus config set rockylinux-test-9 snapshots.expiry 1d
```