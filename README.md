# Debezium mysql connector example

## 1. install docker

```bash
apt-get update && apt-get install docker.io
```

## 2. create user and group

```bash
groupadd -g 2000 k8sadm
useradd -m -u 2000 -g 2000 -s /bin/bash k8sadm
echo ' k8sadm ALL=(ALL)   ALL' >> /etc/sudoers
```


### 3. Run Zookeeper / Kafka / MySQL / MySQL Client
```bash
docker run -d -it --rm --name zookeeper -p 2181:2181 -p 2888:2888 -p 3888:3888 quay.io/debezium/zookeeper:2.1

docker run -d -it --rm --name kafka -p 9092:9092 --link zookeeper:zookeeper quay.io/debezium/kafka:2.1

docker run -d -it --rm --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw quay.io/debezium/example-mysql:2.1

docker run -it --rm --name mysqlterm --link mysql --rm mysql:8.0 sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'
```

### 4. Test DB
```bash
mysql> use inventory;
mysql> show tables;
mysql> SELECT * FROM customers;
```

### 5. Run Kakfa Connect 
```bash
docker run -d -it --rm --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link kafka:kafka --link mysql:mysql quay.io/debezium/connect:2.1

curl -H "Accept:application/json" localhost:8083/

curl -H "Accept:application/json" localhost:8083/connectors/

curl -X GET -H "Accept:application/json" localhost:8083/connectors/inventory-connector | jq '.'
```

### 6. Register inventory connector
```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "topic.prefix": "dbserver1", "database.include.list": "inventory", "schema.history.internal.kafka.bootstrap.servers": "kafka:9092", "schema.history.internal.kafka.topic": "schemahistory.inventory" } }'
```


### 6. Update & Delete Data
```bash
UPDATE customers SET first_name='Anne Marie' WHERE id=1004;
DELETE FROM addresses WHERE customer_id=1004;
DELETE FROM customers WHERE id=1004;

INSERT INTO customers VALUES (default, "Sarah", "Thompson", "kitt@acme.com");
INSERT INTO customers VALUES (default, "Kenneth", "Anderson", "kander@acme.com");
```


