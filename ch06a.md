---
layout: answer

title: "Chương 6"
subtitle: "Cluster Setup (CKS)"
exam_objectives:
  - "Dùng NetworkPolicy để giới hạn truy cập ở mức cluster."
  - "Dùng CIS Benchmark để đánh giá cấu hình bảo mật của etcd, kubelet, kube-apiserver."
  - "Thiết lập Ingress với TLS an toàn."
  - "Bảo vệ node metadata và endpoints; xác minh platform binaries trước khi deploy."
---

## Đáp án {#answers}

**1. Đáp án đúng là C.**

**Giải thích:**

- **A)** `policyTypes: [Ingress]` chỉ khiến chiều ingress bị "khóa" (deny mặc định) và không có rule nào nên mọi traffic vào bị chặn. Tuy nhiên egress vẫn được phép tự do vì chiều egress không nằm trong `policyTypes` — không đạt yêu cầu "cô lập hoàn toàn".

- **B)** `podSelector: {matchLabels: {app: api}}` chỉ select các pod có label `app=api`. Các pod khác trong namespace `payments` không bị policy này quản lý nên vẫn nhận/gửi traffic bình thường. Đây là lỗi phổ biến khi copy policy từ ví dụ mà quên rằng "cô lập toàn namespace" phải dùng `podSelector: {}`.

- **C)** Đáp án này đúng. `podSelector: {}` chọn **mọi pod** trong namespace `payments`. Khi `policyTypes` ghi cả `Ingress` và `Egress` mà không có rule nào, cả hai chiều đều bị deny mặc định — đúng với mẫu default-deny-all dùng làm nền trước khi mở dần từng luồng.

- **D)** `policyTypes: [Egress]` với `egress: []` chỉ chặn chiều ra. Chiều ingress vẫn mở hoàn toàn vì không có policy nào quản lý nó. Ngoài ra việc khai báo tường minh `egress: []` là dư thừa — bỏ trống rules đã có cùng ý nghĩa.



**2. Đáp án đúng là A, B và C.**

**Giải thích:**

- **A)** Đáp án này đúng. NetworkPolicy không có cơ chế ưu tiên hay ghi đè: mọi policy cùng select một pod được đánh giá độc lập và kết quả là **hợp (union)** của tất cả rule cho phép. Muốn traffic đi qua, chỉ cần **một** policy cho phép nó.

- **B)** Đáp án này đúng. Đây là cơ chế cốt lõi: khi một pod bị select với `policyTypes` chứa `Egress`, chiều egress mặc định bị chặn; không có rule egress nào nghĩa là không có ngoại lệ nào, tức toàn bộ egress bị chặn.

- **C)** Đáp án này đúng. Trong cùng một peer, `podSelector` giới hạn **pod nào** còn `namespaceSelector` giới hạn **namespace nào**; hai điều kiện phải đồng thời thỏa mãn (AND). Nhiều peer khác nhau trong cùng mảng `from`/`to` mới là OR.

- **D)** Đáp án này sai. NetworkPolicy là mô hình allow-only, không có rule deny tường minh. Muốn "chặn một IP cụ thể" bạn phải dùng cách gián tiếp: cho phép dải rộng hơn với `ipBlock` và loại trừ địa chỉ đó bằng `except`.



**3. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai ở hai điểm: schema của NetworkPolicy không có trường `deny` cho peer, nên manifest sẽ bị từ chối hoặc field bị bỏ qua; và nếu chỉ liệt kê `169.254.169.254/32`, ý nghĩa của mô hình allow-only là **chỉ cho phép** truy cập đúng IP đó — ngược hoàn toàn với mục tiêu chặn metadata.

- **B)** Đáp án này đúng. `cidr: 0.0.0.0/0` cho phép egress ra mọi địa chỉ IPv4, còn `except: [169.254.169.254/32]` loại trừ endpoint metadata khỏi phạm vi cho phép. Đây là cách diễn đạt "allow all trừ X" đúng chuẩn trong mô hình allow-only.

- **C)** Sai vì trường đúng là `except`, không phải `exclude`. Tùy chế độ validate, `kubectl apply` có thể báo lỗi unknown field; nếu bị bỏ qua thì policy sẽ **cho phép cả metadata**, tức lỗ hổng vẫn còn mà bạn tưởng đã chặn.

- **D)** Sai vì đây là rule `ingress` — nó quản lý traffic đi **vào** pod, trong khi truy cập metadata là traffic đi **ra** từ pod (egress). Đặt sai chiều thì policy không có tác dụng với tình huống đề bài.



**4. Đáp án đúng là B.**

**Giải thích:**

- **A)** Rule này chỉ mở UDP 53. Phần lớn truy vấn DNS dùng UDP nên hệ thống trông như hoạt động, nhưng khi response vượt kích thước cho phép (DNSSEC, nhiều bản ghi) client sẽ fallback sang TCP 53 và bị chặn — tạo ra lỗi "chập chờn" rất khó debug. Đây là lựa chọn *gần đúng* nhưng thiếu, nên không phải đáp án tốt nhất.

