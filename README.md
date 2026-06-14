# Momo Store Docker Project

Проект содержит два сервиса:

- `backend` - Go API, порт `8081`;
- `frontend` - Vue.js SPA, публикуется на порту `80` и проксирует `/api` в backend.

## Быстрый запуск

```bash
docker compose build
docker compose up -d
```

Проверка:

```bash
curl http://localhost:8081/health
curl http://localhost/
```

Остановка:

```bash
docker compose down
```

## Образы

Образы имеют явные локальные теги:

- `momo-store-backend:1.0.0`;
- `momo-store-frontend:1.0.0`.

Проверить итоговые размеры после сборки:

```bash
docker images momo-store-backend:1.0.0 momo-store-frontend:1.0.0
```

Ожидаемо backend получается небольшим, потому что финальный образ содержит только скомпилированный Go-бинарник, `ca-certificates` и минимальный Alpine runtime. Frontend собирается в отдельной Node-стадии, а финальный образ содержит только nginx и статические файлы из `dist`.

## Конфигурация

Backend:

- слушает порт `8081`;
- использует переменную `GOMAXPROCS=1` в `docker-compose.yml`.

Frontend:

- принимает build argument `VUE_APP_API_URL`;
- по умолчанию использует `/api`;
- nginx проксирует `/api/` на `http://backend:8081/`.

Пример переопределения API URL при отдельной сборке frontend:

```bash
docker build --build-arg VUE_APP_API_URL=/api -t momo-store-frontend:1.0.0 ./frontend
```

## Docker Compose

`docker-compose.yml` описывает сервисы `backend` и `frontend`.

Настроены:

- `depends_on` с ожиданием healthy backend для frontend;
- отдельные сети `frontend` и `backend`;
- внутренний `backend` network с `internal: true`;
- volume `backend-data` для данных приложения;
- healthchecks для обоих контейнеров;
- restart policy `unless-stopped`;
- лимиты `cpus` и `mem_limit`.

Backend можно масштабировать, так как frontend обращается к нему по имени сервиса внутри Docker network:

```bash
docker compose up -d --scale backend=2
```

Frontend публикует порт `80`, поэтому его масштабирование требует отдельного внешнего балансировщика или снятия прямого host port mapping.

## Безопасность

В Dockerfile и Compose применены базовые меры безопасности:

- multi-stage builds для backend и frontend;
- легковесные Alpine-based runtime images;
- непривилегированные пользователи в финальных контейнерах;
- отсутствие dev-инструментов в финальных образах;
- `.dockerignore` для сокращения build context;
- `read_only: true`;
- `tmpfs` для временных директорий;
- `cap_drop: [ALL]`;
- `security_opt: no-new-privileges:true`;
- открыты только необходимые порты: `80` и `8081`;
- секреты не встраиваются в Docker images.

DockerHub credentials хранятся только в GitHub Actions Secrets:

- `DOCKER_USER`;
- `DOCKER_PASSWORD` - DockerHub access token с правом `Read & Write`.

## Сканирование образов

В `.github/workflows/deploy.yaml` добавлен job `trivy-image-scan`.

Он:

1. собирает backend image;
2. собирает frontend image;
3. сканирует оба образа через `aquasecurity/trivy-action`;
4. падает при `HIGH` или `CRITICAL` уязвимостях.

Локальная проверка:

```bash
docker compose build
trivy image --severity HIGH,CRITICAL --ignore-unfixed momo-store-backend:1.0.0
trivy image --severity HIGH,CRITICAL --ignore-unfixed momo-store-frontend:1.0.0
```
