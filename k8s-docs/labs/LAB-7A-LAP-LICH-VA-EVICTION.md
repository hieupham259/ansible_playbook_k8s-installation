# Lab 7a — Lập lịch và eviction

> **Điểm bắt đầu:** snapshot `03-storage-ready` — mốc do Lab 6a tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot).
> **Điểm kết thúc:** cleanup trả cluster về đúng `03-storage-ready`, **không tạo snapshot mới**.
> Lab này không cài thêm bất kỳ thành phần hạ tầng nào.
> **Lab trước:** Lab 6b — Snapshot và volume nâng cao (chưa viết, xem
> [bản đồ lab](README.md#4-bản-đồ-lab)) cũng trả cluster về `03-storage-ready`. Cluster vào lab
> này phải ở đúng mốc đó: có StorageClass mặc định, không còn object nào của lab trước.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục
[7a. Scheduling và eviction](../00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction): mười ba bài
khái niệm 136–148 cộng ba bài thực hành 267, 266, 210.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không
chép lại con số phiên bản nào** và **không cài thêm gì**. Thành phần ngoài baseline mà bạn thấy
đang chạy — CNI thay Flannel, ingress controller, dynamic provisioner — đã được khóa ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) và do Lab 5b, Lab 6a
cài; lab 7a chỉ **đọc** chúng, không đụng vào.

Topology vẫn là một control plane `lab-k8s-master` mang taint `NoSchedule` và hai worker
`lab-k8s-worker1`, `lab-k8s-worker2`. **Hai worker là hai miền topology duy nhất** của cluster
này — đó là dữ kiện quyết định của B3, B5 và B6.

Lab dùng Pod trần, Deployment, Service, PodDisruptionBudget của các giai đoạn đã học làm công
cụ. **Không** dùng ResourceQuota, LimitRange (nhóm
[7b](../00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên)), RBAC
([giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy)), metrics-server
và HPA ([giai đoạn 11](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability)), DRA
([giai đoạn 13](../00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao)), và
**không** chạy `kubectl drain`, `cordon`, `uncordon` — ba lệnh đó thuộc
[giai đoạn 16](../00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node).

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`, rồi thêm hai lệnh riêng của lab này:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Hai lệnh riêng của lab 7a: xác nhận đúng điểm bắt đầu và bàn cờ lập lịch còn sạch.
kubectl get storageclass | grep -q '(default)' \
  && echo 'PASS: co StorageClass mac dinh — dung diem bat dau 03-storage-ready'
kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints'
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả
`No resources found`; CoreDNS đủ replica `READY`; namespace `default` không có Pod; dòng
`PASS: co StorageClass mac dinh …` xuất hiện; **chỉ `lab-k8s-master` có taint**, hai worker ở
cột `TAINTS` là `<none>`. Nếu một worker đã mang taint từ trước, dừng lại và restore snapshot —
mọi phép thử của B4, B5, B6 dựa trên việc hai worker sạch taint.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Chu trình hai bước của kube-scheduler: **lọc** tạo ra tập node khả thi, **chấm điểm** xếp
  hạng chính tập đó — và đọc được cả hai bước trong `Events` của một Pod `Pending`.
- Scheduler **không chạy container**: sản phẩm của nó là một binding; `nodeName` viết tay bỏ
  qua toàn bộ chu trình đó và bỏ qua luôn taint `NoSchedule`.
- Ba cách ràng buộc theo label node — `nodeSelector`, node affinity `required`, node affinity
  `preferred` — khác nhau ở đâu, và vì sao `IgnoredDuringExecution` không đuổi Pod đang chạy.
- Quy tắc kết hợp: nhiều `nodeSelectorTerms` được **OR**, nhiều biểu thức trong một
  `matchExpressions` được **AND**, và đặt cả `nodeSelector` lẫn `nodeAffinity` thì cả hai phải
  thỏa.
- Inter-pod anti-affinity `required` với `topologyKey: kubernetes.io/hostname` cho **mỗi node
  đúng một bản sao**, nên replica thứ ba trên cluster hai worker nằm `Pending` — và đổi sang
  `preferred` thì nó chạy.
- Taint là **lực đẩy của node**, toleration là **giấy phép của Pod**: toleration cho phép chứ
  không đảm bảo, `NoSchedule` không đuổi Pod đang chạy, `NoExecute` thì có, và
  `tolerationSeconds` quyết định Pod còn ở lại bao lâu.
- Nhiều taint được xử lý như một **bộ lọc**: chỉ cần một taint không được dung thứ với effect
  `NoSchedule` là Pod không lên được node.
- Ràng buộc phân bố theo topology: `maxSkew` đo chênh lệch so với mức tối thiểu toàn cục,
  `DoNotSchedule` giữ Pod ở `Pending`, `ScheduleAnyway` thì không — và **node thiếu
  `topologyKey` bị scheduler bỏ qua hoàn toàn**.
- PriorityClass sắp thứ tự hàng đợi, và chỉ khi Pod vẫn không lập lịch được thì **preemption**
  mới chạy: đọc được `nominatedNodeName`, chỉ ra đúng nạn nhân, và giải thích vì sao preemption
  **không** loại bỏ hết Pod ưu tiên thấp hơn.
- Hai thứ cùng tên "eviction" nhưng đối lập: eviction qua API **tôn trọng** PDB và
  `terminationGracePeriodSeconds`; eviction do áp lực node là phản xạ tự vệ của kubelet, không
  tôn trọng cả hai. Đọc được ngưỡng eviction **thật** của kubelet trên node lab và tính được
  cluster đang cách ngưỡng đó bao xa.
- `schedulingGates` giữ Pod ở `SchedulingGated` — khác hẳn `Pending` vì thiếu tài nguyên — và
  chỉ gỡ được, không thêm được sau khi Pod đã tạo.
- Cleanup đúng phạm vi: gỡ hết taint, xóa PriorityClass, gỡ hết label lab đặt lên node, và đưa
  cluster về `03-storage-ready`.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 7a | Phần lab kiểm chứng |
| --- | --- |
| [136 — Lập lịch, Preemption và Eviction](../136-scheduling-eviction-vi.md) | B0.2 — trang mục lục; lab tìm trong cluster đúng ba hiện vật ứng với ba từ khóa: Pod chưa được lập lịch, `PriorityClass`, subresource `pods/eviction` |
| [137 — Bộ lập lịch của Kubernetes](../137-kube-scheduler-vi.md) | B1 — binding do `default-scheduler` ghi, bước lọc loại node qua `Events`, danh sách khả thi rỗng thì Pod chưa được lập lịch, `nodeName` bỏ qua scheduler |
| [138 — Gán Pod cho Node](../138-assign-pod-node-vi.md) | B2 (`nodeSelector`, node affinity `required`/`preferred`, OR/AND, `IgnoredDuringExecution`), B3 (inter-pod affinity và anti-affinity), B1.3 và B4.5 (`nodeName`) |
| [267 — Gán Pod vào Node](../267-assign-pods-nodes-vi.md) | B2.1–B2.2 — gắn label cho node rồi dùng `nodeSelector`, đúng trình tự của trang task |
| [266 — Gán Pod vào Node bằng Node Affinity](../266-assign-pods-nodes-node-affinity-vi.md) | B2.3–B2.4 — cùng label đó, lần này qua `requiredDuringScheduling…` rồi `preferredDuringScheduling…` |
| [139 — Taint và Toleration](../139-taint-and-toleration-vi.md) | B4 — đọc taint thật của control plane, tự taint `lab-k8s-worker2`, quy tắc khớp `Equal`/`Exists`, nhiều taint như bộ lọc, `NoExecute` và `tolerationSeconds`; B8.3 — taint tích hợp sẵn theo điều kiện node và toleration tự thêm |
| [140 — Ràng buộc phân bố Pod theo topology](../140-topology-spread-constraints-vi.md) | B5 — `maxSkew` theo `kubernetes.io/hostname`, `DoNotSchedule` so với `ScheduleAnyway`, giao rỗng với `nodeSelector`, và node thiếu `topologyKey` bị bỏ qua |
| [141 — Độ ưu tiên và Preemption của Pod](../141-pod-priority-preemption-vi.md) | B6 — PriorityClass cao/thấp, làm hết chỗ trên `lab-k8s-worker2`, `nominatedNodeName`, chọn nạn nhân tối thiểu, PDB chỉ được tôn trọng ở mức nỗ lực tốt nhất |
| [210 — Bảo đảm lập lịch cho các Pod add-on quan trọng](../210-guaranteed-scheduling-critical-addon-pods-vi.md) | B6.1 — hai PriorityClass tích hợp sẵn và add-on thật của cluster đang dùng chúng |
| [143 — Eviction khởi phát qua API](../143-api-eviction-vi.md) | B7 — tạo object `Eviction`, `200 OK` khi không có PDB, `429` khi PDB chặn, grace period được tôn trọng, và Pod rời EndpointSlice **trước khi** container chết |
| [142 — Eviction do áp lực node](../142-node-pressure-eviction-vi.md) | B8 — đọc `evictionHard` thật của kubelet, tính khoảng cách tới ngưỡng `nodefs.available`, ba điều kiện node và ánh xạ sang taint, toleration tự thêm của Pod thường và của DaemonSet. Phần kích hoạt eviction thật xem bảng bên dưới |
| [145 — Mức sẵn sàng lập lịch của Pod](../145-pod-scheduling-readiness-vi.md) | B9 — `schedulingGates`, trạng thái `SchedulingGated`, gỡ từng gate, cấm thêm gate sau khi tạo, và quy tắc chỉ được siết chặt chỉ thị lập lịch |
| [144 — Pod Overhead](../144-pod-overhead-vi.md) | B10.1 — trường `spec.overhead` có trong API của baseline, cluster không có RuntimeClass nào khai `overhead` nên Pod không mang khoản cộng thêm nào |
| [146 — Tinh chỉnh hiệu năng bộ lập lịch](../146-scheduler-perf-tuning-vi.md) | B10.2 — kube-scheduler chạy bằng cấu hình mặc định, và cluster ba node luôn nằm dưới ngưỡng mà `percentageOfNodesToScore` bắt đầu có tác dụng |
| [147 — Scheduling Framework](../147-scheduling-framework-vi.md) | B10.3 — ánh xạ bằng chứng đã thu được ở B1, B6, B9 vào đúng ba điểm mở rộng `Filter`/`Score`, `PostFilter`, `PreEnqueue` |
| [148 — Đóng gói tài nguyên](../148-resource-bin-packing-vi.md) | B10.4 — `NodeResourcesFit` đang dùng chiến lược chấm điểm mặc định vì không có `KubeSchedulerConfiguration` nào được nạp |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài [142](../142-node-pressure-eviction-vi.md) — kích hoạt eviction do áp lực node thật và quan sát thứ tự trục xuất | Trên bố cục của cluster lab, `nodefs` **chính là filesystem gốc của node**: ép `nodefs.available` xuống dưới `10%` đồng nghĩa với làm đầy đĩa host, việc mà [mục 2](#2-quy-ước-và-an-toàn) cấm tuyệt đối. Ép `memory.available` xuống dưới `100Mi` thì nạn nhân đầu tiên có thể là chính `kubelet` và `containerd`. Không có cách nào giới hạn phép thử này vào một volume riêng của lab, nên lab **không làm**. B8 thay bằng đọc ngưỡng thật và tính headroom; thứ tự trục xuất đọc ở mục *Lựa chọn pod cho eviction của kubelet* |
| Bài [142](../142-node-pressure-eviction-vi.md) — toleration `node.kubernetes.io/memory-pressure` mà control plane thêm cho Pod khác `BestEffort` | Chỉ thấy được **hệ quả** khi node thật sự ở `MemoryPressure`: Pod `BestEffort` mới không lên node đó còn Pod `Burstable` thì lên được. Cùng lý do an toàn như dòng trên |
| Bài [141](../141-pod-priority-preemption-vi.md) — `preemptionPolicy: Never` | Bài xếp mục này vào phần đọc lướt, dành cho workload dạng job/batch; thực hành ở [giai đoạn 13](../00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao), bài [150](../150-gang-scheduling-vi.md) |
| Bài [144](../144-pod-overhead-vi.md) — `spec.overhead` mang giá trị thật | Cần một RuntimeClass khai `overhead.podFixed`, tức một container runtime ảo hóa kiểu Kata/Firecracker nằm ngoài bảng A1.3. Cài thêm runtime là đổi hạ tầng và tạo mốc mới, trái với điểm kết thúc của lab |
| Bài [146](../146-scheduler-perf-tuning-vi.md) — đổi `percentageOfNodesToScore` và đo tác động; bài [148](../148-resource-bin-packing-vi.md) — bật `MostAllocated` hoặc `RequestedToCapacityRatio` | Cả hai đòi sửa `KubeSchedulerConfiguration` của control plane; sửa cấu hình cluster đang chạy thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). Riêng bài 146 còn không có gì để đo: cluster ba node luôn nằm dưới ngưỡng 100 node khả thi |
| Bài [147](../147-scheduling-framework-vi.md) — viết plugin hoặc chạy scheduler thứ hai | Phải biên dịch scheduler hoặc triển khai scheduler riêng; thuộc [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) |
| Bài [140](../140-topology-spread-constraints-vi.md) — `minDomains`, `matchLabelKeys`, `nodeAffinityPolicy`, `nodeTaintsPolicy`, ràng buộc mặc định cấp cluster | Bài xếp cả bốn vào phần đọc lướt; hai cái đầu cần node pool co giãn hoặc workload nhiều revision, hai cái sau và ràng buộc mặc định lại phải sửa cấu hình scheduler |
| Bài [138](../138-assign-pod-node-vi.md) — `NodeRestriction` và tiền tố `node-restriction.kubernetes.io/` | Là thao tác hardening, cần Node authorizer và admission controller; thuộc [giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) |
| Bài [139](../139-taint-and-toleration-vi.md) — toán tử so sánh số `Gt`/`Lt` cho toleration | Tính năng alpha ở phiên bản baseline, phải bật feature gate `TaintTolerationComparisonOperators`; bật feature gate là làm lệch baseline |
| Bài [143](../143-api-eviction-vi.md) — `kubectl drain` như một quy trình bảo trì | `drain`, `cordon`, `uncordon` thuộc [giai đoạn 16](../00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node). B7 gọi thẳng subresource `eviction`, tức đúng cơ chế nằm dưới `drain`, mà không dùng lệnh đó |

Đây **không phải nợ lộ trình mới**. Chúng là giới hạn cố định của môi trường lab hoặc là nội
dung đã có chỗ đứng sẵn ở giai đoạn sau; [sổ nợ lab](README.md#5-sổ-nợ-lab) không thay đổi vì
lab 7a.

### 1.2. Thời lượng

3–4 giờ, tính từ lúc gate mở đầu đã PASS. B4, B6 và B7 có bước phải chờ controller hội tụ hoặc
chờ hết grace period; thời gian chờ **phụ thuộc cấu hình cluster**, nên mọi bước chờ đều viết
dưới dạng vòng lặp có điều kiện dừng chứ không phải một con số cố định.

---

## 2. Quy ước và an toàn

> **Cảnh báo — lab này KHÔNG làm đầy đĩa để kích hoạt eviction, và bạn cũng đừng tự thêm bước
> đó.** Trên cluster lab, `nodefs` của kubelet nằm trên filesystem gốc của node; mọi cách đẩy
> `nodefs.available` xuống dưới ngưỡng `10%` đều là làm đầy đĩa host, và một node hết đĩa thì
> `etcd`, `containerd`, `kubelet` cùng hỏng chứ không chỉ Pod bị trục xuất. Tương tự với
> `memory.available`. B8 kiểm chứng phần **đọc được**: ngưỡng thật trong cấu hình kubelet, giá
> trị tín hiệu thật trên node, và khoảng cách giữa hai thứ đó. Nếu bạn muốn thấy eviction thật,
> làm nó trên cluster dùng một lần, không phải trên cluster mang chuỗi snapshot của lộ trình.

Các quy ước còn lại:

- Mọi lệnh chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, **trừ khi ghi rõ node
  khác**. Các bước phải chạy trên node khác đều mở đầu bằng dòng "Chạy trên `<node>`".
- **Chạy toàn bộ phần B trong cùng một phiên shell.** Nhiều gate so sánh biến và hàm đặt ở B0
  (`cpu2m`, `W2_FREE_M`, `W2_AVAIL_KB0`…); mở shell mới giữa chừng là mất hết.
- **Fault injection chỉ trên `lab-k8s-worker2`.** Taint (B4), làm hết chỗ để ép preemption (B6)
  và mọi phép thử có thể đuổi Pod đều ghim vào đúng node này. `lab-k8s-worker1` chỉ nhận label
  và Pod bình thường.
- Lab tạo Namespace `lab-7a` và các object bên trong nó, cộng **năm thứ nằm ngoài namespace**:
  hai PriorityClass `lab-7a-low` và `lab-7a-high`, ba label lab đặt lên node (`disktype`,
  `lab7a-zone`, `lab7a-preempt`) và các taint tạm trên `lab-k8s-worker2`. Toàn bộ năm nhóm này
  bị xóa ở B11.
- **Không gỡ taint của control plane.** `node-role.kubernetes.io/control-plane:NoSchedule` trên
  `lab-k8s-master` là dữ kiện của bài học, không phải chướng ngại; gỡ nó là đổi hành vi lập
  lịch của mọi lab sau, đúng như [A5.4.3 của Lab
  00](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready) đã dặn.
- **Không sửa cấu hình kube-scheduler và kubelet.** B8 và B10 chỉ **đọc**
  `/etc/kubernetes/manifests/kube-scheduler.yaml` và endpoint `configz` của kubelet. Một thao
  tác ghi nhầm ở thư mục manifest làm chết control plane và phải restore snapshot.
- **Không đụng vào `kube-system`**, vào namespace của CNI, của ingress controller, hay của
  dynamic provisioner. B4.6 dùng taint `NoExecute` và có thể **đẩy Pod hạ tầng đang nằm trên
  `lab-k8s-worker2` sang `lab-k8s-worker1`** — đó là hành vi đúng của `NoExecute`, không phải
  hỏng hóc; các Pod đó không tự quay lại sau khi gỡ taint và gate cuối chỉ yêu cầu chúng
  `Running`. Bước đó chụp lại danh sách Pod trên worker2 trước và sau để bạn đối chiếu.
- Image dùng cho toàn bộ lab là `busybox:1.37` đã khóa ở
  [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và đã có sẵn trên cả ba node
  từ Lab 00, nên lab không phụ thuộc mạng ra ngoài. Các bài gốc dùng `nginx`, `redis` hay
  `registry.k8s.io/pause`; lab thay bằng `busybox:1.37` và **không đổi gì khác** trong manifest.
- Manifest tạm ghi vào `~/lab-work/7a/`; bằng chứng ghi vào `~/lab-evidence/7a/`. Dòng bắt đầu
  bằng `PASS:` là gate; không đi tiếp khi gate fail.

---

# Phần B — Thực hành kiến thức 7a

## B0. Chuẩn bị workspace và ảnh nền của bàn cờ lập lịch

**Mục đích:** ghi lại trạng thái node **trước** khi lab đụng vào — taint, label, allocatable,
dung lượng đĩa — để B11 có cái đối chiếu, và dựng sẵn hai hàm tính toán mà B6 cần.

```bash
mkdir -p ~/lab-work/7a ~/lab-evidence/7a
kubectl config current-context
kubectl create namespace lab-7a
kubectl get namespace lab-7a -o jsonpath='{.status.phase}'; echo
```

```bash
{
  echo "=== $(date -Is) — ban co lap lich truoc Lab 7a ==="
  echo '--- nodes ---'
  kubectl get nodes -o wide
  echo '--- taints ---'
  kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints'
  echo '--- labels ---'
  kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels}{"\n"}{end}'
  echo '--- allocatable ---'
  kubectl get nodes -o custom-columns='NODE:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory,PODS:.status.allocatable.pods'
  echo '--- priorityclass ---'
  kubectl get priorityclass
} | tee ~/lab-evidence/7a/b0-truoc.txt
```

```bash
LAB_LABELS="$(kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.labels}{"\n"}{end}' \
  | grep -cE 'disktype|lab7a' || true)"
WORKER_TAINTS="$(kubectl get nodes -o jsonpath='{.items[?(@.spec.taints)].metadata.name}' \
  | tr ' ' '\n' | grep -c 'worker' || true)"

test "$LAB_LABELS" -eq 0 \
  && echo 'PASS: chua node nao mang label cua lab 7a'
test "$WORKER_TAINTS" -eq 0 \
  && echo 'PASS: hai worker chua mang taint nao'
