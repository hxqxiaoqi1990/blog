---
title: k8s之ingress-nginx
tags: [k8s]
categories: kubernetes
date: 2020-07-09 15:06:00
---

# 介绍

**Ingress是什么？**

ingress 是除了 hostport  nodeport  clusterIP以及云环境专有的负载均衡器外的访问方式,官方提供了Nginx ingress controller。ingress-nginx本身就是nodeport模式

 

**Ingress能做什么？**

k8s中，不管是哪种类型的svc，不管是用iptables还是ipvs实现端口转发实现负载均衡，也只是实现了四层的负载均衡，但是，如果有需求要进行七层负载均衡呢？比如你想将你的网站设置为https呢？Ingress就是来帮你解决此问题的。

 

**Ingress工作原理及主要组成部分？**

工作原理：

类似于Nginx，可以理解为在Ingress建立一个个映射规则，Ingress Controller通过监听Ingress这个api对象里的规则并转化为Nginx/HAporxy等的配置，然后对外部提供服务。

组成部分：

ingress controller：

核心是一个deployment，实现方式有很多种，比如Nignx、HAproxy、trafik(配置教程：https://zhuanlan.zhihu.com/p/105239989)、lstio，需要编写的yaml有：Deployment、Service、ConfigMap、ServiceAccount(Auth)，其中Service类型可以是NodePort或者LoadBalance

ingress resources：这个是类型为ingress的k8s api对象，主要面向开发人员。

# nginx-ingress-controller yaml部署

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx
---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      # wait up to five minutes for the drain of connections
      hostNetwork: true
      terminationGracePeriodSeconds: 300
      serviceAccountName: nginx-ingress-serviceaccount
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: nginx-ingress-controller
          image: bitnami/nginx-ingress-controller:0.26.2
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
            - name: https
              containerPort: 443
              protocol: TCP
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown
```

# ingress 配置

## 示例一：暴露域名

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress1
spec:
  rules:
  - host: www1.wuzi.com
    http:
      paths:
      - path: /
        backend:
          serviceName: ingress-svc1
          servicePort: 80
```

## 示例二：暴露ip

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gw
  namespace: b2b-test
  annotations:
    kubernetes.io/ingress.class: "nginx"
    # 地址重写
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - http:
      paths:
      # 绑定的svc内容。可以配置多个
        - backend:
            serviceName: mmhsy-gateway
            servicePort: 9900
          path: /b2b-gw(/|$)(.*)
        - backend:
            serviceName: nginx
            servicePort: 80
          path: /
```

## 示例三：https配置

```bash
## 创建私钥key
openssl genrsa -des3 -out server.key 2048
## 创建csr请求
openssl req -new -key server.key -out server.csr
## 去除私钥的连接密码
cp server.key{,.org}
openssl rsa -in server.key.org -out server.key
## 生成证书文件
openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt
## 生成tls格式
kubectl create secret tls tls-secret --key server.key --cert server.crt
```

ingress引用tls证书

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-https
spec:
  tls:
  - hosts:
    - www3.wuzi.com
    secretName: tls-secret
  rules:
  - host: www3.wuzi.com
    http:
      paths:
      - path: /
        backend:
          serviceName: ingress-https
          servicePort: 80
```

## 示例四：重写

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: https://www3.wuzi.com:31087
  name: rewrite
  namespace: default
spec:
  rules:
  - host: www5.wuzi.com
    http:
      paths:
      - backend:
          serviceName: ingress-svc1
          servicePort: 80
        path: /
```

