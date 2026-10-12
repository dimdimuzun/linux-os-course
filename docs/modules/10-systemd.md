# Модуль 10. Systemd

## 1. Навчальні цілі

Після опрацювання модуля студент повинен уміти:

- пояснювати роль `systemd`, юнітів і цілей та описувати послідовність завантаження Linux;
- розрізняти типи юнітів: `service`, `socket`, `timer`, `target`, `mount` та інші;
- керувати службами за допомогою `systemctl`: `status`, `start`, `stop`, `restart`, `reload`, `enable`, `disable`, `mask`;
- розрізняти запуск служби й увімкнення її автозапуску, стани `active`, `inactive`, `failed`, `enabled`, `disabled`;
- інтерпретувати виведення `systemctl status`;
- знаходити й читати файли юнітів, розуміти пріоритет каталогів і drop-in-налаштування;
- створювати власні `.service`-юніти, використовуючи директиви `[Unit]`, `[Service]`, `[Install]`;
- пояснювати політику перезапуску й зв'язок `systemd` із сигналами та cgroups;
- створювати й перевіряти таймери як альтернативу `cron`;
- переглядати журнали служб за допомогою `journalctl`;
- застосовувати алгоритм діагностики несправної служби;
- аналізувати завантаження системи за допомогою `systemd-analyze`;
- безпечно працювати зі службами, не порушуючи роботу системи.

## 2. Вступ

Коли Linux завантажується, ядро запускає один процес із PID 1 (модуль 8), який породжує решту процесів і підтримує систему в робочому стані. У сучасних дистрибутивах, зокрема в Ubuntu, цю роль виконує **systemd**. Він запускає служби під час завантаження, стежить за ними, перезапускає при збоях, веде їхні журнали, виконує завдання за розкладом і керує зупинкою системи.

**Служба** (service) — це фоновий процес або група процесів, які виконують певну функцію: веб-сервер, планувальник, мережевий демон. Служби не прив'язані до терміналу; адміністратор керує ними не через `&` і `kill` (модуль 8), а через `systemctl`, який забезпечує єдиний інтерфейс.

Модуль спирається на попередні: процеси й сигнали (модуль 8), користувачі (модуль 7), права доступу (модуль 6), пакети, що встановлюють служби (модуль 9), файлова система й `/etc`, `/usr/lib` (модуль 3). Для зміни стану служб потрібні права адміністратора, тому вправи виконуються **лише в навчальній VM**.

Приклади виведення є умовними: у вашому середовищі результати можуть відрізнятися.

## 3. Основні поняття

### 3.1. `systemd`, юніти та cgroups

`systemd` оперує **юнітами** (units) — описами об'єктів, якими він керує. Кожен юніт має ім'я з суфіксом, що вказує на тип:

| Тип | Суфікс | Призначення |
|---|---|---|
| Служба | `.service` | Процес або група процесів (веб-сервер, `cron`) |
| Ціль | `.target` | Група юнітів, що визначає стан системи (`multi-user.target`) |
| Таймер | `.timer` | Запуск іншого юніта за розкладом |
| Сокет | `.socket` | Запуск служби при першому зверненні до сокета |
| Монтування | `.mount` | Точка монтування (модуль 3) |
| Шлях | `.path` | Запуск при зміні файла чи каталогу |
| Пристрій | `.device` | Пристрої, відомі ядру |
| Зріз | `.slice` | Група ресурсів (cgroup) |

`systemd` використовує **cgroups** (control groups) ядра: усі процеси служби об'єднуються в окрему групу. Завдяки цьому він завжди знає, які процеси належать службі, і може зупинити їх усі, навіть якщо вони створили власні дочірні процеси.

### 3.2. Служби та процеси

Служба — це не окремий вид процесу, а спосіб його запуску й керування:

- `systemd` запускає процес служби, є його батьком і відстежує код завершення;
- `systemctl stop` надсилає сигнал `SIGTERM`, а якщо процес не завершується протягом певного часу, — `SIGKILL` (модуль 8);
- служба може працювати від імені певного користувача (`User=`), як правило, не `root` (модуль 7);
- вивід служби (stdout, stderr) за замовчуванням потрапляє в журнал `journald`.

Ознаки процесів служб видно в `ps`, `top` та `systemctl status`.

## 4. Процес завантаження та цілі

### 4.1. Етапи завантаження