```

**Ý nghĩa:** ba thứ trên là **bàn cờ** mà mọi bài của nhóm 7a can thiệp vào: label để thu hút,
taint để đẩy ra, allocatable để quyết định còn chỗ hay không. Chụp lại chúng bây giờ là cách
duy nhất để B11 chứng minh được lab đã trả bàn cờ về nguyên trạng.

**PASS:** namespace `lab-7a` ở phase `Active`; hai dòng `PASS:` của bước này xuất hiện.

### B0.1. Hai hàm đọc số thật từ cluster

B6 cần biết `lab-k8s-worker2` **còn trống bao nhiêu CPU**. Không hard-code con số nào: đọc
`allocatable` và phần đã bị đặt trước từ chính cluster.

```bash
cpu2m() {
  case "$1" in
    '') echo 0 ;;
    *m) echo "${1%m}" ;;
    *)  awk -v x="$1" 'BEGIN{printf "%d\n", x*1000}' ;;
  esac
}

node_used_cpu_m() {
  cpu2m "$(kubectl describe node "$1" \
    | awk '/^Allocated resources/,/^Events/' \
    | awk '$1=="cpu"{print $2; exit}')"
}

W2_ALLOC_M="$(cpu2m "$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.allocatable.cpu}')")"
W2_USED_M="$(node_used_cpu_m lab-k8s-worker2)"
W2_FREE_M=$(( W2_ALLOC_M - W2_USED_M ))
echo "worker2: allocatable=${W2_ALLOC_M}m requested=${W2_USED_M}m free=${W2_FREE_M}m" \
  | tee ~/lab-evidence/7a/b0-worker2-cpu.txt

W2_AVAIL_KB0="$(ssh lab-k8s-worker2 'df -Pk /var/lib/kubelet | tail -1' | awk '{print $4}')"
echo "worker2: /var/lib/kubelet available = ${W2_AVAIL_KB0} KiB" \
  | tee -a ~/lab-evidence/7a/b0-worker2-cpu.txt
```

```bash
test "$W2_ALLOC_M" -gt 0 && test "$W2_FREE_M" -ge 400 \
  && echo "PASS: worker2 con ${W2_FREE_M}m CPU trong, du de dung tinh huong preemption o B6"
test "$W2_AVAIL_KB0" -gt 0 \
  && echo "PASS: doc duoc dung luong trong cua nodefs tren worker2"
```

**Ý nghĩa:** `allocatable` là con số mà bộ lập lịch thật sự đối chiếu — bài
[137](../137-kube-scheduler-vi.md) gọi bộ lọc đó là `PodFitsResources`, và Lab 3c đã chứng minh
nó không so với mức dùng thực tế. `W2_AVAIL_KB0` được ghi lại ở đây để B11 chứng minh lab
**không tiêu một byte đĩa nào** của worker2 — đúng cam kết ở mục 2.

**PASS:** hai dòng `PASS:` của bước này xuất hiện. Nếu `W2_FREE_M` nhỏ hơn 400, có workload còn
sót từ lab trước; tìm và xóa nó rồi chạy lại B0.1.

### B0.2. Ba từ khóa của nhóm 7a, ba hiện vật trong cluster

Bài [136](../136-scheduling-eviction-vi.md) là trang mục lục và chỉ định nghĩa ba từ: **lập
lịch** ghép Pod với Node, **preemption** chấm dứt Pod ưu tiên thấp để Pod ưu tiên cao lập lịch
được, **eviction** chấm dứt Pod đang nằm trên Node. Ba từ đó không trừu tượng — mỗi từ có một
hiện vật đọc được ngay trong cluster.

```bash
{
  echo '--- lap lich: Pod chua duoc gan Node thi scheduler moi nhin toi ---'
  kubectl explain pod.spec.nodeName | head -n 6
  echo '--- preemption: PriorityClass la mot kind that ---'
  kubectl api-resources --api-group=scheduling.k8s.io
  echo '--- eviction: subresource cua Pod ---'
  kubectl get --raw /api/v1 | tr ',' '\n' | grep 'pods/eviction'
} | tee ~/lab-evidence/7a/b0-ba-tu-khoa.txt

grep -q 'priorityclasses' ~/lab-evidence/7a/b0-ba-tu-khoa.txt \
  && echo 'PASS: preemption co object dieu khien rieng — PriorityClass'
grep -q 'pods/eviction' ~/lab-evidence/7a/b0-ba-tu-khoa.txt \
  && echo 'PASS: eviction la mot subresource cua Pod, khong phai mot lenh kubectl'
```

**Ý nghĩa:** ranh giới chia đôi cả nhóm bài nằm ở thời điểm: **lập lịch xảy ra trước khi Pod
chạy**, còn preemption và eviction tác động vào Pod **đã có chỗ**. B1–B5 và B9 nằm ở nửa trước,
B6–B8 nằm ở nửa sau.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B1. Chu trình filter rồi score

Bài [137](../137-kube-scheduler-vi.md) mô tả đúng hai bước: **lọc** tìm tập node khả thi, rồi
**chấm điểm** xếp hạng chính tập đó; danh sách sau bước lọc rỗng thì Pod ở trạng thái chưa được
lập lịch chứ **không phải lỗi**. Mục này đọc cả hai bước từ `Events` của Pod thật.

### B1.1. Sản phẩm của scheduler là một binding, không phải một container

```bash
cat > ~/lab-work/7a/b1-plain.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: plain
  namespace: lab-7a
  labels:
    app: b1
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/7a/b1-plain.yaml
kubectl apply -f ~/lab-work/7a/b1-plain.yaml
kubectl wait --for=condition=Ready pod/plain -n lab-7a --timeout=180s

kubectl get pod plain -n lab-7a -o wide
kubectl get event -n lab-7a --field-selector involvedObject.name=plain \
  -o custom-columns='REASON:.reason,SOURCE:.source.component,MSG:.message' \
  | tee ~/lab-evidence/7a/b1-events-plain.txt
```

```bash
PLAIN_NODE="$(kubectl get pod plain -n lab-7a -o jsonpath='{.spec.nodeName}')"
echo "plain -> $PLAIN_NODE"

grep -q 'default-scheduler' ~/lab-evidence/7a/b1-events-plain.txt \
  && echo 'PASS: quyet dinh dat Pod do default-scheduler ghi, khong phai kubelet'
case "$PLAIN_NODE" in
  lab-k8s-worker1|lab-k8s-worker2) echo "PASS: Pod thuong roi xuong worker ($PLAIN_NODE)" ;;
  *) echo "FAIL: Pod thuong khong duoc phep len $PLAIN_NODE" ;;
esac
```

**Ý nghĩa:** scheduler theo dõi các Pod **chưa được gán Node**, chọn node, rồi **thông báo cho
API server** — quá trình đó gọi là binding. Việc chạy container là của kubelet. Bạn thấy đúng
hai dấu vết đó: event `Scheduled` do `default-scheduler` phát, và các event `Pulled`/`Created`/
`Started` do `kubelet` phát ngay sau.

Pod không bao giờ rơi xuống `lab-k8s-master` vì node đó bị loại ở **bước lọc** — lý do chính
xác nằm ở B1.2.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không có dòng `FAIL:`.

### B1.2. Danh sách khả thi rỗng: Pod chưa được lập lịch, không phải Pod lỗi

Xin nhiều CPU hơn `allocatable` của node lớn nhất. Con số dưới đây **tính từ cluster**, không
bịa:

```bash
MAX_ALLOC_M=0
for n in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  M="$(cpu2m "$(kubectl get node "$n" -o jsonpath='{.status.allocatable.cpu}')")"
  test "$M" -gt "$MAX_ALLOC_M" && MAX_ALLOC_M="$M"
done
TOO_BIG_M=$(( MAX_ALLOC_M + 500 ))
echo "node lon nhat co ${MAX_ALLOC_M}m CPU; se xin ${TOO_BIG_M}m"

sed -e 's/name: plain/name: too-big/' \
    -e "s/cpu: \"20m\"/cpu: \"${TOO_BIG_M}m\"/" \
    ~/lab-work/7a/b1-plain.yaml > ~/lab-work/7a/b1-too-big.yaml
kubectl apply -f ~/lab-work/7a/b1-too-big.yaml

for i in $(seq 1 30); do
  REASON="$(kubectl get pod too-big -n lab-7a \
    -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
  test "$REASON" = 'Unschedulable' && break
  sleep 2
done

kubectl get pod too-big -n lab-7a
kubectl get pod too-big -n lab-7a \
  -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].message}' \
  | tee ~/lab-evidence/7a/b1-unschedulable.txt; echo
```

```bash
PHASE="$(kubectl get pod too-big -n lab-7a -o jsonpath='{.status.phase}')"
REASON="$(kubectl get pod too-big -n lab-7a \
  -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
NODE="$(kubectl get pod too-big -n lab-7a -o jsonpath='{.spec.nodeName}')"
echo "phase=$PHASE reason=$REASON nodeName='${NODE}'"

test "$PHASE" = 'Pending' && test "$REASON" = 'Unschedulable' && test -z "$NODE" \
  && echo 'PASS: Pod o trang thai chua duoc lap lich, chua duoc gan node nao'
grep -q 'Insufficient cpu' ~/lab-evidence/7a/b1-unschedulable.txt \
  && echo 'PASS: buoc loc bao dung ly do tai nguyen — Insufficient cpu'
grep -q 'untolerated taint' ~/lab-evidence/7a/b1-unschedulable.txt \
  && echo 'PASS: buoc loc bao ca ly do taint cua control plane'
```

**Ý nghĩa:** một thông điệp duy nhất chứa **cả hai loại lý do lọc**: control plane bị loại vì
taint không được dung thứ, hai worker bị loại vì thiếu CPU. Đó chính là bước lọc đang liệt kê
lý do cho từng node. Bước chấm điểm **không hề chạy** — không có node nào đi tới đó.

Pod này `Pending`, không `Failed`: bài 137 nói rõ nếu không có node nào phù hợp thì pod ở trạng
thái chưa được lập lịch **cho đến khi** scheduler sắp đặt được nó.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

Scheduler **không bỏ rơi** Pod chưa lập lịch được: nó giữ Pod trong hàng đợi và thử lại khi
trạng thái cluster đổi — B2.2 chứng minh điều đó bằng một Pod `Pending` tự chạy sau khi bạn gắn
label cho node. Đây cũng là lý do bài [145](../145-pod-scheduling-readiness-vi.md) tồn tại: Pod
nằm chờ lâu gây xáo trộn cho scheduler, và `schedulingGates` ở B9 là cách nói "đừng thử vội".

### B1.3. `nodeName` bỏ qua toàn bộ chu trình

```bash
cat > ~/lab-work/7a/b1-direct.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: direct
  namespace: lab-7a
  labels:
    app: b1
spec:
  nodeName: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b1-direct.yaml
kubectl wait --for=condition=Ready pod/direct -n lab-7a --timeout=180s
kubectl get event -n lab-7a --field-selector involvedObject.name=direct \
  -o custom-columns='REASON:.reason,SOURCE:.source.component' \
  | tee ~/lab-evidence/7a/b1-events-direct.txt
```

```bash
SCHED_EVENTS="$(kubectl get event -n lab-7a \
  --field-selector involvedObject.name=direct,reason=Scheduled --no-headers 2>/dev/null | wc -l)"
DNODE="$(kubectl get pod direct -n lab-7a -o jsonpath='{.spec.nodeName}')"
echo "direct -> $DNODE | so event Scheduled = $SCHED_EVENTS"

test "$DNODE" = 'lab-k8s-worker2' && test "$SCHED_EVENTS" -eq 0 \
  && echo 'PASS: Pod chay dung node ma khong co event Scheduled nao — scheduler bo qua Pod nay'
```

**Ý nghĩa:** `nodeName` không rỗng thì **bộ lập lịch bỏ qua Pod** và kubelet trên node có tên
đó cố đặt Pod lên node. Không có bước lọc, không có bước chấm điểm, nên cũng không có event
`Scheduled`. Bài 138 cảnh báo rõ: nếu node không đủ tài nguyên, Pod **thất bại** với lý do
`OutOfcpu`/`OutOfmemory` chứ không được sắp xếp lại chỗ khác. B4.5 sẽ dùng lại `direct` để cho
thấy hệ quả thứ hai — nó lên được cả node đang mang taint `NoSchedule`.

**PASS:** dòng `PASS: Pod chay dung node…` xuất hiện.

```bash
kubectl delete pod plain too-big -n lab-7a --wait=true --timeout=120s
```

Giữ lại Pod `direct` — B4.5 và B4.6 còn dùng.

---

## B2. `nodeSelector` và node affinity — thu hút Pod bằng label của node

Bài [138](../138-assign-pod-node-vi.md) gộp ba cơ chế cùng dựa trên **label của node** vào một
trang, và hai trang task [267](../267-assign-pods-nodes-vi.md),
[266](../266-assign-pods-nodes-node-affinity-vi.md) đi đúng trình tự đó: gắn label trước, rồi
lần lượt dùng `nodeSelector`, `required`, `preferred`.

Mọi ví dụ trong ba bài dùng label zone (`topology.kubernetes.io/zone`, `antarctica-east1`);
cluster lab **không node nào có label zone**, nên lab dùng `kubernetes.io/hostname` cho miền
topology và một label tự đặt `disktype` cho phần còn lại. Quy tắc y hệt, chỉ đổi miền.

### B2.1. Gắn label cho một node

```bash
kubectl label nodes lab-k8s-worker1 disktype=ssd
kubectl get nodes -L disktype | tee ~/lab-evidence/7a/b2-labels.txt
```

```bash
W1_DISK="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.metadata.labels.disktype}')"
W2_DISK="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.metadata.labels.disktype}')"
echo "worker1.disktype='${W1_DISK}' worker2.disktype='${W2_DISK}'"

test "$W1_DISK" = 'ssd' && test -z "$W2_DISK" \
  && echo 'PASS: dung mot node mang label disktype=ssd'
```

**Ý nghĩa:** node là object có label như mọi object khác; Kubernetes tự điền một tập label
chuẩn (`kubernetes.io/hostname`, `kubernetes.io/os`, `kubernetes.io/arch`…) và bạn thêm label
của mình bên cạnh. Bài 138 dặn: nếu dùng label để **cô lập** node thì phải chọn khóa mà kubelet
không tự sửa được — đó là phần `NodeRestriction` của giai đoạn 9, ở đây chưa dùng tới.

**PASS:** dòng `PASS: dung mot node mang label disktype=ssd` xuất hiện.

### B2.2. `nodeSelector` chỉ lên node có **đủ tất cả** label được liệt kê

```bash
cat > ~/lab-work/7a/b2-selector.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: sel-nvme
  namespace: lab-7a
  labels:
    app: b2
spec:
  nodeSelector:
    disktype: nvme
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b2-selector.yaml

for i in $(seq 1 30); do
  R="$(kubectl get pod sel-nvme -n lab-7a \
    -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
  test "$R" = 'Unschedulable' && break
  sleep 2
done
kubectl get pod sel-nvme -n lab-7a \
  -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].message}' \
  | tee ~/lab-evidence/7a/b2-selector-pending.txt; echo
```

```bash
grep -q 'node affinity/selector' ~/lab-evidence/7a/b2-selector-pending.txt \
  && echo 'PASS: buoc loc tu choi vi khong node nao khop nodeSelector'
```

Bây giờ gắn đúng label đó cho `lab-k8s-worker2` và **không tạo lại Pod**:

```bash
kubectl label nodes lab-k8s-worker2 disktype=nvme

for i in $(seq 1 30); do
  N="$(kubectl get pod sel-nvme -n lab-7a -o jsonpath='{.spec.nodeName}')"
  test -n "$N" && break
  sleep 2
done
kubectl get pod sel-nvme -n lab-7a -o wide
```

```bash
N="$(kubectl get pod sel-nvme -n lab-7a -o jsonpath='{.spec.nodeName}')"
test "$N" = 'lab-k8s-worker2' \
  && echo 'PASS: chinh Pod cu duoc lap lich sau khi node co du label' \
  || echo "FAIL: sel-nvme dang o '${N}'"

kubectl delete pod sel-nvme -n lab-7a --wait=true --timeout=120s
kubectl label nodes lab-k8s-worker2 disktype-
```

**Ý nghĩa:** hai điều cùng lúc. Một, `nodeSelector` là ràng buộc **cứng**: Pod chỉ lên node có
**đủ tất cả** label bạn liệt kê. Hai, Pod chưa lập lịch được **không chết**; scheduler giữ nó
trong hàng đợi và lập lịch ngay khi cluster đổi trạng thái — ở đây là khi một node vừa có đủ
label.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không có dòng `FAIL:`.

### B2.3. Node affinity `required` — cùng hiệu lực, giàu cú pháp hơn

```bash
cat > ~/lab-work/7a/b2-required.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: aff-required
  namespace: lab-7a
  labels:
    app: b2
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b2-required.yaml
kubectl wait --for=condition=Ready pod/aff-required -n lab-7a --timeout=180s
kubectl get pod aff-required -n lab-7a -o wide
```

```bash
N="$(kubectl get pod aff-required -n lab-7a -o jsonpath='{.spec.nodeName}')"
test "$N" = 'lab-k8s-worker1' \
  && echo 'PASS: node affinity required dua Pod ve dung node mang label'
```

**Ý nghĩa:** `requiredDuringSchedulingIgnoredDuringExecution` hoạt động **giống**
`nodeSelector`, khác ở khả năng biểu đạt: có `operator` (`In`, `NotIn`, `Exists`,
`DoesNotExist`, và `Gt`/`Lt` chỉ dành cho node affinity), có nhiều term, và có bản mềm ở B2.4.

**PASS:** dòng `PASS: node affinity required…` xuất hiện.

### B2.4. Node affinity `preferred` — không thỏa vẫn được lập lịch

```bash
cat > ~/lab-work/7a/b2-preferred.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: aff-pref-hit
  namespace: lab-7a
  labels:
    app: b2
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: aff-pref-miss
  namespace: lab-7a
  labels:
    app: b2
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - khong-node-nao-co
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b2-preferred.yaml
kubectl wait --for=condition=Ready pod/aff-pref-hit pod/aff-pref-miss -n lab-7a --timeout=180s
kubectl get pods -n lab-7a -l app=b2 -o wide | tee ~/lab-evidence/7a/b2-preferred.txt
```

```bash
HIT="$(kubectl get pod aff-pref-hit -n lab-7a -o jsonpath='{.spec.nodeName}')"
MISS_PHASE="$(kubectl get pod aff-pref-miss -n lab-7a -o jsonpath='{.status.phase}')"
echo "aff-pref-hit -> $HIT | aff-pref-miss phase=$MISS_PHASE"

test "$HIT" = 'lab-k8s-worker1' \
  && echo 'PASS: quy tac mem co weight keo Pod ve node thoa man'
test "$MISS_PHASE" = 'Running' \
  && echo 'PASS: quy tac mem khong thoa duoc thi Pod van duoc lap lich'
```

**Ý nghĩa:** đây là chỗ **duy nhất trong nhóm 7a mà bạn nhìn thấy bước chấm điểm làm việc**.
`weight` từ 1 đến 100 được cộng vào tổng điểm của node thỏa mãn quy tắc preferred, rồi cộng
tiếp vào điểm của các hàm ưu tiên khác; node điểm cao nhất thắng. Với `aff-pref-miss`, không
node nào cộng được điểm gì nên bước chấm điểm chỉ còn các hàm mặc định — nhưng **tập khả thi
không hề bị thu hẹp**, nên Pod vẫn chạy. Đó chính là ranh giới cứng/mềm.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B2.5. Nhiều term thì OR, nhiều biểu thức trong một term thì AND

```bash
cat > ~/lab-work/7a/b2-or-and.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: aff-or
  namespace: lab-7a
  labels:
    app: b2
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - khong-node-nao-co
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: aff-and
  namespace: lab-7a
  labels:
    app: b2
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
          - key: kubernetes.io/hostname
            operator: In
            values:
            - lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b2-or-and.yaml
kubectl wait --for=condition=Ready pod/aff-or -n lab-7a --timeout=180s

for i in $(seq 1 30); do
  R="$(kubectl get pod aff-and -n lab-7a \
    -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
  test "$R" = 'Unschedulable' && break
  sleep 2
done
kubectl get pods -n lab-7a -o wide | tee -a ~/lab-evidence/7a/b2-preferred.txt
```

```bash
OR_NODE="$(kubectl get pod aff-or -n lab-7a -o jsonpath='{.spec.nodeName}')"
AND_REASON="$(kubectl get pod aff-and -n lab-7a \
  -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
echo "aff-or -> $OR_NODE | aff-and reason=$AND_REASON"

test "$OR_NODE" = 'lab-k8s-worker2' \
  && echo 'PASS: hai nodeSelectorTerms duoc OR — thoa mot term la du'
test "$AND_REASON" = 'Unschedulable' \
  && echo 'PASS: hai matchExpressions trong cung mot term duoc AND'
