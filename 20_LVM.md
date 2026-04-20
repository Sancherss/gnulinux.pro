# LVM (Logical Volume Manager)

## Вопросы и ответы

### Вопрос 1: Зачем нужен LVM?

**Ответ:** LVM (Logical Volume Manager) нужен для гибкого управления дисковым пространством. Он позволяет объединять несколько дисков в один пул и создавать логические тома произвольного размера.

#### Основные преимущества LVM

| Возможность | Что даёт | Без LVM |
|-------------|----------|---------|
| **Объединение дисков** | несколько дисков → один пул пространства | каждый диск отдельно |
| **Изменение размера** | можно увеличивать/уменьшать том на лету | нужно пересоздавать раздел |
| **Снапшоты** | мгновенный снимок состояния | нет |
| **Гибкость** | можно перемещать данные между дисками | нет |

#### Структура LVM

```
Физические диски → Физические тома (PV) → Группа томов (VG) → Логические тома (LV) → Файловая система

Пример:
/dev/sda1 ─┐
/dev/sdb1 ─┼→ VG (20GB) → LV1 (10GB) → ext4 /data
/dev/sdc  ─┘             → LV2 (10GB) → xfs /backup
```

---

### Вопрос 2: Опишите весь процесс от добавления диска до настройки fstab с LVM.

**Ответ:** процесс состоит из следующих шагов:

| Шаг | Действие | Команда |
|-----|----------|---------|
| 1 | Добавить диск в систему | (через VirtualBox/VMware) |
| 2 | Создать физический том (PV) | `sudo pvcreate /dev/sdb` |
| 3 | Создать группу томов (VG) | `sudo vgcreate vg_main /dev/sdb` |
| 4 | Создать логический том (LV) | `sudo lvcreate -L 5G -n lv_data vg_main` |
| 5 | Создать файловую систему | `sudo mkfs.ext4 /dev/vg_main/lv_data` |
| 6 | Создать точку монтирования | `sudo mkdir /data` |
| 7 | Примонтировать вручную | `sudo mount /dev/vg_main/lv_data /data` |
| 8 | Добавить в fstab | `echo "/dev/vg_main/lv_data /data ext4 defaults 0 0" \| sudo tee -a /etc/fstab` |
| 9 | Проверить fstab | `sudo mount -a` |

#### Визуализация

```
Добавлен диск /dev/sdb
       ↓
pvcreate /dev/sdb      →  Physical Volume (PV)
       ↓
vgcreate vg_main /dev/sdb → Volume Group (VG)
       ↓
lvcreate -n lv_data -L 5G vg_main → Logical Volume (LV)
       ↓
mkfs.ext4 /dev/vg_main/lv_data → Файловая система
       ↓
mount /dev/vg_main/lv_data /data → Монтирование
       ↓
fstab → Автомонтирование при загрузке
```

> Всегда проверяй fstab командой `sudo mount -a` после редактирования.

---

### Вопрос 3: На ФС с LV заканчивается место. Добавили новый диск. Как увеличить ФС?

**Ответ:** нужно добавить новый диск, расширить VG, затем LV и файловую систему.

| Шаг | Действие | Команда |
|-----|----------|---------|
| 1 | Добавить новый диск | (через VirtualBox/VMware) |
| 2 | Создать PV | `sudo pvcreate /dev/sdc` |
| 3 | Расширить VG | `sudo vgextend vg_main /dev/sdc` |
| 4 | Расширить LV | `sudo lvextend -L +10G /dev/vg_main/lv_data` |
| 5 | Расширить ФС (ext4) | `sudo resize2fs /dev/vg_main/lv_data` |
| 5 | Расширить ФС (xfs) | `sudo xfs_growfs /mount/point` |

#### Визуализация

```
Было: VG (10GB) → LV (10GB) → ФС (10GB) → 100% занято

Добавили диск /dev/sdc (10GB):
pvcreate /dev/sdc → vgextend vg_main /dev/sdc

Стало: VG (20GB) → LV (10GB) → нужно расширить
           ↓
lvextend -L +10G /dev/vg_main/lv_data

Стало: VG (20GB) → LV (20GB) → ФС (10GB) → нужно расширить ФС
           ↓
resize2fs /dev/vg_main/lv_data

Итог: VG (20GB) → LV (20GB) → ФС (20GB) 
```

>  Главное преимущество LVM — можно расширить ФС без остановки работы.

---

### Вопрос 4: Как посмотреть информацию о PV, VG, LV?

**Ответ:** с помощью команд `pvs`/`pvdisplay`, `vgs`/`vgdisplay`, `lvs`/`lvdisplay`.

| Уровень | Краткая информация | Детальная информация |
|---------|---------------------|----------------------|
| **PV** | `pvs` | `pvdisplay` |
| **VG** | `vgs` | `vgdisplay` |
| **LV** | `lvs` | `lvdisplay` |

#### Примеры вывода

