# Lab 1b — Object, label, kubectl và kubeconfig

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem
> [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> **Điểm kết thúc:** cleanup trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới.
> **Lab trước:** [Lab 1a — Kiến trúc và mô hình điều khiển](LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md)
> đã hoàn tất và không để lại workload.
>
> **Cập nhật và đối chiếu phiên bản:** 23/08/2026.

Lab này đi cùng mục
[1b. Làm việc với object và kubectl](../00-ALO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl).
Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa):
Kubernetes `v1.35.7`, containerd `2.2.1`, Flannel `v0.28.9`, Pod CIDR `10.244.0.0/16` và
Service CIDR `10.96.0.0/12`. Lab không cài package, CNI, controller hay add-on mới.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
và xác nhận gate `01-cluster-ready` PASS.

Gate ngắn, chạy trên `lab-k8s-master`:

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

- Name định danh object trong phạm vi API group, resource type và namespace; UID định danh một
  lần tồn tại cụ thể của object.
- Label dùng để chọn một tập object; annotation lưu metadata nhưng không dùng làm selector.
- Equality-based và set-based label selector, cùng ngữ nghĩa AND giữa nhiều requirement.
- Resource nào namespaced và resource nào cluster-scoped; cách `-n` và context chọn namespace.
- Ý nghĩa các label khuyến nghị `app.kubernetes.io/*`.
- `kubectl` luôn giao tiếp với API server và cách đọc schema bằng `kubectl explain`.
- Context gồm cluster, user và namespace; thứ tự ưu tiên `--kubeconfig`, `KUBECONFIG`, rồi
  `$HOME/.kube/config`.
- Khác biệt giữa imperative command, imperative object configuration và declarative
  configuration.
- Label selector lọc metadata do người dùng gắn; field selector lọc field được API hỗ trợ.
- Chạy được `kubectl apply -f pod.yaml`, chờ container `Running`, chọn đúng Pod bằng label và
  namespace, rồi cleanup hoàn toàn.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong 1b | Phần lab kiểm chứng |
| --- | --- |
| [Tên và ID](../17-names-vi.md) | B1 — Scope của name và vòng đời UID |
| [Namespaces](../19-namespaces-vi.md) | B1 — Namespace, `-n`, namespaced/cluster-scoped |
| [Label và Selector](../18-labels-vi.md) | B3 — Equality, set, exists và AND |
| [Annotations](../20-annotations-vi.md) | B3 — Metadata không thể select |
| [Các label được khuyến nghị](../31-common-labels-vi.md) | B2 — `app.kubernetes.io/*` trong manifest |
| [kubectl](../26-kubectl-vi.md) | B0–B7 — create/get/apply/diff/explain/delete |
| [kubeconfig](../111-kubeconfig-vi.md) | B4 — Bản sao tạm, context và namespace mặc định |
| [Quản lý object](../27-object-management-vi.md) | B5 — Imperative so với declarative |
| [Field selector](../28-field-selectors-vi.md) | B6 — Field hợp lệ và lỗi BadRequest |

### 1.2. Thời lượng

Khoảng 3–4 giờ, tính từ lúc gate `01-cluster-ready` đã PASS.

---

## 2. Quy ước và an toàn

- Mọi lệnh chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, trừ khi ghi rõ khác.
- Lab chỉ tạo hai namespace `lab-1b` và `lab-1b-alt`; không sửa object trong `default`,
  `kube-system`, `kube-flannel` hay `kube-node-lease`.
- Pod lab dùng đúng `busybox:1.37` đã khóa và cache từ Lab 00, với
  `imagePullPolicy: IfNotPresent`; không dùng tag `latest` và không thêm image mới.
- File kubeconfig tạm chứa thông tin xác thực quản trị. Giữ quyền `0600`, đặt trong
  `~/lab-work/1b/`, không chép vào `~/lab-evidence`, Git, tin nhắn hoặc dịch vụ bên ngoài.