1. **Мікропрограма** (BIOS або UEFI) ініціалізує обладнання та запускає завантажувач.
2. **Завантажувач** (GRUB) завантажує ядро й початковий образ у пам'ять.
3. **Ядро** ініціалізує обладнання, монтує кореневу файлову систему.
4. **`initramfs`** — тимчасова файлова система, яка допомагає підготувати доступ до кореня.
5. **`systemd`** (PID 1) запускає юніти й досягає цільового стану `default.target`.

```bash
ps -p 1 -o pid,ppid,user,comm
systemctl get-default
```

### 4.2. Цілі

**Ціль** (target) — це точка синхронізації, яка визначає набір юнітів, що мають працювати. Основні цілі:

| Ціль | Призначення |
|---|---|
| `multi-user.target` | Багатокористувацький режим без графічного інтерфейсу (типова ціль сервера) |
| `graphical.target` | Режим із графічним інтерфейсом (типова ціль робочої станції) |
| `rescue.target` | Однокористувацький аварійний режим |
| `emergency.target` | Мінімальна оболонка для відновлення |
| `poweroff.target`, `reboot.target` | Вимкнення й перезавантаження |
| `network.target`, `timers.target` | Допоміжні цілі синхронізації |

```bash
systemctl get-default
systemctl list-units --type=target
systemctl list-dependencies multi-user.target | head -n 15
```

Змінювати типову ціль (`systemctl set-default`) або перемикатися на іншу (`systemctl isolate rescue.target`) слід лише в навчальному середовищі: це змінює стан усієї системи.

## 5. Файли юнітів

### 5.1. Розташування

| Каталог | Призначення | Пріоритет |
|---|---|---|
| `/etc/systemd/system` | Юніти й налаштування адміністратора; сюди ж потрапляють символічні посилання автозапуску | Найвищий |
| `/run/systemd/system` | Тимчасові юніти поточного сеансу | Середній |
| `/usr/lib/systemd/system` | Юніти, що постачаються пакетами (у Ubuntu `/lib/systemd/system` є посиланням на нього) | Найнижчий |

Якщо юніт із таким самим ім'ям є в кількох каталогах, застосовується файл із вищим пріоритетом. **Файли в `/usr/lib/systemd/system` не редагують**: пакет перезапише зміни. Налаштування адміністратора розміщують у `/etc/systemd/system`.

```bash
systemctl cat cron.service
systemctl show -p FragmentPath cron.service
ls /etc/systemd/system | head
```

Команда `systemctl cat` показує повний ефективний текст юніта разом із drop-in-файлами й шляхами до них.

### 5.2. Drop-in-налаштування

Щоб змінити окремі параметри юніта, не копіюючи весь файл, створюють drop-in у каталозі `/etc/systemd/system/імя.service.d/`:

```bash
sudo systemctl edit nginx.service
```

Команда відкриває редактор і створює файл `override.conf`. Для повної заміни юніта існує `sudo systemctl edit --full імя.service`. Після змін `systemctl edit` виконує перечитування автоматично.

### 5.3. Автозапуск: символічні посилання

Автозапуск (`enable`) реалізований символічними посиланнями. Для юніта з `WantedBy=multi-user.target` команда `enable` створює посилання в каталозі `/etc/systemd/system/multi-user.target.wants/`:

```bash
ls -l /etc/systemd/system/multi-user.target.wants | head
```

Нагадування: посилання — це файл-покажчик (модулі 3 і 5), тому `disable` просто видаляє його.

## 6. Керування службами: `systemctl`

Перегляд стану зазвичай не потребує прав адміністратора. Для зміни стану (`start`, `stop`, `enable`, …) потрібен `sudo` (модуль 7).

### 6.1. Основні команди

| Команда | Дія |
|---|---|
| `systemctl status служба` | Докладний стан |
| `sudo systemctl start служба` | Запустити зараз |
| `sudo systemctl stop служба` | Зупинити зараз |
| `sudo systemctl restart служба` | Зупинити й запустити |
| `sudo systemctl reload служба` | Перечитати конфігурацію без зупинки (якщо служба підтримує) |
| `sudo systemctl enable служба` | Увімкнути автозапуск під час завантаження |
| `sudo systemctl disable служба` | Вимкнути автозапуск |
| `sudo systemctl enable --now служба` | Увімкнути автозапуск і запустити одразу |
| `sudo systemctl disable --now служба` | Вимкнути автозапуск і зупинити одразу |
| `systemctl is-active служба` | Чи працює зараз |
| `systemctl is-enabled служба` | Чи ввімкнено автозапуск |
| `systemctl is-failed служба` | Чи завершилася збоєм |
| `sudo systemctl mask служба` | Повністю заборонити запуск (посилання на `/dev/null`) |
| `sudo systemctl unmask служба` | Зняти заборону |
| `sudo systemctl daemon-reload` | Перечитати файли юнітів після їх змін |
| `sudo systemctl reset-failed` | Скинути стан `failed` |

