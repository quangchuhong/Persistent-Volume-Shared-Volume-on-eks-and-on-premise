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
**Lưu ý:**

**volumeBindingMode** là thuộc tính của StorageClass, quyết định khi nào và ở đâu Kubernetes sẽ tạo & bind PersistentVolume (PV) cho PVC.

Có 2 giá trị chính:

**1. volumeBindingMode: Immediate (mặc định)**
  - PV được tạo & bind ngay khi PVC được tạo, trước khi pod được schedule.

  - Scheduler chưa biết pod sẽ chạy ở node/AZ nào → với storage có ràng buộc AZ (EBS) dễ bị:

    - PV tạo ở AZ A,
    - Pod được schedule vào node ở AZ B → không attach được volume → pod Pending / phải reschedule.

ví dụ:
```bash
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-ebs-immediate
provisioner: ebs.csi.aws.com
volumeBindingMode: Immediate
parameters:
  type: gp3
  fsType: ext4

```
Dùng ổn với:

  - Storage không phụ thuộc topology (NFS, EFS, v.v.)
  - Môi trường đơn giản, 1 AZ.
    
**2. volumeBindingMode: WaitForFirstConsumer (nên dùng cho EBS)**

  - PV chỉ được tạo & bind khi có pod đầu tiên dùng PVC đó được schedule.

  - Scheduler sẽ:
    - Chọn node (và AZ) phù hợp cho pod,
    - Sau đó provisioner tạo EBS volume trong AZ của node đó → tránh mismatch AZ.
      
Ví dụ (EKS/EBS khuyên dùng):
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
Lợi ích:

  - Đảm bảo PV (EBS) nằm đúng AZ với node chạy pod.
  - Tránh lỗi pod Pending do “volume node affinity conflict”.
---


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
---
### 2. Shared Volume trên EKS – dùng EFS

2.1. Mục đích

Dùng khi:

  - Nhiều pod / nhiều deployment (có thể trên nhiều node) cần chung 1 thư mục
  - Ví dụ: thư mục upload chung, log chung, share config, build cache…
    
EBS không share giữa nhiều node. Cần dùng EFS (Network FS).

### 2.2. Cài AWS EFS CSI Driver
```bash
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver
helm repo update

helm upgrade --install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  -n kube-system

```
Tạo EFS filesystem trước (trên AWS console/CLI), lấy fileSystemId.

### 2.3. StorageClass dùng EFS
```bash
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  fileSystemId: fs-xxxxxxxx      # ID EFS
  directoryPerms: "700"
reclaimPolicy: Retain
volumeBindingMode: Immediate

```
```bash
kubectl apply -f storageclass-efs-sc.yaml
```
### 2.4. PVC RWMany (share cho nhiều pod)
```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data-pvc
  namespace: shared-apps
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi    # EFS là elastic, con số này chỉ là logic

```
### 2.5. Dùng chung PVC trong nhiều Deployment
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
  namespace: shared-apps
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-a
  template:
    metadata:
      labels:
        app: app-a
    spec:
      containers:
        - name: app-a
          image: your-image-a
          volumeMounts:
            - name: shared
              mountPath: /shared
      volumes:
        - name: shared
          persistentVolumeClaim:
            claimName: shared-data-pvc
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-b
  namespace: shared-apps
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-b
  template:
    metadata:
      labels:
        app: app-b
    spec:
      containers:
        - name: app-b
          image: your-image-b
          volumeMounts:
            - name: shared
              mountPath: /shared
      volumes:
        - name: shared
          persistentVolumeClaim:
            claimName: shared-data-pvc

```
app-a và app-b cùng đọc/ghi /shared trên EFS.

---
### 3. Shared Volume trên các server Docker on‑premise
**3.1. Trường hợp 1 – Share volume trên CÙNG 1 server Docker**
Dùng bind mount hoặc docker volume local.

Bind mount:
```bash
mkdir -p /data/shared

docker run -d --name app1 \
  -v /data/shared:/shared \
  your-image-a

docker run -d --name app2 \
  -v /data/shared:/shared \
  your-image-b

```
Docker volume local:

```bash
docker volume create myshared

docker run -d --name app1 -v myshared:/shared your-image-a
docker run -d --name app2 -v myshared:/shared your-image-b

```

Giới hạn:

Chỉ share được giữa các container trên cùng host.

---
**3.2. Trường hợp 2 – Share volume giữa NHIỀU server Docker (multi‑host)**

Cần 1 filesystem mạng (NFS, GlusterFS, CephFS…). Phổ biến và dễ nhất là NFS.

Giả sử:

  - NFS server: nfs01, export /data/shared
  - Docker hosts: docker01, docker02
    
**3.2.1. NFS server**
/etc/exports:
```bash
/data/shared 10.0.0.0/24(rw,sync,no_root_squash)

```
```bash
mkdir -p /data/shared
exportfs -ra

```

**3.2.2. Trên mỗi Docker host**
```bash
mkdir -p /mnt/shared
mount -t nfs nfs01:/data/shared /mnt/shared

# Nếu muốn mount tự động:
echo "nfs01:/data/shared /mnt/shared nfs defaults 0 0" >> /etc/fstab

```

**3.2.3. Chạy container, mount từ NFS đã mount**

Trên docker01:
```bash
docker run -d --name app1 \
  -v /mnt/shared:/shared \
  your-image-a

```
Trên docker02:

```bash
docker run -d --name app2 \
  -v /mnt/shared:/shared \
  your-image-b

```
Cả hai app đều dùng chung /shared (thực chất là EFS NFS on‑prem).

---
### 4. Chọn giải pháp nào?

**EKS + EBS (PV per app)**

  - Khi mỗi app cần volume riêng, không share.
  - Dùng cho database, stateful service, tool có data riêng.
    
**EKS + EFS (shared volume)**

  - Khi nhiều pod trên nhiều node cần share thư mục.
  - Dùng cho upload chung, log chung, build cache, file share.
    
**Docker on‑prem 1 host**

  - Bind mount hoặc docker volume local là đủ.
    
**Docker on‑prem nhiều host**
  - Dùng NFS hoặc distributed FS (GlusterFS/Ceph) → mount vào host → -v vào container.

---

