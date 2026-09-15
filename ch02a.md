---
layout: answer

title: "Chương 2"
subtitle: "Networking & Cluster Networking"
exam_objectives:
  - "Giải thích TCP/IP, DNS, HTTP/TLS và luồng đi của một request từ client tới service."
  - "Cấu hình reverse proxy, load balancing với nginx/HAProxy và debug network bằng các công cụ dòng lệnh."
  - "Hiểu CNI, Service, Ingress, CoreDNS và NetworkPolicy trong Kubernetes."
  - "Thiết lập Ingress với TLS và mã hóa Pod-to-Pod (mTLS với Cilium/Istio) theo yêu cầu CKS."
---

## Đáp án {#answers}

**1. Đáp án đúng là B.**

**Giải thích:**

- **A)** Chỉ khai báo `policyTypes: [Ingress]` nên policy chỉ cô lập chiều vào. Mọi traffic egress từ pod trong namespace `demo` vẫn được phép đi ra (ví dụ gọi ra Internet, gọi sang namespace khác) trừ khi có policy egress khác chọn các pod đó. Đáp án này không đáp ứng yêu cầu "chặn cả hai chiều".
- **B)** Đáp án này đúng. `podSelector: {}` là empty selector — nó chọn **mọi pod** trong namespace chứa policy. Khi `policyTypes` chứa cả `Ingress` và `Egress`, mọi pod trong namespace bị cô lập theo cả hai chiều: traffic vào và ra đều bị chặn cho tới khi có policy khác mở lại từng luồng cụ thể. Đây chính là khuôn mẫu default-deny toàn namespace.
- **C)** Label selector của Kubernetes không hỗ trợ wildcard trong giá trị label. `app: "*"` được hiểu là giá trị literal `"*"`, gần như chắc chắn không khớp với bất kỳ pod nào. Kết quả là policy không chọn pod nào và không cô lập được gì — một "silent failure" rất nguy hiểm vì `kubectl get networkpolicy` vẫn hiển thị policy bình thường.
- **D)** Policy nằm trong namespace `kube-system` nên `podSelector: {}` chọn mọi pod của `kube-system` (CoreDNS, kube-proxy...) chứ không phải namespace `demo`. Ngoài việc không cô lập được `demo`, cấu hình này còn có thể gây sự cố hệ thống nghiêm trọng.

**2. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. `ndots:5` quy định rằng với tên có **ít hơn 5 dấu chấm**, resolver (glibc/stub resolver trong pod) sẽ thử lần lượt các search domain trong `/etc/resolv.conf` trước khi truy vấn tên gốc. Vì pod có ba search domain (`demo.svc.cluster.local`, `svc.cluster.local`, `cluster.local`), `api.example.com` bị ghép thêm thành ba tên không tồn tại và sinh ra ba truy vấn NXDOMAIN trước khi được phân giải đúng.
- **B)** Đáp án này đúng. Thêm dấu chấm cuối (`api.example.com.`) biến tên thành **tên tuyệt đối (FQDN)**, resolver hiểu đây là tên đầy đủ và bỏ qua toàn bộ search domain — loại bỏ các truy vấn NXDOMAIN thừa. Đây là mẹo đơn giản nhưng hiệu quả khi ứng dụng gọi nhiều API bên ngoài cluster.
- **C)** Đáp án này sai. Với `ndots:1`, tên `backend` (không có dấu chấm nào, tức 0 < 1) vẫn được thử qua các search domain trước, nên pod vẫn phân giải được service trong cluster. Giảm `ndots` ảnh hưởng tới thứ tự thử nghiệm, không phá vỡ việc phân giải tên service nội bộ.
- **D)** Đáp án này đúng. `spec.dnsConfig.options` cho phép ghi đè `ndots` cho riêng pod, ví dụ `ndots: "2"`: tên có từ 2 dấu chấm trở lên (như `api.example.com`) sẽ được truy vấn trực tiếp, còn tên ngắn trong cluster vẫn phân giải qua search domain. Đây là cách tinh chỉnh phổ biến cho workload gọi nhiều dịch vụ ngoài.

**3. Đáp án đúng là B.**

**Giải thích:**

- **A)** API server không kiểm tra sự tồn tại của cloud controller khi tạo Service. Service với `type: LoadBalancer` luôn được tạo thành công; việc cấp IP ngoài là trách nhiệm của controller bất đồng bộ phía sau.
- **B)** Đáp án này đúng. Không có cloud controller (hoặc không có LB implementation như MetalLB), Service nằm ở trạng thái chờ và `EXTERNAL-IP` hiển thị `<pending>` vô thời hạn. Bản thân Service vẫn hoạt động ở tầng NodePort (LoadBalancer được xây trên NodePort), nên bạn vẫn truy cập được qua `<IP-node>:<nodePort>`. Cài MetalLB là cách chuẩn để cấp IP cho `LoadBalancer` trên bare-metal, hoạt động ở chế độ L2 (ARP) hoặc BGP.
- **C)** Sai. Pod và Deployment không phụ thuộc vào việc Service có external IP hay không. Pod vẫn được schedule và chạy bình thường; chỉ phần expose ra ngoài bị treo ở trạng thái pending.
- **D)** Sai. NodePort vẫn hoạt động đầy đủ vì `LoadBalancer` tự động cấp một NodePort cho mọi node. `kubectl port-forward` chỉ là một cách truy cập tạm thời khác, không phải cách duy nhất.

