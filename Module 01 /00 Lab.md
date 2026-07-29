
## Модуль 0: Установка Docker на Debian 13 (Trixie)

**Цель модуля:** Подготовить три виртуальные машины с Debian 13 для работы с Docker и Docker Compose.

**Важно:** Debian 13 (Trixie) имеет особенности при установке Docker. Использование стандартного пакета `docker.io` из репозитория Debian может дать устаревшую версию Docker Buildx, что приводит к проблемам с некоторыми образами . Рекомендуется устанавливать Docker из официального репозитория Docker .

---

### 0.1. Подготовка системы

На каждом из трёх узлов выполните следующие команды для обновления системы и установки базовых утилит:

```bash
# Обновление списка пакетов
sudo apt update

# Обновление установленных пакетов
sudo apt upgrade -y

# Установка необходимых утилит (wget, curl, etc.)
sudo apt install -y wget curl gnupg2 lsb-release ca-certificates
```

---

### 0.2. Удаление старых версий Docker

Перед установкой убедитесь, что на системе нет конфликтующих пакетов :

```bash
# Удаление старых пакетов Docker (если они есть)
sudo apt remove -y docker.io docker-doc docker-compose podman-docker containerd runc
```

---

### 0.3. Добавление официального репозитория Docker

Для получения актуальной версии Docker добавим официальный репозиторий :

```bash
# Создание директории для ключей
sudo install -m 0755 -d /etc/apt/keyrings

# Загрузка GPG-ключа Docker
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Добавление репозитория Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Обновление списка пакетов с новым репозиторием
sudo apt update
```

---

### 0.4. Установка Docker Engine и Docker Compose

Установите полный набор компонентов Docker: сам движок, CLI-клиент, containerd, плагины для сборки образов и Docker Compose :

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

### 0.5. Проверка установки

Проверьте версии установленных компонентов:

```bash
# Проверка версии Docker
docker --version

# Проверка версии Docker Compose (плагин)
docker compose version
```

Запустите тестовый контейнер для проверки работоспособности :

```bash
sudo docker run hello-world
```

Если вы видите сообщение `Hello from Docker!`, установка выполнена успешно.

---

### 0.6. Настройка пользователя для работы с Docker

По умолчанию для выполнения команд Docker требуются права `sudo`. Чтобы этого избежать, добавьте текущего пользователя в группу `docker`:

```bash
# Добавление пользователя в группу docker
sudo usermod -aG docker $USER

# Применение изменений (без выхода из сессии)
newgrp docker
```

После этого команды можно выполнять без `sudo`:

```bash
docker run hello-world
```

---

### 0.7. Настройка Docker для автоматического запуска

Настройте Docker на автоматический запуск при загрузке системы :

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Проверьте статус сервиса:

```bash
sudo systemctl status docker
```

---

### 0.8. Проверка Docker Compose

Создайте простой тестовый файл `docker-compose.yml` для проверки работы Docker Compose:

```bash
mkdir ~/compose-test && cd ~/compose-test

cat > docker-compose.yml << 'YAML'
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
YAML

docker compose up -d
docker compose ps
docker compose down
```

Если контейнер успешно запустился и остановился, Docker Compose работает корректно .

