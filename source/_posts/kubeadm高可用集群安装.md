---
title: kubeadm高可用集群安装
tags: [k8s]
categories: kubernetes
date: 2021-08-20 15:06:00
---

# 介绍

运行以下脚本`init.sh`，初始化环境，主要修改版本号与主机名

```bash
#!/bin/bash

# docker版本，k8s版本
DockerV='-18.09.9'
K8sV='-1.19.9'

echo "设置主机名"
cat >> /etc/hosts <<EOF
192.168.40.100 master0
192.168.40.101 master1
192.168.40.102 master2
192.168.40.200 work0
EOF

#yum源

yum install -y yum-utils device-mapper-persistent-data lvm2 epel-release vim screen bash-completion mtr lrzsz wget telnet zip unzip sysstat  ntpdate libcurl openssl bridge-utils nethogs dos2unix iptables-services 

wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

wget http://mirrors.aliyun.com/repo/epel-7.repo -O /etc/yum.repos.d/epel.repo

cat >>/etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

#安装K8S组件
yum install -y kubelet${K8sV} kubeadm${K8sV} kubectl${K8sV} docker-ce${DockerV}

systemctl enable kubelet

mkdir -p /etc/docker

tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://pthx0mbz.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

systemctl restart docker && systemctl enable docker

#禁用防火墙与selinux

setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
sed -i 's/SELINUX=permissive/SELINUX=disabled/g' /etc/selinux/config

service firewalld stop
systemctl disable firewalld.service
service iptables stop
systemctl disable iptables.service

service postfix stop
systemctl disable postfix.service


wget http://mirrors.aliyun.com/repo/epel-7.repo -O /etc/yum.repos.d/epel.repo

note='#Ansible: nptdate-time'
task='*/10 * * * * /usr/sbin/ntpdate -u ntp.sjtu.edu.cn &> /dev/null'
echo "$(crontab -l)" | grep "^${note}$" &>/dev/null || echo -e "$(crontab -l)\n${note}" | crontab -
echo "$(crontab -l)" | grep "^${task}$" &>/dev/null || echo -e "$(crontab -l)\n${task}" | crontab -

echo '/etc/security/limits.conf 参数调优，需重启系统后生效'

cp -rf /etc/security/limits.conf /etc/security/limits.conf.back

cat > /etc/security/limits.conf << EOF
* soft nofile 655350
* hard nofile 655350
* soft nproc unlimited
* hard nproc unlimited
* soft core unlimited
* hard core unlimited
root soft nofile 655350
root hard nofile 655350
root soft nproc unlimited
root hard nproc unlimited
root soft core unlimited
root hard core unlimited
EOF

echo '/etc/sysctl.conf 文件调优'

cp -rf /etc/sysctl.conf /etc/sysctl.conf.back
cat > /etc/sysctl.conf << EOF

vm.swappiness = 0
net.ipv4.neigh.default.gc_stale_time = 120

# see details in https://help.aliyun.com/knowledge_detail/39428.html
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_announce = 2

# see details in https://help.aliyun.com/knowledge_detail/41334.html
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_synack_retries = 2

net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1

kernel.sysrq = 1
kernel.pid_max=1000000

vm.swappiness=0
EOF
sysctl -p

echo "ipvs模块开启"
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

echo "1">/proc/sys/net/bridge/bridge-nf-call-iptables

chmod 755 /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules

lsmod | grep -e ip_vs -e nf_conntrack_ipv4

echo "禁用swap"
swapoff -a
sed -i '/swap/d' /etc/fstab
```

如果需要安装高可用集群，运行以下脚本，`install.sh`，该脚本安装keepalived与nginx，变量根据情况修改，如果单节点，可以不执行该脚本

