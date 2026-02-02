# webhook-subscription-test

Backend тестовое задание  
Часть 1. Схема данных, уникальности, индексы, статусы

USERS  : 
Поля :

id UUID PK
email TEXT NOT NULL
created_at TIMESTAMPTZ
updated_at TIMESTAMPTZ

Уникальности : 
UNIQUE(email) - 1 пользователь на email.

Индексы : 
PK(id)
UNIQUE(email) (это же индекс) - быстрый поиск пользователя по email.

SUBSCRIPTIONS : 
Поля  : 

id UUID PK
user_id UUID FK -> users.id
status ENUM('INACTIVE','ACTIVE','CANCELED')
current_period_end TIMESTAMPTZ
created_at TIMESTAMPTZ
updated_at TIMESTAMPTZ

Уникальности :
UNIQUE(user_id) - в MVP одна “текущая” подписка на пользователя (мы её продлеваем).

Индексы :
UNIQUE(user_id) (уже индекс)
INDEX(status, current_period_end) - выборки активных/истекающих подписок.


 PAYMENTS :

Ключевой принцип: status=SUCCEEDED (провайдер) ≠ “подписка уже продлена”.
Поэтому фиксируем отдельно факт “применено к подписке”.

Поля : 
id UUID PK
provider TEXT (например "tilda")
external_payment_id TEXT (orderId/paymentId)
user_id UUID FK -> users.id NULL (может быть NULL = unlinked)
email TEXT NULL
amount DECIMAL(12,2)
currency TEXT
status ENUM('RECEIVED','SUCCEEDED','FAILED','REFUNDED') -статус провайдера
paid_at TIMESTAMPTZ NULL
subscription_applied_at TIMESTAMPTZ NULL - когда платёж реально применили к подписке
subscription_id UUID NULL
created_at TIMESTAMPTZ

Уникальности :
UNIQUE(provider, external_payment_id)- идемпотентность оплаты: повторный webhook не создаст дубль payment.

Индексы :
INDEX(user_id, created_at DESC) -история оплат пользователя
INDEX(status) - поиск проблемных/необработанных
INDEX(email) - связывание unlinked оплат с user по email (если пришёл email в вебхуке)


WEBHOOK_EVENTS :

Поля :
id UUID PK
provider TEXT
external_event_id TEXT NULL
external_payment_id TEXT NULL
payload JSONB (сырой JSON)
payload_hash TEXT (sha256 от rawBody) - дедуп, если нет external_event_id
signature_valid BOOLEAN
status ENUM('RECEIVED','VALIDATED','PROCESSED','IGNORED','FAILED_RETRYABLE','FAILED_FINAL')
received_at TIMESTAMPTZ
processed_at TIMESTAMPTZ NULL
error_code TEXT NULL
error_message TEXT NULL

Уникальности :
UNIQUE(provider, external_event_id) WHERE external_event_id IS NOT NULL — дедуп событий по eventId
UNIQUE(provider, payload_hash) - дедуп событий, если eventId отсутствует

Индексы :
INDEX(external_payment_id) - быстро найти события по orderId/paymentId
INDEX(status, received_at) - мониторинг зависших/ошибочных событий



Часть 2. Webhook handler - полный псевдокод

Endpoint: POST /webhooks/payments

0) Прочитать тело и распарсить
rawBody = readRawBody()
payload = tryParseJson(rawBody)
IF payload invalid:
  return 400

1) Базовая валидация
IF payload.external_payment_id missing: return 400
IF payload.status missing: return 400

provider = "tilda"
externalPaymentId = payload.external_payment_id
externalEventId = payload.event_id ?? NULL
payloadHash = sha256(provider + rawBody)

2) Idempotency inbox: сохранить webhook_event (дедуп) + конкретно про “дубль”
Идея: сначала кладём событие в webhook_events. Если это дубль не обрабатываем заново.
eventRowId = NULL

- 2.1 Пытаемся вставить событие (дедуп по event_id, если он есть)
IF externalEventId IS NOT NULL:
  eventRowId = INSERT INTO webhook_events(
    provider, external_event_id, external_payment_id, payload, payload_hash,
    signature_valid, status, received_at
  )
  VALUES (
    :provider, :externalEventId, :externalPaymentId, :payload, :payloadHash,
    NULL, 'RECEIVED', now()
  )
  ON CONFLICT (provider, external_event_id) DO NOTHING
  RETURNING id

- 2.2 Если event_id нет -> дедуп по payload_hash
ELSE:
  eventRowId = INSERT INTO webhook_events(
    provider, external_event_id, external_payment_id, payload, payload_hash,
    signature_valid, status, received_at
  )
  VALUES (
    :provider, NULL, :externalPaymentId, :payload, :payloadHash,
    NULL, 'RECEIVED', now()
  )
  ON CONFLICT (provider, payload_hash) DO NOTHING
  RETURNING id

