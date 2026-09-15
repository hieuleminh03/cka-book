---
layout: answer

title: "Chương 11"
subtitle: "Monitoring, Logging and Runtime Security (CKS)"
exam_objectives:
  - "Thực hiện behavioral analytics để phát hiện hành vi độc hại."
  - "Phát hiện mối đe dọa trong hạ tầng, ứng dụng, mạng, dữ liệu, người dùng và workload."
  - "Điều tra và nhận diện các pha tấn công và bad actor trong môi trường."
  - "Đảm bảo immutability của container runtime và dùng Kubernetes audit logs để giám sát truy cập."
---

## Đáp án {#answers}

**1. Đáp án đúng là A, C và D.**

**Giải thích:**

- **A)** Đáp án này đúng. Kiến trúc Falco tách đôi trách nhiệm: driver (kmod hoặc eBPF) gắn vào các syscall như `execve`, `open`, `connect` để thu event và đẩy qua ring buffer/BPF map lên userspace; rule engine nằm ở userspace mới là nơi đánh giá `condition` và sinh alert.

- **B)** Đáp án này sai. Rule engine ở userspace nghĩa là Falco **không chặn** được syscall — đến lúc rule được đánh giá thì hành vi đã xảy ra. Đây là lý do Falco được mô tả là detect-only; muốn enforcement phải dùng Tetragon, AppArmor/seccomp hoặc response engine ngoài.

- **C)** Đáp án này đúng. Nhờ container metadata collector (container runtime socket, plugin `k8smeta` + `k8s-metacollector` từ Falco 0.37+), alert mới hiển thị pod name, namespace, image thay vì chỉ `container.id`. Không có enrichment thì alert vẫn fire nhưng thiếu ngữ cảnh điều tra.

- **D)** Đáp án này đúng. Driver được chọn theo khả năng kernel: `modern_ebpf` ưu tiên khi có BTF, fallback `kmod`. Khi môi trường không cho load module và thiếu BTF (một số kernel WSL2/managed), Falco có thể chạy dạng plugin như `k8saudit` để phân tích audit log — phạm vi phát hiện hẹp hơn nhưng vẫn hữu ích.

**2. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai ở hai điểm: `ls` không phải shell (không nằm trong `shell_procs`), và exec không có `-t` nên `proc.tty == 0`. Rule yêu cầu `proc.tty != 0`, do đó không fire.

- **B)** Đáp án này đúng. `sh` nằm trong `shell_procs`, process chạy trong container, và `-t` cấp TTY nên `proc.tty != 0`. Điều kiện `container_entrypoint` cũng thỏa vì process cha của shell khi exec là runtime shim (`containerd-shim`/`runc`). Đây là kịch bản chuẩn để demo rule.

- **C)** Sai vì `kubectl logs` chỉ đọc log của container, không tạo process nào trong container; không có `spawned_process` nên không có gì để rule đánh giá.

- **D)** Đây là lựa chọn "gần đúng" gây nhầm lẫn: `sh` có được spawn, nhưng vì không có `-t` nên `proc.tty == 0` và rule `Terminal shell in container` hầu như không fire. Rule `Run shell untrusted` (rule khác) có thể vẫn fire — nhưng câu hỏi hỏi về đúng rule này, nên D không phải đáp án.

**3. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. Rule `Detect crypto miners using the Stratum protocol` kiểm tra `proc.cmdline contains "stratum+tcp"` (và các biến thể `stratum2+tcp`, `stratum+ssl`, `stratum2+ssl`). Vì dựa trên chuỗi command line nên rule rất dễ bị bypass bằng cách đổi tên tham số, dùng biến, hoặc wrapper script — bản thân `desc` của rule cũng thừa nhận điều này.

- **B)** Đáp án này đúng. Rule `Detect outbound connections to common miner pool ports` dùng macro `net_miner_pool` với các list `miner_ports` (3333, 4444, 5555...), `miner_domains`, `https_miner_domains`, `http_miner_domains` và đặt `enabled: false` mặc định vì Falco phải chủ động resolve domain của miner, có thể sinh lưu lượng DNS đáng ngờ trong môi trường được giám sát chặt.

- **C)** Đáp án này sai. Chart Falco mặc định chỉ cài `[falco-rules:5]`; các rule mining nằm trong bộ **sandbox rules** nên không được load mặc định. Muốn dùng phải bổ sung refs `falco-sandbox-rules` hoặc tự viết rule tương tự trong `customRules`.

- **D)** Đáp án này đúng. Tín hiệu mining thực tế là đa nguồn: process/binary lạ trong container (Falco), kết nối ra port/domain lạ (Falco/flow log), CPU tăng vọt kéo dài (Prometheus). Không nguồn đơn lẻ nào đủ để kết luận chắc chắn, và đây cũng là ví dụ điển hình của việc cần correlate alert trước khi kết luận.

