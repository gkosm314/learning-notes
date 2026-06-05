- Although some pods work independently, most **pods need to respond to requests** (from other pods or from external clients)
- Pods need a way to find other pods in order to send them requests.
# Services
## Why?
- In the non-Kubernetes world, **sysadmins would configure each client by specifying the IP address** of the server.
- In the Kubernetes world, this does not work, because:
	- pods are **ephemeral** (examples: pod evicted from node, node failed, replicas terminated because of scale down)
	- an **IP address is assigned to the pod after it has been scheduled** to a node and before starting
	- **multiple pods may provide the same service** (replicas)
- In Kubernetes it is **not possible to configure the client with the IP address** upfront. Even if we could, **we want the service to be accessible from a single IP address** instead of multiple IP addresses.
## Services
### Services
- A **Service** is a resource that provides a **single and constant point of entry** to a group of pods.
- Each service has a **IP address** (`ClusterIP`) that **never changes** while the service exists.
- Services can be backed by more than one pod. A service is backed by a **sets of pods that match a certain label selector**.
	- same mechanism as ReplicaSets
![[Pasted image 20260507111522.png|400]]
### Ports
- **Clients open connections** to the service's IP in combination with a port. These connections are **routed to one of the pods backing the service.**
	- `port` the port this service will be available on
	- `targetPort` the container port to which the service will forward the requests
- **Services can support multiple ports**.
	- example: 80 -> 8080 for HTTP, 443 -> 8443 for HTTPS
	- label selector is applied to the pod as a whole, cannot be configured per port - this requires a new service
- If the container ports are named, we can use their names to refer to them in `targetPort`
	- allows to change port numbers later without having to change the service spec
### Endpoints
- Although the **pod selector** is defined in the service spec, it is **not used directly when redirecting incoming connections**.
	- the selector is used to build a list of IPs and ports, which is then stored in a **Endpoints resource**
	- when a client connects to a service, the service proxy selects one of those IP and port pairs and redirects the incoming connection
	- it is allowed to start a Service without a pod selector and manually create the Endpoints resource so that it points to custom IPs
- An **Endpoints** resource **is a list of IP addresses and ports backing a service**.
	- a **separate resource** and not an attribute of a service
	- the name of the Endpoints resource must match the name of the corresponding Service resource
- Changes to the pod selector of a Service resource:
	- if we delete the pod selector from a Service, then the current Endpoints remain but they are not managed anymore (=not automatically updated)
	- if we add a pod selector to a Service that did not have one, its Endpoints will become managed and they will be changed accordingly by the control plane
### Session Affinity
- If we send multiple requests we will notice that each request gets randomly routed to one of the pods.
- We can makes the service proxy redirect all requests originating from the same client IP to the same pod by setting `sessionAffinity` property to `ClientIP`.
- There is no support for cookie-based session affinity because **services operate at the TCP/UDP level** and cookies are a HTTP construct => services don't even know about them.
## Discovering Services
- We now have a single and constant IP:port pair for our services. But **how will the other pods learn about it**?
- It is not practical to create the service and then manually find its `ClusterIP` and copy it to the pod's configuration.
- Kubernetes allows us to discover services using **environment variables** or **DNS**.
### Discovery with environment variables
- **When a pod is started, Kubernetes initializes environment variables that point to each service** that exists at the moment.
- Pods that were created before a service will not have the environment variables that point to it.
- Example: for service `my-database` the env variables are called `MY_DATABASE_SERVICE_HOST` and `MY_DATABASE_SERVICE_PORT`.
### Discovery with DNS
- `kube-dns` is a pod in the `kube-system` namespace that runs a DNS server.
	- not a control-plane component but a required add-on
	- all other pods running in the cluster are automatically configured to use it
		- achieved by modifying each container's `/etc/resolv.conf` file
	- whether a pod uses the internal DNS server or not is configurable through the `dnsPolicy` property in its spec
- **Each service gets a DNS entry in the internal DNS server.**
	- the FQDN for a service is `<service-name>.<namespace>.svc.<custom-cluster-domain-suffix>`
	- the default for the configurable cluster domain suffix is `cluster.local`
- Clients that know the name of the service can access it through its fully qualified domain name (FQDN) instead of using env variables.	
	- the suffix can be omitted, and the namespace can also be omitted when the calling pod is in the same namespace as the service
		- DNS resolver is setup with `search <namespace>.svc.cluster.local svc.cluster.local cluster.local ...`
	- note: discovery through DNS does not allow you to discover ports, but for most use cases there is a standard port (HTTP etc)=
### Headless services
- Services forward each connection to one pod. **What if a client wants to connect to all the pods?**
	- example: peer-to-peer network
	- we could make a call to the K8S API and ask for the pods, but we want our applications K8S-agnostic
- When you perform DNS lookup for a service, the DNS server returns a single IP (its `ClusterIP`).
- If you tell Kubernetes you don’t need a cluster IP for your service, the DNS server will return the pod IPs instead of the single service IP
	- we can do this by setting `clusterIP: None`
	- instead of a single DNS `A record`, **it will return multiple `A records`**, each pointing to the IP of an individual pod backing the service
- Clients can still connect to headless services by using the service's DNS name.
	- it is not possible to connect to headless services using an IP, which also means that there is no environment variables discovery
	- load balancing is still provided through the DNS round-robin mechanism (results are rotated) instead of through the service proxy (probabilistic rules in iptables)