- 2.3 Если id не вернулся => это дубль, читаем существующую запись
IF eventRowId IS NULL:
  existing = SELECT * FROM webhook_events
             WHERE provider=:provider AND
               (
                 (external_event_id IS NOT NULL AND external_event_id=:externalEventId)
                 OR
                 (external_event_id IS NULL AND payload_hash=:payloadHash)
               )

  eventRowId = existing.id

  IF existing.status IN ('PROCESSED','IGNORED','FAILED_FINAL'):
    return 200
  - иначе продолжаем (например: RECEIVED / VALIDATED / FAILED_RETRYABLE)

3) Проверить подпись/secret
IF verifySignature(rawBody, headers) == false:
  UPDATE webhook_events SET
    signature_valid=false,
    status='FAILED_FINAL',
    processed_at=now(),
    error_code='INVALID_SIGNATURE',
    error_message='signature check failed'
  WHERE id = eventRowId
  return 401/403

UPDATE webhook_events SET
  signature_valid=true,
  status='VALIDATED'
WHERE id = eventRowId

4) Неуспешный статус оплаты → игнор
IF payload.status NOT IN ('paid','succeeded'):
  UPDATE webhook_events SET
    status='IGNORED',
    processed_at=now(),
    error_code='NON_SUCCESS_STATUS'
  WHERE id = eventRowId
  return 200

5) Основная логика (транзакция)
Требования:
повторный webhook не создаёт дубль payment
не продлевает подписку второй раз
если сервер упал посередине — повторный webhook безопасно восстановит

BEGIN TRANSACTION

- 5.1 Upsert payment по (provider, external_payment_id) + блокировка строки
payment = SELECT * FROM payments
          WHERE provider=:provider AND external_payment_id=:externalPaymentId
          FOR UPDATE

IF payment not found:
  payment = INSERT INTO payments(
    provider, external_payment_id,
    user_id, email,
    amount, currency,
    status, paid_at,
    subscription_applied_at, subscription_id,
    created_at
  )
  VALUES (
    :provider, :externalPaymentId,
    NULL, (payload.email ?? NULL),
    payload.amount, payload.currency,
    'SUCCEEDED', (payload.paid_at ?? now()),
    NULL, NULL,
    now()
  )
  RETURNING *
ELSE:
  payment = UPDATE payments SET
    status = forwardOnlyStatus(payments.status, payload.status),
    paid_at = COALESCE(payments.paid_at, payload.paid_at),
    email = COALESCE(payments.email, payload.email)
  WHERE id=payment.id
  RETURNING *

- 5.2 Привязка payment -> user
IF payment.user_id IS NULL AND payment.email IS NOT NULL:
  user = SELECT * FROM users WHERE email=payment.email
  IF user found:
    UPDATE payments SET user_id=user.id WHERE id=payment.id
    payment.user_id = user.id

- user неизвестен -> не теряем событие, откладываем
IF payment.user_id IS NULL:
  UPDATE webhook_events SET
    status='FAILED_RETRYABLE',
    processed_at=now(),
    error_code='USER_MISSING'
  WHERE id = eventRowId
  COMMIT
  return 200

- 5.3 Найти/создать subscription и заблокировать
sub = SELECT * FROM subscriptions
      WHERE user_id=payment.user_id
      FOR UPDATE

IF sub not found:
  sub = INSERT INTO subscriptions(
    user_id, status, current_period_end, created_at, updated_at
  )
  VALUES (
    payment.user_id, 'INACTIVE', now(), now(), now()
  )
  RETURNING *

- 5.4 EXACTLY-ONCE apply (фикс гонки / защита от повторов)
applied = UPDATE payments SET
  subscription_applied_at=now(),
  subscription_id=sub.id
WHERE id=payment.id
  AND subscription_applied_at IS NULL
RETURNING id

IF applied empty:
  UPDATE webhook_events SET
    status='PROCESSED',
    processed_at=now()
  WHERE id = eventRowId
  COMMIT
  return 200

- 5.5 Продлить подписку
base = MAX(sub.current_period_end, now())
newEnd = base + planDuration(payload.plan_id OR defaultPlan)

UPDATE subscriptions SET
  status='ACTIVE',
  current_period_end=newEnd,
  updated_at=now()
WHERE id=sub.id

- 5.6 Завершить событие
UPDATE webhook_events SET
  status='PROCESSED',
  processed_at=now()
WHERE id = eventRowId

COMMIT
return 200

6) Ошибки
ON db/network error:
  ROLLBACK
  return 500    чтобы провайдер ретраил

Коды ответов :
200 - обработали / дубль / проигнорировали / приняли и отложили (retryable)
400 - сломанный payload (JSON / нет external_payment_id / нет status)
401/403 - неверная подпись/secret
500 - временная ошибка (БД/сеть), нужно ретраить

Восстановление :
Вся критичная логика в транзакции.

