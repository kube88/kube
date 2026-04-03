

# 🛡️ Kubernetes Security Lab: Calico Network Policies

## 🎯 Lab Objective
In this lab, you will explore how Kubernetes handles network security using Calico as the Container Network Interface (CNI) plugin. You will learn how to:
1. Verify Calico's presence in the cluster.
2. Establish a secure workspace and implement real-world namespace labeling.
3. Implement a "Namespace Isolation" policy using explicit label selectors.
4. Adopt a "Zero Trust" model by restricting traffic to specific ports.
5. Control external Egress traffic to the internet.
6. **Extension**: Use Calico-specific Custom Resources to explicitly Deny traffic.

---

### 🧠 Core Concepts: What are we actually doing?
Before jumping into the command line, it is important to understand the tools we are using:
* **What is a CNI?** 🌐 Kubernetes does not actually handle networking itself; it delegates that job to a CNI (Container Network Interface) plugin. The CNI assigns IP addresses to pods and routes traffic between them.
* **What is Calico?** 🐱 Calico is one of the most popular CNI plugins. Beyond just routing traffic, Calico is famous for its robust security features, enforcing firewall rules across your cluster.
* **The "Flat Network" Problem:** 🔓 By default, Kubernetes is designed so that *any* pod can talk to *any* other pod, even across different namespaces. If a hacker breaches a low-priority frontend application, they can freely scan and attack your backend databases.
* **Network Policies:** 🧱 These are Kubernetes rules that tell the CNI (Calico) how to build a firewall around a group of pods. Think of them as security groups or firewalls tailored specifically for microservices.

---

## 📋 Prerequisites
* Access to the command line in your dedicated HOL instance.

---

### 🛠️ Step 0: Initial Configuration
Before we can interact with the Kubernetes cluster, you must ensure your environment is configured correctly. Run the following commands to set up your Kubernetes configuration:

```bash
# 1. Create the .kube directory
mkdir -p ~/.kube

# 2. Copy the master config file to your local directory
cp /home/config ~/.kube/config

# 3. Secure the config file permissions
chmod 600 ~/.kube/config

# 4. Navigate into the lab directory
cd ~/kube
cd "FRI 01 - Networking"
```

💡 **What is happening here?** Kubernetes uses a configuration file (often called `kubeconfig`) to know which cluster to talk to and what credentials to use. By copying the shared config file to your home directory, you are giving your `kubectl` command the "keys" to the cluster. Finally, we navigate to the correct folder so that our YAML files are easy to find.

---

### 🔎 Step 2: Exploring the Calico CNI
Before we write security policies, we need to ensure Calico is managing our cluster's network.

Run the following command to look for Calico components in the `kube-system` namespace:

```bash
kubectl get pods -n kube-system -l k8s-app=calico-node
```

💡 **What is happening here?** We are filtering pods by the label `k8s-app=calico-node`. You should see a Calico pod running on every single node in your cluster. This architecture is called a **DaemonSet**. Because these Calico pods live on every physical server, they intercept the NetworkPolicies we write and translate them into actual low-level Linux firewall rules on the host machine.

---

### 🏗️ Step 3: Setting Up Your Workspace & Labels
It is a Kubernetes best practice to group related applications together. We will create a dedicated namespace called `lab` to hold our work. 

Furthermore, in enterprise environments, security policies rarely rely on hardcoded names. Instead, administrators apply **Labels** 🏷️ to namespaces (e.g., `environment=prod`) and write policies targeting those labels.

Create the namespace and label it:

```bash
kubectl create namespace lab
kubectl label namespace lab environment=student-lab
```

Now, create a specific context for your namespace and switch to using it:

```bash
kubectl config set-context lab-context --current --namespace=lab
kubectl config use-context lab-context
```

💡 **What is happening here?** A Kubernetes **Namespace** is a logical boundary—like a folder—that groups your resources. By attaching the `environment=student-lab` label, we can now write intelligent security rules that apply to this environment. Finally, by creating the `lab-context`, you are telling `kubectl` to send all future commands directly into this namespace. You will no longer need to type `-n lab`!

---

### 🔐 Step 4: Enforcing Namespace Isolation
Our first security goal is to fix the "flat network" problem. Imagine this cluster is shared by multiple teams. We want to lock down the `lab` namespace so our pods can *only* talk to other pods inside namespaces labeled `environment=student-lab`.

