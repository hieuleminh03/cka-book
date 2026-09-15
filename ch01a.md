---
layout: answer

title: "Chương 1"
subtitle: "Linux & System Hardening nền tảng"
exam_objectives:
  - "Quản lý process, job và service trên Linux bằng systemd, systemctl và journalctl."
  - "Làm việc với user, group, permission, sudo và chính sách least privilege ở mức hệ điều hành."
  - "Phân tích hiệu năng hệ thống (CPU, memory, disk, I/O, network) và xử lý sự cố bằng các công cụ có sẵn."
  - "Áp dụng hardening host: giảm attack surface, AppArmor và seccomp (chuẩn bị cho CKS System Hardening)."
---

## Đáp án {#answers}

**1. Đáp án đúng là B.**

**Giải thích:**

- **A)** systemd đã vô hiệu hóa vĩnh viễn unit; chỉ có thể tạo unit file mới để service chạy tiếp.
  - Đáp án này sai. Unit không bị vô hiệu hóa vĩnh viễn; systemd chỉ chuyển unit sang trạng thái `failed` sau khi vượt giới hạn khởi động lại. Unit file vẫn còn nguyên và bạn có thể đưa service trở lại bằng `systemctl reset-failed` rồi `systemctl start`.

- **B)** Đây là cơ chế rate limiting khởi động (StartLimitIntervalSec/StartLimitBurst). Cần sửa nguyên nhân crash; để chạy lại ngay có thể dùng `systemctl reset-failed` rồi `systemctl start`, hoặc điều chỉnh giới hạn có chủ đích.
  - Đáp án này đúng. Khi `Restart=on-failure` khiến service crash và được restart liên tục, systemd áp dụng giới hạn mặc định (5 lần khởi động trong 10 giây) để tránh "crash loop" đốt CPU. Khi vượt giới hạn, systemd ghi log `start request repeated too quickly` và đưa unit vào trạng thái `failed`. Việc cần làm trước tiên là sửa lỗi khiến service crash, sau đó `systemctl reset-failed <unit>` để xóa bộ đếm rồi start lại. Chỉ nên tăng `StartLimitBurst`/`StartLimitIntervalSec` khi bạn hiểu rõ vì sao cần.

- **C)** Đây là rate limit của journald, cần tăng `RateLimitBurst` trong `/etc/systemd/journald.conf` rồi restart journald.
  - Đáp án này sai. Thông báo `start request repeated too quickly` do **systemd manager** phát ra, không phải journald. journald cũng có rate limiting nhưng nó giới hạn tốc độ **ghi log** (tránh log flood), và việc chỉnh nó không liên quan đến vòng lặp restart của service.

- **D)** Đây là hành vi đúng của `Restart=on-failure` khi process thoát với mã 0; cần đổi thành `Restart=always` để khắc phục.
  - Đáp án này sai. `on-failure` chỉ restart khi process thoát với mã lỗi hoặc bị signal — nó **không** restart khi exit code là 0 (muốn vậy phải dùng `Restart=always`). Hơn nữa, hiện tượng trong đề bài là rate limit do crash liên tục; đổi sang `always` chỉ khiến service crash nhanh hơn và không giải quyết nguyên nhân gốc.

**2. Đáp án đúng là A.**

**Giải thích:**

- **A)** `drwxr-x---` và `-rw-r-----`.
  - Đáp án này đúng. Process tạo thư mục với mode nền `777` và file với mode nền `666`, sau đó kernel trừ đi umask `027`: thư mục `777 AND NOT 027 = 750` (`drwxr-x---`) và file `666 AND NOT 027 = 640` (`-rw-r-----`). Lưu ý file mới không bao giờ có bit thực thi nên mode nền là `666` chứ không phải `777`.

- **B)** `drwxrwx---` và `-rw-rw----`.
  - Đáp án này sai. Đây là kết quả của umask `007` (777 − 007 = 770; 666 − 007 = 660), không phải umask `027`. Với umask 007, bit `others` bị chặn còn group vẫn có toàn quyền.

- **C)** `drwxr-xr-x` và `-rw-r--r--`.
  - Đáp án này sai. Đây là kết quả của umask `022` (mặc định phổ biến trên nhiều distro), khi đó group và others chỉ bị gỡ quyền write, còn lại `755` cho thư mục và `644` cho file.

