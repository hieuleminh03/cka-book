---
layout: answer

title: "Chương 3"
subtitle: "Container & Docker"
exam_objectives:
  - "Hiểu container runtime: namespace, cgroup, UnionFS và kiến trúc Docker/containerd."
  - "Viết Dockerfile hiệu quả: multi-stage build, layer caching và giảm image footprint."
  - "Quản lý image, registry, tagging và bảo mật image."
  - "Thực hành build, run, debug container và chuyển workload sang Kubernetes."
---

## Đáp án {#answers}

**1. Đáp án đúng là A và B.**

**Giải thích:**

- **A)** Đáp án này đúng. Container không ảo hóa phần cứng và không có kernel riêng: nó là process chạy trong namespace/cgroup của kernel host, còn "hệ điều hành" trong image chỉ gồm app cùng vài thư viện user-space cần thiết. Nhờ đó container khởi động tính bằng mili giây thay vì boot cả một hệ điều hành.
- **B)** Đáp án này đúng. Mỗi VM chạy kernel và hệ điều hành đầy đủ trên lớp ảo hóa phần cứng, nên cô lập mạnh hơn nhưng tốn CPU, RAM và dung lượng đĩa cho từng instance; mật độ trên một máy vì thế thấp hơn container nhiều lần.
- **C)** Đáp án này sai. Chính vì dùng chung kernel, container có **mức cô lập yếu hơn** VM: kernel là bề mặt tấn công chung, một lỗ hổng kernel bị khai thác từ container có thể ảnh hưởng toàn bộ node. Cần phân biệt rõ "nhẹ và nhanh" không đồng nghĩa "cô lập mạnh hơn".
- **D)** Đáp án này sai. Image container gắn với kiến trúc CPU (amd64, arm64...); image x86 không chạy nguyên vẹn trên arm64 nếu không có cơ chế dịch (qemu/binfmt) và nó cũng phụ thuộc ABI của kernel host (ví dụ base musl so với glibc). Đây là lý do tồn tại `docker buildx --platform` và multi-arch manifest.

**2. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. `COPY . .` không tách riêng `server.js`; mọi thay đổi trong build context đều làm layer `COPY . .` mất cache. Vì layer này nằm **trước** `RUN npm ci`, mất cache ở nó kéo theo mất cache toàn bộ instruction phía sau — nghĩa là `npm ci` chạy lại từ đầu.
- **B)** Đáp án này đúng. Cache của BuildKit tính theo parent layer + nội dung instruction, nên instruction bị đổi làm mất cache của nó và mọi instruction sau. Cách viết tốt hơn là sao chép `package.json`/`package-lock.json` rồi chạy `npm ci` trước, chỉ `COPY . .` sau cùng: khi code thay đổi, chỉ các layer cuối bị rebuild còn dependency vẫn được cache.
- **C)** Sai. `CMD` đúng là không tạo layer, nhưng nó không liên quan đến vấn đề cache ở đây: layer `COPY . .` và `RUN npm ci` vẫn bị invalidate vì chúng đứng trước phần nội dung thay đổi.
- **D)** Sai. Docker/BuildKit không phân tích nội dung file để "tách" `server.js` ra khỏi `COPY`. Chiến lược cache là trách nhiệm của người viết Dockerfile, không phải cơ chế tự động.

**3. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. Đây là hành vi của shell form nếu `CMD` không bị thay, nhưng trong câu hỏi, đối số truyền trên dòng lệnh `docker run` thay thế `CMD`. Ngoài ra `ENTRYPOINT ["/app"]` là exec form nên không có `/bin/sh -c` chen vào.
- **B)** Đáp án này đúng. Khi image có `ENTRYPOINT ["/app"]` và `CMD ["--port=8080"]`, `CMD` chỉ là tham số mặc định cho `ENTRYPOINT`. Khi bạn truyền `--port=9090` trong `docker run`, Docker thay thế toàn bộ `CMD` bằng đối số mới, kết quả process chạy là `/app --port=9090`.
- **C)** Sai. `--port=8080` bị thay thế hoàn toàn bởi `--port=9090`; Kubernetes và Docker đều xử lý override theo cách "đối số dòng lệnh thay CMD", không merge từng phần.
- **D)** Sai. Việc `CMD` bị ghi đè là hành vi bình thường, không phải lỗi. Muốn ghi đè cả `ENTRYPOINT` phải dùng cờ `--entrypoint`.

**4. Đáp án đúng là C.**

**Giải thích:**