- Không sửa trực tiếp `$HOME/.kube/config`. Mọi bài context dùng flag `--kubeconfig` trỏ tới bản
  sao tạm.
- Không trộn nhiều kỹ thuật quản lý trên cùng một object. Object imperative và declarative dùng
  tên khác nhau.
- Evidence không chứa Secret, token, client key hay kubeconfig thô.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail.

---

# Phần B — Thực hành kiến thức 1b

## B0. Chuẩn bị workspace và xác nhận client/server

**Mục đích:** tạo nơi lưu manifest/evidence, xác nhận context hiện tại và độ lệch phiên bản
`kubectl` vẫn phù hợp với control plane `v1.35.7`.

```bash
mkdir -p ~/lab-work/1b ~/lab-evidence/1b
kubectl config current-context
kubectl version -o yaml | tee ~/lab-evidence/1b/00-kubectl-version.yaml
kubectl cluster-info
kubectl get nodes
```

**Ý nghĩa:** `lab-work` chứa file tạm sẽ xóa cuối lab; `lab-evidence` chỉ chứa output không bí
mật. `kubectl version` hỏi cả client và API server. Theo version-skew policy, `kubectl` được hỗ
trợ trong khoảng một minor so với API server; baseline dùng cùng `v1.35.7` nên không có skew.

**PASS:** context trỏ cluster lab; client/server đều `v1.35.7`; API server phản hồi; cả ba node
`Ready`.

## B1. Namespace, name và UID

### B1.1. Tạo hai phạm vi tên

**Mục đích:** chứng minh cùng resource type và name có thể tồn tại ở hai namespace khác nhau,
đồng thời phân biệt resource namespaced với cluster-scoped.

```bash
kubectl create namespace lab-1b
kubectl create namespace lab-1b-alt

kubectl create configmap identity-demo -n lab-1b \
  --from-literal=scope=primary
kubectl create configmap identity-demo -n lab-1b-alt \
  --from-literal=scope=alternate

kubectl get configmap identity-demo -n lab-1b \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,UID:.metadata.uid,VALUE:.data.scope'
kubectl get configmap identity-demo -n lab-1b-alt \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,UID:.metadata.uid,VALUE:.data.scope'

kubectl api-resources --namespaced=true | head -n 20
kubectl api-resources --namespaced=false | head -n 20
kubectl get namespace lab-1b -o jsonpath='{.metadata.labels.kubernetes\.io/metadata\.name}{"\n"}'
```

**Ý nghĩa:** Namespace tạo scope cho name. Hai ConfigMap cùng tên hợp lệ vì namespace khác
nhau. `api-resources --namespaced` hỏi discovery API thay vì đoán loại resource. Namespace và
Node là cluster-scoped; ConfigMap và Pod là namespaced. Control plane tự gắn label bất biến
`kubernetes.io/metadata.name` lên Namespace.

**PASS:** cả hai ConfigMap tồn tại với cùng name nhưng namespace, UID và value khác nhau; hai
danh sách discovery phân biệt được resource scope; lệnh cuối in `lab-1b`.

### B1.2. Cùng name nhưng lần tồn tại mới có UID mới

```bash
OLD_UID="$(kubectl get configmap identity-demo -n lab-1b \
  -o jsonpath='{.metadata.uid}')"

kubectl delete configmap identity-demo -n lab-1b --wait=true
kubectl create configmap identity-demo -n lab-1b \
  --from-literal=scope=recreated

NEW_UID="$(kubectl get configmap identity-demo -n lab-1b \
  -o jsonpath='{.metadata.uid}')"

echo "old UID: $OLD_UID"
echo "new UID: $NEW_UID"
test -n "$OLD_UID"
test -n "$NEW_UID"
test "$OLD_UID" != "$NEW_UID" && echo 'PASS: same name, new object UID'
```

**Ý nghĩa:** name có thể tái sử dụng sau khi object cũ bị xóa; UID không tái sử dụng. Owner
reference ở Lab 1c cần UID chính vì name một mình không phân biệt được hai lần tồn tại.

