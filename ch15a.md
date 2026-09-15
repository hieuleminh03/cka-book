---
layout: answer

title: "Chương 15"
subtitle: "GitOps & Deployment Strategies"
exam_objectives:
  - "Triển khai GitOps với ArgoCD và Flux."
  - "Áp dụng các chiến lược deploy: rolling, blue-green và canary."
  - "Rollback và quản lý release với Helm."
  - "Quản lý secret trong GitOps."
---

## Đáp án {#answers}

**1. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** Đáp án này đúng. Đây chính là pull model: agent (ArgoCD/Flux) chạy trong cluster, tự clone repo theo chu kỳ, và cluster chỉ cần credential đọc Git. CI không giữ kubeconfig của production nên nếu pipeline bị chiếm, kẻ tấn công cũng khó leo thang vào cluster.

- **B)** Đáp án này đúng. Vì mọi thay đổi trạng thái mong muốn đều phải qua commit/PR, bạn có lịch sử đầy đủ (ai đổi gì, khi nào, review bởi ai) và rollback chỉ là `git revert` — không phụ thuộc vào việc nhớ đúng cú pháp của từng công cụ.

- **C)** Đáp án này sai. Đối soát liên tục (continuously reconciled) là **một trong bốn nguyên lý bắt buộc** của GitOps, không phải tính năng tùy chọn. Nếu cluster chỉ được cập nhật lúc pipeline chạy mà không có vòng lặp đối soát, đó là push model mặc áo Git, không phải GitOps.

- **D)** Đáp án này sai. Push model thường **kém an toàn hơn** vì CI giữ credential ghi vào cluster với thời gian sống dài; việc "thu hồi khi pipeline kết thúc" không đúng với thực tế phần lớn hệ thống (secret được lưu cố định trong CI). Pull model không cần cấp quyền ghi cluster cho CI ngay từ đầu, và còn có khả năng phát hiện/sửa drift mà push model không có.



**2. Đáp án đúng là C.**

**Giải thích:**

- **A)** Đáp án này sai ở cả hai vế. `selfHeal: true` khiến ArgoCD đưa replicas về giá trị trong Git, và `prune` **không** mặc định tắt khi bạn đã khai báo `prune: true` trong `automated` — Deployment bị xóa khỏi Git sẽ bị xóa trên cluster.

- **B)** Đáp án này sai. `syncPolicy.automated` nghĩa là ArgoCD tự sync khi phát hiện OutOfSync; không cần ai bấm `argocd app sync`. Đây là điểm khác biệt cốt lõi giữa app có auto-sync và app chỉ sync thủ công.

- **C)** Đáp án này đúng. Hai cơ chế chạy cùng nhau trong vòng đối soát: selfHeal phát hiện replicas bị sửa ngoài Git và đưa về đúng manifest (ví dụ 2 replicas như trong Git), còn prune phát hiện resource đã biến mất khỏi Git và xóa nó khỏi cluster. Kết quả cuối cùng là cluster khớp hoàn toàn với Git.

- **D)** Đáp án này sai. ArgoCD không "xóa trước rồi đối soát" và kết quả không phụ thuộc thứ tự như mô tả. Việc prune và self-heal đều dựa trên cùng một phép so sánh desired vs live; thứ tự thực thi giữa các resource có thể khác nhau (và có thể điều khiển bằng sync wave), nhưng trạng thái hội tụ vẫn là: replicas theo Git, Deployment bị xóa.



**3. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đáp án này đúng. `argocd-repo-server` là thành phần clone repository, cache và render manifest — nó gọi `helm template` cho chart, `kustomize build` cho overlay, hoặc render Jsonnet/plain YAML. Đây là nơi "build" desired state trước khi controller so sánh với cluster.

- **B)** Đáp án này sai. `argocd-application-controller` so sánh desired state (đã render) với live state, quyết định và thực thi sync, đồng thời đánh giá health. Nó tiêu thụ output của repo-server, không tự clone hay render.

- **C)** Đáp án này sai. `argocd-dex-server` là thành phần SSO (OIDC/SAML/LDAP), không liên quan tới việc render manifest hay quản lý certificate như câu này mô tả.

- **D)** Đáp án này sai. `argocd-server` phục vụ API/UI, xử lý session và là điểm vào của RBAC; nó không clone repo và không tính diff. Việc tách vai trò như vậy giúp repo-server có thể scale ngang và giới hạn rủi ro khi render template.



**4. Đáp án đúng là D.**

**Giải thích:**

- **A)** Đáp án này sai. Helm không xóa revision khỏi history khi rollback. History vẫn giữ đủ các revision (4, 5, và cả 6 sau khi rollback) để bạn có thể tra cứu hoặc rollback tiếp.

