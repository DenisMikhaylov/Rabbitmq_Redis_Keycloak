# Лабораторная работа: Развертывание и настройка RabbitMQ

## Создание отказоустойчивого кластера RabbitMQ на Debian 13

---

## Цель работы

Получить практические навыки по установке, настройке и управлению кластером RabbitMQ. В ходе выполнения лабораторной работы вы развернете кластер из трех узлов, настроите веб-интерфейс, создадите обменники и очереди, настроите маршрутизацию и проверите работу брокера через отправку и получение сообщений.

---

## Требования к окружению

- **Операционная система:** Debian 13 (без GUI) на всех трех узлах
- **ОЗУ:** минимум 2 ГБ на каждый узел
- **Дисковое пространство:** минимум 5 ГБ
- **Сеть:** Узлы должны иметь возможность общаться друг с другом по сети

---

## Топология стенда

```
┌─────────────────────────────────────────────────────────────┐
│                      Лабораторный стенд                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │   Node 1    │  │   Node 2    │  │   Node 3    │       │
│  │ rabbit@mq1  │  │ rabbit@mq2  │  │ rabbit@mq3  │       │
│  │ 192.168.1.10│  │ 192.168.1.11│  │ 192.168.1.12│       │
│  │             │  │             │  │             │       │
│  │   Master    │  │   Replica   │  │   Replica   │       │
│  └─────────────┘  └─────────────┘  └─────────────┘       │
│         │                │                │               │
│         └────────────────┼────────────────┘               │
│                          │                                 │
│                   Общая сеть /24                           │
└─────────────────────────────────────────────────────────────┘
```

---

## Часть 1. Подготовка узлов

### 1.1. Настройка сети и хостов

На каждом узле необходимо настроить `/etc/hosts` для разрешения имен друг друга:

```bash
# Выполнить на ВСЕХ узлах
sudo tee -a /etc/hosts << EOF
192.168.1.10  mq1
192.168.1.11  mq2
192.168.1.12  mq3
EOF
```

### 1.2. Обновление системы

```bash
# На каждом узле
sudo apt update && sudo apt upgrade -y
```

### 1.3. Установка зависимостей

```bash
# На каждом узле
sudo apt install -y wget gnupg apt-transport-https curl software-properties-common
```

---

## Часть 2. Установка RabbitMQ

### 2.1. Добавление официального репозитория

```bash
# На каждом узле
curl -1sLf "https://keys.openpgp.org/vks/v1/by-fingerprint/0A9AF2115F4687BD29803A206B73A36E6026DFCA" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/com.rabbitmq.team.gpg > /dev/null

curl -1sLf "https://keys.openpgp.org/vks/v1/by-fingerprint/F77F1EDA57EBB9CCB02B5BD3046C200EA6671B6C" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/io.rabbitmq.erlang.gpg > /dev/null

# Добавление репозитория Erlang
sudo tee /etc/apt/sources.list.d/rabbitmq.list << EOF
deb [signed-by=/usr/share/keyrings/io.rabbitmq.erlang.gpg] https://ppa1.novemberain.com/rabbitmq/rabbitmq-erlang/deb/ubuntu noble main
deb [signed-by=/usr/share/keyrings/com.rabbitmq.team.gpg] https://ppa1.novemberain.com/rabbitmq/rabbitmq-server/deb/ubuntu noble main
EOF
```

### 2.2. Установка RabbitMQ

```bash
# На каждом узле
sudo apt update
sudo apt install -y erlang-base \
                    erlang-asn1 erlang-crypto erlang-eldap erlang-ftp erlang-inets \
                    erlang-mnesia erlang-os-mon erlang-parsetools erlang-public-key \
                    erlang-runtime-tools erlang-ssl erlang-syntax-tools erlang-tftp \
                    erlang-tools erlang-xmerl rabbitmq-server
```

### 2.3. Проверка установки

```bash
# На каждом узле
sudo systemctl status rabbitmq-server
sudo rabbitmqctl status
```

---

## Часть 3. Настройка кластера