- **A)** Sai. `docker attach` chỉ gắn stdio của bạn vào process chính (PID 1) của container, không mở shell và không giúp chạy lệnh nếu process không đọc stdin như terminal. Attach nhầm còn có thể gửi tín hiệu làm chết container.
- **B)** Sai. `docker exec` không "tự cung cấp binary": nó exec lệnh **bên trong filesystem của container**, nên với image distroless không có `ip` (thậm chí không có `sh`), lệnh thất bại với `executable file not found in $PATH`.
- **C)** Đáp án này đúng. `docker top <container>` liệt kê process của container kèm PID thật trên host. Từ host (có quyền root), `nsenter --target <PID> --net -- ip -br a` nhảy thẳng vào network namespace của container và chạy lệnh bằng binary của **host** — cách chuẩn để debug image tối giản mà không cần cài gì vào container.
- **D)** Sai. `docker logs` chỉ hiển thị stdout/stderr của process chính. Interface mạng và bảng route là trạng thái của kernel namespace, không xuất hiện trong log trừ khi ứng dụng tự in ra.

**5. Đáp án đúng là A, B và C.**

**Giải thích:**

- **A)** Đáp án này đúng. Image cuối chỉ chứa stage runtime cùng những gì bạn chủ động copy sang. Stage build (compiler, toolchain, cache) bị bỏ hoàn toàn khỏi kết quả — đây chính là lý do image giảm từ hàng trăm MB xuống vài MB.
- **B)** Đáp án này đúng. `COPY --from=<stage>` là cơ chế kết nối các stage; nó cũng có thể lấy file trực tiếp từ một image khác trên registry mà không cần định nghĩa stage tương ứng.
- **C)** Đáp án này đúng. `scratch` là base image hoàn toàn trống. Ứng dụng gọi HTTPS cần bundle CA certificate (thường copy từ builder: `/etc/ssl/certs/ca-certificates.crt`) và có thể cần `tzdata` nếu dùng múi giờ; thiếu certificate là nguyên nhân lỗi `x509: certificate signed by unknown authority` rất phổ biến khi chuyển sang scratch.
- **D)** Sai. Stage build không được đưa vào image cuối ở bất kỳ dạng nén nào. Nếu bạn thấy image vẫn lớn, nguyên nhân thường là base image của stage runtime lớn, hoặc bạn copy nhầm cả thư mục (kể cả build cache) từ stage build sang.

**6. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. Layer không phải "khóa cứng" theo nghĩa đó — bạn hoàn toàn có thể xóa file ở layer sau, nhưng việc xóa chỉ ảnh hưởng đến cái nhìn của container, không ảnh hưởng đến dữ liệu đã ghi ở layer trước.
- **B)** Đáp án này đúng. Mỗi instruction tạo một layer chỉ đọc. `RUN rm -rf ...` ở layer trên cùng chỉ tạo các file "whiteout" che `/var/lib/apt/lists/*`; dữ liệu thật vẫn nằm trong layer của `apt-get install` và vẫn được tính vào kích thước image. Cách đúng là gộp cài đặt và dọn dẹp vào cùng một `RUN` để chúng nằm trong một layer duy nhất.
- **C)** Sai. `apt-get` ghi vào filesystem của image (layer đang build), không ghi vào Docker volume. Volume chỉ được mount khi container chạy và không được tính vào kích thước image.
- **D)** Sai. Build cache là cơ chế tăng tốc build, không phải một bản sao nội dung nằm trong image. Kể cả khi bạn xóa build cache (`docker builder prune`), kích thước image vẫn không đổi vì dữ liệu nằm trong chính layer của image.

**7. Đáp án đúng là A, B và D.**

**Giải thích:**

- **A)** Đáp án này đúng. Base image là nguồn CVE lớn nhất: `golang:1.24` chứa cả Debian/glibc, compiler và hàng trăm package chỉ cần cho việc build. Chuyển sang multi-stage với runtime distroless/alpine loại bỏ phần lớn package đó, còn rebuild định kỳ giúp nhận bản vá của base image mới — vì CVE trong OS package chỉ được sửa khi bạn đổi sang image đã vá.
- **B)** Đáp án này đúng. Ghim digest đảm bảo mọi môi trường chạy đúng cùng một artifact (loại trừ rủi ro tag bị đẩy đè), còn quy trình cập nhật có kiểm soát giúp việc nâng base image diễn ra có chủ đích thay vì "build hôm nào biết hôm đó".
- **C)** Sai. `cosign sign` chứng minh **nguồn gốc và tính toàn vẹn** của image (ai ký, bị sửa chưa), không sửa hay loại bỏ package. Ký một image đầy CVE vẫn cho ra một image đầy CVE, chỉ được xác thực là "đúng của bạn". Scan và sign là hai lớp bổ trợ.
- **D)** Đáp án này đúng. `--exit-code 1` biến kết quả scan thành cổng chặn trong CI: image có CVE mức chỉ định sẽ không được đẩy lên/không được deploy. Đây là cách hiện thực hóa chính sách "không đưa lỗ hổng nghiêm trọng vào môi trường chạy".

