---
layout: answer

title: "Chương 17"
subtitle: "Mock Exam & Labs CKA/CKS"
exam_objectives:
  - "Luyện tập dưới áp lực thời gian với bộ đề mô phỏng CKA và CKS."
  - "Kiểm tra kỹ năng theo từng domain của curriculum CNCF."
  - "Nắm mẹo phòng thi performance-based và cách phân bổ thời gian."
---

Lời giải dưới đây trình bày theo kiểu **trang đáp án của đề thi thật**: mục tiêu task, các bước lệnh cụ thể và bước verify. Với task troubleshooting, tôi tách rõ phần **chẩn đoán** (bạn cần chỉ ra nguyên nhân gốc) và phần **khắc phục**; khi tự chấm, hãy cho điểm cả hai phần. Các lệnh đều chạy trong WSL2/Git Bash trên cluster kind đã dựng ở `ch17.md`.

## Lời giải Set A (CKA) {#cka-mock-solutions}

### Task A1 — ServiceAccount chỉ đọc pod {#loi-giai-a1}

**Mục tiêu kiểm tra:** RBAC cơ bản — Role/RoleBinding gắn với ServiceAccount, quyền giới hạn theo namespace, kiểm tra bằng `kubectl auth can-i`.

**Lời giải:**

```bash
kubectl -n team-alpha create serviceaccount ci-bot
kubectl -n team-alpha create role pod-reader --verb=get,list,watch --resource=pods
kubectl -n team-alpha create rolebinding ci-bot-binding \
  --role=pod-reader \
  --serviceaccount=team-alpha:ci-bot
```

**Verify:**

```bash
SA=system:serviceaccount:team-alpha:ci-bot

kubectl auth can-i list pods -n team-alpha --as=$SA        # yes
kubectl auth can-i delete pods -n team-alpha --as=$SA      # no
kubectl auth can-i list pods -n default --as=$SA           # no
kubectl auth can-i --list -n team-alpha --as=$SA           # xem toàn bộ quyền hiệu lực
```

**Ghi chú:** Role chỉ có hiệu lực trong namespace chứa nó; đây là điểm khác biệt cốt lõi với ClusterRole. Nếu `can-i` báo `yes` ở namespace `default`, gần như chắc chắn bạn đã tạo ClusterRoleBinding thay vì RoleBinding — lỗi rất hay gặp trong đề thi.

---

### Task A2 — Backup etcd và kiểm tra snapshot {#loi-giai-a2}

**Mục tiêu kiểm tra:** Snapshot etcd qua mTLS, kiểm tra tính hợp lệ, và nắm quy trình restore trên cluster kubeadm (kind).

**Chẩn đoán/khảo sát:**

```bash
kubectl -n kube-system get pod -l component=etcd -o wide
# etcd-ch17-cka-control-plane   1/1   Running   0   ...
```

**Lời giải:**

```bash
kubectl -n kube-system exec etcd-ch17-cka-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/backup-ch17.db
```

**Verify:**

```bash
# 1. Trạng thái snapshot: revision, total keys, size
kubectl -n kube-system exec etcd-ch17-cka-control-plane -- etcdctl \
  --write-out=table snapshot status /var/lib/etcd/backup-ch17.db

# 2. File tồn tại trên node (và > 0 bytes)
docker exec ch17-cka-control-plane ls -lh /var/lib/etcd/backup-ch17.db
```

**Kịch bản restore (viết sẵn, không chạy trong mock):**

```bash
# 1. Restore ra data dir MỚI (chạy được ngay cả khi etcd đang chạy)
kubectl -n kube-system exec etcd-ch17-cka-control-plane -- etcdctl snapshot restore \
  /var/lib/etcd/backup-ch17.db \
  --data-dir=/var/lib/etcd-restore \
  --name=ch17-cka-control-plane \
  --initial-cluster=ch17-cka-control-plane=https://127.0.0.1:2380 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

# 2. Dừng etcd + apiserver: chuyển manifest ra khỏi /etc/kubernetes/manifests
docker exec ch17-cka-control-plane bash -c \
  'mv /etc/kubernetes/manifests/etcd.yaml /etc/kubernetes/manifests/kube-apiserver.yaml /root/'

# 3. Sửa hostPath volume "etcd-data" trong /root/etcd.yaml:
#    path: /var/lib/etcd  →  path: /var/lib/etcd-restore
docker exec -it ch17-cka-control-plane bash
# vi /root/etcd.yaml ... rồi:
# mv /root/etcd.yaml /root/kube-apiserver.yaml /etc/kubernetes/manifests/

# 4. Đợi control plane lên lại và kiểm tra
kubectl get --raw='/readyz?verbose'
```

**Ghi chú:** Hai cái bẫy kinh điển: (1) đổi `--data-dir` mà không đổi `hostPath` — etcd tạo database mới trong volume cũ, cluster "rỗng" dù restore thành công; (2) restore đè lên `/var/lib/etcd` khi etcd còn chạy — dễ hỏng cả dữ liệu cũ. Ngoài ra, snapshot chứa secret nên phải được mã hóa và lưu ngoài cluster trong môi trường thật.

---

### Task A3 — Kustomize overlay cho production {#loi-giai-a3}

**Mục tiêu kiểm tra:** Kustomize base/overlay — kỹ năng "helm/kustomize" trong domain Cluster Architecture.

**Chuẩn bị thư mục:**

```bash
mkdir -p ~/ch17/cka/kustomize/base ~/ch17/cka/kustomize/overlays/prod
cd ~/ch17/cka/kustomize
```

`base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
```

`base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
```

`overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: apps
namePrefix: prod-
commonLabels:
  env: prod
replicas:
  - name: web
    count: 3
images:
  - name: nginx
    newTag: 1.27-alpine
resources:
  - ../../base
```

**Lời giải:**

```bash
cd ~/ch17/cka/kustomize
kubectl kustomize overlays/prod | head -n 20     # xem trước output
kubectl apply -k overlays/prod
```

**Verify:**

```bash
kubectl -n apps get deploy prod-web
kubectl -n apps get deploy prod-web -o jsonpath='{.spec.replicas}{"\n"}'          # 3
kubectl -n apps get deploy prod-web -o jsonpath='{.metadata.labels.env}{"\n"}'    # prod
kubectl -n apps get pods -l env=prod -o wide
```

**Ghi chú:** `replicas` trong overlay khớp tên **trước** khi áp `namePrefix` (tức `web`), còn kết quả tạo ra là `prod-web` — chi tiết nhỏ nhưng hay làm sai tên. Nếu phải chạy production thật, hãy ghim `newTag` cụ thể (ví dụ `1.27.1`) thay vì tag trôi.

---

### Task A4 — Rollout và rollback {#loi-giai-a4}

**Mục tiêu kiểm tra:** Vòng đời Deployment — cập nhật image, quan sát trạng thái, rollback và đọc `rollout history`.

**Lời giải:**

```bash
# 1. Cập nhật image và đợi xong
kubectl -n apps set image deployment/api api=nginx:1.27.1-alpine
kubectl -n apps rollout status deployment/api --timeout=120s
kubectl -n apps rollout history deployment/api

# 2. Cố ý gây lỗi
kubectl -n apps set image deployment/api api=nginx:1.27.99-alpine
kubectl -n apps get pods -l app=api
# STATUS: ImagePullBackOff / ErrImagePull

# 3. Rollback về revision tốt gần nhất
kubectl -n apps rollout undo deployment/api
kubectl -n apps rollout status deployment/api --timeout=120s
```

