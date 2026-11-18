
# Лабораторная работа: Настройка OAuth 2.0 аутентификации для RabbitMQ 4.x с помощью Keycloak

## Цель работы
Настроить RabbitMQ версии **4.x** для аутентификации пользователей через Keycloak с использованием протокола OAuth 2.0 и JWT-токенов, обеспечив безопасное HTTPS-соединение между компонентами.

## Требования
- Виртуальная машина или сервер с **Debian 12 (Bookworm)**.
- Права `sudo`.
- **Erlang/OTP 27.x** (обязательное требование для RabbitMQ 4.x).
- Установленный Java Runtime Environment (JRE) версии 17 или выше для Keycloak.

> **Важно:** RabbitMQ 4.x **не будет работать** на старых версиях Erlang. Минимально поддерживаемая версия — **27.0** (для серии 4.4.1) или **26.2** (для 4.0.x). Рекомендуется использовать Erlang 27.x из репозитория Cloudsmith.

---

## Задание 1: Установка RabbitMQ 4.x из актуального репозитория

> **Важно:** Репозиторий `dl.bintray.com` закрыт и не работает. Для установки современных версий RabbitMQ и Erlang используется репозиторий Cloudsmith, поддерживаемый командой RabbitMQ.

1.  **Установите необходимые зависимости:**
    ```bash
    sudo apt-get update
    sudo apt-get install -y curl gnupg apt-transport-https debian-keyring debian-archive-keyring
    ```

2.  **Добавьте репозитории Cloudsmith** (рекомендуемый способ для RabbitMQ 4.x):
    ```bash
    # Репозиторий Erlang (современные версии, включая 27.x)
    curl -1sLf 'https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-erlang/setup.deb.sh' | sudo -E bash

    # Репозиторий RabbitMQ Server
    curl -1sLf 'https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-server/setup.deb.sh' | sudo -E bash
    ```
    *Эти скрипты автоматически добавят GPG-ключи и списки источников для вашего дистрибутива (Debian 12).*

3.  **Обновите индексы и установите RabbitMQ 4.x:**
    ```bash
    sudo apt-get update
    sudo apt-get install -y rabbitmq-server
    ```
    *По умолчанию будет установлена последняя доступная версия из репозитория Cloudsmith (на момент написания — 4.x).*

4.  **Проверьте установленную версию** (убедитесь, что это 4.x, и используется Erlang 27.x):
    ```bash
    sudo rabbitmqctl version
    sudo rabbitmqctl eval 'erlang:system_info(otp_release).'
    ```

5.  **Включите плагин OAuth 2.0 и Management UI**:
    ```bash
    sudo rabbitmq-plugins enable rabbitmq_management rabbitmq_auth_backend_oauth2
    ```

---

## Задание 2: Настройка HTTPS для Keycloak (обязательный шаг)

RabbitMQ требует, чтобы URL-адрес провайдера идентификации (Issuer URL) **обязательно использовал HTTPS**. Поэтому Keycloak должен быть настроен на работу с TLS.

1.  **Сгенерируйте самоподписанный сертификат и приватный ключ** (для учебной среды):
    ```bash
    openssl req -newkey rsa:2048 -nodes \
      -keyout keycloak.key.pem \
      -x509 -days 3650 \
      -out keycloak.crt.pem \
      -subj "/CN=keycloak.local"
    ```
    *Примечание: Вместо `keycloak.local` можно указать IP-адрес или доменное имя вашего сервера.*

2.  **Запустите Keycloak с указанием сертификата и ключа** (пример для standalone-запуска):
    ```bash
    bin/kc.sh start \
      --https-certificate-file=/path/to/keycloak.crt.pem \
      --https-certificate-key-file=/path/to/keycloak.key.pem \
      --hostname=keycloak.local
    ```
    *Убедитесь, что Keycloak доступен по адресу `https://keycloak.local:8443` (порт по умолчанию для HTTPS).*

---

