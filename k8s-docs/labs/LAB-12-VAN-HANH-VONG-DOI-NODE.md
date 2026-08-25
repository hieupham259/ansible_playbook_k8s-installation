# Lab 12 — Vận hành vòng đời node

> **Điểm bắt đầu:** snapshot `04-metrics-ready` — mốc do Lab 11a tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot).
> **Điểm kết thúc:** cleanup trả cluster về đúng `04-metrics-ready`, **không tạo snapshot mới**.
> Lab này không cài thêm bất kỳ thành phần hạ tầng nào và không sửa cấu hình node nào.
> **Lab trước:** Lab 11b — HPA và VPA (chưa viết, xem [bản đồ lab](README.md#4-bản-đồ-lab)) cũng
> trả cluster về `04-metrics-ready`. Cluster vào lab này phải ở đúng mốc đó: có CNI thực thi
> NetworkPolicy, ingress controller, StorageClass mặc định và metrics-server; không workload.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục
[Giai đoạn 12 — Quản trị cluster nâng cao](../00-ALO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao)
— tám bài [155](../155-cluster-administration-vi.md), [169](../169-node-shutdown-vi.md),
[170](../170-swap-memory-management-vi.md), [171](../171-node-autoscaling-vi.md),
[165](../165-addons-vi.md), [168](../168-compatibility-version-vi.md),
[167](../167-coordinated-leader-election-vi.md), [156](../156-certificates-vi.md), cộng ba bài
thực hành [189](../189-administer-cluster-vi.md), [190](../190-access-cluster-api-vi.md),
[198](../198-controller-manager-leader-migration-vi.md).

**Xương sống của lab là một vòng bảo trì node đầy đủ** trên `lab-k8s-worker2`: `cordon` →
`drain` → tắt máy → bật lại → `uncordon`. Bảy mục còn lại là phần **đọc cấu hình thật** của
cluster: swap, graceful node shutdown, add-on, phiên bản tương thích, Lease của control plane.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không
chép lại con số phiên bản nào**. Thành phần ngoài baseline mà lab thấy đang chạy — CNI thay
Flannel, ingress controller, dynamic provisioner, metrics-server — đã được khóa ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) và do Lab 5b, Lab 6a,
Lab 11a cài; lab 12 chỉ **đọc** và **liệt kê** chúng, không đụng vào.

Quan hệ với hai lab đã học: [Lab 7a](LAB-7A-LAP-LICH-VA-EVICTION.md) đã gọi thẳng subresource
`pods/eviction` và đã làm taint/toleration cùng PodDisruptionBudget; lab này dùng `kubectl drain`
ở **góc vận hành** và chỉ ra quan hệ giữa hai thứ.
[Lab 1c](LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md#b3-lease-heartbeat-leader-election-và-api-server-identity)
đã đọc Lease ở mức khái niệm; B4 ở đây đọc lại chúng như một tín hiệu vận hành.

Lab dùng Deployment, `topologySpreadConstraints`, `nodeSelector`, ServiceAccount token của các
giai đoạn đã học làm công cụ. **Không** dùng CRD tự tạo hay DRA — hai thứ đó thuộc
[giai đoạn 13](../00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao) và
[giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), đứng **sau** lab này.
Cũng **không** dùng HPA: nó là nội dung của Lab 11b.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`, rồi thêm ba lệnh riêng của lab này:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Ba lệnh riêng của lab 12: đúng mốc 04-metrics-ready, chưa node nào bị cordon, chưa có PDB.
kubectl top node
kubectl get nodes -o custom-columns='NODE:.metadata.name,UNSCHEDULABLE:.spec.unschedulable'
kubectl get pdb -A
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod; **`kubectl top node` in đủ ba dòng
số liệu** (đó là định nghĩa của mốc `04-metrics-ready`); cột `UNSCHEDULABLE` của cả ba node là
`<none>`; `kubectl get pdb -A` trả `No resources found`.

Nếu `kubectl top node` báo lỗi thì cluster của bạn chưa ở mốc đầu vào — restore cả ba VM về
`04-metrics-ready` trước khi tiếp tục. Nếu một node đã `SchedulingDisabled` hoặc còn PDB sót lại,
cũng dừng: B7 dựa hẳn vào việc cả hai worker sạch và không có PDB nào chặn `drain`.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Bốn nhóm việc quản trị mà bài [155](../155-cluster-administration-vi.md) chia ra, và với mỗi
  nhóm chỉ ra được **một hiện vật có thật** trong cluster của mình.
- Add-on nào đang chạy trên cluster, xếp vào **nhóm chức năng nào** của bài
  [165](../165-addons-vi.md) — và vì sao có những thứ đang chạy mà danh mục đó **không liệt kê**.
- `--emulated-version` làm gì, ràng buộc `≤ binaryVersion` nghĩa là gì, và chứng minh được cluster
  của mình **không** đặt flag đó nên emulation version bằng đúng binary version.
- Lease của control plane ở góc vận hành: đọc `holderIdentity`, thấy `renewTime` tiến lên, và
  chứng minh **bầu chọn leader có phối hợp đang tắt** trên cluster này.
- Trạng thái swap thật của ba node, `failSwapOn`/`memorySwap` hiệu lực của kubelet, và giải thích
  được vì sao baseline tắt swap thay vì bật nó.
- Graceful node shutdown **có được kích hoạt hay không** trên node lab — đọc từ cấu hình hiệu lực
  của kubelet, không đoán từ feature gate — và hệ quả của câu trả lời đó với thao tác tắt máy.
- Phân biệt dứt khoát `cordon` với `drain`: một cái chỉ **đánh dấu**, một cái đánh dấu **và**
  trục xuất; chứng minh bằng số Pod trên node trước và sau mỗi lệnh.
- Vì sao `--ignore-daemonsets` gần như luôn cần, đọc đúng câu từ chối của `kubectl drain` khi
  thiếu nó, và nói được `drain` gọi cơ chế nào ở bên dưới.
- Chạy trọn vòng bảo trì `cordon` → `drain` → tắt máy → bật lại → `uncordon` trên
  `lab-k8s-worker2`, chứng minh workload **dịch chuyển** sang worker còn lại rồi **quay về** sau
  khi uncordon, và biết reboot **không** tự gỡ cordon.
- Điều gì xảy ra ở phía control plane khi node biến mất: condition `Ready` thành `Unknown`, taint
  `node.kubernetes.io/unreachable`, Node Lease ngừng được gia hạn — và vì sao không Pod nào kẹt
  `Terminating` trong kịch bản này.
- Vì sao cluster ba VM cố định **không thể** có autoscaling Node, và định nghĩa "Node rỗng" của
  bài [171](../171-node-autoscaling-vi.md) khớp với trạng thái node vừa drain xong.
- Truy cập cluster qua Kubernetes API bằng ba đường: `kubectl`, `kubectl proxy`, và HTTP client
  trực tiếp có xác thực — kèm lý do không dùng `--insecure`.
- Vì sao quy trình Leader Migration của bài [198](../198-controller-manager-leader-migration-vi.md)
  **không bao giờ chạy** trên cluster này, chứng minh bằng bốn dữ kiện đọc được.
- Cleanup đúng phạm vi: không node nào còn cordon, cấu hình node không đổi, cluster về đúng
  `04-metrics-ready` và **cả ba node `Ready`**.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 12 | Phần lab kiểm chứng |
| --- | --- |
| [155 — Quản trị cluster](../155-cluster-administration-vi.md) | B1 — trang mục lục; lab tìm trong cluster một hiện vật cho từng nhóm: lập kế hoạch (`providerID` rỗng, `nodeInfo`), quản lý cluster (Node và ResourceQuota), bảo mật cluster và bảo mật kubelet (`--authorization-mode`, kubelet authn/authz, chứng chỉ do TLS bootstrapping cấp), dịch vụ tùy chọn (CoreDNS và nguồn metric) |
| [165 — Cài đặt các Add-on](../165-addons-vi.md) | B2 — kiểm kê add-on **đang chạy thật**, xếp vào năm nhóm chức năng của bài, và chỉ ra ba thứ đang chạy mà danh mục không liệt kê |
| [168 — Phiên bản tương thích cho Control Plane](../168-compatibility-version-vi.md) | B3 — đọc ba manifest control plane để chứng minh không thành phần nào đặt `--emulated-version`; so `gitVersion` của API server với tag image và với baseline A1.3 |
| [167 — Bầu chọn leader có phối hợp](../167-coordinated-leader-election-vi.md) | B4 — `holderIdentity`, `acquireTime`, `renewTime`, `leaseDurationSeconds`, `leaseTransitions` của Lease control plane; `renewTime` tiến lên theo thời gian; API group `coordination.k8s.io/v1beta1` không được phục vụ nên `LeaseCandidate` không tồn tại |
| [170 — Quản lý bộ nhớ swap](../170-swap-memory-management-vi.md) | B5 — `swapon --show` và `free` trên cả ba node; trường `swap` vắng mặt trong Node status; `failSwapOn` và `memorySwap` hiệu lực của ba kubelet |
| [169 — Tắt node](../169-node-shutdown-vi.md) | B6 — `shutdownGracePeriod`, `shutdownGracePeriodCriticalPods`, feature gate, và ba điều kiện nền mà bài cảnh báo (`unattended-upgrades`, file `logind.conf.d`, `systemd-logind`); B7 — vòng bảo trì đầy đủ, gồm cả phần *tắt node không nhẹ nhàng* quan sát từ phía control plane |
| [171 — Tự động mở rộng Node](../171-node-autoscaling-vi.md) | B7.4 — trạng thái "Node rỗng" (chỉ còn Pod DaemonSet) sau khi drain, đúng định nghĩa của bài; B8 — không autoscaler nào chạy, không node nào có `providerID`, số node không đổi qua cả vòng bảo trì |
| [189 — Quản trị một Cluster](../189-administer-cluster-vi.md) | B9.1 — trang mục lục; lab tìm trong cluster ba hiện vật ứng với ba mục của trang: `/api` (bài 190), subresource `pods/eviction` (bài 255, cơ chế nằm dưới `drain` của B7), annotation `storageclass.kubernetes.io/is-default-class` (bài 192) |
| [190 — Truy cập cluster bằng Kubernetes API](../190-access-cluster-api-vi.md) | B9.2–B9.4 — `kubectl config view`, `kubectl proxy` rồi `curl` qua proxy, và HTTP client trực tiếp dùng token của ServiceAccount cộng CA lấy từ kubeconfig thay cho `--insecure` |
| [198 — Di trú control plane sang Cloud Controller Manager](../198-controller-manager-leader-migration-vi.md) | B10 — bốn dữ kiện đọc được chứng minh quy trình này không áp dụng: không `--cloud-provider`, không `cloud-controller-manager`, `providerID` rỗng, control plane một node; cộng nội dung role `system::leader-locking-kube-controller-manager` giải thích vì sao bước cấp quyền RBAC là bắt buộc |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài [156](../156-certificates-vi.md) — toàn bộ | Đây là **trang trỏ hướng sáu dòng**, không chứa một thao tác nào; nội dung thật là quản lý vòng đời certificate. Đó là **[nợ #7](README.md#5-sổ-nợ-lab)**, phát sinh ở chính giai đoạn 12 và **chưa được trả ở lab này**. Trả ở [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ) với bài [219](../219-kubeadm-certs-vi.md) và [191](../191-certificates-manual-vi.md). Lab 12 **không** chạy `kubeadm certs`, không đọc hạn certificate, không xoay CA |
| Bài [169](../169-node-shutdown-vi.md) — **bật** graceful node shutdown và đo hai giai đoạn chấm dứt Pod | Phải đặt `shutdownGracePeriod` và `shutdownGracePeriodCriticalPods` khác 0 trong `KubeletConfiguration`, tức sửa cấu hình kubelet trên node đang chạy — thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), bài [224](../224-kubelet-config-file-vi.md). B6 kiểm chứng **phần đọc được**: giá trị hiệu lực, feature gate, và ba điều kiện nền |
| Bài [169](../169-node-shutdown-vi.md) — taint `node.kubernetes.io/out-of-service` và Pod StatefulSet kẹt `Terminating` | Kịch bản đó chỉ xuất hiện khi node chết **mà chưa drain** và trên node có Pod StatefulSet gắn volume. Cố ý dựng nó là cố ý làm hỏng dữ liệu trên mốc snapshot của lộ trình; B7 làm **đúng quy trình** (drain trước) và chứng minh hệ quả ngược lại: không Pod nào kẹt |
| Bài [169](../169-node-shutdown-vi.md) — `shutdownGracePeriodByPodPriority` và `disable-force-detach-on-timeout` | Cái đầu cần bật thêm feature gate và nhiều lớp PriorityClass; cái sau phải sửa cấu hình `kube-controller-manager`. Cả hai được chính bài xếp vào phần đọc lướt |
| Bài [170](../170-swap-memory-management-vi.md) — bật swap, `LimitedSwap`, `kubectl top --show-swap` có số liệu thật | Bật swap là đổi prereq OS mà [A4.1 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites) đã khóa, cộng sửa `failSwapOn` của kubelet. Baseline tắt swap có chủ đích; B5 kiểm chứng đúng trạng thái đó và đọc ranh giới |
| Bài [171](../171-node-autoscaling-vi.md) — chạy Cluster Autoscaler hoặc Karpenter thật | Autoscaler cấp phát Node bằng cách **tạo và xóa tài nguyên của cloud provider**; ba VM tự dựng không có API nào để nó gọi. Đây là giới hạn về bản chất, không phải thiếu cấu hình. B8 chứng minh bằng bốn dữ kiện đọc được |
| Bài [167](../167-coordinated-leader-election-vi.md) — bật `CoordinatedLeaderElection` và quan sát chiến lược `OldestEmulationVersion` | Phải thêm `--feature-gates` và `--runtime-config` vào manifest static Pod của kube-apiserver, và phải có **nhiều instance** mỗi thành phần mới có gì để tranh. Cả hai điều kiện đều nằm ngoài topology một control plane; sửa manifest thuộc giai đoạn 20, nhiều control plane thuộc lab 8b/8c |
| Bài [168](../168-compatibility-version-vi.md) — đặt `--emulated-version` và quan sát API bị cắt | Đổi flag control plane trên cluster đang chạy; giá trị thật của tính năng chỉ hiện ra trong một lần **nâng cấp**, tức [giai đoạn 17](../00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster) |
| Bài [198](../198-controller-manager-leader-migration-vi.md) — chạy Leader Migration thật | Cần cloud provider in-tree, một `cloud-controller-manager` do nhà cung cấp build, control plane **nhân bản**, và sửa manifest của cả hai controller manager. Không dữ kiện nào trong bốn cái đó tồn tại ở đây; B10 kiểm chứng đúng điều đó |
| Bài [190](../190-access-cluster-api-vi.md) — thư viện client Go/Python/Java/dotnet/JavaScript/Haskell | Phải cài toolchain ngôn ngữ lên node lab, tức thêm phần mềm ngoài baseline. B9 làm phần tương đương bằng HTTP client có sẵn |
| Bài [189](../189-administer-cluster-vi.md) — các trang con còn lại của mục | Đó là mục lục của gần 50 trang task trải khắp [Phần II](../00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster); mỗi trang có chỗ đứng riêng ở giai đoạn 16–27. B9.1 chỉ dùng nó đúng công dụng của một trang mục lục |

Đây **không phải nợ lộ trình mới**, trừ nợ #7 vốn đã có sẵn trong
[sổ nợ lab](README.md#5-sổ-nợ-lab) và **không được trả ở lab này**. Những phần còn lại là giới
hạn cố định của môi trường lab hoặc là nội dung đã có chỗ đứng ở giai đoạn sau.

Lab này **không** dạy quy trình rút node khỏi cluster rồi join lại, cũng không dạy nâng cấp node.
Cả hai thuộc [giai đoạn 16](../00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node) và
[giai đoạn 17](../00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster), với bài
[255](../255-safely-drain-node-vi.md) đi sâu hơn vào chính lệnh `drain` mà B7 dùng ở đây.

### 1.2. Thời lượng

2–3 giờ, tính từ lúc gate mở đầu đã PASS. B7 có ba bước phải chờ: chờ node chuyển khỏi `Ready`
sau khi tắt máy, chờ VM boot lại, chờ Pod hội tụ. **Thời gian chờ phụ thuộc cấu hình cluster và
phần cứng host**, nên mọi bước chờ đều viết dưới dạng vòng lặp có điều kiện dừng hoặc
`kubectl wait --timeout`, không phải một con số cố định.

---

## 2. Quy ước và an toàn

> **Cảnh báo — lab này TẮT và BẬT LẠI một máy ảo.** Đây là lab đầu tiên của chuỗi làm việc đó.
> Chỉ `lab-k8s-worker2` được tắt. **Không bao giờ tắt `lab-k8s-master`**: mất control plane là
> mất luôn đường quan sát, và không có gate nào chạy được nữa. Cũng không tắt `lab-k8s-worker1`:
> nó là nơi workload chạy trong lúc worker2 vắng mặt.

**Trình tự an toàn bắt buộc của B7**, không đảo thứ tự và không bỏ bước:

1. `cordon` worker2 — node ngừng nhận Pod mới.
2. `drain` worker2 — Pod đang chạy được trục xuất có kiểm soát và lên lại ở worker1.
3. Xác nhận worker2 **chỉ còn Pod DaemonSet** trước khi chạm vào lệnh tắt máy.
4. `sudo shutdown -h now` **trên chính worker2**, qua `ssh` từ master.
5. Xác nhận trên máy host rằng VM đã ở trạng thái *Powered off* trước khi bật lại.
6. Bật VM, chờ `kubectl wait --for=condition=Ready node/lab-k8s-worker2` PASS.
7. `uncordon` worker2, rồi chứng minh workload quay lại.

**Cách xác nhận node đã trở lại `Ready`** — dùng đúng lệnh này, không dựa vào việc `ssh` vào được:

```bash
kubectl wait --for=condition=Ready node/lab-k8s-worker2 --timeout=600s
kubectl get node lab-k8s-worker2 \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}{"\n"}'
```

**Đường quay lui khi worker2 không lên được:** xem [mục 4](#4-troubleshooting-của-lab-này) trước.
Nếu không sửa được trong vài phút thì tắt **cả ba VM** và restore **cả ba** về `04-metrics-ready`
— không bao giờ restore riêng một VM, xem ghi chú cuối
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường) — rồi bật lại theo
thứ tự master → worker 1 → worker 2 và chạy lại từ gate mở đầu.

**Trước khi chạy B0, xác nhận trên máy host rằng cả ba VM đều còn mốc `04-metrics-ready`.** Không
có mốc này thì không có đường lui. Chạy trên **máy host**, PowerShell:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$vmx = @(
  'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
  'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
  'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
)
foreach ($f in $vmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '04-metrics-ready') { "PASS: $f" } else { "FAIL: $f" }
}
```

**PASS:** đúng ba dòng `PASS:`. Không mở B0 khi còn dòng `FAIL:`. Đường dẫn `.vmx` ở trên theo máy
host đang dùng; sửa lại nếu VM nằm chỗ khác. Giữ nguyên cửa sổ PowerShell này — B7 dùng lại hai
biến `$vmrun` và `$vmx`.

Các quy ước còn lại:

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, **trừ khi ghi rõ
  node khác**. Lệnh cần đọc file trên worker chạy qua `ssh` từ master, đúng cách
  [Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) và [Lab 11a](LAB-11A-OBSERVABILITY.md) đã dùng.
- Các khối PowerShell chạy trên **máy host Windows**, không phải trên VM. Chúng luôn được mở đầu
  bằng dòng "Chạy trên máy host".
- **Chạy toàn bộ phần B trong cùng một phiên shell.** Gần như mọi gate so sánh với biến đặt ở B0
  (`NODES`, `W1`, `W2`, `MASTER`, `EV`, `WK`, `NODE_N0`, `DS_W2_0`) và hàm `cfgz`; mở shell mới
  giữa chừng là mất hết. Phiên shell này nằm trên master nên nó sống sót qua lúc worker2 tắt máy.
- **Lab này chỉ ĐỌC cấu hình node và cấu hình control plane.** Tuyệt đối không sửa
  `/var/lib/kubelet/config.yaml`, không sửa file nào trong `/etc/kubernetes/manifests`, không đổi
  feature gate, không restart kubelet bằng tay. B0.3 ghi checksum của sáu file đó và B11.2 đối
  chiếu lại — đó là gate chứng minh bạn không đụng vào.
- **Lab không cài thêm gì và không tạo snapshot mới.** Thành phần duy nhất lab tạo là namespace
  `lab-12` cùng object bên trong nó; toàn bộ bị xóa ở B11.
- **Fault injection chỉ trên `lab-k8s-worker2`.** Cordon, drain và tắt máy đều ghim vào đúng node
  này. `lab-k8s-worker1` chỉ nhận Pod bình thường.
- **Đừng dùng `--force` với `kubectl drain` trong lab này.** `--force` xóa cả Pod không có
  controller quản lý, tức xóa vĩnh viễn. Lab cố ý không để Pod trần nào còn sống lúc drain, nên
  không cần cờ đó. Nếu drain đòi `--force`, dừng lại và đọc
  [mục 4](#4-troubleshooting-của-lab-này).
- **`--delete-emptydir-data` xóa dữ liệu trong volume `emptyDir` của Pod bị trục xuất.** Trên
  cluster này các volume đó là thư mục tạm của add-on nên xóa được. Đừng mang thói quen này sang
  cluster có workload thật mà chưa kiểm tra Pod nào đang giữ dữ liệu ở `emptyDir`.
- Image dùng cho toàn bộ workload của lab là `busybox:1.37` đã khóa ở
  [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và đã có sẵn trên cả ba node từ
  Lab 00, nên lab không phụ thuộc mạng ra ngoài.
- **Mọi con số của cluster đều đọc từ cluster thật** — tên node, số worker, số replica, số Pod
  DaemonSet trên worker2, `gitVersion` của API server. Không con số nào của cluster được viết cứng
  vào gate.
- Manifest tạm ghi vào `~/lab-work/12/`; bằng chứng ghi vào `~/lab-evidence/12/`. Token sinh ra ở
  B9 chỉ tồn tại trong biến shell, **không ghi vào evidence**.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail.

---

# Phần B — Thực hành kiến thức giai đoạn 12

## B0. Chuẩn bị workspace và ảnh chụp "trước"

**Mục đích:** dựng chỗ làm việc, khóa tên node và số node vào biến, lấy cấu hình hiệu lực của ba
kubelet, và chụp checksum của sáu file cấu hình để B11 chứng minh lab không sửa gì.

### B0.1. Workspace, namespace và topology

```bash
mkdir -p ~/lab-work/12 ~/lab-evidence/12
WK=~/lab-work/12
EV=~/lab-evidence/12

kubectl config current-context
kubectl create namespace lab-12

MASTER='lab-k8s-master'
W1='lab-k8s-worker1'
W2='lab-k8s-worker2'
NODES="$MASTER $W1 $W2"

CP_N="$(kubectl get nodes -l 'node-role.kubernetes.io/control-plane' --no-headers | wc -l)"
WK_N="$(kubectl get nodes -l '!node-role.kubernetes.io/control-plane' --no-headers | wc -l)"
NODE_N0="$(kubectl get nodes --no-headers | wc -l)"
echo "control plane=$CP_N | worker=$WK_N | tong node=$NODE_N0"

test "$CP_N" -eq 1 && test "$WK_N" -eq 2 && test "$NODE_N0" -eq 3 \
  && test "$(kubectl get namespace lab-12 -o jsonpath='{.status.phase}')" = 'Active' \
  && echo 'PASS: dung topology mot control plane hai worker, namespace lab-12 da Active'
```

**Ý nghĩa:** `-l '!node-role.kubernetes.io/control-plane'` chọn node **không mang** label đó, tức
hai worker. Ba con số này là dữ kiện của B7 (bao nhiêu replica), B8 (số node có đổi không) và B10
(control plane một node hay nhiều node).

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B0.2. Cấu hình hiệu lực của ba kubelet

Cấu hình *hiệu lực* — gồm cả giá trị mặc định mà file trên đĩa không viết ra — đọc qua endpoint
`configz`, đi đúng đường control plane → kubelet mà
[tầng 6 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet) đã kiểm.
B5 và B6 dựa hẳn vào file này:

```bash
for n in $NODES; do
  kubectl get --raw "/api/v1/nodes/$n/proxy/configz" > "$EV/b0-configz-$n.json" 2>/dev/null || true
  echo "$n -> $(wc -c < "$EV/b0-configz-$n.json") byte"
done

OKZ=0
for n in $NODES; do
  grep -q '"kubeletconfig"' "$EV/b0-configz-$n.json" && OKZ=$(( OKZ + 1 ))
done
echo "configz doc duoc tren $OKZ/3 node"
test "$OKZ" -eq 3 && echo 'PASS: doc duoc cau hinh hieu luc cua ca ba kubelet'
```

Hàm rút một trường vô hướng ra khỏi file JSON đó, không cần `jq` — node lab không cài `jq`. Hàm
này giống bản dùng ở [Lab 11a](LAB-11A-OBSERVABILITY.md), giữ nguyên để hai lab đọc cùng một cách:

```bash
cfgz() { grep -o "\"$1\":[^,}]*" "$2" | head -1 | cut -d: -f2- | tr -d '"'; }

test -n "$(cfgz cgroupDriver "$EV/b0-configz-$W2.json")" \
  && echo "PASS: cfgz doc duoc — cgroupDriver = $(cfgz cgroupDriver "$EV/b0-configz-$W2.json")"
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện. Nếu `configz` không đọc được, dừng lại và xem
[mục 4](#4-troubleshooting-của-lab-này) trước khi đi tiếp.

### B0.3. Checksum của cấu hình node và của manifest control plane

```bash
{
  echo "$MASTER kubelet $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in "$W1" "$W2"; do
    echo "$n kubelet $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
  for f in kube-apiserver kube-scheduler kube-controller-manager; do
    echo "$MASTER $f $(sudo sha256sum /etc/kubernetes/manifests/$f.yaml | awk '{print $1}')"
  done
} | tee "$EV/b0-config-sha.txt"

test "$(wc -l < "$EV/b0-config-sha.txt")" -eq 6 \
  && test "$(awk '{print $3}' "$EV/b0-config-sha.txt" | grep -c '^[0-9a-f]\{64\}$')" -eq 6 \
  && echo 'PASS: ghi duoc checksum cua 3 cau hinh kubelet va 3 manifest control plane'
```

**Ý nghĩa:** lab đọc rất nhiều thứ trên node, và cám dỗ "sửa thử một dòng để xem graceful shutdown
chạy thế nào" là có thật. Gate này biến lời hứa "chỉ đọc" thành thứ kiểm chứng được ở B11.2. Nó
cũng bắt được một sự cố khác: node boot lại sau khi tắt máy mà cấu hình bị đổi.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B0.4. Ảnh chụp "trước" của worker2 và của cluster

Số Pod DaemonSet đang chạy trên worker2 là **dữ kiện quyết định** của B7: sau khi drain, con số đó
phải là toàn bộ những gì còn lại trên node; sau khi node boot lại, nó phải trở về đúng mức cũ.

```bash
ALL_W2_0="$(kubectl get pods -A \
  --field-selector "spec.nodeName=$W2,status.phase=Running" --no-headers 2>/dev/null | wc -l)"
DS_W2_0="$(kubectl get pods -A --field-selector "spec.nodeName=$W2,status.phase=Running" \
  -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].kind}{"\n"}{end}' \
  | grep -c '^DaemonSet$')"
echo "worker2 truoc lab: tong Pod Running=$ALL_W2_0 | do DaemonSet quan ly=$DS_W2_0"

kubectl get pods -A --field-selector "spec.nodeName=$W2,status.phase=Running" -o wide \
  | tee "$EV/b0-pods-worker2.txt"

test "$DS_W2_0" -gt 0 && test "$ALL_W2_0" -ge "$DS_W2_0" \
  && echo 'PASS: doc duoc so Pod va so Pod DaemonSet dang chay tren worker2'
```

**Ý nghĩa:** `--field-selector` lọc ngay ở API server theo `spec.nodeName` và `status.phase`, nên
con số không lẫn Pod đã `Succeeded` hay `Failed`. `ownerReferences[0].kind` cho biết ai tạo ra Pod;
Pod của DaemonSet có `kind: DaemonSet` ở đó. Đây chính là ranh giới mà `kubectl drain` dùng để
quyết định Pod nào nó **không** trục xuất được nếu thiếu `--ignore-daemonsets`.

```bash
{
  echo "=== $(date -Is) — trang thai truoc Lab 12 ==="
  echo '--- nodes'; kubectl get nodes -o wide
  echo '--- unschedulable'
  kubectl get nodes -o custom-columns='NODE:.metadata.name,UNSCHEDULABLE:.spec.unschedulable'
  echo '--- pdb'; kubectl get pdb -A 2>&1
  echo '--- top node'; kubectl top node 2>&1
  echo '--- storageclass'; kubectl get storageclass
  echo '--- ingressclass'; kubectl get ingressclass 2>&1
  echo '--- apiservice metrics'; kubectl get apiservice v1beta1.metrics.k8s.io 2>&1
  echo '--- namespaces'; kubectl get namespaces
} | tee "$EV/b0-truoc.txt"

test -s "$EV/b0-truoc.txt" && echo 'PASS: da ghi anh chup truoc cua cluster'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B1. Bốn nhóm việc quản trị, bốn hiện vật trong cluster

**Mục đích:** bài [155](../155-cluster-administration-vi.md) là **trang mục lục** của cả nhánh
quản trị cluster. Giá trị của nó nằm ở cấu trúc: biết công việc quản trị chia thành nhóm nào để
khi gặp câu hỏi vận hành thì biết mở nhánh nào. Ở đây bạn đi tìm, với từng nhóm, một hiện vật có
thật trong cluster của mình — mục lục chỉ hữu ích khi nó nối được với thứ sờ được.

### B1.1. Nhóm *Lập kế hoạch cho cluster*

```bash
kubectl get nodes -o wide | tee "$EV/b1-nodes.txt"

for n in $NODES; do
  kubectl get node "$n" -o jsonpath='{.metadata.name}{"\t"}{.status.nodeInfo.osImage}{"\t"}{.status.nodeInfo.kernelVersion}{"\t"}{.status.nodeInfo.containerRuntimeVersion}{"\n"}'
done | tee "$EV/b1-nodeinfo.txt"

PID_N="$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.providerID}{"\n"}{end}' \
  | grep -c .)"
echo "so node co providerID = $PID_N"

test "$PID_N" -eq 0 && test "$(wc -l < "$EV/b1-nodeinfo.txt")" -eq 3 \
  && echo 'PASS: khong node nao co providerID — cluster tu host, khong phai IaaS'
```

**Ý nghĩa:** `.spec.providerID` là chuỗi định danh tài nguyên của nhà cung cấp cloud đứng sau Node.
Rỗng ở cả ba node nghĩa là **không có nhà cung cấp cloud nào biết tới ba máy này**. Đối chiếu với
danh sách cân nhắc ở mục *Lập kế hoạch cho cluster*: bạn đang **tự host**, **on-premises**, trên
**máy ảo**, cluster **nhiều node nhưng không HA**, và mục đích là **vận hành**. Chính dữ kiện
`providerID` rỗng này còn quay lại ở B8 và B10.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B1.2. Nhóm *Quản lý cluster*

Bài xếp vào nhóm này đúng hai việc: quản lý node, và resource quota cho cluster dùng chung.

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,ROLES:.metadata.labels.node-role\.kubernetes\.io/control-plane,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory' \
  | tee "$EV/b1-allocatable.txt"

RQ_N="$(kubectl get resourcequota -A --no-headers 2>/dev/null | wc -l)"
LR_N="$(kubectl get limitrange -A --no-headers 2>/dev/null | wc -l)"
echo "resourcequota=$RQ_N | limitrange=$LR_N"

test "$RQ_N" -eq 0 && test "$LR_N" -eq 0 \
  && echo 'PASS: chua namespace nao co chinh sach tai nguyen — dung moc 04-metrics-ready'
```

**Ý nghĩa:** hai việc của nhóm này bạn đã làm ở hai lab khác nhau — node ở
[Lab 1a](LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md), quota ở
[Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md). Trang 155 chỉ ghép chúng lại thành một nhóm
công việc. Con số 0 ở đây là bằng chứng Lab 7b đã cleanup sạch.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B1.3. Nhóm *Bảo mật cluster* và nhánh *Bảo mật kubelet*

```bash
AUTHZ="$(sudo grep -oE '\-\-authorization-mode=[^ ]+' \
  /etc/kubernetes/manifests/kube-apiserver.yaml | head -1 | cut -d= -f2)"
echo "authorization-mode cua API server = $AUTHZ"
case "$AUTHZ" in
  *RBAC*) echo 'PASS: API server dang cuong che phan quyen bang RBAC' ;;
  *)      echo "FAIL: khong thay RBAC trong authorization-mode ($AUTHZ)" ;;
esac
```

Ba mục của nhánh *Bảo mật kubelet* — giao tiếp control plane ↔ node, TLS bootstrapping, và
authn/authz cho kubelet — đều để lại dấu vết đọc được:

```bash
KA_OK=0
for n in $NODES; do
  M="$(grep -c '"mode":"Webhook"' "$EV/b0-configz-$n.json")"
  A="$(grep -c '"anonymous":{"enabled":false}' "$EV/b0-configz-$n.json")"
  echo "$n kubelet authorization Webhook=$M | anonymous bi tat=$A"
  test "$M" -ge 1 && KA_OK=$(( KA_OK + 1 ))
done
test "$KA_OK" -eq 3 \
  && echo 'PASS: ca ba kubelet uy quyen quyet dinh phan quyen cho API server (Webhook)'

ssh "$W2" 'test -e /var/lib/kubelet/pki/kubelet-client-current.pem' \
  && echo 'PASS: kubelet cua worker2 dang dung client certificate do TLS bootstrapping cap'
```

**Ý nghĩa:** kubelet **không** tự quyết định ai được gọi nó. `authorization.mode: Webhook` nghĩa là
mỗi request tới kubelet được hỏi ngược lại API server, và `anonymous.enabled: false` chặn client
không danh tính. Đó là hai trong ba mục mà bài 155 tách riêng thành nhánh *Bảo mật kubelet*. File
`kubelet-client-current.pem` là kết quả của TLS bootstrapping: kubelet tự xin certificate cho
chính nó thay vì được phát bằng tay. Đường control plane → kubelet mà B0.2 vừa dùng là mục thứ ba.

**PASS:** ba dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B1.4. Nhóm *Các dịch vụ cluster tùy chọn*

```bash
DNS_RDY="$(kubectl -n kube-system get deployment coredns -o jsonpath='{.status.readyReplicas}')"
MS_AVAIL="$(kubectl get apiservice v1beta1.metrics.k8s.io \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
echo "CoredDNS readyReplicas=$DNS_RDY | APIService metrics.k8s.io Available=$MS_AVAIL"

test -n "$DNS_RDY" && test "$DNS_RDY" -ge 1 && test "$MS_AVAIL" = 'True' \
  && echo 'PASS: ca hai dich vu tuy chon cua nhom nay dang chay: DNS va nguon metric'
```

**Ý nghĩa:** bài 155 gọi DNS và logging/monitoring là **dịch vụ tùy chọn** — Kubernetes chạy được
mà không có chúng, nhưng gần như không ai vận hành thiếu chúng. Trên cluster này, DNS đến từ Lab 00
và nguồn metric đến từ Lab 11a. Phần logging tập trung thì **chưa có** và đó là ranh giới bạn đã
ghi lại ở Lab 11a.

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B2. Kiểm kê add-on đang chạy và xếp theo nhóm chức năng

**Mục đích:** bài [165](../165-addons-vi.md) là một **danh mục**, không phải bài giảng. Ở đây bạn
làm ngược lại việc đọc danh mục: đi từ cluster của mình lên danh mục, và phát hiện ra rằng có
những thứ đang chạy mà danh mục **không** liệt kê — đúng như trang tự cảnh báo.

### B2.1. Toàn cảnh workload hệ thống

```bash
kubectl get daemonset -A -o wide | tee "$EV/b2-daemonset.txt"
kubectl get deployment -A -o wide | tee "$EV/b2-deployment.txt"

test -s "$EV/b2-daemonset.txt" && test -s "$EV/b2-deployment.txt" \
  && echo 'PASS: da ghi danh sach DaemonSet va Deployment cua toan cluster'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B2.2. Nhóm *Mạng và Network Policy*

```bash
CNI_CONF="$(ssh "$W2" 'sudo ls /etc/cni/net.d/ 2>/dev/null' | wc -l)"
ssh "$W2" 'sudo ls -l /etc/cni/net.d/' | tee "$EV/b2-cni-netd.txt"
NP_KIND="$(kubectl api-resources --api-group=networking.k8s.io --no-headers \
  | grep -c 'networkpolicies')"
echo "file cau hinh CNI tren worker2=$CNI_CONF | NetworkPolicy co trong API=$NP_KIND"

test "$CNI_CONF" -ge 1 && test "$NP_KIND" -eq 1 \
  && echo 'PASS: nhom Mang va Network Policy co dai dien that tren node'
```

**Ý nghĩa:** đây là nhóm dài nhất của danh mục và cũng là nhóm bạn đã trực tiếp đổi ở Lab 5b. Bài
165 mô tả Flannel đúng là "**nhà cung cấp mạng overlay**", không nhắc network policy; còn các mục
như Calico, Canal, Cilium, Antrea, OVN-Kubernetes thì có kèm network policy. Chính khác biệt một
dòng đó là lý do lộ trình phải đổi CNI. Lưu ý: `NetworkPolicy` **có trong API** kể cả khi CNI không
thực thi nó — sự tồn tại của kind không chứng minh chính sách được cưỡng chế; phần chứng minh đó
đã làm ở Lab 5b.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B2.3. Bốn nhóm còn lại: cái nào có, cái nào không

```bash
KSM_N="$(kubectl get deployment,statefulset -A --no-headers 2>/dev/null \
  | grep -ci 'kube-state-metrics')"
NPD_N="$(kubectl get daemonset -A --no-headers 2>/dev/null | grep -ci 'node-problem-detector')"
UI_N="$(kubectl get deployment -A --no-headers 2>/dev/null | grep -ciE 'dashboard|headlamp')"
DNS_N="$(kubectl -n kube-system get deployment coredns --no-headers 2>/dev/null | wc -l)"
echo "kube-state-metrics=$KSM_N | node-problem-detector=$NPD_N | UI=$UI_N | CoreDNS=$DNS_N"

test "$DNS_N" -eq 1 \
  && echo 'PASS: nhom Kham pha dich vu co dai dien — CoreDNS'
test "$KSM_N" -eq 0 && test "$NPD_N" -eq 0 && test "$UI_N" -eq 0 \
  && echo 'PASS: ba nhom Do luong, Ha tang va Truc quan hoa dang TRONG tren cluster nay'
```

**Ý nghĩa:** nhóm *Đo lường* của danh mục chỉ có đúng một mục,
[kube-state-metrics](../163-kube-state-metrics-vi.md), và bạn **không** cài nó ở Lab 11a — đó
chính là khoảng trống "metric trạng thái đối tượng" mà Lab 11a đã đo và ghi lại. Nhóm *Hạ tầng* có
node-problem-detector, thứ báo sự cố hệ thống của máy dưới dạng Event hoặc Node condition; nếu
cluster này có nó, B7 sẽ có thêm một nguồn tín hiệu lúc worker2 tắt. Không có nó cũng là một câu
trả lời hợp lệ, miễn là bạn biết mình đang thiếu gì.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B2.4. Ba thứ đang chạy mà danh mục không liệt kê

```bash
IC_N="$(kubectl get ingressclass --no-headers 2>/dev/null | wc -l)"
SC_DEF="$(kubectl get storageclass \
  -o jsonpath='{range .items[*]}{.metadata.name}{"="}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}' \
  | grep -c '=true')"
MS_AVAIL="$(kubectl get apiservice v1beta1.metrics.k8s.io \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
echo "IngressClass=$IC_N | StorageClass mac dinh=$SC_DEF | metrics APIService=$MS_AVAIL"

kubectl get storageclass -o wide | tee "$EV/b2-storageclass.txt"

test "$IC_N" -ge 1 && test "$SC_DEF" -eq 1 && test "$MS_AVAIL" = 'True' \
  && echo 'PASS: ba add-on ngoai danh muc 165 deu dang chay'
```

**Ý nghĩa:** ingress controller của Lab 5b, dynamic provisioner của Lab 6a và metrics-server của
Lab 11a **đều là add-on** theo đúng định nghĩa của bài — thứ mở rộng chức năng Kubernetes, phải
cài thêm, là dự án bên thứ ba — nhưng **không** có tên trong danh mục. Trang tự nói rõ: "danh sách
này không nhằm liệt kê đầy đủ tất cả". Kết luận đúng khi tra không thấy một công cụ là *danh mục
chưa liệt kê nó*, không phải *nó không tồn tại*.

Ghi bảng phân loại vào evidence để checkpoint có cái đối chiếu:

```bash
{
  echo "=== $(date -Is) — add-on cua cluster xep theo nhom cua bai 165 ==="
  echo "Mang va Network Policy : CNI cua Lab 5b (file cau hinh: $CNI_CONF tren worker2)"
  echo "Kham pha dich vu       : CoreDNS ($DNS_N Deployment trong kube-system)"
  echo "Truc quan hoa & Dieu khien : khong co ($UI_N)"
  echo "Ha tang                : khong co node-problem-detector ($NPD_N)"
  echo "Do luong               : khong co kube-state-metrics ($KSM_N)"
  echo "Ngoai danh muc         : ingress controller ($IC_N IngressClass), provisioner"
  echo "                         ($SC_DEF StorageClass mac dinh), metrics-server ($MS_AVAIL)"
} | tee "$EV/b2-phan-loai.txt"

test "$(wc -l < "$EV/b2-phan-loai.txt")" -ge 7 && echo 'PASS: da ghi bang phan loai add-on'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B3. Phiên bản tương thích của control plane

**Mục đích:** bài [168](../168-compatibility-version-vi.md) giới thiệu một ý tưởng đổi cách nghĩ về
nâng cấp: **đổi binary và đổi hành vi là hai việc tách rời được**. Ở đây bạn chứng minh cluster của
mình đang ở trạng thái mặc định — không tách gì cả — và đọc đúng ràng buộc của flag.

### B3.1. Không thành phần nào đặt `--emulated-version`

```bash
for f in kube-apiserver kube-controller-manager kube-scheduler; do
  echo "--- $f: $(sudo grep -c 'emulated-version' /etc/kubernetes/manifests/$f.yaml) dong khop"
done | tee "$EV/b3-emulated.txt"

EMU_N=0
for f in kube-apiserver kube-controller-manager kube-scheduler; do
  EMU_N=$(( EMU_N + $(sudo grep -c 'emulated-version' /etc/kubernetes/manifests/$f.yaml) ))
done
echo "tong so dong dat --emulated-version = $EMU_N"

test "$EMU_N" -eq 0 \
  && echo 'PASS: khong thanh phan control plane nao dat --emulated-version'
```

**Ý nghĩa:** không đặt flag nghĩa là emulation version **bằng đúng** binary version, tức mọi khả
năng của binary đều khả dụng và không khả năng nào bị cắt. Đó là trạng thái mặc định của một
cluster kubeadm và cũng là trạng thái bạn muốn ở lab.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B3.2. Binary version đọc từ hai nguồn độc lập

```bash
SRV_VER="$(kubectl get --raw /version | grep -o '"gitVersion":"[^"]*"' | cut -d'"' -f4)"
IMG_VER="$(sudo grep -oE 'image: registry\.k8s\.io/kube-apiserver:[^[:space:]]+' \
  /etc/kubernetes/manifests/kube-apiserver.yaml | head -1 | cut -d: -f3)"
echo "API server gitVersion=$SRV_VER | tag image trong manifest=$IMG_VER"

test -n "$SRV_VER" && test "$SRV_VER" = "$IMG_VER" \
  && echo 'PASS: hai nguon doc lap cho cung mot binary version'
```

Bây giờ đối chiếu với baseline. **Không chép con số vào lab** — mở
[bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa), đọc dòng
*Kubernetes control plane*, rồi điền vào biến dưới đây:

```bash
export K8S_BASELINE='<phien ban Kubernetes control plane trong bang A1.3, dang vX.Y.Z>'

case "$K8S_BASELINE" in
  ''|*'<'*) echo 'FAIL: chua dien gia tri that vao K8S_BASELINE — quay lai bang A1.3' ;;
  v*)       echo "OK: K8S_BASELINE = $K8S_BASELINE" ;;
  *)        echo 'FAIL: gia tri phai bat dau bang chu v' ;;
