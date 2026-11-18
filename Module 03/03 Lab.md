# Лабораторная работа №3: Интеграция Keycloak и RabbitMQ

## Общая информация

**Цель работы:** Получение практических навыков по настройке аутентификации RabbitMQ через OAuth 2.0 с использованием Keycloak в качестве Identity Provider (IdP).

**Продолжительность:** 90 минут

**ОС:** Debian 12/13 (headless)

**Стенд:** 3 виртуальные машины Debian 12 без GUI (из предыдущих лабораторных работ)

---

## Схема лабораторного стенда

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ЛАБОРАТОРНЫЙ СТЕНД №3                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐          │
│  │   Сервер 1       │    │   Сервер 2       │    │   Сервер 3       │          │
│  │   Keycloak-01    │    │   RabbitMQ-01    │    │   RabbitMQ-02    │          │
│  │   IP: .10        │    │   IP: .20        │    │   IP: .21        │          │
│  │   Порт: 8080     │    │   Порт: 5672     │    │   Порт: 5672     │          │
│  │                  │    │   Порт: 15672    │    │   Порт: 15672    │          │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘          │
│           │                      │                      │                    │
│           └──────────────────────┼──────────────────────┘                    │
│                                  │                                           │
│                                  ▼                                           │
│                   ┌─────────────────────────────┐                           │
│                   │      PostgreSQL (внешний)    │                           │
│                   │      192.168.1.10:5432      │                           │
│                   └─────────────────────────────┘                           │
│                                                                              │
│  Все серверы: Debian 12/13, без GUI, 2-4 GB RAM                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Часть 1: Подготовка инфраструктуры

### Задание 1.1: Настройка серверов

Если вы выполняете эту лабораторную работу после предыдущих, серверы уже настроены. В противном случае выполните базовую настройку на **каждом** сервере:

```bash
# Обновление системы
sudo apt update && sudo apt upgrade -y

# Установка необходимых пакетов
sudo apt install -y wget curl unzip gnupg2 ca-certificates

# Настройка hostname
# На первом сервере (Keycloak):
sudo hostnamectl set-hostname keycloak-01
# На втором сервере (RabbitMQ):
sudo hostnamectl set-hostname rabbitmq-01
# На третьем сервере (RabbitMQ):
sudo hostnamectl set-hostname rabbitmq-02

# Синхронизация времени (критично для JWT)
sudo timedatectl set-timezone Europe/Moscow
sudo apt install -y ntp
sudo systemctl enable --now ntp
```

### Задание 1.2: Настройка файла hosts

На **каждом** сервере добавьте записи в `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Добавьте (замените IP-адреса на реальные):

```
192.168.1.10   keycloak-01
192.168.1.20   rabbitmq-01
192.168.1.21   rabbitmq-02
```

Проверьте доступность узлов:

```bash
ping -c 3 rabbitmq-01
ping -c 3 rabbitmq-02
```

---

## Часть 2: Установка и настройка Keycloak

### Задание 2.1: Установка Keycloak

На **сервере keycloak-01** выполните установку (если Keycloak ещё не установлен):

```bash
# Установка Java
sudo apt install -y openjdk-17-jdk-headless

# Создание пользователя
sudo useradd -r -s /bin/false keycloak

# Создание каталогов
sudo mkdir -p /opt/keycloak
sudo mkdir -p /var/log/keycloak

# Назначение прав
sudo chown -R keycloak:keycloak /opt/keycloak
sudo chown -R keycloak:keycloak /var/log/keycloak

# Скачивание Keycloak
cd /tmp
sudo wget https://github.com/keycloak/keycloak/releases/download/24.0.1/keycloak-24.0.1.tar.gz

# Распаковка
sudo tar -xzf keycloak-24.0.1.tar.gz -C /opt/keycloak --strip-components=1

# Настройка прав
sudo chown -R keycloak:keycloak /opt/keycloak
sudo chmod +x /opt/keycloak/bin/kc.sh
```

### Задание 2.2: Настройка Keycloak.conf

На **сервере keycloak-01** создайте файл конфигурации:

```bash
sudo nano /opt/keycloak/conf/keycloak.conf
```

```properties
# HTTP настройки
http-host=0.0.0.0
http-port=8080
hostname-strict=false
hostname-strict-https=false