## Задание 3: Конфигурация RabbitMQ 4.x для OAuth 2.0

1.  **Отредактируйте файл `/etc/rabbitmq/rabbitmq.conf`** (создайте его, если он отсутствует):

    ```ini
    # Включаем OAuth 2.0 как бэкенд аутентификации (полное имя модуля)
    auth_backends.1 = rabbit_auth_backend_oauth2

    # Идентификатор ресурса, который должен присутствовать в токене (audience)
    auth_oauth2.resource_server_id = rabbitmq

    # URL Issuer'а Keycloak — ОБЯЗАТЕЛЬНО HTTPS
    auth_oauth2.issuer = https://keycloak.local:8443/realms/master
    # (замените master на имя вашего realm, если используется другой)

    # Доверие к самоподписанному сертификату Keycloak
    auth_oauth2.https.verify = verify_peer
    auth_oauth2.https.cacertfile = /path/to/keycloak.crt.pem

    # Отключаем проверку audience (опционально, для упрощения в учебной среде)
    auth_oauth2.verify_aud = false

    # Включаем плагин управления (если ещё не включён)
    management.oauth_enabled = true
    management.oauth_provider_url = https://keycloak.local:8443/realms/master
    ```

    > **Ключевые изменения для RabbitMQ 4.x:**
    > - **`auth_backends.1 = rabbit_auth_backend_oauth2`** — полное имя модуля, рекомендуется для 4.x.
    > - **`auth_oauth2.issuer`** — URL **должен быть HTTPS**, иначе RabbitMQ не запустится.
    > - **`auth_oauth2.jwks_uri`** — если вы используете этот параметр напрямую, в RabbitMQ 4.x он называется **`jwks_uri`** (в 3.x был `jwks_url`).
    > - **`auth_oauth2.https.cacertfile`** — путь к сертификату, которым подписан HTTPS-сертификат Keycloak. Для самоподписанного сертификата это тот же файл `keycloak.crt.pem`.

2.  **Перезапустите RabbitMQ**:
    ```bash
    sudo systemctl restart rabbitmq-server
    ```

3.  **Проверьте статус и логи**:
    ```bash
    sudo systemctl status rabbitmq-server
    sudo journalctl -u rabbitmq-server -n 50 --no-pager
    ```
    Убедитесь, что в логах нет ошибок, связанных с `auth_oauth2.issuer` или версией Erlang.

---

## Задание 4: Настройка Keycloak (Realm, Client, User)

1.  Войдите в административную консоль Keycloak: `https://keycloak.local:8443/admin`.
2.  Создайте новый **Realm** (например, `rabbitmq-realm`) или используйте существующий (`master`).
3.  В разделе **Clients** создайте клиент:
    - **Client ID**: `rabbitmq`
    - **Client Protocol**: `openid-connect`
    - **Access Type**: `confidential`
    - **Valid Redirect URIs**: `https://<ваш-rabbitmq-хост>:15671/*` (для Management UI)
    - Сохраните и скопируйте **Client Secret**.
4.  Создайте пользователя и задайте ему пароль.
5.  Настройте **Client Scopes** или **Mappers**, чтобы в токене присутствовал параметр `aud` (audience) со значением `rabbitmq` (или отключите проверку audience в RabbitMQ, как показано выше).

> **Примечание:** RabbitMQ 4.x официально совместим с Keycloak в качестве OAuth 2.0 сервера. Убедитесь, что версия Keycloak актуальна (например, 24.x или выше).

---

## Задание 5: Проверка работоспособности

1.  Откройте RabbitMQ Management UI: `https://<ваш-rabbitmq-хост>:15671`.
2.  Нажмите кнопку входа через OAuth 2.0 / Keycloak.
3.  Вы будете перенаправлены на страницу входа Keycloak. Введите учётные данные созданного пользователя.
4.  После успешной аутентификации вы должны быть перенаправлены обратно в RabbitMQ Management UI с активной сессией.

---
