# Межсетевой экран - firewalld
## 1. Концепция и архитектура

### Зачем нужен файрвол на системном уровне?

**Многоуровневая безопасность (Security in Depth):**
```
Уровень 1: Сетевая безопасность (firewalld)
    ↓ блокирует соединения ДО того, как они достигнут сервиса
Уровень 2: Безопасность процессов (SELinux)
    ↓ не позволяет процессу выполнять несанкционированные действия
Уровень 3: Безопасность приложений
    ↓ настройки самого сервиса
```

**Почему нельзя полагаться только на SELinux?**
- SELinux защищает **после** того, как соединение установлено
- Если злоумышленник не может **подключиться** к SSH - он не сможет и эксплуатировать уязвимости
- Файрвол - первый рубеж обороны

### Полная архитектура (от ядра до пользователя)

```
┌─────────────────────────────────────────────────────────────┐
│                     ПОЛЬЗОВАТЕЛЬСКИЙ УРОВЕНЬ                  │
├─────────────────────────────────────────────────────────────┤
│  firewall-cmd (команда для управления)                      │
│       ↓                                                      │
│  firewalld (демон, systemd service)                         │
│       ↓                                                      │
│  /etc/firewalld/ (пользовательские конфигурации)            │
│  /lib/firewalld/ (системные конфигурации по умолчанию)      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                     УРОВЕНЬ СИСТЕМНЫХ ВЫЗОВОВ                 │
├─────────────────────────────────────────────────────────────┤
│  nft (команда) / libnftables (библиотека)                   │
│       ↓                                                      │
│  nftables (фреймворк в пространстве пользователя)           │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                         УРОВЕНЬ ЯДРА                          │
├─────────────────────────────────────────────────────────────┤
│  netfilter (фреймворк в ядре Linux)                         │
│  - Хук-точки (hook points): PREROUTING, INPUT, FORWARD,     │
│    OUTPUT, POSTROUTING                                       │
│  - Таблицы: filter, nat, mangle, raw, security             │
└─────────────────────────────────────────────────────────────┘
```

### Ключевое понимание

**netfilter** - это каркас (framework) в ядре Linux:
- Работает на уровне ядра
- Содержит "хуки" (точки перехвата) в сетевом стеке
- Ничего не блокирует сам по себе, только предоставляет механизмы

**nftables** - замена iptables (с 2014 года):
- Инструмент для управления netfilter
- Заменяет старые iptables, ip6tables, arptables, ebtables
- Использует единый синтаксис

**firewalld** - демон для управления nftables:
- Добавляет концепцию "зон"
- Позволяет изменять правила на лету (без перезагрузки)
- Упрощает администрирование

### Историческая справка

```
2007 и ранее: ipfwadm → ipchains → iptables
2014: nftables (новая генерация)
Сейчас: firewalld (поверх nftables) в RHEL-based системах
```

---

## 2. Установка и базовая работа

### Проверка и управление демоном

```bash
# Проверка статуса firewalld
sudo systemctl status firewalld
# Вывод покажет: active (running) или inactive (dead)

# Запуск (если не запущен)
sudo systemctl start firewalld

# Автозапуск при загрузке
sudo systemctl enable firewalld

# Остановка (только для отладки!)
sudo systemctl stop firewalld

# Перезапуск
sudo systemctl restart firewalld
```

### Базовые операции firewall-cmd

```bash
# Показать всё (текущая зона)
sudo firewall-cmd --list-all

# Пример вывода:
# public (active)
#   target: default
#   icmp-block-inversion: no
#   interfaces: enp0s3 enp0s8
#   sources: 
#   services: cockpit dhcpv6-client ssh
#   ports: 2233/tcp
#   protocols: 
#   forward: no
#   masquerade: no
#   forward-ports: 
#   source-ports: 
#   icmp-blocks: 
#   rich rules:
```

### Важные ключи управления

```bash
# Временные изменения (исчезают после reload)
sudo firewall-cmd --add-port=8080/tcp
sudo firewall-cmd --remove-port=8080/tcp

# Постоянные изменения (требуют reload)
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --remove-port=8080/tcp --permanent

# Применение постоянных правил
sudo firewall-cmd --reload

# Сохранить текущие временные правила как постоянные
sudo firewall-cmd --runtime-to-permanent
```

