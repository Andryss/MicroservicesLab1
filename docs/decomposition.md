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

## 1. Context Map

Внутренние контексты: Customer Management, Program and Partner Management, Bonus Accounts,
Loyalty Rules and Levels, Campaigns and Promo Codes, Notifications. Operation History хранит историю операций.

Каждый блок обозначает область ответственности внутри системы. Все связи показаны одинаково:
стрелка направлена от источника информации к её получателю, подпись описывает передаваемые данные.

![Context Map](img/context-map-20261023-120351.svg)

<details>
<summary>Исходник PlantUML и генерация PNG/SVG</summary>

```plantuml
@startuml ContextMap
!pragma layout smetana
left to right direction

rectangle "Customer Management" as customers
rectangle "Program and Partner Management" as programs
rectangle "Bonus Accounts" as accounts
rectangle "Loyalty Rules and Levels" as loyalty
rectangle "Campaigns and Promo Codes" as campaigns
rectangle "Notifications" as notifications
rectangle "Operation History" as history

customers --> accounts : Customer identity
customers --> notifications : Contact details
programs --> accounts : Program membership\nand partner permissions
programs --> history : History access scope
programs --> loyalty : Program configuration
campaigns --> accounts : Campaign and promo terms
loyalty --> accounts : Bonus policy\nand loyalty level
accounts --> loyalty : Account activity
accounts --> history : Account transactions
accounts --> notifications : Balance changes
loyalty --> notifications : Loyalty level changes
@enduml
```

```bash
# Из корня репозитория; нужна команда plantuml.
format=svg # или png
image="docs/img/context-map-$(date +%Y%m%d-%H%M%S).$format"
sed -n '/^@startuml/,/^@enduml/p' docs/decomposition.md > /tmp/microservices-context-map.puml
plantuml -t"$format" -pipe < /tmp/microservices-context-map.puml > "$image"
```

</details>

## 2. Каталог микросервисов

### 2.1. Customer Management

```text
Service: Customer Management

Responsibility:
единая идентичность клиента, профиль и контактные данные.

Owned Data:
Customer (профиль, контакты, внешние идентификаторы в торговых системах)

API:
POST /customers
GET /customers/{customerId}
PATCH /customers/{customerId}

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
Шардирование БД по клиентам.
```

### 2.2. Program and Partner Management

```text
Service: Program and Partner Management

Responsibility:
создание программ, регистрация компаний, одобрение участия и управление правами партнёров. Создание
программы включает участие её владельца.

Owned Data:
LoyaltyProgram (компания-владелец)
Merchant
ProgramMembership (статус участия компании и её права в программе)

API:
POST /merchants
POST /programs
POST /programs/{programId}/memberships
PATCH /programs/{programId}/memberships/{merchantId}
POST /permissions/check

Dependencies:
in:
- Нет
out:
- Bonus Accounts : участие компании в программе и права на операции
- Loyalty Rules and Levels : настройки программы и права на изменение правил
- Campaigns and Promo Codes : участие компании и права управления акциями
- Operation History : область доступа к истории операций

Communication:
REST - API программ и компаний, управление участием и проверка прав.
Kafka - передача настроек созданных программ в Loyalty Rules and Levels.

Scalability:
Горизонтальное масштабирование сервиса.
```

### 2.3. Bonus Accounts

```text
Service: Bonus Accounts

Responsibility:
открытие счёта клиента в программе, баланс, проведение начислений и списаний; защита
от повторных операций и отрицательного остатка.

Owned Data:
LoyaltyAccount (баланс клиента в конкретной программе)
Transaction (начисление или списание, компания и результат операции)

API:
POST /programs/{programId}/accounts
GET /accounts/{accountId}
POST /accounts/{accountId}/accruals
POST /accounts/{accountId}/redemptions

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
Шардирование БД по бонусным счетам.
```

### 2.4. Loyalty Rules and Levels

```text
Service: Loyalty Rules and Levels

Responsibility:
расчёт начисления по правилам программы, уровню клиента и условиям акции; определение и пересчёт
уровней.

Owned Data:
Rule
LoyaltyLevel
CustomerLevel (уровень и прогресс клиента в конкретной программе)

API:
POST /programs/{programId}/calculations
PUT /programs/{programId}/rules/{ruleId}
GET /programs/{programId}/customers/{customerId}/level

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
Кеширование правил программы.
```

### 2.5. Campaigns and Promo Codes