**Verify:**

```bash
kubectl -n apps get deploy api -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
# nginx:1.27.1-alpine

kubectl -n apps get pods -l app=api -o wide
kubectl -n apps rollout history deployment/api
```

**Ghi chú:** `rollout undo` không có tham số sẽ quay về revision trước đó; nếu bạn lỡ "lùi" quá xa (ví dụ `--to-revision=1` khi revision 1 là image gốc), hãy kiểm tra image bằng `jsonpath` chứ đừng tin vào tên revision. Đây là dạng task "khóa điểm" — chỉ mất 3–4 phút nếu gõ thành thạo.

### Task A5 — Scheduling trên node có taint {#loi-giai-a5}

**Mục tiêu kiểm tra:** `nodeSelector`/nodeAffinity kết hợp `tolerations`; đọc label và taint của node.

**Chẩn đoán/khảo sát:**

```bash
kubectl get nodes --show-labels | grep ch17-cka-worker2
kubectl describe node ch17-cka-worker2 | grep -A2 Taints
# Taints: dedicated=gpu:NoSchedule
```

**Lời giải:**

```yaml
# ai-worker.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-worker
  namespace: apps
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-worker
  template:
    metadata:
      labels:
        app: ai-worker
    spec:
      nodeSelector:
        disk: ssd
      tolerations:
        - key: dedicated
          operator: Equal
          value: gpu
          effect: NoSchedule
      containers:
        - name: app
          image: nginx:1.27-alpine
```

```bash
kubectl -n apps apply -f ai-worker.yaml
```

**Verify:**

```bash
kubectl -n apps get pods -l app=ai-worker -o wide
# NODE phải là ch17-cka-worker2

kubectl -n apps describe pod -l app=ai-worker | grep -E 'Node:|Tolerations' -A2
```

**Ghi chú:** Chỉ có `nodeSelector` mà thiếu `toleration` thì pod sẽ Pending với thông báo kiểu `1 node(s) had untolerated taint {dedicated: gpu}` — hãy tập thói quen đọc Events thay vì sửa mò. `nodeSelector: disk: ssd` là "điều kiện chọn node", còn `toleration` là "xin phép được ở lại node bị taint"; thiếu một trong hai đều không chạy.

---

### Task A6 — Ingress cho hai host {#loi-giai-a6}

**Mục tiêu kiểm tra:** Ingress class, rule dạng host-based, backend service đúng namespace.

**Lời giải:**

```yaml
# public-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: public
  namespace: apps
spec:
  ingressClassName: nginx
  rules:
    - host: shop.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
    - host: api.shop.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

```bash
kubectl -n apps apply -f public-ingress.yaml
```

**Verify:**

```bash
kubectl -n apps get ingress public
# ADDRESS hiển thị IP của controller sau vài giây

curl -s -o /dev/null -w "shop.local: %{http_code}\n" -H "Host: shop.local" http://localhost:8080/
# shop.local: 200
curl -s -o /dev/null -w "api.shop.local: %{http_code}\n" -H "Host: api.shop.local" http://localhost:8080/
# api.shop.local: 200

# Soi nginx sinh cấu hình gì cho Ingress này
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller --tail=20
```

**Ghi chú:** Thiếu `ingressClassName: nginx` thì Ingress được tạo nhưng controller bỏ qua — `ADDRESS` rỗng và request trả 404; đây là lỗi kinh điển khi cluster có nhiều IngressClass. Ngoài ra backend service chỉ resolve được trong **cùng namespace** với Ingress; đừng cố trỏ sang service ở namespace khác.

---

### Task A7 — NetworkPolicy frontend đến backend {#loi-giai-a7}

**Mục tiêu kiểm tra:** Namespace policy (`podSelector` + `namespaceSelector` trong cùng một `from` = AND), kèm egress DNS.

**Lời giải:**

```yaml
# backend-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: frontend
          podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 80
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

```bash
kubectl -n backend apply -f backend-netpol.yaml
```

**Verify:**

```bash
# 1. frontend -> backend: OK (curl trong netshoot)
kubectl -n frontend exec deploy/frontend -- \
  curl -sS --max-time 3 -o /dev/null -w "frontend->backend: %{http_code}\n" \
  http://backend.backend.svc.cluster.local

# 2. namespace khác -> backend: bị chặn
kubectl -n apps run probe --image=nicolaka/netshoot:v0.13 --restart=Never --command -- sleep 1d
kubectl -n apps wait --for=condition=Ready pod/probe --timeout=60s
kubectl -n apps exec probe -- \
  curl -sS --max-time 3 http://backend.backend.svc.cluster.local -o /dev/null -w "apps->backend: %{http_code}\n"
# curl: (28) Connection timed out  → đúng, bị chặn

# 3. DNS trong backend: OK (busybox nslookup có sẵn trong image nginx alpine)
kubectl -n backend exec deploy/backend -- nslookup kubernetes.default.svc.cluster.local

kubectl -n apps delete pod probe
```

**Ghi chú:** `namespaceSelector` và `podSelector` nằm trong **cùng một phần tử** của `from` là điều kiện AND (namespace frontend **và** label app=frontend). Nếu tách thành hai phần tử list, cả hai nguồn đều được phép — đây là bẫy ngữ nghĩa rất hay hỏi. Phần egress DNS là bắt buộc vì `policyTypes` có `Egress`: nếu thiếu, mọi kết nối DNS từ backend đều chết, kể cả tra tên service.

---

### Task A8 — PVC đang Pending {#loi-giai-a8}

**Mục tiêu kiểm tra:** StorageClass, tính immutable của PVC, vòng đời dữ liệu với persistent volume.

**Chẩn đoán:**

```bash
kubectl -n storage-a get pvc
kubectl -n storage-a describe pvc data | tail -n 15
# Events: storageclass "standard-rwo" not found
kubectl get sc
# standard (default)   rancher.io/local-path
```

**Lời giải (PVC immutable — phải xoá và tạo lại):**

```bash
kubectl -n storage-a delete pvc data

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
  namespace: storage-a
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
EOF
# Bỏ storageClassName để dùng default StorageClass ("standard")
```

**Pod ghi dữ liệu:**

```yaml
# writer.yaml
apiVersion: v1
kind: Pod
metadata:
  name: writer
  namespace: storage-a
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data
```

```bash
kubectl -n storage-a apply -f writer.yaml
kubectl -n storage-a exec writer -- sh -c 'echo ch17 > /data/hello.txt'
kubectl -n storage-a exec writer -- cat /data/hello.txt
```

**Verify dữ liệu bền vững:**

```bash
kubectl -n storage-a delete pod writer
kubectl -n storage-a apply -f writer.yaml
kubectl -n storage-a wait --for=condition=Ready pod/writer --timeout=60s
kubectl -n storage-a exec writer -- cat /data/hello.txt
# ch17  → dữ liệu vẫn còn
kubectl -n storage-a get pvc data
# STATUS: Bound
```

**Ghi chú:** Bạn **không thể** sửa `storageClassName` của PVC đã tạo (immutable); cách duy nhất là xoá/tạo lại khi chưa có dữ liệu quan trọng, hoặc tạo PVC mới. Trên kind, StorageClass mặc định tên là `standard` (local-path-provisioner); trên managed cloud thì tên thường là `gp2`/`gp3`/`standard-rwo` tùy nền tảng — đừng bê nguyên tên giữa các cluster.

