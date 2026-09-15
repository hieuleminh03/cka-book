---
layout: answer

title: "Chương 7"
subtitle: "Cluster Hardening (CKS)"
exam_objectives:
  - "Dùng RBAC theo nguyên tắc least privilege để giảm thiểu exposure."
  - "Quản lý ServiceAccount an toàn: vô hiệu hóa default và tối thiểu quyền."
  - "Hạn chế truy cập Kubernetes API."
  - "Nâng cấp Kubernetes để vá lỗ hổng bảo mật."
---

## Đáp án {#answers}

**1. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đáp án này đúng. `kubectl auth can-i --list` tính toán **quyền hiệu lực cuối cùng** của subject: hợp (union) của mọi Role/ClusterRole được bind (kể cả ClusterRole qua RoleBinding), cộng với quyền mặc định mà mọi authenticated user được hưởng, và in ra bảng resource/verb. Subject của ServiceAccount phải viết đúng định dạng `system:serviceaccount:<namespace>:<name>`. Đây là công cụ audit chuẩn và cũng là lệnh verify sau remediation.

- **B)** `kubectl describe serviceaccount` chỉ hiển thị thông tin của object SA: `imagePullSecrets`, danh sách Secret (mountable secrets), trạng thái automount token, thời điểm tạo. Nó **không** cho biết SA đang được bind Role/ClusterRole nào, chứ chưa nói tới quyền hiệu lực. Đây là nhầm lẫn rất phổ biến khi mới làm audit.

- **C)** `kubectl -n team-a get roles,rolebindings -o yaml` trả về rule thô và cấu trúc binding, nhưng bạn phải tự "hợp" thủ công: binding nào trỏ role nào, role có `aggregationRule` không, subject có thuộc group nào, và các quyền cluster-scoped đến từ ClusterRoleBinding ở ngoài namespace có bị bỏ sót không. Cách này dễ sai sót và không phải một câu trả lời duy nhất cho "quyền hiệu lực là gì".

- **D)** `kubectl auth whoami` (từ v1.27) chỉ trả lời "tôi là ai" — username và groups — chứ không cho biết subject **được phép làm gì**. Nó hữu ích để xác minh danh tính khi debug authentication, không dùng để audit authorization.


**2. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đáp án này đúng. Rule chỉ dùng đúng core API group (`apiGroups: [""]`), đúng hai resource cần thiết (`pods`, `pods/log`) và đúng ba verb chỉ-đọc (`get`, `list`, `watch`). Không wildcard, không verb phá hoại, và Role là namespace-scoped nên phạm vi tối thiểu. Đây là ví dụ chuẩn của least privilege cho nhu cầu "đọc pod và log".

- **B)** Rule này thừa `delete` — quyền xóa pod không nằm trong nhu cầu "đọc". Trong mô hình RBAC không có deny, quyền thừa không thể bị "khóa lại"; nó tồn tại vĩnh viễn cho tới khi ai đó sửa Role. Least privilege yêu cầu loại bỏ ngay từ đầu, không cấp "cho chắc".

- **C)** `apiGroups: ["*"]` là wildcard trên API group. Dù `resources` đã giới hạn, wildcard group mở đường cho bất kỳ API group nào có resource tên `pods` (kể cả CRD trong tương lai do người khác cài), và làm audit khó hơn vì quyền phụ thuộc vào cluster đang cài gì tại thời điểm truy cập.

- **D)** `pods/exec` với verb `create` là quyền **chạy lệnh trong container** — nguy hiểm hơn hẳn nhu cầu đọc log. Nếu pod đang chạy bằng một SA mạnh, `pods/exec` trở thành đường lấy token của SA đó. Đây là ví dụ ngược lại của least privilege, thường xuất hiện trong các bộ quyền "developer tiện dụng" cần được thu hẹp.


**3. Đáp án đúng là A, B và C.**

**Giải thích:**

- **A)** Đáp án này đúng. Quyền `create pods` cho phép chỉ định `serviceAccountName` tùy ý trong namespace. Nếu tồn tại một SA được bind `cluster-admin` (hoặc quyền mạnh khác), kẻ tấn công tạo pod chạy bằng SA đó rồi đọc token trong `/var/run/secrets/kubernetes.io/serviceaccount/token` để hành động với toàn quyền. Đây là lý do "quyền tạo workload" phải luôn được đánh giá cùng với "SA nào tồn tại trong namespace".