Якщо назва не містить суфікса, `systemctl` припускає `.service`.

### 6.2. Запуск і автозапуск — різні речі

| Дія | `start` / `stop` | `enable` / `disable` |
|---|---|---|
| Що змінює | Поточний стан зараз | Поведінку під час наступного завантаження |
| Діє до перезавантаження? | Так | Зберігається |
| Запускає службу зараз? | Так | Ні (лише з `--now`) |

Комбінації: служба може бути запущена, але без автозапуску; або мати автозапуск, але бути зараз зупиненою.

### 6.3. Читання виведення `status`

```bash
systemctl status cron.service --no-pager
```

```text
● cron.service - Regular background program processing daemon
     Loaded: loaded (/usr/lib/systemd/system/cron.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-10-12 09:01:12 EEST; 1h 20min ago
       Docs: man:cron(8)
   Main PID: 782 (cron)
      Tasks: 1 (limit: 4547)
     Memory: 1.2M (peak: 1.9M)
        CPU: 120ms
     CGroup: /system.slice/cron.service
             └─782 /usr/sbin/cron -f -P

Oct 12 10:17:01 vm CRON[2450]: (root) CMD (cd / && run-parts --report /etc/cron.hourly)
```

| Рядок | Значення |
|---|---|
| Кружок `●` | Колір показує стан: зелений — активна, червоний — збій, білий — неактивна |
| `Loaded` | Шлях до файла юніта; `enabled`/`disabled`; пресет за замовчуванням |
| `Active` | Стан і час його початку |
| `Main PID` | PID головного процесу (модуль 8) |
| `Tasks`, `Memory`, `CPU` | Ресурси групи процесів служби |
| `CGroup` | Дерево процесів служби |
| Останні рядки | Свіжі записи журналу служби |

### 6.4. Стани

Стан активності (`Active`):

| Стан | Значення |
|---|---|
| `active (running)` | Процес служби працює |
| `active (exited)` | Одноразове завдання успішно виконано (`Type=oneshot` з `RemainAfterExit`) |
| `active (waiting)` | Очікує події (таймер) |
| `inactive (dead)` | Не запущена |
| `activating` / `deactivating` | Запускається / зупиняється |
| `failed` | Завершилася помилкою або порушила умови |

Стан автозапуску (`is-enabled`):

| Стан | Значення |
|---|---|
| `enabled` | Є посилання для автозапуску |
| `disabled` | Немає |
| `static` | Не має секції `[Install]`, запускається як залежність інших юнітів або вручну |
| `masked` | Заблокований |
| `indirect` / `alias` | Увімкнений іншим юнітом або псевдонімом |

### 6.5. Перелік юнітів

```bash
systemctl list-units --type=service
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service | head -n 20
systemctl --failed
```

- `list-units` показує завантажені юніти та їхній стан;
- `list-unit-files` — усі файли юнітів і стан автозапуску;
- `--failed` — юніти зі збоєм.

### 6.6. Властивості: `systemctl show`

```bash
systemctl show -p MainPID,ActiveState,SubState cron.service
systemctl show -p MainPID --value cron.service
```

Параметр `--value` виводить лише значення: його зручно підставляти в інші команди, наприклад `ps -p "$(systemctl show -p MainPID --value cron.service)"`.

## 7. Створення власної служби

### 7.1. Структура `.service`-файла

Файл юніта складається з секцій і директив у форматі `Ключ=Значення`:

```ini
[Unit]
Description=Demo service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/demo.sh
User=nobody
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
```

| Секція | Призначення |
|---|---|
| `[Unit]` | Опис і зв'язки з іншими юнітами |
| `[Service]` | Як запускати й зупиняти процес |
| `[Install]` | Куди підключати при `enable` |

### 7.2. Основні директиви

| Директива | Значення |
|---|---|
| `Description=` | Короткий опис (його видно в `status`) |
| `After=`, `Before=` | Порядок запуску відносно інших юнітів |
| `Wants=`, `Requires=` | М'яка й жорстка залежності |
| `Type=` | Спосіб запуску (див. нижче) |
| `ExecStart=` | Команда запуску (**абсолютний шлях**) |
| `ExecStartPre=`, `ExecStop=`, `ExecReload=` | Команди перед запуском, при зупинці, при перечитуванні |
| `User=`, `Group=` | Від імені кого запускати |
| `WorkingDirectory=` | Робочий каталог |
| `Environment=` | Змінні середовища |
| `Restart=` | Політика перезапуску |
| `RestartSec=` | Затримка перед перезапуском |
| `TimeoutStopSec=` | Скільки чекати завершення після `SIGTERM` перед `SIGKILL` |
| `WantedBy=` | Ціль, яка «хоче» цю службу при `enable` |

