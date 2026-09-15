---
layout: answer

title: "Chương 8"
subtitle: "System Hardening (CKS)"
exam_objectives:
  - "Giảm host OS footprint (attack surface) của node."
  - "Áp dụng least-privilege identity và access management."
  - "Giảm truy cập mạng từ bên ngoài."
  - "Dùng kernel hardening tools như AppArmor và seccomp."
---

## Đáp án {#answers}

**1. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Đáp án này đúng. Mask các service thừa và purge package không cần (đặc biệt trình biên dịch) là cách giảm attack surface trực tiếp: ít process chạy bằng quyền cao hơn, ít công cụ cho kẻ tấn công sau khi chiếm được shell. Đây là bước "kiểm kê → loại bỏ" của quy trình footprint reduction.

- **B)** Đáp án này sai. Đổi port SSH chỉ giảm log scan tự động, không giảm attack surface thật: service vẫn lộ ra Internet và kẻ tấn công có kỹ thuật scan theo port. Trên node production, cách đúng là không expose SSH ra Internet mà đi qua bastion/VPN hoặc SSM Session Manager.

- **C)** Đáp án này đúng. Node image tối giản/immutable giới hạn những gì tồn tại trên node ở mức image — không có package manager tương tác, filesystem gốc read-only, thay đổi phải qua image mới. Đây là mức triệt để nhất của footprint reduction, đồng thời giảm drift cấu hình giữa các node.

- **D)** Đáp án này sai. `tcpdump`, `nmap`, `python3` trên node production là công cụ tấn công tiềm năng khi node bị chiếm; nhu cầu debug không biện minh cho việc tăng footprint trên mọi node. Cách thay thế: dùng pod debug (netshoot) gắn tạm thời, thu thập log tập trung, và cài công cụ chẩn đoán theo yêu cầu có thời hạn.

**2. Đáp án đúng là A, B và C.**

**Giải thích:**

- **A)** Đáp án này đúng. `/etc/kubernetes/admin.conf` là kubeconfig có quyền cluster-admin. Quyền `644` cho phép mọi user trên node đọc được credential này — chỉ cần một tài khoản có shell là toàn bộ cluster bị chiếm. Tiêu chuẩn là `600` với chủ sở hữu `root:root` (CIS Kubernetes Benchmark kiểm tra đúng mục này).

- **B)** Đáp án này đúng. User trong group `docker` tương đương quyền root trên host: họ có thể chạy container với mount filesystem của node. Trên node chỉ dùng containerd qua CRI, group `docker` thường không cần tồn tại; nếu node chạy Docker daemon song song thì bản thân việc đó đã là vấn đề về footprint.

- **C)** Đáp án này đúng. `NOPASSWD: ALL` cho phép user `ops` chạy mọi lệnh bằng root không cần mật khẩu — biến một tài khoản bị chiếm thành root ngay lập tức. Least privilege yêu cầu giới hạn cả lệnh cụ thể (ví dụ chỉ `systemctl restart kubelet`, `journalctl -u kubelet`) thay vì cấp toàn quyền.

- **D)** Đáp án này sai. `runAsUser: 0` nghĩa là container chạy bằng root — vi phạm trực tiếp nguyên tắc least privilege, bất kể `readOnlyRootFilesystem` có bật hay không (read-only chỉ chặn ghi file, không giảm đặc quyền của process). Hai trường này độc lập; muốn least privilege phải set user non-root.

**3. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đáp án này đúng. Profile seccomp tùy biến nằm trên node trong seccomp root của kubelet (mặc định `/var/lib/kubelet/seccomp/`); pod khai `type: Localhost` cùng `localhostProfile` là đường dẫn tương đối so với seccomp root. Cần nhớ profile phải tồn tại trên mọi node mà pod có thể chạy tới.

- **B)** Đáp án này sai. `securityContext.seccompProfile` không có trường `profile` để nhúng nội dung JSON. Schema chỉ gồm `type` và `localhostProfile` (đường dẫn). Cố nhúng JSON vào spec sẽ bị API server từ chối (unknown field).

- **C)** Đáp án này sai ở hai điểm: kubelet không quét `/etc/seccomp/` (seccomp root mặc định là `/var/lib/kubelet/seccomp/`), và `type: RuntimeDefault` không liên quan tới file của bạn — nó dùng profile mặc định của container runtime, file trong `/etc/seccomp/` sẽ không bao giờ được áp.

- **D)** Đáp án này sai. Kubelet đọc profile từ filesystem của **node**, không phải root filesystem của container. Đặt file trong image rồi khai `Localhost` sẽ khiến pod fail với lỗi không tìm thấy profile.

