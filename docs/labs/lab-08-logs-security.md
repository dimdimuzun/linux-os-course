# Лабораторна робота № 8. Логи та безпека

## Тема

**Аналіз журналів, аудит безпеки та комплексна діагностика Linux. Робота з `journalctl` і текстовими журналами, розслідування інциденту за журналами, ротація журналів, аудит безпеки системи, діагностика несправностей за методикою.**

## Пов'язані модулі

- Модуль 15. Логи та безпека.
- Модуль 16. Діагностика Linux.

## Мета

Закріпити теоретичні знання про журналювання, безпеку й діагностику Linux, сформувати практичні навички аналізу журналів, розпізнавання ознак атак і компрометації, виконання аудиту безпеки та систематичної діагностики несправностей із поєднанням знань усього курсу.

Під час виконання лабораторної роботи студент повинен:

- читати журнали `systemd` за допомогою `journalctl` із фільтрами та записувати власні події за допомогою `logger`;
- досліджувати файли в `/var/log`, журнали входів і налаштування ротації системи;
- аналізувати журнал веб-сервера за допомогою `cut`, `awk`, `sort`, `uniq`, `grep` і формулювати висновки;
- розслідувати інцидент за журналом автентифікації: виявляти підбір паролів, успішну компрометацію й подальші дії зловмисника;
- розпізнавати реальні події `sudo` і входів у системі;
- налаштовувати й перевіряти ротацію журналів за допомогою `logrotate`;
- запускати й інтерпретувати скрипт аудиту безпеки, перевіряти й доповнювати його;
- діагностувати кілька одночасних несправностей (процесор, диск, служба, порт) за методикою, фіксуючи докази;
- скласти звіт про інцидент (post-mortem);
- повністю відновити початковий стан системи.

## Вимоги

Для виконання лабораторної роботи необхідно мати:

- навчальну віртуальну машину з Ubuntu 24.04 LTS;
- обліковий запис із правом на `sudo` (член групи `sudo`);
- не менше 500 МБ вільного місця на кореневій файловій системі;
- доступ до навчальних матеріалів модулів 15 і 16;
- результати лабораторних робіт № 1–7 для довідки.

**Рекомендація:** створіть **знімок VM** перед початком роботи (модуль 2).

Допускається використання іншого дистрибутива Linux. Наявність файлів `/var/log/auth.log`, `/var/log/syslog`, служби SSH, `ufw`, AppArmor залежить від середовища; відсутність окремих компонентів необхідно зафіксувати у звіті, а за можливості скористатися `journalctl`.

**Важливо:** у цій роботі використовується `sudo`. Усі дії виконуються лише у навчальній VM. Журнали для аналізу (`access.log` і `auth.log`) є **тестовими** (створюються в роботі) й не містять реальних даних. Адреси в них належать зарезервованим діапазонам документації. **Не змінюйте** налаштування брандмауера, SSH, облікові записи й системні файли, окрім об'єктів, названих у роботі.

## Правила виконання роботи

1. Уважно прочитайте умову кожного завдання.
2. Виконуйте команди послідовно: наступні завдання використовують результати попередніх.
3. Спочатку **спостерігайте**, потім змінюйте. Зміни виконуйте лише в завданнях, де це прямо передбачено.
4. Для кожного твердження в аналізі наводьте **доказ**: команду й фрагмент виводу.
5. Записуйте результати та повідомлення про помилки у звіт.
6. Не змінюйте кілька речей одночасно під час діагностики; фіксуйте кожну дію та її результат.
7. Не вставляйте у звіт справжні паролі, ключі й особисті дані.
8. Деякі помилки в цій роботі є навмисними: фіксуйте повідомлення та пояснюйте їх.
9. Обов'язково виконайте прибирання (завдання 9).

Усі приклади виведення команд у цій роботі є умовними. Результати у вашому середовищі можуть відрізнятися.

---

## Початкова підготовка

### 1. Перевірте право на `sudo` і наявність інструментів

```bash
whoami
id
sudo -l
command -v journalctl logger logrotate awk ufw aa-status python3
```

Рядки, яких немає у виведенні, відповідають відсутнім інструментам. Запишіть їх. Установлювати пакети в цій роботі не потрібно, але якщо `logrotate` відсутній, його можна встановити (`sudo apt install logrotate`) і зазначити в звіті.

### 2. Зафіксуйте початковий стан

```bash
systemctl --failed --no-legend | wc -l
ls /etc/systemd/system | grep -c lab08
ls /usr/local/bin | grep -c lab08
ls /etc/lab08-app.conf
findmnt | wc -l
ls /mnt
pgrep -c yes
pgrep -fc 'http.server'
ss -tuln | wc -l
jobs
```

Запишіть результати. Команда `ls /etc/lab08-app.conf` має повідомити, що файла немає.

### 3. Створіть робочі каталоги та знімок VM

```bash
mkdir -p ~/lab08/logs ~/lab08/scripts ~/lab08/trouble ~/lab08/web
cd ~/lab08
pwd
```

Створіть знімок VM і назвіть його, наприклад, `before-lab08`.

### Контрольна перевірка

Після початкової підготовки ви повинні вміти відповісти на запитання:

- Які інструменти журналювання й безпеки доступні у вашій VM?
- Чи є у вашій системі збійні служби до початку роботи?
- Який стан системи ви вважаєте початковим?

---

## Завдання 1. Журнал `systemd` і власні повідомлення

**Теорія:** модуль 15, розділи 3 і 4.

### Мета

Навчитися фільтрувати журнал `journald` і записувати власні події.

### Порядок виконання

**Крок 1. Огляд журналу.**

```bash
journalctl --disk-usage
ls -ld /var/log/journal
journalctl --list-boots
journalctl -b -n 20 --no-pager
```

**Крок 2. Фільтри за пріоритетом, службою, часом і ядром.**

```bash
journalctl -p err -b --no-pager | tail -n 15
journalctl -p warning..err -b -n 10 --no-pager
journalctl -u systemd-logind -n 10 --no-pager
journalctl --since "1 hour ago" --until "5 minutes ago" -p warning --no-pager | tail -n 10
journalctl -k -n 10 --no-pager
```

**Крок 3. Формати виведення.**

```bash
journalctl -n 3 -o short-iso --no-pager
journalctl -n 1 -o verbose --no-pager
journalctl -n 1 -o json-pretty --no-pager
```

**Крок 4. Запишіть власні події.**

```bash
logger -t lab08 -p user.info "Початок лабораторної роботи 8"
logger -t lab08 -p user.warning "Тестове попередження"
logger -t lab08 -p user.err "Тестова помилка"
logger -t lab08 -p auth.notice "Тестове повідомлення автентифікації"
journalctl -t lab08 -o short-iso --no-pager
journalctl -t lab08 -p warning --no-pager
journalctl -t lab08 -n 1 -o verbose --no-pager
```

**Крок 5. Порівняйте з текстовими журналами (якщо вони є).**

```bash
ls -l /var/log/syslog /var/log/auth.log
sudo grep lab08 /var/log/syslog
sudo grep lab08 /var/log/auth.log
```

Зверніть увагу, у якому з файлів опинилося повідомлення з категорією `auth`.

### Результати

Заповніть таблицю.