Типи служб (`Type=`):

| Тип | Поведінка |
|---|---|
| `simple` | За замовчуванням; процес `ExecStart` є головним процесом служби |
| `oneshot` | Виконує одне завдання й завершується; зручний для скриптів і таймерів |
| `forking` | Процес створює копію й завершується, копія залишається як демон |
| `notify` | Служба сама повідомляє `systemd` про готовність |

Політика `Restart=`:

| Значення | Перезапуск |
|---|---|
| `no` | Ніколи (за замовчуванням) |
| `on-failure` | Якщо ненульовий код завершення, завершення сигналом або тайм-аут |
| `on-abnormal` | Лише при завершенні сигналом або тайм-ауті |
| `always` | Завжди, навіть після чистого завершення |

Нескінченні перезапуски обмежуються: якщо служба завершується частіше, ніж дозволяють налаштування (за замовчуванням 5 перезапусків за 10 секунд), `systemd` припиняє спроби зі станом `start-limit-hit`. Скинути його можна командою `systemctl reset-failed`.

### 7.3. Особливості `ExecStart`

`ExecStart` не передає рядок оболонці. Тому:

- шлях до програми має бути **абсолютним**;
- конвеєри `|`, перенаправлення `>`, шаблони `*`, підстановка `$(...)` й змінні `$HOME` не працюють так, як у терміналі (модуль 4);
- якщо потрібні можливості оболонки, використовують `ExecStart=/bin/bash -c '...'`.

Програма, яку запускає служба, має бути **виконуваною** для користувача, від імені якого працює служба (модуль 6): файл повинен мати право `x`, а на всіх каталогах шляху — `x` для цього користувача.

### 7.4. Розміщення й активація

Власні юніти розміщують у `/etc/systemd/system`. Створити файл від імені `root` без редактора можна за допомогою `tee` (модуль 7):

```bash
sudo tee /etc/systemd/system/demo.service > /dev/null <<'EOF'
[Unit]
Description=Demo service

[Service]
ExecStart=/usr/local/bin/demo.sh
User=nobody

[Install]
WantedBy=multi-user.target
EOF
```

Порядок дій після створення або зміни юніта:

```bash
systemd-analyze verify /etc/systemd/system/demo.service
sudo systemctl daemon-reload
sudo systemctl start demo.service
systemctl status demo.service --no-pager
sudo systemctl enable demo.service
```

- `systemd-analyze verify` перевіряє синтаксис і доступність виконуваних файлів;
- `daemon-reload` **обов'язковий** після будь-якої зміни файла юніта;
- `enable` створює посилання для автозапуску; `start` запускає зараз.

### 7.5. Обмеження та захист

Служби можна обмежити за допомогою директив безпеки:

| Директива | Ефект |
|---|---|
| `User=`, `Group=` | Працювати без прав `root` |
| `NoNewPrivileges=true` | Заборонити отримання нових привілеїв (зокрема через `setuid`) |
| `PrivateTmp=true` | Окремий `/tmp` для служби |
| `ProtectSystem=strict` | Файлова система лише для читання, крім дозволених шляхів |
| `ProtectHome=true` | Закрити домашні каталоги |
| `MemoryMax=`, `CPUQuota=` | Обмеження ресурсів |

Оцінити, наскільки служба захищена, допомагає `systemd-analyze security імя.service`.

## 8. Зупинка служби, сигнали та коди завершення

### 8.1. Що відбувається при `stop`

1. `systemd` виконує `ExecStop=`, якщо вона є;
2. надсилає `SIGTERM` головному процесу (за замовчуванням — усім процесам групи);
3. чекає `TimeoutStopSec` (за замовчуванням 90 секунд);
4. якщо процеси лишилися, надсилає `SIGKILL`.

Це реалізація порядку «м'яко, потім жорстко», розглянутого в модулі 8. Надіслати довільний сигнал службі можна командою:

```bash
sudo systemctl kill --signal=SIGHUP імя.service
```

### 8.2. Збій служби та перезапуск

Якщо процес служби завершиться через `SIGKILL` (наприклад, `sudo kill -9 PID`), `systemd` розцінить це як збій. При `Restart=on-failure` він автоматично запустить службу знову, а в журналі з'являться записи на кшталт:

