# Lab 6a — PV, PVC và StorageClass

> **Điểm bắt đầu:** snapshot `02-net-ready` — mốc do
> [Lab 5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) tạo, xem [chuỗi snapshot](README.md#3-chuỗi-snapshot).
> Repo `~/bookstore` ở tag `lab-5b` (hoặc bundle `~/bookstore-lab-5b.bundle` nếu bạn vừa restore
> snapshot).
> **Điểm kết thúc:** lab **tạo mốc mới `03-storage-ready`** — cluster có thêm một dynamic
> provisioner và StorageClass mặc định, **không** có object hay dữ liệu nào của dự án. Repo
> `~/bookstore` có tag `lab-6a` và một bản bundle nằm ngoài VM.
> **Sprint dự án:** Sprint 8 — Dữ liệu bền của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 5b — NetworkPolicy, Ingress và CNI](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md).
> **Lab sau:** [Lab 6b — Snapshot và volume nâng cao](LAB-6B-SNAPSHOT-VA-VOLUME-NANG-CAO.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng phần cốt lõi của
[Giai đoạn 6 — Lưu trữ](../00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ): tám bài khái niệm 90, 91,
92, 96, 98, 93, 94, 95 và bốn bài thực hành 277, 280, 343, 344. Phần snapshot, nhân bản và
VolumeAttributesClass thuộc Lab 6b. Mọi ticket dựa vào hợp đồng kỹ thuật ở
[mục 4 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật) và bố cục repo ở
[mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint). Kế hoạch gốc của sprint nằm ở
[mục 7.2](DU-AN-BOOKSTORE.md#sprint-8--lab-6a-dữ-liệu-bền).

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). Thành phần duy nhất lab
cài thêm là dynamic provisioner, version của nó đã khóa ở dòng `local-path-provisioner` của
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). Lab không chép lại con số
phiên bản nào trong phần văn bản.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó, cộng hai lệnh riêng của lab này.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi hệ thống Bookstore đạt trạng thái sau sprint 8 — **dữ liệu của `db` và bản
backup sống sót qua việc xóa Pod, xóa workload** (yêu cầu `R8` của dự án) — **và** bạn chứng minh,
giải thích được:

- Vì sao trước sprint này dữ liệu của `db` sống qua lần khởi động lại container nhưng mất khi Pod
  bị thay, dù Pod mới vẫn tên `db-0` — và thư mục đó thật sự nằm ở đâu trên node.
- Cấp phát động trên cluster này: ai là quản trị viên, ai là người dùng; hai điều kiện để có class
  mặc định; PVC bỏ trống `storageClassName` khác PVC khai `""` ở đâu.
- StorageClass của dự án quyết định những gì cho mọi PV nó sinh ra — `reclaimPolicy`,
  `volumeBindingMode`, mở rộng dung lượng — và vì sao lưu trữ gắn với node bắt buộc
  `WaitForFirstConsumer`.
- `volumeClaimTemplates` sinh PVC `data-db-0` thế nào; PV của nó mang những trường nào bạn chưa
  từng viết; dữ liệu nằm ở thư mục nào trên `lab-k8s-worker2` — **đây là phần trả nợ lab #2**.
- Vì sao dữ liệu bền buộc mật khẩu DB vào chính dữ liệu, và chính sách `secrets/db.env` của dự án.
- `ReadWriteOnce` là ràng buộc **theo node**: Pod backup phải chạy ở node có PV, và chọn node đó là
  một quyết định thiết kế.
- Ba lần "xóa nhầm" — Pod, StatefulSet, PVC: cái nào hệ thống tự chịu được, cái nào cần người; khôi
  phục dữ liệu từ PV `Released` bằng một PV tĩnh trỏ lại thư mục cũ.
- Projected volume gom nhiều nguồn vào một thư mục, hai khác biệt cú pháp, và volume
  `kube-api-access-*` mà mọi Pod của bạn vẫn mang.
- Lưu trữ tạm thời cục bộ là tài nguyên của node: `sizeLimit`, `limits.ephemeral-storage`, ai trục
  xuất Pod, và vì sao `emptyDir` dạng `tmpfs` tính vào bộ nhớ.
- Vì sao `hostPath` là lối thoát nguy hiểm — chứng minh bằng chính file dữ liệu của `db`.
- Teardown phải dọn những gì để mốc `03-storage-ready` chỉ chứa hạ tầng.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 6a | Ticket kiểm chứng |
| --- | --- |
| [90 — Lưu trữ](../90-storage-vi.md) | B1 — nhánh tạm thời: `emptyDir` đang giữ dữ liệu `db`; B2 — nhánh bền vững chưa có gì trên cluster (không StorageClass, không PV, không CSI driver) trước khi cài provisioner |
| [91 — Các Volume](../91-volumes-vi.md) | B1 — ranh giới vòng đời `emptyDir`; B3 — `persistentVolumeClaim`, câu hỏi `subPath` hay biến `PGDATA`; B6 — `configMap` và `downwardAPI` đi qua projected; B7 — `emptyDir.sizeLimit`, `medium: Memory`; B8 — rủi ro `hostPath`, mount chỉ đọc |
| [92 — Volume bền vững](../92-persistent-volumes-vi.md) | B2 — gán class mặc định hồi tố, `""` so với bỏ trống; B3 — ràng buộc PVC↔PV, `nodeAffinity` do provisioner ghi, mở rộng bị từ chối; B4 — `ReadWriteOnce` theo node; B5 — bảo vệ PVC đang dùng, `Retain`, pha `Released`, thu hồi thủ công, dành riêng PV bằng `claimRef` |
| [96 — Lớp lưu trữ](../96-storage-classes-vi.md) | B2 — `provisioner`, class mặc định; B3 — `reclaimPolicy`, `volumeBindingMode` (kèm thử `Immediate`), `allowVolumeExpansion` |
| [98 — Cấp phát Volume động](../98-dynamic-provisioning-vi.md) | B2 — ranh giới admin/người dùng, hai điều kiện của hành vi mặc định; B3 — tạo PVC là đủ để PV xuất hiện |
| [93 — Volume dạng projected](../93-projected-volumes-vi.md) | B6 — `status-writer` đọc label và cấu hình từ một mount; `name` thay `secretName`, `defaultMode` chỉ ở cấp projected; `serviceAccountToken` trong `kube-api-access-*` |
| [94 — Volume tạm thời](../94-ephemeral-volumes-vi.md) | B9 ⭐ — volume tạm thời tổng quát sinh PVC `<Pod>-<volume>`, owner reference, xung đột tên; B4 — câu hỏi vì sao nó không phải chỗ cho bản backup |
| [95 — Lưu trữ tạm thời cục bộ](../95-ephemeral-storage-vi.md) | B7 — `ephemeral-storage` trong Node, `requests` khi lập lịch, `sizeLimit` và `limits` dẫn tới trục xuất, `tmpfs` tính vào bộ nhớ, đơn vị byte |
| [277 — Pod dùng projected Volume](../277-configure-projected-volume-vi.md) | B6 — một mục `volumes`, một `volumeMounts`, nhiều nguồn |
| [280 — Pod dùng Volume để lưu trữ](../280-configure-volume-storage-vi.md) | B1 — phép thử "giết tiến trình rồi đọc lại" trên chính `db`: dữ liệu sống qua container restart, chết theo Pod |
| [343 — Ứng dụng có trạng thái được nhân bản](../343-run-replicated-stateful-application-vi.md) | B3 — `volumeClaimTemplates`, PVC theo ordinal; B5 — xóa Pod thì gặp lại đúng PVC cũ, thu về 0 bản sao PVC vẫn còn, thiếu PVC thì controller tự sinh PVC mới. Phần replication MySQL xem bảng dưới |
| [344 — Ứng dụng có trạng thái đơn thực thể](../344-run-single-instance-stateful-vi.md) | B3 — câu hỏi thiết kế: Deployment + PVC + `Recreate` so với StatefulSet cho `db`; B5 — PV tạo tay thì phải dọn tay |

Bài [65 — StatefulSets](../65-statefulset-vi.md) không thuộc nhóm này nhưng là nơi phát sinh
[nợ lab #2](README.md#5-sổ-nợ-lab). B3 trả nợ đó; B5 dùng tiếp ba mục *Hạn chế*, *Lưu trữ ổn định*,
*Lưu giữ PersistentVolumeClaim*. **Trước khi làm B3, đọc lại bài 65** ở đúng ba mục đó cộng mục
*Template cho volume claim*.

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài 343, phần replication MySQL (`init-mysql`, `clone-mysql`, `xtrabackup`, drain node) | Dự án chỉ có một PostgreSQL không nhân bản (ADR `db-statefulset` ở Lab 4b); ba MySQL kèm sidecar vượt headroom ba VM; `kubectl drain` thuộc [giai đoạn 16](../00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node). B3, B5 giữ đúng phần lưu trữ của bài |
| Bài 343, scale lên sinh PVC cho ordinal mới | `db` không được có hơn một bản sao (Lab 4b). B5 chứng minh cùng cơ chế theo chiều ngược: thiếu `data-db-0` thì controller tự sinh PVC mới |
| Bài 92, `ReadWriteOncePod`; ghi chú khuyến nghị `ReadWriteOncePod` của bài 65 | Chỉ hỗ trợ cho volume CSI; provisioner của lab không phải CSI driver. B3 hỏi điều đó có nghĩa gì với `db` |
| Bài 92, mục *Hỗ trợ raw block volume* | Provisioner chỉ cấp `volumeMode: Filesystem` |
| Bài 92, các mục snapshot, nhân bản, volume populator | Cần CSI driver hỗ trợ — **nợ lab #5**, Lab 6b |
| Bài 92 mục *Recycle*; bài 96 các mục AWS, Azure, vSphere, Ceph, Portworx | `Recycle` đã lỗi thời; cluster bare-metal không có provisioner của cloud |
| Bài 92, mục *Cập nhật node affinity* | Alpha, feature gate tắt trên baseline |
| Bài 94, mục *Volume tạm thời CSI* | Cần CSI driver viết riêng cho chế độ inline |
| Bài 95, mục *Hạn ngạch project của filesystem* | Beta, tắt mặc định; cần user namespace và filesystem bật `prjquota` |

Nợ sau sprint: **trả nợ lab #2** (B3); **trả `TD-04`** (B3) và **`TD-08`** (B4); **phát sinh
`TD-10`** — bản backup vẫn nằm trong cùng cluster với DB (B4). Nợ lab #5 không thuộc lab này. Xem
[sổ nợ lab](README.md#5-sổ-nợ-lab) và [sổ nợ kỹ thuật dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án).

### 1.2. Thời lượng

4–5 giờ, **chưa kể** thời gian tắt máy và chụp snapshot. Phần cốt lõi (B0–B8, B10) khoảng 4 giờ;
B9 là tùy chọn. B5 và B7 có các đoạn chờ controller và kubelet; thời gian chờ phụ thuộc cấu hình nên
mọi script đều chờ theo điều kiện, không theo con số cố định. Có thể chia hai phiên: phiên 1 làm
B0–B4, phiên 2 làm B5–B10. Giữa hai phiên cứ để nguyên cluster; đầu phiên 2 chạy lại gate ngắn
A5.5 và `./scripts/deploy.sh`.

---

## 2. Quy ước và an toàn

> **Cảnh báo — lab này đổi hạ tầng vĩnh viễn.** B2 cài dynamic provisioner vào namespace mới
> `local-path-storage` và để lại một StorageClass mặc định cho toàn cluster. B10 chụp mốc
> `03-storage-ready`. Trước khi chạy bất kỳ lệnh nào của phần B, **bảo đảm cả ba VM đều còn
> snapshot `02-net-ready`** — đó là điểm quay lui duy nhất.

Kiểm tra trên PowerShell của máy host; sửa đường dẫn `.vmx` nếu VM nằm chỗ khác:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$vmx = @(
  'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
  'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
  'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
)
foreach ($f in $vmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '02-net-ready') { "PASS: $f" }
  else { "FAIL: $f -> $($names -join ', ')" }
}
```

**PASS:** đúng ba dòng `PASS:`, không dòng `FAIL:` nào. Thiếu mốc trên VM nào thì dừng: quay về
Lab 5b và chụp mốc đó trước.

**Quay lui khi hỏng:** chép bundle repo mới nhất ra máy host (nếu đã có commit trong lab này), tắt
**cả ba VM**, restore **cả ba** về `02-net-ready` — không bao giờ restore riêng một VM, xem ghi chú
cuối [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường) — rồi bật lại theo
thứ tự master → worker 1 → worker 2 và bắt đầu lại từ B0.

Các quy ước còn lại:

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác. Lệnh trên worker chạy qua
  `ssh <node> 'sudo ...'` như các lab trước; B0 kiểm `sudo` không hỏi mật khẩu.
- **Fault injection chỉ trên `lab-k8s-worker2`.** `db-0` ghim ở node này; mọi diễn tập xóa dữ liệu
  (B1, B5) và mọi Pod cố ý vượt giới hạn (B7) đều nằm trên `lab-k8s-worker2`. B4 và B8 có đặt Pod
  thử nghiệm lên `lab-k8s-worker1`, nhưng chỉ để **đọc**.
- Repo dự án: `~/bookstore`. Manifest thử nghiệm: `~/lab-work/6a/`. Bằng chứng: `~/lab-evidence/6a/`.
- Object thử nghiệm đặt tên bắt đầu bằng `thu-`, nằm trong namespace `bookstore` và **không** mang
  label `app.kubernetes.io/part-of`. Ngoại lệ duy nhất là StorageClass thử nghiệm `thu-immediate`
  (cấp cluster, B3): nó **mang** label dự án để `teardown.sh` dọn được nếu bạn quên.
- Lab chỉ dùng kiến thức nhóm 6 và các nhóm trước: Pod, probe, downward API (3a), ConfigMap, Secret,
  Kustomize (3b), `requests`/`limits` (3c), Deployment (4a), StatefulSet, Job, CronJob (4b), Service
  (5a), NetworkPolicy, Ingress (5b). **Không** dùng ResourceQuota, LimitRange, PriorityClass mới,
  taint/toleration mới (7a, 7b), RBAC (9a). Khối `nodeAffinity` của PV chỉ được **đọc** hoặc **chép
  nguyên** từ PV có sẵn; cú pháp affinity học ở 7a.
- NetworkPolicy của dự án **không đổi** trong sprint này: không thành phần mới nào cần đường mạng mới.
  Pod thử nghiệm trong `bookstore` rơi vào default deny của Lab 5b — chúng không cần mạng.
- `kubectl scale`, `kubectl delete statefulset`, `kubectl delete pvc` trên object của dự án **chỉ**
  dùng trong diễn tập B5. Repo là nguồn sự thật duy nhất.
- **Không** tự `rm` gì trong `/opt/local-path-provisioner` trên node. Thư mục dữ liệu chỉ được xóa
  bởi provisioner hoặc bởi `scripts/teardown.sh`, sau khi PV tương ứng đã bị xóa.
- **Vệ sinh bí mật** như Lab 3b: không in mật khẩu ra màn hình, không ghi vào `~/lab-evidence/`, không
  xóa `secrets/db.env` khi dữ liệu DB còn trên cluster (B3 giải thích vì sao).
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

---

# Phần B — Sprint 8: Dữ liệu bền

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `02-net-ready` với tầng lưu trữ còn trống, repo ở đúng tag
`lab-5b`, và hệ thống của sprint 7 dựng lại được **chỉ từ repo**.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Hai lệnh riêng của lab này — tầng lưu trữ phải còn trống, và ghi lại ảnh chụp "trước":

```bash
mkdir -p ~/lab-work/6a ~/lab-evidence/6a
SC_N="$(kubectl get storageclass --no-headers 2>/dev/null | wc -l)"
PV_N="$(kubectl get persistentvolume --no-headers 2>/dev/null | wc -l)"
echo "storageclass=$SC_N persistentvolume=$PV_N"
{ [ "$SC_N" -eq 0 ] && [ "$PV_N" -eq 0 ]; } && echo 'PASS: tang luu tru trong, dung diem xuat phat'
{
  echo "=== $(date -Is) — tang luu tru truoc Lab 6a ==="
  kubectl get storageclass,persistentvolume 2>&1
  kubectl get csidrivers 2>&1
  kubectl get namespaces
} > ~/lab-evidence/6a/b0-truoc.txt
```

`ssh` và `sudo` không hỏi mật khẩu tới hai worker — từ lab này `scripts/teardown.sh` cần chúng:

```bash
for N in lab-k8s-worker1 lab-k8s-worker2; do
  ssh "$N" 'sudo -n true' && echo "PASS: ssh + sudo khong hoi mat khau toi $N" \
    || echo "FAIL: $N can mat khau sudo hoac khong ssh duoc"
done
```

Repo và hệ thống của sprint 7. Nếu vừa restore snapshot, repo đã mất — khôi phục từ bundle theo
[mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm) trước:

```bash
test -d ~/bookstore/.git || git clone ~/bookstore-lab-5b.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-5b' && echo 'PASS: HEAD dung tag lab-5b'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lan truoc' || echo 'PASS: cluster chua co du an'

./scripts/deploy.sh > ~/lab-evidence/6a/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/6a/b0-smoke-lab-5b.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/6a/b0-smoke-lab-5b.txt
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; dòng `PASS: tang luu tru trong…`; hai dòng `PASS: ssh +
sudo…`; ba dòng `PASS:` về repo và cluster; `deploy exit=0`, `smoke exit=0` và file smoke không có
dòng `FAIL:`.

`SC_N` khác 0 nghĩa là bạn đang đứng trên cluster đã chạy Lab 6a dở dang — restore `02-net-ready`
cho cả ba VM. `deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint 7 **chưa đủ**; sửa repo ở tag
`lab-5b` trước, không sửa tay trên cluster ([mục 6.1](DU-AN-BOOKSTORE.md#61-mở-đầu)).

## B1. [BS-6A-01] Điều tra: dữ liệu của `db` sống tới đâu

**Loại:** Điều tra — trên `lab-k8s-worker2`, nơi `db-0` chạy.

**Bối cảnh:** `TD-04` nằm trong sổ nợ từ Lab 3a: dữ liệu DB ở trên `emptyDir`. Trước khi sửa, đội
cần bằng chứng đo được về **ranh giới** của nó trên chính hệ thống đang chạy — sống qua sự cố nào,
chết theo sự cố nào, và nằm ở đâu trên đĩa node — để sau B3 chạy lại đúng phép thử đó và so sánh.

**Yêu cầu:**

1. Ghi một đơn hàng đánh dấu vào `api.orders` của DB đang chạy; ghi lại `id` của nó.
2. Làm tiến trình PostgreSQL kết thúc để kubelet khởi động lại container `postgres` (như kịch bản A
   ở Lab 3a B4). Chứng minh: `db-0` giữ nguyên UID, `restartCount` tăng, đơn hàng còn.
3. Từ UID của `db-0`, tìm thư mục của volume dữ liệu trên `lab-k8s-worker2`; chứng minh nó chứa
   file `PG_VERSION` của PostgreSQL.
4. Xóa Pod `db-0`. Chứng minh: StatefulSet tạo lại Pod cùng tên nhưng khác UID; thư mục ở bước 3 đã
   biến mất khỏi node; đơn hàng mất; số sách tụt về số của lần khởi tạo.
5. Khôi phục hệ thống bằng `./scripts/deploy.sh`.

**Ràng buộc:** không sửa repo; không xóa cưỡng bức. Trên node chỉ **đọc**.

**Đọc lại:**

- Bài [280](../280-configure-volume-storage-vi.md#cấu-hình-một-volume-cho-pod-configure-a-volume-for-a-pod)
  — phép thử "giết tiến trình rồi đọc lại".
- Bài [91](../91-volumes-vi.md#emptydir) — mục `emptyDir` và ghi chú về container crash; mục
  [*Cách volume hoạt động*](../91-volumes-vi.md#cách-volume-hoạt-động-how-volumes-work).
- Bài [90](../90-storage-vi.md) — hai nhánh dài hạn và tạm thời.

**Bàn giao:** `docs/postmortem/6a-du-lieu-emptydir.md`; `~/lab-evidence/6a/b1-*.txt`.

**Nghiệm thu:**

- `PASS:` container `postgres` khởi động lại, `db-0` cùng UID, đơn hàng còn.
- `PASS:` volume dữ liệu của `db-0` là một thư mục dưới `/var/lib/kubelet/pods/<UID>` trên
  `lab-k8s-worker2`, có `PG_VERSION`.
- `PASS:` sau khi xóa `db-0`: UID mới, thư mục cũ không còn trên node, đơn hàng mất, số sách giảm.
- `PASS:` `deploy.sh` đưa số sách trở lại ít nhất 8.

**Câu hỏi thiết kế** — trả lời trong `docs/postmortem/6a-du-lieu-emptydir.md`:

1. Bài 280 nói `emptyDir` tồn tại "trong suốt vòng đời của Pod". `db-0` mới cùng tên, cùng node,
   cùng manifest — vì sao vẫn là Pod khác với volume khác? Bằng chứng nào ở bước 3–4 cho thấy điều đó?
2. Ai tạo thư mục đó trên node, ai xóa, khi nào? Nó nằm dưới thư mục gốc nào của kubelet, và vì sao
   điều đó khiến nó thuộc về "lưu trữ tạm thời cục bộ" của bài 95?
3. Bài 65 hứa StatefulSet cấp "lưu trữ ổn định". Vì sao lời hứa đó chưa áp dụng cho `db` của bạn?
   Thiếu đúng thứ gì trong spec?

<details>
<summary>Gợi ý</summary>

- Ghi và lấy `id` trong một lệnh: `INSERT ... RETURNING id`. Chạy `psql` với `-q -tA` để chỉ nhận về
  con số.
- Kubelet đặt volume `emptyDir` của Pod ở
  `/var/lib/kubelet/pods/<UID của Pod>/volumes/kubernetes.io~empty-dir/<tên volume>`. Tên volume đọc
  từ spec của StatefulSet, ở `volumeMounts` có `mountPath` là thư mục dữ liệu PostgreSQL.
- Trên node: `ssh lab-k8s-worker2 "sudo find '<thư mục>' -maxdepth 2 -name PG_VERSION"`.
- Việc dọn thư mục của Pod cũ do kubelet làm sau khi Pod kết thúc; chờ theo điều kiện, đừng kết luận
  ngay sau `kubectl delete`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đọc xong ticket; script này chính là cuộc điều tra</summary>

```bash
NS=bookstore; EV=~/lab-evidence/6a
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
SQL() { kubectl -n "$NS" exec db-0 -c postgres -- psql -X -q -U postgres -d bookstore -tAc "$1"; }
rc() { jp pod/db-0 '{.status.containerStatuses[?(@.name=="postgres")].restartCount}'; }

kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=300s
NB="$(SQL 'SELECT count(*) FROM api.books')"
ID="$(SQL "INSERT INTO api.orders (book_id, quantity) SELECT min(id), 3 FROM api.books RETURNING id" | head -n1)"
echo "don hang danh dau id=$ID ; so sach truoc=$NB" > "$EV/b1-don-hang.txt"
con_don() { [ "$(SQL "SELECT count(*) FROM api.orders WHERE id = ${ID}")" = '1' ]; }

# 1. Container khởi động lại, Pod giữ nguyên
U0="$(jp pod/db-0 '{.metadata.uid}')"; R0="$(rc)"
kubectl -n "$NS" exec db-0 -c postgres -- kill -INT 1
for i in $(seq 1 36); do [ "$(rc)" -gt "$R0" ] && break; sleep 5; done
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=180s
{ [ "$(jp pod/db-0 '{.metadata.uid}')" = "$U0" ] && [ "$(rc)" -gt "$R0" ] && con_don; } \
  && echo 'PASS: container postgres khoi dong lai, db-0 cung UID, don hang con'

# 2. Volume dữ liệu là một thư mục của kubelet trên node
VOL="$(jp statefulset/db '{.spec.template.spec.containers[?(@.name=="postgres")].volumeMounts[?(@.mountPath=="/var/lib/postgresql/data")].name}')"
DIR="/var/lib/kubelet/pods/${U0}/volumes/kubernetes.io~empty-dir/${VOL}"
echo "$DIR" > "$EV/b1-emptydir.txt"
ssh lab-k8s-worker2 "sudo find '$DIR' -maxdepth 2 -name PG_VERSION" >> "$EV/b1-emptydir.txt" 2>&1
grep -q 'PG_VERSION' "$EV/b1-emptydir.txt" \
  && echo 'PASS: du lieu db-0 nam duoi /var/lib/kubelet/pods/<UID> tren lab-k8s-worker2'

# 3. Pod bị thay
kubectl -n "$NS" delete pod db-0 --wait=true
for i in $(seq 1 60); do U1="$(jp pod/db-0 '{.metadata.uid}')"; [ -n "$U1" ] && [ "$U1" != "$U0" ] && break; sleep 2; done
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=300s
echo "uid cu=$U0 ; uid moi=$U1" >> "$EV/b1-don-hang.txt"
[ "$U1" != "$U0" ] && echo 'PASS: StatefulSet tao lai db-0 cung ten, khac UID'
# Ba kết quả tách bạch: CON, MAT, hoặc rỗng khi ssh/sudo lỗi — rỗng không phải "đã bị xóa".
con_dir() {
  ssh -o BatchMode=yes lab-k8s-worker2 "sudo -n true && if sudo test -d '$DIR'; then echo CON; else echo MAT; fi"
}
for i in $(seq 1 30); do [ "$(con_dir)" = CON ] || break; sleep 2; done
case "$(con_dir)" in
  MAT) echo 'PASS: thu muc emptyDir cua Pod cu da bi xoa khoi node' ;;
  CON) echo 'FAIL: thu muc emptyDir cua Pod cu van con' ;;
  *)   echo 'STOP: khong kiem duoc worker2 (ssh hoac sudo loi) — chua ket luan' ;;
