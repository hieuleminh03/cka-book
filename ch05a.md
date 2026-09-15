---
layout: answer

title: "Chương 5"
subtitle: "Kubernetes vận hành & troubleshooting (CKA)"
exam_objectives:
  - "Quản lý vòng đời cluster: upgrade với kubeadm, backup/restore etcd và node maintenance (cordon/drain)."
  - "Triển khai RBAC: Role, ClusterRole, RoleBinding, ClusterRoleBinding và ServiceAccount."
  - "Kiểm tra cấu hình bảo mật component theo CIS Benchmark (kubelet, kube-apiserver, etcd)."
  - "Xử lý sự cố cluster và workload: node NotReady, pod Pending, DNS, network và control plane."
---

## Đáp án {#answers}

**1. Đáp án đúng là C.**

**Giải thích:**

- **A)** v1.30 hợp lệ. Theo version skew policy hiện hành, kubelet được phép cũ hơn kube-apiserver tối đa 3 minor version; với apiserver v1.33 thì v1.30 (cũ hơn 3 minor) vẫn nằm trong giới hạn.
- **B)** v1.31 hợp lệ. Đây là kubelet cũ hơn apiserver 2 minor — hoàn toàn được hỗ trợ.
- **C)** v1.34 là đáp án đúng của câu hỏi. kubelet **không bao giờ được mới hơn** kube-apiserver, dù chỉ 1 minor. Một node chạy kubelet v1.34 với apiserver v1.33 là cấu hình bị cấm: kubelet có thể gọi các API chưa tồn tại trên apiserver và gây lỗi khó lường.
- **D)** v1.32 hợp lệ. Đây là kubelet cũ hơn apiserver 1 minor, mức lệch phổ biến và an toàn trong thực tế.

**2. Đáp án đúng là B và C.**

**Giải thích:**

- **A)** Sai. kube-apiserver không cho phép bỏ qua minor khi nâng cấp; muốn lên v1.33 từ v1.31 bạn phải đi qua v1.32. Nhảy cóc minor có thể làm hỏng dữ liệu và bị kubeadm từ chối (hoặc để lại trạng thái không được hỗ trợ).
- **B)** Đúng. `kubeadm upgrade apply` chỉ chạy trên control plane node đầu tiên (nơi nó nâng control plane và etcd/addon), còn các control plane node khác và worker node dùng `kubeadm upgrade node`.
- **C)** Đúng. Minor version upgrade của kubelet không được hỗ trợ in-place, vì vậy phải drain node trước khi thay binary để pod được evict an toàn sang node khác.
- **D)** Sai. kubeadm **không có lệnh rollback**. Khi upgrade thất bại, đường lui đúng là restore etcd từ snapshot (hoặc chạy lại `kubeadm upgrade apply --force` vì lệnh này idempotent); hạ cấp package không được khuyến nghị và có thể để lại cluster ở trạng thái hỗn hợp.

**3. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đúng. Lệnh có đủ `--endpoints`, CA và cặp cert/key client để etcd xác thực qua mTLS. `etcdctl snapshot save` tạo bản snapshot nhất quán ngay khi etcd đang chạy và không ảnh hưởng hiệu năng đáng kể.
- **B)** Sai. etcdctl không có subcommand `backup save`; `--data-dir` là tham số của quá trình **restore**, không dùng khi backup.
- **C)** Sai. Copy trực tiếp thư mục dữ liệu trong lúc etcd đang hoạt động không đảm bảo tính nhất quán (etcd đang ghi liên tục). Tài liệu Kubernetes chỉ cho phép copy file `member/snap/db` từ data directory **không còn được process etcd sử dụng**, còn cách chuẩn để backup khi etcd đang chạy là built-in snapshot.
- **D)** Sai. Đây là lệnh restore, không phải backup. Tệ hơn, restore đè lên `/var/lib/etcd` đang được etcd sử dụng có thể làm hỏng cả dữ liệu hiện tại lẫn dữ liệu khôi phục.