- **D)** `drwxr-xr-x` và `-rw-r-----`.
  - Đáp án này sai vì trộn hai umask khác nhau: thư mục `755` tương ứng umask `022`, còn file `640` tương ứng umask `027`. Cùng một umask tác động nhất quán lên cả hai loại mode nền, nên kết quả này không thể xảy ra.

**3. Đáp án đúng là B.**

**Giải thích:**

- **A)** `chmod u+s /usr/local/bin/agent` để binary chạy với quyền root hiệu dụng.
  - Đáp án này sai vì SUID khiến toàn bộ chương trình chạy với quyền root đầy đủ (UID 0), trong khi nhu cầu chỉ là bind port thấp. Đây là cấp thừa quyền nghiêm trọng: mọi lỗ hổng hoặc shell escape trong binary sẽ cho kẻ tấn công quyền root — vi phạm least privilege.

- **B)** `setcap 'cap_net_bind_service=+ep' /usr/local/bin/agent`.
  - Đáp án này đúng. `CAP_NET_BIND_SERVICE` cho phép bind các port dưới 1024 mà không cần là root; file capability này gắn với binary, còn process vẫn chạy dưới user thường. Đây chính là ví dụ mẫu của least privilege ở mức hệ điều hành: cấp đúng đặc quyền nhỏ nhất cần thiết.

- **C)** Thêm user chạy agent vào group `root` để có đủ quyền.
  - Đáp án này sai và phản least privilege. Thành viên group `root` không tự động có quyền bind port thấp hay các đặc quyền kernel khác — quyền kernel được kiểm tra theo UID/capability, không theo group `root`. Thêm user vào group root còn tạo rủi ro truy cập file/directory của root trên nhiều hệ thống.

- **D)** `chown root:root /usr/local/bin/agent && chmod 4755 /usr/local/bin/agent`.
  - Đáp án này sai vì `chmod 4755` chính là bật SUID root — giống đáp án A, cấp toàn bộ quyền root thay vì chỉ một capability hẹp. Đây là cách làm cũ và nguy hiểm hơn hẳn `setcap`.

**4. Đáp án đúng là B.**

**Giải thích:**

- **A)** Chạy `fsck` ngay trên filesystem đang mount để "thu hồi" dung lượng bị thất lạc.
  - Đáp án này sai. `fsck` trên filesystem đang mount ở chế độ ghi có thể gây hỏng dữ liệu, và tình huống mô tả (df đầy nhưng du thấp) không phải lỗi cấu trúc filesystem. Trước khi nghĩ đến fsck, phải loại trừ deleted-but-open file.

- **B)** Dùng `lsof +L1` để tìm file đã bị xóa nhưng còn process giữ file descriptor, sau đó restart/reload process để giải phóng dung lượng.
  - Đáp án này đúng. Đây là nguyên nhân kinh điển của "df đầy nhưng du không thấy": file đã bị unlink khỏi thư mục nhưng process vẫn giữ file descriptor, nên kernel chưa giải phóng block. `lsof +L1` liệt kê các file có link count nhỏ hơn 1 (đã bị xóa); sau khi restart/reload process giữ file (hoặc truncate qua `/proc/<pid>/fd/<fd>` nếu hiểu rõ rủi ro), dung lượng được trả lại.

- **C)** Dùng `df -i` để kiểm tra inode và tăng số inode của filesystem bằng `tune2fs`.
  - Đáp án này sai trong ngữ cảnh này. Hết inode (`%IUse` 100%) là nguyên nhân khác, cũng gây "No space left on device", nhưng đề bài cho thấy dung lượng `df -h` đã 100% trong khi `du` chỉ 12G — đây là hiện tượng dung lượng block bị giữ, không phải inode. Ngoài ra, số inode chỉ được quyết định khi tạo filesystem, không tăng đơn giản bằng `tune2fs`.

- **D)** Reboot máy chủ vì đây là lỗi kernel và không thể xử lý online.
  - Đáp án này sai. Đây không phải lỗi kernel mà là hành vi bình thường của Unix khi file bị xóa but vẫn còn open. Reboot sẽ giải phóng dung lượng nhưng là biện pháp thô, gây downtime không cần thiết — có thể xử lý online bằng cách restart/reload đúng process.

