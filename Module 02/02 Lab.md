# Лабораторная работа: Исследование механизмов персистентности Redis

## Продолжительность: 45 минут

---

## Цель работы

Изучить механизмы персистентности Redis (RDB, AOF, гибридный режим) на практике, сравнить их влияние на производительность и надежность хранения данных.

---

## Подготовка

Для выполнения работы используется инфраструктура, созданная в предыдущей лабораторной работе (репликация с 3 узлами).

**Убедитесь, что у вас есть:**
- Docker и Docker Compose установлены
- Файл `docker-compose-replication.yml` из предыдущей работы

Если лабораторная среда не сохранилась, создайте новый файл `docker-compose-persistence.yml`:

```yaml
version: "3.8"

services:
  redis-master:
    image: redis:7.2-alpine
    container_name: redis-master
    command: redis-server --port 7001 --appendonly yes
    ports:
      - "7001:7001"
    volumes:
      - redis_data_master:/data
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
    volumes:
      - redis_data_replica1:/data
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
    volumes:
      - redis_data_replica2:/data
    networks:
      - redis-network
    depends_on:
      - redis-master

networks:
  redis-network:
    driver: bridge

volumes:
  redis_data_master:
  redis_data_replica1:
  redis_data_replica2:
```

---

## Часть 1. RDB (Snapshot) — Режим снимков (15 минут)

### 1.1. Теоретическая справка

RDB (Redis Database) — механизм создания снимков состояния базы данных в бинарный файл (dump.rdb). Настройка через параметры save в redis.conf.

**Ключевые параметры:**
- `save <seconds> <changes>` — условие создания снимка
- `stop-writes-on-bgsave-error` — остановка записи при ошибке создания снимка
- `rdbcompression` — сжатие данных (по умолчанию yes)
- `rdbchecksum` — контрольная сумма (по умолчанию yes)

### 1.2. Запуск с различными настройками RDB

#### Эксперимент 1.1. RDB по умолчанию

**Запустите мастер с базовыми настройками RDB:**
```bash
docker-compose -f docker-compose-persistence.yml up -d
```

**Проверьте настройки RDB:**
```bash
docker exec -it redis-master redis-cli -p 7001 CONFIG GET save
```

Ожидаемый результат:
```
1) "save"
2) "3600 1 300 100 60 10000"
```
Это означает:
- 3600 1 — сохранять каждые 3600 сек, если изменился хотя бы 1 ключ
- 300 100 — сохранять каждые 300 сек, если изменилось 100 ключей
- 60 10000 — сохранять каждые 60 сек, если изменилось 10000 ключей

**Создайте тестовые данные:**
```bash
docker exec -it redis-master redis-cli -p 7001
```

В CLI:
```bash
SET test:1 "value1"
SET test:2 "value2"
SAVE
EXIT
```

**Проверьте создание файла dump.rdb:**
```bash
docker exec -it redis-master ls -la /data/
```

#### Эксперимент 1.2. Изменение условий RDB

**Перезапустите мастер с кастомными настройками RDB:**
```bash
docker stop redis-master
docker rm redis-master
docker run -d --name redis-master \
  --network redis-network \
  -v redis_data_master:/data \
  -p 7001:7001 \
  redis:7.2-alpine \
  redis-server --port 7001 \
  --save "10 1" \
  --save "60 100" \
  --rdbcompression yes
```

**Проверьте новые настройки:**
```bash
docker exec -it redis-master redis-cli -p 7001 CONFIG GET save
```

**Запишите данные и подождите 10 секунд:**
```bash
docker exec -it redis-master redis-cli -p 7001 SET test:3 "will_be_saved_after_10sec"
sleep 12
docker exec -it redis-master redis-cli -p 7001 CONFIG GET save
```

**Проверьте содержимое dump.rdb:**
```bash
docker exec -it redis-master ls -la /data/dump.rdb
docker exec -it redis-master cat /data/dump.rdb | head -c 100
```

#### Эксперимент 1.3. Нагрузочное тестирование RDB

**Создайте скрипт для записи данных:**
```bash
cat > load_rdb.sh << 'EOF'
#!/bin/bash
for i in {1..10000}; do
  docker exec -it redis-master redis-cli -p 7001 SET key:$i "value_with_data_$i"
  if [ $((i % 1000)) -eq 0 ]; then
    echo "Inserted $i keys"
  fi
done
echo "Done!"
EOF
chmod +x load_rdb.sh
./load_rdb.sh
```

**Проверьте размер файла:**
```bash
docker exec -it redis-master ls -lh /data/dump.rdb
```

