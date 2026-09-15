---
layout: answer

title: "Chương 14"
subtitle: "Observability"
exam_objectives:
  - "Thu thập và truy vấn metrics với Prometheus/PromQL."
  - "Trực quan hóa và alerting với Grafana và Alertmanager."
  - "Logging tập trung (Loki/ELK) và distributed tracing với OpenTelemetry."
  - "Định nghĩa SLI/SLO và xây dựng chiến lược alerting hiệu quả."
---

## Đáp án {#answers}

**1. Đáp án đúng là B.**

**Giải thích:**

- **A)** Biểu thức này so sánh trực tiếp giá trị counter thô của hai selector khác nhau (`code=~"5.."` và toàn bộ request).
  - Đáp án này sai vì counter thô là số tích lũy từ lúc process khởi động, không phải tốc độ; nếu process restart, giá trị reset về 0. Ngoài ra hai vế có label set khác nhau nên phép chia phải qua vector matching, kết quả không còn ý nghĩa "tỉ lệ lỗi trong 5 phút".

- **B)** `sum(rate(http_requests_total{code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.02`
  - Đây là đáp án đúng. `rate(...[5m])` chuẩn hóa counter thành tốc độ request/giây (tự xử lý reset), `sum()` gộp toàn bộ series cùng loại, rồi chia cho tổng request để ra tỉ lệ lỗi. So với `0.02` chính là ngưỡng 2%. Đây là dạng biểu thức chuẩn cho error rate và cũng là nền tảng của alert theo SLO.

- **C)** `avg(http_requests_total{code=~"5.."}) > 0.02`
  - Đáp án này sai vì lấy trung bình giá trị counter thô — con số tích lũy có thể lên hàng triệu, chẳng liên quan tới tỉ lệ lỗi. Counter phải đi qua `rate()`/`increase()` trước khi dùng.

- **D)** `sum(increase(http_requests_total{code=~"5.."}[5m])) / sum(http_requests_total) > 0.02`
  - Đáp án này sai vì tử số là số lỗi trong 5 phút còn mẫu số là counter tích lũy từ trước tới nay (không có cửa sổ thời gian). Tỉ lệ giữa hai đại lượng lệch cửa sổ như vậy không phải error rate; mẫu số phải cũng được tính trên cùng cửa sổ 5 phút bằng `rate()`.




**2. Đáp án đúng là B, C và D.**

**Giải thích:**

- **A)** cAdvisor chạy như một DaemonSet riêng trong namespace `kube-system`.
  - Đáp án này sai. cAdvisor không phải DaemonSet riêng: nó được tích hợp sẵn trong kubelet từ Kubernetes 1.7, expose metric tại endpoint `/metrics/cadvisor` của kubelet. kube-prometheus-stack chỉ cấu hình Prometheus scrape endpoint này.

- **B)** node-exporter chạy dạng DaemonSet và cung cấp metric mức host của từng node.
  - Đáp án này đúng. node-exporter đọc dữ liệu từ `/proc`, `/sys` (qua mount và hostNetwork) để cho các metric như `node_cpu_seconds_total`, `node_memory_MemAvailable_bytes`, `node_filesystem_avail_bytes`. DaemonSet đảm bảo có đúng một instance trên mỗi node.

- **C)** kube-state-metrics đọc Kubernetes API để sinh metric về trạng thái object như replica thiếu, pod restart.
  - Đáp án này đúng. kube-state-metrics không đo tài nguyên; nó chuyển trạng thái object (deployment, pod, job...) thành metric như `kube_deployment_status_replicas_unavailable`, `kube_pod_container_status_restarts_total`. Đây là nguồn cho các alert về trạng thái mong muốn/thực tế.

- **D)** kubelet expose metric container của cAdvisor tại endpoint `/metrics/cadvisor`.
  - Đáp án này đúng. kubelet phục vụ metric cAdvisor tại `/metrics/cadvisor`, và Prometheus trong kube-prometheus-stack scrape endpoint này để lấy `container_cpu_usage_seconds_total`, `container_memory_working_set_bytes`... của từng container.




**3. Đáp án đúng là B.**

**Giải thích:**

- **A)** `histogram_quantile(0.95, rate(http_request_duration_seconds_sum[5m]))`
  - Đáp án này sai. `_sum` chỉ là tổng thời gian của tất cả request, không chứa thông tin phân bố; `histogram_quantile` yêu cầu series `_bucket` kèm label `le`. Dùng `_sum` sẽ lỗi hoặc cho kết quả vô nghĩa.

