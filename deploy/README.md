# Практическое занятие №18: деплой TeaRoad

**Студент:** Егор Горычев

**Группа:** заполнить перед сдачей

**Проект:** интернет-магазин чая TeaRoad

## Цель и выбранный вариант

Развёртывание фронтенда в Docker Compose с HTTPS через Nginx. Репозиторий [`egorychbog/deploy`](https://github.com/egorychbog/deploy) хранит только конфигурацию развертывания и этот отчёт. Исходники сайта и Dockerfile фронтенда находятся отдельно — в [`egorychbog/TeaRoad`](https://github.com/egorychbog/TeaRoad). Compose собирает `app` непосредственно из ветки `main` этого репозитория.

Приложение full-stack: интерфейс React обращается к Supabase, где работают база PostgreSQL и аутентификация. Supabase — внешний управляемый сервис, поэтому в Docker-проекте нет API/backend-контейнера и нет копии backend-кода. В Compose запущены ровно `app` и `nginx`.

## Схема развертывания

```text
Пользовательский браузер
        │ HTTP :80 / HTTPS :443, только loopback 127.0.0.1
        ▼
nginx — TLS, перенаправление HTTP → HTTPS, reverse proxy
        │ HTTP :8080, внутренняя сеть Compose
        ▼
app — сборка React 19 + Vite + Tailwind CSS, статические файлы
        │ HTTPS из браузера; публичный ключ Supabase
        ▼
Supabase — PostgreSQL и Auth, внешний сервис
```

Nginx — единственная опубликованная точка входа. Порт `app` доступен только внутри Compose-сети. Сайт открывается на этом компьютере по `https://tearoad.localhost`.

## Состав сервисов

| Сервис | Технологии | Внутренние порты | Размещение в рабочей схеме |
| --- | --- | --- | --- |
| `app` | React 19, Vite, Tailwind CSS 3; статическая production-сборка из `TeaRoad` | `8080/tcp` | Контейнер `tearoad-app`; порт наружу не опубликован |
| `nginx` | Nginx Alpine, TLS termination, reverse proxy и HTTP → HTTPS | `8080/tcp` (HTTP), `8443/tcp` (HTTPS) | Контейнер `nginx`; на компьютере порты `127.0.0.1:80` и `127.0.0.1:443` |
| Supabase | Управляемые PostgreSQL и Auth | внешний HTTPS `443` | Внешний сервис; в Compose не запускается |

`app` собирается Dockerfile из `egorychbog/TeaRoad`. Его production-образ содержит собранный фронтенд и внутреннюю раздачу статических файлов. Отдельный `nginx` принимает HTTP/HTTPS с хоста и проксирует запросы в `app`.

## Переменные окружения

Создайте локальный `deploy/.env` на основе `deploy/.env.example`. В Git и отчёте хранятся только названия переменных, не значения.

| Переменная | Назначение |
| --- | --- |
| `VITE_SUPABASE_URL` | Адрес проекта Supabase; подставляется при сборке фронтенда |
| `VITE_SUPABASE_PUBLISHABLE_KEY` | Публичный ключ Supabase для браузера; подставляется при сборке |
| `VITE_SITE_URL` | Адрес сайта, используемый при сборке фронтенда |

Все переменные `VITE_*` доступны в браузерной сборке. Здесь разрешено хранить только публичные значения; `service_role`/secret key, пароль базы и приватные ключи в них передавать нельзя. Локальный `.env` и сертификаты исключены из Git.

Для CI в настройках GitHub Actions репозитория `deploy` задайте repository variables `VITE_SUPABASE_URL` и `VITE_SUPABASE_PUBLISHABLE_KEY`. Значения не вставляйте в workflow или README.

## Настройка HTTPS и домена

Для учебного локального домена используется `mkcert`:

```powershell
mkcert -install
New-Item -ItemType Directory -Force deploy/certs | Out-Null
mkcert -cert-file deploy/certs/tearoad.localhost.pem -key-file deploy/certs/tearoad.localhost-key.pem tearoad.localhost
```

Сертификат и закрытый ключ остаются только на компьютере. В конфигурации Nginx HTTP перенаправляется на HTTPS, а TLS завершается на `nginx`. Домен `.localhost` предназначен для локальной проверки; он не является публичным DNS-именем.

## Запуск и остановка

Команды выполняются из корня клона `deploy` в PowerShell:

```powershell
Copy-Item deploy/.env.example deploy/.env
# Заполните deploy/.env публичным URL и publishable key Supabase.
docker compose --env-file deploy/.env -f deploy/docker-compose.yml up --build -d
docker compose --env-file deploy/.env -f deploy/docker-compose.yml ps
docker compose --env-file deploy/.env -f deploy/docker-compose.yml logs app
docker compose --env-file deploy/.env -f deploy/docker-compose.yml logs nginx
```

Откройте `https://tearoad.localhost`. В браузере проверьте HTTPS-замок, а в DevTools → Network — успешный запрос к Supabase и заголовки ответа. Остановка:

```powershell
docker compose --env-file deploy/.env -f deploy/docker-compose.yml down
```

## Фактическая проверка

Проверка локального стенда: `docker compose ps` показал два сервиса, оба `healthy`; host-порты опубликованы только у `nginx`. Запрос `curl.exe --head https://tearoad.localhost` вернул `HTTP/1.1 200 OK` с `Server: nginx`. Отдельная проверка публичного каталога Supabase `GET /rest/v1/products?select=id&is_active=eq.true&limit=1` вернула HTTP `200` и одну запись.

## CI/CD

Workflow [`.github/workflows/build.yml`](../.github/workflows/build.yml) собирает Docker image фронтенда для push в `main`, pull request в `main` и ручного запуска (`workflow_dispatch`). Контекст сборки — удалённая ветка `main` репозитория `egorychbog/TeaRoad`; сам репозиторий `deploy` содержит workflow и конфигурацию, но не код приложения. Так проверяется, что конфигурация развертывания может собрать текущий фронтенд.

Workflow выполняет CI-сборку и не публикует image в registry, не подключается к удалённому серверу и не запускает Compose там. Локальный запуск выполняется командой выше. Для автоматического CD на реальный сервер потребуются сервер, домен и настроенные защищённые GitHub Actions secrets для доступа к нему; таких секретов в репозитории нет.

## Материалы для сдачи

Сохраните фактические скриншоты в `deploy/screenshots/` и вставьте их в отчёт перед сдачей. Каталог подготовлен; визуальные скриншоты в эту версию ещё не добавлены.

| Скриншот | Что должно быть видно |
| --- | --- |
| `compose-ps.png` | `docker compose ps`: сервисы `app` и `nginx` в состоянии `healthy` |
| `https-localhost.png` | Главная страница сайта по HTTPS и замок браузера |
| `network.png` | Успешный запрос к Supabase и заголовки ответа в DevTools → Network |
| `docker-desktop.png` | В Docker Desktop ровно два контейнера; host-порты опубликованы только у `nginx` |
| `github-actions.png` | Успешная сборка workflow в GitHub Actions (дополнительное подтверждение CI) |

Не используйте макеты вместо снимков работающего стенда. Перед добавлением убедитесь, что в кадр не попали email, пользовательские данные, `.env` или ключи.

## Ошибки и способы устранения

| Ошибка | Причина и решение |
| --- | --- |
| `Cannot connect to the Docker daemon` | Docker Desktop выключен или Engine ещё запускается. Запустите Docker Desktop, дождитесь готовности Engine и повторите Compose-команду. |
| Браузер не доверяет локальному сертификату | Проверьте `mkcert -install`, наличие обоих файлов сертификата и совпадение имени `tearoad.localhost`. |
| `app` или `nginx` не становится `healthy` | Проверьте `docker compose logs app` и `docker compose logs nginx`, локальный `.env` и сертификаты в `deploy/certs/`. |
| Данные Supabase не загружаются | Проверьте URL и publishable key, а также запрос/ответ в DevTools → Network. Не добавляйте secret key в клиентскую сборку. |

## Что изменится для реального сервера

Потребуется VPS или другой Linux-хост, публичный домен с DNS-записью на сервер, firewall с доступом к `80/tcp` и `443/tcp`, и действующий TLS-сертификат, например Let's Encrypt. В Compose нужно привязать опубликованные порты к публичному интерфейсу, заменить локальный сертификат и `server_name`, настроить автоматическое продление сертификата и резервное хранение конфигурации. Для CD workflow потребуется защищённый доступ к серверу и команда безопасного обновления Compose. Secret/service-role ключ Supabase нельзя помещать во frontend image.
