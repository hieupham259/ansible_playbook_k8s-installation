# Lab 6b — Snapshot và volume nâng cao

> **Điểm bắt đầu:** snapshot `03-storage-ready` — mốc do [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md)
> tạo, xem [chuỗi snapshot](README.md#3-chuỗi-snapshot). Repo `~/bookstore` ở tag `lab-6a` (hoặc
> bundle `~/bookstore-lab-6a.bundle` nếu bạn vừa restore snapshot).
> **Điểm kết thúc:** teardown trả cluster về đúng `03-storage-ready`, **không tạo snapshot mới** và
> không cài thêm thành phần hạ tầng nào. Repo `~/bookstore` có tag `lab-6b`; bundle repo **và** bản
> dump DB mới nhất nằm ngoài VM.
> **Sprint dự án:** Sprint 9 — Chiến lược sao lưu của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 6a — PV, PVC và StorageClass](LAB-6A-PV-PVC-VA-STORAGECLASS.md).
> **Lab sau:** [Lab 7a — Lập lịch và eviction](LAB-7A-LAP-LICH-VA-EVICTION.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng **phần còn lại** của
[Giai đoạn 6 — Lưu trữ](../00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ): tám bài 97, 99, 100, 101,
102, 103, 104, 105. Phần cốt lõi PV/PVC/StorageClass thuộc Lab 6a, không lặp lại ở đây. Mọi ticket
dựa vào hợp đồng kỹ thuật ở [mục 4 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật) và bố cục
repo ở [mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint). Kế hoạch gốc của sprint nằm ở
[mục 7.2](DU-AN-BOOKSTORE.md#sprint-9--lab-6b-chiến-lược-sao-lưu).

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). Image của dự án nằm
ở [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00); lab này không chép lại
con số phiên bản nào trong phần văn bản và **không cài thêm gì**: không CSI driver, không snapshot
controller, không CRD, không sửa cấu hình apiserver hay kubelet.

**Lab này là chỗ trả [nợ lab #5](README.md#5-sổ-nợ-lab) — ảnh chụp nhanh và nhân bản volume — nếu
đủ điều kiện.** Sổ nợ ghi rõ điều kiện: CSI driver đang dùng phải hỗ trợ snapshot. B1 kiểm điều
kiện đó bằng API của cluster và chốt **nhánh A** (đủ điều kiện, chụp snapshot thật ở B2) hoặc
**nhánh B** (không đủ, ghi hồ sơ nợ). Với local-path-provisioner của baseline, kết quả trung thực là
nhánh B — và chính kết quả đó là đầu vào cho quyết định thiết kế của sprint: Bookstore sao lưu bằng
`pg_dump`. **Đọc lại ba bài [99](../99-volume-snapshots-vi.md),
[100](../100-volume-snapshot-classes-vi.md), [101](../101-volume-pvc-datasource-vi.md) trước khi làm
B1** — mục *Giới thiệu* của cả ba chứa toàn bộ danh sách điều kiện mà B1 kiểm.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi Bookstore có **chiến lược sao lưu đã được quyết định, ghi thành tài liệu và
diễn tập khôi phục** — **và** bạn chứng minh, giải thích được:

- Năng lực thật của tầng lưu trữ mà `data-db-0` đang dùng: provisioner có phải CSI driver không,
  ba CRD snapshot và snapshot controller có không, VolumeSnapshotClass nào khớp driver — và kết
  luận nợ #5 trả được hay không **bằng bằng chứng chứ không bằng phỏng đoán**.
- Ánh xạ `VolumeSnapshot`, `VolumeSnapshotContent`, `VolumeSnapshotClass` sang PVC, PV,
  StorageClass, kèm phạm vi namespace hay cluster; vì sao ba API đó không tự có và ai chịu trách
  nhiệm cài chúng.
- VolumeAttributesClass khác StorageClass ở điểm căn bản nào, và `parameters` của một class đã tạo
  là bất biến.
- Nhân bản PVC bằng `dataSource` có những ràng buộc nào, và trên provisioner **không phải** CSI thì
  không có lỗi nào — chỉ có một volume rỗng. `dataSource` và `dataSourceRef` khác nhau ở hai điểm
  nào.
- Ba điều kiện để scheduler xét dung lượng lưu trữ, ai công bố giới hạn số volume theo node, ai báo
  tình trạng volume — và vì sao cả ba đều vắng mặt với `data-db-0` trên `lab-k8s-worker2`.
- Vì sao Bookstore dùng sao lưu logic `pg_dump` thay cho VolumeSnapshot trên cluster này, và cái giá
  phải trả: RPO theo lịch, thời gian khôi phục tăng theo dữ liệu, role không nằm trong bản dump.
- Bản dump chỉ thành bản sao lưu khi nằm **ngoài cluster**: lấy ra từ đúng node giữ PV, kiểm
  checksum, chép tới máy host; RPO thật khi mất cả cluster bằng bao nhiêu.
- Khôi phục bản dump vào một DB mới, cách ly, đối chiếu với DB gốc và đo thời gian; vì sao `Ready`
  không có nghĩa là đã khôi phục; mật khẩu role sau khi khôi phục đến từ đâu.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 6b | Ticket kiểm chứng |
| --- | --- |
| [97 — Lớp thuộc tính Volume](../97-volume-attributes-classes-vi.md) | B1 — API `volumeattributesclasses` có trong cluster, PVC có trường chọn class, `parameters` của class đã tạo là bất biến; câu hỏi thiết kế về đổi hiệu năng đĩa của `data-db-0` |
| [99 — Ảnh chụp nhanh Volume](../99-volume-snapshots-vi.md) | B1 — bốn điều kiện, ba API là CRD chứ không phải core API, ánh xạ phạm vi; B2 nhánh A — chụp, `readyToUse`, ràng buộc một-một, khôi phục qua `dataSource`, `DeletionPolicy`; B4 — "sao lưu trước khi chỉnh sửa" của snapshot so với `pg_dump` |
| [100 — Các lớp Volume Snapshot](../100-volume-snapshot-classes-vi.md) | B1 — không có CRD thì tạo VolumeSnapshotClass thất bại, `driver` phải khớp provisioner; B2 nhánh A — `deletionPolicy` quyết định số phận `VolumeSnapshotContent` |
| [101 — Nhân bản CSI Volume](../101-volume-pvc-datasource-vi.md) | B2 — PVC nguồn đã bound và không đang được dùng, cùng namespace, dung lượng đích, clone độc lập, và kết quả thật khi provisioner không phải CSI; B6 — vì sao không "nhân bản" được dữ liệu sang namespace diễn tập |
| [102 — Volume Populator và Nguồn dữ liệu](../102-volume-populators-vi.md) | B2 — `dataSource` im lặng bỏ qua giá trị không hợp lệ, `dataSourceRef` báo lỗi, hai trường được gán khớp nhau và bất biến, `dataSourceRef` nhận kiểu tùy chỉnh; B6 — khung populator cho bước nạp bản dump vào volume mới |
| [103 — Dung lượng lưu trữ](../103-storage-capacity-vi.md) | B3 — API `CSIStorageCapacity` và ai tạo đối tượng, ba điều kiện đồng thời với `bookstore-retain`, PVC 500Gi được cấp trên đĩa nhỏ hơn nhiều |
| [104 — Giới hạn volume theo từng Node](../104-storage-limits-vi.md) | B3 — `CSINode.spec.drivers[].allocatable.count`, `attachable-volumes-*` trong `allocatable` của Node, trường nào của `CSIDriver` có trong schema của cluster |
| [105 — Giám sát tình trạng volume](../105-volume-health-monitoring-vi.md) | B3 — metric `kubelet_volume_stats_health_status_abnormal` cho `data-db-0` đọc từ kubelet qua node proxy, Event trên PVC |

Các phần dưới đây **không kiểm chứng được trên cluster lab**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài [99](../99-volume-snapshots-vi.md), chụp và khôi phục snapshot **thật**; bài [101](../101-volume-pvc-datasource-vi.md), nhân bản **thật** | Chỉ chạy ở nhánh A. Ở nhánh B phần này giữ nguyên trong [sổ nợ lab](README.md#5-sổ-nợ-lab) (nợ #5) và B1 ghi lại đúng điều kiện còn thiếu |
| Bài [99](../99-volume-snapshots-vi.md), *Bảo vệ PersistentVolumeClaim đang làm nguồn Snapshot* | Phải xóa PVC **đúng lúc** snapshot đang chụp dở. Ngay cả ở nhánh A cũng không ép được thời điểm đó mà không can thiệp vào driver |
| Bài [99](../99-volume-snapshots-vi.md), cấp phát sẵn (`snapshotHandle`, `volumeHandle`) và *Chuyển đổi volume mode của một Snapshot* | Cần định danh của một snapshot **có thật trên backend lưu trữ**; cluster lab không có backend nào cấp được id đó |
| Bài [97](../97-volume-attributes-classes-vi.md), đổi `volumeAttributesClassName` của PVC đang chạy | Cần lưu trữ qua CSI **và** driver có triển khai `ModifyVolume`, cộng external-resizer. B1 kiểm đúng hai điều kiện đó và ghi kết quả |
| Bài [102](../102-volume-populators-vi.md), populator tùy chỉnh và *Nguồn dữ liệu giữa các namespace* | Populator là controller **bên ngoài** phải cài thêm, và nguồn dữ liệu của nó là custom resource — CRD học ở giai đoạn 14. Phần giữa các namespace còn alpha và cần ReferenceGrant của Gateway API |
| Bài [103](../103-storage-capacity-vi.md), *Lập lịch lại* | Cần một driver có theo dõi dung lượng và cố tình làm cạn chỗ trên một node để driver báo lỗi tạo volume |
| Bài [104](../104-storage-limits-vi.md), `MutableCSINodeAllocatableCount`, `nodeAllocatableUpdatePeriodSeconds`, `preventPodSchedulingIfMissing` | Cần một CSI driver thật; riêng `preventPodSchedulingIfMissing` còn cần bật feature gate alpha trên control plane, việc thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). B3 chỉ đọc schema |
| Bài [105](../105-volume-health-monitoring-vi.md), Event tình trạng volume và cờ `enable-node-watcher` | Cần External Health Monitor controller chạy cạnh một CSI driver **có hiện thực** tính năng này |

Nợ sau sprint — nói được vì sao ở checkpoint:

| Loại | Mã | Trạng thái sau Lab 6b |
| --- | --- | --- |
| Nợ lab | #5 — snapshot và nhân bản volume | **vẫn mở** ở nhánh B; trả bằng cách chạy lại B1, B2 nhánh A khi cluster có CSI driver hỗ trợ snapshot |
| Nợ kỹ thuật | `TD-10` — bản backup nằm cùng cluster với DB | **chấp nhận có điều kiện**: chép ra ngoài bằng tay theo `docs/runbook-backup.md` ([mục 8 của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án)) |

### 1.2. Thời lượng

4–5 giờ. Phần cốt lõi (B0–B6 và B8) khoảng 4 giờ; B7 là diễn tập tùy chọn ⭐, thêm khoảng 30–40
phút. B2, B3 và B6 có bước chờ controller hội tụ; thời gian chờ **phụ thuộc cấu hình cluster** nên
mọi bước chờ viết dưới dạng vòng lặp hoặc `kubectl wait` có điều kiện dừng. Có thể chia hai phiên:
phiên 1 làm B0–B3, phiên 2 làm B4–B8. Giữa hai phiên cứ để nguyên cluster; đầu phiên 2 chạy lại gate
ngắn A5.5 và `./scripts/deploy.sh` — dữ liệu nằm trên PV `Retain` nên còn nguyên.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác. Lệnh `ssh <node> 'sudo ...'`
  tới worker **chỉ để đọc**.
- **Fault injection chỉ trên `lab-k8s-worker2`.** `db-0` và `data-db-0` nằm ở đó theo
  [thiết kế dự án](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch). Mọi Pod thử nghiệm, DB
  diễn tập ở B6 và sự cố cố ý ở B7 ⭐ đều ghim `lab-k8s-worker2` bằng `nodeSelector`.
- Repo dự án: `~/bookstore`. Manifest thử nghiệm và diễn tập: `~/lab-work/6b/`. Bằng chứng:
  `~/lab-evidence/6b/`. **Bản dump ngoài cluster**: `~/bookstore-backups/` trên master (quyền `700`)
  rồi thư mục `lab-backup` trên máy host.
- Bản dump chứa dữ liệu khách hàng. Nó **không bao giờ** nằm trong `~/bookstore` (git, bundle) hay
  `~/lab-evidence/` — bằng chứng chỉ ghi checksum, số dòng, kích thước. Không in mật khẩu, không ghi
  mật khẩu vào bằng chứng; không chạy `set -x` khi làm với Secret.
- Lab chỉ dùng kiến thức nhóm 6b và các nhóm trước: Pod, probe, init script của image (3a), ConfigMap,
  Secret (3b), StatefulSet, Job, CronJob (4b), Service, NetworkPolicy (5), PV, PVC, StorageClass (6a).
  **Không** dùng node affinity, PriorityClass mới, ResourceQuota (7), RBAC (9), CRD (14).
- **Lab tuyệt đối không cài thêm gì:** không CSI driver, không snapshot controller, không CRD, không
  Helm chart, không bật feature gate. Thiếu điều kiện thì ghi nhận thiếu, không đi vá.
- Sprint này **không đổi** manifest trong `k8s/` và **không đổi** `scripts/deploy.sh`,
  `scripts/teardown.sh`. Repo chỉ thay đổi ở `docs/` và `tests/smoke.sh`.
- Object thử nghiệm của ticket *Điều tra*: namespace `bookstore-thu` (mang label dự án để teardown
  dọn được nếu bạn quên), object cấp cluster tên bắt đầu bằng `bookstore-thu-`. PVC thử nghiệm dùng
  StorageClass mặc định `local-path` (`reclaimPolicy: Delete`), **không** dùng `bookstore-retain` —
  class `Retain` để lại PV và thư mục trên node. Xóa hết khi xong ticket.
- Namespace diễn tập `bookstore-restore-test` (B6) mang label dự án vì cùng lý do.
- Không ghi vào `data-db-0` và `backup-store` ngoài đường của chính hệ thống, trừ sự cố cố ý ở B7 ⭐.
  Pod đọc bản dump `backup-reader` mount `backup-store` ở chế độ chỉ đọc.
- **Không dùng `nodeName` cho Pod tiêu thụ PVC.** Với `WaitForFirstConsumer`, node của volume do
  scheduler chọn; Pod khai `nodeName` bỏ qua scheduler và PVC không bao giờ được cấp. Dùng
  `nodeSelector`.
- Mỗi script nghiệm thu chạy trong một shell liền mạch. Giá trị dùng chung giữa các ticket lưu ở
  `~/lab-evidence/6b/*.env` để mở shell mới không mất.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong. Dòng `STOP:` nghĩa là dừng đúng bước đó và ghi lại, không đi vòng.

---

# Phần B — Sprint 9: Chiến lược sao lưu

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `03-storage-ready`, chốt các con số nền của tầng lưu trữ để
gate cuối có cái đối chiếu, và dựng lại hệ thống của sprint 8 **chỉ từ repo** ở tag `lab-6a`.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Chốt nền tầng lưu trữ **trước** khi triển khai dự án. File `b0-baseline.env` là thứ gate cuối ở mục
đóng sprint đọc lại; đừng xóa nó.

```bash
EV=~/lab-evidence/6b
mkdir -p ~/lab-work/6b "$EV"
{
  printf "SC_DEFAULT=%s\n" "$(kubectl get storageclass --no-headers | awk '/\(default\)/{print $1}')"
  printf "SC_BEFORE='%s'\n" "$(kubectl get storageclass -o name | sort | tr '\n' ' ')"
  printf "NS_BEFORE='%s'\n" "$(kubectl get namespace -o name | sort | tr '\n' ' ')"
  printf "PV_BEFORE=%s\n" "$(kubectl get pv --no-headers 2>/dev/null | wc -l)"
  printf "PVC_BEFORE=%s\n" "$(kubectl get pvc -A --no-headers 2>/dev/null | wc -l)"
  printf "CSIDRIVER_BEFORE=%s\n" "$(kubectl get csidriver --no-headers 2>/dev/null | wc -l)"
  printf "SNAPCRD_BEFORE=%s\n" "$(kubectl get crd -o name 2>/dev/null | grep -c '\.snapshot\.storage\.k8s\.io$')"
  printf "VAC_BEFORE=%s\n" "$(kubectl get volumeattributesclass --no-headers 2>/dev/null | wc -l)"
} > "$EV/b0-baseline.env"
cat "$EV/b0-baseline.env"
. "$EV/b0-baseline.env"

[ "$SC_DEFAULT" = 'local-path' ] && [ "$SC_BEFORE" = 'storageclass.storage.k8s.io/local-path ' ] \
  && echo 'PASS: dung mot StorageClass, local-path la mac dinh'
[ "$PV_BEFORE" -eq 0 ] && [ "$PVC_BEFORE" -eq 0 ] && echo 'PASS: chua co PV, PVC nao'
R="$(kubectl -n local-path-storage get deployment local-path-provisioner -o jsonpath='{.status.readyReplicas}')"
[ "${R:-0}" -ge 1 ] && echo 'PASS: local-path-provisioner dang chay'
# Liệt kê /opt/local-path-provisioner trên worker. Thư mục gốc chưa từng có thì in rỗng, thoát 0;
# lỗi ssh, sudo hay find thì thoát khác 0 — "không đọc được" không bao giờ được hiểu thành "rỗng".
lp_ls() {
  ssh -o BatchMode=yes "$1" \
    'sudo -n test ! -e /opt/local-path-provisioner || sudo -n find /opt/local-path-provisioner -mindepth 1 -maxdepth 1'
}
for N in lab-k8s-worker1 lab-k8s-worker2; do
  if ! L="$(lp_ls "$N")"; then echo "STOP: khong kiem duoc $N (ssh, sudo hoac find loi)"
  elif [ -z "$L" ]; then echo "PASS: $N khong con thu muc du lieu cu"
  else echo "FAIL: $N con $(printf '%s\n' "$L" | grep -c .) muc trong /opt/local-path-provisioner"; fi
done
```

Repo và thư mục làm việc. Nếu vừa restore snapshot mà repo không có, chép
`bookstore-lab-6a.bundle` từ máy host lên master theo
[mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm) trước:

```bash
test -d ~/bookstore/.git || git clone ~/bookstore-lab-6a.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-6a' && echo 'PASS: HEAD dung tag lab-6a'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lan truoc' || echo 'PASS: cluster chua co du an'
```

Kéo sẵn hai image mà sprint này dùng nhiều nhất lên hai worker. Tag lấy đúng từ
[A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và
[A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00):

```bash
for N in lab-k8s-worker1 lab-k8s-worker2; do
  for I in docker.io/library/busybox:1.37 docker.io/library/postgres:17.11; do
    ssh "$N" "sudo crictl pull $I" >/dev/null && echo "PASS: $N co $I" || echo "FAIL: $N khong keo duoc $I"
  done
done
```

Dựng lại hệ thống của sprint 8 bằng script của chính sprint đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/6b/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/6b/b0-smoke-lab-6a.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/6b/b0-smoke-lab-6a.txt
kubectl -n bookstore get pvc
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; năm dòng `PASS:` của khối tầng lưu trữ (hai dòng cuối
mỗi worker một dòng); ba dòng `PASS:` về repo và cluster; bốn dòng `PASS: ... co ...`;
`deploy exit=0`, `smoke exit=0` và file smoke không có dòng `FAIL:`; `kubectl -n bookstore get pvc`
liệt kê `data-db-0` và `backup-store`.

`deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint 8 **chưa đủ** — có thứ chỉ tồn tại trên
cluster cũ. Sửa repo ở tag `lab-6a` trước, không sửa tay trên cluster
([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)). Khối tầng lưu trữ fail nghĩa là cluster
không ở đúng `03-storage-ready` — xem mục 4.

## B1. [BS-6B-01] Năng lực snapshot và lớp thuộc tính của tầng lưu trữ

**Loại:** Điều tra.

**Bối cảnh:** hệ thống cũ sao lưu bằng vòng lặp shell cùng máy với DB (`P6`). Sprint 8 đã đưa dữ
liệu lên PV `Retain` và bản dump lên PVC `backup-store`. Trước khi chốt chiến lược sao lưu, đội cần
biết tầng lưu trữ có sẵn **ảnh chụp nhanh volume** không — cách sao lưu mà Kubernetes chuẩn hóa qua
CSI, và bài 99 nói là dành cho đúng việc "sao lưu cơ sở dữ liệu trước khi chỉnh sửa". Câu trả lời
phải đến từ API của cluster, không từ trí nhớ hay từ tên provisioner.

**Yêu cầu:**

1. Xác định provisioner của StorageClass mà `data-db-0` đang dùng, rồi kiểm **bốn điều kiện** để
   chụp được snapshot theo bài 99 và 100: provisioner là một CSI driver đã đăng ký với cluster; ba
   CRD của snapshot đã cài; snapshot controller đang chạy; có một VolumeSnapshotClass khai đúng
   driver đó. Chốt **nhánh A** (đủ cả bốn) hoặc **nhánh B** (thiếu ít nhất một).
2. Chứng minh bằng API của cluster: PVC thuộc namespace còn PV và StorageClass thuộc cluster; số API
   trong nhóm `snapshot.storage.k8s.io` đúng bằng số CRD snapshot đếm được — tức ba API đó không
   thuộc core API.
3. Viết `~/lab-work/6b/thu-snapclass.yaml`: một VolumeSnapshotClass tên `bookstore-thu-snapclass`
   khai đúng driver của tầng lưu trữ, xóa snapshot bên dưới khi xóa snapshot. Chứng minh API server
   từ chối nó khi thiếu CRD — **mà không ghi object nào** vào cluster.
4. Viết `~/lab-work/6b/thu-vac.yaml`: một VolumeAttributesClass tên `bookstore-thu-silver` cùng
   driver, đúng một tham số tùy ý. Chứng minh: API có kiểu này, PVC có trường chọn class, và tham
   số của class đã tạo **không sửa được**. Xóa class khi xong.
5. Mở ADR `docs/adr/NNNN-sao-luu-logic.md` (`NNNN` là số tiếp theo trong `docs/adr/`, trạng thái
   *đề xuất*) và viết phần **Bối cảnh — snapshot và lớp thuộc tính**: bảng bốn điều kiện, kết quả
   thật trên cluster, đường dẫn file bằng chứng. B2, B3 viết tiếp phần Bối cảnh; B4 hoàn thiện ADR.

**Ràng buộc:** không cài CSI driver, CRD hay snapshot controller để ép sang nhánh A — việc đó đổi
hạ tầng vĩnh viễn, mà lab này trả cluster về `03-storage-ready`. VolumeSnapshotClass chỉ thử bằng
server dry-run. Không gắn VolumeAttributesClass vào PVC nào.

**Đọc lại:**

- Bài [99](../99-volume-snapshots-vi.md): [*Giới thiệu*](../99-volume-snapshots-vi.md#giới-thiệu-introduction),
  [*Ràng buộc*](../99-volume-snapshots-vi.md#ràng-buộc-binding),
  [*Các VolumeSnapshot*](../99-volume-snapshots-vi.md#các-volumesnapshot-volumesnapshots).
- Bài [100](../100-volume-snapshot-classes-vi.md):
  [*Tài nguyên VolumeSnapshotClass*](../100-volume-snapshot-classes-vi.md#tài-nguyên-volumesnapshotclass-the-volumesnapshotclass-resource)
  — đặc biệt ghi chú về CRD,
  [*Các phụ thuộc của VolumeSnapshotClass*](../100-volume-snapshot-classes-vi.md#các-phụ-thuộc-của-volumesnapshotclass-volumesnapshotclass-dependencies).
- Bài [97](../97-volume-attributes-classes-vi.md): đoạn mở đầu,
  [*API VolumeAttributesClass*](../97-volume-attributes-classes-vi.md#api-volumeattributesclass-the-volumeattributesclass-api),
  [*Trình thay đổi kích thước*](../97-volume-attributes-classes-vi.md#trình-thay-đổi-kích-thước-resizer).

**Bàn giao:** `~/lab-work/6b/thu-snapclass.yaml`, `~/lab-work/6b/thu-vac.yaml`; phần Bối cảnh đầu
tiên của `docs/adr/NNNN-sao-luu-logic.md`; `~/lab-evidence/6b/b1-*`.

**Nghiệm thu:**

- `PASS:` đọc được provisioner của StorageClass mà `data-db-0` dùng; bốn điều kiện được đếm và ghi
  vào `~/lab-evidence/6b/b1-nang-luc.env` cùng nhánh đã chốt.
- `PASS:` PVC thuộc namespace; PV và StorageClass thuộc cluster.
- `PASS:` số API của nhóm `snapshot.storage.k8s.io` bằng số CRD snapshot.
- `PASS:` VolumeSnapshotClass bị từ chối khi không có CRD (hoặc được chấp nhận khi có), và sau bước
  này không có VolumeSnapshotClass `bookstore-thu-snapclass` nào trong cluster.
- `PASS:` PVC có trường `volumeAttributesClassName`; `bookstore-thu-silver` tạo được; sửa tham số bị
  API từ chối vì bất biến; class đã xóa.
- `PASS:` ADR có phần Bối cảnh nói về snapshot.

**Câu hỏi thiết kế** — trả lời trong phần Bối cảnh của `docs/adr/NNNN-sao-luu-logic.md`:

1. Cluster thiếu điều kiện nào trong bốn điều kiện, và lệnh nào chỉ ra điều đó? Nợ #5 trả hay chưa,
   và cần chính xác những gì để trả?
2. Ba API của snapshot ứng với ba object nào bạn đã dùng ở Lab 6a, cái nào thuộc namespace, cái nào
   thuộc cluster? Trong cấp phát động, ai tạo cái nào?
3. Ai chịu trách nhiệm cài CRD và snapshot controller — CSI driver hay bản phân phối Kubernetes? Vì
   sao lab không được tự cài để có snapshot?
4. Sau này muốn nâng hiệu năng đĩa của `data-db-0` mà không tạo lại volume: VolumeAttributesClass có
   giúp được không trên cluster này? Nếu `silver` đang dùng cần đổi tham số, bạn làm gì?

<details>
<summary>Gợi ý</summary>

1. StorageClass của `data-db-0` nằm ngay trong spec của PVC đó. Provisioner nằm trong StorageClass.
2. Bốn phép kiểm: `kubectl get csidriver`, `kubectl get crd` lọc theo `snapshot`,
   `kubectl api-resources --api-group=snapshot.storage.k8s.io`, `kubectl get pods -A` lọc theo
   `snapshot-controller`. `kubectl get csinodes` cho biết node nào quảng bá driver nào.
3. Phạm vi của một kiểu: `kubectl api-resources --namespaced=true -o name` và `--namespaced=false`.
4. `kubectl apply --dry-run=server` gửi request để API server xác thực đầy đủ nhưng không ghi gì.
5. VolumeAttributesClass có `apiVersion: storage.k8s.io/v1`, hai trường `driverName` và
   `parameters`; API đòi ít nhất một tham số. `kubectl explain volumeattributesclass` nếu không chắc.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
EV=~/lab-evidence/6b; W=~/lab-work/6b
cd ~/bookstore

# 1. Tầng lưu trữ của data-db-0 và bốn điều kiện
SC="$(kubectl -n bookstore get pvc data-db-0 -o jsonpath='{.spec.storageClassName}')"
PROVISIONER="$(kubectl get storageclass "$SC" -o jsonpath='{.provisioner}')"
[ -n "$PROVISIONER" ] && echo "PASS: data-db-0 dung StorageClass $SC, provisioner $PROVISIONER"

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
printf 'SC=%s\nPROVISIONER=%s\nIS_CSI=%s\nCSIDRIVER_N=%s\nCRD_N=%s\nSNAPCTRL_N=%s\nVSC=%s\nNHANH=%s\n' \
  "$SC" "$PROVISIONER" "$IS_CSI" "$CSIDRIVER_N" "$CRD_N" "$SNAPCTRL_N" "$VSC" "$NHANH" > "$EV/b1-nang-luc.env"
{
  echo "=== $(date -Is) — nang luc snapshot cua tang luu tru ==="
  cat "$EV/b1-nang-luc.env"
  echo '--- csidriver ---'; kubectl get csidriver 2>&1
  echo '--- csinodes ---'
  kubectl get csinodes -o custom-columns='NODE:.metadata.name,DRIVERS:.spec.drivers[*].name' 2>&1
  echo '--- api snapshot.storage.k8s.io ---'; kubectl api-resources --api-group=snapshot.storage.k8s.io 2>&1
} > "$EV/b1-nang-luc.txt"
grep -Eq '^NHANH=(A|B)$' "$EV/b1-nang-luc.env" && echo "PASS: da kiem bon dieu kien, NHANH=$NHANH"
if [ "$NHANH" = A ]; then
  echo 'NHANH A: du bon dieu kien — lam them phan nhanh A o B2, no #5 duoc tra trong lab nay'
else
  echo "NHANH B: IS_CSI=$IS_CSI CRD=$CRD_N/3 controller=$SNAPCTRL_N class=${VSC:-khong} — no #5 VAN MO"
fi

# 2. Phạm vi và core API
{ kubectl api-resources --namespaced=true  -o name | grep -qx 'persistentvolumeclaims' && \
  kubectl api-resources --namespaced=false -o name | grep -qx 'persistentvolumes' && \
  kubectl api-resources --namespaced=false -o name | grep -qx 'storageclasses.storage.k8s.io'; } \
  && echo 'PASS: PVC thuoc namespace; PV va StorageClass thuoc cluster'
SNAP_API_N="$(kubectl api-resources --api-group=snapshot.storage.k8s.io -o name 2>/dev/null | wc -l)"
[ "$SNAP_API_N" -eq "$CRD_N" ] \
  && echo "PASS: nhom snapshot.storage.k8s.io co $SNAP_API_N API, dung bang $CRD_N CRD — khong thuoc core API"

# 3. VolumeSnapshotClass
awk '/^driver:/{print $2}' "$W/thu-snapclass.yaml" | grep -qx "$PROVISIONER" \
  && echo 'PASS: thu-snapclass khai driver khop provisioner'
if [ "$CRD_N" -ge 3 ]; then
  kubectl apply --dry-run=server -f "$W/thu-snapclass.yaml" > "$EV/b1-snapclass.txt" 2>&1 \
    && echo 'PASS: co CRD nen API chap nhan VolumeSnapshotClass (dry-run, khong ghi gi)'
else
  kubectl apply --dry-run=server -f "$W/thu-snapclass.yaml" > "$EV/b1-snapclass.txt" 2>&1 \
    && echo 'FAIL: API chap nhan du khong co CRD — quay lai buoc 1' \
    || echo 'PASS: khong co CRD nen tao VolumeSnapshotClass that bai'
fi
kubectl get volumesnapshotclass bookstore-thu-snapclass >/dev/null 2>&1 \
  && echo 'FAIL: co VolumeSnapshotClass bookstore-thu-snapclass trong cluster' \
  || echo 'PASS: khong object nao duoc ghi'

# 4. VolumeAttributesClass
kubectl explain persistentvolumeclaim.spec.volumeAttributesClassName > "$EV/b1-explain-vac.txt" 2>&1 \
  && echo 'PASS: PVC co truong volumeAttributesClassName'
VAC_API="$(kubectl api-resources --api-group=storage.k8s.io -o name | grep -c '^volumeattributesclasses\.')"
if [ "$VAC_API" -eq 1 ]; then
  kubectl apply -f "$W/thu-vac.yaml"
  [ "$(kubectl get volumeattributesclass bookstore-thu-silver -o jsonpath='{.driverName}')" = "$PROVISIONER" ] \
    && echo 'PASS: bookstore-thu-silver tao duoc — API khong hoi driver co ModifyVolume hay khong'
  kubectl patch volumeattributesclass bookstore-thu-silver --type=merge \
    -p '{"parameters":{"bookstore-thu-tier":"gold"}}' > "$EV/b1-vac-patch.txt" 2>&1 \
    && echo 'FAIL: sua duoc parameters — xem muc 4' \
    || { grep -qi 'immutable' "$EV/b1-vac-patch.txt" && echo 'PASS: parameters cua class da tao la bat bien'; }
  kubectl delete volumeattributesclass bookstore-thu-silver --wait=true
  kubectl get volumeattributesclass bookstore-thu-silver >/dev/null 2>&1 \
    && echo 'FAIL: class thu nghiem con' || echo 'PASS: da xoa class thu nghiem'
else
  echo 'STOP: cluster tat API VolumeAttributesClass — ghi vao ADR, bo qua phan nay'
fi

# 5. ADR
ADR="$(ls docs/adr/*-sao-luu-logic.md 2>/dev/null | head -n1)"
[ -n "$ADR" ] && grep -q '^## Bối cảnh' "$ADR" && grep -qi 'snapshot' "$ADR" \
  && echo "PASS: $ADR co phan Boi canh ve snapshot"
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/6b/thu-snapclass.yaml — driver lấy đúng provisioner đọc được ở bước 1
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: bookstore-thu-snapclass
driver: rancher.io/local-path
deletionPolicy: Delete
```

```yaml
# ~/lab-work/6b/thu-vac.yaml
apiVersion: storage.k8s.io/v1
kind: VolumeAttributesClass
metadata:
  name: bookstore-thu-silver
driverName: rancher.io/local-path
parameters:
  bookstore-thu-tier: "silver"
```

Khung ADR, phần đầu — viết bằng lời của bạn và số liệu của cluster bạn:

```markdown
# NNNN — Sao lưu logic bằng pg_dump thay cho VolumeSnapshot

- Trạng thái: đề xuất
- Sprint: Lab 6b, ticket BS-6B-01 → BS-6B-04

## Bối cảnh

### Snapshot và lớp thuộc tính (BS-6B-01)

| Điều kiện (bài 99, 100) | Cluster lab | Bằng chứng |
| --- | --- | --- |
| Provisioner của bookstore-retain là CSI driver | không: rancher.io/local-path không có CSIDriver | ~/lab-evidence/6b/b1-nang-luc.txt |
| Ba CRD snapshot | 0/3 | như trên |
| Snapshot controller | không có Pod nào | như trên |
| VolumeSnapshotClass khớp driver | không tạo được — API không biết kiểu này | ~/lab-evidence/6b/b1-snapclass.txt |

Kết luận: nhánh B, nợ lab #5 vẫn mở.
```

Ý chính cho câu hỏi thiết kế:

1. Thiếu cả bốn trên baseline: `kubectl get csidriver` không có `rancher.io/local-path`, không CRD,
   không controller, nên cũng không có class nào. Nợ #5 trả khi có một CSI driver hỗ trợ snapshot làm
   provisioner, ba CRD, snapshot controller cộng sidecar csi-snapshotter, và một VolumeSnapshotClass
   khớp driver — rồi chạy lại B1 và phần nhánh A của B2.
2. `VolumeSnapshot` ↔ PVC (yêu cầu của người dùng, thuộc namespace); `VolumeSnapshotContent` ↔ PV
   (tài nguyên trong cluster, thuộc cluster); `VolumeSnapshotClass` ↔ StorageClass (thuộc cluster).
   Cấp phát động: người dùng tạo `VolumeSnapshot`, snapshot controller tạo `VolumeSnapshotContent`,
   sidecar csi-snapshotter gọi `CreateSnapshot` xuống endpoint CSI. Ràng buộc là một-một.
3. Bài 99 và ghi chú của bài 100: cài CRD và snapshot controller là trách nhiệm của **bản phân phối**.
   Cài chúng là đổi hạ tầng của cluster — việc của một lab tạo mốc snapshot mới, không phải của
   sprint trả cluster về `03-storage-ready`.
4. Không. VolumeAttributesClass chỉ dùng được với lưu trữ qua CSI và driver có `ModifyVolume`; cả hai
   đều không có. API vẫn cho tạo class — API không kiểm năng lực của driver. Tham số của class đã tạo
   bất biến: muốn đổi thì tạo class mới (`gold`) rồi trỏ PVC sang, đúng ví dụ của bài 97.

</details>

## B2. [BS-6B-02] Nguồn dữ liệu cho PVC mới: nhân bản và `dataSourceRef`

**Loại:** Điều tra.

**Bối cảnh:** hai đề xuất trong đội. Đề xuất một: "bỏ `pg_dump`, mỗi đêm **nhân bản** PVC
`data-db-0` — nhanh, không cần hiểu PostgreSQL". Đề xuất hai: "khi khôi phục, tạo một PVC mới **tự
nạp sẵn** dữ liệu thay vì nạp tay". Cả hai dựa vào cơ chế nguồn dữ liệu của PVC (bài 101, 102). Thử
trên PVC thử nghiệm có dữ liệu đánh dấu, không đụng vào `data-db-0`.

**Yêu cầu:**

1. `~/lab-work/6b/thu-ns.yaml`: namespace `bookstore-thu` mang label dự án. B3 và B6 dùng lại file
   này.
2. `~/lab-work/6b/thu-nguon.yaml`: PVC nguồn `thu-nguon` (StorageClass mặc định, 100Mi) và Pod
   `thu-ghi` chạy một lần trên `lab-k8s-worker2`, ghi dòng `bookstore-6b` vào `/data/marker.txt` của
   volume rồi kết thúc. Sau khi Pod xong thì xóa nó: nguồn **đã bound và không đang được dùng**.
3. Chỉ khi B1 chốt nhánh A: `~/lab-work/6b/thu-snap.yaml` chụp VolumeSnapshot `thu-snap` từ
   `thu-nguon` bằng class tìm được ở B1, khôi phục thành PVC `thu-tu-snap` và Pod `thu-doc-snap`.
   Chứng minh volume khôi phục có dòng đánh dấu và `deletionPolicy` quyết định số phận của
   `VolumeSnapshotContent`.
4. `~/lab-work/6b/thu-clone.yaml`: PVC `thu-clone` khai `thu-nguon` làm nguồn dữ liệu, thỏa mọi ràng
   buộc của bài 101 mà bạn kiểm soát được; Pod `thu-doc-clone` trên `lab-k8s-worker2` mount nó ở
   `/data`. Ghi lại volume đích **có** dòng đánh dấu hay không, và có Event lỗi nào không.
5. Ghi một file vào volume của clone; chứng minh nguồn không thấy file đó.
6. `~/lab-work/6b/thu-ds-sai.yaml`: PVC `thu-ds-sai` khai nguồn dữ liệu là ConfigMap
   `khong-ton-tai` bằng trường **cũ**. Chứng minh PVC vẫn được tạo và giá trị bị bỏ đi; rồi chứng
   minh **cùng giá trị đó** trong trường **mới** bị API từ chối. Thụt lề manifest hai dấu cách — script
   sinh bản trường mới từ file của bạn.
7. Trên `thu-clone`: hai trường nguồn dữ liệu mang cùng giá trị dù bạn chỉ khai một; sửa sau khi tạo
   bị từ chối.
8. `~/lab-work/6b/thu-dsr-cr.yaml`: PVC `thu-dsr-cr` tham chiếu kiểu tùy chỉnh `ExampleDataSource`
   của nhóm `example.storage.k8s.io`. Chứng minh API nhận; ghi lại PVC đứng ở đâu và Event của nó.
9. Xóa namespace `bookstore-thu`; mọi PV thử nghiệm biến mất theo.
10. Viết phần **Bối cảnh — nhân bản và nguồn dữ liệu** của ADR.

**Ràng buộc:** PVC thử nghiệm dùng StorageClass mặc định, **không** dùng `bookstore-retain`. Pod ghim
`lab-k8s-worker2` bằng `nodeSelector`, không `nodeName`. Không nhân bản `data-db-0` hay
`backup-store` — hai PVC đó đang được dùng và thuộc hệ thống thật. Pod thử nghiệm đặt
`terminationGracePeriodSeconds` nhỏ để xóa nhanh.

**Đọc lại:**

- Bài [101](../101-volume-pvc-datasource-vi.md): [*Giới thiệu*](../101-volume-pvc-datasource-vi.md#giới-thiệu-introduction),
  [*Cấp phát*](../101-volume-pvc-datasource-vi.md#cấp-phát-provisioning) — ghi chú về dung lượng,
  [*Sử dụng*](../101-volume-pvc-datasource-vi.md#sử-dụng-usage).
- Bài [102](../102-volume-populators-vi.md): đoạn mở đầu,
  [*Volume populator và nguồn dữ liệu*](../102-volume-populators-vi.md#volume-populator-và-nguồn-dữ-liệu-volume-populators-and-data-sources),
  [*Tham chiếu nguồn dữ liệu*](../102-volume-populators-vi.md#tham-chiếu-nguồn-dữ-liệu-data-source-references),
  [*Sử dụng volume populator*](../102-volume-populators-vi.md#sử-dụng-volume-populator-using-volume-populators).
- Bài [99](../99-volume-snapshots-vi.md):
  [*Cấp phát Volume từ Snapshot*](../99-volume-snapshots-vi.md#cấp-phát-volume-từ-snapshot-provisioning-volumes-from-snapshots),
  [*Xóa*](../99-volume-snapshots-vi.md#delete) — chỉ cần ở nhánh A.
- Bài [92](../92-persistent-volumes-vi.md#volume-cloning), mục *Nhân bản volume*; bài
  [96](../96-storage-classes-vi.md#volume-binding-mode) — vì sao không dùng `nodeName`.

**Bàn giao:** năm file `thu-*.yaml` (sáu ở nhánh A); phần Bối cảnh thứ hai của ADR;
`~/lab-evidence/6b/b2-*`.

**Nghiệm thu:**

- `PASS:` namespace `bookstore-thu` mang label dự án; `thu-nguon` dùng class mặc định, `Bound`, có
  dòng đánh dấu, không Pod nào đang dùng.
- `PASS:` (nhánh A) snapshot `readyToUse`, ràng buộc một-một, volume khôi phục có dòng đánh dấu,
  `deletionPolicy` thể hiện đúng.
- `PASS:` `thu-clone` giữ nguồn dữ liệu trỏ `thu-nguon`; volume đích có dòng đánh dấu **khi và chỉ
  khi** provisioner là CSI driver.
- `PASS:` clone và nguồn là hai PV, hai thư mục khác nhau; ghi vào clone không hiện ở nguồn.
- `PASS:` ConfigMap trong trường cũ bị bỏ đi mà PVC vẫn tạo được; cùng giá trị trong trường mới bị
  từ chối.
- `PASS:` hai trường nguồn dữ liệu của `thu-clone` khớp nhau; sửa bị từ chối.
- `PASS:` `thu-dsr-cr` giữ nguyên kiểu tùy chỉnh.
- `PASS:` namespace `bookstore-thu` đã xóa, không còn PV nào của nó.

**Câu hỏi thiết kế** — trả lời trong phần Bối cảnh của ADR:

1. Trên cluster này, đề xuất "nhân bản `data-db-0` mỗi đêm" dẫn tới gì? Vì sao kết quả đó nguy hiểm
   hơn một lỗi rõ ràng?
2. Giả sử cluster có CSI driver hỗ trợ nhân bản. Clone nằm ở namespace nào, trên backend nào so với
   nguồn? Nó có phải bản sao lưu không — mất node `lab-k8s-worker2` thì sao?
3. Manifest của dự án nên khai `dataSource` hay `dataSourceRef`? Dẫn kết quả của yêu cầu 6.
4. Đề xuất hai cần những thành phần nào để chạy trên Kubernetes, theo bài 102? Thành phần nào chưa
   học, và nó thuộc giai đoạn nào?

<details>
<summary>Gợi ý</summary>

1. Bỏ `storageClassName` thì PVC lấy class mặc định. Trường nguồn dữ liệu nằm ở `spec` của PVC, gồm
   `kind` và `name` (thêm `apiGroup` khi kiểu không thuộc core).
2. Ghi chú của bài 101: dung lượng xin của PVC đích bằng hoặc lớn hơn dung lượng nguồn. Đọc dung
   lượng nguồn ở `status.capacity.storage`.
3. Chờ Pod chạy một lần xong: `kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/<tên>`.
4. Đường dẫn thật của một PV `local-path` trên node nằm ở `spec.hostPath.path` của PV.
5. Ở nhánh A, PVC khôi phục khai nguồn là `VolumeSnapshot` của nhóm `snapshot.storage.k8s.io`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự viết đủ các file; script này chính là thí nghiệm</summary>

```bash
NS=bookstore-thu; EV=~/lab-evidence/6b; W=~/lab-work/6b
. "$EV/b1-nang-luc.env"
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
pvpath() { kubectl get pv "$1" -o jsonpath='{.spec.hostPath.path}{.spec.local.path}' 2>/dev/null; }

kubectl apply -f "$W/thu-ns.yaml"
[ "$(kubectl get namespace "$NS" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'bookstore' ] \
  && echo 'PASS: namespace bookstore-thu mang label du an'

# 1. Nguồn có dữ liệu đánh dấu, đã bound, không ai dùng
kubectl apply -f "$W/thu-nguon.yaml"
kubectl -n "$NS" wait --for=jsonpath='{.status.phase}'=Succeeded pod/thu-ghi --timeout=300s
kubectl -n "$NS" delete pod thu-ghi --wait=true
SRC_CAP="$(jp pvc/thu-nguon '{.status.capacity.storage}')"
PV_SRC="$(jp pvc/thu-nguon '{.spec.volumeName}')"; P_SRC="$(pvpath "$PV_SRC")"
USED="$(kubectl -n "$NS" get pods -o jsonpath='{range .items[*]}{.spec.volumes[*].persistentVolumeClaim.claimName}{" "}{end}' | grep -cw thu-nguon)"
if [ -n "$P_SRC" ]; then   # PV hostPath của local-path: đọc thẳng thư mục trên node, chỉ đọc
  ssh lab-k8s-worker2 "sudo grep -qx bookstore-6b '$P_SRC/marker.txt'" && SRC_OK=1 || SRC_OK=0
else                       # driver khác: dòng đánh dấu được kiểm gián tiếp qua bản khôi phục ở nhánh A
  SRC_OK=1
fi
{ [ "$(jp pvc/thu-nguon '{.status.phase}/{.spec.storageClassName}')" = 'Bound/local-path' ] && \
  [ "$USED" -eq 0 ] && [ "$SRC_OK" -eq 1 ]; } \
  && echo "PASS: thu-nguon Bound ($SRC_CAP, local-path), co dong danh dau, khong Pod nao dang dung"

# 2. Nhánh A: snapshot thật
if [ "$NHANH" = A ]; then
  kubectl apply -f "$W/thu-snap.yaml"
  for i in $(seq 1 60); do [ "$(jp volumesnapshot/thu-snap '{.status.readyToUse}')" = 'true' ] && break; sleep 3; done
  CONTENT="$(jp volumesnapshot/thu-snap '{.status.boundVolumeSnapshotContentName}')"
  { [ "$(jp volumesnapshot/thu-snap '{.status.readyToUse}')" = 'true' ] && \
    [ "$(kubectl get volumesnapshotcontent "$CONTENT" -o jsonpath='{.spec.volumeSnapshotRef.namespace}/{.spec.volumeSnapshotRef.name}')" = "$NS/thu-snap" ]; } \
    && echo 'PASS: snapshot san sang, rang buoc mot-mot voi mot VolumeSnapshotContent'
  kubectl -n "$NS" wait --for=condition=Ready pod/thu-doc-snap --timeout=300s
  kubectl -n "$NS" exec thu-doc-snap -- grep -qx 'bookstore-6b' /data/marker.txt \
    && echo 'PASS: volume khoi phuc tu snapshot co dong danh dau'
  POLICY="$(kubectl get volumesnapshotcontent "$CONTENT" -o jsonpath='{.spec.deletionPolicy}')"
  kubectl -n "$NS" delete volumesnapshot thu-snap --wait=true
  for i in $(seq 1 20); do kubectl get volumesnapshotcontent "$CONTENT" >/dev/null 2>&1 || break; sleep 3; done
  if [ "$POLICY" = Delete ]; then
    kubectl get volumesnapshotcontent "$CONTENT" >/dev/null 2>&1 \
      && echo 'FAIL: Delete nhung content van con' || echo 'PASS: deletionPolicy Delete — content mat cung snapshot'
  else
    kubectl get volumesnapshotcontent "$CONTENT" >/dev/null 2>&1 \
      && echo 'PASS: deletionPolicy Retain — content con lai, xoa tay' && kubectl delete volumesnapshotcontent "$CONTENT"
  fi
fi

# 3. Nhân bản
kubectl apply -f "$W/thu-clone.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/thu-doc-clone --timeout=300s
[ "$(jp pvc/thu-clone '{.spec.dataSource.kind}/{.spec.dataSource.name}')" = 'PersistentVolumeClaim/thu-nguon' ] \
  && echo 'PASS: API giu nguon du lieu tro thu-nguon'
if kubectl -n "$NS" exec thu-doc-clone -- test -f /data/marker.txt; then MARK=co; else MARK=khong; fi
kubectl -n "$NS" get events --field-selector involvedObject.name=thu-clone > "$EV/b2-clone-events.txt" 2>&1
echo "dong danh dau trong clone: $MARK | IS_CSI=$IS_CSI" | tee "$EV/b2-clone.txt"
if [ "$IS_CSI" -eq 1 ]; then
  [ "$MARK" = co ] && echo 'PASS: CSI driver sao chep noi dung — nhan ban that'
else
  [ "$MARK" = khong ] && echo 'PASS: provisioner khong phai CSI bo qua nguon — PVC Bound, Pod chay, volume RONG'
fi

# 4. Clone độc lập với nguồn
kubectl -n "$NS" exec thu-doc-clone -- sh -c 'echo chi-o-clone > /data/rieng.txt && sync'
PV_CL="$(jp pvc/thu-clone '{.spec.volumeName}')"; P_CL="$(pvpath "$PV_CL")"
if [ -n "$P_SRC" ] && [ -n "$P_CL" ]; then
  { [ "$PV_SRC" != "$PV_CL" ] && [ "$P_SRC" != "$P_CL" ] && \
    ssh lab-k8s-worker2 "sudo test -f '$P_CL/rieng.txt' && ! sudo test -e '$P_SRC/rieng.txt'"; } \
    && echo 'PASS: hai PV, hai thu muc khac nhau; ghi vao clone khong hien o nguon'
else
  [ "$PV_SRC" != "$PV_CL" ] && echo 'PASS: hai PVC gan hai PV khac nhau (kiem them bang mot Pod doc nguon)'
fi

# 5. dataSource bỏ qua, dataSourceRef từ chối — cùng một giá trị
kubectl apply -f "$W/thu-ds-sai.yaml"
[ -z "$(jp pvc/thu-ds-sai '{.spec.dataSource}')" ] && [ -n "$(jp pvc/thu-ds-sai '{.metadata.uid}')" ] \
  && echo 'PASS: ConfigMap trong dataSource bi bo qua nhu de trong — PVC van duoc tao'
sed 's/name: thu-ds-sai/name: thu-dsr-sai/; s/^  dataSource:$/  dataSourceRef:/' \
  "$W/thu-ds-sai.yaml" > "$W/thu-dsr-sai.yaml"
grep -q '^  dataSourceRef:$' "$W/thu-dsr-sai.yaml" \
  || echo 'FAIL: sed khong doi duoc dataSource — xem lai thut le cua thu-ds-sai.yaml'
kubectl apply --dry-run=server -f "$W/thu-dsr-sai.yaml" > "$EV/b2-dsr-sai.txt" 2>&1 \
  && echo 'FAIL: API chap nhan dataSourceRef kieu core khong phai PVC' \
  || echo 'PASS: cung gia tri do, dataSourceRef bi API tu choi'

# 6. Hai trường khớp nhau và bất biến
V1="$(jp pvc/thu-clone '{.spec.dataSource.kind}/{.spec.dataSource.name}')"
V2="$(jp pvc/thu-clone '{.spec.dataSourceRef.kind}/{.spec.dataSourceRef.name}')"
[ -n "$V1" ] && [ "$V1" = "$V2" ] && echo "PASS: chi khai mot truong, API server gan ca hai ($V1)"
kubectl -n "$NS" patch pvc thu-clone --type=merge \
  -p '{"spec":{"dataSourceRef":{"kind":"PersistentVolumeClaim","name":"ten-khac"}}}' > "$EV/b2-bat-bien.txt" 2>&1 \
  && echo 'FAIL: sua duoc nguon du lieu sau khi tao' || echo 'PASS: nguon du lieu bat bien sau khi tao'

# 7. Kiểu tùy chỉnh
kubectl apply -f "$W/thu-dsr-cr.yaml"
[ "$(jp pvc/thu-dsr-cr '{.spec.dataSourceRef.apiGroup}/{.spec.dataSourceRef.kind}')" = 'example.storage.k8s.io/ExampleDataSource' ] \
  && echo 'PASS: dataSourceRef giu kieu ngoai PVC va VolumeSnapshot'
{ jp pvc/thu-dsr-cr 'dataSource={.spec.dataSource}{"\n"}phase={.status.phase}{"\n"}'
  kubectl -n "$NS" get events --field-selector involvedObject.name=thu-dsr-cr; } > "$EV/b2-dsr-cr.txt" 2>&1

# 8. Dọn
kubectl delete namespace "$NS" --wait=true --timeout=300s
for i in $(seq 1 60); do
  N="$(kubectl get pv -o jsonpath='{range .items[?(@.spec.claimRef.namespace=="bookstore-thu")]}{.metadata.name}{"\n"}{end}' | grep -c .)"
  [ "$N" -eq 0 ] && break; sleep 5
done
[ "$N" -eq 0 ] && echo 'PASS: namespace bookstore-thu da xoa, PV thu nghiem bien mat theo (Delete)'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/6b/thu-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bookstore-thu
  labels:
    app.kubernetes.io/part-of: bookstore
```

```yaml
# ~/lab-work/6b/thu-nguon.yaml — không khai storageClassName: lấy class mặc định local-path
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: thu-nguon
  namespace: bookstore-thu
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 100Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: thu-ghi
  namespace: bookstore-thu
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  restartPolicy: Never
  terminationGracePeriodSeconds: 2
  containers:
  - name: ghi
    image: busybox:1.37
    command: ["sh", "-c", "echo bookstore-6b > /data/marker.txt && sync"]
    resources:
      requests: {cpu: 10m, memory: 8Mi}
      limits: {cpu: 50m, memory: 16Mi}
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: thu-nguon
```

```yaml
# ~/lab-work/6b/thu-clone.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: thu-clone
  namespace: bookstore-thu
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 100Mi           # >= dung lượng nguồn (ghi chú của bài 101)
  dataSource:
    kind: PersistentVolumeClaim
    name: thu-nguon
---
apiVersion: v1
kind: Pod
metadata:
  name: thu-doc-clone
  namespace: bookstore-thu
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  terminationGracePeriodSeconds: 2
  containers:
  - name: doc
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 10m, memory: 8Mi}
      limits: {cpu: 50m, memory: 16Mi}
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: thu-clone
```

```yaml
# ~/lab-work/6b/thu-ds-sai.yaml — thụt lề hai dấu cách: script sinh thu-dsr-sai.yaml bằng sed
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: thu-ds-sai
  namespace: bookstore-thu
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 100Mi
  dataSource:
    kind: ConfigMap
    name: khong-ton-tai
```

```yaml
# ~/lab-work/6b/thu-dsr-cr.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: thu-dsr-cr
  namespace: bookstore-thu
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 100Mi
  dataSourceRef:
    apiGroup: example.storage.k8s.io
    kind: ExampleDataSource
    name: bookstore-thu
```

Chỉ ở nhánh A — `volumeSnapshotClassName` lấy giá trị `VSC` trong `b1-nang-luc.env`:

```yaml
# ~/lab-work/6b/thu-snap.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: thu-snap
  namespace: bookstore-thu
spec:
  volumeSnapshotClassName: <VSC>
  source:
    persistentVolumeClaimName: thu-nguon
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: thu-tu-snap
  namespace: bookstore-thu
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 100Mi
  dataSource:
    apiGroup: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: thu-snap
---
apiVersion: v1
kind: Pod
metadata:
  name: thu-doc-snap
  namespace: bookstore-thu
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  terminationGracePeriodSeconds: 2
  containers:
  - name: doc
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 10m, memory: 8Mi}
      limits: {cpu: 50m, memory: 16Mi}
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: thu-tu-snap
```

Ý chính cho phần Bối cảnh của ADR:

1. API nhận manifest, PVC `Bound`, Pod chạy, không một Event lỗi bắt buộc nào — chỉ có volume rỗng.
   Bài 101 đặt "chỉ khả dụng cho các CSI driver" ngay dòng đầu mục *Giới thiệu*; thiếu điều kiện đó
   hệ thống không báo lỗi. Một bản sao lưu "thành công" mỗi đêm mà rỗng chỉ bị phát hiện đúng ngày
   cần khôi phục.
2. Bài 101: nguồn và đích **cùng namespace**, và clone do chính backend lưu trữ tạo. Với local-path,
   backend là đĩa của `lab-k8s-worker2` — mất node là mất cả nguồn lẫn clone. Clone là bản sao để
   **dùng**, không phải bản sao lưu nằm ở miền lỗi khác.
3. `dataSourceRef`: cùng một giá trị sai, trường cũ im lặng bỏ đi còn trường mới báo lỗi ngay (bài
   102 khuyên ưu tiên `dataSourceRef`). Tính năng nguồn dữ liệu tùy ý đã bật mặc định ở baseline nên
   không có lý do quay về trường cũ.
4. Một custom resource mô tả nguồn (CRD — giai đoạn 14), một populator controller **bên ngoài** đọc
   nó và điền volume, cộng Event trên PVC để báo trạng thái. Không có populator thì PVC
   `thu-dsr-cr` không bao giờ được điền — B6 làm thủ công đúng vai trò của populator.

</details>

## B3. [BS-6B-03] Lưu trữ gặp lập lịch: dung lượng, giới hạn gắn, tình trạng volume

**Loại:** Điều tra.

**Bối cảnh:** `db-0` ghim `lab-k8s-worker2`, và `data-db-0` là một thư mục trên đĩa của chính node
đó. Ba câu hỏi vận hành chưa có lời đáp: đĩa worker2 không đủ chỗ thì scheduler có biết trước không;
worker2 gắn được tối đa bao nhiêu volume; và volume của DB hỏng thì Kubernetes báo ở đâu. Ba câu
trả lời đều là "tùy CSI driver" (bài 103, 104, 105) — đọc năng lực thật của cluster.

**Yêu cầu:**

1. API `CSIStorageCapacity` có trong cluster; đếm đối tượng hiện có và nói ai sẽ tạo chúng. Với
   StorageClass `bookstore-retain`, ba điều kiện của bài 103 đạt mấy.
2. `~/lab-work/6b/thu-500gi.yaml` trong namespace `bookstore-thu`: PVC `thu-500gi` xin `500Gi` từ
   class mặc định, Pod `thu-doc-500gi` trên `lab-k8s-worker2` mount nó ở `/data`. Chứng minh PVC được
   cấp, Pod chạy, PV khai đúng `500Gi` trong khi filesystem thật ở `/data` nhỏ hơn nhiều — khi cluster
   không theo dõi dung lượng.
3. Mỗi Node có đúng một `CSINode`; có driver nào công bố số volume tối đa không; Node có tài nguyên
   `attachable-volumes-*` nào không; bốn trường `attachRequired`, `storageCapacity`,
   `nodeAllocatableUpdatePeriodSeconds`, `preventPodSchedulingIfMissing` của `CSIDriver` trường nào có
   trong schema của cluster.
4. Đọc `/metrics` của kubelet trên node chạy `db-0` **qua API server**: có series tình trạng volume
   nào cho `data-db-0` không; Event trên PVC `data-db-0`.
5. Xóa namespace `bookstore-thu`.
6. Viết phần **Bối cảnh — lập lịch và giám sát lưu trữ** của ADR.

**Ràng buộc:** không bật feature gate, không SSH sửa gì. PVC 500Gi chỉ dùng class mặc định — với
local-path, con số đó không cấp phát đĩa thật nên không hại node.

**Đọc lại:**

- Bài [103](../103-storage-capacity-vi.md): [*API*](../103-storage-capacity-vi.md#api),
  [*Lập lịch*](../103-storage-capacity-vi.md#lập-lịch-scheduling),
  [*Hạn chế*](../103-storage-capacity-vi.md#hạn-chế-limitations).
- Bài [104](../104-storage-limits-vi.md):
  [*Giới hạn volume động*](../104-storage-limits-vi.md#giới-hạn-volume-động-dynamic-volume-limits),
  [*Cập nhật định kỳ*](../104-storage-limits-vi.md#cập-nhật-định-kỳ-periodic-updates),
  [*Ngăn việc đặt Pod khi không có CSI driver*](../104-storage-limits-vi.md#ngăn-việc-đặt-pod-khi-không-có-csi-driver-preventing-pod-placement-without-csi-driver).
- Bài [105](../105-volume-health-monitoring-vi.md):
  [*Giám sát tình trạng volume*](../105-volume-health-monitoring-vi.md#giám-sát-tình-trạng-volume-volume-health-monitoring).
- Bài [96](../96-storage-classes-vi.md#volume-binding-mode) — `Immediate` và `WaitForFirstConsumer`.

**Bàn giao:** `~/lab-work/6b/thu-500gi.yaml`; phần Bối cảnh thứ ba của ADR; `~/lab-evidence/6b/b3-*`.

**Nghiệm thu:**

- `PASS:` API `CSIStorageCapacity` có; khi cluster không có CSI driver thì không có đối tượng nào.
- `PASS:` ba điều kiện của bài 103 được đánh giá cho `bookstore-retain`, kết luận đúng một dòng.
- `PASS:` (khi không theo dõi dung lượng) PV `500Gi` được cấp cho `thu-500gi` trên filesystem nhỏ hơn
  hẳn.
- `PASS:` mỗi Node đúng một `CSINode`; khi không có CSI driver thì không bản ghi driver nào và không
  Node nào có `attachable-volumes-*`.
- `PASS:` `attachRequired` và `storageCapacity` có trong schema `CSIDriver`.
- `PASS:` đọc được `/metrics` của kubelet trên node của `db-0`; khi không có CSI driver thì không có
  series `kubelet_volume_stats_health_status_abnormal`.
- `PASS:` namespace `bookstore-thu` đã xóa.

**Câu hỏi thiết kế** — trả lời trong phần Bối cảnh của ADR:

1. PVC `500Gi` được cấp trên đĩa nhỏ hơn nhiều. Điều đó nói gì về `data-db-0` khi đĩa worker2 đầy?
   Kubernetes có bị lỗi không? Trên cluster này ai sẽ báo cho bạn?
2. Ba điều kiện của bài 103 — `bookstore-retain` đạt mấy? Nếu ai đó đổi class sang `Immediate`, PVC
   `data-db-0` mới và `db-0` (ghim worker2) sẽ ra sao? Bạn đã thấy hiện tượng nào ở Lab 6a?
3. Ai công bố số volume tối đa gắn được vào một node, qua API nào của CSI, và bạn đọc con số đó ở đâu?
   Bảng EBS/GCE/Azure đầu bài 104 có liên quan tới cluster này không?
4. Đĩa chứa `data-db-0` bắt đầu hỏng. Theo bài 105, Event được báo ở đâu và metric nào đổi giá trị —
   khi có đủ điều kiện? Trên cluster này, dấu hiệu đầu tiên bạn thấy sẽ là gì?

<details>
<summary>Gợi ý</summary>

1. `kubectl api-resources --api-group=storage.k8s.io` liệt kê mọi kiểu của nhóm lưu trữ.
2. `kubectl explain csidriver.spec.<trường>` thành công khi trường có trong schema.
3. `kubectl get --raw "/api/v1/nodes/<node>/proxy/metrics"` đi qua API server tới kubelet — đúng con
   đường tầng 6 của [gate A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready) đã kiểm.
4. `df -k /data` trong busybox: cột thứ hai của dòng thứ hai là tổng dung lượng, tính bằng KiB.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự viết thu-500gi.yaml</summary>

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
NS=bookstore-thu; EV=~/lab-evidence/6b; W=~/lab-work/6b
. "$EV/b1-nang-luc.env"

# 1. CSIStorageCapacity và ba điều kiện
kubectl api-resources --api-group=storage.k8s.io -o name | grep -q '^csistoragecapacities\.' \
  && echo 'PASS: API CSIStorageCapacity co trong cluster'
CAP_N="$(kubectl get csistoragecapacities -A --no-headers 2>/dev/null | wc -l)"
kubectl get csistoragecapacities -A > "$EV/b3-capacity.txt" 2>&1
[ "$CSIDRIVER_N" -eq 0 ] && [ "$CAP_N" -eq 0 ] \
  && echo 'PASS: khong CSI driver nao nen khong ai tao doi tuong CSIStorageCapacity'
VBM="$(kubectl get storageclass bookstore-retain -o jsonpath='{.volumeBindingMode}')"
if [ "$IS_CSI" -eq 1 ]; then
  SCAP="$(kubectl get csidriver "$PROVISIONER" -o jsonpath='{.spec.storageCapacity}')"; SCAP="${SCAP:-false}"
else
  SCAP='<khong co doi tuong CSIDriver>'
fi
{
  echo "1. Pod dung volume chua duoc tao       : DAT voi PVC moi cap phat dong"
  echo "2. SC tro CSI driver + WaitForFirstConsumer: provisioner=$PROVISIONER IS_CSI=$IS_CSI binding=$VBM"
  echo "3. CSIDriver.spec.storageCapacity=true  : $SCAP"
} | tee "$EV/b3-ba-dieu-kien.txt"
if [ "$IS_CSI" -eq 1 ] && [ "$VBM" = 'WaitForFirstConsumer' ] && [ "$SCAP" = 'true' ]; then
  CAP_TRACK=1; echo 'PASS: du ba dieu kien — scheduler CO xet dung luong voi bookstore-retain'
else
  CAP_TRACK=0; echo 'PASS: thieu it nhat mot dieu kien — scheduler KHONG xet dung luong voi bookstore-retain'
fi

# 2. 500Gi trên đĩa nhỏ
kubectl apply -f "$W/thu-ns.yaml"
if [ "$CAP_TRACK" -eq 0 ]; then
  kubectl apply -f "$W/thu-500gi.yaml"
  kubectl -n "$NS" wait --for=condition=Ready pod/thu-doc-500gi --timeout=300s
  PV_BIG="$(kubectl -n "$NS" get pvc thu-500gi -o jsonpath='{.spec.volumeName}')"
  CAP_BIG="$(kubectl get pv "$PV_BIG" -o jsonpath='{.spec.capacity.storage}')"
  DF_KB="$(kubectl -n "$NS" exec thu-doc-500gi -- df -k /data | awk 'NR==2{print $2}')"
  echo "PV=$PV_BIG capacity=$CAP_BIG filesystem-that=${DF_KB} KiB (500Gi = 524288000 KiB)" | tee "$EV/b3-500gi.txt"
  [ "$CAP_BIG" = '500Gi' ] && [ "${DF_KB:-0}" -gt 0 ] && [ "$DF_KB" -lt 524288000 ] \
    && echo 'PASS: cluster cap PV 500Gi tren filesystem nho hon han — khong ai kiem dung luong'
else
  echo 'BO QUA: cluster co theo doi dung luong, yeu cau 500Gi se bi tu choi dung thiet ke'
fi

# 3. Giới hạn volume theo node
NODE_N="$(kubectl get nodes --no-headers | wc -l)"
[ "$(sl kubectl get csinodes --no-headers)" -eq "$NODE_N" ] && echo 'PASS: moi Node co dung mot CSINode'
kubectl get csinodes -o custom-columns='NODE:.metadata.name,DRIVER:.spec.drivers[*].name,COUNT:.spec.drivers[*].allocatable.count' \
  | tee "$EV/b3-csinode.txt"
ENTRIES="$(kubectl get csinodes -o jsonpath='{range .items[*]}{range .spec.drivers[*]}{.name}{"\n"}{end}{end}' | grep -c .)"
ATT_N="$(kubectl get nodes -o json | grep -c 'attachable-volumes')"
[ "$CSIDRIVER_N" -eq 0 ] && [ "$ENTRIES" -eq 0 ] && [ "$ATT_N" -eq 0 ] \
  && echo 'PASS: khong driver nao cong bo allocatable.count, khong Node nao co attachable-volumes-*'
for f in attachRequired storageCapacity nodeAllocatableUpdatePeriodSeconds preventPodSchedulingIfMissing; do
  kubectl explain "csidriver.spec.$f" >/dev/null 2>&1 && echo "co: $f" || echo "khong co: $f"
done | tee "$EV/b3-truong-csidriver.txt"
grep -qx 'co: attachRequired' "$EV/b3-truong-csidriver.txt" && grep -qx 'co: storageCapacity' "$EV/b3-truong-csidriver.txt" \
  && echo 'PASS: schema CSIDriver co attachRequired va storageCapacity'

# 4. Tình trạng volume của data-db-0
NODE_DB="$(kubectl -n bookstore get pod db-0 -o jsonpath='{.spec.nodeName}')"
kubectl get --raw "/api/v1/nodes/$NODE_DB/proxy/metrics" > "$EV/b3-kubelet-metrics.txt"
[ -s "$EV/b3-kubelet-metrics.txt" ] && echo "PASS: doc duoc /metrics cua kubelet tren $NODE_DB qua API server"
HEALTH_N="$(grep -c '^kubelet_volume_stats_health_status_abnormal' "$EV/b3-kubelet-metrics.txt")"
DB_SERIES="$(grep '^kubelet_volume_stats_' "$EV/b3-kubelet-metrics.txt" | grep -c 'persistentvolumeclaim="data-db-0"')"
echo "health_status_abnormal=$HEALTH_N | series kubelet_volume_stats_* cua data-db-0=$DB_SERIES" | tee "$EV/b3-health.txt"
[ "$CSIDRIVER_N" -eq 0 ] && [ "$HEALTH_N" -eq 0 ] \
  && echo 'PASS: khong CSI driver nao — khong co series tinh trang volume, ke ca cho data-db-0'
kubectl -n bookstore get events --field-selector involvedObject.kind=PersistentVolumeClaim,involvedObject.name=data-db-0 \
  > "$EV/b3-data-db-0-events.txt" 2>&1

# 5. Dọn
kubectl delete namespace "$NS" --wait=true --timeout=300s
for i in $(seq 1 60); do
  N="$(kubectl get pv -o jsonpath='{range .items[?(@.spec.claimRef.namespace=="bookstore-thu")]}{.metadata.name}{"\n"}{end}' | grep -c .)"
  [ "$N" -eq 0 ] && break; sleep 5
done
[ "$N" -eq 0 ] && ! kubectl get namespace "$NS" >/dev/null 2>&1 && echo 'PASS: namespace bookstore-thu da xoa'
```

Bài 103 và 104 được dịch từ bản tài liệu **mới hơn baseline** khóa ở
[A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); đừng suy trạng thái tính năng từ bài.
Hai trường cuối của `CSIDriver` in `co:` hay `khong co:` đều hợp lệ — ghi đúng giá trị đọc được.
Một trường có trong schema **không** có nghĩa là tính năng đang bật.

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/6b/thu-500gi.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: thu-500gi
  namespace: bookstore-thu
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 500Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: thu-doc-500gi
  namespace: bookstore-thu
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  terminationGracePeriodSeconds: 2
  containers:
  - name: doc
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 10m, memory: 8Mi}
      limits: {cpu: 50m, memory: 16Mi}
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: thu-500gi
```

Ý chính cho phần Bối cảnh của ADR:

1. Không ai được giao việc so dung lượng: không có `CSIStorageCapacity`, phép so của bài 103 không có
   dữ liệu để chạy. Kubernetes không lỗi — nó làm đúng thiết kế khi thiếu điều kiện. Khi đĩa worker2
   đầy, PostgreSQL ghi thất bại và `db-0` lỗi; trên cluster này chưa có gì cảnh báo trước. Quan sát
   đĩa node thuộc phần quan sát hệ thống ở Lab 11a — ghi vào phần Hệ quả của ADR như một rủi ro.
2. `bookstore-retain` đạt điều kiện về `WaitForFirstConsumer` nhưng không trỏ CSI driver, nên chỉ đạt
   một nửa điều kiện 2 và trượt điều kiện 3. Đổi sang `Immediate`: provisioner được gọi trước khi có
   Pod và không biết cấp thư mục trên node nào — PVC nằm `Pending` kèm Event lỗi như Lab 6a đã thấy,
   `db-0` không bao giờ lên.
3. CSI driver tự công bố qua `NodeGetInfo`; con số hiện ra ở
   `CSINode.spec.drivers[].allocatable.count`. Cluster không chạy trên đám mây nào nên bảng mặc định
   của bài 104 không liên quan; không có driver nào nên scheduler không có giới hạn nào để tôn trọng.
4. Phía controller báo Event trên PVC, phía node báo Event trên mọi Pod dùng PVC và đặt
   `kubelet_volume_stats_health_status_abnormal` bằng `1` với label `namespace`,
   `persistentvolumeclaim` — chỉ khi driver hiện thực tính năng và có External Health Monitor, gate
   `CSIVolumeHealth` cho phía node. Ở đây dấu hiệu đầu tiên là lỗi I/O trong log của `postgres`,
   `db-0` mất Ready, và bản dump ngừng có `db-backup: OK`.

</details>

## B4. [BS-6B-04] ADR: sao lưu logic bằng `pg_dump` thay cho VolumeSnapshot

**Loại:** Điều tra.

**Bối cảnh:** B1–B3 cho bức tranh đầy đủ về tầng lưu trữ. Bây giờ đội phải **quyết định** và ghi
lại vì sao Bookstore sao lưu bằng bản dump logic của CronJob `db-backup` chứ không bằng
VolumeSnapshot, cái giá của quyết định đó, và khi nào phải xem lại. Quyết định cần số đo thật, không
cần cảm nhận.

**Yêu cầu:**

1. Đo một lần sao lưu **ngoài lịch**: thời gian Job chạy và kích thước bản dump.
2. Xác định node đang giữ PV của `data-db-0` và node đang giữ PV của `backup-store`.
3. Hoàn thiện `docs/adr/NNNN-sao-luu-logic.md`, trạng thái *chấp nhận*, đủ bốn mục của mẫu ADR. ADR
   phải nêu: kết luận nhánh ở B1 kèm đường dẫn bằng chứng; lý do vẫn chọn `pg_dump` **ngay cả khi**
   cluster có snapshot; RPO theo lịch hiện tại và RPO khi mất cả cluster; số đo của yêu cầu 1 và 2;
   nợ lab #5 còn mở và điều kiện trả; `TD-10` và việc B5 sẽ làm; điều kiện để xem lại ADR.

**Ràng buộc:** không đổi lịch hay manifest của `db-backup` trong sprint này — ADR ghi nhận hiện trạng
và cái giá của nó. Job chạy ngoài lịch được xóa khi đo xong.

**Đọc lại:**

- Bài [99](../99-volume-snapshots-vi.md#giới-thiệu-introduction), mục *Giới thiệu* — snapshot cho phép
  "sao lưu cơ sở dữ liệu trước khi thực hiện các thao tác chỉnh sửa hoặc xóa".
- Bài [101](../101-volume-pvc-datasource-vi.md#giới-thiệu-introduction) — nguồn và đích cùng
  namespace; bài [92](../92-persistent-volumes-vi.md#node-affinity), mục *Node Affinity* — PV cục bộ
  gắn với node.
- PostgreSQL 17, [*SQL Dump*](https://www.postgresql.org/docs/17/backup-dump.html) — đoạn mở đầu về
  tính nhất quán, mục *Using pg_dumpall* về role.
- [Mục 3.4](DU-AN-BOOKSTORE.md#34-luồng-request-và-luồng-dữ-liệu) và
  [mục 8](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án) của file dự án — luồng dữ liệu bền và `TD-10`.

**Bàn giao:** `docs/adr/NNNN-sao-luu-logic.md` hoàn chỉnh; `~/lab-evidence/6b/b4-backup.txt`.

**Nghiệm thu:**

- `PASS:` một lần sao lưu ngoài lịch thành công; thời gian và kích thước được ghi lại.
- `PASS:` biết node giữ PV `data-db-0` và node giữ PV `backup-store`.
- `PASS:` ADR ở trạng thái *chấp nhận*, đủ bốn mục.
- `PASS:` ADR dẫn bằng chứng của B1 và nhắc nợ #5, `TD-10`, RPO.

**Câu hỏi thiết kế** — trả lời trong các mục Quyết định, Lý do, Hệ quả của ADR:

1. Vì sao `pg_dump` trên cluster này? Nếu ngày mai cluster có CSI driver hỗ trợ snapshot, bạn có bỏ
   `pg_dump` không — vì sao?
2. `pg_dump` cho bản sao nhất quán thế nào trong khi DB vẫn nhận ghi? So với chép thẳng thư mục dữ
   liệu của `data-db-0` trên worker2 trong lúc PostgreSQL đang chạy — nhớ lại vì sao Lab 3a cần
   *fast shutdown* và checkpoint.
3. Cái giá phải trả: RPO bao nhiêu với lịch hiện tại, và khi mất cả cluster thì RPO tính từ mốc nào?
   Bản dump thiếu thứ gì mà một bản sao thư mục dữ liệu có?
4. Mất `lab-k8s-worker2` thì mất những gì, dựa trên số đo của yêu cầu 2?
5. Điều gì phải xảy ra thì ADR này cần xem lại? Làm sao để hệ thống tự nhắc bạn?

<details>
<summary>Gợi ý</summary>

- Chạy ngay một lần từ CronJob: `kubectl -n bookstore create job <tên> --from=cronjob/db-backup`.
- Thời gian chạy: `.status.startTime` và `.status.completionTime` của Job. Kích thước: dòng
  `db-backup: OK <file> <byte> byte` trong log.
- Node của một PV cục bộ nằm ở `spec.nodeAffinity` của PV đó.
- Khi đọc PostgreSQL docs: `pg_dump` chỉ dump **một** database; role là đối tượng toàn cluster
  PostgreSQL.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã viết xong ADR</summary>

```bash
NS=bookstore; EV=~/lab-evidence/6b
cd ~/bookstore

J="thu-adr-$(date +%s)"
kubectl -n "$NS" create job "$J" --from=cronjob/db-backup
kubectl -n "$NS" wait --for=condition=Complete "job/$J" --timeout=300s && echo 'PASS: mot lan sao luu ngoai lich thanh cong'
ST="$(kubectl -n "$NS" get job "$J" -o jsonpath='{.status.startTime}')"
CT="$(kubectl -n "$NS" get job "$J" -o jsonpath='{.status.completionTime}')"
BYTES="$(kubectl -n "$NS" logs "job/$J" | awk '/^db-backup: OK/{print $4}')"
DUR=$(( $(date -d "$CT" +%s) - $(date -d "$ST" +%s) ))
echo "job=$J thoi-gian=${DUR}s kich-thuoc=${BYTES} byte" | tee "$EV/b4-backup.txt"
[ "${BYTES:-0}" -gt 0 ] && echo "PASS: ban dump ${BYTES} byte, Job chay ${DUR} giay"
kubectl -n "$NS" delete job "$J" --wait=true

pvnode() {
  kubectl get pv "$(kubectl -n "$NS" get pvc "$1" -o jsonpath='{.spec.volumeName}')" \
    -o jsonpath='{.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}'
}
ND="$(pvnode data-db-0)"; NB="$(pvnode backup-store)"
echo "data-db-0 tren $ND ; backup-store tren $NB" | tee -a "$EV/b4-backup.txt"
[ -n "$ND" ] && [ -n "$NB" ] && echo 'PASS: biet node giu du lieu va node giu ban dump'
[ "$ND" = "$NB" ] && echo 'GHI CHU: du lieu va ban dump cung mot node — mat node la mat ca hai' \
                  || echo 'GHI CHU: khac node nhung van cung mot cluster'

ADR="$(ls docs/adr/*-sao-luu-logic.md 2>/dev/null | head -n1)"
grep -q '^- Trạng thái: chấp nhận' "$ADR" && echo "PASS: $ADR o trang thai chap nhan"
BAD=0
for H in '## Bối cảnh' '## Quyết định' '## Lý do' '## Hệ quả'; do
  grep -q "^$H" "$ADR" || { echo "FAIL: ADR thieu muc $H"; BAD=1; }
done
for K in 'b1-nang-luc' '#5' 'TD-10' 'RPO'; do
  grep -q -- "$K" "$ADR" || { echo "FAIL: ADR khong nhac $K"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: ADR du bon muc, dan bang chung B1, nhac no #5, TD-10, RPO'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của ADR</summary>

**Quyết định:** Bookstore sao lưu bằng bản dump logic `pg_dump` do CronJob `db-backup` tạo, lưu
trên PVC `backup-store` và chép ra ngoài cluster theo `docs/runbook-backup.md`. Không dùng
VolumeSnapshot hay nhân bản PVC làm cơ chế sao lưu.

**Lý do:**

1. Trên cluster này không có lựa chọn khác: provisioner không phải CSI, không CRD, không controller
   (`~/lab-evidence/6b/b1-nang-luc.txt`); nhân bản trả về volume rỗng mà không báo lỗi
   (`b2-clone.txt`). Nợ lab #5 vẫn mở.
2. Ngay cả khi có snapshot, vẫn giữ `pg_dump`: snapshot và clone do **cùng backend** tạo, cùng
   namespace với nguồn — với lưu trữ cục bộ là cùng một đĩa của `lab-k8s-worker2`. Bản dump là một
   file SQL độc lập với backend, với node và với cluster: chép ra ngoài được (`TD-10`), nạp vào DB
   trên cluster dựng lại ở Lab 8a được. Snapshot hợp với việc "chụp trước khi thao tác" — bản tương
   đương ở đây là chạy `db-backup` ngoài lịch trước mọi thao tác rủi ro.
3. PostgreSQL docs: bản dump của `pg_dump` nhất quán nội bộ — là ảnh của DB tại lúc `pg_dump` bắt
   đầu — và không chặn thao tác ghi thông thường. Chép thư mục dữ liệu trong lúc PostgreSQL đang
   chạy cho một tập file ở các thời điểm khác nhau, tương đương trạng thái "mất điện" mà Lab 3a dùng
   *fast shutdown* để tránh.

**Hệ quả:**

- RPO theo lịch là tối đa một chu kỳ của `db-backup` (30 phút với lịch của Lab 4b). Khi mất cả
  cluster, RPO tính từ **lần chép ra ngoài gần nhất**, không phải lần dump gần nhất — đó là điều kiện
  của `TD-10`.
- Thời gian khôi phục tăng theo kích thước dữ liệu (đo ở B6); `pg_dump` thêm tải đọc lên DB mỗi lần
  chạy (đo ở yêu cầu 1).
- Bản dump **không chứa role** (`pg_dump` chỉ dump một database): khôi phục phải tạo `web_anon`,
  `authenticator` trước khi nạp — B6 diễn tập đúng điểm này.
- Đĩa worker2 đầy hay hỏng không được Kubernetes báo trước trên cluster này (B3) — rủi ro mở cho
  phần quan sát ở Lab 11a.
- Xem lại ADR khi cluster có API `VolumeSnapshot` hoặc CSI driver mới. `tests/smoke.sh` của sprint này
  kiểm điều kiện đó ở mỗi lần chạy và báo `FAIL` khi nó thay đổi.

Trả lời câu 4 theo số đo của bạn: nếu hai PV cùng ở worker2, mất node là mất cả DB lẫn mọi bản dump
trong cluster; nếu khác node, mất worker2 vẫn còn bản dump nhưng cả hai vẫn chung số phận khi mất
cluster.

</details>

## B5. [BS-6B-05] Bản dump ra ngoài cluster — `TD-10`

**Loại:** Tính năng — quy trình vận hành ghi trong repo.

**Bối cảnh:** `TD-10` ghi rằng bản backup vẫn nằm trong cùng cluster với DB. PVC `backup-store` là PV
cục bộ trên **một** node, bị `teardown.sh` xóa cuối mỗi sprint và biến mất khi restore snapshot VM.
[File dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án) chấp nhận `TD-10` **có điều kiện**: bản dump
được chép ra ngoài bằng tay theo một runbook. Ticket này viết và chạy runbook đó.

**Yêu cầu:**

1. Tạo một bản dump mới ngay, không chờ lịch.
2. Một Pod đọc tạm tên `backup-reader` trong namespace `bookstore` (`busybox`, manifest
   `~/lab-work/6b/backup-reader.yaml`) mount `backup-store` ở `/backup` **chỉ đọc**. Pod phải chạy
   trên đúng node giữ PV `backup-store` — tự xác định node đó từ PV, không đoán.
3. Chép bản dump vừa tạo về `~/bookstore-backups/` trên master; checksum trên master khớp checksum
   tính trong Pod; thư mục quyền `700`, file quyền `600`; kèm một file checksum để kiểm ở máy host.
4. Trên master, kiểm nội dung bản sao: đúng là dump SQL thuần của PostgreSQL, có dữ liệu `api.books`,
   có cấp quyền cho `web_anon` nhưng **không** tạo role nào.
5. Chép bản dump và file checksum ra máy host; kiểm checksum ở máy host.
6. Xóa `backup-reader` và Job tạo ở yêu cầu 1.
7. `docs/runbook-backup.md`, mục **1. Chép bản dump ra ngoài cluster**: từng bước có lệnh, ai làm,
   khi nào làm (ít nhất: cuối mỗi sprint cùng lúc với bundle; trước mọi thao tác phá dữ liệu hay phá
   cluster), node và đường dẫn trên node của PV `backup-store` làm **đường thoát khi API server không
   trả lời**, và giữ bao nhiêu bản.
8. `docs/tech-debt.md`: dòng `TD-10` chuyển trạng thái thành *chấp nhận có điều kiện* và nêu điều kiện
   (thêm dòng nếu Lab 6a chưa ghi).

**Ràng buộc:** không ghim `backup-reader` bằng tay vào một node đoán trước. Không ghi vào
`backup-store`. Bản dump không vào repo, không vào `~/lab-evidence/`. Không dùng `hostPath` trong Pod
để đọc thư mục của PV.

**Đọc lại:**

- Bài [92](../92-persistent-volumes-vi.md): [*Node Affinity*](../92-persistent-volumes-vi.md#node-affinity),
  [*Dùng claim làm volume*](../92-persistent-volumes-vi.md#claims-as-volumes) — `readOnly` khi mount
  claim, [*Các chế độ truy cập*](../92-persistent-volumes-vi.md#access-modes) — `ReadWriteOnce` là
  ràng buộc theo node.
- Bài [96](../96-storage-classes-vi.md#volume-binding-mode) — vì sao PV của `backup-store` nằm ở node
  của Pod tiêu thụ đầu tiên.
- [Mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm) — quy trình bundle mà bản
  dump đi kèm.

**Bàn giao:** `~/lab-work/6b/backup-reader.yaml`; `docs/runbook-backup.md` mục 1; `docs/tech-debt.md`;
bản dump trong `~/bookstore-backups/` và trên máy host; `~/lab-evidence/6b/b5-*`.

**Nghiệm thu:**

- `PASS:` đọc được node và đường dẫn trên node của PV `backup-store`.
- `PASS:` `backup-reader` chạy đúng node đó; ghi vào `/backup` thất bại.
- `PASS:` bản dump mới nhất ở `~/bookstore-backups/` có trên PVC và checksum khớp.
- `PASS:` thư mục `700`, file `600`, có file checksum đi kèm và tự kiểm được.
- `PASS:` bản sao là dump SQL thuần, có dữ liệu `api.books`, cấp quyền cho `web_anon`, không tạo role.
- `PASS:` file của bản dump có mặt trên node tại đường dẫn của PV — đường thoát khi API không trả lời.
- `PASS:` repo không chứa bản dump nào; runbook mục 1 đủ các bước chính; `TD-10` *chấp nhận có điều
  kiện*.
- `PASS:` `backup-reader` đã xóa.
- Ở máy host: dòng `PASS: ban dump tren may host khop checksum` của khối PowerShell.

**Câu hỏi thiết kế** — trả lời trong `docs/runbook-backup.md`, mục *Vì sao quy trình như vậy*:

1. Vì sao bản dump nằm trên `backup-store` chưa phải bản sao lưu đủ? Kể ít nhất ba tình huống làm mất
   nó cùng lúc với DB.
2. Vì sao `backup-reader` không cần — và không nên — tự ghim node? Nếu ghim sai thì Pod ra sao?
3. Vì sao không commit bản dump vào repo, cũng không để nó đi theo bundle?
4. API server không trả lời (ví dụ khi Lab 8a phá control plane). Lấy bản dump bằng đường nào, và
   runbook phải ghi sẵn thông tin gì **lúc cluster còn sống**?
5. Điều kiện nào thì `TD-10` được chấp nhận? RPO thật khi mất cả cluster bằng bao nhiêu?

<details>
<summary>Gợi ý</summary>

1. Tên file của bản dump vừa tạo nằm trong dòng `db-backup: OK` ở log Job.
2. Node của PV đọc ở `spec.nodeAffinity`, đường dẫn trên node ở `spec.hostPath.path` của PV.
3. Không khai node nào cho `backup-reader`: PV đã mang `nodeAffinity`, scheduler tự đặt Pod lên đúng
   node. Volume `persistentVolumeClaim` nhận `readOnly: true`; container mount cũng nhận `readOnly`.
4. `kubectl cp <pod>:<đường dẫn> <file cục bộ> -c <container>` cần `tar` trong container —
   `busybox` có.
5. `sha256sum` có ở cả master lẫn `busybox`. Ở máy host dùng `Get-FileHash -Algorithm SHA256` như
   [Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) kiểm ISO.
6. Dump SQL thuần bắt đầu bằng chú thích `PostgreSQL database dump`; dump định dạng custom bắt đầu
   bằng `PGDMP`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script tạo lại backup-reader từ manifest của bạn</summary>

```bash
NS=bookstore; EV=~/lab-evidence/6b; W=~/lab-work/6b; BK=~/bookstore-backups
cd ~/bookstore

PV="$(kubectl -n "$NS" get pvc backup-store -o jsonpath='{.spec.volumeName}')"
NODE="$(kubectl get pv "$PV" -o jsonpath='{.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}')"
HP="$(kubectl get pv "$PV" -o jsonpath='{.spec.hostPath.path}{.spec.local.path}')"
echo "backup-store: PV=$PV node=$NODE duong-dan=$HP" | tee "$EV/b5-backup-store.txt"
[ -n "$NODE" ] && [ -n "$HP" ] && echo 'PASS: doc duoc node va duong dan tren node cua PV backup-store'

kubectl apply -f "$W/backup-reader.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/backup-reader --timeout=180s
[ "$(kubectl -n "$NS" get pod backup-reader -o jsonpath='{.spec.nodeName}')" = "$NODE" ] \
  && echo "PASS: backup-reader chay dung node giu PV ($NODE)"
kubectl -n "$NS" exec backup-reader -- sh -c 'touch /backup/.thu-ghi' >/dev/null 2>&1 \
  && echo 'FAIL: ghi duoc vao backup-store' || echo 'PASS: backup-store mount chi doc'

LOCAL="$(ls -1t "$BK"/*.sql 2>/dev/null | head -n1)"; F="$(basename "${LOCAL:-khong-co}")"
S1="$(kubectl -n "$NS" exec backup-reader -- sha256sum "/backup/$F" 2>/dev/null | cut -d' ' -f1)"
S2="$(sha256sum "$LOCAL" 2>/dev/null | cut -d' ' -f1)"
[ -n "$S1" ] && [ "$S1" = "$S2" ] && echo "PASS: $F co tren PVC va checksum khop ban ngoai cluster"
{ [ "$(stat -c %a "$BK")" = '700' ] && [ "$(stat -c %a "$LOCAL")" = '600' ]; } && echo 'PASS: thu muc 700, file 600'
( cd "$BK" && sha256sum -c "$F.sha256" >/dev/null 2>&1 ) && echo 'PASS: co file checksum di kem va tu kiem duoc'

head -c 5 "$LOCAL" | grep -q 'PGDMP' && echo 'FAIL: ban dump dang custom — runbook phai dung pg_restore (muc 4)'
{ grep -q 'PostgreSQL database dump' "$LOCAL" && grep -q '^COPY api\.books ' "$LOCAL"; } \
  && echo 'PASS: dump SQL thuan, co du lieu api.books'
{ grep -q 'TO web_anon' "$LOCAL" && ! grep -q 'CREATE ROLE' "$LOCAL"; } \
  && echo 'PASS: ban dump cap quyen cho web_anon nhung khong tao role nao — role khong nam trong ban dump'
{ echo "file=$F sha256=$S2 byte=$(stat -c %s "$LOCAL")"
  echo "so dong COPY api.books=$(awk '/^COPY api\.books /{f=1;next} /^\\\.$/{f=0} f' "$LOCAL" | wc -l)"; } \
  > "$EV/b5-ban-dump.txt"

ssh "$NODE" "sudo test -f '$HP/$F'" \
  && echo 'PASS: file co tren node tai duong dan cua PV — duong thoat khi API khong tra loi'

git grep -l 'PostgreSQL database dump' >/dev/null 2>&1 \
  && echo 'FAIL: repo chua ban dump' || echo 'PASS: repo khong chua ban dump'
BAD=0
for K in 'backup-reader' 'kubectl.* cp ' 'sha256' 'scp' "$NODE"; do
  grep -Eq -- "$K" docs/runbook-backup.md || { echo "FAIL: runbook muc 1 thieu: $K"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: runbook muc 1 du cac buoc chinh va ghi node cua backup-store'
grep 'TD-10' docs/tech-debt.md | grep -q 'chấp nhận có điều kiện' && echo 'PASS: TD-10 chap nhan co dieu kien'

kubectl -n "$NS" delete pod backup-reader --wait=true
kubectl -n "$NS" get pod backup-reader >/dev/null 2>&1 || echo 'PASS: da xoa backup-reader'
```

Ở **máy host Windows**, PowerShell — sửa user, tên file và thư mục đích cho đúng máy của bạn:

```powershell
$f = '<ten-file>.sql'
New-Item -ItemType Directory -Force "$HOME\lab-backup" | Out-Null
scp "<user>@192.168.100.221:~/bookstore-backups/$f" "$HOME\lab-backup\"
scp "<user>@192.168.100.221:~/bookstore-backups/$f.sha256" "$HOME\lab-backup\"
$h = (Get-FileHash -Algorithm SHA256 -LiteralPath "$HOME\lab-backup\$f").Hash.ToLowerInvariant()
$e = (Get-Content -LiteralPath "$HOME\lab-backup\$f.sha256").Split(' ')[0]
if ($h -eq $e) { 'PASS: ban dump tren may host khop checksum' } else { "FAIL: checksum lech: $h" }
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/6b/backup-reader.yaml — runbook mục 1 chép nguyên văn manifest này
apiVersion: v1
kind: Pod
metadata:
  name: backup-reader
  namespace: bookstore
spec:
  # Không khai node: PV của backup-store mang nodeAffinity, scheduler đặt Pod đúng chỗ.
  restartPolicy: Never
  terminationGracePeriodSeconds: 2
  containers:
  - name: reader
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 10m, memory: 8Mi}
      limits: {cpu: 50m, memory: 16Mi}
    volumeMounts:
    - name: backup
      mountPath: /backup
      readOnly: true
  volumes:
  - name: backup
    persistentVolumeClaim:
      claimName: backup-store
      readOnly: true
```

Các lệnh của runbook mục 1:

```bash
NS=bookstore; BK=~/bookstore-backups

# 1. Bản dump mới, không chờ lịch — cũng là bước bắt buộc trước mọi thao tác rủi ro
J="backup-tay-$(date +%Y%m%d%H%M%S)"
kubectl -n "$NS" create job "$J" --from=cronjob/db-backup
kubectl -n "$NS" wait --for=condition=Complete "job/$J" --timeout=300s
F="$(basename "$(kubectl -n "$NS" logs "job/$J" | awk '/^db-backup: OK/{print $3}')")"; echo "$F"

# 2. Node và đường dẫn của PV backup-store — chép hai giá trị này vào runbook
PV="$(kubectl -n "$NS" get pvc backup-store -o jsonpath='{.spec.volumeName}')"
kubectl get pv "$PV" -o jsonpath='{.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}{" "}{.spec.hostPath.path}{"\n"}'

# 3. Pod đọc, chép ra, checksum
kubectl apply -f ~/lab-work/6b/backup-reader.yaml
kubectl -n "$NS" wait --for=condition=Ready pod/backup-reader --timeout=180s
mkdir -p "$BK" && chmod 700 "$BK"
kubectl -n "$NS" cp "backup-reader:/backup/$F" "$BK/$F" -c reader
chmod 600 "$BK/$F"
kubectl -n "$NS" exec backup-reader -- sha256sum "/backup/$F"
( cd "$BK" && sha256sum "$F" > "$F.sha256" && chmod 600 "$F.sha256" && cat "$F.sha256" )

# 4. Kiểm nội dung trên master
grep -m1 'PostgreSQL database dump' "$BK/$F"
grep -c '^COPY api\.' "$BK/$F"
grep -c 'CREATE ROLE' "$BK/$F"          # phải là 0

# 5. Dọn
kubectl -n "$NS" delete pod backup-reader --wait=true
kubectl -n "$NS" delete job "$J" --wait=true
# 6. Máy host: scp + Get-FileHash (khối PowerShell của script nghiệm thu)
```

Dòng `TD-10` trong `docs/tech-debt.md`:

```markdown
| TD-10 | Bản backup nằm cùng cluster với DB | 6a | 6b | chấp nhận có điều kiện: chép ra ngoài theo docs/runbook-backup.md mục 1 cuối mỗi sprint và trước mọi thao tác phá; checksum khớp ở máy host; diễn tập khôi phục ở 6b | ~/lab-evidence/6b/b5-ban-dump.txt |
```

Ý chính cho mục *Vì sao quy trình như vậy*:

1. Cùng cluster, cùng node cục bộ: mất node, `teardown.sh` cuối sprint (xóa cả PV `Retain` và thư mục
   trên node), restore snapshot VM — mỗi tình huống mất bản dump cùng lúc với DB.
2. PV cục bộ mang `nodeAffinity`; scheduler chỉ đặt Pod tiêu thụ lên node đó. Ghim sai node thì Pod
   nằm `Pending` với sự kiện xung đột node affinity của volume.
3. Bản dump chứa dữ liệu khách hàng; git giữ mọi phiên bản vĩnh viễn, còn repo và bundle được chép
   tới nhiều nơi hơn mức dữ liệu được phép đi.
4. SSH vào node giữ PV và đọc thẳng thư mục ở đường dẫn của PV. Node và đường dẫn phải ghi vào runbook
   khi API còn trả lời — lúc API chết thì không hỏi được nữa.
5. Điều kiện: runbook có, được chạy cuối mỗi sprint và trước thao tác phá, checksum khớp ở máy host,
   khôi phục đã diễn tập (B6). RPO khi mất cluster bằng khoảng thời gian từ lần chép ra ngoài gần
   nhất — có thể dài hơn rất nhiều so với chu kỳ 30 phút của `db-backup`.

</details>

## B6. [BS-6B-06] Diễn tập khôi phục vào DB mới trong namespace tạm

**Loại:** Diễn tập sự cố — DB diễn tập ghim `lab-k8s-worker2`.

**Bối cảnh:** một bản sao lưu chưa từng được khôi phục thì chưa phải bản sao lưu. Kịch bản: DB
production coi như đã mất, thứ duy nhất còn lại là bản dump đã chép ra ngoài ở B5. Đội phải chứng
minh bản dump đó dựng lại được một DB **đúng dữ liệu**, và đo mất bao lâu. Diễn tập không được chạm
vào production: DB mới nằm trong một namespace tạm, cách ly mạng, rồi bị xóa.

**Yêu cầu:**

1. Trước diễn tập: đặt một đơn hàng đánh dấu (số lượng `6`) vào DB thật, ghi id của nó vào
   `~/lab-evidence/6b/b6-dau-don-hang.txt`; rồi chạy runbook mục 1 để có một bản dump mới **ngoài
   cluster** chứa đơn hàng đó. Bản sao trên master là **đầu vào duy nhất** của diễn tập.
2. Ghi mốc thời gian vào `~/lab-evidence/6b/b6-thoi-gian.txt`, mỗi dòng `ten=giay-epoch`, theo thứ
   tự: `bat_dau`, `db_ready`, `nap_bat_dau`, `nap_xong`, `doi_chieu_xong`.
3. `~/lab-work/6b/restore-ns.yaml`: namespace `bookstore-restore-test` mang label dự án; mọi lưu lượng
   vào và ra của mọi Pod trong namespace đều bị chặn.
4. Bí mật **mới** sinh tại chỗ vào `~/lab-work/6b/restore.env` (quyền `600`, đúng hai key như
   `secrets/db.env`), nạp thành Secret `restore-credentials` của namespace tạm. Không dùng
   `secrets/db.env` của production.
5. `~/lab-work/6b/restore-db.yaml`: DB mới gồm script khởi tạo trong ConfigMap, PVC `restore-data` và
   Pod `db-restore`:
   - image `postgres` đúng tag A1.4, container `postgres`, database `bookstore`, mật khẩu superuser
     đọc từ file của Secret, thư mục dữ liệu là **thư mục con** của volume — như `db-0` sau Lab 6a;
   - volume bền mới từ StorageClass **mặc định**; Pod chạy trên `lab-k8s-worker2`;
   - script khởi tạo **chỉ** tạo hai role: `web_anon` không đăng nhập, và `authenticator` đăng nhập
     bằng mật khẩu trong Secret, được cấp `web_anon`. Không schema, không dữ liệu mẫu;
   - readiness chỉ đạt khi PostgreSQL nhận kết nối TCP, như `db-0`. Không có Service nào.
6. Ngay khi `db-restore` Ready: dừng lại, chạy **Khối A** của script nghiệm thu.
7. Nạp bản dump từ bản sao trên master vào `db-restore` trong **một giao dịch**, dừng ở lỗi đầu tiên.
8. Tự đối chiếu với `db-0`: số dòng và checksum nội dung hai bảng, giá trị hai sequence, đơn hàng
   đánh dấu, `web_anon` đọc được `api.books`, `authenticator` đăng nhập qua TCP bằng mật khẩu **mới**.
   Ghi mốc `doi_chieu_xong`, rồi chạy **Khối B**.
9. Dọn: xóa namespace tạm và `restore.env`; PV và thư mục dữ liệu trên worker2 phải biến mất. Chạy
   **Khối C**.
10. `docs/runbook-backup.md`, mục **2. Kiểm chứng bản dump bằng DB tạm**: manifest đầy đủ nằm ngay
    trong runbook (vì `~/lab-work` bị dọn cuối sprint) và các lệnh theo thứ tự.
    `docs/postmortem/6b-dien-tap-khoi-phuc.md`: bảng thời gian đo được và trả lời câu hỏi thiết kế.

**Ràng buộc:** không ghi vào `db-0`, `data-db-0`, `backup-store` ngoài đơn hàng đánh dấu ở yêu cầu 1.
Không nạp trực tiếp từ `backup-store`. Không dùng `bookstore-retain` cho DB tạm. Không in mật khẩu.

**Đọc lại:**

- Bài [102](../102-volume-populators-vi.md): đoạn mở đầu,
  [*Sử dụng volume populator*](../102-volume-populators-vi.md#sử-dụng-volume-populator-using-volume-populators)
  — khung tư duy cho bước nạp dữ liệu vào volume mới.
- Bài [101](../101-volume-pvc-datasource-vi.md#giới-thiệu-introduction) — nguồn và đích nhân bản
  phải cùng namespace.
- Bài [84](../84-network-policies-vi.md#mặc-định-từ-chối-mọi-traffic-chiều-đến-và-chiều-đi-default-deny-all-ingress-and-all-egress-traffic),
  mục *Mặc định từ chối mọi traffic chiều đến và chiều đi*.
- Bài [92](../92-persistent-volumes-vi.md#reclaiming), mục *Thu hồi*; bài
  [96](../96-storage-classes-vi.md#reclaim-policy), mục *Chính sách thu hồi*.
- Docker Hub, image [postgres](https://hub.docker.com/_/postgres), mục *Initialization scripts* — đoạn
  về script lỗi giữa chừng.
- PostgreSQL 17, [*SQL Dump*](https://www.postgresql.org/docs/17/backup-dump.html), mục *Restoring
  the Dump*.
- [Mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm), đoạn cuối — mật khẩu khi
  dữ liệu được giữ lại hoặc nạp lại.

**Bàn giao:** `~/lab-work/6b/restore-ns.yaml`, `~/lab-work/6b/restore-db.yaml`;
`docs/runbook-backup.md` mục 2; `docs/postmortem/6b-dien-tap-khoi-phuc.md`; `~/lab-evidence/6b/b6-*`.

**Nghiệm thu:**

- `PASS:` đơn hàng đánh dấu có trong DB thật và trong bản dump ngoài cluster mới nhất.
- `PASS:` namespace tạm mang label dự án; `db-restore` Ready trên `lab-k8s-worker2`; `restore-data`
  dùng class mặc định với PV `Delete`; không Service nào.
- `PASS:` lúc vừa Ready, `db-restore` **chưa có** schema `api` nhưng đã có hai role.
- `PASS:` một Pod ngoài namespace không tới được cổng PostgreSQL của DB tạm.
- `PASS:` số dòng, checksum nội dung, sequence của `api.books` và `api.orders` khớp `db-0`; đơn hàng
  đánh dấu có mặt.
- `PASS:` `web_anon` đọc được `api.books`; `authenticator` đăng nhập qua TCP bằng mật khẩu mới; mật
  khẩu production **không** vào được.
- `PASS:` đủ năm mốc thời gian theo đúng thứ tự; `db-0` không bị khởi động lại hay thay thế.
- `PASS:` namespace tạm, PV và thư mục dữ liệu của DB tạm đã biến mất; `restore.env` đã xóa.
- `PASS:` runbook mục 2 và postmortem có mặt.

**Câu hỏi thiết kế** — trả lời trong `docs/postmortem/6b-dien-tap-khoi-phuc.md`:

1. Bảng thời gian của bạn: chặng nào gần như cố định, chặng nào tăng theo kích thước dữ liệu? Với
   50 GB dữ liệu, nút cổ chai nằm ở đâu trong cách nạp bạn dùng?
2. Ánh xạ diễn tập sang mô hình của bài 102: đâu là "nguồn dữ liệu", ai đóng vai "populator", và lúc
   nào volume được giao cho workload. Khối A cho thấy một khoảng hở mà populator thật không có — đó là
   gì, và bạn bịt nó bằng những gì?
3. Bản dump có mang role và mật khẩu không? Vì sao DB tạm dùng bí mật hoàn toàn mới vẫn cho
   `authenticator` đăng nhập? Đoạn cuối mục 5.4 của file dự án cảnh báo về mật khẩu cũ — cảnh báo đó
   đúng với trường hợp nào, và không áp vào trường hợp nào?
4. Vì sao không nạp bản dump bằng cách đặt nó vào `/docker-entrypoint-initdb.d` cho gọn? Chuyện gì
   xảy ra nếu bản dump lỗi giữa chừng và container khởi động lại? Một giao dịch duy nhất bảo vệ bạn
   khỏi điều gì, và cái giá của nó là gì?
5. Vì sao DB tạm dùng class mặc định chứ không `bookstore-retain`? Vì sao không "nhân bản"
   `data-db-0` sang namespace tạm cho nhanh, kể cả trên cluster có CSI? Diễn tập làm tăng dùng đĩa
   của worker2 bao nhiêu, và theo B3 có gì chặn nếu đĩa không đủ không?

<details>
<summary>Gợi ý</summary>

1. Ghi mốc: `moc() { echo "$1=$(date +%s)" >> ~/lab-evidence/6b/b6-thoi-gian.txt; }` rồi
   `moc bat_dau`, … Xóa file trước khi bắt đầu một lần diễn tập mới.
2. Đơn hàng đánh dấu — `-q` để psql chỉ in id, không in dòng `INSERT 0 1`:

   ```bash
   kubectl -n bookstore exec db-0 -c postgres -- psql -U postgres -d bookstore -qtAc \
     "INSERT INTO api.orders (book_id, quantity) SELECT min(id), 6 FROM api.books RETURNING id"
   ```

3. Bí mật mới: cùng cách `deploy.sh` sinh `secrets/db.env` (Lab 3b). Secret từ file
   `key=value`: `kubectl create secret generic ... --from-env-file="$HOME/lab-work/6b/restore.env"`
   — dùng `$HOME`, không dùng `~` sau dấu `=`.
4. Script khởi tạo: chép cách viết `02-authenticator.sh` của `db-init` (Lab 3b) — entrypoint *source*
   file `.sh` không có quyền thực thi; `psql --set=pw=...` và `:'pw'` để psql tự đặt nháy. Thứ tự
   tạo: namespace → Secret → ConfigMap, PVC, Pod.
5. Nạp: `kubectl exec -i ... -- psql -X -v ON_ERROR_STOP=1 --single-transaction ... < <file>`.
6. Đối chiếu nội dung một bảng:
   `SELECT md5(string_agg(t::text, '|' ORDER BY t.id)) FROM api.books t`. Sequence:
   `SELECT last_value FROM api.books_id_seq`. Lệnh có nhiều câu SQL thì thêm `-q` để psql không in
   dòng trạng thái như `SET`.
7. Đăng nhập TCP mà không lộ mật khẩu: đẩy mật khẩu qua stdin và nối tới **IP của Pod**, như Lab 3b
   B5 — `127.0.0.1` trong image này là `trust`, không kiểm mật khẩu.

</details>

<details>
<summary>Script nghiệm thu — ba khối, chạy đúng thời điểm ghi ở đầu mỗi khối</summary>

**Khối A** — ngay khi `db-restore` Ready, **trước** khi nạp:

```bash
NS=bookstore; RNS=bookstore-restore-test; EV=~/lab-evidence/6b; W=~/lab-work/6b; BK=~/bookstore-backups
q() { kubectl -n "$1" exec "$2" -c postgres -- psql -X -q -U postgres -d bookstore -tAc "$3" 2>/dev/null; }

DUMP="$(ls -1t "$BK"/*.sql | head -n1)"; MARK="$(cat "$EV/b6-dau-don-hang.txt")"
echo "dau vao: $DUMP ; don hang danh dau id=$MARK" | tee "$EV/b6-dau-vao.txt"
[ "$(q "$NS" db-0 "SELECT quantity FROM api.orders WHERE id = $MARK")" = '6' ] \
  && echo 'PASS: don hang danh dau co trong DB that'
awk '/^COPY api\.orders /{f=1;next} /^\\\.$/{f=0} f{print $1}' "$DUMP" | grep -qx "$MARK" \
  && echo 'PASS: ban dump ngoai cluster moi nhat chua don hang danh dau'

[ "$(kubectl get namespace "$RNS" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'bookstore' ] \
  && echo 'PASS: namespace tam mang label du an'
{ [ "$(kubectl -n "$RNS" get pod db-restore -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')" = 'True' ] && \
  [ "$(kubectl -n "$RNS" get pod db-restore -o jsonpath='{.spec.nodeName}')" = 'lab-k8s-worker2' ]; } \
  && echo 'PASS: db-restore Ready tren lab-k8s-worker2'
PVR="$(kubectl -n "$RNS" get pvc restore-data -o jsonpath='{.spec.volumeName}')"
{ [ "$(kubectl -n "$RNS" get pvc restore-data -o jsonpath='{.status.phase}/{.spec.storageClassName}')" = 'Bound/local-path' ] && \
  [ "$(kubectl get pv "$PVR" -o jsonpath='{.spec.persistentVolumeReclaimPolicy}')" = 'Delete' ]; } \
  && echo 'PASS: restore-data Bound, class mac dinh, PV Delete'
kubectl get pv "$PVR" -o jsonpath='{.metadata.name} {.spec.hostPath.path}{"\n"}' > "$EV/b6-pv.txt"
[ -z "$(kubectl -n "$RNS" get service -o name 2>/dev/null)" ] && echo 'PASS: khong Service nao tro toi DB tam'

[ "$(q "$RNS" db-restore "SELECT count(*) FROM pg_namespace WHERE nspname = 'api'")" = '0' ] \
  && echo 'PASS: db-restore Ready nhung CHUA co schema api — Ready khong co nghia la da khoi phuc'
[ "$(q "$RNS" db-restore "SELECT count(*) FROM pg_roles WHERE rolname IN ('web_anon','authenticator')")" = '2' ] \
  && echo 'PASS: hai role web_anon, authenticator co truoc khi nap'

RIP="$(kubectl -n "$RNS" get pod db-restore -o jsonpath='{.status.podIP}')"
kubectl apply -f "$W/thu-ns.yaml" >/dev/null
kubectl -n bookstore-thu run thu-pg-ngoai --rm -i --restart=Never --image=postgres:17.11 \
  -- pg_isready -h "$RIP" -p 5432 -t 3 > "$EV/b6-netpol.txt" 2>&1
grep -q 'no response' "$EV/b6-netpol.txt" && echo 'PASS: Pod ngoai namespace khong toi duoc DB tam'

kubectl -n "$NS" get pod db-0 \
  -o jsonpath='{.metadata.uid} {.status.containerStatuses[?(@.name=="postgres")].restartCount}' > "$EV/b6-db0-truoc.txt"
```

**Khối B** — sau khi nạp xong và đã ghi mốc `doi_chieu_xong`:

```bash
NS=bookstore; RNS=bookstore-restore-test; EV=~/lab-evidence/6b; W=~/lab-work/6b
q() { kubectl -n "$1" exec "$2" -c postgres -- psql -X -q -U postgres -d bookstore -tAc "$3" 2>/dev/null; }
MARK="$(cat "$EV/b6-dau-don-hang.txt")"

: > "$EV/b6-doi-chieu.txt"; BAD=0
for S in 'SELECT count(*) FROM api.books' \
         'SELECT count(*) FROM api.orders' \
         "SELECT coalesce(md5(string_agg(t::text, '|' ORDER BY t.id)), 'rong') FROM api.books t" \
         "SELECT coalesce(md5(string_agg(t::text, '|' ORDER BY t.id)), 'rong') FROM api.orders t" \
         'SELECT last_value FROM api.books_id_seq' \
         'SELECT last_value FROM api.orders_id_seq'; do
  A="$(q "$NS" db-0 "$S")"; B="$(q "$RNS" db-restore "$S")"
  printf '%s\n  goc=%s\n  khoi-phuc=%s\n' "$S" "$A" "$B" >> "$EV/b6-doi-chieu.txt"
  { [ -n "$A" ] && [ "$A" = "$B" ]; } || { echo "FAIL: lech o: $S"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: so dong, checksum noi dung, sequence cua books va orders khop db-0'
[ "$(q "$RNS" db-restore "SELECT quantity FROM api.orders WHERE id = $MARK")" = '6' ] \
  && echo "PASS: don hang danh dau $MARK co trong DB khoi phuc"

NB="$(q "$RNS" db-restore 'SELECT count(*) FROM api.books')"
[ "$(q "$RNS" db-restore 'SET ROLE web_anon; SELECT count(*) FROM api.books')" = "$NB" ] \
  && echo 'PASS: web_anon doc duoc api.books — quyen da duoc khoi phuc'
AUPW="$(sed -n 's/^authenticator-password=//p' "$W/restore.env")"
PROD_AU="$(sed -n 's/^authenticator-password=//p' ~/bookstore/secrets/db.env)"
RIP="$(kubectl -n "$RNS" get pod db-restore -o jsonpath='{.status.podIP}')"
login() { printf '%s' "$1" | kubectl -n "$RNS" exec -i db-restore -c postgres -- env H="$RIP" \
  sh -c 'PGPASSWORD="$(cat)" psql -X -q -h "$H" -U authenticator -d bookstore -tAc "SET ROLE web_anon; SELECT count(*) FROM api.books"' 2>/dev/null; }
[ -n "$AUPW" ] && [ "$AUPW" != "$PROD_AU" ] && echo 'PASS: DB tam dung bi mat rieng, khac secrets/db.env'
[ "$(login "$AUPW")" = "$NB" ] && echo 'PASS: authenticator dang nhap qua TCP bang mat khau MOI cua DB tam'
[ -n "$(login "$PROD_AU")" ] && echo 'FAIL: mat khau production vao duoc DB tam' \
  || echo 'PASS: mat khau production KHONG vao duoc — ban dump khong mang mat khau'

. "$EV/b6-thoi-gian.txt"
if [ "${bat_dau:-0}" -gt 0 ] && [ "${db_ready:-0}" -ge "$bat_dau" ] && [ "${nap_bat_dau:-0}" -ge "$db_ready" ] \
   && [ "${nap_xong:-0}" -ge "$nap_bat_dau" ] && [ "${doi_chieu_xong:-0}" -ge "$nap_xong" ]; then
  R=$((db_ready - bat_dau)); L=$((nap_xong - nap_bat_dau))
  echo "PASS: dung DB ${R}s, nap ${L}s, thoi gian may ${R}+${L}=$((R + L))s, ca dien tap $((doi_chieu_xong - bat_dau))s" \
    | tee "$EV/b6-thoi-gian-tong.txt"
fi
[ "$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.metadata.uid} {.status.containerStatuses[?(@.name=="postgres")].restartCount}')" \
  = "$(cat "$EV/b6-db0-truoc.txt")" ] && echo 'PASS: db-0 khong bi khoi dong lai hay thay the'
```

**Khối C** — sau khi dọn:

```bash
RNS=bookstore-restore-test; EV=~/lab-evidence/6b; W=~/lab-work/6b
kubectl get namespace "$RNS" >/dev/null 2>&1 && echo 'FAIL: namespace tam con' || echo 'PASS: namespace bookstore-restore-test da xoa'
read -r PVR PATHR < "$EV/b6-pv.txt"
for i in $(seq 1 60); do kubectl get pv "$PVR" >/dev/null 2>&1 || break; sleep 5; done
kubectl get pv "$PVR" >/dev/null 2>&1 && echo 'FAIL: PV cua DB tam con' || echo 'PASS: PV cua DB tam bien mat theo PVC (Delete)'
# Ba kết quả tách bạch: CON, MAT, hoặc rỗng khi ssh/sudo lỗi — rỗng không phải "đã dọn".
case "$(ssh -o BatchMode=yes lab-k8s-worker2 \
          "sudo -n true && if sudo test -e '$PATHR'; then echo CON; else echo MAT; fi")" in
  MAT) echo 'PASS: thu muc du lieu cua DB tam da duoc don' ;;
  CON) echo 'FAIL: thu muc du lieu cua DB tam con tren worker2' ;;
  *)   echo 'STOP: khong kiem duoc worker2 (ssh hoac sudo loi) — chua ket luan, sua ket noi roi chay lai khoi C' ;;
esac
test ! -e "$W/restore.env" && echo 'PASS: da xoa restore.env'
kubectl delete namespace bookstore-thu --ignore-not-found --wait=true
kubectl get namespace bookstore-thu >/dev/null 2>&1 || echo 'PASS: namespace bookstore-thu da xoa'
cd ~/bookstore
BAD=0
for K in 'bookstore-restore-test' 'single-transaction' 'CREATE ROLE' 'web_anon'; do
  grep -q -- "$K" docs/runbook-backup.md || { echo "FAIL: runbook muc 2 thieu $K"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: runbook muc 2 co manifest va lenh nap'
grep -q 'giây' docs/postmortem/6b-dien-tap-khoi-phuc.md && echo 'PASS: postmortem co bang thoi gian'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/6b/restore-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bookstore-restore-test
  labels:
    app.kubernetes.io/part-of: bookstore
---
# DB khôi phục chứa dữ liệu thật: không ai trong cluster được nói chuyện với nó qua mạng.
# Nạp dữ liệu và đối chiếu đi qua kubectl exec (API server -> kubelet), không qua mạng Pod.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: chan-tat-ca
  namespace: bookstore-restore-test
  labels:
    app.kubernetes.io/part-of: bookstore
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
```

```yaml
# ~/lab-work/6b/restore-db.yaml — áp sau khi đã có Secret restore-credentials
apiVersion: v1
kind: ConfigMap
metadata:
  name: restore-init
  namespace: bookstore-restore-test
  labels:
    app.kubernetes.io/part-of: bookstore
data:
  01-roles.sh: |
    # Entrypoint "source" file này dưới user postgres, chỉ khi thư mục dữ liệu rỗng.
    # Chỉ tạo role — bản dump của pg_dump không chứa role. Schema và dữ liệu đến từ bản dump.
    if [ -z "${AUTHENTICATOR_PASSWORD:-}" ]; then
      echo '01-roles.sh: thieu AUTHENTICATOR_PASSWORD' >&2
      exit 1
    fi
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" \
      --set=pw="$AUTHENTICATOR_PASSWORD" <<'EOSQL'
    CREATE ROLE web_anon NOLOGIN;
    CREATE ROLE authenticator NOINHERIT LOGIN PASSWORD :'pw';
    GRANT web_anon TO authenticator;
    EOSQL
    echo '01-roles.sh: da tao web_anon va authenticator'
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-data
  namespace: bookstore-restore-test
  labels:
    app.kubernetes.io/part-of: bookstore
spec:
  accessModes: ["ReadWriteOnce"]   # không khai class: local-path mặc định, reclaimPolicy Delete
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: db-restore
  namespace: bookstore-restore-test
  labels:
    app.kubernetes.io/part-of: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  terminationGracePeriodSeconds: 60
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: restore-data
  - name: initdb
    configMap:
      name: restore-init
  - name: credentials
    secret:
      secretName: restore-credentials
      defaultMode: 0400
      items:
      - key: postgres-password
        path: postgres-password
  containers:
  - name: postgres
    image: postgres:17.11
    env:
    - {name: POSTGRES_DB, value: bookstore}
    - {name: POSTGRES_PASSWORD_FILE, value: /etc/db-credentials/postgres-password}
    - {name: PGDATA, value: /var/lib/postgresql/data/pgdata}
    - name: AUTHENTICATOR_PASSWORD
      valueFrom:
        secretKeyRef: {name: restore-credentials, key: authenticator-password}
    ports:
    - {name: postgres, containerPort: 5432}
    readinessProbe:
      exec:
        command: ["pg_isready", "-h", "127.0.0.1", "-p", "5432", "-U", "postgres", "-d", "bookstore"]
      periodSeconds: 5
      timeoutSeconds: 3
    resources:
      requests: {cpu: 250m, memory: 256Mi}
      limits: {cpu: 500m, memory: 512Mi}
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
    - name: initdb
      mountPath: /docker-entrypoint-initdb.d
      readOnly: true
    - name: credentials
      mountPath: /etc/db-credentials
      readOnly: true
```

Trình tự diễn tập — đây cũng là phần lệnh của runbook mục 2:

```bash
EV=~/lab-evidence/6b; W=~/lab-work/6b; RNS=bookstore-restore-test
moc() { echo "$1=$(date +%s)" >> "$EV/b6-thoi-gian.txt"; }

kubectl -n bookstore exec db-0 -c postgres -- psql -U postgres -d bookstore -qtAc \
  "INSERT INTO api.orders (book_id, quantity) SELECT min(id), 6 FROM api.books RETURNING id" > "$EV/b6-dau-don-hang.txt"
# ... chạy runbook mục 1: bản dump mới, chép ra ~/bookstore-backups, checksum ...
DUMP="$(ls -1t ~/bookstore-backups/*.sql | head -n1)"

rm -f "$EV/b6-thoi-gian.txt"; moc bat_dau
kubectl apply -f "$W/restore-ns.yaml"
( umask 077
  printf 'postgres-password=%s\nauthenticator-password=%s\n' \
    "$(openssl rand -hex 24)" "$(openssl rand -hex 24)" > "$W/restore.env" )
kubectl -n "$RNS" create secret generic restore-credentials --from-env-file="$HOME/lab-work/6b/restore.env"
kubectl apply -f "$W/restore-db.yaml"
kubectl -n "$RNS" wait --for=condition=Ready pod/db-restore --timeout=300s; moc db_ready
# >>> chạy Khối A ở đây <<<

moc nap_bat_dau
kubectl -n "$RNS" exec -i db-restore -c postgres -- \
  psql -X -q -v ON_ERROR_STOP=1 --single-transaction -U postgres -d bookstore < "$DUMP" \
  && echo "nap xong, psql exit=0"
moc nap_xong
# ... tự đối chiếu như yêu cầu 8 ...
moc doi_chieu_xong
# >>> chạy Khối B ở đây <<<

kubectl delete namespace "$RNS" --wait=true --timeout=300s
rm -f "$W/restore.env"
# >>> chạy Khối C ở đây <<<
```

Ý chính cho postmortem:

1. Dựng DB (kéo image sẵn có, `initdb`, script role) gần như cố định; nạp và đối chiếu tăng theo dữ
   liệu. Với cách nạp này, mọi byte đi từ master qua API server tới kubelet bằng luồng `exec` — đường
   ống hẹp nhất. Dữ liệu lớn cần bản dump nằm sẵn cạnh DB đích (một volume được điền trước) — đúng bài
   toán mà populator giải.
2. Nguồn dữ liệu là bản dump ngoài cluster; populator là bạn cộng `kubectl exec ... psql`; volume được
   "giao" cho workload ngay khi Pod Ready — **trước** khi có dữ liệu. Populator thật chỉ giao PVC sau
   khi đã điền xong (bài 102), nên workload không bao giờ thấy volume rỗng. Khối A cho thấy DB tạm
   Ready mà rỗng. Bịt bằng: không có Service, NetworkPolicy chặn hai chiều, và quy tắc "chỉ công bố
   khôi phục xong sau khi đối chiếu PASS".
3. Không. PostgreSQL docs: `pg_dump` chỉ dump một database và không dump role, vì role là đối tượng
   toàn cluster PostgreSQL. Bản dump chỉ **tham chiếu** `web_anon` trong các lệnh cấp quyền, và
   PostgreSQL đòi role tồn tại trước khi nạp. Role và mật khẩu đến từ script khởi tạo của DB đích, tức
   từ Secret của môi trường đích. Cảnh báo ở mục 5.4 đúng khi **thư mục dữ liệu** được giữ lại (PV
   `Retain` của Lab 6a: `pg_authid` nằm trong đó, mật khẩu cũ đi theo); nó không áp vào bản dump của
   `pg_dump` — điều đó làm việc dựng lại cluster ở Lab 8a đơn giản hơn: bí mật mới, dữ liệu cũ.
4. Docker Hub: script trong `/docker-entrypoint-initdb.d` chỉ chạy khi thư mục dữ liệu rỗng, và nếu
   một script lỗi rồi container khởi động lại thì entrypoint thấy thư mục đã khởi tạo và **không chạy
   tiếp** — DB lên Ready với dữ liệu thiếu. `--single-transaction` kèm `ON_ERROR_STOP`: lỗi ở đâu cũng
   rollback toàn bộ, nạp lại được ngay; cái giá là một lỗi nhỏ ở cuối làm mất cả lần nạp dài.
5. DB tạm không có giá trị sau diễn tập: `Delete` để provisioner dọn PV và thư mục; `Retain` để lại
   rác cần dọn tay. Nhân bản chỉ trong cùng namespace (bài 101) và trên cluster này còn cho volume
   rỗng (B2); nó cũng không kiểm chứng gì về bản dump — mục đích của diễn tập. Diễn tập tăng dùng đĩa
   worker2 đúng bằng kích thước DB khôi phục, và B3 cho thấy không có gì chặn nếu đĩa không đủ.

</details>

## B7. [BS-6B-07] ⭐ Diễn tập xóa nhầm dữ liệu, khôi phục tại chỗ

**Loại:** Diễn tập sự cố — tùy chọn, trên `db-0` ở `lab-k8s-worker2`.

**Bối cảnh:** sự cố hay gặp nhất không phải mất node mà là **một lệnh SQL chạy nhầm**. Khi đó PV
`Retain` không giúp gì — dữ liệu sai đã được ghi bền vững. Cách cứu duy nhất là bản dump. Lab 8a sẽ
khôi phục sau khi dựng lại cả cluster; ticket này diễn tập quy trình khôi phục **tại chỗ** vào `db-0`
đang chạy, và cho bạn thấy RPO bằng một đơn hàng cụ thể.

**Yêu cầu:**

1. Chạy runbook mục 1 để có bản dump mới ngoài cluster, rồi chạy **Khối A** — script kiểm bản dump khớp
   DB hiện tại, ghi trạng thái, và đặt một đơn hàng **sau** bản dump.
2. Chạy **Khối B** — script gây sự cố: xóa sạch đơn hàng và đặt tồn kho mọi sách về 0. Khách nhìn qua
   web thấy toàn bộ sách hết hàng.
3. Khôi phục `db-0` theo quy trình bạn viết thành `docs/runbook-backup.md` mục **3. Khôi phục tại chỗ
   vào db-0**: thay toàn bộ schema `api` bằng nội dung bản dump trong **một** giao dịch; không đổi mật
   khẩu, không khởi động lại `api`; báo PostgREST nạp lại schema cache.
4. Chạy **Khối C**. Viết `docs/postmortem/6b-xoa-nham-du-lieu.md`.

**Ràng buộc:** không xóa `db-0` hay `data-db-0`, không teardown, không sửa Secret. Lỗi giữa chừng
không được để DB ở trạng thái nửa vời.

**Đọc lại:** PostgreSQL 17, [*SQL Dump*](https://www.postgresql.org/docs/17/backup-dump.html), mục
*Restoring the Dump*; PostgREST v14,
[Schema Cache](https://docs.postgrest.org/en/v14/references/schema_cache.html), mục *Schema Cache
Reloading*; bài [92](../92-persistent-volumes-vi.md#reclaiming) — `Retain` giữ dữ liệu, kể cả dữ
liệu sai.

**Bàn giao:** `docs/runbook-backup.md` mục 3; `docs/postmortem/6b-xoa-nham-du-lieu.md`;
`~/lab-evidence/6b/b7-*`.

**Nghiệm thu:**

- `PASS:` bản dump ngoài cluster mới nhất khớp DB trước sự cố; một đơn hàng được đặt sau bản dump.
- `PASS:` sự cố xảy ra: web trả mọi sách tồn kho 0.
- `PASS:` sau khôi phục, `api.books` đúng như trước sự cố; số đơn hàng bằng số trong bản dump, đơn
  hàng đặt sau bản dump **không còn** — đó là RPO.
- `PASS:` web trả lại tồn kho thật; Pod `api` không khởi động lại.
- `PASS:` runbook mục 3 và postmortem có mặt.

**Câu hỏi thiết kế** — trả lời trong `docs/postmortem/6b-xoa-nham-du-lieu.md`:

1. Đơn hàng nào mất, vì sao? RPO quan sát được là bao nhiêu phút, và nó phụ thuộc vào điều gì?
2. Vì sao xóa schema cũ và nạp bản dump phải nằm trong **cùng một** giao dịch? Tách hai bước mà bước
   nạp lỗi thì khách thấy gì?
3. Vì sao không phải đổi mật khẩu hay khởi động lại `api`? Vì sao vẫn phải báo PostgREST?
4. Khi nào chọn khôi phục tại chỗ, khi nào khôi phục sang DB mới như B6 rồi mới chuyển?

<details>
<summary>Gợi ý</summary>

- Ghép một câu lệnh vào trước bản dump và đẩy cả hai vào cùng một `psql --single-transaction`:
  `{ echo '<câu lệnh>'; cat "$DUMP"; } | kubectl exec -i ...`.
- `DROP SCHEMA ... CASCADE` xóa schema cùng mọi bảng, sequence, quyền bên trong; role nằm ngoài schema
  nên không bị đụng.
- PostgREST nghe kênh `pgrst`: `NOTIFY pgrst, 'reload schema'`.

</details>

<details>
<summary>Script nghiệm thu — ba khối</summary>

**Khối A** — sau khi chạy runbook mục 1:

```bash
NS=bookstore; EV=~/lab-evidence/6b; BK=~/bookstore-backups
q() { kubectl -n "$NS" exec db-0 -c postgres -- psql -X -q -U postgres -d bookstore -tAc "$1" 2>/dev/null; }
DUMP="$(ls -1t "$BK"/*.sql | head -n1)"
DUMP_ORDERS="$(awk '/^COPY api\.orders /{f=1;next} /^\\\.$/{f=0} f' "$DUMP" | wc -l)"
A_ORDERS="$(q 'SELECT count(*) FROM api.orders')"
A_BOOKS="$(q "SELECT md5(string_agg(t::text, '|' ORDER BY t.id)) FROM api.books t")"
[ "$DUMP_ORDERS" -eq "${A_ORDERS:--1}" ] && echo "PASS: ban dump moi nhat khop DB hien tai ($A_ORDERS don hang)"
LATE="$(q 'INSERT INTO api.orders (book_id, quantity) SELECT min(id), 7 FROM api.books RETURNING id')"
printf 'DUMP=%s\nA_ORDERS=%s\nA_BOOKS=%s\nLATE=%s\n' "$DUMP" "$A_ORDERS" "$A_BOOKS" "$LATE" > "$EV/b7-truoc.env"
kubectl -n "$NS" get pod -l app.kubernetes.io/component=api \
  -o jsonpath='{range .items[*]}{.metadata.name}={.status.containerStatuses[0].restartCount}{"\n"}{end}' > "$EV/b7-api-truoc.txt"
[ -n "$LATE" ] && echo "PASS: da dat don hang $LATE SAU ban dump"
```

**Khối B** — sự cố, chạy ngay sau Khối A:

```bash
NS=bookstore; EV=~/lab-evidence/6b
kubectl -n "$NS" exec db-0 -c postgres -- psql -X -q -U postgres -d bookstore \
  -c 'DELETE FROM api.orders; UPDATE api.books SET stock = 0;'
WEB="$(kubectl -n "$NS" get pod -l app.kubernetes.io/component=web --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')"
S="$(kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -qO- http://127.0.0.1:8080/api/books \
     | grep -o '"stock" *: *[0-9]*' | sed 's/.*: *//' | sort -u | tr '\n' ' ')"
echo "ton kho khach thay: $S" | tee "$EV/b7-su-co.txt"
[ "$S" = '0 ' ] && echo 'PASS: su co da xay ra — moi sach het hang, don hang mat'
```

**Khối C** — sau khi khôi phục theo runbook mục 3:

```bash
NS=bookstore; EV=~/lab-evidence/6b
q() { kubectl -n "$NS" exec db-0 -c postgres -- psql -X -q -U postgres -d bookstore -tAc "$1" 2>/dev/null; }
. "$EV/b7-truoc.env"
[ "$(q "SELECT md5(string_agg(t::text, '|' ORDER BY t.id)) FROM api.books t")" = "$A_BOOKS" ] \
  && echo 'PASS: api.books dung nhu truoc su co'
{ [ "$(q 'SELECT count(*) FROM api.orders')" = "$A_ORDERS" ] && \
  [ "$(q "SELECT count(*) FROM api.orders WHERE id = $LATE")" = '0' ]; } \
  && echo "PASS: con $A_ORDERS don hang cua ban dump; don hang $LATE dat sau ban dump DA MAT — do la RPO"
WEB="$(kubectl -n "$NS" get pod -l app.kubernetes.io/component=web --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')"
J="$(kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -qO- http://127.0.0.1:8080/api/books)"
{ [ "$(echo "$J" | grep -o '"title"' | wc -l)" -ge 8 ] && \
  echo "$J" | grep -o '"stock" *: *[0-9]*' | sed 's/.*: *//' | grep -qv '^0$'; } \
  && echo 'PASS: web tra lai ton kho that qua api'
kubectl -n "$NS" get pod -l app.kubernetes.io/component=api \
  -o jsonpath='{range .items[*]}{.metadata.name}={.status.containerStatuses[0].restartCount}{"\n"}{end}' \
  | diff -q - "$EV/b7-api-truoc.txt" >/dev/null && echo 'PASS: Pod api khong khoi dong lai, mat khau khong doi'
cd ~/bookstore
{ grep -q 'single-transaction' docs/runbook-backup.md && grep -q 'DROP SCHEMA' docs/runbook-backup.md && \
  grep -q 'reload schema' docs/runbook-backup.md && test -s docs/postmortem/6b-xoa-nham-du-lieu.md; } \
  && echo 'PASS: runbook muc 3 va postmortem co mat'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Lệnh của runbook mục 3:

```bash
NS=bookstore
DUMP="$(ls -1t ~/bookstore-backups/*.sql | head -n1)"     # bản ngoài cluster đã kiểm checksum
{ echo 'DROP SCHEMA IF EXISTS api CASCADE;'; cat "$DUMP"; } \
  | kubectl -n "$NS" exec -i db-0 -c postgres -- \
      psql -X -q -v ON_ERROR_STOP=1 --single-transaction -U postgres -d bookstore \
  && echo 'khoi phuc xong, psql exit=0'
kubectl -n "$NS" exec db-0 -c postgres -- psql -X -U postgres -d bookstore -c "NOTIFY pgrst, 'reload schema'"
```

Ý chính cho postmortem:

1. Đơn hàng đặt sau thời điểm `pg_dump` bắt đầu. RPO là khoảng từ lúc bản dump được tạo tới lúc sự cố
   — tối đa một chu kỳ `db-backup` nếu bản dump trong cluster còn, hoặc từ lần chép ra ngoài gần nhất
   nếu không.
2. Cùng giao dịch: hoặc schema cũ còn nguyên, hoặc schema mới đầy đủ — không có khoảnh khắc nào khách
   thấy bảng không tồn tại. Tách hai bước mà nạp lỗi thì `api` trả lỗi cho mọi request tới khi có người
   sửa tay.
3. Role và mật khẩu nằm ngoài schema `api` và ngoài bản dump: `DROP SCHEMA` không đụng tới chúng, nên
   `authenticator` vẫn đăng nhập bằng mật khẩu trong Secret hiện tại. PostgREST giữ schema cache của
   các bảng cũ; tài liệu PostgREST yêu cầu nạp lại cache sau mỗi thay đổi schema.
4. Tại chỗ: nhanh, nhưng mọi ghi sau bản dump mất và không còn đường lui nếu bản dump hỏng. Sang DB mới
   (B6): chậm hơn, DB gốc vẫn còn để so và để trích phần dữ liệu đặt sau bản dump — chọn khi chưa chắc
   bản dump tốt hay khi phần dữ liệu mất có giá trị.

</details>

## B8. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 9 cùng chiến lược sao lưu
đã diễn tập, chép bundle **và** bản dump mới nhất ra máy host, rồi trả cluster về đúng
`03-storage-ready`.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 6b — kiểm thử khói **tích lũy**, chép nguyên văn.
   Script kiểm mọi hợp đồng còn hiệu lực ở cuối sprint 9 theo
   [mục 4 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật), cộng một giả định của sprint này:
   cluster vẫn chưa có API `VolumeSnapshot` — điều kiện để ADR sao lưu logic còn hiệu lực. Giả định thứ
   hai — diễn tập đã dọn sạch — được kiểm ở bước 3, trước teardown.

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 6b (sprint 9). Tích lũy mọi hợp đồng còn hiệu lực:
# Pod và controller (3a–4b), cấu hình và bí mật (3b), tài nguyên và PDB (3c), Service và DNS (5a),
# NetworkPolicy và HTTPS qua Ingress (5b), dữ liệu bền (6a), giả định của chiến lược sao lưu (6b).
# Giữ mọi kiểm tra của smoke Lab 6a — sprint 9 không đổi manifest. Script không in mật khẩu và không
# đọc nội dung bản dump.
set -u
cd "$(dirname "$0")/.."
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
HOST=bookstore.lab.local
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
k() { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
SQL() { k exec db-0 -c postgres -- psql -X -q -U postgres -d bookstore -tAc "$1" 2>/dev/null; }
has_cm() {  # $1 = các tên ConfigMap workload tham chiếu; $2 = tên theo hợp đồng (từ 5a có thể kèm hậu tố hash)
  for N in $1; do case "$N" in "$2"|"$2"-*) k get configmap "$N" >/dev/null 2>&1 && return 0 ;; esac; done
  return 1
}

# 1. Namespace, object cấp cluster, hạ tầng nền (02-net-ready, 03-storage-ready)
[ "$(kubectl get namespace "$NS" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null)" = 'bookstore' ] \
  && pass 'namespace bookstore mang label du an' || fail 'namespace bookstore thieu hoac thieu label'
PRIO="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}' 2>/dev/null)"
[ -n "$PRIO" ] && pass "co PriorityClass bookstore-critical ($PRIO)" || fail 'thieu PriorityClass bookstore-critical'
NODES="$(kubectl get nodes --no-headers | wc -l)"
R="$(kubectl -n kube-system get daemonset calico-node -o jsonpath='{.status.numberReady}' 2>/dev/null)"
[ "${R:-0}" -eq "$NODES" ] && pass "Calico Ready tren $R/$NODES node" || fail "Calico ${R:-0}/$NODES node"
[ "$(kubectl get ingressclass traefik -o jsonpath='{.metadata.annotations.ingressclass\.kubernetes\.io/is-default-class}' 2>/dev/null)" = 'true' ] \
  && pass 'IngressClass traefik la mac dinh' || fail 'thieu IngressClass traefik mac dinh'
SCJ="$(kubectl get storageclass bookstore-retain -o jsonpath='{.provisioner}|{.reclaimPolicy}|{.volumeBindingMode}|{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null)"
[ "$SCJ" = 'rancher.io/local-path|Retain|WaitForFirstConsumer|bookstore' ] \
  && pass 'StorageClass bookstore-retain: local-path, Retain, WaitForFirstConsumer' || fail "StorageClass bookstore-retain sai ($SCJ)"
DEF="$(kubectl get storageclass -o jsonpath='{range .items[*]}{.metadata.name}={.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}' 2>/dev/null | grep '=true$')"
[ "$DEF" = 'local-path=true' ] && pass 'class mac dinh duy nhat la local-path' || fail "class mac dinh sai: $(echo $DEF)"

# 2. Cấu hình, bí mật, PDB (3b, 3c, 5b) — tên ConfigMap đọc từ spec của workload; từ 6a web-config có
#    thể đi qua volume projected
has_cm "$(jp deployment/web '{.spec.template.spec.volumes[*].configMap.name} {.spec.template.spec.volumes[*].projected.sources[*].configMap.name}')" web-config \
  && pass 'web dung ConfigMap web-config' || fail 'web khong tham chieu ConfigMap web-config ton tai'
has_cm "$(jp deployment/api '{.spec.template.spec.containers[*].envFrom[*].configMapRef.name}')" api-config \
  && pass 'api dung ConfigMap api-config' || fail 'api khong tham chieu ConfigMap api-config ton tai'
has_cm "$(jp statefulset/db '{.spec.template.spec.volumes[*].configMap.name}')" db-init \
  && pass 'db dung ConfigMap db-init' || fail 'db khong tham chieu ConfigMap db-init ton tai'
D="$(jp secret/db-credentials '{.data}')"
{ echo "$D" | grep -q '"postgres-password"' && echo "$D" | grep -q '"authenticator-password"'; } \
  && pass 'Secret db-credentials du hai key' || fail 'Secret db-credentials thieu key'
D="$(jp secret/bookstore-tls '{.data}')"
{ [ "$(jp secret/bookstore-tls '{.type}')" = 'kubernetes.io/tls' ] && echo "$D" | grep -q '"tls.crt"' \
  && echo "$D" | grep -q '"tls.key"'; } \
  && pass 'Secret bookstore-tls kieu TLS, du tls.crt va tls.key' || fail 'Secret bookstore-tls sai kieu hoac thieu key'
for P in web api; do
  k get pdb "$P" >/dev/null 2>&1 && pass "co PodDisruptionBudget $P" || fail "thieu PodDisruptionBudget $P"
done

# 3. Controller, tài nguyên, emptyDir (3c, 4a, 4b, 6a)
[ "$(jp statefulset/db '{.status.readyReplicas}')" = '1' ] && pass 'StatefulSet db 1/1 Ready' || fail 'StatefulSet db chua Ready'
[ "$(jp statefulset/db '{.spec.serviceName}')" = 'db' ] && pass 'StatefulSet db khai serviceName db' || fail 'serviceName khong phai db'
[ "$(jp pod/db-0 '{.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].name}')" = 'StatefulSet/db' ] \
  && pass 'db-0 thuoc StatefulSet db' || fail 'db-0 khong thuoc StatefulSet db'
[ "$(jp pod/db-0 '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && pass 'db-0 tren lab-k8s-worker2' || fail 'db-0 khong o lab-k8s-worker2'
[ "$(jp pod/db-0 '{.status.qosClass}')" = 'Guaranteed' ] && pass 'db-0 QoS Guaranteed' || fail 'db-0 khong Guaranteed'
{ [ -n "$PRIO" ] && [ "$(jp pod/db-0 '{.spec.priority}')" = "$PRIO" ]; } && pass 'db-0 dung bookstore-critical' || fail 'db-0 sai priority'
for N in web api; do
  R="$(jp "deployment/$N" '{.spec.replicas}/{.status.readyReplicas}')"
  { [ "${R%%/*}" = "${R##*/}" ] && [ "${R%%/*}" -ge 2 ] 2>/dev/null; } \
    && pass "Deployment $N $R Ready" || fail "Deployment $N chua du ban sao Ready ($R)"
done
ORPHAN="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}:{.metadata.ownerReferences[0].kind}{"\n"}{end}' 2>/dev/null | grep -c ':$')"
[ "$ORPHAN" -eq 0 ] && pass 'moi Pod cua du an deu co controller' || fail "$ORPHAN Pod tran"
MISS="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{end}' 2>/dev/null | grep -E '=,|,$' | sort -u)"
[ -z "$MISS" ] && pass 'moi container co requests va limits' || fail "container thieu tai nguyen: $(echo $MISS)"
NOLIM="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.volumes[*]}{.name}={.emptyDir}{"\n"}{end}{end}' 2>/dev/null | grep '={}$' | sort -u)"
[ -z "$NOLIM" ] && pass 'moi emptyDir cua du an co sizeLimit' || fail "emptyDir khong tran: $(echo $NOLIM)"

# 4. Dữ liệu và migration (4b)
N="$(SQL 'SELECT count(*) FROM api.books')"
[ "${N:-0}" -ge 8 ] && pass "db co $N cuon sach" || fail "db co ${N:-0} cuon sach, can >= 8"
if k get job db-migrate >/dev/null 2>&1; then
  [ "$(jp job/db-migrate '{.status.conditions[?(@.type=="Complete")].status}')" = 'True' ] \
    && pass 'Job db-migrate Complete' || fail 'Job db-migrate chua Complete'
else
  pass 'Job db-migrate da duoc TTL don, ket qua nam trong db'
fi

# 5. Dữ liệu bền của db (6a)
[ "$(jp pvc/data-db-0 '{.status.phase}|{.spec.storageClassName}|{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'Bound|bookstore-retain|bookstore' ] \
  && pass 'PVC data-db-0 Bound, class bookstore-retain, label du an' || fail 'PVC data-db-0 sai hoac chua Bound'
VOL="$(jp pod/db-0 '{.spec.containers[?(@.name=="postgres")].volumeMounts[?(@.mountPath=="/var/lib/postgresql/data")].name}')"
{ [ -n "$VOL" ] && [ "$(jp pod/db-0 "{.spec.volumes[?(@.name==\"$VOL\")].persistentVolumeClaim.claimName}")" = 'data-db-0' ]; } \
  && pass 'du lieu db-0 nam tren PVC data-db-0' || fail 'db-0 khong dung PVC data-db-0'
PGD="$(jp pod/db-0 '{.spec.containers[?(@.name=="postgres")].env[?(@.name=="PGDATA")].value}')"
case "$PGD" in
  /var/lib/postgresql/data/?*) pass "PGDATA=$PGD la thu muc con cua volume" ;;
  *) fail "PGDATA='$PGD' khong phai thu muc con cua volume" ;;
esac
PV="$(jp pvc/data-db-0 '{.spec.volumeName}')"
PVJ="$(kubectl get pv "$PV" -o jsonpath='{.spec.persistentVolumeReclaimPolicy}|{.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}' 2>/dev/null)"
[ "$PVJ" = "Retain|$(jp pod/db-0 '{.spec.nodeName}')" ] \
  && pass "PV cua data-db-0: Retain, cung node voi db-0" || fail "PV cua data-db-0 sai ($PVJ)"
[ "$(jp statefulset/db '{.spec.persistentVolumeClaimRetentionPolicy.whenDeleted}|{.spec.persistentVolumeClaimRetentionPolicy.whenScaled}')" = 'Retain|Retain' ] \
  && pass 'StatefulSet db giu PVC khi xoa va khi thu nho' || fail 'chinh sach giu PVC cua db sai'

# 6. Service và DNS (5a); web về ClusterIP, NetworkPolicy chặn mặc định (5b)
svc() {  # $1 tên; $2 cổng Service; $3 các targetPort chấp nhận, ngăn bằng |
  local P T; P="$(jp "service/$1" '{.spec.ports[0].port}')"; T="$(jp "service/$1" '{.spec.ports[0].targetPort}')"
  { [ "$P" = "$2" ] && echo "|$3|" | grep -qF "|$T|"; } && pass "Service $1 cong $P -> $T" || fail "Service $1 sai cong ($P -> $T)"
}
svc web 80 'http|8080'; svc api 3000 'http|3000'; svc db 5432 'postgres|5432'; svc node-agent 9100 'http|9100'
{ [ "$(jp service/web '{.spec.type}')" = 'ClusterIP' ] && [ -z "$(jp service/web '{.spec.ports[*].nodePort}')" ]; } \
  && pass 'Service web la ClusterIP, khong con NodePort' || fail 'Service web van mo NodePort'
[ "$(jp service/db '{.spec.clusterIP}')" = 'None' ] && pass 'Service db headless' || fail 'Service db khong headless'
[ "$(jp service/node-agent '{.spec.internalTrafficPolicy}')" = 'Local' ] \
  && pass 'Service node-agent internalTrafficPolicy Local' || fail 'Service node-agent khong Local'
DENY="$(jp networkpolicy '{range .items[*]}{.spec.podSelector}|{.spec.policyTypes}{"\n"}{end}' | grep '^{}|' | grep 'Ingress' | grep -c 'Egress')"
[ "${DENY:-0}" -ge 1 ] && pass 'co NetworkPolicy chan mac dinh hai chieu cho ca namespace' || fail 'thieu NetworkPolicy chan mac dinh'
DB_IP="$(jp pod/db-0 '{.status.podIP}')"

# 7. web, api, node-agent bên trong cluster — các luồng ma trận cho phép (5a, 5b, 6a)
WEB="$(k get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
WNODE="$(jp "pod/$WEB" '{.spec.nodeName}')"
get() { k exec "$WEB" -c status-writer -- wget -qO- -T 5 "$1" 2>/dev/null; }
# Hỏi DNS của cluster từ Pod khác: trong chính db-0, tên này nằm sẵn ở /etc/hosts do kubelet ghi (5a).
k exec "$WEB" -c status-writer -- nslookup "db-0.db.$NS.svc.cluster.local" 2>/dev/null | grep -qwF "$DB_IP" \
  && pass 'ten db-0.db phan giai ve IP cua db-0' || fail 'ten db-0.db khong phan giai dung'
T="$(get http://api:3000/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "Service api /books tra $T cuon" || fail "Service api /books tra $T cuon"
get http://127.0.0.1:8080/ | grep -q 'Bookstore' && pass 'web phuc vu trang chu' || fail 'web khong phuc vu trang chu'
S="$(get http://127.0.0.1:8080/status.json)"
{ echo "$S" | grep -Eq "\"pod\" *: *\"$WEB\"" && echo "$S" | grep -q '"version"' && echo "$S" | grep -q '"updated"'; } \
  && pass 'status.json dung ten Pod, co version va updated' || fail 'status.json sai hoac thieu'
T="$(get http://127.0.0.1:8080/api/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "nginx proxy /api/ toi api ($T cuon)" || fail "nginx /api/ tra $T cuon"
PROJ="$(jp "pod/$WEB" '{range .spec.volumes[*]}{.name}|{.projected.sources[*].downwardAPI.items[*].path}|{.projected.sources[*].configMap.name}{"\n"}{end}' \
        | grep -v '^kube-api-access-' | awk -F'|' '$2!="" && $3 ~ /^web-config/ {print $1; exit}')"
{ [ -n "$PROJ" ] && [ "$(jp "pod/$WEB" "{.spec.initContainers[?(@.name==\"status-writer\")].volumeMounts[?(@.name==\"$PROJ\")].mountPath}")" = '/etc/status-writer' ]; } \
  && pass 'status-writer doc label va cau hinh tu mot volume projected' || fail 'status-writer chua dung volume projected'
get http://node-agent:9100/node.json | grep -Eq "\"node\" *: *\"$WNODE\"" \
  && pass 'node-agent qua Service tra dung node cua Pod goi' || fail 'node-agent qua Service sai node'
R="$(jp daemonset/node-agent '{.status.numberReady}')"
[ "${R:-0}" -eq "$NODES" ] && pass "node-agent Ready tren $R/$NODES node" || fail "node-agent ${R:-0}/$NODES node"
[ "$(jp daemonset/node-agent '{.spec.template.spec.containers[?(@.name=="agent")].ports[?(@.name=="http")].containerPort}')" = '9100' ] \
  && pass 'node-agent cong http 9100' || fail 'node-agent sai cong'
BAD=0
for L in $(k get pod -l "$SEL,app.kubernetes.io/component=agent" -o jsonpath='{range .items[*]}{.spec.nodeName}={.status.podIP}{"\n"}{end}' 2>/dev/null); do
  get "http://${L##*=}:9100/node.json" | grep -Eq "\"node\" *: *\"${L%%=*}\"" || { echo "FAIL: node-agent tren ${L%%=*} sai /node.json"; BAD=1; }
done
[ "$BAD" -eq 0 ] && pass 'moi node-agent tra /node.json dung ten node' || FAIL=1

# 8. Các luồng ma trận KHÔNG cho phép phải bị chặn (5b). Phủ định chỉ tính khi kết nối hết giờ hoặc bị
#    từ chối, và có đối chứng dương; lỗi khác là "không kết luận được", không phải PASS.
WEB_IP="$(jp "pod/$WEB" '{.status.podIP}')"
# Probe phải được phép đi ra thì mới thử được ingress của web: trong bookstore, default-deny chặn luôn
# egress của chính probe. Dùng namespace thử bookstore-thu (không NetworkPolicy, mang label dự án),
# chỉ xóa nếu smoke tự tạo. DNS-OK là đối chứng dương: probe có mạng.
PNS=bookstore-thu; MADE=0
if ! kubectl get namespace "$PNS" >/dev/null 2>&1; then
  kubectl create namespace "$PNS" >/dev/null 2>&1 \
    && kubectl label namespace "$PNS" app.kubernetes.io/part-of=bookstore >/dev/null 2>&1 && MADE=1
fi
OUT="$(kubectl -n "$PNS" run "smoke-np-$(date +%s)" --rm -i --restart=Never --image=busybox:1.37 \
         --command -- sh -c \
         "nslookup web.$NS.svc.cluster.local >/dev/null 2>&1 && echo DNS-OK; wget -q -T 3 -O /dev/null http://${WEB_IP}:8080/healthz && echo LOT-QUA || echo BI-CHAN" 2>/dev/null)"
[ "$MADE" -eq 1 ] && kubectl delete namespace "$PNS" --ignore-not-found --wait=true --timeout=120s >/dev/null 2>&1
{ echo "$OUT" | grep -q 'DNS-OK' && echo "$OUT" | grep -q 'BI-CHAN'; } \
  && pass 'Pod ngoai Traefik co mang nhung bi ingress cua web chan toi web:8080' \
  || fail "Pod ngoai Traefik toi web:8080: $(echo $OUT)"
# web -> api:3001. Đối chứng: cùng Pod web tới cùng IP Pod api ở cổng 3000 được (mục 7 dùng tên Service,
# ở đây dùng IP Pod); cổng 3001 đang nghe (readinessProbe của api đi qua cổng admin và Pod Ready).
API_POD="$(k get pod -l "$SEL,app.kubernetes.io/component=api" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
API_IP="$(jp "pod/$API_POD" '{.status.podIP}')"
RP="$(jp "pod/$API_POD" '{.spec.containers[?(@.name=="postgrest")].readinessProbe.httpGet.port}')"
RD="$(jp "pod/$API_POD" '{.status.conditions[?(@.type=="Ready")].status}')"
if [ -z "$API_IP" ]; then fail 'khong tim thay Pod api'
elif ! get "http://${API_IP}:3000/books" | grep -q '"title"'; then
  fail 'doi chung: web khong toi duoc api:3000 — khong ket luan duoc ve cong 3001'
elif [ "$RD" != True ] || { [ "$RP" != admin ] && [ "$RP" != 3001 ]; }; then
  fail "doi chung: chua chac cong admin 3001 dang nghe (probe=$RP, Ready=$RD)"
else
  OUT="$(k exec "$WEB" -c status-writer -- timeout 10 wget -q -T 5 -O /dev/null "http://${API_IP}:3001/live" 2>&1)"; RC=$?
  if [ "$RC" -eq 0 ]; then fail 'web toi duoc cong admin 3001 cua api - ma tran rong hon muc 3.5'
  elif echo "$OUT" | grep -Eq 'timed out|Connection refused|exit code 143'; then pass 'web -> api:3001 bi chan, chi cong 3000 duoc mo'
  else fail "web toi api:3001 loi khac chan mang — khong ket luan duoc: $(echo $OUT)"; fi
fi
# node-agent không có luật egress nào ngoài DNS: gọi api phải hết giờ hoặc bị từ chối (6b). Đối chứng:
# chính agent phân giải được tên api (luật DNS chung cho mọi Pod).
AG="$(k get pod -l "$SEL,app.kubernetes.io/component=agent" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
if [ -z "$AG" ]; then
  fail 'khong co Pod node-agent de thu chan mac dinh'
elif ! k exec "$AG" -c agent -- nslookup "api.$NS.svc.cluster.local" >/dev/null 2>&1; then
  fail 'doi chung: node-agent khong phan giai duoc api — khong ket luan duoc'
else
  OUT="$(k exec "$AG" -c agent -- timeout 10 wget -q -T 5 -O /dev/null http://api:3000/books 2>&1)"; RC=$?
  if [ "$RC" -eq 0 ]; then fail 'node-agent goi duoc api — chan mac dinh khong co hieu luc'
  elif echo "$OUT" | grep -Eq 'timed out|Connection refused|exit code 143'; then pass 'chan mac dinh: node-agent khong goi duoc api'
  else fail "node-agent toi api loi khac chan mang — khong ket luan duoc: $(echo $OUT)"; fi
fi
if k exec db-0 -c postgres -- getent hosts web >/dev/null 2>&1; then
  if k exec db-0 -c postgres -- timeout 5 bash -c 'exec 3<>/dev/tcp/web/80' >/dev/null 2>&1; then
    fail 'db-0 mo duoc ket noi toi web:80 - chieu di chua bi chan mac dinh'
  else
    pass 'db-0 -> web:80 bi chan, DNS van phan giai'
  fi
else
  fail 'db-0 khong phan giai duoc ten web - DNS bi chan?'
fi

# 9. Đường vào duy nhất: HTTPS qua Traefik bằng certificate của dự án (5b)
ING='{.spec.rules[0].host}|{.spec.ingressClassName}|{.spec.tls[0].hosts[0]}|{.spec.tls[0].secretName}|{.spec.rules[0].http.paths[0].backend.service.name}'
[ "$(jp ingress/bookstore "$ING")" = "$HOST|traefik|$HOST|bookstore-tls|web" ] \
  && pass "Ingress bookstore: $HOST, class traefik, TLS bookstore-tls, ve web" || fail 'Ingress bookstore sai host, class, TLS hoac backend'
NP="$(kubectl -n traefik get service traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}' 2>/dev/null)"
if [ -s secrets/tls.crt ] && [ -n "$NP" ]; then
  openssl x509 -in secrets/tls.crt -noout -checkend 0 >/dev/null && pass 'certificate trong secrets/ con han' \
    || fail 'certificate trong secrets/ da het han (TD-11)'
  for W in lab-k8s-worker1 lab-k8s-worker2; do
    IP="$(kubectl get node "$W" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
    C="curl -fsS --max-time 10 --cacert secrets/tls.crt --resolve $HOST:$NP:$IP"
    $C "https://$HOST:$NP/" 2>/dev/null | grep -q 'Bookstore' \
      && pass "HTTPS $HOST qua $W:$NP tra trang chu" || fail "HTTPS qua $W khong tra trang chu"
    T="$($C "https://$HOST:$NP/api/books" 2>/dev/null | grep -o '"title"' | wc -l)"
    [ "$T" -ge 8 ] && pass "HTTPS /api/books qua $W tra $T cuon" || fail "HTTPS /api/books qua $W tra $T cuon"
  done
else
  fail 'thieu secrets/tls.crt hoac NodePort websecure cua Traefik'
fi

# 10. Backup trên PVC backup-store (4b, 6a)
[ "$(jp cronjob/db-backup '{.spec.concurrencyPolicy}')" = 'Forbid' ] && pass 'db-backup Forbid' || fail 'db-backup khong Forbid'
SDS="$(jp cronjob/db-backup '{.spec.startingDeadlineSeconds}')"
[ "${SDS:-0}" -ge 10 ] && pass "db-backup startingDeadlineSeconds=$SDS" || fail 'db-backup thieu startingDeadlineSeconds >= 10'
[ "$(jp cronjob/db-backup '{.spec.suspend}')" != 'true' ] && pass 'db-backup khong bi tam ngung' || fail 'db-backup dang suspend'
jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.volumes[*].persistentVolumeClaim.claimName}' | grep -qw 'backup-store' \
  && pass 'db-backup ghi vao PVC backup-store' || fail 'db-backup khong dung PVC backup-store'
[ "$(jp pvc/backup-store '{.metadata.labels.app\.kubernetes\.io/component}|{.spec.storageClassName}')" = 'backup|bookstore-retain' ] \
  && pass 'PVC backup-store component=backup, class bookstore-retain' || fail 'PVC backup-store sai'
J="smoke-backup-$(date +%s)"; F=''
if k create job "$J" --from=cronjob/db-backup >/dev/null 2>&1 \
   && k wait --for=condition=Complete "job/$J" --timeout=300s >/dev/null 2>&1; then
  F="$(k logs "job/$J" 2>/dev/null | awk '/^db-backup: OK/ {print $3; exit}')"
fi
[ -n "$F" ] && pass "chay thu db-backup: $F" || fail 'chay thu db-backup that bai'
JN="$(k get pod -l "batch.kubernetes.io/job-name=$J" -o jsonpath='{.items[0].spec.nodeName}' 2>/dev/null)"
PVB="$(jp pvc/backup-store '{.spec.volumeName}')"
PVBJ="$(kubectl get pv "$PVB" -o jsonpath='{.spec.persistentVolumeReclaimPolicy}|{.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}' 2>/dev/null)"
{ [ -n "$JN" ] && [ "$PVBJ" = "Retain|$JN" ]; } \
  && pass "backup-store: PV Retain, cung node voi Pod backup ($JN)" || fail "backup-store sai ($PVBJ, Pod backup o ${JN:-?})"
k delete job "$J" --ignore-not-found --wait=true >/dev/null 2>&1
if [ -n "$F" ]; then
  RD="smoke-doc-backup-$(date +%s)"
  k apply -f - >/dev/null 2>&1 <<POD
apiVersion: v1
kind: Pod
metadata:
  name: $RD
spec:
  restartPolicy: Never
  containers:
  - name: doc
    image: busybox:1.37
    command: ["sh", "-c", "ls -1 /backup; echo ---; find /backup -maxdepth 1 -name '*.tmp' -mmin +60"]
    volumeMounts:
    - {name: b, mountPath: /backup, readOnly: true}
  volumes:
  - name: b
    persistentVolumeClaim: {claimName: backup-store, readOnly: true}
POD
  # Hợp đồng ghi file backup (6a): đọc thẳng PVC, không tin log — tối đa 10 bản, không còn file tạm
  # cũ hơn một giờ. Đọc PVC hỏng thì FAIL, không phải "0 bản".
  if k wait --for=jsonpath='{.status.phase}'=Succeeded "pod/$RD" --timeout=120s >/dev/null 2>&1; then
    OUT="$(k logs "$RD" 2>/dev/null)"
    echo "$OUT" | sed '/^---$/,$d' | grep -qx "$(basename "$F")" \
      && pass 'ban dump van tren backup-store sau khi Job bi xoa' || fail 'khong thay ban dump tren backup-store'
    NB="$(echo "$OUT" | sed '/^---$/,$d' | grep -c '^bookstore-.*\.sql$')"
    NT="$(echo "$OUT" | sed '1,/^---$/d' | grep -c .)"
    { [ "$NB" -le 10 ] && [ "$NT" -eq 0 ]; } && pass "backup-store giu $NB ban (toi da 10), khong file tam cu" \
      || fail "backup-store sai hop dong 6a ($NB ban, $NT file tam cu)"
  else
    fail 'khong doc duoc noi dung backup-store'
  fi
  k delete pod "$RD" --ignore-not-found --wait=false >/dev/null 2>&1
fi

# 11. Chiến lược sao lưu (6b). Diễn tập đã dọn sạch được kiểm ở bước 3 của B8, TRƯỚC teardown — sau
#     teardown hai object diễn tập luôn vắng nên kiểm ở đây không bao giờ fail.
if kubectl api-resources --api-group=snapshot.storage.k8s.io -o name 2>/dev/null | grep -q '^volumesnapshots\.'; then
  fail 'cluster da co API VolumeSnapshot — xem lai ADR sao luu logic (Lab 6b)'
else
  pass 'cluster chua co API VolumeSnapshot — ADR sao luu logic con hieu luc'
fi

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

2. Cập nhật `docs/tech-debt.md`: `TD-10` ở trạng thái *chấp nhận có điều kiện* (đã làm ở B5) kèm
   bằng chứng `~/lab-evidence/6b/b5-ban-dump.txt` và `~/lab-evidence/6b/b6-doi-chieu.txt`. Thêm vào
   `README.md` của repo một dòng trỏ tới `docs/runbook-backup.md`.

3. Kiểm diễn tập đã dọn sạch — **trước** teardown, vì teardown cũng xóa hai object này — rồi dựng lại
   sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
# --ignore-not-found: "không có" là output rỗng với mã 0; lỗi API là mã khác 0, không bị hiểu thành "không có".
if R="$(kubectl get namespace bookstore-restore-test --ignore-not-found -o name)"; then
  [ -z "$R" ] && echo 'PASS: khong con namespace dien tap' || echo 'FAIL: namespace dien tap bookstore-restore-test con sot'
else echo 'STOP: khong hoi duoc API'; fi
if R="$(kubectl -n bookstore get pod backup-reader --ignore-not-found -o name)"; then
  [ -z "$R" ] && echo 'PASS: khong con Pod doc ban dump' || echo 'FAIL: Pod doc ban dump backup-reader con sot'
else echo 'STOP: khong hoi duoc API'; fi
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/6b/b8-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/6b/b8-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/6b/b8-smoke.txt
```

4. Chạy runbook mục 1 một lần nữa trên hệ thống vừa dựng lại: bản dump mới nhất ra
   `~/bookstore-backups/` và ra máy host, checksum khớp. Đây là thói quen của mọi sprint từ nay —
   bundle repo và bản dump luôn đi cùng nhau.

5. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-6b: sprint 9 — chien luoc sao luu, runbook va dien tap khoi phuc'
git tag lab-6b
git bundle create ~/bookstore-lab-6b.bundle --all && git bundle verify ~/bookstore-lab-6b.bundle
```

Chép `~/bookstore-lab-6b.bundle` về máy host bằng `scp` trước khi đi tiếp.

6. Dọn object thử nghiệm và trả cluster:

```bash
kubectl delete namespace bookstore-thu bookstore-restore-test --ignore-not-found --wait=true
kubectl delete volumeattributesclass bookstore-thu-silver --ignore-not-found
kubectl -n bookstore delete pod backup-reader --ignore-not-found --wait=true
rm -rf ~/lab-work/6b
cd ~/bookstore && ./scripts/teardown.sh
```

7. Gate cuối:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
cd ~/bookstore
. ~/lab-evidence/6b/b0-baseline.env
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-6b' && echo 'PASS: HEAD mang tag lab-6b'
test -s ~/bookstore-lab-6b.bundle && echo 'PASS: co bundle'
git diff --quiet lab-6a HEAD -- k8s scripts && echo 'PASS: sprint 9 khong doi manifest va script trien khai'
BAD=0
for F in 'docs/adr/*-sao-luu-logic.md' docs/runbook-backup.md docs/postmortem/6b-dien-tap-khoi-phuc.md; do
  ls $F >/dev/null 2>&1 || { echo "FAIL: thieu $F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du ADR, runbook, postmortem cua sprint 9'
grep 'TD-10' docs/tech-debt.md | grep -q 'chấp nhận có điều kiện' && echo 'PASS: tech-debt.md ghi TD-10 chap nhan co dieu kien'
git grep -l 'PostgreSQL database dump' >/dev/null 2>&1 && echo 'FAIL: repo chua ban dump' || echo 'PASS: repo khong chua ban dump'
ls ~/bookstore-backups/*.sql >/dev/null 2>&1 && echo 'PASS: co ban dump ngoai cluster tren master'

{ [ "$(sl kubectl get namespace,priorityclass,storageclass -l app.kubernetes.io/part-of=bookstore --no-headers)" -eq 0 ] && \
  [ "$(sl kubectl get all -A -l app.kubernetes.io/part-of=bookstore --no-headers)" -eq 0 ]; } \
  && echo 'PASS: khong con object nao cua du an'
{ [ "$(sl kubectl get pv --no-headers)" -eq "$PV_BEFORE" ] && \
  [ "$(sl kubectl get pvc -A --no-headers)" -eq "$PVC_BEFORE" ]; } \
  && echo "PASS: so PV, PVC ve dung muc dau lab ($PV_BEFORE, $PVC_BEFORE)"
{ [ "$(kubectl get storageclass -o name | sort | tr '\n' ' ')" = "$SC_BEFORE" ] && \
  [ "$(kubectl get storageclass --no-headers | awk '/\(default\)/{print $1}')" = "$SC_DEFAULT" ]; } \
  && echo "PASS: StorageClass ve dung 03-storage-ready ($SC_DEFAULT mac dinh)"
[ "$(kubectl get namespace -o name | sort | tr '\n' ' ')" = "$NS_BEFORE" ] \
  && echo 'PASS: danh sach namespace khong doi — namespace cua provisioner con nguyen'
{ [ "$(sl kubectl get csidriver --no-headers)" -eq "$CSIDRIVER_BEFORE" ] && \
  [ "$(kubectl get crd -o name 2>/dev/null | grep -c '\.snapshot\.storage\.k8s\.io$')" -eq "$SNAPCRD_BEFORE" ] && \
  [ "$(sl kubectl get volumeattributesclass --no-headers)" -eq "$VAC_BEFORE" ]; } \
  && echo 'PASS: lab khong cai them CSI driver, CRD snapshot hay VolumeAttributesClass'
# Liệt kê /opt/local-path-provisioner trên worker. Thư mục gốc chưa từng có thì in rỗng, thoát 0;
# lỗi ssh, sudo hay find thì thoát khác 0 — "không đọc được" không bao giờ được hiểu thành "rỗng".
lp_ls() {
  ssh -o BatchMode=yes "$1" \
    'sudo -n test ! -e /opt/local-path-provisioner || sudo -n find /opt/local-path-provisioner -mindepth 1 -maxdepth 1'
}
for N in lab-k8s-worker1 lab-k8s-worker2; do
  if ! L="$(lp_ls "$N")"; then echo "STOP: khong kiem duoc $N (ssh, sudo hoac find loi) — khong chup snapshot"
  elif [ -z "$L" ]; then echo "PASS: $N khong con thu muc du lieu nao"
  else echo "FAIL: $N con $(printf '%s\n' "$L" | grep -c .) muc trong /opt/local-path-provisioner"; fi
done
test ! -e ~/lab-work/6b && echo 'PASS: da don lab-work'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** `smoke exit=0` và file smoke ở bước 3 in toàn `PASS:`; khối PowerShell ở máy host in
`PASS: ban dump tren may host khop checksum` cho bản dump của bước 4; mười sáu dòng `PASS:` đầu của
gate cuối xuất hiện, không dòng `FAIL:` nào; dòng `PASS: readyz ok`; ba node `Ready`; lệnh field
selector trả `No resources found`; CoreDNS đủ replica; namespace `default` không có Pod. Thư mục
`~/lab-evidence/6b/` và `~/bookstore-backups/` **giữ lại**. Cluster trở về `03-storage-ready`;
**không tạo snapshot mới**.

---

## 3. Checkpoint 6b

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Bốn điều kiện để chụp được một snapshot thật là gì? Cluster của bạn thiếu điều kiện nào, bạn đọc
      điều đó từ lệnh nào? Nợ #5 trả chưa — nếu chưa, cần chính xác những gì và trả ở đâu?
- [ ] `VolumeSnapshot`, `VolumeSnapshotContent`, `VolumeSnapshotClass` ứng với ba object nào của
      Lab 6a, cái nào thuộc namespace, cái nào thuộc cluster, ai tạo cái nào? Vì sao `kubectl apply`
      một VolumeSnapshotClass thất bại trên cluster này, và ai chịu trách nhiệm cài thứ còn thiếu?
- [ ] `thu-clone` `Bound`, Pod chạy, không một lỗi nào — nhưng volume rỗng. Vì sao? Đề xuất "nhân bản
      `data-db-0` mỗi đêm" sai ở hai tầng nào, kể cả trên cluster có CSI?
- [ ] `dataSource` và `dataSourceRef` khác nhau ở hai điểm nào? Bạn chứng minh điểm thứ nhất bằng
      hai phép thử nào, và vì sao hai phép thử phải dùng **cùng một giá trị**? Chỉ khai một trường thì
      trường kia mang gì, và sau khi tạo có sửa được không?
- [ ] Ba điều kiện để scheduler xét dung lượng lưu trữ là gì, `bookstore-retain` đạt mấy? PVC 500Gi
      được cấp trên đĩa nhỏ hơn nhiều nói gì về `data-db-0` khi đĩa worker2 đầy? Đổi
      `bookstore-retain` sang `Immediate` thì `db-0` ra sao?
- [ ] Ai công bố số volume tối đa gắn được vào một node, đọc ở object nào? Volume health: Event báo
      trên PVC hay trên Pod, metric tên gì, giá trị `1` nghĩa là gì — và vì sao `data-db-0` không có
      series đó?
- [ ] VolumeAttributesClass khác StorageClass ở điểm căn bản nào? Cần đổi tham số của một class đang
      dùng thì làm gì?
- [ ] Vì sao Bookstore sao lưu bằng `pg_dump` — và vì sao vẫn giữ `pg_dump` nếu ngày mai cluster có
      snapshot? `pg_dump` nhất quán ra sao khi DB đang nhận ghi, so với chép thư mục dữ liệu trên node?
- [ ] Bản dump trên `backup-store` chưa phải bản sao lưu vì sao? Kể quy trình chép ra ngoài; vì sao Pod
      đọc phải nằm trên đúng node và bạn không cần ghim nó; API server chết thì lấy bản dump bằng đường
      nào? RPO thật khi mất cả cluster tính từ mốc nào?
- [ ] Ở B6, `db-restore` Ready khi chưa có schema `api`. Ánh xạ diễn tập sang mô hình populator của bài
      102; khoảng hở đó là gì và bạn bịt bằng gì? Vì sao nạp trong một giao dịch, không qua
      `/docker-entrypoint-initdb.d`?
- [ ] Bản dump có mang role và mật khẩu không? Vì sao DB tạm với bí mật hoàn toàn mới vẫn phục vụ được
      `authenticator`? Cảnh báo về mật khẩu cũ ở mục 5.4 của file dự án áp vào trường hợp nào?
- [ ] Diễn tập của bạn mất bao lâu ở từng chặng, chặng nào phình theo dữ liệu? (⭐) Ở B7, đơn hàng nào
      mất và RPO quan sát được là bao nhiêu?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- Nợ lab #5 (snapshot và nhân bản volume): **vẫn mở** — cluster không có CSI driver hỗ trợ snapshot;
  trả bằng cách chạy lại B1 và nhánh A của B2 khi cluster có đủ bốn điều kiện. Nợ lab #1 (HPA ở Lab
  11b; VPA chưa có lab), #6, #7, #8 (Phần II của lộ trình) giữ nguyên — xem
  [sổ nợ lab](README.md#5-sổ-nợ-lab).
- Nợ kỹ thuật: `TD-10` **chấp nhận có điều kiện** (runbook mục 1, chạy cuối mỗi sprint và trước mọi
  thao tác phá; checksum khớp ở máy host; khôi phục đã diễn tập); `TD-06` (`db` khởi động bằng root —
  Lab 9b), `TD-07` (quyền debug — Lab 9a) vẫn mở. Rủi ro đĩa worker2 đầy không được báo trước ghi ở
  phần Hệ quả của ADR, chờ phần quan sát của Lab 11a.

### Bài giải thích cuối cùng

Trong vài phút, kể lại **ngày xấu nhất của Bookstore**: `lab-k8s-worker2` chết hẳn, không bật lại được.
Bắt đầu từ những gì mất — `data-db-0`, có thể cả `backup-store`, tùy node mà B4 đo được — và vì sao PV
`Retain` không cứu được trong trường hợp này. Rồi kể thứ còn lại: bản dump trên máy host, checksum của
nó, thời điểm nó được chép ra — và từ đó suy ra bao nhiêu đơn hàng mất. Kể từng bước dựng lại: DB mới
từ repo với bí mật mới, role đến từ đâu, bản dump được nạp thế nào để không bao giờ ra nửa vời, đối
chiếu bằng gì trước khi mở lại cho khách, mất bao lâu theo số đo của B6. Ở mỗi bước nói rõ vì sao
snapshot hay nhân bản volume **không** có mặt trong câu chuyện trên cluster này. Kết thúc bằng việc chỉ
ra Lab 8a sẽ đặt câu chuyện này lên một cluster dựng lại từ đầu, và Lab 14 sẽ biến việc sao lưu thành
API khai báo của chính dự án.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: không có `local-path` mặc định, hoặc đã có PV/PVC | Cluster không ở `03-storage-ready` | Restore **cả ba** VM về `03-storage-ready`; không tự tạo StorageClass ở lab này |
| B0: `/opt/local-path-provisioner` trên worker còn thư mục | `teardown.sh` của Lab 6a chưa dọn PV `Retain`, hoặc snapshot chụp trước teardown | Chạy `./scripts/teardown.sh` của tag `lab-6a`; còn sót thì dọn theo cách của Lab 6a rồi chạy lại B0 |
| B0: smoke của tag `lab-6a` fail ở `data-db-0` hay `backup-store` | Repo sprint 8 chưa đủ | Sửa repo ở tag `lab-6a`, gắn lại tag; không sửa tay trên cluster |
| B1: `kubectl get volumesnapshot` báo `the server doesn't have a resource type` | Không phải lỗi — đúng kết quả nhánh B | Ghi lại làm bằng chứng; **không** cài CRD |
| B1: dry-run VolumeSnapshotClass báo `no matches for kind` | Đây là gate PASS của bước 3 | Không làm gì thêm |
| B1: tạo VolumeAttributesClass bị từ chối vì `parameters` | API đòi ít nhất một tham số | Thêm một cặp key/value |
| B1: patch `parameters` **thành công** | Hành vi khác mô tả của bài 97 ở phiên bản đang chạy | Ghi nguyên văn vào ADR; không kết luận theo bài |
| B2, B3: Pod thử nghiệm mãi `Pending`, PVC `Pending` | Dùng `nodeName` thay `nodeSelector`, nên scheduler không ghi node cho PVC `WaitForFirstConsumer` | Đổi sang `nodeSelector` |
| B2: dòng đánh dấu **có** trong clone dù `IS_CSI=0` | B1 đọc thiếu, hoặc provisioner có hỗ trợ sao chép riêng | Chạy lại B1; ghi cả hai kết quả và đọc lại kết luận |
| B2: `thu-clone` `Pending` kèm Event nhắc `dataSource` | Provisioner từ chối nguồn dữ liệu thay vì bỏ qua | Ghi Event vào ADR; kết luận vẫn là "không nhân bản được trên cluster này" |
| B2: script báo `sed khong doi duoc dataSource` | `thu-ds-sai.yaml` thụt lề khác hai dấu cách | Viết lại manifest với thụt lề hai dấu cách |
| B2, B3, B6: namespace kẹt `Terminating` | Pod còn dùng PVC, finalizer bảo vệ PVC giữ lại | Chờ Pod tắt; **không** gỡ finalizer bằng tay |
| B3: PVC `thu-500gi` `Pending` | Provisioner có kiểm dung lượng | Ghi Event; kết luận "có thành phần kiểm dung lượng", giảm kích thước rồi chạy lại |
| B3: `DF_KB` rỗng hoặc không phải số | Tên filesystem dài làm `df` của busybox xuống dòng | `kubectl -n bookstore-thu exec thu-doc-500gi -- df -k`, đọc dòng có `/data` ở cột cuối và lấy cột thứ hai |
| B3: `get --raw .../proxy/metrics` báo `403` | Không dùng kubeconfig quản trị | Chạy trên `lab-k8s-master` bằng kubeconfig của Lab 00; không tạo RBAC |
| B4, B5: Job từ `db-backup` `Failed` | DNS tới `db-0.db`, NetworkPolicy của Lab 5b, hoặc sai key Secret | `kubectl -n bookstore logs job/<tên>`; sửa repo, không sửa tay |
| B5: `backup-reader` `Pending`, sự kiện xung đột node affinity của volume | Pod bị ghim vào node khác node giữ PV | Bỏ ghim; để scheduler đọc `nodeAffinity` của PV |
| B5: `backup-reader` `Pending` trong lúc một Job sao lưu đang chạy | `backup-store` dùng chế độ chỉ một Pod được mount | Chờ Job xong; ghi điều đó vào runbook |
| B5: file bản dump bắt đầu bằng `PGDMP` | `db-backup` của bạn dùng định dạng custom | Runbook và B6 dùng `pg_restore` thay `psql`; các phép kiểm chuỗi trong script cần đổi theo |
| B5: checksum lệch | Chép dở, hoặc chép nhầm file | Chép lại đúng tên file của dòng `db-backup: OK` |
| B6: `db-restore` `CrashLoopBackOff`, log `01-roles.sh: thieu AUTHENTICATOR_PASSWORD` | Pod tạo trước Secret, hoặc sai tên key | Xóa namespace tạm, làm lại đúng thứ tự namespace → Secret → DB |
| B6: `db-restore` Ready nhưng thiếu role | Script khởi tạo lỗi giữa chừng; lần khởi động sau thấy thư mục dữ liệu đã khởi tạo nên bỏ qua script | Xóa Pod **và** PVC `restore-data`, làm lại — đúng cái bẫy Docker Hub cảnh báo |
| B6: `psql` dừng với `role "web_anon" does not exist`, exit 3 | Nạp bản dump trước khi có role | Đúng điều PostgreSQL docs cảnh báo; giao dịch đã rollback — tạo role rồi nạp lại |
| B6: `psql` báo `schema "api" already exists` | DB tạm dùng nhầm `db-init` của production, hoặc nạp lần hai | DB tạm chỉ tạo role; nạp lại vào DB tạm mới |
| B6: `pg_isready` từ ngoài trả `accepting connections` | NetworkPolicy chưa áp, hoặc cluster không ở CNI có thực thi policy | `kubectl -n bookstore-restore-test get networkpolicy`; kiểm cluster đang ở chuỗi `02-net-ready` trở đi |
| B6: `authenticator` vào được bằng mật khẩu bất kỳ | Nối tới `127.0.0.1` — kết nối loopback là `trust` | Nối tới IP của Pod |
| B6: checksum `api.orders` lệch | Có ghi vào `db-0` sau bản dump, hoặc dùng bản dump cũ | Làm lại runbook mục 1 ngay trước diễn tập |
| B6: PV của DB tạm còn sau khi xóa namespace | PVC dùng `bookstore-retain` | Xóa PV và thư mục theo cách của Lab 6a; lần sau dùng class mặc định |
| B7: web trả lỗi sau khi khôi phục | PostgREST còn giữ schema cache cũ | `NOTIFY pgrst, 'reload schema'` |
| B7: lệnh khôi phục treo | `pg_dump` của một Job sao lưu đang giữ khóa đọc trên bảng | Chờ Job xong; xem `pg_stat_activity` |
| B8: `teardown.sh` chờ lâu | `db-0` dùng hết ân hạn khi tắt | Chờ; script có giới hạn chờ riêng |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Volume Attributes Classes](https://v1-35.docs.kubernetes.io/docs/concepts/storage/volume-attributes-classes/)
- [Kubernetes v1.35 — Volume Snapshots](https://v1-35.docs.kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [Kubernetes v1.35 — Volume Snapshot Classes](https://v1-35.docs.kubernetes.io/docs/concepts/storage/volume-snapshot-classes/)
- [Kubernetes v1.35 — CSI Volume Cloning](https://v1-35.docs.kubernetes.io/docs/concepts/storage/volume-pvc-datasource/)
- [Kubernetes v1.35 — Volume Populators and Data Sources](https://v1-35.docs.kubernetes.io/docs/concepts/storage/volume-populators-and-data-sources/)
- [Kubernetes v1.35 — Storage Capacity](https://v1-35.docs.kubernetes.io/docs/concepts/storage/storage-capacity/)
- [Kubernetes v1.35 — Node-specific Volume Limits](https://v1-35.docs.kubernetes.io/docs/concepts/storage/storage-limits/)
- [Kubernetes v1.35 — Volume Health Monitoring](https://v1-35.docs.kubernetes.io/docs/concepts/storage/volume-health-monitoring/)
- [Kubernetes v1.35 — Persistent Volumes](https://v1-35.docs.kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes v1.35 — Feature Gates](https://v1-35.docs.kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
- [Kubernetes — `kubectl cp`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_cp/)
- [Kubernetes CSI — danh sách driver và tính năng từng driver hỗ trợ](https://kubernetes-csi.github.io/docs/drivers.html)
- [PostgreSQL 17 — SQL Dump (Restoring the Dump, Using pg_dumpall)](https://www.postgresql.org/docs/17/backup-dump.html)
- [PostgreSQL 17 — pg_dump](https://www.postgresql.org/docs/17/app-pgdump.html)
- [PostgreSQL 17 — psql](https://www.postgresql.org/docs/17/app-psql.html)
- [Docker Hub — postgres (Initialization scripts)](https://hub.docker.com/_/postgres)
- [PostgREST v14 — Schema Cache](https://docs.postgrest.org/en/v14/references/schema_cache.html)