esac

test "$SRV_VER" = "$K8S_BASELINE" \
  && echo 'PASS: cluster van dung dung baseline cua bang A1.3, khong bi ha bang emulation'
```

**Ý nghĩa:** ràng buộc cứng của bài là `--emulated-version` **phải ≤ `binaryVersion`**: bạn giả lập
được phiên bản cũ hơn binary đang chạy, không bao giờ giả lập được phiên bản mới hơn. Hai quy tắc
đối xứng đi kèm: khả năng được **giới thiệu sau** emulation version thì **không** khả dụng, khả
năng bị **loại bỏ sau** emulation version thì **vẫn** khả dụng. Nghĩa là emulation cắt cả hai
chiều — giữ được cái cũ nhưng mất quyền dùng cái mới.

Công dụng thực dụng chỉ hiện ra trong một lần nâng cấp: triển khai binary mới trong khi hành vi
giữ nguyên, xác nhận cluster ổn, rồi mới nâng emulation version thành một bước riêng — và nếu bước
đó gây sự cố thì lùi lại chỉ là đổi một flag, không phải hạ cấp binary. Việc đó thuộc
[giai đoạn 17](../00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster).

**PASS:** hai dòng `PASS:` của bước này xuất hiện, và dòng `OK: K8S_BASELINE = …` chứa con số thật.

---

## B4. Lease của control plane ở góc vận hành

**Mục đích:** [Lab 1c](LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md#b3-lease-heartbeat-leader-election-và-api-server-identity)
đã đọc Lease ở mức khái niệm. Ở đây bạn đọc chúng như một **tín hiệu vận hành**: ai đang là leader,
lần gia hạn gần nhất lúc nào, và quyền leader đã đổi chủ bao nhiêu lần. Bài
[167](../167-coordinated-leader-election-vi.md) thêm một tầng lên trên cơ chế đó; B4.3 chứng minh
tầng ấy đang tắt.

### B4.1. Năm trường của Lease control plane

```bash
kubectl -n kube-system get lease kube-controller-manager kube-scheduler \
  -o custom-columns='NAME:.metadata.name,HOLDER:.spec.holderIdentity,ACQUIRE:.spec.acquireTime,RENEW:.spec.renewTime,DUR:.spec.leaseDurationSeconds,TRANS:.spec.leaseTransitions' \
  | tee "$EV/b4-lease.txt"

