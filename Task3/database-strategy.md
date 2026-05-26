# Database per Service

|Сервис|	Тип базы данных|	Обоснование|
|---|---|---|
|Сервис ставок (RTB System)|PostgreSQL (Primary + Read Replica) + Redis| latency-critical система требует быстрого времени поиска данных, преобладает read-heavy логика|
|Сервис кампаний (Campaign System)|PostgreSQL (Primary + Read Replica)|Преобладает запись, редкие чтения|
|Сервис креативов (Delivery System)|PostgreSQL (Primary + Read Replica) + Redis|latency-critical система требует быстрого времени предоставления креативов, преобладает read-heavy логика|
|Сервис статистики (Statistic System)|ClickHouse|write-heavy логка, time-series данные|
|Сервис аналитики (Analytics System)|ClickHouse|write-heavy логка, аналитические запросы требующие быстрой аггрегации для построения отчетов|
|Сервис финансов (Financial System)|PostgreSQL|Требуется ACID-транзакционность для учета критичных финансовых операций, strong consistency|
