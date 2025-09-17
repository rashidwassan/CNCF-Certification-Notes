# CKAD


## Core Concepts:
- Nodes were initially called minions.
- If the questions says 'container level' means container level, not deployment level.

### Copy one File Contents to Another
```bash
cp one.yaml two.yaml
```

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

### Deployments
- Imperative Commands:
```bash
kubectl create deployment my-deploy --image=nginx --replicas=3

kubectl expose deployment my-deploy --port=80 --target-port=80 --type=ClusterIP

# Scale to 5 replicas
kubectl scale deployment my-deploy --replicas=5

kubectl set image deployment/my-deploy nginx=nginx:1.21 --record

kubectl rollout status deployment/my-deploy

kubectl rollout history deployment/my-deploy

kubectl rollout undo deployment/my-deploy

kubectl rollout undo deployment/my-deploy --to-revision=2
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

### Ingress


### Volumes
- HostPath
```yaml
spec:
  containers:
  -  image: alpine
     volumeMounts:
     - mountPath: /opt
     - name: data-volume
  volumes:
  - name: data-volume
    hostPath:
      path: /data
      type: Directory
```

### Persistent Volume
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  storageClassName: manual
  hostPath:
    path: /tmp/data
  
  # Or Cloud:
  awsElasticBlockStore:
    volumeID: idhere
    fsType: ext4
```

### Persistent Volume Claims
- Administrators usually create Persistent Volumes.
- Users create Persistent Volume Claims.
- Every Persistent Volume Claim is bound to only one Persistent Volume.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
spec:
  storageClassName: manual 
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500mi
```

- Using PVC in Pod:
```yaml
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
    - name: time-volume
      emptyDir: {}
```


### Exec into pod
```bash
kubectl exec podname -- command
```

### Security
- All user access is managed by API Server.
- Basic auth can be used to have .csv password file with: password,user,userID,(optional)group columns.
- Then add `--basic-auth-file=file.csv` into exec params of kube API Server.
- Or `--token-auth-file` is there are tokens instead of passwords in file.

### Role
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer

rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]

- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["list", "get", "create"]
```

### Role Binding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ex-rb

subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-user-role
  apiGroup: rbac.authorization.k8s.io
```

### Role & RoleBinding IMPERATIVE
```bash
kubectl create role developer --verb=get,create,... --resource=pods,deployments

kubectl create rolebinding dev-user-binding --role=developer --user=dev-user  
```

### can-i
- To check authorization to perform certain action
```bash
kubectl auth can-i create deployments
kubectl auth can-i delete nodes

----------------

kubectl auth can-i create deployments --as dev-user [--namespace name]
```

### NetworkPolicy Example
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific-traffic
  namespace: default  # Namespace where policy applies
spec:
  podSelector:
    matchLabels:
      role: backend   # Select Pods with label role=backend
  policyTypes:
    - Ingress
    - Egress  # Policy applies to both Ingress and Egress traffic

  # INGRESS RULES
  ingress:
    # 1. Allow traffic from Pods with label app=frontend in ANY namespace
    - from:
        - namespaceSelector:
            matchLabels:
              env: production
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 80    # Allow HTTP traffic
        - protocol: TCP
          port: 443   # Allow HTTPS traffic

    # 2. Allow traffic from specific IP block (e.g., your internal network)
    - from:
        - ipBlock:
            cidr: 10.0.0.0/24   # Allow this subnet
            except:
              - 10.0.0.128/25  # Block part of subnet
      ports:
        - protocol: TCP
          port: 3306   # Allow MySQL DB traffic

  # EGRESS RULES
  egress:
    # 1. Allow Pods to connect to DNS service in kube-system namespace
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
        - podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53    # DNS over UDP
        - protocol: TCP
          port: 53    # DNS over TCP

    # 2. Allow Pods to connect to an external IP range (e.g., Internet API)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0   # All IPs
      ports:
        - protocol: TCP
          port: 443   # HTTPS only


```

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


```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: dice
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      completions: 1
      backoffLimit: 25
      activeDeadlineSeconds: 20
      template:
        spec:
          containers:
          - name: throw-a-dice-container
            image: kodekloud/throw-dice
            imagePullPolicy: IfNotPresent
          restartPolicy: Never
```

```yaml
piVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-vh-routing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: "watch.ecom-store.com"
    http:
      paths:
      - pathType: Prefix
        path: "/video"
        backend:
          service:
            name: video-service
            port:
              number: 8080
  - host: "apparels.ecom-store.com"
    http:
      paths:
      - pathType: Prefix
        path: "/wear"
        backend:
          service:
            name: apparels-service
            port:
              number: 8080
```
