# toolbox
我自己常用的一些脚本

``

远程节点安装k8s

https://zhuanlan.zhihu.com/p/410371256

    # 更新yum 安装工具
    sudo yum update -y
    sudo yum install -y yum-utils
    
    # 主节点master节点执行
    echo "设置主节点主机名" > /dev/null
    hostnamectl set-hostname k8s-master
    
    echo "停止、关闭防火墙" > /dev/null
    systemctl stop firewalld
    systemctl disable firewalld
    
    echo "永久关闭SELinux" > /dev/null
    sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
    echo  "临时关闭SELinux" > /dev/null
    setenforce 0
    echo "查看SELinux状态，显示为[SELinux status:                 disabled]视为设置成功" > /dev/null
    sestatus
    
    echo "永久关闭Swap" > /dev/null
    sed -ri 's/.*swap.*/#&/' /etc/fstab
    echo "临时关闭Swap" > /dev/null
    swapoff -a
    
    echo "安装、配置时间同步" > /dev/null
    yum install ntpdate -y
    ntpdate time.windows.com

    echo "创建/etc/sysctl.d/k8s.conf配置文件，将桥接的 IPv4 流量传递到 iptables 的链" > /dev/null
    cat > /etc/sysctl.d/k8s.conf << EOF 
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1 
    EOF
    
    
    # 追加host
    cat >> /etc/hosts << EOF 
    <主节点公网IP> k8s-master
    <工作节点公网IP> k8s-node1
    EOF

    echo "使host配置生效" > /dev/null
    /etc/init.d/network restart
    
    # 安装docker
    yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
    
    yum install docker-ce docker-ce-cli containerd.io -y
    
    # 查看docker版本
    docker --version
    
    # 重启Docker,避免安装了没有启动
    systemctl restart docker
    
    # 设置开机自启
    systemctl enable docker.service
    
    安装 k8s管理工具
    docker run -d \
    --restart=unless-stopped \
    --name=kuboard \
    -p 80:80/tcp \
    -p 10081:10081/tcp \
    -e KUBOARD_ENDPOINT="http://内网IP:80" \
    -e KUBOARD_AGENT_SERVER_TCP_PORT="10081" \
    -v /root/kuboard-data:/data \
    eipwork/kuboard:v3
    
    
    # 添加yum源，并配置为阿里云源，国内下载更快。
    https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
    
    echo "安装Kubeadm、Kubelet、Kubectl" > /dev/null
    yum install -y kubelet kubeadm kubectl -y
    echo "设置开机启动" > /dev/null
    systemctl enable kubelet
    
    # 集群初始化（主）
    
    # 通过kubeadm执行初始化，apiserver-advertise-address填主节点的公网IP。service-cidr是service网段。pod-network-cidr是Pod网段。
    kubeadm init \
    --apiserver-advertise-address=<公网IP> \
    --image-repository registry.aliyuncs.com/google_containers \
    --kubernetes-version v1.18.0 \
    --service-cidr=10.96.0.0/12 \
    --pod-network-cidr=10.244.0.0/16
    
    # 报错 “Failed to run kubelet” err=“failed to run Kubelet: misconfiguration: kubelet cgroup driver: “systemd” is different from dock…er: “cgroupfs””
    https://blog.csdn.net/skyroach/article/details/118325866
    
    # Kubernetes(K8s) 使用kubeadm reset重置后 kubeadm init 失败的解决方法
    https://www.cjavapy.com/article/2392/
    
    
    kubeadm init \
    --apiserver-advertise-address=139.59.243.125 \
    --image-repository registry.aliyuncs.com/google_containers \
    --kubernetes-version v1.23.4 \
    --service-cidr=10.96.0.0/12 \
    --pod-network-cidr=10.244.0.0/16
    
    # 执行完上面的初始化脚本后会卡在 etcd 初始化的位置，因为 etcd 绑定端口的时候使用外网 IP，而云服务器外网 IP 并不是本机的网卡，而是网关分配的一个供外部访问的 IP，从而导致初始化进程一直重试绑定，长时间卡住后失败。
    # 解决办法是在卡住时，另启一个命令行窗口修改初始化生成的 etcd.yaml 中的配置成示例图一样，进程会自动重试并初始化成功。
    vim /etc/kubernetes/manifests/etcd.yaml
    
    Kubectl设置。kubeadm初始化完成后，控制台会打印下面一模一样的指令，直接执行。
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
    
    # kubeadm init 后master一直处于notready状态
    https://blog.csdn.net/wangmiaoyan/article/details/101216496
    
    
    工作节点加入集群（工）-Centos系统
    # 主节点通过kubeadm初始化成功后控制台会打印出join脚本，复制到从节点执行。
    # 如果忘记了join脚本，可以在主节点执行以下命令重新生成。
    kubeadm token create --print-join-command
    
    echo "设置从节点主机名" > /dev/null
    hostnamectl set-hostname k8s-node1
    
    echo "停止、关闭防火墙" > /dev/null
    systemctl stop firewalld
    systemctl disable firewalld
    
    echo "永久关闭SELinux" > /dev/null
    sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
    echo  "临时关闭SELinux" > /dev/null
    setenforce 0
    echo "查看SELinux状态，显示为[SELinux status:                 disabled]视为设置成功" > /dev/null
    sestatus
    
    echo "永久关闭Swap" > /dev/null
    sed -ri 's/.*swap.*/#&/' /etc/fstab
    echo "临时关闭Swap" > /dev/null
    swapoff -a
    
    echo "安装、配置时间同步" > /dev/null
    yum install ntpdate -y
    ntpdate time.windows.com

    echo "创建/etc/sysctl.d/k8s.conf配置文件，将桥接的 IPv4 流量传递到 iptables 的链" > /dev/null
    cat > /etc/sysctl.d/k8s.conf << EOF 
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1 
    EOF
    
    # 安装docker
    yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
    
    yum install docker-ce docker-ce-cli containerd.io -y
    
    # 查看docker版本
    docker --version
    
    # 重启Docker,避免安装了没有启动
    systemctl restart docker
    
    # 设置开机自启
    systemctl enable docker.service
    
    
    # 解决：k8s[kubelet-check] The HTTP call equal to ‘curl -sSL http://localhost:10248/healthz’ failed with
   
    https://blog.csdn.net/qq_29349143/article/details/120872330
    
    # 解决kubeadm init /proc/sys/net/bridge/bridge-nf-call-iptables contents are not set to 1
    https://developer.aliyun.com/article/701252
    
    kubectl  delete nodes k8s-node1
    
    kubectl get nodes
    
    
    

    
    
    
    
    
