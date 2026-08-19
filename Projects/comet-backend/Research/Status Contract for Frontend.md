---
project: comet-backend
created: 2026-08-06
updated: 2026-08-19
source: docs/status_contract_for_frontend.md
tags: [project, research, frontend-contract, status, deal, offers, orders]
---

# Status Contract for Frontend

#research #status #frontend-contract

Связано: [[DFDEV-2052 Deal and Offer Status Contract]], [[Offer Order Status]],
[[API]], [[Domain Model]], [[Stand Testing 2026-08-05]].


Справка для фронтенда по DFDEV-2052/DFDEV-2200: какие статусные данные уже есть в API,
в каких полях, с какого момента жизни сделки они появляются и чего пока нет. Всё
перечисленное проверено живьём на `comet-backend-test` 2026-08-05 (версии 0.1.62–64).

Коротко: **уже можно собрать** бейдж согласования сделки, дровер с маршрутом и
решениями, КП-статус оффера и блок заказа с мини-таймлайном. **Нельзя пока** —
статусную строку этапов сделки по БТ01 («Этап 4 из 9») и вкладку «История».

## Когда какой статус появляется

| Шаг | Что произошло | `approval_summary.status` | КП-фаза оффера (`offers[].status`) |
|---|---|---|---|
| 1 | `POST /deals` — сделка создана | `not_required` (записи согласования ещё нет) | офферов нет |
| 2 | `POST /offers` — первый оффер | `draft` (маршрут построен) либо `not_required` (маршрут пуст) | `formed`; без тарифов — `in_progress` |
| 3 | `POST /deals/{id}/approval/request` | `pending`, заполнен `active_stage` | `approval` + `stage_code` |
| 4 | стадии решают | `approved` / `rejected` / `blocked` | `approved` / `returned` / `approval` |
| 5 | `POST /deals/{id}/start-implementation` | без изменений | без изменений; у оффера появляется `order`. Частичный провал — 422 с `failed_offers`, см. [[Видимая причина провала имплементации]] |

Согласование рождается **при первом оффере** (маршрут строится из тарифов), а не при
создании сделки и не при отправке: request только запускает уже существующий draft.
Пока процесс `pending`/`blocked`, snapshot-поля сделки и офферов менять нельзя — бэк
ответит 409 `approval_subject_locked`.

## Статусы согласования сделки — есть

`GET /deals`, `GET /deals/{deal_id}`:

- `approval_summary.status` — агрегат: `not_required | draft | pending | blocked |
  approved | rejected | canceled`;
- `approval_summary.active_stage` — код стадии, позиция, `required_permission`,
  `assigned_user_id/email` (у единоличной), `requested_at`, `due_at` (SLA);
- `approval_summary.completed_stages[]` — пройденные стадии с решением,
  `decision_comment`, `decided_at`; `skipped_stages`, `canceled_stages`,
  `stage_counts_by_status`;
- `approval_summary.blocked_permission` + `blocked_at` — на ком процесс встал;
- `current_approval` — полная текущая версия процесса;
- `actions` — доступность действий (отправить/отозвать/имплементация/редактирование)
  уже с учётом прав пользователя и состояния процесса: фронту не нужно вычислять
  доступность кнопок самому.

Рядом: история версий `GET /deals/{deal_id}/approvals?limit&offset`, задачи
согласующего `GET /approval-stages/pending`, решения/skip/reassign —
`POST /approval-stages/{stage_id}/...`.

## КП-фаза оффера — есть

`offers[].status` (и в карточке сделки, и в `/offers`) — общая форма
`{code, title, kind, stage_code}`:

| `code` | `title` | `kind` | когда |
|---|---|---|---|
| `in_progress` | В работе | `progress` | у оффера нет тарифов |
| `formed` | Сформировано | `progress` | согласование `draft/not_required/canceled` |
| `approval` | На согласовании | `waiting` | `pending/blocked`; заполнен `stage_code` |
| `approved` | КП согласовано | `success` | одобрено |
| `returned` | Возвращено в работу | `returned` | отклонено |

Поле обязательное (не nullable), вычисляется на лету — у старых офферов есть всегда.
`stage_code` приходит **кодом** (`presale`, `lead`, `sales_director`,
`finance_director`, `product_owner`, `lawyer`) — русские названия стадий рисует
фронт, бэк их не отдаёт. Вне фазы `approval` поле `stage_code = null`. Для
заблокированного процесса стадия тоже заполнена (восстанавливается по
`blocked_permission`).

Фаза «Отправлено клиенту» не реализована — события нет в бэке.

`offers[].approval_ref` — ссылка на сделочное согласование (`approval_id`, `version`,
`status`, `subject_hash`); собственного процесса у оффера нет.

## Блок заказа — есть (кроме живого ITSM-заголовка на тестовом стенде)

`offers[].order` (`null`, пока заказ не создан):

```json
{
  "order_id": "…", "order_number": "70195",
  "status": null,                    // null = БЗ создан, ITSM ещё не ответил
  "order_type": "commercial",        // test | inner | commercial, из ОП
  "order_type_title": "Коммерческий",
  "kp_phase": {                      // снимок КП-фазы на момент создания БЗ
    "code": "approved", "title": "КП согласовано",
    "kind": "success", "at": "2026-08-05T14:33:09Z"
  },
  "created_at": "2026-08-05T14:33:11Z",
  "order_status": null,              // deprecated → status.code
  "synced_at": "2026-08-05T14:35:08Z" // deprecated → status.synced_at
}
```

- `status` заполненный — `{code, title, kind: "external", link, synced_at}`: код ITSM,
  русский заголовок и ссылка на БЗ приходят из ОП;
- мини-таймлайн карточки собирается из одного ответа:
  `kp_phase.at` → `created_at` → `status`;
- **деградация**: если ОП недоступен, приходит последнее сохранённое значение —
  форма та же, признак свежести `status.synced_at` (чип «Данные ITSM устарели»
  считает фронт по нему). В этом режиме `title` равен коду и `link = null`;
- `order_type` при недоступном ОП — `null` (не хранится);
- `kp_phase = null` у заказов, созданных до 2026-08-05 (снимка тогда не было);
- `order_status`/`synced_at` верхнего уровня — deprecated, дублируют `status.*` на
  переходный период (релиз фронта + спринт).

## Чего нет (и когда будет)

| Чего нет | Что это | Статус |
|---|---|---|
| `deal.status` / `stage` / `blocker` / `forecast_close_date` | статусная строка «Этап N из 9» по БТ01, макеты 3a–3b | этап 3 нарезки; заблокирован скелетом от аналитика (9 этапов, 32 статуса, правила переходов) |
| `client_title`, `created_by_name`, `history_events_count` в сделке | шапка карточки без N+1 | этап 1, не начат |
| `GET /deals/{id}/history` | вкладка «История» | этап 5, после этапов 3–4 |
| `changed_at` в статусах | «N дн. в статусе» | требует решения об источнике времени; для заказа — доработки ОП |

До появления `deal.status` статус сделки в списках можно показывать из
`approval_summary.status` — сейчас это единственный статус уровня сделки.
