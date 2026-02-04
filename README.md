# Hướng dẫn sử dụng Persistent Volume & Shared Volume  
## Trên EKS (AWS) và Docker on‑premise

Tài liệu này tóm tắt cách:

1. Dùng **Persistent Volume (PV)** trên EKS (EBS)
2. Dùng **Shared Volume** trên EKS (EFS)
3. Dùng **Shared Volume** trên các server Docker on‑premise

---

## 1. Persistent Volume trên EKS (EBS) – cho từng app/pod

### 1.1. Mục đích

Dùng khi:

- Mỗi app cần **storage riêng**, không share cho app khác
- Dùng cho: database, Jenkins home, SonarQube data, Prometheus TSDB…

EBS đặc điểm:

- Volume gắn với **1 AZ**
- Access mode: **ReadWriteOnce** (1 node tại 1 thời điểm)

### 1.2. Cài AWS EBS CSI Driver

Nếu chưa có, cài addon (EKS):

```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster <cluster-name> \
  --region <region> \
  --force
```
Hoặc Helm (nếu không dùng addon):

```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm upgrade --install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  -n kube-system

```
### 1.3. StorageClass sử dụng EBS

Trên EKS thường đã có gp2 hoặc gp3. Nếu muốn tạo riêng:
```bash
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-ebs
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete

```
Apply:
```bash
kubectl apply -f storageclass-gp3-ebs.yaml

```
### 1.4. PVC (PersistentVolumeClaim) dùng EBS
Ví dụ PVC 20Gi cho 1 app:
```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
  namespace: my-app
spec:
  accessModes:
    - ReadWriteOnce      # EBS: chỉ 1 node mount được
  storageClassName: gp3-ebs
  resources:
    requests:
      storage: 20Gi

```
### 1.5. Gắn PVC vào Deployment/StatefulSet
Ví dụ Deployment:
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: your-image
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: app-data-pvc

```
**Khi pod tạo lần đầu** → EBS volume được tạo & attach vào node, mount tại /data. Pod xoá đi (recreate trên cùng node hoặc node khác) → volume được re-attach & mount lại.

**Lưu ý hạn chế EBS:**

  - volume EBS:
    - Chỉ 1 AZ,
    - AccessMode ReadWriteOnce (một node tại một thời điểm).
  - Không dùng EBS để share đồng thời giữa nhiều pod trên nhiều node.
