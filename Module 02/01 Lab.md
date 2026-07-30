# Лабораторная работа: Развертывание Redis в Docker, репликация и кластер

---

## Цель работы

Научиться разворачивать Redis в среде Docker, настраивать репликацию Master-Replica и создавать отказоустойчивый Redis Cluster.

---

## Часть 1. Базовое развертывание Redis в Docker (15 минут)

### 1.1. Запуск одиночного экземпляра Redis

**Команда:**
```bash
docker run -d --name redis-single -p 6379:6379 redis:7.2-alpine
```

**Проверка:**
```bash
docker ps
docker exec -it redis-single redis-cli ping
# Ожидаемый ответ: PONG
```

**Подключение к CLI:**
```bash
docker exec -it redis-single redis-cli
```

**Тестовые команды:**
```bash
SET test "Hello Redis"
GET test
EXIT
```

### 1.2. Запуск с сохранением данных (persistent volume)

```bash
docker run -d --name redis-persistent \
  -v redis_data:/data \
  -p 6379:6379 \
  redis:7.2-alpine redis-server --appendonly yes
```

**Проверка:** Запишите данные, перезапустите контейнер и убедитесь, что они сохранились.

```bash
docker exec -it redis-persistent redis-cli SET user:1 "Ivan Petrov"
docker restart redis-persistent
docker exec -it redis-persistent redis-cli GET user:1
```

---

## Часть 2. Репликация Master-Replica (25 минут)

### 2.1. Архитектура

Мы создадим 3 контейнера Redis:
- **Master** — порт 7001
- **Replica 1** — порт 7002
- **Replica 2** — порт 7003

### 2.2. Файл docker-compose.yml

Создайте файл `docker-compose-replication.yml` со следующим содержимым:

```yaml
version: "3.8"

services:
  redis-master:
    image: redis:7.2-alpine
    container_name: redis-master
    command: redis-server --port 7001 --appendonly yes
    ports:
      - "7001:7001"
    networks:
      - redis-network

  redis-replica-1:
    image: redis:7.2-alpine
    container_name: redis-replica-1
    command: >
      redis-server --port 7002 --appendonly yes
      --replicaof redis-master 7001
    ports:
      - "7002:7002"
    networks:
      - redis-network
    depends_on:
      - redis-master

  redis-replica-2:
    image: redis:7.2-alpine
    container_name: redis-replica-2
    command: >
      redis-server --port 7003 --appendonly yes
      --replicaof redis-master 7001
    ports:
      - "7003:7003"
    networks:
      - redis-network
    depends_on:
      - redis-master

networks:
  redis-network:
    driver: bridge
```

### 2.3. Запуск кластера репликации

```bash
docker-compose -f docker-compose-replication.yml up -d
```

**Проверка статуса контейнеров:**
```bash
docker-compose -f docker-compose-replication.yml ps
```

### 2.4. Проверка статуса репликации

**Подключитесь к мастеру:**
```bash
docker exec -it redis-master redis-cli -p 7001
```

**Выполните:**
```bash
INFO replication
```

Ожидаемый результат:
```
# Replication
role:master
connected_slaves:2
slave0:ip=172.x.x.x,port=7002,state=online,offset=...
slave1:ip=172.x.x.x,port=7003,state=online,offset=...
```

**Подключитесь к реплике 1:**
```bash
docker exec -it redis-replica-1 redis-cli -p 7002
INFO replication
```

Ожидаемый результат:
```
# Replication
role:slave
master_host:redis-master
master_port:7001
```

### 2.5. Тестирование репликации

**На мастере:**
```bash
docker exec -it redis-master redis-cli -p 7001 SET test_key "Master data"
docker exec -it redis-master redis-cli -p 7001 GET test_key
# Ожидаемый ответ: "Master data"
```

**На реплике 1:**
```bash
docker exec -it redis-replica-1 redis-cli -p 7002 GET test_key
# Ожидаемый ответ: "Master data"
```

**Проверка ограничения записи на реплике:**
```bash
docker exec -it redis-replica-1 redis-cli -p 7002 SET new_key "test"
# Ожидаемый ответ: (error) READONLY You can't write against a read only replica.
```

### 2.6. Ручное переключение (Failover)

**Остановите мастер:**
```bash
docker stop redis-master
```

**На реплике 1 выполните повышение до мастера:**
```bash
docker exec -it redis-replica-1 redis-cli -p 7002 REPLICAOF NO ONE
```

**Проверьте статус:**
```bash
docker exec -it redis-replica-1 redis-cli -p 7002 INFO replication
# role:master
```

