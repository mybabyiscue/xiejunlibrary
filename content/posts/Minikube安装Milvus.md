---
title: 测试文件
date: 2024-12-09T14:49:41+0800
description: "这是一个测试文件"
tags: [test]
---

<h1 id="ezRe8">环境准备</h1>
<h2 id="o8a2V">硬件设备</h2>
| hostname | OS Version | IP | cpu |
| --- | --- | --- | --- |
| master | centos7.9 | 192.168.7.241 | 4 |
| slave1 | centos7.9 | 192.168.130.201 | 4 |


<h2 id="ytr4C">系统配置</h2>
<h3 id="BzxTv">免密设置(所有机器)</h3>
```shell
# 设置机器名称
hostnamectl set-hostname xxx
su # 这样机器名称可以直接展示
# 登录 master、slave1 每个节点都执行。
ssh-keygen -t rsa	（ssh-keygen这里一路回车就行）

ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.7.241 && \
ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.130.201 && \

```

上述执行有点麻烦，下面提供一个shell脚本，方便运行

```shell
#!/bin/bash

# 用户名和密码
username="YourUsername"
password="cstorfs"

# 机器列表
machines=("192.168.10.55" "192.168.10.56" "192.168.10.57")

# 生成 SSH 密钥对
echo "Generating SSH key pair..."
ssh-keygen -b 2048 -t rsa -f ~/.ssh/id_rsa -N ""

# 拷贝公钥到其他机器并设置免密登录
for machine in "${machines[@]}"; do
  echo "Copying public key to $machine..."
  sshpass -p "$password" ssh-copy-id -o StrictHostKeyChecking=no "$username@$machine"
  echo "Done for $machine"
done

echo "SSH key setup completed."

```



<h3 id="w17Pe">配置/etc/hosts</h3>
```shell
# vim /etc/hosts
192.168.10.245 master
192.168.10.219 slave1
192.168.10.252 slave2
```



<h3 id="sALAX">关闭编辑 /etc/sysctl.d/k8s.conf 将流量传递到iptables链</h3>
```shell
# vim /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
# sysctl --system
# vim /etc/sysctl.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
# sysctl -p
```



<h3 id="FChqu">关闭防⽕墙和NetworkManager服务与swap，并设置开机不启动</h3>
```shell
# systemctl stop firewalld NetworkManager
# systemctl disable firewalld NetworkManager

# vim /etc/selinux/config
SELINUX=disabled


#暂时关闭SWAP，重启后恢复

swapoff   -a

#永久关闭SWAP

vim /etc/fstab

   # swap was on /dev/sda11 during installation
   #UUID=0a55fdb5-a9d8-4215-80f7-f42f75644f69 none  swap    sw      0       0
```





<h2 id="DZ1Z3">部署docker（所有节点）</h2>
<h3 id="Q1EjN">1、安装驱动⼯具</h3>
```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
```

<h3 id="iNQz8">2、添加docker源</h3>
```shell
yum-config-manager --add-repo "http://mirrors.aliyun.com/docker-ce/linux/centos/dockerce.repo"
```

<h3 id="Biyzk">3、安装docker</h3>
```shell
yum install -y docker-ce-20.10.11 docker-ce-cli-20.10.11 containerd.io
```

<h3 id="XdtID">4、配置docker/daemon.json</h3>
```shell
# mkdir /etc/docker
# tee /etc/docker/daemon.json <<EOF
{
"storage-driver": "devicemapper",
"storage-opts": [
"dm.thinpooldev=/dev/mapper/docker-thinpool",
"dm.use_deferred_removal=true",
"dm.use_deferred_deletion=true",
"dm.basesize=30G"
]
}
EOF
```

<h3 id="kI1bz">5、启动docker 并设置为开机启动</h3>
```shell
systemctl start docker && systemctl enable docker
```





<h2 id="WBbHK">部署Kubernetes(所有节点)</h2>
<h3 id="sNAAD">1、添加k8s源</h3>
```shell
# cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
EOF
```

<h3 id="dqj0V">2、安装k8s⼯具</h3>
```shell
yum install -y kubelet-1.21.0 kubeadm-1.21.0 kubectl-1.21.0 --disableexcludes=kubernetes
```

<h3 id="q5h4h">3、开机启动kubelet</h3>
```shell
systemctl enable --now kubelet

#添加环境变量
/etc/profile
export KUBECONFIG=/etc/kubernetes/admin.conf
```

<h2 id="iKrIZ">master节点初始化Kubernetes</h2>
<h3 id="a3MND">1、初始化命令，并输出相应的信息</h3>
```shell
kubeadm init --apiserver-advertise-address=192.168.130.133 --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.21.0 --service-cidr=10.254.0.0/16 --pod-network-cidr=10.30.0.0/16

# 后续添加节点。可以使用（没有试过）
kubeadm token create --print-join-command
```

