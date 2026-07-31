# Лабораторная работа №2: Интеграция Keycloak с Redis

## Общая информация

**Цель работы:** Получение практических навыков по интеграции Keycloak с внешним кеш-хранилищем Redis для обеспечения отказоустойчивости и сохранения сессий при перезапуске узлов.

**Продолжительность:** 90 минут

**ОС:** Debian 12/13 (headless)

**Стенд:** 3 виртуальные машины Debian 12 без GUI (из предыдущей лабораторной работы)

---

## Теоретическое введение

**Важно:** Keycloak **не поддерживает Redis нативно**, так как его система кеширования построена на Infinispan. Однако существуют два основных подхода для использования Redis:

1. **Skycloak/Locke** — форк Keycloak, позволяющий переключить кеш на Redis одной переменной окружения.
2. **Phase Two** — расширение, реализующее замену распределённых кешей Infinispan на Redis.

В данной лабораторной работе используется **Skycloak/Locke** как наиболее простой способ для ознакомления.

**Преимущества Redis перед Infinispan:**
- Быстрый старт Pod'а (менее 5 секунд)
- Отсутствие проблем с перебалансировкой JGroups
- Сохранение сессий при перезапуске узлов

---

## Схема лабораторного стенда

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ЛАБОРАТОРНЫЙ СТЕНД №2                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐          │
│  │   Сервер 1       │    │   Сервер 2       │    │   Сервер 3       │          │
│  │   Keycloak-01    │    │   Keycloak-02    │    │   Keycloak-03    │          │
│  │   (Locke)        │    │   (Locke)        │    │   (Locke)        │          │
│  │   IP: .10        │    │   IP: .11        │    │   IP: .12        │          │
│  │   Порт: 8080     │    │   Порт: 8080     │    │   Порт: 8080     │          │
│  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘          │
│           │                      │                      │                    │
│           └──────────────────────┼──────────────────────┘                    │
│                                  │                                           │
│                                  ▼                                           │
│                   ┌─────────────────────────────┐                           │
│                   │         Redis Server         │                           │
│                   │      192.168.1.20:6379       │                           │
│                   │   (общее хранилище сессий)   │                           │
│                   └─────────────────────────────┘                           │
│                                                                              │
│  Все серверы: Debian 12/13, без GUI, 2-4 GB RAM                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Часть 1: Подготовка инфраструктуры

### Задание 1.1: Настройка серверов

Если вы выполняете эту лабораторную работу после первой, серверы уже настроены. В противном случае выполните базовую настройку на **каждом** сервере:

```bash
# Обновление системы
sudo apt update && sudo apt upgrade -y

# Установка необходимых пакетов
sudo apt install -y wget curl unzip gnupg2 ca-certificates

# Настройка hostname (на каждом сервере своё имя)
sudo hostnamectl set-hostname keycloak-01   # на первом
sudo hostnamectl set-hostname keycloak-02   # на втором
sudo hostnamectl set-hostname keycloak-03   # на третьем

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
192.168.1.11   keycloak-02
192.168.1.12   keycloak-03
192.168.1.20   redis-server
```

Проверьте доступность:

```bash
ping -c 3 keycloak-02
ping -c 3 keycloak-03
ping -c 3 redis-server
```

---

## Часть 2: Установка и настройка Redis

### Задание 2.1: Установка Redis

На **отдельном сервере** (или на одном из существующих, выделенном под Redis) выполните:

```bash
# Установка Redis
sudo apt install -y redis-server

# Проверка версии
redis-server --version
```

### Задание 2.2: Настройка Redis для работы с Keycloak

Отредактируйте конфигурационный файл:

```bash
sudo nano /etc/redis/redis.conf
```

Найдите и измените следующие параметры:

```conf
# Разрешаем подключения с других узлов
bind 0.0.0.0

# Устанавливаем пароль (обязательно для продакшена)
requirepass KC_Redis_Pass2024!

# Политика вытеснения (важно для предотвращения ошибок при переполнении)
maxmemory 1gb
maxmemory-policy allkeys-lru

# Включаем сохранение на диск (RDB)
save 900 1
save 300 10
save 60 10000
```

### Задание 2.3: Запуск Redis

```bash
# Перезапуск с новыми настройками
sudo systemctl restart redis-server

# Включение автозапуска
sudo systemctl enable redis-server

# Проверка статуса
sudo systemctl status redis-server

# Проверка работы с паролем
redis-cli -a KC_Redis_Pass2024! PING
# Ожидаемый ответ: PONG
```

