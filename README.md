<img width="633" height="139" alt="image" src="https://github.com/user-attachments/assets/2afd7141-a169-49c6-be87-f90be78807ac" /># Практическая работа №3
## Подсистема безопасности Linux: пользователи, права, SSH, процессы

**Студент:** Сафин Даниил Рустемович
**Университет:** ИТМО, факультет прикладной информатики
**Дата:** 09.11.2026


---

## Часть 1. Стенд и подключение по паролю

```bash
# 1. SSH-сервер
systemctl status ssh
ss -ltnp | grep ':22'
apt install openssh-server          

# 2. Учётная запись студента
adduser sdr-1

# 3. Проброс порта
#    Настройки ВМ → Сеть → Дополнительно → Проброс портов:
#    хост 127.0.0.1:2221 → гость :22)
```


```
PermitRootLogin yes
MaxAuthTries 2
LoginGraceTime 30
```

```bash
sshd -t                                   
sshd -T | grep -Ei '^(permitrootlogin|maxauthtries|logingracetime)'
systemctl restart ssh
systemctl status ssh
ss -ltnp | grep ':22'
```

**Подключения с основного компьютера**:
```bash
ssh -p 2221 root@127.0.0.1
ssh -p 2221 sdr-1@127.0.0.1
```
Отпечаток ключа сервера при первом подключении:
<img width="633" height="139" alt="image" src="https://github.com/user-attachments/assets/df350447-a74f-474a-a380-c46fc76cbe52" />


**На ВМ:**
```bash
who | tee -a otchet.txt
ss -tnp | grep ssh | tee -a otchet.txt
```
<img width="449" height="96" alt="image" src="https://github.com/user-attachments/assets/9fa3e82f-73b8-4437-886c-b248dc598748" />

**scp в обе стороны**:
```bash
scp -P 2221 test.txt sdr-1@127.0.0.1:/home/sdr-1/       
scp -P 2221 sdr-1@127.0.0.1:/etc/hostname ./hostname_vm  
```

---

## Часть 2. Пользователи и группы

### Скрипт `script31`
```bash
#!/bin/bash


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

```bash
chmod +x script31             
./script31 5 1                
./script31 5 1                
groupadd project
```
<img width="371" height="112" alt="image" src="https://github.com/user-attachments/assets/4776cce3-a978-47a7-875f-8174adcd4152" />


<img width="498" height="137" alt="image" src="https://github.com/user-attachments/assets/3cada9ef-ef2d-4c67-9387-fdc58705c145" />

| № | Поле | Назначение |
|---|------|------------|
| 1 | `u1` | имя пользователя (логин) |
| 2 | `x` | пароль; `x` — хеш хранится в `/etc/shadow` |
| 3 | `1001` | UID |
| 4 | `1001` | GID основной группы |
| 5 | (пусто) | GECOS — комментарий (полное имя и т. п.) |
| 6 | `/home/u1` | домашний каталог |
| 7 | `/bin/bash` | шелл при входе |

**Сравнение хешей u1 и u2. Пароли разные (`DerParol1`/`DerParol2`), поэтому и хеши разные; даже при одинаковых паролях они различались бы из-за случайной соли (см. вопрос 3).
<img width="873" height="57" alt="image" src="https://github.com/user-attachments/assets/c4e67fb3-2b23-45bf-9f39-e73aeb57fd92" />

**Групповая сессия (п. 6):**
```bash
# на физическом терминале:
id                       
# в другой сессии :
su - u1 -c id            # project в списке групп
usermod -aG project u3
# в уже открытой сессии u3:
id                       # project по-прежнему НЕТ
# выйти и войти как u3 заново:
id                       # project появилась
```
**Почему не обновляется:** список групп процесса фиксируется ядром при входе и наследуется всеми потомками. `usermod` меняет только `/etc/group`; уже работающие процессы свой список не перечитывают, новый вход — перечитывает.

```
<img width="371" height="112" alt="image" src="https://github.com/user-attachments/assets/d9834abf-cf71-4834-8ef5-2892aa6f5b2e" />

