---
layout: answer

title: "Chương 16"
subtitle: "SRE & Incident Response"
exam_objectives:
  - "Xây dựng on-call, runbook và quy trình xử lý incident."
  - "Thực hiện root cause analysis và postmortem không đổ lỗi."
  - "Áp dụng chaos engineering và kiểm thử disaster recovery."
  - "Capacity planning và trade-off giữa reliability và chi phí."
---

## Đáp án {#answers}

**1. Đáp án đúng là C.**

**Giải thích:**

- **A)** Đáp án này sai. Nâng SLO lên 99.99% trong lúc budget đã tiêu hao 80% còn làm ngân sách lỗi **nhỏ hơn** (0.01% thay vì 0.1%), nghĩa là đội càng dễ vi phạm mục tiêu hơn. SLO là cam kết giữa business và engineering dựa trên chi phí - trải nghiệm, không phải công cụ để "thúc" đội kỹ thuật tập trung.

- **B)** Đáp án này sai. Budget còn 20% không có nghĩa là được tiếp tục deploy 4 feature lớn: chính sách budget thấp được thiết kế để giảm rủi ro trước khi cháy hết ngân sách. Tiếp tục như bình thường sẽ biến sự suy giảm thành vi phạm SLO và bào mòn niềm tin với người dùng.

- **C)** Đáp án này đúng. Đây chính là vận hành error budget policy: khi budget còn dưới ngưỡng cảnh báo (thường là 25%), hạn chế thay đổi rủi ro cao, dồn nguồn lực cho công việc reliability, chỉ deploy hotfix/rollback, và thống nhất thứ tự ưu tiên với product. Quyết định được đưa ra dựa trên số liệu thay vì tranh luận cảm tính.

- **D)** Đáp án này sai theo hai hướng. Thứ nhất, ngừng mọi deploy kể cả hotfix bảo mật là cực đoan và có thể gây thiệt hại lớn hơn. Thứ hai, không có khái niệm "reset budget rồi làm lại": budget được tính trên cửa sổ trượt, hành vi cũ sẽ vẫn phản ánh vào chỉ số.




**2. Đáp án đúng là A và D.**

**Giải thích:**

- **A)** Đáp án này đúng. Tỉ lệ lỗi vượt ngưỡng SLO kèm burn rate cao ở cả cửa sổ ngắn và dài là tín hiệu người dùng đang bị ảnh hưởng ngay lúc này — phải page. Burn rate cao còn cho biết nếu không xử lý, budget sẽ cháy hết trong vài giờ.

- **B)** Đáp án này sai. Disk 71% là sự kiện hạ tầng, chưa ảnh hưởng người dùng và còn nhiều thời gian xử lý — phù hợp với ticket. Chỉ nên page khi có nguy cơ cạn disk trong thời gian ngắn (dự báo dưới vài giờ) hoặc disk full thật.

- **C)** Đáp án này sai. Certificate còn 30 ngày là việc cần lên lịch gia hạn, có thể theo dõi bằng ticket/calendar; page nó chỉ tạo thêm nhiễu và góp phần gây alert fatigue.

- **D)** Đáp án này đúng. Mất toàn bộ replica Ready trong 2 phút đồng nghĩa availability về 0 — tác động trực tiếp và nghiêm trọng, phải page ngay. Nguyên tắc chung: page theo triệu chứng người dùng, ticket theo sự kiện hạ tầng.




**3. Đáp án đúng là B, C và D.**

**Giải thích:**

- **A)** Đáp án này sai. Incident Commander phải giữ tầm nhìn tổng thể, điều phối nguồn lực, ra quyết định ưu tiên và là đầu mối thông tin duy nhất. Nếu IC tự nhảy vào debug, đội sẽ mất người điều phối, timeline không được ghi, comms bị bỏ quên — đúng loại sai lầm khiến incident kéo dài.

- **B)** Đáp án này đúng. IC không cần là người giỏi nhất về kỹ thuật trong sự cố, nhưng phải là người quyết định và giữ mọi người tập trung vào mục tiêu giảm tác động.

- **C)** Đáp án này đúng. Scribe ghi timeline, quyết định, giả thuyết và hành động — đây là nguyên liệu đầu vào không thể thay thế cho postmortem. Không có scribe, dữ kiện sẽ mất sau vài giờ căng thẳng.

- **D)** Đáp án này đúng. Với đội nhỏ không thể đủ 5 vai trò, hãy gộp (IC vừa comms vừa ghi chú chẳng hạn) nhưng **không gộp IC với người trực tiếp sửa hệ thống**: người đang debug bị cuốn vào chi tiết và dễ quên điều phối.




**4. Đáp án đúng là B.**

