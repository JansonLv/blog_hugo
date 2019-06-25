---
desc: 由 genpost (https://github.com/hidevopsio/genpost) 代码生成器生成
title: k8s安装
date: 2019-03-25T12:19:22+08:00
author: jansonlv
draft: false
tags:
- docker k8s 
---

## 镜像加速
https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors
![](media/15564543905177.jpg)

## 生成ssl工具
https://blog.51cto.com/liuzhengwei521/2120535?utm_source=oschina-app
https://www.cnblogs.com/fanqisoft/p/10765038.html

## docker安装
https://www.cnblogs.com/yufeng218/p/8370670.html

## centos安装
https://www.cnblogs.com/zhaojiedi1992/p/zhaojiedi_linux_043_hostname.html
centos7修改hostname
```
[root@centos7 ~]$ hostnamectl set-hostname centos77.magedu.com             # 使用这个命令会立即生效且重启也生效
[root@centos7 ~]$ hostname                                                 # 查看下
centos77.magedu.com
[root@centos7 ~]$ vim /etc/hosts                                           # 编辑下hosts文件， 给127.0.0.1添加hostname
[root@centos7 ~]$ cat /etc/hosts                                           # 检查
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 centos77.magedu.com
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```

## etcd配置 /opt/kubernetes/cfg
```
[member]
ETCD_NAME="etcd01" #节点名称，如果有多个节点，这里必须要改，etcd02，etcd03
ETCD_DATA_DIR="/var/lib/etcd/default.etcd" #数据目录
E
ETCD_LISTEN_PEER_URLS="https://10.211.55.101:2380" #集群沟通端口2380
ETCD_LISTEN_CLIENT_URLS="https://10.211.55.101:2379,http://127.0.0.1:2379" #客户端沟通端口2379

[cluster]
ETCD_INITIAL_ADVERTlSE_PEER_URLS="https://10.211.55.101:2380" #集群通告地址
ETCD_INITIAL_CLUSTER="etcd01=https://10.211.55.101:2380,etcd02=https://10.211.55.102:2380,etcd03=https://10.211.55.103:2380" #这个集群中所有节点，每个节点都要有
ETCD_INITIAL_CLUSTER_STATE="new" #新创建集群，existing表示加入已有集群
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster" #集群token
ETCD_ADVERTISE_CLIENT_URLS="https://10.211.55.101:2379" 客户端通告地址
```

### /usr/lib/systemd/system/etcd.service启动service配置
```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
 
[Service]
Type=notify
EnvironmentFile=-/opt/kubernetes/cfg/etcd
ExecStart=/opt/kubernetes/bin/etcd \
--name=${ETCD_NAME} \
--data-dir=${ETCD_DATA_DIR} \
--listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \
--listen-client-urls=${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
--advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS} \
--initial-advertise-peer-urls=${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
--initial-cluster=${ETCD_INITIAL_CLUSTER} \
--initial-cluster-token=${ETCD_INITIAL_CLUSTER_TOKEN} \
--initial-cluster-state=new \
--cert-file=/opt/kubernetes/ssl/server.pem \
--key-file=/opt/kubernetes/ssl/server-key.pem \
--peer-cert-file=/opt/kubernetes/ssl/server.pem \
--peer-key-file=/opt/kubernetes/ssl/server-key.pem \
--trusted-ca-file=/opt/kubernetes/ssl/ca-pem \
--peer-trusted-ca-file=/opt/kubernetes/ssl/ca.pem
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### 重新加载配置文件并启动
```
systemctl daemon-reload
systemctl enable etcd
systemctl restart etcd
```

### 节点测试
```
/opt/kubernetes/bin/etcdctl --ca-file=/opt/kubernetes/ssl/ca.pem --cert-file=/opt/kubernetes/ssl/server.pem --key-file=/opt/kubernetes/ssl/server-key.pem --endpoints="https://10.211.55.101:2379,https://10.211.55.102:2379,https://10.211.55.104:2379"  cluster-health
```

*有问题先ps出etcd的启动命令再结合tail -f /var/log/messages日志，查看是否有问题*


## flannel安装
配置service
```
file：/usr/lib/systemd/system/flanneld.service

[Unit]
Description=Flanneld overlay address etcd agent
After=network-online.target network.target
Before=docker.service

[Service]
Type=notify
EnvironmentFi1e=/opt/kubernetes/cfg/flaaneld
ExecStart=/opt/kubernetes/bin/flanneld —ip-masq $FLANNEL_OPTIONS
ExecStartPost=/opt/kubernetes/bin/mk-docker-opts.sh -k DOCKER_NERWORK_OPTIONS -d /run/flannel/subnet.env
Restart=on-failure

[Install]
wantedBy=multi-user.target
```

配置config
```
/opt/kubernetes/cfg/flannel

FLANNEL_OPTIONS="--etcd-endpoints=https://192.168.0.211:2379,https://192.168.0.212:2379,https://192.168.0.213:2379 -etcd-cafile=/opt/kubernetes/ssl/ca.pem -etcd-certfile=/opt/kubernetes/ssl/server.pem -etcd-keyfile=/opt/kubernetes/ssl/server-key.pem"
```

设置网段信息
```
/opt/kubernetes/bin/etcdctl --ca-file=ca.pem --cert-file=server.pem --key-file=server-key.pem --endpoints="https://10.211.55.101:2379,https://10.211.55.102:2379,https://10.211.55.103:2379" set /coreos.com/network/config '{ "Network": "172.17.0.0/16", "Backend": {"Type": "vxlan"}}'
```
启动flannel

验证
```
/opt/kubernetes/bin/etcdctl --ca-file=/opt/kubernetes/ssl/ca.pem --cert-file=/opt/kubernetes/ssl/server.pem --key-file=/opt/kubernetes/ssl/server-key.pem --endpoints="https://10.211.55.101:2379,https://10.211.55.102:2379,https://10.211.55.103:2379" get /coreos.com/network/config
{ "Network": "172.17.0.0/16", "Backend": {"Type": "vxlan"}}

ifconfig
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 172.17.80.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::80b0:eaff:fe9d:a545  prefixlen 64  scopeid 0x20<link>
        ether 82:b0:ea:9d:a5:45  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 8 overruns 0  carrier 0  collisions 0
        
[root@master ssl]# cat /run/flannel/subnet.env
DOCKER_OPT_BIP="--bip=172.17.80.1/24"
DOCKER_OPT_IPMASQ="--ip-masq=true"
DOCKER_OPT_MTU="--mtu=1450"
DOCKER_NERWORK_OPTIONS=" --bip=172.17.80.1/24 --ip-masq=true --mtu=1450"
```

### 配置docker网络连接flannel
```
# file：/usr/lib/systemd/system/docker.service

[Unit]
Description=Docker Application container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
wants=network-online.target

[Service]
Type=notify
# 修改下面两行，其他不需要修改
EnvironmentFi1e=/run/flannel/subnet.env
ExecStart=/usr/bin/dockerd $DOCKER_NERWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
startLimitInterval=60s

[Install]
wantedBy=multi-user.target
```
重启docker
```
[root@master ssl]# systemctl daemon-reload
[root@master ssl]# systemctl restart docker
[root@master ssl]# ifconfig
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:49:d2:29:8f  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

### 配置k8s
在master节点生成
```
export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
[root@master k8s]# cat > token.csv <<EOF
> ${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
> EOF


#---------------------

# 创建kubelet bootstrapping kubeconfig
# 设置kube api访问入口
export KUBE_APISERVER="https//10.211.55.101:6443"

#设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=./ca.pem \
  --embed-certs=true \
  --server=${KUBE_APlSERVER} \
  --kubeconfig=bootstrap.kubeconfig

#设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig

  #设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig


kubectl config set-cluster kubernetes \
  --certificate-authority=./ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
  --client-certificate=./kube-proxy.pem \
  --client-key=./kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

## master节点组件
### apiserver配置
```
MASTER_ADDRESS=${1:-"192.168.1.195"}
ETCD_SERVERS={2:-"http://127.0.0.1:2379"}

cat <<EOF >/opt/kubernetes/cfg/kube-apiserver

KUBE_APISERVER_OPTS="--logtostderr=true \\
--v=4 \\
--etcd-servers=${ETCD_SERVERS} \\
--insecure-bind-address=127.0.0.1 \\
--bind-address={MASTER_ADDRESS} \\
--insecure-port=8080 \\
--secure-port=6443 \\
--advertise-address=${MASTER_ADDRESS} \\
--allow-privileged=true \\
--service-cluster-ip-range=10.10.10.0/24 \\
--admission-control=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota,NodeRestriction \\
--authorization-mode=RBAC,Node \\
--kubelet-https=true \\
--enable-bootstrap-token-auth \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
--service-node-port-range=30000-50000 \\
--tls-cert-file=/opt/kubernetes/ssl/server.pem \\
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--etcd-cafile=/opt/kubernetes/ssl/ca.pem \\
--etcd-certfile=/opt/kubernetes/ssl/server.pem \\
--etcd-keyfile=/opt/kubernetes/ssl/server-key.pem"
EOF

cat << EOF >/usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/kube-apiserver
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

### kube-controller-manager配置
```
MASTER_ADDRESS=${1:-"127.0.0.1"}
cat << EOF >/opt/kubernetes/cfg/kube-controller-manager

KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=true \\
--v=4 \\
--master=${MASTER_ADDRESS}:8080 \\
--leader-elect=true \\
--address=127.0.0.1 \\
--service-cluster-ip-range=10.10.10.0/24 \\
--cluster-name=kubernetes \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem"
EOF

cat << EOF >/usr/lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/kube-controller-manager
ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

### kube-scheduler配置
```shell
MASTER_ADDRESS=${1:-"127.0.0.1"}
cat << EOF >/opt/kubernetes/cfg/kube-scheduler

KUBE_SCHEDULER_OPTS="--logtostderr=true \\
--v=4 \\
--master=${MASTER_ADDRESS}:8080 \\
--leader-elect"

cat << EOF >/usr/lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/kube-scheduler
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

## node节点配置
kubelet 配置
```shell
NODE_ADDRESS=${1:-"192.168.1.196"}
DNS_SERVER_IP=${2:-"10.10.10.2"}

cat <<EOF >/opt/kubernetes/cfg/kubelet

KUBELET_OPTS="--logtostderr=true \\
--v=4 \\
--address=${NODE_ADDRESS} \\
--hostname-override=${NODE_ADDRESS} \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--experimental-bootstrap-kubeconfig=/opt/kubernetes/ssl/bootstrap.kubeconfig \\
--cert-dir=/opt/kubernetes/ssl \\
--allow-privileqed=true \\
--cluster-dns=${DNS_SERVER_iP} \\
--cluster-domain=cluster.local \\
--fail-swap-on=false \\
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0"
EOF

cat << EOF >/usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]  
EnvironmentFile=-/opt/kubernetes/cfg/kubelet
ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

###kube-proxy配置
```shell
NODE_ADDRESS=${1:-`ip addr | grep 'state UP' -A2 | tail -n1 | awk '{print $2}' | cut -f1 -d '/'`}

cat << EOF >/opt/kubernetes/cfg/kube-proxy

KUBE_PROXY_OPTS="--logtostderr=true \
--v=4 \
--hostname-override=${NODE_ADDRESS} \
--kubeconfig=/opt/kubernetes/ssl/kube-proxy.kubeconfig"

EOF

cat << EOF >/usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/kube-proxy
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```
### 测试
[Kubernetes角色访问控制RBAC](https://www.jianshu.com/p/9991f189495f)
```
// 创建权限
[root@master ssl]# kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
clusterrolebinding "kubelet-bootstrap" created

[root@master ssl]# kubectl get node
No resources found.
[root@master ssl]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-HNPo0dTn31gru7oye4PKEgKpKTsAiB1UCg21JsbtmjY   17m       kubelet-bootstrap   Pending

// 开放权限
[root@master ssl]# kubectl certificate approve node-csr-HNPo0dTn31gru7oye4PKEgKpKTsAiB1UCg21JsbtmjY
certificatesigningrequest

"node-csr-HNPo0dTn31gru7oye4PKEgKpKTsAiB1UCg21JsbtmjY" approved
```

# 节点测试
```
[root@master ssl]# kubectl run nginx --replicas=2 --image=nginx
deployment "nginx" created
[root@master ssl]# kubectl get pod
NAME                   READY     STATUS              RESTARTS   AGE
nginx-8586cf59-78jwk   0/1       ContainerCreating   0          21s
nginx-8586cf59-zn4tz   0/1       ContainerCreating   0          21s
[root@master ssl]# kubectl get all
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/nginx   2         2         2            0           33s

NAME                DESIRED   CURRENT   READY     AGE
rs/nginx-8586cf59   2         2         0         33s

NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/nginx   2         2         2            0           33s

NAME                DESIRED   CURRENT   READY     AGE
rs/nginx-8586cf59   2         2         0         33s

NAME                      READY     STATUS              RESTARTS   AGE
po/nginx-8586cf59-78jwk   0/1       ContainerCreating   0          33s
po/nginx-8586cf59-zn4tz   0/1       ContainerCreating   0          33s

NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
svc/kubernetes   ClusterIP   10.10.10.1   <none>        443/TCP   12h

[root@master ssl]# kubectl get pod -o wide
NAME                   READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-8586cf59-78jwk   1/1       Running   0          5m        172.17.0.2   10.211.55.102
nginx-8586cf59-zn4tz   1/1       Running   0          5m        172.17.0.3   10.211.55.102

[root@master ssl]# kubectl expose deployment nginx --port=88 --target-port=80 --type=NodePort 
service "nginx" exposed

[root@master ssl]# kubectl get pod -o wide
NAME                   READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-8586cf59-78jwk   1/1       Running   0          9m        172.17.0.2   10.211.55.102
nginx-8586cf59-zn4tz   1/1       Running   0          9m        172.17.0.3   10.211.55.102
[root@master ssl]# kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.10.10.1    <none>        443/TCP        12h
nginx        NodePort    10.10.10.22   <none>        88:48099/TCP   1m
[root@master ssl]# kubectl get pod
NAME                   READY     STATUS    RESTARTS   AGE
nginx-8586cf59-78jwk   1/1       Running   0          11m
nginx-8586cf59-zn4tz   1/1       Running   0          11m
[root@master ssl]#  kubectl logs nginx-8586cf59-78jwk
[root@master ssl]#  kubectl logs nginx-8586cf59-zn4tz
10.211.55.102 - - [30/Apr/2019:16:25:26 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
172.17.0.1 - - [30/Apr/2019:16:26:32 +0000] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.131 Safari/537.36" "-"
2019/04/30 16:26:34 [error] 5#5: *2 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 172.17.0.1, server: localhost, request: "GET /favicon.ico HTTP/1.1", host: "10.211.55.102:48099", referrer: "http://10.211.55.102:48099/"
172.17.0.1 - - [30/Apr/2019:16:26:34 +0000] "GET /favicon.ico HTTP/1.1" 404 556 "http://10.211.55.102:48099/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.131 Safari/537.36" "-"
```


## 部署问题
```
● kubelet.service - Kubernetes kubelet
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; disabled; vendor preset: disabled)
   Active: failed (Result: start-limit) since 五 2019-05-03 23:23:58 CST; 1s ago
     Docs: https://github.com/kubernetes/kubernetes
  Process: 5577 ExecStart=/opt/kubernetes/bin/kubelet $KUBELET_OPTS (code=exited, status=1/FAILURE)
 Main PID: 5577 (code=exited, status=1/FAILURE)

5月 03 23:23:58 node03 kubelet[5577]: I0503 23:23:58.137747    5577 plugins.go:101] No cloud provider specified.
5月 03 23:23:58 node03 kubelet[5577]: I0503 23:23:58.137757    5577 server.go:303] No cloud provider specified: "" from the config file: ""
5月 03 23:23:58 node03 kubelet[5577]: I0503 23:23:58.137776    5577 bootstrap.go:58] Using bootstrap kubeconfig to generate TLS client ce...nfig file
5月 03 23:23:58 node03 kubelet[5577]: error: failed to run Kubelet: unable to create certificates signing request client: host must be a ...101:6443"
5月 03 23:23:58 node03 systemd[1]: kubelet.service holdoff time over, scheduling restart.
5月 03 23:23:58 node03 systemd[1]: Stopped Kubernetes kubelet.
5月 03 23:23:58 node03 systemd[1]: start request repeated too quickly for kubelet.service
5月 03 23:23:58 node03 systemd[1]: Failed to start Kubernetes kubelet.
5月 03 23:23:58 node03 systemd[1]: Unit kubelet.service entered failed state.
5月 03 23:23:58 node03 systemd[1]: kubelet.service failed.

