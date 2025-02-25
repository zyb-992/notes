# 在Minikube上使用Flannel网络插件

---

> Minikube v1.31.2
>
> Docker v26.0.1
>
> Docker Desktop v27.4.0

1. 通过` minikube start --network-plugin=cni --cni=flannel`启动镜像

2. 启动后，执行`kubectl get pods -A`，会看到如下

   ```shell
   NAMESPACE      NAME                               READY   STATUS    RESTARTS      AGE
   kube-flannel   kube-flannel-ds-d7p4x              1/1     Running   0             59s
   kube-system    coredns-5d78c9869d-dt9wq           1/1     Running   3 (19s ago)   60s
   kube-system    etcd-minikube                      1/1     Running   0             72s
   kube-system    kube-apiserver-minikube            1/1     Running   0             72s
   kube-system    kube-controller-manager-minikube   1/1     Running   0             72s
   kube-system    kube-proxy-jzwqp                   1/1     Running   0             59s
   kube-system    kube-scheduler-minikube            1/1     Running   0             72s
   kube-system    storage-provisioner                1/1     Running   1 (48s ago)   71s
   ```

3. 执行` kubectl run test-pod --image=busybox --restart=Never --command -- sleep 86400`命令创建临时容器进行验证

   ```shell
   # 创建临时容器后 kubectl get po -n default
   NAMESPACE      NAME               READY   STATUS    RESTARTS        AGE
   default        test-pod           1/1     Running   0               44s
   ```

4. 进入Pod查看网络相关信息

   ```shell
   # 这里Pod的IP地址:**10.244.0.3**
   / # ifconfig
   eth0      Link encap:Ethernet  HWaddr 6A:9B:09:1E:1F:EC
             inet addr:10.244.0.3  Bcast:10.244.0.255  Mask:255.255.255.0
             UP BROADCAST RUNNING MULTICAST  MTU:65485  Metric:1
             RX packets:5 errors:0 dropped:0 overruns:0 frame:0
             TX packets:1 errors:0 dropped:0 overruns:0 carrier:0
             collisions:0 txqueuelen:0
             RX bytes:446 (446.0 B)  TX bytes:42 (42.0 B)
   
   lo        Link encap:Local Loopback
             inet addr:127.0.0.1  Mask:255.0.0.0
             UP LOOPBACK RUNNING  MTU:65536  Metric:1
             RX packets:0 errors:0 dropped:0 overruns:0 frame:0
             TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
             collisions:0 txqueuelen:1000
             RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
             
   # 查看容器里的路由表
   # 网段=10.244的数据包会发往eth0网卡
   # 网段=10.244.0网段的数据包会直接发往该容器的回环设备
   / # ip route
   default via 10.244.0.1 dev eth0
   10.244.0.0/24 dev eth0 scope link  src 10.244.0.3
   10.244.0.0/16 via 10.244.0.1 dev eth0
   
   # 查看容器里的网络设备信息
   # 网络接口索引11连接到了索引为15的网络设备
   / # ip link
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
   2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop qlen 1000
       link/ipip 0.0.0.0 brd 0.0.0.0
   11: eth0@if15: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 65485 qdisc noqueue
       link/ether 6a:9b:09:1e:1f:ec brd ff:ff:ff:ff:ff:ff
   ```

5. `minikube ssh`进入minikube节点

   ```shell
   # 查看节点的网络设备信息
   docker@minikube:/etc/cni/net.d$ ip link show
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
   2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN mode DEFAULT group default qlen 1000
       link/ipip 0.0.0.0 brd 0.0.0.0
   11: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
   15: veth591c28f1@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 65485 qdisc noqueue master cni0 state UP mode DEFAULT group default
   
   # 查看flannel配置
   docker@minikube:~$ cd /etc/cni/net.d
   
   docker@minikube:/etc/cni/net.d$ ls
   10-flannel.conflist               200-loopback.conf
   100-crio-bridge.conf.mk_disabled  87-podman-bridge.conflist.mk_disabled
   
   # 定义了Flannel的工作模式 
   docker@minikube:/etc/cni/net.d$ cat 10-flannel.conflist
   {
     "name": "cbr0",
     "cniVersion": "0.3.1",
     "plugins": [
       {
         "type": "flannel",
         "delegate": {
           "hairpinMode": true,
           "isDefaultGateway": true
         }
       },
       {
         "type": "portmap",
         "capabilities": {
           "portMappings": true
         }
       }
     ]
   }
   ```

6. 退出Minikube节点，本地查看Flannel的ConfigMap

   ```shell
   % kubectl get configmap -n kube-flannel kube-flannel-cfg -o yaml
   apiVersion: v1
   data:
     cni-conf.json: |
       {
         "name": "cbr0",
         "cniVersion": "0.3.1",
         "plugins": [
           {
             "type": "flannel",
             "delegate": {
               "hairpinMode": true,
               "isDefaultGateway": true
             }
           },
           {
             "type": "portmap",
             "capabilities": {
               "portMappings": true
             }
           }
         ]
       }
     # 这里的网段就是与后续创建Pod时分配的IP网段一致
     net-conf.json: |
       {
         "Network": "10.244.0.0/16",
         "Backend": {
           # 表示Flannel的工作类型为vxlan，二层虚拟网络，工作在物理层
           "Type": "vxlan"
         }
       }
   kind: ConfigMap
   metadata:
     annotations:
       kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"v1","data":{"cni-conf.json":"{\n  \"name\": \"cbr0\",\n  \"cniVersion\": \"0.3.1\",\n  \"plugins\": [\n    {\n      \"type\": \"flannel\",\n      \"delegate\": {\n        \"hairpinMode\": true,\n        \"isDefaultGateway\": true\n      }\n    },\n    {\n      \"type\": \"portmap\",\n      \"capabilities\": {\n        \"portMappings\": true\n      }\n    }\n  ]\n}\n","net-conf.json":"{\n  \"Network\": \"10.244.0.0/16\",\n  \"Backend\": {\n    \"Type\": \"vxlan\"\n  }\n}\n"},"kind":"ConfigMap","metadata":{"annotations":{},"labels":{"app":"flannel","k8s-app":"flannel","tier":"node"},"name":"kube-flannel-cfg","namespace":"kube-flannel"}}
     creationTimestamp: "2025-01-13T15:46:04Z"
     labels:
       app: flannel
       k8s-app: flannel
       tier: node
     name: kube-flannel-cfg
     namespace: kube-flannel
     resourceVersion: "296"
     uid: 6dd1d0fe-9e7a-40d4-9591-3b1d705cfde7
   ```

   

6. 