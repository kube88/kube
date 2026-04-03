
***

### 💾 Kubernetes Storage Lab: Persistent Volumes & Storage Classes

🎯 **Lab Objective**
In this lab, you will explore how Kubernetes handles stateful applications and persistent data. You will learn how to:
* Perform initial cluster configuration.
* Explore and understand the available Storage Classes in your cluster.
* Establish a secure workspace and context.
* Create a Persistent Volume Claim (PVC) to dynamically provision storage.
* Inspect both the local PVCs and the cluster-wide Persistent Volumes (PVs).
* Deploy a Pod that mounts and consumes the persistent volume.
* Prove data persistence by destroying and recreating the pod.
* **Extension:** Dynamically expand the size of an existing volume.

🧠 **Core Concepts: What are we actually doing?**
Before jumping into the command line, it is important to understand how Kubernetes handles data:
* **The Ephemeral Problem:** 💨 By default, containers are ephemeral. If a pod crashes or is deleted, any data written inside that container is destroyed. This is fine for web servers, but catastrophic for databases.
* **Persistent Volumes (PV):** 💽 A PV is a piece of actual storage infrastructure (like a vSphere VMDK or vSAN object) that has been provisioned for the cluster to use. It exists independently of any single pod's lifecycle.
* **Persistent Volume Claims (PVC):** 🎫 A PVC is a user's "request" for storage. You don't need to know the underlying storage tech; you simply claim "I need 1GB of fast storage," and Kubernetes finds or creates a PV to match.
* **Storage Classes (SC):** 🏭 Storage Classes are profiles set up by administrators that define *how* storage should be created. They enable "Dynamic Provisioning," meaning a physical volume is automatically created on the backend the moment you submit a PVC.

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
cd "FRI 02 - Storage"
```

💡 **What is happening here?** Kubernetes uses a configuration file (often called `kubeconfig`) to know which cluster to talk to and what credentials to use. By copying the shared config file to your home directory, you are giving your `kubectl` command the "keys" to the cluster. Finally, we navigate to the correct folder so that our YAML files are easy to find.

🔎 **Step 2: Exploring Storage Classes**
Before we request storage, we need to see what "profiles" (Storage Classes) are available to us. These classes define how storage is provisioned on the backend.

Run the following command:
```bash
kubectl get storageclass
```

You should see the following list of available policies returned from the environment:
```text
NAME
cluster-wld01-01a-vsan-storage-policy
cluster-wld01-01a-vsan-storage-policy-latebinding
k8s-harbor
k8s-harbor-latebinding
k8s-storage-policy
k8s-storage-policy-latebinding
management-storage-policy-encryption
management-storage-policy-encryption-latebinding
management-storage-policy-regular
management-storage-policy-regular-latebinding
management-storage-policy-single-node
management-storage-policy-single-node-latebinding
management-storage-policy-stretched-lite
management-storage-policy-stretched-lite-latebinding
management-storage-policy-thin
management-storage-policy-thin-latebinding
vm-encryption-policy
vm-encryption-policy-latebinding
vsan-default-storage-policy
vsan-default-storage-policy-latebinding
```

💡 **What do these names mean?** Storage Classes can vary by cluster. Some common parameters include "Reclaim Policy" (what happens to data when a PVC is deleted) and "Volume Binding Mode" (when the volume is physically created). 

For our lab, we will be using the default storage class provided by the cluster.

🏗️ **Step 3: Setting Up Your Workspace**
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

🎫 **Step 4: Requesting Storage (Creating a PVC)**
Now, let's ask Kubernetes for a persistent disk. We will explicitly tell Kubernetes which Storage Class to use.

Create a file named `1-pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data-claim
spec:
  storageClassName: management-storage-policy-regular 
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply the claim:
```bash
kubectl apply -f 1-pvc.yaml
```

**Verify the PVC (The Request):**
Let's list the Persistent Volume Claims in our namespace to ensure our request was successful.
```bash
kubectl get pvc
```
*You should see `my-data-claim` with a status of **Bound**.*

**Verify the PV (The Physical Disk):**
Now, let's look at the actual cluster-wide physical volumes that Kubernetes knows about. 
```bash
kubectl get pv
```
*Notice the output! You will see a dynamically generated PV with a long, random name (e.g., `pvc-1a2b3c4d...`). Look at the `CLAIM` column—you will see it is explicitly mapped to `storage-lab/my-data-claim`.*

💡 **What is happening here?** We asked for `1Gi` using the defined policy (`get pvc`). Kubernetes saw this request, talked directly to the storage provider, created a physical 1GB disk, registered it as a Persistent Volume (`get pv`), and tied the two together. 

📦 **Step 5: Deploying a Stateful Pod**
Storage is useless without an application to consume it. Let's deploy an Nginx web server and mount our new volume into its web directory.

Create a file named `2-stateful-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-server
  labels:
    app: web
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html" # 1. Where inside the container the disk lives
      name: my-storage-volume
  volumes:
  - name: my-storage-volume
    persistentVolumeClaim:
      claimName: my-data-claim # 2. The K8s resource we are attaching
```