### 3.1. Остановка сервиса на всех узлах

```bash
# На каждом узле
sudo systemctl stop rabbitmq-server
```

### 3.2. Настройка Erlang Cookie

Erlang Cookie должен быть одинаковым на всех узлах кластера. Используем cookie с первого узла:

```bash
# На узле mq1 - сохраняем cookie
sudo cat /var/lib/rabbitmq/.erlang.cookie
# Скопируйте вывод (например, "PVTCJBOMJCEUITZEBGIP")

# На узлах mq2 и mq3 - устанавливаем такой же cookie
sudo sh -c 'echo "PVTCJBOMJCEUITZEBGIP" > /var/lib/rabbitmq/.erlang.cookie'
sudo chmod 0400 /var/lib/rabbitmq/.erlang.cookie
```

### 3.3. Формирование кластера

**На узле mq1 (главный узел):**
```bash
sudo systemctl start rabbitmq-server
sudo rabbitmqctl cluster_status
# Должен отображаться один узел: rabbit@mq1
```

**На узле mq2:**
```bash
sudo systemctl start rabbitmq-server
sudo rabbitmqctl stop_app
sudo rabbitmqctl reset
sudo rabbitmqctl join_cluster rabbit@mq1
sudo rabbitmqctl start_app
```

**На узле mq3:**
```bash
sudo systemctl start rabbitmq-server
sudo rabbitmqctl stop_app
sudo rabbitmqctl reset
sudo rabbitmqctl join_cluster rabbit@mq1
sudo rabbitmqctl start_app
```

### 3.4. Проверка кластера

```bash
# На любом узле
sudo rabbitmqctl cluster_status
```

**Ожидаемый результат:**
```
Cluster status of node rabbit@mq1 ...
Basics

Cluster name: rabbit@mq1

Disk Nodes

rabbit@mq1
rabbit@mq2
rabbit@mq3

Running Nodes

rabbit@mq1
rabbit@mq2
rabbit@mq3
```

---

## Часть 4. Включение веб-интерфейса

### 4.1. Активация Management Plugin

```bash
# На каждом узле
sudo rabbitmq-plugins enable rabbitmq_management
```

### 4.2. Проверка работы веб-интерфейса

```bash
# Проверяем, что порт 15672 открыт
sudo ss -tlnp | grep 15672
```

### 4.3. Создание администратора

```bash
# На любом узле (например, mq1)
sudo rabbitmqctl add_user admin AdminPass123!
sudo rabbitmqctl set_user_tags admin administrator
sudo rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```

### 4.4. Доступ к веб-интерфейсу

Откройте браузер и перейдите по адресу: `http://192.168.1.10:15672`

Логин: `admin`
Пароль: `AdminPass123!`

---

## Часть 5. Настройка политик высокой доступности

### 5.1. Создание политики через веб-интерфейс

1. Перейдите в раздел **Admin → Policies**
2. Нажмите **Add policy**
3. Заполните поля:
   - **Name:** `ha-policy`
   - **Pattern:** `^.*$` (применить ко всем очередям)
   - **Apply to:** `Queues`
   - **Definition:**
     ```json
     {
       "ha-mode": "exactly",
       "ha-params": 2,
       "ha-sync-mode": "automatic"
     }
     ```
4. Нажмите **Add policy**

### 5.2. Создание политики через командную строку

```bash
sudo rabbitmqctl set_policy ha-policy "^.*$" '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}' --apply-to queues
```

---

## Часть 6. Создание обменников и очередей

### 6.1. Создание обменников

**Через веб-интерфейс:**

1. Перейдите в раздел **Exchanges**
2. Нажмите **Add a new exchange**
3. Создайте следующие обменники:

| Имя обменника | Тип | Durability |
|---------------|-----|------------|
| `app.logs` | topic | Durable |
| `app.events` | direct | Durable |
| `app.notifications` | fanout | Durable |

**Через командную строку:**

