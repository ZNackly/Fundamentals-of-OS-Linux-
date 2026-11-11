# Практическая работа №3
## Подсистема безопасности Linux: пользователи, права, SSH, процессы

| | |
|---|---|
| **Студент** | Сафин Даниил Рустемович |
| **Группа** | АиСД TIP 1.3, факультет прикладной информатики, ИТМО |
| **Дата** | 11.11.2026 |
| **Стенд** | Debian 13 в VirtualBox (хост ВМ `Debian23`, ядро 6.12.116+deb13-amd64), сеть NAT, проброс 127.0.0.1:2221 → 22 |
| **Учётные записи** | `vboxuser` — администратор ВМ (sudo); `drs-1` — учётная запись студента (Часть 1); `student-1` — основная учётная запись из предыдущих работ (Часть 4, вход по ключу) |

---

## Часть 1. Стенд и подключение по паролю

```bash
# 1. SSH-сервер
systemctl status ssh
ss -ltnp | grep ':22'
sudo apt install openssh-server          # только если службы нет

# 2. Учётная запись студента
sudo adduser drs-1
```

Проброс порта в VirtualBox (Настройки ВМ → Сеть → Дополнительно → Проброс портов): адрес хоста `127.0.0.1`, порт хоста `2221`, порт гостя `22`.

Правка `/etc/ssh/sshd_config` (значения проверить также в `/etc/ssh/sshd_config.d/*.conf`, там может быть другое, приоритетное):

```
PermitRootLogin yes
MaxAuthTries 2
LoginGraceTime 30
```

```bash
sudo sshd -t                                                       # проверка синтаксиса
sudo sshd -T | grep -Ei '^(permitrootlogin|maxauthtries|logingracetime)'
sudo systemctl restart ssh
systemctl status ssh
ss -ltnp | grep ':22'
```

> **[Нужен скриншот]** `systemctl status ssh` и `ss -ltnp` (два способа показать, что служба слушает порт 22).
>
> **[Нужен скриншот]** вывод `sshd -T`: `permitrootlogin yes`, `maxauthtries 2`, `logingracetime 30`.

Подключения с основного компьютера (два окна терминала):

```powershell
ssh -p 2221 root@127.0.0.1
ssh -p 2221 drs-1@127.0.0.1
```

На ВМ:

```bash
who | tee -a otchet.txt
ss -tnp | grep ssh | tee -a otchet.txt
```

> **[Нужен скриншот]** отпечаток ключа сервера при первом подключении.
>
> **[Нужен скриншот]** вывод `who` и `ss -tnp` при двух одновременных сессиях (root и drs-1).

Копирование `scp` в обе стороны (у `scp` порт задаётся заглавной `-P`, у `ssh` — строчной `-p`):

```powershell
scp -P 2221 test.txt drs-1@127.0.0.1:/home/drs-1/          # с основного компьютера на ВМ
scp -P 2221 drs-1@127.0.0.1:/etc/hostname ./hostname_vm    # с ВМ на основной компьютер
```

> **[Нужен скриншот]** выполнение обеих команд `scp`.

---

## Часть 2. Пользователи и группы

### Скрипт `script31`

Скрипт проверяет число параметров и то, что они целые числа, проверяет права (нужен root), пропускает уже существующих пользователей, задаёт пароль через `chpasswd`.

```bash
#!/bin/bash
# script31 — создание пользователей uN с паролями DerParolN
# Использование: ./script31 <количество> <начальный_номер>   (например: ./script31 5 1)

# 1. Проверка числа параметров
if [ "$#" -ne 2 ]; then
    echo "Использование: $0 <количество> <начальный_номер>" >&2
    exit 1
fi

# 2. Параметры должны быть положительными целыми числами
case "$1$2" in
    *[!0-9]*) echo "Ошибка: параметры должны быть целыми неотрицательными числами" >&2; exit 1 ;;
esac
if [ "$1" -lt 1 ]; then
    echo "Ошибка: количество пользователей должно быть >= 1" >&2
    exit 1
fi

# 3. Проверка прав (нужен root)
if [ "$(id -u)" -ne 0 ]; then
    echo "Ошибка: скрипт нужно запускать от root (или через sudo)" >&2
    exit 2
fi

count=$1
start=$2
end=$((start + count - 1))

for n in $(seq "$start" "$end"); do
    name="u$n"
    # 4. Не создаём повторно, если пользователь уже есть
    if id "$name" >/dev/null 2>&1; then
        echo "Пользователь $name уже существует — пропускаю"
        continue
    fi
    adduser --disabled-password --gecos "" "$name" >/dev/null
    echo "$name:DerParol$n" | chpasswd
    echo "Создан пользователь $name (пароль DerParol$n)"
done
```

Запуск и настройка группы:

```bash
chmod +x script31
sudo ./script31 5 1            # создаёт u1–u5
sudo ./script31 5 1            # повторный запуск: «уже существует», без ошибок
sudo groupadd project
sudo usermod -aG project u1
sudo usermod -aG project u2
```

> **[Нужен скриншот]** запуск `sudo ./script31 5 1` (создание) и повторный запуск (пропуск существующих), а также `./script31` без параметров и без прав.

### Членство в группе и вывод `id`

Проверка членства командой `id` (на скриншоте сверху видна команда `sudo usermod -aG project u2`). Группа `project` получила GID 1007, у `u1` UID 1002, у `u2` UID 1003:

![id u1, id u2](img/02_id_u1_u2.png)

Одной командой вывести UID, GID и группы обоих пользователей можно так: `id u1 u2`.

