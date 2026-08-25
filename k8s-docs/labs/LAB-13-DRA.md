# Lab 13 — DRA

> **Điểm bắt đầu:** snapshot `04-metrics-ready` — mốc do Lab 11a tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot).
> **Điểm kết thúc:** cleanup trả cluster về đúng `04-metrics-ready`, **không tạo snapshot mới**.
> Lab này không cài driver, không bật feature gate, không sửa cấu hình apiserver/kubelet/scheduler.
> **Lab tùy chọn.** [Bản đồ lab](README.md#4-bản-đồ-lab) ghi rõ: lab 13 *chỉ làm được nếu lab có GPU
> hoặc thiết bị chuyên dụng*. Ba VM của [Lab 00](LAB-00-MOI-TRUONG-1.35.7.md) **không có** GPU và
> **không có** thiết bị chuyên dụng, nên phần cấp phát thiết bị thật sẽ không chạy được. Đó không
> phải lý do bỏ lab: [B1](#b1-kiểm-tra-năng-lực-dra--chọn-nhánh) đo năng lực thật của cluster rồi rẽ
> nhánh, và nhánh dành cho cluster không có thiết bị vẫn kiểm chứng được phần lớn nhóm bài.
> **Lab trước:** [Lab 12 — Vận hành vòng đời node](LAB-12-VAN-HANH-VONG-DOI-NODE.md), cũng trả
> cluster về `04-metrics-ready`.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục
[Giai đoạn 13 — Lập lịch và workload nâng cao](../00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao)
— mười lăm bài [149](../149-dynamic-resource-allocation-vi.md),
[172](../172-cluster-admin-dra-vi.md), [125](../125-hardening-dra-vi.md),
[59](../59-scheduling-group-vi.md), [75](../75-podgroup-api-vi.md),
[76](../76-podgroup-lifecycle-vi.md), [77](../77-workload-api-vi.md),
[78](../78-workload-disruption-priority-vi.md), [79](../79-workload-policies-vi.md),
[80](../80-workload-topology-scheduling-vi.md), [150](../150-gang-scheduling-vi.md),
[151](../151-podgroup-scheduling-vi.md), [152](../152-workload-aware-preemption-vi.md),
[153](../153-topology-aware-scheduling-vi.md), [124](../124-hardening-scheduler-vi.md), cộng năm
bài thực hành [271](../271-set-up-dra-cluster-vi.md), [270](../270-allocate-devices-dra-vi.md),
[269](../269-access-dra-device-metadata-vi.md), [268](../268-assign-resources-vi.md),
[211](../211-hardening-dra-tasks-vi.md).

**Xương sống của lab là một câu hỏi và một phép đo.** Câu hỏi: cấp phát thiết bị bằng DRA đi được
tới đâu trên cluster của bạn? Phép đo là [B1](#b1-kiểm-tra-năng-lực-dra--chọn-nhánh) — năm phép
kiểm bằng lệnh, không suy đoán, rồi chốt nhánh. Mười mục còn lại chia làm hai loại: phần **chạy
được ở cả hai nhánh** (bề mặt API, phạm vi từng kind, ranh giới phân quyền, so sánh ba con đường
cấp phát thiết bị, cấu hình thật của scheduler) và phần **chỉ chạy ở một nhánh**.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không chép
lại con số phiên bản nào**. Thành phần ngoài baseline đang chạy — CNI thay Flannel, ingress
controller, dynamic provisioner, metrics-server — đã khóa ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00); lab 13 chỉ đọc, không
đụng vào, và **không thêm bất kỳ dòng nào vào hai bảng đó**.

Quan hệ với ba lab đã học, để không lặp lại:
[Lab 3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md#b5-extended-resource-tài-nguyên-do-bạn-tự-đặt-tên)
đã quảng bá một extended resource lên `lab-k8s-worker2` và tiêu thụ nó thật — đó là **con đường
thứ nhất** mà B5 đem ra so sánh, và lab này chỉ đọc lại dấu vết chứ không làm lại.
[Lab 7a](LAB-7A-LAP-LICH-VA-EVICTION.md#b6-priorityclass-và-preemption) đã làm trọn chu trình
filter/score, taint, topology spread, PriorityClass và preemption mặc định; lab 13 **không lặp lại
phần lập lịch cơ bản**, chỉ đọc những gì nhóm PodGroup *thêm vào* trên nền đó.
[Lab 11a](LAB-11A-OBSERVABILITY.md#b3-đọc-metric-là-một-hành-động-được-ủy-quyền) đã dựng đường đọc
`/metrics` có ủy quyền; ở đây bạn chỉ dùng lại đường đã có, không dựng ClusterRole mới.

Lab dùng Namespace, Pod, ServiceAccount, `kubectl auth can-i`, `kubectl explain`, `kubectl
api-resources` và node proxy của các giai đoạn đã học làm công cụ. **Không** tạo CRD tự viết — đó
là nội dung [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) và của Lab 14
(chưa viết, xem [bản đồ lab](README.md#4-bản-đồ-lab)); đọc CRD có sẵn thì được, và B0 đếm chúng để
gate cuối chứng minh lab không cài thêm cái nào.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`, rồi thêm ba lệnh riêng của lab này:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Ba lenh rieng cua lab 13: dung moc 04-metrics-ready, chua co gi cua lab 13 trong cluster.
kubectl top node
kubectl get namespace lab-13 --ignore-not-found
kubectl get resourceclaims -A 2>/dev/null || echo '(cluster khong co kieu resourceclaims)'
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod; **`kubectl top node` in đủ ba dòng số
liệu** (đó là định nghĩa của mốc `04-metrics-ready`); lệnh thứ hai không in gì — chưa có namespace
`lab-13`; lệnh thứ ba in `No resources found` **hoặc** dòng `(cluster khong co kieu
resourceclaims)` — **cả hai đều hợp lệ ở bước này**, B1 mới là chỗ kết luận.

Nếu `kubectl top node` báo lỗi thì cluster chưa ở mốc đầu vào — restore cả ba VM về
`04-metrics-ready` trước khi tiếp tục. Nếu `kubectl get resourceclaims -A` trả về một ResourceClaim
nào đó thì cluster đã lệch mốc: mốc `04-metrics-ready` không có object DRA nào.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Đọc **năng lực DRA thật** của cluster mình — nhóm API `resource.k8s.io` có được phục vụ không, có
  DeviceClass nào, có ResourceSlice nào, driver nào đang công bố thiết bị, feature gate liên quan
  đang ở trạng thái gì — và kết luận DRA dùng được hay không **bằng bằng chứng chứ không bằng phỏng
  đoán**, kể cả khi kết luận là "không".
- Bốn kind API của DRA, ai tạo cái nào, và phạm vi của từng cái đọc từ chính API của cluster:
  DeviceClass và ResourceSlice cluster-scoped, ResourceClaim và ResourceClaimTemplate
  namespace-scoped — và vì sao đúng ranh giới đó là ranh giới phân quyền mà bài 172 khuyến nghị.
- Vì sao một DeviceClass mới tạo **không sinh ra thiết bị nào**, và ai mới là bên công bố phần
  cứng.
- Chuyện gì xảy ra với một Pod tham chiếu ResourceClaim khi (a) claim chưa tồn tại và (b) claim tồn
  tại nhưng cluster không có ResourceSlice nào khớp — phân biệt hai tình huống bằng trạng thái
  object chứ không bằng cảm giác.
- **DRA khác device plugin và khác extended resource ở đâu** — ba con đường đưa một thiết bị vào
  Pod, ba chỗ khác nhau trong Pod spec, ba nguồn số liệu khác nhau trong cluster, và bạn đọc được
  cả ba con đường đó đang trống hay đầy trên cluster của mình.
- Nhóm PodGroup/Workload cần chính xác những gì mới dùng được, và cách đo xem apiserver của bạn có
  *biết tên* các feature gate đó hay không — thứ trả lời được câu "API có thể đổi giữa các phiên
  bản" bằng dữ liệu.
- Đối chiếu cấu hình kube-scheduler **đang chạy** với từng khuyến nghị của bài 124, biết cái nào
  baseline đã đạt, cái nào không, và vì sao "không đạt" ở đây không phải lỗi mà là một quyết định
  thuộc giai đoạn khác.
- Mô hình phân quyền DRA của bài 125 và 211 ở mức đọc được: hai subresource tổng hợp không xuất
  hiện trong `api-resources` nhưng có thật ở tầng ủy quyền, và quyền thật mà scheduler đang có trên
  `resourceclaims`.
- Cleanup đúng phạm vi: xóa cả object phạm vi cluster, và chứng minh cluster trở về đúng
  `04-metrics-ready` bằng những phép **so sánh giá trị trước/sau** — trong đó có phép so chứng minh
  lab **không cài thêm API resource, CRD hay feature gate nào**.

### Nhánh A hay nhánh B — quyết định bằng B1

[B1](#b1-kiểm-tra-năng-lực-dra--chọn-nhánh) chạy năm phép kiểm và đặt biến `NHANH`:

| Nhánh | Khi nào | B3 và B4 làm gì | Kết luận |
| --- | --- | --- | --- |
| **A** | API `resource.k8s.io` có, **và** có ít nhất một ResourceSlice, **và** có ít nhất một driver công bố thiết bị, **và** có ít nhất một DeviceClass sẵn có | đọc ResourceSlice thật, cấp phát thiết bị cho Pod qua ResourceClaim rồi qua ResourceClaimTemplate, quan sát scheduler ghi kết quả cấp phát | DRA **dùng được**, checkpoint trả lời từ dữ liệu thật |
| **B** | thiếu ít nhất một điều kiện | ghi hồ sơ "DRA chưa dùng được" kèm bằng chứng, và kiểm chứng phần khái niệm/ranh giới vẫn chạy được | DRA **chưa dùng được**, và bạn nói được thiếu chính xác cái gì |

Nhánh B **không phải thất bại của lab**. Nó là kết quả trung thực của ba VM không có phần cứng
chuyên dụng — đúng điều kiện mà bản đồ lab đã ghi từ đầu. Điều bị cấm là làm ngược lại: cài một DRA
driver, bật feature gate, hay tự tạo ResourceSlice để "có dữ liệu". Cả ba việc đó đổi hạ tầng hoặc
dựng thiết bị giả, mà lab này khai báo trả cluster về `04-metrics-ready` và **không chụp mốc mới**.

**Trước khi làm B3 và B4, đọc lại [bài 149](../149-dynamic-resource-allocation-vi.md)** — cụ thể là
hai mục *Thuật ngữ DRA* và *Luồng công việc của Kubernetes*, vì mọi thứ B3 và B4 kiểm nằm ở đó — và
[bài 271](../271-set-up-dra-cluster-vi.md) mục *Xác minh rằng DRA đã được bật*, vì đó chính là phép
kiểm đầu tiên của B1.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm giai đoạn 13 | Kiểm chứng ở |
| --- | --- |
| [149 — Cấp phát tài nguyên động](../149-dynamic-resource-allocation-vi.md) | B1 (năm phép kiểm năng lực), B2 (bốn kind API, ai tạo cái nào, phạm vi), B3 (DeviceClass là danh mục, ResourceSlice là nguồn cung, driver ghi đè sửa tay), B4 (luồng ResourceClaim → Pod; nhánh A cấp phát thật, nhánh B dừng đúng ở bước *Lọc ResourceSlice*), B5 (ba điểm bài nêu device plugin không làm được) |
| [172 — Thực hành tốt về DRA cho quản trị viên](../172-cluster-admin-dra-vi.md) | B2.3 (ranh giới phân quyền: hai API cluster-scoped cho admin/driver, hai API namespace-scoped cho người triển khai Pod), B5.3 (metric `dra_operations_duration_seconds` của kubelet — nơi `NodePrepareResources`/`NodeUnprepareResources` hiện ra), B8.2 (controller ResourceClaim nằm trong kube-controller-manager) |
| [125 — Hardening DRA](../125-hardening-dra-vi.md) | B10 — hai subresource tổng hợp `resourceclaims/binding` và `resourceclaims/driver` không có trong `api-resources` nhưng trả lời được ở tầng ủy quyền; verb có tiền tố nhận biết node; rule thật trong ClusterRole `system:kube-scheduler` |
| [59 — Nhóm lập lịch](../59-scheduling-group-vi.md) | B6.2 — trường `spec.schedulingGroup` có trong schema Pod của cluster này không, và phép kiểm chéo với sự tồn tại của kind PodGroup |
| [75 — PodGroup API](../75-podgroup-api-vi.md) | B6.1 — `podgroups` trong `api-resources`, và nhóm `scheduling.k8s.io` đang phục vụ những version nào |
| [76 — Vòng đời của PodGroup](../76-podgroup-lifecycle-vi.md) | B6.1 — cùng phép kiểm; phần finalizer, `ownerReferences` và thứ tự tạo ở bảng lý do bên dưới |
| [77 — Workload API](../77-workload-api-vi.md) | B6.1 (`workloads` trong `api-resources`), B6.3 (apiserver có biết tên gate `GenericWorkload` và `WorkloadWithJob` không) |
| [78 — Gián đoạn và độ ưu tiên của PodGroup](../78-workload-disruption-priority-vi.md) | B7.1 (PriorityClass thật của cluster và câu hỏi `globalDefault`), B7.2 (`priorityClassName`/`disruptionMode` ở cấp nhóm có trong schema không) |
| [79 — Các chính sách lập lịch PodGroup](../79-workload-policies-vi.md) | B6.2 — `basic` và `gang` chỉ tồn tại bên trong schema PodGroup; không có schema thì không có chỗ nào trong cluster nhận hai giá trị đó |
| [80 — Lập lịch nhận biết topology (Workload API)](../80-workload-topology-scheduling-vi.md) | B8.1 — ba node đang mang nhãn topology nào, và `schedulingConstraints` đọc được ở đâu |
| [150 — Lập lịch theo nhóm](../150-gang-scheduling-vi.md) | B6.3 — gate `GangScheduling` và `GenericWorkload`, cộng nhóm API `scheduling.k8s.io/v1alpha2` |
| [151 — Lập lịch PodGroup](../151-podgroup-scheduling-vi.md) | B8.2 — thuật toán theo placement chạy bằng hai điểm mở rộng plugin; scheduler của cluster đang nạp cấu hình nào |
| [152 — Preemption nhận biết workload](../152-workload-aware-preemption-vi.md) | B7 — đường preemption mặc định vẫn là đường duy nhất của cluster này; gate `WorkloadAwarePreemption` |
| [153 — Lập lịch nhận biết topology (scheduling)](../153-topology-aware-scheduling-vi.md) | B8.2 — không nạp `KubeSchedulerConfiguration` thì không cấu hình được plugin `TopologyPlacement`, `NodeResourcesFit` ở điểm `placementScore`, `PodGroupPodsCount` |
| [124 — Hardening scheduler](../124-hardening-scheduler-vi.md) | B9 — mười hai cờ đọc từ manifest thật của kube-scheduler, quyền file của `authentication-kubeconfig`, và quyền gán nhãn node |
| [271 — Thiết lập DRA trong một cluster](../271-set-up-dra-cluster-vi.md) | B1.1 (`kubectl get deviceclasses` đúng là phép xác minh mà bài này quy định), B3.2 (tạo DeviceClass bằng selector CEL), B11 (mục *Dọn dẹp* của chính bài) |
| [270 — Cấp phát thiết bị cho workload bằng DRA](../270-allocate-devices-dra-vi.md) | B4 — ResourceClaim so với ResourceClaimTemplate, `pod.spec.resourceClaims` và `resources.claims` của container, và chuyện gì xảy ra khi claim được tham chiếu không tồn tại |
| [269 — Truy cập metadata thiết bị DRA](../269-access-dra-device-metadata-vi.md) | B5.4 — thư mục `/var/run/kubernetes.io/dra-device-attributes` chỉ có mặt khi container thật sự yêu cầu thiết bị; nhánh A đọc file JSON tại đó |
| [268 — Gán thiết bị cho Pod và Container](../268-assign-resources-vi.md) | Trang mục lục, không có thao tác riêng: ba trang con 271, 270 và 269 phủ toàn bộ nội dung và được kiểm ở B1, B3, B4, B5 |
| [211 — Tăng cường bảo mật DRA trong cluster của bạn](../211-hardening-dra-tasks-vi.md) | B10 — bước *Xác định các thành phần DRA ghi vào status* làm được ngay, bước *Kiểm chứng* làm bằng `kubectl auth can-i` cho từng danh tính |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài [149](../149-dynamic-resource-allocation-vi.md), mục *Hạn chế* — scheduler không preempt cho tài nguyên DRA | Cần **hai** Pod tranh nhau một thiết bị thật, tức là cần ít nhất một thiết bị đã cấp phát. Không thiết bị thì không có tranh chấp để quan sát, và không có cách nào tạo tranh chấp mà không dựng thiết bị giả |
| Bài [149](../149-dynamic-resource-allocation-vi.md), toàn bộ *Các tính năng beta của DRA* và *Các tính năng alpha của DRA* — admin access, thiết bị phân vùng, dung lượng tiêu thụ được, taint/toleration thiết bị và DeviceTaintRule, điều kiện gắn kết, tài nguyên node allocatable, thuộc tính kiểu danh sách, trạng thái resource pool | Mỗi mục phụ thuộc một feature gate riêng và/hoặc một nhóm API beta/alpha riêng, cộng một driver công bố ResourceSlice có đúng cấu trúc. Bật gate là sửa cấu hình control plane — việc đó thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), không phải lab này |
| Bài [149](../149-dynamic-resource-allocation-vi.md), mục *Khả năng quan sát tài nguyên động* — `PodResourcesLister`, `status.devices`, giám sát sức khỏe thiết bị | Cả ba đều là dữ liệu **do driver ghi**. Không driver thì ba nguồn này rỗng theo định nghĩa; B4 chứng minh `status` của một ResourceClaim rỗng và dừng đúng ở đó |
| Bài [172](../172-cluster-admin-dra-vi.md), các truy vấn PromQL và phần tinh chỉnh QPS/burst | Cần stack giám sát cộng tải thật để có dữ liệu; endpoint `/metrics` của kube-controller-manager bind `127.0.0.1` và cách đọc có ủy quyền đã làm trọn ở [Lab 11a B3](LAB-11A-OBSERVABILITY.md#b3-đọc-metric-là-một-hành-động-được-ủy-quyền). Lặp lại ở đây chỉ để in ra các histogram rỗng thì không phải kiểm chứng |
| Bài [172](../172-cluster-admin-dra-vi.md), nâng cấp liền mạch, liveness probe của driver, thứ tự drain driver | Ba việc đều thao tác trên **một DaemonSet driver đang chạy**. Cluster không có driver nào để nâng cấp, thăm dò hay drain |
| Bài [125](../125-hardening-dra-vi.md) và [211](../211-hardening-dra-tasks-vi.md), áp ba ClusterRole ví dụ và roll out các thành phần DRA | Ba role đó chỉ có nghĩa khi gắn vào ServiceAccount của một driver thật; tạo role trống rồi gắn vào một danh tính không tồn tại là diễn, không phải kiểm chứng. B10 đọc quyền **thật sự đang có** thay vì tạo quyền mới |
| Bài [211](../211-hardening-dra-tasks-vi.md), theo dõi sự kiện audit của API server | Cluster baseline chưa bật audit policy; bật nó thuộc [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu) |
| Bài [59](../59-scheduling-group-vi.md), [75](../75-podgroup-api-vi.md), [76](../76-podgroup-lifecycle-vi.md), [77](../77-workload-api-vi.md), [78](../78-workload-disruption-priority-vi.md), [79](../79-workload-policies-vi.md), [80](../80-workload-topology-scheduling-vi.md), [150](../150-gang-scheduling-vi.md), [151](../151-podgroup-scheduling-vi.md), [152](../152-workload-aware-preemption-vi.md), [153](../153-topology-aware-scheduling-vi.md) — toàn bộ phần **chạy thật** | Cả nhóm phụ thuộc feature gate `GenericWorkload` (và các gate kèm theo) cộng nhóm API alpha `scheduling.k8s.io/v1alpha2`. Bật hai thứ này là sửa cấu hình control plane, lệch baseline, và lab này khai báo trả cluster về `04-metrics-ready`. B6, B7 và B8 kiểm đúng **phần đọc được**: bề mặt API, schema và tên gate mà apiserver biết |
| Bài [124](../124-hardening-scheduler-vi.md), đoạn `KubeSchedulerConfiguration` tắt plugin theo điểm mở rộng | Cần chạy một scheduler tùy chỉnh. Cluster chạy đúng một kube-scheduler mặc định, và thay cấu hình của nó là sửa manifest control plane |
| Bài [270](../270-allocate-devices-dra-vi.md) và [269](../269-access-dra-device-metadata-vi.md), phần cấp phát thiết bị thật và đọc file metadata JSON | Chỉ chạy ở nhánh A. Ở nhánh B, B4 dừng đúng tại điểm mà tài liệu nói nó phải dừng, và B5.4 kiểm chứng vế phủ định: không yêu cầu thiết bị thì không có thư mục metadata |

### 1.2. Thời lượng

2–3 giờ, tính từ lúc gate mở đầu đã PASS. Nhánh A dài hơn nhánh B khoảng nửa giờ. B4 có bước phải
chờ controller và scheduler phản ứng; thời gian chờ **phụ thuộc cấu hình cluster** nên mọi bước chờ
đều viết dưới dạng vòng lặp có điều kiện dừng, không phải con số cố định.

---

## 2. Quy ước và an toàn

- Mọi lệnh chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, **trừ khi ghi rõ node
  khác**. Lab này không cần SSH sang node nào; các lệnh `sudo` đều chạy trên chính `lab-k8s-master`
  và đều là lệnh **đọc**.
- **Chạy toàn bộ phần B trong cùng một phiên shell.** Rất nhiều gate so sánh biến đặt ở bước trước
  (`EV`, `WK`, `MASTER`, `W2`, `DRA_API`, `NHANH`, `DC_BEFORE`, `SLICE_BEFORE`, `CRD_BEFORE`,
  `API_BEFORE`, `FG_ON_BEFORE`, `NS_BEFORE`, `LBL_BEFORE`…); mở shell mới giữa chừng là mất biến và
  mất luôn gate cuối.
- **Lab tuyệt đối không cài và không sửa gì:** không DRA driver, không device plugin, không CRD,
  không Helm chart, không bật feature gate, không sửa file trong `/etc/kubernetes/manifests`, không
  sửa cấu hình kubelet, không gán nhãn node, không tạo ResourceSlice bằng tay. Thiếu điều kiện thì
  ghi nhận thiếu, không đi vá.
- Lab tạo Namespace `lab-13` và các object bên trong nó, cộng **một object phạm vi cluster**:
  DeviceClass `lab-13-example` (chỉ khi nhóm API `resource.k8s.io` được phục vụ). Cả hai bị xóa ở
  B11, và gate cuối so số lượng trước/sau để chứng minh điều đó.
- **Không hardcode tên nhóm API hay tên kind vào kết luận.** B0 và B1 đọc chúng từ cluster thật;
  mọi mục sau dùng biến `DRA_API` và `NHANH`. Nếu một manifest bị API server từ chối vì
  `apiVersion`, đọc cột `APIVERSION` trong `~/lab-evidence/13/b1-api-resource.txt` và sửa cho khớp
  — **không đoán**.
- Image dùng cho toàn bộ lab là `busybox:1.37` đã khóa ở
  [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và đã có sẵn trên cả ba node từ
  Lab 00, nên phần B không phụ thuộc mạng ra ngoài.
- Lab này **không có fault injection**: không ép node vào áp lực, không giết Pod hệ thống, không
  phá mạng. Nếu bạn tự thêm bước phá hoại nào, quy ước chung vẫn áp dụng — chỉ chạy trên
  `lab-k8s-worker2`.
- Manifest tạm ghi vào `~/lab-work/13/`; bằng chứng ghi vào `~/lab-evidence/13/`. Không lưu token,
  key hay certificate.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng bắt đầu bằng `STOP:` nghĩa là
  dừng đúng bước đó và ghi lại, không đi vòng. Dòng bắt đầu bằng `GHI NHAN:` là kết quả **hợp lệ
  nhưng khác mặc định** — chép nguyên văn vào evidence rồi đọc lại phần *Ý nghĩa* trước khi kết
  luận.
- **Cách quay lui khi hỏng:** tắt **cả ba VM**, restore **cả ba** về `04-metrics-ready` — không bao
  giờ restore riêng một VM, xem ghi chú cuối
  [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường) — rồi bật lại theo
  thứ tự master → worker 1 → worker 2, chạy lại gate mở đầu và bắt đầu lại từ B0.

---

# Phần B — Thực hành kiến thức giai đoạn 13

## B0. Chuẩn bị workspace và ảnh nền của cluster

**Mục đích:** chốt các con số nền để B11 có cái đối chiếu. Lab này khẳng định "không cài gì", và
khẳng định đó chỉ có giá trị khi có số đo trước và số đo sau.

```bash
mkdir -p ~/lab-work/13 ~/lab-evidence/13
EV="$HOME/lab-evidence/13"
WK="$HOME/lab-work/13"

kubectl config current-context
MASTER="$(kubectl get node -l node-role.kubernetes.io/control-plane \
  -o jsonpath='{.items[0].metadata.name}')"
W2='lab-k8s-worker2'
echo "MASTER=$MASTER | W2=$W2"

kubectl create namespace lab-13
kubectl get namespace lab-13 -o jsonpath='{.status.phase}'; echo
```

Chốt các con số nền:

```bash
NODE_N="$(kubectl get nodes --no-headers | wc -l)"
API_BEFORE="$(kubectl api-resources -o name 2>/dev/null | sort -u | wc -l)"
CRD_BEFORE="$(kubectl get crd -o name 2>/dev/null | wc -l)"
NS_BEFORE="$(kubectl get namespace -o name | sort | tr '\n' ' ')"
PC_BEFORE="$(kubectl get priorityclass -o name | sort | tr '\n' ' ')"
LBL_BEFORE="$(kubectl get nodes --no-headers --show-labels | awk '{print $1, $NF}' | sort | md5sum)"

{
  echo "=== $(date -Is) — anh nen truoc Lab 13 ==="
  echo "MASTER=$MASTER NODE_N=$NODE_N"
  echo "API_BEFORE=$API_BEFORE CRD_BEFORE=$CRD_BEFORE"
  echo "NS_BEFORE=$NS_BEFORE"
  echo "PC_BEFORE=$PC_BEFORE"
  echo "LBL_BEFORE=$LBL_BEFORE"
} | tee "$EV/b0-anh-nen.txt"
```

Chụp bảng feature gate của apiserver — đây là ảnh nền quan trọng nhất của lab, vì nó là thứ chứng
minh cuối lab rằng bạn **không bật gate nào**:

```bash
kubectl get --raw '/metrics' > "$EV/b0-apiserver-metrics.txt"
FG_ALL_BEFORE="$(grep -c 'kubernetes_feature_enabled{' "$EV/b0-apiserver-metrics.txt")"
FG_ON_BEFORE="$(grep 'kubernetes_feature_enabled{' "$EV/b0-apiserver-metrics.txt" | grep -c ' 1$')"
echo "FG_ALL_BEFORE=$FG_ALL_BEFORE | FG_ON_BEFORE=$FG_ON_BEFORE" | tee -a "$EV/b0-anh-nen.txt"

test "$NODE_N" -eq 3 && echo 'PASS: cluster du ba node'
test -s "$EV/b0-apiserver-metrics.txt" && echo 'PASS: doc duoc /metrics cua apiserver'
test "$FG_ALL_BEFORE" -gt 0 \
  && echo "PASS: apiserver cong bo bang feature gate — $FG_ALL_BEFORE gate, dang bat $FG_ON_BEFORE"
```

**Ý nghĩa:** `kubernetes_feature_enabled` là bảng feature gate **do chính apiserver công bố**: mỗi
dòng là một gate mà binary đó biết tên, kèm giá trị `1` (đang bật) hoặc `0` (đang tắt). Ba con số
`API_BEFORE`, `CRD_BEFORE`, `FG_ON_BEFORE` là ba trục mà B11 dùng để chứng minh lab không mở rộng
bề mặt API của cluster theo bất kỳ cách nào.

**PASS:** namespace `lab-13` ở phase `Active`; ba dòng `PASS:` của bước này xuất hiện; file
`b0-anh-nen.txt` có đủ sáu dòng biến. Nếu `FG_ALL_BEFORE` bằng 0, xem mục 4 — B1 vẫn chạy được
bằng đường đọc manifest và `configz`, nhưng bạn phải ghi lại điều đó.

---

## B1. Kiểm tra năng lực DRA — chọn nhánh

**Mục đích:** trả lời một câu hỏi duy nhất bằng bằng chứng: **DRA có dùng được trên cluster này
không?** Bài [271](../271-set-up-dra-cluster-vi.md) mục *Xác minh rằng DRA đã được bật* và mục *Cài
đặt driver thiết bị* đưa ra đúng hai lệnh xác minh; mục này biến chúng cộng ba phép kiểm nữa thành
một quyết định.

### B1.1. Năm phép kiểm

```bash
echo '--- 1. Nhom API resource.k8s.io co duoc phuc vu khong ---'
kubectl api-resources --api-group=resource.k8s.io 2>&1 | tee "$EV/b1-api-resource.txt"

echo '--- 2. DeviceClass: danh muc thiet bi ---'
kubectl get deviceclasses 2>&1 | tee "$EV/b1-deviceclass.txt"

echo '--- 3. ResourceSlice: phan cung da duoc cong bo ---'
kubectl get resourceslices 2>&1 | tee "$EV/b1-resourceslice.txt"

echo '--- 4. Driver nao dang cong bo thiet bi ---'
kubectl get resourceslices \
  -o jsonpath='{range .items[*]}{.spec.driver}{"\n"}{end}' 2>/dev/null \
  | sed '/^$/d' | sort -u | tee "$EV/b1-drivers.txt"
```

**Ý nghĩa của bốn phép kiểm đầu, theo đúng thứ tự tài liệu nêu:**

1. Bài 271 nói thẳng: `kubectl get deviceclasses` trả `No resources found` nghĩa là **cấu hình
   đúng**, còn `error: the server doesn't have a resource type "deviceclasses"` nghĩa là nhóm API
   `resource.k8s.io` đang bị tắt. Phép kiểm 1 nhìn thẳng vào bề mặt API để phân biệt hai tình huống
   đó trước, phép kiểm 2 xác nhận lại bằng đúng lệnh mà bài quy định.
2. **DeviceClass không tự có.** Bài 149 nói rõ nó do quản trị viên **hoặc** driver tạo; cluster
   sạch thì con số là 0, và đó không phải lỗi.
3. **ResourceSlice là thứ duy nhất chứng minh có phần cứng.** Bài 149 mục *Luồng công việc của
   Kubernetes* đặt "tạo ResourceSlice" ở bước 1: không có ResourceSlice thì bước *Lọc ResourceSlice*
   không có gì để lọc, và không Pod nào dùng DRA được lập lịch.
4. Trường `.spec.driver` của ResourceSlice là **chữ ký của driver**. Không dòng nào ở đây nghĩa là
   không driver DRA nào đang đăng ký và công bố thiết bị trong cluster.

### B1.2. Feature gate: đọc, không sửa

**STOP:** mục này **chỉ đọc**. Không thêm `--feature-gates` vào bất kỳ file nào, không sửa file
trong `/etc/kubernetes/manifests`, không sửa cấu hình kubelet.

```bash
for g in DynamicResourceAllocation DRAAdminAccess DRAResourceClaimDeviceStatus \
         DRAExtendedResource DRAPartitionableDevices DRADeviceTaints DRADeviceTaintRules \
         DRAWorkloadResourceClaims DRAListTypeAttributes ResourceHealthStatus; do
  line="$(grep 'kubernetes_feature_enabled{' "$EV/b0-apiserver-metrics.txt" \
          | grep "name=\"$g\"" | head -1)"
  if [ -z "$line" ]; then
    echo "$g : apiserver khong biet ten gate nay"
  else
    echo "$g : $line"
  fi
done | tee "$EV/b1-feature-gates-dra.txt"

test "$(wc -l < "$EV/b1-feature-gates-dra.txt")" -eq 10 \
  && echo 'PASS: doi chieu du muoi feature gate ma nhom bai 13 nhac ten'
```

Đối chiếu với cấu hình đang chạy của control plane và của kubelet — vẫn chỉ đọc:

```bash
sudo grep -n -- '--feature-gates' /etc/kubernetes/manifests/*.yaml 2>/dev/null \
  > "$EV/b1-manifest-gates.txt"
if [ -s "$EV/b1-manifest-gates.txt" ]; then
  echo 'GHI NHAN: co thanh phan control plane khai --feature-gates — doc file truoc khi ket luan'
  cat "$EV/b1-manifest-gates.txt"
else
  echo 'PASS: khong thanh phan control plane nao khai --feature-gates — dang dung mac dinh baseline'
fi

kubectl get --raw "/api/v1/nodes/$W2/proxy/configz" > "$EV/b1-configz-$W2.json" 2>/dev/null || true
if grep -q '"featureGates"' "$EV/b1-configz-$W2.json" 2>/dev/null; then
  echo "GHI NHAN: kubelet tren $W2 co khai featureGates"
  grep -o '"featureGates":{[^}]*}' "$EV/b1-configz-$W2.json"
else
  echo "PASS: kubelet tren $W2 khong khai featureGates rieng"
fi
```

**Ý nghĩa:** ba nguồn này trả lời ba câu hỏi khác nhau. Bảng metric cho biết **apiserver biết tên
gate nào và đang bật cái nào**; manifest cho biết **quản trị viên có ép gate nào không**; `configz`
cho biết **kubelet có gate riêng không**. Ba nguồn phải nhất quán, và nếu lệch thì bạn có đúng file
để đọc.

Bài 149 ghi DRA là `stable`, còn gần như mọi mục con của nó vẫn alpha/beta và phụ thuộc gate riêng.
Bảng bạn vừa in ra là cách duy nhất để biết **binary trong cluster của bạn** đứng ở đâu trong bức
tranh đó — thay vì đọc trạng thái tính năng trong tài liệu rồi suy ra.

**PASS:** dòng `PASS: doi chieu du muoi feature gate…` xuất hiện; hai phép kiểm còn lại in ra một
dòng `PASS:` hoặc một dòng `GHI NHAN:` và nội dung được ghi vào evidence.

### B1.3. Tính toán và chốt nhánh

```bash
DRA_API=0
kubectl api-resources --api-group=resource.k8s.io -o name 2>/dev/null \
  | grep -qx 'deviceclasses.resource.k8s.io' && DRA_API=1

DC_BEFORE=0
SLICE_BEFORE=0
if [ "$DRA_API" -eq 1 ]; then
  DC_BEFORE="$(kubectl get deviceclasses --no-headers 2>/dev/null | wc -l)"
  SLICE_BEFORE="$(kubectl get resourceslices --no-headers 2>/dev/null | wc -l)"
fi
DRV_N="$(sed '/^$/d' "$EV/b1-drivers.txt" 2>/dev/null | wc -l)"
DRA_GATE="$(grep '^DynamicResourceAllocation :' "$EV/b1-feature-gates-dra.txt")"

if [ "$DRA_API" -eq 1 ] && [ "$SLICE_BEFORE" -ge 1 ] \
   && [ "$DRV_N" -ge 1 ] && [ "$DC_BEFORE" -ge 1 ]; then
  NHANH=A
else
  NHANH=B
fi

{
  echo "=== $(date -Is) — kiem tra nang luc DRA ==="
  echo "1. nhom API resource.k8s.io duoc phuc vu : DRA_API=$DRA_API"
  echo "2. so DeviceClass co san                 : $DC_BEFORE"
  echo "3. so ResourceSlice                      : $SLICE_BEFORE"
  echo "4. so driver dang cong bo thiet bi       : $DRV_N"
  echo "5. gate DynamicResourceAllocation        : $DRA_GATE"
  echo "NHANH                                    : $NHANH"
} | tee "$EV/b1-nang-luc.txt"

echo "$NHANH" > "$EV/b1-nhanh.txt"

if [ "$NHANH" = A ]; then
  echo 'NHANH A: du bon dieu kien — B3 va B4 cap phat thiet bi that'
else
  echo 'NHANH B: thieu it nhat mot dieu kien — khong cap phat gi, ghi ho so kem bang chung'
fi
```

**PASS:** in ra **đúng một** dòng bắt đầu bằng `NHANH A:` hoặc `NHANH B:`; file `b1-nang-luc.txt`
có đủ năm dòng đánh số cộng dòng `NHANH`; file `b1-nhanh.txt` chứa đúng một ký tự `A` hoặc `B`. Năm
con số trong file phải khớp với những gì B1.1 và B1.2 in ra.

**STOP:** không cài DRA driver, không bật feature gate, không tạo ResourceSlice bằng tay để ép sang
nhánh A. Nhánh B là kết quả hợp lệ và là lý do lab 13 được đánh dấu **tùy chọn** ngay từ bản đồ
lab. Muốn có thiết bị thật thì đó phải là một lab khác, chạy trên phần cứng khác, và tạo mốc
snapshot của riêng nó.

---

## B2. Bốn kind API của DRA: phạm vi, schema và ranh giới phân quyền

**Mục đích:** kiểm chứng mục *Thuật ngữ DRA* của bài [149](../149-dynamic-resource-allocation-vi.md)
và mục *Tách quyền truy cập các API liên quan đến DRA* của bài
[172](../172-cluster-admin-dra-vi.md). Cả hai nói về **API**, không về dữ liệu, nên mục này chạy ở
**cả hai nhánh** — miễn là `DRA_API` bằng 1.

Nếu `DRA_API=0`, **STOP:** ghi lại kết quả B1 và bỏ qua B2 tới B5 phần DRA; đi thẳng tới
[B5.2](#b52-node-status-không-biết-gì-về-dra) và các mục sau, rồi nói rõ ở checkpoint rằng bề mặt
API DRA không tồn tại trên cluster của bạn.

### B2.1. Phạm vi từng kind, đọc từ chính API

```bash
if [ "$DRA_API" -eq 1 ]; then
  kubectl api-resources --api-group=resource.k8s.io -o wide 2>&1 | tee "$EV/b2-api-wide.txt"

  kubectl api-resources --namespaced=false -o name 2>/dev/null \
    | grep -qx 'deviceclasses.resource.k8s.io' \
    && echo 'PASS: deviceclasses — cluster-scoped'
  kubectl api-resources --namespaced=false -o name 2>/dev/null \
    | grep -qx 'resourceslices.resource.k8s.io' \
    && echo 'PASS: resourceslices — cluster-scoped'
  kubectl api-resources --namespaced=true -o name 2>/dev/null \
    | grep -qx 'resourceclaims.resource.k8s.io' \
    && echo 'PASS: resourceclaims — namespace-scoped'
  kubectl api-resources --namespaced=true -o name 2>/dev/null \
    | grep -qx 'resourceclaimtemplates.resource.k8s.io' \
    && echo 'PASS: resourceclaimtemplates — namespace-scoped'
fi
```

**Ý nghĩa:** bốn dòng này là **nửa bên trái** của phép ánh xạ mà bài 149 dựng: DeviceClass đứng ở
vị trí StorageClass, ResourceClaim đứng ở vị trí PersistentVolumeClaim. Phạm vi khớp y hệt —
StorageClass cluster-scoped, PVC namespace-scoped — và đó không phải trùng hợp: **object mô tả
nguồn cung của cluster thì thuộc về cluster, object là yêu cầu của người dùng thì thuộc về
namespace**.

Nửa bên phải là câu hỏi ai tạo cái nào, và nó quyết định luôn ranh giới phân quyền ở B2.3.

**PASS:** đúng bốn dòng `PASS:` của bước này xuất hiện.

### B2.2. Schema: bốn kind, bốn hình dạng

```bash
if [ "$DRA_API" -eq 1 ]; then
  OK=0
  for p in deviceclass.spec \
           resourceslice.spec \
           resourceclaim.spec.devices.requests \
           resourceclaim.status \
           resourceclaimtemplate.spec.spec \
           pod.spec.resourceClaims \
           pod.spec.containers.resources.claims; do
    f="$EV/b2-explain-$(echo "$p" | tr '.' '-').txt"
    if kubectl explain "$p" > "$f" 2>&1 && [ -s "$f" ]; then
      OK=$(( OK + 1 ))
    else
      echo "GHI NHAN: khong doc duoc schema $p"
    fi
  done
  echo "doc duoc $OK/7 schema"
  test "$OK" -eq 7 && echo 'PASS: doc duoc du bay duong dan schema cua DRA'
fi
```

**Ý nghĩa:** bảy đường dẫn này là toàn bộ bề mặt mà một quản trị viên chạm vào. Hai đường cuối là
chỗ dễ nhầm nhất và sẽ quay lại ở B5: `pod.spec.resourceClaims` là nơi Pod **khai** nó dùng những
claim nào, còn `pod.spec.containers.resources.claims` là nơi **từng container** nói nó dùng claim
nào trong số đó. Hai trường khác cấp, và bài 270 dùng đúng cặp này để cho một container dùng riêng
một claim còn hai container khác chia sẻ một claim thứ hai.

`resourceclaim.status` là chỗ B4 sẽ đọc: `status.allocation` và `status.reservedFor` — đúng hai
trường mà bài [125](../125-hardening-dra-vi.md) gắn vào subresource tổng hợp
`resourceclaims/binding`.

**PASS:** dòng `PASS: doc duoc du bay duong dan schema…` xuất hiện.

### B2.3. Ranh giới phân quyền của bài 172

Bài [172](../172-cluster-admin-dra-vi.md) khuyến nghị: DeviceClass và ResourceSlice giới hạn cho
quản trị viên và driver; ResourceClaim và ResourceClaimTemplate mở cho người triển khai Pod. Kiểm
xem một danh tính thường có đi qua ranh giới đó không:

```bash
if [ "$DRA_API" -eq 1 ]; then
  SA='system:serviceaccount:lab-13:default'
  {
    echo "danh tinh thu nghiem: $SA"
    echo "create deviceclasses          -> $(kubectl auth can-i create deviceclasses --as="$SA")"
    echo "create resourceslices         -> $(kubectl auth can-i create resourceslices --as="$SA")"
    echo "create resourceclaims (ns)    -> $(kubectl auth can-i create resourceclaims -n lab-13 --as="$SA")"
    echo "list resourceclaimtemplates   -> $(kubectl auth can-i list resourceclaimtemplates -n lab-13 --as="$SA")"
  } | tee "$EV/b2-can-i.txt"

  NO_N="$(grep -c '\-> no$' "$EV/b2-can-i.txt")"
  test "$NO_N" -eq 4 \
    && echo 'PASS: ServiceAccount tran khong dung duoc API DRA nao — ca hai phia ranh gioi'
fi
```

**Ý nghĩa:** cả bốn đều `no`, và đó đúng là điểm xuất phát mà bài 172 giả định: mặc định **không ai
có gì**, việc của quản trị viên là mở đúng hai API namespace-scoped cho đúng namespace của từng
nhóm. Đặc điểm khiến ranh giới đó gọn nằm ngay ở B2.1: hai API dành cho người triển khai Pod đều
namespace-scoped nên cấp quyền theo namespace là đủ, trong khi hai API kia cluster-scoped nên mở ra
là mở cho toàn cluster.

**Lab không tạo Role hay ClusterRole nào để "chữa" bốn dòng `no` này.** Cấp quyền cho một driver
không tồn tại là diễn, không phải kiểm chứng — xem bảng lý do ở mục 1.1.

**PASS:** dòng `PASS: ServiceAccount tran khong dung duoc API DRA nao…` xuất hiện.

---

## B3. DeviceClass là danh mục, ResourceSlice là nguồn cung

**Mục đích:** tách rành mạch hai thứ hay bị gộp làm một. DeviceClass là **danh mục** do người đặt
ra; ResourceSlice là **hàng có thật** do driver công bố. Bài [271](../271-set-up-dra-cluster-vi.md)
đi đúng thứ tự đó: bật DRA → cài driver → kiểm `resourceslices` → mới tạo DeviceClass.

### B3.1. ResourceSlice: ai tạo, cluster này có bao nhiêu

```bash
if [ "$DRA_API" -eq 1 ]; then
  kubectl get resourceslices -o wide 2>&1 | tee "$EV/b3-slices.txt"
  echo "SLICE_BEFORE=$SLICE_BEFORE | DRV_N=$DRV_N"
  test -s "$EV/b3-slices.txt" && echo 'PASS: da ghi lai trang thai ResourceSlice cua cluster'
fi
```

**Ý nghĩa:** bài 149 mô tả ResourceSlice bằng ba mảng thông tin — resource pool, danh sách device,
danh sách node truy cập được — và nói một câu quyết định: driver chạy một controller **đối chiếu
ResourceSlice và ghi đè mọi thay đổi thủ công**. Nghĩa là kể cả khi bạn tạo tay một ResourceSlice
trông y như thật, không có driver đứng sau thì nó là dữ liệu chết, và có driver thì nó bị xóa ngay
lần đối chiếu kế tiếp.

**STOP:** không tạo ResourceSlice bằng tay để "có dữ liệu cho lab". Đó là dựng thiết bị giả, và mọi
kết luận rút ra từ nó đều sai.

**PASS:** dòng `PASS: da ghi lai trang thai ResourceSlice…` xuất hiện.

### B3.2. Tạo một DeviceClass và xem nó khớp gì

Manifest dưới đây là manifest ví dụ của bài [271](../271-set-up-dra-cluster-vi.md), đổi tên driver
và tên class cho khớp quy ước của lab:

```bash
if [ "$DRA_API" -eq 1 ]; then
cat > "$WK/deviceclass.yaml" <<'YAML'
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: lab-13-example
spec:
  selectors:
  - cel:
      expression: |-
        device.driver == "lab-13.example.com"
YAML

kubectl apply -f "$WK/deviceclass.yaml"
kubectl get deviceclass lab-13-example -o yaml | tee "$EV/b3-deviceclass.yaml"
fi
```

> Nếu `apply` báo không nhận ra `apiVersion`, mở `~/lab-evidence/13/b1-api-resource.txt`, đọc cột
> `APIVERSION` của dòng `deviceclasses` và sửa dòng `apiVersion:` cho khớp rồi apply lại. Đừng
> đoán.

```bash
if [ "$DRA_API" -eq 1 ]; then
  DC_NOW="$(kubectl get deviceclasses --no-headers 2>/dev/null | wc -l)"
  SLICE_NOW="$(kubectl get resourceslices --no-headers 2>/dev/null | wc -l)"
  echo "DC_NOW=$DC_NOW (truoc: $DC_BEFORE) | SLICE_NOW=$SLICE_NOW (truoc: $SLICE_BEFORE)"

  test "$DC_NOW" -eq $(( DC_BEFORE + 1 )) \
    && echo 'PASS: DeviceClass da duoc tao — so luong tang dung 1'
  test "$SLICE_NOW" -eq "$SLICE_BEFORE" \
    && echo 'PASS: tao DeviceClass khong sinh them ResourceSlice nao — danh muc khong tao ra hang'
fi
```

**Ý nghĩa:** gate thứ hai là bài học của cả mục. Một DeviceClass là **một bộ selector CEL và một
cái tên**; API server nhận nó mà không hỏi thiết bị nào khớp, y hệt cách nó nhận một StorageClass
trỏ tới provisioner không tồn tại. Bài 149 nói rõ: tham số của DeviceClass "có thể khớp với **không
hoặc nhiều** thiết bị trong các ResourceSlice" — và trên cluster này, con số đó là không.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B3.A1. Đọc ResourceSlice thật *(chỉ nhánh A)*

Chỉ chạy khi `NHANH=A`. Bài [271](../271-set-up-dra-cluster-vi.md) mục *Tạo DeviceClass* dùng đúng
đường này để tìm thuộc tính mà biểu thức CEL có thể chọn:

```bash
if [ "$NHANH" = A ]; then
  SLICE1="$(kubectl get resourceslices -o jsonpath='{.items[0].metadata.name}')"
  kubectl get resourceslice "$SLICE1" -o yaml | tee "$EV/b3-slice-mau.yaml"

  kubectl get resourceslices -o custom-columns=\
'NAME:.metadata.name,NODE:.spec.nodeName,DRIVER:.spec.driver,POOL:.spec.pool.name,DEVICES:.spec.devices[*].name' \
    | tee "$EV/b3-slice-bang.txt"

  DEV_N="$(kubectl get resourceslices \
    -o jsonpath='{range .items[*]}{range .spec.devices[*]}{.name}{"\n"}{end}{end}' \
    | sed '/^$/d' | wc -l)"
  echo "tong so device duoc cong bo = $DEV_N"
  test "$DEV_N" -ge 1 && echo 'PASS: cluster co it nhat mot thiet bi duoc cong bo qua ResourceSlice'
fi
```

**Ý nghĩa:** ba thứ phải đọc ra được từ file YAML vừa lưu, vì bài 149 nói mỗi ResourceSlice **phải**
có đủ chúng: pool nào, những device nào kèm thuộc tính và dung lượng, và node nào truy cập được
(`nodeName` cho một node cụ thể, hoặc `allNodes: true` cho cả cluster). Đây cũng là nguyên liệu cho
biểu thức CEL ở B3.2 — trên nhánh A bạn nên sửa selector của DeviceClass cho khớp driver thật thay
vì giữ `lab-13.example.com`.

**PASS:** dòng `PASS: cluster co it nhat mot thiet bi…` xuất hiện.

---

## B4. ResourceClaim, ResourceClaimTemplate và Pod

**Mục đích:** đi hết luồng của bài [270](../270-allocate-devices-dra-vi.md) tới điểm dừng thật của
cluster này, và ở mỗi điểm dừng nói được **vì sao dừng ở đó** bằng đúng bước nào trong *Luồng công
việc của Kubernetes* của bài 149.

### B4.A1. Cấp phát một thiết bị cho Pod *(chỉ nhánh A)*

```bash
if [ "$NHANH" = A ]; then
  DC_USE="$(kubectl get deviceclasses -o jsonpath='{.items[0].metadata.name}')"
  echo "dung DeviceClass co san: $DC_USE"

cat > "$WK/claim-va-pod.yaml" <<YAML
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: lab-13-claim
  namespace: lab-13
spec:
  devices:
    requests:
    - name: device-0
      exactly:
        deviceClassName: $DC_USE
---
apiVersion: v1
kind: Pod
metadata:
  name: dra-consumer
  namespace: lab-13
spec:
  restartPolicy: Never
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
    resources:
      claims:
      - name: dev
  resourceClaims:
  - name: dev
    resourceClaimName: lab-13-claim
YAML

  kubectl apply -f "$WK/claim-va-pod.yaml"

  for i in $(seq 1 60); do
    ALLOC="$(kubectl -n lab-13 get resourceclaim lab-13-claim \
      -o jsonpath='{.status.allocation.devices.results[0].device}' 2>/dev/null)"
    test -n "$ALLOC" && break
    sleep 5
  done
  kubectl -n lab-13 get resourceclaim lab-13-claim -o yaml | tee "$EV/b4-claim-da-cap-phat.yaml"
  kubectl -n lab-13 get pod dra-consumer -o wide | tee "$EV/b4-pod-consumer.txt"

  test -n "$ALLOC" && echo "PASS: scheduler da cap phat thiet bi '$ALLOC' vao ResourceClaim"

  RES="$(kubectl -n lab-13 get resourceclaim lab-13-claim \
    -o jsonpath='{.status.reservedFor[0].name}')"
  test "$RES" = 'dra-consumer' \
    && echo 'PASS: status.reservedFor ghi dung Pod dang giu claim'

  PHASE="$(kubectl -n lab-13 get pod dra-consumer -o jsonpath='{.status.phase}')"
  echo "phase cua dra-consumer = $PHASE"
fi
```

**Ý nghĩa:** hai gate trên là hai bước cuối của *Luồng công việc của Kubernetes*: scheduler cập
nhật ResourceClaim với chi tiết cấp phát (`status.allocation`), rồi ghi Pod vào `status.reservedFor`
và đặt Pod lên node truy cập được thiết bị. Từ đây trở đi, kubelet và driver trên node mới là bên
cấu hình thiết bị cho container.

**PASS:** hai dòng `PASS:` của bước này xuất hiện và `$ALLOC` không rỗng.

### B4.A2. ResourceClaimTemplate: mỗi Pod một claim riêng *(chỉ nhánh A)*

```bash
if [ "$NHANH" = A ]; then
cat > "$WK/template-va-pod.yaml" <<YAML
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: lab-13-template
  namespace: lab-13
spec:
  spec:
    devices:
      requests:
      - name: device-0
        exactly:
          deviceClassName: $DC_USE
---
apiVersion: v1
kind: Pod
metadata:
  name: dra-template-consumer
  namespace: lab-13
spec:
  restartPolicy: Never
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
    resources:
      claims:
      - name: dev
  resourceClaims:
  - name: dev
    resourceClaimTemplateName: lab-13-template
YAML

  kubectl apply -f "$WK/template-va-pod.yaml"

  for i in $(seq 1 60); do
    GEN="$(kubectl -n lab-13 get pod dra-template-consumer \
      -o jsonpath='{.status.resourceClaimStatuses[0].resourceClaimName}' 2>/dev/null)"
    test -n "$GEN" && break
    sleep 5
  done
  echo "ResourceClaim duoc sinh ra: ${GEN:-<chua co>}"

  test -n "$GEN" \
    && echo 'PASS: controller da sinh ResourceClaim rieng cho Pod tu template'
  kubectl -n lab-13 get resourceclaim "$GEN" \
    -o jsonpath='{.metadata.ownerReferences[0].kind}{" "}{.metadata.ownerReferences[0].name}{"\n"}' \
    | tee "$EV/b4-owner.txt"
  grep -q '^Pod dra-template-consumer$' "$EV/b4-owner.txt" \
    && echo 'PASS: claim sinh ra thuoc so huu cua chinh Pod — song chet theo Pod do'
fi
```

**Ý nghĩa:** đây là khác biệt thực dụng giữa hai cách trong bài 270. ResourceClaim tự tạo thì **bạn
quản vòng đời** và nhiều Pod chia sẻ được; ResourceClaimTemplate thì **Kubernetes sinh một claim cho
mỗi Pod**, gắn `ownerReferences` vào Pod, và xóa claim khi Pod kết thúc. Trường
`status.resourceClaimStatuses` của Pod là chỗ duy nhất tra được tên claim sinh tự động, vì tên đó
không đoán trước được.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B4.B1. Pod tham chiếu một ResourceClaim chưa tồn tại *(chỉ nhánh B)*

Bài 270 nói: *"Nếu ResourceClaim được tham chiếu không tồn tại, Pod sẽ ở trạng thái pending cho đến
khi ResourceClaim được tạo."* Kiểm chứng trực tiếp:

```bash
if [ "$NHANH" = B ] && [ "$DRA_API" -eq 1 ]; then
cat > "$WK/pod-thieu-claim.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: dra-thieu-claim
  namespace: lab-13
spec:
  restartPolicy: Never
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
    resources:
      claims:
      - name: dev
  resourceClaims:
  - name: dev
    resourceClaimName: lab-13-khong-ton-tai
YAML

  kubectl apply -f "$WK/pod-thieu-claim.yaml"

  for i in $(seq 1 24); do
    kubectl -n lab-13 get events \
      --field-selector involvedObject.name=dra-thieu-claim -o name 2>/dev/null \
      | grep -q . && break
    sleep 5
  done

  PHASE1="$(kubectl -n lab-13 get pod dra-thieu-claim -o jsonpath='{.status.phase}')"
  COND1="$(kubectl -n lab-13 get pod dra-thieu-claim \
    -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].status}')"
  NODE1="$(kubectl -n lab-13 get pod dra-thieu-claim -o jsonpath='{.spec.nodeName}')"
  kubectl -n lab-13 describe pod dra-thieu-claim | tee "$EV/b4-thieu-claim.txt"
  echo "phase=$PHASE1 | PodScheduled=${COND1:-<chua co>} | nodeName=${NODE1:-<chua gan>}"

  test "$PHASE1" = 'Pending' \
    && echo 'PASS: Pod tham chieu ResourceClaim khong ton tai nam Pending'
  test -z "$NODE1" \
    && echo 'PASS: Pod chua duoc gan node nao — no dung o buoc lap lich, khong phai buoc chay'
  if [ "$COND1" = 'False' ]; then
    echo 'PASS: condition PodScheduled=False — scheduler da xet va tu choi'
  else
    echo "GHI NHAN: PodScheduled=${COND1:-<chua co>} — chep nguyen van describe vao evidence"
  fi
fi
```

**Ý nghĩa:** ba tín hiệu phải đọc cùng nhau. `Pending` một mình không nói được gì — Pod đang kéo
image cũng `Pending`. `spec.nodeName` rỗng mới là bằng chứng Pod chưa qua được bước lập lịch, và
`PodScheduled=False` là chữ ký của scheduler nói rằng nó đã xét và không xếp được. Hành vi này
giống hệt một Pod tham chiếu PersistentVolumeClaim không tồn tại — đúng phép so sánh mà bài 149
dựng ngay ở mục đầu.

```bash
if [ "$NHANH" = B ] && [ "$DRA_API" -eq 1 ]; then
  kubectl -n lab-13 delete pod dra-thieu-claim --wait=true
fi
```

**PASS:** hai dòng `PASS:` đầu xuất hiện; dòng thứ ba là `PASS:` hoặc `GHI NHAN:` và nội dung đã
vào evidence.

### B4.B2. Claim có thật, thiết bị thì không *(chỉ nhánh B)*

```bash
if [ "$NHANH" = B ] && [ "$DRA_API" -eq 1 ]; then
cat > "$WK/claim-va-pod.yaml" <<'YAML'
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: lab-13-claim
  namespace: lab-13
spec:
  devices:
    requests:
    - name: device-0
      exactly:
        deviceClassName: lab-13-example
---
apiVersion: v1
kind: Pod
metadata:
  name: dra-consumer
  namespace: lab-13
spec:
  restartPolicy: Never
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
    resources:
      claims:
      - name: dev
  resourceClaims:
  - name: dev
    resourceClaimName: lab-13-claim
YAML

  kubectl apply -f "$WK/claim-va-pod.yaml"
  kubectl -n lab-13 get resourceclaim lab-13-claim >/dev/null 2>&1 \
    && echo 'PASS: API server chap nhan ResourceClaim tro toi mot DeviceClass khong khop thiet bi nao'

  for i in $(seq 1 24); do
    kubectl -n lab-13 get events \
      --field-selector involvedObject.name=dra-consumer -o name 2>/dev/null | grep -q . && break
    sleep 5
  done

  ALLOC="$(kubectl -n lab-13 get resourceclaim lab-13-claim -o jsonpath='{.status.allocation}')"
  RESV="$(kubectl -n lab-13 get resourceclaim lab-13-claim -o jsonpath='{.status.reservedFor}')"
  PHASE2="$(kubectl -n lab-13 get pod dra-consumer -o jsonpath='{.status.phase}')"
  NODE2="$(kubectl -n lab-13 get pod dra-consumer -o jsonpath='{.spec.nodeName}')"

  kubectl -n lab-13 get resourceclaim lab-13-claim -o yaml | tee "$EV/b4-claim-rong.yaml"
  kubectl -n lab-13 describe pod dra-consumer | tee "$EV/b4-consumer-describe.txt"
  echo "allocation=${ALLOC:-<rong>} | reservedFor=${RESV:-<rong>} | phase=$PHASE2 | node=${NODE2:-<chua gan>}"

  test -z "$ALLOC" \
    && echo 'PASS: status.allocation rong — khong ResourceSlice nao khop nen khong co gi de cap phat'
  test -z "$RESV" \
    && echo 'PASS: status.reservedFor rong — khong Pod nao duoc ghi nhan giu claim'
  test "$PHASE2" = 'Pending' && test -z "$NODE2" \
    && echo 'PASS: Pod Pending va chua gan node — dung o buoc Loc ResourceSlice'
fi
```

**Ý nghĩa:** đây là điểm dừng thật của cluster này, và nó dừng ở **đúng một bước xác định**. Bài 149
liệt kê năm bước; bước 1 (driver tạo ResourceSlice) chưa từng xảy ra, nên bước 3 (*Lọc
ResourceSlice*) không tìm được gì khớp, nên bước 4 (*Cấp phát tài nguyên*) không ghi được gì vào
`status.allocation`, nên bước 5 (*Lập lịch Pod*) không có node nào để chọn. Bốn hệ quả nối nhau từ
một nguyên nhân, và bạn có bốn số đo cho từng khâu.

Chú ý thứ tự: **API server nhận manifest bình thường** — không lỗi, không cảnh báo. Sai lầm nếu có
sẽ nằm ở chỗ workload không bao giờ chạy, chứ không nằm ở chỗ object bị từ chối. Đó là lý do B1
phải chạy **trước**, không phải sau.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B4.B3. ResourceClaimTemplate và controller sinh claim *(chỉ nhánh B)*

Bước 2 của *Luồng công việc của Kubernetes* thuộc về `resourceclaim-controller` trong
kube-controller-manager, **không** thuộc về scheduler và **không** phụ thuộc thiết bị. Kiểm xem nó
có chạy độc lập với việc cluster có thiết bị hay không:

```bash
if [ "$NHANH" = B ] && [ "$DRA_API" -eq 1 ]; then
cat > "$WK/template-va-pod.yaml" <<'YAML'
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: lab-13-template
  namespace: lab-13
spec:
  spec:
    devices:
      requests:
      - name: device-0
        exactly:
          deviceClassName: lab-13-example
---
apiVersion: v1
kind: Pod
metadata:
  name: dra-template-consumer
  namespace: lab-13
spec:
  restartPolicy: Never
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
    resources:
      claims:
      - name: dev
  resourceClaims:
  - name: dev
    resourceClaimTemplateName: lab-13-template
YAML

  kubectl apply -f "$WK/template-va-pod.yaml"

  for i in $(seq 1 24); do
    GEN="$(kubectl -n lab-13 get pod dra-template-consumer \
      -o jsonpath='{.status.resourceClaimStatuses[0].resourceClaimName}' 2>/dev/null)"
    test -n "$GEN" && break
    sleep 5
  done

  kubectl -n lab-13 get resourceclaims -o wide | tee "$EV/b4-claims-sau-template.txt"
  echo "ten claim sinh tu template: ${GEN:-<khong co>}"

  if [ -n "$GEN" ]; then
    echo 'PASS: controller sinh ResourceClaim cho Pod du cluster khong co thiet bi nao'
    kubectl -n lab-13 get resourceclaim "$GEN" -o yaml | tee "$EV/b4-claim-sinh-ra.yaml"
    OWNER="$(kubectl -n lab-13 get resourceclaim "$GEN" \
      -o jsonpath='{.metadata.ownerReferences[0].kind}{" "}{.metadata.ownerReferences[0].name}')"
    echo "ownerReferences = $OWNER"
    test "$OWNER" = 'Pod dra-template-consumer' \
      && echo 'PASS: claim sinh ra thuoc so huu cua chinh Pod'
    GALLOC="$(kubectl -n lab-13 get resourceclaim "$GEN" -o jsonpath='{.status.allocation}')"
    test -z "$GALLOC" \
      && echo 'PASS: claim sinh ra van rong status.allocation — sinh claim khac voi cap phat thiet bi'
  else
    echo 'GHI NHAN: khong co claim nao duoc sinh ra — xem muc 4 truoc khi ket luan'
  fi
fi
```

**Ý nghĩa:** nếu claim được sinh ra, bạn vừa tách được hai việc mà người mới hay gộp làm một:
**sinh ResourceClaim** là việc của một controller ở control plane và xảy ra ngay khi Pod ra đời;
**cấp phát thiết bị** là việc của scheduler và cần ResourceSlice. Trên cluster này việc thứ nhất
chạy, việc thứ hai không — và `status.allocation` rỗng là bằng chứng.

**PASS:** ba dòng `PASS:` khi claim được sinh ra, hoặc một dòng `GHI NHAN:` kèm bằng chứng.

### B4.B4. Hồ sơ "DRA chưa dùng được" *(chỉ nhánh B)*

Nhánh này **không giả vờ** đã cấp phát được gì. Nó chốt hồ sơ để lần sau có cái đối chiếu:

```bash
if [ "$NHANH" = B ]; then
{
  echo "=== $(date -Is) — Lab 13: DRA chua dung duoc tren cluster nay ==="
  echo "nhom API resource.k8s.io duoc phuc vu : $( [ "$DRA_API" -eq 1 ] && echo co || echo khong )"
  echo "so DeviceClass co san truoc lab        : $DC_BEFORE"
  echo "so ResourceSlice                       : $SLICE_BEFORE"
  echo "so driver DRA cong bo thiet bi         : $DRV_N"
  echo "gate DynamicResourceAllocation         : $DRA_GATE"
  echo 'KET LUAN                               : DRA CHUA DUNG DUOC — dung o buoc Loc ResourceSlice'
  echo 'DIEU KIEN DE DUNG DUOC                 : (1) nhom API resource.k8s.io duoc phuc vu;'
  echo '                                         (2) thiet bi duoc gan vao node;'
  echo '                                         (3) mot DRA driver chay tren node va cong bo ResourceSlice;'
  echo '                                         (4) mot DeviceClass co selector khop thiet bi do.'
  echo 'DIEU KIEN THIEU O DAY                  : (2) va (3) — ba VM khong co GPU hay thiet bi chuyen dung.'
  echo 'CHO CHAY LAI                           : nhanh A cua chinh Lab 13, tren cluster co phan cung that.'
} | tee "$EV/b4-ho-so-dra.txt"

grep -q 'KET LUAN' "$EV/b4-ho-so-dra.txt" \
  && echo 'PASS: da ghi ho so DRA chua dung duoc kem bang chung'
fi
```

**Ý nghĩa:** hồ sơ này **không** phải một món nợ trong [sổ nợ lab](README.md#5-sổ-nợ-lab). Sổ nợ ghi
những phần bị chặn bởi **kiến thức của giai đoạn sau**; ở đây không có kiến thức nào bị thiếu, chỉ
có phần cứng bị thiếu — và đó chính là lý do bản đồ lab đánh dấu lab 13 là **tùy chọn** thay vì ghi
nợ. Đừng thêm dòng nào vào sổ nợ vì mục này.

Phần khái niệm thì **không** thiếu gì: B2 đã kiểm chứng bốn kind API và ranh giới phân quyền, B3 đã
tách danh mục khỏi nguồn cung, B4.B1 và B4.B2 đã chỉ ra hai điểm dừng khác nhau, và B5 ngay sau đây
là phần trả lời trọn vẹn câu hỏi mà checkpoint của giai đoạn 13 đặt ra.

**PASS:** dòng `PASS: da ghi ho so DRA chua dung duoc…` xuất hiện.

---

## B5. DRA, device plugin và extended resource — ba con đường, một cluster

**Mục đích:** đây là **mục quan trọng nhất của lab ở nhánh B**, và cũng là thứ mà checkpoint của
giai đoạn 13 yêu cầu khi cluster không có GPU: *giải thích được DRA khác device plugin truyền thống
ở điểm nào*. Mục này chạy ở **cả hai nhánh**.

Ba con đường đưa một thiết bị vào Pod, và bạn đã gặp cả ba ở ba chỗ khác nhau của lộ trình:

| | Extended resource | Device plugin | DRA |
| --- | --- | --- | --- |
| Ai công bố | quản trị viên vá `status.allocatable` của Node | một plugin chạy trên node, đăng ký với kubelet rồi báo số lượng | driver DRA tạo **ResourceSlice** |
| Đơn vị | một **con số nguyên** dưới một tên tự đặt | một **con số nguyên** dưới tên do plugin đặt | một **danh sách thiết bị** kèm thuộc tính và dung lượng |
| Pod xin thế nào | `resources.limits["ten"]: N` trong container | `resources.limits["ten"]: N` trong container | `spec.resourceClaims` của Pod + `resources.claims` của container |
| Chọn lọc | không — chỉ đếm | không — chỉ đếm | **CEL** trên thuộc tính thiết bị |
| Chia sẻ giữa container/Pod | không | không | có, qua cùng một ResourceClaim |
| Khai báo ở đâu | từng container | từng container | claim ở cấp Pod, container chỉ tham chiếu |
| Bạn đã làm ở đâu | [Lab 3c B5](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md#b5-extended-resource-tài-nguyên-do-bạn-tự-đặt-tên) — thật | [bài 184](../184-device-plugins-vi.md), giai đoạn 14 — chưa đọc | lab này |

Bài [149](../149-dynamic-resource-allocation-vi.md) tóm ba cột giữa bằng đúng **ba thiếu sót** của
device plugin: nó **yêu cầu khai báo thiết bị theo từng container**, **không hỗ trợ chia sẻ thiết
bị**, và **không hỗ trợ lọc thiết bị dựa trên biểu thức**. Bốn mục con dưới đây đo từng vế của bảng
trên bằng dữ liệu của chính cluster bạn.

> **Không nhảy cóc:** [bài 184 — Device Plugin](../184-device-plugins-vi.md) thuộc
> [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) và bạn **chưa đọc**. Mọi
> điều nói về device plugin ở đây lấy từ chính bài 149 — nó mô tả đủ ba thiếu sót cần cho phép so
> sánh. Đừng nhảy sang bài 184 lúc này; khi đọc tới nó ở giai đoạn 14, quay lại bảng này.

### B5.1. Ba con đường nằm ở ba chỗ khác nhau trong cùng một Pod spec

```bash
OK=0
for p in pod.spec.containers.resources.limits \
         pod.spec.containers.resources.claims \
         pod.spec.resourceClaims; do
  f="$EV/b5-explain-$(echo "$p" | tr '.' '-').txt"
  if kubectl explain "$p" > "$f" 2>&1 && [ -s "$f" ]; then
    OK=$(( OK + 1 ))
    echo "doc duoc: $p"
  else
    echo "GHI NHAN: khong doc duoc $p"
  fi
done
test "$OK" -ge 2 && echo 'PASS: doc duoc it nhat hai trong ba duong dan schema cua ba con duong'
```

**Ý nghĩa:** hai con đường đầu — extended resource và device plugin — dùng **chung một trường**:
`resources.limits` của container, với khác biệt duy nhất là ai điền con số vào `status.allocatable`
của Node. Đó là lý do bài 149 gộp chúng khi nói về "cách cũ". Con đường thứ ba dùng **hai trường
khác hẳn, ở hai cấp khác nhau**: claim khai ở cấp Pod, container chỉ trỏ tên. Chính hình dạng này
làm cho việc chia sẻ một thiết bị giữa nhiều container trở nên khả thi — điều mà một con số trong
`limits` không diễn đạt được.

**PASS:** dòng `PASS: doc duoc it nhat hai trong ba duong dan schema…` xuất hiện.

### B5.2. Node status không biết gì về DRA

```bash
kubectl get node "$W2" -o jsonpath='{.status.allocatable}' \
  | tr ',' '\n' | tr -d '{}"' | sed '/^$/d' | tee "$EV/b5-allocatable.txt"

EXTRA="$(grep -vcE '^(cpu|memory|pods|ephemeral-storage|hugepages-)' "$EV/b5-allocatable.txt")"
echo "so tai nguyen ngoai chuan trong allocatable cua $W2 = $EXTRA"

if [ "$EXTRA" -eq 0 ]; then
  echo 'PASS: allocatable chi co tai nguyen chuan — khong extended resource, khong device plugin'
else
  echo "GHI NHAN: $EXTRA tai nguyen ngoai chuan — doc lai b5-allocatable.txt truoc khi ket luan"
fi

grep -c 'resource.k8s.io' "$EV/b5-allocatable.txt" \
  | grep -qx '0' \
  && echo 'PASS: khong dau vet nao cua resource.k8s.io trong allocatable cua Node'
```

**Ý nghĩa:** hai kết luận, một phép đo. Thứ nhất: Lab 3c đã dọn sạch extended resource
`example.com/dongle` mà nó quảng bá, đúng như gate cuối của lab đó cam kết — nếu `EXTRA` khác 0 thì
một lab trước để sót. Thứ hai, và quan trọng hơn: **thiết bị DRA không xuất hiện trong
`status.allocatable` của Node**. Đây là khác biệt kiến trúc, không phải chi tiết: hai con đường cũ
hạch toán bằng một con số trên Node, còn DRA hạch toán bằng object riêng (ResourceSlice cho nguồn
cung, ResourceClaim cho phần đã giữ chỗ). Một Pod dùng DRA không "ăn" gì trong `allocatable` cả.

**PASS:** hai dòng `PASS:` của bước này xuất hiện — hoặc một dòng `PASS:` cộng một `GHI NHAN:` được
ghi lại.

### B5.3. Ba con đường đều trống, đọc từ kubelet

```bash
kubectl get --raw "/api/v1/nodes/$W2/proxy/metrics" > "$EV/b5-kubelet-metrics.txt"
test -s "$EV/b5-kubelet-metrics.txt" && echo "PASS: doc duoc /metrics cua kubelet tren $W2"

DRA_OPS="$(grep -c '^dra_operations_duration_seconds' "$EV/b5-kubelet-metrics.txt")"
DRA_GRPC="$(grep -c '^dra_grpc_operations_duration_seconds' "$EV/b5-kubelet-metrics.txt")"
DP_REG="$(grep -c '^kubelet_device_plugin_registration_total' "$EV/b5-kubelet-metrics.txt")"
echo "dra_operations=$DRA_OPS | dra_grpc_operations=$DRA_GRPC | device_plugin_registration=$DP_REG"

{
  echo "con duong 1 — extended resource : tai nguyen ngoai chuan trong allocatable cua $W2 = $EXTRA"
  echo "con duong 2 — device plugin     : dong metric kubelet_device_plugin_registration_total = $DP_REG"
  echo "con duong 3 — DRA               : ResourceSlice=$SLICE_BEFORE driver=$DRV_N \
dong metric dra_operations=$DRA_OPS dra_grpc=$DRA_GRPC"
} | tee "$EV/b5-ba-con-duong.txt"

test "$(wc -l < "$EV/b5-ba-con-duong.txt")" -ge 3 \
  && echo 'PASS: da chot so lieu cua ca ba con duong vao mot cho'
```

**Ý nghĩa:** bài [172](../172-cluster-admin-dra-vi.md) chỉ đúng hai metric của kubelet cần theo dõi
cho DRA — `NodePrepareResources` và `NodeUnprepareResources`, hiện ra dưới họ
`dra_operations_duration_seconds`, cộng họ `dra_grpc_operations_duration_seconds` do gói
`kubeletplugin` phát ra. Chúng chỉ có dữ liệu khi kubelet **thật sự gọi xuống một driver DRA**. Con
số bạn vừa đo trả lời câu hỏi "đã bao giờ có driver nào làm việc trên node này chưa" chính xác hơn
mọi phỏng đoán.

Ba dòng trong `b5-ba-con-duong.txt` là **bằng chứng gọn nhất của cả lab**: ba con đường cấp phát
thiết bị, ba nguồn số liệu độc lập, cùng một kết luận.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B5.4. Metadata thiết bị chỉ có khi container yêu cầu thiết bị

Bài [269](../269-access-dra-device-metadata-vi.md) và mục *Metadata thiết bị DRA trong container*
của bài 149 nói cùng một quy tắc: metadata **chỉ khả dụng bên trong một container khi container đó
yêu cầu thiết bị trong đặc tả của nó, và không khả dụng trong trường hợp khác**. Vế phủ định kiểm
được ngay:

```bash
cat > "$WK/pod-khong-claim.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: dra-none
  namespace: lab-13
spec:
  restartPolicy: Never
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
YAML

kubectl apply -f "$WK/pod-khong-claim.yaml"
kubectl -n lab-13 wait --for=condition=Ready pod/dra-none --timeout=180s

META='/var/run/kubernetes.io/dra-device-attributes'
if kubectl -n lab-13 exec dra-none -- test -d "$META" 2>/dev/null; then
  echo "GHI NHAN: thu muc $META ton tai du container khong yeu cau thiet bi nao"
  kubectl -n lab-13 exec dra-none -- ls -la "$META" | tee "$EV/b5-metadata-dir.txt"
else
  echo "PASS: khong yeu cau thiet bi thi khong co thu muc $META trong container" \
    | tee "$EV/b5-metadata-dir.txt"
fi
```

Ở nhánh A, đọc luôn vế khẳng định trên chính Pod đã được cấp phát thiết bị:

```bash
if [ "$NHANH" = A ]; then
  kubectl -n lab-13 exec dra-consumer -- \
    find "$META" -name '*-metadata.json' -print 2>/dev/null \
    | tee "$EV/b5-metadata-files.txt"
  if [ -s "$EV/b5-metadata-files.txt" ]; then
    echo 'PASS: container yeu cau thiet bi thay duoc file metadata tai duong dan quy uoc'
  else
    echo 'GHI NHAN: driver cua ban khong bat metadata thiet bi — doc tai lieu cua driver'
  fi
fi
```

**Ý nghĩa:** đường dẫn `/var/run/kubernetes.io/dra-device-attributes/...` là **quy ước cứng** của
giao thức metadata: cùng một bố cục trên mọi driver và mọi cluster, tên file theo mẫu
`<driverName>-metadata.json`, nội dung là JSON có `apiVersion` và `kind`. Bài 149 nói rõ đây là
**tính năng phía driver**, không cần thay đổi API Kubernetes và không cần feature gate — nên nếu
nhánh A của bạn không thấy file, lý do nằm ở driver chứ không ở cluster.

**PASS:** một dòng `PASS:` hoặc `GHI NHAN:` của bước này, có ghi vào evidence.

---

## B6. PodGroup và Workload — phần đọc được

**Mục đích:** nhóm mười một bài từ [59](../59-scheduling-group-vi.md) tới
[153](../153-topology-aware-scheduling-vi.md) đều alpha và phụ thuộc feature gate cộng nhóm API
alpha. Lab **không bật gì**. Nhưng có ba thứ đọc được ngay và trả lời được câu quan trọng nhất của
cả nhóm: **cluster của bạn đang đứng ở đâu so với những gì tài liệu mô tả**.

### B6.1. Nhóm `scheduling.k8s.io` đang phục vụ những gì

```bash
kubectl api-versions | grep '^scheduling\.k8s\.io/' | sort | tee "$EV/b6-scheduling-versions.txt"
SCHED_V_N="$(wc -l < "$EV/b6-scheduling-versions.txt")"

kubectl api-resources --api-group=scheduling.k8s.io 2>&1 | tee "$EV/b6-scheduling-resources.txt"

PG_API=0
kubectl api-resources --api-group=scheduling.k8s.io -o name 2>/dev/null \
  | grep -q '^podgroups\.' && PG_API=1
WL_API=0
kubectl api-resources --api-group=scheduling.k8s.io -o name 2>/dev/null \
  | grep -q '^workloads\.' && WL_API=1
echo "SCHED_V_N=$SCHED_V_N | PG_API=$PG_API | WL_API=$WL_API"

test "$SCHED_V_N" -ge 1 \
  && echo 'PASS: nhom scheduling.k8s.io co ton tai — no la nha cua PriorityClass'
test "$PG_API" -eq "$WL_API" \
  && echo 'PASS: PodGroup va Workload cung co hoac cung khong — dung vi ca hai di cung mot gate'
```

**Ý nghĩa:** đây là chỗ dễ hiểu nhầm nhất của cả giai đoạn. Nhóm API `scheduling.k8s.io` **có tồn
tại** trên mọi cluster — PriorityClass sống ở đó và bạn đã dùng nó từ Lab 7a. Nhưng PodGroup và
Workload nằm ở một **version khác** của cùng nhóm đó, và version ấy phải được bật riêng. Danh sách
version bạn vừa in ra là câu trả lời chính xác cho câu "cluster của tôi có PodGroup không", chứ
không phải suy từ việc nhóm API có tên quen thuộc.

Hai bài [75](../75-podgroup-api-vi.md) và [77](../77-workload-api-vi.md) đều nhắc điều kiện này hai
lần trong cùng một bài, vì thiếu một trong hai thứ — nhóm API alpha và feature gate — thì đối tượng
không tồn tại trong cluster.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B6.2. Trường `schedulingGroup` trong Pod spec

```bash
SG_FIELD=0
kubectl explain pod.spec.schedulingGroup > "$EV/b6-explain-schedulinggroup.txt" 2>&1 && SG_FIELD=1
cat "$EV/b6-explain-schedulinggroup.txt"
echo "SG_FIELD=$SG_FIELD | PG_API=$PG_API"

test "$SG_FIELD" -eq "$PG_API" \
  && echo 'PASS: hai phep kiem nhat quan — truong trong Pod spec va kind PodGroup cung co hoac cung khong'
```

**Ý nghĩa:** bài [59](../59-scheduling-group-vi.md) là cửa vào của cả nhóm và nó chỉ thêm **đúng
một trường** vào Pod: `spec.schedulingGroup.podGroupName`, một liên kết theo tên tới một PodGroup
**trong cùng namespace**, và **bất biến** sau khi đặt. Phép kiểm chéo ở trên có giá trị vì hai vế
đi cùng một feature gate: có kind PodGroup mà không có trường trong Pod spec, hoặc ngược lại, là
dấu hiệu cluster đang ở trạng thái nửa vời và bạn phải đọc lại `b1-feature-gates-podgroup.txt` ở
bước sau.

Nếu `SG_FIELD=0`: hai chính sách `basic` và `gang` của bài [79](../79-workload-policies-vi.md) không
có chỗ nào trong cluster để nhận giá trị, và cơ chế "tất cả hoặc không có gì" của bài
[150](../150-gang-scheduling-vi.md) không có đối tượng nào để áp lên. Đó là kết luận trung thực,
không phải thiếu sót của lab.

**PASS:** dòng `PASS: hai phep kiem nhat quan…` xuất hiện.

### B6.3. Apiserver của bạn biết tên feature gate nào

```bash
for g in GenericWorkload GangScheduling WorkloadAwarePreemption WorkloadWithJob; do
  line="$(grep 'kubernetes_feature_enabled{' "$EV/b0-apiserver-metrics.txt" \
          | grep "name=\"$g\"" | head -1)"
  if [ -z "$line" ]; then
    echo "$g : apiserver khong biet ten gate nay"
  else
    echo "$g : $line"
  fi
done | tee "$EV/b1-feature-gates-podgroup.txt"

test "$(wc -l < "$EV/b1-feature-gates-podgroup.txt")" -eq 4 \
  && echo 'PASS: doi chieu du bon feature gate cua nhom PodGroup/Workload'
KNOWN="$(grep -vc 'khong biet ten gate nay' "$EV/b1-feature-gates-podgroup.txt")"
echo "apiserver biet ten $KNOWN/4 gate"
```

**Ý nghĩa:** khác biệt giữa "gate tồn tại nhưng đang tắt" và "binary này chưa từng nghe tên gate
đó" là khác biệt giữa *bật lên là dùng được* và *phải đổi phiên bản Kubernetes mới có*. Đây là cách
đo cụ thể cho câu cảnh báo lặp lại trong cả nhóm bài: **API có thể đổi giữa các phiên bản**. Ghi con
số `KNOWN` vào checkpoint; nó là câu trả lời của riêng cluster bạn, không phải của tài liệu.

**PASS:** dòng `PASS: doi chieu du bon feature gate…` xuất hiện.

---

## B7. Preemption: đường mặc định và đường nhận biết workload

**Mục đích:** bài [78](../78-workload-disruption-priority-vi.md) và
[152](../152-workload-aware-preemption-vi.md) xây trên nền PriorityClass mà Lab 7a đã làm thật. Ở
đây chỉ đọc **cái mà chúng thêm vào** và xác nhận cái nền vẫn là đường duy nhất của cluster này.

### B7.1. PriorityClass thật của cluster

```bash
kubectl get priorityclass -o custom-columns=\
'NAME:.metadata.name,VALUE:.value,GLOBALDEFAULT:.globalDefault,PREEMPTION:.preemptionPolicy' \
  | tee "$EV/b7-priorityclass.txt"

GD_N="$(kubectl get priorityclass \
  -o jsonpath='{range .items[?(@.globalDefault==true)]}{.metadata.name}{"\n"}{end}' \
  | sed '/^$/d' | wc -l)"
echo "so PriorityClass co globalDefault=true : $GD_N"

test "$GD_N" -eq 0 \
  && echo 'PASS: khong PriorityClass nao la globalDefault — object khong khai priorityClassName co priority 0'
test "$(wc -l < "$EV/b7-priorityclass.txt")" -ge 3 \
  && echo 'PASS: cluster co san it nhat hai PriorityClass tich hop'
```

**Ý nghĩa:** bài 78 nói PodGroup dùng **cùng khái niệm PriorityClass** như Pod đơn lẻ, và quy tắc
mặc định giống hệt: không khai `priorityClassName` thì Kubernetes tìm một class có
`globalDefault: true`; không có class nào như vậy thì độ ưu tiên bằng không. Con số `GD_N` bạn vừa
đo là vế thứ hai của quy tắc đó, đúng trên cluster của bạn, và nó áp cho cả Pod lẫn PodGroup.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B7.2. Những gì bài 78 và 152 thêm vào

```bash
PG_PRIO=0
kubectl explain podgroup.spec.priorityClassName > "$EV/b7-explain-pg-prio.txt" 2>&1 && PG_PRIO=1
PG_DM=0
kubectl explain podgroup.spec.disruptionMode > "$EV/b7-explain-pg-dm.txt" 2>&1 && PG_DM=1
echo "PG_PRIO=$PG_PRIO | PG_DM=$PG_DM | PG_API=$PG_API"

test "$PG_PRIO" -eq "$PG_API" && test "$PG_DM" -eq "$PG_API" \
  && echo 'PASS: hai truong cap nhom co hay khong dung theo su ton tai cua kind PodGroup'

kubectl explain pod.spec.priorityClassName > "$EV/b7-explain-pod-prio.txt" 2>&1 \
  && test -s "$EV/b7-explain-pod-prio.txt" \
  && echo 'PASS: duong preemption mac dinh van nguyen ven — Pod van co priorityClassName'
kubectl explain pod.spec.preemptionPolicy > "$EV/b7-explain-pod-pp.txt" 2>&1 \
  && test -s "$EV/b7-explain-pod-pp.txt" \
  && echo 'PASS: truong preemptionPolicy cua Pod van doc duoc'
```

**Ý nghĩa:** hai bài này thêm đúng hai thứ lên nền cũ, và cả hai đều **ở cấp nhóm chứ không ở cấp
Pod**: một độ ưu tiên có tính quyết định ghi đè độ ưu tiên riêng của từng Pod trong nhóm, và một
chế độ gián đoạn quyết định nhóm bị làm gián đoạn theo kiểu từng Pod hay cả nhóm. Không có kind
PodGroup thì không có chỗ nào đặt hai giá trị đó, và cluster của bạn còn đúng một đường preemption:
đường mặc định mà [Lab 7a B6](LAB-7A-LAP-LICH-VA-EVICTION.md#b6-priorityclass-và-preemption) đã làm
thật.

Ghi nhớ hai ranh giới mà hai bài nhấn mạnh và lab không kiểm được: `priority` và `disruptionMode`
của PodGroup **chỉ có hiệu lực trong workload-aware preemption**, không tham gia vào việc xếp hàng
lúc lập lịch; và khi bên chiếm chỗ là một Pod đơn lẻ đi đường mặc định, nó **không tôn trọng** hai
trường đó của nhóm nạn nhân.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

---

## B8. Topology và thuật toán placement

**Mục đích:** bài [80](../80-workload-topology-scheduling-vi.md) và
[153](../153-topology-aware-scheduling-vi.md) cần hai thứ: **nhiều miền topology** để có gì mà chọn,
và **một KubeSchedulerConfiguration** để cấu hình plugin. Đo cả hai.

### B8.1. Ba node đang mang nhãn topology nào

```bash
kubectl get nodes --show-labels | tee "$EV/b8-node-labels.txt"

HOST_N="$(kubectl get nodes --no-headers --show-labels | grep -c 'kubernetes\.io/hostname=')"
ZONE_N="$(kubectl get nodes --no-headers --show-labels | grep -c 'topology\.kubernetes\.io/zone=')"
REGION_N="$(kubectl get nodes --no-headers --show-labels | grep -c 'topology\.kubernetes\.io/region=')"
echo "hostname=$HOST_N/$NODE_N | zone=$ZONE_N/$NODE_N | region=$REGION_N/$NODE_N" \
  | tee "$EV/b8-topology-dem.txt"

test "$HOST_N" -eq "$NODE_N" \
  && echo 'PASS: ca ba node deu co nhan kubernetes.io/hostname'
if [ "$ZONE_N" -eq 0 ] && [ "$REGION_N" -eq 0 ]; then
  echo 'PASS: khong node nao mang nhan zone hay region — cluster chi co mot truc topology la hostname'
else
  echo "GHI NHAN: zone=$ZONE_N region=$REGION_N — doc lai b8-node-labels.txt truoc khi ket luan"
fi
```

**Ý nghĩa:** ràng buộc topology của bài 80 là **một `key` trỏ tới một nhãn node**, và scheduler thực
thi nghiêm ngặt rằng mọi Pod của nhóm nằm trên những node có **cùng chính xác giá trị** cho nhãn
đó. Với `kubernetes.io/hostname`, mỗi node là một miền riêng — nghĩa là một PodGroup nhiều Pod
không bao giờ vừa một miền, và ràng buộc trở nên vô nghĩa. Cluster ba VM trên một host vật lý không
có trục topology nào đáng để tối ưu, và đó là lý do thật khiến nhóm bài này không kiểm chứng được ở
đây — không phải vì thiếu feature gate.

**STOP:** không tự gán nhãn zone cho node để "có nhiều miền". Ngoài việc đó là dựng dữ liệu giả,
[B9.3](#b93-không-cho-người-dùng-gán-nhãn-node) ngay sau đây sẽ chứng minh vì sao quyền gán nhãn
node là một vấn đề **bảo mật**. Lab này không gán nhãn nào, và gate cuối so md5 của toàn bộ nhãn
node để chứng minh điều đó.

**PASS:** hai dòng của bước này, trong đó dòng đầu là `PASS:`.

### B8.2. Scheduler đang chạy bằng cấu hình nào

Chạy trên `lab-k8s-master`. **Chỉ đọc** — không sửa file trong `/etc/kubernetes/manifests`.

```bash
sudo cat /etc/kubernetes/manifests/kube-scheduler.yaml > "$EV/b8-scheduler-manifest.txt"
test -s "$EV/b8-scheduler-manifest.txt" && echo 'PASS: doc duoc manifest cua kube-scheduler'

if grep -q -- '--config=' "$EV/b8-scheduler-manifest.txt"; then
  echo 'GHI NHAN: kube-scheduler dang nap mot KubeSchedulerConfiguration — doc file do truoc khi ket luan'
else
  echo 'PASS: kube-scheduler khong nap file cau hinh nao — khong plugin placement nao duoc cau hinh'
fi

kubectl -n kube-system get pods -l component=kube-controller-manager -o name \
  | tee "$EV/b8-kcm-pod.txt"
sudo grep -o -- '--controllers=[^ "]*' /etc/kubernetes/manifests/kube-controller-manager.yaml \
  | tee "$EV/b8-kcm-controllers.txt" || echo '(khong khai --controllers, dang dung mac dinh)'
```

**Ý nghĩa:** [Lab 7a B10](LAB-7A-LAP-LICH-VA-EVICTION.md) đã đọc chính file này ở **góc hiệu năng**
— `percentageOfNodesToScore` và bin packing. Ở đây đọc lại ở **góc khác**: thuật toán lập lịch theo
placement của bài 151 chạy bằng hai điểm mở rộng plugin, `PlacementGeneratePlugin` và
`PlacementScorePlugin`, còn bài 153 cấu hình ba plugin cụ thể vào đó cùng trọng số của chúng. Cả
hai việc **chỉ làm được qua một KubeSchedulerConfiguration**. Không có file cấu hình thì không có
chỗ để khai, và cluster chạy đúng thuật toán lập lịch từng Pod mà Lab 7a đã mổ xẻ.

Hai lệnh cuối chỉ ra nơi ở của controller ResourceClaim mà bài 172 nói tới: nó là một controller
**nội bộ** do kube-controller-manager điều phối, không phải một Deployment riêng — nên tìm nó bằng
`kubectl get pods` là tìm nhầm chỗ.

**PASS:** dòng `PASS: doc duoc manifest…` xuất hiện, cộng một dòng `PASS:` hoặc `GHI NHAN:` cho cấu
hình.

---

## B9. Hardening scheduler — đối chiếu cấu hình thật với bài 124

**Mục đích:** [bài 124](../124-hardening-scheduler-vi.md) là bài dễ chịu nhất của giai đoạn: nó
không alpha, và phần lớn nội dung áp dụng được cho **mọi** cluster, kể cả cluster lab. Mục này biến
bài đó thành một bảng đối chiếu với cấu hình đang chạy.

**STOP:** toàn bộ mục này **chỉ đọc**. Không sửa manifest, không đổi quyền file, không thêm cờ. Sửa
cấu hình control plane thuộc
[giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy).

### B9.1. Mười hai cờ, đọc từ manifest thật

```bash
M="$EV/b8-scheduler-manifest.txt"
for f in --authentication-kubeconfig --authorization-kubeconfig \
         --authentication-tolerate-lookup-failure --authentication-skip-lookup \
         --authorization-always-allow-paths --profiling --requestheader-client-ca-file \
         --bind-address --permit-address-sharing --permit-port-sharing \
         --tls-cipher-suites --config; do
  v="$(grep -o -- "$f=[^ \"]*" "$M" | head -1)"
  if [ -n "$v" ]; then
    echo "khai      : $v"
  else
    echo "khong khai: $f"
  fi
done | tee "$EV/b9-scheduler-hardening.txt"

test "$(wc -l < "$EV/b9-scheduler-hardening.txt")" -eq 12 \
  && echo 'PASS: doi chieu du muoi hai co ma bai 124 nhac ten'
```

Hai cờ đáng kết luận ngay:

```bash
BIND="$(grep -o -- '--bind-address=[^ "]*' "$M" | head -1 | cut -d= -f2)"
case "$BIND" in
  127.0.0.1|localhost|::1)
    echo "PASS: bind-address=$BIND — kube-scheduler khong lo ra ngoai node control plane" ;;
  '')
    echo 'GHI NHAN: khong khai bind-address — doc mac dinh cua binary truoc khi ket luan' ;;
  *)
    echo "GHI NHAN: bind-address=$BIND — bai 124 khuyen dat localhost" ;;
esac

if grep -q -- '--requestheader-client-ca-file' "$M"; then
  echo 'GHI NHAN: kube-scheduler nhan --requestheader-client-ca-file — bai 124 khuyen tranh truyen'
else
  echo 'PASS: khong truyen --requestheader-client-ca-file, dung khuyen nghi cua bai 124'
fi
```

**Ý nghĩa:** đọc `b9-scheduler-hardening.txt` rồi tự trả lời ba câu, và ghi câu trả lời vào
evidence:

1. Cờ nào bài 124 khuyên đặt mà cluster **không khai**? Với mỗi cờ như vậy, giá trị thực tế là mặc
   định của binary — và mặc định đó có trùng khuyến nghị không?
2. Cờ nào cluster khai **đúng như** bài khuyên, và ai đặt nó — bạn hay kubeadm?
3. Cờ duy nhất bài khuyên **tránh truyền** có xuất hiện không?

Điểm quan trọng: một cờ "không khai" **không đồng nghĩa với "cấu hình sai"**. Bài 124 nêu ba nhóm
lý do khác nhau — xác thực/phân quyền phải nhất quán với apiserver, mạng không nên lộ ra ngoài, TLS
phải khai tường minh bộ mã hóa — và mỗi nhóm có một mặc định riêng. Việc của bạn ở lab này là
**biết cluster đang ở đâu**, không phải sửa nó.

Lý do bài 124 xếp đây là vấn đề **bảo mật** chứ không phải hiệu năng nằm ngay câu mở đầu của nó:
một scheduler bị cấu hình sai có thể **nhắm vào node cụ thể và trục xuất workload đang chia sẻ node
đó**.

**PASS:** dòng `PASS: doi chieu du muoi hai co…` xuất hiện, cộng hai dòng kết luận cho hai cờ trên.

### B9.2. Quyền file của `authentication-kubeconfig`

```bash
AUTHKC="$(grep -o -- '--authentication-kubeconfig=[^ "]*' "$M" | head -1 | cut -d= -f2)"
echo "authentication-kubeconfig = ${AUTHKC:-<khong khai>}"

if [ -n "$AUTHKC" ] && [ -e "$AUTHKC" ]; then
  stat -c '%a %U:%G %n' "$AUTHKC" | tee "$EV/b9-authkubeconfig-perm.txt"
  MODE="$(stat -c '%a' "$AUTHKC")"
  case "$MODE" in
    600|400) echo "PASS: quyen file $MODE — chi chu so huu doc duoc, dung khuyen nghi cua bai 124" ;;
    *)       echo "GHI NHAN: quyen file $MODE — bai 124 yeu cau quyen truy cap file nghiem ngat" ;;
  esac
else
  echo 'GHI NHAN: khong xac dinh duoc duong dan authentication-kubeconfig — ghi lai va bo qua buoc nay'
fi
```

**Ý nghĩa:** đây là khuyến nghị duy nhất của bài 124 **không nằm trong dòng lệnh**. File kubeconfig
này là thứ scheduler dùng để hỏi apiserver về cấu hình xác thực; ai đọc được nó thì mượn được danh
tính của scheduler. Bài yêu cầu bảo vệ nó bằng quyền truy cập file nghiêm ngặt, và `stat` là cách
kiểm rẻ nhất.

**PASS:** một dòng `PASS:` hoặc `GHI NHAN:` của bước này, có ghi vào evidence.

### B9.3. Không cho người dùng gán nhãn node

```bash
{
  echo "SA lab-13/default  patch nodes  -> $(kubectl auth can-i patch nodes \
    --as=system:serviceaccount:lab-13:default)"
  echo "SA lab-13/default  update nodes -> $(kubectl auth can-i update nodes \
    --as=system:serviceaccount:lab-13:default)"
  echo "user thuong        patch nodes  -> $(kubectl auth can-i patch nodes \
    --as=nguoi-dung-thu --as-group=system:authenticated)"
  echo "user thuong        update nodes -> $(kubectl auth can-i update nodes \
    --as=nguoi-dung-thu --as-group=system:authenticated)"
} | tee "$EV/b9-quyen-gan-nhan-node.txt"

NO_N="$(grep -c '\-> no$' "$EV/b9-quyen-gan-nhan-node.txt")"
test "$NO_N" -eq 4 \
  && echo 'PASS: khong danh tinh thuong nao gan duoc nhan cho node'
```

**Ý nghĩa:** mục ngắn nhất của bài 124 cũng là mục thực dụng nhất. Gán nhãn node là `patch` hoặc
`update` trên tài nguyên `nodes` — không có verb riêng tên "label". Ai làm được việc đó thì dùng
`nodeSelector` để **đẩy workload lên những node lẽ ra không được chạm tới**, ví dụ node control
plane hoặc node dành cho workload nhạy cảm. Kẻ tấn công không cần phá scheduler; chỉ cần nói dối
scheduler về node.

Nối với B8.1: đó cũng chính là lý do lab này **không gán nhãn zone** cho node để làm cho phần
topology chạy được. Nhãn node là đầu vào của quyết định lập lịch, không phải dữ liệu trang trí.

**PASS:** dòng `PASS: khong danh tinh thuong nao gan duoc nhan cho node` xuất hiện.

---

## B10. Hardening DRA — đối chiếu quyền thật với bài 125 và 211

**Mục đích:** bài [125](../125-hardening-dra-vi.md) mô tả một cơ chế phân quyền, bài
[211](../211-hardening-dra-tasks-vi.md) biến nó thành quy trình. Bước đầu tiên của bài 211 — *xác
định các thành phần DRA ghi vào status* — và bước cuối — *kiểm chứng* — làm được ngay, không cần
driver.

### B10.1. Hai subresource tổng hợp không có trong `api-resources`

```bash
if [ "$DRA_API" -eq 1 ]; then
  SUB_N="$(grep -cE 'resourceclaims/(binding|driver)' "$EV/b2-api-wide.txt")"
  echo "so dong nhac resourceclaims/binding hoac resourceclaims/driver trong api-resources = $SUB_N"
  test "$SUB_N" -eq 0 \
    && echo 'PASS: hai subresource tong hop khong xuat hien trong api-resources'

  {
    echo "quan tri vien  update resourceclaims/status  -> $(kubectl auth can-i update resourceclaims \
      --subresource=status -n lab-13)"
    echo "quan tri vien  update resourceclaims/binding -> $(kubectl auth can-i update resourceclaims \
      --subresource=binding -n lab-13)"
    echo "quan tri vien  update resourceclaims/driver  -> $(kubectl auth can-i update resourceclaims \
      --subresource=driver -n lab-13)"
  } | tee "$EV/b10-can-i-admin.txt"

  YES_N="$(grep -c '\-> yes$' "$EV/b10-can-i-admin.txt")"
  test "$YES_N" -eq 3 \
    && echo 'PASS: tang uy quyen tra loi duoc ca ba subresource — chung ton tai o day, khong o api-resources'
fi
```

**Ý nghĩa:** hai chỗ này là hai tầng khác nhau và đó là toàn bộ ý của chữ **"tổng hợp"
(synthetic)**. `api-resources` liệt kê những gì API server **phục vụ như một tài nguyên**;
`resourceclaims/binding` và `resourceclaims/driver` không phải endpoint để đọc ghi, mà là **tên
dùng trong phép kiểm phân quyền**. Chúng chỉ hiện ra khi bạn hỏi đúng câu hỏi ủy quyền — và
`kubectl auth can-i --subresource` là cách hỏi câu đó.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B10.2. Quyền thật của scheduler trên `resourceclaims`

```bash
kubectl get clusterrole system:kube-scheduler -o yaml > "$EV/b10-clusterrole-scheduler.yaml" 2>&1
test -s "$EV/b10-clusterrole-scheduler.yaml" \
  && echo 'PASS: doc duoc ClusterRole system:kube-scheduler'

RC_N="$(grep -c 'resourceclaims' "$EV/b10-clusterrole-scheduler.yaml")"
echo "so dong nhac resourceclaims trong ClusterRole system:kube-scheduler = $RC_N"
if [ "$RC_N" -ge 1 ]; then
  grep -n -B2 -A6 'resourceclaims' "$EV/b10-clusterrole-scheduler.yaml" \
    | tee "$EV/b10-scheduler-resourceclaims.txt"
  echo 'PASS: da trich duoc rule that cua scheduler tren resourceclaims'
else
  echo 'GHI NHAN: ClusterRole cua scheduler khong nhac resourceclaims — ghi lai va doc lai muc y nghia'
fi

kubectl get clusterrolebinding system:kube-scheduler -o yaml \
  > "$EV/b10-clusterrolebinding-scheduler.yaml" 2>&1
test -s "$EV/b10-clusterrolebinding-scheduler.yaml" \
  && echo 'PASS: doc duoc ClusterRoleBinding gan role do cho danh tinh nao'
```

**Ý nghĩa:** bài 125 nói scheduler là bên cần `resourceclaims/binding` — nó ghi `status.allocation`
và `status.reservedFor`, đúng hai trường mà B4 đã đọc. File bạn vừa trích là **quyền thật đang có
trong cluster**, không phải manifest ví dụ. So nó với mẫu `dra-binding-updater` của bài 125 và trả
lời: cluster của bạn cấp cho scheduler đúng những verb nào, trên đúng những resource nào?

Đây cũng là bước 1 của bài 211: liệt kê những danh tính đang cập nhật status của ResourceClaim.
Trên cluster này danh sách chỉ có một cái tên — kube-scheduler — vì hai nhóm còn lại (driver cục bộ
trên node và controller đa node) không tồn tại.

**PASS:** hai dòng `PASS:` đầu và dòng `PASS:` cuối xuất hiện; dòng giữa là `PASS:` hoặc `GHI NHAN:`.

### B10.3. Danh tính không liên quan không được ghi status

```bash
if [ "$DRA_API" -eq 1 ]; then
  SA='system:serviceaccount:lab-13:default'
  for v in update patch 'associated-node:update' 'arbitrary-node:update'; do
    r="$(kubectl auth can-i "$v" resourceclaims --subresource=driver -n lab-13 --as="$SA" 2>&1 \
         | tail -1)"
    echo "$v -> $r"
  done | tee "$EV/b10-verbs.txt"

  STD_NO="$(grep -cE '^(update|patch) -> no$' "$EV/b10-verbs.txt")"
  test "$STD_NO" -eq 2 \
    && echo 'PASS: danh tinh khong lien quan bi tu choi ca hai verb tieu chuan tren resourceclaims/driver'

  PREF_NO="$(grep -cE '^(associated-node|arbitrary-node):update -> no$' "$EV/b10-verbs.txt")"
  if [ "$PREF_NO" -eq 2 ]; then
    echo 'PASS: ca hai verb co tien to nhan biet node cung bi tu choi'
  else
    echo 'GHI NHAN: hai verb co tien to tra ve ket qua khac — chep nguyen van b10-verbs.txt'
  fi
fi
```

**Ý nghĩa:** hai tiền tố verb là phần đặc thù nhất của bài 125.
`associated-node:<verb>` dành cho driver **cục bộ trên node** — API server xác minh liên kết node
của bên gửi yêu cầu; `arbitrary-node:<verb>` dành cho controller ở control plane hoặc controller đa
node. Cấp nhầm tiền tố thứ hai cho một driver chạy dạng DaemonSet là **mất đúng phép kiểm đó**:
driver trên một node cập nhật được claim từ bất kỳ node nào. Lớp thứ hai là `resourceNames` giới
hạn theo tên driver, để một driver không đụng vào thiết bị của driver khác.

Bài 125 còn có một bẫy đáng nhớ mà B10.1 đã chứng minh gián tiếp: quyền `update` trên
`resourceclaims/status` **chưa đủ** để ghi `status.devices` — phải có thêm quyền trên đúng
subresource tổng hợp tương ứng.

**PASS:** hai dòng của bước này, trong đó dòng đầu là `PASS:`.

---

## B11. Cleanup và gate cuối

**Mục đích:** xóa mọi thứ lab tạo ra — trong namespace và ở phạm vi cluster — rồi **chứng minh bằng
so sánh giá trị** rằng cluster trở về đúng `04-metrics-ready`, và rằng lab **không cài thêm gì**.

### B11.1. Xóa object của lab

Xóa Pod trước rồi mới xóa namespace: ResourceClaim sinh tự động mang finalizer, và finalizer đó chỉ
được gỡ khi Pod giữ claim đã kết thúc.

```bash
kubectl -n lab-13 delete pod --all --wait=true --timeout=180s
kubectl -n lab-13 get resourceclaims 2>/dev/null | tee "$EV/b11-claims-truoc-khi-xoa.txt"
kubectl delete namespace lab-13 --wait=true --timeout=300s

if [ "$DRA_API" -eq 1 ]; then
  kubectl delete deviceclass lab-13-example --ignore-not-found
fi

rm -f "$WK"/*.yaml
rmdir "$WK"
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` bên dưới biến điều đó
thành gate thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/13/` **giữ lại** — đó là bằng chứng, và
ở nhánh B thì `b4-ho-so-dra.txt` là hồ sơ để lần sau đối chiếu.

### B11.2. Gate cuối

```bash
kubectl get namespace lab-13 >/dev/null 2>&1 \
  && echo 'FAIL: namespace lab-13 van con' \
  || echo 'PASS: namespace lab-13 da xoa'

if [ "$DRA_API" -eq 1 ]; then
  kubectl get deviceclass lab-13-example >/dev/null 2>&1 \
    && echo 'FAIL: DeviceClass cua lab van con' \
    || echo 'PASS: DeviceClass cua lab da xoa'

  DC_AFTER="$(kubectl get deviceclasses --no-headers 2>/dev/null | wc -l)"
  SLICE_AFTER="$(kubectl get resourceslices --no-headers 2>/dev/null | wc -l)"
  test "$DC_AFTER" -eq "$DC_BEFORE" \
    && echo "PASS: so DeviceClass tro ve dung muc dau lab ($DC_BEFORE)"
  test "$SLICE_AFTER" -eq "$SLICE_BEFORE" \
    && echo "PASS: so ResourceSlice khong doi ($SLICE_BEFORE) — lab khong cai driver nao"
fi

API_AFTER="$(kubectl api-resources -o name 2>/dev/null | sort -u | wc -l)"
test "$API_AFTER" -eq "$API_BEFORE" \
  && echo "PASS: so API resource khong doi ($API_BEFORE) — lab khong bat nhom API nao"

CRD_AFTER="$(kubectl get crd -o name 2>/dev/null | wc -l)"
test "$CRD_AFTER" -eq "$CRD_BEFORE" \
  && echo "PASS: so CRD khong doi ($CRD_BEFORE) — lab khong cai CRD nao"

kubectl get --raw '/metrics' > "$EV/b11-apiserver-metrics.txt"
FG_ALL_AFTER="$(grep -c 'kubernetes_feature_enabled{' "$EV/b11-apiserver-metrics.txt")"
FG_ON_AFTER="$(grep 'kubernetes_feature_enabled{' "$EV/b11-apiserver-metrics.txt" | grep -c ' 1$')"
test "$FG_ALL_AFTER" -eq "$FG_ALL_BEFORE" && test "$FG_ON_AFTER" -eq "$FG_ON_BEFORE" \
  && echo "PASS: bang feature gate khong doi ($FG_ON_BEFORE/$FG_ALL_BEFORE) — lab khong bat gate nao"

NS_AFTER="$(kubectl get namespace -o name | sort | tr '\n' ' ')"
test "$NS_AFTER" = "$NS_BEFORE" \
  && echo 'PASS: danh sach namespace khong doi'

PC_AFTER="$(kubectl get priorityclass -o name | sort | tr '\n' ' ')"
test "$PC_AFTER" = "$PC_BEFORE" \
  && echo 'PASS: danh sach PriorityClass khong doi'

LBL_AFTER="$(kubectl get nodes --no-headers --show-labels | awk '{print $1, $NF}' | sort | md5sum)"
test "$LBL_AFTER" = "$LBL_BEFORE" \
  && echo 'PASS: nhan cua ca ba node khong doi — lab khong gan nhan nao'

test ! -e "$WK" && echo 'PASS: manifest tam da xoa'

kubectl wait --for=condition=Ready node --all --timeout=180s
kubectl get nodes
kubectl top node
kubectl get pods -n default
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
```

**Ý nghĩa:** năm phép so ở giữa — `API_AFTER`, `CRD_AFTER`, `FG_ALL_AFTER`/`FG_ON_AFTER`,
`SLICE_AFTER`, `LBL_AFTER` — là thứ biến câu "lab không cài gì" từ một lời hứa thành một phép đo.
Bề mặt API không rộng ra, không CRD nào được thêm, không feature gate nào bị bật, không driver nào
công bố thiết bị mới, không nhãn node nào bị đổi.

**PASS:** không có dòng `FAIL:` nào; đủ các dòng `PASS:` của bước này (mười một dòng khi `DRA_API`
bằng 1, tám dòng khi bằng 0); ba node `Ready`; `kubectl top node` in đủ ba dòng số liệu;
`kubectl get pods -n default` trả `No resources found in default namespace.`; lệnh field selector
trả `No resources found`; CoreDNS đủ replica `READY`. Cluster trở về `04-metrics-ready`; **không
chụp snapshot mới** và **không cần restore** — lab sau bắt đầu ngay từ trạng thái này.

---

## 3. Checkpoint 13

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Bốn kind API của DRA là gì, **ai tạo cái nào**, và cái nào namespace-scoped, cái nào
      cluster-scoped? Bạn đọc phạm vi đó ra bằng lệnh nào, và vì sao phạm vi ấy trùng khớp với ranh
      giới phân quyền mà bài 172 khuyến nghị?
- [ ] **DRA khác device plugin và khác extended resource ở điểm nào?** Kể đúng ba thiếu sót mà bài
      149 gán cho device plugin. Trong Pod spec, ba con đường nằm ở những trường nào? Vì sao viết
      `example.com/gpu: 2` vào `resources.limits` **không** phải cách làm việc của DRA, dù bạn đã
      làm đúng động tác đó ở Lab 3c? Trên cluster của bạn, ba con đường đó đang trống hay đầy — bạn
      đọc ba con số ấy ra từ ba nguồn nào, và chúng nằm ở file evidence nào?
- [ ] Năm phép kiểm năng lực của B1 là gì, cluster của bạn thiếu điều kiện nào, và **thiếu vì lý do
      gì** — phần cứng, feature gate, hay bản phân phối? Vì sao câu trả lời đó không tạo ra một
      món nợ trong sổ nợ lab?
- [ ] Bạn tạo một DeviceClass thành công nhưng số ResourceSlice không đổi. Điều đó nói lên gì về
      quan hệ giữa danh mục và nguồn cung? Nếu bạn tự tạo một ResourceSlice bằng tay thì chuyện gì
      xảy ra khi cluster có driver, và khi không có?
- [ ] Một Pod tham chiếu ResourceClaim **chưa tồn tại** và một Pod tham chiếu ResourceClaim **đã tồn
      tại nhưng không khớp thiết bị nào** — hai tình huống khác nhau ở chỗ nào? Bạn phân biệt bằng
      ba tín hiệu nào trên object, và trong năm bước của *Luồng công việc của Kubernetes*, mỗi
      trường hợp dừng ở bước nào?
- [ ] ResourceClaim so với ResourceClaimTemplate: chọn cái nào khi nào? Ai sinh ra claim từ
      template, thành phần đó nằm ở đâu trong control plane, và việc sinh claim có phụ thuộc việc
      cluster có thiết bị hay không?
- [ ] Nhóm PodGroup/Workload cần chính xác hai thứ gì để dùng được? Nhóm API `scheduling.k8s.io`
      **có** trên cluster bạn — vì sao điều đó vẫn chưa đủ? Apiserver của bạn biết tên bao nhiêu
      trong bốn feature gate của nhóm này, và con số đó nói lên điều gì khác với "gate đang tắt"?
- [ ] Ràng buộc topology của bài 80 dựa trên nhãn node nào của cluster bạn? Vì sao ba VM này không
      có trục topology đáng để tối ưu, và vì sao lab **không** gán nhãn zone để chữa điều đó?
- [ ] Trong mười hai cờ của bài 124, cluster bạn khai bao nhiêu cờ, và cờ nào bài khuyên **tránh**
      truyền? Vì sao "không khai" khác "cấu hình sai"? Vì sao bài coi scheduler bị cấu hình sai là
      rủi ro **bảo mật**?
- [ ] Vì sao không cho người dùng gán nhãn node là biện pháp bảo mật? Verb nào thực sự cho phép gán
      nhãn, và bạn chứng minh cluster mình an toàn bằng lệnh nào?
- [ ] Hai subresource tổng hợp của bài 125 tên gì, mỗi cái gác những trường nào của `status`, và vì
      sao chúng **không** xuất hiện trong `kubectl api-resources` mà vẫn trả lời được ở
      `kubectl auth can-i`? Hai tiền tố verb nhận biết node dùng cho ai, và mất gì khi cấp nhầm?
- [ ] Lab này không tạo snapshot mới. Bạn đã dùng những phép so sánh nào để chứng minh cluster trở
      về đúng `04-metrics-ready` **và** rằng lab không cài thêm gì?

### Bài giải thích cuối cùng

Trong tối đa 10 phút, kể lại **một yêu cầu "cho tôi một GPU" đi được tới đâu thì dừng trên cluster
này, và vì sao**:

1. Người vận hành workload muốn một thiết bị. Câu hỏi đầu tiên không phải "viết YAML thế nào" mà là
   "cluster này có ResourceSlice nào không" — và bạn trả lời bằng `kubectl get resourceslices` chứ
   không bằng trí nhớ. Trước đó nữa là câu hỏi nhóm API `resource.k8s.io` có được phục vụ không,
   với đúng lệnh mà bài 271 quy định.
2. Nếu có: driver đã công bố thiết bị kèm thuộc tính; admin hoặc driver tạo DeviceClass làm danh
   mục; người vận hành tạo ResourceClaim hoặc ResourceClaimTemplate trong namespace của mình; Pod
   khai `spec.resourceClaims`, container trỏ tên qua `resources.claims`. Scheduler lọc
   ResourceSlice, ghi `status.allocation`, ghi `status.reservedFor`, rồi mới đặt Pod lên node truy
   cập được thiết bị. Năm bước, và mỗi bước có một chỗ để nhìn.
3. Nếu **không** có driver: API server vẫn nhận mọi manifest, ResourceClaim vẫn được tạo, controller
   vẫn sinh claim từ template. Không lỗi nào cả — chỉ có `status.allocation` rỗng và một Pod nằm
   `Pending` mãi mãi. Đó là lý do phải kiểm năng lực **trước**, không phải sau.
4. Cùng một thiết bị ấy, hai con đường cũ làm khác hẳn: một con số trong `status.allocatable` của
   Node, một con số trong `resources.limits` của container, không lọc được theo thuộc tính, không
   chia sẻ được, khai lại ở từng container. Ba thiếu sót ấy chính là ba lợi ích mà DRA đổi lấy bằng
   bốn object và một driver.
5. Phần lập lịch theo nhóm — PodGroup, Workload, gang, topology, preemption nhận biết workload —
   đứng ở một tầng khác và cần một bộ điều kiện khác: nhóm API alpha cộng feature gate. Cluster của
   bạn không có, và bạn nói được điều đó bằng số version của nhóm `scheduling.k8s.io` cộng số gate
   mà apiserver biết tên.
6. Cuối cùng là hai bài hardening, và cả hai đều trả lời được bằng cấu hình **đang chạy**: scheduler
   của bạn khai những cờ nào so với khuyến nghị, ai được gán nhãn node, và scheduler đang có đúng
   quyền gì trên `resourceclaims`.
7. Và điều quan trọng nhất của lab: khi điều kiện phần cứng không có, việc đúng là **đo, ghi lại và
   nói rõ dừng ở đâu**, không phải cài thêm thứ gì hay dựng dữ liệu giả để bảng kết quả trông đẹp
   hơn.

Khi mọi checkbox được đánh dấu và không còn nhầm DeviceClass với ResourceSlice, ResourceClaim với
ResourceClaimTemplate, "API có trường" với "tính năng đang bật", hay "claim đã được sinh ra" với
"thiết bị đã được cấp phát", Lab 13 và
[giai đoạn 13](../00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao) hoàn tất — với
ghi chú rõ ràng rằng phần cấp phát thiết bị thật cần một cluster có phần cứng chuyên dụng.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Bảng dưới chỉ liệt kê
sự cố phát sinh từ nội dung bài học giai đoạn 13.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| `kubectl get deviceclasses` báo `the server doesn't have a resource type` | `echo $DRA_API`; `cat ~/lab-evidence/13/b1-api-resource.txt` | **Không phải lỗi.** Đó đúng là tình huống mà bài 271 mô tả: nhóm API `resource.k8s.io` không được phục vụ. Ghi lại, đặt `DRA_API=0`, bỏ qua B2–B4 phần DRA và đi tiếp từ B5.2. **Không** bật nhóm API — đó là sửa cấu hình apiserver |
| `kubectl apply` DeviceClass báo không nhận ra `apiVersion` | Cột `APIVERSION` của dòng `deviceclasses` trong `~/lab-evidence/13/b1-api-resource.txt` | Sửa dòng `apiVersion:` trong manifest cho khớp giá trị đọc được rồi apply lại. Không đoán version |
| `kubectl get --raw '/metrics'` không có dòng `kubernetes_feature_enabled` | `grep -c 'kubernetes_feature_enabled' ~/lab-evidence/13/b0-apiserver-metrics.txt` | Ghi lại `FG_ALL_BEFORE=0` vào evidence và dùng hai đường còn lại của B1.2: `--feature-gates` trong manifest và `featureGates` trong `configz`. Gate cuối về feature gate khi đó so `0` với `0` — vẫn hợp lệ, nhưng nói rõ điều đó ở checkpoint |
| Pod `dra-thieu-claim` không ở `Pending` mà `Failed` | `kubectl -n lab-13 describe pod dra-thieu-claim`; `~/lab-evidence/13/b4-thieu-claim.txt` | Ghi nguyên văn `describe` vào evidence. Kết luận của bài 270 về việc Pod nằm pending không áp dụng cho cluster của bạn — nói rõ điều đó ở checkpoint, đừng sửa lab cho khớp bài |
| `status.allocation` của `lab-13-claim` **không** rỗng ở nhánh B | `kubectl get resourceslices`; `~/lab-evidence/13/b1-nang-luc.txt` | Cluster có thiết bị mà B1 đọc thiếu. Chạy lại B1, cập nhật `NHANH`, ghi cả hai kết quả vào evidence rồi đọc lại kết luận của B4 theo nhánh A |
| Không có ResourceClaim nào sinh ra từ ResourceClaimTemplate | `kubectl -n lab-13 get pod dra-template-consumer -o yaml` phần `status`; `~/lab-evidence/13/b8-kcm-controllers.txt` | Controller ResourceClaim là controller nội bộ của kube-controller-manager. Nếu `--controllers` loại nó ra thì cluster lệch baseline — ghi lại và **không** sửa manifest; đây là kết quả cần báo, không phải cần vá |
| Namespace `lab-13` kẹt `Terminating` | `kubectl -n lab-13 get pods,resourceclaims`; `kubectl -n lab-13 get resourceclaims -o yaml \| grep finalizers -A3` | Còn Pod giữ claim nên finalizer chưa được gỡ. Xóa Pod trước rồi chờ; **không** cưỡng chế finalizer |
| `kubectl get --raw .../proxy/metrics` hoặc `.../proxy/configz` báo `403` | `kubectl config current-context`; user đang dùng | Chạy trên `lab-k8s-master` bằng kubeconfig quản trị. **Không** tạo ClusterRoleBinding để chữa — cách làm đúng và đủ đã có ở [Lab 11a B3](LAB-11A-OBSERVABILITY.md#b3-đọc-metric-là-một-hành-động-được-ủy-quyền) |
| `sudo cat /etc/kubernetes/manifests/kube-scheduler.yaml` báo không có file | `hostname`; `kubectl get node -l node-role.kubernetes.io/control-plane` | Bạn đang ở worker. B8.2, B9.1 và B9.2 chỉ chạy trên `lab-k8s-master` |
| `wc -l` của `b9-scheduler-hardening.txt` khác 12 | `cat ~/lab-evidence/13/b9-scheduler-hardening.txt` | Vòng lặp bị ngắt giữa chừng hoặc `$M` rỗng. Chạy lại B8.2 để nạp lại `$M` rồi chạy lại B9.1; đừng sửa danh sách cờ cho khớp con số |
| `kubectl auth can-i` với verb `associated-node:update` báo lỗi thay vì in `yes`/`no` | `~/lab-evidence/13/b10-verbs.txt` | Ghi nguyên văn lỗi vào evidence. Gate của B10.3 chỉ tính hai verb tiêu chuẩn nên vẫn đi tiếp được; nói rõ ở checkpoint rằng client của bạn không gửi được verb có tiền tố |
| `EXTRA` khác 0 ở B5.2 | `cat ~/lab-evidence/13/b5-allocatable.txt`; gate cuối của Lab 3c | Một lab trước để sót extended resource trên node. Ghi lại tên tài nguyên và **không** tự gỡ ở lab này: gỡ đúng cách nằm ở mục cleanup của [Lab 3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md). Nếu muốn sạch tuyệt đối, restore cả ba VM về `04-metrics-ready` |
| `LBL_AFTER` khác `LBL_BEFORE` ở gate cuối | `kubectl get nodes --show-labels`; `~/lab-evidence/13/b8-node-labels.txt` | Có nhãn node bị thêm hoặc sửa trong lúc chạy lab. So hai file, gỡ đúng nhãn đã thêm; nếu không xác định được, restore cả ba VM về `04-metrics-ready` |
| `API_AFTER`, `CRD_AFTER` hoặc `FG_ON_AFTER` khác giá trị trước lab | `~/lab-evidence/13/b0-anh-nen.txt` | Có thứ gì đó được cài hoặc bật trong lúc chạy lab — đó là điều lab cấm. Restore cả ba VM về `04-metrics-ready` và chạy lại từ B0 |

---

## 5. Nguồn chính thức

- [Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
- [Good practices for Dynamic Resource Allocation as a Cluster Admin](https://kubernetes.io/docs/concepts/cluster-administration/dra/)
- [Hardening Guide — Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/security/hardening-guide/dynamic-resource-allocation/)
- [Hardening Guide — Scheduler Configuration](https://kubernetes.io/docs/concepts/security/hardening-guide/scheduler/)
- [Scheduling Group](https://kubernetes.io/docs/concepts/workloads/pods/scheduling-group/)
- [PodGroup API](https://kubernetes.io/docs/concepts/workloads/podgroup-api/)
- [PodGroup Lifecycle](https://kubernetes.io/docs/concepts/workloads/podgroup-api/lifecycle/)
- [Workload API](https://kubernetes.io/docs/concepts/workloads/workload-api/)
- [Workload Disruption and Priority](https://kubernetes.io/docs/concepts/workloads/workload-api/disruption-and-priority/)
- [PodGroup Scheduling Policies](https://kubernetes.io/docs/concepts/workloads/workload-api/policies/)
- [Topology-Aware Scheduling (Workload API)](https://kubernetes.io/docs/concepts/workloads/workload-api/topology-aware-scheduling/)
- [Gang Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/)
- [PodGroup Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/)
- [Workload-Aware Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/workload-aware-preemption/)
- [Topology-Aware Scheduling (scheduling)](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-aware-scheduling/)
- [Assign Devices to Pods and Containers](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/)
- [Set Up DRA in a Cluster](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/set-up-dra-cluster/)
- [Allocate Devices to Workloads with DRA](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/allocate-devices-dra/)
- [Access DRA Device Metadata](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/access-dra-device-metadata/)
- [Harden Dynamic Resource Allocation in Your Cluster](https://kubernetes.io/docs/tasks/administer-cluster/hardening-dra/)
- [Device Plugins — đọc ở giai đoạn 14, dùng để so sánh với DRA](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
