## Вопросы

### Вопрос 1: Расшифровка строки из /etc/passwd

**Пример строки:**
```
root:x:0:0:Super User:/root:/bin/bash
```

**Расшифровка полей:**

| Поле | Значение | Что означает |
|------|----------|--------------|
| `root` | имя пользователя | логин для входа |
| `x` | пароль | зашифрованный пароль хранится в `/etc/shadow` |
| `0` | UID | идентификатор пользователя (0 = root) |
| `0` | GID | идентификатор основной группы |
| `Super User` | комментарий | полное имя или описание (GECOS) |
| `/root` | домашняя директория | куда попадает пользователь после входа |
| `/bin/bash` | оболочка | какой shell запускается |

---

### Вопрос 2: Файлы с информацией о пользователях и группах

| Что хранится | Файл |
|--------------|------|
| Информация о пользователях | `/etc/passwd` |
| Зашифрованные пароли пользователей | `/etc/shadow` |
| Информация о группах | `/etc/group` |
| Пароли групп | `/etc/gshadow` |

---

## Задание 1: Создание пользователей IT-отдела

### Условие
- Создать директорию `/home/it`
- Создать группу `IT` с идентификатором 10000
- Создать пользователей: `sysadmin`, `helpdesk`, `netadmin`
- Домашние директории в `/home/it`
- Основная группа — `IT`
- Оболочка — `/bin/bash`
- UID: 10101, 10201, 10301

### Выполнение

```bash
# 1. Создание директории
sudo mkdir -p /home/it

# 2. Создание группы с GID 10000
sudo groupadd -g 10000 IT

# 3. Создание пользователей
sudo useradd sysadmin -b /home/it -g IT -u 10101 -s /bin/bash
sudo useradd helpdesk -b /home/it -g IT -u 10201 -s /bin/bash
sudo useradd netadmin -b /home/it -g IT -u 10301 -s /bin/bash

# 4. Задание паролей
sudo passwd sysadmin
sudo passwd helpdesk
sudo passwd netadmin
```

### Проверка

```bash
# Просмотр созданных пользователей
tail -3 /etc/passwd
# sysadmin:x:10101:10000::/home/it/sysadmin:/bin/bash
# helpdesk:x:10201:10000::/home/it/helpdesk:/bin/bash
# netadmin:x:10301:10000::/home/it/netadmin:/bin/bash
```

---

## Задание 2: Настройка прав через sudoers

### Условие
- `sysadmin` — все права на систему
- `netadmin` — возможность запускать `nmtui` от root
- `helpdesk` — возможность менять пароли всех, кроме `sysadmin` и `root`

### Выполнение

Создан файл `/etc/sudoers.d/custom`:

```bash
# sysadmin — полные права
sysadmin ALL=(ALL) ALL

# netadmin — только nmtui от root
netadmin ALL=(root) /usr/bin/nmtui

# helpdesk — смена паролей (кроме sysadmin и root)
helpdesk ALL=(ALL) /usr/bin/passwd, !/usr/bin/passwd sysadmin, !/usr/bin/passwd root
```

### Проверка

| Пользователь | Команда | Результат |
|--------------|---------|-----------|
| sysadmin | `sudo cat /etc/shadow` | успешно |
| netadmin | `sudo nmtui` | открывается |
| netadmin | `sudo cat /etc/shadow` | Permission denied |
| helpdesk | `sudo passwd netadmin` | успешно |
| helpdesk | `sudo passwd sysadmin` | Permission denied |

---

## Задание 3: Создание пользователей для отделов

### Условие
- Создать группы: `marketing`, `sales`, `it`
- Создать директории: `/home/marketing`, `/home/sales`, `/home/it`
- Создать пользователей:

| Группа | Пользователи | Может логиниться? | Домашняя директория |
|--------|--------------|-------------------|---------------------|
| marketing | `user.marketing` | нет | `/home/marketing` |
| marketing | `manager.marketing` | да | `/home/marketing/manager.marketing` |
| sales | `user.sales` | нет | `/home/sales` |
| sales | `manager.sales` | да | `/home/sales/manager.sales` |
| it | `admin` | да | `/home/it/admin` |
| it | `helpdesk` | да | `/home/it/helpdesk` |