**Giải thích:**

- **A)** Đáp án này sai. Đây là phản xạ "tìm nguyên nhân trước, mitigate sau" — trong lúc đọc code và bật debug log, 15% người dùng vẫn đang gặp lỗi. Bật debug log còn có thể làm tăng tải và tăng nhiễu dữ liệu ngay giữa sự cố.

- **B)** Đáp án này đúng. Deploy vừa xảy ra là nghi phạm số một; rollback là hành động giảm tác động nhanh nhất, ít rủi ro nhất và có thể đảo ngược. Song song đó đội thu thập log/timeline để điều tra nguyên nhân sau — mitigation trước, root cause sau.

- **C)** Đáp án này sai. Restart toàn bộ database là hành động có bán kính ảnh hưởng lớn, có thể gây mất kết nối diện rộng và không có bằng chứng liên quan. Đây là ví dụ của "hành động quá tay" khi chưa hiểu vấn đề.

- **D)** Đáp án này sai. Scale gấp đôi không chắc giảm tỉ lệ lỗi (lỗi 500 sau deploy thường do code/config), làm tăng chi phí, tăng tải lên dependency và trì hoãn hành động có hiệu quả rõ ràng là rollback.




**5. Đáp án đúng là B và D.**

**Giải thích:**

- **A)** Đáp án này sai. 5 Whys giả định một chuỗi nguyên nhân tuyến tính, trong khi sự cố thật thường có nhiều điều kiện góp phần. Cần bổ sung timeline (để thấy khoảng trống phát hiện/phản ứng) và fishbone (để quét nhiều nhánh nguyên nhân), cùng việc kiểm tra từng lớp phòng thủ vì sao không hoạt động.

- **B)** Đáp án này đúng. Trigger (deploy, config change, một sự kiện bên ngoài) chỉ là ngòi nổ; điều đáng sửa là các điều kiện hệ thống cho phép trigger gây hậu quả lớn như thiếu test nhánh lỗi, alert phát hiện chậm, không có canary hoặc rollback tự động.

- **C)** Đáp án này sai. Kết luận "engineer X chạy lệnh sai" là blame và chấm dứt RCA quá sớm. Nếu một thao tác tay có thể gây sự cố, vấn đề nằm ở guardrail thiếu (dry-run, phân quyền, cảnh báo), không phải ở phẩm chất cá nhân.

- **D)** Đáp án này đúng. Blameless nghĩa là tìm "second story": vì sao hệ thống cho phép hành động đó gây sự cố và cần thay đổi gì. Cách này giữ được sự minh bạch — mọi người dám khai báo lỗi và cung cấp đủ dữ kiện cho postmortem.




**6. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Đáp án này đúng. Action item phải cụ thể, có owner, deadline và ticket trong cùng hệ thống công việc với feature. Một câu như "cải thiện monitoring" không thể theo dõi và gần như chắc chắn không được làm.

- **B)** Đáp án này sai. Danh sách 20–30 action item nghe có vẻ bao quát nhưng thực tế không ai hoàn thành; đội mất niềm tin vào postmortem và các việc quan trọng bị chìm. Kinh nghiệm chung là giới hạn 3–5 việc ưu tiên cao nhất.

- **C)** Đáp án này đúng. Giới hạn số lượng giúp tập trung, còn việc theo dõi tỉ lệ hoàn thành trong review hàng tháng biến postmortem thành công việc thật thay vì văn bản lưu trữ. Tỉ lệ thấp là tín hiệu quy trình đang hình thức.

- **D)** Đáp án này sai. Bắt đầu bằng việc xác định cá nhân chịu trách nhiệm là postmortem kiểu buộc tội, đi ngược blameless culture và khiến mọi người che giấu thông tin ở sự cố sau. Mục tiêu là cải thiện hệ thống, không phải tìm người để phạt.



**7. Đáp án đúng là B và C.**

**Giải thích:**

- **A)** Đáp án này sai. `mode: all` trên toàn namespace production là blast radius lớn nhất có thể, có nguy cơ làm sập dịch vụ thật. Chaos phải bắt đầu từ thí nghiệm nhỏ nhất có ý nghĩa rồi mở rộng dần khi giả thuyết được xác nhận.

- **B)** Đáp án này đúng. `mode: one` kết hợp `labelSelectors` hẹp giới hạn số đối tượng bị ảnh hưởng, giúp thí nghiệm an toàn và dễ quan sát. Đây là cách chuẩn để chạy thí nghiệm đầu tiên, lý tưởng là trên staging trước production.