- **B)** `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))`
  - Đây là đáp án đúng. `rate()` trên từng bucket cho tốc độ quan sát theo từng ngưỡng `le`, `sum by (le)` gộp các pod/instance nhưng vẫn giữ label `le` mà `histogram_quantile` cần, rồi hàm này nội suy ra p95. Đây là công thức chuẩn cho classic histogram và là câu hỏi rất hay gặp.

- **C)** `quantile(0.95, http_request_duration_seconds)`
  - Đáp án này sai. `quantile()` của PromQL tính phân vị trên tập giá trị instant vector giữa các series (ví dụ giữa các pod), không phải trên phân bố bucket của latency. Nó không thay thế được `histogram_quantile`.

- **D)** `avg by (pod) (http_request_duration_seconds{quantile="0.95"})`
  - Đáp án này sai theo hai điểm: label `quantile` thuộc về summary (không có trên classic histogram), và lấy trung bình các quantile là phép toán không hợp lệ về mặt thống kê. Muốn quantile theo pod với histogram, phải dùng `histogram_quantile(0.95, sum by (le, pod) (...))`.




**4. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Alert ở trạng thái `pending` khi biểu thức đã true nhưng chưa đủ thời gian `for`.
  - Đáp án này đúng. Vòng đời là `inactive → pending → firing`: khi lần đánh giá đầu tiên thấy biểu thức true, alert chuyển sang `pending` và chỉ chuyển `firing` nếu true liên tục suốt `for`. Nếu biểu thức false trước khi hết `for`, alert quay lại `inactive` mà không gửi gì.

- **B)** Alert `pending` cũng được gửi tới Alertmanager ngay để kịp grouping.
  - Đáp án này sai. Prometheus chỉ gửi alert đã `firing` (và thông báo `resolved`) tới Alertmanager. `pending` là trạng thái nội bộ của Prometheus; nó chưa tạo notification nào.

- **C)** Chỉ alert `firing` (và thông báo `resolved`) mới được gửi tới Alertmanager.
  - Đáp án này đúng. Alertmanager là nơi grouping/routing/silence, nhưng chỉ nhận được alert khi Prometheus đã chuyển `firing` hoặc khi alert resolved. Đây là lý do vì sao `for` được xem như "hàng rào lọc nhiễu" trước khi alert đi vào hệ thống notification.

- **D)** `for: 30s` nghĩa là Alertmanager gửi lại notification mỗi 30 giây.
  - Đáp án này sai và là sự nhầm lẫn kinh điển giữa hai hệ thống: `for` thuộc alert rule của Prometheus (thời gian giữ true trước khi firing), còn việc gửi lại notification do `repeat_interval` (và `group_interval`) của Alertmanager quyết định, với giá trị mặc định điển hình là 4 giờ, không phải 30 giây.




**5. Đáp án đúng là B.**

**Giải thích:**

- **A)** Do số node tăng; giảm bằng cách tăng `scrape_interval` cho toàn cluster.
  - Đáp án này sai. Tăng số node không thể làm số series tăng gấp 10 lần. Tăng `scrape_interval` chỉ giảm số sample theo thời gian, **không** giảm số series duy nhất — mà cardinality (số series) mới là thứ tiêu tốn RAM/disk và làm query chậm.

- **B)** Do label cardinality cao (path chứa ID, user_id, error message); xử lý bằng metric relabeling/drop label và aggregate trong recording rule.
  - Đây là đáp án đúng. Label chứa giá trị động (ID, message, tham số URL) nhân số series lên theo cấp số nhân. Cách xử lý đúng gồm: `metric_relabel_configs` để drop metric/label không cần, không gắn dữ liệu động vào label, và dùng recording rule `sum by (...)` để loại label thừa sau khi aggregate. Theo dõi bằng `prometheus_tsdb_head_series` và `topk` theo `__name__`.

- **C)** Do histogram bucket chia quá thô; xử lý bằng cách bỏ `sum by (le)` khi tính quantile.
  - Đáp án này sai. Bucket thô hay mịn không làm tăng số series lên 3 triệu, và bỏ `sum by (le)` là phá biểu thức quantile đúng (xem câu 3) chứ không giảm cardinality.

- **D)** Cardinality không ảnh hưởng RAM/disk vì TSDB nén rất tốt; chỉ cần tăng retention.
  - Đáp án này sai. Mỗi series đều chiếm entry trong head block và ảnh hưởng index; tăng retention chỉ làm tình hình tệ hơn. Cardinality là yếu tố số một quyết định chi phí Prometheus.




**6. Đáp án đúng là B.**

