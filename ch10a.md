---
layout: answer

title: "Chương 10"
subtitle: "Supply Chain Security (CKS)"
exam_objectives:
  - "Giảm base image footprint."
  - "Hiểu supply chain: SBOM, CI/CD, artifact repository."
  - "Bảo vệ supply chain: registry tin cậy, ký và xác thực artifact."
  - "Static analysis workload và container image: Kubesec, KubeLinter, Trivy."
---

## Đáp án {#answers}

**1. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** Đáp án này đúng. Multi-stage build cho phép stage runtime chỉ nhận những gì được `COPY --from=` từ stage build. Compiler, toolchain, source code và các gói phục vụ build không tồn tại trong image cuối, nên kích thước giảm mạnh và số package — cùng số CVE tiềm năng — cũng giảm theo.

- **B)** Đáp án này đúng. Distroless được thiết kế để không có shell, package manager hay công cụ tương tác, nên `kubectl exec ... sh` sẽ thất bại. Cách debug đúng là dùng ephemeral container (`kubectl debug`), biến thể image `:debug` của distroless, hoặc quan sát qua log/metrics.

- **C)** Đáp án này sai. Không có quy luật "image nhỏ hơn thì luôn ít CVE hơn" tại mọi thời điểm — kết quả phụ thuộc dữ liệu CVE, các gói cụ thể và việc base image có được cập nhật hay không. Điều đúng là image ít thành phần hơn có bề mặt tấn công nhỏ hơn và ít CVE tiềm năng hơn; vẫn phải scan thực tế để biết con số.

- **D)** Đáp án này sai. Thêm `curl`, `bash`, `netcat` vào runtime image làm tăng bề mặt tấn công: đó chính là những công cụ kẻ tấn công cần sau khi khai thác được lỗ hổng (tải payload, mở reverse shell, pivot). Khả năng quan sát nên đến từ logging, metrics, tracing và ephemeral container — không phải từ việc cài thêm công cụ vào image.

**2. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. `ubuntu:24.04` kèm `curl`, `bash` và `ca-certificates` mang theo hàng chục gói không cần thiết, chạy mặc định bằng root và có bề mặt tấn công lớn. Đây là ví dụ của base "full" mà objective giảm footprint muốn tránh.

- **B)** Đáp án này đúng. Binary Go build tĩnh (`CGO_ENABLED=0`) không cần libc động, nên chạy được trên `distroless/static-debian12:nonroot`. Image cuối chỉ có vài MB, không shell, không package manager và đã được cấu hình chạy non-root (UID 65532) — đúng cả ba yêu cầu: nhỏ nhất, ít bề mặt nhất, non-root.

- **C)** Sai. Giữ nguyên toolchain Go trong image cuối khiến image nặng hàng trăm MB, mang theo nhiều package và CVE không liên quan đến runtime. Ngoài ra image `golang` mặc định chạy root. Nếu muốn tái sử dụng build cache thì dùng cache của CI, không phải nhét toolchain vào image production.

- **D)** Sai vì build trực tiếp trong image cuối đồng nghĩa toàn bộ build tool (`go`, module cache, có thể cả git) nằm trong artifact được deploy. Đây là cách làm phổ biến của người mới và là ví dụ kinh điển cho việc "image phình to vì build tool".

**3. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Đáp án này đúng. SBOM liệt kê thành phần (package, version, nguồn, license) theo định dạng chuẩn — phổ biến nhất là SPDX và CycloneDX. Đây là mô tả "có gì bên trong artifact", phục vụ cả bảo mật lẫn compliance.

- **B)** Đáp án này sai. SBOM không chứa thông tin lỗ hổng; nó chỉ là danh sách thành phần. Muốn biết thành phần nào có CVE, bạn phải đưa SBOM qua scanner (`trivy sbom`, `grype sbom:...`) hoặc scan trực tiếp image. SBOM và scan CVE là hai việc bổ trợ, không thay thế nhau.

- **C)** Đáp án này đúng. Đây là giá trị thực tế lớn nhất của SBOM: khi một CVE mới được công bố, bạn tra trong SBOM đã lưu theo từng release để xác định ngay image nào bị ảnh hưởng, thay vì phải scan lại toàn bộ hoặc chờ từng team trả lời.

- **D)** Đáp án này sai ở cả hai vế. `syft` chạy hoàn toàn local và nhận nhiều nguồn: `docker:`, `podman:`, `registry:`, `dir:`, `file:` — không cần cluster. Và SBOM là file JSON có thể dùng lại vô số lần để scan về sau, kể cả với artifact đã nghỉ hưu.

**4. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. `.trivyignore` dành cho false positive đã được review, không phải cho việc che toàn bộ CVE HIGH. Pipeline sẽ xanh nhưng rủi ro thật vẫn còn nguyên; đây là hành vi che bệnh và là điều tệ nhất có thể làm với một báo cáo scan.

