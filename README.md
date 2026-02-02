# webhook-subscription-test

Backend тестовое задание

## Содержание
- Часть 1. Схема данных, уникальности, индексы, статусы
- Часть 2. Webhook handler — полный псевдокод
- Часть 3. Edge cases
- Часть 4. Debuggability и наблюдаемость

---

## Часть 1. Схема данных, уникальности, индексы, статусы

### USERS

**Поля**
- `id` UUID PK
- `email` TEXT NOT NULL
- `created_at` TIMESTAMPTZ
- `updated_at` TIMESTAMPTZ

**Уникальности**
- `UNIQUE(email)` — 1 пользователь на email.

**Индексы**
- `PK(id)`
- `UNIQUE(email)` (уникальность = индекс) — быстрый поиск пользователя по email.

---

### SUBSCRIPTIONS

**Поля**
- `id` UUID PK
- `user_id` UUID FK -> `users.id`
- `status` ENUM('INACTIVE','ACTIVE','CANCELED')
- `current_period_end` TIMESTAMPTZ
- `created_at` TIMESTAMPTZ
- `updated_at` TIMESTAMPTZ

**Уникальности**
- `UNIQUE(user_id)` — в MVP одна “текущая” подписка на пользователя (мы её продлеваем).

**Индексы**
- `UNIQUE(user_id)` (уже индекс)
- `INDEX(status, current_period_end)` — выборки активных/истекающих подписок.

---

### PAYMENTS

Ключевой принцип: `status=SUCCEEDED` (провайдер) ≠ “подписка уже продлена”.
Поэтому фиксируем отдельно факт “применено к подписке”.

**Поля**
- `id` UUID PK
- `provider` TEXT (например "tilda")
- `external_payment_id` TEXT (orderId/paymentId)
- `user_id` UUID FK -> `users.id` NULL (может быть NULL = unlinked)
- `email` TEXT NULL
- `amount` DECIMAL(12,2)
- `currency` TEXT
- `status` ENUM('RECEIVED','SUCCEEDED','FAILED','REFUNDED') — статус провайдера
- `paid_at` TIMESTAMPTZ NULL
- `subscription_applied_at` TIMESTAMPTZ NULL — когда платёж реально применили к подписке
- `subscription_id` UUID NULL
- `created_at` TIMESTAMPTZ

**Уникальности**
- `UNIQUE(provider, external_payment_id)` — идемпотентность оплаты: повторный webhook не создаст дубль payment.

**Индексы**
- `INDEX(user_id, created_at DESC)` — история оплат пользователя
- `INDEX(status)` — поиск проблемных/необработанных
- `INDEX(email)` — связывание unlinked оплат с user по email (если пришёл email в вебхуке)

---

### WEBHOOK_EVENTS

**Поля**
- `id` UUID PK
- `provider` TEXT
- `external_event_id` TEXT NULL
- `external_payment_id` TEXT NULL
- `payload` JSONB (сырой JSON)
- `payload_hash` TEXT (sha256 от rawBody) — дедуп, если нет external_event_id
- `signature_valid` BOOLEAN
- `status` ENUM('RECEIVED','VALIDATED','PROCESSED','IGNORED','FAILED_RETRYABLE','FAILED_FINAL')
- `received_at` TIMESTAMPTZ
- `processed_at` TIMESTAMPTZ NULL
- `error_code` TEXT NULL
- `error_message` TEXT NULL

**Уникальности**
- `UNIQUE(provider, external_event_id) WHERE external_event_id IS NOT NULL` — дедуп событий по eventId
- `UNIQUE(provider, payload_hash)` — дедуп событий, если eventId отсутствует

**Индексы**
- `INDEX(external_payment_id)` — быстро найти события по orderId/paymentId
- `INDEX(status, received_at)` — мониторинг зависших/ошибочных событий

---

## Часть 2. Webhook handler — полный псевдокод

Endpoint: `POST /webhooks/payments`

### 0) Прочитать тело и распарсить
- `rawBody = readRawBody()`
- `payload = tryParseJson(rawBody)`
- если JSON невалидный → `return 400`

### 1) Базовая валидация
- если нет `payload.external_payment_id` → `400`
- если нет `payload.status` → `400`
provider = "tilda"
externalPaymentId = payload.external_payment_id
externalEventId = payload.event_id ?? NULL
payloadHash = sha256(provider + rawBody)