**5. Đáp án đúng là B.**

**Giải thích:**

- **A)** `resize2fs /dev/vg_data/lv_app 10G` rồi `lvextend -L +10G /dev/vg_data/lv_app`.
  - Đáp án này sai vì ngược thứ tự: filesystem ext4 chỉ có thể giãn tới kích thước của LV, nên phải tăng LV trước rồi mới giãn filesystem. Ngoài ra, `resize2fs` với một tham số kích thước phải nhỏ hơn hoặc bằng dung lượng thiết bị — ở đây thiết bị chưa lớn hơn nên lệnh không thể thành công như mong đợi.

- **B)** `lvextend -L +10G /dev/vg_data/lv_app` rồi `resize2fs /dev/vg_data/lv_app`.
  - Đáp án này đúng. VG còn dư 20G nên `lvextend` tăng LV thêm 10G không cần thêm PV; sau đó `resize2fs` giãn filesystem ext4 để dùng hết không gian mới — cả hai đều online, không cần unmount. Thực tế có thể dùng một lệnh `lvextend -r -L +10G /dev/vg_data/lv_app` để tự động gọi công cụ resize filesystem.

- **C)** `vgextend vg_data /dev/sdb1` rồi `mkfs.ext4 /dev/vg_data/lv_app`.
  - Đáp án này sai ở hai điểm chí mạng. VG đã còn 20G nên không cần thêm PV (`vgextend`) — chỉ dùng khi VG hết dung lượng free. Quan trọng hơn, `mkfs.ext4` sẽ **định dạng lại và xóa sạch dữ liệu** trên LV, biến việc "mở rộng" thành thảm họa mất dữ liệu.

- **D)** `lvreduce -L -10G /dev/vg_data/lv_app` rồi `pvresize /dev/sdb`.
  - Đáp án này sai vì `lvreduce` là thu nhỏ LV — ngược hoàn toàn với yêu cầu tăng thêm 10G, và thu nhỏ LV không đúng cách có thể mất dữ liệu. `pvresize` dùng khi thay đổi kích thước của chính PV vật lý, không liên quan ở đây.

**6. Đáp án đúng là B.**

**Giải thích:**

- **A)** CPU đang quá tải nghiêm trọng; cần tăng số vCPU ngay lập tức.
  - Đáp án này sai. Cột `r` chỉ 1–2 (không có hàng đợi process chờ CPU), `id` còn khoảng 25% nghĩa là CPU còn rảnh, nên CPU không phải nút cổ chai. Tăng vCPU không giải quyết được vấn đề và làm tăng chi phí không cần thiết.

- **B)** Load average cao chủ yếu do I/O wait và các process ở trạng thái D; nút cổ chai nằm ở disk/storage chứ không phải CPU.
  - Đáp án này đúng. Load average trên Linux tính cả process runnable (R) lẫn process ở trạng thái uninterruptible sleep (D) — thường là đang chờ I/O. Ở đây `wa` 40–70% và cột `b` lớn xác nhận các process đang bị chặn bởi I/O; `r` thấp cho thấy không thiếu CPU. Hướng xử lý đúng là điều tra storage (`iostat` để xem `await`, `%util`, `aqu-sz`, tìm process gây I/O).

- **C)** Load average 9.5 nghĩa là trung bình 9.5% CPU đang được sử dụng.
  - Đáp án này sai về định nghĩa. Load average là **số process** trung bình ở trạng thái R hoặc D, không phải phần trăm CPU. Muốn biết % CPU hãy đọc `us`/`sy`/`id` trong `vmstat` hoặc `top`.

- **D)** Cần tăng `vm.swappiness` để giảm load average.
  - Đáp án này sai. `vm.swappiness` điều chỉnh mức độ kernel ưu tiên swap; tăng nó có thể khiến hệ thống swap nhiều hơn (làm tăng I/O — đúng nguyên nhân đang gây tải), không liên quan gì đến việc giảm load average do I/O wait. Đây là "chữa sai bệnh".