### 1.3. Симуляция аварии и восстановление из RDB

**Остановите мастер принудительно (симуляция сбоя):**
```bash
docker stop redis-master
```

**Проверьте, что данные в RDB сохранены:**
```bash
docker run --rm -v redis_data_master:/data alpine ls -lh /data/dump.rdb
```

**Запустите новый мастер и проверьте данные:**
```bash
docker run -d --name redis-master-new \
  --network redis-network \
  -v redis_data_master:/data \
  -p 7001:7001 \
  redis:7.2-alpine \
  redis-server --port 7001

sleep 5
docker exec -it redis-master-new redis-cli -p 7001 GET key:1000
# Ожидаемый ответ: "value_with_data_1000"
```

**Сравните количество ключей:**
```bash
docker exec -it redis-master-new redis-cli -p 7001 DBSIZE
```

### 1.4. Выводы по RDB

**Заполните таблицу:**

| Параметр | Значение по умолчанию | Ваше значение |
|----------|----------------------|---------------|
| save (условия) | | |
| rdbcompression | yes | |
| rdbchecksum | yes | |
| Размер dump.rdb (байт) | | |
| Время создания снимка | | |

---

## Часть 2. AOF (Append Only File) — Журнал операций (15 минут)

### 2.1. Теоретическая справка

AOF — механизм, записывающий каждую команду записи в лог-файл.

**Ключевые параметры:**
- `appendonly` — включение AOF (yes/no)
- `appendfilename` — имя файла (appendonly.aof)
- `appendfsync` — режим синхронизации:
  - `always` — синхронизация после каждой команды
  - `everysec` — синхронизация раз в секунду (по умолчанию)
  - `no` — синхронизация по усмотрению ОС
- `auto-aof-rewrite-percentage` — процент роста для перезаписи
- `auto-aof-rewrite-min-size` — минимальный размер для перезаписи
- `aof-use-rdb-preamble` — гибридный режим (Redis 4.0+)

### 2.2. Запуск с AOF

**Остановите предыдущие контейнеры:**
```bash
docker stop redis-master-new
docker rm redis-master-new
```

**Запустите мастер с AOF:**
```bash
docker run -d --name redis-master-aof \
  --network redis-network \
  -v redis_data_master:/data \
  -p 7001:7001 \
  redis:7.2-alpine \
  redis-server --port 7001 \
  --appendonly yes \
  --appendfilename appendonly.aof \
  --appendfsync everysec
```

**Проверьте настройки AOF:**
```bash
docker exec -it redis-master-aof redis-cli -p 7001 CONFIG GET appendonly
docker exec -it redis-master-aof redis-cli -p 7001 CONFIG GET appendfsync
```

### 2.3. Эксперименты с разными режимами синхронизации

#### Эксперимент 2.1. Режим everysec (по умолчанию)

**Создайте тестовые данные:**
```bash
for i in {1..100}; do
  docker exec -it redis-master-aof redis-cli -p 7001 SET aof:key:$i "data_$i"
done
```

**Проверьте содержимое AOF-файла:**
```bash
docker exec -it redis-master-aof cat /data/appendonly.aof | head -20
```

**Примечание:** AOF-файл содержит команды в текстовом формате.

#### Эксперимент 2.2. Режим always (максимальная надежность)

**Остановите контейнер:**
```bash
docker stop redis-master-aof
docker rm redis-master-aof
```

**Запустите с appendfsync always:**
```bash
docker run -d --name redis-master-aof-always \
  --network redis-network \
  -v redis_data_master:/data \
  -p 7001:7001 \
  redis:7.2-alpine \
  redis-server --port 7001 \
  --appendonly yes \
  --appendfsync always
```

**Проверьте производительность:**
```bash
time for i in {1..100}; do
  docker exec -it redis-master-aof-always redis-cli -p 7001 SET fast:key:$i "fast_data_$i"
done
```

#### Эксперимент 2.3. Режим no (максимальная производительность)

**Запустите с appendfsync no:**
```bash
docker run -d --name redis-master-aof-no \
  --network redis-network \
  -v redis_data_master:/data \
  -p 7001:7001 \
  redis:7.2-alpine \
  redis-server --port 7001 \
  --appendonly yes \
  --appendfsync no
```

**Сравните время выполнения:**
```bash
time for i in {1..100}; do
  docker exec -it redis-master-aof-no redis-cli -p 7001 SET no:key:$i "no_data_$i"
done
```

### 2.4. Симуляция аварии и восстановление из AOF