## Communication between pods inside the cluster
- The service will get assigned a **`ClusterIP`**. This IP is **only accessible within the cluster**.
- Example of executing `curl` from inside the container of a pod to send an HTTP request to another pod:
	- `curl` sends an HTTP request to the Service IP
	- Service selects a random pod among the pods from which it's backed and forwards the request
	- The pod processes the request and sends an HTTP response **directly to the pod that sent it**.
![[Pasted image 20260507125820.png|400]]
## Communication with services living outside the cluster
- We can create a Service without a pod selector and manually create an Endpoints resource that points to the external server. 
	- after both of them are posted to the server, the service is ready to be used like a regular service defined with a selector.
	- if we decide to switch from an external service to internal pods, we can add a selector and the Endpoints controller will make the necessary changes

![[Pasted image 20260507162032.png|400]]
### Alias to external service
- You can create a Service that **serves as an alias for an external service**.
	- Define the Service with `type: ExternalName` and `externalName: someapi.somecompany.com`.
- Pods can **use the service's FQDN to connect to the external service**, instead of using the actual FQDN.
	- The service act as an abstraction that hides that actual FQDN, allowing you to switch to another FQDN or even to pods inside the cluster.
- `ExternalName` services are implemented solely at the DNS level.
	- A simple `CNAME record` is created for the service.
	- A `CNAME record` points to a fully qualified domain name instead of a numeric IP address.
	- The pod will look up the `ExternalName` service FQDN, receive the `CNAME record`, look up the actual FQDN and then connect to the IP. 
## Exposing services outside of the cluster
- Other than exposing services to clients inside the cluster, we also want to **serve clients outside of the cluster**.
- We have three ways to achieve that:
	- creating a NodePort service
	- creating a LoadBalancer service
	- creating an Ingress
### NodePort/LoadBalancer
- **Makes each cluster node open a port to the node itself** and **redirects the traffic received on that port to the service**.
	- use `type: Nodeport` and add `nodePort: ...` field in the ports
	- the service will be accessible at `<Service_ClusterIP>:<port>` and at `<Node1_IP>:<nodePort>, <Node2_IP>:<nodePort>,...`
	- connections will be redirected to the service and from there to a randomly selected pod that will receive them to its `targetPort`
- The randomly selected pod that will receive the connection **may not be the running on that node**.
	- to avoid the extra network hop, we can use `externalTrafficPolicy: Local` (if possible redirect to a local pod, else hang the connection)
	- => LoadBalancer must redirect the traffic only to nodes that include pods, but...
	- assume node A runs 1 pod and node B runs 2 pods, If the LB splits the traffic evenly, then the pod on node A will receive more traffic
- **If you point your clients to a specific node, then if this node fails, all the requests from that client will fail.**
	- => we have to point our client to a **LoadBalancer** that will redirect the traffic to all the healthy nodes
	- cloud providers usually support the automatic provision of a load balancer with a unique, publicly accessible IP address.
	- if the LoadBalancer cannot be provisioned, the service will fall back to acting as a NodePort (LoadBalancer is an extension of NodePort)
![[Pasted image 20260507194521.png|400]]
# Ingress
- **Requirement** for Ingress resources: an **Ingress controller** 
## Why
- Each LoadBalancer service requires its own load balancer with its own public IP address. Ingress requires only one for N services.
- Operates at the application layer of the network stack (HTTP), so it can provide features such as cookie-based session affinity.
	- note: HTTPS is also supported
## Rule-based routing
- When defining an Ingress, you **define rules** that perform **host -> service traffic redirection**.
	- note that the traffic is **redirected directly to the pod**
- For the Ingress to work, the host must resolve to the IP of the Ingress.
	- either configure the client or configure the DNS server
- It is possible to expose different services to different paths of the same hosts, and to expose different hosts.
## Example
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
  - host: kubia.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubia-nodeport
          servicePort: 80
```
![[Pasted image 20260511220217.png|450]]
# Readiness Probes
## Why?
- A pod that matches the selector of a service becomes part of the service immediately after it is created.
- This means that requests are being redirected to it. **What if the pod is not ready to process requests immediately?**
	- example: initialization takes time, it needs to download something, ...
- Readiness probes allow Kubernetes to **check if a container is still ready** by **periodically executing** the probe.
	- similar to liveness probes
## About liveness probes
 -  It is specified in the pod's spec (`readinessProbe`).
 - Three probing mechanisms (same as the liveness probes):
	 - *HTTP* - send an HTTP GET request to a specified port and path, check if a non-erroneous response was returned 
	 - *TCP* - try to open a TCP connection with a port, check if it was successfully established
	 - *Exec* - execute a command inside the container, check exit status code
 - if a pod’s readiness probe fails, the pod is removed from the Endpoints object.
## Good practices
 - Always define a readiness probe.
 - Don't use the readiness probe as a way to manually remove a pod from the service.
 - Don't include pod shutdown logic in the readiness probe. The pod is removed from the service as soon as it is deleted.

See also Liveness Probes
# Gotchas
- You cannot ping services. Their IP is a virtual IP that only has meaning when combined with the service port. (more about this later)
	- this is related to the fact that the Service does not actually run somewhere (like for example a pod)