**Giải thích:**

- **A)** Page khi error rate > 1% trong 1 phút, vì càng nhạy càng tốt.
  - Đáp án này sai. Cửa sổ 1 phút với ngưỡng thấp tạo rất nhiều false positive do nhiễu (spike nhỏ, deploy lăn bánh). Alert kiểu này nhanh chóng gây alert fatigue và không gắn với error budget của SLO.

- **B)** Dùng multi-window multi-burn-rate: burn rate 14.4 (1h + 5m) để page, burn rate 6 (6h + 30m) để ticket, burn rate 1 (3d) để theo dõi xu hướng; tính tỉ lệ bằng recording rule.
  - Đây là đáp án đúng. Multi-window multi-burn-rate là khuôn mẫu chuẩn (Google SRE): cửa sổ dài đảm bảo mức độ nghiêm trọng thật, cửa sổ ngắn đảm bảo tốc độ phát hiện, kết hợp bằng phép `and` để giảm false positive. Tính trước tỉ lệ bằng recording rule giúp alert nhanh, nhất quán và tái sử dụng cho dashboard error budget.

- **C)** Alert khi CPU > 80% và memory > 80% vì đó là nguyên nhân gốc.
  - Đáp án này sai. Đây là alert theo nguyên nhân, không theo triệu chứng: tài nguyên cao chưa chắc ảnh hưởng người dùng, và ngược lại sự cố có thể xảy ra khi tài nguyên thấp. Loại alert này nên đưa vào dashboard/ticket trừ khi đã chứng minh luôn dẫn tới sự cố.

- **D)** Giãn `repeat_interval` lên 24 giờ và hạ `severity` mọi alert xuống warning để tránh alert fatigue.
  - Đáp án này sai. Đây là cách "che" vấn đề: alert quan trọng vẫn cần page đúng lúc, còn alert nhiễu cần được sửa hoặc xóa. Giãn repeat và hạ severity đồng loạt có thể khiến sự cố nghiêm trọng không được xử lý kịp.




**7. Đáp án đúng là A, B và C.**

**Giải thích:**

- **A)** Loki chỉ index label, không index toàn văn bản nội dung log.
  - Đáp án này đúng. Loki index label của stream; nội dung log được nén thành chunk và tìm bằng cách quét khi truy vấn. Nhờ vậy dung lượng index nhỏ hơn nhiều so với inverted index toàn văn bản của Elasticsearch, đổi lại truy vấn phụ thuộc vào việc chọn label tốt.

- **B)** Loki thường rẻ hơn ELK ở cùng volume log, nhưng truy vấn phụ thuộc mạnh vào chất lượng label.
  - Đáp án này đúng. Chi phí lưu trữ và CPU của Loki thấp hơn khi volume lớn, nhưng nếu label không chọn lọc tốt (ví dụ query quét toàn bộ cluster), Loki phải đọc rất nhiều chunk và trở nên chậm. Kỷ luật label (`namespace`, `app`, `container`) là điều kiện tiên quyết khi vận hành Loki.

- **C)** Elasticsearch cho tìm kiếm full-text và aggregation mạnh hơn, đổi lại chi phí lưu trữ/vận hành cao hơn.
  - Đáp án này đúng. Inverted index toàn văn bản cho phép Query DSL, aggregation, relevance search rất mạnh — phù hợp compliance/BI trên log — nhưng cần quản lý shard, mapping, heap, ILM và tốn tài nguyên hơn hẳn so với Loki.

- **D)** Loki bắt buộc gom toàn bộ log vào một index duy nhất nên luôn tìm nhanh hơn Elasticsearch trong mọi trường hợp.
  - Đáp án này sai. Loki không có khái niệm "một index duy nhất"; dữ liệu được chia theo stream label và chunk. Tốc độ tìm kiếm phụ thuộc vào selector label truy vấn — với truy vấn full-text không chọn lọc, Elasticsearch thường nhanh hơn nhiều.




**8. Đáp án đúng là A.**

**Giải thích:**

- **A)** `sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="demo", container!=""}[5m]))`
  - Đây là đáp án đúng. `container_cpu_usage_seconds_total` là counter (giây CPU tích lũy) nên phải qua `rate()` để ra cores đang dùng; `container!=""` loại các series tổng hợp mức pod và các mục không phải container thật; `sum by (pod)` gộp các series trùng (cgroup/container id) thành một con số cho mỗi pod. Đây là dạng query chuẩn để tính CPU theo pod.

