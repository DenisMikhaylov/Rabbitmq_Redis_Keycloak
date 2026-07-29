# Лабораторная работа: Развертывание и настройка RabbitMQ в Docker

## Создание отказоустойчивого кластера RabbitMQ с использованием Docker

---

## Цель работы

Получить практические навыки по установке, настройке и управлению кластером RabbitMQ с использованием Docker. В ходе выполнения лабораторной работы вы развернете кластер из трех контейнеров, настроите веб-интерфейс, создадите обменники и очереди, настроите маршрутизацию и проверите работу брокера через отправку и получение сообщений.

---

## Требования к окружению

- **Операционная система:** Любая с поддержкой Docker (Linux, macOS, Windows)
- **Docker:** версия 20.10+
- **Docker Compose:** версия 2.0+
- **ОЗУ:** минимум 4 ГБ
- **Дисковое пространство:** минимум 2 ГБ

---

## Топология стенда

```
┌─────────────────────────────────────────────────────────────┐
│                      Лабораторный стенд                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │  Container  │  │  Container  │  │  Container  │       │
│  │   rabbit1   │  │   rabbit2   │  │   rabbit3   │       │
│  │             │  │             │  │             │       │
│  │   Master    │  │   Replica   │  │   Replica   │       │
│  │   :5672     │  │   :5673     │  │   :5674     │       │
│  │   :15672    │  │   :15673    │  │   :15674    │       │
│  └─────────────┘  └─────────────┘  └─────────────┘       │
│         │                │                │               │
│         └────────────────┼────────────────┘               │
│                          │                                 │
│                   Docker Network                           │
│                    rabbitmq-net                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Часть 1. Подготовка Docker окружения

### 1.1. Проверка Docker

```bash
# Проверка версии Docker
docker --version

# Проверка Docker Compose
docker-compose --version

# Проверка, что Docker работает
docker info
```

### 1.2. Создание директории для проекта

```bash
mkdir ~/rabbitmq-lab
cd ~/rabbitmq-lab
```

---

## Часть 2. Создание Docker Compose файла

### 2.1. Создание файла docker-compose.yml

Создайте файл `docker-compose.yml` со следующим содержимым:

```yaml
version: '3.8'

services:
  rabbit1:
    image: rabbitmq:3.13-management
    container_name: rabbit1
    hostname: rabbit1
    environment:
      RABBITMQ_ERLANG_COOKIE: "PVTCJBOMJCEUITZEBGIP"
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_DEFAULT_VHOST: /
    ports:
      - "5672:5672"      # AMQP порт
      - "15672:15672"    # Web UI
      - "25672:25672"    # Кластерный порт
    volumes:
      - rabbit1-data:/var/lib/rabbitmq
      - ./rabbit1.conf:/etc/rabbitmq/rabbitmq.conf
    networks:
      - rabbitmq-net

  rabbit2:
    image: rabbitmq:3.13-management
    container_name: rabbit2
    hostname: rabbit2
    environment:
      RABBITMQ_ERLANG_COOKIE: "PVTCJBOMJCEUITZEBGIP"
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_DEFAULT_VHOST: /
    ports:
      - "5673:5672"
      - "15673:15672"
      - "25673:25672"
    volumes:
      - rabbit2-data:/var/lib/rabbitmq
      - ./rabbit2.conf:/etc/rabbitmq/rabbitmq.conf
    networks:
      - rabbitmq-net

  rabbit3:
    image: rabbitmq:3.13-management
    container_name: rabbit3
    hostname: rabbit3
    environment:
      RABBITMQ_ERLANG_COOKIE: "PVTCJBOMJCEUITZEBGIP"
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_DEFAULT_VHOST: /
    ports:
      - "5674:5672"
      - "15674:15672"
      - "25674:25672"
    volumes:
      - rabbit3-data:/var/lib/rabbitmq
      - ./rabbit3.conf:/etc/rabbitmq/rabbitmq.conf
    networks:
      - rabbitmq-net