### 2) Idempotency inbox: сохранить webhook_event (дедуп) + “дубль”
Сначала кладём событие в `webhook_events`. Если дубль — не обрабатываем заново.

- 2.1 Если `externalEventId != NULL`:
  - INSERT `webhook_events` с `ON CONFLICT (provider, external_event_id) DO NOTHING RETURNING id`

- 2.2 Если `externalEventId == NULL`:
  - INSERT `webhook_events` с `ON CONFLICT (provider, payload_hash) DO NOTHING RETURNING id`

- 2.3 Если `id` не вернулся → дубль:
  - читаем существующую запись по ключу (event_id или payload_hash)
  - если статус `PROCESSED | IGNORED | FAILED_FINAL` → `return 200`
  - иначе продолжаем (например `RECEIVED | VALIDATED | FAILED_RETRYABLE`)

### 3) Проверить подпись/secret
- если `verifySignature(rawBody, headers) == false`:
  - `signature_valid=false`
  - `status='FAILED_FINAL'`
  - `error_code='INVALID_SIGNATURE'`
  - `return 401/403`

Иначе:
- `signature_valid=true`
- `status='VALIDATED'`

### 4) Неуспешный статус оплаты → игнор
Если `payload.status NOT IN ('paid','succeeded')`:
- `status='IGNORED'`
- `error_code='NON_SUCCESS_STATUS'`
- `return 200`

### 5) Основная логика (транзакция)
Требования:
- повторный webhook не создаёт дубль payment
- не продлевает подписку второй раз
- если сервер упал посередине — повторный webhook безопасно восстановит

**BEGIN TRANSACTION**

#### 5.1 Upsert payment по (provider, external_payment_id) + блокировка строки
- `SELECT ... FOR UPDATE`
- если не найден:
  - INSERT в `payments` со статусом `SUCCEEDED`, `user_id=NULL`, `email` из payload (если есть)
- если найден:
  - UPDATE: `status` (forward-only), `paid_at` (COALESCE), `email` (COALESCE)

#### 5.2 Привязка payment -> user
Если `payment.user_id IS NULL AND payment.email IS NOT NULL`:
- найти `users` по email
- если найден → записать `payments.user_id`

Если `payment.user_id IS NULL`:
- `webhook_events.status='FAILED_RETRYABLE'`
- `error_code='USER_MISSING'`
- **COMMIT**
- `return 200`

#### 5.3 Найти/создать subscription и заблокировать
- `SELECT subscriptions WHERE user_id=... FOR UPDATE`
- если нет:
  - INSERT `subscriptions` со статусом `INACTIVE`, `current_period_end=now()`

#### 5.4 EXACTLY-ONCE apply (защита от повторов)
applied = UPDATE payments SET
subscription_applied_at=now(),
subscription_id=sub.id
WHERE id=payment.id
AND subscription_applied_at IS NULL
RETURNING id


Если `applied` пустой (уже применён):
- `webhook_events.status='PROCESSED'`
- **COMMIT**
- `return 200`

#### 5.5 Продлить подписку
- `base = MAX(sub.current_period_end, now())`
- `newEnd = base + planDuration(payload.plan_id OR defaultPlan)`
- UPDATE `subscriptions`:
  - `status='ACTIVE'`
  - `current_period_end=newEnd`

#### 5.6 Завершить событие
- `webhook_events.status='PROCESSED'`
- `processed_at=now()`

**COMMIT**
`return 200`

### 6) Ошибки
- при db/network ошибке: `ROLLBACK` и `return 500` (чтобы провайдер ретраил)

**Коды ответов**
- `200` — обработали / дубль / проигнорировали / приняли и отложили (retryable)
- `400` — сломанный payload (JSON / нет external_payment_id / нет status)
- `401/403` — неверная подпись/secret
- `500` — временная ошибка (БД/сеть), нужно ретраить

### Восстановление (переобработка retryable)
Фоновая джоба (каждые N минут) переобрабатывает `webhook_events` со статусом `FAILED_RETRYABLE`:

- SELECT событий `FAILED_RETRYABLE` `FOR UPDATE SKIP LOCKED`
- для каждого события повторить ту же обработку на основе `payload`
- успех → `webhook_events.status='PROCESSED'`, иначе оставить `FAILED_RETRYABLE` (или backoff/attempts)