- **B)** `avg(container_cpu_usage_seconds_total{namespace="demo"})`
  - Đáp án này sai vì lấy trung bình giá trị counter thô — kết quả vô nghĩa và không phải tốc độ sử dụng CPU.

- **C)** `rate(container_cpu_usage_seconds_total{namespace="demo"}[5m])`
  - Đáp án này sai vì không lọc `container!=""` và không aggregate theo pod. Kết quả trả về nhiều series (pod-level, infra container, các cgroup khác nhau), dễ gây đếm trùng và không trả lời câu hỏi "pod nào dùng bao nhiêu CPU".

- **D)** `irate(container_cpu_usage_seconds_total{namespace="demo"}[1m])`
  - Đáp án này sai theo hai điểm: `irate` chỉ dùng hai sample cuối nên rất nhiễu, không phù hợp để đo mức sử dụng ổn định; và vẫn thiếu lọc/aggregate nên kết quả bị phân mảnh theo cgroup. `irate` dùng cho dashboard "tức thời", còn tính toán/alert nên dùng `rate`.




**9. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** `group_wait` là thời gian chờ gom thêm alert trước khi gửi notification đầu tiên cho một group mới.
  - Đáp án này đúng. Khi group mới xuất hiện (theo `group_by`), Alertmanager chờ `group_wait` để các alert cùng đợt được gom chung, sau đó mới gửi notification đầu tiên. Đây là lý do alert có thể đến muộn hơn vài chục giây so với thời điểm firing.

- **B)** `repeat_interval` là khoảng cách gửi lại notification khi alert vẫn chưa resolve và group không có thay đổi mới.
  - Đáp án này đúng. Sau lần gửi đầu, nếu alert vẫn firing và không có alert mới/thay đổi, Alertmanager gửi lại sau `repeat_interval` (mặc định 4h). Đây là cơ chế nhắc nhở người trực, khác hoàn toàn với `for` của Prometheus.

- **C)** `group_interval` là tần suất Prometheus đánh giá alert rule.
  - Đáp án này sai. Tần suất đánh giá rule do `evaluation_interval` của Prometheus quyết định; `group_interval` là khoảng cách tối thiểu giữa hai lần gửi notification khi group có alert mới hoặc thay đổi.

- **D)** `group_wait` là khoảng thời gian tối đa một alert được phép ở trạng thái pending trước khi bị hủy.
  - Đáp án này sai. Không có cơ chế "hủy pending" nào theo `group_wait`: thời gian một alert ở `pending` do `for` của alert rule quyết định, còn `group_wait` là khái niệm của Alertmanager sau khi alert đã firing.




**10. Đáp án đúng là B.**

**Giải thích:**

- **A)** Collector thay thế SDK trong ứng dụng và tự sinh trace từ mọi service mà không cần instrumentation.
  - Đáp án này sai. Collector không sinh telemetry thay ứng dụng: muốn có trace, service vẫn phải được instrumentation (SDK hoặc auto-instrumentation) để tạo span. Collector chỉ là thành phần nhận, xử lý và chuyển tiếp dữ liệu.

- **B)** Collector nhận telemetry qua OTLP (gRPC 4317 hoặc HTTP 4318), xử lý trong pipeline receivers → processors → exporters, có thể chạy dạng agent hoặc gateway.
  - Đây là đáp án đúng. Đây là mô hình cốt lõi của OpenTelemetry Collector: nhận dữ liệu qua receiver (phổ biến nhất là OTLP với gRPC 4317 và HTTP/protobuf 4318), xử lý qua processor (batch, memory_limiter, k8sattributes, tail_sampling...), rồi export tới backend. Kiểu agent (DaemonSet) gom gần nguồn; kiểu gateway (Deployment) xử lý tập trung — nhiều hệ thống dùng cả hai.

- **C)** OTLP chỉ hỗ trợ gRPC nên mọi backend bắt buộc phải mở cổng 4317.
  - Đáp án này sai. OTLP có hai transport chính thức: gRPC (mặc định cổng 4317) và HTTP/protobuf (mặc định cổng 4318); nhiều SDK mặc định dùng HTTP và nhiều backend nhận cả hai. Cổng chỉ là quy ước mặc định, có thể cấu hình lại.

- **D)** Tail-based sampling luôn được thực hiện trong SDK để giảm tải cho Collector.
  - Đáp án này sai. Tail-based sampling cần thấy toàn bộ trace trước khi quyết định, nên phải chạy ở Collector (processor `tail_sampling`), không phải SDK. SDK chỉ làm head-based sampling vì tại thời điểm bắt đầu request chưa biết trace có lỗi hay chậm hay không.

