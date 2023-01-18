#### 环境准备

- 3台机器

  | 角色     | 主机名  | IP地址         | 组件                              |
  | :------- | ------- | :------------- | :-------------------------------- |
  | master01 | node101 | 192.168.56.101 | docker，kubectl，kubeadm，kubelet |
  | node01   | node102 | 192.168.56.102 | docker，kubectl，kubeadm，kubelet |
  | node02   | node103 | 192.168.56.103 | docker，kubectl，kubeadm，kubelet |

  机器要求至少2G内存

  

- 查看操作系统版本

  ```shell
  cat /etc/redhat-release 
  CentOS Linux release 7.7.1908 (Core)
  ```

- 修改主机名

  ```shell
  hostnamectl set-hostname node101
  hostnamectl set-hostname node102
  hostnamectl set-hostname node103
  ```

- 主机名解析

  编辑三台机器的`/etc/hosts`文件，添加下面内容

  ```shell
  192.168.56.101 node101
  192.168.56.102 node102
  192.168.56.103 node103
  ```

- 时间同步

  ```shell
  systemctl start chronyd
  systemctl enable chronyd
  date
  ```

- 禁用iptable和firewalld服务

  ```shell
  systemctl stop firewalld
  systemctl disable firewalld
  
  systemctl stop iptables
  systemctl disable iptables
  ```

- 禁用selinux

  ```shell
  # selinux是linux系统下的一个安全服务，如果不关闭它，在安装集群中会产生各种各样的奇葩问题
  # 编辑 /etc/selinux/config 文件，修改SELINUX的值为disable
  # 注意修改完毕之后需要重启linux服务
  vi /etc/selinux/config
  SELINUX=disabled
  ```

  ```shell
  # 或执行下面命令
  sudo setenforce 0
  sudo sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
  ```

- 禁用swap 分区

  ```shell
  # swap分区指的是虚拟内存分区，它的作用是物理内存使用完，之后将磁盘空间虚拟成内存来使用，启用swap设备会对系统的性能产生非常负面的影响，因此kubernetes要求每个节点都要禁用swap设备，但是如果因为某些原因确实不能关闭swap分区，就需要在集群安装过程中通过明确的参数进行配置说明
  
  # 编辑分区配置文件/etc/fstab，注释掉swap分区一行
  # 注意修改完毕之后需要重启linux服务
  vi /etc/fstab
  注释掉 /dev/mapper/centos-swap swap
  # /dev/mapper/centos-swap swap
  ```

  ```shell
  # 或执行下面命令
  swapoff -a  
  sed -ri 's/.*swap.*/#&/' /etc/fstab
  ```

- 修改Linux内核参数

  ```shell
  # 修改linux的内核采纳数，添加网桥过滤和地址转发功能
  # 编辑/etc/sysctl.d/kubernetes.conf文件，添加如下配置：
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  net.ipv4.ip_forward = 1
  
  # 重新加载配置
  sysctl -p
  # 加载网桥过滤模块
  modprobe br_netfilter
  # 查看网桥过滤模块是否加载成功
  lsmod | grep br_netfilter
  ```

**操作完以上步骤，重启系统。**

安装docker

```shell
# 1.移除以前docker相关包
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
                  
# 2.配置yum源
sudo yum install -y yum-utils
sudo yum-config-manager \
--add-repo \
http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 3.安装docker
# yum install -y docker-ce docker-ce-cli containerd.io
#以下是在安装k8s的时候使用（指定docker版本）
yum install -y docker-ce-20.10.7 docker-ce-cli-20.10.7  containerd.io-1.4.6

# 4.启动docker
systemctl enable docker --now

# 5.配置加速
这里额外添加了docker的生产环境核心配置cgroup
registry-mirrors 可以配置成自己的

sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://82m9ar63.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker

查看镜像地址
docker info 
自己配置的镜像地址
Registry Mirrors:
  https://82m9ar63.mirror.aliyuncs.com/


```

安装docker

https://www.yuque.com/leifengyang/oncloud/mbvigg

安装k8s

https://www.yuque.com/leifengyang/oncloud/ghnb83

根据上面文档配置基础环境、先安装docker

### 安装k8s

#### 安装kubelet、kubeadm、kubectl

