# Отчёт по лабораторной работе №3

## Цель

Организовать запуск всего стека приложения:

- **Основное FastAPI-приложение** (лабораторная работа №1).
- **Сервис парсинга данных** (лабораторная работа №2).
- **База данных PostgreSQL**.

## Задачи

1. Создать HTTP-обёртку для парсера на FastAPI.
2. Настроить Dockerfile для каждого сервиса.
3. Сформировать `docker-compose.yml` для одновременного запуска всех компонентов.
4. Реализовать эндпоинт в основном приложении для вызова парсера.

## Архитектура решения

finance-app/
├─ app/ # Основное FastAPI-приложение
│ └─ main.py
├─ parser_service/ # FastAPI-обёртка для парсера
│ └─ main.py
├─ Lab2/Task2/… # Код парсера и утилиты
├─ requirements.txt
├─ Dockerfile.app # Dockerfile для основного приложения
├─ Dockerfile.parser # Dockerfile для сервиса парсера
└─ docker-compose.yml

### Сервисы

- **app** – основное веб-приложение для управления финансами.
- **parser** – сервис FastAPI, вызывающий асинхронный парсер и сохраняющий результат в БД.

## Реализация HTTP-вызова парсера

### Обёртка FastAPI для парсера (`parser_service/main.py`)

```python
from fastapi import FastAPI, HTTPException, Query
import asyncio
from Lab2.Task2.async_parser import parse_and_save
import aiohttp

app = FastAPI(title="Parser API")

@app.post("/parse")
async def parse_endpoint(url: str = Query(..., description="URL для парсинга")):
    try:
        async with aiohttp.ClientSession() as session:
            await parse_and_save(url)
        return {"status": "ok", "url": url}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Эндпоинт в основном приложении (app/main.py)

```
from fastapi import FastAPI, HTTPException
import httpx

app = FastAPI(title="Finance API")

@app.post("/run-parse")
async def run_parse(url: str):
    try:
        async with httpx.AsyncClient() as client:
            resp = await client.post("http://parser:8001/parse", params={"url": url})
        resp.raise_for_status()
        return resp.json()
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Файлы конфигурации

requirements.txt

```
fastapi
uvicorn[standard]
aiohttp
requests
beautifulsoup4
psycopg2-binary
httpx
```

### Dockerfile для основного приложения (Dockerfile.app)

```
FROM python:3.11-slim
WORKDIR /code
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app ./app
ENV PYTHONUNBUFFERED=1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Dockerfile для парсера (Dockerfile.parser)

```
FROM python:3.11-slim
WORKDIR /code
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY parser_service ./parser_service
COPY Lab2 ./Lab2
ENV PYTHONUNBUFFERED=1
CMD ["uvicorn", "parser_service.main:app", "--host", "0.0.0.0", "--port", "8001"]
```

### docker-compose.yml

```
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.app
    environment:
      # подключаемся к postgres, который запущен на хосте
      DATABASE_URL: postgresql+psycopg2://postgres:admin1@host.docker.internal:5432/financedb
    ports:
      - "8000:8000"

  parser:
    build:
      context: .
      dockerfile: Dockerfile.parser
    environment:
      DATABASE_URL: postgresql+psycopg2://postgres:admin1@host.docker.internal:5432/financedb
    ports:
      - "8001:8001"
```

### Запуск проекта

```
docker compose up --build
```

После сборки доступны сервисы:

- http://localhost:8000
  – основное FastAPI-приложение.

- http://localhost:8001
  – сервис парсера.

### Пример вызова парсинга

```
curl -X POST "http://localhost:8000/run-parse" -d "url=https://itmo.ru/"
```

Ответ:

```
{
  "status": "ok",
  "url": "https://itmo.ru/"
}
```

Вывод

Выполнена полная контейнеризация проекта:

- Упакованы три сервиса: FastAPI, PostgreSQL, парсер данных.

- Организован HTTP-вызов парсера из основного приложения.

- Использован docker-compose для оркестрации и удобного запуска всего стека.
