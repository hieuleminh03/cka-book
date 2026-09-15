---
layout: answer

title: "Chương 13"
subtitle: "Cloud & IaC"
exam_objectives:
  - "Nắm core services của AWS: VPC, EC2, IAM, S3, RDS, ELB và Route 53."
  - "Quản lý hạ tầng bằng Terraform: state, module và workspace."
  - "Tự động hóa cấu hình với Ansible."
  - "Áp dụng IAM least privilege và cost optimization."
---

## Đáp án {#answers}

**1. Đáp án đúng là B.**

**Giải thích:**

- **A)** Đáp án này sai. Internet Gateway được gắn vào **VPC**, không gắn vào subnet, và bản thân nó cũng không quyết định subnet nào ra được Internet. Subnet được gọi là public khi **route table** của nó có default route trỏ tới IGW; gắn IGW vào private subnet cũng sẽ khiến instance lộ trực tiếp ra Internet — ngược với yêu cầu đề bài.

- **B)** Đáp án này đúng. NAT Gateway nằm trong public subnet (nơi có route tới IGW) và dịch địa chỉ cho traffic đi ra từ private subnet. Private subnet cần một route table có default route `0.0.0.0/0` trỏ tới NAT Gateway đó. Instance nhận response cho kết nối nó khởi tạo, nhưng không ai từ Internet kết nối vào được — đúng mô hình outbound-only.

- **C)** Đáp án này sai. Elastic IP gắn vào ENI không giải quyết bài toán định tuyến: subnet private không có route tới IGW nên traffic vẫn không ra được. Nếu có route tới IGW thì gắn EIP lại khiến instance trở thành public — trái yêu cầu "không nhận kết nối từ Internet".

- **D)** Đáp án này sai. VPC peering nối hai VPC với nhau, không cung cấp đường ra Internet. Nếu VPC đích có IGW và cấu hình route đúng thì đây là mô hình hub-and-spoke phức tạp, không phải "thành phần bắt buộc" cho tình huống này.

**2. Đáp án đúng là B.**

**Giải thích:**

- **A)** Đáp án này sai. Security Group là **stateful**: khi kết nối được phép vào, traffic trả về (response) tự động được cho phép, không cần rule outbound riêng cho ephemeral ports. Thêm nữa, security group mặc định vốn cho phép toàn bộ outbound.

- **B)** Đáp án này đúng. Network ACL là **stateless**: chiều vào và chiều ra được đánh giá độc lập theo số thứ tự rule. SYN từ client (destination port 443) khớp rule inbound 443, nhưng SYN-ACK từ server có **destination port là ephemeral port** của client (1024–65535), không khớp rule outbound 443 nên bị drop — client retry SYN cho tới khi timeout, handshake không bao giờ hoàn tất. Cách sửa đúng là thêm outbound rule cho dải ephemeral ports 1024–65535 (và giữ inbound 443).

- **C)** Đáp án này sai. Route table đã có default route tới Internet Gateway; routing không cần route riêng cho từng client, và nếu thiếu routing thì triệu chứng sẽ giống nhau nhưng nguyên nhân nằm ở bảng route chứ không phải NACL. NACL lọc theo port/protocol, không phải theo route.

- **D)** Đáp án này sai. NAT Gateway dành cho traffic **outbound** khởi tạo từ private subnet; nó không liên quan tới kết nối inbound từ Internet vào server public. Ở đây vấn đề nằm ở rule lọc chiều về của NACL.

**3. Đáp án đúng là B.**

**Giải thích:**

- **A)** Đáp án này sai và vi phạm least privilege nặng nhất. `s3:*` trên mọi resource (`"Resource": "*"`) cho phép tạo bucket mới, sửa policy, xóa cả bucket trên toàn bộ account S3 — trong khi Lambda chỉ cần đọc một bucket. Đây là kiểu policy xuất hiện nhiều nhất trong các đợt audit bảo mật.

