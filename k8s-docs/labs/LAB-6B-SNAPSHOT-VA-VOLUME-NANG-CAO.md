# Lab 6b — Snapshot và volume nâng cao

> **Điểm bắt đầu:** snapshot `03-storage-ready` — mốc do
> [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md) tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot).
> **Điểm kết thúc:** cleanup trả cluster về đúng `03-storage-ready`, **không tạo snapshot mới**.
> Lab này không cài thêm bất kỳ thành phần hạ tầng nào.
> **Lab trước:** [Lab 6a — PV, PVC và StorageClass](LAB-6A-PV-PVC-VA-STORAGECLASS.md) đã cài dynamic
> provisioner, để lại một StorageClass mặc định và chụp mốc `03-storage-ready`.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng **phần còn lại** của
[Giai đoạn 6 — Lưu trữ](../00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ): tám bài 97, 99, 100, 101,
102, 103, 104, 105. Phần cốt lõi PV/PVC/StorageClass thuộc Lab 6a, không lặp lại ở đây.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không chép
lại con số phiên bản Kubernetes nào** và **không cài thêm gì**: không CSI driver, không snapshot
controller, không CRD, không sửa cấu hình apiserver hay kubelet.

**Lab này là chỗ trả [nợ #5](README.md#5-sổ-nợ-lab) — ảnh chụp nhanh và nhân bản volume.** Nhưng sổ
nợ ghi rõ điều kiện: nợ chỉ trả được **nếu CSI driver đang dùng hỗ trợ snapshot**. Vì vậy
[B1](#b1-kiểm-tra-năng-lực-csi-và-snapshot--chọn-nhánh) là một mục **kiểm tra năng lực** và toàn bộ
phần sau rẽ theo kết quả của nó. Xem [mục 1](#nợ-5--trả-hay-không-trả-quyết-định-bằng-b1).

Lab dùng PVC, StorageClass của giai đoạn 6, Pod và ConfigMap của giai đoạn 3, Deployment/StatefulSet
của giai đoạn 4 và Service của giai đoạn 5 làm công cụ. **Không** dùng ResourceQuota, LimitRange,
node affinity, taint/toleration (giai đoạn 7), RBAC (giai đoạn 9) hay metrics-server (giai đoạn 11).

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`, rồi thêm ba lệnh riêng của lab này:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Ba lệnh riêng của lab 6b: tầng lưu trữ phải đúng như Lab 6a để lại.
kubectl get storageclass
kubectl get persistentvolumeclaim -A
kubectl get namespace lab-6b --ignore-not-found
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod; `kubectl get storageclass` liệt kê
**đúng một** class có hậu tố `(default)` ở cột `NAME`; `kubectl get persistentvolumeclaim -A` trả
`No resources found`; lệnh cuối không in gì — chưa có namespace `lab-6b`.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Đọc **năng lực thật** của tầng lưu trữ trên cluster của mình — provisioner có phải CSI driver
  không, ba CRD snapshot có không, snapshot controller có chạy không, có VolumeSnapshotClass nào
  khớp driver không — và kết luận nợ #5 trả được hay không **bằng bằng chứng chứ không bằng phỏng
  đoán**.
- Ánh xạ ba object của snapshot sang ba object đã học ở Lab 6a: `VolumeSnapshotContent` ↔ PV,
  `VolumeSnapshot` ↔ PVC, `VolumeSnapshotClass` ↔ StorageClass — kèm đúng phạm vi namespace hay
  cluster của từng cái, đọc từ chính API của cluster.
- Vì sao ba API đó **không tự có**: chúng là CRD, không thuộc core API, và việc cài chúng là trách
  nhiệm của bản phân phối chứ không phải của CSI driver.
- Nhân bản volume bằng `dataSource`: bốn ràng buộc bắt buộc, và điều gì thật sự xảy ra khi bộ cấp
  phát **không phải** CSI mà manifest vẫn khai `dataSource` — không có lỗi nào, chỉ có dữ liệu sai.
- Phân biệt `dataSource` và `dataSourceRef` bằng hai phép thử ngược nhau: một trường im lặng bỏ qua
  giá trị không hợp lệ, trường kia báo lỗi; cả hai bất biến sau khi tạo và được API server gán khớp
  nhau.
- **Ba điều kiện đồng thời** để scheduler xét dung lượng lưu trữ, cluster của bạn đạt mấy điều kiện,
  và hệ quả thấy được khi không đạt.
- Ai công bố **giới hạn volume theo node**, qua API nào của CSI, và đọc con số đó ở object nào trong
  cluster.
- Điều kiện để có **giám sát tình trạng volume**, metric nào mang thông tin đó, hai label của nó, và
  vì sao cluster của bạn không thấy metric ấy.
- **VolumeAttributesClass** khác StorageClass ở điểm căn bản nào, và chứng minh được `parameters` của
  một class đã tạo là bất biến.
- Cleanup đúng phạm vi: xóa mọi object lab tạo ra, kể cả object phạm vi cluster, và chứng minh cluster
  trở về đúng `03-storage-ready` bằng những phép **so sánh giá trị trước/sau**, không bằng cảm tính.

### Nợ #5 — trả hay không trả, quyết định bằng B1

[Sổ nợ lab](README.md#5-sổ-nợ-lab) ghi nợ #5 — ảnh chụp nhanh và nhân bản volume — phát sinh ở
[giai đoạn 6](../00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài
[99](../99-volume-snapshots-vi.md)–[101](../101-volume-pvc-datasource-vi.md), bị chặn bởi điều kiện
**CSI driver có hỗ trợ snapshot**. Lab 6b là nơi trả — *nếu* điều kiện thỏa.

[B1](#b1-kiểm-tra-năng-lực-csi-và-snapshot--chọn-nhánh) kiểm tra bốn điều kiện và đặt biến `NHANH`:

| Nhánh | Khi nào | B4 làm gì | Kết luận nợ #5 |
| --- | --- | --- | --- |
| **A** | đủ cả bốn điều kiện | chụp snapshot thật, khôi phục PVC từ snapshot, kiểm `deletionPolicy` | **đã trả** |
| **B** | thiếu ít nhất một điều kiện | ghi nhận nợ chưa trả kèm bằng chứng, và kiểm chứng đúng phần đọc được | **chưa trả** |

Nhánh B **không phải thất bại của lab**. Nó là kết quả trung thực của một cluster bare-metal dùng
provisioner không phải CSI. Điều bị cấm là làm ngược lại: cài thêm CSI driver hoặc snapshot
controller để ép sang nhánh A. Việc đó đổi hạ tầng, mà lab này khai báo trả cluster về
`03-storage-ready` và **không chụp mốc mới**.

**Trước khi làm B3, B4 và B5, đọc lại ba bài [99](../99-volume-snapshots-vi.md),
[100](../100-volume-snapshot-classes-vi.md), [101](../101-volume-pvc-datasource-vi.md)** — cụ thể là
mục *Giới thiệu* của cả ba, vì toàn bộ danh sách điều kiện mà B1 kiểm nằm ở đó.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 6b | Kiểm chứng ở |
| --- | --- |
| [97 — Lớp thuộc tính Volume](../97-volume-attributes-classes-vi.md) | B2 — API `volumeattributesclasses` có trong cluster không, trường `volumeAttributesClassName` có trong schema PVC không, và `parameters` của một class đã tạo là **bất biến** |
| [99 — Ảnh chụp nhanh Volume](../99-volume-snapshots-vi.md) | B1 (bốn điều kiện để snapshot chạy được), B3 (ba API là CRD chứ không phải core API; ánh xạ phạm vi), B4 nhánh A (chụp, `readyToUse`, ràng buộc một-một với `VolumeSnapshotContent`, khôi phục qua `dataSource`, `DeletionPolicy`) hoặc B4 nhánh B (ghi nhận nợ #5 kèm bằng chứng) |
| [100 — Các lớp Volume Snapshot](../100-volume-snapshot-classes-vi.md) | B1 (tìm VolumeSnapshotClass khớp `driver` của provisioner), B3.3 (không có CRD thì việc tạo VolumeSnapshotClass **thất bại** — đúng ghi chú của bài), B4 nhánh A (`deletionPolicy` của class quyết định số phận `VolumeSnapshotContent`) |
| [101 — Nhân bản CSI Volume](../101-volume-pvc-datasource-vi.md) | B5 — `dataSource` trỏ tới một PVC cùng namespace, nguồn đã bound và không đang được dùng, dung lượng đích bằng nguồn, clone là object độc lập, và kết quả thật khi provisioner không phải CSI driver |
| [102 — Volume Populator và Nguồn dữ liệu](../102-volume-populators-vi.md) | B6 — `dataSource` im lặng bỏ qua giá trị không hợp lệ, `dataSourceRef` báo lỗi với đúng giá trị đó, API server gán hai trường khớp nhau, cả hai bất biến sau khi tạo, và `dataSourceRef` nhận kiểu ngoài PVC/VolumeSnapshot |
| [103 — Dung lượng lưu trữ](../103-storage-capacity-vi.md) | B7 — API `CSIStorageCapacity` và ai tạo ra các đối tượng đó, ba điều kiện đồng thời để scheduler xét dung lượng, và thực nghiệm cho thấy khi không đủ điều kiện thì không ai chặn một PVC lớn hơn cả đĩa node |
| [104 — Giới hạn volume theo từng Node](../104-storage-limits-vi.md) | B8 — `CSINode.spec.drivers[].allocatable.count` là nơi driver công bố giới hạn qua `NodeGetInfo`, `attachable-volumes-*` trong `allocatable` của Node, và những trường của `CSIDriver` có thật trong schema của cluster này |
| [105 — Giám sát tình trạng volume](../105-volume-health-monitoring-vi.md) | B9 — metric `kubelet_volume_stats_health_status_abnormal` đọc từ kubelet qua node proxy, Event trên PVC, và feature gate `CSIVolumeHealth` ở phía node |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài [99](../99-volume-snapshots-vi.md), mục *Bảo vệ PersistentVolumeClaim đang làm nguồn Snapshot* | Phải xóa PVC **đúng lúc** snapshot đang chụp dở. Lab không ép được thời điểm đó: backend nào chụp nhanh thì `readyToUse` đã true trước khi kịp gõ lệnh, và làm nó chậm lại là can thiệp vào driver |
| Bài [99](../99-volume-snapshots-vi.md), phần cấp phát sẵn (`snapshotHandle`, `volumeHandle`) và *Chuyển đổi volume mode của một Snapshot* | Cần định danh của một snapshot **có thật trên backend lưu trữ**; cluster lab không có backend nào cấp được id đó, và phần đổi volume mode còn cần annotation trên `VolumeSnapshotContent` do admin điền |
| Bài [97](../97-volume-attributes-classes-vi.md), đổi `volumeAttributesClassName` của PVC từ `silver` sang `gold` | Cần lưu trữ đi qua CSI **và** driver có triển khai `ModifyVolume`, cộng external-resizer. B2 kiểm đúng hai điều kiện đó và ghi lại kết quả thật |
| Bài [102](../102-volume-populators-vi.md), volume populator tùy chỉnh và *Nguồn dữ liệu giữa các namespace* | Populator là controller **bên ngoài** phải cài thêm; phần cross-namespace còn alpha và cần thêm ReferenceGrant của Gateway API. Cài controller mới là đổi hạ tầng, mà lab này trả cluster về `03-storage-ready` |
| Bài [103](../103-storage-capacity-vi.md), mục *Lập lịch lại* | Cần một driver có theo dõi dung lượng và phải cố tình làm cạn chỗ trên một node để driver báo lỗi tạo volume |
| Bài [104](../104-storage-limits-vi.md), `MutableCSINodeAllocatableCount`, `nodeAllocatableUpdatePeriodSeconds`, `preventPodSchedulingIfMissing` | Cần một CSI driver thật, cộng bật feature gate trên `kube-apiserver` và `kubelet`. Sửa cấu hình control plane là lệch baseline; việc đó thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). B8 chỉ đọc schema để biết trường có tồn tại ở API của cluster này không |
| Bài [105](../105-volume-health-monitoring-vi.md), Event tình trạng volume và cờ `enable-node-watcher` | Cần External Health Monitor controller chạy cạnh một CSI driver **có hiện thực** tính năng này. Không driver thì không có gì để báo |
| Bài [99](../99-volume-snapshots-vi.md)–[101](../101-volume-pvc-datasource-vi.md), phần chụp và nhân bản **thật** | Chỉ chạy ở nhánh A. Ở nhánh B, phần này giữ nguyên trong [sổ nợ lab](README.md#5-sổ-nợ-lab) và B4.B1 ghi lại đúng điều kiện còn thiếu |

### 1.2. Thời lượng

2–3 giờ, tính từ lúc gate mở đầu đã PASS. Nhánh A dài hơn nhánh B khoảng nửa giờ. B5, B7 và B10 có
bước phải chờ controller hội tụ; thời gian chờ **phụ thuộc cấu hình cluster** nên mọi bước chờ đều
viết dưới dạng vòng lặp có điều kiện dừng, không phải con số cố định.

---

## 2. Quy ước và an toàn

- Mọi lệnh chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, **trừ khi ghi rõ node khác**.
  Lab này không cần SSH sang node nào.
- **Chạy toàn bộ phần B trong cùng một phiên shell.** Rất nhiều gate so sánh biến đặt ở bước trước
  (`SC`, `PROVISIONER`, `VBM`, `IS_CSI`, `CRD_N`, `NHANH`, `PV_BEFORE`…); mở shell mới giữa chừng là
  mất biến và mất luôn gate cuối.
- **Không hardcode tên StorageClass hay tên provisioner.** B0 đọc chúng từ cluster thật bằng
  `kubectl get storageclass` và mọi manifest sau đó dùng biến. Lab 6a để lại một provisioner và một
  StorageClass mặc định; lab này chỉ cần biết chúng **là gì trên máy bạn**, không cần chúng trùng tên
  với ví dụ nào.
- **Lab tuyệt đối không cài thêm gì:** không CSI driver, không snapshot controller, không CRD, không
  Helm chart, không sửa manifest control plane, không bật feature gate. Thiếu điều kiện thì ghi nhận
  thiếu, không đi vá.
- Lab tạo Namespace `lab-6b` và các object bên trong nó, cộng **một object phạm vi cluster**:
  VolumeAttributesClass `lab-6b-silver` (chỉ khi API đó tồn tại). Cả hai bị xóa ở B10. Nhánh A có thể
  để lại một `VolumeSnapshotContent` nếu `deletionPolicy` là `Retain`; B10 dọn cả trường hợp đó.
- **Không dùng `nodeName` cho Pod tiêu thụ PVC.** Với `volumeBindingMode: WaitForFirstConsumer`,
  annotation chọn node trên PVC do **scheduler** ghi; Pod khai `nodeName` bỏ qua scheduler nên PVC
  không bao giờ được cấp phát và Pod treo. Mọi Pod của lab để scheduler tự đặt.
- Lab này **không có fault injection**: không ép node vào áp lực, không giết Pod hệ thống, không phá
  mạng. Nếu bạn tự thêm bước phá hoại nào, quy ước chung vẫn áp dụng — chỉ chạy trên
  `lab-k8s-worker2`.
- Image dùng cho toàn bộ lab là `busybox:1.37` đã khóa ở
  [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và đã có sẵn trên cả ba node từ
  Lab 00, nên phần B không phụ thuộc mạng ra ngoài.
- Manifest tạm ghi vào `~/lab-work/6b/`; bằng chứng ghi vào `~/lab-evidence/6b/`. Không lưu token,
  key hay certificate.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng bắt đầu bằng `STOP:` nghĩa là
  dừng đúng bước đó và ghi lại, không đi vòng.
- **Cách quay lui khi hỏng:** tắt **cả ba VM**, restore **cả ba** về `03-storage-ready` — không bao
  giờ restore riêng một VM, xem ghi chú cuối
  [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường) — rồi bật lại theo thứ
  tự master → worker 1 → worker 2, chạy lại gate mở đầu và bắt đầu lại từ B0.

---

# Phần B — Thực hành kiến thức 6b

## B0. Chuẩn bị workspace và đọc tầng lưu trữ thật

**Mục đích:** đọc từ cluster mọi thứ mà các mục sau cần, và chốt các con số nền để B10 có cái đối
chiếu. Không mục nào sau đây được phép giả định tên class hay tên provisioner.

```bash
mkdir -p ~/lab-work/6b ~/lab-evidence/6b
kubectl config current-context
kubectl get nodes -o wide
kubectl create namespace lab-6b
kubectl get namespace lab-6b -o jsonpath='{.status.phase}'; echo
```

Đọc StorageClass mặc định và bốn thuộc tính của nó:

```bash
SC="$(kubectl get storageclass --no-headers | awk '/\(default\)/{print $1; exit}')"
PROVISIONER="$(kubectl get storageclass "$SC" -o jsonpath='{.provisioner}')"
VBM="$(kubectl get storageclass "$SC" -o jsonpath='{.volumeBindingMode}')"
RECLAIM="$(kubectl get storageclass "$SC" -o jsonpath='{.reclaimPolicy}')"
EXPAND="$(kubectl get storageclass "$SC" -o jsonpath='{.allowVolumeExpansion}')"

echo "SC=$SC"
echo "provisioner=$PROVISIONER | volumeBindingMode=$VBM | reclaimPolicy=$RECLAIM | allowVolumeExpansion=${EXPAND:-<khong dat>}"
```

Chốt các con số nền dùng cho gate cuối:

```bash
NODE_N="$(kubectl get nodes --no-headers | wc -l)"
PV_BEFORE="$(kubectl get pv --no-headers 2>/dev/null | wc -l)"
PVC_BEFORE="$(kubectl get pvc -A --no-headers 2>/dev/null | wc -l)"
SC_BEFORE="$(kubectl get storageclass -o name | sort | tr '\n' ' ')"
NS_BEFORE="$(kubectl get namespace -o name | sort | tr '\n' ' ')"

{
  echo "=== $(date -Is) — tang luu tru truoc Lab 6b ==="
  echo "SC=$SC provisioner=$PROVISIONER binding=$VBM reclaim=$RECLAIM expansion=${EXPAND:-<khong dat>}"
  echo "NODE_N=$NODE_N PV_BEFORE=$PV_BEFORE PVC_BEFORE=$PVC_BEFORE"
  echo "SC_BEFORE=$SC_BEFORE"
  echo "NS_BEFORE=$NS_BEFORE"
  echo '--- storageclass ---'; kubectl get storageclass -o wide
  echo '--- persistentvolume ---'; kubectl get pv 2>&1
  echo '--- pods toan cluster ---'; kubectl get pods -A -o wide
} | tee ~/lab-evidence/6b/b0-truoc.txt

test -n "$SC" && test -n "$PROVISIONER" \
  && echo 'PASS: doc duoc StorageClass mac dinh va provisioner tu cluster that'
```

**Ý nghĩa:** `NS_BEFORE` bao gồm namespace của provisioner mà Lab 6a cài. B10 so lại đúng chuỗi này,
nên nếu lab lỡ xóa nhầm namespace của provisioner thì gate cuối bắt được ngay. `PV_BEFORE` là số PV
tại `03-storage-ready`; cleanup phải đưa số này về đúng giá trị cũ, không phải về 0 một cách máy móc.

**PASS:** namespace `lab-6b` ở phase `Active`; dòng `PASS: doc duoc StorageClass mac dinh…` xuất
hiện; file `~/lab-evidence/6b/b0-truoc.txt` có đủ năm dòng biến ở đầu. Nếu `SC` rỗng, cluster không
có StorageClass mặc định — dừng lại, xem mục 4.

---

## B1. Kiểm tra năng lực CSI và snapshot — chọn nhánh

**Mục đích:** trả lời một câu hỏi duy nhất bằng bằng chứng: **nợ #5 có trả được trên cluster này
không?** Bài [99](../99-volume-snapshots-vi.md) mục *Giới thiệu* liệt kê đúng những gì phải có; mục
này biến danh sách đó thành bốn phép kiểm.

### B1.1. Bốn phép kiểm

```bash
echo '--- 1. CSI driver dang dang ky trong cluster ---'
kubectl get csidriver 2>&1
kubectl get csinodes -o custom-columns='NODE:.metadata.name,DRIVERS:.spec.drivers[*].name'

echo '--- 2. CRD cua snapshot ---'
kubectl get crd -o name 2>/dev/null | grep 'snapshot' || echo '(khong CRD nao chua chu snapshot)'
kubectl api-resources --api-group=snapshot.storage.k8s.io 2>&1

echo '--- 3. VolumeSnapshotClass ---'
kubectl get volumesnapshotclass 2>&1 | head -5

echo '--- 4. snapshot controller ---'
kubectl get pods -A --no-headers 2>/dev/null | grep -i 'snapshot' \
  || echo '(khong Pod nao ten chua snapshot)'
```

**Ý nghĩa của bốn phép kiểm này, theo đúng thứ tự bài 99 nêu:**

1. **Hỗ trợ `VolumeSnapshot` chỉ khả dụng với các CSI driver.** Nếu provisioner của StorageClass
   mặc định không xuất hiện trong `kubectl get csidriver`, nó không phải CSI driver và mọi thứ sau
   đó vô nghĩa. `csinodes` cho biết node nào quảng bá driver nào — cột `DRIVERS` rỗng nghĩa là không
   có driver nào đăng ký với kubelet.
2. **Ba API là CRD, không phải core API.** Không có CRD thì `kubectl` thậm chí không biết kiểu
   `VolumeSnapshot` tồn tại.
3. **Cần một VolumeSnapshotClass có `driver` khớp** với driver quản lý volume nguồn — bài
   [100](../100-volume-snapshot-classes-vi.md) mục *Các phụ thuộc của VolumeSnapshotClass*.
4. **Cần snapshot controller ở control plane** cộng sidecar csi-snapshotter cạnh driver. Bài 99 nói
   rõ: cài CRD và controller là **trách nhiệm của bản phân phối Kubernetes**, không phải của driver.

### B1.2. Tính toán và chốt nhánh

```bash
kubectl get csidriver "$PROVISIONER" >/dev/null 2>&1 && IS_CSI=1 || IS_CSI=0
CSIDRIVER_N="$(kubectl get csidriver --no-headers 2>/dev/null | wc -l)"
CRD_N="$(kubectl get crd -o name 2>/dev/null | grep -c '\.snapshot\.storage\.k8s\.io$')"
SNAPCTRL_N="$(kubectl get pods -A --no-headers 2>/dev/null | grep -c 'snapshot-controller')"

VSC=''
if [ "$CRD_N" -ge 3 ]; then
  VSC="$(kubectl get volumesnapshotclass --no-headers 2>/dev/null \
        | awk -v d="$PROVISIONER" '$2==d{print $1; exit}')"
fi

if [ "$IS_CSI" -eq 1 ] && [ "$CRD_N" -ge 3 ] && [ "$SNAPCTRL_N" -ge 1 ] && [ -n "$VSC" ]; then
  NHANH=A
else
  NHANH=B
fi

{
  echo "=== $(date -Is) — kiem tra nang luc snapshot ==="
  echo "provisioner                       : $PROVISIONER"
  echo "1. provisioner la CSI driver      : IS_CSI=$IS_CSI (tong so csidriver=$CSIDRIVER_N)"
  echo "2. CRD snapshot da cai            : $CRD_N/3"
  echo "3. VolumeSnapshotClass khop driver: ${VSC:-<khong co>}"
  echo "4. Pod snapshot-controller        : $SNAPCTRL_N"
  echo "NHANH                             : $NHANH"
} | tee ~/lab-evidence/6b/b1-nang-luc.txt

echo "$NHANH" > ~/lab-evidence/6b/b1-nhanh.txt

if [ "$NHANH" = A ]; then
  echo 'NHANH A: du bon dieu kien — B4 chup snapshot that, no #5 duoc TRA trong lab nay'
else
  echo 'NHANH B: thieu it nhat mot dieu kien — khong chup snapshot, no #5 VAN CHUA TRA'
fi
```

**PASS:** in ra **đúng một** dòng bắt đầu bằng `NHANH A:` hoặc `NHANH B:`; file
`~/lab-evidence/6b/b1-nang-luc.txt` có đủ bốn dòng đánh số cộng dòng `NHANH`; file `b1-nhanh.txt`
chứa đúng một ký tự `A` hoặc `B`. Bốn con số trong file phải khớp với những gì B1.1 in ra.

**STOP:** không cài CSI driver, snapshot controller hay CRD để ép sang nhánh A. Nhánh B là kết quả
hợp lệ và là lý do nợ #5 được viết kèm điều kiện ngay từ đầu. Cài driver mới đổi hạ tầng vĩnh viễn,
mà lab này khai báo **trả cluster về `03-storage-ready`** và không chụp mốc mới — muốn có snapshot
thật thì đó phải là một lab tạo mốc, không phải lab này.

---

## B2. VolumeAttributesClass — API có, driver mới là thứ quyết định

**Mục đích:** kiểm chứng phần **đọc được** của bài [97](../97-volume-attributes-classes-vi.md): API
có tồn tại trên cluster này không, trường tương ứng có trong schema PVC không, và điều duy nhất bài
đó bắt phải nhớ — **tham số của một class đã tồn tại là bất biến**.

### B2.1. Bề mặt API của nhóm `storage.k8s.io`

```bash
kubectl api-resources --api-group=storage.k8s.io | tee ~/lab-evidence/6b/b2-api-storage.txt

for r in storageclasses csidrivers csinodes csistoragecapacities; do
  kubectl api-resources --api-group=storage.k8s.io -o name | grep -q "^$r\." \
    && echo "co: $r" || echo "THIEU: $r"
done

VAC_API="$(kubectl api-resources --api-group=storage.k8s.io -o name 2>/dev/null \
          | grep -c '^volumeattributesclasses\.')"
echo "VAC_API=$VAC_API"

kubectl explain persistentvolumeclaim.spec.volumeAttributesClassName 2>&1 \
  | head -8 | tee ~/lab-evidence/6b/b2-explain-pvc-vac.txt
```

**PASS:** bốn dòng `co:` xuất hiện, không dòng `THIEU:` nào; `VAC_API` in ra `0` hoặc `1` và giá trị
đó được ghi lại. Bảng ở `b2-api-storage.txt` là nơi đọc **APIVERSION thật** của
`volumeattributesclasses` trên cluster của bạn — bước sau cần đúng giá trị đó.

### B2.2. `parameters` của một class đã tạo là bất biến

Chỉ chạy khi `VAC_API=1`. Nếu `VAC_API=0`, **STOP:** ghi lại kết quả B2.1 và bỏ qua toàn bộ B2.2 —
cluster đã tắt tính năng, và bật nó là sửa cấu hình apiserver.

```bash
if [ "$VAC_API" -eq 1 ]; then
cat > ~/lab-work/6b/vac-silver.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: VolumeAttributesClass
metadata:
  name: lab-6b-silver
driverName: $PROVISIONER
parameters:
  lab-6b-tier: "silver"
EOF

kubectl apply -f ~/lab-work/6b/vac-silver.yaml
kubectl get volumeattributesclass lab-6b-silver -o yaml | tee ~/lab-evidence/6b/b2-vac.yaml
fi
```

> Nếu `apply` báo không nhận ra `apiVersion`, mở `~/lab-evidence/6b/b2-api-storage.txt`, đọc cột
> `APIVERSION` của dòng `volumeattributesclasses` và sửa dòng `apiVersion:` cho khớp rồi apply lại.
> Đừng đoán.

Bây giờ thử làm đúng điều bài 97 nói là không làm được — sửa tham số của class đang tồn tại:

```bash
if [ "$VAC_API" -eq 1 ]; then
  if kubectl patch volumeattributesclass lab-6b-silver --type=merge \
       -p '{"parameters":{"lab-6b-tier":"gold"}}' 2> ~/lab-evidence/6b/b2-patch-err.txt; then
    echo 'BAT NGO: sua duoc parameters — xem muc 4 truoc khi ket luan'
  else
    echo 'PASS: API tu choi sua parameters — class da tao la bat bien'
    cat ~/lab-evidence/6b/b2-patch-err.txt
  fi
fi
```

**Ý nghĩa:** đây chính là lý do ví dụ của bài 97 phải tạo hẳn một class `gold` mới thay vì sửa
`silver`. Tên class trong `volumeAttributesClassName` của PVC thì đổi được; **tham số bên trong một
class thì không**. Còn hai điều kiện để đi xa hơn — lưu trữ phải đi qua CSI, và driver phải triển
khai `ModifyVolume` — thì cluster này trả lời ngay ở B1: `IS_CSI` bằng bao nhiêu quyết định điều kiện
thứ nhất, và điều kiện thứ hai không kết luận được khi điều kiện thứ nhất chưa đạt. Phần đổi class
trên PVC đang chạy nằm ở bảng *không kiểm chứng được* của mục 1.1.

**PASS:** khi `VAC_API=1`, dòng `PASS: API tu choi sua parameters…` xuất hiện và không có dòng
`BAT NGO:`; khi `VAC_API=0`, B2.2 bị bỏ qua và lý do được ghi vào evidence.

---

## B3. Ba object của snapshot: phạm vi và điều kiện tồn tại

**Mục đích:** kiểm chứng mô hình khái niệm của bài [99](../99-volume-snapshots-vi.md) và ghi chú của
bài [100](../100-volume-snapshot-classes-vi.md) — cả hai đều kiểm được **ở cả hai nhánh**, vì chúng
nói về API chứ không về dữ liệu.

### B3.1. Ánh xạ phạm vi, đọc từ chính API của cluster

```bash
kubectl api-resources --namespaced=true  -o name | grep -qx 'persistentvolumeclaims' \
  && echo 'PASS: persistentvolumeclaims — namespace-scoped'
kubectl api-resources --namespaced=false -o name | grep -qx 'persistentvolumes' \
  && echo 'PASS: persistentvolumes — cluster-scoped'
kubectl api-resources --namespaced=false -o name | grep -qx 'storageclasses.storage.k8s.io' \
  && echo 'PASS: storageclasses — cluster-scoped'
```

**Ý nghĩa:** bài 99 mô tả ba object snapshot bằng đúng ba object này. `VolumeSnapshot` là **yêu cầu
của người dùng**, nên nó namespace-scoped như PVC. `VolumeSnapshotContent` là **tài nguyên trong
cluster**, nên nó cluster-scoped như PV. `VolumeSnapshotClass` là **lớp do admin định nghĩa**, nên nó
cluster-scoped như StorageClass. Ba dòng lệnh trên chứng minh nửa bên trái của phép ánh xạ; nửa bên
phải chỉ tồn tại nếu CRD đã được cài.

**PASS:** đúng ba dòng `PASS:` của bước này xuất hiện.

### B3.2. Số API trong nhóm `snapshot.storage.k8s.io` khớp số CRD đếm được

```bash
kubectl api-resources --api-group=snapshot.storage.k8s.io 2>&1 \
  | tee ~/lab-evidence/6b/b3-api-snapshot.txt

SNAP_API_N="$(kubectl api-resources --api-group=snapshot.storage.k8s.io -o name 2>/dev/null | wc -l)"
echo "SNAP_API_N=$SNAP_API_N | CRD_N=$CRD_N"
test "$SNAP_API_N" -eq "$CRD_N" \
  && echo 'PASS: so API trong group snapshot.storage.k8s.io khop so CRD dem duoc o B1'
```

**Ý nghĩa:** phép so này chứng minh trực tiếp câu quan trọng nhất của bài 99: ba API đó **không thuộc
core API**. Chúng chỉ xuất hiện trong `api-resources` khi và chỉ khi CRD tương ứng đã được cài, và số
lượng luôn bằng nhau. Trên cluster chưa cài CRD, cả hai vế đều bằng 0 — gate vẫn PASS, và đó chính là
bằng chứng.

**PASS:** dòng `PASS: so API trong group…` xuất hiện.

### B3.3. Tạo VolumeSnapshotClass khi chưa có CRD

```bash
cat > ~/lab-work/6b/snapclass.yaml <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: lab-6b-snapclass
driver: $PROVISIONER
deletionPolicy: Delete
EOF

if [ "$CRD_N" -ge 3 ]; then
  kubectl apply --dry-run=server -f ~/lab-work/6b/snapclass.yaml \
    && echo 'PASS: co CRD nen API chap nhan VolumeSnapshotClass'
else
  kubectl apply --dry-run=server -f ~/lab-work/6b/snapclass.yaml \
      2> ~/lab-evidence/6b/b3-snapclass-err.txt \
    && echo 'BAT NGO: API chap nhan du khong dem duoc CRD nao — quay lai B1' \
    || { echo 'PASS: khong co CRD nen viec tao VolumeSnapshotClass that bai'; \
         cat ~/lab-evidence/6b/b3-snapclass-err.txt; }
fi
```

**Ý nghĩa:** ghi chú của bài 100 nói đúng một câu — *"Nếu không có sẵn các CRD cần thiết, việc tạo
một VolumeSnapshotClass sẽ thất bại"* — và đây là bản kiểm chứng của câu đó. `--dry-run=server` gửi
request đi để API server xác thực đầy đủ nhưng **không ghi object nào**, nên bước này không để lại
rác phạm vi cluster ở cả hai nhánh.

**PASS:** đúng một dòng `PASS:` của bước này xuất hiện, khớp với giá trị `CRD_N`; không có dòng
`BAT NGO:`.

---

## B4. Snapshot thật, hoặc ghi nhận nợ #5

**Mục đích:** đây là mục trả nợ. Nó **rẽ nhánh theo biến `NHANH`** đặt ở B1. Làm đúng một nhánh:
`NHANH=A` thì chạy B4.A1 → B4.A4 và bỏ qua B4.B1; `NHANH=B` thì bỏ qua B4.A* và chạy B4.B1.

```bash
echo "Nhanh dang chay: $NHANH"
```

### B4.A1. PVC nguồn và dữ liệu để chụp *(chỉ nhánh A)*

```bash
cat > ~/lab-work/6b/src-snap.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: src-snap
  namespace: lab-6b
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: $SC
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: writer-snap
  namespace: lab-6b
spec:
  restartPolicy: Never
  containers:
  - name: writer
    image: busybox:1.37
    command: ["sh", "-c", "echo lab-6b-snapshot-marker > /data/marker.txt; sync"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: src-snap
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/6b/src-snap.yaml
kubectl apply -f ~/lab-work/6b/src-snap.yaml

for i in $(seq 1 90); do
  PH="$(kubectl get pod writer-snap -n lab-6b -o jsonpath='{.status.phase}' 2>/dev/null)"
  test "$PH" = Succeeded && break
  sleep 2
done
echo "writer-snap phase=$PH"

kubectl delete pod writer-snap -n lab-6b --wait=true
PVC_PHASE="$(kubectl get pvc src-snap -n lab-6b -o jsonpath='{.status.phase}')"
SRC_SIZE="$(kubectl get pvc src-snap -n lab-6b -o jsonpath='{.status.capacity.storage}')"
echo "src-snap phase=$PVC_PHASE size=$SRC_SIZE"

test "$PH" = Succeeded && test "$PVC_PHASE" = Bound \
  && echo 'PASS: PVC nguon da Bound, du lieu da ghi, Pod ghi da bi xoa'
```

**PASS:** `writer-snap` đạt `Succeeded`; `src-snap` ở phase `Bound`; Pod đã bị xóa nên PVC nguồn
**không còn đang được sử dụng** — đúng điều kiện mà bài [101](../101-volume-pvc-datasource-vi.md) đòi
cho phần nhân bản ở B5.

### B4.A2. Chụp snapshot và đọc ràng buộc một-một *(chỉ nhánh A)*

```bash
cat > ~/lab-work/6b/snap-1.yaml <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: snap-1
  namespace: lab-6b
spec:
  volumeSnapshotClassName: $VSC
  source:
    persistentVolumeClaimName: src-snap
EOF

kubectl apply -f ~/lab-work/6b/snap-1.yaml

for i in $(seq 1 90); do
  RTU="$(kubectl get volumesnapshot snap-1 -n lab-6b -o jsonpath='{.status.readyToUse}' 2>/dev/null)"
  test "$RTU" = true && break
  sleep 2
done

CONTENT="$(kubectl get volumesnapshot snap-1 -n lab-6b \
          -o jsonpath='{.status.boundVolumeSnapshotContentName}')"
echo "readyToUse=$RTU | content=$CONTENT"

kubectl get volumesnapshotcontent "$CONTENT" \
  -o jsonpath='{.spec.volumeSnapshotRef.namespace}/{.spec.volumeSnapshotRef.name}{"\n"}' \
  | tee ~/lab-evidence/6b/b4-content-ref.txt

test "$RTU" = true && test -n "$CONTENT" \
  && grep -qx 'lab-6b/snap-1' ~/lab-evidence/6b/b4-content-ref.txt \
  && echo 'PASS: snapshot san sang va rang buoc mot-mot voi dung mot VolumeSnapshotContent'
```

**Ý nghĩa:** bạn tạo `VolumeSnapshot`; **snapshot controller** tạo `VolumeSnapshotContent`; sidecar
csi-snapshotter mới là thứ gọi `CreateSnapshot` xuống endpoint CSI. `volumeSnapshotRef` trong content
trỏ ngược đúng namespace và tên của snapshot — đó là hình dạng cụ thể của ánh xạ một-một mà bài 99
mô tả ở mục *Ràng buộc*.

**PASS:** dòng `PASS: snapshot san sang…` xuất hiện.

### B4.A3. Khôi phục một volume mới từ snapshot *(chỉ nhánh A)*

```bash
cat > ~/lab-work/6b/restore.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored
  namespace: lab-6b
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: $SC
  resources:
    requests:
      storage: $SRC_SIZE
  dataSource:
    apiGroup: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: snap-1
---
apiVersion: v1
kind: Pod
metadata:
  name: reader-restored
  namespace: lab-6b
spec:
  containers:
  - name: reader
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: restored
EOF

kubectl apply -f ~/lab-work/6b/restore.yaml
kubectl wait --for=condition=Ready pod/reader-restored -n lab-6b --timeout=300s

kubectl exec -n lab-6b reader-restored -- cat /data/marker.txt
kubectl exec -n lab-6b reader-restored -- grep -qx 'lab-6b-snapshot-marker' /data/marker.txt \
  && echo 'PASS: volume moi duoc nap san du lieu tu snapshot'
```

**Ý nghĩa:** đây là mục *Cấp phát Volume từ Snapshot* của bài 99 — dùng trường `dataSource` của PVC,
với `apiGroup: snapshot.storage.k8s.io` và `kind: VolumeSnapshot`. Volume mới là một PV khác, một PVC
khác, nhưng nội dung là bản sao tại thời điểm chụp.

**PASS:** dòng `PASS: volume moi duoc nap san du lieu tu snapshot` xuất hiện.

### B4.A4. `DeletionPolicy` quyết định số phận của content *(chỉ nhánh A)*

```bash
POLICY="$(kubectl get volumesnapshotcontent "$CONTENT" -o jsonpath='{.spec.deletionPolicy}')"
echo "deletionPolicy=$POLICY"

kubectl delete volumesnapshot snap-1 -n lab-6b --wait=true
for i in $(seq 1 30); do
  kubectl get volumesnapshotcontent "$CONTENT" >/dev/null 2>&1 || break
  sleep 2
done

if [ "$POLICY" = Delete ]; then
  kubectl get volumesnapshotcontent "$CONTENT" >/dev/null 2>&1 \
    && echo 'FAIL: deletionPolicy Delete nhung content van con' \
    || echo 'PASS: deletionPolicy Delete — content bi xoa cung VolumeSnapshot'
else
  kubectl get volumesnapshotcontent "$CONTENT" >/dev/null 2>&1 \
    && echo 'PASS: deletionPolicy Retain — content duoc giu lai, B10 se don tay' \
    || echo 'FAIL: deletionPolicy Retain nhung content bien mat'
fi

kubectl exec -n lab-6b reader-restored -- grep -qx 'lab-6b-snapshot-marker' /data/marker.txt \
  && echo 'PASS: xoa snapshot khong dung toi volume da khoi phuc'
```

**Ý nghĩa:** cặp `Delete`/`Retain` ở đây là đúng cặp bạn đã gặp với `reclaimPolicy` của PV ở Lab 6a,
chỉ khác đối tượng áp dụng. Và volume đã khôi phục là một object độc lập: xóa snapshot nguồn không
đụng tới nó.

**PASS:** một dòng `PASS:` khớp với giá trị `deletionPolicy` đọc được, không có dòng `FAIL:`; dòng
`PASS: xoa snapshot khong dung toi volume da khoi phuc` xuất hiện. **Nợ #5 phần snapshot và khôi phục
đã trả** — phần nhân bản trả tiếp ở B5.

### B4.B1. Ghi nhận nợ #5 chưa trả, kèm bằng chứng *(chỉ nhánh B)*

Nhánh này **không giả vờ** chạy snapshot. Nó làm hai việc: chốt hồ sơ nợ, và chứng minh bằng lệnh
rằng kết luận ở B1 là đúng chứ không phải suy đoán.

```bash
{
  echo "=== $(date -Is) — Lab 6b, no #5 chua tra ==="
  echo "StorageClass mac dinh        : $SC"
  echo "provisioner                  : $PROVISIONER"
  echo "provisioner la CSI driver    : $( [ "$IS_CSI" -eq 1 ] && echo co || echo khong )"
  echo "CRD snapshot da cai          : $CRD_N/3"
  echo "Pod snapshot-controller      : $SNAPCTRL_N"
  echo "VolumeSnapshotClass khop     : ${VSC:-khong co}"
  echo 'KET LUAN                     : no #5 CHUA TRA'
  echo 'DIEU KIEN DE TRA             : (1) mot CSI driver co ho tro snapshot lam provisioner;'
  echo '                               (2) ba CRD VolumeSnapshot/Content/Class da cai;'
  echo '                               (3) snapshot controller o control plane + sidecar csi-snapshotter;'
  echo '                               (4) mot VolumeSnapshotClass co driver khop provisioner.'
  echo 'CHO TRA                      : chay lai nhanh A cua chinh Lab 6b sau khi cluster co du bon dieu kien.'
} | tee ~/lab-evidence/6b/b4-no-5.txt

grep -q 'no #5 CHUA TRA' ~/lab-evidence/6b/b4-no-5.txt \
  && echo 'PASS: da ghi nhan no #5 chua tra kem bang chung'
```

Chứng minh bằng lệnh rằng kiểu `VolumeSnapshot` thật sự không dùng được:

```bash
if [ "$CRD_N" -ge 3 ]; then
  echo 'CRD co san nhung thieu dieu kien khac — doc lai b1-nang-luc.txt de biet thieu cai nao'
  kubectl get volumesnapshot -n lab-6b 2>&1 | tee ~/lab-evidence/6b/b4-vs.txt
  echo 'PASS: da ghi lai trang thai API snapshot o nhanh B'
else
  kubectl get volumesnapshot -n lab-6b 2> ~/lab-evidence/6b/b4-vs-err.txt \
    && echo 'BAT NGO: doc duoc VolumeSnapshot du CRD_N=0 — quay lai B1' \
    || { echo 'PASS: cluster khong co kieu VolumeSnapshot, dung voi ket luan nhanh B'; \
         cat ~/lab-evidence/6b/b4-vs-err.txt; }
fi
```

**Ý nghĩa:** nợ được giữ nguyên trong [sổ nợ lab](README.md#5-sổ-nợ-lab) với đúng điều kiện cũ. Đừng
đánh dấu giai đoạn 6 là xong khi file `b4-no-5.txt` còn dòng `no #5 CHUA TRA`. Phần khái niệm thì
**không** nợ: B3 đã kiểm chứng mô hình ba object, B5 kiểm chứng ràng buộc của nhân bản, B6 kiểm chứng
hai trường nguồn dữ liệu — tất cả đều chạy được ở nhánh B.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không có dòng `BAT NGO:`.

---

## B5. Nhân bản volume bằng `dataSource`

**Mục đích:** kiểm chứng bài [101](../101-volume-pvc-datasource-vi.md). Mục này chạy ở **cả hai
nhánh**, nhưng kết luận của B5.3 khác nhau: nhánh A cho một bản sao thật, nhánh B cho thấy cái bẫy
nguy hiểm nhất của giai đoạn 6.

### B5.1. PVC nguồn, đã bound và không đang được dùng

```bash
cat > ~/lab-work/6b/src-clone.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: src-clone
  namespace: lab-6b
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: $SC
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: writer-clone
  namespace: lab-6b
spec:
  restartPolicy: Never
  containers:
  - name: writer
    image: busybox:1.37
    command: ["sh", "-c", "echo lab-6b-clone-marker > /data/marker.txt; sync"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: src-clone
EOF

kubectl apply -f ~/lab-work/6b/src-clone.yaml

for i in $(seq 1 90); do
  PH="$(kubectl get pod writer-clone -n lab-6b -o jsonpath='{.status.phase}' 2>/dev/null)"
  test "$PH" = Succeeded && break
  sleep 2
done
kubectl delete pod writer-clone -n lab-6b --wait=true

CLONE_SRC_PHASE="$(kubectl get pvc src-clone -n lab-6b -o jsonpath='{.status.phase}')"
CLONE_SRC_SIZE="$(kubectl get pvc src-clone -n lab-6b -o jsonpath='{.status.capacity.storage}')"
IN_USE="$(kubectl get pods -n lab-6b -o jsonpath='{range .items[*]}{range .spec.volumes[*]}{.persistentVolumeClaim.claimName}{"\n"}{end}{end}' \
         | grep -c '^src-clone$')"
echo "src-clone phase=$CLONE_SRC_PHASE size=$CLONE_SRC_SIZE dang-duoc-dung=$IN_USE"

test "$PH" = Succeeded && test "$CLONE_SRC_PHASE" = Bound && test "$IN_USE" -eq 0 \
  && echo 'PASS: nguon da bound va khong dang duoc su dung — dung dieu kien bai 101'
```

**PASS:** dòng `PASS: nguon da bound va khong dang duoc su dung…` xuất hiện.

### B5.2. Tạo PVC đích trỏ `dataSource` tới PVC nguồn

```bash
cat > ~/lab-work/6b/clone.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: clone-of-src
  namespace: lab-6b
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: $SC
  resources:
    requests:
      storage: $CLONE_SRC_SIZE
  dataSource:
    kind: PersistentVolumeClaim
    name: src-clone
EOF

kubectl apply -f ~/lab-work/6b/clone.yaml
kubectl get pvc clone-of-src -n lab-6b -o jsonpath='{.spec.dataSource}{"\n"}' \
  | tee ~/lab-evidence/6b/b5-datasource.txt

DS_KIND="$(kubectl get pvc clone-of-src -n lab-6b -o jsonpath='{.spec.dataSource.kind}')"
DS_NAME="$(kubectl get pvc clone-of-src -n lab-6b -o jsonpath='{.spec.dataSource.name}')"
test "$DS_KIND" = 'PersistentVolumeClaim' && test "$DS_NAME" = 'src-clone' \
  && echo 'PASS: API giu nguyen dataSource vi PVC la gia tri hop le cua truong nay'
```

**Ý nghĩa:** dung lượng đích lấy đúng `status.capacity.storage` của nguồn, vì ghi chú của bài 101 bắt
buộc `spec.resources.requests.storage` phải **bằng hoặc lớn hơn** dung lượng volume nguồn. Nguồn và
đích cùng namespace `lab-6b` — ràng buộc thứ hai. Cả hai cùng `volumeMode` mặc định `Filesystem` —
ràng buộc thứ ba. Ràng buộc thứ tư, *chỉ CSI driver*, là thứ B5.3 đem ra thử.

**PASS:** dòng `PASS: API giu nguyen dataSource…` xuất hiện.

### B5.3. Mount volume đích và đối chiếu nội dung

```bash
cat > ~/lab-work/6b/reader-clone.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: reader-clone
  namespace: lab-6b
spec:
  containers:
  - name: reader
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: clone-of-src
EOF

kubectl apply -f ~/lab-work/6b/reader-clone.yaml
kubectl wait --for=condition=Ready pod/reader-clone -n lab-6b --timeout=300s

if kubectl exec -n lab-6b reader-clone -- test -f /data/marker.txt; then MARK=co; else MARK=khong; fi
kubectl exec -n lab-6b reader-clone -- ls -la /data | tee ~/lab-evidence/6b/b5-noi-dung-dich.txt
echo "marker trong volume dich: $MARK | IS_CSI=$IS_CSI"

if [ "$IS_CSI" -eq 1 ]; then
  test "$MARK" = co \
    && echo 'PASS: driver CSI da sao chep noi dung — nhan ban that, no #5 phan nhan ban da tra'
else
  test "$MARK" = khong \
    && echo 'PASS: bo cap phat khong phai CSI da bo qua dataSource — volume dich rong'
fi
```

**Ý nghĩa:** đọc kỹ chuỗi sự kiện ở nhánh không phải CSI. API server **chấp nhận** manifest. PVC
**bound** bình thường. Pod **chạy** bình thường. Không một Event lỗi nào bắt buộc phải có. Chỉ có
đúng một thứ sai: dữ liệu không được sao chép. Đó chính là lý do bài 101 đặt câu *"Hỗ trợ nhân bản
chỉ khả dụng cho các CSI driver"* ngay dòng đầu mục *Giới thiệu* — thiếu điều kiện đó thì hệ thống
không báo lỗi, nó chỉ lặng lẽ đưa cho bạn một volume rỗng. Trong vận hành thật, tin rằng mình vừa
nhân bản xong rồi xóa nguồn là mất dữ liệu.

**PASS:** đúng một dòng `PASS:` của bước này xuất hiện, khớp với giá trị `IS_CSI`. Nếu `IS_CSI=0` mà
marker lại **có**, xem mục 4 trước khi kết luận — provisioner có thể thật sự là CSI driver và B1 đọc
sai.

### B5.4. Clone là object độc lập với nguồn

```bash
kubectl exec -n lab-6b reader-clone -- sh -c 'echo chi-co-o-clone > /data/rieng.txt; sync'

cat > ~/lab-work/6b/reader-src.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: reader-src
  namespace: lab-6b
spec:
  containers:
  - name: reader
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: src-clone
EOF

kubectl apply -f ~/lab-work/6b/reader-src.yaml
kubectl wait --for=condition=Ready pod/reader-src -n lab-6b --timeout=300s

kubectl exec -n lab-6b reader-src -- ls -la /data
kubectl exec -n lab-6b reader-src -- test -f /data/rieng.txt \
  && echo 'FAIL: ghi vao clone lai hien o nguon — hai PVC dang dung chung mot volume' \
  || echo 'PASS: ghi vao clone khong anh huong nguon — hai object doc lap'
```

**Ý nghĩa:** mục *Sử dụng* của bài 101 nói clone là **đối tượng độc lập**: dùng, nhân bản tiếp, chụp
snapshot hay xóa đều không cần quan tâm tới nguồn, và ngược lại. Phép thử này đúng ở cả hai nhánh —
kể cả khi nội dung không được sao chép, hai PVC vẫn là hai volume khác nhau.

**PASS:** dòng `PASS: ghi vao clone khong anh huong nguon…` xuất hiện, không có dòng `FAIL:`.

---

## B6. `dataSource` so với `dataSourceRef`

**Mục đích:** kiểm chứng phần đáng nhớ nhất của bài [102](../102-volume-populators-vi.md) với vai trò
admin — **khác biệt giữa hai trường**. Toàn bộ mục này chạy được ở cả hai nhánh vì nó chỉ đụng tới
lớp API.

### B6.1. `dataSource` im lặng bỏ qua giá trị không hợp lệ

```bash
cat > ~/lab-work/6b/ds-invalid.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ds-invalid
  namespace: lab-6b
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: $SC
  resources:
    requests:
      storage: 100Mi
  dataSource:
    kind: ConfigMap
    name: khong-ton-tai
EOF

kubectl apply -f ~/lab-work/6b/ds-invalid.yaml

DS="$(kubectl get pvc ds-invalid -n lab-6b -o jsonpath='{.spec.dataSource}')"
DSR="$(kubectl get pvc ds-invalid -n lab-6b -o jsonpath='{.spec.dataSourceRef}')"
echo "dataSource='${DS:-<rong>}' | dataSourceRef='${DSR:-<rong>}'"

test -z "$DS" \
  && echo 'PASS: dataSource khong hop le bi bo qua nhu the truong de trong'
```

**Ý nghĩa:** `ConfigMap` là **đối tượng core không phải PVC**, đúng định nghĩa "giá trị không hợp lệ"
của bài 102. PVC vẫn được tạo, không lỗi, không cảnh báo — bạn chỉ phát hiện ra khi volume hóa ra
rỗng. Đây là lý do bài khuyên ưu tiên `dataSourceRef`.

**PASS:** dòng `PASS: dataSource khong hop le bi bo qua…` xuất hiện.

### B6.2. Cùng giá trị đó trong `dataSourceRef` thì bị từ chối

```bash
sed 's/name: ds-invalid/name: dsr-invalid/; s/^  dataSource:$/  dataSourceRef:/' \
  ~/lab-work/6b/ds-invalid.yaml > ~/lab-work/6b/dsr-invalid.yaml
grep -n 'name: dsr-invalid\|dataSourceRef:' ~/lab-work/6b/dsr-invalid.yaml

if kubectl apply --dry-run=server -f ~/lab-work/6b/dsr-invalid.yaml \
     2> ~/lab-evidence/6b/b6-dsr-err.txt; then
  echo 'BAT NGO: API chap nhan dataSourceRef kieu core khong phai PVC'
else
  echo 'PASS: dataSourceRef bao loi thay vi im lang bo qua'
  cat ~/lab-evidence/6b/b6-dsr-err.txt
fi
```

**Ý nghĩa:** hai bước B6.1 và B6.2 gửi **cùng một giá trị** vào hai trường khác nhau và nhận hai hành
vi ngược nhau. Đó là khác biệt thứ nhất trong hai khác biệt mà bài 102 liệt kê. `--dry-run=server`
cho phép xác thực đầy đủ mà không tạo object.

**PASS:** dòng `PASS: dataSourceRef bao loi…` xuất hiện, không có dòng `BAT NGO:`.

### B6.3. API server gán hai trường khớp nhau

```bash
cat > ~/lab-work/6b/dsr-pvc.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dsr-pvc
  namespace: lab-6b
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: $SC
  resources:
    requests:
      storage: 100Mi
  dataSourceRef:
    kind: PersistentVolumeClaim
    name: src-clone
EOF

kubectl apply -f ~/lab-work/6b/dsr-pvc.yaml
kubectl get pvc dsr-pvc -n lab-6b \
  -o jsonpath='dataSource   ={.spec.dataSource}{"\n"}dataSourceRef={.spec.dataSourceRef}{"\n"}' \
  | tee ~/lab-evidence/6b/b6-hai-truong.txt

V1="$(kubectl get pvc dsr-pvc -n lab-6b -o jsonpath='{.spec.dataSource.kind}/{.spec.dataSource.name}')"
V2="$(kubectl get pvc dsr-pvc -n lab-6b -o jsonpath='{.spec.dataSourceRef.kind}/{.spec.dataSourceRef.name}')"
echo "V1=$V1 | V2=$V2"
test -n "$V1" && test "$V1" = "$V2" \
  && echo "PASS: chi khai mot truong, API server gan gia tri cho ca hai ($V1)"
```

**PASS:** dòng `PASS: chi khai mot truong…` xuất hiện.

### B6.4. Cả hai trường bất biến sau khi tạo

```bash
if kubectl patch pvc dsr-pvc -n lab-6b --type=merge \
     -p '{"spec":{"dataSourceRef":{"kind":"PersistentVolumeClaim","name":"mot-ten-khac"}}}' \
     2> ~/lab-evidence/6b/b6-immutable-err.txt; then
  echo 'BAT NGO: sua duoc dataSourceRef sau khi tao'
else
  echo 'PASS: dataSourceRef bat bien sau khi tao'
  cat ~/lab-evidence/6b/b6-immutable-err.txt
fi
```

**PASS:** dòng `PASS: dataSourceRef bat bien sau khi tao` xuất hiện, không có dòng `BAT NGO:`.

### B6.5. `dataSourceRef` nhận kiểu ngoài PVC và VolumeSnapshot

```bash
cat > ~/lab-work/6b/dsr-custom.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dsr-custom
  namespace: lab-6b
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: $SC
  resources:
    requests:
      storage: 100Mi
  dataSourceRef:
    apiGroup: example.storage.k8s.io
    kind: ExampleDataSource
    name: lab-6b-example
EOF

kubectl apply -f ~/lab-work/6b/dsr-custom.yaml
DSR_KIND="$(kubectl get pvc dsr-custom -n lab-6b -o jsonpath='{.spec.dataSourceRef.kind}')"
DSR_GROUP="$(kubectl get pvc dsr-custom -n lab-6b -o jsonpath='{.spec.dataSourceRef.apiGroup}')"
echo "dataSourceRef: $DSR_GROUP/$DSR_KIND"

test "$DSR_KIND" = 'ExampleDataSource' && test "$DSR_GROUP" = 'example.storage.k8s.io' \
  && echo 'PASS: dataSourceRef giu nguyen kieu ngoai PVC va VolumeSnapshot'
```

Ghi lại hai thứ nữa để đối chiếu, **không phải gate**:

```bash
kubectl get pvc dsr-custom -n lab-6b \
  -o jsonpath='dataSource={.spec.dataSource}{"\n"}phase={.status.phase}{"\n"}' \
  | tee ~/lab-evidence/6b/b6-dsr-custom.txt
kubectl get events -n lab-6b --field-selector involvedObject.name=dsr-custom \
  | tee ~/lab-evidence/6b/b6-dsr-custom-events.txt
kubectl get --raw /metrics 2>/dev/null | grep 'kubernetes_feature_enabled' \
  | grep -i 'AnyVolumeDataSource' | tee ~/lab-evidence/6b/b6-feature-gate.txt \
  || echo '(apiserver khong phoi bay trang thai gate nay — co the tinh nang da GA)'
```

**Ý nghĩa:** đây là khác biệt thứ hai của bài 102 — `dataSourceRef` chứa được **mọi kiểu đối tượng**
trong cùng namespace, còn `dataSource` sinh ra chỉ để nhận PVC và VolumeSnapshot. PVC này sẽ không
bao giờ được cấp phát vì không có populator controller nào đăng ký xử lý kiểu `ExampleDataSource`, và
bài nói rõ hệ quả: populator là **thành phần bên ngoài**, thiếu nó thì việc tạo volume có thể thất
bại, và nơi để tìm nguyên nhân là **Event trên chính PVC đó** — file
`b6-dsr-custom-events.txt` là chỗ bạn nhìn. Cài populator để đi tiếp là đổi hạ tầng, nằm ngoài lab
này; xem bảng *không kiểm chứng được* ở mục 1.1.

**PASS:** dòng `PASS: dataSourceRef giu nguyen kieu ngoai PVC va VolumeSnapshot` xuất hiện; ba file
evidence của bước ghi lại được tạo.

---

## B7. Dung lượng lưu trữ và lập lịch

**Mục đích:** kiểm chứng bài [103](../103-storage-capacity-vi.md) — nơi lưu trữ gặp lập lịch.

### B7.1. API `CSIStorageCapacity` và ai tạo ra các đối tượng đó

```bash
kubectl api-resources --api-group=storage.k8s.io -o name | grep -q '^csistoragecapacities\.' \
  && echo 'PASS: API CSIStorageCapacity co trong cluster nay'

CAP_N="$(kubectl get csistoragecapacities -A --no-headers 2>/dev/null | wc -l)"
echo "csistoragecapacity=$CAP_N | csidriver=$CSIDRIVER_N"
kubectl get csistoragecapacities -A 2>&1 | tee ~/lab-evidence/6b/b7-capacity.txt

if [ "$CSIDRIVER_N" -eq 0 ]; then
  test "$CAP_N" -eq 0 \
    && echo 'PASS: khong CSI driver nao nen khong doi tuong CSIStorageCapacity nao duoc tao'
fi
```

**Ý nghĩa:** bài 103 nói các đối tượng `CSIStorageCapacity` **do CSI driver tạo ra trong namespace
nơi driver được cài**, mỗi đối tượng mang thông tin dung lượng cho **một storage class** và định
nghĩa những node nào truy cập được. API luôn có sẵn ở tầng cluster; thứ vắng mặt là dữ liệu, vì
không ai tạo ra nó.

**PASS:** dòng `PASS: API CSIStorageCapacity co trong cluster nay` xuất hiện; khi `CSIDRIVER_N` bằng
0 thì có thêm dòng `PASS: khong CSI driver nao…`; khi khác 0 thì `b7-capacity.txt` liệt kê các đối
tượng và mỗi đối tượng phải khai một `storageClassName`.

### B7.2. Ba điều kiện đồng thời

```bash
if [ "$IS_CSI" -eq 1 ]; then
  SCAP="$(kubectl get csidriver "$PROVISIONER" -o jsonpath='{.spec.storageCapacity}')"
  SCAP="${SCAP:-false}"
else
  SCAP='<khong co doi tuong CSIDriver>'
fi

{
  echo "dieu kien 1 — Pod dung volume chua duoc tao        : DAT (moi PVC cua lab deu cap phat dong)"
  echo "dieu kien 2 — SC tro CSI driver + WaitForFirstConsumer: provisioner=$PROVISIONER IS_CSI=$IS_CSI binding=$VBM"
  echo "dieu kien 3 — CSIDriver.spec.storageCapacity=true   : $SCAP"
} | tee ~/lab-evidence/6b/b7-ba-dieu-kien.txt

if [ "$IS_CSI" -eq 1 ] && [ "$VBM" = 'WaitForFirstConsumer' ] && [ "$SCAP" = 'true' ]; then
  CAP_TRACK=1
  echo 'PASS: du ba dieu kien — scheduler CO xet dung luong luu tru tren cluster nay'
else
  CAP_TRACK=0
  echo 'PASS: thieu it nhat mot dieu kien — scheduler KHONG xet dung luong luu tru tren cluster nay'
fi
```

**Ý nghĩa:** ba điều kiện phải đủ **cả ba**. Đáng chú ý là điều kiện 2 gồm hai vế: trỏ tới một CSI
driver **và** dùng `WaitForFirstConsumer`. Với `Immediate`, thứ tự bị đảo ngược — driver quyết định
nơi tạo volume trước, scheduler chỉ chạy theo sau và đặt Pod lên node mà volume đã khả dụng.

**PASS:** đúng một dòng `PASS:` của bước này xuất hiện và khớp với ba giá trị vừa in.

### B7.3. Không có theo dõi dung lượng thì không ai chặn gì

Chỉ chạy khi `CAP_TRACK=0`. Bước này xin một volume lớn hơn hẳn đĩa của node và quan sát xem có ai
phản đối không.

```bash
if [ "$CAP_TRACK" -eq 1 ]; then
  echo 'BO QUA B7.3: cluster co theo doi dung luong nen yeu cau 500Gi se bi tu choi dung nhu thiet ke'
else
cat > ~/lab-work/6b/big.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: big
  namespace: lab-6b
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: $SC
  resources:
    requests:
      storage: 500Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: reader-big
  namespace: lab-6b
spec:
  containers:
  - name: reader
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: big
EOF

kubectl apply -f ~/lab-work/6b/big.yaml
kubectl wait --for=condition=Ready pod/reader-big -n lab-6b --timeout=300s

PV_BIG="$(kubectl get pvc big -n lab-6b -o jsonpath='{.spec.volumeName}')"
CAP_BIG="$(kubectl get pv "$PV_BIG" -o jsonpath='{.spec.capacity.storage}')"
NODE_BIG="$(kubectl get pod reader-big -n lab-6b -o jsonpath='{.spec.nodeName}')"
DF_KB="$(kubectl exec -n lab-6b reader-big -- df -k /data | awk 'NR==2{print $2}')"
echo "PV=$PV_BIG capacity=$CAP_BIG node=$NODE_BIG filesystem-that=${DF_KB} KiB (500Gi = 524288000 KiB)"

test "$CAP_BIG" = '500Gi' && test "$DF_KB" -lt 524288000 \
  && echo 'PASS: cluster cap mot PV 500Gi tren mot filesystem nho hon han — khong ai kiem dung luong'
fi
```

**Ý nghĩa:** PVC được bound, PV khai đúng `500Gi`, Pod chạy, và filesystem bên dưới nhỏ hơn con số đó
nhiều lần. Không thành phần nào nói dối — chỉ là **không có ai được giao việc kiểm tra**: thiếu
`CSIStorageCapacity` thì phép so "kích thước volume với dung lượng liệt kê" mà bài 103 mô tả không
có dữ liệu để chạy. Đó là toàn bộ giá trị của tính năng theo dõi dung lượng, nhìn từ phía vắng mặt
của nó. Và nhớ điều bài nói ở mục *Hạn chế*: kể cả khi bật, nó **tăng xác suất thành công ngay lần
đầu chứ không bảo đảm**.

**PASS:** khi `CAP_TRACK=0`, dòng `PASS: cluster cap mot PV 500Gi…` xuất hiện; khi `CAP_TRACK=1`,
dòng `BO QUA B7.3:` xuất hiện và không tạo PVC nào.

---

## B8. Giới hạn volume theo node

**Mục đích:** kiểm chứng bài [104](../104-storage-limits-vi.md) — ràng buộc lập lịch thứ hai đến từ
lưu trữ: node còn gắn thêm được volume không.

Bài 104 và bài 103 được dịch từ một bản tài liệu **mới hơn baseline khóa ở
[bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa)**, nên đừng suy trạng thái tính năng
từ bài. Mục này đọc thẳng API của cluster đang chạy.

### B8.1. Nơi driver công bố giới hạn: `CSINode`

```bash
CSINODE_N="$(kubectl get csinodes --no-headers 2>/dev/null | wc -l)"
echo "csinode=$CSINODE_N | node=$NODE_N"
test "$CSINODE_N" -eq "$NODE_N" && echo 'PASS: moi Node co dung mot doi tuong CSINode'

kubectl get csinodes \
  -o custom-columns='NODE:.metadata.name,DRIVER:.spec.drivers[*].name,COUNT:.spec.drivers[*].allocatable.count' \
  | tee ~/lab-evidence/6b/b8-csinode.txt

DRIVER_ENTRIES="$(kubectl get csinodes \
  -o jsonpath='{range .items[*]}{range .spec.drivers[*]}{.name}{"\n"}{end}{end}' | grep -c .)"
echo "so ban ghi driver tren cac CSINode = $DRIVER_ENTRIES"

if [ "$CSIDRIVER_N" -eq 0 ]; then
  test "$DRIVER_ENTRIES" -eq 0 \
    && echo 'PASS: khong driver nao cong bo allocatable.count qua NodeGetInfo'
fi
```

**Ý nghĩa:** với cluster không chạy trên nhà cung cấp đám mây nào, ba con số mặc định ở đầu bài 104
(EBS, GCE PD, Azure Disk) hoàn toàn không liên quan. Nguồn duy nhất của giới hạn là **CSI driver tự
công bố qua `NodeGetInfo`**, và chỗ con số đó hiện ra trong Kubernetes chính là
`CSINode.spec.drivers[].allocatable.count`. Cột `COUNT` rỗng nghĩa là kube-scheduler không có giới
hạn nào để tôn trọng.

**PASS:** dòng `PASS: moi Node co dung mot doi tuong CSINode` xuất hiện; khi `CSIDRIVER_N` bằng 0 thì
có thêm dòng `PASS: khong driver nao cong bo allocatable.count…`.

### B8.2. `allocatable` của Node không có tài nguyên `attachable-volumes-*`

```bash
ATT_N="$(kubectl get nodes -o json | grep -c 'attachable-volumes')"
echo "so dong 'attachable-volumes' trong node object = $ATT_N"
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.status.allocatable}{"\n"}{end}' \
  | tee ~/lab-evidence/6b/b8-node-allocatable.txt

if [ "$CSIDRIVER_N" -eq 0 ]; then
  test "$ATT_N" -eq 0 \
    && echo 'PASS: khong node nao quang ba tai nguyen so luong volume gan duoc'
fi
```

**PASS:** khi `CSIDRIVER_N` bằng 0, dòng `PASS: khong node nao quang ba…` xuất hiện; file
`b8-node-allocatable.txt` ghi lại `allocatable` thật của ba node.

### B8.3. Những trường của `CSIDriver` có thật trong schema cluster này

```bash
kubectl explain csidriver.spec > ~/lab-evidence/6b/b8-csidriver-spec.txt 2>&1
for f in attachRequired storageCapacity nodeAllocatableUpdatePeriodSeconds preventPodSchedulingIfMissing; do
  if kubectl explain "csidriver.spec.$f" >/dev/null 2>&1; then
    echo "co: $f"
  else
    echo "khong co: $f"
  fi
done | tee ~/lab-evidence/6b/b8-truong-csidriver.txt
```

**Ý nghĩa:** `storageCapacity` là trường bật cơ chế của bài 103; `nodeAllocatableUpdatePeriodSeconds`
là trường cho phép kubelet định kỳ gọi lại `NodeGetInfo`; `preventPodSchedulingIfMissing` là trường
chặn Pod lên node chưa cài driver, và **chỉ áp dụng cho những Pod cần CSI volume tương ứng**. Việc
một trường có mặt trong schema **không** có nghĩa là feature gate tương ứng đang bật — nó chỉ nói API
của cluster biết trường đó. Muốn dùng thật thì phải bật gate trên `kube-apiserver` và `kubelet`, tức
sửa cấu hình control plane, việc đó thuộc
[giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy).

**PASS:** hai dòng `co: attachRequired` và `co: storageCapacity` xuất hiện. Hai trường còn lại in
`co:` hay `khong co:` đều hợp lệ — giá trị thật được ghi vào `b8-truong-csidriver.txt`, đừng sửa nó
cho khớp với bài.

---

## B9. Giám sát tình trạng volume

**Mục đích:** kiểm chứng bài [105](../105-volume-health-monitoring-vi.md) — khi lưu trữ bên dưới hỏng
thì Kubernetes báo cho bạn ở đâu, và vì sao trên cluster này chưa có chỗ nào để báo.

### B9.1. Phải có PVC đang được mount thì kubelet mới có gì mà báo

```bash
MOUNTED="$(kubectl get pods -n lab-6b \
  -o jsonpath='{range .items[*]}{range .spec.volumes[*]}{.persistentVolumeClaim.claimName}{"\n"}{end}{end}' \
  | grep -c .)"
kubectl get pods -n lab-6b -o wide
echo "so tham chieu PVC trong cac Pod dang chay = $MOUNTED"
test "$MOUNTED" -ge 1 && echo 'PASS: co it nhat mot PVC dang duoc mount'

TARGET_NODE="$(kubectl get pod reader-clone -n lab-6b -o jsonpath='{.spec.nodeName}')"
echo "node dang mount PVC cua lab: $TARGET_NODE"
```

**PASS:** `MOUNTED` lớn hơn hoặc bằng 1; `TARGET_NODE` in ra tên một node thật. Pod `reader-clone` của
B5 vẫn đang chạy, nên bước này không tạo thêm object nào.

### B9.2. Đọc metric từ kubelet qua node proxy

```bash
kubectl get --raw "/api/v1/nodes/$TARGET_NODE/proxy/metrics" \
  > ~/lab-evidence/6b/b9-kubelet-metrics.txt
test -s ~/lab-evidence/6b/b9-kubelet-metrics.txt \
  && echo 'PASS: doc duoc /metrics cua kubelet qua node proxy'

HEALTH_N="$(grep -c '^kubelet_volume_stats_health_status_abnormal' ~/lab-evidence/6b/b9-kubelet-metrics.txt)"
STATS_N="$(grep -c '^kubelet_volume_stats_' ~/lab-evidence/6b/b9-kubelet-metrics.txt)"
echo "kubelet_volume_stats_* tong=$STATS_N | rieng health_status_abnormal=$HEALTH_N"
grep '^kubelet_volume_stats_' ~/lab-evidence/6b/b9-kubelet-metrics.txt | head -20 \
  | tee ~/lab-evidence/6b/b9-volume-stats.txt

if [ "$CSIDRIVER_N" -eq 0 ]; then
  test "$HEALTH_N" -eq 0 \
    && echo 'PASS: khong CSI driver nao -> khong co metric tinh trang volume'
fi

kubectl get events -n lab-6b --field-selector involvedObject.kind=PersistentVolumeClaim \
  | tee ~/lab-evidence/6b/b9-pvc-events.txt
```

**Ý nghĩa:** đường đi `kubectl → API server → kubelet` ở đây đúng là đường đã kiểm ở tầng 6 của
[gate A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready). Bài 105 nói tính năng được hiện
thực ở **hai chỗ**: External Health Monitor controller báo Event **trên PVC**, còn kubelet báo Event
**trên mọi Pod đang dùng PVC** và phơi bày metric
`kubelet_volume_stats_health_status_abnormal` với hai label `namespace` và `persistentvolumeclaim`,
giá trị `1` là không khỏe mạnh và `0` là khỏe mạnh. Không có CSI driver nào hiện thực tính năng thì
không có gì để báo — kể cả khi PVC đang được mount ngay lúc này.

**PASS:** dòng `PASS: doc duoc /metrics cua kubelet…` xuất hiện; khi `CSIDRIVER_N` bằng 0 thì có thêm
dòng `PASS: khong CSI driver nao…`; file `b9-pvc-events.txt` được ghi (rỗng cũng hợp lệ — nó là bằng
chứng rằng không controller nào báo Event tình trạng nào lên PVC).

### B9.3. Feature gate phía node

```bash
grep 'kubernetes_feature_enabled' ~/lab-evidence/6b/b9-kubelet-metrics.txt \
  | grep -i 'CSIVolumeHealth' | tee ~/lab-evidence/6b/b9-gate.txt \
  || echo '(kubelet khong phoi bay trang thai gate CSIVolumeHealth trong /metrics)'
```

**Ý nghĩa:** ghi lại để đối chiếu, **không phải gate**. Bài 105 nói phần phía node cần feature gate
`CSIVolumeHealth`; nhưng gate bật hay tắt cũng không đổi kết luận ở đây, vì điều kiện đứng trước nó —
một CSI driver **có hiện thực** giám sát tình trạng — chưa thỏa. Toàn bộ tính năng còn ở mức alpha,
nên đừng xây quy trình vận hành dựa vào nó.

---

## B10. Cleanup và gate cuối

**Mục đích:** xóa mọi thứ lab tạo ra — trong namespace và ở phạm vi cluster — rồi **chứng minh bằng
so sánh giá trị** rằng cluster trở về đúng `03-storage-ready`.

### B10.1. Xóa object của lab

Trước khi xóa, ghi lại các PV mà lab đã sinh ra cùng đường dẫn thật của chúng trên node — đây là thứ
duy nhất của lab nằm ngoài API và không xóa được bằng `kubectl`:

```bash
kubectl get pv \
  -o jsonpath='{range .items[?(@.spec.claimRef.namespace=="lab-6b")]}{.metadata.name}{" | "}{.spec.local.path}{.spec.hostPath.path}{" | "}{.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}{"\n"}{end}' \
  | tee ~/lab-evidence/6b/b10-pv-truoc-khi-xoa.txt
```

**Ý nghĩa:** với PV cấp phát động và `reclaimPolicy: Delete`, provisioner tự dọn thư mục rồi mới xóa
object PV, nên gate `PV_NOW = PV_BEFORE` bên dưới đã bao hàm cả phần đĩa. File này là bằng chứng để
đối chiếu nếu gate đó không đạt.

```bash
kubectl delete namespace lab-6b --wait=true --timeout=300s

if [ "$VAC_API" -eq 1 ]; then
  kubectl delete volumeattributesclass lab-6b-silver --ignore-not-found
fi

if [ "$NHANH" = A ]; then
  for c in $(kubectl get volumesnapshotcontent \
      -o jsonpath='{range .items[?(@.spec.volumeSnapshotRef.namespace=="lab-6b")]}{.metadata.name}{"\n"}{end}' \
      2>/dev/null); do
    kubectl delete volumesnapshotcontent "$c"
  done
fi

for i in $(seq 1 60); do
  PV_NOW="$(kubectl get pv --no-headers 2>/dev/null | wc -l)"
  test "$PV_NOW" -eq "$PV_BEFORE" && break
  sleep 5
done
echo "PV_NOW=$PV_NOW | PV_BEFORE=$PV_BEFORE"
```

**Ý nghĩa:** xóa namespace là đủ cho PVC và Pod, nhưng **không** đủ cho object phạm vi cluster.
VolumeAttributesClass ở B2 và `VolumeSnapshotContent` còn sót ở nhánh A khi `deletionPolicy` là
`Retain` phải xóa tay. Vòng lặp chờ PV giảm về `PV_BEFORE` là nơi `reclaimPolicy` của StorageClass
thể hiện: `Delete` thì PV tự biến mất theo PVC, `Retain` thì không và bạn phải dọn tay.

```bash
rm -f ~/lab-work/6b/*.yaml
rmdir ~/lab-work/6b
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` bên dưới biến điều đó
thành gate thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/6b/` **giữ lại** — đó là bằng chứng, và ở
nhánh B thì `b4-no-5.txt` là hồ sơ nợ.

### B10.2. Gate cuối

```bash
kubectl get namespace lab-6b >/dev/null 2>&1 \
  && echo 'FAIL: namespace lab-6b van con' \
  || echo 'PASS: namespace lab-6b da xoa'

test "$PV_NOW" -eq "$PV_BEFORE" \
  && echo "PASS: so PV tro ve dung muc dau lab ($PV_BEFORE)"

PVC_NOW="$(kubectl get pvc -A --no-headers 2>/dev/null | wc -l)"
test "$PVC_NOW" -eq "$PVC_BEFORE" \
  && echo "PASS: so PVC tro ve dung muc dau lab ($PVC_BEFORE)"

SC_AFTER="$(kubectl get storageclass -o name | sort | tr '\n' ' ')"
test "$SC_AFTER" = "$SC_BEFORE" && echo 'PASS: danh sach StorageClass khong doi'
test "$(kubectl get storageclass --no-headers | awk '/\(default\)/{print $1; exit}')" = "$SC" \
  && echo "PASS: StorageClass mac dinh van la $SC"

NS_AFTER="$(kubectl get namespace -o name | sort | tr '\n' ' ')"
test "$NS_AFTER" = "$NS_BEFORE" \
  && echo 'PASS: danh sach namespace khong doi — namespace cua provisioner con nguyen'

CSIDRIVER_AFTER="$(kubectl get csidriver --no-headers 2>/dev/null | wc -l)"
test "$CSIDRIVER_AFTER" -eq "$CSIDRIVER_N" \
  && echo 'PASS: so CSI driver khong doi — lab khong cai them driver nao'

CRD_AFTER="$(kubectl get crd -o name 2>/dev/null | grep -c '\.snapshot\.storage\.k8s\.io$')"
test "$CRD_AFTER" -eq "$CRD_N" \
  && echo 'PASS: so CRD snapshot khong doi — lab khong cai CRD hay snapshot controller nao'

if [ "$VAC_API" -eq 1 ]; then
  kubectl get volumeattributesclass lab-6b-silver >/dev/null 2>&1 \
    && echo 'FAIL: VolumeAttributesClass cua lab van con' \
    || echo 'PASS: VolumeAttributesClass cua lab da xoa'
fi

test ! -e ~/lab-work/6b && echo 'PASS: manifest tam da xoa'

kubectl wait --for=condition=Ready node --all --timeout=180s
kubectl get nodes
kubectl get pods -n default
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
```

**PASS:** không có dòng `FAIL:` nào; các dòng `PASS:` của bước này xuất hiện (chín dòng khi `VAC_API`
bằng 1, tám dòng khi bằng 0); ba node `Ready`; `kubectl get pods -n default` trả
`No resources found in default namespace.`; lệnh field selector trả `No resources found`; CoreDNS đủ
replica `READY`. Cluster trở về `03-storage-ready`; **không chụp snapshot mới** và **không cần restore**
— lab sau bắt đầu ngay từ trạng thái này.

---

## 3. Checkpoint 6b

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] `VolumeSnapshot`, `VolumeSnapshotContent`, `VolumeSnapshotClass` ứng với ba object nào bạn đã
      dùng ở Lab 6a? Cái nào namespace-scoped, cái nào cluster-scoped, và trong cấp phát động thì ai
      tạo cái nào?
- [ ] Bốn điều kiện phải đủ để chụp được một snapshot thật là gì? Cluster của bạn thiếu điều kiện
      nào, và bạn đọc điều đó ra từ những lệnh nào?
- [ ] Bạn `kubectl apply` một VolumeSnapshotClass trên cluster chưa cài CRD. Kết quả? Ai chịu trách
      nhiệm cài CRD và snapshot controller — CSI driver hay bản phân phối Kubernetes?
- [ ] Hai VolumeSnapshotClass cùng đánh dấu mặc định: khi nào chấp nhận được, khi nào làm hỏng việc
      tạo VolumeSnapshot? Chỗ này khác StorageClass thế nào?
- [ ] Bốn ràng buộc của nhân bản volume là gì? Trên cluster dùng bộ cấp phát **không phải** CSI, điều
      gì xảy ra khi bạn vẫn khai `dataSource` — API báo lỗi, PVC `Pending`, hay chuyện khác? Vì sao
      đó là tình huống nguy hiểm hơn một lỗi rõ ràng?
- [ ] `dataSource` và `dataSourceRef` khác nhau ở hai điểm nào? Bạn đã chứng minh điểm khác biệt thứ
      nhất bằng hai lệnh nào, và tại sao hai lệnh đó phải dùng **cùng một giá trị**? Khi chỉ khai một
      trong hai trường thì trường còn lại mang giá trị gì, và sau khi PVC đã tạo có sửa được không?
- [ ] Ba điều kiện đồng thời để scheduler xét dung lượng lưu trữ là gì? Cluster bạn đạt mấy? Việc một
      PVC 500Gi được cấp trên đĩa 40 GB nói lên điều gì — và nó có nghĩa là Kubernetes bị lỗi không?
- [ ] Ai công bố giới hạn số volume gắn được vào một node, qua API nào của CSI, và bạn đọc con số đó
      ở object nào trong cluster? Bảng EBS/GCE/Azure ở đầu bài 104 có liên quan tới cluster của bạn
      không?
- [ ] Volume health: Event được báo trên PVC hay trên Pod, và điều đó phụ thuộc vào cái gì? Metric
      tên gì, hai label nào, giá trị `1` nghĩa là gì? Vì sao cluster của bạn không thấy metric đó dù
      đang có PVC được mount?
- [ ] VolumeAttributesClass khác StorageClass ở điểm căn bản nào? Bạn cần nâng `iops` của một class
      đang dùng — sửa thẳng class đó được không, và bạn đã chứng minh câu trả lời bằng lệnh nào?
- [ ] **Nợ #5 đã trả hay chưa trên cluster của bạn?** Bằng chứng nằm ở file nào trong
      `~/lab-evidence/6b/`? Nếu chưa trả, cần đúng những gì để trả, và trả ở đâu?
- [ ] Lab này không tạo snapshot mới. Bạn đã dùng những phép so sánh nào để chứng minh cluster trở về
      đúng `03-storage-ready`, và vì sao so `PV_NOW` với `PV_BEFORE` tốt hơn là so với 0?

### Bài giải thích cuối cùng

Trong tối đa 10 phút, kể lại **một yêu cầu lưu trữ nâng cao đi được tới đâu thì dừng trên cluster
này, và vì sao**:

1. Người dùng muốn một bản sao dữ liệu tại một thời điểm. Câu hỏi đầu tiên không phải "viết YAML thế
   nào" mà là "tầng lưu trữ bên dưới có phải CSI không" — và bạn trả lời bằng `kubectl get csidriver`
   chứ không bằng trí nhớ.
2. Nếu là CSI: ba CRD phải có, snapshot controller phải chạy, VolumeSnapshotClass phải khớp driver.
   Người dùng tạo `VolumeSnapshot`, controller tạo `VolumeSnapshotContent`, sidecar gọi xuống endpoint
   CSI. Ràng buộc một-một, y hệt PVC với PV.
3. Volume mới sinh ra từ snapshot qua `dataSource`, và từ một PVC khác qua cũng chính `dataSource` —
   hai nguồn dữ liệu tích hợp sẵn duy nhất. Mọi nguồn khác cần một populator bên ngoài và đi qua
   `dataSourceRef`.
4. Nếu **không** phải CSI: API vẫn nhận manifest, PVC vẫn bound, Pod vẫn chạy. Sai lầm nằm ở dữ liệu,
   không ở trạng thái object. Đó là lý do phải kiểm năng lực trước, không phải sau.
5. Cùng một chữ "CSI" còn quyết định ba thứ nữa: scheduler có xét dung lượng node không, node có giới
   hạn số volume gắn được không, và có ai báo cho bạn biết volume hỏng không. Cả ba đều dừng ở cùng
   một chỗ trên cluster này, vì cùng một lý do.
6. Và điều quan trọng nhất của lab: khi điều kiện không đủ, việc đúng là **ghi nhận nợ kèm bằng
   chứng**, không phải cài thêm thứ gì để bảng kết quả trông đẹp hơn.

Khi mọi checkbox được đánh dấu và không còn nhầm `VolumeSnapshot` với `VolumeSnapshotContent`,
`dataSource` với `dataSourceRef`, "API có trường" với "tính năng đang bật", hay "PVC bound" với "dữ
liệu đã được sao chép", Lab 6b và
[giai đoạn 6](../00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) hoàn tất — với ghi chú rõ ràng về trạng
thái nợ #5.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Bảng dưới chỉ liệt kê
sự cố phát sinh từ nội dung bài học 6b.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| `SC` rỗng ở B0 | `kubectl get storageclass` — phải có đúng một dòng `NAME` kèm `(default)` | Cluster chưa ở `03-storage-ready`. Quay lại [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md) hoặc restore cả ba VM về mốc đó; **không** tự tạo StorageClass ở lab này |
| PVC của lab mãi `Pending` dù Pod đã tạo | `kubectl describe pvc <ten> -n lab-6b`; `echo $VBM`; `kubectl get pvc <ten> -n lab-6b -o yaml \| grep selected-node` | Với `WaitForFirstConsumer`, annotation chọn node do **scheduler** ghi. Nếu bạn tự thêm `nodeName` vào Pod thì scheduler bị bỏ qua và PVC không bao giờ được cấp phát — bỏ `nodeName` đi |
| Pod của lab kẹt `ContainerCreating` | `kubectl describe pod <ten> -n lab-6b` phần `Events`; Pod của provisioner có `Running` không | Đọc Event trước. Nếu provisioner chết, restore cả ba VM về `03-storage-ready` thay vì sửa tay |
| `kubectl get volumesnapshot` báo `the server doesn't have a resource type` | `echo $CRD_N` | **Không phải lỗi.** Đó đúng là kết quả nhánh B mà B4.B1 dùng làm bằng chứng. Đừng cài CRD |
| `kubectl apply` VolumeSnapshotClass báo `no matches for kind` | `echo $CRD_N` | Như trên — đây là gate PASS của B3.3, không phải sự cố |
| B2.2 patch `parameters` **thành công** | `kubectl explain volumeattributesclass`; `kubectl get volumeattributesclass lab-6b-silver -o yaml` | Ghi nguyên văn kết quả vào evidence và **không** kết luận theo bài. Đừng đi tiếp phần suy luận về tính bất biến khi cluster của bạn cho kết quả khác |
| B2.2 `apply` báo không nhận ra `apiVersion` | Cột `APIVERSION` của dòng `volumeattributesclasses` trong `~/lab-evidence/6b/b2-api-storage.txt` | Sửa dòng `apiVersion:` trong manifest cho khớp giá trị đọc được rồi apply lại. Không đoán version |
| B5.3: `IS_CSI=0` nhưng marker **có** trong volume đích | `kubectl get csidriver`; `kubectl get sc $SC -o yaml`; `~/lab-evidence/6b/b1-nang-luc.txt` | Provisioner có thể thật sự là CSI driver và B1 đọc thiếu. Chạy lại B1, cập nhật `IS_CSI`, ghi cả hai kết quả vào evidence rồi đọc lại kết luận của B5 |
| B6.1: PVC `ds-invalid` bị API **từ chối** thay vì tạo được | Nội dung lỗi trả về | Ghi nguyên văn lỗi vào `~/lab-evidence/6b/`. Kết luận của bài 102 về việc `dataSource` im lặng bỏ qua giá trị không hợp lệ không áp dụng cho cluster của bạn — nói rõ điều đó ở checkpoint |
| B6.2 hoặc B6.4 **không** bị từ chối | `kubectl get pvc dsr-pvc -n lab-6b -o yaml` | Như trên: ghi lại kết quả thật, không sửa lab cho khớp bài |
| B7.3: PVC `big` `Pending` | `kubectl describe pvc big -n lab-6b` phần `Events` | Provisioner của bạn **có** kiểm dung lượng. Ghi lại Event, giảm `500Gi` xuống một giá trị nhỏ hơn đĩa node rồi chạy lại; kết luận của B7.3 lúc này là "có ai đó kiểm", ghi đúng như vậy |
| `df -k /data` trả rỗng ở B7.3 | `kubectl exec -n lab-6b reader-big -- df -k` (không tham số đường dẫn) | Đọc dòng có `/data` ở cột cuối và lấy cột thứ hai; đừng đổi sang tham số khác của `df` trong busybox |
| `kubectl get --raw .../proxy/metrics` báo `403` | `kubectl config current-context`; user đang dùng | Chạy trên `lab-k8s-master` bằng kubeconfig quản trị. **Không** tạo ClusterRoleBinding để chữa — RBAC thuộc [giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) |
| `CSINODE_N` khác `NODE_N` ở B8.1 | `kubectl get csinodes`; `kubectl get nodes` | Đối tượng `CSINode` do kubelet tạo cho node của nó, độc lập với việc có driver nào đăng ký hay không. Thiếu một cái nghĩa là kubelet trên node đó chưa khởi tạo xong — chạy lại gate mở đầu; nếu vẫn thiếu, ghi lại và bỏ qua gate này, phần còn lại của B8 vẫn đọc được |
| `HEALTH_N` khác 0 dù không có CSI driver | `grep '^kubelet_volume_stats_health' ~/lab-evidence/6b/b9-kubelet-metrics.txt` | Ghi lại nguyên văn các dòng metric và label của chúng; đọc lại B1 xem `CSIDRIVER_N` có thật sự bằng 0 không |
| Namespace `lab-6b` kẹt `Terminating` | `kubectl get pods,pvc -n lab-6b`; `kubectl get pvc -n lab-6b -o yaml \| grep finalizers -A3` | Còn Pod đang dùng PVC nên finalizer `kubernetes.io/pvc-protection` giữ lại. Xóa Pod trước rồi chờ; **không** cưỡng chế finalizer |
| `PV_NOW` không giảm về `PV_BEFORE` | `kubectl get pv` cột `RECLAIM POLICY` và `STATUS`; `~/lab-evidence/6b/b10-pv-truoc-khi-xoa.txt` | PV `Released` với `reclaimPolicy: Retain` không tự biến mất. Xóa tay **đúng những PV có `claimRef` trỏ vào namespace `lab-6b`**, không đụng PV nào khác. Nếu thư mục trên node còn sót sau khi PV đã biến mất, dọn theo đúng cách mà [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md) mô tả ở mục cleanup của nó, dùng đường dẫn ghi trong file evidence |
| `NS_AFTER` khác `NS_BEFORE` ở gate cuối | So hai chuỗi bằng mắt; `kubectl get namespace` | Nếu thiếu namespace của provisioner thì bạn đã xóa nhầm hạ tầng của `03-storage-ready` — restore cả ba VM về mốc đó, không cài lại tay |

---

## 5. Nguồn chính thức

- [Volume Attributes Classes](https://kubernetes.io/docs/concepts/storage/volume-attributes-classes/)
- [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [Volume Snapshot Classes](https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/)
- [CSI Volume Cloning](https://kubernetes.io/docs/concepts/storage/volume-pvc-datasource/)
- [Volume Populators and Data Sources](https://kubernetes.io/docs/concepts/storage/volume-populators-and-data-sources/)
- [Storage Capacity](https://kubernetes.io/docs/concepts/storage/storage-capacity/)
- [Node-specific Volume Limits](https://kubernetes.io/docs/concepts/storage/storage-limits/)
- [Volume Health Monitoring](https://kubernetes.io/docs/concepts/storage/volume-health-monitoring/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [CSI Drivers — danh sách driver và tính năng từng driver hỗ trợ](https://kubernetes-csi.github.io/docs/drivers.html)