### Task A9 — Node NotReady {#loi-giai-a9}

**Mục tiêu kiểm tra:** Troubleshooting node — đọc trạng thái, tìm agent lỗi, sửa và chứng minh node nhận workload trở lại.

**Chẩn đoán:**

```bash
kubectl get nodes
# ch17-cka-worker    NotReady  ...

kubectl describe node ch17-cka-worker | tail -n 20
# Conditions: Ready=False, "kubelet stopped posting node status"

# Kiểm tra agent trên node (kind node là container — dùng docker exec)
docker exec ch17-cka-worker systemctl status kubelet --no-pager
# Active: inactive (dead)
docker exec ch17-cka-worker journalctl -u kubelet -n 30 --no-pager
```

**Lời giải:**

```bash
docker exec ch17-cka-worker systemctl start kubelet
docker exec ch17-cka-worker systemctl is-active kubelet     # active
```

**Verify:**

```bash
kubectl get nodes -w
# ch17-cka-worker   Ready   <40s sau khi kubelet lên lại

# Chứng minh node nhận được workload mới
kubectl run a9-test --image=nginx:1.27-alpine \
  --overrides='{"spec":{"nodeName":"ch17-cka-worker"}}'
kubectl wait --for=condition=Ready pod/a9-test --timeout=90s
kubectl get pod a9-test -o wide
kubectl delete pod a9-test
```

**Ghi chú:** Trong đề thi thật, nguyên nhân `NotReady` thường là: kubelet chết, sai config trong `/var/lib/kubelet/config.yaml`, certificate hết hạn, container runtime down, hoặc disk pressure. Quy trình chẩn đoán nên theo thứ tự: `describe node` → `systemctl status kubelet` → `journalctl -u kubelet` → kiểm tra cert/runtime. Lưu ý: node `NotReady` vẫn giữ nguyên pod đang chạy; drain chỉ cần khi bạn chuẩn bị bảo trì node.

---

### Task A10 — Pod Pending vì requests quá lớn {#loi-giai-a10}

**Mục tiêu kiểm tra:** Đọc Events, hiểu scheduler tôn trọng `resources.requests`, dùng `kubectl set resources`.

**Chẩn đoán:**

```bash
kubectl -n payments get deploy payments -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
# {"requests":{"cpu":"32","memory":"128Gi"}}

kubectl -n payments describe pod -l app=payments | grep -A5 Events
# 0/3 nodes are available: 3 Insufficient cpu, 3 Insufficient memory.
# preemption: 0/3 nodes are available: ...
```

**Lời giải:**

```bash
kubectl -n payments set resources deployment/payments \
  --requests=cpu=100m,memory=128Mi

kubectl -n payments rollout status deployment/payments --timeout=120s
```

**Verify:**

```bash
kubectl -n payments get pods -o wide
# 2 pod Running
kubectl -n payments get deploy payments -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
# {"requests":{"cpu":"100m","memory":"128Mi"}}
```

**Ghi chú:** Scheduler chỉ nhìn vào `requests` (không nhìn `limits`) khi xếp pod; `limits` chỉ ảnh hưởng cgroup lúc chạy. Vì vậy đặt `requests` bằng giá trị của cả node là cách tự chặn pod hiệu quả nhất — và cũng là "bẫy" phổ biến khi dev copy cấu hình từ máy khỏe. Với pod đã Pending, `kubectl rollout restart` không cần thiết vì `set resources` đã tạo ReplicaSet mới.

---

### Task A11 — DNS bị NetworkPolicy chặn {#loi-giai-a11}

**Mục tiêu kiểm tra:** Vì sao DNS chết khi bật egress policy; `namespaceSelector` phải dùng label nào; cách mở đúng phạm vi.

**Chẩn đoán:**

```bash
kubectl -n search get networkpolicy
kubectl -n search get networkpolicy lockdown-egress -o yaml
# namespaceSelector.matchLabels: name: kube-system   ← label này KHÔNG tồn tại
kubectl get ns kube-system --show-labels
# kubernetes.io/metadata.name=kube-system   ← đây mới là label thật

kubectl -n search exec deploy/client -- nslookup api.search.svc.cluster.local
# ;; connection timed out; no servers could be reached
```

**Lời giải:**

```yaml
# lockdown-egress.yaml (bản đã sửa)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: lockdown-egress
  namespace: search
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    # 1. Cho phép traffic nội bộ namespace (client -> api)
    - to:
        - podSelector: {}
    # 2. Cho phép DNS tới kube-dns
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

```bash
kubectl -n search apply -f lockdown-egress.yaml
```

**Verify:**

```bash
kubectl -n search exec deploy/client -- nslookup api.search.svc.cluster.local
# Address: 10.x.x.x  → DNS OK

kubectl -n search exec deploy/client -- \
  curl -sS --max-time 3 -o /dev/null -w "client->api: %{http_code}\n" \
  http://api.search.svc.cluster.local
# client->api: 200

# Chứng minh egress ra Internet vẫn bị chặn (đúng yêu cầu "không mở thêm")
kubectl -n search exec deploy/client -- \
  curl -sS --max-time 3 https://example.com -o /dev/null -w "internet: %{http_code}\n" || echo "internet: bị chặn (đúng)"
```

**Ghi chú:** Mọi namespace từ Kubernetes v1.21+ đều tự động có label `kubernetes.io/metadata.name=<tên-ns>`, nhưng rất nhiều người viết policy theo thói quen `name: kube-system` (label này chỉ đúng nếu ai đó tự gán). Khi egress bị default-deny, **DNS là nạn nhân đầu tiên** — luôn kiểm tra policy egress trước khi nghi ngờ CoreDNS.

---

### Task A12 — Service không có endpoint {#loi-giai-a12}

**Mục tiêu kiểm tra:** Chẩn đoán endpoint rỗng: sai namespace, sai selector, sai targetPort; dọn workload rác.

**Chẩn đoán:**

```bash
kubectl -n catalog get endpoints catalog-svc
# NAME          ENDPOINTS   AGE
# catalog-svc   <none>      ...

kubectl -n catalog describe svc catalog-svc | tail -n 10
kubectl -n catalog get pods --show-labels
# Không có pod app=catalog trong namespace catalog

kubectl get deploy -A | grep catalog
# default   catalog   1/1   ...        ← workload lạc namespace!

kubectl -n catalog get svc catalog-svc -o jsonpath='{.spec.ports}{"\n"}'
# [{"port":80,"protocol":"TCP","targetPort":8080}]   ← container nginx nghe 80
```

**Lời giải:**

```bash
# 1. Đưa workload về đúng namespace với 2 replicas
kubectl -n catalog create deployment catalog --image=nginx:1.27-alpine --replicas=2

# 2. Sửa targetPort về 80 (port là merge key của strategic merge patch)
kubectl -n catalog patch svc catalog-svc \
  -p '{"spec":{"ports":[{"port":80,"targetPort":80}]}}'

# 3. Dọn workload lạc namespace
kubectl -n default delete deployment catalog
```

**Verify:**

```bash
kubectl -n catalog get endpoints catalog-svc
# ENDPOINTS: 10.244.x.x:80,10.244.x.y:80

