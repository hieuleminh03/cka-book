---
layout: answer

title: "Chương 18"
subtitle: "Phỏng vấn Middle/Senior DevOps"
exam_objectives:
  - "Trả lời tự tin các câu hỏi phỏng vấn middle/senior theo từng chủ đề."
  - "Trình bày system design cho hạ tầng DevOps và Kubernetes."
  - "Xử lý câu hỏi tình huống production và câu hỏi behavioral."
  - "Nắm mặt bằng lương và cách đàm phán tại thị trường Việt Nam."
---

## Gợi ý trả lời {#sample-answers}

Phần này gợi ý trả lời cho **10 câu hỏi tự luyện** ở cuối chương 18. Mỗi câu gồm: **dàn ý trả lời** (các ý cần nói theo thứ tự), **ý nên nói** (điểm ghi ấn tượng với interviewer), và **ý không nên nói** (lỗi thường gặp). Đây là khung để bạn luyện nói, không phải đáp án duy nhất — miễn các ý chính đầy đủ và có trade-off rõ ràng là đạt.

---

**1. Chuẩn hóa GitOps cho 3 cluster Kubernetes (dev, staging, production).**

**Dàn ý trả lời:**

1. Làm rõ yêu cầu: cloud nào, ai được merge vào repo chứa manifest production, có yêu cầu compliance/audit không, tần suất release.
2. Cấu trúc repo: tách **repo ứng dụng** (code + Dockerfile + Helm chart/template) và **repo config** (manifest theo môi trường); trong repo config dùng base/overlay (Kustomize) hoặc values theo môi trường (Helm); một thư mục cho mỗi cluster (`clusters/dev`, `clusters/staging`, `clusters/prod`) chứa Application/AppProject.
3. ArgoCD: một instance cho non-prod, một instance (hoặc AppProject riêng) cho prod; AppProject giới hạn mỗi team chỉ deploy vào namespace/repo được phép; bật auto-sync cho dev/staging, **manual sync hoặc sync có approval** cho prod.
4. Promotion: CI build image và push với tag bất biến (semver/commit SHA); dev tự động cập nhật digest; staging/prod nhận PR thay đổi digest — review là gate; không ai `kubectl apply` tay, mọi thay đổi đều qua Git.
5. Rollback: `git revert` (hoặc rollback ArgoCD) đưa về digest cũ; với prod có thể dùng blue-green/canary qua Argo Rollouts để giảm blast radius.
6. Bảo mật và audit: OIDC cho CI, không để kubeconfig prod trong CI; secret dùng SealedSecret/External Secrets; mọi thay đổi có PR + history trong Git làm audit log.
7. Vận hành: alert khi OutOfSync/Drift, dashboard sync status từng cluster, kế hoạch onboarding team theo template.

**Nên nói:**

- "Git là nguồn sự thật duy nhất" và mọi thay đổi prod qua PR + approval.
- Tách quyền non-prod tự động — prod cần người duyệt; audit có sẵn trong Git.
- Nêu rõ cấu trúc repo cụ thể và lý do (multi-cluster scale, tránh copy-paste manifest).
- Nhắc drift detection và cảnh báo OutOfSync.

**Không nên nói:**

- Cho CI giữ kubeconfig prod và `kubectl apply` từ pipeline (push-based cho prod) mà không nêu trade-off bảo mật.
- Để mọi team dùng chung một AppProject toàn quyền hoặc một namespace cho cả 3 môi trường.
- Commit secret thô vào repo.

---

**2. Giảm pipeline từ 35–45 phút xuống dưới 10 phút cho PR, không giảm độ tin cậy.**

**Dàn ý trả lời:**

1. Đo trước: tách queue time vs run time; xác định top job tốn thời gian bằng báo cáo từng step (Gantt trong GitLab, timing trong GitHub Actions).
2. Cache: dependency theo lockfile (`npm ci` với cache key `hashFiles('**/package-lock.json')`), cache layer Docker (registry cache/BuildKit), cache build artifact giữa các job.
3. Song song hóa: chia job theo DAG (`needs`), shard test suite theo test file/nhóm, chạy lint + unit + scan song song thay vì tuần tự.
4. Bỏ việc dư thừa: không build lại ở nhiều job (build một lần rồi dùng artifact), không chạy E2E nặng trên mọi PR — chuyển E2E sang staging/nightly, PR chỉ chạy smoke nhỏ; dùng path filter cho monorepo.
5. Hạ tầng runner: runner đủ mạnh, disk nhanh, image base pre-pull; giảm thời gian setup tool bằng image đóng gói sẵn.
6. Flaky test: đo flake rate, retry **có giới hạn** kèm cảnh báo, cách ly test flaky khỏi required check, và sửa nguyên nhân gốc (timing, shared state, port cố định) — không retry vô hạn, không rerun mù.
7. Duy trì: dashboard p50/p95 thời gian pipeline, alert khi vượt ngưỡng, budget thời gian cho từng stage.

