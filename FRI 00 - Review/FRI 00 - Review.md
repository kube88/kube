kubectl config set-context ${NS}-context --namespace=$NS --username=admin --cluster=vke-1057e5ab-738e-4d02-af04-35add3ea7170
***

# 🚀 Kubernetes Core Lab: Mastering Deployments

## 🎯 Lab Objective
In this lab, you will explore the most common way to run applications in Kubernetes: The Deployment. Because you are working on a **shared cluster**, you will first secure your own isolated workspace. You will learn how to:
1. Prepare an isolated workspace using variables.
2. Deploy an application declaratively using a YAML manifest.
3. Scale the application up and down.
4. Perform a zero-downtime Rolling Update to a new application version.
5. Perform a Rollback to undo a bad update.

---

### 🧠 Core Concepts: What is a Deployment?
Before jumping in, let's establish what a Deployment actually does:
* **Pods are fragile:** 🥚 A Pod is the smallest unit in Kubernetes, but Pods are mortal. If a node dies, the Pod dies with it and does not come back.
* **The ReplicaSet:** 👯 To fix this, Kubernetes uses ReplicaSets to ensure a specific number of Pods are always running. If one dies, the ReplicaSet creates a new one.
* **The Deployment:** 🏗️ A Deployment is a higher-level manager that controls ReplicaSets. It allows you to easily update your application to a new version, or roll back to an old version, without any downtime. **You should almost always use Deployments, not bare Pods!**

---

## 📋 Prerequisites
* Access to the command line CLI connected to your Kubernetes cluster.
* Your assigned Student ID (e.g., `s1`, `s2`, ... `s12`).

---

### 🛠️ Step 0: Initial Configuration
Before we can interact with the Kubernetes cluster, you must ensure your environment is configured correctly. Run the following commands to set up your Kubernetes configuration and download the lab files:

```bash
# 1. Create the .kube directory
mkdir -p ~/.kube

# 2. Copy the master config file to your local directory
cp /home/config ~/.kube/config

# 3. Secure the config file permissions
chmod 600 ~/.kube/config

# 4. Clone the lab repository and enter the lab directory
cd ~
git clone https://github.com/kube88/kube.git
cd kube
cd "FRI 00 - Review"
```

💡 **What is happening here?** 
* **Kubeconfig:** Kubernetes uses a configuration file to know which cluster to talk to and what credentials to use. By copying the shared config file to your home directory, you are giving your `kubectl` command the "keys" to the cluster.
* **Git Clone:** You are downloading all the YAML files and instructions from GitHub so you have everything you need locally on your server.

---

### 🔌 Step 1: Setup Your Workspace
Because you are sharing this cluster with other students, you must work inside your assigned namespace. We will use a variable (`$NS`) to make copying and pasting commands easier.

Execute the following commands (be sure to change `s1` to your actual assigned ID!):

```bash
# 1. Set your Student ID as a variable
export NS=s1
```
```bash
# 2. Create your personal namespace
kubectl create namespace $NS

# 3. Create and switch to a custom context locked to your namespace
kubectl config set-context ${NS}-context-current --namespace=$NS
kubectl config use-context ${NS}-context-current
```

💡 **What is happening here?** By setting the `$NS` variable, your terminal remembers your student ID. By creating and using `${NS}-context`, you are telling Kubernetes to automatically route all future commands directly into your specific namespace. This ensures you don't accidentally delete another student's work!

---

### 📝 Step 2: Create a Deployment (Declarative Method)
While you can create Deployments via single-line commands, the industry standard is to use declarative YAML files so your infrastructure can be tracked in version control (like Git). 

*(Note: You do not need to put your namespace in this YAML file, because your Context is already handling it automatically!)*

Create a file named `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-web
  labels:
    app: nginx
spec:
  replicas: 3 # We are asking K8s for exactly 3 copies of our app
  selector:
    matchLabels:
      app: nginx # The deployment manages pods with this label
  template:
    metadata:
      labels:
        app: nginx # The label applied to the pods
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2 # We are starting with an older version of Nginx
        ports:
        - containerPort: 80
```

Apply the deployment to your cluster:
```bash
kubectl apply -f deployment.yaml
```

