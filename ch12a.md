---
layout: answer

title: "Chương 12"
subtitle: "CI/CD Engineering"
exam_objectives:
  - "Thiết kế pipeline CI/CD với Jenkins, GitLab CI và GitHub Actions."
  - "Quản lý artifact, cache, versioning và môi trường deploy."
  - "Tích hợp quality gate và security scan vào pipeline."
  - "Bảo mật CI/CD và xử lý sự cố pipeline ở quy mô lớn."
---

## Đáp án {#answers}

**1. Đáp án đúng là C.**

**Giải thích:**

- **A)** Key cố định `{% raw %}${{ runner.os }}-npm{% endraw %}` không bao giờ thay đổi, nên cache không bao giờ bị invalidate khi `package-lock.json` đổi. Job sẽ dùng dependency cũ — đúng loại lỗi "chạy được trên máy tôi nhưng sai trên CI". Ngoài ra không có `restore-keys` làm giảm tỉ lệ hit khi key đổi.

- **B)** Key theo `{% raw %}${{ github.run_id }}{% endraw %}` là duy nhất cho mỗi lần chạy, nghĩa là cache mới được tạo mỗi lần và **không bao giờ** được tái sử dụng — tốn dung lượng lưu trữ mà không tăng tốc được gì.

- **C)** Đáp án này đúng. Key gắn với hash của lockfile nên cache đổi đúng lúc dependency đổi; `restore-keys` cho phép dùng cache gần nhất (theo tiền tố) khi lockfile vừa thay đổi, giúp lần cài đầu vẫn nhanh; và vì cache là best-effort, job luôn chạy `npm ci` để đảm bảo dependency đúng kể cả khi cache miss.

- **D)** Chỉ restore mà không bao giờ save thì cache không được tạo/gia hạn bởi job của bạn; sau khi cache cũ bị evict, mọi lần chạy đều chậm. Cách này không phải "an toàn hơn" mà chỉ làm mất tác dụng của cache.




**2. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. Action như `aws-actions/configure-aws-credentials` cần xin OIDC token từ GitHub, và việc này chỉ được phép khi job khai báo `id-token: write` trong `permissions`.

- **B)** Đáp án này đúng. Trust policy là tuyến phòng thủ chính: condition trên `sub` (ví dụ `repo:my-org/my-repo:ref:refs/heads/main`) đảm bảo chỉ workflow đúng repo, đúng branch/environment mới assume được role. Trust policy quá rộng nghĩa là repo bất kỳ trên GitHub đều mạo danh được role của bạn.

- **C)** Đáp án này sai. OIDC sinh ra chính là để **loại bỏ** static key: không có `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` nào được lưu. Nếu vẫn còn key tĩnh trong secret, bạn chưa tận dụng được lợi ích của OIDC và vẫn phải rotate định kỳ.

- **D)** Đáp án này đúng. STS trả về credential tạm thời (mặc định khoảng 1 giờ), tự hết hạn — không có key dài hạn để rò rỉ, giảm mạnh nhu cầu rotate và mọi lần assume role đều có dấu vết trong CloudTrail.




**3. Đáp án đúng là A, B và C.**

**Giải thích:**

- **A)** Đáp án này đúng. Cache của GitLab mang tính best-effort: có thể miss do chưa được tạo, bị evict, hoặc storage (S3/MinIO) trục trặc. Vì vậy job luôn phải chạy được đúng khi cache trống, cache chỉ là tối ưu tốc độ.

- **B)** Đáp án này đúng. Artifact được upload ở job này và download ở job sau (`needs`/`dependencies`), đồng thời có thể tải thủ công từ UI hoặc API — rất hữu ích cho test report và debug.

- **C)** Đáp án này đúng. `expire_in` (ví dụ `1 week`) giới hạn thời gian lưu artifact, giúp kiểm soát dung lượng storage; mặc định có thể khá dài nên production nên đặt rõ ràng.

- **D)** Đáp án này sai. Không có gì đảm bảo cache tồn tại lâu dài: nó phụ thuộc cấu hình storage, chính sách evict và trạng thái runner. Chỉ artifact mới là thứ được gắn với pipeline và có vòng đời xác định.




**4. Đáp án đúng là C.**

**Giải thích:**

- **A)** Đáp án này sai. Với sự kiện `pull_request`, workflow chạy trên code của PR **trước khi** merge và trước khi review hoàn tất. Branch protection chỉ chặn merge, không chặn việc code độc hại được execute trên runner.