```bash
sudo rabbitmqctl add_exchange / app.logs topic --durable
sudo rabbitmqctl add_exchange / app.events direct --durable
sudo rabbitmqctl add_exchange / app.notifications fanout --durable
```

### 6.2. Создание очередей

**Через веб-интерфейс:**

1. Перейдите в раздел **Queues**
2. Нажмите **Add a new queue**
3. Создайте следующие очереди:

| Имя очереди | Durability | Arguments |
|-------------|------------|-----------|
| `logs.error` | Durable | x-dead-letter-exchange: dead-letter |
| `logs.warning` | Durable | x-message-ttl: 60000 |
| `logs.info` | Durable | - |
| `events.created` | Durable | - |
| `events.updated` | Durable | - |
| `notifications.email` | Durable | - |
| `notifications.sms` | Durable | - |
| `dead-letter` | Durable | - |

**Через командную строку:**

```bash
# Очереди для логов
sudo rabbitmqctl add_queue logs.error --durable
sudo rabbitmqctl add_queue logs.warning --durable
sudo rabbitmqctl add_queue logs.info --durable

# Очереди для событий
sudo rabbitmqctl add_queue events.created --durable
sudo rabbitmqctl add_queue events.updated --durable

# Очереди для уведомлений
sudo rabbitmqctl add_queue notifications.email --durable
sudo rabbitmqctl add_queue notifications.sms --durable

# Dead Letter очередь
sudo rabbitmqctl add_queue dead-letter --durable
```

### 6.3. Настройка привязок (Bindings)

**Через веб-интерфейс:**

1. Перейдите в раздел **Exchanges**
2. Нажмите на имя обменника `app.logs`
3. В разделе **Bindings** добавьте следующие привязки:

| Exchange | Queue | Routing Key |
|----------|-------|-------------|
| app.logs | logs.error | `*.error` |
| app.logs | logs.warning | `*.warning` |
| app.logs | logs.info | `*.info` |
| app.logs | logs.warning | `*.critical` |
| app.events | events.created | `create` |
| app.events | events.updated | `update` |
| app.notifications | notifications.email | (игнорируется) |
| app.notifications | notifications.sms | (игнорируется) |

**Через командную строку:**

```bash
# Привязки для обменника app.logs
sudo rabbitmqctl add_binding / app.logs logs.error '*.error'
sudo rabbitmqctl add_binding / app.logs logs.warning '*.warning'
sudo rabbitmqctl add_binding / app.logs logs.info '*.info'
sudo rabbitmqctl add_binding / app.logs logs.warning '*.critical'

# Привязки для обменника app.events
sudo rabbitmqctl add_binding / app.events events.created create
sudo rabbitmqctl add_binding / app.events events.updated update

# Привязки для обменника app.notifications (Fanout - ключ игнорируется)
sudo rabbitmqctl add_binding / app.notifications notifications.email
sudo rabbitmqctl add_binding / app.notifications notifications.sms
```

---

## Часть 7. Отправка и получение сообщений

### 7.1. Отправка сообщений через веб-интерфейс

1. Перейдите в раздел **Exchanges**
2. Нажмите на имя обменника `app.logs`
3. В разделе **Publish message** заполните поля:

**Пример 1: Отправка сообщения в очередь logs.error**
- **Routing key:** `error`
- **Message body:**
```json
{
  "timestamp": "2026-07-29T10:00:00Z",
  "level": "ERROR",
  "service": "auth-service",
  "message": "User authentication failed",
  "user_id": 12345
}
```

**Пример 2: Отправка сообщения в очередь logs.warning**
- **Routing key:** `database.warning`
- **Message body:**
```json
{
  "timestamp": "2026-07-29T10:05:00Z",
  "level": "WARNING",
  "service": "db-service",
  "message": "Connection pool 80% used",
  "pool_size": 20
}
```

**Пример 3: Отправка сообщения в очередь logs.info**
- **Routing key:** `auth.info`
- **Message body:**
```json
{
  "timestamp": "2026-07-29T10:10:00Z",
  "level": "INFO",
  "service": "auth-service",
  "message": "User successfully logged in",
  "user_id": 12345
}
```