Exactly-once применяем платёж через:
UPDATE payments ... WHERE subscription_applied_at IS NULL
Если сервер упал до продления подписки - subscription_applied_at останется NULL, и повторная обработка применит платёж ровно один раз.

Recovery (переобработка retryable)
Есть фоновая джоба (каждые N минут), которая переобрабатывает webhook_events со статусом FAILED_RETRYABLE (например USER_MISSING):

events = SELECT id, payload FROM webhook_events
         WHERE status='FAILED_RETRYABLE'
         ORDER BY received_at
         LIMIT 100
         FOR UPDATE SKIP LOCKED

FOR each event in events:
  повторно запускаем ту же обработку на основе payload
  (upsert payment + apply once через subscription_applied_at IS NULL)

  IF успешно:
    UPDATE webhook_events SET status='PROCESSED', processed_at=now()
  ELSE:
    оставить FAILED_RETRYABLE (или backoff/attempts)
Так вебхук не теряется, даже если он пришёл до создания user, и будет применён позже безопасно и один раз.



Часть 3. Edge cases (что делаем в каждом кейсе)

Webhook пришёл дважды :
payments не дублируется: UNIQUE(provider, external_payment_id).
В транзакции:
если payments.subscription_applied_at IS NOT NULL → уже применено → 200.
если status=SUCCEEDED, но subscription_applied_at IS NULL → apply через UPDATE ... WHERE subscription_applied_at IS NULL → 200.
webhook_events: первый раз PROCESSED, повтор - безопасно → 200.

Webhook пришёл раньше создания user:
payment создаём/обновляем как unlinked (user_id=NULL, сохраняем email если есть).
Подписку не трогаем (нет user_id).
webhook_events = FAILED_RETRYABLE (USER_MISSING) и 200.
Recovery воркером при появлении user.

Webhook без email, но есть externalPaymentId : 
payment создаём/обновляем (user_id=NULL, email=NULL) — платёж не теряем, но привязать не можем.
Подписку не активируем.
webhook_events = FAILED_RETRYABLE (UNLINKED_PAYMENT) и 200.
Recovery возможен при появлении способа связать платёж (другое событие/процесс/ручная привязка), иначе остаётся unlinked.

Webhook с другой суммой, чем план : 
Подписку не активируем/не продлеваем.
webhook_events = FAILED_FINAL (AMOUNT_MISMATCH) + alert/manual review.
Возвращаем 200 .

Webhook пришёл через неделю : 
Обрабатываем как обычно.
Продление считаем от base = max(current_period_end, now()), чтобы задержка не уменьшила срок.
Логируем late_webhook=true → 200.


Сервер упал после записи payment, но до subscription:
При повторной доставке / reprocess:
если subscription_applied_at IS NULL → доприменить ровно один раз → 200
если subscription_applied_at IS NOT NULL → дубль → 200



Часть 4. Debuggability и наблюдаемость
Логи (минимально достаточно, чтобы расследовать инциденты)
Обязательные поля логов (structured JSON log):
request_id (correlation id), provider
webhook_event_id
external_event_id (если есть), external_payment_id
event_key (dedup key: provider:eventId или provider:payloadHash)
signature_valid
incoming_status, amount, currency
email, user_id (если найден)
result = processed | ignored | duplicate | unlinked | failed_retryable | failed_final
payment_status_before -> payment_status_after
subscription_end_before -> subscription_end_after
subscription_applied_before -> subscription_applied_after (по subscription_applied_at)
event_age_ms (если в payload есть время события; иначе null)
error_code, error_message (если ошибка)
duration_ms

Хранение payload:
сырой payload сохраняется в webhook_events.payload (JSONB) для расследований.
payload не логируем целиком в runtime-логах (не засорять и не тащить PII), но в БД он есть.


Метрики :
Счётчики:
webhook_requests_total{provider,http_code}
webhook_processed_total{provider}
webhook_duplicate_total{provider}
webhook_failed_retryable_total{provider}
webhook_failed_final_total{provider}
webhook_invalid_signature_total{provider}
payments_unlinked_total{provider}
payments_amount_mismatch_total{provider}
webhook_recovered_total{provider}

Латентность/лаги:
webhook_processing_duration_ms{provider} (p95/p99)
webhook_processing_lag_ms{provider} = processed_at - received_at

Backlog / “зависшие”:
webhook_events_backlog{status}

Алерты :
рост 5xx или рост FAILED_RETRYABLE → проблемы БД/сети/транзакций
invalid_signature > 0 или резкий рост → неправильный secret / атака
payments_unlinked_total растёт или “unlinked older than X” → оплаты не привязываются
amount_mismatch > 0 → ошибка тарифов/подозрительные оплаты
webhook_processed_total == 0 за N минут при наличии входящих → обработчик сломан
рост p95 processing_duration → деградация
events stuck: FAILED_RETRYABLE backlog или RECEIVED/VALIDATED older than X
SUCCEEDED && subscription_applied_at IS NULL older than X → риск “деньги есть, подписки нет”
