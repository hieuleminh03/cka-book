---
layout: chapter

title: "Phụ lục A"
subtitle: "Cheat sheet"

previous_link: "ch18.html"
previous_title: "Phỏng vấn Middle/Senior DevOps"
next_link: "appB.html"
next_title: "Phụ lục B. Lab & môi trường ôn tập"
---

Phụ lục này gom những lệnh dùng hằng ngày khi làm lab và trong phòng thi, xếp theo nhóm công cụ để tra cứu nhanh. Các lệnh được kiểm chứng với Kubernetes v1.33, kubectl v1.33 và các tool phổ biến năm 2025–2026.

Cách dùng hiệu quả nhất: đừng đọc như tài liệu, hãy gõ lại từng lệnh trong lab cho đến khi thành phản xạ. Quy ước: `<...>` là giá trị bạn phải thay, `#` là comment giải thích. Trong phòng thi bạn được dùng kubernetes.io/docs, nhưng những lệnh dưới đây nên nằm trong trí nhớ để tiết kiệm thời gian.

## Nội dung phụ lục {#appendix-content}

- [kubectl](#kubectl) — context, get/describe/logs/exec, tạo nhanh với dry-run, jsonpath, secret, rollout, debug, api-resources
- [Vim cho YAML](#vim-yaml)
- [Linux vận hành](#linux)
- [Network tooling](#network-tools)
- [Docker, containerd, crictl, nerdctl](#docker-containerd)
- [Helm](#helm)
- [Terraform](#terraform)
- [PromQL](#promql)
- [AWS CLI](#aws-cli)
- [Security tooling](#security-tooling)
- [Kubernetes API quick reference](#k8s-api)

---

## kubectl {#kubectl}

### Context, namespace và cấu hình {#kubectl-context}

```bash
alias k=kubectl
export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'
kubectl config get-contexts
kubectl config use-context kind-cka            # việc ĐẦU TIÊN của mỗi task thi
kubectl config set-context --current --namespace=dev
kubectl config view --minify -o jsonpath='{..namespace}'
```

| Flag | Ý nghĩa |
|---|---|
| `-n <ns>` / `-A` | Chọn namespace / tất cả namespace |
| `-o wide` / `-o yaml` / `-o json` | Thêm cột / manifest đầy đủ |
| `-l app=web` | Lọc theo label |
| `--field-selector status.phase=Running` | Lọc theo field |
| `-w` / `--sort-by=.metadata.creationTimestamp` | Watch / sắp xếp |
| `--show-labels` / `--no-headers` | Hiện label / bỏ dòng tiêu đề |

### get, describe, logs, exec, apply, delete {#kubectl-get-describe}

```bash
kubectl get pods,svc,deploy -n app
kubectl get pods -A -o wide --show-labels
kubectl get pods -l app=web --field-selector status.phase=Running
kubectl get events -A --sort-by=.lastTimestamp | tail -n 20
kubectl describe pod web-0 ; kubectl get pod web-0 -o yaml
kubectl logs -f deploy/web --tail=100 --timestamps
kubectl logs web-0 -c sidecar --previous      # container trước khi restart
kubectl exec -it web-0 -- sh ; kubectl exec web-0 -- env | sort
kubectl apply -f manifests/ -R ; kubectl diff -f manifests/
kubectl delete pods -l app=web -n dev
kubectl delete pod web-0 --force --grace-period=0   # pod kẹt terminating
kubectl replace --force -f pod.yaml           # khi field immutable bị đổi
kubectl patch deploy web --type=merge -p '{"spec":{"replicas":5}}'
kubectl edit deploy web                       # sửa nhanh trong editor
```

### Tạo nhanh tài nguyên (dry-run) {#kubectl-create}

```bash
kubectl run nginx --image=nginx:1.27 --port=80 $do > pod.yaml
kubectl create deployment web --image=nginx:1.27 --replicas=3 $do > deploy.yaml
kubectl expose deployment web --port=80 --target-port=8080 --type=ClusterIP $do > svc.yaml
kubectl create job oneshot --image=busybox:1.37 $do -- date
kubectl create cronjob backup --image=busybox:1.37 --schedule='*/5 * * * *' $do -- date
kubectl create configmap app-cfg --from-literal=ENV=prod --from-file=app.conf $do
kubectl create secret generic db --from-literal=password='S3cet' $do
kubectl create secret tls web-tls --cert=tls.crt --key=tls.key
kubectl create token app-sa -n dev --duration=1h
```

### jsonpath, go-template và lọc kết quả {#kubectl-jsonpath}

```bash
kubectl get pods -n app -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
kubectl get deploy web -o jsonpath='{.spec.template.spec.containers[*].image}'
kubectl get secret db -o jsonpath='{.data.password}' | base64 -d
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP
kubectl get pod web-0 -o go-template='{% raw %}{{ .status.podIP }}{% endraw %}'
kubectl get pods --no-headers | grep -c Running
kubectl get pvc -o jsonpath='{.items[?(@.status.phase!="Bound")].metadata.name}'
```

### Secret và ConfigMap {#kubectl-secret-configmap}

```bash
kubectl patch secret db -p "{\"data\":{\"password\":\"$(printf 'New' | base64)\"}}"
kubectl edit secret db                        # giá trị phải ở dạng base64
kubectl exec web-0 -- printenv | grep -i db   # pod đọc qua biến môi trường
kubectl auth can-i get secrets -n dev --as=system:serviceaccount:dev:app-sa
```

Secret chỉ là base64, không mã hoá; bảo vệ bằng RBAC và encryption at rest (chương 9).### Rollout, scale và node maintenance {#kubectl-rollout}

```bash
kubectl rollout status deploy/web ; kubectl rollout history deploy/web
kubectl rollout undo deploy/web --to-revision=1
kubectl rollout restart deploy/web ; kubectl set image deploy/web nginx=nginx:1.28
kubectl scale deploy/web --replicas=5
kubectl autoscale deploy/web --min=2 --max=10 --cpu-percent=70
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
kubectl cordon node-1 ; kubectl uncordon node-1
kubectl taint node node-1 dedicated=gpu:NoSchedule
kubectl taint node node-1 dedicated=gpu:NoSchedule-   # gỡ taint
```

### Debug với kubectl debug {#kubectl-debug}

```bash
kubectl debug -it web-0 --image=busybox:1.37 --target=web -- sh   # ephemeral container
kubectl debug web-0 --copy-to=web-debug --image=busybox:1.37 -it  # bản copy, không đụng pod gốc
kubectl debug node/worker-1 -it --image=busybox:1.37              # vào host namespace
kubectl run tmp --rm -it --image=busybox:1.37 --restart=Never -- nslookup kubernetes.default
kubectl run tmp --rm -it --image=nicolaka/netshoot --restart=Never -- bash
```

### api-resources, explain và quyền {#kubectl-api-resources}

```bash
kubectl api-resources | grep -i networkpolicy      # tìm apiGroup chính xác
kubectl explain pod.spec.containers.securityContext
kubectl explain networkpolicy.spec.ingress --recursive
kubectl auth can-i create pods --as=system:serviceaccount:dev:app-sa
kubectl auth whoami ; kubectl cluster-info ; kubectl version
```

---

## Vim cho YAML {#vim-yaml}

Trong phòng thi không có GUI và mọi manifest đều sửa bằng vim. Nhóm lệnh cần nhớ:

| Nhóm | Lệnh | Việc cần làm |
|---|---|---|
| Di chuyển | `gg` / `G` / `:42` | Đầu file / cuối file / dòng 42 |
| Di chuyển | `0` / `$` / `w` / `b` | Đầu dòng / cuối dòng / từ sau / từ trước |
| Sửa | `i` / `a` / `o` / `O` | Chèn trước / sau con trỏ / dòng mới dưới / trên |
| Sửa | `cw` / `ciw` / `cc` | Đổi từ / đổi cả từ / đổi cả dòng |
| Sửa | `dd` / `yy` / `p` / `x` / `D` | Xoá dòng / copy dòng / dán / xoá ký tự / xoá tới cuối dòng |
| Hoàn tác | `u` / `Ctrl-r` / `.` | Undo / redo / lặp thao tác vừa rồi |
| Tìm kiếm | `/pattern` / `n` / `N` | Tìm xuôi / kết quả tiếp / kết quả trước |
| Thay thế | `:%s/old/new/g` / `:10,20s/old/new/gc` | Toàn file / theo khoảng dòng |
| Thụt lề | `>>` / `<<` / `gg=G` / `:set et ts=2 sw=2` | Thụt vào / thụt ra / căn lại file / chuẩn YAML 2 spaces |
| Dán code | `:set paste` / `:set nopaste` | Tránh vim phá thụt lề khi dán YAML |
| Hiển thị | `:set nu` / `:set list` / `:noh` | Số dòng / ký tự ẩn / tắt highlight tìm kiếm |
| Lưu thoát | `:w` / `:wq` / `:q!` / `ZZ` | Lưu / lưu và thoát / bỏ thay đổi / thoát |

Ba mẹo thực chiến:

- `:r !kubectl get pod web-0 -o yaml` chèn output lệnh vào file đang mở để tham chiếu.
- `:!kubectl apply -f %` apply file đang mở rồi quay lại vim ngay.
- Gõ `:set paste` trước khi dán YAML từ docs — nếu không manifest dễ lỗi `mapping values are not allowed`.

---

## Linux vận hành {#linux}

### systemd và journalctl {#linux-systemd}

| Lệnh | Ý nghĩa |
|---|---|
| `systemctl status kubelet -l` | Trạng thái service, không cắt log |
| `systemctl start/stop/restart/reload <svc>` | Điều khiển service |
| `systemctl enable --now kubelet` | Tự chạy khi boot và chạy ngay |
| `systemctl daemon-reload` | Nạp lại unit file sau khi sửa |
| `journalctl -u kubelet -f` | Theo log service realtime |
| `journalctl -u kubelet --since "10 min ago" -p err` | Log theo thời gian, mức lỗi |
| `journalctl -b -1 -p warning` / `journalctl -k` | Log lần boot trước / log kernel |

### Process, disk và file {#linux-process-disk}

```bash
ps aux --sort=-%cpu | head ; pgrep -af kubelet ; top -o %MEM
dmesg -T | tail -n 50                          # OOM kill, lỗi kernel
kill -15 <pid> ; kill -9 <pid> ; pkill -f pattern
df -h ; df -i ; du -h --max-depth=1 /var | sort -h | tail
lsblk ; mount | column -t ; cat /etc/fstab
find / -xdev -type f -size +200M 2>/dev/null   # file lớn
find /var/log -name "*.log" -mtime +7          # file cũ hơn 7 ngày
find / -xdev -perm -4000 -type f 2>/dev/null   # SUID
tar -czf backup.tgz /etc/kubernetes ; tar -xzf backup.tgz -C /tmp
```

### Permission và user {#linux-permission}

```bash
ls -l ; ls -ld /etc/kubernetes ; ls -lZ file   # chi tiết + SELinux context
stat file ; id ; whoami ; groups ; umask 027
chmod 600 ~/.kube/config                       # chuẩn bắt buộc cho kubeconfig
chmod u+x script.sh ; chmod -R g-w /etc/kubernetes
chown -R kube:kube /var/lib/etcd
sudo -u etcd-user cmd ; sudo -i
chattr +i /etc/passwd ; lsattr /etc/passwd     # immutable, nhớ gỡ khi nâng cấp
```

---

## Network tooling {#network-tools}

### dig, curl, openssl {#network-dig-curl-openssl}

```bash
dig +short kubernetes.default.svc.cluster.local
dig @10.96.0.10 example.com ; dig -x 10.244.0.5 +short
curl -sS -o /dev/null -w '%{http_code} %{time_total}s\n' https://svc.example.com
curl -v -k https://10.0.0.10:6443/healthz
curl --resolve api.example.com:443:10.0.0.10 https://api.example.com
openssl s_client -connect api.example.com:443 -servername api.example.com -showcerts </dev/null
openssl s_client -connect 10.0.0.10:2379 -CAfile /etc/kubernetes/pki/etcd/ca.crt
openssl x509 -in cert.pem -noout -text | grep -A2 'Validity\|Subject:'
openssl x509 -in cert.pem -noout -enddate
```

Port đáng nhớ: etcd 2379/2380, kube-apiserver 6443, kubelet 10250, kube-proxy 10256, NodePort 30000–32767, CoreDNS 53. Kiểm tra socket: `ss -tulpn`, `lsof -i :6443`, `lsof -p <pid>`.

### tcpdump {#network-tcpdump}

```bash
sudo tcpdump -i any -nn port 6443 -c 20
sudo tcpdump -i eth0 -nn 'host 10.244.1.5 and port 8080'
sudo tcpdump -i any -nn -A 'tcp port 80' | grep -i host
sudo tcpdump -i any -w capture.pcap ; tcpdump -r capture.pcap -nn
```

### nginx và HAProxy {#network-nginx-haproxy}

| Tiêu chí | nginx | HAProxy |
|---|---|---|
| Vai trò chính | Web server + reverse proxy + Ingress controller | Load balancer chuyên dụng |
| Cấu hình | `nginx.conf` theo `server`/`location`, reload mượt | `haproxy.cfg` với `frontend`/`backend`, reload bằng `-sf` |
| Health check | Passive (`max_fails`, `fail_timeout`) | Passive + active (HTTP/TCP check) rất mạnh |
| L7 features | Rewrite, cache, gzip, TLS tốt | ACL linh hoạt, sticky session, rate limit, stats page |
| Trong Kubernetes | `ingress-nginx` phổ biến | `haproxy-ingress`, HAProxy làm LB ngoài cluster |

Validate trước khi reload: `nginx -t` và `haproxy -c -f /etc/haproxy/haproxy.cfg`.

---

## Docker, containerd, crictl, nerdctl {#docker-containerd}

```bash
docker build -t ghcr.io/org/app:1.0.0 .
docker build --no-cache --build-arg VERSION=1.0.0 -f docker/Dockerfile -t app:dev .
docker run --rm -it --entrypoint sh nginx:1.27
docker run -d --name web -p 8080:80 -e ENV=prod -v "$PWD/html:/usr/share/nginx/html" nginx:1.27
docker logs -f web ; docker exec -it web sh
docker inspect web ; docker inspect -f '{% raw %}{{ .State.Pid }}{% endraw %}' web
docker system df ; docker system prune -af --volumes
docker save app:dev | gzip > app.tgz ; docker load < app.tgz
```

Trong cluster hiện đại, kubelet nói chuyện với containerd qua CRI — `docker ps` không thấy container Kubernetes:

| Tác vụ | docker | crictl | nerdctl |
|---|---|---|---|
| Container | `docker ps -a` | `crictl ps -a` | `nerdctl ps -a` |
| Image | `docker images` | `crictl images` | `nerdctl images -a` |
| Log / exec | `docker logs` / `docker exec` | `crictl logs` / `crictl exec` | `nerdctl logs` / `nerdctl exec` |
| Inspect | `docker inspect <id>` | `crictl inspect <id>` | `nerdctl inspect <id>` |
| Pull | `docker pull <img>` | `crictl pull <img>` | `nerdctl pull <img>` |
| Dọn image | `docker image prune -a` | `crictl rmi --prune` | `nerdctl image prune -a` |

```bash
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/containerd/containerd.sock
crictl pods ; crictl ps -a ; crictl images ; crictl stats
crictl inspect <container-id> | jq '.status.exitCode'
ctr -n k8s.io images ls ; ctr -n k8s.io containers ls ; ctr -n k8s.io tasks ls
```

---

## Helm {#helm}

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update ; helm search repo ingress-nginx
helm install web bitnami/nginx -n app --create-namespace -f values.yaml
helm install web ./chart --set image.tag=1.0.0 --atomic --timeout 5m
helm upgrade --install web ./chart -n app -f values.yaml
helm upgrade web ./chart --set image.tag=1.0.1 --wait
helm rollback web 1 -n app ; helm history web ; helm status web
helm uninstall web --keep-history
helm template web ./chart -f values.yaml --debug   # render thử trước khi apply
helm lint ./chart ; helm get values web ; helm get manifest web
```

Thứ tự ưu tiên values (tăng dần): `values.yaml` của chart → các `-f custom.yaml` (file sau ghi đè file trước) → `--set` / `--set-string` / `--set-file`. Template Helm là Go template, ví dụ:

{% raw %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount | default 1 }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```
{% endraw %}

Nếu `helm template` lỗi `nil pointer evaluating interface`, thường do values thiếu key — kiểm tra bằng `helm get values`.

---

## Terraform {#terraform}

### Workflow cơ bản {#terraform-workflow}

```bash
terraform init -upgrade                # tải provider/module
terraform fmt -recursive ; terraform validate
terraform plan -out=tfplan ; terraform apply tfplan
terraform apply -auto-approve          # chỉ dùng trong lab/CI có kiểm soát
terraform plan -destroy ; terraform destroy
terraform plan -target=aws_instance.web -replace=aws_instance.web
terraform plan -var-file=prod.tfvars
TF_LOG=DEBUG terraform apply 2> tf.log # debug provider
```

### State và import {#terraform-state}

```bash
terraform state list ; terraform state show aws_instance.web
terraform state mv aws_instance.web aws_instance.app     # đổi tên resource
terraform state rm aws_instance.web                      # bỏ khỏi state, không xoá thật
terraform output -json ; terraform force-unlock <lock-id>
terraform workspace list ; terraform workspace new staging
terraform import aws_s3_bucket.data my-bucket-name
terraform plan -generate-config-out=generated.tf         # sinh config khi import
```

```hcl
# import block (Terraform 1.5+) — khai báo trong code rồi plan/apply
import {
  to = aws_s3_bucket.data
  id = "my-bucket-name"
}

# backend chuẩn: state trên S3, khoá bằng lockfile (Terraform 1.10+)
terraform {
  backend "s3" {
    bucket       = "my-tf-state-bucket"
    key          = "prod/terraform.tfstate"
    region       = "ap-southeast-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

Ba lưu ý sống còn: state chứa secret nên phải mã hoá và hạn chế truy cập; không sửa state khi còn người khác làm việc (dùng lock); luôn review `plan` trước `apply`, đặc biệt các dòng `destroy` và `forces replacement`.

---

## PromQL {#promql}

```promql
up == 0
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} < 0.1
topk(5, sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{container!=""}[5m])))
increase(kube_pod_container_status_restarts_total[1h]) > 0
kube_pod_status_phase{phase="Pending"} == 1
kube_deployment_spec_replicas - kube_deployment_status_replicas_available != 0
histogram_quantile(0.99, sum by (le) (rate(apiserver_request_duration_seconds_bucket{verb!~"WATCH|CONNECT"}[5m])))
```

Nguyên tắc: dùng `rate(...[5m])` cho counter trước khi tính toán; gộp chiều bằng `by (label)`; giữ label `le` trong `by (le)` khi dùng `histogram_quantile`; dùng `offset 1w` để so cùng giờ tuần trước. Alert rule cơ bản:

{% raw %}
```yaml
groups:
  - name: node.rules
    rules:
      - alert: NodeMemoryHigh
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.9
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.instance }} dùng quá 90% memory"
```
{% endraw %}

---

## AWS CLI {#aws-cli}

```bash
aws configure --profile dev ; export AWS_PROFILE=dev AWS_REGION=ap-southeast-1
aws sts get-caller-identity
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{Id:InstanceId,Type:InstanceType,IP:PublicIpAddress}' --output table
aws ec2 start-instances --instance-ids i-0123456789abcdef0
aws ec2 stop-instances --instance-ids i-0123456789abcdef0
aws s3 ls ; aws s3 mb s3://bucket-demo-123 ; aws s3 rb s3://bucket-demo-123 --force
aws s3 cp file.txt s3://bucket-demo-123/ ; aws s3 sync ./dist s3://bucket-demo-123/ --delete
aws s3 presign s3://bucket-demo-123/file.txt --expires-in 3600
aws iam list-users --query 'Users[].UserName'
aws iam list-attached-role-policies --role-name my-role
aws eks list-clusters ; aws eks describe-cluster --name demo --query 'cluster.status'
aws eks update-kubeconfig --name demo --region ap-southeast-1
aws eks list-nodegroups --cluster-name demo
aws eks describe-nodegroup --cluster-name demo --nodegroup-name ng-1
```

Dọn dẹp theo thứ tự để tránh phát sinh chi phí (xoá phụ thuộc trước):

```bash
aws ec2 terminate-instances --instance-ids i-...       # EC2
aws ec2 release-address --allocation-id eipalloc-...   # Elastic IP rảnh vẫn tính phí
aws ec2 describe-volumes --filters Name=status,Values=available   # EBS mồ côi
aws ec2 describe-nat-gateways --filter Name=state,Values=available
aws elbv2 describe-load-balancers ; aws eks delete-nodegroup --cluster-name demo --nodegroup-name ng-1
aws eks delete-cluster --name demo
```

---

## Security tooling {#security-tooling}

| Tool | Mục đích | Lệnh tiêu biểu |
|---|---|---|
| kube-bench | Đối chiếu CIS Benchmark cho cluster | `kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml` |
| Trivy | CVE image/fs, misconfig IaC, secret, SBOM | `trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 img:tag` |
| cosign | Ký và verify image/artifact | `cosign sign --key cosign.key img@sha256:...` |
| Syft | Sinh SBOM từ image/filesystem | `syft img:tag -o spdx-json > sbom.json` |
| Grype | Scan CVE từ SBOM hoặc image | `grype img:tag --fail-on high` |
| Falco | Phát hiện hành vi bất thường ở runtime | `falco --modern-bpf` hoặc Helm chart `falcosecurity/falco` |
| KubeLinter | Lint manifest theo best practice | `kube-linter lint manifests/` |
| Kubesec | Chấm điểm rủi ro cấu hình pod | `kubesec scan pod.yaml` |
| Kubescape | Scan compliance cluster (NSA, MITRE) | `kubescape scan framework nsa --submit=false` |

```bash
sudo kube-bench run --targets master,node,etcd --json | jq '.Controls[].tests[].results[] | select(.status=="FAIL")'
trivy image --severity HIGH,CRITICAL --ignore-unfixed ghcr.io/org/app:1.0.0
cosign sign ghcr.io/org/app:1.0.0
cosign verify ghcr.io/org/app:1.0.0 --certificate-identity-regexp 'https://github.com/org/repo/.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
syft ghcr.io/org/app:1.0.0 -o cyclonedx-json > sbom.json ; grype sbom:sbom.json --fail-on high
kube-linter lint deploy.yaml ; kubesec scan pod.yaml | jq '.[0].score'
kubescape scan framework nsa --submit=false
```

---

## Kubernetes API quick reference {#k8s-api}

| apiVersion | Kind thường dùng |
|---|---|
| `v1` | Pod, Service, ConfigMap, Secret, ServiceAccount, Namespace, Node, PersistentVolume, PersistentVolumeClaim, ResourceQuota, LimitRange, Endpoints (deprecated) |
| `apps/v1` | Deployment, StatefulSet, DaemonSet, ReplicaSet |
| `batch/v1` | Job, CronJob |
| `networking.k8s.io/v1` | Ingress, IngressClass, NetworkPolicy |
| `rbac.authorization.k8s.io/v1` | Role, ClusterRole, RoleBinding, ClusterRoleBinding |
| `storage.k8s.io/v1` | StorageClass, CSIDriver, VolumeAttachment |
| `autoscaling/v2` | HorizontalPodAutoscaler |
| `policy/v1` | PodDisruptionBudget |
| `admissionregistration.k8s.io/v1` | MutatingWebhookConfiguration, ValidatingWebhookConfiguration, ValidatingAdmissionPolicy |
| `apiextensions.k8s.io/v1` | CustomResourceDefinition |
| `scheduling.k8s.io/v1` | PriorityClass |
| `node.k8s.io/v1` | RuntimeClass |
| `certificates.k8s.io/v1` | CertificateSigningRequest |
| `coordination.k8s.io/v1` | Lease |
| `discovery.k8s.io/v1` | EndpointSlice |
| `events.k8s.io/v1` | Event |
| `flowcontrol.apiserver.k8s.io/v1` | FlowSchema, PriorityLevelConfiguration |
| `apiregistration.k8s.io/v1` | APIService |
| `authentication.k8s.io/v1` | TokenReview |
| `authorization.k8s.io/v1` | SubjectAccessReview, SelfSubjectAccessReview |
| `gateway.networking.k8s.io/v1` | Gateway, GatewayClass, HTTPRoute |
| `snapshot.storage.k8s.io/v1` | VolumeSnapshot, VolumeSnapshotClass |

Hai lệnh cứu cánh khi không nhớ API: `kubectl api-resources | grep <tên>` để tìm `apiVersion` + `kind`, và `kubectl explain <kind>.<field> --recursive` để xem schema. Trong manifest luôn khai báo đủ `apiVersion`, `kind`, `metadata` — thiếu `apiVersion` là lỗi phổ biến nhất khi viết tay.
