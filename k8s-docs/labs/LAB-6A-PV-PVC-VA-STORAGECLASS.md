# Lab 6a — PV, PVC và StorageClass

> **Điểm bắt đầu:** snapshot `02-net-ready` — mốc do Lab 5b tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot).
> **Điểm kết thúc:** lab **tạo mốc mới `03-storage-ready`**. Đây là một trong bốn lab đổi hạ tầng
> vĩnh viễn trên chuỗi chính: nó cài một dynamic provisioner và để lại một StorageClass mặc định.
> **Lab trước:** Lab 5b — NetworkPolicy, Ingress và CNI (chưa viết, xem
> [bản đồ lab](README.md#4-bản-đồ-lab)) đã đổi CNI, cài ingress controller và chụp `02-net-ready`.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng phần cốt lõi của
[Giai đoạn 6 — Lưu trữ](../00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ): tám bài khái niệm 90, 91,
92, 96, 98, 93, 94, 95 cộng bốn bài thực hành 277, 280, 343, 344. Phần snapshot, nhân bản và
VolumeAttributesClass **không** thuộc lab này — chúng là nội dung của Lab 6b.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không chép
lại con số phiên bản Kubernetes nào**. Thành phần duy nhất lab cài thêm là dynamic provisioner, và
version của nó đã được khóa sẵn ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) — xem
[B3](#b3-cài-dynamic-provisioner-và-storageclass-mặc-định).

Lab dùng Pod, ConfigMap, Secret, downwardAPI của giai đoạn 3, Deployment/StatefulSet của giai đoạn
4 và Service headless của giai đoạn 5 làm công cụ. **Không** dùng ResourceQuota, LimitRange, node
affinity, taint/toleration, PriorityClass (giai đoạn 7), RBAC (giai đoạn 9) hay metrics-server
(giai đoạn 11).

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`, rồi thêm hai lệnh riêng của lab này:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Hai lệnh riêng của lab 6a: tầng lưu trữ phải còn trống hoàn toàn.
kubectl get storageclass
kubectl get persistentvolume
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod; **hai lệnh cuối đều trả
`No resources found`** — cluster chưa có StorageClass và chưa có PersistentVolume nào.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Khác biệt giữa **Volume** và **PersistentVolume**: `emptyDir` sống theo Pod và bị xóa vĩnh viễn
  khi Pod bị gỡ khỏi node, còn dữ liệu sau một PVC sống lâu hơn Pod dùng nó.
- Vòng đời PV/PVC đầy đủ: cấp phát tĩnh và động, `Available` → `Bound` → `Released`, ràng buộc
  **một-một và độc quyền** qua `claimRef`, và claim không có volume khớp thì `Pending` vô thời hạn.
- Ý nghĩa thật của `accessModes`: `ReadWriteOnce` là ràng buộc **theo node**, không phải theo Pod;
  access mode dùng để **khớp** PVC với PV chứ không cưỡng chế chống ghi.
- Ba trạng thái của `storageClassName` trên PVC — đặt tên class, đặt `""`, và bỏ trống — cho ba
  hành vi khác nhau, kể cả khi StorageClass mặc định xuất hiện **sau** khi PVC được tạo.
- Một **StorageClass** gồm những gì: `provisioner`, `reclaimPolicy`, `volumeBindingMode`,
  `allowVolumeExpansion`, và cách đánh dấu class mặc định cho cluster.
- **Cấp phát động** làm gì: tạo PVC là đủ để một PV xuất hiện, ai tạo ra nó, và nó tạo cái gì trên
  đĩa của node.
- `volumeBindingMode: WaitForFirstConsumer` giữ PVC ở `Pending` cho tới khi có Pod tiêu thụ, và vì
  sao `Immediate` không dùng được với backend bị ràng buộc theo topology.
- `reclaimPolicy: Delete` và `Retain` khác nhau ra sao **khi bạn xóa PVC**, và ba bước thu hồi thủ
  công của `Retain`.
- `volumeClaimTemplates` của StatefulSet sinh **một PVC riêng cho mỗi replica** theo quy tắc tên
  `<claim>-<statefulset>-<ordinal>`, và các PVC đó **không** bị xóa khi thu nhỏ hay khi xóa
  StatefulSet — **đây là phần trả nợ #2**.
- **Projected volume** gộp ConfigMap, Secret và downwardAPI vào một thư mục duy nhất, kèm hai khác
  biệt cú pháp so với volume rời.
- **Volume tạm thời tổng quát** trông như `emptyDir` nhưng sinh ra một PVC thật, tên xác định
  `<tên Pod>-<tên volume>`, và bị xóa theo Pod nhờ owner reference.
- **Lưu trữ tạm thời cục bộ** là tài nguyên đĩa của node: đọc được ở `Allocatable`, khai được
  `requests`/`limits.ephemeral-storage`, và vượt limit thì kubelet **trục xuất Pod**.
- Cleanup đúng phạm vi: xóa mọi object của lab nhưng **giữ lại provisioner và StorageClass mặc
  định**, rồi chụp mốc `03-storage-ready`.

### Nợ #2 được trả ở lab này

[Sổ nợ lab](README.md#5-sổ-nợ-lab) ghi nợ #2 — `volumeClaimTemplates` của StatefulSet — phát sinh ở
[giai đoạn 4](../00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller), bài
[65](../65-statefulset-vi.md), và bị hoãn ở
[Lab 4b](LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) vì cluster khi đó chưa có StorageClass lẫn
provisioner. [B6](#b6-volumeclaimtemplates--trả-nợ-2) trả món nợ đó.

**Trước khi làm B6, đọc lại bài [65](../65-statefulset-vi.md)** — cụ thể hai mục *Template cho
volume claim*, *Lưu trữ ổn định*, và mục *Lưu giữ PersistentVolumeClaim*.

Nợ #3 (Service headless quản trị cho StatefulSet) đã được trả ở Lab 5a, nên B6 dùng Service
headless như một thứ đã học. Nợ #5 (snapshot và nhân bản volume) **không** thuộc lab này.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 6a | Kiểm chứng ở |
| --- | --- |
| [90 — Lưu trữ](../90-storage-vi.md) | B0 — trang mục lục; lab đi đúng hai nhánh nó chia: bền vững (B2–B6) và tạm thời (B1, B7–B9) |
| [91 — Các Volume](../91-volumes-vi.md) | B1 (`emptyDir` và ranh giới vòng đời của nó, chia sẻ filesystem giữa hai container), B2.1 và B2.6 (`hostPath` và vì sao nó nguy hiểm trong cluster nhiều node), B7 (`configMap`, `secret`), B9.4 (`emptyDir.medium: Memory` là tmpfs) |
| [92 — Volume bền vững](../92-persistent-volumes-vi.md) | B2 (cấp phát tĩnh, `Available`→`Bound`, ràng buộc độc quyền, `accessModes`, `storageClassName: ""`), B3.4 (gán class mặc định hồi tố), B4 (cấp phát động, `allowVolumeExpansion`), B5 (bảo vệ object đang dùng, `Delete` so với `Retain`, pha `Released`, thu hồi thủ công) |
| [96 — Lớp lưu trữ](../96-storage-classes-vi.md) | B3 (`provisioner`, class mặc định), B4.1 và B4.4 (`volumeBindingMode`), B4.5 (`allowVolumeExpansion`), B5.3 (`reclaimPolicy` của class quyết định policy của PV nó sinh ra) |
| [98 — Cấp phát Volume động](../98-dynamic-provisioning-vi.md) | B3 (ranh giới admin/người dùng, hai điều kiện của hành vi mặc định), B4.2 (tạo PVC là đủ để PV xuất hiện), B5.2 (xóa claim thì volume bị hủy) |
| [93 — Volume dạng projected](../93-projected-volumes-vi.md) | B7 — gộp ba nguồn vào một thư mục, `name` thay cho `secretName`, `defaultMode` so với `mode`, và projection `serviceAccountToken` mà mọi Pod đang dùng sẵn |
| [94 — Volume tạm thời](../94-ephemeral-volumes-vi.md) | B8 — volume tạm thời tổng quát sinh PVC thật, quy tắc tên `<Pod>-<volume>`, owner reference, và xung đột tên làm Pod không khởi động được |
| [95 — Lưu trữ tạm thời cục bộ](../95-ephemeral-storage-vi.md) | B9 — `ephemeral-storage` trong `Capacity`/`Allocatable`, khai `requests`/`limits`, bẫy hậu tố, và trục xuất khi vượt limit |
| [277 — Pod dùng projected Volume](../277-configure-projected-volume-vi.md) | B7.1–B7.2 — đóng gói secret từ file rồi mount nhiều nguồn vào cùng một thư mục |
| [280 — Pod dùng Volume để lưu trữ](../280-configure-volume-storage-vi.md) | B1 — dữ liệu trong `emptyDir` sống qua lần khởi động lại container, nhưng chết theo Pod |
| [343 — Ứng dụng có trạng thái được nhân bản](../343-run-replicated-stateful-application-vi.md) | B6 — `volumeClaimTemplates`, mỗi replica một PVC, scale lên/xuống và PVC **không** tự biến mất. Phần replication MySQL không chạy, xem bảng dưới |
| [344 — Ứng dụng có trạng thái đơn thực thể](../344-run-single-instance-stateful-vi.md) | B2 — PV `hostPath` tạo tay + PVC + workload đơn thực thể ghi dữ liệu; B2.4 chứng minh dữ liệu sống lâu hơn Pod |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài [343](../343-run-replicated-stateful-application-vi.md), phần replication MySQL | Ba instance MySQL kèm sidecar xtrabackup vượt headroom của ba VM lab, và kịch bản hỏng của bài cần `kubectl drain` — nội dung [giai đoạn 12](../00-ALO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao). B6 giữ đúng phần lưu trữ của bài |
| Bài [92](../92-persistent-volumes-vi.md), mục *Hỗ trợ raw block volume* | Ứng dụng phải tự xử lý thiết bị khối thô; provisioner của lab chỉ cấp `volumeMode: Filesystem` |
| Bài [92](../92-persistent-volumes-vi.md), các mục snapshot / nhân bản / volume populator | Cần CSI driver có hỗ trợ. Đây là **nợ #5**, trả ở Lab 6b |
| Bài [92](../92-persistent-volumes-vi.md) mục *Recycle*; bài [96](../96-storage-classes-vi.md) các mục AWS/Azure/vSphere/Ceph/Portworx | `Recycle` đã lỗi thời; các provisioner cloud không tồn tại trên cluster bare-metal |
| Bài [94](../94-ephemeral-volumes-vi.md), mục *Volume tạm thời CSI* | Cần một CSI driver viết riêng cho chế độ inline; provisioner của lab không phải CSI driver |
| Bài [95](../95-ephemeral-storage-vi.md), mục *Hạn ngạch project của filesystem* | Beta và tắt mặc định; cần user namespace và filesystem bật `prjquota` |

### 1.2. Thời lượng

3–4 giờ, tính từ lúc gate mở đầu đã PASS, **chưa kể** thời gian tắt máy và chụp snapshot ở B10.
B4, B5 và B9 có bước phải chờ controller hội tụ; thời gian chờ phụ thuộc cấu hình cluster nên mọi
bước chờ đều viết dưới dạng vòng lặp có điều kiện dừng, không phải con số cố định.

---

## 2. Quy ước và an toàn

> **Cảnh báo — lab này đổi hạ tầng vĩnh viễn.** B3 cài một dynamic provisioner vào namespace mới
> `local-path-storage` và để lại một StorageClass mặc định cho toàn cluster. B10 chụp mốc
> `03-storage-ready`. Trước khi chạy bất kỳ lệnh nào của phần B, **bảo đảm cả ba VM đều còn
> snapshot `02-net-ready`** — đó là điểm quay lui duy nhất.

Kiểm tra trên PowerShell của máy host trước khi bắt đầu; sửa đường dẫn `.vmx` nếu VM nằm chỗ khác:

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

**PASS:** đúng ba dòng `PASS:`, không dòng `FAIL:` nào. Nếu một VM thiếu mốc `02-net-ready`, dừng
lại: quay về Lab 5b và chụp mốc đó trước, đừng chạy tiếp lab này.

**Cách quay lui khi hỏng:** tắt **cả ba VM**, restore **cả ba** về `02-net-ready` — không bao giờ
restore riêng một VM, xem ghi chú cuối [mục 4 của Lab
00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường) — rồi bật lại theo thứ tự master →
worker 1 → worker 2, chạy lại gate mở đầu, và bắt đầu lại từ B0.

Các quy ước còn lại:

- Mọi lệnh chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, **trừ khi ghi rõ node
  khác**. Các bước phải chạy trên node khác đều mở đầu bằng dòng "Chạy trên `<node>`".
- **Chạy toàn bộ phần B trong cùng một phiên shell.** Nhiều gate so sánh biến đặt ở bước trước
  (`PV_DYN`, `PV_PATH`, `PV_RETAIN`…); mở shell mới giữa chừng là mất biến.
- Lab tạo Namespace `lab-6a` và các object bên trong nó, cộng ba object phạm vi cluster: PV tĩnh
  `lab-6a-static`, StorageClass `lab-6a-retain` và StorageClass `lab-6a-immediate`. Ba thứ này bị
  xóa ở B10. Namespace `local-path-storage` và StorageClass `local-path` thì **ở lại**.
- **Fault injection chỉ trên `lab-k8s-worker2`** — B9.3 cố tình làm một Pod vượt limit lưu trữ tạm
  thời và bị trục xuất; bước đó ghim Pod vào đúng node này.
- B2.6 cũng ghim một Pod vào `lab-k8s-worker2`, nhưng đó là bước **đọc**, không phá gì; nó chỉ
  chứng minh cái bẫy của `hostPath` trong cluster nhiều node.
- Không sửa manifest control plane, không sửa cấu hình kubelet/containerd, không đụng vào
  `kube-system`, vào namespace của CNI hay của ingress controller do Lab 5b cài.
- Lab cần kéo được image từ internet: `busybox` theo bảng A1.3, image của provisioner, và image
  `docker.io/library/busybox` **không pin tag** mà manifest upstream dùng cho helper Pod. Nếu môi
  trường cô lập, xem mục 4.
- Manifest tạm ghi vào `~/lab-work/6a/`; bằng chứng ghi vào `~/lab-evidence/6a/`. Không lưu token,
  key hay certificate.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail.

---

# Phần B — Thực hành kiến thức 6a

## B0. Chuẩn bị workspace và ảnh chụp "trước" của tầng lưu trữ

**Mục đích:** ghi lại trạng thái tầng lưu trữ **trước** khi lab đụng vào, để B10 có cái đối chiếu.

```bash
mkdir -p ~/lab-work/6a ~/lab-evidence/6a
kubectl config current-context
kubectl get nodes -o wide
kubectl create namespace lab-6a

{
  echo "=== $(date -Is) — tầng lưu trữ trước Lab 6a ==="
  echo '--- storageclass ---';      kubectl get storageclass 2>&1
  echo '--- persistentvolume ---';  kubectl get persistentvolume 2>&1
  echo '--- csidrivers ---';        kubectl get csidrivers 2>&1
  echo '--- csinodes ---';          kubectl get csinodes 2>&1
  echo '--- namespaces ---';        kubectl get namespaces
} | tee ~/lab-evidence/6a/b0-truoc.txt
```

**Ý nghĩa:** bài [90](../90-storage-vi.md) chia phần Storage làm hai nhánh — lưu trữ dài hạn và lưu
trữ tạm thời. Cluster lab đang ở trạng thái mà nhánh dài hạn **chưa có gì**: không StorageClass,
không PV, và `csinodes` không quảng bá driver nào. Đó chính là lý do giai đoạn 6 cần một lab riêng
để cài provisioner trước khi thực hành được nhánh dài hạn.

```bash
SC_N="$(kubectl get storageclass --no-headers 2>/dev/null | wc -l)"
PV_N="$(kubectl get persistentvolume --no-headers 2>/dev/null | wc -l)"
NS_LPP="$(kubectl get namespace local-path-storage --ignore-not-found -o name | wc -l)"
echo "storageclass=$SC_N | persistentvolume=$PV_N | namespace local-path-storage=$NS_LPP"

test "$SC_N" -eq 0 && test "$PV_N" -eq 0 && test "$NS_LPP" -eq 0 \
  && echo 'PASS: tầng lưu trữ trống, lab bắt đầu từ đúng điểm xuất phát'

kubectl get namespace lab-6a -o jsonpath='{.status.phase}'; echo
```

**PASS:** dòng `PASS: tầng lưu trữ trống…` xuất hiện; namespace `lab-6a` ở phase `Active`. Nếu
`SC_N` khác 0, bạn đang đứng trên một cluster đã chạy lab 6a dở dang — restore `02-net-ready` cho
cả ba VM rồi bắt đầu lại.

---

## B1. `emptyDir` — volume chết theo Pod

**Mục đích:** dựng mốc đối chiếu cho toàn bộ phần còn lại của lab. Bài
[91](../91-volumes-vi.md#emptydir) nói `emptyDir` sống qua lần khởi động lại container nhưng bị
**xóa vĩnh viễn** khi Pod bị gỡ khỏi node; bài [280](../280-configure-volume-storage-vi.md) là bản
thực hành của đúng câu đó. B2 và B4 lặp lại y hệt kịch bản này với PVC và cho kết quả ngược lại.

### B1.1. Một Pod, hai container, một `emptyDir` dùng chung

```bash
cat > ~/lab-work/6a/emptydir.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: scratch
  namespace: lab-6a
spec:
  containers:
  - name: writer
    image: busybox:1.37
    command: ["sh", "-c", "date -Is >> /cache/boots.txt; sleep 30; exit 1"]
    volumeMounts:
    - name: cache
      mountPath: /cache
  - name: reader
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir:
      sizeLimit: 64Mi
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/6a/emptydir.yaml
kubectl apply -f ~/lab-work/6a/emptydir.yaml
kubectl wait --for=condition=Ready pod/scratch -n lab-6a --timeout=180s
```

**Ý nghĩa của từng lựa chọn:**

- **Hai container cùng mount một volume** là công dụng thứ hai mà bài 91 nêu ở mục *Tại sao volume
  quan trọng*: chia sẻ filesystem giữa các container trong cùng Pod. `reader` sống suốt mục này để
  bạn `exec` vào đọc bất cứ lúc nào.
- **`writer` cố tình `exit 1`** sau 30 giây. Không cần gửi tín hiệu vào PID 1 — container tự thoát,
  `restartPolicy` mặc định `Always` khởi động lại nó, và `restartCount` tăng một cách xác định.
- **`sizeLimit: 64Mi`** là trần dung lượng của volume, lấy từ lưu trữ tạm thời của node. B9 quay
  lại đúng con số này ở góc nhìn tài nguyên node.

**PASS:** `--dry-run=server` in `pod/scratch created (server dry run)` không lỗi validation; Pod
`scratch` đạt condition `Ready`.

### B1.2. Dữ liệu sống qua lần khởi động lại container

```bash
kubectl exec -n lab-6a scratch -c reader -- sh -c 'echo lab-6a-emptydir > /cache/marker'
kubectl exec -n lab-6a scratch -c reader -- cat /cache/marker

for i in $(seq 1 60); do
  RC="$(kubectl get pod scratch -n lab-6a \
    -o jsonpath='{.status.containerStatuses[?(@.name=="writer")].restartCount}')"
  test "${RC:-0}" -ge 1 && break
  sleep 2
done
echo "restartCount của writer = ${RC:-0}"

BOOTS="$(kubectl exec -n lab-6a scratch -c reader -- sh -c 'wc -l < /cache/boots.txt' | tr -d ' ')"
MARK="$(kubectl exec -n lab-6a scratch -c reader -- cat /cache/marker)"
echo "số dòng boots.txt = $BOOTS | marker = $MARK"

test "${RC:-0}" -ge 1 && test "$BOOTS" -ge 2 && test "$MARK" = 'lab-6a-emptydir' \
  && echo 'PASS: container khởi động lại, dữ liệu emptyDir vẫn còn'
```

**Ý nghĩa:** `boots.txt` có từ hai dòng trở lên chứng minh `writer` đã chạy lại ít nhất một lần, và
file cũ vẫn nằm nguyên trong volume. Đúng ghi chú của bài 91: *"Container bị crash không làm Pod bị
gỡ khỏi node. Dữ liệu trong volume `emptyDir` vẫn an toàn khi container bị crash."*

**PASS:** dòng `PASS: container khởi động lại, dữ liệu emptyDir vẫn còn` xuất hiện.

### B1.3. Xóa Pod là mất sạch

```bash
kubectl delete pod scratch -n lab-6a --wait=true
kubectl apply -f ~/lab-work/6a/emptydir.yaml
kubectl wait --for=condition=Ready pod/scratch -n lab-6a --timeout=180s

if kubectl exec -n lab-6a scratch -c reader -- test -f /cache/marker; then
  echo 'FAIL: marker vẫn còn sau khi Pod bị xóa và tạo lại'
else
  echo 'PASS: emptyDir trống trơn — dữ liệu bị xóa vĩnh viễn cùng Pod'
fi

kubectl exec -n lab-6a scratch -c reader -- sh -c 'wc -l < /cache/boots.txt' | tr -d ' '
kubectl delete pod scratch -n lab-6a --wait=true
```

**Ý nghĩa:** Pod mới trùng tên, trùng manifest, có thể trùng cả node — nhưng volume là một thư mục
mới tinh, `boots.txt` quay về 1 dòng. Đây là ranh giới mà cả giai đoạn 6 sinh ra để vượt qua: muốn
dữ liệu sống lâu hơn Pod thì phải tách vòng đời lưu trữ khỏi vòng đời Pod, tức là PersistentVolume.

**PASS:** dòng `PASS: emptyDir trống trơn…` xuất hiện; `boots.txt` chỉ còn 1 dòng.

---

## B2. Cấp phát tĩnh — PV do bạn tự tạo

**Mục đích:** đi hết vòng đời PV/PVC của bài [92](../92-persistent-volumes-vi.md) khi cluster
**chưa có provisioner nào**. Đây đúng là con đường mà bài
[98](../98-dynamic-provisioning-vi.md) mô tả là "công việc mà cấp phát động sinh ra để loại bỏ", và
là cách bài [344](../344-run-single-instance-stateful-vi.md) dựng lưu trữ cho MySQL đơn thực thể.

### B2.1. PV tĩnh và pha `Available`

```bash
cat > ~/lab-work/6a/pv-static.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: lab-6a-static
spec:
  capacity:
    storage: 128Mi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: /srv/lab-6a/static
    type: DirectoryOrCreate
EOF

kubectl apply -f ~/lab-work/6a/pv-static.yaml
kubectl get pv lab-6a-static \
  -o custom-columns=NAME:.metadata.name,CAP:.spec.capacity.storage,\
MODE:.spec.accessModes,RECLAIM:.spec.persistentVolumeReclaimPolicy,\
STATUS:.status.phase,CLAIM:.spec.claimRef.name

PHASE="$(kubectl get pv lab-6a-static -o jsonpath='{.status.phase}')"
CLAIMREF="$(kubectl get pv lab-6a-static -o jsonpath='{.spec.claimRef.name}')"
echo "phase=$PHASE | claimRef=${CLAIMREF:-<rỗng>}"

test "$PHASE" = 'Available' && test -z "$CLAIMREF" \
  && echo 'PASS: PV ở pha Available và chưa có claimRef'
```

**Ý nghĩa của từng trường:**

- **`hostPath` + `type: DirectoryOrCreate`** — kubelet tạo thư mục nếu chưa có, quyền `0755`. Bài
  92 cảnh báo thẳng: PV `hostPath` chỉ dùng để **kiểm thử trên một node**, "SẼ KHÔNG HOẠT ĐỘNG
  trong cluster nhiều node". B2.6 sẽ chứng minh đúng câu cảnh báo đó thay vì né nó.
- **`storageClassName: ""`** — PV này **không thuộc class nào**, nên chỉ bind được với PVC cũng
  khai `""`. Đó là điều kiện để B3 chứng minh việc gán class mặc định hồi tố không đụng tới nó.
- **`persistentVolumeReclaimPolicy: Retain`** — PV tạo tay giữ nguyên policy bạn gán lúc tạo (bài
  96, mục *Chính sách thu hồi*). B5.5 dùng đúng PV này để thực hành ba bước thu hồi thủ công.
- **`capacity: 128Mi`** — với `hostPath` đây chỉ là con số để **khớp** PVC, không phải hạn ngạch.
  Không có gì chặn container ghi quá 128Mi vào thư mục đó.

**PASS:** dòng `PASS: PV ở pha Available và chưa có claimRef` xuất hiện.

### B2.2. PVC bind vào PV: ánh xạ một-một qua `claimRef`

```bash
cat > ~/lab-work/6a/pvc-static.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-static
  namespace: lab-6a
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 128Mi
EOF

kubectl apply -f ~/lab-work/6a/pvc-static.yaml

for i in $(seq 1 60); do
  PVC_PHASE="$(kubectl get pvc pvc-static -n lab-6a -o jsonpath='{.status.phase}')"
  test "$PVC_PHASE" = 'Bound' && break
  sleep 2
done

PV_PHASE="$(kubectl get pv lab-6a-static -o jsonpath='{.status.phase}')"
CR_NS="$(kubectl get pv lab-6a-static -o jsonpath='{.spec.claimRef.namespace}')"
CR_NAME="$(kubectl get pv lab-6a-static -o jsonpath='{.spec.claimRef.name}')"
PVC_VOL="$(kubectl get pvc pvc-static -n lab-6a -o jsonpath='{.spec.volumeName}')"
echo "PVC=$PVC_PHASE | PV=$PV_PHASE | claimRef=$CR_NS/$CR_NAME | volumeName=$PVC_VOL"

test "$PVC_PHASE" = 'Bound' && test "$PV_PHASE" = 'Bound' \
  && test "$CR_NS/$CR_NAME" = 'lab-6a/pvc-static' && test "$PVC_VOL" = 'lab-6a-static' \
  && echo 'PASS: ràng buộc hai chiều PV <-> PVC đã hình thành'
```

**Ý nghĩa:** không ai tạo PV mới cả — vòng lặp điều khiển trong control plane tìm thấy một PV khớp
về class, access mode và dung lượng, rồi ghi hai chiều: `claimRef` trên PV và `volumeName` trên
PVC. Bài 92 gọi đây là ánh xạ **một-một, độc quyền**. B2.3 chứng minh vế "độc quyền".

**PASS:** dòng `PASS: ràng buộc hai chiều PV <-> PVC đã hình thành` xuất hiện.

### B2.3. Claim không có volume khớp thì `Pending` vô thời hạn

```bash
cat > ~/lab-work/6a/pvc-static-2.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-static-2
  namespace: lab-6a
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 128Mi
EOF

kubectl apply -f ~/lab-work/6a/pvc-static-2.yaml
sleep 20

P2="$(kubectl get pvc pvc-static-2 -n lab-6a -o jsonpath='{.status.phase}')"
PV_TOTAL="$(kubectl get pv --no-headers | wc -l)"
echo "pvc-static-2 = $P2 | tổng số PV trong cluster = $PV_TOTAL"

test "$P2" = 'Pending' && test "$PV_TOTAL" -eq 1 \
  && echo 'PASS: claim thứ hai đứng Pending, không có PV nào được sinh thêm'

kubectl describe pvc pvc-static-2 -n lab-6a | tail -n 12 \
  | tee ~/lab-evidence/6a/b2-pvc-pending.txt
```

**Ý nghĩa:** hai PVC giống hệt nhau, nhưng chỉ một cái được phục vụ. Bài 92 nói rõ: *"Claim sẽ ở
trạng thái chưa ràng buộc vô thời hạn nếu không tồn tại volume khớp"*, và không có gì trong
Kubernetes tự tạo volume cho nó — vì cluster chưa có provisioner, và vì `storageClassName: ""`
**tự vô hiệu hóa cấp phát động** cho chính claim đó. B3.4 kiểm tra lại PVC này sau khi đã có
StorageClass mặc định: nó vẫn `Pending`.

**PASS:** dòng `PASS: claim thứ hai đứng Pending…` xuất hiện; số PV trong cluster vẫn là 1.

### B2.4. Dữ liệu sống lâu hơn Pod

```bash
cat > ~/lab-work/6a/pod-static-a.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: static-a
  namespace: lab-6a
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker1
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-static
EOF

kubectl apply -f ~/lab-work/6a/pod-static-a.yaml
kubectl wait --for=condition=Ready pod/static-a -n lab-6a --timeout=180s

kubectl exec -n lab-6a static-a -- sh -c 'echo lab-6a-static-payload > /data/marker'
kubectl exec -n lab-6a static-a -- cat /data/marker

kubectl delete pod static-a -n lab-6a --wait=true
kubectl apply -f ~/lab-work/6a/pod-static-a.yaml
kubectl wait --for=condition=Ready pod/static-a -n lab-6a --timeout=180s

AFTER="$(kubectl exec -n lab-6a static-a -- cat /data/marker)"
echo "sau khi xóa và tạo lại Pod: $AFTER"

test "$AFTER" = 'lab-6a-static-payload' \
  && echo 'PASS: dữ liệu trong PVC sống lâu hơn Pod dùng nó'
```

**Ý nghĩa:** đây là B1.3 chạy lại với một PVC thay cho `emptyDir`, và kết quả ngược hẳn. Vòng đời
của PV **độc lập với bất kỳ Pod riêng lẻ nào** — đúng câu mở đầu của bài 92.

`nodeSelector: kubernetes.io/hostname` là cách bài [96](../96-storage-classes-vi.md#volume-binding-mode)
tự khuyến nghị để ghim Pod vào một node cụ thể. Ở đây nó cần thiết vì PV `hostPath` **không mang
node affinity**: không ghim thì Pod có thể lên node khác và thấy một thư mục khác. B4 sẽ cho thấy
PV do provisioner sinh ra không có vấn đề này.

**PASS:** dòng `PASS: dữ liệu trong PVC sống lâu hơn Pod dùng nó` xuất hiện.

### B2.5. `ReadWriteOnce` là ràng buộc theo node, không phải theo Pod

```bash
sed -e 's/name: static-a/name: static-b/' ~/lab-work/6a/pod-static-a.yaml \
  > ~/lab-work/6a/pod-static-b.yaml
kubectl apply -f ~/lab-work/6a/pod-static-b.yaml
kubectl wait --for=condition=Ready pod/static-b -n lab-6a --timeout=180s

kubectl get pods -n lab-6a -o wide | tee ~/lab-evidence/6a/b2-rwo.txt

NODE_A="$(kubectl get pod static-a -n lab-6a -o jsonpath='{.spec.nodeName}')"
NODE_B="$(kubectl get pod static-b -n lab-6a -o jsonpath='{.spec.nodeName}')"
SEEN_B="$(kubectl exec -n lab-6a static-b -- cat /data/marker)"
echo "static-a trên $NODE_A | static-b trên $NODE_B | static-b đọc được: $SEEN_B"

test "$NODE_A" = "$NODE_B" && test "$SEEN_B" = 'lab-6a-static-payload' \
  && echo 'PASS: hai Pod trên cùng một node dùng chung được một PVC ReadWriteOnce'
```

**Ý nghĩa:** trực giác thông thường đọc `ReadWriteOnce` thành "chỉ một Pod". Bài 92 nói khác:
*"Chế độ truy cập ReadWriteOnce vẫn có thể cho phép nhiều pod truy cập volume đó khi các pod chạy
trên cùng một node."* Muốn giới hạn còn đúng một Pod trong toàn cluster thì phải dùng
`ReadWriteOncePod` — và chế độ đó chỉ được hỗ trợ cho volume CSI, nên provisioner của lab này không
cấp được.

**PASS:** dòng `PASS: hai Pod trên cùng một node dùng chung được một PVC ReadWriteOnce` xuất hiện.

### B2.6. Cái bẫy `hostPath`: cùng manifest, khác node, khác dữ liệu

```bash
sed -e 's/name: static-a/name: static-c/' \
    -e 's/lab-k8s-worker1/lab-k8s-worker2/' ~/lab-work/6a/pod-static-a.yaml \
  > ~/lab-work/6a/pod-static-c.yaml
kubectl apply -f ~/lab-work/6a/pod-static-c.yaml
kubectl wait --for=condition=Ready pod/static-c -n lab-6a --timeout=180s

NODE_C="$(kubectl get pod static-c -n lab-6a -o jsonpath='{.spec.nodeName}')"
echo "static-c trên $NODE_C"

if kubectl exec -n lab-6a static-c -- test -f /data/marker; then
  echo 'FAIL: static-c thấy marker — kiểm tra lại nodeSelector, Pod có thể đang ở worker1'
else
  echo 'PASS: cùng một PVC, Pod ở node khác thấy thư mục rỗng — đúng cảnh báo của bài 92'
fi

kubectl delete pod static-b static-c -n lab-6a --wait=true
```

**Ý nghĩa:** PVC vẫn `Bound`, PV vẫn `Bound`, Pod vẫn `Running` — không có lỗi nào ở tầng
Kubernetes. Nhưng dữ liệu thì không có, vì `hostPath` trỏ tới filesystem của **node đang chạy Pod**.
Đây chính là gạch đầu dòng thứ hai trong cảnh báo của bài 91: *"Các Pod có cấu hình giống hệt nhau
có thể hành xử khác nhau trên các node khác nhau do các file trên các node khác nhau."*

Bài 91 nêu lối thoát đúng là volume `local` kèm `nodeAffinity` trên PV, để scheduler biết Pod chỉ
được lên node có dữ liệu. Lab không viết tay `nodeAffinity` — B4.2 **đọc** trường đó trên PV do
provisioner sinh ra, nơi nó được đặt tự động.

**PASS:** dòng `PASS: cùng một PVC, Pod ở node khác thấy thư mục rỗng…` xuất hiện.

### B2.7. Một PVC không khai `storageClassName`, để dành cho B3

```bash
cat > ~/lab-work/6a/pvc-no-class.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-no-class
  namespace: lab-6a
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 256Mi
EOF

kubectl apply -f ~/lab-work/6a/pvc-no-class.yaml
sleep 10

kubectl get pvc pvc-no-class -n lab-6a -o json | grep -n 'storageClassName' || \
  echo '(không có khóa storageClassName trong spec — đúng như mong đợi)'
NC_PHASE="$(kubectl get pvc pvc-no-class -n lab-6a -o jsonpath='{.status.phase}')"
NC_CLASS="$(kubectl get pvc pvc-no-class -n lab-6a -o jsonpath='{.spec.storageClassName}')"
echo "pvc-no-class: phase=$NC_PHASE | storageClassName=${NC_CLASS:-<không đặt>}"

test "$NC_PHASE" = 'Pending' && test -z "$NC_CLASS" \
  && echo 'PASS: PVC không khai class, chưa có class mặc định, nên đứng Pending và spec vẫn trống'
```

**Ý nghĩa:** ba trạng thái `storageClassName` của bài 92 giờ đã đủ mặt trong namespace `lab-6a`:
`pvc-static` và `pvc-static-2` khai `""`, `pvc-no-class` **bỏ trống**. Bài 92 nói cluster xử lý hai
thứ đó **khác nhau**, và khác biệt chỉ lộ ra khi một StorageClass mặc định xuất hiện — B3.4 kiểm
đúng điểm đó.

PVC này xin `256Mi`, lớn hơn `capacity` của PV tĩnh, nên nó không thể vô tình cướp mất
`lab-6a-static`; ngoài ra PV đó đã `Bound` và ràng buộc là độc quyền.

**PASS:** dòng `PASS: PVC không khai class…` xuất hiện.

---

## B3. Cài dynamic provisioner và StorageClass mặc định

**Mục đích:** đây là bước đổi hạ tầng của lab. Bài [98](../98-dynamic-provisioning-vi.md) đặt ranh
giới rõ: **quản trị viên** tạo sẵn StorageClass, **người dùng** chỉ chọn class qua
`storageClassName`. Từ B4 trở đi bạn đóng vai người dùng; ở B3 bạn đóng vai quản trị viên.

Cluster lab là ba VM bare-metal, không có cloud provider, nên không có provisioner nội bộ nào dùng
được: bảng *Provisioner* của bài [96](../96-storage-classes-vi.md#provisioner) chỉ liệt kê
provisioner nội bộ cho AzureFile, Portworx và vSphere. Lựa chọn còn lại là một **external
provisioner** — chương trình độc lập tuân theo đặc tả cấp phát volume của Kubernetes.

Lab dùng **local-path-provisioner của Rancher**, vì ba lý do:

1. Nó cấp phát bằng thư mục trên đĩa sẵn có của node, không cần thêm disk, không cần NFS server,
   không cần thay đổi gì trên hạ tầng ngoài cluster.
2. Nó cài bằng **đúng một manifest**, không cần Helm — nên lab này không dùng tới dòng `Helm` của
   bảng A1.4; ngoài việc kéo image, nó không cần internet lúc chạy.
3. StorageClass nó tạo dùng `volumeBindingMode: WaitForFirstConsumer` — đúng chế độ mà bài 96
   khuyến nghị cho lưu trữ gắn với node, nên B4 kiểm chứng được cả hai chế độ gắn kết.

Đánh đổi phải biết: volume nằm trên **một node cụ thể**, nên nó không hỗ trợ `ReadWriteMany`, không
hỗ trợ mở rộng dung lượng, và Pod dùng volume đó chỉ chạy được trên đúng node đó. Bài
[91](../91-volumes-vi.md#local) nói thẳng về giới hạn này: node hỏng thì volume không truy cập được
và ứng dụng phải chịu được mức khả dụng suy giảm đó.

### B3.1. Tải manifest và đọc trước khi apply

Version của provisioner **không nằm trong file này và cũng không được chép vào đây**. Nó đã được
khóa ở dòng `local-path-provisioner` của
[bảng A1.4 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) — bảng đó
tồn tại đúng để giữ con số ở một chỗ cho lab nào thật sự cần thì link về. Lab 6a là lab đưa thành
phần đó vào chuỗi snapshot, nên nó **bổ sung cho bảng A1.3**; muốn đổi version thì sửa A1.4 trước,
đừng sửa ở đây.

Mở A1.4, đọc dòng `local-path-provisioner`, rồi điền vào biến dưới đây:

```bash
# Điền giá trị đọc được từ bảng A1.4 của Lab 00, dòng local-path-provisioner.
LPP_VERSION='<điền version từ A1.4>'

case "$LPP_VERSION" in
  ''|*'<'*|*'>'*|*'điền'*)
    echo 'FAIL: chưa điền LPP_VERSION — lấy version ở bảng A1.4 của Lab 00 rồi chạy lại' ;;
  v[0-9]*)
    echo "PASS: LPP_VERSION = $LPP_VERSION" ;;
  *)
    echo "FAIL: LPP_VERSION = $LPP_VERSION không có dạng vX.Y.Z như A1.4 ghi" ;;
