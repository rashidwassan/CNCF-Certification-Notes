# KCSA

## 4 Cs of Cloud Native Security
1. **Cloud** - Datacenter, Network, Servers.
2. **Cluster** - Authentication, Authorization, Admission, Network Policy.
3. **Containers** - Restrict Images, Supply Chain, Sandboxing, Priveleged.
4. **Code** - Code Security Best Practices.

## Cloud Provider Security:
### Threat Management and Response Techniques
- Azure: Microsoft Sentinet, provides SIEM, SOAR.
- GCP: Security Command Center
- AWS: Guard Duty

### Web Application Firewalls (WAF)
- Azure: Azure WAF, provides security against SQL Injections and XSS Attacks.
- GCP: Google Cloud Armor, provides protection against DDoS Attack.
- AWS: AWS WAF, works as Load Balancer and with AWS CloudFront.

### Container Security:
- Azure: AKS
- GCP: GKE, Google's Anthos, Open Policy Agent.
- AWS: EKS, Bottlerocket, Kube-bench, CIS.

### Shared Responsibility Model:
- Cloud Interface like `network settings`, and `firewall settings` are managed by user.
- Cloud provider manages the `underlying hardware` that user does not has access to.
![alt text](images/kcsa/image.png)

### Infrastructure Security:
- Network Configuration & Server Hardening.

- **Stages:**
  - **Stage 1**: IP address isolation for each service. Masking public IP address of node, or enforcing VPN access to reduce risk.
  - **Stage 2**: Open ports like Docker port can cause compromise in access. Using firewalls to restrict traffic to port exposing Docker can help. Network policies can also be used for underlying worker node VMs.
  - **Stage 3**: Previleged container can be used to escalate attacker's priveleges to exploit a vulnerability. Adhering to the principle of least privelege can be helpful in this situation. RBAC should be used to restrict the access to the Kubernetes Dashboard.
  - **Stage 4**: Attacker can read credentials stored as environment variables in pods and use them. Using Kubernetes secrets would have protected in this case. Securing etcd by enforcing TLS communication and RBAC.

## Kubernetes Isolation
- Namespace separation for each env like dev, test, prod.
- Multi-tenancy with namespaces.
- Implementing RBAC.
- Resource quota and limits.
- Security contexts to manage user access inside containers or to run them in non-root user mode.

## Artifact Repository and Image Security:
- **Vulnerability Scanning Tools:** Trivy & Clair.
- Use minimal base images from trusted sources.
### Build Artifacts:
- Files generated as a package after build, can be, code, package, WAR file, logs, report, container image. These are often stored in artifact repositories like Docker, Nexus Repository or JFrog.
### Enhancing Image Security with Digital Signatures:
1. Proactive Security Alerts.
2. Addressing Security Concerns.
3. Applying Digital Signatures.
4. Ensuring Image Authenticity.

## Code Level Security Practices:
- SQL Injection Attacks can happen if code is not properly tested.
- **SonarQube:** Detecting problematic code patterns, mitigating identified risks.
- **Third Party Dependencies:** OWASP security check can be performed.
- **Realtime Security Monitoring:** Datadog Application Security Monitoring (ASM).
- **Performance Monitoring:** Sysdig provides deep insights into containerized environments.


## Kubernetes Cluster Component Security

### API Server
- Authentication and Authorization.

### Kubernetes Controller Manager and Scheduler
- Should be run on the separated, isolated node from where the applications are running.
- Managing permissions using RBAC, to restrict these components to certain permissions only.
- Using TLS communication between components for communication.
- Implement auditlogging to these components using tools like Prometheus and Grafana.

### Securing Kubelet
- Kubeadm does not install Kubelet by default.
- By default, you can do anonymous curl on Kube API Server to check the existing pods.
- There are two ports:
  - 10250 - Serves API that allows full access.
  - 10255 - Serves API that allows unauthenticated read-only access.
- This default anonymous access flag can be



## Multiple Choice Questions (MCQs)

---

### 1. **Question:**

An enterprise utilizing Azure for their cloud solutions requires a tool to perform security analysis and offer actionable recommendations to enhance their security posture.  
**Which Azure service is best suited for this purpose?**


### ✅ **Answer:**

**Azure Security Center**

Azure Security Center is a unified infrastructure security management system that strengthens the security posture of your data centers and provides advanced threat protection across all of your Azure resources.

---

### 2. **Question**
Scenario: A technology firm needs to ensure that all deployments in their Kubernetes cluster adhere to specific security protocols, including mandatory security labels.
**Which feature can enforce these deployment standards within the cluster?**


### ✅ **Answer:**

**Pod Security Admission (PSA)

---

### 3. **Question:**

A company has migrated its services to a public cloud provider. They need to ensure that the virtual machines hosting their applications are properly secured.
**Which entity is responsible for configuring the network security groups and firewalls for these virtual machines?**


### ✅ **Answer:**

**The Customer**

When migrating services to a public cloud provider, the customer is responsible for configuring network security groups and firewalls for their virtual machines to ensure proper security.

---
