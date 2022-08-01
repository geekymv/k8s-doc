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
  192.168.56.201 node101
  192.168.56.202 node102
  192.168.56.203 node103
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





操作完以上步骤，重启系统。



集群初始化

```shell
kubeadm init \
--apiserver-advertise-address=192.168.56.101 \
--control-plane-endpoint=cluster-endpoint \
--image-repository registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images \
--kubernetes-version v1.20.9 \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=192.110.0.0/16

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



```shell
curl https://docs.projectcalico.org/archive/v3.21/manifests/calico.yaml -O
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















```shell
# 创建集群
kubeadm init \
	--apiserver-advertise-address=192.168.56.201 \
	--image-repository registry.aliyuncs.com/google_containers \
	--kubernetes-version=v1.17.4 \
	--service-cidr=10.96.0.0/12 \
	--pod-network-cidr=10.244.0.0/16
```

注意：apiserver-advertise-address 是 master 节点的ip



```text
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.56.201:6443 --token xy3gng.5yxil16iymzcunom \
    --discovery-token-ca-cert-hash sha256:c134ea141bed517d90ac0b38b8682ee7081edd234af1d299a6d05453848cb4e1

```





安装网络插件，只在 master 节点操作 

```shell
https://github.com/flannel-io/flannel/tree/master/Documentation/kube-flannel.yml
```

将 kube-flannel.yml 文件下载下来，放到master 节点上，然后执行下面命令

```shell
kubectl apply -f kube-flannel.yml
```



在node202、node203 上分别执行（根据自己的实际参数执行）

```shell
kubeadm join 192.168.56.201:6443 --token o8rqe7.2u5xa365rsjdh6v6 \
    --discovery-token-ca-cert-hash sha256:0bdfee67dac5e5f72a215bacc10e605e4c8f098360025bc44bc9c4ddbacf7532 
```



集群安装完毕

```shell
[root@node201 ~]# kubectl get nodes
NAME      STATUS   ROLES    AGE     VERSION
node201   Ready    master   4h16m   v1.17.4
node202   Ready    <none>   5m20s   v1.17.4
node203   Ready    <none>   5m24s   v1.17.4
```



```shell
kubectl get pods -o wide

journalctl -f -u kubelet.service 
```



