# Nghiên cứu thị trường DevOps Middle/Senior tại Việt Nam

> Tài liệu nền tảng cho việc biên soạn sách. Khảo sát thực hiện T8–T9/2026.
>
> **Phương pháp:** đọc và đếm tần suất kỹ năng từ ~30 JD DevOps middle/senior đăng trên ITviec, Monster VN, TopDev, Joboko, CareerViet và trang tuyển dụng của công ty; đối chiếu với báo cáo ITviec Vietnam IT Salary & Recruitment Market Report 2025–2026, TopDev Vietnam IT Market Report 2024–2025 và thống kê VietnamDevs IT Jobs Statistics 2026.
>
> **Lưu ý:** mẫu khảo sát nhỏ so với toàn thị trường; các con số dưới đây là ước lượng xu hướng theo tỷ lệ xuất hiện trong mẫu, dùng để quyết định độ ưu tiên nội dung sách.

## 1. Tần suất kỹ năng theo JD

### Tier 1 — Bắt buộc, xuất hiện trong ~80–100% JD ("core 80%")

| Kỹ năng | Tần suất | Chi tiết ghi nhận từ JD |
|---|---|---|
| CI/CD | 100% | Jenkins ~50%, GitLab CI ~50%, GitHub Actions ~43%, ArgoCD/GitOps ~30%, Azure DevOps ~13% |
| Git/version control | ~100% | thường gộp trong yêu cầu CI/CD, branching strategy, GitOps |
| Kubernetes | ~93% | EKS phổ biến nhất; AKS, GKE; OpenShift/Rancher ở enterprise, bank |
| Docker | ~93% | kèm containerd, Helm (~23%) |
| Linux + shell | ~87% | administration, troubleshooting, systemd, performance |
| Terraform | ~83% | CloudFormation ~17%, Pulumi rất ít |
| AWS | ~80% | Azure ~37%, GCP ~27% |
| Bash/Python scripting | ~77% | Python là ngôn ngữ automation số 1 (~67%); Go ~17%, PowerShell ~10% |

### Tier 2 — Xuất hiện nhiều (~40–70%)

| Kỹ năng | Tần suất | Chi tiết |
|---|---|---|
| Prometheus + Grafana | ~60% | Alertmanager; Datadog, Zabbix, CloudWatch ở một số JD |
| Networking | ~53% | TCP/IP, DNS, TLS, nginx/HAProxy/traefik/kong, load balancing, VPC |
| Database/middleware | ~50% | PostgreSQL, MySQL, Redis, MongoDB, Kafka, Elasticsearch |
| Ansible | ~47% | configuration management, idempotency |
| ELK/OpenSearch/Splunk/Loki | ~40% | logging stack |

### Tier 3 — Lợi thế cạnh tranh (~15–40%)

| Kỹ năng | Tần suất | Ghi chú |
|---|---|---|
| GitOps/ArgoCD | ~30% | đang tăng nhanh, gắn với K8s |
| Helm | ~23% | chart packaging/release |
| SRE practices (SLI/SLO, incident, on-call) | ~20% | đặc trưng yêu cầu senior |
| DevSecOps | ~17% | SonarQube, Trivy, SAST/SCA/DAST, secrets management |
| OpenTelemetry/Tracing (Jaeger, Tempo) | ~15% | observability nâng cao |
| VMware/on-prem/hybrid | ~13% | bank, enterprise, hệ thống "local cloud" |
| Go | ~17% | automation tooling, operator |
| FinOps/cost optimization | ~10% | thường ghi trong mô tả senior |
| MLOps | ~7% | AI/ML platform (VinFast, CADDI...) |

## 2. Phân khúc tuyển dụng và khác biệt yêu cầu

| Phân khúc | Đại diện | Đặc thù tech stack |
|---|---|---|
| Fintech/ngân hàng | MB, NAB, DNSE, VSI, GPBank, Tyme | DevSecOps, OpenShift/K8s on-prem, compliance, observability, SRE |
| Product company | FPT, Tiki, VNG, MoMo, what3words | AWS/EKS, GitOps, microservices, Terraform, cost optimization |
| Outsourcing/service | NashTech, ELCA, Saigon Technology, KMS | đa dạng cloud (AWS+Azure), CI/CD cho khách hàng, tiếng Anh bắt buộc |
| Enterprise/truyền thống | MiTek, Bosch, Renesas | Azure, Azure DevOps, PowerShell, VMware, Windows Server |
| Startup | Optisigns, CodeLeap, Trustify | AWS serverless, GitHub Actions, làm việc đa nhiệm |

## 3. Khác biệt Middle vs Senior (từ mô tả công việc)

- **Middle (3–5 năm):** thành thạo tool (K8s, Terraform, CI/CD), xử lý task có hướng dẫn, rotation on-call, tham gia incident.
- **Senior (5+ năm):** thiết kế hệ thống (system design), reliability/SLO, security hardening, cost optimization, mentoring, chủ trì incident/root cause analysis, ra quyết định công nghệ.
- Senior không phải là "biết thêm tool" mà là **hiểu sâu bản chất** (tại sao chọn giải pháp này, trade-off, vận hành ở quy mô lớn).

## 4. Chứng chỉ được nhắc trong JD

- CKA/CKAD/CKS (Kubernetes) — xuất hiện trực tiếp trong JD bank/enterprise.
- AWS Certified (SAA, DevOps Engineer Professional), Azure (AZ-104/AZ-400), Terraform Associate.
- Chứng chỉ là lợi thế filter CV, nhưng phỏng vấn middle/senior chủ yếu hỏi kinh nghiệm vận hành thực tế và troubleshooting.

## 5. Kết luận — bộ "core 80%" mà sách phải bao phủ

```
Linux + Networking + Git + CI/CD + Docker + Kubernetes
+ Terraform + một cloud chính (AWS trước, Azure sau)
+ Prometheus/Grafana + Bash/Python
```

Mở rộng bắt buộc để lên senior: Ansible, database/middleware ops, GitOps/ArgoCD,
observability nâng cao (OTel, Loki), SRE practices, DevSecOps/security hardening,
cost optimization, system design.

## 6. Hàm ý cho sách

1. Trục chính là **Kubernetes & security** (khung CKS) vì K8s xuất hiện trong ~93% JD và là kỹ năng phân hóa middle/senior rõ nhất.
2. Vẫn phải có **chương nền tảng Linux/Networking/Docker/Cloud/CI-CD** vì đó là prerequisite của CKS và cũng là yêu cầu phổ quát của thị trường.
3. Mỗi chương cần **lab thực hành** (CKS thi performance-based) và **câu hỏi phỏng vấn** vì người đọc mục tiêu là đi làm phỏng vấn middle/senior.
4. Nội dung security không đứng riêng: gắn supply chain security với CI/CD, runtime security với observability — đúng cách thị trường mô tả công việc.
