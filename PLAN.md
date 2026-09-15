# PLAN — Sách **CKA & CKS Study Guide (bản tiếng Việt)**

Repo remote: https://github.com/hieuleminh03/cka-book — deploy GitHub Pages tại https://hieuleminh03.github.io/cka-book/

## 1. Mục tiêu & định vị

- **Đối tượng:** DevOps/Cloud/SRE engineer trình độ middle–senior (2–7 năm KN) tại VN, muốn (1) ôn phỏng vấn middle/senior, (2) lấy chứng chỉ CKA & CKS, (3) hệ thống hóa kiến thức vận hành production.
- **Định vị:** dùng **exam objectives CKA (nền tảng Kubernetes) và CKS (bảo mật Kubernetes) làm khung xương** (giống ocsj21-book dùng exam objectives 1Z0-830) nhưng **bao phủ toàn bộ bộ kỹ năng core 80% của thị trường** (xem `RESEARCH.md`) — giống cách sách Java cover toàn bộ ngôn ngữ Java chứ không chỉ mẹo thi.
- **Khác biệt cốt lõi:** CKS là kỳ thi performance-based (làm task trên terminal), nên mỗi chương = lý thuyết + **lab phải chạy được** + câu hỏi ôn tập + câu hỏi phỏng vấn có trả lời mẫu.

## 2. Thông tin kỳ thi CKS (dùng làm khung objectives)

| Hạng mục | Thông tin |
|---|---|
| Tên | Certified Kubernetes Security Specialist (CKS) |
| Tổ chức | Linux Foundation / CNCF |
| Hình thức | Online, proctored, **performance-based** (15–20 task trên CLI) |
| Thời gian | 2 giờ |
| Điểm đạt | **67%** |
| Điều kiện | Phải có **CKA còn hiệu lực** trước khi thi CKS |
| Phiên bản K8s | Theo bản minor mới nhất (~v1.33; cập nhật 4–8 tuần sau mỗi bản K8s) |
| Hiệu lực | 2 năm; từ 18/06/2026 (CARE) đậu CKS sẽ tự gia hạn CKA tương ứng |
| Học phí | ~$445 (gồm 2 lượt thi, 12 tháng eligibility, 2 lượt Killer.sh simulator) |

### CKS domains hiện hành

| Domain | Trọng số | Competencies chính |
|---|---|---|
| Cluster Setup | 15% | NetworkPolicy, CIS benchmark (etcd/kubelet/kubedns/kubeapi), Ingress TLS, bảo vệ node metadata/endpoints, verify platform binaries |
| Cluster Hardening | 15% | RBAC least privilege, service account (disable default, tối thiểu quyền), restrict K8s API, upgrade cluster để vá lỗ hổng |
| System Hardening | 10% | Giảm host OS footprint, least-privilege IAM, giảm network exposure, AppArmor/seccomp |
| Minimize Microservice Vulnerabilities | 20% | Pod Security Standards, K8s secrets, isolation (multi-tenancy, sandboxed container), Pod-to-Pod encryption (Cilium/Istio) |
| Supply Chain Security | 20% | Giảm base image footprint, SBOM/CI-CD/artifact repo, trusted registry, sign & verify (cosign), static analysis (Kubesec, KubeLinter, Trivy) |
| Monitoring, Logging and Runtime Security | 20% | Behavioral analytics (Falco), threat detection, điều tra các pha tấn công, immutability runtime, Kubernetes audit logs |

## 3. Format sách — bám cấu trúc ocsj21-book

- Jekyll + GitHub Pages, mỗi chương 1 file markdown (`chNN.md`), deploy bằng `.github/workflows/pages.yml`.
- Front matter mỗi chương: `layout: chapter`, `title`, `subtitle`, `previous_link/next_link` (điều hướng như sách gốc).
- **Template một chương:**
  1. **Mục tiêu chương** — liệt kê CKS objective + JD requirement liên quan (thay cho exam objective của sách gốc).
  2. **Lý thuyết** — giải thích tiếng Việt, giữ nguyên thuật ngữ kỹ thuật EN.
  3. **Hands-on lab** — chạy trên kind/k3s (local) hoặc cluster tự dựng, có bước verify rõ ràng.
  4. **Câu hỏi phỏng vấn thường gặp** — kèm trả lời mẫu và lỗi sai phổ biến.
  5. **Bài tập ôn tập cuối chương** — dạng task giống đề CKS + trắc nghiệm kiểm tra khái niệm, có giải thích từng đáp án (giống 1Z0-830).
  6. **Cheat sheet** — lệnh/API snippet tóm tắt chương.