Kiểm tra server từ chối name không hợp lệ:

```bash
if kubectl create configmap Bad_Name -n lab-1b >/tmp/lab1b-invalid-name.txt 2>&1; then
  echo 'FAIL: invalid object name was accepted'
else
  cat /tmp/lab1b-invalid-name.txt
  echo 'PASS: invalid object name rejected'
fi
```

**PASS:** UID trước/sau khác nhau; name chứa chữ hoa và `_` bị API server từ chối.

## B2. Viết và apply `pod.yaml`

**Mục đích:** tạo ba Pod tối thiểu bằng declarative manifest, dùng label nhiều chiều và label
khuyến nghị. Đây chỉ là object mẫu để học metadata/kubectl; vòng đời Pod học ở Lab 3a.

Tạo file trên `lab-k8s-master`:

```bash
cat > ~/lab-work/1b/pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: web-prod
  namespace: lab-1b
  labels:
    training.example.com/exercise: selectors
    environment: production
    tier: frontend
    app.kubernetes.io/name: demo-app
    app.kubernetes.io/instance: web-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: web
    app.kubernetes.io/part-of: lab-1b
    app.kubernetes.io/managed-by: kubectl
spec:
  terminationGracePeriodSeconds: 5
  containers:
  - name: sleeper
    image: busybox:1.37
    imagePullPolicy: IfNotPresent
    command: ["sh", "-c", "sleep 3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: web-qa
  namespace: lab-1b
  labels:
    training.example.com/exercise: selectors
    environment: qa
    tier: frontend
    app.kubernetes.io/name: demo-app
    app.kubernetes.io/instance: web-qa
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: web
    app.kubernetes.io/part-of: lab-1b
    app.kubernetes.io/managed-by: kubectl
spec:
  terminationGracePeriodSeconds: 5
  containers:
  - name: sleeper
    image: busybox:1.37
    imagePullPolicy: IfNotPresent
    command: ["sh", "-c", "sleep 3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: api-prod
  namespace: lab-1b
  labels:
    training.example.com/exercise: selectors
    environment: production
    tier: backend
    app.kubernetes.io/name: demo-app
    app.kubernetes.io/instance: api-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: lab-1b
    app.kubernetes.io/managed-by: kubectl
spec:
  terminationGracePeriodSeconds: 5
  containers:
  - name: sleeper
    image: busybox:1.37
    imagePullPolicy: IfNotPresent
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply --dry-run=server --validate=strict \
  -f ~/lab-work/1b/pod.yaml

kubectl diff -f ~/lab-work/1b/pod.yaml
DIFF_RC=$?
test "$DIFF_RC" -eq 1 && echo 'PASS: diff detects objects to create'

kubectl apply -f ~/lab-work/1b/pod.yaml
kubectl wait --for=condition=Ready pod \
  -n lab-1b -l training.example.com/exercise=selectors --timeout=120s
kubectl get pods -n lab-1b \
  -L environment,tier,app.kubernetes.io/name -o wide \
  | tee ~/lab-evidence/1b/02-pods-and-labels.txt
```

**Ý nghĩa:** server-side dry-run chạy validation/admission nhưng không persist. `kubectl diff`
trả exit code `1` khi có khác biệt dự kiến; đây không phải lỗi lab. `apply` tự chọn create vì
object chưa tồn tại. `wait` chờ condition `Ready`; `-L` chỉ thêm cột label, không lọc.

**Tác động mong đợi:** ba Pod được scheduler đặt lên một hoặc cả hai worker tùy trạng thái tài
nguyên; lab không áp đặt cách phân bố Pod vì scheduling chi tiết thuộc giai đoạn 7. Không sửa
CNI, Service hay workload hệ thống. Image đã có từ gate Lab 00 nên không cần pull image mới.

**PASS:** dry-run hợp lệ; diff trả `1`; apply tạo ba Pod; cả ba `Ready`, `1/1 Running`.

## B3. Label selector và annotation