esac
NA="$(SQL 'SELECT count(*) FROM api.books')"
echo "so sach sau khi thay Pod=$NA" >> "$EV/b1-don-hang.txt"
{ ! con_don && [ "${NA:-0}" -lt "$NB" ]; } && echo "PASS: don hang $ID mat, so sach tu $NB tut ve $NA"

# 4. Khôi phục bằng repo
cd ~/bookstore && ./scripts/deploy.sh > "$EV/b1-deploy.txt" 2>&1
N="$(SQL 'SELECT count(*) FROM api.books')"
[ "${N:-0}" -ge 8 ] && echo "PASS: deploy.sh chay lai migration, api.books = $N"
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

1. Bài 47 (Lab 3a): Pod thay thế là một Pod **mới** — UID mới — dù cùng tên. `emptyDir` được tạo khi
   Pod được gán vào node và bị xóa khi Pod rời node (bài 91), nên nó gắn với **UID**, không gắn với
   tên. Đường dẫn trên node chứa chính UID đó; thư mục cũ biến mất, Pod mới nhận thư mục rỗng và
   entrypoint chạy lại script khởi tạo. Kill tiến trình chỉ khởi động lại **container** trong cùng
   Pod — đúng phép thử của bài 280.
2. Kubelet tạo khi dựng volume cho Pod và xóa khi dọn Pod đã kết thúc. Nó nằm dưới `/var/lib/kubelet`
   — thư mục gốc của kubelet mà bài 95 gọi là nơi chứa dữ liệu tạm thời cục bộ, cùng filesystem với
   log và lớp ghi được của container. Dữ liệu DB đang ăn chung chỗ với chúng.
3. Bài 65, *Template cho volume claim* và *Lưu trữ ổn định*: lưu trữ ổn định đến từ
   `volumeClaimTemplates`, mỗi Pod một PVC. StatefulSet `db` của Lab 4b cố ý không có trường đó vì
   cluster chưa có StorageClass — nợ lab #2, trả ở B3.

</details>

## B2. [BS-6A-02] Vận hành: dynamic provisioner và StorageClass mặc định

**Loại:** Vận hành.

