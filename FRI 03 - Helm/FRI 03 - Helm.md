
⛵ **Kubernetes Management Lab: Deploying the Dashboard & Harbor with Helm**

## 🎯 Lab Objective
In this lab, you will use Helm to deploy the official Kubernetes Web UI (Dashboard) and an enterprise container registry. You will learn how to:
* Perform initial cluster configuration.
* Establish a secure workspace using variables.
* Add official project repositories to Helm.
* Deploy a complex, multi-resource application (The Dashboard).
* **Security & Access:** Create an Admin user and generate an authentication token.
* **Visual Verification:** Use port-forwarding to log into the UI from your browser.
* **Extension:** Quickly deploy the Harbor Container Registry using inline Helm variables.

---

### 🧠 Core Concepts: What are we actually doing?
* **The Kubernetes Dashboard:** 🖥️ This is a web-based UI for your cluster. It allows you to see your workloads, logs, and resource usage without typing commands in a terminal.
* **Helm for Complexity:** 📦 Installing the Dashboard manually requires creating about 5-7 different YAML files. Helm simplifies this into a single "Chart" that handles the logic for you.
* **RBAC (Role-Based Access Control):** 🔐 By default, the Dashboard is "locked." To see anything, we must create a **ServiceAccount** (a user for the dashboard) and give it **ClusterAdmin** permissions.
* **Authentication Tokens:** 🔑 Modern Kubernetes clusters use temporary bearer tokens for security. We will generate one of these to "log in" to the web interface.

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
cd "FRI 03 - Helm"
```

💡 **What is happening here?** Kubernetes uses a configuration file (often called `kubeconfig`) to know which cluster to talk to and what credentials to use. By copying the shared config file to your home directory, you are giving your `kubectl` command the "keys" to the cluster. Finally, we navigate to the correct folder so that our YAML files are easy to find.

---

### 🏗️ Step 2: Setting Up Your Workspace
Because you are sharing this cluster with other students, you must work inside your assigned namespace. We will use a variable (`$NS`) to make copying and pasting commands easier.

Execute the following command (be sure to change `s1` to your actual assigned ID!):

```bash
# 1. Set your Student ID as a variable
export NS=s1
```
```bash
# 2. Create your personal namespace
kubectl create namespace $NS
```

💡 **What is happening here?** By setting the `$NS` variable, your terminal remembers your student ID. We use this variable in all future commands to ensure you are only interacting with your own resources!

---

### 🔎 Step 3: Adding the Dashboard Repository
The Dashboard chart is maintained by the official Kubernetes community.

```bash
helm repo add kubernetes-dashboard https://kubernetes-retired.github.io/dashboard/
helm repo update
```

---

### 📦 Step 4: Installing the Dashboard
We will now install the dashboard. We'll give our release the name `my-dash`.

```bash
helm install my-dash kubernetes-dashboard/kubernetes-dashboard -n $NS
```
💡 **What is happening here?** Helm is currently creating a Deployment (the app), two Services (for networking), and several RBAC roles inside your personal namespace. You can see the progress by running `kubectl get pods -n $NS`.

---

### 🔐 Step 5: Creating the Admin User (The Login Key)
If you tried to open the dashboard now, you wouldn't be able to log in. We need to create an admin user and a "token" to act as our password.

**1. Create a ServiceAccount:**
```bash
kubectl create serviceaccount admin-user -n $NS
```

**2. Give that user "Cluster Admin" permissions:**
```bash
kubectl create clusterrolebinding admin-user-binding --clusterrole=cluster-admin --serviceaccount=${NS}:admin-user
```

**3. Generate your Login Token:**
```bash
kubectl create token admin-user -n $NS
```
⚠️ **Copy this token!** You will need this long string of characters to log into the web browser in the next step.

---

### 🌐 Step 6: Accessing the Dashboard
Because the dashboard is inside a private network, we will use a tunnel to reach it from our local browser.

Run the port-forward command:
```bash
kubectl port-forward svc/my-dash-kong-proxy 8443:443 -n $NS
```
*(Leave this terminal open!)*

🖥️ **Open your browser and log in:**
1. Navigate to: `https://localhost:8443`
2. **Note:** Your browser will show a "Privacy/Security Warning" because the dashboard uses a self-signed certificate. Click **Advanced** and then **Proceed to localhost**.
3. Select the **Token** login option.
4. Paste the long token you generated in Step 5.
✅ **Success!** You are now viewing your Kubernetes cluster in a beautiful web interface. Go back to your terminal and press **Ctrl + C** to close the tunnel.

---

### 🛠️ Step 7: Customizing with Values (Scaling)
Let's use Helm's "Upgrade" feature to change a setting. By default, the dashboard runs 1 pod. Let's scale it to 2 for high availability.

Create `dash-values.yaml`:
```yaml
web:
  replicaCount: 2
```

Apply the upgrade:
```bash
helm upgrade my-dash kubernetes-dashboard/kubernetes-dashboard -f dash-values.yaml -n $NS
```

💡 **What is happening here?** Helm updates the existing deployment without you having to delete and recreate the user accounts or services we set up earlier.

---

### 🌟 Extension Lab: Installing Harbor Registry
What if you want to deploy a massive enterprise application without writing a `values.yaml` file? Let's quickly deploy **Harbor**, a popular open-source container image registry, using inline Helm variables.

```bash
# 1. Add the Harbor repository
helm repo add harbor https://helm.goharbor.io
helm repo update

# 2. Install Harbor with inline configurations
helm install my-harbor harbor/harbor -n $NS \
  --set expose.type=ClusterIP \
  --set expose.tls.enabled=false \
  --set persistence.enabled=false
```

💡 **What is happening here?** Harbor is a massive application with databases, caching, and storage requirements. By using the `--set` flag, we told Helm to override specific default settings on the fly. You can run `kubectl get pods -n $NS` to watch the registry spin up!

---

### 🧹 Lab Cleanup
To clean up your environment, delete your resources and remove your personal namespace.

```bash
# Delete the releases
helm uninstall my-dash -n $NS
helm uninstall my-harbor -n $NS

# Delete the admin user binding
kubectl delete clusterrolebinding admin-user-binding

# Delete your student namespace
kubectl delete namespace $NS
```

---

### 🎓 Lab Recap & Review
Congratulations on completing the lab! Let's quickly review the core concepts you just put into practice:

* **Helm managed the complexity:** ⛵ You deployed secure, multi-tier applications with just a few commands.
* **RBAC is required for UIs:** 🔐 You learned that "installing" an app isn't enough; you must also manage the permissions (ServiceAccounts) so users can actually see data.
* **The Port-Forwarding Tunnel:** 🌐 You used a tunnel to bridge the gap between your local browser and the secure cluster environment.
* **Inline Variables:** ⚙️ You used the `--set` flag to quickly customize the Harbor installation without needing to draft a full YAML file.