```

**Ý nghĩa:** đây là chỗ dễ nhầm ngược nhất của bài 138, vì cả hai cùng nằm trong một khối YAML
lồng nhau. `aff-and` đòi một node vừa có `disktype=ssd` (chỉ worker1) vừa tên `lab-k8s-worker2`
— giao rỗng, nên Pod nằm chờ mãi.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B2.6. Đặt cả `nodeSelector` lẫn `nodeAffinity` thì **cả hai** phải thỏa

```bash
cat > ~/lab-work/7a/b2-both.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: aff-both
  namespace: lab-7a
  labels:
    app: b2
spec:
  nodeSelector:
    disktype: ssd
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b2-both.yaml
for i in $(seq 1 30); do
  R="$(kubectl get pod aff-both -n lab-7a \
    -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
  test "$R" = 'Unschedulable' && break
  sleep 2
done
kubectl get pod aff-both -n lab-7a
```

```bash
R="$(kubectl get pod aff-both -n lab-7a \
  -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
test "$R" = 'Unschedulable' \
  && echo 'PASS: nodeSelector va nodeAffinity duoc AND voi nhau, khong phai OR'
```

**Ý nghĩa:** `nodeSelector` nói "worker1", `nodeAffinity` nói "worker2"; ghi chú của bài 138 nói
**cả hai phải được thỏa mãn** thì Pod mới lên node, nên giao rỗng và Pod `Pending`. Nếu chúng
được OR với nhau thì Pod đã chạy — đây là phép thử phân biệt hai giả thuyết.

**PASS:** dòng `PASS: nodeSelector va nodeAffinity duoc AND…` xuất hiện.

### B2.7. `IgnoredDuringExecution` — đổi label không đuổi Pod đang chạy

`aff-required` từ B2.3 đang chạy trên `lab-k8s-worker1` nhờ label `disktype=ssd`. Gỡ chính
label đó đi:

```bash
kubectl get pod aff-required -n lab-7a -o wide
kubectl label nodes lab-k8s-worker1 disktype-
kubectl get nodes -L disktype

for i in $(seq 1 10); do sleep 3; done
kubectl get pod aff-required -n lab-7a -o wide | tee -a ~/lab-evidence/7a/b2-preferred.txt
```

```bash
W1_DISK="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.metadata.labels.disktype}')"
PHASE="$(kubectl get pod aff-required -n lab-7a -o jsonpath='{.status.phase}')"
NODE="$(kubectl get pod aff-required -n lab-7a -o jsonpath='{.spec.nodeName}')"
echo "worker1.disktype='${W1_DISK}' | aff-required phase=$PHASE node=$NODE"

test -z "$W1_DISK" && test "$PHASE" = 'Running' && test "$NODE" = 'lab-k8s-worker1' \
  && echo 'PASS: label bien mat nhung Pod van chay — affinity chi danh gia luc lap lich'
```

**Ý nghĩa:** `IgnoredDuringExecution` nghĩa là **nếu label của node thay đổi sau khi Kubernetes
đã lập lịch Pod, Pod vẫn tiếp tục chạy**. Muốn có hành vi đẩy Pod đang chạy ra khỏi node thì
phải dùng cơ chế khác — taint effect `NoExecute` ở B4.6. Đó chính là lý do bài
[139](../139-taint-and-toleration-vi.md) tồn tại bên cạnh bài 138 chứ không thay thế nó.

**PASS:** dòng `PASS: label bien mat nhung Pod van chay…` xuất hiện.

```bash
kubectl delete pod aff-required aff-pref-hit aff-pref-miss aff-or aff-and aff-both \
  -n lab-7a --wait=true --timeout=120s
```

---

## B3. Inter-pod affinity và anti-affinity — ràng buộc theo label của **Pod khác**

Hai mục trước ràng buộc theo label của **node**. Inter-pod affinity ràng buộc theo label của
**các Pod đang chạy** trong một miền topology do `topologyKey` chỉ ra. Bài
[138](../138-assign-pod-node-vi.md), mục *Các trường hợp sử dụng thực tế hơn*, dựng đúng cặp
mẫu redis-cache + web-server; lab dựng lại cặp đó trên hai worker, đổi image sang
`busybox:1.37`.

Với `podAntiAffinity` loại `requiredDuringSchedulingIgnoredDuringExecution`, admission
controller `LimitPodHardAntiAffinityTopology` giới hạn `topologyKey` **chỉ được là**
`kubernetes.io/hostname` — nên đây cũng là giá trị duy nhất lab dùng.

### B3.1. Mỗi node đúng một bản sao

```bash
cat > ~/lab-work/7a/b3-store.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store
  namespace: lab-7a
spec:
  replicas: 3
  selector:
    matchLabels:
      app: store
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: app
        image: busybox:1.37
        command: ["sh", "-c", "sleep 3600"]
        resources:
          requests:
            cpu: "20m"
            memory: "16Mi"
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/7a/b3-store.yaml
kubectl apply -f ~/lab-work/7a/b3-store.yaml

for i in $(seq 1 60); do
  RUN="$(kubectl get pods -n lab-7a -l app=store --field-selector=status.phase=Running \
    --no-headers 2>/dev/null | wc -l)"
  PEND="$(kubectl get pods -n lab-7a -l app=store --field-selector=status.phase=Pending \
    --no-headers 2>/dev/null | wc -l)"
  test "$RUN" -eq 2 && test "$PEND" -eq 1 && break
  sleep 3
done
kubectl get pods -n lab-7a -l app=store -o wide | tee ~/lab-evidence/7a/b3-store.txt
```

```bash
RUN="$(kubectl get pods -n lab-7a -l app=store --field-selector=status.phase=Running \
  --no-headers | wc -l)"
PEND="$(kubectl get pods -n lab-7a -l app=store --field-selector=status.phase=Pending \
  --no-headers | wc -l)"
DISTINCT="$(kubectl get pods -n lab-7a -l app=store \
  -o jsonpath='{range .items[?(@.spec.nodeName)]}{.spec.nodeName}{"\n"}{end}' | sort -u | wc -l)"
echo "running=$RUN pending=$PEND distinct_nodes=$DISTINCT"

test "$RUN" -eq 2 && test "$PEND" -eq 1 \
  && echo 'PASS: chi 2 replica chay duoc — moi node dung mot ban sao'
test "$DISTINCT" -eq 2 \
  && echo 'PASS: hai replica dang chay nam tren hai node khac nhau'
```

**Ý nghĩa:** anti-affinity `required` với `topologyKey: kubernetes.io/hostname` yêu cầu bộ lập
lịch **tránh đặt nhiều Pod khớp selector lên cùng một node** — bài gọi đó là "tạo mỗi cache
trên một node riêng biệt". Cluster lab chỉ có **hai miền `hostname` nhận Pod thường**, nên
replica thứ ba không có miền nào trống. Vì đây là loại **cứng**, nó nằm `Pending` chứ không dồn
lên node đã có Pod.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B3.2. `podAffinity` kéo Pod về đúng node đang có Pod bạn đồng hành

```bash
cat > ~/lab-work/7a/b3-web.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-store
  namespace: lab-7a
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-store
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: app
        image: busybox:1.37
        command: ["sh", "-c", "sleep 3600"]
        resources:
          requests:
            cpu: "20m"
            memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b3-web.yaml

for i in $(seq 1 60); do
  RUN="$(kubectl get pods -n lab-7a -l app=web-store --field-selector=status.phase=Running \
    --no-headers 2>/dev/null | wc -l)"
  test "$RUN" -eq 2 && break
  sleep 3
done
kubectl get pods -n lab-7a -o wide | tee ~/lab-evidence/7a/b3-web.txt
```

```bash
STORE_NODES="$(kubectl get pods -n lab-7a -l app=store \
  -o jsonpath='{range .items[?(@.spec.nodeName)]}{.spec.nodeName}{"\n"}{end}' | sort -u)"
WEB_NODES="$(kubectl get pods -n lab-7a -l app=web-store \
  -o jsonpath='{range .items[?(@.spec.nodeName)]}{.spec.nodeName}{"\n"}{end}' | sort -u)"
printf 'store:\n%s\nweb-store:\n%s\n' "$STORE_NODES" "$WEB_NODES"

test "$STORE_NODES" = "$WEB_NODES" \
  && echo 'PASS: web-store dat cung tap node voi store — podAffinity da keo dung cho'

RUN="$(kubectl get pods -n lab-7a -l app=web-store --field-selector=status.phase=Running \
  --no-headers | wc -l)"
PEND="$(kubectl get pods -n lab-7a -l app=web-store --field-selector=status.phase=Pending \
  --no-headers | wc -l)"
test "$RUN" -eq 2 && test "$PEND" -eq 1 \
  && echo 'PASS: anti-affinity voi chinh no van giu moi node dung mot web-store'
```

**Ý nghĩa:** một Pod có thể mang **cả hai** loại cùng lúc. `podAffinity` bảo "chỉ lên node đã có
Pod `app=store`", `podAntiAffinity` bảo "đừng có hai `web-store` trên cùng node". Kết quả là bố
cục mà bài mô tả: mỗi web server ở cùng chỗ với đúng một cache. Trên cluster lab chỉ có hai
node nên bảng ba cột của bài rút còn hai cột.

Ranh giới cần nhớ: `podAffinity` chỉ thỏa được khi **đã có** Pod khớp selector. Nếu bạn tạo
`web-store` trước `store`, cả ba replica sẽ nằm `Pending` — cơ chế chống deadlock mà bài mô tả
chỉ áp dụng cho Pod có affinity **với chính nhóm của nó**, không áp dụng cho affinity trỏ sang
nhóm khác.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B3.3. Đổi sang `preferred` thì replica thứ ba chạy được

```bash
cat > ~/lab-work/7a/b3-store-soft.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-soft
  namespace: lab-7a
spec:
  replicas: 3
  selector:
    matchLabels:
      app: store-soft
  template:
    metadata:
      labels:
        app: store-soft
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - store-soft
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: app
        image: busybox:1.37
        command: ["sh", "-c", "sleep 3600"]
        resources:
          requests:
            cpu: "20m"
            memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b3-store-soft.yaml
kubectl rollout status deploy/store-soft -n lab-7a --timeout=300s
kubectl get pods -n lab-7a -l app=store-soft -o wide | tee -a ~/lab-evidence/7a/b3-web.txt
```

```bash
RUN="$(kubectl get pods -n lab-7a -l app=store-soft --field-selector=status.phase=Running \
  --no-headers | wc -l)"
DISTINCT="$(kubectl get pods -n lab-7a -l app=store-soft \
  -o jsonpath='{range .items[?(@.spec.nodeName)]}{.spec.nodeName}{"\n"}{end}' | sort -u | wc -l)"
echo "running=$RUN distinct_nodes=$DISTINCT"

test "$RUN" -eq 3 \
  && echo 'PASS: preferred khong chan lap lich — ca ba replica deu chay'
test "$DISTINCT" -eq 2 \
  && echo 'PASS: nhung scheduler van trai ra ca hai node truoc khi don'
```

**Ý nghĩa:** cùng một quy tắc, đổi một từ, đổi hẳn hạng: `required` là bộ lọc, `preferred` là
điểm số. Đây cũng là chỗ bài [140](../140-topology-spread-constraints-vi.md) bắt đầu: nó chỉ ra
`podAntiAffinity` chỉ cho bạn chọn giữa "mỗi miền tối đa một Pod" và "không ép được gì cả", còn
ràng buộc phân bố theo topology cho bạn chỉnh mức lệch được phép bằng một con số. B5 làm đúng
việc đó.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

```bash
kubectl delete deployment store web-store store-soft -n lab-7a --wait=true --timeout=180s
for i in $(seq 1 40); do
  LEFT="$(kubectl get pods -n lab-7a --no-headers 2>/dev/null | grep -vc '^direct ' || true)"
  test "${LEFT:-0}" -eq 0 && break
  sleep 3
done
kubectl get pods -n lab-7a -o wide
```

**PASS:** trong namespace `lab-7a` chỉ còn Pod `direct` của B1.3.

---

## B4. Taint và toleration — mặt đối ngẫu của affinity

Affinity là thuộc tính của **Pod** dùng để thu hút Pod về một tập node. Taint là thuộc tính của
**node** dùng để đẩy Pod ra. Bạn đã sống chung với cơ chế này từ Lab 1a mà chưa gọi tên: mọi
Pod thường của bạn luôn rơi xuống hai worker, và B1.2 vừa in ra đúng lý do.

### B4.1. Đọc taint thật của control plane

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints' \
  | tee ~/lab-evidence/7a/b4-taints-truoc.txt

CP_KEY="$(kubectl get node lab-k8s-master -o jsonpath='{.spec.taints[0].key}')"
CP_EFFECT="$(kubectl get node lab-k8s-master -o jsonpath='{.spec.taints[0].effect}')"
CP_VALUE="$(kubectl get node lab-k8s-master -o jsonpath='{.spec.taints[0].value}')"
echo "master taint: key='${CP_KEY}' value='${CP_VALUE}' effect='${CP_EFFECT}'"
```

```bash
test "$CP_KEY" = 'node-role.kubernetes.io/control-plane' && test "$CP_EFFECT" = 'NoSchedule' \
  && echo 'PASS: control plane bi taint NoSchedule — day la vi du co san cua bai 139'
test -z "$CP_VALUE" \
  && echo 'PASS: taint nay khong co value, nen toleration phai dung operator Exists'
```

**Ý nghĩa:** hai gate trên là dữ kiện, không phải nghi lễ. `value` **rỗng** nghĩa là một
toleration `Equal` với `value: ""` mới khớp, còn cách sạch sẽ là `operator: Exists`. Và
`NoSchedule` giải thích chính xác vì sao `kubectl run` không bao giờ đưa Pod lên master: Pod
mặc định không có toleration nào nên bị loại **ngay ở bước lọc**.

**Không gỡ taint này.** Gỡ nó là đổi hành vi lập lịch của mọi lab sau.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B4.2. Tự taint `lab-k8s-worker2` và chứng minh Pod thường không lên

```bash
kubectl taint nodes lab-k8s-worker2 lab7a=demo:NoSchedule
kubectl get node lab-k8s-worker2 -o jsonpath='{.spec.taints}'; echo

cat > ~/lab-work/7a/b4-notol.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: tol-none
  namespace: lab-7a
  labels:
    app: b4
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b4-notol.yaml
for i in $(seq 1 30); do
  R="$(kubectl get pod tol-none -n lab-7a \
    -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
  test "$R" = 'Unschedulable' && break
  sleep 2
done
kubectl get pod tol-none -n lab-7a \
  -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].message}' \
  | tee ~/lab-evidence/7a/b4-notol.txt; echo
```

```bash
R="$(kubectl get pod tol-none -n lab-7a \
  -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
test "$R" = 'Unschedulable' \
  && echo 'PASS: Pod ghim vao worker2 khong len duoc vi taint'
grep -q 'untolerated taint' ~/lab-evidence/7a/b4-notol.txt \
  && echo 'PASS: thong diep loc noi thang ly do la taint khong duoc dung thu'
```

**Ý nghĩa:** `nodeSelector` chỉ ra node duy nhất được phép, taint gạt đúng node đó khỏi tập khả
thi, nên giao rỗng. Đây là điểm khác nhau căn bản với bài 138: label **thu hút**, taint **đẩy
ra**, và hai lực đó độc lập nhau.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B4.3. Hai cách viết toleration khớp cùng một taint

```bash
cat > ~/lab-work/7a/b4-tol.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: tol-equal
  namespace: lab-7a
  labels:
    app: b4
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  tolerations:
  - key: "lab7a"
    operator: "Equal"
    value: "demo"
    effect: "NoSchedule"
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: tol-exists
  namespace: lab-7a
  labels:
    app: b4
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  tolerations:
  - key: "lab7a"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b4-tol.yaml
kubectl wait --for=condition=Ready pod/tol-equal pod/tol-exists -n lab-7a --timeout=180s
kubectl get pods -n lab-7a -l app=b4 -o wide | tee -a ~/lab-evidence/7a/b4-notol.txt
```

```bash
E1="$(kubectl get pod tol-equal  -n lab-7a -o jsonpath='{.spec.nodeName}')"
E2="$(kubectl get pod tol-exists -n lab-7a -o jsonpath='{.spec.nodeName}')"
echo "tol-equal -> $E1 | tol-exists -> $E2"

test "$E1" = 'lab-k8s-worker2' && test "$E2" = 'lab-k8s-worker2' \
  && echo 'PASS: ca hai cach viet toleration deu khop taint va Pod len duoc worker2'
```

**Ý nghĩa:** một toleration khớp một taint khi **key giống nhau, effect giống nhau**, và hoặc
`operator: Exists` (không được ghi `value`), hoặc `operator: Equal` với `value` bằng nhau. Mặc
định của `operator` là `Equal`.

Nhớ ranh giới ở B4.1: toleration **cho phép** lập lịch chứ **không đảm bảo**. Hai Pod trên lên
đúng worker2 là nhờ `nodeSelector`, không phải nhờ toleration. Nếu bỏ `nodeSelector` đi, bộ lập
lịch hoàn toàn có thể chọn worker1 — toleration chỉ gỡ rào, không phải lực hút.

**PASS:** dòng `PASS: ca hai cach viet toleration…` xuất hiện.

### B4.4. Nhiều taint được xử lý như một bộ lọc

```bash
kubectl taint nodes lab-k8s-worker2 lab7a-extra=demo:NoSchedule
kubectl get node lab-k8s-worker2 -o jsonpath='{.spec.taints}'; echo

for i in $(seq 1 10); do sleep 3; done
kubectl get pods -n lab-7a -l app=b4 -o wide
```

`tol-partial` dưới đây dung thứ **đúng một** trong hai taint:

```bash
cat > ~/lab-work/7a/b4-partial.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: tol-partial
  namespace: lab-7a
  labels:
    app: b4
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  tolerations:
  - key: "lab7a"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b4-partial.yaml
for i in $(seq 1 30); do
  R="$(kubectl get pod tol-partial -n lab-7a \
    -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
  test "$R" = 'Unschedulable' && break
  sleep 2
done
kubectl get pods -n lab-7a -l app=b4 -o wide | tee -a ~/lab-evidence/7a/b4-notol.txt
```

```bash
P1="$(kubectl get pod tol-equal  -n lab-7a -o jsonpath='{.status.phase}')"
P2="$(kubectl get pod tol-exists -n lab-7a -o jsonpath='{.status.phase}')"
P3="$(kubectl get pod tol-partial -n lab-7a \
  -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
echo "tol-equal=$P1 tol-exists=$P2 tol-partial.reason=$P3"

test "$P1" = 'Running' && test "$P2" = 'Running' \
  && echo 'PASS: taint NoSchedule moi KHONG duoi hai Pod dang chay'
test "$P3" = 'Unschedulable' \
  && echo 'PASS: chi can mot taint khong duoc dung thu la Pod moi khong len duoc'
```

**Ý nghĩa:** hai kết luận đối lập trên cùng một node, cùng một thời điểm.

Một: `NoSchedule` **chỉ chặn Pod mới**; Pod hiện đang chạy trên node **không** bị thu hồi. Hai
Pod của B4.3 sống sót dù chúng không hề dung thứ `lab7a-extra`.

Hai: cách Kubernetes xử lý nhiều taint giống một **bộ lọc** — bắt đầu với tất cả taint của
node, bỏ qua những taint được Pod dung thứ, những taint còn lại vẫn phát huy effect. `tol-partial`
dung thứ `lab7a` nhưng không dung thứ `lab7a-extra`, và một taint `NoSchedule` không được bỏ
qua là đủ để chặn.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B4.5. `nodeName` lên được cả node đang mang hai taint `NoSchedule`

```bash
cat > ~/lab-work/7a/b4-direct2.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: direct2
  namespace: lab-7a
  labels:
    app: b4
spec:
  nodeName: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b4-direct2.yaml
kubectl wait --for=condition=Ready pod/direct2 -n lab-7a --timeout=180s
kubectl get pod direct direct2 -n lab-7a -o wide
```

```bash
D2_NODE="$(kubectl get pod direct2 -n lab-7a -o jsonpath='{.spec.nodeName}')"
D2_TOL="$(kubectl get pod direct2 -n lab-7a \
  -o jsonpath='{.spec.tolerations[?(@.key=="lab7a")].key}')"
echo "direct2 -> $D2_NODE | toleration cho lab7a: '${D2_TOL}'"

test "$D2_NODE" = 'lab-k8s-worker2' && test -z "$D2_TOL" \
  && echo 'PASS: Pod dat bang nodeName len duoc node co taint NoSchedule ma khong can toleration'
```

**Ý nghĩa:** bài 139 nói thẳng: nếu bạn chỉ định thủ công `.spec.nodeName`, hành động đó **bỏ
qua bộ lập lịch**; Pod được gắn vào node bạn chọn ngay cả khi node đó có taint `NoSchedule`.
Taint sống ở bước lọc, mà Pod này không đi qua bước lọc.

