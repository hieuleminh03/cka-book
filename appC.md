---
layout: chapter

title: "Phụ lục C"
subtitle: "Lộ trình học 8–10 tuần"

previous_link: "appB.html"
previous_title: "Phụ lục B. Lab & môi trường ôn tập"
next_link: ""
next_title: ""
---

Con số 8–10 tuần là mốc tham chiếu cho một chứng chỉ khi bạn đã có nền tảng và học đều mỗi ngày. Trong thực tế, thời gian phụ thuộc vào xuất phát điểm, nên phụ lục này chia thành hai lộ trình rõ ràng: **Lộ trình A** cho người đã có nền Kubernetes (CKA 6 tuần + CKS 6 tuần, tổng 12 tuần) và **Lộ trình B** cho người chưa có nền (CKA 8 tuần + CKS 6 tuần, tổng 14 tuần).

Bạn có thể học nhanh hơn hoặc chậm hơn — điều quan trọng là giữ nhịp và tự đánh giá bằng checkpoint ở cuối mỗi tuần, thay vì học hết sách rồi mới kiểm tra. Nếu chỉ có khoảng 1 giờ mỗi ngày, hãy giãn lộ trình thêm 1.5 lần; đừng cố nhồi cho kịp bảng.

## Nội dung phụ lục {#appendix-content}