```shell
# 1.安装kubelet、kubeadm、kubectl
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
   http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 安装
sudo yum install -y kubelet-1.20.9 kubeadm-1.20.9 kubectl-1.20.9 --disableexcludes=kubernetes
# 启动 kubelet
sudo systemctl enable --now kubelet

# 2.使用kubeadm引导集群
# 2.1 下载各个机器需要的镜像
sudo tee ./images.sh <<-'EOF'
#!/bin/bash
images=(
kube-apiserver:v1.20.9
kube-proxy:v1.20.9
kube-controller-manager:v1.20.9
kube-scheduler:v1.20.9
coredns:1.7.0
etcd:3.4.13-0
pause:3.2
)
for imageName in ${images[@]} ; do
docker pull registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/$imageName
done
EOF
   
chmod +x ./images.sh && ./images.sh
# 2.2 初始化主节点
#所有机器添加master域名映射，以下需要修改为自己的
echo "192.168.56.101  cluster-endpoint" >> /etc/hosts
```



#### 集群初始化

```shell
# 只在 master 执行
kubeadm init \
--apiserver-advertise-address=192.168.56.101 \
--control-plane-endpoint=cluster-endpoint \
--image-repository registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images \
--kubernetes-version v1.20.9 \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=192.110.0.0/16

# apiserver-advertise-address 是 master 节点的ip
# service 网络范围 10.96.0.0/16
# pod 网络范围 192.110.0.0/16
# --pod-network-cidr 默认值是 192.168.0.0/16

#所有网络范围不重叠
```

```shell
# 执行 kubeadm init 命令后，可以看到一下提示
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join cluster-endpoint:6443 --token wvntod.h10mx2jgawjr76gv \
    --discovery-token-ca-cert-hash sha256:6e2ba8fd9c8f8eeffd5b84bb878e87ad3763df6d3b30994db796e9dc9edb466d \
    --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join cluster-endpoint:6443 --token wvntod.h10mx2jgawjr76gv \
    --discovery-token-ca-cert-hash sha256:6e2ba8fd9c8f8eeffd5b84bb878e87ad3763df6d3b30994db796e9dc9edb466d 

```

根据上面提示

```shell
# 1.先执行下面命令
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 加入新的 master 节点（此处不执行）
kubeadm join cluster-endpoint:6443 --token wvntod.h10mx2jgawjr76gv \
    --discovery-token-ca-cert-hash sha256:6e2ba8fd9c8f8eeffd5b84bb878e87ad3763df6d3b30994db796e9dc9edb466d \
    --control-plane 
    
# 需要加入新的 worker 节点，在其他机器上执行（此处不执行）
kubeadm join cluster-endpoint:6443 --token wvntod.h10mx2jgawjr76gv \
    --discovery-token-ca-cert-hash sha256:6e2ba8fd9c8f8eeffd5b84bb878e87ad3763df6d3b30994db796e9dc9edb466d 
```



#### 安装网络插件，只在 master 节点操作 

```shell
curl https://docs.projectcalico.org/archive/v3.21/manifests/calico.yaml -O
```

因为我在执行`kubeadm init`修改 `pod-network-cidr`的默认值， 这里需要修改 `calico.yaml` 内的配置

```shell
# 打开下载的 calico.yaml 文件，修改value和上面 pod-network-cidr 的值保持一致
- name: CALICO_IPV4POOL_CIDR
  value: "192.110.0.0/16"
```

```shell
# 修改完后执行
kubectl apply -f calico.yaml
```



#### 加入 worker 节点

在node102、node103 上分别执行（根据自己的实际参数执行），加入 worker 节点

```shell
kubeadm join cluster-endpoint:6443 --token wvntod.h10mx2jgawjr76gv \
    --discovery-token-ca-cert-hash sha256:6e2ba8fd9c8f8eeffd5b84bb878e87ad3763df6d3b30994db796e9dc9edb466d 
```

> 获取新令牌
>
> kubeadm token create --print-join-command

#### 验证集群节点状态

```shell
kubectl get nodes
```

等一会就可以看到所有节点都是 Ready 状态

```shell
[root@node101 ~]# kubectl get nodes
NAME      STATUS   ROLES                  AGE     VERSION
node101   Ready    control-plane,master   131d    v1.20.9
node102   Ready    <none>                 131d    v1.20.9
node103   Ready    <none>                 2m42s   v1.20.9

```