HOLDER="$(kubectl -n kube-system get lease kube-scheduler \
  -o jsonpath='{.spec.holderIdentity}')"
DUR="$(kubectl -n kube-system get lease kube-scheduler \
  -o jsonpath='{.spec.leaseDurationSeconds}')"
echo "holderIdentity cua kube-scheduler = $HOLDER | leaseDurationSeconds = $DUR"

case "$HOLDER" in
  *"$MASTER"*) echo "PASS: leader dang nam tren $MASTER — dung topology mot control plane" ;;
  *)           echo "FAIL: holderIdentity khong chua ten master ($HOLDER)" ;;
esac
test -n "$DUR" && test "$DUR" -gt 0 && echo 'PASS: doc duoc leaseDurationSeconds'
```

**Ý nghĩa:** `holderIdentity` là danh tính của leader hiện tại — trên cluster kubeadm nó là chuỗi
dựa trên hostname của node chạy static Pod. `leaseDurationSeconds` là khoảng hiệu lực; ứng viên
phải chờ hết khoảng này cộng một grace period nhỏ trước khi thử giành một Lease đã hết hạn.
`leaseTransitions` đếm số lần quyền leader đổi chủ — trên cluster một control plane, con số này chỉ
tăng khi Pod control plane khởi động lại.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B4.2. `renewTime` tiến lên — leader đang còn sống

```bash
R1="$(kubectl -n kube-system get lease kube-scheduler -o jsonpath='{.spec.renewTime}')"
sleep 20
R2="$(kubectl -n kube-system get lease kube-scheduler -o jsonpath='{.spec.renewTime}')"
echo "renewTime: $R1  ->  $R2"

test -n "$R1" && test -n "$R2" && test "$R1" != "$R2" \
  && echo 'PASS: leader dang dinh ky gia han Lease — renewTime tien len'
```

**Ý nghĩa:** leader gia hạn Lease bằng cách cập nhật `renewTime`, thường mỗi `leaseDurationSeconds`
÷ 2 để tránh xung đột lúc Lease sắp hết hạn. Chừng nào việc gia hạn còn diễn ra trước khi Lease hết
hạn thì leader hiện tại giữ quyền. Nếu leader crash hoặc mất liên lạc, Lease hết hạn và các instance
khỏe mạnh khác tiến hành cuộc bầu chọn mới. **Đây chính là cơ chế bạn sẽ thấy dừng lại ở B7.5** —
lần đó trên Node Lease của worker2 thay vì Lease của scheduler.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B4.3. Bầu chọn leader có phối hợp đang tắt

```bash
kubectl api-resources --api-group=coordination.k8s.io | tee "$EV/b4-api-resources.txt"

