+++
title = '深入理解Istio：网络原理与Sidecar的自动注入机制'
date = 2024-06-20T16:24:32+08:00
draft = false
+++

## Istio 简介
![mps](./pics/istio-pod-sidecar-container-envoy.png)
Istio，作为一款服务网格领域的佼佼者，为微服务架构提供了强大而灵活的网络能力。
本文旨在深入探讨Istio的网络原理，揭示其如何在复杂的服务间通信中发挥关键作用，同时保持高可观察性和安全性。

Istio 的核心在于其服务网格模型，它围绕着Envoy sideCar构建。每个服务实例旁边都会部署一个Envoy代理，负责处理进出服务的所有网络流量。
这一设计允许Istio在透明地拦截、路由和监控所有通信的同时，不对服务代码本身做出任何修改。

## Istio sidecar 注入过程
![mps](./pics/Istio-Sidecar-Injector-Webhook.png)
Istio sidecar 注入过程是指在 Kubernetes 集群中，将 Istio 的 Envoy 代理作为一个 sidecar 容器，注入到每个业务容器所在的 Pod 中，从而实现对业务容器的流量管理和控制。

Istio sidecar 注入过程主要有两种方式：手动注入和自动注入。

手动注入是指在创建 Pod 时，在 Pod 的 YAML 文件中，手动添加 Istio 的 Envoy 代理容器，并且配置好 Istio 和 Envoy 代理容器之间的通信方式。这种方式比较繁琐，且容易出错。

自动注入是指在创建 Pod 时，由 Istio 的 sidecar 注入器（`Istio-sidecar-injector`）自动将 Istio 的 Envoy 代理容器注入到 Pod 中。这种方式比较简便，且可以保证 Istio 和 Envoy 代理容器之间的通信方式是正确的。

Istio-sidecar-injector 的工作原理是基于 Kubernetes 的 Admission Controller 机制。在创建 Pod 时，Kubernetes 会将 Pod 的创建请求提交给 Admission Controller 进行校验，如果校验通过，则将 Pod 创建成功。Istio-sidecar-injector 就是一个自定义的 Admission Controller，它在接收到 Pod 的创建请求时，会将 Istio 的 Envoy 代理容器注入到 Pod 中，然后将修改后的 Pod 创建请求提交给 Kubernetes 进行创建。

以下是 Istio-sidecar-injector 的工作流程：

1. 在 Kubernetes 集群中部署 Istio-sidecar-injector，并将其注册为一个 Admission Controller。
2. 在创建 Pod 时，将 Pod 的创建请求提交给 Kubernetes。
3. Kubernetes 接收到 Pod 的创建请求后，会将其提交给 Istio-sidecar-injector 进行校验。
4. Istio-sidecar-injector 在接收到 Pod 的创建请求时，会首先判断该 Pod 是否需要注入 Istio 的 Envoy 代理容器。如果不需要，则直接将 Pod 创建请求提交给 Kubernetes 进行创建。如果需要，则进行下一步。
5. Istio-sidecar-injector 会根据 Istio 的 ConfigMap 中的配置，将 Istio 的 Envoy 代理容器注入到 Pod 中，并且配置好 Istio 和 Envoy 代理容器之间的通信方式。
6. Istio-sidecar-injector 将修改后的 Pod 创建请求提交给 Kubernetes 进行创建。
7. Kubernetes 接收到修改后的 Pod 创建请求后，会将 Pod 创建成功。
```yaml
webhooks:
  - name: rev.namespace.sidecar-injector.istio.io
    clientConfig:
      service:
        namespace: istio-system
        name: istiod
        path: /inject
        port: 443
    rules:
      - operations:
          - CREATE
        apiGroups:
          - ''
        apiVersions:
          - v1
        resources:
          - pods
        scope: '*'
    failurePolicy: Fail
    matchPolicy: Equivalent
    namespaceSelector:
      matchExpressions:
        - key: istio.io/rev
          operator: In
          values:
            - default
        - key: istio-injection
          operator: DoesNotExist
    objectSelector:
      matchExpressions:
        - key: sidecar.istio.io/inject
          operator: NotIn
          values:
            - 'false'
    sideEffects: None
    timeoutSeconds: 10
    admissionReviewVersions:
      - v1beta1
      - v1
    reinvocationPolicy: Never
```
以下是 Istio-sidecar-injector 的部分代码：
```go
func (c *Controller) Handle(ctx context.Context, req admission.Request, resp admission.Response) error {
    obj, ok, err := c.extractor.Extract(req)
    if err != nil {
        return err
    }
    if !ok {
        resp.Allowed = true
        return nil
    }

    pod, ok := obj.(*v1.Pod)
    if !ok {
        resp.Allowed = true
        return nil
    }

    if shouldInjectSidecar(pod) {
        injectedPod, err := injectSidecar(pod)
        if err != nil {
            return err
        }
        req.Object = injectedPod
    }

    resp.Allowed = true
    return nil
}

func shouldInjectSidecar(pod *v1.Pod) bool {
    // 判断 Pod 是否需要注入 Istio 的 Envoy 代理容器
    // 具体的判断逻辑可能会根据 Istio 的 ConfigMap 中的配置而有所不同
    // 这里以判断 Pod 是否带有 istio-injection=enabled 的标签为例
    if val, ok := pod.Labels["istio-injection"]; ok && val == "enabled" {
        return true
    }
    return false
}

func injectSidecar(pod *v1.Pod) (*v1.Pod, error) {
    // 根据 Istio 的 ConfigMap 中的配置，将 Istio 的 Envoy 代理容器注入到 Pod 中
    // 并且配置好 Istio 和 Envoy 代理容器之间的通信方式
    // 这里的具体实现可能会比较复杂，且可能会根据 Istio 的版本和配置而有所不同
    // 因此这里省略了具体的实现
    return pod, nil
}
```
可以看到，Istio-sidecar-injector 的 Handle 函数是在接收到 Pod 的创建请求时，
首先判断该 Pod 是否需要注入 Istio 的 Envoy 代理容器。如果不需要，则直接将 Pod 创建请求提交给 Kubernetes 进行创建。
如果需要，则调用 injectSidecar 函数将 Istio 的 Envoy 代理容器注入到 Pod 中，
并且配置好 Istio 和 Envoy 代理容器之间的通信方式。最后将修改后的 Pod 创建请求提交给 Kubernetes 进行创建。

