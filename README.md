# debian-server-setup

Учебный стенд для администрирования Debian в VirtualBox. Я подключился к ВМ по SSH из Windows, проверил выход Debian в интернет и настроил доступ к общему каталогу через группу Linux.

При повторении стенда замените `labuser` на имя своей учётной записи Debian в командах и в копии конфигурации SSH.

## Перед началом

1. Установите на хосте программу виртуализации. Для этого стенда я использовал Oracle VirtualBox в Windows 11. [Скачать VirtualBox с сайта Oracle](https://www.oracle.com/virtualization/technologies/vm/downloads/virtualbox-downloads.html).
2. Скачайте установочный ISO-образ Debian для виртуальной машины. [Скачать Debian с официального сайта](https://www.debian.org/distrib/).

## Виртуалка

| Параметр | На стенде |
| --- | --- |
| Хост | Windows 11 |
| Виртуализация | Oracle VirtualBox |
| Гостевая ОС | Debian 13.7 (Trixie) |
| Ресурсы ВМ | 2 vCPU, 5 961 МБ ОЗУ, динамический VDI на 40 ГиБ |

### Выбор программ при установке Debian

Загрузите ВМ с ISO Debian и выберите **Graphical install**. [Запуск установщика Debian](https://www.debian.org/releases/trixie/amd64/ch05s01.en.html).

На экране **Выбор программного обеспечения** (`Software selection`) отметьте два пункта.

- **SSH server** установит OpenSSH Server для подключения из Windows. [Пакет Debian](https://packages.debian.org/trixie/task-ssh-server).
- **standard system utilities** установит стандартные системные утилиты. [Выбор программ в установщике Debian](https://www.debian.org/releases/trixie/amd64/ch06s03.en.html#pkgsel).

Для графического рабочего стола, как на скриншотах, дополнительно выберите **Debian desktop environment** и **GNOME**. На консольной установке для команд `nmcli` из раздела сети понадобится NetworkManager. [Задача GNOME в Debian](https://packages.debian.org/trixie/task-gnome-desktop).

На установленной ВМ проверил версию в консоли Debian.

```bash
cat /etc/debian_version
```

Команда вернула `13.7`.

### Применимость к Ubuntu Server

Ubuntu основана на Debian и тоже использует APT. Базовые команды из этого проекта для диагностики сети, подключения по SSH, управления пользователями и правами и настройки UFW пригодятся и на Ubuntu Server. [Связь Ubuntu с Debian](https://ubuntu.com/project/docs/how-ubuntu-is-made/processes/merges-and-syncs/), [UFW в Ubuntu Server](https://ubuntu.com/server/docs/firewalls/).

Сетевые интерфейсы в Ubuntu Server обычно настраивают через Netplan. Команды `nmcli` подходят для интерфейсов, которыми управляет NetworkManager. [Настройка сети в Ubuntu Server](https://ubuntu.com/server/docs/configuring-networks/).

## Сеть VirtualBox

| Адаптер | Интерфейс Debian | IPv4 Debian | Назначение |
| --- | --- | --- | --- |
| 1, NAT | `enp0s3` | `10.0.2.15/24` | Выход в интернет через шлюз `10.0.2.2` |
| 2, Host-only | `enp0s8` | `192.168.56.101/24` | Подключение по SSH из Windows с адреса `192.168.56.1` |

Адреса и имена интерфейсов в командах относятся к этому стенду.

### Первый адаптер NAT

Для повторения в настройках ВМ откройте **Сеть → Адаптер 1**, включите сетевой адаптер, выберите **NAT** и отметьте **Подключить виртуальный кабель**.

На стенде сначала был включён только первый адаптер NAT с подключённым виртуальным кабелем. Его IPv4-адрес проверил в консоли Debian.

```bash
ip -br -4 addr
```

До подключения второго адаптера `enp0s3` был в состоянии `UP` с адресом `10.0.2.15/24`.

### Второй адаптер Host-only

В настройках выключенной ВМ открыл **Сеть → Адаптер 2**, включил сетевой адаптер, выбрал **Host-only** («Виртуальный адаптер»), указал `VirtualBox Host-Only Ethernet Adapter` и отметил **Подключить виртуальный кабель**.

После загрузки Debian проверил, появился ли второй интерфейс.

```bash
ip -br addr
```

Интерфейс `enp0s8` получил `192.168.56.101/24`. У `enp0s3` в этом выводе IPv4-адреса не было.

### Диагностика NAT

После подключения второго адаптера проверил маршруты и профили NetworkManager в Debian.

```bash
ip -4 route
nmcli device status
nmcli -f NAME,TYPE,DEVICE connection show
```

В таблице маршрутов была только сеть `192.168.56.0/24`. NetworkManager показывал для `enp0s3` состояние «отключено». Активное проводное подключение было только на `enp0s8`.

Создал для `enp0s3` отдельный профиль с получением IPv4 по DHCP.

```bash
nmcli connection add type ethernet ifname enp0s3 con-name 'VirtualBox NAT' ipv4.method auto
```

После активации профиля `ip -4 route` показала `default via 10.0.2.2 dev enp0s3` и адрес `10.0.2.15` в этом маршруте. Затем проверил HTTPS-запрос к репозиторию пакетов из ВМ.

```bash
wget -4 --spider -T 10 -t 1 https://deb.debian.org/debian/dists/trixie/InRelease
```

Сервер вернул `200 OK`.

![Маршруты Debian и HTTPS-проверка через NAT](screenshots/nat-routing-https.png)

## SSH из Windows

1. **В консоли Debian в VirtualBox** проверьте службу OpenSSH.

   ```bash
   systemctl status ssh --no-pager
   ```

   На стенде служба была запущена (`active (running)`), а её запуск при загрузке включён (`enabled`).

2. **В терминале Windows** начните подключение через Host-only.

   ```powershell
   ssh labuser@192.168.56.101
   ```

3. **При первом запросе подтверждения ключа** оставьте терминал Windows открытым. На стенде SSH показал отпечаток `SHA256:…` ключа ED25519. В консоли Debian в VirtualBox получите отпечаток открытого ключа сервера.

   ```bash
   ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
   ```

4. Сравните два отпечатка. При совпадении введите `yes` в терминале Windows, затем пароль своей учётной записи Debian. Приглашение вида `labuser@debianlab:~$` подтверждает, что открылась оболочка Debian.

На стенде вход с паролем прошёл, а отпечаток из Windows совпал с отпечатком из консоли ВМ. После восстановления NAT из Windows также проверил доступность `192.168.56.101:22`.

### Межсетевой экран UFW

Для команд в этом разделе нужна учётная запись с правами `sudo`. [Как настроить `sudo` через `visudo`](#права-sudo-через-visudo).

В консоли VirtualBox установил UFW.

```bash
sudo apt install ufw
```

Перед добавлением правила проверил слушающие сокеты командой `sudo ss -lntup`. Процесс `sshd` слушал `22/tcp` на IPv4 и IPv6.

![Слушающие сокеты Debian и порт SSH](screenshots/listening-ports.png)

Добавил правило для подключения из Windows перед включением фильтра.

```bash
sudo ufw allow in on enp0s8 proto tcp from 192.168.56.1 to any port 22
sudo ufw enable
sudo ufw status verbose
```

UFW показал `Status: active`, запрет входящих и разрешение исходящих соединений по умолчанию. Правило `22/tcp on enp0s8` разрешает входящий SSH с IPv4-адреса Windows `192.168.56.1` через Host-only.

![Активный UFW и правило SSH](screenshots/ufw-status.png)

После включения UFW я открыл новый SSH-сеанс из Windows и успешно вошёл в Debian.

## Пользователи, группы и права

Для примера создал группу `admin` и выделил ей общий каталог `/srv/admin`. Команды настройки выполняются под `root`.

```bash
groupadd admin
usermod -aG admin labuser
install -d -o root -g admin -m 2770 /srv/admin
```

Вывод `ls -ld /srv/admin` подтвердил `root:admin` и `drwxrws---`. Права `770` дают владельцу и группе доступ к каталогу. Начальная `2` устанавливает бит `setgid`, поэтому группа `admin` наследуется при создании файлов и подкаталогов.

Проверка от имени участника группы и от имени системной учётной записи `nobody`.

```bash
runuser -u labuser -- touch /srv/admin/access-test.txt
runuser -u nobody -- ls /srv/admin
```

Участник группы создал файл, который получил группу `admin`. При попытке прочитать каталог от имени `nobody` команда `ls` вернула «Отказано в доступе».

![Владелец, группа и права каталога /srv/admin](screenshots/directory-mode.png)

### Права `sudo` через `visudo`

Для существующей учётной записи Debian я настроил право выполнять административные команды через `sudo`. Для изменения правил `sudoers` использовал `visudo`. Он проверяет синтаксис при сохранении.

При первом назначении прав откройте редактор из оболочки `root`.

```bash
visudo
```

Пример правила, которое даёт пользователю `labuser` право запускать любые команды через `sudo`.

```sudoers
labuser ALL=(ALL:ALL) ALL
```

Из учётной записи с действующим `sudo` редактор открывается командой `sudo visudo`. После сохранения проверьте доступ из обычного сеанса пользователя.

```bash
sudo whoami
```

Ожидаемый вывод — `root`.

## Клавиши в терминале Debian

В Bash набери начало команды или пути и нажми `Tab`. Например, `systemct` + `Tab` дополнит команду до `systemctl`. Если совпадений несколько, нажми `Tab` ещё раз, чтобы увидеть варианты.

| Программа | Сохранение и выход |
| --- | --- |
| Nano | `Ctrl+O`, затем `Enter` сохраняют файл. `Ctrl+X` закрывает редактор. |
| Vim | `Esc`, затем `:wq` и `Enter` сохраняют файл и закрывают редактор. |
| `less` | `q` закрывает просмотр. |

Клавиши в `sudo visudo` зависят от открывшегося редактора. В Nano подсказка `^X` означает `Ctrl+X`.

## Примеры конфигурации

- Профиль NetworkManager для `enp0s3` с автоматическим подключением и IPv4 по DHCP. [Файл профиля NAT](configs/virtualbox-nat.nmconnection.example), [справка NetworkManager для Debian](https://manpages.debian.org/trixie/network-manager/nm-settings-keyfile.5.en.html).
- Конфигурация SSH-клиента для подключения через Host-only. Запустите `ssh -F configs/ssh-client.conf debianlab` из корня репозитория. [Файл конфигурации SSH](configs/ssh-client.conf).
