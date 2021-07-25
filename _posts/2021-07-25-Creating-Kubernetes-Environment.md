---
layout: post
title: 手把手教你搭建k8s集群环境
---

## 0x00 环境
- Vmware虚拟机三台  
- Centos7  
- 可以访问k8s官方仓库的网络代理  
- Docker Version 20.10.7（截止目前最新）
- kube* v1.21.3  

## 0x01 准备
准备好三台Centos7的虚拟机，**CPU的核心至少为2核，内存为2G以上**，虚拟机网卡模式是Nat，只要三台虚拟机能互相ping通即可，三台虚拟机的作用如下：  
|ip|说明|  
|---|---|  
|192.168.10.4 |master|  
|192.168.10.5 |node1|  
|192.168.10.6 |node2|  
|192.168.10.1   |网关(物理机)|  
在 **/etc/hosts**中添加以上hostname对应的ip。  

## 0x02 开始
### 安装docker
**三台虚拟机都需要安装以及配置**，参考[官网的教程即可](https://docs.docker.com/engine/install/centos/)，这里我们安装的是最新版，安装好了之后运行:   
```bash
sudo docker run hello-world
```
如果成功打印出信息，证明安装成功。  
安装成功之后，我们需要给docker配置代理,切换到`root`用户:  
```bash
mkdir -p /etc/systemd/system/docker.service.d

cat <<EOF >/etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=socks5://192.168.10.1:1080/"
Environment="HTTPS_PROXY=socks5://192.168.10.1:1080/"
Environment="NO_PROXY=localhost,127.0.0.1,localaddress,.localdomain.com"
EOF

systemctl daemon-reload
systemctl restart docker
```
验证代理是否配置成功：  
```bash
docker info | grep Proxy  # 有输出说明配置成功
sudo docker pull gcr.io/google-containers/busybox:1.27 # pull 成功代表工作正常。
```  
由于docker默认`cgroupdriver`为`cgroupfs`,为了安装`kubeadm`，我们需要将`cgroupdriver`更改为`systemd`：  
```bash
sudo vim /etc/docker/daemon.json
```
写入:
```json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```
重启docker
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```
### 安装k8s集群
#### 关闭SELinux
三台都关闭：  
```bash
# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
设置完毕之后重启，**别忘了开启docker服务。**

#### 关闭防火墙
三台虚拟机关闭防火墙：  
```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```
#### 配置iptables

```bash
sudo vim /etc/sysctl.d/k8s.conf
```
三台都写入:  

```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```
运行
```bash
sudo sysctl --system
```
让配置生效。  
#### 配置代理
当前shell配置代理  
```bash
export http_proxy="socks5://192.168.10.1:1080/" && 
export https_proxy="socks5://192.168.10.1:1080/" && 
export no_proxy="192.168.10.4,192.168.10.5,192.168.10.6,192.168.10.1"
```
为`yum`配置代理

#### 关闭交换空间
三台都关闭  
```bash
sudo swapoff -a
```

#### 安装k8套件
三台都需要安装，切换到`root`用户，[安装参考官网](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)：
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
yum update
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable --now kubelet
```
初始化`master`节点，**在master虚拟机上运行**：  

```bash
kubeadm init --apiserver-advertise-address=192.168.10.4 --pod-network-cidr=10.244.0.0/16
```
- `--apiserver-advertise-address`绑定`apiserver`到`master`节点，这里要写`master`的地址  
- `--pod-network-cidr`代表`pod`的网络地址  

当最后出现：
```
Then you can join any number of worker nodes by running the following on each as root:
kubeadm join 192.168.10.4:6443 --token
```
那么恭喜你，初始化成功！保存好输出的信息，加入节点会用到，接下来切换成普通用户，按照输出的提示，执行：  
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
#### 部署flannel网络
初始化成功之后我们还需要对网络部署，还是在master中，[参考github](https://github.com/flannel-io/flannel#deploying-flannel-manually),对于k8s套件版本在1.17以上，直接运行:  
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
#### 加入节点
找到之前保存好的初始化后的输出信息，里面有`token`和`sha256`，在`node1`和`node2`下root权限运行:  
```bash
kubeadm join 192.168.10.4:6443 --token xxxx.xxxxxxxxxx \
	--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxx 
```
返回`master`节点，运行:   
```bash
kubectl get nodes
```
输出:  
```console
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   15h   v1.21.3
node1    Ready    <none>                 14h   v1.21.3
node2    Ready    <none>                 14h   v1.21.3

```
如果都是Ready，证明加入成功  

#### 部署Dashboard
执行：  
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
kubectl proxy
```
访问<http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login>   
如果出现登录页面，说明成功，这时候会提示我们输入`token`，为了获取`token`，我们需要为`ServiceAccount`创建`ClusterRoleBinding`,[参考github教程](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)  
创建配置文件:  
```bash
vim dashboard-adminuser.yaml
```
写入：  
```yaml
apiVersion: rbac.authorization.k8s.io/v1
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
  namespace: kubernetes-dashboard

```
执行：  
```bash
kubectl apply -f dashboard-adminuser.yaml
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
```
当打印出token后，即可登录到dashboard。

## 0x04 遇到的问题
### dashboard 503
部署好dashboard之后访问，出现错误：  
```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "no endpoints available for service \"https:kubernetes-dashboard:\"",
  "reason": "ServiceUnavailable",
  "code": 503
}
```
通过:  
```bash
kubectl get pods --all-namespaces
```
发现`coredns`的状态是`CrashLoopBackOff`，继续查看日志：  
```bash
kubectl logs coredns-xxxx --namespace=kube-system
```
发现错误信息如下：  
```
Get https://10.96.0.1:443/api/v1/endpoints?limit=500&resourceVersion=0: dial tcp 10.96.0.1:443: connect: connection refused
```
解决方法:  
```bash
vim /etc/sysctl.d/k8s.conf
```
添加  
```
net.ipv4.conf.all.forwarding = 1
```
添加iptables，重启服务  
```bash
sudo iptables -P FORWARD ACCEPT
sudo sysctl --system
sudo systemctl restart kubelet
sudo systemctl restart docker
```
这时候`coredns`的状态便正常了。  

### 物理机无法访问Dashboard
由于高版本的`Dashboard`只能`master`本地访问，我们需要做一个端口转发:  
```bash
ssh -L localhost:8001:localhost:8001 -NT user@master
```
这时，物理机的8001端口就是`master`的8001端口。

## 0x03 参考
- https://github.com/flannel-io/flannel#deploying-flannel-manually
- https://github.com/c-rainstorm/blog/blob/master/devops/%E6%9C%AC%E6%9C%BA%E6%90%AD%E5%BB%BA%E4%B8%89%E8%8A%82%E7%82%B9k8s%E9%9B%86%E7%BE%A4.md
- https://blog.csdn.net/YuYunTan/article/details/101558792
- https://stackoverflow.com/questions/60782064/coredns-has-problems-getting-endpoints-services-namespaces
- https://github.com/kubernetes/kubeadm/issues/193
- https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md
- https://blog.csdn.net/LoveyourselfJiuhao/article/details/91044268
- https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- https://docs.docker.com/engine/install/centos
- https://segmentfault.com/a/1190000023130407