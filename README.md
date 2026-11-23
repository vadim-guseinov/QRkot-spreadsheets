<div align="center">

# 🐈 QRKot

**Асинхронный REST API благотворительного фонда: целевые сборы, автоматическое распределение пожертвований и Excel-отчёты на Яндекс Диске**

![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.135-009688?logo=fastapi&logoColor=white)
![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy-2.0_async-D71F00?logo=sqlalchemy&logoColor=white)
![Pydantic](https://img.shields.io/badge/Pydantic-v2-E92063?logo=pydantic&logoColor=white)
![Tests](https://img.shields.io/badge/pytest-87%2F87_passed-2ea44f?logo=pytest&logoColor=white)

</div>

## О проекте

QRKot — бэкенд фонда, который собирает деньги на целевые проекты (корм, лечение, приюты для кошек).
Администратор открывает проекты с нужной суммой, пользователи делают пожертвования, а сервис **сам распределяет деньги** по открытым проектам в порядке очереди и закрывает проект, как только сумма собрана.
По закрытым проектам суперпользователь одной командой получает **Excel-отчёт на Яндекс Диске** с публичной ссылкой.

## Возможности

- **Целевые проекты** — создание, изменение и удаление (только суперпользователь), просмотр — всем.
- **Пожертвования** — авторизованный пользователь жертвует любую сумму и видит историю своих пожертвований.
- **Автоматическое инвестирование (FIFO)** — новое пожертвование распределяется по самым старым открытым проектам; остаток переходит в следующий проект; новый проект сразу забирает свободные средства.
- **Бизнес-правила на уровне API** — нельзя изменить закрытый проект, уменьшить требуемую сумму ниже уже собранной, удалить проект с внесёнными средствами или создать два проекта с одним именем.
- **Регистрация и JWT-аутентификация** через FastAPI Users, роли: аноним / пользователь / суперпользователь.
- **Отчёт в Excel** (xlsxwriter) по закрытым проектам, отсортированным по скорости сбора, с загрузкой и публикацией через REST API Яндекс Диска (httpx, асинхронно).
- Строгая валидация входных данных (Pydantic v2, `extra='forbid'`), миграции схемы через Alembic, документация OpenAPI (Swagger / ReDoc).

## Стек

| Слой | Технологии |
|---|---|
| Язык | Python 3.12, asyncio |
| Веб-фреймворк | FastAPI, Uvicorn |
| Данные | SQLAlchemy 2.0 (async), Alembic, SQLite / aiosqlite (меняется на PostgreSQL через `DATABASE_URL`) |
| Аутентификация | FastAPI Users, JWT (Bearer) |
| Интеграции | Яндекс Диск REST API, httpx, xlsxwriter |
| Качество | pytest, pytest-asyncio, freezegun, flake8 |

## Что сделано

- Спроектированы модели `CharityProject`, `Donation`, `User` и общий асинхронный CRUD-слой (`CRUDBase`) с наследниками под каждую сущность.
- Реализован сервис распределения средств `invest()` — одна функция обслуживает оба направления: «проект ← пожертвования» и «пожертвование → проекты».
- Логика отчёта вынесена в сервисный слой: формирование книги в памяти (`BytesIO`), форматирование длительности сбора, загрузка и публикация файла через асинхронный клиент Яндекс Диска.
- **87 из 87 автотестов проходят**: аутентификация, CRUD проектов и пожертвований, сценарии распределения средств, клиент и настройки Яндекс Диска.

## Структура

```
app/
├── api/endpoints/   # роутеры: charity_project, donation, user, yandex_api
├── core/            # настройки, подключение к БД, FastAPI Users, клиент Яндекс Диска
├── crud/            # асинхронный CRUD-слой
├── models/          # модели SQLAlchemy
├── schemas/         # схемы Pydantic
└── services/        # распределение средств, генерация отчёта
alembic/             # миграции
tests/               # pytest
```

## Запуск локально

**Требования:** Python 3.10+ (проверено на 3.12), pip. Для отчёта — OAuth-токен Яндекс Диска.

```bash
git clone https://github.com/vadim-guseinov/QRkot-spreadsheets.git
cd QRkot-spreadsheets

python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env            # заполните SECRET и YANDEX_DISK_TOKEN
alembic upgrade head
uvicorn app.main:app --reload
```

Документация API: http://127.0.0.1:8000/docs (Swagger) и http://127.0.0.1:8000/redoc.

### Переменные окружения

| Переменная | Назначение |
|---|---|
| `DATABASE_URL` | строка подключения SQLAlchemy, по умолчанию `sqlite+aiosqlite:///./qrkot.db` |
| `SECRET` | секрет для подписи JWT |
| `YANDEX_DISK_TOKEN` | OAuth-токен Яндекс Диска для выгрузки отчёта |

### Тесты

```bash
pytest
```

## Эндпоинты

| Метод | URL | Доступ | Описание |
|---|---|---|---|
| POST | `/auth/register` | все | регистрация |
| POST | `/auth/jwt/login` | все | получение JWT |
| GET | `/charity_project/` | все | список проектов |
| POST | `/charity_project/` | суперпользователь | создать проект |
| PATCH | `/charity_project/{id}` | суперпользователь | изменить проект |
| DELETE | `/charity_project/{id}` | суперпользователь | удалить проект без пожертвований |
| POST | `/donation/` | пользователь | сделать пожертвование |
| GET | `/donation/my` | пользователь | мои пожертвования |
| GET | `/donation/` | суперпользователь | все пожертвования |
| POST | `/yandex/` | суперпользователь | сформировать отчёт на Яндекс Диске |

## Примеры запросов

Получить токен:

```bash
curl -X POST http://127.0.0.1:8000/auth/jwt/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=user@example.com&password=strongpassword"
```

Создать проект (суперпользователь):

```bash
curl -X POST http://127.0.0.1:8000/charity_project/ \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "Корм для приюта", "description": "Сухой корм на зиму для 40 кошек", "full_amount": 15000}'
```

Ответ:

```json
{
  "name": "Корм для приюта",
  "description": "Сухой корм на зиму для 40 кошек",
  "full_amount": 15000,
  "id": 1,
  "invested_amount": 0,
  "fully_invested": false,
  "create_date": "2026-08-12T10:00:00",
  "close_date": null
}
```

Сделать пожертвование:

```bash
curl -X POST http://127.0.0.1:8000/donation/ \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"full_amount": 5000, "comment": "На корм"}'
```

## Связанные репозитории

Проект развивался в три этапа:
[cat-charity-1](https://github.com/vadim-guseinov/cat-charity-1) (базовый API и распределение средств) →
[cat-charity-2](https://github.com/vadim-guseinov/cat-charity-2) (пользователи и JWT) →
**QRkot-spreadsheets** (отчёты на Яндекс Диске, итоговая версия).

## Автор

**Вадим Гусейнов** — Python-разработчик (Backend) · [Telegram](https://t.me/eviliing)
