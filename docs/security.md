# Подсистема безопасности AccessEvent

## 1. Аутентификация

### Регистрация
- Пользователь указывает: `username`, `email`, `password`, `role` (по умолчанию `user`), `hearing_status` (для адаптации интерфейса)
- Пароль хешируется через **Werkzeug** (`pbkdf2:sha256`) с солью
- Хранится только `password_hash`, пароль в открытом виде не сохраняется и не логируется
- Email проверяется на уникальность и формат (валидация Marshmallow/WTForms)
- Пароль должен соответствовать политике: минимум 8 символов, хотя бы одна буква и одна цифра

### Вход (login)
- Пользователь вводит `email` и `password`
- Сервер находит пользователя по email и проверяет хеш через `check_password_hash()`
- При успехе генерируется **JWT access token** (срок 1 час) и **refresh token** (срок 30 дней)
- Refresh token сохраняется в БД (таблица `refresh_tokens`) для возможности отзыва
- При неверных данных возвращается HTTP 401 без указания, что именно неверно (email или пароль)

### Проверка токена
- Каждый защищённый запрос: заголовок `Authorization: Bearer <access_token>`
- Сервер декодирует JWT, проверяет подпись (HS256) и срок действия (`exp`)
- При истечении access token — обновление через `POST /auth/refresh` с refresh token
- При недействительном/отозванном refresh token — HTTP 401 и принудительный logout

### Выход (logout)
- Клиент удаляет токены; refresh token помечается как отозванный в БД
- Опционально: чёрный список access-токенов до истечения `exp`

---

## 2. Хеширование паролей

- **Метод:** `pbkdf2:sha256` (Werkzeug), количество итераций ≥ 600 000
- **Генерация хеша:**
  ```python
  from werkzeug.security import generate_password_hash
  password_hash = generate_password_hash("pass123", method='pbkdf2:sha256')
  ```
- **Проверка:**
  ```python
  from werkzeug.security import check_password_hash
  check_password_hash(stored_hash, "pass123")  # -> True / False
  ```
- **Правила:**
  - Хеш никогда не возвращается в API-ответах
  - При смене пароля — повторное хеширование и инвалидация всех refresh-токенов
  - В логах пароли и хеши маскируются

---

## 3. Авторизация (RBAC)

### Роли
| Роль | Права |
|------|-------|
| guest | Просмотр событий и категорий, поиск, фильтрация |
| user | Регистрация на события, чат, уведомления, профиль, отзывы |
| organizer | Создание и управление своими событиями, одобрение заявок, загрузка материалов (все права user) |
| admin | Управление пользователями, модерация, категории, блокировки (все права organizer) |

### Наследование прав
```
guest -> user -> organizer -> admin
```
Каждая старшая роль автоматически включает все права младших.

### Матрица доступа (основные ресурсы)

| Ресурс / Действие | guest | user | organizer | admin |
|-------------------|:-----:|:----:|:---------:|:-----:|
| GET /events | ✅ | ✅ | ✅ | ✅ |
| POST /events | ❌ | ❌ | ✅ | ✅ |
| PUT /events/{id} (своё) | ❌ | ❌ | ✅ | ✅ |
| DELETE /events/{id} (своё) | ❌ | ❌ | ✅ | ✅ |
| POST /events/{id}/registrations | ❌ | ✅ | ✅ | ✅ |
| PUT /registrations/{id} (одобрить) | ❌ | ❌ | ✅ | ✅ |
| POST /events/{id}/messages | ❌ | ✅ | ✅ | ✅ |
| POST /events/{id}/reviews | ❌ | ✅ | ✅ | ✅ |
| POST /admin/users/{id}/block | ❌ | ❌ | ❌ | ✅ |
| GET /admin/stats | ❌ | ❌ | ❌ | ✅ |

### Реализация
- Декоратор `@role_required(*roles)` проверяет роль из JWT-payload
- Дополнительно `@ownership_required` проверяет, что организатор владеет ресурсом
- При недостатке прав — **HTTP 403 Forbidden**
- При отсутствии токена — **HTTP 401 Unauthorized**