- **B)** Đáp án này đúng. `create` trên `serviceaccounts/token` cho phép gọi TokenRequest API để đúc token cho ServiceAccount — bao gồm cả SA mạnh — mà không cần tạo pod nào. Quyền này nên chỉ thuộc về controller thật sự cần (ví dụ một operator tự cấp token cho workload), tuyệt đối không cấp cho pipeline hay user thông thường.

- **C)** Đáp án này đúng. `impersonate` trên `users`/`groups` cho phép gửi request với danh tính bất kỳ. Đặc biệt nguy hiểm khi kẻ tấn công impersonate group `system:masters` — group này bỏ qua toàn bộ RBAC, nên đây là đường leo thang lên cluster-admin không cần binding nào. Vì vậy `impersonate` phải được xếp ngang hàng với `cluster-admin` khi audit.

- **D)** `watch` trên `configmaps` trong chính namespace chỉ cho phép theo dõi và đọc dữ liệu cấu hình — không đổi danh tính, không tạo tài nguyên, không lấy được credential của SA khác. Nó có thể gây rò rỉ thông tin nếu ConfigMap chứa dữ liệu nhạy cảm (và vẫn nên hạn chế), nhưng không phải đường leo thang đặc quyền trực tiếp như ba đáp án trên.


**4. Đáp án đúng là A, B và C.**

**Giải thích:**

- **A)** Đáp án này đúng. Quy tắc precedence của `automountServiceAccountToken` là: nếu pod spec set giá trị tường minh, giá trị đó thắng; chỉ khi pod để trống (nil) thì giá trị của SA mới được áp dụng. Nếu SA đặt `false` mà pod đặt `true`, token **vẫn được mount** — đây là lý do ở môi trường yêu cầu chặt cần thêm admission policy để cấm pod override ngược lại cấu hình của SA.

- **B)** Đáp án này đúng. Service account controller đảm bảo mỗi namespace luôn có SA `default`; nếu bạn xóa nó, controller sẽ tạo lại. Vì vậy "vô hiệu hóa" trong thực tế là: đặt `automountServiceAccountToken: false` để pod không tự cầm token, gỡ mọi binding cấp quyền cho `default`, và cấp SA riêng (đúng quyền) cho từng workload cần gọi API.

- **C)** Đáp án này đúng. Bound token (TokenRequest API) có thời hạn (`exp`, mặc định khoảng 1 giờ và được kubelet tự gia hạn khi pod còn chạy), có `aud` gắn với API server, và gắn với Pod/SA. Hệ quả: xóa SA vô hiệu hóa toàn bộ token cấp cho SA đó; xóa Pod hủy token gắn với Pod; hết hạn mà không gia hạn thì token tự chết — điều legacy token Secret không làm được.

- **D)** Phát biểu này sai. Bạn tắt/bật automount được ở **cả hai cấp**: trường `automountServiceAccountToken` có trong ServiceAccount spec và trong Pod spec, với quy tắc pod thắng SA. Vì vậy biện pháp ở SA bảo vệ được đa số workload (những pod không set gì), nhưng không phải "khóa cứng" trước pod set tường minh.


**5. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đáp án này đúng. HTTP 403 kèm thông báo `User "system:anonymous"` chứng tỏ request **đã đi qua tầng authentication** và được gán danh tính ẩn danh, sau đó bị RBAC từ chối. Nghĩa là `--anonymous-auth` đang bật (giá trị mặc định). HTTP 200 ở `/version` là vì endpoint này nằm trong ClusterRole `system:public-info-viewer` được bind sẵn cho `system:unauthenticated` (cùng với `/healthz`, `/livez`, `/readyz`). Cách siết đúng là đặt `--anonymous-auth=false` trên kube-apiserver, khi đó request vô danh bị chặn ở authentication và nhận 401.

- **B)** Phát biểu này sai. Nếu anonymous auth đã tắt, cả `/version` lẫn mọi endpoint khác đều trả 401 cho request không credential — không có chuyện 200. HTTP 200 ở `/version` chính là bằng chứng anonymous **đang được phép** truy cập một số endpoint công khai.

- **C)** Sai ở hai điểm. Thứ nhất, lỗi TLS biểu hiện bằng lỗi handshake/certificate (ví dụ `x509: certificate signed by unknown authority`) chứ không phải JSONStatus 403 với danh tính `system:anonymous`. Thứ hai, `--insecure-port` đã bị **xóa hoàn toàn từ Kubernetes v1.24**, không còn để "bật lại".

