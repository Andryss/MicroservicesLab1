# Декомпозиция системы на микросервисы

## 0. Допущения

- **Клиенты и бонусные счета**
  - у клиента один глобальный ИД
  - но отдельный бонусный счет в каждой программе

- **Владение программами и участие компаний**
  - у программы есть компания-владелец
  - другие компании могут подключаться к программе с ее разрешения
  - одна компания может владеть своей программой и участвовать в чужих

- **Операции и изоляция балансов**
  - компании подключаются к программе, а операции проводят по счетам ее клиентов
  - баланс счёта общий для участников одной программы
  - балансы разных программ независимы, обмена бонусами между ними нет

- **Задержка обновления истории**
  - после подтверждения начисления или списания операция может появиться в истории позже обновления счёта

- **Профиль входящей нагрузки**
  - 50 000 RPS: 85% чтений (42 500 RPS), 15% записей (7 500 RPS)
  - [данные Rakuten Points за 2023 год][rakuten-load]: 50 млрд справочных, 8 млрд транзакционных запросов

[rakuten-load]: https://static.pingcap.co.jp/files/2024/09/26135458/HTAP-Summit-2024-Rakuten.pdf#page=8

## 1. Context Map

![Context Map](img/context-map-20261112-233401.svg)

Стрелка направлена от источника информации к её получателю.

<details>
<summary>Исходник PlantUML и генерация PNG/SVG</summary>

```plantuml
@startuml ContextMap
left to right direction
skinparam ranksep 28
skinparam nodesep 42

rectangle "**Customer Management**\n<size:12>Идентичность и профили клиентов</size>" as customers
rectangle "**Loyalty Programs**\n<size:12>Владельцы и настройки программ</size>" as programs
rectangle "**Partner Participation**\n<size:12>Допуск и права компаний</size>" as partners
rectangle "**Bonus Accounts**\n<size:12>Баланс и операции по счёту</size>" as accounts
rectangle "**Bonus Rules**\n<size:12>Правила расчёта начислений</size>" as rules
rectangle "**Loyalty Levels**\n<size:12>Уровни и прогресс клиентов</size>" as levels
rectangle "**Campaigns and Promo Codes**\n<size:12>Управление акциями и промокодами</size>" as campaigns
rectangle "**Notifications**\n<size:12>Доставка уведомлений</size>" as notifications

customers --> accounts : Идентичность клиента
customers --> notifications : Контактные данные
programs -down-> partners : Владелец\nпрограммы
programs --> rules : Настройки программы
partners --> accounts : Участие в программе\nи права на операции
partners --> rules : Права на изменение правил
partners --> campaigns : Права на управление\nакциями
campaigns --> accounts : Условия акций\nи промокодов
accounts -left-> rules : Параметры начисления\nи условия акции
rules -right-> accounts : Рассчитанное начисление
accounts --> levels : Проведённые\nоперации
levels --> rules : Уровень клиента
accounts --> notifications : Изменения баланса
levels --> notifications : Изменения\nуровня клиента
@enduml
```

```bash
# Из корня репозитория; нужна команда plantuml.
format=svg # или png
image="docs/img/context-map-$(date +%Y%m%d-%H%M%S).$format"
sed -n '/^@startuml ContextMap$/,/^@enduml$/p' docs/decomposition.md > /tmp/microservices-context-map.puml
plantuml -t"$format" -pipe < /tmp/microservices-context-map.puml > "$image"
```

</details>

## 2. Каталог микросервисов

### 2.1. Customer Management

```text
Service: Customer Management (контекст: Customer Management)

Responsibility:
единая идентичность клиента, профиль и контактные данные.
При регистрации merchantId сверяется с компанией из проверенной учётной записи вызывающего.

Owned Data:
Customer (профиль, контакты, привязки к компаниям и внешним идентификаторам)

API:
- POST /customers — зарегистрировать клиента
  Запрос:
    {
      "name": "Анна",
      "contacts": {
        "email": "anna@example.test",
        "phone": "+12025550142"
      },
      "merchantId": "shop-8",
      "externalCustomerId": "ext-42"
    }
  Ответ:
    {
      "customerId": "cust-42"
    }

- GET /customers/{customerId} — получить профиль и контакты
  Ответ:
    {
      "name": "Анна",
      "contacts": {
        "email": "anna@example.test",
        "phone": "+12025550142"
      }
    }

- PATCH /customers/{customerId} — обновить профиль или контакты
  Запрос:
    {
      "contacts": {
        "email": "anna.new@example.test",
        "phone": "+12025550143"
      }
    }
  Ответ:
    OK

Dependencies:
in:
- Нет
out:
- Bonus Accounts : идентичность клиента для открытия счёта
- Notifications : контакты клиента для доставки уведомлений

Communication:
REST - API профиля клиента и получение контактных данных.

Scalability:
Горизонтальное масштабирование сервиса.
Шардирование профилей по customerId; поиск профиля и контактов по первичному ключу.
Уникальный индекс (merchant_id, external_customer_id) для сопоставления с customerId.
```

### 2.2. Program and Partner Management

```text
Service: Program and Partner Management (контексты: Loyalty Programs, Partner Participation)

Responsibility:
создание программ, регистрация компаний, одобрение участия и управление правами партнёров. Создание
программы включает участие её владельца.

Owned Data:
LoyaltyProgram (компания-владелец)
Merchant (название и юридические реквизиты)
ProgramMembership (статус участия компании и её права в программе)

API:
- POST /merchants — зарегистрировать компанию
  Запрос:
    {
      "name": "Магазин",
      "legalName": "ООО «Пример»",
      "inn": "<ИНН>",
      "ogrn": "<ОГРН>",
      "legalAddress": "<юридический адрес>"
    }
  Ответ:
    {
      "merchantId": "shop-7"
    }

- POST /programs — создать программу
  Запрос:
    {
      "name": "Бонусы за покупки",
      "ownerMerchantId": "shop-7",
      "settings": {
        "currency": "RUB"
      }
    }
  Ответ:
    {
      "programId": "prog-1"
    }

- POST /programs/{programId}/memberships — подать заявку на участие
  Запрос:
    {
      "merchantId": "shop-8"
    }
  Ответ:
    {
      "status": "PENDING"
    }

- PATCH /programs/{programId}/memberships/{merchantId} — изменить статус или права участника
  Запрос:
    {
      "status": "ACTIVE",
      "permissions": [
        "ACCRUE",
        "REDEEM"
      ]
    }
  Ответ:
    OK

- POST /permissions/check — проверить право на действие в программе
  Запрос:
    {
      "programId": "prog-1",
      "merchantId": "shop-7",
      "action": "ACCRUE"
    }
  Ответ:
    {
      "allowed": true
    }

Dependencies:
in:
- Нет
out:
- Bonus Accounts : участие компании в программе и права на операции
- Loyalty Rules and Levels : настройки программы и права на изменение правил
- Campaigns and Promo Codes : участие компании и права управления акциями

Communication:
REST - API программ и компаний, управление участием и проверка прав.
Kafka - передача настроек созданных программ в Loyalty Rules and Levels.

Scalability:
Горизонтальное масштабирование сервиса.
Уникальный индекс (program_id, merchant_id) для проверки участия и прав компании в программе.
```