```python
def role_required(*allowed_roles):
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            payload = get_jwt_payload()
            if not payload or payload.get("role") not in allowed_roles:
                abort(403)
            return f(*args, **kwargs)
        return wrapper
    return decorator
```

---

## 4. JWT (JSON Web Token)

- **Header:**
  ```json
  {"alg": "HS256", "typ": "JWT"}
  ```
- **Payload (access token):**
  ```json
  {
    "user_id": 1,
    "role": "organizer",
    "hearing_status": "deaf",
    "iat": 1699996400,
    "exp": 1700000000
  }
  ```
- **Payload (refresh token):**
  ```json
  {
    "user_id": 1,
    "jti": "unique-token-id",
    "iat": 1699996400,
    "exp": 1702592000
  }
  ```
- **Signature:** HMAC-SHA256 с секретным ключом приложения (`SECRET_KEY`)
- **Правила:**
  - Секретный ключ хранится в переменных окружения (не в коде)
  - Refresh-токены хранятся в БД (`refresh_tokens`) для отзыва
  - Access-токен не хранится на сервере (stateless)
  - Ротация refresh-токенов при каждом обновлении
  - `jti` (JWT ID) используется для отзыва конкретных токенов

---

## 5. Инклюзивные функции безопасности

- Поле `hearing_status` в таблице `users` (`hearing` / `hard_of_hearing` / `deaf`) — адаптация интерфейса под потребности пользователя
- **Визуальные уведомления** (таблица `notifications`) вместо звуковых:
  - 🟡 жёлтый — «скоро начало» (за 30 минут)
  - 🟢 зелёный — «мероприятие началось»
  - 🔴 красный — «отменено/перенесено»
- **Текстовый чат** (таблица `messages`) вместо голосового общения
- **Цветовая индикация статусов**:
  - `pending` — жёлтый
  - `confirmed` / `approved` — зелёный
  - `cancelled` / `rejected` — красный
- **Дублирование информации:** любое системное сообщение имеет текстовый эквивалент (нет звуковых сигналов как единственного способа уведомления)
- **Уведомления о безопасности** (вход с нового устройства, смена пароля) также отображаются визуально в личном кабинете

---

## 6. Защита от угроз (OWASP Top 10)

| Угроза | Мера защиты |
|--------|-------------|
| SQL-инъекции | SQLAlchemy ORM с параметризацией, запрет raw SQL |
| XSS | Экранирование Jinja2, CSP-заголовки, санитизация пользовательского ввода |
| CSRF | CSRF-токены для форм (Flask-WTF), SameSite=Strict для cookie |
| Broken Auth | JWT с коротким TTL, refresh-ротация, bcrypt/pbkdf2, лимит попыток входа |
| Sensitive Data Exposure | HTTPS (TLS 1.2+), HttpOnly/Secure cookie, маскирование логов |
| Broken Access Control | RBAC + `@ownership_required`, проверка прав на каждом эндпоинте |
| Security Misconfiguration | Docker с non-root пользователем, `.env` вне репозитория, минимальные образы |
| Insecure Deserialization | Только JSON, без `pickle`/`eval` |
| Insufficient Logging | Структурированные логи (JSON), аудит критичных действий |
| SSRF | Валидация URL при загрузке материалов, whitelist домены |

---

## 7. Безопасность данных

- **Транспорт:** только HTTPS (TLS 1.2+), HSTS-заголовок
- **Хранение:**
  - Пароли — только хеши (`pbkdf2:sha256`)
  - Токены — refresh в БД, access — stateless
  - Персональные данные — минимально необходимый набор
- **Резервное копирование:** ежедневный `pg_dump`, шифрование бэкапов, хранение ≥ 30 дней
- **Доступ к БД:** только из внутренней Docker-сети, отдельный пользователь с минимальными правами
- **Аудит:** логирование входов, смены пароля, блокировок, изменений мероприятий

---

## 8. Безопасность в CI/CD

- Секреты хранятся в **GitHub Secrets**, не в коде
- Сканирование зависимостей (`pip-audit`, Dependabot)
- Линтинг безопасности (`bandit`)
- Docker-образы сканируются (`trivy`) перед публикацией в GHCR
- Ветка `main` защищена: обязательный PR review и прохождение CI