### B3.1. Equality, set, exists và AND

```bash
kubectl get pods -n lab-1b -l environment=production
kubectl get pods -n lab-1b -l 'environment in (production,qa)'
kubectl get pods -n lab-1b \
  -l 'environment in (production,qa),tier notin (frontend)'
kubectl get pods -n lab-1b -l training.example.com/exercise

PROD_COUNT="$(kubectl get pods -n lab-1b \
  -l environment=production -o name | wc -l)"
BACKEND_COUNT="$(kubectl get pods -n lab-1b \
  -l 'environment in (production,qa),tier notin (frontend)' -o name | wc -l)"
EXERCISE_COUNT="$(kubectl get pods -n lab-1b \
  -l training.example.com/exercise -o name | wc -l)"

test "$PROD_COUNT" -eq 2
test "$BACKEND_COUNT" -eq 1
test "$EXERCISE_COUNT" -eq 3
echo 'PASS: equality, set, exists and AND selectors matched expected sets'
```

**Ý nghĩa:** `=` là equality selector; `in`/`notin` là set-based; chỉ viết key kiểm tra key
tồn tại. Dấu phẩy kết hợp requirement bằng AND. Không có OR giữa hai key; `in (a,b)` chỉ là
nhiều value được chấp nhận cho cùng một key. Label không cần duy nhất.

### B3.2. Cập nhật label và chứng minh annotation không select được

Object `identity-demo` được tạo bằng imperative command ở B1, nên tiếp tục quản lý nó bằng
imperative command thay vì trộn với `apply`:

```bash
kubectl label configmap identity-demo -n lab-1b \
  team=platform stage=learning
kubectl annotate configmap identity-demo -n lab-1b \
  training.example.com/build='{"commit":"abc123","owner":"team-platform"}'

kubectl get configmap -n lab-1b -l team=platform -L team,stage
kubectl get configmap identity-demo -n lab-1b \
  -o jsonpath='{.metadata.annotations.training\.example\.com/build}{"\n"}'

ANNOTATION_SELECTOR_COUNT="$(kubectl get configmap -n lab-1b \
  -l training.example.com/build -o name | wc -l)"
test "$ANNOTATION_SELECTOR_COUNT" -eq 0
echo 'PASS: label selects; annotation is stored but is not selectable'
```

**Ý nghĩa:** label value bị giới hạn và dùng được với `-l`; annotation value là string có thể
chứa JSON, khoảng trắng và ký tự đặc biệt, nhưng API không hỗ trợ annotation selector.

**PASS:** query `team=platform` trả `identity-demo`; JSON annotation đọc lại đúng; selector
dùng annotation key trả tập rỗng.

## B4. Kubeconfig, context và namespace mặc định

**Mục đích:** thao tác trên bản sao kubeconfig để không làm đổi context quản trị thật.

```bash
LAB_KUBECONFIG="$HOME/lab-work/1b/kubeconfig"
LAB_CLUSTER="$(kubectl config view --minify \
  -o jsonpath='{.contexts[0].context.cluster}')"
LAB_USER="$(kubectl config view --minify \
  -o jsonpath='{.contexts[0].context.user}')"

cp "$HOME/.kube/config" "$LAB_KUBECONFIG"
chmod 600 "$LAB_KUBECONFIG"

kubectl --kubeconfig="$LAB_KUBECONFIG" config set-context lab-1b \
  --cluster="$LAB_CLUSTER" --user="$LAB_USER" --namespace=lab-1b
kubectl --kubeconfig="$LAB_KUBECONFIG" config use-context lab-1b

kubectl --kubeconfig="$LAB_KUBECONFIG" config current-context
kubectl --kubeconfig="$LAB_KUBECONFIG" config view --minify \
  -o jsonpath='context={.current-context}{"\n"}cluster={.contexts[0].context.cluster}{"\n"}user={.contexts[0].context.user}{"\n"}namespace={.contexts[0].context.namespace}{"\n"}' \
  | tee ~/lab-evidence/1b/04-context-summary.txt

kubectl --kubeconfig="$LAB_KUBECONFIG" get pods

test "$(kubectl --kubeconfig="$LAB_KUBECONFIG" config current-context)" = 'lab-1b'
test "$(kubectl --kubeconfig="$LAB_KUBECONFIG" config view --minify \
  -o jsonpath='{.contexts[0].context.namespace}')" = 'lab-1b'
echo 'PASS: temporary context selects lab-1b without changing the real kubeconfig'
```