### 2.3. Bonus Accounts

```text
Service: Bonus Accounts (контекст: Bonus Accounts)

Responsibility:
открытие счёта клиента в программе, баланс, проведение начислений и списаний; защита
от повторных операций и отрицательного остатка.

Owned Data:
LoyaltyAccount (баланс клиента в конкретной программе)
Transaction (начисление или списание, компания и результат операции)

API:
- POST /programs/{programId}/accounts — открыть бонусный счёт в указанной программе
  Запрос:
    {
      "customerId": "cust-42"
    }
  Ответ:
    {
      "accountId": "acc-42"
    }

- GET /accounts/{accountId} — получить счёт и текущий баланс
  Ответ:
    {
      "programId": "prog-1",
      "customerId": "cust-42",
      "balance": 1200
    }

- POST /accounts/{accountId}/accruals — начислить бонусы за покупку
  Запрос:
    Idempotency-Key: order-816
    {
      "purchaseAmount": 3000,
      "currency": "RUB",
      "promoCode": "WELCOME" // при наличии
    }
  Ответ:
    {
      "operationId": "op-900",
      "points": 300,
      "balance": 1500
    }

- POST /accounts/{accountId}/redemptions — списать бонусы
  Запрос:
    Idempotency-Key: order-817
    {
      "points": 300
    }
  Ответ:
    {
      "operationId": "op-901",
      "balance": 1200
    }

Dependencies:
in:
- Customer Management : идентичность клиента
- Program and Partner Management : участие компании и права на операции
- Loyalty Rules and Levels : результат расчёта начисления и уровень клиента
- Campaigns and Promo Codes : условия применения акций и промокодов
out:
- Loyalty Rules and Levels : операции для пересчёта уровня и условия акции для расчёта бонусов
- Operation History : проведённые операции по счёту
- Notifications : изменения баланса клиента

Communication:
REST - API счетов, начислений и списаний; проверка прав и получение условий начисления.
Kafka - передача проведённых операций для истории, пересчёта уровней и уведомлений.

Scalability:
Горизонтальное масштабирование сервиса.
Счёт и его операции хранятся на одном шарде. Размещение счёта определяется парой (program_id, customer_id),
по accountId можно определить его шард.
Уникальный индекс (program_id, customer_id) для открытия одного счёта клиента в программе.
Уникальный индекс (account_id, merchant_id, idempotency_key) для защиты от повторного проведения операции.
Индекс outbox (created_at, id) для выборки сообщений к публикации.
```

### 2.4. Loyalty Rules and Levels

```text
Service: Loyalty Rules and Levels (контексты: Bonus Rules, Loyalty Levels)

Responsibility:
расчёт начисления по правилам программы, уровню клиента и условиям акции; определение и пересчёт
уровней.

Owned Data:
Rule
LoyaltyLevel
CustomerLevel (уровень и прогресс клиента в конкретной программе)

API:
- POST /programs/{programId}/calculations — рассчитать начисление
  Запрос:
    {
      "customerId": "cust-42",
      "purchaseAmount": 3000,
      "currency": "RUB",
      "campaignTerms": { // при наличии
        "bonusMultiplier": 2
      }
    }
  Ответ:
    {
      "points": 300
    }

- PUT /programs/{programId}/rules/{ruleId} — сохранить правило начисления
  Запрос:
    {
      "conditions": {
        "minPurchaseAmount": 1000
      },
      "reward": {
        "percent": 5
      }
    }
  Ответ:
    OK

- GET /programs/{programId}/customers/{customerId}/level — получить уровень и прогресс
  Ответ:
    {
      "level": "SILVER",
      "progress": {
        "earnedPoints": 1500
      }
    }

Dependencies:
in:
- Program and Partner Management : настройки программы и права на изменение правил
- Bonus Accounts : проведённые операции и проверенные условия акции
out:
- Bonus Accounts : результат расчёта начисления и уровень клиента
- Notifications : изменения уровня клиента

Communication:
REST - управление правилами, расчёт начисления и получение уровня клиента.
Kafka - получение настроек программ и операций по счетам, передача изменений уровня.

Scalability:
Горизонтальное масштабирование сервиса.
Отдельная инсталляция Redis: набор действующих правил по programId.
При промахе читаем из БД и сохраняем с TTL; после изменения правил или настроек программы сбрасываем запись.
Индекс (program_id, status) для загрузки действующих правил программы.
Уникальный индекс (program_id, customer_id) для чтения и обновления уровня клиента.
```

### 2.5. Campaigns and Promo Codes

```text
Service: Campaigns and Promo Codes (контекст: Campaigns and Promo Codes)

Responsibility:
управление акциями и промокодами. При проверке кода учитываются срок действия, компания и условия покупки.

Owned Data:
Campaign (программа, условия и компании-участники)
PromoCode (программа, условия, срок действия и компании-участники; код уникален в программе)

API:
- POST /programs/{programId}/campaigns — создать акцию
  Запрос:
    {
      "name": "Двойные бонусы",
      "terms": {
        "bonusMultiplier": 2
      },
      "merchantIds": [
        "shop-8"
      ]
    }
  Ответ:
    {
      "campaignId": "campaign-1"
    }

- POST /programs/{programId}/promocodes — создать промокод
  Запрос:
    {
      "code": "WELCOME",
      "expiresAt": "2027-12-31T23:59:59Z",
      "terms": {
        "bonusMultiplier": 2
      },
      "merchantIds": [
        "shop-8"
      ]
    }
  Ответ:
    {
      "promoCodeId": "promo-1"
    }

- POST /programs/{programId}/promocodes/validate — проверить применимость промокода к покупке
  Запрос:
    {
      "code": "WELCOME",
      "merchantId": "shop-8",
      "purchaseAmount": 3000,
      "currency": "RUB"
    }
  Ответ:
    {
      "applicable": true,
      "campaignTerms": {
        "bonusMultiplier": 2
      }
    }

Dependencies:
in:
- Program and Partner Management : участие компании и права управления акциями
out:
- Bonus Accounts : условия применения акций и промокодов

Communication:
REST - API акций и промокодов, проверка условий их применения.

Scalability:
Горизонтальное масштабирование сервиса.
Отдельная инсталляция Redis: определения акций по (programId, campaignId), промокодов — по (programId, code).
При промахе читаем из БД и сохраняем с TTL; после изменений сбрасываем затронутые записи.
Уникальный индекс (program_id, code) для поиска промокода и уникальности кода внутри программы.
```

