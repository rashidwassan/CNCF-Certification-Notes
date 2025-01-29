# KCNA

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

### ContainerD
- Comes with commandline tool `ctr` for debugging, has limited functionality.