volumes:
  rabbit1-data:
  rabbit2-data:
  rabbit3-data:

networks:
  rabbitmq-net:
    driver: bridge
```

### 2.2. Создание конфигурационных файлов

Создайте файл `rabbit1.conf`:

```ini
# Базовые настройки
loopback_users = none

# Настройки кластеризации
cluster_partition_handling = autoheal
cluster_keepalive_interval = 10000

# Настройки памяти и диска
vm_memory_high_watermark.relative = 0.4
disk_free_limit.absolute = 50MB

# Настройки логирования
log.console.level = info
log.console = true
```

Скопируйте этот файл для rabbit2 и rabbit3:

```bash
cp rabbit1.conf rabbit2.conf
cp rabbit1.conf rabbit3.conf
```

---

## Часть 3. Запуск контейнеров

### 3.1. Запуск всех контейнеров

```bash
docker-compose up -d
```

### 3.2. Проверка состояния контейнеров

```bash
docker-compose ps
```

**Ожидаемый результат:**

```
NAME      IMAGE                       STATUS              PORTS
rabbit1   rabbitmq:3.13-management   Up 10 seconds       0.0.0.0:5672->5672/tcp, 0.0.0.0:15672->15672/tcp
rabbit2   rabbitmq:3.13-management   Up 10 seconds       0.0.0.0:5673->5672/tcp, 0.0.0.0:15673->15672/tcp
rabbit3   rabbitmq:3.13-management   Up 10 seconds       0.0.0.0:5674->5672/tcp, 0.0.0.0:15674->15672/tcp
```

### 3.3. Просмотр логов

```bash
# Логи всех контейнеров
docker-compose logs

# Логи конкретного контейнера
docker-compose logs rabbit1

# Логи в реальном времени
docker-compose logs -f
```

---

## Часть 4. Настройка кластера

### 4.1. Вход в контейнеры

```bash
# Вход в контейнер rabbit1
docker exec -it rabbit1 bash

# Вход в контейнер rabbit2
docker exec -it rabbit2 bash

# Вход в контейнер rabbit3
docker exec -it rabbit3 bash
```

### 4.2. Проверка Erlang Cookie

```bash
# На каждом контейнере
cat /var/lib/rabbitmq/.erlang.cookie
```

**Ожидаемый результат:** Во всех контейнерах должен быть одинаковый cookie: `PVTCJBOMJCEUITZEBGIP`

### 4.3. Формирование кластера

**На узле rabbit1 (главный узел):**

```bash
docker exec -it rabbit1 bash
rabbitmqctl cluster_status
# Должен отображаться один узел: rabbit@rabbit1
exit
```

**На узле rabbit2:**

```bash
docker exec -it rabbit2 bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster rabbit@rabbit1
rabbitmqctl start_app
exit
```

**На узле rabbit3:**

```bash
docker exec -it rabbit3 bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster rabbit@rabbit1
rabbitmqctl start_app
exit
```

### 4.4. Проверка кластера

```bash
# Проверка на любом узле
docker exec -it rabbit1 rabbitmqctl cluster_status
```

**Ожидаемый результат:**

```
Cluster status of node rabbit@rabbit1 ...

Basics

Cluster name: rabbit@rabbit1

Disk Nodes

rabbit@rabbit1
rabbit@rabbit2
rabbit@rabbit3

Running Nodes

rabbit@rabbit1
rabbit@rabbit2
rabbit@rabbit3
```

---

## Часть 5. Доступ к веб-интерфейсу

### 5.1. Проверка доступности Web UI

Откройте браузер и перейдите по адресам:

- `http://localhost:15672` (rabbit1)
- `http://localhost:15673` (rabbit2)
- `http://localhost:15674` (rabbit3)

**Логин:** `admin`
**Пароль:** `admin123`

### 5.2. Проверка Management Plugin

```bash
# На каждом узле
docker exec -it rabbit1 rabbitmq-plugins list | grep management
```

---

## Часть 6. Настройка политик высокой доступности