- Đầu sách (`intro.md`) có bảng mapping **CKS objective → chương** và **JD requirement → chương**.

## 4. Cấu trúc chương (18 chương + 3 phụ lục)

### Phần I — Nền tảng (prerequisite CKS + yêu cầu phổ quát của JD)

| Ch | Tên chương | Nội dung chính | Liên quan |
|---|---|---|---|
| 1 | Linux & System Hardening nền tảng | process/systemd, user & permission, filesystem, resource, host attack surface, AppArmor/seccomp | CKS: System Hardening; JD: Linux ~87% |
| 2 | Networking & Cluster Networking | TCP/IP, DNS, TLS, reverse proxy (nginx/HAProxy), CNI, NetworkPolicy, Ingress TLS, mTLS/Pod-to-Pod encryption (Cilium, Istio) | CKS: Cluster Setup + Microservice encryption; JD ~53% |
| 3 | Container & Docker | namespaces/cgroups, image layer, registry, multi-stage build, image footprint, runtime (containerd) | CKS: Supply Chain; JD ~93% |
| 4 | Kubernetes căn bản (CKA-level) | kiến trúc control plane/node, workload, service, configmap/secret, volume, kubectl, Helm | Prerequisite CKS; JD ~93% |
| 5 | Kubernetes vận hành (CKA-level) | upgrade cluster, etcd backup/restore, RBAC/ServiceAccount, API access, kubelet & CIS benchmark components | CKS: Cluster Hardening; JD vận hành |

### Phần II — CKS Domains (trọng tâm chứng chỉ)

| Ch | Tên chương | Nội dung chính | Domain |
|---|---|---|---|
| 6 | Cluster Setup | NetworkPolicy nâng cao, CIS benchmark, Ingress TLS, node metadata/endpoints, verify binaries | 15% |
| 7 | Cluster Hardening | RBAC least privilege, service account hygiene, restrict API, upgrade/ vá lỗ hổng | 15% |
| 8 | System Hardening | giảm footprint host, least privilege, network exposure, AppArmor, seccomp | 10% |
| 9 | Minimize Microservice Vulnerabilities | Pod Security Standards/Admission, secrets, isolation (multi-tenancy, gVisor/Kata), mã hóa Pod-to-Pod, OPA/Kyverno | 20% |
| 10 | Supply Chain Security | SBOM, CI/CD & artifact repo, trusted registry, cosign/Notary, static analysis (Kubesec, KubeLinter, Trivy, Grype) | 20% |
| 11 | Monitoring, Logging & Runtime Security | Falco, behavioral analytics, audit logs, immutability, điều tra tấn công | 20% |

### Phần III — Mở rộng theo thị trường VN (JD mapping, đào sâu senior)

| Ch | Tên chương | Nội dung chính | JD |
|---|---|---|---|
| 12 | CI/CD Engineering | Jenkins, GitLab CI, GitHub Actions, pipeline design, caching/artifact, security gate (SAST/SCA, image scan), CI hardening cho supply chain | ~100% |
| 13 | Cloud & IaC | AWS core (VPC/EC2/IAM/S3/RDS/EKS/ECS), Terraform (state/module/CI), Ansible; Azure/GCP mapping nhanh; cost & IAM security | ~80% + 83% + 47% |
| 14 | Observability | Prometheus/PromQL, Grafana, Alertmanager, Loki/ELK, OpenTelemetry tracing, SLI/SLO/error budget, alerting on-call | ~60% |
| 15 | GitOps & Deployment Strategies | ArgoCD/Flux, GitOps repository model, rolling/blue-green/canary, rollback, Helm release, secrets trong GitOps | ~30% |
| 16 | SRE & Incident Response | on-call, runbook, incident command, root cause/postmortem, chaos testing, capacity & FinOps | senior JD ~20% |

### Phần IV — Ôn tập & phỏng vấn

| Ch | Tên chương | Nội dung chính |
|---|---|---|
| 17 | CKS Mock & Labs | 15–20 task mô phỏng đề thi (từng domain) + 100+ câu kiểm tra khái niệm có giải thích + mẹo phòng thi (time management, docs nào được dùng) |
| 18 | Phỏng vấn Middle/Senior DevOps | 200+ câu hỏi theo chủ đề, system design cho DevOps (CI/CD at scale, multi-env, DR), behavioral, câu hỏi tình huống production |
| A | Cheat sheet | kubectl, terraform, linux, aws-cli, security tooling |
| B | Lab & môi trường | kind/k3s setup, tài nguyên ôn thi miễn phí, Killer.sh |
| C | Lộ trình học 8–10 tuần | chia theo level (middle/senior), kèm checklist tiến độ |