### Режим паники

```bash
# Активация паники (обрывает ВСЕ соединения)
sudo firewall-cmd --panic-on
# Проверка
sudo firewall-cmd --query-panic

# Деактивация
sudo firewall-cmd --panic-off
```

**Когда использовать:** При обнаружении взлома, DDoS-атаки или любой другой угрозы, требующей немедленной изоляции сервера.

---

## 3. Зоны (Zones) - Главная концепция

### Концепция зон

**Основная идея:** Разные сети имеют разный уровень доверия.

```
Интернет (недоверенная сеть)    Домашняя сеть (доверенная)
    ↓                                    ↓
Зона public                      Зона home
- Минимум сервисов               - Больше разрешений
- Строгие ограничения            - Можно файлы шарить
- DROP неизвестных пакетов       - ACCEPT доверенных
```

### Полный список зон и их особенности

| Зона | Уровень доверия | Типичное использование | Поведение по умолчанию |
|------|----------------|------------------------|------------------------|
| **trusted** | Полное | Внутренние сервера, кластеры | Все пакеты ACCEPT |
| **home** | Высокий | Домашние сети | Разрешены common сервисы |
| **internal** | Высокий | Корпоративная сеть | Как home + особые правила |
| **work** | Средний | Офисные сети | Разрешены рабочие сервисы |
| **public** | Низкий | Общественные WiFi | Только базовые сервисы |
| **external** | Низкий | Внешние сети | Включён masquerade |
| **dmz** | Ограниченный | Демилитаризованная зона | Ограниченный доступ |
| **block** | Минимальный | Чёрные списки | Все пакеты REJECT |
| **drop** | Минимальный | Максимальная скрытность | Все пакеты DROP |

### Работа с зонами

```bash
# Просмотр всех зон
sudo firewall-cmd --get-zones
# Вывод: block dmz drop external home internal public trusted work

# Узнать активную зону по умолчанию
sudo firewall-cmd --get-default-zone

# Сменить зону по умолчанию
sudo firewall-cmd --set-default-zone=work

# Посмотреть правила конкретной зоны
sudo firewall-cmd --list-all --zone=drop

# Посмотреть все зоны и их правила
sudo firewall-cmd --list-all-zones
```

### Детальный разбор зон на примерах

#### Зона trusted - полное доверие
```bash
sudo firewall-cmd --zone=trusted --list-all
# target: ACCEPT
# Все пакеты разрешены, никаких ограничений
```

#### Зона block - явный отказ с уведомлением
```bash
sudo firewall-cmd --zone=block --list-all
# target: REJECT
# ping 192.168.1.204 → Packet filtered (ответ от системы)
```

#### Зона drop - молчаливый отказ
```bash
sudo firewall-cmd --zone=drop --list-all
# target: DROP
# ping 192.168.1.204 → ... (просто таймаут, без ответа)
```

---

## 4. Сервисы и порты

### Концепция сервисов

**Сервис** - это именованный шаблон, который включает:
- Один или несколько портов (tcp/udp)
- Опционально: модули помощи, адреса назначения

**Почему сервисы лучше портов?**
```bash
# Через порт - непонятно, что это
sudo firewall-cmd --add-port=22/tcp

# Через сервис - ясно, что это SSH
sudo firewall-cmd --add-service=ssh
```

### Файлы сервисов

```bash
# Системные сервисы (не редактировать!)
ls /lib/firewalld/services/
# ssh.xml http.xml https.xml ftp.xml ...

# Пользовательские сервисы (переопределяют системные)
ls /etc/firewalld/services/

# Просмотр содержимого сервиса
cat /lib/firewalld/services/ssh.xml
```

**Пример содержимого ssh.xml:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>SSH</short>
  <description>Secure Shell (SSH) is a protocol for secure remote login...</description>
  <port protocol="tcp" port="22"/>
</service>
```

### Управление сервисами

```bash
# Список разрешённых сервисов
sudo firewall-cmd --list-services

# Информация о сервисе
sudo firewall-cmd --info-service=ssh
# Вывод:
# ssh
#   ports: 22/tcp
#   protocols: 
#   source-ports: 
#   modules: 
#   destination: 

# Добавление сервиса
sudo firewall-cmd --add-service=http --permanent

