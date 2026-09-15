---
layout: answer

title: "Chương 4"
subtitle: "Kubernetes căn bản (CKA)"
exam_objectives:
  - "Hiểu kiến trúc cluster Kubernetes: control plane, node, etcd, kubelet, kube-proxy và luồng điều khiển."
  - "Làm việc thành thạo với Pod, Deployment, StatefulSet, DaemonSet, Job và CronJob."
  - "Cấu hình Service, Ingress, ConfigMap, Secret và storage (StorageClass, PV, PVC)."
  - "Sử dụng kubectl hiệu quả: context, namespace, label/selector, rollout và debugging."
---

## Đáp án {#answers}

**1. Đáp án đúng là C.**

**Giải thích:**

- **A)** Sai. `kubectl logs` đọc log của container đang chạy; Pod `Pending` chưa được schedule nên chưa có container nào, vì vậy không có log để xem.
- **B)** Sai. `kubectl exec` yêu cầu container đã khởi động; với Pod `Pending`, lệnh sẽ báo lỗi không có container nào đang chạy.
- **C)** Đúng. Sự kiện `Insufficient cpu` nghĩa là scheduler đã lọc hết node vì tổng CPU `requests` vượt `Allocatable`. Hãy chạy `kubectl describe node` để so sánh, rồi giảm requests của Pod, dọn workload không cần thiết hoặc thêm node. Đây là nguyên nhân phổ biến nhất khiến Pod `Pending`.
- **D)** Sai. `restartPolicy` chỉ chi phối việc kubelet restart container sau khi Pod đã được schedule; nó không liên quan tới bài toán thiếu CPU của scheduler.

**2. Đáp án đúng là B, C và D.**

**Giải thích:**

- **A)** Sai. Mặc định Deployment dùng `RollingUpdate`; `Recreate` phải chỉ định tường minh và gây downtime vì xóa toàn bộ Pod cũ trước khi tạo Pod mới.
- **B)** Đúng. Mặc định `maxSurge: 25%` và `maxUnavailable: 25%`, có thể chỉnh trong `spec.strategy.rollingUpdate` (ví dụ đặt `maxUnavailable: 0` để update không làm giảm số Pod ready).
- **C)** Đúng. `kubectl rollout undo deployment/web` không kèm `--to-revision` sẽ quay về revision liền trước; Deployment tạo revision mới trỏ về pod template cũ chứ không "quay ngược thời gian" các ReplicaSet.
- **D)** Đúng. `kubectl rollout restart` cập nhật annotation `kubectl.kubernetes.io/restartedAt` trong pod template, làm phát sinh revision mới và kích hoạt thay thế Pod lần lượt theo chiến lược rolling update.

**3. Đáp án đúng là A, C và D.**

**Giải thích:**

- **A)** Đúng. ClusterIP là VIP ảo chỉ định tuyến được từ bên trong cluster. Muốn truy cập từ ngoài phải dùng NodePort, LoadBalancer, Ingress hoặc `kubectl port-forward`.
- **B)** Sai. NodePort chỉ mở một port cố định trên mọi node; nó không tạo load balancer và không cấp external IP. Việc cấp external IP là chức năng của LoadBalancer thông qua cloud-controller-manager (hoặc MetalLB).
- **C)** Đúng. LoadBalancer mở rộng NodePort và thêm external load balancer. Trên kind/minikube không có thành phần cấp LB, `EXTERNAL-IP` sẽ ở trạng thái `Pending`.
- **D)** Đúng. Headless Service (`clusterIP: None`) không có VIP; DNS trả về A record cho từng Pod ready. StatefulSet dùng cơ chế này để mỗi Pod có DNS ổn định dạng `<pod-name>.<service>.<namespace>.svc.cluster.local`.

**4. Đáp án đúng là C.**

**Giải thích:**

- **A)** Sai. Env var không có cơ chế đồng bộ lại như volume mount; kubelet không ghi lại giá trị vào tiến trình đang chạy.
- **B)** Sai. Xóa rồi tạo lại ConfigMap cũng không làm container đang chạy đọc lại env — biến môi trường đã được resolve từ lúc start.
- **C)** Đúng. Env var chỉ được nạp một lần khi container khởi động, nên phải `kubectl rollout restart deployment/api` (hoặc xóa Pod) để Pod mới nhận giá trị cập nhật.
- **D)** Sai. ConfigMap hỗ trợ đầy đủ `envFrom`, `configMapKeyRef` và volume mount; không có chuyện phải chuyển sang Secret vì lý do này.

**5. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. Với dynamic provisioning, PV được provisioner tạo **sau** khi PVC xuất hiện; không thấy PV sẵn không chứng minh được điều gì.
- **B)** Đúng. `kubectl describe pvc data-pvc` hiển thị Events (`storageclass not found`, provisioning failed, đang chờ Pod...) và `kubectl get storageclass` xác nhận class tồn tại, provisioner sẵn sàng cũng như `volumeBindingMode`. Nếu là `WaitForFirstConsumer`, PVC `Pending` khi chưa có Pod là hành vi bình thường.
- **C)** Sai. Hết dung lượng đĩa trên node không phải nguyên nhân điển hình làm PVC `Pending` (nhất là với provisioner mạng như EBS), và `kubectl top nodes` còn cần metrics-server.
- **D)** Sai. kind dùng local-path-provisioner hỗ trợ RWO; đổi sang RWX mới là hướng đi sai vì local-path không hỗ trợ RWX.

