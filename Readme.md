# Dynamic NFS Provisioning with `nfs-subdir-external-provisioner`

Perfect 👍 — since you already have an NFS server (`192.168.0.11`), this guide shows  
a real‑world, easy, production‑style way to set up **dynamic provisioning**.

We'll use:

* **nfs-subdir-external-provisioner** (most common & simple)
* **Helm** (cleanest installation method)

This allows you to:

* stop creating PersistentVolumes (PVs) manually
* create only PersistentVolumeClaims (PVCs)
* automatically create/delete NFS subdirectories

---

## ✅ Architecture (What Will Happen)

1. Install the provisioner in the cluster.
2. It mounts the exported base directory on your NFS server.
3. When a PVC is created it makes a subdirectory.
4. When the PVC is deleted (and `reclaimPolicy=Delete`) the folder is deleted too.

No more:

* `kubectl patch pv`
* dealing with `Released` state
* manual cleanup

---

## 🚀 Step‑by‑Step Setup

### 1️⃣ Add the Helm repository

```sh
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update
```

This project is maintained under Kubernetes SIG Storage.

### 2️⃣ Install the Provisioner

Replace:

* `192.168.0.11` → your NFS server
* `/nfs/nfs_share` → your exported base directory

```sh
helm install nfs-client nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=192.168.0.11 \
  --set nfs.path=/nfs/nfs_share \
  --set storageClass.name=nfs-client \
  --set storageClass.defaultClass=true
```

### 3️⃣ Verify It’s Running

```sh
kubectl get pods
```

You should see something like:

* `nfs-client-nfs-subdir-external-provisioner-xxxxx` Running

Check StorageClass:

```sh
kubectl get storageclass
```

You should see:

* `nfs-client` (default)

✅ Now This Is ALL You Need (No PVs)

Create only a PVC:

```sh
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gatus-data
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-client
  resources:
    requests:
      storage: 1Gi
```

Apply:

```sh
kubectl apply -f pvc.yaml
```

Check:

```sh
kubectl get pvc
kubectl get pv
```

You’ll see:

* A PV automatically created
* Bound to your PVC

On your NFS server, a folder like this will appear:

`/nfs/nfs_share/default-gatus-data-pvc-xxxxx`

🔁 Deleting Is Now Easy

Just:

```sh
kubectl delete pvc gatus-data
```

And:

* PV is deleted
* NFS subdirectory is deleted automatically

No patching.
No manual claimRef clearing.

---

## 🛠 Production-Safe Helm Install (Recommended Options)

If this is production, I recommend adding:

* `--set storageClass.reclaimPolicy=Delete \
* `--set storageClass.archiveOnDelete=false`

That ensures:

* Folder is removed when PVC is deleted
* No leftover archived folders

---

## 🔐 Important: NFS Server Requirements

Make sure your NFS export allows:

* `rw`
* `no_root_squash` (or proper UID/GID handling)

Access from all worker nodes

Example /etc/exports:

`/nfs/nfs_share *(rw,sync,no_subtree_check,no_root_squash)`

Then reload:

```sh
exportfs -ra
```

---

## 🏆 This Is How Most Real Clusters Run NFS

Home labs

On-prem clusters

Rancher setups

Bare metal Kubernetes

Very common setup.

If you tell me:

* Which Kubernetes distro (kubeadm? RKE? k3s?)
* Are you using Helm already?
* Do you want namespace isolation per team?

I can give you a slightly more production-hardened version.

---

## Any permission need to check on nfs ? /nfs/nfs_share

Yes ✅ — permissions on `/nfs/nfs_share` are VERY important for dynamic NFS provisioning.

Most “PVC stuck” or “permission denied” problems come from NFS permissions.

Here’s what you should check.

---

## ✅ 1️⃣ Check NFS Export Configuration

On your NFS server:

```sh
cat /etc/exports
```

You should have something like:

`/nfs/nfs_share *(rw,sync,no_subtree_check,no_root_squash)`
Important options:
Option	Why
rw	Allow write access
sync	Safer writes
no_subtree_check	Avoid path issues
no_root_squash	Prevent permission problems

After editing:

```sh
exportfs -ra
```

⚠️ About no_root_squash

Without no_root_squash:

* Root inside container becomes nobody
* Folder creation may fail
* You get permission denied

For home lab / internal cluster → it's fine.

For production → better to manage with proper UID/GID instead of disabling root squash.

---

## ✅ 2️⃣ Check Directory Ownership

Check:

```sh
ls -ld /nfs/nfs_share
```

Common safe setup:

* `chown -R nobody:nogroup /nfs/nfs_share`
* `chmod -R 0777 /nfs/nfs_share`

Easy Mode (Most People Do This in Labs)
```sh
chmod 0777 /nfs/nfs_share
```

It just works.

---

## 🏭 Production-Style (Cleaner Way)

Instead of 777, use group-based access.

Example:

* `groupadd nfsusers`
* `chgrp -R nfsusers /nfs/nfs_share`
* `chmod -R 0775 /nfs/nfs_share`

Then in your pod:

```sh
securityContext:
  fsGroup: 1000
```

This ensures containers can write properly.

---

## ✅ 3️⃣ Make Sure All Worker Nodes Can Mount It

From each worker node:

```sh
showmount -e 192.168.0.11
```

Then test mount:

```sh
mount -t nfs 192.168.0.11:/nfs/nfs_share /mnt
```

If this fails → Kubernetes will fail too.

---

## 🔎 4️⃣ If Using nfs-subdir-external-provisioner

The provisioner pod must be able to:

* Mount `/nfs/nfs_share`
* Create directories
* Delete directories

If permissions are wrong, you’ll see errors like:

* `permission denied`
* `mkdir failed`

Check logs:

```sh
kubectl logs <nfs-provisioner-pod>
```

---

## 🧠 Real-World Recommendation

For most clusters (home lab / internal infra):

Just do:

* `chmod -R 0777 /nfs/nfs_share`
* `no_root_squash`

That solves 95% of issues.

🚨 If You Want More Secure Setup

Tell me:

* Is this internal-only network?
* Is it production with security requirements?
* Are pods running as non-root?

Then I’ll give you a secure-but-working configuration.