### Istio如何动态加载ConfigMap配置`istio-sidecar-injector`

尽管在istiod的部署配置中未直接显现对`istio-sidecar-injector`的引用，实际上，`istio-sidecar-injector`的配置是通过`MutatingWebhookConfiguration`动态加载的。

```shell
➜  k get deploy -n istio-system istiod -o yaml | grep istio-sidecar-injector | wc
       0       0       0
```
源码分析证实了这一点，如`kubeinject.go`中的逻辑展示了如何从ConfigMap读取Istio的网格配置信息，进而指导sidecar的注入过程。这表明，尽管直接检查部署描述可能看不到明确的关联，但sidecar-injector确实在背后利用ConfigMap动态配置其行为。

```go
func getMeshConfigFromConfigMap(ctx cli.Context, command, revision string) (*meshconfig.MeshConfig, error) {
	client, err := ctx.CLIClient()
	if err != nil {
		return nil, err
	}

	if meshConfigMapName == defaultMeshConfigMapName && revision != "" {
		meshConfigMapName = fmt.Sprintf("%s-%s", defaultMeshConfigMapName, revision)
	}
	meshConfigMap, err := client.Kube().CoreV1().ConfigMaps(ctx.IstioNamespace()).Get(context.TODO(), meshConfigMapName, metav1.GetOptions{})
	if err != nil {
		return nil, fmt.Errorf("could not read valid configmap %q from namespace %q: %v - "+
			"Use --meshConfigFile or re-run "+command+" with `-i <istioSystemNamespace> and ensure valid MeshConfig exists",
			meshConfigMapName, ctx.IstioNamespace(), err)
	}
	// values in the data are strings, while proto might use a
	// different data type.  therefore, we have to get a value by a
	// key
	configYaml, exists := meshConfigMap.Data[configMapKey]
	if !exists {
		return nil, fmt.Errorf("missing configuration map key %q", configMapKey)
	}
	cfg, err := mesh.ApplyMeshConfigDefaults(configYaml)
	if err != nil {
		err = multierror.Append(err, fmt.Errorf("istioctl version %s cannot parse mesh config.  Install istioctl from the latest Istio release",
			version.Info.Version))
	}
	return cfg, err
}
```

综上所述，Istio通过巧妙的sidecar注入机制和动态配置加载，实现了对微服务网络的深度管理和优化，展现了其在现代云原生架构中的核心价值。
## Reference
- [istios-networking-in-depth/](https://www.solo.io/blog/istios-networking-in-depth/)
- [Installing the Sidecar](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/#customizing-injection)
- [Learn Kubernetes Basics with Learning Paths-Istio](https://kubebyexample.com/learning-paths/istio/intro)