| № | Параметр | Значення |
|---|---|---|
| 1 | Обсяг журналу й кількість збережених завантажень | |
| 2 | Чи постійний журнал (`/var/log/journal`) | |
| 3 | Три помилки (`-p err`) поточного завантаження: час, джерело, текст | |
| 4 | Записи служби `systemd-logind` (одне-два) | |
| 5 | Рядки з `PRIORITY` для чотирьох повідомлень `lab08` | |
| 6 | Поля `_UID`, `_PID`, `SYSLOG_IDENTIFIER` одного повідомлення (`verbose`) | |
| 7 | У яких файлах `/var/log` знайдено повідомлення `lab08` | |

### Запитання для аналізу

1. Чим журнал `journald` відрізняється від текстових журналів `rsyslog`?
2. Чому `-p warning` не показав повідомлення рівня `info`?
3. Чому повідомлення з категорією `auth` потрапило в `auth.log`, а решта — у `syslog`?
4. Чим `short-iso` кращий за `short` для аналізу?
5. Чому `logger` краще за власні файли журналів у скриптах?
6. Що означає відсутність каталогу `/var/log/journal`?

---

## Завдання 2. Файли журналів, облік входів і ротація в системі

**Теорія:** модуль 15, розділи 3.2, 6 і 7.

### Мета

Навчитися читати права на журнали, переглядати облік входів і розуміти системні налаштування ротації.

### Порядок виконання

Усі команди цього завдання лише читають інформацію.

**Крок 1. Файли й права.**

```bash
ls -l /var/log | head -n 25
id -Gn
stat -c '%A %U:%G %n' /var/log/syslog /var/log/auth.log /var/log/wtmp /var/log/btmp /var/log/lastlog
tail -n 3 /var/log/auth.log
```

Якщо `tail` завершилася помилкою, поясніть її за `ls -l` та `id`.

**Крок 2. Облік входів.**

```bash
last -n 10
last -x | head -n 10
lastlog | head -n 10
who -b
w
sudo lastb | head -n 10
```

**Крок 3. Ротація й обмеження.**

```bash
grep -vE '^(#|$)' /etc/logrotate.conf
ls /etc/logrotate.d
cat /etc/logrotate.d/rsyslog
systemctl list-timers | grep logrotate
ls /var/log | grep -E '\.[0-9]+(\.gz)?$'
grep -vE '^(#|$)' /etc/systemd/journald.conf
```

### Результати

Заповніть таблицю.

| № | Параметр | Значення |
|---|---|---|
| 1 | Власник, група й права `auth.log`, `syslog`, `wtmp`, `btmp` | |
| 2 | Чи читаєте ви `auth.log` без `sudo` і чому | |
| 3 | Останній вхід кожного з трьох користувачів (`lastlog`) | |
| 4 | Час останнього перезавантаження | |
| 5 | Результат `sudo lastb` (кількість невдалих спроб) | |
| 6 | Періодичність і кількість копій ротації `rsyslog` | |
| 7 | Таймер `logrotate`: наступний запуск | |
| 8 | Приклади ротованих файлів (`.1`, `.gz`) | |
| 9 | Налаштування `journald.conf` (непорожні рядки) | |

### Запитання для аналізу

1. Чому для читання більшості журналів потрібна група `adm`?
2. Чим `last` відрізняється від `lastb` і чому `lastb` потребує `sudo`?
3. Що показує `lastlog` і навіщо адміністратору цей облік?
4. Яка роль таймера `logrotate`?
5. Чому журнали потрібно ротувати й обмежувати обсяг?
6. Що станеться з системою, якщо диск заповниться журналами?

---

## Завдання 3. Аналіз журналу веб-сервера

**Теорія:** модуль 15, розділи 5.2–5.5.

### Мета

Навчитися аналізувати текстовий журнал за допомогою конвеєрів і робити обґрунтовані висновки.

### Порядок виконання

**Крок 1. Створіть тестовий журнал.**

```bash
cd ~/lab08/logs
cat > access.log <<'EOF'
192.0.2.44 - - [12/Oct/2026:10:00:07 +0300] "GET / HTTP/1.1" 200 612 "-" "curl/8.5.0"
192.0.2.44 - - [12/Oct/2026:10:00:17 +0300] "GET /about.html HTTP/1.1" 200 1045 "-" "curl/8.5.0"
198.51.100.7 - - [12/Oct/2026:10:00:30 +0300] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0"
198.51.100.7 - - [12/Oct/2026:10:00:46 +0300] "GET /index.html HTTP/1.1" 200 612 "-" "Mozilla/5.0"
198.51.100.7 - - [12/Oct/2026:10:00:54 +0300] "GET /api/data HTTP/1.1" 200 2048 "-" "Mozilla/5.0"
198.51.100.23 - - [12/Oct/2026:10:01:05 +0300] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0"
198.51.100.23 - - [12/Oct/2026:10:01:19 +0300] "GET /missing.html HTTP/1.1" 404 162 "-" "Mozilla/5.0"
203.0.113.5 - - [12/Oct/2026:10:01:36 +0300] "GET /wp-login.php HTTP/1.1" 404 162 "-" "python-requests/2.31"
203.0.113.5 - - [12/Oct/2026:10:01:45 +0300] "GET /admin HTTP/1.1" 403 150 "-" "python-requests/2.31"
203.0.113.5 - - [12/Oct/2026:10:01:57 +0300] "GET /admin/config.php HTTP/1.1" 404 162 "-" "python-requests/2.31"
203.0.113.5 - - [12/Oct/2026:10:02:12 +0300] "GET /.env HTTP/1.1" 404 162 "-" "python-requests/2.31"
203.0.113.5 - - [12/Oct/2026:10:02:19 +0300] "GET /phpmyadmin HTTP/1.1" 404 162 "-" "python-requests/2.31"
203.0.113.5 - - [12/Oct/2026:10:02:29 +0300] "POST /login HTTP/1.1" 401 190 "-" "python-requests/2.31"
203.0.113.5 - - [12/Oct/2026:10:02:42 +0300] "POST /login HTTP/1.1" 401 190 "-" "python-requests/2.31"
203.0.113.5 - - [12/Oct/2026:10:02:58 +0300] "POST /login HTTP/1.1" 401 190 "-" "python-requests/2.31"
203.0.113.5 - - [12/Oct/2026:10:03:06 +0300] "POST /login HTTP/1.1" 200 340 "-" "python-requests/2.31"
203.0.113.9 - - [12/Oct/2026:10:03:17 +0300] "GET /robots.txt HTTP/1.1" 200 68 "-" "Mozilla/5.0"
203.0.113.9 - - [12/Oct/2026:10:03:31 +0300] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0"
203.0.113.9 - - [12/Oct/2026:10:03:48 +0300] "GET /api/data HTTP/1.1" 500 321 "-" "Mozilla/5.0"
203.0.113.9 - - [12/Oct/2026:10:03:57 +0300] "GET /api/data HTTP/1.1" 500 321 "-" "Mozilla/5.0"
198.51.100.7 - - [12/Oct/2026:10:04:09 +0300] "GET /about.html HTTP/1.1" 200 1045 "-" "Mozilla/5.0"
198.51.100.7 - - [12/Oct/2026:10:04:24 +0300] "GET /missing.html HTTP/1.1" 404 162 "-" "Mozilla/5.0"
198.51.100.23 - - [12/Oct/2026:10:04:31 +0300] "GET /api/data HTTP/1.1" 200 2048 "-" "Mozilla/5.0"
192.0.2.44 - - [12/Oct/2026:10:04:41 +0300] "GET /index.html HTTP/1.1" 200 612 "-" "curl/8.5.0"
192.0.2.44 - - [12/Oct/2026:10:04:54 +0300] "GET /api/data HTTP/1.1" 200 2048 "-" "curl/8.5.0"
198.51.100.23 - - [12/Oct/2026:10:05:10 +0300] "POST /login HTTP/1.1" 200 340 "-" "Mozilla/5.0"
192.0.2.44 - - [12/Oct/2026:10:05:18 +0300] "GET /about.html HTTP/1.1" 304 0 "-" "curl/8.5.0"
198.51.100.7 - - [12/Oct/2026:10:05:29 +0300] "GET /index.html HTTP/1.1" 304 0 "-" "Mozilla/5.0"
203.0.113.9 - - [12/Oct/2026:10:05:43 +0300] "GET /missing.html HTTP/1.1" 404 162 "-" "Mozilla/5.0"
203.0.113.5 - - [12/Oct/2026:10:06:00 +0300] "GET /robots.txt HTTP/1.1" 200 68 "-" "python-requests/2.31"
EOF
wc -l access.log
head -n 3 access.log
```