esac
```

**PASS:** dòng `PASS: LPP_VERSION = …` xuất hiện. Không đi tiếp khi còn thấy `FAIL:` — mọi lệnh
phía dưới đều dùng lại biến này, và một placeholder chưa điền sẽ kéo theo cả chuỗi lỗi khó đọc.

```bash
LPP_URL="https://raw.githubusercontent.com/rancher/local-path-provisioner/${LPP_VERSION}/deploy/local-path-storage.yaml"

curl -fsSL -o ~/lab-work/6a/local-path-storage.yaml "$LPP_URL"
grep -nE '^kind:|^  name:|^  namespace:|image:|provisioner:|volumeBindingMode:|reclaimPolicy:|local-path-provisioner"' \
  ~/lab-work/6a/local-path-storage.yaml | tee ~/lab-evidence/6a/b3-manifest.txt
```

Gate ba thứ **trước khi** apply — không apply một manifest chưa đối chiếu:

```bash
LPP_FILE=~/lab-work/6a/local-path-storage.yaml
grep -q "image: docker.io/rancher/local-path-provisioner:${LPP_VERSION}" "$LPP_FILE" \
  && echo "PASS: image đúng version ${LPP_VERSION}"
grep -q 'provisioner: rancher.io/local-path' "$LPP_FILE" \
  && echo 'PASS: tên provisioner đúng như mong đợi'
