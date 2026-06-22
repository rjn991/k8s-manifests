# MinIO Deployment on Kubernetes

Deploy MinIO in standalone mode using:

* Local Persistent Volume (20Gi)
* Manual PV/PVC binding
* Traefik Ingress
* Wildcard TLS certificate
* Custom domains:

  * https://minio.ranjan.host (Console)
  * https://s3.ranjan.host (S3 API)

---

# Folder Structure

```text
minio-manifest/
├── 01-pv.yaml
├── 02-pvc.yaml
├── middleware.yaml
├── minio-console-ingress.yaml
├── minio-api-ingress.yaml
└── README.md
```

---

# Prerequisites

## Verify Nodes

```bash
kubectl get nodes
```

Expected:

```text
NAME          STATUS   ROLES
kube-master   Ready    control-plane
kube-slave    Ready
```

## Verify Traefik

```bash
kubectl get ingressclass
```

Expected:

```text
NAME      CONTROLLER
traefik   traefik.io/ingress-controller
```

## Verify TLS Secret

```bash
kubectl get secret wildcard-ranjan-host -A
```

---

# Prepare Storage

SSH into the worker node:

```bash
ssh kube-slave
```

Create MinIO storage directory:

```bash
sudo mkdir -p /data/minio
sudo chown -R 1000:1000 /data/minio
sudo chmod -R 755 /data/minio
```

Verify:

```bash
ls -ld /data/minio
```

Expected:

```text
drwxr-xr-x 2 1000 1000 ...
```

---

# Create Namespace

```bash
kubectl create namespace minio
```

Ignore the error if the namespace already exists.

---

# Deploy Persistent Volume

Apply:

```bash
kubectl apply -f 01-pv.yaml
```

Verify:

```bash
kubectl get pv
```

Expected:

```text
NAME       CAPACITY   STATUS
minio-pv   20Gi       Available
```

---

# Deploy Persistent Volume Claim

Apply:

```bash
kubectl apply -f 02-pvc.yaml
```

Verify:

```bash
kubectl get pvc -n minio
```

Expected:

```text
NAME        STATUS
minio-pvc   Bound
```

---

# Deploy Middleware

Apply:

```bash
kubectl apply -f middleware.yaml
```

Verify:

```bash
kubectl get middleware -n minio
```

---

# Deploy Ingress Resources

Apply:

```bash
kubectl apply -f minio-console-ingress.yaml
kubectl apply -f minio-api-ingress.yaml
```

Verify:

```bash
kubectl get ingress -n minio
```

Expected:

```text
NAME                    HOSTS
minio-console-ingress   minio.ranjan.host
minio-api-ingress       s3.ranjan.host
```

---

# Install MinIO

Add MinIO repository:

```bash
helm repo add minio https://charts.min.io/
helm repo update
```

Install MinIO:

```bash
helm upgrade --install minio minio/minio \
  -n minio \
  --create-namespace \
  --set mode=standalone \
  --set replicas=1 \
  --set persistence.enabled=true \
  --set persistence.existingClaim=minio-pvc \
  --set rootUser=minioadmin \
  --set rootPassword='ChangeMe123!' \
  --set resources.requests.memory=512Mi \
  --set resources.requests.cpu=250m
```

## Why Resource Requests Are Set

The MinIO Helm chart defaults to requesting:

```yaml
resources:
  requests:
    memory: 16Gi
```

On a small lab cluster this prevents scheduling.

This deployment uses:

```yaml
resources:
  requests:
    memory: 512Mi
    cpu: 250m
```

which is sufficient for testing and small workloads.

---

# Verify Deployment

Pods:

```bash
kubectl get pods -n minio -o wide
```

Expected:

```text
NAME                     READY   STATUS    NODE
minio-xxxxxxxxxx-xxxxx   1/1     Running   kube-slave
```

Services:

```bash
kubectl get svc -n minio
```

Expected:

```text
NAME            TYPE        PORT(S)
minio           ClusterIP   9000/TCP
minio-console   ClusterIP   9001/TCP
```

PVC:

```bash
kubectl get pvc -n minio
```

Expected:

```text
NAME        STATUS   VOLUME
minio-pvc   Bound    minio-pv
```

Ingress:

```bash
kubectl get ingress -n minio
```

---

# Access MinIO

## Console

```text
https://minio.ranjan.host
```

## S3 API

```text
https://s3.ranjan.host
```

## Default Credentials

```text
Username: minioadmin
Password: ChangeMe123!
```

Change the default password after the initial login.

---

# Troubleshooting

## Check Pods

```bash
kubectl get pods -n minio
```

## Check Logs

```bash
kubectl logs -n minio -l app=minio
```

## Describe Pod

```bash
kubectl describe pod -n minio <pod-name>
```

## Check PVC

```bash
kubectl describe pvc minio-pvc -n minio
```

## Check PV

```bash
kubectl describe pv minio-pv
```

## Check Ingress

```bash
kubectl describe ingress minio-console-ingress -n minio
kubectl describe ingress minio-api-ingress -n minio
```

## Check Middleware

```bash
kubectl get middleware -n minio
```

---

# Upgrade MinIO

```bash
helm repo update

helm upgrade minio minio/minio \
  -n minio \
  --reuse-values
```

---

# Uninstall

Remove MinIO:

```bash
helm uninstall minio -n minio
```

Remove Kubernetes resources:

```bash
kubectl delete -f minio-console-ingress.yaml
kubectl delete -f minio-api-ingress.yaml
kubectl delete -f middleware.yaml
kubectl delete -f 02-pvc.yaml
kubectl delete -f 01-pv.yaml
```

Data remains on the worker node because the PV uses:

```yaml
persistentVolumeReclaimPolicy: Retain
```

Storage location:

```text
/data/minio
```
