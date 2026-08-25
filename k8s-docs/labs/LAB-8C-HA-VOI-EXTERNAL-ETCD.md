# Lab 8c — HA với external etcd

> **Điểm bắt đầu:** snapshot `8x-ha-stacked` trên **cả sáu VM** của nhánh HA — mốc do Lab 8b tạo,
> xem [chuỗi snapshot](README.md#3-chuỗi-snapshot). Nhánh HA có tiền tố `8x-` và **không nằm trên
> chuỗi snapshot chính**.
> **Điểm kết thúc:** **tạo** snapshot `8x-ha-external` trên cả sáu VM. Lab này thay đổi hạ tầng
> vĩnh viễn — nó phá cụm stacked của Lab 8b và dựng lại theo topology external etcd — nên bắt buộc
> chụp mốc mới trước khi rời máy.
> **Lab trước:** [Lab 8b — HA với stacked etcd](LAB-8B-HA-VOI-STACKED-ETCD.md), lab đã dựng bộ VM
> này và chụp `8x-ha-stacked`.
> **Bộ VM chính** (`lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`) **không được đụng tới**
> trong toàn bộ lab; B0 và B15 đều có gate chứng minh điều đó.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục
[Giai đoạn 8 — Dựng cluster bằng kubeadm](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm)
— ba bài [06 — Các lựa chọn topology cho tính sẵn sàng cao](../06-ha-topology-vi.md),
[07 — Thiết lập cluster etcd có tính sẵn sàng cao với kubeadm](../07-setup-ha-etcd-with-kubeadm-vi.md)
và [08 — Tạo cluster có tính sẵn sàng cao với kubeadm](../08-high-availability-vi.md).
**Bài 07 là bài trục**: hơn một nửa phần B của lab là dựng cụm etcd ngoài theo đúng tám bước của nó.
Hai bài nền dùng để tra cứu là
[02 — Tạo một cluster với kubeadm](../02-create-cluster-kubeadm-vi.md) (phần `kubeadm reset` và gỡ
bỏ node) và [09 — Xử lý sự cố kubeadm](../09-troubleshooting-kubeadm-vi.md).

Lab 8b đã chứng minh nhánh **stacked** của bài 08. Lab 8c đi nhánh còn lại: **external etcd**. Hai
lab dùng **chung một bộ VM**, và chính việc đổi vai của sáu máy đó là cách rẻ nhất để thấy sự khác
nhau giữa hai topology — cùng phần cứng, cùng load balancer, chỉ khác chỗ etcd đứng.

Cluster của lab này giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab **không chép lại
con số phiên bản nào**. Kubernetes, Flannel và Pod CIDR đọc từ A1.3 rồi điền vào biến ở B0.4; image
etcd **không ghim tay** mà lấy đúng cách bài 07 chỉ — `kubeadm config images list` — rồi gate rằng
tag thật trong static Pod manifest khớp con số kubeadm khai.

Lab dùng Deployment và Service của
[giai đoạn 4](../00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller) và
[giai đoạn 5](../00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) làm công cụ chứng minh cluster sống.
**Không** dùng CRD (giai đoạn 14), audit log và mã hóa at rest (giai đoạn 22), metrics-server hay
HPA (giai đoạn 11), ResourceQuota (nhóm 7b) — tất cả đều đứng sau giai đoạn 8. Và **không** làm
backup/restore etcd: đó là [nợ #8](README.md#5-sổ-nợ-lab), trả ở
[giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa). `etcdctl`
trong lab này **chỉ được dùng để đọc**.

Trước khi bắt đầu, đọc [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
của Lab 00 để nhớ nguyên tắc "gate PASS rồi mới mở nội dung". A5.5 viết cho bộ VM chính; nhánh HA có
gate mở đầu riêng ở [B0](#b0-gate-mở-đầu-và-workspace) vì nó chạy trên bộ VM khác và bắt đầu từ mốc
khác. Chạy ngay gate ngắn dưới đây trên **máy host Windows**, PowerShell, trước khi bật bất kỳ máy
nào:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
if (Test-Path $vmrun) { 'PASS: tim thay vmrun' } else { 'FAIL: sua duong dan vmrun' }
& $vmrun -T ws list
```

**PASS:** dòng `PASS: tim thay vmrun` xuất hiện, và danh sách VM đang chạy **không chứa** máy nào
tên `lab-k8s-*`. Bộ VM chính phải đang tắt trong suốt lab này; B0.1 kiểm lại điều đó chặt hơn.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Hai topology HA của bài [06](../06-ha-topology-vi.md) khác nhau ở **quan hệ giữa etcd member và
  `kube-apiserver`**, và chỉ ra được quan hệ đó bằng số liệu đọc từ hai cluster mình đã tự dựng.
- `kubeadm reset` **xóa gì** và **không xóa gì** — bốn thứ còn lại sau reset cùng cách dọn từng thứ
  — chứng minh bằng gate chứ không bằng trí nhớ.
- Vì sao trên ba host etcd, kubelet phải chạy bằng một unit file **có độ ưu tiên cao hơn** unit của
  kubeadm, và vì sao ở bước đó kubelet **không nói chuyện với API server nào**.
- Vì sao mỗi host etcd cần **bộ certificate riêng**, SAN của chúng phải chứa gì, và vì sao `ca.key`
  không được rời host đầu tiên — chứng minh bằng danh sách file thật trên cả ba host.
- Hai vai trò của port `2380` và `2379`, đọc ra từ chính `kubeadmcfg.yaml` và từ static Pod manifest
  mà kubeadm sinh.
- Cụm etcd ba thành viên chạy **trước khi có bất kỳ cluster Kubernetes nào**, kiểm bằng
  `etcdctl member list`, `endpoint health` và `endpoint status` với đúng ba cờ certificate.
- Khối `etcd.external` trong `ClusterConfiguration` là **thứ duy nhất** phân biệt hai nhánh của bài
  [08](../08-high-availability-vi.md), và vì sao nhánh này **buộc** phải dùng `--config`.
- Ba bằng chứng độc lập rằng cluster đang dùng etcd ngoài: không Pod `component=etcd` nào, không
  `etcd.yaml` trong `/etc/kubernetes/manifests/`, và `--etcd-servers` của `kube-apiserver` trỏ tới
  ba host etcd.
- **Tính tách rời** của topology external, bằng ba phép thử thật: mất một etcd node, mất hai etcd
  node, mất một control plane node — và đối chiếu từng phép với điều đã xảy ra ở Lab 8b.
- Vì sao **hai** control plane cộng **ba** etcd member vẫn là cấu hình hợp lệ, quorum nằm ở tầng
  nào, và đánh đổi mà lab phải chấp nhận khi không còn worker riêng.
- Xóa `/etc/kubernetes/manifests/kube-apiserver.yaml` trên một control plane node rồi khôi phục,
  chứng minh cluster vẫn phục vụ qua load balancer trong suốt thời gian đó.
- Cleanup đúng phạm vi: cụm etcd ngoài và hai control plane còn nguyên, taint control plane trở về
  mặc định kubeadm, snapshot `8x-ha-external` có trên đủ sáu VM, và bộ VM chính không hề bị chạm.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm | Phần lab kiểm chứng |
| --- | --- |
| [06 — Các lựa chọn topology cho tính sẵn sàng cao](../06-ha-topology-vi.md) | B1 — đọc cụm stacked **đang chạy** của Lab 8b lần cuối: mỗi control plane node đúng một etcd member, tức quan hệ một-một của [topology stacked](../06-ha-topology-vi.md#stacked-etcd-topology); B11 — sau khi dựng xong, chứng minh quan hệ nhiều-nhiều của [topology external](../06-ha-topology-vi.md#external-etcd-topology) bằng `--etcd-servers` của **cả hai** apiserver cùng trỏ tới **cả ba** etcd host; B13 — "hỏng theo cặp" của stacked so với "tách rời" của external, đo bằng ba phép thử tắt máy; [mục 2.2](#22-vì-sao-hai-control-plane-cộng-ba-etcd-vẫn-hợp-lệ) — bảng đánh đổi số host |
| [07 — Thiết lập cluster etcd HA với kubeadm](../07-setup-ha-etcd-with-kubeadm-vi.md) | B4 — bước 1, kubelet làm trình quản lý dịch vụ cho etcd trên ba host; B5 — bước 2, `kubeadmcfg.yaml` riêng cho từng member; B6 — bước 3, 4 và 5, sinh CA, sinh certificate từng member, phân phối và giữ `ca.key` ở lại; B7 — bước 6 và 7, checklist file bắt buộc trên từng host rồi `kubeadm init phase etcd local`; B8 — bước 8, kiểm sức khỏe cụm bằng `etcdctl` chỉ đọc |
| [08 — Tạo cluster HA với kubeadm](../08-high-availability-vi.md) | B3 — mục *Tạo bộ cân bằng tải cho kube-apiserver*, phần chung của cả hai phương pháp, đổi backend sang hai control plane mới; B9 — mục *Các node etcd bên ngoài*: chép ba file certificate sang control plane đầu tiên, viết `kubeadm-config.yaml` với `etcd.external`, `kubeadm init --config --upload-certs`, cài CNI; B10 — mục *Các bước cho các node control plane còn lại* và ghi chú cân bằng lại CoreDNS; B14 — kiểm chứng `ControlPlaneEndpoint` luôn khớp địa chỉ load balancer bằng cách bỏ đi một apiserver |
| [02 — Tạo một cluster với kubeadm](../02-create-cluster-kubeadm-vi.md) (bài nền) | B2 — mục *Gỡ bỏ node* và *Dọn dẹp control plane*: thứ tự `drain` → `kubeadm reset` → `kubectl delete node`, và bốn thứ reset **không** dọn; B12 — mục *Cách ly control plane node*, gỡ taint `node-role.kubernetes.io/control-plane` |
| [09 — Xử lý sự cố kubeadm](../09-troubleshooting-kubeadm-vi.md) (bài nền) | [Mục 4](#4-troubleshooting-của-lab-này) — dùng đúng công dụng của một trang tra cứu: `kubeadm reset` bị treo khi container runtime không xóa container, lỗi certificate TLS của `$HOME/.kube/config`, Pod control plane crashloop |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài [07](../07-setup-ha-etcd-with-kubeadm-vi.md) — vai trò riêng của từng certificate `peer`, `server`, `healthcheck-client`, `apiserver-etcd-client` ở mức PKI | Lab **sinh và phân phối** đủ bốn bộ, và B6.4 đọc SAN của chúng, nhưng không đi vào vòng đời PKI: ký lại, xoay CA, đọc chuỗi tin cậy. Đó là **[nợ #7](README.md#5-sổ-nợ-lab)**, trả ở [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ) với bài [219](../219-kubeadm-certs-vi.md) |
| Bài [07](../07-setup-ha-etcd-with-kubeadm-vi.md) — chạy `etcdctl` bằng `crictl run` từ một image rời | Lab đạt cùng mục đích mà không kéo thêm image nào: `crictl exec` vào **chính container etcd đang chạy**, nơi `etcdctl` đã có sẵn trong image mà kubeadm chọn. Cách của bài chỉ cần khi etcd chưa chạy — không phải tình huống của B8 |
| Bài [07](../07-setup-ha-etcd-with-kubeadm-vi.md) — ghi chú "etcd không hỗ trợ dual-stack" | Toàn bộ nhánh HA chạy IPv4. Dual-stack là bài [05](../05-dual-stack-support-vi.md) riêng của giai đoạn 8, và bật nó lên đòi dựng lại cả cụm etcd lẫn control plane |
| Backup và restore etcd bằng `etcdctl snapshot save`/`restore` | **[Nợ #8](README.md#5-sổ-nợ-lab)**, phát sinh từ giai đoạn 8 và **chưa được trả ở lab này**. Lab dùng `etcdctl` **chỉ để đọc** — `member list`, `endpoint health`, `endpoint status` — đúng phần bài 07 dạy. Trả ở [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa) |
| Bài [08](../08-high-availability-vi.md) — mục *Cài đặt các worker* | Bộ VM của nhánh HA có **đúng sáu máy** và cả sáu đã có vai; xem [mục 2.2](#22-vì-sao-hai-control-plane-cộng-ba-etcd-vẫn-hợp-lệ). Lệnh `kubeadm join` cho worker đã được thực hành ở [A5.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a53-join-hai-worker) và ở Lab 8b; ở đây workload chạy trên control plane sau khi gỡ taint |
| Bài [08](../08-high-availability-vi.md) — mục *Phân phối certificate thủ công* | Lab dùng `--upload-certs`, tức nhánh còn lại của cùng mục đó. Chép tay bằng `ssh-agent` và `scp` là **cách khác cho cùng một việc**, không thêm kiến thức mới, và nhân đôi thời lượng vốn đã 4 giờ. B6 **đã** thực hành chính kỹ thuật `scp` certificate đó, ở tầng etcd |
| Bài [08](../08-high-availability-vi.md) — load balancer có tính sẵn sàng cao | Nhánh HA chỉ có một máy `lab-ha-lb`, nên load balancer là **điểm hỏng đơn** — xem [mục 2.3](#23-một-load-balancer-vẫn-là-điểm-hỏng-đơn). Dựng cặp LB cần thêm máy thứ bảy và một giao thức VIP; cả hai nằm ngoài phạm vi giai đoạn 8 |
| Bài [06](../06-ha-topology-vi.md) — cấu hình ba control plane node đầy đủ theo khuyến nghị | Cần bảy máy: 3 etcd + 3 control plane + 1 LB. Bộ VM có sáu. [Mục 2.2](#22-vì-sao-hai-control-plane-cộng-ba-etcd-vẫn-hợp-lệ) trình bày đánh đổi này ở dạng bài học chứ không giấu nó đi |

Đây **không phải nợ lộ trình mới**. Nợ #7 và nợ #8 vốn đã có trong [sổ nợ lab](README.md#5-sổ-nợ-lab)
và **không được trả ở lab này**; những phần còn lại là giới hạn cố định của môi trường lab.

### 1.2. Thời lượng

4 giờ, tính từ lúc gate B0 đã PASS. Ba đoạn tốn thời gian nhất là B2 (reset năm node), B6–B7
(certificate và static Pod trên ba host) và B13 (ba phép thử tắt máy, mỗi phép có hai chiều tắt và
bật). **Mọi bước chờ phụ thuộc cấu hình cluster và phần cứng host**, nên chúng đều được viết dưới
dạng `kubectl wait --timeout` hoặc vòng lặp có điều kiện dừng và số vòng tối đa, không phải một con
số cố định.

---

## 2. Quy ước và an toàn

> **Cảnh báo — lab này PHÁ một cluster đang chạy.** B2 chạy `kubeadm reset` trên năm node của cụm
> stacked mà Lab 8b đã dựng. Đó là thao tác **không hoàn tác được bằng lệnh**; đường lui duy nhất là
> snapshot. Không mở B2 khi gate B0.2 chưa PASS.

> **Cảnh báo — lab này TẮT máy ảo nhiều lần, kể cả hai máy cùng lúc.** B13 cố ý làm mất quorum của
> cụm etcd. Dữ liệu etcd nằm trên đĩa của từng member nên cụm hồi phục khi các máy bật lại, nhưng
> **chỉ khi bạn tắt đúng máy và bật lại đủ máy**. Đọc hết [mục 2.1](#21-bảng-đổi-vai-của-sáu-vm)
> trước khi chạm vào bất kỳ nút nguồn nào.

### 2.1. Bảng đổi vai của sáu VM

Lab 8c chạy trên **đúng bộ VM mà Lab 8b đã dựng**, không thêm máy nào. Spec máy, cách cài OS,
containerd và kubeadm nằm ở
[bộ VM riêng của nhóm lab HA](LAB-8B-HA-VOI-STACKED-ETCD.md#21-bộ-vm-riêng-của-nhóm-lab-ha) và ở
[phần A của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a4-chuẩn-bị-os-và-container-runtime); lab này
**không chép lại** một dòng nào trong đó.

Tên máy được đặt trung tính (`lab-ha-1` … `lab-ha-5`) chính là để đổi vai giữa hai lab:

| Tên máy | IP | Vai ở Lab 8b | **Vai ở Lab 8c** | Ghi chú của Lab 8c |
| --- | --- | --- | --- | --- |
| `lab-ha-lb` | `192.168.100.230` | load balancer | load balancer | Giữ vai, **đổi backend** ở B3 |
| `lab-ha-1` | `192.168.100.231` | control plane 1 | **etcd node 1** | `$HOST0` của bài 07 — nơi sinh CA, nơi `ca.key` ở lại |
| `lab-ha-2` | `192.168.100.232` | control plane 2 | **etcd node 2** | `$HOST1` của bài 07 |
| `lab-ha-3` | `192.168.100.233` | control plane 3 | **etcd node 3** | `$HOST2` của bài 07 |
| `lab-ha-4` | `192.168.100.234` | worker 1 | **control plane 1** | Nơi chạy `kubeadm init`; **node mặc định gõ lệnh của lab này** |
| `lab-ha-5` | `192.168.100.235` | worker 2 | **control plane 2** | Node join thêm, và là **fault target duy nhất** ở tầng control plane |

Hệ quả phải nhớ suốt lab:

- **`controlPlaneEndpoint` là `192.168.100.230:6443`**, tức `lab-ha-lb`. Không phải IP của một
  control plane node nào. Bài [08](../08-high-availability-vi.md) đặt thành yêu cầu riêng: địa chỉ
  của load balancer phải **luôn khớp** với `ControlPlaneEndpoint` của kubeadm.
- **Backend của HAProxy đổi thành `192.168.100.234:6443` và `192.168.100.235:6443`.** Ở Lab 8b nó
  trỏ tới `.231`, `.232`, `.233` — ba máy giờ đã thành etcd node và **không còn nghe cổng 6443
  nữa**. Quên bước này thì `kubeadm init` treo ở giai đoạn chờ control plane.
- **Ba etcd host nghe `2379` và `2380`, không nghe `6443`.** Hai control plane node nghe `6443`,
  không nghe `2379`/`2380`. Đây là ranh giới cụ thể của cái mà bài 06 gọi là *tách rời*.
- **Fault injection ở tầng control plane chỉ chạy trên `lab-ha-5`.** `lab-ha-4` giữ kubeconfig quản
  trị và là nơi bạn quan sát; mất nó là mất đường quan sát. Riêng B13 tắt thêm etcd node — đó là nội
  dung bài học, có mục riêng, và luôn bật lại ngay sau khi đo xong.
- **Quy ước "fault injection chỉ trên `lab-k8s-worker2`" của
  [mục 6 README](README.md#6-quy-ước-chung-trong-mọi-lab) áp cho bộ VM chính.** Bộ VM chính **không
  chạy** trong lab này; trên nhánh HA, vai "máy được phép làm hỏng" thuộc về `lab-ha-5`.

### 2.2. Vì sao hai control plane cộng ba etcd vẫn hợp lệ

Đây là chỗ dễ tưởng lab đang làm ẩu, nên đọc kỹ. Bài [06](../06-ha-topology-vi.md) viết rằng topology
external **yêu cầu số lượng host gấp đôi** so với stacked: tối thiểu **ba host cho control plane** và
**ba host cho etcd**, chưa tính worker. Bộ VM của nhánh HA có **sáu máy**, trong đó một máy đã là
load balancer. Làm phép trừ thì còn năm máy cho hai tầng — không đủ cho khuyến nghị đầy đủ. Lab chọn
**3 etcd + 2 control plane** và **không có worker riêng**.

Lựa chọn đó không tùy tiện. Nó rơi đúng vào điểm mà bài 06 muốn dạy:

- **Quorum nằm ở tầng etcd, không nằm ở tầng control plane.** Bài
  [08](../08-high-availability-vi.md) nói rõ hai mức khác nhau: với etcd, "việc có số lẻ thành viên
  trong cluster etcd **là một yêu cầu** để đạt được quorum bỏ phiếu tối ưu"; còn với control plane
  thì chỉ là "việc có số lẻ node control plane **có thể giúp ích** cho việc bầu chọn leader". Một
  bên là yêu cầu, một bên là khuyến nghị — và lab giữ nguyên vẹn cái là yêu cầu: **ba etcd member**.
- **Vì tách rời, số control plane không quyết định số bản sao dữ liệu.** Bài 06 mô tả topology
  external là nơi "mỗi etcd host giao tiếp với `kube-apiserver` của **từng** control plane node" —
  quan hệ nhiều-nhiều. Thêm hay bớt một `kube-apiserver` không thêm hay bớt một bản sao dữ liệu nào,
  vì trạng thái nằm ở cụm etcd. Đó chính là câu bài 06 viết: mất một instance control plane "có ít
  tác động hơn và **không ảnh hưởng đến tính dự phòng của cluster** nhiều như topology HA xếp chồng".
- **So thẳng với Lab 8b thì thấy ngay.** Ở stacked, ba control plane node **là** ba etcd member; bớt
  một máy là bớt cả hai thứ cùng lúc — bài 06 gọi đó là **hỏng theo cặp**. Ở external, bớt
  `lab-ha-5` chỉ bớt một `kube-apiserver`; cụm etcd vẫn ba member. B13 đo đúng hai vế này.

Bảng đối chiếu để nhớ:

| | Lab 8b — stacked | Lab 8c — external |
| --- | --- | --- |
| Số máy chạy `kube-apiserver` | 3 | 2 |
| Số etcd member | 3, nằm trên chính ba máy đó | 3, nằm trên ba máy khác |
| Tắt **một** máy control plane | mất 1 apiserver **và** 1 etcd member | mất 1 apiserver, etcd vẫn 3 member |
| Tắt **một** máy etcd | không tồn tại tình huống riêng biệt này | mất 1 etcd member, apiserver vẫn 2 |
| Cụm còn chịu được mất bao nhiêu etcd member | 1 | 1 |
| Số máy phải trả | 3, cộng worker | 3 + 2, cộng worker |

**Đánh đổi phải nói thẳng:** vì hết máy, cluster của Lab 8c **không có worker riêng**. Để chạy được
một Deployment, B12 **gỡ taint** `node-role.kubernetes.io/control-plane` trên cả hai control plane
node — đúng thao tác mà bài [02](../02-create-cluster-kubeadm-vi.md) mô tả ở mục *Cách ly control
plane node*. **Production không làm vậy**: bài 02 nói mặc định cluster không lập lịch Pod trên
control plane node **vì lý do bảo mật**, và một workload thoát ra được sẽ đứng ngay cạnh
`admin.conf`, khóa CA và cổng `10250`. Ở đây gỡ taint là thỏa hiệp có ý thức của môi trường lab, và
B15 **trả taint về nguyên trạng** trước khi chụp snapshot.

Cấu hình khuyến nghị đầy đủ cần **bảy** máy: 3 etcd + 3 control plane + 1 LB, cộng worker. Nếu host
Windows của bạn dư RAM cho máy thứ bảy thì thêm một control plane node là cách nâng cấp đúng — nhưng
bộ VM chuẩn của chuỗi lab dừng ở sáu, và mọi gate trong file này viết theo con số sáu.

### 2.3. Một load balancer vẫn là điểm hỏng đơn

`lab-ha-lb` là máy duy nhất đứng trước hai `kube-apiserver`. Nó chết thì `controlPlaneEndpoint` chết
theo, và cluster mất đường vào dù cả hai control plane node còn sống. Lab 8b đã bàn kỹ điều này khi
dựng load balancer; xem [Lab 8b](LAB-8B-HA-VOI-STACKED-ETCD.md). Ở đây chỉ nhắc lại một câu để không
ai kết luận sai từ ba phép thử của B13: **B13 chứng minh tính tách rời giữa etcd và control plane,
không chứng minh cluster chịu được mọi loại hỏng hóc.**

### 2.4. Các quy ước còn lại

- **Mọi khối lệnh đều mở đầu bằng dòng ghi rõ chạy trên máy nào.** Lab có sáu máy Linux cộng một máy
  host Windows; đây là chỗ dễ sai nhất của cả chuỗi lab. Không khối nào trong phần B được phép mơ hồ
  về node. Khi không có dòng đó, mặc định là `lab-ha-4`.
- **Các khối PowerShell chạy trên máy host Windows**, luôn mở đầu bằng "Chạy trên **máy host**".
  Chúng dùng lại hai biến `$vmrun` và `$vmxHA` đặt ở B0.1, nên giữ nguyên cửa sổ PowerShell đó suốt
  lab.
- **Chạy phần B trong cùng một phiên shell trên mỗi máy.** Gate của lab so sánh với biến đặt ở B0.4
  (`K8S_VER`, `FLANNEL_VER`, `POD_CIDR`, `LB`, `E1`, `E2`, `E3`, `CP1`, `CP2`, `EV`, `WK`); mở shell
  mới giữa chừng là mất hết. Với ba host etcd, B4 và B6 hướng dẫn đặt lại cùng bộ biến trên từng máy.
- **Bằng chứng ghi vào `~/lab-evidence/8c/`** trên máy sinh ra nó; manifest và file tạm ghi vào
  `~/lab-work/8c/`. Vì lab chạy trên sáu máy, mỗi file evidence mang tên node ở cuối để gộp lại
  không lẫn.
- **Không con số phiên bản nào được viết cứng.** Kubernetes, Flannel và Pod CIDR điền vào biến ở
  B0.4 từ [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); thành phần ngoài Lab 00
  tra ở [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). **Image etcd
  không điền tay**: B7.3 lấy nó từ `kubeadm config images list` rồi gate rằng manifest dùng đúng tag
  đó.
- **HAProxy lấy từ gói phân phối của Ubuntu, không ghim version.** Nó không nằm trong A1.3 vì nó
  không phải thành phần Kubernetes; nó chỉ chuyển tiếp TCP. Lab 8b đã cài nó trên `lab-ha-lb`.
- **Tuyệt đối không chạm bộ VM chính.** `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2` phải
  ở trạng thái *Powered off* và giữ nguyên snapshot `03-storage-ready` từ đầu đến cuối. B0.1 kiểm
  trước, B15.5 kiểm sau.
- **`etcdctl` chỉ dùng để đọc.** Ba lệnh được phép trong lab này là `member list`, `endpoint health`
  và `endpoint status`. `snapshot save`, `snapshot restore`, `member add`, `member remove`, `put`,
  `del` đều **cấm** — chúng thuộc [nợ #8](README.md#5-sổ-nợ-lab) và
  [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa).
- Image dùng cho workload của lab là `busybox:1.37` đã khóa ở
  [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) — cùng image mà gate A5.4 của Lab
  00 dùng, nên không phát sinh thành phần mới.
- Dòng bắt đầu bằng `**PASS:**` là gate; không đi tiếp khi gate fail.

---

# Phần B — Thực hành kiến thức giai đoạn 8 (HA external etcd)

> **Node gõ lệnh đổi giữa chừng.** Từ B0 đến hết B2, cụm đang chạy là cụm **stacked** của Lab 8b,
> nên kubeconfig quản trị nằm trên `lab-ha-1`; mọi lệnh `kubectl` chạy ở đó. Từ B9 trở đi, cluster
> mới có kubeconfig trên `lab-ha-4`, và **`lab-ha-4` là node mặc định** cho phần còn lại của lab.
> Mỗi khối lệnh đều ghi rõ node; đừng suy từ thói quen.

## B0. Gate mở đầu và workspace

**Mục đích:** chứng minh điểm bắt đầu đúng — bộ VM chính còn nguyên và đang tắt, sáu VM HA còn mốc
`8x-ha-stacked`, cụm stacked bật lên chạy được — rồi khóa các con số phiên bản vào biến.

### B0.1. Gate trên máy host: hai bộ VM, hai trạng thái

Chạy trên **máy host**, PowerShell. Sửa đường dẫn `.vmx` nếu VM của bạn nằm chỗ khác; **giữ nguyên
thứ tự** của `$vmxHA` vì B0.1 và B15 đều bật/tắt máy theo đúng thứ tự này:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'

$vmxMain = @(
  'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
  'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
  'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
)

$vmxHA = @(
  'E:\Virtual Machines\lab-ha-lb\lab-ha-lb.vmx'
  'E:\Virtual Machines\lab-ha-1\lab-ha-1.vmx'
  'E:\Virtual Machines\lab-ha-2\lab-ha-2.vmx'
  'E:\Virtual Machines\lab-ha-3\lab-ha-3.vmx'
  'E:\Virtual Machines\lab-ha-4\lab-ha-4.vmx'
  'E:\Virtual Machines\lab-ha-5\lab-ha-5.vmx'
)

$running = & $vmrun -T ws list
```

Gate thứ nhất — **bộ VM chính phải tắt và còn `03-storage-ready`**:

```powershell
foreach ($f in $vmxMain) {
  $isOff  = ($running -notcontains $f)
  $snaps  = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  $hasSnap = ($snaps -ccontains '03-storage-ready')
  if ($isOff -and $hasSnap) { "PASS: $f (off, co 03-storage-ready)" }
  elseif (-not $isOff)      { "FAIL: $f dang CHAY — tat no truoc khi mo Lab 8c" }
  else                      { "FAIL: $f mat snapshot 03-storage-ready -> $($snaps -join ', ')" }
}
```

**PASS:** đúng ba dòng `PASS:`. Còn một dòng `FAIL:` thì **dừng**. Bộ VM chính là nơi 15 lab khác
đang đứng; Lab 8c không được đụng vào nó, và bật nó song song với sáu VM HA sẽ vắt kiệt RAM host.

Gate thứ hai — **sáu VM HA phải còn `8x-ha-stacked`**:

```powershell
foreach ($f in $vmxHA) {
  $snaps = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  $stacked = ($snaps -ccontains '8x-ha-stacked')
  $vmready = ($snaps -ccontains '8x-vm-ready')
  if     ($stacked -and $vmready) { "PASS: $f (co ca 8x-ha-stacked va 8x-vm-ready)" }
  elseif ($stacked)               { "PASS-YEU: $f co 8x-ha-stacked nhung THIEU 8x-vm-ready" }
  else                            { "FAIL: $f -> $($snaps -join ', ')" }
}
```

**PASS:** sáu dòng bắt đầu bằng `PASS`. Một dòng `FAIL:` nghĩa là máy đó chưa qua Lab 8b — quay lại
[Lab 8b](LAB-8B-HA-VOI-STACKED-ETCD.md) trước, đừng dựng tay.

Dòng `PASS-YEU:` không chặn bạn, nhưng phải biết hệ quả: `8x-vm-ready` là mốc "VM sạch đã cài
kubeadm" mà Lab 8b chụp, và nó là **đường lui của B2**. Thiếu nó thì khi gate reset fail bạn chỉ còn
đường quay về `8x-ha-stacked` và làm lại từ đầu. Ghi lại máy nào thiếu.

Bật sáu VM theo đúng thứ tự mảng — load balancer trước, control plane của cụm stacked sau, hai
worker cuối:

```powershell
foreach ($f in $vmxHA) { & $vmrun -T ws start $f }
& $vmrun -T ws list
```

**PASS:** `vmrun list` liệt kê đủ sáu `.vmx` của `$vmxHA` và **không** liệt kê máy nào của
`$vmxMain`. Giữ nguyên cửa sổ PowerShell này — B13 và B15 dùng lại `$vmrun`, `$vmxHA`, `$vmxMain`.

### B0.2. Gate trên cluster: cụm stacked của Lab 8b còn sống

Chạy trên **`lab-ha-1`** (control plane 1 của cụm stacked):

```bash
kubectl wait --for=condition=Ready node --all --timeout=600s
kubectl get nodes -o wide
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'

NODE_N="$(kubectl get nodes --no-headers | wc -l)"
CP_N="$(kubectl get nodes -l 'node-role.kubernetes.io/control-plane' --no-headers | wc -l)"
ETCD_POD_N="$(kubectl -n kube-system get pod -l component=etcd --no-headers | wc -l)"
API_POD_N="$(kubectl -n kube-system get pod -l component=kube-apiserver --no-headers | wc -l)"
echo "node=$NODE_N | control plane=$CP_N | Pod etcd=$ETCD_POD_N | Pod apiserver=$API_POD_N"

test "$NODE_N" -eq 5 && test "$CP_N" -eq 3 \
  && test "$ETCD_POD_N" -eq 3 && test "$API_POD_N" -eq 3 \
  && echo 'PASS: dung moc 8x-ha-stacked — 3 control plane, 3 Pod etcd, 2 worker'
```

**PASS:** dòng `PASS: readyz ok` và dòng `PASS: dung moc 8x-ha-stacked …` cùng xuất hiện. Con số
**3 Pod etcd** ở đây là dữ kiện quan trọng nhất của cả lab: B11 sẽ chứng minh cluster mới có **0**
Pod như vậy. Nếu gate fail, restore cả sáu VM về `8x-ha-stacked` rồi chạy lại B0 — đừng vá tay một
cụm sắp bị phá.

### B0.3. Workspace và bằng chứng "trước"

Chạy trên **`lab-ha-1`**:

```bash
mkdir -p ~/lab-work/8c ~/lab-evidence/8c
WK=~/lab-work/8c
EV=~/lab-evidence/8c

{
  echo "=== $(date -Is) — cum stacked cua Lab 8b, ngay truoc khi reset ==="
  echo '--- nodes'; kubectl get nodes -o wide
  echo '--- control plane pods'; kubectl -n kube-system get pods -l tier=control-plane -o wide
  echo '--- etcd-servers cua tung apiserver'
  for n in lab-ha-1 lab-ha-2 lab-ha-3; do
    echo "== $n"
    ssh "$n" "sudo grep -E '^\s+- --etcd-servers=' /etc/kubernetes/manifests/kube-apiserver.yaml"
  done
  echo '--- kubeadm-config ClusterConfiguration'
  kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}'
} | tee "$EV/b0-truoc-stacked.txt"

test -s "$EV/b0-truoc-stacked.txt" && echo 'PASS: da ghi anh chup cum stacked truoc khi pha'
```

**PASS:** dòng `PASS: da ghi anh chup …` xuất hiện, và trong file có ba dòng `--etcd-servers=`.
File này là bằng chứng đối chiếu của B1 và B11; **giữ lại**, đừng xóa ở cleanup.

Nếu `ssh lab-ha-2` hỏi mật khẩu, đó là điều bình thường — Lab 8b không bắt buộc dựng SSH key. Lab
8c dùng `ssh`/`scp` giữa các node ở nhiều bước (B6, B9), nên nếu chưa có key thì **tạo ngay bây
giờ** trên `lab-ha-1` để phần sau không bị hỏi mật khẩu giữa chừng:

```bash
test -f ~/.ssh/id_ed25519 || ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519

# Ke ca chinh lab-ha-1: nhieu gate cua B2 va B7 chay vong lap ssh qua ca nam node,
# trong do co ca may dang go lenh.
for n in lab-ha-1 lab-ha-2 lab-ha-3 lab-ha-4 lab-ha-5; do
  ssh-copy-id -o StrictHostKeyChecking=accept-new "$n"
done

OKSSH=0
for n in lab-ha-1 lab-ha-2 lab-ha-3 lab-ha-4 lab-ha-5; do
  ssh -o BatchMode=yes "$n" 'hostname' >/dev/null 2>&1 && OKSSH=$(( OKSSH + 1 ))     || echo "FAIL: ssh khong mat khau toi $n"
done
echo "ssh khong mat khau toi $OKSSH/5 node"
test "$OKSSH" -eq 5 && echo 'PASS: ssh giua cac node HA da thong'
```

**PASS:** dòng `PASS: ssh giua cac node HA da thong` xuất hiện. Đây là điều kiện mà chính bài
[07](../07-setup-ha-etcd-with-kubeadm-vi.md) đặt ra ở mục *Trước khi bạn bắt đầu*: "một hạ tầng nào
đó để sao chép file giữa các host. Ví dụ, `ssh` và `scp`".

### B0.4. Khóa phiên bản vào biến

Ba giá trị dưới đây **không được viết cứng vào lab này**. Mở
[bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa), đọc ba dòng
*Kubernetes control plane*, *CNI (Flannel manifest release)* và *Pod CIDR*, rồi điền:

```bash
# Dien tu bang A1.3 cua Lab 00. Vi du dinh dang: vX.Y.Z / vX.Y.Z / 10.x.0.0/16
K8S_VER='<dien Kubernetes control plane tu A1.3>'
FLANNEL_VER='<dien Flannel manifest release tu A1.3>'
POD_CIDR='<dien Pod CIDR tu A1.3>'

BAD=0
for v in "$K8S_VER" "$FLANNEL_VER"; do
  case "$v" in
    v[0-9]*.[0-9]*.[0-9]*) ;;
    *) echo "FAIL: '$v' khong co dang vX.Y.Z nhu A1.3 ghi"; BAD=1 ;;
  esac
done
case "$POD_CIDR" in
  [0-9]*.[0-9]*.[0-9]*.[0-9]*/[0-9]*) ;;
  *) echo "FAIL: POD_CIDR='$POD_CIDR' khong co dang CIDR"; BAD=1 ;;
esac

test "$BAD" -eq 0 && echo "PASS: K8S_VER=$K8S_VER FLANNEL_VER=$FLANNEL_VER POD_CIDR=$POD_CIDR"
```

**PASS:** dòng `PASS: K8S_VER=… FLANNEL_VER=… POD_CIDR=…` xuất hiện, không còn dòng `FAIL:`.

Đối chiếu `K8S_VER` với binary thật đang có trên máy — nếu lệch thì cluster mới sẽ không khớp
baseline:

```bash
test "$(kubeadm version -o short)" = "$K8S_VER" \
  && echo 'PASS: kubeadm tren lab-ha-1 dung version cua A1.3' \
  || echo "FAIL: kubeadm la $(kubeadm version -o short), A1.3 ghi $K8S_VER"
```

**PASS:** dòng `PASS: kubeadm tren lab-ha-1 dung version cua A1.3` xuất hiện. Nếu FAIL, xử lý theo
[A4.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl) — không đi
tiếp với version lệch.

Đặt luôn địa chỉ của sáu máy vào biến, dùng lại suốt lab:

```bash
LB='192.168.100.230'
E1='192.168.100.231'; E2='192.168.100.232'; E3='192.168.100.233'
CP1='192.168.100.234'; CP2='192.168.100.235'
N1='lab-ha-1'; N2='lab-ha-2'; N3='lab-ha-3'; N4='lab-ha-4'; N5='lab-ha-5'

OKIP=0
for pair in "$N1 $E1" "$N2 $E2" "$N3 $E3" "$N4 $CP1" "$N5 $CP2"; do
  set -- $pair
  test "$(getent hosts "$1" | awk '{print $1}')" = "$2" && OKIP=$(( OKIP + 1 )) \
    || echo "FAIL: $1 khong phan giai ra $2"
done
echo "cap ten/IP dung: $OKIP/5"
test "$OKIP" -eq 5 && echo 'PASS: /etc/hosts khop bang doi vai o muc 2.1'
```

**PASS:** dòng `PASS: /etc/hosts khop bang doi vai o muc 2.1` xuất hiện. Sai ở đây là sai suốt lab:
mọi certificate SAN, mọi endpoint etcd và cả `--etcd-servers` đều xây trên năm cặp tên/IP này.

---

## B1. Bài 06 — đọc cụm stacked lần cuối trước khi phá

**Mục đích:** biến bài [06](../06-ha-topology-vi.md) từ hai đoạn văn thành hai con số đọc được trên
cluster thật, ngay trước khi cluster đó biến mất. B11 sẽ đọc lại đúng hai con số này trên cluster
external và so.

### B1.1. Quan hệ một-một của topology stacked

Chạy trên **`lab-ha-1`**:

```bash
kubectl -n kube-system get pod -l component=etcd -o wide \
  | tee "$EV/b1-etcd-pods-stacked.txt"

for n in "$N1" "$N2" "$N3"; do
  echo "== $n"
  ssh "$n" "sudo grep -E '^\s+- --etcd-servers=' /etc/kubernetes/manifests/kube-apiserver.yaml"
done | tee "$EV/b1-etcd-servers-stacked.txt"

CNT_LOCAL="$(grep -c 'etcd-servers=https://127.0.0.1:2379' "$EV/b1-etcd-servers-stacked.txt")"
echo "so apiserver tro toi etcd cuc bo cua chinh no: $CNT_LOCAL/3"

test "$CNT_LOCAL" -eq 3 \
  && echo 'PASS: stacked — moi apiserver chi noi chuyen voi etcd member tren chinh node do'
```

**PASS:** dòng `PASS: stacked — …` xuất hiện, và `kubectl get pod -l component=etcd` liệt kê đúng ba
Pod, mỗi Pod trên một node khác nhau.

**Ý nghĩa:** đây là câu của bài 06 dịch sang dữ liệu. Bài viết: trong stacked, "mỗi control plane
node tạo một etcd member cục bộ và etcd member này **chỉ giao tiếp với `kube-apiserver` của chính
node đó**". Bằng chứng là ba dòng `--etcd-servers=https://127.0.0.1:2379` — ba apiserver, ba địa chỉ
loopback, **không apiserver nào biết địa chỉ của hai member kia**. Ba member vẫn hợp thành một cụm
etcd và nói chuyện với nhau qua port peer, nhưng quan hệ *apiserver ↔ member* thì đúng là một-một.

Ghi con số này ra giấy: **stacked = 3 apiserver, 3 etcd member, 3 máy**. B11 sẽ cho ra
**2 apiserver, 3 etcd member, 5 máy**.

### B1.2. "Hỏng theo cặp" đọc từ chính bảng node

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,ROLES:.metadata.labels.node-role\.kubernetes\.io/control-plane' \
  | tee "$EV/b1-nodes-stacked.txt"

kubectl -n kube-system get pods -l tier=control-plane \
  -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName,COMP:.metadata.labels.component' \
  | tee "$EV/b1-tier-cp-stacked.txt"

PAIR="$(awk '$3=="etcd"{print $2}' "$EV/b1-tier-cp-stacked.txt" | sort -u | wc -l)"
APISRV="$(awk '$3=="kube-apiserver"{print $2}' "$EV/b1-tier-cp-stacked.txt" | sort -u | wc -l)"
echo "node co etcd=$PAIR | node co apiserver=$APISRV"

test "$PAIR" -eq "$APISRV" && test "$PAIR" -eq 3 \
  && echo 'PASS: tap node chay etcd TRUNG KHIT tap node chay apiserver — do la "hong theo cap"'
```

**PASS:** dòng `PASS: tap node chay etcd TRUNG KHIT …` xuất hiện.

**Ý nghĩa:** bài 06 gọi rủi ro của stacked là **hỏng hóc theo cặp**: "nếu một node bị sập, cả một
etcd member lẫn một instance control plane đều bị mất, và tính dự phòng bị suy giảm". Hai tập node
trùng khít nhau chính là định nghĩa toán học của câu đó. Sau khi dựng xong external, hai tập này sẽ
**rời nhau hoàn toàn** — giao bằng rỗng — và B13 sẽ đo hệ quả của việc rời nhau ấy.

---

## B2. `kubeadm reset` — phá cụm stacked có hiểu

**Mục đích:** đây **không phải bước dọn dẹp**, đây là nội dung học. Bài
[02](../02-create-cluster-kubeadm-vi.md) dạy hai điều mà người mới hay bỏ qua: thứ tự đúng khi gỡ
node, và việc `kubeadm reset` là dọn dẹp **best-effort** — nó để lại bốn thứ mà bạn phải tự dọn.
Chưa dọn hết bốn thứ đó thì cụm etcd của B7 sẽ dựng trên nền bẩn và hỏng theo cách rất khó chẩn
đoán.

> **Không hoàn tác được.** Sau B2.2, cụm stacked của Lab 8b không còn. Bạn vẫn còn snapshot
> `8x-ha-stacked` để quay lại, nhưng không còn lệnh nào undo được. Đảm bảo B0.1 và B0.2 đều PASS
> trước khi chạy dòng đầu tiên.

### B2.1. Đọc trước: reset xóa gì và không xóa gì

Bài 02 nói ngắn gọn ở mục *Gỡ bỏ node* và *Dọn dẹp control plane*: `kubeadm reset` "kích hoạt việc
dọn dẹp ở mức nỗ lực tốt nhất có thể (best-effort)", và "quá trình reset **không** đặt lại hay dọn
dẹp các quy tắc iptables hoặc các bảng IPVS". Bài dẫn tiếp sang
[tài liệu tham khảo `kubeadm reset`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
cho danh sách đầy đủ. Gộp lại thành bảng làm việc:

| Nhóm | Thứ | Reset có tự dọn không |
| --- | --- | --- |
| Static Pod manifest | `/etc/kubernetes/manifests/*.yaml` | **Có** |
| kubeconfig của cluster | `/etc/kubernetes/admin.conf`, `kubelet.conf`, `controller-manager.conf`, `scheduler.conf` | **Có** |
| PKI của cluster | `/etc/kubernetes/pki/` | **Có** |
| Thành viên etcd cục bộ | member của node này bị gỡ khỏi cụm etcd (phase `remove-etcd-member`) | **Có** |
| Dữ liệu etcd | `/var/lib/etcd/` khi etcd là **cục bộ** | **Có** — nhưng tài liệu ghi rõ reset **không** xóa dữ liệu etcd khi dùng **external** etcd |
| Cấu hình CNI | `/etc/cni/net.d/` | **Không** |
| kubeconfig của user | `$HOME/.kube/` | **Không** |
| Quy tắc mạng do kube-proxy tạo | iptables / nftables / IPVS | **Không** |

Dòng cuối cùng của cột giữa là thứ đáng nhớ nhất cho **chính lab này**: từ B9 trở đi cluster dùng
external etcd, nên nếu sau này bạn reset `lab-ha-4` hay `lab-ha-5`, **dữ liệu trong cụm etcd không
mất đi đâu cả**. Đó là một mặt khác của tính tách rời.

### B2.2. Gỡ hai worker theo đúng thứ tự của bài 02

Bài 02 quy định thứ tự: `kubectl drain` → `kubeadm reset` **trên chính node đó** →
`kubectl delete node`.

Chạy trên **`lab-ha-1`** — bước drain:

```bash
for n in "$N4" "$N5"; do
  kubectl drain "$n" --delete-emptydir-data --ignore-daemonsets --timeout=300s
done
kubectl get nodes | tee "$EV/b2-nodes-drained.txt"

DRAINED="$(grep -c 'SchedulingDisabled' "$EV/b2-nodes-drained.txt")"
test "$DRAINED" -eq 2 && echo 'PASS: ca hai worker da cordon va drain xong'
```

**PASS:** dòng `PASS: ca hai worker da cordon va drain xong` xuất hiện. `--ignore-daemonsets` là bắt
buộc vì Pod DaemonSet không trục xuất được; `--delete-emptydir-data` cần cho Pod hạ tầng dùng
`emptyDir`. **Không dùng `--force`** — không Pod trần nào được phép tồn tại ở mốc `8x-ha-stacked`.

Chạy **trên `lab-ha-4`**, rồi lặp lại y hệt **trên `lab-ha-5`**:

```bash
sudo kubeadm reset -f | tee ~/reset.log
```

Quay lại **`lab-ha-1`**:

```bash
for n in "$N4" "$N5"; do kubectl delete node "$n"; done
kubectl get nodes | tee "$EV/b2-nodes-3cp.txt"

LEFT="$(kubectl get nodes --no-headers | wc -l)"
echo "node con lai: $LEFT"
test "$LEFT" -eq 3 && echo 'PASS: chi con ba control plane node'
```

**PASS:** dòng `PASS: chi con ba control plane node` xuất hiện.

### B2.3. Reset hai control plane node thứ hai và thứ ba

Thứ tự ngược: bỏ node xa trước, giữ `lab-ha-1` đến cuối cùng vì nó là nơi bạn đang gõ lệnh và là nơi
API server còn sống để nhận lệnh `remove-etcd-member`.

Chạy trên **`lab-ha-1`** — drain `lab-ha-3`:

```bash
kubectl drain "$N3" --delete-emptydir-data --ignore-daemonsets --timeout=300s
```

Chạy **trên `lab-ha-3`**:

```bash
sudo kubeadm reset -f | tee ~/reset.log
grep -i 'etcd' ~/reset.log
```

Quay lại **`lab-ha-1`**:

```bash
kubectl delete node "$N3"

MEM="$(kubectl -n kube-system get pod -l component=etcd --no-headers | wc -l)"
echo "Pod etcd con lai: $MEM"
test "$MEM" -eq 2 && echo 'PASS: cum etcd con 2 member, quorum van du (2/2)'
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: API van phuc vu'
```

**PASS:** hai dòng `PASS:` xuất hiện.

**Ý nghĩa:** `kubeadm reset` trên một control plane node chạy phase `remove-etcd-member` — nó **gỡ
member cục bộ ra khỏi cụm etcd** trước khi dọn node. Nhờ vậy cụm đi từ 3 member xuống 2 member một
cách có trật tự thay vì mất một member đột ngột. Đây cũng là lý do bạn **không được tắt máy thay cho
reset**: tắt máy để lại một member chết trong danh sách, và cụm 3 member mất 1 vẫn đủ quorum nhưng
cụm 2 member mất 1 thì không.

Lặp lại y hệt cho `lab-ha-2`. Chạy trên **`lab-ha-1`**:

```bash
kubectl drain "$N2" --delete-emptydir-data --ignore-daemonsets --timeout=300s
```

Chạy **trên `lab-ha-2`**:

```bash
sudo kubeadm reset -f | tee ~/reset.log
```

Quay lại **`lab-ha-1`**:

```bash
kubectl delete node "$N2"

LEFT="$(kubectl get nodes --no-headers | wc -l)"
echo "node con lai: $LEFT"
test "$LEFT" -eq 1 && echo 'PASS: chi con lab-ha-1'
```

**PASS:** dòng `PASS: chi con lab-ha-1` xuất hiện.

### B2.4. Reset control plane node cuối cùng

Chạy **trên `lab-ha-1`**. Sau lệnh này không còn API server nào; mọi lệnh `kubectl` sẽ báo lỗi kết
nối, và đó là điều đúng:

```bash
sudo kubeadm reset -f | tee ~/reset.log

kubectl --request-timeout=10s get nodes 2>&1 | tee "$EV/b2-kubectl-sau-reset.txt"
grep -qiE 'refused|no such host|Unable to connect|timeout' "$EV/b2-kubectl-sau-reset.txt" \
  && echo 'PASS: khong con API server — dung nhu mong doi'
```

**PASS:** dòng `PASS: khong con API server …` xuất hiện.

**Ý nghĩa:** thông báo lỗi này đến từ `$HOME/.kube/config` — file **vẫn còn nguyên** dù cluster đã
biến mất. Đó là bằng chứng sống cho dòng "kubeconfig của user" trong bảng B2.1, và cũng là nguyên
nhân của mục *Lỗi certificate TLS* trong bài [09](../09-troubleshooting-kubeadm-vi.md): một
kubeconfig cũ trỏ vào một cluster mới sẽ báo `x509: certificate signed by unknown authority`. B2.6
xóa nó.

### B2.5. Gate "reset đã xóa những gì"

Chạy trên **`lab-ha-1`**. Khối này kiểm cả năm node qua `ssh`, nên nó phải chạy sau khi B0.3 đã dựng
SSH key:

```bash
mkdir -p ~/lab-evidence/8c; EV=~/lab-evidence/8c
ALL='lab-ha-1 lab-ha-2 lab-ha-3 lab-ha-4 lab-ha-5'

OKR=0
: > "$EV/b2-reset-xoa.txt"
for n in $ALL; do
  M="$(ssh -o BatchMode=yes "$n" 'sudo ls -A /etc/kubernetes/manifests 2>/dev/null | wc -l')"
  A="$(ssh -o BatchMode=yes "$n" 'sudo test -e /etc/kubernetes/admin.conf && echo 1 || echo 0')"
  P="$(ssh -o BatchMode=yes "$n" 'sudo test -d /etc/kubernetes/pki && echo 1 || echo 0')"
  K="$(ssh -o BatchMode=yes "$n" 'sudo test -e /etc/kubernetes/kubelet.conf && echo 1 || echo 0')"
  echo "$n -> manifests=$M admin.conf=$A pki=$P kubelet.conf=$K" | tee -a "$EV/b2-reset-xoa.txt"
  test "$M" -eq 0 && test "$A" -eq 0 && test "$P" -eq 0 && test "$K" -eq 0 \
    && OKR=$(( OKR + 1 ))
done

echo "node sach: $OKR/5"
test "$OKR" -eq 5 \
  && echo 'PASS: reset da xoa manifest, admin.conf, kubelet.conf va thu muc pki tren ca nam node'
```

**PASS:** dòng `PASS: reset da xoa …` xuất hiện. Node nào chưa sạch thì xem
[mục 4](#4-troubleshooting-của-lab-này) — trường hợp hay gặp nhất là `kubeadm reset` bị treo ở bước
xóa container, đúng tình huống bài [09](../09-troubleshooting-kubeadm-vi.md) mô tả.

Dữ liệu etcd của ba control plane cũ — đây là thứ **bắt buộc** phải sạch, vì B7 dựng cụm etcd mới
với `initial-cluster-state: new`:

```bash
OKE=0
for n in "$N1" "$N2" "$N3"; do
  C="$(ssh -o BatchMode=yes "$n" 'sudo ls -A /var/lib/etcd 2>/dev/null | wc -l')"
  echo "$n -> so muc trong /var/lib/etcd: $C"
  test "$C" -eq 0 && OKE=$(( OKE + 1 ))
done
echo "host etcd sach du lieu: $OKE/3"

test "$OKE" -eq 3 && echo 'PASS: /var/lib/etcd rong tren ca ba host — reset da xoa du lieu etcd cuc bo'
```

**PASS:** dòng `PASS: /var/lib/etcd rong …` xuất hiện.

Nếu còn dữ liệu — reset là best-effort, nên khả năng đó có thật — xóa tay rồi chạy lại gate. Chỉ
chạy khi con số ở trên khác 0:

```bash
for n in "$N1" "$N2" "$N3"; do ssh "$n" 'sudo rm -rf /var/lib/etcd/*'; done
```

> **Không mang lệnh này sang cluster thật.** Ở đây `/var/lib/etcd` chứa dữ liệu của một cụm bạn vừa
> cố ý phá. Trên cluster đang chạy, đó là **toàn bộ trạng thái** của Kubernetes.

### B2.6. Gate "reset KHÔNG xóa những gì" — và dọn tay từng thứ

Ba thứ dưới đây phải **còn** sau reset. Gate xác nhận chúng còn, rồi mới dọn.

Chạy trên **`lab-ha-1`** — thứ nhất, cấu hình CNI:

```bash
CNI_LEFT=0
for n in $ALL; do
  C="$(ssh -o BatchMode=yes "$n" 'sudo ls -A /etc/cni/net.d 2>/dev/null | wc -l')"
  echo "$n -> file trong /etc/cni/net.d: $C"
  test "$C" -gt 0 && CNI_LEFT=$(( CNI_LEFT + 1 ))
done
echo "node con cau hinh CNI: $CNI_LEFT/5"

test "$CNI_LEFT" -ge 1 \
  && echo 'PASS: reset KHONG don /etc/cni/net.d — dung nhu tai lieu ghi'
```

**PASS:** dòng `PASS: reset KHONG don /etc/cni/net.d …` xuất hiện. Dọn tay:

```bash
for n in $ALL; do ssh "$n" 'sudo rm -rf /etc/cni/net.d'; done

CNI_AFTER=0
for n in $ALL; do
  ssh -o BatchMode=yes "$n" 'test -d /etc/cni/net.d' && CNI_AFTER=$(( CNI_AFTER + 1 ))
done
echo "node con thu muc /etc/cni/net.d: $CNI_AFTER/5"
test "$CNI_AFTER" -eq 0 && echo 'PASS: da don cau hinh CNI cu tren ca nam node'
```

**PASS:** dòng `PASS: da don cau hinh CNI cu …` xuất hiện. Bỏ bước này thì Flannel của cluster mới
sẽ đọc phải file `.conflist` của Flannel cũ và Pod có thể nhận IP thuộc dải của cụm đã chết.

Thứ hai, kubeconfig của user. Chạy trên **`lab-ha-1`**:

```bash
KC_LEFT=0
for n in "$N1" "$N2" "$N3"; do
  ssh -o BatchMode=yes "$n" 'test -e "$HOME/.kube/config"' && KC_LEFT=$(( KC_LEFT + 1 ))
done
echo "node con \$HOME/.kube/config: $KC_LEFT/3"

test "$KC_LEFT" -ge 1 \
  && echo 'PASS: reset KHONG don $HOME/.kube — dung nhu tai lieu ghi'

for n in "$N1" "$N2" "$N3"; do ssh "$n" 'rm -rf "$HOME/.kube"'; done

KC_AFTER=0
for n in "$N1" "$N2" "$N3"; do
  ssh -o BatchMode=yes "$n" 'test -e "$HOME/.kube/config"' && KC_AFTER=$(( KC_AFTER + 1 ))
done
test "$KC_AFTER" -eq 0 && echo 'PASS: da xoa kubeconfig cu cua user tren ba control plane cu'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện. Bài 02 còn cho một cách gọn hơn khi bạn chỉ muốn
bỏ tham chiếu tới một cluster mà giữ các cluster khác: `kubectl config delete-cluster`. Ở đây file
chỉ có một cluster nên xóa cả file là đủ.

Thứ ba, quy tắc mạng do kube-proxy để lại. Chạy trên **`lab-ha-1`**:

```bash
IPT_LEFT=0
for n in $ALL; do
  C="$(ssh -o BatchMode=yes "$n" 'sudo iptables-save 2>/dev/null | grep -c "KUBE-"')"
  echo "$n -> dong iptables con chua KUBE-: $C"
  test "$C" -gt 0 && IPT_LEFT=$(( IPT_LEFT + 1 ))
done
echo "node con luat kube-proxy: $IPT_LEFT/5"

test "$IPT_LEFT" -ge 1 \
  && echo 'PASS: reset KHONG dat lai iptables — dung cau bai 02 viet'
```

**PASS:** dòng `PASS: reset KHONG dat lai iptables …` xuất hiện. Dọn bằng đúng hai lệnh bài 02 đưa:

```bash
for n in $ALL; do
  ssh "$n" 'sudo sh -c "iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X"'
  ssh "$n" 'command -v ipvsadm >/dev/null 2>&1 && sudo ipvsadm -C || echo "khong co ipvsadm, bo qua"'
done

IPT_AFTER=0
for n in $ALL; do
  C="$(ssh -o BatchMode=yes "$n" 'sudo iptables-save 2>/dev/null | grep -c "KUBE-"')"
  echo "$n -> con lai: $C"
  test "$C" -eq 0 && IPT_AFTER=$(( IPT_AFTER + 1 ))
done
echo "node da sach luat: $IPT_AFTER/5"
test "$IPT_AFTER" -eq 5 && echo 'PASS: khong node nao con chuoi KUBE- cua kube-proxy'
```

**PASS:** dòng `PASS: khong node nao con chuoi KUBE- …` xuất hiện.

> Hai lệnh trên **xóa toàn bộ** quy tắc iptables của máy, không riêng phần Kubernetes. Chấp nhận
> được ở đây vì [A4.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a44-firewall-cho-mạng-lab-cô-lập) đã
> tắt UFW trên mạng lab cô lập, nên không có luật nào khác cần giữ. **Đừng chạy chúng trên máy có
> firewall thật.**

Cuối cùng, không container nào của Kubernetes còn sống. Chạy trên **`lab-ha-1`**:

```bash
POD_LEFT=0
for n in $ALL; do
  C=0
  for i in $(seq 1 24); do
    C="$(ssh -o BatchMode=yes "$n" 'sudo crictl pods -q 2>/dev/null | wc -l')"
    test "$C" -eq 0 && break
    sleep 5
  done
  echo "$n -> sandbox con lai: $C"
  test "$C" -eq 0 && POD_LEFT=$(( POD_LEFT + 1 ))
done
echo "node khong con sandbox: $POD_LEFT/5"

test "$POD_LEFT" -eq 5 && echo 'PASS: container runtime khong con sandbox nao cua Kubernetes'
```

**PASS:** dòng `PASS: container runtime khong con sandbox nao …` xuất hiện. Vòng lặp có số vòng tối
đa vì thời gian dọn container **phụ thuộc cấu hình và phần cứng host**; hết vòng mà vẫn còn sandbox
thì đó đúng là tình huống *"kubeadm bị treo khi xóa các container được quản lý"* của bài
[09](../09-troubleshooting-kubeadm-vi.md) — xem [mục 4](#4-troubleshooting-của-lab-này).

### B2.7. Fallback — khi gate reset không đạt

Không cố sửa một node reset dở. Nếu bất kỳ gate nào của B2.5 hoặc B2.6 còn fail sau một lần thử lại,
chuyển sang mốc `8x-vm-ready` — mốc "VM sạch đã cài kubeadm, chưa có cluster" mà Lab 8b chụp.

Chạy trên **máy host**, PowerShell, dùng lại `$vmrun` và `$vmxHA`:

```powershell
foreach ($f in $vmxHA) {
  $snaps = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($snaps -ccontains '8x-vm-ready') { "PASS: $f co 8x-vm-ready" } else { "FAIL: $f KHONG co" }
}
```

**PASS:** sáu dòng `PASS:`. Còn dòng `FAIL:` thì fallback này không dùng được — quay về
`8x-ha-stacked` và chạy lại từ B0.

Tắt rồi khôi phục cả sáu VM, sau đó bật lại theo thứ tự mảng:

```powershell
foreach ($f in $vmxHA) { & $vmrun -T ws stop $f soft }
foreach ($f in $vmxHA) { & $vmrun -T ws revertToSnapshot $f '8x-vm-ready' }
foreach ($f in $vmxHA) { & $vmrun -T ws start $f }
& $vmrun -T ws list
```

**PASS:** `vmrun list` liệt kê đủ sáu máy HA. Khôi phục là thao tác **trên cả sáu VM cùng một mốc** —
không bao giờ khôi phục riêng một máy, đúng ghi chú cuối
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).

Gate riêng cho đường fallback. Chạy trên **`lab-ha-1`** sau khi sáu máy đã lên:

```bash
mkdir -p ~/lab-work/8c ~/lab-evidence/8c
WK=~/lab-work/8c; EV=~/lab-evidence/8c
ALL='lab-ha-1 lab-ha-2 lab-ha-3 lab-ha-4 lab-ha-5'

# Moc 8x-vm-ready co truoc B0.3 nen khoa SSH da bien mat cung snapshot — dung lai truoc.
test -f ~/.ssh/id_ed25519 || ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
for n in $ALL; do ssh-copy-id -o StrictHostKeyChecking=accept-new "$n"; done

OKF=0
for n in $ALL; do
  R="$(ssh -o BatchMode=yes "$n" '
    KA=$(kubeadm version -o short 2>/dev/null)
    CT=$(systemctl is-active containerd)
    M=$(sudo ls -A /etc/kubernetes/manifests 2>/dev/null | wc -l)
    K=$(test -e "$HOME/.kube/config" && echo 1 || echo 0)
    E=$(sudo ls -A /var/lib/etcd 2>/dev/null | wc -l)
    echo "$KA $CT $M $K $E"')"
  echo "$n -> $R"
  case "$R" in
    v*' active 0 0 0') OKF=$(( OKF + 1 )) ;;
  esac
done
echo "node dat moc 8x-vm-ready: $OKF/5"

test "$OKF" -eq 5 \
  && echo 'PASS: fallback thanh cong — kubeadm da cai, containerd chay, khong con dau vet cluster'
```

**PASS:** dòng `PASS: fallback thanh cong …` xuất hiện. Sau đó chạy lại **B0.3 và B0.4** (workspace,
SSH key, biến), **bỏ qua B0.2, B1 và phần còn lại của B2** — không còn cụm stacked để đọc, và bằng
chứng của B1 bạn đã ghi ở `$EV/b0-truoc-stacked.txt` từ trước. Đi thẳng tới B3.

---

## B3. Bài 08 — đổi backend của load balancer

**Mục đích:** thực hành mục *Tạo bộ cân bằng tải cho kube-apiserver* — phần bài
[08](../08-high-availability-vi.md) ghi rõ là **bước đầu tiên và chung cho cả hai phương pháp**. Ở
Lab 8b, backend là ba máy `.231`, `.232`, `.233`. Ba máy đó sắp thành etcd node và sẽ không nghe
`6443` nữa, nên backend phải trỏ sang `.234` và `.235` **trước** khi `kubeadm init` chạy.

### B3.1. Ghi lại cấu hình cũ rồi thay backend

Chạy trên **`lab-ha-lb`**:

```bash
mkdir -p ~/lab-evidence/8c
EV=~/lab-evidence/8c

haproxy -v | head -n 1 | tee "$EV/b3-haproxy-version.txt"
sudo cp -a /etc/haproxy/haproxy.cfg "$EV/b3-haproxy-cfg-8b.bak"
grep -nE 'server|bind' /etc/haproxy/haproxy.cfg | tee "$EV/b3-backend-cu.txt"
```

**PASS:** `haproxy -v` in ra một dòng version, và `$EV/b3-backend-cu.txt` cho thấy backend đang trỏ
tới `192.168.100.231/232/233`. Nếu `haproxy` không có trên máy thì Lab 8b chưa hoàn tất — cài lại từ
gói phân phối của Ubuntu (`sudo apt-get install -y haproxy`) và **không ghim version**: HAProxy ở
đây chỉ chuyển tiếp TCP, nó không phải thành phần Kubernetes nên không nằm trong
[bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa).

Ghi cấu hình mới. Khối `global`/`defaults` dưới đây cố ý tối giản để file này đứng độc lập được; nếu
cấu hình của Lab 8b có thêm phần bạn muốn giữ, chỉ thay hai khối `frontend` và `backend`:

```bash
sudo tee /etc/haproxy/haproxy.cfg >/dev/null <<'EOF'
global
    daemon
    maxconn 4096

defaults
    mode                tcp
    timeout connect     10s
    timeout client      4h
    timeout server      4h
    retries             3

frontend kube-apiserver
    bind *:6443
    default_backend kube-apiserver-backend

backend kube-apiserver-backend
    option tcp-check
    balance roundrobin
    server cp1 192.168.100.234:6443 check inter 2s fall 3 rise 2
    server cp2 192.168.100.235:6443 check inter 2s fall 3 rise 2
EOF

sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
systemctl is-active haproxy
```

**PASS:** `haproxy -c` in `Configuration file is valid`; `systemctl is-active haproxy` trả `active`.

**Ý nghĩa:** `mode tcp` cộng `option tcp-check` là đúng thứ bài 08 mô tả — load balancer **chuyển
tiếp TCP** và health check là **kiểm tra TCP trên cổng mà kube-apiserver lắng nghe (mặc định
`:6443`)**. Không có kiểm tra HTTP nào ở đây, vì `/readyz` của apiserver đòi certificate client và
load balancer không có certificate đó.

### B3.2. Gate: LB nghe đúng cổng, trỏ đúng hai backend mới

Chạy trên **`lab-ha-lb`**:

```bash
sudo ss -lntp | grep ':6443' | tee "$EV/b3-listen.txt"
grep -c '192.168.100.234:6443\|192.168.100.235:6443' /etc/haproxy/haproxy.cfg
grep -c '192.168.100.231:6443\|192.168.100.232:6443\|192.168.100.233:6443' /etc/haproxy/haproxy.cfg

test -s "$EV/b3-listen.txt" && echo 'PASS: LB dang nghe 6443'
test "$(grep -c '192.168.100.23[45]:6443' /etc/haproxy/haproxy.cfg)" -eq 2 \
  && echo 'PASS: backend tro dung hai control plane moi'
test "$(grep -c '192.168.100.23[123]:6443' /etc/haproxy/haproxy.cfg)" -eq 0 \
  && echo 'PASS: khong con backend nao tro toi ba host etcd'
```

**PASS:** ba dòng `PASS:` của bước này xuất hiện. Dòng thứ ba là gate quan trọng nhất: bỏ sót một
backend cũ thì load balancer sẽ round-robin request tới một máy etcd và bạn nhận về lỗi TLS khó hiểu
ngay giữa `kubeadm init`.

Kiểm đường đi từ phía control plane tương lai. Chạy trên **`lab-ha-4`**:

```bash
timeout 3 bash -c 'echo > /dev/tcp/192.168.100.230/6443' 2>/dev/null
RC=$?
echo "ma thoat: $RC"
case "$RC" in
  124) echo 'FAIL: TIMEOUT — lab-ha-4 khong toi duoc load balancer' ;;
  *)   echo 'PASS: khong timeout — duong toi LB thong; ket noi bi dong la binh thuong vi chua co apiserver' ;;
esac
```

**PASS:** dòng bắt đầu bằng `PASS: khong timeout …` xuất hiện.

**Ý nghĩa:** bài 08 dạy đọc chính xác kết quả của phép thử này bằng `nc -zv -w 2 <LB_IP> <PORT>`:
**`connection refused` là điều được mong đợi** vì API server chưa chạy, còn **timeout nghĩa là load
balancer không giao tiếp được với node control plane**. Lab dùng `/dev/tcp` của bash thay `nc` vì
`nc` không chắc có sẵn trên bản cài minimal, và mã thoát `124` của `timeout` cho ta đúng ranh giới
đó ở dạng kiểm chứng được. Bằng chứng thật rằng LB hoạt động sẽ đến ở B9.5, khi `curl` qua
`https://192.168.100.230:6443/version` trả về JSON.

---

## B4. Bài 07 bước 1 — kubelet làm trình quản lý dịch vụ cho etcd

**Mục đích:** ba host `lab-ha-1`, `lab-ha-2`, `lab-ha-3` phải chạy etcd **trước khi có bất kỳ cluster
Kubernetes nào**. Bài [07](../07-setup-ha-etcd-with-kubeadm-vi.md) giải bài toán đó bằng cách trưng
dụng kubelet làm trình quản lý dịch vụ cục bộ cho static Pod.

### B4.1. Ghi đè unit của kubelet trên cả ba host

> **Chạy nguyên mục B4 trên cả ba host etcd**: `lab-ha-1`, rồi `lab-ha-2`, rồi `lab-ha-3`. Lặp lại y
> hệt, không đổi gì giữa các máy — file này giống nhau trên cả ba.

```bash
mkdir -p ~/lab-work/8c ~/lab-evidence/8c
WK=~/lab-work/8c; EV=~/lab-evidence/8c
ME="$(hostname -s)"

sudo mkdir -p /etc/systemd/system/kubelet.service.d /etc/kubernetes/manifests

sudo tee /etc/systemd/system/kubelet.service.d/kubelet.conf >/dev/null <<'EOF'
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: false
authorization:
  mode: AlwaysAllow
cgroupDriver: systemd
address: 127.0.0.1
containerRuntimeEndpoint: unix:///var/run/containerd/containerd.sock
staticPodPath: /etc/kubernetes/manifests
EOF

sudo tee /etc/systemd/system/kubelet.service.d/20-etcd-service-manager.conf >/dev/null <<'EOF'
[Service]
ExecStart=
ExecStart=/usr/bin/kubelet --config=/etc/systemd/system/kubelet.service.d/kubelet.conf
Restart=always
EOF

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Hai giá trị trong `kubelet.conf` bám vào baseline, **không** được đổi:

- `cgroupDriver: systemd` phải khớp cgroup driver của containerd. Baseline đặt
  `SystemdCgroup = true` ở
  [A4.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a42-cài-containerd-và-runc-đúng-version); đổi một
  bên là kubelet và runtime đá nhau.
- `containerRuntimeEndpoint: unix:///var/run/containerd/containerd.sock` là giá trị bài 07 viết.
  Trên Ubuntu, `/var/run` là symlink tới `/run`, nên nó trỏ về đúng socket mà
  [`/etc/crictl.yaml` của A4.3](LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl)
  khai. Giữ nguyên chuỗi của bài để dễ đối chiếu.

### B4.2. Gate: kubelet chạy bằng unit mới và không nói chuyện với API server nào

Chạy trên **từng host etcd**:

```bash
ACT=''
for i in $(seq 1 24); do
  ACT="$(systemctl is-active kubelet)"
  test "$ACT" = 'active' && break
  sleep 5
done
echo "kubelet tren $ME: $ACT"

systemctl show kubelet -p ExecStart | tee "$EV/b4-execstart-$ME.txt"
systemctl cat kubelet | grep -n 'etcd-service-manager' | tee "$EV/b4-unit-$ME.txt"

test "$ACT" = 'active' && echo 'PASS: kubelet dang chay'
grep -q 'kubelet.service.d/kubelet.conf' "$EV/b4-execstart-$ME.txt" \
  && echo 'PASS: ExecStart da tro vao KubeletConfiguration rieng cua etcd'
test -s "$EV/b4-unit-$ME.txt" \
  && echo 'PASS: unit 20-etcd-service-manager.conf dang duoc ap dung'
```

**PASS:** ba dòng `PASS:` xuất hiện trên **cả ba host**. Vòng lặp có số vòng tối đa vì thời gian
kubelet lên **phụ thuộc cấu hình và phần cứng host**.

Chứng minh vế còn lại — kubelet này **không** thuộc cluster nào:

```bash
test ! -e /etc/kubernetes/kubelet.conf \
  && echo 'PASS: khong co kubelet.conf — kubelet khong dang ky voi API server nao'
grep -q 'address: 127.0.0.1' /etc/systemd/system/kubelet.service.d/kubelet.conf \
  && echo 'PASS: kubelet chi nghe tren loopback'
grep -q 'mode: AlwaysAllow' /etc/systemd/system/kubelet.service.d/kubelet.conf \
  && echo 'PASS: authorization.mode la AlwaysAllow — khong co API server de hoi'
test -d /etc/kubernetes/manifests \
  && echo 'PASS: staticPodPath da san sang'
```

**PASS:** bốn dòng `PASS:` xuất hiện trên cả ba host.

**Ý nghĩa:** bài 07 giải thích vì sao phải làm chuyện lạ đời này: **etcd được tạo trước**, nên phải
tạo một unit file **có độ ưu tiên cao hơn** unit kubelet mà kubeadm cung cấp. Số `20-` ở đầu tên file
chính là độ ưu tiên: systemd nạp các file drop-in theo thứ tự chữ cái, nên `20-etcd-service-manager.conf`
đứng sau `10-kubeadm.conf` và ghi đè nó. Dòng `ExecStart=` rỗng phía trên là cú pháp systemd để
**xóa** lệnh cũ trước khi đặt lệnh mới; thiếu dòng rỗng đó thì unit có hai `ExecStart` và không khởi
động được.

Kết quả là một kubelet chạy ở chế độ độc lập: nó chỉ đọc thư mục `staticPodPath` và chạy những gì
tìm thấy ở đó. Không kubeconfig, không đăng ký Node, không ai trong Kubernetes biết ba máy này tồn
tại. Bài 07 nói thẳng điều đó ở phần *Tự kiểm tra*: chạy hết tám bước xong, **bạn vẫn chưa có một
cluster Kubernetes nào**.

---

## B5. Bài 07 bước 2 — `kubeadmcfg.yaml` cho từng member

**Mục đích:** mỗi etcd member có một file cấu hình riêng, khác nhau ở SAN và ở địa chỉ của chính nó,
nhưng **giống nhau** ở danh sách `initial-cluster`. Hiểu chỗ nào riêng và chỗ nào chung là hiểu cách
một cụm etcd tĩnh hình thành.

Chạy trên **`lab-ha-1`** — đây là `$HOST0` của bài 07 cho toàn bộ B5, B6:

```bash
WK=~/lab-work/8c; EV=~/lab-evidence/8c
mkdir -p "$WK" "$EV"

export HOST0='192.168.100.231'
export HOST1='192.168.100.232'
export HOST2='192.168.100.233'
export NAME0='lab-ha-1'
export NAME1='lab-ha-2'
export NAME2='lab-ha-3'

mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/

HOSTS=(${HOST0} ${HOST1} ${HOST2})
NAMES=(${NAME0} ${NAME1} ${NAME2})

for i in "${!HOSTS[@]}"; do
HOST=${HOSTS[$i]}
NAME=${NAMES[$i]}
cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
---
apiVersion: "kubeadm.k8s.io/v1beta4"
kind: InitConfiguration
nodeRegistration:
    name: ${NAME}
localAPIEndpoint:
    advertiseAddress: ${HOST}
---
apiVersion: "kubeadm.k8s.io/v1beta4"
kind: ClusterConfiguration
etcd:
    local:
        serverCertSANs:
        - "${HOST}"
        peerCertSANs:
        - "${HOST}"
        extraArgs:
        - name: initial-cluster
          value: ${NAMES[0]}=https://${HOSTS[0]}:2380,${NAMES[1]}=https://${HOSTS[1]}:2380,${NAMES[2]}=https://${HOSTS[2]}:2380
        - name: initial-cluster-state
          value: new
        - name: name
          value: ${NAME}
        - name: listen-peer-urls
          value: https://${HOST}:2380
        - name: listen-client-urls
          value: https://${HOST}:2379
        - name: advertise-client-urls
          value: https://${HOST}:2379
        - name: initial-advertise-peer-urls
          value: https://${HOST}:2380
EOF
done
```

Gate:

```bash
OKC=0
for H in "$HOST0" "$HOST1" "$HOST2"; do
  F="/tmp/$H/kubeadmcfg.yaml"
  SAN="$(grep -c "\"$H\"" "$F")"
  IC="$(grep -c "initial-cluster$" "$F")"
  ALL3="$(grep -c "$HOST0:2380,.*$HOST1:2380,.*$HOST2:2380" "$F")"
  STATE="$(grep -c 'value: new' "$F")"
  echo "$F -> SAN=$SAN initial-cluster=$IC du-ba-member=$ALL3 state-new=$STATE"
  test "$SAN" -eq 2 && test "$IC" -eq 1 && test "$ALL3" -eq 1 && test "$STATE" -eq 1 \
    && OKC=$(( OKC + 1 ))
done
echo "file cau hinh dung: $OKC/3"

test "$OKC" -eq 3 \
  && echo 'PASS: ba file kubeadmcfg.yaml dung SAN rieng va dung danh sach ba member chung'

cp /tmp/"$HOST0"/kubeadmcfg.yaml "$EV/b5-kubeadmcfg-host0.txt"
```

**PASS:** dòng `PASS: ba file kubeadmcfg.yaml …` xuất hiện. `SAN=2` là đúng: mỗi file khai IP của
chính nó **hai lần** — một lần trong `serverCertSANs`, một lần trong `peerCertSANs`.

**Ý nghĩa — đọc kỹ hai cột riêng và chung:**

| Trường | Riêng của từng host hay chung cả cụm | Vì sao |
| --- | --- | --- |
| `nodeRegistration.name`, `extraArgs.name` | riêng | định danh member trong cụm etcd |
| `serverCertSANs`, `peerCertSANs` | riêng | certificate của member chỉ hợp lệ cho **địa chỉ của chính nó** |
| `listen-peer-urls`, `initial-advertise-peer-urls` (`:2380`) | riêng | địa chỉ mà member này nhận và quảng bá lưu lượng **giữa các member** |
| `listen-client-urls`, `advertise-client-urls` (`:2379`) | riêng | địa chỉ mà **client** — `kube-apiserver`, `etcdctl` — kết nối vào |
| `initial-cluster` | **chung** | cả ba member phải thấy **cùng một** danh sách thì cụm mới hình thành |
| `initial-cluster-state: new` | **chung** | khai rằng đây là cụm mới, không phải member join vào cụm có sẵn |

Đây cũng là chỗ trả lời câu hỏi của bài 07 về hai port: **`2380` là port peer**, **`2379` là port
client**. Chặn `2380` thì ba member không hình thành được cụm dù `etcdctl` vẫn chạm được từng
endpoint qua `2379`. Mục *Trước khi bạn bắt đầu* của bài 07 yêu cầu **cả hai** port TCP thông giữa ba
host; [A4.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a44-firewall-cho-mạng-lab-cô-lập) đã tắt UFW nên
điều kiện đó có sẵn.

---

## B6. Bài 07 bước 3–5 — CA, certificate từng member, và phân phối

**Mục đích:** thực hành mô hình mà bài 07 gọi là "tạo tất cả các certificate trên một node và chỉ
phân phối các file **cần thiết** đến các node khác". Điểm học nằm ở chữ *cần thiết*.

### B6.1. Sinh CA của etcd

Chạy trên **`lab-ha-1`**:

```bash
sudo kubeadm init phase certs etcd-ca

sudo ls -l /etc/kubernetes/pki/etcd/ | tee "$EV/b6-ca.txt"
sudo test -f /etc/kubernetes/pki/etcd/ca.crt && echo 'PASS: co ca.crt'
sudo test -f /etc/kubernetes/pki/etcd/ca.key && echo 'PASS: co ca.key'
```

**PASS:** hai dòng `PASS:` xuất hiện. Bài 07 nói rõ kubeadm **chứa đủ cơ chế mã hóa** để làm việc
này — không cần `openssl`, `cfssl` hay công cụ ngoài nào.

### B6.2. Sinh bốn certificate cho từng member

Chạy trên **`lab-ha-1`**, đúng thứ tự của bài 07 — `HOST2` trước, `HOST1` giữa, `HOST0` cuối:

```bash
sudo kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
sudo kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
sudo kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
sudo kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
sudo cp -R /etc/kubernetes/pki /tmp/${HOST2}/
sudo find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

sudo kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
sudo kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
sudo kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
sudo kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
sudo cp -R /etc/kubernetes/pki /tmp/${HOST1}/
sudo find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

sudo kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
sudo kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
sudo kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
sudo kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml

sudo find /tmp/${HOST2} -name ca.key -type f -delete
sudo find /tmp/${HOST1} -name ca.key -type f -delete
```

Gate — `ca.key` chỉ được tồn tại đúng một chỗ:

```bash
CAK_TMP="$(sudo find /tmp/${HOST1} /tmp/${HOST2} -name ca.key -type f | wc -l)"
CAK_LOCAL="$(sudo find /etc/kubernetes/pki -name ca.key -type f | wc -l)"
echo "ca.key trong thu muc se gui di: $CAK_TMP | ca.key giu lai tren HOST0: $CAK_LOCAL"

test "$CAK_TMP" -eq 0 && echo 'PASS: khong ban sao ca.key nao sap roi lab-ha-1'
test "$CAK_LOCAL" -eq 1 && echo 'PASS: ca.key cua etcd van o lai lab-ha-1'
```

**PASS:** hai dòng `PASS:` xuất hiện.

**Ý nghĩa:** hai lệnh `find … -name ca.key -delete` không phải thao tác dọn dẹp tùy hứng. **CA key
ký được certificate mới**; ai cầm nó thì tự cấp được certificate hợp lệ cho cụm etcd này. Các member
chỉ cần certificate **đã ký** của chính chúng cộng `ca.crt` để xác minh lẫn nhau, nên `ca.key` không
có lý do gì rời `lab-ha-1`. Lệnh `find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type
f -delete` xen giữa các vòng cũng cùng một logic ngược lại: dọn certificate của host trước để vòng
sau không phát nhầm sang host sau.

### B6.3. Phân phối sang hai host còn lại

Bài 07 chạy trong shell root nên `scp` đọc được mọi file. Lab dùng `sudo` từng lệnh, nên thư mục tạm
đang thuộc `root` — trả quyền sở hữu về user trước khi gửi:

```bash
sudo chown -R "$(id -u):$(id -g)" /tmp/${HOST1} /tmp/${HOST2}
```

Gửi đi, đúng cách bài 07 mô tả. Chạy trên **`lab-ha-1`**:

```bash
USER=ubuntu
for H in "$HOST1" "$HOST2"; do
  scp -r /tmp/${H}/* ${USER}@${H}:
done
```

Chạy **trên `lab-ha-2`**, rồi lặp lại y hệt **trên `lab-ha-3`**:

```bash
test ! -e /etc/kubernetes/pki \
  && echo 'PASS: chua co /etc/kubernetes/pki — an toan de di chuyen vao' \
  || echo 'FAIL: da ton tai /etc/kubernetes/pki, quay lai B2.5'

sudo chown -R root:root ~/pki
sudo mv ~/pki /etc/kubernetes/
sudo chmod 600 /etc/kubernetes/pki/apiserver-etcd-client.key
sudo find /etc/kubernetes/pki -name '*.key' -exec chmod 600 {} \;
```

**PASS:** dòng `PASS: chua co /etc/kubernetes/pki …` xuất hiện **trước** khi bạn chạy `mv`. Thấy
`FAIL:` thì dừng: `mv` vào một thư mục đã tồn tại sẽ tạo `/etc/kubernetes/pki/pki` và mọi bước sau
hỏng theo.

### B6.4. Bước 6 của bài 07 — checklist file bắt buộc trên từng host

Chạy trên **`lab-ha-1`**:

```bash
check_host () {
  local n="$1" want_cakey="$2"
  echo "== $n"
  ssh -o BatchMode=yes "$n" 'sudo ls -1 /etc/kubernetes/pki /etc/kubernetes/pki/etcd'
  local CK
  CK="$(ssh -o BatchMode=yes "$n" 'sudo test -f /etc/kubernetes/pki/etcd/ca.key && echo 1 || echo 0')"
  local NEED
  NEED="$(ssh -o BatchMode=yes "$n" '
    c=0
    for f in /etc/kubernetes/pki/apiserver-etcd-client.crt \
             /etc/kubernetes/pki/apiserver-etcd-client.key \
             /etc/kubernetes/pki/etcd/ca.crt \
             /etc/kubernetes/pki/etcd/healthcheck-client.crt \
             /etc/kubernetes/pki/etcd/healthcheck-client.key \
             /etc/kubernetes/pki/etcd/peer.crt \
             /etc/kubernetes/pki/etcd/peer.key \
             /etc/kubernetes/pki/etcd/server.crt \
             /etc/kubernetes/pki/etcd/server.key; do
      sudo test -f "$f" && c=$(( c + 1 ))
    done
    echo "$c"')"
  echo "$n -> du file bat buoc: $NEED/9 | co ca.key: $CK (mong doi $want_cakey)"
  test "$NEED" -eq 9 && test "$CK" -eq "$want_cakey" && return 0
  return 1
}

OKH=0
check_host "$N1" 1 && OKH=$(( OKH + 1 ))
check_host "$N2" 0 && OKH=$(( OKH + 1 ))
check_host "$N3" 0 && OKH=$(( OKH + 1 ))
echo "host dat checklist: $OKH/3"

test "$OKH" -eq 3 \
  && echo 'PASS: du chin file bat buoc tren ca ba host, va chi HOST0 giu ca.key'
```

**PASS:** dòng `PASS: du chin file bat buoc …` xuất hiện. Đây chính là ba cây thư mục mà bài 07 liệt
kê ở bước 6, viết lại thành gate: `$HOST0` có **cả `ca.crt` và `ca.key`**; `$HOST1` và `$HOST2` chỉ
có `ca.crt`.

Đọc SAN để thấy vì sao mỗi host cần bộ certificate riêng. Chạy trên **`lab-ha-1`**:

```bash
for pair in "$N1 $HOST0" "$N2 $HOST1" "$N3 $HOST2"; do
  set -- $pair
  echo "== $1 (mong doi SAN chua $2)"
  ssh -o BatchMode=yes "$1" \
    "sudo openssl x509 -noout -text -in /etc/kubernetes/pki/etcd/server.crt \
       | grep -A1 'Subject Alternative Name'"
done | tee "$EV/b6-san.txt"

OKS=0
for pair in "$N1 $HOST0" "$N2 $HOST1" "$N3 $HOST2"; do
  set -- $pair
  ssh -o BatchMode=yes "$1" \
    "sudo openssl x509 -noout -text -in /etc/kubernetes/pki/etcd/server.crt" \
    | grep -A1 'Subject Alternative Name' | grep -q "IP Address:$2" \
    && OKS=$(( OKS + 1 )) || echo "FAIL: server.crt tren $1 khong co SAN $2"
done
echo "certificate co SAN dung: $OKS/3"

test "$OKS" -eq 3 \
  && echo 'PASS: moi server.crt chi hop le cho dia chi cua chinh host do'
```

**PASS:** dòng `PASS: moi server.crt chi hop le …` xuất hiện.

**Ý nghĩa:** đây là câu trả lời cho "vì sao mỗi host cần bộ certificate riêng". TLS xác minh **danh
tính của địa chỉ được kết nối tới**. `kube-apiserver` mở kết nối tới `https://192.168.100.232:2379`
và sẽ từ chối nếu certificate mà server trình ra không có `192.168.100.232` trong SAN. Ba member ba
địa chỉ nên phải ba certificate; dùng chung một certificate thì hoặc là bạn phải nhét cả ba IP vào
SAN của nó, hoặc là hai trong ba kết nối sẽ hỏng. `serverCertSANs` và `peerCertSANs` trong
`kubeadmcfg.yaml` ở B5 chính là chỗ khai điều đó, và `openssl` vừa đọc lại kết quả.

---

## B7. Bài 07 bước 7 — sinh static Pod manifest cho từng host

**Mục đích:** biến ba bộ certificate cộng ba file cấu hình thành ba etcd đang chạy, và chốt rằng
**image etcd là do kubeadm chọn**, không do bạn ghim tay.

### B7.1. Chốt image etcd trước khi sinh manifest

Chạy trên **`lab-ha-1`**:

```bash
KV="$(kubeadm version -o short)"
ETCD_IMG="$(sudo kubeadm config images list --kubernetes-version "$KV" 2>/dev/null | grep '/etcd:')"
echo "kubeadm $KV khai image etcd: $ETCD_IMG"
echo "$ETCD_IMG" | tee "$EV/b7-etcd-image-khai.txt"

test -n "$ETCD_IMG" && echo 'PASS: da lay duoc image etcd tu kubeadm, khong ghim tay'
```

**PASS:** dòng `PASS: da lay duoc image etcd tu kubeadm …` xuất hiện.

**Ý nghĩa:** bài 07 yêu cầu mỗi host "có quyền truy cập đến container image registry của Kubernetes
(`registry.k8s.io`) **hoặc liệt kê/pull image etcd cần thiết bằng `kubeadm config images list/pull`**".
Đó là cách đúng để biết version etcd: **kubeadm tự chọn bản etcd hợp với phiên bản Kubernetes**, nên
con số đó không thuộc về bạn và không có chỗ trong
[bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). `--kubernetes-version "$KV"` giữ
kubeadm khỏi dò remote và chọn một patch khác binary đang cài — cùng lý do
[A5.1 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane) giữ cờ đó.

Kéo trước image trên **cả ba host etcd** để bước sau không lẫn giữa "chưa tải xong" và "cấu hình
sai". Chạy trên **`lab-ha-1`**:

```bash
for n in "$N1" "$N2" "$N3"; do
  echo "== $n"
  ssh "$n" "sudo crictl pull '$ETCD_IMG'"
done

OKI=0
for n in "$N1" "$N2" "$N3"; do
  ssh -o BatchMode=yes "$n" "sudo crictl images -q '$ETCD_IMG'" | grep -q . \
    && OKI=$(( OKI + 1 )) || echo "FAIL: $n chua co image"
done
echo "host da co image etcd: $OKI/3"
test "$OKI" -eq 3 && echo 'PASS: ca ba host deu co image etcd kubeadm khai'
```

**PASS:** dòng `PASS: ca ba host deu co image etcd …` xuất hiện. Fail ở đây gần như luôn là DNS hoặc
egress — xử lý theo
[tầng 4 của A5.4 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a545-tầng-4--dns-trong-cluster-và-ra-internet).

### B7.2. Sinh manifest trên từng host

Chạy **trên `lab-ha-1`**:

```bash
sudo kubeadm init phase etcd local --config=/tmp/${HOST0}/kubeadmcfg.yaml
```

Chạy **trên `lab-ha-2`**, rồi **trên `lab-ha-3`** — file cấu hình của hai máy này nằm ở thư mục home
vì `scp` của B6.3 đã đặt nó ở đó:

```bash
sudo kubeadm init phase etcd local --config=$HOME/kubeadmcfg.yaml
```

Gate, chạy trên **`lab-ha-1`**:

```bash
OKM=0
for n in "$N1" "$N2" "$N3"; do
  M="$(ssh -o BatchMode=yes "$n" 'sudo test -f /etc/kubernetes/manifests/etcd.yaml && echo 1 || echo 0')"
  echo "$n -> co etcd.yaml: $M"
  test "$M" -eq 1 && OKM=$(( OKM + 1 ))
done
echo "host co static Pod manifest: $OKM/3"
test "$OKM" -eq 3 && echo 'PASS: ca ba host da co /etc/kubernetes/manifests/etcd.yaml'
```

**PASS:** dòng `PASS: ca ba host da co … etcd.yaml` xuất hiện.

### B7.3. Gate: image thật khớp image kubeadm khai, và etcd đã chạy

Chạy trên **`lab-ha-1`**:

```bash
OKT=0
for n in "$N1" "$N2" "$N3"; do
  REAL="$(ssh -o BatchMode=yes "$n" \
    "sudo grep -E '^[[:space:]]+image:' /etc/kubernetes/manifests/etcd.yaml | awk '{print \$2}'")"
  echo "$n -> image trong manifest: $REAL"
  test "$REAL" = "$ETCD_IMG" && OKT=$(( OKT + 1 )) \
    || echo "FAIL: $n dung '$REAL', kubeadm khai '$ETCD_IMG'"
done
echo "manifest dung image: $OKT/3"

test "$OKT" -eq 3 \
  && echo 'PASS: tag etcd thuc te khop dung thu kubeadm khai — khong ai ghim tay'
```

**PASS:** dòng `PASS: tag etcd thuc te khop …` xuất hiện.

Chờ ba container etcd lên. Chạy trên **`lab-ha-1`**:

```bash
OKR2=0
for n in "$N1" "$N2" "$N3"; do
  C=0
  for i in $(seq 1 36); do
    C="$(ssh -o BatchMode=yes "$n" 'sudo crictl ps --name etcd --state Running -q 2>/dev/null | wc -l')"
    test "$C" -ge 1 && break
    sleep 5
  done
  echo "$n -> container etcd dang chay: $C"
  test "$C" -ge 1 && OKR2=$(( OKR2 + 1 ))
done
echo "host co etcd chay: $OKR2/3"

test "$OKR2" -eq 3 && echo 'PASS: ca ba etcd member da chay duoi dang static Pod'
```

**PASS:** dòng `PASS: ca ba etcd member da chay …` xuất hiện. Vòng lặp có số vòng tối đa vì thời gian
etcd bầu leader và ổn định **phụ thuộc cấu hình và phần cứng host**. Hết vòng mà container không lên
thì đọc log của kubelet trước: `ssh <node> 'journalctl -u kubelet -n 80 --no-pager'`, rồi xem
[mục 4](#4-troubleshooting-của-lab-này).

Đọc luôn hai port trong manifest — đây là chỗ nhìn thấy `2379`/`2380` bằng mắt:

```bash
ssh "$N1" "sudo grep -E 'listen-(peer|client)-urls|advertise' /etc/kubernetes/manifests/etcd.yaml" \
  | tee "$EV/b7-urls-host0.txt"

grep -q "listen-peer-urls=https://$HOST0:2380" "$EV/b7-urls-host0.txt" \
  && echo 'PASS: 2380 la port giua cac member'
grep -q "listen-client-urls=https://$HOST0:2379" "$EV/b7-urls-host0.txt" \
  && echo 'PASS: 2379 la port cho client'
```

**PASS:** hai dòng `PASS:` xuất hiện.

---

## B8. Bài 07 bước 8 — kiểm sức khỏe cụm etcd, chỉ đọc

**Mục đích:** chứng minh ba member đã hợp thành **một** cụm, bằng đúng ba cờ certificate mà bài 07
đưa. Đây cũng là bước đầu tiên trong lab dùng `etcdctl`, nên đọc kỹ giới hạn.

> **Chỉ đọc.** Ba lệnh được phép trong lab này là `member list`, `endpoint health`, `endpoint
> status`. `snapshot save`, `snapshot restore`, `member add/remove`, `put`, `del` đều **cấm** — chúng
> thuộc [nợ #8](README.md#5-sổ-nợ-lab), trả ở
> [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa).

### B8.1. Đặt hàm `ectl` trên host etcd

Bài 07 nói: nếu `etcdctl` không có sẵn thì chạy nó **bên trong một container image**, làm việc trực
tiếp với container runtime chứ không qua Kubernetes. Lab làm đúng ý đó theo cách rẻ nhất: `etcdctl`
đã nằm sẵn trong image mà kubeadm chọn, và container etcd **đang chạy**, nên chỉ cần `crictl exec`
vào chính nó — không kéo thêm image nào.

Chạy trên **`lab-ha-1`**:

```bash
EPS="https://192.168.100.231:2379,https://192.168.100.232:2379,https://192.168.100.233:2379"

ectl () {
  local cid
  cid="$(sudo crictl ps --name etcd --state Running -q | head -n 1)"
  test -n "$cid" || { echo 'FAIL: khong tim thay container etcd dang chay'; return 1; }
  sudo crictl exec "$cid" etcdctl \
    --cert /etc/kubernetes/pki/etcd/peer.crt \
    --key /etc/kubernetes/pki/etcd/peer.key \
    --cacert /etc/kubernetes/pki/etcd/ca.crt \
    --endpoints "$EPS" "$@"
}

ectl version && echo 'PASS: goi duoc etcdctl trong container etcd'
```

**PASS:** dòng `PASS: goi duoc etcdctl trong container etcd` xuất hiện. Ba cờ `--cert`, `--key`,
`--cacert` là đúng bộ bài 07 đưa; đường dẫn bên trong container trùng đường dẫn trên host vì static
Pod của kubeadm mount `/etc/kubernetes/pki/etcd` vào đúng chỗ đó.

`etcdctl` của etcd 3.5 trở lên mặc định dùng API v3, nên không cần biến `ETCDCTL_API=3` mà bài 07 đặt
ở đầu lệnh. Nếu `ectl version` báo lỗi về API version, thêm biến đó theo đúng cách bài viết.

### B8.2. `member list`, `endpoint health`, `endpoint status`

Chạy trên **`lab-ha-1`**:

```bash
ectl member list | tee "$EV/b8-member-list.txt"
MEM="$(grep -c ', started,' "$EV/b8-member-list.txt")"
echo "member o trang thai started: $MEM"
test "$MEM" -eq 3 && echo 'PASS: cum etcd co du ba member'

for h in lab-ha-1 lab-ha-2 lab-ha-3; do
  grep -q "$h" "$EV/b8-member-list.txt" || echo "FAIL: thieu member $h"
done
grep -q 'lab-ha-1' "$EV/b8-member-list.txt" \
  && grep -q 'lab-ha-2' "$EV/b8-member-list.txt" \
  && grep -q 'lab-ha-3' "$EV/b8-member-list.txt" \
  && echo 'PASS: ba member dung ba ten khai o initial-cluster'
```

```bash
ectl endpoint health | tee "$EV/b8-endpoint-health.txt"
HEA="$(grep -c 'is healthy' "$EV/b8-endpoint-health.txt")"
echo "endpoint khoe: $HEA/3"
test "$HEA" -eq 3 && echo 'PASS: ca ba endpoint deu healthy'
```

```bash
ectl endpoint status --write-out=table | tee "$EV/b8-endpoint-status.txt"
LEAD="$(ectl endpoint status --write-out=fields | grep -c 'IsLeader.*true')"
echo "so member dang la leader: $LEAD"
test "$LEAD" -eq 1 && echo 'PASS: dung mot leader trong cum'
```

**PASS:** bốn dòng `PASS:` của B8.2 xuất hiện. Nếu `--write-out=fields` không được hỗ trợ ở bản
etcdctl trong image của bạn, đọc cột `IS LEADER` trong bảng của lệnh ngay trên và đếm bằng mắt — kết
luận phải giống nhau: **đúng một** `true`.

### B8.3. Chốt lại: vẫn chưa có Kubernetes nào

Chạy trên **`lab-ha-1`**:

```bash
test ! -e /etc/kubernetes/admin.conf \
  && echo 'PASS: khong co admin.conf — chua co cluster Kubernetes'
test ! -e /etc/kubernetes/manifests/kube-apiserver.yaml \
  && echo 'PASS: khong co kube-apiserver.yaml tren host etcd'
sudo crictl ps --state Running -o table | tee "$EV/b8-containers-host0.txt"

NON_ETCD="$(sudo crictl ps --state Running -q | wc -l)"
echo "container dang chay tren lab-ha-1: $NON_ETCD"
test "$NON_ETCD" -eq 1 && echo 'PASS: dung mot container — chi etcd, khong co thanh phan control plane nao'
```

**PASS:** ba dòng `PASS:` xuất hiện.

**Ý nghĩa:** đây là câu chốt của bài 07. Bạn vừa dựng xong **một cụm etcd ba thành viên chạy dưới
dạng static Pod do một kubelet quản lý**, và **chưa có API server, chưa có cluster Kubernetes**. Ba
máy này sẽ **không bao giờ** xuất hiện trong `kubectl get nodes` của cluster sắp dựng — chúng không
join, không có kubelet.conf, không có danh tính trong Kubernetes. Đó chính xác là điều bài
[06](../06-ha-topology-vi.md) gọi là etcd "nằm **bên ngoài** cluster hình thành từ các node đang
chạy các thành phần control plane".

Ghi bằng chứng để B13 đối chiếu:

```bash
{
  echo "=== $(date -Is) — cum etcd external, truoc khi dung control plane ==="
  ectl member list
  ectl endpoint health
  ectl endpoint status --write-out=table
} | tee "$EV/b8-etcd-truoc-control-plane.txt"

test -s "$EV/b8-etcd-truoc-control-plane.txt" && echo 'PASS: da ghi bang chung cum etcd'
```

**PASS:** dòng `PASS: da ghi bang chung cum etcd` xuất hiện.

---

## B9. Bài 08 — control plane đầu tiên với `etcd.external`

**Mục đích:** đây là mục *Các node etcd bên ngoài* của bài [08](../08-high-availability-vi.md). Bài
nói việc thiết lập cluster với etcd ngoài "tương tự với quy trình dùng cho stacked etcd, **ngoại
trừ** việc bạn cần thiết lập etcd trước, và bạn cần truyền thông tin etcd vào file cấu hình kubeadm".
Ba mục B trước đã làm vế thứ nhất; B9 làm vế thứ hai.

### B9.1. Chép ba file certificate sang control plane đầu tiên

Bài 08 liệt kê đúng ba file phải chép **từ bất kỳ node etcd nào** sang control plane node đầu tiên.
Chạy trên **`lab-ha-1`**:

```bash
CONTROL_PLANE="ubuntu@192.168.100.234"
scp /etc/kubernetes/pki/etcd/ca.crt "${CONTROL_PLANE}":
sudo cat /etc/kubernetes/pki/apiserver-etcd-client.crt > /tmp/apiserver-etcd-client.crt
sudo cat /etc/kubernetes/pki/apiserver-etcd-client.key > /tmp/apiserver-etcd-client.key
chmod 600 /tmp/apiserver-etcd-client.key
scp /tmp/apiserver-etcd-client.crt /tmp/apiserver-etcd-client.key "${CONTROL_PLANE}":
rm -f /tmp/apiserver-etcd-client.crt /tmp/apiserver-etcd-client.key
```

Hai lệnh `sudo cat > /tmp/…` thay cho `scp /etc/kubernetes/pki/…` của bài 08 vì bài chạy trong shell
root còn lab dùng `sudo` từng lệnh: `scp` chạy bằng user thường không đọc được file khóa dưới
`/etc/kubernetes`. Nội dung file không đổi; `ca.crt` đọc được nên `scp` thẳng.

Chạy **trên `lab-ha-4`** — đặt file vào đúng chỗ và khóa quyền:

```bash
mkdir -p ~/lab-work/8c ~/lab-evidence/8c
WK=~/lab-work/8c; EV=~/lab-evidence/8c

sudo mkdir -p /etc/kubernetes/pki/etcd
sudo mv ~/ca.crt /etc/kubernetes/pki/etcd/ca.crt
sudo mv ~/apiserver-etcd-client.crt /etc/kubernetes/pki/apiserver-etcd-client.crt
sudo mv ~/apiserver-etcd-client.key /etc/kubernetes/pki/apiserver-etcd-client.key
sudo chown -R root:root /etc/kubernetes/pki
sudo chmod 600 /etc/kubernetes/pki/apiserver-etcd-client.key

sudo ls -l /etc/kubernetes/pki /etc/kubernetes/pki/etcd | tee "$EV/b9-pki.txt"

OK9=0
for f in /etc/kubernetes/pki/etcd/ca.crt \
         /etc/kubernetes/pki/apiserver-etcd-client.crt \
         /etc/kubernetes/pki/apiserver-etcd-client.key; do
  sudo test -f "$f" && OK9=$(( OK9 + 1 )) || echo "FAIL: thieu $f"
done
echo "file certificate etcd tren control plane: $OK9/3"

test "$OK9" -eq 3 && echo 'PASS: du ba file certificate ma bai 08 yeu cau'
sudo test -f /etc/kubernetes/pki/etcd/ca.key \
  && echo 'FAIL: ca.key cua etcd KHONG duoc co mat o day' \
  || echo 'PASS: khong co ca.key — control plane khong ky duoc certificate etcd moi'
```

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

**Ý nghĩa:** ba file này là toàn bộ những gì `kube-apiserver` cần để nói chuyện với cụm etcd ngoài:
`ca.crt` để **xác minh** certificate mà etcd trình ra, còn cặp `apiserver-etcd-client.crt/.key` là
**danh tính client** của apiserver khi kết nối vào port `2379`. Không có `ca.key` ở đây là đúng: một
client không cần khả năng ký certificate mới.

### B9.2. Đặt biến và viết `kubeadm-config.yaml`

Chạy trên **`lab-ha-4`**. Điền lại ba giá trị từ
[bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa), giống B0.4 — máy này là
shell khác nên biến của `lab-ha-1` không có ở đây:

```bash
K8S_VER='<dien Kubernetes control plane tu A1.3>'
FLANNEL_VER='<dien Flannel manifest release tu A1.3>'
POD_CIDR='<dien Pod CIDR tu A1.3>'

LB='192.168.100.230'
E1='192.168.100.231'; E2='192.168.100.232'; E3='192.168.100.233'
CP1='192.168.100.234'; CP2='192.168.100.235'
N4='lab-ha-4'; N5='lab-ha-5'

test "$(kubeadm version -o short)" = "$K8S_VER" \
  && echo "PASS: kubeadm tren $(hostname -s) dung version cua A1.3" \
  || echo "FAIL: kubeadm la $(kubeadm version -o short), A1.3 ghi $K8S_VER"
```

**PASS:** dòng `PASS: kubeadm tren lab-ha-4 dung version cua A1.3` xuất hiện. Placeholder chưa điền
sẽ làm gate này FAIL ngay — đó là mục đích của nó.

```bash
cat > "$WK/kubeadm-config.yaml" <<EOF
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: ${K8S_VER}
controlPlaneEndpoint: "${LB}:6443"
networking:
  podSubnet: ${POD_CIDR}
etcd:
  external:
    endpoints:
      - https://${E1}:2379
      - https://${E2}:2379
      - https://${E3}:2379
    caFile: /etc/kubernetes/pki/etcd/ca.crt
    certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
    keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
EOF

cat "$WK/kubeadm-config.yaml" | tee "$EV/b9-kubeadm-config.txt"
```

Gate trước khi chạy `init` — file cấu hình sai thì `kubeadm init` mới phát hiện ra sau khi đã đổi
trạng thái máy:

```bash
grep -q "controlPlaneEndpoint: \"${LB}:6443\"" "$WK/kubeadm-config.yaml" \
  && echo 'PASS: controlPlaneEndpoint tro toi load balancer, khong tro toi mot control plane node'
test "$(grep -c 'https://192.168.100.23[123]:2379' "$WK/kubeadm-config.yaml")" -eq 3 \
  && echo 'PASS: khai du ba endpoint etcd'
grep -q '^    external:' "$WK/kubeadm-config.yaml" \
  && echo 'PASS: dung khoi etcd.external'
grep -q '^    local:' "$WK/kubeadm-config.yaml" \
  && echo 'FAIL: con khoi etcd.local — day la cau hinh stacked' \
  || echo 'PASS: khong co khoi etcd.local'
grep -qE 'podSubnet: [0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/[0-9]+' "$WK/kubeadm-config.yaml" \
  && echo 'PASS: podSubnet da duoc dien tu A1.3'
```

**PASS:** năm dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

**Ý nghĩa — đây là toàn bộ khác biệt giữa hai nhánh của bài 08.** Bài ghi chú thẳng: "sự khác biệt
giữa stacked etcd và external etcd ở đây là thiết lập external etcd yêu cầu một file cấu hình với
các endpoint của etcd nằm dưới đối tượng `external` của `etcd`. Trong trường hợp topology stacked
etcd, điều này được quản lý tự động." Và đó cũng là lý do nhánh này **buộc** dùng `--config`: không
có cờ dòng lệnh nào của `kubeadm init` khai được khối `etcd.external`.

Một hệ quả kèm theo mà bài 08 cảnh báo: `--config` **không dùng chung được với `--certificate-key`**.
Nếu bạn muốn tự chọn certificate key thì phải đặt trường `certificateKey` trong `InitConfiguration`
và trong `JoinConfiguration: controlPlane`. Lab đi đường khác — dùng `--upload-certs` rồi lấy key ở
B10.1 — nên không vướng ràng buộc đó.

### B9.3. `kubeadm init`

Chạy **trên `lab-ha-4`**. Không ngắt lệnh này giữa chừng:

```bash
sudo kubeadm init --config "$WK/kubeadm-config.yaml" --upload-certs | tee "$EV/b9-init.log"
```

```bash
grep -q 'Your Kubernetes control-plane has initialized successfully' "$EV/b9-init.log" \
  && echo 'PASS: kubeadm init thanh cong'
grep -q 'etcd' "$EV/b9-init.log" && grep -c 'external etcd' "$EV/b9-init.log"
```

**PASS:** dòng `PASS: kubeadm init thanh cong` xuất hiện. Nếu lệnh dừng ở giai đoạn chờ control plane
khỏe, nguyên nhân thường gặp nhất là load balancer chưa đổi backend — quay lại B3.2 kiểm hai gate ở
đó trước khi làm gì khác.

Cấu hình kubeconfig cho user thường, đúng bốn lệnh của
[A5.1 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane); phần giải thích từng lệnh
nằm ở đó, không chép lại:

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
chmod 600 "$HOME/.kube/config"

kubectl config current-context
SRV="$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
echo "server trong kubeconfig: $SRV"

test "$SRV" = "https://${LB}:6443" \
  && echo 'PASS: kubeconfig tro toi load balancer, dung ControlPlaneEndpoint'
```

**PASS:** dòng `PASS: kubeconfig tro toi load balancer …` xuất hiện. Đây là bằng chứng đầu tiên rằng
`ControlPlaneEndpoint` đã "luôn khớp với địa chỉ của load balancer" như bài 08 đòi hỏi: kubeadm ghi
chính giá trị đó vào kubeconfig, và mọi node join sau này cũng nhận đúng địa chỉ đó.

### B9.4. Cài CNI

Chạy trên **`lab-ha-4`**:

```bash
kubectl apply -f \
  "https://github.com/flannel-io/flannel/releases/download/${FLANNEL_VER}/kube-flannel.yml"

kubectl -n kube-flannel rollout status daemonset/kube-flannel-ds --timeout=300s
kubectl -n kube-flannel get pods -o wide | tee "$EV/b9-flannel.txt"

NET="$(kubectl -n kube-flannel get configmap kube-flannel-cfg \
  -o jsonpath='{.data.net-conf\.json}' | tr -d ' \n')"
echo "net-conf.json: $NET"

echo "$NET" | grep -q "\"Network\":\"${POD_CIDR}\"" \
  && echo 'PASS: dai Pod network cua Flannel khop podSubnet trong kubeadm-config.yaml'
```

**PASS:** rollout thành công và dòng `PASS: dai Pod network cua Flannel khop podSubnet …` xuất hiện.
Lệch hai giá trị này là rơi đúng cảnh báo của bài [02](../02-create-cluster-kubeadm-vi.md): dải mà
kubeadm cấp phát và dải mà YAML của plugin dùng phải là một, và phải sửa **cả hai chỗ cùng lúc**.

```bash
kubectl wait --for=condition=Ready node --all --timeout=300s
kubectl get nodes -o wide | tee "$EV/b9-node-1.txt"
kubectl -n kube-system get pods -o wide | tee "$EV/b9-kube-system.txt"

test "$(kubectl get nodes --no-headers | wc -l)" -eq 1 \
  && echo 'PASS: dung mot node — ba host etcd KHONG xuat hien trong cluster'
```

**PASS:** dòng `PASS: dung mot node — ba host etcd KHONG xuat hien …` xuất hiện.

**Ý nghĩa:** đây là hình ảnh trực quan nhất của chữ *bên ngoài*. Ba máy `lab-ha-1/2/3` đang chạy etcd
phục vụ chính cluster này, nhưng chúng **không phải Node của nó** — kubelet trên đó không có
`kubelet.conf`, không đăng ký với API server nào, đúng như B4.2 đã gate. Cluster nhìn thấy etcd chỉ
qua ba địa chỉ IP trong `--etcd-servers`.

### B9.5. Gate: load balancer đã thật sự phục vụ

Chạy trên **`lab-ha-4`**:

```bash
curl -sk "https://${LB}:6443/version" | tee "$EV/b9-version-qua-lb.txt"; echo
grep -q 'gitVersion' "$EV/b9-version-qua-lb.txt" \
  && echo 'PASS: goi duoc API server qua load balancer'

test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
```

**PASS:** hai dòng `PASS:` xuất hiện. Đây là phép thử mà B3.2 hẹn lại: lúc đó `/dev/tcp` chỉ chứng
minh "không timeout", giờ `curl` chứng minh load balancer chuyển tiếp được TCP tới một apiserver
đang sống.

---

## B10. Bài 08 — join control plane thứ hai

**Mục đích:** mục *Các bước cho các node control plane còn lại*. Bài 08 nói các bước "giống như với
thiết lập stacked etcd", và cảnh báo khóa giải mã từ `--certificate-key` **mặc định hết hạn sau hai
giờ**.

### B10.1. Sinh lệnh join, không phụ thuộc thời gian

Chạy trên **`lab-ha-4`**:

```bash
CERT_KEY="$(sudo kubeadm init phase upload-certs --upload-certs | tail -n 1)"
JOIN_BASE="$(kubeadm token create --print-join-command)"
JOIN_CP="$JOIN_BASE --control-plane --certificate-key $CERT_KEY"
echo "$JOIN_CP"

echo "$JOIN_CP" | grep -q -- '--control-plane' && echo 'PASS: lenh join co co --control-plane'
echo "$JOIN_CP" | grep -q -- '--certificate-key' && echo 'PASS: lenh join co khoa giai ma'
echo "$JOIN_CP" | grep -q "${LB}:6443" && echo 'PASS: lenh join tro toi load balancer'
```

**PASS:** ba dòng `PASS:` xuất hiện.

**Ý nghĩa:** thay vì chép lại lệnh join in ra ở B9.3, lab sinh lại nó. Đây đúng là cách bài 08 chỉ
để chữa lỗi hết hạn: "để tải lên lại các certificate và tạo một khóa giải mã mới, hãy dùng lệnh sau
trên một node control plane đã được join vào cluster: `sudo kubeadm init phase upload-certs
--upload-certs`". Làm vậy thì khoảng cách thời gian giữa B9 và B10 không còn quan trọng — bạn có thể
nghỉ giữa chừng mà không hỏng bài.

> **Certificate key là bí mật.** Bài 08 cảnh báo nó "cho phép truy cập vào dữ liệu nhạy cảm của
> cluster". Token trong `JOIN_BASE` cũng vậy: ai có nó đều thêm node được vào cluster. **Không** ghi
> hai giá trị này vào `~/lab-evidence/8c/`.

### B10.2. Join `lab-ha-5`

Chạy **trên `lab-ha-5`**, dán nguyên lệnh `JOIN_CP` vừa in ra ở bước trên, chạy bằng `sudo`:

```bash
sudo kubeadm join 192.168.100.230:6443 \
  --token <token-that> \
  --discovery-token-ca-cert-hash sha256:<hash-that> \
  --control-plane --certificate-key <key-that>
```

**PASS:** output có dòng `This node has joined the cluster` và
`This node has joined the cluster and a new control plane instance was created`. Không copy
placeholder ở trên — dùng chuỗi thật mà B10.1 in ra.

Chạy trên **`lab-ha-4`**:

```bash
kubectl wait --for=condition=Ready node --all --timeout=600s
kubectl get nodes -o wide | tee "$EV/b10-nodes.txt"

NODE_N="$(kubectl get nodes --no-headers | wc -l)"
CP_N="$(kubectl get nodes -l 'node-role.kubernetes.io/control-plane' --no-headers | wc -l)"
API_N="$(kubectl -n kube-system get pod -l component=kube-apiserver --no-headers | wc -l)"
CM_N="$(kubectl -n kube-system get pod -l component=kube-controller-manager --no-headers | wc -l)"
SC_N="$(kubectl -n kube-system get pod -l component=kube-scheduler --no-headers | wc -l)"
echo "node=$NODE_N control plane=$CP_N apiserver=$API_N controller-manager=$CM_N scheduler=$SC_N"

test "$NODE_N" -eq 2 && test "$CP_N" -eq 2 \
  && test "$API_N" -eq 2 && test "$CM_N" -eq 2 && test "$SC_N" -eq 2 \
  && echo 'PASS: hai control plane node, moi node mot bo ba thanh phan control plane'
```

**PASS:** dòng `PASS: hai control plane node …` xuất hiện.

**Ý nghĩa:** con số `2 / 2 / 2` này là điểm chung mà bài [06](../06-ha-topology-vi.md) nêu cho **cả
hai** topology: "mỗi control plane node chạy một instance của `kube-apiserver`, `kube-scheduler` và
`kube-controller-manager`". Cái khác nằm ở dòng thứ tư mà bảng này **không có** — số Pod `etcd`. B11
đo đúng dòng đó.

### B10.3. Cân bằng lại CoreDNS

Bài 08 có ghi chú riêng: vì các node được khởi tạo tuần tự, Pod CoreDNS nhiều khả năng đều nằm trên
node đầu tiên. Chạy trên **`lab-ha-4`**:

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide | tee "$EV/b10-coredns-truoc.txt"

kubectl -n kube-system rollout restart deployment coredns
kubectl -n kube-system rollout status deployment coredns --timeout=300s
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide | tee "$EV/b10-coredns-sau.txt"

DNS_NODES="$(kubectl -n kube-system get pods -l k8s-app=kube-dns \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | wc -l)"
echo "so node dang chua Pod CoreDNS: $DNS_NODES"

test "$DNS_NODES" -ge 1 && echo 'PASS: CoreDNS da duoc rollout lai'
test "$DNS_NODES" -eq 2 \
  && echo 'PASS: hai Pod CoreDNS nam tren hai node khac nhau' \
  || echo 'GHI NHAN: hai Pod CoreDNS van cung mot node — hop le, scheduler khong bat buoc trai deu'
```

**PASS:** dòng `PASS: CoreDNS da duoc rollout lai` xuất hiện, cộng một trong hai dòng còn lại. Đây là
chỗ **không được** biến thành gate cứng: bài 08 chỉ nói rollout restart là cách "cung cấp tính sẵn
sàng cao hơn", không hứa scheduler sẽ trải đều. Ghi kết quả thật vào evidence.

---

## B11. Bằng chứng "external" thật sự

**Mục đích:** ba phép kiểm độc lập chứng minh cluster đang dùng etcd ngoài. Một phép nhìn từ phía
Kubernetes, một phép nhìn từ đĩa của node, một phép nhìn từ cấu hình apiserver. Ba nguồn khác nhau
cùng chỉ về một kết luận thì kết luận đó chắc.

### B11.1. Không Pod `component=etcd` nào

Chạy trên **`lab-ha-4`**:

```bash
kubectl -n kube-system get pod -l component=etcd -o wide | tee "$EV/b11-etcd-pods.txt"
ETCD_POD_N="$(kubectl -n kube-system get pod -l component=etcd --no-headers 2>/dev/null | wc -l)"
echo "Pod component=etcd trong cluster nay: $ETCD_POD_N"

test "$ETCD_POD_N" -eq 0 \
  && echo 'PASS: 0 Pod etcd — etcd khong con do control plane quan ly'
```

**PASS:** dòng `PASS: 0 Pod etcd …` xuất hiện.

**Ý nghĩa:** đối chiếu thẳng với con số bạn đã ghi ở B0.2: cụm stacked của Lab 8b có **3** Pod như
vậy. Cluster này có **0**. Không phải vì etcd biến mất — B8 đã chứng minh nó đang chạy trên ba máy
khác — mà vì nó **không còn là workload của Kubernetes**. kubelet trên ba host etcd không đăng ký
Node, nên static Pod của chúng không có mirror Pod trong API server để mà đếm.

### B11.2. Không `etcd.yaml` trên đĩa của control plane

Chạy trên **`lab-ha-4`**:

```bash
sudo ls -1 /etc/kubernetes/manifests/ | tee "$EV/b11-manifests-cp1.txt"
ssh -o StrictHostKeyChecking=accept-new "$N5" 'sudo ls -1 /etc/kubernetes/manifests/' \
  | tee "$EV/b11-manifests-cp2.txt"

OKM=0
for f in "$EV/b11-manifests-cp1.txt" "$EV/b11-manifests-cp2.txt"; do
  grep -q 'etcd.yaml' "$f" && echo "FAIL: $f co etcd.yaml" || OKM=$(( OKM + 1 ))
  grep -q 'kube-apiserver.yaml' "$f" || echo "FAIL: $f thieu kube-apiserver.yaml"
done
echo "control plane node khong co etcd.yaml: $OKM/2"

test "$OKM" -eq 2 \
  && echo 'PASS: khong control plane node nao chay static Pod etcd'
```

**PASS:** dòng `PASS: khong control plane node nao chay static Pod etcd` xuất hiện, và cả hai file
đều liệt kê `kube-apiserver.yaml`, `kube-controller-manager.yaml`, `kube-scheduler.yaml`.

Nếu `ssh lab-ha-5` hỏi mật khẩu, tạo key giống B0.3 — B13 và B14 còn cần đường `ssh` này.

### B11.3. `--etcd-servers` trỏ tới ba host etcd

Chạy trên **`lab-ha-4`**:

```bash
{
  echo "== lab-ha-4"
  sudo grep -E '^[[:space:]]+- --etcd-' /etc/kubernetes/manifests/kube-apiserver.yaml
  echo "== lab-ha-5"
  ssh "$N5" "sudo grep -E '^[[:space:]]+- --etcd-' /etc/kubernetes/manifests/kube-apiserver.yaml"
} | tee "$EV/b11-etcd-flags.txt"

SRV_LINES="$(grep -c -- '--etcd-servers=' "$EV/b11-etcd-flags.txt")"
BOTH3="$(grep -- '--etcd-servers=' "$EV/b11-etcd-flags.txt" \
  | grep -c "https://${E1}:2379,https://${E2}:2379,https://${E3}:2379")"
LOOP="$(grep -c -- '--etcd-servers=https://127.0.0.1:2379' "$EV/b11-etcd-flags.txt")"
CAF="$(grep -c -- '--etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt' "$EV/b11-etcd-flags.txt")"
CRT="$(grep -c -- '--etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt' "$EV/b11-etcd-flags.txt")"
KEY="$(grep -c -- '--etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key' "$EV/b11-etcd-flags.txt")"
echo "etcd-servers=$SRV_LINES | du ba host=$BOTH3 | tro loopback=$LOOP | cafile=$CAF certfile=$CRT keyfile=$KEY"

test "$SRV_LINES" -eq 2 && test "$BOTH3" -eq 2 \
  && echo 'PASS: CA HAI apiserver deu tro toi CA BA host etcd — quan he nhieu-nhieu'
test "$LOOP" -eq 0 \
  && echo 'PASS: khong apiserver nao tro toi etcd cuc bo'
test "$CAF" -eq 2 && test "$CRT" -eq 2 && test "$KEY" -eq 2 \
  && echo 'PASS: ca ba cofile certificate deu dung duong dan da khai trong kubeadm-config.yaml'
```

**PASS:** ba dòng `PASS:` xuất hiện.

**Ý nghĩa — đây là chỗ bài 06 được chứng minh bằng số.** Ở B1.1, ba apiserver của cụm stacked cho ra
ba dòng `--etcd-servers=https://127.0.0.1:2379`: quan hệ **một-một**, mỗi apiserver chỉ biết member
cục bộ. Ở đây, hai apiserver cho ra hai dòng liệt kê **cả ba** địa chỉ etcd: quan hệ **nhiều-nhiều**,
đúng câu "mỗi etcd host giao tiếp với `kube-apiserver` của từng control plane node". Cùng một trường
cấu hình, hai topology, hai hình dạng dữ liệu.

### B11.4. `ClusterConfiguration` trong ConfigMap `kubeadm-config`

Chạy trên **`lab-ha-4`**:

```bash
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' \
  | tee "$EV/b11-clusterconfig.txt"; echo

grep -q 'external:' "$EV/b11-clusterconfig.txt" \
  && echo 'PASS: ClusterConfiguration luu trong cluster khai etcd.external'
grep -q '  local:' "$EV/b11-clusterconfig.txt" \
  && echo 'FAIL: con khai etcd.local' \
  || echo 'PASS: khong khai etcd.local'
test "$(grep -c "https://192.168.100.23[123]:2379" "$EV/b11-clusterconfig.txt")" -eq 3 \
  && echo 'PASS: du ba endpoint etcd trong ClusterConfiguration'
grep -q "controlPlaneEndpoint: ${LB}:6443" "$EV/b11-clusterconfig.txt" \
  && echo 'PASS: controlPlaneEndpoint luu trong cluster van la dia chi load balancer'
```

**PASS:** bốn dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

**Ý nghĩa:** ConfigMap `kubeadm-config` là nơi kubeadm **lưu lại** cấu hình cluster để các lệnh sau
— `kubeadm join`, `kubeadm upgrade`, `kubeadm reset` — đọc lại. Nói cách khác, cluster **tự khai
báo** rằng nó dùng etcd ngoài, và mọi node join sau này sẽ được cấu hình theo đúng khai báo đó chứ
không phải theo file `kubeadm-config.yaml` còn nằm trong `~/lab-work/8c/` của bạn. Đây cũng là nguồn
mà `kubeadm reset` trên `lab-ha-4`/`lab-ha-5` sẽ đọc để biết **không được xóa dữ liệu etcd** — đúng
dòng cuối bảng ở B2.1.

Ghi ảnh chụp "sau" để đối chiếu với `b0-truoc-stacked.txt`:

```bash
{
  echo "=== $(date -Is) — cluster external etcd, sau khi dung xong ==="
  echo '--- nodes'; kubectl get nodes -o wide
  echo '--- control plane pods'; kubectl -n kube-system get pods -l tier=control-plane -o wide
  echo '--- so Pod component=etcd'; kubectl -n kube-system get pod -l component=etcd --no-headers | wc -l
  echo '--- etcd flags'; cat "$EV/b11-etcd-flags.txt"
  echo '--- ClusterConfiguration'; cat "$EV/b11-clusterconfig.txt"
} | tee "$EV/b11-sau-external.txt"

test -s "$EV/b11-sau-external.txt" && echo 'PASS: da ghi anh chup cluster external'
```

**PASS:** dòng `PASS: da ghi anh chup cluster external` xuất hiện.

---

## B12. Workload chứng minh cluster phục vụ được

**Mục đích:** một cluster "khỏe" theo `/readyz` vẫn có thể không chạy nổi một Pod. B12 dựng một
Deployment và một Service — công cụ của giai đoạn 4 và 5 — để B13 và B14 có thứ thật để đo trong lúc
tắt máy.

### B12.1. Gỡ taint control plane, có ý thức

Chạy trên **`lab-ha-4`**:

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints' \
  | tee "$EV/b12-taint-truoc.txt"

kubectl taint nodes --all node-role.kubernetes.io/control-plane-

kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints' \
  | tee "$EV/b12-taint-sau.txt"

TAINT_LEFT="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}' \
  | grep -c 'node-role.kubernetes.io/control-plane')"
echo "node con taint control-plane: $TAINT_LEFT"

grep -q 'control-plane' "$EV/b12-taint-truoc.txt" \
  && echo 'PASS: truoc do ca hai node deu mang taint control-plane'
test "$TAINT_LEFT" -eq 0 && echo 'PASS: da go taint tren ca hai node'
```

**PASS:** hai dòng `PASS:` xuất hiện.

> **Đây là thỏa hiệp của môi trường lab, không phải cách làm đúng.** Cluster này không có worker
> riêng vì bộ VM chỉ có sáu máy — xem [mục 2.2](#22-vì-sao-hai-control-plane-cộng-ba-etcd-vẫn-hợp-lệ).
> Bài [02](../02-create-cluster-kubeadm-vi.md) nói mặc định cluster **không lập lịch Pod trên control
> plane node vì lý do bảo mật**. B15.2 trả taint về nguyên trạng trước khi chụp snapshot.

### B12.2. Deployment và Service

Chạy trên **`lab-ha-4`**:

```bash
kubectl create namespace lab-8c

cat > "$WK/web.yaml" <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: lab-8c
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: httpd
          image: busybox:1.37
          command:
            - sh
            - -c
            - 'mkdir -p /www && echo web-8c-ok > /www/index.html && httpd -f -p 8080 -h /www'
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: lab-8c
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
EOF

kubectl apply -f "$WK/web.yaml"
kubectl -n lab-8c rollout status deploy/web --timeout=300s
kubectl -n lab-8c get pods -o wide | tee "$EV/b12-pods.txt"
kubectl -n lab-8c get endpointslice -l kubernetes.io/service-name=web | tee "$EV/b12-eps.txt"
```

```bash
READY="$(kubectl -n lab-8c get deploy web -o jsonpath='{.status.readyReplicas}')"
SPREAD="$(kubectl -n lab-8c get pods -l app=web \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | wc -l)"
echo "replica Ready=$READY | so node dang chay Pod web=$SPREAD"

test "$READY" -eq 4 && echo 'PASS: du bon replica Ready'
test "$SPREAD" -eq 2 && echo 'PASS: Pod trai tren ca hai control plane node'
```

**PASS:** hai dòng `PASS:` xuất hiện.

Kiểm Service từ bên trong cluster. Chạy trên **`lab-ha-4`**:

```bash
kubectl -n lab-8c run probe --image=busybox:1.37 --restart=Never --command -- sleep 3600
kubectl -n lab-8c wait --for=condition=Ready pod/probe --timeout=180s

kubectl -n lab-8c exec probe -- wget -q -O- http://web \
  | grep -qx 'web-8c-ok' && echo 'PASS: goi duoc Service bang ten ngan — DNS, ClusterIP va kube-proxy cung song'
```

**PASS:** dòng `PASS: goi duoc Service bang ten ngan …` xuất hiện.

Đặt sẵn một hàm để B13 và B14 gọi lại — nó là phép đo "cluster còn phục vụ không", dùng chung cho cả
ba phép thử:

```bash
serve_check () {
  local ns='lab-8c'
  local rz body
  rz="$(kubectl --request-timeout=15s get --raw='/readyz' 2>&1)"
  body="$(kubectl --request-timeout=20s -n "$ns" exec probe -- wget -q -O- http://web 2>&1)"
  if [ "$rz" = 'ok' ] && [ "$body" = 'web-8c-ok' ]; then
    echo 'SERVE: yes'
  else
    echo "SERVE: no (readyz=[$rz])"
  fi
}

serve_check | tee "$EV/b12-serve-baseline.txt"
grep -q 'SERVE: yes' "$EV/b12-serve-baseline.txt" \
  && echo 'PASS: ham serve_check hoat dong, moc goc la SERVE: yes'
```

**PASS:** dòng `PASS: ham serve_check hoat dong …` xuất hiện.

---

## B13. Chứng minh tính tách rời — ba phép thử tắt máy

**Mục đích:** phần đắt nhất của lab. Bài [06](../06-ha-topology-vi.md) khẳng định topology external
**tách rời** control plane và etcd member, nên mất một instance control plane hoặc một etcd member
"có ít tác động hơn". Ba phép thử dưới đây biến khẳng định đó thành số đo, và mỗi phép đều có một
dòng đối chiếu thẳng với Lab 8b.

> **Đọc trước khi chạy.** Phép thử thứ hai **cố ý làm mất quorum** của cụm etcd: API server ngừng
> phục vụ. Dữ liệu vẫn nằm trên đĩa của cả ba member, nên cụm hồi phục khi hai máy bật lại. Điều
> kiện là bạn **tắt đúng máy** và **bật lại đủ máy**. Không bao giờ tắt cả ba etcd node cùng lúc
> trong lab này, và không tắt máy thay cho `kubeadm reset` khi muốn gỡ hẳn một member.

### B13.0. Hai hàm đo dùng chung

Chạy trên **`lab-ha-4`**. `serve_check` đã có từ B12.2; thêm hàm đọc sức khỏe cụm etcd qua
`lab-ha-1` — máy này **không bị tắt** trong bất kỳ phép thử nào:

```bash
EPS='https://192.168.100.231:2379,https://192.168.100.232:2379,https://192.168.100.233:2379'

ehealth () {
  ssh -o BatchMode=yes -o ConnectTimeout=10 lab-ha-1 \
    'CID=$(sudo crictl ps --name etcd --state Running -q | head -n 1);
     sudo crictl exec "$CID" etcdctl \
       --cert /etc/kubernetes/pki/etcd/peer.crt \
       --key /etc/kubernetes/pki/etcd/peer.key \
       --cacert /etc/kubernetes/pki/etcd/ca.crt \
       --endpoints '"$EPS"' \
       --command-timeout=10s endpoint health 2>&1'
}

ehealth | tee "$EV/b13-0-health-goc.txt"
H0="$(grep -c 'is healthy' "$EV/b13-0-health-goc.txt")"
echo "endpoint khoe truoc khi thu: $H0/3"

test "$H0" -eq 3 && echo 'PASS: moc goc — ca ba etcd member khoe'
serve_check | grep -q 'SERVE: yes' && echo 'PASS: moc goc — cluster dang phuc vu'
```

**PASS:** hai dòng `PASS:` xuất hiện. Không mở B13.1 khi mốc gốc chưa đúng `3/3` và `SERVE: yes`.

### B13.1. Phép thử 1 — tắt **một** etcd node

Chạy trên **`lab-ha-4`**:

```bash
ssh lab-ha-3 'sudo shutdown -h now' || true
```

Chạy trên **máy host**, PowerShell, dùng lại `$vmrun` và `$vmxHA`:

```powershell
$e3 = $vmxHA | Where-Object { $_ -like '*lab-ha-3*' }
$running = & $vmrun -T ws list
if ($running -notcontains $e3) { "PASS: lab-ha-3 da Powered off" }
else { "CHUA TAT: doi them roi chay lai lenh nay" }
```

**PASS:** dòng `PASS: lab-ha-3 da Powered off` xuất hiện. Chạy lại khối này cho tới khi đạt; **không**
dùng `vmrun stop … hard` trừ khi [mục 4](#4-troubleshooting-của-lab-này) chỉ dẫn.

Đo, trên **`lab-ha-4`**:

```bash
H1=0
for i in $(seq 1 24); do
  ehealth > "$EV/b13-1-health.txt" 2>&1
  H1="$(grep -c 'is healthy' "$EV/b13-1-health.txt")"
  test "$H1" -eq 2 && break
  sleep 5
done
cat "$EV/b13-1-health.txt"
echo "endpoint khoe khi mat mot member: $H1/3"

serve_check | tee "$EV/b13-1-serve.txt"

test "$H1" -eq 2 && echo 'PASS: dung hai member con khoe — quorum 2/3 van du'
grep -q 'SERVE: yes' "$EV/b13-1-serve.txt" \
  && echo 'PASS: cluster VAN phuc vu day du khi mat mot etcd node'

kubectl get nodes | tee "$EV/b13-1-nodes.txt"
kubectl -n lab-8c get deploy web | tee "$EV/b13-1-deploy.txt"
test "$(kubectl get nodes --no-headers | wc -l)" -eq 2 \
  && echo 'PASS: hai control plane node van Ready — chung khong lien quan gi toi may vua tat'
```

**PASS:** ba dòng `PASS:` xuất hiện.

**Ý nghĩa:** `2/3` là con số của quorum. Cụm ba member chịu được mất **một**; hai member còn lại vẫn
đủ đa số để ghi. Và điều đáng chú ý hơn nằm ở phía Kubernetes: `kubectl get nodes` vẫn trả về **hai**
node, không node nào đổi trạng thái. Máy vừa tắt không phải Node của cluster này, nên cluster không
có gì để báo mất.

**Đối chiếu Lab 8b:** ở cụm stacked, không tồn tại phép thử này. Không có máy nào chỉ chạy etcd —
tắt bất kỳ máy nào cũng đồng thời là tắt một `kube-apiserver`. Đó chính là **hỏng theo cặp**.

Bật lại và chờ hồi phục. Chạy trên **máy host**:

```powershell
$e3 = $vmxHA | Where-Object { $_ -like '*lab-ha-3*' }
& $vmrun -T ws start $e3
$running = & $vmrun -T ws list
if ($running -contains $e3) { "PASS: lab-ha-3 dang chay" } else { "FAIL: VM chua len" }
```

Chạy trên **`lab-ha-4`**:

```bash
H1b=0
for i in $(seq 1 60); do
  ehealth > "$EV/b13-1-health-hoi-phuc.txt" 2>&1
  H1b="$(grep -c 'is healthy' "$EV/b13-1-health-hoi-phuc.txt")"
  test "$H1b" -eq 3 && break
  sleep 5
done
cat "$EV/b13-1-health-hoi-phuc.txt"
echo "endpoint khoe sau khi bat lai: $H1b/3"

test "$H1b" -eq 3 && echo 'PASS: cum etcd tro lai du ba member khoe'
```

**PASS:** dòng `PASS: cum etcd tro lai du ba member khoe` xuất hiện. Vòng lặp có số vòng tối đa vì
thời gian máy boot và member bắt kịp log **phụ thuộc cấu hình và phần cứng host**.

### B13.2. Phép thử 2 — tắt **hai** etcd node, mất quorum

> **Đây là phép thử phá hoại có kiểm soát.** Tắt `lab-ha-2` và `lab-ha-3`; **giữ `lab-ha-1` chạy**.
> Đừng bỏ dở phép thử này giữa chừng — luôn chạy hết phần bật lại ở cuối mục.

Chạy trên **`lab-ha-4`**:

```bash
ssh lab-ha-2 'sudo shutdown -h now' || true
ssh lab-ha-3 'sudo shutdown -h now' || true
```

Chạy trên **máy host**, PowerShell:

```powershell
$two = $vmxHA | Where-Object { $_ -like '*lab-ha-2*' -or $_ -like '*lab-ha-3*' }
$running = & $vmrun -T ws list
$still = $two | Where-Object { $running -contains $_ }
if ($still.Count -eq 0) { "PASS: lab-ha-2 va lab-ha-3 deu Powered off" }
else { "CHUA TAT: $($still -join ', ')" }
```

**PASS:** dòng `PASS: lab-ha-2 va lab-ha-3 deu Powered off` xuất hiện.

Đo, trên **`lab-ha-4`**:

```bash
SERVE2='chua do'
for i in $(seq 1 24); do
  SERVE2="$(serve_check)"
  echo "$SERVE2" | grep -q 'SERVE: no' && break
  sleep 5
done
echo "$SERVE2" | tee "$EV/b13-2-serve.txt"

kubectl --request-timeout=15s get nodes 2>&1 | tee "$EV/b13-2-kubectl.txt"

echo "$SERVE2" | grep -q 'SERVE: no' \
  && echo 'PASS: mat quorum — API server ngung phuc vu'
```

**PASS:** dòng `PASS: mat quorum — API server ngung phuc vu` xuất hiện. Đọc kỹ `b13-2-kubectl.txt`:
thông báo lỗi ở đó đến **từ apiserver**, không phải từ `kubectl` — apiserver còn sống và còn nhận kết
nối, nó chỉ không đọc/ghi được dữ liệu nữa.

```bash
ehealth > "$EV/b13-2-health.txt" 2>&1
cat "$EV/b13-2-health.txt"
H2="$(grep -c 'is healthy' "$EV/b13-2-health.txt")"
echo "endpoint khoe khi mat hai member: $H2/3"

test "$H2" -eq 0 \
  && echo 'PASS: khong endpoint nao healthy — ke ca member con song, vi mot mot khong lam nen da so'
```

**PASS:** dòng `PASS: khong endpoint nao healthy …` xuất hiện.

**Ý nghĩa — đây là chỗ nhiều người hiểu sai.** `lab-ha-1` **vẫn đang chạy**, container etcd của nó
vẫn sống, `crictl exec` vẫn vào được. Nhưng nó vẫn báo không healthy, vì một member trên ba **không
lập thành đa số**: nó không có quyền xác nhận một ghi nào, nên nó cũng không dám khẳng định dữ liệu
nó đang giữ là mới nhất. Quorum không phải "còn ít nhất một bản sao"; quorum là "có quá nửa số
member đồng ý".

Và hệ quả dây chuyền: `kube-apiserver` không có nơi lưu trạng thái thì **không phục vụ được**, dù cả
hai control plane node vẫn bật, vẫn nghe cổng `6443`, và load balancer vẫn chuyển tiếp TCP bình
thường. Đó là câu trả lời cho "vì sao quorum nằm ở tầng etcd" trong
[mục 2.2](#22-vì-sao-hai-control-plane-cộng-ba-etcd-vẫn-hợp-lệ).

Bật lại. Chạy trên **máy host**:

```powershell
$two = $vmxHA | Where-Object { $_ -like '*lab-ha-2*' -or $_ -like '*lab-ha-3*' }
foreach ($f in $two) { & $vmrun -T ws start $f }
$running = & $vmrun -T ws list
$up = $two | Where-Object { $running -contains $_ }
if ($up.Count -eq 2) { "PASS: ca hai VM da chay lai" } else { "FAIL: con thieu VM" }
```

Chạy trên **`lab-ha-4`**:

```bash
H2b=0
for i in $(seq 1 72); do
  ehealth > "$EV/b13-2-health-hoi-phuc.txt" 2>&1
  H2b="$(grep -c 'is healthy' "$EV/b13-2-health-hoi-phuc.txt")"
  test "$H2b" -eq 3 && break
  sleep 5
done
cat "$EV/b13-2-health-hoi-phuc.txt"
echo "endpoint khoe sau khi bat lai: $H2b/3"

SERVE2b=''
for i in $(seq 1 36); do
  SERVE2b="$(serve_check)"
  echo "$SERVE2b" | grep -q 'SERVE: yes' && break
  sleep 5
done
echo "$SERVE2b" | tee "$EV/b13-2-serve-hoi-phuc.txt"

test "$H2b" -eq 3 && echo 'PASS: cum etcd hoi phuc du ba member'
grep -q 'SERVE: yes' "$EV/b13-2-serve-hoi-phuc.txt" \
  && echo 'PASS: cluster phuc vu tro lai, khong mat du lieu'

kubectl -n lab-8c get deploy web -o wide | tee "$EV/b13-2-deploy-hoi-phuc.txt"
test "$(kubectl -n lab-8c get deploy web -o jsonpath='{.status.readyReplicas}')" -eq 4 \
  && echo 'PASS: Deployment cua B12 con nguyen — du lieu nam tren dia cua tung member'
```

**PASS:** ba dòng `PASS:` xuất hiện.

### B13.3. Phép thử 3 — tắt **một** control plane node

Đây là phép thử phân biệt hai topology rõ nhất. Chạy trên **`lab-ha-4`**:

```bash
kubectl get nodes -o wide | tee "$EV/b13-3-nodes-truoc.txt"
ssh lab-ha-5 'sudo shutdown -h now' || true
```

Chạy trên **máy host**, PowerShell:

```powershell
$cp2 = $vmxHA | Where-Object { $_ -like '*lab-ha-5*' }
$running = & $vmrun -T ws list
if ($running -notcontains $cp2) { "PASS: lab-ha-5 da Powered off" }
else { "CHUA TAT: doi them roi chay lai lenh nay" }
```

**PASS:** dòng `PASS: lab-ha-5 da Powered off` xuất hiện.

Đo, trên **`lab-ha-4`**:

```bash
serve_check | tee "$EV/b13-3-serve.txt"
grep -q 'SERVE: yes' "$EV/b13-3-serve.txt" \
  && echo 'PASS: cluster VAN phuc vu qua load balancer khi mat mot control plane node'

ehealth > "$EV/b13-3-health.txt" 2>&1
H3="$(grep -c 'is healthy' "$EV/b13-3-health.txt")"
cat "$EV/b13-3-health.txt"
echo "endpoint etcd khoe: $H3/3"
test "$H3" -eq 3 \
  && echo 'PASS: cum etcd VAN du ba member — tat control plane khong dung toi etcd'

ST=''
for i in $(seq 1 60); do
  ST="$(kubectl --request-timeout=15s get node lab-ha-5 \
    -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null)"
  test "$ST" != 'True' && break
  sleep 5
done
echo "Ready condition cua lab-ha-5 = $ST"
kubectl get nodes | tee "$EV/b13-3-nodes-down.txt"
test "$ST" = 'Unknown' \
  && echo 'PASS: node controller doi Ready thanh Unknown khi mat heartbeat'

API_UP="$(kubectl -n kube-system get pod -l component=kube-apiserver \
  --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)"
echo "Pod kube-apiserver dang Running: $API_UP"
```

**PASS:** ba dòng `PASS:` xuất hiện.

**Ý nghĩa — đặt cạnh Lab 8b thì thấy toàn bộ nội dung của bài 06:**

| Đo cái gì | Lab 8b, tắt một control plane | Lab 8c, tắt `lab-ha-5` |
| --- | --- | --- |
| Số `kube-apiserver` còn lại | 2/3 | 1/2 |
| Số etcd member còn khỏe | **2/3 — mất một member cùng lúc** | **3/3 — không member nào bị đụng** |
| Cluster còn phục vụ | có | có |
| Còn chịu được mất thêm bao nhiêu etcd member | **0** | **1** |

Dòng cuối là điều bài 06 gọi là "tính dự phòng bị suy giảm". Ở stacked, sau khi mất một máy bạn đang
đứng **sát mép**: mất thêm một máy nữa là mất quorum. Ở external, sau khi mất `lab-ha-5` bạn vẫn còn
nguyên biên độ ở tầng etcd — đúng câu bài viết: mất một instance control plane "không ảnh hưởng đến
tính dự phòng của cluster nhiều như topology HA xếp chồng".

Bật lại. Chạy trên **máy host**:

```powershell
$cp2 = $vmxHA | Where-Object { $_ -like '*lab-ha-5*' }
& $vmrun -T ws start $cp2
$running = & $vmrun -T ws list
if ($running -contains $cp2) { "PASS: lab-ha-5 dang chay" } else { "FAIL: VM chua len" }
```

Chạy trên **`lab-ha-4`**:

```bash
kubectl wait --for=condition=Ready node/lab-ha-5 --timeout=600s
kubectl get nodes -o wide | tee "$EV/b13-3-nodes-up.txt"

API_UP2=0
for i in $(seq 1 60); do
  API_UP2="$(kubectl -n kube-system get pod -l component=kube-apiserver \
    --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)"
  test "$API_UP2" -eq 2 && break
  sleep 5
done
echo "Pod kube-apiserver dang Running: $API_UP2"

test "$API_UP2" -eq 2 && echo 'PASS: ca hai kube-apiserver da chay lai'
serve_check | grep -q 'SERVE: yes' && echo 'PASS: cluster phuc vu binh thuong'
```

**PASS:** hai dòng `PASS:` xuất hiện.

---

## B14. Checkpoint giai đoạn 8 — xóa `kube-apiserver.yaml` rồi khôi phục

**Mục đích:** [Checkpoint của giai đoạn 8](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm)
yêu cầu "phá rồi khôi phục: tắt một control plane node và chứng minh cluster vẫn phục vụ; xóa
`/etc/kubernetes/manifests/kube-apiserver.yaml` rồi khôi phục". Vế thứ nhất đã xong ở B13.3; B14 làm
vế thứ hai — và lần này cluster mất một apiserver **mà không mất node**, nên phép đo sạch hơn.

> **Chỉ chạm `lab-ha-5`.** Đây là fault target duy nhất ở tầng control plane, theo
> [mục 2.1](#21-bảng-đổi-vai-của-sáu-vm). Xóa file này trên `lab-ha-4` là tự cắt đường quan sát của
> chính mình.

### B14.1. Chuyển manifest ra khỏi thư mục static Pod

Chạy trên **`lab-ha-4`**:

```bash
ssh lab-ha-5 \
  'sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml'

ssh lab-ha-5 'sudo ls -1 /etc/kubernetes/manifests/' | tee "$EV/b14-manifests-thieu.txt"
grep -q 'kube-apiserver.yaml' "$EV/b14-manifests-thieu.txt" \
  && echo 'FAIL: file van con trong thu muc' \
  || echo 'PASS: kube-apiserver.yaml da roi khoi /etc/kubernetes/manifests'
ssh lab-ha-5 'sudo test -f /root/kube-apiserver.yaml' \
  && echo 'PASS: da giu ban sao o /root, khoi phuc duoc'
```

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`. Dùng `mv` chứ không `rm` là có chủ ý:
bài học ở đây là **kubelet phản ứng với nội dung thư mục**, không phải luyện gõ lại một manifest.

### B14.2. Đo trong lúc thiếu một apiserver

Chạy trên **`lab-ha-4`**:

```bash
APIN=2
for i in $(seq 1 36); do
  APIN="$(kubectl --request-timeout=15s -n kube-system get pod -l component=kube-apiserver \
    --no-headers 2>/dev/null | wc -l)"
  test "$APIN" -le 1 && break
  sleep 5
done
kubectl -n kube-system get pod -l component=kube-apiserver -o wide | tee "$EV/b14-api-pods.txt"
echo "Pod kube-apiserver con lai: $APIN"

test "$APIN" -eq 1 \
  && echo 'PASS: chi con mot kube-apiserver — kubelet tren lab-ha-5 da xoa Pod theo manifest'

ONNODE="$(kubectl -n kube-system get pod -l component=kube-apiserver \
  -o jsonpath='{.items[0].spec.nodeName}')"
echo "apiserver con lai nam tren: $ONNODE"
test "$ONNODE" = 'lab-ha-4' && echo 'PASS: apiserver con lai dung la cua lab-ha-4'
```

```bash
serve_check | tee "$EV/b14-serve.txt"
grep -q 'SERVE: yes' "$EV/b14-serve.txt" \
  && echo 'PASS: cluster VAN phuc vu qua load balancer voi mot apiserver'

curl -sk "https://${LB}:6443/version" | grep -q 'gitVersion' \
  && echo 'PASS: goi thang qua load balancer van co ket qua'

READY5="$(kubectl get node lab-ha-5 \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
echo "lab-ha-5 Ready=$READY5"
test "$READY5" = 'True' \
  && echo 'PASS: lab-ha-5 VAN Ready — kubelet cua no van song, chi thieu apiserver'
```

**PASS:** bốn dòng `PASS:` của B14.2 xuất hiện.

**Ý nghĩa:** ba kết luận từ một thao tác.

1. **Static Pod bám vào thư mục, không bám vào API server.** Bạn không xóa Pod bằng `kubectl`; bạn
   dời một file, và kubelet trên `lab-ha-5` tự dừng container rồi xóa mirror Pod. Đây là điều bài
   [128](../128-api-server-bypass-risks-vi.md) của giai đoạn 9 sẽ quay lại ở góc bảo mật: ai ghi được
   vào `/etc/kubernetes/manifests` thì chạy được container trên node mà không đi qua API server.
2. **Load balancer làm đúng việc của nó.** Health check TCP trên `:6443` thấy `lab-ha-5` không còn
   nghe cổng đó, nên nó ngừng gửi request sang. Bạn không phải sửa gì trong kubeconfig — địa chỉ vẫn
   là `ControlPlaneEndpoint`, còn việc chọn backend nào là chuyện của HAProxy.
3. **Node vẫn `Ready`.** kubelet là thứ báo cáo Node status, và kubelet không chết. Đây là khác biệt
   so với B13.3, nơi cả máy tắt: ở đó Node chuyển `Unknown`, ở đây thì không. Hai sự cố khác nhau cho
   hai triệu chứng khác nhau — nhớ cặp này để chẩn đoán nhanh trên cluster thật.

### B14.3. Khôi phục

Chạy trên **`lab-ha-4`**:

```bash
ssh lab-ha-5 'sudo mv /root/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml'

APIN2=0
for i in $(seq 1 60); do
  APIN2="$(kubectl -n kube-system get pod -l component=kube-apiserver \
    --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)"
  test "$APIN2" -eq 2 && break
  sleep 5
done
kubectl -n kube-system get pod -l component=kube-apiserver -o wide | tee "$EV/b14-api-pods-hoi-phuc.txt"
echo "Pod kube-apiserver Running: $APIN2"

ssh lab-ha-5 'sudo test -f /root/kube-apiserver.yaml' \
  && echo 'FAIL: ban sao van con o /root' \
  || echo 'PASS: da tra file ve dung cho, khong con ban sao lac'
test "$APIN2" -eq 2 && echo 'PASS: ca hai kube-apiserver da chay lai'
serve_check | grep -q 'SERVE: yes' && echo 'PASS: cluster phuc vu binh thuong'
```

**PASS:** ba dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

---

## B15. Cleanup, snapshot `8x-ha-external` và gate cuối

**Mục đích:** xóa mọi thứ của bài học, trả taint control plane về mặc định kubeadm, chụp mốc mới
trên đủ sáu VM, và chứng minh bộ VM chính không hề bị chạm.

### B15.1. Xóa object và file tạm của bài học

Chạy trên **`lab-ha-4`**:

```bash
kubectl delete namespace lab-8c --wait=true --timeout=300s

rm -f "$WK/web.yaml" "$WK/kubeadm-config.yaml"
rmdir "$WK"

NS_LEFT="$(kubectl get namespace lab-8c --ignore-not-found -o name 2>/dev/null | wc -l)"
POD_DEF="$(kubectl get pods -n default --no-headers 2>/dev/null | wc -l)"
echo "namespace lab-8c con=$NS_LEFT | Pod trong default=$POD_DEF"

test "$NS_LEFT" -eq 0 && echo 'PASS: namespace lab-8c da bien mat'
test "$POD_DEF" -eq 0 && echo 'PASS: namespace default khong con Pod nao'
test ! -e "$WK" && echo 'PASS: manifest tam da xoa het'
```

**PASS:** ba dòng `PASS:` xuất hiện. `rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ,
và `test ! -e` biến điều đó thành gate. Thư mục `~/lab-evidence/8c/` trên **cả sáu máy** thì **giữ
lại** — đó là bằng chứng, và snapshot sắp chụp sẽ mang theo nó.

Trên `lab-ha-1` còn ba thư mục tạm chứa certificate của B5–B6. Chạy trên **`lab-ha-4`**:

```bash
ssh lab-ha-1 'rm -rf /tmp/192.168.100.231 /tmp/192.168.100.232 /tmp/192.168.100.233'
ssh lab-ha-1 'ls -d /tmp/192.168.100.23* 2>/dev/null | wc -l' \
  | grep -qx '0' && echo 'PASS: da xoa thu muc tam chua certificate tren lab-ha-1'

for n in lab-ha-2 lab-ha-3; do
  ssh "$n" 'rm -f $HOME/kubeadmcfg.yaml'
done
echo 'PASS: da xoa kubeadmcfg.yaml con lai o thu muc home cua hai host etcd'
```

**PASS:** hai dòng `PASS:` xuất hiện. Certificate của etcd **vẫn ở nguyên** `/etc/kubernetes/pki` —
cụm etcd đang chạy bằng chúng; chỉ các bản sao trong `/tmp` và `$HOME` bị xóa.

### B15.2. Trả taint control plane về mặc định kubeadm

Chạy trên **`lab-ha-4`**:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule

kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints' \
  | tee "$EV/b15-taint-cuoi.txt"

TAINT_N="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"="}{.effect}{"\n"}{end}{end}' \
  | grep -c '^node-role.kubernetes.io/control-plane=NoSchedule$')"
echo "node mang taint control-plane:NoSchedule: $TAINT_N/2"

test "$TAINT_N" -eq 2 \
  && echo 'PASS: ca hai node da tro lai trang thai mac dinh cua kubeadm'
```

**PASS:** dòng `PASS: ca hai node da tro lai trang thai mac dinh cua kubeadm` xuất hiện.

Snapshot `8x-ha-external` vì thế được định nghĩa là: **2 control plane + 3 etcd node ngoài + 1 load
balancer, Flannel, taint control plane ở mặc định kubeadm, không workload.** Lab nào dùng lại mốc này
mà cần chạy workload thì tự gỡ taint bằng đúng lệnh của B12.1 và tự chịu trách nhiệm với ghi chú bảo
mật ở đó.

### B15.3. Gate sức khỏe cuối trên cả hai tầng

Chạy trên **`lab-ha-4`**:

```bash
kubectl wait --for=condition=Ready node --all --timeout=300s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns

NODE_N="$(kubectl get nodes --no-headers | wc -l)"
ETCD_POD_N="$(kubectl -n kube-system get pod -l component=etcd --no-headers 2>/dev/null | wc -l)"
echo "node=$NODE_N | Pod component=etcd=$ETCD_POD_N"

test "$NODE_N" -eq 2 && test "$ETCD_POD_N" -eq 0 \
  && echo 'PASS: van dung hinh dang external — 2 node, 0 Pod etcd'

ehealth > "$EV/b15-health.txt" 2>&1
HF="$(grep -c 'is healthy' "$EV/b15-health.txt")"
cat "$EV/b15-health.txt"
test "$HF" -eq 3 && echo 'PASS: cum etcd du ba member khoe sau ba phep thu cua B13'
```

**PASS:** ba dòng `PASS:` xuất hiện; lệnh `--field-selector` trả `No resources found`; CoreDNS đủ
replica `READY`.

Ghi bằng chứng cuối:

```bash
{
  echo "=== $(date -Is) — trang thai cuoi Lab 8c ==="
  echo '--- nodes'; kubectl get nodes -o wide
  echo '--- taints'
  kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints'
  echo '--- control plane pods'; kubectl -n kube-system get pods -l tier=control-plane -o wide
  echo '--- so Pod component=etcd'; kubectl -n kube-system get pod -l component=etcd --no-headers | wc -l
  echo '--- namespaces'; kubectl get namespaces
  echo '--- etcd health'; cat "$EV/b15-health.txt"
} | tee "$EV/b15-cuoi.txt"

test -s "$EV/b15-cuoi.txt" && echo 'PASS: da ghi bang chung cuoi'
```

**PASS:** dòng `PASS: da ghi bang chung cuoi` xuất hiện.

### B15.4. Tắt sáu VM và chụp `8x-ha-external`

Tắt theo thứ tự ngược với thứ tự bật. Từ `lab-ha-4`, tắt `lab-ha-5` trước, rồi ba host etcd, cuối
cùng mới tắt chính `lab-ha-4`; `lab-ha-lb` tắt riêng từ phiên SSH của nó. Chạy trên **`lab-ha-4`**:

```bash
ssh lab-ha-5 'sudo shutdown -h now' || true
for n in lab-ha-3 lab-ha-2 lab-ha-1; do ssh "$n" 'sudo shutdown -h now' || true; done
sudo shutdown -h now
```

Tắt `lab-ha-lb` từ phiên SSH riêng của nó, **trên `lab-ha-lb`**:

```bash
sudo shutdown -h now
```

Chạy trên **máy host**, PowerShell — chờ đủ sáu máy về *Powered off*:

```powershell
$running = & $vmrun -T ws list
$still = $vmxHA | Where-Object { $running -contains $_ }
if ($still.Count -eq 0) { "PASS: ca sau VM HA da Powered off" }
else { "CHUA TAT: $($still -join ', ')" }
```

**PASS:** dòng `PASS: ca sau VM HA da Powered off` xuất hiện. Chạy lại cho tới khi đạt; chụp snapshot
khi VM đã tắt để mốc không kèm trạng thái RAM, đúng nguyên tắc
[A5.4.8 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a548-dọn-resource-test-ghi-evidence-và-chụp-snapshot).

Chụp trên **cả sáu VM**: chuột phải VM → **Snapshot → Take Snapshot** → ô *Name* điền đúng chuỗi:

```text
8x-ha-external
```

Ô *Description* ghi `dung bang LAB-8C-HA-VOI-EXTERNAL-ETCD.md, chup <ngay>`.

Quy tắc tên gọi, giống hệt quy tắc của Lab 00: đúng nguyên văn `8x-ha-external` trên cả sáu VM,
không hậu tố theo VM, không thêm ngày, không đổi hoa thường, không thừa khoảng trắng. Snapshot thuộc
về từng VM nên sáu snapshot trùng tên không xung đột.

Verify trên **máy host**, PowerShell:

```powershell
foreach ($f in $vmxHA) {
  $snaps = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($snaps -ccontains '8x-ha-external') { "PASS: $f" }
  else { "FAIL: $f -> $($snaps -join ', ')" }
}
```

**PASS:** đúng sáu dòng `PASS:`, không có dòng `FAIL:`. `-ccontains` phân biệt hoa thường nên gate
này bắt được cả lỗi gõ sai tên.

### B15.5. Gate cuối cùng: bộ VM chính vẫn nguyên vẹn

Chạy trên **máy host**, PowerShell:

```powershell
$running = & $vmrun -T ws list
foreach ($f in $vmxMain) {
  $isOff = ($running -notcontains $f)
  $snaps = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  $has03 = ($snaps -ccontains '03-storage-ready')
  $hasHA = @($snaps | Where-Object { $_ -like '8x-*' }).Count
  if ($isOff -and $has03 -and $hasHA -eq 0) { "PASS: $f khong bi cham" }
  else { "FAIL: $f -> off=$isOff co03=$has03 so-snapshot-8x=$hasHA" }
}
```

**PASS:** đúng ba dòng `PASS: … khong bi cham`. Ba điều kiện của gate này nói ba chuyện khác nhau:
máy vẫn tắt (không ai bật nhầm), mốc `03-storage-ready` vẫn còn (chuỗi snapshot chính không hỏng), và
**không có snapshot nào tiền tố `8x-`** trên chúng (không ai chụp nhầm mốc của nhánh HA lên bộ VM
chính).

Lab 8c kết thúc ở đây. Để sáu VM HA ở trạng thái tắt; lab sau — nếu bạn quay lại nhánh HA — bật máy
theo thứ tự của `$vmxHA` và bắt đầu từ mốc `8x-ha-external`.

---

## 3. Checkpoint 8c

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Bạn có sáu máy và muốn dựng HA. Kể ra **hai** cách chia vai — một theo stacked, một theo
      external — và với mỗi cách nói rõ: mấy máy chạy `kube-apiserver`, mấy máy giữ etcd member, còn
      mấy máy làm worker. Bài 06 khuyến nghị bao nhiêu host cho external, và vì sao lab vẫn dựng
      được với sáu?
- [ ] Trong cluster bạn vừa dựng, `--etcd-servers` của apiserver trên `lab-ha-4` trỏ tới đâu? Của
      `lab-ha-5` thì sao? Ở Lab 8b hai giá trị đó là gì? Chỉ nhìn hai dòng cấu hình này, bạn phân
      biệt được hai topology không — và vì sao?
- [ ] `kubeadm reset` vừa chạy xong trên một node. Kể **bốn** thứ nó **không** dọn, và với mỗi thứ
      nói hậu quả cụ thể nếu bạn dựng cluster mới mà quên dọn nó. Riêng dữ liệu etcd: reset xóa nó
      trong trường hợp nào và **không** xóa trong trường hợp nào?
- [ ] Trên ba host etcd, vì sao phải tạo `20-etcd-service-manager.conf` thay vì dùng unit kubelet mà
      gói `kubeadm` đã cài? Con số `20-` có nghĩa gì? Dòng `ExecStart=` rỗng ở đầu để làm gì? Sau
      bước đó, kubelet trên ba máy ấy đang nói chuyện với API server nào?
- [ ] Sau khi chạy hết bài 07, `ca.key` của etcd nằm trên máy nào, và vì sao hai lệnh
      `find /tmp/… -name ca.key -delete` là **nội dung bảo mật** chứ không phải dọn rác? Ba member
      lại có ba `server.crt` **khác nhau** — vì sao không dùng chung một certificate cho cả ba, SAN
      của `server.crt` trên `lab-ha-2` phải chứa gì, và ai là bên kiểm tra SAN đó?
- [ ] Firewall của bạn chỉ mở TCP `2379` giữa ba host etcd, chặn `2380`. `etcdctl endpoint health`
      chạy từ một host có ra kết quả không? Cụm etcd có hình thành được không? Giải thích bằng vai
      trò của từng port.
- [ ] Bạn dựng cluster với `kubeadm init --config`. Vì sao nhánh external **buộc** phải dùng
      `--config` chứ không dùng được cờ dòng lệnh? Cờ nào của `kubeadm init` **không** dùng chung
      được với `--config`, và nếu cần nó thì phải khai ở đâu?
- [ ] Kể **ba** bằng chứng độc lập chứng minh cluster đang dùng etcd ngoài, mỗi bằng chứng lấy từ
      một nguồn khác nhau. Vì sao `kubectl -n kube-system get pod -l component=etcd` trả về 0 Pod
      trong khi etcd rõ ràng đang chạy? Ba máy etcd có xuất hiện trong `kubectl get nodes` không?
- [ ] Bạn tắt `lab-ha-1`. Cluster còn phục vụ không? Bạn tắt tiếp `lab-ha-2` — còn phục vụ không?
      `lab-ha-3` vẫn đang chạy và container etcd của nó vẫn sống, vậy vì sao `endpoint health` báo
      **không** healthy? Giải thích bằng định nghĩa của quorum, đừng nói "vì hỏng".
- [ ] Bạn tắt `lab-ha-5`. So với việc tắt một control plane node ở Lab 8b, cluster của bạn **mất
      thêm** cái gì và **không mất** cái gì? Sau sự cố đó, mỗi cluster còn chịu được mất thêm bao
      nhiêu etcd member? Đó chính là câu nào của bài 06?
- [ ] Bạn `mv` file `kube-apiserver.yaml` ra khỏi `/etc/kubernetes/manifests` trên `lab-ha-5`. Ai
      xóa Pod — bạn hay kubelet? Vì sao `kubectl get nodes` vẫn báo `lab-ha-5` là `Ready`, khác hẳn
      lúc bạn tắt hẳn máy đó? Ai quyết định ngừng gửi request sang máy ấy?
- [ ] **Nợ #8 chưa được trả ở lab này.** Món nợ đó là gì, vì sao lab chỉ cho phép ba lệnh `etcdctl`
      và cấm hẳn `snapshot save`/`restore`, và nó được trả ở giai đoạn nào? Trước khi làm giai đoạn
      đó bạn phải đọc lại bài nào?

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng sau, không nhìn tài liệu:

1. **Từ một cụm stacked đang chạy tới một cluster external etcd.** Bắt đầu từ lúc bạn quyết định đổi
   topology. Kể đủ sáu chặng: gỡ cluster cũ theo thứ tự nào và vì sao thứ tự đó quan trọng; bốn thứ
   bạn phải dọn tay sau `kubeadm reset`; vì sao bước tiếp theo là sửa load balancer chứ không phải
   chạy `kubeadm init`; ba host etcd đi qua những gì để thành một cụm — kubelet, cấu hình, certificate,
   static Pod — và ở mỗi bước cái gì là **riêng** của từng host, cái gì là **chung**; thứ duy nhất
   trong `kubeadm-config.yaml` khiến cluster mới dùng etcd ngoài; và cuối cùng, ba bằng chứng bạn
   trưng ra khi có người hỏi "sao biết nó là external?".
2. **Cái giá và cái được của tách rời.** Đi qua ba phép thử của B13 và với mỗi phép nói: bạn tắt gì,
   cluster phản ứng ra sao, cụm etcd phản ứng ra sao, và **Lab 8b sẽ khác ở chỗ nào**. Rồi trả lời
   ba câu ranh giới: quorum nằm ở tầng nào và vì sao hai control plane vẫn hợp lệ; vì sao ba phép
   thử đó **không** chứng minh cluster chịu được mọi loại hỏng hóc — chỉ ra thành phần vẫn là điểm
   hỏng đơn; và vì sao cluster này không có worker riêng, việc gỡ taint control plane đánh đổi cái
   gì, production làm khác thế nào. Kết bằng: **một chủ đề của giai đoạn 8 mà lab cố ý không làm**,
   và nó được trả ở đâu.

Khi mọi checkbox được đánh dấu và bạn không còn nhầm *etcd member* với *control plane node*,
*port 2380* với *port 2379*, *`etcd.local`* với *`etcd.external`*, hay *node bị tắt* với *node thiếu
một static Pod* — Lab 8c và
[giai đoạn 8](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) hoàn tất.

Lab này **không phát sinh nợ mới** và **không trả nợ nào**; xem
[sổ nợ lab](README.md#5-sổ-nợ-lab). Riêng **[nợ #8](README.md#5-sổ-nợ-lab) — backup và restore etcd
— vẫn đang treo sau khi bạn học xong Lab 8c**: bạn đã dựng được một cụm etcd ba thành viên và đọc
được sức khỏe của nó, nhưng **chưa có quy trình sao lưu và khôi phục nào**. Cho tới
[giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa), đường lui
duy nhất của bạn vẫn là snapshot VM `8x-ha-external`. [Nợ #7](README.md#5-sổ-nợ-lab) — vòng đời
certificate — cũng chưa trả, dù lab này vừa sinh ra một CA và tám certificate.

---

## 4. Troubleshooting của lab này

Sự cố **dựng môi trường** — VM không ping nhau, swap bật lại, containerd hỏng, package sai version —
nằm ở [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Sự cố **của
kubeadm nói chung** tra ở [bài 09 — Xử lý sự cố kubeadm](../09-troubleshooting-kubeadm-vi.md); đó là
trang tra cứu, không đọc tuần tự. Bảng dưới chỉ liệt kê sự cố phát sinh từ **nội dung bài học 8c**.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| **B2.5/B2.6: `kubeadm reset` treo ở `Removing kubernetes-managed containers`** | `sudo crictl ps -a`; `systemctl is-active containerd` | Đây đúng tình huống *"kubeadm bị treo khi xóa các container được quản lý"* của [bài 09](../09-troubleshooting-kubeadm-vi.md): container runtime dừng hoạt động và không xóa container. Cách bài đưa: **khởi động lại container runtime rồi chạy lại `kubeadm reset`**, và dùng `crictl` để gỡ lỗi trạng thái runtime. Thử đúng một lần; vẫn treo thì sang [fallback B2.7](#b27-fallback--khi-gate-reset-không-đạt) |
| B2.5: `/var/lib/etcd` còn dữ liệu sau reset | `ssh <node> 'sudo ls -A /var/lib/etcd'` | Reset là **best-effort**. Xóa tay bằng lệnh trong B2.5 rồi chạy lại gate. **Bắt buộc phải sạch**: B7 dựng cụm mới với `initial-cluster-state: new`, và dữ liệu cũ làm member từ chối khởi động hoặc khởi động với cluster ID sai |
| B2 hoặc B9: `kubectl` báo `x509: certificate signed by unknown authority` | `ls -l $HOME/.kube/config`; `kubectl config view --minify` | Kubeconfig cũ trỏ vào cluster mới — đúng mục *Lỗi certificate TLS* của [bài 09](../09-troubleshooting-kubeadm-vi.md). Xóa `$HOME/.kube` theo B2.6 rồi chép lại `admin.conf` theo B9.3. **Đừng** thêm `--insecure-skip-tls-verify` để đi tiếp |
| B4.2: kubelet trên host etcd không lên `active` | `ssh <node> 'journalctl -u kubelet -n 80 --no-pager'`; `systemctl cat kubelet` | Đọc từ trên xuống. **(1)** Lỗi cú pháp YAML trong `kubelet.conf` → viết lại bằng heredoc của B4.1, đừng sửa tay. **(2)** Unit có hai `ExecStart` → thiếu dòng `ExecStart=` rỗng trong `20-etcd-service-manager.conf`. **(3)** Lỗi cgroup driver → đối chiếu `SystemdCgroup = true` theo [A4.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a42-cài-containerd-và-runc-đúng-version) |
| B6.3: `scp -r /tmp/${H}/*` báo `Permission denied` | `ls -l /tmp/192.168.100.232` | Thư mục còn thuộc `root` vì các lệnh `kubeadm init phase certs` chạy bằng `sudo`. Chạy lệnh `chown` đầu B6.3 rồi `scp` lại. **Không** `scp` bằng `sudo scp` — khóa SSH nằm ở home của user thường |
| B6.3: `mv pki /etc/kubernetes/` tạo ra `/etc/kubernetes/pki/pki` | `sudo ls -R /etc/kubernetes/pki \| head` | Thư mục đích đã tồn tại từ trước, tức gate ngay trên `mv` đã FAIL mà bạn đi tiếp. Xóa `/etc/kubernetes/pki` trên đúng host đó, chạy lại B6.3 từ đầu cho host đó — certificate trong `/tmp` trên `lab-ha-1` vẫn còn nên không phải sinh lại |
| B7.3: container etcd không lên sau nhiều vòng | `ssh <node> 'journalctl -u kubelet -n 80 --no-pager'`; `ssh <node> 'sudo crictl ps -a'`; `ssh <node> 'sudo crictl logs <id>'` | **(1)** Log etcd báo `cluster ID mismatch` hoặc `member already bootstrapped` → `/var/lib/etcd` chưa sạch, quay lại B2.5. **(2)** Log báo không kết nối được peer → hai host chưa thông port `2380`; kiểm UFW theo [A4.4](LAB-00-MOI-TRUONG-1.35.7.md#a44-firewall-cho-mạng-lab-cô-lập). **(3)** `ImagePullBackOff` → chạy lại phần pull image ở B7.1 |
| B7.3: image trong manifest khác `$ETCD_IMG` | `sudo grep image: /etc/kubernetes/manifests/etcd.yaml`; `kubeadm version -o short` trên từng host | Một trong ba host cài `kubeadm` khác version. Sửa ở tầng package theo [A4.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl), rồi xóa `etcd.yaml` trên host đó và chạy lại `kubeadm init phase etcd local`. **Không** sửa tay tag trong manifest |
| B8.1: `crictl ps --name etcd` không trả về id | `sudo crictl ps -a --name etcd`; `sudo crictl pods` | Container chưa `Running` — quay lại B7.3. Nếu nó ở trạng thái `Exited`, đọc `sudo crictl logs` của chính container đó trước khi làm gì khác |
| B8.2: `endpoint health` chỉ khỏe 1/3 hoặc 2/3 | `ectl member list`; `ssh <host thiếu> 'sudo crictl ps --name etcd'` | Đếm xem host nào vắng mặt rồi xử lý host đó theo dòng B7.3 ở trên. Đừng chạy `member add`/`member remove` để "sửa" — hai lệnh đó nằm ngoài phạm vi lab, xem [mục 2.4](#24-các-quy-ước-còn-lại) |
| **B9.3: `kubeadm init` dừng ở giai đoạn chờ control plane khỏe** | Trên `lab-ha-lb`: `grep server /etc/haproxy/haproxy.cfg`; trên `lab-ha-4`: `sudo crictl ps -a`; `sudo crictl logs <id apiserver>` | Ba nguyên nhân theo thứ tự thường gặp. **(1)** Backend HAProxy chưa đổi → quay lại B3.2. **(2)** Log apiserver báo không kết nối được etcd → kiểm ba file certificate ở B9.1 và `endpoint health` ở B8.2. **(3)** `--etcd-servers` sai địa chỉ → sửa `kubeadm-config.yaml`, `sudo kubeadm reset -f`, rồi `init` lại |
| B9.3: log apiserver có lỗi TLS khi nối tới etcd | `sudo crictl logs <id apiserver> \| grep -i etcd`; trên host etcd: `sudo openssl x509 -noout -text -in /etc/kubernetes/pki/etcd/server.crt \| grep -A1 'Alternative Name'` | SAN của `server.crt` không chứa IP mà apiserver đang gọi tới, hoặc `apiserver-etcd-client.crt` không do CA của cụm etcd ký. Cả hai đều là lỗi ở B5/B6: chạy lại B6.4 để đối chiếu SAN từng host trước khi sinh lại certificate |
| B10.2: `kubeadm join --control-plane` báo lỗi giải mã certificate | Thời gian giữa `upload-certs` và `join`; `kubectl -n kube-system get secret kubeadm-certs` | Secret `kubeadm-certs` và khóa giải mã **hết hạn sau hai giờ** (bài 08). Sinh lại theo B10.1 — đó chính là công dụng của `kubeadm init phase upload-certs --upload-certs` — rồi join lại |
| B12.2: Pod `web` kẹt `Pending` | `kubectl -n lab-8c describe pod <ten> \| tail -20` | Nếu lý do là `untolerated taint node-role.kubernetes.io/control-plane` thì B12.1 chưa chạy hoặc chạy hụt — kiểm `TAINT_LEFT` rồi gỡ taint lại. Nếu là `Insufficient cpu/memory` thì hai control plane node không đủ chỗ cho 4 replica; giảm `replicas` xuống 2 rồi chạy lại B12.2 và ghi con số thật vào evidence |
| B13.2: sau khi bật lại hai máy, `endpoint health` mãi không đủ 3/3 | `ehealth`; trên từng host: `sudo crictl ps --name etcd`; `sudo crictl logs <id>` | Tăng số vòng trong `seq` của vòng lặp hồi phục — thời gian member bắt kịp log **phụ thuộc cấu hình**. Nếu log báo `cluster ID mismatch`, đã có ai xóa `/var/lib/etcd` của một member trong lúc thử; đó là lệch state, restore cả sáu VM về `8x-ha-stacked` và làm lại từ B0 |
| B13: `serve_check` báo `SERVE: no` ở chỗ lẽ ra phải `yes` | `kubectl --request-timeout=15s get --raw='/readyz?verbose'`; `kubectl -n lab-8c get pod probe` | Phân biệt hai nguyên nhân: `/readyz` khác `ok` là vấn đề ở control plane hoặc etcd — đo tiếp bằng `ehealth`. `/readyz` là `ok` mà `wget` fail thì vấn đề nằm ở Pod `probe` hoặc Service, không liên quan tới phép thử — tạo lại `probe` theo B12.2 rồi đo lại |
| B14.2: xóa manifest mà `kubectl -n kube-system get pod -l component=kube-apiserver` vẫn trả 2 | `ssh lab-ha-5 'sudo ls -1 /etc/kubernetes/manifests'`; `ssh lab-ha-5 'sudo crictl ps --name kube-apiserver'` | Bạn `mv` nhầm máy, hoặc mirror Pod chưa kịp biến mất. Kiểm thư mục trên đúng `lab-ha-5` trước; tăng số vòng của vòng lặp rồi đo lại. **Không** `kubectl delete pod` — làm thế là bỏ mất bài học về static Pod |
| B15.4: một VM không chuyển sang *Powered off* | Chạy lại khối PowerShell của B15.4; `vmrun -T ws list` | Node đang tắt dịch vụ, chờ thêm rồi chạy lại. Chỉ khi chờ mãi không xong mới dùng `vmrun -T ws stop <vmx> hard`, và khi đó **ghi vào evidence** rằng lần tắt này là cưỡng bức — snapshot chụp sau một lần tắt cưỡng bức có thể mang theo trạng thái ghi dở của etcd |
| B15.5: bộ VM chính có snapshot tiền tố `8x-` | So sánh với danh sách `$vmxMain` trong B0.1 | Bạn đã chụp nhầm mốc của nhánh HA lên bộ VM chính. Xóa đúng snapshot chụp nhầm trong Snapshot Manager của VMware, **giữ nguyên `03-storage-ready`**, rồi chạy lại gate. Đừng xóa snapshot nào khác |

---

## 5. Nguồn chính thức

Các phần giải thích trong thân bài ưu tiên snapshot tài liệu Kubernetes v1.35
(`https://v1-35.docs.kubernetes.io/`) để hành vi, flag và trường cấu hình khớp minor version của
cluster trong [bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa).

- [Kubernetes v1.35 — Options for Highly Available Topology](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)
- [Kubernetes v1.35 — Set up a High Availability etcd Cluster with kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/)
- [Kubernetes v1.35 — Creating Highly Available Clusters with kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [Kubernetes v1.35 — Creating a cluster with kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Kubernetes v1.35 — Installing kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Kubernetes v1.35 — Troubleshooting kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)
- [Kubernetes v1.35 — `kubeadm reset`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
- [Kubernetes v1.35 — `kubeadm init`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)
- [Kubernetes v1.35 — `kubeadm init phase`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init-phase/)
- [Kubernetes v1.35 — `kubeadm join`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/)
- [Kubernetes v1.35 — kubeadm Configuration (v1beta4)](https://v1-35.docs.kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/)
- [Kubernetes v1.35 — Static Pods](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
- [Kubernetes v1.35 — Debugging Kubernetes nodes with crictl](https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)
- [Kubernetes v1.35 — Operating etcd clusters for Kubernetes](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [kubeadm — Options for Software Load Balancing](https://git.k8s.io/kubeadm/docs/ha-considerations.md#options-for-software-load-balancing) (nguồn mà bài 08 dẫn cho phần load balancer)
- [HAProxy — Configuration Manual](https://docs.haproxy.org/) (cú pháp `frontend`/`backend` dùng ở B3)
