# K8S 安装

## 系统检查

 ```shell
# 修改主机名称
hostnamectl set-hostname k8s-master
# 修改hosts文件
echo "10.4.7.21 k8s-master1" >> /etc/hosts
echo "10.4.7.22 k8s-master2" >> /etc/hosts
echo "10.4.7.23 k8s-master3" >> /etc/hosts
# 查看ufw状态
ufw status
# 临时关闭swap
swapoff -a
# 永久关闭swap
vi /etc/fstab
# /swap.img     none    swap    sw      0       0
free -m
# 同时调整k8s的swappiness参数
echo "vm.swappiness=0" >> /etc/sysctl.d/k8s.conf
sysctl -p /etc/sysctl.d/k8s.conf

# Kubectl自动补全
source <(kubectl completion bash)
 ```



## Docker-CE

```shell
# 国内镜像
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
# 查看dockers版本
docker --version
# 给普通用户添加权限
sudo usermod -aG docker $USER
# docker阿里云加速器
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s https://b33dfgq9.mirror.aliyuncs.com
# 修改进程隔离工具
# docker默认cgroupfs  k8s使用systemd
vim /etc/docker/daemon.json
{
...
	"exec-opts": [ "native.cgroupdriver=systemd" ]
}
# 重启docker
systemctl daemon-reload && systemctl restart docker
```

## kubeadm

```shell
# 安装https
apt-get update && apt-get install -y apt-transport-https
# 添加密钥
curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
# 添加源
add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main"
# 更新源
apt-get update
# 查看最新版本
apt-cache madison kubelet kubectl kubeadm
# 安装指定的版本
apt install -y kubelet kubectl kubeadm
# 查看kubelet版本
kubectl version --client=true -o yaml
# 配置kubelet禁用swap
echo 'KUBELET_EXTRA_ARGS="--fail-swap-on=false"' > /etc/default/kubelet
# 重启kubelet
systemctl daemon-reload && systemctl restart kubelet
# 查看kubelet服务启动状态(因缺少缺少很多参数配置文件,需要等待kubeadm init 后生成)
journalctl -u kubelet -f
```

## 初始化k8s-m1

```shell
cat > kubeadm-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.22.1
imageRepository: registry.aliyuncs.com/google_containers
controlPlaneEndpoint: "10.4.7.21:6443"
apiServer:
  certSANs:    #填写所有kube-apiserver节点的hostname、IP、VIP
  - k8s-m1
  - k8s-m2
  - k8s-m3
  - k8s-w1
  - k8s-w2
  - k8s-w3
  - apiserver.lb
  - 10.4.7.20
networking:
  serviceSubnet: "10.96.0.0/16"
  podSubnet: "10.244.0.0/16"
  dnsDomain: "cluster.local"
EOF

# kubeadm init
# 根据您服务器网速的情况，您需要等候 3 - 10 分钟
kubeadm init --config=kubeadm-config.yaml --upload-certs

# docker pull coredns
docker pull registry.aliyuncs.com/google_containers/coredns:1.8.4
docker tag registry.aliyuncs.com/google_containers/coredns:1.8.4 registry.aliyuncs.com/google_containers/coredns:v1.8.4

# 配置 kubectl
rm -rf /root/.kube/
mkdir /root/.kube/
cp -i /etc/kubernetes/admin.conf /root/.kube/config

# 安装 flannel 网络插件
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# master 去除污点
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### k8s-m2,k8s-m3

```shell
kubeadm join 10.4.7.21:6443 --token swcdwc.etn06vu7t11r1ywq \
	--discovery-token-ca-cert-hash sha256:0df815a693038402057acb22f2698375d563a03fd16e1050933d25210bda6f5b \
	--control-plane --certificate-key d192afd75926b20289a6a489272e161c4f4962c97de4d09dd76cc5d89463ba3c
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
kubectl label nodes k8s-master IngressProxy=true
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
            - --kubernetes.endpoint=https://10.4.7.11:7443
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
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=cert.k8s.com"

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
sudo k8s kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml
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
kubectl -n kube-system describe secret admin-user-token | grep '^token'
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

## 集群访问

```shell
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile
```



## 测试集群