### 2.6. Notifications

```text
Service: Notifications (контекст: Notifications)

Responsibility:
формирование и доставка уведомлений об изменениях баланса и уровня. Сервис отдельно обрабатывает события,
повторяет неудачные отправки и изолирует бонусные операции от сбоев провайдеров доставки.

Owned Data:
Notification (получатель, канал, статус и время следующей попытки доставки)

API:
- GET /notifications/{notificationId} — получить статус доставки
  Ответ:
    {
      "channel": "EMAIL",
      "status": "DELIVERED"
    }

Dependencies:
in:
- Bonus Accounts : изменения баланса клиента
- Loyalty Rules and Levels : изменения уровня клиента
- Customer Management : контакты клиента
out:
- Messaging Providers : уведомления для внешней доставки клиентам

Communication:
REST - API статуса уведомления и получение контактов клиента.
Kafka - получение изменений баланса и уровня для подготовки уведомлений.
HTTPS - передача уведомлений внешним провайдерам доставки.

Scalability:
Горизонтальное масштабирование обработчиков уведомлений.
Индекс (status, next_attempt_at) для выборки уведомлений, готовых к повторной отправке.
```

### 2.7. Operation History

```text
Service: Operation History (проекция контекста Bonus Accounts, без собственного контекста)

Responsibility:
построение и чтение долговременной истории операций в отдельном от транзакционных данных хранилище.
Чтение истории и обработку событий можно масштабировать отдельно от проведения бонусных операций.
История доступна только клиенту по его счёту. При чтении сервис сверяет ID клиента с владельцем счёта в проекции.

Owned Data:
OperationHistory (счёт, программа, клиент, компания и проведённые операции)

API:
- GET /accounts/{accountId}/history — получить историю своего счёта от новых операций к старым
  Запрос:
    ?cursor=cursor-1  // для следующей страницы
  Ответ:
    {
      "operations": [
        {
          "operationId": "op-901",
          "type": "REDEMPTION",
          "points": 300,
          "occurredAt": "2026-11-13T10:00:00Z",
          "merchantId": "shop-8"
        }
      ],
      "nextCursor": "cursor-2"
    }

Dependencies:
in:
- Bonus Accounts : проведённые операции по счёту
out:
- Нет

Communication:
REST - чтение истории счёта его владельцем.
Kafka - получение проведённых операций для построения истории.

Scalability:
Горизонтальное масштабирование сервиса.
Шардирование БД по бонусным счетам.
Индекс B-tree (account_id, occurred_at DESC, operation_id DESC) для курсорной пагинации по счёту.
Уникальный индекс (account_id, operation_id) для защиты от дублей при повторной доставке событий.
```

### 2.8. API Gateway

```text
Service: API Gateway (без доменного контекста)

Responsibility:
единая точка входа для прикладного API и входа в систему.
Маршруты входа и выдачи токенов проксирует в Keycloak без требования токена доступа.
На прикладных маршрутах проверяет токен доступа и направляет вызов нужному сервису.

Owned Data:
Нет

API:
- GET /accounts/{accountId} — пример внешнего маршрута к Bonus Accounts
  Запрос:
    Authorization: Bearer <access_token>
  Ответ:
    {
      "programId": "prog-1",
      "customerId": "cust-42",
      "balance": 1200
    }

Dependencies:
in:
- Keycloak : публичные ключи для проверки токенов (JWKS)
out:
- Keycloak : запросы входа и выдачи токенов
- Прикладные сервисы : запросы с проверенной идентичностью вызывающего

Communication:
HTTPS / REST - маршрутизация прикладных запросов.
HTTPS - проксирование маршрутов входа и выдачи токенов, получение JWKS.

Scalability:
Горизонтальное масштабирование реплик.
Публичные ключи хранятся локально в памяти реплик с периодическим обновлением.
Ориентир — 50 000 RPS.
```

### 2.T. Команды

| Команда | Сервисы | Зона ответственности |
| --- | --- | --- |
| 1 | `Bonus Accounts`, `Operation History` | Проведение операций и их отображение в истории |
| 2 | `Program and Partner Management`, `Campaigns and Promo Codes` | Программы и акции |
| 3 | `Loyalty Rules and Levels` | Расчёт начислений и прогресс клиента |
| 4 | `Customer Management`, `Notifications` | Профили, контакты и доставка уведомлений |
| 5 | `API Gateway`, Keycloak (SSO), общая Kafka | Доступность и мониторинг |

## 3. ADR-001. Единое владение бонусным балансом

### 3.1. Контекст

Ожидаемая нагрузка: 20 млн клиентов и 50 000 RPS. По допущению это 42 500 запросов на чтение
и 7 500 запросов на запись в секунду.

Части системы должны масштабироваться независимо. История операций хранится отдельно от транзакционных данных
и по допущению может обновляться позже баланса.

Счёт клиента в программе используется разными компаниями-участниками. Нужно определить владельца баланса
и границы его ответственности при начислении и списании бонусов.

### 3.2. Выбранное решение

Единственным владельцем счетов и их балансов является `Bonus Accounts`. Он открывает счета клиентов в программах,
проводит начисления и списания, проверяет остаток и защищает от повторного проведения операции.

Проверка остатка, изменение баланса и фиксация результата операции выполняются одной транзакцией в его БД.
Остальные сервисы не изменяют баланс напрямую.

Расчёт бонусов, применение промокодов, пересчёт уровней, долговременная история и доставка уведомлений
остаются за пределами ответственности `Bonus Accounts`. Он проводит только операции с балансом.

Остальные сервисы получают сведения о подтверждённых изменениях баланса через асинхронные события или выгрузки операций
и самостоятельно обновляют свои данные. Ответ о проведении операции не ожидает обработки её результатов потребителями.

Для публикации событий используется паттерн Transactional Outbox: операция и сообщение фиксируются одной транзакцией,
а отправка в брокер выполняется после её завершения.

### 3.3. Альтернативы

1. **Объединить баланс, расчёт бонусов, промокоды и уровни в `Bonus Accounts`**
   - **Преимущества:** меньше межсервисных взаимодействий при начислении, связанные изменения выполняются локально.
   - **Недостатки:** изменение правил и маркетинговых условий связано с выпуском сервиса счетов.
     Расчёт бонусов и проведение операций используют общие ресурсы и масштабируются вместе.
