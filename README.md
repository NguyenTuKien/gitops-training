# gitops-training — Deployment repo (Repo B)

Repo GitOps quản lý **nhiều project**, mỗi project **1 branch** (branch-based); trong mỗi
project mỗi **env là 1 directory** (directory-based). Application/ApplicationSet đều khai báo
trong Git. Base chart lấy từ Repo A (OCI Helm trên Harbor).

## Layout branch

| Branch | Nội dung |
|--------|----------|
| `main` | bootstrap: `root-application.yaml`, `bootstrap/appprojects/`, `bootstrap/sealed-secrets-app.yaml`, `bootstrap/projects-appset.yaml` |
| `project-payment` | `apps/payment-gateway` (java-app), `apps/payment-processor` (python-app) — overlay dev/staging/prod |
| `project-order` | `apps/order-service` (java-app), `apps/order-pipeline` (python-app) — overlay dev/staging/prod |

## Bootstrap (1 lệnh duy nhất chạy tay)

```bash
kubectl apply -f root-application.yaml
```

Chain tự động:
`root` → `bootstrap/` → (wave -2) AppProjects → (wave -1) sealed-secrets-controller
→ (wave 0) ApplicationSet `all-projects` → quét 2 branch project → sinh **12 Application**
(2 project × 2 app × 3 env).

## Mỗi env = 1 directory chứa 2 file

```
apps/<app>/overlays/<env>/
├── config.yaml     # { chart: java-app|python-app, version: <semver OCI> }  -> generator đọc
└── values.yaml     # Helm values override theo env (replicas, config, sealedSecret.encryptedData)
```

> Mỗi env pin **version chart riêng** trong `config.yaml` ⇒ promotion độc lập
> (dev chạy chart mới, prod giữ chart cũ tới khi sẵn sàng).

## SealedSecret — QUAN TRỌNG

`encryptedData` trong các `values.yaml` ở repo này là **PLACEHOLDER**, KHÔNG decrypt được.
Sinh ciphertext thật bằng kubeseal (gắn với public key của controller trong cluster):

```bash
./scripts/seal.sh payment-dev payment-gateway-secret DB_PASSWORD=... API_KEY=...
# copy block encryptedData vào values.yaml tương ứng, commit, push.
```

Xem `docs/repo-server-setup.md` cho phần đăng ký Harbor OCI + git creds + network policy.
