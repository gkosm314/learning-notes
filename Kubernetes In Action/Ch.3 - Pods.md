# Pods

## Definition
- A **pod** is a co-located group of containers
	- **all the containers of a pod run in a single worker node**
	- the basic building block of Kubernetes
	- we do not deploy containers individually, we deploy pods
## Containers and Pods
- Containers should not contain multiple processes.
- Pods should not contain multiple containers unless if they **must** be co-located.
	- otherwise, scheduling and scaling are restricted unnecessarily
	- most common pattern is a main process and supplementary (sidecar) processes
	- pods are relatively lightweight - we don't have to worry about overhead
- Pods are logical hosts that correspond to physical hosts/VMs in the non-container world
	- processes in a pod <->  processes in physical host, except that they are containerized
## Why can't we use containers directly?
- We want to run processes A and B that are closely-related and must be co-located.
	- because of inter-process communication, using the same local file system, etc
- Why don't we run them in the same container?
	- Because then we have to reimplement mechanisms such as restart after crash and separate logging for each process.
- To run the closely-related containers, we introduce a new high-level concept, the **pod**
	- allows us to manage the containers together
	- allows us to provide the same environment to the containers
## Isolation
### How is the isolation achieved?
- The closely-related containers should share (almost) all resources
- K8S does this by configuring Docker to make the containers use the same **Linux namespaces**
	- same Network/UTS namespaces => same hostname/network interfaces
	- same IPC namespace => communicate through IPC
	- by default, separate filesystems
### Networking
- Containers of a pod use the same Network namespace.
	- they **share the same IP address
	- they **share the same ports**
	- they must take care to avoid port conflicts
- Pods of a cluster share the same flat network-address space
	- every pod can talk to every other pod by using its IP address
	- this holds true regardless of whether the pods are located in the same node
- Port-forwarding allows you to forward your local machine's port to a pod's port.
	- this allows you to hit the pod directly without going through the service
---
## Manifests
- K8S resources are created by posting a YAML/JSON **manifest** to the Kubernetes API.
- When looking at the manifest of an existing (running) resource, we see:
	1. **Kubernetes API version**
	2. **Kind** - the kind of the resource (e.g. `Pod`,`Service`, `Deployment`, ...)
	3. **Metadata** - name, namespace, labels and other metadata for the resource
	4. **Spec** - description of the resource's content/what the manifest declaratively asked for
	5. **Status** - current info about the running state of the resource (not info from Spec)
- When writing the manifest for a new resource we want to create, we don't have to write everything (but there are fields that are necessary)
## Logging
- Docker redirects stdout/stderr and kubernetes allows us to read the logs.
- When a pod is deleted, its logs are also deleted.
	- we need a centralized logging system for the whole cluster to get over this 
- A container's logs are automatically rotated daily and every time the log file reaches 10MB.

---
## Labels/Annotations
- **Labels** are arbitrary **key-value pairs** that get attached to a resource.
- Utilized when **selecting resources through** **label selectors**.
	- contains/doesn't contain a label with a key K/set of keys K1,K2,..
	- contains a label with key K and value is equal/not equal to V
	- comma-separated list the aforementioned criteria
- **Annotations** are also key-value pairs.
	- they cannot be used for identification - no annotation selectors
	- they can hold larger pieces of information (because they are not indexed)
## Scheduling
- There are cases where when you want certain pods to run on specific nodes.
	- example: only some of the nodes have SSDs and we want IO-heavy pods to run there
- We can decorate nodes with labels and then use pod's `spec.nodeSelector`to select nodes.
- We should not schedule pods to a specific node, only to logical groups of nodes.
	- what if that specific node is offline? then the pod will be unschedulable
## Namespaces
- **Namespaces** provide a scope for resource names
	- labels do not provide non-overlapping groups and they are not practical
	- resource names only need to be unique within a namespace.
	- exception: cluster-wide resources like nodes
- Namespaces **do not provide any kind of isolation** to the running objects
	- they only provide name scoping - essentially splitting the objects to groups
	- nothing to do with the Linux namespaces that provide process isolation
	- for example, by default pods in different namespaces can send traffic to each other
- In practice:
	- When creating a resource, you can add a `namespace:` field in its metadata section
	- When using `kubectl` commands you have to use the `-n` flag, otherwise the action is performed in the default namespace of the current context.
## Stop Pods
- When deleting a pod, K8S terminates all containers of the pod.
	- send `SIGTERM`, wait X seconds for it to shutdown gracefully, send `SIGKILL`