2. **Разделить начисления и списания между сервисами с доступом к общей БД счетов**
   - **Преимущества:** независимое развёртывание обработчиков, проверка остатка и изменение баланса в транзакции общей БД.
   - **Недостатки:** общая схема и правила проведения операций требуют согласования между сервисами.
     Ответственность за корректность баланса распределена, а БД остаётся общим ограничением масштабирования.
3. **Разделить начисления и списания между сервисами с собственными БД**
   - **Преимущества:** обработчики и их хранилища можно развёртывать и масштабировать независимо.
   - **Недостатки:** для проверки остатка нужна координация изменений одного счёта между сервисами.
     Согласованность баланса и защита от повторов усложняются из-за распределения данных.
4. **Выделить отдельный сервис учёта для каждой программы**
   - **Преимущества:** программы изолированы по данным и ресурсам, их учёт можно развивать независимо.
   - **Недостатки:** нужно поддерживать реализацию учёта и инфраструктуру для каждой программы.
     Участников необходимо направлять в сервис нужной программы, сохраняя единый счёт клиента внутри неё.

### 3.4. Обоснование выбора

- Единый владелец баланса обеспечивает одну точку проведения операций разных компаний по общему счёту клиента
  и позволяет проверять остаток и изменять его в одной транзакции.
- Для системы с 20 млн клиентов счёт остаётся самостоятельной единицей учёта. Владение балансом не требует
  включать в сервис счетов все процессы, использующие сведения о клиенте и его операциях.
- При целевых 50 000 RPS расчёт правил, чтение истории и доставка уведомлений могут масштабироваться отдельно
  от изменения баланса. Ресурсы каждого сервиса подбираются под фактическую долю запросов.
- События и выгрузки позволяют потребителям самостоятельно обрабатывать изменения без участия `Bonus Accounts`
  в их бизнес-процессах. Отдельное хранение истории выполняет прямое требование задания.
- Граница владения данными остаётся явной: только `Bonus Accounts` принимает решения о проведении операции
  и хранит актуальный баланс счёта.

### 3.5. Последствия

- **Положительные:** явный владелец баланса, локальная согласованность начислений и списаний, независимое
  развитие остальных сервисов. Недоступность потребителя событий не требует ожидать его при проведении операции.
- **Отрицательные:** недоступность `Bonus Accounts` блокирует изменение баланса. Нужны надёжная публикация,
  обработка повторов и контроль отставания потребителей. Сбой публикации увеличивает outbox,
  а выгрузки создают дополнительную нагрузку на БД счетов.
- **Согласованность:** данные об истории, уровнях и уведомлениях могут отставать от актуального баланса.
  Допустимую задержку необходимо согласовать с бизнес-требованиями.
- **Проверка производительности:** единое владение не определяет число реплик или производительность БД
  и не гарантирует выдержки заданной нагрузки. Нужны профиль запросов и нагрузочные испытания.

### 3.6. Итоговые C4-диаграммы

#### 3.6.1. Контекстная диаграмма

![Контекстная диаграмма C4 — Loyalty Platform](img/c4-context-20261115-162502.svg)

<details>
<summary>Исходник PlantUML и генерация PNG/SVG</summary>

```plantuml
@startuml C4Context
!include <C4/C4_Context>
left to right direction
title Контекстная диаграмма — Loyalty Platform

Person(customer, "Клиент", "Накапливает и тратит бонусы у компаний-участников")
Person(manager, "Представитель компании", "Управляет своей программой и участием в других программах")
System_Ext(trading, "Торговые системы", "Клиентские и административные каналы компаний-участников")
System(platform, "Loyalty Platform", "Общие счета клиентов внутри программы; правила, уровни и промокоды")
System_Ext(sso, "Внешний SSO-провайдер", "Подтверждает личность при входе в кабинет")
System_Ext(messaging, "Провайдеры уведомлений", "Доставляют сообщения клиентам")

Rel(customer, trading, "Накапливает и тратит бонусы")
Rel(customer, platform, "Входит в личный кабинет и смотрит баланс и историю")
Rel(manager, platform, "Управляет программами, партнёрами, правилами и акциями")
Rel(trading, platform, "Ведёт клиентов, программы, правила и промокоды; проводит бонусные операции")
Rel(platform, sso, "Использует для входа и получения идентификатора пользователя")
Rel(platform, messaging, "Передаёт уведомления об изменениях баланса и уровня")
Rel(messaging, customer, "Доставляет уведомления")

SHOW_LEGEND()
@enduml
```

```bash
# Из корня репозитория; нужна команда plantuml со стандартной библиотекой C4.
format=svg # или png
image="docs/img/c4-context-$(date +%Y%m%d-%H%M%S).$format"
sed -n '/^@startuml C4Context$/,/^@enduml$/p' docs/decomposition.md > /tmp/microservices-c4-context.puml
plantuml -t"$format" -pipe < /tmp/microservices-c4-context.puml > "$image"
```

</details>

#### 3.6.2. Контейнерная диаграмма

![Контейнерная диаграмма C4 — Loyalty Platform](img/c4-container-20261115-162502.svg)

<details>
<summary>Исходник PlantUML и генерация PNG/SVG</summary>