grep -q '"/opt/local-path-provisioner"' "$LPP_FILE" \
  && echo 'PASS: đường dẫn gốc trên node là /opt/local-path-provisioner'
```

**Ý nghĩa:** ba dòng gate trên khóa ba thứ mà phần còn lại của lab dựa vào: **version** đúng bảng
A1.4, **tên provisioner** mà StorageClass sẽ trỏ tới, và **thư mục gốc trên node** mà B4.3 đi kiểm
tra bằng tay. Manifest còn tạo namespace `local-path-storage`, một ServiceAccount kèm ClusterRole,
một Deployment, một ConfigMap cấu hình và một StorageClass.

Trong ConfigMap có một helper Pod dùng image `docker.io/library/busybox` **không pin tag** — đó là
lựa chọn của upstream, lab giữ nguyên manifest chứ không sửa. Helper Pod này là thứ thật sự
`mkdir` và `rm -rf` thư mục volume trên node. Node phải kéo được image đó, xem mục 4 nếu môi
trường cô lập.

**PASS:** đủ ba dòng `PASS:`.

### B3.2. Apply và chờ provisioner chạy

```bash
kubectl apply -f ~/lab-work/6a/local-path-storage.yaml
kubectl -n local-path-storage rollout status deploy/local-path-provisioner --timeout=300s
kubectl -n local-path-storage get pods -o wide

IMG="$(kubectl -n local-path-storage get deploy local-path-provisioner \
  -o jsonpath='{.spec.template.spec.containers[0].image}')"
READY="$(kubectl -n local-path-storage get deploy local-path-provisioner \
  -o jsonpath='{.status.readyReplicas}')"
echo "image đang chạy = $IMG | readyReplicas = ${READY:-0}"

test "$IMG" = "docker.io/rancher/local-path-provisioner:${LPP_VERSION}" \
  && test "${READY:-0}" -ge 1 \
  && echo 'PASS: provisioner đang chạy đúng version đã khóa'

kubectl get storageclass \
  -o custom-columns=NAME:.metadata.name,PROVISIONER:.provisioner,RECLAIM:.reclaimPolicy,BINDING:.volumeBindingMode,EXPAND:.allowVolumeExpansion \
  | tee ~/lab-evidence/6a/b3-storageclass.txt
```

**Ý nghĩa:** bảng cuối là toàn bộ "hợp đồng" mà StorageClass công bố cho người dùng, đúng bốn
trường mà bài 96 bảo phải nắm. `EXPAND` in `<none>` nghĩa là `allowVolumeExpansion` không được đặt —
B4.5 khai thác đúng chỗ này.

**PASS:** dòng `PASS: provisioner đang chạy đúng version đã khóa` xuất hiện; bảng StorageClass có
đúng một dòng `local-path` với `rancher.io/local-path`, `Delete`, `WaitForFirstConsumer`.

### B3.3. Đánh dấu class mặc định

```bash
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get storageclass

DEF="$(kubectl get sc local-path \
  -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}')"
DEF_N="$(kubectl get sc -o json | grep -c '"storageclass.kubernetes.io/is-default-class": "true"')"
echo "annotation trên local-path = ${DEF:-<không có>} | số class mang cờ mặc định = $DEF_N"

test "$DEF" = 'true' && test "$DEF_N" -eq 1 \
  && echo 'PASS: đúng một StorageClass được đánh dấu mặc định'
```

**Ý nghĩa:** bài 96 khuyến nghị cluster chỉ nên có **một** class mặc định; nhiều class mặc định thì
Kubernetes lấy cái **được tạo gần đây nhất**, và đó là hành vi dành cho lúc chuyển đổi chứ không
phải trạng thái bình thường. Gate đếm số class mang cờ để bắt lỗi này ngay từ đầu.

Bài 98 nêu **hai** điều kiện cho hành vi mặc định, không phải một: đánh dấu class, **và** admission
controller `DefaultStorageClass` phải bật trên API server. Điều kiện thứ hai không kiểm bằng
annotation được — B3.4 và B3.5 kiểm nó bằng hành vi thật.

**PASS:** dòng `PASS: đúng một StorageClass được đánh dấu mặc định` xuất hiện; `kubectl get sc` in
`local-path (default)`.

### B3.4. Ba PVC cũ, ba số phận khác nhau

```bash
for i in $(seq 1 30); do
  NC_CLASS="$(kubectl get pvc pvc-no-class -n lab-6a -o jsonpath='{.spec.storageClassName}')"
  test -n "$NC_CLASS" && break
  sleep 2
done

STATIC_JSON="$(kubectl get pvc pvc-static -n lab-6a -o json)"
S2_JSON="$(kubectl get pvc pvc-static-2 -n lab-6a -o json)"
S2_PHASE="$(kubectl get pvc pvc-static-2 -n lab-6a -o jsonpath='{.status.phase}')"

echo "pvc-no-class.spec.storageClassName = ${NC_CLASS:-<vẫn trống>}"
echo "$STATIC_JSON" | grep -c '"storageClassName": ""' | sed 's/^/pvc-static giữ chuỗi rỗng: /'
echo "$S2_JSON"     | grep -c '"storageClassName": ""' | sed 's/^/pvc-static-2 giữ chuỗi rỗng: /'
echo "pvc-static-2 phase = $S2_PHASE"

test "$NC_CLASS" = 'local-path' \
  && echo "$STATIC_JSON" | grep -q '"storageClassName": ""' \
  && echo "$S2_JSON" | grep -q '"storageClassName": ""' \
  && test "$S2_PHASE" = 'Pending' \
  && echo 'PASS: hồi tố chỉ chạm PVC bỏ trống class, không chạm PVC khai chuỗi rỗng'
```

**Ý nghĩa:** ba PVC tạo ở B2 khi cluster chưa có class mặc định, giờ rẽ ba hướng:

- `pvc-no-class` **bỏ trống** `storageClassName` → control plane **gán hồi tố** class mặc định vào
  spec. Đây là mục *Gán StorageClass mặc định hồi tố* của bài 92 ở dạng quan sát được.
- `pvc-static` khai `""` và đã `Bound` vào PV không class → **không bị sửa**.
- `pvc-static-2` khai `""` và chưa bind → vẫn `Pending` **vĩnh viễn**, vì `""` tự vô hiệu hóa cấp
  phát động. Có provisioner rồi cũng không cứu được nó.

Chính vì có `pvc-no-class` bị sửa mà bạn biết admission controller `DefaultStorageClass` đang bật —
điều kiện thứ hai của bài 98.

**PASS:** dòng `PASS: hồi tố chỉ chạm PVC bỏ trống class…` xuất hiện.

### B3.5. PVC mới không khai class cũng được điền

```bash
kubectl create -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-default-probe
  namespace: lab-6a
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 128Mi
EOF

PROBE="$(kubectl get pvc pvc-default-probe -n lab-6a -o jsonpath='{.spec.storageClassName}')"
echo "storageClassName được admission controller điền = ${PROBE:-<trống>}"

test "$PROBE" = 'local-path' \
  && echo 'PASS: admission controller DefaultStorageClass điền class ngay lúc tạo'