**Nên nói:**

- Đo trước, tối ưu sau — chỉ ra thứ tự ưu tiên dựa trên dữ liệu, không "cảm giác".
- Nêu con số mục tiêu từng bước (ví dụ cache −30–50% thời gian cài dependency).
- Flaky test phải xử lý như vấn đề kỹ thuật, không phải rerun.
- Trade-off: E2E ít chạy hơn trên PR nhưng chạy đủ trên staging/nightly; nêu cách bù độ tin cậy.

**Không nên nói:**

- Tăng retry vô hạn hoặc tắt required check để pipeline "xanh".
- Chỉ mua runner mạnh hơn mà không cache/song song hóa.
- Bỏ test để nhanh — giảm độ tin cậy là đi ngược đề bài.

---

**3. Kubernetes multi-tenant cho 5 team dùng chung node pool và Prometheus.**

**Dàn ý trả lời:**

1. Làm rõ: mức cô lập cần thiết (nội bộ hay khách hàng), yêu cầu quota/chargeback, ai vận hành platform.
2. Mô hình: namespace-per-team cho mặc định; mỗi team có quota, RBAC, NetworkPolicy riêng; workload nhạy cảm tách node pool bằng taint/toleration.
3. Cô lập 4 chiều:
   - **Identity**: ServiceAccount + Role/RoleBinding giới hạn trong namespace; không cấp quyền cluster; audit bằng `kubectl auth can-i --list`.
   - **Network**: NetworkPolicy default-deny, mở ingress từ ingress-controller và egress tới DNS kube-system (bắt buộc có rule DNS), DB/service dùng chung mở theo selector.
   - **Resource**: ResourceQuota (CPU/RAM/object count) + LimitRange đặt default requests/limits; PriorityClass để workload quan trọng không bị evict.
   - **Compute**: node pool riêng cho team lớn/nhạy cảm, topology spread tránh dồn một node, cluster autoscaler có trần.
4. Policy engine: Kyverno hoặc Gatekeeper enforce không privileged, bắt buộc requests/limits, chỉ image từ registry nội bộ, verify chữ ký image; PSA `restricted` cho namespace thường.
5. Xử lý team A ăn hết tài nguyên: quota chặn ở tầng namespace; nếu team A vượt limit gây áp lực node, kiểm tra requests/limits pod của A, dùng PriorityClass/preemption để bảo vệ workload quan trọng; tách A sang node pool riêng nếu cần; cost showback theo namespace để team thấy chi phí thực.
6. Quan sát: Prometheus dùng chung nhưng label theo tenant, Grafana phân quyền theo folder, dashboard quota usage từng team, cảnh báo khi usage gần trần.

**Nên nói:**

- Trả lời bằng khung 4 chiều cô lập — đúng yêu cầu và dễ nhớ.
- Quota + default-deny + policy engine là bộ ba bắt buộc.
- Đưa phương án cụ thể cho tình huống "team A ăn hết tài nguyên".
- Nêu khả năng tách node pool hoặc cluster cho workload nhạy cảm như bước leo thang.

**Không nên nói:**

- "Chỉ cần namespace là đủ" — thiếu quota/network/policy là điểm trừ lớn.
- Quên rule DNS trong NetworkPolicy egress (lỗi kinh điển).
- Cho tất cả team dùng chung ServiceAccount hoặc quyền cluster-admin.

---

**4. ALB → nginx Ingress → service → database, lỗi 502 tăng vọt sau deploy 30 phút trước.**

**Dàn ý trả lời:**