LC_N="$(kubectl api-resources --api-group=coordination.k8s.io --no-headers \
  | grep -ci 'leasecandidate')"
V1B="$(kubectl get --raw /apis/coordination.k8s.io | grep -c 'v1beta1')"
FG_N="$(sudo grep -c 'CoordinatedLeaderElection' /etc/kubernetes/manifests/kube-apiserver.yaml)"
echo "leasecandidates trong API=$LC_N | v1beta1 duoc phuc vu=$V1B | feature gate=$FG_N"

test "$LC_N" -eq 0 && test "$V1B" -eq 0 && test "$FG_N" -eq 0 \
  && echo 'PASS: CoordinatedLeaderElection dang TAT — dung mac dinh cua tinh nang beta nay'
```

**Ý nghĩa:** tính năng này ở mức beta và **mặc định tắt**; muốn bật phải có **cả hai**: feature gate
`CoordinatedLeaderElection` và API group `coordination.k8s.io/v1beta1`. Cả hai đều vắng ở đây, nên
`LeaseCandidate` không tồn tại và các thành phần vẫn bầu leader theo kiểu thông thường.

Điều quan trọng phải hiểu đúng: bật tính năng ấy **không thay thế** cơ chế Lease. Các ứng viên vẫn
phối hợp qua **một Lease dùng chung**; cái được thêm vào là đối tượng `LeaseCandidate` mang danh
tính, binary version và emulation version của từng instance. Khác biệt thật nằm ở **cách chọn ai
thắng**: bầu chọn thông thường về bản chất là ai giành được Lease trước thì thắng — do điều khiển
đồng thời lạc quan qua `resourceVersion`, chỉ một lần cập nhật thành công; còn bầu chọn có phối hợp
chọn leader **một cách xác định**, theo chiến lược dựng sẵn duy nhất hiện nay là
`OldestEmulationVersion` — ưu tiên emulation version thấp nhất, rồi binary version, rồi thời điểm
tạo. Đó là chỗ bài 167 nối thẳng vào `--emulated-version` của B3.

Và trên cluster một control plane thì **không có gì để tranh**: mỗi thành phần đúng một instance.
Giá trị thật của cả hai cơ chế chỉ xuất hiện ở cluster HA — nội dung của lab 8b và 8c.

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B5. Swap trên ba node

**Mục đích:** ở [A4.1 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites)
bạn đã chạy `swapoff -a` theo kiểu copy-paste. Đây là chỗ kiểm chứng trạng thái đó còn đúng và đọc
ranh giới mà bài [170](../170-swap-memory-management-vi.md) vẽ ra. Lab **không bật swap**.

### B5.1. Trạng thái swap thật của ba node

```bash
cat > "$WK/doc-swap.sh" <<'EOS'
swapon --show | wc -l
free -m | sed -n 's/^Swap:[[:space:]]*\([0-9][0-9]*\).*/\1/p'
grep -cE '^[^#].*[[:space:]]swap[[:space:]]' /etc/fstab
EOS

for n in $NODES; do
  if [ "$n" = "$MASTER" ]; then OUT="$(sh "$WK/doc-swap.sh")"
  else OUT="$(ssh "$n" 'sh -s' < "$WK/doc-swap.sh")"; fi
  L="$(echo "$OUT" | sed -n 1p)"
  T="$(echo "$OUT" | sed -n 2p)"
  F="$(echo "$OUT" | sed -n 3p)"
  echo "$n: swapon=$L swaptotal=$T fstab=$F"
done | tee "$EV/b5-swap-node.txt"

SWAP_OK="$(grep -cF 'swapon=0 swaptotal=0 fstab=0' "$EV/b5-swap-node.txt")"
echo "so node tat swap hoan toan = $SWAP_OK/3"

test "$SWAP_OK" -eq 3 \
  && echo 'PASS: ca ba node tat swap, va fstab khong bat lai sau reboot'
```

**Ý nghĩa:** ba số này trả lời ba câu khác nhau. `swapon --show` in ra **thiết bị swap đang hoạt
động**, không có thì không in dòng nào. `free -m` cho tổng dung lượng swap. Dòng `swap` còn hiệu
lực trong `/etc/fstab` là thứ quyết định swap có bật lại sau reboot hay không — câu hỏi rất đáng
hỏi trong lab này, vì **B7 sẽ reboot worker2**. Nếu `fstab` còn dòng swap chưa comment, node quay
lại với swap bật và kubelet có thể không khởi động được.

Chú ý cách đếm: vòng lặp bị đẩy vào **subshell** vì nó đứng trước `| tee`, nên một biến đếm tăng
bên trong đó **không thoát ra được**. Vì vậy `SWAP_OK` được tính lại từ chính file evidence bằng
`grep -cF` chứ không cộng dồn trong vòng lặp. Cùng lý do đó, mọi vòng lặp có `| tee` trong lab này
đều lấy kết quả từ file, không từ biến.

**PASS:** dòng `PASS:` của bước này xuất hiện. Nếu một node fail, dừng lại và xem
[mục 4](#4-troubleshooting-của-lab-này) **trước khi** chạy B7.

### B5.2. Node không báo cáo dung lượng swap

```bash
kubectl get nodes -o json > "$EV/b5-nodes.json"
SWAP_FIELD="$(grep -c '"swap"' "$EV/b5-nodes.json")"
echo "so lan truong swap xuat hien trong Node object = $SWAP_FIELD"

test "$SWAP_FIELD" -eq 0 \
  && echo 'PASS: khong node nao bao cao swap capacity — dung voi node khong duoc cap phat swap'
```

**Ý nghĩa:** bài 170 mô tả trường trạng thái `node.status.nodeInfo.swap.capacity`. Trên cluster có
swap, lệnh go-template của bài in ra số byte cho từng node; ở đây trường ấy **không tồn tại**, và
bài nói rõ điều đó "rất có thể nghĩa là node không được cấp phát swap". Đó là cách đọc trạng thái
swap từ phía Kubernetes, không phải từ phía OS như B5.1.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B5.3. `failSwapOn` và `memorySwap` hiệu lực của kubelet

```bash
for n in $NODES; do
  FS="$(grep -o '"failSwapOn":[a-z]*' "$EV/b0-configz-$n.json" | head -1 | cut -d: -f2)"
  MS="$(grep -o '"memorySwap":{[^}]*}' "$EV/b0-configz-$n.json" | head -1)"
  echo "$n failSwapOn=${FS:-<khong co trong configz>} memorySwap=${MS:-<khong co trong configz>}"
done | tee "$EV/b5-kubelet-swap.txt"

FS_OK="$(grep -c 'failSwapOn=[a-z]' "$EV/b5-kubelet-swap.txt")"
test "$FS_OK" -eq 3 \
  && echo 'PASS: doc duoc gia tri failSwapOn hieu luc cua ca ba kubelet'