每秒查看

```shell
watch -n 1 kubectl get pod -A

kubectl get pod -w
```



#### 安装 dashboard （可选）

令牌

```text
eyJhbGciOiJSUzI1NiIsImtpZCI6Im5BR0lyWklSQnlJbHExV2FIaVZtR0p1MnJaMjZUelo0eW5vS3I3d1VDMDAifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXFjcXJkIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI5ZDBjNDBmNC1iMWY2LTRlOTAtYTI4YS1iZjRmZGQxMDViYzAiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.OzwIlibvEEqxfSN7EFAwVivJbzN_-WHLSaDJCCNU1JxWjYKbkBDiPW86riKN4N7MlNfPWxuMTEvMdVZzXfW5HI1lHeUcWIPhsb-TdPOC_ep2YK_kuCkgEwNR9vSYeSYGAfobr0DPpO9vXWBkVnwOvSB9zAWuv0A67L2Z8Ixf0LsPASXnwHLs-qov6jhBjLd-1zqLzYLLHCle8BM-j1wZsW4VvirsjdfNtTHTHexB5UDim7tNEK0VWNV7Wt1_Yii-eUiUP2fRLwqpmREYo54Wy3PPYQdusaCu0ZgOlOZ3Fdfl6lSq3WMmSqHDCbEwyR-ZcUYMrUoVs8oqJrKN8eruyw
```



访问 https://192.168.56.101:31769/

```text
Chrome 安全限制
https://blog.csdn.net/CloudNineN/article/details/119725431
thisisunsafe 回车
```



### k8s核心

#### Namespace

```shell
# 查看所有 namespace
kubectl get namespace
kubectl get ns

# 创建 namespace
kubectl create namespace dev
kubectl create ns dev

# 查看 namespace 详细信息
kubectl describe ns default

# 删除
kubectl delete ns dev
```



```yml
# vi ns-dev.yaml
apiVersion: v1
kind: Namespace
metadata:
 name: dev
 
---
apiVersion: v1
kind: Pod
metadata:
 name: mynginx
 namespace: dev
spec:
 containers:
 - name: nginx-container
   image:  nginx:1.17.1
```



```shell
kubectl create -f ns-dev.yaml
kubectl delete -f ns-dev.yaml

kubectl apply -f ns-dev.yaml
```



```shell
# 进入容器内部
kubectl exec -it pod-name -n dev sh


```



查看帮助

```shell
kubectl help

kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
# 查看镜像拉取策略
kubectl explain pod.spec.containers.imagePullPolicy
```



#### Pod

```shell
kubectl get pod -n dev

kubectl run nginx --image=nginx:1.17.1 --port=80 --namespace dev

# kubectl run pod名称
# --image= 指定pod镜像
# --port 指定端口
# --namespace 指定namespace

# 查看pod
kubectl get pod -n dev

# 查看某个pod
kubectl get pod pod_name -n dev

# 查看pod的详细信息
kubectl get pod -o wide -n dev

# 查看pod 以yaml形式展示结果
kubectl get pod pod_name -o yaml -n dev

# 查看pod 以json形式展示结果
kubectl get pod pod_name -o json -n dev

# 查看pod描述信息
kubectl describe pod pod_name -n dev

kubectl delete pod nginx

```



### Label

pod-nginx.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: nginx
 namespace: dev
spec:
 containers:
 - image: nginx:1.17.1
   name: pod
   ports:
   - name: nginx-port
     containerPort: 80
     protocol: TCP
```

```shell
# 创建 pod
kubectl apply -f pod-nginx.yaml
```

```shell
# 添加标签
kubectl label pod nginx version=1.0 -n dev

# 查看标签
kubectl get pod -n dev --show-labels

# 更新标签
kubectl label pod nginx version=2.0 --overwrite -n dev

# 筛选标签
kubectl get pod -n dev -l version=1.0 --show-labels

# 删除标签
kubectl label pod nginx version- -n dev

```

配置方式打标签

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: nginx
 namespace: dev
 labels:
  version: "3.0"
  env: "dev"
spec:
 containers:
 - image: nginx:1.17.1
   name: pod
   ports:
   - name: nginx-port
     containerPort: 80
     protocol: TCP
```