**4. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai và nguy hiểm. Xóa pod ngay lập tức phá hủy bằng chứng quan trọng nhất: filesystem đang chạy, process list, trạng thái kết nối. Nó cũng không cho biết kẻ tấn công đã lấy gì, và nếu pod do controller quản lý thì nó sẽ được tạo lại ngay.

- **B)** Đáp án này đúng. Quy trình điều tra hợp lý: (1) xác minh alert và lấy ngữ cảnh pod/node/image; (2) tra audit log event `pods/exec` của đúng pod để biết `user.username`, `sourceIPs`, `userAgent`, thời điểm; (3) đối chiếu với change request/on-call/automation được phép; (4) nếu bất thường thì thu thập bằng chứng (không xóa) rồi mới containment. Falco trả lời "container nào", audit log trả lời "ai" — hai nguồn phải được tương quan.

- **C)** Sai vì tắt rule là che giấu sự việc thay vì điều tra. Alert đêm khuya không đáng ngờ hơn alert giờ làm việc — thậm chí tấn công thường chọn thời điểm vắng người. Nếu rule nhiễu, xử lý đúng là whitelist/tuning có kiểm soát và ghi lại lý do.

- **D)** Sai vì restart node không chỉ phá bằng chứng mà còn gây gián đoạn dịch vụ diện rộng, và không loại trừ được nguyên nhân (kẻ tấn công có thể quay lại qua credential đã lấy được). Với node bị xâm phạm, hướng đúng là cách ly và **thay node mới** sau khi thu thập bằng chứng.

**5. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. `Metadata` ghi thông tin mô tả request: user/group, `verb`, resource, `objectRef` (kể cả tên object), `sourceIPs`, `userAgent`, `responseStatus` — nhưng **không** ghi request body hay response body. Đây là mức cân bằng phù hợp cho hầu hết tài nguyên và cho secrets.

- **B)** Đáp án này đúng. `RequestResponse` ghi cả request body lẫn response body, nên khi áp cho secrets, giá trị secret (base64) sẽ nằm trong file audit log — biến log thành nơi chứa credential và vi phạm nguyên tắc bảo vệ dữ liệu nhạy cảm. Đây là lỗi cấu hình audit policy nghiêm trọng hay được hỏi trong CKS.

- **C)** Đáp án này sai. Chỉ `ResponseComplete` (và `Panic`) là hành vi mặc định; bạn hoàn toàn có thể (và thường nên) đặt `omitStages: [RequestReceived]` để giảm khối lượng log. Kubernetes cũng cho phép omit các stage khác qua audit policy.

- **D)** Đáp án này đúng. Audit policy là danh sách rule được đánh giá **từ trên xuống**, rule đầu tiên match sẽ quyết định level cho request đó. Vì vậy phải đặt rule cụ thể (RBAC, secrets) lên trên và catch-all `level: Metadata` ở cuối.

**6. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đáp án này đúng. Trên cluster kubeadm/kind, kube-apiserver là static pod nên audit được bật bằng cách: tạo audit policy file trên node, thêm `--audit-policy-file` và `--audit-log-path` (kèm rotation flags nếu cần) vào command của container, thêm `volumeMounts` cho policy (read-only) và thư mục log, thêm `volumes` hostPath tương ứng, rồi để kubelet tự tạo lại static pod khi thấy manifest thay đổi. Verify bằng cách kiểm tra flag, file log và event mới.

- **B)** Sai. `--audit-log-path` chỉ định nơi ghi log; nếu thiếu `--audit-policy-file` thì Kubernetes không có policy để quyết định ghi gì — audit không hoạt động như mong đợi. Policy không có "mặc định" ghi tất cả.

- **C)** Sai. Webhook là **một backend tùy chọn** để gửi audit event ra ngoài (`--audit-webhook-config-file`). Ghi ra file với `--audit-log-path` hoàn toàn độc lập và không cần webhook.

- **D)** Sai vì audit không phải admission plugin; `--enable-admission-plugins=Audit` không tồn tại. Admission plugin xử lý việc cho phép/từ chối object; audit là cơ chế ghi log request ở apiserver.

**7. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai và đi ngược mục tiêu bảo mật. `privileged: true` cấp toàn bộ quyền cho container (bao gồm ghi vào nhiều thứ trên node), biến một cấu hình immutability thành lỗ hổng nghiêm trọng. Nginx không cần privileged để chạy.

