# K3S 安装

## 系统检查

 ```shell
# 修改主机名称
hostnamectl set-hostname k3s-master
# 修改hosts文件
echo "10.4.7.21 k3s-master" >> /etc/hosts
# 查看ufw状态
ufw status
# 临时关闭swap
swapoff -a
# 永久关闭swap
vi /etc/fstab
# /swap.img     none    swap    sw      0       0
free -m
# 同时调整k8s的swappiness参数
echo "vm.swappiness=0" >> /etc/sysctl.d/k3s.conf
sysctl -p /etc/sysctl.d/k3s.conf
 ```

## 脚本安装

```shell
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn INSTALL_K3S_EXEC="--disable=traefik" sh -
```



## Ingress-traefik

https://github.com/containous/traefik

### crd

```shell
#  traefik-crd.yaml 
## IngressRoute
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutes.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRoute
    plural: ingressroutes
    singular: ingressroute
---
## IngressRouteTCP
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutetcps.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteTCP
    plural: ingressroutetcps
    singular: ingressroutetcp
---
## Middleware
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: middlewares.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: Middleware
    plural: middlewares
    singular: middleware
---
## TLSOption
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsoptions.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSOption
    plural: tlsoptions
    singular: tlsoption
---
## TraefikService
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: traefikservices.traefik.containo.us
spec:
  scope: Namespaced
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TraefikService
    plural: traefikservices
    singular: traefikservice
```

### rbac

```shell
# traefik-rbac.yaml 
## ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: kube-system
  name: traefik-ingress-controller
---
## ClusterRole
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups: [""]
    resources: ["services","endpoints","secrets"]
    verbs: ["get","list","watch"]
  - apiGroups: ["extensions"]
    resources: ["ingresses"]
    verbs: ["get","list","watch"]
  - apiGroups: ["extensions"]
    resources: ["ingresses/status"]
    verbs: ["update"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["middlewares"]
    verbs: ["get","list","watch"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["ingressroutes"]
    verbs: ["get","list","watch"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["ingressroutetcps"]
    verbs: ["get","list","watch"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["tlsoptions"]
    verbs: ["get","list","watch"]
  - apiGroups: ["traefik.containo.us"]
    resources: ["traefikservices"]
    verbs: ["get","list","watch"]
---
## ClusterRoleBinding
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: kube-system
```

### configMap

```shell
# traefik-config.yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: traefik-config
data:
  traefik.yaml: |-
    ping: ""                    ## 启用 Ping
    serversTransport:
      insecureSkipVerify: true  ## Traefik 忽略验证代理服务的 TLS 证书
    api:
      insecure: true            ## 允许 HTTP 方式访问 API
      dashboard: true           ## 启用 Dashboard
      debug: false              ## 启用 Debug 调试模式
    metrics:
      prometheus: ""            ## 配置 Prometheus 监控指标数据，并使用默认配置
    entryPoints:
      web:
        address: ":80"          ## 配置 80 端口，并设置入口名称为 web
      websecure:
        address: ":443"         ## 配置 443 端口，并设置入口名称为 websecure
    providers:
      kubernetesCRD: ""         ## 启用 Kubernetes CRD 方式来配置路由规则
      kubernetesIngress: ""     ## 启动 Kubernetes Ingress 方式来配置路由规则
    log:
      filePath: ""              ## 设置调试日志文件存储路径，如果为空则输出到控制台
      level: error              ## 设置调试日志级别
      format: json              ## 设置调试日志格式
    accessLog:
      filePath: ""              ## 设置访问日志文件存储路径，如果为空则输出到控制台
      format: json              ## 设置访问调试日志格式
      bufferingSize: 0          ## 设置访问日志缓存行数
      filters:
        #statusCodes: ["200"]   ## 设置只保留指定状态码范围内的访问日志
        retryAttempts: true     ## 设置代理访问重试失败时，保留访问日志
        minDuration: 20         ## 设置保留请求时间超过指定持续时间的访问日志
      fields:                   ## 设置访问日志中的字段是否保留（keep 保留、drop 不保留）
        defaultMode: keep       ## 设置默认保留访问日志字段
        names:                  ## 针对访问日志特别字段特别配置保留模式
          ClientUsername: drop  
        headers:                ## 设置 Header 中字段是否保留
          defaultMode: keep     ## 设置默认保留 Header 中字段
          names:                ## 针对 Header 中特别字段特别配置保留模式
            User-Agent: redact
            Authorization: drop
            Content-Type: keep
```