kubectl -n catalog run probe --image=nicolaka/netshoot:v0.13 --restart=Never --command -- sleep 1d
kubectl -n catalog wait --for=condition=Ready pod/probe --timeout=60s
kubectl -n catalog exec probe -- curl -s -o /dev/null -w "catalog-svc: %{http_code}\n" http://catalog-svc
# catalog-svc: 200
kubectl -n catalog delete pod probe

kubectl -n default get deploy      # No resources found
```

**Ghi chú:** Endpoint rỗng có đúng 3 nguyên nhân phổ biến: không có pod khớp selector, pod chưa Ready, hoặc pod nằm sai namespace (namespace không nằm trong selector của service). Sau khi có endpoint mà vẫn lỗi kết nối, nguyên nhân tiếp theo là **targetPort sai** — phân biệt rõ `port` (cổng service), `targetPort` (cổng container) và `containerPort` (chỉ là thông tin khai báo).

## Lời giải Set B (CKS) {#cks-mock-solutions}

### Task B1 — Default-deny, DNS và web đến api {#loi-giai-b1}

**Mục tiêu kiểm tra:** NetworkPolicy nhiều lớp: default-deny hai chiều, mở DNS, mở đúng luồng web→api **cả ingress lẫn egress**.

**Lời giải:**

```yaml
# netpol-b.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: netpol-b
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: netpol-b
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
  namespace: netpol-b
spec:
  # Chiều egress: web được đi ra tới api
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - protocol: TCP
          port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-from-web
  namespace: netpol-b
spec:
  # Chiều ingress: api chỉ nhận từ web
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web
      ports:
        - protocol: TCP
          port: 80
```

```bash
kubectl -n netpol-b apply -f netpol-b.yaml
```

**Verify (4 kịch bản):**

```bash
# 1. web -> api: OK (busybox wget trong nginx alpine)
kubectl -n netpol-b exec deploy/web -- wget -T 3 -qO- http://api.netpol-b.svc.cluster.local >/dev/null \
  && echo "web->api: OK" || echo "web->api: FAIL"

# 2. web -> web: bị chặn
kubectl -n netpol-b exec deploy/web -- wget -T 3 -qO- http://web.netpol-b.svc.cluster.local \
  && echo "web->web: OK" || echo "web->web: bị chặn (đúng)"

# 3. pod mới -> api: bị chặn (cả egress của pod mới lẫn ingress của api)
kubectl -n netpol-b run probe --image=nicolaka/netshoot:v0.13 --restart=Never --command -- sleep 1d
kubectl -n netpol-b wait --for=condition=Ready pod/probe --timeout=60s
kubectl -n netpol-b exec probe -- curl -sS --max-time 3 http://api.netpol-b.svc.cluster.local \
  && echo "probe->api: OK" || echo "probe->api: bị chặn (đúng)"

# 4. DNS: OK
kubectl -n netpol-b exec deploy/web -- nslookup api.netpol-b.svc.cluster.local | tail -n 3
kubectl -n netpol-b delete pod probe
```

**Ghi chú:** Bẫy lớn nhất của bài này là quên **egress của web**: default-deny egress đã chặn web đi ra, nên chỉ mở ingress cho api là chưa đủ — cần thêm policy egress cho web (hoặc một policy `podSelector: {app: web}` với egress tới api). Ngoài ra, "default-deny cả hai chiều" là hai `policyTypes` trong cùng một policy; nếu chỉ ghi `Ingress` thì egress vẫn tự do và bạn sẽ trượt cả 4 tiêu chí verify.

---

### Task B2 — kube-bench và remediation {#loi-giai-b2}

**Mục tiêu kiểm tra:** CIS Benchmark cho worker: chạy kube-bench, đọc output, sửa cấu hình kubelet trên node và chứng minh bằng re-scan.

**Lời giải — chạy kube-bench trên worker:**

```yaml
# kube-bench-node.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench-node
spec:
  template:
    spec:
      hostPID: true
      nodeSelector:
        kubernetes.io/hostname: ch17-cks-worker
      tolerations:
        - operator: Exists
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:latest
          command: ["kube-bench", "run", "--targets", "node", "--json"]
          volumeMounts:
            - name: etc-kubernetes
              mountPath: /etc/kubernetes
              readOnly: true
            - name: var-lib-kubelet
              mountPath: /var/lib/kubelet
              readOnly: true
      restartPolicy: Never
      volumes:
        - name: etc-kubernetes
          hostPath:
            path: /etc/kubernetes
        - name: var-lib-kubelet
          hostPath:
            path: /var/lib/kubelet
```

```bash
kubectl apply -f kube-bench-node.yaml
kubectl wait --for=condition=complete job/kube-bench-node --timeout=240s

# Liệt kê các check FAIL
kubectl logs job/kube-bench-node | jq -r \
  '.Tests[]?.Results[]? | select(.status=="FAIL") | .test_number + " " + .test_desc'
```

**Remediation (ví dụ các check hay gặp trên kind):**

```bash
# Ví dụ 1 — check 4.1.9: quyền file cấu hình kubelet phải là 600
docker exec ch17-cks-worker stat -c '%a %n' /var/lib/kubelet/config.yaml
docker exec ch17-cks-worker chmod 600 /var/lib/kubelet/config.yaml

# Ví dụ 2 — nhóm 4.2: siết kubelet qua kubelet config (không dùng flag dòng lệnh)
docker exec -it ch17-cks-worker vi /var/lib/kubelet/config.yaml
```

Các giá trị cần có trong `/var/lib/kubelet/config.yaml` (thêm nếu thiếu):

```yaml
readOnlyPort: 0
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
```

```bash
docker exec ch17-cks-worker systemctl restart kubelet
kubectl wait --for=condition=Ready node/ch17-cks-worker --timeout=120s
```

**Verify (re-scan):**

```bash
kubectl delete job kube-bench-node
kubectl apply -f kube-bench-node.yaml
kubectl wait --for=condition=complete job/kube-bench-node --timeout=240s

# Đổi 4.1.9 thành test_number bạn đã sửa
kubectl logs job/kube-bench-node | jq -r \
  '.Tests[]?.Results[]? | select(.test_number=="4.1.9") | .status'