- **B)** Đáp án này đúng và là quy trình chuẩn: ưu tiên các CVE có bản fix (sửa được ngay bằng nâng base/dependency rồi rebuild), giảm dài hạn bằng cách chuyển sang base nhỏ hơn (distroless), ghi nhận có trách nhiệm các CVE chưa có fix vào VEX/backlog, và ghim digest cho artifact mới để cố định kết quả đã kiểm chứng.

- **C)** Sai. Hạ severity filter chỉ làm thay đổi cái bạn nhìn thấy, không thay đổi thực tế image đang chạy. Đây là biến thể tinh vi hơn của việc che báo cáo, và sẽ khiến bạn mất khả năng phát hiện vấn đề về sau.

- **D)** Sai. Chuyển sang distroless là một phần của giải pháp (giảm bề mặt và số CVE tiềm năng) nhưng không có nghĩa "không cần quan tâm CVE còn lại". Image distroless vẫn có thể dính CVE trong glibc/openssl hoặc trong dependency của ứng dụng, và vẫn phải được theo dõi định kỳ.

**5. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** Đáp án này đúng. `cosign.key` là private key được mã hóa bằng `COSIGN_PASSWORD`; để lộ nó đồng nghĩa ai cũng ký được image "hợp lệ". Trong production nên lưu trong KMS (`--key awskms://...`, `gcpkms://...`) hoặc dùng keyless, không để file key trong repo hay trong CI ở dạng plain text.

- **B)** Đáp án này đúng. cosign ký lên digest của image, và digest thay đổi ngay khi nội dung (layer, config) thay đổi. Vì vậy kẻ tấn công có ghi được tag cũng không làm chữ ký "chuyển" sang image mới; policy verify (đặc biệt khi bật `mutateDigest`) sẽ phát hiện ra.

- **C)** Sai. Verify chỉ cần public key (hoặc certificate identity issuer với keyless). Chữ ký số được kiểm tra bằng public key, không cần private key; private key chỉ dùng để ký. Nhầm lẫn này khá phổ biến và là dấu hiệu chưa hiểu mô hình.

- **D)** Sai. Mỗi lần rebuild, image có digest mới, nên chữ ký cũ không còn áp dụng cho image mới. Pipeline phải ký lại sau mỗi lần build; đây cũng là lý do bước "sign" nằm trong CI/CD chứ không phải thao tác làm một lần thủ công.

**6. Đáp án đúng là A và C.**

**Giải thích:**

- **A)** Đáp án này đúng. Keyless bỏ private key dài hạn: cosign dùng OIDC token (ví dụ của GitHub Actions hoặc đăng nhập tương tác) để đổi lấy certificate ngắn hạn từ Fulcio, gắn chặt với identity đã được xác thực bởi OIDC provider.

- **B)** Đáp án này sai. Rekor là transparency log lưu **bằng chứng** của việc ký (signature, certificate, thời điểm), không lưu private key — Rekor không thể dùng để ký thay bạn. Bản thân keyless cũng không có private key dài hạn để lưu.

- **C)** Đáp án này đúng. Khi verify keyless, không có public key cố định để so; thay vào đó bạn khai báo **identity và issuer** mà bạn chấp nhận (`--certificate-identity`, `--certificate-oidc-issuer`, hoặc `subject`/`issuer` trong policy Kyverno). Vì vậy phải ghim thật chặt — chấp nhận `https://github.com/*` gần như vô hiệu hóa lớp bảo vệ này.

- **D)** Đáp án này sai. Keyless giảm rủi ro quản lý khóa nhưng không loại bỏ rủi ro: identity OIDC có thể bị chiếm (token rò rỉ), policy danh tính có thể quá rộng, và bạn phải tin tưởng hạ tầng Fulcio/Rekor cùng OIDC provider. Ngoài ra keyless còn phụ thuộc mạng, khó dùng trong môi trường air-gapped nếu không có hạ tầng tương đương nội bộ.

**7. Đáp án đúng là A, C và D.**

**Giải thích:**

- **A)** Đáp án này đúng. Policy khớp pod trong namespace `secure-app` với pattern image đã khai báo; vì `required: true` và `validationFailureAction: Enforce`, image khớp pattern nhưng không có chữ ký sẽ bị admission webhook từ chối với lỗi `no signatures found`.

- **B)** Đáp án này sai. Mục đích của attestor `keys.publicKeys` là chỉ chấp nhận chữ ký verify được bằng đúng public key đó. Image ký bằng key khác sẽ không qua được (lỗi dạng `no matching signatures`/`invalid signature`), bất kể nó nằm trong registry nào — đây chính là điểm phân biệt giữa "allowlist registry" (chỉ kiểm tra hình dạng tham chiếu) và "verify chữ ký" (kiểm tra nguồn gốc thật).

- **C)** Đáp án này đúng. Kyverno phải tự gọi registry để tải chữ ký và manifest, vì vậy controller cần: DNS phân giải được tên registry, kết nối mạng tới registry (NetworkPolicy/egress), tin cậy TLS (hoặc cờ insecure cho lab) và credential nếu registry private. Nếu các điều kiện này thiếu, verification fail; với `failurePolicy: Fail`, pod bị chặn — đây là kiểu sự cố "chặn nhầm do hạ tầng verify" rất đáng lưu ý.