**Ý nghĩa:** context gom ba tham số cluster, user và namespace. Flag `--kubeconfig` có ưu tiên
cao nhất và dùng đúng một file; nếu không có flag, `KUBECONFIG` được merge; nếu không có cả hai,
kubectl dùng `$HOME/.kube/config`. Lệnh `get pods` không cần `-n` vì context tạm đã chọn
`lab-1b`.

**An toàn:** evidence chỉ lưu tên context/cluster/user/namespace, không lưu certificate, key hay
token. Không chạy `kubectl config set-context --current` trên kubeconfig thật.

**PASS:** current context của file tạm là `lab-1b`; namespace là `lab-1b`; lệnh không có `-n`
thấy đúng ba Pod lab; context thật không đổi.

## B5. Ba kỹ thuật quản lý object

### B5.1. Imperative command

```bash
kubectl create configmap imperative-demo -n lab-1b \
  --from-literal=mode=imperative
kubectl get configmap imperative-demo -n lab-1b -o yaml
```

Command diễn tả trực tiếp hành động và không tạo source-of-record bên ngoài live object. Kỹ
thuật này phù hợp cho tác vụ một lần và lab.

### B5.2. Imperative object configuration với create/replace

```bash
cat > ~/lab-work/1b/imperative-file.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: imperative-file-demo
  namespace: lab-1b
data:
  mode: create
EOF

kubectl create --dry-run=server --validate=strict \
  -f ~/lab-work/1b/imperative-file.yaml
kubectl create -f ~/lab-work/1b/imperative-file.yaml

sed -i 's/mode: create/mode: replace/' ~/lab-work/1b/imperative-file.yaml
kubectl replace -f ~/lab-work/1b/imperative-file.yaml

test "$(kubectl get configmap imperative-file-demo -n lab-1b \
  -o jsonpath='{.data.mode}')" = 'replace'
echo 'PASS: imperative file create/replace followed the requested operations'
```

**Ý nghĩa:** file là bản mô tả đầy đủ, nhưng người vận hành vẫn chỉ định thao tác `create` hoặc
`replace`. `replace` ghi live object theo file và có thể làm mất field không còn trong file;
không dùng kỹ thuật này trên object được `apply` quản lý.

### B5.3. Declarative configuration với diff/apply

```bash
cat > ~/lab-work/1b/management.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: declarative-demo
  namespace: lab-1b
  labels:
    app.kubernetes.io/name: management-demo
    app.kubernetes.io/instance: lab-1b
    app.kubernetes.io/managed-by: kubectl
data:
  mode: v1
  owner: platform
EOF

kubectl explain configmap.data
kubectl apply --dry-run=server --validate=strict \
  -f ~/lab-work/1b/management.yaml

kubectl diff -f ~/lab-work/1b/management.yaml
DIFF_RC=$?
test "$DIFF_RC" -eq 1
kubectl apply -f ~/lab-work/1b/management.yaml
kubectl apply -f ~/lab-work/1b/management.yaml

sed -i 's/mode: v1/mode: v2/' ~/lab-work/1b/management.yaml
kubectl diff -f ~/lab-work/1b/management.yaml
DIFF_RC=$?
test "$DIFF_RC" -eq 1
kubectl apply -f ~/lab-work/1b/management.yaml

test "$(kubectl get configmap declarative-demo -n lab-1b \
  -o jsonpath='{.data.mode}')" = 'v2'
echo 'PASS: declarative diff/apply converged live object to file state'
```