# Логирование
log-level=INFO
log-file=/var/log/keycloak/keycloak.log
```

### Задание 2.3: Создание администратора и запуск

```bash
# Создание администратора
sudo -u keycloak KEYCLOAK_ADMIN=admin KEYCLOAK_ADMIN_PASSWORD=admin123 /opt/keycloak/bin/kc.sh build

# Создание systemd сервиса
sudo nano /etc/systemd/system/keycloak.service
```

```ini
[Unit]
Description=Keycloak Identity and Access Management
After=network.target

[Service]
User=keycloak
Group=keycloak
Environment=KEYCLOAK_ADMIN=admin
Environment=KEYCLOAK_ADMIN_PASSWORD=admin123
ExecStart=/opt/keycloak/bin/kc.sh start-dev
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now keycloak
sudo systemctl status keycloak
```

### Задание 2.4: Настройка Realm для RabbitMQ

1. Откройте браузер и перейдите на `http://<IP-адрес_keycloak>:8080`
2. Войдите в Admin Console (`admin` / `admin123`)
3. Создайте Realm: **"rabbitmq"**

### Задание 2.5: Создание клиентов в Keycloak

Для работы RabbitMQ с OAuth 2.0 необходимо создать три клиента:

**Клиент 1: rabbitmq-client-code (для Management UI)**

1. Перейдите в **"Clients"** → **"Create client"**
2. **Client ID:** `rabbitmq-client-code`
3. **Client Protocol:** `OpenID Connect`
4. **Client authentication:** `Off` (Public)
5. **Standard flow:** `On`
6. **Valid redirect URIs:** `http://<IP-адрес_RabbitMQ>:15672/*`
7. Нажмите **"Save"**

**Клиент 2: mgt_api_client (для Management API)**

1. **Client ID:** `mgt_api_client`
2. **Client Protocol:** `OpenID Connect`
3. **Client authentication:** `On` (Confidential)
4. **Standard flow:** `On`
5. **Service accounts enabled:** `On`
6. Нажмите **"Save"**
7. Перейдите на вкладку **"Credentials"** и скопируйте **Client Secret**

**Клиент 3: producer (для AMQP-подключений)**

1. **Client ID:** `producer`
2. **Client Protocol:** `OpenID Connect`
3. **Client authentication:** `On` (Confidential)
4. **Standard flow:** `On`
5. **Service accounts enabled:** `On`
6. Нажмите **"Save"**
7. Перейдите на вкладку **"Credentials"** и скопируйте **Client Secret**

### Задание 2.6: Создание пользователя и назначение ролей

1. Перейдите в **"Users"** → **"Add user"**
   - **Username:** `rabbit_admin`
   - **Email:** `rabbit@test.local`
   - **First name:** `Rabbit`
   - **Last name:** `Admin`
   - **Email Verified:** поставьте галочку
2. Нажмите **"Create"**
3. Перейдите на вкладку **"Credentials"** → установите пароль: `rabbit_admin`
4. Уберите галочку **"Temporary"** → **"Set Password"**

**Назначение ролей пользователю:**

1. Перейдите в **"Realm Roles"** → **"Create role"**
   - Создайте роль: `rabbitmq.tag:administrator`
2. Откройте карточку пользователя `rabbit_admin` → вкладка **"Role Mappings"**
3. Назначьте роль `rabbitmq.tag:administrator`

---

## Часть 3: Установка и настройка RabbitMQ

### Задание 3.1: Установка RabbitMQ

На **каждом** сервере RabbitMQ (rabbitmq-01 и rabbitmq-02) выполните:

```bash
#!/bin/sh

sudo apt-get install curl gnupg apt-transport-https -y

## Team RabbitMQ's signing key
curl -1sLf "https://keys.openpgp.org/vks/v1/by-fingerprint/0A9AF2115F4687BD29803A206B73A36E6026DFCA" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/com.rabbitmq.team.gpg > /dev/null

## Add apt repositories maintained by Team RabbitMQ
sudo tee /etc/apt/sources.list.d/rabbitmq.list <<EOF
## Modern Erlang/OTP releases
##
deb [arch=amd64 signed-by=/usr/share/keyrings/com.rabbitmq.team.gpg] https://deb1.rabbitmq.com/rabbitmq-erlang/debian/bookworm bookworm main
deb [arch=amd64 signed-by=/usr/share/keyrings/com.rabbitmq.team.gpg] https://deb2.rabbitmq.com/rabbitmq-erlang/debian/bookworm bookworm main

## Latest RabbitMQ releases
##
deb [arch=amd64 signed-by=/usr/share/keyrings/com.rabbitmq.team.gpg] https://deb1.rabbitmq.com/rabbitmq-server/debian/bookworm bookworm main
deb [arch=amd64 signed-by=/usr/share/keyrings/com.rabbitmq.team.gpg] https://deb2.rabbitmq.com/rabbitmq-server/debian/bookworm bookworm main
EOF

## Update package indices
sudo apt-get update -y

## Install Erlang packages
##
## For versions not compatible with the latest available Erlang series, which is the case
## for 3.13.x, apt must be instructed to install specifically Erlang 26.
## Alternatively this can be done via version pinning, documented further in this guide.
supported_erlang_version="1:26.2.5.13-1"
sudo apt-get install -y erlang-base=$supported_erlang_version \
                        erlang-asn1=$supported_erlang_version \
                        erlang-crypto=$supported_erlang_version \
                        erlang-eldap=$supported_erlang_version \
                        erlang-ftp=$supported_erlang_version \
                        erlang-inets=$supported_erlang_version \
                        erlang-mnesia=$supported_erlang_version \
                        erlang-os-mon=$supported_erlang_version \
                        erlang-parsetools=$supported_erlang_version \
                        erlang-public-key=$supported_erlang_version \
                        erlang-runtime-tools=$supported_erlang_version \
                        erlang-snmp=$supported_erlang_version \
                        erlang-ssl=$supported_erlang_version \
                        erlang-syntax-tools=$supported_erlang_version \
                        erlang-tftp=$supported_erlang_version \
                        erlang-tools=$supported_erlang_version \
                        erlang-xmerl=$supported_erlang_version

## Install rabbitmq-server and its dependencies
sudo apt-get install rabbitmq-server -y --fix-missing
```

### Задание 3.2: Включение плагина OAuth 2.0

На **каждом** сервере RabbitMQ выполните:

```bash
# Включение плагина OAuth 2.0
sudo rabbitmq-plugins enable rabbitmq_auth_backend_oauth2

# Включение Management UI (если ещё не включен)
sudo rabbitmq-plugins enable rabbitmq_management

# Проверка включенных плагинов
sudo rabbitmq-plugins list
```

### Задание 3.3: Настройка rabbitmq.conf

На **каждом** сервере RabbitMQ создайте файл конфигурации:

```bash
sudo nano /etc/rabbitmq/rabbitmq.conf
```

Содержимое (замените `<IP_Keycloak>` на реальный IP-адрес сервера Keycloak):

```ini
# ============================================
# OAuth 2.0 Authentication Backend
# ============================================

# Включаем OAuth 2.0 как бэкенд аутентификации
auth_backends.1 = oauth2

# ============================================
# Настройка Resource Server
# ============================================
auth_oauth2.resource_server_id = rabbitmq

# ============================================
# Настройка Identity Provider
# ============================================
auth_oauth2.issuer = http://<IP_Keycloak>:8080/realms/rabbitmq

# ============================================
# Настройка имени пользователя из JWT
# ============================================
auth_oauth2.preferred_username_claims.1 = preferred_username
auth_oauth2.preferred_username_claims.2 = user_name

# ============================================
# Настройка дополнительных scopes из JWT
# ============================================
auth_oauth2.additional_scopes_key = realm_access.roles

# ============================================
# Настройка Management UI
# ============================================
management.oauth_enabled = true
management.oauth_client_id = rabbitmq-client-code
management.oauth_provider_url = http://<IP_Keycloak>:8080/realms/rabbitmq
management.oauth_scopes = openid profile email rabbitmq.tag:administrator

# ============================================
# Отключение проверки audience (для тестирования)
# ============================================
auth_oauth2.verify_aud = false

# ============================================
# Настройка логирования для отладки
# ============================================
log.default.level = debug
```

