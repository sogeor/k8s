# k8s

Данный репозиторий содержит глобальные файлы конфигураций, необходимые для развертывания инфраструктуры.

## Развертывание инфраструктуры

> Обратите внимание: все действия выполняются на операционной системе `Ubuntu 24.04.3 LTS` от имени пользователя `root`.

Для того чтобы развернуть инфраструктуру, выполните следующие команды:

```shell
# change ssh port
nano sshd_config # uncomment `Port 22` & set it to `1024`
systemctl daemod-reload
systemctl restart ssh.socket

# firewall
ufw default deny incoming
ufw default allow outgoing
ufw allow 1024/tcp
ufw allow 6443/tcp
ufw allow 10250/tcp
ufw enable

# cri-o & k8s pkgs repos
apt-get update
apt-get install -y software-properties-common curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v1.34/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v1.34/deb/ /" | tee /etc/apt/sources.list.d/cri-o.list

# cri-o & k8s pkgs
apt-get update
apt-get install -y cri-o kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl cri-o

# cri-o
systemctl start crio.service
nano /etc/crio/crio.conf.d/*.conf # add `pause_image="registry.k8s.io/pause:3.10.1"` after `[crio.image]`
systemctl restart crio.service

# k8s env
swapoff -a
sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab
modprobe br_netfilter
echo "br_netfilter" >> /etc/modules
echo 1 > /proc/sys/net/ipv4/ip_forward

# k8s repo
apt install -y git
cd /home
git config credential.helper store
git clone https://github.com/sogeor/k8s.git

# k8s cluster
cd /home/k8s
kubeadm init --config misc/kubeadm/init.yaml
export KUBECONFIG=/etc/kubernetes/admin.conf

# k8s env
nano ~/.bashrc # add to eof `export KUBECONFIG=/etc/kubernetes/admin.conf`

# k8s post-setup
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

Теперь необходимо прописать следующую команду:

```shell
kubectl edit configmap -n kube-system kube-proxy
```

Далее найти часть конфигурации, содержащую:

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration 
```

После этого задать следующие значения:

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
ipvs:
  strictARP: true
```

Далее введите следующие команды:

```shell
# k8s proxy
kubectl delete pod -n kube-system -l k8s-app=kube-proxy

# flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/download/v0.27.4/kube-flannel.yml

# metallb
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml
cd /home/k8s
kubectl apply -f misc/metallb

# metrics-server
cd /home/k8s
kubectl apply -f misc/metrics-server

# istio
cd
curl -L https://istio.io/downloadIstio | sh -
mv istio-*/ istio/
export PATH=~/istio/bin:$PATH

# istio env
nano ~/.bashrc # add to eof `export PATH=~/istio/bin:$PATH`

# istio install
cd /home/k8s
istioctl install -f misc/istioctl/install.yaml

# cert manager
kubectl apply --server-side -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.1/cert-manager.yaml
curl -fsSL -o cmctl https://github.com/cert-manager/cmctl/releases/latest/download/cmctl_linux_amd64
chmod +x cmctl
sudo mv cmctl /usr/local/bin

# configure istio gateway & cert manager
kubectl apply -f misc/istio-system

# prometheus operator
kubectl apply -f misc/monitoring/namespace.yaml
TMPDIR=$(mktemp -d)
LATEST=$(curl -s https://api.github.com/repos/prometheus-operator/prometheus-operator/releases/latest | jq -cr .tag_name)
cd $TMPDIR
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
curl -s "https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/refs/tags/$LATEST/kustomization.yaml" > kustomization.yaml
curl -s "https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/refs/tags/$LATEST/bundle.yaml" > bundle.yaml
./kustomize edit set namespace monitoring && kubectl create -k "$TMPDIR"

# postgres operator
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.27/releases/cnpg-1.27.1.yaml
kubectl apply -f misc/cnpg-system

# misc
kubectl apply -f misc/namespace.yaml
kubectl apply -f misc/native-storage.yaml
kubectl apply -f misc/nexus-docker-secret.yaml
kubectl apply -f misc/robots-virtual-service.yaml

# keycloak
mkdir -p /root/data/sso/postgres-cluster/volume-0
mkdir /root/data/sso/postgres-cluster/volume-1
mkdir /root/data/sso/postgres-cluster/volume-2
kubectl apply -f sso

# secrets reflector
kubectl -n kube-system apply -f https://github.com/emberstack/kubernetes-reflector/releases/latest/download/reflector.yaml

# prometheus
# TODO

# grafana
# TODO

# nexus
mkdir -p /root/data/nexus/nexus/volume
mkdir -p /root/data/nexus/postgres-cluster/volume-0
mkdir /root/data/nexus/postgres-cluster/volume-1
mkdir /root/data/nexus/postgres-cluster/volume-2
kubectl apply -f nexus
```