```plantuml
@startuml C4Container
!pragma layout elk
!include <C4/C4_Container>
LAYOUT_TOP_DOWN()
title Контейнерная диаграмма — Loyalty Platform

Person(customer, "Клиент", "Накапливает и тратит бонусы у компаний-участников")
Person(manager, "Представитель компании", "Управляет программами и участием компаний")
System_Ext(trading, "Торговые системы", "Клиентские операции и управление программами компаний")
System_Ext(sso, "Внешний SSO-провайдер", "Подтверждает личность при входе в кабинет")
System_Ext(messaging, "Провайдеры уведомлений", "Доставляют сообщения клиентам")

System_Boundary(platform, "Loyalty Platform") {
  Container(gateway, "API Gateway", "HTTPS", "Единая точка входа, проверка токенов и маршрутизация")
  Container(keycloak, "Keycloak", "OAuth 2.0 / OIDC", "Вход через внешнего провайдера и токены для API платформы")
  Container(customers, "Customer Management", "REST-сервис", "Идентичность клиента и контакты")
  Container(programs, "Program and Partner Management", "REST-сервис", "Программы, участие компаний и права")
  Container(accounts, "Bonus Accounts", "REST-сервис", "Единственный владелец счетов, балансов и результатов операций")
  Container(rules, "Loyalty Rules and Levels", "REST-сервис", "Расчёт бонусов и уровни клиентов")
  Container(campaigns, "Campaigns and Promo Codes", "REST-сервис", "Условия акций и промокодов")
  Container(notifications, "Notifications", "Сервис", "Подготовка, доставка и повторы уведомлений")
  Container(history, "Operation History", "REST-сервис", "Долговременная история операций и её чтение")

  ContainerQueue(events, "Platform Events", "Kafka", "Топики настроек, операций по счетам и изменений уровней")
  ContainerDb(rules_cache, "Rules Cache", "Redis", "Действующие правила по программе")
  ContainerDb(campaigns_cache, "Campaigns Cache", "Redis", "Определения акций и промокодов")

  ContainerDb(customers_db, "Customer DB", "PostgreSQL", "Customer: профиль и контакты")
  ContainerDb(programs_db, "Program DB", "PostgreSQL", "LoyaltyProgram, Merchant, ProgramMembership")
  ContainerDb(accounts_db, "Account DB", "PostgreSQL", "LoyaltyAccount, Transaction и outbox")
  ContainerDb(rules_db, "Rules and Levels DB", "PostgreSQL", "Rule, LoyaltyLevel, CustomerLevel")
  ContainerDb(campaigns_db, "Campaign DB", "PostgreSQL", "Campaign, PromoCode")
  ContainerDb(notifications_db, "Notification DB", "PostgreSQL", "Notification: состояние доставки")
  ContainerDb(history_db, "History DB", "PostgreSQL", "OperationHistory: отдельно от транзакционных данных")
}

Rel(customer, trading, "Накапливает и тратит бонусы")
Rel_U(messaging, customer, "Доставляет уведомления")
Rel(customer, gateway, "Входит в кабинет и смотрит баланс и историю", "HTTPS")
Rel(manager, gateway, "Входит и управляет программами", "HTTPS")
Rel(trading, gateway, "Получает токен интеграции и вызывает API", "HTTPS")
Rel(gateway, keycloak, "Проксирует вход и выдачу токенов; получает JWKS", "HTTPS")
Rel(keycloak, sso, "Обменивает код и получает ID пользователя", "OAuth 2.0")

Rel(gateway, customers, "Профили клиентов", "REST")
Rel(gateway, programs, "Программы и участие компаний", "REST")
Rel(gateway, campaigns, "Акции и промокоды", "REST")
Rel(gateway, rules, "Правила начисления и уровни", "REST")
Rel(gateway, accounts, "Счета, баланс, начисления и списания", "REST")
Rel(gateway, history, "Передаёт запрос клиента на историю его счёта", "REST")
Rel(gateway, notifications, "Статус доставки", "REST")

Rel(accounts, customers, "Проверяет идентичность клиента", "REST")
Rel(accounts, programs, "Проверяет участие компании и права на операцию", "REST")
Rel(accounts, rules, "Запрашивает расчёт бонусов", "REST")
Rel(accounts, campaigns, "Проверяет условия акций и промокодов", "REST")
Rel(notifications, messaging, "Отправляет уведомления", "HTTPS")

Rel(programs, events, "Публикует настройки программ")
Rel(accounts, events, "Публикует проведённые операции из outbox")
Rel(rules, events, "Публикует изменения уровней")
Rel(events, rules, "Настройки программ и операции для пересчёта уровня")
Rel(events, history, "Операции для построения истории")
Rel(events, notifications, "Изменения баланса и уровня")

Rel(rules, rules_cache, "Кэширует действующие правила")
Rel(campaigns, campaigns_cache, "Кэширует акции и промокоды")

Rel_D(customers, customers_db, "Ведёт профили клиентов")
Rel_D(programs, programs_db, "Ведёт программы и участие компаний")
Rel_D(accounts, accounts_db, "Фиксирует баланс и outbox одной транзакцией")
Rel_D(rules, rules_db, "Ведёт правила и уровни")
Rel_D(campaigns, campaigns_db, "Ведёт акции и промокоды")
Rel_D(notifications, notifications_db, "Сохраняет состояние доставки")
Rel_D(history, history_db, "Хранит и читает историю операций")

SHOW_LEGEND()
@enduml
```

```bash
# Из корня репозитория; нужна команда plantuml со стандартной библиотекой C4.
format=svg # или png
image="docs/img/c4-container-$(date +%Y%m%d-%H%M%S).$format"
sed -n '/^@startuml C4Container$/,/^@enduml$/p' docs/decomposition.md > /tmp/microservices-c4-container.puml
plantuml -t"$format" -pipe < /tmp/microservices-c4-container.puml > "$image"
```

</details>

#### 3.6.3. Компонентные диаграммы

##### [Customer Management](#21-customer-management)

![Компонентная диаграмма C4 — Customer Management](img/c4-component-customer-management-20261115-160425.svg)

<details>
<summary>Исходник PlantUML и генерация PNG/SVG</summary>

```plantuml
@startuml C4ComponentCustomers
!pragma layout elk
!include <C4/C4_Component>
LAYOUT_TOP_DOWN()
title Компонентная диаграмма — Customer Management

Container(gateway, "API Gateway", "HTTPS", "Маршрутизирует запросы к профилям")
Container(accounts, "Bonus Accounts", "REST-сервис", "Проверяет клиента при открытии счёта")
Container(notifications, "Notifications", "Сервис", "Запрашивает контакты для доставки")
ContainerDb(db, "Customer DB", "PostgreSQL", "Профили, контакты и привязки к торговым системам")

Container_Boundary(service, "Customer Management") {
  Component(api, "Customer API", "REST", "Регистрация, чтение и обновление профилей")
  Component(registration, "Customer Registration", "Компонент", "Сверяет компанию и связывает внешние ID с клиентом")
  Component(profiles, "Customer Profiles", "Компонент", "Возвращает и обновляет профиль и контакты")
  Component(repository, "Customer Repository", "Компонент", "Работает с профилями и внешними привязками")
}

Rel(gateway, api, "Работает с профилями", "REST")
Rel(accounts, api, "Запрашивает данные клиента", "REST")
Rel(notifications, api, "Запрашивает контакты", "REST")
Rel(api, registration, "Передаёт регистрацию")
Rel(api, profiles, "Передаёт запросы профиля")
Rel(registration, repository, "Ищет привязку и сохраняет клиента")
Rel(profiles, repository, "Читает и обновляет профиль")
Rel(repository, db, "Читает и сохраняет данные", "SQL")

SHOW_LEGEND()
@enduml
```

