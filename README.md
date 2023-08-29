# server-setup
### Install OS
Install Debain with SSH server setup and no desktop environment.
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

#### Unnatended update
sudo apt install unattended-upgrades apt-listchanges
##### Setup email notification
TODO
https://medium.com/@carlesanagustin/sending-emails-from-ssmtp-with-a-gmail-account-cbefdcac91f8
You should at least uncomment the following line:
Unattended-Upgrade::Mail "root";

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

#### Kubernetes plugins
##### Ingress
```shell
kubectl create namespace ingress
helm upgrade --install ingress -n ingress oci://ghcr.io/nginxinc/charts/nginx-ingress --set=controller.ingressClass=public
```

##### Logging
```shell
kubectl create namespace fluentd
```
###### Elasticsearch
```shell
helm repo add elastic https://helm.elastic.co
helm install elasticsearch bitnami/elasticsearch -n logging \
  --set master.replicaCount=1 \
  --set data.replicaCount=1 \
  --set coordinating.replicaCount=1 \
  --set ingest.replicaCount=1
```
###### fluent-bit
```shell
helm repo add fluent https://fluent.github.io/helm-charts
helm upgrade --install fluent-bit fluent/fluent-bit \
  -n logging \
  --values fluent-bit-values.yaml
```
###### Kibana
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install kibana bitnami/kibana -n logging \
  --set elasticsearch.hosts[0]=elasticsearch-master-hl \
  --set elasticsearch.port=9200 \
  --set ingress.enabled=true \
  --set ingress.ingressClassName=public
```