```text
Service: Campaigns and Promo Codes

Responsibility:
управление акциями и промокодами, проверка сроков и условий их применения в программе и компании.

Owned Data:
Campaign (программа, условия и компании-участники)
PromoCode (программа, условия, срок действия и компании-участники; код уникален в программе)

API:
POST /programs/{programId}/campaigns
POST /programs/{programId}/promocodes
POST /programs/{programId}/promocodes/validate

Dependencies:
in:
- Program and Partner Management : участие компании и права управления акциями
out:
- Bonus Accounts : условия применения акций и промокодов

Communication:
REST - API акций и промокодов, проверка условий их применения.

Scalability:
Горизонтальное масштабирование сервиса.
Кеширование акций и промокодов.
```

### 2.6. Notifications

```text
Service: Notifications

Responsibility:
формирование и доставка уведомлений об изменениях баланса и уровня, повторные попытки доставки.

Owned Data:
Notification (получатель, канал и статус доставки)

API:
GET /notifications/{notificationId}

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
```

### 2.7. Operation History

```text
Service: Operation History

Responsibility:
построение и чтение долговременной истории операций в отдельном от транзакционных данных хранилище.

Owned Data:
OperationHistory (счёт, программа, клиент, компания и проведённые операции)

API:
GET /accounts/{accountId}/history
Курсорная пагинация.

Dependencies:
in:
- Bonus Accounts : проведённые операции по счёту
- Program and Partner Management : область доступа к истории операций
out:
- Нет

Communication:
REST - API истории операций с проверкой прав доступа.
Kafka - получение проведённых операций для построения истории.

Scalability:
Горизонтальное масштабирование сервиса.
Шардирование БД по бонусным счетам.
```

## 3. ADR-001. Асинхронное ведение истории операций

### 3.1. Контекст

Ожидаемая нагрузка: 20 млн клиентов и 50 000 RPS

Также требуется независимое масштабирование частей системы, хранение истории операций отдельно от транзакционных данных.

Нужно выбрать способ передачи подтверждённых операций из `Bonus Accounts` в `Operation History`.

Доли чтений и записей, объём и срок хранения истории, допустимая задержка ее обновления не определены.

### 3.2. Выбранное решение

Передавать подтверждённые операции через Kafka. `Operation History` обрабатывает их асинхронно и сохраняет
историю в собственной БД.

Ответ о проведении бонусной операции не ожидает обновления истории.

Передача сведений должна восстанавливаться после сбоя, а повторная доставка - сохранять одну запись об операции.

Механизм согласованной фиксации операции и публикации в Kafka происходит при помощи паттерна Transactional Outbox.

### 3.3. Альтернативы

1. **Синхронная запись через REST в `Operation History`**
   Позволяет дождаться появления операции в истории перед ответом клиенту. Но добавляет ожидание другого
   сервиса при проведении операции (добавляет точку отказа) и связывает доступность начислений и списаний
   с доступностью истории.
2. **Периодический перенос подтверждённых операций из `Bonus Accounts`**
   Позволяет обойтись без брокера сообщений. История обновляется пакетами и создаёт дополнительную
   нагрузку на источник транзакционных данных. Частоту переноса и размер пакета нужно подбирать по профилю нагрузки.
3. **Асинхронная запись через Outbox с прямой доставкой в `Operation History`**
   `Bonus Accounts` сохраняет операцию и сообщение для отправки одной транзакцией. Фоновый отправитель
   доставляет сообщение через REST и очищает outbox после подтверждения сохранения в истории.
   При недоступности истории неотправленные сообщения накапливаются в БД аккаунтов, увеличивают размер,
   а обработка и повторы конкурируют с другими операциями за ресурсы БД.

### 3.4. Обоснование выбора

- БД Operation History выполняет требование задания об отдельном хранении истории.
- Для системы с 20 млн клиентов выделение истории позволяет отдельно масштабировать её хранение и чтение.
  Число клиентов определяет масштаб системы.
- При целевых 50 000 RPS разделение потоков проведения операций и обработки истории позволяет независимо
  подбирать ресурсы под каждый поток.
- Асинхронная передача убирает ожидание `Operation History` из обработки бонусной операции и позволяет
  продолжать её проведение при временной недоступности истории.
- Паттерн Transactional Outbox позволяет реализовать транзакционность между изменением баланса аккаунта и
  отправкой сообщения внешней системе (Kafka) и не потерять сообщения.

### 3.5. Последствия

- **Положительные:** отдельное хранение истории, независимое масштабирование её обработки и чтения,
  отсутствие зависимости проведения операции от `Operation History`.
- **Отрицательные:** нужны брокер сообщений, надёжная публикация подтверждённых операций, обработка повторов
  и контроль отставания. Длительный сбой истории приводит к накоплению необработанных сообщений.
- **Согласованность:** история может отставать от текущего баланса. Допустимую задержку необходимо уточнить.
- **Проверка производительности:** решение не гарантирует выдержку при 50 000 RPS. Для проверки нужны
  нагрузочные испытания.
