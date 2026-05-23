## Шпаргалка: ограничение CPU через systemd slices (cgroups)

> **Цель:** гарантировать, что нагрузка на процессор никогда не превысит заданный порог (например, 50% от всех ядер).  
> **Работает:** на любом Linux с systemd (Ubuntu 20.04+, Debian 10+, CentOS 8+ и т.п.).  
> **Подход:** создаём общий слайс с квотой CPU и помещаем в него все «тяжёлые» сервисы (Docker, пользовательские процессы).

---

### 1. Теория: как считать квоту

- `CPUQuota` в systemd задаётся **в процентах от одного ядра**.
- Чтобы ограничить общую загрузку **всей машины** до X%, ставьте `CPUQuota = X% * количество_ядер`.
- Примеры:
  - **1 ядро**, цель 50% → `CPUQuota=50%`
  - **2 ядра**, цель 50% → `CPUQuota=100%` (потому что 100% одного ядра = 50% от двух)
  - **4 ядра**, цель 30% → `CPUQuota=120%`

Узнать число ядер: `nproc`

---

### 2. Создаём общий слайс `limited.slice`

Создайте файл `/etc/systemd/system/limited.slice`:

```bash
cat <<EOF > /etc/systemd/system/limited.slice
[Unit]
Description=Ограниченный CPU-слайс (50% от 1 ядра)
[Slice]
CPUQuota=50%
EOF
```
Примените изменения:
```bash
systemctl daemon-reload
systemctl start limited.slice
```
Проверка:
```bash
systemctl show limited.slice | grep CPUQuota
```

### 3. Помещаем сервисы в слайс
#### 3.1. Ограничение Docker (демон + все контейнеры)
Шаг 1. Настройка демона `Docker`
Создайте drop-in для службы docker.service:
```bash
mkdir -p /etc/systemd/system/docker.service.d
cat <<EOF > /etc/systemd/system/docker.service.d/50-slice.conf
[Service]
Slice=limited.slice
EOF
```