💡 **What is happening here?** You just handed a blueprint to Kubernetes. The Deployment controller reads the blueprint and creates a ReplicaSet. That ReplicaSet then spins up exactly `3` Pods inside your `$NS` namespace running the `nginx:1.14.2` image. 

---

### 🔎 Step 3: Verify the Deployment
Let's look at the resources Kubernetes created just for you.

View your Deployments, ReplicaSets, and Pods:
```bash
kubectl get deployments,replicasets,pods
```

* **Deployments:** You should see `frontend-web` with `3/3` ready.
* **ReplicaSets:** You will see a ReplicaSet with a random string attached to the name (e.g., `frontend-web-5d59d67564`).
* **Pods:** You will see three individual pods running.

---

### 📈 Step 4: Scaling the Application
Imagine your web application suddenly gets a spike in traffic. We need to scale up our resources to handle the load.

Scale the deployment up to 5 replicas:
```bash
kubectl scale deployment frontend-web --replicas=5
```

Watch the pods spin up in real-time (press `Ctrl+C` to exit the watch screen):
```bash
kubectl get pods -w
```

💡 **What is happening here?** We used an imperative command to change the desired state from 3 to 5. The ReplicaSet immediately noticed the discrepancy between the *desired state* (5) and the *actual state* (3) and spun up 2 more pods to fulfill the request.

---

### 🔄 Step 5: The Rolling Update
The development team just released a new version of the frontend. We need to update our Nginx image from `1.14.2` to `1.16.1`. Deployments handle this gracefully by spinning up new pods before destroying the old ones, ensuring zero downtime.

Update the image version:
```bash
kubectl set image deployment/frontend-web nginx=nginx:1.16.1
```

Check the rollout status to ensure it completes successfully:
```bash
kubectl rollout status deployment/frontend-web
```

Now, check your ReplicaSets again:
```bash
kubectl get replicasets
```

💡 **What is happening here?** You will now see **two** ReplicaSets. The Deployment created a new ReplicaSet for the new image version, scaled it up, and slowly scaled the old ReplicaSet down to `0`. The old ReplicaSet is kept around just in case we need to go back!

---

### ⏪ Step 6: The Rollback
Uh oh! The new `1.16.1` version has a critical bug, and the users are complaining. We need to instantly revert to the old version.

Check your rollout history to see previous versions:
```bash
kubectl rollout history deployment/frontend-web
```

Undo the latest rollout to revert to the previous working state:
```bash
kubectl rollout undo deployment/frontend-web
```

Verify the rollback was successful by checking the image version currently running:
```bash
kubectl describe deployment frontend-web | grep Image:
```

💡 **What is happening here?** The Deployment simply scaled the *old* ReplicaSet back up to 5, and scaled the *new* (broken) ReplicaSet down to 0. You successfully mitigated a production outage in seconds!

---

### 🧹 Lab Cleanup
To clean up your environment, delete your Deployment, return to the default context, and remove your personal namespace.

```bash
# Delete the deployment
kubectl delete -f deployment.yaml

# Switch back to the default context
kubectl config use-context default

# Delete your student namespace
kubectl delete namespace $NS

# Delete the custom context you created
kubectl config delete-context ${NS}-context
```

---

### 🎓 Lab Recap & Review
Congratulations on completing the lab! Let's review the core concepts you just put into practice:

* **Contexts Keep Shared Clusters Safe:** 🤝 By using a variable (`$NS`) and locking your context to a specific namespace, you can safely write and deploy YAML files in a shared cluster without hardcoding namespace names or overwriting a classmate's work.
* **Deployments > Pods:** 🏗️ You learned that you rarely create Pods directly. You create Deployments, which manage ReplicaSets, which in turn ensure your Pods stay alive and healthy.
* **Imperative Scaling is Fast:** 📈 You can rapidly adjust to web traffic spikes using a simple `kubectl scale` command, letting Kubernetes handle the heavy lifting of scheduling the new containers.
* **Zero-Downtime Rollouts:** 🔄 By changing the image version in a Deployment, Kubernetes automatically performs a Rolling Update, standing up the new application before tearing down the old one.
* **Instant Rollbacks:** ⏪ Because Deployments keep older ReplicaSets scaled down to `0`, reverting a bad update is as simple as running an `undo` command to bring the previous version back online instantly.
