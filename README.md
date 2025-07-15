# mikrotik-torrserver-MatriX

```bash
# === Подготовка образа на MacBook ===
# Устанавливаем skopeo и сохраняем multi-arch TorrServer в локальный TAR
brew install skopeo   # установка skopeo через Homebrew
skopeo copy \
  --override-os linux \
  --override-arch arm64 \
  --override-variant v8 \
  docker://ghcr.io/yourok/torrserver:latest \
  docker-archive:torrserver-arm64.tar:yourok/torrserver:latest  # сохраняем образ с правильным именем и тегом

# === Включение режима контейнеров на MikroTik ===
# Включаем поддержку контейнеров и выполняем перезагрузку
# /system device-mode update container=yes
# /system reboot

# === Подготовка USB-хранилища ===
# Проверяем список дисков и форматируем usb1 в ext4 для надёжности
/disk print
/disk format usb1 filesystem=ext4

#Функция mkdir для удобного создания папок/Функция mkdir сохраняется в scripts
{
    :local result [/tool fetch \
    url="https://raw.githubusercontent.com/phistrom/routeros-mkdir/master/persist_create_mkdir_function.rsc" \
    as-value output=user];
    :local script [:parse ($result->"data")]
    $script;
}

# Создание директорий на USB (используется внешний скрипт $mkdir)
$mkdir usb1/opt/ts           # базовая папка для данных TorrServer
$mkdir usb1/opt/ts/config    # папка для конфигурационных файлов
$mkdir usb1/opt/ts/log       # папка для логов сервера
$mkdir usb1/opt/ts/torrents  # папка для загруженных торрентов
$mkdir usb1/tmp              # временная папка для распаковки слоёв образов

# === Сетевая настройка для контейнеров ===
# Создаём пару VETH, мост и настраиваем NAT, чтобы контейнеры имели доступ к интернету
/interface veth add name=veth1 address=172.17.0.2/24 gateway=172.17.0.1  # виртуальный интерфейс torrserver-MatriX и его ip
/interface bridge add name=containers                                                     # создаём мост
/interface bridge port add bridge=containers interface=veth1                              # привязываем veth к мосту
/ip address add address=172.17.0.1/24 interface=containers                              # даём IP мосту
/ip firewall nat add chain=srcnat action=masquerade src-address=172.17.0.0/24            # NAT для исходящих контейнеров

# === Конфигурация Container package ===
# Указываем GHCR как реестр и задаём tmpdir для распаковки образов
/container config set registry-url=https://ghcr.io tmpdir=usb1/tmp

# === Настройка группы переменных окружения 'tsM' ===
# Будет использоваться при создании контейнера для передачи путей и порта
/container envs add list=tsM key=TS_CONF_PATH value=/opt/ts/config   # путь к конфигурации
/container envs add list=tsM key=TS_LOG_PATH value=/opt/ts/log       # путь к логам
/container envs add list=tsM key=TS_TORR_DIR value=/opt/ts/torrents  # папка с торрентами
/container envs add list=tsM key=TS_PORT value=8090                  # HTTP-порт сервера

# === Создание группы монтирований 'tsM' ===
# Связываем локальные папки USB с путями внутри контейнера\ n/container mount add comment=tsM name=cfg src=usb1/opt/ts/config dst=/opt/ts/config
/container mount add comment=tsM name=log src=usb1/opt/ts/log dst=/opt/ts/log
/container mount add comment=tsM name=torr src=usb1/opt/ts/torrents dst=/opt/ts/torrents

# === Добавление и запуск контейнера TorrServer ===
# Используем локальный TAR, группу монтирований и переменных окружения
/container add name=TorrServer-MatriX \
    file=usb1/torrserver-arm64.tar \            # локальный образ из TAR
    interface=veth1 \                            # подключение к виртуальной сети
    root-dir=usb1/containers/ts-MatriX \         # рабочая директория контейнера
    mounts=cfg,log,torr \                        # алиасы для монтированных папок
    envlists=tsM \                               # алиас для группы переменных окружения
    start-on-boot=yes                             # автозапуск при перезагрузке

# === Запуск контейнера вручную (если необходимо) ===
/container start TorrServer-MatriX
