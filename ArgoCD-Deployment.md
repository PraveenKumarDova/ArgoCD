## **1. What is GitOps? **

GitOps is a modern way to manage Kubernetes deployments.

**Key points:**

* **Git = Source of Truth** → All Kubernetes manifests live in Git.
* **Declarative** → You describe *what* you want (YAML), not *how* to do it.
* **Pull-based** → A tool (like Argo CD) pulls from Git and updates the cluster.
* **Continuous Reconciliation** → It keeps checking if the cluster matches Git, and fixes it if not.

**Why GitOps?**

* Every change is tracked in Git history.
* Easy rollbacks.
* Fewer manual mistakes.
* Strong security (no direct cluster access for everyone).

---

## **2. What is Argo CD? **

**Definition:**
Argo CD is a **GitOps operator** for Kubernetes that:

* Continuously monitors a Git repo.
* Deploys changes automatically or manually.
* Shows visual status in a Web UI.
* Can manage multiple clusters.

**Argo CD Components:**

1. **argocd-server** → Web UI + API.
2. **application-controller** → Syncs Git with cluster.
3. **repo-server** → Fetches Git repo, processes Helm/Kustomize.
4. **Redis** → Cache for performance.

---

## **3. How Argo CD Works **

1. You define your Kubernetes YAML in Git.
2. Argo CD pulls those files.
3. It compares **Git (desired state)** vs **Cluster (live state)**.
4. If different → you sync manually or auto-sync.
5. If auto-sync + self-heal → it updates automatically.

---

## **4. Setting up Argo CD on our AWS Kubernetes Cluster **

### **Step 1 – Install Argo CD**

We install Argo CD in its own namespace:

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check if all pods are running:

```bash
kubectl get pods -n argocd
```

---

### **Step 2 – Access Argo CD UI**

Since we’re using a self-managed kubeadm cluster, we’ll use **NodePort**:

```bash
kubectl -n argocd patch svc argocd-server \
  -p '{"spec":{"type":"NodePort","ports":[{"name":"https","port":443,"targetPort":8080,"nodePort":30080}]}}'


# Switch the existing service to an internet-facing ELB
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"LoadBalancer"}}'

# (Optional but recommended) ensure it listens on 80/443 cleanly
kubectl patch svc argocd-server -n argocd --type='json' -p='[
  {"op":"replace","path":"/spec/ports/0/port","value":80},
  {"op":"replace","path":"/spec/ports/1/port","value":443}
]'

```

Get Node Public IP:

```bash
kubectl get nodes -o wide
```

Open browser:

```
https://<NODE_PUBLIC_IP>:30080
```

**Note:** Accept security warning (self-signed certificate).

---

### **Step 3 – Login to Argo CD**

Get the admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

* Username: `admin`
* Password: (from above)

---

## **5. Deploying a Sample App with Argo CD **

We will deploy the **Guestbook app**.

---

### **Step 1 – Prepare Namespace**

```bash
kubectl create namespace dev
```

---

### **Step 2 – Fork the Sample Repo**

* Go to: [https://github.com/argoproj/argocd-example-apps](https://github.com/argoproj/argocd-example-apps)
* Click **Fork** into your GitHub account.

---

### **Step 3 – Create App in Argo CD UI**

* Click **NEW APP**.
* Fill in:

  * **Application Name**: `guestbook-dev`
  * **Project**: `default`
  * **Sync Policy**: Manual (first time)
  * **Repository URL**: (Your fork HTTPS URL)
  * **Revision**: `main`
  * **Path**: `guestbook`
  * **Cluster URL**: `https://kubernetes.default.svc`
  * **Namespace**: `dev`

Click **CREATE**.

---

### **Step 4 – Sync the App**

* Open the new app.
* Click **SYNC** → **Synchronize**.
* Argo CD will create Kubernetes resources.

---

### **Step 5 – Verify in Cluster**

```bash
kubectl get pods -n dev
kubectl get svc -n dev
```

You should see the guestbook pods running.

---

## **6. Auto-Sync & Self-Heal Demo **

**Step 1 – Enable Auto-Sync**

* In the app → **App Details** → **Actions** → **Enable Auto-Sync**.
* Check **Prune** and **Self-Heal**.

---

**Step 2 – Change in Git → Auto Deployment**

* In your fork, edit `guestbook-ui-deployment.yaml`:

```yaml
replicas: 1
```

change to:

```yaml
replicas: 3
```

* Commit and push.
* Argo CD will detect the change and deploy automatically.

---

**Step 3 – Self-Heal**
Manually scale in cluster:

```bash
kubectl scale deploy guestbook-ui -n dev --replicas=5
```

* Wait \~1 min → Argo CD will revert it back to **3**.

---

## **7. Wrap-Up **

**Summary for Students:**

* Argo CD is a **GitOps tool** for Kubernetes.
* Everything comes from **Git**.
* You can deploy **manually** or **automatically**.
* **Self-Heal** enforces Git state on the cluster.
* NodePort is easiest for accessing Argo CD in self-managed clusters.

---

## **8. Commands Recap**

```bash
# Install Argo CD
kubectl create ns argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access via NodePort
kubectl -n argocd patch svc argocd-server -p '{"spec":{"type":"NodePort","ports":[{"name":"https","port":443,"targetPort":8080,"nodePort":30080}]}}'
kubectl get nodes -o wide  # get public IP

# Get password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Create namespace for app
kubectl create ns dev

# Verify resources
kubectl get pods -n dev
kubectl get svc -n dev
```

---

## **9. Homework for Students**

* Try deploying another app from the same repo.
* Enable auto-sync + self-heal for it.
* Change replicas in Git and watch Argo CD update.
* Delete a pod manually → see self-heal recreate it.

---