Create a file named `1-namespace-isolation.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
spec:
  podSelector: {} # 1. The Target: Applies this policy to ALL pods in our namespace
  policyTypes:
  - Ingress
  - Egress
  
  # 2. The Ingress Rule (Incoming traffic)
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          environment: student-lab # ONLY allow traffic from namespaces with this label
          
  # 3. The Egress Rule (Outgoing traffic)
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          environment: student-lab # ONLY allow traffic to namespaces with this label
  - ports: # 4. The DNS Exception
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

Apply the policy:
```bash
kubectl apply -f 1-namespace-isolation.yaml
```

💡 **What is happening here?**
* **The Target:** `podSelector: {}` acts as a wildcard, telling Calico to wrap this firewall around **all** pods in our namespace.
* **Default Deny:** By simply declaring `policyTypes: [Ingress, Egress]`, we activate a "default deny" shield 🛡️. Any traffic not matching the rules below will be dropped.
* **The Allow Rules:** Instead of guessing where traffic is coming from, we explicitly use `namespaceSelector`. This rule states: "Allow traffic in and out, *but only if the other side of the connection belongs to a namespace labeled `environment: student-lab`.*"
* **The DNS Exception:** Kubernetes relies heavily on DNS. If pods can't reach the CoreDNS servers, they can't resolve names like `http://web` into IP addresses. We explicitly open port 53 outward so DNS lookups don't break.

---

### 📦 Step 5: Deploying Test Pods
Let's deploy two pods to test our policies: a Web server and a Client pod.

Deploy an Nginx web server and expose it as a Service:
```bash
kubectl run web --image=nginx --labels="app=web" --expose --port=80
```

Deploy a temporary Client pod (we will use an Alpine Linux image with `curl` installed so we can easily test HTTP requests):
```bash
kubectl run client --image=alpine/curl --labels="app=client" -- sleep 3600
```

