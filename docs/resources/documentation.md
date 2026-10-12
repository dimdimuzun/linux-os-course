# Документація

## Документація всередині системи

Найшвидше джерело відповіді — сама система. Вона завжди відповідає **вашій** версії програм.

```bash
man ls
man 5 passwd
man 7 signal
man -k слово
ls --help
type ls
info coreutils
```

| Засіб | Призначення |
|---|---|
| `man команда` | Сторінка довідки (розділи: 1 — команди, 5 — формати файлів, 7 — огляди, 8 — адміністрування) |
| `man -k слово` (`apropos`) | Пошук за описом |
| `команда --help` | Короткий опис параметрів |
| `info` | Детальна документація проєктів GNU |
| `/usr/share/doc/пакет` | Додаткова документація пакета |
| `systemctl help сервіс` | Довідка служби |

Сторінки `man` корисно читати за схемою: **NAME → SYNOPSIS → DESCRIPTION → OPTIONS → EXAMPLES / SEE ALSO**. У `man` пошук — `/слово`, вихід — `q`.

!!! warning "Перевіряйте посилання"

    Посилання нижче наведено за пам'яттю автора, без перевірки в Інтернеті. Адреси документації й назви розділів можуть змінюватися, тому в разі помилки шукайте розділ за назвою на головній сторінці ресурсу або скористайтеся локальними `man`-сторінками.

## Загальні довідники

| Ресурс | Адреса | Що містить |
|---|---|---|
| Linux man-pages | <https://man7.org/linux/man-pages/> | Сторінки довідки з розділів 2, 3, 4, 5, 7 (системні виклики, формати файлів, огляди: `hier`, `signal`, `credentials`, `proc`) |
| Linux man-pages (kernel.org) | <https://www.kernel.org/doc/man-pages/> | Проєкт man-pages |
| Ubuntu Documentation | <https://documentation.ubuntu.com/> | Офіційна документація Ubuntu, зокрема Ubuntu Server |
| Filesystem Hierarchy Standard | <https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html> | Стандарт структури каталогів (модуль 3) |
| Arch Wiki | <https://wiki.archlinux.org/> | Дуже докладна й практична вікі, більшість порад застосовна до інших дистрибутивів; перевіряйте відмінності пакетів і шляхів |

## За темами курсу

| Тема | Ресурс | Адреса |
|---|---|---|
| Bash (модулі 4, 14) | GNU Bash Reference Manual | <https://www.gnu.org/software/bash/manual/bash.html> |
| Bash | ShellCheck (аналіз і вікі помилок) | <https://www.shellcheck.net/> |
| Утиліти GNU (модулі 3, 5, 6) | GNU Coreutils Manual | <https://www.gnu.org/software/coreutils/manual/coreutils.html> |
| `find` (модуль 5) | GNU Findutils Manual | <https://www.gnu.org/software/findutils/manual/html_mono/find.html> |
| `tar` (модуль 5) | GNU tar Manual | <https://www.gnu.org/software/tar/manual/tar.html> |
| `awk` (модуль 15) | GNU Awk User's Guide | <https://www.gnu.org/software/gawk/manual/gawk.html> |
| systemd (модуль 10) | Сайт проєкту | <https://systemd.io/> |
| systemd | Man-сторінки (`systemctl`, `systemd.unit`, `systemd.service`, `journalctl`) | <https://www.freedesktop.org/software/systemd/man/latest/> |
| Пакети Debian (модуль 9) | Debian Reference | <https://www.debian.org/doc/manuals/debian-reference/> |
| Snap (модуль 9) | Документація Snap | <https://snapcraft.io/docs> |
| Мережі (модулі 11–12) | Netplan | <https://netplan.readthedocs.io/> |
| Мережі | OpenSSH (`sshd_config`) | <https://www.openssh.com/manual.html> |
| HTTP-клієнт | Документація curl | <https://curl.se/docs/> |
| Захоплення пакетів | tcpdump і libpcap | <https://www.tcpdump.org/> |
| Файлові системи (модуль 13) | Документація ядра: ext4 | <https://docs.kernel.org/admin-guide/ext4.html> |
| Діагностика (модуль 16) | Brendan Gregg: Linux Performance | <https://www.brendangregg.com/linuxperf.html> |
| Діагностика | Сайт `strace` | <https://strace.io/> |

## Навчальні ресурси

| Ресурс | Адреса | Примітка |
|---|---|---|
| The Linux Command Line | <https://linuxcommand.org/> | Безкоштовна книга й практичні матеріали |
| tldr pages | <https://tldr.sh/> | Короткі приклади використання команд |
| explainshell | <https://explainshell.com/> | Розбір рядка команди за частинами за матеріалами `man` |
| Операційні системи: Three Easy Pieces | <https://pages.cs.wisc.edu/~remzi/OSTEP/> | Безкоштовний підручник з теорії ОС |

## Як читати документацію

1. Починайте з `--help` і `man` своєї версії програми.
2. Шукайте розділ **EXAMPLES**: готові приклади допоможуть швидше за опис параметрів.
3. Звертайте увагу на розділ **SEE ALSO** — пов'язані команди й файли.
4. Перевіряйте дату й версію в онлайн-статтях: поради двадцятирічної давності можуть бути небезпечними або застарілими (наприклад, `ifconfig` замість `ip`).
5. Не виконуйте команди з форумів без розуміння їх дії; спершу прочитайте `man` для кожного параметра.
