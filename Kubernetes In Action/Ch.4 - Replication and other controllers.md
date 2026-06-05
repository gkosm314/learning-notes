- Pods are the basic deployable unit of Kubernetes. However, we almost never deploy them directly.
- If the container is not healthy, it will be restarted by the Kubelet.
- However, because the Kubelet lives on a node, this mechanism only works on the node level. But **what if the whole node crashes**?
- We need a **mechanisms** that **involve** the K8S **control plane** to extend this guarantee to cover node failures.
# Liveness Probes
## Why?
- When a pod is scheduled to a node, Kubelet will run its containers. If the container's main process crashes, it will restart it.
- However, there are cases where the process is unhealthy but the process did not crash (example: deadlock, OOM but JVM still running).
- Liveness probes allow Kubernetes to **check if a container is still healthy** by **periodically executing** the probe and **restarting** the container **if the probe fails**.
## About liveness probes
 -  It is specified in the pod's spec (`livenessProbe`). There are additional optional parameters to configure (e.g. `delay`,`timeout`,`period`,`failure`,...)
 - Three probing mechanisms:
	 - *HTTP* - send an HTTP GET request to a specified port and path, check if a non-erroneous response was returned 
	 - *TCP* - try to open a TCP connection with a port, check if it was successfully established
	 - *Exec* - execute a command inside the container, check exit status code
 - If the liveness probe fails, the container is **killed and re-created** from Kubelet. The container is killed with a `SIGKILL` signal.
## Good practices
 - For prod, always define a liveness probe. If you don't, then the only information K8S has about your container's health is whether the process is still running.
 - The liveness probe should **only fail when the internals of the applications are broken** (= can be fixed with a restart). This is not an integration test.
 - The liveness probe should be **lightweight**, since **it will run repeatedly** and we don't want to waste resources on this.
 - Don't implement retry loops, K8S does this by default.

See also Readiness Probes
# ReplicationControllers/ReplicaSets
## ReplicationControllers
- A **ReplicationController** is a resource that ensures that an **exact number of pods of a certain "type" are always running**.
	- operates on **sets of pods that match a certain label selector** (see previous chapter)
	- if the number of pods that match a label selector does not match the desired number, the controller takes appropriate action
	- provides robustness against crashed nodes and easy horizontal scaling
- A ReplicationController has three essential parts:
	- _label selector_ - determines pods in the ReplicationController's scope
	- _replica count_ - determines desired number of pods
	- _pod template_ - used when creating new pod replicas
		- pod's labels must match the label selector, label selector is equal to the pod template's labels by default
- The aforementioned parts can be modified at any time:
	- modifications to label selector => scope of the controller changes
		- maybe some pods are not monitored anymore
		- maybe some new pods are created
	- modifications to replica count => create/delete existing pods
	- modifications to pod template => only affects new pods created by the controller
		- the controller does not care about the "content" of the pods it monitors after creating them, it only cares about keeping them running
## Reconciliation Loop
![[replication_controller_reconciliation_loop.png.png|500]]
- Note that when a node is deleted, the controller does not react to the deletion itself. In detail:
	- the **controller is a client of the API server** and it **watches changes to pods**
	- **when a pod is deleted**, the **API server notifies** the controller
	- this notification triggers the **reconciliation loop**
	- the controller takes action because the actual state is different than the desired state, not because of the deletion per se
- How can we end up with
	- less pods than desired: node crashed, scheduler evicted a node, someone manually deleted a pod, ...
	- more pods than desired: the number of desired pods was decreased, someone manually created/changed a pod, ...
## ReplicaSet
- **ReplicaSet** is a resource that behaves exactly like ReplicationControllers but supports more expressive label selectors.
	- should always create ReplicaSets instead of ReplicationControllers (to be deprecated)
	- usually not created directly but as part of Deployments
# DaemonSet
- A **DaemonSet** is a resource that **ensures that all** (or some) **Nodes run a copy of a Pod**.
	- could be started by scripts on the node, but then you can't take advantage of K8S
	- example: infrastructure-related pods that perform system-level operations
	- can specify a subset of the nodes by using node labels  (`node-Selector`)
- In general, it works similarly to ReplicaController/ReplicaSet, but instead of ensuring a desired number of replicas, it ensures one pod per node is running
	- if a new node is created, it creates a pod on it
	- if a pod gets deleted, it re-creates it on the same node 
	- if a node goes down, it does not create a pod elsewhere
- Pods created by a DaemonSet already have a node specified and they skip the K8S Scheduler.
	- since they skip the scheduler, they are created even in nodes that are marked unschedulable
# Job
- A **Job** is a resource that allows you to run a **pod whose container is not restarted when** the process running inside **it finishes successfully**.
	- process failure: Job can be configured to restart the container
	- node failure: pods managed by a Job will be rescheduled to other nodes
- Successfully completed jobs are not deleted to allow log examination. They will be deleted when the Job or the pod are manually deleted.
	- completed pods do not appear in `k get pods` unless you use the `-a` flag
- Some basic Job configurations:
	- `restartPolicy: OnFailure` or `restartPolicy: Never` - process failure handling
	- `completions` - run a job more than once
	- `parallelism` - run more than one pod at once
	- `activeDeadlineSeconds` - set deadline for the pod to finish
	- `backoffLimit` - specify how many retries - the backoff is exponential
![[Pasted image 20260425165029.png|500]]
# CronJob
- A **CronJob** is a resource that creates Jobs on a repeating schedule.
	- `schedule` - uses the cron format
	- `jobTemplate` - used to create the Job
	- `startingDeadlineSeconds` - fail the Job if it did not start within X seconds from the scheduled time
- In K8S it can happen that the Job runs multiple times or that it does not run at all. Your CronJobs should be robust to this issues.
	- **idempotent Jobs** to be **robust again running multiple times**
	- **perform the work of the previous Job** to be **robust against skipped executions**