测试过程中会出现如下可能的报错：

![](https://cdn.nlark.com/yuque/0/2024/png/29581643/1730269021855-3b55c55f-260c-4952-904e-254e0a619890.png)

> 解决方案：
>
> docker pull registry.aliyuncs.com/google_containers/coredns:v1.8.0
>
> docker tag 296a6d5035e2  registry.aliyuncs.com/google_containers/coredns/coredns:v1.8.0  重新打印标签
>
> 这个报错很奇怪，就是多了一层coredns，到时候可以先测试一下报错再改就OK了
>

<h2 id="JtLIF">slave节点join</h2>
+ 上述初始化后，运行会生成token命令，直接在子节点运行

<h2 id="VHDtu">flannel (master)</h2>
<h3 id="vYQur">部署flannel⽹络插件</h3>
1. 下载kube-flannel.yml，地址：

```shell
```
https://raw.githubusercontent.com/coreos/flannel/master/Document4 ation/kube-flannel.yml
```
```

2. 修改ConfigMap中net-conf.json配置的⽹段

```shell
因为主节点kubeadm init 的时候设置的pod-network-cidr
为10.254.0.0/16 ，这⾥也需要将net-conf.json 中默认的"Network":
"10.244.0.0/16" 修改为"Network": "10.30.0.0/16"
```

3. 创建flannel

```shell
kubectl apply -f kube-flannel.yml
```



如果出现notready的情况

```shell
# 查看污点
 kubectl describe node master  | grep -i taint

# 删除污点
kubectl taint node master node.kubernetes.io/not-ready:NoSchedule-

# 修改变量
/var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.4.1"

# 查看pod详细情况
kubectl describe pod coredns-545d6fc579-kxjdk -n kube-system
```

<h2 id="m0W3b">重新安装小技巧</h2>
```shell
# 有旧版本安装，安装有问题，初始化有问题。更换IP
kubeadm reset 

rm -rf /etc/kubernetes/
rm -rf /var/lib/kubelet/

# 安装完成后，先检查master节点是否是NoSchedule
kubectl describe node master
# 有则执行下面命令
kubectl taint node master node-role.kubernetes.io/master:NoSchedule-

```



<h2 id="iZFgU">minikube安装</h2>
<h3 id="SbCkz">下载</h3>
![](https://cdn.nlark.com/yuque/0/2023/png/29581643/1698136578967-a115a5b7-eafa-4333-9c09-63e62fd1814b.png)

<h3 id="DL471">下载到服务器后直接安装即可</h3>
rph -Uvh minikube-latest.x86_64.rpm



<h3 id="VbpmL">启动minikube </h3>
minikube start --force



<h3 id="ZmcY9">启动milvus</h3>
```shell
# 启动
kubectl apply -f milvus_manifest.yaml

# 卸载
kubectl delete -f milvus_manifest.yaml
```

 下图则是成功![](https://cdn.nlark.com/yuque/0/2023/png/29581643/1698136732259-54f70134-2139-4d17-9f4b-2ebeaf2bdbd3.png)



<h3 id="vQ3qI">连接milvus</h3>
端口指定

```shell
kubectl port-forward --address 0.0.0.0 service/my-release-milvus 9999:19530
```



minikube可视化页面



```shell
minikube dashboard

kubectl proxy --port=8100 --address=192.168.7.241 --accept-hosts='^.*' &
```



<h2 id="wYHIL"><font style="color:rgb(24, 24, 24);">Kuboard</font></h2>
kubectl apply -f [https://addons.kuboard.cn/kuboard/kuboard-v3-swr.yaml](https://addons.kuboard.cn/kuboard/kuboard-v3-swr.yaml)   --使用华为安装。下面的好像不行了

```shell
curl -o kuboard-v3.yaml https://addons.kuboard.cn/kuboard/kuboard-v3.yaml

kubectl create -f kuboard-v3.yaml

kubectl get pods -n kuboard

访问 Kuboard 在浏览器中打开链接 http://192.168.12.108:30080 ，输入初始用户名和密码登录

用户名： admin
密码： Kuboard123

# 清理遗留数据
rm -rf /usr/share/kuboard
```

![](https://cdn.nlark.com/yuque/0/2023/png/29581643/1698216356819-cd04a1ab-194a-4804-be98-da41253fb135.png)



Attu

```shell
docker run -p 8000:3000 -e MILVUS_URL=192.168.10.130:30852 zilliz/attu:v2.3.1
http://{ Attu IP }:8000
```