### Записи в `/etc/passwd`, `/etc/group`, `/etc/shadow`

![getent passwd, group, shadow](img/02_getent_shadow.png)

**Поля записи `/etc/passwd`** на примере `u1:x:1002:1002::/home/u1:/bin/bash`:

| № | Поле | Значение | Назначение |
|---|------|----------|------------|
| 1 | `u1` | имя | логин пользователя |
| 2 | `x` | пароль | `x` — хеш хранится в `/etc/shadow` |
| 3 | `1002` | UID | числовой идентификатор пользователя |
| 4 | `1002` | GID | числовой идентификатор основной группы |
| 5 | (пусто) | GECOS | комментарий: полное имя и т. п. |
| 6 | `/home/u1` | домашний каталог | |
| 7 | `/bin/bash` | шелл | программа, запускаемая при входе |

Запись группы `project:x:1007:u1,u2`: имя группы, `x` (пароля группы нет), GID 1007, список дополнительных членов (`u1`, `u2`).

**Сравнение записей `u1` и `u2` в `/etc/shadow`:** поля записи — имя; хеш пароля; дата последней смены пароля (дни с 01.01.1970, у обоих `20766`); минимальный срок (0); максимальный срок (99999); предупреждение за 7 дней; остальные поля пусты. Хеши обоих начинаются с `$y$j9T$` (алгоритм yescrypt с параметрами `j9T`), затем идут соль и сам хеш. У `u1` и `u2` пароли разные (`DerParol1`, `DerParol2`), поэтому разными получились и соль, и хеш; остальные поля совпадают. Даже при одинаковых паролях хеши различались бы из-за случайной соли (см. вопрос 3).

### Группа в открытой сессии и перелогин (п. 6)

Порядок: войти под `u3` (на терминале ВМ), `su - u1` и проверить `id`; затем добавить `u3` в `project` и проверить группы в уже открытой сессии `u3`; перелогиниться.

Сессия `u3`, открытая до добавления в группу: `groups` показывает только `u3`:

![groups в открытой сессии u3](img/02_u3_groups_open_session.png)

Последовательность из терминала: вход `su - u3` → `groups` (только `u3`) → `exit` → `sudo usermod -aG project u3` → новый вход `su - u3` → `groups` (теперь `u3 project`):

![перелогин u3 и появление группы project](img/02_u3_relogin.png)

**Почему в открытой сессии группа не появляется.** Список групп процесса ядро получает при входе (`login`/`su` вызывают `initgroups`) и наследует его все дочерние процессы сессии. `usermod` меняет только файл `/etc/group`; уже работающие процессы свой список групп не перечитывают. Новый вход заново читает базу групп, поэтому `project` появился.

---

## Часть 3. Права на файлы и каталоги

### 3.1. Каталог `/DATA`

Группа `project` — владелец каталога с правами `rwx`, остальные только `r-x`:

```bash
sudo mkdir /DATA
sudo chgrp project /DATA
sudo chmod 775 /DATA
```

Проверка: `u1` (в группе) создаёт файл; `u4` (вне группы) файл создать не может (`Permission denied`), но читает (`cat` отработал без ошибки):

```bash
sudo -u u1 touch /DATA/f1
sudo -u u4 touch /DATA/f4
sudo -u u4 cat /DATA/f1
```

![проверка прав на /DATA](img/03_data_rw.png)

Права на каталог (`w` и `x`) позволяют членам группы создавать и удалять любые файлы в нём. Чтобы они могли ещё и *изменять* файлы друг друга, у самих файлов должна быть запись для группы (создавать файлы с `umask 002` или выставлять `chmod g+w`).

### 3.2. Каталог `/DATA/sec1` (sticky bit)

```bash
sudo mkdir /DATA/sec1
sudo chmod 1777 /DATA/sec1
sudo -u u1 touch /DATA/sec1/a
sudo -u u3 touch /DATA/sec1/b
sudo -u u1 rm /DATA/sec1/b          # u1 пытается удалить файл u3
```

![sticky bit на /DATA/sec1](img/03_sec1_sticky.png)

Записывать в каталог может любой пользователь. При попытке удалить чужой файл `b` `rm` сначала спрашивает подтверждение (у `u1` нет записи в этот файл); из-за sticky bit удаление чужого файла всё равно не пройдёт (`Operation not permitted`), удалить можно только свои файлы.

> **[Нужен скриншот]** продолжение: ответ `y` на запрос `rm` и сообщение `Operation not permitted`, затем успешное удаление `u3` собственного файла `b`.

### 3.3. Каталог `/DATA/sec2`

`u1` — владелец (полный доступ), `u4` и `u5` — через группу `readers` только чтение, остальным доступ закрыт:

```bash
sudo groupadd readers
sudo usermod -aG readers u4
sudo usermod -aG readers u5
sudo mkdir /DATA/sec2
sudo chown u1:readers /DATA/sec2
sudo chmod 750 /DATA/sec2
sudo -u u1 touch /DATA/sec2/x
sudo -u u4 cat /DATA/sec2/x
sudo -u u5 cat /DATA/sec2/x
sudo -u u3 ls /DATA/sec2
```

![права на /DATA/sec2](img/03_sec2.png)

`u1` создал файл, `u4` и `u5` его читают, `u3` получает `Permission denied` даже на просмотр каталога.

### 3.4. Каталог `/DATA/sec3` и suid-копия `nano`

