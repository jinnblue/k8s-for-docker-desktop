# 一.添加相应的源

由于需要下载Kubeadm，Kubelet和Kubernetes-cni，多以需要添加源。国外的直接添加google源，具体可以网上搜索。国内的推荐中科大/腾讯(http://mirrors.tencent.com/)的源，命令如下：
```bash
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main
EOF
```

# 二.下载Docker & Kubeadm & Kubelet & Kubernetes-cni

```bash
sudo apt-get update
apt-get install -y docker.io kubelet kubernetes-cni kubeadm
```
注意:如果出现以下错误
```bash
GPG error: ...  could not be verified because the public key is not available: NO_PUBKEY FEEA9169307EA071 NO_PUBKEY 8B57C5C2836F4BEB
```
解决办法:
```bash
sudo gpg --keyserver keyserver.ubuntu.com --recv 8B57C5C2836F4BEB //(这个公钥根据提示来写)
sudo gpg --export --armor 8B57C5C2836F4BEB | sudo apt-key add -
```

# 三.关闭swap
K8S 要求禁用服务器的 swap 分区，同时由于所有容器的网络流量需要流经 iptables，因此还需要加载 linux 内核中的 br_netfilter 模块。如果不关闭kubernetes运行会出现错误， 即使安装成功了，node重启后也会出现kubernetes server运行错误。
```bash
sudo swapoff -a #暂时关闭，永久关闭可以上网查询
```

# 四.学习环境
[K8S官方文档Doc](https://kubernetes.io/zh/docs/home/)
```bash
minikube start --kubernetes-version='1.21.1' --registry-mirror=https://docker.mirrors.ustc.edu.cn

minikube dashboard
```
安装minikube[官方文档](https://minikube.sigs.k8s.io/docs/start/)


# 五.正式环境
## 4.获取镜像列表
- 简单方法:
```bash
kubeadm config images pull --image-repository=http://registry.cn-hangzhou.aliyuncs.com/google_containers 
```

- 手动换源 -- 由于官方镜像地址被墙，所以我们需要首先获取所需镜像以及它们的版本。然后从国内镜像站获取。
```bash
kubeadm config images list
```
```bash
#下面的镜像版本换成上面命令获取到的版本(去除前缀"k8s.gcr.io/")
images=(
    kube-apiserver:v1.21.1
    kube-controller-manager:v1.21.1
    kube-scheduler:v1.21.1
    kube-proxy:v1.21.1
    pause:3.4.1
    etcd:3.4.13-0
    coredns:v1.8.0
)

for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
```

## 5.初始化环境
这个就很简单了，只需要简单的一个命令：
```bash
kubeadm init # 这一步注意，如果需要特定的网络插件，需要额外加参数，具体看网络插件的介绍
#kubeadm config images pull --image-repository=http://registry.cn-hangzhou.aliyuncs.com/google_containers 能够 pull 镜像，但是最后还是要手动改 tag 为 http://k8s.gcr.io/xxxx，否则 kubeadm 识别不了,会重新拉取 k8s.gcr.io/xxx 镜像
#1.使用 docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/xxxxxxxxx k8s.gcr.io/xxxxxx
#2.在初始化时指定repository  指定网段pod-network-cidr
sudo swapoff -a
kubeadm init --image-repository=http://registry.aliyuncs.com/google_containers --pod-network-cidr=10.244.0.0/16
```

## 6.配置授权信息
所需的命令在init成功后也会有提示，主要是为了保存相关的配置信息在用户目录下，这样不用每次都输入相关的认证信息。
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```