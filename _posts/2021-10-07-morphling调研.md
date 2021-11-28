---
layout:     post
title:      morphling调研 
subtitle:   
date:       2021-10-07
author:     jabin
header-img: 
catalog: true
tags:
    - 计算机
    - Model Serving
    
---

# AI推理服务特性

显卡资源。 单个推理服务独占，整张显卡将造成资源的极度浪费
性能的资源瓶颈多样。 复杂数据前处理  和 结果后处理，将占用大量cpu资源
容器的运行参数
# 推理服务的配置调优

开发倾向，冗余配置
默认配置
## 配置调优痛点

自动化性能测试、参数调优
稳定、非侵入式的服务性能测试流程。 不能直接在现网上测试
参数组合调优


# Morphling是什么

基于k8s的AI推理服务配置框架。 将参数组合调优全流程自动化，并结合高效的智能化调优算法，使推理业务的配置调优流程，能够高效地运行在 Kubernetes 之上，解决机器学习在产业实际部署中的性能和成本挑战。



# 实践

本地基于minikube install morphling，  跑mobilenet推理服务模型，发现morphling-ui  coredump…  且跑个几小时都没有结果。 可能是本地m1的原因？  不过因为刚出，没太多的资料， 也不是很稳定的， 可后续观察，再考虑
