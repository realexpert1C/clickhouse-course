C4Container
    title Container Diagram — ClickHouse Analytics Platform

    Person(analyst, "Аналитик данных", "Изучает отчеты и графики")

    System_Boundary(c1, "Аналитическая Платформа") {
        Container(airflow, "Apache Airflow", "Python/Docker", "Оркестрация батчевой загрузки и логика Replace Partition")
        ContainerQueue(kafka, "Kafka Cluster", "Kafka Engine", "Буферизация потоковых данных (Streaming)")
        
        System_Boundary(ch_cluster, "ClickHouse Cluster") {
            ContainerDb(ch_node1, "CH Shard 1 Replica 1", "ClickHouse", "Хранение и обработка данных")
            ContainerDb(ch_node2, "CH Shard 1 Replica 2", "ClickHouse", "Хранение и обработка данных")
            ContainerDb(ch_node3, "CH Shard 2 Replica 1", "ClickHouse", "Хранение и обработка данных")
            ContainerDb(ch_node4, "CH Shard 2 Replica 2", "ClickHouse", "Хранение и обработка данных")
        }

        Container(keeper, "CH Keeper", "Raft/C++", "Координация репликации и шардирования")
        Container(datalens, "Yandex DataLens", "BI Tool", "Визуализация и построение дашбордов")
    }

    SystemExternal(source_db, "Postgres / S3", "Источник данных (Taxi & Weather)")

    Rel(source_db, airflow, "Запрос данных (Batch)", "JDBC/S3 API")
    Rel(source_db, kafka, "Передача событий (CDC)", "Debezium")
    
    Rel(airflow, ch_node1, "Вставка батчей", "Native Protocol")
    Rel(kafka, ch_node1, "Чтение потока", "Kafka Engine")
    
    Rel(ch_node1, keeper, "Состояние кворума", "Internal")
    Rel(datalens, ch_node1, "SQL запросы", "HTTP/Interface")
    Rel(analyst, datalens, "Просмотр графиков", "Web")