### Set Lable

```shell
kubectl label nodes k3s-master IngressProxy=true
kubectl get nodes --show-labels
```



### ds资源清单

```shell
# traefik-deploy.yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik
spec:
  ports:
    - name: web
      port: 80
    - name: websecure
      port: 443
    - name: admin
      port: 8080
  selector:
    app: traefik
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: traefik-ingress-controller
  labels:
    app: traefik
spec:
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      name: traefik
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 1
      containers:
        - image: traefik:v2.1.2
          name: traefik-ingress-lb
          ports:
            - name: web
              containerPort: 80
              hostPort: 80         ## 将容器端口绑定所在服务器的 80 端口
            - name: websecure
              containerPort: 443
              hostPort: 443        ## 将容器端口绑定所在服务器的 443 端口
            - name: admin
              containerPort: 8080  ## Traefik Dashboard 端口
          resources:
            limits:
              cpu: 2000m
              memory: 1024Mi
            requests:
              cpu: 1000m
              memory: 1024Mi
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
          args:
            - --configfile=/config/traefik.yaml
          volumeMounts:
            - mountPath: "/config"
              name: "config"
      volumes:
        - name: config
          configMap:
            name: traefik-config 
      tolerations:              ## 设置容忍所有污点，防止节点被设置污点
        - operator: "Exists"
      nodeSelector:             ## 设置node筛选器，在特定label的节点上启动
        IngressProxy: "true"
```

### IngressRoute

```shell
# traefik-dashboard-route.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard-route
spec:
  entryPoints:
    - web
  routes:
  # 这里设置你的域名
    - match: Host(`traefik.li.com`)
      kind: Rule
      services:
        - name: traefik
          port: 8080
```

```shell
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=who.qikqiak.com"

kubectl create secret tls https --cert=tls.crt --key=tls.key -n kube-system

kubectl apply -f traefik-crd.yaml
kubectl apply -f traefik-rbac.yaml -n kube-system
kubectl apply -f traefik-config.yaml -n kube-system
kubectl apply -f traefik-deploy.yaml -n kube-system
kubectl apply -f traefik-dashboard-route.yaml -n kube-system
```



## kubernetes-dashboard

