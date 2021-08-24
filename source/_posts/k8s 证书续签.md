---
title: k8s证书续签
tags: [k8s]
categories: kubernetes
date: 2021-05-18 14:54:00
---



# 介绍

安装k8s集群时，kubelet组件默认是1年，如果到期，需要手动更新该证书，1.8版本之后也可配置自动更新。

## 手动更新

删除原证书

```bash
# 进入到ssl配置目录，删除 kubelet 证书
rm -f kubelet-client-current.pem kubelet-client-2019-05-10-09-57-21.pem kubelet.key kubelet.crt
```

重启kubelet

```bash
systemctl restart kubelet && systemctl status kubelet
```

查看csr证书请求

```bash
kubectl get csr
```

手动签发

```bash
kubectl certificate approve node-csr--k3G2G1EoM4h9w1FuJRjJjfbIPNxa551A8TZfW9dG-g
```

验证

```bash
kubectl get csr
kubectl get nodes
```

## 自动更新

添加kubelet组件参数，也就是启动时的参数：

```bash
# 自动重载证书
--rotate-certificates
```

重启服务

```bash
systemctl restart kubelet
systemctl status kubelet
```

添加kube-controller-manager组件参数，也就是启动时的参数：

```bash
# 自动允许签发证书
--feature-gates=RotateKubeletServerCertificate=true
```

重启服务

```bash
systemctl restart kube-controller-manager
systemctl status kube-controller-manager
```

创建自动批准相关 CSR 请求的 ClusterRole

```bash
cat >> tls-instructs-csr.yaml << EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeserver
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/selfnodeserver"]
  verbs: ["create"]
EOF

kubectl apply -f tls-instructs-csr.yaml

# 自动批准 kubelet-bootstrap 用户 TLS bootstrapping 首次申请证书的 CSR 请求
kubectl create clusterrolebinding node-client-auto-approve-csr --clusterrole=system:certificates.k8s.io:certificatesigningrequests:nodeclient --user=kubelet-bootstrap

# 自动批准 system:nodes 组用户更新 kubelet 自身与 apiserver 通讯证书的 CSR 请求
kubectl create clusterrolebinding node-client-auto-renew-crt --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeclient --group=system:nodes

# 自动批准 system:nodes 组用户更新 kubelet 10250 api 端口证书的 CSR 请求
kubectl create clusterrolebinding node-server-auto-renew-crt --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeserver --group=system:nodes
```

重启docker

```bash
systemctl daemon-reload
systemctl status kube-controller-manager
```

测试

```bash
# 进入到ssl配置目录，删除 kubelet 证书
rm -f kubelet-client-current.pem kubelet-client-2019-05-10-09-57-21.pem kubelet.key kubelet.crt

# 重启启动，启动正常后会颁发有效期10年的ssl证书
systemctl restart kubelet

# 进入到ssl配置目录，查看证书有效期
openssl x509 -in kubelet-client-current.pem -noout -text | grep "Not"
```