- **D)** Phát biểu này sai về cả hai vế. 403 chỉ nói "danh tính hiện tại chưa được cấp quyền", không phải "an toàn tuyệt đối": chỉ cần một binding sai cho `system:unauthenticated` (hoặc group `system:authenticated`) là anonymous có quyền. Và `/version` tiết lộ chính xác phiên bản Kubernetes, giúp kẻ tấn công chọn đúng CVE/exploit phù hợp.


**6. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. Private endpoint loại bỏ hoàn toàn khả năng truy cập API server từ Internet; quản trị đi qua VPN hoặc bastion host. Đây là biện pháp rẻ và hiệu quả nhất trên cloud, nên đặt làm mặc định.

- **B)** Đáp án này đúng. Khi buộc phải có đường vào API server (ví dụ CI runner bên ngoài), firewall/security group chỉ cho phép CIDR tin cậy tới cổng 6443 là nguyên tắc allowlist chuẩn. Nhớ áp dụng cho cả etcd (2379/2380) và kubelet (10250/10255) — các cổng này tuyệt đối không được public.

- **C)** Phát biểu này sai. Mở 6443 cho `0.0.0.0/0` tạo bề mặt tấn công toàn Internet: credential stuffing, khai thác lỗ hổng authentication chưa có bản vá, DoS... `--anonymous-auth=false` và audit log chỉ giảm/ghi nhận rủi ro, không **ngăn** được kẻ tấn công chạm tới API. Audit log là lớp phát hiện, không phải lớp phòng ngừa.

- **D)** Đáp án này đúng. Khi buộc phải giữ public endpoint (ví dụ phục vụ CI ở nhiều nơi), hãy dùng allowlist CIDR của nhà cung cấp: `publicAccessCidrs` (EKS) hoặc `--master-authorized-networks` (GKE). Đây là cách thu hẹp nguồn truy cập mà không phải dựng VPN cho mọi client.


**7. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Đáp án này đúng. kubeconfig chứa credential thật (client certificate/token), không phải file cấu hình vô hại, nên phải `chmod 600`, không commit vào Git, không copy lung tung. Tách kubeconfig theo người/vai trò/cluster giúp thu hồi độc lập khi một credential bị lộ và giúp audit biết hành động thuộc về ai.

- **B)** Phát biểu này sai. `admin.conf` do kubeadm tạo chứa client cert thuộc group `system:masters` — group được hard-code để **bỏ qua toàn bộ RBAC**. Chia sẻ file này cho cả team nghĩa là mọi thành viên đều là superuser, không ai bị RBAC giới hạn, không thể thu hồi riêng từng người (phải đổi cert/CA), và audit log không phân biệt được ai làm gì.

- **C)** Đáp án này đúng. OIDC và exec credential plugin (`aws eks get-token`, `gke-gcloud-auth-plugin`, `kubelogin`...) cấp credential **ngắn hạn**, tự xoay vòng, thu hồi được ở tầng IdP/cloud. Đây là cách thay thế đúng cho client cert dài hạn (static credential) trong môi trường production.

- **D)** Phát biểu này sai. Nhúng client cert vào image là phát tán credential: bất kỳ ai kéo được image (hoặc đọc được layer) đều có credential của cluster, và việc thu hồi đòi hỏi rebuild/rotate. Application gọi API server nên dùng ServiceAccount token/workload identity với quyền tối thiểu, hoặc tốt nhất là không nói chuyện trực tiếp với API nếu không cần.


**8. Đáp án đúng là A, B và C.**

**Giải thích:**

- **A)** Đáp án này đúng. Từ v1.28, version skew policy cho phép kubelet cũ hơn kube-apiserver tối đa **3 minor version** (trước đó là 2). Kubelet **không bao giờ** được mới hơn apiserver. Đây là khoảng đệm để bạn drain và nâng từng node mà không phải nâng cả cluster trong một lần.

- **B)** Đáp án này đúng. kubectl được hỗ trợ trong khoảng ±1 minor so với cluster. Vì vậy `kubectl` 1.34 dùng với cluster 1.33 có thể chạy nhưng nằm ngoài policy; còn 1.31 với cluster 1.33 là lệch 2 minor, không được hỗ trợ.

