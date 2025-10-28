# CKA

## How to refer to these notes?
- These are very precise notes for CKA exam with all the essential concepts and commands that you need to know in order to pass the exam.
- Practice or at least remember Vim commands to save your time during the exams. Deleting chunks of code, indenting, and other modification operations tend to consume a lot of time if not performed smartly.
- Almost 70% of the solutions can be found in the documentation, so, I have added doc links so taht you can be familiar and extract code from docs instead of memorizing it. Just make sure to be smart enough to solve scenario based questions using Docs.
- Take out a day or two before the exam and solve the scenario based questions given at the end.
- Memorize only the keys and values that are very difficult to find on the respective documentation page. All other snippets and code can be extracted from docs, reducing preparation load.
- TIP: To be quick in the exam, try doing control+f to find the desired snippet, like I want to directly fine the snippet of persistent volume, I can search 'kind: persistentvolume' to directly scroll to the snippet. Since the UI is slow, this trick works extremely well.

## Some Important Linux Commands
- To check OS info: `cat /etc/*release*`.

## VIM Shortcuts

### Useful VIM Commands
- Command `:startlinenumber,endline>` like: `:35,53>` for one indent forward.
- Command `:startlinenumber,endline>` like: `:35,53>>` for double indent forward.
- Command `:startlinenumber,endline>` like: `:35,53<` for one level indent removal.
- Command `:startlinenumber,endline>` like: `:35,53<<` for two level indent removal.
- Command `:startlinenumber,endline` like: `35,53d` to delete the chunk.
- Tip: to delete a complete line, exit the insert mode, and press `dd`, it will remove the selected line. 

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
- If anything related to scheduler profiling appears in the exam, [this page](https://kubernetes.io/docs/reference/scheduling/config/), is all you need to go through.
- Refer to the above given snippet and remember the only key that you might not find the documentation.

### Labels and Selectors
- `--show-labels` can be used to list down resources with their labels diplayed.
- `-l key=value` when added to get command, retrieves resources with key:value label applied.
```bash
# lists resources with their labels
kubectl get deployments --show-labels

# lists pods with specific labels applied
kubectl get pods -l env=prod,bu=finance,tier=frontend
```
### Taints and Tolerations
- Below are some quick ways to apply or remove taints to/from nodes.
```bash 
# Listing taints applied to a node.
kubectl describe node node01 | grep -i 'taints'

# Adding a taint.
kubectl taint node node01 spray=mortein:NoSchedule`

# Removing a taint:
kubectl taint node node01 spray=mortein:NoSchedule-
```
- For making pods tolerate these taints, you can refer [this page](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) during exam if needed.

### Node Affinity
- To label a node:
```bash 
kubectl label node node01 color=blue
```
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
- If there is any question regarding scheduling a pod or deployment to a specific node, you might need to refer node affinity, on [this page](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/).

### Priority Classes
- Can't be changed at runtime.
- Help us define priorities for different workloads and resources.
- Priority class is created and referenced in `spec.priorityClassName` field.
- `globalDefault: true` is the priority class attribute used in only of the PriorityClasses across the cluster. This enables global default value usage by pods if they do not explicitly define priority value.
```bash
k create priorityclass high-priority --value=100000 --global-default=false --preemption-policy=PreemptLowerPriority
```
- PriorityClass YAML Code:
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Any description here."
```
- This code is all you need for the exam, you can get this snippet during the exam from [pod priority preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/) page.

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
- Jsonpath support in Kubectl allows you to retrieve any data field from any resource, this is usually used to obtain custom output from kubectl get commands, important for exams.
- Refering to [this page](https://kubernetes.io/docs/reference/kubectl/jsonpath/) and getting little familiar with syntax is all you need.

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
- ⚠️ IMPORTANT: You do not need to remember any of these commands, you can find all here on [this page](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/).
- If you do not see the version required, visit [this page](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/#verifying-if-the-kubernetes-package-repositories-are-used) to update package repoistory.
- These both pages are accessible during the exam, make sure to perform an upgrade at least a once using these docs before appearing in the exam.


### Backups
- Resource Configuration
  - `kubectl get all -A -o yaml > all.yml`

- Etcd Cluster
  - To save snapshot: `etcdctl snapshot save snapshot.db`
  - To get info about snapshot: `etcdctl snapshot status snapshot.db`
  - To restore data from snapshot:
    - `systemctl stop kube-apiserver` or stop API-Server Pod
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

- Upgrading a Release
```bash
helm list
helm repo list
helm repo update
helm search nginx (chartname)
helm search repo nginx --versions
helm search repo nginx --versions | grep 18.1.5
helm upgrade myrelease kk-mock1/nginx --version 18.1.15 -n mynamespace
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


### Troubleshooting
- Control Plane Failure:
  - If master node components are deployed as services in Linux:
    - `journalctl -u kube-apiserver`
- Worker Node Failure:
  - `journalctl -u kubelet`
  - `vi /var/lib/kubelet/config.yaml`
  - `vi /etc/kubernetes/kubelet.conf`


### Networking in Kubernetes
- port: 8080:80 means port 80 of container can be accessed by port 8080 on host.
- Check for open ports on nodes (inbound):
  - 6443 on master
  - 10250 for Kubelet
  - 10259 for Kube Scheduler
  - 10257 for Kube Controller Manager
  - 2379, 2380 for ETCD
  - 30,000-32,767 for NodePort Services on Worker Nodes

- Check for default gateway: `ip route show default`
- To check what applications are listening on what port: `netstat`
- To check for connections on port: `netstat -anp | grep etcd`

- CNI Directory: `ls /opt/cni/bin`
- CNI Config Directory: `ls /etc/cni/net.d` - used to check what CNI is being used in cluster.

- Checking pod to pod connection: `kubectl exec -it SOURCE-POD-NAME -- curl -m 5 DESTINATION-POD-IP`

- Cluster CIDR can be found in Kube-Controller-Manager's manifest.
- Cluster IP range for services can be found in Kube-API Server's manifest.

- CoreDNS Config File Location: mentioned in args of coredns deployments.

### Ingress
- Create ingress in imperative way:
```bash
kubectl create ingress ingress-test --rule="wear.my-online-store.com/wear*=wear-service:80"**
```

### Gateway API

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes: 
        namespaces: 
          from: All
```

- Creating a rule with parent ref of a Gateway in another namespace:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-route
  namespace: default
spec:
  parentRefs:
    - name: nginx-gateway           # Name of the Gateway
      namespace: nginx-gateway      # Namespace where the Gateway is deployed
      sectionName: http             # Attach to the 'http' listener
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: frontend-svc
          port: 80
```

## IMPORTANT ⚠️

### Scenario Based Questions and their Solutions: 
## Listing Deployments in Alphabetical Order

```bash
kubectl -n admin2406 get deployment -o custom-columns=DEPLOYMENT:.metadata.name,CONTAINER_IMAGE:.spec.template.spec.containers[].image,READY_REPLICAS:.status.readyReplicas,NAMESPACE:.metadata.namespace --sort-by=.metadata.name > /opt/admin2406_data
```

## Updating a deployment using Rolling Update
```bash
kubectl set image deploy nginx-deploy nginx=nginx:1.17

kubectl annotate deployment nginx-deploy description="Updated nginx image to 1.17"
  
```

## Installing a .deb package (you might be required to install and enable a servic on Node):
```bash
dpkg -i ./filename.deb
systemctl start service-name
systemctl enable service-name
```

## Exposing a Pod Imperatively (Creating Service):
```bash
k expose pod podname --port=123 --name=servicename
```

## Upgrading a Helm Release
```bash
helm list
helm repo list
helm repo update
helm search chartnamehere
helm search repo nginx --versions
helm search repo nginx --versions | grep 18.1.5
helm upgrade myrelease kk-mock1/nginx --version 18.1.15 -n mynamespace
helm lint ./directorynamehavingchartinit
```

- Troubleshoot Ingress
```bash
k get ingressclass
```
- And then set ingressClassName to the one.


## Gateway with HTTPS and TLS Secret Ref:
```yaml
# web-gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: cka5673
spec:
  gatewayClassName: kodekloud
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: kodekloud.com
      tls:
        certificateRefs:
          - name: kodekloud-tls
```
- IMPORTANT: if there is any task for updating or creating gatway to use HTTPS protoocl, use the snippet given above, protocol, port will change, also, hostname can be added if mentioned in the question.
- You can use tls feild to use tls secret from any already running ingress.

- Getting Node CIDR:
```bash
kubectl get node controlplane -o jsonpath='{.spec.podCIDR}' > /root/pod-cidr.txt
```

## Network policy to allow all ingress on a pod:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-to-nptest
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: np-test-1
  policyTypes:
  - Ingress
  ingress:
  - ports:
    - protocol: TCP
      port: 80
```

## HTTP Route with Weights:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: default
spec:
  parentRefs:
    - name: web-gateway
      namespace: default
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: web-service
          port: 80
          weight: 80
        - name: web-service-v2
          port: 80
          weight: 20
```

## Restore ETCD Backup
```bash
ETCDCTL_API=3 etcdctl --data-dir="/var/lib/etcd-backup" \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot restore etcd-backup.db
```

## Prepare Linux System for Kubeadm:
```bash
sudo dpkg -i /dir/cri-dockerd.deb
sudo systemctl enable --now cri-dockerd.service

vi /etc/sysctl.d/cka.conf (and paste provided values in file)
sudo sysctl --system
```

## Install ArgoCD using Helm:
1. helm repo add argo https://argoproj.github.com/argo-helm
2. k create ns argocd
3. helm template argocd argo/argo-cd --namespace=argocd --version 7.7.3 --set crds.install=false > /argo-helm.yaml
4. helm install argocd argo/argo-cd --namespace=argocd --version 7.7.3 --set crds.install=false
5. k get pods - argocd

### Install CNI
```bash
k get pods -A | grep -E 'calico|canal|flannel|weave|cni' (and delete pods/ds if there)
sudo rm -rf /etc/cni/net.d/*

Then install (download custom resources using wget and set spec.calicoNetwork.bgp: Disabled, and change pod CIDR usually /24)
```

### Sidecar Task
  - Visit sidecar Kubernetes docs.
  - Use emptyDir as volume type.

### Gateway API Task
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
spec:
  gatewayClassName: nginx-class
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: "gateway.web.k8s.local"
    tls:
      mode: Terminate
      certificateRefs:
        - kind: secret
          name: web-tls

---

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: web-gateway
  hostnames:
  - "gateway.web.k8s.local"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-service
      port: 80
```

### Pod Resource Allocation Task
**For memory**:
  - Describe node and extract allocatable memory in Ki.
  - Convert memory in to Mi by dividing the Ki value 1024.
  - Calculate already used memory and subtract it from the allocatatble memory.
  - Finally, subtract 10% overhead memory value from allocatable memory, divide the final value by number of deployment pods to find how much memory can be allocated to each pod.

**For CPU (milis - unit of 1000):**
  - Describe node and find allocatable CPU.
  - Subtract the sum of used CPU on the node from the total allocatable value.
  - Calculate 10% overhead and subtract from the result fo step 2.
  - Now divide it by the number of proposed pods for the deployment.

**Optional (If Asked to Add Limits):**
  - Keep limit a bit higher than request value.

### For Storage Class
 - Use minimal template from docs with default annotation as it is.
 - To reconfigure it as a default storageclass, use:
   - `k patch sc local-sc -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class": "true"}}}'
  
### Patch a Deployment with PriorityClass
- Use comamnd:
```bash
    k patch deploy busybox-logger -p '{"spec":{"template":{"spec": {"priorityClassName":"high-priority"}}}}'
```

### Create HPA for a deployment with cooldown window:
- Get the template from horizontal pod autoscaling documentation page.
- Add this snippet under spec (this was tricky to find):
```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
```

### NodePort Service that Exposes the Deployment using TCP Protocol, and also its pods are being exposed.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
namespace: namespacename
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      nodePort: 30007
```

## Fix Broken Cluster that was Using Old ETCD Server:
- journalctl -u kubelet -f
- cd /etc/kubernetes/manifests
- vi kube-apiserver.yaml
- update ETCD IP or port number, set port to 2379 and adress to 127.0.0.1 (etcd --listen-clients-urls)
- sudo systemctl restart kubelet
- check other manifest files if issues persist


## Create PriorityClass and Patch the Deployment to Use it:
- k get pc
- get value from user critical pc and use 'expr 1235374 - 1' or that particular value to get exact -1 than that.
- create new pc with that priority (priority class code can be found in documentation)
- kubectl patch deploy busybox-logger -n priority -p {"spec": {"template":{"spec":{"priorityClassName":"high-priority"}}}}'
- k rollout restart deploy busybox-logger -n priority