```bash
# Из корня репозитория; нужна команда plantuml со стандартной библиотекой C4.
format=svg # или png
image="docs/img/c4-component-customer-management-$(date +%Y%m%d-%H%M%S).$format"
sed -n '/^@startuml C4ComponentCustomers$/,/^@enduml$/p' docs/decomposition.md > /tmp/microservices-c4-component.puml
plantuml -t"$format" -pipe < /tmp/microservices-c4-component.puml > "$image"
```

</details>

##### [Program and Partner Management](#22-program-and-partner-management)

![Компонентная диаграмма C4 — Program and Partner Management](img/c4-component-program-and-partner-management-20261115-162502.svg)

<details>
<summary>Исходник PlantUML и генерация PNG/SVG</summary>

```plantuml
@startuml C4ComponentPrograms
!pragma layout elk
!include <C4/C4_Component>
LAYOUT_TOP_DOWN()
title Компонентная диаграмма — Program and Partner Management

Container(gateway, "API Gateway", "HTTPS", "Маршрутизирует административные запросы")
Container(accounts, "Bonus Accounts", "REST-сервис", "Проверяет права на бонусные операции")
Container(rules, "Loyalty Rules and Levels", "REST-сервис", "Проверяет права на изменение правил")
Container(campaigns, "Campaigns and Promo Codes", "REST-сервис", "Проверяет права на управление акциями")
ContainerDb(db, "Program DB", "PostgreSQL", "Программы, компании, заявки и права участия")
ContainerQueue(events, "Platform Events", "Kafka", "Топик настроек программ")

Container_Boundary(service, "Program and Partner Management") {
  Component(api, "Programs API", "REST", "Принимает запросы управления и проверки прав")
  Component(programs, "Loyalty Programs", "Компонент", "Создаёт программы и задаёт настройки")
  Component(partners, "Partner Participation", "Компонент", "Регистрирует компании, оформляет участие и меняет права")
  Component(permissions, "Permission Check", "Компонент", "Сверяет действие с правами участника")
  Component(publisher, "Program Settings Publisher", "Kafka Producer", "Публикует настройки созданной программы")
}

Rel(gateway, api, "Управляет программами и участием", "REST")
Rel(accounts, api, "Проверяет права на операции", "REST")
Rel(rules, api, "Проверяет права на правила", "REST")
Rel(campaigns, api, "Проверяет права на акции", "REST")
Rel(api, programs, "Передаёт создание программы")
Rel(api, partners, "Передаёт запросы по компаниям и участию")
Rel(api, permissions, "Передаёт проверку действия")
Rel(programs, partners, "Оформляет участие владельца")
Rel(programs, db, "Сохраняет программу", "SQL")
Rel(partners, db, "Сохраняет компании, заявки и права", "SQL")
Rel(permissions, db, "Читает права участия", "SQL")
Rel(programs, publisher, "Передаёт настройки созданной программы")
Rel(publisher, events, "Публикует настройки", "Kafka")

SHOW_LEGEND()
@enduml
```

```bash
# Из корня репозитория; нужна команда plantuml со стандартной библиотекой C4.
format=svg # или png
image="docs/img/c4-component-program-and-partner-management-$(date +%Y%m%d-%H%M%S).$format"
sed -n '/^@startuml C4ComponentPrograms$/,/^@enduml$/p' docs/decomposition.md > /tmp/microservices-c4-component.puml
plantuml -t"$format" -pipe < /tmp/microservices-c4-component.puml > "$image"
```

</details>

##### [Bonus Accounts](#23-bonus-accounts)

![Компонентная диаграмма C4 — Bonus Accounts](img/c4-component-bonus-accounts-20261115-160425.svg)

<details>
<summary>Исходник PlantUML и генерация PNG/SVG</summary>

```plantuml
@startuml C4ComponentAccounts
!pragma layout elk
!include <C4/C4_Component>
LAYOUT_TOP_DOWN()
title Компонентная диаграмма — Bonus Accounts

Container(gateway, "API Gateway", "HTTPS", "Маршрутизирует запросы к счетам")
Container(customers, "Customer Management", "REST-сервис", "Хранит идентичность клиента")
Container(programs, "Program and Partner Management", "REST-сервис", "Хранит права компаний в программе")
Container(rules, "Loyalty Rules and Levels", "REST-сервис", "Рассчитывает сумму начисления")
Container(campaigns, "Campaigns and Promo Codes", "REST-сервис", "Проверяет применимость промокода")
ContainerDb(db, "Account DB", "PostgreSQL", "Счета, операции и записи outbox на шардах")
ContainerQueue(events, "Platform Events", "Kafka", "Топик проведённых операций по счетам")

Container_Boundary(service, "Bonus Accounts") {
  Component(api, "Accounts API", "REST", "Открытие и чтение счёта, начисления и списания")
  Component(accounts, "Account Management", "Компонент", "Открывает один счёт на клиента в программе и читает баланс")
  Component(operations, "Bonus Operations", "Компонент", "Проверяет права и получает условия начисления")
  Component(ledger, "Account Ledger", "Компонент", "Проверяет остаток и идемпотентность, фиксирует операции")
  Component(publisher, "Outbox Publisher", "Kafka Producer", "Публикует подтверждённые операции после фиксации в БД")
}

Rel(gateway, api, "Запрашивает операции и баланс", "REST")
Rel(api, accounts, "Передаёт открытие и чтение счёта")
Rel(api, operations, "Передаёт начисление или списание")
Rel(accounts, customers, "Проверяет идентичность клиента", "REST")
Rel(accounts, db, "Открывает счёт и читает баланс", "SQL")
Rel(operations, programs, "Проверяет права на операцию", "REST")
Rel(operations, campaigns, "Проверяет промокод при наличии", "REST")
Rel(operations, rules, "Запрашивает расчёт начисления", "REST")
Rel(operations, ledger, "Передаёт операцию к проведению")
Rel(ledger, db, "Атомарно сохраняет баланс, операцию и запись outbox", "SQL")
Rel(publisher, db, "Читает записи outbox", "SQL")
Rel(publisher, events, "Публикует проведённые операции", "Kafka")

SHOW_LEGEND()
@enduml
```

```bash
# Из корня репозитория; нужна команда plantuml со стандартной библиотекой C4.
format=svg # или png
image="docs/img/c4-component-bonus-accounts-$(date +%Y%m%d-%H%M%S).$format"
sed -n '/^@startuml C4ComponentAccounts$/,/^@enduml$/p' docs/decomposition.md > /tmp/microservices-c4-component.puml
plantuml -t"$format" -pipe < /tmp/microservices-c4-component.puml > "$image"
```

</details>

##### [Loyalty Rules and Levels](#24-loyalty-rules-and-levels)

![Компонентная диаграмма C4 — Loyalty Rules and Levels](img/c4-component-loyalty-rules-and-levels-20261115-160425.svg)

