+++
title = 'Kyverno ImageValidatingPolicy CRD 冲突问题及解决方案'
date = 2025-12-04T10:11:13+08:00
draft = false
+++
# Kyverno ImageValidatingPolicy CRD 冲突问题及解决方案

在 Kubernetes 集群中，Kyverno 是一个强大的策略引擎，用于管理和验证资源。然而，在将 Kyverno 从 **1.13** 升级到 **1.15** 的过程中，我们遇到了一个复杂的 CRD 冲突问题，导致 `ImageValidatingPolicy` 无法正常工作。本文记录了从权限问题的误导性排查到最终发现 CRD 冲突的完整过程。

---

## 问题背景

在升级 Kyverno 后，控制器日志中出现以下错误：

```bash
2025-12-03T06:08:13Z ERR ... Failed to watch error="failed to list *v1alpha1.ImageValidatingPolicy: the server could not find the requested resource (get imagevalidatingpolicies.policies.kyverno.io)"
2025-12-03T06:08:19Z ERR ... no matches for kind "ImageValidatingPolicy" in version "policies.kyverno.io/v1alpha1"
2025-12-03T06:09:59Z ERR ... failed to wait for imagevalidatingpolicy caches to sync
```

Kyverno 控制器无法访问 `imagevalidatingpolicies.policies.kyverno.io`。
API Server 未正确服务该资源类型。

## 初步排查：权限问题的误导
1. 检查 ServiceAccount 和 RBAC 配置
首先，我们检查了 `Kyverno Admission Controller` 的 `ServiceAccount` 和 `ClusterRoleBinding` 配置：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kyverno-admission-controller
  namespace: ssdl-kyverno
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kyverno:admission-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kyverno:admission-controller
subjects:
- kind: ServiceAccount
  name: kyverno-admission-controller
  namespace: ssdl-kyverno
```

ClusterRole 的规则中包含了对 imagevalidatingpolicies 的访问权限：

```yaml
rules:
- apiGroups: ["policies.kyverno.io"]
  resources: ["imagevalidatingpolicies"]
  verbs: ["get", "list", "watch"]
```

通过检查发现，RBAC 配置是正确的。

2. 使用 `kubectl auth can-i` 验证权限
进一步验证 ServiceAccount 是否有权限访问资源：

```bash
kubectl auth can-i list imagevalidatingpolicies.policies.kyverno.io --as=system:serviceaccount:ssdl-kyverno:kyverno-admission-controller

Warning: the server doesn't have a resource type 'imagevalidatingpolicies' in group 'policies.kyverno.io'
no
```

结果返回 `no`，并提示资源类型不存在。这一结果将我们误导到权限问题上，但实际上问题的根因并非 RBAC。

## 深入排查：CRD 冲突导致资源类型未被服务

1. 检查 CRD 是否存在
使用以下命令检查 CRD 是否已安装：
```bash
kubectl get crd imagevalidatingpolicies.policies.kyverno.io
```
结果显示 CRD 存在，但问题并未解决。

2. 检查 API 资源是否可用
使用以下命令检查 API Server 是否服务该资源：
```bash
kubectl api-resources --api-group=policies.kyverno.io
```

3. 检查 CRD 状态
 
进一步查看 CRD 的详细状态：
```bash
kubectl get crd imagevalidatingpolicies.policies.kyverno.io -o yaml
```

关键部分如下：
```yaml
status:
  conditions:
  - type: NamesAccepted
    status: "False"
    reason: ShortNamesConflict
    message: '"ivpol" is already in use'
  - type: Established
    status: "False"
    reason: NotAccepted
    message: not all names are accepted
```

发现问题：

- `NamesAccepted=False`：短名（shortNames）`ivpol` 已被占用。
- `Established=False`：CRD 未能完全建立，API Server 无法服务该资源

4. 确认短名冲突
使用以下命令查找占用 `ivpol` 的 CRD：

```bash
kubectl get crd -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.names.shortNames}{"\n"}{end}' | grep -w ivpol
```

输出结果：
```bash
imagevalidatingpolicies.policies.kyverno.io ["ivpol"]
imageverificationpolicies.policies.kyverno.io ["ivpol"]
```

**冲突来源**

Kyverno 1.13 中的 CRD `imageverificationpolicies.policies.kyverno.io` 使用了短名 `ivpol`。
Kyverno 1.15 中，`imageverificationpolicies` 已被废弃，取而代之的是新的 CRD `imagevalidatingpolicies.policies.kyverno.io`，但 Helm Chart 中仍然为其分配了相同的短名 `ivpol`。
由于升级过程中旧的 CRD `imageverificationpolicies` 未被清理，导致短名冲突，新的 CRD `imagevalidatingpolicies` 无法建立。

## 解决方案

将老的 CRD `imageverificationpolicies.policies.kyverno.io` 删除，释放短名 `ivpol`。
```bash
kubectl delete crd imageverificationpolicies.policies.kyverno.io
```

### 验证修复
确认 CRD `imagevalidatingpolicies.policies.kyverno.io` 状态
```bash
kubectl describe crd imagevalidatingpolicies.policies.kyverno.io
```

期望输出中包含以下内容：
```yaml
status:
  acceptedNames:
    categories:
    - kyverno
    kind: ImageValidatingPolicy
    listKind: ImageValidatingPolicyList
    plural: imagevalidatingpolicies
    shortNames:
    - ivpol
    singular: imagevalidatingpolicy
  conditions:
  - lastTransitionTime: "2025-12-03T07:39:18Z"
    message: no conflicts found
    reason: NoConflicts
    status: "True"
    type: NamesAccepted
  - lastTransitionTime: "2025-12-03T07:39:18Z"
    message: the initial names have been accepted
    reason: InitialNamesAccepted
    status: "True"
    type: Established
  storedVersions:
  - v1alpha1
```

接下来，重启所有的 `kyverno-admission-controller` Pod，以确保它们使用最新的 CRD 定义。
```bash
kubectl rollout restart deployment kyverno-admission-controller -n ssdl-kyverno
```

## 总结

**现象**：Kyverno 控制器无法访问 `imagevalidatingpolicies`，API Server 提示资源类型不存在。
**根因**：升级过程中，旧的 CRD `imageverificationpolicies` 未被清理，导致短名 `ivpol` 冲突，新的 CRD `imagevalidatingpolicies` 无法建立。
