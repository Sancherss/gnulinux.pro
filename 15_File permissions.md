# Права доступа, пользователи, группы, ACL, SGID, Sticky bit

## Вопрос: Расшифровка прав файлов

### Пример вывода `ls -l /etc`

```bash
-rw-r--r--.  1 root root   16 апр 12 11:49 adjtime
lrwxrwxrwx.  1 root root   12 авг  1  2025 yum.conf -> dnf/dnf.conf
drwxr-xr-x.  6 root root   81 апр 12 11:42 wireplumber
```

### Расшифровка

| Файл | Права | Численно | Расшифровка |
|------|-------|----------|-------------|
| `adjtime` | `-rw-r--r--` | `644` | владелец: чтение/запись, группа/остальные: только чтение |
| `yum.conf` | `lrwxrwxrwx` | `777` | символическая ссылка (реальные права у целевого файла) |
| `wireplumber` | `drwxr-xr-x` | `755` | директория: владелец — всё, группа/остальные — чтение/выполнение |

### Таблица численных прав

| Число | Буквы | Значение |
|-------|-------|----------|
| 7 | `rwx` | чтение + запись + выполнение |
| 6 | `rw-` | чтение + запись |
| 5 | `r-x` | чтение + выполнение |
| 4 | `r--` | только чтение |
| 0 | `---` | нет прав |

---

## Задание 1: Создание пользователей и групп (IT-отдел)

### Выполненные команды

```bash
# Группы
sudo groupadd company
sudo groupadd it
sudo groupadd sales

# Директории
sudo mkdir -p /company/it /company/sales /company/shared

# Пользователи (it*)
sudo useradd -g company -G it -b /company/it -m -s /bin/bash ituser1
sudo useradd -g company -G it -b /company/it -m -s /bin/bash ituser2
sudo useradd -g company -G it -b /company/it -m -s /bin/bash itadmin

# Пользователи (s*)
sudo useradd -g company -G sales -b /company/sales -m -s /bin/sh suser1
sudo useradd -g company -G sales -b /company/sales -m -s /bin/sh suser2
sudo useradd -g company -G sales -b /company/sales -m -s /bin/sh sadmin

# Права
sudo chown root:it /company/it && sudo chmod 750 /company/it
sudo chown root:sales /company/sales && sudo chmod 750 /company/sales
sudo chown root:company /company/shared && sudo chmod 775 /company/shared

# Файлы в shared
sudo -u ituser1 touch /company/shared/ituser1
sudo -u ituser2 touch /company/shared/ituser2
sudo -u itadmin touch /company/shared/itadmin
sudo -u suser1 touch /company/shared/suser1
sudo -u suser2 touch /company/shared/suser2
sudo -u sadmin touch /company/shared/sadmin
```

### Описание команд

| Команда | Что делает |
|---------|------------|
| `groupadd` | создаёт новую группу |
| `mkdir -p` | создаёт директорию и все родительские директории |
| `useradd -g` | задаёт основную группу |
| `useradd -G` | задаёт дополнительные группы |
| `useradd -b` | базовая директория для домашней папки |
| `useradd -m` | создаёт домашнюю директорию |
| `useradd -s` | задаёт оболочку (bash/sh) |
| `chown root:group` | меняет владельца и группу |
| `chmod 750` | `rwxr-x---` |
| `sudo -u user` | выполняет команду от имени другого пользователя |

### Результат

| Директория | Владелец | Группа | Права | Доступ |
|------------|----------|--------|-------|--------|
| `/company/it` | root | it | `750` | root, группа it |
| `/company/sales` | root | sales | `750` | root, группа sales |
| `/company/shared` | root | company | `775` | все (чтение/запись для группы) |


---

## Задание 2: Академия (academy, teachers, students, staff)

### Выполненные команды

