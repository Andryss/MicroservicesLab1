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

Внутренние контексты: Customer Management, Program and Partner Management, Bonus Accounts, Loyalty Rules and Levels, Campaigns and Promo Codes, Notifications. Operation History хранит историю операций.

Каждый блок обозначает область ответственности внутри системы. Все связи показаны одинаково: стрелка направлена от источника информации к её получателю, подпись описывает передаваемые данные.

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