```bash
sudo mkdir /DATA/sec3
sudo cp /usr/bin/nano /DATA/sec3/nano
sudo chown u1:u1 /DATA/sec3/nano      # владелец u1, он же владелец /DATA/sec2
sudo chmod 4755 /DATA/sec3/nano       # suid; chmod после chown, т. к. chown сбрасывает suid
sudo -u u3 /DATA/sec3/nano /DATA/sec2/x
```

Идея: запущенный любым пользователем `nano` исполняется с правами владельца `u1` (suid), поэтому может изменять файлы в `/DATA/sec2`.

> **[Нужен скриншот]** `ls -l /DATA/sec3/nano` (`-rwsr-xr-x u1 u1`) и правка файла в `/DATA/sec2` под `u3` через этот `nano`.

### 3.5. suid-файлы в системе

```bash
find /usr/bin -perm -4000 -ls
```

![suid-файлы в /usr/bin](img/03_suid_find.png)

В списке есть `/usr/bin/passwd` (`-rwsr-xr-x root root`): ему нужен suid, чтобы обычный пользователь мог записать свой хеш в `/etc/shadow`. Если же владельцем копии редактора будет `root` с suid, любой пользователь получит редактор с правами root (см. вопрос 13).

### 3.6. Права созданных каталогов

```bash
ls -ld /DATA /DATA/sec1 /DATA/sec2 /DATA/sec3
```

![ls -ld по созданным каталогам](img/03_ls_ld.png)

| Каталог | Символьно | Восьмеричные | Владелец:группа | Смысл |
|---|---|---|---|---|
| `/DATA` | `drwxrwxr-x` | 775 | root:project | группа project: rwx, остальные: r-x |
| `/DATA/sec1` | `drwxrwxrwt` | 1777 | root:root | все rwx; sticky bit — удалять/переименовывать только свои файлы |
| `/DATA/sec2` | `drwxr-x---` | 750 | u1:readers | u1: rwx; readers (u4, u5): r-x; остальные: ничего |
| `/DATA/sec3` | `drwxr-xr-x` | 755 | root:root | всем чтение и вход; внутри suid-копия nano |

> **[Нужен скриншот]** `ls -l /DATA /DATA/sec1 /DATA/sec2 /DATA/sec3` (файлы внутри) и при необходимости `stat`.

---

## Часть 4. Аутентификация по ключам SSH

Ключ передаётся на основную учётную запись из предыдущих работ — `student-1`.

### 4.1. Создание пары ключей (основной компьютер, Windows PowerShell)

```powershell
ssh-keygen -t ed25519
```

Задана парольная фраза. Закрытый ключ — `C:\Users\User\.ssh\id_ed25519`, открытый — `C:\Users\User\.ssh\id_ed25519.pub`:

![ssh-keygen](img/04_ssh_keygen.png)

### 4.2. Установка открытого ключа на ВМ

