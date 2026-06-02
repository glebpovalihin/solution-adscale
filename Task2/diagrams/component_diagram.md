```plantuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml
LAYOUT_WITH_LEGEND()
LAYOUT_TOP_DOWN()
title C4 Component Diagram — RTB Service
System_Ext(dspPartner, "DSP-партнёр", "Bid requests / responses по OpenRTB")
Container_Ext(dspGateway, "DSP Gateway", "Envoy", "Маршрутизация, rate limiting, auth")
System_Boundary(rtb, "RTB Service (Go)") {
  Container(bidHandler, "Bid Handler", "Go", "Приём bid request, валидация, парсинг OpenRTB")
  Container(Kafka_Consumer, "Kafka Consumer", "Handles events", "Потребление событий из топиков")

  Container(userResolver, "AD Server", "Go", "Идентификация/резолвинг пользователя, подбирает кандидатов в рекламу")
  Container(auctionEngine, "Auction Engine", "Go", "Взвешивание ставок, применение правил, определение победителя")

  Container(RepositoryL, "Repository Layer", "Data access logic", "Чтение/запись")

  ContainerDb(redis, "Redis Cache", "Redis", "Кэш: кампании, ставки, таргетинг, правила")
  ContainerDb(pgPrimary, "PostgreSQL (Primary)", "PostgreSQL", "Кампании, ставки, правила, конфигурации")
  ContainerDb(pgReplica, "PostgreSQL (Read Replica)", "PostgreSQL", "Read-only копия для чтения при cache miss")

  Rel(userResolver, RepositoryL, "Чтение данных", "Internal")
  Rel(auctionEngine, RepositoryL, "Чтение данных", "Internal")
  Rel(Kafka_Consumer, RepositoryL, "Запись данных", "Internal")

  Rel(RepositoryL, pgReplica, "Чтение кампаний, таргетинга, правил и ставок (cache miss)", "SQL")
  Rel(RepositoryL, pgPrimary, "Запись результатов аукциона", "SQL")
  Rel(RepositoryL, pgPrimary, "Запись кампаний, таргетинга, ставок", "SQL")

  Rel(RepositoryL, redis, "Чтение/запись кампаний и ставок", "Redis Protocol")
  Rel(RepositoryL, redis, "Чтение/запись таргетинга и правил", "Redis Protocol")
  Rel(RepositoryL, redis, "Чтение/Запись конфигураций аукциона", "Redis Protocol")

  Rel(bidHandler, userResolver, "Передаёт запрос", "Internal")
  Rel(userResolver, auctionEngine, "Передаёт кандидатов с макс. ставками", "Internal")
}
System_Boundary(messaging, "Messaging") {
  ContainerQueue(bidTopic, "топик ставок", "Kafka", "События изменения ставок")
  ContainerQueue(targTopic, "топик таргетинга", "Kafka", "События изменения таргетинга")
  ContainerQueue(campTopic, "топик кампаний", "Kafka", "События изменения кампаний")
}
System_Ext(campaignService, "Campaign Service", "Go", "CRUD кампаний — публикует изменения в Kafka")

Rel(dspPartner, dspGateway, "Bid request / response", "HTTPS / OpenRTB")
Rel(dspGateway, bidHandler, "Bid request", "HTTPS / OpenRTB")
Rel(campaignService, bidTopic, "Публикует изменение ставки", "Event")
Rel(campaignService, targTopic, "Публикует изменение таргетинга", "Event")
Rel(campaignService, campTopic, "Публикует изменение кампании", "Event")
Rel(bidTopic, Kafka_Consumer, "Потребляет обновления ставок", "Event")
Rel(targTopic, Kafka_Consumer, "Потребляет обновления таргетинга", "Event")
Rel(campTopic, Kafka_Consumer, "Потребляет обновления кампаний", "Event")
SHOW_LEGEND()
```