- [Chọn lộ trình](#chon-lo-trinh)
- [Lộ trình A — đã có nền Kubernetes (12 tuần)](#track-a)
- [Lộ trình B — từ đầu (14 tuần)](#track-b)
- [Mẫu lịch học 2h/ngày và 4h cuối tuần](#schedule)
- [Checklist tiến độ theo tuần](#progress-checklist)
- [Tự đánh giá đã sẵn sàng thi](#readiness)
- [Ôn tập nhắc lại (spaced repetition)](#spaced-repetition)
- [Xử lý khi lệch lộ trình](#xu-ly-lech)
- [Checklist trước ngày thi](#exam-day)

---

## Chọn lộ trình {#chon-lo-trinh}

Hãy tự trả lời nhanh ba câu hỏi sau. Nếu trả lời "có" cho cả ba, chọn Lộ trình A.

1. Bạn tạo được Deployment + Service + Ingress từ manifest trống mà không cần tra cứu, và giải thích được sự khác nhau giữa ClusterIP, NodePort và LoadBalancer?
2. Bạn đọc hiểu một Role/RoleBinding và tự viết được NetworkPolicy default-deny cho namespace?
3. Bạn đã từng dùng `kubectl logs`, `kubectl describe`, `kubectl exec` để debug một workload lỗi trong công việc thật?

| Tiêu chí | Lộ trình A | Lộ trình B |
|---|---|---|
| Xuất phát điểm | Đã làm việc với Kubernetes | Chưa hoặc mới dùng ở mức cơ bản |
| Thời gian CKA | 6 tuần | 8 tuần |
| Thời gian CKS | 6 tuần | 6 tuần |
| Tổng | **12 tuần** | **14 tuần** |
| Nhịp học đề xuất | 2h/ngày thường + 4h cuối tuần | 2h/ngày thường + 4h cuối tuần |
| Chương cần đọc kỹ | ch05–ch11 | ch01–ch11 |

Một lưu ý về thứ tự: đăng ký thi CKA **trước khi** bắt đầu để có deadline thật và kích hoạt 2 lượt Killer.sh. Với CKS, bạn phải đậu CKA còn hiệu lực trước khi thi; đừng đăng ký CKS quá sớm nếu chưa chắc ngày đậu CKA.

---

## Lộ trình A — đã có nền Kubernetes (12 tuần) {#track-a}

### Giai đoạn 1: CKA (tuần 1–6) {#track-a-cka}

| Tuần | Mục tiêu | Chương | Lab | Checkpoint |
|---|---|---|---|---|
| 1 | Ôn nền Linux và networking phục vụ troubleshooting | ch01, ch02 | Lab ch01, lab ch02 | Viết được NetworkPolicy default-deny; giải thích đường đi của request trong cluster |
| 2 | Container và workload | ch03, ch04 (workload) | Lab ch03, ch04 bước 1–5 | Tạo Deployment/Service trong dưới 5 phút; debug được CrashLoopBackOff |
| 3 | Service, Ingress, ConfigMap/Secret, storage | ch04 (phần còn lại) | Lab ch04 đầy đủ | Expose app qua Ingress TLS; mount PVC thành công |
| 4 | Vận hành cluster: RBAC, etcd, upgrade | ch05 | Lab ch05 | Backup/restore etcd; tạo Role/RoleBinding; drain/uncordon đúng thứ tự |
| 5 | Troubleshooting và mock CKA | ch05, ch17 (CKA) | 2 bộ mock CKA | Mock đạt ≥ 70%; trung bình mỗi task dưới 8 phút |
| 6 | Nước rút và thi CKA | Ôn theo file lỗi | Killer.sh lượt 1 và 2 | Killer.sh đạt mức điểm đậu, dư ít nhất 15 phút |

### Giai đoạn 2: CKS (tuần 7–12) {#track-a-cks}

| Tuần | Mục tiêu | Chương | Lab | Checkpoint |
|---|---|---|---|---|
| 7 | Cluster Setup: NetworkPolicy nâng cao, CIS, Ingress TLS | ch06 | Lab ch06 | Chạy kube-bench, remediation một check FAIL và verify lại |
| 8 | Cluster Hardening và System Hardening | ch07, ch08 | Lab ch07, lab ch08 (VM kubeadm) | RBAC least privilege; AppArmor/seccomp chặn được hành vi |
| 9 | Minimize Microservice Vulnerabilities | ch09 | Lab ch09 | Áp Pod Security Standards; mã hoá Pod-to-Pod (Cilium) |
| 10 | Supply Chain Security | ch10 | Lab ch10 | Trivy/cosign/kube-linter chặn được image xấu trong pipeline |
| 11 | Monitoring, Logging and Runtime Security | ch11 | Lab ch11 | Viết Falco rule; điều tra một chuỗi sự kiện từ audit log |
| 12 | Mock CKS và thi CKS | ch17 (CKS) | Mock CKS + Killer.sh lượt 1 và 2 | Mock ≥ 70%; Killer.sh đạt mức điểm đậu |

Sau khi đậu CKS, các chương 12–16 là phần mở rộng theo JD thị trường — học tiếp như kỹ năng nghề, không còn là điều kiện thi.

---

## Lộ trình B — từ đầu (14 tuần) {#track-b}

### Giai đoạn 1: CKA (tuần 1–8) {#track-b-cka}

| Tuần | Mục tiêu | Chương | Lab | Checkpoint |
|---|---|---|---|---|
| 1 | Linux nền tảng: process, systemd, permission, disk | ch01 | Lab ch01 | Dùng thành thạo systemctl, journalctl, ps/top, ss, df/du |
| 2 | Networking nền tảng: TCP/IP, DNS, TLS, reverse proxy | ch02 | Lab ch02 | Phân giải DNS, kiểm tra TLS bằng openssl, đọc được tcpdump cơ bản |
| 3 | Container và Docker | ch03 | Lab ch03 | Viết Dockerfile multi-stage; thao tác image bằng crictl |
| 4 | Kubernetes căn bản: kiến trúc và workload | ch04 (workload) | Lab ch04 bước 1–4 | Tạo/xoá Pod, Deployment, Service; đọc được pod describe |
| 5 | Config, Secret, storage và Ingress | ch04 (phần còn lại) | Lab ch04 đầy đủ | Cấu hình ConfigMap/Secret/PVC; expose app qua Ingress |
| 6 | Vận hành: RBAC, etcd, service account | ch05 (phần đầu) | Lab ch05 bước 1–4 | Tạo Role/RoleBinding; snapshot và restore etcd |
| 7 | Upgrade cluster và troubleshooting | ch05 (phần còn lại) | Lab ch05 đầy đủ + 2 VM kubeadm | Chạy kubeadm upgrade plan; xử lý node NotReady, pod Pending |
| 8 | Mock và thi CKA | ch17 (CKA) | 2 bộ mock + Killer.sh | Mock ≥ 70%; Killer.sh đạt mức điểm đậu |

### Giai đoạn 2: CKS (tuần 9–14) {#track-b-cks}

| Tuần | Mục tiêu | Chương | Lab | Checkpoint |
|---|---|---|---|---|
| 9 | Cluster Setup | ch06 | Lab ch06 | NetworkPolicy nâng cao, kube-bench, Ingress TLS |
| 10 | Cluster Hardening + System Hardening | ch07, ch08 | Lab ch07, lab ch08 | RBAC least privilege; AppArmor/seccomp hoạt động trên VM |
| 11 | Microservice Vulnerabilities | ch09 | Lab ch09 | Pod Security Standards; isolation; mã hoá Pod-to-Pod |
| 12 | Supply Chain Security | ch10 | Lab ch10 | SBOM, ký và verify image, static analysis manifest |
| 13 | Monitoring, Logging and Runtime Security | ch11 | Lab ch11 | Falco phát hiện hành vi; đọc audit log |
| 14 | Mock CKS và thi CKS | ch17 (CKS) | Mock + Killer.sh | Mock ≥ 70%; Killer.sh đạt mức điểm đậu |

---

## Mẫu lịch học 2h/ngày và 4h cuối tuần {#schedule}

| Ngày | Thời lượng | Cấu trúc buổi học |
|---|---|---|
| Thứ 2 – Thứ 6 | 2 giờ | 20 phút ôn bài cũ (spaced repetition) + 50 phút lý thuyết chương + 40 phút lab + 10 phút ghi chú và checkpoint |
| Thứ 7 | 4 giờ | 2 giờ lab lớn hoặc mock theo phần + 1 giờ đọc lại đáp án/lỗi + 1 giờ ôn flashcard và cheat sheet |
| Chủ nhật | 4 giờ | 2 giờ mock hoặc task-based + 1 giờ review chi tiết + 1 giờ lên kế hoạch tuần sau |

Tổng khoảng 18 giờ/tuần. Vài nguyên tắc để lịch này chạy được:

- **Không bỏ buổi lab.** Lý thuyết 50 phút chỉ có giá trị khi bạn gõ 40 phút ngay sau đó.
- **Bắt đầu buổi học bằng ôn bài cũ**, không phải bài mới — đây là phần dễ bị cắt nhất nhưng lại quyết định trí nhớ dài hạn.
- **Đo thời gian task.** Tập thói quen bấm giờ cho mỗi task mock: đề thi 2 giờ cho 15–20 task, trung bình 6–8 phút/task.
- **Nghỉ giữa buổi.** Học 2 giờ liền không nghỉ hiệu quả thấp hơn 2 hiệp 50 phút có nghỉ 10 phút.
- **Chủ nhật không học lý thuyết mới** — chỉ làm đề, chữa bài và lên kế hoạch.

Nếu bạn học theo Lộ trình B và chưa quen gõ lệnh, hãy chuyển 20 phút "ôn bài cũ" thành 20 phút luyện gõ imperative command và vim cho đến tuần 4.

---

## Checklist tiến độ theo tuần {#progress-checklist}

Đánh dấu khi checkpoint của tuần đã đạt. Chỉ chuyển tuần khi các ô của tuần hiện tại đã tick — ngoại lệ duy nhất là tuần mock.

**Lộ trình A (12 tuần):**

- [ ] **A1** — Lab ch01 và ch02 xong; viết NetworkPolicy default-deny không cần tra docs.
- [ ] **A2** — Tạo Deployment/Service dưới 5 phút; debug được CrashLoopBackOff.
- [ ] **A3** — Ingress TLS và PVC hoạt động; giải thích được storage class.
- [ ] **A4** — Backup/restore etcd thành công; RBAC đúng least privilege.
- [ ] **A5** — Mock CKA ≥ 70%; mỗi task trung bình dưới 8 phút.
- [ ] **A6** — Thi CKA; Killer.sh đạt mức điểm đậu với thời gian dư.
- [ ] **A7** — kube-bench chuyển một check FAIL sang PASS có verify.
- [ ] **A8** — AppArmor/seccomp chặn được hành vi; RBAC hardening xong.
- [ ] **A9** — Pod Security Standards và mã hoá Pod-to-Pod chạy được.
- [ ] **A10** — Pipeline chặn image có CVE cao; cosign verify thành công.
- [ ] **A11** — Falco rule tự viết bắt được sự kiện; đọc được audit log.
- [ ] **A12** — Thi CKS; mock và Killer.sh đều đạt mức điểm đậu.

**Lộ trình B (14 tuần):**

- [ ] **B1** — Thành thạo systemctl, journalctl, ps/top, ss, df/du, permission.
- [ ] **B2** — DNS/TLS/reverse proxy: tự kiểm tra bằng dig, curl, openssl.
- [ ] **B3** — Dockerfile multi-stage; crictl thao tác được image/container.
- [ ] **B4** — Pod/Deployment/Service: tạo, sửa, xoá, đọc describe thành thạo.
- [ ] **B5** — ConfigMap/Secret/PVC/Ingress chạy được trên cluster kind.
- [ ] **B6** — RBAC và etcd snapshot/restore không cần xem lại lý thuyết.
- [ ] **B7** — Upgrade plan trên VM kubeadm; xử lý được node NotReady và pod Pending.
- [ ] **B8** — Thi CKA; mock ≥ 70% và Killer.sh đạt mức điểm đậu.
- [ ] **B9** — Cluster Setup: NetworkPolicy nâng cao, kube-bench, Ingress TLS.
- [ ] **B10** — Hardening cluster và host: RBAC, AppArmor, seccomp.
- [ ] **B11** — Pod Security Standards, secret an toàn, encryption Pod-to-Pod.
- [ ] **B12** — Supply chain: SBOM, ký/verify image, lint và scan manifest.
- [ ] **B13** — Falco và audit log: phát hiện và điều tra được sự kiện.
- [ ] **B14** — Thi CKS; mock và Killer.sh đều đạt mức điểm đậu.

---

## Tự đánh giá đã sẵn sàng thi {#readiness}

Đừng dựa vào cảm giác "đã đọc hết sách". Dùng các tiêu chí đo được sau đây; nếu còn bất kỳ ô nào chưa đạt, hãy lùi ngày thi thay vì thi với hi vọng.

- [ ] **Mock CKA ≥ 70%** hai lần liên tiếp, làm trong đúng 2 giờ, không tra docs ngoài kubernetes.io.
- [ ] **Mock CKS ≥ 70%** hai lần liên tiếp, bao gồm cả task NetworkPolicy, RBAC, PSS, supply chain, runtime.
- [ ] **Killer.sh đạt mức điểm đậu** (66% CKA, 67% CKS) và còn ít nhất 15 phút khi hết giờ.
- [ ] **Tạo manifest trọng tâm dưới 5 phút** không cần docs: Pod + Service, NetworkPolicy, Role/RoleBinding, PVC, Deployment có probe và resources.
- [ ] **Mỗi domain đạt tối thiểu 60%** theo bảng dưới — không có domain nào bị bỏ trắng.
- [ ] **Xử lý sự cố dưới 10 phút**: từ pod Pending hoặc CrashLoopBackOff, tìm ra nguyên nhân bằng describe/logs/events.
- [ ] **Gõ lệnh không cần cheat sheet**: context, rollout, scale, drain, crictl, helm, etcd snapshot.
- [ ] **Đã làm ít nhất một lần full 2 giờ liên tục**, không ngắt, không nghỉ dài, mô phỏng đúng điều kiện phòng thi.

| Chứng chỉ | Domain | Trọng số | Mức tối thiểu tự đặt |
|---|---|---|---|
| CKA | Troubleshooting | 30% | 70% |
| CKA | Cluster Architecture, Installation & Configuration | 25% | 65% |
| CKA | Services & Networking | 20% | 65% |
| CKA | Workloads & Scheduling | 15% | 65% |
| CKA | Storage | 10% | 60% |
| CKS | Minimize Microservice Vulnerabilities | 20% | 70% |
| CKS | Supply Chain Security | 20% | 70% |
| CKS | Monitoring, Logging and Runtime Security | 20% | 70% |
| CKS | Cluster Setup | 15% | 65% |
| CKS | Cluster Hardening | 15% | 65% |
| CKS | System Hardening | 10% | 60% |

---

## Ôn tập nhắc lại (spaced repetition) {#spaced-repetition}

Trí nhớ về lệnh và manifest chỉ bền nếu được nhắc lại theo khoảng giãn. Hãy đưa các mốc sau vào lịch học ngay từ tuần 1:

| Mốc | Việc làm | Thời lượng |
|---|---|---|
| D+1 | Gõ lại các lệnh chính của bài hôm trước, không nhìn tài liệu | 15 phút |
| D+3 | Làm lại một lab nhỏ của bài từ trí nhớ, chỉ mở docs khi bí | 30 phút |
| D+7 | Đọc "Các điểm chính" của chương và làm 5 câu hỏi thực hành | 30 phút |
| D+21 | Làm lại mock hoặc task-based của chương đó | 45 phút |
| Trước thi 3 ngày | Đọc file lỗi cá nhân + lướt Phụ lục A | 60 phút |

Vài quy tắc thực dụng:

- **Không đọc lại lý thuyết nếu đã làm được lab.** Đọc lại tạo cảm giác quen thuộc giả, không tạo kỹ năng.
- **Flashcard chỉ dành cho thứ cần nhớ chính xác**: flag của kubectl, cổng dịch vụ, apiVersion, thứ tự lệnh upgrade, tham số etcdctl.
- **Mỗi tuần một buổi "từ trí nhớ"**: chọn một lab cũ, làm lại không xem hướng dẫn, sau đó mới so kết quả.
- **Ghi lỗi, không ghi thành tích.** File `loi-thi.md` càng dày thì ngày thi càng nhẹ.
- **Chỉ học thêm chủ đề mới khi các mốc cũ đã xong** — nhồi chương mới trong khi bài cũ chưa chắc là cách quên nhanh nhất.

---

## Xử lý khi lệch lộ trình {#xu-ly-lech}

Gần như ai cũng lệch lộ trình ít nhất một lần: công việc gấp, ốm, hoặc một chương khó hơn dự kiến. Điều quan trọng không phải là không bao giờ lệch, mà là biết cách xử lý:

- **Lệch dưới 1 tuần**: cắt buổi lý thuyết, giữ buổi lab. Kỹ năng thực hành là thứ không thể học bù nhanh.
- **Lệch 1–2 tuần**: gộp các chương bạn đã vững (ví dụ ch03 nếu đã quen Docker) thành một buổi đọc nhanh, tuyệt đối không cắt lab của chương trọng số cao như ch05, ch06, ch09–ch11.
- **Lệch hơn 2 tuần**: dời ngày thi. Gói exam có hiệu lực 12 tháng, còn thi khi chưa sẵn sàng thì vừa mất tiền vừa mất tinh thần.
- **Không bao giờ cắt buổi mock** để học lý thuyết — mock là cách duy nhất để biết mình đang ở đâu.
- **Đừng học bù bằng cách học 2 chương trong một ngày**: hãy chọn một chương, học cho chắc, và chấp nhận trượt tiến độ vài ngày.

---

## Checklist trước ngày thi {#exam-day}

### Trước ngày thi 1–2 tuần {#exam-day-before}

- [ ] Đăng ký thi trên training.linuxfoundation.org, chọn slot và múi giờ phù hợp (tránh khung giờ quá khuya).
- [ ] Xác nhận CKA còn hiệu lực nếu bạn thi CKS.
- [ ] Cài và chạy **system check** của PSI Secure Browser trên đúng máy sẽ dùng khi thi; kiểm tra webcam, micro, loa và tốc độ mạng.
- [ ] Tắt VPN, proxy, tường lửa cá nhân và các ứng dụng chặn màn hình khi chạy system check.
- [ ] Chuẩn bị phòng thi: bàn trống chỉ có laptop và giấy tờ tuỳ thân, đèn đủ sáng, không có màn hình phụ, điện thoại để ngoài tầm với.
- [ ] Kiểm tra giấy tờ tuỳ thân có ảnh còn hạn (CCCD hoặc hộ chiếu) — tên phải khớp tài khoản Linux Foundation.
- [ ] Chuẩn bị phương án mạng dự phòng (hotspot 4G) và sạc laptop.
- [ ] Đọc lại chính sách được dùng tài liệu: chỉ kubernetes.io/docs, kubernetes.io/blog và github.com/kubernetes; mở thử vài trang quen thuộc để chắc chắn truy cập được.

### Trong ngày thi {#exam-day-during}

- [ ] Đăng nhập sớm 30 phút để hoàn tất check-in và các bước xác minh danh tính với giám thị.
- [ ] Đóng mọi ứng dụng không cần thiết, tắt thông báo, bật chế độ im lặng.
- [ ] Ngay khi vào đề: set alias `k=kubectl`, `export do='--dry-run=client -o yaml'`, kiểm tra context và namespace.
- [ ] Đọc lướt toàn bộ task, đánh dấu task khó, làm task dễ trước.
- [ ] Với mỗi task: đổi context, làm, verify, chỉ chuyển tiếp khi đã kiểm tra kết quả.
- [ ] Không dành quá 10 phút cho một task; nếu bí, đánh dấu và quay lại khi còn thời gian.
- [ ] Dành 10 phút cuối để rà lại các task đã flag và kiểm tra các object dễ sai tên/namespace/label.
- [ ] Giữ bình tĩnh với các câu hỏi ngoài dự đoán: đề thi cập nhật theo version mới, nhưng kiến thức nền trong sách đủ để suy luận.

Bộ lệnh thiết lập nhanh, lặp lại ở đầu mỗi task:

```bash
alias k=kubectl
export do='--dry-run=client -o yaml'
kubectl config use-context <context-cua-task>
kubectl config set-context --current --namespace=<namespace-cua-task>
```

### Sau khi thi {#exam-day-after}

- [ ] Ghi lại ngay các chủ đề gặp trong đề và cảm nhận về quản lý thời gian (chỉ ghi chủ đề, không ghi nội dung đề).
- [ ] Nếu đậu: lên kế hoạch dùng chứng chỉ (cập nhật CV, LinkedIn) và tiếp tục chương 12–16 cho phỏng vấn.
- [ ] Nếu chưa đậu: dùng lượt thi lại trong gói, khoanh vùng domain yếu từ bảng tự đánh giá và lặp lại lộ trình nước rút 2–3 tuần.
- [ ] Dù kết quả thế nào, giữ nhịp ôn tập nhắc lại — kiến thức Kubernetes mất đi nhanh nếu không dùng đến.