## 5. Mapping CKS objectives → chương (dùng cho intro.md)

| CKS Objective | Chương |
|---|---|
| Cluster Setup (15%) | 2, 6 |
| Cluster Hardening (15%) | 5, 7 |
| System Hardening (10%) | 1, 8 |
| Minimize Microservice Vulnerabilities (20%) | 2, 4, 9 |
| Supply Chain Security (20%) | 3, 10, 12 |
| Monitoring, Logging and Runtime Security (20%) | 11, 14 |

## 6. Mapping JD requirement → chương (dùng cho intro.md)

| JD requirement | Tần suất (RESEARCH.md) | Chương |
|---|---|---|
| Linux + shell | ~87% | 1 |
| Networking, LB, TLS | ~53% | 2 |
| Docker/container | ~93% | 3 |
| Kubernetes | ~93% | 4–11 |
| Terraform + IaC | ~83% | 13 |
| Cloud (AWS/Azure/GCP) | ~80% / 37% / 27% | 13 |
| CI/CD (Jenkins, GitLab CI, GH Actions) | ~100% | 12 |
| Monitoring (Prometheus/Grafana) | ~60% | 14 |
| GitOps/ArgoCD | ~30% | 15 |
| SRE/SLO/incident | ~20% | 16 |
| DevSecOps/security | ~17% | 1, 3, 6–12 |
| Kho phỏng vấn | — | 17, 18 |

## 7. Roadmap thực hiện (~20 tuần)

| Phase | Output | Thời lượng |
|---|---|---|
| P0 | Scaffold Jekyll + GitHub Pages workflow + layouts + `intro.md` (chứa 2 bảng mapping) | Tuần 1 |
| P1 | Phần I: ch01–ch05, mỗi chương có lab chạy được trên kind/k3s | Tuần 2–5 |
| P2 | Phần II: ch06–ch11 (trọng tâm CKS, ưu tiên chất lượng cao nhất) | Tuần 6–11 |
| P3 | Phần III: ch12–ch16 | Tuần 12–16 |
| P4 | Phần IV: ch17–ch18 + phụ lục A/B/C | Tuần 17–20 |
| P5 | Review kỹ thuật (soát version tool/K8s), beta đọc, deploy chính thức | Sau tuần 20 |

**Nguyên tắc thứ tự ưu tiên nội dung:** Tier 1 (RESEARCH.md) viết trước và phải xong chất lượng cao; Tier 2 nằm trong Phần I/III; Tier 3 đưa vào Phần III/IV dưới dạng mục nâng cao.

## 8. Quy ước biên soạn

- **Ngôn ngữ:** giải thích tiếng Việt, giữ nguyên thuật ngữ EN (pod, RBAC, NetworkPolicy, admission controller...); code/YAML/command giữ nguyên 100%, chỉ dịch comment.
- **Anchor heading ổn định** để mục lục hoạt động; front matter có prev/next link.
- **Practice questions:** multiple-choice (kiểm tra khái niệm, giống 1Z0-830) + task-based (giống đề CKS thật).
- **Labs:** mọi lab phải chạy được trên kind/k3s local hoặc AWS free tier; ghi rõ version K8s/tool.
- **Không dịch lại nguyên văn tài liệu có bản quyền khác** — toàn bộ nội dung tự soạn, trích dẫn nguồn khi tham khảo (K8s docs, CNCF curriculum...).

## 9. Cấu trúc repo dự kiến

```
devops-book/
├── _config.yml               # title, description, baseurl/url GitHub Pages
├── Gemfile / Gemfile.lock
├── index.md                  # layout: index
├── intro.md                  # giới thiệu + 2 bảng mapping + lộ trình học
├── ch01.md … ch18.md
├── _layouts/                 # default.html, chapter.html, index.html
├── _includes/
├── assets/                   # css, hình ảnh, sơ đồ
├── 404.html
├── RESEARCH.md
├── PLAN.md
├── LICENSE.txt
└── .github/workflows/pages.yml
```

## 10. License

- Nội dung sách (tự soạn): **CC BY-NC-SA 4.0** (kế thừa tinh thần của sách tham khảo).
- Code trong ví dụ/lab: **MIT**.
