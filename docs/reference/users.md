# Користувачі

Докладніше: [модуль 7](../modules/07-users-and-groups.md).

## Огляд

```bash
whoami
id
id -Gn
groups
getent passwd
getent passwd "$USER"
getent group sudo
who
w
last -n 5
```

| Команда | Дія |
|---|---|
| `id` | UID, основний GID і всі групи |
| `getent passwd/group` | Записи з усіх джерел облікових записів |
| `who`, `w` | Хто увійшов зараз |
| `last` | Історія входів (`/var/log/wtmp`) |
| `sudo lastb` | Невдалі спроби входу |

## Типи облікових записів

| Тип | Типовий UID | Примітка |
|---|---|---|
| `root` | `0` | Повний доступ |
| Системні | `1`–`999` | Служби, оболонка `nologin` |
| Звичайні | `1000` і більше | Люди |
| `nobody` | `65534` | Ізоляція |

Діапазони задає `/etc/login.defs` (`grep '^UID_' /etc/login.defs`).

## Файли облікових записів

| Файл | Зміст | Права |
|---|---|---|
| `/etc/passwd` | Ім'я, `x`, UID, GID, коментар, домашній каталог, оболонка | `644` |
| `/etc/shadow` | Хеші паролів і терміни дії | `640`, `root:shadow` |
| `/etc/group` | Ім'я групи, `x`, GID, додаткові члени | `644` |
| `/etc/gshadow` | Паролі груп | `640` |
| `/etc/skel` | Шаблон файлів для нових домашніх каталогів | — |
| `/etc/sudoers`, `/etc/sudoers.d/` | Правила `sudo` (редагувати лише `visudo`) | — |

Формат `/etc/passwd`: `name:x:UID:GID:comment:home:shell`.

```bash
cut -d: -f1,3,6,7 /etc/passwd | head
grep '^student:' /etc/passwd
```

У `/etc/shadow` `!` на початку поля — пароль заблоковано, `*` — вхід за паролем неможливий.

## Користувачі

```bash
sudo useradd -m -s /bin/bash -c "Demo User" demo1
sudo passwd demo1
sudo passwd -S demo1
sudo passwd -l demo1
sudo passwd -u demo1
sudo usermod -aG labteam demo1
sudo usermod -s /usr/sbin/nologin demo1
sudo chage -l demo1
sudo userdel -r demo1
```

| Параметр / команда | Дія |
|---|---|
| `useradd -m` | Створити домашній каталог (без нього його не буде) |
| `useradd -s` | Оболонка входу (в Ubuntu за замовчуванням `/bin/sh`) |
| `useradd -c`, `-g`, `-G`, `-u`, `-d`, `-r` | Коментар, основна група, додаткові групи, UID, каталог, системний запис |
| `adduser` | Інтерактивна оболонка Debian/Ubuntu над `useradd` |
| `passwd -S` | Стан пароля: `P` задано, `L` заблоковано, `NP` немає |
| `usermod -aG` | **Додати** до груп |
| `usermod -G` | **Замінити** список груп (без `-a` — небезпечно) |
| `chage -l`, `-M`, `-E` | Терміни дії пароля й запису |
| `userdel -r` | Видалити з домашнім каталогом |

!!! warning

    Завжди `usermod -aG`, а не `usermod -G`: без `-a` користувач втратить усі інші групи, зокрема `sudo`.

Новий запис, створений `useradd`, **заблокований і без пароля**: розблокувати акаунт без пароля система відмовляється, спершу задайте пароль.

## Групи

```bash
sudo groupadd labteam
getent group labteam
sudo gpasswd -a user labteam
sudo gpasswd -d user labteam
sudo groupmod -n newname labteam
sudo groupdel labteam
```

- група, яка є основною для наявного користувача, видаленню не підлягає;
- зміни груп поточного користувача діють після нового входу (`id "$USER"` бачить оновлення, `id` у старому сеансі — ні).

## Спільний каталог групи

```bash
sudo mkdir /srv/team
sudo chown root:labteam /srv/team
sudo chmod 2770 /srv/team
```

`770` обмежує доступ власником і групою, `setgid` (`2`) змушує нові файли успадковувати групу каталогу.

## `sudo` і `su`

```bash
sudo -l
sudo whoami
sudo -u demo1 id
sudo -iu demo1
sudo -k
sudo visudo -c
su - demo1
```

| Команда | Дія |
|---|---|
| `sudo команда` | Одна команда від імені `root`; запитує **ваш** пароль |
| `sudo -u користувач команда` | Від імені іншого користувача (зручно перевіряти права) |
| `sudo -i`, `sudo -iu` | Оболонка входу від `root` / користувача |
| `sudo -l` | Що вам дозволено |
| `sudo -k` | Скинути кеш автентифікації |
| `su - користувач` | Оболонка користувача; запитує **його** пароль |
| `visudo` | Безпечне редагування `sudoers` (перевірка синтаксису) |

У Ubuntu право на `sudo` мають члени групи `sudo`; прямий вхід `root` за паролем вимкнений.

### `sudo` і перенаправлення

`sudo echo test > /etc/file` не працює: перенаправлення виконує **ваша** оболонка. Використовуйте:

```bash
echo test | sudo tee /srv/team/file.txt
sudo bash -c 'echo test > /srv/team/file.txt'
```

## Блокування, оболонка, термін дії

| Механізм | Як | Що забороняє |
|---|---|---|
| Блокування пароля | `passwd -l`, `usermod -L` | Вхід за паролем (ключі SSH і `sudo -u` лишаються) |
| `nologin` | `usermod -s /usr/sbin/nologin` | Інтерактивний вхід, `su` в оболонку |
| Закінчення запису | `chage -E дата` | Усі способи входу після дати |

## Навчальна практика

Для навчальної системи (лише в VM) створюйте тестових користувачів із префіксом і **видаляйте їх після вправ**:

```bash
sudo useradd -m -s /bin/bash demo1
sudo passwd demo1
sudo userdel -r demo1
getent passwd demo1
```

Останню команду виконайте для перевірки: вона не повинна нічого виводити. Для навчальних паролів не використовуйте реальні.
