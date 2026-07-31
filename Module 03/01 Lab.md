# Лабораторная работа: Развертывание и тестирование Keycloak на Debian 12

## Общая информация

**Цель работы:** Получение практических навыков по установке, настройке и тестированию Keycloak на сервере Debian 12 (без GUI).

**Продолжительность:** 90 минут

**ОС:** Debian 12/13 (headless)

**Стенд:** 3 виртуальные машины Debian 12 без GUI

---

## Схема лабораторного стенда

```
┌─────────────────────────────────────────────────────────────────┐
│                    ЛАБОРАТОРНЫЙ СТЕНД                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │   Сервер 1    │    │   Сервер 2   │    │   Сервер 3   │      │
│  │  keycloak-01  │    │ keycloak-02  │    │ keycloak-03  │      │
│  │  IP: .10      │    │  IP: .11     │    │  IP: .12     │      │
│  │  Порт: 8080   │    │  Порт: 8080  │    │  Порт: 8080  │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│           │                    │                    │           │
│           └────────────────────┼────────────────────┘           │
│                                │                                │
│                                ▼                                │
│                 ┌─────────────────────────────┐                │
│                 │      PostgreSQL (внешний)    │                │
│                 │      192.168.1.10:5432      │                │
│                 └─────────────────────────────┘                │
│                                                                  │
│  Все серверы: Debian 12/13, без GUI, 2-4 GB RAM                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Подготовка инфраструктуры

### Задание 1.1: Первоначальная настройка серверов

Выполните на **каждом** из трёх серверов:

```bash
# Обновление системы
sudo apt update && sudo apt upgrade -y

# Установка необходимых пакетов
sudo apt install -y wget curl unzip gnupg2 ca-certificates

# Настройка hostname
# На первом сервере:
sudo hostnamectl set-hostname keycloak-01
# На втором сервере:
sudo hostnamectl set-hostname keycloak-02
# На третьем сервере:
sudo hostnamectl set-hostname keycloak-03

# Проверка синхронизации времени
sudo timedatectl set-timezone Europe/Moscow
sudo apt install -y ntp
sudo systemctl enable --now ntp
sudo timedatectl status
```

### Задание 1.2: Настройка файла hosts

На **каждом** сервере отредактируйте файл `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Добавьте следующие строки (замените IP-адреса на реальные):

```
192.168.1.10   keycloak-01
192.168.1.11   keycloak-02
192.168.1.12   keycloak-03
```

Проверьте доступность узлов:

```bash
ping -c 3 keycloak-02
ping -c 3 keycloak-03
```

---

## Установка Java

### Задание 2.1: Установка OpenJDK 17

На **каждом** сервере выполните:

```bash
# Установка OpenJDK 17
sudo apt install -y openjdk-17-jdk-headless

# Проверка версии
java -version
```

**Ожидаемый результат:** `openjdk version "17.x.x"`

---

## Установка и настройка PostgreSQL

### Задание 3.1: Установка PostgreSQL

На **первом сервере** (выбранном как база данных) выполните:

```bash
# Установка PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Запуск и включение автозапуска
sudo systemctl enable --now postgresql

# Проверка статуса
sudo systemctl status postgresql
```

### Задание 3.2: Создание базы данных и пользователя

```bash
# Переключение на пользователя postgres
sudo -u postgres psql

# Выполните следующие SQL-команды
CREATE DATABASE keycloak;
CREATE USER keycloak_user WITH ENCRYPTED PASSWORD 'KC_SecurePass2024!';
GRANT ALL PRIVILEGES ON DATABASE keycloak TO keycloak_user;

# Выход из psql
\q
```

### Задание 3.3: Настройка доступа к PostgreSQL

Разрешите подключения с других узлов:

```bash
sudo nano /etc/postgresql/15/main/postgresql.conf
```

Найдите и измените строку:
```
listen_addresses = '*'
```

Настройте правила доступа:

```bash
sudo nano /etc/postgresql/15/main/pg_hba.conf
```

Добавьте в конец файла (замените подсеть на вашу):
```
host    keycloak        keycloak_user   192.168.1.0/24          scram-sha-256
```

Перезапустите PostgreSQL:

```bash
sudo systemctl restart postgresql

# Проверка доступности с других узлов
sudo netstat -tulpn | grep 5432
```

---

## Установка Keycloak

### Задание 4.1: Скачивание и установка

На **каждом** сервере выполните:

```bash
# Создание пользователя
sudo useradd -r -s /bin/false keycloak

# Создание каталогов
sudo mkdir -p /opt/keycloak
sudo mkdir -p /var/log/keycloak
sudo mkdir -p /etc/keycloak

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

### Задание 4.2: Создание конфигурационного файла

На **каждом** сервере создайте файл конфигурации:

```bash
sudo nano /opt/keycloak/conf/keycloak.conf
```

Содержимое (настройте IP-адрес сервера БД):

```properties
# База данных
db=postgres
db-url=jdbc:postgresql://192.168.1.10:5432/keycloak
db-username=keycloak_user
db-password=KC_SecurePass2024!

# HTTP настройки
http-host=0.0.0.0
http-port=8080
https-port=8443
hostname-strict=false
hostname-strict-https=false

# Логирование
log-level=INFO
log-file=/var/log/keycloak/keycloak.log
```

### Задание 4.3: Создание администратора

```bash
sudo -u keycloak KEYCLOAK_ADMIN=admin KEYCLOAK_ADMIN_PASSWORD=admin123 /opt/keycloak/bin/kc.sh build
```

---

## Запуск и тестирование

### Задание 5.1: Запуск в режиме разработки

```bash
sudo -u keycloak /opt/keycloak/bin/kc.sh start-dev
```

**Проверка:** Откройте браузер и перейдите на `http://<IP-адрес_сервера>:8080`

