# CKAD


## Core Concepts:
- Nodes were initially called minions.

### Base64 Encoding and Decoding
```bash
echo -n 'mysql' | base64

echo -n 'XHFDHE' | base64 --decode
```

Information Provider Commands

```bash
kubectl cluster-info
```

```bash
kubectl get nodes
```

### Kubectl Set Image
- Kubectl set image pod/redis redis=redis
- Here it changes image in redis container in redis pod.

### Scale Replicaset
- Kubectl scale --replicas=5 rs/rsname

## KillerShell Questions and Solutions:

1. Usage of parallelism and completions fields in Jobs. Busybox container can be used with `command:` field to execute certains commands.

###
- To install 

### Generate YAML boilerplates
- kubectl run nginx --image=nginx --dry-run=client -o yaml
- kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml

### Generate Other Output Types
- **kubectl [command] [TYPE] [NAME] -o **
- `-o json` Output a JSON formatted API object.
- `-o name` Print only the resource name and nothing else.
- `-o wide` Output in the plain-text format with any additional information.
- `-o yaml`  Output a YAML formatted API object.

### Commandline & Args for Pods
- Numbers should be enclosed in quotation marks.
- Pod needs to be deleted and recreated to change command.
- The ENTRYPOINT in the Dockerfile is overridden by the command in the pod definition file.
```yaml
apiVersion: v1
kind: Pod 
metadata:
  name: ubuntu-sleeper-2
spec:
  containers:
  - command:
    - sleep
    - "5000"
    name: ubuntu
    image: ubuntu
```

- OR

```yaml
spec:
  containers:
  - name: simple-webapp
    image: kodekloud/webapp-color
    command: ["python", "app.py"]
    args: ["--color", "pink"]
```

### Environment Variables
- Under Continers[containernum].env
- Can also be used using secrets and configmaps.
```yaml
spec:
  containers:
  - name: example
    image: nginx
    ports:
      - containerPort: 8080
    env:
      - name: JAVA_HOME
        value: /opt/java
       #valueFrom:
         #configMapKeyRef:
         #secretKeyRef:
           #name: key
```

### ConfigMaps
- Used for centralized management of configurations for pod containers.
- Imperative creation:
```bash
kubectl create cm frontend --from-literal=port=8080
```

### Secrets
- Secrets are namespace scoped.
- Imperative creation:
```bash
kubectl create secret generic example --from-literal=DB_Host=mysql
```
- Using a Secret:

``` yaml
spec:
  containers:
    envFrom:
      - secretRef:
        name: my-secret
```

- Using Single Value from Secret:

```yaml
env:
  - name: DB_Password
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: DB_Password
```

- Using Secrets as Volume:

```yaml
volumes:
- name: app-secret-volume
  secret:
    secretName: my-secret
```

> When secret is mounted as a volume, all keys are created as files in that volume with their values as content of those files.



### Quick Notes:
- Names and labels are children of metadata.
- Lables is a dictionary under metadata dictionary.
- There can be any key or value literal in labels.

### Must needed fields for pod:
- apiVersion
- kind
- metadata
- spec

### API Versions for Different Types of Kubernetes Resources
- ✅ Core Resources → v1 (Pods, Services, ConfigMaps, Secrets, PVC, PV, Namespace)
- ✅ Workloads → apps/v1 (Deployment, ReplicaSet, StatefulSet, DaemonSet)
- ✅ Jobs & Autoscaling → batch/v1 (Job, CronJob), autoscaling/v2 (HPA)
- ✅ Networking & RBAC → networking.k8s.io/v1 (Ingress, NetworkPolicy), rbac.authorization.k8s.io/v1 (Roles, Bindings), storage.k8s.io/v1 (StorageClass)

## Exam Tricks:
1. Do not forget to set context.
2. Using vim, use set paste command to paste stuff without affecting the indentation.
3. 4

