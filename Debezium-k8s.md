### 1. Install Cluster

```bash
groupadd -g 2000 k8sadm
useradd -m -u 2000 -g 2000 -s /bin/bash k8sadm
echo -e "1\n1" | passwd k8sadm >/dev/null 2>&1
echo ' k8sadm ALL=(ALL)   ALL' >> /etc/sudoers

apt update && apt install docker.io -y
usermod -aG docker k8sadm
newgrp docker

curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.21.10+k3s1 sh -s - --docker --write-kubeconfig-mode 644

mkdir -p ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod 755 kubectl && sudo mv kubectl /usr/local/bin
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

cat <<EOF >> ~/.bashrc
# k8s alias
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
alias k=kubectl
alias vi=vim
alias kn='kubectl config set-context --current --namespace'
alias kc='kubectl config use-context'
alias kcg='kubectl config get-contexts'
alias di='docker images --format "table {{.Repository}}:{{.Tag}}\t{{.ID}}\t{{.Size}}\t{{.CreatedSince}}"'
EOF

source ~/.bashrc
```

### 2. Install Kafka Helm chart

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install kafka bitnami/kafka -n kafka --create-namespace



kubectl run -it kakfa-kafka-client --restart='Never' --image docker.io/bitnami/kafka:3.3.1-debian-11-r25 --namespace kafka -- bash
kafka-console-producer.sh --broker-list kafka-headless.kafka.svc.cluster.local:9092 --topic test
kafka-console-consumer.sh --bootstrap-server kafka-headless.kafka.svc.cluster.local:9092 --topic test --from-beginning

```

### 3. Install Kafdrop

```bash
helm repo add rhcharts https://ricardo-aires.github.io/helm-charts/
helm upgrade --install kafdrop --set kafka.enabled=false --set kafka.bootstrapServers=kafka-headless.kafka.svc.cluster.local:9092 rhcharts/kafdrop

cat << EOF | kubectl create -n kafka -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kafdrop
  namespace: kafka
spec:
  rules:
    - host: kafdrop.kw01
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kafdrop
                port:
                  number: 9000
EOF
```

### 4. Install Mysql Example

```bash
cat << EOF | kubectl create -n kafka -f -
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: quay.io/debezium/example-mysql:2.1
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: debezium
        - name: MYSQL_USER
          value: mysqluser
        - name: MYSQL_PASSWORD
          value: mysqlpw
        ports:
        - containerPort: 3306
          name: mysql
EOF

kubectl exec -it $(kubectl get pods -l app=mysql -o name) -- mysql -uroot -p
password: debezium

use invetntory
show tables;
select * from customers;

```

### 5. Install Kafka Connect

```bash
cat << EOF | kubectl create -n kafka -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: debezium-connect
  namespace: kafka
spec:
  selector:
    matchLabels:
      app: debezium-connect
  template:
    metadata:
      labels:
        app: debezium-connect
    spec:
      containers:
      - env:
        - name: GROUP_ID
          value: "1"
        - name: CONFIG_STORAGE_TOPIC
          value: my_connect_configs
        - name: OFFSET_STORAGE_TOPIC
          value: my_connect_offsets
        - name: STATUS_STORAGE_TOPIC
          value: my_connect_statuses
        image: quay.io/debezium/connect:2.1
        imagePullPolicy: IfNotPresent
        name: debezium-connect
        ports:
        - containerPort: 8083
          name: tcp
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: debezium-connect
spec:
  ports:
  - port: 8083
    targetPort: 8083
    nodePort: 30883
  selector:
    app: debezium-connect
  type: NodePort
EOF
```

### 6. Install Debezium MySQL Connector

```bash
curl -X DELETE http://localhost:30883/connectors/inventory-connector

curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:30883/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "topic.prefix": "dbserver1", "database.include.list": "inventory", "schema.history.internal.kafka.bootstrap.servers": "kakfa-kafka:9092", "schema.history.internal.kafka.topic": "schema-changes.inventory" } }'
```

### 7. Update Database

```bash
kubectl exec -it $(kubectl get pods -l app=mysql -o name) -- mysql -uroot -p
password: debezium

UPDATE customers SET first_name='Anne Marie' WHERE id=1004;
DELETE FROM addresses WHERE customer_id=1004;
DELETE FROM customers WHERE id=1004;

INSERT INTO customers VALUES (default, "Sarah", "Thompson", "kitt@acme.com");
INSERT INTO customers VALUES (default, "Kenneth", "Anderson", "kander@acme.com");

Check Kafdrop : http://kafdrop.kw01/topic/dbserver1.inventory.customers/messages
```

### 8. JDBC sync to PostgreSQL - Work In Progress
Build debezium connect with jdbc connenctor
https://github.com/debezium/debezium-examples/tree/main/unwrap-smt
https://github.com/debezium/debezium-examples/blob/main/unwrap-smt/debezium-jdbc-es/Dockerfile