Открытый ключ передан на ВМ в домашний каталог `student-1`, добавлен в `authorized_keys`, права выставлены (`~/.ssh` — 700, `authorized_keys` — 600), временный файл удалён:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
cat ~/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
rm ~/id_ed25519.pub
```

![установка ключа на ВМ](img/04_authorized_keys.png)

Права 700/600 нужны, потому что при `StrictModes` `sshd` отвергает ключ, если каталог или файл доступны для записи другим пользователям: те могли бы подменить ключ.

### 4.3. Конфигурация sshd

Изменённые параметры в `/etc/ssh/sshd_config`:

```
PermitRootLogin no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no
```

```bash
sudo sshd -t
sudo sshd -T | grep -Ei '^(permitrootlogin|pubkeyauthentication|authorizedkeysfile|passwordauthentication)'
sudo systemctl restart ssh
```

> **[Нужен скриншот]** вывод `sshd -T` с новыми значениями (`permitrootlogin no`, `pubkeyauthentication yes`, `authorizedkeysfile .ssh/authorized_keys`, `passwordauthentication no`).

### 4.4. Проверки входа

```powershell
ssh -p 2221 student-1@127.0.0.1                                 # по ключу
ssh -p 2221 root@127.0.0.1                                      # вход root
ssh -p 2221 -o PubkeyAuthentication=no student-1@127.0.0.1      # вход по паролю
```

![проверки входа по SSH](img/04_ssh_checks.png)

- Вход `student-1` по ключу работает (запрошена парольная фраза ключа, затем приглашение `student-1@Debian23`).
- Вход `root` не состоялся: сервер закрыл соединение (`Too many authentication failures`, т. к. `MaxAuthTries 2`, а `PermitRootLogin no`).
- Вход по паролю отклонён: `Permission denied (publickey)`.

### 4.5. Копирование с ВМ по ключу

```powershell
scp -P 2221 student-1@127.0.0.1:/home/student-1/script31 ./
```

![scp по ключу](img/04_scp.png)

---

## Часть 5. su, sudo, процессы и фоновый запуск

### 5.1. sudo, su и `su -`

```bash
which sudo || sudo apt install sudo
sudo usermod -aG sudo drs-1
# новый вход под drs-1:
sudo -l
sudo id                                      # uid=0(root)
su -c 'pwd; echo $HOME; env | head'          # su: окружение пользователя в основном сохраняется
su - -c 'pwd; echo $HOME; env | head'        # su -: полноценный вход, HOME=/root
```

> **[Нужен скриншот]** `sudo -l`, `sudo id` и сравнение вывода `su` и `su -` (`pwd`, `$HOME`, `env | head`).

### 5.2. Правило sudo для `u1`

Файл `/etc/sudoers.d/u1`, создан через `visudo -f /etc/sudoers.d/u1`:

```
u1 ALL=(root) /usr/bin/passwd [A-Za-z]*, !/usr/bin/passwd root, !/usr/bin/passwd *root*
```

> **[Нужен скриншот]** `sudo cat /etc/sudoers.d/u1` (содержимое должно совпадать с правилом выше; если правило другое — заменить текст).

Проверка файла утилитой `visudo -c`: на скриншоте первая проверка показала неверные права на файл (`/etc/sudoers.d/u1: bad permissions, should be mode 0440`):

![visudo -c: неверные права на файл](img/05_visudo_check.png)

Исправление: файл в `sudoers.d` должен иметь права 0440 (чтение только для root и группы root):

```bash
sudo chmod 0440 /etc/sudoers.d/u1
sudo visudo -c
ls -l /etc/sudoers.d/u1
```

> **[Нужен скриншот]** повторный `sudo visudo -c` без ошибки и `ls -l /etc/sudoers.d/u1` (`-r--r----- root root`).

### 5.3. Проверка ограничений

```bash
sudo -u u1 sudo passwd u2
sudo -u u1 sudo passwd root
sudo -u u1 sudo ls /root
```

![ограничения sudo для u1](img/05_sudo_u1.png)

- `sudo passwd u2` под `u1` работает: пароль `u2` изменён (`password updated successfully`).
- `sudo passwd root` и `sudo ls /root` отклонены: `Sorry, user u1 is not allowed to execute '…' as root`.

> **[Нужен скриншот или описание]** попытка обойти ограничение (например, `sudo passwd u2 root` или `sudo passwd -S root`) и её результат.

### 5.4. htop

```bash
sudo apt install htop
htop        # F6 — сортировка (MEM%), F5 — дерево, F4/F3 — фильтр/поиск (sshd), F10 — выход
```

Меню выбора сортировки (F6), выбран столбец `PERCENT_MEM`:

![htop, сортировка по памяти](img/05_htop.png)

> **[Нужен скриншот]** `htop` в древовидном режиме (F5) с найденным процессом `sshd` своего подключения.

### 5.5. Цепочка родитель — потомок

```bash
ps -eo pid,ppid,user,stat,cmd
pstree -sp $$
```

Вывод `pstree -sp $$`:

![pstree -sp $$](img/05_pstree.png)

Цепочка от оболочки до PID 1 (по PPID): `bash (7558)` ← `gnome-terminal- (7551)` ← `systemd (1361)`, пользовательский менеджер ← `systemd (1)`. Родитель оболочки — процесс эмулятора терминала `gnome-terminal`; при входе по SSH на его месте был бы процесс `sshd: <пользователь>@pts/N`.

> **[Нужен скриншот]** `ps -eo pid,ppid,user,stat,cmd` с процессами своей сессии (по ним строится цепочка).

### 5.6. Сигналы SIGTERM и SIGKILL

Во второй сессии запущен `sleep 600`; в первой найден его PID (`pgrep -a sleep`, `ps -o pid,ppid,stat,cmd -C sleep`) и отправлен `SIGTERM`:

```bash
kill -TERM 7848
```

![kill -TERM](img/05_kill_term.png)

Во второй сессии процесс завершился штатно (`Terminated`):

![sleep 600 завершён по SIGTERM](img/05_sleep_terminated.png)

> **[Нужен скриншот]** определение PID и PPID у `sleep 600` (`ps -o pid,ppid,stat,cmd -C sleep`) и повторный запуск с `kill -9` (во второй сессии должно появиться `Killed`).

**Различие:** `SIGTERM` (15) — вежливая просьба завершиться, процесс может перехватить сигнал, закрыть файлы и завершиться сам (оболочка пишет `Terminated`). `SIGKILL` (9) ядро исполняет принудительно, перехватить или игнорировать его нельзя (оболочка пишет `Killed`).

### 5.7. SIGSTOP и SIGCONT

```bash
sleep 600 &
kill -STOP <PID>; ps -o pid,stat,cmd -p <PID>      # STAT = T (остановлен)
kill -CONT <PID>; ps -o pid,stat,cmd -p <PID>      # STAT = S (спит)
kill <PID>
```

> **[Нужен скриншот]** `ps` до и после `SIGSTOP`/`SIGCONT`.

### 5.8. Скрипт `script32`

```bash
#!/bin/bash
# script32 — раз в 2 секунды дописывает в ~/script32.log время, свой PID и PPID
while true; do
    echo "$(date '+%F %T') PID=$$ PPID=$PPID" >> ~/script32.log
    sleep 2
