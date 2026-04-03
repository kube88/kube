
***

# ⚖️ Kubernetes Resource Management Lab: Quotas & Limits

## 🎯 Lab Objective
In this lab, you will explore how Kubernetes manages compute resources (CPU and Memory) to prevent the "Noisy Neighbor" problem. You will learn how to:
1. Establish a **LimitRange** to set default resource allocations for individual pods.
2. Implement a **ResourceQuota** to set a hard ceiling on an entire namespace.
3. Observe how Kubernetes behaves when a pod attempts to hoard resources.
4. Observe how Kubernetes behaves when a namespace runs out of capacity.

---

### 🧠 Core Concepts: What are we actually doing?
Before jumping into the command line, let's understand the problem we are solving:
* **The Noisy Neighbor Problem:** 📢 By default, a pod in Kubernetes can consume as much CPU and memory as the underlying physical node has available. If one developer writes a memory leak, their pod could crash the entire server, taking down everyone else's applications with it!
* **Requests vs. Limits:** 📏 
  * **Request:** The guaranteed minimum amount of resources a pod needs to run. Kubernetes uses this for scheduling (finding a node with enough room).
  * **Limit:** The absolute maximum amount of resources a pod is allowed to use. If it tries to use more memory than its limit, Kubernetes will kill it (OOMKilled). If it tries to use more CPU, it gets throttled (slowed down).
* **LimitRange:** 🎯 A policy that automatically injects default Requests and Limits into any new pod that forgets to specify them. It also sets minimum and maximum sizes for a single pod.
* **ResourceQuota:** 🛑 A policy applied to the *entire namespace*. It says, "The combined total of all pods in this namespace cannot exceed X amount of CPU, Y amount of Memory, and Z total Pods."

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
cd "FRI 05 - Quotas and Limits"
```

💡 **What is happening here?** Kubernetes uses a configuration file (often called `kubeconfig`) to know which cluster to talk to and what credentials to use. By copying the shared config file to your home directory, you are giving your `kubectl` command the "keys" to the cluster. Finally, we navigate to the correct folder so that our YAML files are easy to find.

---

### 🏗️ Step 2: Setting Up Your Workspace
Let's create our secure sandbox namespace and switch our context over to it.

```bash
kubectl create namespace lab
kubectl config set-context lab-context --current --namespace=lab
kubectl config use-context lab-context
```

💡 **What is happening here?** We are creating a namespace called `lab` and setting our default context to it so we don't have to type `-n lab` at the end of every command.

---

### 📏 Step 3: Setting Default Boundaries (LimitRange)
Before we put a hard cap on the namespace, we need to set up defaults. If you apply a Quota to a namespace, Kubernetes *forces* every pod to declare its resource limits. Instead of making developers type this out every time, we will use a `LimitRange` to automatically assign sizes to pods.

Create a file named `1-limit-range.yaml`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-resource-limits
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: 200m     # 200 millicores (20% of 1 CPU)
      memory: 256Mi # 256 Megabytes
    default:        # This is the Limit!
      cpu: 500m     # 500 millicores (50% of 1 CPU)
      memory: 512Mi # 512 Megabytes
```

Apply the LimitRange:
```bash
kubectl apply -f 1-limit-range.yaml
```

💡 **What is happening here?** We are telling Kubernetes: "If a developer deploys a container in the `lab` namespace and doesn't specify how big it should be, automatically *request* `256Mi` of RAM to start, but *limit* it so it can never exceed `512Mi` of RAM."

---

### 🛑 Step 4: Setting the Namespace Ceiling (ResourceQuota)
Now we will set the absolute maximum capacity for the entire `lab` namespace. We will pretend we are billing a customer for a tiny slice of our cluster.

Create a file named `2-quota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lab-hard-quota
spec:
  hard:
    pods: "3"             # Maximum of 3 total pods allowed in this namespace
    requests.cpu: "1"     # Total requested CPU across all pods cannot exceed 1 Core
    requests.memory: 1Gi  # Total requested RAM cannot exceed 1 Gigabyte
    limits.cpu: "2"       # Total CPU limits cannot exceed 2 Cores
    limits.memory: 2Gi    # Total RAM limits cannot exceed 2 Gigabytes
```

Apply the ResourceQuota:
```bash
kubectl apply -f 2-quota.yaml
```

Check the status of your Quota:
```bash
kubectl describe quota lab-hard-quota
```

