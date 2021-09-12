---
layout:     post
title:      Serverless之autoscale
subtitle:   
date:       2021-09-12
author:     jabin
header-img: 
catalog: true
tags:
    - 计算机
    - Serverless
    - autoscale
    - knative
---

the ability to scale workloads down to zero and quickly scale them back up as demands arrive, arguably, constitutes the most important trait of a serverless architecture

目前主流三种方案：k8s的HPA, knative, keda
# k8s自带
HPA机制， 俩痛点：
- a limited number of metrics that users can use to perform autoscaling
- cannot scale a deployment to 0 pods.

所以直接pass

# Knative
## Serving最重要概念
- 基于k8s。 a set of custom resources or CRDs that extend K8s to run serverless workloads
- KPA provides request-bases autoscaling capabilities。
- Autoscaller（KPA）：指标来源， • 通过获取每个 pod queue-proxy 中的指标  • Activator 通过 websocket 主动上报
- Activator：实例为0 时（冷启动），流量会先转发到 Activator，由 Activator 通过 websocket 主动触发 Autoscaler 扩缩容。  基于hpa
- We can use the standard Kubernetes HPA or implement our own pod autoscaler with a specialized autoscaling algorithm.
- 痛点：not so simple for everyone

## 安装
- K8s版本最低要求。目前天机k8s版本为1.16， 后续需升级，涉及多方改动。   https://github.com/knative/community/blob/main/mechanics/RELEASE-VERSIONING-PRINCIPLES.md#knative-serving-version-table
- 安装KubeSphere, 开启Istio组件
- 版本。 考虑目前使用的k8s为1.16, 所以选择安装kn的0.17.3
- 方式：[operator](https://github.com/knative/operator/blob/release-0.17/docs/installation.md)；
- 镜像拉取代理配置：/etc/systemd/system/docker.service.d/http-proxy.conf；
- demo服务启动：kn service create helloworld-go --image registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8 --env TARGET="World" --annotation autoscaling.knative.dev/target=10
- 访问配置到 istio gateway svc clusterip

# KEDA
- 基于k8s。 通过在底层使用 HPA 来扩展 Kubernetes，HPA 使用我们的外部指标，这些指标由我们自己的指标适配器提供，该适配器取代了所有其他适配器
- KEDA operator解决HPA无法将pod数缩到0的问题，缩到1则还是由HPA进行，扩容同理
- ScaledObject (KEDA custom resource definition)。提供多种metric provider： Redis / RabittMQ queue length， PubSub / SQS number of messages， Postgres / MySQL value of an arbitrary query
- 阿里的edas是基于KEDA实践

## 优势和痛点
- 相较Knative优势：The difference between Knative and KEDA is how to detect incoming request. Knative is very network oriented architecture. On the other hand, KEDA is not dependent on network so much. Instead of that, KEDA just depends on other services like Apache Kafka. This means KEDA is more light-weight.
- 痛点： 基于http的负载支持还是beta版本 https://github.com/kedacore/http-add-on

结论：基于我们目前的情况，基于Http的负载进行autoscale是强需求，因而需要用Knative


# Admission Webhook
业务场景的推理服务需要通过initContainer和volume提供算法模型的下载， 但是knative本身不支持， 所以通过webhook来支持。
webhook分两种validating admission webhook 和 mutating admission webhook, 这里我们讨论后一种
## Mutating Admission Webhooks
- 编写webhook服务(http)， 构建镜像
- 同deployment部署webhook
- 创建webhook对应的service
- 创建MutatingWebhookConfiguration， 注册webhook信息到k8s api server
- 通过api去创建k8s资源时，若匹配到对应的标签，则会走webhook
- 测试demo。 kn service create helloworld-go --force --image registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8 --env TARGET="World" --annotation autoscaling.knative.dev/target=10 --annotation modelBucket=autotables_47 --annotation modelPath=/autotables/47/trained_model/2616/1629363245/ --annotation initContainerImage=ccr.ccs.tencentyun.com/deepwisdom/metalab-downloader:v1.0 --annotation injectModelInitContainer=true

# refer
- https://itnext.io/serverless-keda-for-scaling-down-your-containers-to-zero-5bf35a75072c
- https://zhuanlan.zhihu.com/p/371987472
- https://v3-1.docs.kubesphere.io/zh/docs/project-user-guide/application-workloads/horizontal-pod-autoscaling/
- https://zhuanlan.zhihu.com/p/379361295
- https://developer.ibm.com/blogs/challenges-creating-an-open-scalable-and-secure-serverless-platform/
- https://dzone.com/articles/scale-to-zero-with-kubernetes
- https://www.jianshu.com/p/10b5b68e8cb1
- https://turbaszek.medium.com/application-scalability-part-3-knative-and-keda-6d277a8bb41c
- https://cloudnative.to/blog/mutating-admission-webhook/#%E5%88%9B%E5%BB%BA-admission-webhook
- https://medium.com/ibm-cloud/diving-into-kubernetes-mutatingadmissionwebhook-6ef3c5695f74



