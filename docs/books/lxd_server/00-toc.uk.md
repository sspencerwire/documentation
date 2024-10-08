---
title: Вступ
author: Steven Spencer
contributors: Ezequiel Bruni, Ganna Zhyrnova
tested_with: 8.8, 9.2
tags:
  - lxd
  - enterprise
---

# Створення повноцінного сервера LXD

!!! info "примітка"

    Ця процедура має працювати для Rocky Linux 8.x або 9.x. Якщо ви шукаєте сучасну реалізацію цього проекту від колишніх провідних розробників LXD, але доступну лише для Rocky Linux 9.x, перегляньте [книгу Incus Server](../incus_server/00-toc.md).

!!! info "Що сталося з проектом LXD"

    Понад рік тому в списку розсилки lxc-users з’явилося таке оголошення:
    
    > Компанія Canonical, творець і головний учасник проекту LXD, вирішила, що після восьми років роботи в спільноті Linux Containers цей проект буде краще обслуговуватися безпосередньо у власному наборі проектів Canonical.
    
    Одним із вирішальних факторів стали відставки деяких провідних розробників LXD, які потім продовжили форк LXD на Incus, оголосивши про форк у серпні 2023 року. Версія випуску (0.1) вийшла в жовтні 2023 року, і з тих пір розробники швидко створили цю версію, випускаючи поетапно до 0.7 (березень 2024). Слідом за версією 0.7 4 квітня 2024 року з’явилася версія з довгостроковою підтримкою 6.0 LTS, а тепер – 6.4 LTS (станом на вересень 2024 року).
    
    Вважалося, що протягом усього процесу Canonical продовжувала підтримувати посилання на зображення контейнерів, надані Linux Containers. Тим не менш, через [зміну ліцензії](https://stgraber.org/2023/12/12/lxd-now-re-licensed-and-under-a-cla/) контейнери Linux стали неможливими для продовження пропонуючи зображення контейнерів у LXD. Хоча контейнери Linux більше не можуть надавати зображення контейнерів для LXD, проекту LXD вдалося створити деякі контейнери, включно з контейнерами для Rocky Linux. 
    
    Цей документ використовує LXD, а не Incus.

## Вступ

LXD найкраще описано на [офіційному сайті](https://documentation.ubuntu.com/lxd/en/latest/), але розглядайте його як контейнерну систему, яка надає переваги віртуальних серверів у контейнері.

Він дуже потужний, і з правильним апаратним забезпеченням і налаштуваннями його можна використовувати для запуску багатьох екземплярів сервера на одному апаратному забезпеченні. Якщо ви об’єднаєте це з сервером snapshot, у вас також буде набір контейнерів, які можна розкрутити майже негайно, якщо ваш основний сервер вийде з ладу.

(Не слід сприймати це як традиційне резервне копіювання. Вам усе ще потрібна якась звичайна система резервного копіювання, наприклад [rsnapshot](../../guides/backup/rsnapshot_backup.md).)

Крива навчання для LXD може бути трохи крутою, але ця книга спробує дати вам велику кількість знань, щоб допомогти вам розгорнути та використовувати LXD на Rocky Linux.

Для тих, хто хоче використовувати LXD як лабораторне середовище на власних ноутбуках або робочих станціях, перегляньте [Додаток A: налаштування робочої станції](30-appendix_a.md).

## Передумови та припущення

* Один сервер Rocky Linux, гарно налаштований. Вам слід розглянути окремий жорсткий диск для дискового простору ZFS (вам потрібно, якщо ви використовуєте ZFS) у робочому середовищі. І так, ми припускаємо, що це чистий сервер, а не VPS.
* Це слід вважати розширеною темою, але ми зробили все можливе, щоб зробити її максимально зрозумілою для всіх. Тим не менш, знання кількох основних речей про керування контейнерами займе у вас довгий шлях. Знання кількох базових речей про керування контейнерами значно допоможе.
* Ви маєте добре володіти командним рядком на своїй машині (машинах) і вільно володіти редактором командного рядка. (У цьому прикладі ми використовуємо _vi_, але ви можете замінити його но свій улюблений редактор.)
* Ви повинні бути непривілейованим користувачем для більшості цих процесів. Якщо не зазначено, вводьте команди LXD як непривілейований користувач. Ми припускаємо, що ви ввійшли як користувач з іменем "lxdadmin" для команд LXD. Пізніше вам потрібно буде створити цей обліковий запис користувача.
* Для ZFS переконайтеся, що безпечне завантаження UEFI НЕ ввімкнено. В іншому випадку вам доведеться підписати модуль ZFS, щоб змусити його завантажити.
* Здебільшого ми використовуємо контейнери на основі Rocky Linux.

## Короткий Огляд

* **Розділ 1: Встановлення та налаштування** стосується встановлення основного сервера. Загалом, правильний спосіб зробити LXD у виробництві — мати як основний сервер, так і сервер знімків.
* **Розділ 2: Налаштування ZFS** розповідає про налаштування та налаштування ZFS. ZFS — це диспетчер логічних томів і файлова система з відкритим вихідним кодом, створена Sun Microsystems спочатку для своєї операційної системи Solaris.
* **Розділ 3: Ініціалізація LXD і налаштування користувача** стосується базової ініціалізації та параметрів, а також налаштування непривілейованого користувача, якого ви використовуватимете протягом більшої частини решти процесу
* **Розділ 4: Налаштування брандмауера** стосується параметрів налаштування як `iptables`, так і `firewalld`, але ми рекомендуємо використовувати `firewalld`для всіх поточних версій Rocky Linux.
* **Розділ 5: Налаштування образів і керування ними** описує процес встановлення образів ОС до контейнера та їх налаштування.
* **Розділ 6: Профілі** розповідає про додавання профілів і їх застосування до контейнерів, а також охоплює macvlan і його важливість для IP-адресації у вашій LAN або WAN
* **Розділ 7: Параметри конфігурації контейнера** коротко описує деякі основні параметри конфігурації для контейнерів і пропонує деякі плюси та мінуси для зміни параметрів конфігурації.
* **Розділ 8: Знімки контейнерів** детально описує процес створення знімків для контейнерів на основному сервері.
* **Розділ 9. Сервер Snapshot** розповідає про налаштування та конфігурацію сервера snapshot, а також про те, як створити симбіотичний зв’язок між основним і сервером snapshot.
* **Розділ 10: Автоматизація Snapshots** розповідає про автоматизацію створення snapshot і заповнення сервера snapshot новими snapshots.
* **Додаток A: налаштування робочої станції** технічно не є частиною документів робочого сервера, але пропонує рішення для людей, які хочуть у простий спосіб створити лабораторію контейнерів LXD на своїх особистих ноутбуках або робочіх станціях.

## Висновки

Ви можете використовувати ці розділи для ефективного налаштування основного сервера LXD на рівні підприємства та знімків. У процесі ви дізнаєтеся багато нового про LXD. Просто майте на увазі, що ще багато чого потрібно дізнатися, і розглядайте ці документи як відправну точку.

Найбільша перевага LXD полягає в тому, що він економічний у використанні на сервері, дозволяє швидко розкручувати встановлення операційної системи, дозволяє багато автономних серверів додатків працювати на одному апаратному забезпеченні, використовуючи це обладнання для максимального використання.