### Задание 2.4: Проверка доступности Redis с других узлов

С **любого** сервера Keycloak выполните:

```bash
redis-cli -h redis-server -p 6379 -a KC_Redis_Pass2024! PING
# Ожидаемый ответ: PONG
```

---

## Часть 3: Установка Java

На **каждом** сервере Keycloak выполните:

```bash
sudo apt install -y openjdk-17-jdk-headless
java -version
```

**Ожидаемый результат:** `openjdk version "17.x.x"`

---

## Часть 4: Установка Keycloak (Locke)

Вместо стандартного Keycloak используем **Skycloak/Locke** — форк с поддержкой Redis.

### Задание 4.1: Скачивание и установка Locke

На **каждом** сервере Keycloak выполните:

```bash
# Создание пользователя
sudo useradd -r -s /bin/false keycloak

# Создание каталогов
sudo mkdir -p /opt/keycloak
sudo mkdir -p /var/log/keycloak

# Назначение прав
sudo chown -R keycloak:keycloak /opt/keycloak
sudo chown -R keycloak:keycloak /var/log/keycloak

# Скачивание Locke (используем актуальную версию)
cd /tmp
sudo wget https://github.com/keycloak/keycloak/releases/download/26.0.7/keycloak-26.0.7.tar.gz

# Распаковка
sudo tar -xzf keycloak-26.0.7.tar.gz -C /opt/keycloak --strip-components=1

# Настройка прав
sudo chown -R keycloak:keycloak /opt/keycloak
sudo chmod +x /opt/keycloak/bin/kc.sh
```

**Примечание:** Locke использует те же бинарные файлы, что и Keycloak, но с дополнительной логикой переключения кеша.

### Задание 4.2: Настройка Keycloak.conf

На **каждом** сервере создайте файл конфигурации:

```bash
sudo nano /opt/keycloak/conf/keycloak.conf
```

Содержимое (настройте IP-адрес сервера Redis и PostgreSQL):

```properties
# База данных (используем PostgreSQL из первой лабораторной)
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

# ===== НАСТРОЙКА REDIS (ключевое отличие) =====
# Переключаем кеш на Redis
cache=redis
# URL подключения к Redis (формат: redis://:пароль@хост:порт)
cache-redis-url=redis://:KC_Redis_Pass2024!@redis-server:6379

# Логирование
log-level=INFO
log-file=/var/log/keycloak/keycloak.log
```

**Пояснение:** Переменная `cache=redis` переключает Keycloak на использование Redis вместо встроенного Infinispan.

### Задание 4.3: Создание администратора

```bash
sudo -u keycloak KEYCLOAK_ADMIN=admin KEYCLOAK_ADMIN_PASSWORD=admin123 /opt/keycloak/bin/kc.sh build
```

---

## Часть 5: Запуск и тестирование

### Задание 5.1: Ручной запуск (проверка)

```bash
sudo -u keycloak /opt/keycloak/bin/kc.sh start-dev
```

В логах должны появиться сообщения о подключении к Redis.

### Задание 5.2: Создание systemd сервиса

На **каждом** сервере создайте файл службы:

```bash
sudo nano /etc/systemd/system/keycloak.service
```

```ini
[Unit]
Description=Keycloak Identity and Access Management (with Redis)
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

### Задание 5.3: Проверка логов

Убедитесь, что Keycloak успешно подключился к Redis:

```bash
sudo journalctl -u keycloak -f | grep -i redis
```

**Ожидаемые сообщения:** Информация об успешном подключении к Redis.

---

## Часть 6: Тестирование работы Redis

### Задание 6.1: Создание Realm и пользователя

1. Откройте браузер и перейдите на `http://<IP-адрес_сервера>:8080`
2. Войдите в Admin Console (`admin` / `admin123`)
3. Создайте Realm: **"TestRealm"**
4. Создайте пользователя: **"testuser"** с паролем **"Test123!"**
5. Создайте клиент: **"test-app"** (тип Public)

### Задание 6.2: Проверка хранения сессий в Redis

1. Выполните вход от имени пользователя `testuser`
2. На сервере Redis проверьте наличие ключей сессий:

```bash
redis-cli -a KC_Redis_Pass2024! KEYS "*"
```

Вы должны увидеть ключи, начинающиеся с `keycloak-session:` или подобных.

