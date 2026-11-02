# Volumes
## Why?
- **Containers running inside the same pod share resources** like CPU, RAM and network interfaces.
- **They don't share disks**. Each container has its own **isolated filesystem** that comes **from the container's image**.
- This means that when a container is started/restarted, it cannot see that was written to the filesystem by the previous container (even though they run on the same pod).
## Volumes
- Volumes are defined as part of a pod. They are **not top-level resources**.
- Volumes have the **same lifecycle as the pod** => they persist across container restarts.
	- process of **mounting** or populating a volume is **performed before the pod's container's are started.**
- **The volume must be mounted in each container that needs access to it.**
	- volumes are available to all containers of the pod at the same time => **shared storage between containers**
	- in each container the volume can be mounted in any location of its filesystem (`VolumeMount`)
### Types
- Multiple volume types exist. Some are generic while others are storage-specific.
	- `emptyDir` - simple empty directory used for storing transient data
	- `hostPath` - used to mount directories from the node's filesystem into the pod
	- `nfs` - NFS share mounted into the pod
	- `gcePersistentDisk`,`awsElasticBlockStore`,`azureDisk` - cloud provider-specific storage
	- `cinder, cephfs, ...` - a bunch of other types of network storage
	- `configMap, secret, downwardAPI` - special types that expose certain K8S resources to the po
- Configuration options
	- you can define a volume mount as read-only
	- you can specify whether the volume will be created on disk or in memory (`medium`)
- Even though `emptyDir` and other volume types create a new directory, some of the **other volume types mount an existing directory** into the filesystems of the pod's containers.

## Accessing files on the node's filesystem
- A `hostPath` volume points to a specific file or directory on the node’s filesystem.
- Pods running on the same node and using the same path in their hostPath volume see the same files. => **shared storage between pods running on the same node**
- This is a type of persistent storage
	- `emptyDir` gets deleted when the pod is deleted
- `hostPath` volumes should only be used to read/write files from the node. They can't be used to persist data across pods.
## Persistent Storage
- When a pod needs to persist data to disk and have that same data available even when the pod is rescheduled to another node, they must be stored to some type of network-attached storage (NAS). This will make them available from every cluster node.
- You can create a volume that points to a persistent storage solution of your choice (e.g. GCE Persistent Disk, Azure Disc, ...) and then just mount that volume to your container.
	- every container will then point to that persistent storage, regardless of which node they run on
	- for details on how to define the volume properly, refer to the Kubernetes reference and study the relevant API

## Decoupling pods from the underlying storage technology
- Pointing pods to persistent storage works. However, it goes against the basic idea of Kubernetes to hide the infrastructure.
	- developers have to deal with infra-specific storage details
	- the pod definiton becomes cluster-specific
- Ideally, we want developers to request persistent storage from Kubernetes in the same way they request CPU/memory.
- To achieve this, Kubernetes provides two resources: **PersistentVolumes** and **PersistentVolumeClaims**.

### Using PVs and PVCs
![[Pasted image 20260703192351.png]]
- admin creates a PV = registers a storage solution and provides an abstraction
	- points to the storage and specifies size, access mode, what to do when released, ...
- user creates a PVC = asks to use storage and specifies minimum size and access mode
	- Kubernetes finds the appropriate PV and binds it to the PVC
- user creates a pod that uses the PVC as a volume

- PersistentVolumes don’t belong to any namespace. They’re cluster-level resources like nodes.
- Creating a PVC is a completely separate process from creating a pod, because you want the same PVC to stay available if the pod is rescheduled.

Q: 