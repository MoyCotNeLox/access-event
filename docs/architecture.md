# Архитектура системы AccessEvent

## 1. Трёхслойная архитектура

### Клиентский слой (Frontend)

- **Технологии:** HTML5, CSS3, JavaScript (ES6+), Jinja2, Fetch API, WebSocket (Socket.IO client)
- **Компоненты:**
  - Страницы каталога мероприятий (список, фильтры, поиск)
  - Карточка мероприятия (описание, опции доступности, материалы)
  - Личный кабинет участника (мои регистрации, уведомления)
  - Панель организатора (создание/редактирование событий, заявки, статистика)
  - Панель администратора (модерация, категории, пользователи)
  - Текстовый чат мероприятия (WebSocket)
  - Компонент визуальных уведомлений (цветные баннеры)
- **Адаптация для глухих пользователей:**
  - Цветовая индикация статусов вместо звуковых сигналов (🟡 🟢 🔴)
  - Визуальные баннеры-уведомления с высоким контрастом
  - Счётчики непрочитанных сообщений и подсветка новых
  - Полная навигация с клавиатуры, ARIA-атрибуты
  - Отсутствие звуковых элементов как единственного способа передачи информации
  - Шрифт ≥ 16px, контрастность ≥ 4.5:1 (WCAG 2.1 AA)

### Серверный слой (Backend)

- **Технологии:** Python 3.11+, Flask 2.3+, SQLAlchemy 2.x, Flask-Login, Flask-SocketIO, Alembic, bcrypt, Marshmallow (валидация)
- **Компоненты:**
  - **Маршрутизация** — Blueprint'ы: `auth`, `events`, `registrations`, `chat`, `notifications`, `reviews`, `admin`
  - **Бизнес-логика** — сервисный слой (`services/`): управление событиями, регистрациями, уведомлениями, чатом
  - **ORM** — модели SQLAlchemy: `User`, `Event`, `Category`, `Registration`, `Notification`, `ChatMessage`, `Review`, `AccessibilityOption`
  - **Подсистема безопасности** — аутентификация (Flask-Login/JWT), авторизация по ролям (RBAC), CSRF-защита, bcrypt-хеширование паролей
  - **Планировщик задач** — APScheduler/Celery для отправки визуальных уведомлений по расписанию
  - **WebSocket-сервер** — Flask-SocketIO для чата в реальном времени
- **Паттерн:** MVC (Model-View-Controller)
  - **Model** — SQLAlchemy-модели и сервисы
  - **View** — Jinja2-шаблоны + JSON API
  - **Controller** — Flask Blueprint'ы и обработчики маршрутов

### Слой данных

- **СУБД:** PostgreSQL 15
- **Таблицы:**
  - `users` — пользователи (организаторы, участники, админы)
  - `events` — мероприятия
  - `registrations` — заявки на участие
  - `messages` — сообщения текстового чата
  - `event_categories` — категории мероприятий
  - `event_category_link` — связь M:N между событиями и категориями
  - `notifications` — визуальные уведомления
  - `reviews` — отзывы и оценки
  - `accessibility_options` — опции доступности мероприятия
- **Связи:**
  - `users (organizer) 1—N events`
  - `users (participant) 1—N registrations`
  - `events 1—N registrations, notifications, messages, reviews, accessibility_options`
  - `events N—M event_categories` (через `event_category_link`)
- **Индексы:** по `events.date_start`, `registrations.status`, `messages.event_id`, `notifications.user_id`
- **Миграции:** Alembic
- **Резервное копирование:** ежедневный pg_dump, хранение ≥ 30 дней

---

## 2. Клиент-серверное взаимодействие

### REST API

| Метод | Endpoint | Описание |
|-------|----------|----------|
| GET | `/events` | Список мероприятий (с фильтрами и пагинацией) |
| GET | `/events/{id}` | Детали мероприятия |
| POST | `/events` | Создание мероприятия (организатор) |
| PUT | `/events/{id}` | Обновление мероприятия |
| DELETE | `/events/{id}` | Удаление мероприятия |
| POST | `/events/{id}/registrations` | Регистрация на мероприятие |
| PUT | `/registrations/{id}` | Одобрение/отклонение заявки |
| DELETE | `/registrations/{id}` | Отмена регистрации |
| GET | `/events/{id}/messages` | История чата |
| POST | `/events/{id}/messages` | Отправить сообщение |
| GET | `/notifications` | Список уведомлений |
| POST | `/events/{id}/reviews` | Оставить отзыв |
| GET | `/categories` | Список категорий |
| POST | `/admin/users/{id}/block` | Блокировка пользователя |

