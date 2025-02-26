# Kubernetes Install Tutorial

## 环境说明

```bash
Operating System: Ubuntu 22.04.5 LTS              
Kernel: Linux 6.8.0-52-generic
Architecture: x86-64
Kubernetes v1.30.10
Virtual Machine Applicatoin: Vmware Workstation pro
```

环境下载：[Ubuntu iso](https://releases.ubuntu.com/jammy/)，[Vmware Workstation Pro17 Windows](https://www.reddit.com/r/vmware/comments/1d4eivz/vmware_workstation_pro_17_windows_where_to/)。下载后镜像安装过程网上随便搜下就有~

## 安装过程

1. 先安装了Docker，毕竟现在用的用户基数还是比较大，这里直接参照[官网安装过程](https://docs.docker.com/engine/install/ubuntu/)；下面是我安装过程中会遇到的问题

   - `The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 7EA0A9C3F273FCD8`
   - 关于镜像加速，我是使用了阿里云的镜像加速，但拉镜像的时候一直拉不下来，网上现在很多的文档给的镜像源都失效了，这里最终查看了[daocloud](https://github.com/DaoCloud/public-image-mirror?tab=readme-ov-file#%E5%8A%A0%E9%80%9F-docker)才获取到了有效的镜像加速。

   ```bash
   # 错误描述
   Err:1 https://download.docker.com/linux/ubuntu jammy InRelease                 
     The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 7EA0A9C3F273FCD8
     
   # 解决
   sudo mkdir -p /etc/apt/keyrings
   sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor --yes -o /etc/apt/keyrings/docker.gpg
   sudo chmod a+r /etc/apt/keyrings/docker.gpg
   
   
   # 镜像加速 /etc/docker/daemon.json
   {
     "registry-mirrors": [
       "https://docker.m.daocloud.io"
     ]
   }
   ```

   

2. 接下来是安装Kubernetes，我安装的版本是v1.30.10，`cri`是直接使用`containerd`，记得提前关闭*swap分区*哦

   ```bash
   # 关闭swap分区
   sudo swapoff -a
   # 据说是保证虚拟机重启后swap分区不会被重新挂载
   sudo sed -i 's/.*swap.*/#&/' /etc/fstab
   ```

   通过`swapon -s`或者`free -h`确认是否已关闭

   ```bash
   root@master:/etc/cni/net.d# swapon -s
   root@master:/etc/cni/net.d# free -h
                  total        used        free      shared  buff/cache   available
   Mem:           7.7Gi       1.6Gi       3.5Gi        52Mi       2.6Gi       5.8Gi
   Swap:             0B          0B          0B
   ```

   关闭后参照[kubernetes安装文档](https://v1-30.docs.kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)来安装kubeadm、kubectl、kubelet。别忘记了参照文档锁住这几个工具的版本

   

3. 这三个工具安装好后，你是否就想立即运行起k8s环境呢？别着急，潜在的问题可能还没有被你挖掘出来。你可以先通过输入`kubectl version`确认安装的Kubernetes版本是否正确，接着我们要提前检查`containerd`的配置文件

   ```bash
   # config path: /etc/containerd/config.toml，如果不存在，使用这条命令生成
   containerd config default > /etc/containerd/config.toml
   ```

   查看配置文件，找到`disabled_plugins`字段，查看它的值是否有`cri`，有的话把它去掉，然后重启`containerd`服务。否则你在执行`kubeadm init`时其预检日志可能会报如下错误

   ```bash
   # 重启服务
   systemctl daemon-reload
   systemctl restart containerd.service
   
   # 错误
   sudo kubeadm init --pod-network-cidr=10.244.0.0/16
   
   error execution phase preflight: [preflight] Some fatal errors occurred:
   	[ERROR CRI]: container runtime is not running: output: time="2025-02-23T23:20:55+08:00" level=fatal msg="validate service connection: validate CRI v1 runtime API for endpoint \"unix:///var/run/containerd/containerd.sock\": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService"
   , error: exit status 1
   ```

   现在我们要预先拉取运行k8s服务的必要镜像，这里我的虚拟机网络没有代理到主机的VPN上，所以如果你也访问不了外网，可以通过上面安装docker时提到的[daocloud](https://github.com/DaoCloud/public-image-mirror?tab=readme-ov-file#%E5%8A%A0%E9%80%9F%E5%AE%89%E8%A3%85-kubeadm)作为镜像源拉取镜像，但是要注意的比较坑的一个点，直接使用`kubeadm config images pull --image-repository k8s-gcr.m.daocloud.io`的话，对于`coredns`这个服务它的路径有点问题，拉取的时候你可能会遇到这个错误

   ```bash
   # 错误
   failed to pull image "k8s-gcr.m.daocloud.io/coredns:v1.11.3": output: E0224 00:03:19.869658    8977 remote_image.go:180] "PullImage from image service failed" err="rpc error: code = Unknown desc = failed to pull and unpack image \"k8s-gcr.m.daocloud.io/coredns:v1.11.3\": failed to resolve reference \"k8s-gcr.m.daocloud.io/coredns:v1.11.3\": unexpected status from HEAD request to https://k8s-gcr.m.daocloud.io/v2/coredns/manifests/v1.11.3: 403 Forbidden" image="k8s-gcr.m.daocloud.io/coredns:v1.11.3"
   ```

   可以参考这个[issue](https://github.com/kubernetes/kubeadm/issues/2714#issuecomment-1278671260)，我是参考了[这个回答](https://github.com/kubernetes/kubeadm/issues/2714#issuecomment-1278671260)、[这个回答](https://github.com/DaoCloud/public-image-mirror/issues/36080)后成功拉取的。你可以按我下面的步骤来

   ```bash
   vim init-pull-image.yaml
   
   # 将如下内容填充进去，不要把我贴心的注释也填进去，谢谢🙏
   kind: InitConfiguration
   localAPIEndpoint:
   # 通过ifconfig获取到你当前的虚拟网卡ens33的ip
     advertiseAddress: 192.168.150.128
     bindPort: 6443
   nodeRegistration:
     criSocket: unix:///var/run/containerd/containerd.sock
     imagePullPolicy: IfNotPresent
     # name修改成你的主机名，通过hostname可以获取到，我的是master
     name: master
     taints: null
   ---
   apiServer:
     timeoutForControlPlane: 4m0s
   apiVersion: kubeadm.k8s.io/v1beta3
   certificatesDir: /etc/kubernetes/pki
   clusterName: kubernetes
   controllerManager: {}
   # 镜像仓库替换成k8s.m.daocloud.io/coredns
   dns:
     imageRepository: k8s.m.daocloud.io/coredns
   # 镜像仓库替换成k8s.m.daocloud.io
   imageRepository: k8s.m.daocloud.io
   kind: ClusterConfiguration
   # 这里修改成你安装的k8s版本，我的是1.30.10
   kubernetesVersion: 1.30.10
   networking:
   # 提前指定cidr，避免后续安装网络插件报错没有指定
     podSubnet: 10.244.0.0/16
     dnsDomain: cluster.local
     serviceSubnet: 10.96.0.0/12
   etcd:
     local:
       dataDir: /var/lib/etcd
   scheduler: {}
   ```

   上面的配置文件内容其实通过`kubeadm config print init-defaults`命令自动生成的，你也可以自己修改客制化的配置文件。修改好后，执行`kubeadm init --config ./init-pull-image.yaml`命令，每个镜像拉下来后它都会有日志打到标准输出提示，等待镜像全部拉取后自动启动k8s；启动过程中不要忘记仔细查看打印的`preflight`日志哦，像提示你更改pause镜像等还是参照它的建议完成吧。

   ```bash
   # 我的preflight日志只有这些
   [preflight] Running pre-flight checks
   [preflight] Pulling images required for setting up a Kubernetes cluster
   [preflight] This might take a minute or two, depending on the speed of your internet connection
   [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
   ```

   正常启动后，你可以通过`kubectl`查看信息，`kubectl get po -A`查看系统pod运行状态，官方文档解释的是正常启动时，除了`coredns`其他都应该时候在`Running`状态，因为`coredns`需要安装`cni`网络插件才会正常启动

   ```bash
   root@master:/home/zhengyb/install# kubectl get po -A
   NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
   kube-system   coredns-6f5b87b887-8h68j         0/1     Pending   0          5s
   kube-system   coredns-6f5b87b887-jhkt9         0/1     Pending   0          5s
   kube-system   etcd-master                      1/1     Running   1          20s
   kube-system   kube-apiserver-master            1/1     Running   1          20s
   kube-system   kube-controller-manager-master   1/1     Running   22         20s
   kube-system   kube-proxy-zdh69                 1/1     Running   0          6s
   kube-system   kube-scheduler-master            1/1     Running   21         20s
   ```

   如果运行`kubectl get po -A`时提示你连接到主机上的6443端口被拒绝时，大概率是你的k8s环境没有正常跑起来，你可以通过`crictl ps -a`看到运行异常的容器。我这里是正常运行的，因为都在`Running`；你可以随机找一个异常的容器通过`crictl logs ${container_id}`获取错误日志再进行排查。

   ```bash
   root@master:/home/zhengyb/install# crictl ps -a
   WARN[0000] runtime connect using default endpoints: [unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead. 
   WARN[0000] image connect using default endpoints: [unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead. 
   CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD
   c387919bb2830       a21d1b47e8572       2 minutes ago       Running             kube-proxy                0                   16e7e15834afe       kube-proxy-zdh69
   2928079bdb954       64edffde4bf75       2 minutes ago       Running             kube-scheduler            21                  b03ae0175aeb4       kube-scheduler-master
   488dfe50dea79       f81ad4d47d775       2 minutes ago       Running             kube-controller-manager   22                  ef9b7c8ba1f0e       kube-controller-manager-master
   ca59b9d394754       2e96e5913fc06       2 minutes ago       Running             etcd                      1                   8cc936c3b980e       etcd-master
   582a3c1ce1e15       172a4e0b731db       2 minutes ago       Running             kube-apiserver            1                   94d3110ddedc3       kube-apiserver-master
   
   ```

   像我遇到了一个晚上没解决的问题和给etcd提[issue](https://github.com/etcd-io/etcd/issues/13670)的老哥一样，错误日志`"caller":"osutil/interrupt_unix.go:64","msg":"received signal; shutting down","signal":"terminated"}`，按照这位大佬的[回答](https://github.com/etcd-io/etcd/issues/13670#issuecomment-1337727102)修改完就好了，别忘记重启`containerd`服务~。

4. 大致的安装过程就是这样了，暂时还没有遇到过其他的大坑。。这里还有篇[虚拟机连接主机VPN设置](https://blog.csdn.net/nomoremorphine/article/details/138738065)，需要的可以参考下，修改完网络连接方式后你的虚拟网卡IP可能会更改，记得同步修改掉`init-pull-image.yaml`的`advertiseAddress`字段。









## sudo权限设置



```bash
sudo usermod -aG sudo username
newgrp sudo
groups
sudo whoami -> root
```

|            |                               |
| ---------- | ----------------------------- |
| crictl     |                               |
| ctr        |                               |
| lsmod      |                               |
| ufw        |                               |
| journalctl | journalctl -xeu               |
| swapon -s  | 查看swap分区是否开启(free -h) |