kubectl delete pvc pvc-default-probe pvc-no-class -n lab-6a --wait=true
```

**Ý nghĩa:** khác với B3.4, ở đây trường được điền **ngay lúc object được tạo**, không phải do vòng
lặp hồi tố. Đó là hai cơ chế khác nhau cùng phục vụ một mục tiêu, và cả hai chỉ hoạt động khi
`DefaultStorageClass` bật.

Hai PVC bị xóa ngay: chúng đã làm xong việc, và giữ lại thì B4 khó đếm PV.

**PASS:** dòng `PASS: admission controller DefaultStorageClass điền class ngay lúc tạo` xuất hiện.

---

## B4. Cấp phát động và `volumeBindingMode`

**Mục đích:** từ đây bạn đóng vai người dùng cluster: chỉ tạo PVC, không bao giờ tạo PV nữa.

### B4.1. PVC không có Pod tiêu thụ thì đứng `Pending`

```bash
cat > ~/lab-work/6a/pvc-dyn.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dyn
  namespace: lab-6a
spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 128Mi
EOF

kubectl apply -f ~/lab-work/6a/pvc-dyn.yaml
sleep 20

DYN_PHASE="$(kubectl get pvc pvc-dyn -n lab-6a -o jsonpath='{.status.phase}')"
PV_N1="$(kubectl get pv --no-headers | wc -l)"
echo "pvc-dyn = $DYN_PHASE | tổng số PV = $PV_N1"

kubectl get events -n lab-6a --field-selector \
  involvedObject.kind=PersistentVolumeClaim,involvedObject.name=pvc-dyn \
  -o custom-columns=REASON:.reason,MESSAGE:.message | tee ~/lab-evidence/6a/b4-wffc-events.txt

test "$DYN_PHASE" = 'Pending' && test "$PV_N1" -eq 1 \
  && grep -q 'WaitForFirstConsumer' ~/lab-evidence/6a/b4-wffc-events.txt \
  && echo 'PASS: WaitForFirstConsumer giữ PVC ở Pending và chưa cấp phát gì'
```

**Ý nghĩa:** class đúng, provisioner chạy, dung lượng thừa — nhưng không có gì được tạo. Bài 96 mô
tả `WaitForFirstConsumer` là chế độ **trì hoãn việc gắn kết và cấp phát cho tới khi một Pod dùng
PVC được tạo**, để scheduler được cân nhắc mọi ràng buộc của Pod trước khi volume bị đóng đinh vào
một node. Event `WaitForFirstConsumer` là lời giải thích mà controller tự ghi ra.

`PV_N1` vẫn bằng 1 vì PV duy nhất trong cluster lúc này là `lab-6a-static` của B2.

**PASS:** dòng `PASS: WaitForFirstConsumer giữ PVC ở Pending và chưa cấp phát gì` xuất hiện.

### B4.2. Một Pod xuất hiện, một PV được sinh ra

```bash
cat > ~/lab-work/6a/pod-dyn.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dyn-a
  namespace: lab-6a
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker1
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-dyn
EOF

kubectl apply -f ~/lab-work/6a/pod-dyn.yaml
kubectl wait --for=condition=Ready pod/dyn-a -n lab-6a --timeout=300s

PV_DYN="$(kubectl get pvc pvc-dyn -n lab-6a -o jsonpath='{.spec.volumeName}')"
PROV_BY="$(kubectl get pv "$PV_DYN" \
  -o jsonpath='{.metadata.annotations.pv\.kubernetes\.io/provisioned-by}')"
PV_RECLAIM="$(kubectl get pv "$PV_DYN" -o jsonpath='{.spec.persistentVolumeReclaimPolicy}')"
PV_NODE="$(kubectl get pv "$PV_DYN" \
  -o jsonpath='{.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}')"
PV_PATH="$(kubectl get pv "$PV_DYN" -o jsonpath='{.spec.hostPath.path}{.spec.local.path}')"
echo "PV=$PV_DYN"
echo "provisioned-by=$PROV_BY | reclaimPolicy=$PV_RECLAIM"
echo "nodeAffinity=$PV_NODE | path=$PV_PATH"

test -n "$PV_DYN" && test "$PROV_BY" = 'rancher.io/local-path' \
  && test "$PV_RECLAIM" = 'Delete' && test "$PV_NODE" = 'lab-k8s-worker1' \
  && echo 'PASS: PV được sinh tự động, kế thừa reclaimPolicy của class, và bị ghim vào đúng node'
```

**Ý nghĩa:** bốn điều cùng lúc, và mỗi điều là một câu trong tài liệu:

- **PV xuất hiện mà không ai tạo tay** — bài 98: volume được tạo khi người dùng tạo PVC. Tên PV có
  dạng `pvc-<uid của PVC>`, do provisioner đặt chứ không phải bạn.
- **`provisioned-by`** ghi tên provisioner đã cấp phát; đây là cách đọc "PV này từ đâu ra".
- **`reclaimPolicy: Delete`** — PV cấp phát động **kế thừa** policy của StorageClass, mặc định là
  `Delete` (bài 92 và 96). Bạn không khai nó ở đâu cả.
- **`nodeAffinity` trỏ đúng node mà scheduler đã chọn cho Pod** — đây là phần mà PV `hostPath` tự
  viết ở B2 **không có**, và là lý do `WaitForFirstConsumer` tồn tại: thứ tự đúng phải là chọn node
  trước, cấp phát sau.

**PASS:** dòng `PASS: PV được sinh tự động, kế thừa reclaimPolicy của class…` xuất hiện.

### B4.3. Volume đó thật sự là gì trên đĩa của node

Đọc đường dẫn từ cluster, đừng gõ tay:

```bash
echo "$PV_PATH" | tee ~/lab-evidence/6a/b4-pv-path.txt
kubectl exec -n lab-6a dyn-a -- sh -c 'echo lab-6a-dyn-payload > /data/marker; ls -l /data'
```

Chạy trên **`lab-k8s-worker1`** — thay `<PV_PATH>` bằng đúng chuỗi in ra ở trên:

```bash
sudo ls -ld /opt/local-path-provisioner
sudo ls -l <PV_PATH>
sudo cat <PV_PATH>/marker
```

**PASS:** thư mục `<PV_PATH>` tồn tại trên `lab-k8s-worker1`, nằm dưới `/opt/local-path-provisioner`,
và `marker` chứa đúng `lab-6a-dyn-payload`. Tên thư mục có dạng `<tên PV>_<namespace>_<tên PVC>` —
đó là quy ước của provisioner, không phải của Kubernetes.

Quay lại **`lab-k8s-master`** và lặp lại phép thử của B1.3 lần thứ ba:

```bash
kubectl delete pod dyn-a -n lab-6a --wait=true
kubectl apply -f ~/lab-work/6a/pod-dyn.yaml
kubectl wait --for=condition=Ready pod/dyn-a -n lab-6a --timeout=300s

AFTER_DYN="$(kubectl exec -n lab-6a dyn-a -- cat /data/marker)"
PV_STILL="$(kubectl get pvc pvc-dyn -n lab-6a -o jsonpath='{.spec.volumeName}')"
echo "marker sau khi xóa Pod = $AFTER_DYN | PVC vẫn bind vào $PV_STILL"

test "$AFTER_DYN" = 'lab-6a-dyn-payload' && test "$PV_STILL" = "$PV_DYN" \
  && echo 'PASS: xóa Pod không đụng tới PVC, PV hay dữ liệu'
```

**Ý nghĩa:** ba lần cùng một phép thử, ba kết quả: `emptyDir` mất sạch (B1.3), PV tĩnh giữ nguyên
(B2.4), PV động giữ nguyên (ở đây). Khác biệt không nằm ở Pod mà ở **vòng đời của volume**.

**PASS:** dòng `PASS: xóa Pod không đụng tới PVC, PV hay dữ liệu` xuất hiện.

### B4.4. `Immediate` với backend gắn với node: bind sớm là hỏng sớm

```bash
cat > ~/lab-work/6a/sc-immediate.yaml <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: lab-6a-immediate
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF

kubectl apply -f ~/lab-work/6a/sc-immediate.yaml

kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-immediate
  namespace: lab-6a
spec:
  storageClassName: lab-6a-immediate
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 128Mi
EOF

sleep 30
IMM_PHASE="$(kubectl get pvc pvc-immediate -n lab-6a -o jsonpath='{.status.phase}')"
IMM_WARN="$(kubectl get events -n lab-6a --field-selector \
  involvedObject.kind=PersistentVolumeClaim,involvedObject.name=pvc-immediate,type=Warning \
  --no-headers 2>/dev/null | wc -l)"
echo "pvc-immediate = $IMM_PHASE | số event Warning = $IMM_WARN"

kubectl describe pvc pvc-immediate -n lab-6a | tail -n 10 \
  | tee ~/lab-evidence/6a/b4-immediate.txt

test "$IMM_PHASE" = 'Pending' && test "$IMM_WARN" -ge 1 \
  && echo 'PASS: Immediate cấp phát ngay và thất bại ngay vì chưa biết node nào'

kubectl delete pvc pvc-immediate -n lab-6a --wait=true
kubectl delete storageclass lab-6a-immediate
```

**Ý nghĩa:** `Immediate` là **giá trị mặc định** khi `volumeBindingMode` không được đặt, và bài 96
nói rõ hệ quả: *"Với các backend lưu trữ bị ràng buộc theo topology và không thể truy cập toàn cục
từ mọi Node, các PersistentVolume sẽ được gắn kết hoặc cấp phát mà không biết gì về các yêu cầu
lập lịch của Pod. Điều này có thể dẫn đến các Pod không thể lập lịch được."*

Ở đây provisioner được gọi ngay khi PVC ra đời, nhưng nó cần biết **cấp thư mục trên node nào** —
mà chưa có Pod thì chưa có node. Nó ghi event lỗi và PVC nằm lại `Pending`. So sánh với B4.1:
cùng một trạng thái `Pending`, hai nguyên nhân hoàn toàn khác nhau, và `kubectl describe` là chỗ
phân biệt.

Chế độ `Immediate` không sai — nó đúng cho backend truy cập được từ mọi node, ví dụ NFS. Sai là
dùng nó cho lưu trữ cục bộ.

**PASS:** dòng `PASS: Immediate cấp phát ngay và thất bại ngay…` xuất hiện.

### B4.5. Không có `allowVolumeExpansion` thì API server từ chối tăng dung lượng

```bash
EXP="$(kubectl get sc local-path -o jsonpath='{.allowVolumeExpansion}')"
SIZE_BEFORE="$(kubectl get pvc pvc-dyn -n lab-6a -o jsonpath='{.spec.resources.requests.storage}')"
echo "allowVolumeExpansion của local-path = ${EXP:-<không đặt>} | size hiện tại = $SIZE_BEFORE"

PATCH_OUT="$(kubectl patch pvc pvc-dyn -n lab-6a \
  -p '{"spec":{"resources":{"requests":{"storage":"256Mi"}}}}' 2>&1)"
RC_PATCH=$?
echo "$PATCH_OUT" | tail -n 2 | tee ~/lab-evidence/6a/b4-expand.txt

SIZE_AFTER="$(kubectl get pvc pvc-dyn -n lab-6a -o jsonpath='{.spec.resources.requests.storage}')"
echo "mã trả về của patch = $RC_PATCH | size sau khi patch = $SIZE_AFTER"

test -z "$EXP" && test "$RC_PATCH" -ne 0 && test "$SIZE_AFTER" = "$SIZE_BEFORE" \
  && echo 'PASS: class không cho mở rộng thì PVC không sửa được dung lượng'
```

**Ý nghĩa:** bài 92 nói *"Bạn chỉ có thể mở rộng một PVC nếu trường `allowVolumeExpansion` của
storage class của nó được đặt là true"*, và thao tác mở rộng là **sửa PVC**, không phải sửa PV.
Ở đây class không đặt trường đó, nên yêu cầu bị chặn ngay ở tầng API — không có PV mới nào được
tạo, dung lượng không đổi.

Đây cũng là ranh giới của provisioner mà lab chọn: nó cấp một thư mục, không phải một thiết bị khối
có thể resize. Muốn thực hành mở rộng thật thì phải có CSI driver hỗ trợ, cùng lớp điều kiện với
nợ #5 của Lab 6b.

**PASS:** dòng `PASS: class không cho mở rộng thì PVC không sửa được dung lượng` xuất hiện.

---

## B5. `reclaimPolicy` — `Delete` so với `Retain`

**Mục đích:** trả lời đúng một câu hỏi: **xóa PVC thì chuyện gì xảy ra với dữ liệu?** Bài
[92](../92-persistent-volumes-vi.md#reclaiming) cho hai câu trả lời khác nhau tùy `reclaimPolicy`,
và mục này chạy cả hai trên cùng một cluster để so.

### B5.1. PVC đang được Pod dùng thì không xóa được ngay

```bash
kubectl delete pvc pvc-dyn -n lab-6a --wait=false

DEL_TS="$(kubectl get pvc pvc-dyn -n lab-6a -o jsonpath='{.metadata.deletionTimestamp}')"
FINAL="$(kubectl get pvc pvc-dyn -n lab-6a -o jsonpath='{.metadata.finalizers[*]}')"
POD_PHASE="$(kubectl get pod dyn-a -n lab-6a -o jsonpath='{.status.phase}')"
echo "deletionTimestamp = ${DEL_TS:-<rỗng>}"
echo "finalizers = ${FINAL:-<rỗng>}"
echo "Pod dyn-a = $POD_PHASE"

test -n "$DEL_TS" && test "$FINAL" = 'kubernetes.io/pvc-protection' \
  && test "$POD_PHASE" = 'Running' \
  && echo 'PASS: PVC bị hoãn xóa vì còn Pod dùng, Pod không bị ảnh hưởng'
```

**Ý nghĩa:** đây là *Bảo vệ đối tượng lưu trữ đang được sử dụng* của bài 92. Lệnh `delete` đã ghi
`deletionTimestamp`, nhưng finalizer `kubernetes.io/pvc-protection` giữ object lại cho tới khi
không còn Pod nào dùng nó. Mục đích duy nhất là chống mất dữ liệu do gõ nhầm. Bài 92 nhấn mạnh:
PVC được coi là đang dùng khi **tồn tại một Pod** tham chiếu nó — Pod không cần đang chạy.

**PASS:** dòng `PASS: PVC bị hoãn xóa vì còn Pod dùng…` xuất hiện.

### B5.2. `Delete`: xóa claim là mất cả PV lẫn dữ liệu

```bash
kubectl delete pod dyn-a -n lab-6a --wait=true

for i in $(seq 1 90); do
  kubectl get pvc pvc-dyn -n lab-6a >/dev/null 2>&1 || break
  sleep 2
done
for i in $(seq 1 90); do
  kubectl get pv "$PV_DYN" >/dev/null 2>&1 || break
  sleep 2
done

PVC_LEFT="$(kubectl get pvc pvc-dyn -n lab-6a --ignore-not-found -o name | wc -l)"
PV_LEFT="$(kubectl get pv "$PV_DYN" --ignore-not-found -o name | wc -l)"
echo "pvc-dyn còn lại = $PVC_LEFT | PV $PV_DYN còn lại = $PV_LEFT"

test "$PVC_LEFT" -eq 0 && test "$PV_LEFT" -eq 0 \
  && echo 'PASS: xóa Pod xong thì PVC đi, và PV đi theo vì reclaimPolicy là Delete'
```

Chạy trên **`lab-k8s-worker1`**, với `<PV_PATH>` là đường dẫn đã ghi ở
`~/lab-evidence/6a/b4-pv-path.txt`:

```bash
sudo test -d <PV_PATH> && echo 'FAIL: thư mục volume vẫn còn' || echo 'PASS: thư mục volume đã bị xóa khỏi node'
sudo ls -l /opt/local-path-provisioner
```

**Ý nghĩa:** bài 98 tóm gọn: *"Khi claim bị xóa, volume sẽ bị hủy."* Chuỗi thật dài hơn thế — xóa
Pod → finalizer PVC được gỡ → PVC biến mất → PV mất claim → reclaim policy `Delete` chạy →
provisioner tạo một helper Pod để `rm -rf` thư mục trên node → PV bị xóa khỏi API. Đây cũng là lý
do bài 92 dành hẳn một mục cho finalizer bảo vệ việc xóa PV: đối tượng API chỉ được gỡ **sau khi**
phần lưu trữ nền đã bị dọn.

**PASS:** dòng `PASS: xóa Pod xong thì PVC đi…` trên master và dòng `PASS: thư mục volume đã bị xóa
khỏi node` trên worker1.

### B5.3. Một StorageClass `Retain`

```bash
cat > ~/lab-work/6a/sc-retain.yaml <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: lab-6a-retain
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
EOF