```

**Ý nghĩa và ranh giới cần nhớ:**

- Mặc định trên Linux, **kubelet không khởi động trên node đang bật swap**; muốn chạy phải đặt
  `failSwapOn: false`. Node Windows thì ngược lại: kubelet không khởi động khi swap **tắt**.
- Cho kubelet chạy được với swap **chưa** đồng nghĩa workload dùng được swap. Mặc định kubelet chỉ
  thị cho CRI cấp phát **0 byte swap** cho workload, và hành vi mặc định là **`NoSwap`**. Chỉ
  `LimitedSwap` mới cho workload đụng vào swap.
- Ngay cả với `NoSwap`, các tiến trình **ngoài** container do Kubernetes quản lý — dịch vụ systemd,
  và cả bản thân kubelet — **vẫn có thể** bị swap.
- Với `LimitedSwap`, chỉ Pod QoS **Burstable** được dùng swap; `BestEffort` và `Guaranteed` bị cấm.
- **Scheduler không xét swap** khi đặt Pod: Pod chỉ request `memory`, không request swap. Hệ quả là
  rủi ro "hàng xóm ồn ào", và cách phòng mà bài đề xuất là **gắn taint cho node có swap** — đúng cơ
  chế bạn đã làm ở [Lab 7a](LAB-7A-LAP-LICH-VA-EVICTION.md#b4-taint-và-toleration--mặt-đối-ngẫu-của-affinity).

**Vì sao baseline tắt swap:** cluster lab chạy trên đĩa của một máy host duy nhất. Bài nói thẳng
rằng hiệu năng node có swap phụ thuộc thiết bị lưu trữ bên dưới và tệ đi đáng kể trong môi trường
bị hạn chế IOPS. Thêm nữa, bật swap làm **giảm tính dự đoán được** — thứ đối lập hẳn với mục tiêu
của một môi trường học: mỗi phép đo ở lab 3c, 7b, 11a đều muốn kết quả lặp lại được. Và ngưỡng
eviction trên node có swap phải đặt tương quan với `vm.min_free_kbytes`, một bài toán chỉnh ngưỡng
thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy).

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B6. Graceful node shutdown — đọc cấu hình thật

**Mục đích:** trả lời một câu hỏi vận hành cụ thể trước khi B7 tắt máy: **nếu tôi gõ `shutdown -h
now` trên worker2 mà không drain trước, kubelet có kịp chấm dứt Pod nhẹ nhàng không?** Bài
[169](../169-node-shutdown-vi.md) cảnh báo cái bẫy lớn nhất ở đây, và lab **không sửa cấu hình
kubelet** để "chữa" nó.

### B6.1. Hai con số quyết định

```bash
for n in $NODES; do
  SGP="$(grep -o '"shutdownGracePeriod":"[^"]*"' "$EV/b0-configz-$n.json" | head -1 | cut -d'"' -f4)"
  SGC="$(grep -o '"shutdownGracePeriodCriticalPods":"[^"]*"' "$EV/b0-configz-$n.json" \
    | head -1 | cut -d'"' -f4)"
  echo "$n shutdownGracePeriod=${SGP:-<khong co>} shutdownGracePeriodCriticalPods=${SGC:-<khong co>}"
done | tee "$EV/b6-shutdown.txt"

SGP2="$(grep -o '"shutdownGracePeriod":"[^"]*"' "$EV/b0-configz-$W2.json" \
  | head -1 | cut -d'"' -f4)"
SGC2="$(grep -o '"shutdownGracePeriodCriticalPods":"[^"]*"' "$EV/b0-configz-$W2.json" \
  | head -1 | cut -d'"' -f4)"

case "${SGP2:-0s}|${SGC2:-0s}" in
  '0s|0s')
    echo 'PASS: ca hai bang 0s — graceful node shutdown KHONG duoc kich hoat tren worker2' ;;
  *)
    echo "PASS: hai gia tri khac 0s (${SGP2}/${SGC2}) — tinh nang DANG kich hoat; ghi lai va doc muc 4" ;;
esac
```

**Ý nghĩa — đây là cái bẫy của bài:** feature gate `GracefulNodeShutdown` **bật mặc định từ 1.21**,
nhưng `shutdownGracePeriod` và `shutdownGracePeriodCriticalPods` **mặc định đều bằng 0**, nghĩa là
tính năng **không hoạt động** cho tới khi bạn đặt cả hai khác 0. Thấy dòng "enabled by default"
rồi kết luận mình đang được bảo vệ là chỗ dễ ngã nhất của giai đoạn 12.

Cách đọc hai con số khi chúng khác 0: `shutdownGracePeriod` là **tổng**; phần dành cho Pod thông
thường bằng tổng trừ đi `shutdownGracePeriodCriticalPods`. Với `30s`/`10s` thì 20 giây đầu cho Pod
thường, 10 giây cuối cho Pod quan trọng — và **Pod thông thường bị chấm dứt trước**.

**PASS:** một trong hai dòng `PASS:` của bước này xuất hiện, kèm hai giá trị thật cho cả ba node.

### B6.2. Feature gate không bị ai tắt

```bash
FGS="$(grep -o '"featureGates":{[^}]*}' "$EV/b0-configz-$W2.json" | head -1)"
echo "featureGates cua kubelet worker2: ${FGS:-<khong dat gate nao>}"
GNS="$(echo "$FGS" | grep -c 'GracefulNodeShutdown')"
echo "so lan GracefulNodeShutdown xuat hien trong featureGates = $GNS"

test "$GNS" -eq 0 \
  && echo 'PASS: khong ai tat GracefulNodeShutdown bang feature gate — no giu mac dinh'
```

**Ý nghĩa:** kết hợp với B6.1, bạn có câu trả lời đầy đủ: gate **bật**, nhưng hai con số **bằng 0**,
nên tính năng **không chạy**. Đây đúng là tình huống mà bài mô tả ở mục *Xử lý tắt node không nhẹ
nhàng*: "lỗi của người dùng, tức `ShutdownGracePeriod` và `ShutdownGracePeriodCriticalPods` không
được cấu hình đúng". Lab **không sửa** nó — sửa cấu hình kubelet thuộc giai đoạn 20 — mà rút ra hệ
quả vận hành: **trên cluster này, drain trước khi tắt máy không phải lựa chọn, mà là bắt buộc.**

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B6.3. Ba điều kiện nền mà bài cảnh báo

```bash
cat > "$WK/doc-shutdown-prereq.sh" <<'EOS'
dpkg-query -W -f='goi unattended-upgrades: ${Status}\n' unattended-upgrades 2>/dev/null \
  || echo 'goi unattended-upgrades: chua cai'
if [ -e /etc/systemd/logind.conf.d/unattended-upgrades-logind-maxdelay.conf ]; then
  echo "file logind maxdelay: co -> $(readlink -f /etc/systemd/logind.conf.d/unattended-upgrades-logind-maxdelay.conf)"
else
  echo 'file logind maxdelay: khong co'
fi
echo "systemd-logind: $(systemctl is-active systemd-logind)"
EOS

ssh "$W2" 'sh -s' < "$WK/doc-shutdown-prereq.sh" | tee "$EV/b6-prereq-$W2.txt"

grep -q 'file logind maxdelay' "$EV/b6-prereq-$W2.txt" \
  && grep -q 'systemd-logind: active' "$EV/b6-prereq-$W2.txt" \
  && echo 'PASS: doc duoc ba dieu kien nen ma bai 169 canh bao, va systemd-logind dang chay'
```

**Ý nghĩa:** tính năng tắt node nhẹ nhàng trên Linux **phụ thuộc systemd**, vì nó dùng
[systemd inhibitor lock](https://www.freedesktop.org/wiki/Software/systemd/inhibit/) để trì hoãn
việc tắt máy. `systemd-logind` không chạy thì không có lock để lấy. Và bài có một khối *Thận trọng*
riêng: cấu hình thông thường của gói `unattended-upgrades` **xung đột** với tính năng này — nó tùy
chỉnh grace period khi tắt máy, khiến kubelet không lấy được lock cần thiết. Vấn đề xảy ra khi
`shutdownGracePeriod` lớn hơn 30 giây, và cách né là biến file
`/etc/systemd/logind.conf.d/unattended-upgrades-logind-maxdelay.conf` thành symlink tới `/dev/null`.

Ba dòng bạn vừa ghi là **ảnh chụp điều kiện nền**, để khi nào thật sự bật tính năng ở giai đoạn 20
thì biết mình phải xử lý gì trước. Lab **không** đụng vào chúng.

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B7. Vòng bảo trì node: cordon → drain → tắt → bật → uncordon

**Mục đích:** đây là xương sống của lab. Bạn chạy trọn một chu trình bảo trì phần cứng trên
`lab-k8s-worker2` và chứng minh bằng con số rằng workload **dịch chuyển** sang worker còn lại rồi
**quay về**. Mỗi bước có một câu hỏi riêng, đừng gộp chúng lại.

> **Đọc lại [mục 2](#2-quy-ước-và-an-toàn) trước khi chạy B7.** Bước B7.5 tắt một máy ảo. Không
> chạy nó nếu gate snapshot `04-metrics-ready` ở mục 2 chưa PASS, hoặc nếu B5.1 báo `/etc/fstab`
> còn dòng swap chưa comment.

### B7.1. Workload nền trải trên cả hai worker

```bash
REPL=$(( WK_N * 2 ))
echo "so worker=$WK_N -> so replica=$REPL"

cat > "$WK/b7-web.yaml" <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: lab-12
spec:
  replicas: $REPL
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: web
      containers:
        - name: web
          image: busybox:1.37
          command: ["sh", "-c", "while true; do sleep 30; done"]
          resources:
            requests:
              cpu: 10m
              memory: 16Mi
            limits:
              cpu: 50m
              memory: 32Mi
EOF

kubectl apply -f "$WK/b7-web.yaml"
kubectl -n lab-12 rollout status deployment/web --timeout=300s
```

```bash
count_on() {
  kubectl -n lab-12 get pods -l app=web \
    --field-selector "spec.nodeName=$1,status.phase=Running" --no-headers 2>/dev/null | wc -l
}

W1_0="$(count_on "$W1")"; W2_0="$(count_on "$W2")"
echo "web tren $W1=$W1_0 | tren $W2=$W2_0 | tong can co=$REPL"
kubectl -n lab-12 get pods -o wide | tee "$EV/b7-1-truoc.txt"

test "$W1_0" -gt 0 && test "$W2_0" -gt 0 && test $(( W1_0 + W2_0 )) -eq "$REPL" \
  && echo 'PASS: workload trai deu tren ca hai worker truoc khi bao tri'
```

**Ý nghĩa:** `whenUnsatisfiable: ScheduleAnyway` là lựa chọn có chủ đích, không phải tiện tay.
`DoNotSchedule` sẽ giữ Pod ở `Pending` khi một worker biến mất — đúng như bạn đã thấy ở
[Lab 7a B5](LAB-7A-LAP-LICH-VA-EVICTION.md#b5-ràng-buộc-phân-bố-pod-theo-topology) — và như vậy sẽ
không quan sát được việc **dịch chuyển**. Với `ScheduleAnyway`, ràng buộc chỉ là một tín hiệu chấm
điểm: lúc đủ node thì Pod trải đều, lúc thiếu node thì chúng dồn hết về nơi còn lại.

Không dùng PersistentVolumeClaim trong lab này: provisioner của baseline gắn dữ liệu vào **thư mục
trên chính node**, nên một Pod có PVC sẽ không lập lịch được sang worker kia — nó biến bài học về
dịch chuyển workload thành bài học về ràng buộc lưu trữ.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B7.2. `cordon` chỉ đánh dấu, không đuổi ai

```bash
kubectl cordon "$W2"
kubectl get nodes | tee "$EV/b7-2-nodes-cordon.txt"

UNSCH="$(kubectl get node "$W2" -o jsonpath='{.spec.unschedulable}')"
kubectl get node "$W2" -o jsonpath='{range .spec.taints[*]}{.key}{"="}{.effect}{"\n"}{end}' \
  | tee "$EV/b7-2-taint-cordon.txt"
W2_1="$(count_on "$W2")"
echo "unschedulable=$UNSCH | web tren worker2 truoc cordon=$W2_0, sau cordon=$W2_1"

test "$UNSCH" = 'true' && echo 'PASS: cordon dat .spec.unschedulable=true'
grep -q '^node.kubernetes.io/unschedulable=NoSchedule$' "$EV/b7-2-taint-cordon.txt" \
  && echo 'PASS: cordon them taint node.kubernetes.io/unschedulable:NoSchedule'
grep -q 'SchedulingDisabled' "$EV/b7-2-nodes-cordon.txt" \
  && echo 'PASS: kubectl get nodes hien SchedulingDisabled'
test "$W2_1" -eq "$W2_0" \
  && echo 'PASS: cordon KHONG duoi Pod nao — so Pod web tren worker2 khong doi'
```

**Ý nghĩa:** đây là ranh giới đầu tiên phải nhớ. `cordon` **chỉ** đặt `.spec.unschedulable=true` và
gắn taint `node.kubernetes.io/unschedulable:NoSchedule`. Taint effect `NoSchedule` **không đuổi Pod
đang chạy** — bạn đã chứng minh điều đó ở
[Lab 7a B4](LAB-7A-LAP-LICH-VA-EVICTION.md#b4-taint-và-toleration--mặt-đối-ngẫu-của-affinity), và
đây là hệ quả trực tiếp.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B7.3. Node đã cordon không nhận Pod mới

```bash
cat > "$WK/b7-thu-cordon.yaml" <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: thu-cordon
  namespace: lab-12
spec:
  nodeSelector:
    kubernetes.io/hostname: $W2
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "while true; do sleep 30; done"]
EOF

kubectl apply -f "$WK/b7-thu-cordon.yaml"

PH=''
for i in $(seq 1 24); do
  PH="$(kubectl -n lab-12 get pod thu-cordon -o jsonpath='{.status.phase}')"
  kubectl -n lab-12 describe pod thu-cordon > "$EV/b7-3-thu-cordon.txt" 2>&1
  if [ "$PH" = 'Pending' ] && grep -qi 'unschedulable' "$EV/b7-3-thu-cordon.txt"; then break; fi
  sleep 5
done
sed -n '/Events:/,$p' "$EV/b7-3-thu-cordon.txt"
echo "phase = $PH"

test "$PH" = 'Pending' && grep -qi 'unschedulable' "$EV/b7-3-thu-cordon.txt" \
  && echo 'PASS: Pod moi khong len duoc node da cordon, va scheduler noi ro ly do'

kubectl -n lab-12 delete pod thu-cordon --wait=true --timeout=120s
test "$(kubectl -n lab-12 get pod thu-cordon --ignore-not-found -o name | wc -l)" -eq 0 \
  && echo 'PASS: da xoa Pod thu — khong con Pod tran nao truoc khi drain'
```

**Ý nghĩa:** `nodeSelector` ghim Pod vào đúng worker2, nên nếu node đó nhận Pod thì Pod chạy, còn
không thì Pod nằm `Pending` với lý do do scheduler ghi vào `Events`. Đây là mặt còn lại của cordon:
**không đuổi ai, nhưng cũng không nhận thêm ai.**

Xóa Pod này trước khi drain là bắt buộc: nó là Pod **trần**, không có controller quản lý, nên
`kubectl drain` sẽ đòi `--force` — mà `--force` thì xóa vĩnh viễn, và [mục 2](#2-quy-ước-và-an-toàn)
cấm dùng cờ đó trong lab này.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B7.4. `drain` — đánh dấu **và** trục xuất

Chạy `drain` trần trước, để chính lệnh nói cho bạn biết nó thiếu gì:

```bash
kubectl drain "$W2" > "$EV/b7-4-drain-tran.txt" 2>&1 || true
cat "$EV/b7-4-drain-tran.txt"

grep -q 'ignore-daemonsets' "$EV/b7-4-drain-tran.txt" \
  && echo 'PASS: drain tran bi tu choi va noi ro no can --ignore-daemonsets'
```

Đọc chính lời giải thích của lệnh:

```bash
kubectl drain --help > "$EV/b7-4-drain-help.txt" 2>&1
sed -n '/--ignore-daemonsets/,+4p' "$EV/b7-4-drain-help.txt"
sed -n '/--delete-emptydir-data/,+4p' "$EV/b7-4-drain-help.txt"

grep -q -- '--ignore-daemonsets' "$EV/b7-4-drain-help.txt" \
  && grep -q -- '--delete-emptydir-data' "$EV/b7-4-drain-help.txt" \
  && echo 'PASS: doc duoc mo ta cua hai co tu chinh kubectl drain'
```

**Vì sao `--ignore-daemonsets` gần như luôn cần:** DaemonSet controller có nhiệm vụ giữ đúng một
Pod trên **mỗi** node phù hợp, và nó **không quan tâm node có unschedulable hay không**. Trục xuất
một Pod DaemonSet chỉ khiến controller tạo lại ngay Pod đó trên chính node vừa drain, nên `drain`
sẽ không bao giờ kết thúc. `kubectl drain` xử lý bằng cách từ chối chạy cho tới khi bạn nói rõ là
mình chấp nhận **bỏ qua** nhóm Pod đó. Trên mọi cluster thật đều có ít nhất CNI và kube-proxy chạy
dạng DaemonSet, nên trên thực tế cờ này gần như luôn phải có.

Bây giờ chạy thật:

```bash
if kubectl drain "$W2" --ignore-daemonsets --delete-emptydir-data --timeout=300s \
     > "$EV/b7-4-drain.txt" 2>&1; then
  echo 'PASS: drain hoan tat'
else
  echo 'FAIL: drain khong hoan tat — doc file evidence va muc 4'
fi
cat "$EV/b7-4-drain.txt"
```

```bash
kubectl -n lab-12 rollout status deployment/web --timeout=300s
W1_2="$(count_on "$W1")"; W2_2="$(count_on "$W2")"
echo "web tren $W1=$W1_2 | tren $W2=$W2_2"
kubectl -n lab-12 get pods -o wide | tee "$EV/b7-4-sau-drain.txt"

test "$W2_2" -eq 0 && test "$W1_2" -eq "$REPL" \
  && echo 'PASS: toan bo Pod web da chuyen sang worker con lai'
```

```bash
ALL_W2_2="$(kubectl get pods -A \
  --field-selector "spec.nodeName=$W2,status.phase=Running" --no-headers 2>/dev/null | wc -l)"
DS_W2_2="$(kubectl get pods -A --field-selector "spec.nodeName=$W2,status.phase=Running" \
  -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].kind}{"\n"}{end}' \
  | grep -c '^DaemonSet$')"
echo "worker2 sau drain: tong Pod Running=$ALL_W2_2 | do DaemonSet quan ly=$DS_W2_2"
kubectl get pods -A --field-selector "spec.nodeName=$W2,status.phase=Running" -o wide \
  | tee "$EV/b7-4-pods-worker2.txt"

test "$ALL_W2_2" -eq "$DS_W2_2" && test "$DS_W2_2" -gt 0 \
  && echo 'PASS: worker2 chi con Pod DaemonSet — dung dinh nghia "Node rong" cua bai 171'
```

**Ý nghĩa:** đây là ranh giới thứ hai và là điểm mấu chốt của cả lab. `drain` làm **hai việc**:
`cordon` node, rồi **trục xuất** mọi Pod mà nó được phép trục xuất. Cơ chế nằm dưới không phải
`kubectl delete pod`: `drain` gọi **Eviction API**, đúng subresource `pods/eviction` mà bạn đã gọi
tay ở [Lab 7a B7](LAB-7A-LAP-LICH-VA-EVICTION.md#b7-eviction-khởi-phát-qua-api). Hệ quả trực tiếp:
`drain` **tôn trọng PodDisruptionBudget**, và nó **thử lại** các yêu cầu bị từ chối cho tới khi
xong hoặc hết `--timeout`. Nếu cluster có một PDB quá chặt, `drain` sẽ treo — đó là lý do gate mở
đầu của lab bắt `kubectl get pdb -A` phải trả `No resources found`.

Bảng phân biệt phải thuộc:

| | `cordon` | `drain` |
| --- | --- | --- |
| Đặt `.spec.unschedulable` | có | có |
| Thêm taint `node.kubernetes.io/unschedulable:NoSchedule` | có | có |
| Trục xuất Pod đang chạy | **không** | **có**, qua Eviction API |
| Tôn trọng PodDisruptionBudget | không liên quan | **có** |
| Đụng Pod DaemonSet | không | chỉ khi có `--ignore-daemonsets` để **bỏ qua** chúng |

Và trạng thái worker2 lúc này khớp đúng định nghĩa của bài
[171](../171-node-autoscaling-vi.md): một Node được coi là **rỗng** nếu chỉ còn Pod DaemonSet và
static Pod chạy trên đó. B8 dùng lại quan sát này.

**PASS:** năm dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B7.5. Tắt máy và quan sát từ phía control plane

Ghi lại Node Lease **trước** khi tắt, để lát nữa so:

```bash
kubectl -n kube-node-lease get lease "$W2" \
  -o custom-columns='NAME:.metadata.name,HOLDER:.spec.holderIdentity,RENEW:.spec.renewTime,DUR:.spec.leaseDurationSeconds' \
  | tee "$EV/b7-5-lease-truoc.txt"

test -s "$EV/b7-5-lease-truoc.txt" && echo 'PASS: da ghi Node Lease cua worker2 truoc khi tat may'
```

Tắt **đúng worker2**, từ master:

```bash
ssh "$W2" 'sudo shutdown -h now' || true
```

**Chạy trên máy host**, PowerShell — dùng lại `$vmrun` và `$vmx` đã đặt ở mục 2:

```powershell
$w2 = $vmx | Where-Object { $_ -like '*worker2*' }
$running = & $vmrun -T ws list
if ($running -notcontains $w2) { "PASS: worker2 da Powered off" }
else { "CHUA TAT: doi them roi chay lai lenh nay" }
```

**PASS:** dòng `PASS: worker2 da Powered off` xuất hiện. Nếu sau vài lần chạy lại vẫn chưa tắt, xem
[mục 4](#4-troubleshooting-của-lab-này); **không** dùng `vmrun stop … hard` trừ khi mục 4 chỉ dẫn.

Quay lại master, quan sát control plane phản ứng:

```bash
ST=''
for i in $(seq 1 180); do
  ST="$(kubectl get node "$W2" \
    -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
  test "$ST" != 'True' && break
  sleep 5
done
echo "Ready condition cua $W2 = $ST"
kubectl get nodes | tee "$EV/b7-5-nodes-down.txt"

test "$ST" = 'Unknown' \
  && echo 'PASS: node controller da doi Ready thanh Unknown khi mat heartbeat'
```

```bash
kubectl get node "$W2" -o jsonpath='{range .spec.taints[*]}{.key}{"="}{.effect}{"\n"}{end}' \
  | tee "$EV/b7-5-taint-down.txt"

grep -q 'node.kubernetes.io/unreachable' "$EV/b7-5-taint-down.txt" \
  && echo 'PASS: node mang them taint node.kubernetes.io/unreachable'
grep -q 'node.kubernetes.io/unschedulable' "$EV/b7-5-taint-down.txt" \
  && echo 'PASS: taint cordon van con — tat may khong go no'
```

```bash
R1="$(kubectl -n kube-node-lease get lease "$W2" -o jsonpath='{.spec.renewTime}')"
sleep 30
R2="$(kubectl -n kube-node-lease get lease "$W2" -o jsonpath='{.spec.renewTime}')"
echo "Node Lease renewTime: $R1  ->  $R2"

test -n "$R1" && test "$R1" = "$R2" \
  && echo 'PASS: Node Lease dung yen — khong con kubelet nao gia han no'
```

```bash
STILL="$(kubectl get pods -A --field-selector "spec.nodeName=$W2" --no-headers 2>/dev/null | wc -l)"
TERM="$(kubectl get pods -A --field-selector "spec.nodeName=$W2" \
  -o jsonpath='{range .items[*]}{.metadata.deletionTimestamp}{"\n"}{end}' | grep -c .)"
echo "Pod con gan voi worker2 trong API=$STILL | trong do dang bi xoa=$TERM"
kubectl get pods -A --field-selector "spec.nodeName=$W2" -o wide | tee "$EV/b7-5-pods-down.txt"

test "$STILL" -gt 0 \
  && echo 'PASS: Pod DaemonSet van con trong API du node da tat — API server khong xoa ho'
test "$TERM" -eq 0 \
  && echo 'PASS: khong Pod nao ket Terminating — vi da drain truoc khi tat may'
```

**Ý nghĩa:** ba tín hiệu bạn vừa thấy đến từ ba nơi khác nhau.

- **Condition `Ready` thành `Unknown`** là việc của **node controller**: khi node trở nên không thể
  truy cập được, nó cập nhật condition trong `.status` của Node. Bài [23](../23-nodes-vi.md) mô tả
  chu kỳ kiểm tra và khoảng chờ trước yêu cầu trục xuất đầu tiên là **giá trị mặc định, phụ thuộc
  cấu hình** — chúng đọc được từ các cờ `--node-monitor-period` và các cờ liên quan trên
  `kube-controller-manager`. Đừng học thuộc con số; hãy đọc cấu hình thật khi cần.
- **Node Lease dừng lại** là mặt còn lại của cùng một sự việc, nhìn từ phía kubelet: kubelet gia hạn
  Lease trùng tên Node trong `kube-node-lease`, và khi kubelet chết thì `renewTime` đứng im. Đây
  chính là cơ chế bạn thấy còn sống ở B4.2, giờ thấy nó dừng.
- **Taint `node.kubernetes.io/unreachable`** được control plane tự gắn theo condition. Pod thường
  mang toleration tự thêm cho taint này với một `tolerationSeconds` hữu hạn nên sẽ bị đuổi sau đó;
  Pod DaemonSet thì tolerate **không giới hạn**, nên chúng ở lại — đúng những gì bạn đang thấy.

Và điều **không** xảy ra cũng quan trọng không kém: không Pod nào kẹt `Terminating`. Bài
[169](../169-node-shutdown-vi.md) mô tả kịch bản ngược lại — node tắt mà kubelet không kịp biết,
Pod của StatefulSet kẹt `Terminating` vĩnh viễn và VolumeAttachment không được xóa nên volume không
attach sang node khác được, buộc phải gắn tay taint `node.kubernetes.io/out-of-service`. Bạn tránh
được toàn bộ chuỗi đó **chỉ bằng cách drain trước**. Đó là toàn bộ lý do B6 phải chạy trước B7.

**PASS:** bảy dòng `PASS:` của bước này xuất hiện.

### B7.6. Bật lại máy và chờ node trở lại `Ready`

**Chạy trên máy host**, PowerShell:

```powershell
$w2 = $vmx | Where-Object { $_ -like '*worker2*' }
& $vmrun -T ws start $w2
Start-Sleep -Seconds 5
$running = & $vmrun -T ws list
if ($running -contains $w2) { "PASS: worker2 dang chay" } else { "FAIL: VM chua len" }
```

**PASS:** dòng `PASS: worker2 dang chay` xuất hiện. Nếu bạn quen bật VM bằng giao diện VMware
Workstation thì làm cách đó cũng được; điều bắt buộc là **bật lại đúng một VM là worker2**.

Quay lại master:

```bash
kubectl wait --for=condition=Ready node/"$W2" --timeout=600s
kubectl get nodes -o wide | tee "$EV/b7-6-nodes-up.txt"

RDY="$(kubectl get node "$W2" \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
UNSCH2="$(kubectl get node "$W2" -o jsonpath='{.spec.unschedulable}')"
echo "Ready=$RDY | unschedulable=$UNSCH2"

test "$RDY" = 'True' && echo 'PASS: worker2 da tro lai Ready'
test "$UNSCH2" = 'true' \
  && echo 'PASS: node VAN unschedulable — reboot khong tu go cordon'
grep -q 'SchedulingDisabled' "$EV/b7-6-nodes-up.txt" && echo 'PASS: van hien SchedulingDisabled'
```

```bash
DS_UP=0
for i in $(seq 1 60); do
  DS_UP="$(kubectl get pods -A --field-selector "spec.nodeName=$W2,status.phase=Running" \
    -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].kind}{"\n"}{end}' \
    | grep -c '^DaemonSet$')"
  test "$DS_UP" -ge "$DS_W2_0" && break
  sleep 5
done
echo "Pod DaemonSet dang chay tren worker2: $DS_UP (truoc lab: $DS_W2_0)"

test "$DS_UP" -ge "$DS_W2_0" \
  && echo 'PASS: du Pod DaemonSet da chay lai tren worker2 sau khi boot'
```

**Ý nghĩa:** hai kết luận từ bước này. Thứ nhất, **reboot không gỡ cordon**: `.spec.unschedulable`
nằm trong Node object trên API server, không nằm trên đĩa của node, nên nó sống sót qua việc tắt
máy. Người vận hành quên bước cuối sẽ có một node `Ready` mà mãi không nhận Pod nào — một sự cố
kinh điển. Thứ hai, Pod DaemonSet quay lại **mà không cần bạn làm gì**: DaemonSet controller đặt Pod
lên node bất kể node có unschedulable hay không, đúng lý do B7.4 cần `--ignore-daemonsets`.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B7.7. `uncordon` và workload quay về

```bash
kubectl uncordon "$W2"
kubectl get nodes | tee "$EV/b7-7-nodes-uncordon.txt"

UNSCH3="$(kubectl get node "$W2" -o jsonpath='{.spec.unschedulable}')"
TAINT_N="$(kubectl get node "$W2" -o jsonpath='{range .spec.taints[*]}{.key}{"\n"}{end}' \
  | grep -c 'node.kubernetes.io/unschedulable')"
echo "unschedulable=[$UNSCH3] | taint unschedulable con lai=$TAINT_N"

test -z "$UNSCH3" && echo 'PASS: uncordon da go .spec.unschedulable'
test "$TAINT_N" -eq 0 && echo 'PASS: taint node.kubernetes.io/unschedulable da bien mat'
grep -q 'SchedulingDisabled' "$EV/b7-7-nodes-uncordon.txt" \
  || echo 'PASS: khong node nao con SchedulingDisabled'
```

```bash
kubectl -n lab-12 rollout restart deployment/web
kubectl -n lab-12 rollout status deployment/web --timeout=300s

W1_3="$(count_on "$W1")"; W2_3="$(count_on "$W2")"
echo "web tren $W1=$W1_3 | tren $W2=$W2_3 | tong can co=$REPL"
kubectl -n lab-12 get pods -o wide | tee "$EV/b7-7-sau-uncordon.txt"

test "$W2_3" -gt 0 && test $(( W1_3 + W2_3 )) -eq "$REPL" \
  && echo 'PASS: sau uncordon workload quay lai worker2'
```

**Ý nghĩa:** `uncordon` gỡ dấu, nhưng **không tự mang Pod về**. Kubernetes không chủ động cân bằng
lại Pod đang chạy: Pod đã lập lịch thì ở yên nơi nó đang ở. Thứ đưa workload quay lại là một sự
kiện tạo Pod mới — ở đây là `rollout restart`, ngoài đời thường là lần deploy tiếp theo, một lần
scale, hoặc một Pod chết đi và được tạo lại. Ràng buộc `topologySpreadConstraints` của B7.1 khi đó
mới có cơ hội chấm điểm và kéo Pod về worker2.

Nếu bạn muốn Pod tự phân bố lại mà không đợi sự kiện nào, đó là việc của một
[Descheduler](https://github.com/kubernetes-sigs/descheduler) — add-on riêng mà bài
[171](../171-node-autoscaling-vi.md) nhắc tới ở mục *Các thành phần liên quan*, và cluster này
không có.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B7.8. Ghi lại toàn bộ vòng bảo trì

```bash
{
  echo "=== $(date -Is) — vong bao tri $W2 ==="
  echo "buoc 1 truoc bao tri  : web tren $W1=$W1_0, tren $W2=$W2_0"
  echo "buoc 2 sau cordon     : web tren $W2=$W2_1 (khong doi), unschedulable=true"
  echo "buoc 3 sau drain      : web tren $W1=$W1_2, tren $W2=$W2_2; con lai tren node=$ALL_W2_2"
  echo "                        trong do DaemonSet=$DS_W2_2 -> node RONG"
  echo "buoc 4 khi tat may    : Ready=$ST, Node Lease dung tai $R1"
  echo "buoc 5 sau khi bat lai: Ready=$RDY, unschedulable=$UNSCH2, DaemonSet=$DS_UP"
  echo "buoc 6 sau uncordon   : web tren $W1=$W1_3, tren $W2=$W2_3"
} | tee "$EV/b7-8-tong-ket.txt"

test "$(wc -l < "$EV/b7-8-tong-ket.txt")" -ge 7 \
  && echo 'PASS: da ghi tong ket sau buoc cua vong bao tri'
```

**PASS:** dòng `PASS:` của bước này xuất hiện, và bảng tổng kết đọc thành một câu chuyện liền mạch.

---

## B8. Tự động mở rộng node — vì sao cluster này không có

**Mục đích:** bài [171](../171-node-autoscaling-vi.md) **không kiểm chứng được** trên cluster lab,
và đó chính là bài học đầu tiên. Ở đây bạn chứng minh điều đó bằng dữ kiện thay vì tin lời, và đọc
lại quan sát của B7.4 dưới đúng thuật ngữ của bài.

```bash
AS_N="$(kubectl get deployment,statefulset,daemonset -A --no-headers 2>/dev/null \
  | grep -ciE 'cluster-autoscaler|karpenter|descheduler')"
PID_N2="$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.providerID}{"\n"}{end}' \
  | grep -c .)"
NODE_N1="$(kubectl get nodes --no-headers | wc -l)"
CP_FLAG0="$(sudo grep -c 'cloud-provider' /etc/kubernetes/manifests/kube-controller-manager.yaml)"
echo "autoscaler/descheduler=$AS_N | node co providerID=$PID_N2"
echo "so node=$NODE_N1 (dau lab: $NODE_N0) | --cloud-provider tren KCM=$CP_FLAG0"

test "$AS_N" -eq 0 && echo 'PASS: khong autoscaler nao dang chay tren cluster'
test "$PID_N2" -eq 0 && test "$CP_FLAG0" -eq 0 \
  && echo 'PASS: khong co cloud provider nao dung sau Node — khong co API de autoscaler goi'
test "$NODE_N1" -eq "$NODE_N0" \
  && echo 'PASS: so node khong doi qua ca vong bao tri — ba VM co dinh'
```

**Ý nghĩa:** autoscaler cấp phát Node bằng cách **tạo và xóa tài nguyên của nhà cung cấp đám mây**
đứng sau Node — phổ biến nhất là máy ảo — nên chúng cần **tích hợp tường minh với từng cloud
provider**. Ba VM bạn tự tạo trong VMware Workstation không có API nào để autoscaler gọi. Đây là
giới hạn về **bản chất**, không phải thiếu một dòng cấu hình.

Điều vẫn phải nắm dù không chạy được:

- Hai thao tác đối xứng: **cấp phát** (provisioning, trước gọi là *scale-up*) khi có Pod không lập
  lịch được, và **hợp nhất** (consolidation, trước gọi là *scale-down*) khi Node đang dùng dưới mức.
- **Cả hai chỉ xét resource request của Pod, không xét mức sử dụng thực tế.** Request quá thấp thì
  cấp Node mới cũng không cứu được Pod; request quá cao thì chặn nhầm việc hợp nhất Node.
- Node **rỗng** = chỉ còn Pod DaemonSet và static Pod. **Bạn vừa tạo ra đúng trạng thái đó ở
  B7.4** — bằng tay, bằng `drain`. Loại bỏ Node rỗng đơn giản; loại bỏ Node **không rỗng** gây gián
  đoạn vì Pod bị terminate và phải được tạo lại.
- Kết hợp với autoscaling workload: **theo chiều ngang** tạo thêm Pod theo tải rồi autoscaling Node
  cấp Node để chứa chúng — thứ tự phản ứng luôn là workload trước, Node theo sau; **theo chiều dọc**
  sửa lại request cho đúng, nhưng **không dùng cho Pod DaemonSet** vì autoscaler phải dự đoán được
  Pod DaemonSet chiếm bao nhiêu trên một Node mới.

Phần autoscaling workload theo chiều ngang là nội dung của **Lab 11b**, nơi
[nợ #1](README.md#5-sổ-nợ-lab) được trả. Lab 12 **không** tạo HPA nào.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

---

## B9. Truy cập cluster qua Kubernetes API

**Mục đích:** bài [189](../189-administer-cluster-vi.md) là **trang mục lục** của cả nhánh
*Tasks → Administer a Cluster*; bài [190](../190-access-cluster-api-vi.md) là một trang trong đó.
B9.1 dùng trang mục lục đúng công dụng của nó, ba mục còn lại đi ba đường khác nhau tới cùng một
API server.

### B9.1. Ba mục của trang mục lục, ba hiện vật trong cluster

```bash
API_V="$(kubectl get --raw /api | grep -c '"versions"')"
EVICT="$(kubectl get --raw /api/v1 | grep -c 'pods/eviction')"
SC_DEF2="$(kubectl get storageclass \
  -o jsonpath='{range .items[*]}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}' \
  | grep -c '^true$')"
echo "/api tra versions=$API_V | subresource pods/eviction=$EVICT | StorageClass mac dinh=$SC_DEF2"

test "$API_V" -eq 1 \
  && echo 'PASS: muc "Truy cap cluster bang Kubernetes API" -> endpoint /api co that'
test "$EVICT" -ge 1 \
  && echo 'PASS: muc "Drain mot node an toan" -> subresource pods/eviction, co che duoi drain o B7'
test "$SC_DEF2" -eq 1 \
  && echo 'PASS: muc "Doi StorageClass mac dinh" -> annotation is-default-class dang o dung mot class'
```

**Ý nghĩa:** trang mục lục không dạy gì, nhưng nó rút ngắn quãng đường từ một câu hỏi vận hành tới
đúng tài liệu. Ba mục bạn vừa nối vào cluster là ba mục có liên quan trực tiếp tới lab này: một cái
là B9 tiếp theo, một cái là cơ chế nằm dưới `drain` của B7, một cái là hạ tầng Lab 6a để lại. Gần
50 mục còn lại trải khắp [Phần II](../00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster) và có chỗ
đứng riêng ở giai đoạn 16–27.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B9.2. Đường thứ nhất — `kubectl` và kubeconfig

```bash
kubectl config view | tee "$EV/b9-config-view.txt"

APISERVER="$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
CTX="$(kubectl config current-context)"
echo "context=$CTX | server=$APISERVER"

grep -q 'DATA+OMITTED\|REDACTED' "$EV/b9-config-view.txt" \
  && echo 'PASS: kubectl config view che du lieu nhay cam — dung de dan file nay di dau'
case "$APISERVER" in
  https://*) echo 'PASS: kubectl biet vi tri cluster, va no la mot endpoint HTTPS' ;;
  *)         echo "FAIL: server khong phai https ($APISERVER)" ;;