- **B)** Đáp án này đúng. `namespaceSelector` trỏ tới `kube-system` (dùng label tự động `kubernetes.io/metadata.name`) đảm bảo đúng phạm vi chứa CoreDNS, và mở cả UDP lẫn TCP 53 đảm bảo DNS hoạt động đầy đủ trong mọi tình huống.

- **C)** `podSelector: {matchLabels: {k8s-app: kube-dns}}` không có `namespaceSelector` sẽ được hiểu là "pod khớp label trong **chính namespace chứa policy**" (namespace của pod bị select, ở đây là namespace ứng dụng). Vì CoreDNS nằm ở `kube-system`, peer này không khớp pod nào và DNS vẫn bị chặn. Muốn trỏ đúng phải thêm `namespaceSelector` — minh họa luôn quy tắc AND.

- **D)** Sai vì cả hai lý do: DNS là luồng **egress** từ pod nên phải nằm trong policy có `policyTypes` chứa `Egress`; thêm policy `Ingress` không mở được chiều ra. Ngoài ra rule DNS đặt ở chiều ingress cũng không có ý nghĩa vì CoreDNS không phải là client khởi tạo kết nối tới pod ứng dụng.



**5. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Đáp án này đúng. kube-bench đọc file cấu hình trên node (`/etc/kubernetes/manifests`, `/var/lib/kubelet`, `/var/lib/etcd`...) nên khi chạy dạng Job phải mount hostPath và bật `hostPID: true`; ngoài ra có thể chạy binary trực tiếp trên node khi có SSH.

- **B)** Đáp án này sai. CIS Benchmark là baseline khuyến nghị, không phải danh sách lỗ hổng đang bị khai thác. Nhiều check FAIL là do môi trường (managed cluster, cluster học tập) hoặc là trade-off vận hành. Cách làm đúng là đánh giá theo rủi ro, ưu tiên các cấu hình ảnh hưởng bề mặt tấn công, chứ không phải "sửa bằng mọi giá trước khi cluster được hoạt động".

- **C)** Đáp án này đúng. Output chia PASS/FAIL/WARN/INFO và mỗi check FAIL/WARN đều có phần Remediation mô tả file cần sửa, flag cần thêm — đây là lý do kube-bench hữu ích trong phòng thi và trong audit.

- **D)** Đáp án này sai. kube-bench chỉ **báo cáo** trạng thái tuân thủ; nó không có cơ chế tự động sửa cấu hình component. Việc remediation (sửa manifest, restart kubelet...) phải do con người hoặc công cụ khác thực hiện có kiểm soát.



**6. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đáp án này đúng. Đây là bộ cấu hình chuẩn: tắt anonymous (`authentication.anonymous.enabled: false`), bật xác thực qua API server (`webhook.enabled: true`), phân quyền qua API server (`authorization.mode: Webhook`) và đóng cổng đọc ẩn danh (`readOnlyPort: 0`). Sau khi sửa phải restart kubelet và verify lại.

- **B)** Sai vì giữ `anonymous.enabled: true` chính là để nguyên lỗ hổng đang bị kube-bench cảnh báo. Tệ hơn, `authorization.mode: AlwaysAllow` bỏ qua toàn bộ bước phân quyền: bất kỳ ai xác thực được cũng làm được mọi thứ, kể cả `exec` vào container.

- **C)** Sai vì `AlwaysAllow` tháo bỏ lớp authorization và `readOnlyPort: 10255` mở lại cổng đọc không xác thực — đúng hai thứ mà CIS yêu cầu loại bỏ.

- **D)** Sai vì tắt webhook authentication và xóa `clientCAFile` khiến kubelet không còn cách xác thực client hợp lệ; đây là bước lùi bảo mật, có thể làm mất luôn khả năng kết nối của control plane với kubelet.



**7. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đáp án này đúng. `spec.tls` trỏ Secret `demo-tls` để controller terminate TLS cho `demo.local`; `ssl-redirect` bật chuyển hướng HTTP→HTTPS (mặc định đã bật khi có TLS) và `force-ssl-redirect` ép redirect ngay cả với request đến listener HTTP không khớp rule có TLS. `ingressClassName: nginx` đảm bảo đúng controller xử lý.

- **B)** `backend-protocol: HTTPS` dùng cho mô hình **re-encrypt**: controller mở kết nối HTTPS tới backend, không liên quan đến việc chuyển hướng người dùng cuối từ HTTP sang HTTPS. Với cấu hình này, request HTTP vẫn được phục vụ bình thường nếu không có annotation redirect — sai mục tiêu đề bài.

