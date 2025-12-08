# k8s

Данный репозиторий содержит глобальные файлы конфигураций, необходимые для развертывания инфраструктуры.

## Развертывание инфраструктуры

> Обратите внимание: все действия выполняются на операционной системе `Ubuntu 24.04.3 LTS` от имени пользователя `root`.

### Подготовка основной части

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

# secrets reflector
kubectl -n kube-system apply -f https://github.com/emberstack/kubernetes-reflector/releases/latest/download/reflector.yaml

# misc
kubectl apply -f misc/namespace.yaml
kubectl apply -f misc/native-storage.yaml
kubectl apply -f misc/nexus-docker-secret.yaml
kubectl apply -f misc/robots-virtual-service.yaml
```

### Настройка Cloudflare

Для того чтобы настроить Cloudflare, выполните следующие действия:

1. Войдите в Cloudflare Dashboard по адресу: https://dash.cloudflare.com.
2. Выберите раздел `Account API tokens`.
3. Нажмите `Create Token`.
4. Возле `Edit zone DNS` нажмите `Use template`.
5. В 3-м разделе поля `Zone Resources` выберите ваш домен (sogeor.com).
6. Нажмите `Continue to summary`.
7. Нажмите `Create Token`.
8. Скопируйте API ключ и сохраните его для дальнейших действий.

Чтобы Cert Manager мог работать с DNS записями домена самостоятельно, выполните следующие команды:

```shell
CLOUDFLARE_TOKEN= # Укажите здесь API токен, полученный на 8 шаге при работе с Cloudflare
kubectl create secret generic cloudflare -n istio-system \
        --type='Opaque' \
        --from-literal=token=${CLOUDFLARE_TOKEN}
```

> Чтобы задать свой логин Cloudflare, обратитесь к полю `email` в файле `misc/istio-system/letsencrypt.yaml`.

### Развертывание SSO сервера

Для того чтобы начать работу с SSO сервером, выполните следующие команды:

```shell
kubectl create secret generic keycloak-postgres-cluster-role -n sso \
        --type='kubernetes.io/basic-auth' \
        --from-literal=username='keycloak' \
        --from-literal=password=$(shuf -er -n32  {A..Z} {a..z} {0..9} | tr -d '\n')
kubectl label secret keycloak-postgres-cluster-role -n sso "cnpg.io/reload=true"
mkdir -p /root/data/sso/postgres-cluster/volume-0
mkdir /root/data/sso/postgres-cluster/volume-1
mkdir /root/data/sso/postgres-cluster/volume-2
cd /home/k8s
git pull
kubectl apply -f sso
```

### Настройка SSO сервера

Из-за Istio первый вход в панель администратора не удастся, пока будет существовать `RequestAuthentication`. Собственно
говоря, чтобы настроить Keycloak, введите следующие команды:

```shell
cd /home/k8s
git pull
kubectl detele -f misc/istio-system/request-authentication.yaml
```

Теперь создайте пароль для bootstrap пользователя, используя следующие команды:

```shell
KC_BOOTSTRAP_PASSWORD= # Укажите здесь любой пароль для bootstrap пользователя
kubectl create secret generic keycloak-bootstrap-secret -n sso \
        --type='kubernetes.io/basic-auth' \
        --from-literal=username='bootstrap' \
        --from-literal=password=${KC_BOOTSTRAP_PASSWORD}