**4. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Đáp án này đúng. File trong `/etc/sysctl.d/` là cách cấu hình bền vững qua reboot: `sysctl --system` nạp toàn bộ các file theo thứ tự, và hệ thống tự nạp lại khi boot. Đây là cách chuẩn để áp các giá trị hardening như `tcp_syncookies`, `dmesg_restrict`.

- **B)** Đáp án này sai. `sysctl -w` chỉ thay đổi giá trị runtime và **mất khi reboot**. Nó hữu ích để thử nhanh, nhưng nếu không được ghi vào file cấu hình thì node sẽ quay về giá trị cũ sau lần khởi động kế tiếp — một lỗi audit kinh điển.

- **C)** Đáp án này đúng. `net.ipv4.ip_unprivileged_port_start` là sysctl **namespaced** và nằm trong danh sách safe của Kubernetes (từ v1.22), nên set được qua `securityContext.sysctls` ở cấp pod. Đây là cách cho container non-root bind port thấp mà không cần capability.

- **D)** Đáp án này sai. `net.core.somaxconn` là sysctl **unsafe** (ảnh hưởng toàn node): kubelet sẽ từ chối pod với thông báo `forbidden sysctl: not allowlisted` trừ khi bật `--allowed-unsafe-sysctls`. Thuộc tính `runAsNonRoot` không thay đổi được ranh giới này.

**5. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Đáp án này đúng. Port `10255` là kubelet read-only endpoint **không xác thực**: bất kỳ ai kết nối được sẽ đọc được danh sách pod, spec, thông tin workload — đủ để trinh sát cluster. Nếu nó trả JSON từ máy khác, đó là phát hiện ưu tiên cao: đặt `readOnlyPort: 0` trong KubeletConfiguration và kiểm tra lại.

- **B)** Đáp án này sai — đây là cấu hình **tốt**, không phải dấu hiệu nguy hiểm. etcd chỉ listen trên `127.0.0.1` của control plane (hoặc mạng nội bộ được kiểm soát) là mục tiêu của mọi lần hardening; etcd từng bị khai thác trong nhiều sự cố vì listen `0.0.0.0` không xác thực.

- **C)** Đáp án này đúng. API server mở `0.0.0.0/0` trên cluster production nghĩa là toàn bộ Internet có thể thử xác thực/khai thác lỗ hổng API server. Cách đúng là private endpoint, hoặc public endpoint kèm CIDR allowlist chặt của VPN/bastion, cùng audit logging.

- **D)** Đáp án này sai — đây cũng là cấu hình tốt. Kubelet API (10250) cho phép exec/log nên chỉ nên cho control plane và các node liên quan kết nối. Giới hạn theo CIDR nội bộ là trạng thái mong muốn, không phải vấn đề.

**6. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Đáp án này đúng. `NET_BIND_SERVICE` là capability chính xác cho nhu cầu bind port < 1024. Cách làm đúng là `drop: ["ALL"]` rồi `add: ["NET_BIND_SERVICE"]` — process vẫn chạy non-root, chỉ được cấp đúng một đặc quyền cần thiết.

- **B)** Đáp án này sai. `runAsUser: 0` là quay lại chạy root, phá vỡ mục tiêu least privilege; `readOnlyRootFilesystem` không bù đắp được vì nó chỉ chặn ghi file, không giảm đặc quyền. Đây là "giải pháp" đi ngược hoàn toàn với yêu cầu hardening.

- **C)** Đáp án này đúng. `net.ipv4.ip_unprivileged_port_start=0` là sysctl namespaced an toàn, cho phép process non-root trong pod bind port thấp mà không cần capability nào. Trong một số tổ chức, cách này được ưu tiên hơn add capability vì quyền lực nằm trong network namespace của chính pod.

- **D)** Đáp án này sai. `privileged: true` cấp toàn bộ capability và gỡ hầu hết giới hạn thiết bị — tương đương root trên host, là điều tệ nhất có thể làm trong tình huống này. Có hai cách đúng (A và C) nên không có lý do gì dùng privileged.

**7. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đáp án này đúng. Pod Security Admission với label `pod-security.kubernetes.io/enforce: restricted` sẽ chặn mọi pod không thỏa chuẩn restricted tại thời điểm tạo — đây là enforce tập trung ở tầng cluster, không phụ thuộc vào việc từng team nhớ cấu hình. Có thể đặt thêm `warn`/`audit` ở mức khác để rollout dần.

- **B)** Đáp án này sai. Gửi email nhắc nhở không phải cơ chế kiểm soát kỹ thuật: pod vi phạm vẫn được tạo, và sẽ có team không phản hồi. Admission control phải chặn hoặc audit được tự động, không dựa vào hành vi con người.

