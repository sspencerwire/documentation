---
title: Система виявлення вторгнень на основі хоста (HIDS)
author: Steven Spencer
contributors: Ezequiel Bruni
tested_with: 8.8, 9.2
tags:
  - web
  - безпека
  - ossec-hids
  - hids
---

# Система виявлення вторгнень на основі хоста (HIDS)

## Передумови

* Вміння працювати з текстовим редактором командного рядка (у цьому прикладі використовується `vi`)
* Високий рівень комфорту з видачею команд із командного рядка, переглядом журналів та іншими загальними обов’язками системного адміністратора
* Розуміння того, що встановлення цього інструменту також потребує моніторингу дій і налаштування під ваше середовище
* Користувач root виконує всі команди або звичайний користувач за допомогою `sudo`

## Вступ

`ossec-hids` – це система виявлення вторгнень на хост, яка пропонує автоматичні кроки дії-відповіді для пом’якшення атак вторгнення на хост. Це лише одна з можливих частин посиленого веб-сервера Apache. Ви можете використовувати його з іншими інструментами або без них.

Якщо ви бажаєте скористатися цим та іншими інструментами захисту, зверніться до документа [Apache Hardened Web Server](index.md). У цьому документі також використовуються всі припущення та умовності, викладені в цьому оригінальному документі. Перш ніж продовжити, варто переглянути його.

## Встановлення репозиторію Atomicorp

Щоб установити `ossec-hids`, нам потрібен сторонній репозиторій від Atomicorp. Atomicorp також пропонує платну підтримувану версію за розумною ціною для тих, хто потребує професійної підтримки, якщо у них виникнуть проблеми.