**7. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Có thể tìm bằng chứng trong kernel log: `dmesg -T | grep -i oom` hoặc `journalctl -k -g oom`.
  - Đáp án này đúng. OOM killer là hành vi của kernel nên bằng chứng nằm trong kernel ring buffer: các dòng kiểu `Out of memory: Killed process 1234 (java) ...`. `dmesg` đọc buffer trực tiếp, còn `journalctl -k` đọc cùng nội dung qua journald (và tiện lọc theo từ khóa hoặc thời gian).

- **B)** `/proc/<pid>/oom_score_adj` cho phép điều chỉnh ưu tiên bị chọn của process; giá trị -1000 gần như bảo vệ process khỏi bị kill.
  - Đáp án này đúng. `oom_score_adj` (giá trị -1000 đến 1000) điều chỉnh điểm mà OOM killer dùng để chọn "nạn nhân": giá trị âm làm process ít có khả năng bị chọn, -1000 gần như miễn nhiễm (trừ trường hợp cạn kiệt cực đoan). Đây là cách bảo vệ process quan trọng (sshd, database) trên host.

- **C)** OOM killer luôn chọn process có RSS lớn nhất để kết thúc.
  - Đáp án này sai. Kernel không chỉ so RSS: `oom_score` là điểm tổng hợp từ nhiều yếu tố (lượng memory ảo và RSS, swap đang dùng, tuổi process, các process con, `oom_score_adj`, quyền root...). Trong thực tế process ngốn ít RSS hơn nhưng có `oom_score_adj` cao hoặc giữ nhiều mapping vẫn có thể bị chọn. Vì vậy câu "luôn chọn RSS lớn nhất" là tuyệt đối hóa sai.

- **D)** Với cgroup v2, bộ đếm `oom_kill` trong `memory.events` là bằng chứng của việc process trong cgroup bị kill do vượt `memory.max`.
  - Đáp án này đúng. Trên cgroup v2, mỗi cgroup có `memory.max` (limit) và file `memory.events` chứa các bộ đếm `oom`, `oom_kill`, `max`... Bộ đếm `oom_kill` tăng nghĩa là đã có process trong cgroup đó (ví dụ một container) bị kernel kill vì cạn memory trong phạm vi giới hạn — đây chính là bằng chứng của container/Kubernetes bị `OOMKilled` (exit code 137).

**8. Đáp án đúng là A.**

**Giải thích:**

- **A)** Thư mục `/var/log/journal` không tồn tại nên journald chạy ở chế độ volatile; tạo thư mục này, đặt `Storage=persistent` trong `/etc/systemd/journald.conf` rồi restart `systemd-journald`.
  - Đáp án này đúng. Mặc định `Storage=auto`, journald chỉ ghi log vào `/var/log/journal` (persistent) khi thư mục này tồn tại; nếu không, log nằm ở `/run/log/journal` (trong RAM) và mất sạch khi reboot — vì vậy `journalctl -b -1` không có dữ liệu. Tạo thư mục và đặt `Storage=persistent` (hoặc `auto` sau khi đã tạo thư mục) rồi restart `systemd-journald` sẽ giữ log qua các lần boot.

- **B)** journald không hỗ trợ lưu log giữa các lần reboot; cần cài rsyslog để thay thế.
  - Đáp án này sai. journald **có** hỗ trợ persistent journal (đó là mục đích của `/var/log/journal`); chỉ là cấu hình mặc định của một số bản phân phối chạy volatile. rsyslog là một giải pháp logging khác, có thể chạy song song và đẩy log đi xa, nhưng không phải điều kiện để journal lưu qua reboot.

- **C)** Cần chạy `systemctl enable systemd-journald` vì journald mặc định không được enable.
  - Đáp án này sai. `systemd-journald` luôn hoạt động như một phần của systemd (được khởi động sớm qua socket/unit), không phải dịch vụ cần `enable` thủ công. Vấn đề trong đề bài là chế độ lưu trữ (volatile vs persistent), không phải enable/disable.

- **D)** Log cũ bị `logrotate` xóa trong vòng một giờ; cần tăng tham số `rotate` trong `/etc/logrotate.conf`.
  - Đáp án này sai. Journal nhị phân của journald **không** do `logrotate` quản lý; journald tự xoay vòng/xóa theo `SystemMaxUse`, `MaxRetentionSec` hoặc khi bạn chạy `journalctl --vacuum-*`. Thêm nữa, không có cơ chế mặc định nào xóa sạch log trong một giờ.

