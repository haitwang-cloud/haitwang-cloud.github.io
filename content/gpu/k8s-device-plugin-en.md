+++
title = 'Kubernetes GPU Management Basics: Introduction to Device Plugin and Source Code Analysis'
date = 2024-06-27T16:50:15+08:00
draft = false
+++

## Introduction

![plugin01](/pics/plugin01.png)

In a Kubernetes cluster, certain workloads require access to hardware resources on the nodes, such as GPUs, FPGAs, and InfiniBand. To enable Kubernetes to recognize and schedule these hardware resources, the concept of Device Plugins was introduced. This article will explain the working principles of Device Plugins in detail.

## What is a Device Plugin?

A Device Plugin is a special type of plugin in Kubernetes that manages hardware devices on a node and presents device information to Kubernetes. Each type of hardware device requires a corresponding Device Plugin process to be running, typically provided by the hardware vendors.

Device Plugins communicate with kubelet through a gRPC service and perform two main functions:

1. Discovering hardware devices on the node
2. Preparing and setting up the runtime environment for containers that need to use the devices

## How Device Plugins Work

![plugin02](/pics/plugin02.png)

Let's explore the workflow of Device Plugins with a concrete example. Suppose a node has three NVIDIA GPU devices installed, and we want to let a Pod use one of these GPU devices.

### Device Discovery

When a node starts, kubelet first initiates all the Device Plugins on that node. For NVIDIA GPU devices, kubelet will start the `nvidia-device-plugin` process.

This Device Plugin uses the gRPC `ListAndWatch` remote procedure call to periodically report the list of available GPU devices on the node to kubelet. In our example, the list would be [GPU0, GPU1, GPU2]. Note that this list only contains the IDs of the GPU devices and does not include any detailed information about the devices themselves. Kubelet stores these device IDs in memory and exposes the number of GPUs as Extended Resources via the API Server.

### Scheduling Decisions

When a user wants to create a Pod that uses a GPU, they simply need to add a `limits` section in the Pod specification, specifying the required number of GPUs, like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vector-add
spec:
  restartPolicy: OnFailure
  containers:
    - name: cuda-vector-add
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1 # request 1 GPU