cat > ~/lab-work/6a/keep.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-keep
  namespace: lab-6a
spec:
  storageClassName: lab-6a-retain
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 128Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: keep-a
  namespace: lab-6a
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker1
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-keep
EOF

kubectl apply -f ~/lab-work/6a/sc-retain.yaml
kubectl apply -f ~/lab-work/6a/keep.yaml
kubectl wait --for=condition=Ready pod/keep-a -n lab-6a --timeout=300s

kubectl exec -n lab-6a keep-a -- sh -c 'echo lab-6a-keep-payload > /data/marker'

PV_RETAIN="$(kubectl get pvc pvc-keep -n lab-6a -o jsonpath='{.spec.volumeName}')"
RET_POLICY="$(kubectl get pv "$PV_RETAIN" -o jsonpath='{.spec.persistentVolumeReclaimPolicy}')"
RET_PATH="$(kubectl get pv "$PV_RETAIN" -o jsonpath='{.spec.hostPath.path}{.spec.local.path}')"
echo "PV=$PV_RETAIN | reclaimPolicy=$RET_POLICY"
echo "$RET_PATH" | tee ~/lab-evidence/6a/b5-retain-path.txt

test "$RET_POLICY" = 'Retain' \
  && echo 'PASS: PV kế thừa reclaimPolicy Retain từ class đã cấp phát nó'
```

**Ý nghĩa:** bạn không khai `Retain` ở đâu trong PVC hay Pod — nó đến từ **StorageClass**. Bài 96:
*"Các PersistentVolume được tạo động bởi một StorageClass sẽ có chính sách thu hồi được chỉ định
trong trường `reclaimPolicy` của lớp đó."* Đó là công cụ để quản trị viên cấu hình class *theo kỳ
vọng của người dùng*, thay vì bắt từng người sửa từng PV.

**PASS:** dòng `PASS: PV kế thừa reclaimPolicy Retain từ class đã cấp phát nó` xuất hiện.

### B5.4. Xóa PVC: PV chuyển sang `Released`, dữ liệu ở lại

```bash
kubectl delete pod keep-a -n lab-6a --wait=true
kubectl delete pvc pvc-keep -n lab-6a --wait=true

for i in $(seq 1 60); do
  RET_PHASE="$(kubectl get pv "$PV_RETAIN" -o jsonpath='{.status.phase}')"
  test "$RET_PHASE" = 'Released' && break
  sleep 2
done

STALE_NS="$(kubectl get pv "$PV_RETAIN" -o jsonpath='{.spec.claimRef.namespace}')"
STALE_NAME="$(kubectl get pv "$PV_RETAIN" -o jsonpath='{.spec.claimRef.name}')"
echo "PV $PV_RETAIN: phase=$RET_PHASE | claimRef còn trỏ tới $STALE_NS/$STALE_NAME"

test "$RET_PHASE" = 'Released' && test "$STALE_NAME" = 'pvc-keep' \
  && echo 'PASS: PV ở pha Released và vẫn giữ claimRef của claim đã bị xóa'
```

Chạy trên **`lab-k8s-worker1`**, với `<RET_PATH>` lấy từ `~/lab-evidence/6a/b5-retain-path.txt`:

```bash
sudo cat <RET_PATH>/marker
```

**Ý nghĩa:** so với B5.2 chỉ đổi đúng một trường của StorageClass, và kết quả ngược hẳn: PVC biến
mất nhưng PV còn, dữ liệu còn nguyên trên đĩa node. Bài 92 gọi trạng thái này là *"đã giải phóng"*
và nói rõ nó **chưa sẵn sàng cho một claim khác vì dữ liệu của người claim trước vẫn còn trên
volume**. `claimRef` giữ nguyên chính là cơ chế khóa đó.

**PASS:** dòng `PASS: PV ở pha Released và vẫn giữ claimRef…` trên master; trên worker1
`marker` in ra `lab-6a-keep-payload`.

### B5.5. `Released` không tự tái dùng được

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-reuse
  namespace: lab-6a
spec:
  storageClassName: lab-6a-retain
  volumeName: $PV_RETAIN
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 128Mi
EOF

sleep 30
REUSE_PHASE="$(kubectl get pvc pvc-reuse -n lab-6a -o jsonpath='{.status.phase}')"
PHASE_NOW="$(kubectl get pv "$PV_RETAIN" -o jsonpath='{.status.phase}')"
echo "pvc-reuse = $REUSE_PHASE | PV $PV_RETAIN = $PHASE_NOW"

test "$REUSE_PHASE" = 'Pending' && test "$PHASE_NOW" = 'Released' \
  && echo 'PASS: claim mới không cướp được PV đã Released'

kubectl delete pvc pvc-reuse -n lab-6a --wait=true
```

**Ý nghĩa:** PVC này chỉ đích danh PV bằng `volumeName`, đúng class, đúng dung lượng, đúng access
mode — vẫn không bind được. Bài 92 mô tả chính xác tình huống: *"Nếu PV được chỉ định đã ràng buộc
với một PVC khác, việc ràng buộc sẽ bị kẹt ở trạng thái pending."* `claimRef` cũ vẫn nằm đó, và
không ai gỡ nó thay bạn.

**PASS:** dòng `PASS: claim mới không cướp được PV đã Released` xuất hiện.

### B5.6. Thu hồi thủ công đúng ba bước

Bài 92 liệt kê ba bước cho `Retain`, và ba bước đó phải làm bằng tay theo đúng thứ tự:

```bash
# Bước 1: xóa đối tượng PersistentVolume. Tài sản lưu trữ bên ngoài KHÔNG bị xóa theo.
kubectl delete pv "$PV_RETAIN" --wait=true
kubectl get pv
```

Chạy trên **`lab-k8s-worker1`** — bước 2 và 3, với `<RET_PATH>` như trên:

```bash
# Bước 2: dữ liệu vẫn còn sau khi đối tượng PV đã biến mất — kiểm tra rồi mới dọn.
sudo cat <RET_PATH>/marker

# Bước 3: tự xóa tài sản lưu trữ liên kết.
sudo rm -rf <RET_PATH>
sudo test -d <RET_PATH> && echo 'FAIL: thư mục vẫn còn' || echo 'PASS: đã dọn tay tài sản lưu trữ'
sudo ls -l /opt/local-path-provisioner
```

Quay lại **`lab-k8s-master`**:

```bash
PV_TOTAL2="$(kubectl get pv --no-headers | wc -l)"
echo "tổng số PV còn lại = $PV_TOTAL2"
test "$PV_TOTAL2" -eq 1 && echo 'PASS: chỉ còn PV tĩnh lab-6a-static'
```

**Ý nghĩa:** bước 2 là thứ phân biệt `Retain` với `Delete`: có một khoảnh khắc mà đối tượng API đã
biến mất nhưng dữ liệu vẫn nằm trên đĩa, và đó chính là cơ hội để bạn sao lưu trước khi dọn. Bài 92
còn nói: muốn tái sử dụng đúng tài sản lưu trữ đó thì tạo một PersistentVolume **mới** với cùng
định nghĩa — không có cách nào "hồi sinh" PV cũ.

**PASS:** hai dòng `PASS:` trên worker1 và master.

---

## B6. `volumeClaimTemplates` — trả nợ #2

> **Đọc lại trước khi làm:** bài [65 — StatefulSet](../65-statefulset-vi.md), ba mục *Template cho
> volume claim*, *Lưu trữ ổn định* và *Lưu giữ PersistentVolumeClaim*. Ở Lab 4b bạn đã chạy
> StatefulSet **không có** `volumeClaimTemplates` vì cluster chưa có StorageClass; mục này là phần
> còn thiếu của bài 65, và cũng là phần lưu trữ của bài
> [343](../343-run-replicated-stateful-application-vi.md).

### B6.1. StatefulSet ba replica, mỗi replica một claim

```bash
cat > ~/lab-work/6a/statefulset.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web-headless
  namespace: lab-6a
spec:
  clusterIP: None
  selector:
    app: web
  ports:
  - name: web
    port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: lab-6a
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
      terminationGracePeriodSeconds: 5
      containers:
      - name: app
        image: busybox:1.37
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: www
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 128Mi
EOF

kubectl apply --dry-run=server --validate=strict -f ~/lab-work/6a/statefulset.yaml
kubectl apply -f ~/lab-work/6a/statefulset.yaml
kubectl rollout status statefulset/web -n lab-6a --timeout=600s
kubectl get pods,pvc -n lab-6a -o wide | tee ~/lab-evidence/6a/b6-sts.txt
```

**Ý nghĩa của những thứ *không* có trong manifest:**

- **`volumeClaimTemplates` không khai `storageClassName`.** Bài 65: *"Nếu không chỉ định
  StorageClass, StorageClass mặc định sẽ được dùng."* Đây là chỗ B3.3 trả cổ tức.
- **Không có PV nào được tạo tay.** Bài 65 nêu hai điều kiện để `volumeClaimTemplates` cấp được lưu
  trữ ổn định; cluster lab thỏa điều kiện thứ nhất — class dùng cấp phát động.
- **Service `web-headless` là thật**, không phải tên trỏ vào hư không như ở Lab 4b: nợ #3 đã được
  trả ở Lab 5a.

**PASS:** `--dry-run=server` không lỗi validation; `rollout status` báo `statefulset rolling update
complete 3 pods...`.

### B6.2. Quy tắc tên `<claim>-<statefulset>-<ordinal>`

```bash
kubectl get pvc -n lab-6a -o name | sort | tee ~/lab-evidence/6a/b6-pvc-names.txt

PVC_LIST="$(kubectl get pvc -n lab-6a --no-headers -o custom-columns=N:.metadata.name \
  | grep '^www-web-' | sort | paste -sd' ' -)"
echo "PVC do volumeClaimTemplates sinh ra: $PVC_LIST"

BOUND_N=0
for p in www-web-0 www-web-1 www-web-2; do
  PH="$(kubectl get pvc "$p" -n lab-6a -o jsonpath='{.status.phase}')"
  VOL="$(kubectl get pvc "$p" -n lab-6a -o jsonpath='{.spec.volumeName}')"
  SC="$(kubectl get pvc "$p" -n lab-6a -o jsonpath='{.spec.storageClassName}')"
  NODE="$(kubectl get pod "${p#www-}" -n lab-6a -o jsonpath='{.spec.nodeName}')"
  echo "$p | $PH | class=$SC | PV=$VOL | Pod ở node $NODE"
  test "$PH" = 'Bound' && BOUND_N=$((BOUND_N + 1))
done

UNIQ_PV="$(kubectl get pvc -n lab-6a -o jsonpath='{range .items[*]}{.spec.volumeName}{"\n"}{end}' \
  | grep '^pvc-' | sort -u | wc -l)"
echo "số PVC Bound = $BOUND_N | số PV riêng biệt = $UNIQ_PV"

test "$PVC_LIST" = 'www-web-0 www-web-1 www-web-2' && test "$BOUND_N" -eq 3 \
  && test "$UNIQ_PV" -eq 3 \
  && echo 'PASS: mỗi replica có PVC riêng, tên theo <claim>-<statefulset>-<ordinal>, PV riêng'
```

**Ý nghĩa:** đây là trọng tâm của nợ #2. Một `volumeClaimTemplates` duy nhất trong manifest sinh ra
**ba** PVC thật, tên ghép từ tên claim template, tên StatefulSet và ordinal của Pod. Bài 65 mô tả
đúng cơ chế: *"Với mỗi mục VolumeClaimTemplate được định nghĩa trong một StatefulSet, mỗi Pod nhận
một PersistentVolumeClaim."* Không có Pod nào dùng chung volume với Pod khác — đó là phần "lưu trữ
ổn định" trong định danh của Pod StatefulSet, bên cạnh tên và định danh mạng đã học ở Lab 4b.

**PASS:** dòng `PASS: mỗi replica có PVC riêng…` xuất hiện.

### B6.3. Dữ liệu riêng cho từng ordinal

```bash
for i in 0 1 2; do
  kubectl exec -n lab-6a "web-$i" -- sh -c "echo payload-cua-web-$i > /data/who"
done

for i in 0 1 2; do
  echo "web-$i đọc được: $(kubectl exec -n lab-6a "web-$i" -- cat /data/who)"
done

OK=0
for i in 0 1 2; do
  V="$(kubectl exec -n lab-6a "web-$i" -- cat /data/who)"
  test "$V" = "payload-cua-web-$i" && OK=$((OK + 1))
done
test "$OK" -eq 3 && echo 'PASS: ba volume độc lập, không Pod nào thấy dữ liệu của Pod khác'
```

**PASS:** dòng `PASS: ba volume độc lập…` xuất hiện.

### B6.4. Thu nhỏ: Pod biến mất, PVC ở lại

```bash
kubectl scale statefulset web -n lab-6a --replicas=1
kubectl rollout status statefulset/web -n lab-6a --timeout=300s

POD_N="$(kubectl get pods -n lab-6a -l app=web --no-headers | wc -l)"
PVC_N="$(kubectl get pvc -n lab-6a --no-headers | grep -c '^www-web-')"
W1="$(kubectl get pvc www-web-1 -n lab-6a -o jsonpath='{.status.phase}')"
W2="$(kubectl get pvc www-web-2 -n lab-6a -o jsonpath='{.status.phase}')"
echo "Pod còn lại = $POD_N | PVC còn lại = $PVC_N | www-web-1=$W1 | www-web-2=$W2"

WS="$(kubectl get sts web -n lab-6a \
  -o jsonpath='{.spec.persistentVolumeClaimRetentionPolicy.whenScaled}')"
WD="$(kubectl get sts web -n lab-6a \
  -o jsonpath='{.spec.persistentVolumeClaimRetentionPolicy.whenDeleted}')"
echo "persistentVolumeClaimRetentionPolicy: whenScaled=${WS:-<không đặt>} whenDeleted=${WD:-<không đặt>}"

test "$POD_N" -eq 1 && test "$PVC_N" -eq 3 && test "$W1" = 'Bound' && test "$W2" = 'Bound' \
  && test "$WS" != 'Delete' \
  && echo 'PASS: thu nhỏ xuống 1 replica nhưng ba PVC vẫn Bound'
```

**Ý nghĩa:** bài [343](../343-run-replicated-stateful-application-vi.md) ghi chú đúng điều này:
*"Mặc dù việc scale lên sẽ tự động tạo các PersistentVolumeClaim mới, việc scale xuống không tự
động xóa các PVC này."* Bài 65 giải thích vì sao: `persistentVolumeClaimRetentionPolicy` mặc định
là `Retain` cho cả `whenScaled` lẫn `whenDeleted`. Bạn được quyền chọn: giữ PVC để scale lên lại
cho nhanh, hay trích dữ liệu ra rồi mới xóa.

**PASS:** dòng `PASS: thu nhỏ xuống 1 replica nhưng ba PVC vẫn Bound` xuất hiện.

### B6.5. Mở rộng lại: Pod cũ gặp lại đúng dữ liệu cũ

```bash
kubectl scale statefulset web -n lab-6a --replicas=3
kubectl rollout status statefulset/web -n lab-6a --timeout=600s

BACK=0
for i in 0 1 2; do
  V="$(kubectl exec -n lab-6a "web-$i" -- cat /data/who)"
  echo "web-$i sau khi scale lại: $V"
  test "$V" = "payload-cua-web-$i" && BACK=$((BACK + 1))
done

PVC_N2="$(kubectl get pvc -n lab-6a --no-headers | grep -c '^www-web-')"
echo "số PVC = $PVC_N2 (không sinh thêm cái nào)"

test "$BACK" -eq 3 && test "$PVC_N2" -eq 3 \
  && echo 'PASS: mỗi ordinal nhận lại đúng PVC cũ và đúng dữ liệu cũ'
```