- **C)** Ingress controller không tự sinh certificate cho hostname (trừ khi bạn cấu hình default certificate ở cấp controller, và khi đó cert không khớp hostname tùy biến). Không khai báo `spec.tls` nghĩa là không terminate TLS đúng cách, và `ssl-redirect` cũng không có Secret nào để dùng.

- **D)** Đặt `ssl-redirect: "false"` và `force-ssl-redirect: "false"` đúng là tránh được vòng lặp redirect trong một số cấu hình phức tạp, nhưng như vậy request HTTP sẽ không được chuyển sang HTTPS — trái yêu cầu "ép mọi request HTTP sang HTTPS" của đề bài.



**8. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** Đáp án này đúng. cert-manager quản lý vòng đời certificate qua CRD: `Issuer`/`ClusterIssuer` định nghĩa nguồn cấp (self-signed, CA, ACME, Vault...), `Certificate` mô tả certificate mong muốn và cert-manager ghi `tls.crt`/`tls.key` vào Secret được chỉ định trong `secretName`.

- **B)** Đáp án này đúng. Annotation `cert-manager.io/cluster-issuer` trên Ingress khiến cert-manager tự tạo/cập nhật Secret trùng với `spec.tls.secretName`, và nhờ `duration`/`renewBefore` (hoặc giá trị mặc định) nó tự gia hạn trước khi hết hạn — giải quyết bài toán "cert hết hạn lúc 2 giờ sáng".

- **C)** Đáp án này sai. cert-manager không phải Ingress controller và không nhận traffic: nó chỉ cấp phát certificate. Ingress controller (nginx, Traefik, Envoy Gateway...) vẫn là thành phần terminate TLS và route traffic; hai thành phần phối hợp với nhau qua Secret.

- **D)** Đáp án này sai. cert-manager hỗ trợ nhiều loại issuer: `SelfSigned`, `CA` (CA nội bộ), `ACME` (Let's Encrypt), `Vault`, `Venafi`... Trong đó self-signed và CA nội bộ rất hữu ích cho môi trường lab, dev hoặc mTLS nội bộ.



**9. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. `169.254.169.254` là địa chỉ link-local mà AWS, GCP, Azure (và nhiều cloud khác) dùng cho Instance Metadata Service. Nếu pod gọi được endpoint này, kẻ tấn công có thể đọc credential tạm thời của IAM role gắn với instance — con đường leo thang rất phổ biến.

- **B)** Đáp án này đúng. IMDSv2 (yêu cầu token qua PUT) kết hợp hop limit 1 làm packet từ container không tới được metadata; nhưng đây chỉ là một lớp. NetworkPolicy egress, least privilege IAM và phát hiện (GuardDuty/Falco) vẫn cần thiết vì phòng thủ theo chiều sâu.

- **C)** Đáp án này sai. Traffic từ pod tới metadata là chiều **egress**, nên policy phải có `policyTypes` chứa `Egress` (kèm rule `ipBlock` với `except`). Một policy ghi `Ingress` không chặn được kết nối ra ngoài.

- **D)** Đáp án này đúng. NetworkPolicy gắn với network namespace của pod; pod dùng `hostNetwork: true` dùng chung namespace mạng với node nên thoát khỏi policy pod. Các host process (kubelet, container runtime...) cũng không nằm dưới NetworkPolicy — vì vậy cần thêm các lớp bảo vệ ở node/host (iptables/nftables, firewall, IMDSv2).



**10. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đáp án này đúng. Quy trình chuẩn là tải binary và file `.sha256` từ **cùng URL chính thức** trên `dl.k8s.io`, verify bằng `sha256sum --check`, sau đó nếu cần chứng minh nguồn gốc thì tải `.sig`/`.cert` và chạy `cosign verify-blob` với identity `krel-staging@k8s-releng-prod.iam.gserviceaccount.com` và OIDC issuer `https://accounts.google.com` (cơ chế ký keyless của Kubernetes từ v1.26+).

- **B)** Sai ở hai điểm: MD5 không còn được coi là thuật toán an toàn, và một bài blog không phải nguồn tin cậy để lấy checksum — kẻ tấn công có thể chỉnh cả bài blog lẫn binary. Checksum phải lấy từ kênh chính thức (lý tưởng là từ kênh độc lập, không phải cùng server với binary).

- **C)** Sai về mặt kỹ thuật: `gpg --verify kubectl` cần file chữ ký rời (`.sig`/`.asc`) và public key của người ký trong keyring; chỉ có binary thì lệnh không thể xác minh. Ngoài ra binary Kubernetes hiện đại được ký bằng cosign keyless, không phải GPG, nên quy trình này cũng không khớp với artifact bạn đang tải.

- **D)** Sai vì "chạy được" không chứng minh file toàn vẹn hay đến từ nguồn chính thức — mã độc hoàn toàn có thể là một binary `kubectl` hợp lệ về mặt chức năng. Đây chính là kịch bản supply chain attack mà bước verify nhằm ngăn chặn.