Якщо ви віддаєте перевагу підтримці та маєте на це бюджет, перегляньте [платні `ossec-hids`](https://atomicorp.com/atomic-enterprise-ossec/) Atomicorp версії. Вам потрібно лише кілька пакетів із безкоштовного сховища Atomicorp. Ви збираєтеся змінити репозиторій після завантаження.

Для завантаження репозиторію потрібен `wget`. Спочатку встановіть це, а потім інсталюйте репозиторій EPEL, якщо він у вас ще не встановлений, за допомогою:

```
dnf install wget epel-release
```

Завантажте та ввімкніть безкоштовний репозиторій Atomicorp:

```
wget -q -O - http://www.atomicorp.com/installers/atomic | sh
```

Цей сценарій попросить вас погодитися з умовами. Введіть «yes» або <kbd>Enter</kbd>, щоб прийняти значення за замовчуванням.

Далі він запитає, чи бажаєте ви ввімкнути репозиторій за замовчуванням, і ще раз, чи бажаєте ви прийняти значення за замовчуванням або введіть «так».

### Налаштування репозиторію Atomicorp

Вам потрібен лише атомарний репозиторій для кількох пакунків. З цієї причини ви збираєтеся змінити репозиторій і вказати лише потрібні пакети:

```
vi /etc/yum.repos.d/atomic.repo
```

Додайте цей рядок під «enabled = 1» у верхній частині:

```
includepkgs = ossec* GeoIP* inotify-tools
```

Це єдина зміна, яка вам потрібна. Збережіть зміни та вийдіть із сховища (у `vi` це <kbd>esc</kbd>, щоб увійти в командний режим, потім <kbd>SHIFT</kbd>+<kbd>:</kbd>+<kbd>wq</kbd>, щоб зберегти та вийти).

Це обмежує репозиторій Atomicorp лише для встановлення та оновлення цих пакетів.

## Встановлення `ossec-hids`

З налаштованим репозиторієм вам потрібно встановити пакунки:

```
dnf install ossec-hids-server ossec-hids inotify-tools
```

### Налаштування `ossec-hids`

Конфігурація за замовчуванням знаходиться в стані, що потребує багатьох змін. Більшість із них стосується сповіщень адміністратора сервера та розташування журналів.

`ossec-hids` переглядає журнали, щоб спробувати вирішити, чи відбувається атака та чи застосовувати пом’якшення. Він також надсилає звіти адміністратору сервера зі сповіщенням або повідомленням про процедуру пом’якшення, запущену на основі того, що побачив `ossec-hids`.

Щоб відредагувати файл конфігурації, введіть:

```
vi /var/ossec/etc/ossec.conf
```

Автор розбере цю конфігурацію, показуючи зміни в рядку та пояснюючи їх:

```
<global>
  <email_notification>yes</email_notification>  
  <email_to>admin1@youremaildomain.com</email_to>
  <email_to>admin2@youremaildomain.com</email_to>
  <smtp_server>localhost</smtp_server>
  <email_from>ossec-webvms@yourwebserverdomain.com.</email_from>
  <email_maxperhour>1</email_maxperhour>
  <white_list>127.0.0.1</white_list>
  <white_list>192.168.1.2</white_list>
</global>
```

Сповіщення електронною поштою вимкнено за умовчанням, а конфігурація `<global>` майже порожня. Ви хочете ввімкнути сповіщення електронною поштою та визначити людей, які отримуватимуть звіти електронною поштою, за їхніми електронними адресами.

У розділі `<smtp_server>` наразі показано локальний хост, однак ви можете вказати ретранслятор електронної пошти, якщо бажаєте, або налаштувати постфіксні параметри електронної пошти для локального хосту, виконавши [цей посібник](../../email/postfix_reporting.md).

Вам потрібно встановити адресу електронної пошти "from". Це потрібно для роботи з фільтрами СПАМу на вашому сервері електронної пошти, які можуть бачити цей електронний лист як СПАМ. Щоб уникнути перенасичення електронною поштою, встановіть звітування електронною поштою 1 раз на годину. Ви можете розширити або виділити цю команду, починаючи з `ossec-hids`.

Розділи `<white_list>` стосуються локальної IP-адреси сервера та «загальнодоступної» IP-адреси (пам’ятайте, що ми замінюємо приватну IP-адресу) брандмауера, з якого відображатимуться всі з’єднання в довіреній мережі. Ви можете додати багато записів `<white_list>`.

```
<syscheck>
  <!-- Frequency that syscheck is executed -- default every 22 hours -->
  <frequency>86400</frequency>
...
</syscheck>
```

Розділ `<syscheck>` переглядає список каталогів, які слід включити та виключити під час пошуку скомпрометованих файлів. Подумайте про це як про ще один інструмент для спостереження та захисту файлової системи від уразливостей. Вам потрібно переглянути список каталогів і додати інші, які ви хочете, до розділу `<syscheck>`.

Розділ `<rootcheck>` безпосередньо під розділом `<syscheck>` є ще одним рівнем захисту. Розташування, які спостерігають `<syscheck>` і `<rootcheck>`, можна редагувати, але вам, ймовірно, не потрібно буде вносити в них жодних змін.

Змінення `<frequence>` для виконання `<rootcheck>` на один раз кожні 24 години (86400 секунд) замість 22 годин за замовчуванням є необов’язковою зміною.

```
<localfile>
  <log_format>apache</log_format>
  <location>/var/log/httpd/*access_log</location>
</localfile>
<localfile>
  <log_format>apache</log_format>
  <location>/var/log/httpd/*error_log</location>
</localfile>
```

Розділ `<localfile>` стосується розташування журналів, які ви хочете переглянути. У журналах _syslog_ і _secure_ вже є записи, шлях до яких потрібно перевірити, але все інше може залишитися.

Вам потрібно додати розташування журналів Apache і додати їх як символи підстановки, оскільки ви можете мати купу журналів для багатьох різних веб-клієнтів.

```
  <command>
    <name>firewalld-drop</name>
    <executable>firewall-drop.sh</executable>
    <expect>srcip</expect>
  </command>

  <active-response>
    <command>firewall-drop</command>
    <location>local</location>
    <level>7</level>
  </active-response>
```

Нарешті, ближче до кінця файлу вам потрібно додати розділ активної відповіді. Він має дві частини: розділ `<command>` і розділ `<active-response>`.

Сценарій "firewall-drop" уже існує в межах шляху `ossec-hids`. Він повідомляє `ossec-hids`, що якщо виникає рівень 7, додайте правило брандмауера для блокування IP-адреси.

Увімкніть і запустіть службу після завершення всіх змін конфігурації. Якщо все розпочато правильно, ви готові рухатися далі:

```
systemctl enable ossec-hids
systemctl start ossec-hids
```

Файл конфігурації `ossec-hids`. Ви можете дізнатися про ці параметри, відвідавши [сайт офіційної документації](https://www.ossec.net/docs/).

## Висновок

`ossec-hids` — це лише один із захищених Apache елементів веб-сервера. Ви можете отримати кращий захист, вибравши його з іншими інструментами.

Хоча встановлення та конфігурація відносно прості, ви побачите, що це **не** програма «встановити й забути». Ви повинні налаштувати його відповідно до свого середовища, щоб отримати максимальну безпеку з найменшою кількістю хибно-позитивних відповідей.