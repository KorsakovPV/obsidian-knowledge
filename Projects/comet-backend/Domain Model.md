---
project: comet-backend
created: 2026-06-25
updated: 2026-08-25
tags: [project, domain, database, sqlalchemy]
---

# Domain Model

#project

Доменная модель [[Overview|comet-backend]]. ORM-модели — в `app/models/`, общий `Base`,
PostgreSQL. Ниже основные таблицы и связи. Бизнес-смысл сущностей см. в [[Overview]],
процессы — в [[Architecture]].

## Связи (кратко)

```
Deal 1───* Approval (версии; ровно одна is_current)
  │            ├──* ApprovalStage ──* ApprovalStageToken ──* ApprovalStageNotification ──* Delivery
  │            │                   └─* ApprovalStageDecisionRequest / ApprovalStageMutationRequest
  │            ├──* ApprovalStageEvent   (ordered audit по sequence)
  │            └──* OfferImplementation  (по одному на offer одобренного snapshot)
  ├──* Offer 1───0..1 OfferOrder  (заказ в ОП)
  └──* DealAttachment

PreTariff 1───* PreTariffComment
          1───* PreTariffAttachment

ClientPrice            (цены клиента)
LkmUser / LkmRole / LkmPermission  (ролевая модель ЛКМ)
```

Согласование живёт только на уровне сделки: offer-level связь `Offer 0..1 Approval` и
служебные таблицы cutover удалены миграцией
`2026_08_06_..._drop_legacy_approval_contract.py`.

## Deal — `deal`

Коммерческое предложение (сделка).

- `client_id`, `contract_id?`, `region_id` — UUID заказчика / договора / региона.
  `contract_id` проставляется при создании сделки и после этого не меняется:
  передан фронтом — сделка коммерческая, не передан — подставляется тестовый
  (`TST*`) договор клиента. Тип договора **не хранится**, он вычисляется по номеру
  договора из customers (см. [[Deal Contract Selection]]).
- `title`, `description?`.
- `seller` — организатор продаж, enum `Seller`: `datafort` | `vimpelcom`.
- `staffers_contractor`, `staffers_customer`, `staffers_partner_customer` —
  списки UUID сотрудников.
- `number` — порядковый номер сделки; `deal_date`, `created_by`, `updated_by?`.
- Связи: `offers` (cascade delete-orphan), `attachments`, `approvals`.

## Offer — `offer`

Оффер: один продукт = один бланк заказа. Принадлежит сделке.

- `product_id` (int) — продукт/услуга; `deal_id` → `deal`.
- `technical_parameters` (JSONB), `tariffs` (JSONB) — тех. параметры и тарифы бланка заказа.
  Тариф хранит присланные клиентом `pk`, `quantity`, `price` (цена продажи),
  `partner_client_price` (цена конечного потребителя), `group_code` и рассчитанные
  сервером `base_price`, `base_price_source`, `discount_percent` — по ним route builder
  выбирает скидочный маршрут. `base_price_source` — `classifier` (прайс) либо
  `client_price` (персональная цена клиента, тариф продан не ниже её и согласования не
  требует, см. [[Discount Base and Personal Price]]).
- `created_by`, `updated_by?`.
- Связи: `deal`, `order` (0..1, `OfferOrder`). Offer-level связи с `approval` больше
  нет — согласование принадлежит сделке.
- КП-фаза оффера (`in_progress` / `formed` / `approval` / `approved` / `returned`)
  **не хранится**: вычисляется из состояния оффера и текущей версии согласования
  сделки — см. [[Offer Order Status]].
- `special_conditions?` — nullable особые условия КП; передаются без преобразований
  в создаваемый заказ ОП.

## Approval — `approval`

Процесс согласования сделки: владелец — `deal_id`, все офферы рассматриваются вместе.

- `deal_id` → `deal`.
- Версии: `version`, `is_current`, `superseded_at`, `workflow_version`
  (`1` — legacy, `2` — staged). Ограничения: `UNIQUE(deal_id, version)` и partial
  unique index по `deal_id WHERE is_current` — у сделки не больше одной текущей версии.
- Предмет согласования: `subject_snapshot` (JSONB со сделкой и всеми офферами),
  `subject_hash`, `route_context` (триггеры маршрута, максимальная скидка и её источник,
  версии builder-а и `STAGE_ORDER`).
- `status` (`ApprovalStatus`):
  `draft` | `pending` | `blocked` | `answered` | `canceled` | `not_required`.
- `decision?` (`ApprovalDecision`): `approve` | `reject`; `decision_comment?`.
- `blocked_permission?`, `blocked_at?` — стадия, для которой не нашёлся согласующий.
- `requested_at?`, `requested_by_user_id?` → `lkm_users` — когда и кем сделка отправлена
  на согласование (событие `started` создаётся системой и actor не содержит), `decided_at?`.
