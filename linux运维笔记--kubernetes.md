## linux运维笔记-->kubernetes

继docker swarm（现在已少用）后的容器编排工具，现已成为行业标准，下面简称kubernetes为k8s。它可编排的容器不止是有docker，还兼容其他的容器如红帽的podman，原因是他们都有实现OCRI容器接口标准（docker没有，它是CRI，但它有自己进行OCRI到CRI的转换）。

### 一、各基本组件结构介绍

#### etcd

它是一个使用raft算法的键值对数据库，用来存储集群服务器信息、集群资源占用信息等

在v2版本中数据存储在内存中，不稳定已弃用；在v3版存储在磁盘中，变得更可靠

#### 调度器（scheduler）

它是用来让资源合理的分发到各节点上并且监控节点运行情况的，核心算法是不能让节点空闲

#### 控制器（controller）

它是用来管理一组pod的，通过和api server交互来管理pod组的状态。在一个集群中可以有多个控制器，k8s提供多种内置的控制器，如Deployments和job等。

一些常用的控制器命令，以deployment控制器为例

```shell
# 创建控制器
kubectl create deployment <控制器名称> --image=<docker镜像名> -n <命名空间名称>
# 删除控制器，当控制器删除时底下的容器组一同被删除
kubectl delete deployment <控制器名称>
```

也可以使用yaml和kubectl创建

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <控制器名称>
  namespace: <所属命名空间名称>
  labels:
    app: <控制器名称>
spec:
  replicas: <集群容器数量>
  selector:
    matchLabels:
      app: <容器名称>
  template:
    metadata:
      labels:
        app: <容器名称>
    spec:
      containers:
      - name: <容器名称>
        image: <docker镜像名>
        ports:
        - containerPort: <容器对外暴露端口号>
```

控制器创建好后可以在kuboard图形界面中的工作负载中可以查看到，还可以看到它管理的pod组

#### 接口服务（api server）

它是所有管理员请求、用户请求的入口，我们需要k8s执行任务、查询请求、访问pod中服务等，都要下发给这个入口

下面是一些常用的命令

```shell
# 想要使用server访问到pod，首先要建立与控制器的联系，以deployment控制器为例
# 服务模式有NodePort模式可以让服务对集群外暴露端口，ClusterIP只对集群内暴露
kubectl expose deployment <控制器名称> --port=<server暴露端口> --target-port=<容器对外暴露端口> --type=<服务模式> -n <命名空间名称>
# 查看创建好的服务
kubectl get service -n <命名空间名称>
# 删除服务，服务名称一般和控制器名称会一样
kubectl delete service <服务名称> -n <命名空间名称>
```

也可以通过yaml文件来创建服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <和控制器名称相同的服务名称>
  namespace: <所属命名空间名称>
  labels:
    app: <和控制器名称相同的服务名称>
  resourceVersion: ‘<service对集群外暴露端口号>’
spec:
  clusterIP: <service的集群ip>
  selector:
    app: <联系的容器名称>
  port:
  - port: <server暴露端口>
    targetPort: <联系的容器对外暴露端口>
  type: <服务模式>
```



建立好联系后，可通过service的地址加端口访问pod服务，并且访问时是负载均衡的

#### 命令行管理工具（kubectl）

用来管理k8s的命令行工具，可安装在任何节点上，只要能连接到接口服务（但一般安装在master节点）

一些常用的pod操作命令

```shell
# 用于查看pod状态信息的命令，常用来排not ready错
kubectl describe pod <pod名称> -n <命名空间名称>
# 和docker一样可以进入容器内
kubectl exec -it <pod名称> -n <命名空间名称> -- bash
# 使用yaml文件创建集群，必须不存在，否则就报错，和apply相似但有区别
kubectl create -f <yaml文件>
# 使用yaml文件创建集群，如果文件内资源不存在则创建，存在就更新
kubectl apply -f <yaml文件>
# 使用yaml文件删除集群，和上面同一个文件，但是反向操作
kubectl delete -f <yaml文件>
# 修改创建集群的yaml，修改好后摇删除原来的pod让它再创建才生效
kubectl edit <资源> <文件名> -n <命名空间>
```