- **B)** Đáp án này đúng. `s3:ListBucket` là action cấp bucket nên resource phải là ARN của bucket (`arn:aws:s3:::app-assets`); `s3:GetObject` là action cấp object nên resource phải là ARN của object (`arn:aws:s3:::app-assets/*`). Tách thành hai statement theo đúng cách S3 phân quyền, và không cấp quyền nào dư thừa.

- **C)** Đáp án này sai vì gộp hai action vào bucket ARN: `ListBucket` chạy được nhưng `GetObject` trên bucket ARN **không** hợp lệ — S3 đánh giá `GetObject` theo ARN object (`.../*`), nên Lambda sẽ bị `AccessDenied` khi tải object. Đây là lỗi cấu hình rất hay gặp trên thực tế.

- **D)** Đáp án này sai. Khi có cả Deny và Allow cho cùng một action trên cùng resource, **explicit Deny luôn thắng** — kết quả là `GetObject` bị chặn hoàn toàn dù có statement Allow. Không chỉ không "an toàn hơn" mà policy vô dụng đúng cho hành vi cần cấp.

**4. Đáp án đúng là B, C và D.**

**Giải thích:**

- **A)** Đáp án này sai. Gắn role vào node group cho mọi pod trên node kế thừa quyền: bất kỳ pod nào (kể cả pod của team khác hoặc mã độc chạy trong container) cũng có thể lấy credential của node qua IMDS và gọi DynamoDB. Đây là ví dụ chuẩn của việc vi phạm least privilege, không phải cách được khuyến nghị.

- **B)** Đáp án này đúng. Đây chính là IRSA (IAM Roles for Service Accounts): EKS sinh OIDC provider cho cluster, IAM role có trust policy `sts:AssumeRoleWithWebIdentity` với condition giới hạn `sub` = `system:serviceaccount:<namespace>:<name>` và `aud` = `sts.amazonaws.com`; ServiceAccount được annotate `eks.amazonaws.com/role-arn`. Quyền gắn theo service account, không theo node.

- **C)** Đáp án này đúng. Đây là cơ chế hoạt động bên dưới: kubelet chiếu projected service account token (web identity) vào pod; AWS SDK đổi token này lấy credential tạm thời từ STS. Không có access key tĩnh để rò rỉ, credential tự hết hạn, mọi lần assume đều được CloudTrail ghi lại.

- **D)** Đáp án này đúng. EKS Pod Identity là cơ chế mới hơn, gắn role với service account qua add-on của EKS mà không cần tự cấu hình OIDC provider và trust policy phức tạp. Về mô hình bảo mật tương đương IRSA: quyền theo từng service account, credential tạm thời.

**5. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. State map địa chỉ resource trong config với tài nguyên thật, nhờ đó Terraform biết cần tạo/sửa/xóa gì. State cũng lưu thuộc tính của resource, trong đó có những giá trị nhạy cảm ở dạng plaintext (ví dụ password RDS do provider trả về), nên backend phải được mã hóa, giới hạn truy cập và không để lộ ra ngoài.

- **B)** Đáp án này đúng. S3 là remote backend phổ biến nhất (kèm versioning để khôi phục state cũ và mã hóa). Terraform 1.10+ hỗ trợ **native locking** trên S3 bằng `use_lockfile = true`, dần thay thế bảng DynamoDB (khóa `LockID`) vốn là cách làm truyền thống và vẫn tương thích.

- **C)** Đáp án này sai. Terraform không theo dõi hạ tầng theo thời gian thực; nó chỉ đọc lại trạng thái thật khi `plan`/`apply` refresh (hoặc `-refresh-only`). Thay đổi ngoài Terraform (console, CLI, người khác) tạo ra **drift** và chỉ được phát hiện ở lần refresh kế tiếp — đó là lý do nhiều đội chạy `plan` định kỳ để phát hiện sớm.

- **D)** Đáp án này đúng. `terraform state rm` chỉ bỏ resource khỏi state để Terraform "quên" nó (tài nguyên thật vẫn còn nguyên); ngược lại `terraform import` đưa một tài nguyên đang tồn tại vào state để Terraform bắt đầu quản lý, nhưng không tự sinh code — bạn phải viết config khớp thì `plan` mới sạch.