### 6.1. Создание политики через Web UI

1. Перейдите в раздел **Admin → Policies**
2. Нажмите **Add policy**
3. Заполните поля:
   - **Name:** `ha-policy`
   - **Pattern:** `^.*$`
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

### 6.2. Создание политики через командную строку

```bash
docker exec -it rabbit1 rabbitmqctl set_policy ha-policy "^.*$" \
  '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}' \
  --apply-to queues
```

### 6.3. Проверка политики

```bash
docker exec -it rabbit1 rabbitmqctl list_policies
```

---

## Часть 7. Создание обменников и очередей

### 7.1. Создание обменников через Web UI

1. Перейдите в раздел **Exchanges**
2. Нажмите **Add a new exchange**
3. Создайте следующие обменники:

| Имя обменника | Тип | Durability |
|---------------|-----|------------|
| `app.logs` | topic | Durable |
| `app.events` | direct | Durable |
| `app.notifications` | fanout | Durable |

### 7.2. Создание обменников через командную строку

```bash
# Создание обменников на узле rabbit1
docker exec -it rabbit1 rabbitmqctl add_exchange app.logs topic --durable
docker exec -it rabbit1 rabbitmqctl add_exchange app.events direct --durable
docker exec -it rabbit1 rabbitmqctl add_exchange app.notifications fanout --durable
```

### 7.3. Создание очередей через Web UI

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

### 7.4. Создание очередей через командную строку

```bash
# Очереди для логов
docker exec -it rabbit1 rabbitmqctl add_queue logs.error --durable
docker exec -it rabbit1 rabbitmqctl add_queue logs.warning --durable
docker exec -it rabbit1 rabbitmqctl add_queue logs.info --durable

# Очереди для событий
docker exec -it rabbit1 rabbitmqctl add_queue events.created --durable
docker exec -it rabbit1 rabbitmqctl add_queue events.updated --durable

# Очереди для уведомлений
docker exec -it rabbit1 rabbitmqctl add_queue notifications.email --durable
docker exec -it rabbit1 rabbitmqctl add_queue notifications.sms --durable

# Dead Letter очередь
docker exec -it rabbit1 rabbitmqctl add_queue dead-letter --durable
```

### 7.5. Настройка привязок через Web UI

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

### 7.6. Настройка привязок через командную строку

```bash
# Привязки для обменника app.logs
docker exec -it rabbit1 rabbitmqctl add_binding app.logs logs.error '*.error'
docker exec -it rabbit1 rabbitmqctl add_binding app.logs logs.warning '*.warning'
docker exec -it rabbit1 rabbitmqctl add_binding app.logs logs.info '*.info'
docker exec -it rabbit1 rabbitmqctl add_binding app.logs logs.warning '*.critical'

# Привязки для обменника app.events
docker exec -it rabbit1 rabbitmqctl add_binding app.events events.created create
docker exec -it rabbit1 rabbitmqctl add_binding app.events events.updated update

# Привязки для обменника app.notifications (Fanout - ключ игнорируется)
docker exec -it rabbit1 rabbitmqctl add_binding app.notifications notifications.email
docker exec -it rabbit1 rabbitmqctl add_binding app.notifications notifications.sms
```

---

## Часть 8. Отправка и получение сообщений

### 8.1. Отправка сообщений через Web UI

1. Перейдите в раздел **Exchanges**
2. Нажмите на имя обменника `app.logs`
3. В разделе **Publish message** заполните поля:

**Пример 1: Отправка в очередь logs.error**
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

**Пример 2: Отправка в очередь logs.warning**
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

**Пример 3: Отправка в очередь logs.info**
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

**Пример 4: Отправка в обменник app.events**
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

**Пример 5: Отправка в обменник app.notifications**
- **Exchange:** `app.notifications`
- **Routing key:** (оставить пустым)
- **Message body:**
```json
{
  "notification_type": "SYSTEM_ALERT",
  "timestamp": "2026-07-29T10:20:00Z",
  "message": "High CPU usage detected on node rabbit2",
  "severity": "critical"
}
```

