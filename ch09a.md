---
layout: answer

title: "Chương 9"
subtitle: "Minimize Microservice Vulnerabilities (CKS)"
exam_objectives:
  - "Áp dụng Pod Security Standards và admission control."
  - "Quản lý Kubernetes secrets an toàn."
  - "Hiện thực isolation: multi-tenancy, sandboxed containers."
  - "Mã hóa Pod-to-Pod (Cilium, Istio)."
---

## Đáp án {#answers}

**1. Đáp án đúng là A, B và C.**

**Giải thích:**

- **A)** `allowPrivilegeEscalation: false` là yêu cầu bắt buộc của restricted để chặn leo thang đặc quyền kiểu setuid/setgid. Đây là một trong những lý do pod bị từ chối phổ biến nhất.

- **B)** `capabilities.drop: ["ALL"]` là bắt buộc ở restricted. Khác với baseline (chỉ cấm thêm capabilities ngoài bộ mặc định), restricted yêu cầu drop **toàn bộ** capabilities; nếu cần, chỉ được add lại duy nhất `NET_BIND_SERVICE`.

- **C)** `runAsNonRoot: true` là bắt buộc; có thể đặt ở cấp pod (mọi container thừa hưởng) hoặc ở từng container. Thiếu cả hai nơi thì pod bị chặn với thông báo "runAsNonRoot != true".

- **D)** Đáp án này sai vì hai lý do. Thứ nhất, restricted chỉ **cấm** `privileged: true` chứ không bắt buộc khai báo `privileged: false` (giá trị mặc định đã là false). Thứ hai, `readOnlyRootFilesystem` là best practice nhưng **không nằm trong** bộ yêu cầu của PSS restricted — đây chính là điểm mà config audit/policy engine (Kyverno, Gatekeeper) mới enforce được.

**2. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. `audit` chỉ ghi annotation vi phạm vào audit event, hoàn toàn không chặn request; nó không tương đương enforce. Trong thi CKS, nhầm lẫn giữa ba mode này là lỗi rất dễ mất điểm.

- **B)** Đáp án này đúng. Vì không có label `enforce`, deployment/pod được tạo bình thường. Mode `warn` áp cho cả pod template nên khi apply Deployment bạn nhận warning trả về client ("would violate PodSecurity ..."), còn mode `audit` ghi annotation `pod-security.kubernetes.io/audit-violations` vào audit event của apiserver. Cả hai đều là cơ chế "đo" trước khi chuyển sang enforce.

- **C)** Sai. PSA chỉ chặn pod ở tầng admission khi namespace có `enforce`; ở đây không có, nên pod vẫn được tạo và chạy.

- **D)** Sai. Ngược lại: `warn` **có** áp cho workload resource (pod template), nên warning xuất hiện ngay khi apply Deployment. Nếu pod bị từ chối thì đó là do `enforce`, không phải `warn`.

**3. Đáp án đúng là C.**

**Giải thích:**

- **A)** Sai. PSA là admission controller chỉ đọc và quyết định allow/deny; nó **không mutate** pod. Muốn tự động thêm `securityContext`, bạn cần Kyverno/Gatekeeper (mutation) hoặc mutating webhook.

- **B)** Sai. PSA không có label exemption kiểu `pod-security.kubernetes.io/exempt`. Exemption chỉ cấu hình cluster-wide qua `AdmissionConfiguration` (usernames, runtimeClasses, namespaces) gắn với `--admission-control-config-file`.

- **C)** Đáp án này đúng. PSA chỉ làm đúng một việc: áp ba profile PSS cho pod theo label namespace. Nó không kiểm tra image registry/digest, không kiểm tra label bắt buộc của workload, không mutation, không policy report — đó là lý do các tổ chức dùng thêm Kyverno/Gatekeeper hoặc `ValidatingAdmissionPolicy` (CEL).

- **D)** Sai. PodSecurityPolicy đã bị xóa từ Kubernetes v1.25; PSA không đọc cấu hình PSP cũ. Khi migrate cần chuyển các ràng buộc PSP sang PSS/policy engine.

**4. Đáp án đúng là A, B và C.**

**Giải thích:**

- **A)** Đáp án này đúng. Gatekeeper dùng Rego: `ConstraintTemplate` định nghĩa loại luật và sinh CRD, `Constraint` là instance áp cho phạm vi cụ thể. Kyverno viết policy bằng YAML với cú pháp pattern/anchors (`=(...)`, `X(...)`), không cần học ngôn ngữ mới.