done
```

Фоновые задания:

```bash
chmod +x script32
./script32 &       # [1] <PID>
jobs
echo $!
fg                 # затем Ctrl+Z → [1]+ Stopped
bg                 # продолжение в фоне
kill %1            # завершение
```

> **[Нужен скриншот]** `jobs`, `echo $!`, `fg`, Ctrl+Z, `bg`, `kill` и содержимое `~/script32.log`.

### 5.9. SIGHUP: запуск с `&` и с `nohup`

```bash
# вариант 1: ssh → ./script32 & → exit
# вариант 2: ssh → nohup ./script32 & → exit
# во второй сессии:
ps -ef | grep script32
tail -3 ~/script32.log
ps -o pid,ppid,cmd -C script32
pkill -f script32        # по окончании
```

| Запуск | Процесс жив после `exit`? | Файл пишется? | PPID |
|---|---|---|---|
| `&` | _вписать_ | _вписать_ | _вписать_ |
| `nohup … &` | _вписать_ | _вписать_ | _вписать_ |

> **[Нужен скриншот]** оба эксперимента и PPID у `nohup`-варианта. Ожидаемо: без `nohup` процесс получает `SIGHUP` и завершается; с `nohup` продолжает работать, а PPID становится PID 1 или PID `systemd --user` (на скриншоте `pstree` он имеет PID 1361).

---

## Часть 6. Скрипт `script_passport_v3.sh`

Выполнен рефакторинг по этапам «сбор → отчёт → лог».

- **Функции:** `check_privileges`, `check_tool`, `write_section`, `log_run`, `usage`, `die`, `cleanup`, а также `run_priv`, `dmi`, `last_login`, `pass_change`, `prepare_dirs`, `collect_*` для каждого этапа сбора. Повторяющихся фрагментов, оформленных функциями, больше пяти.
- **Права:** `check_privileges` определяет root, либо пользователя из группы `sudo` с рабочим `sudo` (`sudo -v`), иначе сообщает об отсутствии прав и завершается с кодом 2 (код 3 — `sudo` не работает). Команды через `sudo` вызываются только функцией `run_priv` и только когда скрипт не от root.
- **Временные файлы:** `cleanup` вызывается через `trap … EXIT`, поэтому `.tmp` удаляются при любом завершении, включая Ctrl+C (`INT`/`TERM` → `exit 130` → `EXIT`).
- **Параметры:** `-h` — справка, `-o файл` — имя отчёта (по умолчанию `PASSPORT/inventory.txt`).
- **Новые разделы:** «Пользователи», «Параметры sshd» (эффективные значения из `sshd -T` и пометка «задан явно / по умолчанию»), «Топ-5 процессов по памяти», «Подозрительные учётные записи», «Файлы с suid».
- **Лог:** строка с датой, версией формата отчёта `v3`, режимом запуска и именем отчёта.

Для данных об оборудовании и входах нужны пакеты: `sudo apt install pciutils dmidecode lastlog2` (после установки `lastlog2` — перезагрузка). Если утилиты нет, раздел содержит «нет данных».

```bash
#!/bin/bash
# script_passport_v3.sh — инвентаризация + аудит безопасности (формат отчёта v3)
# Этапы (как в ПР №1 и №2): 1) сбор сырых данных в PASSPORT/tmp/*.tmp
#                           2) формирование отчёта из временных файлов
#                           3) запись строки о запуске в лог
# Для полного отчёта нужны права root или sudo.

export PATH="$PATH:/usr/sbin:/sbin"

FORMAT_VERSION="v3"
PASSPORT_DIR="PASSPORT"
TMP_DIR="$PASSPORT_DIR/tmp"
LOG_DIR="$PASSPORT_DIR/log"
LOG_FILE="$LOG_DIR/report.log"
REPORT="$PASSPORT_DIR/inventory.txt"   # по умолчанию; меняется параметром -o
PRIV_MODE=""
SUDO=""

# ---------------------------------------------------------------- функции --

usage() {
    cat <<EOF
Использование: $0 [-h] [-o файл_отчёта]
  -h          показать эту справку
  -o файл     имя итогового отчёта (по умолчанию: $REPORT)
Скрипт нужно запускать от root или от пользователя из группы sudo.
EOF
}

die() {            # die "сообщение" [код_выхода]
    echo "Ошибка: $1" >&2
    exit "${2:-1}"
}

cleanup() {        # удаление временных файлов при любом завершении
    [ -d "$TMP_DIR" ] && find "$TMP_DIR" -name '*.tmp' -delete 2>/dev/null
}

check_privileges() {
    if [ "$(id -u)" -eq 0 ]; then
        PRIV_MODE="root"; SUDO=""
        return 0
    fi
    if id -nG | tr ' ' '\n' | grep -qx sudo && command -v sudo >/dev/null 2>&1; then
        if sudo -v 2>/dev/null; then
            PRIV_MODE="sudo"; SUDO="sudo"
            return 0
        fi
        die "sudo не работает (неверный пароль или нет прав)" 3
    fi
    die "недостаточно прав: запустите от root или от пользователя из группы sudo" 2
}

run_priv() {       # sudo используется только когда он реально нужен
    if [ -n "$SUDO" ]; then sudo "$@"; else "$@"; fi
}

check_tool() {     # check_tool утилита пакет
    command -v "$1" >/dev/null 2>&1 && return 0
    echo "нет данных: утилита '$1' не найдена (установите пакет: apt install $2)"
    return 1
}

write_section() {  # write_section "Заголовок" файл
    {
        echo "=== $1 ==="
        if [ -s "$2" ]; then cat "$2"; else echo "нет данных"; fi
        echo
    } >> "$REPORT"
}

log_run() {
    echo "Скрипт был запущен: $(date) | формат отчёта: $FORMAT_VERSION | режим: $PRIV_MODE | отчёт: $REPORT" >> "$LOG_FILE"
}

dmi() {            # dmi ключ_dmidecode
    local v
    v=$(run_priv dmidecode -s "$1" 2>/dev/null | head -1)
    echo "${v:-нет данных}"
}

last_login() {     # последний вход пользователя
    local out
    check_tool lastlog2 lastlog2 >/dev/null || { echo "нет данных (lastlog2 не установлен)"; return; }
    out=$(lastlog2 --user "$1" 2>/dev/null | tail -n +2 | tr -s ' ')
    echo "${out:-нет данных}"
}

pass_change() {    # дата последней смены пароля
    local v
    check_tool chage passwd >/dev/null || { echo "нет данных (chage не установлен)"; return; }
    v=$(run_priv env LC_ALL=C chage -l "$1" 2>/dev/null | sed -n 's/^Last password change[^:]*: *//p')
    echo "${v:-нет данных}"
}