#### 端口设置

```yaml
# vi pod-ports.yaml
apiVersion: v1
kind: Pod
metadata:
 name: pod-ports
 namespace: dev
spec:
 containers:
 - name: nginx
   image: nginx:1.17.1
   ports:
   - name: nginx-port
     containerPort: 80
     protocol: TCP
```

访问容器内的程序需要使用podIP:containerPort

#### 资源配额

```yaml
# vi pod-resources.yaml
apiVersion: v1
kind: Pod
metadata:
 name: pod-resources
 namespace: dev
spec:
 containers:
 - name: nginx
   image: nginx:1.17.1
   resources:
    limits: # 限制资源上限
     cpu: "2" # CPU限制，单位时core数
     memory: "1Gi" # 内存限制
    requests: # 请求资源下限
     cpu: "1"
     memory: "10Mi"
```

CPU：core数，可以是整数或小数

Memory：内存大小，可以使用Gi、Mi、G、M等形式



### Pod控制器

pod 控制器用于pod的管理，确保pod资源符合预期的状态，当pod资源出现故障时，会尝试进行重启或重建pod。

#### deployment 无状态应用部署

```shell
# 直接运行一个pod
kubectl run mynginx --image=nginx --namespace=dev
# --namespace 简写成 -n

# 通过 deployment 创建，控制 pod 行为
kubectl create deployment mynginx --image=nginx:1.17.1 -n=dev
```
通过yaml创建deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: mynginx
 namespace: dev
 labels:
  app: mynginx
spec:
 replicas: 3
 selector:
  matchLabels:
   app: mynginx
 template:
  metadata:
   labels:
    app: mynginx
  spec:
   containers:
   - image: nginx:1.17.1
     imagePullPolicy: IfNotPresent
     name: nginx
```

```shell
# 查看RS(ReplicaSet) 和 Pod 信息
kubectl get rs,pod -n dev
```

```shell
# 查看 pod
kubectl get pod -n dev --show-labels
# 可以看到 mynginx-xxx 的 pod 包含 app=mynginx 标签

# 查看 deployment
kubectl get deploy -n dev -o wide
# 可以看到 mynginx 的 deployment 的选择器是 app=mynginx
# deployment 和 pod 是通过 label 关联的

# 以 yaml 形式查看 deployment
kubectl get deploy mynginx -n dev -o yaml

# 删除 pod
kubectl delete pod mynginx -n dev
# 删除 deployment 会自动删除pod
kubectl delete deploy mytomcat -n dev

```

- 多副本

```shell
kubectl create deployment mynginx --image=nginx --replicas=3 -n=dev

kubectl describe deploy mynginx -n dev
```



```shell
# -o wide 可以查看更多信息
kubectl get pod -o wide
```

- 扩缩容

```shell
# 手动通过kubectl scale 命令
kubectl scale --replicas=5 deploy mynginx -n dev

# 自动扩缩容 HPA

```

- 自愈和故障转移，自动

  

- 滚动更新（不停机维护）
```shell
kubectl edit deploy mynginx -n dev
# 修改镜像版本

# 查看Deployment 更新过程
kubectl rollout status deploy mynginx -n dev

# 可以发现之前的rs还存在，只是副本数为0
kubectl get rs,pod -n dev
NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/mynginx-6cb95bd59d   0         0         0       20m
replicaset.apps/mynginx-75b7cc87bc   3         3         3       5m35s

NAME                           READY   STATUS    RESTARTS      AGE
pod/mynginx-75b7cc87bc-f2z65   1/1     Running   0             43s
pod/mynginx-75b7cc87bc-gq2kl   1/1     Running   0             45s
pod/mynginx-75b7cc87bc-r88b2   1/1     Running   0             5m35s

```
- 版本回退
```shell
# 查看deployment 部署的历史记录（在创建deployment时使用--record参数，就可以在CHANGE-CAUSE列看到每个版本使用的命令了）
#
kubectl rollout history deploy mynginx -n dev
deployment.apps/mynginx 
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=nginx-deployment.yaml --record=true
2         kubectl apply --filename=nginx-deployment.yaml --record=true

