---
project: comet-backend
created: 2026-08-05
tags: [project, devops, security, k8s, ci, secrets]
---

# DevOps Secrets and Deploy

#project #devops #security

Связано: [[Stand Testing 2026-08-05]], [[Architecture]], [[Approval Debug Mode]].

Задача для девопса: перенос секретов comet-backend из git в управляемые
Kubernetes Secrets **с перевыпуском** всех засвеченных значений, плюс починка
деплой-пайплайна, из-за которой 2026-08-05 лежал тестовый стенд.

## Инцидент, с которого всё началось

3 августа в `k8s/manifests/base-test.yaml` добавили Secret `approval-workflow-key`,
но в кластер его никто не применил: `base*.yaml` живёт вне пайплайна и накатывается
руками один раз при создании контура. Следующий деплой пересоздал под →
`CreateContainerConfigError: secret "approval-workflow-key" not found` → из-за
`strategy: Recreate` + `replicas: 1` старый под уже убит → стенд лежал до ручного
`kubectl create secret`. Прод устроен идентично и не падает только потому, что туда
секрет донесли руками; **любой следующий секрет повторит инцидент на каждом контуре**.

## Часть 1. Инвентаризация: что засвечено в git

Всё перечисленное лежит в репозитории в открытом виде (base64 — не шифрование) и
уже отправлено в origin, поэтому **ротация обязательна**, простого переноса мало.

Из `k8s/manifests/base.yaml` / `base-test.yaml`:

| Секрет | Что внутри |
|---|---|
| `pg-key` | пароль PostgreSQL |
| `approval-workflow-key` | Fernet-ключ шифрования approval outbox |
| `ors-registry-cred` | dockerconfigjson доступа к registry |

Из `k8s/configmaps/configmap-comet-backend.yaml` и `-test.yaml` (оба контура!):

| Переменная | Что это |
|---|---|
| `KEYCLOAK_CLIENT_SECRET` | клиентский секрет Keycloak |
| `CLASSIFICATOR_PASSWORD` | пароль сервиса классификатора |
| `CUSTOMERS_PASSWORD` | пароль сервиса customers |
| `BITRIX_WEBHOOK` | вебхук Bitrix24 (в URL зашит токен) |
| `ORDER_PROCESSING_X_API_KEY` | API-ключ Order Processing |
| `S3_ACCESS_KEY`, `S3_SECRET_KEY` | ключи S3 |
| `SMTP_PASSWORD`, `IMAP_PASSWORD` | почтовые пароли |

## Часть 2. Целевая схема

1. **Все значения из таблиц — в Kubernetes Secrets**, конфигмапы оставляют только
   несекретное (URL, имена, флаги). Приложение готово: все переменные читаются из
   env, достаточно перевести их в деплойменте с `configMapRef` на `secretKeyRef`
   (по образцу уже существующих `DB_PASSWORD` и `APPROVAL_WORKFLOW_OUTBOX_ENCRYPTION_KEY`).
2. **Секреты создаются деплой-джобой из GitLab CI variables** (masked + protected),
   идемпотентно:
   ```
   kubectl create secret generic <name> --from-literal=KEY="$CI_VAR" \
     -n $namespace --dry-run=client -o yaml | kubectl apply -f -
   ```
   Тогда «новый секрет» = CI-переменная + строка в джобе, и забыть донести его в
   кластер невозможно. (Вариант «применять base*.yaml в пайплайне» отвергнут: apply
   перетёр бы живые креды значениями из git и увековечил секреты в репозитории.)
3. Из `base*.yaml` секреты удаляются (остаются namespace, RBAC, Redis); из
   конфигмапов — все значения из таблицы выше. После этого git больше не содержит
   ни одного секрета.

## Часть 3. Ротация — порядок и подводные камни

- **`OUTBOX_ENCRYPTION_KEY` (Fernet)** — ротация делает нечитаемыми уже
  зашифрованные, но недоставленные payload'ы outbox. Код переживает это штатно
  (повреждённый ciphertext отменяет только свою доставку), но менять ключ нужно в
  окно, когда `approval_stage_notification_deliveries` без pending-строк, иначе
  зависшие письма согласования будут отменены.
- **Пароль PostgreSQL** — согласованно с изменением пароля роли в самой БД
  (`ALTER ROLE`), на каждом контуре свой.
- **`CUSTOMERS_PASSWORD`, `CLASSIFICATOR_PASSWORD`, `ORDER_PROCESSING_X_API_KEY`** —
  перевыпуск на стороне владельцев этих сервисов; для ОП это ключ, по которому нас
  аутентифицируют, координация с их командой.
  Дополнение 2026-08-11: в тестовой конфигмапе теперь засвечена и учётка
  `apigateway` для customers-**dev** (временное переключение стенда, см.
  [[Restore Customers-T on Test Stand]]) — ротировать вместе с остальными.
- **`KEYCLOAK_CLIENT_SECRET`** — regenerate в клиенте `comet-backend-api`.
- **`BITRIX_WEBHOOK`** — перевыпуск вебхука в Bitrix24 (старый URL отзывается).
- **S3, SMTP/IMAP** — перевыпуск у соответствующих владельцев.
- **`ors-registry-cred`** — сменить пароль технической УЗ registry.

История git: значения уже в origin, чистить историю смысла мало — считать их
скомпрометированными и заменить все. После ротации старые значения в истории
безвредны.

## Часть 4. Починка пайплайна (`k8s/pipelines/deploy*.yaml`)

1. **Preflight до apply** — деплой падает до убийства старого пода, а не после:
   ```
   kubectl -n $namespace get secret pg-key approval-workflow-key ors-registry-cred ...
   ```
2. **`alembic upgrade head` перенести из `after_script` в `script`** после
   `rollout status`: сейчас он выполняется даже при упавшем rollout (вторая ошибка
   маскирует первую) и не выполняется тогда, когда нужен. Лучше — initContainer с
   миграциями: схема гарантированно обновляется до старта кода.
3. **Readiness probe** в деплойменты (`GET /api/healthcheck`): сейчас «available» =
   «контейнер создан», упавшее на старте приложение rollout не заметит.

## Критерии приёмки

- `git grep` по репозиторию не находит ни одного живого секрета.
- Все старые значения перевыпущены; старые отозваны/деактивированы.
- Деплой на чистый namespace поднимает контур без единого ручного `kubectl`.
- Деплой при отсутствующей CI-переменной падает на preflight, старый под жив.
- Миграции не выполняются при упавшем rollout и выполняются при успешном.