- `reason_type` (`ApprovalReasonType`): `discount` | `product_rule` — legacy scalar-причина;
  полный набор причин лежит в `route_context`.
- `source` (`ApprovalSource`): `email` | `auto_related_deals` | `auto_rule`.
- Legacy-полей offer-level workflow (`offer_id`, `approvers`, `email_token`,
  `email_token_expires_at`) больше нет — сняты contract-фазой Ticket 15. Строки старых
  согласований остались и читаются через `GET /deals/{deal_id}/approvals`.
- Связи: `stages` (сортировка по `position`), `events` (сортировка по `sequence`).
- Маршрут строится по максимальной скидке среди всех offers сделки; особые условия
  любого offer добавляют обязательную цепочку `product_owner → lawyer`.
- После `start` snapshot неизменяем; повторное согласование создаёт следующую version,
  предыдущая помечается `is_current=false`.

## ApprovalStage — `approval_stages`

Явная стадия последовательного согласования.

- `approval_id` → `approval`, `stage_code` (`ApprovalStageCode`):
  `presale` | `lead` | `sales_director` | `finance_director` | `product_owner` | `lawyer`.
- `position` — порядковый номер стадии внутри approval; `required_permission` — permission,
  необходимая для закрытия стадии.
- `status` (`ApprovalStageStatus`): `waiting` | `pending` | `approved` | `rejected` |
  `skipped` | `canceled`.
- `is_optional` — можно ли пропустить стадию; mandatory-стадии нельзя skip даже с правом.
- `scope_product_ids` (JSONB) — продукты, вызвавшие включение стадии. Заполняется только
  у `product_owner` и фиксируется вместе с маршрутом: активация происходит позже, и
  повторный вывод причины дал бы другой состав получателей. Пусто — стадия к продукту
  не привязана.
- `assigned_user_id?` → `lkm_users`, `assigned_email?` — заполняются только для
  единоличной стадии; у групповой остаются `NULL`, получатели считаются по
  `required_permission`.
- `requested_at?`, `due_at?` (SLA стадии), `decided_at?`, `decision?`, `decision_comment?`.
- `skipped_by_user_id?` → `lkm_users`, `skipped_at?`, `skip_reason?`.
- Собственных token-полей у стадии нет: токен живёт в `approval_stage_tokens`.
- Ограничения: уникальны `(approval_id, position)` и `(approval_id, stage_code)`;
  partial unique index по `approval_id WHERE status = 'pending'` не допускает две
  активные стадии в одном процессе.

## ApprovalStageToken — `approval_stage_tokens`

История bearer token стадии: `stage_id`, уникальный `token_hash`, `issued_at`,
`expires_at`, `used_at`, `invalidated_at`. Raw token в БД не хранится. Старые записи
сохраняются для идемпотентной обработки повторных ответов.

## ApprovalStageNotification — `approval_stage_notifications`

Одна логическая notification generation: `stage_id`, `token_id`, immutable payload
snapshot и временно encrypted raw token.

## ApprovalStageNotificationDelivery — `approval_stage_notification_deliveries`

Получатели transactional outbox: `notification_id`, recipient user/email, delivery
status (`pending/sent/failed/canceled`), attempts, `next_attempt_at`, `sent_at`,
`last_error`. Уникальна пара `(notification_id, canonical_recipient_email)`. Для group
stage несколько delivery rows относятся к одной notification/stage/token.

Помимо доставки писем таблица служит источником истории согласующего: по
`recipient_user_id` список стадий отвечает на вопрос «стадию мне присылали». Отсюда индекс
`(recipient_user_id, notification_id)`.

## ApprovalStageEvent — `approval_stage_events`

Неизменяемая ordered история переходов: `stage_id`, `approval_id`, `actor_user_id?`,
`action` (`started` | `approved` | `rejected` | `skipped` | `reassigned` | `canceled` |
`superseded` | `activation_failed` | `activation_retried`), `old_status`, `new_status`,
`reason?`, `metadata` (JSONB), `sequence`, `source` (`system` | `api` | `email`),
`source_id?`. Уникальны `(approval_id, sequence)` и — при заполненном `source_id` —
`(approval_id, stage_id, actor_user_id, source, source_id)`.

`actor_user_id` — единственное место, где хранится автор решения: у групповых стадий
`assigned_user_id` пуст всегда, поэтому список стадий согласующего отвечает по событиям на
вопрос «стадию решил я». Отсюда частичный индекс `(actor_user_id, stage_id)` по решающим
действиям. Пропуск в них не входит: его делает держатель права управления стадиями,
согласующим быть не обязанный.

## Идемпотентность команд — `approval_stage_decision_requests`, `approval_stage_mutation_requests`