**4. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. iptables mode viết rule dạng chuỗi (chain) cho từng service và từng endpoint; số rule tăng theo `số service × số endpoint`. Khi cluster có hàng nghìn service, việc cập nhật và duyệt rule trở nên tốn kém, ảnh hưởng latency khi service thay đổi. Với cluster nhỏ và vừa, iptables vẫn ổn định và là mặc định.
- **B)** Đáp án này đúng. IPVS dùng hash table trong kernel, tra cứu nhanh hơn, và hỗ trợ sẵn nhiều scheduler như round-robin, weighted round-robin, least connection. Đây là lý do IPVS phù hợp cluster lớn với nhiều service.
- **C)** Đáp án này sai. Kể cả khi chạy IPVS mode, kube-proxy vẫn cần iptables cho một số việc như SNAT/masquerade, đánh dấu gói tin (mark) và các rule phụ trợ. IPVS không thay thế hoàn toàn iptables trong mode này.
- **D)** Đáp án này đúng. nftables mode của kube-proxy đã chuyển sang GA (stable) từ Kubernetes v1.33, với ruleset gọn hơn, hiệu năng tốt hơn và giảm số rule so với iptables — là lựa chọn đáng cân nhắc cho cluster mới. (Một hướng khác là để CNI eBPF như Cilium thay thế hoàn toàn kube-proxy.)

**5. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Đáp án này đúng. Khi `namespaceSelector` và `podSelector` nằm trong **cùng một phần tử** của mảng `from` (hoặc `to`), Kubernetes áp dụng phép **AND**: pod nguồn phải khớp cả hai điều kiện — có label `app=frontend` **và** nằm trong namespace có label `team=a`.
- **B)** Đáp án này sai. Hai selector nằm ở **hai phần tử khác nhau** của mảng là phép **OR**: cho phép hoặc mọi pod `app=frontend` ở bất kỳ namespace nào, hoặc mọi pod trong namespace `team=a`. Đây là nhầm lẫn phổ biến nhất khi viết NetworkPolicy và thường vô tình mở rộng quyền truy cập hơn dự kiến.
- **C)** Đáp án này đúng. `podSelector` đứng một mình (không kèm `namespaceSelector` trong cùng phần tử) chỉ chọn pod trong **namespace của policy**. Muốn cho phép pod từ namespace khác, bắt buộc phải thêm `namespaceSelector` (thường dùng nhãn `kubernetes.io/metadata.name: <namespace>`).
- **D)** Đáp án này sai. `namespaceSelector: {}` là empty selector — nó khớp **mọi namespace**, tương đương "cho phép tất cả pod trong cluster", không phải chỉ namespace `default`.

**6. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. Ingress controller không tự động copy Secret giữa các namespace; việc này sẽ vi phạm ranh giới bảo mật namespace. Controller chỉ đọc Secret theo tên trong chính namespace của Ingress.
- **B)** Đáp án này đúng. Trường `spec.tls[].secretName` chỉ là **tên Secret**, không có trường namespace. Nó luôn được resolve trong namespace của Ingress. Secret ở namespace khác sẽ không bao giờ được tìm thấy, và controller thường fallback về certificate mặc định (self-signed) hoặc từ chối cấu hình TLS cho host đó. Cách xử lý: tạo Secret cùng namespace với Ingress (có thể dùng công cụ như cert-manager hoặc đồng bộ Secret có kiểm soát).
- **C)** Sai. API server không validate sự tồn tại của Secret khi tạo Ingress — nó chỉ lưu object. Vì vậy Ingress được tạo thành công và lỗi chỉ lộ ra khi kiểm tra TLS thực tế, khiến việc chẩn đoán khó hơn.
- **D)** Sai. Không có cơ chế tự chuyển sang TLS passthrough. Passthrough là cấu hình chủ động (thường qua annotation hoặc ConfigMap riêng của controller) và không liên quan tới việc thiếu certificate.

