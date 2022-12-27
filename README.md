# Debezium mysql connector example

- MySQL의 BinLog를 이용하여 DB의 변경사항을 Capture하여 Kafka Topic에 전달하는 예제
- Kafka Connect / Debezium MySQL connector와 watch-topic을 이용하여 변경사항 확인

## 1. install docker

```bash
apt-get update && apt-get install docker.io zsh jq -y
```

## 2. create user and group

```bash
groupadd -g 2000 k8sadm
useradd -m -u 2000 -g 2000 -s /bin/bash k8sadm
echo ' k8sadm ALL=(ALL)   ALL' >> /etc/sudoers
echo -e "1\n1" | passwd k8sadm >/dev/null 2>&1
usermod -aG docker k8sadm

wget https://github.com/rancher/rke/releases/download/v1.3.15/rke_linux-amd64
chmod 755 rke_linux-amd64 && sudo mv rke_linux-amd64 /usr/local/bin/rke
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod 755 kubectl && sudo mv kubectl /usr/local/bin
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

git clone https://github.com/zsh-users/zsh-autosuggestions ~/.zsh/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ~/.zsh/zsh-syntax-highlighting
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ~/.zsh/powerlevel10k

echo 'source ~/.zsh/powerlevel10k/powerlevel10k.zsh-theme' >>~/.zshrc
echo 'source ~/.zsh/zsh-autosuggestions/zsh-autosuggestions.zsh' >>~/.zshrc
echo 'source ~/.zsh/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh' >>~/.zshrc

$ cat <<EOF >> ~/.zshrc

# k8s alias

source <(kubectl completion zsh)

alias k=kubectl
alias vi=vim
alias kn='kubectl config set-context --current --namespace'
alias kc='kubectl config use-context'
alias kcg='kubectl config get-contexts'
alias di='docker images --format "table {{.Repository}}:{{.Tag}}\t{{.ID}}\t{{.Size}}\t{{.CreatedSince}}"'
EOF
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

### 6. Register inventory connector, start watch-topic customer
```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "topic.prefix": "dbserver1", "database.include.list": "inventory", "schema.history.internal.kafka.bootstrap.servers": "kafka:9092", "schema.history.internal.kafka.topic": "schemahistory.inventory" } }'


docker run -it --rm --name watcher --link zookeeper:zookeeper --link kafka:kafka quay.io/debezium/kafka:2.1 watch-topic -a -k dbserver1.inventory.customers
```


### 6. Update & Delete Data
```bash
UPDATE customers SET first_name='Anne Marie' WHERE id=1004;
DELETE FROM addresses WHERE customer_id=1004;
DELETE FROM customers WHERE id=1004;

INSERT INTO customers VALUES (default, "Sarah", "Thompson", "kitt@acme.com");
INSERT INTO customers VALUES (default, "Kenneth", "Anderson", "kander@acme.com");
```