### Задание 3.4: Настройка прав доступа к файлам

```bash
sudo chown rabbitmq:rabbitmq /etc/rabbitmq/rabbitmq.conf
sudo chmod 640 /etc/rabbitmq/rabbitmq.conf
```

### Задание 3.5: Перезапуск RabbitMQ

```bash
sudo systemctl restart rabbitmq-server
sudo systemctl status rabbitmq-server
```

### Задание 3.6: Проверка логов

```bash
sudo journalctl -u rabbitmq-server -f | grep -i oauth
```

**Ожидаемые сообщения:** Информация о загрузке плагина OAuth 2.0 и успешной инициализации.

---

## Часть 4: Тестирование

### Задание 4.1: Доступ к Management UI через Keycloak

1. Откройте браузер и перейдите на `http://<IP-адрес_RabbitMQ>:15672`
2. Нажмите кнопку **"Click here to log in"** — произойдёт перенаправление на страницу входа Keycloak
3. Введите учётные данные:
   - **Username:** `rabbit_admin`
   - **Password:** `rabbit_admin`
4. После успешной аутентификации вы будете перенаправлены обратно в Management UI

### Задание 4.2: Проверка Management API через curl

На сервере Keycloak или любом другом узле выполните:

```bash
# Получение токена для mgt_api_client
curl -X POST http://<IP_Keycloak>:8080/realms/rabbitmq/protocol/openid-connect/token \
  -d "grant_type=client_credentials" \
  -d "client_id=mgt_api_client" \
  -d "client_secret=<SECRET>" > token.json

# Извлечение access_token
TOKEN=$(cat token.json | jq -r '.access_token')

# Проверка доступа к Management API
curl -H "Authorization: Bearer $TOKEN" http://<IP_RabbitMQ>:15672/api/overview
```

**Ожидаемый результат:** JSON-ответ с информацией о RabbitMQ (версия, статистика).

### Задание 4.3: Тестирование AMQP-подключения (Client Credentials Flow)

Для тестирования AMQP-подключения используйте утилиту `rabbitmqadmin` или Python-скрипт.

**Python-скрипт для тестирования:**

```bash
# Установка pika
pip3 install pika requests

# Создание скрипта
nano test_amqp_oauth.py
```

```python
#!/usr/bin/env python3
import pika
import requests
import json

# Получение токена от Keycloak
token_response = requests.post(
    'http://<IP_Keycloak>:8080/realms/rabbitmq/protocol/openid-connect/token',
    data={
        'grant_type': 'client_credentials',
        'client_id': 'producer',
        'client_secret': '<SECRET>'
    }
)
token_data = token_response.json()
access_token = token_data['access_token']

# Подключение к RabbitMQ с использованием токена
credentials = pika.PlainCredentials('', access_token)
parameters = pika.ConnectionParameters(
    host='<IP_RabbitMQ>',
    port=5672,
    credentials=credentials
)

connection = pika.BlockingConnection(parameters)
channel = connection.channel()

# Проверка: объявление очереди
channel.queue_declare(queue='test_queue', durable=True)
print("✅ Очередь успешно создана")

connection.close()
```

Запустите скрипт:

```bash
python3 test_amqp_oauth.py
```

### Задание 4.4: Проверка логов RabbitMQ

```bash
sudo journalctl -u rabbitmq-server -f
```

Вы должны увидеть сообщения о успешной аутентификации через OAuth 2.0.

---

## Часть 5: Кластеризация RabbitMQ с OAuth 2.0

### Задание 5.1: Настройка второго узла RabbitMQ