esac
```

**Ý nghĩa:** để truy cập một cluster bạn cần **vị trí** và **thông tin xác thực**. Với `kubectl`,
cả hai nằm trong kubeconfig và bạn không phải nghĩ tới chúng. Lưu ý `kubectl config view` **che**
phần dữ liệu nhạy cảm; muốn thấy nguyên bản phải thêm `--raw` — và lúc đó file chứa client
certificate của admin, đừng dán nó vào đâu.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B9.3. Đường thứ hai — `kubectl proxy`

```bash
kubectl proxy --port=8080 > "$WK/proxy.log" 2>&1 &
PROXY_PID=$!
echo "kubectl proxy PID=$PROXY_PID"

OK_PROXY=0
for i in $(seq 1 20); do
  if curl -s --max-time 3 http://127.0.0.1:8080/api/ > "$EV/b9-proxy-api.json" 2>/dev/null; then
    grep -q '"versions"' "$EV/b9-proxy-api.json" && OK_PROXY=1 && break
  fi
  sleep 1
done
cat "$EV/b9-proxy-api.json"

test "$OK_PROXY" -eq 1 \
  && echo 'PASS: curl khong kem chung thuc gi van goi duoc /api qua kubectl proxy'

kill "$PROXY_PID" 2>/dev/null
sleep 2
curl -s --max-time 3 http://127.0.0.1:8080/api/ > /dev/null 2>&1 \
  || echo 'PASS: da tat proxy — cong 8080 khong con phuc vu'
```

**Ý nghĩa:** đây là cách **được khuyến nghị** trong bài. `kubectl proxy` chạy như một reverse proxy:
nó tự định vị API server, tự xác thực, và **xác minh danh tính API server bằng certificate tự ký**
đã lưu sẵn — nên không thể xảy ra tấn công xen giữa. Cái giá phải trả là proxy mở một cổng trên
localhost mà **bất kỳ tiến trình nào trên máy đó cũng gọi được với quyền của bạn**; đó là lý do lab
tắt nó ngay sau khi dùng xong.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B9.4. Đường thứ ba — HTTP client trực tiếp, có xác thực

```bash
kubectl config view --raw --minify \
  -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
  | base64 -d > "$WK/ca.crt"
chmod 600 "$WK/ca.crt"
head -1 "$WK/ca.crt"

TOKEN="$(kubectl create token default -n lab-12)"

test -s "$WK/ca.crt" && grep -q 'BEGIN CERTIFICATE' "$WK/ca.crt" \
  && echo 'PASS: lay duoc CA cua cluster tu kubeconfig'
test -n "$TOKEN" \
  && echo 'PASS: tao duoc token cho ServiceAccount default cua namespace lab-12'
```

```bash
curl -s --cacert "$WK/ca.crt" -H "Authorization: Bearer $TOKEN" \
  "$APISERVER/api" > "$EV/b9-direct-api.json"
cat "$EV/b9-direct-api.json"

grep -q '"versions"' "$EV/b9-direct-api.json" \
  && echo 'PASS: HTTP client truc tiep goi duoc /api, KHONG can --insecure'
```

```bash
CODE="$(curl -s -o /dev/null -w '%{http_code}' --cacert "$WK/ca.crt" \
  -H "Authorization: Bearer $TOKEN" "$APISERVER/api/v1/namespaces/lab-12/pods")"
echo "HTTP code khi liet ke Pod bang token cua ServiceAccount default = $CODE"

test "$CODE" = '403' \
  && echo 'PASS: xac thuc thanh cong nhung phan quyen tu choi — hai buoc khac nhau'