**Ý nghĩa:** `web-1` là một Pod **mới toanh** — `uid` khác, có thể ở node khác — nhưng nó nhận lại
`www-web-1`, tức đúng volume cũ. Bài 65: *"Khi một Pod được lập lịch (lại) lên một node, các
`volumeMounts` của nó sẽ mount các PersistentVolume gắn với các PersistentVolume Claim của nó."*
Đây là khác biệt cốt lõi giữa StatefulSet và Deployment: Deployment giữ **số lượng**, StatefulSet
giữ **danh tính**, và từ giai đoạn 6 trở đi danh tính đó bao gồm cả dữ liệu.

**PASS:** dòng `PASS: mỗi ordinal nhận lại đúng PVC cũ và đúng dữ liệu cũ` xuất hiện.

### B6.6. Xóa StatefulSet không xóa PVC

```bash
kubectl delete statefulset web -n lab-6a --wait=true
kubectl delete service web-headless -n lab-6a --wait=true

for i in $(seq 1 60); do
  POD_LEFT="$(kubectl get pods -n lab-6a -l app=web --no-headers 2>/dev/null | wc -l)"
  test "$POD_LEFT" -eq 0 && break
  sleep 2
done

PVC_N3="$(kubectl get pvc -n lab-6a --no-headers | grep -c '^www-web-')"
PV_N3="$(kubectl get pv --no-headers | grep -c 'lab-6a/www-web-')"
echo "Pod còn lại = $POD_LEFT | PVC còn lại = $PVC_N3 | PV còn lại của StatefulSet = $PV_N3"
kubectl get pvc -n lab-6a | tee ~/lab-evidence/6a/b6-pvc-con-lai.txt

test "$POD_LEFT" -eq 0 && test "$PVC_N3" -eq 3 && test "$PV_N3" -eq 3 \
  && echo 'PASS: StatefulSet biến mất, ba PVC và ba PV vẫn còn nguyên'
```

**Ý nghĩa:** bài 65 nói thẳng: *"các PersistentVolume gắn với các PersistentVolume Claim của các
Pod không bị xóa khi các Pod, hay StatefulSet bị xóa. Việc này phải được thực hiện thủ công."* Bài
[340](../340-delete-stateful-set-vi.md) lặp lại cùng lý do: *"Điều này nhằm đảm bảo rằng bạn có cơ
hội sao chép dữ liệu ra khỏi volume trước khi xóa nó."*

Đây là cái bẫy vận hành thật: xóa một StatefulSet mà quên PVC thì cluster âm thầm giữ lại volume và
tiếp tục tiêu đĩa của node.

**PASS:** dòng `PASS: StatefulSet biến mất, ba PVC và ba PV vẫn còn nguyên` xuất hiện.

### B6.7. Dọn tay đúng như bài 340 chỉ

```bash
kubectl delete pvc www-web-0 www-web-1 www-web-2 -n lab-6a --wait=true

for i in $(seq 1 90); do
  PV_LEFT3="$(kubectl get pv --no-headers | grep -c 'lab-6a/www-web-')"
  test "$PV_LEFT3" -eq 0 && break
  sleep 2
done

PVC_N4="$(kubectl get pvc -n lab-6a --no-headers 2>/dev/null | grep -c '^www-web-')"
echo "PVC còn = $PVC_N4 | PV của StatefulSet còn = $PV_LEFT3"

test "$PVC_N4" -eq 0 && test "$PV_LEFT3" -eq 0 \
  && echo 'PASS: xóa PVC bằng tay thì PV đi theo — nợ #2 đã trả xong'
```

**Ý nghĩa:** PV đi theo vì class mặc định dùng `reclaimPolicy: Delete`. Bài 340 cảnh báo đúng chỗ
này: *"Việc xóa PVC sau khi các Pod đã kết thúc có thể kích hoạt việc xóa các Persistent Volume
phía sau, tùy thuộc vào storage class và reclaim policy. Bạn không bao giờ nên giả định rằng mình
vẫn có khả năng truy cập một volume sau khi claim đã bị xóa."* Nếu StatefulSet này dùng
`lab-6a-retain` của B5.3 thì kết quả đã khác.

**PASS:** dòng `PASS: xóa PVC bằng tay thì PV đi theo — nợ #2 đã trả xong` xuất hiện.

---

## B7. Projected volume — nhiều nguồn, một thư mục

**Mục đích:** bài [93](../93-projected-volumes-vi.md) không nói về lưu trữ bền vững mà về cách gộp
nhiều nguồn dữ liệu của Kubernetes vào **một thư mục duy nhất** trong container. Bài
[277](../277-configure-projected-volume-vi.md) là bản thực hành của nó.

### B7.1. Đóng gói nguồn rồi gộp làm một

```bash
cd ~/lab-work/6a
echo -n "admin" > username.txt
echo -n "1f2d1e2e67df" > password.txt
kubectl create secret generic user -n lab-6a --from-file=./username.txt
kubectl create secret generic pass -n lab-6a --from-file=./password.txt
kubectl create configmap app-conf -n lab-6a --from-literal=mode=lab --from-literal=stage=6

cat > ~/lab-work/6a/projected.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: projected
  namespace: lab-6a
  labels:
    app: projected
    stage: "6a"
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      defaultMode: 0444
      sources:
      - secret:
          name: user
      - secret:
          name: pass
          items:
          - key: password.txt
            path: creds/password
            mode: 0400
      - configMap:
          name: app-conf
      - downwardAPI:
          items:
          - path: "meta/labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "meta/namespace"
            fieldRef:
              fieldPath: metadata.namespace
      - serviceAccountToken:
          audience: api
          expirationSeconds: 3600
          path: token
EOF

kubectl apply -f ~/lab-work/6a/projected.yaml
kubectl wait --for=condition=Ready pod/projected -n lab-6a --timeout=180s
kubectl exec -n lab-6a projected -- find /projected-volume -type f -o -type l \
  | sort | tee ~/lab-evidence/6a/b7-files.txt
```

**Ý nghĩa:** năm nguồn khác loại — hai Secret, một ConfigMap, downwardAPI và một token
ServiceAccount — cùng đổ vào `/projected-volume`. Bài 93 nhấn mạnh **mọi nguồn phải nằm cùng
namespace với Pod**; không có cách nào chiếu một Secret từ namespace khác.

Hai khác biệt cú pháp so với volume rời cũng nằm ngay trong manifest: secret dùng `name` chứ không
phải `secretName`, và `defaultMode` chỉ đặt được **ở cấp projected**, còn `mode` thì đặt được cho
từng projection.

**PASS:** Pod `Ready`; danh sách file chứa `username.txt`, `creds/password`, `mode`, `stage`,
`meta/labels`, `meta/namespace` và `token`.

### B7.2. Nội dung và quyền đúng như khai báo

```bash
U="$(kubectl exec -n lab-6a projected -- cat /projected-volume/username.txt)"
P="$(kubectl exec -n lab-6a projected -- cat /projected-volume/creds/password)"
M="$(kubectl exec -n lab-6a projected -- cat /projected-volume/mode)"
NS="$(kubectl exec -n lab-6a projected -- cat /projected-volume/meta/namespace)"
MODE_PASS="$(kubectl exec -n lab-6a projected -- stat -c '%a' /projected-volume/creds/password)"
MODE_USER="$(kubectl exec -n lab-6a projected -- stat -c '%a' /projected-volume/username.txt)"
DOTS="$(kubectl exec -n lab-6a projected -- sh -c "tr -cd '.' < /projected-volume/token | wc -c" \
  | tr -d ' ')"
kubectl exec -n lab-6a projected -- cat /projected-volume/meta/labels

echo "username=$U | password=$P | mode=$M | namespace=$NS"
echo "quyền: creds/password=$MODE_PASS, username.txt=$MODE_USER | số dấu chấm trong token=$DOTS"

test "$U" = 'admin' && test "$P" = '1f2d1e2e67df' && test "$M" = 'lab' \
  && test "$NS" = 'lab-6a' && test "$MODE_PASS" = '400' && test "$MODE_USER" = '444' \
  && test "$DOTS" -eq 2 \
  && echo 'PASS: năm nguồn hiện đúng nội dung, đúng đường dẫn và đúng quyền'
```

**Ý nghĩa:** `mode: 0400` của riêng một projection thắng `defaultMode: 0444` của cả volume — đúng
mô tả của bài 93. Token có đúng hai dấu chấm vì JWT gồm ba phần; đó là bằng chứng
`serviceAccountToken` đã tiêm token thật chứ không phải file rỗng. Bài 93 nêu ba trường cần nắm
của projection này: `audience`, `expirationSeconds` (mặc định 1 giờ, tối thiểu 600 giây) và `path`
tính tương đối so với điểm mount.

**PASS:** dòng `PASS: năm nguồn hiện đúng nội dung…` xuất hiện.

### B7.3. Mọi Pod đều đang dùng projected volume mà bạn không viết

```bash
kubectl get pod projected -n lab-6a \
  -o jsonpath='{range .spec.volumes[*]}{.name}{"\n"}{end}'
kubectl get pod projected -n lab-6a -o yaml \
  | grep -A6 'kube-api-access' | head -n 20 | tee ~/lab-evidence/6a/b7-sa-token.txt

AUTO="$(kubectl get pod projected -n lab-6a -o json \
  | grep -c 'kube-api-access')"
HAS_SAT="$(kubectl get pod projected -n lab-6a -o json | grep -c 'serviceAccountToken')"
echo "số lần xuất hiện kube-api-access = $AUTO | serviceAccountToken = $HAS_SAT"

test "$AUTO" -ge 1 && test "$HAS_SAT" -ge 2 \
  && echo 'PASS: volume kube-api-access-* cũng là một projected volume do API server tự thêm'
```

**Ý nghĩa:** ngoài volume bạn viết, Pod còn một volume `kube-api-access-<hậu tố>` được thêm tự
động, và nó cũng là `projected` với ba nguồn: `serviceAccountToken`, một `configMap` chứa CA của
cluster, và `downwardAPI` chứa namespace. Nói cách khác, cơ chế của bài 93 đã chạy trong mọi Pod
bạn tạo từ giai đoạn 3 tới giờ; hôm nay bạn chỉ mới gọi tên nó.

**PASS:** dòng `PASS: volume kube-api-access-* cũng là một projected volume…` xuất hiện.

### B7.4. `secretName` không tồn tại bên trong `projected`

```bash
cat > ~/lab-work/6a/projected-sai.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: projected-sai
  namespace: lab-6a
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 60"]
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          secretName: user
EOF

DRY_OUT="$(kubectl apply --dry-run=server --validate=strict \
  -f ~/lab-work/6a/projected-sai.yaml 2>&1)"
RC_DRY=$?
echo "$DRY_OUT" | tee ~/lab-evidence/6a/b7-secretname-sai.txt

if test "$RC_DRY" -ne 0; then
  echo 'PASS: API server từ chối secretName trong projected — trường đúng là name'
else
  echo 'FAIL: API server chấp nhận secretName trong projected'
fi

kubectl delete pod projected -n lab-6a --wait=true
kubectl delete secret user pass -n lab-6a
kubectl delete configmap app-conf -n lab-6a
rm -f ~/lab-work/6a/username.txt ~/lab-work/6a/password.txt
```

**Ý nghĩa:** đây là bẫy cú pháp mà bài 93 gọi tên: `secretName` là trường của volume `secret` rời,
còn bên trong `projected` thì trường đó là `name`. `--validate=strict` biến một lỗi im lặng thành
một lỗi phát hiện được ngay lúc `apply`, trước khi Pod chạy sai.

**PASS:** dòng `PASS: API server từ chối secretName trong projected…` xuất hiện.

---

## B8. Volume tạm thời tổng quát — `emptyDir` mà lại là PVC

**Mục đích:** bài [94](../94-ephemeral-volumes-vi.md) mô tả một loại volume khai **inline** trong
spec Pod nhưng lại sinh ra một PersistentVolumeClaim thật. Nó chỉ chạy được khi cluster có
provisioner — nên chỗ của nó là sau B3.

### B8.1. Khai inline, PVC hiện ra với tên xác định

```bash
cat > ~/lab-work/6a/ephemeral.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: eph-a
  namespace: lab-6a
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: scratch
      mountPath: /scratch
  volumes:
  - name: scratch
    ephemeral:
      volumeClaimTemplate:
        metadata:
          labels:
            type: lab-6a-scratch
        spec:
          accessModes: ["ReadWriteOnce"]
          storageClassName: local-path
          resources:
            requests:
              storage: 128Mi
EOF

kubectl apply -f ~/lab-work/6a/ephemeral.yaml
kubectl wait --for=condition=Ready pod/eph-a -n lab-6a --timeout=300s
kubectl get pvc -n lab-6a -l type=lab-6a-scratch | tee ~/lab-evidence/6a/b8-pvc.txt

EPH_PHASE="$(kubectl get pvc eph-a-scratch -n lab-6a -o jsonpath='{.status.phase}')"
OWNER_KIND="$(kubectl get pvc eph-a-scratch -n lab-6a \
  -o jsonpath='{.metadata.ownerReferences[0].kind}')"
OWNER_NAME="$(kubectl get pvc eph-a-scratch -n lab-6a \
  -o jsonpath='{.metadata.ownerReferences[0].name}')"
EPH_PV="$(kubectl get pvc eph-a-scratch -n lab-6a -o jsonpath='{.spec.volumeName}')"
echo "eph-a-scratch: $EPH_PHASE | owner=$OWNER_KIND/$OWNER_NAME | PV=$EPH_PV"

test "$EPH_PHASE" = 'Bound' && test "$OWNER_KIND" = 'Pod' && test "$OWNER_NAME" = 'eph-a' \
  && echo 'PASS: PVC tên <Pod>-<volume>, do ephemeral volume controller tạo, Pod là chủ sở hữu'
```

**Ý nghĩa:** bạn không viết PVC nào, nhưng có một PVC thật trong namespace. Bài 94: *"ephemeral
volume controller sau đó sẽ tạo một object PersistentVolumeClaim thực sự trong cùng namespace với
Pod"*, tên là **ghép tên Pod và tên volume bằng một dấu gạch nối** — `eph-a` + `scratch` =
`eph-a-scratch`. Owner reference trỏ về Pod chính là thứ sẽ dọn PVC ở B8.3.

Class dùng ở đây là `WaitForFirstConsumer`, đúng chế độ mà bài 94 khuyến nghị cho volume tạm thời
tổng quát, vì khi đó scheduler được tự do chọn node cho Pod trước.

**PASS:** dòng `PASS: PVC tên <Pod>-<volume>…` xuất hiện.

### B8.2. Xóa Pod là PVC và PV đi theo

```bash
kubectl exec -n lab-6a eph-a -- sh -c 'echo lab-6a-ephemeral > /scratch/marker'
kubectl delete pod eph-a -n lab-6a --wait=true

for i in $(seq 1 90); do
  kubectl get pvc eph-a-scratch -n lab-6a >/dev/null 2>&1 || break
  sleep 2
done
for i in $(seq 1 90); do
  kubectl get pv "$EPH_PV" >/dev/null 2>&1 || break
  sleep 2
done

PVC_GONE="$(kubectl get pvc eph-a-scratch -n lab-6a --ignore-not-found -o name | wc -l)"
PV_GONE="$(kubectl get pv "$EPH_PV" --ignore-not-found -o name | wc -l)"
echo "PVC còn lại = $PVC_GONE | PV còn lại = $PV_GONE"

test "$PVC_GONE" -eq 0 && test "$PV_GONE" -eq 0 \
  && echo 'PASS: garbage collector xóa PVC theo Pod, và reclaimPolicy Delete xóa PV theo PVC'
```

