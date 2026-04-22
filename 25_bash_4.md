## Полный отчёт по теме: Bash-скрипты №4 (for, IFS, функции, меню)

## Вопросы и ответы

### Вопрос 1: Что можно задать в качестве списка значений для for?

**Мой ответ:** список слов, файл, вывод команды, диапазон чисел

**Правильный ответ:**

| Способ | Пример | Что делает |
|--------|--------|------------|
| **Список слов** | `for i in 1 2 3 4 5` | перебирает указанные слова |
| **Диапазон чисел** | `for i in {1..10}` | перебирает числа от 1 до 10 |
| **Диапазон с шагом** | `for i in {1..10..2}` | перебирает числа 1,3,5,7,9 |
| **Вывод команды** | `for i in $(ls)` | перебирает вывод команды |
| **Содержимое файла** | `for i in $(cat file.txt)` | перебирает слова из файла |

```bash
# Примеры
for color in red green blue; do echo "Color: $color"; done
for i in {1..5}; do echo "Number: $i"; done
for user in $(cut -d: -f1 /etc/passwd); do echo "User: $user"; done
```
---

### Вопрос 2: Для чего нужен IFS?

**Мой ответ:** IFS задает разделитель в скрипте, то есть мы можем его изменить, но тогда команды, которые применяют разделитель, будут его применять

**Правильный ответ:** IFS (Internal Field Separator) — переменная, которая задаёт разделитель полей в bash. По умолчанию это пробел, табуляция и новая строка.

```bash
# По умолчанию (IFS = пробел, табуляция, новая строка)
data="one two three"
for i in $data; do echo $i; done
# one two three

# Меняем IFS на двоеточие
IFS=':'
data="one:two:three"
for i in $data; do echo $i; done
# one two three

# Чтение /etc/passwd
while IFS=: read -r user pass uid gid comment home shell; do
    echo "User: $user, Home: $home"
done < /etc/passwd
```

**Важно:** всегда сохраняйте и восстанавливайте IFS:

```bash
oldIFS="$IFS"
IFS=':'
# ... код ...
IFS="$oldIFS"
```
---

### Вопрос 3: Для чего нужны функции?

**Мой ответ:** Функции нужны чтобы не писать один и тот же код несколько раз. Мы просто пишем функцию, а потом вызываем её в необходимых местах. Очень удобная вещь в плане программирования и написания скриптов, так как мы избавляем себя от лишнего кода.

**Правильный ответ:**

```bash
# Синтаксис
function имя_функции() {
    команды
}

# Пример
create_user() {
    useradd -m -s /bin/bash "$1"
    echo "Пользователь $1 создан"
}

create_user john
create_user mike
```

**Преимущества функций:**

| Преимущество | Объяснение |
|--------------|------------|
| Переиспользование | код пишется один раз, используется много раз |
| Упрощение | основной код становится короче и понятнее |
| Лёгкость изменений | правишь в одном месте — меняется везде |
| Модульность | скрипт разбивается на логические блоки |

---

## Практические задания

### Задание 1: Вывод списка пользователей с for

**Условие:** создать файл со списком пользователей, с помощью `for` вывести текст «Creating new user: useradd имя_пользователя».

**Мой скрипт:**

```bash
#!/usr/bin/env bash
file=/home/sancher/users
for line in $(cat $file)
do
    echo "Creating new user: useradd $line"
done
```

**Результат:**
```
Creating new user: useradd user1
Creating new user: useradd user2
Creating new user: useradd user3
Creating new user: useradd user4
Creating new user: useradd user5
```

**Что использовал:** `for`, `cat`, переменные


---

### Задание 2: Функции и меню (select + case)

**Условие:** создать функции просмотра, создания и удаления пользователя. Использовать `select` и `case` для меню.

**Мой скрипт:**