```bash
# Группы
sudo groupadd academy teachers students staff

# Директории
sudo mkdir -p /academy/students /academy/teachers /academy/staff /academy/secret

# Пользователи
sudo useradd -g students -b /academy/students -m student
sudo useradd -g teachers -b /academy/teachers -m teacher
sudo useradd -g staff -b /academy/staff -m staffuser

# Права на директории
sudo chown root:students /academy/students && sudo chmod 750 /academy/students
sudo chown root:teachers /academy/teachers && sudo chmod 750 /academy/teachers
sudo chown root:staff /academy/staff && sudo chmod 750 /academy/staff

# Secret директория (SGID + Sticky)
sudo chown root:teachers /academy/secret
sudo chmod 1770 /academy/secret
sudo setfacl -m g:staff:rwx /academy/secret

# Программа spy
sudo tee /usr/local/bin/spy << 'EOF'
#!/bin/bash
ls -la /academy/secret
EOF
sudo chmod 755 /usr/local/bin/spy

# sudoers для студентов
echo "%students ALL=(:teachers) /usr/local/bin/spy" | sudo tee /etc/sudoers.d/students
echo "%students ALL=(:teachers) /bin/ls" | sudo tee -a /etc/sudoers.d/students
echo "%students ALL=(:teachers) /bin/rm /academy/secret/*" | sudo tee -a /etc/sudoers.d/students
sudo chmod 440 /etc/sudoers.d/students

# Проверка
sudo -u teacher touch /academy/secret/teacher_file
sudo -u student /usr/local/bin/spy
sudo -u teacher rm /academy/secret/teacher_file
```

### Описание команд

| Команда | Что делает |
|---------|------------|
| `groupadd` | создаёт новую группу |
| `chmod 1770` | `rwxrwx---` + sticky bit |
| `setfacl -m g:group:rwx` | даёт права группе через ACL |
| `tee` | создаёт файл и записывает в него содержимое |
| `chmod 755` | `rwxr-xr-x` (для исполняемых скриптов) |
| `visudo -f` | редактирует файлы sudoers |
| `chmod 440` | права для файлов sudoers (`r--r-----`) |
| `sudo -u user` | выполняет команду от имени другого пользователя |

### Результат

| Директория | Владелец | Группа | Права | Sticky | SGID | ACL |
|------------|----------|--------|-------|--------|------|-----|
| `/academy/secret` | root | teachers | `1770` | да | нет | staff:rwx |

---

## Задание 3: Компания (home/company)

### Выполненные команды

```bash
# Группы
sudo groupadd company sales marketing

# Директории
sudo mkdir -p /home/company/homedirs /home/company/sales /home/company/marketing /home/company/shared

# Пользователи
sudo useradd -g sales -G company -b /home/company/homedirs -m mike
sudo useradd -g marketing -G company -b /home/company/homedirs -m john

# Права
sudo chown :sales /home/company/sales && sudo chmod 770 /home/company/sales
sudo chown :marketing /home/company/marketing && sudo chmod 770 /home/company/marketing
sudo chown :company /home/company/shared && sudo chmod 2770 /home/company/shared

# Проверка SGID
sudo -u mike touch /home/company/shared/testfile
```

### Описание команд

| Команда | Что делает |
|---------|------------|
| `chown :group dir` | меняет только группу владельца |
| `chmod 770` | `rwxrwx---` |
| `chmod 2770` | `rwxrwx---` + SGID (группа наследуется) |
| `sudo -u mike touch` | создаёт файл от имени пользователя mike |

### Что такое SGID на директории

- SGID (Set Group ID) — бит, который заставляет все новые файлы в директории наследовать группу директории, а не группу пользователя.

### Результат

| Директория | Владелец | Группа | Права | SGID |
|------------|----------|--------|-------|------|
| `/home/company/sales` | root | sales | `770` | нет |
| `/home/company/marketing` | root | marketing | `770` | нет |
| `/home/company/shared` | root | company | `2770` | да |

---

## Задание 4: Директории /data

### Выполненные команды

