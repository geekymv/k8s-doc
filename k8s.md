#### 环境准备

- 3台机器

  | 角色     | 主机名  | IP地址         | 组件                              |
  | :------- | ------- | :------------- | :-------------------------------- |
  | master01 | node101 | 192.168.56.101 | docker，kubectl，kubeadm，kubelet |
  | node01   | node102 | 192.168.56.102 | docker，kubectl，kubeadm，kubelet |
  | node02   | node103 | 192.168.56.103 | docker，kubectl，kubeadm，kubelet |

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



**操作完以上步骤，重启系统。**



集群初始化

```shell
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
```

```shell
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



安装网络插件，只在 master 节点操作 

```shell
curl https://docs.projectcalico.org/archive/v3.21/manifests/calico.yaml -O
```



在node102、node103 上分别执行（根据自己的实际参数执行）

```shell
kubeadm join cluster-endpoint:6443 --token wvntod.h10mx2jgawjr76gv \
    --discovery-token-ca-cert-hash sha256:6e2ba8fd9c8f8eeffd5b84bb878e87ad3763df6d3b30994db796e9dc9edb466d 
```







每秒查看

```shell
watch -n 1 kubectl get pod -A

kubectl get pod -w
```

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



#### Namespace

```shell
# 查看所有 namespace
kubectl get namespace
kubectl get ns

# 创建 namespace
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
```



```shell
kubectl create -f ns-dev.yaml

kubectl delete -f ns-dev.yaml
```





#### Pod

```shell
kubectl get pod -n dev

kubectl run nginx --image=nginx:1.17.1 --port=80 --namespace dev

# kubectl run pod名称
# --image= 指定pod镜像
# --port 指定端口
# --namespace 指定namespace

kubectl get pod -n dev

kubectl get pod -o wide -n dev

kubectl describe pod nginx -n dev

kubectl delete pod nginx

```



#### deployment 无状态应用部署

```shell
kubectl run mynginx --image=nginx

# 通过 deployment 创建，控制 pod 行为
kubectl create deployment mytomcat --image=tomcat:8.5.68
```



```shell
kubectl get pod

kubectl get deploy

kubectl delete pod mynginx
# 删除 deployment 会自动删除pod
kubectl delete deploy mytomcat

```

- 多副本

```shell
kubectl create deployment mynginx --image=nginx --replicas=3
```



```shell
# -o wide 可以查看更多信息
kubectl get pod -o wide
```

- 扩缩容

```shell
kubectl scale --replicas=5 deployment/mynginx
```

- 自愈和故障转移，自动

  

- 滚动更新（不停机维护）、版本回退



#### Service：Pod 的服务发现和负载均衡

- ClusterIP 集群内部访问

```shell
# 查看 pod 标签
kubectl get pod --show-labels


kubectl get svc

服务名.名称空间.svc:port 应用内部访问
```

- NodePort 集群外部也可以访问



#### Ingress：Service 的统一网关入口

```shell
ingress-nginx
```





