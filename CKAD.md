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

### Security Contexts:
- Usage:
```yaml
spec:
  containers:
    securityContext:
      runAsUser: 1000
      capabilities:
        add: ["MAC_ADMIN"] # only supported at pod level.
```

### Resource Requirements
1. Requests and limits

```yaml
spec:
  containers:
    resources:
      requests:
        memory: "1Gi"
        cpu: 1
      limits:
        memory: "2Gi"
        cpu: 2
```

2. Limit ranges

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit
spec:
  limits:
  - default
      cpu: 500m 
    defaultRequest:
      cpu: 500m
    max:
      cpu: "1"
    min:
      cpu: 100m
    type: container
```

### Service Accounts
```bash
kubectl create serviceaccount sa
```
- For Kubernetes 1.24 and later, you need to generate token manually:
```bash
kubectl create token sa
```
- Update service account for deployment:
```bash
kubectl set serviceaccount deploy/web-dashboard dashboard-saf
```

### Node Selectors
```yaml
spec:
  containers:
  nodeSelector:
    size: Large
```
- Now let's label the node:
```bash
kubectl label nodes node1 size=Large
```

### Node Affinity
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - node-1
  containers:
  - name: nginx
    image: nginx
```

### Checking Node Labels
```bash
kubectl get nodes --show-labels
```

### Checking Taints on Nodes
```bash
k describe nodes nodename | grep -i taints
```

### Readiness Probe
- Under Containers, on same level as container specs:
```yaml
readinessProbe:
  httpGet:
    path: /api/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 8

  # for TCP
  tcpSocket:
    port: 3306

 # command verification
  exec:
    command:
      - cat
      - /app/is_ready
```

### Liveness Probe
- Under Containers, on same level as container specs:
```yaml
containers:
  - image: nginx
  livenessProbe:
    httpGet:
      path: /api/healthy
      port: 8080
```

### Container Logs from Multi Container Pods
```bash
kubectl logs pod1 containername
```

### Labels & Selectors
- Listing pods having a certain label:
```bash
kubectl get pods --selector app=app1
```

### Services

- NodePort
```yaml
apiVersion: V1
kind: Service
metadata:
  name: serviceexp
spec:
  type: NodePort
  ports:
    - targetPort: 80
      port: 80
      nodePort: 30002
  selector:
    app: myapp
    type: frontend
```

### Helm
- To install on a linux host.
```bash
sudo snap install helm --classic
```
- Searching a chart:
```bash
helm search hub wordpress
helm search repo name
```
- Adding chart from a repo in Helm
```bash
helm repo add name url
```
- Listing repos:
```bash
helm repo list
```
- Listing packages installed by helm:
```bash
helm list
```
- Downloading charts
```bash
helm pull -untar bitnami/apache
```

### Kustomize
- Transformers:
  - commonLabel
  - namePrefix/Suffix
  - Namespace
  - commonAnnotations
  - images:


### Quick Notes:
- Names and labels are children of metadata.
- Lables is a dictionary under metadata dictionary.
- There can be any key or value literal in labels.
- Blue Green Deployment: Create 2nd deployment, and then update reference in the corresponding service to route traffic to it.
- Canary: Have two deployments, with canary one with less pods so that less traffic can be entertained by them.

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

