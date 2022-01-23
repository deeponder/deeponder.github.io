---
layout:     post
title:      KubeSphere运维
subtitle:   
date:       2022-1-23
author:     jabin
header-img: 
catalog: true
tags:
    - 计算机
    - KubeSphere
    - k8s
---


# 部署方式
1. kubeadm 部署k8s集群， 在K8s集群上直接部署ks。 直接官方kubeadm比较繁琐，且国内有访问问题
2. kubekey直接在裸linux机器上部署k8s和ks。 这里采用kk进行安装，底层也是基于kubeadm

# kk源码走读
https://github.com/kubesphere/kubekey
以`./kk create cluster` 为例
1. 入口。 Cmd/main.go -> NewKubeKeyCommand
2. op封装。 Cmd/op/xx
3. 创建集群命令。 NewCmdCreateCluster->CreateClusterOptions.Run
4. 执行创建流水线。 Piplines.CreateCluster->NewCreateClusterPipeline
5. 流水线要执行的模块。 m:=[]module.Module。包括InstallKubeBinariesModule、InitKubernetesModule、DeployNetworkPluginModule等
6. 遍历模块，执行模块对应的Init、Run等操作。
7. 继承BaseTaskModule， 每个模块定义N个要执行的Task,  流水线执行Run的操作时，会执行每个任务， 比如GenerateKubeadmConfig、KubeadmInit等

# 基于kk集群部署

## 环境
两台CentOS 7

## 重置k8s集群
Kubadm reset

## 安装
root用户操作
### 版本
为了后续验证升级，这里K8s: v1.19.8， Ks: v3.1.1

### 下载kk
```
export KKZONE=cn


curl -sfL https://get-kk.kubesphere.io | VERSION=v1.2.1 sh -

chmod +x kk

```

### 集群配置文件
- 创建
```
./kk create config --with-kubernetes v1.19.8 --with-kubesphere v3.1.1

```

- 配置文件说明
```
# 完整的配置文件说明：https://github.com/kubesphere/kubekey/blob/release-1.2/docs/config-example.md
apiVersion: kubekey.kubesphere.io/v1alpha1
kind: Cluster
metadata:
  name: sample
spec:
  hosts:    #集群节点的详细信息
  - {name: node1, address: 172.16.0.2, internalAddress: 172.16.0.2, user: ubuntu, password: Qcloud@123}
  - {name: node2, address: 172.16.0.3, internalAddress: 172.16.0.3, user: ubuntu, password: Qcloud@123}
  roleGroups:  # master、worker、etcd所在的机器节点
    etcd:
    - node1
    master:
    - node1
    worker:
    - node1
    - node2
  controlPlaneEndpoint:   # 高可用配置  todo
    ##Internal loadbalancer for apiservers
    #internalLoadbalancer: haproxy

    #domain: lb.kubesphere.local
    #address: ""
    #port: 6443
  kubernetes:
    version: v1.19.8
    clusterName: cluster.local
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
  registry:
    registryMirrors: []
    insecureRegistries: []
  addons: []

```

