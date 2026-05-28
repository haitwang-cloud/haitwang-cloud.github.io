
[English](https://tim-wang-tecg-blog.pages.dev/) |
简体中文

欢迎来到我的博客！🚀 这里是一个探索前沿技术的空间，包括 Kubernetes、Istio、GPU 管理、Golang 和软件开发。每篇文章都致力于将理论知识与实践技巧相结合，为云原生系统、分布式架构和高级工程实践提供实用指导。无论是解决 Kubernetes 问题、深入 Istio 流量管理，还是优化 GPU 使用率，您都能在这里找到有价值的资源。让我们一起解锁复杂技术概念，提升技能水平！

## 大语言模型 和 AI 基础设施

- [vLLM分布式推理](/llm/vllm-distributed-inference-doc/)
- [大语言模型推理引擎性能对比分析：vLLM与sglang](/llm/llm-benchmark-cn/)
- [SGLang vs vLLM: Streaming JSON 性能分析](/llm/sglang-vs-vllm-streaming-json/)

## GPU：探索GPU的奥秘，从基础到高级应用

- Kubernetes GPU 管理基础：Device Plugin 介绍与源码分析 [Kubernetes GPU 管理基础](/gpu/k8s-device-plugin/)
- Kubernetes GPU 管理进阶：启用 Nvidia MPS [启用 Nvidia MPS](/gpu/k8s-device-plugin-mps/)
- 故障排查：解决 "Failed to initialize NVML: Unknown Error" [解决 Kubernetes GPU 管理错误](/gpu/nvml-error/)
- Kubernetes GPU 优化：最大化 GPU 利用率 [Kubernetes GPU 优化](/gpu/how-to-increase-gpu-utilization-in-kubernetes/)
- 特定环境 GPU 管理：在 Rocky Linux 上安装 NVIDIA GPU Operator [在 Rocky Linux 上安装 NVIDIA GPU Operator](/gpu/how-to-install-nvidia-gpu-operator-with-a100-on-kubernetes-base-rocky-linux/)

## Istio：深入理解微服务之间的流量管理

- Kubernetes上的Istio控制面管理：多实例部署实战 [多控制平面配置指南](/istio/how-to-install-multi-istio-control-plane/)
- 多环境应用开发实战：Istio下的微服务构建策略 [无缝对接多环境:基于Istio的微服务应用构建技巧](/istio/build-app-under-multi-istio/)
- Istio流量异常处理：502错误根源分析与解决 [Istio应用故障排查手册:上游连接重置502错误详解](/istio/istio-upstream-error/)
- Istio核心技术探秘：网络原理与Sidecar自动注入机制 [深入理解Istio：网络原理与Sidecar的自动注入机制](/istio/istio-sidecar-inject/)
- 如何解决 Istioctl Analyze -A 命令报invalid memory address问题 [panic: runtime error: invalid memory address](/istio/istioctl-analyze-error/)

## Golang 资源大全：从基础到高级特性

探索 Golang 的世界，这里有你需要的所有资源，从基础语法到高级特性，一网打尽！

### Golang 基础与入门

- Cobra使用指南：学习如何使用Cobra创建强大的现代CLI应用程序 [阅读更多](/golang/cobra-user-guide/)
- Go init 函数详解：深入了解Go程序的自动初始化过程 [点击了解](/golang/init-function-introduction/)
- Go init 函数使用介绍 [点击了解](/golang/the-golang-init-func/)
- Golang中的值传递与引用传递：掌握Go中数据传递的基本概念 [深入理解](/golang/golang-pass-by-value-vs-pass-by-reference/)
- Golang 中的有效错误处理：探索错误处理的最佳实践，提高代码的健壮性 [掌握技巧](/golang/error-handle/)
- Go切片操作秘籍：三种高效比较技巧 [探索技巧](/golang/compare-slice/)

### Golang 并发与性能

- Golang sync.Map使用介绍：理解并使用Go的高效并发Map。 [学习使用](/golang/go-sync-Map/)
- Go 1.18 中的新功能：探索Go语言最新版本的特性，提升开发效率。 [查看更新](/golang/go-version-118-release-new/)
- Golang中正确使用条件变量sync.Cond：学习如何在并发编程中正确使用条件变量。 [学习使用](/golang/go-sync-cond/)

### Golang 测试、调试与性能优化

- 在Go中进行 Fuzz Testing：使用Fuzz Testing发现代码中的潜在问题 [探索测试](/golang/go-fuzz-testing/)
- 在 Golang中进行 Table Driven Unit Tests：通过表驱动测试提高代码的测试覆盖率 [高效测试](/golang/table-driven-unit-tests/)
- Golang内存泄漏问题详细：诊断和修复内存泄漏，优化程序性能 [解决内存问题](/golang/golang-Memory-Leaks/)
- LeakProf: 轻量级在线Goroutine泄漏检测：使用LeakProf工具检测和修复Goroutine泄漏 [检测泄漏](/golang/leakprof-featherlight/)

### Golang 高级特性与最佳实践

- go.mod 文件中的直接和间接依赖：掌握依赖管理，优化项目构建过程。 [深入了解](/golang/direct-indirect-dependency-module-go/)
- 如何升级Golang module的依赖：学习如何高效地管理和升级项目依赖。 [升级指南](/golang/how-to-upgrade-golang-dependencies/)
- 开始使用Golang Plugins：探索Go的插件系统，扩展语言能力。 [插件入门](/golang/getting-started-with-golang-plugins/)
- 使用Go管理多个Go版本：学习如何在开发环境中管理不同版本的Go [高效管理](/golang/managing-multiple-go-versions-with-go/)

## Kubernetes (K8s)

从裸机部署到高级调度策略，全方位掌握Kubernetes。

### Kubernetes 基础和核心概念

- 通过运行应用学习Kubernetes：为初学者设计的入门指南 [开始学习](/k8s/learning-k8s-by-running-app/)
- Kubernetes headless Service介绍：了解headless服务的基本概念 [了解headless服务](/k8s/headLess-svc/)
- k8s Affinity与 taint/toleration的区别：理解工作节点的亲和性与排斥性配置 [比较affinity与taint/toleration](/k8s/diff-of-Affinity-and-taint/)
- k8s 默认的调度器工作机制和策略：深入理解Kubernetes调度器的工作原理。[调度器机制](/k8s/k8s-schedule-road-path/)
- 如何在 Kubernetes 中有效使用 Secret、ConfigMap 和 Lease：详解及示例。[使用 Secret、ConfigMap 和 Lease](/k8s/k8s-secret-configMap-Lease/)

### Kubernetes 高级特性与优化 (Advanced)

- 全面解析Bare Metal Kubernetes:必知的关键点：深入理解在裸金属上部署Kubernetes的关键要素 [Bare Metal Kubernetes解析](/k8s/bare-metal-kubernetes/)
- K8s Cloud Provider源码解析：深入分析Kubernetes云服务提供商的源码 [源码解析](/k8s/k8s-cloud-provider/)
- Kubernetes与K3s比较：探索Kubernetes与K3s的不同之处 [Kubernetes与K3s比较](/k8s/k8s-vs-k3s/)
- K8s informers的介绍：掌握Kubernetes的事件通知机制 [K8s informers的介绍](/k8s/k8s_informers/)
- [Kyverno ImageValidatingPolicy CRD 冲突问题及解决方案](/k8s/kyverno-crd-missing-zh/)

### Kubernetes 运维与开发

- 在K8s controller-runtime和client-go中实现速率限制：学习如何在Kubernetes中控制API调用频率。[速率限制实现](/k8s/controller-runtime-client-go-rate-limiting/)
- OCI runtime create failed: expected cgroupsPath：解决容器运行时配置问题。 [解决OCI配置问题](/k8s/oci-error/)
- Client-go 中的label selector 引起的 CPU Throttling问题：诊断和修复CPU限制问题。[CPU Throttling问题解决](/k8s/oom-killed-by-client-go-label-select/)
- 使用client-go在Kubernetes中进行leader election：实现高可用性集群。[leader选举](/k8s/leader-election-in-kubernetes-using-client-go/)
- 深入了解Kubernetes控制器对象存储（object stores）和索引器（indexers）：提升控制器性能的高级技巧 [控制器对象存储与索引器](/k8s/object-stores-and-indexers/)

### Kubernetes 工具与实践

- 用k8sgpt-localai解锁Kubernetes的超能力：[探索AI技术在Kubernetes中的应用](/k8s/k8sgpt-operater/)
- 简化Helm Charts部署：使用tpl函数引用Values：提高Helm部署效率 [简化Helm 部署](/k8s/using-the-helm-tpl-function/)

## 网络技术深度解析：实战指南与核心概念

- BGP路由协议深度揭秘：网络架构的基石 [网络基础构建：揭秘BGP路由协议](/network/what-is-bgp/)
- Linux网络管理实战：精通ipset命令运用 [Linux网络高手秘籍：高效利用ipset命令](/network/how-to-use-ipset/)
- SSL/TLS安全基础：根证书与中间证书解析 [安全基石：根证书与中间证书的区别与应用](/network/root-certificates-intermediate/)
- OSX网络监控精粹：tcpdump高级操作手册 [OSX环境下的网络监听艺术：tcpdump高级技巧](/network/tcp-dump-in-OSX/)
- QUIC协议前瞻：下一代互联网传输协议的探索 [QUIC协议：重塑互联网传输的未来之路](/network/the-road-to-quic/)

## Software Development：软件开发实践与技术

- 软件开发的上下游链路解析 [探索软件开发生命周期中的上下游协作](/software/upstream-downstream/)
- Apple M1芯片（M1 Mac）Docker镜像构建全面指南 [M1芯片上的Docker构建实战](/software/docker-build-on-m1-mac/)
- 解决Elasticsearch连接问题：No Node Available [实战教程](/software/elastic/)
- JSON补丁技术对比：Patch vs Merge Patch [JSON补丁技术深度比较：Patch与Merge Patch的优劣分析](/software/json-patch-vs-merge-patch/)
- 即时编译(JIT)全面解读：提升程序执行效率的关键 [即时编译(JIT)技术揭秘](/software/just-in-time/)

🎯 关于我：在过去的一年里，我持续活跃于开源社区，主要聚焦于云原生、Kubernetes 以及 AI 基础设施相关项目。我的工作涵盖了在生产环境中报告和解决复杂的 bug，为 GPU 调度与可观测性提出并实现新特性，优化了如 vLLM 等大规模 AI 推理框架的部署文档。同时，我还参与了工作流优化，协助排查 GPU Operator 问题，并积极推动相关特性以更好地支持企业在混合云集群中的需求。我的贡献体现了我对提升现代基础设施可靠性、可扩展性和用户体验的持续追求。