### WebSocket-события

| Событие | Направление | Описание |
|---------|-------------|----------|
| `join_event` | client → server | Подключение к чату мероприятия |
| `new_message` | server → client | Новое сообщение в чате |
| `notification` | server → client | Визуальное уведомление (🟡 🟢 🔴) |
| `registration_update` | server → client | Изменение статуса регистрации |

### Формат обмена

- **Запросы/ответы:** JSON (`Content-Type: application/json`)
- **Аутентификация:** сессионные cookie (HttpOnly, Secure) или JWT в заголовке `Authorization: Bearer <token>`
- **Коды ответов:** 200, 201, 204, 400, 401, 403, 404, 409, 422, 500

---

## 3. Инфраструктура

- **GitHub Actions** — автоматизированная проверка (CI/CD):
  - Линтинг (flake8, black)
  - Тесты (pytest + coverage)
  - Сборка Docker-образа
  - Публикация в GHCR
- **Docker** — контейнеризация приложения:
  - `Dockerfile` — multi-stage сборка (builder + runtime)
  - `docker-compose.yml` — сервисы: `web`, `db` (PostgreSQL), `scheduler`
- **GHCR** — реестр Docker-образов (`ghcr.io/your-org/accessevent`)
- **Обратный прокси:** Nginx (HTTPS, статика, WebSocket-проксирование)
- **Мониторинг:** health-check `/health`, структурированные логи (JSON)
- **Бэкапы:** ежедневный дамп PostgreSQL, хранение ≥ 30 дней
- **Окружения:** `development`, `staging`, `production` (через `.env` и docker-compose override)

---

## 4. Схема взаимодействия (ASCII)

```
┌─────────────────────────────────────────────────────────────┐
│                    КЛИЕНТ (Browser)                          │
│  HTML5 / CSS3 / JS / Jinja2 / Socket.IO client              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐   │
│  │ Каталог  │ │ Карточка │ │  Чат     │ │ Уведомления  │   │
│  │ событий  │ │ события  │ │ (WS)     │ │ (баннеры)    │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘   │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTPS / WebSocket
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                NGINX (reverse proxy, TLS)                    │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              BACKEND (Flask + Gunicorn)                      │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────┐ │
│  │  Routes    │ │ Services   │ │  Models    │ │ Security │ │
│  │ (Blueprint)│ │ (business) │ │(SQLAlchemy)│ │  (RBAC)  │ │
│  └────────────┘ └────────────┘ └────────────┘ └──────────┘ │
│  ┌────────────┐ ┌────────────┐                              │
│  │ Scheduler  │ │ SocketIO   │                              │
│  │(APScheduler│ │  (chat,    │                              │
│  │ / Celery)  │ │ notifications)                            │
│  └────────────┘ └────────────┘                              │
└────────────────────────┬────────────────────────────────────┘
                         │ SQLAlchemy / psycopg
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  PostgreSQL 15                               │
│  users │ events │ registrations │ messages │ notifications  │
│  event_categories │ event_category_link │ reviews           │
│  accessibility_options                                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Ключевые архитектурные решения

| Решение | Обоснование |
|---------|-------------|
| **MVC + сервисный слой** | Разделение ответственности, тестируемость, переиспользование логики |
| **Blueprint'ы Flask** | Модульность, изоляция доменов (auth, events, chat, admin) |
| **WebSocket для чата** | Реальное время, мгновенная доставка сообщений |
| **APScheduler для уведомлений** | Точная отправка визуальных баннеров по расписанию |
| **PostgreSQL 15** | Надёжность, поддержка JSONB, полнотекстовый поиск |
| **Docker + GHCR** | Воспроизводимость, быстрый деплой, версионирование |
| **WCAG 2.1 AA** | Инклюзивность — ключевое требование проекта |
| **RBAC** | Разграничение доступа: организатор / участник / админ |