```bash
#!/bin/bash

# 定义变量
master0="192.168.40.100:6443"	#k8s master ip
master1="192.168.40.101:6443"
master2="192.168.40.102:6443"
vip="192.168.40.201"	# keepalived vip地址
role=MASTER				# keepalived的角色，MASTER或BACKUP
network=ens33			# 本机网卡名
router_id=LVS_1			# keepalived本机名，一般是主机名，也可以自己定义
priority=100			# keepalived权重
port=7443				# nginx代理监听端口
nginx_dir=/etc			# nginx安装目录

#安装依赖
yum -y install gcc gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel wget

#下载nginx
wget http://nginx.org/download/nginx-1.14.2.tar.gz

#解压并安装
tar xf ./nginx-1.14.2.tar.gz
cd nginx-1.14.2/
./configure --prefix=$nginx_dir/nginx --user=root --group=root --with-http_ssl_module --with-http_stub_status_module --with-http_gzip_static_module --with-http_realip_module --with-stream
make && make install
[ ! -d /var/log/nginx ] && mkdir /var/log/nginx

cat >/etc/nginx/conf/nginx.conf <<EOF
user root;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
  worker_connections 1024;
}
stream {
  log_format proxy '\$remote_addr \$remote_port - [\$time_local] \$status \$protocol '
  '"\$upstream_addr" "\$upstream_bytes_sent" "\$upstream_connect_time"' ;
  
  access_log /var/log/nginx/nginx-proxy.log proxy;
  
  upstream kubernetes_lb{
    server $master0 weight=5 max_fails=3 fail_timeout=30s;
    server $master1 weight=5 max_fails=3 fail_timeout=30s;
    server $master2 weight=5 max_fails=3 fail_timeout=30s;
  }
  
  server {
    listen $port;
    proxy_connect_timeout 30s;
    proxy_timeout 30s;
    proxy_pass kubernetes_lb;
  }
}
EOF

/etc/nginx/sbin/nginx

yum -y install keepalived
systemctl start keepalived && systemctl enable keepalived

cat >/etc/keepalived/keepalived.conf <<EOF
global_defs {
  router_id $router_id
}
vrrp_instance VI_1 {
  state $role
  interface $network  #网卡设备名称，根据自己网卡信息进行更改
  lvs_sync_daemon_inteface $network 
  virtual_router_id 100
  advert_int 1
  priority $priority
  authentication {
    auth_type PASS
    auth_pass 1111
}
  virtual_ipaddress {
    $vip  # 这就就是虚拟IP地址
  }
}
EOF

systemctl restart keepalived
```

创建：kubeadm-init.yaml，添加以下内容

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.19.9
apiServer:
controlPlaneEndpoint: "192.168.40.201:7443"
networking:
  dnsDomain: cluster.local
  podSubnet: "10.244.0.0/16"
  serviceSubnet: 10.96.0.0/12
imageRepository: registry.aliyuncs.com/google_containers
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 0s
  scheduler: "rr"
  strictARP: false
  syncPeriod: 30s
```

执行初始化k8s集群命令

```bash
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile

kubeadm init --config kubeadm-init.yaml
```

如果失败可以以执行重置

```bash
kubeadm reset
```

执行成功会出现添加集群的命令，需要记下，以方便添加新master和node。

创建文件：kube-flannel.yml，添加以下内容

```yaml
cat > kube-flannel.yml << EOF
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.flannel.unprivileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
    seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
    apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
    apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
spec:
  privileged: false
  volumes:
  - configMap
  - secret
  - emptyDir
  - hostPath
  allowedHostPaths:
  - pathPrefix: "/etc/cni/net.d"
  - pathPrefix: "/etc/kube-flannel"
  - pathPrefix: "/run/flannel"
  readOnlyRootFilesystem: false
  # Users and groups
  runAsUser:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  # Privilege Escalation
  allowPrivilegeEscalation: false
  defaultAllowPrivilegeEscalation: false
  # Capabilities
  allowedCapabilities: ['NET_ADMIN', 'NET_RAW']
  defaultAddCapabilities: []
  requiredDropCapabilities: []
  # Host namespaces
  hostPID: false
  hostIPC: false
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  # SELinux
  seLinux:
    # SELinux is unused in CaaSP
    rule: 'RunAsAny'
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups: ['extensions']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames: ['psp.flannel.unprivileged']
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
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
  net-conf.json: |
    {
      "Network": "10.200.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: jmgao1983/flannel
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: jmgao1983/flannel
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
EOF
```

执行命令

```bash
kubectl apply -f kube-flannel.yml
```

执行以下命令，解决组件健康检查报错问题

```bash
sed -i '/--port=0/d' /etc/kubernetes/manifests/kube-controller-manager.yaml
sed -i '/--port=0/d' /etc/kubernetes/manifests/kube-scheduler.yaml
systemctl restart kubelet
```

新增master

```bash
USER=root
CONTROL_PLANE_IPS="master1 master2"
for host in ${CONTROL_PLANE_IPS}; do
ssh "${USER}"@$host "mkdir -p /etc/kubernetes/pki/etcd"
scp /etc/kubernetes/pki/ca.* "${USER}"@$host:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.* "${USER}"@$host:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.* "${USER}"@$host:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/etcd/ca.* "${USER}"@$host:/etc/kubernetes/pki/etcd/
scp /etc/kubernetes/admin.conf "${USER}"@$host:/etc/kubernetes/
done
```

添加web管理界面

```bash
kubectl apply -f https://kuboard.cn/install-script/kuboard.yaml
kubectl apply -f https://addons.kuboard.cn/metrics-server/0.3.7/metrics-server.yaml
echo $(kubectl -n kube-system get secret $(kubectl -n kube-system get secret | grep kuboard-user | awk '{print $1}') -o go-template='{{.data.token}}' | base64 -d)
```

命令补全

```bash
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

