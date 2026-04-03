
***

# 🗺️ Kubernetes Configuration Lab: ConfigMaps & Secrets

## 🎯 Lab Objective
In this lab, you will learn how Kubernetes manages application configuration and sensitive data. You will explore how to decouple configuration from your container images by learning how to:
1. Create ConfigMaps for standard configuration data.
2. Create Secrets for sensitive data (like passwords).
3. Inject configurations into Pods as **Environment Variables**.
4. Mount configurations into Pods as **File Volumes**.
5. Decode Base64 Kubernetes Secrets.

---

### 🧠 Core Concepts: What are we actually doing?
Before jumping into the command line, let's understand the problem we are solving:
* **The Hardcoding Problem:** If you hardcode a database password or an API URL directly into your application's source code, you have to rebuild the entire container image every time that configuration changes. This is slow and a massive security risk.
* **ConfigMaps:** 📄 These are Kubernetes objects used to store non-confidential data in key-value pairs. Think of them as configuration files (like a `.ini` or `.json` settings file) that live inside the cluster, separate from the pod.
* **Secrets:** 🤫 These are similar to ConfigMaps but are specifically designed to hold sensitive information, such as passwords, OAuth tokens, and SSH keys.
* **The Magic of Decoupling:** By using ConfigMaps and Secrets, your container image remains generic. You can take the *exact same* image and run it in Dev, Test, and Prod simply by swapping out the ConfigMap/Secret it connects to!

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
cd "FRI 04 - Config Maps and Secrets"
```

💡 **What is happening here?** Kubernetes uses a configuration file (often called `kubeconfig`) to know which cluster to talk to and what credentials to use. By copying the shared config file to your home directory, you are giving your `kubectl` command the "keys" to the cluster. Finally, we navigate to the correct folder so that our YAML files are easy to find.

---

### 🏗️ Step 2: Setting Up Your Workspace
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
kubectl config set-context ${NS}-context --namespace=$NS
kubectl config use-context ${NS}-context
```

💡 **What is happening here?** By setting the `$NS` variable, your terminal remembers your student ID. By creating and using `${NS}-context`, you are telling Kubernetes to automatically route all future commands directly into your specific namespace. This ensures you don't accidentally delete another student's work!

---

### 📄 Step 3: Creating a ConfigMap
Let's pretend we have a web application, and we want to control its UI theme and default language without changing the code.

Create a ConfigMap using the `--from-literal` flag:

```bash
kubectl create configmap ui-config --from-literal=theme=dark-mode --from-literal=language=english
```

Verify it was created:
```bash
kubectl get configmaps
kubectl describe configmap ui-config
```

💡 **What is happening here?** We created a ConfigMap named `ui-config`. Inside it, we stored two distinct key-value pairs: `theme` and `language`. The `kubectl describe` command allows you to view the raw data stored inside.

---

### 🤫 Step 4: Creating a Secret
Now let's pretend our application needs to connect to a backend database. We cannot put the password in a ConfigMap because anyone with access to the cluster could read it easily.

Create a Kubernetes Secret:

```bash
kubectl create secret generic db-credentials --from-literal=username=admin --from-literal=password=SuperSecret123!
```

Verify it was created, but try to read the data:
```bash
kubectl get secret
kubectl describe secret db-credentials
```

💡 **What is happening here?** Notice the difference? When you describe the Secret, Kubernetes **hides the values**. It tells you the keys (`username` and `password`) exist and how many bytes they are, but it protects the actual text from prying eyes on the command line.

---

### 💉 Step 5: Injecting as Environment Variables
We have our configuration data floating in the cluster. Now we need to give it to an application. The most common way to do this is by injecting the data as Environment Variables.