- **B)** Đáp án này sai. `helm rollback` re-apply toàn bộ manifest của revision đích (Deployment, Service, ConfigMap...), không chỉ đổi image tag trong values. Nó là một thao tác ở tầng release, không phải một lần chỉnh values.

- **C)** Đáp án này sai. `--rollback-on-failure` (tên cũ `--atomic`) chỉ điều khiển hành vi **tự động rollback khi upgrade thất bại**; nó không phải điều kiện để `helm rollback` hoạt động. Release không dùng flag này vẫn rollback bình thường.

- **D)** Đáp án này đúng. `helm rollback web 3` tạo một revision mới (revision 6) với nội dung của revision 3 — theo đúng tinh thần "rollback cũng là một lần deploy". Vì revision 5 vẫn nằm trong history, bạn có thể rollback "tiến" về revision 5 nếu bản 3 cũng có vấn đề.



**5. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** Đáp án này đúng. `helm.sh/hook-weight` quyết định thứ tự chạy giữa các hook (số nhỏ chạy trước, cùng weight thì thứ tự không được đảm bảo nên best practice là đặt weight rõ ràng). Với `hook-delete-policy`, mặc định Helm áp dụng `before-hook-creation` — xóa resource của hook cũ trước khi tạo hook mới.

- **B)** Đáp án này đúng và đây là cạm bẫy kinh điển: hook resource **không** được Helm track như tài nguyên của release. `helm uninstall` không dọn chúng; muốn dọn phải dùng `hook-delete-policy` (`hook-succeeded`, `hook-failed`) hoặc TTL trên Job.

- **C)** Đáp án này sai ở vế sau. ArgoCD có map `pre-upgrade` thành `PreSync` và `pre-install` cũng thành `PreSync`, nhưng ArgoCD **không phân biệt được** install với upgrade — mọi thao tác đều là "sync" — nên hai hook này có thể chạy cùng nhau. Đây là lý do hook dùng với ArgoCD phải idempotent.

- **D)** Đáp án này sai. Hook không được ghi vào revision như tài nguyên thường; rollback release không tự xóa hook đã tạo. Ngược lại, chính vì không bị quản lý nên hook còn có thể tồn tại "mồ côi" sau khi release bị gỡ, gây nhầm lẫn nếu không đặt delete policy.



**6. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** Đáp án này đúng. Kustomize không có ngôn ngữ template: nó lấy manifest YAML gốc làm base rồi áp overlay/patch (`replicas`, `images`, `namePrefix`, `patches`...). Helm ngược lại là template Go đầy đủ với `if`, `range`, hàm và pipeline.

- **B)** Đáp án này đúng. `kubectl kustomize` và `kubectl apply -k` có sẵn khi bạn có kubectl, chạy hoàn toàn offline; Helm cũng render offline bằng `helm template`. Đây là thao tác nên đưa vào CI để bắt lỗi manifest trước khi merge.

- **C)** Đáp án này sai. Không có cơ sở nào để nói Kustomize không dùng được cho production — nó được dùng rộng rãi (Config Sync, ArgoCD, Flux đều hỗ trợ native). Việc không đóng gói package có version là một hạn chế về phân phối, nhưng bù lại manifest cuối dễ đọc và diff chính xác.

- **D)** Đáp án này sai. Hai công cụ kết hợp được: Kustomize có `helmCharts` generator để render chart rồi patch tiếp; Flux đặt `HelmRelease` bên trong `Kustomization`; ArgoCD cũng có thể render Helm rồi áp overlay tùy cách tổ chức repo. Chọn "hoặc Helm hoặc Kustomize" cho toàn hệ thống là quan điểm quá cứng nhắc.



**7. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đáp án này đúng. `maxUnavailable: 0` đảm bảo không pod cũ nào bị xóa trước khi có pod mới Ready (Kubernetes tôn trọng readiness probe), `maxSurge: 1` cho phép thêm đúng một pod tạm thời — khớp với yêu cầu "thêm tối đa một pod". Kèm readiness probe đúng và `preStop`/graceful shutdown để request đang bay không bị cắt, đây là cấu hình rolling update zero-downtime tiêu chuẩn cho production.

- **B)** Đáp án này sai. Recreate **gây downtime** vì toàn bộ pod cũ bị tắt trước khi pod mới được tạo. Ưu điểm duy nhất của nó là không bao giờ có hai version chạy song song, nhưng với yêu cầu zero-downtime thì đây không phải lựa chọn.

- **C)** Đáp án này sai. `maxSurge: 0` + `maxUnavailable: 100%` cho phép xóa toàn bộ pod cùng lúc — rollout nhanh nhưng mất sạch capacity, chắc chắn downtime. Đây là cấu hình gần với Recreate hơn là rolling update.

