# Systemd

Докладніше: [модуль 10](../modules/10-systemd.md), журнали — [модуль 15](../modules/15-logs-and-security.md).

## Керування службами

```bash
systemctl status SERVICE
sudo systemctl start SERVICE
sudo systemctl stop SERVICE
sudo systemctl restart SERVICE
sudo systemctl reload SERVICE
sudo systemctl enable SERVICE
sudo systemctl disable SERVICE
sudo systemctl enable --now SERVICE
sudo systemctl disable --now SERVICE
systemctl is-active SERVICE
systemctl is-enabled SERVICE
systemctl --failed
journalctl -u SERVICE
```

| Команда | Дія |
|---|---|
| `start` / `stop` | Змінити **поточний** стан |
| `enable` / `disable` | Змінити поведінку **під час завантаження** (посилання в `…wants/`) |
| `--now` | Об'єднати з `start` / `stop` |
| `restart` / `reload` | Перезапустити (новий PID) / перечитати конфігурацію без зупинки |
| `mask` / `unmask` | Заборонити запуск повністю / зняти заборону |
| `daemon-reload` | Перечитати файли юнітів **після їх змін** |
| `reset-failed` | Скинути стан `failed` (зокрема `start-limit-hit`) |
| `cat`, `show -p MainPID,ActiveState SERVICE` | Ефективний текст юніта; властивості |

## Стани

| Активність (`Active`) | Значення |
|---|---|
| `active (running)` | Процес працює |
| `active (exited)` | Одноразове завдання виконано |
| `inactive (dead)` | Не запущена |
| `activating` / `deactivating` | Запускається / зупиняється |
| `failed` | Збій |

| Автозапуск (`is-enabled`) | Значення |
|---|---|
| `enabled` / `disabled` | Є / немає посилання автозапуску |
| `static` | Немає `[Install]`: запускається як залежність або вручну |
| `masked` | Заблокований |

## Читання `status`

```text
● cron.service - Regular background program processing daemon
     Loaded: loaded (/usr/lib/systemd/system/cron.service; enabled; preset: enabled)
     Active: active (running) since …
   Main PID: 782 (cron)
     CGroup: /system.slice/cron.service
```

`Loaded` — шлях до юніта й автозапуск; `Active` — стан; `Main PID` — головний процес; `CGroup` — дерево процесів служби; нижче — свіжі записи журналу.

## Розташування файлів юнітів

| Каталог | Призначення | Пріоритет |
|---|---|---|
| `/etc/systemd/system` | Юніти й налаштування адміністратора | Найвищий |
| `/run/systemd/system` | Тимчасові юніти | Середній |
| `/usr/lib/systemd/system` | Юніти пакетів (**не редагувати**) | Найнижчий |

Drop-in для зміни окремих параметрів: `sudo systemctl edit SERVICE` (створює `override.conf`).

## Шаблон служби

```ini
[Unit]
Description=Demo service

[Service]
Type=simple
ExecStart=/usr/local/bin/demo.sh
User=nobody
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
```

```bash
systemd-analyze verify /etc/systemd/system/demo.service
sudo systemctl daemon-reload
sudo systemctl start demo.service
systemctl status demo.service --no-pager
```

| Директива | Значення |
|---|---|
| `ExecStart=` | Команда з **абсолютним шляхом**; оболонка не використовується (немає `\|`, `>`, `$VAR`) |
| `Type=` | `simple`, `oneshot`, `forking`, `notify` |
| `User=`, `Group=` | Від імені кого працює служба |
| `Restart=` | `no`, `on-failure`, `on-abnormal`, `always` |
| `RestartSec=`, `TimeoutStopSec=` | Затримка перезапуску; очікування після `SIGTERM` |
| `After=`, `Wants=`, `Requires=` | Порядок і залежності |
| `WantedBy=` | Ціль, яка «хоче» службу при `enable` |

Директиви захисту: `NoNewPrivileges=true`, `PrivateTmp=true`, `ProtectSystem=strict`, `ProtectHome=true`. Оцінка — `systemd-analyze security SERVICE`.

`systemctl stop` виконує `ExecStop=`, надсилає `SIGTERM`, чекає `TimeoutStopSec` і лише потім `SIGKILL`.

## Коди збоїв

| Запис | Причина |
|---|---|
| `203/EXEC` | Файл відсутній, не виконуваний, невірний шлях |
| `217/USER` | Користувача з `User=` не існує |
| `200/CHDIR` | Немає `WorkingDirectory` |
| `1/FAILURE` та ін. | Програма завершилась із помилкою: читайте журнал |
| `status=9/KILL` | Завершено сигналом |
| `start-limit-hit` | Забагато швидких перезапусків: усуньте причину, `reset-failed` |

## Таймери

```ini
# /etc/systemd/system/report.timer
[Timer]
OnActiveSec=10s
OnUnitActiveSec=30s

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl start report.timer
systemctl list-timers
systemd-analyze calendar 'Mon *-*-* 09:00'
```

Запускається **таймер**, який активує однойменну службу (`Type=oneshot`). Директиви: `OnBootSec`, `OnActiveSec`, `OnUnitActiveSec`, `OnCalendar`, `Persistent=true`.

## Цілі й завантаження

```bash
systemctl get-default
systemctl list-units --type=target
systemctl list-dependencies multi-user.target
systemd-analyze
systemd-analyze blame
systemd-analyze critical-chain
hostnamectl
timedatectl
```

| Ціль | Призначення |
|---|---|
| `multi-user.target` | Багатокористувацький режим без графіки |
| `graphical.target` | З графічним інтерфейсом |
| `rescue.target` / `emergency.target` | Відновлення |

Не виконуйте `set-default`, `isolate`, `reboot`, `poweroff` без потреби.

## `journalctl`

```bash
journalctl -u SERVICE -n 50 --no-pager
journalctl -b -p err --no-pager
journalctl -k -n 20
journalctl --since "1 hour ago" -p warning
journalctl _COMM=sudo -n 10
journalctl -t TAG -n 5
journalctl -f
journalctl -n 3 -o short-iso
journalctl --disk-usage
```

| Параметр | Дія |
|---|---|
| `-u юніт` | Записи юніта |
| `-b`, `-b -1`, `--list-boots` | Поточне й попереднє завантаження |
| `-p рівень` | `emerg`, `alert`, `crit`, `err`, `warning`, `notice`, `info`, `debug` (включно з серйознішими); діапазон `warning..err` |
| `-k` | Ядро |
| `--since`, `--until` | Проміжок часу |
| `-t тег`, `_COMM=`, `_UID=`, `_PID=` | Фільтри за полями |
| `-f`, `-n N`, `-r` | Стежити, останні N, найновіші першими |
| `-o short-iso/verbose/json/cat` | Формат виведення |

`logger -t тег -p user.notice "текст"` пише власні події. Читати журнал повністю можуть члени груп `adm` і `systemd-journal`.

## Діагностика служби

1. `systemctl status SERVICE --no-pager` — стан і код.
2. `journalctl -u SERVICE -n 50 --no-pager` — причина.
3. `systemctl cat SERVICE` — що запускається й від кого.
4. `systemd-analyze verify файл`.
5. Файл, права, користувач: `ls -l`, `namei -l`, `getent passwd`.
6. Ручний запуск: `sudo -u користувач команда`.
7. Після виправлення: `daemon-reload` → `reset-failed` → `start` → `status`.
