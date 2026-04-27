# Логирование в Linux

### Ответы на теоретические вопросы

**1. Куда программы ведут свои логи?**
Большинство программ записывают логи в директорию `/var/log`. Системные службы могут использовать `syslog` или `journald`.

**2. Что такое syslog? Как программы отправляют логи в syslog?**
Это стандарт и протокол для отправки и регистрации сообщений о событиях. Программы используют библиотеку `libc` (функция `syslog()`) или системную утилиту `logger`.

**3. Какие объекты (facilities) логирования есть у syslog?**
Основные объекты: `auth`, `authpriv`, `cron`, `daemon`, `kern`, `lpr`, `mail`, `news`, `syslog`, `user`, `uucp`, а также зарезервированные пользовательские каналы от `local0` до `local7`.

**4. В чем отличие journald от rsyslog?**
`journald` — это часть `systemd`, она собирает логи на ранних этапах загрузки и хранит их в бинарном формате. `rsyslog` — классический демон, который работает с текстовыми файлами и обладает мощными инструментами фильтрации и удаленной передачи данных.

**5. Зачем в системе два демона логирования?**
`journald` обеспечивает современный сбор структурированных данных (метаданные процессов), а `rsyslog` отвечает за текстовые файлы логов, к которым привыкли администраторы, и за совместимость со старым софтом.

**6. Как сделать так, чтобы логи с journald оставались после перезагрузки?**
Нужно создать директорию `/var/log/journal` (система сама начнет туда писать) или в конфиге `/etc/systemd/journald.conf` выставить параметр `Storage=persistent`.

**7. Как изменить время хранения логов journald?**
Это делается через параметры `MaxRetentionSec` (время) или `MaxUsage` (объем диска) в файле конфигурации `/etc/systemd/journald.conf`.

**8. Как посмотреть сообщения ядра?**
С помощью команды `dmesg` или через фильтр в журнале: `journalctl -k`.

**9. Как посмотреть логи определённого сервиса?**
Используйте команду: `journalctl -u <service_name>`.

**10. Как посмотреть последние логи сервиса?**
Добавьте флаг `-n`: `journalctl -u <service_name> -n 50` (покажет последние 50 строк).

**11. Как посмотреть логи с последнего запуска системы?**
Используйте флаг `-b`: `journalctl -b`.

**12. Что такое ротация логов и как её настроить?**
Это процесс циклической замены файлов логов (сжатие старых, удаление совсем древних и создание новых), чтобы диск не переполнился. Настраивается в `/etc/logrotate.conf` и в директории `/etc/logrotate.d/`.

**13. Как отправить свои сообщения в syslog?**
С помощью утилиты `logger`. Например: `logger -p user.info "Мое сообщение"`.

**14. Как смотреть логи определённого сервиса в реальном времени?**
С помощью флага `-f` (follow): `journalctl -u <service_name> -f` или классическим способом `tail -f /var/log/messages`.

---

## 2. Выполнение практических заданий

### Задание 1: Автоматизация сбора логов
**Цель:** Создать скрипт для захвата последней строки лога и настройки его хранения.

**Скрипт `task1.sh`:**
```bash
#!/usr/bin/env bash
tail -1 /var/log/messages > file
logger -p news.alert "Logs collected"
```

**Конфигурация rsyslog (`/etc/rsyslog.d/collect.conf`):**
```text
news.alert    /var/log/collect.log
```

**Конфигурация logrotate (`/etc/logrotate.d/collect`):**
```text
/var/log/collect.log {
    weekly
    rotate 4
    create
    missingok
    notifempty
}
```

---

### Задание 2: Сортировка логов по уровням важности
**Цель:** Настроить раздельное хранение логов разных приоритетов с разной политикой ротации.

**Скрипт `task2.sh`:**
```bash
#!/usr/bin/env bash
logger -p local0.debug "This is normal log"
logger -p local0.error "This is error"
logger -p local0.crit "This is critical error"
```

**Конфигурация rsyslog (`/etc/rsyslog.d/mylogs.conf`):**
```text
local0.=debug    /var/log/mylog.debug
local0.=error    /var/log/mylog.error
local0.=crit     /var/log/mylog.crit
```

**Конфигурация logrotate (`/etc/logrotate.d/mylogs`):**
```text
/var/log/mylog.debug {
    daily
    rotate 5
    compress
}
/var/log/mylog.error {
    daily
    rotate 5
    nocompress
}
/var/log/mylog.crit {
    weekly
    rotate 5
}
```

---

### Задание 3: Контроль доступа и модификация метаданных
**Цель:** Написать скрипт с проверкой прав пользователя по списку и интерактивным изменением даты файлов.

**Список разрешенных пользователей (`/etc/allowedusers`):**
```text
sancher
```

**Скрипт `task3.sh`:**
```bash
#!/usr/bin/env bash

check=1
for line in $(cat /etc/allowedusers)
do
    if [ $(id -u) = $(grep "^$line:" /etc/passwd | cut -d: -f3) ]  
    then 
        check=0 
    fi
done

if [ $check -eq 1 ]
then 
    echo "This incident will be reported"
    logger -p local0.error "User $USER tried to run this script!"
    exit 1
else
    echo "Доступ разрешен. Список файлов в /data/:"
    ls /data/
    read -p "Введите имя файла: " filename
    read -p "Введите дату и время (ГГГГММДДччмм): " filedate
    
    if [ -f "/data/$filename" ]; then
        touch -t "$filedate" "/data/$filename"
        echo "Дата модификации успешно изменена."
    else
        echo "Файл не найден."
    fi
fi
```