**Восстановите старый мастер как реплику нового:**
```bash
docker start redis-master
docker exec -it redis-master redis-cli -p 7001 REPLICAOF redis-replica-1 7002
```

### 2.7. Очистка

```bash
docker-compose -f docker-compose-replication.yml down
docker volume prune -f
```

---

## Часть 3. Redis Cluster (40 минут)

### 3.1. Архитектура

Создадим Redis Cluster из 6 узлов:
- 3 мастер-ноды (распределение слотов 0-16383)
- 3 реплики (по одной на каждый мастер)

**Порты:** 7001-7006

### 3.2. Файл docker-compose-cluster.yml

Создайте файл со следующим содержимым:

```yaml
version: "3.8"

services:
  redis-7001:
    image: redis:7.2-alpine
    container_name: redis-7001
    command: >
      redis-server
      --port 7001
      --cluster-enabled yes
      --cluster-config-file /data/nodes-7001.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --dir /data
      --bind 0.0.0.0
      --protected-mode no
    ports:
      - "7001:7001"
      - "17001:17001"
    volumes:
      - redis_data_7001:/data
    networks:
      cluster_network:
        ipv4_address: 172.20.0.11

  redis-7002:
    image: redis:7.2-alpine
    container_name: redis-7002
    command: >
      redis-server
      --port 7002
      --cluster-enabled yes
      --cluster-config-file /data/nodes-7002.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --dir /data
      --bind 0.0.0.0
      --protected-mode no
    ports:
      - "7002:7002"
      - "17002:17002"
    volumes:
      - redis_data_7002:/data
    networks:
      cluster_network:
        ipv4_address: 172.20.0.12

  redis-7003:
    image: redis:7.2-alpine
    container_name: redis-7003
    command: >
      redis-server
      --port 7003
      --cluster-enabled yes
      --cluster-config-file /data/nodes-7003.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --dir /data
      --bind 0.0.0.0
      --protected-mode no
    ports:
      - "7003:7003"
      - "17003:17003"
    volumes:
      - redis_data_7003:/data
    networks:
      cluster_network:
        ipv4_address: 172.20.0.13

  redis-7004:
    image: redis:7.2-alpine
    container_name: redis-7004
    command: >
      redis-server
      --port 7004
      --cluster-enabled yes
      --cluster-config-file /data/nodes-7004.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --dir /data
      --bind 0.0.0.0
      --protected-mode no
    ports:
      - "7004:7004"
      - "17004:17004"
    volumes:
      - redis_data_7004:/data
    networks:
      cluster_network:
        ipv4_address: 172.20.0.14

  redis-7005:
    image: redis:7.2-alpine
    container_name: redis-7005
    command: >
      redis-server
      --port 7005
      --cluster-enabled yes
      --cluster-config-file /data/nodes-7005.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --dir /data
      --bind 0.0.0.0
      --protected-mode no
    ports:
      - "7005:7005"
      - "17005:17005"
    volumes:
      - redis_data_7005:/data
    networks:
      cluster_network:
        ipv4_address: 172.20.0.15

  redis-7006:
    image: redis:7.2-alpine
    container_name: redis-7006
    command: >
      redis-server
      --port 7006
      --cluster-enabled yes
      --cluster-config-file /data/nodes-7006.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --dir /data
      --bind 0.0.0.0
      --protected-mode no
    ports:
      - "7006:7006"
      - "17006:17006"
    volumes:
      - redis_data_7006:/data
    networks:
      cluster_network:
        ipv4_address: 172.20.0.16

  cluster-creator:
    image: redis:7.2-alpine
    container_name: cluster-creator
    depends_on:
      - redis-7001
      - redis-7002
      - redis-7003
      - redis-7004
      - redis-7005
      - redis-7006
    entrypoint:
      - sh
      - -c
      - |
        echo "Waiting for Redis nodes..."
        sleep 10
        
        echo "Creating Redis Cluster..."
        echo yes | redis-cli --cluster create \
          redis-7001:7001 redis-7002:7002 redis-7003:7003 \
          redis-7004:7004 redis-7005:7005 redis-7006:7006 \
          --cluster-replicas 1
        
        echo "Cluster created successfully!"
        tail -f /dev/null
    networks:
      cluster_network:
        ipv4_address: 172.20.0.10

networks:
  cluster_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

volumes:
  redis_data_7001:
  redis_data_7002:
  redis_data_7003:
  redis_data_7004:
  redis_data_7005:
  redis_data_7006:
```