💡 **What is happening here?** When you describe the quota, you will see a table showing `Used` vs `Hard`. Right now, `Used` is at zero. This quota guarantees that no matter what happens, this namespace can never consume more than 2 Cores of CPU, 2 Gigabytes of RAM, or run more than 3 Pods.

---

### 🟢 Step 5: Test 1 - The Good Citizen Pod
Let's deploy a standard web server without specifying any resources to see the LimitRange in action.

```bash
kubectl run web-server --image=nginx
```

Verify it is running, and then inspect it:
```bash
kubectl describe pod web-server
```

💡 **What is happening here?** Look closely at the `Containers` section in the output. Even though we didn't type anything about CPU or Memory when we ran the pod, you will see that it has `Requests` of `256Mi` and `Limits` of `512Mi`. The **LimitRange** successfully intercepted the pod creation and injected our defaults!

Let's check our Quota again:
```bash
kubectl describe quota lab-hard-quota
```
*Notice how the `Used` column now shows 1 pod, 200m CPU requests, and 256Mi Memory requests.*

---

### 🔴 Step 6: Test 2 - The Greedy Pod
Now let's try to deploy a pod that explicitly asks for more resources than our namespace quota allows. We will ask for 4 Gigabytes of RAM (our namespace only allows 2 Gigabytes max).

Create a file named `3-greedy-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: greedy-database
spec:
  containers:
  - name: db
    image: redis
    resources:
      requests:
        memory: "4Gi"
      limits:
        memory: "4Gi"
```

Try to apply it:
```bash
kubectl apply -f 3-greedy-pod.yaml
```

❌ **Expected Result:** You will get a nasty error message from the API server!
`Error from server (Forbidden): error when creating "3-greedy-pod.yaml": pods "greedy-database" is forbidden: exceeded quota: lab-hard-quota...`

💡 **What is happening here?** The Kubernetes API server rejected the pod before it was even scheduled. Our **ResourceQuota** protected the cluster from an oversized workload that would have starved other tenants.

---

### 🔴 Step 7: Test 3 - Too Many Pods
Our quota also limits us to a total of **3 Pods**. We already have 1 running (`web-server`). Let's try to deploy 3 more using a Deployment.

Create a file named `4-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-fleet
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
```

Apply the deployment:
```bash
kubectl apply -f 4-deployment.yaml
```

Now, let's look at what actually happened:
```bash
kubectl get pods
kubectl describe replicaset web-fleet
```

❌ **Expected Result:** You will see your original `web-server` pod, and exactly **two** `web-fleet` pods. The third one is missing! If you look at the ReplicaSet events, you will see a warning: `created pod: Exceeded quota: lab-hard-quota, requested: pods=1, used: pods=3, limited: pods=3`.

💡 **What is happening here?** A Deployment creates a ReplicaSet, which tries to spawn the pods. It successfully created the first two (bringing our total namespace pod count to exactly 3). When it tried to create the final pod, the Quota blocked it. 

---

### ⚠️ Common Mistakes & Troubleshooting
* **The "No Default" Deadlock:** 🚨 If you create a `ResourceQuota` for a namespace, but you *forget* to create a `LimitRange`, any pod deployed without explicit resources will instantly fail to create. Kubernetes says: "You have a quota, so you MUST tell me how big you are. Since you didn't, I'm rejecting you." Always pair Quotas with LimitRanges!
* **CPU Measurements:** In Kubernetes, `1` CPU equals 1 full core/vCPU. You can express fractions as `0.5` or use millicores like `500m`. `1000m` is exactly equal to `1`.
* **Memory Measurements:** `Mi` stands for Mebibytes (Base 2), which is how computers actually read memory, while `M` stands for Megabytes (Base 10). Always use `Mi` and `Gi` in Kubernetes to ensure exact byte calculations.

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

* **Good Fences Make Good Neighbors:** 🏡 You learned that in a multi-tenant cluster, setting strict boundaries is the only way to prevent one team's bad code from crashing another team's application.
* **LimitRange (The Defaults):** 🎯 You learned how to automate resource assignments. Instead of trusting developers to remember to set `requests` and `limits`, you intercepted their pods and injected sensible defaults automatically.
* **ResourceQuota (The Ceiling):** 🛑 You learned how to turn a logical Namespace into a strictly defined virtual data center, capping total compute resources and total object counts.
* **API Rejection:** 🛡️ You saw firsthand how Kubernetes acts as a bouncer, outright rejecting configurations (`greedy-database`) that violate the established rules before they can impact the cluster.