```

Далее войдите в панель администратора, расположенную по адресу: https://sso.sogeor.com/admin. Выполните следующие
действия в области `master`:

> При входе в панель используйте логин `bootstrap` и пароль, указанный на предыдущем шаге.

1. Во вкладке `Groups` нажмите `Create group`.
2. В открывшемся окне введите в поле имени `Cluster Operators` и нажмите `Create`.
3. Выберите группу `Cluster Operators` в списке доступных.
4. В разделе `Role mapping` нажмите `Assign role`, выберите `Realm roles`.
5. В открывшемся окне поставьте галочку напротив роли `admin` и нажмите `Assign`.
6. Во вкладке `Users` нажмите `Add user`.
7. В поле `Required user actions` добавьте `Update Password`.
8. В поле `Email verified` установите переключатель в положение `On`.
9. Заполните поля `Username`, `Email`, `First name`, `Last name`.
10. Нажмите `Join Groups`.
11. В открывшемся окне поставьте галочку напротив группы `Cluster Operators` и нажмите `Join`.
12. Нажмите `Create`.
13. В разделе `Credentials` нажмите `Set password`.
14. В поля `Password` и `Password confirmation` введите временный пароль.
15. В поле `Temporary` установите переключатель в положение `On`.
16. Нажмите `Save`, потом `Save password`.
17. В правом верхнем углу нажмите на `bootstrap`, потом на `Sign out`.
18. Войдите в свой аккаунт, используя временный пароль, заданный на 14 шаге.
19. В разделе `Users` поставьте галочку напротив пользователя `bootstrap` и нажмите `Delete user`.
20. В разделе `Manage realms` нажмите `Create realm`.
21. В поле `Realm name` введите `public`.
22. Нажмите `Create`.
23. Нажмите `Create realm`.
24. В поле `Realm name` введите `system`.
25. Нажмите `Create`.
26. В разделе `Clients` нажмите `Create client`.
27. В поле `Client ID` введите `service-oauth2-proxy`.
28. Нажмите `Next`.
29. В поле `Client authentication` переключите переключатель в положение `On`.
30. В поле `Authentication flow` оставьте галочку только у `Standard flow`.
31. В поле `PKCE Method` выберите `S256`.
32. Нажмите `Next`.
33. В поле `Valid redirect URIs` добавьте `https://service.sogeor.com/oauth2/callback`.
34. Нажмите `Save`.
35. В разделе `Credentials` скопируйте и сохраните пароль из поля `Client Secret` — он понадобится для настройки OAuth
    Proxy для мониторинга.
36. Во вкладке `Realm roles` нажмите `Create role`.
37. В поле `Role name` задайте `operator`.
38. Нажмите `Save`.
39. Во вкладке `Groups` нажмите `Create group`.
40. В открывшемся окне введите в поле имени `Cluster Operators` и нажмите `Create`.
41. Выберите группу `Cluster Operators` в списке доступных.
42. В разделе `Role mapping` нажмите `Assign role`, выберите `Realm roles`.
43. В открывшемся окне поставьте галочку напротив роли `operator` и нажмите `Assign`.
44. Во вкладке `Users` нажмите `Create new user`.
45. В поле `Required user actions` добавьте `Update Password`.
46. В поле `Email verified` установите переключатель в положение `On`.
47. Заполните поля `Username`, `Email`, `First name`, `Last name`.
48. Нажмите `Join Groups`.
49. В открывшемся окне поставьте галочку напротив группы `Cluster Operators` и нажмите `Join`.
50. Нажмите `Create`.
51. В разделе `Credentials` нажмите `Set password`.
52. В поля `Password` и `Password confirmation` введите временный пароль и запомните его — он понадобится при входе в
    OAuth Proxy для мониторинга.
53. В поле `Temporary` установите переключатель в положение `On`.
54. Нажмите `Save`, потом `Save password`.

Далее необходимо применить `RequestAuthentication` обратно. Для этого выполните следующие команды:

```shell
cd /home/k8s
git pull
kubectl apply -f misc/istio-system/request-authentication.yaml
```

### Развертывание Prometheus

Для того чтобы начать работу с Prometheus, выполните следующие команды:

```shell
cd /home/k8s
git pull
kubectl apply -f misc/monitoring/prometheus
```

### Развертывание Alert Manager

Для того чтобы начать работу с Alert Manager, выполните следующие команды:

```shell
cd /home/k8s
git pull
kubectl apply -f misc/monitoring/alertmanager
```

### Развертывание OAuth Proxy для мониторинга

Для того чтобы начать работу с OAuth Proxy, выполните следующие команды:

```shell
KC_SERVICE_OAUTH2_PROXY_PASSWORD= # Укажите здесь пароль, полученный на 35 шаге при настройке Keycloak
kubectl create secret generic oauth2-proxy -n service \
        --type='Opaque' \
        --from-literal=client-id='service-oauth2-proxy' \
        --from-literal=client-secret=${KC_SERVICE_OAUTH2_PROXY_PASSWORD} \
        --from-literal=cookie-secret=$(shuf -er -n32  {A..Z} {a..z} {0..9} | tr -d '\n')
cd /home/k8s
git pull
kubectl apply -f service/oauth2-proxy
```

### Развертывание Grafana

Для того чтобы начать работу с Grafana, выполните следующие команды:

