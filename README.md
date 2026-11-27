# TorrServer MatriX.151 на MikroTik hAP ax³

Инструкция по установке ARM64-образа TorrServer на USB-диск, обновлению с сохранением данных и откату. Версия закреплена: `docker.io/yourok/torrserver:MatriX.151`.

## Готовый образ

- [Скачать torrserver-arm64-151.tar](https://github.com/Skaystk/mikrotik-torrserver-MatriX/releases/download/MatriX.151/torrserver-arm64-151.tar)
- [Релиз MatriX.151](https://github.com/Skaystk/mikrotik-torrserver-MatriX/releases/tag/MatriX.151)
- Размер опубликованного TAR: **180 344 832 байта** (около 172 МиБ).
- Формат: Docker archive; платформа: **linux/arm64**. Это образ приложения, без пользовательских настроек и логов.

SHA-256 именно опубликованного файла:

```text
b906cacdac1b47df1511d09f09ff08eaafd376a2a29acd47638e267efb02ce6d
```

Проверка на Windows:

```powershell
Get-FileHash .\torrserver-arm64-151.tar -Algorithm SHA256
```

На macOS:

```bash
shasum -a 256 torrserver-arm64-151.tar
```

Самостоятельно созданный TAR может иметь другую контрольную сумму из-за упаковки и метаданных.

## 1. Требования и обозначения

Нужны RouterOS 7 с пакетом `container` той же версии и архитектуры ARM64, включённый режим контейнеров и USB-накопитель с достаточным свободным местом. Для данных предпочтителен ext4. **Существующий диск при обновлении не форматировать.**

Команды ниже используют современный синтаксис RouterOS: `mountlists`, `envlists`, `/container mounts add list=...`. Если терминал не узнаёт параметр, проверьте справку через `?` и версию RouterOS; не смешивайте эти команды со старым `mounts=` или `envlist=`.

`usb1`, `veth1` и сеть `172.17.0.0/24` — примеры. Проверьте имя диска и отсутствие пересечения адресов со своими сетями. Блоки первичной установки выполняются один раз; при обновлении переходите к разделу 7.

```routeros
/system resource print
/system package print
/system device-mode print
/disk print detail
/container print detail
```

Если контейнеры ещё запрещены:

```routeros
/system device-mode update container=yes
```

Выполните физическое подтверждение по подсказке RouterOS. Обычная программная перезагрузка не заменяет требуемое подтверждение.

## 2. Подготовка TAR самостоятельно

Этот раздел можно пропустить при использовании готового файла из Releases. Сохраняйте образ через `save`/Skopeo, а не через `docker export`: RouterOS нужны также метаданные образа.

### Windows: Docker Desktop

Запустите Docker Desktop в режиме Linux containers. В PowerShell, в папке для TAR:

```powershell
docker pull --platform linux/arm64 docker.io/yourok/torrserver:MatriX.151
docker image inspect docker.io/yourok/torrserver:MatriX.151 --format "{{.Os}}/{{.Architecture}}"
docker image save --output torrserver-arm64-151.tar docker.io/yourok/torrserver:MatriX.151
```

Проверьте результат каждой команды; ожидаемая платформа — `linux/arm64`. Использование `--output` исключает проблемы с перенаправлением бинарных данных в старом PowerShell.

### Windows: Skopeo через Docker

Альтернатива `docker pull/save`; запускается только инструмент Skopeo, сам ARM64-образ запускать не нужно:

```powershell
docker run --rm -v "${PWD}:/out" quay.io/skopeo/stable copy --override-os linux --override-arch arm64 --override-variant v8 --format v2s2 docker://docker.io/yourok/torrserver:MatriX.151 docker-archive:/out/torrserver-arm64-151.tar:docker.io/yourok/torrserver:MatriX.151
```

### macOS: Skopeo

```bash
brew install skopeo
skopeo copy \
  --override-os linux \
  --override-arch arm64 \
  --override-variant v8 \
  --format v2s2 \
  docker://docker.io/yourok/torrserver:MatriX.151 \
  docker-archive:torrserver-arm64-151.tar:docker.io/yourok/torrserver:MatriX.151
```

Загрузите готовый TAR через WinBox → Files в `usb1/torrserver-arm64-151.tar`. Дождитесь окончания передачи.

## 3. Постоянные данные и отдельный каталог импорта

Создайте каталоги через WinBox → Files → New Directory, последовательно создавая родительские папки. Существующие каталоги с данными используйте без очистки.

| На USB-диске | В контейнере | Назначение |
| --- | --- | --- |
| `/usb1/opt/ts/config` | `/opt/ts/config` | Настройки и база |
| `/usb1/opt/ts/log` | `/opt/ts/log` | Файлы журналов |
| `/usb1/opt/ts/torrents` | `/opt/ts/torrents` | Файлы торрентов |
| `/usb1/tmp` | `/tmp` | Постоянный дисковый кэш TorrServer |
| `/usb1/container-import-tmp` | Не монтируется | Временная распаковка образов RouterOS |
| `/usb1/containers` | Не монтируется | Родительский каталог корней контейнеров |

**`tmpdir` импорта и `/usb1/tmp` с кэшем должны быть разными каталогами.** Не используйте рабочий кэш для распаковки или очистки слоёв. Для нового контейнера укажите новый, ещё не занятый `root-dir`.

```routeros
/container config set tmpdir=usb1/container-import-tmp
/container mounts add list=conf src=/usb1/opt/ts/config dst=/opt/ts/config
/container mounts add list=log src=/usb1/opt/ts/log dst=/opt/ts/log
/container mounts add list=torr src=/usb1/opt/ts/torrents dst=/opt/ts/torrents
/container mounts add list=tmp src=/usb1/tmp dst=/tmp
```

До добавления проверьте `/container mounts print detail`: не создавайте дубликаты существующих списков. При обновлении сохраните фактические пути своих данных.

## 4. Переменные окружения

```routeros
/container envs add list=tsPersistent key=TS_CONF_PATH value=/opt/ts/config
/container envs add list=tsPersistent key=TS_LOG_PATH value=/opt/ts/log/torrserver.log
/container envs add list=tsPersistent key=TS_TORR_DIR value=/opt/ts/torrents
/container envs add list=tsPersistent key=TS_PORT value=8090
```

`TS_LOG_PATH` — **путь к файлу**, а не к каталогу. Значение `/opt/ts/log` вызывает ошибку `is a directory`. Перед повторным выполнением проверьте `/container envs print detail` и измените существующую запись вместо создания второго ключа. Во всех новых контейнерах подключайте `envlists=tsPersistent`.

## 5. Сеть при первой установке

Если рабочие bridge/VETH/NAT уже есть, используйте их. Для новой установки пример такой:

```routeros
/interface bridge add name=containers
/interface veth add name=veth1 address=172.17.0.2/24 gateway=172.17.0.1
/interface bridge port add bridge=containers interface=veth1
/ip address add address=172.17.0.1/24 interface=containers
/ip firewall nat add chain=srcnat action=masquerade src-address=172.17.0.0/24 comment="Container outbound NAT"
```

В своей политике firewall разрешите необходимый исходящий трафик контейнера и доступ из доверенной LAN к TCP 8090. Укажите доступные DNS-серверы в настройках RouterOS или контейнера. Не открывайте веб-интерфейс TorrServer в интернет. Наличие NAT не заменяет правила firewall.

## 6. Импорт и первый запуск

```routeros
/container add name=TorrServer-MatriX file=usb1/torrserver-arm64-151.tar interface=veth1 root-dir=usb1/containers/ts-MatriX-151 mountlists=conf,log,torr,tmp envlists=tsPersistent logging=yes start-on-boot=no
/container print detail where name="TorrServer-MatriX"
```

Дождитесь завершения импорта и статуса `STOPPED`; при ошибке не запускайте контейнер, сначала проверьте диагностику. Команда запуска уже находится в образе — вручную переопределять `cmd` не требуется.

```routeros
/container start [find where name="TorrServer-MatriX"]
/container print detail where name="TorrServer-MatriX"
/log print where topics~"container"
```

Откройте `http://172.17.0.2:8090` из разрешённой локальной сети. Проверьте **MatriX.151**, настройки, список торрентов и воспроизведение. В настройках TorrServer включите дисковый кэш; если требуется его сохранение, отключите удаление кэша при отключении торрента. Само монтирование `/tmp` эту настройку не включает.

Для проверки диска откройте оболочку работающего контейнера:

```routeros
/container shell [find where name="TorrServer-MatriX"]
```

Затем внутри Linux-оболочки:

```sh
ls -lah /opt/ts/config /opt/ts/log /opt/ts/torrents
du -sh /tmp
df -h /tmp
exit
```

После успешной проверки включите автозапуск. Изменение параметров работающего контейнера может вызвать перезапуск; выполняйте это вне просмотра:

```routeros
/container stop [find where name="TorrServer-MatriX"]
```

Дождитесь `STOPPED`, затем:

```routeros
/container set [find where name="TorrServer-MatriX"] start-on-boot=yes
/container start [find where name="TorrServer-MatriX"]
```

## 7. Безопасное обновление

1. Сохраните старый TAR и параметры контейнера: имя, `root-dir`, VETH, списки mounts/envs, DNS, лимиты ресурсов. Скачайте резервную копию данных на другой носитель. Экспорт RouterOS не содержит базу и кэш TorrServer.
2. Приостановите просмотр, добавление торрентов и задания очистки. Отключите автозапуск старого контейнера, остановите его и дождитесь `STOPPED`. Сделайте согласованную копию `/usb1/opt/ts/config` и `/usb1/opt/ts/torrents`; для сохранения кэша скопируйте также `/usb1/tmp`. Запомните исходные пути и время копии.
3. Переименуйте остановленный контейнер в `TorrServer-MatriX-old`. Если имя действующего контейнера другое, подставьте его в команды.

```routeros
/container stop [find where name="TorrServer-MatriX"]
```

После `STOPPED`:

```routeros
/container set [find where name="TorrServer-MatriX"] start-on-boot=no
/container set [find where name="TorrServer-MatriX"] name=TorrServer-MatriX-old
```

4. Проверьте списки из разделов 3–4. Если меняете `TS_LOG_PATH`, сохраните прежнее значение для отката. Импортируйте MatriX.151 по разделу 6, с **новым** `root-dir=usb1/containers/ts-MatriX-151`. Если этот путь уже занят, выберите другой пустой путь. Убедитесь, что импорт завершён, и запустите новый контейнер.
5. Проверьте конфигурацию, торренты, доступность дискового кэша и воспроизведение, затем включите автозапуск нового контейнера. Возобновите задания обслуживания после проверки их путей и имён контейнеров.

**Не запускайте старый и новый контейнеры одновременно с общей базой, кэшем или VETH/IP.** Новая версия может изменить формат базы: наличие старого контейнера без резервной копии данных не гарантирует откат. Старый корень, TAR и резервные копии сохраняйте до окончания проверки.

### Откат

Остановите новую версию, дождитесь `STOPPED` и отключите её автозапуск. Переименуйте её, чтобы освободить рабочее имя:

```routeros
/container set [find where name="TorrServer-MatriX"] start-on-boot=no
/container set [find where name="TorrServer-MatriX"] name=TorrServer-MatriX-151-failed
```

При обоих остановленных контейнерах сохраните текущие данные отдельно и восстановите согласованную резервную копию конфигурации и торрентов **до обновления**. Кэш восстановите из той же копии, если требуется его возврат или изменилась его структура. Восстановите старые mounts/envs и исходный `TS_LOG_PATH`, если они менялись. Затем:

```routeros
/container set [find where name="TorrServer-MatriX-old"] name=TorrServer-MatriX
/container start [find where name="TorrServer-MatriX"]
```

Проверьте старую версию и воспроизведение. Автозапуск включайте после проверки по разделу 6. Не удаляйте общие каталоги данных вместе с неудачным контейнером.

## 8. Контроль кэша: 30 дней

Целевая политика — проверять файлы кэша возрастом **30 дней и старше** и удалять только подтверждённо ненужные данные. Постоянный `/tmp` не имеет автоматического срока хранения.

Для просмотра кандидатов войдите в оболочку работающего контейнера, как в разделе 6, и выполните:

```sh
du -sh /tmp
df -h /tmp
find /tmp -type f -mtime +29 -print
```

`-mtime +29` означает не менее 30 полных суток с последнего изменения. Это только список: команда ничего не удаляет. `mtime` не показывает время последнего чтения или активность торрента. Каталог `/tmp` может содержать также временные файлы других функций — сначала определите фактические файлы кэша.

**Предупреждение: старые файлы могут принадлежать активным торрентам, раздаче или текущему просмотру. Не запускайте удаление по возрасту во время работы и не очищайте весь `/usb1/tmp` по расписанию.**

Порядок очистки:

1. Сопоставьте кандидатов с торрентами и закончите активные просмотры/раздачи. При сомнении сохраните файл.
2. По возможности удалите ненужный кэш средствами TorrServer. Для ручной очистки файлов остановите контейнер и дождитесь `STOPPED`.
3. Через WinBox удалите только проверенные файлы кэша. Не трогайте конфигурацию, `.torrent`, каталоги импорта или корни контейнеров. После удаления кэш будет скачиваться заново.
4. Запустите контейнер и проверьте воспроизведение и свободное место.

Можно добавить напоминание раз в 30 дней; оно **не удаляет** файлы. Перед добавлением убедитесь, что задания с таким именем ещё нет:

```routeros
/system scheduler add name=TorrServer-Cache-Review interval=30d on-event={:log warning "TorrServer: review cache aged 30 days; check active torrents before cleanup"}
```

Проверьте `next-run` в `/system scheduler print detail where name="TorrServer-Cache-Review"`. При интенсивном использовании проверяйте свободное место чаще: 30 дней не ограничивают объём кэша. Если есть старое задание `TorrServer-Cache-Cleanup`, прочитайте его содержимое и отключите безусловное удаление до проверки безопасности.

## 9. Диагностика

```routeros
/container print detail
/container mounts print detail
/container envs print detail
/container config print
/disk print detail
/interface veth print detail
/ip dns print
/log print where topics~"container"
```

Эти команды предназначены для локальной проверки. Перед публикацией вывода удаляйте пароли, токены, публичные адреса, имена устройств, пути пользователей, названия торрентов и magnet-ссылки. Не загружайте журналы и резервные копии в публичный репозиторий.

| Симптом | Что проверить |
| --- | --- |
| `open /opt/ts/log: is a directory` | Задать `TS_LOG_PATH=/opt/ts/log/torrserver.log` в `tsPersistent`; подключить `envlists=tsPersistent`, проверить mount `log`, перезапустить контейнер |
| Ошибка `mounts` / `envlist` / `expected end of command` | Версию RouterOS и актуальные `mountlists`, `envlists`, `list=`; не вставлять комментарии после символа продолжения строки |
| Импорт не завершился | Полноту TAR, ARM64, свободное место, отдельный `tmpdir`, новый `root-dir` и журнал RouterOS |
| `exec format error` | Загружен ли именно `linux/arm64`, а не AMD64 |
| Настройки или кэш пропали | Фактический USB-диск, пути `src`/`dst`, все четыре mountlists, envlists и настройку дискового кэша в TorrServer |
| DNS timeout / System DNS check failed | Доступность DNS из контейнерной сети, UDP/TCP 53, шлюз, NAT и firewall; успешный резервный DNS не устраняет причину таймаута основного |
| Веб-интерфейс недоступен | Статус контейнера, VETH/bridge/IP, TCP 8090 и разрешение доступа из LAN |
| Bonjour: no suitable interface addresses | Ограничения multicast/mDNS через контейнерный NAT; отдельно проверить доступ по IP и воспроизведение |
| Мало пиров или низкая скорость | Здоровье раздачи, трекеры, DHT/PEX, TCP/µTP и лимиты; принудительное шифрование может снижать совместимость |
| Высокая нагрузка или остановки | Свободные RAM/диск, скорость USB, число соединений и размер кэша; увеличивать лимиты постепенно под наблюдением |

Добавление трекеров не гарантирует сидов. Для приватных трекеров соблюдайте их правила DHT/PEX. UPnP и проброс портов не включайте автоматически: сначала определите, нужны ли входящие соединения и допустимы ли они в вашей сети.

## Источники

- [TorrServer: исходный проект и релизы](https://github.com/YouROK/TorrServer)
- [MikroTik: контейнеры](https://manual.mikrotik.com/docs/containers/)
- [MikroTik: параметры контейнера](https://manual.mikrotik.com/docs/cli-reference/container/)
- [MikroTik: mountlists](https://manual.mikrotik.com/docs/cli-reference/container/mounts/)
- [MikroTik: envlists](https://manual.mikrotik.com/docs/cli-reference/container/envs/)