<details>
<summary>Исходник PlantUML и генерация PNG/SVG</summary>

```plantuml
@startuml C4ComponentRules
!pragma layout elk
!include <C4/C4_Component>
LAYOUT_TOP_DOWN()
title Компонентная диаграмма — Loyalty Rules and Levels

Container(gateway, "API Gateway", "HTTPS", "Маршрутизирует запросы к правилам и уровням")
Container(accounts, "Bonus Accounts", "REST-сервис", "Запрашивает расчёт начисления")
Container(programs, "Program and Partner Management", "REST-сервис", "Проверяет права на изменение правил")
ContainerQueue(events, "Platform Events", "Kafka", "Настройки программ, операции по счетам и изменения уровней")
ContainerDb(cache, "Rules Cache", "Redis", "Действующие правила по программе")
ContainerDb(db, "Rules and Levels DB", "PostgreSQL", "Правила, уровни и прогресс клиентов")

Container_Boundary(service, "Loyalty Rules and Levels") {
  Component(api, "Rules and Levels API", "REST", "Изменение правил, расчёт начисления и чтение уровня")
  Component(catalog, "Bonus Rules", "Компонент", "Ведёт правила и обновляет их кэш")
  Component(calculator, "Bonus Calculator", "Компонент", "Учитывает правила, уровень клиента и условия акции")
  Component(levels, "Loyalty Levels", "Компонент", "Читает уровень и пересчитывает прогресс по операциям")
  Component(consumer, "Loyalty Event Consumer", "Kafka Consumer", "Принимает настройки программ и операции по счетам")
  Component(publisher, "Level Event Publisher", "Kafka Producer", "Публикует изменения уровня")
}

Rel(gateway, api, "Управляет правилами и запрашивает уровень", "REST")
Rel(accounts, api, "Запрашивает расчёт начисления", "REST")
Rel(api, catalog, "Передаёт изменение правила")
Rel(api, calculator, "Передаёт параметры расчёта")
Rel(api, levels, "Запрашивает уровень и прогресс")
Rel(catalog, programs, "Проверяет право изменить правило", "REST")
Rel(catalog, cache, "Кэширует правила с TTL и сбрасывает устаревшие записи")
Rel(catalog, db, "Сохраняет правила и читает их при промахе", "SQL")
Rel(calculator, catalog, "Получает действующие правила")
Rel(calculator, levels, "Получает уровень клиента")
Rel(levels, db, "Читает и сохраняет уровни и прогресс", "SQL")
Rel(events, consumer, "Передаёт настройки и проведённые операции", "Kafka")
Rel(consumer, catalog, "Передаёт новые настройки программы")
Rel(consumer, levels, "Передаёт операции для пересчёта")
Rel(levels, publisher, "Передаёт изменение уровня")
Rel(publisher, events, "Публикует изменение уровня", "Kafka")

SHOW_LEGEND()
@enduml
```

```bash
# Из корня репозитория; нужна команда plantuml со стандартной библиотекой C4.
format=svg # или png
image="docs/img/c4-component-loyalty-rules-and-levels-$(date +%Y%m%d-%H%M%S).$format"
sed -n '/^@startuml C4ComponentRules$/,/^@enduml$/p' docs/decomposition.md > /tmp/microservices-c4-component.puml
plantuml -t"$format" -pipe < /tmp/microservices-c4-component.puml > "$image"
```

</details>

##### [Campaigns and Promo Codes](#25-campaigns-and-promo-codes)

![Компонентная диаграмма C4 — Campaigns and Promo Codes](img/c4-component-campaigns-and-promo-codes-20261115-160425.svg)

<details>
<summary>Исходник PlantUML и генерация PNG/SVG</summary>

```plantuml
@startuml C4ComponentCampaigns
!pragma layout elk
!include <C4/C4_Component>
LAYOUT_TOP_DOWN()
title Компонентная диаграмма — Campaigns and Promo Codes

Container(gateway, "API Gateway", "HTTPS", "Маршрутизирует запросы к акциям и промокодам")
Container(accounts, "Bonus Accounts", "REST-сервис", "Запрашивает применимость промокода при начислении")
Container(programs, "Program and Partner Management", "REST-сервис", "Хранит права компаний в программе")
ContainerDb(cache, "Campaigns Cache", "Redis", "Определения акций и промокодов")
ContainerDb(db, "Campaign DB", "PostgreSQL", "Акции, промокоды и условия применения")

Container_Boundary(service, "Campaigns and Promo Codes") {
  Component(api, "Campaigns API", "REST", "Создание акций и промокодов, проверка кода")
  Component(management, "Campaign Management", "Компонент", "Создаёт акции и промокоды для программы")
  Component(validation, "Promo Code Validation", "Компонент", "Проверяет срок, компанию и условия покупки")
  Component(repository, "Campaign Repository", "Компонент", "Хранит определения и управляет их кэшем")
}

Rel(gateway, api, "Управляет акциями и промокодами", "REST")
Rel(accounts, api, "Проверяет код и получает условия акции", "REST")
Rel(api, management, "Передаёт запрос на создание")
Rel(api, validation, "Передаёт код и параметры покупки")
Rel(management, programs, "Проверяет права на управление акциями", "REST")
Rel(management, repository, "Сохраняет акции и промокоды")
Rel(validation, repository, "Получает определения акции и кода")
Rel(repository, cache, "Кэширует определения с TTL и сбрасывает изменённые записи")
Rel(repository, db, "Сохраняет данные и читает при промахе", "SQL")

SHOW_LEGEND()
@enduml
```

```bash
# Из корня репозитория; нужна команда plantuml со стандартной библиотекой C4.
format=svg # или png
image="docs/img/c4-component-campaigns-and-promo-codes-$(date +%Y%m%d-%H%M%S).$format"
sed -n '/^@startuml C4ComponentCampaigns$/,/^@enduml$/p' docs/decomposition.md > /tmp/microservices-c4-component.puml
plantuml -t"$format" -pipe < /tmp/microservices-c4-component.puml > "$image"
```

</details>

##### [Notifications](#26-notifications)

![Компонентная диаграмма C4 — Notifications](img/c4-component-notifications-20261115-160425.svg)

<details>
<summary>Исходник PlantUML и генерация PNG/SVG</summary>