1. Xác nhận tác động: tỉ lệ lỗi, bao nhiêu user/endpoint, từ thời điểm nào; kiểm tra dashboard SLO.
2. Quyết định nhanh: deploy 30 phút trước là **nghi phạm số một** → nếu lỗi ảnh hưởng diện rộng, **rollback trước, điều tra sau** (mitigate first). Nếu chỉ một phần, có thể scale thêm hoặc tắt feature liên quan.
3. Khoanh vùng theo tầng nếu cần điều tra: ALB (target group health), Ingress controller (log 502 do upstream timeout/refused), pod (readiness, OOM, restart count), database (connection, slow query, pool exhausted), node (CPU/disk).
4. Dữ liệu cần xem: log nginx (upstream status), metric error rate theo version (label version trong service mesh hoặc Prometheus), trace để biết nằm ở hop nào, `kubectl describe pod` cho các pod mới.
5. Tiêu chí quyết định rollback: lỗi do version mới (so sánh error theo label version), lỗi không giảm sau mitigate, hoặc ảnh hưởng khách hàng lớn. Rollback về digest cũ và xác nhận error rate trở lại bình thường.
6. Sau sự cố: postmortem — tìm root cause thật (config, timeout, migration, resource), bổ sung test/pre-check (canary, smoke test, alert sớm), cập nhật runbook.

**Nên nói:**

- Ưu tiên mitigate/rollback trước khi tìm root cause — thể hiện tư duy SRE.
- Quy trình có thứ tự và dùng dữ liệu để khoanh vùng, không đoán.
- Nêu cụ thể log/metric/trace sẽ xem ở từng tầng.
- Đề xuất cải tiến sau sự cố: canary + auto-rollback theo SLO.

**Không nên nói:**

- "Đọc log đến khi tìm ra lỗi rồi mới xử lý" trong khi production đang cháy.
- Đổ lỗi cho một tầng duy nhất mà không kiểm tra.
- Quên bước xác nhận dịch vụ đã phục hồi sau rollback.

---

**5. Migrate 200 VM on-prem, 30 service lên Kubernetes cloud trong 6 tháng, downtime tối thiểu.**

**Dàn ý trả lời:**

1. Làm rõ: cloud nào, ràng buộc compliance (bank?), service nào stateful, traffic profile, mục tiêu downtime cho phép.
2. Kiến trúc đích: EKS/managed Kubernetes (control plane do nhà cung cấp lo), private subnet, IaC bằng Terraform module, CI/CD + GitOps (ArgoCD), observability chuẩn (Prometheus/Grafana/Loki), security baseline (PSA restricted, Kyverno, external secrets).
3. Lộ trình theo giai đoạn:
   - Tháng 1–2: dựng landing zone (account, VPC, cluster, registry, CI/CD, observability) + chuyển 2–3 service stateless đơn giản làm pilot, viết runbook.
   - Tháng 3–4: migrate phần lớn service stateless theo sóng (wave), dùng chiến lược **strangler/kết nối lai**: on-prem và cloud chạy song song, DNS/load balancer chia traffic, test từng service.
   - Tháng 5–6: xử lý stateful (database, queue), cắt traffic dần, tắt on-prem theo từng nhóm.
4. Stateful:
   - Database: ưu tiên managed service (RDS/Aurora) — dùng replication (logical/physical) từ on-prem lên cloud, cutover ngắn bằng cách chuyển write trong maintenance window nhỏ; luôn có đường quay lui (fallback về on-prem).
   - File storage: S3 + CSI driver, migrate data trước (rclone/DataSync), chuyển dần từ NFS.
   - Kafka/queue: mirror topic (MirrorMaker) rồi cutover producer/consumer theo thứ tự.
5. Downtime tối thiểu: chấp nhận "dual-run" tốn kém một thời gian; mọi cutover có kế hoạch và rehearsal; dùng feature flag/DNS TTL thấp/hai chiều để quay lui.
6. Đo thành công: checklist trước khi cắt từng service (test, monitoring, backup, rollback đã diễn tập), KPI migration (số service/tuần, tỉ lệ lỗi, chi phí).

**Nên nói:**

- Chia sóng (wave) và pilot trước khi làm toàn bộ.
- Strangler pattern + dual-run là cách thực tế để downtime tối thiểu.
- Managed database thay vì tự chạy DB trên Kubernetes (nêu trade-off nếu buộc tự chạy: StatefulSet + operator + backup).
- Có kế hoạch rollback cho từng bước.

**Không nên nói:**

- "Big bang" migrate toàn bộ trong một đêm.
- Đưa database ngay lên pod chạy trong cluster mà không nói lý do/trade-off.
- Kế hoạch không có bước test và rehearsal.

---