# Удаление сервиса
sudo firewall-cmd --remove-service=dhcpv6-client --permanent

# Создание своего сервиса
sudo cp /lib/firewalld/services/ssh.xml /etc/firewalld/services/myssh.xml
sudo nano /etc/firewalld/services/myssh.xml
# Изменить порт на 2233
sudo firewall-cmd --reload
sudo firewall-cmd --add-service=myssh --permanent
```

### Работа с портами

```bash
# Добавление порта (временное)
sudo firewall-cmd --add-port=8080/tcp

# Добавление порта (постоянное)
sudo firewall-cmd --add-port=8080/tcp --permanent

# Диапазон портов
sudo firewall-cmd --add-port=5000-5100/tcp --permanent

# Удаление порта
sudo firewall-cmd --remove-port=8080/tcp --permanent

# Список открытых портов
sudo firewall-cmd --list-ports

# Source-порты (исходящие)
sudo firewall-cmd --add-source-port=10050-10060/tcp
```

### Практический пример смены порта SSH

```bash
# 1. Меняем порт в конфиге SSH
sudo nano /etc/ssh/sshd_config
# Port 2233

# 2. Перезапускаем SSH
sudo systemctl restart sshd

# 3. Настройка firewalld (вариант через порт)
sudo firewall-cmd --add-port=2233/tcp --permanent
sudo firewall-cmd --remove-service=ssh --permanent

# 4. Или через изменение сервиса
sudo firewall-cmd --permanent --service=ssh --remove-port=22/tcp
sudo firewall-cmd --permanent --service=ssh --add-port=2233/tcp
sudo firewall-cmd --reload

# 5. Проверка
ssh user@server -p 2233
```

---

## 5. Приоритеты и порядок применения правил

### Главное правило: конкретнее = важнее

```
Приоритет (от highest до lowest):
┌─────────────────────────────────────────────┐
│  1. Source-based rules (IP-адреса)         │ ← HIGHEST
│     - Конкретный IP переопределяет всё      │
├─────────────────────────────────────────────┤
│  2. Interface-based rules (интерфейсы)     │
│     - Для трафика через конкретный интерфейс│
├─────────────────────────────────────────────┤
│  3. Default zone (зона по умолчанию)        │ ← LOWEST
│     - Для всего остального                  │
└─────────────────────────────────────────────┘
```

### Детальный механизм принятия решений

```
Пакет пришёл на интерфейс enp0s3 от IP 192.168.56.1

Шаг 1: Есть ли source rule для 192.168.56.1?
    ├─ ДА → Применяем зону из source rule
    └─ НЕТ → Идём к шагу 2

Шаг 2: Есть ли interface rule для enp0s3?
    ├─ ДА → Применяем зону из interface rule
    └─ НЕТ → Идём к шагу 3

Шаг 3: Применяем default zone
```

### Практический пример с пояснениями

```bash
# Исходная конфигурация
# Интерфейс enp0s3 в зоне trusted (всё разрешено)
sudo firewall-cmd --zone=trusted --add-interface=enp0s3

# Интерфейс enp0s8 в зоне public (ограниченно)
sudo firewall-cmd --zone=public --add-interface=enp0s8

# СИТУАЦИЯ 1: Приходит пакет от 192.168.1.100 на enp0s3
# Применяется trusted (interface rule)

# СИТУАЦИЯ 2: Добавляем исключение
sudo firewall-cmd --zone=block --add-source=192.168.1.100

# Теперь пакет от 192.168.1.100 на ЛЮБОМ интерфейсе получит блокировку!
# Потому что source rule переопределяет interface rule

# Проверка
sudo firewall-cmd --list-all --zone=block
# sources: 192.168.1.100

# Удаление исключения
sudo firewall-cmd --zone=block --remove-source=192.168.1.100
```

### Пример с несколькими source правилами

```bash
# Разрешаем всей офисной сети (доверенные)
sudo firewall-cmd --zone=trusted --add-source=192.168.10.0/24

# Но конкретный IP в офисе - проблемный (блокируем)
sudo firewall-cmd --zone=drop --add-source=192.168.10.50