#### 网页UI（web UI）

用来管理k8s的网页，需要连接到接口服务，一般只用来看数据

#### node组件-kubelet

它会根据接口服务发来的命令，对docker容器进行操作

#### node组件-kube proxy

它会监听接口服务，调用linux内核的net link接口，对linux的ipvs和防火墙进行管控。负载均衡和自动转发都由它实现

### 二、k8s名词解释

#### 节点

k8s通过将容器放入在节点（Node）上运行的 Pod 中来执行你的工作负载。 节点可以是一个虚拟机或者物理机器，取决于所在的集群配置。 每个节点包含运行Pod所需的服务； 这些节点由控制平面负责管理。

#### 污点

污点是一个被打上污点标签的节点，所有的容器不能在此节点运行，即使用标签指定了某容器在此节点运行，那么容器的状态也会一直持续“Pending”状态。用来避免 Pod 被分配到不合适的节点上

```bash
# 查看有没有污点pod
kubectl describe nodes master1 | grep Tain
# 添加污点pod
kubectl taint nodes --all node-role.kubernetes.io/control-plane node-role.kubernetes.io/master
# 移除污点pod
kubectl taint nodes --all node-role.kubernetes.io/control-plane- node-role.kubernetes.io/master-
```

#### 容忍度

它是应用于 Pod 上的。容忍度允许调度器调度带有对应污点的 Pod。 容忍度允许调度但并不保证调度。用来避免 Pod 被分配到不合适的节点上。

#### Pod

Pod是k8s中能够创建和部署的最小单元，是k8s集群中的一个应用实例，也是扩容的一个最小单元。它总是部署在同一个节点上。Pod中包含了一个或多个容器，还包括了存储、网络等各个容器共享的资源。Pod支持多种容器环境，Docker则是最流行的容器环境。它一般会挂在控制器下，用一对多的方式管理pod。

- 单容器Pod，最常见的应用方式。
- 多容器Pod，对于多容器Pod，k8s会保证所有的容器都在同一台物理主机或虚拟主机中运行。多容器Pod是相对高阶的使用方式，除非应用耦合特别严重，一般不推荐使用这种方式。一个Pod内的容器共享IP地址和端口范围，容器之间可以通过 localhost 互相访问。

以下命令都需要在master节点下执行

通过这个命令可以查看所有的pod

```shell
# 不加-A就只查看当前命名空间下的
kubectl get pods -A
# 加-n可以查看指定命名空间下的
kubectl get pod -n <命名空间名称>
```

通过这个指令可以根据镜像运行pod，如果镜像拉取失败也可以使用docker命令直接拉取，或者检查dns和work节点问题

```shell
# pod名称英文名可随意取，--image指定启动的镜像，镜像名docker官网可以是url地址，-n指定所属命名空间，不指定就默认
kubectl run <pod名称> --image=<docker镜像名> -n <命名空间>
```

通过这个指令可以删除运行的pod

```shell
# 切记指定命名空间，否则可能误删其他命名空间下同名pod
kubectl delete pod <pod名称> -n <命名空间>
```

也可以通过yaml文件夹kubectl来创建一个多容器的pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <pod名称>
  namespace: <命名空间名称>
spec:
  containers:
  - image: <docker镜像名>
    name: <容器名称>
  - image: <docker镜像名>
    name: <容器名称>
```

yaml写好后使用这个命令就可以创建出一个多容器的pod

```shell
kubectl apply -f <yaml文件>
```



#### 网页管理（kuboard）

kuboard是一个图形化的网页管理集群插件，kuboard官网（kuboard.cn）有安装方式。

#### IngressClass

它是一个允许使用域名去访问service提供的动态ip的插件，从而做到无论service如何重启如何变更ip，域名都能够访问到service的效果。这个插件需要额外安装，在kuboard的网络下可以自动安装并且创建ingress容器。

安装好后可以通过yaml来创建容器

```yaml
apiVersion: neworking.k8s.io/v1
kind: Ingress
metadata:
  name: <Ingress名称>
  namespace: <命名空间名称>