**9. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** seccomp hoạt động ở mức system call (lọc syscall), còn AppArmor là MAC lọc theo đường dẫn file, quyền network và một số hành vi của process.
  - Đáp án này đúng. seccomp cài filter vào chính process, cho phép/chặn từng syscall (ví dụ chặn `mkdir`, `ptrace`, `mount`) với các action như `SCMP_ACT_ALLOW`, `SCMP_ACT_ERRNO`. AppArmor là LSM hoạt động theo profile gắn với chương trình, giới hạn truy cập theo đường dẫn file, quyền network và một số hành vi — hai cơ chế khác lớp nhau hoàn toàn.

- **B)** Process chạy dưới AppArmor profile ở chế độ enforce vẫn bị giới hạn kể cả khi có UID 0 (root); vi phạm profile bị chặn bất kể quyền DAC.
  - Đáp án này đúng. Đây là bản chất của **Mandatory** Access Control: policy do quản trị viên định nghĩa và được kernel thực thi độc lập với danh tính process. Root vượt qua được kiểm tra quyền DAC (rwx/owner) nhưng vẫn bị chặn nếu profile không cho phép — ví dụ profile `deny /** w` khiến `touch` thất bại dù chạy bằng `sudo`.

- **C)** Chế độ `complain` của AppArmor chặn mọi hành vi vi phạm profile và chỉ ghi log khi đã chặn xong.
  - Đáp án này sai, mô tả ngược chức năng. `complain` chỉ **ghi log** các hành vi lệch profile mà **không chặn** — dùng để xây dựng/debug profile trước khi chuyển sang `enforce`. Chế độ chặn vi phạm là `enforce` (và cũng ghi log song song).

- **D)** seccomp chỉ áp dụng được cho process chạy bằng quyền root; process user thường không thể bị seccomp filter.
  - Đáp án này sai. seccomp filter áp dụng cho bất kỳ process nào — thực tế container runtime áp seccomp mặc định cho mọi container, dù process bên trong chạy bằng user thường; ngược lại, filter do process tự cài thì process cần quyền phù hợp, nhưng filter của runtime kế thừa xuống mọi process con bất kể UID.

**10. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** `capabilities.drop: ["ALL"]` kết hợp `capabilities.add: ["NET_BIND_SERVICE"]`.
  - Đáp án này đúng. Để bind port 80 (port < 1024), process cần capability `CAP_NET_BIND_SERVICE`. Nguyên tắc least privilege là drop toàn bộ capability của container trước, rồi add đúng một capability cần thiết — vừa đáp ứng yêu cầu, vừa thu nhỏ bề mặt tấn công khi container bị chiếm quyền.

- **B)** `runAsNonRoot: true` kết hợp `allowPrivilegeEscalation: false`.
  - Đáp án này đúng và bổ trợ cho A: `runAsNonRoot` buộc process chạy bằng user không phải root, còn `allowPrivilegeEscalation: false` chặn mọi con đường leo thang đặc quyền (SUID, file capabilities) trong container. Đây cũng là các ràng buộc của Pod Security Standards mức `restricted` — cấu hình chuẩn cho workload production.

- **C)** `privileged: true` để container có thể bind mọi port mà không gặp giới hạn.
  - Đáp án này sai và rất nguy hiểm. `privileged: true` cấp cho container gần như toàn bộ đặc quyền của host: mọi capability, quyền truy cập thiết bị, thao tác kernel — phá vỡ cách ly với host và bị PSS `baseline`/`restricted` từ chối. Bind port 80 không cần đến mức đó; `NET_BIND_SERVICE` là đủ.

- **D)** `allowPrivilegeEscalation: true` kết hợp `readOnlyRootFilesystem: false` để dễ gỡ lỗi trên production.
  - Đáp án này sai. Cả hai lựa chọn đều đi ngược nguyên tắc least privilege: cho phép leo thang đặc quyền và cho phép ghi root filesystem làm tăng bề mặt tấn công (kẻ tấn công có thể ghi binary/script vào container). "Dễ gỡ lỗi" không phải lý do chính đáng để nới lỏng các thiết lập này trên production; nếu cần debug, hãy dùng ephemeral container hoặc môi trường staging.