```bash
# Краткая информация
pvs
PV         VG      Fmt  Attr PSize   PFree
/dev/sdb1  vg_main lvm2 a--  100.00g 20.00g

vgs
VG      #PV #LV #SN Attr   VSize   VFree
vg_main   2   2   0 wz--n- 200.00g 50.00g

lvs
LV     VG      Attr       LSize
lv_data vg_main -wi-ao---- 100.00g
lv_backup vg_main -wi-ao---- 50.00g
```

> `pvs`, `vgs`, `lvs` — для быстрой проверки. `pvdisplay`, `vgdisplay`, `lvdisplay` — когда нужны детали.

---

### Вопрос 5: Опишите принцип работы Copy-on-Write (COW).

**Ответ:** Copy-on-Write — это механизм, при котором при изменении данных в оригинальном томе, **старые данные сначала копируются в снапшот**, и только потом новые данные записываются в оригинал.

#### Как работает COW

```
1. Создаётся снапшот
   Оригинал: [A][B][C][D]
   Снапшот:  (пустой, ссылается на оригинал)

2. Изменяем блок B на B'
   - Старый B копируется в снапшот
   - Новый B' записывается в оригинал

   Оригинал: [A][B'][C][D]
   Снапшот:  [B]

3. Изменяем блок D на D'
   - Старый D копируется в снапшот
   - Новый D' записывается в оригинал

   Оригинал: [A][B'][C][D']
   Снапшот:  [B][D]
```

#### Что это даёт

| Преимущество | Объяснение |
|--------------|------------|
| Мгновенное создание снапшота | не нужно копировать все данные |
| Экономия места | хранятся только изменения |
| Быстрое восстановление | достаточно скопировать изменённые блоки обратно |

>  COW позволяет создавать мгновенные снапшоты, экономя место и время.

---

### Вопрос 6: Как создать снапшот, восстановить или удалить?

**Ответ:** с помощью команд `lvcreate -s` (создать), `lvconvert --merge` (восстановить), `lvremove` (удалить).

| Действие | Команда | Пример |
|----------|---------|--------|
| **Создать снапшот** | `lvcreate -s -n snap_name -L size /dev/vg/lv` | `lvcreate -s -n mysnap -L 1G /dev/vg_main/lv_data` |
| **Восстановить** | `lvconvert --merge /dev/vg/snap_name` | `lvconvert --merge /dev/vg_main/mysnap` |
| **Удалить** | `lvremove /dev/vg/snap_name` | `lvremove /dev/vg_main/mysnap` |

#### Пример работы со снапшотом

```bash
# Создать снапшот
sudo lvcreate -s -n mysnapshot -L 1G /dev/vg_main/lv_data

# Восстановить
sudo umount /mnt/data
sudo lvconvert --merge /dev/vg_main/mysnapshot
sudo mount /mnt/data

# Удалить снапшот
sudo lvremove /dev/vg_main/mysnapshot
```

>  Снапшот нужно создавать с запасом (20-30% от оригинала).

---

### Вопрос 7: Почему не стоит сразу выделять всё пространство под LV?

**Ответ:** нужно оставлять свободное место в группе томов (VG) для снапшотов, расширения томов и других операций LVM.

#### Основные причины

| Причина | Объяснение |
|---------|------------|
| **Снапшоты** | для снапшотов нужно свободное место в VG |
| **Расширение томов** | можно быстро увеличить LV без добавления нового диска |
| **Гибкость** | можно создать новый LV, если появится потребность |

#### Рекомендации

| Сценарий | Рекомендуемый запас в VG |
|----------|-------------------------|
| Тестовая среда | 10-20% |
| Боевой сервер со снапшотами | 20-30% |
| Системы с частыми изменениями | 30-50% |

---

## Задание 1: Создание LVM из двух разделов и целого диска

### Условие
Создать LVM из двух разделов и одного целого диска. Внутри VG создать два LV по 30% пространства. Настроить автомонтирование в `/data` и `/backup`.

### Выполнение

```bash
# 1. Создание PV
sudo pvcreate /dev/sde1 /dev/sde2 /dev/sdd

# 2. Создание VG
sudo vgcreate myvg /dev/sde1 /dev/sde2 /dev/sdd

# 3. Создание LV (по 30% от VG)
sudo lvcreate -l 30%VG -n lv1 myvg
sudo lvcreate -l 30%VG -n lv2 myvg

# 4. Создание ФС
sudo mkfs.ext4 /dev/myvg/lv1
sudo mkfs.ext4 /dev/myvg/lv2

# 5. Монтирование
sudo mkdir -p /data /backup
sudo mount /dev/myvg/lv1 /data
sudo mount /dev/myvg/lv2 /backup

# 6. Добавление в fstab
echo "/dev/myvg/lv1 /data ext4 defaults 0 0" | sudo tee -a /etc/fstab
echo "/dev/myvg/lv2 /backup ext4 defaults 0 0" | sudo tee -a /etc/fstab
```

