# Лабораторная работа №2. Виртуальная машина Debian Server в QEMU

## Выполнил

* **Студент:** Splavski Maxim
* **Группа:** I2402

## Описание задачи

Цель работы - подготовить виртуальную машину Debian Server x64 в QEMU, установить LAMP-окружение, развернуть phpMyAdmin и WordPress, настроить виртуальные хосты Apache и проверить доступность сайтов через проброшенный порт.

В ходе работы были выполнены:

- подготовлен ISO-образ Debian Server и виртуальный диск `debian.qcow2`;
- создана виртуальная машина с параметрами памяти, диска и сетевого проброса портов;
- установлены Apache, PHP, MariaDB и дополнительные пакеты;
- загружены и распакованы phpMyAdmin и WordPress;
- создана база данных `wordpress_db` и пользователь `user`;
- подготовлены конфигурации Apache для `phpmyadmin.localhost` и `wordpress.localhost`.

## Структура проекта

- `.gitignore` - исключает из репозитория тяжелые файлы `.qcow2`, `.iso`, `.zip`.
- `dvd/readme.md` - поясняет, какой ISO-образ должен находиться в папке `dvd`.
- `apache/01-phpmyadmin.conf` - виртуальный хост phpMyAdmin.
- `apache/02-wordpress.conf` - виртуальный хост WordPress.
- `notes/commands.md` - основные команды, использованные в работе.
- `readme.md` - отчет по лабораторной работе.

Файлы `debian.iso`, `debian.qcow2`, `latest.zip` и архив phpMyAdmin не добавляются в репозиторий.

## Подготовка QEMU и диска

ISO-образ Debian был переименован в:

```text
dvd/debian.iso
```

Виртуальный диск создан командой:

```text
qemu-img create -f qcow2 debian.qcow2 8G
```

Ожидаемый результат:

```text
Formatting 'debian.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off compression_type=zlib size=8589934592 lazy_refcounts=off refcount_bits=16
```

Команда для установки Debian:

```text
qemu-system-x86_64 -hda debian.qcow2 -cdrom dvd/debian.iso -boot d -m 2G
```

Параметры установки:

- имя компьютера: `debian`;
- хостовое имя: `debian.localhost`;
- пользователь: `user`;
- пароль пользователя: `password`.

## Запуск виртуальной машины

После установки Debian виртуальная машина запускается командой:

```text
qemu-system-x86_64 -hda debian.qcow2 -m 2G -smp 2 -device e1000,netdev=net0 -netdev user,id=net0,hostfwd=tcp::1080-:80,hostfwd=tcp::1022-:22
```

Параметр `hostfwd=tcp::1080-:80` пробрасывает HTTP-порт гостевой системы на порт `1080` хоста. Параметр `hostfwd=tcp::1022-:22` позволяет подключаться к SSH гостевой системы через порт `1022`.

## Установка LAMP

Внутри Debian выполнены команды:

```text
su
apt update -y
apt install -y apache2 php libapache2-mod-php php-mysql mariadb-server mariadb-client unzip
```

Назначение пакетов:

- `apache2` - веб-сервер;
- `php` - интерпретатор PHP;
- `libapache2-mod-php` - модуль Apache для выполнения PHP;
- `php-mysql` - расширение PHP для подключения к MySQL/MariaDB;
- `mariadb-server` - сервер базы данных;
- `mariadb-client` - консольный клиент MariaDB;
- `unzip` - распаковка ZIP-архивов.

## Установка phpMyAdmin и WordPress

Загрузка файлов:

```text
wget https://files.phpmyadmin.net/phpMyAdmin/5.2.2/phpMyAdmin-5.2.2-all-languages.zip
wget https://wordpress.org/latest.zip
```

Проверка:

```text
ls -l
```

Пример результата:

```text
-rw-r--r-- 1 root root 15024774 Apr 19 11:36 phpMyAdmin-5.2.2-all-languages.zip
-rw-r--r-- 1 root root 26771790 Apr 19 11:37 latest.zip
```

Распаковка:

```text
mkdir /var/www
unzip phpMyAdmin-5.2.2-all-languages.zip
mv phpMyAdmin-5.2.2-all-languages /var/www/phpmyadmin
unzip latest.zip
mv wordpress /var/www/wordpress
```

## Настройка MariaDB

В MariaDB создана база данных и пользователь:

```sql
CREATE DATABASE wordpress_db;
CREATE USER 'user'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'user'@'localhost';
FLUSH PRIVILEGES;
```