解决方案：
 error: failed to run Kubelet: unable to create certificates signing request client: host must be a ...101:6443"
证书有问题，重新生成
```

### etcd无法构建集群
```
1、etcd tls: bad certificate
2、 
```
定义server-csr.json时，节点ip是否包含


## 构建dashboard ui
参考
https://www.cnblogs.com/guigujun/p/8366546.html
https://www.cnblogs.com/ericnie/p/6826908.html
image镜像使用:registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1



## 编排
### delopy
```yaml
# api版本 kubectl api-versions命令获取
apiVersion: apps/v1beta2
# 资源类型 kubectl create -h
kind: Deployment
# 元数据
metadata:
  name: nginx-deployment
  namespace: default
  labels: 
    web: nginx
# k8s delopy信息
spec:
  # 副本数量
  replicas: 3 
  # 选择器
  selector:
    # 匹配pod标签
    matchLabels:
      app: nginx
  # 创建pod的信息
  template:
    metadata:
      labels:
        app: nginx
    spec:
      # 容器信息
      containers:
      - name: nginx
        image: nginx:1.10 
        ports:
          - containerPort: 80
```

### service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx 
spec:
  ports:
  # 集群的88端口
  - port: 88
    # 容器端口
    targetPort: 80
  selector:
    app: nginx
```