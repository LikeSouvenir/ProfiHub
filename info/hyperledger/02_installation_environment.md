# Установка окружения (Windows)

Fabric и все его инструменты рассчитаны на Linux-окружение, поэтому под Windows разработка ведётся через **WSL (Windows Subsystem for Linux)** — Linux-подсистему, работающую поверх Windows без установки отдельной виртуальной машины вручную.

## Необходимое ПО
| Инструмент | Зачем нужен |
|---|---|
| **WSL 2 + Ubuntu** | Linux-окружение, в котором реально выполняются скрипты Fabric |
| **Docker Desktop** | Peer, orderer, CA и CouchDB в Fabric запускаются как Docker-контейнеры |
| **Node.js** | Написание chaincode и клиентского API |
| **cURL** | Скачивание установочного скрипта Fabric |
| **jq** | Обработка JSON в bash-скриптах Fabric (используется скриптами `test-network` внутри) |

## Шаг 1. Установка WSL 2 и Ubuntu
Открыть PowerShell **от имени администратора** и выполнить по очереди:
```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

wsl --set-default-version 2
wsl --install -d Ubuntu
wsl --set-default Ubuntu
```
- Первые две команды включают саму подсистему Linux и компонент виртуализации, необходимый для WSL 2.
- `--set-default-version 2` делает WSL 2 (более быстрый, с полноценным ядром Linux) версией по умолчанию для новых дистрибутивов.
- `--install -d Ubuntu` скачивает и устанавливает сам дистрибутив Ubuntu.

После установки открыть Ubuntu (через меню Пуск или `wsl` в терминале) и задать имя пользователя и пароль при первом запуске.

## Шаг 2. Настройка Docker Desktop для работы с WSL
1. Открыть **Docker Desktop → Settings → Resources → WSL Integration**.
2. Включить **Enable integration with my default WSL distro**.
3. Включить переключатель напротив **Ubuntu**.
4. Нажать **Apply & Restart**.

Это связывает Docker Desktop (запущенный на Windows) с командой `docker`, вызываемой уже изнутри Ubuntu — без этой настройки команды Docker внутри WSL не будут работать.

### Права на выполнение Docker без sudo
Внутри терминала Ubuntu:
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```
- `groupadd docker` создаёт системную группу `docker`.
- `usermod -aG docker $USER` добавляет текущего пользователя (`$USER` — переменная окружения с именем пользователя) в эту группу, что даёт право запускать Docker-команды без постоянного набора `sudo` перед каждой из них.

Если после этих команд `docker` всё ещё требует `sudo` — стоит просто перезагрузить компьютер (изменение членства в группе применяется только к новым сессиям).

## Шаг 3. Установка jq
```bash
sudo apt-get update
sudo apt-get install jq
```
`jq` — утилита командной строки для обработки JSON, используется внутренними скриптами Fabric (например, при формировании конфигурации каналов) — без неё часть скриптов `test-network` завершится с ошибкой.

## Проверка готовности окружения
```bash
wsl --version        # проверить, что используется WSL 2 (в Windows, PowerShell)
docker --version      # проверить Docker (внутри Ubuntu)
docker ps             # должен вывести пустой список без ошибок доступа
node --version        # проверить Node.js
jq --version           # проверить jq
```

## Частые проблемы на этом этапе
| Проблема | Решение |
|---|---|
| `docker: command not found` внутри Ubuntu | Проверить, включена ли WSL Integration в Docker Desktop для нужного дистрибутива |
| `permission denied` при вызове docker без sudo | Перезайти в систему или перезагрузить компьютер после `usermod -aG docker` |
| WSL не запускается / зависает | Проверить, что виртуализация включена в BIOS/UEFI компьютера (для VirtualMachinePlatform) |
| Docker Desktop "Docker Engine stopped" | Убедиться, что в настройках выбран движок **WSL 2 based engine**, а не Hyper-V (в старых версиях Docker Desktop) |

## Оптимизация: чего стоит избегать
- **Не стоит** держать проект физически в файловой системе Windows (`C:\Users\...`), но открывать его из WSL — при активной работе Fabric со множеством мелких файлов (сертификаты, крипто-материалы) производительность файловой системы через границу Windows/WSL заметно ниже. Оптимальнее хранить и запускать проект **внутри файловой системы самого WSL** (например, `\\wsl$\Ubuntu\home\user\projects\...` или прямо через терминал Ubuntu в `~/projects`), открывая его в VS Code через расширение **WSL Remote**.