```

---

## Часть 3. Права на файлы и каталоги

```bash
# --- 1. /DATA: группа project пишет, остальные только читают ---
mkdir /DATA
chgrp project /DATA
chmod 2775 /DATA            
sudo -u u1 bash -c 'umask 002; echo test > /DATA/f_u1.txt'
sudo -u u2 bash -c 'echo more >> /DATA/f_u1.txt'   
sudo -u u2 rm /DATA/f_u1.txt                         
sudo -u u4 cat /DATA/f_u1.txt                       
sudo -u u4 touch /DATA/f_u4.txt                    

# --- 2. /DATA/sec1: все пишут, удалять только своё (sticky bit) ---
mkdir /DATA/sec1
chmod 1777 /DATA/sec1     
sudo -u u1 touch /DATA/sec1/a_u1
sudo -u u3 touch /DATA/sec1/a_u3
sudo -u u3 rm /DATA/sec1/a_u1     
sudo -u u3 rm /DATA/sec1/a_u3

<img width="434" height="39" alt="image" src="https://github.com/user-attachments/assets/c4095ab9-bbd9-4437-969f-fcb840fd1714" />

<img width="639" height="38" alt="image" src="https://github.com/user-attachments/assets/521f7f7e-3567-48a2-a4fa-e5c99701ecd7" />


# --- 3. /DATA/sec2: u1 — полный доступ, u4/u5 — чтение, остальные — ничего ---
groupadd sec2ro
usermod -aG sec2ro u4
usermod -aG sec2ro u5
mkdir /DATA/sec2
chown u1:sec2ro /DATA/sec2
chmod 2750 /DATA/sec2       
sudo -u u1 bash -c 'umask 027; echo secret > /DATA/sec2/doc.txt'   # файл 640 u1:sec2ro
sudo -u u4 cat /DATA/sec2/doc.txt     
sudo -u u4 touch /DATA/sec2/x        
sudo -u u3 ls /DATA/sec2              

# --- 4. /DATA/sec3: suid-копия nano ---
mkdir /DATA/sec3
cp /usr/bin/nano /DATA/sec3/nano
chown u1:u1 /DATA/sec3/nano           # владелец u1 — он же владелец /DATA/sec2
chmod 4755 /DATA/sec3/nano            # rwsr-xr-x (chown сбрасывает suid, поэтому chmod — после chown)
sudo -u u3 /DATA/sec3/nano /DATA/sec2/doc.txt   # изменить и сохранить (Ctrl+O, Ctrl+X)
cat /DATA/sec2/doc.txt                           # изменения видны

# --- 5. suid в системе ---
find /usr/bin -perm -4000 -type f -ls
ls -l /usr/bin/passwd                 # -rwsr-xr-x root root

# --- 6. Права созданного ---
ls -ld /DATA /DATA/sec1 /DATA/sec2 /DATA/sec3
ls -l /DATA /DATA/sec1 /DATA/sec2 /DATA/sec3
stat -c '%A %a %U:%G %n' /DATA /DATA/sec1 /DATA/sec2 /DATA/sec3 /DATA/sec3/nano
```

**Пояснение прав:**

| Каталог | Символьно | Восьмеричные | Смысл |
|---|---|---|---|
| `/DATA` | `drwxrwsr-x root:project` | 2775 | project: rwx; остальные: r-x; sgid — наследование группы |
| `/DATA/sec1` | `drwxrwxrwt` | 1777 | все rwx; sticky — удалять/переименовывать только своё |
| `/DATA/sec2` | `drwxr-s--- u1:sec2ro` | 2750 | u1: rwx; sec2ro (u4, u5): r-x; остальные: ничего |
| `/DATA/sec3` | `drwxr-xr-x` | 755 | всем чтение/вход |
| `/DATA/sec3/nano` | `-rwsr-xr-x u1:u1` | 4755 | запускается с правами u1 (suid) |

**Вывод проверок и `ls -l`/`stat`:**
```
<img width="639" height="38" alt="image" src="https://github.com/user-attachments/assets/29f595f6-8ef7-4943-b04c-690f90a2b286" />