# Результат:
# 192.168.10.50 → блокируется (более конкретное правило)
# 192.168.10.1-49,51-254 → разрешены
```

### Приоритет между source правилами

```bash
# Более конкретная сеть имеет приоритет
sudo firewall-cmd --zone=trusted --add-source=192.168.0.0/16
sudo firewall-cmd --zone=block --add-source=192.168.1.0/24
sudo firewall-cmd --zone=drop --add-source=192.168.1.50/32

# Приоритет: /32 > /24 > /16 > default
```

---

## 6. Интерфейсы и источники

### Управление интерфейсами

```bash
# Просмотр активных зон с интерфейсами
sudo firewall-cmd --get-active-zones
# Пример вывода:
# public
#   interfaces: enp0s3
# trusted
#   interfaces: enp0s8

# Добавить интерфейс в зону (без удаления из старой)
sudo firewall-cmd --add-interface=enp0s9 --zone=work

# Переместить интерфейс в другую зону
sudo firewall-cmd --change-interface=enp0s8 --zone=public

# Удалить интерфейс из зоны
sudo firewall-cmd --remove-interface=enp0s8

# Список интерфейсов в зоне
sudo firewall-cmd --list-interfaces --zone=trusted
```

### Управление источниками (IP-адресами)

```bash
# Добавление источника
sudo firewall-cmd --zone=trusted --add-source=192.168.1.100
sudo firewall-cmd --zone=trusted --add-source=10.0.0.0/8

# Поддержка IPv6
sudo firewall-cmd --zone=trusted --add-source=2001:db8::/32

# Список источников в зоне
sudo firewall-cmd --list-sources --zone=block

# Удаление источника
sudo firewall-cmd --zone=block --remove-source=192.168.56.1

# Проверка, есть ли источник
sudo firewall-cmd --zone=block --query-source=192.168.56.1
```

### Полный пример с интерфейсами и источниками

```bash
# Задача: 
# - Интерфейс в интернет (enp0s3) - все пакеты дропать (скрытность)
# - Но с IP офиса (5.5.5.5) разрешить SSH и HTTP

# 1. Интерфейс в зону drop (всё дропается)
sudo firewall-cmd --zone=drop --add-interface=enp0s3 --permanent

# 2. IP офиса в зону public (ограниченно, но разрешены некоторые сервисы)
sudo firewall-cmd --zone=public --add-source=5.5.5.5 --permanent

# 3. Разрешаем нужные сервисы в зоне public
sudo firewall-cmd --zone=public --add-service=ssh --permanent
sudo firewall-cmd --zone=public --add-service=http --permanent

# 4. Применяем
sudo firewall-cmd --reload

# Результат:
# С 5.5.5.5: можно SSH и HTTP
# С других IP: тишина (DROP)
```

---

## 7. ICMP и контроль пакетов

### Что такое ICMP?

**ICMP** (Internet Control Message Protocol) - протокол для:
- Диагностики сети (ping, traceroute)
- Сообщения об ошибках (Destination Unreachable, Time Exceeded)
- Управлении маршрутизацией (Redirect)

### Типы ICMP:

```bash
# Список поддерживаемых типов ICMP
sudo firewall-cmd --get-icmptypes
# Пример: echo-request, echo-reply, destination-unreachable, time-exceeded

# Информация о типе
sudo firewall-cmd --info-icmptype=echo-request
```

### REJECT vs DROP - глубокое понимание

```
REJECT (отказ с уведомлением):
Клиент → пакет → Сервер (блокировка)
Сервер → ICMP type=3 (Destination Unreachable) → Клиент
Клиент видит: "Packet filtered" или "Connection refused"

DROP (отбрасывание без уведомления):
Клиент → пакет → Сервер (блокировка)
Сервер → (молчание, никакого ответа)
Клиент: ожидание таймаута (обычно 20-60 секунд)

Когда что использовать:
- REJECT: когда важна обратная связь (тестирование, внутренние сети)
- DROP: когда нужно скрыть существование сервера (публичные сервера)
```

### Управление ICMP блокировками

```bash
# Блокировка определённого типа ICMP
sudo firewall-cmd --add-icmp-block=echo-request
# Теперь ping не работает

# Снятие блокировки
sudo firewall-cmd --remove-icmp-block=echo-request

# Постоянная блокировка
sudo firewall-cmd --add-icmp-block=timestamp-request --permanent