**Создайте данные:**
```bash
docker exec -it redis-master-aof redis-cli -p 7001 SET user:a "Alice"
docker exec -it redis-master-aof redis-cli -p 7001 SET user:b "Bob"
docker exec -it redis-master-aof redis-cli -p 7001 INCR counter
```

**Принудительно остановите:**
```bash
docker stop redis-master-aof
```

**Запустите новый экземпляр с теми же данными:**
```bash
docker run -d --name redis-restored \
  --network redis-network \
  -v redis_data_master:/data \
  -p 7001:7001 \
  redis:7.2-alpine \
  redis-server --port 7001 \
  --appendonly yes
```

**Проверьте восстановление:**
```bash
docker exec -it redis-restored redis-cli -p 7001 GET user:a
docker exec -it redis-restored redis-cli -p 7001 GET counter
```

### 2.5. Перезапись AOF (Rewrite)

**Создайте много данных:**
```bash
for i in {1..500}; do
  docker exec -it redis-restored redis-cli -p 7001 SET rewrite:key:$i "value_$i"
done
```

**Проверьте размер AOF:**
```bash
docker exec -it redis-restored ls -lh /data/appendonly.aof
```

**Запустите перезапись вручную:**
```bash
docker exec -it redis-restored redis-cli -p 7001 BGREWRITEAOF
```

**Проверьте размер после перезаписи:**
```bash
sleep 5
docker exec -it redis-restored ls -lh /data/appendonly.aof
```

### 2.6. Повреждение AOF и восстановление

**Имитация повреждения:**
```bash
# Добавьте поврежденные данные в AOF
docker exec -it redis-restored sh -c "echo 'corrupted data' >> /data/appendonly.aof"
docker exec -it redis-restored cat /data/appendonly.aof | tail -5
```

**Попытка запуска с поврежденным AOF:**
```bash
docker stop redis-restored
docker start redis-restored
docker logs redis-restored | tail -10
# Ожидаемая ошибка: Bad file format reading the append only file
```

**Восстановление AOF:**
```bash
docker run --rm -v redis_data_master:/data redis:7.2-alpine redis-check-aof --fix /data/appendonly.aof -y
```

**Перезапуск:**
```bash
docker start redis-restored
docker exec -it redis-restored redis-cli -p 7001 GET rewrite:key:1
```

---

## Часть 3. Гибридный режим (RDB + AOF) (10 минут)

### 3.1. Теоретическая справка

Гибридный режим (Redis 4.0+) использует RDB как базовый снимок и AOF для инкрементальных изменений.

**Параметр:** `aof-use-rdb-preamble yes`

### 3.2. Эксперимент с гибридным режимом

**Запустите мастер в гибридном режиме:**
```bash
docker run -d --name redis-master-hybrid \
  --network redis-network \
  -v redis_data_hybrid:/data \
  -p 7001:7001 \
  redis:7.2-alpine \
  redis-server --port 7001 \
  --appendonly yes \
  --aof-use-rdb-preamble yes
```

**Создайте данные:**
```bash
for i in {1..1000}; do
  docker exec -it redis-master-hybrid redis-cli -p 7001 SET hybrid:key:$i "hybrid_data_$i"
done
```

**Запустите перезапись:**
```bash
docker exec -it redis-master-hybrid redis-cli -p 7001 BGREWRITEAOF
sleep 5
```

**Проверьте структуру файла:**
```bash
docker exec -it redis-master-hybrid cat /data/appendonly.aof | head -20
```

### 3.3. Сравнение размеров файлов

**Создайте таблицу сравнения:**

| Режим | Количество ключей | Размер файла | Время восстановления | Потеря данных при сбое |
|-------|-------------------|--------------|----------------------|------------------------|
| RDB только | 10000 | | | |
| AOF (everysec) | 10000 | | | |
| AOF (always) | 10000 | | | |
| Гибридный | 10000 | | | |

---

## Часть 4. Настройка персистентности в кластере (5 минут)

### 4.1. Обновление docker-compose для кластера

Создайте файл `docker-compose-cluster-persist.yml` с гибридным режимом:

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
      --aof-use-rdb-preamble yes
      --appendfsync everysec
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

  # ... (остальные узлы аналогично)
```

### 4.2. Проверка персистентности в кластере

```bash
docker-compose -f docker-compose-cluster-persist.yml up -d
```

---

## Часть 5. Сравнительный анализ (5 минут)

### 5.1. Тест на производительность

**Создайте скрипт сравнения:**
```bash
cat > compare_persist.sh << 'EOF'
#!/bin/bash
echo "=== Testing RDB only ==="
docker run -d --name test-rdb redis:7.2-alpine redis-server --save "1 1"
time for i in {1..1000}; do docker exec test-rdb redis-cli SET key:$i $i; done
docker stop test-rdb && docker rm test-rdb