prepare_dirs() {
    mkdir -p "$TMP_DIR" "$LOG_DIR" "$(dirname "$REPORT")" || die "не удалось создать каталоги" 4
}

# ------------------------------------------------- этап 1: сбор данных -----

collect_basic() {
    { date; hostname; } > "$TMP_DIR/host.tmp"
    { echo "Пользователь: $(whoami)"; who; } > "$TMP_DIR/user.tmp"
    { pwd; ls -la; } > "$TMP_DIR/dir.tmp"
}

collect_system() {
    { echo "Ядро: $(uname -sr)"; echo "Разрядность/архитектура: $(uname -m)"; } > "$TMP_DIR/kernel.tmp"
    grep -E '^(PRETTY_NAME|VERSION)=' /etc/os-release > "$TMP_DIR/os.tmp" 2>/dev/null
    {
        if check_tool lscpu util-linux; then
            echo "Модель: $(LC_ALL=C lscpu | sed -n 's/^Model name: *//p')"
            mhz=$(LC_ALL=C lscpu | sed -n 's/^CPU max MHz: *//p')
            [ -z "$mhz" ] && mhz=$(sed -n 's/^cpu MHz[[:space:]]*: *//p' /proc/cpuinfo | head -1)
            echo "Частота (МГц): ${mhz:-нет данных}"
        fi
    } > "$TMP_DIR/cpu.tmp"
    { nproc 2>/dev/null || grep -c '^processor' /proc/cpuinfo; } > "$TMP_DIR/cores.tmp"
    { check_tool lscpu util-linux && LC_ALL=C lscpu | grep -E '^L[0-9][a-z]* cache'; } > "$TMP_DIR/cache.tmp"
    awk '/^MemTotal/ {printf "%.0f МБ (%s кБ)\n", $2/1024, $2}' /proc/meminfo > "$TMP_DIR/mem.tmp"
    {
        if check_tool dmidecode dmidecode; then
            echo "Производитель платы: $(dmi baseboard-manufacturer)"
            echo "Модель платы: $(dmi baseboard-product-name)"
        fi
    } > "$TMP_DIR/board.tmp"
    {
        if check_tool dmidecode dmidecode; then
            echo "Версия BIOS: $(dmi bios-version)"
            echo "Дата выпуска BIOS: $(dmi bios-release-date)"
        fi
    } > "$TMP_DIR/bios.tmp"
    { check_tool lspci pciutils && lspci; } > "$TMP_DIR/pci.tmp"
}

collect_users() {
    {
        for u in $(getent passwd | awk -F: '$1=="root" || ($3>=1000 && $3<65534) {print $1}'); do
            echo "Пользователь: $u"
            echo "  UID: $(id -u "$u")"
            echo "  Группы: $(id -nG "$u")"
            echo "  Последний вход: $(last_login "$u")"
            echo "  Последняя смена пароля: $(pass_change "$u")"
        done
    } > "$TMP_DIR/users.tmp"
}

collect_sshd() {
    local eff p src
    {
        if check_tool sshd openssh-server; then
            eff=$(run_priv sshd -T 2>/dev/null)
            if [ -z "$eff" ]; then
                echo "нет данных: sshd -T не выполнился (нужны права root или ошибка конфигурации)"
            else
                echo "Эффективные значения (sshd -T):"
                echo "$eff" | grep -E '^(permitrootlogin|port|logingracetime|maxauthtries|passwordauthentication|pubkeyauthentication) ' | sed 's/^/  /'
                echo
                echo "Происхождение значений:"
                for p in PermitRootLogin Port LoginGraceTime MaxAuthTries PasswordAuthentication PubkeyAuthentication; do
                    src=$(grep -iHE "^[[:space:]]*$p[[:space:]]" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*.conf 2>/dev/null | head -1 | cut -d: -f1)
                    if [ -n "$src" ]; then
                        echo "  $p: задан явно ($src)"
                    else
                        echo "  $p: взят по умолчанию"
                    fi
                done
            fi
        fi
    } > "$TMP_DIR/sshd.tmp"
}

collect_top5() {
    { check_tool ps procps && ps -eo pid,ppid,user,%mem,comm --sort=-%mem | head -6; } > "$TMP_DIR/top5.tmp"
}

collect_suspicious() {
    local empty uid0
    {
        echo "Пользователи с пустым паролем в /etc/shadow:"
        empty=$(run_priv awk -F: '$2=="" {print $1}' /etc/shadow 2>/dev/null)
        if [ -n "$empty" ]; then echo "$empty" | sed 's/^/  /'; else echo "  не обнаружено"; fi
        echo "Пользователи с UID 0, кроме root:"
        uid0=$(getent passwd | awk -F: '$3==0 && $1!="root" {print $1}')
        if [ -n "$uid0" ]; then echo "$uid0" | sed 's/^/  /'; else echo "  не обнаружено"; fi
    } > "$TMP_DIR/suspicious.tmp"
}

collect_suid() {
    local out
    out=$(find /usr/bin /usr/sbin -type f -perm -4000 -exec ls -l {} + 2>/dev/null | sort -u)
    echo "${out:-suid-файлов не найдено}" > "$TMP_DIR/suid.tmp"
}

# ------------------------------------------- этап 2: формирование отчёта ----