# Список заблокированных ICMP типов
sudo firewall-cmd --list-icmp-blocks

# Инверсия блокировки (блокировать ВСЕ, кроме указанных)
sudo firewall-cmd --add-icmp-block-inversion
# Теперь разрешены только явно добавленные типы
sudo firewall-cmd --add-icmp-block=echo-request  # заблокирует ping
```

### Полный пример с ICMP

```bash
# Задача: 
# Сделать сервер "невидимым" для сканирования, но оставить возможность
# проверки доступности для доверенных IP

# 1. Ставим target DROP
sudo firewall-cmd --set-target=DROP --permanent

# 2. Для доверенных IP создаём зону с разрешённым ping
sudo firewall-cmd --zone=trusted --add-source=192.168.1.0/24 --permanent
sudo firewall-cmd --zone=trusted --add-icmp-block-inversion --permanent
sudo firewall-cmd --zone=trusted --add-icmp-block=echo-request --permanent

# Результат:
# Для 192.168.1.0/24 → пинг не блокируется (из-за инверсии)
# Для остальных → полная тишина
```

---

## 8. NAT и маскарадинг

### Основы NAT

**NAT** (Network Address Translation) - технология замены IP-адресов в пакетах.

```
Source NAT (SNAT) - замена адреса отправителя:
[Внутренняя сеть] 192.168.1.100 → [Роутер] → 5.5.5.5 → [Интернет]

Destination NAT (DNAT) - замена адреса получателя:
[Интернет] 5.5.5.5:8080 → [Роутер] → 192.168.1.100:80
```

### Masquerade (Source NAT)

```bash
# Включение маскарадинга
sudo firewall-cmd --add-masquerade --zone=external

# Проверка
sudo firewall-cmd --query-masquerade --zone=external

# Выключение
sudo firewall-cmd --remove-masquerade --zone=external

# Сделать постоянным
sudo firewall-cmd --add-masquerade --zone=external --permanent
```

**Когда нужен masquerade:**
- Linux как маршрутизатор
- Предоставление интернета внутренней сети
- VPN-сервер

### Forward Ports (Destination NAT / проброс портов)

```bash
# Формат: port=внешний_порт:proto=протокол:toport=внутренний_порт:toaddr=внутренний_IP

# Пример: проброс 8080 порта на внутренний сервер
sudo firewall-cmd --add-forward-port=port=8080:proto=tcp:toport=80:toaddr=192.168.1.100

# Проброс без смены порта
sudo firewall-cmd --add-forward-port=port=2222:proto=tcp:toaddr=192.168.1.50

# Проброс диапазона портов
sudo firewall-cmd --add-forward-port=port=10000-10100:proto=tcp:toport=5000-5100:toaddr=192.168.1.200

# Список проброшенных портов
sudo firewall-cmd --list-forward-ports

# Удаление
sudo firewall-cmd --remove-forward-port=port=8080:proto=tcp:toport=80:toaddr=192.168.1.100
```

### Полный пример: Linux как маршрутизатор

```bash
# Настройка на Linux сервере (192.168.1.1 - внутренний интерфейс, 
# enp0s3 - внешний интерфейс в интернет)

# 1. Включить IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
sudo echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf

# 2. Настроить firewall
sudo firewall-cmd --zone=external --add-interface=enp0s3 --permanent
sudo firewall-cmd --zone=internal --add-interface=enp0s8 --permanent
sudo firewall-cmd --zone=external --add-masquerade --permanent

# 3. Проброс порта 8080 на внутренний веб-сервер
sudo firewall-cmd --zone=external --add-forward-port=port=8080:proto=tcp:toport=80:toaddr=192.168.1.100 --permanent

# 4. Применить
sudo firewall-cmd --reload
```

---

## 9. Диагностика и тестирование

### Утилита ss (Socket Statistics)

**Что показывают сокеты:**
- ESTAB - установленные соединения
- LISTEN - сервера, ожидающие соединений
- TIME_WAIT - закрытые соединения в ожидании

```bash
# Базовые команды
ss -t     # TCP сокеты
ss -u     # UDP сокеты
ss -l     # Слушающие сокеты (LISTEN)
ss -a     # Все сокеты
ss -n     # Числовые порты (без разрешения имён)
ss -p     # Показать процесс
ss -4     # Только IPv4
ss -6     # Только IPv6