# 查看特定版本的详细信息
kubectl rollout history deploy mynginx --revision=2  -n dev
deployment.apps/mynginx with revision #2
Pod Template:
  Labels:	app=mynginx
	pod-template-hash=75b7cc87bc
  Annotations:	kubernetes.io/change-cause: kubectl apply --filename=nginx-deployment.yaml --record=true
  Containers:
   nginx:
    Image:	nginx:1.17.2
    Port:	<none>
    Host Port:	<none>
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>



get deploy -n dev -owide
NAME      READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES         SELECTOR
mynginx   3/3     3            3           7m15s   nginx        nginx:1.17.2   app=mynginx

# 回滚到上一个版本
kubectl rollout undo deploy mynginx

kubectl get deploy -n dev -owide          
NAME      READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES         SELECTOR
mynginx   3/3     2            3           7m34s   nginx        nginx:1.17.1   app=mynginx

# 此时nginx镜像版本已经回退到了nginx:1.17.1

# 使用 --to-revision 回退到指定版本
kubectl rollout history deploy mynginx --to-revision=2  -n dev 
```


#### Service：Pod 的服务发现和负载均衡

- ClusterIP 集群内部访问

```shell
# 暴露 Service，Service 是通过 selector Deployment 的 label 进行关联的
kubectl expose deploy mynginx --name=svc-mynginx --type=ClusterIP --port=80 --target-port=80 -n dev

# 查看 Service
kubectl get svc -n dev -o wide
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE   SELECTOR
svc-mynginx   ClusterIP   10.96.74.128   <none>        80/TCP    19s   app=mynginx

# 查看 pod 标签
kubectl get pod --show-labels


kubectl get svc

服务名.名称空间.svc:port 应用内部访问
```

- NodePort 集群外部也可以访问

```shell
kubectl expose deploy mynginx --name=svc-mynginx2 --type=NodePort --port=80 --target-port=80 -n dev

# 查看
kubectl get svc svc-mynginx2  -n dev -o wide
NAME           TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE   SELECTOR
svc-mynginx2   NodePort   10.96.4.121   <none>        80:32185/TCP   49s   app=mynginx

http://192.168.56.101:32185

# 删除svc
kubectl delete svc svc-mynginx2 -n dev


```



svc-nginx.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
 name: svc-nginx
 namespace: dev
spec:
 clusterIP: 10.96.4.100
 ports:
 - port: 80
   protocol: TCP
   targetPort: 80
 selector:
  app: mynginx
 type: ClusterIP 
```



#### Ingress：Service 的统一网关入口

```shell
ingress-nginx
```



#### ConfigMap

https://kubernetes.io/docs/concepts/configuration/configmap/

```yaml
# vi configmap-demo.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
  namespace: dev
data:
  # property-like keys; each key maps to a simple value
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # file-like keys
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
```

- 创建ConfigMap

`kubectl apply -f configmap-demo.yaml`

- 查看ConfigMap

`kubectl describe cm game-demo -n dev`

- 在Pod中使用ConfigMap，通过环境变量或挂载卷的形式使用ConfigMap

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
  namespace: dev
spec:
  containers:
    - name: demo
      image: alpine
      imagePullPolicy: IfNotPresent
      command: ["sleep", "3600"]
      env:
        # Define the environment variable
        - name: PLAYER_INITIAL_LIVES # Notice that the case is different here
                                     # from the key name in the ConfigMap.
          valueFrom:
            configMapKeyRef:
              name: game-demo           # The ConfigMap this value comes from.
              key: player_initial_lives # The key to fetch.
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
  # You set volumes at the Pod level, then mount them into containers inside that Pod
  - name: config
    configMap:
      # Provide the name of the ConfigMap you want to mount.
      name: game-demo
      # An array of keys from the ConfigMap to create as files
      items:
      - key: "game.properties"
        path: "game.properties"
      - key: "user-interface.properties"
        path: "user-interface.properties"
```

与官网相比，增加了namespace 和 imagePullPolicy

- 进入容器

`kubectl exec -it configmap-demo-pod -n dev sh`

- 查看配置

```sh
echo $UI_PROPERTIES_FILE_NAME

ls /config
```

#### Configure a Pod to Use a ConfigMap

https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/

