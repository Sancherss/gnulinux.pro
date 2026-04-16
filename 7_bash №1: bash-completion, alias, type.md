#  bash №1: bash-completion, alias, type

## Вопросы

### Вопрос 1: Tab дополняет файлы вместо опций команды

**Ответ:** это нормальное поведение для **базового** автодополнения Bash.

**Проблема:** базовый Bash дополняет только имена файлов и директорий, не знает про опции команд.

**Решение:** установить пакет `bash-completion`

```bash
sudo dnf -y install bash-completion   # CentOS/RHEL/Fedora
sudo apt install bash-completion      # Ubuntu/Debian
```

**Что даёт `bash-completion`:**

| Без пакета | С пакетом |
|------------|-----------|
| `ls --[Tab]` → список файлов | `ls --[Tab]` → `--all --color --help` |
| `systemctl [Tab]` → список файлов | `systemctl [Tab]` → список служб |
| `git [Tab]` → список файлов | `git [Tab]` → `commit push pull status` |

---

### Вопрос 2: Вы запускаете команду ls file, но ничего не происходит. У вас подозрения, что кто-то подменил команду ls. Как это проверить?

**Способы проверки:**

```bash
alias | grep ls        # 1. Посмотреть алиасы
type ls                # 2. Узнать тип команды
type -a ls             # 3. Показать все варианты
which ls               # 4. Найти путь к файлу
whereis ls             # 5. Все места
```

**Что может быть не так:**

| Результат | Что означает |
|-----------|--------------|
| `alias ls=''` | пустой алиас (ничего не делает) |
| `alias ls='echo'` | алиас на echo (игнорирует аргументы) |
| `ls is /some/fake/ls` | подмена через PATH |
| `ls is a function` | функция-обманка |

**Как восстановить:**

```bash
unalias ls                     # удалить алиас
/usr/bin/ls file               # использовать оригинал
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin  # очистить PATH
```


---

### Вопрос 3: Вы открыли файл ~/.bashrc, добавили в нём алиас, сохранили и закрыли текстовой редактор. Но алиас не работает. В чём может быть причина?

**Ситуация:**
1. Открыл `~/.bashrc`
2. Добавил алиас
3. Сохранил и закрыл
4. Алиас **не работает**

**Причина:** `~/.bashrc` загружается **только при запуске новой Bash-сессии**.

**Как исправить:**

```bash
source ~/.bashrc   # или . ~/.bashrc
```

**Проверка:**

```bash
cat ~/.bashrc | grep alias   # проверить, что запись в файле
source ~/.bashrc             # применить изменения
alias                        # проверить, что алиас появился
```

---

## Практические задания

### Задание 1: Алиас для сохранения списка алиасов

```bash
alias save="alias > ~/aliases.txt"
```

**Использование:**

```bash
save                    # сохраняет текущие алиасы в файл
cat ~/aliases.txt       # посмотреть сохранённый список
```

**Проверка:**

```bash
sancher@centos8:~$ alias save="alias > ~/aliases.txt"
sancher@centos8:~$ save
sancher@centos8:~$ cat aliases.txt
alias egrep='grep -E --color=auto'
alias fgrep='grep -F --color=auto'
alias grep='grep --color=auto'
...
```
---

### Задание 2: Два алиаса с одинаковым именем

**Эксперимент в `~/.bashrc`:**

```bash
alias show="grep user /etc/passwd"
alias show="cat ~/aliases.txt"
```

**Результат:** работает **второй** (последний).

**Почему:** Bash читает файл `~/.bashrc` сверху вниз. Последнее определение перезаписывает предыдущие.

**Проверка:**

```bash
sancher@centos8:~$ alias show
alias show='cat ~/aliases.txt'
```
---

### Задание 3: Алиас getlog

**Условие:** показывать последние 5 строк `/var/log/kdump.log` и одновременно добавлять вывод в файл `mykernellog`.

**Команда:**

```bash
alias getlog="tail -5 /var/log/kdump.log | tee -a ~/mykernellog"
```

**Адаптация под реальную систему (нет kdump.log):**

```bash
alias getlog='tail -5 /var/log/dnf.log | tee -a ~/mykernellog'
```

**Что делает `-a` в `tee`:**

| Без `-a` | С `-a` |
|----------|--------|
| `tee file` | `tee -a file` |
| перезаписывает файл | **добавляет** в конец |

**Использование:**

```bash
getlog                    # выводит 5 строк на экран и сохраняет в mykernellog
getlog                    # при повторном вызове — добавит ещё 5 строк
cat ~/mykernellog         # видно все 10 строк
```

---


### Основные команды

| Команда | Что делает |
|---------|-------------|
| `alias` | показать все алиасы |
| `alias имя="команда"` | создать алиас |
| `unalias имя` | удалить алиас |
| `type имя` | узнать тип команды |
| `type -a имя` | показать все варианты |

### Постоянные алиасы (в `~/.bashrc`)

```bash
# Примеры полезных алиасов
alias ll='ls -la --color=auto'
alias la='ls -A --color=auto'
alias l='ls -CF --color=auto'
alias update='sudo dnf update -y'
alias myip='curl ifconfig.me'
```

### Применить изменения после редактирования `~/.bashrc`

```bash
source ~/.bashrc
```

---

## Итог

- Понял разницу между базовым и продвинутым автодополнением (`bash-completion`)
- Научился проверять подмену команд через `type`, `which`, `alias`
- Узнал, почему после редактирования `~/.bashrc` нужно делать `source`
- Создал алиасы: `save` (сохранить список алиасов), `getlog` (хвост лога + tee)
- Разобрался, что при одинаковых именах алиасов работает последний