**Пример 4: Отправка сообщения в обменник app.events (Direct)**
- **Exchange:** `app.events`
- **Routing key:** `create`
- **Message body:**
```json
{
  "event_type": "USER_CREATED",
  "timestamp": "2026-07-29T10:15:00Z",
  "data": {
    "user_id": 12345,
    "email": "user@example.com",
    "role": "admin"
  }
}
```

**Пример 5: Отправка сообщения в обменник app.notifications (Fanout)**
- **Exchange:** `app.notifications`
- **Routing key:** (можно оставить пустым)
- **Message body:**
```json
{
  "notification_type": "SYSTEM_ALERT",
  "timestamp": "2026-07-29T10:20:00Z",
  "message": "High CPU usage detected on node mq2",
  "severity": "critical"
}
```

### 7.2. Получение сообщений через веб-интерфейс

1. Перейдите в раздел **Queues**
2. Нажмите на имя очереди (например, `logs.error`)
3. В разделе **Get messages**:
   - **Mode:** `Ack mode: automatic` (сообщение будет удалено после получения)
   - **Count:** укажите количество сообщений (например, 10)
   - Нажмите **Get message(s)**

4. Просмотрите полученные сообщения. Обратите внимание на:
   - **exchange:** через какой обменник пришло сообщение
   - **routing_key:** с каким ключом маршрутизации
   - **payload:** содержимое сообщения

### 7.3. Отправка сообщений через командную строку

```bash
# Отправка сообщения в обменник app.logs
sudo rabbitmqctl publish --exchange app.logs --routing-key "error" \
  '{"level":"ERROR","service":"api","message":"Connection timeout"}'

# Отправка сообщения в обменник app.events
sudo rabbitmqctl publish --exchange app.events --routing-key "update" \
  '{"event_type":"ORDER_UPDATED","order_id":1001,"status":"shipped"}'
```

### 7.4. Проверка очередей через командную строку

```bash
# Просмотр всех очередей с количеством сообщений
sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged

# Просмотр всех привязок
sudo rabbitmqctl list_bindings

# Просмотр обменников
sudo rabbitmqctl list_exchanges name type durable
```

---

## Часть 8. Мониторинг и диагностика

### 8.1. Мониторинг через веб-интерфейс

1. **Overview** — общая статистика:
   - Количество сообщений (входящие/исходящие)
   - Использование памяти и диска
   - Статус алертов

2. **Connections** — активные подключения:
   - Проверьте, какие клиенты подключены
   - Состояние подключений (running/idle/blocked)

3. **Channels** — активные каналы:
   - Количество неподтвержденных сообщений
   - Prefetch count

4. **Queues** — детали очередей:
   - `Ready` — сообщения, готовые к обработке
   - `Unacked` — сообщения, взятые в обработку, но не подтвержденные
   - `Consumers` — количество активных потребителей

### 8.2. Мониторинг через командную строку

```bash
# Статус кластера
sudo rabbitmqctl cluster_status

# Список всех очередей с количеством сообщений
sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers

# Список всех пользователей
sudo rabbitmqctl list_users

# Состояние памяти и диска
sudo rabbitmqctl status | grep -A 5 "memory\|disk"

# Логи в реальном времени
sudo tail -f /var/log/rabbitmq/rabbit@$(hostname).log
```

---

## Часть 9. Проверка отказоустойчивости

### 9.1. Тест падения узла

1. **Проверьте статус кластера:**
   ```bash
   sudo rabbitmqctl cluster_status
   ```

2. **Остановите узел mq2:**
   ```bash
   # На узле mq2
   sudo systemctl stop rabbitmq-server
   ```

3. **Проверьте статус кластера с mq1:**
   ```bash
   # На узле mq1
   sudo rabbitmqctl cluster_status
   ```

4. **Проверьте, что очереди доступны:**
   - Зайдите в веб-интерфейс на mq1
   - Проверьте, что очереди видны