build_report() {
    {
        echo "ОТЧЁТ ОБ ИНВЕНТАРИЗАЦИИ (формат $FORMAT_VERSION)"
        echo "Режим запуска: $PRIV_MODE"
        echo "Внимание: чтение /etc/shadow, chage -l, sshd -T и dmidecode требуют прав root или sudo."
        echo
    } > "$REPORT"
    write_section "Информация о хосте"        "$TMP_DIR/host.tmp"
    write_section "Пользователь, запустивший скрипт" "$TMP_DIR/user.tmp"
    write_section "Текущий каталог и файлы"   "$TMP_DIR/dir.tmp"
    write_section "Ядро и разрядность"        "$TMP_DIR/kernel.tmp"
    write_section "Операционная система"      "$TMP_DIR/os.tmp"
    write_section "Процессор"                 "$TMP_DIR/cpu.tmp"
    write_section "Логические ядра"           "$TMP_DIR/cores.tmp"
    write_section "Кэш процессора"            "$TMP_DIR/cache.tmp"
    write_section "Физическая память"         "$TMP_DIR/mem.tmp"
    write_section "Материнская плата"         "$TMP_DIR/board.tmp"
    write_section "BIOS"                      "$TMP_DIR/bios.tmp"
    write_section "Устройства PCI"            "$TMP_DIR/pci.tmp"
    write_section "Пользователи"              "$TMP_DIR/users.tmp"
    write_section "Параметры sshd"            "$TMP_DIR/sshd.tmp"
    write_section "Топ-5 процессов по памяти" "$TMP_DIR/top5.tmp"
    write_section "Подозрительные учётные записи" "$TMP_DIR/suspicious.tmp"
    write_section "Файлы с suid (/usr/bin, /usr/sbin)" "$TMP_DIR/suid.tmp"
}

# ------------------------------------------------------------------ main ---

trap cleanup EXIT
trap 'echo "Прервано пользователем" >&2; exit 130' INT TERM

while getopts ":ho:" opt; do
    case "$opt" in
        h) usage; exit 0 ;;
        o) REPORT="$OPTARG" ;;
        :) echo "Ошибка: параметру -$OPTARG нужно значение" >&2; usage >&2; exit 1 ;;
        *) usage >&2; exit 1 ;;
    esac
done

check_privileges
prepare_dirs

# этап 1
collect_basic
collect_system
collect_users
collect_sshd
collect_top5
collect_suspicious
collect_suid
# этап 2
build_report
# этап 3
log_run