**6. Đáp án đúng là B.**

**Giải thích:**

- **A)** Đáp án này sai và còn tệ hơn. `-replace` chủ động yêu cầu Terraform hủy rồi tạo lại tài nguyên — đúng thứ bạn đang muốn tránh, chỉ khác là thực hiện có chủ đích thay vì do đổi địa chỉ.

- **B)** Đáp án này đúng. Đổi tên resource khiến Terraform thấy địa chỉ cũ "biến mất" và địa chỉ mới "xuất hiện", nên mặc định nó lập kế hoạch destroy + create. Block `moved { from = aws_instance.web, to = aws_instance.app }` (Terraform 1.1+) hoặc lệnh `terraform state mv aws_instance.web aws_instance.app` báo cho Terraform biết đây **vẫn là một tài nguyên**, chỉ đổi địa chỉ trong state — không có thao tác nào lên hạ tầng thật, không downtime.

- **C)** Đáp án này sai. `.terraform` chỉ chứa provider plugin và metadata; xóa nó rồi `init` lại chỉ tải lại provider. State vẫn giữ địa chỉ cũ còn config dùng địa chỉ mới, nên `plan` vẫn báo destroy/create như trước.

- **D)** Đáp án này sai. `prevent_destroy` chặn thao tác phá hủy (apply sẽ báo lỗi), nhưng không giải thích cho Terraform rằng đây là cùng một tài nguyên; `plan` vẫn hiển thị kế hoạch replace và công việc của bạn bị chặn, không đi tới đâu. Nó phù hợp làm lưới an toàn cho tài nguyên quý (database), không phải cách rename.

**7. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. Workspace tách state nhưng dùng chung code và biến; mọi thay đổi ảnh hưởng đồng thời đến tất cả môi trường, và chỉ cần quên `workspace select` đúng là bạn có thể `apply` nhầm dev/prod. Vì vậy workspace phù hợp cho non-prod, không dùng làm ranh giới an toàn cho prod.

- **B)** Đáp án này đúng. Tách thư mục (hoặc repo) cho prod tạo cô lập thật: state riêng, pipeline riêng, quyền riêng, có thể yêu cầu approval riêng và giới hạn blast radius khi thao tác sai. Đây là khuyến nghị phổ biến khi có yêu cầu vận hành nghiêm ngặt cho production.

- **C)** Đáp án này sai. Best practice là **không** khai báo `provider` bên trong child module. Provider được cấu hình ở root module và truyền xuống (kể cả alias), nhờ vậy module tái sử dụng được cho nhiều region/account khác nhau, không phát sinh xung đột version hay cấu hình trùng lặp. Khai báo provider trong module chỉ gây rối và khó kiểm soát.

- **D)** Đáp án này đúng. `version = "~> 5.0"` cho phép nhận bản 5.x nhưng chặn bản 6.x, để việc nâng cấp chỉ xảy ra khi bạn chủ động và có cơ hội review thay đổi — module registry có thể ra breaking change giữa các major version.

**8. Đáp án đúng là A, C và D.**

**Giải thích:**

- **A)** Đáp án này đúng. Handler chỉ chạy khi có task gọi `notify`; Ansible gom các notify và chạy handler **một lần ở cuối play** dù nó được kích hoạt nhiều lần — đúng ngữ nghĩa "restart service một lần sau khi tất cả config đã thay đổi". Muốn chạy ngay giữa play thì dùng `meta: flush_handlers`.

- **B)** Đáp án này sai. `command`/`shell`/`raw` luôn chạy và luôn báo `changed` vì Ansible không biết trạng thái mong muốn của một lệnh tùy ý. Muốn idempotent phải thêm `creates`/`removes`, dùng `changed_when`, hoặc thay bằng module chuyên dụng (`copy`, `template`, `apt`, `service`).

