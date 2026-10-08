# CKS

## How to refer to these notes?
- These are high-yield notes for the CKS exam, written for the **Kubernetes v1.35** exam environment and the current curriculum (checked Oct 2026). Same approach as my CKA notes: understand the concept, know the few commands, and know *where in the docs* to grab the YAML from.
- Exam format: **15–20 tasks, 2 hours, 67% to pass**, active CKA required. CKS is much more time-pressured than CKA. If a task takes more than ~7–8 minutes, flag it and move on.
- Every task tells you which host to work on. `ssh <host>`, do the task, then `exit` back to the `base` node before the next one (nested ssh is not supported). Use `sudo -i` for root.
- On task hosts, `kubectl` with the `k` alias + bash completion, `yq`, `curl`, `wget` and man pages are pre-installed. The `base` node has none of these.
- **Allowed docs:** kubernetes.io/docs, kubernetes.io/blog, falco.org/docs, bom CLI reference, etcd.io/docs, ingress-nginx docs, docs.cilium.io, istio.io/latest/docs, plus whatever links appear in each task's **Quick Reference** box. Trivy, kube-bench, kubesec and kube-linter are NOT on the list, so memorize the handful of flags below (or use `--help`).
- Almost every YAML in this exam is on a docs page. I've added the page name and what to Ctrl+F for, so you can jump straight to the snippet instead of memorizing it.
- You get 2 killer.sh sessions with the exam. They're harder than the real exam. Do one a week before and one two days before.

## Exam Domains

| Domain | Weight |
|---|---|
| Cluster Setup | 15% |
| Cluster Hardening | 15% |
| System Hardening | 10% |
| Minimize Microservice Vulnerabilities | 20% |
| Supply Chain Security | 20% |
| Monitoring, Logging and Runtime Security | 20% |

## Docs Search Cheat Sheet (Ctrl+F is your best friend)

| Need | Docs page | Ctrl+F for |
|---|---|---|
| Audit policy + apiserver flags/volumes | Auditing | `audit-policy-file` |
| Encryption at rest | Encrypting Confidential Data at Rest | `kind: EncryptionConfiguration` |
| ImagePolicyWebhook | Admission Control | `ImagePolicyWebhook` |
| ValidatingAdmissionPolicy | Validating Admission Policy | `ValidatingAdmissionPolicyBinding` |
| AppArmor | Restrict a Container's Access to Resources with AppArmor | `appArmorProfile` |
| Seccomp | Restrict a Container's Syscalls with seccomp | `localhostProfile` |
| Pod Security Admission | Enforce Pod Security Standards with Namespace Labels | `pod-security.kubernetes.io` |
| PSA restricted fields | Pod Security Standards | `Restricted` |
| RuntimeClass | Runtime Class | `handler:` |
| NetworkPolicy | Network Policies | `ipBlock` |
| Ingress TLS | Ingress | `secretName` |
| Security context | Configure a Security Context for a Pod or Container | `readOnlyRootFilesystem` |
| SA token automount | Configure Service Accounts for Pods | `automountServiceAccountToken` |
| Anonymous auth config | Authenticating | `AuthenticationConfiguration` |
| Kubelet config fields | Kubelet Configuration (v1beta1) | `readOnlyPort` |
| Verify k8s binaries | Verify Signed Kubernetes Artifacts | `cosign verify-blob` |
| Falco fields | falco.org → Supported Fields for Conditions and Outputs | `container.id` |
| Falco overrides | falco.org → Overriding Rules | `override:` |
| Cilium encryption | docs.cilium.io → WireGuard Transparent Encryption | `encryption.type` |
| Cilium mutual auth | docs.cilium.io → Mutual Authentication Example | `authentication:` |
| Istio mTLS | istio.io → PeerAuthentication / Mutual TLS Migration | `mode: STRICT` |

## Exam Setup (First Minute)
```bash
# Handy shortcuts (set per host if you'll be doing a lot of YAML there)
export do="--dry-run=client -o yaml"      # k run x --image=nginx $do > pod.yaml
export now="--force --grace-period=0"     # k delete pod x $now

# Vim: proper YAML indentation
echo 'set expandtab tabstop=2 shiftwidth=2' >> ~/.vimrc
```
- `k explain pod.spec.securityContext --recursive` is often faster than searching docs for field names.

## ⚠️ Editing Control Plane Manifests Safely (half the exam touches these)
- Static pod manifests live in `/etc/kubernetes/manifests/`. Kubelet recreates the pod on every save.
- **Back up OUTSIDE the manifests dir.** A backup inside it gets run as a second static pod.
```bash
cp /etc/kubernetes/manifests/kube-apiserver.yaml ~/kube-apiserver.yaml.bak
```
- After editing, wait and watch it come back:
```bash
watch crictl ps            # wait until kube-apiserver is Running again (30–90s)
k get pods -n kube-system  # confirm API answers
```
- If the API server does not come back:
```bash
crictl ps -a | grep kube-apiserver          # exited container?
crictl logs <container-id>                  # usually shows the bad flag/path
ls /var/log/pods/ | grep apiserver          # logs even when container keeps dying
tail -20 /var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log
journalctl -u kubelet -f | grep -i apiserver   # YAML syntax errors show here
```
- To force a restart (e.g. after changing a policy file the flag points to):
```bash
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/ && sleep 20 && mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```
- **#1 cause of a broken API server in CKS:** you added a flag pointing to a file but didn't mount that file into the pod. Every new file/dir needs a `volumeMount` + `hostPath` volume:
```yaml
    volumeMounts:
    - name: my-config
      mountPath: /etc/kubernetes/my-config
      readOnly: true
  volumes:
  - name: my-config
    hostPath:
      path: /etc/kubernetes/my-config
      type: DirectoryOrCreate
```

---

## 1. Cluster Setup (15%)

### Security Benchmark: Center for Internet Security (CIS) with kube-bench
- CIS Kubernetes Benchmark = list of recommended settings for control plane, etcd, kubelet, and policies. `kube-bench` runs these checks.
- Output has `[PASS]`, `[FAIL]`, `[WARN]`, `[INFO]` and a **Remediations** section that tells you exactly what to change. Read the remediation, don't guess.
```bash
kube-bench run --targets=master                 # control plane (alias: controlplane)
kube-bench run --targets=node                   # kubelet/worker checks
kube-bench run --targets=etcd
kube-bench run --targets=master --check=1.2.16  # re-check a single item after fixing
kube-bench run --targets=master | grep -A3 FAIL
```
- **Common fixes** (manifests in `/etc/kubernetes/manifests/`, kubelet in `/var/lib/kubelet/config.yaml`):

| Component | Setting |
|---|---|
| kube-apiserver | `--profiling=false`, `--authorization-mode=Node,RBAC`, `--enable-admission-plugins=NodeRestriction,...`, audit flags set, `--anonymous-auth=false` (only if asked, see API access section) |
| kube-controller-manager | `--profiling=false`, `--terminated-pod-gc-threshold=10`, `--use-service-account-credentials=true`, `--bind-address=127.0.0.1` |
| kube-scheduler | `--profiling=false`, `--bind-address=127.0.0.1` |
| etcd | `--client-cert-auth=true`, `--peer-client-cert-auth=true`, `--auto-tls=false` |
| File permissions | `chmod 600 /etc/kubernetes/manifests/*.yaml`, `chown root:root ...`, `chmod 600 /etc/kubernetes/pki/*.key` |
| etcd data dir | `chmod 700 /var/lib/etcd`, `chown -R etcd:etcd /var/lib/etcd` (create user if missing: `useradd -r -s /sbin/nologin etcd`) |

- **Kubelet hardening** (`/var/lib/kubelet/config.yaml` → then `systemctl daemon-reload && systemctl restart kubelet`):
```yaml
authentication:
  anonymous:
    enabled: false          # no unauthenticated access to :10250
  webhook:
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook             # never AlwaysAllow
readOnlyPort: 0             # disables unauthenticated 10255
protectKernelDefaults: true
rotateCertificates: true
seccompDefault: true        # RuntimeDefault seccomp for every pod on this node
```
- Find which config file kubelet actually uses: `ps aux | grep kubelet | grep -o -- '--config=[^ ]*'`. Flags in the systemd drop-in (`/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf` or `/etc/systemd/system/kubelet.service.d/`) override the config file.