echo "Готово. Отчёт: $REPORT, лог: $LOG_FILE"
```

Три проверки:

```bash
sudo ./script_passport_v3.sh            # под root: полный отчёт
./script_passport_v3.sh                 # под drs-1 из группы sudo: запрос пароля sudo, отчёт полный
su - u3 -c ./script_passport_v3.sh      # без прав: отказ, код возврата 2 (echo $?)
./script_passport_v3.sh -h              # справка
./script_passport_v3.sh -o /tmp/my.txt  # свой файл отчёта
cat PASSPORT/log/report.log             # после двух запусков — две строки
ls PASSPORT/tmp                         # пусто: .tmp удалены
```

> **[Нужен скриншот]** три запуска (root, пользователь с sudo, пользователь без прав) и три разных результата.
>
> **[Нужен файл/скриншот]** содержимое сгенерированного `PASSPORT/inventory.txt` и `PASSPORT/log/report.log` (после двух запусков — две строки с `формат отчёта: v3`).

**Сравнение `lastlog2` и `lslogins`.** `lastlog2` хранит историю входов в собственной базе (`/var/lib/lastlog/lastlog2.db`), её пишет PAM-модуль при входе; для новых учётных записей показывает, что входов не было. `lslogins` — сводка по учётным записям из `/etc/passwd`, `shadow` и журналов входов (`wtmp`/`btmp`): UID, группы, статус пароля, последний вход. Источником данных о последнем входе на Debian 13 выбран `lastlog2`, `lslogins` удобен для общей сводки по пользователям.

---

## Ответы на вопросы

**1. Зачем при создании SSH-ключа предлагается задать парольную фразу?**
Она шифрует закрытый ключ на диске. Если файл украдут (копия диска, вредоносное ПО, чужой доступ к компьютеру), без фразы им нельзя воспользоваться. Это защита от кражи и использования закрытого ключа: к «чем владеешь» (файл ключа) добавляется «что знаешь» (фраза).

**2. Зачем при первом подключении по SSH показывается отпечаток ключа сервера?**
Чтобы убедиться, что вы подключаетесь к настоящему серверу, а не к подставному (MITM, подмена DNS или IP). Принятый ключ сохраняется в `~/.ssh/known_hosts`. Если отпечаток изменится, клиент выдаст предупреждение `REMOTE HOST IDENTIFICATION HAS CHANGED` и откажется подключаться: возможна подмена сервера или ключ сервера был сменён (например, переустановка ОС).

**3. Если у двух пользователей одинаковые пароли, можно ли понять это по `/etc/shadow`?**
Нет. В хеш входит случайная соль, поэтому у одинаковых паролей разные записи (`$y$j9T$<соль>$<хеш>`). Это защищает от радужных таблиц и от обнаружения совпадающих паролей.

**4. Таблица атрибутов**

| Атрибут | Сокращённое название | Для файла | Для каталога |
|---|---|---|---|
| r | read | читать содержимое | получать список имён (`ls`) |
| w | write | изменять содержимое | создавать, переименовывать, удалять записи (только вместе с x) |
| x | execute | запускать как программу | входить (`cd`), обращаться к файлам по имени |
| suid (4000) | set user ID | запуск с правами владельца файла | обычно не действует |
| sgid (2000) | set group ID | запуск с правами группы-владельца | новые файлы наследуют группу каталога |
| sticky (1000) | sticky bit | обычно не действует | удалять и переименовывать файлы могут только владелец файла, владелец каталога и root |

**5. Почему для удаления файла нужны права на каталог? Что даёт sticky bit?**
Файл — это запись (имя → inode) в каталоге. Удаление меняет каталог, а не содержимое файла, поэтому нужны `w` и `x` на каталог. Sticky bit на каталоге ограничивает удаление и переименование: только владелец файла, владелец каталога или root (пример: `/tmp`, права 1777; на скриншоте — `/DATA/sec1`).

**6. Команда `setfacl`; как решить задание 3 с помощью ACL?**

```bash
setfacl -m u:username:rwx файл
```

Задание 3 с ACL:

```bash
mkdir /DATA/sec2
chown root:root /DATA/sec2
chmod 700 /DATA/sec2
setfacl -m u:u1:rwx,u:u4:rx,u:u5:rx /DATA/sec2
setfacl -d -m u:u1:rwx,u:u4:rx,u:u5:rx /DATA/sec2     # default ACL для новых файлов
getfacl /DATA/sec2
```

Остальным права не даны (other = `---`), дополнительная группа `readers` не нужна.

**7. Чем различаются `su`, `su -` и `sudo`? Почему для повседневной работы рекомендуют `sudo`?**
`su` переключает на пользователя по его паролю, окружение в основном сохраняется. `su -` запускает полноценный login-шелл целевого пользователя (его `HOME`, `PATH`, переменные). `sudo` выполняет одну команду от имени другого пользователя, спрашивает пароль вызывающего и работает по правилам sudoers. Для повседневной работы `sudo` лучше: пароль root не раздаётся, права выдаются точечно (принцип наименьших привилегий), каждое действие журналируется, права отзываются удалением из группы без смены общего пароля.

**8. Удалось ли обойти ограничение sudo из Части 5? Чем опасны wildcard в sudoers?**
Прямые варианты закрыты (скриншот): `sudo passwd root` и `sudo ls /root` отклоняются, `sudo passwd u2` работает. Результат моей попытки обхода (несколько аргументов, опции) описан в разделе 5.3. Подстановка `*` в sudoers соответствует любой строке, включая пробелы, дополнительные аргументы и опции, поэтому правило вида `passwd *` разрешает комбинации, которых автор не предполагал (например, несколько имён в одной команде). Исключения со знаком `!` зависят от порядка правил (действует последнее совпадение) и не покрывают всех вариантов. Надёжнее — явные команды без wildcard или обёртка-скрипт с проверкой аргументов.

**9. Чем отличается SIGTERM от SIGKILL? Какой можно перехватить?**
`SIGTERM` (15) — просьба завершиться: процесс может перехватить сигнал, закрыть файлы, освободить ресурсы или проигнорировать. `SIGKILL` (9) ядро исполняет немедленно: перехватить, заблокировать или игнорировать его нельзя. Перехватить можно только `SIGTERM`.

**10. Что происходит с процессом из `&` при закрытии SSH-сессии? Почему `nohup` меняет поведение, а PPID — другой?**
Процесс из `&` остаётся в сессии оболочки; при закрытии сессии оболочка и терминал посылают ему `SIGHUP`, и по умолчанию он завершается. `nohup` запускает программу с игнорированием `SIGHUP` (вывод перенаправляется в `nohup.out`), поэтому она переживает закрытие сессии. PPID у процесса, запущенного в оболочке, — PID этой `bash`. После закрытия сессии `bash` завершается, осиротевший процесс «усыновляется» системным процессом — PID 1 или `systemd --user`; поэтому PPID у `nohup`-варианта становится другим.

**11. Что такое зомби-процесс? Можно ли его убить `kill -9`?**
Процесс, который завершился, но родитель ещё не прочитал его код возврата (`wait()`); в таблице процессов остаётся запись со статусом `Z`. `kill -9` не помогает: процесс уже мёртв, сигналу некому доставляться, ресурсов он не занимает, кроме записи в таблице. Зомби исчезает, когда родитель вызовет `wait` либо завершится (тогда зомби усыновляет init и подбирает).

**12. Чем отличается вывод `sshd -T` от строк `sshd_config`, найденных через `grep`?**
`sshd -T` показывает итоговую конфигурацию, которую реально использует сервер: с учётом значений по умолчанию, `Include` и файлов из `sshd_config.d`, правила «действует первое найденное значение» и параметров командной строки. `grep` показывает только текст файла, включая закомментированные строки. Примеры: в `sshd_config` строка `#PermitRootLogin prohibit-password` закомментирована, а `sshd -T` всё равно выводит `permitrootlogin without-password` (значение по умолчанию); параметр `Port` может отсутствовать в `grep`, а `sshd -T` покажет `port 22`; если во вложенном `sshd_config.d/*.conf` задано другое значение раньше, `sshd -T` покажет именно его.

**13. Какой риск, если владелец исполняемого файла — root и установлен suid?**
Файл запускается с правами root независимо от того, кто его вызвал. Любая уязвимость в программе или возможность запустить из неё команды/шелл (редакторы, интерпретаторы) приводит к получению root: чтение и запись `/etc/shadow`, `/etc/sudoers`, установка бэкдоров. Поэтому suid-root-файлов должно быть минимум, они должны быть проверены и находиться под аудитом (раздел «Файлы с suid» в отчёте инвентаризации).