На **rabbitmq-02** повторите задания 3.1–3.5.

### Задание 5.2: Формирование кластера

На **rabbitmq-02** выполните:

```bash
# Остановка приложения
sudo rabbitmqctl stop_app

# Присоединение к кластеру
sudo rabbitmqctl join_cluster rabbit@rabbitmq-01

# Запуск приложения
sudo rabbitmqctl start_app

# Проверка статуса кластера
sudo rabbitmqctl cluster_status
```

### Задание 5.3: Проверка доступа через оба узла

1. Откройте Management UI на `http://<IP_rabbitmq-02>:15672`
2. Нажмите **"Click here to log in"** — вход через Keycloak должен работать на обоих узлах

---

## Часть 6: Диагностика и устранение неисправностей

### Задание 6.1: Проверка конфигурации

```bash
# Проверка файла конфигурации
sudo rabbitmqctl eval 'application:get_env(rabbitmq_auth_backend_oauth2, resource_server_id).'
sudo rabbitmqctl eval 'application:get_env(rabbitmq_auth_backend_oauth2, issuer).'
```

### Задание 6.2: Проверка токена вручную

Декодируйте JWT-токен, чтобы убедиться, что он содержит правильные поля:

```bash
# Установка jq
sudo apt install -y jq

# Декодирование токена (без проверки подписи)
echo "$TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | jq .
```

### Задание 6.3: Просмотр логов ошибок

```bash
# Логи RabbitMQ
sudo journalctl -u rabbitmq-server -f | grep -i error

# Логи Keycloak
sudo journalctl -u keycloak -f | grep -i error
```

---

## Чек-лист для самопроверки

| № | Действие | Статус |
|---|----------|--------|
| 1 | Keycloak установлен и запущен | ⬜ |
| 2 | Realm "rabbitmq" создан | ⬜ |
| 3 | Клиенты созданы (rabbitmq-client-code, mgt_api_client, producer) | ⬜ |
| 4 | Пользователь rabbit_admin создан | ⬜ |
| 5 | Роль rabbitmq.tag:administrator назначена | ⬜ |
| 6 | RabbitMQ установлен | ⬜ |
| 7 | Плагин rabbitmq_auth_backend_oauth2 включён | ⬜ |
| 8 | rabbitmq.conf настроен | ⬜ |
| 9 | Management UI доступен через Keycloak | ⬜ |
| 10 | Management API работает с токеном | ⬜ |
| 11 | AMQP-подключение работает | ⬜ |
| 12 | Кластер из 2 узлов работает | ⬜ |

---

## Дополнительные задания

### Задание 7.1: Настройка HTTPS для Keycloak

Для продакшена OAuth 2.0 требует HTTPS. Настройте SSL-сертификат для Keycloak и обновите `auth_oauth2.issuer` на `https://`.

### Задание 7.2: Настройка нескольких audiences

Настройте RabbitMQ для работы с несколькими OAuth 2.0 ресурсами.

### Задание 7.3: Интеграция с Redis (из предыдущей лабораторной)

Объедините настройки: Keycloak с Redis + RabbitMQ с OAuth 2.0.

### Задание 7.4: Настройка Service Account для приложений

Создайте сервисный аккаунт в Keycloak и настройте автоматическое получение токенов для микросервисов.

---

## Полезные команды

```bash
# Проверка статуса RabbitMQ
sudo systemctl status rabbitmq-server

# Просмотр логов RabbitMQ
sudo journalctl -u rabbitmq-server -f

# Проверка включенных плагинов
sudo rabbitmq-plugins list

# Проверка статуса кластера
sudo rabbitmqctl cluster_status

# Получение токена для producer
curl -X POST http://<IP_Keycloak>:8080/realms/rabbitmq/protocol/openid-connect/token \
  -d "grant_type=client_credentials" \
  -d "client_id=producer" \
  -d "client_secret=<SECRET>"

# Проверка Management API
curl -H "Authorization: Bearer $TOKEN" http://<IP_RabbitMQ>:15672/api/overview
```

---

**Конец лабораторной работы №3.**