- **B)** Đáp án này đúng. Vấn đề chỉ là nginx cần ghi ở vài path cố định (`/var/cache/nginx` cho temp/cache, `/var/run` cho pid, có thể `/tmp`). Gắn `emptyDir` cho các path này giữ rootfs read-only mà app vẫn chạy, đúng tinh thần "immutable trừ allow-list".

- **C)** Sai vì bỏ `readOnlyRootFilesystem` là bỏ luôn biện pháp phòng ngừa persistence quan trọng — kẻ tấn công có thể ghi webshell, cron, binary vào container. Đây là lựa chọn phổ biến nhưng sai trong bối cảnh CKS.

- **D)** Sai. Mount `hostPath` từ node vào container khiến pod phụ thuộc node cụ thể, dữ liệu cache bị chia sẻ nhầm giữa các pod, và quan trọng hơn là mở thêm bề mặt tấn công (container có đường ghi ra node). Với path tạm, `emptyDir` là lựa chọn đúng.

**8. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Đáp án này đúng. `kubectl debug --target=web` tạo ephemeral container và cho nó join process namespace của container `web`, nhờ đó bạn thấy process của container đích (`ps`, đọc `/proc`) — đây là cách debug tại chỗ mà không cần `exec` và không phải restart pod.

- **B)** Đáp án này sai. Ephemeral container **không thể xóa** hoặc restart sau khi được thêm vào pod (chúng chỉ biến mất khi pod bị xóa), và không được quản lý bởi spec của controller. Chính vì tính "một chiều" này mà cần cân nhắc khi nào dùng và có nên chặn subresource `pods/ephemeralcontainers` hay không.

- **C)** Đáp án này đúng. Vì tool debug nằm trong image ephemeral (được kéo riêng), image production có thể giữ tối giản — không cần `curl`, `tcpdump`, `strace` trong image chính. Đây là lợi ích kép: giảm bề mặt supply chain (chương 10) và vẫn debug được.

- **D)** Đáp án này sai. Ephemeral container không hỗ trợ `ports`, `resources`, `lifecycle` hay probe như container thường — giới hạn này có chủ đích để tránh bị lạm dụng như container "hạng nhất".

**9. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. Cả hai đều làm việc ở tầng kernel (syscall), không đọc audit log của kube-apiserver. Việc đọc audit log (nếu cần) là chức năng của plugin `k8saudit` của Falco hoặc backend SIEM, không phải cơ chế phát hiện chính của hai công cụ này.

- **B)** Đáp án này đúng. Đây là khác biệt cốt lõi: Tetragon thực thi action **trong kernel** (`Sigkill`, `Signal`, `Override` kết quả syscall, `NotifyEnforcer`) theo selector trong `TracingPolicy`/`TracingPolicyNamespaced`; Falco chủ yếu phát hiện và chuyển alert, muốn phản ứng phải nối thêm Falcosidekick/webhook/automation.

- **C)** Sai vì ngược lại hoàn toàn: Falco cấu hình bằng YAML rules (condition/macro/list), còn Tetragon cấu hình bằng CRD TracingPolicy.

- **D)** Sai vì Falco cần kernel driver để thu syscall. Falco chỉ có thể chạy không driver khi dùng chế độ plugin (ví dụ `k8saudit`), và khi đó phạm vi phát hiện hẹp hơn nhiều — không phải cách hoạt động mặc định.

**10. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. `create clusterrolebindings` gắn một serviceaccount (đặc biệt là SA mới) vào `cluster-admin` là dấu hiệu leo thang đặc quyền kinh điển (liên quan T1078/T1098). Cần alert kèm `user.username`, `sourceIPs`, `userAgent`, và kiểm tra các hành động khác của cùng danh tính.

- **B)** Đáp án này đúng. Pod với `privileged: true` và `hostPath: /` là cấu hình escape-to-host (T1611) điển hình — kết hợp cả quyền kernel lẫn khả năng đọc/ghi filesystem node. Trên cluster đã hardening, loại pod này thường bị chặn từ admission (PSA restricted); nếu vẫn tạo được, đó là sự kiện phải điều tra ngay.

- **C)** Sai. `watch pods` bởi `system:kube-controller-manager` là hoạt động bình thường của control plane — controller manager cần watch pod để điều khiển replicaset/job. Alert theo tiêu chí này sẽ tạo false positive khổng lồ; muốn lọc phải dựa trên danh tính bất thường, không phải hành vi bình thường của hệ thống.

- **D)** Đáp án này đúng. `pods/exec` vào pod trong `kube-system` từ IP ngoài cluster, lại dùng `userAgent` là `curl/8.x` (không phải `kubectl`) là tổ hợp rất bất thường. Đây là ví dụ phải correlate nhiều field: subresource + namespace đích + nguồn IP + userAgent mới đủ kết luận.
