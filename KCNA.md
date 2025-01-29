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