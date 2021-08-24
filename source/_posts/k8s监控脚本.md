---
title: k8s监控脚本
tags: [python脚本]
categories: 脚本
date: 2020-07-04 09:06:00
---

```bash
#!/usr/bin/python3
# -*- coding: UTF-8 -*-
# 获取k8s集群基本信息，根据kubernets模块提供的接口
from kubernetes import client, config
from collections import Counter
import requests
import datetime

config.kube_config.load_kube_config(config_file="/var/www/html/cgi-bin/b2b-kubeconfig.yaml")

# 选择名称空间
clusername = "b2b"

#获取API的CoreV1Api版本对象
v1 = client.CoreV1Api()
k8snamespaces = v1.list_pod_for_all_namespaces(watch=False)
k8snamespacessvc = v1.list_namespaced_service(namespace=clusername)

nodename = {"192.168.1.10":"(外网IP1#k1)","192.168.1.10":"(外网IP2#k2)","192.168.1.10":"(外网IP3#k3)"}

def k8sinfo(podname):
    #集群接口信息
    k8spodinfo = v1.read_namespaced_pod(namespace=clusername,name=podname)

    # 获取pod信息
    print("<tr>")
    if k8spodinfo.status.pod_ip != None:
        global podip
        podip = k8spodinfo.status.pod_ip
        print("<td bgcolor='e8e8e8'>&nbsp;%s<font size='1' color='#575757'>%s</font</td>" % (podip,nodename[k8spodinfo.status.host_ip]))
    else:
        print("<td bgcolor='e8e8e8' align='center'>None</td>")
        
    print("<td bgcolor='e8e8e8'>&nbsp;%s</td>" % (podname))
    print("<td bgcolor='e8e8e8'>&nbsp;%s </td>" % (k8spodinfo.status.host_ip))
    
    if k8spodinfo.spec.containers[0].ports[0].container_port != None:
        k8spodport = k8spodinfo.spec.containers[0].ports[0].container_port
        print("<td bgcolor='e8e8e8' align='center'>%s</td>" % (k8spodport))
    else:
        print("<td bgcolor='e8e8e8' align='center'>None</td>")
    
    # 获取pod运行状态
    k8sstatus = k8spodinfo.status.container_statuses[0].ready
    if k8sstatus == True:
        print("<td bgcolor='e8e8e8' align='center'><font color='green'>&nbsp;True</font></td>")
    elif k8sstatus == False:
        print("<td bgcolor='e8e8e8' align='center'><font color='red'>&nbsp;False</font></td>")
    else:
        print("<td bgcolor='e8e8e8' align='center'><font color='red'>&nbsp;Error</font></td>")
    
    # 健康检查
    if k8spodinfo.spec.containers[0].readiness_probe != None:
        if k8spodinfo.spec.containers[0].readiness_probe.http_get != None:
            checkurl = k8spodinfo.spec.containers[0].readiness_probe.http_get
            url="http://"+podip+":"+str(k8spodport)+checkurl.path
        
            try:
                request = requests.get(url,timeout=2.2)
                httpStatusCode = request.status_code
            except requests.exceptions.ConnectTimeout:
                httpStatusCode = 502
            except requests.exceptions.Timeout:
                httpStatusCode = 503
            except requests.exceptions.ConnectionError:
                httpStatusCode=504
    
            if httpStatusCode == 200:
                print("<td bgcolor='e8e8e8' align='center'><font color='green'>"+str(httpStatusCode)+"</font></td>")
            else:
                print("<td bgcolor='e8e8e8' align='center'><font color='red'>"+str(httpStatusCode)+"</font></td>")
        else:
            print("<td bgcolor='e8e8e8' align='center'>none</td>")
    else:
        print("<td bgcolor='e8e8e8' align='center'>none</td>")
        
    #版本
    if k8spodinfo.metadata.annotations['aliyun.kubernetes.io/deploy-timestamp'] != None:
        k8spodversion = k8spodinfo.metadata.annotations['aliyun.kubernetes.io/deploy-timestamp']
        k8spodversion1 = k8spodversion.split("T")[0]+" "+k8spodversion.split("T")[1].split("Z")[0]
        k8spodversion2 = datetime.datetime.strptime(k8spodversion1,"%Y-%m-%d %H:%M:%S")
        add = datetime.timedelta(hours=8)
        k8spodversion = (k8spodversion2+add).strftime("%Y-%m-%d %H:%M:%S")
        print("<td bgcolor='e8e8e8' align='center'>%s</td>" % (k8spodversion))
    else:
        print("<td bgcolor='e8e8e8' align='center'>None</td>")
   
    
    # 重启次数
    k8srestart = k8spodinfo.status.container_statuses[0].restart_count
    print("<td bgcolor='e8e8e8' align='center'>%s</td>" % (k8srestart))
    
    #最后重启时间
    if k8spodinfo.status.container_statuses[0].state.running != None:
        restarttime = k8spodinfo.status.container_statuses[0].state.running.started_at
        restarttime1 = str(restarttime).split("+")[0]
        restarttime2 = datetime.datetime.strptime(restarttime1,"%Y-%m-%d %H:%M:%S")
        addtime = datetime.timedelta(hours=8)
        restarttime = (restarttime2+addtime).strftime("%Y-%m-%d %H:%M:%S")
        print("<td bgcolor='e8e8e8' align='center'>%s</td>" % (restarttime))
    else:
        print("<td bgcolor='e8e8e8' align='center'>None</td>")
    
    print("</tr>")
 
# 获取pod名，去重，相同pod打印同一表格内
def podinfo():
    podapp = []
    podlist = []
    for i in k8snamespaces.items:
        if i.metadata.namespace == clusername:
            podapp.append(i.metadata.labels['app'])
            podlist.append(i.metadata.name)
    b = dict(Counter(podapp))
    c = [key for key, value in b.items() if value > 0]

    for y in c:
        for cluster in k8snamespacessvc.items:
            if cluster.spec.selector['app'] == y:
                cluster_ip = cluster.spec.cluster_ip
                cluster_port = cluster.spec.ports[0].port
                
        #num = 0
        print("<b><font size=4>%s</font></b>" % (y))
        print('<table border=1 cellspacing=0><tr style="color: #ffffff" bgcolor="#444444" align="center"><td width=220px><b>Container IP</b></td><td width=360px><b>&nbsp;Name</b></td><td width=150px><b>node IP</b></td><td width=62px><b>&nbsp;Port</b></td><td width=80px><b>Status</b></td><td><b><font size=1>UrlCheck(2.2s)</font></b></td><td width=130px><b>Version</b></td><td><b><font size=1>Refresh_Conf</font></b></td><td><b>Log</b></td><td>restart</td><td>restarttime</td></tr>')
        for x in podlist:
            if x.startswith(y):
                #num = num+1
                k8sinfo(x)
        print ('</table>')
        print("<br><br>")
        
podinfo()
```