Формат рядка (`combined`): `$1` — IP, `$4` — час, `$6`–`$8` — запит, `$9` — код відповіді, `$10` — розмір.

**Крок 2. Загальна статистика.**

```bash
wc -l access.log
cut -d' ' -f1 access.log | sort -u | wc -l
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn
awk '{print $9}' access.log | sort | uniq -c | sort -rn
awk '{s += $10} END {print s}' access.log
awk '{print substr($4,14,5)}' access.log | sort | uniq -c
```

**Крок 3. Помилки.**

```bash
awk '$9 >= 400' access.log | wc -l
awk '$9 >= 500' access.log | wc -l
awk '$9 == 404 {print $7}' access.log | sort | uniq -c | sort -rn
awk '$9 >= 400 && $9 < 500 {print $1}' access.log | sort | uniq -c | sort -rn
awk '{t++; if ($9 >= 400) e++} END {printf "%.1f%%\n", e*100/t}' access.log
```

**Крок 4. Дослідіть найактивнішу адресу.**

```bash
grep '^203.0.113.5 ' access.log | wc -l
grep '^203.0.113.5 ' access.log | awk '{print $7}' | sort | uniq -c | sort -rn
grep '^203.0.113.5 ' access.log | awk -F'"' '{print $6}' | sort | uniq -c
grep 'POST /login' access.log | awk '{print $1, $9}' | sort | uniq -c
grep '^203.0.113.5 .*POST /login' access.log | awk '{print $4, $9}'
```

**Крок 5. Підсумуйте у вигляді таблиці та текстового висновку.**

### Результати

Заповніть таблицю.

| № | Показник | Значення | Команда |
|---|---|---|---|
| 1 | Кількість запитів | | |
| 2 | Кількість унікальних IP | | |
| 3 | Адреса з найбільшою кількістю запитів і кількість | | |
| 4 | Розподіл кодів відповіді | | |
| 5 | Кількість помилок клієнта (4xx) і сервера (5xx) | | |
| 6 | Частка запитів із помилками, % | | |
| 7 | Адреса з найбільшою кількістю помилок 4xx | | |
| 8 | Шлях, що найчастіше повертає `404` | | |
| 9 | Сумарний обсяг відповідей, байтів | | |
| 10 | `User-Agent` найактивнішої адреси | | |
| 11 | Послідовність відповідей на `POST /login` для найактивнішої адреси | | |

Додатково підготуйте **висновок** (5–8 речень): що ви виявили в журналі, які запити виглядають підозріло, чому, які ознаки свідчать про сканування чи підбір пароля, який ризик це становить і що слід зробити.

### Запитання для аналізу

1. Чому перед `uniq -c` необхідний `sort`?
2. Які запити від адреси `203.0.113.5` вказують на сканування вразливостей?
3. Що означає послідовність `401, 401, 401, 200` для `POST /login`?
4. Чому код `500` від адреси `203.0.113.9` має іншу природу, ніж `404`?
5. Чим може відрізнятися поведінка програми-бота від браузера за полем `User-Agent`?
6. Які заходи ви запропонували б для захисту (три-чотири)?

---

## Завдання 4. Розслідування за журналом автентифікації

**Теорія:** модуль 15, розділи 7 і 14.

### Мета

Навчитися відновлювати хронологію інциденту за журналом автентифікації й розпізнавати ознаки підбору паролів та компрометації.

### Порядок виконання

**Крок 1. Створіть тестовий журнал.**

```bash
cd ~/lab08/logs
cat > auth.log <<'EOF'
2026-10-12T09:58:11.402113+03:00 vm sshd[2210]: Accepted publickey for student from 192.0.2.44 port 50622 ssh2: ED25519 SHA256:Zk3p0lAbCdEfGhIjKlMnOpQrStUvWxYz0123456789a
2026-10-12T09:58:11.455201+03:00 vm sshd[2210]: pam_unix(sshd:session): session opened for user student(uid=1000) by (uid=0)
2026-10-12T10:01:22.118830+03:00 vm sudo:  student : TTY=pts/0 ; PWD=/home/student ; USER=root ; COMMAND=/usr/bin/apt update
2026-10-12T10:01:22.119901+03:00 vm sudo: pam_unix(sudo:session): session opened for user root(uid=0) by student(uid=1000)
2026-10-12T10:01:25.771002+03:00 vm sudo: pam_unix(sudo:session): session closed for user root
2026-10-12T10:07:03.100412+03:00 vm sshd[2301]: Invalid user admin from 203.0.113.5 port 41022
2026-10-12T10:07:05.215930+03:00 vm sshd[2301]: Failed password for invalid user admin from 203.0.113.5 port 41022 ssh2
2026-10-12T10:07:08.332118+03:00 vm sshd[2303]: Invalid user admin from 203.0.113.5 port 41030
2026-10-12T10:07:10.450276+03:00 vm sshd[2303]: Failed password for invalid user admin from 203.0.113.5 port 41030 ssh2
2026-10-12T10:07:13.561447+03:00 vm sshd[2305]: Invalid user admin from 203.0.113.5 port 41044
2026-10-12T10:07:15.678590+03:00 vm sshd[2305]: Failed password for invalid user admin from 203.0.113.5 port 41044 ssh2
2026-10-12T10:07:19.790733+03:00 vm sshd[2307]: Failed password for root from 203.0.113.5 port 41052 ssh2
2026-10-12T10:07:22.901880+03:00 vm sshd[2309]: Failed password for root from 203.0.113.5 port 41060 ssh2
2026-10-12T10:07:26.013021+03:00 vm sshd[2311]: Failed password for root from 203.0.113.5 port 41066 ssh2
2026-10-12T10:07:29.124160+03:00 vm sshd[2311]: Connection closed by authenticating user root 203.0.113.5 port 41066 [preauth]
2026-10-12T10:08:41.230305+03:00 vm sshd[2315]: Invalid user test from 198.51.100.23 port 52114
2026-10-12T10:08:43.341452+03:00 vm sshd[2315]: Failed password for invalid user test from 198.51.100.23 port 52114 ssh2
2026-10-12T10:08:47.452598+03:00 vm sshd[2317]: Failed password for invalid user test from 198.51.100.23 port 52120 ssh2
2026-10-12T10:09:30.563740+03:00 vm sshd[2320]: Failed password for student from 203.0.113.5 port 41100 ssh2
2026-10-12T10:09:34.674886+03:00 vm sshd[2322]: Failed password for student from 203.0.113.5 port 41106 ssh2
2026-10-12T10:09:38.786031+03:00 vm sshd[2324]: Accepted password for student from 203.0.113.5 port 41112 ssh2
2026-10-12T10:09:38.830175+03:00 vm sshd[2324]: pam_unix(sshd:session): session opened for user student(uid=1000) by (uid=0)
2026-10-12T10:10:12.941320+03:00 vm sudo:  student : TTY=pts/1 ; PWD=/home/student ; USER=root ; COMMAND=/usr/bin/cat /etc/shadow
2026-10-12T10:10:12.942410+03:00 vm sudo: pam_unix(sudo:session): session opened for user root(uid=0) by student(uid=1000)
2026-10-12T10:10:12.980550+03:00 vm sudo: pam_unix(sudo:session): session closed for user root
2026-10-12T10:11:05.093690+03:00 vm sudo:  student : TTY=pts/1 ; PWD=/home/student ; USER=root ; COMMAND=/usr/sbin/useradd -m -s /bin/bash support
2026-10-12T10:11:05.150820+03:00 vm useradd[2399]: new user: name=support, UID=1001, GID=1001, home=/home/support, shell=/bin/bash, from=/dev/pts/1
2026-10-12T10:11:40.264960+03:00 vm sudo:  student : TTY=pts/1 ; PWD=/home/student ; USER=root ; COMMAND=/usr/sbin/usermod -aG sudo support
2026-10-12T10:11:40.318100+03:00 vm usermod[2410]: add 'support' to group 'sudo'
2026-10-12T10:12:15.425240+03:00 vm sshd[2324]: pam_unix(sshd:session): session closed for user student
2026-10-12T10:15:02.536380+03:00 vm sudo: pam_unix(sudo:auth): authentication failure; logname=student uid=1000 euid=0 tty=/dev/pts/0 ruser=student rhost=  user=student
2026-10-12T10:15:08.647520+03:00 vm sudo:  student : 1 incorrect password attempt ; TTY=pts/0 ; PWD=/home/student ; USER=root ; COMMAND=/usr/bin/true
EOF
wc -l auth.log
head -n 3 auth.log
```

