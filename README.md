#  EFK Logging on AWS EKS and 11 log monitor Demo Project with K8s AWS EKS.



## For more projects, check out  
[https://harishnshetty.github.io/projects.html](https://harishnshetty.github.io/projects.html)

[![Video Tutorial](https://github.com/harishnshetty/image-data-project/blob/5d2e06ffa3dd7687607b0c7d4892a6b161c077f9/10%20microservice%20online%20shop%20project.jpg)](https://youtu.be/KNH_qe1vJAg)

## 🔍 Overview
EFK logging is crucial in any distributed system, especially in Kubernetes, to monitor application behavior, detect issues, and ensure the smooth functioning of microservices.

## 🚀 Importance
- **Debugging**: Logs provide critical information for troubleshooting.
- **Auditing**: Logs serve as an audit trail, showing actions and actors.
- **Performance Monitoring**: Analyzing logs helps identify bottlenecks.
- **Security**: Logs help detect unauthorized access or malicious activity.

## 🛠️ Logging Tools for Kubernetes
- **EFK Stack** (Elasticsearch, Fluent Bit, Kibana)
- **EFK Stack** (Elasticsearch, FluentD, Kibana)
- **ELK Stack** (Elasticsearch, Logstash, Kibana)
- **Promtail + Loki + Grafana**

## 📦 EFK Stack (Elasticsearch, Fluent Bit, Kibana)
- **Elasticsearch**: Stores and indexes log data.
- **Fluent Bit**: Lightweight log forwarder that collects logs and sends them to Elasticsearch.
- **Kibana**: Visualization tool for exploring and analyzing logs.

---

## System Update & Common Packages

```bash
sudo apt update
sudo apt upgrade -y

# Common tools
sudo apt install -y bash-completion wget git zip unzip curl jq net-tools build-essential ca-certificates apt-transport-https gnupg fontconfig
```
Reload bash completion if needed:
```bash
source /etc/bash_completion
```

**Install latest Git:**
```bash
sudo add-apt-repository ppa:git-core/ppa
sudo apt update
sudo apt install git -y
```

## 1. AWS CLI Installation

Refer: [AWS CLI Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

```bash
sudo apt install -y unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

---

## 2. kubectl Installation

Refer: [kubectl Installation Guide](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

```bash
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

# If the folder `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly

sudo apt-get update
sudo apt-get install -y kubectl bash-completion

# Enable kubectl auto-completion
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -F __start_kubectl k' >> ~/.bashrc

# Apply changes immediately
source ~/.bashrc
```

---

## 3. eksctl Installation

Refer: [eksctl Installation Guide](https://eksctl.io/installation/)

```bash
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl

# Install bash completion
sudo apt-get install -y bash-completion

# Enable eksctl auto-completion
echo 'source <(eksctl completion bash)' >> ~/.bashrc
echo 'alias e=eksctl' >> ~/.bashrc
echo 'complete -F __start_eksctl e' >> ~/.bashrc

# Apply changes immediately
source ~/.bashrc
```

---

## 4. Helm Installation

Refer: [Helm Installation Guide](https://helm.sh/docs/intro/install/)

```bash
sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm bash-completion

# Enable Helm auto-completion
echo 'source <(helm completion bash)' >> ~/.bashrc
echo 'alias h=helm' >> ~/.bashrc
echo 'complete -F __start_helm h' >> ~/.bashrc

# Apply changes immediately
source ~/.bashrc
```

---

## 5. AWS CLI Configuration

```bash
aws configure
aws configure list
```


## 🚀 Cluster Setup

### 1. Launch the EKS Cluster

```bash
eksctl create cluster -f 0-eks-creation-config.yml
```
> EKS ensures each AZ gets at least one node. If you request 3 worker nodes across 3 AZs, one AZ will get a second node.

Check node distribution:
```bash
aws eks update-kubeconfig --name my-cluster --region ap-south-1
kubectl get nodes --show-labels | grep topology.kubernetes.io/zone
```

### 2. Install the AWS EBS CSI Plugin

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.44"
```

Verify deployment:
```bash
kubectl get daemonset ebs-csi-node -n kube-system
kubectl get deploy ebs-csi-controller -n kube-system
```

**Plugin Components:**
- `ebs-csi-node`: DaemonSet on every worker node for attaching/mounting EBS volumes.
- `ebs-csi-controller`: Deployment for volume provisioning and lifecycle management.

---

## 📦 Deploying the EFK Stack

### 3. Install Elasticsearch

```bash
# Create namespace if it doesn’t exist
kubectl create namespace efk-logging --dry-run=client -o yaml | kubectl apply -f -

# Add and update Elastic Helm repo
helm repo add elastic https://helm.elastic.co
helm repo update

# Install/upgrade Elasticsearch
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
- Installs Elasticsearch in the `efk-logging` namespace with persistence enabled.

### 4. Retrieve Elasticsearch Credentials

```bash
# Username
kubectl get secrets --namespace=efk-logging elasticsearch-master-credentials -ojsonpath='{.data.username}' | base64 -d
# Password
kubectl get secrets --namespace=efk-logging elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d
```
> **Note:** Save the password for future reference.

### 5. Install Kibana

```bash
helm upgrade --install kibana elastic/kibana \
  --namespace efk-logging \
  --set service.type=LoadBalancer \
  --set replicaCount=1 \
  --wait
```
- Kibana is exposed as a LoadBalancer service for external access.

### 6. Install Fluent Bit

> **Note:** Update the `HTTP_Passwd` field in `1-fluentbit-values.yml` with the password from step 4.

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm install fluent-bit fluent/fluent-bit \
  -f 1-fluentbit-values.yml \
  -n efk-logging
```

Check services:
```bash
kubectl get svc -n efk-logging
```
- Access Kibana at `http://<load-balancer-dns>:5601`

---

## 🚀 Deploy a Sample Python app

```bash
kubectl create namespace app
kubectl apply -f 2-app.yml -n microservice
```

---

## 🧼 Clean Up

```bash
helm uninstall fluent-bit -n efk-logging
helm uninstall elasticsearch -n efk-logging
helm uninstall kibana -n efk-logging
eksctl delete cluster -f 0-eks-creation-config.yml
```