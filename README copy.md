# efk-logging-efk-logging-AWS-EKS-Project

# 🔍 efk-logging overview
- efk-logging is crucial in any distributed system, especially in Kubernetes, to monitor application behavior, detect issues, and ensure the smooth functioning of microservices.


## 🚀 Importance:
- **Debugging**: Logs provide critical information when debugging issues in applications.
- **Auditing**: Logs serve as an audit trail, showing what actions were taken and by whom.
- **Performance** Monitoring: Analyzing logs can help identify performance bottlenecks.
- **Security**: Logs help in detecting unauthorized access or malicious activities.

## 🛠️ Tools Available for efk-logging in Kubernetes
- 🗂️ efk-logging Stack (Elasticsearch, Fluentbit, Kibana)
- 🗂️ efk-logging Stack (Elasticsearch, FluentD, Kibana)
- 🗂️ ELK Stack (Elasticsearch, Logstash, Kibana)
- 📊 Promtail + Loki + Grafana

## 📦 efk-logging Stack (Elasticsearch, Fluentbit, Kibana)
- efk-logging is a popular efk-logging stack used to collect, store, and analyze logs in Kubernetes.
- **Elasticsearch**: Stores and indexes log data for easy retrieval.
- **Fluentbit**: A lightweight log forwarder that collects logs from different sources and sends them to Elasticsearch.
- **Kibana**: A visualization tool that allows users to explore and analyze logs stored in Elasticsearch.




Launch the cluster:

```bash
eksctl create cluster -f 0-eks-creation-config.yml
```

> Although we specify **3 AZs**, we request **3 worker nodes**. EKS ensures each AZ gets at least one node, with one AZ receiving a second node.

Check node distribution:

```bash
aws eks update-kubeconfig --name my-cluster --region ap-south-1
```
```bash
kubectl get nodes --show-labels | grep topology.kubernetes.io/zone
```


install the CSI plugin:

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.44"
```

Verify successful deployment:

```bash
kubectl get daemonset ebs-csi-node -n kube-system
kubectl get deploy ebs-csi-controller -n kube-system
```

#### Plugin Components

* **`ebs-csi-node`**
  A **DaemonSet** that runs on every worker node. It handles **attaching and mounting** EBS volumes so pods can access data locally.

* **`ebs-csi-controller`**
  A **central Deployment** that manages volume **provisioning and lifecycle**, coordinating with the AWS API.

---



### 5) Install Elasticsearch on K8s

```bash
# 1️⃣ Create namespace if it doesn’t exist
kubectl create namespace efk-logging --dry-run=client -o yaml | kubectl apply -f -

# 2️⃣ Add and update Elastic Helm repo
helm repo add elastic https://helm.elastic.co
helm repo update

# 3️⃣ Install or upgrade Elasticsearch (safe for existing PVCs)
helm upgrade --install elasticsearch elastic/elasticsearch \
  -n efk-logging \
  --set replicas=1 \
  --set persistence.enabled=true \
  --set persistence.labels.enabled=true \
  --set volumeClaimTemplate.storageClassName=gp2 \
  --set resources.requests.memory=2Gi \
  --set resources.requests.cpu=1000m \
  --set resources.limits.memory=2Gi \
  --set resources.limits.cpu=1000m \
  --set esJavaOpts="-Xms1g -Xmx1g" \
  --wait

```
```bash
helm repo add elastic https://helm.elastic.co

helm install elasticsearch \
 --set replicas=1 \
 --set volumeClaimTemplate.storageClassName=gp2 \
 --set persistence.labels.enabled=true elastic/elasticsearch -n logging

 ```
- Installs Elasticsearch in the `efk-logging` namespace.
- It sets the number of replicas, specifies the storage class, and enables persistence labels to ensure
data is stored on persistent volumes.

### 6) Retrieve Elasticsearch Username & Password
```bash
# for username
kubectl get secrets --namespace=efk-logging elasticsearch-master-credentials -ojsonpath='{.data.username}' | base64 -d
# for password
kubectl get secrets --namespace=efk-logging elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d
```
- Retrieves the password for the Elasticsearch cluster's master credentials from the Kubernetes secret.
- The password is base64 encoded, so it needs to be decoded before use.
- 👉 **Note**: Please write down the password for future reference

### 7) Install Kibana
```bash
helm upgrade --install kibana elastic/kibana \
  --namespace efk-logging \
  --set service.type=LoadBalancer \
  --set replicaCount=1 \
  --wait

```
- Kibana provides a user-friendly interface for exploring and visualizing data stored in Elasticsearch.
- It is exposed as a LoadBalancer service, making it accessible from outside the cluster.

### 8) Install Fluentbit with Custom Values/Configurations
- 👉 **Note**: Please update the `HTTP_Passwd` field in the `fluentbit-values.yml` file with the password retrieved earlier in step 6: (i.e NJyO47UqeYBsoaEU)"
```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm install fluent-bit fluent/fluent-bit \
  -f 1-fluentbit-values.yml \
  -n efk-logging

```

```bash
kubectl get svc
```
## http://alb:5601


```bash
kubectl create namespace microservice
kubectl create -f 4-microservice-deployment.yml -n microservice
```

## 🧼 Clean Up
```bash


helm uninstall fluent-bit -n efk-logging

helm uninstall elasticsearch -n efk-logging

helm uninstall kibana -n efk-logging

```


```bash
eksctl delete cluster -f 0-eks-creation-config.yml
```