Журнал містить записи про вхід і дії в системі `vm`. Адреси є зарезервованими адресами документації.

**Крок 2. Невдалі спроби.**

```bash
grep -c 'Failed password' auth.log
grep 'Failed password' auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn
grep 'Failed password' auth.log | awk '{ if ($7 == "invalid") print $9; else print $7 }' | sort | uniq -c | sort -rn
grep -c 'Invalid user' auth.log
grep -oE 'Invalid user [a-z]+' auth.log | sort | uniq -c
```

**Крок 3. Часові межі атаки.**

```bash
grep 'Failed password' auth.log | grep 203.0.113.5 | head -n 1 | cut -c1-19
grep 'Failed password' auth.log | grep 203.0.113.5 | tail -n 1 | cut -c1-19
grep 'Failed password' auth.log | grep -c 203.0.113.5
grep 203.0.113.5 auth.log | cut -c1-19,33-120
```

**Крок 4. Успішні входи.**

```bash
grep Accepted auth.log | cut -c1-19,33-130
grep Accepted auth.log | awk '{print $1, $5, $7, $9}'
```

**Крок 5. Що відбувалося після успішного входу з підозрілої адреси.**

```bash
awk '/Accepted password/ {f=1} f' auth.log | cut -c1-19,33-140
grep -o 'COMMAND=.*' auth.log
grep -E 'useradd\[|usermod\[' auth.log
```

**Крок 6. Зіставте з журналом веб-сервера.**

```bash
grep -h 203.0.113.5 access.log auth.log | wc -l
```

### Результати

Заповніть таблицю.

| № | Показник | Значення |
|---|---|---|
| 1 | Загальна кількість `Failed password` | |
| 2 | Невдалі спроби за адресами | |
| 3 | Імена користувачів, на які націлено спроби (з кількістю) | |
| 4 | Кількість повідомлень `Invalid user` | |
| 5 | Час першої й останньої невдалої спроби від `203.0.113.5` | |
| 6 | Успішні входи (час, користувач, адреса, метод) | |
| 7 | Команди `sudo`, виконані після підозрілого входу | |
| 8 | Створені користувачі й зміни груп | |
| 9 | Збіг адреси з журналом веб-сервера | |

Складіть **хронологію інциденту**:

| Час | Подія | Доказ (рядок журналу) | Значення |
|---|---|---|---|
| | | | |

Складіть **короткий звіт про інцидент** (розділ 12.2 модуля 16): короткий опис, хронологія, ознаки компрометації, першопричина, **упорядкований** перелік дій реагування (що зробити спершу, а що потім), профілактика.

### Запитання для аналізу

1. Які ознаки вказують на підбір паролів?
2. Чому успішний вхід `student` після невдалих спроб — ключовий індикатор?
3. Які дії після входу свідчать про компрометацію (читання `shadow`, нові користувачі, `sudo`)?
4. Чому створення користувача `support` і додавання його до `sudo` небезпечні?
5. У якому порядку слід діяти в реальній ситуації: ізоляція, збереження доказів, блокування, відновлення?
6. Які налаштування SSH і брандмауера могли б запобігти інциденту?

---

## Завдання 5. Реальні події: `sudo` і входи

**Теорія:** модуль 15, розділ 7.

### Мета

Навчитися знаходити в реальних журналах системи власні дії й зіставляти їх із форматами із завдання 4.

### Порядок виконання

**Крок 1. Створіть події.**

```bash
sudo -k
echo 'wrong-password' | sudo -S -p '' true
echo "$?"
sudo -v
sudo true
sudo ls /root > /dev/null
sudo cat /etc/shadow > /dev/null
logger -t lab08 -p auth.warning "Подія автентифікації для аналізу"
```

Перша команда з неправильним паролем має завершитися помилкою; зафіксуйте повідомлення. Дві останні `sudo`-команди не виводять вміст, але фіксуються в журналі.

**Крок 2. Знайдіть події в журналі.**

```bash
journalctl _COMM=sudo --since "10 minutes ago" -o short-iso --no-pager
journalctl _COMM=sudo --since "10 minutes ago" --no-pager | grep -o 'COMMAND=.*'
journalctl -t sudo --since "10 minutes ago" --no-pager | grep -i 'authentication failure\|incorrect password'
sudo grep sudo /var/log/auth.log | tail -n 12
```

Якщо `auth.log` відсутній, зазначте це й скористайтеся `journalctl`.

**Крок 3. Облік входів.**

```bash
last -n 5
lastlog -u "$USER"
who
journalctl -u ssh -n 10 --no-pager
```

### Результати

Заповніть таблицю.

| № | Параметр | Значення |
|---|---|---|
| 1 | Повідомлення й код завершення невдалого `sudo` | |
| 2 | Запис журналу про невдалу автентифікацію `sudo` | |
| 3 | Записи `COMMAND=` для ваших трьох команд | |
| 4 | Поля запису `sudo`: користувач, `TTY`, `PWD`, `USER` | |
| 5 | Ваш останній вхід (`last`, `lastlog`) | |
| 6 | Записи служби SSH (або повідомлення про відсутність) | |

