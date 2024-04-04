# Single host Kubernetes cluster
Sets up a single host Kubernets cluster using Debian and k3s. With Ingress support, metrics(Prometehus), load balancer(MetalLB) and logging(Elasticsearch, fluent-bit and Kibana or Opensearch with fluent-bit).
## Install Host OS
Install Debain with SSH server setup and no desktop environment on a VM or bare-metal.
### Key based authentication setup
TODO
ssh-keygen -t rsa
### Tools
```shell
su root
apt install -y sudo htop curl ca-certificates apt-transport-https gpg
```

####
Add user to sudoers

```shell
/usr/sbin/usermod -aG sudo <your username>
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
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl=1.27*
```

#### K3s
```shell
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=v1.27 INSTALL_K3S_EXEC="server --disable traefik" sh -
```

##### Configure kubectl
```shell
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && sudo chown $USER ~/.kube/config
sudo chmod 600 ~/.kube/config && export KUBECONFIG=~/.kube/config
```

##### Helm setup
```shell
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt install helm=3.12.*
```

## Kubernetes plugins
### Ingress
Ingress is used to expose HTTP and HTTPS services within the cluster.
```shell
kubectl create namespace ingress
helm upgrade --install ingress -n ingress oci://ghcr.io/nginxinc/charts/nginx-ingress --set=controller.ingressClass=public
```

### MetalLB
MetalLB allows us to give each externally avalible service a seperate IP.
```shell
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb
```

### Metrics
#### Prometheus
For metrics we will use Prometheus.
```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install kibana bitnami/kibana -n logging \
  --set server.ingress.enabled=true \
  --set server.ingress.hosts[0]=prometheus.local \
  --set server.ingress.ingressClassName=public
```

#### Grafana
```shell
kubectl create configmap memory-dashboard --from-file=memory-dashboard.json -n prometheus
helm install grafana bitnami/grafana -n prometheus \
  --set ingress.enabled=true \
  --set ingress.hosts[0]=prometheus.local \
  --set ingress.ingressClassName=public \
  --set admin.user=admin \
  --set admin.password=admin \
  --set dashboardsProvider.enabled=true \
  --set dashboardsConfigMaps[0].configMapName=memory-dashboard \
  --set dashboardsConfigMaps[0].fileName=memory-dashboard.json \
  --values grafana-values.yaml
```
Username: admin

Password: admin

Url: 

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
helm install opensearch opensearch/opensearch -n logging \
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
Password: admin

Url: http://opensearch.local/app/discover
##### fluent-bit opensearch
```shell
helm repo add fluent https://fluent.github.io/helm-charts
helm upgrade --install fluent-bit fluent/fluent-bit \
  -n logging \
  --values fluent-bit-values-opensearch.yaml
```