**6. Observability cho 60 microservices, ngân sách hạn chế.**

**Dàn ý trả lời:**

1. Xác định yêu cầu: cần trả lời câu hỏi gì (độ trễ ở đâu, lỗi do ai, service nào tệ nhất), SLO nào quan trọng, ngân sách bao nhiêu/tháng.
2. Kiến trúc:
   - **Metric**: Prometheus (hoặc managed) + node-exporter, kube-state-metrics, app expose `/metrics`; Grafana dashboard chuẩn theo service + RED method (Rate, Errors, Duration).
   - **Log**: stdout/stderr + Fluent Bit DaemonSet → Loki (rẻ hơn ELK) hoặc OpenSearch; log dạng JSON có cấu trúc, retention theo mức quan trọng.
   - **Trace**: OpenTelemetry SDK + collector, sampling hợp lý (ví dụ 1–10% + tail sampling cho lỗi/chậm), backend Tempo/Jaeger.
3. Kiểm soát chi phí: giảm cardinality (không dùng label vô hạn như user_id/request_id, log/trace để tra cứu chi tiết); retention metric (15 ngày local + downsampling/long-term rẻ hơn), retention log ngắn cho debug, dài cho audit chỉ với nguồn bắt buộc; tắt metric không dùng; dùng `--ignore-unfixed` ở tầng khác… (với scan) — tập trung quy tắc: **đo cái được dùng**.
4. Chọn SLO: 3 service quan trọng nhất (ví dụ checkout, payment, search) — chọn SLI theo hành trình người dùng (success rate, p95 latency), SLO 99.9% hoặc 99.5%, alert multi-window burn-rate thay vì alert theo mọi metric.
5. Chống alert fatigue: alert theo triệu chứng + actionable, có runbook, phân cấp page/ticket; review alert nhiễu định kỳ; dashboard theo dịch vụ thay vì "tường metric".
6. Lộ trình: giai đoạn 1 metric + log cơ bản + SLO 3 service; giai đoạn 2 trace + dashboard theo team; giai đoạn 3 tuning chi phí, autoscaling, anomaly detection.

**Nên nói:**

- Ba trụ cột có mục đích khác nhau — không thay thế nhau.
- Cardinality là yếu tố chi phí số một của metric; nói được cách kiểm soát.
- SLO alert theo burn-rate; alert theo triệu chứng.
- Có lộ trình theo giai đoạn để phù hợp ngân sách hạn chế.

**Không nên nói:**

- Đề xuất "thu mọi thứ" mà không nói chi phí/cardinality.
- Chỉ nói tên tool mà không có luồng dữ liệu và quyết định thiết kế.
- Đặt alert cho mọi metric và mọi service (tạo alert fatigue ngay từ đầu).

---

**7. 12/30 service dùng base image có CVE critical đang bị khai thác — kế hoạch 48 giờ.**

**Dàn ý trả lời:**

1. Giờ 0–4 (triage): xác định chính xác image/digest nào chứa CVE, service nào đang chạy production, mức độ phơi nhiễm (CVE yêu cầu điều kiện gì, đã có exploit công khai chưa). Công cụ: SBOM có sẵn (Syft/Trivy) + scan registry theo digest; ưu tiên theo tác động thực tế (internet-facing trước).
2. Giờ 4–24 (mitigation trước khi patch): áp biện pháp giảm thiểu — NetworkPolicy hạn chế egress/ingress liên quan, WAF rule, tắt tính năng bị ảnh hưởng, tăng giám sát (Falco rule) để phát hiện khai thác.
3. Giờ 24–48 (patch): build lại image với base image/library đã vá, test (unit + smoke trên staging), deploy theo canary/rolling; với service không patch được ngay, ghi nhận rủi ro và phương án chấp nhận có sign-off. Nếu nhà cung cấp chưa có patch: dùng rebuild với phiên bản gần nhất, hoặc thay library dependency; cân nhắc viết lại phần liên quan nếu khả thi.
4. Sau 48 giờ: tăng cường quy trình — scan tự động trên mọi PR (Trivy gate theo severity + có fix), Renovate/Dependabot bump base image định kỳ, registry mirror nội bộ với base image được duyệt, SBOM lưu mỗi build để lần sau trả lời triage trong vài phút, và có quy trình phản ứng CVE định kỳ (diễn tập).
5. Giao tiếp: thông báo stakeholder trạng thái theo mốc, tài liệu hóa quyết định chấp nhận rủi ro nếu có — minh bạch là phần của kế hoạch.