- **C)** Đáp án này đúng. Thứ tự chuẩn là control plane trước (kubelet không được mới hơn apiserver), worker sau; trên mỗi node phải drain để evict workload trước khi thay binary kubelet, rồi uncordon. Drain giúp không gián đoạn dịch vụ khi node restart kubelet.

- **D)** Phát biểu này sai. kubeadm upgrade phải đi **một minor một lần**: 1.30 → 1.31 → 1.32 → 1.33. Chạy `kubeadm upgrade apply v1.33.5` trên cluster 1.30 sẽ bị từ chối (hoặc để lại trạng thái không được hỗ trợ). Muốn nhanh hơn thì giảm thời gian giữa các bước, không nhảy cóc version.


**9. Đáp án đúng là B.**

**Giải thích:**

- **A)** Phương án này sai vì hai lý do. Thứ nhất, không thể xóa vĩnh viễn SA `default` — service account controller sẽ tạo lại ngay, nên đây không phải biện pháp bền vững. Thứ hai, xóa ClusterRole `view` là hành động phá hoại: rất nhiều user/SA khác đang dùng `view`, và bạn không thể "thu hồi có mục tiêu" bằng cách xóa một ClusterRole dùng chung.

- **B)** Đáp án này đúng và đủ ba bước chuẩn của ServiceAccount hygiene: (1) đặt `automountServiceAccountToken: false` trên SA `default` để pod không tự động cầm token — giảm credential nằm trong container; (2) gỡ binding trỏ tới `default` để danh tính này không còn quyền gì; (3) cấp SA riêng với Role hẹp cho những pod thật sự cần gọi API. Cách này xử lý đúng gốc rủi ro mà không ảnh hưởng workload khác.

- **C)** Phương án này sai vì vẫn giữ thói quen xấu: mọi pod đều mount token (credential luôn nằm trong container, chỉ chờ một lỗ hổng RCE là bị lấy), và SA `default` vẫn là một danh tính tập thể dùng chung — khó audit ai làm gì, khó thu hồi khi một pod bị xâm phạm. Giảm quyền là tốt, nhưng chưa đủ nếu token vẫn được mount vô điều kiện.

- **D)** Phương án này sai. Đổi tên SA `default` không có tác dụng: Kubernetes tự tạo lại SA tên `default` cho namespace, và bạn vừa tạo thêm một SA "legacy" mới không ai quản lý — tăng bề mặt thay vì giảm. Ngoài ra pod không chỉ định `serviceAccountName` vẫn dùng `default`, nên hành vi thực tế không thay đổi.


**10. Đáp án đúng là B.**

**Giải thích:**

- **A)** Phương án này sai vì bỏ sót control plane. Các node control plane cũng chạy kubelet, nên nếu CVE ảnh hưởng kubelet thì lỗ hổng vẫn còn nguyên trên control plane; nếu CVE ảnh hưởng thêm apiserver thì càng sai. Ngoài ra cluster sẽ rơi vào trạng thái version không đồng nhất (worker mới hơn control plane), khó audit và vận hành.

- **B)** Đáp án này đúng. Bản vá nằm trong dòng 1.32 (1.32.11), nên nâng control plane lên 1.32.11 trước, rồi cuốn chiếu từng worker: drain → nâng kubeadm/kubelet → uncordon, và verify version sau mỗi node. Patch trong cùng minor ít rủi ro, không có API removal, và đúng nguyên tắc "control plane trước, worker sau, drain trước khi thay kubelet".

- **C)** Phương án này sai vì biến một bản vá bảo mật khẩn thành một đợt minor upgrade. Về mặt kỹ thuật 1.32 → 1.33 không vi phạm skew (một minor một lần), nhưng minor upgrade mang theo thay đổi hành vi, API bị bỏ, cần test admission/webhook/CNI kỹ — thời gian vá kéo dài đúng lúc cần vá nhanh. Bản vá 1.32.11 đã đủ để xử lý CVE.

- **D)** Phương án này sai. Chờ đến quý sau nghĩa là chấp nhận lỗ hổng đã biết trong nhiều tuần — không phù hợp với CVE ảnh hưởng trực tiếp tới kubelet. Ngoài ra 1.32 → 1.34 phải đi qua 1.33 (không nhảy minor), nên đây vừa là trì hoãn vừa là kế hoạch sai về mặt version skew.
