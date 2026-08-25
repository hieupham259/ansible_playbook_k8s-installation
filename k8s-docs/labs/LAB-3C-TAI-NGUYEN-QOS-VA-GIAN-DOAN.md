# Lab 3c — Tài nguyên, QoS và gián đoạn

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem
> [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> **Điểm kết thúc:** cleanup trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới.
> **Lab trước:** [Lab 3b — Cấu hình ứng dụng: ConfigMap, Secret và dữ liệu cho Pod](LAB-3B-CAU-HINH-UNG-DUNG.md)
> đã cleanup namespace `lab-3b`. Cluster vào lab này vẫn phải ở đúng `01-cluster-ready`,
> không còn object nào của lab trước.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục
[3c. Tài nguyên, QoS và gián đoạn](../00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn).
Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này không
chép lại con số phiên bản nào và **không cài thêm gì**. Topology vẫn là một control plane
`lab-k8s-master` mang taint `NoSchedule` và hai worker `lab-k8s-worker1`, `lab-k8s-worker2` —
đây là dữ kiện quyết định của B4 và B5.

Toàn bộ lab chỉ dùng **Pod trần**. Deployment, ReplicaSet, StatefulSet thuộc
[giai đoạn 4](../00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller); Service thuộc giai
đoạn 5; PVC thuộc giai đoạn 6; ResourceQuota và LimitRange thuộc giai đoạn 7b. Không dùng thứ
nào trong số đó ở đây.

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

- `requests` nói chuyện với **scheduler**, `limits` nói chuyện với **kernel** — chỉ ra được
  chính xác trường cgroup v2 mà mỗi giá trị biến thành trên node.
- Đặt `limits` mà không đặt `requests` thì Kubernetes sao chép limit thành request; chiều
  ngược lại **không** xảy ra.
- Vượt `limits.cpu` bị **throttle** và container vẫn sống; vượt `limits.memory` bị
  **OOMKilled** — và việc kill đó tác động ở mức **container**, không phải mức Pod.
- Suy ra QoS class của một Pod **chỉ bằng cách nhìn manifest**, rồi đối chiếu với
  `status.qosClass` thật.
- Scheduler so request với `.status.allocatable` chứ không so với mức dùng thực tế: node rảnh
  hoàn toàn vẫn có thể từ chối Pod mới.
- Đọc `kubectl describe` của một Pod `Pending` và chỉ ra đúng lý do trong `Events`.
- Quảng bá một extended resource lên một node, tiêu thụ nó từ Pod, và giải thích vì sao nó
  không ảnh hưởng QoS class.
- Resize tài nguyên của một container đang chạy tại chỗ; phân biệt tài nguyên *mong muốn* với
  tài nguyên *thực tế*, và biết khi nào container bị khởi động lại.
- Tạo PodDisruptionBudget cho một nhóm Pod trần, đọc `status`, và chứng minh PDB chặn
  Eviction API nhưng **không** chặn `kubectl delete pod`.
- Chỉ ra static Pod của control plane, giải thích vì sao xóa mirror Pod bằng `kubectl` thì nó
  quay lại, và cách duy nhất để xóa thật một static Pod.
- Cleanup toàn bộ object lab và đưa cluster về `01-cluster-ready`.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong 3c | Phần lab kiểm chứng |
| --- | --- |
| [110 — Quản lý tài nguyên cho Pod và Container](../110-manage-resources-containers-vi.md) | B1 (request/limit xuống cgroup), B2 (throttle và OOM), B4 (lập lịch theo request), B5 (extended resource) |
| [263 — Gán tài nguyên CPU cho Container và Pod](../263-assign-cpu-resource-vi.md) | B1.2 (`cpu.max`), B2.1 (throttling), B4.1 (request CPU vượt sức node) |
| [264 — Gán tài nguyên memory cho Container và Pod](../264-assign-memory-resource-vi.md) | B1.2 (`memory.max`), B2.2 (`OOMKilled`), B4.1 (request memory vượt sức node) |
| [54 — Các lớp chất lượng dịch vụ của Pod](../54-pod-qos-vi.md) | B3 (suy QoS từ manifest), B2.2 (kill ở mức container), B5.2 (extended resource không đổi QoS) |
| [288 — Cấu hình Quality of Service cho Pod](../288-quality-service-pod-vi.md) | B3 (năm dạng manifest và `status.qosClass`) |
| [284 — Gán Extended Resource cho một Container](../284-extended-resource-vi.md) | B5 (quảng bá dongle, tiêu thụ, `Pending` vì hết dongle) |
| [289 — Thay đổi kích thước tài nguyên của Container](../289-resize-container-resources-vi.md) | B6 (resize CPU không restart, resize memory có restart, `Infeasible`, QoS bất biến) |
| [53 — Sự gián đoạn](../53-disruptions-vi.md) | B7 (Eviction API bị PDB chặn, `delete` bỏ qua PDB) |
| [339 — Chỉ định Disruption Budget cho ứng dụng của bạn](../339-configure-pdb-vi.md) | B7.1 (PDB cho Pod trần, đọc `status`) |
| [58 — Pod tĩnh](../58-static-pods-vi.md) | B8 (mirror Pod control plane, static Pod thử nghiệm trên worker2) |

Hai bài thực hành của nhóm 3c **không kiểm chứng được trên cluster lab**, đọc để biết:

| Bài | Vì sao không thực hành ở đây |
| --- | --- |
| [265 — Gán tài nguyên CPU và memory ở cấp Pod](../265-assign-pod-level-resources-vi.md) | `spec.resources` cấp Pod cần feature gate `PodLevelResources`; baseline Lab 00 không bật nó — xem bảng *Đọc lướt* của bài [54](../54-pod-qos-vi.md). Bật feature gate là sửa cấu hình apiserver và kubelet, tức làm lệch baseline; việc đó thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |
| [290 — Thay đổi kích thước tài nguyên CPU và Memory của Pod](../290-resize-pod-resources-vi.md) | Cần đồng thời bốn feature gate, trong đó `InPlacePodLevelResourcesVerticalScaling` còn ở mức alpha. Cùng lý do như bài 265, và tính năng alpha thì không đưa vào lab |

Ngoài ra, hai phần dưới đây **cố ý không làm** ở lab này vì kiến thức của chúng nằm ở giai
đoạn sau — chúng không phải nợ mới, mà đúng thứ tự lộ trình đã định:

- `kubectl drain`, `cordon`, `uncordon` và việc PDB chặn drain: thuộc
  [giai đoạn 12](../00-ALO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao) và
  [giai đoạn 16](../00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node), theo đúng bảng
  *Đọc lướt* của bài [53](../53-disruptions-vi.md).
- Trục xuất do áp lực node (thứ tự `BestEffort` → `Burstable` → `Guaranteed`): thuộc
  [giai đoạn 7a](../00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction), bài
  [142](../142-node-pressure-eviction-vi.md). Ở đây chỉ đọc thứ tự, không ép node vào áp lực.

### 1.2. Thời lượng

2–3 giờ. B2 và B6 phải chờ kubelet phản ứng nên mất thêm thời gian đứng chờ; mọi vòng chờ
trong lab đều có điều kiện thoát, không có bước nào đứng chờ vô hạn.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, trừ khi ghi
  rõ node khác. Lệnh cần `sudo` để đọc file cấu hình node chạy trên chính node đó qua SSH.
- **Tuyệt đối không sửa, không xóa, không di chuyển file trong `/etc/kubernetes/manifests` trên
  `lab-k8s-master`.** B8 chỉ **đọc** thư mục đó. Một thao tác ghi nhầm ở đây làm chết control
  plane và phải restore snapshot.
- Static Pod thử nghiệm chỉ được tạo trên `lab-k8s-worker2`, và phải bị xóa ở B9.
- **Tải nặng và fault injection chỉ chạy trên `lab-k8s-worker2`**: B2 (vòng lặp CPU và OOM)
  ghim vào node này.
- Lab chỉ tạo Namespace `lab-3c`, Pod trần, một PodDisruptionBudget, một entry extended
  resource trên `lab-k8s-worker2`, và một file static Pod trên `lab-k8s-worker2`.
  **Không** tạo Deployment, ReplicaSet, StatefulSet, Service, PVC, ResourceQuota, LimitRange.
- **Không cài metrics-server.** Nó thuộc [giai đoạn 11](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability)
  và tạo snapshot `04-metrics-ready`. Vì vậy lab **không dùng `kubectl top`**; mọi con số đọc
  từ API object và từ cgroup v2 trên node — cách đọc cgroup đã học ở
  [Lab 2 B2](LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md#b2-cgroup-v2-và-cgroup-driver).
- **Không chạy `kubectl drain`, `cordon`, `uncordon`** và không gỡ taint của control plane.
- B7 gọi thẳng subresource `eviction` đúng hai lần, để kiểm chứng nguyên văn câu của bài
  [53](../53-disruptions-vi.md): PDB chỉ ràng buộc gián đoạn tự nguyện **đi qua Eviction API**.
  Trang task riêng về Eviction API là bài [143](../143-api-eviction-vi.md), đọc ở giai đoạn 7a;
  ở đây không đi sâu vào mã phản hồi hay tự động hóa bảo trì.
- Image dùng cho toàn bộ lab là `busybox:1.37` đã khóa ở
  [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và đã có sẵn trên cả ba node
  từ Lab 00, nên lab không phụ thuộc mạng ra ngoài.
- Manifest tạm ghi vào `~/lab-work/3c/`; bằng chứng ghi vào `~/lab-evidence/3c/`. Dòng bắt đầu
  bằng `PASS:` là gate; không đi tiếp khi gate fail.

---

# Phần B — Thực hành kiến thức 3c

## B0. Chuẩn bị workspace, namespace và số liệu nền của node

```bash
mkdir -p ~/lab-work/3c ~/lab-evidence/3c
kubectl config current-context
kubectl get nodes -o wide
kubectl create namespace lab-3c
kubectl get namespace lab-3c -o jsonpath='{.status.phase}'; echo
```

**Ý nghĩa:** `lab-work` chứa manifest tạm, `lab-evidence` chứa output. Namespace `lab-3c` cô
lập mọi Pod của lab; hai thứ nằm ngoài namespace — extended resource trên Node object và file
static Pod trên worker2 — được dọn riêng ở B9.

**PASS:** context trỏ đúng cluster lab; ba node `Ready`; namespace `lab-3c` ở phase `Active`.

### B0.1. `capacity` không phải `allocatable`

Bài [110](../110-manage-resources-containers-vi.md) nói lượng tài nguyên dành cho Pod **nhỏ
hơn** dung lượng node, vì daemon hệ thống đã chiếm một phần. Ghi lại con số nền trước khi làm
gì khác:

```bash
kubectl get nodes \
  -o custom-columns='NODE:.metadata.name,CPU_CAP:.status.capacity.cpu,CPU_ALLOC:.status.allocatable.cpu,MEM_CAP:.status.capacity.memory,MEM_ALLOC:.status.allocatable.memory' \
  | tee ~/lab-evidence/3c/b0-allocatable.txt

kubectl describe nodes | grep -A6 'Allocated resources' \
  | tee ~/lab-evidence/3c/b0-allocated.txt
```

```bash
BAD=0
for n in $(kubectl get nodes -o name); do
  CAPM="$(kubectl get "$n" -o jsonpath='{.status.capacity.memory}')"
  ALLM="$(kubectl get "$n" -o jsonpath='{.status.allocatable.memory}')"
  case "$CAPM" in *Ki) : ;; *) echo "FAIL: $n capacity.memory khong theo don vi Ki ($CAPM)"; BAD=1; continue ;; esac
  case "$ALLM" in *Ki) : ;; *) echo "FAIL: $n allocatable.memory khong theo don vi Ki ($ALLM)"; BAD=1; continue ;; esac
  if [ "${CAPM%Ki}" -le "${ALLM%Ki}" ]; then
    echo "FAIL: $n co allocatable memory khong nho hon capacity"
    BAD=1
  fi
done
test "$BAD" -eq 0 \
  && echo 'PASS: allocatable memory nho hon capacity tren ca ba node'
```

**Ý nghĩa:** phần chênh giữa `capacity` và `allocatable` là thứ kubelet giữ lại cho hệ thống và
cho ngưỡng eviction. Scheduler đối chiếu request với **`allocatable`**, không phải `capacity` —
B4 sẽ dùng đúng con số này để tính.

**PASS:** dòng `PASS: allocatable memory nho hon capacity tren ca ba node` xuất hiện, không có
dòng `FAIL:`.

## B1. `requests` và `limits` biến thành gì trên node

Bài [110](../110-manage-resources-containers-vi.md) tách bạch hai vai: `requests` là đầu vào
của lập lịch, `limits` là đầu vào của thực thi mà kubelet chuyển cho container runtime rồi
kernel áp bằng cgroups. Mục này nhìn thẳng vào cgroup v2 bên trong container để thấy sự tách
bạch đó.

### B1.1. Chỉ có `requests` thì không có giới hạn thực thi nào

```bash
cat > ~/lab-work/3c/b1-req-only.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: req-only
  namespace: lab-3c
spec:
  nodeName: lab-k8s-worker1
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
EOF

cat > ~/lab-work/3c/b1-req-heavy.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: req-heavy
  namespace: lab-3c
spec:
  nodeName: lab-k8s-worker1
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "400m"
        memory: "64Mi"
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/3c/b1-req-only.yaml
kubectl apply -f ~/lab-work/3c/b1-req-only.yaml
kubectl apply -f ~/lab-work/3c/b1-req-heavy.yaml
kubectl wait --for=condition=Ready pod/req-only pod/req-heavy -n lab-3c --timeout=180s
```

```bash
CPU_MAX="$(kubectl exec -n lab-3c req-only -- cat /sys/fs/cgroup/cpu.max | awk '{print $1}')"
MEM_MAX="$(kubectl exec -n lab-3c req-only -- cat /sys/fs/cgroup/memory.max | tr -d '[:space:]')"
echo "cpu.max    = $CPU_MAX"
echo "memory.max = $MEM_MAX"

test "$CPU_MAX" = 'max' && test "$MEM_MAX" = 'max' \
  && echo 'PASS: chi co requests thi kernel khong ap gioi han nao'
```

**Ý nghĩa:** hai giá trị `max` chứng minh trực tiếp câu của bài 110 — request **không** giới
hạn container. Container này được phép dùng nhiều hơn 100m CPU và nhiều hơn 64Mi RAM nếu node
còn rảnh.

**PASS:** dòng `PASS: chi co requests thi kernel khong ap gioi han nao` xuất hiện.

### B1.2. `limits` đi thẳng xuống cgroup v2

```bash
cat > ~/lab-work/3c/b1-lim-set.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: lim-set
  namespace: lab-3c
spec:
  nodeName: lab-k8s-worker1
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "200m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"
EOF

kubectl apply -f ~/lab-work/3c/b1-lim-set.yaml
kubectl wait --for=condition=Ready pod/lim-set -n lab-3c --timeout=180s
```

```bash
CPU_LINE="$(kubectl exec -n lab-3c lim-set -- cat /sys/fs/cgroup/cpu.max)"
QUOTA="$(echo "$CPU_LINE" | awk '{print $1}')"
PERIOD="$(echo "$CPU_LINE" | awk '{print $2}')"
MEM_MAX="$(kubectl exec -n lab-3c lim-set -- cat /sys/fs/cgroup/memory.max | tr -d '[:space:]')"
MILLI="$(( QUOTA * 1000 / PERIOD ))"

echo "cpu.max    = $CPU_LINE  -> $MILLI milliCPU"
echo "memory.max = $MEM_MAX byte"

test "$MILLI" -eq 200 && test "$MEM_MAX" -eq 134217728 \
  && echo 'PASS: limits.cpu thanh quota tren chu ky, limits.memory thanh memory.max'
```

**Ý nghĩa:** `200m` không phải một con số trừu tượng trong API — nó là `quota/period` của
cgroup, và `128Mi` là đúng `134217728` byte trong `memory.max`. Đây là chỗ `limits` rời khỏi
Kubernetes và trở thành ràng buộc của kernel Linux.

**PASS:** dòng `PASS: limits.cpu thanh quota tren chu ky, limits.memory thanh memory.max`
xuất hiện.

### B1.3. `requests.cpu` là trọng số khi có tranh chấp

```bash
W_LOW="$(kubectl exec -n lab-3c req-only  -- cat /sys/fs/cgroup/cpu.weight | tr -d '[:space:]')"
W_HIGH="$(kubectl exec -n lab-3c req-heavy -- cat /sys/fs/cgroup/cpu.weight | tr -d '[:space:]')"
echo "req-only  (100m) cpu.weight = $W_LOW"
echo "req-heavy (400m) cpu.weight = $W_HIGH"

test "$W_HIGH" -gt "$W_LOW" \
  && echo 'PASS: request cpu lon hon cho trong so lon hon'
```

**Ý nghĩa:** bài 110 nói request CPU "thường định nghĩa một trọng số": khi nhiều cgroup cùng
muốn chạy, workload có request lớn hơn được cấp nhiều thời gian CPU hơn. Giá trị tuyệt đối của
`cpu.weight` phụ thuộc cách runtime quy đổi, nên gate ở đây so **quan hệ lớn hơn**, không so
một con số cố định.

**PASS:** dòng `PASS: request cpu lon hon cho trong so lon hon` xuất hiện.

### B1.4. Đặt limit mà quên request thì Kubernetes tự điền

```bash
cat > ~/lab-work/3c/b1-lim-only.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: lim-only
  namespace: lab-3c
spec:
  nodeName: lab-k8s-worker1
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      limits:
        memory: "128Mi"
EOF

kubectl apply -f ~/lab-work/3c/b1-lim-only.yaml
kubectl wait --for=condition=Ready pod/lim-only -n lab-3c --timeout=180s
```

```bash
REQ_MEM="$(kubectl get pod lim-only -n lab-3c \
  -o jsonpath='{.spec.containers[0].resources.requests.memory}')"
REQ_CPU="$(kubectl get pod lim-only -n lab-3c \
  -o jsonpath='{.spec.containers[0].resources.requests.cpu}')"
CPU_MAX="$(kubectl exec -n lab-3c lim-only -- cat /sys/fs/cgroup/cpu.max | awk '{print $1}')"

echo "requests.memory = $REQ_MEM"
echo "requests.cpu    = ${REQ_CPU:-<khong co>}"
echo "cpu.max         = $CPU_MAX"

test "$REQ_MEM" = '128Mi' && test -z "$REQ_CPU" && test "$CPU_MAX" = 'max' \
  && echo 'PASS: limit duoc sao chep thanh request; chieu nguoc lai khong xay ra'

kubectl get pod req-only req-heavy lim-set lim-only -n lab-3c \
  -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName,QOS:.status.qosClass,REQ_CPU:.spec.containers[0].resources.requests.cpu,LIM_CPU:.spec.containers[0].resources.limits.cpu,REQ_MEM:.spec.containers[0].resources.requests.memory,LIM_MEM:.spec.containers[0].resources.limits.memory' \
  | tee ~/lab-evidence/3c/b1-resources.txt
```

**Ý nghĩa:** ghi chú ở mục *Yêu cầu và giới hạn* của bài 110 chạy đúng một chiều —
**limit → request**. Container này có `requests.memory: 128Mi` mà bạn không hề viết, còn CPU
thì không có request lẫn limit nào (`cpu.max` vẫn là `max`), vì bạn không khai limit CPU để mà
sao chép. So với B1.1: đặt request mà thiếu limit thì **không** có gì được sinh ra.

**PASS:** dòng `PASS: limit duoc sao chep thanh request; chieu nguoc lai khong xay ra`
xuất hiện.

## B2. Vượt limit: CPU bị điều tiết, memory bị kill

Bài [110](../110-manage-resources-containers-vi.md) nói hai loại limit hành xử khác nhau: limit
CPU thực thi bằng throttling và **không bao giờ** làm container bị chấm dứt; limit memory thực
thi **bị động** bằng OOM kill. Mục này chạy cả hai trên `lab-k8s-worker2`.

### B2.1. Vượt `limits.cpu` — bị throttle, không bị chấm dứt

```bash
cat > ~/lab-work/3c/b2-cpu-throttle.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: cpu-throttle
  namespace: lab-3c
spec:
  nodeName: lab-k8s-worker2
  containers:
  - name: burn
    image: busybox:1.37
    command: ["sh", "-c", "while :; do :; done"]
    resources:
      requests:
        cpu: "50m"
        memory: "32Mi"
      limits:
        cpu: "100m"
        memory: "64Mi"
EOF

kubectl apply -f ~/lab-work/3c/b2-cpu-throttle.yaml
kubectl wait --for=condition=Ready pod/cpu-throttle -n lab-3c --timeout=180s
```

Container chạy một vòng lặp bận, tức nó luôn muốn dùng nhiều CPU hơn `100m`. Chờ tới khi kernel
ghi nhận lần điều tiết đầu tiên — số chu kỳ cần chờ **phụ thuộc cấu hình** chu kỳ CFS của node,
nên vòng lặp dưới đây thoát theo điều kiện chứ không theo một mốc thời gian cố định:

```bash
NR_THROTTLED=0
for i in $(seq 1 24); do
  NR_THROTTLED="$(kubectl exec -n lab-3c cpu-throttle -- \
    awk '/^nr_throttled/{print $2}' /sys/fs/cgroup/cpu.stat 2>/dev/null | tr -d '[:space:]')"
  if [ -n "$NR_THROTTLED" ] && [ "$NR_THROTTLED" -gt 0 ]; then break; fi
  sleep 5
done

kubectl exec -n lab-3c cpu-throttle -- cat /sys/fs/cgroup/cpu.stat \
  | tee ~/lab-evidence/3c/b2-cpu-stat.txt
```

```bash
PHASE="$(kubectl get pod cpu-throttle -n lab-3c -o jsonpath='{.status.phase}')"
RESTARTS="$(kubectl get pod cpu-throttle -n lab-3c \
  -o jsonpath='{.status.containerStatuses[0].restartCount}')"
echo "nr_throttled = $NR_THROTTLED"
echo "phase        = $PHASE"
echo "restartCount = $RESTARTS"

test "${NR_THROTTLED:-0}" -gt 0 \
  && test "$PHASE" = 'Running' \
  && test "$RESTARTS" -eq 0 \
  && echo 'PASS: vuot limit cpu bi throttle nhung container van song'
```

**Ý nghĩa:** `nr_throttled` tăng là bằng chứng kernel đã chặn cgroup này lại trong một số chu
kỳ lập lịch. Đồng thời `restartCount` vẫn là `0` và Pod vẫn `Running` — đúng câu của bài 110:
"container runtime không chấm dứt Pod hoặc container vì sử dụng CPU quá mức".

**PASS:** dòng `PASS: vuot limit cpu bi throttle nhung container van song` xuất hiện.

### B2.2. Vượt `limits.memory` — bị `OOMKilled`, và chỉ container đó chết

```bash
cat > ~/lab-work/3c/b2-mem-oom.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: mem-oom
  namespace: lab-3c
spec:
  nodeName: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: hog
    image: busybox:1.37
    command: ["sh", "-c", "dd if=/dev/zero of=/dev/null bs=256M count=1"]
    resources:
      requests:
        cpu: "50m"
        memory: "32Mi"
      limits:
        cpu: "200m"
        memory: "64Mi"
  - name: keeper
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "50m"
        memory: "32Mi"
      limits:
        cpu: "200m"
        memory: "64Mi"
EOF

kubectl apply -f ~/lab-work/3c/b2-mem-oom.yaml
```

Container `hog` bắt `dd` cấp phát một buffer `256M` rồi đổ đầy nó bằng zero, trong khi
`limits.memory` chỉ là `64Mi`. `restartPolicy: Never` giữ nguyên hiện trường thay vì đẩy Pod
vào vòng khởi động lại.

```bash
REASON=''
for i in $(seq 1 36); do
  REASON="$(kubectl get pod mem-oom -n lab-3c \
    -o jsonpath='{.status.containerStatuses[?(@.name=="hog")].state.terminated.reason}' 2>/dev/null)"
  if [ -n "$REASON" ]; then break; fi
  sleep 5
done

kubectl get pod mem-oom -n lab-3c -o yaml > ~/lab-evidence/3c/b2-mem-oom.yaml
kubectl describe pod mem-oom -n lab-3c > ~/lab-evidence/3c/b2-mem-oom-describe.txt
grep -i -A6 'Last State\|State' ~/lab-evidence/3c/b2-mem-oom-describe.txt | head -30
```

```bash
EXIT_CODE="$(kubectl get pod mem-oom -n lab-3c \
  -o jsonpath='{.status.containerStatuses[?(@.name=="hog")].state.terminated.exitCode}')"
KEEPER_STARTED="$(kubectl get pod mem-oom -n lab-3c \
  -o jsonpath='{.status.containerStatuses[?(@.name=="keeper")].state.running.startedAt}')"
KEEPER_RESTARTS="$(kubectl get pod mem-oom -n lab-3c \
  -o jsonpath='{.status.containerStatuses[?(@.name=="keeper")].restartCount}')"

echo "hog    reason = $REASON, exitCode = $EXIT_CODE"
echo "keeper startedAt = ${KEEPER_STARTED:-<khong chay>}, restartCount = $KEEPER_RESTARTS"

test "$REASON" = 'OOMKilled' \
  && test "$EXIT_CODE" -eq 137 \
  && test -n "$KEEPER_STARTED" \
  && test "$KEEPER_RESTARTS" -eq 0 \
  && echo 'PASS: container vuot limit memory bi OOMKilled, container con lai khong bi anh huong'
```

**Ý nghĩa:** hai kết luận trong cùng một bước.

1. Bài 110 và [264](../264-assign-memory-resource-vi.md): limit memory được thực thi bằng OOM
   kill, dấu hiệu là `reason: OOMKilled` và `exitCode: 137`. Việc kill là **bị động** — nó xảy
   ra khi kernel phát hiện áp lực bộ nhớ, chứ không phải chặn ngay tại mốc `64Mi`.
2. Bài [54](../54-pod-qos-vi.md), mục *Một số hành vi không phụ thuộc vào QoS class*: container
   vượt limit bị kill **mà không ảnh hưởng các container khác** trong Pod — `keeper` vẫn
   `Running` với `restartCount` bằng 0. Đơn vị của việc kill vì vượt limit là **container**.
   Đơn vị của **trục xuất** thì là cả **Pod**, nhưng trục xuất do áp lực node thuộc giai đoạn
   7a nên lab này không dựng lại.

**PASS:** dòng `PASS: container vuot limit memory bi OOMKilled, container con lai khong bi anh huong`
xuất hiện.

### B2.3. Trả `lab-k8s-worker2` về trạng thái rảnh

B4 sẽ tính toán trên CPU còn trống của hai worker, nên phải dọn tải trước:

```bash
kubectl delete pod cpu-throttle mem-oom -n lab-3c --wait=true --timeout=120s

REMAIN="$(kubectl get pods -n lab-3c --field-selector=spec.nodeName=lab-k8s-worker2 \
  -o name | wc -l)"
test "$REMAIN" -eq 0 \
  && echo 'PASS: lab-k8s-worker2 khong con Pod nao cua lab'
```

**PASS:** dòng `PASS: lab-k8s-worker2 khong con Pod nao cua lab` xuất hiện.

## B3. Suy QoS class chỉ bằng cách nhìn manifest

Bài [54](../54-pod-qos-vi.md) nói QoS class **được suy ra**, không được khai báo. Bài
[288](../288-quality-service-pod-vi.md) đưa các dạng manifest tiêu biểu. Mục này bắt bạn **dự
đoán trước, đối chiếu sau** — xem `status.qosClass` trước thì bước này mất hết ý nghĩa.

### B3.1. Tạo năm Pod với năm cách khai báo tài nguyên khác nhau

```bash
cat > ~/lab-work/3c/b3-qos.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: qos-a
  namespace: lab-3c
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "64Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-b
  namespace: lab-3c
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "64Mi"
      limits:
        memory: "128Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-c
  namespace: lab-3c
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-d
  namespace: lab-3c
spec:
  containers:
  - name: first
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "50m"
        memory: "64Mi"
  - name: second
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-e
  namespace: lab-3c
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "100m"
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/3c/b3-qos.yaml
kubectl apply -f ~/lab-work/3c/b3-qos.yaml
kubectl wait --for=condition=Ready pod/qos-a pod/qos-b pod/qos-c pod/qos-d pod/qos-e \
  -n lab-3c --timeout=180s
```

### B3.2. Dự đoán trước khi nhìn `status`

Mở lại **manifest ở trên**, không mở `kubectl get -o yaml`. Điền đúng một trong ba giá trị
`Guaranteed`, `Burstable`, `BestEffort` vào từng dòng, không có dấu cách quanh dấu `=`:

```bash
cat > ~/lab-work/3c/b3-du-doan.txt <<'EOF'
qos-a=
qos-b=
qos-c=
qos-d=
qos-e=
EOF

vi ~/lab-work/3c/b3-du-doan.txt
cat ~/lab-work/3c/b3-du-doan.txt
```

**STOP:** không chạy bước B3.3 khi còn dòng nào bỏ trống.

### B3.3. Đối chiếu với `status.qosClass`

```bash
for p in qos-a qos-b qos-c qos-d qos-e; do
  echo "$p=$(kubectl get pod "$p" -n lab-3c -o jsonpath='{.status.qosClass}')"
done | tee ~/lab-evidence/3c/b3-thuc-te.txt

if diff -u <(sort ~/lab-work/3c/b3-du-doan.txt) <(sort ~/lab-evidence/3c/b3-thuc-te.txt); then
  echo 'PASS: du doan QoS khop hoan toan voi status.qosClass'
else
  echo 'FAIL: co it nhat mot du doan sai — doc lai muc Tieu chi cua bai 54 truoc khi di tiep'
fi
```

**Ý nghĩa:** đọc lại từng dòng theo tiêu chí của bài 54.

- `qos-a` — mọi container có đủ request và limit cho **cả CPU lẫn memory**, và bằng nhau từng
  cặp: `Guaranteed`.
- `qos-b` — có khai báo nhưng limit khác request: `Burstable`.
- `qos-c` — không container nào có bất kỳ request hay limit CPU/memory nào: `BestEffort`.
- `qos-d` — container `first` hoàn hảo nhưng `second` không khai gì. Tiêu chí `Guaranteed` áp
  cho **mọi** container, nên cả Pod tụt xuống `Burstable`. Một sidecar viết cẩu thả đủ để hạ
  QoS của cả Pod.
- `qos-e` — **câu bẫy**: CPU đã hoàn hảo, memory có request nhưng thiếu `limits.memory`. Quy
  tắc sao chép ở B1.4 chỉ chạy theo chiều limit → request, không có chiều ngược lại, nên Pod
  không đạt `Guaranteed` và rơi xuống `Burstable`.

**PASS:** dòng `PASS: du doan QoS khop hoan toan voi status.qosClass` xuất hiện. Sai dòng nào
thì đọc lại đúng tiêu chí tương ứng và giải thích được vì sao mình sai, trước khi đi tiếp.

### B3.4. Dọn Pod của B3

```bash
kubectl delete -f ~/lab-work/3c/b3-qos.yaml --wait=true --timeout=120s
kubectl get pods -n lab-3c
```

**PASS:** `kubectl get pods -n lab-3c` chỉ còn bốn Pod của B1 — `req-only`, `req-heavy`,
`lim-set`, `lim-only`.

## B4. `requests` quyết định lập lịch, không phải mức dùng thực tế

### B4.1. Request vượt sức mọi node thì `Pending` vĩnh viễn

Tính con số từ `.status.allocatable` **thật** của cluster, không bịa một giá trị:

```bash
MAX_CPU_M=0
MAX_MEM_MI=0
for n in $(kubectl get nodes -o name); do
  RAW_CPU="$(kubectl get "$n" -o jsonpath='{.status.allocatable.cpu}')"
  case "$RAW_CPU" in
    *m) V_CPU="${RAW_CPU%m}" ;;
    *)  V_CPU="$(( RAW_CPU * 1000 ))" ;;
  esac
  RAW_MEM="$(kubectl get "$n" -o jsonpath='{.status.allocatable.memory}')"
  case "$RAW_MEM" in
    *Ki) V_MEM="$(( ${RAW_MEM%Ki} / 1024 ))" ;;
    *Mi) V_MEM="${RAW_MEM%Mi}" ;;
    *)   echo "FAIL: khong doc duoc allocatable memory cua $n ($RAW_MEM)"; V_MEM=0 ;;
  esac
  echo "$n cpu=${V_CPU}m mem=${V_MEM}Mi"
  [ "$V_CPU" -gt "$MAX_CPU_M" ] && MAX_CPU_M="$V_CPU"
  [ "$V_MEM" -gt "$MAX_MEM_MI" ] && MAX_MEM_MI="$V_MEM"
done

OVER_CPU_M=$(( MAX_CPU_M + 1000 ))
OVER_MEM_MI=$(( MAX_MEM_MI + 4096 ))
echo "node lon nhat: ${MAX_CPU_M}m CPU / ${MAX_MEM_MI}Mi memory"
echo "se xin: ${OVER_CPU_M}m CPU va ${OVER_MEM_MI}Mi memory"

test "$MAX_CPU_M" -gt 0 && test "$MAX_MEM_MI" -gt 0 \
  && echo 'PASS: doc duoc allocatable that cua ca ba node'
```

Giữ nguyên shell này tới hết B6 — biến `OVER_CPU_M` còn được dùng lại ở B6.4.

```bash
cat > ~/lab-work/3c/b4-over.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: over-cpu
  namespace: lab-3c
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "${OVER_CPU_M}m"
      limits:
        cpu: "${OVER_CPU_M}m"
---
apiVersion: v1
kind: Pod
metadata:
  name: over-mem
  namespace: lab-3c
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        memory: "${OVER_MEM_MI}Mi"
      limits:
        memory: "${OVER_MEM_MI}Mi"
EOF

kubectl apply -f ~/lab-work/3c/b4-over.yaml

for i in $(seq 1 24); do
  R1="$(kubectl get pod over-cpu -n lab-3c \
    -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}' 2>/dev/null)"
  R2="$(kubectl get pod over-mem -n lab-3c \
    -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}' 2>/dev/null)"
  if [ "$R1" = 'Unschedulable' ] && [ "$R2" = 'Unschedulable' ]; then break; fi
  sleep 5
done

kubectl describe pod over-cpu -n lab-3c | tee ~/lab-evidence/3c/b4-over-cpu-describe.txt
kubectl describe pod over-mem -n lab-3c > ~/lab-evidence/3c/b4-over-mem-describe.txt
```

```bash
P1="$(kubectl get pod over-cpu -n lab-3c -o jsonpath='{.status.phase}')"
P2="$(kubectl get pod over-mem -n lab-3c -o jsonpath='{.status.phase}')"
echo "over-cpu phase = $P1 ; over-mem phase = $P2"

test "$P1" = 'Pending' && test "$P2" = 'Pending' \
  && echo 'PASS: ca hai Pod dung o Pending'
grep -q 'Insufficient cpu'    ~/lab-evidence/3c/b4-over-cpu-describe.txt \
  && echo 'PASS: describe over-cpu chi dung ly do Insufficient cpu'
grep -q 'Insufficient memory' ~/lab-evidence/3c/b4-over-mem-describe.txt \
  && echo 'PASS: describe over-mem chi dung ly do Insufficient memory'
grep -q 'FailedScheduling'    ~/lab-evidence/3c/b4-over-cpu-describe.txt \
  && echo 'PASS: co su kien FailedScheduling trong Events'
```

**Ý nghĩa:** đây đúng là tình huống mà mục *Khắc phục sự cố* của bài 110 mô tả — Pod lớn hơn
mọi node thì **không bao giờ** được lập lịch, và thêm một node giống hệt cũng không cứu được.
Trong output `describe` bạn còn thấy `lab-k8s-master` bị loại vì taint `NoSchedule` chứ không
vì thiếu tài nguyên; hai lý do loại node đó là hai chuyện khác nhau.

**PASS:** cả bốn dòng `PASS:` của bước này xuất hiện.

```bash
kubectl delete -f ~/lab-work/3c/b4-over.yaml --wait=true --timeout=120s
```

### B4.2. Node rảnh trơn vẫn từ chối Pod mới

Ba Pod dưới đây chỉ `sleep` — chúng không tiêu thụ CPU thật. Mỗi Pod xin 60% CPU khả cấp của
worker nhỏ nhất, nên hai Pod đầu chiếm hai worker, Pod thứ ba không còn chỗ:

```bash
MIN_WORKER_CPU_M=0
for n in lab-k8s-worker1 lab-k8s-worker2; do
  RAW_CPU="$(kubectl get node "$n" -o jsonpath='{.status.allocatable.cpu}')"
  case "$RAW_CPU" in
    *m) V_CPU="${RAW_CPU%m}" ;;
    *)  V_CPU="$(( RAW_CPU * 1000 ))" ;;
  esac
  if [ "$MIN_WORKER_CPU_M" -eq 0 ] || [ "$V_CPU" -lt "$MIN_WORKER_CPU_M" ]; then
    MIN_WORKER_CPU_M="$V_CPU"
  fi
done
CHUNK_M=$(( MIN_WORKER_CPU_M * 60 / 100 ))
echo "worker nho nhat: ${MIN_WORKER_CPU_M}m -> moi Pod xin ${CHUNK_M}m"

test "$CHUNK_M" -gt 0 && echo 'PASS: tinh duoc kich thuoc chunk tu allocatable that'
```

```bash
for i in 1 2 3; do
cat > ~/lab-work/3c/b4-chunk-$i.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: chunk-$i
  namespace: lab-3c
  labels:
    app: chunk
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "${CHUNK_M}m"
        memory: "32Mi"
EOF
done

kubectl apply -f ~/lab-work/3c/b4-chunk-1.yaml
kubectl wait --for=condition=Ready pod/chunk-1 -n lab-3c --timeout=180s
kubectl apply -f ~/lab-work/3c/b4-chunk-2.yaml
kubectl wait --for=condition=Ready pod/chunk-2 -n lab-3c --timeout=180s
kubectl apply -f ~/lab-work/3c/b4-chunk-3.yaml

for i in $(seq 1 24); do
  R3="$(kubectl get pod chunk-3 -n lab-3c \
    -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}' 2>/dev/null)"
  if [ "$R3" = 'Unschedulable' ]; then break; fi
  sleep 5
done

kubectl get pods -l app=chunk -n lab-3c -o wide | tee ~/lab-evidence/3c/b4-chunk.txt
kubectl describe pod chunk-3 -n lab-3c > ~/lab-evidence/3c/b4-chunk-3-describe.txt
ssh lab-k8s-worker1 'cat /proc/loadavg'
ssh lab-k8s-worker2 'cat /proc/loadavg'
kubectl describe node lab-k8s-worker1 | grep -A6 'Allocated resources'
kubectl describe node lab-k8s-worker2 | grep -A6 'Allocated resources'
```

```bash
N1="$(kubectl get pod chunk-1 -n lab-3c -o jsonpath='{.spec.nodeName}')"
N2="$(kubectl get pod chunk-2 -n lab-3c -o jsonpath='{.spec.nodeName}')"
P3="$(kubectl get pod chunk-3 -n lab-3c -o jsonpath='{.status.phase}')"
echo "chunk-1 -> $N1 ; chunk-2 -> $N2 ; chunk-3 phase = $P3"

test -n "$N1" && test -n "$N2" && test "$N1" != "$N2" \
  && echo 'PASS: hai Pod dau chiem hai worker khac nhau'
test "$P3" = 'Pending' \
  && grep -q 'Insufficient cpu' ~/lab-evidence/3c/b4-chunk-3-describe.txt \
  && echo 'PASS: Pod thu ba Pending vi het CPU da cam ket, du node dang ranh'
```

**Ý nghĩa:** hai lệnh `cat /proc/loadavg` cho thấy hai worker gần như không làm gì, còn
`Allocated resources` cho thấy phần trăm **CPU Requests** đã cao. Bài 110 nói thẳng: "mặc dù
mức sử dụng bộ nhớ hoặc CPU thực tế trên các node rất thấp, scheduler vẫn từ chối đặt một Pod
lên node nếu việc kiểm tra dung lượng thất bại". Đây chính là hình ảnh của câu đó.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B4.3. Trả lại chỗ thì Pod `Pending` tự chạy

```bash
kubectl delete pod chunk-1 -n lab-3c --wait=true --timeout=120s
kubectl wait --for=condition=Ready pod/chunk-3 -n lab-3c --timeout=180s

N3="$(kubectl get pod chunk-3 -n lab-3c -o jsonpath='{.spec.nodeName}')"
echo "chunk-3 -> $N3"
test -n "$N3" \
  && echo 'PASS: giai phong request lam Pod Pending duoc lap lich ngay'

kubectl delete pod chunk-2 chunk-3 -n lab-3c --wait=true --timeout=120s
```

**Ý nghĩa:** không có ai "đánh thức" Pod cả — scheduler liên tục thử lại các Pod chưa được lập
lịch, và điều kiện vừa đổi là **request đã cam kết trên node**, không phải tải thật.

**PASS:** dòng `PASS: giai phong request lam Pod Pending duoc lap lich ngay` xuất hiện.

## B5. Extended resource: tài nguyên do bạn tự đặt tên

Bài [110](../110-manage-resources-containers-vi.md), mục *Tài nguyên mở rộng*, mô tả đủ cả hai
vế: người vận hành quảng bá bằng một HTTP PATCH vào `/status/capacity` của Node, người dùng
tiêu thụ bằng `resources` trong Pod. Bài [284](../284-extended-resource-vi.md) là vế thứ hai.
Trang task riêng cho vế thứ nhất là bài [209](../209-extended-resource-node-vi.md), đọc ở giai
đoạn 25; ở đây chỉ dùng đúng lệnh PATCH mà bài 110 đã trình bày.

### B5.1. Quảng bá bốn dongle trên `lab-k8s-worker2`

```bash
kubectl patch node lab-k8s-worker2 --subresource=status --type=json \
  -p='[{"op": "add", "path": "/status/capacity/example.com~1dongle", "value": "4"}]'

DONGLE_ALLOC=''
for i in $(seq 1 24); do
  DONGLE_ALLOC="$(kubectl get node lab-k8s-worker2 \
    -o jsonpath='{.status.allocatable.example\.com/dongle}' 2>/dev/null)"
  if [ -n "$DONGLE_ALLOC" ]; then break; fi
  sleep 5
done

kubectl describe node lab-k8s-worker2 | grep -i dongle \
  | tee ~/lab-evidence/3c/b5-node-dongle.txt
```

```bash
DONGLE_CAP="$(kubectl get node lab-k8s-worker2 \
  -o jsonpath='{.status.capacity.example\.com/dongle}')"
echo "capacity = $DONGLE_CAP ; allocatable = $DONGLE_ALLOC"

test "$DONGLE_CAP" = '4' && test "$DONGLE_ALLOC" = '4' \
  && echo 'PASS: node quang ba 4 dongle va kubelet da cap nhat allocatable'
```

**Ý nghĩa:** `~1` trong path là cách JSON-Patch mã hóa ký tự `/`, vì path được diễn giải theo
JSON-Pointer. Kubernetes **không kiểm chứng** node có dongle thật hay không; nó chỉ ghi nhận
con số. Bạn vừa vá `capacity`, còn `allocatable` do kubelet cập nhật **bất đồng bộ** — đó là lý
do bước trên phải chờ thay vì đọc ngay.

**PASS:** dòng `PASS: node quang ba 4 dongle va kubelet da cap nhat allocatable` xuất hiện.

### B5.2. Pod tiêu thụ dongle, và dongle không đổi QoS class

```bash
cat > ~/lab-work/3c/b5-dongle-a.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dongle-a
  namespace: lab-3c
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        example.com/dongle: "3"
      limits:
        example.com/dongle: "3"
EOF

kubectl apply -f ~/lab-work/3c/b5-dongle-a.yaml
kubectl wait --for=condition=Ready pod/dongle-a -n lab-3c --timeout=180s
```

```bash
NODE_A="$(kubectl get pod dongle-a -n lab-3c -o jsonpath='{.spec.nodeName}')"
QOS_A="$(kubectl get pod dongle-a -n lab-3c -o jsonpath='{.status.qosClass}')"
echo "dongle-a -> $NODE_A, qosClass = $QOS_A"

test "$NODE_A" = 'lab-k8s-worker2' \
  && echo 'PASS: scheduler chon dung node duy nhat co dongle'
test "$QOS_A" = 'BestEffort' \
  && echo 'PASS: request tai nguyen khac CPU/memory khong lam mat tu cach BestEffort'
```

**Ý nghĩa:** Pod này không hề khai `nodeName`, nhưng scheduler chỉ có một node thỏa mãn nên nó
rơi đúng vào `lab-k8s-worker2` — extended resource tham gia lập lịch y như CPU và memory. Đồng
thời bài 54 nói rõ: "các Container trong Pod vẫn có thể yêu cầu các tài nguyên khác (không phải
CPU hay memory) mà vẫn được phân loại là `BestEffort`". Ba dongle không nâng QoS lên chút nào.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B5.3. Hết dongle thì Pod tiếp theo `Pending`

```bash
cat > ~/lab-work/3c/b5-dongle-b.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dongle-b
  namespace: lab-3c
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        example.com/dongle: "2"
      limits:
        example.com/dongle: "2"
EOF

kubectl apply -f ~/lab-work/3c/b5-dongle-b.yaml

for i in $(seq 1 24); do
  RB="$(kubectl get pod dongle-b -n lab-3c \
    -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}' 2>/dev/null)"
  if [ "$RB" = 'Unschedulable' ]; then break; fi
  sleep 5
done

kubectl describe pod dongle-b -n lab-3c > ~/lab-evidence/3c/b5-dongle-b-describe.txt
grep -A4 'Events' ~/lab-evidence/3c/b5-dongle-b-describe.txt
```

```bash
PB="$(kubectl get pod dongle-b -n lab-3c -o jsonpath='{.status.phase}')"
echo "dongle-b phase = $PB"

test "$PB" = 'Pending' \
  && grep -q 'Insufficient example.com/dongle' ~/lab-evidence/3c/b5-dongle-b-describe.txt \
  && echo 'PASS: het dongle thi Pod tiep theo dung o Pending dung ly do'
```

**PASS:** dòng `PASS: het dongle thi Pod tiep theo dung o Pending dung ly do` xuất hiện.

### B5.4. Extended resource không overcommit được

```bash
cat > ~/lab-work/3c/b5-dongle-bad.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dongle-bad
  namespace: lab-3c
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        example.com/dongle: "1"
      limits:
        example.com/dongle: "2"
EOF

if kubectl apply --dry-run=server --validate=strict -f ~/lab-work/3c/b5-dongle-bad.yaml \
     > ~/lab-evidence/3c/b5-dongle-bad.txt 2>&1; then
  echo 'FAIL: API server chap nhan request khac limit cho extended resource'
else
  cat ~/lab-evidence/3c/b5-dongle-bad.txt
  echo 'PASS: extended resource khong overcommit duoc — request phai bang limit'
fi
```

**Ý nghĩa:** bài 110 nói rõ hai ràng buộc của extended resource — số lượng phải là **số
nguyên**, và request phải **bằng** limit khi cả hai cùng xuất hiện. Bước này dùng
`--dry-run=server` nên không tạo ra object nào; chỉ tầng validation của API server trả lời.

**PASS:** dòng `PASS: extended resource khong overcommit duoc — request phai bang limit`
xuất hiện.

```bash
kubectl delete pod dongle-a dongle-b -n lab-3c --wait=true --timeout=120s
```

## B6. Resize tài nguyên của container đang chạy

Bài [289](../289-resize-container-resources-vi.md) tách hai khái niệm:
`spec.containers[*].resources` là tài nguyên **mong muốn**, còn
`status.containerStatuses[*].resources` là tài nguyên **thực tế** đang được cấu hình. Mục này
bám đúng ba ví dụ của bài.

### B6.1. Tạo Pod `Guaranteed` có `resizePolicy` tường minh

```bash
cat > ~/lab-work/3c/b6-resize.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: resize-demo
  namespace: lab-3c
spec:
  nodeName: lab-k8s-worker1
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired
    - resourceName: memory
      restartPolicy: RestartContainer
    resources:
      requests:
        cpu: "200m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"
EOF

kubectl apply -f ~/lab-work/3c/b6-resize.yaml
kubectl wait --for=condition=Ready pod/resize-demo -n lab-3c --timeout=180s
```

```bash
QOS="$(kubectl get pod resize-demo -n lab-3c -o jsonpath='{.status.qosClass}')"
RC0="$(kubectl get pod resize-demo -n lab-3c \
  -o jsonpath='{.status.containerStatuses[0].restartCount}')"
echo "qosClass = $QOS ; restartCount = $RC0"

test "$QOS" = 'Guaranteed' && test "$RC0" -eq 0 \
  && echo 'PASS: xuat phat tu Guaranteed va chua co lan khoi dong lai nao'
```

**PASS:** dòng `PASS: xuat phat tu Guaranteed va chua co lan khoi dong lai nao` xuất hiện.

### B6.2. Resize CPU — không khởi động lại

```bash
kubectl patch pod resize-demo -n lab-3c --subresource resize --patch \
  '{"spec":{"containers":[{"name":"app","resources":{"requests":{"cpu":"300m"},"limits":{"cpu":"300m"}}}]}}'

for i in $(seq 1 24); do
  ACT_CPU="$(kubectl get pod resize-demo -n lab-3c \
    -o jsonpath='{.status.containerStatuses[0].resources.limits.cpu}' 2>/dev/null)"
  if [ "$ACT_CPU" = '300m' ]; then break; fi
  sleep 5
done
```

```bash
SPEC_CPU="$(kubectl get pod resize-demo -n lab-3c \
  -o jsonpath='{.spec.containers[0].resources.limits.cpu}')"
ACT_CPU="$(kubectl get pod resize-demo -n lab-3c \
  -o jsonpath='{.status.containerStatuses[0].resources.limits.cpu}')"
RC1="$(kubectl get pod resize-demo -n lab-3c \
  -o jsonpath='{.status.containerStatuses[0].restartCount}')"
MILLI="$(kubectl exec -n lab-3c resize-demo -- cat /sys/fs/cgroup/cpu.max \
  | awk '{print $1*1000/$2}')"

echo "spec = $SPEC_CPU ; status = $ACT_CPU ; restartCount = $RC1 ; cgroup = ${MILLI}m"

test "$SPEC_CPU" = '300m' && test "$ACT_CPU" = '300m' \
  && test "$RC1" -eq 0 && test "$MILLI" -eq 300 \
  && echo 'PASS: resize cpu di den tan cgroup ma khong khoi dong lai container'
```

**Ý nghĩa:** ba con số phải trùng nhau — mong muốn, thực tế, và giá trị kernel đang áp. Việc
`restartCount` giữ nguyên là hệ quả của `resizePolicy` CPU đặt `NotRequired`.

**PASS:** dòng `PASS: resize cpu di den tan cgroup ma khong khoi dong lai container` xuất hiện.

### B6.3. Resize memory — container bị khởi động lại theo chính sách

```bash
kubectl patch pod resize-demo -n lab-3c --subresource resize --patch \
  '{"spec":{"containers":[{"name":"app","resources":{"requests":{"memory":"192Mi"},"limits":{"memory":"192Mi"}}}]}}'

for i in $(seq 1 24); do
  RC2="$(kubectl get pod resize-demo -n lab-3c \
    -o jsonpath='{.status.containerStatuses[0].restartCount}' 2>/dev/null)"
  ACT_MEM="$(kubectl get pod resize-demo -n lab-3c \
    -o jsonpath='{.status.containerStatuses[0].resources.limits.memory}' 2>/dev/null)"
  if [ "${RC2:-0}" -ge 1 ] && [ "$ACT_MEM" = '192Mi' ]; then break; fi
  sleep 5
done
kubectl wait --for=condition=Ready pod/resize-demo -n lab-3c --timeout=180s
```

```bash
ACT_MEM="$(kubectl get pod resize-demo -n lab-3c \
  -o jsonpath='{.status.containerStatuses[0].resources.limits.memory}')"
RC2="$(kubectl get pod resize-demo -n lab-3c \
  -o jsonpath='{.status.containerStatuses[0].restartCount}')"
MEM_MAX="$(kubectl exec -n lab-3c resize-demo -- cat /sys/fs/cgroup/memory.max \
  | tr -d '[:space:]')"
echo "status memory = $ACT_MEM ; restartCount = $RC2 ; memory.max = $MEM_MAX"

test "$ACT_MEM" = '192Mi' && test "$RC2" -eq 1 && test "$MEM_MAX" -eq 201326592 \
  && echo 'PASS: resize memory ap dung kem mot lan khoi dong lai container'
```

**Ý nghĩa:** cùng một Pod, cùng một cơ chế resize, nhưng kết quả khác nhau vì `resizePolicy`
của memory là `RestartContainer`. `201326592` là `192Mi` tính bằng byte.

**PASS:** dòng `PASS: resize memory ap dung kem mot lan khoi dong lai container` xuất hiện.

### B6.4. Resize bất khả thi — mong muốn đổi, thực tế không đổi

```bash
kubectl patch pod resize-demo -n lab-3c --subresource resize --patch \
  "{\"spec\":{\"containers\":[{\"name\":\"app\",\"resources\":{\"requests\":{\"cpu\":\"${OVER_CPU_M}m\"},\"limits\":{\"cpu\":\"${OVER_CPU_M}m\"}}}]}}"

for i in $(seq 1 24); do
  REASON="$(kubectl get pod resize-demo -n lab-3c \
    -o jsonpath='{.status.conditions[?(@.type=="PodResizePending")].reason}' 2>/dev/null)"
  if [ -n "$REASON" ]; then break; fi
  sleep 5
done

kubectl get pod resize-demo -n lab-3c -o yaml > ~/lab-evidence/3c/b6-infeasible.yaml
kubectl get pod resize-demo -n lab-3c \
  -o jsonpath='{.status.conditions[?(@.type=="PodResizePending")].message}{"\n"}'
```

```bash
REASON="$(kubectl get pod resize-demo -n lab-3c \
  -o jsonpath='{.status.conditions[?(@.type=="PodResizePending")].reason}')"
SPEC_CPU="$(kubectl get pod resize-demo -n lab-3c \
  -o jsonpath='{.spec.containers[0].resources.limits.cpu}')"
ACT_CPU="$(kubectl get pod resize-demo -n lab-3c \
  -o jsonpath='{.status.containerStatuses[0].resources.limits.cpu}')"
RC3="$(kubectl get pod resize-demo -n lab-3c \
  -o jsonpath='{.status.containerStatuses[0].restartCount}')"
echo "reason = $REASON ; spec = $SPEC_CPU ; status = $ACT_CPU ; restartCount = $RC3"

test "$REASON" = 'Infeasible' \
  && test "$SPEC_CPU" = "${OVER_CPU_M}m" \
  && test "$ACT_CPU" = '300m' \
  && test "$RC3" -eq 1 \
  && echo 'PASS: resize bat kha thi chi doi mong muon, khong doi thuc te'
```

Đưa Pod về giá trị khả thi trước khi đi tiếp:

```bash
kubectl patch pod resize-demo -n lab-3c --subresource resize --patch \
  '{"spec":{"containers":[{"name":"app","resources":{"requests":{"cpu":"300m"},"limits":{"cpu":"300m"}}}]}}'

for i in $(seq 1 24); do
  REASON="$(kubectl get pod resize-demo -n lab-3c \
    -o jsonpath='{.status.conditions[?(@.type=="PodResizePending")].reason}' 2>/dev/null)"
  if [ -z "$REASON" ]; then break; fi
  sleep 5
done
test -z "$REASON" && echo 'PASS: condition PodResizePending da bien mat'
```

**Ý nghĩa:** đây là lý do bài 289 bắt phân biệt hai trường. `spec` là thứ **bạn muốn**, nó nhận
mọi giá trị hợp lệ về cú pháp; `status.containerStatuses[*].resources` là thứ **kubelet đã cấp
được**. Chỉ nhìn `spec` mà kết luận resize thành công là sai.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B6.5. Resize không được đổi QoS class

```bash
if kubectl patch pod resize-demo -n lab-3c --subresource resize --patch \
     '{"spec":{"containers":[{"name":"app","resources":{"requests":{"cpu":"200m"},"limits":{"cpu":"300m"}}}]}}' \
     > ~/lab-evidence/3c/b6-qos-reject.txt 2>&1; then
  echo 'FAIL: resize da bien Pod Guaranteed thanh Burstable'
else
  cat ~/lab-evidence/3c/b6-qos-reject.txt
  echo 'PASS: tang admission tu choi resize lam doi QoS class'
fi

QOS_END="$(kubectl get pod resize-demo -n lab-3c -o jsonpath='{.status.qosClass}')"
test "$QOS_END" = 'Guaranteed' && echo 'PASS: qosClass van la Guaranteed'

kubectl delete pod resize-demo -n lab-3c --wait=true --timeout=120s
```

**Ý nghĩa:** bài 54 nói QoS class "được xác định khi Pod được tạo và giữ nguyên không đổi trong
suốt vòng đời của Pod"; bài 289 nói cụ thể hơn — với Pod `Guaranteed`, request phải tiếp tục
bằng limit sau khi resize. Tách request khỏi limit sẽ biến Pod thành `Burstable`, nên tầng
admission chặn lại.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

## B7. PodDisruptionBudget trên Pod trần

Bài [339](../339-configure-pdb-vi.md), mục *Workload tùy ý và selector tùy ý*, cho phép dùng
PDB với Pod trần kèm hai hạn chế: **chỉ** `.spec.minAvailable`, và **chỉ** giá trị số nguyên —
không dùng được `maxUnavailable` hay phần trăm, vì không có workload resource nào để Kubernetes
suy ra tổng số Pod. Cluster lab chưa học controller nên đây đúng là trường hợp phải dùng.

### B7.1. Ba Pod trần và một PDB `minAvailable: 2`

```bash
cat > ~/lab-work/3c/b7-pdb.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pdb-1
  namespace: lab-3c
  labels:
    app: pdb-demo
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "50m"
        memory: "32Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: pdb-2
  namespace: lab-3c
  labels:
    app: pdb-demo
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "50m"
        memory: "32Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: pdb-3
  namespace: lab-3c
  labels:
    app: pdb-demo
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "50m"
        memory: "32Mi"
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: pdb-demo
  namespace: lab-3c
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: pdb-demo
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/3c/b7-pdb.yaml
kubectl apply -f ~/lab-work/3c/b7-pdb.yaml
kubectl wait --for=condition=Ready pod/pdb-1 pod/pdb-2 pod/pdb-3 -n lab-3c --timeout=180s

for i in $(seq 1 24); do
  EXPECTED="$(kubectl get pdb pdb-demo -n lab-3c -o jsonpath='{.status.expectedPods}' 2>/dev/null)"
  if [ "${EXPECTED:-0}" -eq 3 ]; then break; fi
  sleep 5
done

kubectl get pdb pdb-demo -n lab-3c | tee ~/lab-evidence/3c/b7-pdb-status.txt
kubectl get pdb pdb-demo -n lab-3c -o yaml > ~/lab-evidence/3c/b7-pdb.yaml
```

```bash
EXPECTED="$(kubectl get pdb pdb-demo -n lab-3c -o jsonpath='{.status.expectedPods}')"
HEALTHY="$(kubectl get pdb pdb-demo  -n lab-3c -o jsonpath='{.status.currentHealthy}')"
DESIRED="$(kubectl get pdb pdb-demo  -n lab-3c -o jsonpath='{.status.desiredHealthy}')"
ALLOWED="$(kubectl get pdb pdb-demo  -n lab-3c -o jsonpath='{.status.disruptionsAllowed}')"
echo "expectedPods=$EXPECTED currentHealthy=$HEALTHY desiredHealthy=$DESIRED disruptionsAllowed=$ALLOWED"

test "$EXPECTED" -eq 3 && test "$HEALTHY" -eq 3 \
  && test "$DESIRED" -eq 2 && test "$ALLOWED" -eq 1 \
  && echo 'PASS: disruption controller da dem du Pod va tinh ra ngan sach 1'
```

**Ý nghĩa:** `disruptionsAllowed` khác 0 nghĩa là disruption controller đã nhìn thấy Pod, đếm
được và cập nhật `status`. Với Pod trần, `expectedPods` là số Pod khớp selector; với một
Deployment thì con số đó lấy từ `.spec.replicas` của workload resource, tìm ra qua
`.metadata.ownerReferences` của Pod — cơ chế đó chỉ kiểm chứng được từ giai đoạn 4.

Ba Pod đều `Ready` nên `currentHealthy` bằng 3: bài 339 định nghĩa Pod khỏe mạnh là Pod có
condition `Ready=True`.

**PASS:** dòng `PASS: disruption controller da dem du Pod va tinh ra ngan sach 1` xuất hiện.

### B7.2. Eviction API tiêu đúng một đơn vị ngân sách

```bash
cat > ~/lab-work/3c/b7-evict-1.json <<'EOF'
{"apiVersion": "policy/v1", "kind": "Eviction", "metadata": {"name": "pdb-1", "namespace": "lab-3c"}}
EOF

kubectl create --raw /api/v1/namespaces/lab-3c/pods/pdb-1/eviction \
  -f ~/lab-work/3c/b7-evict-1.json

for i in $(seq 1 24); do
  if ! kubectl get pod pdb-1 -n lab-3c >/dev/null 2>&1; then break; fi
  sleep 5
done

for i in $(seq 1 24); do
  ALLOWED="$(kubectl get pdb pdb-demo -n lab-3c -o jsonpath='{.status.disruptionsAllowed}')"
  if [ "${ALLOWED:-1}" -eq 0 ]; then break; fi
  sleep 5
done
kubectl get pdb pdb-demo -n lab-3c | tee -a ~/lab-evidence/3c/b7-pdb-status.txt
```

```bash
HEALTHY="$(kubectl get pdb pdb-demo -n lab-3c -o jsonpath='{.status.currentHealthy}')"
ALLOWED="$(kubectl get pdb pdb-demo -n lab-3c -o jsonpath='{.status.disruptionsAllowed}')"
echo "currentHealthy=$HEALTHY disruptionsAllowed=$ALLOWED"

kubectl get pod pdb-1 -n lab-3c >/dev/null 2>&1 \
  && echo 'FAIL: pdb-1 van con' \
  || echo 'PASS: eviction dau tien duoc cho phep va Pod da bien mat'
test "$HEALTHY" -eq 2 && test "$ALLOWED" -eq 0 \
  && echo 'PASS: ngan sach gian doan da can'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B7.3. Eviction thứ hai bị PDB từ chối

```bash
cat > ~/lab-work/3c/b7-evict-2.json <<'EOF'
{"apiVersion": "policy/v1", "kind": "Eviction", "metadata": {"name": "pdb-2", "namespace": "lab-3c"}}
EOF

if kubectl create --raw /api/v1/namespaces/lab-3c/pods/pdb-2/eviction \
     -f ~/lab-work/3c/b7-evict-2.json > ~/lab-evidence/3c/b7-evict-2.txt 2>&1; then
  echo 'FAIL: PDB khong chan duoc eviction thu hai'
else
  cat ~/lab-evidence/3c/b7-evict-2.txt
  echo 'PASS: Eviction API bi PDB tu choi'
fi

kubectl get pod pdb-2 -n lab-3c -o jsonpath='{.status.phase}{"\n"}'
kubectl get pod pdb-2 -n lab-3c >/dev/null 2>&1 \
  && echo 'PASS: pdb-2 van con nguyen vi yeu cau bi tu choi'
```

**Ý nghĩa:** đây là toàn bộ giá trị của PDB — nó **từ chối** một gián đoạn tự nguyện sẽ làm số
Pod khỏe mạnh tụt xuống dưới `minAvailable`. Công cụ bảo trì cư xử đúng mực (ví dụ
`kubectl drain`, học ở giai đoạn 12 và 16) đi qua đúng đường này và sẽ thử lại cho tới khi
ngân sách hồi lại hoặc hết timeout.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B7.4. `kubectl delete pod` bỏ qua PDB

```bash
kubectl delete pod pdb-2 -n lab-3c --wait=true --timeout=120s

for i in $(seq 1 24); do
  HEALTHY="$(kubectl get pdb pdb-demo -n lab-3c -o jsonpath='{.status.currentHealthy}')"
  if [ "${HEALTHY:-2}" -le 1 ]; then break; fi
  sleep 5
done
kubectl get pdb pdb-demo -n lab-3c | tee -a ~/lab-evidence/3c/b7-pdb-status.txt
```

```bash
HEALTHY="$(kubectl get pdb pdb-demo -n lab-3c -o jsonpath='{.status.currentHealthy}')"
DESIRED="$(kubectl get pdb pdb-demo -n lab-3c -o jsonpath='{.status.desiredHealthy}')"
echo "currentHealthy=$HEALTHY desiredHealthy=$DESIRED"

kubectl get pod pdb-2 -n lab-3c >/dev/null 2>&1 \
  && echo 'FAIL: pdb-2 van con sau khi delete' \
  || echo 'PASS: DELETE truc tiep xoa duoc Pod ma PDB vua tu choi evict'
test "$HEALTHY" -lt "$DESIRED" \
  && echo 'PASS: ngan sach da bi thung — PDB khong bao ve truoc kubectl delete'
```

**Ý nghĩa:** hai bước liên tiếp trên cùng một Pod, hai kết quả ngược nhau. Đó chính là khối
*Thận trọng* của bài [53](../53-disruptions-vi.md): "việc xóa deployment hoặc pod sẽ **bỏ qua**
Pod Disruption Budget". PDB bảo vệ khỏi tự động hóa cư xử đúng mực, không bảo vệ khỏi một lệnh
`delete` gõ tay. Gián đoạn **không tự nguyện** — kernel panic, mất node — cũng không bị PDB
ngăn, nhưng vẫn được tính vào ngân sách; lab này không dựng lại tình huống đó vì nó đòi hỏi
phá node.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

```bash
kubectl delete -f ~/lab-work/3c/b7-pdb.yaml --ignore-not-found=true --wait=true --timeout=120s
```

## B8. Static Pod và mirror Pod

### B8.1. Kiểm kê static Pod của control plane — chỉ đọc

```bash
sudo ls -l /etc/kubernetes/manifests/
sudo awk '/^staticPodPath:/{print}' /var/lib/kubelet/config.yaml

kubectl -n kube-system get pods \
  -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName,MIRROR:.metadata.annotations.kubernetes\.io/config\.mirror' \
  | tee ~/lab-evidence/3c/b8-mirror-pods.txt
```

```bash
FILE_COUNT="$(sudo ls -1 /etc/kubernetes/manifests/ | grep -c '\.yaml$')"
MIRROR_COUNT="$(kubectl -n kube-system get pods \
  -o jsonpath='{range .items[*]}{.metadata.annotations.kubernetes\.io/config\.mirror}{"\n"}{end}' \
  | grep -c '.')"
echo "file manifest = $FILE_COUNT ; mirror Pod = $MIRROR_COUNT"

test "$FILE_COUNT" -eq 4 && test "$MIRROR_COUNT" -eq 4 \
  && echo 'PASS: bon file manifest ung voi dung bon mirror Pod'

MISSING=0
for c in etcd kube-apiserver kube-controller-manager kube-scheduler; do
  kubectl -n kube-system get pod "$c-lab-k8s-master" >/dev/null 2>&1 \
    || { echo "FAIL: khong thay mirror Pod $c-lab-k8s-master"; MISSING=1; }
done
test "$MISSING" -eq 0 \
  && echo 'PASS: ten mirror Pod = ten file manifest cong hostname node'
```

**Ý nghĩa:** bốn thành phần control plane mà bạn đã kiểm kê ở
[Lab 1a](LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md) chính là static Pod. Annotation
`kubernetes.io/config.mirror` là dấu hiệu nhận diện; hậu tố `-lab-k8s-master` trong tên là
hostname của node, đúng quy tắc đặt tên mirror Pod trong bài [58](../58-static-pods-vi.md).

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B8.2. Xóa mirror Pod bằng `kubectl` không đụng tới tiến trình thật

Chọn `kube-scheduler` chứ không phải `kube-apiserver` hay `etcd`: nếu có gì bất thường, một
scheduler khởi động lại là vô hại, còn hai thành phần kia thì không.

```bash
CID_BEFORE="$(kubectl -n kube-system get pod kube-scheduler-lab-k8s-master \
  -o jsonpath='{.status.containerStatuses[0].containerID}')"
RC_BEFORE="$(kubectl -n kube-system get pod kube-scheduler-lab-k8s-master \
  -o jsonpath='{.status.containerStatuses[0].restartCount}')"
STARTED_BEFORE="$(kubectl -n kube-system get pod kube-scheduler-lab-k8s-master \
  -o jsonpath='{.status.containerStatuses[0].state.running.startedAt}')"
echo "truoc: containerID=$CID_BEFORE restartCount=$RC_BEFORE startedAt=$STARTED_BEFORE"

kubectl -n kube-system delete pod kube-scheduler-lab-k8s-master --wait=false

for i in $(seq 1 36); do
  if kubectl -n kube-system get pod kube-scheduler-lab-k8s-master >/dev/null 2>&1; then
    DT="$(kubectl -n kube-system get pod kube-scheduler-lab-k8s-master \
      -o jsonpath='{.metadata.deletionTimestamp}')"
    if [ -z "$DT" ]; then break; fi
  fi
  sleep 5
done
```

```bash
CID_AFTER="$(kubectl -n kube-system get pod kube-scheduler-lab-k8s-master \
  -o jsonpath='{.status.containerStatuses[0].containerID}')"
RC_AFTER="$(kubectl -n kube-system get pod kube-scheduler-lab-k8s-master \
  -o jsonpath='{.status.containerStatuses[0].restartCount}')"
STARTED_AFTER="$(kubectl -n kube-system get pod kube-scheduler-lab-k8s-master \
  -o jsonpath='{.status.containerStatuses[0].state.running.startedAt}')"
echo "sau:   containerID=$CID_AFTER restartCount=$RC_AFTER startedAt=$STARTED_AFTER"

{ echo "before $CID_BEFORE $RC_BEFORE $STARTED_BEFORE"
  echo "after  $CID_AFTER $RC_AFTER $STARTED_AFTER"; } \
  | tee ~/lab-evidence/3c/b8-mirror-delete.txt

test -n "$CID_AFTER" \
  && echo 'PASS: mirror Pod quay lai tren API server'
test "$CID_AFTER" = "$CID_BEFORE" \
  && test "$RC_AFTER" -eq "$RC_BEFORE" \
  && test "$STARTED_AFTER" = "$STARTED_BEFORE" \
  && echo 'PASS: container that khong he bi dung lai — chi object tren API bi xoa'
```

**Ý nghĩa:** `containerID` và `startedAt` không đổi là bằng chứng mạnh nhất: tiến trình
scheduler chưa từng dừng. Thứ bạn xóa chỉ là **bản phản chiếu** trên API server, và kubelet tạo
lại nó. Static Pod gắn với kubelet của đúng node đó, không có control plane nào quan sát nó.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B8.3. Static Pod thử nghiệm trên `lab-k8s-worker2`

Cách duy nhất để thật sự xóa một static Pod là xóa file manifest trên node. Chứng minh điều đó
bằng một Pod của riêng lab, trên node fault injection — **không** bao giờ trên `lab-k8s-master`.

```bash
SPP="$(ssh lab-k8s-worker2 "sudo awk '/^staticPodPath:/{print \$2}' /var/lib/kubelet/config.yaml")"
echo "staticPodPath tren lab-k8s-worker2 = $SPP"
test "$SPP" = '/etc/kubernetes/manifests' \
  && echo 'PASS: kubelet worker2 doc static Pod tu /etc/kubernetes/manifests'
```

Ghi lại thư mục manifest trên worker đã tồn tại từ trước hay do lab tạo ra, để B9 trả đúng
trạng thái cũ:

```bash
ssh lab-k8s-worker2 'test -d /etc/kubernetes/manifests && echo existed || echo created' \
  | tee ~/lab-evidence/3c/b8-manifests-dir-before.txt

ssh lab-k8s-worker2 'sudo mkdir -p /etc/kubernetes/manifests'
ssh lab-k8s-worker2 'sudo tee /etc/kubernetes/manifests/lab3c-static.yaml >/dev/null' <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: lab3c-static
  labels:
    app: lab3c-static
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "50m"
        memory: "32Mi"
EOF

for i in $(seq 1 36); do
  if kubectl get pod lab3c-static-lab-k8s-worker2 -n default >/dev/null 2>&1; then break; fi
  sleep 5
done
kubectl get pods -n default -o wide | tee ~/lab-evidence/3c/b8-static-worker2.txt
kubectl get pods -A -l app=lab3c-static -o wide
```

```bash
MIRROR="$(kubectl get pod lab3c-static-lab-k8s-worker2 -n default \
  -o jsonpath='{.metadata.annotations.kubernetes\.io/config\.mirror}' 2>/dev/null)"
NODE="$(kubectl get pod lab3c-static-lab-k8s-worker2 -n default \
  -o jsonpath='{.spec.nodeName}' 2>/dev/null)"
LABELLED="$(kubectl get pods -A -l app=lab3c-static -o name | wc -l)"
echo "mirror annotation = ${MIRROR:-<khong co>} ; node = $NODE ; so Pod khop label = $LABELLED"

test -n "$MIRROR" && test "$NODE" = 'lab-k8s-worker2' && test "$LABELLED" -eq 1 \
  && echo 'PASS: kubelet tao mirror Pod, giu label va dat ten theo hostname node'
```

**Ý nghĩa:** bạn không hề chạy `kubectl apply`, mà Pod vẫn xuất hiện trong cluster — vì kubelet
đọc thẳng file. Label được lan truyền sang mirror Pod nên selector vẫn dùng được bình thường.
Mirror Pod này nằm ở namespace `default`, không phải `kube-system`: namespace của static Pod
control plane là do chính manifest của kubeadm quy định, không phải đặc quyền của static Pod.

Static Pod này cố tình **không** tham chiếu ConfigMap, Secret hay ServiceAccount, vì mục *Giới
hạn* của bài 58 nói spec static Pod không tham chiếu được đối tượng API khác — kubelet phải
khởi động được nó trước cả khi API server sẵn sàng.

**PASS:** hai dòng `PASS:` của mục B8.3 xuất hiện.

### B8.4. Xóa file manifest mới thật sự xóa static Pod

```bash
ssh lab-k8s-worker2 'sudo rm -f /etc/kubernetes/manifests/lab3c-static.yaml'
ssh lab-k8s-worker2 'sudo ls -1 /etc/kubernetes/manifests/'

for i in $(seq 1 36); do
  if ! kubectl get pod lab3c-static-lab-k8s-worker2 -n default >/dev/null 2>&1; then break; fi
  sleep 5
done
```

```bash
kubectl get pod lab3c-static-lab-k8s-worker2 -n default >/dev/null 2>&1 \
  && echo 'FAIL: mirror Pod van con sau khi xoa file manifest' \
  || echo 'PASS: xoa file manifest tren node moi that su xoa static Pod'

LEFT="$(ssh lab-k8s-worker2 'sudo ls -1 /etc/kubernetes/manifests/ | grep -c lab3c || true')"
test "${LEFT:-0}" -eq 0 \
  && echo 'PASS: khong con file static Pod cua lab tren lab-k8s-worker2'
```

**Ý nghĩa:** so bước này với B8.2 — `kubectl delete` xóa mirror Pod và kubelet dựng lại; xóa
file thì mirror Pod biến mất luôn. Nguồn sự thật của static Pod nằm ở **file trên node**, không
nằm ở API server. Đây cũng là lý do bài 58 khuyên dùng DaemonSet cho workload cấp node:
DaemonSet rollout, rollback và co giãn được, còn static Pod thì phải sửa tay trên từng node.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

## B9. Cleanup và gate cuối

**Mục đích:** xóa mọi thứ lab tạo ra — trong namespace, trên Node object, và trên filesystem
của `lab-k8s-worker2` — rồi chứng minh cluster trở về `01-cluster-ready`.

```bash
kubectl delete namespace lab-3c --wait=true --timeout=180s

kubectl patch node lab-k8s-worker2 --subresource=status --type=json \
  -p='[{"op": "remove", "path": "/status/capacity/example.com~1dongle"}]'

for i in $(seq 1 24); do
  LEFT="$(kubectl describe node lab-k8s-worker2 | grep -c -i dongle || true)"
  if [ "${LEFT:-1}" -eq 0 ]; then break; fi
  sleep 5
done

ssh lab-k8s-worker2 'sudo rm -f /etc/kubernetes/manifests/lab3c-static.yaml'
if grep -qx 'created' ~/lab-evidence/3c/b8-manifests-dir-before.txt; then
  ssh lab-k8s-worker2 'sudo rmdir /etc/kubernetes/manifests'
fi

rm -f ~/lab-work/3c/b1-req-only.yaml ~/lab-work/3c/b1-req-heavy.yaml \
      ~/lab-work/3c/b1-lim-set.yaml ~/lab-work/3c/b1-lim-only.yaml \
      ~/lab-work/3c/b2-cpu-throttle.yaml ~/lab-work/3c/b2-mem-oom.yaml \
      ~/lab-work/3c/b3-qos.yaml ~/lab-work/3c/b3-du-doan.txt \
      ~/lab-work/3c/b4-over.yaml ~/lab-work/3c/b4-chunk-1.yaml \
      ~/lab-work/3c/b4-chunk-2.yaml ~/lab-work/3c/b4-chunk-3.yaml \
      ~/lab-work/3c/b5-dongle-a.yaml ~/lab-work/3c/b5-dongle-b.yaml \
      ~/lab-work/3c/b5-dongle-bad.yaml ~/lab-work/3c/b6-resize.yaml \
      ~/lab-work/3c/b7-pdb.yaml ~/lab-work/3c/b7-evict-1.json ~/lab-work/3c/b7-evict-2.json
rmdir ~/lab-work/3c
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` bên dưới biến
điều đó thành gate thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/3c/` **giữ lại** — đó là
bằng chứng.

```bash
kubectl get namespace lab-3c >/dev/null 2>&1 \
  && echo 'FAIL: namespace lab-3c van con' \
  || echo 'PASS: namespace lab-3c da xoa'

kubectl describe node lab-k8s-worker2 | grep -i dongle \
  && echo 'FAIL: node van con quang ba dongle' \
  || echo 'PASS: extended resource da go khoi lab-k8s-worker2'

STATIC_LEFT="$(ssh lab-k8s-worker2 'sudo ls -1 /etc/kubernetes/manifests/ 2>/dev/null | wc -l')"
test "$STATIC_LEFT" -eq 0 \
  && echo 'PASS: lab-k8s-worker2 khong con file static Pod nao'

CP_FILES="$(sudo ls -1 /etc/kubernetes/manifests/ | grep -c '\.yaml$')"
test "$CP_FILES" -eq 4 \
  && echo 'PASS: bon file manifest control plane con nguyen ven'

test ! -e ~/lab-work/3c && echo 'PASS: manifest tam da xoa'

kubectl wait --for=condition=Ready node --all --timeout=180s
kubectl get nodes
kubectl get pods -n default
kubectl get pdb -A
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl -n kube-system get pods -o wide
```

**PASS:** không có dòng `FAIL:` nào; năm dòng `PASS:` của bước này xuất hiện; ba node `Ready`;
`kubectl get pods -n default` trả `No resources found in default namespace.`;
`kubectl get pdb -A` không còn PDB nào; lệnh field selector trả `No resources found`; CoreDNS
đủ replica `READY`; bốn Pod control plane vẫn `Running` với `RESTARTS` không tăng so với đầu
lab. Cluster trở về `01-cluster-ready`; **không tạo snapshot mới**.

---

## 3. Checkpoint 3c

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] `requests` nói chuyện với thành phần nào của Kubernetes, `limits` nói chuyện với thành
      phần nào? Kể đường đi của giá trị `200m` từ manifest tới kernel.
- [ ] Trên cgroup v2, `limits.cpu`, `limits.memory` và `requests.cpu` biến thành ba trường
      nào? Container không có limit thì các trường đó mang giá trị gì?
- [ ] Bạn đặt `limits.memory: 128Mi` và không viết `requests` gì cả. Request memory bằng bao
      nhiêu? Làm ngược lại — chỉ đặt request, không đặt limit — thì có gì được sinh ra không?
- [ ] Container A vượt `limits.cpu`, container B vượt `limits.memory`. Cái nào bị chấm dứt,
      cái nào không, và vì sao?
- [ ] Một container trong Pod ba container bị `OOMKilled`. Hai container còn lại có sao không?
      Câu trả lời khác đi thế nào nếu cả Pod bị **trục xuất** do node chịu áp lực?
- [ ] Một Pod có `requests.cpu` bằng `limits.cpu`, `requests.memory` có nhưng thiếu
      `limits.memory`. QoS class là gì, và vì sao trực giác "CPU đã chặt rồi" lại sai?
- [ ] Pod hai container, container thứ hai không khai tài nguyên nào. QoS class của **Pod** là
      gì và vì sao?
- [ ] `kubectl describe node` cho thấy CPU thực dùng gần bằng 0 nhưng Pod mới vẫn `Pending`.
      Scheduler đang so request với con số nào của node? Thêm một worker giống hệt có cứu được
      một Pod xin nhiều CPU hơn mọi node không?
- [ ] Sau khi `kubectl patch ... --subresource resize`, trường nào cho biết bạn **muốn** gì và
      trường nào cho biết kubelet **đã cấp** được gì? `PodResizePending` kèm `reason:
      Infeasible` nghĩa là gì? Resize có đổi được QoS class không?
- [ ] Ai quảng bá một extended resource, Kubernetes có kiểm chứng node thật sự có nó không, và
      request với limit của nó phải quan hệ thế nào? Nó có làm Pod thoát khỏi `BestEffort`
      không?
- [ ] PDB chặn được loại gián đoạn nào và **không** chặn được loại nào? Số Pod "dự kiến" lấy từ
      đâu khi Pod thuộc một controller, và khi là Pod trần? Gián đoạn không tự nguyện có bị trừ
      vào ngân sách không?
- [ ] Vì sao xóa mirror Pod của `kube-scheduler` bằng `kubectl` thì nó quay lại mà tiến trình
      không hề dừng? Cách duy nhất để xóa thật một static Pod là gì? Vì sao kubeadm chạy
      `kube-apiserver` bằng static Pod chứ không phải DaemonSet?

### Bài giải thích cuối cùng

Trong tối đa 10 phút, kể lại hành trình của hai con số trong một manifest:

1. `requests.cpu: 200m` đi tới scheduler và được so với `.status.allocatable` của từng node —
   và vì sao mức dùng thực tế của node không tham gia phép so đó.
2. `limits.cpu: 200m` đi tới kubelet, tới container runtime, rồi thành `cpu.max` của cgroup;
   vượt nó thì bị throttle chứ không bị kill.
3. `limits.memory` cũng đi đường ấy nhưng thành `memory.max`, và vượt nó thì kernel OOM kill —
   ở mức container, không phải mức Pod.
4. Quan hệ giữa hai con số quyết định QoS class, và QoS class chốt lại lúc Pod được tạo.
5. Muốn đổi hai con số đó trên một Pod đang chạy thì dùng subresource `resize`, và những gì
   kubelet không cấp được sẽ đọng lại ở condition chứ không lặng lẽ trôi qua.
6. Khi ai đó muốn gỡ Pod ra để bảo trì node, PDB là thứ đứng giữa — nhưng chỉ đứng giữa những
   yêu cầu đi qua Eviction API.
7. Và có một loại Pod mà toàn bộ câu chuyện API ở trên không áp dụng: static Pod, nơi kubelet
   đọc thẳng file trên node.

Khi mọi checkbox được đánh dấu và không còn nhầm request với limit, throttle với OOM kill, QoS
class với độ ưu tiên lập lịch, hay mirror Pod với static Pod, Lab 3c và
[giai đoạn 3](../00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình) hoàn tất.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Bảng dưới chỉ
liệt kê sự cố phát sinh từ nội dung bài học 3c.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| `cat /sys/fs/cgroup/cpu.max` trong container báo không có file | `stat -fc %T /sys/fs/cgroup/` trên node phải là `cgroup2fs` — xem [Lab 2 B2](LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md#b2-cgroup-v2-và-cgroup-driver) | Nếu node đúng cgroup v2 mà container không thấy, đọc giá trị từ node bằng `sudo crictl inspect <container-id>` rồi so với manifest; không sửa cấu hình containerd |
| `nr_throttled` mãi bằng 0 ở B2.1 | `kubectl exec ... -- cat /sys/fs/cgroup/cpu.max` phải khác `max`; Pod phải `Running` | Chờ thêm vài vòng lặp — số chu kỳ cần để kernel ghi nhận **phụ thuộc cấu hình** chu kỳ CFS; nếu `cpu.max` là `max` thì manifest thiếu `limits.cpu` |
| B2.2 không bao giờ đạt `OOMKilled` | `swapon --show` trên `lab-k8s-worker2` phải rỗng; `memory.max` của container `hog` phải là `67108864` | Swap bật lại làm kernel đẩy trang ra swap thay vì OOM — quay lại [gate A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready). Nếu swap đã tắt, tăng `bs` của `dd` trong manifest |
| `chunk-3` vẫn `Running` chứ không `Pending` | In lại `MIN_WORKER_CPU_M` và `CHUNK_M`; `kubectl describe node` xem `CPU Requests` | Worker có nhiều CPU hơn dự kiến nên hai chunk vẫn vừa một node — tăng tỉ lệ trong `CHUNK_M` (ví dụ 70) rồi chạy lại B4.2 |
| `chunk-1` và `chunk-2` rơi vào cùng một node | `kubectl get pods -l app=chunk -o wide` | Chunk quá nhỏ so với node — tăng tỉ lệ như trên; **không** dùng `nodeName`, vì `nodeName` bỏ qua scheduler và làm hỏng phép thử |
| `kubectl patch node ... --subresource=status` báo cờ không hợp lệ | `kubectl version` | Client quá cũ so với baseline; thay bằng cách `kubectl proxy` cộng `curl` mà bài [110](../110-manage-resources-containers-vi.md) và [209](../209-extended-resource-node-vi.md) mô tả, giữ nguyên nội dung JSON-Patch |
| `allocatable` không hiện dongle sau khi PATCH | `kubectl get node lab-k8s-worker2 -o yaml` phần `status.capacity` | `capacity` do bạn vá còn `allocatable` do kubelet cập nhật **bất đồng bộ**; chờ hết vòng lặp. Nếu `capacity` cũng không có, path JSON-Patch viết sai chỗ `~1` |
| `kubectl patch ... --subresource resize` báo `invalid subresource` | `kubectl version` — client phải mới hơn mốc mà bài [289](../289-resize-container-resources-vi.md) nêu | Dùng `kubectl` của baseline trên `lab-k8s-master`, không dùng client cài riêng |
| Resize không đổi `status` và không có condition nào | `kubectl get pod resize-demo -n lab-3c -o yaml`, phần `status.conditions` | Đọc `message` của `PodResizePending`; nếu `reason` là `Deferred` thì node đang thiếu chỗ — dọn Pod thừa của B1 rồi thử lại |
| `disruptionsAllowed` mãi bằng 0 ở B7.1 | `kubectl get pods -l app=pdb-demo -n lab-3c` phải đủ ba Pod `Ready` | PDB chỉ đếm Pod có condition `Ready=True`; chờ đủ ba Pod Ready rồi đọc lại `status` |
| Eviction đầu tiên đã bị từ chối | `kubectl get pdb pdb-demo -n lab-3c -o yaml` | Ngân sách cạn từ trước vì một Pod chưa `Ready`; đợi `currentHealthy` bằng 3 rồi chạy lại B7.2 |
| B8.2 cho thấy `containerID` đổi | `kubectl -n kube-system describe pod kube-scheduler-lab-k8s-master`; `sudo journalctl -u kubelet -n 50` trên master | Kết luận của bài không đổi — `kubectl` vẫn không xóa được static Pod. Ghi hiện tượng vào evidence và **không** đụng vào file manifest của control plane |
| Static Pod ở B8.3 không xuất hiện | `ssh lab-k8s-worker2 'sudo journalctl -u kubelet -n 50'`; giá trị `staticPodPath` in ở đầu B8.3 | Đặt file đúng đường dẫn mà kubelet khai báo; nếu YAML sai cú pháp, kubelet ghi lỗi vào log và không tạo Pod nào |
| Namespace `lab-3c` kẹt `Terminating` | `kubectl get pods -n lab-3c -o wide` | Còn Pod trong grace period; chờ hết `terminationGracePeriodSeconds`. Nếu vẫn kẹt, xem `kubectl get ns lab-3c -o yaml` — không cưỡng chế finalizer namespace |

---

## 5. Nguồn chính thức

- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Pod Quality of Service Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)
- [Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Static Pods](https://kubernetes.io/docs/concepts/workloads/pods/static-pods/)
- [Assign CPU Resources to Containers and Pods](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)
- [Assign Memory Resources to Containers and Pods](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)
- [Assign Pod-level CPU and memory resources](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pod-level-resources/)
- [Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
- [Assign Extended Resources to a Container](https://kubernetes.io/docs/tasks/configure-pod-container/extended-resource/)
- [Resize CPU and Memory Resources assigned to Containers](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/)
- [Resize CPU and Memory Resources assigned to Pods](https://kubernetes.io/docs/tasks/configure-pod-container/resize-pod-resources/)
- [Specifying a Disruption Budget for your Application](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
