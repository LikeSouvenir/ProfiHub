# 02. Установка окружения (Windows), шаг за шагом

Этот конспект написан так, чтобы его можно было выполнять командой за командой, ничего не додумывая. Каждый шаг — проверяемый: сразу после него написано, как убедиться, что всё получилось, прежде чем идти дальше.

## Чек-лист того, что должно быть установлено к концу этого конспекта
- [ ] WSL 2 с дистрибутивом Ubuntu
- [ ] Docker Desktop, интегрированный с Ubuntu
- [ ] Node.js (внутри Ubuntu)
- [ ] cURL
- [ ] jq
- [ ] Go (понадобится позже, для chaincode на Go — можно поставить сразу)
- [ ] Java + Gradle (понадобится позже, для chaincode на Java — можно поставить сразу)

## Шаг 1. Установка WSL 2 и Ubuntu

Открыть **PowerShell от имени администратора** (правая кнопка мыши по значку PowerShell → "Запуск от имени администратора").

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```
Что это делает: первая команда включает саму подсистему Linux внутри Windows, вторая — компонент виртуализации, без которого WSL 2 не запустится.

```powershell
wsl --set-default-version 2
wsl --install -d Ubuntu
```
Что это делает: первая команда говорит Windows использовать более быструю версию WSL (2, с настоящим ядром Linux) для всех новых дистрибутивов. Вторая — скачивает и устанавливает Ubuntu.

**Перезагрузите компьютер**, если Windows об этом попросит.

После перезагрузки откройте **Ubuntu** из меню Пуск. При первом запуске она попросит придумать имя пользователя и пароль внутри Linux — они не связаны с паролем от Windows, придумайте любые (пароль пригодится дальше для команд `sudo`).

### ✅ Проверка шага 1
```powershell
wsl --list --verbose
```
Ожидаемый результат — в списке есть `Ubuntu`, и в колонке `VERSION` стоит `2`.

## Шаг 2. Установка Docker Desktop

1. Скачать с [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/) и установить с настройками по умолчанию.
2. Запустить Docker Desktop, дождаться, пока иконка кита в трее перестанет "крутиться" (означает, что движок запустился).
3. Открыть **Settings (⚙️) → Resources → WSL Integration**.
4. Включить переключатель **Enable integration with my default WSL distro**.
5. Ниже, в списке **Enable integration with additional distros**, включить **Ubuntu**.
6. Нажать **Apply & Restart**.

### ✅ Проверка шага 2
Открыть терминал **Ubuntu** (не PowerShell!) и ввести:
```bash
docker --version
docker ps
```
Ожидаемый результат: версия Docker выводится, `docker ps` выводит пустую таблицу без ошибок доступа. Если появляется `permission denied` — переходите к шагу 3, это нормально на этом этапе.

## Шаг 3. Права на запуск Docker без sudo (в Ubuntu)
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```
- `groupadd docker` — создаёт системную группу с именем `docker` (если её ещё нет — в норме команда может вывести "группа уже существует", это не ошибка).
- `usermod -aG docker $USER` — добавляет вашего пользователя в эту группу. `$USER` — системная переменная, автоматически подставляющая ваше имя пользователя Ubuntu.

**После этой команды обязательно закройте терминал Ubuntu полностью и откройте заново** (или перезагрузите компьютер) — изменение членства в группе применяется только к новым сессиям.

### ✅ Проверка шага 3
```bash
docker ps
```
Должно сработать **без** `sudo` и без ошибки `permission denied`.

## Шаг 4. Установка jq
```bash
sudo apt-get update
sudo apt-get install -y jq
```
`jq` — утилита для обработки JSON в bash. Используется скриптами Fabric при работе с конфигурацией каналов — без неё часть команд `network.sh` завершится с ошибкой на середине выполнения.

### ✅ Проверка шага 4
```bash
jq --version
```

## Шаг 5. Проверка Node.js
Node.js обычно уже установлен, но проверьте версию — Fabric SDK требует Node.js версии 18 или новее:
```bash
node -v
```
Если версия ниже 18 или Node.js не установлен вовсе:
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
```

### ✅ Проверка шага 5
```bash
node -v      # должно быть v18.x.x или выше
npm -v
```

## Шаг 6 (заранее, для будущих тем). Установка Go
Пригодится в конспекте про написание chaincode на Go — можно поставить сразу, чтобы не прерываться позже:
```bash
wget https://go.dev/dl/go1.22.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.22.0.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```

### ✅ Проверка шага 6
```bash
go version
```

## Шаг 7 (заранее, для будущих тем). Установка Java и Gradle
Пригодится в конспекте про написание chaincode на Java:
```bash
sudo apt-get install -y openjdk-17-jdk
sudo apt-get install -y gradle
```

### ✅ Проверка шага 7
```bash
java -version
gradle -version
```

## Частые проблемы этого этапа — таблица решений
| Симптом | Причина | Решение |
|---|---|---|
| `docker: command not found` в Ubuntu | WSL Integration не включена для Ubuntu | Повторить шаг 2, пункты 3–6 |
| `permission denied` при `docker ps` | Пользователь не в группе `docker`, либо сессия терминала не обновилась | Повторить шаг 3, затем **полностью перезапустить терминал** |
| WSL не устанавливается / зависает на "Установка" | В BIOS/UEFI выключена виртуализация (VT-x/AMD-V) | Включить виртуализацию в BIOS, затем повторить `wsl --install` |
| Docker Desktop пишет "Docker Engine stopped" | Выбран Hyper-V движок вместо WSL 2 (старые версии Docker Desktop) | Settings → General → включить "Use the WSL 2 based engine" |
| `jq: command not found` после установки | Установка была сделана не внутри Ubuntu, а в другой оболочке | Убедиться, что команда выполнялась именно в терминале Ubuntu |
| Проект "тормозит" при работе с сертификатами Fabric | Проект физически лежит на диске Windows (`/mnt/c/...`), а команды выполняются из WSL | Хранить и открывать проект внутри файловой системы Linux (`~/projects/...`), а не на диске Windows — граница файловых систем Windows/WSL заметно медленнее на большом числе мелких файлов, которых у Fabric очень много (сертификаты, MSP) |

## Итог
После этого конспекта в системе готово ровно то окружение, которое требуется для всех следующих шагов: контейнеризация (Docker), сама Linux-среда (WSL/Ubuntu), и три языка для будущего chaincode (Node.js — обязательно сейчас, Go и Java — впрок). Следующий конспект — установка самого Fabric и его образов.