Nhưng câu tiếp theo của bài mới là phần đắt giá, và B4.6 kiểm chứng nó: **nếu node đó cũng có
taint `NoExecute`, kubelet sẽ đẩy Pod ra** — vì `NoExecute` không sống ở bước lọc.

**PASS:** dòng `PASS: Pod dat bang nodeName…` xuất hiện.

### B4.6. `NoExecute` đuổi cả Pod đang chạy, `tolerationSeconds` quyết định muộn bao lâu

> **Fault injection — chỉ chạy trên `lab-k8s-worker2`.** Taint `NoExecute` dưới đây đuổi **mọi**
> Pod trên worker2 không dung thứ nó, kể cả Pod hạ tầng do Lab 5b và Lab 6a cài. Đó là hành vi
> **đúng** của `NoExecute`, không phải hỏng hóc: các Pod đó thuộc Deployment nên control plane
> tạo bản thay thế trên `lab-k8s-worker1`, và chúng **không tự quay lại worker2** sau khi bạn gỡ
> taint. Gate cuối của lab chỉ yêu cầu chúng `Running`, không yêu cầu đúng node cũ. Pod của
> DaemonSet (CNI, kube-proxy) khai toleration `operator: Exists` nên không bị ảnh hưởng — bước
> dưới chụp lại danh sách trước và sau để bạn tự đối chiếu.

```bash
kubectl get pods -A -o wide --field-selector spec.nodeName=lab-k8s-worker2 \
  | tee ~/lab-evidence/7a/b4-worker2-truoc.txt
```

Ba Pod dưới đây khác nhau **đúng một chỗ**: phần toleration cho effect `NoExecute`.

```bash
cat > ~/lab-work/7a/b4-noexec.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nx-none
  namespace: lab-7a
  labels:
    app: b4nx
spec:
  terminationGracePeriodSeconds: 5
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  tolerations:
  - key: "lab7a"
    operator: "Exists"
    effect: "NoSchedule"
  - key: "lab7a-extra"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: nx-secs
  namespace: lab-7a
  labels:
    app: b4nx
spec:
  terminationGracePeriodSeconds: 5
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  tolerations:
  - key: "lab7a"
    operator: "Exists"
    effect: "NoSchedule"
  - key: "lab7a-extra"
    operator: "Exists"
    effect: "NoSchedule"
  - key: "lab7a"
    operator: "Equal"
    value: "demo"
    effect: "NoExecute"
    tolerationSeconds: 120
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: nx-forever
  namespace: lab-7a
  labels:
    app: b4nx
spec:
  terminationGracePeriodSeconds: 5
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  tolerations:
  - key: "lab7a"
    operator: "Exists"
    effect: "NoSchedule"
  - key: "lab7a-extra"
    operator: "Exists"
    effect: "NoSchedule"
  - key: "lab7a"
    operator: "Equal"
    value: "demo"
    effect: "NoExecute"
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b4-noexec.yaml
kubectl wait --for=condition=Ready pod/nx-none pod/nx-secs pod/nx-forever \
  -n lab-7a --timeout=180s
kubectl get pods -n lab-7a -o wide
```

Đặt taint `NoExecute` rồi quan sát **ngay**:

```bash
kubectl taint nodes lab-k8s-worker2 lab7a=demo:NoExecute

WINDOW=0
for i in $(seq 1 40); do
  NONE_LEFT="$(kubectl get pod nx-none -n lab-7a --no-headers 2>/dev/null | wc -l)"
  SECS_LEFT="$(kubectl get pod nx-secs -n lab-7a --no-headers 2>/dev/null | wc -l)"
  if [ "$NONE_LEFT" -eq 0 ] && [ "$SECS_LEFT" -eq 1 ]; then WINDOW=1; break; fi
  if [ "$NONE_LEFT" -eq 0 ] && [ "$SECS_LEFT" -eq 0 ]; then break; fi
  sleep 2
done
echo "window quan sat duoc = $WINDOW"
kubectl get pods -n lab-7a -o wide | tee ~/lab-evidence/7a/b4-noexec.txt
```

```bash
test "$WINDOW" -eq 1 \
  && echo 'PASS: nx-none bi thu hoi ngay, trong khi nx-secs con dung tolerationSeconds cua no'
```

Chờ hết `tolerationSeconds` của `nx-secs`:

```bash
for i in $(seq 1 60); do
  SECS_LEFT="$(kubectl get pod nx-secs -n lab-7a --no-headers 2>/dev/null | wc -l)"
  test "$SECS_LEFT" -eq 0 && break
  sleep 5
done
kubectl get pods -n lab-7a -o wide | tee -a ~/lab-evidence/7a/b4-noexec.txt
kubectl get pods -A -o wide --field-selector spec.nodeName=lab-k8s-worker2 \
  | tee ~/lab-evidence/7a/b4-worker2-sau.txt
```

```bash
NONE_LEFT="$(kubectl get pod nx-none    -n lab-7a --no-headers 2>/dev/null | wc -l)"
SECS_LEFT="$(kubectl get pod nx-secs    -n lab-7a --no-headers 2>/dev/null | wc -l)"
FOREVER="$(kubectl get pod nx-forever -n lab-7a -o jsonpath='{.status.phase}' 2>/dev/null)"
D1="$(kubectl get pod direct  -n lab-7a --no-headers 2>/dev/null | wc -l)"
D2="$(kubectl get pod direct2 -n lab-7a --no-headers 2>/dev/null | wc -l)"
echo "nx-none=$NONE_LEFT nx-secs=$SECS_LEFT nx-forever=$FOREVER direct=$D1 direct2=$D2"

test "$NONE_LEFT" -eq 0 && test "$SECS_LEFT" -eq 0 \
  && echo 'PASS: het tolerationSeconds thi Pod cung bi thu hoi'
test "$FOREVER" = 'Running' \
  && echo 'PASS: toleration NoExecute khong kem tolerationSeconds thi Pod o lai mai mai'
test "$D1" -eq 0 && test "$D2" -eq 0 \
  && echo 'PASS: ca hai Pod dat bang nodeName cung bi kubelet day ra — NoExecute khong song o buoc loc'
```

**Ý nghĩa:** ba số phận, ba dòng toleration khác nhau, đúng như bài 139 liệt kê:

- Pod **không dung thứ** taint bị thu hồi ngay lập tức.
- Pod dung thứ mà **có** `tolerationSeconds` ở lại đúng khoảng đó rồi bị node lifecycle
  controller thu hồi. Nếu bạn gỡ taint **trước** thời điểm đó, Pod sẽ không bị thu hồi.
- Pod dung thứ mà **không** chỉ định `tolerationSeconds` vẫn gắn với node mãi mãi.

`direct` và `direct2` là phần thưởng: chúng vào node bằng `nodeName`, tức bỏ qua bộ lập lịch —
nhưng `NoExecute` do **kubelet và taint-eviction-controller** thi hành chứ không phải bộ lập
lịch, nên không ai thoát được.

Thời điểm chính xác của mỗi lần thu hồi **phụ thuộc cấu hình** control plane, trong đó có cả cơ
chế giới hạn tốc độ thêm taint mà bài nhắc tới; vì vậy mọi bước chờ ở trên là vòng lặp theo
điều kiện chứ không phải một con số.

**PASS:** bốn dòng `PASS:` của mục B4.6 xuất hiện.

### B4.7. Gỡ hết taint và trả worker2 về nguyên trạng

```bash
kubectl taint nodes lab-k8s-worker2 lab7a=demo:NoExecute-
kubectl taint nodes lab-k8s-worker2 lab7a=demo:NoSchedule-
kubectl taint nodes lab-k8s-worker2 lab7a-extra=demo:NoSchedule-
kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints' \
  | tee ~/lab-evidence/7a/b4-taints-sau.txt

kubectl delete pod tol-equal tol-exists tol-partial tol-none nx-forever \
  -n lab-7a --ignore-not-found=true --wait=true --timeout=120s

for i in $(seq 1 60); do
  BAD="$(kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded \
    --no-headers 2>/dev/null | wc -l)"
  test "$BAD" -eq 0 && break
  sleep 5
done
kubectl get pods -A -o wide | tee -a ~/lab-evidence/7a/b4-worker2-sau.txt
```

```bash
W2_TAINTS="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.spec.taints}')"
BAD="$(kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded \
  --no-headers 2>/dev/null | wc -l)"
LEFT="$(kubectl get pods -n lab-7a --no-headers 2>/dev/null | wc -l)"
echo "worker2.taints='${W2_TAINTS}' | pod khong Running toan cluster=$BAD | pod con trong lab-7a=$LEFT"

test -z "$W2_TAINTS" \
  && echo 'PASS: worker2 sach taint'
test "$BAD" -eq 0 \
  && echo 'PASS: moi Pod trong cluster da tro lai Running'
test "$LEFT" -eq 0 \
  && echo 'PASS: namespace lab-7a trong truoc khi sang B5'
```

**PASS:** ba dòng `PASS:` của bước này xuất hiện. Nếu còn Pod hạ tầng chưa `Running`, chờ thêm
vài vòng: chúng đang được tạo lại trên `lab-k8s-worker1`.

---

## B5. Ràng buộc phân bố Pod theo topology

`podAntiAffinity` chỉ cho bạn hai lựa chọn: `required` là "mỗi miền tối đa một Pod", `preferred`
là "không ép được gì cả". Bài [140](../140-topology-spread-constraints-vi.md) đưa vào một con
số ở giữa: `maxSkew`.

Cluster lab không có label zone nào, nên mục này **tự tạo miền topology** bằng một label riêng
`lab7a-zone` — cách đọc mọi ví dụ `topologyKey: zone` của bài mà không phải tưởng tượng.

### B5.1. Dựng hai miền và một tình huống lệch sẵn

```bash
kubectl label nodes lab-k8s-worker1 lab7a-zone=a
kubectl label nodes lab-k8s-worker2 lab7a-zone=b
kubectl get nodes -L lab7a-zone | tee ~/lab-evidence/7a/b5-zones.txt

cat > ~/lab-work/7a/b5-seed.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: seed-1
  namespace: lab-7a
  labels:
    app: spread
spec:
  nodeSelector:
    lab7a-zone: a
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: seed-2
  namespace: lab-7a
  labels:
    app: spread
spec:
  nodeSelector:
    lab7a-zone: a
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b5-seed.yaml
kubectl wait --for=condition=Ready pod/seed-1 pod/seed-2 -n lab-7a --timeout=180s
kubectl get pods -n lab-7a -l app=spread -o wide
```

```bash
ZA="$(kubectl get pods -n lab-7a -l app=spread -o wide --no-headers \
  | grep -c 'lab-k8s-worker1' || true)"
ZB="$(kubectl get pods -n lab-7a -l app=spread -o wide --no-headers \
  | grep -c 'lab-k8s-worker2' || true)"
echo "zone a=$ZA zone b=$ZB"

test "$ZA" -eq 2 && test "$ZB" -eq 0 \
  && echo 'PASS: hai mien topology, phan bo lech san [2, 0]'
```

**Ý nghĩa:** một **miền** là một cặp `<key, value>` của label node; `topologyKey: lab7a-zone`
tạo ra đúng hai miền `a` và `b`. Hai Pod `seed-*` không mang ràng buộc phân bố nào — chúng chỉ
là các Pod hiện có mà Pod đến sau sẽ **đếm**, vì chúng khớp `labelSelector` mà B5.2 dùng.

**PASS:** dòng `PASS: hai mien topology…` xuất hiện.

### B5.2. `maxSkew: 1` đẩy Pod mới sang miền còn trống

```bash
cat > ~/lab-work/7a/b5-spread.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: spread-1
  namespace: lab-7a
  labels:
    app: spread
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: lab7a-zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: spread
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/7a/b5-spread.yaml
kubectl apply -f ~/lab-work/7a/b5-spread.yaml
kubectl wait --for=condition=Ready pod/spread-1 -n lab-7a --timeout=180s
kubectl get pods -n lab-7a -l app=spread -o wide | tee ~/lab-evidence/7a/b5-spread.txt
```

```bash
N="$(kubectl get pod spread-1 -n lab-7a -o jsonpath='{.spec.nodeName}')"
echo "spread-1 -> $N"

test "$N" = 'lab-k8s-worker2' \
  && echo 'PASS: dat vao mien a se lech 3-0=3 > maxSkew, nen scheduler chon mien b'
```

**Ý nghĩa:** phép tính đúng như bài mô tả. Trước khi đặt: miền `a` có 2 Pod khớp, miền `b` có 0,
**mức tối thiểu toàn cục** là 0. Nếu đặt vào `a` thì `a` thành 3 và độ lệch là `3 - 0 = 3`, vi
phạm `maxSkew: 1`. Đặt vào `b` thì `b` thành 1, mức tối thiểu toàn cục thành 1, độ lệch là
`2 - 1 = 1` — vừa đúng ngưỡng. Chỉ còn một chỗ hợp lệ.

**PASS:** dòng `PASS: dat vao mien a se lech…` xuất hiện.

### B5.3. `nodeAffinityPolicy` quyết định miền nào được đưa vào phép tính

Hai Pod dưới đây **giống hệt nhau** trừ một dòng: `nodeAffinityPolicy`.

```bash
cat > ~/lab-work/7a/b5-policy.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: spread-honor
  namespace: lab-7a
  labels:
    app: spread
spec:
  nodeSelector:
    lab7a-zone: a
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: lab7a-zone
    whenUnsatisfiable: DoNotSchedule
    nodeAffinityPolicy: Honor
    labelSelector:
      matchLabels:
        app: spread
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b5-policy.yaml
kubectl wait --for=condition=Ready pod/spread-honor -n lab-7a --timeout=180s

cat > ~/lab-work/7a/b5-policy-ignore.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: spread-ignore
  namespace: lab-7a
  labels:
    app: spread
spec:
  nodeSelector:
    lab7a-zone: a
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: lab7a-zone
    whenUnsatisfiable: DoNotSchedule
    nodeAffinityPolicy: Ignore
    labelSelector:
      matchLabels:
        app: spread
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b5-policy-ignore.yaml
for i in $(seq 1 30); do
  R="$(kubectl get pod spread-ignore -n lab-7a \
    -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
  test "$R" = 'Unschedulable' && break
  sleep 2
done
kubectl get pods -n lab-7a -l app=spread -o wide | tee -a ~/lab-evidence/7a/b5-spread.txt
```

```bash
HN="$(kubectl get pod spread-honor -n lab-7a -o jsonpath='{.spec.nodeName}')"
IR="$(kubectl get pod spread-ignore -n lab-7a \
  -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
echo "spread-honor -> $HN | spread-ignore.reason=$IR"

test "$HN" = 'lab-k8s-worker1' \
  && echo 'PASS: Honor chi dem mien khop nodeSelector, nen mien a la mien hop le duy nhat'
test "$IR" = 'Unschedulable' \
  && echo 'PASS: Ignore dem ca hai mien, do lech vuot maxSkew nen Pod nam Pending'
```

**Ý nghĩa:** đây là cặp thử phân biệt sạch nhất của bài 140.

Với `Honor` (**giá trị mặc định** khi bạn không ghi gì), chỉ node khớp `nodeSelector`/node
affinity mới được đưa vào tính toán: miền hợp lệ duy nhất là `a`, mức tối thiểu toàn cục lấy
trong chính miền đó, nên độ lệch luôn bằng 0 và Pod lên được.

Với `Ignore`, cả hai miền cùng được đếm: miền `b` chỉ có 1 Pod, đặt thêm vào `a` là lệch quá
`maxSkew`, mà `nodeSelector` lại cấm sang `b` — giao rỗng, Pod `Pending`. Đúng tình huống mà
bài gọi là *các ràng buộc phân bố xung đột nhau*.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B5.4. `ScheduleAnyway` không giữ Pod lại

```bash
sed -e 's/name: spread-ignore/name: spread-anyway/' \
    -e 's/whenUnsatisfiable: DoNotSchedule/whenUnsatisfiable: ScheduleAnyway/' \
    ~/lab-work/7a/b5-policy-ignore.yaml > ~/lab-work/7a/b5-anyway.yaml
grep -nE 'name: spread-anyway|whenUnsatisfiable' ~/lab-work/7a/b5-anyway.yaml

kubectl apply -f ~/lab-work/7a/b5-anyway.yaml
kubectl wait --for=condition=Ready pod/spread-anyway -n lab-7a --timeout=180s
kubectl get pods -n lab-7a -l app=spread -o wide | tee -a ~/lab-evidence/7a/b5-spread.txt
```

```bash
AN="$(kubectl get pod spread-anyway -n lab-7a -o jsonpath='{.spec.nodeName}')"
PH="$(kubectl get pod spread-anyway -n lab-7a -o jsonpath='{.status.phase}')"
echo "spread-anyway -> $AN phase=$PH"

test "$PH" = 'Running' && test "$AN" = 'lab-k8s-worker1' \
  && echo 'PASS: cung phep tinh lech do, ScheduleAnyway van lap lich Pod'
```

**Ý nghĩa:** `whenUnsatisfiable` không đổi phép tính, nó đổi **hậu quả** của phép tính:
`DoNotSchedule` (mặc định) biến ràng buộc thành bộ lọc, `ScheduleAnyway` biến nó thành điểm số —
scheduler vẫn ưu tiên miền ít Pod hơn, nhưng nếu không đi được thì vẫn đặt. Đây đúng là cặp
`required`/`preferred` của B2 và B3, mang tên khác.

**PASS:** dòng `PASS: cung phep tinh lech do…` xuất hiện.

### B5.5. Node thiếu `topologyKey` bị bỏ qua hoàn toàn

```bash
kubectl label nodes lab-k8s-worker2 lab7a-zone-
kubectl get nodes -L lab7a-zone | tee -a ~/lab-evidence/7a/b5-zones.txt

sed -e 's/name: spread-1/name: spread-orphan/' ~/lab-work/7a/b5-spread.yaml \
  > ~/lab-work/7a/b5-orphan.yaml
grep -n 'name: spread-orphan' ~/lab-work/7a/b5-orphan.yaml

kubectl apply -f ~/lab-work/7a/b5-orphan.yaml
kubectl wait --for=condition=Ready pod/spread-orphan -n lab-7a --timeout=180s
kubectl get pods -n lab-7a -l app=spread -o wide | tee -a ~/lab-evidence/7a/b5-spread.txt
```

```bash
OR_NODE="$(kubectl get pod spread-orphan -n lab-7a -o jsonpath='{.spec.nodeName}')"
W2_ZONE="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.metadata.labels.lab7a-zone}')"
echo "spread-orphan -> $OR_NODE | worker2.lab7a-zone='${W2_ZONE}'"

test -z "$W2_ZONE" && test "$OR_NODE" = 'lab-k8s-worker1' \
  && echo 'PASS: worker2 thieu topologyKey nen bi bo qua, Pod don ve worker1 du worker2 rong'
```

**Ý nghĩa:** đây là **quy ước ngầm định** nguy hiểm nhất của bài 140, và cũng là lỗi cấu hình
hay gặp nhất ngoài thực tế. Scheduler chỉ xem xét node có **đầy đủ tất cả** `topologyKey` được
nêu; node thiếu một khóa nào đó thì bị bỏ qua — kéo theo hai hệ quả: Pod trên node đó không
được tính vào `maxSkew`, và Pod mới **không có cơ hội** lên node đó. Một label gõ sai
(`lab7a-zon` thay vì `lab7a-zone`) cho ra đúng hiện tượng này, trong khi node vẫn `Ready` và
vẫn còn tài nguyên.

Hai quy ước còn lại của cùng mục, không dựng riêng bước nhưng phải nhớ: **chỉ Pod cùng
namespace** mới được đếm; và Pod có label **không khớp chính `labelSelector` của mình** trở
thành "pod ma" — nó lập lịch được nhưng không tự tính vào phép đo, nên ràng buộc chạy sai kỳ
vọng.

**PASS:** dòng `PASS: worker2 thieu topologyKey…` xuất hiện.

### B5.6. Dọn B5 và trả label về nguyên trạng

```bash
kubectl delete pod -n lab-7a -l app=spread --wait=true --timeout=180s
kubectl label nodes lab-k8s-worker1 lab7a-zone-
kubectl get nodes -L lab7a-zone
```

```bash
ZONED="$(kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.labels}{"\n"}{end}' \
  | grep -c 'lab7a-zone' || true)"
LEFT="$(kubectl get pods -n lab-7a --no-headers 2>/dev/null | wc -l)"
echo "node con label lab7a-zone=$ZONED | pod con lai=$LEFT"

test "$ZONED" -eq 0 && test "$LEFT" -eq 0 \
  && echo 'PASS: label mien topology da go, namespace lab-7a trong'
```

**PASS:** dòng `PASS: label mien topology da go…` xuất hiện.

---