```text
lab.service: Main process exited, code=killed, status=9/KILL
lab.service: Failed with result 'signal'.
lab.service: Scheduled restart job, restart counter is at 1.
```

Якщо службу зупинено командою `systemctl stop`, це чисте завершення, і перезапуск не відбувається.

### 8.3. Типові коди збоїв у `status`

| Запис | Значення | Типова причина |
|---|---|---|
| `status=203/EXEC` | Не вдалося запустити `ExecStart` | Файл не існує, не виконуваний, невірний шлях, немає права `x` |
| `status=217/USER` | Неможливо визначити користувача | Користувача з `User=` не існує |
| `status=200/CHDIR` | Помилка зміни каталогу | Немає `WorkingDirectory` або доступу до нього |
| `status=1/FAILURE` (та інші) | Програма завершилася з ненульовим кодом | Помилка самої програми: дивіться журнал |
| `status=9/KILL`, `status=15/TERM` | Процес завершено сигналом | `kill`, тайм-аут, нестача пам'яті |
| `start-limit-hit` | Забагато швидких перезапусків | Служба постійно падає; усуньте причину й виконайте `reset-failed` |

Коди `203`, `217` тощо — це коди системи `systemd`, а не коди завершення програми (модуль 4).

## 9. Таймери

**Таймер** (`.timer`) запускає пов'язаний юніт (зазвичай `.service` з такою самою назвою) за розкладом. Це альтернатива `cron`, яка інтегрується з журналом і керуванням залежностями.

Приклад пари файлів:

```ini
# /etc/systemd/system/report.service
[Unit]
Description=Report job

[Service]
Type=oneshot
ExecStart=/usr/local/bin/report.sh
User=nobody
```

```ini
# /etc/systemd/system/report.timer
[Unit]
Description=Run report periodically

[Timer]
OnActiveSec=10s
OnUnitActiveSec=30s

[Install]
WantedBy=timers.target
```

Основні директиви секції `[Timer]`:

| Директива | Значення |
|---|---|
| `OnBootSec=` | Через скільки часу після завантаження системи |
| `OnActiveSec=` | Через скільки часу після активації самого таймера |
| `OnUnitActiveSec=` | Через скільки часу після останнього запуску служби |
| `OnCalendar=` | За календарним виразом (`Mon *-*-* 09:00`, `daily`) |
| `Persistent=true` | Виконати пропущений запуск після завантаження |

Керування й перевірка:

```bash
sudo systemctl daemon-reload
sudo systemctl start report.timer
systemctl list-timers
systemctl status report.timer --no-pager
systemd-analyze calendar 'Mon *-*-* 09:00'
```

Запускати треба **таймер** (`.timer`), а не службу. Команда `systemd-analyze calendar` нормалізує вираз і показує наступний запуск, тому її зручно використовувати для перевірки розкладів. У системі вже є таймери, наприклад для оновлення списків пакетів і очищення тимчасових файлів:

```bash
systemctl list-timers --all | head -n 10
```

## 10. Журнали: `journalctl`

`systemd` має власну службу журналювання `journald`, яка збирає повідомлення ядра, служб і системи. Читають її командою `journalctl` (докладно в модулі 15).

```bash
journalctl -u cron.service -n 10 --no-pager
journalctl -u cron.service --since "1 hour ago" --no-pager
journalctl -b -p err -n 10 --no-pager
journalctl -f
```

| Параметр | Дія |
|---|---|
| `-u юніт` | Записи конкретного юніта |
| `-n N` | Останні N записів |
| `-f` | Стежити за новими записами (зупинка — `Ctrl+C`) |
| `-b` | Лише поточне завантаження |
| `-p err` | Лише повідомлення рівня `err` і серйозніші |
| `--since`, `--until` | Часовий проміжок |
| `-x`, `-e` | Додаткові пояснення, перехід у кінець |
| `--no-pager` | Вивід без `less` |

Права доступу (модулі 6–7): користувачі, які входять до груп `adm` або `systemd-journal`, бачать повні журнали; інші — лише власні повідомлення. Тому додавання користувача до групи `adm` надає йому доступ до журналів.

## 11. Аналіз завантаження та інформація про систему

```bash
systemd-analyze
systemd-analyze blame | head -n 10
systemd-analyze critical-chain | head -n 15
hostnamectl
timedatectl
```