### 8.2. Получение сообщений через Web UI

1. Перейдите в раздел **Queues**
2. Нажмите на имя очереди (например, `logs.error`)
3. В разделе **Get messages**:
   - **Mode:** `Ack mode: automatic`
   - **Count:** укажите количество (например, 10)
   - Нажмите **Get message(s)**

### 8.3. Отправка сообщений через командную строку

```bash
# Отправка через rabbit1
docker exec -it rabbit1 rabbitmqctl publish \
  --exchange app.logs --routing-key error \
  '{"level":"ERROR","service":"api","message":"Connection timeout"}'

# Отправка в обменник app.events
docker exec -it rabbit1 rabbitmqctl publish \
  --exchange app.events --routing-key update \
  '{"event_type":"ORDER_UPDATED","order_id":1001,"status":"shipped"}'
```

### 8.4. Просмотр очередей через командную строку

```bash
# Просмотр всех очередей
docker exec -it rabbit1 rabbitmqctl list_queues name messages_ready messages_unacknowledged

# Просмотр всех привязок
docker exec -it rabbit1 rabbitmqctl list_bindings

# Просмотр обменников
docker exec -it rabbit1 rabbitmqctl list_exchanges name type durable
```

---

## Часть 9. Проверка отказоустойчивости

### 9.1. Тест падения узла

**Шаг 1: Проверка статуса кластера**
```bash
docker exec -it rabbit1 rabbitmqctl cluster_status
```

**Шаг 2: Остановка узла rabbit2**
```bash
docker stop rabbit2
```

**Шаг 3: Проверка статуса кластера**
```bash
docker exec -it rabbit1 rabbitmqctl cluster_status
```
Должны отображаться только узлы rabbit1 и rabbit3.

**Шаг 4: Проверка доступности очередей**
- Зайдите в Web UI на `http://localhost:15672`
- Проверьте, что очереди доступны

**Шаг 5: Восстановление узла rabbit2**
```bash
docker start rabbit2
```

**Шаг 6: Проверка, что узел вернулся в кластер**
```bash
docker exec -it rabbit1 rabbitmqctl cluster_status
```

### 9.2. Тест Mirrored Queue

**Шаг 1: Создание тестовой очереди**
```bash
docker exec -it rabbit1 rabbitmqctl add_queue test_ha --durable
docker exec -it rabbit1 rabbitmqctl set_policy test-policy "^test_ha$" \
  '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'
```

**Шаг 2: Отправка тестового сообщения**
```bash
docker exec -it rabbit1 rabbitmqctl publish \
  --exchange amq.direct --routing-key test_ha "Test HA message"
```

**Шаг 3: Проверка расположения Master**
```bash
docker exec -it rabbit1 rabbitmqctl list_queues name pid | grep test_ha
```

**Шаг 4: Остановка узла с Master**
```bash
# Определите, на каком узле Master, и остановите его
docker stop rabbit2  # или rabbit3, в зависимости от результата
```

**Шаг 5: Проверка доступности сообщения**
- Зайдите в Web UI на другом узле
- Проверьте очередь test_ha — сообщение должно быть доступно

**Шаг 6: Восстановление узла**
```bash
docker start rabbit2
```

---

## Часть 10. Мониторинг и диагностика

### 10.1. Мониторинг через Web UI

1. **Overview** — общая статистика:
   - Количество сообщений (входящие/исходящие)
   - Использование памяти и диска
   - Статус алертов

2. **Connections** — активные подключения

3. **Channels** — активные каналы

4. **Queues** — детали очередей

### 10.2. Мониторинг через командную строку

```bash
# Статус кластера
docker exec -it rabbit1 rabbitmqctl cluster_status

# Список очередей
docker exec -it rabbit1 rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers

# Список пользователей
docker exec -it rabbit1 rabbitmqctl list_users

# Состояние памяти и диска
docker exec -it rabbit1 rabbitmqctl status | grep -A 5 "memory\|disk"

# Просмотр логов
docker logs rabbit1
docker logs -f rabbit1
```