# PASS
```

**Ghi chú:** kube-bench chấm theo CIS benchmark cho phiên bản K8s nó phát hiện; output JSON có `Tests[].Results[]` với `status` là `PASS`/`FAIL`/`WARN`. Trong đề thi, bạn không cần "sạch" mọi check — chỉ cần sửa đúng check được yêu cầu và **chứng minh trước/sau**. Nhớ rằng sửa `/var/lib/kubelet/config.yaml` cần restart kubelet, còn sửa quyền file thì không. Nếu image kube-bench của bạn không nhận subcommand `run`, kiểm tra `kube-bench run --help` hoặc dùng dạng `kube-bench --targets node --json` cho phù hợp version.

---

### Task B3 — RBAC least privilege {#loi-giai-b3}

**Mục tiêu kiểm tra:** Role với `resourceNames`, giới hạn verb, và kiểm tra bằng `kubectl auth can-i` theo từng ngữ cảnh.

**Lời giải:**

```yaml
# reporter-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: reporter
  namespace: secure-rbac
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: reporter
  namespace: secure-rbac
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["app-config"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: reporter-binding
  namespace: secure-rbac
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: reporter
subjects:
  - kind: ServiceAccount
    name: reporter
    namespace: secure-rbac
```

```bash
kubectl apply -f reporter-rbac.yaml
```

**Verify:**

```bash
SA=system:serviceaccount:secure-rbac:reporter

kubectl auth can-i list pods -n secure-rbac --as=$SA         # yes
kubectl auth can-i get configmaps/app-config -n secure-rbac --as=$SA   # yes
kubectl auth can-i get configmaps/other-config -n secure-rbac --as=$SA # no
kubectl auth can-i delete pods -n secure-rbac --as=$SA       # no
kubectl auth can-i list pods -n default --as=$SA             # no
kubectl auth can-i --list -n secure-rbac --as=$SA
```

**Ghi chú:** `resourceNames` chỉ áp dụng cho verb không phải `list`/`watch` (bạn không thể giới hạn theo tên khi list cả collection). Cũng lưu ý `get configmaps/app-config` trong `can-i` chính là dạng resource/name. Đây là pattern "least privilege" chuẩn cho service account chỉ cần đọc đúng một cấu hình.

---

### Task B4 — Tắt automount service account token {#loi-giai-b4}

**Mục tiêu kiểm tra:** Token hygiene — giảm bề mặt: pod không tự động nhận API credential.

**Lời giải:**

```bash
# 1. Tắt automount ở default ServiceAccount (áp dụng cho pod tương lai của namespace)
kubectl -n secure-rbac patch serviceaccount default \
  -p '{"automountServiceAccountToken": false}'
```

```yaml
# no-token.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: no-token
  namespace: secure-rbac
spec:
  replicas: 1
  selector:
    matchLabels:
      app: no-token
  template:
    metadata:
      labels:
        app: no-token
    spec:
      serviceAccountName: reporter
      automountServiceAccountToken: false
      containers:
        - name: app
          image: nginx:1.27-alpine
```

```bash
kubectl -n secure-rbac apply -f no-token.yaml
kubectl -n secure-rbac rollout status deployment/no-token --timeout=120s
```

**Verify:**

```bash
kubectl -n secure-rbac get sa default -o jsonpath='{.automountServiceAccountToken}{"\n"}'
# false

kubectl -n secure-rbac exec deploy/no-token -- ls /var/run/secrets/kubernetes.io/serviceaccount
# ls: /var/run/secrets/kubernetes.io/serviceaccount: No such file or directory

kubectl -n secure-rbac get pod -l app=no-token -o jsonpath='{.items[0].spec.automountServiceAccountToken}{"\n"}'
# false
```

**Ghi chú:** `automountServiceAccountToken` được kubelet đọc **lúc tạo pod**; bạn không thể thêm/bớt volume token của pod đang chạy — phải tạo pod mới (xoá pod cũ, để controller tạo lại). Thứ tự ưu tiên: giá trị trên Pod spec thắng giá trị trên ServiceAccount. Pod nào không cần gọi API server (web server, worker xử lý file...) nên tắt hẳn để giảm bề mặt tấn công.

### Task B5 — seccomp Localhost và RuntimeDefault {#loi-giai-b5}

**Mục tiêu kiểm tra:** Áp seccomp profile ở hai dạng: profile tùy chỉnh trên node (Localhost) và profile mặc định của runtime.

**Lời giải:**

```bash
# 1. Tạo profile audit: log mọi syscall nhưng không chặn (chế độ quan sát)
cat > audit.json <<'EOF'
{
  "defaultAction": "SCMP_ACT_LOG"
}
EOF

# 2. Đưa profile lên node worker2 (kind node là container — dùng docker cp)
docker exec ch17-cks-worker2 mkdir -p /var/lib/kubelet/seccomp/profiles
docker cp audit.json ch17-cks-worker2:/var/lib/kubelet/seccomp/profiles/audit.json
docker exec ch17-cks-worker2 ls -l /var/lib/kubelet/seccomp/profiles/audit.json
```

```yaml
# seccomp-pods.yaml
apiVersion: v1
kind: Pod
metadata:
  name: audited
  namespace: hardening
spec:
  nodeSelector:
    kubernetes.io/hostname: ch17-cks-worker2
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
    - name: app
      image: nginx:1.27-alpine
---
apiVersion: v1
kind: Pod
metadata:
  name: default-seccomp
  namespace: hardening
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginx:1.27-alpine
```

```bash
kubectl apply -f seccomp-pods.yaml
kubectl -n hardening wait --for=condition=Ready pod/audited pod/default-seccomp --timeout=120s
```

**Verify:**

```bash
# Seccomp mode 2 = filter đang bật; 0 = unconfined
kubectl -n hardening exec audited -- grep -E '^Seccomp' /proc/1/status
# Seccomp: 2
kubectl -n hardening exec default-seccomp -- grep -E '^Seccomp' /proc/1/status
# Seccomp: 2

kubectl -n hardening get pod audited -o jsonpath='{.spec.securityContext.seccompProfile.type}{"\n"}'
# Localhost
kubectl -n hardening get pod audited -o jsonpath='{.spec.securityContext.seccompProfile.localhostProfile}{"\n"}'
# profiles/audit.json
```

**Ghi chú:** `localhostProfile` là đường dẫn **tương đối** so với thư mục seccomp của kubelet (`/var/lib/kubelet/seccomp`), không phải đường dẫn tuyệt đối — viết `/var/lib/kubelet/seccomp/profiles/audit.json` sẽ bị từ chối hoặc lỗi `cannot load seccomp profile`. Vì profile nằm trên **một node cụ thể**, pod Localhost phải được ghim vào node đó (hoặc bạn phải phân phối profile lên mọi node). Để có profile thật, hãy chạy ứng dụng ở chế độ audit, đọc syscall thực tế, rồi chuyển dần sang `SCMP_ACT_ERRNO` cho các syscall không dùng.

---

### Task B6 — Hardening workload {#loi-giai-b6}

**Mục tiêu kiểm tra:** `securityContext` toàn diện và hệ quả của `readOnlyRootFilesystem` (cần volume ghi được).

**Chẩn đoán:**

```bash
kubectl -n hardening get deploy web -o yaml | grep -A8 securityContext
# runAsUser: 0, privileged: true, capabilities add NET_ADMIN ...
```

**Lời giải — ghi đè toàn bộ spec bằng manifest đã hardening:**

```yaml
# hardening-web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: hardening
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        runAsGroup: 101
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: web
          image: nginxinc/nginx-unprivileged:1.27-alpine
          ports:
            - containerPort: 8080
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: cache
              mountPath: /var/cache/nginx
            - name: run
              mountPath: /var/run
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: cache
          emptyDir: {}
        - name: run
          emptyDir: {}
        - name: tmp
          emptyDir: {}
```

```bash
kubectl -n hardening apply -f hardening-web.yaml
kubectl -n hardening rollout status deployment/web --timeout=120s
```

**Verify:**

```bash
kubectl -n hardening exec deploy/web -- id
# uid=101 gid=101

kubectl -n hardening exec deploy/web -- sh -c 'touch /khong-the-ghi'
# touch: /khong-the-ghi: Read-only file system

kubectl -n hardening exec deploy/web -- wget -qO- http://localhost:8080 | head -n 3
# <!DOCTYPE html> ...  (nginx vẫn phục vụ)

kubectl -n hardening exec deploy/web -- grep '^Seccomp' /proc/1/status
# Seccomp: 2
```

**Ghi chú:** Image `nginx-unprivileged` lắng nghe cổng 8080 và chạy UID 101 sẵn — nếu giữ image `nginx` thường với `runAsNonRoot: true`, container sẽ fail vì không bind được cổng 80 dưới quyền non-root. Khi bật `readOnlyRootFilesystem`, phải xác định mọi path ứng dụng cần ghi (`/var/cache/nginx`, `/var/run`, `/tmp`) và mount `emptyDir` cho chúng; nếu không, lỗi trông giống "nginx crash bí ẩn".

---

### Task B7 — Pod Security Standards restricted {#loi-giai-b7}

**Mục tiêu kiểm tra:** Pod Security Admission — gán nhãn namespace, đọc thông báo vi phạm, sửa workload cho hợp lệ.

**Lời giải — bật enforce:**

```bash
kubectl label namespace pss-lab \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
kubectl get ns pss-lab --show-labels
```

**Chẩn đoán vi phạm (từ ReplicaSet Events):**

```bash
kubectl -n pss-lab get rs
kubectl -n pss-lab describe rs -l app=legacy | grep -A5 Events
# Warning  FailedCreate  ... pods "legacy-..." is forbidden:
#   violates PodSecurity "restricted:latest":
#   privileged (container "app" must not be privileged),
#   allowPrivilegeEscalation != false, unrestricted capabilities,
#   hostPath volumes ...
```

**Sửa Deployment (được phép đổi image sang bản unprivileged):**

```yaml
# legacy-restricted.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy
  namespace: pss-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: legacy
  template:
    metadata:
      labels:
        app: legacy
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        runAsGroup: 101
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: nginxinc/nginx-unprivileged:1.27-alpine
          ports:
            - containerPort: 8080
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
```

```bash
kubectl -n pss-lab apply -f legacy-restricted.yaml
kubectl -n pss-lab rollout status deployment/legacy --timeout=120s
```

**Verify:**

```bash
kubectl -n pss-lab get pods -l app=legacy
# 1/1 Running

kubectl -n pss-lab exec deploy/legacy -- id
# uid=101 gid=101

# Chứng minh enforce: pod privileged bị chặn ngay ở admission
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bad-pss
  namespace: pss-lab
spec:
  containers:
    - name: bad
      image: nginx:1.27-alpine
      securityContext:
        privileged: true
EOF
# Error from server (Forbidden): error when creating ...:
# pods "bad-pss" is forbidden: violates PodSecurity "restricted:latest":
# privileged (container "bad" must not be privileged), ...
```

**Ghi chú:** PSS `restricted` yêu cầu: không privileged, `allowPrivilegeEscalation: false`, drop ALL capabilities, `runAsNonRoot: true`, seccomp `RuntimeDefault`/`Localhost`, và chỉ dùng volume thuộc allowlist (không có `hostPath`). Vì vậy phải **bỏ volume hostPath** chứ không chỉ đổi securityContext. Thứ tự làm gọn nhất: sửa manifest đầy đủ rồi `apply` thay vì patch từng field — pod cũ đang chạy không bị kill khi gán nhãn, nhưng mọi pod mới sẽ bị admission chặn.

---

### Task B8 — Encryption at rest cho secrets {#loi-giai-b8}

**Mục tiêu kiểm tra:** EncryptionConfiguration + sửa static pod manifest kube-apiserver, kiểm chứng qua etcd.

**Bước 1 — chứng minh chưa mã hóa:**

```bash
ETCD_POD=etcd-ch17-cks-control-plane
etcd_get() {
  kubectl -n kube-system exec "$ETCD_POD" -- etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    get "$1"
}

etcd_get /registry/secrets/secure-app/db-creds | head -c 200; echo
# k8s:enc:... KHÔNG có — giá trị password nằm plaintext (base64)
```

**Bước 2 — sinh khóa và tạo config:**

```bash
cd ~/ch17-cks
ENC_KEY=$(openssl rand -base64 32 | tr -d '\n')
cat > encryption-config.yaml <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENC_KEY}
      - identity: {}