| Команда | Дія |
|---|---|
| `systemd-analyze` | Загальний час завантаження (ядро, userspace) |
| `systemd-analyze blame` | Юніти за часом запуску |
| `systemd-analyze critical-chain` | Ланцюжок юнітів, що визначає загальну тривалість |
| `hostnamectl` | Ім'я хоста й відомості про систему |
| `timedatectl` | Час, дата, часовий пояс, синхронізація |

Завершення роботи системи також керується через `systemctl`: `sudo systemctl reboot`, `sudo systemctl poweroff`. Такі команди виконуйте свідомо й лише в навчальному середовищі.

## 12. Діагностика несправної служби

Якщо служба не запускається або падає, діють за порядком:

1. **Стан:** `systemctl status служба --no-pager` — `Active`, код `status=`, останні рядки журналу.
2. **Журнал:** `journalctl -u служба -n 30 --no-pager` — повідомлення про причину (наприклад, `Permission denied`, `No such file`).
3. **Юніт:** `systemctl cat служба` — що саме запускається, від кого, з якими параметрами; `systemd-analyze verify файл`.
4. **Виконуваний файл:** `ls -l шлях`, `namei -l шлях` — чи існує, чи є `x` (модулі 3 і 6).
5. **Користувач:** `getent passwd користувач` — чи існує `User=` (модуль 7).
6. **Ручний запуск:** запустіть команду з `ExecStart` від того ж користувача (`sudo -u користувач команда`) і подивіться на результат.
7. **Залежності й порти:** чи не зайнято ресурс іншим процесом (`ps`, модуль 8; порти — модуль 11).
8. **Після виправлення:** `sudo systemctl daemon-reload` (якщо змінювали юніт), `sudo systemctl reset-failed`, `sudo systemctl start служба`, повторна перевірка `status`.

Правило: **не змінюйте кілька речей одночасно** й завжди читайте повідомлення журналу, а не вгадуйте причину.

## 13. Типові помилки

1. **Забули `daemon-reload` після зміни юніта.** `systemd` працює зі старою версією файла.
2. **Редагування файлів у `/usr/lib/systemd/system`.** Зміни зникнуть після оновлення пакета; використовуйте `/etc/systemd/system` або `systemctl edit`.
3. **Плутанина між `start` та `enable`.** Одне не включає інше.
4. **Відносний шлях у `ExecStart`.** Потрібен абсолютний.
5. **Використання `|`, `>` або `$VAR` у `ExecStart` без `bash -c`.** Вони не працюють як у терміналі.
6. **Скрипт без права `x` або з недоступним для `User=` каталогом.** Результат — `203/EXEC`.
7. **Вказаний користувач не існує.** Результат — `217/USER`.
8. **Запуск служби командою `&` у терміналі замість `systemd`.** Така служба не керується, не перезапускається й не веде журнал.
9. **`Restart=always` без `RestartSec`.** Швидкі цикли перезапуску й `start-limit-hit`.
10. **Відключення або маскування незнайомих служб.** Можна втратити доступ до системи.

!!! warning "Навчальне середовище"

    Змінювати стан служб, створювати юніти й цілі слід лише в лабораторній VM. Не вимикайте, не маскуйте й не зупиняйте системні служби (`systemd-logind`, `dbus`, мережеві служби, `ssh`) без розуміння наслідків. Не змінюйте `default.target` і не виконуйте `systemctl isolate`, `reboot`, `poweroff`, крім випадків, вказаних завданням.

## 14. Security Note

!!! danger

    Служби часто працюють із правами `root` та приймають зовнішні дані. Помилка в налаштуванні служби або її юніта може відкрити доступ до системи чи вивести її з ладу. Файли юнітів і значення директив видимі іншим користувачам.

Правила безпечної роботи:

1. Запускайте служби від імені окремого непривілейованого користувача (`User=`), а не від `root`.
2. Не зберігайте паролі, токени й ключі в файлах юнітів і в `Environment=`: файли читаються всіма, а значення видно в `systemctl show`.
3. Файли юнітів у `/etc/systemd/system` мають належати `root` і не бути доступними для запису іншим (модуль 6).
4. Виконувані файли служб і каталоги шляху не повинні бути доступними для запису непривілейованим користувачам.
5. Використовуйте директиви захисту: `NoNewPrivileges`, `PrivateTmp`, `ProtectSystem`, `ProtectHome`.
6. Вимикайте й видаляйте непотрібні служби: менше служб — менше вразливостей.
7. Не маскуйте й не зупиняйте служби безпеки (оновлень, журналювання, міжмережевого екрана) без причини.
8. Не змінюйте незнайомі системні юніти; для налаштувань використовуйте drop-in.
9. Переглядайте журнали служб після збоїв: повторювані помилки можуть свідчити про атаку.
10. Не запускайте невідомі скрипти як служби.