### Задание 5.2: Создание systemd сервиса

```bash
sudo nano /etc/systemd/system/keycloak.service
```

Содержимое:

```ini
[Unit]
Description=Keycloak Identity and Access Management
After=network.target postgresql.service

[Service]
User=keycloak
Group=keycloak
Environment=KEYCLOAK_ADMIN=admin
Environment=KEYCLOAK_ADMIN_PASSWORD=admin123
ExecStart=/opt/keycloak/bin/kc.sh start
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Запустите сервис:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now keycloak
sudo systemctl status keycloak
```

---

## Тестирование Keycloak

### Задание 6.1: Вход в консоль администратора

1. Откройте браузер и перейдите на `http://<IP-адрес>:8080`
2. Нажмите **"Administration Console"**
3. Введите учетные данные: `admin` / `admin123`

### Задание 6.2: Создание Realm

1. Наведите на выпадающий список `Master` в левом верхнем углу
2. Нажмите **"Create Realm"**
3. Введите имя: `TestRealm`
4. Нажмите **"Create"**

### Задание 6.3: Создание пользователя

1. В левом меню перейдите в **"Users"**
2. Нажмите **"Add user"**
3. Заполните поля:
   - **Username:** `testuser`
   - **Email:** `test@test.local`
   - **First name:** `Test`
   - **Last name:** `User`
   - **Email Verified:** поставьте галочку
4. Нажмите **"Create"**
5. Перейдите на вкладку **"Credentials"**
6. Установите пароль: `Test123!`
7. Уберите галочку **"Temporary"**
8. Нажмите **"Set Password"**

### Задание 6.4: Создание клиента

1. Перейдите в раздел **"Clients"**
2. Нажмите **"Create client"**
3. **Client ID:** `test-app`
4. **Client Protocol:** `OpenID Connect`
5. Нажмите **"Next"**
6. **Client authentication:** `Off`
7. **Standard flow:** `On`
8. Нажмите **"Next"**
9. **Valid redirect URIs:** `http://localhost:8080/*`
10. Нажмите **"Save"**

### Задание 6.5: Тестирование аутентификации

Сформируйте URL для авторизации:

```
http://<IP-адрес>:8080/realms/TestRealm/protocol/openid-connect/auth
?client_id=test-app
&redirect_uri=http://localhost:8080
&response_type=code
&scope=openid%20profile%20email
```

Откройте этот URL в браузере, введите учетные данные `testuser` / `Test123!` и убедитесь, что происходит перенаправление.

### Задание 6.6: Получение токена через CLI

На сервере выполните:

```bash
curl -X POST http://localhost:8080/realms/TestRealm/protocol/openid-connect/token \
  -d "grant_type=password" \
  -d "client_id=test-app" \
  -d "username=testuser" \
  -d "password=Test123!"
```

**Ожидаемый результат:** JSON-ответ с `access_token`

---

## Дополнительные задания

### Задание 7.1: Установка Keycloak на второй сервер

Повторите задания 4.1, 4.2, 4.3, 5.2 на втором и третьем серверах.

### Задание 7.2: Настройка кластеризации

На **каждом** сервере добавьте в `/opt/keycloak/conf/keycloak.conf`:

```properties
cache=ispn
cache-stack=tcp
```

### Задание 7.3: Включение двухфакторной аутентификации

1. Перейдите в **Realm Settings** → **Authentication** → **Flows**
2. Найдите поток **"Browser"** → **"Actions"** → **"Duplicate"**
3. Назовите: `Browser with OTP`
4. В скопированном потоке найдите **"OTP Form"**
5. Измените состояние с `DISABLED` на **`REQUIRED`**
6. Выберите **"Browser with OTP"** как **"Browser Flow"**

### Задание 7.4: Включение защиты от брутфорса

1. Перейдите в **Realm Settings** → **"Security Defenses"**
2. Найдите **"Brute Force Detection"** → включите `On`
3. Настройте:
   - **Max Login Failures:** `5`
   - **Failure Reset Time:** `300`
   - **Minimum Quick Login Wait Time:** `60`

---

## Проверка работоспособности

### Чек-лист для самопроверки

| № | Действие | Статус |
|---|----------|--------|
| 1 | Java 17 установлен | ✅ |
| 2 | PostgreSQL запущен | ✅ |
| 3 | База данных и пользователь созданы | ✅ |
| 4 | Keycloak установлен | ✅ |
| 5 | Keycloak запущен через systemd | ✅ |
| 6 | Admin Console доступна | ✅ |
| 7 | Realm создан | ✅ |
| 8 | Пользователь создан | ✅ |
| 9 | Клиент создан | ✅ |
| 10 | Токен получен через curl | ✅ |
| 11 | Кластер работает (если настраивался) | ✅ |
| 12 | 2FA работает (если настраивалась) | ✅ |

### Полезные команды для проверки

```bash
# Проверка статуса Keycloak
sudo systemctl status keycloak

# Просмотр логов
sudo journalctl -u keycloak -f
tail -f /var/log/keycloak/keycloak.log

# Проверка портов
sudo netstat -tulpn | grep 8080

# Проверка подключения к БД
psql -h 192.168.1.10 -U keycloak_user -d keycloak -W
```


---

**Конец лабораторной работы.**