### Результат

```bash
df -h /data /backup
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/myvg-lv1  347M   14K  324M   1% /data
/dev/mapper/myvg-lv2  347M   14K  324M   1% /backup
```

---

## Задание 2: Комплексная работа с LVM

### Условие
Добавить 3 диска. На первом — MBR + 2 раздела, на втором — GPT + 1 раздел, третий без разделов. Выполнить серию операций с LVM.

### Выполнение

#### Шаг 1: Подготовка дисков

```bash
# Диск sde (MBR, 2 раздела)
sudo fdisk /dev/sde
# o → n → p → 1 → +500M → n → p → 2 → +500M → w

# Диск sdf (GPT, 1 раздел)
sudo fdisk /dev/sdf
# g → n → 1 → Enter → Enter → w

# Диск sdg — без разделов
```

#### Шаг 2: Создание VG и LV

```bash
sudo pvcreate /dev/sde2
sudo vgcreate vg2 /dev/sde2
sudo lvcreate -L 300M -n lv_test vg2
sudo mkfs.ext4 /dev/vg2/lv_test
sudo mount /dev/vg2/lv_test /mnt
echo "/dev/vg2/lv_test /mnt ext4 defaults 0 0" | sudo tee -a /etc/fstab
```

#### Шаг 3: Расширение VG и снапшоты

```bash
# Расширение VG через sdf1
sudo pvcreate /dev/sdf1
sudo vgextend vg2 /dev/sdf1

# Создание снапшота
sudo lvcreate -s -L 100M -n lv_test_snap /dev/vg2/lv_test

# Создание файлов и восстановление
echo "After snapshot" | sudo tee /mnt/after.txt
sudo umount /mnt
sudo lvconvert --merge /dev/vg2/lv_test_snap
sudo mount /mnt
# Файл after.txt исчез

# Расширение VG через sdg
sudo pvcreate /dev/sdg
sudo vgextend vg2 /dev/sdg

# Второй снапшот
sudo lvcreate -s -L 100M -n lv_test_snap2 /dev/vg2/lv_test
echo "After second snapshot" | sudo tee /mnt/after2.txt

# Удаление снапшота
sudo lvremove /dev/vg2/lv_test_snap2
```

### Финальное состояние

```bash
sudo pvs
PV         VG  PSize
/dev/sde1  vg2 496M
/dev/sdf1  vg2 1.25G
/dev/sdg   vg2 1.25G

sudo vgs
VG  #PV #LV VSize  VFree
vg2   3   1 2.98G 2.68G

sudo lvs
LV      VG  Attr       LSize
lv_test vg2 -wi-ao---- 300M

df -h /mnt
/dev/mapper/vg2-lv_test  272M  17K  253M   1% /mnt
```


---

### Физические тома (PV)

| Команда | Что делает |
|---------|------------|
| `pvcreate /dev/sdb1` | создать PV |
| `pvs` | список PV (кратко) |
| `pvdisplay` | детальная информация о PV |
| `pvremove /dev/sdb1` | удалить PV |

### Группы томов (VG)

| Команда | Что делает |
|---------|------------|
| `vgcreate vg_name /dev/sdb1` | создать VG |
| `vgs` | список VG (кратко) |
| `vgdisplay` | детальная информация о VG |
| `vgextend vg_name /dev/sdc1` | расширить VG |
| `vgreduce vg_name /dev/sdc1` | уменьшить VG |
| `vgremove vg_name` | удалить VG |

### Логические тома (LV)

| Команда | Что делает |
|---------|------------|
| `lvcreate -L 10G -n lv_name vg_name` | создать LV (фиксированный размер) |
| `lvcreate -l 30%VG -n lv_name vg_name` | создать LV (процент от VG) |
| `lvs` | список LV (кратко) |
| `lvdisplay` | детальная информация о LV |
| `lvextend -L +5G /dev/vg/lv` | расширить LV |
| `lvreduce -L -5G /dev/vg/lv` | уменьшить LV |
| `lvremove /dev/vg/lv` | удалить LV |

### Снапшоты

| Команда | Что делает |
|---------|------------|
| `lvcreate -s -L 1G -n snap /dev/vg/lv` | создать снапшот |
| `lvconvert --merge /dev/vg/snap` | восстановить из снапшота |
| `lvremove /dev/vg/snap` | удалить снапшот |

### Расширение файловой системы

| ФС | Команда |
|----|---------|
| ext4 | `resize2fs /dev/vg/lv` |
| xfs | `xfs_growfs /mount/point` |

---

## Итог

- Понял, зачем нужен LVM (гибкость, объединение дисков, снапшоты)
- Освоил полный процесс от добавления диска до fstab
- Научился увеличивать ФС при нехватке места
- Освоил просмотр информации о PV, VG, LV
- Понял принцип Copy-on-Write и работу со снапшотами
- Узнал, почему нужно оставлять место в VG
- Выполнил два практических задания по LVM