## 15. Практичні вправи

Вправи 1–3 та 5–6 лише читають інформацію або змінюють тільки вашу демонстраційну службу. У вправі 7 виконується прибирання.

### Вправа 1. `systemd` як PID 1

```bash
ps -p 1 -o pid,ppid,user,comm
systemctl get-default
systemctl status cron.service --no-pager
systemctl is-active cron.service
systemctl is-enabled cron.service
```

Завдання:

1. Запишіть, який процес має PID 1.
2. Розберіть рядки `Loaded`, `Active`, `Main PID`, `CGroup`.
3. Поясніть різницю між `is-active` і `is-enabled`.

### Вправа 2. Перегляд юнітів

```bash
systemctl list-units --type=service --state=running | head -n 15
systemctl list-unit-files --type=service | head -n 15
systemctl --failed
systemctl list-units --type=target
```

Завдання:

1. Скільки служб зараз запущено?
2. Чи є юніти в стані `failed`?
3. Яка ціль є типовою для вашої системи?

### Вправа 3. Файли юнітів

```bash
systemctl cat cron.service
systemctl show -p FragmentPath cron.service
systemctl show -p MainPID,ActiveState,SubState cron.service
ls -l /etc/systemd/system | head
ls -l /etc/systemd/system/multi-user.target.wants | head
```

Завдання:

1. Знайдіть, де розташований файл юніта `cron.service`.
2. Визначте `ExecStart`, `Type` і `WantedBy`.
3. Поясніть, що таке посилання в `multi-user.target.wants`.

### Вправа 4. Власна служба

```bash
sudo tee /usr/local/bin/demo-hello.sh > /dev/null <<'EOF'
#!/bin/bash
while true; do
  echo "demo tick $(date +%T)"
  sleep 5
done
EOF
sudo chmod 755 /usr/local/bin/demo-hello.sh
sudo tee /etc/systemd/system/demo-hello.service > /dev/null <<'EOF'
[Unit]
Description=Demo hello service

[Service]
ExecStart=/usr/local/bin/demo-hello.sh
User=nobody
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF
systemd-analyze verify /etc/systemd/system/demo-hello.service
sudo systemctl daemon-reload
sudo systemctl start demo-hello
systemctl status demo-hello --no-pager
journalctl -u demo-hello -n 5 --no-pager
ps -u nobody -o pid,ppid,user,stat,args
```

Завдання:

1. Опишіть, що означає кожна директива в юніті.
2. Визначте користувача, від імені якого працює процес.
3. Знайдіть записи служби в журналі.

### Вправа 5. Збій і перезапуск

```bash
PID="$(systemctl show -p MainPID --value demo-hello)"
echo "$PID"
sudo kill -9 "$PID"
sleep 4
systemctl status demo-hello --no-pager
journalctl -u demo-hello -n 12 --no-pager
sudo systemctl stop demo-hello
systemctl status demo-hello --no-pager
```

Завдання:

1. Порівняйте `Main PID` до й після `kill -9`.
2. Знайдіть у журналі запис про завершення сигналом і перезапуск.
3. Поясніть, чому після `systemctl stop` служба не перезапустилась.

### Вправа 6. Автозапуск і таймери

```bash
sudo systemctl enable demo-hello
systemctl is-enabled demo-hello
ls -l /etc/systemd/system/multi-user.target.wants/ | grep demo
sudo systemctl disable demo-hello
systemctl is-enabled demo-hello
systemctl list-timers --all | head -n 10
systemd-analyze calendar 'Mon *-*-* 09:00'
```

Завдання:

1. Зафіксуйте повідомлення `enable` про створення посилання.
2. Поясніть, що змінилося в `is-enabled`.
3. Знайдіть наступний запуск для виразу `Mon *-*-* 09:00`.

### Вправа 7. Прибирання

```bash
sudo systemctl stop demo-hello
sudo systemctl disable demo-hello
sudo rm /etc/systemd/system/demo-hello.service /usr/local/bin/demo-hello.sh
sudo systemctl daemon-reload
sudo systemctl reset-failed
systemctl status demo-hello --no-pager
ls /etc/systemd/system | grep demo
```

Після прибирання `status` має повідомити, що юніт не знайдено, а `ls ... | grep demo` — нічого не виводити.

## 16. Самоперевірка