**8. Đáp án đúng là C.**

**Giải thích:**

- **A)** Sai. `latest` với `imagePullPolicy: Always` chỉ đảm bảo **pull lại** — nội dung `latest` giữa các lần pull có thể khác nhau, nên bạn không biết chính xác node đang chạy artifact nào và rollback không xác định. Đây là cấu hình nguy hiểm kép: vừa mơ hồ về phiên bản, vừa tăng tải registry.
- **B)** Sai. `stable` cũng là tag trôi nổi: nội dung thay đổi theo mỗi lần pipeline gắn lại tag. Nó tiện cho môi trường phát triển nhưng không đáp ứng yêu cầu "rollback xác định" ở production, và việc hai lần deploy cùng tên tag có thể chạy hai code khác nhau.
- **C)** Đáp án này đúng. Thực hành chuẩn là tag bất biến kèm phiên bản rõ ràng (semver, git SHA) và/hoặc pin theo digest. Digest là hash nội dung image nên cùng một digest luôn là cùng một artifact; rollback chỉ đơn giản là trỏ về tag/digest trước đó. Kết hợp bật immutable tag trên registry để chống ghi đè.
- **D)** Sai. Hash của node không liên quan đến định danh artifact; nó chỉ cho biết node nào đã pull, không cho biết nội dung image là gì. Cách nhận diện artifact phải dựa trên tag bất biến hoặc digest của image.

**9. Đáp án đúng là B.**

**Giải thích:**

- **A)** Sai. `--link` là cơ chế cũ, chỉ hoạt động một chiều, khó quản lý khi số container tăng, và Docker đã khuyến nghị thay thế bằng user-defined network từ lâu. Dùng `--link` không giải quyết được vấn đề một cách bền vững.
- **B)** Đáp án này đúng. Trên user-defined bridge network, Docker chạy embedded DNS tại `127.0.0.11` cho mỗi container, phân giải tên container và `--network-alias` thành IP nội bộ. Đây là lý do `curl http://api:8080` hoạt động giữa hai container cùng mạng, và cũng là mô hình tổ tiên của Service DNS trong Kubernetes.
- **C)** Sai. `--network host` bỏ hoàn toàn network namespace, khiến hai container dùng chung cổng của host và dễ xung đột port; đây là cách đối phó thô bạo, không áp dụng được cho nhiều container, và không giúp "gọi bằng tên" đúng nghĩa.
- **D)** Sai. Default bridge **không có** DNS nội bộ theo tên container; `--hostname` chỉ đổi hostname của container, không tạo bản ghi phân giải tên cho các container khác. Đây chính là nguyên nhân lỗi "could not resolve host" trên default bridge.

**10. Đáp án đúng là A, B và C.**

**Giải thích:**

- **A)** Đáp án này đúng. dockershim bị gỡ từ Kubernetes v1.24; kubelet dùng CRI để nói chuyện trực tiếp với containerd hoặc CRI-O. Đây là lý do trong cluster hiện đại, việc debug runtime dùng `crictl` (CRI) thay vì lệnh `docker`.
- **B)** Đáp án này đúng. Image OCI là chuẩn mở định nghĩa manifest, config và layer; cả Docker và containerd đều đọc chuẩn này. Vì vậy image build bằng Docker chạy nguyên vẹn trên cluster dùng containerd, và ngược lại image build bằng buildah/podman cũng chạy được trên Docker.
- **C)** Đáp án này đúng. Trong kind, Docker daemon trên host và containerd trong node kind là hai image store độc lập; Docker đã pull/build image không có nghĩa node kind thấy được. `kind load docker-image` chuyển image (thực chất là `docker save` rồi import vào containerd) vào node; kiểm chứng bằng `crictl images` chạy bên trong node.
- **D)** Sai. Kubernetes không yêu cầu image phải build trên node, cũng không yêu cầu Docker daemon. Node chỉ cần pull image đã được đẩy lên registry (hoặc được import vào runtime) và chạy nó qua CRI. Việc build thuộc pipeline CI/CD, hoàn toàn tách khỏi node.
