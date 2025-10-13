# CKA

## Some Important Linux Commands
- To check OS info: `cat /etc/*release*`.

## VIM Shortcuts

### Useful Range Commands
- Command `:startline,endline>` like: `:35,53>` for one indent.
- Command `:startline,endlined` like: `35,53d>` to delete the chunk.

## Scheduling

### Scheduler Profiles
``` yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
    plugins:
      score:
        disabled:
          - name: PodTopologySpread
        enabled:
          - name: NodeResourcesFit
    pluginConfig:
      - name: NodeResourcesFit
        args:
          scoringStrategy:
            type: MostAllocated
            resources:
              - name: cpu
                weight: 1
              - name: memory
                weight: 1
leaderElection:
  leaderElect: true
  leaseDuration: 15s
  renewDeadline: 10s
  retryPeriod: 2s
clientConnection:
  kubeconfig: "/etc/kubernetes/scheduler.conf"

```

### Labels and Selectors
- `--show-labels` can be used to list down resources with their labels diplayed.
- `-l key=value` when added to get command, retrieves resources with key:value label applied.
  - ```bash
       k get pods -l env=prod,bu=finance,tier=frontend
       ```
### Taints and Tolerations
- `k describe node node01 | grep -i 'taints'`
- `k taint node node01 spray=mortein:NoSchedule`
- Removing a taint: `k taint node node01 spray=mortein:NoSchedule-`.

### Node Affinity
- To label a node `k label node node01 color=blue`.
- To add NodeAffinity to a Deployment:
- Add following snippet under `spec.template.spec`
``` yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: color 
          operator: In
          values:
          - blue
```

### Priority Classes
- Can't be changed at runtime.
- Help us define priorities for different workloads and resources.
- Priority class is created and referenced in `spec.priorityClassName` field.
- `globalDefault: true` is the priority class attribute used in only of the PriorityClasses across the cluster. This enables global default value usage by pods if they do not explicitly define priority value.
```bash
k create priorityclass high-priority --value=100000 --global-default=false --preemption-policy=PreemptLowerPriority
```
TODO: LABS TO BE CONTINUED

## Custom Scheduler for Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-custom-scheduler
spec:
  schedulerName: my-scheduler   # 👈 custom scheduler name
  containers:
  - name: nginx
    image: nginx:1.27
```

## JSON Path in Kubernetes


## Cluster Maintenance
- Operating System Upgrade
  - Pod eviction time out is set on kube-controller-manager in params --pod-eviction-timeout=5m0s manner.
  - `kubectl drain node-1` moves workloads to another node safely and cordon node if node-1 has to be taken down for some reason.
  - Nodes must be uncordoned after they're brought online to resume scheduling on them - `kubectl uncordon node-1`.

### Cluster Upgrade using Kubeadm
- Update Kubeadm first, only to the next minor version at a time.
- `apt-get upgrade -y kubeadm=1.12.0-00` - this upgrades kubeadm itself.
- `kubeadm upgrade apply v1.12.0` - upgrades cluster components then.
- `kubectl get nodes` shows `kubelet` version on each node.
- To then update kubelet on nodes:
  - `kubectl drain node01`
  - `apt-get upgrade -y kubeadm=1.12.0-00`
  - `apt-get upgrade -y kubelet=1.12.0-00`
  - `kubeadm upgrade node config --kubelet-version v1.12.0`
  - `systemctl restart kubelet`
  - `kubectl uncordon node01`
- For worker nodes, only kubelet update is needed.


### Designing a Kubernetes Cluster
- 

### Backups
- Resource Configuration
  - `kubectl get all -A -o yaml > all.yml`

- Etcd Cluster
  - To save snapshot: `etcdctl snapshot save snapshot.db`
  - To get info about snapshot: `etcdctl snapshot status snapshot.db`
  - To restore data from snapshot:
    - `systemctl stop kube-apiserver`
    - `etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd-from-backup`
    - Update `--data-dir /var/lib/etcd-from-backup` in etcd service (can be done in manifest)
    - `systemctl daemon-reload`
    - `systemctl restart etcd`
    - `systemctl start kube-apiserver`
- Full command with certificates passed:
> ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key snapshot save /backup/etcd-snapshot.db
- For offline file-level backup of the data directory:
> etcdutl backup \ --data-dir /var/lib/etcd \ --backup-dir /backup/etcd-backup
- To use a backup made with etcdutl backup, simply copy the backup contents back into /var/lib/etcd and restart etcd.

- Persistent Volumes

### Admissions Controlers
- Mutating Admission Controllers run Before Validating Admission Controllers so that mutations can be validated afterwards.
- Mutating Admission Controllers
- Validating Admission Controllers

- To have own Admission Controllers, we use:
  - MutatingAdmissionWebhook
  - ValidatingAdmissionWebhook
- 'Admission Review' kind object is sent to these webhooks and they then return 'allowed: true' or false flag with in the response JSON.

### Monitoring Cluster Components
- Metrics Server is used.
- It uses cAdvisor on each node to collect metrics.

### Autoscaling
- **Horizontal Pod Autoscaling (HPA):**
  - `kubectl autoscale deployment my-app --cpu-percent=50 --min=1 --max=10`
  - `kubectl delete hpa my-app`
- **Vertical Pod Autoscaling (VPA):**
```yaml
apiVersion: "autoscaling.k8s.io/v1"
kind: VerticalPodAutoscaler
metadata:
  name: flask-app
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: flask-app-4
  updatePolicy:
    updateMode: "Off"  # You can set this to "Auto" if you want automatic updates
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 100m
        maxAllowed:
          cpu: 1000m
        controlledResources: ["cpu"]