- **B)** Đáp án này đúng. Kyverno hỗ trợ đầy đủ bốn nhóm validate/mutate/generate/verifyImages; Gatekeeper chủ yếu là validating, mutation có tồn tại nhưng ít được dùng và không phải điểm mạnh.

- **C)** Đáp án này đúng. Gatekeeper có audit controller quét tài nguyên đang tồn tại và ghi vi phạm vào `status.violations` của Constraint; Kyverno ghi vào `PolicyReport`/`ClusterPolicyReport`. Cả hai đều cho phép chế độ Audit (không chặn) để theo dõi tuân thủ.

- **D)** Sai. Kyverno là admission webhook và có `failureAction: Enforce`: request vi phạm bị **deny** ngay ở admission, không chỉ ghi report. Trong lab của chương, pod vi phạm bị chặn với thông báo `admission webhook "validate.kyverno.svc-fail" denied the request`.

**5. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. Secret chỉ được base64-encode trong API; nếu không bật encryption at rest, giá trị nằm nguyên trong etcd và ai có quyền đọc etcd (hoặc backup etcd) đều đọc được. Lab của chương minh họa điều này bằng `etcdctl get /registry/secrets/...`.

- **B)** Đáp án này đúng. Env bị "chụp" vào tiến trình lúc tạo pod, có thể lộ qua `/proc/<pid>/environ`, crash dump, log debug hoặc kế thừa cho tiến trình con. Volume mount dùng tmpfs, kiểm soát được quyền file (`defaultMode`, `items`) và không xuất hiện trong môi trường tiến trình.

- **C)** Sai vì ngược lại: env từ Secret **không** cập nhật khi Secret thay đổi (giá trị cố định từ lúc tạo pod), còn volume mount được kubelet sync định kỳ và ứng dụng có thể đọc lại (nếu app tự reload file).

- **D)** Đáp án này đúng. `get`/`list`/`watch` trên Secret là quyền nhạy cảm, cần cấp tối thiểu theo namespace và tránh wildcard. `automountServiceAccountToken: false` giảm bề mặt khi pod không cần gọi Kubernetes API.

**6. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai ở thứ tự provider. Provider **đầu tiên** trong danh sách là provider dùng để **ghi**; nếu `identity` đứng trước, mọi Secret mới vẫn được ghi dạng plaintext dù aescbc có mặt phía sau. Khi đọc, apiserver thử lần lượt các provider nên `identity` phía sau chỉ giúp đọc dữ liệu cũ.

- **B)** Đáp án này đúng. Đặt `aescbc` trước để mọi ghi mới được mã hóa, `identity` phía sau để apiserver vẫn đọc được Secret cũ chưa mã hóa trong giai đoạn chuyển tiếp. Vì mã hóa chỉ áp cho lần ghi mới, Secret tồn tại trước đó vẫn plaintext trong etcd cho tới khi được ghi lại — cách phổ biến là `kubectl get secrets -A -o json | kubectl replace -f -`.

- **C)** Sai. Không có flag `--encrypt-all-resources` trên kube-apiserver. Mã hóa at rest cấu hình qua `EncryptionConfiguration` (flag `--encryption-provider-config`), trong đó liệt kê resource cần mã hóa (thường là `secrets`, có thể thêm `configmaps`).

- **D)** Sai. Kubelet không ghi Secret vào etcd; kube-apiserver mới là thành phần ghi/đọc etcd. Sửa `KubeletConfiguration` không liên quan đến mã hóa at rest.

**7. Đáp án đúng là A, B và C.**

**Giải thích:**

- **A)** Đáp án này đúng. RuntimeClass (`node.k8s.io/v1`) ánh xạ tên runtime class tới `handler` đã cấu hình trong CRI (containerd) và runtime binary phải được cài trên node; pod chọn runtime bằng `runtimeClassName`. Nếu RuntimeClass không tồn tại, pod bị từ chối ngay ở admission; nếu handler không có trên node, pod không chạy được.

- **B)** Đáp án này đúng. gVisor (`runsc`) chặn và tự xử lý syscall trong user space nên giảm mạnh rủi ro kernel host bị khai thác, nhưng có thể không tương thích một số ứng dụng (I/O, syscall đặc biệt) và chậm hơn runc. Kata Containers chạy pod sandbox trong VM nhẹ với kernel riêng, cần nested virtualization/KVM, cô lập mạnh nhưng overhead và thời gian khởi động cao hơn.