```plantuml
@startuml C4ComponentNotifications
!pragma layout elk
!include <C4/C4_Component>
LAYOUT_TOP_DOWN()
title Компонентная диаграмма — Notifications

Container(gateway, "API Gateway", "HTTPS", "Маршрутизирует запросы о статусе доставки")
Container(customers, "Customer Management", "REST-сервис", "Хранит контакты клиентов")
ContainerQueue(events, "Platform Events", "Kafka", "Изменения баланса и уровня")
System_Ext(providers, "Провайдеры уведомлений", "Доставляют сообщения клиентам")
ContainerDb(db, "Notification DB", "PostgreSQL", "Уведомления, статусы и время следующей попытки")

Container_Boundary(service, "Notifications") {
  Component(api, "Notification Status API", "REST", "Возвращает канал и статус доставки")
  Component(consumer, "Notification Event Consumer", "Kafka Consumer", "Принимает события баланса и уровня")
  Component(preparation, "Notification Preparation", "Компонент", "Определяет получателя и готовит уведомление")
  Component(delivery, "Delivery Worker", "Компонент", "Отправляет уведомления и планирует повторы")
}

Rel(gateway, api, "Запрашивает статус", "REST")
Rel(api, db, "Читает канал и статус", "SQL")
Rel(events, consumer, "Передаёт изменения баланса и уровня", "Kafka")
Rel(consumer, preparation, "Передаёт событие для уведомления")
Rel(preparation, customers, "Получает контакты клиента", "REST")
Rel(preparation, db, "Сохраняет уведомление для доставки", "SQL")
Rel(delivery, db, "Выбирает задачи и сохраняет результат отправки", "SQL")
Rel(delivery, providers, "Отправляет уведомление", "HTTPS")

SHOW_LEGEND()
@enduml
```

```bash
# Из корня репозитория; нужна команда plantuml со стандартной библиотекой C4.
format=svg # или png
image="docs/img/c4-component-notifications-$(date +%Y%m%d-%H%M%S).$format"
sed -n '/^@startuml C4ComponentNotifications$/,/^@enduml$/p' docs/decomposition.md > /tmp/microservices-c4-component.puml
plantuml -t"$format" -pipe < /tmp/microservices-c4-component.puml > "$image"
```

</details>

##### [Operation History](#27-operation-history)

![Компонентная диаграмма C4 — Operation History](img/c4-component-operation-history-20261115-162502.svg)

<details>
<summary>Исходник PlantUML и генерация PNG/SVG</summary>

```plantuml
@startuml C4ComponentHistory
!pragma layout elk
!include <C4/C4_Component>
LAYOUT_TOP_DOWN()
title Компонентная диаграмма — Operation History

Container(gateway, "API Gateway", "HTTPS", "Маршрутизирует запросы истории")
ContainerQueue(events, "Platform Events", "Kafka", "Проведённые операции Bonus Accounts")
ContainerDb(db, "History DB", "PostgreSQL", "Проекция операций, индекс по счёту и курсору")

Container_Boundary(service, "Operation History") {
  Component(api, "History API", "REST", "Возвращает страницу операций по счёту")
  Component(query, "History Query", "Компонент", "Проверяет владельца счёта и читает историю по курсору")
  Component(projector, "History Projector", "Kafka Consumer", "Пополняет историю без дублей при повторной доставке")
}

Rel(gateway, api, "Передаёт запрос и проверенную идентичность клиента", "REST")
Rel(api, query, "Передаёт ID клиента, счёт и курсор")
Rel(query, db, "Читает операции только по счёту этого клиента", "SQL")
Rel(events, projector, "Передаёт проведённые операции", "Kafka")
Rel(projector, db, "Записывает проекцию операции", "SQL")

SHOW_LEGEND()
@enduml
```

```bash
# Из корня репозитория; нужна команда plantuml со стандартной библиотекой C4.
format=svg # или png
image="docs/img/c4-component-operation-history-$(date +%Y%m%d-%H%M%S).$format"
sed -n '/^@startuml C4ComponentHistory$/,/^@enduml$/p' docs/decomposition.md > /tmp/microservices-c4-component.puml
plantuml -t"$format" -pipe < /tmp/microservices-c4-component.puml > "$image"
```

</details>

##### [API Gateway](#28-api-gateway)

![Компонентная диаграмма C4 — API Gateway](img/c4-component-api-gateway-20261115-164932.svg)

<details>
<summary>Исходник PlantUML и генерация PNG/SVG</summary>

```plantuml
@startuml C4ComponentGateway
!pragma layout elk
!include <C4/C4_Component>
LAYOUT_TOP_DOWN()
title Компонентная диаграмма — API Gateway

Person(customer, "Клиент", "Пользуется личным кабинетом")
Person(manager, "Представитель компании", "Управляет программами и акциями")
System_Ext(trading, "Торговые системы", "Получают токены интеграций и вызывают API")
Container(keycloak, "Keycloak", "OAuth 2.0 / OIDC", "Организует вход и выдаёт токены доступа")
Container(domain, "Domain сервис", "REST-сервис", "Обрабатывает прикладной запрос")

Container_Boundary(service, "API Gateway") {
  Component(ingress, "HTTPS Entry Point", "HTTPS", "Разделяет маршруты входа и прикладного API")
  Component(sso, "SSO Proxy", "Компонент", "Проксирует вход, возврат и запросы на выдачу токенов")
  Component(validator, "Token Validator", "JWT", "Проверяет подпись, издателя, назначение и срок токена")
  Component(keys, "JWKS Cache", "Память реплики", "Хранит публичные ключи и обновляет их из Keycloak")
  Component(router, "API Router", "Компонент", "Передаёт запрос и проверенную идентичность нужному сервису")
}

Rel(customer, ingress, "Входит и пользуется кабинетом", "HTTPS")
Rel(manager, ingress, "Входит и управляет программами", "HTTPS")
Rel(trading, ingress, "Получает токен интеграции и вызывает API", "HTTPS")
Rel(ingress, sso, "Передаёт маршруты входа без токена API")
Rel(sso, keycloak, "Проксирует запросы входа и выдачи токенов", "HTTPS")
Rel(ingress, validator, "Передаёт прикладной запрос на проверку")
Rel(validator, keys, "Получает ключ проверки подписи")
Rel(keys, keycloak, "Обновляет публичные ключи", "HTTPS / JWKS")
Rel(validator, router, "Передаёт запрос после проверки")
Rel(router, domain, "Передаёт прикладной запрос", "REST")

SHOW_LEGEND()
@enduml
```

```bash
# Из корня репозитория; нужна команда plantuml со стандартной библиотекой C4.
format=svg # или png
image="docs/img/c4-component-api-gateway-$(date +%Y%m%d-%H%M%S).$format"
sed -n '/^@startuml C4ComponentGateway$/,/^@enduml$/p' docs/decomposition.md > /tmp/microservices-c4-component.puml
plantuml -t"$format" -pipe < /tmp/microservices-c4-component.puml > "$image"
```

</details>
