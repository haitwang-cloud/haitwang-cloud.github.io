---
title: "Welcome to My Blog"
draft: false
---

English |
[简体中文]({{< relref "intro-zh" >}})
 |[SAP内推]({{< relref "referral" >}})


Welcome to my blog! 🚀 This is a hub for exploring cutting-edge technologies like Kubernetes, Istio, GPU management, Golang, and software development. Each article bridges theoretical knowledge with practical insights, offering hands-on guidance for cloud-native systems, distributed architectures, and advanced engineering practices. Whether you're troubleshooting Kubernetes issues, diving into Istio's traffic management, or optimizing GPU utilization, you'll find valuable resources here. Join me in unraveling complex tech concepts and advancing your skills!

## Large Language Models (LLMs) and AI Infrastructure

- [SGLang vs vLLM: Streaming JSON Performance Analysis](/llm/sglang-vs-vllm-streaming-json/)
- [Distributed Inference with vLLM](/llm/vllm-distributed-inference-doc/)
- [LLM Inference Engines Performance Comparison: vLLM vs sglang](/llm/llm-benchmark-en/)


## Delving into GPU Management with Kubernetes

- [Kubernetes GPU Management Basics: Introduction to Device Plugin and Source Code Analysis](/gpu/k8s-device-plugin-en/)
- [Advanced Kubernetes GPU Management: Enabling Nvidia MPS](/gpu/k8s-device-plugin-mps/)
- [Optimizing GPU Utilization in Kubernetes](/gpu/how-to-increase-gpu-utilization-in-kubernetes/)
- [Installing NVIDIA GPU Operator with A100 on Rocky Linux-based Kubernetes](/gpu/how-to-install-nvidia-gpu-operator-with-a100-on-kubernetes-base-rocky-linux/)
- [Troubleshooting: Resolving "Failed to initialize NVML: Unknown Error"](/gpu/nvml-error/)

## Istio: In-Depth Traffic Management for Microservices

- [Istio Control Plane Management on Kubernetes: Multi-Instance Deployment](/istio/how-to-install-multi-istio-control-plane/)
- [Practical Multi-Environment Application Development: Building Microservices with Istio](/istio/build-app-under-multi-istio/)
- [Exploring Istio Core Technologies: Network Principles and Sidecar Auto Injection](/istio/istio-sidecar-inject/)
- [Istio 502 Upstream Connection Reset: Root Cause Analysis and Troubleshooting](/istio/istio-upstream-error/)
- [Resolving Memory Error in Istioctl Analyze Command](/istio/istioctl-analyze-error-en/)

## Kubernetes (K8s)

### Core Concepts