- **C)** Đáp án này đúng. Ansible Vault mã hóa file hoặc giá trị đơn lẻ bằng AES256; khi chạy có thể nhập mật khẩu tương tác (`--ask-vault-pass`) hoặc trỏ file (`--vault-password-file`) — cách sau phù hợp CI, với vault password lấy từ secret store của CI và không bao giờ commit.

- **D)** Đáp án này đúng. Module `template` render file Jinja2 rồi so sánh với nội dung đích; chỉ khi khác nhau (hoặc thuộc tính file đổi) mới ghi và báo `changed`, nên chạy lại nhiều lần không làm thay đổi gì — đúng tinh thần idempotent.

**9. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** Đáp án này đúng. Kiến trúc multi-region cần mỗi region một bộ ALB + target group với health check riêng (app của region dự phòng phải chạy sẵn, thường ở chế độ standby hoặc active-active). Route 53 failover routing dùng record PRIMARY cho region chính và SECONDARY cho region dự phòng; khi health check của PRIMARY fail, DNS bắt đầu trả về SECONDARY — đây là cơ chế failover cấp DNS.

- **B)** Đáp án này đúng. ALIAS record trỏ thẳng tới ALB mà không cần biết IP (ALB có IP thay đổi), dùng được ở apex domain và cập nhật tự động khi ALB thay đổi. Health check là thành phần không thể thiếu: không có nó, Route 53 không biết region chính "chết" để chuyển traffic.

- **C)** Đáp án này sai. Multi-AZ chỉ nhân bản standby **trong cùng một region**; khi cả region gặp sự cố, standby cũng đi theo. DR cross-region cần cơ chế khác: read replica cross-region rồi promote (thời gian promote và mất mát dữ liệu cần được tính vào RPO), hoặc Aurora Global Database với replica ở region khác.

- **D)** Đáp án này sai. ALB là tài nguyên **theo region**, không thể phục vụ xuyên region; một ALB duy nhất không giúp gì khi region đó sự cố. Để tiết kiệm, có thể dùng mô hình warm standby (hạ tầng dự phòng nhỏ hơn) nhưng vẫn phải có thành phần phân phối traffic toàn cầu như Route 53 hoặc Global Accelerator.

**10. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. Phần baseline ổn định của web tier (chạy 24/7, biết trước nhu cầu) là ứng viên lý tưởng cho Savings Plans hoặc Reserved Instances — giảm giá đáng kể đổi lấy cam kết 1–3 năm. Batch job chạy đêm và chịu được gián đoạn là ứng viên lý tưởng cho Spot (giảm sâu, chỉ cần thiết kế retry vì có thể bị thu hồi với cảnh báo 2 phút).

- **B)** Đáp án này đúng. Right-sizing bằng Compute Optimizer xử lý phần lãng phí lớn nhất thường gặp: instance over-provisioned. Chuyển sang Graviton (ARM) cho hiệu năng/giá tốt hơn khoảng 20% nếu stack tương thích — hai việc này không đánh đổi độ tin cậy.

- **C)** Đáp án này sai. Bật Multi-AZ và NAT Gateway "ở mọi subnet" làm tăng chi phí cố định đáng kể (NAT tính theo giờ và theo GB, mỗi AZ một NAT là nhân đôi/ba chi phí) mà không phải lúc nào cũng cần — quyết định HA phải dựa trên yêu cầu RTO của workload. Đây đúng là kiểu tư duy "độ tin cậy trước, chi phí tính sau" làm hóa đơn phình to.

- **D)** Đáp án này đúng. S3 lifecycle chuyển object cũ sang Standard-IA/Glacier và xóa dữ liệu hết hạn là cách giảm chi phí lưu trữ an toàn. Bên cạnh đó, EBS volume và Elastic IP không gắn máy vẫn bị tính phí (EIP unattached bị tính theo giờ từ 2024) — rà soát và xóa tài nguyên mồ côi là khoản tiết kiệm nhanh và rõ ràng.