1. Яку роль виконує `systemd` і чому він має PID 1?
2. Що таке юніт? Які бувають типи юнітів?
3. Що таке cgroup і навіщо `systemd` використовує їх для служб?
4. Які етапи проходить завантаження Linux?
5. Що таке ціль? Чим `multi-user.target` відрізняється від `graphical.target`?
6. Де розташовані файли юнітів і який із каталогів має найвищий пріоритет?
7. Чому не слід редагувати файли в `/usr/lib/systemd/system` і що використовувати натомість?
8. Чим `start` відрізняється від `enable`? Що робить `enable --now`?
9. Що означають стани `active (running)`, `inactive (dead)`, `failed`?
10. Що означають `enabled`, `disabled`, `static`, `masked`?
11. Яку інформацію містить `systemctl status`?
12. З яких секцій складається `.service`-файл і що задає `ExecStart`?
13. Чому в `ExecStart` не працюють `|` та `>`? Як це обійти?
14. Для чого потрібен `daemon-reload`?
15. Що відбувається під час `systemctl stop` з точки зору сигналів?
16. Чим `Restart=on-failure` відрізняється від `Restart=always`?
17. Що означають `203/EXEC` і `217/USER`?
18. Чим таймер `systemd` відрізняється від `cron`? Що запускати: `.timer` чи `.service`?
19. Як переглянути журнал конкретної служби та лише помилки поточного завантаження?
20. У якому порядку діагностують служби, що не запускаються?

## 17. Підготовка до лабораторної роботи

Матеріал цього модуля разом із модулем 9 є підготовкою до **лабораторної роботи № 5 «Пакети та Systemd»**. Студент повинен керувати службами, читати `status` і журнали, створювати власні юніти й таймери, розуміти зв'язок служб із процесами, користувачами й правами та діагностувати збої.

Рекомендовані команди для повторення:

```bash
systemctl status
systemctl start
systemctl stop
systemctl restart
systemctl reload
systemctl enable
systemctl disable
systemctl is-active
systemctl is-enabled
systemctl --failed
systemctl cat
systemctl show -p
systemctl daemon-reload
systemctl reset-failed
systemctl list-timers
systemd-analyze verify
systemd-analyze blame
journalctl -u
journalctl -b -p err
```

Виконайте вправи 1–7 у власній навчальній VM, створіть знімок перед початком і переконайтеся, що вправу 7 (прибирання) виконано.

## 18. Додаткові матеріали

1. [systemd documentation](https://systemd.io/) — офіційний сайт проєкту.
2. [freedesktop.org: systemd man pages](https://www.freedesktop.org/software/systemd/man/latest/) — довідкові сторінки `systemd` (`systemctl`, `systemd.unit`, `systemd.service`, `systemd.timer`).
3. [Ubuntu Server documentation](https://documentation.ubuntu.com/server/) — документація Ubuntu Server.
4. [Linux man-pages](https://www.kernel.org/doc/man-pages/) — довідкова документація Linux.

Для локального ознайомлення використовуйте:

```bash
man systemd
man systemctl
man systemd.unit
man systemd.service
man systemd.timer
man journalctl
man systemd-analyze
```

Посилання на зовнішні ресурси варто перевіряти на актуальність.

## 19. Ключові висновки

1. `systemd` — процес із PID 1, який запускає служби, стежить за ними, перезапускає їх, веде журнали й керує завантаженням.
2. Усім керують юніти: `.service`, `.timer`, `.target`, `.socket` та інші; цілі групують юніти й визначають стан системи.
3. `systemd` об'єднує процеси служби в cgroup, тому знає всі її процеси й завершує їх разом.
4. `start`/`stop` змінюють стан зараз, `enable`/`disable` — поведінку під час завантаження; `enable --now` об'єднує обидва.
5. `systemctl status` показує файл юніта, стан, головний PID, групу процесів і останні записи журналу.
6. Власні юніти розміщують у `/etc/systemd/system`; після змін обов'язковий `daemon-reload`; системні файли в `/usr/lib` не редагують.
7. `ExecStart` потребує абсолютного шляху й не інтерпретується оболонкою; виконуваний файл повинен бути доступним користувачу, від імені якого працює служба.
8. `systemctl stop` реалізує порядок «`SIGTERM`, потім `SIGKILL`»; `Restart=` керує перезапусками після збоїв.
9. Таймери замінюють `cron`: запускається `.timer`, який активує однойменну службу.
10. Діагностика збою: `status` → `journalctl -u` → `systemctl cat` → перевірка файлів, прав і користувача → виправлення → `daemon-reload` → `reset-failed`.
