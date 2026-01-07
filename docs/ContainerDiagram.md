flowchart LR
    %% Actors
    analyst["👤 Data Analyst"]

    %% External Sources
    subgraph external["External Sources"]
        pg["Postgres / S3\n(Taxi & Weather Data)"]
    end

    %% Platform
    subgraph platform["ClickHouse Analytics Platform"]

        airflow["Apache Airflow\nBatch Orchestration"]

        kafka["Kafka + Debezium\nStreaming / CDC"]

        keeper["ClickHouse Keeper\n(Replication & Metadata)"]

        subgraph ch_cluster["ClickHouse Cluster"]
            ch1["Shard 1\nReplica 1"]
            ch2["Shard 1\nReplica 2"]
        end

        datalens["Yandex DataLens\nBI & Dashboards"]
    end

    %% Batch Flow
    pg -->|Batch Extract| airflow
    airflow -->|REPLACE PARTITION| ch1

    %% Streaming Flow
    pg -. CDC .-> kafka
    kafka -. Kafka Engine .-> ch1

    %% Internal
    ch1 <--> keeper
    ch2 <--> keeper

    %% Visualization
    ch1 -->|SQL| datalens
    analyst -->|Web UI| datalens