### 3.3. Запуск кластера

```bash
docker-compose -f docker-compose-cluster.yml up -d
```

**Мониторинг создания кластера:**
```bash
docker logs -f cluster-creator
```

Успешное создание кластера подтверждается сообщением:
```
[OK] All 16384 slots covered.
```

### 3.4. Проверка состояния кластера

**Информация о кластере:**
```bash
docker exec -it redis-7001 redis-cli -p 7001 cluster info
```

Ожидаемый результат:
```
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
```

**Список узлов:**
```bash
docker exec -it redis-7001 redis-cli -p 7001 cluster nodes
```

### 3.5. Тестирование работы с кластером

**Подключение с поддержкой кластера:**
```bash
docker exec -it redis-7001 redis-cli -c -p 7001
```

**Запись и чтение данных:**
```bash
SET user:1 "Ivan"
GET user:1

SET user:2 "Petr"
GET user:2
```

Redis автоматически перенаправит запрос к нужному узлу, если ключ принадлежит другому слоту.

### 3.6. Хеш-теги для многоключевых операций

**Запись в один слот с помощью хеш-тега:**
```bash
SET {user:1}:name "Ivan"
SET {user:1}:age "30"
MGET {user:1}:name {user:1}:age
# Ожидаемый ответ: "Ivan" "30"
```

**Попытка MGET из разных слотов:**
```bash
SET user:1:name "Ivan"
SET user:2:name "Petr"
MGET user:1:name user:2:name
# Ожидаемый ответ: (error) CROSSSLOT Keys in request don't hash to the same slot
```

### 3.7. Проверка отказоустойчивости (Failover)

**Определите, какой узел является мастером:**
```bash
docker exec -it redis-7001 redis-cli -p 7001 cluster nodes
```

**Остановите мастер-узел (например, redis-7001):**
```bash
docker stop redis-7001
```

**Через 10-15 секунд проверьте состояние кластера:**
```bash
docker exec -it redis-7002 redis-cli -p 7002 cluster nodes
```

Одна из реплик должна стать мастером (role:master).

**Восстановите остановленный узел:**
```bash
docker start redis-7001
```

**Проверьте, что узел стал репликой:**
```bash
docker exec -it redis-7001 redis-cli -p 7001 INFO replication
```

### 3.8. Управление кластером

**Проверка целостности кластера:**
```bash
docker exec -it redis-7001 redis-cli -p 7001 --cluster check redis-7001:7001
```

**Получение информации о слотах:**
```bash
docker exec -it redis-7001 redis-cli -p 7001 cluster slots
```

### 3.9. Очистка

```bash
docker-compose -f docker-compose-cluster.yml down -v
```

---

## Часть 4. Итоговое задание (10 минут)

### Задание 4.1. Проверка знаний

1. Как проверить статус репликации на мастер-узле?
   ```bash
   docker exec -it redis-master redis-cli -p 7001 INFO replication
   ```

2. Какая команда делает узел репликой другого узла?
   ```bash
   REPLICAOF <master_host> <master_port>
   ```

3. Как инициализировать Redis Cluster с 6 узлами?
   ```bash
   redis-cli --cluster create node1:port1 node2:port2 ... --cluster-replicas 1
   ```

4. Как проверить состояние кластера?
   ```bash
   redis-cli -p 7001 cluster info
   ```

### Задание 4.2. Самостоятельная практика

1. Добавьте еще один мастер-узел в кластер и перераспределите слоты (reshard)
2. Настройте аутентификацию (пароль) для кластера
3. Подключитесь к кластеру через Python (библиотека redis-py) и выполните операции

---

## Приложение: Полезные команды

### Docker команды

| Команда | Описание |
|---------|----------|
| `docker ps` | Список запущенных контейнеров |
| `docker logs <container>` | Логи контейнера |
| `docker exec -it <container> sh` | Вход в контейнер |
| `docker stop <container>` | Остановка контейнера |
| `docker start <container>` | Запуск контейнера |
| `docker rm <container>` | Удаление контейнера |
| `docker volume ls` | Список томов |

### Redis CLI команды

| Команда | Описание |
|---------|----------|
| `INFO replication` | Статус репликации |
| `INFO cluster` | Статус кластера |
| `CLUSTER NODES` | Список узлов кластера |
| `CLUSTER INFO` | Информация о кластере |
| `REPLICAOF host port` | Назначить мастером |
| `REPLICAOF NO ONE` | Стать мастером |
| `CLUSTER SLOTS` | Распределение слотов |