**Bối cảnh:** muốn trả `TD-04` thì cluster phải cấp được PersistentVolume. Cluster lab là ba VM
bare-metal: bảng *Provisioner* của bài 96 chỉ có provisioner nội bộ cho AzureFile, Portworx và
vSphere, nên phải dùng một **external provisioner**. Dự án chọn local-path-provisioner: cấp phát bằng
thư mục trên đĩa sẵn có của node, cài bằng một manifest, StorageClass của nó dùng
`WaitForFirstConsumer`. Đây là **hạ tầng nền** của cluster
([mục 3.3 của file dự án](DU-AN-BOOKSTORE.md#33-hạ-tầng-nền-của-cluster)): nó nằm trong snapshot, không
nằm trong `deploy.sh`; repo chỉ giữ ghi chép ở `platform/` để Lab 8a dựng lại được.

**Yêu cầu:**

1. **Trước khi cài**, tạo trong `bookstore` hai PVC thử nghiệm 64Mi, `ReadWriteOnce`:
   `thu-khong-class` (không khai class) và `thu-chuoi-rong` (class là chuỗi rỗng). Ghi trạng thái.
2. Tải manifest chính thức của local-path-provisioner **đúng version ở A1.4**, lưu nguyên văn vào
   `platform/local-path-provisioner/local-path-storage.yaml`. Trước khi apply, đối chiếu ba điều: image
   đúng version, tên provisioner, thư mục gốc trên node.
3. Kéo sẵn image của provisioner và image helper mà manifest dùng lên hai worker.
4. Provisioner chạy; StorageClass `local-path` của manifest là class mặc định **duy nhất** của
   cluster.
5. Chứng minh bằng hành vi cả hai điều kiện của hành vi mặc định (bài 98): `thu-khong-class` được gán
   class hồi tố; `thu-chuoi-rong` không bị đụng và vẫn `Pending`; một PVC mới `thu-moi` không khai class
   được điền class **ngay lúc tạo**.
6. `platform/local-path-provisioner/README.md` ghi: version (trỏ về A1.4), nguồn tải, các bước cài,
   cách kiểm, và vì sao không nằm trong `deploy.sh`. Commit hai file.
7. Xóa ba PVC thử nghiệm.

**Ràng buộc:** không cài bằng Helm. Không sửa manifest upstream — kể cả image helper không pin tag.
Không đưa provisioner vào `k8s/` hay `scripts/`.

**Đọc lại:**

- Bài [98](../98-dynamic-provisioning-vi.md): toàn bài, đặc biệt
  [*Hành vi mặc định*](../98-dynamic-provisioning-vi.md#hành-vi-mặc-định-defaulting-behavior).
- Bài [96](../96-storage-classes-vi.md#provisioner): *Provisioner*,
  [*StorageClass mặc định*](../96-storage-classes-vi.md#storageclass-mặc-định-default-storageclass).
- Bài [92](../92-persistent-volumes-vi.md#retroactive-default-storageclass-assignment): *Gán
  StorageClass mặc định hồi tố*, và mục [*Lớp*](../92-persistent-volumes-vi.md#lớp-class-1) của PVC.
- Bài [91](../91-volumes-vi.md#local) — mục `local`: lưu trữ gắn với node chịu được gì, không chịu
  được gì.

**Bàn giao:** `platform/local-path-provisioner/{README.md,local-path-storage.yaml}`;
`docs/adr/NNNN-local-path-provisioner.md` (`NNNN` là số tiếp theo trong `docs/adr/`);
`~/lab-work/6a/thu-pvc-truoc.yaml`; `~/lab-evidence/6a/b2-*.txt`.

**Nghiệm thu:**

- `PASS:` trước khi cài, `thu-khong-class` `Pending` và chưa có class.
- `PASS:` file manifest trong `platform/` đúng version A1.4, provisioner `rancher.io/local-path`, thư
  mục gốc `/opt/local-path-provisioner`.
- `PASS:` Deployment `local-path-provisioner` Ready với đúng image đó.
- `PASS:` đúng một StorageClass mặc định: `local-path`, `Delete`, `WaitForFirstConsumer`.
- `PASS:` `thu-khong-class` nhận `local-path` hồi tố; `thu-chuoi-rong` vẫn `""` và `Pending`;
  `thu-moi` có class ngay khi được tạo.
- `PASS:` không có CSIDriver nào tên `rancher.io/local-path`.
- `PASS:` README nằm trong repo và đã commit; không còn PVC `thu-*`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-local-path-provisioner.md`:

1. Vì sao provisioner nằm ở `platform/` và trong snapshot chứ không nằm trong `deploy.sh`? Trong dự
   án này ai đóng vai "quản trị viên", ai là "người dùng" theo bài 98?
2. Annotation mặc định kiểm được bằng `kubectl get`. Điều kiện thứ hai của bài 98 bạn chứng minh bằng
   hành vi nào?
3. `thu-khong-class` và `thu-chuoi-rong` giống hệt nhau trước khi có class mặc định, khác nhau sau đó.
   Vì sao? Khi nào bạn **muốn** khai `""`?
4. Ba giới hạn của local-path mà dự án phải sống cùng: volume gắn với node (bài 91, mục `local`),
   dung lượng xin trong PVC không được cưỡng chế, và nó không phải CSI driver. Mỗi giới hạn ảnh hưởng
   tới sprint nào sau này?

<details>
<summary>Gợi ý</summary>

1. Manifest nằm ở
   `https://raw.githubusercontent.com/rancher/local-path-provisioner/<version>/deploy/local-path-storage.yaml`.
   Đọc version ở dòng `local-path-provisioner` của A1.4 rồi đặt vào biến `LPP_VERSION`.
2. Ba gate bằng `grep` trên file đã tải: dòng `image: docker.io/rancher/local-path-provisioner:...`,
   dòng `provisioner: rancher.io/local-path`, chuỗi `"/opt/local-path-provisioner"` trong ConfigMap.
3. Image helper nằm trong ConfigMap của manifest (`helperPod.yaml`). Kéo bằng
   `ssh <worker> 'sudo crictl pull <image>'`.
4. Đánh dấu class mặc định bằng annotation `storageclass.kubernetes.io/is-default-class: "true"` —
   `kubectl patch storageclass ...`. Manifest upstream không tự đặt annotation này.
5. Gán hồi tố do một vòng lặp trong control plane làm, không tức thì; chờ theo điều kiện.

</details>

<details>
<summary>Script nghiệm thu — phần A chạy trước khi cài, phần B chạy sau</summary>

Phần A — sau khi đã viết `~/lab-work/6a/thu-pvc-truoc.yaml`, **trước** khi cài:

```bash
NS=bookstore; EV=~/lab-evidence/6a
kubectl apply -f ~/lab-work/6a/thu-pvc-truoc.yaml
sleep 10
kubectl -n "$NS" get pvc thu-khong-class thu-chuoi-rong > "$EV/b2-pvc-truoc.txt" 2>&1
{ [ "$(kubectl -n "$NS" get pvc thu-khong-class -o jsonpath='{.status.phase}')" = 'Pending' ] && \
  [ -z "$(kubectl -n "$NS" get pvc thu-khong-class -o jsonpath='{.spec.storageClassName}')" ]; } \
  && echo 'PASS: truoc khi cai, thu-khong-class Pending va chua co class'
```

Phần B — sau khi cài xong, trong cùng shell có biến `LPP_VERSION`:

```bash
NS=bookstore; EV=~/lab-evidence/6a; P=~/bookstore/platform/local-path-provisioner
case "${LPP_VERSION:-}" in
  v[0-9]*) echo "PASS: LPP_VERSION = $LPP_VERSION" ;;
  *) echo 'FAIL: chua dat LPP_VERSION theo bang A1.4' ;;
esac
{ grep -q "image: docker.io/rancher/local-path-provisioner:${LPP_VERSION}" "$P/local-path-storage.yaml" && \
  grep -q 'provisioner: rancher.io/local-path' "$P/local-path-storage.yaml" && \
  grep -q '"/opt/local-path-provisioner"' "$P/local-path-storage.yaml"; } \
  && echo 'PASS: manifest trong platform/ dung version, dung provisioner, dung thu muc goc'

IMG="$(kubectl -n local-path-storage get deploy local-path-provisioner -o jsonpath='{.spec.template.spec.containers[0].image}')"
{ [ "$IMG" = "docker.io/rancher/local-path-provisioner:${LPP_VERSION}" ] && \
  [ "$(kubectl -n local-path-storage get deploy local-path-provisioner -o jsonpath='{.status.readyReplicas}')" -ge 1 ]; } \
  && echo 'PASS: provisioner Ready dung image da khoa'

kubectl get storageclass -o custom-columns=NAME:.metadata.name,PROVISIONER:.provisioner,RECLAIM:.reclaimPolicy,BINDING:.volumeBindingMode,EXPAND:.allowVolumeExpansion \
  > "$EV/b2-storageclass.txt"
DEF="$(kubectl get storageclass -o jsonpath='{range .items[*]}{.metadata.name}={.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}' | grep '=true$')"
LP="$(kubectl get storageclass local-path -o jsonpath='{.reclaimPolicy}|{.volumeBindingMode}')"
{ [ "$DEF" = 'local-path=true' ] && [ "$LP" = 'Delete|WaitForFirstConsumer' ]; } \
  && echo 'PASS: dung mot class mac dinh: local-path, Delete, WaitForFirstConsumer'

for i in $(seq 1 30); do
  [ -n "$(kubectl -n "$NS" get pvc thu-khong-class -o jsonpath='{.spec.storageClassName}')" ] && break; sleep 2
done
kubectl -n "$NS" create -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: thu-moi
  namespace: bookstore
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 64Mi
EOF
kubectl -n "$NS" get pvc -o custom-columns=NAME:.metadata.name,CLASS:.spec.storageClassName,PHASE:.status.phase \
  > "$EV/b2-pvc-sau.txt"
{ [ "$(kubectl -n "$NS" get pvc thu-khong-class -o jsonpath='{.spec.storageClassName}')" = 'local-path' ] && \
  kubectl -n "$NS" get pvc thu-chuoi-rong -o json | grep -q '"storageClassName": ""' && \
  [ "$(kubectl -n "$NS" get pvc thu-chuoi-rong -o jsonpath='{.status.phase}')" = 'Pending' ] && \
  [ "$(kubectl -n "$NS" get pvc thu-moi -o jsonpath='{.spec.storageClassName}')" = 'local-path' ]; } \
  && echo 'PASS: gan hoi to cho PVC bo trong, khong dung PVC khai "", admission dien class cho PVC moi'

kubectl get csidrivers -o name > "$EV/b2-csidrivers.txt" 2>&1
grep -q 'rancher.io/local-path' "$EV/b2-csidrivers.txt" \
  && echo 'FAIL: co CSIDriver rancher.io/local-path' || echo 'PASS: local-path khong phai CSI driver'

kubectl -n "$NS" delete pvc thu-khong-class thu-chuoi-rong thu-moi --ignore-not-found --wait=true
cd ~/bookstore
{ git ls-files --error-unmatch platform/local-path-provisioner/README.md platform/local-path-provisioner/local-path-storage.yaml >/dev/null 2>&1 && \
  [ -z "$(git status --porcelain platform/)" ]; } && echo 'PASS: README va manifest da commit trong platform/'
[ -z "$(kubectl -n "$NS" get pvc -o name | grep '/thu-')" ] && echo 'PASS: da xoa PVC thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/6a/thu-pvc-truoc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: thu-khong-class
  namespace: bookstore
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 64Mi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: thu-chuoi-rong
  namespace: bookstore
spec:
  storageClassName: ""
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 64Mi
```

Cài, sau khi đã chạy phần A của script nghiệm thu:

```bash
LPP_VERSION='<điền version từ dòng local-path-provisioner của A1.4>'
P=~/bookstore/platform/local-path-provisioner
mkdir -p "$P"
curl -fsSL -o "$P/local-path-storage.yaml" \
  "https://raw.githubusercontent.com/rancher/local-path-provisioner/${LPP_VERSION}/deploy/local-path-storage.yaml"
grep -nE '^kind:|image:|provisioner:|volumeBindingMode:|reclaimPolicy:|local-path-provisioner"' \
  "$P/local-path-storage.yaml" > ~/lab-evidence/6a/b2-manifest.txt

for N in lab-k8s-worker1 lab-k8s-worker2; do
  for I in "docker.io/rancher/local-path-provisioner:${LPP_VERSION}" docker.io/library/busybox:latest; do
    ssh "$N" "sudo crictl pull $I" >/dev/null && echo "PASS: $N co $I" || echo "FAIL: $N khong keo duoc $I"
  done
done

kubectl apply -f "$P/local-path-storage.yaml"
kubectl -n local-path-storage rollout status deployment/local-path-provisioner --timeout=300s
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get storageclass
```

`docker.io/library/busybox:latest` là cách containerd gọi image `docker.io/library/busybox` không
pin tag mà helper Pod của manifest dùng; helper Pod là thứ thật sự `mkdir` và `rm -rf` thư mục volume
trên node.

README — heredoc **không** quote để shell chèn đúng version đã cài:

```bash
cat > ~/bookstore/platform/local-path-provisioner/README.md <<EOF
# local-path-provisioner — hạ tầng nền của mốc 03-storage-ready

- Cài ở Lab 6a, ticket BS-6A-02. Có sẵn trong snapshot 03-storage-ready trở đi.
- Version: ${LPP_VERSION} — giá trị lấy từ bảng A1.4 của Lab 00; đổi version thì sửa A1.4 trước.
- Nguồn: https://raw.githubusercontent.com/rancher/local-path-provisioner/${LPP_VERSION}/deploy/local-path-storage.yaml
- File local-path-storage.yaml ở đây là bản tải về, không sửa gì.
- Provisioner: rancher.io/local-path. Thư mục gốc trên node: /opt/local-path-provisioner.
  Mỗi volume là một thư mục <tên PV>_<namespace>_<tên PVC>.
- Image helper docker.io/library/busybox không pin tag (lựa chọn của upstream): kéo sẵn trên worker.

Cài (quản trị viên cluster, một lần cho mỗi cluster):

    kubectl apply -f platform/local-path-provisioner/local-path-storage.yaml
    kubectl -n local-path-storage rollout status deployment/local-path-provisioner --timeout=300s
    kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

Kiểm: đúng một StorageClass mặc định (local-path), provisioner Ready, kubectl get pv rỗng.

Không nằm trong scripts/deploy.sh hay scripts/teardown.sh: đây là hạ tầng của cluster, teardown
của dự án không được gỡ nó. Gỡ thì xóa hết PV trước, rồi mới delete -f manifest.
EOF
cd ~/bookstore && git add platform/ && git commit -qm 'lab-6a: platform — local-path-provisioner'
```

Ý chính cho ADR:

1. Bài 98: **quản trị viên** dựng sẵn provisioner và StorageClass; **người dùng** chỉ chọn class qua
   `storageClassName`. Ở đây quản trị viên là người dựng cluster (snapshot, `platform/`), người dùng là
   repo ứng dụng. `teardown.sh` chạy cuối mỗi sprint — nếu provisioner nằm trong đó, mỗi lab sau sẽ
   phải cài lại hạ tầng, và mốc `03-storage-ready` mất ý nghĩa.
2. Admission controller `DefaultStorageClass` phải bật trên API server. Bằng chứng: `thu-moi` không
   khai class mà spec có `local-path` ngay khi `create` trả về — chỉ admission làm được việc đó.
3. Bỏ trống nghĩa là "chưa chọn, cho cluster chọn"; `""` nghĩa là "không class" và tự tắt cấp phát
   động (bài 92). Gán hồi tố chỉ chạm PVC bỏ trống. Khai `""` khi bạn muốn PVC chỉ bind vào PV tĩnh
   không class do admin tạo sẵn, kể cả khi cluster có class mặc định.
4. Gắn node: Pod dùng PV chỉ chạy được trên node có thư mục — `db` đã ghim `lab-k8s-worker2`, bảo trì
   node đó là gián đoạn DB (Lab 12). Không cưỡng chế dung lượng: PVC xin 1Gi vẫn ghi được tới khi đầy
   đĩa node — B4 phải tự giới hạn số bản dump. Không phải CSI: không snapshot, không clone, không
   `ReadWriteOncePod`, không mở rộng — nợ #5 ở Lab 6b.

</details>

## B3. [BS-6A-03] Dữ liệu bền cho `db`: StorageClass Retain và `volumeClaimTemplates`

**Loại:** Tính năng — trả **nợ lab #2** và `TD-04`.

> **Đọc lại trước khi làm:** bài [65 — StatefulSets](../65-statefulset-vi.md), bốn mục
> [*Template cho volume claim*](../65-statefulset-vi.md#volume-claim-templates),
> [*Lưu trữ ổn định*](../65-statefulset-vi.md#stable-storage),
> [*Lưu giữ PersistentVolumeClaim*](../65-statefulset-vi.md#persistentvolumeclaim-retention) và
> [*Hạn chế*](../65-statefulset-vi.md#limitations). Ở Lab 4b, `db` lên StatefulSet **không có**
> `volumeClaimTemplates` vì cluster chưa có StorageClass; ticket này là phần còn thiếu của bài 65.

**Bối cảnh:** B1 đã đo xong ranh giới của `emptyDir`; B2 cho cluster khả năng cấp PV. Class mặc định
`local-path` dùng `reclaimPolicy: Delete`: xóa PVC — do gõ nhầm hay do một script — là thư mục dữ liệu
trên node bị xóa theo ngay. Dữ liệu của một hiệu sách không được phép mất theo một lệnh `delete`, nên
`db` cần một class mà **dự án** chọn chính sách giữ dữ liệu.

**Yêu cầu:**

1. StorageClass `bookstore-retain` ở `k8s/cluster/storageclass-retain.yaml`, label theo hợp đồng
   ([mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label)): cùng provisioner với B2; **giữ** volume khi PVC bị
   xóa; chỉ cấp phát khi đã có Pod tiêu thụ; **không** phải class mặc định.
2. StatefulSet `db` có claim template tên `data` dùng class đó, `ReadWriteOnce`, 1Gi, mang label dự
   án (`component: db`, `name: postgresql`) — sinh ra PVC `data-db-0` theo hợp đồng. Volume
   `emptyDir` dữ liệu bị bỏ hẳn.
3. Thư mục dữ liệu của PostgreSQL là một **thư mục con** của volume, không phải gốc của volume.
4. Spec khai tường minh rằng cả thu nhỏ lẫn xóa StatefulSet đều **giữ** PVC.
5. **Chính sách mật khẩu.** Từ sprint này dữ liệu sống lâu hơn Pod, nên mật khẩu nằm trong dữ liệu
   cũng sống lâu hơn `secrets/db.env` ([mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm)).
   Dự án chốt: `secrets/db.env` **đi cùng** dữ liệu DB; `deploy.sh` chỉ sinh bí mật mới khi trên cluster
   không còn dữ liệu DB nào của dự án, và **từ chối** chạy nếu thiếu `secrets/db.env` trong khi dữ
   liệu cũ còn. Ghi ADR.
6. `scripts/deploy.sh`, `scripts/teardown.sh` thay bằng phiên bản Lab 6a — chép nguyên văn từ khối
   bên dưới. `teardown.sh` mới dọn cả PV `Retain` và thư mục dữ liệu trên node.
7. Đọc trên PV của `data-db-0`: ai cấp phát nó, reclaim policy, node, thư mục trên node; chứng minh
   thư mục đó trên `lab-k8s-worker2` chứa dữ liệu PostgreSQL trong thư mục con.
8. Chạy lại `deploy.sh` không mất dữ liệu: đơn hàng ghi trước lần chạy thứ hai còn nguyên, số sách
   không đổi (migration bất biến kết quả).
9. Điều tra, xóa ngay khi xong: StorageClass `thu-immediate` (provisioner của B2, gắn kết tức thời,
   label dự án) và PVC `thu-immediate` — chứng minh vì sao lưu trữ gắn node phải hoãn gắn kết; một yêu
   cầu tăng dung lượng `data-db-0` bị API server từ chối.

**Ràng buộc:** không đổi tên claim template hay PVC — `data` và `data-db-0` là hợp đồng. Không viết PV
cho `db`; để provisioner cấp. Không scale `db`. Không đặt `bookstore-retain` làm mặc định. API server
không cho sửa `volumeClaimTemplates` của StatefulSet đang tồn tại: chuyển đổi bằng
`./scripts/teardown.sh && ./scripts/deploy.sh` — dữ liệu `emptyDir` đằng nào cũng mất (`TD-04`).

**Đọc lại:**

- Bài [96](../96-storage-classes-vi.md): [*Chính sách thu hồi*](../96-storage-classes-vi.md#reclaim-policy),
  [*Chế độ gắn kết volume*](../96-storage-classes-vi.md#volume-binding-mode),
  [*Mở rộng volume*](../96-storage-classes-vi.md#allow-volume-expansion).
- Bài [98](../98-dynamic-provisioning-vi.md#sử-dụng-cấp-phát-động-using-dynamic-provisioning) — *Sử
  dụng cấp phát động*.
- Bài [92](../92-persistent-volumes-vi.md): [*Ràng buộc*](../92-persistent-volumes-vi.md#binding),
  [*Thu hồi*](../92-persistent-volumes-vi.md#reclaiming),
  [*Node Affinity*](../92-persistent-volumes-vi.md#node-affinity),
  [*Mở rộng Persistent Volume Claim*](../92-persistent-volumes-vi.md#expanding-persistent-volumes-claims),
  [*Các chế độ truy cập*](../92-persistent-volumes-vi.md#access-modes).
- Bài [343](../343-run-replicated-stateful-application-vi.md#statefulset) — mục *Tạo StatefulSet*,
  khối `volumeClaimTemplates` cuối manifest.
- Bài [344](../344-run-single-instance-stateful-vi.md#cập-nhật-updating) — mục *Cập nhật*: hai ràng
  buộc của Deployment cho ứng dụng có trạng thái.
- Bài [91](../91-volumes-vi.md#using-subpath) — mục *Sử dụng subPath*.
- [Mục 4.3 của file dự án](DU-AN-BOOKSTORE.md#43-dữ-liệu) — script khởi tạo chỉ chạy khi thư mục dữ
  liệu rỗng.

**Bàn giao:** `k8s/cluster/storageclass-retain.yaml`; `k8s/app/db/statefulset.yaml`; hai script mới;
`docs/adr/NNNN-du-lieu-ben-db.md`; `docs/adr/NNNN-mat-khau-du-lieu-ben.md`;
`~/lab-work/6a/thu-immediate.yaml`; `~/lab-evidence/6a/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` `teardown.sh` in `teardown: sach`; `deploy.sh` chạy hai lần liên tiếp đều thành công.
- `PASS:` StorageClass `bookstore-retain` mang label dự án, provisioner `rancher.io/local-path`,
  `Retain`, `WaitForFirstConsumer`, không phải mặc định.
- `PASS:` PVC `data-db-0` `Bound`, class `bookstore-retain`, mang label dự án; thư mục dữ liệu
  PostgreSQL của `db-0` là volume từ PVC đó.
- `PASS:` PV của `data-db-0` do `rancher.io/local-path` cấp, `Retain`, `nodeAffinity` trỏ
  `lab-k8s-worker2`, thư mục `/opt/local-path-provisioner/<PV>_bookstore_data-db-0`.
- `PASS:` trên `lab-k8s-worker2`, `PG_VERSION` nằm trong một thư mục con của thư mục volume; `PGDATA`
  trong spec là thư mục con của điểm mount.
- `PASS:` `persistentVolumeClaimRetentionPolicy` khai tường minh `Retain` cho cả hai trường hợp.
- `PASS:` đơn hàng ghi trước lần `deploy.sh` thứ hai còn; số sách không đổi.
- `PASS:` thiếu `secrets/db.env` trong khi dữ liệu cũ còn thì `deploy.sh` thoát khác 0, không sinh
  bí mật mới.
- `PASS:` PVC `thu-immediate` đứng `Pending` kèm event `Warning`; cả hai object thử nghiệm đã xóa.
- `PASS:` yêu cầu tăng dung lượng `data-db-0` bị API server từ chối.
- `PASS:` `tests/smoke.sh` của Lab 5b vẫn PASS.

Hai script, chép nguyên văn:

```bash
cat > ~/bookstore/scripts/deploy.sh <<'EOF'
#!/usr/bin/env bash
# Triển khai dự án Bookstore từ repo — phiên bản Lab 6a.
# So với Lab 5b:
#  - kiểm hạ tầng lưu trữ của mốc 03-storage-ready trước khi triển khai;
#  - không sinh secrets/db.env mới khi dữ liệu DB cũ còn trên cluster: mật khẩu thật nằm trong
#    chính dữ liệu đó (ADR mat-khau-du-lieu-ben, Lab 6a);
#  - Job db-migrate chỉ được tạo sau khi db-0 Ready — lần đầu db-0 còn phải chờ cấp phát volume.
set -euo pipefail
cd "$(dirname "$0")/.."
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
HOST=bookstore.lab.local
RENDER="$(mktemp)"
trap 'rm -f "$RENDER"' EXIT

# 0. Hạ tầng nền thuộc cluster, không thuộc repo: thiếu thì dừng, không tự cài.
#    Calico, IngressClass traefik, Service traefik của mốc 02-net-ready (5b).
for X in 'daemonset/calico-node -n kube-system' 'ingressclass/traefik' 'service/traefik -n traefik'; do
  kubectl get $X >/dev/null 2>&1 || { echo "deploy: thieu $X - cluster chua o moc 02-net-ready"; exit 1; }
done
#    Provisioner rancher.io/local-path đang chạy (platform/local-path-provisioner, 6a).
kubectl get storageclass -o jsonpath='{range .items[*]}{.provisioner}{"\n"}{end}' \
  | grep -qx 'rancher.io/local-path' \
  || { echo 'deploy: cluster chua co provisioner rancher.io/local-path (moc 03-storage-ready)'; exit 1; }
kubectl -n local-path-storage rollout status deployment/local-path-provisioner --timeout=120s

# 1. Bí mật sinh tại chỗ khi thiếu; secrets/ không vào git (Lab 3b, 5b).
mkdir -p secrets
if [ ! -s secrets/db.env ]; then
  CU="$( { kubectl -n "$NS" get pvc data-db-0 -o name 2>/dev/null || true
           kubectl get pv -o jsonpath='{range .items[*]}{.spec.claimRef.namespace}/{.spec.claimRef.name}{"\n"}{end}' \
             | grep -x "$NS/data-db-0" || true; } | wc -l)"
  if [ "$CU" -gt 0 ]; then
    echo 'deploy: thieu secrets/db.env trong khi du lieu db cu con tren cluster.'
    echo 'deploy: chep lai secrets/db.env cu, hoac chay scripts/teardown.sh de xoa du lieu cu.'
    exit 1
  fi
  ( umask 077
    printf 'postgres-password=%s\nauthenticator-password=%s\n' \
      "$(openssl rand -hex 24)" "$(openssl rand -hex 24)" > secrets/db.env )
  echo 'deploy: da sinh secrets/db.env moi'
fi
if [ ! -s secrets/tls.crt ] || [ ! -s secrets/tls.key ]; then
  ( umask 077
    openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
      -subj '/CN=bookstore.lab.local' -addext 'subjectAltName=DNS:bookstore.lab.local' \
      -keyout secrets/tls.key -out secrets/tls.crt 2>/dev/null )
  echo 'deploy: da sinh certificate tu ky moi cho bookstore.lab.local'
fi

# 2. Object cấp cluster (PriorityClass, StorageClass của dự án), rồi render namespace bookstore.
#    secrets/ nằm ngoài thư mục gốc k8s/app nên phải render với --load-restrictor (Lab 3b, ADR 0007).
if compgen -G 'k8s/cluster/*.yaml' >/dev/null; then kubectl apply -f k8s/cluster/; fi
kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app > "$RENDER"

# 3. Mọi thứ trừ migration. Pod template của Job không sửa được: xóa db-migrate cũ để migration
#    chạy lại ở mỗi lần triển khai (bất biến kết quả, Lab 4b).
kubectl -n "$NS" delete job db-migrate --ignore-not-found --wait=true
kubectl apply -f "$RENDER" -l 'app.kubernetes.io/component!=migrate'

# 4. db-0. Lần đầu, PVC data-db-0 chỉ được cấp PV sau khi db-0 được lập lịch (WaitForFirstConsumer).
kubectl -n "$NS" rollout status statefulset/db --timeout=300s
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=300s

wait_job() {   # thành công khi Job Complete, thất bại ngay khi Job Failed
  local c=''
  for _ in $(seq 1 100); do
    c="$(kubectl -n "$NS" get job "$1" \
          -o jsonpath='{.status.conditions[?(@.status=="True")].type}' 2>/dev/null || true)"
    case " $c " in
      *' Complete '*) return 0 ;;
      *' Failed '*) kubectl -n "$NS" logs "job/$1" --tail=20 || true; return 1 ;;
    esac
    sleep 3
  done
  return 1
}

# 5. Migration, sau khi db-0 đã nhận kết nối.
if grep -q 'app.kubernetes.io/component: migrate' "$RENDER"; then
  kubectl apply -f "$RENDER" -l 'app.kubernetes.io/component=migrate'
  wait_job db-migrate || { echo 'deploy: db-migrate that bai'; exit 1; }
fi

# 6. Chờ các controller còn lại.
for R in $(kubectl -n "$NS" get deployment,daemonset -l "$SEL" -o name); do
  kubectl -n "$NS" rollout status "$R" --timeout=300s
done

# 7. Đường vào (5b): Traefik phải phục vụ HTTPS bằng đúng certificate của dự án.
NP="$(kubectl -n traefik get service traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}')"
IP="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
OK=0
for _ in $(seq 1 30); do
  if curl -fsS --max-time 5 --cacert secrets/tls.crt --resolve "${HOST}:${NP}:${IP}" \
       "https://${HOST}:${NP}/healthz" >/dev/null 2>&1; then OK=1; break; fi
  sleep 2
done
[ "$OK" -eq 1 ] || { echo "deploy: https://${HOST}:${NP}/ chua phuc vu qua Traefik"; exit 1; }

kubectl -n "$NS" get statefulset,deployment,daemonset,cronjob,job,pvc -o wide
EOF

cat > ~/bookstore/scripts/teardown.sh <<'EOF'
#!/usr/bin/env bash
# Gỡ toàn bộ dự án Bookstore khỏi cluster — phiên bản Lab 6a.
# Từ 6a, dữ liệu của dự án sống ngoài vòng đời namespace: PV reclaimPolicy Retain nằm lại ở pha
# Released sau khi PVC bị xóa, và thư mục do local-path-provisioner tạo trên node vẫn còn sau khi
# PV bị xóa (bài 92, mục Retain). Snapshot chỉ chứa hạ tầng, nên teardown dọn cả hai rồi tự kiểm.
# secrets/ trên master được giữ nguyên.
set -euo pipefail
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
SC=bookstore-retain
BASE=/opt/local-path-provisioner   # thư mục gốc của local-path-provisioner (Lab 6a, BS-6A-02)
# Kind namespaced được dọn theo label ở mọi namespace (3b, 4a, 5a, 5b); 6a thêm PVC.
KINDS='statefulsets,deployments,daemonsets,cronjobs,jobs,replicasets,pods,poddisruptionbudgets,services,networkpolicies,ingresses,configmaps,secrets,persistentvolumeclaims'

# PV của dự án, mỗi dòng "tên|reclaimPolicy": mang label dự án, thuộc StorageClass của dự án,
# hoặc từng gắn với một PVC trong namespace bookstore.
pv_du_an() {
  kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}|{.spec.persistentVolumeReclaimPolicy}|{.spec.storageClassName}|{.spec.claimRef.namespace}|{.metadata.labels.app\.kubernetes\.io/part-of}{"\n"}{end}' \
    | awk -F'|' -v ns="$NS" -v sc="$SC" '$3==sc || $4==ns || $5=="bookstore" {print $1 "|" $2}'
}
# Worker — nơi local-path-provisioner có thể đã tạo thư mục cho Pod của dự án.
workers() {
  kubectl get nodes -l '!node-role.kubernetes.io/control-plane' \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
}
# Liệt kê thư mục dữ liệu của dự án trên một worker. Thư mục gốc chưa từng có nghĩa là không có gì;
# lỗi ssh, sudo hay find thì trả mã khác 0 — "không kiểm được" không bao giờ được hiểu thành
# "không còn gì".
dirs_on() {
  ssh -o BatchMode=yes "$1" \
    "sudo -n test ! -e $BASE || sudo find $BASE -mindepth 1 -maxdepth 1 -name '*_${NS}_*'"
}

# 1. db tắt êm trước khi xóa (bài 65, mục Hạn chế).
if kubectl -n "$NS" get statefulset db >/dev/null 2>&1; then
  kubectl -n "$NS" scale statefulset db --replicas=0
  kubectl -n "$NS" wait --for=delete pod/db-0 --timeout=180s || true
fi

# 2. Namespace kéo theo mọi PVC bên trong; rồi object mang label ở namespace khác.
kubectl delete namespace -l "$SEL" --ignore-not-found --wait=true --timeout=300s
kubectl delete "$KINDS" --all-namespaces -l "$SEL" \
  --ignore-not-found --wait=true --timeout=120s

# 3. PV: loại Delete do provisioner tự thu hồi — chờ một lúc; loại Retain phải xóa tay (bước 1 của
#    thu hồi thủ công, bài 92). Hết thời gian chờ thì xóa nốt mọi PV còn lại của dự án.
for _ in $(seq 1 30); do
  case "$(pv_du_an)" in *'|Delete'*) sleep 2 ;; *) break ;; esac
done
for P in $(pv_du_an | cut -d'|' -f1); do
  kubectl delete pv "$P" --ignore-not-found --wait=true --timeout=120s
done

# 4. Thư mục dữ liệu trên node, chỉ sau khi PV đã bị xóa. Thư mục volume tên <PV>_<namespace>_<PVC>,
#    nên mẫu *_bookstore_* chỉ khớp dữ liệu của namespace bookstore.
WS="$(workers)" && [ -n "$WS" ] \
  || { echo 'teardown: STOP - khong lay duoc danh sach worker tu API server'; exit 1; }
for N in $WS; do
  ssh -o BatchMode=yes "$N" "sudo -n sh -c 'rm -rf $BASE/*_${NS}_*'" \
    || { echo "teardown: STOP - khong xoa duoc thu muc du lieu tren $N (ssh hoac sudo loi)"; exit 1; }
done

# 5. Object cấp cluster.
kubectl delete storageclass "$SC" --ignore-not-found
kubectl delete storageclass,priorityclass -l "$SEL" --ignore-not-found

# 6. Tự kiểm.
# Mỗi truy vấn kiểm kê chạy riêng và phải thành công: API không trả lời hay Forbidden thì dừng —
# "không hỏi được" không bao giờ được đếm thành "không còn gì". dem <biến> <lệnh...> cộng số dòng
# output của lệnh vào biến; hàm chạy trong shell chính nên exit dừng cả script.
dem() {
  local v="$1" out; shift
  out="$("$@" 2>/dev/null)" || { echo "teardown: STOP - truy van loi, chua ket luan sach: $*"; exit 1; }
  printf -v "$v" '%d' "$(( ${!v:-0} + $(printf '%s\n' "$out" | awk 'NF' | wc -l) ))"
}
LEFT=0
dem LEFT kubectl get namespace,priorityclass,storageclass -l "$SEL" --no-headers
dem LEFT kubectl get storageclass "$SC" --no-headers --ignore-not-found
dem LEFT kubectl get "all,$KINDS,endpointslices" -A -l "$SEL" --no-headers
dem LEFT pv_du_an
# Mỗi worker cho một trong ba kết quả: còn thư mục, đã xác nhận không còn, hoặc không kiểm được.
# Kết quả thứ ba dừng script: chưa kết luận sạch thì không được chụp snapshot.
DIRS=0
for N in $WS; do
  L="$(dirs_on "$N")" \
    || { echo "teardown: STOP - khong kiem duoc thu muc du lieu tren $N; chua ket luan sach, KHONG chup snapshot"; exit 1; }
  DIRS=$((DIRS + $(printf '%s\n' "$L" | grep -c . || true)))
done
if [ "$LEFT" -eq 0 ] && [ "$DIRS" -eq 0 ]; then
  echo 'teardown: sach'
else
  echo "teardown: con $LEFT object va $DIRS thu muc du lieu tren node"; exit 1
fi
# Hạ tầng nền không mang label dự án và phải còn nguyên (5b).
{ kubectl get ingressclass traefik >/dev/null 2>&1 && kubectl -n kube-system get daemonset calico-node >/dev/null 2>&1; } \
  || echo 'teardown: CANH BAO - ha tang 02-net-ready khong con nguyen (khong do script nay go)'
EOF

chmod +x ~/bookstore/scripts/*.sh
```

`deploy.sh` giả định bố cục của tag `lab-5b` theo [mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint):
mọi object trong namespace `bookstore` — kể cả Job `db-migrate`, NetworkPolicy, Ingress — được
`k8s/app/kustomization.yaml` gom; Secret `db-credentials` sinh từ `secrets/db.env`, Secret TLS sinh từ
`secrets/tls.crt` và `secrets/tls.key`. Nếu `deploy.sh` ở tag `lab-5b` sinh bí mật khác đi (tên file,
tham số certificate), giữ nguyên phần sinh bí mật của bản cũ trong khối số 1 và chỉ thay phần còn lại.

Hai điểm cần hiểu ở hai script — chúng là công cụ, nhưng hành vi của chúng là nội dung của sprint:

- `deploy.sh` không còn tạo Job `db-migrate` cùng lúc với `db`: lần đầu, `db-0` phải chờ provisioner
  tạo thư mục và PV, và một migration chạy sớm sẽ đốt hết số lần thử lại.
- `teardown.sh` xóa thư mục trên node bằng mẫu `*_bookstore_*` **sau** khi PV đã bị xóa — đúng thứ tự
  thu hồi thủ công của bài 92. Đó là lần duy nhất trong cả dự án một script xóa dữ liệu trên node.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-du-lieu-ben-db.md`:

1. Cluster đã có class mặc định `local-path`. Vì sao `db` cần một class riêng? Ai quyết định reclaim
   policy của một PV cấp phát động, và sửa class sau khi PV đã tạo thì PV cũ có đổi theo không?
2. Bằng chứng ở `thu-immediate`: vì sao gắn kết tức thời hỏng với provisioner này, và
   `WaitForFirstConsumer` sửa đúng điều gì? Vì sao Pod dùng class đó không được chọn node bằng
   `nodeName`?
3. Bài 344 chạy DB đơn thực thể bằng Deployment + PVC + `Recreate`. Ở quy mô một bản sao, cách đó và
   StatefulSet + `volumeClaimTemplates` mỗi cách cho gì, mất gì? Vì sao dự án giữ StatefulSet?
4. Ba cách đặt thư mục dữ liệu: gốc volume, mount bằng `subPath`, thư mục con qua `PGDATA`. Vì sao
   chọn cách cuối — nghĩ tới Lab 9b, khi `db` phải chạy uid 999 ngay từ đầu?
5. Bài 65 khuyến nghị `ReadWriteOncePod` cho production. Cluster này cấp được không? `ReadWriteOnce`
   bảo vệ dữ liệu tới đâu, và thứ gì thật sự giữ "nhiều nhất một Pod ghi vào `data-db-0`"?

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-mat-khau-du-lieu-ben.md`:

1. Vì sao script khởi tạo (kể cả `02-authenticator.sh` đặt mật khẩu `authenticator`) không chạy lại
   khi dữ liệu đã bền? Hệ quả gì nếu `secrets/db.env` bị sinh lại trong khi `data-db-0` còn — với
   `api` (PostgREST coi lỗi xác thực là lỗi không thử lại), với `db-backup`?
2. Vì sao `deploy.sh` **từ chối** thay vì tự đặt lại mật khẩu trong DB cho khớp Secret mới? Viết quy
   trình khi mất `secrets/db.env` mà dữ liệu còn.
3. Bundle không chứa `secrets/`; restore snapshot đưa cả `~/bookstore/secrets/` về quá khứ. Cặp
   "dữ liệu — bí mật" nào có thể lệch nhau sau một lần restore, và vì sao `teardown.sh` trước khi chụp
   snapshot loại bỏ được rủi ro đó?

<details>
<summary>Gợi ý</summary>

1. StorageClass: bốn trường `provisioner`, `reclaimPolicy`, `volumeBindingMode`, và label. Class
   không mặc định là class không mang annotation `is-default-class: "true"`.
2. `volumeClaimTemplates` là trường của `spec` StatefulSet, cùng cấp với `template`. Mỗi phần tử có
   `metadata` (tên, label) và `spec` như một PVC. Container mount bằng **tên** của claim template, như
   mount một volume thường.
3. Image `postgres` đọc biến `PGDATA`. Thư mục con chưa tồn tại thì entrypoint tự tạo với quyền đúng.
4. `kubectl explain statefulset.spec.persistentVolumeClaimRetentionPolicy`.
5. Đọc PV: `kubectl get pv <tên> -o yaml`; các trường cần xem nằm ở `metadata.annotations`,
   `spec.persistentVolumeReclaimPolicy`, `spec.nodeAffinity`, `spec.hostPath`.
6. Tăng dung lượng mà không đổi gì thật: `kubectl patch pvc ... --dry-run=server`.
7. Event của PVC: `kubectl get events --field-selector involvedObject.name=<tên>`.
8. Trình tự an toàn: viết hai manifest, chép hai script, `./scripts/teardown.sh && ./scripts/deploy.sh`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
NS=bookstore; EV=~/lab-evidence/6a
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
SQL() { kubectl -n "$NS" exec db-0 -c postgres -- psql -X -q -U postgres -d bookstore -tAc "$1"; }

./scripts/teardown.sh > "$EV/b3-teardown.txt" 2>&1
grep -q 'teardown: sach' "$EV/b3-teardown.txt" && echo 'PASS: teardown.sh go sach du an'
./scripts/deploy.sh > "$EV/b3-deploy-1.txt" 2>&1 && echo 'PASS: deploy.sh lan 1 thanh cong'

SCJ="$(kubectl get storageclass bookstore-retain -o jsonpath='{.provisioner}|{.reclaimPolicy}|{.volumeBindingMode}|{.metadata.labels.app\.kubernetes\.io/part-of}')"
SCD="$(kubectl get storageclass bookstore-retain -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}')"
echo "$SCJ default=$SCD" > "$EV/b3-storageclass.txt"
{ [ "$SCJ" = 'rancher.io/local-path|Retain|WaitForFirstConsumer|bookstore' ] && [ "$SCD" != 'true' ]; } \
  && echo 'PASS: bookstore-retain: local-path, Retain, WaitForFirstConsumer, label du an, khong mac dinh'

[ "$(jp pvc/data-db-0 '{.status.phase}|{.spec.storageClassName}|{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'Bound|bookstore-retain|bookstore' ] \
  && echo 'PASS: PVC data-db-0 Bound, class bookstore-retain, label du an'
VOL="$(jp pod/db-0 '{.spec.containers[?(@.name=="postgres")].volumeMounts[?(@.mountPath=="/var/lib/postgresql/data")].name}')"
[ "$(jp pod/db-0 "{.spec.volumes[?(@.name==\"$VOL\")].persistentVolumeClaim.claimName}")" = 'data-db-0' ] \
  && echo 'PASS: thu muc du lieu PostgreSQL cua db-0 la volume tu PVC data-db-0'

PV="$(jp pvc/data-db-0 '{.spec.volumeName}')"
pv() { kubectl get pv "$PV" -o jsonpath="$1"; }
DIR="$(pv '{.spec.hostPath.path}{.spec.local.path}')"
kubectl get pv "$PV" -o yaml > "$EV/b3-pv.yaml"
{ [ "$(pv '{.metadata.annotations.pv\.kubernetes\.io/provisioned-by}')" = 'rancher.io/local-path' ] && \
  [ "$(pv '{.spec.persistentVolumeReclaimPolicy}')" = 'Retain' ] && \
  [ "$(pv '{.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}')" = 'lab-k8s-worker2' ] && \
  [ "$DIR" = "/opt/local-path-provisioner/${PV}_bookstore_data-db-0" ]; } \
  && echo "PASS: PV $PV do provisioner cap, Retain, gan lab-k8s-worker2, thu muc $DIR"
ssh lab-k8s-worker2 "sudo find '$DIR' -mindepth 2 -maxdepth 2 -name PG_VERSION" > "$EV/b3-tren-node.txt" 2>&1
grep -q 'PG_VERSION' "$EV/b3-tren-node.txt" && echo 'PASS: PG_VERSION nam trong thu muc con cua volume tren lab-k8s-worker2'
MP="$(jp pod/db-0 "{.spec.containers[?(@.name==\"postgres\")].volumeMounts[?(@.name==\"$VOL\")].mountPath}")"
PGD="$(jp pod/db-0 '{.spec.containers[?(@.name=="postgres")].env[?(@.name=="PGDATA")].value}')"
case "$PGD" in "$MP"/?*) echo "PASS: PGDATA=$PGD la thu muc con cua $MP" ;; *) echo "FAIL: PGDATA='$PGD' khong nam duoi $MP" ;; esac
{ [ "$(jp statefulset/db '{.spec.persistentVolumeClaimRetentionPolicy.whenDeleted}|{.spec.persistentVolumeClaimRetentionPolicy.whenScaled}')" = 'Retain|Retain' ] && \
  grep -q 'persistentVolumeClaimRetentionPolicy' k8s/app/db/statefulset.yaml; } \
  && echo 'PASS: chinh sach giu PVC khai tuong minh Retain/Retain'

NB="$(SQL 'SELECT count(*) FROM api.books')"
ID="$(SQL "INSERT INTO api.orders (book_id, quantity) SELECT min(id), 2 FROM api.books RETURNING id" | head -n1)"
./scripts/deploy.sh > "$EV/b3-deploy-2.txt" 2>&1 && echo 'PASS: deploy.sh lan 2 thanh cong'
{ [ "$(SQL "SELECT count(*) FROM api.orders WHERE id = ${ID}")" = '1' ] && [ "$(SQL 'SELECT count(*) FROM api.books')" = "$NB" ]; } \
  && echo "PASS: don hang $ID con, so sach giu $NB sau lan deploy thu hai"

mv secrets/db.env secrets/db.env.giu
./scripts/deploy.sh > "$EV/b3-deploy-thieu-bi-mat.txt" 2>&1; RC=$?
mv secrets/db.env.giu secrets/db.env
{ [ "$RC" -ne 0 ] && grep -q 'du lieu db cu' "$EV/b3-deploy-thieu-bi-mat.txt" && [ -s secrets/db.env ]; } \
  && echo 'PASS: thieu secrets/db.env khi du lieu cu con thi deploy.sh tu choi, khong sinh bi mat moi'

kubectl apply -f ~/lab-work/6a/thu-immediate.yaml
sleep 30
PH="$(jp pvc/thu-immediate '{.status.phase}')"
W="$(kubectl -n "$NS" get events --field-selector involvedObject.kind=PersistentVolumeClaim,involvedObject.name=thu-immediate,type=Warning --no-headers 2>/dev/null | wc -l)"
kubectl -n "$NS" describe pvc thu-immediate > "$EV/b3-immediate.txt" 2>&1
{ [ "$PH" = 'Pending' ] && [ "$W" -ge 1 ]; } && echo 'PASS: Immediate cap phat ngay va that bai ngay vi chua biet node'
kubectl -n "$NS" delete pvc thu-immediate --ignore-not-found --wait=true
kubectl delete storageclass thu-immediate --ignore-not-found
{ ! kubectl get storageclass thu-immediate >/dev/null 2>&1 && ! kubectl -n "$NS" get pvc thu-immediate >/dev/null 2>&1; } \
  && echo 'PASS: da xoa hai object thu nghiem'

OUT="$(kubectl -n "$NS" patch pvc data-db-0 --dry-run=server -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}' 2>&1)"; RC=$?
echo "$OUT" > "$EV/b3-mo-rong.txt"
{ [ "$RC" -ne 0 ] && grep -qi 'forbidden' "$EV/b3-mo-rong.txt"; } && echo 'PASS: class khong cho mo rong thi API server tu choi tang dung luong'

./tests/smoke.sh > "$EV/b3-smoke-5b.txt" 2>&1 && echo 'PASS: smoke cua Lab 5b van PASS'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/cluster/storageclass-retain.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: bookstore-retain
  labels:
    app.kubernetes.io/part-of: bookstore
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

Khung StatefulSet giữ nguyên như tag `lab-5b`; dưới đây chỉ các phần thay đổi. Phần `template.spec`
không nhắc tới (probe, tài nguyên, PriorityClass, ân hạn, Secret, ConfigMap `db-init`) **giữ của
bạn**.

```yaml
# k8s/app/db/statefulset.yaml — các phần thay đổi ở Lab 6a
spec:
  serviceName: db
  replicas: 1
  persistentVolumeClaimRetentionPolicy:    # mặc định đã là Retain; khai tường minh để ghi ý đồ
    whenDeleted: Retain
    whenScaled: Retain
  template:
    spec:
      volumes:
      # - name: data            <- XÓA: emptyDir của Lab 4b (TD-04)
      #   emptyDir: {}
      - name: initdb
        configMap:
          name: db-init
      - name: credentials
        secret:
          secretName: db-credentials
          defaultMode: 0400
          items:
          - key: postgres-password
            path: postgres-password
      containers:
      - name: postgres
        env:
        - name: PGDATA                     # thư mục con của volume (ADR du-lieu-ben-db)
          value: /var/lib/postgresql/data/pgdata
        # ... các biến khác giữ nguyên
        volumeMounts:
        - name: data                       # tên claim template, không phải tên PVC
          mountPath: /var/lib/postgresql/data
        # ... initdb, credentials giữ nguyên
  volumeClaimTemplates:
  - metadata:
      name: data                           # PVC sinh ra: data-db-0
      labels:
        app.kubernetes.io/part-of: bookstore
        app.kubernetes.io/component: db
        app.kubernetes.io/name: postgresql
    spec:
      storageClassName: bookstore-retain
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

Nếu `volumes` của bạn sinh từ Kustomize với tên ConfigMap có hậu tố hash, giữ nguyên cách đó; chỉ
bỏ volume `emptyDir` dữ liệu.

```yaml
# ~/lab-work/6a/thu-immediate.yaml — mang label dự án để teardown dọn nếu quên
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: thu-immediate
  labels:
    app.kubernetes.io/part-of: bookstore
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: Immediate
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: thu-immediate
  namespace: bookstore
spec:
  storageClassName: thu-immediate
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 64Mi
```

```bash
cd ~/bookstore
# chép hai script ở trên, viết hai manifest, rồi:
./scripts/teardown.sh && ./scripts/deploy.sh
kubectl -n bookstore get pvc,pod -o wide
```

Ý chính cho ADR `du-lieu-ben-db`:

1. Bài 96: PV cấp phát động **kế thừa** `reclaimPolicy` của class tại lúc cấp; PV tạo tay giữ policy
   được gán lúc tạo. Quản trị viên cấu hình class theo kỳ vọng của người dùng thay vì sửa từng PV.
   Class mặc định là của cả cluster — đổi nó sang `Retain` là áp chính sách của DB lên mọi PVC của mọi
   người. Sửa class không đổi PV đã có; muốn đổi phải patch từng PV.
2. Provisioner phải biết **node** để tạo thư mục; với `Immediate`, PVC được xử lý khi chưa có Pod nên
   chưa có node — event `ProvisioningFailed`. `WaitForFirstConsumer` để scheduler chọn node cho Pod
   trước (theo `nodeSelector`, tài nguyên...), rồi mới cấp volume ở đúng node đó (bài 96).
   `nodeName` bỏ qua scheduler nên PVC đứng mãi ở `Pending`.
3. Deployment + PVC + `Recreate` chạy được một bản: PVC tự tạo tay, tên tùy ý, Pod tên ngẫu nhiên;
   `Recreate` bắt buộc vì không thể có hai Pod cùng lúc (bài 344). StatefulSet cho thêm: PVC sinh theo
   ordinal và **gắn với danh tính** `db-0`, tên DNS `db-0.db`, ngữ nghĩa nhiều nhất một Pod khi thay
   (bài 341), thu nhỏ hay xóa không đụng tới PVC. Dự án đã dựa vào cả bốn điều đó.
4. Gốc volume: trên đĩa khối thật thường có `lost+found`, `initdb` từ chối thư mục không rỗng.
   `subPath`: thư mục con do kubelet tạo trước khi container chạy, chủ sở hữu là root — PostgreSQL chạy
   uid 999 không `chmod`/`chown` được nó. `PGDATA` trỏ thư mục con chưa tồn tại: entrypoint tự tạo,
   quyền 700, đúng chủ — kể cả khi Lab 9b bỏ hẳn root.
5. Không: `ReadWriteOncePod` chỉ cho volume CSI (bài 92), provisioner này không phải CSI. Access mode
   chỉ dùng để khớp PVC với PV và ràng buộc nơi mount, không chống ghi; `ReadWriteOnce` còn cho nhiều
   Pod cùng node dùng chung. Thứ thật sự giữ "một người ghi" là StatefulSet một bản sao và ngữ nghĩa
   nhiều nhất một — và lý do không bao giờ buộc xóa `db-0` (Lab 4b).

Ý chính cho ADR `mat-khau-du-lieu-ben`:

1. Entrypoint chỉ khởi tạo khi `PGDATA` chưa có `PG_VERSION` ([mục 4.3](DU-AN-BOOKSTORE.md#43-dữ-liệu)).
   Mật khẩu `postgres` và `authenticator` được đặt **một lần** rồi nằm trong dữ liệu. Sinh lại
   `secrets/db.env` thì Secret mới, dữ liệu cũ: `api` không đăng nhập được và PostgREST thoát ngay
   (lỗi xác thực không được thử lại) — `CrashLoopBackOff`; `db-backup` đăng nhập TCP bằng
   `scram-sha-256` cũng thất bại.
2. Tự đặt lại mật khẩu là âm thầm sửa dữ liệu thật mỗi lần triển khai — che mất sự cố "mất bí mật"
   và biến `deploy.sh` thành công cụ xoay mật khẩu không ai duyệt. Từ chối thì người vận hành phải
   chọn: chép lại `secrets/db.env` cũ (từ nơi lưu ngoài VM), hoặc xóa dữ liệu (`teardown.sh`), hoặc
   chủ động đặt lại mật khẩu role qua Unix socket trong `db-0` (xác thực `trust`) rồi viết
   `secrets/db.env` khớp với nó. Lab 6b và 8a nạp lại bản dump sẽ gặp đúng lựa chọn này.
3. Restore snapshot đưa `secrets/db.env` về thời điểm chụp; dữ liệu trên node thì không theo bundle.
   Vì mốc `03-storage-ready` được chụp **sau** teardown — không còn PV, không còn thư mục dữ liệu nào
   của dự án — lần `deploy.sh` đầu tiên sau restore luôn khởi tạo DB mới khớp với `secrets/db.env` đang
   có.

</details>

## B4. [BS-6A-04] PVC `backup-store` cho `db-backup`

**Loại:** Tính năng — trả `TD-08`, phát sinh `TD-10`.

**Bối cảnh:** từ Lab 4b, CronJob `db-backup` ghi bản dump vào `emptyDir` của Pod Job — Pod bị dọn là
bản dump mất theo (`TD-08`). Yêu cầu `R8` đòi cả bản backup sống sót qua việc xóa Pod, xóa workload
([luồng dữ liệu bền](DU-AN-BOOKSTORE.md#34-luồng-request-và-luồng-dữ-liệu)). PV của provisioner nằm
trên **một** node, nên Pod nào ghi hay đọc bản backup cũng phải chạy ở đúng node đó — và chọn node nào
là một quyết định thiết kế, không phải chi tiết.

**Yêu cầu:**

1. PVC `backup-store` ở `k8s/app/backup/pvc.yaml`, label theo hợp đồng (`component: backup`,
   `name: postgresql`), class `bookstore-retain`, `ReadWriteOnce`, 1Gi; thêm vào
   `k8s/app/kustomization.yaml`.
2. CronJob `db-backup` ghi bản dump vào PVC đó, mount tại `/backup`, thay cho `emptyDir`. Mỗi lần
   chạy: dump ra file tạm, kiểm file có bảng `api.books`, rồi mới đổi sang tên chính thức — một bản dở
   dang không bao giờ mang tên của bản hoàn chỉnh; giữ tối đa **10** bản mới nhất; file tạm bỏ dở của
   một lần chạy hỏng trước đó (cũ hơn một giờ) được dọn; in đúng một dòng
   `db-backup: OK <đường dẫn file> <số byte> byte`. Lịch, `concurrencyPolicy`,
   `startingDeadlineSeconds`, giới hạn lịch sử, số lần thử lại của Lab 4b giữ nguyên.
3. Pod của Job backup chạy trên một worker **chọn có chủ đích**, ghim bằng `nodeSelector`; PV của
   `backup-store` nằm đúng trên node đó.
4. Bản dump sống sót khi Job và Pod của nó bị xóa.
5. Điều tra, xóa khi xong: Pod `thu-doc-backup` (không ghim node, mount `backup-store` **chỉ đọc**
   tại `/backup`, ngủ một giờ) phải tự được đặt lên node của PV và đọc được bản dump — kể cả khi một
   Job backup khác đang ghi cùng lúc; Pod `thu-sai-node` mount `backup-store` nhưng ghim sang worker
   **kia** phải đứng `Pending`.
6. `docs/tech-debt.md`: `TD-08` đổi thành đã trả, kèm bằng chứng; thêm `TD-10`.

**Ràng buộc:** không dùng `hostPath`. Không sửa, không xóa tay file trong PVC — Pod thử nghiệm chỉ
đọc. Chưa có PriorityClass `bookstore-batch` (7a). Tiền tố `db-backup: OK` là hợp đồng
([mục 4.2](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe)).

**Đọc lại:**

- Bài [92](../92-persistent-volumes-vi.md): [*Các chế độ truy cập*](../92-persistent-volumes-vi.md#access-modes)
  — câu về `ReadWriteOnce` và nhiều Pod cùng node;
  [*Dùng claim làm volume*](../92-persistent-volumes-vi.md#claims-as-volumes);
  [*Node Affinity*](../92-persistent-volumes-vi.md#node-affinity).
- Bài [96](../96-storage-classes-vi.md#volume-binding-mode) — PV được chọn hoặc cấp phát theo ràng
  buộc lập lịch của Pod **đầu tiên**.
- Bài [91](../91-volumes-vi.md#persistentvolumeclaim) — volume `persistentVolumeClaim`.
- Bài [94](../94-ephemeral-volumes-vi.md#vòng-đời-và-persistentvolumeclaim-lifecycle-and-persistentvolumeclaim)
  — vì sao volume tạm thời tổng quát cũng chết theo Pod.
- Bài [69](../69-cron-jobs-vi.md) và Lab 4b ticket `BS-4B-06` — chính sách của CronJob.

**Bàn giao:** `k8s/app/backup/pvc.yaml`; manifest CronJob `db-backup`; `k8s/app/kustomization.yaml`;
`~/lab-work/6a/thu-doc-backup.yaml`, `~/lab-work/6a/thu-sai-node.yaml`; `docs/adr/NNNN-backup-store.md`;
`docs/tech-debt.md`; `~/lab-evidence/6a/b4-*.txt`.

**Nghiệm thu:**

- `PASS:` PVC `backup-store` mang label `component=backup`, class `bookstore-retain`, `ReadWriteOnce`;
  CronJob ghi vào nó, không còn `emptyDir`.
- `PASS:` ngay sau `deploy.sh` trên cluster sạch, `backup-store` ở `Pending`.
- `PASS:` hai Job chạy tay từ CronJob đều in `db-backup: OK` với hai file khác nhau; Pod của chúng
  và PV của `backup-store` cùng nằm trên node trong `nodeSelector`; PV là `Retain`.
- `PASS:` xóa hai Job rồi, cả hai bản dump vẫn nằm trên PVC.
- `PASS:` `thu-doc-backup` tự lên node của PV và đọc được dump; một Job backup chạy thành công trong
  lúc nó đang mount PVC.
- `PASS:` sau hơn 10 lần chạy, PVC giữ đúng tối đa 10 bản `bookstore-*.sql` — đếm bằng file trên PVC,
  không bằng log.
- `PASS:` một lần chạy không kết nối được DB thì Job `Failed` và **không** sinh thêm file nào mang tên
  bản hoàn chỉnh.
- `PASS:` `thu-sai-node` `Pending`, lý do nhắc tới xung đột node affinity của volume; hai Pod thử
  nghiệm đã xóa.
- `PASS:` `docs/tech-debt.md` có `TD-08` đã trả và `TD-10`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-backup-store.md`:

1. Vì sao `backup-store` đứng `Pending` sau `deploy.sh` mà không phải lỗi? `deploy.sh` có nên chờ nó
   `Bound` không?
2. PV đã mang `nodeAffinity`, scheduler tự đặt Pod lên đúng node. Vậy vì sao vẫn ghim Job bằng
   `nodeSelector`? Nghĩ tới lần chạy **đầu tiên** trên một cluster sạch.
3. Đặt `backup-store` cùng node với `db` hay ở worker kia: mỗi cách được gì, mất gì khi một node chết
   hẳn? Viết `TD-10` bằng lời của bạn cho đúng lựa chọn đó.
4. `ReadWriteOnce` mà `thu-doc-backup` vẫn đọc được trong lúc Job đang ghi — vì sao? Thứ gì thật sự
   ngăn hai lần backup ghi chồng nhau: access mode hay chính sách của CronJob?
5. PVC xin 1Gi nhưng provisioner không cưỡng chế con số đó. Vì sao giữ tối đa 10 bản là bắt buộc?
   Vì sao volume tạm thời tổng quát (bài 94) không phải lời giải cho `TD-08`?

<details>
<summary>Gợi ý</summary>

1. Volume `persistentVolumeClaim` trong `jobTemplate.spec.template.spec.volumes`, mount như mọi
   volume khác. Chỉ đọc ở phía Pod thử nghiệm: `readOnly: true` trên `volumeMounts` hoặc trên
   `persistentVolumeClaim`.
2. Đổi tên nguyên tử trong cùng thư mục: ghi `<file>.tmp` rồi `mv`. Giữ 10 bản mới nhất bằng
   `ls -1t /backup/<mẫu tên> | tail -n +11 | xargs -r rm -f` — image `postgres` có GNU coreutils.
3. Chạy thử ngay: `kubectl -n bookstore create job <tên> --from=cronjob/db-backup`.
4. Node của PV: `{.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}`.
5. Lý do `Pending` của scheduler nằm trong `kubectl describe pod`, mục `Events`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
NS=bookstore; EV=~/lab-evidence/6a; W=~/lab-work/6a
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
chay() {   # tạo Job từ CronJob, chờ xong, in log của nó
  kubectl -n "$NS" create job "$1" --from=cronjob/db-backup >/dev/null
  kubectl -n "$NS" wait --for=condition=Complete "job/$1" --timeout=300s >/dev/null
  kubectl -n "$NS" logs "job/$1"
}

{ ./scripts/teardown.sh > "$EV/b4-teardown.txt" 2>&1 && ./scripts/deploy.sh > "$EV/b4-deploy.txt" 2>&1; } \
  && echo 'PASS: teardown roi deploy.sh thanh cong'
[ "$(jp pvc/backup-store '{.metadata.labels.app\.kubernetes\.io/component}|{.spec.storageClassName}|{.spec.accessModes[0]}')" = 'backup|bookstore-retain|ReadWriteOnce' ] \
  && echo 'PASS: PVC backup-store dung label, class, access mode'
CJV='{.spec.jobTemplate.spec.template.spec.volumes[*]'
{ jp cronjob/db-backup "$CJV.persistentVolumeClaim.claimName}" | grep -qw 'backup-store' && \
  [ -z "$(jp cronjob/db-backup "$CJV.emptyDir}")" ]; } && echo 'PASS: CronJob ghi vao backup-store, khong con emptyDir'
[ "$(jp pvc/backup-store '{.status.phase}')" = 'Pending' ] && echo 'PASS: sau deploy.sh, backup-store Pending vi chua co Pod tieu thu'

NODE="$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.nodeSelector.kubernetes\.io/hostname}')"
echo "node chon cho backup: ${NODE:-<chua ghim>}" > "$EV/b4-node.txt"
chay thu-backup-1 > "$EV/b4-backup-1.txt" 2>&1
sleep 2
chay thu-backup-2 > "$EV/b4-backup-2.txt" 2>&1
F1="$(awk '/^db-backup: OK/{print $3; exit}' "$EV/b4-backup-1.txt")"
F2="$(awk '/^db-backup: OK/{print $3; exit}' "$EV/b4-backup-2.txt")"
{ [ -n "$F1" ] && [ -n "$F2" ] && [ "$F1" != "$F2" ]; } && echo "PASS: hai lan chay in db-backup: OK ($F1, $F2)"
PN="$(kubectl -n "$NS" get pod -l batch.kubernetes.io/job-name=thu-backup-1 -o jsonpath='{.items[0].spec.nodeName}')"
PVB="$(jp pvc/backup-store '{.spec.volumeName}')"
PVN="$(kubectl get pv "$PVB" -o jsonpath='{.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}')"
{ [ -n "$NODE" ] && [ "$PN" = "$NODE" ] && [ "$PVN" = "$NODE" ] && \
  [ "$(kubectl get pv "$PVB" -o jsonpath='{.spec.persistentVolumeReclaimPolicy}')" = 'Retain' ]; } \
  && echo "PASS: Pod backup va PV backup-store (Retain) cung o $NODE"
kubectl -n "$NS" delete job thu-backup-1 thu-backup-2 --wait=true

kubectl apply -f "$W/thu-doc-backup.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/thu-doc-backup --timeout=180s
kubectl -n "$NS" exec thu-doc-backup -- ls -l /backup > "$EV/b4-ls.txt"
{ grep -q "$(basename "$F1")" "$EV/b4-ls.txt" && grep -q "$(basename "$F2")" "$EV/b4-ls.txt"; } \
  && echo 'PASS: xoa Job va Pod roi, hai ban dump van tren PVC'
[ "$(jp pod/thu-doc-backup '{.spec.nodeName}')" = "$NODE" ] && echo 'PASS: scheduler tu dat thu-doc-backup len node cua PV'
chay thu-backup-3 > "$EV/b4-backup-3.txt" 2>&1
grep -q '^db-backup: OK' "$EV/b4-backup-3.txt" \
  && echo 'PASS: Job ghi thanh cong trong luc Pod khac dang mount PVC tren cung node'
kubectl -n "$NS" delete job thu-backup-3 --wait=true

# Giữ tối đa 10 bản: chạy tuần tự thêm 9 lần (tổng 12), đếm bằng file trên PVC.
dem() { kubectl -n "$NS" exec thu-doc-backup -- sh -c 'ls -1 /backup/bookstore-*.sql 2>/dev/null | wc -l'; }
: > "$EV/b4-backup-nhieu.txt"
for i in $(seq 4 12); do chay "thu-backup-$i" >> "$EV/b4-backup-nhieu.txt" 2>&1; sleep 1; done
kubectl -n "$NS" get job -o name | grep '/thu-backup-' | xargs -r kubectl -n "$NS" delete --wait=true
N="$(dem)"; echo "so ban bookstore-*.sql tren PVC: $N" | tee "$EV/b4-retention.txt"
{ [ "$(grep -c '^db-backup: OK' "$EV/b4-backup-nhieu.txt")" -eq 9 ] && [ "$N" -eq 10 ]; } \
  && echo 'PASS: sau 12 lan chay PVC giu dung 10 ban'

# Một lần dump thất bại: PGHOST trỏ tới tên không tồn tại; Job phải Failed, không thêm bản hoàn chỉnh.
TRUOC="$(kubectl -n "$NS" exec thu-doc-backup -- ls -1 /backup | sort)"
kubectl -n "$NS" create job thu-backup-loi --from=cronjob/db-backup --dry-run=client -o yaml \
  | kubectl set env --local -f - PGHOST=khong-ton-tai.invalid -o yaml \
  | kubectl apply -f - >/dev/null
kubectl -n "$NS" wait --for=condition=Failed job/thu-backup-loi --timeout=300s >/dev/null \
  && echo 'PASS: Job backup khong ket noi duoc DB thi Failed'
SAU="$(kubectl -n "$NS" exec thu-doc-backup -- ls -1 /backup | sort)"
MOI="$(comm -13 <(echo "$TRUOC") <(echo "$SAU") | grep -E '^bookstore-.*\.sql$' || true)"
echo "file moi sau lan chay loi: [${MOI}]" | tee -a "$EV/b4-retention.txt"
[ -z "$MOI" ] && echo 'PASS: lan dump loi khong sinh file mang ten ban hoan chinh'
kubectl -n "$NS" delete job thu-backup-loi --wait=true

kubectl apply -f "$W/thu-sai-node.yaml"
sleep 15
kubectl -n "$NS" describe pod thu-sai-node > "$EV/b4-sai-node.txt" 2>&1
{ [ "$(jp pod/thu-sai-node '{.spec.nodeSelector.kubernetes\.io/hostname}')" != "$NODE" ] && \
  [ "$(jp pod/thu-sai-node '{.status.phase}')" = 'Pending' ] && \
  grep -qi 'volume node affinity conflict' "$EV/b4-sai-node.txt"; } \
  && echo 'PASS: Pod ghim sang node khac dung Pending vi xung dot node affinity cua volume'
kubectl -n "$NS" delete pod thu-doc-backup thu-sai-node --ignore-not-found --wait=true
[ -z "$(kubectl -n "$NS" get pod -o name | grep '/thu-')" ] && echo 'PASS: da xoa Pod thu nghiem'

{ grep -E 'TD-08.*đã trả' docs/tech-debt.md >/dev/null && grep -q 'TD-10' docs/tech-debt.md; } \
  && echo 'PASS: tech-debt.md: TD-08 da tra, co TD-10'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/backup/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backup-store
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: backup
    app.kubernetes.io/name: postgresql
spec:
  storageClassName: bookstore-retain
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
```

Thêm `- backup/pvc.yaml` vào mục `resources` của `k8s/app/kustomization.yaml`.

CronJob — chỉ phần `jobTemplate.spec.template.spec` thay đổi; phần đầu (lịch, `Forbid`,
`startingDeadlineSeconds`, giới hạn lịch sử, `backoffLimit`, `activeDeadlineSeconds`, label) giữ như
tag `lab-5b`. `PGHOST` giữ đúng giá trị ở tag `lab-5b` của bạn (từ Lab 5a là tên DNS của `db-0`).

```yaml
# k8s/app/jobs/db-backup.yaml — tên file như ở tag lab-5b; phần thay đổi ở Lab 6a
        spec:
          restartPolicy: Never
          nodeSelector:                       # cùng node với db và với PV backup-store (ADR backup-store)
            kubernetes.io/hostname: lab-k8s-worker2
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: backup-store         # trả TD-08; còn TD-10
          containers:
          - name: backup
            image: postgres:17.11
            env:
            - {name: PGHOST, value: "db-0.db"}
            - {name: PGUSER, value: postgres}
            - {name: PGDATABASE, value: bookstore}
            - {name: PGCONNECT_TIMEOUT, value: "5"}
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef: {name: db-credentials, key: postgres-password}
            command:
            - sh
            - -c
            - |
              set -e
              find /backup -maxdepth 1 -name 'bookstore-*.sql.tmp' -mmin +60 -delete
              F="/backup/bookstore-$(date +%Y%m%dT%H%M%S).sql"
              pg_dump --no-owner --file="$F.tmp"
              grep -q 'CREATE TABLE api.books' "$F.tmp"
              mv "$F.tmp" "$F"
              ls -1t /backup/bookstore-*.sql | tail -n +11 | xargs -r rm -f
              echo "db-backup: OK $F $(wc -c < "$F") byte"
            volumeMounts:
            - {name: backup, mountPath: /backup}
            resources:
              requests: {cpu: 50m, memory: 64Mi}
              limits: {cpu: 200m, memory: 128Mi}
```

```yaml
# ~/lab-work/6a/thu-doc-backup.yaml — thu-sai-node.yaml giống hệt, đổi tên và thêm
#   nodeSelector: {kubernetes.io/hostname: <worker KHÔNG chứa PV backup-store>}
apiVersion: v1
kind: Pod
metadata:
  name: thu-doc-backup
  namespace: bookstore
spec:
  terminationGracePeriodSeconds: 2
  containers:
  - name: doc
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
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

Dòng trong `docs/tech-debt.md`:

```markdown
| TD-08 | Bản dump của db-backup nằm trên emptyDir, mất cùng Pod | 4b | 6a | đã trả (6a) | ~/lab-evidence/6a/b4-ls.txt |
| TD-10 | Bản backup nằm trong cùng cluster — và cùng node lab-k8s-worker2 — với DB | 6a | 6b (chép ra ngoài bằng tay) | mở | docs/adr/NNNN-backup-store.md |
```

Ý chính cho ADR:

1. Class dùng `WaitForFirstConsumer`: không có Pod nào dùng PVC thì không có node, không có PV. CronJob
   chỉ tạo Pod theo lịch. `deploy.sh` chờ `Bound` sẽ treo tới lần backup đầu tiên — sai; đúng là để
   lần chạy đầu cấp volume.
2. Lần chạy **đầu tiên** quyết định PV nằm ở đâu: Pod đó lên worker nào thì thư mục được tạo ở đó, mãi
   mãi. Không ghim thì vị trí bản backup phụ thuộc may rủi của scheduler, và Lab 6b, 8a, 12 không biết
   tìm nó ở đâu. Ghim biến vị trí đó thành quyết định ghi trong repo.
3. Cùng node `lab-k8s-worker2`: mọi dữ liệu của dự án ở một chỗ, bảo trì một node (Lab 12), diễn tập
   chỉ trên node được phép phá — nhưng mất `lab-k8s-worker2` là mất cả DB lẫn backup. Ở worker kia: sống
   sót qua mất một node, đổi lại dữ liệu nằm rải hai node. Cả hai cách đều để bản backup **trong cùng
   cluster** — `TD-10`, chấp nhận có điều kiện ở Lab 6b bằng việc chép ra ngoài. Lời giải tham khảo chọn
   cùng node; chọn khác đều được nếu ADR nói rõ đánh đổi.
4. `ReadWriteOnce` là ràng buộc **theo node** (bài 92): mọi Pod trên node đó đều mount được. Thứ ngăn
   hai bản backup ghi chồng nhau là `concurrencyPolicy: Forbid` (Lab 4b) — trong phạm vi một CronJob;
   Job tạo tay bằng `--from` nằm ngoài chính sách đó, nên tên file theo thời điểm là lớp bảo vệ thứ hai.
5. Provisioner bỏ qua dung lượng xin trong PVC: không giới hạn số bản thì `backup-store` ăn dần đĩa
   `lab-k8s-worker2` — đĩa của chính `db`, của log và `emptyDir` (bài 95). Volume tạm thời tổng quát có
   PVC thật nhưng PVC đó thuộc về Pod và bị garbage collector xóa cùng Pod (bài 94) — đúng vòng đời mà
   `TD-08` cần thoát ra.

</details>

## B5. [BS-6A-05] Diễn tập: mất Pod, mất StatefulSet, mất PVC

**Loại:** Diễn tập sự cố — chỉ trên `lab-k8s-worker2`, nơi có `db-0` và dữ liệu của nó.

**Bối cảnh:** đội phải biết trước, bằng bằng chứng, hệ thống chịu được loại "xóa nhầm" nào và loại nào
cần người can thiệp — và có sẵn runbook khôi phục cho loại tệ nhất. Mục tiêu trong
[bảng chịu lỗi của dự án](DU-AN-BOOKSTORE.md#37-chịu-lỗi-và-vận-hành): xóa nhầm StatefulSet hoặc PVC
thì dữ liệu còn trên PV `Retain` và khôi phục được.

**Yêu cầu:** ghi một đơn hàng đánh dấu, rồi đi qua bốn kịch bản theo thứ tự. Sau mỗi kịch bản, đơn
hàng phải còn.

| Kịch bản | Thao tác | Phải chứng minh |
| --- | --- | --- |
| A — mất Pod | xóa `db-0` | Pod mới khác UID, gắn lại **đúng** PVC cũ (cùng UID PVC, cùng PV) |
| B — thu nhỏ | đưa StatefulSet `db` về 0 bản sao, rồi `deploy.sh` | khi 0 bản sao PVC vẫn `Bound`; `deploy.sh` đưa về 1 bản sao |
| C — mất StatefulSet | xóa StatefulSet `db`, rồi `deploy.sh` | không còn Pod, PVC vẫn nguyên; StatefulSet mới dùng lại PVC cũ |
| D — mất PVC | xóa PVC `data-db-0` khi `db-0` còn chạy, rồi xóa StatefulSet | PVC bị giữ ở `Terminating` khi còn Pod dùng; sau đó PVC mất, PV chuyển `Released`, dữ liệu còn trên node; một PVC mới chỉ đích danh PV đó vẫn không bind được |

Sau kịch bản D, **khôi phục** dữ liệu cũ mà không mất đơn hàng:

1. **Không** chạy `deploy.sh` ngay — trước hết giải thích trong postmortem vì sao.
2. Thu hồi thủ công theo bài 92: xóa object PV cũ, chứng minh dữ liệu vẫn nằm trên node.
3. Viết `~/lab-work/6a/pv-khoi-phuc.yaml`: PV tĩnh `data-db-0-khoi-phuc`, label dự án
   (`component: db`), trỏ lại đúng thư mục cũ trên `lab-k8s-worker2`, **dành riêng** cho PVC
   `bookstore/data-db-0`, cùng class và dung lượng với PV cũ, `Retain`. Khai báo sao cho nếu thư mục
   không tồn tại thì Pod **không** khởi động được, thay vì được một thư mục rỗng.
4. `deploy.sh`: PVC `data-db-0` mới bind vào PV tĩnh; entrypoint bỏ qua khởi tạo; đơn hàng còn; `api`
   đăng nhập được bằng mật khẩu trong `secrets/db.env`.
5. Viết `docs/runbook-khoi-phuc-pv.md` — quy trình khôi phục tổng quát kèm mẫu PV — và postmortem.

**Ràng buộc:** không buộc xóa Pod nào; không xóa namespace. Không sửa `claimRef` của PV cũ — quy trình
của ticket là PV tĩnh mới (cách sửa `claimRef` là câu hỏi thiết kế). Khối `nodeAffinity` của PV tĩnh
chép nguyên từ PV cũ, không tự viết. Không tự xóa thư mục trên node.

**Đọc lại:**

- Bài [92](../92-persistent-volumes-vi.md):
  [*Bảo vệ đối tượng lưu trữ đang được sử dụng*](../92-persistent-volumes-vi.md#bảo-vệ-đối-tượng-lưu-trữ-đang-được-sử-dụng-storage-object-in-use-protection),
  [*Thu hồi*](../92-persistent-volumes-vi.md#reclaiming) — mục *Retain*,
  [*Dành riêng một PersistentVolume*](../92-persistent-volumes-vi.md#dành-riêng-một-persistentvolume-reserving-a-persistentvolume),
  [*Pha*](../92-persistent-volumes-vi.md#pha-phase),
  [*Tên volume*](../92-persistent-volumes-vi.md#tên-volume-volume-name), mục *Khôi phục khi mở rộng
  volume thất bại* (bước 3).
- Bài [65](../65-statefulset-vi.md): [*Hạn chế*](../65-statefulset-vi.md#limitations),
  [*Lưu trữ ổn định*](../65-statefulset-vi.md#stable-storage),
  [*Lưu giữ PersistentVolumeClaim*](../65-statefulset-vi.md#persistentvolumeclaim-retention),
  [*Replicas*](../65-statefulset-vi.md#replicas).
- Bài [343](../343-run-replicated-stateful-application-vi.md): mục *Xóa các Pod* và ghi chú trong mục
  [*Scale số lượng replica*](../343-run-replicated-stateful-application-vi.md#scale-số-lượng-replica-scaling-the-number-of-replicas).
- Bài [344](../344-run-single-instance-stateful-vi.md#xóa-deployment-deleting-a-deployment) — PV cấp
  phát thủ công phải xóa thủ công.
- Bài [91](../91-volumes-vi.md#hostpath) — bảng các giá trị `type` của `hostPath`.

**Bàn giao:** `~/lab-work/6a/pv-khoi-phuc.yaml`; `docs/runbook-khoi-phuc-pv.md`;
`docs/postmortem/6a-mat-pvc.md`; `~/lab-evidence/6a/b5-*`.

**Nghiệm thu:**

- `PASS:` A — `db-0` mới khác UID, cùng PVC (UID không đổi), đơn hàng còn.
- `PASS:` B — 0 bản sao mà `data-db-0` vẫn `Bound`; `deploy.sh` đưa về 1, đơn hàng còn.
- `PASS:` C — StatefulSet mất, PVC còn nguyên UID; `deploy.sh` dựng lại trên PVC cũ, đơn hàng còn.
- `PASS:` D — PVC `Terminating` với finalizer `kubernetes.io/pvc-protection` trong khi `db-0` chạy;
  sau đó PVC mất, PV cũ `Released` vẫn giữ `claimRef` cũ, `PG_VERSION` còn trên node.
- `PASS:` D — PVC `thu-gianh-pv` chỉ đích danh PV cũ đứng `Pending`, PV vẫn `Released`; PVC đó đã xóa.
- `PASS:` khôi phục — xóa PV cũ, dữ liệu vẫn trên node; PV tĩnh `Available`, đã dành cho
  `bookstore/data-db-0`.
- `PASS:` khôi phục — sau `deploy.sh`: `data-db-0` bind vào PV tĩnh; log có dòng bỏ qua khởi tạo;
  đơn hàng còn; `/books` qua Service `api` trả dữ liệu.
- `PASS:` runbook và postmortem tồn tại.

**Câu hỏi thiết kế** — trả lời trong `docs/postmortem/6a-mat-pvc.md`:

1. Kịch bản nào hệ thống tự chịu được, kịch bản nào cần người? Với mỗi kịch bản, chỉ ra cơ chế giữ
   dữ liệu (StatefulSet, retention policy, finalizer, reclaim policy).
2. Ngay sau khi mất PVC, nếu bạn chạy `deploy.sh` thì chuyện gì xảy ra — với PVC, với PV cũ, với dữ
   liệu mà `api` phục vụ? Vì sao một kiểm thử khói chỉ kiểm hợp đồng, như `tests/smoke.sh`, **không**
   bắt được sự cố đó?
3. Vì sao PVC mới, kể cả trùng tên `data-db-0`, không tự bind lại vào PV `Released`? Bài 92 nêu cách
   nào khác để PV cũ thành `Available`, và so với PV tĩnh mới thì cách nào an toàn hơn cho dự án?
4. Vì sao PV khôi phục dùng `type: Directory` chứ không `DirectoryOrCreate` như PV do provisioner tạo?
5. Bước khôi phục dùng lại đúng `secrets/db.env` đang có. Nếu thư mục cũ đến từ một thời điểm mà bí
   mật đã khác, runbook phải thêm bước gì?

<details>
<summary>Gợi ý</summary>

- Chờ Pod mới thay Pod cũ: so `metadata.uid` trong một vòng lặp.
- Xóa PVC mà không chờ: `kubectl delete pvc ... --wait=false`; đọc `metadata.deletionTimestamp` và
  `metadata.finalizers`.
- Lưu PV cũ trước khi xóa: `kubectl get pv <tên> -o yaml > ~/lab-evidence/6a/b5-pv-cu.yaml`. Từ file
  đó lấy `hostPath.path`, `capacity`, khối `nodeAffinity`.
- PV dành riêng cho một PVC: `spec.claimRef` chỉ có `namespace` và `name` — không có `uid`.
- Bỏ khỏi bản chép: `status`, `metadata.uid`, `resourceVersion`, `creationTimestamp`, `finalizers`,
  annotation `pv.kubernetes.io/provisioned-by` — PV tĩnh không thuộc provisioner nào.
- Dòng mà entrypoint `postgres` in khi thư mục dữ liệu đã có sẵn: `Skipping initialization`.

</details>

<details>
<summary>Script nghiệm thu — phần 1 là cuộc diễn tập A→D; phần 2 chạy sau khi viết PV khôi phục</summary>

Phần 1:

```bash
NS=bookstore; EV=~/lab-evidence/6a
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
SQL() { kubectl -n "$NS" exec db-0 -c postgres -- psql -X -q -U postgres -d bookstore -tAc "$1"; }
cho_db() { kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=300s >/dev/null; }
cho_pod_moi() { for i in $(seq 1 90); do U="$(jp pod/db-0 '{.metadata.uid}')"; [ -n "$U" ] && [ "$U" != "$1" ] && return 0; sleep 2; done; return 1; }

cho_db
ID="$(SQL "INSERT INTO api.orders (book_id, quantity) SELECT min(id), 4 FROM api.books RETURNING id" | head -n1)"
echo "$ID" > "$EV/b5-don-hang.txt"
con_don() { [ "$(SQL "SELECT count(*) FROM api.orders WHERE id = ${ID}")" = '1' ]; }
PVC0="$(jp pvc/data-db-0 '{.metadata.uid}')"; PV0="$(jp pvc/data-db-0 '{.spec.volumeName}')"
echo "$PV0" > "$EV/b5-pv-cu-ten.txt"

# A. Mất Pod
U0="$(jp pod/db-0 '{.metadata.uid}')"
kubectl -n "$NS" delete pod db-0 --wait=true
cho_pod_moi "$U0" && cho_db
{ [ "$(jp pvc/data-db-0 '{.metadata.uid}')" = "$PVC0" ] && [ "$(jp pvc/data-db-0 '{.spec.volumeName}')" = "$PV0" ] && con_don; } \
  && echo 'PASS: A — db-0 moi khac UID, gan lai dung PVC cu, don hang con'

# B. Thu nhỏ về 0
kubectl -n "$NS" scale statefulset db --replicas=0
kubectl -n "$NS" wait --for=delete pod/db-0 --timeout=180s
[ "$(jp pvc/data-db-0 '{.status.phase}')" = 'Bound' ] && echo 'PASS: B — 0 ban sao, data-db-0 van Bound'
./scripts/deploy.sh > "$EV/b5-deploy-b.txt" 2>&1; cho_db
{ [ "$(jp statefulset/db '{.spec.replicas}')" = '1' ] && con_don; } && echo 'PASS: B — deploy.sh dua db ve 1 ban sao, don hang con'

# C. Mất StatefulSet
kubectl -n "$NS" delete statefulset db --wait=true
kubectl -n "$NS" wait --for=delete pod/db-0 --timeout=180s
{ ! kubectl -n "$NS" get statefulset db >/dev/null 2>&1 && [ "$(jp pvc/data-db-0 '{.metadata.uid}')" = "$PVC0" ] && \
  [ "$(jp pvc/data-db-0 '{.status.phase}')" = 'Bound' ]; } && echo 'PASS: C — StatefulSet mat, PVC data-db-0 con nguyen'
./scripts/deploy.sh > "$EV/b5-deploy-c.txt" 2>&1; cho_db
{ [ "$(jp pvc/data-db-0 '{.metadata.uid}')" = "$PVC0" ] && con_don; } && echo 'PASS: C — StatefulSet moi dung lai PVC cu, don hang con'

# D. Mất PVC
kubectl -n "$NS" delete pvc data-db-0 --wait=false
sleep 3
{ [ -n "$(jp pvc/data-db-0 '{.metadata.deletionTimestamp}')" ] && \
  jp pvc/data-db-0 '{.metadata.finalizers}' | grep -q 'kubernetes.io/pvc-protection' && \
  [ "$(jp pod/db-0 '{.status.phase}')" = 'Running' ]; } \
  && echo 'PASS: D — PVC dang duoc dung bi giu o Terminating (pvc-protection), db-0 van chay'
kubectl -n "$NS" delete statefulset db --wait=true
kubectl -n "$NS" wait --for=delete pod/db-0 --timeout=180s
for i in $(seq 1 60); do kubectl -n "$NS" get pvc data-db-0 >/dev/null 2>&1 || break; sleep 2; done
for i in $(seq 1 60); do [ "$(kubectl get pv "$PV0" -o jsonpath='{.status.phase}')" = 'Released' ] && break; sleep 2; done
kubectl get pv "$PV0" -o yaml > "$EV/b5-pv-cu.yaml"
DIR0="$(kubectl get pv "$PV0" -o jsonpath='{.spec.hostPath.path}{.spec.local.path}')"
echo "$DIR0" > "$EV/b5-thu-muc-cu.txt"
{ ! kubectl -n "$NS" get pvc data-db-0 >/dev/null 2>&1 && \
  [ "$(kubectl get pv "$PV0" -o jsonpath='{.status.phase}|{.spec.claimRef.name}')" = 'Released|data-db-0' ]; } \
  && echo 'PASS: D — PVC mat, PV cu Released, claimRef cu van giu cho'
ssh lab-k8s-worker2 "sudo find '$DIR0' -maxdepth 2 -name PG_VERSION" > "$EV/b5-tren-node.txt" 2>&1
grep -q 'PG_VERSION' "$EV/b5-tren-node.txt" && echo 'PASS: D — du lieu con nguyen tren lab-k8s-worker2'

CAP="$(kubectl get pv "$PV0" -o jsonpath='{.spec.capacity.storage}')"
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: thu-gianh-pv
  namespace: bookstore
spec:
  storageClassName: bookstore-retain
  volumeName: $PV0
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: $CAP
EOF
sleep 20
{ [ "$(jp pvc/thu-gianh-pv '{.status.phase}')" = 'Pending' ] && [ "$(kubectl get pv "$PV0" -o jsonpath='{.status.phase}')" = 'Released' ]; } \
  && echo 'PASS: D — PVC chi dich danh PV Released van Pending'
kubectl -n "$NS" delete pvc thu-gianh-pv --wait=true
```

Phần 2 — sau khi viết `~/lab-work/6a/pv-khoi-phuc.yaml`, **chưa** chạy `deploy.sh`:

```bash
NS=bookstore; EV=~/lab-evidence/6a
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
SQL() { kubectl -n "$NS" exec db-0 -c postgres -- psql -X -q -U postgres -d bookstore -tAc "$1"; }
ID="$(cat "$EV/b5-don-hang.txt")"; PV0="$(cat "$EV/b5-pv-cu-ten.txt")"; DIR0="$(cat "$EV/b5-thu-muc-cu.txt")"

kubectl delete pv "$PV0" --wait=true
ssh lab-k8s-worker2 "sudo find '$DIR0' -maxdepth 2 -name PG_VERSION" | grep -q 'PG_VERSION' \
  && echo 'PASS: xoa object PV cu, du lieu van tren node (Retain)'
kubectl apply -f ~/lab-work/6a/pv-khoi-phuc.yaml
sleep 5
{ [ "$(kubectl get pv data-db-0-khoi-phuc -o jsonpath='{.status.phase}')" = 'Available' ] && \
  [ "$(kubectl get pv data-db-0-khoi-phuc -o jsonpath='{.spec.claimRef.namespace}/{.spec.claimRef.name}')" = 'bookstore/data-db-0' ] && \
  [ "$(kubectl get pv data-db-0-khoi-phuc -o jsonpath='{.spec.hostPath.path}|{.spec.hostPath.type}')" = "$DIR0|Directory" ]; } \
  && echo 'PASS: PV tinh Available, tro dung thu muc cu, danh rieng cho bookstore/data-db-0'

./scripts/deploy.sh > "$EV/b5-deploy-khoi-phuc.txt" 2>&1 && echo 'PASS: deploy.sh thanh cong'
[ "$(jp pvc/data-db-0 '{.spec.volumeName}')" = 'data-db-0-khoi-phuc' ] && echo 'PASS: data-db-0 bind vao PV khoi phuc'
kubectl -n "$NS" logs db-0 -c postgres > "$EV/b5-db-log.txt" 2>&1
grep -q 'Skipping initialization' "$EV/b5-db-log.txt" && echo 'PASS: entrypoint bo qua khoi tao — dung du lieu cu'
[ "$(SQL "SELECT count(*) FROM api.orders WHERE id = ${ID}")" = '1' ] && echo "PASS: don hang $ID con sau khi khoi phuc"
WEB="$(kubectl -n "$NS" get pod -l app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=web --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')"
T="$(kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -qO- http://api:3000/books 2>/dev/null | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && echo "PASS: api dang nhap duoc bang secrets/db.env, /books tra $T cuon"
{ test -s docs/runbook-khoi-phuc-pv.md && test -s docs/postmortem/6a-mat-pvc.md; } && echo 'PASS: co runbook va postmortem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

PV khôi phục — dựng từ `~/lab-evidence/6a/b5-pv-cu.yaml`. Tên thư mục, dung lượng và khối
`nodeAffinity` dưới đây là ví dụ; lấy đúng giá trị trong file của bạn.

```yaml
# ~/lab-work/6a/pv-khoi-phuc.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-db-0-khoi-phuc
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: db
    app.kubernetes.io/name: postgresql
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: bookstore-retain
  claimRef:                                   # dành riêng — chỉ namespace và name, không uid
    namespace: bookstore
    name: data-db-0
  hostPath:
    path: /opt/local-path-provisioner/pvc-0f3c...-..._bookstore_data-db-0   # đúng chuỗi của PV cũ
    type: Directory                           # thư mục phải có sẵn; sai đường dẫn thì Pod không lên
  nodeAffinity:                               # chép nguyên từ PV cũ
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - lab-k8s-worker2
```

Khung `docs/runbook-khoi-phuc-pv.md`:

```markdown
# Runbook — khôi phục dữ liệu db từ PV Released

Dùng khi: PVC data-db-0 bị xóa, PV của nó ở pha Released (class bookstore-retain, Retain).

1. Dừng mọi thứ có thể tạo lại data-db-0: không chạy deploy.sh; StatefulSet db phải không tồn tại
   hoặc 0 bản sao.
2. Lưu PV cũ: kubectl get pv <PV> -o yaml > pv-cu.yaml. Ghi lại hostPath.path, capacity, nodeAffinity.
3. Kiểm dữ liệu trên node (chỉ đọc): ssh <node> "sudo find <path> -maxdepth 2 -name PG_VERSION".
4. Đối chiếu bí mật: secrets/db.env phải là bản đi cùng dữ liệu này (ADR mat-khau-du-lieu-ben).
5. Xóa object PV cũ (dữ liệu ở lại vì Retain).
6. Tạo PV tĩnh từ mẫu pv-khoi-phuc: cùng path (type Directory), cùng class, dung lượng, nodeAffinity;
   claimRef = bookstore/data-db-0 không uid; reclaim Retain; label dự án.
7. kubectl get pv -> Available. Rồi ./scripts/deploy.sh.
8. Kiểm: PVC bind vào PV tĩnh; log db-0 có "Skipping initialization"; ./tests/smoke.sh PASS.
```

Ý chính cho postmortem:

1. A, B, C tự chịu được: StatefulSet gắn PVC theo ordinal (A, C), `whenScaled`/`whenDeleted: Retain`
   giữ PVC khi thu nhỏ và khi xóa StatefulSet (B, C), và `apply` ghi đè `replicas` thủ công (B, bài 65
   *Replicas*). D cần người: `pvc-protection` chỉ **hoãn** việc xóa PVC tới khi không còn Pod dùng;
   `Retain` giữ dữ liệu, nhưng không ai tự nối nó lại.
2. Khi tạo `db-0`, StatefulSet tạo PVC còn thiếu từ `volumeClaimTemplates` — cùng cơ chế bài 343 mô
   tả khi scale lên → provisioner cấp PV mới, thư mục rỗng → entrypoint khởi tạo DB mới, `deploy.sh`
   chạy migration → `api` Ready, `/books` trả đủ sách. Hệ thống "khỏe" trong khi mọi đơn hàng nằm lại
   trên PV cũ. Kiểm thử khói kiểm hợp đồng (số sách tối thiểu, Pod Ready), không kiểm sự liên tục của
   dữ liệu. Vì thế bước 1 của runbook là **dừng**.
3. `claimRef` của PV cũ còn `uid` của PVC đã xóa; PVC mới trùng tên nhưng khác UID, nên PV không còn
   là "của" ai mới — bài 92: `Released` chưa sẵn sàng cho claim khác, chỉ định `volumeName` thì kẹt
   `Pending`. Cách khác: xóa `claimRef` khỏi PV cũ để nó thành `Available` (bài 92, khôi phục khi mở
   rộng thất bại, bước 3) — nhanh, nhưng sửa trực tiếp object do provisioner quản lý, và PV `Available`
   không `claimRef` có thể bị **bất kỳ** PVC khớp nào lấy mất. PV tĩnh có `claimRef` dành riêng thì chỉ
   `data-db-0` lấy được.
4. Bài 91: `DirectoryOrCreate` tạo thư mục rỗng nếu đường dẫn sai — PostgreSQL thấy thư mục rỗng sẽ
   khởi tạo DB mới, lặp lại đúng sự cố câu 2. `Directory` bắt thư mục phải tồn tại: sai một ký tự thì
   Pod đứng ở `ContainerCreating` với lỗi rõ ràng.
5. Mật khẩu nằm trong dữ liệu. Bí mật lệch thì phải tìm lại `secrets/db.env` đi cùng thời điểm đó, hoặc
   sau khi `db-0` lên, đặt lại mật khẩu hai role qua Unix socket rồi cập nhật `secrets/db.env` và chạy
   `deploy.sh` để `api`, `db-backup` nhận bí mật mới.

</details>

## B6. [BS-6A-06] Projected volume cho `status-writer`

**Loại:** Tính năng.

**Bối cảnh:** từ Lab 3a, sidecar `status-writer` đọc label của Pod qua volume `downwardAPI` riêng, còn
chu kỳ ghi là hằng số trong script. Đội muốn chu kỳ thành cấu hình (ConfigMap `web-config`), nhưng
không muốn mỗi nguồn dữ liệu mới lại thêm một volume và một điểm mount cho sidecar. Volume `projected`
gom nhiều nguồn vào **một** thư mục.

**Yêu cầu:**

1. `status-writer` đọc mọi đầu vào của nó từ **một** thư mục `/etc/status-writer`, do **một** volume
   cung cấp, gồm: file `labels` chứa label của Pod (downward API), và file `status-writer.env` lấy từ
   một key mới cùng tên của ConfigMap `web-config`, nội dung `STATUS_INTERVAL=5`. Script đọc chu kỳ ghi
   từ file đó.
2. Bỏ hẳn volume `downwardAPI` riêng của Lab 3a. Không tạo ConfigMap mới — tên ConfigMap là hợp đồng
   ([mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label)).
3. Mọi file trong thư mục là chỉ đọc cho mọi người (`0444`), đặt **một lần** cho cả volume.
4. Hành vi cũ giữ nguyên: `/status.json` đủ bốn trường; đổi label `app.kubernetes.io/version` trên một
   Pod `web` đang chạy thì `/status.json` **của Pod đó** đổi theo mà không container nào khởi động lại;
   `nginx` vẫn chỉ thấy `default.conf` trong `/etc/nginx/conf.d`.
5. Điều tra, không vào repo: (a) chứng minh Pod `web` còn mang một volume projected khác mà bạn không
   viết, liệt kê các nguồn của nó; (b) bằng server dry-run có kiểm tra trường chặt, chứng minh hai bẫy
   cú pháp của bài 93 — khai Secret trong projected bằng `secretName`, và đặt `defaultMode` bên trong
   một nguồn — đều bị API server từ chối.

**Ràng buộc:** không đổi tên container, đường dẫn hay trường của `/status.json`
([mục 4.2](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe)). Không mount nguồn
projected bằng `subPath`. ConfigMap của dự án được sinh bằng `configMapGenerator` từ Lab 5a — thêm key
bằng chính cơ chế đó.

**Đọc lại:**

- Bài [93](../93-projected-volumes-vi.md): [*Giới thiệu*](../93-projected-volumes-vi.md#giới-thiệu-introduction),
  [ví dụ secret, downwardAPI, configMap](../93-projected-volumes-vi.md#example-configuration-secret-downwardapi-configmap),
  [ví dụ quyền không mặc định](../93-projected-volumes-vi.md#example-configuration-secrets-nondefault-permission-mode)
  và đoạn hai ngoại lệ ngay sau nó, [*serviceAccountToken*](../93-projected-volumes-vi.md#serviceaccounttoken)
  cùng ghi chú `subPath` cuối mục.
- Bài [277](../277-configure-projected-volume-vi.md#cấu-hình-một-projected-volume-cho-pod-configure-a-projected-volume-for-a-pod)
  — một `volumes`, một `volumeMounts`.
- Bài [91](../91-volumes-vi.md#configmap) — `configMap` và [`downwardAPI`](../91-volumes-vi.md#downwardapi).
- Lab 3b, ticket `BS-3B-04` — vì sao `nginx` mount `web-config` qua `items`.

**Bàn giao:** manifest Deployment `web`; `k8s/app/web/status-writer.env`; `k8s/app/kustomization.yaml`;
`~/lab-work/6a/thu-projected-sai.yaml`, `~/lab-work/6a/thu-projected-mode.yaml`;
`docs/adr/NNNN-projected-status-writer.md`; `~/lab-evidence/6a/b6-*.txt`.

**Nghiệm thu:**

- `PASS:` Pod `web` có một volume projected gom nguồn downward API và ConfigMap `web-config`,
  `defaultMode` `0444`; `status-writer` mount nó tại `/etc/status-writer`; không còn volume
  `downwardAPI` riêng.
- `PASS:` trong `status-writer`, `/etc/status-writer` có `labels` và `status-writer.env` (quyền 444),
  file sau có `STATUS_INTERVAL`.
- `PASS:` `/status.json` đủ bốn trường và được cập nhật theo thời gian.
- `PASS:` đổi label version trên một Pod `web` thì `/status.json` của nó đổi; `restartCount` không đổi.
- `PASS:` `nginx` chỉ thấy `default.conf` trong `/etc/nginx/conf.d`.
- `PASS:` volume `kube-api-access-*` là projected với ba nguồn `serviceAccountToken`, `configMap`,
  `downwardAPI`.
- `PASS:` hai dry-run sai cú pháp bị từ chối, lỗi nhắc đúng trường sai.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-projected-status-writer.md`:

1. Trước và sau ticket, sidecar có bao nhiêu volume, bao nhiêu điểm mount cho đầu vào? Ràng buộc nào
   của bài 93 **không** đổi khi gom nguồn — và nó có ý nghĩa gì nếu sau này sidecar cần dữ liệu của
   namespace `bookstore-system`?
2. Bạn viết `defaultMode: 0444` trong YAML, nhưng `kubectl get -o jsonpath` in ra một con số khác.
   Con số nào, vì sao? Hai khác biệt cú pháp nào của projected so với volume rời?
3. Đổi label trên Pod: file `labels` cập nhật tại chỗ. Đổi `status-writer.env` trong repo rồi
   `deploy.sh`: chuyện gì xảy ra với Pod `web`, và vì sao khác với volume ConfigMap không hậu tố hash ở
   điều tra của Lab 3b?
4. Volume projected `kube-api-access-*` do ai thêm, gồm những gì, để làm gì? `status-writer` có cần nó
   không — và sprint nào xử lý chuyện đó?
5. Nếu mount riêng `status-writer.env` bằng `subPath` cho gọn, điều gì hỏng?

<details>
<summary>Gợi ý</summary>

1. Volume `projected` có `defaultMode` và danh sách `sources`; mỗi phần tử của `sources` là đúng một
   loại: `downwardAPI`, `configMap`, `secret`, `serviceAccountToken`...
2. Trong `configMapGenerator` của `web-config`, thêm file `web/status-writer.env` vào `files` — tên
   file thành tên key. Kustomize tự đổi `name: web-config` bên trong `projected.sources` sang tên có
   hậu tố hash.
3. Trong script: `. /etc/status-writer/status-writer.env` rồi `sleep "$STATUS_INTERVAL" & wait $!`.
4. File trong volume projected là symlink vào `..data/`; đọc quyền của file thật bằng `stat -L -c %a`.
5. JSON không có số bát phân: `0444` (bát phân) là `292` (thập phân).
6. Dry-run kiểm trường chặt: `kubectl create --dry-run=server --validate=strict -f <file>`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong và đã viết hai file dry-run</summary>

```bash
NS=bookstore; EV=~/lab-evidence/6a; W=~/lab-work/6a
cd ~/bookstore
./scripts/deploy.sh > "$EV/b6-deploy.txt" 2>&1 && echo 'PASS: deploy.sh thanh cong'
WEB="$(kubectl -n "$NS" get pod -l app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=web \
       --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')"
pj() { kubectl -n "$NS" get pod "$WEB" -o jsonpath="$1"; }
sw() { kubectl -n "$NS" exec "$WEB" -c status-writer -- "$@"; }

pj '{range .spec.volumes[*]}{.name}|{.projected.sources[*].downwardAPI.items[*].path}|{.projected.sources[*].configMap.name}|{.projected.defaultMode}|{.downwardAPI.items[*].path}{"\n"}{end}' \
  | grep -v '^kube-api-access-' > "$EV/b6-volumes.txt"
PROJ="$(awk -F'|' '$2!="" && $3 ~ /^web-config/ {print $1; exit}' "$EV/b6-volumes.txt")"
{ [ -n "$PROJ" ] && [ "$(awk -F'|' -v v="$PROJ" '$1==v {print $4}' "$EV/b6-volumes.txt")" = '292' ]; } \
  && echo "PASS: volume $PROJ gom downwardAPI va web-config, defaultMode 0444"
[ -z "$(awk -F'|' '$5!=""' "$EV/b6-volumes.txt")" ] && echo 'PASS: khong con volume downwardAPI rieng'
[ "$(pj "{.spec.initContainers[?(@.name==\"status-writer\")].volumeMounts[?(@.name==\"$PROJ\")].mountPath}")" = '/etc/status-writer' ] \
  && echo 'PASS: status-writer mount volume projected tai /etc/status-writer'

{ [ "$(sw stat -L -c %a /etc/status-writer/labels)" = '444' ] && \
  [ "$(sw stat -L -c %a /etc/status-writer/status-writer.env)" = '444' ] && \
  sw grep -q '^STATUS_INTERVAL=' /etc/status-writer/status-writer.env; } \
  && echo 'PASS: hai file tu hai nguon trong mot thu muc, quyen 444, co STATUS_INTERVAL'

get() { sw wget -qO- "http://127.0.0.1:8080$1"; }
S1="$(get /status.json)"; sleep 12; S2="$(get /status.json)"
echo "$S2" > "$EV/b6-status.txt"
BAD=0
for K in pod node version updated; do echo "$S2" | grep -q "\"$K\"" || BAD=1; done
{ [ "$BAD" -eq 0 ] && [ "$S1" != "$S2" ]; } && echo 'PASS: status.json du bon truong va duoc cap nhat'

V0="$(pj '{.metadata.labels.app\.kubernetes\.io/version}')"
R0="$(pj '{.status.containerStatuses[*].restartCount} {.status.initContainerStatuses[*].restartCount}')"
kubectl -n "$NS" label pod "$WEB" app.kubernetes.io/version=9.9.9-thu --overwrite
for i in $(seq 1 36); do get /status.json | grep -q '"version" *: *"9.9.9-thu"' && break; sleep 5; done
get /status.json | grep -q '"version" *: *"9.9.9-thu"' && echo 'PASS: status.json nhan version moi qua volume projected'
[ "$(pj '{.status.containerStatuses[*].restartCount} {.status.initContainerStatuses[*].restartCount}')" = "$R0" ] \
  && echo 'PASS: khong container nao khoi dong lai'
kubectl -n "$NS" label pod "$WEB" "app.kubernetes.io/version=$V0" --overwrite

[ "$(kubectl -n "$NS" exec "$WEB" -c nginx -- ls /etc/nginx/conf.d)" = 'default.conf' ] \
  && echo 'PASS: nginx van chi thay default.conf'

KA="$(pj '{range .spec.volumes[*]}{.name}|{.projected.sources[*].serviceAccountToken.path}|{.projected.sources[*].configMap.name}|{.projected.sources[*].downwardAPI.items[*].path}{"\n"}{end}' | grep '^kube-api-access-')"
echo "$KA" > "$EV/b6-kube-api-access.txt"
echo "$KA" | awk -F'|' '$2!="" && $3!="" && $4!="" {ok=1} END {exit !ok}' \
  && echo 'PASS: kube-api-access-* la projected: serviceAccountToken, configMap, downwardAPI'

kubectl create --dry-run=server --validate=strict -f "$W/thu-projected-sai.yaml" > "$EV/b6-sai-secretname.txt" 2>&1 \
  && echo 'FAIL: API server chap nhan secretName' \
  || { grep -q 'secretName' "$EV/b6-sai-secretname.txt" && echo 'PASS: secretName trong projected bi tu choi'; }
kubectl create --dry-run=server --validate=strict -f "$W/thu-projected-mode.yaml" > "$EV/b6-sai-defaultmode.txt" 2>&1 \
  && echo 'FAIL: API server chap nhan defaultMode trong mot nguon' \
  || { grep -q 'defaultMode' "$EV/b6-sai-defaultmode.txt" && echo 'PASS: defaultMode trong mot nguon bi tu choi'; }
```

Thời gian file trong volume projected nhận label mới phụ thuộc chu kỳ đồng bộ của kubelet; vòng chờ ở
trên cho tối đa ba phút.

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
printf 'STATUS_INTERVAL=5\n' > ~/bookstore/k8s/app/web/status-writer.env
```

Trong `k8s/app/kustomization.yaml`, mục `configMapGenerator` của `web-config` — thêm một dòng vào
`files`, các dòng khác giữ như tag `lab-5b`:

```yaml
configMapGenerator:
- name: web-config
  files:
  - web/default.conf
  - index.html.tpl=web/index.template.html   # như Lab 5a: key giữ nguyên, file nguồn không đuôi .tpl
  - web/status-writer.env      # Lab 6a — chu kỳ ghi của status-writer
```

Deployment `web` — chỉ phần thay đổi; `render-html`, `nginx`, `securityContext`, `hostUsers`,
`preStop`, tài nguyên giữ như tag `lab-5b`:

```yaml
      volumes:
      # - name: podinfo               <- XÓA: volume downwardAPI riêng của Lab 3a
      - name: status-src
        projected:
          defaultMode: 0444            # chỉ đặt được ở cấp projected
          sources:
          - downwardAPI:
              items:
              - path: labels
                fieldRef:
                  fieldPath: metadata.labels
          - configMap:
              name: web-config         # Kustomize đổi sang tên có hậu tố hash
              items:
              - key: status-writer.env
                path: status-writer.env
      initContainers:
      - name: status-writer
        # image, restartPolicy: Always, env POD_NAME/NODE_NAME, tài nguyên: giữ nguyên
        command:
        - sh
        - -c
        - |
          trap 'echo "status-writer: nhan TERM"; exit 0' TERM
          while true; do
            . /etc/status-writer/status-writer.env
            V="$(sed -n 's/^app\.kubernetes\.io\/version="\(.*\)"$/\1/p' /etc/status-writer/labels)"
            # ... phần ghi /html/status.json giữ như tag lab-5b (kể cả phần đọc node-agent từ Lab 5a)
            sleep "${STATUS_INTERVAL:-5}" & wait $!
          done
        volumeMounts:
        - name: html
          mountPath: /html
        - name: status-src
          mountPath: /etc/status-writer
          readOnly: true
```

```yaml
# ~/lab-work/6a/thu-projected-sai.yaml — chỉ dùng cho dry-run
apiVersion: v1
kind: Pod
metadata: {name: thu-projected-sai, namespace: bookstore}
spec:
  containers:
  - {name: app, image: busybox:1.37, command: ["sleep", "60"], volumeMounts: [{name: gom, mountPath: /gom}]}
  volumes:
  - name: gom
    projected:
      sources:
      - secret:
          secretName: db-credentials   # sai: trong projected là "name"
```

`thu-projected-mode.yaml` giống hệt, đổi tên Pod thành `thu-projected-mode` và nguồn thành
`configMap: {name: web-config, defaultMode: 0400}` — `defaultMode` đặt nhầm vào trong một nguồn.

Ý chính cho ADR:

1. Trước: hai volume (`html`, `podinfo`) và thêm một nữa nếu đọc ConfigMap riêng; sau: `html` cộng
   **một** volume đầu vào. Không đổi: mọi nguồn phải cùng namespace với Pod (bài 93) — sidecar không bao
   giờ chiếu được ConfigMap của `bookstore-system`; dữ liệu đó phải đi qua mạng, như `/node.json`.
2. `292` — API lưu số nguyên, JSON không có ký pháp bát phân; `0444` bát phân bằng `292` thập phân.
   Hai khác biệt: Secret trong projected dùng `name` thay `secretName`; `defaultMode` chỉ ở cấp
   projected, từng item dùng `mode`.
3. `configMapGenerator` sinh tên mới khi nội dung đổi → template của Deployment đổi → rollout, Pod mới
   đọc giá trị mới ngay từ đầu. Lab 3b đo ConfigMap tên cố định: file trong volume được kubelet cập
   nhật tại chỗ nhưng tiến trình phải tự đọc lại. Với sidecar đọc lại file mỗi vòng lặp thì cả hai cách
   đều chạy; hash cho thêm lịch sử và khả năng rollback.
4. Admission ServiceAccount thêm vào mọi Pod: `serviceAccountToken` (token có hạn), `configMap`
   `kube-root-ca.crt` (CA của cluster), `downwardAPI` (namespace) — đủ để gọi API server. `status-writer`
   không gọi API; tắt automount là việc của Lab 9a (`automountServiceAccountToken: false`).
5. Bài 93: container mount nguồn projected qua `subPath` không nhận cập nhật — `labels` đứng yên và yêu
   cầu 4 hỏng.

</details>

## B7. [BS-6A-07] Lưu trữ tạm thời cục bộ: trần cho `emptyDir` và trục xuất

**Loại:** Điều tra — Pod vượt giới hạn chỉ chạy trên `lab-k8s-worker2` — kèm một thay đổi nhỏ trong
repo.

**Bối cảnh:** `emptyDir` của `web` (`html`) và của `node-agent` (`www`) không có trần. Một vòng lặp
ghi hỏng là đủ để ăn dần đĩa node — và từ B3, đĩa đó cũng chứa thư mục dữ liệu của `db`. Bài 95 nói
kubelet đo được và trục xuất Pod vượt giới hạn; đội cần thấy điều đó trên chính worker của mình trước
khi dựa vào nó.

**Yêu cầu:**

1. Đọc `ephemeral-storage` mà hai worker công bố (capacity, allocatable). Chứng minh trên
   `lab-k8s-worker2`, `/var/lib/kubelet`, `/var/log` và `/opt/local-path-provisioner` nằm trên **cùng
   một** filesystem.
2. Pod `thu-qua-suc` xin `ephemeral-storage` lớn hơn allocatable của mọi node → không lập lịch được,
   lý do nhắc tới `ephemeral-storage`.
3. Pod `thu-sizelimit` trên `lab-k8s-worker2`: `emptyDir` trần 32Mi, container ghi 64Mi vào đó → bị
   trục xuất, thông điệp nhắc tới `EmptyDir`.
4. Pod `thu-gioi-han` trên `lab-k8s-worker2`: container khai `requests`/`limits` `ephemeral-storage`
   16Mi/64Mi, `emptyDir` không trần, ghi 128Mi → bị trục xuất, thông điệp nhắc tới giới hạn lưu trữ tạm
   thời cục bộ.
5. Pod `thu-tmpfs`: `emptyDir` dạng bộ nhớ, trần 16Mi → được mount bằng `tmpfs`.
6. Repo: mọi `emptyDir` của workload dự án có `sizeLimit` hợp lý; `deploy.sh` thành công.

**Ràng buộc:** mỗi Pod ghi tối đa 128Mi; không chỉnh cấu hình kubelet; xóa Pod thử nghiệm khi xong.
Không thêm `ephemeral-storage` vào `requests`/`limits` của container dự án ở sprint này — hạn mức tài
nguyên theo namespace thuộc Lab 7b.

**Đọc lại:**

- Bài [95](../95-ephemeral-storage-vi.md): [*Các cấu hình*](../95-ephemeral-storage-vi.md#configurations),
  [*Thiết lập request và limit*](../95-ephemeral-storage-vi.md#requests-limits),
  [*Cách các Pod có request ephemeral-storage được lập lịch*](../95-ephemeral-storage-vi.md#cách-các-pod-có-request-ephemeral-storage-được-lập-lịch-how-pods-with-ephemeral-storage-requests-are-scheduled),
  [*Quản lý mức tiêu thụ*](../95-ephemeral-storage-vi.md#resource-emphemeralstorage-consumption) kể cả
  khối *Thận trọng*.
- Bài [91](../91-volumes-vi.md#emptydir) — `sizeLimit`, `medium: Memory`; mục
  [*Tài nguyên*](../91-volumes-vi.md#tài-nguyên-resources).
- Bài [110](../110-manage-resources-containers-vi.md) (Lab 3c) — cú pháp `requests`/`limits`.

**Bàn giao:** Deployment `web`, DaemonSet `node-agent`; `~/lab-work/6a/thu-{qua-suc,sizelimit,gioi-han,tmpfs}.yaml`;
`docs/adr/NNNN-luu-tru-tam-thoi.md`; `~/lab-evidence/6a/b7-*.txt`.

**Nghiệm thu:**

- `PASS:` `lab-k8s-worker2` công bố `ephemeral-storage`, allocatable nhỏ hơn capacity; ba thư mục nằm
  trên cùng một filesystem.
- `PASS:` `thu-qua-suc` `Pending`, `Unschedulable`, lý do nhắc `ephemeral-storage`.
- `PASS:` `thu-sizelimit` `Failed`/`Evicted` trên `lab-k8s-worker2`, thông điệp nhắc `EmptyDir`.
- `PASS:` `thu-gioi-han` `Failed`/`Evicted`, thông điệp nhắc `ephemeral local storage`.
- `PASS:` `thu-tmpfs` mount `/ram` bằng `tmpfs`.
- `PASS:` không còn `emptyDir` nào của Pod dự án thiếu `sizeLimit`; Pod thử nghiệm đã xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-luu-tru-tam-thoi.md`:

1. Worker của bạn theo bố cục nào của bài 95, và kubelet có đo được không? Một `emptyDir` chạy loạn ăn
   vào đâu — kể cả vào thứ gì của `db` kể từ B3?
2. `thu-sizelimit` và `thu-gioi-han` bị trục xuất với hai thông điệp khác nhau. Mỗi cái ứng với giới hạn
   nào? `thu-gioi-han` ghi vào `emptyDir` chứ không ghi vào lớp ghi được của container — nó bị tính ở
   cấp container hay cấp Pod, vì sao?
3. `requests` và `limits` của `ephemeral-storage` mỗi cái tác động lúc nào? Một đồng nghiệp gõ
   `requests.ephemeral-storage: 400m` — đó là bao nhiêu?
4. `thu-tmpfs` ghi vào `/ram` bị tính vào đâu? Một `emptyDir` `medium: Memory` không trần trên Pod
   `web` có thể gây hậu quả gì?
5. Vì sao dự án chọn `sizeLimit` ngay bây giờ, còn `ephemeral-storage` trong `limits` thì để Lab 7b? Nếu
   một ngày kubelet không đo được (bố cục khác), `sizeLimit` còn bảo vệ được gì — đọc khối *Thận trọng*
   của bài 95.

<details>
<summary>Gợi ý</summary>

1. `kubectl get node <tên> -o jsonpath='{.status.capacity.ephemeral-storage}'`, tương tự cho
   `allocatable`.
2. Mount point chứa một đường dẫn: `df -P <đường dẫn>`, cột cuối dòng thứ hai. Không cần `sudo`.
3. Ghi dữ liệu: `dd if=/dev/zero of=/scratch/x bs=1M count=64`, rồi `sleep 3600` để container còn sống
   chờ kubelet đo.
4. Pod bị trục xuất: `.status.phase` là `Failed`, `.status.reason` là `Evicted`, thông điệp ở
   `.status.message`. Kubelet đo theo chu kỳ — chờ theo điều kiện.
5. Lý do không lập lịch được nằm ở condition `PodScheduled`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã viết bốn Pod thử nghiệm và sửa repo</summary>

```bash
NS=bookstore; EV=~/lab-evidence/6a; W=~/lab-work/6a
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
cho_truc_xuat() { for i in $(seq 1 150); do [ "$(jp "pod/$1" '{.status.reason}')" = 'Evicted' ] && return 0; sleep 2; done; return 1; }

kubectl get nodes -o custom-columns=NODE:.metadata.name,EPH_CAP:.status.capacity.ephemeral-storage,EPH_ALLOC:.status.allocatable.ephemeral-storage \
  > "$EV/b7-allocatable.txt"
CAP="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.capacity.ephemeral-storage}')"
ALLOC="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.allocatable.ephemeral-storage}')"
{ [ -n "$CAP" ] && [ -n "$ALLOC" ] && [ "$CAP" != "$ALLOC" ]; } && echo "PASS: worker2 cong bo ephemeral-storage $CAP, allocatable $ALLOC"
ssh lab-k8s-worker2 'for D in /var/lib/kubelet /var/log /opt/local-path-provisioner; do df -P "$D" | awk "NR==2 {print \$6}"; done' \
  > "$EV/b7-filesystem.txt"
[ "$(sort -u "$EV/b7-filesystem.txt" | wc -l)" -eq 1 ] && echo "PASS: ba thu muc cung filesystem $(head -n1 "$EV/b7-filesystem.txt")"

kubectl apply -f "$W/thu-qua-suc.yaml"
sleep 10
{ [ "$(jp pod/thu-qua-suc '{.status.phase}')" = 'Pending' ] && \
  [ "$(jp pod/thu-qua-suc '{.status.conditions[?(@.type=="PodScheduled")].reason}')" = 'Unschedulable' ] && \
  jp pod/thu-qua-suc '{.status.conditions[?(@.type=="PodScheduled")].message}' | grep -q 'ephemeral-storage'; } \
  && echo 'PASS: xin qua allocatable thi khong lap lich duoc'

kubectl apply -f "$W/thu-sizelimit.yaml" -f "$W/thu-gioi-han.yaml" -f "$W/thu-tmpfs.yaml"
cho_truc_xuat thu-sizelimit
jp pod/thu-sizelimit '{.spec.nodeName}|{.status.phase}|{.status.reason}|{.status.message}' > "$EV/b7-sizelimit.txt"
{ grep -q '^lab-k8s-worker2|Failed|Evicted|' "$EV/b7-sizelimit.txt" && grep -q 'EmptyDir' "$EV/b7-sizelimit.txt"; } \
  && echo 'PASS: vuot sizeLimit cua emptyDir thi bi truc xuat'
cho_truc_xuat thu-gioi-han
jp pod/thu-gioi-han '{.spec.nodeName}|{.status.phase}|{.status.reason}|{.status.message}' > "$EV/b7-gioi-han.txt"
{ grep -q '^lab-k8s-worker2|Failed|Evicted|' "$EV/b7-gioi-han.txt" && grep -qi 'ephemeral local storage' "$EV/b7-gioi-han.txt"; } \
  && echo 'PASS: vuot limits.ephemeral-storage thi bi truc xuat'
kubectl -n "$NS" wait --for=condition=Ready pod/thu-tmpfs --timeout=120s
kubectl -n "$NS" exec thu-tmpfs -- sh -c 'mount | grep " /ram "' > "$EV/b7-tmpfs.txt" 2>&1
grep -q 'tmpfs' "$EV/b7-tmpfs.txt" && echo 'PASS: emptyDir medium Memory la tmpfs'
kubectl -n "$NS" delete pod thu-qua-suc thu-sizelimit thu-gioi-han thu-tmpfs --ignore-not-found --wait=true

./scripts/deploy.sh > "$EV/b7-deploy.txt" 2>&1 && echo 'PASS: deploy.sh thanh cong'
kubectl -n "$NS" get pods -l app.kubernetes.io/part-of=bookstore \
  -o jsonpath='{range .items[*]}{range .spec.volumes[*]}{.name}={.emptyDir}{"\n"}{end}{end}' > "$EV/b7-emptydir.txt"
[ -z "$(grep '={}$' "$EV/b7-emptydir.txt")" ] && echo 'PASS: moi emptyDir cua Pod du an co sizeLimit'
[ -z "$(kubectl -n "$NS" get pod -o name | grep '/thu-')" ] && echo 'PASS: da xoa Pod thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/6a/thu-sizelimit.yaml
apiVersion: v1
kind: Pod
metadata:
  name: thu-sizelimit
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  restartPolicy: Never
  terminationGracePeriodSeconds: 2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "dd if=/dev/zero of=/scratch/x bs=1M count=64; sleep 3600"]
    volumeMounts:
    - {name: scratch, mountPath: /scratch}
  volumes:
  - name: scratch
    emptyDir:
      sizeLimit: 32Mi
```

`thu-gioi-han.yaml` giống `thu-sizelimit.yaml`, chỉ khác: tên Pod; `count=128`; volume là
`emptyDir: {}`; container thêm
`resources: {requests: {ephemeral-storage: 16Mi}, limits: {ephemeral-storage: 64Mi}}`.
`thu-qua-suc.yaml`: một container `busybox` `sleep 3600` với `requests.ephemeral-storage: 1Ti`, không
ghim node. `thu-tmpfs.yaml`: một container `busybox` `sleep 600` mount `/ram` từ
`emptyDir: {medium: Memory, sizeLimit: 16Mi}`.

Thay đổi trong repo — cấp `spec.template.spec.volumes`:

```yaml
# Deployment web
      - name: html
        emptyDir:
          sizeLimit: 16Mi      # index.html, healthz, status.json: vài KB
# DaemonSet node-agent
      - name: www
        emptyDir:
          sizeLimit: 1Mi       # node.json: vài trăm byte
```

Ý chính cho ADR:

1. Bố cục *một filesystem duy nhất*: `/var/lib/kubelet` và `/var/log` trên filesystem gốc, đúng thiết kế
   của kubelet — đo được. `/opt/local-path-provisioner` cũng nằm trên đó: `emptyDir` chạy loạn ăn chung
   đĩa với log, image, lớp ghi được **và dữ liệu của `db`**.
2. `thu-sizelimit`: `Usage of EmptyDir volume ... exceeds the limit` — trần riêng của volume. `thu-gioi-han`:
   tổng mức dùng của Pod vượt tổng limit của các container — **cách ly cấp Pod**, vì `emptyDir` chỉ được
   cộng ở cấp Pod; cách ly cấp container chỉ tính lớp ghi được và log của container đó (bài 95).
3. `requests` dùng lúc lập lịch — tổng phải vừa allocatable (`thu-qua-suc`); `limits` dùng lúc chạy —
   vượt là bị trục xuất. `400m` là 0.4 byte; người gõ muốn `400Mi`.
4. `tmpfs` tính vào **bộ nhớ** của container (bài 91, 95). Không trần thì volume lớn tới allocatable
   memory của node; trên `web` (limit 64Mi) ghi nhiều vào đó là `OOMKilled`, không phải trục xuất.
5. `sizeLimit` là trần theo đúng nhu cầu của từng volume, không ảnh hưởng lập lịch, không đụng tới hạn
   mức; `limits.ephemeral-storage` cho mọi container nên đi cùng LimitRange và ResourceQuota ở 7b (bài
   95: quota `ephemeral-storage` chỉ áp dụng khi Pod khai limit). Kubelet không đo được thì không trục
   xuất vì vượt giới hạn; khi đĩa xuống thấp, node tự taint và trục xuất mọi Pod không tolerate — cơ chế
   của Lab 7a.

</details>

## B8. [BS-6A-08] Điều tra: vì sao dữ liệu không đi qua `hostPath`

**Loại:** Điều tra — chỉ đọc, Pod thử nghiệm trên cả hai worker.

**Bối cảnh:** một đề xuất trong đội: "local-path cũng chỉ là một thư mục trên node — sao `db` không
mount thẳng `hostPath` cho gọn?". Trước khi bác đề xuất, đội cần bằng chứng về rủi ro, đo trên chính
dữ liệu của dự án.

**Yêu cầu:**

1. Pod `thu-hostpath-w2` ghim `lab-k8s-worker2`, mount thư mục `/opt` của node **chỉ đọc**, không khai gì
   liên quan tới DB. Chứng minh từ trong Pod đọc được `PG_VERSION` của dữ liệu `db` — tức đọc được file
   dữ liệu mà không qua bất kỳ xác thực nào của PostgreSQL.
2. Ghi thử vào mount đó thất bại.
3. Pod `thu-hostpath-w1`: **cùng** manifest, chỉ khác tên và node → không thấy dữ liệu `db`.
4. Xóa hai Pod; ghi ADR.

**Ràng buộc:** chỉ đọc. Không mount `/`, `/var/lib/kubelet`, socket của container runtime; không
`privileged`. Không đọc file dữ liệu nào ngoài `PG_VERSION`.

**Đọc lại:**

- Bài [91](../91-volumes-vi.md#hostpath) — khối *Cảnh báo* và bảng `type`; mục
  [*Mount chỉ đọc*](../91-volumes-vi.md#mount-chỉ-đọc-read-only-mounts).
- Bài [92](../92-persistent-volumes-vi.md#các-loại-persistent-volume-types-of-persistent-volumes) —
  dòng `hostPath`: chỉ để thử trên một node.
- Bài [91](../91-volumes-vi.md#local) — `local` cùng `nodeAffinity` là lối thay thế.
- [Mục 3.5 của file dự án](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật) — Pod Security Standard của từng
  namespace.

**Bàn giao:** `~/lab-work/6a/thu-hostpath-w1.yaml`, `~/lab-work/6a/thu-hostpath-w2.yaml`;
`docs/adr/NNNN-hostpath.md`; `~/lab-evidence/6a/b8-*.txt`.

**Nghiệm thu:**

- `PASS:` `thu-hostpath-w2` mount chỉ đọc và đọc được `PG_VERSION` của `db`.
- `PASS:` ghi vào mount thất bại.
- `PASS:` `thu-hostpath-w1` không thấy thư mục dữ liệu `db`.
- `PASS:` hai Pod đã xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-hostpath.md`:

1. Ai trong cluster làm được điều bạn vừa làm? Từ Lab 9b, cái gì chặn nó trong namespace `bookstore`, và
   vì sao `db` dùng PVC thì vẫn chạy được?
2. PV của local-path bên dưới cũng là `hostPath`. Khác biệt giữa "Pod tự khai `hostPath`" và "Pod dùng
   PVC mà PV là `hostPath`" nằm ở đâu: ai viết đường dẫn, ai biết node, ai có quyền tạo object đó?
3. Mount chỉ đọc chặn được việc ghi. Vì sao bài 91 vẫn coi `hostPath` chỉ đọc là nguy hiểm — bằng chứng
   nào ở bước 1?
4. Bước 3 minh họa đúng gạch đầu dòng nào trong cảnh báo của bài 91, và câu nào của bài 92?

<details>
<summary>Gợi ý</summary>

- Volume `hostPath` với `path: /opt`, `type: Directory`; `volumeMounts` có `readOnly: true`.
- Thư mục dữ liệu `db` có tên `<PV>_bookstore_data-db-0` dưới `local-path-provisioner`, và `PG_VERSION`
  nằm trong thư mục con `PGDATA` của nó. Dùng `sh -c` để shell trong container mở rộng mẫu `*`.
- Container `busybox` mặc định chạy root, có `CAP_DAC_OVERRIDE` — quyền `0700` của thư mục dữ liệu không
  cản nó.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã viết hai manifest</summary>

```bash
NS=bookstore; EV=~/lab-evidence/6a; W=~/lab-work/6a
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
kubectl apply -f "$W/thu-hostpath-w2.yaml" -f "$W/thu-hostpath-w1.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/thu-hostpath-w2 pod/thu-hostpath-w1 --timeout=180s
MP="$(jp pod/thu-hostpath-w2 '{.spec.containers[0].volumeMounts[0].mountPath}')"
{ [ "$(jp pod/thu-hostpath-w2 '{.spec.volumes[0].hostPath.path}')" = '/opt' ] && \
  [ "$(jp pod/thu-hostpath-w2 '{.spec.containers[0].volumeMounts[0].readOnly}')" = 'true' ] && \
  [ "$(jp pod/thu-hostpath-w2 '{.spec.nodeName}')" = 'lab-k8s-worker2' ]; } \
  && echo "PASS: thu-hostpath-w2 mount /opt chi doc tai $MP tren lab-k8s-worker2"

kubectl -n "$NS" exec thu-hostpath-w2 -- sh -c "cat $MP/local-path-provisioner/*_bookstore_data-db-0/*/PG_VERSION" \
  > "$EV/b8-w2.txt" 2>&1
grep -Eqx '[0-9]+' "$EV/b8-w2.txt" && echo "PASS: doc duoc PG_VERSION cua db ($(cat "$EV/b8-w2.txt")) ma khong qua PostgreSQL"
kubectl -n "$NS" exec thu-hostpath-w2 -- touch "$MP/thu-ghi" > /dev/null 2>&1 \
  && echo 'FAIL: ghi duoc vao /opt cua node' || echo 'PASS: mount chi doc, ghi that bai'

{ [ "$(jp pod/thu-hostpath-w1 '{.spec.nodeName}')" = 'lab-k8s-worker1' ] && \
  [ "$(jp pod/thu-hostpath-w1 '{.spec.volumes[0].hostPath.path}')" = '/opt' ]; } \
  && kubectl -n "$NS" exec thu-hostpath-w1 -- sh -c "ls -d $MP/local-path-provisioner/*_bookstore_data-db-0 2>/dev/null" \
     > "$EV/b8-w1.txt"
[ ! -s "$EV/b8-w1.txt" ] && echo 'PASS: cung manifest tren lab-k8s-worker1 khong thay du lieu db'

kubectl -n "$NS" delete pod thu-hostpath-w1 thu-hostpath-w2 --wait=true
[ -z "$(kubectl -n "$NS" get pod -o name | grep '/thu-hostpath')" ] && echo 'PASS: da xoa hai Pod'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/6a/thu-hostpath-w2.yaml — bản -w1 chỉ đổi tên và giá trị nodeSelector
apiVersion: v1
kind: Pod
metadata:
  name: thu-hostpath-w2
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  terminationGracePeriodSeconds: 2
  containers:
  - name: doc
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: opt-cua-node
      mountPath: /host-opt
      readOnly: true
  volumes:
  - name: opt-cua-node
    hostPath:
      path: /opt
      type: Directory
```

Ý chính cho ADR:

1. Bất kỳ ai tạo được Pod trong namespace — không cần quyền gì trên PV, PVC hay Secret. Từ Lab 9b,
   namespace `bookstore` chạy Pod Security Standard `restricted`, vốn cấm volume `hostPath`; volume
   `persistentVolumeClaim` thì được phép, nên `db` vẫn chạy.
2. Với PVC, đường dẫn do **provisioner** sinh và ghi vào PV — object cấp cluster mà người dùng ứng dụng
   không cần quyền tạo; PV mang `nodeAffinity` nên scheduler tự đưa Pod về đúng node; vòng đời thư mục
   theo reclaim policy. Pod tự khai `hostPath` thì tự chọn bất kỳ đường dẫn nào trên node, không ai
   kiểm.
3. Đọc thôi cũng đủ lộ: credential của kubelet, khóa, và — như bước 1 — dữ liệu của workload khác, bỏ
   qua mọi cơ chế xác thực của ứng dụng. Bài 91 còn nói mount chỉ đọc mặc định không đệ quy: một
   filesystem khác mount bên dưới vẫn có thể ghi được.
4. Gạch đầu dòng thứ hai: Pod cùng PodTemplate hành xử khác nhau trên các node khác nhau vì file trên mỗi
   node khác nhau. Bài 92: PV `hostPath` chỉ dành cho kiểm thử trên một node, sẽ không hoạt động trong
   cluster nhiều node — local-path sửa đúng chỗ đó bằng `nodeAffinity` trên PV.

</details>

## B9. [BS-6A-09] ⭐ Điều tra: volume tạm thời tổng quát

**Loại:** Điều tra — tùy chọn.

**Bối cảnh:** báo cáo tồn kho song song của Lab 4b có thể cần chỗ đệm lớn hơn bộ nhớ, có kích thước cố
định và tự biến mất khi Job xong. Bài 94 mô tả một loại volume khai inline như `emptyDir` nhưng sinh
PVC thật. Trước khi đưa vào thiết kế, kiểm nó hành xử thế nào trên provisioner của cluster.

**Yêu cầu:**

1. Pod `thu-eph` có volume tạm thời tổng quát tên `scratch`, class `local-path`, 64Mi. Chứng minh: PVC
   `thu-eph-scratch` xuất hiện, `Bound`, chủ sở hữu là Pod; một thư mục `*_bookstore_thu-eph-scratch`
   xuất hiện trên node của Pod.
2. Xóa Pod: PVC, PV và thư mục trên node đều biến mất.
3. Tạo tay PVC `thu-eph2-scratch` trước, rồi tạo Pod `thu-eph2` có volume `scratch`: PVC tạo tay không
   bị chiếm, Pod không chạy được. Xóa cả hai.

**Đọc lại:** bài [94](../94-ephemeral-volumes-vi.md#generic-ephemeral-volumes) — *Volume tạm thời tổng
quát*, [*Vòng đời và PersistentVolumeClaim*](../94-ephemeral-volumes-vi.md#vòng-đời-và-persistentvolumeclaim-lifecycle-and-persistentvolumeclaim),
[*Đặt tên PersistentVolumeClaim*](../94-ephemeral-volumes-vi.md#đặt-tên-persistentvolumeclaim-persistentvolumeclaim-naming),
[*Bảo mật*](../94-ephemeral-volumes-vi.md#bảo-mật-security).

**Nghiệm thu:**

- `PASS:` PVC `thu-eph-scratch` `Bound`, `ownerReferences` là Pod `thu-eph`; thư mục có trên node.
- `PASS:` xóa Pod thì PVC, PV, thư mục đều mất.
- `PASS:` PVC tạo tay không có `ownerReferences`; `thu-eph2` không `Running`; cả hai đã xóa.

<details>
<summary>Script nghiệm thu và lời giải tham khảo</summary>

```yaml
# ~/lab-work/6a/thu-eph.yaml — thu-eph2.yaml chỉ đổi tên Pod
apiVersion: v1
kind: Pod
metadata:
  name: thu-eph
  namespace: bookstore
spec:
  terminationGracePeriodSeconds: 2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - {name: scratch, mountPath: /scratch}
  volumes:
  - name: scratch
    ephemeral:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 64Mi
```

```bash
NS=bookstore; EV=~/lab-evidence/6a; W=~/lab-work/6a
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
kubectl apply -f "$W/thu-eph.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/thu-eph --timeout=240s
N="$(jp pod/thu-eph '{.spec.nodeName}')"; PV="$(jp pvc/thu-eph-scratch '{.spec.volumeName}')"
{ [ "$(jp pvc/thu-eph-scratch '{.status.phase}|{.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].name}')" = 'Bound|Pod/thu-eph' ] && \
  ssh "$N" "sudo sh -c 'ls -d /opt/local-path-provisioner/*_bookstore_thu-eph-scratch'" > "$EV/b9-thu-muc.txt" 2>&1; } \
  && echo "PASS: PVC <Pod>-<volume> do controller tao, Pod la chu, thu muc tren $N"
kubectl -n "$NS" delete pod thu-eph --wait=true
for i in $(seq 1 60); do { kubectl -n "$NS" get pvc thu-eph-scratch || kubectl get pv "$PV"; } >/dev/null 2>&1 || break; sleep 2; done
# Thư mục trên node: CON, MAT, hoặc rỗng khi ssh/sudo lỗi — rỗng không phải "đã mất".
TM="$(ssh -o BatchMode=yes "$N" "sudo -n true && if sudo sh -c 'ls -d /opt/local-path-provisioner/*_bookstore_thu-eph-scratch' >/dev/null 2>&1; then echo CON; else echo MAT; fi")"
[ -n "$TM" ] || echo "STOP: khong kiem duoc $N (ssh hoac sudo loi) — chua ket luan"
{ ! kubectl -n "$NS" get pvc thu-eph-scratch >/dev/null 2>&1 && ! kubectl get pv "$PV" >/dev/null 2>&1 && \
  [ "$TM" = MAT ]; } \
  && echo 'PASS: xoa Pod thi PVC, PV va thu muc tren node deu mat'

kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: thu-eph2-scratch
  namespace: bookstore
spec:
  storageClassName: local-path
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 64Mi
EOF
sed 's/name: thu-eph$/name: thu-eph2/' "$W/thu-eph.yaml" > "$W/thu-eph2.yaml"
kubectl apply -f "$W/thu-eph2.yaml"
sleep 30
kubectl -n "$NS" describe pod thu-eph2 > "$EV/b9-xung-dot.txt" 2>&1
{ [ -z "$(jp pvc/thu-eph2-scratch '{.metadata.ownerReferences}')" ] && [ "$(jp pod/thu-eph2 '{.status.phase}')" != 'Running' ]; } \
  && echo 'PASS: PVC co san khong bi chiem, Pod khong chay duoc'
kubectl -n "$NS" delete pod thu-eph2 --wait=true
kubectl -n "$NS" delete pvc thu-eph2-scratch --wait=true
```

Ý chính: volume tạm thời tổng quát cho chỗ đệm có kích thước do provisioner quyết định — với
local-path thì con số chỉ là nhãn, vì provisioner không cưỡng chế dung lượng; thư mục nằm dưới
`/opt/local-path-provisioner` chứ không dưới `/var/lib/kubelet`, nên kubelet **không** tính nó vào
lưu trữ tạm thời của Pod như `emptyDir`. Tên PVC xác định nên dễ xung đột. Ai tạo được Pod là gián tiếp
tạo được PVC (bài 94, *Bảo mật*) — điều Lab 9a, 9b phải tính tới. Với cluster này, `emptyDir` có
`sizeLimit` là chỗ đệm tốt hơn cho Job của dự án.

</details>

## B10. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 8, gỡ **toàn bộ** dự án kể
cả dữ liệu trên node, rồi chụp mốc `03-storage-ready` chỉ chứa hạ tầng: mọi thứ của `02-net-ready`,
cộng namespace `local-path-storage` đang chạy provisioner, cộng StorageClass `local-path` được đánh
dấu mặc định. Không workload, không PV, không PVC, không thư mục dữ liệu.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 6a — kiểm thử khói **tích lũy**, chép nguyên văn:

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 6a: dữ liệu bền cho db và bản backup, cộng mọi hợp đồng còn
# hiệu lực từ sprint 1 tới sprint 7 (controller, tài nguyên, Service, NetworkPolicy, Ingress TLS).
# Tích lũy: giữ mọi kiểm tra của smoke Lab 5b, thêm phần của sprint 8. Script không in bí mật.
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

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

Smoke tạo ba object tạm thời — Pod `smoke-np-*` trong namespace thử `bookstore-thu` (tạo khi chưa có,
mang label dự án, xóa lại nếu smoke tự tạo); Job `smoke-backup-*` và Pod `smoke-doc-backup-*` trong
`bookstore` — và tự xóa chúng; hai Pod không mang label dự án nên không lẫn vào các kiểm tra theo label. Bản dump do smoke tạo nằm lại trên `backup-store`, chịu cùng giới hạn 10 bản.

2. Cập nhật `docs/tech-debt.md`: `TD-04` đổi thành đã trả (6a, bằng chứng
   `~/lab-evidence/6a/b1-don-hang.txt` và `b5-db-log.txt`); `TD-08` đã trả và `TD-10` đã thêm ở B4.

3. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/6a/b10-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/6a/b10-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/6a/b10-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-6a: sprint 8 — du lieu ben cho db va backup'
git tag lab-6a
git bundle create ~/bookstore-lab-6a.bundle --all && git bundle verify ~/bookstore-lab-6a.bundle
```

Chép `~/bookstore-lab-6a.bundle` về máy host bằng `scp` trước khi đi tiếp. `secrets/` không nằm trong
bundle — đúng chủ đích: sau teardown không còn dữ liệu nào mà bí mật cũ phải khớp.

5. Dọn object thử nghiệm và trả cluster về hạ tầng thuần:

```bash
kubectl -n bookstore delete pod,pvc,job -l '!app.kubernetes.io/part-of' --ignore-not-found --wait=true
kubectl delete storageclass thu-immediate --ignore-not-found
rm -rf ~/lab-work/6a
cd ~/bookstore && ./scripts/teardown.sh
```

6. Gate cuối — repo, dự án, tầng lưu trữ:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
cd ~/bookstore
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-6a' && echo 'PASS: HEAD mang tag lab-6a'
test -s ~/bookstore-lab-6a.bundle && echo 'PASS: co bundle'
{ test -f k8s/cluster/storageclass-retain.yaml && test -f k8s/app/backup/pvc.yaml && \
  grep -q 'volumeClaimTemplates' k8s/app/db/statefulset.yaml && test -f k8s/app/web/status-writer.env && \
  test -f platform/local-path-provisioner/README.md && test -f platform/local-path-provisioner/local-path-storage.yaml; } \
  && echo 'PASS: bo cuc repo dung muc 5.5'
{ grep -E 'TD-04.*đã trả' docs/tech-debt.md && grep -E 'TD-08.*đã trả' docs/tech-debt.md && grep -q 'TD-10' docs/tech-debt.md; } >/dev/null \
  && echo 'PASS: tech-debt.md cap nhat TD-04, TD-08, TD-10'
BAD=0
for F in 'adr/*-local-path-provisioner.md' 'adr/*-du-lieu-ben-db.md' 'adr/*-mat-khau-du-lieu-ben.md' 'adr/*-backup-store.md' \
         'adr/*-projected-status-writer.md' 'adr/*-luu-tru-tam-thoi.md' 'adr/*-hostpath.md' \
         'postmortem/6a-du-lieu-emptydir.md' 'postmortem/6a-mat-pvc.md' 'runbook-khoi-phuc-pv.md'; do
  ls docs/$F >/dev/null 2>&1 || { echo "FAIL: thieu docs/$F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du ADR, postmortem va runbook cua sprint 8'

{ [ "$(sl kubectl get namespace,priorityclass,storageclass -l app.kubernetes.io/part-of=bookstore --no-headers)" -eq 0 ] && \
  [ "$(sl kubectl get all -A -l app.kubernetes.io/part-of=bookstore --no-headers)" -eq 0 ]; } \
  && echo 'PASS: khong con object nao cua du an'
{ [ "$(sl kubectl get pv --no-headers)" -eq 0 ] && [ "$(sl kubectl get pvc -A --no-headers)" -eq 0 ]; } \
  && echo 'PASS: khong con PV, PVC nao trong cluster'
SCS="$(kubectl get storageclass -o jsonpath='{range .items[*]}{.metadata.name}={.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}')"
{ [ "$SCS" = 'local-path=true' ] && \
  [ "$(kubectl -n local-path-storage get deploy local-path-provisioner -o jsonpath='{.status.readyReplicas}')" -ge 1 ]; } \
  && echo 'PASS: dung mot StorageClass local-path mac dinh, provisioner dang chay'
# Liệt kê /opt/local-path-provisioner trên worker. Thư mục gốc chưa từng có thì in rỗng, thoát 0;
# lỗi ssh, sudo hay find thì thoát khác 0 — "không đọc được" không bao giờ được hiểu thành "rỗng".
lp_ls() {
  ssh -o BatchMode=yes "$1" \
    'sudo -n test ! -e /opt/local-path-provisioner || sudo -n find /opt/local-path-provisioner -mindepth 1 -maxdepth 1'
}
for N in lab-k8s-worker1 lab-k8s-worker2; do
  if ! L="$(lp_ls "$N")"; then echo "STOP: khong kiem duoc $N (ssh, sudo hoac find loi) — khong chup snapshot"
  elif [ -z "$L" ]; then echo "PASS: $N khong con thu muc volume nao"
  else echo "FAIL: $N con $(printf '%s\n' "$L" | grep -c .) muc trong /opt/local-path-provisioner"; fi
done
test ! -e ~/lab-work/6a && echo 'PASS: da don lab-work'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
{
  echo "=== $(date -Is) — tang luu tru sau Lab 6a ==="
  kubectl get storageclass -o wide; kubectl get pv; kubectl get pvc -A
  kubectl -n local-path-storage get all
} > ~/lab-evidence/6a/b10-sau.txt 2>&1
```

**PASS:** `smoke exit=0` và file smoke ở bước 3 in toàn `PASS:`; mười hai dòng `PASS:` của gate cuối
(hai dòng cho hai worker), không dòng `FAIL:`; dòng `PASS: readyz ok`; ba node `Ready`; lệnh field selector trả
`No resources found`; CoreDNS đủ replica; namespace `default` không có Pod. Thư mục
`~/lab-evidence/6a/` **giữ lại**.

7. Lab này đổi hạ tầng, nên **không** dừng ở gate ngắn. Chạy **toàn bộ bảy tầng** của
   [A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready) theo thứ tự từ dưới lên:

| Tầng | Nội dung | Điều chỉnh cho mốc `03-storage-ready` |
| --- | --- | --- |
| 0 | prereq OS trên cả ba node | không đổi |
| 1 | control plane khỏe thật | không đổi |
| 2 | node, condition, taint và PodCIDR | không đổi |
| 3 | Pod networking xuyên node | CNI là Calico do Lab 5b cài, không phải Flannel |
| 4 | DNS trong cluster và ra Internet | không đổi |
| 5 | Service, EndpointSlice và kube-proxy | không đổi |
| 6 | đường control plane → kubelet | không đổi |

Tập namespace hệ thống hợp lệ ở mốc này: `kube-system`, namespace của Calico và `traefik` do Lab 5b
cài, **cộng** `local-path-storage`.

**PASS:** bảy tầng của A5.4 đều PASS; mọi Pod của `local-path-storage` ở `Running`.

8. Tắt máy và chụp `03-storage-ready`. Chụp khi VM đã tắt để snapshot không kèm trạng thái RAM — cùng lý
   do như A5.4.8 của Lab 00. Chạy trên **từng node** theo thứ tự worker 2 → worker 1 → master:

```bash
sudo shutdown -h now
```

Chờ VMware Workstation hiển thị cả ba VM ở trạng thái *Powered off*. Chụp trên **cả ba VM**: chuột
phải VM → **Snapshot → Take Snapshot** → ô *Name* điền đúng nguyên văn:

```text
03-storage-ready
```

Ô *Description* ghi lab đã dựng mốc này và ngày chụp, ví dụ
`dựng bằng LAB-6A-PV-PVC-VA-STORAGECLASS.md, chụp <ngày>`. Quy tắc tên giống hệt Lab 00: đúng nguyên văn
trên cả ba VM, không hậu tố theo VM, không thêm ngày, không đổi hoa thường. **Giữ nguyên snapshot
`02-net-ready`** — đó vẫn là điểm quay lui của cả hai mốc.

Verify từ PowerShell trên máy host:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$vmx = @(
  'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
  'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
  'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
)
foreach ($f in $vmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if (($names -ccontains '03-storage-ready') -and ($names -ccontains '02-net-ready')) { "PASS: $f" }
  else { "FAIL: $f -> $($names -join ', ')" }
}
```

**PASS:** đúng ba dòng `PASS:`, không dòng `FAIL:` nào. Lab 6a kết thúc ở đây; để ba VM tắt, lab sau
tự bật theo A5.5.

---

## 3. Checkpoint 6a

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Trước sprint này, dữ liệu `db` sống qua sự cố nào, chết theo sự cố nào? Thư mục của nó nằm ở đâu
      trên `lab-k8s-worker2`, và vì sao `db-0` mới cùng tên vẫn nhận một thư mục rỗng?
- [ ] Trong cấp phát động của dự án, ai là quản trị viên, ai là người dùng? Hai điều kiện để có class
      mặc định là gì, bạn chứng minh điều kiện thứ hai bằng hành vi nào? `thu-khong-class` và
      `thu-chuoi-rong` rẽ hai hướng ra sao?
- [ ] Kể chuỗi sự kiện từ lúc `deploy.sh` apply StatefulSet `db` tới lúc thư mục
      `<PV>_bookstore_data-db-0` xuất hiện trên `lab-k8s-worker2`: ai tạo PVC, vì sao PVC chờ, ai chọn
      node, ai tạo thư mục, PV mang những trường nào bạn chưa từng viết. **Nợ lab #2 — đã trả.**
- [ ] Vì sao `db` dùng `bookstore-retain` chứ không dùng class mặc định? `thu-immediate` hỏng ra sao,
      và `WaitForFirstConsumer` sửa đúng điều gì?
- [ ] Vì sao `PGDATA` là thư mục con của volume, và vì sao không dùng `subPath`?
- [ ] Dữ liệu bền buộc mật khẩu vào dữ liệu thế nào? Khi nào `deploy.sh` sinh bí mật mới, khi nào nó từ
      chối, và bạn làm gì khi mất `secrets/db.env` mà dữ liệu còn?
- [ ] `backup-store` đứng `Pending` sau `deploy.sh` vì sao? Vì sao ghim Job backup vào một node, và
      `ReadWriteOnce` cho phép `thu-doc-backup` đọc cùng lúc Job ghi vì sao? `TD-10` là gì?
- [ ] Bốn kịch bản của B5: cái nào tự lành nhờ cơ chế nào, cái nào cần người? Nếu chạy `deploy.sh` ngay
      sau khi mất PVC thì sao? Kể quy trình khôi phục, và vì sao PV tĩnh dùng `claimRef` không `uid` và
      `type: Directory`.
- [ ] Projected volume của `status-writer`: hai khác biệt cú pháp so với volume rời, ràng buộc namespace,
      và volume `kube-api-access-*` gồm những gì?
- [ ] `thu-sizelimit` và `thu-gioi-han` bị trục xuất vì hai giới hạn nào, ai đo, ai trục xuất? `tmpfs`
      tính vào đâu? `400m` ephemeral-storage là bao nhiêu?
- [ ] Vì sao `db` không mount thẳng `hostPath`? Bước nào của B8 chứng minh rủi ro, cái gì chặn nó từ Lab
      9b, và PV `hostPath` của local-path khác ở đâu?
- [ ] `teardown.sh` dọn những gì, theo thứ tự nào, và vì sao mốc `03-storage-ready` phải được chụp sau
      nó?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- Nợ lab #1 (VPA — chưa có lab trả; HPA ở Lab 11b), #5 (snapshot và nhân bản volume — Lab 6b, cần CSI
  driver), #6, #7, #8 (Phần II của lộ trình). Phần replication MySQL của bài 343 không thực hành (mục
  1.1). **Nợ lab #2 đã trả** ở B3.
- Nợ kỹ thuật: `TD-06` (`db` khởi động bằng root — Lab 9b), `TD-07` (quyền debug — Lab 9a), `TD-10`
  mới (bản backup cùng cluster với DB — chấp nhận có điều kiện ở Lab 6b). **`TD-04` và `TD-08` đã trả.**

### Bài giải thích cuối cùng

Trong vài phút, kể lại cuộc đời của một đơn hàng trong hệ thống của bạn: nó được ghi vào đâu — từ bảng
`api.orders` xuống tận thư mục nào trên node nào; nó sống sót thế nào khi `db-0` bị xóa, khi StatefulSet
bị thu về 0 rồi bị xóa; nó nằm ở đâu trong ba mươi phút sau khi một bản backup chạy. Rồi kể kịch bản tệ
nhất của B5 — PVC bị xóa: object nào biến mất, object nào ở lại ở pha nào, vì sao không ai tự nối dữ liệu
lại, và bạn nối lại bằng những bước nào mà không làm lệch mật khẩu. Kết thúc bằng việc chỉ ra Lab 6b
(bản dump ra ngoài cluster), Lab 9b (`db` chạy uid 999 trên chính `PGDATA` này) và Lab 12 (bảo trì
`lab-k8s-worker2`) sẽ dựa vào những gì bạn vừa dựng.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: `SC_N` khác 0 | Cluster đã chạy Lab 6a dở dang | Restore `02-net-ready` cho cả ba VM rồi bắt đầu lại |
| B0: `sudo -n true` fail trên worker | `sudo` của user lab đòi mật khẩu | Sự cố môi trường; `teardown.sh` từ lab này cần `sudo` không hỏi mật khẩu qua `ssh`. Sửa theo quy ước quản trị của bạn trước khi đi tiếp |
| B2: Deployment `local-path-provisioner` không Ready | Không kéo được image, hoặc manifest chưa apply đủ ServiceAccount, ClusterRoleBinding | `kubectl -n local-path-storage describe pod`; kéo image bằng `crictl pull`; apply lại đúng file trong `platform/`, không sửa manifest |
| B2: `thu-khong-class` mãi không được gán class | Chưa đánh dấu `local-path` mặc định, hoặc chưa chờ vòng lặp của control plane | `kubectl get sc` phải in `local-path (default)`; chờ theo điều kiện |
| PVC `Pending`, event `WaitForFirstConsumer` | **Đúng hành vi**: chưa có Pod nào dùng PVC | Với `data-db-0`: xem vì sao `db-0` chưa được lập lịch (`describe pod`); với `backup-store`: chờ Job đầu tiên |
| PVC `Pending`, event `ProvisioningFailed` | Helper Pod không kéo được `docker.io/library/busybox`, hoặc class dùng `Immediate` | `kubectl -n local-path-storage logs deploy/local-path-provisioner --tail=50`; kéo sẵn image helper; xóa rồi tạo lại PVC |
| B3: `deploy.sh` báo `spec: Forbidden: updates to statefulset spec for fields other than ...` | Thêm `volumeClaimTemplates` vào StatefulSet đang tồn tại | `./scripts/teardown.sh && ./scripts/deploy.sh` |
| B3: `db-0` khởi tạo DB mới sau khi đổi `PGDATA` | Dữ liệu cũ nằm ở đường dẫn cũ trong volume; `PGDATA` mới rỗng | Chấp nhận trong lúc chuyển đổi; `./scripts/teardown.sh && ./scripts/deploy.sh` cho sạch |
| B3, B5: `api` `CrashLoopBackOff`, log lỗi xác thực mật khẩu | `secrets/db.env` khác bí mật đã khởi tạo dữ liệu đang có | Theo ADR `mat-khau-du-lieu-ben`: chép lại `secrets/db.env` cũ, hoặc teardown để xóa dữ liệu |
| `deploy.sh` dừng ở `thieu secrets/db.env trong khi du lieu db cu con` | Repo vừa clone từ bundle, hoặc ai đó xóa `secrets/db.env` | Đúng thiết kế. Chép lại file cũ; hoặc `./scripts/teardown.sh` nếu chấp nhận mất dữ liệu |
| `deploy.sh` dừng ở `db-migrate that bai` ngay lần đầu | Hiếm: migration lỗi thật — `deploy.sh` đã chờ `db-0` Ready trước khi tạo Job | `kubectl -n bookstore logs job/db-migrate`; sửa migration ở repo |
| B4: Pod backup `Pending`, `volume node affinity conflict` | PV `backup-store` đã được tạo ở node khác trước khi Job được ghim | `./scripts/teardown.sh && ./scripts/deploy.sh`: PV mới tạo đúng node của lần chạy đầu |
| B5: PVC `data-db-0` biến mất ngay, không qua `Terminating` | `db-0` đã không còn khi xóa PVC | Đúng thứ tự của kịch bản D: xóa PVC khi `db-0` đang chạy, rồi mới xóa StatefulSet |
| B5: `db-0` treo `ContainerCreating`, event `hostPath type check failed` | Đường dẫn trong PV khôi phục sai | Xóa StatefulSet, PVC `data-db-0`, PV khôi phục (Retain — thư mục cũ không bị đụng); sửa đường dẫn từ `b5-pv-cu.yaml`; apply lại; `deploy.sh` |
| B5: `data-db-0` bind vào một PV mới, log `db-0` không có `Skipping initialization` | Chạy `deploy.sh` trước khi có PV khôi phục, hoặc PV khôi phục khác class, dung lượng, `claimRef` | Như dòng trên, cộng xóa PV mới (thư mục của nó được `teardown.sh` dọn sau). Dữ liệu cũ vẫn ở thư mục cũ |
| B6: `deploy.sh` báo không tìm thấy ConfigMap `web-config` | Tên trong `projected.sources` khác tên của generator | Viết đúng tên generator; Kustomize tự thêm hậu tố hash |
| B6: `/status.json` không đổi version sau ba phút | Đổi label trên Deployment thay vì trên Pod; hoặc mount nguồn projected bằng `subPath` | `kubectl label pod <Pod web> ... --overwrite`; mount cả thư mục |
| B7: Pod vượt giới hạn không bị trục xuất | Kubelet không đo được trên bố cục node hiện tại (bài 95) | Đối chiếu `b7-filesystem.txt`; không chỉnh cấu hình kubelet để "ép" bước này |
| B8: `cat .../PG_VERSION` báo không có file | Mount sai đường dẫn, hoặc Pod không ở `lab-k8s-worker2` | Đọc `mountPath` và `nodeName` của Pod; thư mục dữ liệu nằm dưới `<mount>/local-path-provisioner/` |
| B10: `teardown.sh` in `teardown: con ... thu muc du lieu tren node` | `ssh` tới worker hỏng, hoặc PV chưa xóa xong | Chạy lại `teardown.sh`; kiểm `ssh <worker> 'sudo -n true'` và `kubectl get pv` |
| PV kẹt `Terminating` | Còn PVC bind vào nó (`kubernetes.io/pv-protection`), hoặc provisioner không chạy để dọn PV loại `Delete` | Xóa PVC trước; kiểm provisioner Ready. Không gỡ finalizer bằng tay |
| PVC hoặc namespace kẹt `Terminating` | `kubernetes.io/pvc-protection` chờ Pod cuối cùng dùng PVC biến mất | Xóa Pod; chờ hết ân hạn của `db-0` (60 giây) |
| B10: `/opt/local-path-provisioner` còn thư mục con | PV của một PVC thử nghiệm chưa thu hồi xong | `kubectl get pv`; chờ provisioner; chạy lại `teardown.sh` — chỉ xóa tay khi `kubectl get pv` đã rỗng |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Storage](https://v1-35.docs.kubernetes.io/docs/concepts/storage/)
- [Kubernetes v1.35 — Volumes](https://v1-35.docs.kubernetes.io/docs/concepts/storage/volumes/)
- [Kubernetes v1.35 — Persistent Volumes](https://v1-35.docs.kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes v1.35 — Storage Classes](https://v1-35.docs.kubernetes.io/docs/concepts/storage/storage-classes/)
- [Kubernetes v1.35 — Dynamic Volume Provisioning](https://v1-35.docs.kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [Kubernetes v1.35 — Projected Volumes](https://v1-35.docs.kubernetes.io/docs/concepts/storage/projected-volumes/)
- [Kubernetes v1.35 — Ephemeral Volumes](https://v1-35.docs.kubernetes.io/docs/concepts/storage/ephemeral-volumes/)
- [Kubernetes v1.35 — Local Ephemeral Storage](https://v1-35.docs.kubernetes.io/docs/concepts/storage/ephemeral-storage/)
- [Kubernetes v1.35 — StatefulSets](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes v1.35 — Configure a Pod to Use a Projected Volume for Storage](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-projected-volume-storage/)
- [Kubernetes v1.35 — Configure a Pod to Use a Volume for Storage](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-volume-storage/)
- [Kubernetes v1.35 — Run a Single-Instance Stateful Application](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/)
- [Kubernetes v1.35 — Run a Replicated Stateful Application](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/)
- [Kubernetes v1.35 — Change the default StorageClass](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/)
- [Kubernetes — `kubectl apply` (cờ `--selector`)](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [Kubernetes — Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [Rancher — local-path-provisioner](https://github.com/rancher/local-path-provisioner)
- [Docker Hub — postgres (PGDATA, Initialization scripts)](https://hub.docker.com/_/postgres)
- [PostgreSQL 17 — pg_dump](https://www.postgresql.org/docs/17/app-pgdump.html)