<img width="583" height="72" alt="image" src="https://github.com/user-attachments/assets/11b24844-df15-4de4-87fa-99398c7f4c62" />
<img width="574" height="82" alt="image" src="https://github.com/user-attachments/assets/4b885705-9889-47f6-8864-453316ac4501" />

```

**Риск suid-nano с владельцем root:** любой пользователь получает редактор с правами root: можно править `/etc/shadow`, `/etc/sudoers`, добавить себе UID 0, а из nano запускать команды (`^R^X`) — это полный захват системы (см. вопрос 13).

---

## Часть 4. Аутентификация по ключам SSH

```bash
# На основном компьютере
ssh-keygen -t ed25519                  
ssh-copy-id -p 2221 -i ~/.ssh/id_ed25519.pub sdr-1@127.0.0.1   # Linux/macOS
# Windows PowerShell (вручную):
#   type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2221 sdr-1@127.0.0.1 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# На ВМ (под sdr-1): права
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
ls -ld ~/.ssh ~/.ssh/authorized_keys

# Проверка входа по ключу (первая сессия не закрывается!)
ssh -p 2221 sdr-1@127.0.0.1
```
<img width="513" height="299" alt="image" src="https://github.com/user-attachments/assets/adf5fa1f-141b-488a-b648-55b31d0e59ca" />

**Изменения в `/etc/ssh/sshd_config`:**
```
PermitRootLogin no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no
```
```bash
sshd -t
sshd -T | grep -Ei '^(permitrootlogin|pubkeyauthentication|authorizedkeysfile|passwordauthentication)'
systemctl restart ssh
```

**Проверки:**
```bash
ssh -p 2221 sdr-1@127.0.0.1                                  # по ключу — OK
ssh -p 2221 root@127.0.0.1                                   # Permission denied
ssh -p 2221 -o PubkeyAuthentication=no sdr-1@127.0.0.1       # Permission denied (publickey)
scp -P 2221 sdr-1@127.0.0.1:~/script31 .                     # копирование по ключу
```
Права: `~/.ssh` — 700, `authorized_keys` — 600: иначе `sshd` (StrictModes) отвергнет ключ, так как другие пользователи могли бы подменить файл.

```
ВСТАВИТЬ ВЫВОД проверок и sshd -T
```

---

## Часть 5. su, sudo, процессы и фоновый запуск

```bash
which sudo || apt install sudo
usermod -aG sudo sdr-1
# новый вход под sdr-1:
sudo -l
sudo id                     # uid=0(root)
su -c 'pwd; echo $HOME; env | head'      # su: окружение пользователя сохраняется (HOME прежний)
su - -c 'pwd; echo $HOME; env | head'    # su -: HOME=/root, окружение root
```

### sudoers для u1 (`visudo -f /etc/sudoers.d/u1`)
```
u1 ALL=(root) /usr/bin/passwd [A-Za-z]*, !/usr/bin/passwd root, !/usr/bin/passwd *root*
```
```bash
chmod 0440 /etc/sudoers.d/u1
visudo -c
ls -l /etc/sudoers.d/u1
```
Первый шаблон разрешает `passwd` только с аргументом, начинающимся с буквы (это отсекает опции `-…`), исключения со знаком `!` запрещают `root`.

**Проверки (под u1):**
```bash
sudo passwd u2          # OK
sudo passwd root        # запрещено
sudo ls /root           # запрещено
sudo passwd u2 root     # попытка обхода: несколько аргументов
sudo passwd -S root     # попытка обхода через опцию
```
```
ВСТАВИТЬ ВЫВОД и описание результата обхода
```

### Процессы
```bash
apt install htop
htop      # F6 → MEM%; F5 → дерево; F4/F3 → найти sshd вашего соединения; F10 → выход

