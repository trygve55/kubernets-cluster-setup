# Single host Kubernetes cluster
Sets up a single host Kubernets cluster using Debian and k3s. With Ingress support, metrics(Prometehus), load balancer(MetalLB) and logging(Elasticsearch, fluent-bit and Kibana or Opensearch with fluent-bit).
## Install Host OS
Install Debain with SSH server setup and no desktop environment on a VM or bare-metal.

Tested with Debian 12.10.0
### Key based authentication setup
TODO
ssh-keygen -t rsa
### Tools
```shell
sudo apt install -y htop git
```

### Clone Git repo
```shell
git clone https://github.com/trygve55/kubernets-cluster-setup.git
```

### Unnatended update
```shell
sudo apt install unattended-upgrades apt-listchanges
```
### Setup email notification
TODO
https://medium.com/@carlesanagustin/sending-emails-from-ssmtp-with-a-gmail-account-cbefdcac91f8
You should at least uncomment the following line:
Unattended-Upgrade::Mail "root";

### Setup K8s and related utilites
#### kubectl
```shell
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubectl=1.32.*
```

#### K3s
K3s is a lightweight Kubernetes distribution. We will install it.
```shell
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=v1.32 INSTALL_K3S_EXEC="server --disable traefik" sh -
```

##### Configure kubectl
We use Kubectl, the Kubernetes command-line tool for communicating with a Kubernetes cluster's control plane.
```shell
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && sudo chown $USER ~/.kube/config
sudo chmod 600 ~/.kube/config && export KUBECONFIG=~/.kube/config
```
###### Test kubectl
Test that kubectl can connect to Kubernetes.
```shell
kubectl events
```

###### K9s (optional)
K9s is a terminal based UI to interact with your Kubernetes clusters.
```shell
wget https://github.com/derailed/k9s/releases/download/v0.50.2/k9s_linux_amd64.deb
sudo apt install ./k9s_linux_amd64.deb
rm ./k9s_linux_amd64.deb

k9s version # Test that the install was a success.
```

##### Helm setup
Helm is a tool for managing Kubernetes applications with charts.
```shell
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt install helm=3.17.*
```

## Kubernetes plugins
### Ingress
Ingress is used to expose HTTP and HTTPS services within the cluster.
```shell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress -n ingress --create-namespace ingress-nginx/ingress-nginx \
  --set controller.ingressClass=public \
  --set controller.ingressClassResource.name=public \
  --set controller.ingressClassResource.enabled=true \
  --set controller.ingressClassResource.default=true
```

### MetalLB
MetalLB allows us to give each externally avalible service a seperate IP.
```shell
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb -n metallb --create-namespace 
```

### Metrics
#### Prometheus
For metrics we will use Prometheus.
```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install prometheus prometheus-community/prometheus -n prometheus --create-namespace \
  --set server.ingress.enabled=true \
  --set server.ingress.hosts[0]=prometheus.local \
  --set server.ingress.ingressClassName=public
```

Url: http://prometheus.local/

#### Grafana-Loki
Grafana Loki is a fully featured logging stack.
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install grafana-loki bitnami/grafana-loki -n logging --create-namespace
```

#### Grafana
Grafana is a web application for viewing metrics and logs. It accesses logs gathered by Grafana Loki.
```shell
kubectl create namespace monitoring
kubectl create configmap memory-dashboard --from-file=memory-dashboard.json -n monitoring
helm install grafana bitnami/grafana -n monitoring \
  --set ingress.enabled=true \
  --set ingress.hosts[0]=grafana.local \
  --set ingress.ingressClassName=public \
  --set admin.user=admin \
  --set admin.password=admin \
  --set dashboardsProvider.enabled=true \
  --set dashboardsConfigMaps[0].configMapName=memory-dashboard \
  --set dashboardsConfigMaps[0].fileName=memory-dashboard.json \
  --values grafana-values.yaml
```
Username: `admin` <br>
Password: `admin` <br>
Url: http://grafana.local/

### Logging 
For managing logs you can either use EFK(Elasticsearch, fluentbit and Kibana) or Opensearch with fluentbit. Only use one of them!
```shell
kubectl create namespace logging
```
#### EFK
##### Elasticsearch
```shell
helm repo add elastic https://helm.elastic.co
helm install elasticsearch bitnami/elasticsearch -n logging \
  --set master.masterOnly=false \
  --set master.heapSize=128m \
  --set master.replicaCount=1 \
  --set data.replicaCount=0 \
  --set coordinating.replicaCount=0 \
  --set ingest.replicaCount=0 \
  --set metrics.enabled=true
```
##### fluent-bit
```shell
helm repo add fluent https://fluent.github.io/helm-charts
helm upgrade --install fluent-bit fluent/fluent-bit \
  -n logging \
  --values fluent-bit-values.yaml
```
##### Kibana
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install kibana bitnami/kibana -n logging \
  --set elasticsearch.hosts[0]=elasticsearch-master-hl \
  --set elasticsearch.port=9200 \
  --set ingress.enabled=true \
  --set ingress.ingressClassName=public
```

#### Opensearch
```shell
helm repo add opensearch https://opensearch-project.github.io/helm-charts/
helm install opensearch opensearch/opensearch -n logging --create-namespace \
  --set singleNode=true \
  --values opensearch-values.yaml
helm install opensearch-dashboards opensearch/opensearch-dashboards -n logging \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=opensearch.local \
  --set ingress.hosts[0].paths[0].path=/ \
  --set ingress.hosts[0].paths[0].backend.service.name=opensearch-cluster-master \
  --set ingress.ingressClassName=public \
  --values opensearch-dashboards-values.yaml
```
Username: admin

Password: OpensearchPassword123!

Url: http://opensearch.local/app/discover
##### fluent-bit opensearch
```shell
helm repo add fluent https://fluent.github.io/helm-charts
helm upgrade --install fluent-bit fluent/fluent-bit \
  -n logging \
  --values fluent-bit-values-opensearch.yaml
```