5. **Восстановите узел mq2:**
   ```bash
   # На узле mq2
   sudo systemctl start rabbitmq-server
   ```

6. **Проверьте, что узел вернулся в кластер:**
   ```bash
   # На любом узле
   sudo rabbitmqctl cluster_status
   ```

### 9.2. Тест Mirrored Queue

1. **Создайте тестовую очередь с зеркалированием:**
   ```bash
   sudo rabbitmqctl add_queue test_ha --durable
   sudo rabbitmqctl set_policy test-policy "^test_ha$" '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'
   ```

2. **Отправьте тестовое сообщение:**
   ```bash
   sudo rabbitmqctl publish --exchange amq.direct --routing-key "test_ha" "Test HA message"
   ```

3. **Остановите узел, на котором находится Master:**
   ```bash
   # Проверьте, на каком узле находится Master
   sudo rabbitmqctl list_queues name pid | grep test_ha
   
   # Остановите этот узел
   sudo systemctl stop rabbitmq-server
   ```

4. **Проверьте, что сообщение доступно:**
   - Зайдите в веб-интерфейс на другом узле
   - Проверьте очередь test_ha — сообщение должно быть доступно

5. **Восстановите узел:**
   ```bash
   sudo systemctl start rabbitmq-server
   ```

---

## Часть 10. Очистка окружения

### 10.1. Удаление очередей и обменников

```bash
# Удаление очередей
sudo rabbitmqctl delete_queue logs.error
sudo rabbitmqctl delete_queue logs.warning
sudo rabbitmqctl delete_queue logs.info
sudo rabbitmqctl delete_queue events.created
sudo rabbitmqctl delete_queue events.updated
sudo rabbitmqctl delete_queue notifications.email
sudo rabbitmqctl delete_queue notifications.sms
sudo rabbitmqctl delete_queue dead-letter
sudo rabbitmqctl delete_queue test_ha

# Удаление обменников
sudo rabbitmqctl delete_exchange app.logs
sudo rabbitmqctl delete_exchange app.events
sudo rabbitmqctl delete_exchange app.notifications

# Удаление политик
sudo rabbitmqctl clear_policy ha-policy
sudo rabbitmqctl clear_policy test-policy
```

### 10.2. Остановка и удаление RabbitMQ

```bash
# Остановка сервиса
sudo systemctl stop rabbitmq-server

# Удаление пакетов
sudo apt remove --purge rabbitmq-server erlang-*
sudo apt autoremove -y

# Удаление данных
sudo rm -rf /var/lib/rabbitmq
sudo rm -rf /var/log/rabbitmq

# Удаление репозитория
sudo rm -f /etc/apt/sources.list.d/rabbitmq.list
sudo apt update
```




## Приложение A. Полезные команды

### Управление сервисом
```bash
sudo systemctl start rabbitmq-server
sudo systemctl stop rabbitmq-server
sudo systemctl restart rabbitmq-server
sudo systemctl status rabbitmq-server
```

### Управление кластером
```bash
sudo rabbitmqctl cluster_status
sudo rabbitmqctl join_cluster rabbit@mq1
sudo rabbitmqctl stop_app
sudo rabbitmqctl start_app
sudo rabbitmqctl reset
```

### Управление очередями и обменниками
```bash
sudo rabbitmqctl list_queues
sudo rabbitmqctl list_exchanges
sudo rabbitmqctl list_bindings
sudo rabbitmqctl add_queue <name>
sudo rabbitmqctl delete_queue <name>
sudo rabbitmqctl add_exchange <name> <type>
sudo rabbitmqctl delete_exchange <name>
```

### Управление пользователями
```bash
sudo rabbitmqctl add_user <username> <password>
sudo rabbitmqctl delete_user <username>
sudo rabbitmqctl set_user_tags <username> <tag>
sudo rabbitmqctl set_permissions <username> <configure> <write> <read>
sudo rabbitmqctl list_users
```

### Управление политиками
```bash
sudo rabbitmqctl set_policy <name> <pattern> <definition>
sudo rabbitmqctl clear_policy <name>
```