```

**Ý nghĩa:** bài đưa ví dụ dùng `--insecure`, và **nói thẳng rằng điều đó khiến client dễ bị tấn
công xen giữa**. Lab làm đúng phần bài gợi ý ngay sau đó: vì certificate của cluster thường là tự
ký, cần **cấu hình đặc biệt để HTTP client dùng được root certificate** — ở đây là trích CA từ
chính kubeconfig rồi truyền qua `--cacert`.

Hai mã HTTP ở trên nói hai chuyện khác nhau. `/api` trả `200` vì mọi client **đã xác thực** đều
được phép khám phá API. `/api/v1/namespaces/lab-12/pods` trả `403` vì ServiceAccount `default` chưa
có RoleBinding nào — token hợp lệ, danh tính hợp lệ, nhưng **không có quyền**. Đây đúng ranh giới
authn/authz bạn đã dựng ở [Lab 9a](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md).

Xóa dấu vết ngay, đừng để token và CA nằm lại:

```bash
unset TOKEN
rm -f "$WK/ca.crt"
test ! -e "$WK/ca.crt" && echo 'PASS: da xoa ban sao CA; token chi ton tai trong bien shell'
```

**PASS:** năm dòng `PASS:` của bước này xuất hiện.

---

## B10. Leader migration của controller manager — phần đọc được

**Mục đích:** bài [198](../198-controller-manager-leader-migration-vi.md) mô tả một quy trình di
trú **không bao giờ chạy** trên cluster này. Thay vì bỏ trống, bạn chứng minh bằng bốn dữ kiện tại
sao nó không áp dụng, và đọc một chi tiết RBAC giải thích vì sao bước đầu tiên của quy trình lại là
vá role.

### B10.1. Bốn dữ kiện loại trừ

```bash
CP_FLAG="$(sudo grep -c 'cloud-provider' /etc/kubernetes/manifests/kube-controller-manager.yaml)"
MIG_FLAG="$(sudo grep -c 'enable-leader-migration' \
  /etc/kubernetes/manifests/kube-controller-manager.yaml)"
CCM_N="$(kubectl get deployment,daemonset,pod -A --no-headers 2>/dev/null \
  | grep -ci 'cloud-controller-manager')"
PID_N3="$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.providerID}{"\n"}{end}' \
  | grep -c .)"
CP_N2="$(kubectl get nodes -l 'node-role.kubernetes.io/control-plane' --no-headers | wc -l)"
echo "--cloud-provider tren KCM=$CP_FLAG | --enable-leader-migration=$MIG_FLAG"
echo "cloud-controller-manager dang chay=$CCM_N | node co providerID=$PID_N3 | so control plane=$CP_N2"

test "$CP_FLAG" -eq 0 && test "$CCM_N" -eq 0 \
  && echo 'PASS: khong cloud provider in-tree, khong cloud-controller-manager'
test "$PID_N3" -eq 0 \
  && echo 'PASS: khong Node nao gan voi tai nguyen cua mot nha cung cap cloud'
test "$CP_N2" -eq 1 \
  && echo 'PASS: control plane MOT node — bai noi ro truong hop nay khong can Leader Migration'
test "$MIG_FLAG" -eq 0 \
  && echo 'PASS: --enable-leader-migration khong duoc dat o dau ca'
```

**Ý nghĩa:** Leader Migration tồn tại để chuyển ba controller đặc thù cloud — `route`, `service`,
`cloud-node-lifecycle` — từ `kube-controller-manager` sang `cloud-controller-manager` **trong lúc
nâng cấp cuốn chiếu một control plane HA**, sao cho không controller nào chạy đồng thời ở hai nơi.
Bài nêu thẳng khi nào **không** cần nó: control plane một node, hoặc chấp nhận controller manager
không sẵn sàng trong lúc nâng cấp. Cluster của bạn rơi vào cả hai.

### B10.2. Lease di trú không tồn tại, và role chưa cho phép nó tồn tại

```bash
kubectl -n kube-system get lease | tee "$EV/b10-lease.txt"
MIG_LEASE="$(grep -c 'cloud-provider-extraction-migration' "$EV/b10-lease.txt")"

kubectl -n kube-system get role 'system::leader-locking-kube-controller-manager' \
  -o jsonpath='{range .rules[*]}{.apiGroups}{" "}{.resources}{" "}{.resourceNames}{" "}{.verbs}{"\n"}{end}' \
  | tee "$EV/b10-role-kcm.txt"
CCM_ROLE="$(kubectl -n kube-system get role \
  'system::leader-locking-cloud-controller-manager' --ignore-not-found -o name | wc -l)"
echo "Lease di tru=$MIG_LEASE | role leader-locking cho CCM=$CCM_ROLE"

test "$MIG_LEASE" -eq 0 \
  && echo 'PASS: khong co Lease cloud-provider-extraction-migration'
grep -q 'cloud-provider-extraction-migration' "$EV/b10-role-kcm.txt" \
  || echo 'PASS: role chi cho truy cap Lease chinh cua no, chua he duoc va cho Lease di tru'
test "$CCM_ROLE" -eq 0 \
  && echo 'PASS: role leader-locking cua cloud-controller-manager khong ton tai o day'
```

**Ý nghĩa:** cơ chế của Leader Migration là hai controller manager chia sẻ **một Lease di trú** —
một resource lock chung, tên ví dụ `cloud-provider-extraction-migration`; manager nào giữ Lease thì
chạy nhóm controller được khai trong `LeaderMigrationConfiguration`. Nhưng **quyền mặc định của
controller manager chỉ cho phép truy cập Lease chính của nó**, đúng như bạn vừa đọc trong
`.rules` — đó là lý do bước đầu tiên của quy trình là `kubectl patch` hai role
`system::leader-locking-*` để thêm `resourceNames: [cloud-provider-extraction-migration]`.

Lab **không** chạy lệnh patch đó: nó sửa RBAC của control plane trên một cluster không có gì để di
trú. Nếu sau này bạn thật sự chạy quy trình này, hai chi tiết đáng nhớ: đặt `component: *` cho cả
hai phía để một file cấu hình dùng chung được, và từ Kubernetes 1.22 có **cấu hình mặc định** — chỉ
cần `--enable-leader-migration`, không cần file.

**PASS:** bốn dòng `PASS:` của B10.1 và ba dòng `PASS:` của B10.2 xuất hiện.

---

## B11. Cleanup và gate cuối

**Mục đích:** xóa mọi thứ lab tạo ra, chứng minh **không node nào còn cordon** và **cấu hình node
không hề bị sửa** dù đã qua một lần tắt máy, rồi chứng minh cluster về đúng `04-metrics-ready`.

### B11.1. Xóa object của bài học

```bash
kubectl delete namespace lab-12 --wait=true --timeout=300s

rm -f "$WK/b7-web.yaml" "$WK/b7-thu-cordon.yaml" "$WK/doc-swap.sh" \
      "$WK/doc-shutdown-prereq.sh" "$WK/proxy.log" "$WK/ca.crt"
rmdir "$WK"
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` bên dưới biến điều
đó thành gate thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/12/` **giữ lại** — đó là bằng chứng.

```bash
NS_LEFT="$(kubectl get namespace lab-12 --ignore-not-found -o name 2>/dev/null | wc -l)"
POD_DEF="$(kubectl get pods -n default --no-headers 2>/dev/null | wc -l)"
echo "namespace lab-12 con=$NS_LEFT | Pod trong default=$POD_DEF"

test "$NS_LEFT" -eq 0 && echo 'PASS: namespace lab-12 da bien mat'
test "$POD_DEF" -eq 0 && echo 'PASS: namespace default khong con Pod nao'
test ! -e "$WK" && echo 'PASS: manifest tam da xoa het'
```

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B11.2. Gate quan trọng nhất: node trở về nguyên trạng

```bash
UNSCH_N="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{.spec.unschedulable}{"\n"}{end}' | grep -c 'true')"
TAINT_N2="$(kubectl get nodes -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}' \
  | grep -cE 'node.kubernetes.io/(unschedulable|unreachable|not-ready)')"
kubectl get nodes -o custom-columns='NODE:.metadata.name,UNSCHEDULABLE:.spec.unschedulable,TAINTS:.spec.taints' \
  | tee "$EV/b11-nodes.txt"
echo "node dang unschedulable=$UNSCH_N | taint bao tri con sot=$TAINT_N2"

test "$UNSCH_N" -eq 0 \
  && echo 'PASS: khong node nao con bi cordon'
test "$TAINT_N2" -eq 0 \
  && echo 'PASS: khong con taint unschedulable/unreachable/not-ready nao'
```

```bash
{
  echo "$MASTER kubelet $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in "$W1" "$W2"; do
    echo "$n kubelet $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
  for f in kube-apiserver kube-scheduler kube-controller-manager; do
    echo "$MASTER $f $(sudo sha256sum /etc/kubernetes/manifests/$f.yaml | awk '{print $1}')"
  done
} | tee "$EV/b11-config-sha.txt"

diff -u "$EV/b0-config-sha.txt" "$EV/b11-config-sha.txt" \
  && echo 'PASS: sau cau hinh khong he doi — ke ca sau mot lan tat va bat lai worker2' \
  || echo 'FAIL: cau hinh da bi sua — xem muc 4'
```

