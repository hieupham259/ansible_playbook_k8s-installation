# Lab 4b — StatefulSet, DaemonSet và Job

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem
> [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> **Điểm kết thúc:** cleanup trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới.
> **Lab trước:** [Lab 4a — ReplicaSet, Deployment và rollout](LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md)
> đã cleanup namespace `lab-4a` và trả cluster về `01-cluster-ready`, nên điểm bắt đầu của lab này
> là một cluster sạch không workload.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục
[4b. StatefulSet, DaemonSet, Job và autoscaling](../00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling).
Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không chép
lại con số phiên bản nào** và **không cài thêm gì**.

Lab dùng Deployment và ReplicaSet của
[4a](../00-ALO-TRINH-ADMIN.md#4a-replicaset-deployment-và-rollout) làm mốc đối chiếu, và dùng Pod,
probe, `restartPolicy`, hook `PreStop` của giai đoạn 2–3 làm công cụ. Không dùng Service, PVC,
StorageClass, HPA hay NetworkPolicy — xem [ba món nợ](#ba-món-nợ-của-nhóm-4b) ngay bên dưới.

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

- StatefulSet cấp cho mỗi Pod một **định danh cố định** `<tên>-<ordinal>` đi theo Pod chứ không đi
  theo node, và định danh đó xuất hiện ở tên Pod, hostname và hai label do controller gắn.
- Với `podManagementPolicy` mặc định `OrderedReady`, Pod được tạo tuần tự **0 → N-1** và Pod sau chỉ
  xuất hiện sau khi Pod trước đã Running **và Ready**; khi thu nhỏ thì thứ tự **ngược lại**.
- Khác biệt so với Deployment ở 4a: Deployment giữ **số lượng** replica, StatefulSet giữ **danh
  tính** của từng replica.
- Xóa êm thấm và **xóa cưỡng bức** một Pod của StatefulSet khác nhau ở chỗ nào, và vì sao xóa cưỡng
  bức phá vỡ ngữ nghĩa "nhiều nhất một".
- `--cascade=orphan` để lại Pod sau khi object điều khiển đã biến mất.
- DaemonSet đặt **một Pod trên mỗi node đủ điều kiện**; con số đó suy ra được từ số node và taint,
  và đổi khi bạn thêm toleration.
- DaemonSet controller **không tự bind Pod**: nó ghi `nodeAffinity` khớp host rồi để scheduler đặt
  `.spec.nodeName`; controller cũng tự thêm một bộ toleration mà bạn không viết.
- Ranh giới tự phục hồi: **kubelet** khởi động lại container **bên trong** Pod; **controller** tạo
  Pod **mới** khi cả Pod mất — và Pod thay thế của DaemonSet phải lên **đúng node cũ**.
- Ba loại tác vụ Job và cách đặt trường cho từng loại: không song song, số lần hoàn thành cố định,
  hàng đợi công việc; cộng chế độ `Indexed`.
- `backoffLimit` chặn Job ở đâu, `restartPolicy: Never` và `OnFailure` khác nhau thế nào, và vì sao
  Job thất bại **không tự chạy lại**.
- `ttlSecondsAfterFinished` bắt đầu đếm từ lúc Job `Complete`/`Failed`, và xóa Job **theo tầng**.
- CronJob chỉ tạo Job theo lịch; `concurrencyPolicy: Forbid` chặn lần chạy chồng **trong phạm vi
  một CronJob**, và `startingDeadlineSeconds` đổi cách đếm lịch bị lỡ.
- Cleanup toàn bộ object lab — kể cả Job do CronJob sinh ra — và đưa cluster về `01-cluster-ready`.

### Ba món nợ của nhóm 4b

Lab này **cố ý không làm** ba phần dưới đây. Chúng nằm trong
[sổ nợ lab](README.md#5-sổ-nợ-lab) và [Sổ nợ lộ trình](../00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình).

| # | Nợ | Vì cần | Trả ở |
| --- | --- | --- | --- |
| 2 | `volumeClaimTemplates` của StatefulSet | StorageClass và provisioner của [giai đoạn 6](../00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) | Lab 6a |
| 3 | Service headless quản trị cho StatefulSet | bài Service của [giai đoạn 5](../00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) | Lab 5a |
| 1 | Thực hành HPA và VPA | metrics-server của [giai đoạn 11](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability) | Lab 11b |

**Không** dựng StorageClass, **không** tạo PVC, **không** tạo Service, **không** cài metrics-server
hay VPA để "chạy thử cho đủ". StatefulSet của lab này khai báo `serviceName` trỏ tới một Service
**chưa tồn tại** — B1.4 chứng minh Pod vẫn có định danh ổn định đầy đủ, chỉ phần tên DNS là chưa có.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 4b | Kiểm chứng ở |
| --- | --- |
| [65 — StatefulSets](../65-statefulset-vi.md) | B1 (định danh cố định, `OrderedReady`), B2 (thứ tự kết thúc ngược), B3 (xóa và `--cascade=orphan`). Hai mục *Template cho volume claim* và *Định danh mạng ổn định* là nợ #2 và #3 |
| [347 — Scale một StatefulSet](../347-scale-stateful-set-vi.md) | B2 — `kubectl scale statefulsets`, và điều kiện "chỉ scale khi mọi Pod đang khỏe" |
| [341 — Xóa cưỡng bức Pod của StatefulSet](../341-force-delete-stateful-set-pod-vi.md) | B3.2 — đo và so sánh xóa êm thấm với `--grace-period=0 --force` |
| [340 — Xóa một StatefulSet](../340-delete-stateful-set-vi.md) | B3.3 — xóa theo tầng so với `--cascade=orphan`, rồi dọn Pod bằng label |
| [66 — DaemonSet](../66-daemonset-vi.md) | B4 — đếm Pod theo số node, taint/toleration, ai đặt `.spec.nodeName`, toleration controller tự thêm |
| [38 — Khả năng tự phục hồi](../38-self-healing-vi.md) | B5 — kubelet restart container tại chỗ so với controller tạo Pod mới **đúng node cũ** |
| [67 — Jobs](../67-job-vi.md) | B6 (ba loại tác vụ, `completions`/`parallelism`, `Indexed`), B7 (`backoffLimit`, `restartPolicy`, thất bại vĩnh viễn) |
| [349 — Chạy Job](../349-job-tasks-vi.md) | B6 — trang mục lục của mục *Run Jobs*; lab chạy ba mẫu mà nó liệt kê: `Indexed` ở B6.3, CronJob ở B9, khai triển template ở B10 |
| [353 — Indexed Job](../353-indexed-parallel-processing-vi.md) | B6.3 — `completionMode: Indexed`, biến `JOB_COMPLETION_INDEX`, `.status.completedIndexes` |
| [68 — Tự động dọn dẹp Job đã hoàn thành](../68-ttlafterfinished-vi.md) | B8 — `ttlSecondsAfterFinished`, mốc bắt đầu đếm giờ, xóa theo tầng |
| [69 — CronJob](../69-cron-jobs-vi.md) | B9 — `schedule`, `concurrencyPolicy: Forbid`, `startingDeadlineSeconds`, `suspend` |
| [350 — Chạy tác vụ tự động với CronJob](../350-automated-tasks-cron-jobs-vi.md) | B9.1 và B9.5 — tạo CronJob, đọc Job/Pod nó sinh ra, xóa CronJob và quan sát Job biến mất theo |
| [355 — Xử lý song song bằng khai triển template](../355-parallel-processing-expansion-vi.md) | B10 — khai triển `$ITEM` bằng `sed`, thao tác cả nhóm Job bằng label `jobgroup` |
| [71 — Tự động co giãn Workload](../71-autoscaling-vi.md) | B2 — co giãn ngang **thủ công**, đúng trục mà bài đặt tên. Phần tự động là nợ #1 |

Bốn bài dưới đây **không kiểm chứng được trên cluster baseline**, đọc để biết:

| Bài | Vì sao không thực hành ở đây |
| --- | --- |
| [72 — Tự động co giãn Pod theo chiều ngang](../72-horizontal-pod-autoscale-vi.md) | HPA controller cần API `metrics.k8s.io` do metrics-server cung cấp; metrics-server thuộc giai đoạn 11. **Nợ #1**, trả ở Lab 11b |
| [73 — Tự động co giãn Pod theo chiều dọc](../73-vertical-pod-autoscale-vi.md) | VPA không đi kèm Kubernetes — phải cài add-on riêng, và add-on đó lại cần Metrics Server. **Nợ #1**, trả ở Lab 11b |
| [351 — Hàng đợi công việc thô](../351-coarse-parallel-work-queue-vi.md) | Bài dựng một Service RabbitMQ và yêu cầu tự build một image worker rồi đẩy lên registry. Service thuộc giai đoạn 5; lab không build/push image. B6.4 chỉ thực hành **hình dạng** Job hàng đợi công việc |
| [352 — Hàng đợi công việc mịn](../352-fine-parallel-work-queue-vi.md) | Như trên, với một Service Redis và một image worker Python tự build |

### 1.2. Thời lượng

3–4 giờ, tính từ lúc gate `01-cluster-ready` đã PASS. B2, B3 và B9 cố ý chờ — StatefulSet có
`preStop` kéo dài cửa sổ kết thúc để quan sát được thứ tự, còn CronJob chạy theo lịch phút.

---

## 2. Quy ước và an toàn

- Mọi lệnh chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, trừ khi ghi rõ node khác.
- **Chạy toàn bộ phần B trong cùng một phiên shell.** Nhiều gate so sánh biến đặt ở bước trước
  (`T_GRACE`, `UID_BEFORE`, `NODE_SCHEDULABLE`…); mở shell mới giữa chừng là mất biến.
- Lab chỉ tạo Namespace `lab-4b` và các object bên trong nó: StatefulSet, DaemonSet, Job, CronJob,
  Pod. **Không** tạo Service, PVC, PV, StorageClass, HorizontalPodAutoscaler, NetworkPolicy hay
  ResourceQuota. Không cài metrics-server, VPA hay bất kỳ add-on nào.
- **Fault injection chỉ trên `lab-k8s-worker2`** — B5 kill tiến trình và xóa Pod trên đúng node này.
- B3.2 dùng `--grace-period=0 --force`. Thao tác này chỉ được chạy trên StatefulSet `web` của lab,
  vốn không có dữ liệu và không có Service. Đây **không phải** thao tác vận hành bình thường; bài
  [341](../341-force-delete-stateful-set-pod-vi.md) nói rõ trong hoạt động bình thường **không bao
  giờ** có nhu cầu xóa cưỡng bức Pod của StatefulSet.
- Không sửa manifest control plane, không sửa cấu hình kubelet/containerd, không đụng vào
  `kube-system` và `kube-flannel`. B4.4 chỉ **đọc** Pod của Flannel để đối chiếu.
- Lab cần kéo được image `busybox` từ internet. Nếu môi trường cô lập, xem mục 4.
- Manifest tạm ghi vào `~/lab-work/4b/`; bằng chứng ghi vào `~/lab-evidence/4b/`. Không lưu token,
  key hay certificate.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail.

---

# Phần B — Thực hành kiến thức 4b

## B0. Chuẩn bị workspace và namespace

```bash
mkdir -p ~/lab-work/4b ~/lab-evidence/4b
kubectl config current-context
kubectl get nodes -o wide
kubectl create namespace lab-4b
kubectl get namespace lab-4b -o jsonpath='{.status.phase}'; echo
kubectl get svc,pvc,job,cronjob -n lab-4b
```

**Ý nghĩa:** `lab-work` chứa manifest tạm, `lab-evidence` chứa output. Namespace `lab-4b` cô lập
mọi object lab tạo ra. Lệnh cuối là ảnh chụp "trước": namespace phải trống hoàn toàn, đặc biệt là
không có Service và không có PVC — hai thứ mà nợ #2 và #3 cấm dùng trong lab này.

**PASS:** context trỏ đúng cluster lab; ba node `Ready`; namespace `lab-4b` ở phase `Active`;
lệnh cuối trả `No resources found in lab-4b namespace.`

Ghi lại số node và số node có thể lập lịch — B4 dùng lại hai con số này:

```bash
NODE_TOTAL=0
NODE_TAINTED=0
for n in $(kubectl get nodes -o name); do
  NODE_TOTAL=$((NODE_TOTAL + 1))
  T="$(kubectl get "$n" -o jsonpath='{.spec.taints[*].effect}')"
  case "$T" in *NoSchedule*) NODE_TAINTED=$((NODE_TAINTED + 1)) ;; esac
done
NODE_SCHEDULABLE=$((NODE_TOTAL - NODE_TAINTED))

echo "node tổng cộng      : $NODE_TOTAL"
echo "node mang NoSchedule: $NODE_TAINTED"
echo "node lập lịch được  : $NODE_SCHEDULABLE"
{
  echo "NODE_TOTAL=$NODE_TOTAL"
  echo "NODE_TAINTED=$NODE_TAINTED"
  echo "NODE_SCHEDULABLE=$NODE_SCHEDULABLE"
} | tee ~/lab-evidence/4b/b0-node-count.txt

test "$NODE_TOTAL" -eq 3 \
  && test "$NODE_TAINTED" -eq 1 \
  && test "$NODE_SCHEDULABLE" -eq 2 \
  && echo 'PASS: 3 node, 1 control plane mang taint NoSchedule, 2 worker lập lịch được'
```

**PASS:** dòng `PASS: 3 node, ...` xuất hiện. Nếu con số khác, cluster không còn đúng
`01-cluster-ready` — dừng và đối chiếu [tầng 2 của gate A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a543-tầng-2--node-condition-taint-và-podcidr).

## B1. StatefulSet — định danh cố định và thứ tự khởi tạo

### B1.1. Manifest cố ý thiếu hai thứ

```bash
cat > ~/lab-work/4b/statefulset.yaml <<'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: lab-4b
spec:
  serviceName: web-headless
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 15; touch /tmp/ready; sleep 3600"]
        readinessProbe:
          exec:
            command: ["sh", "-c", "test -f /tmp/ready"]
          initialDelaySeconds: 5
          periodSeconds: 3
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 15"]
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/4b/statefulset.yaml
```

**Ý nghĩa của từng lựa chọn:**

- **Không có `volumeClaimTemplates`** — đó là nợ #2. Bài 65 nói lưu trữ cho Pod phải do một
  PersistentVolume provisioner cấp phát theo StorageClass; cluster baseline chưa có thứ đó.
- **`serviceName: web-headless` trỏ tới một Service không tồn tại** — đó là nợ #3. Bài 65 xếp
  "StatefulSet yêu cầu một Headless Service do bạn tự tạo" vào mục *Hạn chế*, và Service đó chỉ
  quyết định **miền DNS** của Pod. B1.4 chứng minh phần định danh còn lại vẫn hoạt động đầy đủ.
- **`readinessProbe` + `sleep 15`** làm cửa sổ "Running nhưng chưa Ready" đủ rộng để quan sát bảo
  đảm thứ tự. Không có probe thì Pod Ready ngay khi container chạy và bạn không nhìn thấy gì.
- **`preStop` + `terminationGracePeriodSeconds`** làm cửa sổ kết thúc đủ rộng cho B2.2 và B3.

**PASS:** `--dry-run=server` in `statefulset.apps/web created (server dry run)`, không có lỗi
validation.

### B1.2. Bảo đảm thứ tự: Pod sau chờ Pod trước Ready

Chạy hai khối dưới đây **liền nhau**, không nghỉ giữa chừng.

```bash
kubectl apply -f ~/lab-work/4b/statefulset.yaml

for i in $(seq 1 120); do
  kubectl get pod web-0 -n lab-4b >/dev/null 2>&1 && break
  sleep 1
done
```

```bash
READY0="$(kubectl get pod web-0 -n lab-4b \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
EXISTS1="$(kubectl get pod web-1 -n lab-4b --ignore-not-found -o name | wc -l)"
EXISTS2="$(kubectl get pod web-2 -n lab-4b --ignore-not-found -o name | wc -l)"

echo "web-0 Ready = ${READY0:-<chưa có condition>}"
echo "web-1 tồn tại = $EXISTS1"
echo "web-2 tồn tại = $EXISTS2"

test "$READY0" != 'True' \
  && test "$EXISTS1" -eq 0 \
  && test "$EXISTS2" -eq 0 \
  && echo 'PASS: web-0 chưa Ready thì web-1 và web-2 chưa được tạo'
```

**Ý nghĩa:** đây là bảo đảm `OrderedReady` của bài 65 ở dạng quan sát được — "trước khi một thao
tác mở rộng được áp dụng cho một Pod, tất cả các Pod đứng trước nó phải ở trạng thái Running và
Ready". Deployment ở 4a **không** có bảo đảm này: ReplicaSet tạo cả ba Pod cùng lúc.

**PASS:** dòng `PASS: web-0 chưa Ready thì web-1 và web-2 chưa được tạo` xuất hiện.

> Nếu gate fail vì `web-0` đã `Ready` khi bạn kịp đo, xóa StatefulSet
> (`kubectl delete -f ~/lab-work/4b/statefulset.yaml --wait=true`) rồi chạy lại hai khối liền nhau.
> Đây là lỗi thao tác, không phải sai lệch của cluster.

### B1.3. Thứ tự tạo được ghi lại ở timestamp

```bash
kubectl rollout status statefulset/web -n lab-4b --timeout=300s

kubectl get pods -n lab-4b -l app=web -o wide \
  | tee ~/lab-evidence/4b/b1-pods.txt

ORDER="$(kubectl get pods -n lab-4b -l app=web \
  --sort-by=.metadata.creationTimestamp -o name | sed 's|pod/||' | paste -sd' ' -)"
echo "thứ tự theo creationTimestamp: $ORDER"

test "$ORDER" = 'web-0 web-1 web-2' \
  && echo 'PASS: Pod được tạo tuần tự 0 -> N-1'
```

**Ý nghĩa:** sắp theo `creationTimestamp` cho đúng dãy `web-0 web-1 web-2` — số thứ tự chạy từ 0
đến N-1 và thứ tự tạo trùng với thứ tự ordinal.

**PASS:** dòng `PASS: Pod được tạo tuần tự 0 -> N-1` xuất hiện.

### B1.4. Định danh gồm những gì, và phần nào còn thiếu

```bash
for p in web-0 web-1 web-2; do
  IDX="$(kubectl get pod "$p" -n lab-4b \
    -o jsonpath='{.metadata.labels.apps\.kubernetes\.io/pod-index}')"
  PNAME="$(kubectl get pod "$p" -n lab-4b \
    -o jsonpath='{.metadata.labels.statefulset\.kubernetes\.io/pod-name}')"
  NODE="$(kubectl get pod "$p" -n lab-4b -o jsonpath='{.spec.nodeName}')"
  HN="$(kubectl exec "$p" -n lab-4b -- hostname)"
  echo "$p | pod-index=$IDX | pod-name=$PNAME | hostname=$HN | node=$NODE"
done | tee ~/lab-evidence/4b/b1-identity.txt
```

```bash
OK=1
for i in 0 1 2; do
  p="web-$i"
  test "$(kubectl get pod "$p" -n lab-4b \
    -o jsonpath='{.metadata.labels.apps\.kubernetes\.io/pod-index}')" = "$i" || OK=0
  test "$(kubectl get pod "$p" -n lab-4b \
    -o jsonpath='{.metadata.labels.statefulset\.kubernetes\.io/pod-name}')" = "$p" || OK=0
  test "$(kubectl exec "$p" -n lab-4b -- hostname)" = "$p" || OK=0
done
test "$OK" -eq 1 \
  && echo 'PASS: ordinal, label pod-index, label pod-name và hostname khớp nhau'

SVCNAME="$(kubectl get statefulset web -n lab-4b -o jsonpath='{.spec.serviceName}')"
SVCCOUNT="$(kubectl get svc -n lab-4b -o name | wc -l)"
echo "spec.serviceName = $SVCNAME"
echo "số Service trong lab-4b = $SVCCOUNT"

test "$SVCNAME" = 'web-headless' \
  && test "$SVCCOUNT" -eq 0 \
  && echo 'PASS: serviceName đã khai nhưng Service quản trị chưa tồn tại (nợ #3)'
```

**Ý nghĩa:** hostname của Pod lấy từ `$(tên StatefulSet)-$(ordinal)`, và controller gắn thêm hai
label — `statefulset.kubernetes.io/pod-name` và `apps.kubernetes.io/pod-index`. Cả bốn thứ này có
mặt **dù không có Service nào**. Phần duy nhất còn thiếu là miền DNS
`$(podname).$(governing service domain)` mà bài 65 mô tả: tên
`web-0.web-headless.lab-4b.svc.cluster.local` chưa phân giải được vì `web-headless` chưa tồn tại.
Đó là **nợ #3**, trả ở Lab 5a — đừng tạo Service ở đây để "cho đủ".

**PASS:** hai dòng `PASS:` xuất hiện.

## B2. Scale StatefulSet — hai chiều, hai thứ tự

### B2.1. Mở rộng: vẫn tuần tự tăng dần

```bash
kubectl scale statefulsets web -n lab-4b --replicas=5

for i in $(seq 1 120); do
  kubectl get pod web-3 -n lab-4b >/dev/null 2>&1 && break
  sleep 1
done
```

```bash
READY3="$(kubectl get pod web-3 -n lab-4b \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
EXISTS4="$(kubectl get pod web-4 -n lab-4b --ignore-not-found -o name | wc -l)"
echo "web-3 Ready = ${READY3:-<chưa có condition>}"
echo "web-4 tồn tại = $EXISTS4"

test "$READY3" != 'True' && test "$EXISTS4" -eq 0 \
  && echo 'PASS: mở rộng vẫn tuần tự — web-4 chờ web-3 Ready'

kubectl rollout status statefulset/web -n lab-4b --timeout=300s
kubectl get pods -n lab-4b -l app=web -o wide | tee ~/lab-evidence/4b/b2-scale-up.txt
```

**Ý nghĩa:** `kubectl scale statefulsets` là cách bài 347 nêu đầu tiên. Đây cũng chính là **co giãn
ngang thủ công** mà bài [71](../71-autoscaling-vi.md) đặt tên: thay đổi **số lượng** replica. Bản
tự động của trục này là HPA — nợ #1, trả ở Lab 11b.

**PASS:** dòng `PASS: mở rộng vẫn tuần tự` xuất hiện và `rollout status` báo 5 replica ready.

### B2.2. Thu nhỏ: thứ tự ngược, từ ordinal lớn nhất

```bash
kubectl scale statefulsets web -n lab-4b --replicas=1

DEL4=''
for i in $(seq 1 120); do
  DEL4="$(kubectl get pod web-4 -n lab-4b \
    -o jsonpath='{.metadata.deletionTimestamp}' 2>/dev/null || true)"
  test -n "$DEL4" && break
  sleep 1
done
DEL3="$(kubectl get pod web-3 -n lab-4b \
  -o jsonpath='{.metadata.deletionTimestamp}' 2>/dev/null || true)"
EXISTS3="$(kubectl get pod web-3 -n lab-4b --ignore-not-found -o name | wc -l)"

echo "web-4 deletionTimestamp = ${DEL4:-<trống>}"
echo "web-3 deletionTimestamp = ${DEL3:-<trống>}"
echo "web-3 còn tồn tại = $EXISTS3"

test -n "$DEL4" && test -z "$DEL3" && test "$EXISTS3" -eq 1 \
  && echo 'PASS: web-4 đang kết thúc trong khi web-3 chưa bị đụng tới'
```

**Ý nghĩa:** bài 65 viết "khi các Pod đang bị xóa, chúng bị kết thúc theo thứ tự ngược lại, từ
{N-1..0}" và "trước khi một Pod bị kết thúc, tất cả các Pod đứng sau nó phải được tắt hoàn toàn".
`preStop` với `sleep 15` giữ `web-4` ở trạng thái đang kết thúc đủ lâu để bạn đo được điều đó bằng
`deletionTimestamp` thay vì "quan sát thấy".

**PASS:** dòng `PASS: web-4 đang kết thúc trong khi web-3 chưa bị đụng tới` xuất hiện.

```bash
for i in $(seq 1 180); do
  READY="$(kubectl get statefulset web -n lab-4b -o jsonpath='{.status.readyReplicas}')"
  CUR="$(kubectl get pods -n lab-4b -l app=web -o name | wc -l)"
  test "${READY:-0}" -eq 1 && test "$CUR" -eq 1 && break
  sleep 2
done
kubectl get pods -n lab-4b -l app=web | tee ~/lab-evidence/4b/b2-scale-down.txt

REMAIN="$(kubectl get pods -n lab-4b -l app=web -o name | sed 's|pod/||' | paste -sd' ' -)"
echo "còn lại: $REMAIN"
test "$REMAIN" = 'web-0' \
  && echo 'PASS: thu nhỏ về 1 giữ lại đúng ordinal nhỏ nhất'
```

**Ý nghĩa:** Pod sống sót là `web-0`, không phải một Pod bất kỳ. Đây là điểm khác Deployment rõ
nhất: ReplicaSet chọn Pod để xóa theo một bộ tiêu chí, còn StatefulSet luôn cắt từ đuôi.

**PASS:** dòng `PASS: thu nhỏ về 1 giữ lại đúng ordinal nhỏ nhất` xuất hiện.

> Bài 347 có một cảnh báo không kiểm chứng được ở đây nhưng phải nhớ: **không thu nhỏ khi còn Pod
> stateful không khỏe mạnh**. Kubernetes không phân biệt được lỗi vĩnh viễn với lỗi tạm thời, nên
> scale trong lúc đó có thể kéo số thành viên xuống dưới ngưỡng tối thiểu của ứng dụng.

## B3. Xóa Pod và xóa StatefulSet

### B3.1. Xóa êm thấm: tên được giữ, danh tính không ghim vào node

```bash
kubectl scale statefulsets web -n lab-4b --replicas=2
kubectl rollout status statefulset/web -n lab-4b --timeout=300s

UID_BEFORE="$(kubectl get pod web-1 -n lab-4b -o jsonpath='{.metadata.uid}')"
NODE_BEFORE="$(kubectl get pod web-1 -n lab-4b -o jsonpath='{.spec.nodeName}')"
echo "web-1 trước khi xóa: uid=$UID_BEFORE node=$NODE_BEFORE"

T0=$(date +%s)
kubectl delete pod web-1 -n lab-4b --wait=true
for i in $(seq 1 180); do
  UID_NOW="$(kubectl get pod web-1 -n lab-4b -o jsonpath='{.metadata.uid}' 2>/dev/null || true)"
  test -n "$UID_NOW" && test "$UID_NOW" != "$UID_BEFORE" && break
  sleep 1
done
T_GRACE=$(( $(date +%s) - T0 ))

UID_AFTER="$(kubectl get pod web-1 -n lab-4b -o jsonpath='{.metadata.uid}')"
NODE_AFTER="$(kubectl get pod web-1 -n lab-4b -o jsonpath='{.spec.nodeName}')"
echo "web-1 sau khi xóa  : uid=$UID_AFTER node=$NODE_AFTER"
echo "T_GRACE = ${T_GRACE}s"

test "$UID_AFTER" != "$UID_BEFORE" \
  && test "$(kubectl get pod web-1 -n lab-4b -o jsonpath='{.metadata.name}')" = 'web-1' \
  && echo 'PASS: Pod thay thế mang đúng tên web-1 nhưng là object mới (uid khác)'
```

**Ý nghĩa:** đây là *thay thế replica* của bài [38](../38-self-healing-vi.md) áp dụng cho
StatefulSet: controller không sửa Pod cũ mà tạo Pod **mới**. Cái được giữ là **định danh**, không
phải object. Nếu `node` trước và sau khác nhau, đó là hành vi đúng — bài 65 nói định danh "gắn chặt
với Pod, bất kể Pod được lập lịch (lại) lên node nào". Nếu chúng giống nhau thì cũng không sai:
scheduler chỉ đơn giản chọn lại cùng node.

**PASS:** dòng `PASS: Pod thay thế mang đúng tên web-1 nhưng là object mới (uid khác)` xuất hiện, và
biến `T_GRACE` có giá trị dương.

### B3.2. Xóa cưỡng bức: tên được giải phóng ngay lập tức

> **Thận trọng.** Bài [341](../341-force-delete-stateful-set-pod-vi.md) nói trong hoạt động bình
> thường **không bao giờ** cần xóa cưỡng bức Pod của StatefulSet. Bước này chạy được ở đây **chỉ
> vì** `web` không có dữ liệu, không có Service, và không có Pod nào đang liên lạc với nhau. Đừng
> mang lệnh này sang cluster thật.

```bash
UID_BEFORE="$(kubectl get pod web-1 -n lab-4b -o jsonpath='{.metadata.uid}')"
echo "web-1 trước khi force delete: uid=$UID_BEFORE"

T0=$(date +%s)
kubectl delete pod web-1 -n lab-4b --grace-period=0 --force
for i in $(seq 1 180); do
  UID_NOW="$(kubectl get pod web-1 -n lab-4b -o jsonpath='{.metadata.uid}' 2>/dev/null || true)"
  test -n "$UID_NOW" && test "$UID_NOW" != "$UID_BEFORE" && break
  sleep 1
done
T_FORCE=$(( $(date +%s) - T0 ))

echo "T_GRACE = ${T_GRACE}s   (B3.1, xóa êm thấm)"
echo "T_FORCE = ${T_FORCE}s   (B3.2, xóa cưỡng bức)"
{
  echo "T_GRACE=$T_GRACE"
  echo "T_FORCE=$T_FORCE"
} | tee ~/lab-evidence/4b/b3-delete-timing.txt

test "$T_FORCE" -lt "$T_GRACE" \
  && echo 'PASS: force delete giải phóng tên web-1 nhanh hơn hẳn xóa êm thấm'

kubectl rollout status statefulset/web -n lab-4b --timeout=300s
```

**Ý nghĩa:** hai con số này đo đúng thứ bài 341 mô tả. Xóa êm thấm chờ kubelet xác nhận Pod đã
kết thúc — ở đây là hết `preStop` rồi mới tới `terminationGracePeriodSeconds`. Xóa cưỡng bức
**không chờ xác nhận nào**: nó giải phóng tên khỏi apiserver ngay, nên StatefulSet controller tạo
được ngay một Pod mang **cùng định danh** trong khi container cũ có thể vẫn đang chạy. Đó chính là
cách ngữ nghĩa "nhiều nhất một" bị phá vỡ, và với một hệ dựa trên quorum thì kết quả là chia não.

Đừng đọc hai con số như một cam kết về thời gian — chúng phụ thuộc `preStop`,
`terminationGracePeriodSeconds` và tải của node. Thứ có nghĩa là **quan hệ** giữa chúng.

**PASS:** dòng `PASS: force delete giải phóng tên web-1 nhanh hơn hẳn xóa êm thấm` xuất hiện.

### B3.3. Xóa StatefulSet: theo tầng so với `--cascade=orphan`

```bash
kubectl delete statefulset web -n lab-4b --cascade=orphan

STS_LEFT="$(kubectl get statefulset web -n lab-4b --ignore-not-found -o name | wc -l)"
POD_LEFT="$(kubectl get pods -n lab-4b -l app=web -o name | wc -l)"
OWNER="$(kubectl get pod web-0 -n lab-4b -o jsonpath='{.metadata.ownerReferences[*].kind}')"

echo "StatefulSet còn lại = $STS_LEFT"
echo "Pod còn lại         = $POD_LEFT"
echo "ownerReferences của web-0 = ${OWNER:-<trống>}"

kubectl get pods -n lab-4b -l app=web -o wide | tee ~/lab-evidence/4b/b3-orphan.txt

test "$STS_LEFT" -eq 0 && test "$POD_LEFT" -eq 2 && test -z "$OWNER" \
  && echo 'PASS: StatefulSet biến mất, hai Pod ở lại và không còn owner'
```

**Ý nghĩa:** bài [340](../340-delete-stateful-set-vi.md) nói khi xóa StatefulSet bằng `kubectl`
theo mặc định, nó được scale xuống 0 và mọi Pod bị xóa theo; `--cascade=orphan` giữ Pod lại. Cơ
chế bên dưới là garbage collection theo owner reference mà bạn đã dựng tay ở
[Lab 1c B2](LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md) — ở đây bạn thấy nó do một workload
controller thật tạo ra.

Dọn Pod mồ côi bằng label, đúng cách bài 340 hướng dẫn:

```bash
kubectl delete pods -l app=web -n lab-4b --wait=true

POD_LEFT="$(kubectl get pods -n lab-4b -l app=web -o name | wc -l)"
test "$POD_LEFT" -eq 0 && echo 'PASS: Pod mồ côi đã được dọn bằng label selector'
```

> Phần *Persistent Volume* của bài 340 — xóa Pod không xóa volume, xóa PVC có thể kích hoạt xóa PV
> tùy reclaim policy — **không kiểm chứng được ở đây** vì lab không có PVC. Đó là **nợ #2**, trả ở
> Lab 6a. Khi làm Lab 6a, đọc lại bài [65](../65-statefulset-vi.md) và
> [340](../340-delete-stateful-set-vi.md) trước.

**PASS:** hai dòng `PASS:` của B3.3 xuất hiện.

## B4. DaemonSet — một Pod trên mỗi node đủ điều kiện

### B4.1. Không toleration: chỉ worker có Pod

```bash
cat > ~/lab-work/4b/daemonset.yaml <<'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nodeprobe
  namespace: lab-4b
spec:
  selector:
    matchLabels:
      app: nodeprobe
  template:
    metadata:
      labels:
        app: nodeprobe
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/4b/daemonset.yaml
kubectl rollout status daemonset/nodeprobe -n lab-4b --timeout=300s
kubectl get pods -n lab-4b -l app=nodeprobe -o wide | tee ~/lab-evidence/4b/b4-ds-workers.txt
```

```bash
DESIRED="$(kubectl get daemonset nodeprobe -n lab-4b \
  -o jsonpath='{.status.desiredNumberScheduled}')"
READY="$(kubectl get daemonset nodeprobe -n lab-4b -o jsonpath='{.status.numberReady}')"
PODS="$(kubectl get pods -n lab-4b -l app=nodeprobe -o name | wc -l)"
NODES_WITH_POD="$(kubectl get pods -n lab-4b -l app=nodeprobe \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | wc -l)"

echo "desiredNumberScheduled = $DESIRED"
echo "numberReady            = $READY"
echo "số Pod                 = $PODS"
echo "số node có Pod         = $NODES_WITH_POD"
echo "node lập lịch được     = $NODE_SCHEDULABLE (đo ở B0)"

test "$DESIRED" -eq "$NODE_SCHEDULABLE" \
  && test "$READY" -eq "$NODE_SCHEDULABLE" \
  && test "$PODS" -eq "$NODES_WITH_POD" \
  && echo 'PASS: đúng một Pod trên mỗi node đủ điều kiện, và không node nào có hai Pod'
```

**Ý nghĩa:** manifest không có `nodeSelector` và không có `affinity`, nên theo bài 66 controller
tạo Pod trên **tất cả** các node. Con số dừng ở 2 chứ không phải 3 vì `lab-k8s-master` mang taint
`node-role.kubernetes.io/control-plane:NoSchedule` do kubeadm đặt, và Pod template chưa tolerate
taint đó. Số Pod bằng số node có Pod là bằng chứng cho "một bản sao trên mỗi node", không phải "hai
Pod tình cờ trên hai node".

**PASS:** dòng `PASS: đúng một Pod trên mỗi node đủ điều kiện...` xuất hiện.

### B4.2. Thêm toleration: control plane node cũng có Pod

```bash
kubectl patch daemonset nodeprobe -n lab-4b --type=merge -p '{
  "spec": {"template": {"spec": {"tolerations": [
    {"key": "node-role.kubernetes.io/control-plane", "operator": "Exists", "effect": "NoSchedule"}
  ]}}}
}'

for i in $(seq 1 90); do
  D="$(kubectl get daemonset nodeprobe -n lab-4b -o jsonpath='{.status.desiredNumberScheduled}')"
  R="$(kubectl get daemonset nodeprobe -n lab-4b -o jsonpath='{.status.numberReady}')"
  test "${D:-0}" -eq "$NODE_TOTAL" && test "${R:-0}" -eq "$NODE_TOTAL" && break
  sleep 2
done
kubectl get pods -n lab-4b -l app=nodeprobe -o wide | tee ~/lab-evidence/4b/b4-ds-all-nodes.txt
```

```bash
DESIRED="$(kubectl get daemonset nodeprobe -n lab-4b \
  -o jsonpath='{.status.desiredNumberScheduled}')"
NODES_WITH_POD="$(kubectl get pods -n lab-4b -l app=nodeprobe \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | wc -l)"
ON_MASTER="$(kubectl get pods -n lab-4b -l app=nodeprobe \
  --field-selector spec.nodeName=lab-k8s-master -o name | wc -l)"

echo "desiredNumberScheduled = $DESIRED (NODE_TOTAL = $NODE_TOTAL)"
echo "số node có Pod         = $NODES_WITH_POD"
echo "Pod trên control plane = $ON_MASTER"

test "$DESIRED" -eq "$NODE_TOTAL" \
  && test "$NODES_WITH_POD" -eq "$NODE_TOTAL" \
  && test "$ON_MASTER" -eq 1 \
  && echo 'PASS: một toleration làm tập node đủ điều kiện tăng từ 2 lên 3'
```

**Ý nghĩa:** đây đúng là hai toleration được viết sẵn trong ví dụ `fluentd` của bài 66, kèm comment
"hãy xóa chúng nếu control plane node của bạn không nên chạy pod". Tập node của DaemonSet không do
bạn liệt kê mà do **điều kiện** quyết định: selector/affinity chọn node, toleration mở khóa taint.

**PASS:** dòng `PASS: một toleration làm tập node đủ điều kiện tăng từ 2 lên 3` xuất hiện.

### B4.3. Ai bind Pod vào node

```bash
DSPOD="$(kubectl get pods -n lab-4b -l app=nodeprobe \
  --field-selector spec.nodeName=lab-k8s-worker2 -o jsonpath='{.items[0].metadata.name}')"
echo "Pod dùng để soi: $DSPOD"

kubectl get pod "$DSPOD" -n lab-4b -o yaml | tee ~/lab-evidence/4b/b4-ds-pod.yaml >/dev/null

NA_KEY="$(kubectl get pod "$DSPOD" -n lab-4b -o jsonpath='{.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchFields[0].key}')"
NA_VAL="$(kubectl get pod "$DSPOD" -n lab-4b -o jsonpath='{.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchFields[0].values[0]}')"
NODENAME="$(kubectl get pod "$DSPOD" -n lab-4b -o jsonpath='{.spec.nodeName}')"
SCHED="$(kubectl get pod "$DSPOD" -n lab-4b -o jsonpath='{.spec.schedulerName}')"

echo "matchFields key   = $NA_KEY"
echo "matchFields value = $NA_VAL"
echo "spec.nodeName     = $NODENAME"
echo "schedulerName     = $SCHED"

test "$NA_KEY" = 'metadata.name' \
  && test "$NA_VAL" = "$NODENAME" \
  && test "$SCHED" = 'default-scheduler' \
  && echo 'PASS: controller ghi nodeAffinity khớp host, scheduler mặc định mới đặt nodeName'
```

**Ý nghĩa:** đây là câu bẫy của bài 66. DaemonSet controller **không** ghim Pod vào node bằng
`.spec.nodeName`; nó tạo Pod kèm `spec.affinity.nodeAffinity` với `matchFields` trỏ đúng
`metadata.name` của host mục tiêu, rồi **scheduler mặc định** mới bind. Hệ quả rất thật: nếu Pod
không vừa trên node, scheduler có thể phải chiếm chỗ Pod khác dựa trên độ ưu tiên — và bạn cũng có
thể thay scheduler bằng `.spec.template.spec.schedulerName`.

**PASS:** dòng `PASS: controller ghi nodeAffinity khớp host...` xuất hiện.

### B4.4. Bộ toleration mà bạn không viết

```bash
kubectl get pod "$DSPOD" -n lab-4b \
  -o jsonpath='{range .spec.tolerations[*]}{.key}{"\t"}{.effect}{"\n"}{end}' \
  | tee ~/lab-evidence/4b/b4-ds-tolerations.txt
```

```bash
KEYS="$(kubectl get pod "$DSPOD" -n lab-4b \
  -o jsonpath='{range .spec.tolerations[*]}{.key}{"\n"}{end}')"
OK=1
for k in node.kubernetes.io/not-ready node.kubernetes.io/unreachable \
         node.kubernetes.io/disk-pressure node.kubernetes.io/memory-pressure \
         node.kubernetes.io/pid-pressure node.kubernetes.io/unschedulable; do
  echo "$KEYS" | grep -qx "$k" || { echo "FAIL: thiếu toleration $k"; OK=0; }
done
test "$OK" -eq 1 && echo 'PASS: controller tự thêm đủ sáu toleration của bảng trong bài 66'

if echo "$KEYS" | grep -qx 'node.kubernetes.io/network-unavailable'; then
  echo 'FAIL: Pod không dùng hostNetwork mà lại có toleration network-unavailable'
else
  echo 'PASS: không có network-unavailable vì Pod này không đặt hostNetwork: true'
fi
```

Đối chiếu với DaemonSet mạng có thật trên cluster — chỉ **đọc**, không sửa:

```bash
FLPOD="$(kubectl -n kube-flannel get pods -l app=flannel \
  -o jsonpath='{.items[0].metadata.name}')"
HOSTNET="$(kubectl -n kube-flannel get pod "$FLPOD" -o jsonpath='{.spec.hostNetwork}')"
FLKEYS="$(kubectl -n kube-flannel get pod "$FLPOD" \
  -o jsonpath='{range .spec.tolerations[*]}{.key}{"\n"}{end}')"

echo "Pod Flannel  = $FLPOD"
echo "hostNetwork  = $HOSTNET"
echo "$FLKEYS" | tee ~/lab-evidence/4b/b4-flannel-tolerations.txt

test "$HOSTNET" = 'true' \
  && echo "$FLKEYS" | grep -qx 'node.kubernetes.io/network-unavailable' \
  && echo 'PASS: Pod DaemonSet dùng hostNetwork mới được thêm network-unavailable'
```

**Ý nghĩa:** bảng toleration của bài 66 không phải lý thuyết — nó nằm nguyên trên Pod của bạn.
Riêng `node.kubernetes.io/network-unavailable` **chỉ** được thêm cho Pod có `spec.hostNetwork: true`,
và Flannel là ví dụ sống: nhờ nó cộng với `not-ready:NoExecute`, Pod mạng lên được node **trước khi**
node Ready. Không có cơ chế đó thì bế tắc — node không Ready vì chưa có network plugin, còn network
plugin không chạy vì node chưa Ready. Chính cluster mà bạn đang gõ lệnh đã boot qua vòng này.

**PASS:** ba dòng `PASS:` của B4.4 xuất hiện, không có dòng `FAIL:`.

## B5. Tự phục hồi — kubelet sửa container, controller thay Pod

Toàn bộ mục này là fault injection và chỉ chạy trên `lab-k8s-worker2`, đúng Pod `$DSPOD` đã chọn ở
B4.3.

### B5.1. Tầng kubelet: container chết, Pod ở nguyên

```bash
RC_BEFORE="$(kubectl get pod "$DSPOD" -n lab-4b \
  -o jsonpath='{.status.containerStatuses[0].restartCount}')"
UID_BEFORE="$(kubectl get pod "$DSPOD" -n lab-4b -o jsonpath='{.metadata.uid}')"
RESTART_POLICY="$(kubectl get pod "$DSPOD" -n lab-4b -o jsonpath='{.spec.restartPolicy}')"
echo "restartCount trước = $RC_BEFORE | restartPolicy = $RESTART_POLICY"

kubectl exec -n lab-4b "$DSPOD" -- sh -c 'kill 1' || true

for i in $(seq 1 90); do
  RC_NOW="$(kubectl get pod "$DSPOD" -n lab-4b \
    -o jsonpath='{.status.containerStatuses[0].restartCount}' 2>/dev/null || true)"
  test -n "$RC_NOW" && test "$RC_NOW" -gt "$RC_BEFORE" && break
  sleep 2
done
```

```bash
RC_AFTER="$(kubectl get pod "$DSPOD" -n lab-4b \
  -o jsonpath='{.status.containerStatuses[0].restartCount}')"
UID_AFTER="$(kubectl get pod "$DSPOD" -n lab-4b -o jsonpath='{.metadata.uid}')"
NODE_AFTER="$(kubectl get pod "$DSPOD" -n lab-4b -o jsonpath='{.spec.nodeName}')"

echo "restartCount sau = $RC_AFTER"
echo "uid giữ nguyên?  = $([ "$UID_AFTER" = "$UID_BEFORE" ] && echo yes || echo no)"
echo "node             = $NODE_AFTER"

test "$RESTART_POLICY" = 'Always' \
  && test "$RC_AFTER" -gt "$RC_BEFORE" \
  && test "$UID_AFTER" = "$UID_BEFORE" \
  && echo 'PASS: kubelet khởi động lại container tại chỗ, object Pod không đổi'
```

**Ý nghĩa:** bài 38 xếp việc này ở tầng thấp nhất — *khởi động lại ở cấp container*, do **kubelet**
làm, theo `restartPolicy`. Pod template của DaemonSet bắt buộc `restartPolicy: Always` (hoặc bỏ
trống, mặc định là `Always`), nên container quay lại còn `metadata.uid` không đổi: vẫn đúng object
Pod cũ, chỉ có `restartCount` tăng.

**PASS:** dòng `PASS: kubelet khởi động lại container tại chỗ, object Pod không đổi` xuất hiện.

### B5.2. Tầng controller: Pod mất, node thì không

```bash
UID_BEFORE="$(kubectl get pod "$DSPOD" -n lab-4b -o jsonpath='{.metadata.uid}')"
kubectl delete pod "$DSPOD" -n lab-4b --wait=true

for i in $(seq 1 90); do
  NEWPOD="$(kubectl get pods -n lab-4b -l app=nodeprobe \
    --field-selector spec.nodeName=lab-k8s-worker2 \
    -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || true)"
  test -n "$NEWPOD" && break
  sleep 2
done
```

```bash
NEW_UID="$(kubectl get pod "$NEWPOD" -n lab-4b -o jsonpath='{.metadata.uid}')"
NEW_NODE="$(kubectl get pod "$NEWPOD" -n lab-4b -o jsonpath='{.spec.nodeName}')"
COUNT_W2="$(kubectl get pods -n lab-4b -l app=nodeprobe \
  --field-selector spec.nodeName=lab-k8s-worker2 -o name | wc -l)"

echo "Pod cũ = $DSPOD"
echo "Pod mới = $NEWPOD (uid=$NEW_UID, node=$NEW_NODE)"
echo "số Pod nodeprobe trên worker2 = $COUNT_W2"

test "$NEWPOD" != "$DSPOD" \
  && test "$NEW_UID" != "$UID_BEFORE" \
  && test "$NEW_NODE" = 'lab-k8s-worker2' \
  && test "$COUNT_W2" -eq 1 \
  && echo 'PASS: controller tạo Pod mới, tên khác nhưng đúng node cũ'
```

**Ý nghĩa:** đây là ranh giới mà bài 38 vẽ. Cả Pod mất thì **controller** vào cuộc, và nó không sửa
Pod cũ mà tạo Pod **mới**. Điểm phân biệt nằm ở cái được giữ:

| Controller | Cái được giữ | Cái đổi |
| --- | --- | --- |
| Deployment/ReplicaSet (4a) | **số lượng** replica | tên Pod và node đều có thể đổi |
| StatefulSet (B3.1) | **tên và ordinal** | uid đổi, node có thể đổi |
| DaemonSet (B5.2) | **node** | tên và uid đều đổi |

Bài 38 nói thẳng: "nếu một Pod thuộc DaemonSet bị lỗi, control plane sẽ tạo một Pod thay thế để
chạy **trên chính node đó**" — vì DaemonSet là tiện ích cấp node, thay thế sang node khác thì node
cũ mất dịch vụ.

**PASS:** dòng `PASS: controller tạo Pod mới, tên khác nhưng đúng node cũ` xuất hiện.

### B5.3. Xóa DaemonSet dọn luôn Pod của nó

```bash
kubectl delete daemonset nodeprobe -n lab-4b --wait=true

DS_LEFT="$(kubectl get daemonset nodeprobe -n lab-4b --ignore-not-found -o name | wc -l)"
for i in $(seq 1 60); do
  POD_LEFT="$(kubectl get pods -n lab-4b -l app=nodeprobe -o name | wc -l)"
  test "$POD_LEFT" -eq 0 && break
  sleep 2
done

test "$DS_LEFT" -eq 0 && test "$POD_LEFT" -eq 0 \
  && echo 'PASS: xóa DaemonSet dọn sạch Pod nó đã tạo'
```

**PASS:** dòng `PASS: xóa DaemonSet dọn sạch Pod nó đã tạo` xuất hiện.

## B6. Job — ba loại tác vụ và ba núm điều khiển

### B6.1. Job không song song: hai trường đều mặc định 1

```bash
cat > ~/lab-work/4b/job-simple.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: job-simple
  namespace: lab-4b
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "echo job-simple da chay; sleep 3"]
EOF

kubectl apply -f ~/lab-work/4b/job-simple.yaml
kubectl wait --for=condition=complete --timeout=300s job/job-simple -n lab-4b
```

```bash
COMPLETIONS="$(kubectl get job job-simple -n lab-4b -o jsonpath='{.spec.completions}')"
PARALLELISM="$(kubectl get job job-simple -n lab-4b -o jsonpath='{.spec.parallelism}')"
BACKOFF="$(kubectl get job job-simple -n lab-4b -o jsonpath='{.spec.backoffLimit}')"
SUCCEEDED="$(kubectl get job job-simple -n lab-4b -o jsonpath='{.status.succeeded}')"
PODS="$(kubectl get pods -n lab-4b -l batch.kubernetes.io/job-name=job-simple -o name | wc -l)"
PHASE="$(kubectl get pods -n lab-4b -l batch.kubernetes.io/job-name=job-simple \
  -o jsonpath='{.items[0].status.phase}')"

echo "completions=$COMPLETIONS parallelism=$PARALLELISM backoffLimit=$BACKOFF"
echo "succeeded=$SUCCEEDED  số Pod=$PODS  phase Pod=$PHASE"

test "$COMPLETIONS" = '1' && test "$PARALLELISM" = '1' && test "$BACKOFF" = '6' \
  && test "$SUCCEEDED" = '1' \
  && echo 'PASS: bỏ trống cả hai trường thì cả hai mặc định 1, backoffLimit mặc định 6'
test "$PODS" -eq 1 && test "$PHASE" = 'Succeeded' \
  && echo 'PASS: Job xong nhưng Pod và object Job vẫn còn lại'
```

**Ý nghĩa:** manifest không khai `completions`, `parallelism` hay `backoffLimit`, nhưng object đã
lưu có đủ ba giá trị — API server điền mặc định. Và đúng như bài 67: Job hoàn tất **không** làm Pod
hay object Job biến mất, vì bạn còn cần xem log và status. Ai dọn thì B8 trả lời.

**PASS:** hai dòng `PASS:` của B6.1 xuất hiện.

### B6.2. Số lần hoàn thành cố định: `parallelism` là trần, không phải mục tiêu

```bash
cat > ~/lab-work/4b/job-fixed.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: job-fixed
  namespace: lab-4b
spec:
  completions: 6
  parallelism: 2
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 8"]
EOF

kubectl apply -f ~/lab-work/4b/job-fixed.yaml

MAXACTIVE=0
for i in $(seq 1 180); do
  A="$(kubectl get job job-fixed -n lab-4b -o jsonpath='{.status.active}')"
  A="${A:-0}"
  test "$A" -gt "$MAXACTIVE" && MAXACTIVE="$A"
  S="$(kubectl get job job-fixed -n lab-4b -o jsonpath='{.status.succeeded}')"
  test "${S:-0}" -ge 6 && break
  sleep 1
done
echo "mức song song cao nhất quan sát được = $MAXACTIVE"
```

```bash
kubectl wait --for=condition=complete --timeout=300s job/job-fixed -n lab-4b
SUCCEEDED="$(kubectl get job job-fixed -n lab-4b -o jsonpath='{.status.succeeded}')"
PODS="$(kubectl get pods -n lab-4b -l batch.kubernetes.io/job-name=job-fixed -o name | wc -l)"

echo "succeeded = $SUCCEEDED | số Pod = $PODS | MAXACTIVE = $MAXACTIVE"
kubectl get pods -n lab-4b -l batch.kubernetes.io/job-name=job-fixed -o wide \
  | tee ~/lab-evidence/4b/b6-job-fixed.txt

test "$MAXACTIVE" -ge 1 && test "$MAXACTIVE" -le 2 \
  && echo 'PASS: không lúc nào quá 2 Pod chạy cùng lúc'
test "$SUCCEEDED" = '6' && test "$PODS" -eq 6 \
  && echo 'PASS: Job hoàn tất đúng khi đủ 6 Pod kết thúc thành công'
```

**Ý nghĩa:** `completions` là **tổng khối lượng**, `parallelism` là **trần đồng thời**. Bài 67 còn
nói mức song song thực tế "không vượt quá số lần hoàn thành còn lại" — ở đợt cuối chỉ còn 2 việc
nên `parallelism` cao hơn cũng vô nghĩa. Đây cũng là chỗ Job khác ReplicaSet ở 4a: ReplicaSet giữ
`replicas` Pod **sống mãi**, Job đếm Pod **kết thúc thành công** rồi dừng.

**PASS:** hai dòng `PASS:` của B6.2 xuất hiện.

### B6.3. Indexed Job: mỗi Pod biết phần việc của mình

```bash
cat > ~/lab-work/4b/job-indexed.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: job-indexed
  namespace: lab-4b
spec:
  completions: 5
  parallelism: 3
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: busybox:1.36
        command: ["sh", "-c"]
        args:
        - |
          set -eu
          items="foo bar baz qux xyz"
          item=$(echo "$items" | cut -d' ' -f $((JOB_COMPLETION_INDEX + 1)))
          echo "index=${JOB_COMPLETION_INDEX} item=${item}"
EOF

kubectl apply -f ~/lab-work/4b/job-indexed.yaml
kubectl wait --for=condition=complete --timeout=300s job/job-indexed -n lab-4b
```

```bash
IDXS="$(kubectl get job job-indexed -n lab-4b -o jsonpath='{.status.completedIndexes}')"
SUCCEEDED="$(kubectl get job job-indexed -n lab-4b -o jsonpath='{.status.succeeded}')"
echo "completedIndexes = $IDXS"
echo "succeeded        = $SUCCEEDED"

for p in $(kubectl get pods -n lab-4b -l batch.kubernetes.io/job-name=job-indexed \
             -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'); do
  LBL="$(kubectl get pod "$p" -n lab-4b \
    -o jsonpath='{.metadata.labels.batch\.kubernetes\.io/job-completion-index}')"
  echo "$p | label index=$LBL | log: $(kubectl logs "$p" -n lab-4b)"
done | sort -t= -k3 | tee ~/lab-evidence/4b/b6-job-indexed.txt
```

```bash
test "$IDXS" = '0-4' && test "$SUCCEEDED" = '5' \
  && echo 'PASS: đủ 5 index 0..4, mỗi index đúng một lần hoàn thành'

OK=1
for i in 0 1 2 3 4; do
  grep -q "index=$i " ~/lab-evidence/4b/b6-job-indexed.txt || { echo "FAIL: thiếu index $i"; OK=0; }
done
test "$OK" -eq 1 \
  && echo 'PASS: mỗi Pod nhận đúng một index qua biến JOB_COMPLETION_INDEX'
```

**Ý nghĩa:** với `completionMode: Indexed`, control plane gán mỗi Pod một index từ 0 tới
`completions-1` và công bố nó qua bốn đường; bài [353](../353-indexed-parallel-processing-vi.md)
chọn đường đơn giản nhất là biến môi trường `JOB_COMPLETION_INDEX`. Vì `parallelism` nhỏ hơn
`completions`, control plane phải chờ một số Pod xong rồi mới khởi chạy phần còn lại — bạn đã đo
điều đó ở B6.2.

**PASS:** hai dòng `PASS:` của B6.3 xuất hiện, không có dòng `FAIL:`.

### B6.4. Hình dạng Job hàng đợi công việc

```bash
cat > ~/lab-work/4b/job-queue.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: job-queue
  namespace: lab-4b
spec:
  parallelism: 3
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: busybox:1.36
        command: ["sh", "-c", "echo worker $(hostname) da lay het viec; sleep 5"]
EOF

kubectl apply -f ~/lab-work/4b/job-queue.yaml
kubectl wait --for=condition=complete --timeout=300s job/job-queue -n lab-4b
```

```bash
COMPLETIONS="$(kubectl get job job-queue -n lab-4b -o jsonpath='{.spec.completions}')"
PARALLELISM="$(kubectl get job job-queue -n lab-4b -o jsonpath='{.spec.parallelism}')"
SUCCEEDED="$(kubectl get job job-queue -n lab-4b -o jsonpath='{.status.succeeded}')"
PODS="$(kubectl get pods -n lab-4b -l batch.kubernetes.io/job-name=job-queue -o name | wc -l)"

echo "completions = ${COMPLETIONS:-<trống>}"
echo "parallelism = $PARALLELISM"
echo "succeeded   = $SUCCEEDED | số Pod = $PODS"

test -z "$COMPLETIONS" && test "$PARALLELISM" = '3' \
  && echo 'PASS: bỏ trống completions thì API server không điền mặc định cho nó'
test "$SUCCEEDED" = '3' && test "$PODS" -eq 3 \
  && echo 'PASS: Job hoàn tất khi mọi Pod đã kết thúc và có ít nhất một Pod thành công'
```

**Ý nghĩa:** so sánh với B6.1 cho thấy quy tắc defaulting của bài 67: bỏ trống **cả hai** thì cả
hai thành 1; chỉ đặt `parallelism` thì `completions` **ở nguyên trạng thái chưa đặt**. Đó chính là
hình dạng của loại tác vụ thứ ba — hàng đợi công việc: không ai biết trước có bao nhiêu việc, Job
kết thúc khi các Pod tự thoát.

Phần còn thiếu là **cái hàng đợi**. Bài [351](../351-coarse-parallel-work-queue-vi.md) dựng một
Service RabbitMQ và bài [352](../352-fine-parallel-work-queue-vi.md) dựng một Service Redis, cộng
với một image worker tự build. Service thuộc giai đoạn 5, và lab này không build/push image — nên
hai bài đó chỉ đọc. Cái bạn vừa kiểm chứng là **hợp đồng của Job**, phần mà Kubernetes chịu trách
nhiệm; phần phối hợp giữa các Pod là việc của ứng dụng.

**PASS:** hai dòng `PASS:` của B6.4 xuất hiện.

## B7. Job thất bại — `backoffLimit` chặn ở đâu

### B7.1. `restartPolicy: Never` — mỗi lần thử là một Pod mới

```bash
cat > ~/lab-work/4b/job-fail-never.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: job-fail-never
  namespace: lab-4b
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "echo lan chay nay se that bai; exit 1"]
EOF

kubectl apply -f ~/lab-work/4b/job-fail-never.yaml

for i in $(seq 1 120); do
  F="$(kubectl get job job-fail-never -n lab-4b \
    -o jsonpath='{.status.conditions[?(@.type=="Failed")].status}')"
  test "$F" = 'True' && break
  sleep 2
done
kubectl describe job job-fail-never -n lab-4b | tee ~/lab-evidence/4b/b7-fail-never.txt >/dev/null
```

```bash
FAILED_COND="$(kubectl get job job-fail-never -n lab-4b \
  -o jsonpath='{.status.conditions[?(@.type=="Failed")].status}')"
REASON="$(kubectl get job job-fail-never -n lab-4b \
  -o jsonpath='{.status.conditions[?(@.type=="Failed")].reason}')"
FAILED="$(kubectl get job job-fail-never -n lab-4b -o jsonpath='{.status.failed}')"
BACKOFF="$(kubectl get job job-fail-never -n lab-4b -o jsonpath='{.spec.backoffLimit}')"
PODS="$(kubectl get pods -n lab-4b -l batch.kubernetes.io/job-name=job-fail-never -o name | wc -l)"

echo "condition Failed = $FAILED_COND (reason=$REASON)"
echo "status.failed = $FAILED | backoffLimit = $BACKOFF | số Pod = $PODS"

test "$FAILED_COND" = 'True' && echo 'PASS: Job mang condition Failed'
case "$REASON" in
  BackoffLimitExceeded) echo "PASS: reason = $REASON" ;;
  *) echo "FAIL: reason ngoài dự kiến ($REASON) — đọc lại describe ở evidence" ;;
esac
test "${FAILED:-0}" -ge "$BACKOFF" && test "$PODS" -eq "$FAILED" \
  && echo 'PASS: mỗi lần thử lại là một Pod mới, và số Pod khớp status.failed'
```

**Ý nghĩa:** với `Never`, container hỏng làm **cả Pod** thất bại, và Job controller tạo Pod **mới**
với backoff tăng theo cấp số nhân. Số Pod bạn đếm được chính là số lần thử mà controller ghi vào
`status.failed` — ghi lại con số đó vào evidence thay vì học thuộc, vì nó phụ thuộc cách controller
đếm chứ không phải một hằng số bạn tự đặt.

**PASS:** ba dòng `PASS:` của B7.1 xuất hiện, không có dòng `FAIL:`.

### B7.2. Thất bại vĩnh viễn: Job không tự chạy lại

```bash
PODS_BEFORE="$(kubectl get pods -n lab-4b -l batch.kubernetes.io/job-name=job-fail-never \
  -o name | wc -l)"
sleep 60
PODS_AFTER="$(kubectl get pods -n lab-4b -l batch.kubernetes.io/job-name=job-fail-never \
  -o name | wc -l)"
ACTIVE="$(kubectl get job job-fail-never -n lab-4b -o jsonpath='{.status.active}')"

echo "số Pod trước = $PODS_BEFORE | sau = $PODS_AFTER | status.active = ${ACTIVE:-0}"

test "$PODS_AFTER" -eq "$PODS_BEFORE" && test "${ACTIVE:-0}" -eq 0 \
  && echo 'PASS: Job đã Failed thì đứng yên — không có Pod mới, không có Pod đang chạy'
```

**Ý nghĩa:** bài 67 nói rõ `restartPolicy` áp dụng cho **Pod**, không áp dụng cho **Job**: "không
có việc tự động khởi động lại Job khi status của Job là `type: Failed`". Muốn chạy lại thì phải xóa
Job và tạo lại — hoặc để một controller cấp cao hơn như CronJob tạo Job mới ở lịch kế tiếp, đúng
thứ bạn sẽ thấy ở B9.

**PASS:** dòng `PASS: Job đã Failed thì đứng yên...` xuất hiện.

### B7.3. `restartPolicy: OnFailure` — Pod ở lại, container chạy lại

```bash
cat > ~/lab-work/4b/job-fail-onfailure.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: job-fail-onfailure
  namespace: lab-4b
spec:
  backoffLimit: 6
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "echo lan chay nay se that bai; exit 1"]
EOF

kubectl apply -f ~/lab-work/4b/job-fail-onfailure.yaml

for i in $(seq 1 90); do
  P="$(kubectl get pods -n lab-4b -l batch.kubernetes.io/job-name=job-fail-onfailure \
    -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || true)"
  test -z "$P" && { sleep 2; continue; }
  RC="$(kubectl get pod "$P" -n lab-4b \
    -o jsonpath='{.status.containerStatuses[0].restartCount}' 2>/dev/null || true)"
  test -n "$RC" && test "$RC" -ge 1 && break
  sleep 2
done
```

```bash
PODS="$(kubectl get pods -n lab-4b -l batch.kubernetes.io/job-name=job-fail-onfailure \
  -o name | wc -l)"
RC="$(kubectl get pod "$P" -n lab-4b -o jsonpath='{.status.containerStatuses[0].restartCount}')"
echo "số Pod = $PODS | Pod = $P | restartCount = $RC"

test "$PODS" -eq 1 && test "$RC" -ge 1 \
  && echo 'PASS: OnFailure giữ nguyên một Pod và đếm restart của container'

kubectl delete job job-fail-onfailure -n lab-4b --wait=true
```

**Ý nghĩa:** đây là câu bẫy của bài 67. Hai giá trị `restartPolicy` hợp lệ cho Job cho hai hành vi
khác hẳn: `Never` tạo Pod mới (B7.1, số Pod tăng), `OnFailure` giữ Pod trên node và chạy lại
container tại chỗ (ở đây, số Pod đứng yên ở 1 còn `restartCount` tăng). Số lần thử lại cũng được
đếm theo hai cách tương ứng. Lab xóa Job ngay sau khi đo, vì bài khuyên dùng `Never` khi debug —
với `OnFailure`, Pod bị chấm dứt đúng lúc chạm giới hạn backoff nên rất dễ mất log.

**PASS:** dòng `PASS: OnFailure giữ nguyên một Pod và đếm restart của container` xuất hiện.

## B8. TTL dọn Job đã hoàn tất

```bash
cat > ~/lab-work/4b/job-ttl.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: job-ttl
  namespace: lab-4b
spec:
  ttlSecondsAfterFinished: 30
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "echo job-ttl da chay; sleep 3"]
EOF

kubectl apply -f ~/lab-work/4b/job-ttl.yaml
kubectl wait --for=condition=complete --timeout=300s job/job-ttl -n lab-4b

TTL="$(kubectl get job job-ttl -n lab-4b -o jsonpath='{.spec.ttlSecondsAfterFinished}')"
COMPLETION="$(kubectl get job job-ttl -n lab-4b -o jsonpath='{.status.completionTime}')"
COMPLETION_EPOCH="$(date -d "$COMPLETION" +%s)"
echo "ttlSecondsAfterFinished = $TTL"
echo "completionTime = $COMPLETION (epoch $COMPLETION_EPOCH)"
```

```bash
GONE_EPOCH=''
for i in $(seq 1 150); do
  if ! kubectl get job job-ttl -n lab-4b >/dev/null 2>&1; then
    GONE_EPOCH="$(date +%s)"
    break
  fi
  sleep 2
done

POD_LEFT="$(kubectl get pods -n lab-4b -l batch.kubernetes.io/job-name=job-ttl -o name | wc -l)"
SIMPLE_LEFT="$(kubectl get job job-simple -n lab-4b --ignore-not-found -o name | wc -l)"
ELAPSED=$(( ${GONE_EPOCH:-0} - COMPLETION_EPOCH ))

echo "job-ttl biến mất sau ${ELAPSED}s kể từ completionTime"
echo "Pod của job-ttl còn lại = $POD_LEFT"
echo "job-simple (không đặt TTL) còn lại = $SIMPLE_LEFT"
{
  echo "ttl=$TTL"
  echo "completionTime=$COMPLETION"
  echo "elapsed_after_completion=$ELAPSED"
} | tee ~/lab-evidence/4b/b8-ttl.txt

test -n "$GONE_EPOCH" && test "$ELAPSED" -ge "$TTL" \
  && echo 'PASS: đồng hồ TTL đếm từ lúc Job Complete, không phải từ lúc apply'
test "$POD_LEFT" -eq 0 \
  && echo 'PASS: controller xóa Job theo tầng — Pod đi theo Job'
test "$SIMPLE_LEFT" -eq 1 \
  && echo 'PASS: Job không đặt ttlSecondsAfterFinished thì không bao giờ được dọn'
```

**Ý nghĩa:** ba gate này là ba câu của bài [68](../68-ttlafterfinished-vi.md). Mốc bắt đầu đếm là
lúc **status condition** chuyển sang `Complete`/`Failed`, nên `ELAPSED` phải ≥ TTL — cận trên thì
phụ thuộc chu kỳ làm việc của controller, đừng coi nó là cam kết. Khi dọn, controller xóa **theo
tầng** nên Pod đi theo. Và `job-simple` từ B6.1 vẫn nằm đó, chứng minh câu "nếu không đặt trường
này, Job sẽ không được TTL controller dọn dẹp sau khi nó hoàn tất".

Bài 68 còn cảnh báo cơ chế này **nhạy với lệch thời gian**, vì nó so timestamp lưu trong object Job.
Cluster lab đã bật đồng bộ thời gian từ
[Lab 00 A4.1](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites) —
đó là lý do phép đo trên đáng tin.

**PASS:** ba dòng `PASS:` của B8 xuất hiện.

```bash
kubectl delete job job-simple job-fixed job-indexed job-queue job-fail-never \
  -n lab-4b --wait=true --ignore-not-found=true
JOB_LEFT="$(kubectl get jobs -n lab-4b -o name | wc -l)"
test "$JOB_LEFT" -eq 0 && echo 'PASS: đã xóa tay các Job không có TTL'
```

**Ý nghĩa:** "người dùng có trách nhiệm xóa các job cũ sau khi đã ghi nhận status của chúng" — bạn
vừa làm đúng việc đó. Xóa Job bằng `kubectl` cũng xóa mọi Pod nó đã tạo.

**PASS:** dòng `PASS: đã xóa tay các Job không có TTL` xuất hiện.

## B9. CronJob — lịch, chạy chồng và thời hạn khởi động

### B9.1. CronJob chỉ tạo Job; Job mới quản Pod

```bash
cat > ~/lab-work/4b/cron-hello.yaml <<'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cron-hello
  namespace: lab-4b
spec:
  schedule: "* * * * *"
  startingDeadlineSeconds: 30
  concurrencyPolicy: Allow
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    metadata:
      labels:
        lab: cron-hello
    spec:
      template:
        metadata:
          labels:
            lab: cron-hello
        spec:
          restartPolicy: OnFailure
          containers:
          - name: hello
            image: busybox:1.36
            command: ["sh", "-c", "date; echo hello from lab-4b"]
EOF

kubectl apply -f ~/lab-work/4b/cron-hello.yaml

for i in $(seq 1 90); do
  LS="$(kubectl get cronjob cron-hello -n lab-4b -o jsonpath='{.status.lastScheduleTime}')"
  test -n "$LS" && break
  sleep 2
done
kubectl get cronjob cron-hello -n lab-4b | tee ~/lab-evidence/4b/b9-cronjob.txt
kubectl get jobs -n lab-4b -l lab=cron-hello | tee -a ~/lab-evidence/4b/b9-cronjob.txt
```

```bash
LS="$(kubectl get cronjob cron-hello -n lab-4b -o jsonpath='{.status.lastScheduleTime}')"
JOBS="$(kubectl get jobs -n lab-4b -l lab=cron-hello -o name | wc -l)"
JOB1="$(kubectl get jobs -n lab-4b -l lab=cron-hello \
  --sort-by=.metadata.creationTimestamp -o name | sed 's|job.batch/||' | tail -1)"
OWNER="$(kubectl get job "$JOB1" -n lab-4b -o jsonpath='{.metadata.ownerReferences[0].kind}')"
STAMP="$(kubectl get job "$JOB1" -n lab-4b \
  -o jsonpath='{.metadata.annotations.batch\.kubernetes\.io/cronjob-scheduled-timestamp}')"

echo "lastScheduleTime = $LS"
echo "số Job = $JOBS | Job mới nhất = $JOB1 | owner = $OWNER"
echo "annotation cronjob-scheduled-timestamp = $STAMP"

test -n "$LS" && test "$JOBS" -ge 1 && test "$OWNER" = 'CronJob' \
  && echo 'PASS: CronJob đã tạo Job và Job mang ownerReference trỏ về CronJob'
test -n "$STAMP" \
  && echo 'PASS: Job mang annotation ghi mốc lịch mà nó được tạo cho'
```

```bash
POD1="$(kubectl get pods -n lab-4b -l batch.kubernetes.io/job-name="$JOB1" \
  -o jsonpath='{.items[0].metadata.name}')"
kubectl logs "$POD1" -n lab-4b | tee ~/lab-evidence/4b/b9-cronjob-log.txt

kubectl logs "$POD1" -n lab-4b | grep -q 'hello from lab-4b' \
  && echo 'PASS: đọc được log của Pod do CronJob sinh ra'
```

**Ý nghĩa:** bài [69](../69-cron-jobs-vi.md) đóng lại bằng đúng câu này — "CronJob chỉ chịu trách
nhiệm tạo các Job khớp với lịch của nó, còn Job đến lượt mình chịu trách nhiệm quản lý các Pod".
Ba tầng object bạn vừa đi qua: CronJob → Job → Pod, nối với nhau bằng owner reference, và tên Pod
khác tên Job đúng như ghi chú của bài [350](../350-automated-tasks-cron-jobs-vi.md).

**PASS:** ba dòng `PASS:` của B9.1 xuất hiện.

### B9.2. `concurrencyPolicy: Forbid` chặn lần chạy chồng

```bash
cat > ~/lab-work/4b/cron-forbid.yaml <<'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cron-forbid
  namespace: lab-4b
spec:
  schedule: "* * * * *"
  startingDeadlineSeconds: 30
  concurrencyPolicy: Forbid
  jobTemplate:
    metadata:
      labels:
        lab: cron-forbid
    spec:
      template:
        metadata:
          labels:
            lab: cron-forbid
        spec:
          restartPolicy: OnFailure
          containers:
          - name: slow
            image: busybox:1.36
            command: ["sh", "-c", "sleep 120"]
EOF

kubectl apply -f ~/lab-work/4b/cron-forbid.yaml

for i in $(seq 1 90); do
  A="$(kubectl get cronjob cron-forbid -n lab-4b -o jsonpath='{.status.active[*].name}' | wc -w)"
  test "$A" -ge 1 && break
  sleep 2
done
echo "đã có Job đang chạy, chờ qua ít nhất một mốc lịch nữa"
sleep 70
```

```bash
ACTIVE="$(kubectl get cronjob cron-forbid -n lab-4b -o jsonpath='{.status.active[*].name}' | wc -w)"
JOBS="$(kubectl get jobs -n lab-4b -l lab=cron-forbid -o name | wc -l)"
echo "status.active = $ACTIVE | số Job của cron-forbid = $JOBS"

kubectl get jobs -n lab-4b -l lab=cron-forbid | tee ~/lab-evidence/4b/b9-forbid.txt

test "$ACTIVE" -eq 1 && test "$JOBS" -eq 1 \
  && echo 'PASS: Forbid bỏ qua mốc lịch mới khi lần chạy trước chưa xong'
```

**Ý nghĩa:** Job chạy 120 giây trong khi lịch là mỗi phút, nên ít nhất một mốc rơi vào lúc lần chạy
trước còn sống. Với `Forbid`, CronJob **bỏ qua** lần chạy mới đó, và bài 69 nói mốc bị bỏ qua ấy
được **tính là bị lỡ (missed)**.

Hai điều bài nhấn mà bạn không được suy sai từ kết quả trên:

- Chính sách này **chỉ áp dụng trong phạm vi cùng một CronJob**. `cron-hello` vẫn tiếp tục chạy song
  song ở namespace này, và đó là hành vi đúng — nhiều CronJob thì Job của chúng luôn được phép chạy
  đồng thời.
- Khi Job đang chạy kết thúc, `startingDeadlineSeconds` **vẫn được tính đến** và có thể dẫn tới một
  lần chạy mới ngay lập tức. Đừng đọc `ACTIVE = 1` như "Forbid làm CronJob dừng hẳn".

**PASS:** dòng `PASS: Forbid bỏ qua mốc lịch mới khi lần chạy trước chưa xong` xuất hiện.

### B9.3. `startingDeadlineSeconds` và cửa sổ đếm lịch bị lỡ

```bash
SDS_HELLO="$(kubectl get cronjob cron-hello -n lab-4b \
  -o jsonpath='{.spec.startingDeadlineSeconds}')"
SDS_FORBID="$(kubectl get cronjob cron-forbid -n lab-4b \
  -o jsonpath='{.spec.startingDeadlineSeconds}')"
echo "cron-hello  startingDeadlineSeconds = $SDS_HELLO"
echo "cron-forbid startingDeadlineSeconds = $SDS_FORBID"

test "$SDS_HELLO" -ge 10 && test "$SDS_FORBID" -ge 10 \
  && echo 'PASS: cả hai CronJob đặt thời hạn khởi động không nhỏ hơn chu kỳ kiểm tra của controller'
```

**Ý nghĩa:** hai giá trị này đổi **hai** hành vi cùng lúc, và đó là lý do bài 69 xếp trường này vào
phần phải hiểu:

1. **Thời hạn khởi động.** Lỡ mốc lịch quá số giây này thì lần chạy đó bị bỏ qua, và Kubernetes coi
   nó là một Job **thất bại**. Không đặt trường này thì các lần chạy không có thời hạn.
2. **Cửa sổ đếm lịch bị lỡ.** Không đặt thì controller đếm từ lần lập lịch cuối tới hiện tại, và
   **quá 100 lịch bị lỡ** thì nó ngừng khởi động Job kèm log
   `too many missed start times...`. Có đặt thì nó chỉ đếm trong khoảng `startingDeadlineSeconds`
   gần nhất — đây chính là khác biệt giữa hai kịch bản `08:29:00`–`10:21:00` trong bài.

Con số 10 trong gate không tùy tiện: bài cảnh báo đặt nhỏ hơn 10 giây thì CronJob **có thể không
được lập lịch**, vì CronJob controller chỉ kiểm tra mỗi 10 giây một lần. Lab không dựng lại kịch bản
control plane chết gần hai tiếng để đo hành vi bù lịch — nó đòi tắt kube-controller-manager, tức
lệch baseline Lab 00.

**PASS:** dòng `PASS: cả hai CronJob đặt thời hạn khởi động...` xuất hiện.

### B9.4. `suspend` dừng lịch nhưng không dừng Job đang chạy

```bash
NEWEST_BEFORE="$(kubectl get jobs -n lab-4b -l lab=cron-hello \
  --sort-by=.metadata.creationTimestamp -o name | tail -1)"
kubectl patch cronjob cron-hello -n lab-4b --type=merge -p '{"spec":{"suspend":true}}'
SUSPEND="$(kubectl get cronjob cron-hello -n lab-4b -o jsonpath='{.spec.suspend}')"
echo "suspend = $SUSPEND | Job mới nhất trước khi chờ = $NEWEST_BEFORE"

sleep 90
```

```bash
NEWEST_AFTER="$(kubectl get jobs -n lab-4b -l lab=cron-hello \
  --sort-by=.metadata.creationTimestamp -o name | tail -1)"
echo "Job mới nhất sau khi chờ = $NEWEST_AFTER"

test "$SUSPEND" = 'true' && test "$NEWEST_AFTER" = "$NEWEST_BEFORE" \
  && echo 'PASS: CronJob bị tạm ngưng không tạo thêm Job nào dù đã qua nhiều mốc lịch'
```

**Ý nghĩa:** `suspend` không ảnh hưởng tới các Job đã khởi động trước đó, và các mốc lịch trôi qua
trong lúc tạm ngưng **được tính là bị lỡ**. Bài 69 kèm một cảnh báo phải nhớ trước khi bật lại: nếu
CronJob **không** có `startingDeadlineSeconds`, khi bạn đưa `suspend` về `false` thì các Job bị lỡ
sẽ được lập lịch **ngay lập tức**.

**PASS:** dòng `PASS: CronJob bị tạm ngưng không tạo thêm Job nào...` xuất hiện.

### B9.5. Xóa CronJob dọn cả Job lẫn Pod

```bash
kubectl delete cronjob cron-hello cron-forbid -n lab-4b --wait=true

for i in $(seq 1 90); do
  JOB_LEFT="$(kubectl get jobs -n lab-4b -o name | wc -l)"
  POD_LEFT="$(kubectl get pods -n lab-4b -o name | wc -l)"
  test "$JOB_LEFT" -eq 0 && test "$POD_LEFT" -eq 0 && break
  sleep 2
done

CJ_LEFT="$(kubectl get cronjobs -n lab-4b -o name | wc -l)"
echo "CronJob còn lại = $CJ_LEFT | Job còn lại = $JOB_LEFT | Pod còn lại = $POD_LEFT"

test "$CJ_LEFT" -eq 0 && test "$JOB_LEFT" -eq 0 && test "$POD_LEFT" -eq 0 \
  && echo 'PASS: xóa CronJob kéo theo toàn bộ Job và Pod nó đã tạo'
```

**Ý nghĩa:** bài 350 nói "việc xóa cron job sẽ xóa tất cả các job và Pod mà nó đã tạo ra, đồng thời
ngăn nó tạo thêm các job mới". Cơ chế vẫn là xóa theo tầng qua owner reference — cùng thứ bạn đã
thấy ở B3.3 và B8. Gate này quan trọng: một CronJob sót lại sẽ âm thầm sinh Job sau khi lab kết
thúc và làm cluster lệch khỏi `01-cluster-ready`.

**PASS:** dòng `PASS: xóa CronJob kéo theo toàn bộ Job và Pod nó đã tạo` xuất hiện.

## B10. Nhiều Job từ một template

```bash
cat > ~/lab-work/4b/job-tmpl.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: process-item-$ITEM
  namespace: lab-4b
  labels:
    jobgroup: jobexample
spec:
  template:
    metadata:
      labels:
        jobgroup: jobexample
    spec:
      restartPolicy: Never
      containers:
      - name: c
        image: busybox:1.36
        command: ["sh", "-c", "echo Processing item $ITEM && sleep 5"]
EOF

mkdir -p ~/lab-work/4b/jobs
for i in apple banana cherry; do
  sed "s/\$ITEM/$i/g" ~/lab-work/4b/job-tmpl.yaml > ~/lab-work/4b/jobs/job-$i.yaml
done
ls ~/lab-work/4b/jobs/
grep -h 'name: process-item' ~/lab-work/4b/jobs/*.yaml
```

```bash
FILES="$(ls ~/lab-work/4b/jobs/ | wc -l)"
test "$FILES" -eq 3 && echo 'PASS: template đã khai triển thành ba manifest'
grep -q '\$ITEM' ~/lab-work/4b/jobs/*.yaml \
  && echo 'FAIL: còn placeholder chưa thay' \
  || echo 'PASS: không manifest nào còn placeholder $ITEM'
```

**Ý nghĩa:** file `job-tmpl.yaml` **không phải** manifest Kubernetes hợp lệ — cú pháp `$ITEM` không
có nghĩa gì với Kubernetes. Bước khai triển xảy ra hoàn toàn ở phía client, trước khi `kubectl` gửi
đi bất cứ thứ gì.

```bash
kubectl create -f ~/lab-work/4b/jobs
for j in apple banana cherry; do
  kubectl wait --for=condition=complete --timeout=300s job/process-item-$j -n lab-4b
done

kubectl get jobs -n lab-4b -l jobgroup=jobexample | tee ~/lab-evidence/4b/b10-jobs.txt
kubectl get pods -n lab-4b -l jobgroup=jobexample | tee -a ~/lab-evidence/4b/b10-jobs.txt

for p in $(kubectl get pods -n lab-4b -l jobgroup=jobexample \
             -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'); do
  kubectl logs "$p" -n lab-4b
done | sort | tee ~/lab-evidence/4b/b10-logs.txt
```

```bash
JOBS="$(kubectl get jobs -n lab-4b -l jobgroup=jobexample -o name | wc -l)"
OK=1
for j in apple banana cherry; do
  S="$(kubectl get job process-item-$j -n lab-4b -o jsonpath='{.status.succeeded}')"
  test "$S" = '1' || { echo "FAIL: process-item-$j chưa succeeded=1"; OK=0; }
  grep -q "Processing item $j" ~/lab-evidence/4b/b10-logs.txt \
    || { echo "FAIL: thiếu log của $j"; OK=0; }
done
test "$JOBS" -eq 3 && test "$OK" -eq 1 \
  && echo 'PASS: ba Job độc lập, mỗi Job xử lý đúng một hạng mục'
```

**Ý nghĩa:** khác ba mẫu ở B6, mẫu này dùng **một object Job cho mỗi hạng mục công việc**, và giá
của nó là bạn phải quản lý nhiều object. Label `jobgroup=jobexample` là công cụ bù lại: một selector
duy nhất thao tác được cả nhóm — xem Job, xem Pod, đọc log, và xóa. Kubernetes không biết `jobgroup`
là gì; nó chỉ là label bạn tự đặt.

Phần Jinja2 của bài 355 **không chạy ở đây**: nó cần cài thêm gói Python bằng `pip`, còn lab này
không cài gì thêm. Bản `sed` đã chứng minh đủ mẫu hình; phần Jinja2 chỉ đổi công cụ sinh manifest.

**PASS:** dòng `PASS: ba Job độc lập, mỗi Job xử lý đúng một hạng mục` xuất hiện, không có dòng
`FAIL:`.

```bash
kubectl delete job -l jobgroup=jobexample -n lab-4b --wait=true
JOBS="$(kubectl get jobs -n lab-4b -l jobgroup=jobexample -o name | wc -l)"
test "$JOBS" -eq 0 && echo 'PASS: một lệnh theo label xóa cả nhóm Job'
```

**PASS:** dòng `PASS: một lệnh theo label xóa cả nhóm Job` xuất hiện.

## B11. Cleanup và gate cuối

**Mục đích:** xóa mọi object lab tạo ra và chứng minh cluster trở về `01-cluster-ready`.

```bash
kubectl delete namespace lab-4b --wait=true --timeout=300s

rm -f ~/lab-work/4b/jobs/job-apple.yaml ~/lab-work/4b/jobs/job-banana.yaml \
      ~/lab-work/4b/jobs/job-cherry.yaml
rmdir ~/lab-work/4b/jobs
rm -f ~/lab-work/4b/statefulset.yaml ~/lab-work/4b/daemonset.yaml \
      ~/lab-work/4b/job-simple.yaml ~/lab-work/4b/job-fixed.yaml \
      ~/lab-work/4b/job-indexed.yaml ~/lab-work/4b/job-queue.yaml \
      ~/lab-work/4b/job-fail-never.yaml ~/lab-work/4b/job-fail-onfailure.yaml \
      ~/lab-work/4b/job-ttl.yaml ~/lab-work/4b/cron-hello.yaml \
      ~/lab-work/4b/cron-forbid.yaml ~/lab-work/4b/job-tmpl.yaml
rmdir ~/lab-work/4b
```

```bash
kubectl get namespace lab-4b >/dev/null 2>&1 \
  && echo 'FAIL: namespace lab-4b vẫn còn' \
  || echo 'PASS: namespace lab-4b đã xóa'

CJ_ALL="$(kubectl get cronjobs -A --no-headers 2>/dev/null | wc -l)"
JOB_ALL="$(kubectl get jobs -A --no-headers 2>/dev/null | wc -l)"
echo "CronJob toàn cluster = $CJ_ALL | Job toàn cluster = $JOB_ALL"
test "$CJ_ALL" -eq 0 && test "$JOB_ALL" -eq 0 \
  && echo 'PASS: không còn CronJob hay Job nào trong cluster'

STS_ALL="$(kubectl get statefulsets -A --no-headers 2>/dev/null | wc -l)"
DS_LAB="$(kubectl get daemonsets -A --no-headers 2>/dev/null \
  | grep -vE '^(kube-system|kube-flannel)[[:space:]]' | wc -l)"
echo "StatefulSet toàn cluster = $STS_ALL | DaemonSet ngoài kube-system/kube-flannel = $DS_LAB"
test "$STS_ALL" -eq 0 && test "$DS_LAB" -eq 0 \
  && echo 'PASS: không còn StatefulSet, và DaemonSet chỉ còn của hệ thống'

test ! -e ~/lab-work/4b && echo 'PASS: manifest tạm đã xóa'

kubectl wait --for=condition=Ready node --all --timeout=180s
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
kubectl get svc -n default
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` biến điều đó thành
gate thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/4b/` **giữ lại** — đó là bằng chứng.

Gate CronJob toàn cluster là gate quan trọng nhất của mục này: một CronJob sót lại vẫn tiếp tục sinh
Job sau khi bạn đóng lab, và snapshot `01-cluster-ready` được khai báo là **không có workload**.

**PASS:** không có dòng `FAIL:` nào; bốn dòng `PASS:` xuất hiện; ba node `Ready`; lệnh field selector
trả `No resources found`; CoreDNS đủ replica; `default` không có Pod và chỉ còn Service `kubernetes`.
Cluster trở về `01-cluster-ready`; **không tạo snapshot mới**.

---

## 3. Checkpoint 4b

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] StatefulSet `web` 3 replica. Bạn xóa `web-1`. Pod thay thế tên gì, `uid` có giữ nguyên không,
      và nó có buộc phải lên đúng node cũ không?
- [ ] Với `podManagementPolicy` mặc định, `web-2` được tạo khi nào? Khi thu nhỏ từ 5 xuống 1 thì Pod
      nào bị kết thúc trước, và Pod nào sống sót?
- [ ] Xóa êm thấm và `--grace-period=0 --force` khác nhau ở chỗ nào, và vì sao cái thứ hai phá vỡ
      ngữ nghĩa "nhiều nhất một" của StatefulSet?
- [ ] Cluster ba node của bạn, DaemonSet không có toleration nào cho ra mấy Pod? Thêm gì để thành
      ba, và vì sao con số ban đầu không phải ba?
- [ ] DaemonSet controller có tự đặt `.spec.nodeName` không? Nếu không thì ai đặt, và controller làm
      gì để chỉ định đúng node?
- [ ] Kể ba toleration mà DaemonSet controller tự thêm. Cái nào **chỉ** được thêm cho Pod dùng
      `hostNetwork`, và nếu không có nó thì cluster bế tắc ra sao?
- [ ] Container trong Pod chết thì ai xử lý và theo trường nào? Cả Pod mất thì ai xử lý và nó làm gì
      khác đi? Deployment, StatefulSet và DaemonSet mỗi cái cam kết giữ **cái gì** cho Pod thay thế?
- [ ] Một Job chỉ đặt `parallelism: 3` và bỏ trống `completions`. Đây là loại tác vụ nào, Job hoàn
      tất khi nào, và vì sao `completions` **không** bị điền mặc định thành 1?
- [ ] `backoffLimit` đạt giới hạn thì Job có tự chạy lại không? `restartPolicy: Never` và `OnFailure`
      cho hai hành vi khác nhau thế nào khi container thoát với mã khác 0?
- [ ] `ttlSecondsAfterFinished: 30` cho một Job chạy 10 phút. Job đủ điều kiện bị xóa vào lúc nào
      tính từ khi `apply`, và Pod của nó ra sao?
- [ ] CronJob A và CronJob B đều đặt `concurrencyPolicy: Forbid`. Job của A có bị chặn vì Job của B
      đang chạy không? `startingDeadlineSeconds` đổi **hai** hành vi nào?
- [ ] **Ba món nợ của nhóm 4b:** kể tên từng món, nói vì sao lab không làm được, và lab nào trả.
      (Gợi ý: `volumeClaimTemplates`, Service headless quản trị, HPA/VPA.)

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng sau, không nhìn tài liệu:

1. **Luồng danh tính.** Bạn `apply` một StatefulSet 3 replica. Ai tạo Pod, theo thứ tự nào, chờ điều
   kiện gì giữa hai Pod liên tiếp; mỗi Pod nhận định danh gồm những phần nào; phần nào của định danh
   đó **chưa** hoạt động trên cluster lab và vì sao. Sau đó bạn xóa `web-1`: mô tả cái được giữ, cái
   mất, và khác biệt nếu đó là Pod của một Deployment hay của một DaemonSet.
2. **Luồng tác vụ.** Bạn `apply` một CronJob lịch mỗi phút. Kể chuỗi object được sinh ra tới tận Pod,
   ai chịu trách nhiệm ở từng tầng; điều gì xảy ra khi một lần chạy vượt qua mốc lịch kế tiếp với
   `Allow` và với `Forbid`; container trong Pod thoát với mã khác 0 thì ai đếm, đếm theo cách nào, và
   Job dừng lại ở đâu; cuối cùng ai dọn Job đã xong và mốc đếm giờ bắt đầu từ đâu.

Khi toàn bộ checkbox được đánh dấu và không còn nhầm **số lượng** với **danh tính**, `completions`
với `parallelism`, `restartPolicy` của Pod với việc thử lại của Job, hay tầng kubelet với tầng
controller — Lab 4b hoàn tất.

Ba món nợ #1, #2 và #3 vẫn **chưa trả**. Đừng coi giai đoạn 4 là đóng cho tới khi bạn làm xong Lab
5a (Service headless), Lab 6a (`volumeClaimTemplates`) và Lab 11b (HPA/VPA) — xem
[Sổ nợ lộ trình](../00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình).

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Bảng dưới chỉ liệt kê sự cố phát sinh từ nội dung bài học 4b.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| B1.2 fail vì `web-0` đã `Ready` | Bạn chạy hai khối cách nhau quá lâu | `kubectl delete -f ~/lab-work/4b/statefulset.yaml --wait=true` rồi chạy lại hai khối liền nhau |
| Pod StatefulSet kẹt ở `Pending` | `kubectl describe pod web-0 -n lab-4b` | Kiểm tra headroom node theo [tầng 2 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a543-tầng-2--node-condition-taint-và-podcidr); không thêm node, không đặt `nodeSelector` |
| B2.2 không bắt được `deletionTimestamp` của `web-4` | `kubectl get pod web-4 -n lab-4b -o yaml` | Vòng lặp chạy chậm hơn `preStop`; scale lại lên 5, chờ ready rồi lặp lại — không rút ngắn `preStop` |
| B3.2 báo `T_FORCE` ≥ `T_GRACE` | `echo "$T_GRACE $T_FORCE"` | Hai biến phải đo trong cùng phiên shell. Nếu mất biến, chạy lại B3.1 rồi B3.2 |
| B4.1 cho `DESIRED` khác `NODE_SCHEDULABLE` | `kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.taints[*].key}{"\n"}{end}'` | Có worker mang taint lạ — cluster lệch baseline; dừng và đối chiếu Lab 00, không tự gỡ taint |
| B5.1 `restartCount` không tăng | `kubectl describe pod "$DSPOD" -n lab-4b` | `kill 1` có thể bị nuốt nếu shell không phải PID 1; chạy lại `kubectl exec -n lab-4b "$DSPOD" -- sh -c 'kill 1'` và chờ thêm vòng lặp |
| B7.1 `reason` không phải `BackoffLimitExceeded` | `~/lab-evidence/4b/b7-fail-never.txt` | Đọc `reason` thật trong describe; gate chính là condition `Failed=True`, reason chỉ để đối chiếu |
| B8 hết 150 vòng mà `job-ttl` vẫn còn | `kubectl get job job-ttl -n lab-4b -o yaml` | Kiểm tra `.spec.ttlSecondsAfterFinished` và `.status.completionTime`; nếu đồng hồ ba VM lệch, xem lại đồng bộ thời gian ở Lab 00 A4.1 |
| B9.1 mãi không có `lastScheduleTime` | `kubectl describe cronjob cron-hello -n lab-4b` | Kiểm tra kube-controller-manager đang `Running`; `schedule` sai cú pháp thì API đã từ chối từ lúc apply |
| B9.2 thấy 2 Job của `cron-forbid` | Thời điểm bạn đo rơi sau khi Job 120 giây đã kết thúc | Xóa CronJob, tạo lại và đo lại ngay sau khi Job đầu tiên `active` |
| B9.4 vẫn có Job mới sau khi suspend | `kubectl get cronjob cron-hello -n lab-4b -o jsonpath='{.spec.suspend}'` | Patch chưa ăn; chạy lại `kubectl patch` rồi đo lại từ đầu mục |
| Xóa namespace `lab-4b` treo ở `Terminating` | `kubectl get all -n lab-4b`; `kubectl get pod -n lab-4b -o yaml` | Pod còn trong `preStop`/grace period — chờ hết. Không gỡ finalizer Namespace |
| Sau cleanup vẫn còn Job lạ | `kubectl get jobs -A` | Còn CronJob nào chưa xóa? `kubectl get cronjobs -A` rồi xóa CronJob trước, Job sẽ đi theo |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — StatefulSets](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes v1.35 — DaemonSet](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [Kubernetes v1.35 — Jobs](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubernetes v1.35 — Automatic Cleanup for Finished Jobs](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)
- [Kubernetes v1.35 — CronJob](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Kubernetes v1.35 — Autoscaling Workloads](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/autoscaling/)
- [Kubernetes v1.35 — Kubernetes Self-Healing](https://v1-35.docs.kubernetes.io/docs/concepts/architecture/self-healing/)
- [Kubernetes v1.35 — Scale a StatefulSet](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/scale-stateful-set/)
- [Kubernetes v1.35 — Delete a StatefulSet](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/delete-stateful-set/)
- [Kubernetes v1.35 — Force Delete StatefulSet Pods](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/)
- [Kubernetes v1.35 — Run Jobs](https://v1-35.docs.kubernetes.io/docs/tasks/job/)
- [Kubernetes v1.35 — Running Automated Tasks with a CronJob](https://v1-35.docs.kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/)
- [Kubernetes v1.35 — Indexed Job for Parallel Processing with Static Work Assignment](https://v1-35.docs.kubernetes.io/docs/tasks/job/indexed-parallel-processing-static/)
- [Kubernetes v1.35 — Parallel Processing using Expansions](https://v1-35.docs.kubernetes.io/docs/tasks/job/parallel-processing-expansion/)