## B6. PriorityClass và preemption

Từ đây trở đi cluster bắt đầu **chấm dứt Pod đang chạy**. Bài
[141](../141-pod-priority-preemption-vi.md) chia làm hai nửa: nửa đầu là cấu hình, nửa sau là
các **ranh giới** của preemption — và nửa sau mới là phần giải thích những hành vi trông như
lỗi.

### B6.1. Hai PriorityClass tích hợp sẵn và add-on đang dùng chúng

```bash
kubectl get priorityclass | tee ~/lab-evidence/7a/b6-priorityclass.txt
kubectl get priorityclass system-cluster-critical system-node-critical \
  -o custom-columns='NAME:.metadata.name,VALUE:.value,GLOBALDEFAULT:.globalDefault'

kubectl -n kube-system get deployment coredns \
  -o jsonpath='{.spec.template.spec.priorityClassName}'; echo
kubectl -n kube-system get daemonset kube-proxy \
  -o jsonpath='{.spec.template.spec.priorityClassName}'; echo
```

```bash
DNS_PC="$(kubectl -n kube-system get deployment coredns \
  -o jsonpath='{.spec.template.spec.priorityClassName}')"
PROXY_PC="$(kubectl -n kube-system get daemonset kube-proxy \
  -o jsonpath='{.spec.template.spec.priorityClassName}')"
NODE_VAL="$(kubectl get priorityclass system-node-critical -o jsonpath='{.value}')"
CLUSTER_VAL="$(kubectl get priorityclass system-cluster-critical -o jsonpath='{.value}')"
echo "coredns=$DNS_PC kube-proxy=$PROXY_PC | node-critical=$NODE_VAL cluster-critical=$CLUSTER_VAL"

test "$DNS_PC" = 'system-cluster-critical' \
  && echo 'PASS: CoreDNS duoc danh dau quan trong dung nhu bai 210 huong dan'
test "$PROXY_PC" = 'system-node-critical' \
  && echo 'PASS: kube-proxy dung muc uu tien cao nhat — system-node-critical'
test "$NODE_VAL" -gt "$CLUSTER_VAL" \
  && echo 'PASS: system-node-critical cao hon system-cluster-critical'
```

**Ý nghĩa:** bài [210](../210-guaranteed-scheduling-critical-addon-pods-vi.md) chỉ có một câu
hành động: để đánh dấu một Pod là quan trọng, đặt `priorityClassName` thành
`system-cluster-critical` hoặc `system-node-critical`, trong đó `system-node-critical` là mức
cao nhất. Cluster của bạn đã làm đúng như thế từ lúc `kubeadm init`, và bạn vừa đọc được nó.

Nhớ ranh giới của bài 210: đánh dấu quan trọng **không** ngăn hoàn toàn việc trục xuất, nó chỉ
ngăn Pod đó rơi vào tình trạng không khả dụng vĩnh viễn. Giá trị của cả hai class đều **vượt xa
1 tỷ** — trần dành cho PriorityClass do người dùng tạo — nên hai PriorityClass mà B6.2 tạo
không bao giờ chạm được tới chúng.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B6.2. Hai PriorityClass của lab và một node để làm chật

```bash
cat > ~/lab-work/7a/b6-priorityclass.yaml <<'EOF'
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: lab-7a-low
value: -10
globalDefault: false
description: "Lab 7a — workload co the bi lay cho. Xoa o B11."
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: lab-7a-high
value: 10
globalDefault: false
description: "Lab 7a — workload duoc uu tien lap lich. Xoa o B11."
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/7a/b6-priorityclass.yaml
kubectl apply -f ~/lab-work/7a/b6-priorityclass.yaml
kubectl label nodes lab-k8s-worker2 lab7a-preempt=yes

W2_ALLOC_M="$(cpu2m "$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.allocatable.cpu}')")"
W2_USED_M="$(node_used_cpu_m lab-k8s-worker2)"
W2_FREE_M=$(( W2_ALLOC_M - W2_USED_M ))
LOW_M=$(( W2_FREE_M * 45 / 100 ))
HIGH_M=$(( W2_FREE_M * 50 / 100 ))
echo "worker2 free=${W2_FREE_M}m | low=${LOW_M}m moi Pod | high=${HIGH_M}m" \
  | tee ~/lab-evidence/7a/b6-so-lieu.txt
```

```bash
LOW_DEFAULT="$(kubectl get priorityclass lab-7a-low  -o jsonpath='{.globalDefault}')"
HIGH_DEFAULT="$(kubectl get priorityclass lab-7a-high -o jsonpath='{.globalDefault}')"
echo "globalDefault: low=$LOW_DEFAULT high=$HIGH_DEFAULT"

test "$LOW_DEFAULT" = 'false' && test "$HIGH_DEFAULT" = 'false' \
  && echo 'PASS: khong PriorityClass nao cua lab la globalDefault'
test "$W2_FREE_M" -ge 400 && test "$HIGH_M" -gt $(( W2_FREE_M - 2 * LOW_M )) \
  && echo 'PASS: so lieu du de dung tinh huong — hai Pod low se lam high khong con cho'
```

**Ý nghĩa:** PriorityClass là object **không thuộc namespace**, ánh xạ tên sang một số nguyên;
giá trị càng cao thì độ ưu tiên càng cao. Chỉ được có **một** PriorityClass đặt `globalDefault:
true` trong cả hệ thống, và Pod không có `priorityClassName` thì độ ưu tiên bằng 0 — vì thế lab
cố tình cho `lab-7a-low` giá trị **âm**: nó phải thấp hơn cả Pod hạ tầng đang chạy trên worker2,
để nạn nhân của preemption chắc chắn là Pod của lab chứ không phải Pod của người khác.

Con số `LOW_M`/`HIGH_M` tính từ `allocatable` thật trừ đi phần đã bị đặt trước, đọc lại tại đây
chứ không dùng lại số của B0 — B4.6 đã làm một số Pod hạ tầng chuyển sang worker1.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B6.3. Làm chật `lab-k8s-worker2` bằng hai Pod ưu tiên thấp

```bash
cat > ~/lab-work/7a/b6-low.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: low-1
  namespace: lab-7a
  labels:
    app: low
spec:
  priorityClassName: lab-7a-low
  terminationGracePeriodSeconds: 60
  nodeSelector:
    lab7a-preempt: "yes"
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "trap '' TERM; while true; do sleep 1; done"]
    resources:
      requests:
        cpu: "${LOW_M}m"
        memory: "16Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: low-2
  namespace: lab-7a
  labels:
    app: low
spec:
  priorityClassName: lab-7a-low
  terminationGracePeriodSeconds: 60
  nodeSelector:
    lab7a-preempt: "yes"
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "trap '' TERM; while true; do sleep 1; done"]
    resources:
      requests:
        cpu: "${LOW_M}m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b6-low.yaml
kubectl wait --for=condition=Ready pod/low-1 pod/low-2 -n lab-7a --timeout=180s
kubectl get pods -n lab-7a -o wide
kubectl get pod low-1 -n lab-7a -o jsonpath='{.spec.priority}{"\n"}'
```

```bash
LOW_PRIO="$(kubectl get pod low-1 -n lab-7a -o jsonpath='{.spec.priority}')"
RUN="$(kubectl get pods -n lab-7a -l app=low --field-selector=status.phase=Running \
  --no-headers | wc -l)"
echo "spec.priority cua low-1 = $LOW_PRIO | low dang chay = $RUN"

test "$LOW_PRIO" = '-10' \
  && echo 'PASS: priority admission controller da phan giai ten class thanh so nguyen'
test "$RUN" -eq 2 \
  && echo 'PASS: hai Pod uu tien thap dang giu gan het cho trong cua worker2'
```

**Ý nghĩa:** bạn viết `priorityClassName`, còn **priority admission controller** điền số vào
`spec.priority`. Nếu tên class không tồn tại, Pod bị **từ chối** ngay tại admission — đó là lý
do B6.2 phải tạo class trước khi tạo Pod.

Hai Pod này dùng `trap '' TERM` và `terminationGracePeriodSeconds: 60` một cách có chủ đích:
chúng sẽ **không chết ngay** khi bị preempt, và chính khoảng trống đó cho bạn nhìn thấy
`nominatedNodeName` ở B6.4.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B6.4. Pod ưu tiên cao lấy chỗ — và chỉ lấy vừa đủ

```bash
cat > ~/lab-work/7a/b6-high.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: high-1
  namespace: lab-7a
  labels:
    app: high
spec:
  priorityClassName: lab-7a-high
  nodeSelector:
    lab7a-preempt: "yes"
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "${HIGH_M}m"
        memory: "16Mi"
EOF

kubectl apply -f ~/lab-work/7a/b6-high.yaml

NOM=''
for i in $(seq 1 40); do
  NOM="$(kubectl get pod high-1 -n lab-7a -o jsonpath='{.status.nominatedNodeName}')"
  test -n "$NOM" && break
  sleep 2
done
echo "high-1.status.nominatedNodeName = '${NOM}'"
kubectl get pods -n lab-7a -o wide | tee ~/lab-evidence/7a/b6-preempt.txt
kubectl get event -n lab-7a --field-selector reason=Preempted \
  -o custom-columns='OBJ:.involvedObject.name,SOURCE:.source.component,MSG:.message' \
  | tee -a ~/lab-evidence/7a/b6-preempt.txt
```

```bash
test -n "$NOM" \
  && echo "PASS: scheduler ghi nominatedNodeName=$NOM — no da chon node de preempt"
```

Chờ nạn nhân chết hẳn và `high-1` được lập lịch:

```bash
for i in $(seq 1 60); do
  HP="$(kubectl get pod high-1 -n lab-7a -o jsonpath='{.status.phase}')"
  test "$HP" = 'Running' && break
  sleep 5
done
kubectl get pods -n lab-7a -o wide | tee -a ~/lab-evidence/7a/b6-preempt.txt
```

```bash
HIGH_NODE="$(kubectl get pod high-1 -n lab-7a -o jsonpath='{.spec.nodeName}')"
HIGH_PHASE="$(kubectl get pod high-1 -n lab-7a -o jsonpath='{.status.phase}')"
LOW_LEFT="$(kubectl get pods -n lab-7a -l app=low --no-headers 2>/dev/null | wc -l)"
echo "high-1 phase=$HIGH_PHASE node=$HIGH_NODE | so Pod low con lai=$LOW_LEFT"

test "$HIGH_PHASE" = 'Running' && test "$HIGH_NODE" = 'lab-k8s-worker2' \
  && echo 'PASS: Pod uu tien cao da lay duoc cho tren worker2'
test "$LOW_LEFT" -eq 1 \
  && echo 'PASS: preemption chi loai bo dung mot Pod uu tien thap, khong loai het'
```

**Ý nghĩa:** đây là toàn bộ cơ chế của bài 141, gói trong ba mươi giây.

Khi `high-1` không tìm được node nào thỏa mãn, **logic preemption được kích hoạt**: scheduler
tìm một node mà việc loại bỏ một hoặc nhiều Pod có độ ưu tiên **thấp hơn** sẽ khiến Pod đang
chờ lập lịch được. Nó ghi tên node đó vào `nominatedNodeName` — trường này vừa giúp scheduler
theo dõi tài nguyên nó đang giữ chỗ, vừa là chỗ **bạn** nhìn thấy preemption đang diễn ra.

Ghi chú của bài nói rõ: **preemption không nhất thiết loại bỏ tất cả** Pod ưu tiên thấp hơn.
Bỏ một Pod `low` là `high-1` đã vừa chỗ, nên Pod `low` còn lại sống sót — và các Pod hạ tầng
trên worker2 (độ ưu tiên 0, cao hơn `-10`) thậm chí không được xét làm nạn nhân.

Hai điều **không** được kết luận từ bước này. Một, `nominatedNodeName` không phải cam kết:
scheduler luôn thử node được đề cử trước, nhưng nếu một node khác trống ra trong lúc chờ nạn
nhân chấm dứt thì nó có thể đổi ý. Hai, khoảng trễ giữa lúc preempt và lúc `high-1` chạy
**phụ thuộc cấu hình** — cụ thể là `terminationGracePeriodSeconds` của nạn nhân, ở đây do chính
bạn đặt là 60.

**PASS:** ba dòng `PASS:` của mục B6.4 xuất hiện.

### B6.5. PodDisruptionBudget được tôn trọng ở mức nỗ lực tốt nhất, không phải bảo đảm

Dựng lại đúng tình huống của B6.4, lần này có PDB che chở các Pod `low`:

```bash
kubectl delete pod high-1 -n lab-7a --wait=true --timeout=120s
kubectl delete pod -n lab-7a -l app=low --wait=true --timeout=180s
kubectl apply -f ~/lab-work/7a/b6-low.yaml
kubectl wait --for=condition=Ready pod/low-1 pod/low-2 -n lab-7a --timeout=180s

kubectl apply -f - <<'EOF'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: low-pdb
  namespace: lab-7a
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: low
EOF

for i in $(seq 1 24); do
  ALLOWED="$(kubectl get pdb low-pdb -n lab-7a -o jsonpath='{.status.disruptionsAllowed}' 2>/dev/null)"
  HEALTHY="$(kubectl get pdb low-pdb -n lab-7a -o jsonpath='{.status.currentHealthy}' 2>/dev/null)"
  test "${HEALTHY:-0}" -eq 2 && break
  sleep 5
done
kubectl get pdb low-pdb -n lab-7a | tee ~/lab-evidence/7a/b6-pdb.txt
```

```bash
HEALTHY="$(kubectl get pdb low-pdb -n lab-7a -o jsonpath='{.status.currentHealthy}')"
ALLOWED="$(kubectl get pdb low-pdb -n lab-7a -o jsonpath='{.status.disruptionsAllowed}')"
echo "currentHealthy=$HEALTHY disruptionsAllowed=$ALLOWED"

test "$HEALTHY" -eq 2 && test "$ALLOWED" -eq 0 \
  && echo 'PASS: PDB dang khong cho phep bat ky gian doan tu nguyen nao'
```

Bây giờ thả `high-1` vào lần nữa:

```bash
kubectl apply -f ~/lab-work/7a/b6-high.yaml
for i in $(seq 1 60); do
  HP="$(kubectl get pod high-1 -n lab-7a -o jsonpath='{.status.phase}')"
  test "$HP" = 'Running' && break
  sleep 5
done
kubectl get pods -n lab-7a -o wide | tee -a ~/lab-evidence/7a/b6-pdb.txt
kubectl get pdb low-pdb -n lab-7a | tee -a ~/lab-evidence/7a/b6-pdb.txt
```

```bash
HIGH_PHASE="$(kubectl get pod high-1 -n lab-7a -o jsonpath='{.status.phase}')"
LOW_LEFT="$(kubectl get pods -n lab-7a -l app=low --no-headers 2>/dev/null | wc -l)"
HEALTHY="$(kubectl get pdb low-pdb -n lab-7a -o jsonpath='{.status.currentHealthy}')"
echo "high-1=$HIGH_PHASE | low con lai=$LOW_LEFT | currentHealthy=$HEALTHY"

test "$HIGH_PHASE" = 'Running' && test "$LOW_LEFT" -eq 1 \
  && echo 'PASS: preemption van xay ra du PDB khong cho phep gian doan nao'
test "$HEALTHY" -lt 2 \
  && echo 'PASS: ngan sach gian doan da bi thung — PDB khong phai hang rao truoc preemption'
```