# Полезные комбинации
ss -tlnp  # Слушающие TCP порты с процессами (самая полезная!)
ss -ulnp  # Слушающие UDP порты
ss -tan   # Все TCP соединения с числовыми портами
ss -tunap # Всё и вся (TCP/UDP, все состояния, процессы)

# Пример вывода ss -tlnp
# State    Recv-Q Send-Q Local Address:Port   Peer Address:Port   Process
# LISTEN   0      128    0.0.0.0:22           0.0.0.0:*           users:(("sshd",pid=1234))
```

### Утилита nc (Netcat)

**Netcat - "швейцарский нож" сетевых утилит**

```bash
# Проверка порта (TCP)
nc -zv 192.168.1.204 22
# -z: нулевой ввод/вывод (только проверка)
# -v: подробный вывод

# Возможные результаты:
# Connection to 192.168.1.204 22 port [tcp/ssh] succeeded! - порт открыт
# Connection refused - порт закрыт, нет сервиса
# (таймаут) - пакеты дропаются или сеть недоступна

# Проверка UDP порта
nc -zuv 192.168.1.204 53

# Тестирование сервера (TCP)
# Сервер:
nc -l 1234
# Клиент:
nc 192.168.1.204 1234
# Теперь можно обмениваться сообщениями

# Передача файла
# Сервер:
nc -l 1234 > received_file.txt
# Клиент:
nc 192.168.1.204 1234 < file_to_send.txt

# Сканирование портов (простое)
nc -zv 192.168.1.204 20-30 2>&1 | grep succeeded
```

### Утилита telnet (альтернатива)

```bash
# Проверка порта
telnet 192.168.1.204 22

# Если порт открыт:
# Connected to 192.168.1.204
# SSH-2.0-OpenSSH_8.0

# Если закрыт:
# telnet: Unable to connect to remote host: Connection refused
```

### Практический сценарий отладки

```bash
# Ситуация: не работает веб-сервер на 8080 порту

# 1. Проверяем, слушает ли сервер
ss -tlnp | grep 8080

# 2. Если слушает, проверяем локально
curl localhost:8080

# 3. Проверяем с другого хоста через nc
nc -zv 192.168.1.204 8080

# 4. Проверяем firewall на сервере
sudo firewall-cmd --list-all | grep 8080

# 5. Смотрим логи firewall (если есть)
sudo journalctl -u firewalld | tail -20

# 6. Временно отключаем firewall для теста
sudo systemctl stop firewalld
# проверяем соединение
sudo systemctl start firewalld
```

---

## 10. Rich Rules (Богатые правила)

### Когда нужны Rich Rules?

Обычные правила (add-service, add-port) покрывают 80% случаев. Rich Rules нужны когда:
- Нужно логировать конкретные соединения
- Разрешить доступ только с определённых IP
- Ограничить частоту подключений
- Комбинировать несколько условий
- Добавить аудит

### Синтаксис Rich Rules

```bash
rule [family="ipv4|ipv6"] 
     [source address="IP" [invert="True"]]
     [destination address="IP" [invert="True"]]
     [service name="имя"]
     [port port="номер" protocol="tcp|udp"]
     [protocol value="proto"]
     [icmp-block name="тип"]
     [masquerade]
     [forward-port port="номер" protocol="tcp|udp" to-port="номер" to-addr="адрес"]
     [log [prefix="префикс"] [level="уровень"] [limit value="скорость"]]
     [audit]
     [accept|reject|drop|mark]
```

### Уровни логирования

| Уровень | Описание | Когда использовать |
|---------|----------|-------------------|
| emerg | Система нестабильна | Критические атаки |
| alert | Немедленное действие | Попытки взлома |
| crit | Критическое состояние | DoS атаки |
| err | Ошибки | Неудачные подключения |
| warning | Предупреждения | Подозрительная активность |
| notice | Важные уведомления | Нестандартные подключения |
| info | Информация | Нормальные подключения |
| debug | Отладка | Только при поиске проблем |

### Практические примеры Rich Rules

```bash
# 1. Разрешить SSH только с определённого IP
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.1.100" service name="ssh" accept'

