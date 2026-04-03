

⛵ **Kubernetes Management Lab: Deploying the Dashboard & Harbor with Helm**

🎯 **Lab Objective**
In this lab, you will use Helm to deploy the official Kubernetes Web UI (Dashboard) and an enterprise container registry. You will learn how to:
* Perform initial cluster configuration.
* Add official project repositories to Helm.
* Deploy a complex, multi-resource application (The Dashboard).
* **Security & Access:** Create an Admin user and generate an authentication token.
* **Visual Verification:** Use port-forwarding to log into the UI from your browser.
* **Extension:** Quickly deploy the Harbor Container Registry using inline Helm variables.

🧠 **Core Concepts: What are we actually doing?**
* **The Kubernetes Dashboard:** 🖥️ This is a web-based UI for your cluster. It allows you to see your workloads, logs, and resource usage without typing commands in a terminal.
* **Helm for Complexity:** 📦 Installing the Dashboard manually requires creating about 5-7 different YAML files. Helm simplifies this into a single "Chart" that handles the logic for you.
* **RBAC (Role-Based Access Control):** 🔐 By default, the Dashboard is "locked." To see anything, we must create a **ServiceAccount** (a user for the dashboard) and give it **ClusterAdmin** permissions.
* **Authentication Tokens:** 🔑 Modern Kubernetes clusters use temporary bearer tokens for security. We will generate one of these to "log in" to the web interface.

📋 **Prerequisites**
* Access to the command line in your dedicated HOL instance.

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

🏗️ **Step 2: Setting Up Your Workspace**
We will create a dedicated namespace for the dashboard to keep it isolated from other cluster services.

```bash
kubectl create namespace kubernetes-dashboard
kubectl config set-context dash-context --current --namespace=kubernetes-dashboard
kubectl config use-context dash-context
```

🔎 **Step 3: Adding the Dashboard Repository**
The Dashboard chart is maintained by the official Kubernetes community.

```bash
helm repo add kubernetes-dashboard https://kubernetes-retired.github.io/dashboard/
helm repo update
```

📦 **Step 4: Installing the Dashboard**
We will now install the dashboard. We'll give our release the name `my-dash`.

```bash
helm install my-dash kubernetes-dashboard/kubernetes-dashboard
```
💡 **What is happening here?** Helm is currently creating a Deployment (the app), two Services (for networking), and several RBAC roles. You can see the progress by running `kubectl get pods`.

🔐 **Step 5: Creating the Admin User (The Login Key)**
If you tried to open the dashboard now, you wouldn't be able to log in. We need to create an admin user and a "token" to act as our password.

**1. Create a ServiceAccount:**
```bash
kubectl create serviceaccount admin-user
```

**2. Give that user "Cluster Admin" permissions:**
```bash
kubectl create clusterrolebinding admin-user-binding --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:admin-user
```

**3. Generate your Login Token:**
```bash
kubectl create token admin-user
```
⚠️ **Copy this token!** You will need this long string of characters to log into the web browser in the next step.

🌐 **Step 6: Accessing the Dashboard**
Because the dashboard is inside a private network, we will use a tunnel to reach it from our local browser.

Run the port-forward command:
```bash
kubectl port-forward svc/my-dash-kong-proxy 8443:443
```
*(Leave this terminal open!)*

🖥️ **Open your browser and log in:**
1. Navigate to: `https://localhost:8443`
2. **Note:** Your browser will show a "Privacy/Security Warning" because the dashboard uses a self-signed certificate. Click **Advanced** and then **Proceed to localhost**.
3. Select the **Token** login option.
4. Paste the long token you generated in Step 5.
✅ **Success!** You are now viewing your Kubernetes cluster in a beautiful web interface. Go back to your terminal and press **Ctrl + C** to close the tunnel.

🛠️ **Step 7: Customizing with Values (Scaling)**
Let's use Helm's "Upgrade" feature to change a setting. By default, the dashboard runs 1 pod. Let's scale it to 2 for high availability.

Create `dash-values.yaml`:
```yaml
web:
  replicaCount: 2
```

Apply the upgrade:
```bash
helm upgrade my-dash kubernetes-dashboard/kubernetes-dashboard -f dash-values.yaml
```

💡 **What is happening here?** Helm updates the existing deployment without you having to delete and recreate the user accounts or services we set up earlier.

🌟 **Extension Lab: Installing Harbor Registry**
What if you want to deploy a massive enterprise application without writing a `values.yaml` file? Let's quickly deploy **Harbor**, a popular open-source container image registry, using inline Helm variables.

```bash
# 1. Add the Harbor repository
helm repo add harbor https://helm.goharbor.io
helm repo update

# 2. Install Harbor with inline configurations
helm install my-harbor harbor/harbor \
  --set expose.type=ClusterIP \
  --set expose.tls.enabled=false \
  --set persistence.enabled=false
```

💡 **What is happening here?** Harbor is a massive application with databases, caching, and storage requirements. By using the `--set` flag, we told Helm to override specific default settings on the fly. We turned off TLS and Persistent Storage just to get it running quickly and lightweight for this lab environment. You can run `kubectl get pods` to watch the registry spin up!

⚠️ **Common Mistakes & Troubleshooting**
* **The Token Timeout:** ⏳ Tokens generated by `kubectl create token` are temporary. If you come back tomorrow and can't log in, you simply need to run the `create token` command again.
* **HTTPS vs HTTP:** 🔒 The dashboard *requires* HTTPS. If you try to go to `http://localhost:8443` (without the 's'), the connection will fail.
* **RBAC Errors:** 🛑 If you log in but see "Forbidden" errors everywhere, it means your `ClusterRoleBinding` from Step 5 failed or was attached to the wrong namespace.

🧹 **Lab Cleanup**
```bash
# Delete the releases
helm uninstall my-dash
helm uninstall my-harbor

# Delete the admin user and binding
kubectl delete clusterrolebinding admin-user-binding
kubectl delete serviceaccount admin-user

# Cleanup namespace
kubectl config use-context default
kubectl delete namespace kubernetes-dashboard
```

🎓 **Lab Recap & Review**
* **Helm managed the complexity:** ⛵ You deployed secure, multi-tier applications with just a few commands.
* **RBAC is required for UIs:** 🔐 You learned that "installing" an app isn't enough; you must also manage the permissions (ServiceAccounts) so users can actually see data.
* **The Port-Forwarding Tunnel:** 🌐 You used a tunnel to bridge the gap between your local browser and the secure cluster environment.
* **Inline Variables:** ⚙️ You used the `--set` flag to quickly customize the Harbor installation without needing to draft a full YAML file.
