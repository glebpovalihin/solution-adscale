```plantuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml
LAYOUT_WITH_LEGEND()
LAYOUT_TOP_DOWN()
title C4 Container Diagram — AdScale (TO-BE, через год)
AddElementTag("gw", $fontColor="black", $bgColor="lightgreen")

Person(advertiser, "Рекламодатель", "Управляет кампаниями, бюджетами, ставками")
Person_Ext(endUser, "Пользователь сайта/приложения", "Просматривает рекламу, кликает по баннерам")
System_Ext(dspPartner, "DSP-партнёр", "Bid requests / responses по OpenRTB")
System_Ext(paymentGateway, "Платёжный шлюз", "Пополнение рекламного бюджета")

Container(dspGateway, "DSP Gateway", "Envoy", "Маршрутизация, rate limiting, auth, SSL termination", $tags="gw")
Container(apiGateway, "API Gateway", "Envoy", "Маршрутизация, rate limiting, auth, SSL termination", $tags="gw")
Container(advGateway, "Advertiser Gateway", "Envoy", "Маршрутизация, rate limiting, auth, SSL termination", $tags="gw")


System_Boundary(analytics_sys, "Analytics System") {
  Container(analyticsService, "Analytics Service", "Python", "Формирование отчётов и аналитики для рекламодателей")
  ContainerDb(clickHouse, "Analytics ClickHouse", "ClickHouse", "Хранилище событий: клики, показы (time-series)")
}

System_Boundary(ad_sys, "AD System") {
  Container(adServer, "Ad Server", "Go", "Приём bid request, идентификация пользователя, подбор кандидатов")
  ContainerDb(ad_postgres, "PostgreSQL (Primary)", "PostgreSQL", "Кампании, таргетинг")
  ContainerDb(ad_redis_read, "Redis", "Redis", "Кеширование кампаний и таргетинга")

  Rel(adServer, ad_postgres, "Создание кампаний, таргетинга")
  Rel(adServer, ad_redis_read, "Чтение кампаний и таргетинга")
}

System_Boundary(auction_sys, "Auction System") {
  Container(biddingService, "Auction Service", "C++ / Go", "Взвешивание ставок, бизнес-правила, определение победителя аукциона")
  ContainerDb(bid_postgres, "PostgreSQL (Primary)", "PostgreSQL", "Ставки, правила, конфигурации")
  ContainerDb(bid_postgres_read, "PostgreSQL (Read Replica)", "PostgreSQL", "Ставки, правила, конфигурации")
  ContainerDb(bid_redis_read, "Redis", "Redis", "Ставки, правила, конфигурации")

  Rel(biddingService, bid_postgres, "Запись ставок, конфигураций, правил")
  Rel(biddingService, bid_postgres_read, "Чтение ставок (cache miss), конфигураций, правил")
  Rel(biddingService, bid_redis_read, "Запись/Чтение ставок кандидатов, конфигураций и правил")
}

System_Boundary(delivery_sys, "Delivery System") {
  Container(deliveryService, "Delivery Service", "Node.js", "Формирование HTML/JS-разметки баннера и HTTP response")
  ContainerDb(del_postgres, "PostgreSQL (Primary)", "PostgreSQL", "Кампании, креативы")
  ContainerDb(del_redis_read, "Redis", "Redis", "Кампании, креативы")

  Rel(deliveryService, del_postgres, "Запись кампаний, креативов")
  Rel(deliveryService, del_redis_read, "Запись/Чтение кампаний и креативов")
}


System_Boundary(campaign_sys, "Campaign System") {
  Container(campaignService, "Campaign Service", "Go", "CRUD рекламных кампаний, управление ставками и таргетингом")
  ContainerDb(camp_postgres, "PostgreSQL (Primary)", "PostgreSQL", "Рекламные кампании, ставки и таргетинг")
  ContainerDb(camp_postgres_read, "PostgreSQL (Read Replica)", "PostgreSQL", "Рекламные кампании, ставки и таргетинг")

  Rel(campaignService, camp_postgres, "Создание/изменение рекламных кампаний, ставок и таргетинга")
  Rel(campaignService, camp_postgres_read, "Чтение рекламных кампаний, ставок и таргетинга")
}

System_Boundary(financial_sys, "Financial System") {
  Container(budgetService, "Financial Service", "Go", "Списание средств, выставление счетов, управление балансами")
  ContainerDb(budgetDB, "Financial DB", "PostgreSQL", "Счета, транзакции, балансы, аудит")

  Rel(budgetService, budgetDB, "Транзакции")
}

System_Boundary(statistic_sys, "Statistic System") {
  Container(statsService, "Statistic Service", "Python", "Запись кликов и показов")
  ContainerDb(statisticsDB, "Statistics DB", "ClickHouse", "Сырые события: клики, показы, time-series")

  Rel(statsService, statisticsDB, "Bulk-запись событий")
}

System_Boundary(adv_sys, "Advertiser System") {
  Container(adCabinet, "Advertiser Dashboard", "Node.js + React", "Web UI для управления кампаниями, отчётности")
}

    ContainerQueue(show_topic, "Топик событий отображения", "Kafka")
    ContainerQueue(click_topic, "Топик событий клика", "Kafka")

    ContainerQueue(bid_topic, "Топик ставок", "Kafka")
    ContainerQueue(targ_topic, "Топик таргетинга", "Kafka")
    ContainerQueue(camp_topic, "Топик кампании", "Kafka")

Rel(dspPartner, dspGateway, "Bid requests / responses", "HTTPS / OpenRTB")
Rel(endUser, apiGateway, "Просмотр рекламы, клики", "HTTPS")
Rel(advertiser, adCabinet, "Управляет кампаниями и бюджетами", "HTTPS")
Rel(adCabinet, paymentGateway, "Пополнение бюджета", "HTTPS / REST")

Rel(dspGateway, adServer, "Маршрутизация bid requests", "HTTPS")
Rel(advGateway, campaignService, "Маршрутизация CRUD кампаний", "HTTPS")
Rel(apiGateway, deliveryService, "Просмотр рекламы", "HTTPS")
Rel(apiGateway, statsService, "Регистрирует показы/клики", "HTTPS")

Rel(advGateway, budgetService, "Маршрутизация управления финансами", "HTTPS")
Rel(advGateway, analyticsService, "Маршрутизация запросов отчётов", "HTTPS")
Rel(adCabinet, advGateway, "Маршрутизация UI-запросов", "HTTPS")


Rel(adServer, biddingService, "Передает кандидатов", "Grpc")

Rel(statsService, show_topic, "Событие отображения", "Event")
Rel(statsService, click_topic, "Событие клика", "Event")

Rel(campaignService, bid_topic, "Событие изменения ставки", "Event")
Rel(bid_topic, biddingService, "Событие изменения ставки", "Event")


Rel(campaignService, targ_topic, "Событие изменения таргетинга", "Event")
Rel(targ_topic, adServer, "Событие изменения таргетинга", "Event")

Rel(campaignService, camp_topic, "Событие изменения кампании", "Event")
Rel(camp_topic, adServer, "Событие изменения кампании", "Event")
Rel(camp_topic, deliveryService, "Событие изменения кампании", "Event")

Rel(show_topic, analyticsService, "Событие отображения", "Event")
Rel(click_topic, analyticsService, "Событие клика", "Event")

Rel(show_topic, budgetService, "Событие отображения", "Event")
Rel(click_topic, budgetService, "Событие клика", "Event")

Rel(analyticsService, clickHouse, "Аналитика по событиям")

note left of biddingService
  **Bounded Context: Аукцион**
  - Наиболее latency-чувствительный (critical path)
  - Свой Redis-кэш для ставок
  - Своя PostgreSQL (изоляция от блокировок)
  - Синхронно предоставляет результаты аукциона
  - Горизонтальное масштабирование
end note

note left of adServer
  **Bounded Context: Таргетинг**
  - Кэширует данные в Redis
  - Получает обновления кампаний и таргетинга через Kafka (async)
  - Вызов AuctionService — синхронный (gRPC)
  - Масштабируется независимо
end note

note bottom of campaignService
  **Bounded Context: Кампании**
  - Своя БД (изоляция от аукциона)
  - Чтение с БД реплики для  снижения нагрузки на мастер PostgreSQL
  - Публикует изменения в Kafka
  - Не является частью крит пути
end note

note bottom of budgetService
  **Bounded Context: Финансы**
  - Своя БД (ACID-транзакции)
  - Получает события списания (просмотр/клик) из Kafka (async)
end note

note right of statsService
  **Bounded Context: События**
  - Write-heavy, полностью асинхронный
  - Bulk-запись в ClickHouse с помощью фоновых воркеров
  - Минимальное влияние на latency аукциона
  - Взаимодействует с end user через API Gateway
end note

note bottom of analyticsService
  **Bounded Context: Аналитика**
  - Read-heavy, асинхронный
  - Потребляет агрегаты из Kafka
  - Свой ClickHouse (не нагружает OLTP-БД)
end note

note right of deliveryService
  **Bounded Context: Доставка**
  - Минимальная логика (формирование разметки)
  - Взаимодействует с end user через API Gateway
end note
```