```shell
mkdir -p /root/data/service/grafana/volume
cd /home/k8s
git pull
kubectl apply -f service/grafana
```

### Настройка Grafana

Войдите в аккаунт администратора по адресу: https://service.sogeor.com/grafana. В качестве логина и пароля используйте
`admin`. Поменяйте имя пользователя и прочие данные в профиле, если необходимо. Установите язык на русский.

Далее необходимо подключить Prometheus к Grafana. Для этого выполните следующие действия:

1. В разделе подключения выберите `Добавление нового подключения`, потом `Prometheus`.
2. Нажмите `Добавить новый источник данных`.
3. В поле `Prometheus server URL` введите `http://prometheus.monitoring.svc.cluster.local:8080/prometheus`.
4. Нажмите `Сохранить и протестировать`.

Также можно добавить некоторые панели в соответствующем разделе `Дашборды`, если требуется.

### Развертывание Nexus Repository Manager

Для того чтобы подготовить Nexus к работе, а также создать возможность для мониторинга его состояния, введите следующие
команды:

```shell
kubectl create secret generic nexus-postgres-cluster-role -n nexus \
        --type='kubernetes.io/basic-auth' \
        --from-literal=username='nexus' \
        --from-literal=password=$(shuf -er -n32  {A..Z} {a..z} {0..9} | tr -d '\n')
kubectl label secret nexus-postgres-cluster-role -n nexus "cnpg.io/reload=true"
mkdir -p /root/data/nexus/nexus/volume
mkdir -p /root/data/nexus/postgres-cluster/volume-0
mkdir /root/data/nexus/postgres-cluster/volume-1
mkdir /root/data/nexus/postgres-cluster/volume-2
cd /home/k8s
git pull
kubectl apply -f nexus
```

### Настройка Nexus Repository Manager

Для того чтобы подготовить Nexus к работе, а также создать возможность для мониторинга его состояния, введите следующие
команды:

Далее войдите в аккаунт администратора, перейдя по адресу: https://nexus.sogeor.com и нажав на иконку человека в правом
верхнем углу. Чтобы посмотреть временный пароль администратора, введите следующую команду:

```shell
cat /root/data/nexus/nexus/volume/admin.password
```

Теперь необходимо выполнить следующие действия:

1. В разделе `Settings -> Security -> Users` нажмите `Create local user`.
2. Задайте поля `ID`, `Fist name`, `Last name`, `Email`, `Password`, `Confirm password`.
3. В поле `Status` установите значение `Active`.
4. В разделе `Roles` перетащите роль `nx-admin` в колонку `Granted`.
5. Нажмите `Create local user`.
6. Выберите пользователя `admin`.
7. В поле `Status` установите значение `Disabled`.
8. Нажмите `Save`.
9. Перезайдите в Nexus, но под своим аккаунтом. Учтите, что нужно использовать значение из поля `ID`, а не `Email` в
   качестве логина.
10. В разделе `Settings -> Security -> Roles` нажмите `Create role`.
11. В поле `Type` установите значение `Nexus role`.
12. В поле `Role ID` установите значение `nx-github`.
13. В поле `Role Name` установите значение `nx-github`.
14. Нажмите `Modify Applied Privileges`.
15. Найдите и поставьте галочку напротив: `nx-repository-admin-*-*-*` и `nx-repository-view-*-*-*`.
16. Нажмите `Confirm`, потом `Save`.
17. Нажмите `Create role`.
18. В поле `Type` установите значение `Nexus role`.
19. В поле `Role ID` установите значение `nx-prometheus`.
20. В поле `Role Name` установите значение `nx-prometheus`.
21. Нажмите `Modify Applied Privileges`.
22. Найдите и поставьте галочку напротив `nx-metrics-all`.
23. Нажмите `Confirm`, потом `Save`.
24. Нажмите `Create role`.
25. В поле `Type` установите значение `Nexus role`.
26. В поле `Role ID` установите значение `nx-k8s`.
27. В поле `Role Name` установите значение `nx-k8s`.
28. Нажмите `Modify Applied Privileges`.
29. Найдите и поставьте галочку напротив: `nx-repository-admin-docker-docker-github-browse`,
    `nx-repository-admin-docker-docker-github-read`, `nx-repository-admin-docker-docker-hub-browse`,
    `nx-repository-admin-docker-docker-hub-read`, `nx-repository-admin-docker-docker-public-browse`,
    `nx-repository-admin-docker-docker-public-read`, `nx-repository-admin-docker-docker-releases-browse`,
    `nx-repository-admin-docker-docker-releases-read`, `nx-repository-admin-docker-docker-snapshots-browse` и
    `nx-repository-admin-docker-docker-snapshots-read`.