### 10.3. Включение Prometheus метрик

```bash
# Включение плагина на всех узлах
docker exec -it rabbit1 rabbitmq-plugins enable rabbitmq_prometheus
docker exec -it rabbit2 rabbitmq-plugins enable rabbitmq_prometheus
docker exec -it rabbit3 rabbitmq-plugins enable rabbitmq_prometheus

# Проверка доступности метрик
curl http://localhost:15692/metrics
```

---

## Часть 11. Очистка окружения

### 11.1. Остановка контейнеров

```bash
docker-compose down
```

### 11.2. Удаление томов с данными

```bash
docker-compose down -v
```

### 11.3. Полная очистка

```bash
docker-compose down -v --rmi all
```

---

## Часть 12. Дополнительные задания

### Задание 1. Настройка Dead Letter Exchange

Настройте DLX для очереди `logs.error`:

```bash
# Удаление существующей очереди
docker exec -it rabbit1 rabbitmqctl delete_queue logs.error

# Создание очереди с DLX
docker exec -it rabbit1 rabbitmqctl add_queue logs.error --durable \
  --arguments '{"x-dead-letter-exchange":"app.logs","x-dead-letter-routing-key":"error.dead"}'

# Создание привязки
docker exec -it rabbit1 rabbitmqctl add_binding app.logs logs.error '*.error'
```

### Задание 2. Создание очереди с TTL

```bash
docker exec -it rabbit1 rabbitmqctl add_queue ttl_queue --durable \
  --arguments '{"x-message-ttl":30000}'
```

### Задание 3. Настройка пользовательских прав

```bash
# Создание пользователя
docker exec -it rabbit1 rabbitmqctl add_user app_user SecurePass123

# Назначение прав
docker exec -it rabbit1 rabbitmqctl set_permissions -p / app_user "^$" "app.logs" "^$"

# Проверка
docker exec -it rabbit1 rabbitmqctl list_user_permissions app_user
```

### Задание 4. Масштабирование кластера

Добавьте четвертый узел в кластер:

1. Добавьте сервис в docker-compose.yml:

```yaml
rabbit4:
  image: rabbitmq:3.13-management
  container_name: rabbit4
  hostname: rabbit4
  environment:
    RABBITMQ_ERLANG_COOKIE: "PVTCJBOMJCEUITZEBGIP"
    RABBITMQ_DEFAULT_USER: admin
    RABBITMQ_DEFAULT_PASS: admin123
    RABBITMQ_DEFAULT_VHOST: /
  ports:
    - "5675:5672"
    - "15675:15672"
    - "25675:25672"
  volumes:
    - rabbit4-data:/var/lib/rabbitmq
  networks:
    - rabbitmq-net
```

2. Добавьте том:

```yaml
volumes:
  rabbit4-data:
```

3. Запустите и присоедините к кластеру:

```bash
docker-compose up -d rabbit4
docker exec -it rabbit4 rabbitmqctl stop_app
docker exec -it rabbit4 rabbitmqctl reset
docker exec -it rabbit4 rabbitmqctl join_cluster rabbit@rabbit1
docker exec -it rabbit4 rabbitmqctl start_app
```

---

## Приложение A. Полезные Docker команды

### Управление контейнерами

```bash
# Запуск всех контейнеров
docker-compose up -d

# Остановка всех контейнеров
docker-compose down

# Перезапуск контейнера
docker-compose restart rabbit1

# Просмотр логов
docker-compose logs -f rabbit1
```

### Вход в контейнер

```bash
docker exec -it rabbit1 bash
```

### Просмотр информации о контейнере

```bash
docker inspect rabbit1
docker stats rabbit1
```

### Копирование файлов

```bash
# Копирование из контейнера
docker cp rabbit1:/var/lib/rabbitmq/.erlang.cookie .

# Копирование в контейнер
docker cp rabbitmq.conf rabbit1:/etc/rabbitmq/
```