**Ý nghĩa:** gate này nặng hơn bản ở [Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md#b92-gate-quan-trọng-nhất-của-lab-cấu-hình-node-không-đổi)
một bậc, vì lab 12 có một VM đi qua chu kỳ tắt–bật. Nếu checksum của worker2 đổi, hoặc là bạn đã
sửa file, hoặc node boot lên với cấu hình khác — cả hai đều làm cluster lệch khỏi
`04-metrics-ready` và **không được** mang sang lab sau.

**PASS:** ba dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B11.3. Hạ tầng của mốc `04-metrics-ready` còn nguyên

```bash
SC_N="$(kubectl get storageclass --no-headers | wc -l)"
SC_DEF3="$(kubectl get storageclass \
  -o jsonpath='{range .items[*]}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}' \
  | grep -c '^true$')"
IC_N2="$(kubectl get ingressclass --no-headers 2>/dev/null | wc -l)"
MS_AVAIL2="$(kubectl get apiservice v1beta1.metrics.k8s.io \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
PV_N="$(kubectl get pv --no-headers 2>/dev/null | wc -l)"
echo "storageclass=$SC_N (mac dinh=$SC_DEF3) | ingressclass=$IC_N2 | metrics=$MS_AVAIL2 | pv=$PV_N"

test "$SC_DEF3" -eq 1 && test "$IC_N2" -ge 1 && test "$MS_AVAIL2" = 'True' && test "$PV_N" -eq 0 \
  && echo 'PASS: StorageClass mac dinh, ingress controller va metrics-server deu con nguyen'
```

```bash
TOP_N="$(kubectl top node --no-headers 2>/dev/null | wc -l)"
kubectl top node | tee "$EV/b11-top-node.txt"
test "$TOP_N" -eq 3 \
  && echo 'PASS: kubectl top node lai in du ba node — worker2 da bao cao metric tro lai'
```

**Ý nghĩa:** `kubectl top node` in đủ ba dòng là bằng chứng gọn nhất rằng worker2 đã thật sự quay
lại phục vụ: kubelet của nó chạy, metrics-server thu được số liệu từ nó, và APIService còn
`Available`. Nếu chỉ có hai dòng, node vừa boot chưa kịp báo cáo — chờ rồi chạy lại lệnh trước khi
kết luận là hỏng.

Lưu ý: các Pod hạ tầng bị trục xuất ở B7.4 — ingress controller, provisioner, metrics-server nếu
chúng từng nằm trên worker2 — **không tự quay lại** worker2 sau uncordon. Đó là hành vi đúng, không
phải hỏng hóc; gate chỉ yêu cầu chúng `Running`.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B11.4. Gate ngắn A5.5 và kết thúc

```bash
kubectl wait --for=condition=Ready node --all --timeout=300s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

READY_N="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}' \
  | grep -c '^True$')"
echo "so node Ready=True: $READY_N"
test "$READY_N" -eq 3 && echo 'PASS: ca ba node Ready'

{
  echo "=== $(date -Is) — trang thai sau Lab 12 ==="
  echo '--- nodes'; kubectl get nodes -o wide
  echo '--- unschedulable'
  kubectl get nodes -o custom-columns='NODE:.metadata.name,UNSCHEDULABLE:.spec.unschedulable'
  echo '--- pdb'; kubectl get pdb -A 2>&1
  echo '--- top node'; kubectl top node 2>&1
  echo '--- storageclass'; kubectl get storageclass
  echo '--- ingressclass'; kubectl get ingressclass 2>&1
  echo '--- apiservice metrics'; kubectl get apiservice v1beta1.metrics.k8s.io 2>&1
  echo '--- namespaces'; kubectl get namespaces
} | tee "$EV/b11-sau.txt"

diff -u "$EV/b0-truoc.txt" "$EV/b11-sau.txt" > "$EV/b11-diff.txt" 2>&1 || true
grep -c '^[+-]' "$EV/b11-diff.txt"
```

**PASS:** ba node `Ready`; dòng `PASS: readyz ok` xuất hiện; lệnh field selector trả
`No resources found`; CoreDNS đủ replica `READY`; `default` không có Pod; dòng
`PASS: ca ba node Ready` xuất hiện; danh sách namespace không còn `lab-12`.

File `b11-diff.txt` sẽ có vài dòng khác biệt: mốc thời gian, `AGE` của object, số liệu `top node`,
và **cột `NODE` của một số Pod hạ tầng đã đổi chỗ ở B7.4**. Đó là những khác biệt hợp lệ. Khác biệt
**không** hợp lệ là: một node còn `SchedulingDisabled`, một StorageClass biến mất, `metrics` không
còn `Available`, hay `top node` báo lỗi — gặp bất kỳ cái nào thì xử lý theo
[mục 4](#4-troubleshooting-của-lab-này) trước khi sang lab sau.

Cluster đã trở về `04-metrics-ready`. **Lab 12 không tạo snapshot mới** — để ba VM nguyên trạng
đang chạy hoặc tắt tùy bạn; lab sau tự bật máy theo
[A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab).

---

## 3. Checkpoint 12

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Bạn gõ `kubectl cordon lab-k8s-worker2` rồi đi uống nước. Quay lại, node có còn Pod nào
      không? Nếu bây giờ bạn tắt máy, chuyện gì xảy ra với những Pod đó? Kể đúng **hai** việc mà
      `drain` làm còn `cordon` không làm, và nói `drain` gọi cơ chế nào ở bên dưới.
- [ ] `kubectl drain` trần bị từ chối. Câu từ chối nhắc tới cờ nào, và vì sao **gần như mọi
      cluster thật** đều cần cờ đó? Điều gì xảy ra nếu Kubernetes cứ để bạn trục xuất Pod
      DaemonSet mà không cần cờ?
- [ ] Trong lúc drain, một PodDisruptionBudget quá chặt sẽ gây ra chuyện gì cho lệnh của bạn? Vì
      sao câu trả lời đó là hệ quả trực tiếp của việc `drain` không dùng `kubectl delete pod`?
- [ ] Node của bạn boot lại và `kubectl get nodes` báo `Ready`, nhưng suốt buổi chiều không Pod
      mới nào lên đó. Chuyện gì đã xảy ra, thứ giữ trạng thái đó nằm ở đâu — trên đĩa của node hay
      trong API server — và bạn sửa bằng lệnh gì?
- [ ] Bạn `uncordon` xong mà workload vẫn nằm hết ở worker còn lại. Kubernetes hỏng à? Cần chuyện
      gì xảy ra thì Pod mới quay về, và thành phần nào (không có trên cluster này) sinh ra để làm
      việc đó tự động?
- [ ] Feature gate `GracefulNodeShutdown` bật mặc định từ 1.21. Trên `lab-k8s-worker2`, Pod có
      được chấm dứt nhẹ nhàng khi bạn `shutdown -h now` không? Bạn đọc **hai giá trị nào** để trả
      lời, và nếu chúng khác 0 thì với `30s`/`10s` Pod thường có bao nhiêu giây, Pod quan trọng
      bao nhiêu giây, ai bị chấm dứt trước?
- [ ] Kể ba tín hiệu mà control plane phát ra khi worker2 tắt máy, và mỗi tín hiệu do **thành
      phần nào** tạo ra. Vì sao Pod DaemonSet vẫn còn nằm trong API server trong khi Pod thường
      thì không? Vì sao trong bài này **không** Pod nào kẹt `Terminating`, và kịch bản nào mới làm
      chúng kẹt?
- [ ] Cluster lab tắt swap. Nêu ba câu hỏi khác nhau mà `swapon --show`, `free -m` và dòng `swap`
      trong `/etc/fstab` trả lời, và giải thích vì sao câu thứ ba đặc biệt quan trọng với **chính
      lab này**. Nếu bật swap, kubelet có tự động cho workload dùng nó không?
- [ ] Cluster của bạn có ba VM cố định. Chạy được autoscaling Node không? Thiếu chính xác thứ gì —
      không phải "thiếu cấu hình"? Sau khi drain xong, worker2 rơi vào trạng thái nào theo thuật
      ngữ của bài 171, và vì sao ranh giới đó đáng nhớ?
- [ ] Cluster đang chạy ít nhất ba add-on **không** có tên trong danh mục của bài 165. Kể tên
      chúng theo chức năng, và nói kết luận đúng khi tra danh mục mà không thấy công cụ mình cần.
      Nhóm *Đo lường* của danh mục có gì, và cluster của bạn có nó không?
- [ ] `--emulated-version` phải thỏa ràng buộc nào so với `binaryVersion`? Nếu đặt nó về một phiên
      bản cũ, một API đã bị xóa ở phiên bản mới còn dùng được không, và một tính năng chỉ có ở
      phiên bản mới thì sao? Tùy chọn này tách **hai việc gì** ra khỏi nhau khi nâng cấp?
- [ ] **Nợ #7 chưa được trả ở lab này.** Món nợ đó là gì, phát sinh ở đâu, vì sao bài
      [156](../156-certificates-vi.md) không đủ để gạch chủ đề certificate khỏi danh sách, và nó
      được trả ở giai đoạn nào bằng quy trình nào? Trước khi làm giai đoạn đó bạn phải đọc lại
      bài nào?

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng sau, không nhìn tài liệu:

1. **Vòng bảo trì một node, từ đầu tới cuối.** Bắt đầu từ lúc bạn quyết định thay RAM cho
   `lab-k8s-worker2`. Kể đủ sáu chặng: bạn gõ gì, mỗi lệnh đổi **trường nào** trong Node object,
   Pod nào đi đâu ở mỗi chặng và **ai** ra quyết định đó, node còn lại gì trước khi bạn dám tắt
   máy, control plane thấy gì trong lúc node vắng mặt, và hai việc bạn **phải** làm sau khi máy
   lên lại — kèm lý do vì sao quên việc thứ hai lại tạo ra một node `Ready` mà vô dụng. Kết bằng
   câu trả lời cho: nếu bỏ hẳn bước `drain` thì hỏng ở đâu, và vì sao câu trả lời đó **phụ thuộc
   vào hai con số** bạn đã đọc ở B6.
2. **Cluster này đứng ở đâu trên bản đồ quản trị.** Đi qua bốn nhóm việc của bài 155 và với mỗi
   nhóm nói một hiện vật bạn đã sờ được trong lab. Rồi trả lời ba câu ranh giới: cluster có
   autoscaling Node không và **vì sao không**; bầu chọn leader có phối hợp đang bật hay tắt, làm
   sao bạn biết, và nó **thêm** gì lên trên cơ chế Lease chứ không thay thế cơ chế nào; quy trình
   Leader Migration của bài 198 có áp dụng không và **bốn dữ kiện nào** loại trừ nó. Cuối cùng,
   nói rõ **một chủ đề của giai đoạn 12 mà lab này cố ý không làm** và nó được trả ở đâu.

Khi mọi checkbox được đánh dấu và bạn không còn nhầm `cordon` với `drain`, *taint `NoSchedule`* với
*taint `NoExecute`*, *tắt node nhẹ nhàng* với *tắt node không nhẹ nhàng*, hay *bầu chọn leader
thông thường* với *bầu chọn leader có phối hợp* — Lab 12 và
[giai đoạn 12](../00-ALO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao) hoàn tất.

Lab này **không phát sinh nợ mới** và **không trả nợ nào**; xem
[sổ nợ lab](README.md#5-sổ-nợ-lab). Riêng [nợ #7](README.md#5-sổ-nợ-lab) — quản lý vòng đời
certificate — vẫn **đang treo**, đúng như bài [156](../156-certificates-vi.md) đã báo trước, và
được trả ở [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ). Những phần
cố ý không làm còn lại đều nằm trong bảng lý do ở
[mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Bảng dưới chỉ liệt kê
sự cố phát sinh từ nội dung bài học 12.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| **B7.6: worker2 không trở lại `Ready` sau khi bật máy** | Trên máy host: `vmrun -T ws list` xem VM có chạy không. Nếu chạy: `ssh lab-k8s-worker2 'systemctl is-active kubelet containerd'`; `ssh lab-k8s-worker2 'journalctl -u kubelet -n 80 --no-pager'` | Đi từ dưới lên, đừng nhảy cóc. **(1)** VM không boot → xem console VM trong VMware, không sửa gì trong cluster. **(2)** VM boot nhưng `ssh` chết → IP tĩnh của [A3](LAB-00-MOI-TRUONG-1.35.7.md#a3-đặt-ip-tĩnh-và-phân-giải-tên) không lên; sửa ở tầng netplan. **(3)** `ssh` được nhưng `kubelet` `inactive` → đọc `journalctl`; nguyên nhân thường gặp nhất là **swap bật lại sau reboot** (xem dòng kế) hoặc `containerd` chưa lên. **(4)** kubelet chạy mà Node vẫn `NotReady` → chạy lại [tầng 0 và tầng 1 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready). **Nếu sau vài phút vẫn không lên: tắt cả ba VM, restore cả ba về `04-metrics-ready`.** Không restore riêng worker2 |
| B7.6: kubelet báo lỗi liên quan swap sau reboot | `ssh lab-k8s-worker2 'swapon --show; grep -nE "^[^#].*[[:space:]]swap[[:space:]]" /etc/fstab'` | `/etc/fstab` còn dòng swap chưa comment nên swap bật lại khi boot, và kubelet mặc định **không khởi động trên node đang bật swap** (B5.3). Đây là lệch baseline của [A4.1](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites), không phải nội dung lab 12. Xử lý theo A4.1 rồi chạy lại B11; ghi vào evidence rằng baseline đã lệch |
| B7.5: VM không chuyển sang *Powered off* | Chạy lại khối PowerShell của B7.5; `vmrun -T ws list` | Node đang tắt dịch vụ, chờ thêm rồi chạy lại. Chỉ khi chờ mãi không xong mới dùng `vmrun -T ws stop <vmx> hard` — và khi đó **ghi vào evidence** rằng lần tắt này là cưỡng bức, vì nó đổi bản chất phép thử |
| B7.4: `drain` treo, không kết thúc | `kubectl get pdb -A`; `kubectl -n <ns> describe pdb <ten>`; đọc `$EV/b7-4-drain.txt` | `drain` gọi Eviction API nên nó **tôn trọng PDB** và thử lại yêu cầu bị từ chối. Tìm PDB đang chặn và xử lý nó; **không** thay `drain` bằng `kubectl delete pod` để "cho nhanh" — làm thế là bỏ qua đúng cơ chế mà bài học này muốn bạn thấy. Nếu PDB không phải của bạn thì cluster lệch mốc đầu vào: restore `04-metrics-ready` |
| B7.4: `drain` đòi `--force` | `kubectl get pods -A --field-selector spec.nodeName=lab-k8s-worker2 -o wide`; tìm Pod không có `ownerReferences` | Còn Pod trần trên node. Nếu là `thu-cordon` của B7.3 thì bạn quên xóa nó — xóa rồi chạy lại. Nếu là Pod lạ, xác định nguồn gốc trước; [mục 2](#2-quy-ước-và-an-toàn) cấm dùng `--force` trong lab này vì nó xóa vĩnh viễn |
| B7.4: `drain` báo lỗi về `emptyDir` | Đọc `$EV/b7-4-drain-tran.txt` | Có Pod dùng volume `emptyDir` trên node. Cờ `--delete-emptydir-data` trong lệnh của B7.4 xử lý đúng trường hợp này; nếu bạn tự gõ lại lệnh thì đừng bỏ cờ đó. Đọc lại cảnh báo về cờ này ở [mục 2](#2-quy-ước-và-an-toàn) trước khi mang thói quen sang cluster thật |
| B7.4: sau drain, số Pod `web` trên worker1 không đủ `$REPL` | `kubectl -n lab-12 get pods -o wide`; `kubectl -n lab-12 describe pod <pending>` | Nếu Pod `Pending` vì `didn't match pod topology spread constraints` thì manifest của bạn đang dùng `DoNotSchedule` thay vì `ScheduleAnyway` — sửa lại theo B7.1 và apply lại. Nếu `Pending` vì `Insufficient cpu/memory` thì worker1 không đủ chỗ; giảm `replicas` xuống `$WK_N` rồi chạy lại từ B7.1 |
| B7.5: `Ready` không chuyển sang `Unknown` sau nhiều vòng lặp | `kubectl get node lab-k8s-worker2 -o yaml \| sed -n '/conditions:/,/^  [a-z]/p'`; trên master `sudo grep -nE 'node-monitor\|pod-eviction' /etc/kubernetes/manifests/kube-controller-manager.yaml` | Khoảng chờ trước khi node controller đổi condition **phụ thuộc cấu hình**; nếu manifest không đặt cờ nào thì đang dùng giá trị mặc định. Tăng số vòng trong `seq` của B7.5 và chờ thêm. Nếu VM thật sự đã tắt mà condition mãi không đổi, kiểm tra `kube-controller-manager` còn `Running` không |
| B7.7: sau `uncordon` và `rollout restart`, worker2 vẫn 0 Pod | `kubectl get node lab-k8s-worker2 -o jsonpath='{.spec.taints}'`; `kubectl -n lab-12 describe pod <ten>` | Còn taint sót trên node (ví dụ `not-ready` chưa kịp biến mất) hoặc node vừa Ready chưa được chấm điểm. Chờ `kubectl wait --for=condition=Ready` rồi `rollout restart` lại một lần. **Không** gỡ taint bằng tay để ép Pod lên |
| B0.2 hoặc B5.3: `configz` không đọc được, hoặc thiếu `failSwapOn` | `kubectl get --raw "/api/v1/nodes/<node>/proxy/healthz"`; [tầng 6 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet) | Nếu `configz` rỗng thì đường control plane → kubelet đang hỏng: chạy lại tầng 6 của A5.4. Nếu đọc được nhưng **thiếu trường**, đọc thẳng file cấu hình: `ssh <node> 'sudo grep -iE "failSwapOn\|memorySwap\|shutdownGracePeriod" /var/lib/kubelet/config.yaml'`, ghi giá trị vào evidence và ghi rõ rằng trường vắng nghĩa là kubelet đang dùng mặc định |
| B6.1: hai giá trị khác `0s` | Đọc `$EV/b6-shutdown.txt` | Node của bạn **đã** bật graceful node shutdown từ trước — cluster lệch baseline. **Không sửa file để "đưa về chuẩn"**: ghi giá trị thật vào evidence, và ở B7.5 hiểu rằng kubelet sẽ chấm dứt Pod theo hai giai đoạn trước khi tắt. Việc đưa ba node về cùng một mức thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |
| B3.2: `SRV_VER` khác `K8S_BASELINE` | `kubectl get --raw /version`; đối chiếu [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) | Hoặc bạn điền nhầm biến, hoặc cluster đã bị nâng cấp ngoài lộ trình. Kiểm tra biến trước. Nếu cluster thật sự lệch version, dừng và cập nhật bảng A1.3 theo đúng quy trình của Lab 00 — không âm thầm sửa gate |
| B9.3: `kubectl proxy` không lên, hoặc cổng 8080 bận | `ss -lntp \| grep 8080`; đọc `$WK/proxy.log` | Có tiến trình khác giữ cổng. Đổi `--port` sang một cổng trống và sửa URL trong hai lệnh `curl` tương ứng. Nhớ `kill` proxy sau khi xong — nó là một cửa vào cluster mở trên localhost |
| B9.4: `curl` báo lỗi certificate | `openssl x509 -noout -subject -issuer -in "$WK/ca.crt"` | File CA trích ra rỗng hoặc sai. Kiểm tra `kubectl config view --raw --minify` có trường `certificate-authority-data` không; nếu kubeconfig dùng `certificate-authority` trỏ file thì đọc đường dẫn đó thay vì giải mã base64. **Đừng** chuyển sang `--insecure` để đi tiếp — đó là chính thứ B9.4 muốn tránh |
| B9.4: mã HTTP khác `403` khi liệt kê Pod | `kubectl auth can-i list pods -n lab-12 --as=system:serviceaccount:lab-12:default` | Nếu là `200`, ServiceAccount `default` đã được gán quyền từ lab trước — cluster lệch mốc. Nếu là `401`, token hết hạn: tạo lại token rồi gọi lại |
| B11.1: namespace `lab-12` kẹt `Terminating` | `kubectl get pods -n lab-12 -o wide`; `kubectl get events -n lab-12` | Thường là Pod còn trong grace period, hoặc một Pod nằm trên node vừa boot lại. Chờ; **không** cưỡng chế finalizer của Namespace |
| B11.2: `diff` báo checksum khác | `diff -u "$EV/b0-config-sha.txt" "$EV/b11-config-sha.txt"` | Cấu hình đã bị sửa, hoặc node boot lên với file khác. Cluster lệch baseline: tắt cả ba VM, restore cả ba về `04-metrics-ready`, và **không** sang lab sau trước khi gate này PASS |
| B11.3: `kubectl top node` chỉ in hai dòng | `kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide`; `kubectl get apiservice v1beta1.metrics.k8s.io` | Worker2 vừa boot chưa kịp báo cáo, hoặc metrics-server vừa bị dời chỗ ở B7.4 và đang khởi động lại. Chờ rồi chạy lại. Nếu APIService không `Available`, xử lý theo [mục 4 của Lab 11a](LAB-11A-OBSERVABILITY.md#4-troubleshooting-của-lab-này) — mốc `04-metrics-ready` được định nghĩa là có nguồn metric hoạt động |

---

## 5. Nguồn chính thức

Các phần giải thích command trong thân bài ưu tiên snapshot tài liệu Kubernetes v1.35
(`https://v1-35.docs.kubernetes.io/`) để hành vi và flag khớp minor version của cluster. Tài liệu
systemd và Linux man-pages chỉ giải thích cú pháp command có sẵn, không thay đổi quy trình hoặc gate.

- [Kubernetes v1.35 — Cluster Administration](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/)
- [Kubernetes v1.35 — Node Shutdowns](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/node-shutdown/)
- [Kubernetes v1.35 — Swap memory management](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/swap-memory-management/)
- [Kubernetes v1.35 — Node Autoscaling](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/)
- [Kubernetes v1.35 — Installing Addons](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/addons/)
- [Kubernetes v1.35 — Compatibility Version For Kubernetes Control Plane Components](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/compatibility-version/)
- [Kubernetes v1.35 — Coordinated Leader Election](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/coordinated-leader-election/)
- [Kubernetes v1.35 — Certificates](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/certificates/)
- [Kubernetes v1.35 — Administer a Cluster](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/)
- [Kubernetes v1.35 — Access Clusters Using the Kubernetes API](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/)
- [Kubernetes v1.35 — Migrate Replicated Control Plane To Use Cloud Controller Manager](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/controller-manager-leader-migration/)
- [Kubernetes v1.35 — Safely Drain a Node](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [Kubernetes v1.35 — `kubectl drain`](https://v1-35.docs.kubernetes.io/docs/reference/kubectl/generated/kubectl_drain/)
- [Kubernetes v1.35 — `kubectl cordon`](https://v1-35.docs.kubernetes.io/docs/reference/kubectl/generated/kubectl_cordon/)
- [Kubernetes v1.35 — `kubectl uncordon`](https://v1-35.docs.kubernetes.io/docs/reference/kubectl/generated/kubectl_uncordon/)
- [Kubernetes v1.35 — Nodes](https://v1-35.docs.kubernetes.io/docs/concepts/architecture/nodes/)
- [Kubernetes v1.35 — Leases](https://v1-35.docs.kubernetes.io/docs/concepts/architecture/leases/)
- [Kubernetes v1.35 — Taints and Tolerations](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Kubernetes v1.35 — API-initiated Eviction](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/)
- [Kubernetes v1.35 — Kubelet Configuration (v1beta1)](https://v1-35.docs.kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
- [systemd — inhibitor locks](https://www.freedesktop.org/wiki/Software/systemd/inhibit/)
- [systemd — `logind.conf`](https://www.freedesktop.org/software/systemd/man/latest/logind.conf.html)