**Ý nghĩa:** đây là điểm khác biệt lớn nhất so với B6. StatefulSet **giữ** PVC lại khi Pod biến
mất; volume tạm thời tổng quát thì **không** — vì owner của PVC là chính Pod, và bài 94 nói rõ dây
chuyền: xóa Pod → GC xóa PVC → reclaim policy `Delete` của class xóa volume. Muốn dữ liệu sống lâu
hơn Pod thì bài 94 chỉ cách: dùng một StorageClass có reclaim policy `Retain`, và tự chịu trách
nhiệm dọn về sau — đúng như B5.6 đã làm bằng tay.

**PASS:** dòng `PASS: garbage collector xóa PVC theo Pod…` xuất hiện.

### B8.3. Tên xác định thì cũng xung đột được

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: eph-b-scratch
  namespace: lab-6a
spec:
  storageClassName: local-path
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 128Mi
EOF

sed -e 's/name: eph-a/name: eph-b/' ~/lab-work/6a/ephemeral.yaml > ~/lab-work/6a/ephemeral-b.yaml
kubectl apply -f ~/lab-work/6a/ephemeral-b.yaml
sleep 45

B_PHASE="$(kubectl get pod eph-b -n lab-6a -o jsonpath='{.status.phase}')"
B_OWNER="$(kubectl get pvc eph-b-scratch -n lab-6a -o jsonpath='{.metadata.ownerReferences[0].kind}')"
echo "Pod eph-b = $B_PHASE | ownerReferences của PVC = ${B_OWNER:-<không có>}"
kubectl describe pod eph-b -n lab-6a | tail -n 12 | tee ~/lab-evidence/6a/b8-xung-dot.txt

test "$B_PHASE" != 'Running' && test -z "$B_OWNER" \
  && echo 'PASS: PVC có sẵn không bị chiếm dụng, và Pod không khởi động được'

kubectl delete pod eph-b -n lab-6a --wait=true --force --grace-period=0 2>/dev/null || \
  kubectl delete pod eph-b -n lab-6a --wait=true
kubectl delete pvc eph-b-scratch -n lab-6a --wait=true
```

**Ý nghĩa:** bài 94 cảnh báo đúng tình huống này: quy tắc đặt tên xác định làm hai Pod khác nhau —
hoặc một Pod và một PVC tạo tay — có thể đòi cùng một tên. Kubernetes phát hiện xung đột **dựa trên
quan hệ sở hữu**: PVC đã tồn tại **không bị ghi đè hay sửa đổi**, nên nó vẫn không có
`ownerReferences`; nhưng vì thiếu đúng PVC cần thiết, Pod không khởi động được. Bài kết luận: hãy
cẩn thận khi đặt tên Pod và tên volume trong cùng một namespace.

Bài 94 còn nêu hệ quả bảo mật của cơ chế này: ai tạo được Pod thì **gián tiếp** tạo được PVC, dù
không có quyền tạo PVC trực tiếp. Cách chặn là admission webhook — thuộc giai đoạn 9, không làm ở
đây.

**PASS:** dòng `PASS: PVC có sẵn không bị chiếm dụng…` xuất hiện.

---

## B9. Lưu trữ tạm thời cục bộ — tài nguyên đĩa của node

**Mục đích:** bài [95](../95-ephemeral-storage-vi.md) không nói về volume mà về **đĩa của node**:
`emptyDir`, log container và lớp ghi được của container ăn chung một chỗ, và vượt mức thì Pod bị
trục xuất. Đây là cầu nối trực tiếp sang eviction của giai đoạn 7.

### B9.1. `ephemeral-storage` nằm ngay trong Node object

```bash
kubectl get nodes -o custom-columns=\
NODE:.metadata.name,\
EPH_CAP:.status.capacity.ephemeral-storage,\
EPH_ALLOC:.status.allocatable.ephemeral-storage,\
MEM_ALLOC:.status.allocatable.memory | tee ~/lab-evidence/6a/b9-allocatable.txt

W2_CAP="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.capacity.ephemeral-storage}')"
W2_ALLOC="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.allocatable.ephemeral-storage}')"
echo "worker2: capacity=$W2_CAP | allocatable=$W2_ALLOC"

test -n "$W2_CAP" && test -n "$W2_ALLOC" && test "$W2_ALLOC" != "$W2_CAP" \
  && echo 'PASS: node quảng bá ephemeral-storage, và allocatable nhỏ hơn capacity'
```

**Ý nghĩa:** `ephemeral-storage` là một tài nguyên node đúng nghĩa, đứng cạnh `cpu` và `memory`, và
scheduler dùng `allocatable` của nó y như cách bạn đã thấy ở Lab 3c: tổng `requests` của các Pod
được lập lịch phải nhỏ hơn dung lượng node. Phần chênh giữa `capacity` và `allocatable` là phần
kubelet giữ lại cho hệ thống.

Bài 95 nói kubelet **chỉ đo được** khi node theo một trong các bố cục được hỗ trợ. Cluster lab theo
bố cục *một filesystem duy nhất*: `/var/lib/kubelet` và `/var/log` cùng nằm trên filesystem gốc —
đúng bố cục mà kubelet được thiết kế cho.

**PASS:** dòng `PASS: node quảng bá ephemeral-storage…` xuất hiện.

### B9.2. Khai `requests` và `limits` cho từng container

```bash
cat > ~/lab-work/6a/eph-budget.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: eph-budget
  namespace: lab-6a
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "dd if=/dev/zero of=/scratch/small bs=1M count=16; sleep 3600"]
    resources:
      requests:
        ephemeral-storage: "32Mi"
      limits:
        ephemeral-storage: "64Mi"
    volumeMounts:
    - name: scratch
      mountPath: /scratch
  volumes:
  - name: scratch
    emptyDir: {}
EOF

kubectl apply -f ~/lab-work/6a/eph-budget.yaml
kubectl wait --for=condition=Ready pod/eph-budget -n lab-6a --timeout=180s

REQ="$(kubectl get pod eph-budget -n lab-6a \
  -o jsonpath='{.spec.containers[0].resources.requests.ephemeral-storage}')"
LIM="$(kubectl get pod eph-budget -n lab-6a \
  -o jsonpath='{.spec.containers[0].resources.limits.ephemeral-storage}')"
QOS="$(kubectl get pod eph-budget -n lab-6a -o jsonpath='{.status.qosClass}')"
USED="$(kubectl exec -n lab-6a eph-budget -- sh -c 'du -sm /scratch' | cut -f1)"
echo "requests=$REQ | limits=$LIM | qosClass=$QOS | đã ghi ${USED}Mi"

test "$REQ" = '32Mi' && test "$LIM" = '64Mi' && test "$USED" -lt 64 \
  && echo 'PASS: Pod khai ngân sách lưu trữ tạm thời và đang nằm dưới trần'
```

**Ý nghĩa:** cú pháp giống hệt CPU và memory của Lab 3c, nhưng đơn vị là **byte**. Bài 95 nêu một
cái bẫy đáng nhớ: `400m` là **0.4 byte**, không phải 400 mebibyte — người gõ như vậy gần như chắc
chắn muốn `400Mi` hoặc `400M`. Limit của **Pod** là tổng limit của các container, và dung lượng mà
`emptyDir` tiêu thụ tính vào chính limit đó.

**PASS:** dòng `PASS: Pod khai ngân sách lưu trữ tạm thời và đang nằm dưới trần` xuất hiện.

### B9.3. Vượt trần thì kubelet trục xuất Pod

> **Fault injection — chỉ chạy trên `lab-k8s-worker2`.** Pod dưới đây cố tình ghi vượt limit của
> chính nó. Nó ghi khoảng 200 mebibyte rồi bị trục xuất, không đủ để gây áp lực đĩa lên node.

```bash
sed -e 's/name: eph-budget/name: eph-overflow/' \
    -e 's/count=16/count=200/' ~/lab-work/6a/eph-budget.yaml \
  > ~/lab-work/6a/eph-overflow.yaml
kubectl apply -f ~/lab-work/6a/eph-overflow.yaml

for i in $(seq 1 150); do
  OV_PHASE="$(kubectl get pod eph-overflow -n lab-6a -o jsonpath='{.status.phase}')"
  OV_REASON="$(kubectl get pod eph-overflow -n lab-6a -o jsonpath='{.status.reason}')"
  test "$OV_REASON" = 'Evicted' && break
  sleep 2
done

echo "phase=$OV_PHASE | reason=${OV_REASON:-<chưa có>}"
kubectl get pod eph-overflow -n lab-6a -o jsonpath='{.status.message}'; echo
kubectl describe pod eph-overflow -n lab-6a | tail -n 12 \
  | tee ~/lab-evidence/6a/b9-evicted.txt

test "$OV_PHASE" = 'Failed' && test "$OV_REASON" = 'Evicted' \
  && echo 'PASS: vượt limit ephemeral-storage thì kubelet đặt tín hiệu trục xuất và trục xuất Pod'

kubectl delete pod eph-overflow eph-budget -n lab-6a --wait=true
```

**Ý nghĩa:** bài 95 mô tả đúng dây chuyền: *"Nếu một Pod dùng nhiều lưu trữ tạm thời hơn mức bạn
cho phép, kubelet đặt một tín hiệu trục xuất kích hoạt việc trục xuất Pod."* Đây là **cách ly cấp
pod**: tổng của các container cộng các volume `emptyDir` vượt limit tổng. Thời điểm trục xuất phụ
thuộc chu kỳ đo của kubelet, nên vòng lặp ở trên chờ theo điều kiện chứ không theo một con số cố
định.

Ranh giới quan trọng nhất của bài nằm ở khối *Thận trọng*: nếu kubelet **không** đo được lưu trữ
tạm thời cục bộ trên bố cục node của bạn thì Pod vượt limit **sẽ không** bị trục xuất vì lý do đó —
nhưng khi filesystem xuống thấp, node tự taint chính nó là đang thiếu lưu trữ cục bộ và taint đó
đuổi mọi Pod không tolerate. Cơ chế taint và eviction do áp lực node là nội dung giai đoạn 7.

**PASS:** dòng `PASS: vượt limit ephemeral-storage…` xuất hiện.

### B9.4. `emptyDir` dạng bộ nhớ không tính vào lưu trữ tạm thời

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: eph-mem
  namespace: lab-6a
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 600"]
    volumeMounts:
    - name: ram
      mountPath: /ram
  volumes:
  - name: ram
    emptyDir:
      medium: Memory
      sizeLimit: 32Mi
EOF

kubectl wait --for=condition=Ready pod/eph-mem -n lab-6a --timeout=180s
kubectl exec -n lab-6a eph-mem -- sh -c 'mount | grep " /ram "' \
  | tee ~/lab-evidence/6a/b9-tmpfs.txt

grep -q 'tmpfs' ~/lab-evidence/6a/b9-tmpfs.txt \
  && echo 'PASS: emptyDir medium Memory được mount bằng tmpfs'

kubectl delete pod eph-mem -n lab-6a --wait=true
```

**Ý nghĩa:** bài 91 nói `medium: "Memory"` đổi backing sang tmpfs, và bài 95 bổ sung hệ quả kế
toán: kubelet theo dõi `emptyDir` dạng tmpfs như **mức sử dụng bộ nhớ của container**, không phải
lưu trữ tạm thời. Nghĩa là file bạn ghi vào `/ram` tiêu vào memory limit — cùng loại giới hạn đã
làm Pod bị `OOMKilled` ở Lab 3c, chứ không phải loại vừa gây eviction ở B9.3.

**PASS:** dòng `PASS: emptyDir medium Memory được mount bằng tmpfs` xuất hiện.

---

## B10. Cleanup và gate tạo snapshot `03-storage-ready`

**Mục đích:** xóa mọi object của bài học, **giữ lại đúng phần hạ tầng** mà mốc mới được định nghĩa
là có — provisioner và StorageClass mặc định — rồi chứng minh cluster khỏe trước khi chụp.