30. Нажмите `Confirm`, потом `Save`.
31. В разделе `Settings -> Security -> Users` нажмите `Create local user`.
32. В поле `ID` установите значение `nx-github`.
33. В поле `First name` установите значение `Github`.
34. В поле `Last name` установите значение `Workflow`.
35. В поле `Email` установите значение `example@gmail.com`.
36. Задайте поля `Password` и `Confirm password`, а также запомните пароль — он понадобится для CI/CD.
37. В поле `Status` установите значение `Active`.
38. В разделе `Roles` перетащите роль `nx-github` в колонку `Granted`.
39. Нажмите `Save`.
40. Нажмите `Create local user`.
41. В поле `ID` установите значение `nx-prometheus`.
42. В поле `First name` установите значение `Nexus`.
43. В поле `Last name` установите значение `Prometheus`.
44. В поле `Email` установите значение `example@gmail.com`.
45. Задайте поля `Password` и `Confirm password`, а также запомните пароль — он понадобится для мониторинга Nexus.
46. В поле `Status` установите значение `Active`.
47. В разделе `Roles` перетащите роль `nx-prometheus` в колонку `Granted`.
48. Нажмите `Save`.
49. Нажмите `Create local user`.
50. В поле `ID` установите значение `nx-k8s`.
51. В поле `First name` установите значение `K8s`.
52. В поле `Last name` установите значение `Cluster`.
53. В поле `Email` установите значение `example@gmail.com`.
54. Задайте поля `Password` и `Confirm password`, а также запомните пароль — он понадобится для K8s.
55. В поле `Status` установите значение `Active`.
56. В разделе `Roles` перетащите роль `nx-k8s` в колонку `Granted`.
57. Нажмите `Save`.
58. В разделе `Realms` нажмите на `Docker Bearer Token Realm`, чтобы активировать.
59. В разделе `Settings -> Repository -> Repositories` нажмите `nuget-group`.
60. Нажмите `Delete repository`, потом `Yes`.
61. Нажмите `nuget-hosted`, потом `Delete repository`, а после `Yes`.
62. Нажмите `nuget-org-proxy`, потом `Delete repository`, а после `Yes`.
63. Нажмите `Create repository`.
64. Выберите `docker (proxy)`.
65. В поле `Name` укажите `docker-github`.
66. В поле `Enable Docker V1 API` поставьте галочку.
67. В поле `Remote storage` укажите `https://ghcr.io`.
68. В поле `Use the Nexus Repository truststore` поставьте галочку.
69. Нажмите `View certificate`.
70. Нажмите `Add certificate to truststore`.
71. Нажмите `Create repository`.
72. Нажмите `Create repository`.
73. Выберите `docker (proxy)`.
74. В поле `Name` укажите `docker-hub`.
75. В поле `Enable Docker V1 API` поставьте галочку.
76. В поле `Remote storage` укажите `https://registry-1.docker.io`.
77. В поле `Use the Nexus Repository truststore` поставьте галочку.
78. Нажмите `View certificate`.
79. Нажмите `Add certificate to truststore`.
80. Нажмите `Create repository`.
81. Нажмите `Create repository`.
82. Выберите `docker (hosted)`.
83. В поле `Name` укажите `docker-releases`.
84. В поле `Enable Docker V1 API` поставьте галочку.
85. В поле `Deployment Policy` выберите `Disable redeploy`.
86. В поле `Proprietary Components` поставьте галочку.
87. В поле `Allow redeploy only on 'latest' tag` поставьте галочку.
88. Нажмите `Create repository`.
89. Нажмите `Create repository`.
90. Выберите `docker (hosted)`.
91. В поле `Name` укажите `docker-snapshots`.
92. В поле `Enable Docker V1 API` поставьте галочку.
93. В поле `Deployment Policy` выберите `Allow redeploy`.
94. В поле `Proprietary Components` поставьте галочку.
95. Нажмите `Create repository`.
96. Нажмите `Create repository`.
97. Выберите `docker (group)`.
98. В поле `Name` укажите `docker-public`.
99. В поле `Enable Docker V1 API` поставьте галочку.
100. В поле `Member repositories` перетащите все репозитории в правую колонку и расположите их в следующем порядке:
     `docker-snapshots`, `docker-releases`, `docker-hub`, `docker-github`.
