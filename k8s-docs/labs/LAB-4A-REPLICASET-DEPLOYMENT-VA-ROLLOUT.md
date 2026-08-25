# Lab 4a — ReplicaSet, Deployment và rollout

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem
> [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> **Điểm kết thúc:** cleanup trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới.
> **Lab trước:** Lab 3c — Tài nguyên, QoS và gián đoạn (chưa viết, xem
> [bản đồ lab](README.md#4-bản-đồ-lab)). Nếu bạn tới thẳng từ [Lab 2](LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md)
> thì cluster vẫn đang ở đúng `01-cluster-ready` và lab này chạy được ngay.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục
[4a. ReplicaSet, Deployment và rollout](../00-ALO-TRINH-ADMIN.md#4a-replicaset-deployment-và-rollout).
Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này không chép
lại con số phiên bản nào và **không cài thêm gì**.

Thứ tự trong lab bám đúng thứ tự lộ trình: **ReplicaSet trước Deployment**. Phần B1 dựng một
ReplicaSet đứng một mình để thấy vòng lặp điều khiển trần trụi, rồi B2 mới cho thấy Deployment
vận hành *thông qua* chính cơ chế đó. Đọc ngược lại thì rollout chỉ còn là một chuỗi lệnh học
thuộc.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- ReplicaSet gồm đúng ba thứ — **selector**, **số replica**, **pod template** — và nó tạo/xóa Pod
  cho tới khi số Pod khớp con số mong muốn; chỉ ra được vòng lặp điều khiển đã học ở giai đoạn 1
  đang chạy ở đâu.
- ReplicaSet **thu nhận** mọi Pod khớp selector mà chưa có controller làm owner, kể cả Pod trần
  bạn tạo tay, và ghi quan hệ đó vào `metadata.ownerReferences` của Pod.
- ReplicaSet mới cùng selector **nhận nuôi** Pod cũ nhưng **không** cập nhật chúng theo template
  mới — chính chỗ này là lý do tồn tại của Deployment.
- Deployment không quản Pod trực tiếp: nó quản ReplicaSet, mỗi pod template sinh một ReplicaSet
  riêng phân biệt bằng nhãn `pod-template-hash`; chỉ ra được chuỗi
  Pod → ReplicaSet → Deployment bằng UID thật.
- Rollout và revision **chỉ** sinh ra khi `.spec.template` đổi; scale không tạo revision và không
  tạo ReplicaSet mới.
- Đọc được `rollout status`, `rollout history`, `rollout undo`; chứng minh revision tăng, ReplicaSet
  cũ vẫn còn ở `DESIRED = 0`, và `revisionHistoryLimit` quyết định bạn còn rollback về đâu.
- `maxSurge` và `maxUnavailable` quy định khoảng dao động của rollout; đo được ràng buộc đó bằng số
  liệu lấy trên cluster của mình chứ không chép từ tài liệu.
- Phân biệt `Recreate` với `RollingUpdate` bằng một phép đo, và nói được vì sao bảo đảm của
  `Recreate` không áp dụng khi bạn xóa tay một Pod.
- Đọc một rollout hỏng: ReplicaSet mới kẹt, ReplicaSet cũ vẫn phục vụ, condition `Progressing`
  chuyển `False` với `reason: ProgressDeadlineExceeded`, và Kubernetes **không tự rollback**.
- Chọn đúng propagation policy khi xóa Deployment: foreground, background hay orphan.
- Nhận ra `ReplicationController` khi gặp cluster cũ và nói được nó khác ReplicaSet ở đúng chỗ nào.
- Cleanup toàn bộ object lab và đưa cluster về `01-cluster-ready`.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 4a | Phần lab kiểm chứng |
| --- | --- |
| [62 — Quản lý Workload, trang mục các controller](../62-controllers-index-vi.md) | B1 và B2 — bạn khai báo object cao hơn Pod, control plane tạo và xóa Pod thay bạn |
| [64 — ReplicaSet](../64-replicaset-vi.md) | B1 — selector/replicas/template, `ownerReferences`, thu nhận Pod trần, `--cascade=orphan` và giới hạn "không rolling update" |
| [63 — Deployment](../63-deployment-vi.md) | B2 `pod-template-hash`; B3 scale không tạo revision; B4 rollout/rollback/`revisionHistoryLimit`; B5 `maxSurge`/`maxUnavailable`; B6 `Recreate`; B7 rollout thất bại và `progressDeadlineSeconds` |
| [61 — Quản lý Workload bằng kubectl](../61-management-vi.md) | B4 `rollout status --timeout`/`--watch=false`; B9 gộp manifest bằng `---`, `-R`, thao tác hàng loạt bằng `-l`, `replace --force`, canary thủ công |
| [70 — ReplicationController](../70-replicationcontroller-vi.md) | B10 — nhận diện `rc` trên API và chỉ ra khác biệt selector; lab **không** dựng `rc` |
| [260 — Sử dụng xóa theo tầng trong Cluster](../260-use-cascading-deletion-vi.md) | B8 — `ownerReferences` trên Pod, rồi foreground, background và orphan trên cùng một Deployment |
| [319 — Quản lý object khai báo bằng file cấu hình](../319-declarative-config-vi.md) | B9 — annotation `last-applied-configuration`, `kubectl diff`, apply ghi đè thao tác scale thủ công |
| [324 — Cập nhật đối tượng API tại chỗ bằng kubectl patch](../324-kubectl-patch-vi.md) | B3 `--subresource=scale` và JSON patch có phép `test`; B9 strategic merge patch so với JSON merge patch trên list `containers` |
| [345 — Chạy một ứng dụng Stateless bằng Deployment](../345-run-stateless-application-vi.md) | B2 — tạo Deployment từ YAML, `describe`, liệt kê Pod theo label; B9 — cập nhật và scale bằng `kubectl apply` |
| [346 — Scale thủ công theo chiều ngang cho một Deployment](../346-scale-deployment-vi.md) | B3 — `kubectl scale` lên, xuống và về 0; `kubectl patch` có tiền kiểm; bảng so sánh với HPA chỉ đọc |
| [348 — Cập nhật một Deployment mà không gây gián đoạn](../348-update-deployment-rolling-vi.md) | B4 `rollout status`/`history`/`undo`, pause và resume; B5 tham số chiến lược; B6 `Recreate`; B7 phát hiện rollout đình trệ |

Một bài của nhóm **không kiểm chứng được trên cluster lab**, đọc để biết:

| Bài | Vì sao không có bước thực hành |
| --- | --- |
| [337 — Chạy ứng dụng](../337-run-application-vi.md) | Là trang mục lục của nhánh `/docs/tasks/run-application/`, toàn bộ nội dung là danh sách link, không có thao tác nào để kiểm chứng. Các trang con thuộc nhóm 4a đã nằm ở bảng trên; các trang con còn lại thuộc [nhóm 4b](../00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling) và giai đoạn sau |

Hai điểm của nhóm bài **cố ý dừng ở phần đạt được bằng kiến thức đã có**, không phát sinh nợ mới:

- Lệnh `kubectl autoscale` trong bài [61](../61-management-vi.md) và bảng so sánh HPA trong bài
  [346](../346-scale-deployment-vi.md) cần metrics-server của giai đoạn 11. Đây đúng là
  [nợ #1 đã ghi sẵn](README.md#5-sổ-nợ-lab) và được trả ở Lab 11b; lab này chỉ scale thủ công.
- Mẫu canary của bài [61](../61-management-vi.md) chia lưu lượng bằng một Service phủ cả hai tập
  Pod. Service thuộc giai đoạn 5, nên B9 chỉ dựng và đo **phần tỷ lệ replica cùng bộ label chung** —
  đúng phần mà chính bài 61 xếp là phải hiểu ở lần đọc này; phần Service định tuyến để lại cho
  giai đoạn 5.

### 1.2. Thời lượng

2–3 giờ. B5, B6 và B7 có bước lấy mẫu và bước chờ deadline nên tốn thời gian hơn phần còn lại.

---

## 2. Quy ước và an toàn

- Mọi lệnh chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, trừ khi ghi rõ khác.
- Mọi object lab tạo ra nằm trong namespace `lab-4a` và luôn được gọi kèm `-n lab-4a`. Lab không
  đổi namespace mặc định của context.
- Lab chỉ tạo Namespace, Deployment, ReplicaSet, Pod và ConfigMap. **Không** tạo Service,
  StatefulSet, DaemonSet, Job, HorizontalPodAutoscaler, PersistentVolumeClaim — tất cả thuộc giai
  đoạn sau. **Không** tạo ReplicationController: bài 70 là tài liệu lịch sử.
- **Fault injection chỉ trên `lab-k8s-worker2`** — B7 ghim toàn bộ Pod của Deployment hỏng vào node
  này bằng `nodeName`.
- Manifest tạm ghi vào `~/lab-work/4a/`, bằng chứng ghi vào `~/lab-evidence/4a/`. Không lưu token,
  key hay certificate.
- Lab cần kéo được image `busybox` từ internet. Nếu môi trường không có egress, xem mục 4.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail.
- Phần B dùng biến và hàm shell nối tiếp giữa các bước, nên chạy **toàn bộ phần B trong cùng một
  phiên shell** trên `lab-k8s-master`. Mở phiên mới giữa chừng thì phải chạy lại từ đầu mục B chứa
  bước đang dở.
- Lab **không** sửa cấu hình node, không sửa manifest control plane, không cài gói.

---

# Phần B — Thực hành kiến thức 4a

## B0. Chuẩn bị workspace và namespace

```bash
mkdir -p ~/lab-work/4a ~/lab-evidence/4a
kubectl config current-context
kubectl get nodes
kubectl create namespace lab-4a
kubectl get namespace lab-4a -o jsonpath='{.status.phase}'; echo
```

**Ý nghĩa:** `lab-work` chứa manifest tạm, `lab-evidence` chứa output. Namespace `lab-4a` cô lập
mọi object lab tạo ra, nên gate cuối chỉ cần chứng minh namespace đã biến mất.

**PASS:** context trỏ đúng cluster lab; ba node `Ready`; namespace `lab-4a` ở phase `Active`.

## B1. ReplicaSet đứng một mình

Bài [64](../64-replicaset-vi.md) nói ReplicaSet được định nghĩa bằng ba thứ: selector, số replica
và pod template. Phần này dựng đúng ba thứ đó và ép chúng lộ ra hành vi.

### B1.1. Hai Pod trần có mặt trước

Tạo Pod trần **trước** ReplicaSet là kịch bản xác định trong bài 64: ReplicaSet sẽ thu nhận chúng
rồi chỉ tạo bù cho đủ số mong muốn.

```bash
cat > ~/lab-work/4a/pod-tran.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  namespace: lab-4a
  labels:
    tier: frontend
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 36000"]
---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  namespace: lab-4a
  labels:
    tier: frontend
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 36000"]
EOF

kubectl apply -f ~/lab-work/4a/pod-tran.yaml
kubectl wait --for=condition=Ready pod/pod1 pod/pod2 -n lab-4a --timeout=180s
```

Hai Pod này chưa có owner nào:

```bash
OWNERS="$(kubectl get pod pod1 pod2 -n lab-4a \
  -o jsonpath='{range .items[*]}{.metadata.ownerReferences}{"\n"}{end}')"
echo "ownerReferences hien tai: [$OWNERS]"
test -z "$(echo "$OWNERS" | tr -d '[:space:]')" \
  && echo 'PASS: hai Pod tran chua co ownerReferences'
```

**PASS:** dòng `PASS: hai Pod tran chua co ownerReferences` xuất hiện.

### B1.2. ReplicaSet thu nhận Pod khớp selector

```bash
cat > ~/lab-work/4a/rs-frontend.yaml <<'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  namespace: lab-4a
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 36000"]
EOF

kubectl apply -f ~/lab-work/4a/rs-frontend.yaml

for i in $(seq 1 60); do
  R="$(kubectl get rs frontend -n lab-4a -o jsonpath='{.status.readyReplicas}')"
  test "${R:-0}" = '3' && break
  sleep 2
done
kubectl get rs frontend -n lab-4a
kubectl get pods -n lab-4a -l tier=frontend -o wide \
  | tee ~/lab-evidence/4a/b1-pods-sau-khi-thu-nhan.txt
```

Kiểm chứng: tổng cộng đúng 3 Pod, nhưng ReplicaSet chỉ **tự tạo đúng 1** vì hai Pod trần đã bị thu
nhận.

```bash
TOTAL="$(kubectl get pods -n lab-4a -l tier=frontend --no-headers | wc -l)"
CREATED="$(kubectl get pods -n lab-4a -l tier=frontend -o name | grep -c 'pod/frontend-')"
echo "tong Pod = $TOTAL, Pod do ReplicaSet tao = $CREATED"

test "$TOTAL" = '3' && test "$CREATED" = '1' \
  && echo 'PASS: ReplicaSet thu nhan 2 Pod tran va chi tao bu 1 Pod'
```

**Ý nghĩa:** ReplicaSet không đếm "Pod do tôi tạo", nó đếm **Pod khớp selector**. Pod nào không có
OwnerReference — hoặc có nhưng không phải controller — mà khớp selector thì bị thu nhận ngay. Đây là
lý do bài 64 khuyến nghị đừng để Pod trần trùng label với selector của bất kỳ ReplicaSet nào.

**PASS:** dòng `PASS: ReplicaSet thu nhan 2 Pod tran va chi tao bu 1 Pod` xuất hiện.

### B1.3. `ownerReferences` là sợi dây thật, đối chiếu bằng UID

```bash
RS_UID="$(kubectl get rs frontend -n lab-4a -o jsonpath='{.metadata.uid}')"
OWNER_KIND="$(kubectl get pod pod1 -n lab-4a -o jsonpath='{.metadata.ownerReferences[0].kind}')"
OWNER_NAME="$(kubectl get pod pod1 -n lab-4a -o jsonpath='{.metadata.ownerReferences[0].name}')"
OWNER_UID="$(kubectl get pod pod1 -n lab-4a -o jsonpath='{.metadata.ownerReferences[0].uid}')"
OWNER_CTRL="$(kubectl get pod pod1 -n lab-4a -o jsonpath='{.metadata.ownerReferences[0].controller}')"

printf 'rs uid   = %s\nowner    = %s/%s\nowner uid= %s\ncontroller=%s\n' \
  "$RS_UID" "$OWNER_KIND" "$OWNER_NAME" "$OWNER_UID" "$OWNER_CTRL" \
  | tee ~/lab-evidence/4a/b1-ownerref-pod1.txt

test "$OWNER_KIND" = 'ReplicaSet' \
  && test "$OWNER_NAME" = 'frontend' \
  && test "$OWNER_UID" = "$RS_UID" \
  && test "$OWNER_CTRL" = 'true' \
  && echo 'PASS: ownerReferences cua pod1 tro dung ReplicaSet frontend bang UID'
```

**Ý nghĩa:** selector trả lời câu hỏi "Pod nào tôi được thu nhận"; `ownerReferences` trả lời câu hỏi
"Pod này thuộc về ai". Hai câu hỏi khác nhau, và bài 64 dùng cả hai. UID mới là thứ chống nhầm lẫn:
tên có thể trùng lại sau khi object bị xóa và tạo lại, UID thì không.

**PASS:** dòng `PASS: ownerReferences cua pod1 tro dung ReplicaSet frontend bang UID` xuất hiện.

### B1.4. Xóa tay một Pod và nhìn vòng lặp điều khiển làm việc

```bash
VICTIM="$(kubectl get pods -n lab-4a -l tier=frontend -o name | grep 'pod/frontend-' | head -1)"
echo "se xoa: $VICTIM"
kubectl delete "$VICTIM" -n lab-4a --wait=true

for i in $(seq 1 60); do
  R="$(kubectl get rs frontend -n lab-4a -o jsonpath='{.status.readyReplicas}')"
  test "${R:-0}" = '3' && break
  sleep 2
done

TOTAL="$(kubectl get pods -n lab-4a -l tier=frontend --no-headers | wc -l)"
echo "tong Pod sau khi xoa tay = $TOTAL"
kubectl describe rs frontend -n lab-4a | tee ~/lab-evidence/4a/b1-describe-rs.txt >/dev/null

test "$TOTAL" = '3' && echo 'PASS: ReplicaSet dua so Pod ve lai 3 sau khi ban xoa tay 1 Pod'
kubectl get pods -n lab-4a --no-headers -o name | grep -qx "$VICTIM" \
  && echo 'FAIL: Pod cu van con' \
  || echo 'PASS: Pod bi xoa khong quay lai, Pod thay the la Pod moi'
grep -q 'SuccessfulCreate' ~/lab-evidence/4a/b1-describe-rs.txt \
  && echo 'PASS: event SuccessfulCreate cua replicaset-controller co trong describe'
```

**Ý nghĩa:** đây chính là vòng lặp điều khiển của giai đoạn 1, chỉ đổi đối tượng. Controller so
**trạng thái mong muốn** (`.spec.replicas = 3`) với **trạng thái thực tế** (số Pod khớp selector),
thấy lệch thì hành động, rồi lặp lại. Nó không "phục hồi" Pod cũ — nó tạo một Pod hoàn toàn mới từ
pod template.

**PASS:** cả ba dòng `PASS:` của bước này xuất hiện, không có dòng `FAIL:`.

### B1.5. Thừa Pod thì ReplicaSet xóa bớt

```bash
kubectl run pod3 -n lab-4a --image=busybox:1.36 --labels=tier=frontend \
  --command -- sh -c 'sleep 36000'

for i in $(seq 1 60); do
  TOTAL="$(kubectl get pods -n lab-4a -l tier=frontend --no-headers | wc -l)"
  test "$TOTAL" = '3' && break
  sleep 2
done
kubectl get pods -n lab-4a -l tier=frontend
kubectl describe rs frontend -n lab-4a | tee ~/lab-evidence/4a/b1-describe-rs-delete.txt >/dev/null

test "$TOTAL" = '3' \
  && echo 'PASS: ReplicaSet dua so Pod ve lai 3 sau khi ban tao them Pod tran'
grep -q 'SuccessfulDelete' ~/lab-evidence/4a/b1-describe-rs-delete.txt \
  && echo 'PASS: event SuccessfulDelete xac nhan ReplicaSet chu dong xoa Pod thua'
```

**Ý nghĩa:** thu nhận là dao hai lưỡi. Pod trần tạo **sau** khi ReplicaSet đã đủ replica vẫn bị thu
nhận, nhưng bị chấm dứt ngay vì khi đó ReplicaSet vượt quá số lượng mong muốn. Thứ tự chọn Pod để
xóa theo bài 64: Pod Pending trước, rồi `pod-deletion-cost` thấp hơn, rồi Pod trên node có nhiều
replica hơn, rồi Pod được tạo gần đây hơn.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B1.6. Hai ràng buộc mà API server từ chối ngay

```bash
cat > ~/lab-work/4a/rs-sai-selector.yaml <<'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-sai-selector
  namespace: lab-4a
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: backend
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 36000"]
EOF

cat > ~/lab-work/4a/rs-sai-restart.yaml <<'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-sai-restart
  namespace: lab-4a
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: backend
  template:
    metadata:
      labels:
        tier: backend
    spec:
      restartPolicy: OnFailure
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 36000"]
EOF
```

Cả hai đều chỉ chạy ở chế độ `--dry-run=server`, nên không object nào được tạo:

```bash
if kubectl apply --dry-run=server -f ~/lab-work/4a/rs-sai-selector.yaml \
     2>~/lab-evidence/4a/b1-loi-selector.txt; then
  echo 'FAIL: API server chap nhan template labels khong khop selector'
else
  grep -qi 'selector' ~/lab-evidence/4a/b1-loi-selector.txt \
    && echo 'PASS: API tu choi ReplicaSet co template labels khong khop selector'
fi

if kubectl apply --dry-run=server -f ~/lab-work/4a/rs-sai-restart.yaml \
     2>~/lab-evidence/4a/b1-loi-restart.txt; then
  echo 'FAIL: API server chap nhan restartPolicy khac Always'
else
  grep -qi 'restartPolicy' ~/lab-evidence/4a/b1-loi-restart.txt \
    && echo 'PASS: API tu choi restartPolicy khac Always trong pod template'
fi
```

**Ý nghĩa:** `.spec.template.metadata.labels` **phải** khớp `.spec.selector`, và
`.spec.template.spec.restartPolicy` chỉ được là `Always`. Hai ràng buộc này được API server ép,
không phải quy ước. Hệ quả thực tế: một ReplicaSet không thể "tạo Pod mà chính nó không quản".

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B1.7. Giới hạn của ReplicaSet: nhận nuôi nhưng không cập nhật

Đây là bước then chốt của cả lab. Nó cho thấy vì sao Deployment tồn tại.

```bash
kubectl delete rs frontend -n lab-4a --cascade=orphan --wait=true

for i in $(seq 1 60); do
  REFS="$(kubectl get pods -n lab-4a -l tier=frontend \
    -o jsonpath='{range .items[*]}{.metadata.ownerReferences}{end}' | tr -d '[:space:]')"
  test -z "$REFS" && break
  sleep 2
done

TOTAL="$(kubectl get pods -n lab-4a -l tier=frontend --no-headers | wc -l)"
echo "so Pod con lai = $TOTAL, ownerReferences con lai = [$REFS]"
test "$TOTAL" = '3' && test -z "$REFS" \
  && echo 'PASS: --cascade=orphan xoa ReplicaSet, giu 3 Pod va go ownerReferences'
```

Bây giờ tạo một ReplicaSet **cùng selector nhưng pod template dùng image mới**:

```bash
sed -e 's/name: frontend$/name: frontend-v2/' \
    -e 's/image: busybox:1.36/image: busybox:1.37/' \
    ~/lab-work/4a/rs-frontend.yaml > ~/lab-work/4a/rs-frontend-v2.yaml
grep -E 'name: frontend-v2|image:' ~/lab-work/4a/rs-frontend-v2.yaml

kubectl apply -f ~/lab-work/4a/rs-frontend-v2.yaml
for i in $(seq 1 60); do
  R="$(kubectl get rs frontend-v2 -n lab-4a -o jsonpath='{.status.readyReplicas}')"
  test "${R:-0}" = '3' && break
  sleep 2
done
kubectl get rs frontend-v2 -n lab-4a
```

Đếm số Pod và đọc image **thực tế đang chạy** so với image trong template của ReplicaSet mới:

```bash
TMPL_IMG="$(kubectl get rs frontend-v2 -n lab-4a \
  -o jsonpath='{.spec.template.spec.containers[0].image}')"
POD_IMGS="$(kubectl get pods -n lab-4a -l tier=frontend \
  -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}' | sort -u)"
TOTAL="$(kubectl get pods -n lab-4a -l tier=frontend --no-headers | wc -l)"

printf 'template cua frontend-v2 = %s\nimage dang chay        = %s\nso Pod = %s\n' \
  "$TMPL_IMG" "$(echo "$POD_IMGS" | tr '\n' ' ')" "$TOTAL" \
  | tee ~/lab-evidence/4a/b1-nhan-nuoi-khong-cap-nhat.txt

test "$TOTAL" = '3' && test "$TMPL_IMG" = 'busybox:1.37' \
  && test "$POD_IMGS" = 'busybox:1.36' \
  && echo 'PASS: ReplicaSet moi nhan nuoi Pod cu va KHONG cap nhat chung theo template moi'
```

Dọn phần B1 trước khi sang Deployment:

```bash
kubectl delete rs frontend-v2 -n lab-4a --wait=true
for i in $(seq 1 60); do
  TOTAL="$(kubectl get pods -n lab-4a -l tier=frontend --no-headers | wc -l)"
  test "$TOTAL" = '0' && break
  sleep 2
done
test "$TOTAL" = '0' && echo 'PASS: xoa ReplicaSet mac dinh xoa luon Pod cua no'
```

**Ý nghĩa:** ReplicaSet chỉ đếm số Pod khớp selector. Template của nó chỉ được dùng **khi tạo Pod
mới**, nên Pod đang chạy không bao giờ được "áp" template mới. Muốn đưa Pod sang spec mới một cách
có kiểm soát thì phải có một tầng ở trên biết tạo ReplicaSet mới rồi chuyển dần replica — tầng đó
chính là Deployment.

**PASS:** cả ba dòng `PASS:` của B1.7 xuất hiện.

## B2. Deployment vận hành thông qua ReplicaSet

```bash
cat > ~/lab-work/4a/web.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: lab-4a
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 36000"]
EOF

kubectl apply -f ~/lab-work/4a/web.yaml
kubectl rollout status deployment/web -n lab-4a --timeout=180s

kubectl get deployment,replicaset,pods -n lab-4a -l app=web \
  | tee ~/lab-evidence/4a/b2-deploy-rs-pods.txt
kubectl describe deployment web -n lab-4a | tee ~/lab-evidence/4a/b2-describe-deploy.txt >/dev/null
kubectl get pods -n lab-4a -l app=web --show-labels
```

Đọc ba cột của `kubectl get deployments`: `READY` là ready/desired, `UP-TO-DATE` là số replica đã
mang pod template mới nhất, `AVAILABLE` là số replica sẵn sàng phục vụ.

### B2.1. Chuỗi Pod → ReplicaSet → Deployment

```bash
DEP_UID="$(kubectl get deployment web -n lab-4a -o jsonpath='{.metadata.uid}')"
RS="$(kubectl get rs -n lab-4a -l app=web -o jsonpath='{.items[0].metadata.name}')"
RS_UID="$(kubectl get rs "$RS" -n lab-4a -o jsonpath='{.metadata.uid}')"
RS_OWNER_UID="$(kubectl get rs "$RS" -n lab-4a -o jsonpath='{.metadata.ownerReferences[0].uid}')"
RS_OWNER_KIND="$(kubectl get rs "$RS" -n lab-4a -o jsonpath='{.metadata.ownerReferences[0].kind}')"

POD="$(kubectl get pods -n lab-4a -l app=web -o jsonpath='{.items[0].metadata.name}')"
POD_OWNER_UID="$(kubectl get pod "$POD" -n lab-4a -o jsonpath='{.metadata.ownerReferences[0].uid}')"
POD_OWNER_KIND="$(kubectl get pod "$POD" -n lab-4a -o jsonpath='{.metadata.ownerReferences[0].kind}')"

printf 'Deployment web      uid=%s\nReplicaSet %s uid=%s owner=%s/%s\nPod %s owner=%s/%s\n' \
  "$DEP_UID" "$RS" "$RS_UID" "$RS_OWNER_KIND" "$RS_OWNER_UID" \
  "$POD" "$POD_OWNER_KIND" "$POD_OWNER_UID" \
  | tee ~/lab-evidence/4a/b2-chuoi-owner.txt

test "$RS_OWNER_KIND" = 'Deployment' && test "$RS_OWNER_UID" = "$DEP_UID" \
  && echo 'PASS: ReplicaSet thuoc so huu cua Deployment web'
test "$POD_OWNER_KIND" = 'ReplicaSet' && test "$POD_OWNER_UID" = "$RS_UID" \
  && echo 'PASS: Pod thuoc so huu cua ReplicaSet, khong phai cua Deployment'
```

**Ý nghĩa:** không có Pod nào mang `ownerReferences` trỏ thẳng tới Deployment. Deployment quản
ReplicaSet, ReplicaSet quản Pod — đúng hai tầng. Mọi thứ Deployment làm khi rollout đều quy về scale
các ReplicaSet của nó.

### B2.2. Nhãn `pod-template-hash` phân biệt các ReplicaSet

```bash
HASH="$(kubectl get rs "$RS" -n lab-4a -o jsonpath="{.metadata.labels['pod-template-hash']}")"
POD_HASH="$(kubectl get pod "$POD" -n lab-4a -o jsonpath="{.metadata.labels['pod-template-hash']}")"
SEL_HASH="$(kubectl get rs "$RS" -n lab-4a \
  -o jsonpath="{.spec.selector.matchLabels['pod-template-hash']}")"

printf 'rs name=%s\nrs label hash=%s\nrs selector hash=%s\npod label hash=%s\n' \
  "$RS" "$HASH" "$SEL_HASH" "$POD_HASH" | tee ~/lab-evidence/4a/b2-pod-template-hash.txt

test "$RS" = "web-$HASH" \
  && echo 'PASS: ten ReplicaSet co dang <ten-deployment>-<pod-template-hash>'
test "$HASH" = "$SEL_HASH" && test "$HASH" = "$POD_HASH" \
  && echo 'PASS: cung mot hash nam tren nhan ReplicaSet, selector cua no va nhan Pod'
```

**Ý nghĩa:** `pod-template-hash` được băm từ chính pod template và Deployment controller gắn nó vào
ReplicaSet, vào selector của ReplicaSet và vào nhãn Pod. Nhờ vậy hai ReplicaSet con của cùng một
Deployment **không bao giờ chồng lấn Pod của nhau** — và cũng vì vậy một Pod trần chỉ mang nhãn
`app: web` sẽ không bị ReplicaSet của Deployment thu nhận, khác hẳn tình huống ở B1.2. Đừng sửa nhãn
này.

**PASS:** bốn dòng `PASS:` của B2 xuất hiện.

## B3. Scale không phải là rollout

```bash
rev() { kubectl get deployment web -n lab-4a \
  -o jsonpath='{.metadata.annotations.deployment\.kubernetes\.io/revision}'; }
rscount() { kubectl get rs -n lab-4a -l app=web --no-headers | wc -l; }

RS_NAME_BEFORE="$(kubectl get rs -n lab-4a -l app=web -o jsonpath='{.items[0].metadata.name}')"
REV_BEFORE="$(rev)"; RS_BEFORE="$(rscount)"
echo "truoc: revision=$REV_BEFORE, so ReplicaSet=$RS_BEFORE, rs=$RS_NAME_BEFORE"
```

### B3.1. `kubectl scale` lên rồi xuống

```bash
kubectl scale deployment/web -n lab-4a --replicas=5
kubectl rollout status deployment/web -n lab-4a --timeout=180s
kubectl get deployment web -n lab-4a

REV_AFTER="$(rev)"; RS_AFTER="$(rscount)"
RS_NAME_AFTER="$(kubectl get rs -n lab-4a -l app=web -o jsonpath='{.items[0].metadata.name}')"
echo "sau: revision=$REV_AFTER, so ReplicaSet=$RS_AFTER, rs=$RS_NAME_AFTER"

test "$REV_AFTER" = "$REV_BEFORE" \
  && echo 'PASS: scale KHONG tao revision moi'
test "$RS_AFTER" = "$RS_BEFORE" && test "$RS_NAME_AFTER" = "$RS_NAME_BEFORE" \
  && echo 'PASS: scale KHONG tao ReplicaSet moi, van la ReplicaSet cu'
```

**Ý nghĩa:** rollout — và revision — chỉ được kích hoạt khi và chỉ khi `.spec.template` đổi. Scale
không đụng tới template, mà ReplicaSet lại được phân biệt bằng hash băm từ template, nên không có
ReplicaSet mới nào ra đời. Bài 63 nói rõ đây là chủ ý: nhờ vậy bạn scale thoải mái mà không làm rối
lịch sử rollout.

### B3.2. Scale về không rồi trở lại

```bash
kubectl scale deployment/web -n lab-4a --replicas=0
for i in $(seq 1 60); do
  P="$(kubectl get pods -n lab-4a -l app=web --no-headers | wc -l)"
  test "$P" = '0' && break
  sleep 2
done
kubectl get deployment web -n lab-4a
kubectl get rs -n lab-4a -l app=web

DEP_EXISTS=0; RS_EXISTS=0
kubectl get deployment web -n lab-4a >/dev/null 2>&1 && DEP_EXISTS=1
kubectl get rs "$RS_NAME_BEFORE" -n lab-4a >/dev/null 2>&1 && RS_EXISTS=1

test "$P" = '0' && test "$DEP_EXISTS" = '1' && test "$RS_EXISTS" = '1' \
  && echo 'PASS: scale ve 0 xoa het Pod nhung giu ca Deployment lan ReplicaSet'

kubectl scale deployment/web -n lab-4a --replicas=3
kubectl rollout status deployment/web -n lab-4a --timeout=180s
```

**Ý nghĩa:** scale về 0 là cách tạm dừng một workload mà không mất object và không mất lịch sử —
đúng ba tình huống bài 346 nêu: tiết kiệm tài nguyên, gỡ lỗi hoặc bảo trì, và kiểm soát chi phí ở
môi trường dev/staging.

### B3.3. Hai cách đổi số replica không dùng `kubectl scale`

```bash
kubectl patch deployment web -n lab-4a --subresource='scale' --type='merge' \
  -p '{"spec":{"replicas":4}}'
kubectl rollout status deployment/web -n lab-4a --timeout=180s
R4="$(kubectl get deployment web -n lab-4a -o jsonpath='{.spec.replicas}')"
test "$R4" = '4' && echo 'PASS: patch qua subresource scale doi duoc so replica'
```

JSON patch có phép `test` làm tiền kiểm — lần đầu phải thành công, lần hai phải bị chặn:

```bash
if kubectl patch deployment web -n lab-4a --type=json -p='[
  {"op": "test", "path": "/spec/replicas", "value": 4},
  {"op": "replace", "path": "/spec/replicas", "value": 3}
]'; then
  echo 'PASS: patch chay khi tien kiem replicas=4 dung'
else
  echo 'FAIL: patch that bai du tien kiem dung'
fi

if kubectl patch deployment web -n lab-4a --type=json -p='[
  {"op": "test", "path": "/spec/replicas", "value": 4},
  {"op": "replace", "path": "/spec/replicas", "value": 9}
]' 2>~/lab-evidence/4a/b3-patch-test-fail.txt; then
  echo 'FAIL: patch chay du tien kiem replicas=4 khong con dung'
else
  echo 'PASS: phep test chan patch khi gia tri hien tai da khac'
fi

kubectl rollout status deployment/web -n lab-4a --timeout=180s
kubectl get deployment web -n lab-4a -o jsonpath='{.spec.replicas}'; echo
```

**Ý nghĩa:** phép `test` của JSON patch là công cụ chống ghi đè khi nhiều người hoặc nhiều script
cùng sửa một Deployment: patch chỉ được áp dụng nếu giá trị hiện tại đúng như bạn tưởng. Còn
`--subresource=scale` cho phép sửa riêng subresource `scale` mà không đụng tới phần còn lại của
object.

**PASS:** sáu dòng `PASS:` của B3 xuất hiện; không có dòng `FAIL:`; `.spec.replicas` cuối cùng là 3.

## B4. Rollout, revision và rollback

### B4.1. Đổi image và theo dõi rollout

```bash
REV0="$(rev)"
kubectl annotate deployment/web -n lab-4a \
  kubernetes.io/change-cause='doi-image-len-1.37' --overwrite
kubectl set image deployment/web -n lab-4a app=busybox:1.37

kubectl rollout status deployment/web -n lab-4a --timeout=180s
echo "exit code cua rollout status = $?"
kubectl rollout status deployment/web -n lab-4a --watch=false
```

```bash
REV1="$(rev)"
COLS='NAME:.metadata.name,DESIRED:.spec.replicas,READY:.status.readyReplicas,IMAGE:.spec.template.spec.containers[0].image'
kubectl get rs -n lab-4a -l app=web -o "custom-columns=$COLS" \
  | tee ~/lab-evidence/4a/b4-rs-sau-rollout.txt

RS_COUNT="$(rscount)"
ZERO_COUNT="$(kubectl get rs -n lab-4a -l app=web \
  -o jsonpath='{range .items[*]}{.spec.replicas}{"\n"}{end}' | grep -c '^0$')"

echo "revision: $REV0 -> $REV1, so ReplicaSet=$RS_COUNT, so ReplicaSet o 0 replica=$ZERO_COUNT"
test "$REV1" = "$((REV0 + 1))" && echo 'PASS: doi template lam revision tang dung 1'
test "$RS_COUNT" = '2' && test "$ZERO_COUNT" = '1' \
  && echo 'PASS: co ReplicaSet moi va ReplicaSet cu duoc giu lai o DESIRED=0'
```

**Ý nghĩa:** rollout không sửa Pod. Nó tạo một ReplicaSet mới cho template mới rồi scale ReplicaSet
mới lên và ReplicaSet cũ về 0. ReplicaSet cũ **không bị xóa** — nó chính là nơi lưu revision cũ.
`--watch=false` chỉ in trạng thái rồi thoát, dùng khi script không muốn dừng chờ; `--timeout` dùng
khi script phải chờ nhưng có giới hạn.

### B4.2. Lịch sử revision

```bash
kubectl rollout history deployment/web -n lab-4a | tee ~/lab-evidence/4a/b4-history.txt
kubectl rollout history deployment/web -n lab-4a --revision="$REV1" \
  | tee ~/lab-evidence/4a/b4-history-rev-moi.txt

grep -q 'doi-image-len-1.37' ~/lab-evidence/4a/b4-history.txt \
  && echo 'PASS: CHANGE-CAUSE lay tu annotation kubernetes.io/change-cause'
grep -q 'busybox:1.37' ~/lab-evidence/4a/b4-history-rev-moi.txt \
  && echo 'PASS: rollout history --revision in ra template cua dung revision do'
```

**Ý nghĩa:** `CHANGE-CAUSE` không tự có. Nó được sao chép từ annotation
`kubernetes.io/change-cause` của Deployment sang revision; không đặt annotation thì cột đó là
`<none>`. Cờ `--record` của kubectl đời cũ đã bị loại bỏ dần, đừng dựa vào nó.

### B4.3. Rollback

```bash
kubectl rollout undo deployment/web -n lab-4a
kubectl rollout status deployment/web -n lab-4a --timeout=180s

REV2="$(rev)"
IMG="$(kubectl get deployment web -n lab-4a \
  -o jsonpath='{.spec.template.spec.containers[0].image}')"
RS_NOW="$(kubectl get rs -n lab-4a -l app=web \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.replicas}{"\n"}{end}' \
  | awk '$2=="3"{print $1}')"

echo "revision sau undo = $REV2, image = $IMG, ReplicaSet dang chay = $RS_NOW"
test "$IMG" = 'busybox:1.36' && echo 'PASS: undo dua pod template ve image cu'
test "$REV2" = "$((REV1 + 1))" \
  && echo 'PASS: undo cung la mot rollout nen revision van tang, khong lui so'
test "$RS_NOW" = "$RS_NAME_BEFORE" \
  && echo 'PASS: undo tai su dung dung ReplicaSet cu chu khong tao ReplicaSet thu ba'
test "$(rscount)" = '2' && echo 'PASS: van chi co 2 ReplicaSet'
```

Rollback về một revision cụ thể:

```bash
kubectl rollout undo deployment/web -n lab-4a --to-revision="$REV1"
kubectl rollout status deployment/web -n lab-4a --timeout=180s
IMG="$(kubectl get deployment web -n lab-4a \
  -o jsonpath='{.spec.template.spec.containers[0].image}')"
test "$IMG" = 'busybox:1.37' && echo 'PASS: --to-revision quay ve dung revision duoc chi dinh'
```

**Ý nghĩa:** rollback **chỉ** hoàn nguyên phần pod template. Số replica hiện tại không bị kéo về giá
trị của revision cũ, vì scale không thuộc revision. Và undo là một rollout mới, nên số revision đi
tiếp chứ không đi lùi.

### B4.4. Tạm dừng và tiếp tục rollout

```bash
kubectl rollout pause deployment/web -n lab-4a

RS_TRUOC="$(rscount)"; REV_TRUOC="$(rev)"
kubectl set image deployment/web -n lab-4a app=busybox:1.36
kubectl set env deployment/web -n lab-4a BUILD=paused
sleep 10
RS_SAU="$(rscount)"; REV_SAU="$(rev)"

echo "khi dang pause: ReplicaSet $RS_TRUOC -> $RS_SAU, revision $REV_TRUOC -> $REV_SAU"
test "$RS_SAU" = "$RS_TRUOC" && test "$REV_SAU" = "$REV_TRUOC" \
  && echo 'PASS: sua template khi dang pause khong tao ReplicaSet moi va khong tao revision'

if kubectl rollout undo deployment/web -n lab-4a 2>~/lab-evidence/4a/b4-undo-khi-pause.txt; then
  echo 'FAIL: rollback duoc mot Deployment dang tam dung'
else
  echo 'PASS: khong the rollback mot Deployment dang tam dung'
fi

kubectl rollout resume deployment/web -n lab-4a
kubectl rollout status deployment/web -n lab-4a --timeout=180s
REV_RESUME="$(rev)"
test "$REV_RESUME" = "$((REV_TRUOC + 1))" \
  && echo 'PASS: hai thay doi gom lai thanh dung mot rollout khi resume'
```

**Ý nghĩa:** pause là cách gom nhiều chỉnh sửa vào một rollout duy nhất thay vì kích hoạt rollout sau
mỗi lệnh. Trong lúc pause, Kubernetes cũng không tính tiến trình so với deadline, nên bạn dừng giữa
chừng bao lâu cũng không sinh condition vượt deadline.

### B4.5. `revisionHistoryLimit` quyết định bạn còn rollback về đâu

```bash
kubectl patch deployment web -n lab-4a -p '{"spec":{"revisionHistoryLimit":1}}'

for n in 1 2 3; do
  kubectl set env deployment/web -n lab-4a BUILD="$n"
  kubectl rollout status deployment/web -n lab-4a --timeout=180s
done

for i in $(seq 1 30); do
  RS_COUNT="$(rscount)"
  test "$RS_COUNT" -le 2 && break
  sleep 2
done
kubectl get rs -n lab-4a -l app=web
kubectl rollout history deployment/web -n lab-4a | tee ~/lab-evidence/4a/b4-history-sau-gc.txt

echo "so ReplicaSet con lai = $RS_COUNT"
test "$RS_COUNT" -le 2 \
  && echo 'PASS: revisionHistoryLimit=1 chi giu 1 ReplicaSet cu ben canh ReplicaSet dang chay'
```

**Ý nghĩa:** lịch sử revision **được lưu trong chính các ReplicaSet cũ**. Xóa ReplicaSet nào là mất
khả năng rollback về revision đó; đặt `revisionHistoryLimit: 0` là mất hẳn khả năng rollback. Mặc
định giữ 10 ReplicaSet cũ. Việc dọn dẹp chỉ bắt đầu **sau khi** Deployment đạt trạng thái hoàn tất,
nên một Deployment cứ crash loop có thể có nhiều ReplicaSet hơn giới hạn.

**PASS:** toàn bộ dòng `PASS:` của B4 xuất hiện, không có dòng `FAIL:`.

## B5. Đo `maxSurge` và `maxUnavailable`

Phần này dùng một Deployment riêng để số liệu không lẫn với `web`. `minReadySeconds` được đặt tường
minh nhằm kéo dài rollout đủ để lấy mẫu — tổng thời gian rollout **phụ thuộc cấu hình** đó.

```bash
cat > ~/lab-work/4a/surge-demo.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: surge-demo
  namespace: lab-4a
  labels:
    app: surge-demo
spec:
  replicas: 4
  minReadySeconds: 12
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  selector:
    matchLabels:
      app: surge-demo
  template:
    metadata:
      labels:
        app: surge-demo
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 36000"]
EOF

kubectl apply -f ~/lab-work/4a/surge-demo.yaml
kubectl rollout status deployment/surge-demo -n lab-4a --timeout=300s
kubectl describe deployment surge-demo -n lab-4a \
  | grep -E 'StrategyType|MinReadySeconds|RollingUpdateStrategy' \
  | tee ~/lab-evidence/4a/b5-strategy-0-1.txt

grep -q '1 max unavailable, 0 max surge' ~/lab-evidence/4a/b5-strategy-0-1.txt \
  && echo 'PASS: chien luoc dang la maxUnavailable=1, maxSurge=0'
```

### B5.1. `maxSurge: 0`, `maxUnavailable: 1`

Hàm lấy mẫu dưới đây ghi ba con số mỗi vòng: `availableReplicas` của Deployment, tổng
`.spec.replicas` của **mọi** ReplicaSet thuộc Deployment, và số Pod thô. Nó chỉ dừng khi
`status.observedGeneration` đã bắt kịp `metadata.generation` — nghĩa là controller đã thật sự xử lý
thay đổi — và đồng thời `updatedReplicas`, `availableReplicas` cùng tổng replica đều bằng số mong
muốn. Không có điều kiện `observedGeneration` thì vòng lặp sẽ dừng ngay ở mẫu đầu tiên, khi status
còn là status của rollout **trước**.

```bash
lay_mau() {
  local d="$1" out="$2" want="$3"
  : > "$out"
  for i in $(seq 1 180); do
    GEN="$(kubectl get deploy "$d" -n lab-4a -o jsonpath='{.metadata.generation}')"
    OBS="$(kubectl get deploy "$d" -n lab-4a -o jsonpath='{.status.observedGeneration}')"
    AV="$(kubectl get deploy "$d" -n lab-4a -o jsonpath='{.status.availableReplicas}')"
    UPD="$(kubectl get deploy "$d" -n lab-4a -o jsonpath='{.status.updatedReplicas}')"
    SUM="$(kubectl get rs -n lab-4a -l app="$d" \
      -o jsonpath='{range .items[*]}{.spec.replicas}{"\n"}{end}' \
      | awk '{s+=$1} END {print s+0}')"
    PODS="$(kubectl get pods -n lab-4a -l app="$d" --no-headers | wc -l)"
    echo "${AV:-0} ${SUM:-0} ${PODS:-0}" >> "$out"
    if [ "$i" -ge 5 ] && [ "${OBS:-0}" = "${GEN:-x}" ] \
       && [ "${UPD:-0}" = "$want" ] && [ "${AV:-0}" = "$want" ] \
       && [ "${SUM:-0}" = "$want" ]; then
      break
    fi
    sleep 1
  done
  wc -l < "$out"
}

kubectl set image deployment/surge-demo -n lab-4a app=busybox:1.37
lay_mau surge-demo ~/lab-evidence/4a/b5-mau-surge0.txt 4
kubectl rollout status deployment/surge-demo -n lab-4a --timeout=300s
```

```bash
F=~/lab-evidence/4a/b5-mau-surge0.txt
MIN_AV="$(awk '{print $1}' "$F" | sort -n | head -1)"
MAX_SUM="$(awk '{print $2}' "$F" | sort -n | tail -1)"
MAX_PODS="$(awk '{print $3}' "$F" | sort -n | tail -1)"
printf 'so mau=%s  min availableReplicas=%s  max tong replica RS=%s  max Pod tho=%s\n' \
  "$(wc -l < "$F")" "$MIN_AV" "$MAX_SUM" "$MAX_PODS" \
  | tee -a ~/lab-evidence/4a/b5-ket-luan.txt

test "$MIN_AV" -ge 3 \
  && echo 'PASS: availableReplicas khong bao gio xuong duoi replicas - maxUnavailable = 4 - 1 = 3'
test "$MAX_SUM" -le 4 \
  && echo 'PASS: tong replica mong muon khong vuot replicas + maxSurge = 4 + 0 = 4'
```

### B5.2. `maxSurge: 1`, `maxUnavailable: 0`

Đổi tham số chiến lược **không** đụng tới `.spec.template`, nên nó không tự kích hoạt rollout — phải
đổi image để có rollout thứ hai.

```bash
kubectl patch deployment surge-demo -n lab-4a \
  -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
kubectl describe deployment surge-demo -n lab-4a | grep -E 'RollingUpdateStrategy' \
  | tee ~/lab-evidence/4a/b5-strategy-1-0.txt
grep -q '0 max unavailable, 1 max surge' ~/lab-evidence/4a/b5-strategy-1-0.txt \
  && echo 'PASS: chien luoc doi thanh maxUnavailable=0, maxSurge=1'

kubectl set image deployment/surge-demo -n lab-4a app=busybox:1.36
lay_mau surge-demo ~/lab-evidence/4a/b5-mau-surge1.txt 4
kubectl rollout status deployment/surge-demo -n lab-4a --timeout=300s
```

```bash
F=~/lab-evidence/4a/b5-mau-surge1.txt
MIN_AV="$(awk '{print $1}' "$F" | sort -n | head -1)"
MAX_SUM="$(awk '{print $2}' "$F" | sort -n | tail -1)"
MAX_PODS="$(awk '{print $3}' "$F" | sort -n | tail -1)"
printf 'so mau=%s  min availableReplicas=%s  max tong replica RS=%s  max Pod tho=%s\n' \
  "$(wc -l < "$F")" "$MIN_AV" "$MAX_SUM" "$MAX_PODS" \
  | tee -a ~/lab-evidence/4a/b5-ket-luan.txt

test "$MIN_AV" -ge 4 \
  && echo 'PASS: maxUnavailable=0 giu availableReplicas luon bang replicas trong suot rollout'
test "$MAX_SUM" -le 5 \
  && echo 'PASS: maxSurge=1 gioi han tong replica mong muon o replicas + 1 = 5'
```

**Ý nghĩa:** hai núm này quy định khoảng dao động của rollout, và hai cấu hình trên là hai cực của
khoảng đó. `maxSurge: 0` nghĩa là không được mượn thêm chỗ, nên phải hạ bớt Pod cũ trước —
availability giảm. `maxUnavailable: 0` nghĩa là không được giảm availability, nên phải tạo dư Pod
trước — tốn thêm tài nguyên. Mặc định là 25% cho cả hai: `maxUnavailable` làm tròn **xuống**,
`maxSurge` làm tròn **lên**.

Cột `max Pod tho` trong hai file mẫu có thể **lớn hơn** `replicas + maxSurge`. Đó không phải lỗi:
bài 63 nói rõ Kubernetes **không tính Pod đang kết thúc** khi tính `availableReplicas`, nên số Pod
thật trên cluster còn cao hơn cho tới khi `terminationGracePeriodSeconds` của các Pod đó hết hạn. Vì
vậy gate của lab đặt trên `.spec.replicas` của các ReplicaSet và trên `availableReplicas` — đúng hai
đại lượng mà controller thực sự điều khiển — chứ không đặt trên số Pod đếm bằng mắt.

**PASS:** sáu dòng `PASS:` của B5 xuất hiện.

## B6. `Recreate` so với `RollingUpdate`

Deployment dưới đây có `preStop` ngủ một lúc và `terminationGracePeriodSeconds` đủ dài, nên khoảng
"không còn Pod nào" của `Recreate` chắc chắn bị lấy mẫu bắt được. Hook vòng đời container đã học ở
[giai đoạn 2](../00-ALO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime).

```bash
cat > ~/lab-work/4a/recreate-demo.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recreate-demo
  namespace: lab-4a
  labels:
    app: recreate-demo
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: recreate-demo
  template:
    metadata:
      labels:
        app: recreate-demo
    spec:
      terminationGracePeriodSeconds: 40
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 36000"]
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 15"]
EOF

kubectl apply -f ~/lab-work/4a/recreate-demo.yaml
kubectl rollout status deployment/recreate-demo -n lab-4a --timeout=300s
kubectl describe deployment recreate-demo -n lab-4a | grep -E 'StrategyType' \
  | tee ~/lab-evidence/4a/b6-strategy.txt
grep -q 'Recreate' ~/lab-evidence/4a/b6-strategy.txt \
  && echo 'PASS: StrategyType la Recreate'
```

```bash
kubectl set image deployment/recreate-demo -n lab-4a app=busybox:1.37
lay_mau recreate-demo ~/lab-evidence/4a/b6-mau-recreate.txt 3
kubectl rollout status deployment/recreate-demo -n lab-4a --timeout=300s

F=~/lab-evidence/4a/b6-mau-recreate.txt
MIN_AV="$(awk '{print $1}' "$F" | sort -n | head -1)"
printf 'so mau=%s  min availableReplicas trong rollout Recreate=%s\n' \
  "$(wc -l < "$F")" "$MIN_AV" | tee -a ~/lab-evidence/4a/b6-ket-luan.txt

test "$MIN_AV" = '0' \
  && echo 'PASS: Recreate ha availableReplicas xuong 0 truoc khi tao Pod moi'
```

So sánh trực tiếp với số đo của B5 trên cùng cluster:

```bash
grep -h '' ~/lab-evidence/4a/b5-ket-luan.txt ~/lab-evidence/4a/b6-ket-luan.txt
```

Bảo đảm của `Recreate` **chỉ áp dụng cho lần nâng cấp**, không áp dụng khi bạn xóa tay một Pod:

```bash
VICTIM="$(kubectl get pods -n lab-4a -l app=recreate-demo -o name | head -1)"
kubectl delete "$VICTIM" -n lab-4a --wait=false

for i in $(seq 1 60); do
  P="$(kubectl get pods -n lab-4a -l app=recreate-demo --no-headers | wc -l)"
  test "$P" -ge 4 && break
  sleep 1
done
kubectl get pods -n lab-4a -l app=recreate-demo | tee ~/lab-evidence/4a/b6-xoa-tay.txt
echo "so Pod trong luc Pod cu con Terminating = $P"

test "$P" -ge 4 \
  && echo 'PASS: Pod thay the duoc tao ngay trong khi Pod cu van dang Terminating'
kubectl rollout status deployment/recreate-demo -n lab-4a --timeout=300s
```

**Ý nghĩa:** `Recreate` kill toàn bộ Pod cũ và **chờ gỡ bỏ xong** rồi mới tạo Pod của revision mới —
nghĩa là có một khoảng không ai phục vụ. `RollingUpdate` thì luôn giữ ít nhất
`replicas - maxUnavailable` Pod sẵn sàng, như B5 vừa đo. Nhưng nếu bạn **xóa tay** một Pod thì vòng
đời của nó do ReplicaSet kiểm soát, và Pod thay thế được tạo ngay lập tức, kể cả khi Pod cũ vẫn đang
`Terminating`. Muốn bảo đảm "tối đa bao nhiêu Pod" thật sự thì bài 63 chỉ sang StatefulSet — nội
dung của [nhóm 4b](../00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling).

Dọn hai Deployment đo đạc:

```bash
kubectl delete deployment surge-demo recreate-demo -n lab-4a --wait=true
kubectl get deployment -n lab-4a
```

**PASS:** ba dòng `PASS:` của B6 xuất hiện; sau khi dọn chỉ còn Deployment `web`.

## B7. Rollout thất bại rồi undo

**Fault injection, toàn bộ Pod của phần này ghim vào `lab-k8s-worker2`.**

```bash
cat > ~/lab-work/4a/fail-demo.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fail-demo
  namespace: lab-4a
  labels:
    app: fail-demo
spec:
  replicas: 3
  progressDeadlineSeconds: 60
  selector:
    matchLabels:
      app: fail-demo
  template:
    metadata:
      labels:
        app: fail-demo
    spec:
      nodeName: lab-k8s-worker2
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 36000"]
EOF

kubectl apply -f ~/lab-work/4a/fail-demo.yaml
kubectl rollout status deployment/fail-demo -n lab-4a --timeout=300s
kubectl get pods -n lab-4a -l app=fail-demo -o wide

NODES="$(kubectl get pods -n lab-4a -l app=fail-demo \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u)"
test "$NODES" = 'lab-k8s-worker2' \
  && echo 'PASS: toan bo Pod cua fail-demo nam tren lab-k8s-worker2'
```

### B7.1. Đẩy một image không tồn tại

```bash
kubectl set image deployment/fail-demo -n lab-4a app=busybox:khong-ton-tai

if kubectl rollout status deployment/fail-demo -n lab-4a --timeout=150s; then
  echo 'FAIL: rollout status bao thanh cong voi image khong ton tai'
else
  echo 'PASS: rollout status tra ma thoat khac 0 khi rollout khong tien trien'
fi
```

### B7.2. Đọc trạng thái kẹt

```bash
COLS='NAME:.metadata.name,DESIRED:.spec.replicas,READY:.status.readyReplicas,IMAGE:.spec.template.spec.containers[0].image'
kubectl get rs -n lab-4a -l app=fail-demo -o "custom-columns=$COLS" \
  | tee ~/lab-evidence/4a/b7-rs-ket.txt
kubectl get pods -n lab-4a -l app=fail-demo
kubectl describe deployment fail-demo -n lab-4a | tee ~/lab-evidence/4a/b7-describe.txt >/dev/null

AV="$(kubectl get deploy fail-demo -n lab-4a -o jsonpath='{.status.availableReplicas}')"
OLD="$(awk '$4=="busybox:1.36"{print $2}' ~/lab-evidence/4a/b7-rs-ket.txt)"
NEW="$(awk '$4=="busybox:khong-ton-tai"{print $2}' ~/lab-evidence/4a/b7-rs-ket.txt)"
BAD="$(kubectl get pods -n lab-4a -l app=fail-demo \
  -o jsonpath='{range .items[*]}{.status.containerStatuses[0].state.waiting.reason}{"\n"}{end}' \
  | grep -c -E 'ImagePullBackOff|ErrImagePull')"

printf 'availableReplicas=%s  RS cu DESIRED=%s  RS moi DESIRED=%s  Pod ket=%s\n' \
  "$AV" "$OLD" "$NEW" "$BAD" | tee ~/lab-evidence/4a/b7-so-lieu.txt

test "${AV:-0}" = '3' && echo 'PASS: ba Pod cu van dang phuc vu, availableReplicas khong tut'
test "$OLD" = '3' && test "$NEW" = '1' \
  && echo 'PASS: ReplicaSet cu giu nguyen 3, ReplicaSet moi dung lai o 1'
test "$BAD" -ge 1 && echo 'PASS: Pod moi ket o ImagePullBackOff/ErrImagePull'
```

**Ý nghĩa:** với `replicas: 3` và tham số mặc định, `maxUnavailable` là 25% làm tròn **xuống** = 0
Pod, còn `maxSurge` là 25% làm tròn **lên** = 1 Pod. Vì thế Deployment controller không được phép hạ
ReplicaSet cũ một Pod nào, và chỉ dám tạo đúng 1 Pod mới. Đó là lý do một tag gõ sai không kéo sập
dịch vụ — con số cứu bạn chính là `maxUnavailable`.

### B7.3. Deadline bị vượt và Kubernetes không tự sửa

```bash
for i in $(seq 1 40); do
  ST="$(kubectl get deploy fail-demo -n lab-4a \
    -o jsonpath='{.status.conditions[?(@.type=="Progressing")].status}')"
  RE="$(kubectl get deploy fail-demo -n lab-4a \
    -o jsonpath='{.status.conditions[?(@.type=="Progressing")].reason}')"
  test "$RE" = 'ProgressDeadlineExceeded' && break
  sleep 5
done
kubectl get deploy fail-demo -n lab-4a -o jsonpath='{.status.conditions}' \
  | tee ~/lab-evidence/4a/b7-conditions.txt; echo

IMG="$(kubectl get deployment fail-demo -n lab-4a \
  -o jsonpath='{.spec.template.spec.containers[0].image}')"
echo "Progressing=$ST reason=$RE image trong spec=$IMG"

test "$ST" = 'False' && test "$RE" = 'ProgressDeadlineExceeded' \
  && echo 'PASS: condition Progressing chuyen False voi reason ProgressDeadlineExceeded'
test "$IMG" = 'busybox:khong-ton-tai' \
  && echo 'PASS: Kubernetes KHONG tu rollback, spec van giu image hong'
```

**Ý nghĩa:** `progressDeadlineSeconds` chỉ là **đồng hồ báo**. Khi vượt deadline, Kubernetes đặt
condition `Progressing = False` với `reason: ProgressDeadlineExceeded` và **không làm gì thêm**.
Việc rollback là quyết định của bạn hoặc của một bộ điều phối cấp cao hơn. Thời điểm condition xuất
hiện phụ thuộc đúng giá trị `progressDeadlineSeconds` bạn đặt trong manifest.

### B7.4. Undo để thoát

```bash
kubectl rollout undo deployment/fail-demo -n lab-4a
kubectl rollout status deployment/fail-demo -n lab-4a --timeout=300s

IMG="$(kubectl get deployment fail-demo -n lab-4a \
  -o jsonpath='{.spec.template.spec.containers[0].image}')"
AV="$(kubectl get deploy fail-demo -n lab-4a -o jsonpath='{.status.availableReplicas}')"
BAD="$(kubectl get pods -n lab-4a -l app=fail-demo \
  -o jsonpath='{range .items[*]}{.status.containerStatuses[0].state.waiting.reason}{"\n"}{end}' \
  | grep -c -E 'ImagePullBackOff|ErrImagePull')"

echo "image=$IMG availableReplicas=$AV Pod ket=$BAD"
test "$IMG" = 'busybox:1.36' && test "${AV:-0}" = '3' && test "$BAD" = '0' \
  && echo 'PASS: undo dua Deployment ve revision on dinh, khong con Pod ket'

kubectl delete deployment fail-demo -n lab-4a --wait=true
```

**PASS:** toàn bộ dòng `PASS:` của B7 xuất hiện, không có dòng `FAIL:`.

## B8. Xóa theo tầng: foreground, background và orphan

Bài [260](../260-use-cascading-deletion-vi.md) yêu cầu tạo lại Deployment cho **từng** kiểu xóa.
Deployment dưới đây có `preStop` để trạng thái trung gian quan sát được một cách xác định.

```bash
cat > ~/lab-work/4a/cascade-demo.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cascade-demo
  namespace: lab-4a
  labels:
    app: cascade-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cascade-demo
  template:
    metadata:
      labels:
        app: cascade-demo
    spec:
      terminationGracePeriodSeconds: 40
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 36000"]
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 15"]
EOF

tao_lai() {
  kubectl apply -f ~/lab-work/4a/cascade-demo.yaml >/dev/null
  kubectl rollout status deployment/cascade-demo -n lab-4a --timeout=300s >/dev/null
}
cho_sach() {
  for i in $(seq 1 90); do
    N="$(kubectl get deployment,rs,pods -n lab-4a -l app=cascade-demo --no-headers 2>/dev/null | wc -l)"
    test "$N" = '0' && break
    sleep 2
  done
  echo "$N"
}

tao_lai
kubectl get pods -n lab-4a -l app=cascade-demo \
  -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.metadata.ownerReferences[0].kind}{"/"}{.metadata.ownerReferences[0].name}{"\n"}{end}' \
  | tee ~/lab-evidence/4a/b8-ownerrefs.txt
grep -q 'ReplicaSet/cascade-demo-' ~/lab-evidence/4a/b8-ownerrefs.txt \
  && echo 'PASS: ownerReferences co mat tren cac Pod truoc khi thu xoa theo tang'
```

### B8.1. Foreground — owner chờ dependent

```bash
kubectl delete deployment cascade-demo -n lab-4a --cascade=foreground --wait=false

FIN=''
for i in $(seq 1 30); do
  FIN="$(kubectl get deployment cascade-demo -n lab-4a \
    -o jsonpath='{.metadata.finalizers[*]}' 2>/dev/null)"
  test -n "$FIN" && break
  sleep 1
done
DTS="$(kubectl get deployment cascade-demo -n lab-4a \
  -o jsonpath='{.metadata.deletionTimestamp}' 2>/dev/null)"
echo "finalizers=$FIN deletionTimestamp=$DTS"

case "$FIN" in
  *foregroundDeletion*) echo 'PASS: Deployment duoc giu lai boi finalizer foregroundDeletion' ;;
  *) echo 'FAIL: khong quan sat duoc foregroundDeletion' ;;
esac

REMAIN="$(cho_sach)"
test "$REMAIN" = '0' && echo 'PASS: foreground hoan tat, khong con Deployment/ReplicaSet/Pod'
```

### B8.2. Background — owner biến mất trước

```bash
tao_lai
kubectl delete deployment cascade-demo -n lab-4a --cascade=background --wait=true

DEP="$(kubectl get deployment cascade-demo -n lab-4a --no-headers 2>/dev/null | wc -l)"
PODS="$(kubectl get pods -n lab-4a -l app=cascade-demo --no-headers 2>/dev/null | wc -l)"
echo "ngay sau delete: Deployment=$DEP, Pod con lai=$PODS"

test "$DEP" = '0' && test "$PODS" -gt 0 \
  && echo 'PASS: background xoa Deployment truoc, Pod van con lai de GC don o nen'
REMAIN="$(cho_sach)"
test "$REMAIN" = '0' && echo 'PASS: garbage collector don sach ReplicaSet va Pod o nen'
```

### B8.3. Orphan — dependent sống sót

```bash
tao_lai
RS_CD="$(kubectl get rs -n lab-4a -l app=cascade-demo -o jsonpath='{.items[0].metadata.name}')"
kubectl delete deployment cascade-demo -n lab-4a --cascade=orphan --wait=true

for i in $(seq 1 60); do
  RS_REFS="$(kubectl get rs "$RS_CD" -n lab-4a \
    -o jsonpath='{.metadata.ownerReferences}' 2>/dev/null | tr -d '[:space:]')"
  test -z "$RS_REFS" && break
  sleep 2
done
DEP="$(kubectl get deployment cascade-demo -n lab-4a --no-headers 2>/dev/null | wc -l)"
PODS="$(kubectl get pods -n lab-4a -l app=cascade-demo --no-headers 2>/dev/null | wc -l)"
kubectl get rs "$RS_CD" -n lab-4a -o yaml | tee ~/lab-evidence/4a/b8-rs-mo-coi.yaml >/dev/null
echo "Deployment=$DEP  Pod=$PODS  ownerReferences cua ReplicaSet=[$RS_REFS]"

test "$DEP" = '0' && test "$PODS" = '2' && test -z "$RS_REFS" \
  && echo 'PASS: orphan xoa Deployment nhung giu ReplicaSet va Pod, va go ownerReferences'

kubectl delete rs "$RS_CD" -n lab-4a --wait=true
REMAIN="$(cho_sach)"
test "$REMAIN" = '0' && echo 'PASS: don sach phan B8'
```

**Ý nghĩa:** background là **mặc định**. Foreground giữ owner lại bằng finalizer `foregroundDeletion`
cho tới khi dependent xóa xong, nên hữu ích khi bạn cần biết chắc mọi thứ đã biến mất trước khi làm
bước tiếp theo. Orphan là con dao mổ: nó cắt Deployment ra khỏi ReplicaSet đang chạy — chính thao tác
mà B1.7 dùng để chứng minh ReplicaSet không tự cập nhật Pod.

**PASS:** toàn bộ dòng `PASS:` của B8 xuất hiện, không có dòng `FAIL:`.

## B9. Vận hành hằng ngày bằng kubectl

### B9.1. Gộp nhiều tài nguyên vào một file và thứ tự tạo

```bash
mkdir -p ~/lab-work/4a/stack/config ~/lab-work/4a/stack/workload

cat > ~/lab-work/4a/stack/config/app-config.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: lab-4a
data:
  greeting: xin-chao
EOF

cat > ~/lab-work/4a/stack/workload/shop.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: shop-config
  namespace: lab-4a
data:
  tier: shop
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop
  namespace: lab-4a
  labels:
    app: shop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shop
  template:
    metadata:
      labels:
        app: shop
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 36000"]
        envFrom:
        - configMapRef:
            name: shop-config
EOF
```

Thiếu `-R` thì thao tác dừng ở cấp thư mục đầu tiên:

```bash
if kubectl apply -f ~/lab-work/4a/stack 2>~/lab-evidence/4a/b9-loi-thieu-R.txt; then
  echo 'FAIL: apply thu muc long nhau chay duoc du khong co -R'
else
  cat ~/lab-evidence/4a/b9-loi-thieu-R.txt
  echo 'PASS: apply dung o cap thu muc dau tien khi khong co -R'
fi

kubectl apply -R -f ~/lab-work/4a/stack | tee ~/lab-evidence/4a/b9-apply-R.txt
kubectl rollout status deployment/shop -n lab-4a --timeout=180s

grep -q 'configmap/shop-config' ~/lab-evidence/4a/b9-apply-R.txt \
  && grep -q 'deployment.apps/shop' ~/lab-evidence/4a/b9-apply-R.txt \
  && echo 'PASS: -R xu ly ca thu muc con'

CM_LINE="$(grep -n 'configmap/shop-config' ~/lab-evidence/4a/b9-apply-R.txt | cut -d: -f1)"
DP_LINE="$(grep -n 'deployment.apps/shop' ~/lab-evidence/4a/b9-apply-R.txt | cut -d: -f1)"
echo "ConfigMap o dong $CM_LINE, Deployment o dong $DP_LINE"
test "$CM_LINE" -lt "$DP_LINE" \
  && echo 'PASS: tai nguyen duoc tao theo dung thu tu xuat hien trong manifest'
```

**Ý nghĩa:** trong một file, các tài nguyên được tạo theo đúng thứ tự chúng xuất hiện, phân tách bằng
`---`. Bài 61 khuyên khai báo Service trước Deployment vì lý do này — Service thuộc giai đoạn 5, ở
đây ConfigMap đóng vai "thứ phải có trước". Còn `--recursive`/`-R` hoạt động với mọi thao tác nhận
`-f`: `create`, `get`, `delete`, `describe`, kể cả `rollout`.

### B9.2. `last-applied-configuration`, `kubectl diff` và apply ghi đè scale thủ công

```bash
kubectl get deployment shop -n lab-4a \
  -o jsonpath="{.metadata.annotations['kubectl\.kubernetes\.io/last-applied-configuration']}" \
  | tee ~/lab-evidence/4a/b9-last-applied.json; echo

grep -q 'shop' ~/lab-evidence/4a/b9-last-applied.json \
  && echo 'PASS: kubectl apply ghi annotation last-applied-configuration'

kubectl scale deployment/shop -n lab-4a --replicas=5
kubectl rollout status deployment/shop -n lab-4a --timeout=180s
kubectl get deployment shop -n lab-4a \
  -o jsonpath="{.metadata.annotations['kubectl\.kubernetes\.io/last-applied-configuration']}" \
  | grep -q '"replicas":5' \
  && echo 'FAIL: scale ghi vao last-applied-configuration' \
  || echo 'PASS: kubectl scale khong ghi vao last-applied-configuration'
```

`kubectl diff` cho biết trước điều gì sẽ xảy ra, và trả mã thoát khác 0 khi có khác biệt:

```bash
if kubectl diff -f ~/lab-work/4a/stack/workload/shop.yaml \
     > ~/lab-evidence/4a/b9-diff.txt 2>&1; then
  echo 'FAIL: diff bao khong co khac biet du replicas dang la 5'
else
  grep -E '^[-+].*replicas' ~/lab-evidence/4a/b9-diff.txt
  echo 'PASS: kubectl diff phat hien khac biet va tra ma thoat khac 0'
fi

kubectl apply -f ~/lab-work/4a/stack/workload/shop.yaml
kubectl rollout status deployment/shop -n lab-4a --timeout=180s
R="$(kubectl get deployment shop -n lab-4a -o jsonpath='{.spec.replicas}')"
echo "replicas sau khi apply lai manifest = $R"
test "$R" = '2' && echo 'PASS: apply manifest co replicas ghi de thao tac scale thu cong'
```

**Ý nghĩa:** `kubectl apply` so file cấu hình, cấu hình thực tế và annotation
`last-applied-configuration` để tính ra patch. Vì manifest của bạn **có** khai `replicas`, apply lại
sẽ kéo số replica về giá trị trong file — đúng cảnh báo của bài 63. Nếu một autoscaler đang quản
việc scale thì bài 63 dặn **đừng** đặt `.spec.replicas` trong manifest.

### B9.3. Strategic merge patch so với JSON merge patch

```bash
cat > ~/lab-work/4a/patch-them-container.yaml <<'EOF'
spec:
  template:
    spec:
      containers:
      - name: sidecar
        image: busybox:1.36
        command: ["sh", "-c", "sleep 36000"]
EOF

kubectl patch deployment shop -n lab-4a --patch-file ~/lab-work/4a/patch-them-container.yaml
kubectl rollout status deployment/shop -n lab-4a --timeout=180s
N="$(kubectl get deployment shop -n lab-4a \
  -o jsonpath='{.spec.template.spec.containers[*].name}' | wc -w)"
kubectl get deployment shop -n lab-4a -o jsonpath='{.spec.template.spec.containers[*].name}'; echo
test "$N" = '2' \
  && echo 'PASS: strategic merge patch TRON container moi vao list dang co'
```

```bash
kubectl patch deployment shop -n lab-4a --type merge \
  --patch-file ~/lab-work/4a/patch-them-container.yaml
kubectl rollout status deployment/shop -n lab-4a --timeout=180s
N="$(kubectl get deployment shop -n lab-4a \
  -o jsonpath='{.spec.template.spec.containers[*].name}' | wc -w)"
kubectl get deployment shop -n lab-4a -o jsonpath='{.spec.template.spec.containers[*].name}'; echo
test "$N" = '1' \
  && echo 'PASS: JSON merge patch THAY THE toan bo list containers'
```

**Ý nghĩa:** cùng một file patch, hai kết quả trái ngược. Mặc định `--type` là `strategic`, và với
`strategic` thì một list được trộn hay bị thay tùy `patchStrategy` khai trong mã nguồn Kubernetes:
`containers` có `patchStrategy: merge` với `patchMergeKey: name` nên được trộn theo tên container.
Với `--type merge` (JSON merge patch, RFC 7386) thì muốn cập nhật một list là phải viết lại toàn bộ
list, và list mới thay hoàn toàn list cũ.

### B9.4. `replace --force` là cập nhật gây gián đoạn

```bash
UID_TRUOC="$(kubectl get deployment shop -n lab-4a -o jsonpath='{.metadata.uid}')"
kubectl replace -f ~/lab-work/4a/stack/workload/shop.yaml --force \
  | tee ~/lab-evidence/4a/b9-replace-force.txt
kubectl rollout status deployment/shop -n lab-4a --timeout=180s
UID_SAU="$(kubectl get deployment shop -n lab-4a -o jsonpath='{.metadata.uid}')"

echo "uid: $UID_TRUOC -> $UID_SAU"
grep -q 'deleted' ~/lab-evidence/4a/b9-replace-force.txt \
  && grep -q 'replaced' ~/lab-evidence/4a/b9-replace-force.txt \
  && echo 'PASS: replace --force in ra deleted roi replaced'
test "$UID_TRUOC" != "$UID_SAU" \
  && echo 'PASS: UID doi, nghia la object bi xoa va tao lai chu khong duoc va tai cho'
```

**Ý nghĩa:** `apply` chỉ áp phần thay đổi và không ghi đè thứ bạn không chỉ định; `replace --force`
**xóa rồi tạo lại**. UID mới là bằng chứng không cãi được. Chỉ dùng nó khi phải sửa field không cập
nhật được sau khi khởi tạo — ví dụ `.spec.selector` của Deployment vốn bất biến.

### B9.5. Canary thủ công và thao tác hàng loạt bằng label

```bash
cat > ~/lab-work/4a/canary.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-stable
  namespace: lab-4a
  labels:
    app: guestbook
    tier: frontend
    track: stable
spec:
  replicas: 3
  selector:
    matchLabels:
      app: guestbook
      tier: frontend
      track: stable
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
        track: stable
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 36000"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-canary
  namespace: lab-4a
  labels:
    app: guestbook
    tier: frontend
    track: canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: guestbook
      tier: frontend
      track: canary
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
        track: canary
    spec:
      containers:
      - name: app
        image: busybox:1.37
        command: ["sh", "-c", "sleep 36000"]
EOF

kubectl apply -f ~/lab-work/4a/canary.yaml
kubectl rollout status deployment/frontend-stable -n lab-4a --timeout=180s
kubectl rollout status deployment/frontend-canary -n lab-4a --timeout=180s
kubectl get pods -n lab-4a -l app=guestbook,tier=frontend --show-labels \
  | tee ~/lab-evidence/4a/b9-canary-pods.txt
```

```bash
CHUNG="$(kubectl get pods -n lab-4a -l app=guestbook,tier=frontend --no-headers | wc -l)"
STABLE="$(kubectl get pods -n lab-4a -l app=guestbook,tier=frontend,track=stable --no-headers | wc -l)"
CANARY="$(kubectl get pods -n lab-4a -l app=guestbook,tier=frontend,track=canary --no-headers | wc -l)"
printf 'selector chung bat %s Pod, trong do stable=%s canary=%s\n' "$CHUNG" "$STABLE" "$CANARY" \
  | tee ~/lab-evidence/4a/b9-canary-ty-le.txt

test "$CHUNG" = '4' && test "$STABLE" = '3' && test "$CANARY" = '1' \
  && echo 'PASS: bo label chung bat ca hai ban phat hanh, ty le replica la 3:1'
```

Thao tác hàng loạt: xóa cả hai Deployment bằng một selector thay vì liệt kê tên.

```bash
kubectl delete deployment -n lab-4a -l app=guestbook,tier=frontend
for i in $(seq 1 60); do
  N="$(kubectl get pods -n lab-4a -l app=guestbook --no-headers 2>/dev/null | wc -l)"
  test "$N" = '0' && break
  sleep 2
done
test "$N" = '0' && echo 'PASS: xoa hang loat bang -l don sach ca hai ban phat hanh'
```

**Ý nghĩa:** hai bản phát hành khác nhau ở **đúng một label riêng** (`track`), còn bộ label chung
(`app`, `tier`) giữ nguyên để một selector duy nhất phủ được cả hai tập Pod. Tỷ lệ lưu lượng do **số
replica** quyết định — 3 và 1 là 3:1. Phần còn thiếu ở đây là một Service dùng đúng selector chung để
thực sự chia lưu lượng; Service thuộc giai đoạn 5, nên lab này chỉ đo phần tập Pod.

Dọn phần B9:

```bash
kubectl delete deployment shop -n lab-4a --wait=true
kubectl delete configmap shop-config app-config -n lab-4a --ignore-not-found=true
kubectl get deployment,configmap -n lab-4a
```

**PASS:** toàn bộ dòng `PASS:` của B9 xuất hiện, không có dòng `FAIL:`.

## B10. ReplicationController: nhận diện, không dựng

Bài [70](../70-replicationcontroller-vi.md) là tài liệu lịch sử. Lab **không tạo** `rc` nào; mục
tiêu duy nhất là nhận ra nó khi tiếp quản cluster cũ và biết nó khác ReplicaSet ở đâu.

```bash
kubectl api-resources | grep -E '^(replicationcontrollers|replicasets)' \
  | tee ~/lab-evidence/4a/b10-api-resources.txt

kubectl explain replicaset.spec.selector > ~/lab-evidence/4a/b10-rs-selector.txt
kubectl explain replicationcontroller.spec.selector > ~/lab-evidence/4a/b10-rc-selector.txt
cat ~/lab-evidence/4a/b10-rc-selector.txt
```

```bash
grep -q '^replicationcontrollers' ~/lab-evidence/4a/b10-api-resources.txt \
  && echo 'PASS: API core v1 van con ReplicationController, viet tat la rc'
grep -q 'matchExpressions' ~/lab-evidence/4a/b10-rs-selector.txt \
  && echo 'PASS: ReplicaSet co matchLabels va matchExpressions'
if grep -q 'matchExpressions' ~/lab-evidence/4a/b10-rc-selector.txt; then
  echo 'FAIL: rc.spec.selector co matchExpressions'
else
  echo 'PASS: rc.spec.selector la map phang, khong ho tro selector dua tren tap hop'
fi
test "$(kubectl get rc -n lab-4a --no-headers 2>/dev/null | wc -l)" = '0' \
  && echo 'PASS: lab khong tao ReplicationController nao'
```

**Ý nghĩa:** ReplicationController và ReplicaSet phục vụ cùng một mục đích và hành xử tương tự nhau;
khác biệt kỹ thuật đáng nhớ chỉ có một — `rc` **không hỗ trợ selector dựa trên tập hợp**. Khoảng cách
thật sự nằm ở tầng trên: `rc` chỉ có rolling update **thủ công** (tạo `rc` mới 1 replica, rồi +1/−1
từng bước, rồi xóa `rc` cũ), còn Deployment làm việc đó bằng một lệnh `kubectl set image` cộng
`rollout status` và `rollout undo` — đúng thứ bạn vừa chạy ở B4. Gặp `rc` trên cluster kế thừa thì
hướng xử lý là di trú sang Deployment, không phải học quy trình thủ công đó.

**PASS:** bốn dòng `PASS:` của B10 xuất hiện, không có dòng `FAIL:`.

## B11. Cleanup và gate cuối

**Mục đích:** xóa mọi object lab tạo ra và chứng minh cluster trở về `01-cluster-ready`.

```bash
kubectl delete namespace lab-4a --wait=true --timeout=300s

if kubectl get namespace lab-4a >/dev/null 2>&1; then
  echo 'FAIL: namespace lab-4a van ton tai'
else
  echo 'PASS: namespace lab-4a da bien mat'
fi

rm -f ~/lab-work/4a/pod-tran.yaml ~/lab-work/4a/rs-frontend.yaml \
      ~/lab-work/4a/rs-frontend-v2.yaml ~/lab-work/4a/rs-sai-selector.yaml \
      ~/lab-work/4a/rs-sai-restart.yaml ~/lab-work/4a/web.yaml \
      ~/lab-work/4a/surge-demo.yaml ~/lab-work/4a/recreate-demo.yaml \
      ~/lab-work/4a/fail-demo.yaml ~/lab-work/4a/cascade-demo.yaml \
      ~/lab-work/4a/patch-them-container.yaml ~/lab-work/4a/canary.yaml
rm -f ~/lab-work/4a/stack/config/app-config.yaml ~/lab-work/4a/stack/workload/shop.yaml
rmdir ~/lab-work/4a/stack/config ~/lab-work/4a/stack/workload ~/lab-work/4a/stack
rmdir ~/lab-work/4a
test ! -e ~/lab-work/4a && echo 'PASS: manifest tam da duoc xoa het'
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu còn file lạ, và `test ! -e` biến điều đó thành gate thay
vì im lặng bỏ qua. Bằng chứng ở `~/lab-evidence/4a/` được giữ lại.

Gate cuối, chạy trên `lab-k8s-master`:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get nodes
kubectl get pods -n default
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get deployment,replicaset,pods -A | grep -E 'lab-4a' \
  && echo 'FAIL: con object cua lab-4a' \
  || echo 'PASS: khong con object nao cua lab-4a tren cluster'
```

**PASS:** không có dòng `FAIL:` nào; ba node `Ready`; `PASS: readyz ok` xuất hiện; namespace
`default` không có Pod; lệnh field selector trả `No resources found`; CoreDNS đủ replica. Cluster trở
về `01-cluster-ready`; **không** chụp snapshot mới.

---

## 3. Checkpoint 4a

Tự trả lời không nhìn tài liệu:

- [ ] Kể đúng ba thứ định nghĩa một ReplicaSet và mô tả vòng lặp của nó bằng ngôn ngữ của mô hình
      điều khiển đã học ở giai đoạn 1.
- [ ] Giải thích vì sao một Pod trần bạn tạo tay lại bị ReplicaSet nuốt mất, và vì sao điều đó
      **không** xảy ra với ReplicaSet do một Deployment tạo ra.
- [ ] Chỉ ra hai thứ khác nhau mà selector và `ownerReferences` trả lời, và vì sao `ownerReferences`
      phải mang UID chứ không chỉ mang tên.
- [ ] Nói được điều gì xảy ra khi bạn xóa một ReplicaSet bằng `--cascade=orphan` rồi tạo một
      ReplicaSet mới cùng selector với image mới — và vì sao đó chính là lý do Deployment tồn tại.
- [ ] Vẽ chuỗi Pod → ReplicaSet → Deployment và nói `pod-template-hash` được sinh ra từ đâu, gắn vào
      những chỗ nào, giải quyết vấn đề gì.
- [ ] Phân biệt thao tác nào tạo revision và thao tác nào không; giải thích vì sao `kubectl scale`
      thuộc nhóm còn lại.
- [ ] Với `replicas: 4`, `maxSurge: 1`, `maxUnavailable: 0`, nói khoảng dao động của tổng replica và
      của `availableReplicas`; rồi làm lại với `maxSurge: 0`, `maxUnavailable: 1`. Nêu quy tắc làm
      tròn của hai tham số khi khai bằng phần trăm.
- [ ] Giải thích vì sao số Pod đếm được trong lúc rollout có thể vượt `replicas + maxSurge` mà vẫn
      không sai.
- [ ] So sánh `Recreate` với `RollingUpdate` bằng chính số đo bạn lấy ở B5 và B6, và nói vì sao bảo
      đảm của `Recreate` không áp dụng khi bạn xóa tay một Pod.
- [ ] Kể lại một rollout hỏng: trường nào giữ cho dịch vụ không sập, condition nào đổi giá trị, và
      Kubernetes làm gì sau khi vượt `progressDeadlineSeconds`.
- [ ] Chọn đúng propagation policy cho ba tình huống: xóa gọn, xóa nhưng phải chắc chắn đã xong, và
      xóa owner mà giữ workload đang chạy.
- [ ] Nói khác biệt kỹ thuật giữa `ReplicationController` và `ReplicaSet`, và hướng xử lý khi tiếp
      quản một cluster còn `rc`.

### Bài giải thích cuối cùng

Trong tối đa 10 phút, kể lại luồng sau mà không mở tài liệu:

1. Bạn `kubectl apply` một Deployment. Deployment controller băm pod template, tạo một ReplicaSet tên
   `<deployment>-<hash>`, gắn `pod-template-hash` vào ReplicaSet, vào selector của nó và vào Pod.
2. ReplicaSet controller so số Pod khớp selector với `.spec.replicas`, tạo bù cho đủ, và ghi
   `ownerReferences` lên từng Pod.
3. Bạn đổi image. Vì `.spec.template` đổi, một revision mới ra đời cùng một ReplicaSet mới; controller
   scale ReplicaSet mới lên và ReplicaSet cũ về 0, luôn nằm trong khoảng do `maxSurge` và
   `maxUnavailable` quy định.
4. ReplicaSet cũ ở lại với `DESIRED = 0` — đó chính là chỗ lưu revision cũ, và `revisionHistoryLimit`
   quyết định bạn còn lùi được tới đâu.
5. Image sai làm rollout kẹt. `maxUnavailable` giữ ReplicaSet cũ nguyên vẹn nên dịch vụ vẫn chạy;
   quá deadline thì condition `Progressing` chuyển `False`, và Kubernetes dừng lại ở đó.
6. Bạn `kubectl rollout undo`. Đó là một rollout mới, dùng lại ReplicaSet cũ, revision tăng tiếp.
7. Bạn xóa Deployment. Propagation policy quyết định ReplicaSet và Pod đi theo ngay, đi theo ở nền,
   hay ở lại làm object mồ côi.

Khi mọi checkbox được đánh dấu và bạn không còn nhầm scale với rollout, không nhầm selector với
`ownerReferences`, không nhầm `Recreate` với `RollingUpdate`, thì Lab 4a hoàn tất. Nhóm bài 4b —
StatefulSet, DaemonSet, Job và autoscaling — là bước tiếp theo.

---

## 4. Troubleshooting của lab này

Sự cố khi dựng môi trường nằm ở [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Bảng dưới chỉ liệt kê sự cố phát sinh từ nội dung bài học 4a.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| B1.2 ra 5 Pod thay vì 3 | `kubectl get pods -n lab-4a --show-labels` | Có Pod trần thừa mang `tier: frontend`; xóa Pod thừa rồi chạy lại B1.2, đừng tăng `replicas` cho khớp |
| B1.7 thấy Pod đã đổi sang image mới | `kubectl get pods -o jsonpath=...{.spec.containers[0].image}` | ReplicaSet cũ chưa bị xóa bằng `--cascade=orphan` mà bị xóa thường, nên Pod là Pod mới; xóa hết Pod `tier=frontend` và chạy lại từ B1.2 |
| Revision không tăng sau `kubectl set image` | `kubectl get deploy web -o yaml \| grep -A3 'containers:'` | Image mới trùng image cũ nên `.spec.template` không đổi; đổi sang tag khác thật sự |
| `rollout status` treo mãi ở B4 | `kubectl get rs,pods -n lab-4a -l app=web` | Pod mới không lên được; đọc `kubectl describe pod` trước khi đổi tham số chiến lược |
| B5 lấy được quá ít mẫu | Số dòng của `b5-mau-*.txt` | Rollout kết thúc nhanh hơn nhịp lấy mẫu; tăng `minReadySeconds` trong `surge-demo.yaml` rồi chạy lại — đừng sửa gate |
| B5 `max Pod tho` vượt `replicas + maxSurge` | So với cột `max tong replica RS` | Đây là hành vi đúng: Pod đang kết thúc không được tính vào `availableReplicas`; gate đặt trên ReplicaSet, không phải trên số Pod |
| B6 `min availableReplicas` không về 0 | `kubectl describe deploy recreate-demo \| grep StrategyType` | Manifest còn `rollingUpdate` hoặc thiếu `preStop`; apply lại đúng file `recreate-demo.yaml` |
| Patch đổi `strategy.type` sang `Recreate` bị API từ chối | Nội dung patch | `rollingUpdate` không được khai cùng `type: Recreate`; dùng manifest đầy đủ như B6 thay vì patch một field |
| B7 không thấy `ProgressDeadlineExceeded` | `kubectl get deploy fail-demo -o jsonpath='{.status.conditions}'` | Chưa đủ thời gian theo `progressDeadlineSeconds` đã đặt; chờ tiếp vòng lặp, đừng hạ deadline giữa chừng |
| B7 Pod hỏng nằm trên worker1 hoặc master | `kubectl get pods -o wide -n lab-4a` | `nodeName` bị sửa; xóa Deployment, khôi phục `nodeName: lab-k8s-worker2` rồi chạy lại — fault injection chỉ được ở worker2 |
| B8 không bắt được `foregroundDeletion` | `kubectl get deploy cascade-demo -o jsonpath='{.metadata.finalizers}'` | Pod chấm dứt quá nhanh; xác nhận `preStop` và `terminationGracePeriodSeconds` còn trong manifest |
| B9 `kubectl diff` báo lỗi quyền | Thông báo lỗi | `diff` chạy apply dry-run phía server, cần quyền `PATCH`/`CREATE`/`UPDATE`; dùng kubeconfig quản trị của Lab 00 |
| Namespace `lab-4a` kẹt `Terminating` | `kubectl get ns lab-4a -o yaml`; `kubectl get all -n lab-4a` | Chờ Pod hết `terminationGracePeriodSeconds`; nếu state khác baseline thì restore cả ba VM về `01-cluster-ready` |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Workload Management](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/)
- [Kubernetes v1.35 — ReplicaSet](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Kubernetes v1.35 — Deployments](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes v1.35 — Managing Workloads](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/management/)
- [Kubernetes v1.35 — ReplicationController](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)
- [Kubernetes v1.35 — Use Cascading Deletion in a Cluster](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/)
- [Kubernetes v1.35 — Declarative Management of Kubernetes Objects Using Configuration Files](https://v1-35.docs.kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)
- [Kubernetes v1.35 — Update API Objects in Place Using kubectl patch](https://v1-35.docs.kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/)
- [Kubernetes v1.35 — Run Applications](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/)
- [Kubernetes v1.35 — Run a Stateless Application Using a Deployment](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/)
- [Kubernetes v1.35 — Horizontal Manual Scaling for a Deployment](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/scale-deployment/)
- [Kubernetes — Update a Deployment Without Downtime](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/)
  — trang này được thêm sau v1.35 nên chỉ có ở bản tài liệu hiện hành, không có bản `v1-35`.