```bash
#!/usr/bin/env bash

show_user() {
    read -p "Введите логин: " user
    if id "$user" &>/dev/null; then
        echo "Информация о пользователе $user:"
        echo "  UID: $(id -u "$user")"
        echo "  Домашняя директория: $(eval echo ~"$user")"
        echo "  Группы: $(id -Gn "$user")"
    else
        echo "Пользователь $user не найден"
    fi
}

add_user() {
    read -p "Введите логин: " user
    if id "$user" &>/dev/null; then
        echo "Пользователь $user уже существует"
    else
        useradd -m -s /bin/bash "$user"
        passwd "$user"
        echo "Пользователь $user создан"
    fi
}

delete_user() {
    read -p "Введите логин: " user
    if id "$user" &>/dev/null; then
        userdel -r "$user"
        echo "Пользователь $user удалён"
    else
        echo "Пользователь $user не найден"
    fi
}

select var in "Show information" "Add user" "Delete user" "Exit"; do
    case $var in
        "Show information") show_user ;;
        "Add user") add_user ;;
        "Delete user") delete_user ;;
        "Exit") break ;;
        *) echo "Неправильный выбор" ;;
    esac
done
```

---

### Задание 3: Запуск newbackup от backup

**Условие:** разрешить пользователю `helpdesk` запускать команду `newbackup` от имени пользователя `backup`. Команда создаёт сжатый архив `/data` с датой в названии.

**Выполнение:**

#### 1. Создание скрипта `newbackup`

```bash
#!/usr/bin/env bash
date=$(date +%Y%m%d_%H%M%S)
backup_dir="/home/backup"
mkdir -p "$backup_dir"
tar -czf "$backup_dir/backup_$date.tar.gz" /data 2>/dev/null
if [ $? -eq 0 ]; then
    echo "Архив создан: $backup_dir/backup_$date.tar.gz"
else
    echo "Ошибка создания архива" >&2
    exit 1
fi
```

#### 2. Настройка прав

```bash
sudo cp newbackup /usr/local/bin/
sudo chmod 755 /usr/local/bin/newbackup
```

#### 3. Настройка sudoers

```bash
echo "helpdesk ALL=(backup) /usr/local/bin/newbackup" | sudo tee /etc/sudoers.d/helpdesk
sudo chmod 440 /etc/sudoers.d/helpdesk
```

#### 4. Проверка

```bash
sudo -u backup /usr/local/bin/newbackup
# Архив создан: /home/backup/backup_20260422_225502.tar.gz
```

---

### Цикл for

| Конструкция | Описание |
|-------------|----------|
| `for i in список; do команды; done` | базовый синтаксис |
| `for i in {1..10}; do ...; done` | диапазон чисел |
| `for i in $(ls); do ...; done` | вывод команды |
| `for ((i=0; i<10; i++)); do ...; done` | C-стиль |

### IFS (Internal Field Separator)

| Значение | Разделители |
|----------|-------------|
| `IFS=' '` (по умолчанию) | пробел, табуляция, новая строка |
| `IFS=$'\n'` | только новая строка |
| `IFS=':'` | двоеточие |

```bash
# Сохранить и восстановить
oldIFS="$IFS"
IFS=$'\n'
# ... код ...
IFS="$oldIFS"
```

### Функции

| Конструкция | Описание |
|-------------|----------|
| `function name() { commands; }` | объявление функции |
| `name` | вызов функции |
| `$1, $2, ...` | параметры функции |
| `return 0` | возврат кода выхода |

### Select и Case

| Конструкция | Описание |
|-------------|----------|
| `select var in list; do ...; done` | создание меню |
| `case $var in pattern) ...;; esac` | выбор по шаблону |

### Tar для бэкапов

| Ключ | Описание |
|------|----------|
| `-c` | создать архив |
| `-z` | сжать gzip |
| `-f` | указать имя файла |
| `-x` | распаковать |

```bash
# Создание архива
tar -czf backup.tar.gz /data

# Просмотр содержимого
tar -tzf backup.tar.gz

# Распаковка
tar -xzf backup.tar.gz
```

### Формат даты

| Символ | Значение | Пример |
|--------|----------|--------|
| `%Y` | год | 2026 |
| `%m` | месяц | 04 |
| `%d` | день | 22 |
| `%H` | час | 22 |
| `%M` | минуты | 55 |
| `%S` | секунды | 02 |

```bash
date=$(date +%Y%m%d_%H%M%S)   # 20260422_225502
```

---

**Отчёт подготовлен на основе практического выполнения заданий по курсу GNU Linux Pro.**