```

### Device Allocation

![plugin03](/pics/plugin03.png)

Once a node is chosen to run the GPU Pod, kubelet on that node allocates the actual GPU device to the Pod's container. Kubelet sends a gRPC Allocate request to the nvidia-device-plugin, specifying the GPU device ID list.

The nvidia-device-plugin retrieves more details about the specified GPU devices, such as device file paths and driver directories, and returns this information to kubelet via the gRPC response.

In kubelet, handling GPU requests and allocation involves the following code:
```go
if pod.Spec.Containers[0].Resources.Limits["nvidia.com/gpu"] == 1 {
    // search for available GPUs
    deviceIDs, err := findAvailableGPUs()
    if err != nil {
       // handle error
    }
    // allocate the GPU device
    _, err = devicePluginClient.Allocate(context.TODO(), &pluginapi.AllocateRequest{ContainerRequests: []*pluginapi.ContainerAllocateRequest{{DevicesIDs: deviceIDs}}})
    if err != nil {
       // handle error
    }
}
```

### Container Initialization

After kubelet receives the GPU device information from the `nvidia-device-plugin`, it can create the proper runtime environment for the Pod's container. Kubelet uses the device file paths as the container's `Devices` parameter and the driver directories as the container's `Volume Mounts` parameter. These are then passed to the container runtime, such as Docker.

The container runtime uses these parameters to start the container and mount the device resources, allowing the Pod's application to access the assigned GPU device.

### Source Code Analysis

For a deeper understanding, let's analyze the source code from [here](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/cm/devicemanager/manager.go).

Kubernetes uses device plugins to manage hardware devices, including GPUs. Device plugins enable third-party resources like GPUs to be recognized and managed by Kubernetes. The core component of this mechanism is `ManagerImpl`, which is written and run by Kubernetes in kubelet.

- The `ManagerImpl` structure manages device plugins. It stores various information about the device plugins, including `endpoints`, which contains endpoint information for each device plugin. Each endpoint represents a device plugin interface. `healthyDevices` stores all healthy device resource names and IDs. `unhealthyDevices` maps device names to sets of device IDs for all unhealthy devices. `allocatedDevices` maps device names to sets of device IDs for all devices that have been allocated (are in use).

By understanding the roles and workflow of Device Plugins, you can efficiently manage hardware resources in your Kubernetes clusters, ensuring that various workloads are met effectively.

```go
type ManagerImpl struct {
	checkpointdir string

	endpoints map[string]endpointInfo // Key is ResourceName
	mutex     sync.Mutex

	server plugin.Server

	// activePods is a method for listing active pods on the node
	// so the amount of pluginResources requested by existing pods
	// could be counted when updating allocated devices
	activePods ActivePodsFunc

	// sourcesReady provides the readiness of kubelet configuration sources such as apiserver update readiness.
	// We use it to determine when we can purge inactive pods from checkpointed state.
	sourcesReady config.SourcesReady

	// allDevices holds all the devices currently registered to the device manager
	allDevices ResourceDeviceInstances

	// healthyDevices contains all the registered healthy resourceNames and their exported device IDs.
	healthyDevices map[string]sets.Set[string]

	// unhealthyDevices contains all the unhealthy devices and their exported device IDs.
	unhealthyDevices map[string]sets.Set[string]

	// allocatedDevices contains allocated deviceIds, keyed by resourceName.
	allocatedDevices map[string]sets.Set[string]

	// podDevices contains pod to allocated device mapping.
	podDevices        *podDevices
	checkpointManager checkpointmanager.CheckpointManager

	// List of NUMA Nodes available on the underlying machine
	numaNodes []int

	// Store of Topology Affinities that the Device Manager can query.
	topologyAffinityStore topologymanager.Store

	// devicesToReuse contains devices that can be reused as they have been allocated to
	// init containers.
	devicesToReuse PodReusableDevices

	// pendingAdmissionPod contain the pod during the admission phase
	pendingAdmissionPod *v1.Pod

	// containerMap provides a mapping from (pod, container) -> containerID
	// for all containers in a pod. Used to detect pods running across a restart
	containerMap containermap.ContainerMap

	// containerRunningSet identifies which container among those present in `containerMap`
	// was reported running by the container runtime when `containerMap` was computed.
	// Used to detect pods running across a restart
	containerRunningSet sets.Set[string]
}
```

- The `Allocate` method is called when a Pod requests device resources from a device plugin, such as a GPU. This method uses `container.Resources.Limits` to determine the quantity of each device resource needed. It then calls the `devicesToAllocate` method for each device type, which returns a list of devices that require allocation. For each device resource type, the `allocateContainerResources`  method is called. This method first tries to allocate devices from the set of healthy devices, updates the podDevices state, and initiates the Allocate request. If the allocation fails, the process rolls back.

```go
func (m *ManagerImpl) Allocate(pod *v1.Pod, container *v1.Container) error {
	// The pod is during the admission phase. We need to save the pod to avoid it
	// being cleaned before the admission ended
	m.setPodPendingAdmission(pod)

	if _, ok := m.devicesToReuse[string(pod.UID)]; !ok {
		m.devicesToReuse[string(pod.UID)] = make(map[string]sets.Set[string])
	}
	// If pod entries to m.devicesToReuse other than the current pod exist, delete them.
	for podUID := range m.devicesToReuse {
		if podUID != string(pod.UID) {
			delete(m.devicesToReuse, podUID)
		}
	}
	// Allocate resources for init containers first as we know the caller always loops
	// through init containers before looping through app containers. Should the caller
	// ever change those semantics, this logic will need to be amended.
	for _, initContainer := range pod.Spec.InitContainers {
		if container.Name == initContainer.Name {
			if err := m.allocateContainerResources(pod, container, m.devicesToReuse[string(pod.UID)]); err != nil {
				return err
			}
			if !types.IsRestartableInitContainer(&initContainer) {
				m.podDevices.addContainerAllocatedResources(string(pod.UID), container.Name, m.devicesToReuse[string(pod.UID)])
			} else {
				// If the init container is restartable, we need to keep the
				// devices allocated. In other words, we should remove them
				// from the devicesToReuse.
				m.podDevices.removeContainerAllocatedResources(string(pod.UID), container.Name, m.devicesToReuse[string(pod.UID)])
			}
			return nil
		}
	}
	if err := m.allocateContainerResources(pod, container, m.devicesToReuse[string(pod.UID)]); err != nil {
		return err
	}
	m.podDevices.removeContainerAllocatedResources(string(pod.UID), container.Name, m.devicesToReuse[string(pod.UID)])
	return nil
}
```

- The `GetCapacity` method, which is called by the kubelet to get the capacity of the device plugin, returns the total number of devices available for each resource type. This information is used to determine whether a pod can be scheduled on a node based on the requested resources.

```go
func (m *ManagerImpl) GetCapacity() (v1.ResourceList, v1.ResourceList, []string) {
	needsUpdateCheckpoint := false
	var capacity = v1.ResourceList{}
	var allocatable = v1.ResourceList{}
	deletedResources := sets.New[string]()
	m.mutex.Lock()
	for resourceName, devices := range m.healthyDevices {
		eI, ok := m.endpoints[resourceName]
		if (ok && eI.e.stopGracePeriodExpired()) || !ok {
			// The resources contained in endpoints and (un)healthyDevices
			// should always be consistent. Otherwise, we run with the risk
			// of failing to garbage collect non-existing resources or devices.
			if !ok {
				klog.ErrorS(nil, "Unexpected: healthyDevices and endpoints are out of sync")
			}
			delete(m.endpoints, resourceName)
			delete(m.healthyDevices, resourceName)
			deletedResources.Insert(resourceName)
			needsUpdateCheckpoint = true
		} else {
			capacity[v1.ResourceName(resourceName)] = *resource.NewQuantity(int64(devices.Len()), resource.DecimalSI)
			allocatable[v1.ResourceName(resourceName)] = *resource.NewQuantity(int64(devices.Len()), resource.DecimalSI)
		}
	}
	for resourceName, devices := range m.unhealthyDevices {
		eI, ok := m.endpoints[resourceName]
		if (ok && eI.e.stopGracePeriodExpired()) || !ok {
			if !ok {
				klog.ErrorS(nil, "Unexpected: unhealthyDevices and endpoints are out of sync")
			}
			delete(m.endpoints, resourceName)
			delete(m.unhealthyDevices, resourceName)
			deletedResources.Insert(resourceName)
			needsUpdateCheckpoint = true
		} else {
			capacityCount := capacity[v1.ResourceName(resourceName)]
			unhealthyCount := *resource.NewQuantity(int64(devices.Len()), resource.DecimalSI)
			capacityCount.Add(unhealthyCount)
			capacity[v1.ResourceName(resourceName)] = capacityCount
		}
	}
	m.mutex.Unlock()
	if needsUpdateCheckpoint {
		if err := m.writeCheckpoint(); err != nil {
			klog.ErrorS(err, "Error on writing checkpoint")
		}
	}
	return capacity, allocatable, deletedResources.UnsortedList()
}
```

## Understanding Kubernetes Device Plugins: A Detailed Guide

### The Design of `ManagerImpl`

The design of `ManagerImpl` provides an abstraction for device plugins, enabling Kubernetes to manage and schedule devices. It also allows for the dynamic operation of device plugins and adapts to changes in device status, such as device loss or plugin process restarts.

## Summary

From the analysis above, we can see that Device Plugins in Kubernetes act as "translators" for hardware devices:

1. They convert hardware device information on nodes into Extended Resources, which the scheduler can use.
2. They translate container requests for hardware devices into actual device files and driver directories, passing this information to the container runtime.

This plugin-based design gives Kubernetes strong support for hardware resources. Whether dealing with existing or new hardware devices, as long as there is a corresponding Device Plugin, the Kubernetes cluster can recognize and use the hardware.

Through this article, you should now have a deeper understanding of how Kubernetes Device Plugins work. If you have any questions, feel free to leave a comment for discussion.

## References

- [https://techmythos.substack.com/p/a-beginners-guide-to-building-a-kubernetes](https://techmythos.substack.com/p/a-beginners-guide-to-building-a-kubernetes)
- [https://kubernetes.io/blog/2022/12/19/devicemanager-ga/](https://kubernetes.io/blog/2022/12/19/devicemanager-ga/)
- [https://www.intel.com/content/www/us/en/developer/articles/technical/device-plugins-path-faster-workloads-in-kubernetes.html](https://www.intel.com/content/www/us/en/developer/articles/technical/device-plugins-path-faster-workloads-in-kubernetes.html)
- [https://artifacts.opnfv.org/ra2/jenkins-cntt-tox-ra2-1376/chapters/chapter03.html](https://artifacts.opnfv.org/ra2/jenkins-cntt-tox-ra2-1376/chapters/chapter03.html)
- [https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)