**Ý nghĩa:** đối chiếu thẳng với [B7 của Lab
3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md#b7-poddisruptionbudget-trên-pod-trần), nơi cùng một
PDB **chặn đứng** Eviction API. Ở đây nó không chặn được gì.

Bài 141 nói rõ vì sao: Kubernetes hỗ trợ PDB khi preempt, nhưng việc tôn trọng PDB **chỉ ở mức
nỗ lực tốt nhất**. Bộ lập lịch cố tìm các nạn nhân mà PDB của chúng không bị vi phạm; không tìm
thấy thì preemption **vẫn xảy ra**. Đây là ranh giới quan trọng nhất cần nhớ về PDB: nó bảo vệ
trước tự động hóa đi qua tầng API (B7), không bảo vệ trước quyết định của bộ lập lịch, và cũng
không bảo vệ trước eviction của kubelet (B8).

**PASS:** ba dòng `PASS:` của mục B6.5 xuất hiện.

### B6.6. Dọn workload của B6

```bash
kubectl delete pdb low-pdb -n lab-7a --ignore-not-found=true
kubectl delete pod high-1 -n lab-7a --ignore-not-found=true --wait=true --timeout=120s
kubectl delete pod -n lab-7a -l app=low --wait=true --timeout=180s
kubectl get pods -n lab-7a
```

Hai PriorityClass và label `lab7a-preempt` **giữ lại tới B11** — B11 là mục chịu trách nhiệm
xóa mọi thứ nằm ngoài namespace, và gate cuối phải chứng minh chúng biến mất.

**PASS:** `kubectl get pods -n lab-7a` trả `No resources found in lab-7a namespace.`

---

## B7. Eviction khởi phát qua API

Cùng một chữ "eviction", hai cơ chế đối lập nhau ở gần như mọi điểm. Bài
[143](../143-api-eviction-vi.md) mô tả cái đi qua **tầng API**: bạn tạo một object `Eviction`,
API server chấm dứt Pod, và toàn bộ chính sách bạn đã cấu hình được tôn trọng. Đây là cơ chế
nằm dưới `kubectl drain` — lệnh của [giai đoạn
16](../00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node), nên lab gọi thẳng subresource thay vì
dùng lệnh đó.

### B7.1. Một Pod đứng sau Service, một Deployment được PDB khóa chặt

```bash
cat > ~/lab-work/7a/b7-setup.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: evict-solo
  namespace: lab-7a
  labels:
    app: evict-solo
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "trap '' TERM; while true; do sleep 1; done"]
    ports:
    - containerPort: 8080
    resources:
      requests:
        cpu: "20m"
        memory: "16Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: evict-solo
  namespace: lab-7a
spec:
  selector:
    app: evict-solo
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: evict-web
  namespace: lab-7a
spec:
  replicas: 3
  selector:
    matchLabels:
      app: evict-web
  template:
    metadata:
      labels:
        app: evict-web
    spec:
      containers:
      - name: app
        image: busybox:1.37
        command: ["sh", "-c", "sleep 3600"]
        resources:
          requests:
            cpu: "20m"
            memory: "16Mi"
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: evict-web
  namespace: lab-7a
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: evict-web
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/7a/b7-setup.yaml
kubectl apply -f ~/lab-work/7a/b7-setup.yaml
kubectl wait --for=condition=Ready pod/evict-solo -n lab-7a --timeout=180s
kubectl rollout status deploy/evict-web -n lab-7a --timeout=300s

for i in $(seq 1 24); do
  H="$(kubectl get pdb evict-web -n lab-7a -o jsonpath='{.status.currentHealthy}' 2>/dev/null)"
  test "${H:-0}" -eq 3 && break
  sleep 5
done
kubectl get pdb -n lab-7a | tee ~/lab-evidence/7a/b7-pdb.txt
kubectl get endpointslice -n lab-7a -l kubernetes.io/service-name=evict-solo \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{" ready="}{.conditions.ready}{" terminating="}{.conditions.terminating}{"\n"}{end}' \
  | tee ~/lab-evidence/7a/b7-endpoints.txt
```

```bash
SOLO_IP="$(kubectl get pod evict-solo -n lab-7a -o jsonpath='{.status.podIP}')"
ALLOWED="$(kubectl get pdb evict-web -n lab-7a -o jsonpath='{.status.disruptionsAllowed}')"
SOLO_PDB="$(kubectl get pdb -n lab-7a -o jsonpath='{.items[*].metadata.name}')"
echo "evict-solo IP=$SOLO_IP | evict-web disruptionsAllowed=$ALLOWED | PDB co trong ns: $SOLO_PDB"

grep -q "^${SOLO_IP} ready=true" ~/lab-evidence/7a/b7-endpoints.txt \
  && echo 'PASS: evict-solo dang la endpoint hop le cua Service'
test "$ALLOWED" -eq 0 \
  && echo 'PASS: PDB cua evict-web dang khoa het ngan sach gian doan'
```

**Ý nghĩa:** hai đối tượng thử nghiệm cho hai mã trả về. `evict-solo` **không thuộc workload nào
có PDB** — bài nói API server khi đó **luôn** trả `200 OK`. `evict-web` có PDB `minAvailable: 3`
trên đúng 3 replica, nên không bao giờ có thời điểm nào evict mà không vi phạm budget.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B7.2. `200 OK` — và sáu bước xóa Pod bắt đầu chạy

```bash
cat > ~/lab-work/7a/b7-evict-solo.json <<'EOF'
{"apiVersion": "policy/v1", "kind": "Eviction", "metadata": {"name": "evict-solo", "namespace": "lab-7a"}}
EOF

kubectl create --raw /api/v1/namespaces/lab-7a/pods/evict-solo/eviction \
  -f ~/lab-work/7a/b7-evict-solo.json \
  | tee ~/lab-evidence/7a/b7-evict-solo.txt; echo

kubectl get pod evict-solo -n lab-7a -o wide
kubectl get pod evict-solo -n lab-7a \
  -o custom-columns='NAME:.metadata.name,DELTS:.metadata.deletionTimestamp,GRACE:.metadata.deletionGracePeriodSeconds' \
  | tee -a ~/lab-evidence/7a/b7-evict-solo.txt
kubectl get endpointslice -n lab-7a -l kubernetes.io/service-name=evict-solo \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{" ready="}{.conditions.ready}{" terminating="}{.conditions.terminating}{"\n"}{end}' \
  | tee ~/lab-evidence/7a/b7-endpoints-sau.txt
```

```bash
STILL="$(kubectl get pod evict-solo -n lab-7a --no-headers 2>/dev/null | wc -l)"
DELTS="$(kubectl get pod evict-solo -n lab-7a -o jsonpath='{.metadata.deletionTimestamp}' 2>/dev/null)"
GRACE="$(kubectl get pod evict-solo -n lab-7a -o jsonpath='{.metadata.deletionGracePeriodSeconds}' 2>/dev/null)"
STILL_READY="$(grep -c "^${SOLO_IP} ready=true" ~/lab-evidence/7a/b7-endpoints-sau.txt || true)"
echo "pod con ton tai=$STILL deletionTimestamp='${DELTS}' grace=$GRACE | endpoint con ready=$STILL_READY"

test "$STILL" -eq 1 && test -n "$DELTS" \
  && echo 'PASS: buoc 1 — API server danh dau xoa, Pod chua chet ngay'
test "$GRACE" = '60' \
  && echo 'PASS: eviction qua API TON TRONG terminationGracePeriodSeconds cua Pod'
test "$STILL_READY" -eq 0 \
  && echo 'PASS: buoc 3 — Pod da bi go khoi EndpointSlice trong khi container van con song'
```

Chờ Pod biến mất hẳn rồi đọc lại:

```bash
for i in $(seq 1 40); do
  S="$(kubectl get pod evict-solo -n lab-7a --no-headers 2>/dev/null | wc -l)"
  test "$S" -eq 0 && break
  sleep 5
done
kubectl get pods -n lab-7a -o wide | tee -a ~/lab-evidence/7a/b7-evict-solo.txt
```

```bash
S="$(kubectl get pod evict-solo -n lab-7a --no-headers 2>/dev/null | wc -l)"
test "$S" -eq 0 \
  && echo 'PASS: het grace period, kubelet cuong buc cham dut va API server xoa object Pod'
```

**Ý nghĩa:** ba dòng `PASS:` đầu là ba mốc trong **trình tự sáu bước** mà bài 143 liệt kê, và
chúng xảy ra đúng thứ tự đó:

1. Tài nguyên `Pod` được cập nhật với **deletion timestamp** và grace period đã cấu hình.
2. kubelet thấy Pod được đánh dấu và bắt đầu tắt Pod một cách êm thấm.
3. **Trong khi kubelet đang tắt Pod**, control plane gỡ Pod khỏi các object EndpointSlice — nên
   controller không còn coi nó là endpoint hợp lệ. Đây là thứ tự cứu bạn khỏi mất request: Pod
   ngừng **nhận** request mới trước khi nó ngừng **xử lý** request đang dở.
4. Hết grace period, kubelet cưỡng bức chấm dứt Pod.
5. kubelet báo API server gỡ tài nguyên `Pod`.
6. API server xóa tài nguyên `Pod`.

Bạn nhìn thấy được bước 3 chỉ vì `evict-solo` cố tình ngó lơ `SIGTERM` và có grace period 60
giây. Với một Pod thoát ngay, cả sáu bước trôi qua trong chớp mắt.

Tùy phiên bản và cấu hình, endpoint của Pod đang chấm dứt có thể **biến mất khỏi EndpointSlice**
hoặc **ở lại với `ready=false`, `terminating=true`**; cả hai đều thỏa cùng một kết luận, nên
gate ở trên kiểm tra "không còn `ready=true`" chứ không kiểm tra sự tồn tại của dòng.

**PASS:** bốn dòng `PASS:` của mục B7.2 xuất hiện.

### B7.3. `429 Too Many Requests` — PDB từ chối

```bash
VICTIM="$(kubectl get pods -n lab-7a -l app=evict-web -o jsonpath='{.items[0].metadata.name}')"
echo "se thu evict: $VICTIM"

cat > ~/lab-work/7a/b7-evict-web.json <<EOF
{"apiVersion": "policy/v1", "kind": "Eviction", "metadata": {"name": "${VICTIM}", "namespace": "lab-7a"}}
EOF

if kubectl create --raw "/api/v1/namespaces/lab-7a/pods/${VICTIM}/eviction" \
     -f ~/lab-work/7a/b7-evict-web.json > ~/lab-evidence/7a/b7-evict-web.txt 2>&1; then
  echo 'FAIL: PDB khong chan duoc eviction'
else
  cat ~/lab-evidence/7a/b7-evict-web.txt
  echo 'PASS: Eviction API bi tu choi'
fi
```

```bash
grep -qi 'disruption budget' ~/lab-evidence/7a/b7-evict-web.txt \
  && echo 'PASS: ly do tu choi dung la PodDisruptionBudget'
STILL="$(kubectl get pod "$VICTIM" -n lab-7a --no-headers 2>/dev/null | wc -l)"
test "$STILL" -eq 1 \
  && echo 'PASS: Pod van con nguyen vi yeu cau bi tu choi tu tang API'
```

**Ý nghĩa:** ba mã trả về của bài 143, giờ bạn đã gặp hai:

- `200 OK` — eviction được cho phép, subresource `Eviction` được tạo, Pod bị xóa như một request
  `DELETE`. Đây là B7.2.
- `429 Too Many Requests` — eviction **hiện** không được phép do PDB đã cấu hình; thử lại sau
  thì có thể được. Cũng có thể là cơ chế giới hạn tốc độ của API.
- `500 Internal Server Error` — cấu hình sai, ví dụ **nhiều PDB cùng tham chiếu một Pod**. Lab
  không dựng tình huống này vì nó là lỗi cấu hình, không phải hành vi cần học thuộc; nhưng phải
  nhớ nó mang nghĩa **khác hẳn** `429`: `429` là "chờ đi", `500` là "bạn cấu hình sai".

Đây cũng chính là tình huống *eviction bị kẹt*: với `minAvailable: 3` trên 3 replica, Eviction
API sẽ trả `429` mãi mãi. Lối thoát mà bài đưa ra là **hủy hoặc tạm dừng thao tác tự động**, điều
tra, rồi hoặc sửa PDB, hoặc xóa thẳng Pod khỏi control plane thay vì dùng Eviction API.

**PASS:** ba dòng `PASS:` của mục B7.3 xuất hiện.

### B7.4. Dọn B7

```bash
kubectl delete -f ~/lab-work/7a/b7-setup.yaml --ignore-not-found=true \
  --wait=true --timeout=180s
kubectl get all -n lab-7a
kubectl get pdb -n lab-7a
```

```bash
LEFT="$(kubectl get pods -n lab-7a --no-headers 2>/dev/null | wc -l)"
PDBS="$(kubectl get pdb -n lab-7a --no-headers 2>/dev/null | wc -l)"
test "$LEFT" -eq 0 && test "$PDBS" -eq 0 \
  && echo 'PASS: namespace lab-7a khong con Pod va khong con PDB'
```

**PASS:** dòng `PASS: namespace lab-7a khong con Pod…` xuất hiện.

---

## B8. Eviction do áp lực node — đọc cấu hình thật, không ép node vào áp lực

> **Mục này cố ý không kích hoạt eviction.** Lý do đầy đủ nằm ở [cảnh báo mục
> 2](#2-quy-ước-và-an-toàn) và ở bảng *không kiểm chứng được* trong
> [1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành): trên cluster lab, `nodefs` chính là filesystem
> gốc của node, nên mọi cách chạm ngưỡng đều là làm hỏng node. Thay vào đó bạn đọc **ngưỡng thật
> mà kubelet đang dùng**, **giá trị tín hiệu thật** trên `lab-k8s-worker2`, và khoảng cách giữa
> hai thứ đó — rồi giải thích thứ tự trục xuất từ những gì bài mô tả.

Điểm khác biệt lớn nhất so với B7: **chủ thể ở đây là kubelet, không phải API server**. Không
object nào được tạo, không PodDisruptionBudget nào được hỏi. Node tự cứu mình.

### B8.1. `evictionHard` mà kubelet đang chạy

```bash
CFG_W2="$(kubectl get --raw '/api/v1/nodes/lab-k8s-worker2/proxy/configz')"
echo "$CFG_W2" | tr ',' '\n' | grep -iE 'evict|nodeStatusUpdateFrequency|housekeeping' \
  | tee ~/lab-evidence/7a/b8-kubelet-config.txt

EH="$(echo "$CFG_W2" | grep -o '"evictionHard":{[^}]*}')"
echo "$EH" | tee -a ~/lab-evidence/7a/b8-kubelet-config.txt; echo
```

```bash
test -n "$EH" \
  && echo 'PASS: doc duoc evictionHard tu cau hinh dang chay cua kubelet'
echo "$EH" | grep -q '"memory.available":"100Mi"' \
  && echo 'PASS: nguong memory.available dung mac dinh Linux 100Mi'
echo "$EH" | grep -q '"nodefs.available":"10%"' \
  && echo 'PASS: nguong nodefs.available dung mac dinh 10%'
echo "$CFG_W2" | grep -q 'evictionPressureTransitionPeriod' \
  && echo 'PASS: cau hinh co evictionPressureTransitionPeriod — nut chong dao dong dieu kien node'
```

**Ý nghĩa:** bốn gate trên xác nhận hai chuyện. Một, kubelet của baseline **chưa bị sửa**: nó
đang chạy đúng bộ ngưỡng cứng mặc định mà bài [142](../142-node-pressure-eviction-vi.md) liệt
kê (`memory.available<100Mi`, `nodefs.available<10%`, `imagefs.available<15%`,
`nodefs.inodesFree<5%` trên node Linux). Hai, bạn biết chỗ đọc con số thật thay vì tin vào trí
nhớ.

Đây cũng là chỗ nhắc lại **cái bẫy cấu hình quan trọng nhất của bài**: nếu bạn đổi giá trị của
**một** tham số ngưỡng cứng, các tham số còn lại **không kế thừa** mặc định mà bị đặt về
**không**. Muốn tùy chỉnh thì phải khai đủ, hoặc bật tùy chọn `MergeDefaultEvictionSettings`.
Lab không sửa gì — mọi thay đổi cấu hình kubelet thuộc [giai đoạn
20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy).

**PASS:** bốn dòng `PASS:` của bước này xuất hiện. Nếu một giá trị khác mặc định, kubelet trên
node đã bị sửa ngoài quy trình — dừng lại và đối chiếu với Lab 00 trước khi đi tiếp.

### B8.2. Tín hiệu thật trên `lab-k8s-worker2` cách ngưỡng bao xa

```bash
kubectl get --raw '/api/v1/nodes/lab-k8s-worker2/proxy/stats/summary' \
  | tr ',' '\n' | grep -E 'nodeName|availableBytes|capacityBytes|inodesFree|workingSetBytes' \
  | head -n 20 | tee ~/lab-evidence/7a/b8-summary.txt

ssh lab-k8s-worker2 'df -Ph /var/lib/kubelet; df -Pi /var/lib/kubelet' \
  | tee -a ~/lab-evidence/7a/b8-summary.txt

FS_PCT="$(ssh lab-k8s-worker2 'df -Pk /var/lib/kubelet | tail -1' \
  | awk '{printf "%d\n", $4*100/$2}')"
IN_PCT="$(ssh lab-k8s-worker2 'df -Pi /var/lib/kubelet | tail -1' \
  | awk '{printf "%d\n", $4*100/$2}')"
echo "nodefs.available ~ ${FS_PCT}% | nodefs.inodesFree ~ ${IN_PCT}%"
```

```bash
test "$FS_PCT" -gt 10 \
  && echo "PASS: nodefs.available (~${FS_PCT}%) dang tren nguong cung 10%"
test "$IN_PCT" -gt 5 \
  && echo "PASS: nodefs.inodesFree (~${IN_PCT}%) dang tren nguong cung 5%"
```

**Ý nghĩa:** tín hiệu eviction là **trạng thái hiện tại của một tài nguyên tại một thời điểm**,
được so với ngưỡng eviction — lượng tài nguyên tối thiểu nên còn trên node. Hai con số vừa in
ra chính là `nodefs.available` và `nodefs.inodesFree` của bảng tín hiệu, đọc bằng `df` trên
đúng filesystem chứa `/var/lib/kubelet`.

Cluster lab dùng **bố cục đơn giản nhất**: mọi thứ nằm trên một `nodefs` duy nhất, nên
`nodefs`, `imagefs` và `containerfs` cùng trỏ vào một filesystem. Đó là lý do bạn có thể bỏ qua
toàn bộ phần `imagefs` tách riêng và `containerfs` của bài — và cũng là lý do **không có cách
nào** giới hạn một phép thử làm đầy đĩa vào riêng lab.

Khoảng cách bạn vừa đo là toàn bộ headroom. Khi nó cạn, dây chuyền chạy theo đúng thứ tự này:

1. kubelet **thu hồi tài nguyên cấp node trước**: garbage collect pod và container đã chết, rồi
   xóa các image không dùng đến.
2. Nếu vẫn chưa xuống dưới ngưỡng, kubelet **bắt đầu trục xuất Pod của người dùng**, xếp hạng
   theo ba tham số đúng thứ tự: mức sử dụng có **vượt requests** hay không → **Priority** của
   Pod → mức vượt so với requests là bao nhiêu.
3. Kết quả là Pod `BestEffort`/`Burstable` **vượt requests** đi trước; Pod `Guaranteed` và
   `Burstable` **dưới requests** bị trục xuất sau cùng.
4. kubelet đặt phase Pod thành `Failed` và **không tôn trọng** PodDisruptionBudget lẫn
   `terminationGracePeriodSeconds` — ngược hẳn B7.

Ghi chú của bài phải nhớ nguyên văn: kubelet **không dùng lớp QoS** để quyết định thứ tự; QoS
chỉ để bạn **ước lượng** thứ tự đó. Và với inode hay PID thì không có `requests` nào để so, nên
kubelet chỉ dùng độ ưu tiên tương đối.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B8.3. Điều kiện node, taint tương ứng, và toleration được thêm sau lưng bạn

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,MEM:.status.conditions[?(@.type=="MemoryPressure")].status,DISK:.status.conditions[?(@.type=="DiskPressure")].status,PID:.status.conditions[?(@.type=="PIDPressure")].status' \
  | tee ~/lab-evidence/7a/b8-conditions.txt

kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}' \
  | tee -a ~/lab-evidence/7a/b8-conditions.txt
```

```bash
PRESSURE_TRUE="$(grep -c 'True' ~/lab-evidence/7a/b8-conditions.txt || true)"
PRESSURE_TAINT="$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.taints}{"\n"}{end}' \
  | grep -c 'pressure' || true)"
echo "so dieu kien *Pressure dang True=$PRESSURE_TRUE | so taint *-pressure=$PRESSURE_TAINT"

test "$PRESSURE_TRUE" -eq 0 && test "$PRESSURE_TAINT" -eq 0 \
  && echo 'PASS: khong node nao chiu ap luc, va cung khong node nao mang taint ap luc'
```

Bây giờ xem hai loại toleration mà **bạn không hề viết**:

```bash
kubectl run tol-auto -n lab-7a --image=busybox:1.37 --restart=Never \
  --command -- sh -c 'sleep 600'
kubectl wait --for=condition=Ready pod/tol-auto -n lab-7a --timeout=180s
kubectl get pod tol-auto -n lab-7a -o jsonpath='{.spec.tolerations}' \
  | tee ~/lab-evidence/7a/b8-tolerations-pod.txt; echo

KP="$(kubectl -n kube-system get pods -l k8s-app=kube-proxy \
  -o jsonpath='{.items[0].metadata.name}')"
echo "doc toleration cua Pod DaemonSet: $KP"
kubectl -n kube-system get pod "$KP" -o jsonpath='{.spec.tolerations}' \
  | tee ~/lab-evidence/7a/b8-tolerations-ds.txt; echo
```

```bash
NR_SECS="$(kubectl get pod tol-auto -n lab-7a \
  -o jsonpath='{.spec.tolerations[?(@.key=="node.kubernetes.io/not-ready")].tolerationSeconds}')"
UR_SECS="$(kubectl get pod tol-auto -n lab-7a \
  -o jsonpath='{.spec.tolerations[?(@.key=="node.kubernetes.io/unreachable")].tolerationSeconds}')"
echo "tol-auto: not-ready=${NR_SECS}s unreachable=${UR_SECS}s"

test "$NR_SECS" = '300' && test "$UR_SECS" = '300' \
  && echo 'PASS: Kubernetes tu them toleration 300s cho not-ready va unreachable'

grep -q 'node.kubernetes.io/disk-pressure' ~/lab-evidence/7a/b8-tolerations-ds.txt \
  && echo 'PASS: Pod DaemonSet duoc them toleration NoSchedule cho disk-pressure'
grep -q 'node.kubernetes.io/unschedulable' ~/lab-evidence/7a/b8-tolerations-ds.txt \
  && echo 'PASS: Pod DaemonSet duoc them toleration NoSchedule cho unschedulable'

kubectl delete pod tol-auto -n lab-7a --wait=true --timeout=120s
```

**Ý nghĩa:** đây là chỗ bài 142 và bài [139](../139-taint-and-toleration-vi.md) khớp vào nhau.

kubelet báo cáo `MemoryPressure`, `DiskPressure`, `PIDPressure` khi tín hiệu chạm ngưỡng — cứng
hay mềm đều báo, **bất kể grace period**. Control plane **ánh xạ các điều kiện đó thành taint**
(`node.kubernetes.io/memory-pressure`, `…/disk-pressure`, `…/pid-pressure`). Bộ lập lịch
**nhìn taint chứ không nhìn điều kiện node**, và nhờ vậy một node đang chịu áp lực tự động ngừng
nhận Pod mới mà không cần logic riêng nào trong scheduler.

Ba dòng `PASS:` cuối cho thấy hệ quả của thiết kế đó với hai loại workload:

- Pod thường được **tự động** thêm toleration `tolerationSeconds=300` cho `not-ready` và
  `unreachable` — nên khi bạn rút mạng một worker, Pod ở đó còn ở lại 5 phút trước khi bị thu
  hồi, trừ khi bạn tự đặt giá trị khác.
- Pod của **DaemonSet** được controller thêm toleration `NoSchedule` cho các taint áp lực và
  cho `unschedulable`, cộng toleration `NoExecute` cho `not-ready`/`unreachable` **không kèm**
  `tolerationSeconds` — nên chúng không bao giờ bị thu hồi vì hai sự cố đó. Đó chính là lý do
  B4.6 không hạ gục được CNI và kube-proxy.

**PASS:** bốn dòng `PASS:` của mục B8.3 xuất hiện.

---

## B9. Mức sẵn sàng lập lịch của Pod

Mọi mục trước trả lời "Pod này lên node nào". Bài
[145](../145-pod-scheduling-readiness-vi.md) trả lời một câu khác: **khi nào** thì được phép bắt
đầu hỏi câu đó.

### B9.1. Hai gate giữ Pod ngoài hàng đợi

```bash
cat > ~/lab-work/7a/b9-gated.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: lab-7a
spec:
  schedulingGates:
  - name: example.com/foo
  - name: example.com/bar
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/7a/b9-gated.yaml
kubectl apply -f ~/lab-work/7a/b9-gated.yaml
sleep 5
kubectl get pod test-pod -n lab-7a | tee ~/lab-evidence/7a/b9-gated.txt
kubectl get pod test-pod -n lab-7a -o jsonpath='{.spec.schedulingGates}' \
  | tee -a ~/lab-evidence/7a/b9-gated.txt; echo
kubectl get pod test-pod -n lab-7a \
  -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}' \
  | tee -a ~/lab-evidence/7a/b9-gated.txt; echo
```

```bash
REASON="$(kubectl get pod test-pod -n lab-7a \
  -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
NGATES="$(kubectl get pod test-pod -n lab-7a \
  -o jsonpath='{.spec.schedulingGates[*].name}' | wc -w)"
NODE="$(kubectl get pod test-pod -n lab-7a -o jsonpath='{.spec.nodeName}')"
echo "reason=$REASON gates=$NGATES nodeName='${NODE}'"

