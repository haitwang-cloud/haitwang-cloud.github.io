---
title: "Welcome to My Blog"
draft: false
---
Welcome to my tech exploration hub! 🚀 Here, we dive into Istio, GPU technology, Golang, networking engineering, software development ecosystems, and practical insights on Kubernetes. Each article, whether original or meticulously translated, aims to build a bridge between theoretical knowledge and hands-on skills, expanding your technical horizons. Let's embark on this journey together, level by level! 📚

🎉 欢迎莅临我的技术探索空间! 🚀 在这里，我们深入探讨 Istio、GPU 技术、Golang、网络工程、软件开发生态以及 Kubernetes 的实践智慧。每一篇文章，无论是原创还是精心翻译，都旨在为您搭建一座桥梁，连接理论知识与实战技巧，拓宽您的技术视界。📚 结伴同行，在技术之旅上步步高升！

## GPU：探索GPU的奥秘，从基础到高级应用

- Kubernetes GPU 管理基础：Device Plugin 介绍与源码分析[Kubernetes GPU 管理基础](./gpu/k8s-device-plugin)
- Kubernetes GPU 管理进阶：启用 Nvidia MPS[启用 Nvidia MPS](./gpu/k8s-device-plugin-mps)
- 故障排查：解决 "Failed to initialize NVML: Unknown Error"[解决 Kubernetes GPU 管理错误](./gpu/nvml-error/)
- Kubernetes GPU 优化：最大化 GPU 利用率[Kubernetes GPU 优化](./gpu/how-to-increase-gpu-utilization-in-kubernetes)
- 特定环境 GPU 管理：在 Rocky Linux 上安装 NVIDIA GPU Operator[在 Rocky Linux 上安装 NVIDIA GPU Operator](./gpu/how-to-install-nvidia-gpu-operator-with-a100-on-kubernetes-base-rocky-linux)

## Istio：深入理解微服务之间的流量管理

- [Kubernetes集群中的Istio环境管理:控制平面的多实例部署实践](./istio/how-to-install-multi-istio-control-plane)
- [Kubernetes集群中的Istio环境管理:多环境应用构建实践](./istio/build-app-under-multi-istio)
- [Istio上游连接重置502错误分析与排查指南](./istio/istio-upstream-error)
- [深入理解Istio：网络原理与Sidecar的自动注入机制](./istio/istio-sidecar-inject)


## Golang 资源大全：从基础到高级特性

探索 Golang 的世界，这里有你需要的所有资源，从基础语法到高级特性，一网打尽！

### Golang 基础与入门

- Cobra使用指南：学习如何使用Cobra创建强大的现代CLI应用程序 [阅读更多](./golang/cobra-user-guide/)
- Go init 函数详解：深入了解Go程序的自动初始化过程 [点击了解](./golang/init-function-introduction)
- Golang中的值传递与引用传递：掌握Go中数据传递的基本概念 [深入理解](./golang/golang-pass-by-value-vs-pass-by-reference)
- Golang 中的有效错误处理：探索错误处理的最佳实践，提高代码的健壮性 [掌握技巧](./golang/error-handling-best-practices)
- Go切片操作秘籍：三种高效比较技巧 [探索技巧](./golang/compare-slice)

### Golang 并发与性能

- Golang sync.Map使用介绍：理解并使用Go的高效并发Map。 [学习使用](./golang/go-sync-Map)
- Go 1.18 中的新功能：探索Go语言最新版本的特性，提升开发效率。 [查看更新](./golang/go-version-118-release-new)
- Golang中正确使用条件变量sync.Cond：学习如何在并发编程中正确使用条件变量。 [学习使用](./golang/go-sync-cond)

### Golang 测试、调试与性能优化

- 在Go中进行 Fuzz Testing：使用Fuzz Testing发现代码中的潜在问题 [探索测试](./golang/go-fuzz-testing)
- 在 Golang中进行 Table Driven Unit Tests：通过表驱动测试提高代码的测试覆盖率 [高效测试](./golang/table-driven-unit-tests)
- Golang内存泄漏问题详细：诊断和修复内存泄漏，优化程序性能 [解决内存问题](./golang/golang-Memory-Leaks)
- LeakProf: 轻量级在线Goroutine泄漏检测：使用LeakProf工具检测和修复Goroutine泄漏 [检测泄漏](./golang/leakprof-featherlight)

### Golang 高级特性与最佳实践

- go.mod 文件中的直接和间接依赖：掌握依赖管理，优化项目构建过程。 [深入了解](./golang/direct-indirect-dependency-module-go)
- 如何升级Golang module的依赖：学习如何高效地管理和升级项目依赖。 [升级指南](./golang/how-to-upgrade-golang-dependencies)
- 开始使用Golang Plugins：探索Go的插件系统，扩展语言能力。 [插件入门](./golang/getting-started-with-golang-plugins)

### Golang 依赖管理与版本控制

- go.mod 文件中的直接和间接依赖：掌握依赖管理，优化项目构建过程。 [深入了解](./golang/direct-indirect-dependency-module-go)
- 使用Go管理多个Go版本：学习如何在开发环境中管理不同版本的Go [高效管理](./golang/managing-multiple-go-versions-with-go)