- [高可用集群配置](https://kubesphere.com.cn/docs/installing-on-linux/high-availability-configurations/ha-configuration/)。 todo

### 创建集群
```
# 等待40来分钟
./kk create cluster -f config-sample.yaml

```

### 完成安装
- console
```
Console: http://xxxx:30880
Account: admin
Password: P@88w0rd

NOTES：
  1. After you log into the console, please check the
     monitoring status of service components in
     "Cluster Management". If any service is not
     ready, please wait patiently until all components 
     are up and running.
  2. Please change the default password after login.
```

- 版本信息
```
# k8s  v1.19.8
[root@zhipeng2 kubekey]# kubectl version
Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.8", GitCommit:"fd5d41537aee486160ad9b5356a9d82363273721", GitTreeState:"clean", BuildDate:"2021-02-17T12:41:51Z", GoVersion:"go1.15.8", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.8", GitCommit:"fd5d41537aee486160ad9b5356a9d82363273721", GitTreeState:"clean", BuildDate:"2021-02-17T12:33:08Z", GoVersion:"go1.15.8", Compiler:"gc", Platform:"linux/amd64"}

# ks v3.1.1
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  30m   default-scheduler  Successfully assigned kubesphere-system/ks-console-58b965dbf5-nzfcl to node1
  Normal  Pulling    30m   kubelet            Pulling image "registry.cn-beijing.aliyuncs.com/kubesphereio/ks-console:v3.1.1"
  Normal  Pulled     29m   kubelet            Successfully pulled image "registry.cn-beijing.aliyuncs.com/kubesphereio/ks-console:v3.1.1" in 50.978835982s
  Normal  Created    29m   kubelet            Created container ks-console
  Normal  Started    29m   kubelet            Started container ks-console
[root@zhipeng2 kubekey]# kubectl describe pod ks-console-58b965dbf5-nzfcl -n kubesphere-system

```

### 多集群配置（可选）
1. https://kubesphere.com.cn/docs/multicluster-management/
2. https://github.com/kubesphere/community/blob/master/sig-multicluster/how-to-setup-multicluster-on-kubesphere/README_zh.md

## 升级
### 版本
K8s:  1.20.4, Ks: v3.2.1

### 生成配置文件
```
./kk create config --from-cluster

apiVersion: kubekey.kubesphere.io/v1alpha1
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  ##You should complete the ssh information of the hosts
  - {name: node1, address: xx, internalAddress: xx, user: root, password: xx}
  - {name: node2, address: xx, internalAddress: xx, user: root, password: xx}
  roleGroups:
    etcd:
    - node1
    master:
    - node1
    worker:
    - node2
  controlPlaneEndpoint:
    ##Internal loadbalancer for apiservers
    #internalLoadbalancer: haproxy

    ##If the external loadbalancer was used, 'address' should be set to loadbalancer's ip.
    domain: lb.kubesphere.local
    address: ""
    port: 6443
  kubernetes:
    version: v1.19.8
    clusterName: cluster.local
    proxyMode: ipvs
    masqueradeAll: false
    maxPods: 110
    nodeCidrMaskSize: 24
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
  registry:
    privateRegistry: ""
```

### 单独升级K8S
```
./kk upgrade --with-kubernetes v1.20.4

# 升级后的版本
[root@node1 ~]# kubectl version
Client Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.4", GitCommit:"e87da0bd6e03ec3fea7933c4b5263d151aafd07c", GitTreeState:"clean", BuildDate:"2021-02-18T16:12:00Z", GoVersion:"go1.15.8", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.4", GitCommit:"e87da0bd6e03ec3fea7933c4b5263d151aafd07c", GitTreeState:"clean", BuildDate:"2021-02-18T16:03:00Z", GoVersion:"go1.15.8", Compiler:"gc", Platform:"linux/amd64"}

# 服务正常， productpage扩容
[root@node1 kubekey]# kubectl get pod -n bookinfo
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-b4d8db7f-vpddb         1/1     Running   0          135m
productpage-v1-59ffbd7657-6f5ln   1/1     Running   0          135m
productpage-v1-59ffbd7657-qjgzj   1/1     Running   0          30s
ratings-v1-6cc556ccfb-qzsbg       1/1     Running   0          141m
reviews-v1-7c95bc9454-qfqv4       1/1     Running   0          141m

```

### 升级KS

- 升级指令 `./kk upgrade --with-kubesphere v3.2.1 -f sample.yaml`
- 升级失败
```
失败1：notification-manager升级报错
"runner_ident": "monitoring",
  "start_line": 114,
  "stdout": "fatal: [localhost]: FAILED! => {\"attempts\": 3, \"changed\": true, \"cmd\": \"/usr/local/bin/helm upgrade --install notification-manager /kubesphere/kubesphere/notification-manager -f /kubesphere/kubesphere/notification-manager/custom-values-notification.yaml -n kubesphere-monitoring-system --force\\n\", \"delta\": \"0:00:01.111866\", \"end\": \"2022-01-11 20:08:27.609767\", \"msg\": \"non-zero return code\", \"rc\": 1, \"start\": \"2022-01-11 20:08:26.497901\", \"stderr\": \"Error: UPGRADE FAILED: failed to replace object: Service \\\"notification-manager-controller-metrics\\\" is invalid: spec.clusterIPs[0]: Invalid value: []string(nil): primary clusterIP can not be unset && failed to replace object: Service \\\"notification-manager-webhook\\\" is invalid: spec.clusterIPs[0]: Invalid value: []string(nil): primary clusterIP can not be unset\", \"stderr_lines\": [\"Error: UPGRADE FAILED: failed to replace object: Service \\\"notification-manager-controller-metrics\\\" is invalid: spec.clusterIPs[0]: Invalid value: []string(nil): primary clusterIP can not be unset && failed to replace object: Service \\\"notification-manager-webhook\\\" is invalid: spec.clusterIPs[0]: Invalid value: []string(nil): primary clusterIP can not be unset\"], \"stdout\": \"\", \"stdout_lines\": []}",


失败2：登录失败报错
  <-- GET / 2022/01/11T20:43:05.997
FetchError: request to http://ks-apiserver/api/v1/nodes failed, reason: getaddrinfo EAI_AGAIN ks-apiserver
    at ClientRequest.<anonymous> (/opt/kubesphere/console/server/server.js:47991:11)
    at ClientRequest.emit (events.js:314:20)
    at Socket.socketErrorListener (_http_client.js:427:9)
    at Socket.emit (events.js:314:20)
    at emitErrorNT (internal/streams/destroy.js:92:8)
    at emitErrorAndCloseNT (internal/streams/destroy.js:60:3)
    at processTicksAndRejections (internal/process/task_queues.js:84:21) {
  type: 'system',
  errno: 'EAI_AGAIN',
  code: 'EAI_AGAIN'
}


问题1：概览的监控数据获取失败

```

- 升级失败解决
```
失败1
# 卸载报错的notification-manager
helm uninstall notification-manager -n kubesphere-monitoring-system

失败2：重启ks-console
kubectl -n kubesphere-system scale deployment ks-console --replicas=0 && kubectl -n kubesphere-system scale deployment ks-console --replicas=1

问题1：重启ks-apiserver
kubectl -n kubesphere-system scale deployment ks-apiserver --replicas=0 && kubectl -n kubesphere-system scale deployment ks-apiserver --replicas=1

```

# 整体排错思路
## 解决路径
1. 查看错误日志/事件，得到关键信息
2. 根据关键信息，试图简单自行解决
3. 解决不了， 根据关键信息到[KS论坛](https://kubesphere.com.cn/forum/)/Google 搜索
4. 搜索后无结果， 尝试深入自行解决（包括重新调度相关问题组件，甚至重启）
5. 深入后无法解决，想planB, 周知风险
6. 最后查看源码，找问题的根源， 社区提issue

## 常见报错场景
1. 前端页面报错。 查看ks-console的错误日志，再顺藤摸瓜往下定位
2. 命令执行报错。 根据报错内容，找对应的组件/服务， 再查看对应的日志/事件
3. 特定组件报错。 直接查看组件对应的日志/事件

## 场景排错命令
1. 找命名空间。 `kubectl get ns`
2. 找pod。 全局：`kubectl get pods -A`， 对应命名空间下： `kubectl get pods -n [ns]`
3. 找日志。 `kubectl logs [pod_name] -n [ns] [-f]`
4. 找调度状态/事件。 `kubectl describe [pod_name/service...] -n [ns]`
5. 强制删除k8s资源。 kubectl delete pods [prometheus-k8s-1]   -n [kubesphere-monitoring-system] --grace-period=0 --force,  kubectl delete --all pods --namespace=istio-system --grace-period=0 --force,  kubectl delete all --all -n kubesphere-logging-system --grace-period=0 --force


# 部署/升级常见错误
- notification-manager升级报错
```
# 报错日志
"runner_ident": "monitoring",
  "start_line": 114,
  "stdout": "fatal: [localhost]: FAILED! => {\"attempts\": 3, \"changed\": true, \"cmd\": \"/usr/local/bin/helm upgrade --install notification-manager /kubesphere/kubesphere/notification-manager -f /kubesphere/kubesphere/notification-manager/custom-values-notification.yaml -n kubesphere-monitoring-system --force\\n\", \"delta\": \"0:00:01.111866\", \"end\": \"2022-01-11 20:08:27.609767\", \"msg\": \"non-zero return code\", \"rc\": 1, \"start\": \"2022-01-11 20:08:26.497901\", \"stderr\": \"Error: UPGRADE FAILED: failed to replace object: Service \\\"notification-manager-controller-metrics\\\" is invalid: spec.clusterIPs[0]: Invalid value: []string(nil): primary clusterIP can not be unset && failed to replace object: Service \\\"notification-manager-webhook\\\" is invalid: spec.clusterIPs[0]: Invalid value: []string(nil): primary clusterIP can not be unset\", \"stderr_lines\": [\"Error: UPGRADE FAILED: failed to replace object: Service \\\"notification-manager-controller-metrics\\\" is invalid: spec.clusterIPs[0]: Invalid value: []string(nil): primary clusterIP can not be unset && failed to replace object: Service \\\"notification-manager-webhook\\\" is invalid: spec.clusterIPs[0]: Invalid value: []string(nil): primary clusterIP can not be unset\"], \"stdout\": \"\", \"stdout_lines\": []}",

# 解决。 卸载报错的notification-manager, 重装
helm uninstall notification-manager -n kubesphere-monitoring-system

```
- 登录失败报错， EAI_AGAIN
```
# 报错日志
  <-- GET / 2022/01/11T20:43:05.997
FetchError: request to http://ks-apiserver/api/v1/nodes failed, reason: getaddrinfo EAI_AGAIN ks-apiserver
    at ClientRequest.<anonymous> (/opt/kubesphere/console/server/server.js:47991:11)
    at ClientRequest.emit (events.js:314:20)
    at Socket.socketErrorListener (_http_client.js:427:9)
    at Socket.emit (events.js:314:20)
    at emitErrorNT (internal/streams/destroy.js:92:8)
    at emitErrorAndCloseNT (internal/streams/destroy.js:60:3)
    at processTicksAndRejections (internal/process/task_queues.js:84:21) {
  type: 'system',
  errno: 'EAI_AGAIN',
  code: 'EAI_AGAIN'
}

# 解决。 重启ks-console
kubectl -n kubesphere-system scale deployment ks-console --replicas=0 && kubectl -n kubesphere-system scale deployment ks-console --replicas=1
```

- 概览的监控数据获取失败
```
# 解决。 重启ks-apiserver
kubectl -n kubesphere-system scale deployment ks-apiserver --replicas=0 && kubectl -n kubesphere-system scale deployment ks-apiserver --replicas=1

```

# refer
- https://www.yuque.com/leifengyang/oncloud/vgf9wk
- https://kubesphere.com.cn/docs/installing-on-linux/introduction/multioverview/
- https://github.com/kubesphere/kubekey/tree/release-1.2/docs