### Запитання для аналізу

1. Які відомості про дії користувача фіксує `sudo`?
2. Чим запис про невдалу автентифікацію `sudo` відрізняється від запису про виконання команди?
3. Що спільного й відмінного між цими записами й записами з тестового журналу із завдання 4?
4. Чому важливо, щоб користувачі не могли змінювати журнали?
5. Навіщо адміністратору регулярно переглядати записи `sudo` та входи?

---

## Завдання 6. Ротація журналів

**Теорія:** модуль 15, розділ 6.

### Мета

Навчитися налаштовувати й перевіряти ротацію журналів за допомогою `logrotate`.

### Порядок виконання

**Крок 1. Підготуйте журнал і конфігурацію.**

```bash
mkdir -p ~/lab08/rot
cd ~/lab08/rot
for i in $(seq 1 30); do echo "рядок $i журналу застосунку, заповнювач заповнювач заповнювач"; done > app.log
cat > rot.conf <<EOF
$PWD/app.log {
    size 1k
    rotate 3
    compress
    missingok
    notifempty
    create 0640
}
EOF
cat rot.conf
ls -l
```

Зверніть увагу: конфігурація містить **абсолютний** шлях, який підставила оболонка.

**Крок 2. Режим налагодження (без змін).**

```bash
logrotate -d -s "$PWD/state" rot.conf
ls -l
```

**Крок 3. Перша ротація.**

```bash
logrotate -f -s "$PWD/state" rot.conf
ls -l
cat state
```

**Крок 4. Ще три ротації з новими даними.**

```bash
for i in 2 3 4; do
  for j in $(seq 1 30); do echo "новий рядок $i-$j заповнювач заповнювач заповнювач заповнювач" >> app.log; done
  logrotate -f -s "$PWD/state" rot.conf
  echo "--- після ротації $i"
  ls -l
done
zcat app.log.1.gz | head -n 2
zcat app.log.3.gz | head -n 2
```

**Крок 5. Умова ротації за розміром.**

```bash
echo "малий запис" >> app.log
logrotate -s "$PWD/state" rot.conf
ls -l
```

### Результати

Заповніть таблицю.

| № | Параметр | Значення |
|---|---|---|
| 1 | Що показав режим `-d` (рядки про рішення) | |
| 2 | Файли після першої ротації (імена, розміри, права) | |
| 3 | Файли після четвертої ротації | |
| 4 | Перший рядок `app.log.1.gz` і `app.log.3.gz` | |
| 5 | Що сталося на кроці 5 (ротація без `-f`) і чому | |
| 6 | Вміст файла стану | |

### Запитання для аналізу

1. Чому після ротації новий `app.log` порожній, а старий вміст в `app.log.1.gz`?
2. Чому ніколи не накопичується більше трьох копій?
3. Що означають директиви `missingok`, `notifempty`, `create`?
4. Для чого файл стану й параметр `-s`?
5. Чому на кроці 5 ротація не виконалася?
6. Чому ротація й обмеження обсягу важливі для безпеки й стабільності?

---

## Завдання 7. Аудит безпеки

**Теорія:** модуль 15, розділи 8–12; модуль 14.

### Мета

Навчитися виконувати базовий аудит безпеки, інтерпретувати результати й давати рекомендації.

### Порядок виконання

**Крок 1. Створіть скрипт аудиту.**

```bash
cd ~/lab08/scripts
cat > audit.sh <<'EOF'
#!/usr/bin/env bash
# audit.sh — базовий аудит безпеки (лише читання). Запускати через sudo.
# Код завершення: 0 — попереджень немає, 1 — є попередження, 2 — помилка використання.
# Без set -e: аудит має виконати ВСІ перевірки, навіть якщо окремі завершаться помилкою.
set -uo pipefail

warnings=0
ok()      { printf '[ OK ] %s\n' "$1"; }
warn()    { printf '[WARN] %s\n' "$1"; warnings=$(( warnings + 1 )); }
info()    { printf '[INFO] %s\n' "$1"; }
section() { printf '\n== %s ==\n' "$1"; }

[[ $EUID -eq 0 ]] || { echo "Потрібні права root: запустіть через sudo" >&2; exit 2; }

section "Облікові записи"
mapfile -t zero < <(awk -F: '$3 == 0 {print $1}' /etc/passwd)
if [[ ${#zero[@]} -eq 1 && "${zero[0]}" == "root" ]]; then
  ok "Єдиний обліковий запис із UID 0: root"
else
  warn "Облікові записи з UID 0: ${zero[*]}"
fi
mapfile -t empty < <(awk -F: '($2 == "") {print $1}' /etc/shadow)
if [[ ${#empty[@]} -eq 0 ]]; then
  ok "Облікових записів із порожнім паролем немає"
else
  warn "Порожній пароль: ${empty[*]}"
fi
info "Облікових записів з інтерактивною оболонкою: $(grep -vcE '(nologin|false)$' /etc/passwd)"
info "Члени групи sudo: $(getent group sudo | cut -d: -f4)"

section "SSH"
if command -v sshd > /dev/null 2>&1; then
  cfg="$(sshd -T 2>/dev/null || true)"
  prl="$(awk '$1 == "permitrootlogin" {print $2}' <<< "$cfg")"
  pa="$(awk '$1 == "passwordauthentication" {print $2}' <<< "$cfg")"
  if [[ "$prl" == "no" || "$prl" == "prohibit-password" ]]; then
    ok "PermitRootLogin = $prl"
  else
    warn "PermitRootLogin = ${prl:-невідомо}"
  fi
  if [[ "$pa" == "no" ]]; then
    ok "PasswordAuthentication = no"
  else
    info "PasswordAuthentication = ${pa:-невідомо} (для серверів рекомендовано вхід за ключами)"
  fi
else
  info "SSH-сервер не встановлено"
fi

section "Мережа"
info "Сокетів, що слухають: $(ss -H -tuln | wc -l)"
ss -H -tuln | awk '$5 !~ /^(127\.|\[::1\]|::1)/ {print "       назовні: " $1, $5}'
if command -v ufw > /dev/null 2>&1 && ufw status | grep -q '^Status: active'; then
  ok "Брандмауер ufw активний"
else
  warn "Брандмауер ufw неактивний або не встановлений"
fi

section "Файлова система"
info "Файлів із SUID у /usr, /bin, /sbin: $(find /usr /bin /sbin -xdev -type f -perm -4000 2>/dev/null | wc -l)"
ww="$(find /etc -xdev -type f -perm -0002 2>/dev/null | wc -l)"
if [[ "$ww" -eq 0 ]]; then ok "У /etc немає файлів, доступних для запису всім"; else warn "Файлів у /etc із записом для всіх: $ww"; fi
wd="$(find / -xdev -type d -perm -0002 ! -perm -1000 2>/dev/null | wc -l)"
if [[ "$wd" -eq 0 ]]; then ok "Каталогів із записом для всіх без sticky-біта немає"; else warn "Каталогів із записом для всіх без sticky: $wd"; fi
if [[ "$(stat -c %a /tmp)" == 1* ]]; then ok "/tmp має sticky-біт"; else warn "/tmp без sticky-біта"; fi
nu="$(find /etc /usr /home -xdev \( -nouser -o -nogroup \) 2>/dev/null | wc -l)"
if [[ "$nu" -eq 0 ]]; then ok "Файлів без власника немає"; else warn "Файлів без власника чи групи: $nu"; fi

section "Оновлення й цілісність"
upg="$(apt list --upgradable 2>/dev/null | tail -n +2 | wc -l)"
if [[ "$upg" -eq 0 ]]; then ok "Доступних оновлень немає"; else warn "Доступних оновлень: $upg"; fi
if [[ -f /var/run/reboot-required ]]; then warn "Потрібне перезавантаження"; else ok "Перезавантаження не потрібне"; fi
if systemctl is-enabled unattended-upgrades > /dev/null 2>&1; then ok "unattended-upgrades увімкнено"; else warn "unattended-upgrades вимкнено"; fi
bad="$(dpkg -V sudo coreutils bash 2>/dev/null | awk '$2 != "c" && $0 !~ /\/usr\/share\/(doc|man|locale|info)\//' | wc -l)"
if [[ "$bad" -eq 0 ]]; then ok "Файли sudo, coreutils, bash не змінено"; else warn "dpkg -V виявив розбіжності: $bad"; fi

section "Служби й система"
failed="$(systemctl --failed --no-legend 2>/dev/null | wc -l)"
if [[ "$failed" -eq 0 ]]; then ok "Збійних юнітів немає"; else warn "Збійних юнітів: $failed"; fi
if aa-status --enabled > /dev/null 2>&1; then ok "AppArmor увімкнено"; else warn "AppArmor вимкнено або недоступний"; fi
if [[ "$(timedatectl show -p NTPSynchronized --value 2>/dev/null)" == "yes" ]]; then ok "Час синхронізовано"; else warn "Час не синхронізовано"; fi

printf '\nПопереджень: %d\n' "$warnings"
(( warnings == 0 ))
EOF
chmod +x audit.sh
bash -n audit.sh
```