- **B)** Đáp án này sai và là combo nguy hiểm nổi tiếng. `pull_request_target` chạy trong ngữ cảnh của repository gốc, **có** secret; nếu workflow checkout code từ PR rồi thực thi (test, build script), kẻ tấn công có thể đọc và đánh cắp secret. Đây là con đường leo thang đặc quyền, không phải cách chuẩn để test fork.

- **C)** Đáp án này đúng. Mặc định, workflow `pull_request` từ fork không nhận secret — nhưng code PR vẫn chạy trên runner, nên self-hosted runner vẫn có thể bị chiếm. Biện pháp đúng: không dùng self-hosted runner lâu dài cho repo public; nếu buộc phải dùng thì runner phải ephemeral, cô lập mạng và không mang credential dài hạn.

- **D)** Đáp án này sai. `permissions` chỉ giới hạn quyền của `GITHUB_TOKEN` trong phạm vi API GitHub, không bảo vệ máy runner khỏi code độc hại — kẻ tấn công vẫn đọc được file, biến môi trường, quét mạng nội bộ và cài backdoor trên runner.




**5. Đáp án đúng là D.**

**Giải thích:**

- **A)** Đáp án này sai. Build lại cho production tạo ra một artifact **khác** với thứ đã test ở staging: dependency có thể trôi, layer khác, kết quả build khác. Nó phá vỡ chính lợi ích mà promotion mang lại là "thứ đã kiểm chứng".

- **B)** Đáp án này sai. Tag mới không đảm bảo cùng nội dung nếu bạn build lại; muốn "đổi tên" image thì phải retag/copy **cùng digest**, không phải build lại. Đây là hiểu nhầm phổ biến giữa tag và artifact identity.

- **C)** Đáp án này sai. Deploy từ nhánh `prod` nghĩa là production chạy artifact được build từ một nhánh khác — lại thêm một biến số (merge, conflict, cherry-pick) và không có gì đảm bảo trùng với staging.

- **D)** Đáp án này đúng. Deploy đúng digest đã test qua từng environment, chỉ inject config khác nhau (biến môi trường, ConfigMap/Secret); production có approval gate riêng. Rollback sau này đơn giản là deploy lại digest cũ — không cần rebuild.

**6. Đáp án đúng là B, C và D.**

**Giải thích:**

- **A)** Đáp án này sai. Controller giữ credential, UI và lịch sử build — chạy build nặng ở đây vừa làm chậm/hỏng UI, vừa đặt code không tin cậy ngay cạnh kho credential. Best practice là controller chỉ điều phối, mọi build chạy trên agent.

- **B)** Đáp án này đúng. Agent ephemeral (thường qua Kubernetes plugin) đảm bảo mỗi build có môi trường sạch: không còn trạng thái tích tụ từ build trước, scale theo nhu cầu, và hủy agent sau khi xong. Cần lưu ý cấp service account tối thiểu cho pod agent.

- **C)** Đáp án này đúng. `withCredentials` binding credential vào biến tạm thời trong scope cần thiết, tránh nhúng vào Jenkinsfile (bị commit, bị review thấy) hoặc `echo` ra log — giá trị trong log gần như không thể thu hồi.

- **D)** Đáp án này đúng. Shared library thay đổi ảnh hưởng tới mọi pipeline dùng nó, nên phải version hóa bằng tag (cho phép pipeline chọn version), review như code và có test. Đây là cách chuẩn hóa mà không biến thay đổi nhỏ thành sự cố diện rộng.




**7. Đáp án đúng là D.**

**Giải thích:**

- **A)** Đáp án này sai. Push trước rồi scan nghĩa là image chưa được kiểm chứng đã nằm trong registry — nếu registry là nguồn deploy (GitOps, watcher, hoặc registry dùng chung với môi trường khác), image lỗi có thể bị deploy trước khi scan xong. Trong mô hình fail-closed, gate phải chạy trước publish.

- **B)** Đáp án này sai. Scan ở staging nghĩa là image đã chạy trong môi trường có network và dữ liệu thật, và lỗi bảo mật đã có cơ hội phát tán vào môi trường gần production. Security gate phải đặt càng sớm càng tốt, trước khi artifact được publish.

