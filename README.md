# gitops-training — Deployment repo

Repo GitOps quản lý **nhiều project**, mỗi project **1 branch** (branch-based); trong mỗi
project mỗi **env là 1 directory** (directory-based). Application/ApplicationSet đều khai báo
trong Git. Có **1 base chart DUY NHẤT** `helm-charts/app` (không version, không OCI); mỗi env
chỉ cần 1 `values.yaml`.

## Layout branch

| Branch | Nội dung |
|--------|----------|
| `main` | bootstrap (`root-application.yaml`, `bootstrap/`) + base chart `helm-charts/app/` |
| `project-mention-mate` | `apps/mention-mate-gateway`, `apps/mention-mate-processor` — overlay dev/staging/prod |
| `project-birdnet-market` | `apps/birdnet-market-service`, `apps/birdnet-market-pipeline` — overlay dev/staging/prod |

## Bootstrap (1 lệnh duy nhất chạy tay)

```bash
kubectl apply -f root-application.yaml
```

Chain tự động:
`root` → `bootstrap/` (recurse) → (wave -2) AppProjects trong `bootstrap/appprojects/`
→ (wave -1) ApplicationSet `platform` trong `bootstrap/appsets/` (sinh sealed-secrets + kgateway-crds + kgateway)
→ (wave 0) ApplicationSet `all-projects` trong `bootstrap/appsets/` → quét 2 branch project → sinh
**12 Application** (2 project × 2 app × 3 env).

```
helm-charts/
└── app/           # 1 chart duy nhất: configmap, sealedsecret, deployment, service, httproute
bootstrap/
├── appprojects/   # AppProject: platform, mention-mate, birdnet-market
└── appsets/       # ApplicationSet: platform (nền tảng), projects (all-projects)
```

## Platform components

Do ApplicationSet `platform` (`bootstrap/appsets/platform.yaml`, list generator) quản lý:

| Component | Nguồn | Namespace |
|-----------|-------|-----------|
| sealed-secrets-controller | Helm `bitnami-labs.github.io/sealed-secrets` | `kube-system` |
| kgateway-crds + kgateway | Helm OCI `cr.kgateway.dev/kgateway-dev/charts` | `kgateway-system` |

## Base chart `app` — 1 chart cho mọi app

Render theo `components` (map). **Mỗi component = 1 Deployment**, kèm Service (nếu có `port`),
ConfigMap (nếu có `config`), HTTPRoute (nếu `httpRoute.enabled`). 1 component → single-deploy;
nhiều component → multi-deploy. SealedSecret là **1 cái dùng chung** cho cả release (envFrom vào
mọi component). HTTPRoute trỏ tới **Gateway dùng chung** (`gateway.name`/`gateway.namespace`).
Xem cấu trúc đầy đủ trong `helm-charts/app/values.yaml`.

## Mỗi env = 1 directory chứa 1 file

```
apps/<app>/overlays/<env>/
└── values.yaml     # Helm values override theo env (components, config, sealedSecret.encryptedData)
```

ApplicationSet dùng git **directories generator** quét `apps/*/overlays/*` để phát hiện env
(không cần `config.yaml`). Chart luôn là `helm-charts/app` trên `main`; khác biệt giữa env/app
nằm hết trong `values.yaml` ⇒ promotion = sửa `values.yaml` của env tương ứng.

## SealedSecret — QUAN TRỌNG

`encryptedData` trong các `values.yaml` ở repo này là **PLACEHOLDER**, KHÔNG decrypt được.
Sinh ciphertext thật bằng kubeseal (gắn với public key của controller trong cluster):

```bash
./scripts/seal.sh mention-mate-dev mention-mate-gateway-secret DB_PASSWORD=... API_KEY=...
# copy block encryptedData vào values.yaml tương ứng, commit, push.
```

## Đăng ký repo cho ArgoCD (on-prem)

```bash
# Git repo (vừa cấp base chart, vừa cấp values)
argocd repo add https://github.com/ongtungduong/gitops-training.git --username '<user>' --password '<PAT>'
# Platform: kgateway OCI Helm registry (sealed-secrets dùng Helm HTTP, không cần)
argocd repo add cr.kgateway.dev/kgateway-dev/charts --type helm --enable-oci
```

- repo-server cần **egress** tới GitHub, `cr.kgateway.dev`, `bitnami-labs.github.io` (air-gapped thì mirror nội bộ).
- ArgoCD **>= 3.1** mới có native OCI Helm (cho kgateway); lab pin dòng **3.3.x**.
- Biến path generator (`.path.path`, `.path.basename`, `.path.segments`) chỉ dùng khi `goTemplate: true`.
