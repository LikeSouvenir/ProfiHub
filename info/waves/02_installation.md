# Установка окружения

Для разработки на Waves Enterprise нужны: **Docker** (чтобы поднять сеть нод и собирать образ контракта), **Java** (JDK, контракты пишутся на Java/Kotlin), **IntelliJ IDEA** (официально поддерживаемая IDE для WE Contract SDK) и опционально **Postman** — для ручных запросов к REST API ноды, пока не написан свой скрипт (см. файл про API).

## Список ПО
- **Docker Desktop** (Windows/macOS) или **Docker Engine** (Linux) — [docker.com](https://www.docker.com/products/docker-desktop/)
- **JDK 17** (Eclipse Temurin/Corretto — платформа собирается под Java 17, контейнер контракта использует `eclipse-temurin:17-jre-alpine`)
- **IntelliJ IDEA** — [jetbrains.com/idea](https://www.jetbrains.com/ru-ru/idea/download/)
- **Postman** (не обязательно, но удобно для первых ручных проверок) — [postman.com](https://www.postman.com/downloads/)

## Windows: WSL2 + Ubuntu
Docker-контейнеры на Windows под капотом работают внутри Linux-подсистемы, поэтому удобнее сразу настроить WSL2 и работать в терминале Ubuntu, а не в PowerShell.

Откройте **PowerShell от имени администратора**:
```powershell
# Включить подсистему Windows для Linux
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

# Включить компонент виртуальных машин (нужен WSL2)
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# Установить последнее ядро Linux для WSL
wsl.exe --install

# Сделать WSL2 версией по умолчанию
wsl --set-default-version 2
```
После перезагрузки поставьте **Ubuntu** из Microsoft Store и запустите её — при первом запуске она попросит задать имя пользователя и пароль Linux-окружения.

### Настройка Docker Desktop под WSL
1. В настройках Docker Desktop включите интеграцию с дистрибутивом Ubuntu (Settings → Resources → WSL Integration).
2. В терминале Ubuntu выполните:
```bash
sudo groupadd docker
sudo usermod -aG docker $USER   # $USER — ваше имя пользователя в Ubuntu
```
Вторая команда даёт вашему пользователю права выполнять `docker`-команды без `sudo` — иначе каждую команду пришлось бы запускать через `sudo`.

3. Перезайдите в терминал (или выполните `newgrp docker`), чтобы группа применилась.

## Linux
Если вы уже работаете в Linux — достаточно поставить Docker для вашего дистрибутива по официальной инструкции для конкретного дистрибутива с сайта docker.com.

Если после установки контейнеры не стартуют с ошибкой уровня безопасности — это, как правило, **SELinux**:
```bash
sudo setenforce 0
```
Команда временно переводит SELinux в permissive-режим (не блокирует, а только логирует нарушения) — этого достаточно для локальной разработки.

## Проверка установки
```bash
docker --version
docker run hello-world     # тестовый контейнер — должен вывести приветствие

java -version               # должна показать JDK 17
```
Если обе команды отработали без ошибок — можно переходить к запуску сети нод.