- **C)** Đáp án này đúng. `duration` rõ ràng, khả năng xóa/chặn experiment (kill switch) và người theo dõi trong suốt thời gian chạy là các guardrail bắt buộc. Thí nghiệm không có kill switch là sự cố đang chờ xảy ra.

- **D)** Đáp án này sai. etcd/control plane là thành phần giữ toàn bộ trạng thái cluster; làm hỏng nó trong cluster chưa HA có thể khiến bạn mất cả cluster. Chaos trên control plane chỉ nên thực hiện khi đã HA, có backup kiểm chứng và quy trình phục hồi đã diễn tập.




**8. Đáp án đúng là C.**

**Giải thích:**

- **A)** Đáp án này sai. Backup hàng ngày lúc 2 giờ sáng cho RPO lên tới 24 giờ — vượt xa yêu cầu 1 phút. Restore bằng tay cũng gần như không thể đạt RTO 15 phút.

- **B)** Đáp án này sai. Single AZ nghĩa là mất AZ sẽ downtime toàn phần; snapshot mỗi giờ cho RPO 1 giờ (chưa đạt yêu cầu 1 phút) và restore thủ công không đạt RTO 15 phút. Đây là mức DR của hệ thống nội bộ, không phải của payment.

- **C)** Đáp án này đúng. Multi-AZ với replication đồng bộ hoặc bán đồng bộ đảm bảo dữ liệu đã ghi được nhân bản trước khi xác nhận (RPO dưới 1 phút), backup liên tục/PITR bổ sung lớp bảo vệ, và failover tự động giữa các AZ giúp RTO trong khoảng 15 phút. Đây là cấu hình cân bằng giữa yêu cầu và chi phí.

- **D)** Đáp án này sai. Multi-region active-active vượt yêu cầu (RTO gần 0), chi phí cao hơn nhiều và kéo theo bài toán consistency/routing. Chỉ nên cân nhắc khi downtime ảnh hưởng doanh thu nghiêm trọng hoặc có yêu cầu pháp lý, không phải "lựa chọn duy nhất".




**9. Đáp án đúng là B.**

**Giải thích:**

- **A)** Đáp án này sai. Thiếu `minReplicas` thì HPA dùng giá trị mặc định 1, vẫn tính toán bình thường — không gây ra trạng thái `unknown`.

- **B)** Đáp án này đúng. HPA tính utilization bằng công thức `usage / requests`; nếu pod không khai báo `requests.cpu`, phép chia không có mẫu số và HPA trả về `unknown`. Việc `kubectl top pods` hoạt động chứng tỏ metrics-server đã cài và metric thô vẫn có — vấn đề nằm ở `requests`, không phải metrics-server.

- **C)** Đáp án này sai. HPA gắn vào workload qua `scaleTargetRef` (Deployment/StatefulSet/ReplicaSet), không liên quan tới Service. Thiếu Service gây lỗi truy cập, không gây `unknown` ở metric.

- **D)** Đáp án này sai. Target CPU được hỗ trợ ở `autoscaling/v2` (và cả `autoscaling/v1`), nên hạ API version không giải quyết vấn đề. Hơn nữa `autoscaling/v2` mới hỗ trợ đầy đủ memory, custom metric và `behavior`.




**10. Đáp án đúng là A, C và D.**

**Giải thích:**

- **A)** Đáp án này đúng. Rightsizing bằng VPA recommendation, xóa workload idle và tắt/giảm non-prod ngoài giờ là những tối ưu giảm chi phí mà **không đụng tới kiến trúc reliability của production** — nhóm việc nên làm trước tiên.

- **B)** Đáp án này sai. Bỏ multi-AZ cho toàn bộ production khiến mất một AZ trở thành downtime toàn phần — đánh đổi reliability lấy chi phí một cách thô bạo. Nếu cần tiết kiệm, hãy đánh giá từng dịch vụ, giữ multi-AZ cho hệ thống trọng yếu và để business quyết định các trường hợp còn lại.

- **C)** Đáp án này đúng. Spot có thể giảm 60–90% chi phí compute cho workload chịu được gián đoạn, nhưng phải đi kèm PDB và xử lý interruption (graceful shutdown, đa dạng instance type). Dọn tài nguyên treo (LoadBalancer, PV/PVC mồ côi, elastic IP, snapshot cũ) là phần "tiền miễn phí" của FinOps.

- **D)** Đáp án này đúng. Giữ headroom N+1 cho dịch vụ trọng yếu bảo vệ khả năng chịu lỗi node/AZ, còn các thay đổi RTO/RPO phải được đưa lên business quyết định kèm con số chi phí - rủi ro thay vì đội kỹ thuật tự hạ mức dự phòng. Đây là cách cắt chi phí có kỷ luật và minh bạch.