ps -eo pid,ppid,user,stat,cmd | grep -E 'sshd|bash'
echo $$                              # PID вашей bash
pstree -sp $$                        # цепочка до systemd(1)
```
Цепочка (пример формы): `bash(PID) ← sshd: user@pts/0 ← sshd: user [priv] ← sshd (listener) ← systemd(1)`. Родитель оболочки — процесс `sshd: <user>@pts/N`.
```
ВСТАВИТЬ ps-вывод и вывод pstree, цепочку PID–PPID
```

**Сигналы (вторая сессия: `sleep 600`; первая сессия):**
```bash
pgrep -a sleep ; ps -o pid,ppid,stat,cmd -C sleep
kill <PID>               # SIGTERM — процесс завершится штатно
sleep 600 &              # заново, затем:
kill -9 <PID>            # SIGKILL — убит ядром принудительно
sleep 600 &
kill -STOP <PID> ; ps -o pid,stat,cmd -p <PID>    # STAT = T
kill -CONT <PID> ; ps -o pid,stat,cmd -p <PID>    # STAT = S
```

### Скрипт `script32`
```bash
#!/bin/bash
# script32 — раз в 2 секунды дописывает в ~/script32.log время, свой PID и PPID
while true; do
    echo "$(date '+%F %T') PID=$$ PPID=$PPID" >> ~/script32.log
    sleep 2