**7. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. NetworkPolicy là API có sẵn của Kubernetes, nhưng việc **thực thi** là tính năng tùy chọn của CNI. Nhiều CNI (như Flannel thuần) lưu object mà không hề enforce, tạo ra "silent failure" cực kỳ nguy hiểm trong môi trường production.
- **B)** Đáp án này đúng. Flannel chỉ lo việc kết nối mạng (VXLAN/host-gw), không có policy engine. Muốn enforce NetworkPolicy, bạn cần thêm một engine như Calico ở chế độ policy-only (mô hình "Canal" = Flannel + Calico policy) hoặc đổi sang CNI hỗ trợ policy đầy đủ như Calico, Cilium, Antrea.
- **C)** Sai. kube-proxy chỉ chịu trách nhiệm phần Service (VIP, DNAT, load balancing). NetworkPolicy được thực thi ở datapath của CNI (iptables/eBPF trong kernel, trên veth của pod...), không phải bởi kube-proxy.
- **D)** Sai. Pod Security admission là cơ chế kiểm soát cấu hình bảo mật của pod (privileged, capability, seccomp...), hoàn toàn độc lập với NetworkPolicy. Bật hay tắt PSA không ảnh hưởng tới việc policy mạng có được enforce hay không.

**8. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. nginx mã nguồn mở **không** hỗ trợ active health check định kỳ. Tính năng health check chủ động thuộc bản thương mại NGINX Plus (hoặc phải dùng công cụ ngoài như nginx-ingress có cơ chế riêng). Đây là khác biệt quan trọng so với HAProxy.
- **B)** Đáp án này đúng. `max_fails=3 fail_timeout=10s` là cấu hình passive health check: nginx quan sát lỗi trên traffic thật, và nếu backend fail 3 lần trong vòng 10 giây thì bị đánh dấu down và tạm loại khỏi vòng xoay trong 10 giây. Sau khoảng thời gian đó, backend được đưa trở lại và thử nghiệm lại.
- **C)** Sai. Backend không bị xóa vĩnh viễn khỏi `upstream`; nó chỉ bị tạm loại trong `fail_timeout` và được thử lại tự động. Muốn loại vĩnh viễn phải sửa cấu hình (kèm `down` marker hoặc xóa server) và reload.
- **D)** Sai. nginx là reverse proxy, không quản lý vòng đời process của ứng dụng backend (khác với systemd/supervisor). Nó chỉ ngừng gửi traffic tới backend bị lỗi.

**9. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. `curl https://<IP>` sẽ không gửi SNI `shop.example.com` (hoặc gửi IP làm SNI). Server dùng chung IP sẽ trả về certificate mặc định hoặc certificate của virtual host khác; request có thể nhận lỗi xác thực hoặc nội dung không mong đợi. Đây không phải cách kiểm tra certificate của hostname cụ thể.
- **B)** Đáp án này đúng. `-servername shop.example.com` gửi SNI đúng như client thật, buộc server chọn certificate tương ứng với hostname đó. Output của `openssl s_client` cho thấy chuỗi certificate, `Verify return code`, giao thức và cipher đang dùng — đây là lệnh chuẩn để kiểm tra certificate khi nhiều domain chia sẻ một IP.
- **C)** Sai. `dig` chỉ truy vấn DNS và trả về bản ghi (A/AAAA/CNAME...), không liên quan tới certificate TLS và không hiển thị thông tin certificate.
- **D)** Sai. `nc -zv <IP> 443` chỉ kiểm tra TCP port 443 có mở (handshake TCP thành công), không thực hiện TLS handshake và không hiển thị certificate. Muốn xem certificate phải dùng công cụ hiểu TLS như `openssl s_client` hoặc `curl -v`.

**10. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** Đáp án này đúng. Cilium hỗ trợ WireGuard qua giá trị Helm (`encryption.enabled=true`, `encryption.type=wireguard`), còn Calico bật qua `FelixConfiguration` (`wireguardEnabled: true`). Cả hai mã hóa traffic giữa các node ở tầng kernel, hoàn toàn trong suốt với ứng dụng — không cần sửa code hay manifest của workload.
- **B)** Đáp án này đúng. Istio mTLS gắn identity dạng SPIFFE (ví dụ `cluster.local/ns/demo/sa/frontend`) với ServiceAccount của workload. `PeerAuthentication` mode STRICT từ chối mọi plaintext trong namespace, còn `AuthorizationPolicy` cho phép kiểm soát tầng L7 (method, path, identity nguồn) — mức kiểm soát mà WireGuard/IPsec không có.
- **C)** Sai. NetworkPolicy chỉ **lọc** traffic được phép (allow/deny theo selector và port); nó không mã hóa dữ liệu. Mã hóa phải được thực hiện bởi lớp khác như WireGuard/IPsec ở CNI hoặc mTLS ở service mesh.
- **D)** Sai. Bảo vệ traffic trên đường truyền vật lý giữa các node chính là mục đích cốt lõi của WireGuard: gói tin giữa các node được bọc trong tunnel đã mã hóa, nên kẻ nghe lén ở tầng underlay không đọc được nội dung. Ngược lại, Istio mTLS bảo vệ workload-to-workload (và chỉ trong mesh); đây là hai lớp bổ trợ nhau chứ không loại trừ nhau.