EOF

docker exec ch17-cks-control-plane mkdir -p /etc/kubernetes/enc
docker cp encryption-config.yaml ch17-cks-control-plane:/etc/kubernetes/enc/encryption-config.yaml
docker exec ch17-cks-control-plane chmod 600 /etc/kubernetes/enc/encryption-config.yaml
```

**Bước 3 — sửa manifest kube-apiserver:**

```bash
docker cp ch17-cks-control-plane:/etc/kubernetes/manifests/kube-apiserver.yaml .
cp kube-apiserver.yaml kube-apiserver.yaml.bak      # luôn giữ bản backup
```

Thêm vào `command:` của container `kube-apiserver`:

```yaml
    - --encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
```

Thêm vào `volumeMounts:` của container:

```yaml
    - mountPath: /etc/kubernetes/enc
      name: enc
      readOnly: true
```

Thêm vào `volumes:` của pod:

```yaml
  - hostPath:
      path: /etc/kubernetes/enc
      type: DirectoryOrCreate
    name: enc
```

```bash
docker cp kube-apiserver.yaml ch17-cks-control-plane:/etc/kubernetes/manifests/kube-apiserver.yaml

# Kubelet tự restart static pod; đợi control plane khỏe lại
kubectl get --raw='/readyz?verbose'
kubectl -n kube-system get pod -l component=kube-apiserver
```

**Verify:**

```bash
# Secret tạo mới đã được mã hóa
kubectl -n secure-app create secret generic new-secret --from-literal=token='TopSecret123'
etcd_get /registry/secrets/secure-app/new-secret | head -c 120; echo
# k8s:enc:aescbc:v1:key1: ...

# Secret cũ vẫn đọc bình thường nhờ provider identity
kubectl -n secure-app get secret db-creds -o jsonpath='{.data.password}' | base64 -d; echo
# S3cretValue!

# Re-encrypt toàn bộ secret cũ
kubectl get secrets -A -o json | kubectl replace -f -
etcd_get /registry/secrets/secure-app/db-creds | head -c 120; echo
# k8s:enc:aescbc:v1:key1: ...
```

**Rollback nếu apiserver không lên:**

```bash
docker cp kube-apiserver.yaml.bak ch17-cks-control-plane:/etc/kubernetes/manifests/kube-apiserver.yaml
kubectl get --raw='/readyz?verbose'
```

**Ghi chú:** `identity` là provider "không mã hóa" nằm **sau** `aescbc`; nó cho phép đọc dữ liệu cũ chưa mã hóa trong lúc migration, và có thể bỏ đi sau khi re-encrypt xong. Trên cluster nhiều control-plane, phải copy **cùng một file/cùng khóa** lên mọi node và cập nhật lần lượt. Với production, thay `aescbc` bằng provider `kms` và bật `--encryption-provider-config-automatic-reload=true` để xoay khóa không cần restart.

### Task B9 — Image nhỏ, ít CVE {#loi-giai-b9}

**Mục tiêu kiểm tra:** Giảm bề mặt supply chain: multi-stage build, base distroless, non-root, và đo lường bằng Trivy.

**Lời giải — build bản "full" và đo baseline:**

```bash
cd ~/ch17-cks

