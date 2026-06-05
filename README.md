# gitops-training

Lab GitOps trên ArgoCD: **App-of-Apps + ApplicationSet**, multi-source, **1 base Helm chart dùng chung**.
**Single branch (`main`), directory-based** — project / app / env đều là thư mục.

## Cấu trúc

```
main
├── root-application.yaml          # App-of-Apps gốc — apply tay 1 lần
├── bootstrap/
│   ├── appprojects/               # AppProject: platform, birdnet-market, mention-mate
│   ├── appsets/                   # ApplicationSet: platform, all-projects
│   └── shared-gateway-app.yaml    # Application → Gateway dùng chung
├── helm-charts/app/               # 1 base chart duy nhất, mọi app dùng chung
├── platform/gateway/              # Gateway dùng chung (shared-gw)
├── apps/                          # overlay env, gom theo project/app
│   ├── birdnet-market/
│   │   ├── frontend/overlays/{dev,staging,prod}/values.yaml
│   │   └── backend/overlays/{dev,staging,prod}/values.yaml
│   └── mention-mate/
│       └── app/overlays/{dev,staging,prod}/values.yaml
└── scripts/seal.sh                # bọc kubeseal
```

## Bootstrap (1 lệnh tay duy nhất)

```bash
kubectl apply -f root-application.yaml
```

`root` → `bootstrap/` (recurse), theo sync-wave:

| Wave | Thành phần |
|------|------------|
| -2 | AppProjects |
| -1 | appset `platform` → sealed-secrets, kgateway-crds, kgateway |
| 0  | appset `all-projects` → quét `apps/*/*/overlays/*`, sinh 1 Application / (project, app, env) |
| 1  | `shared-gateway` → Gateway `shared-gw` (kgateway-system) |

## Apps (directory-based)

`all-projects` dùng **git directories generator** quét `apps/<project>/<app>/overlays/<env>` trên `main`:

- Application = `{project}-{app}-{env}`, namespace = `{project}-{env}`, AppProject = `{project}`
- **Multi-source, cả hai cùng ở `main`** (cùng revision): `source[0]` = chart `helm-charts/app`, `source[1]` = `values.yaml` của env (`$values`)
- Thêm env/app/project = tạo thêm thư mục, không cần đụng appset

| Project | Apps | Application |
|---|---|---|
| birdnet-market | `frontend`, `backend` | 2 × 3 env = 6 |
| mention-mate | `app` (backend + worker) | 1 × 3 env = 3 |

## Base chart `app`

1 chart cho mọi app, render theo map `components`:

- mỗi component → 1 **Deployment** (+ **Service** nếu có `port`, + **ConfigMap** nếu có `config`, + **HTTPRoute** nếu `httpRoute.enabled`)
- 1 component = single-deployment; nhiều component = multi-deployment
- **1 SealedSecret** dùng chung cho cả release (`envFrom` vào mọi component)
- HTTPRoute gắn vào **Gateway dùng chung** `shared-gw` (kgateway-system)

Cấu trúc `components` đầy đủ: xem `helm-charts/app/values.yaml`.

## SealedSecret

Repo test hiện đặt `sealedSecret.enabled: false` (dùng image `traefik/whoami`, không cần secret).
Khi cần secret thật, bật lại và sinh ciphertext bằng kubeseal — scope `strict` gắn theo **name + namespace**,
tên secret phải khớp `<release>-secret`:

```bash
# release = {project}-{app}-{env} ; secret = <release>-secret ; ns = {project}-{env}
./scripts/seal.sh mention-mate-dev mention-mate-app-dev-secret DB_PASSWORD=... API_KEY=...
# dán block encryptedData vào values.yaml của env đó.
```

## Đăng ký repo cho ArgoCD

Repo này **public** → ArgoCD clone không cần creds. Chỉ cần OCI cho kgateway:

```bash
argocd repo add cr.kgateway.dev/kgateway-dev/charts --type helm --enable-oci
```

repo-server cần egress tới GitHub, `cr.kgateway.dev`, `bitnami-labs.github.io`. ArgoCD **≥ 3.1** (native OCI Helm); lab pin **3.3.x**.