echo "=== Testing AOF everysec ==="
docker run -d --name test-aof redis:7.2-alpine redis-server --appendonly yes --appendfsync everysec
time for i in {1..1000}; do docker exec test-aof redis-cli SET key:$i $i; done
docker stop test-aof && docker rm test-aof

echo "=== Testing AOF always ==="
docker run -d --name test-aof-always redis:7.2-alpine redis-server --appendonly yes --appendfsync always
time for i in {1..1000}; do docker exec test-aof-always redis-cli SET key:$i $i; done
docker stop test-aof-always && docker rm test-aof-always

echo "=== Testing Hybrid ==="
docker run -d --name test-hybrid redis:7.2-alpine redis-server --appendonly yes --aof-use-rdb-preamble yes
time for i in {1..1000}; do docker exec test-hybrid redis-cli SET key:$i $i; done
docker stop test-hybrid && docker rm test-hybrid
EOF
chmod +x compare_persist.sh
./compare_persist.sh
```

### 5.2. Таблица сравнения режимов

**Заполните итоговую таблицу:**

| Критерий | RDB | AOF (everysec) | AOF (always) | Гибридный |
|----------|-----|----------------|--------------|-----------|
| Скорость записи | Быстрая | Средняя | Медленная | Высокая |
| Потеря данных при сбое | До 60 сек | До 1 сек | 0 сек | Минимальная |
| Размер файла | Маленький | Большой | Большой | Средний |
| Скорость восстановления | Быстрая | Медленная | Медленная | Быстрая |
| Рекомендуемое использование | Бэкапы | Продакшен | Финансовые системы | Современные системы |

---

## Часть 6. Контрольные вопросы

1. В чем принципиальное отличие RDB от AOF?
2. Какой режим синхронизации AOF обеспечивает максимальную надежность? Какой — максимальную производительность?
3. Как работает гибридный режим и в чем его преимущество?
4. Почему при включенном AOF Redis не использует RDB для восстановления?
5. Как выполнить ручную перезапись AOF и зачем это нужно?
6. Что происходит с данными при повреждении AOF-файла?
7. Как настроить автоматическую перезапись AOF?

---

## Оформление отчета

1. **Титульный лист** (название работы, ФИО, группа)
2. **Цель работы**
3. **Теоретическая часть** (краткое описание механизмов)
4. **Выполнение лабораторной работы** (все команды с пояснениями)
5. **Результаты экспериментов** (скриншоты, таблицы, графики)
6. **Выводы** (сравнение режимов, рекомендации по применению)
7. **Ответы на контрольные вопросы**

---

## Критерии оценки

| Критерий | Баллы |
|----------|-------|
| Запуск всех режимов персистентности | 20 |
| Корректное выполнение экспериментов | 20 |
| Сравнительный анализ (таблицы) | 20 |
| Восстановление после аварии | 20 |
| Выводы и рекомендации | 10 |
| Ответы на контрольные вопросы | 10 |
| **Всего** | **100** |

---

## Приложение: Команды для справки

### Настройка параметров персистентности

```bash
# Просмотр всех настроек
CONFIG GET *

# Настройка RDB
CONFIG SET save "300 1 60 10"

# Настройка AOF
CONFIG SET appendonly yes
CONFIG SET appendfsync everysec

# Настройка гибридного режима
CONFIG SET aof-use-rdb-preamble yes

# Запуск перезаписи AOF
BGREWRITEAOF

# Создание снимка RDB
SAVE
BGSAVE

# Просмотр времени последнего сохранения
LASTSAVE
```

### Проверка файлов

```bash
# Просмотр файлов в data
docker exec -it <container> ls -la /data/

# Просмотр содержимого AOF
docker exec -it <container> cat /data/appendonly.aof | head -50

# Проверка целостности AOF
docker run --rm -v <volume>:/data redis:7.2-alpine redis-check-aof /data/appendonly.aof

# Восстановление AOF
docker run --rm -v <volume>:/data redis:7.2-alpine redis-check-aof --fix /data/appendonly.aof -y
```

### Очистка данных после экспериментов

```bash
# Удаление томов
docker volume ls -qf "name=redis_data*" | xargs docker volume rm

# Остановка всех контейнеров
docker stop $(docker ps -q --filter "name=redis*")
docker rm $(docker ps -aq --filter "name=redis*")
```