test "$REASON" = 'SchedulingGated' && test "$NGATES" -eq 2 && test -z "$NODE" \
  && echo 'PASS: Pod o trang thai SchedulingGated, scheduler chua he thu dat no'
```

**Ý nghĩa:** so với `too-big` ở B1.2, cả hai Pod đều chưa chạy, nhưng lý do **khác hẳn**:
`too-big` mang reason `Unschedulable` — nó **đã được thử** và bị kết luận là không lập lịch
được; `test-pod` mang reason `SchedulingGated` — nó **được đánh dấu tường minh là chưa sẵn
sàng**. Nhầm hai trạng thái này dẫn tới chẩn đoán ngược nhau: một bên phải sửa tài nguyên hoặc
ràng buộc, một bên phải đi tìm ai chưa gỡ gate.

**PASS:** dòng `PASS: Pod o trang thai SchedulingGated…` xuất hiện.

### B9.2. Gỡ một gate là chưa đủ

```bash
cat > ~/lab-work/7a/b9-one-gate.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: lab-7a
spec:
  schedulingGates:
  - name: example.com/bar
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/7a/b9-one-gate.yaml
sleep 5
kubectl get pod test-pod -n lab-7a
kubectl get pod test-pod -n lab-7a -o jsonpath='{.spec.schedulingGates}'; echo
```

```bash
NGATES="$(kubectl get pod test-pod -n lab-7a \
  -o jsonpath='{.spec.schedulingGates[*].name}' | wc -w)"
REASON="$(kubectl get pod test-pod -n lab-7a \
  -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
echo "gates=$NGATES reason=$REASON"

test "$NGATES" -eq 1 && test "$REASON" = 'SchedulingGated' \
  && echo 'PASS: con mot gate thi Pod van chua duoc xet lap lich'
```

**Ý nghĩa:** các gate có thể được gỡ **theo thứ tự tùy ý**, nhưng Pod chỉ vào hàng đợi khi
**không còn gate nào**. Mỗi chuỗi trong `schedulingGates` là một tiêu chí phải thỏa.

**PASS:** dòng `PASS: con mot gate thi Pod van chua duoc xet…` xuất hiện.

### B9.3. Gỡ hết gate thì Pod chạy

```bash
cat > ~/lab-work/7a/b9-no-gate.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: lab-7a
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/7a/b9-no-gate.yaml
kubectl wait --for=condition=Ready pod/test-pod -n lab-7a --timeout=180s
kubectl get pod test-pod -n lab-7a -o wide | tee -a ~/lab-evidence/7a/b9-gated.txt
kubectl get pod test-pod -n lab-7a -o jsonpath='{.spec.schedulingGates}'; echo
```

```bash
NGATES="$(kubectl get pod test-pod -n lab-7a \
  -o jsonpath='{.spec.schedulingGates[*].name}' | wc -w)"
PHASE="$(kubectl get pod test-pod -n lab-7a -o jsonpath='{.status.phase}')"
echo "gates=$NGATES phase=$PHASE"

test "$NGATES" -eq 0 && test "$PHASE" = 'Running' \
  && echo 'PASS: sach gate thi Pod vao hang doi va chay binh thuong'
```

**Ý nghĩa:** `test-pod` không xin tài nguyên CPU/bộ nhớ nào, nên ngay khi hết gate nó lên node
mà không phải chờ gì thêm. Toàn bộ thời gian nó đứng yên là do **bên ngoài** giữ, không phải do
cluster thiếu chỗ. Đó chính là lý do tính năng tồn tại: Pod "thiếu tài nguyên thiết yếu" nằm
chờ lâu gây **xáo trộn không cần thiết** cho scheduler và cho các thành phần tích hợp hạ nguồn.

**PASS:** dòng `PASS: sach gate thi Pod vao hang doi…` xuất hiện.

### B9.4. Không thêm được gate mới sau khi Pod đã tạo

```bash
if kubectl patch pod test-pod -n lab-7a --type=merge \
     -p '{"spec":{"schedulingGates":[{"name":"example.com/late"}]}}' \
     > ~/lab-evidence/7a/b9-add-gate.txt 2>&1; then
  echo 'FAIL: them duoc gate moi — trai voi tai lieu'
else
  cat ~/lab-evidence/7a/b9-add-gate.txt
  echo 'PASS: API tu choi them scheduling gate vao Pod da tao'
fi
```

```bash
NGATES="$(kubectl get pod test-pod -n lab-7a \
  -o jsonpath='{.spec.schedulingGates[*].name}' | wc -w)"
test "$NGATES" -eq 0 \
  && echo 'PASS: spec cua Pod khong he thay doi sau lan patch bi tu choi'
```

**Ý nghĩa:** trường `schedulingGates` **chỉ có thể được khởi tạo khi Pod được tạo** — bởi client
hoặc do bị mutate trong quá trình admission. Sau đó bạn chỉ được **gỡ**. Muốn chặn một Pod thì
phải chặn từ lúc tạo; không có cách nào "bắt lại" một Pod đã thả vào hàng đợi.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B9.5. Trong lúc còn gate, chỉ được **siết chặt** chỉ thị lập lịch

```bash
kubectl delete pod test-pod -n lab-7a --wait=true --timeout=120s
kubectl apply -f ~/lab-work/7a/b9-gated.yaml
sleep 5

if kubectl patch pod test-pod -n lab-7a --type=merge \
     -p '{"spec":{"nodeSelector":{"lab7a-preempt":"yes"}}}' \
     > ~/lab-evidence/7a/b9-tighten.txt 2>&1; then
  echo 'PASS: them nodeSelector cho Pod dang bi gate duoc chap nhan'
else
  cat ~/lab-evidence/7a/b9-tighten.txt
  echo 'FAIL: khong them duoc nodeSelector'
fi
kubectl get pod test-pod -n lab-7a -o jsonpath='{.spec.nodeSelector}'; echo
```

```bash
NS="$(kubectl get pod test-pod -n lab-7a \
  -o jsonpath='{.spec.nodeSelector.lab7a-preempt}')"
REASON="$(kubectl get pod test-pod -n lab-7a \
  -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
echo "nodeSelector.lab7a-preempt='${NS}' reason=$REASON"

test "$NS" = 'yes' && test "$REASON" = 'SchedulingGated' \
  && echo 'PASS: chi thi lap lich duoc siet chat trong khi Pod van dung ngoai hang doi'

kubectl delete pod test-pod -n lab-7a --wait=true --timeout=120s
```

**Ý nghĩa:** khoảng thời gian Pod còn gate là cửa sổ để một controller bên ngoài **thu hẹp** tập
node ứng viên: thêm `nodeSelector` (chỉ được thêm mới), đặt `nodeAffinity` khi nó đang nil, hoặc
thêm biểu thức vào `matchExpressions` đã có. Không được **nới lỏng** hay sửa cái đã có — vì các
term trong `nodeSelectorTerms` được OR còn các biểu thức trong `matchExpressions` được AND, nên
chỉ "thêm" mới bảo toàn được hướng siết chặt. Riêng phần
`preferredDuringSchedulingIgnoredDuringExecution` thì sửa tùy ý, vì nó không mang tính ràng
buộc bắt buộc.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B10. Pod Overhead, hiệu năng scheduler, framework và bin packing — phần đọc được

Bốn bài cuối của nhóm không có trường nào để bạn viết vào Pod trên cluster lab. Mục này kiểm
chứng đúng **phần đọc được từ cluster**, phần còn lại đã ghi rõ lý do ở bảng trong
[1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành). **Không sửa cấu hình scheduler.**

### B10.1. Pod Overhead: trường có trong API, nhưng không có nguồn để điền

```bash
kubectl get runtimeclass 2>&1 | tee ~/lab-evidence/7a/b10-runtimeclass.txt
kubectl get runtimeclass \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.handler}{"\t"}{.overhead}{"\n"}{end}' \
  2>/dev/null | tee -a ~/lab-evidence/7a/b10-runtimeclass.txt

kubectl explain pod.spec.overhead | tee ~/lab-evidence/7a/b10-overhead.txt

kubectl run oh-demo -n lab-7a --image=busybox:1.37 --restart=Never \
  --command -- sh -c 'sleep 300'
kubectl wait --for=condition=Ready pod/oh-demo -n lab-7a --timeout=180s
kubectl get pod oh-demo -n lab-7a -o jsonpath='{.spec.overhead}' \
  | tee -a ~/lab-evidence/7a/b10-overhead.txt; echo
```

```bash
RC_WITH_OVERHEAD="$(kubectl get runtimeclass \
  -o jsonpath='{range .items[*]}{.overhead}{"\n"}{end}' 2>/dev/null \
  | grep -c 'podFixed' || true)"
OH="$(kubectl get pod oh-demo -n lab-7a -o jsonpath='{.spec.overhead}')"
echo "so RuntimeClass co overhead=$RC_WITH_OVERHEAD | oh-demo.spec.overhead='${OH}'"

grep -qi 'overhead' ~/lab-evidence/7a/b10-overhead.txt \
  && echo 'PASS: truong spec.overhead ton tai trong API cua baseline'
test "$RC_WITH_OVERHEAD" -eq 0 \
  && echo 'PASS: khong RuntimeClass nao khai overhead, nen khong co nguon de dien'
test -z "$OH" \
  && echo 'PASS: Pod thuong khong mang khoan cong them nao'

kubectl delete pod oh-demo -n lab-7a --wait=true --timeout=120s
```

**Ý nghĩa:** ba gate trên khóa đúng ranh giới của bài
[144](../144-pod-overhead-vi.md). `spec.overhead` **không phải trường bạn viết**: admission
controller RuntimeClass điền nó theo `overhead.podFixed` của RuntimeClass mà Pod chỉ định, và
nếu PodSpec đã tự định nghĩa sẵn trường này thì **Pod bị từ chối**. Cluster lab chạy containerd
với runc — một runtime không tốn thêm hạ tầng cho mỗi Pod — nên không có RuntimeClass nào khai
`overhead` và mọi Pod của bạn có `spec.overhead` rỗng.

Phải nhớ ba chỗ overhead **sẽ** thay đổi hành vi nếu cluster có runtime ảo hóa: bộ lập lịch cộng
overhead vào tổng request khi tìm node; kubelet cộng nó vào **kích thước cgroup của Pod**; và nó
tham gia **xếp hạng eviction** ở B8. Ngoài ba chỗ đó, nó còn được tính vào ResourceQuota — công
cụ của nhóm [7b](../00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên).

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B10.2. kube-scheduler đang chạy bằng cấu hình mặc định

Chạy trên `lab-k8s-master`. **Chỉ đọc** — không sửa file trong `/etc/kubernetes/manifests`.

```bash
sudo grep -nE 'image:|- kube-scheduler|--' /etc/kubernetes/manifests/kube-scheduler.yaml \
  | tee ~/lab-evidence/7a/b10-scheduler-flags.txt
NODE_COUNT="$(kubectl get nodes --no-headers | wc -l)"
echo "so node cua cluster = $NODE_COUNT" | tee -a ~/lab-evidence/7a/b10-scheduler-flags.txt
```

```bash
if sudo grep -q -- '--config=' /etc/kubernetes/manifests/kube-scheduler.yaml; then
  echo 'FAIL: kube-scheduler dang nap mot KubeSchedulerConfiguration — cluster lech baseline'
else
  echo 'PASS: kube-scheduler khong nap file cau hinh nao, dang dung toan bo mac dinh'
fi
test "$NODE_COUNT" -lt 100 \
  && echo "PASS: cluster $NODE_COUNT node — duoi nguong 100 node kha thi, scheduler van kiem tra tat ca"
```

**Ý nghĩa:** hai kết luận của bài [146](../146-scheduler-perf-tuning-vi.md), kiểm chứng bằng hai
dòng.

`percentageOfNodesToScore` là ngưỡng "đủ" để scheduler **dừng tìm node khả thi** thay vì xem
xét mọi node — một cách đổi độ chính xác lấy độ trễ, vì node bị bỏ qua ở bước lọc không bao giờ
được chấm điểm. Nhưng ngưỡng mặc định được tính tự động theo kích thước cluster và **dưới 100
node khả thi thì scheduler vẫn kiểm tra tất cả**. Cluster ba node của bạn không có gì để tinh
chỉnh, và cũng không có tác động nào để đo — đúng như bài dặn: cluster vài trăm node trở xuống
thì cứ để mặc định.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không có dòng `FAIL:`.

### B10.3. Ánh xạ những gì đã thấy vào các điểm mở rộng của scheduling framework

Bài [147](../147-scheduling-framework-vi.md) viết cho người phát triển plugin, nhưng ở vị trí
này nó là **bản đồ ghép mọi thứ bạn vừa làm vào một khung duy nhất**. Mỗi lần thử lập lịch một
Pod gồm **chu trình lập lịch** (chọn node) rồi **chu trình binding** (áp quyết định lên
cluster); các chu trình lập lịch chạy tuần tự, các chu trình binding có thể chạy song song.

| Điểm mở rộng | Bạn đã nhìn thấy nó ở đâu | Bằng chứng |
| --- | --- | --- |
| `PreEnqueue` — chốt chặn trước khi Pod vào hàng đợi active | B9 — `schedulingGates` giữ Pod ngoài hàng đợi mà **không** gán reason `Unschedulable` | `b9-gated.txt` |
| `QueueSort` — sắp thứ tự hàng đợi (chỉ bật được **một** plugin loại này) | B6 — `spec.priority` quyết định Pod nào được xét trước | `b6-so-lieu.txt` |
| `Filter` — bước lọc | B1.2 và B4.2 — thông điệp liệt kê lý do loại từng node | `b1-unschedulable.txt`, `b4-notol.txt` |
| `Score` + `NormalizeScore` — bước chấm điểm | B2.4 — `weight` của quy tắc `preferred` | `b2-preferred.txt` |
| `PostFilter` — nơi preemption sống, **chỉ chạy khi không tìm thấy node khả thi nào** | B6.4 — `nominatedNodeName` chỉ xuất hiện sau khi bước lọc trả về rỗng | `b6-preempt.txt` |
| `Reserve`/`Unreserve`, `Permit`, `PreBind`, `Bind` | Không quan sát trực tiếp được từ ngoài; `Bind` là bước sinh ra event `Scheduled` mà B1.1 đọc | `b1-events-plain.txt` |

```bash
BAD=0
for f in b9-gated.txt b6-so-lieu.txt b1-unschedulable.txt b4-notol.txt \
         b2-preferred.txt b6-preempt.txt b1-events-plain.txt; do
  if [ ! -s "$HOME/lab-evidence/7a/$f" ]; then
    echo "FAIL: thieu bang chung $f"
    BAD=1
  fi
done
test "$BAD" -eq 0 \
  && echo 'PASS: du bang chung cho tung diem mo rong da quan sat duoc'
```

**Ý nghĩa:** hai điều đáng nhớ nhất của bài 147 nằm ngay trong bảng trên. Một, **preemption
không phải một tính năng rời**: nó là plugin ở `PostFilter`, và `PostFilter` **chỉ chạy khi bước
lọc không trả về node nào** — đúng thứ tự bạn quan sát ở B6.4. Hai, **scheduling gate không phải
một dạng `Unschedulable`**: nó chặn ở `PreEnqueue`, trước cả hàng đợi, nên Pod không mang điều
kiện `Unschedulable`.

Chu trình nào cũng có thể **bị hủy** khi Pod được kết luận là không lập lịch được hoặc có lỗi
nội bộ; khi đó Pod được trả về hàng đợi và thử lại — chính là hành vi bạn thấy ở B2.2.

**PASS:** dòng `PASS: du bang chung…` xuất hiện, không có dòng `FAIL:`.

### B10.4. Bin packing đang tắt, và vì sao

```bash
if sudo grep -q -- '--config=' /etc/kubernetes/manifests/kube-scheduler.yaml; then
  echo 'FAIL: co file cau hinh — phai doc scoringStrategy trong do'
else
  echo 'PASS: khong co scoringStrategy tuy bien, NodeResourcesFit dung chien luoc mac dinh'
fi
kubectl get nodes -o custom-columns='NODE:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory' \
  | tee ~/lab-evidence/7a/b10-binpacking.txt
kubectl describe nodes | grep -A6 'Allocated resources' \
  | tee -a ~/lab-evidence/7a/b10-binpacking.txt
```

**Ý nghĩa:** bài [148](../148-resource-bin-packing-vi.md) là bài duy nhất của nhóm **đổi cách
chấm điểm mặc định** thay vì thêm ràng buộc cho Pod, và toàn bộ nó là cấu hình cấp cluster
trong `KubeSchedulerConfiguration` — không có trường nào trong Pod spec để thử.

Hai chiến lược của plugin `NodeResourcesFit` đều tác động **chỉ vào bước chấm điểm**, không đụng
bước lọc: `MostAllocated` chấm điểm cao cho node đã cấp phát nhiều, để dồn Pod lên ít node nhất
có thể — mục đích bài nêu là chuẩn bị cho việc thu hẹp các node ít dùng.
`RequestedToCapacityRatio` cho bạn tự vẽ đường cong qua `shape`; điểm **tăng** theo `utilization`
là bin packing, đảo ngược lại là chế độ ưu tiên node dùng ít nhất, tức trải Pod ra.

Danh sách `resources` quyết định tài nguyên nào được tính điểm — tài nguyên **không** nằm trong
danh sách không đóng góp gì; mặc định là `cpu` và `memory` với `weight: 1`, `weight` không được
âm, và mọi tài nguyên trong danh sách dùng chung một `shape`.

**PASS:** dòng `PASS: khong co scoringStrategy tuy bien…` xuất hiện.

---

## B11. Cleanup và gate cuối

**Mục đích:** xóa mọi thứ lab tạo ra — trong namespace, trên Node object, ở phạm vi cluster, và
trên filesystem của node — rồi chứng minh cluster trở về đúng `03-storage-ready`.

### B11.1. Xóa object của bài học

```bash
kubectl delete namespace lab-7a --wait=true --timeout=300s

# Taint: B4.7 da go, ba lenh duoi la go lai cho chac va khong bao loi neu da sach.
kubectl taint nodes lab-k8s-worker2 lab7a=demo:NoExecute- 2>/dev/null || true
kubectl taint nodes lab-k8s-worker2 lab7a=demo:NoSchedule- 2>/dev/null || true
kubectl taint nodes lab-k8s-worker2 lab7a-extra=demo:NoSchedule- 2>/dev/null || true

# Label lab dat len node.
kubectl label nodes lab-k8s-worker1 disktype- 2>/dev/null || true
kubectl label nodes lab-k8s-worker2 disktype- 2>/dev/null || true
kubectl label nodes lab-k8s-worker1 lab7a-zone- 2>/dev/null || true
kubectl label nodes lab-k8s-worker2 lab7a-zone- 2>/dev/null || true
kubectl label nodes lab-k8s-worker2 lab7a-preempt- 2>/dev/null || true

# Hai object pham vi cluster con lai.
kubectl delete priorityclass lab-7a-low lab-7a-high --ignore-not-found=true

kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints' \
  | tee ~/lab-evidence/7a/b11-sau.txt
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels}{"\n"}{end}' \
  | tee -a ~/lab-evidence/7a/b11-sau.txt
kubectl get priorityclass | tee -a ~/lab-evidence/7a/b11-sau.txt
```

```bash
NS_LEFT="$(kubectl get namespace lab-7a --no-headers 2>/dev/null | wc -l)"
W2_TAINTS="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.spec.taints}')"
W1_TAINTS="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.spec.taints}')"
LAB_LABELS="$(kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.labels}{"\n"}{end}' \
  | grep -cE 'disktype|lab7a' || true)"
LAB_PC="$(kubectl get priorityclass -o jsonpath='{.items[*].metadata.name}' \
  | tr ' ' '\n' | grep -c '^lab-7a-' || true)"
echo "ns=$NS_LEFT | w1.taints='${W1_TAINTS}' w2.taints='${W2_TAINTS}' | label lab=$LAB_LABELS | priorityclass lab=$LAB_PC"

test "$NS_LEFT" -eq 0   && echo 'PASS: namespace lab-7a da xoa'
test -z "$W1_TAINTS" && test -z "$W2_TAINTS" \
  && echo 'PASS: hai worker sach taint, dung nhu anh nen o B0'
test "$LAB_LABELS" -eq 0 && echo 'PASS: khong node nao con label cua lab 7a'
test "$LAB_PC" -eq 0     && echo 'PASS: hai PriorityClass cua lab da bi xoa'
```

**PASS:** bốn dòng `PASS:` của bước này xuất hiện. Nếu namespace kẹt `Terminating`, còn Pod
trong grace period — chờ hết `terminationGracePeriodSeconds`, đừng cưỡng chế finalizer.

### B11.2. Taint của control plane phải còn nguyên

```bash
CP_KEY="$(kubectl get node lab-k8s-master -o jsonpath='{.spec.taints[0].key}')"
CP_EFFECT="$(kubectl get node lab-k8s-master -o jsonpath='{.spec.taints[0].effect}')"
echo "master taint: ${CP_KEY}:${CP_EFFECT}"