# 2. Заблокировать конкретный IP на всех портах
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="10.0.0.5" drop'

# 3. Логировать все попытки подключения к SSH с логированием
sudo firewall-cmd --add-rich-rule='rule service name="ssh" log prefix="SSH_ATTEMPT" level="info" accept'

# 4. Ограничить количество подключений к SSH (не чаще 2 раз в минуту)
sudo firewall-cmd --add-rich-rule='rule service name="ssh" log limit value="2/m" accept'

# 5. Комплексный пример: разрешить доступ к 2233 порту только с IP 192.168.31.227
# и логировать с критическим уровнем (не чаще раза в минуту)
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.31.227" destination address="192.168.31.6" port port="2233" protocol="tcp" log prefix="MYSSH" level="crit" limit value="1/m" accept'

# 6. Блокировка всех ICMP, кроме нужного типа
sudo firewall-cmd --add-rich-rule='rule protocol value="icmp" icmp-block name="echo-request" drop'

# 7. Проброс порта с rich правилом
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" forward-port port="2222" protocol="tcp" to-port="22" to-addr="192.168.100.10" accept'

# 8. Разрешить доступ к MySQL только из локальной сети и логировать
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="3306" protocol="tcp" log prefix="MYSQL" level="notice" accept'
```

### Управление Rich Rules

```bash
# Просмотр всех rich правил
sudo firewall-cmd --list-rich-rules

# Удаление rich правила (скопировать правило из list-rich-rules)
sudo firewall-cmd --remove-rich-rule='rule family="ipv4" source address="192.168.1.100" service name="ssh" accept'

# Проверка синтаксиса (без применения)
sudo firewall-cmd --add-rich-rule='your rule here' --dry-run

# Постоянное сохранение
sudo firewall-cmd --add-rich-rule='your rule here' --permanent
sudo firewall-cmd --reload
```

### Просмотр логов Rich Rules

```bash
# Просмотр логов с префиксом MYSSH
sudo journalctl | grep MYSSH

# По уровню критичности
sudo journalctl -p crit | tail -10

# В реальном времени
sudo journalctl -f | grep -E "MYSSH|SSH_ATTEMPT"
```

---

## 11. Практические сценарии

### Сценарий 1: Базовый веб-сервер

```bash
# Задача: Веб-сервер (HTTP/HTTPS) + SSH для администрирования
# Требование: Минимальный открытый порты, все остальное - DROP

# 1. Установить target DROP
sudo firewall-cmd --set-target=DROP --permanent

# 2. Разрешить нужные сервисы
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-service=https --permanent
sudo firewall-cmd --add-service=ssh --permanent

# 3. Ограничить SSH админскими IP
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="10.0.0.0/8" service name="ssh" accept' --permanent

# 4. Разрешить ping для мониторинга
sudo firewall-cmd --add-rich-rule='rule protocol value="icmp" accept' --permanent

sudo firewall-cmd --reload
```

### Сценарий 2: VPN-сервер

```bash
# Задача: OpenVPN сервер на порту 1194/udp
# Внутренняя сеть получает доступ через masquerade

# 1. Открыть порт OpenVPN
sudo firewall-cmd --add-port=1194/udp --permanent

# 2. Включить маскарадинг
sudo firewall-cmd --add-masquerade --permanent

# 3. Разрешить форвардинг для VPN клиентов
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="10.8.0.0/24" masquerade' --permanent

sudo firewall-cmd --reload
```

### Сценарий 3: DB-сервер с ограничением

```bash
# Задача: PostgreSQL на порту 5432
# Доступ только с IP приложений
# Логировать все подключения

# 1. Разрешить PostgreSQL для app серверов
sudo firewall-cmd --zone=postgresql --new-zone-from-file=/lib/firewalld/services/postgresql.xml --permanent
sudo firewall-cmd --reload

# 2. Добавить источники
sudo firewall-cmd --zone=postgresql --add-source=10.0.1.10 --permanent
sudo firewall-cmd --zone=postgresql --add-source=10.0.1.11 --permanent

# 3. Добавить логирование
sudo firewall-cmd --zone=postgresql --add-rich-rule='rule service name="postgresql" log prefix="DB_ACCESS" level="info" accept' --permanent

sudo firewall-cmd --reload
```