[kubernetes](https://github.com/kubernetes)/**[dashboard](https://github.com/kubernetes/dashboard)**

```shell
GITHUB_URL=https://github.com/kubernetes/dashboard/releases
VERSION_KUBE_DASHBOARD=$(curl -w '%{url_effective}' -I -L -s -S ${GITHUB_URL}/latest -o /dev/null | sed -e 's|.*/||')
sudo k3s kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml
```



### rbac.yaml

```shell
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

```shell
sudo k3s kubectl -n kube-system describe secret admin-user-token | grep '^token'
```

### ingress.yaml

```shell
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: kubernetes-dashboard-route
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  entryPoints:
    - websecure
  routes:
  # 这里设置你的域名
    - match: Host(`dashboard.li.com`)
      kind: Rule
      services:
        - name: kubernetes-dashboard
          port: 443
  tls:
    secretName: https
```



## [k8s 证书过期时间调整](https://www.cnblogs.com/lixinliang/p/12217328.html)

```shell
cd /etc/kubernetes/pki
kubeadm certs check-expiration

# 安装go环境
wget https://dl.google.com/go/go1.16.1.linux-amd64.tar.gz
tar -zxvf go1.16.1.linux-amd64.tar.gz -C /usr/local/
echo "export PATH=$PATH:/usr/local/go/bin" >> /etc/profile 
source /etc/profile
go version

# 下载kubernetes源码包
git clone https://gitee.com/mirrors/Kubernetes.git
kubeadm version
git checkout -b remotes/origin/release-1.19.3 v1.19.3

# 修改 Kubeadm 源码包更新证书策略
cd Kubernetes
vim ./staging/src/k8s.io/client-go/util/cert/cert.go
96         maxAge := time.Hour * 24 * 365 * 10         // one year self-signed certs

vim ./cmd/kubeadm/app/constants/constants.go
47         CertificateValidity = time.Hour * 24 * 365 * 10

# 只编译kubeadm
apt install make
make WHAT=cmd/kubeadm GOFLAGS=-v
cp _output/bin/kubeadm /root/kubeadm-new


#更新 kubeadm
#将 kubeadm 进行替换
cp /usr/bin/kubeadm /usr/bin/kubeadm.old
cp /root/kubeadm-new /usr/bin/kubeadm
chmod a+x /usr/bin/kubeadm


# 证书更新
cp -r /etc/kubernetes/pki /etc/kubernetes/pki.old
cd /etc/kubernetes/pki
kubeadm certs renew all # 有提示可忽略 
# 查看证书有限期 10年
kubeadm certs check-expiration


kubectl get cm -o yaml -n kube-system kubeadm-config > /root/cluster.yaml
cd /etc/kubernetes
mkdir conf.old
mv *.conf conf.old

kubeadm init phase kubeconfig all   # /root/cluster.yaml

root@k8s-master:/etc/kubernetes# ls -l
total 48
-rw------- 1 root root 5565 Oct 19 06:31 admin.conf
drwxr-xr-x 2 root root 4096 Oct 19 06:29 conf.old
-rw------- 1 root root 5601 Oct 19 06:31 controller-manager.conf
-rw------- 1 root root 5597 Oct 19 06:31 kubelet.conf
drwxr-xr-x 2 root root 4096 Oct 19 00:32 manifests
drwxr-xr-x 3 root root 4096 Oct 19 05:33 pki
drwxr-xr-x 3 root root 4096 Oct 19 06:25 pki.old
-rw------- 1 root root 5553 Oct 19 06:31 scheduler.conf

# 其他master 节点
scp -qpr master01:/usr/bin/kubeadm master02:/usr/bin/kubeadm 
# 然后 进行证书更新操作 和 集群配置文件生成操作

systemctl restart kubelet
kubectl get pod   -n kube-system
```

## Helm

### [安装](https://helm.sh/docs/intro/install/)

```shell
1. 脚本安装
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

2. 源安装
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

3. snap安装
sudo snap install helm --classic
```

### [Quickstart](https://v3.helm.sh/docs/intro/quickstart/)

```shell
helm repo add stable http://mirror.azure.cn/kubernetes/charts
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts 
helm repo update

# 删除aliyun库
helm repo remove aliyu

# 下载charts
helm pull stable/mysql

helm install stable/mysql --generate-name
helm ls
helm uninstall smiling-penguin
helm status smiling-penguin
```

### [Using_Helm](https://helm.sh/docs/intro/using_helm/)

```shell
# 从 Artifact Hub 中搜索所有的 wordpress charts
helm search hub wordpress

helm repo add brigade https://brigadecore.github.io/charts
helm search repo brigade
# 模糊字符串匹配算法
helm search repo kash

helm install happy-panda bitnami/wordpress
helm status happy-panda
# 安装前自定义 chart
helm show values bitnami/wordpress
echo '{mariadb.auth.database: user0db, mariadb.auth.username: user0}' > values.yaml
helm install -f values.yaml bitnami/wordpress --generate-name
```

### mysql-cluster

```shell
mkdir /data/mysql/pv{1,2,3}
chown -R 1001:1001 /data/mysql

helm install mysql-cluster bitnami/mysql
```

```shell
cat > mysql-pv.yaml<<EOF
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv1
  labels:
    type: local
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /data/mysql/pv1
    server: localhost
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv2
  labels:
    type: local
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /data/mysql/pv2
    server: localhost
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv3
  labels:
    type: local
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /data/mysql/pv3
    server: localhost
EOF
```