**4. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. Cụm 3 member cần quorum là 2. Mất 1 node thì còn 2 member — vẫn đủ quorum nên etcd tiếp tục phục vụ đọc và ghi bình thường.
- **B)** Đúng. Với 2/3 member, etcd vẫn hoạt động nhưng **không còn khả năng chịu lỗi**: nếu thêm một member nữa chết (hoặc mất kết nối), cluster mất quorum, mọi thao tác ghi bị treo và apiserver không còn hoạt động đúng. Việc cần làm là khôi phục member thứ ba càng sớm càng tốt.
- **C)** Sai. etcd không tự động thêm member mới; muốn thay member bạn phải làm thủ công (`etcdctl member remove` / `member add`) sau khi node mới sẵn sàng. Ngoài ra member mới chỉ nên được thêm khi cụm khỏe mạnh.
- **D)** Sai. Restore snapshot chỉ dành cho tình huống mất quorum hoặc dữ liệu hỏng. Khi cụm còn quorum, việc restore (đặc biệt là restore một member cũ vào cụm đang chạy) sẽ làm member đó lệch lịch sử so với quorum và gây sự cố nghiêm trọng hơn.

**5. Đáp án đúng là C.**

**Giải thích:**

- **A)** Sai. `kubectl drain worker-1` không kèm flag sẽ báo lỗi ngay: không thể xóa pod do DaemonSet quản lý (CNI) và không thể xóa pod có local storage (emptyDir). Node sẽ không bao giờ drain xong.
- **B)** Sai. Dùng `kubectl delete pods` xóa thẳng pod không qua Eviction API, phá vỡ cam kết PodDisruptionBudget và có thể làm dịch vụ gián đoạn ngoài ý muốn; cách này cũng không đảm bảo pod được tái tạo đúng thứ tự.
- **C)** Đúng. `--ignore-daemonsets` bỏ qua các pod do DaemonSet quản lý (chúng vẫn chạy trên node), `--delete-emptydir-data` cho phép xóa pod dùng emptyDir sau khi bạn chấp nhận mất dữ liệu tạm. Drain dùng Eviction API nên vẫn tôn trọng PDB.
- **D)** Sai. Taint `NoExecute` sẽ đuổi pod không tolerate khỏi node, nhưng đây không phải quy trình bảo trì chuẩn: eviction do taint **không tôn trọng PodDisruptionBudget**, không có cơ chế chờ drain hoàn tất, và bạn phải nhớ gỡ taint sau khi bảo trì. `kubectl drain` mới là công cụ đúng.

**6. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Đúng. RoleBinding có thể trỏ tới ClusterRole; khi đó quyền của ClusterRole chỉ có hiệu lực **trong namespace của RoleBinding**. Đây là cách tái sử dụng ClusterRole (`view`, `edit`...) cho nhiều namespace mà không phải tạo Role trùng lặp.
- **B)** Sai. `roleRef` của ClusterRoleBinding chỉ chấp nhận `kind: ClusterRole`; bạn không thể bind một Role (vốn thuộc namespace) trên toàn cluster.
- **C)** Đúng. Subject `kind: User` bắt buộc có `name` và `apiGroup: rbac.authorization.k8s.io`; tương tự với `kind: Group` và `kind: ServiceAccount` (ServiceAccount cần thêm `namespace`).
- **D)** Sai. `nodes` là **cluster-scoped**, không phải namespaced, nên Role (chỉ có hiệu lực trong một namespace) không thể cấp quyền cho nó; quyền với `nodes` phải đến từ ClusterRole gắn qua ClusterRoleBinding (RoleBinding không giúp được gì với resource cluster-scoped).

