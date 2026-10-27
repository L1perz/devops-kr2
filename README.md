# Контрольная работа 2

Карчевский Лев Олегович, ЭФБО-05-24, №11. Вариант 3: `stats.devops.local`, порт `8003`.

Базовый образ `python:3.12-slim` получен в ПР 7. Логика `app.py` и состав `requirements.txt` сохранены из шаблона; добавлен только комментарий с фамилией.

## Доставка

```bash
rsync -avz --delete --exclude=.env --exclude=.git --dry-run ./ devops:kr2/
rsync -avz --delete --exclude=.env --exclude=.git ./ devops:kr2/
```

## Запуск на сервере

Файл `.env` создаётся один раз на сервере и сохраняется при доставке:

```bash
cd ~/kr2
umask 077
printf 'REDIS_PASSWORD=%s\n' "$(openssl rand -hex 32)" > .env
docker build --network=host -t kr2-app:1.0 .
docker compose up -d --no-build --wait
```

Параметр `--network=host` использован только при сборке в WSL из-за DNS BuildKit. Рабочие контейнеры используют сеть `backend`. Не экспортировать `REDIS_PASSWORD` в оболочку. В `.env.example` находится только заглушка. Действующий пароль не хранится в Git и не попадает в контекст сборки.

## Публикация

Конфигурация Nginx находится в `nginx/kr2`. На сервере создаются отдельные `/etc/ssl/certs/kr2.crt` с SAN `DNS:stats.devops.local` и `/etc/ssl/private/kr2.key` с правами `600`, владельцем `root`. Файл `kr2` активируется ссылкой в `sites-enabled`; затем выполняются `nginx -t` и `systemctl reload nginx`.

На клиенте имя `stats.devops.local` сопоставляется адресу сервера. Открытый сертификат копируется по SSH. Проверки:

```bash
curl --cacert ~/kr2.crt https://stats.devops.local/
curl --cacert ~/kr2.crt https://stats.devops.local/whoami
docker compose ps
docker compose top
```

Приложение опубликовано только на `127.0.0.1:8003`. Redis доступен приложению по имени `redis` в сети `backend`, требует пароль и сохраняет AOF в томе `redis_data`; порт `6379` не опубликован. Оба сервиса имеют `restart: unless-stopped`.

Шаблон: https://gitverse.ru/dgimatdinov/kr2-template.git