### Задание 6.3: Тест сохранения сессии при перезапуске

1. Войдите в Keycloak как пользователь `testuser`
2. Перезапустите контейнер/сервис Keycloak на **одном** из узлов:

```bash
sudo systemctl restart keycloak
```

3. **Проверка:** После перезапуска сессия пользователя должна сохраниться — повторный вход не потребуется.

4. Проверьте содержимое Redis:

```bash
redis-cli -a KC_Redis_Pass2024! KEYS "keycloak-session:*"
redis-cli -a KC_Redis_Pass2024! HGETALL keycloak-session:<ID_сессии>
```

---

## Часть 7: Настройка кластера Keycloak с Redis

### Задание 7.1: Установка Keycloak на все узлы

Повторите задания 4.1, 4.2, 4.3, 5.2 на **втором** и **третьем** серверах.

### Задание 7.2: Настройка кластеризации

На **каждом** сервере в файле `/opt/keycloak/conf/keycloak.conf` добавьте:

```properties
# Настройки кластеризации
cache-stack=tcp
```

### Задание 7.3: Тестирование кластера

1. Запустите Keycloak на всех трёх узлах
2. Войдите через первый узел
3. Переключитесь на второй узел в браузере (тот же домен, другой IP)
4. Сессия должна сохраниться благодаря общему Redis

---

## Часть 8: Мониторинг и диагностика

### Задание 8.1: Просмотр статистики Redis

```bash
redis-cli -a KC_Redis_Pass2024! INFO stats
```

Ключевые метрики:
- `total_connections_received` — число подключений
- `total_commands_processed` — нагрузка
- `used_memory` — использование памяти

### Задание 8.2: Просмотр активных сессий в Redis

```bash
# Количество ключей сессий
redis-cli -a KC_Redis_Pass2024! KEYS "keycloak-session:*" | wc -l

# Просмотр одной сессии
redis-cli -a KC_Redis_Pass2024! HGETALL keycloak-session:<ID>
```

### Задание 8.3: Логи Keycloak

```bash
# Просмотр логов в реальном времени
sudo journalctl -u keycloak -f

# Поиск ошибок Redis
sudo journalctl -u keycloak | grep -i redis
```

---

## Дополнительные задания

### Задание 9.1: Настройка Redis Sentinel (отказоустойчивость)

Для продакшена рекомендуется использовать Redis Sentinel:

```bash
# Установка Sentinel
sudo apt install -y redis-sentinel

# Настройка sentinel.conf
sudo nano /etc/redis/sentinel.conf
```

Добавьте:

```conf
sentinel monitor mymaster 192.168.1.20 6379 2
sentinel auth-pass mymaster KC_Redis_Pass2024!
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
```

Запустите Sentinel:

```bash
sudo systemctl enable --now redis-sentinel
```

### Задание 9.2: Использование Phase Two вместо Locke

Phase Two — это расширение, реализующее замену распределённых кешей Infinispan на Redis. Для его использования требуется сборка из исходников.

---

## Чек-лист для самопроверки

| № | Действие | Статус |
|---|----------|--------|
| 1 | Redis установлен и настроен | ⬜ |
| 2 | Redis доступен с узлов Keycloak | ⬜ |
| 3 | Locke (Keycloak) установлен | ⬜ |
| 4 | Настроен `cache=redis` в keycloak.conf | ⬜ |
| 5 | Указан `cache-redis-url` | ⬜ |
| 6 | Keycloak запущен через systemd | ⬜ |
| 7 | Realm и пользователь созданы | ⬜ |
| 8 | Сессии видны в Redis | ⬜ |
| 9 | Сессия сохраняется после перезапуска | ⬜ |
| 10 | Кластер из 3 узлов работает | ⬜ |

---

## Полезные команды

```bash
# Проверка статуса Keycloak
sudo systemctl status keycloak

# Просмотр логов
sudo journalctl -u keycloak -f

# Проверка Redis
redis-cli -a KC_Redis_Pass2024! PING

# Просмотр всех ключей в Redis
redis-cli -a KC_Redis_Pass2024! KEYS "*"

# Просмотр статистики Redis
redis-cli -a KC_Redis_Pass2024! INFO stats

# Очистка всех сессий в Redis (осторожно!)
redis-cli -a KC_Redis_Pass2024! FLUSHDB
```


**Конец лабораторной работы №2.**