- **D)** Đáp án này sai ở hai điểm. Mặc định 25%/25% vẫn cho phép thiếu tối đa 25% pod, tức chưa đảm bảo "luôn Ready trước khi pod cũ bị xóa"; và Kubernetes **không tự biết** pod sẵn sàng nếu không có readiness probe (thiếu probe, pod được coi là Ready ngay khi container chạy, dẫn tới traffic bị gửi khi app chưa phục vụ được).



**8. Đáp án đúng là C.**

**Giải thích:**

- **A)** Đáp án này sai. Recreate gây downtime và test trên staging không kiểm chứng được hành vi với lưu lượng thật. Câu này còn vi phạm yêu cầu "tự rollback theo ngưỡng lỗi" vì Recreate không có cơ chế phân tích metric.

- **B)** Đáp án này sai. Blue-green chuyển 100% traffic trong một bước (all or nothing), không chia 5% để quan sát dần, và cơ chế "theo dõi rồi đổi selector lại" là rollback thủ công chứ không tự động theo ngưỡng.

- **C)** Đáp án này đúng. Argo Rollouts cho phép chia traffic theo bước (`setWeight` + `trafficRouting` qua Istio/NGINX/ALB...), đồng thời chạy `AnalysisTemplate` truy vấn Prometheus để đánh giá success rate/latency; vượt `failureLimit` thì tự động abort và quay về stable. Đây đúng là progressive delivery có kiểm soát rủi ro theo phần trăm.

- **D)** Đáp án này sai. `partition` của StatefulSet chỉ điều khiển việc update pod theo thứ tự (0..N hoặc N..0), không chia lưu lượng theo phần trăm, không có khái niệm canary weight và không có phân tích metric để tự rollback.



**9. Đáp án đúng là A.**

**Giải thích:**

- **A)** Đáp án này đúng. `argocd app rollback` quay lại một mốc trong history, nhưng application-controller vẫn tiếp tục đối chiếu desired state với `targetRevision` trong Git (nhánh `main` vẫn đang chứa commit lỗi) nên self-heal sẽ đưa bản lỗi trở lại. Cách xử lý đúng là revert commit trên Git (hoặc, trong tình huống khẩn cấp, tắt auto-sync bằng `argocd app set <app> --sync-policy none`, sửa/khôi phục trực tiếp rồi cập nhật Git và bật lại).

- **B)** Đáp án này sai. Cache của repo-server chỉ ảnh hưởng tới việc render manifest; việc bản lỗi quay lại là do vòng lặp reconcile chứ không phải cache bẩn. Restart repo-server không thay đổi desired state.

- **C)** Đáp án này sai. `prune` liên quan tới việc xóa resource không còn trong Git, không liên quan tới việc rollback bị ghi đè. Bật prune thậm chí còn có thể làm tình hình phức tạp hơn nếu resource bị xóa ngoài mong muốn.

- **D)** Đáp án này sai. `allowEmpty` chỉ cho phép sync khi manifest render ra rỗng (tránh vô tình xóa sạch app). Nó không liên quan gì tới việc `rollback` bị auto-sync/self-heal ghi đè.



**10. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. Sealed Secrets dùng cặp khóa của controller: `kubeseal` mã hóa bằng public key, controller trong cluster giải mã bằng private key. Ciphertext (`SealedSecret`) commit được vào Git. Mặc định scope là `strict`, nghĩa là secret gắn với cả tên và namespace — đổi tên/chuyển namespace phải seal lại.

- **B)** Đáp án này đúng. SOPS mã hóa value theo `creation_rules` trong `.sops.yaml` (có thể giới hạn bằng `encrypted_regex` để chỉ mã hóa `data`/`stringData`), giữ nguyên key nên diff vẫn đọc được. Flux giải mã native qua `spec.decryption` của Kustomization, còn ArgoCD cần plugin (ksops, argocd-vault-plugin) hoặc để ESO quản lý secret.

- **C)** Đáp án này sai và là hiểu nhầm nguy hiểm. Sau khi được giải mã, secret tồn tại **dạng plaintext trong etcd** (và trong memory của container). Vì vậy vẫn bắt buộc phải bật encryption at rest, giới hạn RBAC đọc Secret, và hạn chế mount/env cho workload — mã hóa trong Git không thay thế được các lớp bảo vệ trong cluster.

- **D)** Đáp án này đúng. ESO cho phép "không có ciphertext trong Git": `SecretStore`/`ClusterSecretStore` khai báo cách kết nối provider, `ExternalSecret` khai báo secret cần lấy và `refreshInterval` để tự đồng bộ khi giá trị ở Vault/AWS Secrets Manager thay đổi. Secret sau khi đồng bộ vẫn là Secret Kubernetes trong etcd nên vẫn phải kiểm soát truy cập và bật encryption at rest (xem lại chương 9).