Эта учетная запись используется WordPress для подключения к базе `wordpress_db`.

## Настройка Apache

Для phpMyAdmin подготовлен файл `/etc/apache2/sites-available/01-phpmyadmin.conf`. Его копия находится в `apache/01-phpmyadmin.conf`.

Для WordPress подготовлен файл `/etc/apache2/sites-available/02-wordpress.conf`. Его копия находится в `apache/02-wordpress.conf`.

Сайты включаются командами:

```text
/usr/sbin/a2ensite 01-phpmyadmin
/usr/sbin/a2ensite 02-wordpress
```

В `/etc/hosts` добавлены строки:

```text
127.0.0.1 phpmyadmin.localhost
127.0.0.1 wordpress.localhost
```

Перезагрузка Apache:

```text
systemctl reload apache2
```

Также можно использовать:

```text
service apache2 reload
```

## Запуск и тестирование

Команда:

```text
uname -a
```

Ожидаемый вывод:

```text
Linux debian 6.1.0-18-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.76-1 (2024-02-01) x86_64 GNU/Linux
```

Проверяемые адреса:

```text
http://wordpress.localhost:1080
http://phpmyadmin.localhost:1080
```

После открытия `http://wordpress.localhost:1080` завершается мастер установки WordPress. Для подключения используются:

- база данных: `wordpress_db`;
- пользователь: `user`;
- пароль: `password`;
- сервер базы данных: `localhost`.

## Ответы на вопросы

1. Каким образом можно скачать файл в консоли при помощи `wget`?

   Нужно выполнить команду `wget` и передать ей URL файла:

   ```text
   wget https://example.com/file.zip
   ```

   Утилита скачает файл в текущий каталог.

2. Зачем необходимо создавать для каждого сайта свою базу и своего пользователя?

   Это ограничивает права приложения. Если один сайт будет скомпрометирован, злоумышленник не получит автоматический доступ к базам других сайтов.

3. Как поменять доступ к системе управления БД на порт `1234`?

   Если речь идет о доступе к phpMyAdmin через Apache, можно изменить проброс порта QEMU с `hostfwd=tcp::1080-:80` на `hostfwd=tcp::1234-:80`, после чего открывать `http://phpmyadmin.localhost:1234`. Если нужно изменить порт самого MariaDB, параметр `port = 1234` задается в конфигурации MariaDB, а клиентам указывается новый порт подключения.

4. Какие преимущества дает виртуализация?

   Виртуализация позволяет запускать отдельную ОС без изменения основной системы, быстро пересоздавать стенды, изолировать эксперименты и тестировать серверные настройки в контролируемой среде.

5. Для чего необходимо устанавливать время и временную зону на сервере?

   Корректное время нужно для журналов, cron-задач, SSL-сертификатов, сессий, резервного копирования и синхронизации событий между сервисами.

6. Сколько места занимает установленная ОС на хостовой машине?

   Виртуальный диск создан с максимальным размером 8 ГБ, но формат `qcow2` занимает место динамически. После минимальной установки Debian с LAMP фактический размер файла `debian.qcow2` обычно составляет около 2.7-3.4 ГБ.

7. Какие есть рекомендации по разбиению диска для серверов?

   Обычно отдельно выделяют `/`, `/var`, `/home`, `swap`, иногда `/tmp` и `/var/log`. Разделение полезно потому, что переполнение логов или пользовательских данных не блокирует всю систему. Для серверов с базами данных часто отдельно выносят каталог с данными БД, чтобы проще управлять резервным копированием и производительностью.

## Безопасность

В учебной работе используется простой пароль `password`, так как он указан в условии. Для реальной системы пароль должен быть сложным, доступ к MariaDB должен быть ограничен, а админ-панели вроде phpMyAdmin лучше защищать дополнительными правилами доступа.

## Использованные источники

- Документация QEMU: `qemu-img`, `qemu-system-x86_64`.
- Debian Administrator's Handbook: установка пакетов и настройка сервисов.
- Документация Apache HTTP Server: VirtualHost.
- Документация MariaDB: пользователи и привилегии.
- Документация WordPress и phpMyAdmin.

## Краткий вывод

В результате была подготовлена схема виртуальной Debian-среды в QEMU, установлен LAMP-стек и настроены два сайта через Apache VirtualHost. Работа закрепляет базовые навыки администрирования Linux-сервера, настройки веб-сервера, базы данных и проброса портов в виртуальной машине.