Create a file named `1-env-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-test-pod
spec:
  containers:
  - name: my-app
    image: alpine
    command: [ "sleep", "3600" ] # Keeps the pod running so we can look inside
    env:
    # 1. Pulling a specific key from a ConfigMap
    - name: APP_THEME
      valueFrom:
        configMapKeyRef:
          name: ui-config
          key: theme
          
    # 2. Pulling a specific key from a Secret
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

Apply the Pod:
```bash
kubectl apply -f 1-env-pod.yaml
```

💡 **What is happening here?** Inside the `env` section, we are creating standard Linux environment variables (`APP_THEME` and `DB_PASSWORD`). However, instead of hardcoding their values, we are using `valueFrom` to dynamically grab the data from the ConfigMap and Secret we created earlier.

---

### 🧪 Step 6: Verifying Environment Variables
Let's jump *inside* the running pod and see if our application can see the configurations.

Execute the `env` command inside the pod and filter for our specific variables:

```bash
kubectl exec env-test-pod -- env | grep -E 'APP_THEME|DB_PASSWORD'
```

✅ *Result: You should see `APP_THEME=dark-mode` and `DB_PASSWORD=SuperSecret123!` printed to your screen.*

---

### 📂 Step 7: Mounting as Volumes (Files)
Sometimes, applications don't read environment variables. They expect to find a configuration *file* (like `config.json`) sitting on the hard drive. Kubernetes can take a ConfigMap or Secret and mount it as a physical file inside the container!

Create a file named `2-volume-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vol-test-pod
spec:
  containers:
  - name: my-app
    image: alpine
    command: [ "sleep", "3600" ]
    volumeMounts:
    # 1. Where to place the ConfigMap files
    - name: config-volume
      mountPath: /etc/my-config
    # 2. Where to place the Secret files
    - name: secret-volume
      mountPath: /etc/my-secrets
      
  volumes:
  # 3. Defining the ConfigMap as a volume
  - name: config-volume
    configMap:
      name: ui-config
  # 4. Defining the Secret as a volume
  - name: secret-volume
    secret:
      secretName: db-credentials
```

Apply the Pod:
```bash
kubectl apply -f 2-volume-pod.yaml
```

💡 **What is happening here?** This is a two-step process. First, at the bottom under `volumes:`, we tell the Pod to treat our ConfigMap and Secret as storage drives. Second, under `volumeMounts:`, we tell the container exactly which folder path to attach those drives to.

---

### 🧪 Step 8: Verifying Volume Mounts
Let's jump into the second pod and explore the file system.

First, list the contents of the `/etc/my-config` directory:
```bash
kubectl exec vol-test-pod -- ls /etc/my-config
```
*Result: You will see two files named `language` and `theme`.* Now, read the contents of the `theme` file:
```bash
kubectl exec vol-test-pod -- cat /etc/my-config/theme
```
*Result: It will print `dark-mode`.*

Read the contents of the secret file:
```bash
kubectl exec vol-test-pod -- cat /etc/my-secrets/password
```
*Result: It will print `SuperSecret123!`.*

---

### ⚠️ Common Mistakes & Troubleshooting
* **Secrets are Base64 Encoded, NOT Encrypted!** 🚨 A common misconception is that Kubernetes Secrets are highly secure. By default, they are simply Base64 encoded. 
  Let's prove it. Run this command to view the raw YAML of your secret:
  ```bash
  kubectl get secret db-credentials -o yaml
  ```
  Look at the `password` field. It looks like gibberish (`U3VwZXJTZWNyZXQxMjMh`). Now, let's decode it:
  ```bash
  echo "U3VwZXJTZWNyZXQxMjMh" | base64 -d
  ```
  *Result: It instantly decodes back to `SuperSecret123!`.* **Lesson:** Anyone with permissions to run `kubectl get secret` can read your passwords. Use strict Role-Based Access Control (RBAC) to protect them!
* **Environment Variables are Immutable:** If you update a ConfigMap, pods that have that ConfigMap injected as an Environment Variable will **not** see the change until the pod is restarted.
* **Volume Mounts Auto-Update:** If you update a ConfigMap that is mounted as a *Volume*, Kubernetes will dynamically update the file inside the running container (though it may take a minute or two to sync).
* **Missing References:** If a Pod specifies a `configMapKeyRef` but you misspelled the name of the ConfigMap, the Pod will fail to start and will be stuck in a `CreateContainerConfigError` state.

---

### 🧹 Lab Cleanup
To clean up your environment, delete your resources, switch back to the default context, and remove your personal namespace.

```bash
# Switch back to the default context
kubectl config use-context default

# Delete your student namespace
kubectl delete namespace $NS

# Delete the custom context you created
kubectl config delete-context ${NS}-context
```

---

### 🎓 Lab Recap & Review
Congratulations on completing the lab! Let's quickly review the core concepts you just put into practice:

* **Decoupling is Key:** 🔑 You learned how to separate application code (the container image) from its configuration (ConfigMaps) and sensitive data (Secrets).
* **Two Ways to Consume Data:** ✌️ You injected data directly into the container's memory as **Environment Variables**, and you mounted data directly onto the container's hard drive as **Volumes**.
* **Volumes vs. Envs:** 📁 You learned that Volume mounts can dynamically update while the pod is running, whereas Environment Variables require a pod restart to pick up changes.
* **Secrets are just Base64:** 🔓 You learned a critical security lesson: Native Kubernetes secrets are obfuscated using Base64 encoding, not heavily encrypted. RBAC is required to keep them truly safe!
