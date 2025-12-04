+++
title = '# Kyverno ImageValidatingPolicy CRD Conflict Issue and Solution'
date = 2025-12-04T10:09:44+08:00
draft = false
+++

# Kyverno ImageValidatingPolicy CRD Conflict Issue and Solution

In a Kubernetes cluster, `Kyverno` is a powerful policy engine for managing and validating resources. However, during the upgrade of Kyverno from **1.13** to **1.15**, we encountered a complex CRD conflict issue that caused the `ImageValidatingPolicy` to malfunction. This article documents the complete process from the misleading investigation into permission issues to the eventual discovery of the CRD conflict.

---

## Background

After upgrading Kyverno, the following errors appeared in the controller logs:

```bash
2025-12-03T06:08:13Z ERR ... Failed to watch error="failed to list *v1alpha1.ImageValidatingPolicy: the server could not find the requested resource (get imagevalidatingpolicies.policies.kyverno.io)"
2025-12-03T06:08:19Z ERR ... no matches for kind "ImageValidatingPolicy" in version "policies.kyverno.io/v1alpha1"
2025-12-03T06:09:59Z ERR ... failed to wait for imagevalidatingpolicy caches to sync
```


These errors indicate:

- The Kyverno controller cannot access `imagevalidatingpolicies.policies.kyverno.io`.
- The API Server is not correctly serving this resource type.

## Initial Investigation: Misleading Permission Issues

### 1. Checking ServiceAccount and RBAC Configuration

First, we checked the `ServiceAccount` and `ClusterRoleBinding` configuration for the `Kyverno Admission Controller`:

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

The `ClusterRole` rules included access permissions for `imagevalidatingpolicies`:

```yaml
rules:
- apiGroups: ["policies.kyverno.io"]
  resources: ["imagevalidatingpolicies"]
  verbs: ["get", "list", "watch"]
```

Upon inspection, the RBAC configuration was correct.

### 2. Verifying Permissions with `kubectl auth can-i`

We further verified whether the `ServiceAccount` had permissions to access the resource:

```bash
kubectl auth can-i list imagevalidatingpolicies.policies.kyverno.io --as=system:serviceaccount:ssdl-kyverno:kyverno-admission-controller

Warning: the server doesn't have a resource type 'imagevalidatingpolicies' in group 'policies.kyverno.io'
no
```

The result returned `no`, with a warning that the resource type does not exist. This result misled us into focusing on permission issues, but the root cause was not RBAC.

## In-depth Investigation: CRD Conflict Preventing Resource Availability

### 1. Checking if the CRD Exists

We checked whether the CRD was installed using the following command:

```bash
kubectl get crd imagevalidatingpolicies.policies.kyverno.io
```

The result showed that the CRD existed, but the issue persisted.

### 2. Checking if the API Resource is Available

We checked whether the API Server was serving the resource:

```bash
kubectl api-resources --api-group=policies.kyverno.io
```

### 3. Inspecting the CRD Status

We further examined the detailed status of the CRD:

```bash
kubectl get crd imagevalidatingpolicies.policies.kyverno.io -o yaml
```

The critical part of the output was as follows:

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

The findings were:

- `NamesAccepted=False`: The short name `ivpol` was already in use.
- `Established=False`: The CRD was not fully established, and the API Server could not serve the resource.

### 4. Confirming the Short Name Conflict

We identified the CRD occupying the short name `ivpol` using the following command:

```bash
kubectl get crd -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.names.shortNames}{"\n"}{end}' | grep -w ivpol
```

The output was:

```bash
imagevalidatingpolicies.policies.kyverno.io ["ivpol"]
imageverificationpolicies.policies.kyverno.io ["ivpol"]
```

#### Conflict Source

- In Kyverno 1.13, the CRD `imageverificationpolicies.policies.kyverno.io` used the short name `ivpol`.
- In Kyverno 1.15, `imageverificationpolicies` was deprecated and replaced by the new CRD `imagevalidatingpolicies.policies.kyverno.io`, but the Helm Chart still assigned the same short name `ivpol`.
- During the upgrade, the old CRD `imageverificationpolicies` was not cleaned up, causing a short name conflict that prevented the new CRD `imagevalidatingpolicies` from being established.

## Solution

Delete the old CRD `imageverificationpolicies.policies.kyverno.io` to release the short name `ivpol`:

```bash
kubectl delete crd imageverificationpolicies.policies.kyverno.io
```

### Verifying the Fix

Check the status of the CRD `imagevalidatingpolicies.policies.kyverno.io`:

```bash
kubectl describe crd imagevalidatingpolicies.policies.kyverno.io
```

The expected output should include the following:

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

Next, restart all `kyverno-admission-controller` Pods to ensure they use the latest CRD definition:

```bash
kubectl rollout restart deployment kyverno-admission-controller -n ssdl-kyverno
```

## Summary

- **Issue**: The Kyverno controller could not access `imagevalidatingpolicies`, and the API Server reported that the resource type did not exist.
- **Root Cause**: During the upgrade, the old CRD `imageverificationpolicies` was not cleaned up, causing a short name conflict (`ivpol`) that prevented the new CRD `imagevalidatingpolicies` from being established.
- **Solution**: Delete the old CRD to resolve the conflict.