**Nên nói:**

- Ưu tiên theo mức phơi nhiễm thật (internet-facing, exploit tồn tại), không chỉ theo severity label.
- Mitigation trước, patch sau — và có giao tiếp rõ ràng.
- Tự động hóa lâu dài: SBOM, scan gate, bot bump dependency.
- Có bước test và canary trước khi deploy bản vá.

**Không nên nói:**

- "Rebuild tất cả và deploy ngay" không test, không kế hoạch rollback.
- Chỉ nói scan mà không nói ưu tiên/phơi nhiễm.
- Bỏ giao tiếp và tài liệu hóa quyết định chấp nhận rủi ro.

---

**8. DR 2 region cho e-commerce: PostgreSQL + Kafka, RTO 30 phút, RPO 5 phút.**

**Dàn ý trả lời:**

1. Làm rõ: SLO hiện tại, ngân sách, dữ liệu nguồn sự thật, phụ thuộc third-party (payment), người dùng phân bố ở đâu.
2. Chiến lược tổng: chọn **warm standby** (hoặc active-passive có scale-up) cho region thứ hai — vì RTO 30 phút không cần active-active, nhưng RPO 5 phút đòi hỏi replication liên tục.
3. Dữ liệu:
   - **PostgreSQL**: streaming replication (hoặc managed multi-AZ cross-region read replica), đo lag bằng `pg_stat_replication`; RPO 5 phút đạt được với replication bất đồng bộ trong điều kiện mạng tốt, cần giám sát lag và chấp nhận rủi ro mất tối đa vài phút dữ liệu.
   - **Kafka**: MirrorMaker 2 hoặc cluster linking giữa hai region; thứ tự và offset phải được kiểm tra khi failover; consumer phải resume đúng (idempotent).
4. Hạ tầng và traffic: mọi thứ dựng bằng Terraform để region DR có thể scale-up/hoàn thiện nhanh; global load balancer/DNS failover (Route 53 health check hoặc tương đương) với TTL thấp; certificate/domain chuẩn bị sẵn ở DR.
5. Quy trình failover: runbook chi tiết từng bước (freeze deploy, kiểm tra replication lag, promote database/replica, kiểm tra service, chuyển DNS, xác nhận), có người quyết định (incident commander), và kịch bản failback.
6. Test: diễn tập định kỳ (game day) ít nhất mỗi quý — failover thật ở môi trường staging tương đương, đo thời gian thực tế so với RTO; backup PostgreSQL test restore hàng tháng.
7. Rủi ro/trade-off cần nêu: chi phí gấp đôi gần như; split-brain nếu failover sai (cần fencing/quy trình rõ); replication lag tăng khi tải cao → RPO không đảm bảo, phải alert; third-party (payment) không nằm trong tầm kiểm soát.

**Nên nói:**

- Warm standby + replication là điểm cân bằng cho RTO 30 phút/RPO 5 phút; giải thích vì sao không chọn active-active.
- Số cụ thể: window 5 phút = phải theo dõi replication lag có alert.
- Diễn tập failover định kỳ; DR không test = chưa có DR.
- Nêu split-brain và cách phòng (fencing, quy trình promote một chiều).

**Không nên nói:**

- "Bật multi-region là xong" mà không nói replication dữ liệu và thứ tự failover.
- Hứa RPO = 0 với replication bất đồng bộ.
- Bỏ qua Kafka (queue) — đề bài có nhắc, thiếu sẽ bị trừ điểm.

---

**9. Chuẩn hóa upgrade 8 cluster Kubernetes, giảm 2–3 tuần và downtime.**

**Dàn ý trả lời:**

1. Nguyên tắc chung: không nhảy quá một minor version, upgrade theo thứ tự control plane → node → addon → workload; mọi cluster chạy cùng "golden path" version (N−1 hoặc N−2 so với mới nhất).
2. Chuẩn bị trước: checklist gồm version compatibility (Kubernetes, CNI, CSI, ingress, policy engine), đọc release notes và API deprecation (`kubectl deprecations`, `pluto`), scan manifest dùng API cũ, backup etcd, và test trên cluster dev trước.
3. Quy trình cho từng cluster (có thể chạy trên nhiều cluster song song nếu độc lập):
   - Dev trước, staging sau, production cuối cùng.
   - Control plane (managed tự lo; self-managed dùng `kubeadm upgrade`), chờ healthy.
   - Node: drain từng node (có PDB, `--ignore-daemonsets --delete-emptydir-data`), upgrade kubelet/kube-proxy, uncordon; với node pool có thể dùng surge upgrade (thay node mới) để giảm thời gian.
   - Addon: CNI/CSI/ingress/policy theo bảng tương thích, upgrade từng cái có kiểm tra.
   - Verify sau mỗi bước: node Ready, pod healthy, test workload, metric bình thường.