```

- In-place Pod Resizing:
  - Normally we have to kill the pod in order to increase resources requests and limits (till kubernetes 1.33).
    

## Security in Kubernetes

### Security Primitives
- Hosts
  - Password based authentication disabled.
  - SSH key based authentication disabled.

### Controlling Access to the API Server
- API Server is the first line of defense.
- Authorization:
  - RBAC Authorization
  - ABAC Authorization
  - Node Authorization
  - Webhook Mode
  - Always Allow
  - Always Deny
- **Mechanisms**
  - **Static Token File: (not recommended)**
    - contains: `token,username,uid,group` in CSV file.
    - passed as: `--basic-auth-file=user-details.csv` to the api-server container args.
    - used as: `curl -v -k https://masternodeip:6443/api/v1/pods --header "Authorization: Bearer tokenvaluehere"`
  - **Static Password File:**
    - contains: `plaintextpassword,username,group` in CSV file.
    - passed as: `--token-auth-file=user-token-details.csv` to the api-server container args.
    - used as: `curl -v -k https://masternodeip:6443/api/v1/pods -u "user1:password"`
  - **Certificates**
  - **Identity Services**

### Certificates
- Root Certificates - one 
- Client Certificate - Required to prove other servers the authenticity when they use it as client.
- Server Certificate - Required for it to communicate and authenticate itself to its clients.

- **Generating CA Certificate:**
  - Generate keys: `openssl genrsa -out ca.key 2048`
  - Certificate signing request: `openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr`
  - Sign certificates: `openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt`
- **Generating Other Certificates using CA Certificate:**
  - Generate key: `openssl genrsa -out user.key 2048`
  - Certificate signing request: `openssl req -new -key user.key -subj "/CN=KUBERNETES-ADMIN" -out user.csr`
  - Sign certificates: `openssl x509 -req -in user.csr -CA ca.crt -CAkey ca.key -out user.crt` - remember, we will use CA certificate and key for this certiciate to get our CSR signed.
- **Adding Group Info to the Certificate:**
  - In CSR: `openssl req -new -key user.key -subj "/CN=KUBERNETES-ADMIN/OU=system:masters" -out user.csr`
- **Names for Kubernetes Component Certifiates Should have Prefix 'SYSTEM':**
  - `openssl req -new -key kube-scheduler.key -subj "/CN=SYSTEM:KUBE_SCHEDULER" -out kube-scheduler.csr`
- **Generating Kube API Server Certificates:**
  - Kube API Server needs multiple names to be identified by differnet cluster components.
  - It should include names: kubernetes, kubernetes.default, kubernetes.default.svc, kubernetes.default.svc.cluster.local.

- **Check info of a Certificate:**
  - `openssl x509 -in file-path.crt -text -noout`

- **Generate Encoded Version of CSR:**
  - `cat akshay.csr | base64 -w`

- **Generate Kubernetes CSR from the OUTPUT of Previous Command:**
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: akshay
spec:
  groups:
  - system:authenticated
  request: <Paste the base64 encoded value of the CSR file>
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```

- **Approve the Request:**
  - `kubectl certitificate approve akshay`

### Kube Config
- To use any kube config other than the default one: `kubectl config --kubeconfig=/root/my-kube-config use-context research`
- To set it default, use `KUBECONFIG` environent variable: `export KUBECONFIG=/root/my-kube-config` to persist: add that mentioned line in `vi ~/.bashrc`.

### API Groups
- Checking Kubernetes Version: `curl https://kube-master-IP:6443/version`
- To Check Pods: `curl https://kube-master-IP:6443/api/v1/pods`

- **Core Group (/api):**
  - Core functionality

- **Named Group (/apis):**

### Creating Docker Secret:
- `kubectl create secret docker-resitry my-secret --docker-server=DOCKER_REGISTRY_SERVER --docker-username=rashid --docker-password=passwd --docker-email=rashid@gmail.com`

## Network Policies
- Flannel doesn't support network policies
- Ingress rule implies that requests will be responded properly, so we do not need egress rule unless our pod explicitly makes external calls to the other pod set in ingress.

### Custom Resource Definitions
-

### Helm Basics
- Installing Helm
- Helm requires Windows, Linux, or Mac OS host.
- Charts are hosted at `artifacthub.io`.
- **In Linux:**
```bash 
sudo snap install helm --classic
helm -h
helm get -h
helm search wordpress -h
helm search hub wordpress
```
> helm chart apiVersion v1 for Helm 2, v2 for Helm 3.

- Basic Commands
```bash
helm install wordpress
helm upgrade wordpress
helm rollback wordpress
helm uninstall wordpress
helm history my-release
helm rollback my-release 1 (1 is revision number to rollback to)
```
- Adding or Updating Helm Repo
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

- Installing a Release
```bash
helm install my-release bitnami/wordpress
helm install my-release bitnami/wordpress --version 7.1.0
```

- Listing Releases
```bash
helm list
```

- Uninstalling a Release
```bash
helm uninstall my-release
```

- Imperatively Setting Values
``` bash
helm install --set valueInValuesYAML="Hello!" my-release bitnami/wordpress
```

- Loading Values from a Custom Values file:
- This will override values in default values.yaml file.
```bash
helm install --values custom-values.yaml my-release bitnami/wordpress
```

- Getting Chart Files to Manually Update Values.yaml
```bash
helm pull bitnami/wordpress
helm pull --untar bitnami/wordpress
helm install my-release ./wordpress
```

### Installing Kubernetes

### 

### Troubleshooting
- Control Plane Failure:
  - If master node components are deployed as services in Linux:
    - `journalctl -u kube-apiserver`
- Worker Node Failure:
  - `journalctl -u kubelet`
  - `vi /var/lib/kubelet/config.yaml`
  - `vi /etc/kubernetes/kubelet.conf`


### Networking in Kubernetes


### Scenario Based Questions and their Solutions: 
