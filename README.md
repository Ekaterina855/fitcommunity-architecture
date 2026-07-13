# 🏋️ FitCommunity — Платформа для фитнес-сообществ

> **Архитектурный проект** от анализа требований до контрактов микросервисов.  
> Включает REST API, OpenAPI 3.0, генерацию кода, RBAC, статусные модели и правила эволюции.

[![GitHub Pages](https://img.shields.io/badge/GitHub%20Pages-Live-brightgreen)](https://Ekaterina855.github.io/fitcommunity-architecture/docs/index.html)
[![OpenAPI](https://img.shields.io/badge/OpenAPI-3.0-blue)](https://swagger.io/specification/)

---

## 📌 О проекте

Платформа объединяет:
- **Трекинг тренировок** (ручной ввод + синхронизация с Garmin/Apple Health)
- **Социальную сеть** (посты, ленты, группы, лайки)
- **Персонализированные рекомендации** (AI-советник на основе целей)

Проект выполнен в рамках курса «Проектирование архитектуры информационных систем» (Финансовый университет, группа УЦИ23-1) с использованием AI-ассистентов (ChatGPT, DeepSeek) для ускорения разработки.

**Ключевые результаты:**
- 17 REST-эндпойнтов с полной документацией
- 3 OpenAPI-спецификации (test, prod, admin)
- Генерация server stub (Python FastAPI) и client SDK (TypeScript)
- 5 микросервисов с собственными контрактами
- Матрица ролей, таблица взаимодействий, статусные диаграммы
- Правила версионирования API

---

## 🏗️ Архитектурные решения

### Сравнение вариантов архитектуры
Были рассмотрены 4 варианта (монолит/микросервисы × тонкий/толстый клиент). Для MVP выбран **монолит с толстым клиентом** — оптимальное сочетание скорости разработки и пользовательского опыта.

### Точка входа – API Gateway
- Аутентификация через JWT (Bearer)
- Rate limiting (100 запросов/мин)
- Маршрутизация запросов к микросервисам
- **Агрегация данных не выполняется** – клиент сам комбинирует ответы (снижает задержки и связанность)

### Диаграммы архитектуры
Все диаграммы представлены в папке [`diagrams/`](diagrams/). Ниже – ключевые:

| Диаграмма | Описание |
|-----------|----------|
| [Контекстная диаграмма](diagrams/Контекстная диаграмма (C4 Context Level).svg) | Контекстная (C4) – взаимодействие системы с внешним миром |
| [Логическая архитектура](diagrams/Логическая архитектура (уровень контейнеров C4).svg) | Контейнеры (C4) – логическая структура системы |
| [Компоненты монолита](diagrams/Диаграмма компонентов монолита.svg) | Внутренние компоненты монолита (слои: UI, Application, Domain, Data) |
| [Статусы тренировки](diagrams/Статусная модель Тренировка.svg) | Статусная модель тренировки |
| [Статусы подписки](diagrams/Статусная модель Подписка.svg) | Статусная модель подписки |
| [Роли RBAC](diagrams/Диаграмма ролей (RBAC).svg) | Диаграмма ролей (USER, TRAINER, ADMIN) |
> 📸 Полный набор диаграмм доступен [здесь](diagrams/).

---

## 📄 API Контракты (OpenAPI 3.0)

Интерактивная документация доступна по ссылке:  
👉 **[Swagger UI (docs/index.html)](https://ekaterina855.github.io/fitcommunity-architecture/#/)**

### Спецификации монолита
| Файл | Описание |
|------|----------|
| [`openapi-prod.yaml`](openapi/openapi-prod.yaml) | Продуктовое API (17 бизнес-операций) |
| [`openapi-test.yaml`](openapi/openapi-test.yaml) | Тестовая среда (с отладочными ручками) |
| [`openapi-admin.yaml`](openapi/openapi-admin.yaml) | Администрирование (роль ADMIN) |

### Контракты микросервисов
| Сервис | Файл |
|--------|------|
| User Service | [`user-service.yaml`](openapi/user-service.yaml) |
| Training Service | [`training-service.yaml`](openapi/training-service.yaml) |
| Social Service | [`social-service.yaml`](openapi/social-service.yaml) |
| Billing Service | [`billing-service.yaml`](openapi/billing-service.yaml) |
| Recommendation Service | [`recommendation-service.yaml`](openapi/recommendation-service.yaml) |
| API Gateway | [`gateway.yaml`](openapi/gateway.yaml) |

---

## 🔐 Безопасность и роли

- **Аутентификация:** Bearer JWT
- **Роли:** USER, TRAINER, ADMIN
- **Матрица доступа** – в отчёте (часть B)

### Прокси-ручка `/payments/proxy`
Интеграция с внешним платёжным шлюзом CloudPayments:
- **Таймаут:** 5 секунд
- **Ретраи:** 1 повтор при сетевой ошибке
- **Маппинг ошибок:**
  - 4xx от шлюза → 402 Payment Required
  - 5xx от шлюза → 502 Bad Gateway
  - Таймаут → 504 Gateway Timeout

---

## 🧩 Микросервисная декомпозиция

| Сервис | Ответственность | База данных |
|--------|----------------|-------------|
| **User Service** | Регистрация, профили, цели, роли | user_db |
| **Training Service** | Тренировки, синхронизация с Garmin | training_db |
| **Social Service** | Посты, лента, группы, лайки | social_db |
| **Billing Service** | Подписки, платежи, инвойсы | billing_db |
| **Recommendation Service** | AI-рекомендации тренировок и питания | recommend_db |

### Взаимодействия
- **Синхронные вызовы** (REST) – для операций, требующих немедленной консистентности (например, получение веса пользователя для расчёта калорий).
- **Асинхронные события** (RabbitMQ) – для слабой связанности (например, событие `TrainingAdded` → Social публикует пост, Recommendation обновляет ML-модель).

> Таблица всех взаимодействий – в отчёте (часть D).

---

## 📊 Статусные модели (state machine)

### Тренировка
- `created` – ручное создание
- `syncing` – синхронизация с Garmin запущена
- `synced` – успех
- `failed` – ошибка (повторная попытка → syncing)

### Подписка
- `pending` – ожидает оплаты
- `active` – активна
- `expired` – истекла
- `cancelled` – отменена

Недопустимые переходы (например, `expired → active`) возвращают **409 Conflict**.

---

## 🧪 Тестирование API (Postman)

Коллекция Postman содержит **17 эндпойнтов** с тестами (позитивные и негативные сценарии). Включает проверку статусов, обязательных полей, а также сценарии ошибок (401, 404, 409 и др.).

| Файл | Описание |
|------|----------|
| [`FitCommunity_Monolith_API.json`](postman/FitCommunity_Monolith_API.json) | Коллекция с 17 запросами и тестами |
| [`DEV.json`](postman/DEV.json) | Окружение для разработки (`http://localhost:8000`) |
| [`TEST.json`](postman/TEST.json) | Окружение для тестирования (`https://test.fitcommunity.com/api`) |

### Как использовать
1. Импортируйте все три файла в Postman.
2. Выберите окружение DEV или TEST.
3. Запустите запросы вручную или используйте **Collection Runner** для автоматического прогона всех тестов.

**Переменные окружения:**
- `baseUrl` – адрес сервера
- `apiVersion` – версия API (v1)
- `token` – JWT-токен (заполняется после логина)
- `userId`, `trainingId`, `postId`, `groupId` – заполняются автоматически из ответов

---

## 🛠️ Инструменты и стек

- **OpenAPI 3.0** / Swagger UI
- **Postman** (коллекции, переменные окружения, тесты)
- **PlantUML** (диаграммы C4, последовательности, состояний)
- **Python FastAPI** (server stub)
- **TypeScript** (client SDK)
- **JWT, RabbitMQ, PostgreSQL**
- **GitHub Pages** (этот сайт)

---

## 🔗 Ссылки

- [Интерактивная документация API (Swagger UI)](https://ekaterina855.github.io/fitcommunity-architecture/#/)
- [Репозиторий на GitHub](https://github.com/Ekaterina855/fitcommunity-architecture)
- [Postman коллекция](postman/FitCommunity_Monolith_API.json)

---

## 👤 Автор

**Щур Екатерина Сергеевна**  
Группа УЦИ23-1, Финансовый университет  
Проект выполнен при поддержке AI-ассистентов (ChatGPT, DeepSeek).

[![GitHub](https://img.shields.io/badge/GitHub-Ekaterina855-181717?logo=github)](https://github.com/Ekaterina855)

---

## 📝 Примечание

Все артефакты (OpenAPI, диаграммы, Postman коллекции, контракты сервисов) являются частью учебного проекта и демонстрируют навыки архитектурного проектирования, контрактирования API и работы с современными инструментами.