# Dockerfile.full đã được setup tạo sẵn (ubuntu + app Go tĩnh)
docker build -f Dockerfile.full -t ch17-app:full .
{% raw %}docker image ls --format 'table {{.Repository}} {{.Tag}} {{.Size}}' | grep ch17-app{% endraw %}
trivy image --scanners vuln --severity HIGH,CRITICAL ch17-app:full
```

**Viết `Dockerfile.min`:**

```dockerfile
# Dockerfile.min
FROM golang:1.24-alpine AS builder
WORKDIR /src
COPY app/main.go .
RUN go mod init demo.local/ch17 && CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/app .

# Runtime: distroless static nonroot — không shell, không package manager
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /out/app /app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

```bash
docker build -f Dockerfile.min -t ch17-app:min .
{% raw %}docker image ls --format 'table {{.Repository}} {{.Tag}} {{.Size}}' | grep ch17-app{% endraw %}
```

**Verify:**

```bash
# 1. Dung lượng giảm ít nhất 5 lần
{% raw %}FULL=$(docker image inspect ch17-app:full --format '{{.Size}}')
MIN=$(docker image inspect ch17-app:min --format '{{.Size}}'){% endraw %}
echo "full=$FULL min=$MIN"    # ví dụ full≈95MB, min≈13MB

# 2. Không còn HIGH/CRITICAL có bản vá
trivy image --scanners vuln --severity HIGH,CRITICAL --ignore-unfixed ch17-app:min

# 3. Không có shell để kẻ tấn công dùng (lệnh này phải FAIL)
docker run --rm ch17-app:min sh -c 'echo hello'

# 4. Chạy non-root và trả kết quả
docker run --rm -d -p 8082:8080 --name ch17-min ch17-app:min
curl -s http://127.0.0.1:8082/healthz      # ok
docker rm -f ch17-min
```

**Push cả hai image lên registry nội bộ:**

```bash
docker tag ch17-app:min  kind-registry:5000/secure-app/ch17-app:min
docker tag ch17-app:full kind-registry:5000/secure-app/ch17-app:full
docker push kind-registry:5000/secure-app/ch17-app:min
docker push kind-registry:5000/secure-app/ch17-app:full
curl -s http://127.0.0.1:5000/v2/secure-app/ch17-app/tags/list
# {"name":"secure-app/ch17-app","tags":["full","min"]}
```

**Ghi chú:** `--ignore-unfixed` là lựa chọn thực dụng khi đặt gate: CVE chưa có bản vá thì không thể "sửa" bằng cách rebuild, chỉ có thể chấp nhận và theo dõi. Nhớ rằng dung lượng/CVE thay đổi theo ngày — điều quan trọng trong đề thi là **quy trình** (build minimal, scan, chứng minh), không phải con số cụ thể. Tag `:nonroot` của distroless là bắt buộc nếu task yêu cầu chạy non-root; bản `static-debian12` mặc định chạy root.

---

### Task B10 — Ký image và enforce bằng Kyverno {#loi-giai-b10}

**Mục tiêu kiểm tra:** Cosign sign với key, và admission policy `verifyImages` — chốt chặn supply chain.

**Lời giải — ký image:**

```bash
mkdir -p ~/ch17-cks/cosign && cd ~/ch17-cks/cosign

# Đặt password rỗng cho lab để không bị prompt; production dùng secret manager
export COSIGN_PASSWORD=""
cosign generate-key-pair
# sinh cosign.key (private) và cosign.pub (public)

# Ký theo tag; cosign resolve ra digest và gắn chữ ký vào digest
cosign sign --yes --key cosign.key --allow-http-registry \
  kind-registry:5000/secure-app/ch17-app:min

# Xác minh chữ ký trước khi viết policy
cosign verify --key cosign.pub --allow-http-registry \
  kind-registry:5000/secure-app/ch17-app:min | jq -r '.[0].critical.identity["docker-reference"]'
```

**Tạo ClusterPolicy (`cosign.pub` được nhúng trực tiếp):**

```bash
cd ~/ch17-cks
cat > kyverno-verify.yaml <<EOF
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-ch17-images
spec:
  background: false
  validationFailureAction: Enforce
  webhookConfiguration:
    failurePolicy: Fail
    timeoutSeconds: 30
  rules:
    - name: verify-cosign-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - secure-app
      verifyImages:
        - imageReferences:
            - "kind-registry:5000/secure-app/*"
          mutateDigest: true
          verifyDigest: false
          required: true
          imageRegistryCredentials:
            allowInsecureRegistry: true
          attestors:
            - entries:
                - keys:
                    publicKeys: |
$(sed 's/^/                      /' cosign/cosign.pub)
EOF

kubectl apply -f kyverno-verify.yaml
kubectl get clusterpolicy verify-ch17-images
```

**Verify:**

```bash
# 1. Image đã ký -> được admit và chạy
kubectl -n secure-app run signed --image=kind-registry:5000/secure-app/ch17-app:min
kubectl -n secure-app wait --for=condition=Ready pod/signed --timeout=120s
kubectl -n secure-app get pod signed -o jsonpath='{.spec.containers[0].image}{"\n"}'
# kind-registry:5000/secure-app/ch17-app@sha256:...   (mutateDigest đổi tag thành digest)

# 2. Image chưa ký -> bị chặn
kubectl -n secure-app run unsigned --image=kind-registry:5000/secure-app/ch17-app:full
# Error from server: admission webhook "mutate.kyverno.svc-fail" denied the request:
# resource Pod/secure-app/unsigned was blocked due to the following policies
# verify-ch17-images:
#   verify-cosign-signature: 'failed to verify image ...: no signatures found'
```

**Ghi chú:** Ba chi tiết quyết định thành công: (1) Kyverno phải resolve và gọi được registry HTTP nội bộ (hostAlias + `--allowInsecureRegistry=true` đã làm ở setup); (2) `imageReferences` phải khớp **nguyên văn** tên image trong pod spec, kể cả host `kind-registry:5000`; (3) với môi trường air-gapped không có Rekor, phải ký `--tlog-upload=false` và cấu hình policy `rekor.ignoreTlog: true`. Trong production, dùng keyless (OIDC + Fulcio) và ràng buộc `--certificate-identity` theo pipeline thay vì key tĩnh.

---

### Task B11 — Audit logging {#loi-giai-b11}

**Mục tiêu kiểm tra:** Audit policy (level/stage/thứ tự rule), gắn vào kube-apiserver static pod, đọc log bằng jq.

**Lời giải — tạo policy:**

```bash
cd ~/ch17-cks
mkdir -p audit
cat > audit/policy.yaml <<'EOF'
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # 1. exec vào pod của secure-rbac: cần RequestResponse để điều tra
  - level: RequestResponse
    namespaces: ["secure-rbac"]
    resources:
      - group: ""
        resources: ["pods/exec"]
  # 2. secrets: chỉ Metadata — KHÔNG ghi giá trị
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
  # 3. Mặc định: Metadata, bỏ stage RequestReceived cho đỡ nhiễu
  - level: Metadata
    omitStages:
      - RequestReceived
EOF

docker exec ch17-cks-control-plane mkdir -p /etc/kubernetes/audit /var/log/kubernetes
docker cp audit/policy.yaml ch17-cks-control-plane:/etc/kubernetes/audit/policy.yaml
```

**Sửa manifest kube-apiserver** (nhớ backup trước):

```bash
docker cp ch17-cks-control-plane:/etc/kubernetes/manifests/kube-apiserver.yaml .
cp kube-apiserver.yaml kube-apiserver.yaml.bak
```