### Network Policies
- No policy selecting a pod = everything allowed. Once ANY policy selects a pod for a direction (ingress/egress), only explicitly allowed traffic passes in that direction.
- Policies are additive (union). There are no "deny" rules in vanilla NetworkPolicy, only "allow" rules on top of an implicit deny.
- ⚠️ **AND vs OR (most tested detail):**
```yaml
  ingress:
  - from:
    - namespaceSelector:          # ONE list item with both selectors = AND
        matchLabels:              # (pods with tier=web IN namespaces with team=a)
          team: a
      podSelector:
        matchLabels:
          tier: web
  - from:
    - namespaceSelector: {...}    # separate list items = OR
    - podSelector: {...}
```
- Every namespace automatically gets the label `kubernetes.io/metadata.name: <ns-name>`. Use it in `namespaceSelector`.
- **Default deny all (ingress + egress) in a namespace:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```
- **Allow DNS egress** (needed after egress default deny, otherwise service names stop resolving):
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```
- Test connectivity:
```bash
k -n prod exec deploy/frontend -- curl -m 3 backend:8080
k -n prod exec deploy/frontend -- nc -zv -w 3 10.0.0.5 5432
k run tmp --rm -it --image=busybox -n prod -- wget -qO- -T 3 backend:8080
```

### Ingress with TLS
- Create the cert (if not given) and TLS secret:
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=secure.example.com"
k create secret tls secure-tls --cert=tls.crt --key=tls.key -n app
```
- Ingress (from the Ingress docs page, search `secretName`):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: app
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"        # HTTP -> HTTPS (default true when tls is set)
    # nginx.ingress.kubernetes.io/force-ssl-redirect: "true" # redirect even without tls section
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.example.com
    secretName: secure-tls
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```
- Test (get the controller's NodePort with `k get svc -n ingress-nginx`):
```bash
curl -kv https://secure.example.com:31443 --resolve secure.example.com:31443:<node-ip>
# look for the CN in "server certificate" section, and a 308 redirect on http
```
- If the ingress does nothing: `k get ingressclass` and set the right `ingressClassName`.

### Protect Node Metadata and Endpoints
- Cloud metadata endpoint `169.254.169.254` can leak node credentials. Block it with an egress NetworkPolicy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-metadata
  namespace: app
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32
```
- If only some pods may reach it, add a second policy that allows `169.254.169.254/32` for pods with a specific label (policies are additive).
- Kubelet endpoints: `10250` must require auth (see kubelet config above), `10255` read-only port must be `0`.
```bash
curl -sk https://<node-ip>:10250/pods    # should be 401 Unauthorized, NOT a pod list
```

### Verify Platform Binaries Before Deploying
- Compare the hash of a binary with the one published on the Kubernetes release page / given in the task:
```bash
sha512sum /usr/bin/kubelet /usr/bin/kubectl /usr/bin/kubeadm
echo "<expected-hash>  kubelet" | sha512sum -c      # two spaces between hash and file
kubelet --version; kubectl version --client

# Binary running inside a static pod container (e.g. kube-apiserver):
ps aux | grep [k]ube-apiserver           # get PID
sha512sum /proc/<PID>/exe
```
- Delete/replace binaries whose hash doesn't match (if the task says so).
- Release artifacts are signed with cosign. The docs page "Verify Signed Kubernetes Artifacts" has the exact `cosign verify-blob` command with the certificate identity and issuer to use.

---

## 2. Cluster Hardening (15%)

### Role Based Access Control (RBAC)
- Role/RoleBinding = namespaced. ClusterRole/ClusterRoleBinding = cluster-wide. ClusterRole + RoleBinding = ClusterRole's permissions limited to that namespace.
- RBAC is additive only (no deny). To remove a permission, edit the Role or remove the binding.
```bash
k create role pod-reader -n dev --verb=get,list,watch --resource=pods,pods/log
k create rolebinding pod-reader-rb -n dev --role=pod-reader --serviceaccount=dev:app-sa
k create clusterrole node-reader --verb=get,list --resource=nodes
k create clusterrolebinding node-reader-crb --clusterrole=node-reader --user=jane

# Check effective permissions
k auth can-i list secrets -n dev --as=system:serviceaccount:dev:app-sa
k auth can-i --list -n dev --as=jane

# Who is bound to what
k get rolebinding,clusterrolebinding -A -o wide | grep app-sa
```
- Dangerous permissions to look for and remove: `*` verbs/resources, `secrets` get/list (list returns contents too), `pods/exec`, `nodes/proxy`, `serviceaccounts/token` create, verbs `escalate`, `bind`, `impersonate`, and bindings to `cluster-admin`.

### Service Accounts
- Every namespace has a `default` SA. Pods get a token auto-mounted at `/var/run/secrets/kubernetes.io/serviceaccount/` unless you disable it.
- Disable automount on the SA (applies to all pods using it) or on the pod (pod setting wins):
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: app
automountServiceAccountToken: false
---
# in pod / deployment template spec:
spec:
  serviceAccountName: backend-sa
  automountServiceAccountToken: false
```
```bash
k patch sa default -n app -p '{"automountServiceAccountToken": false}'
k set serviceaccount deploy/backend backend-sa -n app
k exec -n app <pod> -- ls /var/run/secrets/kubernetes.io/serviceaccount   # should fail
```
- Short-lived tokens: `k create token backend-sa -n app --duration=10m`. Avoid legacy long-lived `kubernetes.io/service-account-token` Secrets.
- Projected token with custom audience/expiry (when an app needs a token but you disabled automount):
```yaml
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
          audience: vault
```

### Restrict Access to the Kubernetes API
- Request flow: Authentication → Authorization → Admission → etcd.
- Key kube-apiserver flags:
  - `--authorization-mode=Node,RBAC` (never `AlwaysAllow`).
  - `--enable-admission-plugins=NodeRestriction` (kubelet can only modify its own Node object and pods bound to it, and can't set `node-restriction.kubernetes.io/` labels).
  - No `--kubernetes-service-node-port=...` (exposes the API via a NodePort on the `kubernetes` service).
  - `--anonymous-auth=false` disables anonymous requests. Stable since v1.34, you can instead use an `AuthenticationConfiguration` that allows anonymous only on health endpoints (can't be combined with `--anonymous-auth`):
```yaml
# /etc/kubernetes/auth/auth-config.yaml  → --authentication-config=/etc/kubernetes/auth/auth-config.yaml (+ volume mount)
apiVersion: apiserver.config.k8s.io/v1
kind: AuthenticationConfiguration
anonymous:
  enabled: true
  conditions:
  - path: /livez
  - path: /readyz
  - path: /healthz
```
- Test NodeRestriction from a worker:
```bash
export KUBECONFIG=/etc/kubernetes/kubelet.conf
k label node node01 node-restriction.kubernetes.io/test=yes   # should be forbidden
k label node controlplane foo=bar                              # other node: forbidden
```
- API exposed via NodePort? Remove `--kubernetes-service-node-port` from the manifest, then `k delete svc kubernetes` (it's recreated as ClusterIP).
- Check: `curl -k https://<ip>:6443/api/v1/namespaces` should give 401/403, never data.

### Upgrade Kubernetes to Avoid Vulnerabilities
- Same procedure as CKA (see CKA notes / "Upgrading kubeadm clusters" page): one minor version at a time, control plane first, then workers (drain → upgrade kubeadm → `kubeadm upgrade node` → upgrade kubelet/kubectl → restart kubelet → uncordon).
- New minor version = new apt repo (`pkgs.k8s.io/core:/stable:/v1.XX/deb/`), edit `/etc/apt/sources.list.d/kubernetes.list` first.

---

## 3. System Hardening (10%)

### Minimize Host OS Footprint
```bash
# Services
systemctl list-units --type=service --state=running
systemctl disable --now vsftpd              # stop + disable
systemctl mask apache2                      # prevent starting at all

# Packages
apt list --installed | grep -i <pkg>
apt purge -y <pkg>

# Open ports → process → binary
ss -tlnup                                   # or: netstat -tlnp
ss -tlnp | grep :6666
lsof -i :6666
ls -l /proc/<PID>/exe                       # actual binary path
kill -9 <PID>; rm -f <binary-path>

# Users and sudo
getent passwd | grep -v nologin
userdel -r bob                              # remove user + home
usermod -s /usr/sbin/nologin bob            # disable login shell
gpasswd -d bob sudo                         # remove from sudo group
cat /etc/sudoers; ls /etc/sudoers.d/        # edit with visudo

# Kernel modules
lsmod | grep -E 'sctp|dccp'
cat >> /etc/modprobe.d/blacklist.conf <<EOF
blacklist sctp
blacklist dccp
EOF
modprobe -r sctp                            # unload now if not in use

# SSH (/etc/ssh/sshd_config): PermitRootLogin no, PasswordAuthentication no
systemctl restart ssh                       # 'sshd' on some distros
```
- Don't reboot hosts unless the task says so, and never reboot `base`.

### Least-Privilege Identity and Access Management
- Linux: no shared accounts, no unnecessary sudo, service users with `nologin` shells, processes not running as root.
- Kubernetes: least-privilege RBAC + SAs (Domain 2).
- Cloud: node IAM roles should have minimal permissions, and pods should not be able to reach the metadata endpoint to steal them (NetworkPolicy above).

### Minimize External Access to the Network (UFW)
```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow from 10.0.0.0/24 to any port 6443 proto tcp
ufw enable
ufw status numbered
ufw delete 3
```

### AppArmor
- Linux kernel module that restricts what a program can do (file, network, capability access) using profiles. Modes: **enforce** and **complain** (log only).
- ⚠️ The profile must be **loaded on the node where the pod runs**. Pin the pod with `nodeName`/`nodeSelector` if needed.
- ⚠️ In the pod you use the **profile name** from the `profile <name> ...` line inside the file, NOT the filename.
```bash
aa-status                                   # or: apparmor_status (lists loaded profiles)
apparmor_parser -q /etc/apparmor.d/k8s-deny-write     # load
apparmor_parser -r /etc/apparmor.d/k8s-deny-write     # reload after edit
aa-enforce /etc/apparmor.d/k8s-deny-write             # set enforce mode
aa-complain /etc/apparmor.d/k8s-deny-write
```
- Example profile (from the k8s AppArmor tutorial):
```
#include <tunables/global>

profile k8s-apparmor-example-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny all file writes.
  deny /** w,
}
```
- Pod (field is GA since v1.30, the old `container.apparmor.security.beta.kubernetes.io/...` annotation is deprecated):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-apparmor
spec:
  nodeName: node01
  securityContext:
    appArmorProfile:              # can also be set per container
      type: Localhost             # RuntimeDefault | Localhost | Unconfined
      localhostProfile: k8s-apparmor-example-deny-write
  containers:
  - name: hello
    image: busybox:1.36
    command: ["sh", "-c", "echo 'Hello AppArmor!' && sleep 1h"]
```
```bash
k exec hello-apparmor -- cat /proc/1/attr/current    # shows profile + (enforce)
k exec hello-apparmor -- touch /tmp/test              # Permission denied
```

### Seccomp
- Filters the **syscalls** a container can make. Profiles for `Localhost` type live under the kubelet seccomp dir: **`/var/lib/kubelet/seccomp/`**, and `localhostProfile` is a path **relative** to it.
```yaml
spec:
  securityContext:
    seccompProfile:
      type: Localhost                      # RuntimeDefault | Localhost | Unconfined
      localhostProfile: profiles/audit.json   # = /var/lib/kubelet/seccomp/profiles/audit.json
```
- Profile JSON shape:
```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read", "write", "exit", "exit_group", "rt_sigreturn"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```
- Actions: `SCMP_ACT_ALLOW`, `SCMP_ACT_ERRNO` (deny), `SCMP_ACT_LOG` (allow + log to syslog), `SCMP_ACT_KILL`.
- Make RuntimeDefault the default for all pods on a node: `seccompDefault: true` in kubelet config.
- Verify: `k exec <pod> -- grep Seccomp /proc/1/status` → `Seccomp: 2` means a filter is active.
- If the pod is stuck in `CreateContainerError`, the profile file is missing on that node or the path is wrong.

---

## 4. Minimize Microservice Vulnerabilities (20%)

### Security Context
- Pod-level `securityContext` applies to all containers; container-level overrides it.
```yaml
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:1.27
    securityContext:
      allowPrivilegeEscalation: false
      privileged: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
```
- `privileged: true` always implies privilege escalation. The API rejects `privileged: true` together with `allowPrivilegeEscalation: false`.
- Other risky pod fields: `hostPID`, `hostIPC`, `hostNetwork`, `hostPath` volumes (especially `/`, `/var/run/containerd/containerd.sock`, `/etc`), added `SYS_ADMIN`/`NET_ADMIN` capabilities.

### Pod Security Standards (PSS) & Pod Security Admission (PSA)
- Three levels:
  - **privileged**: no restrictions.
  - **baseline**: blocks known privilege escalations (privileged, host namespaces, hostPath, hostPorts, extra capabilities, etc.).
  - **restricted**: baseline + must run as non-root, `allowPrivilegeEscalation: false`, `capabilities.drop: ["ALL"]` (only `NET_BIND_SERVICE` may be added), seccomp `RuntimeDefault` or `Localhost`, limited volume types (configMap, csi, downwardAPI, emptyDir, ephemeral, persistentVolumeClaim, projected, secret).
- Three modes: `enforce` (reject), `audit` (audit log annotation), `warn` (warning to user).
```bash
k label ns team-a \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted

# Preview which existing pods would violate, without applying:
k label --dry-run=server --overwrite ns team-a pod-security.kubernetes.io/enforce=restricted
```
- ⚠️ Enforce only checks **new** pods. Existing pods keep running. Delete them so they're recreated, and check why a Deployment has 0 pods with `k get events -n team-a` or `k describe rs -n team-a` (the ReplicaSet shows the PSA rejection).
- Restricted-compliant container template (copy this):
```yaml
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      seccompProfile:
        type: RuntimeDefault
```
- Cluster-wide defaults via the PodSecurity admission plugin config (`--admission-control-config-file`):
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "baseline"
      enforce-version: "latest"
      audit: "restricted"
      audit-version: "latest"
      warn: "restricted"
      warn-version: "latest"
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces: [kube-system]
```

### Managing Kubernetes Secrets
- Secrets are only **base64-encoded**, not encrypted, unless encryption at rest is configured.
```bash
k create secret generic db-creds -n app --from-literal=user=admin --from-literal=pass=S3cr3t
k get secret db-creds -n app -o jsonpath='{.data.pass}' | base64 -d
```
- Prefer **volume mounts** over env vars: env vars leak through `/proc/<pid>/environ`, `crictl inspect`, crash dumps and child processes.
```yaml
  containers:
  - name: app
    image: nginx:1.27
    volumeMounts:
    - name: creds
      mountPath: /etc/creds
      readOnly: true
  volumes:
  - name: creds
    secret:
      secretName: db-creds
```
- Restrict who can `get`/`list`/`watch` secrets via RBAC. Use `immutable: true` for secrets that shouldn't change.
- Reading a secret directly from etcd (common task: "find secret X and write its value to a file"):
```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/<namespace>/<secret-name>
```
- Reading secrets from a running container on the node: `crictl ps | grep <pod>` → `crictl inspect <id> | grep -A5 -i env` or `crictl exec <id> cat /etc/creds/pass`.

### Encryption at Rest
```bash
head -c 32 /dev/urandom | base64          # generate a 32-byte key
mkdir -p /etc/kubernetes/enc
```
```yaml
# /etc/kubernetes/enc/enc.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:                       # FIRST provider is used to WRITE
          keys:
            - name: key1
              secret: <BASE64-32-BYTE-KEY>
      - identity: {}                  # still lets the API server READ old unencrypted data
```
- kube-apiserver manifest:
```yaml
    - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
    ...
    volumeMounts:
    - name: enc
      mountPath: /etc/kubernetes/enc
      readOnly: true
  volumes:
  - name: enc
    hostPath:
      path: /etc/kubernetes/enc
      type: DirectoryOrCreate
```
- Providers: `identity` (no encryption), `aescbc`, `aesgcm`, `secretbox`, `kms` (v2). If `identity` is first, nothing gets encrypted.
- Existing secrets stay unencrypted until rewritten:
```bash
k get secrets -A -o json | k replace -f -
```
- Verify (should show `k8s:enc:aescbc:v1:key1` prefix, not plain text):
```bash
ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/app/db-creds | hexdump -C | head
```

### Isolation Techniques (Multi-tenancy, Sandboxed Containers)
- Normal containers share the host kernel. **Sandboxed runtimes** add a kernel boundary: **gVisor** (`runsc`, user-space kernel) or **Kata Containers** (lightweight VM).
- RuntimeClass maps a name to a runtime handler configured in containerd:
```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc            # must match the runtime name in /etc/containerd/config.toml
```
```yaml
spec:
  runtimeClassName: gvisor
```
```bash
k exec <pod> -- dmesg      # gVisor prints its own boot messages ("Starting gVisor...")
k exec <pod> -- uname -r   # differs from the node's kernel
```
- If the handler isn't configured yet, look at the existing runtime entries in `/etc/containerd/config.toml`, add `runsc` in the same format, then `systemctl restart containerd`.
- Multi-tenancy toolbox: namespaces + RBAC + NetworkPolicy (default deny) + ResourceQuota/LimitRange + PSA + dedicated nodes (taints/tolerations + nodeSelector) + RuntimeClass.
- User namespaces: `spec.hostUsers: false` maps container root to an unprivileged user on the host (needs node support, see "User Namespaces" docs page).

### Pod-to-Pod Encryption with Cilium
- **Transparent encryption** (WireGuard or IPsec) encrypts all pod traffic between nodes:
```bash
helm upgrade cilium cilium/cilium -n kube-system --reuse-values \
  --set encryption.enabled=true \
  --set encryption.type=wireguard
k -n kube-system rollout restart ds/cilium
k -n kube-system exec ds/cilium -- cilium-dbg status | grep Encryption
```
  (Not installed with Helm? Set `enable-wireguard: "true"` in the `cilium-config` ConfigMap in kube-system and restart the DaemonSet.)
- **CiliumNetworkPolicy** (L3/L4/L7, entities, FQDNs, explicit deny):
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: backend-from-frontend
  namespace: app
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
```
- Explicit deny (beats any allow):
```yaml
spec:
  endpointSelector:
    matchLabels:
      app: frontend
  egressDeny:
  - toEndpoints:
    - matchLabels:
        app: db
```
- Useful entities: `world` (outside cluster), `cluster`, `host`, `kube-apiserver`, e.g. `toEntities: [kube-apiserver]`.
- **Mutual authentication** (beta, SPIFFE/SPIRE based; Helm values `authentication.mutual.spire.enabled=true` and `authentication.mutual.spire.install.enabled=true`). Require it on a rule:
```yaml
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    authentication:
      mode: "required"
```

### Pod-to-Pod Encryption with Istio (mTLS)
```bash
k label ns payments istio-injection=enabled     # sidecar mode (ambient: istio.io/dataplane-mode=ambient)
k rollout restart deploy -n payments            # existing pods need restart to get the sidecar
```
```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: payments        # put it in istio-system for mesh-wide
spec:
  mtls:
    mode: STRICT             # PERMISSIVE accepts plaintext too
```

---

## 5. Supply Chain Security (20%)

### Minimize Base Image Footprint
- Smaller image = fewer packages = fewer CVEs and fewer tools for an attacker.
- Rules: pin specific tags (or digests), use minimal bases (distroless, alpine, `-slim`), multi-stage builds, run as non-root `USER`, no secrets in `ENV`/layers, no package managers/shells/curl in the final image, `COPY` instead of `ADD`, combine `RUN` steps and clean caches.
```dockerfile
# BAD
FROM ubuntu:latest
RUN apt-get update && apt-get install -y curl vim
ADD . /app
ENV DB_PASSWORD=supersecret
CMD ["/app/server"]

# GOOD
FROM golang:1.23 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/server /server
USER 65532:65532
ENTRYPOINT ["/server"]
```
- Pin by digest in manifests: `image: nginx@sha256:<digest>` (get it with `crictl images --digests`).

### Understand Your Supply Chain: SBOM
- SBOM (Software Bill of Materials) = list of all packages/versions in an artifact. Formats: **SPDX** and **CycloneDX**.
- **bom** (docs allowed in exam):
```bash
bom generate --image registry.k8s.io/kube-apiserver:v1.35.0 --format json --output /root/apiserver.spdx.json
bom generate --image nginx:1.27 --output /root/nginx.spdx          # default format: SPDX tag-value
bom generate --image-archive /root/app.tar --output /root/app.spdx  # image tarball
bom document outline /root/nginx.spdx                               # tree view of packages
```
- **Trivy** can also produce and scan SBOMs:
```bash
trivy image --format cyclonedx --output /root/sbom.cdx.json nginx:1.27
trivy image --format spdx-json --output /root/sbom.spdx.json nginx:1.27
trivy sbom /root/sbom.cdx.json        # scan an existing SBOM for CVEs
```
- Find which image contains a package: generate one SBOM per image, then `grep -il 'libcrypto3' /root/*.spdx*` and check the version line.

### Image Vulnerability Scanning with Trivy
```bash
trivy image nginx:1.19
trivy image -s HIGH,CRITICAL nginx:1.19               # --severity
trivy image -q -s CRITICAL --ignore-unfixed nginx:1.19
trivy image --input /root/app.tar                     # image tarball
trivy image -f json -o /root/report.json nginx:1.19
trivy fs /path/to/project; trivy config ./manifests/  # misconfig scanning
```
- Scan every image used in a namespace:
```bash
for img in $(k get pods -n stage -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u); do
  echo "== $img"; trivy image -q -s HIGH,CRITICAL "$img" | grep Total
done

# which pod uses which image
k get pods -n stage -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'
```

### Static Analysis of Workloads (kubesec, KubeLinter)
```bash
kubesec scan pod.yaml                                  # JSON with score, "critical" and "advise"
docker run -i kubesec/kubesec:v2 scan /dev/stdin < pod.yaml

kube-linter lint pod.yaml
kube-linter lint ./manifests/
kube-linter checks list
```
- Manual red flags in YAML/Dockerfiles: `privileged: true`, `hostPath: /`, `hostPID/hostNetwork`, `runAsUser: 0`, `SYS_ADMIN`, secrets in plain env, `:latest`, no resource limits, writable root FS, `USER root`/no `USER`.

### Secure Your Supply Chain: Permitted Registries with ValidatingAdmissionPolicy
- Built in and GA (no webhook needed). Uses CEL expressions. Needs a Policy + a Binding (no binding = no effect).
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: allowed-registries
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["pods"]
  validations:
  - expression: >-
      object.spec.containers.all(c, c.image.startsWith('registry.corp.io/')) &&
      (!has(object.spec.initContainers) ||
       object.spec.initContainers.all(c, c.image.startsWith('registry.corp.io/')))
    message: "Only images from registry.corp.io are allowed"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: allowed-registries-binding
spec:
  policyName: allowed-registries
  validationActions: [Deny]          # Deny | Warn | Audit
  matchResources:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: prod
```
- Other handy expressions: no latest tag `object.spec.containers.all(c, !c.image.endsWith(':latest'))`, require label `has(object.metadata.labels.team)`.

### Secure Your Supply Chain: ImagePolicyWebhook
- Admission plugin that asks an external webhook "is this image allowed?".
```yaml
# /etc/kubernetes/admission/admission-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/admission/kubeconf
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false       # fail closed: deny pods if webhook is unreachable
```
```yaml
# /etc/kubernetes/admission/kubeconf
apiVersion: v1
kind: Config
clusters:
- name: image-checker
  cluster:
    certificate-authority: /etc/kubernetes/admission/external-root-ca.pem
    server: https://image-checker.example.com/image_policy
contexts:
- name: image-checker
  context:
    cluster: image-checker
    user: api-server
current-context: image-checker
users:
- name: api-server
  user:
    client-certificate: /etc/kubernetes/admission/api-server-client.pem
    client-key: /etc/kubernetes/admission/api-server-client-key.pem
```
- kube-apiserver flags (+ mount `/etc/kubernetes/admission`):
```yaml
    - --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
    - --admission-control-config-file=/etc/kubernetes/admission/admission-config.yaml
```
- Test: `k run test --image=nginx` should be denied when the backend is down and `defaultAllow: false`.

### Sign and Validate Artifacts (cosign)
```bash
cosign generate-key-pair                                  # cosign.key / cosign.pub
cosign sign --key cosign.key registry.corp.io/app@sha256:<digest>
cosign verify --key cosign.pub registry.corp.io/app:1.0
```
- Enforcement in-cluster is done by admission (policy engine or webhook) that only admits verified images.

---

## 6. Monitoring, Logging and Runtime Security (20%)

### Falco (Behavioral Analytics)
- Falco watches syscalls (via eBPF/kernel module) and fires alerts when a rule's `condition` matches.
- Files:
  - `/etc/falco/falco.yaml` – main config (outputs, `rules_files` order).
  - `/etc/falco/falco_rules.yaml` – default rules. **Don't edit**, it gets overwritten.
  - `/etc/falco/falco_rules.local.yaml` and `/etc/falco/rules.d/` – your custom rules/overrides (loaded after the defaults).
- Running it:
```bash
systemctl list-units | grep -i falco        # unit is often falco-modern-bpf / falco-kmod / falco
systemctl restart <falco-unit>
journalctl -u <falco-unit> -f               # alerts (or /var/log/syslog)

# Run in foreground for N seconds with specific rules files
falco -M 30 -r /etc/falco/falco_rules.yaml -r /etc/falco/falco_rules.local.yaml
```
- Rule anatomy:
```yaml
- rule: Shell in container
  desc: Detect a shell spawned inside a container
  condition: spawned_process and container and proc.name in (bash, sh, zsh)
  output: "%evt.time,%container.id,%container.name,%user.name,%proc.cmdline"
  priority: WARNING        # EMERGENCY ALERT CRITICAL ERROR WARNING NOTICE INFO DEBUG
  tags: [container, shell]

- rule: Access to /dev/mem
  desc: Process opens /dev/mem in a container
  condition: container and fd.name = /dev/mem and evt.type in (open, openat, openat2)
  output: "%evt.time,%container.id,%container.name,%proc.name,%user.name"
  priority: CRITICAL
```
- Most used macros: `spawned_process`, `container`, `open_read`, `open_write`. Most used fields: `evt.time`, `evt.type`, `proc.name`, `proc.cmdline`, `proc.pname`, `user.name`, `user.uid`, `fd.name`, `container.id`, `container.name`, `container.image.repository`, `k8s.pod.name`, `k8s.ns.name`. Full list: `falco --list` or the "Supported Fields" docs page.
- **Change an existing rule** (e.g. its output format) from the local file using `override` (the old `append: true` is deprecated):
```yaml
# /etc/falco/falco_rules.local.yaml
- rule: Terminal shell in container
  output: "%evt.time,%container.id,%container.name,%user.name"
  override:
    output: replace

- list: my_programs
  items: [nc, ncat]
  override:
    items: append

- rule: Some disabled rule
  enabled: true
  override:
    enabled: replace
```
- Log alerts to a file (`falco.yaml`), then restart Falco:
```yaml
file_output:
  enabled: true
  keep_alive: false
  filename: /var/log/falco-alerts.txt
```
- From alert to pod:
```bash
crictl ps --id <container-id>                       # POD column shows the pod
crictl inspect <container-id> | grep -E 'io.kubernetes.pod.(name|namespace)'
k scale deploy <name> -n <ns> --replicas=0          # what tasks usually ask
```

### Kubernetes Audit Logs
- Levels: `None`, `Metadata` (who/what/when, no bodies), `Request` (+ request body), `RequestResponse` (+ response body).
- Stages: `RequestReceived`, `ResponseStarted`, `ResponseComplete`, `Panic`.
- ⚠️ **Rules are evaluated top to bottom, FIRST match wins.** Put specific rules (and `None` exclusions) before broad ones. Log secrets at `Metadata` only, otherwise their values end up in the log.
```yaml
# /etc/kubernetes/audit/policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  # 1. nothing for noisy system watches
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
  # 2. secrets/configmaps: metadata only
  - level: Metadata
    resources:
    - group: ""
      resources: ["secrets", "configmaps"]
  # 3. full body for pod changes in prod
  - level: RequestResponse
    namespaces: ["prod"]
    verbs: ["create", "update", "patch", "delete"]
    resources:
    - group: ""
      resources: ["pods"]
  # 4. deployments: request body
  - level: Request
    resources:
    - group: "apps"
      resources: ["deployments"]
  # 5. catch-all
  - level: Metadata
```
- kube-apiserver flags + volumes (Auditing docs page, search `audit-policy-file`):
```yaml
    - --audit-policy-file=/etc/kubernetes/audit/policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-log-maxage=30          # days to keep
    - --audit-log-maxbackup=10       # number of rotated files
    - --audit-log-maxsize=100        # MB before rotation
    ...
    volumeMounts:
    - mountPath: /etc/kubernetes/audit/policy.yaml
      name: audit
      readOnly: true
    - mountPath: /var/log/kubernetes/audit/
      name: audit-log
      readOnly: false
  volumes:
  - name: audit
    hostPath:
      path: /etc/kubernetes/audit/policy.yaml
      type: File
  - name: audit-log
    hostPath:
      path: /var/log/kubernetes/audit/
      type: DirectoryOrCreate
```
- Changing only the policy file doesn't reload it. Restart the API server (move the manifest out and back).
- Searching the log (one JSON object per line):
```bash
tail -f /var/log/kubernetes/audit/audit.log | jq .
jq -c 'select(.objectRef.resource=="secrets" and .verb=="get")
       | {user: .user.username, ns: .objectRef.namespace, name: .objectRef.name, time: .requestReceivedTimestamp}' \
  /var/log/kubernetes/audit/audit.log
grep '"resource":"secrets"' /var/log/kubernetes/audit/audit.log | grep -v 'system:' | tail
```

### Investigate Threats in Containers and Processes
```bash
crictl ps -a; crictl pods
crictl inspect <cid> | grep -m1 '"pid"'          # host PID of the container's main process
crictl logs <cid>
ps -ef --forest                                   # parent/child tree, spot odd children
ls -l /proc/<PID>/exe                             # real binary
cat /proc/<PID>/cmdline | tr '\0' ' '; echo
cat /proc/<PID>/environ | tr '\0' '\n'            # env vars (secrets leak here)
find /proc/*/fd -lname '/dev/mem' 2>/dev/null     # which PID has /dev/mem open
lsof -p <PID>                                     # open files/sockets
strace -f -p <PID> -e trace=openat                # live syscalls of a process
```

### Phases of an Attack (what to look for, where)

| Phase | Example in a cluster | Evidence |
|---|---|---|
| Initial access | Vulnerable app, leaked kubeconfig, anonymous kubelet/API | Audit log (`system:anonymous`), ingress logs |
| Execution | `kubectl exec`, shell spawned in container | Audit (`pods/exec`), Falco shell rules |
| Persistence | New RoleBinding/SA, CronJob, file in `/etc/kubernetes/manifests` | Audit (create on rbac resources) |
| Privilege escalation | Privileged pod, `hostPath: /`, `hostPID` + nsenter | PSA warnings, Falco, audit |
| Credential access | Reading secrets/SA tokens, `/etc/shadow`, metadata endpoint | Audit (`get secrets`), Falco sensitive file rules |
| Discovery | `auth can-i --list`, listing everything | Audit (`selfsubjectrulesreviews`) |
| Lateral movement | Pod → kubelet 10250, pod → other namespaces | NetworkPolicy/Cilium drops, Hubble |
| Impact | Crypto miner, deleting workloads, exfiltration | CPU spikes, audit deletes, Falco outbound rules |

### Ensure Immutability of Containers at Runtime
- `readOnlyRootFilesystem: true` + `emptyDir` for paths the app must write to (`/tmp`, `/var/cache/nginx`, `/var/run`):
```yaml
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
    volumeMounts:
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```
- No `privileged`, no running as root, no shells/package managers in image. Change things by redeploying, never by `kubectl exec`.
- Find pods that are NOT immutable / are privileged:
```bash
k get pods -A -o json | jq -r '.items[]
  | select(any(.spec.containers[]; .securityContext.privileged == true))
  | .metadata.namespace + "/" + .metadata.name'

k get pods -A -o json | jq -r '.items[]
  | select(any(.spec.containers[]; .securityContext.readOnlyRootFilesystem != true))
  | .metadata.namespace + "/" + .metadata.name'
```

---

## Important Paths

| What | Path |
|---|---|
| Static pod manifests | `/etc/kubernetes/manifests/` |
| Cluster PKI | `/etc/kubernetes/pki/`, etcd: `/etc/kubernetes/pki/etcd/` |
| Kubelet config | `/var/lib/kubelet/config.yaml` |
| Kubelet systemd drop-in | `/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf` (or `/etc/systemd/system/kubelet.service.d/`) |
| Kubelet kubeconfig | `/etc/kubernetes/kubelet.conf` |
| Seccomp profiles | `/var/lib/kubelet/seccomp/` |
| AppArmor profiles | `/etc/apparmor.d/` |
| Falco | `/etc/falco/falco.yaml`, `/etc/falco/falco_rules.local.yaml`, `/etc/falco/rules.d/` |
| Audit log (by convention) | `/var/log/kubernetes/audit/` |
| Pod logs on node | `/var/log/pods/`, `/var/log/containers/` |
| containerd config | `/etc/containerd/config.toml` |
| Kernel module blacklist | `/etc/modprobe.d/` |
| SSH / sudo | `/etc/ssh/sshd_config`, `/etc/sudoers`, `/etc/sudoers.d/` |

---

## IMPORTANT ⚠️

### Practice Questions (Scenario Based) with Solutions
- Try each one in a lab (killer.sh, KodeKloud, or a kubeadm cluster) before opening the solution.

---

**Q1. CIS Benchmark** (host: `cks-cp`, plus `node01`)
kube-bench reports these FAILs. Fix them all:
(a) kube-apiserver profiling is enabled, (b) kube-controller-manager profiling is enabled, (c) `/var/lib/etcd` is not owned by `etcd:etcd`, (d) on `node01` kubelet allows anonymous auth and uses `AlwaysAllow` authorization.

<details>
<summary>Solution</summary>

```bash
ssh cks-cp; sudo -i
cp /etc/kubernetes/manifests/kube-apiserver.yaml ~/
vi /etc/kubernetes/manifests/kube-apiserver.yaml            # add: - --profiling=false
vi /etc/kubernetes/manifests/kube-controller-manager.yaml   # add: - --profiling=false
id etcd || useradd -r -s /sbin/nologin etcd
chown -R etcd:etcd /var/lib/etcd
watch crictl ps                                             # wait for both pods
kube-bench run --targets=master | grep -E 'FAIL|profiling'
exit

ssh node01; sudo -i
vi /var/lib/kubelet/config.yaml
#   authentication.anonymous.enabled: false
#   authorization.mode: Webhook
systemctl daemon-reload && systemctl restart kubelet
kube-bench run --targets=node | grep FAIL
exit
```

</details>

---

**Q2. NetworkPolicy**
In namespace `restricted`: deny all ingress and egress by default. Pods with `app=api` must accept traffic only from pods labelled `tier=web` in namespace `frontend`, on TCP 8080. All pods in `restricted` must still resolve DNS.

<details>
<summary>Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: restricted
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: restricted
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-from-web
  namespace: restricted
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:                       # AND: same list item
        matchLabels:
          kubernetes.io/metadata.name: frontend
      podSelector:
        matchLabels:
          tier: web
    ports:
    - port: 8080
      protocol: TCP
```

</details>

---

**Q3. Node Metadata**
Pods in namespace `apps` must not reach `169.254.169.254`, but all other egress must keep working. Pods with label `role=metadata-reader` are an exception and may reach it.

<details>
<summary>Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-metadata
  namespace: apps
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-metadata-reader
  namespace: apps
spec:
  podSelector:
    matchLabels:
      role: metadata-reader
  policyTypes: [Egress]
  egress:
  - to:
    - ipBlock:
        cidr: 169.254.169.254/32
```

</details>

---

**Q4. Ingress with TLS**
Cert and key are in `/opt/certs/`. Create secret `web-tls` in namespace `shop`, and an Ingress `web` (class `nginx`) for host `shop.local` → service `web:80`. HTTP must redirect to HTTPS.

<details>
<summary>Solution</summary>

```bash
k create secret tls web-tls -n shop --cert=/opt/certs/tls.crt --key=/opt/certs/tls.key
```
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: shop
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts: [shop.local]
    secretName: web-tls
  rules:
  - host: shop.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```
```bash
curl -kv https://shop.local:<https-nodeport> --resolve shop.local:<https-nodeport>:<node-ip>
curl -I http://shop.local:<http-nodeport> --resolve shop.local:<http-nodeport>:<node-ip>   # 308
```

</details>

---

**Q5. Verify Binaries** (host: `node01`)
Expected SHA512 hashes for `kubelet`, `kubectl` and `kube-proxy` are in `/opt/hashes.txt`. Delete any binary in `/opt/bin/` that doesn't match.

<details>
<summary>Solution</summary>

```bash
ssh node01; sudo -i
cat /opt/hashes.txt
cd /opt/bin && sha512sum kubelet kubectl kube-proxy
# compare visually, or if hashes.txt is in "hash  filename" format:
sha512sum -c /opt/hashes.txt
rm -f /opt/bin/<mismatched-binary>
```

</details>

---

**Q6. RBAC**
ServiceAccount `logger` in namespace `ops` is bound to Role `logger-role`, which allows `*` on `*`. Change it so the SA can only `get`, `list` pods and `get` pods/log in `ops`. Verify.

<details>
<summary>Solution</summary>

```bash
k get rolebinding -n ops -o wide | grep logger
k edit role logger-role -n ops
```
```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```
```bash
k auth can-i list pods -n ops --as=system:serviceaccount:ops:logger      # yes
k auth can-i get secrets -n ops --as=system:serviceaccount:ops:logger    # no
k auth can-i get pods/log -n ops --as=system:serviceaccount:ops:logger   # yes
```

</details>

---

**Q7. Service Accounts**
Deployment `backend` in `payments` uses the `default` SA and has a token mounted. Create SA `backend-sa` (no token automount), switch the deployment to it, and make sure the `default` SA in `payments` also doesn't automount tokens.

<details>
<summary>Solution</summary>

```bash
k create sa backend-sa -n payments
k patch sa backend-sa -n payments -p '{"automountServiceAccountToken": false}'
k patch sa default -n payments -p '{"automountServiceAccountToken": false}'
k set serviceaccount deploy/backend backend-sa -n payments
k rollout status deploy/backend -n payments
k exec -n payments deploy/backend -- ls /var/run/secrets/kubernetes.io/serviceaccount   # No such file
```

</details>

---

**Q8. Restrict API Access** (host: `cks-cp`)
The API server is reachable on a NodePort and the `NodeRestriction` admission plugin is not enabled. Authorization mode is `AlwaysAllow`. Fix all three.

<details>
<summary>Solution</summary>

```bash
ssh cks-cp; sudo -i
cp /etc/kubernetes/manifests/kube-apiserver.yaml ~/
vi /etc/kubernetes/manifests/kube-apiserver.yaml
#   remove: - --kubernetes-service-node-port=31000
#   set:    - --authorization-mode=Node,RBAC
#   set:    - --enable-admission-plugins=NodeRestriction
watch crictl ps
k delete svc kubernetes            # recreated automatically as ClusterIP
k get svc kubernetes
```

</details>

---

**Q9. Host Hardening** (host: `node02`)
Something listens on TCP port `6666`. Kill it and delete its binary. Also disable and stop the `vsftpd` service, purge the `telnetd` package, and remove user `devuser` from the `sudo` group.

<details>
<summary>Solution</summary>

```bash
ssh node02; sudo -i
ss -tlnp | grep 6666            # users:(("evil",pid=1234,...))
ls -l /proc/1234/exe            # -> /usr/local/bin/evil
kill -9 1234 && rm -f /usr/local/bin/evil
systemctl disable --now vsftpd
apt purge -y telnetd
gpasswd -d devuser sudo
```

</details>

---

**Q10. AppArmor** (host: `node01`)
Profile file `/opt/apparmor/nginx-deny` exists on `node01`. Load it in enforce mode, then create pod `secure-nginx` (image `nginx:1.27`) on `node01` that uses it.

<details>
<summary>Solution</summary>

```bash
ssh node01; sudo -i
grep profile /opt/apparmor/nginx-deny        # e.g. "profile custom-nginx-deny flags=..."
apparmor_parser -q /opt/apparmor/nginx-deny
aa-status | grep custom-nginx-deny
exit
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-nginx
spec:
  nodeName: node01
  containers:
  - name: nginx
    image: nginx:1.27
    securityContext:
      appArmorProfile:
        type: Localhost
        localhostProfile: custom-nginx-deny     # profile NAME, not file name
```
```bash
k exec secure-nginx -- cat /proc/1/attr/current
```

</details>

---

**Q11. Seccomp** (host: `node01`)
Seccomp profile `/opt/seccomp/audit.json` is given. Make it usable on `node01` and create pod `audit-pod` (image `busybox:1.36`, `sleep 3600`) on `node01` that uses it. Also make `RuntimeDefault` the default seccomp profile for all pods on `node01`.

<details>
<summary>Solution</summary>

```bash
ssh node01; sudo -i
mkdir -p /var/lib/kubelet/seccomp/profiles
cp /opt/seccomp/audit.json /var/lib/kubelet/seccomp/profiles/
vi /var/lib/kubelet/config.yaml              # add: seccompDefault: true
systemctl restart kubelet
exit
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: audit-pod
spec:
  nodeName: node01
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
  - name: c
    image: busybox:1.36
    command: ["sleep", "3600"]
```

</details>

---

**Q12. Pod Security Admission**
Namespace `team-a` must enforce the `restricted` standard (latest). Deployment `api` in `team-a` then stops creating pods. Fix the deployment so it runs.

<details>
<summary>Solution</summary>

```bash
k label ns team-a pod-security.kubernetes.io/enforce=restricted pod-security.kubernetes.io/enforce-version=latest
k delete pod -n team-a -l app=api           # force recreation, existing pods aren't re-checked
k get events -n team-a | grep -i violat     # shows which fields fail
k edit deploy api -n team-a
```
```yaml
# under spec.template.spec.containers[0]:
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
          seccompProfile:
            type: RuntimeDefault
# also remove any hostPath volumes / hostNetwork / privileged
```

</details>

---

**Q13. Immutable Container**
Deployment `web` in `prod` (nginx): containers must run with a read-only root filesystem, without privilege escalation, and as user/group `101`. nginx needs to write to `/var/cache/nginx` and `/var/run`.

<details>
<summary>Solution</summary>

```yaml
# k edit deploy web -n prod
    spec:
      securityContext:
        runAsUser: 101
        runAsGroup: 101
      containers:
      - name: nginx
        image: nginx:1.27
        securityContext:
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
        volumeMounts:
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
      volumes:
      - name: cache
        emptyDir: {}
      - name: run
        emptyDir: {}
```
```bash
k exec -n prod deploy/web -- touch /test     # Read-only file system
```
- Note: stock nginx listens on 80, which a non-root user can't bind unless the image is the unprivileged variant or `NET_BIND_SERVICE` is added.

</details>

---

**Q14. Encryption at Rest** (host: `cks-cp`)
Encrypt all Secrets at rest with `aescbc` (key name `key1`) using a config at `/etc/kubernetes/enc/enc.yaml`. Make sure existing secrets get encrypted and verify one in etcd.

<details>
<summary>Solution</summary>

```bash
ssh cks-cp; sudo -i
mkdir -p /etc/kubernetes/enc
KEY=$(head -c 32 /dev/urandom | base64)
cat > /etc/kubernetes/enc/enc.yaml <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: $KEY
      - identity: {}
EOF
cp /etc/kubernetes/manifests/kube-apiserver.yaml ~/
vi /etc/kubernetes/manifests/kube-apiserver.yaml
#  - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
#  + volumeMount /etc/kubernetes/enc (readOnly) + hostPath volume
watch crictl ps
k get secrets -A -o json | k replace -f -
ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/<some-secret> | hexdump -C | head   # k8s:enc:aescbc:v1:key1
```

</details>

---

**Q15. Secrets Hygiene**
Pod `report` in namespace `finance` reads secret `db-creds` through env vars. Write the decoded `password` value to `/opt/db-pass.txt`, then change the pod so the secret is mounted read-only at `/etc/db` instead of env vars.

<details>
<summary>Solution</summary>

```bash
k get secret db-creds -n finance -o jsonpath='{.data.password}' | base64 -d > /opt/db-pass.txt
k get pod report -n finance -o yaml > report.yaml
vi report.yaml        # remove env/envFrom secret refs, add volume + volumeMount below
k replace --force -f report.yaml
```
```yaml
    volumeMounts:
    - name: db
      mountPath: /etc/db
      readOnly: true
  volumes:
  - name: db
    secret:
      secretName: db-creds
```

</details>

---

**Q16. Sandboxed Runtime**
gVisor (`runsc`) is configured in containerd on `node01`. Create RuntimeClass `gvisor` and make all pods of deployment `untrusted` (namespace `sandbox`) run with it on `node01`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
```
```bash
k patch deploy untrusted -n sandbox -p \
  '{"spec":{"template":{"spec":{"runtimeClassName":"gvisor","nodeName":"node01"}}}}'
k exec -n sandbox deploy/untrusted -- dmesg | head -3     # gVisor messages
```

</details>

---

**Q17. Cilium**
Namespace `shop`: (a) pods with `app=frontend` must never reach pods with `app=db` (explicit deny), (b) traffic from `app=frontend` to `app=backend` must require Cilium mutual authentication.

<details>
<summary>Solution</summary>

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: frontend-deny-db
  namespace: shop
spec:
  endpointSelector:
    matchLabels:
      app: frontend
  egressDeny:
  - toEndpoints:
    - matchLabels:
        app: db
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: backend-mutual-auth
  namespace: shop
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    authentication:
      mode: "required"
```

</details>

---

**Q18. Istio mTLS**
Istio is installed. Enable sidecar injection for namespace `payments`, restart its workloads and enforce strict mTLS for that namespace.

<details>
<summary>Solution</summary>

```bash
k label ns payments istio-injection=enabled --overwrite
k rollout restart deploy -n payments
k get pods -n payments            # READY should show 2/2
```
```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: payments
spec:
  mtls:
    mode: STRICT
```

</details>

---

**Q19. Dockerfile & Manifest Static Analysis**
Fix `/opt/app/Dockerfile` (two security issues: base `ubuntu:latest`, runs as root) by pinning `ubuntu:24.04` and running as user `appuser` (UID 1001). Then run kubesec on `/opt/app/pod.yaml` and fix whatever it flags as critical.

<details>
<summary>Solution</summary>

```dockerfile
FROM ubuntu:24.04
RUN useradd -u 1001 -m appuser
...
USER appuser
```
```bash
kubesec scan /opt/app/pod.yaml | jq '.[0].scoring.critical'
# typical criticals: privileged: true, hostPID/hostNetwork, SYS_ADMIN, hostPath "/"
vi /opt/app/pod.yaml      # remove/flip them, then re-scan until score > 0 and no criticals
```
- Only change what the task asks. Don't rebuild or push the image unless told to.

</details>

---

**Q20. Image Scanning**
Using Trivy, find which pods in namespace `stage` use images with `CRITICAL` vulnerabilities and delete those pods.

<details>
<summary>Solution</summary>

```bash
k get pods -n stage -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'
trivy image -q -s CRITICAL nginx:1.19     | grep Total
trivy image -q -s CRITICAL redis:7.2      | grep Total
trivy image -q -s CRITICAL httpd:2.4.39   | grep Total
k delete pod <pods-with-CRITICAL-gt-0> -n stage
```

</details>

---

**Q21. SBOM**
Generate an SPDX-JSON SBOM of `registry.k8s.io/kube-controller-manager:v1.35.0` with `bom` at `/opt/sbom/kcm.json`. Then find which of the images in deployment `tools` (namespace `ci`) contains package `libcrypto3` version `3.1.4-r5` and remove that container from the deployment.

<details>
<summary>Solution</summary>

```bash
bom generate --image registry.k8s.io/kube-controller-manager:v1.35.0 --format json --output /opt/sbom/kcm.json

k get deploy tools -n ci -o jsonpath='{.spec.template.spec.containers[*].image}'
for img in alpine:3.19 alpine:3.18 nginx:1.25-alpine; do
  echo "== $img"; bom generate --image $img --format json 2>/dev/null | grep -A3 '"libcrypto3"' | grep versionInfo
done
# or: trivy image --format spdx-json $img | grep -A5 libcrypto3
k edit deploy tools -n ci     # delete the matching container
```

</details>

---

**Q22. ImagePolicyWebhook** (host: `cks-cp`)
Config files are prepared in `/etc/kubernetes/policywebhook/` (`admission_config.json`, `kubeconf`). Complete the config so pods are **denied** if the webhook is unreachable, point it at the kubeconfig, and enable the plugin.

<details>
<summary>Solution</summary>

```bash
ssh cks-cp; sudo -i
vi /etc/kubernetes/policywebhook/admission_config.json
#   "kubeConfigFile": "/etc/kubernetes/policywebhook/kubeconf",
#   "defaultAllow": false
vi /etc/kubernetes/policywebhook/kubeconf          # check server: URL + cert paths
cp /etc/kubernetes/manifests/kube-apiserver.yaml ~/
vi /etc/kubernetes/manifests/kube-apiserver.yaml
#  - --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
#  - --admission-control-config-file=/etc/kubernetes/policywebhook/admission_config.json
#  + volumeMount + hostPath for /etc/kubernetes/policywebhook
watch crictl ps
k run test --image=nginx        # should be forbidden if the backend isn't running
```

</details>

---

**Q23. ValidatingAdmissionPolicy**
In namespace `prod`, only images from `registry.corp.io/` may be used (containers and initContainers). Existing violations don't need fixing.

<details>
<summary>Solution</summary>

- Use the `allowed-registries` Policy + Binding from the Supply Chain section, with the binding's `namespaceSelector` on `kubernetes.io/metadata.name: prod` and `validationActions: [Deny]`.
```bash
k run bad --image=nginx -n prod                        # denied with your message
k run good --image=registry.corp.io/nginx:1.27 -n prod # admitted (may not pull, that's fine)
```

</details>

---

**Q24. Falco** (host: `node01`)
A pod on `node01` is spawning shells and another is reading `/dev/mem`. Make Falco log matching events in the format `time,container-id,container-name,user-name`, collect for ~30 s, save the alerts to `/opt/falco.log`, and scale the offending deployments to 0.

<details>
<summary>Solution</summary>

```bash
ssh node01; sudo -i
grep -n "rule: Terminal shell in container" -A20 /etc/falco/falco_rules.yaml   # check existing rule
cat >> /etc/falco/falco_rules.local.yaml <<'EOF'
- rule: Terminal shell in container
  output: "%evt.time,%container.id,%container.name,%user.name"
  override:
    output: replace

- rule: Access to /dev/mem
  desc: Process reads /dev/mem in a container
  condition: container and fd.name = /dev/mem and evt.type in (open, openat, openat2)
  output: "%evt.time,%container.id,%container.name,%user.name"
  priority: CRITICAL
EOF
falco -M 30 -r /etc/falco/falco_rules.yaml -r /etc/falco/falco_rules.local.yaml > /opt/falco.log
cat /opt/falco.log
crictl ps --id <container-id>          # get pod name(s)
exit
k scale deploy <name> -n <ns> --replicas=0
```
- If Falco runs as a service, use `systemctl restart <falco-unit>` and read `journalctl -u <falco-unit>` instead.
- No Falco at all? `find /proc/*/fd -lname '/dev/mem' 2>/dev/null` gives the PID → `crictl ps` / `crictl inspect` to map it to a pod.

</details>

---

**Q25. Audit Logging** (host: `cks-cp`)
Enable audit logging: policy at `/etc/kubernetes/audit/policy.yaml`, logs at `/var/log/kubernetes/audit/audit.log`, keep 2 backups, max 10 MB each, max 7 days. Policy: nothing for `system:nodes` group read-only requests, Secrets at `Metadata`, `RequestResponse` for Deployments in namespace `webapps`, and `Metadata` for everything else.

<details>
<summary>Solution</summary>

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"
rules:
- level: None
  userGroups: ["system:nodes"]
  verbs: ["get", "list", "watch"]
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]
- level: RequestResponse
  namespaces: ["webapps"]
  resources:
  - group: "apps"
    resources: ["deployments"]
- level: Metadata
```
```yaml
# kube-apiserver manifest
    - --audit-policy-file=/etc/kubernetes/audit/policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-log-maxbackup=2
    - --audit-log-maxsize=10
    - --audit-log-maxage=7
# + the two volumeMounts/hostPath volumes from the Audit section
```
```bash
watch crictl ps
k create deploy test --image=nginx -n webapps
tail -2 /var/log/kubernetes/audit/audit.log | jq .
```

</details>

---

**Q26. Audit Investigation**
Secret `db-creds` in namespace `prod` was deleted. Using the audit log, find who deleted it and write the username to `/opt/deleter.txt`. Also list every user/SA that read any secret in `prod`.

<details>
<summary>Solution</summary>

```bash
AL=/var/log/kubernetes/audit/audit.log
jq -r 'select(.objectRef.resource=="secrets" and .objectRef.namespace=="prod"
              and .objectRef.name=="db-creds" and .verb=="delete") | .user.username' $AL > /opt/deleter.txt

jq -r 'select(.objectRef.resource=="secrets" and .objectRef.namespace=="prod"
              and (.verb=="get" or .verb=="list")) | .user.username' $AL | sort -u
```

</details>

---

## Last-Day Checklist
- [ ] Can I write a default-deny + DNS NetworkPolicy and an AND-selector rule without docs?
- [ ] Do I know which apiserver flags need volume mounts (audit, encryption, admission config, authentication config)?
- [ ] AppArmor: profile name vs filename, loaded on the right node, `appArmorProfile` field.
- [ ] Seccomp: `/var/lib/kubelet/seccomp/` + relative `localhostProfile`.
- [ ] PSA labels by heart: `pod-security.kubernetes.io/enforce=restricted`.
- [ ] EncryptionConfiguration: first provider writes, `k get secrets -A -o json | k replace -f -`.
- [ ] Falco: local rules file, `override:`, output fields, container id → pod.
- [ ] Audit policy: first match wins, policy file change needs apiserver restart.
- [ ] Trivy `-s CRITICAL -q`, bom `generate --image --format json --output`.
- [ ] Always `exit` back to `base` after each task.