- **D)** Đáp án này đúng. `mutateDigest: true` khiến pod spec được cập nhật từ tag sang digest đã verify. Nhờ đó pod chạy đúng artifact đã được xác thực, loại bỏ khoảng trống TOCTOU giữa thời điểm verify và thời điểm kubelet pull.

**8. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. `docker build --no-cache` chỉ ảnh hưởng lần build trên máy bạn; nó không giải thích việc pod mới vẫn chạy code cũ, và cũng không xóa image đã bị kubelet cache trên node. Vấn đề nằm ở tham chiếu image và cache của node, không phải cache build.

- **B)** Đáp án này đúng. Tag `:v1` là con trỏ mutable; kubelet đã có image ứng với digest cũ trong cache và với `imagePullPolicy: IfNotPresent` (mặc định cho tag cụ thể) nó không pull lại. Cách sửa đúng và bền vững là deploy theo digest `@sha256:...` (hoặc dùng tag immutable mới cho mỗi build, ví dụ `v1.2.4`). `imagePullPolicy: Always` chỉ là giải pháp tạm, không giải quyết gốc vấn đề và còn mở đường cho kẻ tấn công ghi đè tag đến được node.

- **C)** Sai. Kubelet không tự xóa image cache khi restart, và bản thân việc restart kubelet cũng không phải cách xử lý sự cố deploy. Kể cả nếu image cache bị xóa và node pull lại, nếu kubelet "thấy" tag mới thì đó là điều bạn muốn — nhưng vấn đề cốt lõi vẫn là tag mutable; không có digest/tag immutable thì sự cố sẽ tái diễn.

- **D)** Sai ngay ở tiền đề: Kubernetes hỗ trợ cả tag và digest trong trường `image`. Nguyên nhân của sự cố không phải "dùng tag là duy nhất" mà là tag mutable kết hợp pull policy và cache — và cách sửa tốt nhất vẫn là ghim digest (hoặc tag immutable), chứ không phải cấm dùng tag.

**9. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** Đáp án này đúng. Trivy là công cụ "một cửa": `trivy image` cho CVE, `trivy config` cho misconfiguration trong IaC/manifest (bao gồm Kubernetes YAML, Helm, Dockerfile, Terraform), và `--scanners secret` để phát hiện secret bị lộ. Ngoài ra còn `trivy sbom` để scan lại SBOM đã lưu.

- **B)** Đáp án này đúng. KubeLinter chỉ đọc YAML/Helm chart và kiểm tra theo bộ check best practice, không cần cluster cũng không cần quyền vào API server. Nhờ vậy nó chạy được ở pre-commit, trên laptop dev và trong CI — đúng tinh thần "shift left".

- **C)** Sai. Kubesec chấm điểm **cấu hình bảo mật của manifest** (ví dụ `privileged: true` −30 điểm, `hostNetwork: true` −9 điểm); nó không đọc filesystem image và không có cơ sở dữ liệu CVE. Vì vậy Kubesec không thể thay thế Trivy — hai công cụ nhìn vào hai lớp khác nhau.

- **D)** Sai. Các công cụ dùng nguồn dữ liệu và cách map khác nhau (NVD, GHSA, vendor advisory; check của KubeLinter so với KSV check của Trivy), nên kết quả thường không trùng khớp. Cách làm đúng là chọn một công cụ làm nguồn chính cho gate, dùng công cụ còn lại để đối chiếu khi điều tra — thay vì giả định "chạy một cái là đủ".

**10. Đáp án đúng là A, B và C.**

**Giải thích:**

- **A)** Đáp án này đúng. Digest ghim đúng một artifact; kẻ tấn công có ghi đè tag thì manifest của bạn vẫn trỏ tới digest cũ (image sạch) và không bị ảnh hưởng. Đây là biện pháp rẻ nhất và hiệu quả nhất, đồng thời là điều nên enforce bằng policy admission.

- **B)** Đáp án này đúng. Immutable tag chặn hẳn hành vi ghi đè tag đã tồn tại — kẻ tấn công buộc phải tạo tag mới, làm tăng khả năng bị phát hiện. RBAC chặt (tài khoản CI chỉ push được vào repo của mình) giới hạn thiệt hại nếu credential bị chiếm, và audit log giúp truy vết về sau.

- **C)** Đáp án này đúng. Verify chữ ký tại admission là lớp phòng thủ trực tiếp trước kịch bản trong đề bài: image backdoor không có chữ ký hợp lệ sẽ bị deny. Kèm `mutateDigest: true`, pod chạy đúng digest đã verify, nên ngay cả khi tag bị đổi giữa chừng thì pod vẫn an toàn.

- **D)** Sai. `imagePullPolicy: Always` chỉ đảm bảo kubelet luôn kéo lại image theo tag — nghĩa là nó cũng kéo chính image độc hại mà kẻ tấn công vừa push. Đây là lựa chọn ngược lại với mục tiêu bảo vệ, và là lý do cần nhấn mạnh: pull policy không bao giờ là biện pháp bảo mật supply chain.
