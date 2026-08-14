# Docker Pet Project: Flask + PostgreSQL

Этот проект демонстрирует развертывание веб-приложения на Flask с базой данных PostgreSQL в Docker-контейнерах.

---

## Содержание

1. [О проекте](#1-о-проекте)
2. [Технологии](#2-технологии)
3. [Структура проекта](#3-структура-проекта)
4. [Требования](#4-требования)
5. [Быстрый старт](#5-быстрый-старт)
6. [Запуск с PostgreSQL](#6-запуск-с-postgresql)
7. [API](#7-api)
8. [Безопасность](#8-безопасность)
9. [CI/CD](#9-cicd)
10. [Автор](#10-автор)

---

## 1. О проекте

Простое Flask-приложение, которое считает количество посещений и сохраняет данные в PostgreSQL.

**Функциональность:**
- Главная страница `/` - возвращает JSON с количеством посещений
- Healthcheck `/health` - проверяет состояние приложения и подключение к БД

---

## 2. Технологии

| Компонент | Технология | Версия |
|-----------|------------|--------|
| Веб-фреймворк | Flask | 2.3.3 |
| База данных | PostgreSQL | 15-alpine |
| Драйвер БД | psycopg2-binary | 2.9.7 |
| Язык | Python | 3.9-slim-bullseye |
| Контейнеризация | Docker | 24.0.5 |
| Сканирование | Trivy | latest |
| CI/CD | GitHub Actions | ubuntu-latest |

---

## 3. Структура проекта

```
Docker/
└── docker-pet-project(Flask+PostgreSQL)/
    ├── app.py                     # Flask-приложение
    ├── requirements.txt           # Python-зависимости
    ├── Dockerfile                 # Сборка образа
    ├── .dockerignore              # Исключения для Docker
    └── .github/
        └── workflows/
            └── docker-build.yml   # CI/CD пайплайн
```

---

## 4. Требования

- Docker (версия 23.0 или выше)
- Git (для клонирования)

---

## 5. Быстрый старт

**Клонируйте репозиторий:**
```bash
git clone https://github.com/denbotanin-source/Docker.git
cd Docker/docker-pet-project\(Flask+PostgreSQL\)
```

**Соберите образ:**
```bash
docker build -t flask-postgres .
```

**Проверьте, что образ собрался:**
```bash
docker images | grep flask-postgres
```

---

## 6. Запуск с PostgreSQL

### Шаг 1. Создайте сеть и volume
```bash
docker network create app-network
docker volume create pg-data
```

### Шаг 2. Запустите PostgreSQL
```bash
docker run -d \
  --name postgres-db \
  --network app-network \
  -v pg-data:/var/lib/postgresql/data \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=postgres \
  postgres:15-alpine
```

### Шаг 3. Запустите приложение
```bash
docker run -d \
  --name flask-app \
  --network app-network \
  -e DB_HOST=postgres-db \
  -e DB_NAME=postgres \
  -e DB_USER=postgres \
  -e DB_PASS=postgres \
  -p 5000:5000 \
  flask-postgres
```

### Шаг 4. Проверьте работу
```bash
curl http://localhost:5000/
# {"message":"Hello from Flask + PostgreSQL!","visits":1}

curl http://localhost:5000/health
# {"status":"healthy","db":"connected"}
```

### Шаг 5. Проверьте сохранение данных
```bash
# Несколько запросов
curl http://localhost:5000/
curl http://localhost:5000/
curl http://localhost:5000/

# Перезапустите PostgreSQL
docker restart postgres-db

# Проверьте снова - счётчик не сбросился
curl http://localhost:5000/
```

### Шаг 6. Остановка контейнеров
```bash
docker stop flask-app postgres-db
docker rm flask-app postgres-db
docker network rm app-network
docker volume rm pg-data
```

---

## 7. API

### `GET /`
Возвращает приветствие и текущее количество посещений.

**Пример ответа:**
```json
{
  "message": "Hello from Flask + PostgreSQL!",
  "visits": 42
}
```

### `GET /health`
Проверяет состояние приложения и подключение к базе данных.

**Пример ответа (успех):**
```json
{
  "status": "healthy",
  "db": "connected"
}
```

**Пример ответа (ошибка):**
```json
{
  "status": "unhealthy",
  "db": "connection to server failed: Connection refused"
}
```

---

## 8. Безопасность

| Мера | Описание |
|------|----------|
| Non-root пользователь | Приложение запускается от `appuser` с UID 1000 |
| Обновление пакетов | `apt-get upgrade` выполняется при сборке |
| Очистка кэша | Удаление `/var/lib/apt/lists/*` и кэша pip |
| Healthcheck | Мониторинг состояния через `/health` |
| Сканирование | Trivy проверяет CRITICAL/HIGH уязвимости |

---

## 9. CI/CD

Проект использует GitHub Actions для автоматической сборки и тестирования.

**Что делает пайплайн:**
1. Клонирует репозиторий
2. Собирает Docker-образ
3. Запускает PostgreSQL в тестовом окружении
4. Проверяет healthcheck эндпоинт

**Файл пайплайна (`.github/workflows/docker-build.yml`):**
```yaml
name: Build Docker Image

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build Docker image
        run: |
          cd "docker-pet-project(Flask+PostgreSQL)"
          docker build -t flask-postgres:latest .

      - name: Run PostgreSQL
        run: |
          docker network create test-network || true
          docker run -d \
            --name postgres-test \
            --network test-network \
            -e POSTGRES_USER=postgres \
            -e POSTGRES_PASSWORD=postgres \
            -e POSTGRES_DB=postgres \
            postgres:15-alpine
          sleep 5

      - name: Run container and test
        run: |
          docker run -d \
            --name test-app \
            --network test-network \
            -e DB_HOST=postgres-test \
            -e DB_NAME=postgres \
            -e DB_USER=postgres \
            -e DB_PASS=postgres \
            -p 5000:5000 \
            flask-postgres:latest
          sleep 5
          curl -f http://localhost:5000/health || exit 1
          docker stop test-app
          docker rm test-app
          docker stop postgres-test
          docker rm postgres-test
```

---

## 10. Автор

**denbotanin-source**

GitHub: [https://github.com/denbotanin-source](https://github.com/denbotanin-source)
