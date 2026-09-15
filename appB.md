---
layout: chapter

title: "Phụ lục B"
subtitle: "Lab & môi trường ôn tập"

previous_link: "appA.html"
previous_title: "Phụ lục A. Cheat sheet"
next_link: "appC.html"
next_title: "Phụ lục C. Lộ trình học 8–10 tuần"
---

CKA và CKS đều là kỳ thi performance-based, nên môi trường lab không phải phần phụ mà là điều kiện bắt buộc: bạn chỉ thực sự học khi gõ lệnh trên cluster thật. Phụ lục này hướng dẫn dựng môi trường local (kind, k3s, minikube, kubeadm trên VM), dùng cloud free tier an toàn, khai thác Killer.sh và các tài nguyên miễn phí.

Mục tiêu tối thiểu: một cluster kind nhiều node chạy được, một cách dựng cluster bằng kubeadm để luyện upgrade/etcd, và một quy trình dọn dẹp để không phát sinh chi phí. Tất cả ví dụ dùng Kubernetes v1.33, kubectl v1.33.

## Nội dung phụ lục {#appendix-content}

- [Yêu cầu máy](#requirements)
- [So sánh các lựa chọn cluster local](#cluster-options)
- [kind — lựa chọn chính](#kind)
- [k3s — nhẹ cho máy yếu](#k3s)
- [minikube](#minikube)
- [Cluster kubeadm trên 2 VM](#kubeadm-vm)
- [AWS free tier an toàn](#aws-free-tier)
- [LocalStack cho Terraform](#localstack)
- [Killer.sh — dùng 2 lượt hiệu quả](#killersh)
- [Tài nguyên miễn phí](#free-resources)
- [Checklist môi trường theo chương](#chapter-checklist)

---

## Yêu cầu máy {#requirements}

| Cấu hình | CPU | RAM | Disk trống | Phù hợp |
|---|---|---|---|---|
| Tối thiểu | 2 core | 8 GB | 30 GB | k3s/minikube 1 node, chương 1–4 |
| Khuyến nghị | 4 core | 16 GB | 60 GB | kind 1 control-plane + 2 worker, chạy đủ lab CKA/CKS |
| Thoải mái | 8 core | 32 GB | 100 GB | nhiều cluster song song, VM kubeadm, Cilium/Falco |

Trên Windows, cách ổn định nhất là Docker Desktop với backend WSL2 rồi chạy mọi thứ trong WSL2 (Ubuntu). Lưu ý:

- kind và minikube (driver docker) cần **Linux container**; đừng bật chế độ Windows containers.
- Cấp phát RAM cho WSL2 bằng file `C:\Users\<ten-ban>\.wslconfig`, sau đó chạy `wsl --shutdown`:

```ini
[wsl2]
memory=10GB
processors=4
swap=2GB
```

- Nếu máy chỉ có 8 GB RAM: dùng k3s thay kind, chỉ chạy một cluster tại một thời điểm và tắt Docker Desktop khi không dùng.
- Disk là tài nguyên dễ cạn nhất: image Kubernetes + Calico + Cilium + Falco có thể chiếm 15–25 GB. Dọn định kỳ bằng `docker system prune -af --volumes`.

---

## So sánh các lựa chọn cluster local {#cluster-options}

| Nền tảng | Cài đặt | Đa node | Dùng cho | Ghi chú |
|---|---|---|---|---|
| kind | Qua Docker | Có | Lab CKA/CKS chính (chương 2–11, 15) | Tạo/xoá nhanh, cấu hình node bằng YAML, sát kubeadm |
| k3s | 1 script | Có (thêm agent) | Máy yếu, lab runtime security | Nhẹ, mặc định có Traefik thay ingress-nginx |
| minikube | Binary | Có (`--nodes`) | Người mới, cần addons nhanh | Addons tiện nhưng chậm hơn kind |
| kubeadm trên VM | Thủ công | Có | Chương 5 (upgrade/etcd), chương 8 (AppArmor/seccomp) | Sát production nhất, tốn tài nguyên |

Chiến lược thực dụng: dùng **kind** cho 90% lab, thêm **2 VM kubeadm** cho các bài upgrade/etcd và kernel hardening. Không cần cả bốn nền tảng.

---

## kind — lựa chọn chính {#kind}

### Cài đặt {#kind-install}

```bash
# Windows (PowerShell): winget install Kubernetes.kind
# macOS/Linux: brew install kind
kind version                    # cần v0.29 trở lên để hỗ trợ node image v1.33
docker version
kubectl version --client
```

### Cấu hình multi-node {#kind-config}

Tạo file `kind-config.yaml` với 1 control-plane và 2 worker, map sẵn port 80/443 cho Ingress:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cka
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

```bash
# Tạo cluster, ghim version để khớp với sách
kind create cluster --name cka --image kindest/node:v1.33.1 --config kind-config.yaml
kubectl config use-context kind-cka
kubectl get nodes -o wide

# Xem và dọn cluster
kind get clusters
kind export kubeconfig --name cka
kind delete cluster --name cka
```

### Load image và vào node {#kind-image}

```bash
# Image build trên host phải được nạp vào node kind (containerd riêng)
docker build -t myapp:dev .
kind load docker-image myapp:dev --name cka

# Kiểm chứng image đã có trong node
docker exec cka-worker crictl images | grep myapp
docker exec -it cka-control-plane bash
```

Vài kinh nghiệm:

- Tạo lại cluster mới cho mỗi chương (`kind delete cluster --name cka` rồi tạo lại) để tránh rác từ lab trước.
- kind không thêm node vào cluster đang chạy; muốn đổi số node thì sửa config và tạo lại.
- Lab NetworkPolicy cần CNI hỗ trợ: kind mặc định dùng kindnet không enforce NetworkPolicy đầy đủ, hãy cài Calico theo hướng dẫn chương 6.
- Lab Cilium/encryption cần thêm `disableDefaultCNI: true` trong `networking` của config — xem chương 9.
- Nếu pod kẹt `ImagePullBackOff` vì mạng chậm, ưu tiên `kind load docker-image` thay vì pull trong cluster.

---

## k3s — nhẹ cho máy yếu {#k3s}

```bash
# Cài server (ghim version để khớp sách)
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.33.3+k3s1 sh -
sudo systemctl status k3s

# Lấy kubeconfig cho user hiện tại
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$(id -u):$(id -g)" ~/.kube/config
chmod 600 ~/.kube/config
kubectl get nodes
```

Thêm worker (trên máy/VM thứ hai):

```bash
# Trên server: lấy token
sudo cat /var/lib/rancher/k3s/server/node-token

# Trên agent
curl -sfL https://get.k3s.io | K3S_URL=https://<server-ip>:6443 K3S_TOKEN=<token> sh -
```

Lưu ý khi dùng k3s ôn CKS:

- k3s đóng gói sẵn Traefik, metrics-server và local-path provisioner — khác cluster kubeadm. Muốn giống sách, cài lại bằng `--disable traefik` rồi dùng ingress-nginx.
- `k3s kubectl` luôn hoạt động kể cả khi chưa cấu hình `KUBECONFIG`.
- AppArmor/seccomp lab chạy được nhưng phụ thuộc kernel host; kiểm tra `aa-status` và `/proc/sys/kernel/seccomp`.
- Gỡ cài đặt: `/usr/local/bin/k3s-uninstall.sh` (agent dùng `k3s-agent-uninstall.sh`).

---

## minikube {#minikube}

```bash
# Cài: winget install Kubernetes.minikube / brew install minikube
minikube start --driver=docker --kubernetes-version=v1.33.1 \
  -p cka --nodes=2 --cpus=2 --memory=4g
minikube profile list ; minikube status -p cka
minikube image load myapp:dev -p cka
minikube addons enable ingress ; minikube addons list
minikube service web -p cka --url
minikube ssh -p cka
minikube stop -p cka ; minikube delete -p cka
```

minikube phù hợp khi bạn muốn addon bật bằng một lệnh hoặc mới làm quen Kubernetes. Nhược điểm: chậm hơn kind, ít dùng trong tài liệu CKS và một số lab kernel (AppArmor, audit log) khó tiếp cận hơn.

---

## Cluster kubeadm trên 2 VM {#kubeadm-vm}

Phần này phục vụ chương 5 (upgrade cluster, backup/restore etcd) và chương 8 (hardening kernel). Yêu cầu: 2 VM Ubuntu 24.04, mỗi máy 2 vCPU / 4 GB RAM / 20 GB disk, cùng dải mạng nội bộ.

Tạo VM bằng Multipass (Windows/macOS/Linux) hoặc Vagrant:

```bash
# Multipass
multipass launch --name cp --cpus 2 --memory 4G --disk 20G 24.04
multipass launch --name worker1 --cpus 2 --memory 4G --disk 20G 24.04
multipass list ; multipass shell cp
```

```ruby
# Vagrantfile (rút gọn) nếu dùng VirtualBox/Hyper-V
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/noble64"
  config.vm.define "cp" do |cp|
    cp.vm.hostname = "cp"
    cp.vm.network "private_network", ip: "192.168.56.10"
    cp.vm.provider "virtualbox" do |vb|
      vb.memory = 4096
      vb.cpus = 2
    end
  end
end
```

Chuẩn bị trên **cả hai node** (swap, kernel module, containerd, package Kubernetes):

```bash
sudo swapoff -a && sudo sed -i '/ swap / s/^/#/' /etc/fstab
sudo modprobe overlay br_netfilter
printf 'overlay\nbr_netfilter\n' | sudo tee /etc/modules-load.d/k8s.conf
printf 'net.bridge.bridge-nf-call-iptables = 1\nnet.ipv4.ip_forward = 1\n' | sudo tee /etc/sysctl.d/k8s.conf
sudo sysctl --system

sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd && containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Khởi tạo control plane (trên `cp`):

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=192.168.56.10
mkdir -p ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown "$(id -u):$(id -g)" ~/.kube/config
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/calico.yaml
kubeadm token create --print-join-command
```

Join worker và verify:

```bash
# Trên worker1, chạy lệnh in ra từ bước trên
sudo kubeadm join 192.168.56.10:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

# Trên cp
kubectl get nodes
kubectl get pods -A
```

Khi đã có 2 VM này, bạn luyện được đúng chuỗi lệnh của đề thi: `kubeadm upgrade plan/apply`, `drain/uncordon`, `etcdctl snapshot save/restore`, sửa `/var/lib/kubelet/config.yaml`, nạp AppArmor profile. Đây là phần mà kind không thể thay thế.

---

## AWS free tier an toàn {#aws-free-tier}

Tài khoản AWS là con dao hai lưỡi: học cloud rất tốt, nhưng một tài nguyên bỏ quên có thể tốn vài chục đô mỗi tháng.

- **EKS không nằm trong free tier**: control plane khoảng 0.10 USD/giờ (~73 USD/tháng) cộng node EC2, NAT Gateway, ELB. Đừng tạo EKS nếu chưa có kế hoạch dọn trong ngày.
- **EC2**: dòng t3.micro/t2.micro thuộc free tier cũ (750 giờ/tháng trong 12 tháng đầu). Từ 15/07/2025, tài khoản mới chuyển sang chương trình **credit-based** (nhận credit ban đầu và credit khi hoàn thành một số hoạt động, hiệu lực 6 tháng) — hãy kiểm tra mục Billing → Free Tier trong console trước khi làm lab.
- **Luôn bật cảnh báo chi phí** trước khi tạo tài nguyên đầu tiên. Ngoài ra có thể xem nhanh chi phí bằng CLI:

```bash
aws ce get-cost-and-usage --time-period Start=2026-09-01,End=2026-09-15 \
  --granularity DAILY --metrics UnblendedCost
```

- Sau mỗi lab Terraform, chạy `terraform destroy` rồi kiểm tra tay các tài nguyên hay bị bỏ sót: Elastic IP đang rảnh, EBS volume `available`, snapshot, NAT Gateway, load balancer.
- Không dùng access key cá nhân cho lab Terraform trên repo công khai; dùng profile riêng và tài khoản học tập tách khỏi tài khoản làm việc.
- Với hầu hết lab Terraform trong sách, LocalStack là lựa chọn miễn phí và không cần lo dọn dẹp.

---

## LocalStack cho Terraform {#localstack}

LocalStack giả lập API AWS ngay trên máy, phù hợp để luyện `terraform plan/apply` mà không tốn phí:

```bash
docker run -d --name localstack -p 4566:4566 \
  -e SERVICES=s3,iam,sts,ec2,dynamodb localstack/localstack:3

aws --endpoint-url=http://localhost:4566 s3 mb s3://demo
aws --endpoint-url=http://localhost:4566 s3 ls
```

Provider trỏ về LocalStack:

```hcl
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
  s3_use_path_style           = true
  endpoints {
    s3  = "http://localhost:4566"
    iam = "http://localhost:4566"
    sts = "http://localhost:4566"
    ec2 = "http://localhost:4566"
  }
}
```

Mẹo: cài `terraform-local` (`pip install terraform-local`) và dùng lệnh `tflocal init/plan/apply` để tự động trỏ endpoint. Bản community hỗ trợ S3, IAM, STS, DynamoDB, Lambda, SQS; một số dịch vụ như RDS hay EKS cần bản Pro trả phí — nếu lab cần đúng EKS, hãy dùng tài khoản AWS thật và dọn ngay sau khi xong.

---

## Killer.sh — dùng 2 lượt hiệu quả {#killersh}

Khi mua exam CKA/CKS, bạn được **2 lượt Killer.sh simulator** cho chứng chỉ đó. Mỗi lượt có **36 giờ truy cập**, có thể reset môi trường và xem solution của từng câu. Đề Killer.sh khó hơn đề thi thật — điểm thấp ở đây không có nghĩa là bạn sẽ trượt, nhưng nếu làm tốt thì gần như chắc chắn đậu.

| Lượt | Thời điểm | Cách làm | Mục tiêu |
|---|---|---|---|
| 1 | Trước ngày thi 3–4 tuần | Làm như thi thật 2 giờ, không xem solution giữa chừng, sau đó đọc solution **mọi câu** kể cả câu đúng | Chẩn đoán điểm yếu; học cách làm nhanh hơn |
| 2 | Trước ngày thi 1–2 tuần | Full 2 giờ, chỉ dùng kubernetes.io/docs, tự chấm theo thang của simulator | Đạt mức tương đương điểm đậu (66% CKA, 67% CKS) và dư ít nhất 15 phút |

Cách khai thác tối đa:

- Lượt 1 dùng sớm, đừng để sát ngày thi: mục đích là phát hiện lỗ hổng để còn thời gian học lại.
- Đọc solution của cả câu đúng — thường có cách ngắn hơn (imperative command, `--dry-run`, jsonpath) mà bạn chưa dùng.
- Ghi lại mọi lỗi vào một file `loi-thi.md`: sai context, quên namespace, quên label, gõ sai tên object. Trước ngày thi chỉ cần đọc file này.
- Reset môi trường trước khi làm lượt 2 để bắt đầu từ trạng thái sạch.
- Không học thuộc đáp án: đề thi thật khác hoàn toàn về nội dung, chỉ giống về dạng task.
- Đặt báo thức và tắt điện thoại khi làm lượt 2 — luyện cả khả năng tập trung liên tục 2 giờ.

---

## Tài nguyên miễn phí {#free-resources}

| Nguồn | Nội dung | Ghi chú |
|---|---|---|
| [kubernetes.io/docs](https://kubernetes.io/docs/) | Tài liệu chính thức | **Được dùng trong phòng thi** (cùng kubernetes.io/blog và github.com/kubernetes); luyện tìm trang trong dưới 30 giây |
| [github.com/cncf/curriculum](https://github.com/cncf/curriculum) | Curriculum chính thức CKA/CKS | Dùng để tự kiểm tra độ phủ, đối chiếu với mục lục sách |
| [killercoda.com](https://killercoda.com/) | Playground + scenario CKA/CKS miễn phí | Không cần cài gì, môi trường có sẵn; tốt khi máy yếu |
| KodeKloud free labs | Lab Kubernetes cơ bản miễn phí | Đủ cho phần nền tảng chương 1–4 |
| Kubernetes Slack | Kênh `#cka`, `#cks`, `#kubernetes-security` | Hỏi đáp thật, tìm được kinh nghiệm thi mới nhất |
| [editor.networkpolicy.io](https://editor.networkpolicy.io/) | Trực quan hoá NetworkPolicy | Hữu ích khi mới học chương 2 và 6 |
| Tài liệu Calico/Cilium/Falco/Aqua | Docs của từng dự án | Bám đúng version trong sách khi làm lab CKS |

Hai nguyên tắc khi dùng tài nguyên ngoài: (1) ưu tiên tài liệu gốc và bản mới (API Kubernetes thay đổi theo version — bài viết từ 2020 dễ sai), (2) mọi thứ đọc được phải được gõ lại trên cluster, nếu không sẽ quên trước ngày thi.

---

## Checklist môi trường theo chương {#chapter-checklist}

| Chương | Môi trường cần | Công cụ thêm |
|---|---|---|
| 1. Linux & System Hardening | VM Ubuntu hoặc WSL2 | systemd, lynis (tùy chọn) |
| 2. Networking & Cluster Networking | kind + Calico | dig, curl, openssl, tcpdump |
| 3. Container & Docker | Docker Desktop/WSL2 | docker, crictl, trivy |
| 4. Kubernetes căn bản | kind 1 control-plane + 1 worker, ingress-nginx | kubectl, helm |
| 5. Kubernetes vận hành | kind đa node + 2 VM kubeadm | etcdctl, kubeadm, jq |
| 6. Cluster Setup | kind + Calico | kube-bench, openssl |
| 7. Cluster Hardening | kind, 2 VM kubeadm (upgrade) | kubeadm, kube-bench |
| 8. System Hardening | 2 VM kubeadm (AppArmor/seccomp cần host) | aa-status, seccomp, kube-bench |
| 9. Microservice Vulnerabilities | kind (tùy chọn Cilium) | kubectl, jq, cilium CLI (tùy chọn) |
| 10. Supply Chain Security | Docker + kind | trivy, cosign, syft, grype, kube-linter, kubesec |
| 11. Monitoring, Logging & Runtime | kind + audit log, Falco (Helm) | falco, kubectl, jq |
| 12. CI/CD Engineering | Tài khoản GitHub + Docker Hub/GHCR | git, docker, trivy, cosign |
| 13. Cloud & IaC | LocalStack hoặc AWS account có budget alarm | terraform, aws cli |
| 14. Observability | kind + kube-prometheus-stack (Helm) | promtool (tùy chọn) |
| 15. GitOps & Deployment | kind + ArgoCD (Helm) | argocd CLI, helm |
| 16. SRE & Incident Response | kind + stack observability của chương 14 | kubectl, k6 (tùy chọn) |
| 17. Mock Exam & Labs | Cluster kind mới hoàn toàn | kubectl, crictl |
| 18. Phỏng vấn | Không cần cluster | Ghi chú, cheat sheet Phụ lục A |

Kiểm tra nhanh công cụ trước mỗi buổi lab:

```bash
for t in docker kind kubectl helm terraform trivy cosign jq yq; do
  printf '%-10s %s\n' "$t" "$(command -v "$t" >/dev/null 2>&1 && "$t" version 2>/dev/null | head -n1 || echo 'MISSING')"
done
```

Nếu một dòng báo `MISSING`, cài công cụ đó trước khi bắt đầu chương tương ứng. Đừng để đến giữa buổi lab mới phát hiện thiếu `jq` hay `cosign` — mất thời gian và làm gián đoạn mạch học.