**7. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** Đúng. Từ Kubernetes v1.24, việc tạo ServiceAccount **không còn tự động sinh Secret token vĩnh viễn**. Cơ chế hiện hành là **bound token**: kubelet (hoặc client) xin token qua TokenRequest API, token có thời hạn và gắn với pod/SA cụ thể.
- **B)** Đúng. `kubectl create token my-sa` tạo bound token với thời hạn mặc định **1 giờ** (có thể chỉnh bằng `--duration`, nhưng bị giới hạn bởi cấu hình apiserver). Token này rất tiện để test API nhanh.
- **C)** Sai. Bound token gắn chặt với ServiceAccount và pod sử dụng nó; khi SA bị xóa (hoặc pod bị xóa), token trở nên vô hiệu. Đây chính là ưu điểm bảo mật so với legacy token Secret không thể thu hồi.
- **D)** Sai. Kubelet tự mount projected token vào pod tại `/var/run/secrets/kubernetes.io/serviceaccount/`; không cần tạo Secret thủ công. Ngược lại, pod nào không cần gọi API server nên đặt `automountServiceAccountToken: false` để giảm bề mặt tấn công.

**8. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đúng. Taint `dedicated=gpu:NoSchedule` khiến scheduler loại `worker-2` khỏi danh sách node ứng viên cho mọi pod không có toleration tương ứng — pod sẽ không bao giờ được schedule lên node này.
- **B)** Sai. Node còn trống không giúp vượt qua taint; taint là ràng buộc cứng với effect `NoSchedule`, scheduler loại node trước khi xét tài nguyên.
- **C)** Sai. `NoSchedule` chỉ áp dụng cho **pod mới**; pod đang chạy trên node không bị đuổi khi taint được thêm vào. Chỉ khi effect là `NoExecute` thì pod không tolerate mới bị evict.
- **D)** Sai. Toleration cần khai báo phải khớp với effect đang tồn tại, tức `effect: NoSchedule` (hoặc bỏ trống effect để khớp mọi effect). `NoExecute` không phải điều kiện bắt buộc để chạy pod trên node bị taint.

**9. Đáp án đúng là A và D.**

**Giải thích:**

- **A)** Đúng. Trên cluster kubeadm, cấu hình kubelet nằm trong `/var/lib/kubelet/config.yaml` (KubeletConfiguration) và `/var/lib/kubelet/kubeadm-flags.env` (flag command line). Sửa trực tiếp systemd unit là cách làm dễ bị ghi đè khi upgrade và khó kiểm toán, nên không được khuyến nghị.
- **B)** Sai. CIS Kubernetes Benchmark có hẳn một nhóm check cho **kubelet** (anonymous auth, authorization mode, read-only port, protect kernel defaults, certificate rotation...). kubelet là một trong những thành phần quan trọng nhất vì nó tiếp xúc trực tiếp với container runtime và host.
- **C)** Sai. `readOnlyPort` mặc định hiện hành là `0` (tắt hoàn toàn). Port 10255 là read-only port cũ, không được metric-server sử dụng — metric-server đọc metrics qua port **10250** với xác thực và phân quyền.
- **D)** Đúng. CIS khuyến nghị `authentication.anonymous.enabled: false` (không cho truy cập ẩn danh) và `authorization.mode: Webhook` để mọi request tới kubelet phải qua xác thực và được apiserver phân quyền.

**10. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** Đúng. EndpointSlice được tạo từ các pod khớp `spec.selector` của Service. Nếu selector sai (ví dụ service chọn `app=web` nhưng pod mang label `app=web-v2`), không pod nào khớp và EndpointSlice rỗng — service tồn tại nhưng không route tới đâu.
- **B)** Đúng. Mặc định chỉ pod ở trạng thái **Ready** mới được đưa vào EndpointSlice. Pod đang khởi động, readiness probe fail, hoặc bị NotReady vì node lỗi đều bị loại khỏi endpoint (trừ khi service đặt `publishNotReadyAddresses: true`).
- **C)** Sai. EndpointSlice do **endpointslice controller** trên control plane tạo và cập nhật dựa trên selector và trạng thái Ready của pod — hoàn toàn không phụ thuộc kube-proxy. kube-proxy chỉ lập trình dataplane (iptables/IPVS) để chuyển traffic tới các endpoint đã có.
- **D)** Sai. CoreDNS chỉ đảm nhiệm phân giải tên miền; DNS lỗi khiến bạn không resolve được tên service, nhưng không làm EndpointSlice rỗng. Khi DNS chết, bạn vẫn có thể truy cập service bằng ClusterIP và endpoint vẫn được cập nhật bình thường.