### Выполнение

#### 1. Создание групп
```bash
sudo groupadd marketing
sudo groupadd sales
sudo groupadd it
```

#### 2. Создание директорий
```bash
sudo mkdir -p /home/marketing /home/sales /home/it
```

#### 3. Создание пользователей

**Группа marketing:**
```bash
# user.marketing (не может логиниться)
sudo useradd -g marketing -d /home/marketing -M -s /sbin/nologin user.marketing

# manager.marketing (может логиниться)
sudo useradd -g marketing -b /home/marketing -m -s /bin/bash manager.marketing
```

**Группа sales:**
```bash
# user.sales (не может логиниться)
sudo useradd -g sales -d /home/sales -M -s /sbin/nologin user.sales

# manager.sales (может логиниться)
sudo useradd -g sales -b /home/sales -m -s /bin/bash manager.sales
```

**Группа it:**
```bash
# admin
sudo useradd -g it -b /home/it -m -s /bin/bash admin

# helpdesk
sudo useradd -g it -b /home/it -m -s /bin/bash helpdesk
```

#### 4. Задание паролей (для manager-ов, admin, helpdesk)
```bash
sudo passwd manager.marketing
sudo passwd manager.sales
sudo passwd admin
sudo passwd helpdesk
```

### Проверка

```bash
# Просмотр всех созданных пользователей
tail -6 /etc/passwd
```

**Ожидаемый результат:**

| Пользователь | Группа | Оболочка | Домашняя директория |
|--------------|--------|----------|---------------------|
| user.marketing | marketing | `/sbin/nologin` | `/home/marketing` |
| manager.marketing | marketing | `/bin/bash` | `/home/marketing/manager.marketing` |
| user.sales | sales | `/sbin/nologin` | `/home/sales` |
| manager.sales | sales | `/bin/bash` | `/home/sales/manager.sales` |
| admin | it | `/bin/bash` | `/home/it/admin` |
| helpdesk | it | `/bin/bash` | `/home/it/helpdesk` |

---


### Управление пользователями

| Команда | Что делает |
|---------|-------------|
| `useradd name` | создать пользователя |
| `useradd -g group name` | создать с основной группой |
| `useradd -d /path name` | указать домашнюю директорию |
| `useradd -m` | создать домашнюю директорию |
| `useradd -M` | не создавать домашнюю директорию |
| `useradd -s /sbin/nologin` | запретить вход |
| `userdel name` | удалить пользователя |
| `userdel -r name` | удалить с домашней директорией |
| `passwd name` | задать/изменить пароль |

### Управление группами

| Команда | Что делает |
|---------|-------------|
| `groupadd group` | создать группу |
| `groupadd -g GID group` | создать с указанным GID |
| `groupdel group` | удалить группу |
| `usermod -aG group user` | добавить пользователя в группу |

### Просмотр информации

| Команда | Что показывает |
|---------|----------------|
| `cat /etc/passwd` | все пользователи |
| `cat /etc/group` | все группы |
| `tail -n /etc/passwd` | последних n пользователей |
| `id user` | информация о пользователе |
| `groups user` | группы пользователя |

### Sudoers (права)

| Правило | Что даёт |
|---------|----------|
| `user ALL=(ALL) ALL` | полные права sudo |
| `user ALL=(root) /usr/bin/command` | выполнять команду от root |
| `user ALL=(ALL) /usr/bin/passwd, !/usr/bin/passwd root` | менять пароли, кроме root |

---

## Итог

- Научился читать и понимать структуру `/etc/passwd`
- Узнал, в каких файлах хранится информация о пользователях и группах
- Создал пользователей с разными параметрами (UID, GID, оболочка, домашняя директория)
- Настроил права через `sudoers` (полные права, ограниченные команды)
- Создал пользователей для разных отделов с разными правами на вход