**Крок 2. Прочитайте скрипт перед запуском від `root`.**

```bash
cat audit.sh
```

Скрипт лише читає інформацію. Запишіть, чому в ньому немає `set -e` і для чого функції `ok`, `warn`, `info`.

**Крок 3. Запустіть аудит.**

```bash
sudo ~/lab08/scripts/audit.sh | tee ~/lab08/audit-report.txt
echo "${PIPESTATUS[0]}"
```

**Крок 4. Перевірте вручну три попередження (`WARN`) із вашого звіту.**

Для кожного підтвердьте результат окремою командою. Приклади: `sudo ufw status verbose`, `apt list --upgradable 2>/dev/null | wc -l`, `systemctl --failed`, `sudo aa-status | head`, `ls /var/run/reboot-required`, `timedatectl`, `find / -xdev -type d -perm -0002 ! -perm -1000 2>/dev/null`.

**Крок 5. Додайте до скрипта дві власні перевірки.**

Додайте до `audit.sh` (перед підсумком) **дві** нові перевірки з використанням функцій `ok`, `warn`, `info`. Наприклад:

- права `/etc/shadow` не повинні перевищувати `640`;
- права `/etc/passwd` не повинні дозволяти запис іншим;
- кількість користувачів із оболонкою входу;
- наявність записів у `/root/.ssh/authorized_keys`;
- значення `MaxAuthTries` у `sshd -T` (якщо SSH-сервер встановлено).

Після змін:

```bash
bash -n audit.sh
sudo ~/lab08/scripts/audit.sh | tail -n 15
```

Наведіть у звіті текст доданих перевірок.

**Крок 6. Складіть план усунення.**

Для кожного `WARN` вкажіть пріоритет (критичний, високий, середній, низький), рекомендовану дію й ризик дії. Змін у системі **не вносьте**.

### Результати

Додайте до звіту **повний звіт аудиту** (`~/lab08/audit-report.txt`) і заповніть таблицю.

| № | Розділ аудиту | Результати (`OK`, `WARN`, `INFO`) | Висновок |
|---|---|---|---|
| 1 | Облікові записи | | |
| 2 | SSH | | |
| 3 | Мережа | | |
| 4 | Файлова система | | |
| 5 | Оновлення й цілісність | | |
| 6 | Служби й система | | |

Заповніть **план усунення**.

| № | Попередження | Пріоритет | Рекомендована дія | Ризик дії |
|---|---|---|---|---|
| | | | | |

Запишіть також: код завершення скрипта, число попереджень, дві додані вами перевірки.

### Запитання для аналізу

1. Чому аудит виконано без `set -e`?
2. Чому скрипт вимагає `root` і чому його слід прочитати перед запуском?
3. Що означає попередження про неактивний брандмауер і чому в Ubuntu `ufw` за замовчуванням вимкнений?
4. Чому до перевірки цілісності `dpkg -V` відфільтровано рядки для файлів конфігурації й документації?
5. Яке з попереджень найкритичніше й чому?
6. Чому не слід одразу виправляти всі попередження, не оцінивши ризиків?

---

## Завдання 8. Комплексна діагностика

**Теорія:** модуль 16, розділи 3–7, 11 і 13.

### Мета

Навчитися діагностувати кілька несправностей за методикою, фіксувати докази й складати звіт.

### Підготовка сценарію

Виконайте команди підготовки. **Не аналізуйте їх як відповідь**: у реальній роботі ви не знаєте, що саме зламано, а бачите лише симптоми.

```bash
cd ~/lab08/trouble
sudo -v

yes > /dev/null &

sudo mkdir -p /mnt/lab08-small
sudo mount -t tmpfs -o size=20M tmpfs /mnt/lab08-small
sudo chown "$(id -u):$(id -g)" /mnt/lab08-small
mkdir -p /mnt/lab08-small/cache
head -c 100M /dev/zero > /mnt/lab08-small/cache/old-dump.bin 2>/dev/null || true

sudo tee /usr/local/bin/lab08-app.sh > /dev/null <<'EOF'
#!/bin/bash
cfg=/etc/lab08-app.conf
if [[ ! -r "$cfg" ]]; then
  echo "lab08-app: не вдалося прочитати конфігурацію $cfg" >&2
  exit 1
fi
while true; do
  echo "lab08-app працює, режим: $(cat "$cfg")"
  sleep 5
done
EOF
sudo chmod 755 /usr/local/bin/lab08-app.sh
sudo tee /etc/systemd/system/lab08-app.service > /dev/null <<'EOF'
[Unit]
Description=Lab 08 demo application

[Service]
ExecStart=/usr/local/bin/lab08-app.sh
User=nobody
Restart=on-failure
RestartSec=1

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl start lab08-app || true

printf '<h1>old server</h1>\n' > ~/lab08/trouble/index.html
python3 -m http.server 8090 --bind 127.0.0.1 --directory ~/lab08/trouble > ~/lab08/trouble/server-old.log 2>&1 &
sleep 12
```

### Симптоми, повідомлені користувачами

| № | Скарга |
|---|---|
| S1 | «Віртуальна машина стала повільною, вентилятор шумить.» |
| S2 | «Додаток не може записати файли в `/mnt/lab08-small`: пише `No space left on device`.» |
| S3 | «Служба `lab08-app` не працює.» |
| S4 | «Мій веб-сервер на порту 8090 не запускається.» |