spec:
  ingressClassName: <安装时被指定的IngressClass名称>
  rules:
  - host: <域名名称>
    http:
      paths:
      - path: <url路径名>
        pathType: <nginxlocation的路径匹配规则，Prefix>
        backend:
          service:
            name: <ingress要映射的service的名称>
            port:
              number: <service的内部暴露端口>
```



#### 命名空间（namespace）

命名空间为区分业务或各种环境而存在，可以通过kubectl创建命名空间

```shell
# 查看所有的命名空间
kubectl get namespace
# 创建命名空间
kubectl create ns <命名空间名称>
```

也可以用yaml创建

```yaml
# 注意必须要有labels这是标准，方便以后出问题查错
apiVersion: v1
kind: Namespace
metadata:
  name: <命名空间名称>
  labels:
    name: <命名空间名称>
```

yaml写好后用这个命令可以直接

```shell
kubectl apply -f <yaml文件>
```

> 注：命名空间被删除下面的东西都会被删除。

### 三、使用控制器管理Pod

#### RC

它支持副本数量与期望值之间的管理

#### RS

它的功能和RC类似，但增加了集合式的标签选择器

#### Deployment

它支持滚动更新和回滚

#### HPA

根据Pod资源使用情况调整副本数量，依赖于RC、RS和Deployment之上，目标是自动扩容缩

#### StatefulSet

它是为了解决有状态服务和无状态服务设计的。简单来说有状态服务指的是不管容器新建还是重启，里面的数据都一直存在。比如mysql数据库就是典型的有状态服务，如果让mysql以无状态方式运行那么再容器宕机新建后保存的数据将不复存在。

#### DeamonSet

它是为了确保每个一节点上都有且只有一个Pod，并且是动态的，在删除或者新增节点时都会让全部节点保持有且只有一个Pod。

#### Job

它保证批处理任务的一个或多个Pod成功结束。目标是成功退出，返回码为0。

#### CronJob

它建立在Job之上，负责基于时间管理周期性的在某个时间点运行的一次任务。例如数据库的定时多次冗余备份就可以用到。

#### replication controller

副本控制器，它是用来管理容器的状态，维稳集群的。处理一些节点扩容，重启节点等

#### 自定义控制器

可以根据k8s的手册开发自己的控制器

### 四、容器网络

#### 容器隔离

容器之间用namespace network进行网络隔离

#### 同Pod内不同容器间通信

使用IO的方式，因为距离近所以这也是通信效率最高的方式，一般是通信比较频繁的应用会放在同一个Pod内（点名nginx和php-fpm）

#### 不同Pod通信-扁平网络空间

k8s网络模型假定所有Pod都在一个可以直接连接的网络内，即使不在同一个物理机上（同一个物理机直接docker网桥），Pod之间也能通过虚拟ip轻易互相连通。但在搭建私有云情况下（谷歌云除外），需要自己实现这个假设的网络，将不同节点上的docker容器之间的互相访问先打开，然后运行k8s。

#### Flannel插件

它是CoreOS团队专门为了解决搭建这个网络模型而针对k8s开发的一个插件。它的功能时让集群中的不同节点的主机创建的docker容器都有全集群中唯一的虚拟ip地址，并且建立一个覆盖网络，通过覆盖网络把数据包原封不动（指ip用覆盖网络内部ip即可）且无视物理ip地传递到目标容器内。

无视物理ip的原理是用数据包套数据包，即用物理上真实要传输的数据包的数据体内套入容器内要传的数据包，最后在到达目标物理主机时拆开就能得到容器内用的数据包。此过程需要etcd了解物理机ip和容器内设置的ip之间的对应关系。在此过程中使用的是UDP，因为速度快但不稳定（使用UDP的原因是不同节点一般都会在同一个机房内部，传输链路短基本UDP不会丢）

#### calico插件

calico因为其性能、灵活性都好而备受欢迎，calico的功能更加全面，不但具有提供主机和pod间网络通信的功能，还有网络安全和管理的功能，而且在CNI框架之内封装了calico的功能，calico还能与服务网络技术Istio集成，不但能够更加清楚的看到网络架构也能进行灵活的网络策略的配置，calico不使用Overlay网络，配置在第三层网络，使用BGP路由协议在主机之间路由数据包，意味着不需要包装额外的封装层。主要点在网络策略配置这一点，可以提高安全性和网络环境的控制。

### 五、k8s部署（kubeadm方式）

#### 系统准备

1. 更改主机名

   ```bash
   # 应该规范命名
   master-<序号>
   worker-<序号>
   ```

2. 配置网卡ip地址

3. 修改hosts地址解析，让每个节点ip对应一个域名

4. 暂时关闭防火墙，等配置完成再开启

5. 关闭selinux

6. 关闭swap分区

7. 使用stpd同步时间

8. 升级内核（可选）

9. 安装ipvsadm和ipset，为了使用高性能的内核转发和网桥过滤（可选）

#### docker准备

1. 安装docker-ce和containd.io，不能是docker

   ```shell
   # 设置阿里的docker-ce镜像源
   yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   # 然后安装
   yum install -y docker-ce containd.io
   ```

2. git下cri-dockerd编译安装，将源码目录下的systemd的所有文件放到/etc/systemd/system/下。但在这之前需要安装go，按照cir-dockerd的官方文档来做，所有节点都要装

3. 修改cri-dockerd.service，注意看自己的cri-dockerd文件在什么地方

   ```bash
   ExecStart=/usr/local/bin/cri-dockerd --pod-infra-container-image=registry.k8s.io/pause:3.9 --container-runtime-endpoint fd://
   ```


#### k8s安装

1. 更改yum源为网速快的

   创建并修改/etc/yum.repos.d/k8s.repo内容为

   ```bash
   [kubernetes]
   name=Kubernetes
   baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
   enabled=1
   gpgcheck=0
   repo_gpgcheck=0
   gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   ```

2. 安装kubeadm kubelet kubectl

3. 修改/etc/sysconfig/kubelet，为了能使用镜像同步

   ```bash
   KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
   ```

4. 创建image_download.sh文件方便下载镜像，在work节点中只用到pause和proxy，master节点才要下这么多

   image_list不一定一样，最好使用kubeadm config images list --kubernetes-version=<需要的版本>对照

   ```bash
   #!/bin/bash
   images_list='
   registry.k8s.io/kube-apiserver:v1.26.0
   registry.k8s.io/kube-controller-manager:v1.26.0
   registry.k8s.io/kube-scheduler:v1.26.0
   registry.k8s.io/kube-proxy:v1.26.0
   registry.k8s.io/pause:3.9
   registry.k8s.io/etcd:3.5.6-0
   registry.k8s.io/coredns/coredns:v1.9.3'
   
   for i in $images_list
   do
   	docker pull $i
   done
   
   docker save -o k8s.tar $images_list
   ```

5. 启动服务kubelet以及开放端口

6. 初始化集群，记录下添加其他工作节点需要使用的token和discovery-token-ca-cert-hash

   ```bash
   # pod-network-cidr是pod使用的网段，根据需要划分；apiserver-advertise-address一般是主节点的ip
   kubeadm init --kubernetes-version=v1.26.0 --pod-network-cidr=10.224.0.0/16 --apiserver-advertise-address=192.168.1.200 --cri-socket unix:///var/run/cri-dockerd.sock
   # 备注如果要重装集群，先在kubectl删除所有nodes然后用下面的命令
   kubeadm reset --cri-socket unix:///var/run/cri-dockerd.sock
   ```

7. 组装组件

   ```bash
   mkdir -p $HOME/.kube
   cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   chown $(id -u):$(id -g) $HOME/.kube/config
   ```

8. 安装集群网络插件（cni）一般选择calico

   ```bash
   # 下载yaml并直接创建空间
   wget tigera-operator.yaml
   kubectl create -f tigera-operator.yaml
   # 下载后改文件中cidr的网段为pod网段，对应下面步骤6的pod-network-cidr
   wget custom-resources.yaml
   vim custom-resources.yaml
   kubectl create -f custom-resources.yaml
   # 移除污点
   kubectl taint nodes --all node-role.kubernetes.io/control-plane-
   # 查看有没有全部都running
   kubectl get pods -n calico-system
   ```

9. 添加工作节点

   ```bash
   # 查询令牌
   kubeadm token list
   # 如果没有–discovery-token-ca-cert-hash，用这个获取
   openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
   # 如果token全部失效，创建新的
   kubeadm token create --print-join-command
   # 然后用生成的命令去另一台节点上添加
   kubeadm join 10.150.1.15:6443 --token 9x9m6k.iz5j8imrsntg74e9 --discovery-token-ca-cert-hash sha256:3de6aa8f0cbcb42b0b5675493fb504cd261c713300f563b9c0340554325d2851 --cri-socket unix:///var/run/cri-dockerd.sock
   ```

  10. 验证集群状态

      ```bash
      # 必须显示ok
      kubectl get cs
      # 必须全部running
      kubectl get pods -n kube-system
      # 获取服务必须要有集群ip，此时集群还不能对外访问所以externalip为空
      kubectl get svc -n kube-system
      # 尝试解析外网的域名，如果关闭防火墙解析才能成功，请放行端口后再开启防火墙，端口参考六章节
       dig -t a www.baidu.com @<集群ip>
      # 查看路由表是否有kubernetes的组件
      iptables -nL
      ```

11. 添加ingress-nginx以开放集群对外访问

         ```bash
         # 下载1.4.0版本ingress-nginx的yaml，如果下载不了就挂梯子，查看里面的镜像拉取要求
         wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.4.0/deploy/static/provider/cloud/deploy.yaml
         # 从上面的yaml文件中可以知道它要这两个镜像，先pull到docker里，如果是其他版本可能不是这两个
         docker pull registry.k8s.io/ingress-nginx/controller:v1.4.0
         docker pull registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20220916-gd32f8c343
         # 再创建deploy，如果用本地镜像的方式还需要去掉yaml文件内的sha
         kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.4.0/deploy/static/provider/cloud/deploy.yaml
         # 查看是否都ready
         kubectl get pods -n ingress-nginx
         ```

12. 配置ingress-nginx，在clusterIPs下加一行externalIPs填入物理机ip地址

          ```yaml
          apiVersion: v1
          kind: Service
          metadata:
            creationTimestamp: "2022-10-26T05:09:37Z"
            labels:
              app.kubernetes.io/component: controller
              app.kubernetes.io/instance: ingress-nginx
              app.kubernetes.io/name: ingress-nginx
              app.kubernetes.io/part-of: ingress-nginx
              app.kubernetes.io/version: 1.4.0
            name: ingress-nginx-controller
            namespace: ingress-nginx
            resourceVersion: "707709"
            uid: d502fe06-d210-461a-92e8-f667eea50136
          spec:
            allocateLoadBalancerNodePorts: true
            clusterIP: 10.98.171.217
            clusterIPs:
            - 10.98.171.217
            externalIPs:
            - 175.27.159.123
          ```

13. 验证是否有externalip

          ```bash
          kubectl get service
          ```

    
    ​      

### 六、端口

   |  协议   |   端口号    |                作用                 | 必要性 |
   | :-----: | :---------: | :---------------------------------: | :----: |
   |   TCP   |     22      |    ssh远程连接，云服务器默认开启    |   是   |
   |   TCP   |    2376     | 物理机与Docker守护进程通信的TLS端口 |   否   |
   |   TCP   |    2379     |           etcd客户端请求            |   是   |
   |   TCP   |    2380     |            etcd节点通信             |   是   |
   |   TCP   |    9099     |        Canal/Flannel健康检查        |   否   |
   |   TCP   |    9796     |   集群监控拉取节点指标的默认端口    |   否   |
   |   TCP   |    6783     |              Weave端口              |   否   |
   |   TCP   |    10250    |             kubelet APl             |   是   |
   |   TCP   |    10254    |     lngress controller健康检查      |   否   |
   |   TCP   |    6443     |           Kubernetes API            |   是   |
   | TCP/UDP | 30000-32767 |          NodePort端口范围           |   是   |
   |   TCP   |    9443     |          Rancher webhooks           |   是   |
   |   TCP   |     80      |             Rancher节点             |   否   |
   |   TCP   |     443     |             Rancher节点             |   否   |

### 七、yaml创建微服务示例

创建一个nginx+自己app的pod


```yaml
apiVersion: v1			#必选，版本号，例如v1
kind: Pod				#必选，Pod
metadata:				#必选，元数据
  name: string			  #必选，Pod名称
  namespace: string		  #必选，Pod所属的命名空间
  labels:				  #自定义标签
    - name: string		    #自定义标签名字
  annotations:			    #自定义注释列表
    - name: string