done
```

**Фоновые задания:**
```bash
chmod +x script32
./script32 &       # [1] PID
jobs
echo $!
fg                 # Ctrl+Z → [1]+ Stopped
bg                 # продолжает в фоне
kill %1            # завершить
```

**SIGHUP-эксперимент:**
```bash
# вариант 1 (без nohup): ssh → ./script32 & → exit
# вариант 2: ssh → nohup ./script32 & → exit
# во второй сессии:
ps -ef | grep script32 ; tail -3 ~/script32.log ; ps -o pid,ppid,cmd -C script32
pkill -f script32
```
| Запуск | Процесс жив после `exit`? | Файл пишется? | PPID |
|---|---|---|---|
| `&` | ВПИСАТЬ | ВПИСАТЬ | ВПИСАТЬ |
| `nohup … &` | ВПИСАТЬ | ВПИСАТЬ | ВПИСАТЬ |

> Ожидаемо: без nohup процесс получает SIGHUP и завершается; с nohup — продолжает работать, а PPID становится 1 (или PID `systemd --user`). Точный результат без nohup может зависеть от настройки `huponexit` в bash — фиксируйте то, что увидели.

---

## Часть 6. Скрипт `script_passport_v3.sh`

Выполнен рефакторинг по этапам «сбор → отчёт → лог»: функции `check_privileges`, `check_tool`, `write_section`, `log_run`, `usage`, `die`, `cleanup`, а также `run_priv`, `dmi`, `last_login`, `pass_change`, `prepare_dirs` (>5 повторяющихся фрагментов вынесено в функции). `cleanup` вызывается через `trap … EXIT`, поэтому временные файлы удаляются и при завершении по ошибке, и при Ctrl+C (`INT`/`TERM` → `exit 130` → `EXIT`). Отчёт задаётся ключом `-o`, по умолчанию `PASSPORT/inventory.txt`. Права: root — работает напрямую; пользователь из группы sudo — команды, требующие root (`dmidecode`, `chage`, `sshd -T`, `/etc/shadow`), идут через `sudo`; остальные — отказ с кодом 2 (код 3 — sudo не работает).

> Примечание: исходный текст `script_passport_v2.sh` в присланных файлах отсутствовал, поэтому v3 собран по описанию ПР №1 и №2 (те же три этапа, те же имена каталогов `PASSPORT/{tmp,log}`, лог `report.log`). Если у вас есть собственный v2 — добавьте его разделы «Информация о хосте» и т. п. в `collect_basic`.

Для данных об оборудовании и входах установите пакеты: `apt install pciutils dmidecode lastlog2` (после установки lastlog2 — перезагрузка).

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

**Три сценария проверки:**
```bash
sudo ./script_passport_v3.sh            # под root: полный отчёт
./script_passport_v3.sh                 # под sdr-1 (в группе sudo): запросит пароль sudo, отчёт полный
su - u3 -c ./script_passport_v3.sh      # без прав: «недостаточно прав…», код 2 (см. echo $?)
./script_passport_v3.sh -h              # справка
./script_passport_v3.sh -o /tmp/my.txt  # свой файл отчёта
cat PASSPORT/log/report.log             # после двух запусков — две строки
ls PASSPORT/tmp                         # пусто: .tmp удалены
```

**Сгенерированный отчёт `PASSPORT/inventory.txt` и лог:**
```
ВСТАВИТЬ СОДЕРЖИМОЕ inventory.txt И report.log (после двух запусков — две строки с «формат отчёта: v3»)
```

**Сравнение `lastlog2` и `lslogins`:** `lastlog2` хранит историю входов в БД (`/var/lib/lastlog/lastlog2.db`), пишется PAM-модулем при входе и покажет «Never logged in» для новых учётных записей; `lslogins` — сводка по учётным записям из `/etc/passwd`, `utmp`/`wtmp`, `shadow`, включает последний вход, статус пароля, группы, но не заменяет `lastlog2` как источник данных о входах и «видит» только то, что есть в wtmp/lastlog.

---

## Ответы на вопросы

**1. Зачем парольная фраза для SSH-ключа?**
Она шифрует закрытый ключ на диске. Если файл украдут (копия диска, вредоносное ПО, чужой доступ к ноутбуку), без фразы им нельзя воспользоваться: защита от кражи и использования закрытого ключа. Это второй фактор («что имеешь» + «что знаешь»).

**2. Зачем отпечаток ключа сервера?**
Чтобы убедиться, что вы подключаетесь к настоящему серверу, а не к подставному (MITM, подмена DNS/IP). Принятый ключ сохраняется в `~/.ssh/known_hosts`. Если отпечаток изменится, клиент выдаст предупреждение `REMOTE HOST IDENTIFICATION HAS CHANGED` и откажется подключаться: возможна подмена сервера, либо ключ сервера был сменён (переустановка ОС).

**3. Одинаковые пароли в `/etc/shadow`?**
Нет, понять нельзя. В хеш входит случайная соль, поэтому у одинаковых паролей разные записи (`$y$j9T$<соль>$<хеш>`). Это защищает от радужных таблиц и от обнаружения совпадающих паролей.

**4. Таблица атрибутов**

| Атрибут | Сокращённое название | Для файла | Для каталога |
|---|---|---|---|
| r | read | читать содержимое | получать список имён (`ls`) |
| w | write | изменять содержимое | создавать, переименовывать, удалять записи (работает только с x) |
| x | execute | запускать как программу | входить (`cd`), обращаться к файлам по имени |
| suid (4000) | set user ID | запуск с правами владельца файла | обычно не действует |
| sgid (2000) | set group ID | запуск с правами группы-владельца | новые файлы наследуют группу каталога |
| sticky (1000) | sticky bit | обычно не действует | удалять/переименовывать файлы только владелец файла, владелец каталога и root |

**5. Почему для удаления нужны права на каталог? Что даёт sticky bit?**
Файл — это запись (имя → inode) в каталоге; удаление изменяет каталог, а не содержимое файла, поэтому нужны `w` и `x` на каталог. Sticky bit на каталоге ограничивает удаление и переименование: только владелец файла, владелец каталога или root (пример: `/tmp`, 1777).

**6. ACL**
```bash
setfacl -m u:username:rwx файл
```
Задание 3 с ACL:
```bash
mkdir /DATA/sec2
chown root:root /DATA/sec2
chmod 700 /DATA/sec2
setfacl -m u:u1:rwx,u:u4:rx,u:u5:rx /DATA/sec2
setfacl -d -m u:u1:rwx,u:u4:rx,u:u5:rx /DATA/sec2   # для новых файлов (default ACL)
getfacl /DATA/sec2
```
Остальным права не даны (other = `---`), доп. группы не нужны.

**7. su, su -, sudo; почему sudo?**
`su` — переключение на пользователя с его паролем, окружение в основном сохраняется. `su -` — полноценный login-шелл целевого пользователя (его HOME, PATH, переменные). `sudo` — выполнение одной команды от имени другого пользователя с паролем вызывающего, по правилам sudoers. Для повседневной работы sudo лучше: пароль root не раздаётся, права выдаются точечно (принцип наименьших привилегий), каждое действие журналируется, право отзывается удалением из группы без смены общего пароля.

**8. Удалось ли обойти ограничение? Чем опасны wildcard?**
*(Впишите, что получилось у вас: типично `sudo passwd u2 root` блокируется шаблоном `!…*root*`, а `passwd -S root` не проходит, так как аргумент должен начинаться с буквы.)* Подстановка `*` в sudoers соответствует любой строке, включая пробелы и дополнительные аргументы/опции, поэтому правило «passwd *» разрешает `passwd -x … root` и другие неожиданные комбинации; такие правила легко обходятся, а правила с исключениями зависят от порядка (последнее совпадение побеждает) и не покрывают всех вариантов. Надёжнее — явные команды без wildcard или обёртка-скрипт с проверкой аргументов.

**9. SIGTERM и SIGKILL**
SIGTERM (15) — просьба завершиться; процесс может перехватить, закрыть файлы, освободить ресурсы или проигнорировать. SIGKILL (9) — немедленное уничтожение ядром; перехватить, заблокировать или игнорировать нельзя (как и SIGSTOP).

**10. `&`, закрытие SSH-сессии, nohup, PPID**
Процесс из `&` остаётся в той же сессии/группе терминала; при закрытии терминала/сессии ему приходит SIGHUP, и по умолчанию он завершается. `nohup` запускает программу с игнорированием SIGHUP (и перенаправляет вывод в `nohup.out`), поэтому она переживает закрытие сессии. PPID у процесса, запущенного в оболочке, — PID этой bash. После закрытия сессии bash завершается, а осиротевший процесс «усыновляется» системным процессом — PID 1 (`systemd`) или `systemd --user`; поэтому PPID становится другим.

**11. Зомби-процесс**
Процесс, который завершился, но родитель ещё не прочитал его код возврата (`wait()`); в таблице процессов остаётся запись со статусом `Z`. `kill -9` не помогает: процесс уже мёртв, сигналу некому доставляться, ресурсов он не занимает, кроме записи в таблице. Устраняется завершением/перезапуском родителя (тогда зомби усыновляет init и подбирает) или исправлением родителя, чтобы он вызывал `wait`.

**12. `sshd -T` и `grep` по `sshd_config`**
`sshd -T` показывает итоговую конфигурацию, которую реально использует сервер, с учётом значений по умолчанию, `Include` и файлов из `sshd_config.d`, порядка «побеждает первое значение» и ключей командной строки; `grep` показывает только текст файла (включая закомментированное). Пример: `PermitRootLogin` — в `sshd_config` закомментировано `#PermitRootLogin prohibit-password`, а `sshd -T` выводит `permitrootlogin without-password`/`prohibit-password`; либо параметр `Port` отсутствует в grep, а `sshd -T` показывает `port 22`; либо в `sshd_config.d/*.conf` задано другое значение, которое стоит раньше.

**13. Риск root + suid**
Файл запускается с правами root независимо от того, кто его вызвал. Любая уязвимость в программе, возможность запуска команд/шелла из неё (редакторы, интерпретаторы) или подмена данных ведёт к получению root: чтение/запись `/etc/shadow`, установка бэкдоров. Поэтому suid-root-файлы должны быть минимальными, проверенными и находиться под аудитом (раздел «Файлы с suid» в отчёте).