test "$CP_KEY" = 'node-role.kubernetes.io/control-plane' && test "$CP_EFFECT" = 'NoSchedule' \
  && echo 'PASS: taint control plane con nguyen — khong lab nao duoc go no'
```

**PASS:** dòng `PASS: taint control plane con nguyen…` xuất hiện. Nếu taint biến mất, một bước
nào đó đã chạy sai node; restore snapshot `03-storage-ready` trên cả ba VM chứ đừng tự đặt lại
taint bằng tay.

### B11.3. Lab không tiêu đĩa của node

Lab 7a **không tạo file lớn nào trên node** — mục 2 đã cấm phép thử làm đầy đĩa, và không bước
nào ghi ra ngoài container. Gate dưới đây chứng minh điều đó bằng số, thay vì bằng lời:

```bash
W2_AVAIL_KB1="$(ssh lab-k8s-worker2 'df -Pk /var/lib/kubelet | tail -1' | awk '{print $4}')"
DELTA_KB=$(( W2_AVAIL_KB0 - W2_AVAIL_KB1 ))
echo "nodefs worker2: truoc=${W2_AVAIL_KB0} KiB sau=${W2_AVAIL_KB1} KiB | tieu them=${DELTA_KB} KiB" \
  | tee -a ~/lab-evidence/7a/b11-sau.txt

ssh lab-k8s-worker2 'ls -1 /var/tmp 2>/dev/null | head' \
  | tee -a ~/lab-evidence/7a/b11-sau.txt
```

```bash
test "$DELTA_KB" -lt 524288 \
  && echo "PASS: worker2 chi tieu them ${DELTA_KB} KiB — khong co file lon nao bi bo lai" \
  || echo "FAIL: worker2 mat ${DELTA_KB} KiB — tim file lab bo quen truoc khi ket thuc"
```

**Ý nghĩa:** ngưỡng 512 MiB không phải hạn mức cho phép, nó là **biên nhiễu**: log của kubelet
và container tăng lên trong 3–4 giờ chạy lab. Con số thật phải nhỏ hơn nhiều. Nếu nó vượt
ngưỡng, có thứ gì đó đã ghi ra đĩa node — tìm và xóa trước khi kết thúc lab, vì snapshot
`03-storage-ready` không kèm nó.

**PASS:** dòng `PASS: worker2 chi tieu them…` xuất hiện, không có dòng `FAIL:`.

### B11.4. Dọn manifest tạm

```bash
rm -f ~/lab-work/7a/b1-plain.yaml ~/lab-work/7a/b1-too-big.yaml ~/lab-work/7a/b1-direct.yaml \
      ~/lab-work/7a/b2-selector.yaml ~/lab-work/7a/b2-required.yaml \
      ~/lab-work/7a/b2-preferred.yaml ~/lab-work/7a/b2-or-and.yaml ~/lab-work/7a/b2-both.yaml \
      ~/lab-work/7a/b3-store.yaml ~/lab-work/7a/b3-web.yaml ~/lab-work/7a/b3-store-soft.yaml \
      ~/lab-work/7a/b4-notol.yaml ~/lab-work/7a/b4-tol.yaml ~/lab-work/7a/b4-partial.yaml \
      ~/lab-work/7a/b4-direct2.yaml ~/lab-work/7a/b4-noexec.yaml \
      ~/lab-work/7a/b5-seed.yaml ~/lab-work/7a/b5-spread.yaml ~/lab-work/7a/b5-policy.yaml \
      ~/lab-work/7a/b5-policy-ignore.yaml ~/lab-work/7a/b5-anyway.yaml \
      ~/lab-work/7a/b5-orphan.yaml \
      ~/lab-work/7a/b6-priorityclass.yaml ~/lab-work/7a/b6-low.yaml ~/lab-work/7a/b6-high.yaml \
      ~/lab-work/7a/b7-setup.yaml ~/lab-work/7a/b7-evict-solo.json \
      ~/lab-work/7a/b7-evict-web.json \
      ~/lab-work/7a/b9-gated.yaml ~/lab-work/7a/b9-one-gate.yaml ~/lab-work/7a/b9-no-gate.yaml
rmdir ~/lab-work/7a
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` bên dưới biến
điều đó thành gate thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/7a/` **giữ lại** — đó là
bằng chứng.

```bash
test ! -e ~/lab-work/7a && echo 'PASS: manifest tam da xoa het'
```

**PASS:** dòng `PASS: manifest tam da xoa het` xuất hiện.

### B11.5. Gate cuối — cluster ở đúng `03-storage-ready`

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl get pods -n default
kubectl get pdb -A
kubectl -n kube-system get deployment coredns
kubectl get storageclass
kubectl get namespace
```

```bash
BAD="$(kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded \
  --no-headers 2>/dev/null | wc -l)"
DEF_PODS="$(kubectl get pods -n default --no-headers 2>/dev/null | wc -l)"
PDB_ALL="$(kubectl get pdb -A --no-headers 2>/dev/null | wc -l)"
SC_DEFAULT="$(kubectl get storageclass 2>/dev/null | grep -c '(default)' || true)"
LAB_NS="$(kubectl get namespace --no-headers | grep -c 'lab-7a' || true)"
echo "pod khong Running=$BAD | pod trong default=$DEF_PODS | PDB toan cluster=$PDB_ALL | SC mac dinh=$SC_DEFAULT | ns lab-7a=$LAB_NS"

test "$BAD" -eq 0        && echo 'PASS: moi Pod trong cluster deu Running hoac Succeeded'
test "$DEF_PODS" -eq 0   && echo 'PASS: namespace default khong con Pod nao'
test "$PDB_ALL" -eq 0    && echo 'PASS: khong con PodDisruptionBudget nao cua lab'
test "$SC_DEFAULT" -ge 1 && echo 'PASS: StorageClass mac dinh con nguyen — moc 03-storage-ready khong bi dung toi'
test "$LAB_NS" -eq 0     && echo 'PASS: khong con namespace lab-7a'
```

**PASS:** không có dòng `FAIL:` nào; sáu dòng `PASS:` của bước này xuất hiện; ba node `Ready`;
CoreDNS đủ replica `READY`. Cluster trở về `03-storage-ready`; **không tạo snapshot mới**.

Một khác biệt được phép so với lúc bắt đầu: **vị trí** của vài Pod hạ tầng có thể đã chuyển từ
`lab-k8s-worker2` sang `lab-k8s-worker1` do taint `NoExecute` ở B4.6. Snapshot
`03-storage-ready` khai báo *hạ tầng nào có mặt*, không khai báo *Pod nằm ở node nào*, nên đây
không phải sai lệch. Nếu bạn muốn bố cục đối xứng lại, xóa Pod của Deployment tương ứng và để
controller đặt lại — nhưng việc đó không bắt buộc.

---

## 3. Checkpoint 7a

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Kể hai bước của kube-scheduler theo đúng thứ tự, mỗi bước nhận vào gì và trả ra gì. Nếu
      danh sách sau bước lọc rỗng thì Pod ở trạng thái nào, và bước chấm điểm có chạy không?
- [ ] `nodeSelector`, node affinity `required` và node affinity `preferred` khác nhau ở chỗ
      nào? Đặt cả `nodeSelector` lẫn `nodeAffinity` thì Pod cần thỏa mấy điều kiện? Trong một
      `nodeAffinity`, hai phần tử của `nodeSelectorTerms` quan hệ với nhau thế nào, còn hai
      biểu thức trong cùng một `matchExpressions` thì thế nào?
- [ ] Một Pod đang chạy nhờ node affinity `required` khớp label `disktype=ssd`. Bạn gỡ label
      đó. Pod có bị đuổi không? Muốn đuổi thật thì phải dùng cơ chế nào?
- [ ] Bạn tạo một Deployment 3 replica có `podAntiAffinity` loại `required`,
      `topologyKey: kubernetes.io/hostname`, selector khớp chính label của các replica đó. Trên
      cluster lab bao nhiêu Pod chạy được, và replica còn lại ở đâu? Đổi sang `preferred` thì sao?
- [ ] Taint đặt trên đối tượng nào, toleration đặt trên đối tượng nào? Thêm toleration có làm
      Pod **được đưa** lên node bị taint không? Một node có ba taint mà Pod chỉ dung thứ hai
      trong đó, taint còn lại là `NoSchedule` — Pod có lên được không, và nếu Pod **đã chạy từ
      trước** thì sao?
- [ ] `NoSchedule`, `PreferNoSchedule` và `NoExecute` khác nhau thế nào với Pod **đang chạy**?
      `tolerationSeconds` thay đổi điều gì, và nếu bạn gỡ taint trước khi hết khoảng đó?
- [ ] Một Pod được đặt bằng `nodeName` lên node có taint `NoSchedule` — nó chạy được. Vì sao?
      Nếu node đó cũng có taint `NoExecute` thì kết cục ra sao, và ai thi hành?
- [ ] `maxSkew` đo chênh lệch giữa cái gì và cái gì? Một node **không có** label được nêu ở
      `topologyKey` thì chuyện gì xảy ra với nó và với các Pod trên nó? `nodeAffinityPolicy`
      `Honor` khác `Ignore` ra sao trong phép tính đó?
- [ ] Preemption được kích hoạt khi nào, nó ghi gì vào `status` của Pod đang chờ, và vì sao nó
      **không** loại bỏ hết các Pod ưu tiên thấp hơn? PodDisruptionBudget có ngăn được nó không?
- [ ] Eviction qua API và eviction do áp lực node đối xử với PodDisruptionBudget và
      `terminationGracePeriodSeconds` khác nhau thế nào? Kể ba mã trả về của Eviction API và ý
      nghĩa từng mã. Trong sáu bước xóa Pod, ở bước nào Pod ngừng **nhận** request mới?
- [ ] kubelet xếp thứ tự trục xuất theo ba tham số nào, đúng thứ tự? Nhóm Pod nào bị trục xuất
      trước, nhóm nào sau cùng? kubelet có dùng QoS class để quyết định thứ tự không?
- [ ] `SchedulingGated` khác `Pending` vì thiếu tài nguyên ở chỗ nào từ góc nhìn scheduler? Sau
      khi Pod đã tạo, bạn còn làm được gì với `schedulingGates` và không làm được gì?

### Bài giải thích cuối cùng

Trong tối đa 10 phút, kể lại vòng đời chỗ ngồi của một Pod trên cluster của bạn:

1. Pod sinh ra chưa có node. Nếu nó mang `schedulingGates`, scheduler **chưa hề nhìn tới** —
   nó đứng trước hàng đợi, không phải trong hàng đợi.
2. Sạch gate, Pod vào hàng đợi và được xếp chỗ theo `spec.priority`.
3. Bước **lọc** loại node: taint không được dung thứ, label không khớp `nodeSelector`/affinity,
   `topologyKey` thiếu, tài nguyên không đủ. Kết quả là tập node khả thi.
4. Bước **chấm điểm** xếp hạng chính tập đó: `weight` của quy tắc `preferred`, độ lệch của ràng
   buộc phân bố `ScheduleAnyway`, chiến lược của `NodeResourcesFit`. Node cao điểm nhất thắng;
   bằng điểm thì chọn ngẫu nhiên.
5. Tập khả thi rỗng thì `PostFilter` chạy — preemption đi tìm nạn nhân ưu tiên thấp hơn, ghi
   `nominatedNodeName`, và PDB chỉ được tôn trọng ở mức nỗ lực tốt nhất.
6. Có node rồi, scheduler ghi **binding**; từ đó trở đi mọi chuyện là của kubelet.
7. Pod đang chạy vẫn có thể mất chỗ theo ba đường khác nhau: taint `NoExecute` được thêm vào
   node; ai đó gọi **Eviction API** và PDB cùng grace period được tôn trọng; hoặc node chạm
   ngưỡng eviction và **kubelet tự** chấm dứt Pod, không hỏi ai.
8. Ba đường đó khác nhau ở **ai ra quyết định** — taint-eviction-controller, API server, hay
   kubelet — và đó là điều duy nhất cần nhớ để chẩn đoán đúng khi một Pod biến mất.

Khi mọi checkbox được đánh dấu và không còn nhầm affinity với taint, `required` với `preferred`,
preemption với eviction, hay eviction qua API với eviction do áp lực node, Lab 7a và nhóm
[7a của giai đoạn 7](../00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction) hoàn tất. Phần còn lại
của giai đoạn 7 — ResourceQuota, LimitRange, PID limit — nằm ở nhóm
[7b](../00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên) và lab 7b.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Bảng dưới chỉ
liệt kê sự cố phát sinh từ nội dung bài học 7a.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| Gate mở đầu báo một worker đã mang taint | `kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints'` | Lab trước chưa cleanup hoặc một phiên 7a dở dang còn sót. Gỡ taint bằng `kubectl taint nodes <node> <key>-`; nếu không rõ nguồn gốc, restore cả ba VM về `03-storage-ready` |
| `W2_FREE_M` nhỏ hơn 400 ở B0.1 | `kubectl get pods -A -o wide --field-selector spec.nodeName=lab-k8s-worker2` | Còn workload của lab trước trên worker2. Xóa nó rồi chạy lại B0.1 — đừng hạ tỉ lệ trong công thức, vì B6 dựa vào khoảng trống này |
| `Insufficient cpu` không xuất hiện ở B1.2 | In lại `MAX_ALLOC_M` và `TOO_BIG_M` | Node có nhiều CPU hơn dự kiến hoặc `cpu2m` đọc sai đơn vị; kiểm tra `kubectl get nodes -o jsonpath='{.items[*].status.allocatable.cpu}'` rồi tăng phần cộng thêm trong `TOO_BIG_M` |
| B2.2 gắn label rồi mà Pod vẫn `Pending` | `kubectl get nodes -L disktype`; `kubectl describe pod sel-nvme -n lab-7a` | Label gắn nhầm node hoặc gõ sai giá trị. Scheduler thử lại theo chu kỳ — chờ thêm vài vòng lặp trước khi kết luận; thời gian thử lại **phụ thuộc cấu hình** |
| B3.1 có 3 Pod `Running` thay vì 2 | `kubectl get pods -n lab-7a -l app=store -o wide` | Một Pod không mang label `app=store` nên không bị anti-affinity đếm, hoặc `topologyKey` bị sửa. So lại manifest với bản trong B3.1 |
| B3.2 cả ba `web-store` đều `Pending` | `kubectl get pods -n lab-7a -l app=store -o wide` | `podAffinity` chỉ thỏa khi **đã có** Pod `app=store` đang chạy. Chạy lại B3.1 trước, đừng đổi thứ tự hai bước |
| B4.6: `nx-none` và `nx-secs` biến mất cùng lúc, `WINDOW` bằng 0 | `kubectl get pod nx-secs -n lab-7a -o jsonpath='{.spec.tolerations}'` | `tolerationSeconds` không được ghi vào Pod (sai thụt lề YAML) nên nó bị coi như không có. Sửa manifest và chạy lại B4.6 từ đầu |
| B4.6 làm một Pod hạ tầng `Pending` mãi | `kubectl describe pod <pod> -n <ns>` | Taint `NoExecute` chưa được gỡ, hoặc worker1 hết chỗ. Chạy B4.7 trước, rồi chờ controller đặt lại Pod; **không** gỡ taint control plane để lấy thêm chỗ |
| B5.2 `spread-1` lên worker1 chứ không worker2 | `kubectl get nodes -L lab7a-zone`; `kubectl get pods -n lab-7a -l app=spread -o wide` | Hai seed Pod không nằm cùng một miền, hoặc worker2 chưa có label `lab7a-zone=b`. Kiểm tra B5.1 rồi chạy lại B5.2 |
| B5.3 `spread-ignore` lại chạy được | `kubectl get pod spread-ignore -n lab-7a -o jsonpath='{.spec.topologySpreadConstraints}'` | `nodeAffinityPolicy: Ignore` không được ghi vào spec. Nếu field bị API server bỏ qua, cluster đang tắt feature gate liên quan — ghi lại hiện tượng vào evidence và đọc mục *nodeAffinityPolicy* của bài [140](../140-topology-spread-constraints-vi.md); **không** bật feature gate |
| B6.4 không bắt được `nominatedNodeName` | `kubectl get pod low-1 -n lab-7a -o jsonpath='{.spec.terminationGracePeriodSeconds}'` | Nạn nhân chết quá nhanh nên cửa sổ quan sát đóng lại. Phải là 60 và container phải dùng `trap '' TERM`; nếu shell vẫn nhận `SIGTERM`, sửa `command` cho đúng rồi chạy lại B6.3–B6.4 |
| B6.4 nạn nhân là một Pod hạ tầng, không phải `low-*` | `kubectl get priorityclass lab-7a-low -o jsonpath='{.value}'` | Giá trị của `lab-7a-low` phải **âm** để thấp hơn Pod ưu tiên 0. Nếu bạn đã sửa thành số dương, xóa PriorityClass, tạo lại theo B6.2 và chạy lại B6.3 |
| B6.4 `high-1` mãi `Pending` và không preempt ai | `kubectl describe pod high-1 -n lab-7a`; in lại `HIGH_M` | `HIGH_M` lớn hơn cả `W2_FREE_M` nên bỏ hết Pod `low` cũng không đủ chỗ — đọc lại `W2_FREE_M` ở B6.2 rồi tính lại. Cũng kiểm tra `lab7a-preempt=yes` còn trên worker2 |
| B7.2 `evict-solo` biến mất ngay, không kịp đọc `deletionTimestamp` | `kubectl get pod evict-solo -n lab-7a -o yaml` | Container thoát ngay khi nhận `SIGTERM`. Manifest phải có `trap '' TERM` và `terminationGracePeriodSeconds: 60`; tạo lại rồi chạy lại B7.2 |
| B7.3 eviction lại thành công | `kubectl get pdb evict-web -n lab-7a -o yaml` | `minAvailable` phải bằng đúng số replica đang `Ready`. Nếu một replica chưa `Ready`, `currentHealthy` khác 3 và ngân sách tính khác — chờ `rollout status` xong rồi thử lại |
| B8.1 `evictionHard` khác giá trị mặc định | `ssh <node> 'sudo cat /var/lib/kubelet/config.yaml'` | Kubelet đã bị sửa ngoài quy trình Lab 00. Không sửa tiếp ở đây: ghi hiện tượng vào evidence, đọc B8 với chính con số của cluster bạn, và xử lý việc khôi phục baseline ở [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |
| `kubectl get --raw .../proxy/configz` bị từ chối | `kubectl auth can-i get nodes/proxy` | User đang dùng không có quyền proxy tới kubelet. Dùng kubeconfig quản trị của Lab 00 trên `lab-k8s-master`; **không** tạo thêm quyền — RBAC là nội dung [giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) |
| B8.3 không thấy toleration `300` trên Pod thường | `kubectl get pod tol-auto -n lab-7a -o yaml` | Một controller hoặc admission đã đặt toleration tường minh trước đó — bài nói rõ Kubernetes chỉ tự thêm khi bạn hoặc controller **chưa** đặt. Đọc giá trị thực tế thay vì kỳ vọng 300 |
| B9.1 Pod hiện `Pending` chứ không `SchedulingGated` | `kubectl get pod test-pod -n lab-7a -o jsonpath='{.spec.schedulingGates}'` | Gate không được ghi vào spec (sai thụt lề). `kubectl apply --dry-run=server --validate=strict` sẽ bắt lỗi này — chạy lại nó trước khi apply thật |
| B11.3 báo worker2 mất nhiều hơn 512 MiB | Chạy trên worker2: `sudo du -xh --max-depth=1 /` rồi sắp xếp theo dung lượng | Có thứ gì đó ghi ra đĩa node. Tìm và xóa **chỉ file do lab tạo**; nếu không xác định được nguồn gốc, restore cả ba VM về `03-storage-ready` thay vì xóa mò |
| Namespace `lab-7a` kẹt `Terminating` | `kubectl get pods -n lab-7a -o wide` | Còn Pod trong grace period — `evict-solo` và `low-*` có grace 60 giây. Chờ hết; nếu vẫn kẹt, xem `kubectl get ns lab-7a -o yaml` và **không** cưỡng chế finalizer namespace |

---

## 5. Nguồn chính thức

- [Scheduling, Preemption and Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [API-initiated Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/)
- [Pod Overhead](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/)
- [Pod Scheduling Readiness](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/)
- [Scheduler Performance Tuning](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/)
- [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [Resource Bin Packing](https://kubernetes.io/docs/concepts/scheduling-eviction/resource-bin-packing/)
- [Assign Pods to Nodes](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/)
- [Assign Pods to Nodes using Node Affinity](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/)
- [Guaranteed Scheduling For Critical Add-On Pods](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/)
- [Eviction API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#create-eviction-pod-v1-core)