---

## Часть 3. Edge cases (что делаем в каждом кейсе)

### Webhook пришёл дважды
- `payments` не дублируется: `UNIQUE(provider, external_payment_id)`
- если `subscription_applied_at IS NOT NULL` → уже применено → 200
- если `status=SUCCEEDED`, но `subscription_applied_at IS NULL` → применяем через `UPDATE ... WHERE subscription_applied_at IS NULL` → 200
- `webhook_events`: первый раз `PROCESSED`, повтор — безопасно → 200

### Webhook пришёл раньше создания user
- payment создаём/обновляем как unlinked (`user_id=NULL`, сохраняем `email` если есть)
- подписку не трогаем (нет `user_id`)
- `webhook_events = FAILED_RETRYABLE (USER_MISSING)` и 200
- recovery воркером при появлении user

### Webhook без email, но есть externalPaymentId
- payment создаём/обновляем (`user_id=NULL`, `email=NULL`) — платёж не теряем
- подписку не активируем
- `webhook_events = FAILED_RETRYABLE (UNLINKED_PAYMENT)` и 200
- recovery возможен при появлении способа связать платёж, иначе остаётся unlinked

### Webhook с другой суммой, чем план
- подписку не активируем/не продлеваем
- `webhook_events = FAILED_FINAL (AMOUNT_MISMATCH)` + alert/manual review
- возвращаем 200

### Webhook пришёл через неделю
- обрабатываем как обычно
- продление считаем от `base = max(current_period_end, now())`, чтобы задержка не уменьшила срок
- логируем `late_webhook=true` → 200

### Сервер упал после записи payment, но до subscription
- при повторной доставке / reprocess:
  - если `subscription_applied_at IS NULL` → доприменить ровно один раз → 200
  - если `subscription_applied_at IS NOT NULL` → дубль → 200

---

## Часть 4. Debuggability и наблюдаемость

### Логи (минимально достаточно для расследований)
Structured JSON log (обязательные поля):
- `request_id` (correlation id), `provider`
- `webhook_event_id`
- `external_event_id` (если есть), `external_payment_id`
- `event_key` (dedup key: provider:eventId или provider:payloadHash)
- `signature_valid`
- `incoming_status`, `amount`, `currency`
- `email`, `user_id` (если найден)
- `result` = processed | ignored | duplicate | unlinked | failed_retryable | failed_final
- `payment_status_before -> payment_status_after`
- `subscription_end_before -> subscription_end_after`
- `subscription_applied_before -> subscription_applied_after` (по `subscription_applied_at`)
- `event_age_ms` (если в payload есть время события; иначе null)
- `error_code`, `error_message` (если ошибка)
- `duration_ms`

**Хранение payload**
- сырой payload сохраняется в `webhook_events.payload` (JSONB) для расследований
- payload целиком не логируем в runtime-логах (PII/объём), но он есть в БД

### Метрики

**Счётчики**
- `webhook_requests_total{provider,http_code}`
- `webhook_processed_total{provider}`
- `webhook_duplicate_total{provider}`
- `webhook_failed_retryable_total{provider}`
- `webhook_failed_final_total{provider}`
- `webhook_invalid_signature_total{provider}`
- `payments_unlinked_total{provider}`
- `payments_amount_mismatch_total{provider}`
- `webhook_recovered_total{provider}`

**Латентность/лаги**
- `webhook_processing_duration_ms{provider}` (p95/p99)
- `webhook_processing_lag_ms{provider}` = `processed_at - received_at`

**Backlog / “зависшие”**
- `webhook_events_backlog{status}`

### Алерты
- рост `5xx` или рост `FAILED_RETRYABLE` → проблемы БД/сети/транзакций
- `invalid_signature > 0` или резкий рост → неправильный secret / атака
- растёт `payments_unlinked_total` или “unlinked older than X” → оплаты не привязываются
- `amount_mismatch > 0` → ошибка тарифов/подозрительные оплаты
- `webhook_processed_total == 0` за N минут при наличии входящих → обработчик сломан
- рост p95 `processing_duration` → деградация
- события “stuck”: `FAILED_RETRYABLE` backlog или `RECEIVED/VALIDATED older than X`
- `SUCCEEDED && subscription_applied_at IS NULL older than X` → риск “деньги есть, подписки нет”