- **C)** Đáp án này sai. "Kiểm tra bằng mắt" không mở rộng được và không enforceable: chỉ cần một deployment bỏ sót hoặc một người push trực tiếp là chuẩn bị phá vỡ. Quy trình review là tốt, nhưng phải kèm enforcement tự động.

- **D)** Đáp án này sai. Xóa ServiceAccount mặc định không chặn được việc tạo pod (pod vẫn có thể dùng SA khác hoặc không khai SA) và gây hỏng các workload hợp lệ. Đây là biện pháp không liên quan tới mục tiêu hardening của đề bài.

**8. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đáp án này đúng. Đây là bộ ba cấu hình trả lời đúng từng yêu cầu: `readOnlyRootFilesystem: true` chặn ghi root fs, `allowPrivilegeEscalation: false` chặn leo thang, `drop: ["ALL"]` xóa capabilities; và hai `emptyDir` vào `/var/cache/web`, `/tmp` đáp ứng nhu cầu ghi thật của ứng dụng. Không yêu cầu nào bị hy sinh.

- **B)** Đáp án này sai. Tắt log của ứng dụng để né `readOnlyRootFilesystem` là thay đổi hành vi nghiệp vụ để phục vụ cấu hình — hướng tiếp cận ngược. Log là nhu cầu chính đáng; cách đúng là mount `emptyDir` cho đường dẫn log (hoặc để log ra stdout theo chuẩn container).

- **C)** Đáp án này sai. Ba yêu cầu **có thể** thỏa mãn đồng thời — đó chính là nội dung của đáp án A. Việc bỏ qua `readOnlyRootFilesystem` và `drop capabilities` vì nghĩ không làm được là hiểu sai khả năng của `securityContext`.

- **D)** Đáp án này sai. `privileged: true` làm ngược lại hoàn toàn yêu cầu "không giữ capability nào" và "không leo thang đặc quyền". Dùng AppArmor để bù cho privileged là cấu hình chồng chéo, khó kiểm chứng và đi ngược nguyên tắc defense in depth từ mức quyền thấp nhất.

**9. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Đáp án này đúng. Với các trường trùng nhau, **container-level securityContext override pod-level**: `runAsUser: 0` và `privileged: true` ở container thắng `runAsNonRoot: true` ở pod. Kết quả là pod không hề hardened dù nhìn pod-level tưởng đã an toàn.

- **B)** Đáp án này sai. Kubelet không từ chối vì xung đột này — nó áp theo thứ tự ưu tiên container-level. Đây chính là điều nguy hiểm: cấu hình "trông có vẻ" tuân thủ nhưng runtime lại chạy khác.

- **C)** Đáp án này đúng. Khi audit, phải đọc cả pod-level lẫn container-level; kiểm tra bằng mắt nên bắt đầu từ container-level `securityContext` vì nó quyết định giá trị hiệu lực cuối cùng. Với nhiều container trong pod, phải kiểm tra từng container.

- **D)** Đáp án này sai. Hai trường độc lập: `readOnlyRootFilesystem: false` chỉ ảnh hưởng khả năng ghi file, không tắt hay vô hiệu hóa `runAsNonRoot`. Phát biểu này gán quan hệ nhân quả không tồn tại.

**10. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** Đáp án này đúng. `sysctl -w` thay đổi giá trị ngay trong phiên, không ghi vào file cấu hình nên mất khi reboot. Nó phù hợp để test nhanh một tham số, không phù hợp để làm cấu hình hardening lâu dài — đó là lý do cần `/etc/sysctl.d/`.

- **B)** Đáp án này đúng. Các file trong `/etc/sysctl.d/` được `sysctl --system` nạp theo thứ tự tên file (và có thể bị ảnh hưởng bởi symlink), nên nếu hai file set cùng một key, file nạp sau thắng. Đây là lý do nên dùng prefix rõ ràng (ví dụ `99-`) và kiểm tra cảnh báo "conflict" khi nạp.

- **C)** Đáp án này sai. File sysctl là định dạng `key = value` thuần văn bản (comment bằng `#`), không phải YAML. Nhiều người nhầm vì Kubernetes dùng YAML, nhưng `sysctl --system` chỉ parse định dạng sysctl truyền thống.

- **D)** Đáp án này sai. Không có cơ chế ConfigMap `sysctl-config` nào đồng bộ sysctl mức node xuống pod. Kubelet chỉ có cơ chế **safe sysctl** cho `securityContext.sysctls` (namespaced), còn sysctl mức node phải được cấu hình trên node (qua `/etc/sysctl.d/`, cloud-init hoặc image).
