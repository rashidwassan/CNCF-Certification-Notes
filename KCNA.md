# KCNA

### Cloud Native Computing Foundation (CNCF)
- A project by Linux Foundation launched in 2015 to help advance container technology.
- Independent organization from its parent.

### Cloud Native Trail Map

![image](https://www.cncf.io/wp-content/uploads/2020/08/CNCF_TrailMap_latest-1.png)
1. Containerization
2. Continuous Integration and Deployment (CI/CD)
3. Orchestration and Application Definition
4. Observability and Analysis
5. Service Proxy, Discovery & Mesh
6. Network Policy & Security
7. Distibuted Databases & Storage
8. Streaming & Messaging 
9. Container Registry & Runtime
10. Software Distribution

- `Etcd` is fully replicated in every master node.
- Kube Api Server makes a master a master.
- Kubelet makes a worker a worker.
- Kubelet communicates to the master via API server.
### To get more info about cluster:
```bash 
    kubectl cluster-info
```

### Container Runtime
- Need of Container Runtime emerged when other tools apart from Docker came into the market.
- For a Container Runtime to be compatible with Kubernetes, it must be Open Container Initiative (OCI) compatible.
- OCI compatible CRI contains `imagespec` and `runtimespec`.
- `Dockershim` provided temporary layer for Kubernetes to support docker.

### Kubelet and CRI Communication using gRPC
- CRI defines `gRPC` protocol that Kubernetes kubelet uses to interact with container runtimes.
- Kubelet utilizes gRPC as the primary communication protocol to interact with the Container Runtime Interface (CRI).
 ![image](https://cl.ly/3I2p0D1V0T26/Image%202016-12-19%20at%2017.13.16.png)

### Docker and Kubernetes
- Docker support from Kubernetes was removed in 1.24.
- Dockershim was basically removed.
- `ContainerD`, that powered Docker, now works directly with Kubernetes.
- Docker primarily uses `LXC` container technology.

### ContainerD and CLI
- Comes with commandline tool `ctr` for debugging, has limited functionality.
- However, `nerdctl` provided a Docker-like CLI for ContainerD and supports Docker Compose.
- Nerdctl also supports new features and implementations of containerd.
- Supports Lazy Pulling.
> 😃 "Lazy pulling" in Kubernetes refers to a technique where a container image is not fully downloaded from a registry before a container is launched; instead, only the necessary parts of the image are pulled on-demand as they are needed during runtime, significantly reducing startup time and optimizing resource usage.

### crictl (installed separately) - Managed by Kubernetes Community
- A command-line tool for managing Kubernetes container runtimes that implement the Container Runtime Interface (CRI), such as containerd and CRI-O.
- Used for debugging and troubleshooting Kubernetes nodes by inspecting containers, pods, images, and logs without relying on kubectl.
- Uses **/etc/crictl.yaml** for runtime settings, typically pointing to a `CRI runtime socket` (e.g., /run/containerd/containerd.sock for containerd).
- Kubelet will delete containers that are manually created with crictl since it would be unaware of such containers.
> A container runtime socket is a Unix domain socket file that facilitates communication between the Kubernetes kubelet and the underlying container runtime.

### YAML ApiVersions & Indentation
- Pod = v1
- Service = v1
- ReplicaSet = apps/v1
- Deployment = apps/v1

> HINT: Prefer using two spaces for indentation instead of tab.

### Kubernetes: ReplicationController vs ReplicaSet
- Major difference is `selector` being required in ReplicaSet, not in Replication controller.
- Yes, you can manage replications of other pods that are not created by ReplicaSet but have matching label for selector.
- Labels in spec->selector->matchLabels === spec->template->metadata->labels
> TERMINATION STATE: ReplicaSet does not allow the manual creation of pod with same label as pods realted to that RS.
> Doing so will terminate newly created pod if the desired number of replicas are already running.

### Kubectl Replace
- Works like apply command, but recreates the resouce than making changes to existing one.
- Can be considered if working with ingress, secrets, and/or storage resources.

```bash
    kubectl replace -f ingress.yaml
```
### Rolling Updates in Deployments
- Pods are replaced by new ones, one by one, with no application downtime.
- Kubernetes creates a new ReplicaSet and keeps creating new pods there and doing the opposite in older ReplicaSet.
- `Undo` command does the opposite of second point mentioned. Upon execution, it starts creating pods in old ReplicaSet while deleting pods from the recent one.
- There can be more than one or two ReplicaSets present for each deployment, supporting cascading rollbacks.
- Commands

```bash
    kubectl rollout status deployment/my-deployment
    kubectl rollout history deploy my-deployment
    kubectl rollout undo deploy my-deployment
```

### Listing Resources Present in All Namespaces
```bash
    kubectl get pods --all-namespaces
```

### Resource Qoutas in Kubernetes

### Accessing Resources Residing in Other Namespaces
- The format will be `resourceName.namespaceName.resourceType.cluster.local`.

### Manual Scheduling
- Specify `spec->nodeName` field to manually schedule the pod on a specic node.
- To schedule an **existing pod** to a node, create a `Binding` object having meatadata.name value equal to metadata.labels.name in pod definition. Then sent a POST request to the pod's binding API containing JSON file obtained after converting binding object YAML to JSON. 
- A `binding object` refers to a specific type of resource that is used to link a Pod to a particular node within the cluster.

### Labels and Selectors in Kubernetes
- Labels are used to group or identify resources and selectors are used to select resources with specific labels.
```bash
    kubectl get pods --selector app=App1
```

### Taints & Tolerations
- Taints are used for nodes
```bash
    kubectl taint nodes <nodename> key=value:taint-effect
```
- **Taint Effects:** NoSchedule, PreferNoSchedule, NoExecute
- Tolerations are applied on pods.
```yaml
# add this in pod manifest.
spec:
  tolerations:
  - key: "app"
    operator: "="
    value: "blue"
    effect: "No Schedule"

```
- Adding No Schedule taint on a node having running pods will cause pods with no supported tolerations to evict.
- There are changes of a pod having matching tolerations get scheduled to another node as taints only make nodes to only schedule pods with matching tolerations.
- To find taints on master node:
``` bash
    kubectl describe node kubemaster | grep Taint
```

### Node Selectors
- Proivde another approach for scheduling by adding `nodeSelector` property in pod yaml file spec.
- This value must match with the label assigned to the node(s) to get that pod scheduled on that node.
- To label a node:
```bash
    kubectl label nodes <node-name> <label-key>=<label-value>
```
- In pod manifest:
```yaml
spec:
  nodeSelector:
    <label-key>: <label-value>
```
- NodeSelector only allows you to select nodes with lables, it will not provide conditional selection or more flexible selection.
> NOTE: You must label node(s) before scheduling pods on them.

### Node Affinity:
- Node affinity supports expressions like In, NotIn, Exists, for scheduling.
- Provides more features compared to NodeSelectors.
- Ensures correct scheduling compared to Taints and Tolerations where a pod with tolerations can end up being scheduled at node with no any taint.
- The syntax looks like this:
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
```
> You can use Tains & Tolerations and Affinty for more customized scheduling.