- **C)** Đáp án này đúng. `scheduling.nodeSelector` ghim pod vào node có handler (ví dụ node pool sandbox) và `overhead.podFixed` khai báo tài nguyên tăng thêm để scheduler tính đúng chỗ đặt pod.

- **D)** Sai. Sandboxed containers giải quyết cô lập **kernel/runtime**, không thay thế NetworkPolicy — chúng không giới hạn luồng mạng giữa các pod. Vẫn cần NetworkPolicy (và policy engine) cho ranh giới dữ liệu giữa tenant.

**8. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** Đáp án này đúng. Cilium WireGuard thiết lập tunnel giữa các node (mỗi node một key pair, public key trao đổi qua annotation `network.cilium.io/wg-pub-key` trên `CiliumNode`), dùng UDP **51871**. Chỉ traffic giữa pod ở **các node khác nhau** mới đi qua tunnel; pod-to-pod cùng node không có gói tin trên dây nên không được mã hóa.

- **B)** Đáp án này đúng. Istio cấp danh tính theo ServiceAccount dưới dạng SPIFFE ID (`spiffe://<trust-domain>/ns/<ns>/sa/<sa>`); `istiod` đóng vai trò CA nội bộ, cấp và tự động xoay certificate ngắn hạn cho sidecar/ztunnel.

- **C)** Sai. Cả hai đều trong suốt với ứng dụng. WireGuard/CNI mã hóa ở tầng mạng (không cần TLS trong app), còn Istio mTLS tự chèn TLS giữa các proxy mà app không phải cấu hình gì.

- **D)** Sai. mTLS trả lời câu hỏi "ai đang gọi" (xác thực danh tính) nhưng không quyết định "được gọi tới đâu". Muốn giới hạn luồng vẫn cần `AuthorizationPolicy` (Istio) hoặc NetworkPolicy — nguyên tắc defense in depth.

**9. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. Policy này là `validate`, không phải `mutate`; Kyverno không sửa pod. Ngoài ra không có khái niệm "hostPath bị vô hiệu hóa" trong pod spec.

- **B)** Đáp án này đúng. Anchor `=(volumes)` là conditional: chỉ khi pod có `volumes` thì Kyverno mới kiểm tra các phần tử; `X(hostPath): "null"` là negation anchor, có nghĩa field `hostPath` **không được phép tồn tại** trong bất kỳ volume nào. Pod không khai báo `volumes` (hoặc `volumes` rỗng) vẫn hợp lệ, còn pod có hostPath bị deny với `failureAction: Enforce`.

- **C)** Sai. `background: true` chỉ bật quét tài nguyên đang tồn tại để tạo report; nó không chuyển policy sang chế độ audit. Chế độ được quyết định bởi `failureAction` (`Enforce` trong ví dụ).

- **D)** Sai. Pattern không yêu cầu `volumes` phải rỗng; nó chỉ ràng buộc rằng nếu volume tồn tại thì không được chứa `hostPath`. Các volume hợp lệ như `configMap`, `secret`, `emptyDir` vẫn qua bình thường.

**10. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. Bỏ enforce không kèm biện pháp nào là mở toang namespace cho mọi pod privileged/hostPath; đây là cách nhanh nhất để tạo lỗ hổng leo thang toàn cluster nếu pod trong namespace đó bị chiếm. Ngoài ra `kube-system` không phải chỗ nên đặt DaemonSet ứng dụng.

- **B)** Đáp án này đúng. Workload hạ tầng cần đặc quyền nên có "trusted zone" riêng: namespace chuyên biệt áp `enforce=privileged` nhưng vẫn bật `audit`/`warn=restricted` để nhìn thấy rủi ro, RBAC chỉ cho platform team, NetworkPolicy chặt, policy engine giới hạn image/field cụ thể, và quy trình review định kỳ. Đây là cách cân bằng giữa yêu cầu chức năng và bảo mật, thay vì hạ chuẩn toàn cluster.

- **C)** Sai. PSA **không** hỗ trợ exemption theo namespace bằng label `pod-security.kubernetes.io/exempt`; label này không tồn tại và sẽ chỉ nằm đó vô tác dụng (trong khi pod vẫn bị enforce chặn). Exemption thật chỉ ở `AdmissionConfiguration` cấp cluster.

- **D)** Sai. PSA là admission controller được enforce thật (không phải "khuyến nghị") và không thể tắt bằng annotation trên namespace hay pod. Nếu DaemonSet cần privileged, phải thiết kế namespace/profile phù hợp như phương án B.
