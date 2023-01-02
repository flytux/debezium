### 1. Install Cluster

```bash
groupadd -g 2000 k8sadm
useradd -m -u 2000 -g 2000 -s /bin/bash k8sadm
echo -e "1\n1" | passwd k8sadm >/dev/null 2>&1
echo ' k8sadm ALL=(ALL)   ALL' >> /etc/sudoers

apt update && apt install docker.io -y
usermod -aG docker k8sadm

curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.21.10+k3s1 sh -s - --docker --write-kubeconfig-mode 644

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

### 2. Install stimzi operator

```bash
k create ns debizium

curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.20.0/install.sh | bash -s v0.20.0

kubectl create -f https://operatorhub.io/install/strimzi-kafka-operator.yaml
```

### 3. Install kafka

```bash
cat << EOF | kubectl create -n debezium -f -
apiVersion: v1
kind: Secret
metadata:
  name: debezium-secret
  namespace: debezium
type: Opaque
data:
  username: ZGViZXppdW0=
  password: ZGJ6
EOF

cat << EOF | kubectl create -n debezium -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: connector-configuration-role
  namespace: debezium
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["debezium-secret"]
  verbs: ["get"]
EOF

cat << EOF | kubectl create -n debezium -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: connector-configuration-role-binding
  namespace: debezium
subjects:
- kind: ServiceAccount
  name: debezium-connect-cluster-connect
  namespace: debezium
roleRef:
  kind: Role
  name: connector-configuration-role
  apiGroup: rbac.authorization.k8s.io
EOF


cat << EOF | kubectl create -n debezium -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: debezium-cluster
spec:
  kafka:
    replicas: 1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: tls
      - name: external
        port: 9094
        type: nodeport
        tls: false
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 1Gi
        deleteClaim: false
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
  zookeeper:
    replicas: 1
    storage:
      type: persistent-claim
      size: 1Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}
EOF


cat << EOF | kubectl create -n debezium -f -
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


cat << EOF | kubectl create -n debezium -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: debezium-connect-cluster
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  version: 3.1.0
  image: quay.io/lordofthejars/debezium-connector-mysql:1.9.4
  replicas: 1
  bootstrapServers: debezium-cluster-kafka-bootstrap:9092
  config:
    config.providers: secrets
    config.providers.secrets.class: io.strimzi.kafka.KubernetesSecretConfigProvider
    group.id: connect-cluster
    offset.storage.topic: connect-cluster-offsets
    config.storage.topic: connect-cluster-configs
    status.storage.topic: connect-cluster-status
    # -1 means it will use the default replication factor configured in the broker
    config.storage.replication.factor: -1
    offset.storage.replication.factor: -1
    status.storage.replication.factor: -1
EOF

cat << EOF | kubectl create -n debezium -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: debezium-connector-mysql
  labels:
    strimzi.io/cluster: debezium-connect-cluster
spec:
  class: io.debezium.connector.mysql.MySqlConnector
  tasksMax: 1
  config:
    tasks.max: 1
    database.hostname: mysql
    database.port: 3306
    database.user: ${secrets:debezium-example/debezium-secret:username}
    database.password: ${secrets:debezium-example/debezium-secret:password}
    database.server.id: 184054
    topic.prefix: mysql
    database.include.list: inventory
    schema.history.internal.kafka.bootstrap.servers: debezium-cluster-kafka-bootstrap:9092
    schema.history.internal.kafka.topic: schema-changes.inventory
EOF

kubectl run -n debezium -it --rm --image=quay.io/debezium/tooling:1.2  --restart=Never watcher -- kcat -b debezium-cluster-kafka-bootstrap:9092 -C -o beginning -t mysql.inventory.customers