4. Giảm rủi ro: chạy upgrade trong cửa sổ có người trực, PDB và capacity dư để chịu node mất, rollback plan (với node có thể downgrade kubelet hoặc thay node; với control plane cần snapshot trước), thông báo cho team dùng cluster.
5. Giảm thời gian và drift: tự động hóa checklist bằng script/CI (ví dụ Ansible cho node upgrade, managed node group cho cloud), lịch upgrade định kỳ cố định (hàng tháng/quarterly theo release train), version matrix cho addon và ghi lại, tự động phát hiện cluster lệch version và báo cáo.

**Nên nói:**

- Thứ tự upgrade cố định và không nhảy minor.
- Kiểm tra API deprecation trước khi upgrade — nguyên nhân số một gây lỗi sau upgrade.
- PDB + capacity dư + drain từng node là cách tránh downtime workload.
- Tự động hóa + lịch định kỳ + version matrix để không lặp lại việc "dự án 2–3 tuần" cho mỗi lần.

**Không nên nói:**

- Upgrade cả cluster một lần hoặc upgrade production trước dev.
- Bỏ qua addon/CNI compatibility.
- Không có backup etcd và không có phương án rollback.

---

**10. Fintech chuyển từ VM lên Kubernetes on-prem (OpenShift/Rancher) — bạn làm gì trong 6 tháng đầu, đo bằng gì?**

**Dàn ý trả lời:**

1. Tháng 1 (assessment): inventory workload (service, stateful, phụ thuộc, mức độ quan trọng), đánh giá mức sẵn sàng container hóa, chọn platform (OpenShift hay Rancher + K8s) theo yêu cầu compliance và kỹ năng đội, xác định quick win.
2. Tháng 2 (nền tảng): dựng cluster chuẩn (theo CIS benchmark), CI/CD + registry nội bộ, observability (Prometheus/Grafana/Loki), baseline security (RBAC, PSA, Kyverno, secret với Vault), quy trình GitOps; pilot 2–3 service stateless.
3. Tháng 3–4 (mở rộng): migrate theo sóng, tạo template/golden path cho team, thiết lập on-call và runbook, SLO cho platform; bắt đầu huấn luyện dev viết manifest/Helm.
4. Tháng 5–6 (ổn định hóa): xử lý stateful (operator cho DB, storage), hardening, DR/backup (etcd, PV snapshot), tối ưu chi phí và hiệu năng, đo lường và cải tiến.
5. KPI đo thành công (chọn số cụ thể phù hợp bối cảnh):
   - **Delivery**: số service lên platform, thời gian deploy từ X xuống Y, tần suất release tăng.
   - **Reliability**: availability của platform, số incident do platform, MTTD/MTTR, tỉ lệ change failure.
   - **Security/compliance**: tỉ lệ workload đạt baseline, số finding critical còn lại, audit log đầy đủ.
   - **Hiệu quả**: mức độ tự động hóa (tỉ lệ deploy qua pipeline/GitOps), thời gian onboarding team mới, chi phí vận hành/đơn vị.
6. Cách báo cáo: dashboard theo KPI, review hàng tháng với stakeholder; điều chỉnh roadmap theo dữ liệu.

**Nên nói:**

- Assessment trước, xây nền tảng trước, migrate sau — không nhảy thẳng vào migrate.
- KPI hai mặt: delivery (tốc độ) và reliability/security (ổn định, an toàn) — cân bằng như đề bài fintech.
- Đào tạo và golden path để dev tự phục vụ — dấu hiệu của platform thành công.
- Đề cập compliance/audit vì bối cảnh fintech.

**Không nên nói:**

- Hứa migrate toàn bộ trong 6 tháng mà không có giai đoạn/ưu tiên.
- Chỉ nêu KPI kỹ thuật mà bỏ KPI kinh doanh/delivery.
- Bỏ qua yếu tố con người: đào tạo, on-call, thay đổi văn hóa vận hành.