101. В поле `Proprietary Components` поставьте галочку.
102. Нажмите `Create repository`.

Далее необходимо сохранить пароль для мониторинга Nexus. Для этого выполните следующие команды:

```shell
NX_PROMETHEUS_PASSWORD= # Укажите здесь пароль, заданный на 45 шаге при настройке Nexus
kubectl create secret generic nexus-prometheus-user -n nexus \
        --type='kubernetes.io/basic-auth' \
        --from-literal=username='nx-prometheus' \
        --from-literal=password=${NX_PROMETHEUS_PASSWORD}
```

### Создание токенов для Docker образов в Nexus

Для того чтобы создать токены и сохранить их в K8s, выполните следующие команды:

```shell
NX_K8S_PASSWORD= # Укажите здесь пароль, заданный на 54 шаге при настройке Nexus

# https://nexus.sogeor.com/repository/docker-github/
kubectl create secret docker-registry nexus-docker-github -n misc --docker-server=https://nexus.sogeor.com/repository/docker-github/ --docker-username=nx-k8s --docker-password={NX_K8S_PASSWORD} --docker-email=example@gmail.com
kubectl annotate secret nexus-docker-github -n misc \
        "reflector.v1.k8s.emberstack.com/reflection-allowed=true" \
        "reflector.v1.k8s.emberstack.com/reflection-auto-enabled=true" \
        "reflector.v1.k8s.emberstack.com/reflection-auto-namespaces=api,nexus,service,sso"

# https://nexus.sogeor.com/repository/docker-hub/
kubectl create secret docker-registry nexus-docker-hub -n misc --docker-server=https://nexus.sogeor.com/repository/docker-hub/ --docker-username=nx-k8s --docker-password={NX_K8S_PASSWORD} --docker-email=example@gmail.com
kubectl annotate secret nexus-docker-hub -n misc \
        "reflector.v1.k8s.emberstack.com/reflection-allowed=true" \
        "reflector.v1.k8s.emberstack.com/reflection-auto-enabled=true" \
        "reflector.v1.k8s.emberstack.com/reflection-auto-namespaces=api,nexus,service,sso"

# https://nexus.sogeor.com/repository/docker-public/
kubectl create secret docker-registry nexus-docker-public -n misc --docker-server=https://nexus.sogeor.com/repository/docker-public/ --docker-username=nx-k8s --docker-password={NX_K8S_PASSWORD} --docker-email=example@gmail.com
kubectl annotate secret nexus-docker-public -n misc \
        "reflector.v1.k8s.emberstack.com/reflection-allowed=true" \
        "reflector.v1.k8s.emberstack.com/reflection-auto-enabled=true" \
        "reflector.v1.k8s.emberstack.com/reflection-auto-namespaces=api,nexus,service,sso"

# https://nexus.sogeor.com/repository/docker-releases/
kubectl create secret docker-registry nexus-docker-releases -n misc --docker-server=https://nexus.sogeor.com/repository/docker-releases/ --docker-username=nx-k8s --docker-password={NX_K8S_PASSWORD} --docker-email=example@gmail.com
kubectl annotate secret nexus-docker-releases -n misc \
        "reflector.v1.k8s.emberstack.com/reflection-allowed=true" \
        "reflector.v1.k8s.emberstack.com/reflection-auto-enabled=true" \
        "reflector.v1.k8s.emberstack.com/reflection-auto-namespaces=api,nexus,service,sso"

# https://nexus.sogeor.com/repository/docker-snapshots/
kubectl create secret docker-registry nexus-docker-snapshots -n misc --docker-server=https://nexus.sogeor.com/repository/docker-snapshots/ --docker-username=nx-k8s --docker-password={NX_K8S_PASSWORD} --docker-email=example@gmail.com
kubectl annotate secret nexus-docker-snapshots -n misc \
        "reflector.v1.k8s.emberstack.com/reflection-allowed=true" \
        "reflector.v1.k8s.emberstack.com/reflection-auto-enabled=true" \
        "reflector.v1.k8s.emberstack.com/reflection-auto-namespaces=api,nexus,service,sso"
```