- [Learning Kubernetes by Running Applications: A Beginner's Guide](/k8s/learning-k8s-by-running-app/)
- [Understanding the Difference Between K8s Affinity and Taint/Toleration](/k8s/diff-of-Affinity-and-taint/)
- [The Mechanism and Strategies of the Default Kubernetes Scheduler](/k8s/k8s-schedule-road-path/)
- [Effective Use of Secret, ConfigMap, and Lease in Kubernetes](/k8s/k8s-secret-configMap-Lease/)
- [Kubernetes Headless Service Explained](/k8s/headLess-svc/)

### Advanced Features

- [Bare Metal Kubernetes: Key Points You Need to Know](/k8s/bare-metal-kubernetes/)
- [K8s Cloud Provider Source Code Analysis](/k8s/k8s-cloud-provider/)
- [Kubernetes vs K3s: What's the Difference?](/k8s/k8s-vs-k3s/)
- [Introduction to K8s Informers](/k8s/k8s_informers/)
- [Kyverno ImageValidatingPolicy CRD Conflict Issue and Solution](/k8s/kyverno-crd-missing-en/)

### Operations & Development

- [Implementing Rate Limiting in K8s controller-runtime and client-go](/k8s/controller-runtime-client-go-rate-limiting/)
- [Resolving OCI Runtime Create Failed: Expected CgroupsPath](/k8s/oci-error/)
- [Client-go Label Selector Causing CPU Throttling: Diagnosis and Fix](/k8s/oom-killed-by-client-go-label-select/)
- [Leader Election in Kubernetes Using client-go](/k8s/leader-election-in-kubernetes-using-client-go/)
- [Deep Dive into Kubernetes Controller Object Stores and Indexers](/k8s/object-stores-and-indexers/)

### Tools & Practices

- [Unlocking Kubernetes Superpowers with k8sgpt-localai](/k8s/k8sgpt-operater/)
- [Simplifying Helm Charts Deployment: Using the tpl Function](/k8s/using-the-helm-tpl-function/)

## Golang

### Basics

- [Getting Started with Cobra CLI Framework](/golang/cobra-user-guide/)
- [Deep Dive into Go's init Function](/golang/init-function-introduction/)
- [Go init Function Introduction](/golang/the-golang-init-func/)
- [Understanding Pass by Value vs Pass by Reference in Golang](/golang/golang-pass-by-value-vs-pass-by-reference/)
- [Best Practices for Error Handling in Golang](/golang/error-handle/)
- [Three Efficient Ways to Compare Go Slices](/golang/compare-slice/)

### Concurrency & Performance

- [Mastering Golang's sync.Map Concurrent Map](/golang/go-sync-Map/)
- [Correct Usage of sync.Cond Condition Variable in Golang](/golang/go-sync-cond/)
- [What's New in Go 1.18](/golang/go-version-118-release-new/)

### Testing & Debugging

- [Fuzz Testing in Golang](/golang/go-fuzz-testing/)
- [Table Driven Unit Tests in Golang](/golang/table-driven-unit-tests/)
- [Golang Memory Leaks: Diagnosis and Fix](/golang/golang-Memory-Leaks/)
- [LeakProf: Lightweight Online Goroutine Leak Detection](/golang/leakprof-featherlight/)

### Dependency Management

- [Direct and Indirect Dependencies in go.mod](/golang/direct-indirect-dependency-module-go/)
- [How to Upgrade Golang Module Dependencies](/golang/how-to-upgrade-golang-dependencies/)
- [Getting Started with Golang Plugins](/golang/getting-started-with-golang-plugins/)
- [Managing Multiple Go Versions](/golang/managing-multiple-go-versions-with-go/)

## Networking

- [What is BGP: The Building Block of Network Architecture](/network/what-is-bgp/)
- [Linux Network Management: Mastering the ipset Command](/network/how-to-use-ipset/)
- [SSL/TLS Security Basics: Root Certificates vs Intermediate Certificates](/network/root-certificates-intermediate/)
- [Network Monitoring on macOS: Advanced tcpdump Usage](/network/tcp-dump-in-OSX/)
- [The Road to QUIC: Next-Generation Internet Transport Protocol](/network/the-road-to-quic/)

## Software Development: Best Practices & Techniques

- [Upstream and Downstream in Software Development](/software/upstream-downstream/)
- [Building Docker Images on Apple Silicon (M1)](/software/docker-build-on-m1-mac/)
- [Resolving Elasticsearch Connection Issues: "No Node Available"](/software/elastic/)
- [JSON Patch vs JSON Merge Patch](/software/json-patch-vs-merge-patch/)
- [Just-in-Time (JIT) Compilation Explained](/software/just-in-time/)


🎯 About Me: Over the past year, I have remained actively engaged in the open-source community, primarily contributing to cloud-native, Kubernetes, and AI infrastructure projects. My work spans reporting and resolving complex bugs in production environments, proposing and implementing new features for GPU scheduling and observability, and improving deployment documentation for large-scale AI serving frameworks such as vLLM. I have also helped refine workflows, provided troubleshooting for GPU operators, and advocated for enhancements to better support enterprise requirements in hybrid cloud clusters. My efforts reflect a deep commitment to improving reliability, scalability, and user experience in modern infrastructure systems.
