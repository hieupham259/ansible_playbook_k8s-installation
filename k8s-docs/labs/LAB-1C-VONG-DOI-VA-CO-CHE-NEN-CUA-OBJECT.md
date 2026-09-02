# Lab 1c — Vòng đời và cơ chế nền của object

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem
> [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> **Điểm kết thúc:** cleanup trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới.
> **Lab trước:** [Lab 1b — Object, label, kubectl và kubeconfig](LAB-1B-OBJECT-LABEL-KUBECTL-VA-KUBECONFIG.md)
> đã cleanup toàn bộ namespace và kubeconfig tạm.
>
> **Cập nhật và đối chiếu phiên bản:** 23/08/2026.

Lab này đi cùng mục
[1c. Vòng đời và cơ chế nền của object](../00-ALO-TRINH-ADMIN.md#1c-vòng-đời-và-cơ-chế-nền-của-object).
Cluster giữ nguyên toàn bộ baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này không
chép lại con số nào. Topology vẫn là một kube-apiserver trên `lab-k8s-master` và hai worker —
đây là dữ kiện quyết định của B3, B5 và B6. Lab không cài cloud-controller-manager, storage
migrator, CRD, controller hay add-on mới.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả
`No resources found`; CoreDNS đủ replica `READY`; namespace `default` không có Pod.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- DELETE một object có finalizer chỉ đặt `deletionTimestamp`; object tồn tại tới khi danh sách
  finalizer rỗng.
- Finalizer là key, không chứa code; controller bên ngoài object thực hiện cleanup rồi gỡ key.
- Owner reference nằm trên dependent và phải mang name lẫn UID của owner.
- Label mô tả tập object; owner reference mô tả quan hệ vòng đời và quyết định garbage
  collection.
- Ba propagation policy: background, foreground và orphan; background là mặc định.
- Lease dùng cho Node heartbeat, leader election và định danh kube-apiserver.
- API version client thấy khác khái niệm storage version; API server chịu trách nhiệm conversion.
- Mixed Version Proxy chỉ có ý nghĩa khi control plane HA tạm chạy nhiều phiên bản API server.
- Cluster self-managed/on-premise này không có cloud-controller-manager; route Pod do Flannel
  thực hiện, không phải cloud route controller.
- Cleanup toàn bộ object lab và đưa cluster về `01-cluster-ready`.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong 1c | Phần lab kiểm chứng |
| --- | --- |
| [Finalizers](../29-finalizers-vi.md) | B1 — `deletionTimestamp`, finalizer và hoàn tất DELETE |
| [Owners và dependents](../30-owners-dependents-vi.md) | B2 — Owner reference gồm name/UID |
| [Garbage collection](../36-garbage-collection-vi.md) | B2 — Background, orphan, foreground |
| [Leases](../35-leases-vi.md) | B3 — Node heartbeat, leader và API server identity |
| [Storage versions](../32-storage-version-vi.md) | B4 — API representation, write và giới hạn quan sát storage version |
| [Mixed Version Proxy](../37-mixed-version-proxy-vi.md) | B5 — Giới hạn của control plane một API server |
| [Cloud Controller Manager](../34-cloud-controller-vi.md) | B6 — Bằng chứng cluster on-premise không có CCM |
| [Phát triển Cloud Controller Manager](../203-developing-cloud-controller-manager-vi.md) — bài **Thực hành** của nhóm | B6 — mô hình plug-in out-of-tree/in-tree không kiểm chứng được trên cluster không có cloud provider; phần kiểm chứng được là `spec.providerID` để trống, tức chưa provider nào đăng ký |

### 1.2. Thời lượng

Khoảng 2–3 giờ, tính từ lúc gate `01-cluster-ready` đã PASS.

---

## 2. Quy ước và an toàn

- Mọi lệnh chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, trừ khi ghi rõ khác.
- Lab chỉ tạo Namespace `lab-1c` và ConfigMap trong namespace đó. Không tạo CRD, PV, Service,
  workload controller hoặc object cloud.
- Chỉ gỡ finalizer thủ công khỏi ConfigMap do chính lab tạo với prefix
  `training.example.com/`. Không gỡ finalizer của Namespace, PV/PVC hay object hệ thống.
- Owner và dependent đều phải ở `lab-1c`; không tạo owner reference cross-namespace.
- Không truy cập etcd trực tiếp bằng `etcdctl`. Phần storage version dừng ở ranh giới API
  discovery; backup/migration etcd thuộc giai đoạn sau.
- Không bật `UnknownVersionInteroperabilityProxy`, không sửa manifest kube-apiserver và không
  giả lập control plane HA trên cluster ba VM này.
- Không cài cloud-controller-manager để “thử”; sự vắng mặt của nó là kết quả đúng cho hạ tầng
  self-managed.
- Evidence ghi vào `~/lab-evidence/1c/`; dòng bắt đầu bằng `PASS:` là gate.

---

# Phần B — Thực hành kiến thức 1c

## B0. Chuẩn bị workspace và namespace

```bash
mkdir -p ~/lab-work/1c ~/lab-evidence/1c
kubectl config current-context
kubectl get nodes
kubectl create namespace lab-1c
kubectl get namespace lab-1c -o yaml \
  | tee ~/lab-evidence/1c/00-namespace.yaml
```

**Ý nghĩa:** `lab-work` chứa manifest tạm; `lab-evidence` chứa output không bí mật. Namespace
cô lập toàn bộ object có tác động của lab.

**PASS:** context đúng cluster lab; ba node `Ready`; namespace `lab-1c` ở phase `Active`.

## B1. Finalizer giữ object ở trạng thái đang xóa

### B1.1. Tạo object với finalizer do lab kiểm soát

```bash
cat > ~/lab-work/1c/finalizer-demo.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: finalizer-demo
  namespace: lab-1c
  finalizers:
  - training.example.com/manual-cleanup
data:
  purpose: observe-finalizer
EOF

kubectl apply --dry-run=server --validate=strict \
  -f ~/lab-work/1c/finalizer-demo.yaml
kubectl apply -f ~/lab-work/1c/finalizer-demo.yaml
kubectl get configmap finalizer-demo -n lab-1c -o yaml \
  | tee ~/lab-evidence/1c/01-finalizer-before.yaml
```

Tên finalizer tùy chỉnh phải là qualified name có prefix DNS. Key này không chạy code; lab sẽ
đóng vai controller thủ công ở bước phục hồi.

### B1.2. Yêu cầu DELETE và quan sát trạng thái trung gian

```bash
kubectl delete configmap finalizer-demo -n lab-1c --wait=false

DELETION_TS="$(kubectl get configmap finalizer-demo -n lab-1c \
  -o jsonpath='{.metadata.deletionTimestamp}')"
FINALIZERS="$(kubectl get configmap finalizer-demo -n lab-1c \
  -o jsonpath='{.metadata.finalizers[*]}')"

echo "deletionTimestamp: $DELETION_TS"
echo "finalizers: $FINALIZERS"
kubectl get configmap finalizer-demo -n lab-1c -o yaml \
  | tee ~/lab-evidence/1c/01-finalizer-terminating.yaml

test -n "$DELETION_TS" \
  && test "$FINALIZERS" = 'training.example.com/manual-cleanup' \
  && echo 'PASS: DELETE accepted, object retained by finalizer'
```

**Ý nghĩa:** API server chấp nhận DELETE, đặt `metadata.deletionTimestamp` và giữ object vì
`metadata.finalizers` chưa rỗng. Sau khi deletion bắt đầu, không thể xóa `deletionTimestamp`
hay thêm finalizer mới; chỉ có thể hoàn tất cleanup rồi gỡ key hiện có.

### B1.3. Phục hồi bắt buộc: hoàn tất cleanup và gỡ finalizer

Trong lab này không có external resource thật; việc “cleanup” đã hoàn tất ngay sau khi evidence
được ghi. Chỉ vì vậy mới được gỡ finalizer do lab tạo:

```bash
kubectl patch configmap finalizer-demo -n lab-1c --type=json \
  -p='[{"op":"remove","path":"/metadata/finalizers"}]'

for i in {1..30}; do
  if ! kubectl get configmap finalizer-demo -n lab-1c >/dev/null 2>&1; then
    break
  fi
  sleep 1
done

if kubectl get configmap finalizer-demo -n lab-1c >/dev/null 2>&1; then
  echo 'FAIL: finalizer-demo still exists'
else
  echo 'PASS: finalizer removed and object deletion completed'
fi
```

**STOP:** không đi tiếp nếu object còn tồn tại. Không force-delete và không mở rộng thao tác sang
object khác.

## B2. Owner reference và garbage collection

Owner reference của các ví dụ dưới đây nằm trên dependent, trỏ tới owner cùng namespace bằng
`apiVersion`, `kind`, `name` và `uid`.

### B2.1. Background deletion — owner biến mất trước

```bash
kubectl create configmap owner-background -n lab-1c \
  --from-literal=policy=background
OWNER_UID="$(kubectl get configmap owner-background -n lab-1c \
  -o jsonpath='{.metadata.uid}')"

cat > ~/lab-work/1c/dependent-background.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: dependent-background
  namespace: lab-1c
  ownerReferences:
  - apiVersion: v1
    kind: ConfigMap
    name: owner-background
    uid: ${OWNER_UID}
    controller: false
    blockOwnerDeletion: false
data:
  policy: background
EOF

kubectl apply --dry-run=server --validate=strict \
  -f ~/lab-work/1c/dependent-background.yaml
kubectl apply -f ~/lab-work/1c/dependent-background.yaml
kubectl get configmap dependent-background -n lab-1c \
  -o jsonpath='owner={.metadata.ownerReferences[0].name}{"\n"}uid={.metadata.ownerReferences[0].uid}{"\n"}'

kubectl delete configmap owner-background -n lab-1c \
  --cascade=background --wait=true

for i in {1..30}; do
  if ! kubectl get configmap dependent-background -n lab-1c >/dev/null 2>&1; then
    break
  fi
  sleep 1
done

if kubectl get configmap owner-background -n lab-1c >/dev/null 2>&1 || \
   kubectl get configmap dependent-background -n lab-1c >/dev/null 2>&1; then
  echo 'FAIL: background garbage collection incomplete'
else
  echo 'PASS: background removed owner, then garbage collector removed dependent'
fi
```

**Ý nghĩa:** background là mặc định. API server xóa owner trước; garbage collector quan sát
owner reference và xóa dependent ở nền.

### B2.2. Orphan deletion — dependent được giữ lại

```bash
kubectl create configmap owner-orphan -n lab-1c \
  --from-literal=policy=orphan
OWNER_UID="$(kubectl get configmap owner-orphan -n lab-1c \
  -o jsonpath='{.metadata.uid}')"

cat > ~/lab-work/1c/dependent-orphan.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: dependent-orphan
  namespace: lab-1c
  ownerReferences:
  - apiVersion: v1
    kind: ConfigMap
    name: owner-orphan
    uid: ${OWNER_UID}
    controller: false
    blockOwnerDeletion: false
data:
  policy: orphan
EOF

kubectl apply -f ~/lab-work/1c/dependent-orphan.yaml
kubectl delete configmap owner-orphan -n lab-1c \
  --cascade=orphan --wait=true

for i in {1..30}; do
  OWNER_REFS="$(kubectl get configmap dependent-orphan -n lab-1c \
    -o jsonpath='{.metadata.ownerReferences[*].uid}' 2>/dev/null || true)"
  if [ -z "$OWNER_REFS" ]; then
    break
  fi
  sleep 1
done

kubectl get configmap dependent-orphan -n lab-1c -o yaml \
  | tee ~/lab-evidence/1c/02-dependent-orphan.yaml

kubectl get configmap dependent-orphan -n lab-1c >/dev/null 2>&1 \
  && test -z "$OWNER_REFS" \
  && echo 'PASS: owner deleted; dependent retained without owner reference'

kubectl delete configmap dependent-orphan -n lab-1c --wait=true
```

**Ý nghĩa:** orphan policy xóa owner nhưng giữ dependent; garbage collector gỡ owner reference
khỏi object được giữ lại. Lab xóa dependent thủ công ngay sau khi ghi evidence.

### B2.3. Foreground deletion — dependent chặn owner

Để quan sát trạng thái trung gian một cách xác định, dependent có
`blockOwnerDeletion: true` và finalizer lab. Foreground phải giữ owner cho tới khi dependent
được xóa xong.

```bash
kubectl create configmap owner-foreground -n lab-1c \
  --from-literal=policy=foreground
OWNER_UID="$(kubectl get configmap owner-foreground -n lab-1c \
  -o jsonpath='{.metadata.uid}')"

cat > ~/lab-work/1c/dependent-foreground.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: dependent-foreground
  namespace: lab-1c
  finalizers:
  - training.example.com/manual-cleanup
  ownerReferences:
  - apiVersion: v1
    kind: ConfigMap
    name: owner-foreground
    uid: ${OWNER_UID}
    controller: false
    blockOwnerDeletion: true
data:
  policy: foreground
EOF

kubectl apply --dry-run=server --validate=strict \
  -f ~/lab-work/1c/dependent-foreground.yaml
kubectl apply -f ~/lab-work/1c/dependent-foreground.yaml
kubectl delete configmap owner-foreground -n lab-1c \
  --cascade=foreground --wait=false

OWNER_DELETE=''
CHILD_DELETE=''
for i in {1..30}; do
  OWNER_DELETE="$(kubectl get configmap owner-foreground -n lab-1c \
    -o jsonpath='{.metadata.deletionTimestamp}' 2>/dev/null || true)"
  CHILD_DELETE="$(kubectl get configmap dependent-foreground -n lab-1c \
    -o jsonpath='{.metadata.deletionTimestamp}' 2>/dev/null || true)"
  if [ -n "$OWNER_DELETE" ] && [ -n "$CHILD_DELETE" ]; then
    break
  fi
  sleep 1
done

OWNER_FINALIZERS="$(kubectl get configmap owner-foreground -n lab-1c \
  -o jsonpath='{.metadata.finalizers[*]}')"
CHILD_FINALIZERS="$(kubectl get configmap dependent-foreground -n lab-1c \
  -o jsonpath='{.metadata.finalizers[*]}')"

echo "owner deletionTimestamp: $OWNER_DELETE"
echo "owner finalizers: $OWNER_FINALIZERS"
echo "dependent deletionTimestamp: $CHILD_DELETE"
echo "dependent finalizers: $CHILD_FINALIZERS"

test -n "$OWNER_DELETE" \
  && test -n "$CHILD_DELETE" \
  && echo 'PASS: owner and dependent both entered deletion'
case "$OWNER_FINALIZERS" in
  *foregroundDeletion*) echo 'PASS: owner retained by foregroundDeletion' ;;
  *) echo 'FAIL: foregroundDeletion not observed' ;;
esac
test "$CHILD_FINALIZERS" = 'training.example.com/manual-cleanup' \
  && echo 'PASS: dependent still held by the lab finalizer'

kubectl get configmap owner-foreground dependent-foreground -n lab-1c -o yaml \
  | tee ~/lab-evidence/1c/02-foreground-terminating.yaml
```

Phục hồi bắt buộc sau khi đã ghi evidence:

```bash
kubectl patch configmap dependent-foreground -n lab-1c --type=json \
  -p='[{"op":"remove","path":"/metadata/finalizers"}]'

for i in {1..30}; do
  OWNER_EXISTS=0
  CHILD_EXISTS=0
  kubectl get configmap owner-foreground -n lab-1c >/dev/null 2>&1 && OWNER_EXISTS=1
  kubectl get configmap dependent-foreground -n lab-1c >/dev/null 2>&1 && CHILD_EXISTS=1
  if [ "$OWNER_EXISTS" -eq 0 ] && [ "$CHILD_EXISTS" -eq 0 ]; then
    break
  fi
  sleep 1
done

test "$OWNER_EXISTS" -eq 0 \
  && test "$CHILD_EXISTS" -eq 0 \
  && echo 'PASS: dependent completed deletion before foreground owner disappeared'
```

**PASS B2:** background xóa cả owner/dependent; orphan giữ dependent rồi lab dọn thủ công;
foreground quan sát được `deletionTimestamp`, `foregroundDeletion` và chỉ hoàn tất sau khi
finalizer dependent được gỡ. Không còn ConfigMap B2.

## B3. Lease: heartbeat, leader election và API server identity

### B3.1. Node Lease tiếp tục đổi `renewTime`

```bash
LEASE_BEFORE="$(kubectl -n kube-node-lease get lease lab-k8s-worker1 \
  -o jsonpath='{.spec.renewTime}')"
sleep 12
LEASE_AFTER="$(kubectl -n kube-node-lease get lease lab-k8s-worker1 \
  -o jsonpath='{.spec.renewTime}')"

echo "before: $LEASE_BEFORE"
echo "after:  $LEASE_AFTER"
test -n "$LEASE_BEFORE" \
  && test -n "$LEASE_AFTER" \
  && test "$LEASE_BEFORE" != "$LEASE_AFTER" \
  && echo 'PASS: kubelet renewed the Node Lease'
```

Mỗi kubelet update Lease trùng tên Node trong `kube-node-lease`; control plane dùng
`spec.renewTime` để đánh giá Node còn liên lạc.

### B3.2. Control-plane Lease

```bash
kubectl -n kube-system get lease kube-controller-manager kube-scheduler \
  -o custom-columns='NAME:.metadata.name,HOLDER:.spec.holderIdentity,RENEW:.spec.renewTime,DURATION:.spec.leaseDurationSeconds' \
  | tee ~/lab-evidence/1c/03-leader-leases.txt

kubectl -n kube-system get lease \
  -l apiserver.kubernetes.io/identity=kube-apiserver \
  -o custom-columns='NAME:.metadata.name,HOLDER:.spec.holderIdentity,HOST:.metadata.labels.kubernetes\.io/hostname,RENEW:.spec.renewTime' \
  | tee ~/lab-evidence/1c/03-apiserver-identity-leases.txt

APISERVER_LEASE_COUNT="$(kubectl -n kube-system get lease \
  -l apiserver.kubernetes.io/identity=kube-apiserver -o name | wc -l)"
test "$APISERVER_LEASE_COUNT" -eq 1 \
  && echo 'PASS: leader leases exist and one kube-apiserver identity is published'
```

**Ý nghĩa:** scheduler/controller-manager dùng Lease để bầu leader; dù cluster hiện chỉ có một
instance, cùng cơ chế cho phép nhiều instance HA chỉ có một leader hoạt động. Kube-apiserver
công bố identity bằng Lease tên `apiserver-<hash>`; một Lease khớp topology một control plane.

**PASS:** Node renewTime thay đổi; Lease scheduler/controller-manager có holder và renewTime;
đúng một API server identity Lease.

## B4. API version và storage version

**Mục đích:** quan sát đúng ranh giới mà client được phép thấy. Lab không giả vờ rằng output từ
`kubectl get` có thể chứng minh encoding thực tế trong etcd.

```bash
kubectl create configmap storage-demo -n lab-1c \
  --from-literal=version=initial

kubectl get configmap storage-demo -n lab-1c -o json \
  | tee ~/lab-evidence/1c/04-api-before.json
kubectl get --raw '/api/v1/namespaces/lab-1c/configmaps/storage-demo' \
  | tee ~/lab-evidence/1c/04-api-raw.json
echo

API_VERSION_BEFORE="$(kubectl get configmap storage-demo -n lab-1c \
  -o jsonpath='{.apiVersion}')"
RESOURCE_VERSION_BEFORE="$(kubectl get configmap storage-demo -n lab-1c \
  -o jsonpath='{.metadata.resourceVersion}')"

kubectl patch configmap storage-demo -n lab-1c --type=merge \
  -p='{"data":{"version":"rewritten"}}'

API_VERSION_AFTER="$(kubectl get configmap storage-demo -n lab-1c \
  -o jsonpath='{.apiVersion}')"
RESOURCE_VERSION_AFTER="$(kubectl get configmap storage-demo -n lab-1c \
  -o jsonpath='{.metadata.resourceVersion}')"

echo "API before:      $API_VERSION_BEFORE"
echo "API after:       $API_VERSION_AFTER"
echo "resourceVersion before: $RESOURCE_VERSION_BEFORE"
echo "resourceVersion after:  $RESOURCE_VERSION_AFTER"

test "$API_VERSION_BEFORE" = 'v1' \
  && test "$API_VERSION_AFTER" = 'v1' \
  && test "$RESOURCE_VERSION_BEFORE" != "$RESOURCE_VERSION_AFTER" \
  && test "$(kubectl get configmap storage-demo -n lab-1c \
    -o jsonpath='{.data.version}')" = 'rewritten' \
  && echo 'PASS: write observed; storage stays hidden behind the API server'
```

**Ý nghĩa:** `apiVersion: v1` là representation mà request/response dùng; nó không chứng minh
object được serialize trong etcd bằng version nào. `resourceVersion` đổi chứng minh một write
mới đã xảy ra, nhưng `resourceVersion` cũng không phải storage version. Theo mô hình Kubernetes,
API server ghi bằng storage version đang hoạt động và tự conversion khi client đọc qua served
version. Object cũ chỉ chuyển sang storage version hiện hành khi có thao tác ghi.

Không chạy `etcdctl get`, không sửa encryption config, không chạy storage migration và không
tạo CRD nhiều version ở giai đoạn này; các thao tác đó đã được lộ trình hoãn sang giai đoạn sau.

**PASS:** hai lần đọc đều trả API representation `v1`; write làm `resourceVersion` đổi và data
thành `rewritten`; người học giải thích được vì sao không output nào ở đây tiết lộ trực tiếp
storage version trong etcd.

## B5. Mixed Version Proxy không áp dụng cho topology này

```bash
kubectl -n kube-system get pods -l component=kube-apiserver -o wide \
  | tee ~/lab-evidence/1c/05-apiserver-pods.txt

APISERVER_COUNT="$(kubectl -n kube-system get pods \
  -l component=kube-apiserver \
  --field-selector=status.phase=Running -o name | wc -l)"

CONTROL_PLANE_VERSIONS="$(kubectl get nodes \
  -l node-role.kubernetes.io/control-plane \
  -o jsonpath='{range .items[*]}{.status.nodeInfo.kubeletVersion}{"\n"}{end}' \
  | sort -u)"
CONTROL_PLANE_VERSION_COUNT="$(printf '%s\n' "$CONTROL_PLANE_VERSIONS" \
  | sed '/^$/d' | wc -l)"

echo "running kube-apiserver Pods: $APISERVER_COUNT"
echo "control-plane versions:"
echo "$CONTROL_PLANE_VERSIONS"

sudo grep -nE -- \
  '--peer-ca-file|--peer-advertise-ip|--peer-advertise-port|UnknownVersionInteroperabilityProxy' \
  /etc/kubernetes/manifests/kube-apiserver.yaml || true

test "$APISERVER_COUNT" -eq 1 \
  && test "$CONTROL_PLANE_VERSION_COUNT" -eq 1 \
  && echo 'PASS: single-version, single-apiserver cluster has no peer path to proxy'
```

**Ý nghĩa:** Mixed Version Proxy giải quyết rolling upgrade của control plane HA, khi request
đến API server cũ không biết resource nhưng peer mới biết. Không có proxy thì đường đó kết thúc
`404`; đã chọn peer nhưng peer chết thì trả `503`. Cluster lab có một kube-apiserver, nên bật
tính năng hoặc fault-inject peer là sai phạm vi. Lệnh `grep` có thể không in gì và chỉ dùng để
đối chiếu manifest read-only.

**PASS:** đúng một kube-apiserver `Running`, một control-plane version và không có peer khác để
proxy.

## B6. Cloud Controller Manager không tồn tại trên cluster self-managed

```bash
kubectl -n kube-system get pods -o name \
  | grep 'cloud-controller-manager' || true

CCM_COUNT="$(kubectl -n kube-system get pods -o name \
  | grep -c 'cloud-controller-manager' || true)"

kubectl get nodes \
  -o custom-columns='NAME:.metadata.name,PROVIDER_ID:.spec.providerID,INTERNAL_IP:.status.addresses[?(@.type=="InternalIP")].address' \
  | tee ~/lab-evidence/1c/06-node-providerid.txt

PROVIDER_ID_COUNT=0
for node in $(kubectl get nodes -o name); do
  PROVIDER_ID="$(kubectl get "$node" -o jsonpath='{.spec.providerID}')"
  if [ -n "$PROVIDER_ID" ]; then
    echo "FAIL: $node has providerID $PROVIDER_ID"
    PROVIDER_ID_COUNT=$((PROVIDER_ID_COUNT + 1))
  fi
done

sudo grep -n -- '--cloud-provider' \
  /etc/kubernetes/manifests/kube-controller-manager.yaml || true

test "$CCM_COUNT" -eq 0 \
  && test "$PROVIDER_ID_COUNT" -eq 0 \
  && echo 'PASS: no cloud-controller-manager and no Node providerID on self-managed VMs'
```

**Ý nghĩa:** cloud-controller-manager chứa Node, Route và Service controller đặc thù cloud;
chúng gọi API cloud để đồng bộ VM, route và load balancer. Cluster VMware self-managed không có
cloud API/provider integration, nên không có CCM và `spec.providerID` để trống. Pod networking
do Flannel thực hiện; node-lifecycle controller trong kube-controller-manager vẫn theo dõi
heartbeat, nhưng không hỏi cloud API và không tự xóa VM/Node đã mất.

**PASS:** không có Pod CCM; cả ba Node không có providerID; manifest controller-manager không
cấu hình external cloud provider.

## B7. Cleanup và gate cuối

```bash
kubectl delete namespace lab-1c --wait=true --timeout=120s

if kubectl get namespace lab-1c >/dev/null 2>&1; then
  echo 'FAIL: namespace lab-1c still exists'
else
  echo 'PASS: namespace lab-1c is gone'
fi

rm -f "$HOME/lab-work/1c/finalizer-demo.yaml"
rm -f "$HOME/lab-work/1c/dependent-background.yaml"
rm -f "$HOME/lab-work/1c/dependent-orphan.yaml"
rm -f "$HOME/lab-work/1c/dependent-foreground.yaml"
rmdir "$HOME/lab-work/1c"
test ! -e "$HOME/lab-work/1c" \
  && echo 'PASS: lab manifests removed'

kubectl wait --for=condition=Ready node --all --timeout=120s
kubectl get nodes
kubectl get pods -n default
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
```

Nhánh `else` mới là gate: chỉ khi API server trả `NotFound` cho `lab-1c` mới có dòng `PASS:`.
`rmdir` cố ý không dùng `-rf` — nó fail nếu `~/lab-work/1c` còn file lạ, và `test ! -e` biến
điều đó thành gate thay vì im lặng bỏ qua. Nếu namespace vẫn tồn tại hoặc `Terminating`, kiểm
tra:

```bash
kubectl get namespace lab-1c -o yaml
kubectl get all,configmap -n lab-1c
```

Chỉ finalizer `training.example.com/manual-cleanup` trên ConfigMap lab mới được gỡ theo bước
phục hồi đã ghi. Không cưỡng chế finalizer Namespace.

**PASS:** không có dòng `FAIL:` nào; cả `PASS: namespace lab-1c is gone` lẫn `PASS: lab
manifests removed` đều xuất hiện; ba node `Ready`; default không có Pod; không có Pod khác
`Running/Succeeded`; CoreDNS đủ replica. Cluster trở về `01-cluster-ready`; không tạo snapshot
mới.

---

## 3. Checkpoint 1c

Tự trả lời không nhìn tài liệu:

- [ ] Mô tả chính xác phản ứng của API server khi DELETE object có finalizer.
- [ ] Giải thích vì sao finalizer không chứa code và khi nào mới được gỡ thủ công.
- [ ] Chỉ đúng owner reference nằm ở dependent và vì sao phải có UID.
- [ ] Phân biệt label với owner reference bằng câu hỏi mà mỗi cơ chế trả lời.
- [ ] So sánh background, foreground và orphan; chỉ ra policy mặc định.
- [ ] Kể ba cách dùng Lease đã quan sát và namespace của từng loại.
- [ ] Phân biệt API version, `resourceVersion` và storage version.
- [ ] Giải thích điều kiện xuất hiện Mixed Version Proxy và vì sao cluster lab không dùng.
- [ ] Kể ba controller của cloud-controller-manager và giải thích vì sao cluster lab không có.
- [ ] Cluster sạch và cả ba node `Ready` sau cleanup.

### Bài giải thích cuối cùng

Trong tối đa 10 phút, kể lại vòng đời sau:

1. Client gửi DELETE; API server đặt `deletionTimestamp` nếu object còn finalizer.
2. Controller hoàn tất cleanup và gỡ finalizer; object mới biến mất.
3. Garbage collector đọc owner reference để xử lý dependent theo propagation policy.
4. Lease cung cấp heartbeat/leader/identity mà không biến thành workload lock tùy tiện.
5. API server che storage representation khỏi client và thực hiện conversion.
6. Mixed Version Proxy và CCM chỉ xuất hiện khi topology/hạ tầng thật sự cần chúng.

Khi toàn bộ checkbox được đánh dấu và không còn nhầm finalizer với code, label với ownership,
API version với storage version, hay node controller với cloud node controller, Lab 1c và
Giai đoạn 1 hoàn tất.

---

## 4. Troubleshooting của lab này

Sự cố khi dựng môi trường nằm ở [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Bảng dưới chỉ liệt kê sự cố phát sinh từ nội dung bài học 1c.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| `finalizer-demo` không biến mất | `kubectl get cm finalizer-demo -n lab-1c -o yaml` | Chỉ gỡ `training.example.com/manual-cleanup` sau khi evidence đã ghi |
| Foreground owner biến mất quá sớm | Kiểm tra `blockOwnerDeletion`, UID và namespace của owner reference | Xóa object thử, tạo lại đúng manifest; không dùng cross-namespace reference |
| Foreground dependent kẹt sau khi quan sát | Kiểm tra finalizer lab trên dependent | Chạy đúng block phục hồi B2.3; không xóa namespace trước khi dependent/owner đã sạch |
| Leader Lease không tồn tại | `kubectl -n kube-system get lease` và manifest scheduler/controller-manager | Dừng ở B3; xác nhận control-plane Pod Running và `--leader-elect` của baseline |
| `resourceVersion` không đổi | Kiểm tra patch trả thành công và data đã thành `rewritten` | Chạy lại B4 từ object mới; không đọc etcd trực tiếp để tìm storage version |
| Có CCM hoặc providerID | Kiểm kê Pod/manifest/Node | Cluster không còn đúng baseline self-managed; dừng và đối chiếu Lab 00 |
| Namespace `Terminating` | `kubectl get ns lab-1c -o yaml`; `kubectl get all,cm -n lab-1c` | Chỉ xử lý finalizer lab đã biết; nếu state khác baseline thì restore snapshot |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Finalizers](https://v1-35.docs.kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/)
- [Kubernetes v1.35 — Owners and Dependents](https://v1-35.docs.kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/)
- [Kubernetes v1.35 — Garbage Collection](https://v1-35.docs.kubernetes.io/docs/concepts/architecture/garbage-collection/)
- [Kubernetes v1.35 — Leases](https://v1-35.docs.kubernetes.io/docs/concepts/architecture/leases/)
- [Kubernetes v1.35 — Storage Versions](https://v1-35.docs.kubernetes.io/docs/concepts/overview/working-with-objects/storage-version/)
- [Kubernetes v1.35 — Mixed Version Proxy](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/mixed-version-proxy/)
- [Kubernetes v1.35 — Cloud Controller Manager](https://v1-35.docs.kubernetes.io/docs/concepts/architecture/cloud-controller/)
