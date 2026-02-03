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