**6. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đúng. Với `podManagementPolicy: OrderedReady` (mặc định), Pod được tạo tuần tự `0 → N-1`, mỗi Pod phải Ready trước khi Pod kế tiếp được tạo; khi xóa thì theo thứ tự ngược lại.
- **B)** Đúng. Headless Service cấp network identity ổn định, và PVC sinh từ `volumeClaimTemplates` vẫn tồn tại khi Pod bị xóa — đây là chủ ý để không mất dữ liệu.
- **C)** Sai. StatefulSet không xóa Pod ngẫu nhiên; nó đảm bảo thứ tự và identity, trái ngược hoàn toàn với mô tả này.
- **D)** Đúng. `kubectl scale statefulset db --replicas=5` sẽ tạo thêm `db-2`, `db-3`, `db-4` theo thứ tự (và scale down sẽ xóa từ ordinal cao nhất, giữ nguyên PVC).

**7. Đáp án đúng là D.**

**Giải thích:**

- **A)** Sai. Ingress không yêu cầu backend phải là Service LoadBalancer; ClusterIP là backend hoàn toàn bình thường vì controller truy cập Service qua DNS/IP nội bộ cluster.
- **B)** Sai. Annotation `rewrite-target` chỉ đổi đường dẫn request; nó không liên quan tới việc cột `ADDRESS` trống — dấu hiệu cho thấy không có controller nào xử lý Ingress.
- **C)** Sai. NetworkPolicy chỉ được tạo khi bạn khai báo và sẽ chặn theo cách khác (timeout, connection refused); đây không phải giả thuyết cần kiểm tra trước tiên.
- **D)** Đúng. Cần xác nhận có IngressClass phù hợp với `spec.ingressClassName`, Pod của controller chạy khỏe trong namespace `ingress-nginx`, và đọc Events/`describe ingress` để xem controller có nhận Ingress hay không. 404 với `ADDRESS` trống thường là chưa có controller hoặc sai class.

**8. Đáp án đúng là A, B và C.**

**Giải thích:**

- **A)** Đúng. Init container chạy tuần tự, từng cái một, và phải thành công hết trước khi container chính start. Nếu fail, kubelet chạy lại theo `restartPolicy` (`Always`/`OnFailure`); với `Never`, Pod bị đánh dấu `Failed`.
- **B)** Đúng. Khi có `startupProbe`, kubelet tắt liveness và readiness cho tới khi startup thành công, nhờ đó app khởi động chậm không bị liveness kill oan. Ngân sách chờ là `failureThreshold × periodSeconds`.
- **C)** Đúng. readinessProbe fail làm Pod bị loại khỏi EndpointSlice (không nhận traffic) nhưng container vẫn chạy, không bị restart.
- **D)** Sai. livenessProbe fail khiến kubelet **restart container tại chỗ** (cùng Pod, cùng node) theo restartPolicy, không xóa Pod và không schedule lại sang node khác.

**9. Đáp án đúng là A, C và D.**

**Giải thích:**

- **A)** Đúng. `parallelism: 2` giới hạn tối đa 2 Pod chạy đồng thời; `completions: 6` nghĩa là Job hoàn thành khi đạt 6 lần chạy thành công.
- **B)** Sai. `backoffLimit` là field của Job (mặc định 6) — nó giới hạn số lần retry trước khi Job bị đánh dấu `Failed`. CronJob không có field này ở cấp của nó, chỉ có `jobTemplate` chứa cấu hình Job.
- **C)** Đúng. Vượt quá `backoffLimit: 3`, Job bị đánh dấu `Failed` và controller ngừng tạo thêm Pod mới.
- **D)** Đúng. `ttlSecondsAfterFinished` tự động dọn Job và các Pod liên quan sau khi Job hoàn thành (hoặc bị đánh dấu Failed), giúp tránh tích tụ object rác.

**10. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. Xóa Deployment gây downtime và mất revision history; đây là cách thô bạo không cần thiết khi bạn còn nguyên các Pod cũ đang phục vụ.
- **B)** Đúng. `kubectl rollout undo deployment/web` đưa Deployment về revision tốt gần nhất; nhờ `maxUnavailable: 0`, các Pod cũ vẫn phục vụ trong lúc sự cố, nên Service không bị gián đoạn. Sau đó sửa tag image đúng và triển khai lại.
- **C)** Sai. `ImagePullBackOff` là lỗi kéo image (sai tag, thiếu registry credential, network tới registry...), không liên quan tới kubelet của node bị hỏng.
- **D)** Sai. `progressDeadlineSeconds` chỉ quy định bao lâu thì Deployment bị đánh dấu `ProgressDeadlineExceeded`; tăng nó không sửa được nguyên nhân (tag image không tồn tại).

