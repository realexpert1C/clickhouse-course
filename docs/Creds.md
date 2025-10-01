#### Облачный сервер для курса Clickhouse:

IP сервера 
```185.207.65.197```


```admin```
 на сервере пароль:
 ``` yT7k!Vpr#q2D```

Доступ SSH:
```ssh admin@185.207.65.197```

MariaDb - strong_root_pass = «```Pz7w!dG3#vRf1LqX```»,  в кликхаус  ```wh_qQ(H*P.;```

IP контейнера с Portainer:
```185.207.65.197:9001```  Пользователь ```admin``` пароль  ```1Ns'[Zgz)@n_BDT-```

```curl 'http://185.207.65.197:8123/replicas_status'```


Создание пользователя в airflow
```docker compose down```

```docker run --rm -ti -v ./dags:/opt/airflow/dags apache/airflow:3.0.0-python3.10 standalone```

Итог: Password for user '```admin```': ```BYssnMgAFz7xsCqv```

Cоздание контейнера с vscode, пароль возьму тот же что в airflow: ```BYssnMgAFz7xsCqv```

Путь к облачному VSCODE 
```185.207.65.197:8443```

Запуск контейнера
```docker run -d --name code-server --network infra-net -p 8443:8080 -v "$HOME/projects:/home/coder/project" -v "$HOME/.config:/home/coder/.config" -e "PASSWORD=BYssnMgAFz7xsCqv" codercom/code-server:latest```

```docker stop code-server && docker rm code-server``` - остановить и удалить контейнер

CLICKHOUSE IP= ```185.207.65.197:8123``` 
CLICKHOUSE_USER=  ```default```
CLICKHOUSE_PASSWORD= ```wh_qQ(H*P.;``` 


docker exec -it airflow-airflow-webserver-1 airflow db migrate
docker exec -it airflow-airflow-webserver-1 airflow users create \
  --username admin \
  --firstname Admin \
  --lastname User \
  --role Admin \
  --email 8505885@gmail.com \
  --password BYssnMgAFz7xsCqv


docker run --rm \
  --network airflow_default \
  -v $(pwd)/dags:/opt/airflow/dags \
  -v $(pwd)/logs:/opt/airflow/logs \
  -v $(pwd)/plugins:/opt/airflow/plugins \
  -e AIRFLOW__CORE__EXECUTOR=LocalExecutor \
  -e AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:airflow@postgres/airflow \
  apache/airflow:3.0.0-python3.10 \
  airflow users create \
  --username admin \
  --firstname Admin \
  --lastname User \
  --role Admin \
  --email 8505885@gmail.com \
  --password BYssnMgAFz7xsCqv



docker run -d \
  --name clickhouse \
  --network infra-net \
  -p 8123:8123 \
  -p 9000:9000 \
  -p 9009:9009 \
  -v clickhouse_data:/var/lib/clickhouse \
  -v clickhouse_logs:/var/log/clickhouse-server \
  -e CLICKHOUSE_DB=default \
  -e CLICKHOUSE_USER=default \
  -e CLICKHOUSE_PASSWORD=«wh_qQ(H*P.;» \
  clickhouse/clickhouse-server:latest


clickhouse-client --host 127.0.0.1 --port 9000 --user default --password wh_qQ(H*P.;


echo "SELECT count() FROM uk_price_paid" \
  | clickhouse-benchmark --host 127.0.0.1 --port 9000 -i 10

echo "SELECT * FROM system.numbers LIMIT 1000000" | \
docker exec -i clickhouse clickhouse-benchmark \
--port 9000 --user default --password «wh_qQ(H*P.;» -i 10

echo "SELECT * FROM default.uk_price_paid LIMIT 10000000 OFFSET 10000000" | clickhouse-benchmark -i 10

echo "SELECT * FROM default.cell_towers LIMIT 10000000 OFFSET 10000000" | clickhouse-benchmark -i 10

echo "SELECT * FROM default.uk_price_paid LIMIT 10000000 OFFSET 10000000" | clickhouse-benchmark --port 9000 --user default --password «wh_qQ(H*P.;» -i 10