- **C)** Đáp án này sai. Không base image chính thức nào miễn nhiễm CVE — các bản Node/Alpine đều có CVE mới theo thời gian. Cách tiếp cận đúng không phải tắt scan, mà là gate theo mức severity và bản vá (`--ignore-unfixed`) để chặn cái có thể sửa được.

- **D)** Đáp án này đúng. Build với `load: true` (image nằm local, chưa push), scan `HIGH,CRITICAL` với `--ignore-unfixed` và `exit-code 1`, chỉ push khi pass — đúng mô hình fail-closed. Nếu vì lý do tổ chức buộc phải push trước, hãy coi registry đó như staging: scan lại theo digest, có policy/retention riêng, và không cho deploy thẳng từ đó vào production.




**8. Đáp án đúng là C.**

**Giải thích:**

- **A)** Đáp án này sai. Retry vô hạn che giấu lỗi và hợp pháp hóa thói quen "rerun đến khi xanh"; lỗi thật cũng sẽ bị bỏ qua cùng với flaky, và compute bị đốt vô ích.

- **B)** Đáp án này sai. Xóa ngay lập tức có thể làm mất coverage giá trị trước khi hiểu nguyên nhân. Nếu buộc phải xóa thì nên kèm issue ghi lại điều kiện viết lại — minh bạch, không phải che giấu.

- **C)** Đáp án này đúng. Tam giác xử lý flaky: retry **có giới hạn** kèm cảnh báo để không chặn PR, đo flake rate để biết quy mô, quarantine vào suite nightly, và đầu tư sửa nguyên nhân gốc (timing, shared state, port cố định, phụ thuộc network/timezone).

- **D)** Đáp án này sai. Tắt required status check phá vỡ chính cơ chế bảo vệ nhánh chính: code lỗi có thể merge và pipeline mất đi ý nghĩa "đỏ là dừng lại".

**9. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Đáp án này đúng. Path filter (`paths` trong GitHub Actions, `rules:changes` trong GitLab, `dorny/paths-filter`) giới hạn job chỉ chạy khi package liên quan thay đổi. Đây là tối ưu lớn nhất của CI monorepo — không đốt compute vào phần không đổi.

- **B)** Đáp án này sai. Chạy full pipeline cho mọi commit là tốn kém và chậm, nhưng không phải "cách duy nhất" để đúng: kết hợp filter với job tổng hợp và cache vẫn đảm bảo độ chính xác. Vấn đề thật của path filter là **skip nhầm** — điều được xử lý bằng umbrella job ở phương án C, không phải bằng cách bỏ filter.

- **C)** Đáp án này đúng. Job umbrella (aggregate/gate) làm required check duy nhất, đảm bảo: nếu package A thay đổi thì chạy job của A; nếu file dùng chung hoặc cấu hình CI thay đổi thì chạy full. Cách này kết hợp tốc độ của filter với an toàn của full build.

- **D)** Đáp án này sai. Tách monorepo thành nhiều repo không phải điều kiện bắt buộc, và nó đánh đổi vấn đề CI lấy vấn đề phối hợp (versioning, release đồng bộ, PR liên repo). Hãy thử path filter + cache + build tăng dần trước khi nghĩ đến việc tách repo.




**10. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. Keyless signing dùng OIDC token của workflow để xin certificate ngắn hạn từ Fulcio, ký image, và ghi bản ghi chữ ký vào transparency log Rekor. Trên GitHub Actions điều này đòi hỏi `id-token: write`.

- **B)** Đáp án này đúng. Vì keyless không có public key cố định để quản lý, bước verify xác định tin cậy qua **identity và issuer** mong đợi: `--certificate-identity` (hoặc `--certificate-identity-regexp`) kết hợp `--certificate-oidc-issuer`. Chỉ verify chung chung "có chữ ký" là vô nghĩa vì ai cũng có thể ký.

- **C)** Đáp án này sai. Keyless giải quyết vấn đề quản lý khóa, **không** thay thế việc verify. Fulcio bảo chứng certificate gắn với OIDC identity, nhưng chính bạn phải kiểm tra identity đó có phải pipeline của mình không — nếu không, image do kẻ tấn công ký vẫn pass.

- **D)** Đáp án này đúng. Deploy theo digest để cố định artifact bất biến, và enforce verify ở admission controller (Kyverno, cosign policy-controller) để workload chỉ được tạo khi image có chữ ký khớp identity tin cậy. Ký ở CI mà không enforce ở cluster thì chuỗi bảo vệ vẫn hở.