**Ý nghĩa:** declarative configuration là `diff/apply -f file`, để kubectl tự chọn create hoặc
patch. `apply` lần hai không đổi object. `apply` dùng patch để hội tụ các field do file quản lý.
Không dùng `edit`, `replace` hay imperative update lên `declarative-demo`, vì trộn kỹ thuật làm
source-of-record không còn đáng tin.

**PASS B5:** imperative command tạo `imperative-demo`; imperative object configuration tạo rồi
replace `imperative-file-demo`; schema declarative được `explain`, dry-run hợp lệ, diff phát
hiện create/update, apply lần hai idempotent và live data cuối là `mode=v2`.

## B6. Field selector

**Mục đích:** phân biệt server-side field selection với label selection.

```bash
kubectl get pods -n lab-1b --field-selector=status.phase=Running
kubectl get pods -n lab-1b --field-selector=metadata.name=web-prod

WEB_NODE="$(kubectl get pod web-prod -n lab-1b \
  -o jsonpath='{.spec.nodeName}')"
echo "web-prod node: $WEB_NODE"
kubectl get pods -n lab-1b --field-selector="spec.nodeName=$WEB_NODE"

RUNNING_COUNT="$(kubectl get pods -n lab-1b \
  --field-selector=status.phase=Running -o name | wc -l)"
test "$RUNNING_COUNT" -eq 3

if kubectl get pods -n lab-1b \
  --field-selector=spec.fieldThatDoesNotExist=value \
  >~/lab-evidence/1b/06-invalid-field-selector.txt 2>&1; then
  echo 'FAIL: unsupported field selector was accepted'
else
  cat ~/lab-evidence/1b/06-invalid-field-selector.txt
  echo 'PASS: unsupported field selector rejected with BadRequest'
fi
```

**Ý nghĩa:** mọi resource hỗ trợ `metadata.name` và `metadata.namespace`; Pod còn hỗ trợ
`status.phase`, `spec.nodeName` và một số field đã tài liệu hóa. Field selector chỉ hỗ trợ
`=`, `==`, `!=`, không hỗ trợ `in/notin`. Field không được hỗ trợ gây `BadRequest`; label key
không tồn tại chỉ cho tập rỗng vì label là metadata tùy ý.

**PASS:** ba Pod được lọc là `Running`; selector theo name trả đúng `web-prod`; selector theo
node chỉ trả Pod trên node đó; field giả bị server từ chối.

## B7. Cleanup và gate cuối

**Mục đích:** xóa toàn bộ object lab, xóa kubeconfig chứa credential và chứng minh cluster trở
về `01-cluster-ready`.

```bash
kubectl delete namespace lab-1b lab-1b-alt --wait=true --timeout=120s
kubectl get namespace lab-1b lab-1b-alt 2>&1 || true

rm -f "$HOME/lab-work/1b/kubeconfig"
rm -f "$HOME/lab-work/1b/pod.yaml"
rm -f "$HOME/lab-work/1b/imperative-file.yaml"
rm -f "$HOME/lab-work/1b/management.yaml"
rmdir "$HOME/lab-work/1b"
rm -f /tmp/lab1b-invalid-name.txt

kubectl wait --for=condition=Ready node --all --timeout=120s
kubectl get nodes
kubectl get pods -n default
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
```

`|| true` chỉ giữ shell tiếp tục để xem `NotFound`; không biến namespace còn tồn tại thành
PASS. Nếu namespace ở `Terminating`, dừng và kiểm tra `kubectl get namespace <name> -o yaml`.

**PASS:** cả hai namespace trả `NotFound`; kubeconfig tạm và manifest trong `lab-work` đã xóa;
ba node `Ready`; namespace `default` không có Pod; không có Pod khác `Running/Succeeded`;
CoreDNS đủ replica. Cluster trở về `01-cluster-ready`; không tạo snapshot mới.

---

## 3. Checkpoint 1b

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời không nhìn tài liệu:

- [ ] Kể đúng bốn phần định danh object và giải thích vì sao API version không nằm trong đó.
- [ ] Chứng minh cùng name có thể tồn tại ở hai namespace và object tái tạo có UID mới.
- [ ] Phân biệt name, label và annotation bằng mục đích sử dụng.
- [ ] Viết được equality, set-based, exists và selector nhiều requirement.
- [ ] Giải thích `-l` lọc còn `-L` chỉ thêm cột hiển thị.
- [ ] Phân biệt namespaced với cluster-scoped resource và dùng đúng `-n`.
- [ ] Kể ba thành phần của context và thứ tự ưu tiên kubeconfig.
- [ ] Phân biệt ba kỹ thuật quản lý object và không trộn chúng trên cùng object.
- [ ] Phân biệt label selector với field selector, gồm hành vi khi key/field không tồn tại.
- [ ] Tự mô tả đường đi `kubectl apply -f pod.yaml` tới khi container chạy.
- [ ] Cluster sạch và cả ba node `Ready` sau cleanup.

### Bài giải thích cuối cùng

Trong tối đa 10 phút, nói lại đường đi sau và nêu bằng chứng đã quan sát:

1. `kubectl` đọc kubeconfig để chọn cluster, user, context và namespace.
2. `kubectl apply` đọc YAML, dùng discovery/schema rồi gửi request tới API server.
3. API server xử lý object và persist state; scheduler chọn node cho Pod.
4. Kubelet trên node nhận assignment qua API server và gọi container runtime qua CRI.
5. Label selector chọn tập Pod theo metadata; field selector lọc theo field được API hỗ trợ.
6. Cleanup xóa namespace và đưa cluster về baseline.

Khi toàn bộ checkbox được đánh dấu và phần giải thích không còn lẫn name/label/annotation,
context/kubeconfig hay imperative/declarative, Lab 1b hoàn tất.

---

## 4. Troubleshooting của lab này

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| Pod `ImagePullBackOff` | `kubectl describe pod -n lab-1b <name>` | Xác nhận image đúng `busybox:1.37`; baseline Lab 00 phải có image cache trên worker; không đổi sang `latest` |
| Pod `Pending` | `kubectl describe pod -n lab-1b <name>` | Kiểm tra hai worker `Ready`, taint và capacity; nếu baseline lệch thì restore `01-cluster-ready` |
| `kubectl diff` trả code `1` | Đọc diff | Đây là kết quả đúng khi có khác biệt; chỉ code lớn hơn `1` là lỗi vận hành |
| Context thật bị đổi | `kubectl config current-context` | Không tiếp tục; đối chiếu baseline. Bài đúng chỉ sửa file `~/lab-work/1b/kubeconfig` |
| Namespace `Terminating` | `kubectl get ns <name> -o yaml` | Không cưỡng chế finalizer ở Lab 1b; chẩn đoán và restore nếu cần. Finalizer học ở Lab 1c |
| Field selector trả BadRequest | Đọc danh sách field hợp lệ trong lỗi | Dùng field được resource hỗ trợ; không đổi sang lọc client-side để che lỗi |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Names](https://v1-35.docs.kubernetes.io/docs/concepts/overview/working-with-objects/names/)
- [Kubernetes v1.35 — Labels and Selectors](https://v1-35.docs.kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Kubernetes v1.35 — Annotations](https://v1-35.docs.kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)
- [Kubernetes v1.35 — Namespaces](https://v1-35.docs.kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Kubernetes v1.35 — Recommended Labels](https://v1-35.docs.kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)
- [Kubernetes v1.35 — kubectl](https://v1-35.docs.kubernetes.io/docs/reference/kubectl/)
- [Kubernetes v1.35 — kubeconfig](https://v1-35.docs.kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
- [Kubernetes v1.35 — Object Management](https://v1-35.docs.kubernetes.io/docs/concepts/overview/working-with-objects/object-management/)
- [Kubernetes v1.35 — Field Selectors](https://v1-35.docs.kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/)
