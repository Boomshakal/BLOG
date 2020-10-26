# K8S kubeadm安装

![](https://dss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=175322010,70959622&fm=26&gp=0.jpg)

## 系统检查

 ```shell
# 修改主机名称
hostnamectl set-hostname k8s-master
# 修改hosts文件
echo "10.4.7.21 k8s-master" >> /etc/hosts
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
# 查看1.15的最新版本
apt-cache madison kubelet kubectl kubeadm |grep '1.15.12-00'
# 安装指定的版本
apt install -y kubelet=1.15.12-00 kubectl=1.15.12-00 kubeadm=1.15.12-00
# 查看kubelet版本
kubectl version --client=true -o yaml
# 配置kubelet禁用swap
echo 'KUBELET_EXTRA_ARGS="--fail-swap-on=false"' > /etc/default/kubelet
# 重启kubelet
systemctl daemon-reload && systemctl restart kubelet
# 查看kubelet服务启动状态(因缺少缺少很多参数配置文件,需要等待kubeadm init 后生成)
journalctl -u kubelet -f
```

## 初始化k8s

```shell
kubeadm init \
  --image-repository registry.aliyuncs.com/google_containers \
  --pod-network-cidr=10.244.0.0/16 \
  --ignore-preflight-errors=Swap


mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 如果The connection to the server localhost:8080 was refused - did you specify the right host or port?
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile

kubectl get pods --all-namespaces
```

## k8s网络flannel

https://github.com/coreos/flannel

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## kubernetes-dashboard

[kubernetes](https://github.com/kubernetes)/**[dashboard](https://github.com/kubernetes/dashboard)**

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.4/aio/deploy/recommended.yaml
```

```shell
# 修改为type:NodePort
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard
spec:
  type: NodePort
  ports:
	  ...
      nodePort: 30001
```

```shell
# 后台开启proxy模式
nohup kubectl proxy --address=10.4.7.21 --disable-filter=true &
```

### rbac.yaml

```shell
cat >rbac.yaml <<EOF
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
EOF
```

```shell
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

## 单节点k8s,默认pod不被调度在master节点

```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## Ingress-traefik

https://github.com/containous/traefik

### pull镜像

```shell
docker pull traefik:v1.7.2-alpine
```
### rbac

```shell
cat >rbac.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
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
EOF
```

### ds资源清单

```shell
cat >ds.yaml <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: traefik-ingress
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress
        name: traefik-ingress
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik:v1.7.2-alpine
        name: traefik-ingress
        ports:
        - name: controller
          containerPort: 80
          hostPort: 81
        - name: admin-web
          containerPort: 8080
        securityContext:
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
        - --insecureskipverify=true
        - --kubernetes.endpoint=https://10.4.7.21:6443
        - --accesslog
        - --accesslog.filepath=/var/log/traefik_access.log
        - --traefiklog
        - --traefiklog.filepath=/var/log/traefik.log
        - --metrics.prometheus
EOF
```

```shell
cat >svc.yaml <<EOF
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress
  ports:
    - protocol: TCP
      port: 80
      name: controller
    - protocol: TCP
      port: 8080
      name: admin-web
EOF
```

### ingress

```shell
cat >ingress.yaml <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: traefik.ikahe.com
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-ingress-service
          servicePort: 8080
EOF
```

```shell
kubectl apply -f rbac.yaml
kubectl apply -f ds.yaml
kubectl apply -f svc.yaml
kubectl apply -f ingress.yaml
```

## [k8s 证书过期时间调整](https://www.cnblogs.com/lixinliang/p/12217328.html)

```shell
cd /etc/kubernetes/pki
openssl x509 -in apiserver.crt -text -noout
	Validity
            Not Before: Oct 19 00:32:46 2020 GMT
            Not After : Sep 25 06:25:28 2120 GMT

# 安装go环境
wget https://dl.google.com/go/go1.15.1.linux-amd64.tar.gz
tar -zxvf go1.15.1.linux-amd64.tar.gz -C /usr/local/
echo "export PATH=$PATH:/usr/local/go/bin" >> /etc/profile 
source /etc/profile
go version

# 下载kubernetes源码包
git clone https://github.com/kubernetes/kubernetes.git
kubeadm version
git checkout -b remotes/origin/release-1.19.3 v1.19.3

# 修改 Kubeadm 源码包更新证书策略
cd kubernetes
vim cmd/kubeadm/app/util/pkiutil/pki_helpers.go
const duration36500d = time.Hour * 24 * 365 * 100
NotAfter: time.Now().Add(duration36500d).UTC()

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
kubeadm alpha certs renew all # 有提示可忽略 
# 查看证书有限期 100年
openssl x509 -in apiserver.crt -text -noout
Validity
            Not Before: Oct 19 00:32:46 2020 GMT
            Not After : Sep 25 06:25:28 2120 GMT


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