Định nghĩa của mốc `03-storage-ready` theo [chuỗi snapshot](README.md#3-chuỗi-snapshot): mọi thứ
của `02-net-ready`, **cộng** namespace `local-path-storage` đang chạy provisioner, **cộng**
StorageClass `local-path` được đánh dấu mặc định. Không workload, không PV, không PVC.

### B10.1. Xóa object của bài học

```bash
kubectl delete namespace lab-6a --wait=true --timeout=600s

for i in $(seq 1 60); do
  ST="$(kubectl get pv lab-6a-static -o jsonpath='{.status.phase}' 2>/dev/null)"
  test "$ST" = 'Released' && break
  sleep 2
done
echo "PV lab-6a-static sau khi namespace biến mất: ${ST:-<không còn>}"

kubectl delete pv lab-6a-static --ignore-not-found --wait=true
kubectl delete storageclass lab-6a-retain lab-6a-immediate --ignore-not-found

rm -f ~/lab-work/6a/emptydir.yaml ~/lab-work/6a/pv-static.yaml \
      ~/lab-work/6a/pvc-static.yaml ~/lab-work/6a/pvc-static-2.yaml \
      ~/lab-work/6a/pvc-no-class.yaml ~/lab-work/6a/pod-static-a.yaml \
      ~/lab-work/6a/pod-static-b.yaml ~/lab-work/6a/pod-static-c.yaml \
      ~/lab-work/6a/local-path-storage.yaml ~/lab-work/6a/pvc-dyn.yaml \
      ~/lab-work/6a/pod-dyn.yaml ~/lab-work/6a/sc-immediate.yaml \
      ~/lab-work/6a/sc-retain.yaml ~/lab-work/6a/keep.yaml \
      ~/lab-work/6a/statefulset.yaml ~/lab-work/6a/projected.yaml \
      ~/lab-work/6a/projected-sai.yaml ~/lab-work/6a/ephemeral.yaml \
      ~/lab-work/6a/ephemeral-b.yaml ~/lab-work/6a/eph-budget.yaml \
      ~/lab-work/6a/eph-overflow.yaml
rmdir ~/lab-work/6a
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và gate dưới đây biến điều đó thành
lỗi thấy được thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/6a/` **giữ lại** — đó là bằng chứng.

**PASS:** namespace biến mất; PV `lab-6a-static` chuyển `Released` rồi bị xóa; `rmdir` không báo
lỗi.

### B10.2. Dọn thư mục còn sót trên node

`hostPath` không dọn theo object. B2 tạo `/srv/lab-6a/static` trên **hai** node: `lab-k8s-worker1`
(B2.4) và `lab-k8s-worker2` (B2.6).

Chạy trên **`lab-k8s-worker1`**, rồi lặp lại y hệt trên **`lab-k8s-worker2`**:

```bash
sudo rm -rf /srv/lab-6a
sudo test -d /srv/lab-6a && echo 'FAIL: /srv/lab-6a vẫn còn' || echo 'PASS: /srv/lab-6a đã dọn'
sudo ls -A /opt/local-path-provisioner
```

**PASS:** cả hai node in `PASS: /srv/lab-6a đã dọn`; `/opt/local-path-provisioner` **rỗng** trên cả
hai — thư mục gốc ở lại vì nó thuộc provisioner, nhưng không được còn volume nào bên trong.

> Nếu `/opt/local-path-provisioner` còn thư mục con, có một PV chưa được thu hồi. Quay lại
> `kubectl get pv` trên master và xử lý theo mục 4 trước khi chụp snapshot.

### B10.3. Gate trạng thái tầng lưu trữ

```bash
PVC_ALL="$(kubectl get pvc -A --no-headers 2>/dev/null | wc -l)"
PV_ALL="$(kubectl get pv --no-headers 2>/dev/null | wc -l)"
SC_ALL="$(kubectl get sc --no-headers | wc -l)"
SC_NAME="$(kubectl get sc -o jsonpath='{.items[0].metadata.name}')"
SC_DEF="$(kubectl get sc local-path \
  -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}')"
LPP_READY="$(kubectl -n local-path-storage get deploy local-path-provisioner \
  -o jsonpath='{.status.readyReplicas}')"
NS_LAB="$(kubectl get namespace lab-6a --ignore-not-found -o name | wc -l)"

echo "PVC toàn cluster=$PVC_ALL | PV=$PV_ALL | StorageClass=$SC_ALL ($SC_NAME, default=$SC_DEF)"
echo "provisioner readyReplicas=${LPP_READY:-0} | namespace lab-6a còn=$NS_LAB"
test ! -e ~/lab-work/6a && echo 'PASS: manifest tạm đã xóa'

test "$PVC_ALL" -eq 0 && test "$PV_ALL" -eq 0 && test "$NS_LAB" -eq 0 \
  && echo 'PASS: không còn PVC, PV hay namespace của lab'
test "$SC_ALL" -eq 1 && test "$SC_NAME" = 'local-path' && test "$SC_DEF" = 'true' \
  && test "${LPP_READY:-0}" -ge 1 \
  && echo 'PASS: đúng một StorageClass mặc định và provisioner đang chạy'

{
  echo "=== $(date -Is) — tầng lưu trữ sau Lab 6a ==="
  kubectl get sc -o wide
  kubectl get pv
  kubectl get pvc -A
  kubectl -n local-path-storage get all
} | tee ~/lab-evidence/6a/b10-sau.txt
```

**Ý nghĩa:** hai gate này là định nghĩa của mốc mới ở dạng lệnh. Gate thứ nhất bảo đảm bạn không
chụp kèm rác của bài học; gate thứ hai bảo đảm bạn **có** chụp kèm phần hạ tầng mà các lab sau
(6b, 7a, 7b, 8a, 9a, 9b, 11a) sẽ dựa vào.

**PASS:** đủ ba dòng `PASS:`.

### B10.4. Chạy trọn bảy tầng gate của A5.4

Lab này đổi hạ tầng, nên **không** được dừng ở gate ngắn A5.5. Chạy **toàn bộ bảy tầng** của
[A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready) theo đúng thứ tự từ dưới lên:

| Tầng | Nội dung | Điều chỉnh cho mốc `03-storage-ready` |
| --- | --- | --- |
| 0 | prereq OS trên cả ba node | không đổi |
| 1 | control plane khỏe thật | không đổi |
| 2 | node, condition, taint và PodCIDR | không đổi |
| 3 | Pod networking xuyên node | CNI là loại do Lab 5b cài, không phải Flannel |
| 4 | DNS trong cluster và ra Internet | không đổi |
| 5 | Service, EndpointSlice và kube-proxy | không đổi |
| 6 | đường control plane → kubelet | không đổi |

Khi đối chiếu danh sách Pod, tập namespace hệ thống hợp lệ ở mốc này là: `kube-system`, namespace
của CNI và của ingress controller do Lab 5b cài, **cộng** `local-path-storage`.

```bash
kubectl get pods -A -o wide | tee ~/lab-evidence/6a/b10-pods.txt
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl get pods -n default
kubectl get svc -n default
```

**PASS:** bảy tầng của A5.4 đều PASS; lệnh field selector trả `No resources found`; `default` không
có Pod và chỉ còn Service `kubernetes`; Pod của `local-path-storage` ở trạng thái `Running`.

### B10.5. Tắt máy và chụp `03-storage-ready`

Chụp khi VM đã tắt để snapshot không kèm trạng thái RAM — cùng lý do như A5.4.8 của Lab 00. Chạy
trên **từng node** theo thứ tự worker 2 → worker 1 → master:

```bash
sudo shutdown -h now
```

Chờ VMware Workstation hiển thị cả ba VM ở trạng thái *Powered off*. Chụp trên **cả ba VM**:
chuột phải VM → **Snapshot → Take Snapshot** → ô *Name* điền đúng nguyên văn:

```text
03-storage-ready
```

Ô *Description* ghi lab đã dựng mốc này và ngày chụp, ví dụ
`dựng bằng LAB-6A-PV-PVC-VA-STORAGECLASS.md, chụp <ngày>`.

Quy tắc tên giống hệt Lab 00: đúng nguyên văn `03-storage-ready` trên cả ba VM, không hậu tố theo
VM, không thêm ngày, không đổi hoa thường. **Giữ nguyên snapshot `02-net-ready`** — đừng xóa nó,
đó vẫn là điểm quay lui của cả hai mốc.

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

**PASS:** đúng ba dòng `PASS:`, không dòng `FAIL:` nào — cả ba VM có **cả hai** mốc, tên phân biệt
hoa thường chính xác. Lab 6a kết thúc ở đây; để ba VM ở trạng thái tắt, lab sau tự bật theo A5.5.

---

## 3. Checkpoint 6a

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Một Pod ghi file vào `emptyDir`. Container crash rồi chạy lại: file còn không? Bạn xóa Pod
      rồi apply lại đúng manifest đó: file còn không? Vì sao hai câu trả lời khác nhau?
- [ ] PV ở pha `Available` và một PVC vừa được tạo. Ai ghép hai cái đó lại, ghi vào những trường
      nào, và vì sao PVC thứ hai giống hệt PVC thứ nhất lại không bao giờ được phục vụ?
- [ ] Ba PVC: một khai `storageClassName: local-path`, một khai `""`, một bỏ trống hẳn. Bạn tạo
      một StorageClass mặc định **sau đó**. Số phận từng PVC ra sao, và hai cơ chế nào của
      Kubernetes tạo ra khác biệt đó?
- [ ] `ReadWriteOnce` có nghĩa là "chỉ một Pod dùng được volume" không? Trên cluster ba node của
      bạn, hai Pod thế nào thì dùng chung được một PVC `ReadWriteOnce`, và access mode nào mới
      thật sự giới hạn còn đúng một Pod?
- [ ] Bạn tạo một PVC dùng class mặc định của lab và không tạo Pod nào. Sau 10 phút PVC vẫn
      `Pending`. Đây là lỗi hay là hành vi đúng? Trường nào của StorageClass quyết định điều đó,
      và đổi nó thành `Immediate` thì chuyện gì xảy ra trên chính cluster này?
- [ ] Kể lại chuỗi sự kiện từ lúc bạn gõ `kubectl delete pvc` cho tới lúc thư mục trên node biến
      mất, khi class dùng `reclaimPolicy: Delete`. Ở bước nào finalizer chặn lại, và vì sao?
- [ ] Với `reclaimPolicy: Retain`, sau khi xóa PVC thì PV ở pha nào, còn giữ trường gì, và vì sao
      một PVC mới chỉ đích danh PV đó bằng `volumeName` vẫn không bind được? Ba bước thu hồi thủ
      công là gì?
- [ ] **Nợ #2 — đã trả ở lab này.** Một StatefulSet tên `web`, 3 replica, `volumeClaimTemplates`
      tên `www`. Kể đủ tên ba PVC sinh ra. Bạn scale xuống 1 rồi xóa luôn StatefulSet: còn lại bao
      nhiêu PVC, vì sao, và trường nào đổi được hành vi đó?
- [ ] Bạn scale StatefulSet đó từ 1 lên lại 3. `web-1` là một Pod mới hoàn toàn — vì sao nó đọc
      được đúng dữ liệu mà `web-1` cũ đã ghi?
- [ ] Volume tạm thời tổng quát và `volumeClaimTemplates` cùng sinh ra PVC. Kể hai khác biệt: quy
      tắc đặt tên, và điều xảy ra với PVC khi Pod bị xóa. Cái gì làm nên khác biệt thứ hai?
- [ ] Trong một volume `projected` gộp Secret, ConfigMap và downwardAPI: trường nào của secret đổi
      tên so với volume `secret` rời, `defaultMode` đặt được ở cấp nào, và ba nguồn đó có được nằm
      ở namespace khác không?
- [ ] Container của bạn khai `limits.ephemeral-storage: 64Mi` và ghi 200Mi vào một `emptyDir`. Ai
      phát hiện, làm gì với Pod, và kết quả đó khác gì với việc ghi 200Mi vào một `emptyDir` có
      `medium: Memory`?

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng sau, không nhìn tài liệu:

1. **Luồng cấp phát.** Bạn tạo một PVC không khai `storageClassName` trên cluster này. Kể đủ:
   thành phần nào điền class vào spec; vì sao chưa có gì xảy ra tiếp; điều gì kích hoạt phần còn
   lại; ai tạo PV; PV mang những trường nào mà bạn chưa từng viết ra và mỗi trường đến từ đâu; và
   cuối cùng thứ gì xuất hiện trên đĩa của node nào. Sau đó kể ngược lại luồng xóa, tới tận lúc
   thư mục biến mất — và chỉ ra đúng chỗ mà `Retain` làm câu chuyện rẽ hướng.
2. **Luồng danh tính có trạng thái.** Bạn apply một StatefulSet 3 replica có `volumeClaimTemplates`
   rồi xóa nó đi. Kể: mỗi Pod nhận những gì trong định danh của nó; PVC nào được tạo, tên theo quy
   tắc nào, ai giữ chúng lại khi Pod biến mất; điều gì còn sót lại trong cluster sau khi
   StatefulSet đã bị xóa, và bạn phải làm gì để dọn cho sạch. So sánh với một Pod dùng volume tạm
   thời tổng quát: cùng sinh PVC, nhưng vì sao cái này tự biến mất còn cái kia thì không.

Khi toàn bộ checkbox được đánh dấu và bạn không còn nhầm **volume** với **PersistentVolume**,
`""` với **bỏ trống**, `Delete` với `Retain`, `Immediate` với `WaitForFirstConsumer`, hay
**lưu trữ tạm thời cục bộ** với **volume tạm thời** — Lab 6a hoàn tất.

**Nợ #2 đã được trả ở lab này** (`volumeClaimTemplates` của StatefulSet, B6). Nợ #5 — ảnh chụp
nhanh và nhân bản volume — **chưa trả**, và nó cần một CSI driver có hỗ trợ snapshot; xem
[sổ nợ lab](README.md#5-sổ-nợ-lab) và Lab 6b.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Bảng dưới chỉ liệt kê sự cố phát sinh từ nội dung bài học 6a.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| **PVC kẹt `Pending`** sau khi đã có provisioner | `kubectl describe pvc <tên> -n lab-6a` rồi đọc phần Events | Ba nguyên nhân khác nhau: event `WaitForFirstConsumer` là **đúng** — tạo Pod tiêu thụ; event `ProvisioningFailed` là provisioner thất bại — xem log ở dòng dưới; không event nào và `storageClassName` là `""` thì claim tự vô hiệu hóa cấp phát động, không có cách nào cứu ngoài sửa PVC |
| PVC `Pending`, event `ProvisioningFailed` | `kubectl -n local-path-storage logs deploy/local-path-provisioner --tail=50` | Thường là helper Pod không kéo được image `docker.io/library/busybox`, hoặc class dùng `Immediate` như B4.4. Sửa nguyên nhân rồi xóa và tạo lại PVC |
| PVC `Pending` mà PV đích ở pha `Released` | `kubectl get pv <tên> -o jsonpath='{.spec.claimRef}'` | `claimRef` cũ vẫn giữ chỗ. Đây là hành vi đúng của `Retain`; muốn dùng lại thì thu hồi thủ công ba bước như B5.6, không xóa `claimRef` bằng tay giữa chừng lab |
| **PV kẹt `Terminating`** | `kubectl get pv <tên> -o jsonpath='{.metadata.finalizers}'` và `kubectl get pvc -A` | Còn PVC bind vào nó thì finalizer `kubernetes.io/pv-protection` giữ lại — xóa PVC trước. Nếu là finalizer của provisioner, kiểm tra provisioner còn chạy không: nó phải sống để dọn thư mục thì PV mới được gỡ |
| PVC kẹt `Terminating` | `kubectl get pvc <tên> -n lab-6a -o jsonpath='{.metadata.finalizers}'`; `kubectl get pods -n lab-6a` | Finalizer `kubernetes.io/pvc-protection` chờ Pod cuối cùng dùng nó biến mất. Xóa Pod, đừng gỡ finalizer bằng tay |
| Deployment `local-path-provisioner` không lên `Ready` | `kubectl -n local-path-storage describe pod` và `logs` | Kiểm tra image kéo được không, và ServiceAccount/ClusterRoleBinding trong manifest đã apply đủ chưa. Apply lại đúng file đã tải ở B3.1, không sửa manifest |
| B3.1 gate image fail | `grep image ~/lab-work/6a/local-path-storage.yaml` | Bạn tải nhầm nhánh hoặc nhầm tag. `LPP_VERSION` phải khớp bảng A1.4 của Lab 00; sửa ở bảng đó trước, rồi tải lại |
| Pod `Pending` với `node(s) had volume node affinity conflict` | `kubectl get pv <tên> -o jsonpath='{.spec.nodeAffinity}'` | Volume đã bị đóng đinh vào một node còn Pod bị ghim sang node khác. Bỏ `nodeSelector` hoặc dùng PVC khác — đừng sửa `nodeAffinity` của PV |
| B2.6 in `FAIL: static-c thấy marker` | `kubectl get pod static-c -n lab-6a -o wide` | Pod không lên `lab-k8s-worker2`. Kiểm tra label `kubernetes.io/hostname` của node và `nodeSelector` trong manifest sinh bởi `sed` |
| B4.3 không tìm thấy thư mục trên worker1 | `kubectl get pv "$PV_DYN" -o yaml` | Đọc lại `nodeAffinity` xem PV nằm ở node nào; nếu là worker2 thì `nodeSelector` của `dyn-a` chưa ăn. Xóa Pod và PVC rồi làm lại B4.1 |
| B6 rollout treo, Pod StatefulSet `Pending` | `kubectl describe pod web-0 -n lab-6a` | Nếu do PVC chưa bind, xem hai dòng đầu bảng này. Nếu do thiếu tài nguyên node, xem [tầng 2 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a543-tầng-2--node-condition-taint-và-podcidr) — không thêm node, không hạ replica |
| B6.5 đọc ra dữ liệu sai ordinal | `kubectl get pvc -n lab-6a -o wide` | Kiểm tra từng PVC còn bind đúng PV cũ không. Nếu bạn lỡ xóa một PVC giữa chừng, StatefulSet sẽ tạo PVC mới rỗng — làm lại B6 từ đầu sau khi xóa cả ba PVC |
| B8.3 Pod `eph-b` lại chạy được | `kubectl get pvc eph-b-scratch -n lab-6a -o jsonpath='{.metadata.ownerReferences}'` | Bạn tạo Pod trước PVC nên controller đã kịp làm chủ PVC. Xóa cả hai rồi làm lại đúng thứ tự: PVC trước, Pod sau |
| B9.3 chờ hết vòng lặp mà Pod không bị trục xuất | `kubectl describe node lab-k8s-worker2 \| grep -i ephemeral`; `du -sh /var/lib/kubelet` trên worker2 | Kubelet chỉ áp giới hạn khi node theo bố cục được hỗ trợ (bài 95). Kiểm tra `/var/lib/kubelet` và `/var/log` có nằm trên filesystem gốc không. Không chỉnh cấu hình kubelet để "ép" bước này |
| Xóa namespace `lab-6a` treo ở `Terminating` | `kubectl get pvc,pod -n lab-6a`; `kubectl get pv` | Gần như luôn là PVC đang chờ Pod, hoặc PV chờ provisioner dọn. Chờ provisioner làm việc; không gỡ finalizer của Namespace |
| `/opt/local-path-provisioner` còn thư mục con sau cleanup | `kubectl get pv`; `kubectl -n local-path-storage logs deploy/local-path-provisioner` | Có PV chưa thu hồi xong, hoặc helper Pod xóa thất bại. Xử lý PV trước; chỉ khi `kubectl get pv` đã rỗng mới xóa tay thư mục thừa trên node |
| Sau cleanup còn StorageClass lạ | `kubectl get sc` | `lab-6a-retain` hoặc `lab-6a-immediate` chưa xóa. Xóa chúng; **giữ lại** `local-path` — mốc `03-storage-ready` được định nghĩa là có class đó và nó là mặc định |

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
- [Kubernetes v1.35 — Delete a StatefulSet](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/delete-stateful-set/)
- [Kubernetes v1.35 — Change the default StorageClass](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/)
- [Rancher — local-path-provisioner](https://github.com/rancher/local-path-provisioner)
