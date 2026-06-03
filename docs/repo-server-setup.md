# Yêu cầu repo-server (môi trường on-prem)

ApplicationSet sinh Application multi-source: 1 source là **OCI Helm chart** trên Harbor,
1 source là **git repo này**. repo-server phải truy cập được cả hai.

## 1. Đăng ký Harbor OCI là Helm repository

```bash
argocd repo add harbor.viettel.vn/charts \
  --type helm --name harbor-charts --enable-oci \
  --username '<robot$ci>' --password '<token>'
```

## 2. Đăng ký git repo (Repo B)

```bash
argocd repo add https://gitlab.viettel.vn/devops/gitops-training.git --username '<user>' --password '<token-or-PAT>'
```

## 3. Network policy

repo-server cần **egress** tới: Harbor (OCI), GitLab (git). Trong môi trường air-gapped,
mở firewall/policy tương ứng.

## Lưu ý version

- ArgoCD **>= 3.1** mới có native OCI Helm; lab pin theo dòng **3.3.x** (3.4 là mới nhất).
- Biến path trong generator (`.path.path`, `.path.basename`, `.path.segments`) áp dụng khi
  `goTemplate: true`. Kiểm tra lại đúng tên biến cho version ArgoCD đang chạy.