Thêm vào `command:`:

```yaml
    - --audit-policy-file=/etc/kubernetes/audit/policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=7
    - --audit-log-maxbackup=3
    - --audit-log-maxsize=50
```

Thêm vào `volumeMounts:` của container:

```yaml
    - mountPath: /etc/kubernetes/audit
      name: audit-policy
      readOnly: true
    - mountPath: /var/log/kubernetes
      name: audit-log
```

Thêm vào `volumes:` của pod:

```yaml
  - hostPath:
      path: /etc/kubernetes/audit
      type: DirectoryOrCreate
    name: audit-policy
  - hostPath:
      path: /var/log/kubernetes
      type: DirectoryOrCreate
    name: audit-log
```

```bash
docker cp kube-apiserver.yaml ch17-cks-control-plane:/etc/kubernetes/manifests/kube-apiserver.yaml
kubectl get --raw='/readyz?verbose'
```

**Sinh event và verify:**

```bash
# Tạo pod để có đối tượng exec, rồi exec vào nó
kubectl -n secure-rbac run exec-test --image=nginx:1.27-alpine --restart=Never
kubectl -n secure-rbac wait --for=condition=Ready pod/exec-test --timeout=90s
kubectl -n secure-rbac exec exec-test -- true

AUDIT=/var/log/kubernetes/audit.log

# 1. Tìm event pods/exec (user, pod, verb)
docker exec ch17-cks-control-plane cat $AUDIT \
  | jq -c 'select(.objectRef.subresource=="exec") | {user: .user.username, pod: .objectRef.name, ns: .objectRef.namespace, verb: .verb}' \
  | tail -n 3
# {"user":"kubernetes-admin","pod":"exec-test","ns":"secure-rbac","verb":"create"}

# 2. Secret: chỉ Metadata, không có request body
docker exec ch17-cks-control-plane cat $AUDIT \
  | jq -c 'select(.objectRef.resource=="secrets") | {user: .user.username, name: .objectRef.name, hasRequestBody: (.requestObject != null)}' \
  | tail -n 3
# ... "hasRequestBody": false

# 3. Chứng minh giá trị secret không nằm trong log
B64=$(kubectl -n secure-app get secret db-creds -o jsonpath='{.data.password}')
docker exec ch17-cks-control-plane grep -c "$B64" $AUDIT || echo "Giá trị secret KHÔNG xuất hiện trong audit log"
```

**Nếu apiserver không lên:** `docker cp kube-apiserver.yaml.bak ch17-cks-control-plane:/etc/kubernetes/manifests/kube-apiserver.yaml`, sau đó `kubectl get --raw='/readyz'`.

**Ghi chú:** Audit policy được đánh giá **từ trên xuống**, rule đầu tiên match quyết định level — nếu đặt rule `secrets: RequestResponse` lên trên, giá trị secret (base64) sẽ nằm nguyên trong log, biến file audit thành nơi rò rỉ credential. Cẩn thận với `omitStages: RequestReceived`: nó làm giảm khối lượng log nhưng cũng có nghĩa bạn không thấy request trước khi xử lý — với `pods/exec` nên giữ cả hai stage để điều tra.

---

### Task B12 — Falco rule {#loi-giai-b12}

**Mục tiêu kiểm tra:** Viết rule behavioral analytics trên syscall, validate cú pháp, và hiểu giới hạn của detect-only.

**Lời giải — viết rule:**

```yaml
# ch17-falco-rules.yaml
- rule: Shell trong namespace secure-rbac
  desc: Phát hiện shell được spawn trong container thuộc namespace secure-rbac
  condition: >
    spawned_process and container
    and k8s.ns.name = "secure-rbac"
    and proc.name in (bash, sh, ash)
  output: >
    Shell trong container (user=%user.name pod=%k8s.pod.name ns=%k8s.ns.name cmd=%proc.cmdline)
  priority: WARNING
  tags: [container, shell, mitre_execution]
```

**Validate bằng container Falco:**

```bash
cd ~/ch17-cks
docker run --rm -v "$PWD/ch17-falco-rules.yaml:/rules.yaml:ro" \
  aquasec/falco:latest falco --validate /rules.yaml
# Không có output lỗi parse nghĩa là file hợp lệ
```

**Chạy live (tùy chọn — cần kernel hỗ trợ eBPF):**

```bash
# Đảm bảo có pod để tạo shell (nếu chưa làm task B11)
kubectl -n secure-rbac run exec-test --image=nginx:1.27-alpine --restart=Never
kubectl -n secure-rbac wait --for=condition=Ready pod/exec-test --timeout=90s

helm repo add falcosecurity https://falcosecurity.github.io/charts && helm repo update

cat > falco-values.yaml <<'EOF'
driver:
  kind: modern_ebpf
collectors:
  kubernetes:
    enabled: true
customRules:
  ch17_rules.yaml: |-
    - rule: Shell trong namespace secure-rbac
      desc: Phát hiện shell được spawn trong container thuộc namespace secure-rbac
      condition: >
        spawned_process and container
        and k8s.ns.name = "secure-rbac"
        and proc.name in (bash, sh, ash)
      output: >
        Shell trong container (user=%user.name pod=%k8s.pod.name ns=%k8s.ns.name cmd=%proc.cmdline)
      priority: WARNING
      tags: [container, shell, mitre_execution]
EOF

helm install falco falcosecurity/falco -n falco --create-namespace -f falco-values.yaml
kubectl -n falco rollout status daemonset/falco --timeout=300s

# Kích hoạt rule
kubectl -n secure-rbac exec exec-test -- sh -c 'echo demo'
kubectl -n falco logs -l app.kubernetes.io/name=falco --tail=50 | grep "Shell trong namespace secure-rbac"
# 14:02:11 Warning Shell trong container (user=... pod=exec-test ns=secure-rbac cmd=sh -c echo demo)
```

**Verify/ghi chú (trường hợp không chạy được driver):**

```bash
kubectl -n falco get pods
# Nếu pod crash với lỗi driver/BTF trên Docker Desktop/WSL2, đây là giới hạn môi trường,
# không phải lỗi rule. Phần bắt buộc là --validate đã pass.

# Kịch bản gây alert cho rule này (chạy tay để ghi vào báo cáo):
kubectl -n secure-rbac exec exec-test -- sh
```

**Ghi chú:** Rule dựa trên `proc.name` rất dễ bị né bằng cách đổi tên binary hoặc dùng `python -c`, vì vậy trong production hãy kết hợp nhiều tín hiệu (process tree, kết nối mạng ra ngoài, ghi file nhạy cảm). Falco là **detect-only**: đến lúc rule fire thì syscall đã thực thi xong — muốn chặn phải dùng Tetragon/AppArmor/seccomp hoặc response engine. `k8s.ns.name` cần enrichment container metadata (image/driver hiện đại + k8s-metacollector); nếu chạy Falco ở dạng plugin/audit-only, field này có thể rỗng.

---

Sau khi chấm xong: nếu bạn đạt trên ngưỡng ở cả hai bộ và làm hết trong thời gian, hãy quay lại các bộ đề ở chương này sau 1–2 tuần để kiểm tra độ bền kỹ năng — đề thi thật luôn có biến thể mới, nhưng kỹ năng chẩn đoán và quy trình verify thì không đổi.