```bash
# Группы и директории
sudo groupadd marketing sales it
sudo mkdir -p /data/marketing /data/sales /data/it

# Пользователи
sudo useradd -g marketing -d /data/marketing -m user.marketing
sudo useradd -g sales -d /data/sales -m user.sales
sudo useradd -G marketing,sales,it -d /home/admin -m admin
sudo useradd -d /home/backup -m backup

# Права
sudo chown user.marketing:marketing /data/marketing && sudo chmod 770 /data/marketing
sudo chown user.sales:sales /data/sales && sudo chmod 770 /data/sales
sudo chown root:it /data/it && sudo chmod 770 /data/it

# ACL для backup
sudo setfacl -m u:backup:r-x /data/marketing
sudo setfacl -m u:backup:r-x /data/sales
sudo setfacl -m u:backup:r-x /data/it

# ACL для admin
sudo setfacl -m u:admin:rwx /data/marketing
sudo setfacl -m u:admin:rwx /data/sales
sudo setfacl -m u:admin:rwx /data/it
```

### Описание команд

| Команда | Что делает |
|---------|------------|
| `useradd -d` | задаёт полный путь домашней директории |
| `setfacl -m u:user:rx` | даёт права пользователю через ACL |
| `getfacl` | показывает ACL на файле/директории |

### Что такое ACL (Access Control Lists)

- Расширение стандартных прав Linux
- Позволяет давать права конкретным пользователям без добавления в группы
- Применяется мгновенно, не требует перезахода

### Результат

| Директория | Владелец | Группа | Права | ACL (backup) | ACL (admin) |
|------------|----------|--------|-------|--------------|-------------|
| `/data/marketing` | user.marketing | marketing | `770` | `r-x` | `rwx` |
| `/data/sales` | user.sales | sales | `770` | `r-x` | `rwx` |
| `/data/it` | root | it | `770` | `r-x` | `rwx` |

**Задание 4 выполнено**

---

## Задание 5: Корневая директория /company

### Выполненные команды

```bash
# Директория и группа
sudo mkdir /company
sudo groupadd company
sudo chown :company /company
sudo chmod 770 /company
sudo chmod g+s /company
sudo chmod +t /company

# Проверка
sudo useradd testuser
sudo usermod -aG company testuser
sudo -u testuser touch /company/testfile
ls -l /company/testfile   # группа должна быть company
```

### Описание команд

| Команда | Что делает |
|---------|------------|
| `chmod g+s` | включает SGID |
| `chmod +t` | включает sticky bit |
| `usermod -aG` | добавляет пользователя в группу |

### Что такое Sticky bit

- Бит, который запрещает пользователям удалять чужие файлы в директории
- Только владелец файла, владелец директории или root могут удалить файл

### Что такое SGID

- Бит, который заставляет новые файлы наследовать группу директории

### Результат

| Директория | Владелец | Группа | Права | SGID | Sticky |
|------------|----------|--------|-------|------|--------|
| `/company` | root | company | `1770` | да | да |


---



### Права доступа (цифры)

| Цифры | Буквы | Значение |
|-------|-------|----------|
| 7 | `rwx` | чтение + запись + выполнение |
| 6 | `rw-` | чтение + запись |
| 5 | `r-x` | чтение + выполнение |
| 4 | `r--` | только чтение |
| 0 | `---` | нет прав |

### Специальные биты

| Бит | Команда | Значение |
|-----|---------|----------|
| SUID | `chmod u+s` | выполнение от имени владельца |
| SGID | `chmod g+s` | наследование группы (на директориях) |
| Sticky | `chmod +t` | только владелец удаляет файлы |

### ACL

| Команда | Значение |
|---------|----------|
| `setfacl -m u:user:rwx dir` | дать права пользователю |
| `setfacl -m g:group:rx dir` | дать права группе |
| `setfacl -b dir` | удалить все ACL |
| `getfacl dir` | просмотреть ACL |

### Пользователи и группы

| Команда | Значение |
|---------|----------|
| `useradd -g group user` | создать с основной группой |
| `useradd -G group1,group2 user` | с дополнительными группами |
| `usermod -aG group user` | добавить в группу |
| `groups user` | показать группы пользователя |

---

## Заключение
Освоил:
- Права доступа (`chmod`, `chown`)
- Специальные биты (SGID, Sticky bit)
- ACL (`setfacl`, `getfacl`)
- Создание пользователей и групп (`useradd`, `groupadd`, `usermod`)
- Настройка `sudoers` для ограниченных прав
- Создание скриптов (`spy`) и их интеграция с `sudo`