## Kubernetes (K8s)

从裸机部署到高级调度策略，全方位掌握Kubernetes。

### Kubernetes 基础和核心概念

- 通过运行应用学习Kubernetes：为初学者设计的入门指南[开始学习](./k8s/learning-k8s-by-running-app/)
- Kubernetes headless Service介绍：了解headless服务的基本概念[了解headless服务](./k8s/headLess-svc/)
- k8s Affinity与 taint/toleration的区别：理解工作节点的亲和性与排斥性配置[比较affinity与taint/toleration](./k8s/diff-of-Affinity-and-taint/)
- k8s 默认的调度器工作机制和策略：深入理解Kubernetes调度器的工作原理。[调度器机制](./k8s/k8s-schedule-road-path/)
- 如何在 Kubernetes 中有效使用 Secret、ConfigMap 和 Lease：详解及示例。[使用 Secret、ConfigMap 和 Lease](./k8s/k8s-secret-configMap-Lease/)
  
### Kubernetes 高级特性与优化 (Advanced)

- 全面解析Bare Metal Kubernetes:必知的关键点：深入理解在裸金属上部署Kubernetes的关键要素[Bare Metal Kubernetes解析](./k8s/bare-metal-kubernetes/)
- K8s Cloud Provider源码解析：深入分析Kubernetes云服务提供商的源码[源码解析](./k8s/k8s-cloud-provider/)
- Kubernetes与K3s比较：探索Kubernetes与K3s的不同之处[Kubernetes与K3s比较](./k8s/k8s-vs-k3s/)
- K8s informers的介绍：掌握Kubernetes的事件通知机制[K8s informers的介绍](./k8s/k8s_informers/)

### Kubernetes 运维与开发

- 在K8s controller-runtime和client-go中实现速率限制：学习如何在Kubernetes中控制API调用频率。[速率限制实现](./k8s/controller-runtime-client-go-rate-limiting/)
- OCI runtime create failed: expected cgroupsPath：解决容器运行时配置问题。 [解决OCI配置问题](./k8s/oci-error/)
- Client-go 中的label selector 引起的 CPU Throttling问题：诊断和修复CPU限制问题。[CPU Throttling问题解决](./k8s/oom-killed-by-client-go-label-select/)
- 使用client-go在Kubernetes中进行leader election：实现高可用性集群。[leader选举](./k8s/leader-election-in-kubernetes-using-client-go/)
- 深入了解Kubernetes控制器对象存储（object stores）和索引器（indexers）：提升控制器性能的高级技巧[控制器对象存储与索引器](./k8s/object-stores-and-indexers/)

### Kubernetes 工具与实践

- 用k8sgpt-localai解锁Kubernetes的超能力：[探索AI技术在Kubernetes中的应用](./k8s/k8sgpt-operater/)
- 简化Helm Charts部署：使用tpl函数引用Values：提高Helm部署效率[简化Helm 部署](./k8s/using-the-helm-tpl-function/)
  
## 网络工程：网络技术的实践与应用

- [什么是BGP？ | BGP路由解析](./network/what-is-bgp)
- [如何在Linux中使用ipset命令](./network/how-to-use-ipset)
- [根证书和中间证书之间的区别](./network/root-certificates-intermediate)
- [在OSX中使用tcpdump](./network/tcp-dump-in-OSX)
- [QUIC的发展之路](./network/the-road-to-quic)

## Software Development：软件开发实践与技术

- [软件开发中的上游和下游](./software/upstream-downstream)
- [如何在Apple 芯片也称为M1芯片）上构建Docker镜像](./software/docker-build-on-m1-mac)
- [如何解决No Elasticsearch Node Available for olivere/elastic](./software/elastic)
- [JSON Patch and JSON Merge Patch](./software/json-patch-vs-merge-patch)
- [什么是即时编译 (Just in Time)](./software/just-in-time)

🎯 About Me: I am a tech enthusiast and an engineer, specializing in cloud-native technologies, distributed systems, network engineering, and Golang. With over 5 years of technical experience 🔧, I have played significant roles at companies like eBay and SAP 👨‍💻, focusing on the construction and maintenance of K8S and Istio mesh. Recently, I have been deeply involved in the management of vGPUs within K8S clusters, exploring new technologies and best practices in this field. As a lifelong learner 🎓, I enjoy sharing my knowledge and experience through my blog. Whether you are a seasoned professional or a newcomer, I welcome you to engage and learn with me!

🎯 关于我：我是一名热爱技术的工程师，专注于云原生、分布式系统、网络工程、Golang 等领域, 我拥有超过 5 年的技术经验🔧，在 eBay 和 SAP 担任过重要角色👨‍💻，专注于 K8S 和 Istio mesh 的构建和维护。最近，我深入研究了 K8S 集群中的 vGPU 管理，探索这一领域的新技术和最佳实践。作为一个终身学习者🎓，我乐于通过博客分享我的知识和经验，无论您是经验丰富的专业人士还是刚起步的新手，都欢迎与我交流和学习！