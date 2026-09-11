# Panduan Deployment Microservice E-Wallet di Kubernetes (K3d)

Dokumentasi ini menjelaskan arsitektur, konfigurasi, dan perintah operasional untuk menjalankan seluruh sistem e-wallet di cluster Kubernetes lokal.

---

## 1. Arsitektur & Service Discovery

Semua komponen dideploy di dalam namespace `ewallet`. Komunikasi internal antar-pod menggunakan DNS Kubernetes (CoreDNS):

```
                        [Client / Postman]
                                │
                        (Port 8000 via Ingress)
                                │
                     ┌──────────┴──────────┐
                     │   Traefik Ingress   │
                     └──────────┬──────────┘
             /user/*            │ /wallet/*         \ /transaction/*
         ┌──────────────────────┼────────────────────┐
         ▼                      ▼                    ▼
   [Service: ums]       [Service: wallet]   [Service: transaction]
   Port :80             Port :80            Port :80
         │                      ▲                    │
         │ gRPC :7000           │ HTTP :80           │ gRPC :7000 (UMS)
         │ (Token Validation)   │ (Credit / Debit)   │ & HTTP :80 (Wallet)
         │                      │                    │
         ▼                      │                    ▼
   [Service: notification] ─────┘              [Service: notification]
   Port :7003                                  Port :7003
```

- **MySQL Database**: `mysql:3306` (PVC 1Gi)
- **UMS (User Management)**:
  - HTTP: `http://ums:80` (Internal port 8080)
  - gRPC: `ums:7000` (Internal port 7000)
- **Wallet Service**:
  - HTTP: `http://wallet:80` (Internal port 8081)
- **Transaction Service**:
  - HTTP: `http://transaction:80` (Internal port 8082)
- **Notification Service**:
  - gRPC: `notification:7003` (Internal port 7003)

---

## 2. Struktur File Manifest

| File | Resource K8s | Deskripsi |
| :--- | :--- | :--- |
| `mysql.yaml` | Namespace, PVC, Deployment, Service | Database MySQL 8.4 dengan Persistent Volume Claim |
| `configmap.yaml` | ConfigMap & Secret | Kredensial DB, JWT secret, dan Host Service Discovery |
| `ingress.yaml` | Ingress (Traefik) | API Gateway routing path `/user`, `/wallet`, `/transaction` |
| `ums-*.yaml` | Deployment & Service | User Management Service |
| `wallet-*.yaml` | Deployment & Service | Wallet & Balance Service |
| `transaction-*.yaml`| Deployment & Service | Transaction Service |
| `notification-*.yaml` | Deployment & Service | Notification Service |

---

## 3. Cara Menjalankan (Deploy)

```bash
# 1. Terapkan Namespace & Database
kubectl apply -f mysql.yaml

# 2. Terapkan ConfigMap & Secret
kubectl apply -f configmap.yaml

# 3. Terapkan Semua Microservice
kubectl apply -f notification-service.yaml -f notification-deployment.yaml \
              -f wallet-service.yaml -f wallet-deployment.yaml \
              -f ums-service.yaml -f ums-deployment.yaml \
              -f transaction-service.yaml -f transaction-deployment.yaml

# 4. Terapkan Ingress
kubectl apply -f ingress.yaml
```

---

## 4. Cara Mengakses Sistem Secara Praktis

### Pilihan A: Menggunakan Ingress (Rekomendasi - Cukup 1 Port)
Jalankan port-forward ke Ingress Controller Traefik:
```bash
kubectl port-forward svc/traefik -n kube-system 8000:80
```
Semua service kini bisa diakses melalui satu pintu di `localhost:8000`:
- **UMS**: `http://localhost:8000/user/v1/...`
- **Wallet**: `http://localhost:8000/wallet/v1/...`
- **Transaction**: `http://localhost:8000/transaction/v1/...`

### Pilihan B: Port Forwarding Per-Service
```bash
kubectl port-forward svc/ums -n ewallet 8080:80
kubectl port-forward svc/wallet -n ewallet 8081:80
kubectl port-forward svc/transaction -n ewallet 8082:80
```

---

## 5. Perintah Troubleshooting & Observability

```bash
# Cek semua resource di namespace ewallet
kubectl get pods,svc,pvc,ingress -n ewallet

# Melihat detail suatu pod (misal terjadi error/pending)
kubectl describe pod <nama-pod> -n ewallet

# Melihat streaming log pod
kubectl logs -l app=ums -n ewallet -f
kubectl logs -l app=wallet -n ewallet -f
kubectl logs -l app=transaction -n ewallet -f
kubectl logs -l app=notification -n ewallet -f

# Masuk ke dalam container (misal cek MySQL CLI)
kubectl exec -it deployment/mysql -n ewallet -- mysql -uroot -proot
```