### Порядок виконання

**Крок 1. Знімок стану («перші хвилини»).**

Виконайте стандартний огляд і збережіть результат:

```bash
cd ~/lab08/trouble
cat > triage.sh <<'EOF'
#!/usr/bin/env bash
set -uo pipefail
section() { printf '\n== %s ==\n' "$1"; }
section "Час і uptime";   date; uptime
section "Збійні юніти";   systemctl --failed --no-legend --no-pager
section "Помилки поточного завантаження"
journalctl -p err -b -n 15 --no-pager
section "Процесор: топ-5";  ps -eo pid,user,stat,%cpu,%mem,comm --sort=-%cpu | head -n 6
section "Пам'ять";        free -h
section "Диски";          df -hT -x devtmpfs -x squashfs; df -i /mnt/lab08-small | tail -n 1
section "Мережа";         ip -br addr; ss -tlnp 2>/dev/null | head -n 12
EOF
chmod +x triage.sh
./triage.sh | tee triage-before.txt
```

**Крок 2. Діагностуйте кожен симптом.**

Для кожного симптому виконайте цикл діагностики (модуль 16, розділ 3.1): збір даних, гіпотези, перевірка, мінімальне виправлення, контроль. Фіксуйте **команди й докази**. Підказки (за потреби):

- **S1:** `uptime`, `nproc`, `top`, `ps --sort=-%cpu`, `ps -o pid,user,etime,args -p PID`.
- **S2:** `df -h`, `df -i`, `du -h -d 2`, `ls -la`, `find -size`.
- **S3:** `systemctl status`, `journalctl -u`, `systemctl cat`, `ls -l`, `namei`.
- **S4:** `python3 -m http.server 8090 --bind 127.0.0.1 --directory ~/lab08/web`, `ss -tlnp`, `ps -o pid,ppid,user,etime,args -p PID`.

Для S4 запустіть **власний** сервер із каталогу `~/lab08/web` (створіть у ньому `index.html`) тільки після діагностики й усунення причини.

**Крок 3. Виправлення.**

Для кожного симптому виконайте **мінімальну** зміну, а потім перевірку:

- S1: завершіть відповідальний процес м'яко (`SIGTERM`) і перевірте `uptime`/`ps`.
- S2: видаліть саме те, що заповнило файлову систему, і перевірте можливість запису (`printf 'ok\n' > /mnt/lab08-small/test.txt`).
- S3: усуньте причину відмови (не змінюючи скрипт і юніт), потім `sudo systemctl reset-failed lab08-app`, `sudo systemctl start lab08-app`, перевірте `status` і журнал.
- S4: з'ясуйте, що займає порт, і вирішіть, чи можна безпечно завершити цей процес; потім запустіть свій сервер і перевірте `curl -s http://127.0.0.1:8090/`.

**Крок 4. Підсумковий знімок.**

```bash
cd ~/lab08/trouble
./triage.sh | tee triage-after.txt
```

Порівняйте `triage-before.txt` і `triage-after.txt`.

### Результати

Заповніть таблицю для **кожного** симптому.

| Параметр | S1 | S2 | S3 | S4 |
|---|---|---|---|---|
| Ключові команди діагностики | | | | |
| Доказ (фрагмент виводу) | | | | |
| Першопричина | | | | |
| Виправлення (команди) | | | | |
| Контроль (команди й результат) | | | | |
| Профілактика | | | | |

Складіть **post-mortem** (розділ 12.2 модуля 16) для інциденту **S3**: короткий опис, хронологія (за часом із журналу), причина, виправлення, що спрацювало, профілактика. Додайте результат аналізу «п'ять чому».

### Запитання для аналізу

1. Який порядок діагностики ви обрали й чому?
2. Як ви відрізнили проблему процесора від проблем диска й пам'яті (`wa`, `r`, `available`)?
3. Чому S2 не усунути командою `df`, і що саме показав `du`/`ls`?
4. Що показав журнал служби `lab08-app` і чому служба перейшла в `start-limit-hit`?
5. Як ви визначили процес, який займає порт у S4, і як вирішили, чи можна його завершити?
6. Чому не варто було виправляти проблеми «усіма способами одразу»?
7. Що змінилося між `triage-before.txt` і `triage-after.txt`?

---

## Завдання 9. Загальний аналіз і прибирання

**Теорія:** модулі 15 і 16.

### Мета

Підсумувати результати роботи й **повністю відновити** початковий стан системи.

### Частина A. Загальний аналіз

На основі виконаних завдань підготуйте короткий опис результатів.

У відповіді зазначте:

1. Які інструменти й підходи ви використали для аналізу журналів і що кожен із них дав.
2. Які ознаки атаки й компрометації ви виявили в тестових журналах і як їх можна запобігти.
3. Які результати показав аудит безпеки й які дії ви рекомендували б першочергово.
4. Як ротація й обмеження журналів впливають на стабільність і безпеку.
5. Яка методика діагностики виявилася найефективнішою в комплексному сценарії.
6. Які знання з яких модулів курсу ви використали в завданні 8 (хоча б п'ять різних модулів).
7. Які помилки виникали під час виконання роботи й чим вони були зумовлені.
8. Які відмінності між вашим середовищем і прикладами в завданнях було виявлено.

### Частина B. Обов'язкове прибирання

**Крок 1. Завершіть процеси.**

```bash
jobs
pgrep -a yes
pgrep -af 'http.server'
```

Завершіть лише процеси, запущені вами в цій роботі (`pkill -x yes`; процеси `python3 -m http.server` — за PID командою `kill PID`).

**Крок 2. Видаліть службу й пов'язані файли.**

```bash
sudo systemctl stop lab08-app
ls /etc/systemd/system | grep lab08
ls /usr/local/bin | grep lab08
ls /etc/lab08-app.conf
```

Перегляньте списки: у них мають бути лише ваші файли `lab08`. Потім (файл конфігурації видаляйте лише якщо ви його створили):

```bash
sudo rm -f /etc/systemd/system/lab08-app.service /usr/local/bin/lab08-app.sh /etc/lab08-app.conf
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

**Крок 3. Відмонтуйте тимчасову файлову систему.**

```bash
cd ~
findmnt /mnt/lab08-small
sudo umount /mnt/lab08-small
sudo rmdir /mnt/lab08-small
ls /mnt
```

**Крок 4. Видаліть робочий каталог.**

```bash
cd ~
ls -ld ~/lab08
rm -rI ~/lab08
```

Параметр `-I` запитує підтвердження один раз: відповідайте `y`, лише переконавшись у правильності шляху.

**Крок 5. Перевірте повернення до початкового стану.**

```bash
systemctl --failed --no-legend | wc -l
ls /etc/systemd/system | grep -c lab08
ls /usr/local/bin | grep -c lab08
ls /etc/lab08-app.conf
findmnt | wc -l
ls /mnt
pgrep -c yes
pgrep -fc 'http.server'
ss -tuln | wc -l
jobs
ls ~/lab08
```

Результати мають збігатися із записаними в початковій підготовці. Остання команда має повідомити про відсутність каталогу. Повідомлення `logger` із тегом `lab08` залишаються в системному журналі: вони безпечні й видаляються під час ротації.

Якщо будь-яка перевірка не пройшла, виправте її й повторіть. За потреби поверніть знімок `before-lab08`, але зазначте це у звіті.

---

## Контрольні запитання

1. Які компоненти складають систему журналювання Linux? Чим `journald` відрізняється від `rsyslog`?
2. Які рівні важливості й категорії має системний журнал? Що покаже `journalctl -p err`?
3. Як переглянути журнал конкретної служби, помилки поточного завантаження й останню годину?
4. Для чого потрібні групи `adm` і `systemd-journal`?
5. Для чого `logger` і чому записи із категорією `auth` потрапляють у `auth.log`?
6. Як працює конвеєр `cut | sort | uniq -c | sort -rn`? Чому `sort` потрібен перед `uniq`?
7. Які основи `awk` ви застосовували (`$1`, `NF`, `-F`, шаблон, `END`)?
8. Що таке ротація журналів? Які директиви `logrotate` ви знаєте? Що робить режим `-d`?
9. Що означають записи `Failed password`, `Invalid user`, `Accepted password`?
10. Які ознаки підбору пароля й компрометації системи?
11. У якому порядку діяти після виявлення компрометації?
12. Назвіть основні принципи безпеки Linux.
13. Які перевірки входять до базового аудиту безпеки?
14. Що таке SUID і чому потрібно знати їхній перелік?
15. Для чого `dpkg -V` і що означає позначка `c`?
16. Як безпечно налаштувати `ufw` і SSH, щоб не втратити доступ?
17. Яку роль відіграє AppArmor?
18. Які кроки включає цикл діагностики?
19. Чому змінювати слід лише одну річ за раз і що таке метод USE?
20. Як діагностувати високе навантаження, нестачу місця, збійну службу й конфлікт порту?
21. Чим відрізняються видалені, але відкриті файли від звичайних і як їх знайти?
22. Що означають `203/EXEC`, `217/USER`, `start-limit-hit`?
23. Що має містити звіт про інцидент (post-mortem)?
24. Які основні правила безпеки слід дотримуватися під час аналізу журналів і діагностики?

---

## Що необхідно здати

Звіт подається через Moodle відповідно до вимог викладача.

Звіт повинен містити:

1. Назву лабораторної роботи.
2. Мету роботи.
3. Короткий опис використаного Linux-середовища та підтвердження створення знімка VM.
4. Початковий стан системи.
5. Результати виконання завдань 1–8.
6. Заповнені таблиці з результатами команд.
7. **Висновок за журналом веб-сервера (завдання 3), хронологію й звіт про інцидент (завдання 4).**
8. **Звіт аудиту, текст доданих перевірок і план усунення (завдання 7).**
9. **Таблицю діагностики, знімки `triage-before.txt`/`triage-after.txt` і post-mortem (завдання 8).**
10. Пояснення отриманих результатів і повідомлень про помилки.
11. Відповіді на запитання для аналізу.
12. Загальний аналіз (завдання 9, частина A).
13. **Результати перевірки прибирання (завдання 9, частина B).**
14. Відповіді на контрольні запитання.
15. Загальний висновок.

### Вимоги до оформлення

- Указуйте команди, які фактично виконувалися.
- Для кожного висновку наводьте доказ: команду й фрагмент виводу.
- Додавайте відповідні результати їх виконання.
- Пояснюйте призначення команд, параметрів і елементів скриптів.
- Не замінюйте пояснення скриншотами.
- За потреби додавайте скриншоти термінала як підтвердження виконання.
- Не включайте у звіт паролі, токени, приватні ключі, хеші паролів і особисті дані; вміст `/etc/shadow` не копіюйте.
- Якщо команда завершилася помилкою, зафіксуйте повідомлення та поясніть можливу причину.
- Якщо певна команда або компонент недоступні у вашому дистрибутиві, зазначте це та, за можливості, наведіть альтернативу.

---

## Безпека

!!! warning "Правила безпечної роботи"

    - Виконуйте роботу лише у власній навчальній VM; створіть знімок перед початком.
    - Аналізуйте лише тестові журнали з роботи; не публікуйте реальні журнали без очищення від секретів.
    - Спершу спостерігайте, потім змінюйте; змінюйте одну річ за раз і фіксуйте кожну дію.
    - Не змінюйте налаштування брандмауера, SSH, облікові записи й системні файли, окрім об'єктів, названих у роботі.
    - Скрипт аудиту виконується від `root`: прочитайте його перед запуском.
    - Для навантаження використовуйте лише `yes` і маленький `tmpfs`; завершуйте процеси.
    - Не вимикайте засоби захисту (`ufw`, AppArmor) для перевірки.
    - Перед `sudo rm`, `sudo umount`, `sudo systemctl stop` перевіряйте об'єкти й шляхи.
    - Не виконуйте атак, сканування й підбору паролів: журнали з роботи є навчальними.
    - Обов'язково виконайте прибирання (завдання 9) і перевірте його результат.

---

## Критерії оцінювання

Максимальна оцінка за лабораторну роботу — **10 балів**.

| Критерій | Бали |
|---|---:|
| Завдання 1. Журнал `systemd` і власні повідомлення | 1,0 |
| Завдання 2. Файли журналів, облік входів і ротація в системі | 0,75 |
| Завдання 3. Аналіз журналу веб-сервера | 1,0 |
| Завдання 4. Розслідування за журналом автентифікації | 1,0 |
| Завдання 5. Реальні події: `sudo` і входи | 0,5 |
| Завдання 6. Ротація журналів | 0,75 |
| Завдання 7. Аудит безпеки | 1,25 |
| Завдання 8. Комплексна діагностика | 1,75 |
| Завдання 9. Загальний аналіз і прибирання | 0,5 |
| Оформлення звіту та пояснення результатів | 0,75 |
| Відповіді на контрольні запитання | 0,75 |
| **Разом** | **10,0** |

Якщо прибирання (завдання 9, частина B) не виконано або його результат не підтверджено, оцінка за роботу знижується відповідно до правил, установлених викладачем.

### Критерії оцінювання завдання 8

| Показник | Бали |
|---|---:|
| Систематичність: знімок стану, гіпотези, перевірки, одна зміна за раз | 0,4 |
| Правильна першопричина й докази для S1–S4 | 0,6 |
| Мінімальні виправлення й контроль | 0,35 |
| Post-mortem для S3 | 0,4 |
| **Разом** | **1,75** |

### Загальні вимоги до оцінювання

- **Високий рівень:** висновки обґрунтовані доказами, розрахунки й підрахунки вірні, інцидент відновлено послідовно, аудит інтерпретовано з пріоритетами, діагностика систематична; прибирання виконано та підтверджено.
- **Достатній рівень:** основні завдання виконані, але окремі висновки не підкріплені доказами або неповні.
- **Початковий рівень:** результати наведені без пояснень і доказів, висновки хибні чи відсутні, діагностика виконувалась хаотично, студент не може пояснити призначення використаних команд.

---

## Висновок

У висновку необхідно коротко зазначити:

- які навички було отримано під час роботи із журналами, аудитом і діагностикою;
- які інструменти й команди виявилися найкориснішими;
- як ви відновили хронологію інциденту за журналами й які індикатори були ключовими;
- як методика діагностики допомогла усунути несправності;
- які ризики несе аналіз журналів і діагностика та як ви їх мінімізували;
- як навички з усіх модулів курсу поєдналися в комплексному завданні;
- які питання залишилися незрозумілими та потребують додаткового опрацювання.