spec:					#必选，Pod中容器的详细定义
  containers:			  #必选，Pod中容器列表
  - name: string		    #必选，容器名称
    image: string		    #必选，容器的镜像名称
    imagePullPolicy: [Always | Never | IfNotPresent]	#获取镜像的策略：Alawys表示总是下载镜像，IfnotPresent表示优先使用本地镜像，否则下载镜像，Nerver表示仅使用本地镜像
    command: [string]		#容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]			#容器的启动命令参数列表
    workingDir: string		#容器的工作目录
    volumeMounts:			#挂载到容器内部的存储卷配置
    - name: string			  #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string		  #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean		  #是否为只读模式
    ports:					#需要暴露的端口库号列表
    - name: string			  #端口号名称
      containerPort: int	  #容器需要监听的端口号
      hostPort: int			  #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string		  #端口协议，支持TCP和UDP，默认TCP
    env:					#容器运行前需设置的环境变量列表
    - name: string			  #环境变量名称
      value: string			  #环境变量的值
    resources:				#资源限制和请求的设置
      limits:				  #资源限制的设置
        cpu: string			    #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string			#内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests:				  #资源请求的设置
        cpu: string			    #Cpu请求，容器启动的初始可用数量
        memory: string		    #内存清楚，容器启动的初始可用数量
    livenessProbe:     		#对Pod内个容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket，对一个容器只需设置其中一种方法即可
      exec:					#对Pod容器内检查方式设置为exec方式
        command: [string]	  #exec方式需要制定的命令或脚本
      httpGet:				#对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:			#对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0	#容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0		#对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0			#对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged:false
    restartPolicy: [Always | Never | OnFailure]		#Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Nerver表示不再重启该Pod
    nodeSelector: obeject		#设置NodeSelector表示将该Pod调度到包含这个label的node上，以key：value的格式指定
    imagePullSecrets:			#Pull镜像时使用的secret名称，以key：secretkey格式指定
    - name: string
    hostNetwork:false			#是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
    volumes:					#在该pod上定义共享存储卷列表
    - name: string				  #共享存储卷名称 （volumes类型有很多种）
      emptyDir: {}				  #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: string			  #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string			    #Pod所在宿主机的目录，将被用于同期中mount的目录
      secret:					#类型为secret的存储卷，挂载集群与定义的secre对象到容器内部
        scretname: string  
        items:     
        - key: string
          path: string
      configMap:				#类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string

```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mcserver
  namespace: mc
  labels:
    name: mcserver
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
    volumeMounts:
      - name: vol-nginx-data
        mountPath: /usr/local/etc/nginx
  volumes:
    - name: vol-nginx-data
      hostPath:
        path: /usr/nginx-config
```