Verify both pods are running (Notice you don't need to specify your namespace here because of your context!):
```bash
kubectl get pods,svc
```

---

### 🧪 Step 6: Testing the Namespace Isolation
Let's execute a command from inside our `client` pod to see if it can reach the `web` pod.

```bash
kubectl exec client -- curl -s --connect-timeout 3 http://web
```
✅ *You should see the "Welcome to nginx!" HTML output.* 💡 **Why did this work?** Because both pods exist inside a namespace that possesses the `environment=student-lab` label. If an application in an untagged namespace tried to curl your web pod, Calico would drop the connection and it would eventually time out.

---

### 🚫 Step 7: Moving to Zero Trust (Granular Port Security)
While namespace isolation is good, it's not perfect. If a hacker breaches your `client` pod, they currently have full network access to your `web` pod on *any* port. 

The **Zero Trust** philosophy dictates "Least Privilege"—entities should only have the exact access they need to function, and nothing more. Let's tighten this up so the web server *only* accepts traffic on Port 80.

⚠️ **Important Concept:** Kubernetes NetworkPolicies are **additive**. This means they only grant access; they never explicitly deny it. Because our previous `namespace-isolation` policy already allows *all* traffic between pods, adding a new rule for Port 80 won't restrict other ports. To achieve true Zero Trust, we must delete our broad isolation policy and replace it with a strict Default Deny foundation.

Delete the broad policy:
```bash
kubectl delete networkpolicy namespace-isolation
```

Create a new file named `2-strict-security.yaml`:

```yaml
---
# 1. THE FOUNDATION: Block EVERYTHING by default (Default Deny)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# 2. THE INFRASTRUCTURE: Allow ALL pods to use DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
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
---
# 3. THE APPLICATION: Allow only Port 80 to the Web pod from the Client pod
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-port80
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client
    ports:
    - protocol: TCP
      port: 80
---
# 4. THE CLIENT: Allow the Client pod to reach the Web pod on Port 80
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-egress
spec:
  podSelector:
    matchLabels:
      app: client
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 80
```

Apply the new strict policies:
```bash
kubectl apply -f 2-strict-security.yaml
```

💡 **What is happening here?** We have broken our security posture into three distinct, manageable pieces. We lay down a total lockdown foundation 🧱, we poke a small hole for DNS infrastructure 🌐, and then we create highly specific, label-based rules allowing only exactly what our application needs to function (Client -> Web over Port 80) 🚪.

---

### 🕵️‍♂️ Step 8: Proving the Policies Work

Let's act as an attacker trying to pivot through the network to prove Calico is enforcing these strict rules.

**Test 1: Can the client access Port 80? (Should SUCCEED ✅)**
This is legitimate application traffic.
```bash
kubectl exec client -- curl -s --connect-timeout 3 http://web
```
*Result: You should see the Nginx HTML output.*

**Test 2: Can the client ping the web pod? (Should FAIL ❌)**
Hackers often use ICMP (Ping) to map out internal networks. Let's find the web pod's IP and try to ping it:
```bash
WEB_IP=$(kubectl get pod web --template '{{.status.podIP}}')
kubectl exec client -- ping -c 3 -W 1 $WEB_IP
```
*Result: 100% packet loss. Calico drops the ICMP packets because they do not match our strict "Port 80 TCP" rule!*

**Test 3: Can the client reach the public internet? (Should FAIL ❌)**
If an attacker compromises a pod, their first move is often to download malware from the internet or establish a Command & Control (C2) connection.
```bash
kubectl exec client -- curl -s --connect-timeout 3 http://google.com
```
*Result: Connection timed out. Our `default-deny-all` blocks external Egress traffic, keeping the compromised pod quarantined.*

---

### 🌟 Extension Lab: Advanced Calico Policies (Explicit Deny)
Native Kubernetes `NetworkPolicy` objects only support **Allow** rules (they are purely additive). But what if you want to allow all outbound traffic *except* for one specific malicious IP address? 

Because Calico is our CNI, we can use Calico's own Custom Resource Definitions (CRDs) to create explicit **Deny** rules and establish rule **Ordering**.

First, let's delete our strict native policies so the client can reach the internet again:
```bash
kubectl delete -f 2-strict-security.yaml
```

Create a file named `3-calico-deny.yaml`. Notice the `apiVersion` is `crd.projectcalico.org/v1` instead of `networking.k8s.io/v1`:

```yaml
apiVersion: crd.projectcalico.org/v1
kind: NetworkPolicy
metadata:
  name: deny-google-dns
spec:
  order: 10 # Lower numbers are evaluated first
  selector: app == 'client' # Calico uses a slightly different selector syntax
  types:
  - Egress
  egress:
  - action: Deny # Native K8s CANNOT do this!
    destination:
      nets:
      - "8.8.8.8/32"
  - action: Allow # Allow everything else
```

Apply the Calico policy:
```bash
kubectl apply -f 3-calico-deny.yaml
```

**Test the Explicit Deny:**
Try to ping Google's Primary DNS (8.8.8.8). This should **FAIL ❌** because of our explicit Deny rule:
```bash
kubectl exec client -- ping -c 3 -W 1 8.8.8.8
```

Now try to ping Cloudflare's DNS (1.1.1.1). This should **SUCCEED ✅** because of our Catch-All Allow rule at the bottom of the policy:
```bash
kubectl exec client -- ping -c 3 -W 1 1.1.1.1
```

---

### ⚠️ Common Mistakes & Troubleshooting
* **Forgetting the DNS Rule:** 🤕 The most common mistake when writing a Default Deny Egress policy is forgetting to allow Port 53 (TCP/UDP). If pods cannot reach CoreDNS, they cannot resolve service names like `http://web`. This makes it look like your internal port rules are broken when it's actually just a DNS failure.
* **Overlapping Policies:** 🥞 Kubernetes NetworkPolicies are additive. If you have an old policy that allows `Ingress: {}` (all traffic), adding a new policy that restricts traffic to Port 80 will **not** work. The most permissive rule always wins. Always delete broad policies before testing strict ones.
* **Label Typos:** 🐛 `podSelector` relies entirely on labels. If your pod has the label `app=web-server` but your NetworkPolicy looks for `app=web`, the policy will not match the pod, leaving it completely unprotected or totally blocked.
* **Context Confusion:** 🤷‍♂️ If you get "NotFound" errors, verify you are working in the correct namespace using `kubectl config current-context`.

---

### 🧹 Lab Cleanup
To clean up your environment, switch your context back to the default, delete the `lab` namespace, and remove your custom context:

```bash
# Switch back to the default context
kubectl config use-context default

# Delete the lab namespace
kubectl delete namespace lab

# Delete the custom context you created
kubectl config delete-context lab-context
```

---

### 🎓 Lab Recap & Review
Congratulations on completing the lab! Let's quickly review the core concepts you just put into practice:

* **Contexts & Labels are your friends:** 🏷️ You learned how to use `kubectl config` to streamline your workflow and how enterprise environments use **Labels** (like `environment=student-lab`) instead of hardcoded names to manage security policies dynamically.
* **The "Flat Network" is dangerous:** 🔓 Out of the box, Kubernetes allows all pods to communicate. We fixed this by introducing the concept of **Namespace Isolation**.
* **Zero Trust starts with Default Deny:** 🧱 Because Kubernetes NetworkPolicies are *additive* (they only grant permission), true security requires laying down a "Default Deny" policy first, and then poking precise holes for required traffic.
* **Don't forget DNS!** 🌐 Blocking Egress traffic breaks DNS resolution. You must explicitly allow UDP/TCP Port 53 to your cluster's CoreDNS, or your pods won't be able to find each other by name.
* **Granular Application Security:** 🚪 We moved beyond basic isolation to application-specific rules, ensuring our Web server only accepted traffic on its operational port (TCP 80), blocking malicious scanning attempts like ICMP (Ping).
* **Native K8s vs. Calico CRDs:** ⚖️ You experienced the limitations of native Kubernetes policies (Allow only) and how leveraging Calico's Custom Resource Definitions unlocks advanced firewall capabilities like explicit **Deny** rules and policy **Ordering**.

***

Does this final version look ready to publish to your GitHub repository, or are there any other sections you'd like to refine?