Сохранённый результат API-команды в области `(stage_id, actor_user_id, idempotency_key)`:
`decision` / `action` + `payload_hash`, `result_code` и полный `response_payload`.
Повтор возвращает сохранённый ответ и не выполняет переход заново; та же строка ключа с
другим решением даёт `decision_conflict`.

## OfferImplementation — `offer_implementations`

Результат передачи одного оффера в ОП внутри одной одобренной версии согласования.

- `approval_id` → `approval`, `deal_id`, `offer_id`, `approval_version`, `subject_hash?`.
- `external_approval_id` — детерминированный UUIDv5 от пары `(approval_id, offer_id)`:
  ключ идемпотентности заказа в ОП.
- `status` (`pending` | `succeeded` | `failed`), `attempts`, `last_error?`,
  `order_id?`, `order_number?`, `order_status?`, `succeeded_at?`.
- Ограничения: уникальны `(approval_id, offer_id)` и `external_approval_id`.

## OfferOrder — `offer_order`

Связь оффера с заказом во внешней системе ОП (Order Processing).

- `offer_id` → `offer`, `order_id` (UUID в ОП), `order_number`, `order_status?`.
- `synced_at?` — когда статус БЗ последний раз успешно прочитан из ОП. Статус читается
  на каждый запрос сделки, а сохранённое значение служит фоллбеком при недоступном ОП;
  по `synced_at` фронт отличает живое значение от устаревшего. `NULL` — из ОП ни разу
  не перечитывали.
- `kp_phase_code?`, `kp_phase_at?` — снимок КП-фазы оффера на момент создания заказа,
  первая точка мини-таймлайна. Хранится только код: подпись и вид выводятся при отдаче.
- Тип заказа и заголовок статуса **не хранятся**: они приходят из ОП вместе со статусом.
  Подробно — [[Offer Order Status]].

## DealAttachment — `deal_attachment`

Вложение сделки (файл в S3) — фактически **файл КП**: он один на сделку, отдельного
признака нет.

- `deal_id` → `deal`, `file_name`, `storage_key` (ключ в S3), `content_type?`,
  `size?`, `uploaded_by?`.
- Хранится последняя версия: загрузка нового файла заменяет прежний
  (`DealAttachmentService.create` удаляет старые вложения после успешной записи
  нового, в порядке «запись БД → объект S3»; версионирование — отдельным тикетом
  [[KP File Versioning]]). Из-за включённого bucket versioning в MinIO старые
  объекты физически переживают удаление (noncurrent за delete-маркером), но со
  строкой БД теряется `storage_key` — приложение их не видит.
  «Текущее КП» — последнее вложение сделки (`get_latest_by_deal_id`, тай-брейк по
  `id` при равных `created_at`); его же outbox worker прикладывает к письмам стадий
  согласования, а задачи согласующего отдают полем `kp_attachment`.
- Замена и удаление идут под `SELECT ... FOR UPDATE` строки сделки и запрещены при
  активном согласовании (`pending`/`blocked` → 409 `approval_subject_locked`):
  файл зафиксирован, как и snapshot-поля самой сделки. Ограничения БД «одно
  вложение на сделку» нет сознательно — оно противоречило бы будущему
  версионированию; сериализацию конкурентных загрузок даёт та же блокировка.

## PreTariff — `pre_tariff` (+ события, комментарии и вложения)

**Заявка на предтариф**: обвязка Кометы вокруг черновика тарифа, который живёт записью
классификатора. Модель заявки (CM-1, миграция `c6d7e8f9a0b1`) и жизненный цикл — переходы
статусов, права, guard удаления (CM-2) — реализованы 24.08.2026
([[Тикеты: предтариф как черновик тарифа]]); прокси в классификатор — ещё нет.

- `pre_tariff`: `status` (`draft` / `review` / `approved`, дефолт `draft`),
  `classifier_tariff_id?` (pk черновика в классификаторе), `product_id?`,
  `client_id?`, `deal_id?`, `title`, `description` (обоснование заявки),
  `price?` / `cost_price?` (`numeric(10,2)`), `quantity_params` (JSONB, дефолт `{"min": 1}`),
  `decided_by?` / `decided_at?`, `created_by`, `updated_by?`;
  связи `events`, `comments`, `attachments`.
- `pre_tariff_event`: `pre_tariff_id`, `actor`, `actor_role?`, `action`, `old_status?`,
  `new_status` — журнал по образцу `approval_stage_events`, но без `sequence`: события
  одной транзакции упорядочены тай-брейком по `id` (uuid6 монотонен по времени).
- `pre_tariff_comment`: `pre_tariff_id`, `comment`, `created_by`, `updated_by?`.
- `pre_tariff_attachment`: `pre_tariff_id`, `file_name`, `storage_key`, `content_type?`,
  `size?`, `uploaded_by?`.

Ограничения:

- `pre_tariff_status_valid` — check-constraint на три значения статуса. Литералы
  сознательно не генерируются из enum: DDL описывает то, что лежит в базе, иначе новое
  значение молча меняло бы модель без миграции.
- `pre_tariff_classifier_tariff_id_key` — partial unique index (`WHERE ... IS NOT NULL`):
  один черновик классификатора описывается одной заявкой, а пустых ссылок много —
  черновик появляется позже самой заявки.
- `client_id`/`deal_id` — UUID **без FK** (урок DFDEV-1846): удаление сделки не должно
  уносить заявку, тариф переживает сделку и может быть продан другой.

Денормализация `title`/`price`/`cost_price` — чтобы список заявок строился без похода в
классификатор построчно; источник истины по этим полям остаётся классификатор
(`numeric(10,2)` совпадает с его колонками, длина имени ограничена 255 на входе).

Создание заявки пишет событие `created` в той же транзакции — иначе история начиналась бы
с первого перехода. `approved` терминален: опубликованный тариф в черновики не возвращают.

Переходы статусов пишутся тем же запросом, что и меняет статус: строка берётся
`FOR UPDATE`, событие журнала вставляется в той же транзакции. Повтор уже выполненного
перехода событий не добавляет.

`classifier_tariff_id` с 25.08.2026 заполняется всегда: заявка заводится вместе с черновиком
тарифа, а `title`, `price` и `cost_price` денормализуются из отправленного в классификатор
payload. Заявки, созданные до этого, ссылки не имеют — правка их тарифа и апрув отвечают
409 `draft_missing`.

- `pre_tariff_read_marker`: `pre_tariff_id`, `user_email`, `last_read_at`, уникальная пара
  «заявка — читатель». Отметка «докуда дочитал»: счётчик непрочитанного у каждого свой, а
  сообщения в чате общие, поэтому хранится не признак у сообщения, а граница у читателя.
  Читатель опознаётся email — тем же, которым подписаны комментарии.

Ещё не реализовано ([[Предтариф как черновик тарифа]]): флаг черновика в офферах и КП (CM-6).

## ClientPrice — `client_price`

Индивидуальная цена клиента на тариф. Не журнал покупок, а **право**: продажа по цене не
ниже персональной открывает ворота, обнуляет скидку и снимает согласование.

- `client_id`, `tariff_id`, `price` (Decimal), `valid_period` (`TSTZRANGE`, полуинтервал
  `[from, to)`; пустая верхняя граница — бессрочно), `source`.
- `source` (`manual` | `order`) — кто завёл цену. Ручной ввод и автозапись живут по разным
  правилам: ручную цену нельзя молча перекрыть другой ручной, а автоматическую ручной ввод
  вправе подрезать, поглотить или разрезать. Значение хранится строкой под check-constraint,
  прошлые записи проставлены как `manual`.
- `ExcludeConstraint client_price_no_overlap` не даёт периодам одной пары
  `(клиент, тариф)` пересекаться. Из-за него запись всегда обрезает соседний период и
  вставляет новый **одной транзакцией**, а пара сериализуется advisory lock — у первой
  покупки строк ещё нет, и блокировать нечего.
- Автоматически заведённая цена конечна по сроку; бессрочной остаётся только ручная.

## Ролевая модель ЛКМ (LKM)

Личный кабинет менеджера: пользователи, роли, права.

- `lkm_users`: `ad_login`, `email?` (канонический delivery email писем согласования),
  `full_name`, `role` (`UserRole`:
  `manager` | `presale` | `sales_lead` | `sales_director` | `finance_director` | `product_owner` | `lawyer` | `admin`), `first_login_at?`, `created_by_admin?`.
- `lkm_permissions` (`codename`, `name`, 24 записи: 22 из БТ02 плюс
  `skip_kp_approval_stage` и `reassign_kp_approval_stage`), `lkm_roles`
  (`codename`, `name`).
- Связки: `lkm_role_permissions` (набор прав роли), `lkm_user_permissions` (персональные права).
- `lkm_user_permissions.scope_product_id` — зона ответственности назначения: `NULL` даёт
  право на все продукты, заполненный ограничивает его одним продуктом. Уникальность
  задана двумя частичными индексами: обычный ключ с nullable-колонкой не запретил бы два
  глобальных назначения, потому что NULL в Postgres не сравнивается сам с собой.
- Пермиссии/роли и наборы засеваются миграциями `..._add_lkm_permissions.py` и
  `..._bt02_roles_permissions.py`; `..._narrow_approval_roles.py` сузила наборы так, что
  постадийная `approve_kp_*` остаётся ровно у своей роли.
- Эффективные права = пермиссии роли ∪ персональные права. Полная спецификация —
  в исследовании [[LKM Role Model]]; проверка прав — в [[Architecture]].

---

Миграции схемы — Alembic (`migrations/versions/`), см. [[Architecture]].