```shell
kubectl create  deployment  nginx-app   --image=nginx   --replicas=3

kubectl get pod -o wide
kubectl get deployment
kubectl expose deployment nginx-app --port=80 --type=NodePort
kubectl get svc
curl 10.4.7.21:32738
```



## Nginx 负载均衡

### 四层代理

```shell
apt install nginx
mkdir /etc/nginx/tcp.d/
echo 'include /etc/nginx/tcp.d/*.conf;' >>/etc/nginx/nginx.conf
cat >/etc/nginx/tcp.d/apiserver.conf <<EOF
stream {
    upstream kube-apiserver {
        server 10.4.7.21:6443     max_fails=3 fail_timeout=30s;
        server 10.4.7.22:6443     max_fails=3 fail_timeout=30s;
        server 10.4.7.23:6443     max_fails=3 fail_timeout=30s;
    }
    server {
        listen 7443;
        proxy_connect_timeout 2s;
        proxy_timeout 900s;
        proxy_pass kube-apiserver;
    }
}
EOF
```
### 七层代理

```
cat >/etc/nginx/conf.d/k8s.com.conf <<EOF
upstream default_backend_traefik {
    server 10.4.7.21:80    max_fails=3 fail_timeout=10s;
    server 10.4.7.22:80    max_fails=3 fail_timeout=10s;
    server 10.4.7.23:80    max_fails=3 fail_timeout=10s;
}
server {
    server_name *.k8s.com;
  
    location / {
        proxy_pass http://default_backend_traefik;
        proxy_set_header Host       $http_host;
        proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    }
}
EOF

nginx -t
nginx -s reload

# Master 同traefik 自签tls
cp ~/tls.* .
# Master2,3
scp root@10.4.7.21:/root/tls.* .

cat >/etc/nginx/conf.d/dashboard.k8s.com.conf <<'EOF'
server {
    listen       80;
    server_name  dashboard.k8s.com;

    rewrite ^(.*)$ https://${server_name}$1 permanent;
}
server {
    listen       443 ssl;
    server_name  dashboard.k8s.com;

    ssl_certificate     "certs/tls.crt";
    ssl_certificate_key "certs/tls.key";
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://default_backend_traefik;
        proxy_set_header Host       $http_host;
        proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    }
}
EOF
```



## Keepalived 高可用

### 创建端口监测脚本

```shell
cat >/etc/keepalived/check_port.sh <<'EOF'
#!/bin/bash
#keepalived 监控端口脚本
#使用方法：等待keepalived传入端口参数,检查改端口是否存在并返回结果
CHK_PORT=$1
if [ -n "$CHK_PORT" ];then
        PORT_PROCESS=`ss -lnt|grep $CHK_PORT|wc -l`
        if [ $PORT_PROCESS -eq 0 ];then
                echo "Port $CHK_PORT Is Not Used,End."
                exit 1
        fi
else
        echo "Check Port Cant Be Empty!"
fi
EOF

chmod +x /etc/keepalived/check_port.sh
```

```shell
# keepalived主
cat >/etc/keepalived/keepalived.conf <<'EOF'
! Configuration File for keepalived
global_defs {
   router_id 10.4.7.21
}
vrrp_script chk_nginx {
    script "/etc/keepalived/check_port.sh 7443"
    interval 2
    weight -20
}
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 251
    priority 100
    advert_int 1
    mcast_src_ip 10.4.7.21
    nopreempt

    authentication {
        auth_type PASS
        auth_pass 11111111
    }
    track_script {
         chk_nginx
    }
    virtual_ipaddress {
        10.4.7.20
    }
}
EOF
```

```shell
# keepalived从
cat >/etc/keepalived/keepalived.conf <<'EOF'
! Configuration File for keepalived
global_defs {
    router_id 10.4.7.22
}
vrrp_script chk_nginx {
    script "/etc/keepalived/check_port.sh 7443"
    interval 2
    weight -20
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 251
    mcast_src_ip 10.4.7.22
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 11111111
    }
    track_script {
        chk_nginx
    }
    virtual_ipaddress {
        10.4.7.20
    }
}
EOF

# 10.4.7.23 同 10.4.7.22 
```

```shell
systemctl start  keepalived
systemctl enable keepalived
ip addr|grep '10.4.7.20'
```