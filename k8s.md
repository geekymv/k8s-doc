#### 环境准备

- 3台机器

  | 角色     | 主机名  | IP地址         | 组件                              |
  | :------- | ------- | :------------- | :-------------------------------- |
  | master01 | node201 | 192.168.56.201 | docker，kubectl，kubeadm，kubelet |
  | node01   | node202 | 192.168.56.202 | docker，kubectl，kubeadm，kubelet |
  | node02   | node203 | 192.168.56.203 | docker，kubectl，kubeadm，kubelet |

- 查看操作系统版本

  ```shell
  [root@node201 ~]# cat /etc/redhat-release 
  CentOS Linux release 7.7.1908 (Core)
  ```

- 修改主机名

  ```shell
  hostnamectl set-hostname node201
  hostnamectl set-hostname node202
  hostnamectl set-hostname node203
  ```