Apply the pod:
```bash
kubectl apply -f 2-stateful-pod.yaml
```

Wait a few moments and verify it is running:
```bash
kubectl get pods
```

🧪 **Step 6: Testing Data Persistence**
Let's prove the data outlives the pod.

**Test 1: Write data to the volume**
Execute a command inside the running pod to create a custom HTML page:
```bash
kubectl exec data-server -- sh -c "echo '<h2>Success! This data is PERSISTENT!</h2>' > /usr/share/nginx/html/index.html"
```

Verify the data is there:
```bash
kubectl exec data-server -- cat /usr/share/nginx/html/index.html
```
✅ You should see the HTML string you just wrote.

**Test 2: Destroy the Pod 💥**
Simulate a node failure or an accidental deletion by destroying the pod:
```bash
kubectl delete pod data-server
```

**Test 3: Recreate the Pod and check the data**
Re-apply the exact same pod definition. Kubernetes will schedule a new container and re-attach the existing PVC.
```bash
kubectl apply -f 2-stateful-pod.yaml
```

Wait for it to run (`kubectl get pods`), then check the data again:
```bash
kubectl exec data-server -- cat /usr/share/nginx/html/index.html
```
✅ **It SUCCEEDED!** Even though the original pod was completely destroyed, the underlying PV retained the data, and the new pod seamlessly picked up right where the old one left off.

🌟 **Extension Lab: Dynamic Volume Expansion**
Imagine your application becomes incredibly popular and you are running out of storage space on your 1Gi volume. Modern Kubernetes allows you to resize volumes on the fly without deleting them.

Edit your existing `1-pvc.yaml` file and change the requested storage from `1Gi` to `2Gi`:
```yaml
  resources:
    requests:
      storage: 2Gi # Changed from 1Gi
```

Apply the updated PVC:
```bash
kubectl apply -f 1-pvc.yaml
```

Check the PVC and PV status to see the size change:
```bash
kubectl get pvc my-data-claim
kubectl get pv
```
*(Note: It may take a minute for the underlying vSphere storage to process the expansion, but you will eventually see the capacity increase to 2Gi on both the PVC and the PV).*

⚠️ **Common Mistakes & Troubleshooting**
* **Pending PVCs:** ⏳ If your PVC stays in a `Pending` state, it means Kubernetes could not fulfill the request. This usually happens if you specify a `storageClassName` that doesn't exist, or if you use a `-latebinding` class and haven't deployed a pod yet to trigger the creation.
* **Multi-Attach Errors:** 🚫 Because we used `ReadWriteOnce`, the volume can only be attached to one physical host at a time. If you try to deploy two pods on *different* physical nodes that both point to the same PVC, the second pod will fail to start with a "Multi-Attach Error". 
* **Context Confusion:** 🤷‍♂️ If you get "NotFound" errors, verify you are working in the correct namespace using `kubectl config current-context`.

🚀 **Bonus Concept: Scaling with StatefulSets**
In this lab, we successfully attached a single Pod to a single PVC. But what if you wanted to run a database like MongoDB with 3 replicas for high availability? 

If you used a standard Kubernetes `Deployment` and scaled it to 3 replicas, all three pods would try to mount our exact same `my-data-claim` PVC. Because the PVC is `ReadWriteOnce`, this would instantly crash two of the pods with a Multi-Attach error!

**Enter the StatefulSet:** 🗄️
A StatefulSet is an advanced Kubernetes controller designed specifically for databases and stateful applications. Instead of referencing a single existing PVC, a StatefulSet uses a `volumeClaimTemplate`. 
When you tell a StatefulSet to scale to 3 replicas:
1. It creates Pod 1 (`db-0`), and dynamically provisions a brand new, unique PVC just for that pod.
2. It waits for `db-0` to be ready, then creates Pod 2 (`db-1`) and provisions a second unique PVC.
3. It repeats this for Pod 3 (`db-2`).

If `db-1` crashes, the StatefulSet restarts it and automatically reattaches its specific, original disk. This ensures sticky identity and ordered storage provisioning, making StatefulSets the gold standard for running databases in Kubernetes!

🧹 **Lab Cleanup**
To clean up your environment, delete the resources, switch your context back to the default, and remove your personal namespace.

```bash
# Delete the Pod and PVC
kubectl delete -f 2-stateful-pod.yaml
kubectl delete -f 1-pvc.yaml

# Switch back to the default context
kubectl config use-context default

# Delete your student namespace
kubectl delete namespace $NS

# Delete the custom context you created
kubectl config delete-context ${NS}-context
```

🎓 **Lab Recap & Review**
* **Storage Classes map to Infrastructure:** 🏭 You saw how a PVC triggered the cluster to automatically provision a real, physical disk using a specific Storage Class policy.
* **PVs vs PVCs:** 💽 You successfully inspected both sides of the storage coin—the developer's request (the PVC in your namespace) and the administrator's actual disk (the global PV).
* **Pods and Storage have separate lifecycles:** 🔄 You proved that destroying a container does not destroy its data.
* **Storage is mutable (sometimes):** 📈 You learned that modern Kubernetes allows for dynamic volume expansion.
