# Lab 8b — HA với stacked etcd

> **Điểm bắt đầu:** **không phải snapshot nào của chuỗi chính**. Lab này dựng mới một **bộ sáu VM
> riêng** — khai báo ở [mục 2.1](#21-bộ-vm-riêng-của-nhóm-lab-ha) — và không đụng vào ba VM
> `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2` của
> [chuỗi snapshot chính](README.md#3-chuỗi-snapshot).
> **Điểm kết thúc:** **tạo hai mốc** trên bộ VM riêng: `8x-vm-ready` (sáu VM đã cài xong OS,
> containerd và kubeadm, **chưa** `kubeadm init`) và `8x-ha-stacked` (mốc cuối của lab). Tiền tố
> `8x-` là bắt buộc để không lẫn với chuỗi `01/02/03/04-` của bộ VM chính.
> **Lab trước:** [Lab 8a — Dựng cluster bằng kubeadm](LAB-8A-DUNG-CLUSTER-BANG-KUBEADM.md); lab đó
> phá rồi dựng lại chính ba VM chính và kết thúc bằng restore về `03-storage-ready`. Lab 8b chỉ mở
> ra **sau khi ba VM đó đã tắt**.
> **Lab sau:** [Lab 8c — HA với external etcd](LAB-8C-HA-VOI-EXTERNAL-ETCD.md), dùng lại **đúng bộ
> VM này** và đổi vai từng máy.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục
[Giai đoạn 8 — Dựng cluster bằng kubeadm](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm),
phần **HA**: hai bài [06 — Các lựa chọn topology cho tính sẵn sàng cao](../06-ha-topology-vi.md)
và [08 — Tạo cluster có tính sẵn sàng cao với kubeadm](../08-high-availability-vi.md). Bài
[07 — Thiết lập cluster etcd HA](../07-setup-ha-etcd-with-kubeadm-vi.md) **không thuộc lab này**;
nó là nội dung của [Lab 8c](LAB-8C-HA-VOI-EXTERNAL-ETCD.md). Ba bài nền đã đọc trước đó —
[01](../01-install-kubeadm-vi.md), [02](../02-create-cluster-kubeadm-vi.md),
[09](../09-troubleshooting-kubeadm-vi.md) — và trang thực hành
[214](../214-kubeadm-tasks-vi.md) được dùng lại ở đây chứ không dạy lại.

[Quy trình mở đầu A5.5 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
**không áp dụng nguyên văn** cho lab này: nó gate một cluster ba node đã tồn tại, còn ở đây cluster
chưa có và số node khác. Bản tương đương cho bộ VM HA nằm ở [B13.4](#b134-gate-ngắn-phiên-bản-ha),
chạy **cuối lab** trước khi chụp `8x-ha-stacked`. Gate mở đầu của Lab 8b là một gate khác: chứng
minh trên máy host rằng **không VM nào đang chạy**, để chắc chắn bộ VM chính không bị hai lab HA
kéo theo. Chạy trên **máy host**, PowerShell:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
if (-not (Test-Path $vmrun)) { 'FAIL: khong tim thay vmrun.exe — sua duong dan' }
$running = & $vmrun -T ws list | Select-Object -Skip 1 | Where-Object { $_ -match '\.vmx$' }
if ($running.Count -eq 0) { 'PASS: khong VM nao dang chay' }
else { "FAIL: dang chay $($running.Count) VM -> $($running -join ', ')" }
```

**PASS:** dòng `PASS: khong VM nao dang chay` xuất hiện. Còn VM nào chạy thì tắt nó trước; lab này
cấp phát gần hết RAM khả dụng của host và không chịu được máy ảo lạ chạy song song. Giữ nguyên cửa
sổ PowerShell này — biến `$vmrun` được dùng lại ở [mục 2.2](#22-gate-bộ-vm-chính-đã-tắt-và-còn-nguyên-mốc)
và ở nhiều bước sau.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Vẽ lại được **ranh giới** hai topology của bài [06](../06-ha-topology-vi.md): trong stacked, etcd
  member cục bộ chỉ nói chuyện với `kube-apiserver` của **chính node đó**; trong external, mỗi etcd
  host nói chuyện với `kube-apiserver` của **từng** control plane node.
- Nêu đúng **số host tối thiểu** của mỗi topology theo bài 06, và tính ra được vì sao bộ sáu VM của
  lab đủ cho stacked nhưng **không** đủ cho external nếu giữ nguyên vai của từng máy.
- Chứng minh bằng cluster thật rằng đây là topology **stacked**: etcd chạy dạng **static Pod trên
  chính** control plane node, và **số member etcd bằng số control plane node**.
- Giải thích được **hỏng theo cặp**: mất một node stacked là mất đồng thời một etcd member *và* một
  instance control plane — và chỉ ra được cả hai mất mát đó trong evidence của mình.
- Dựng được load balancer đứng trước ba `kube-apiserver`: chuyển tiếp **TCP**, health check là kiểm
  tra TCP trên port apiserver, và địa chỉ của nó **khớp** với `controlPlaneEndpoint` của kubeadm.
- Đọc đúng kết quả kiểm tra kết nối trước khi `init`: **connection refused là bình thường**, còn
  **timeout nghĩa là load balancer không giao tiếp được với control plane node**.
- Chỉ ra `--upload-certs` để lại **hiện vật gì** trong cluster, khóa giải mã có hình dạng thế nào,
  **hết hạn sau bao lâu** theo đúng bài 08, và lệnh nào tạo lại nó.
- Phân biệt dứt khoát **hai lệnh join** mà `kubeadm init` in ra, và chứng minh chúng chỉ khác nhau
  đúng **hai cờ**.
- Chứng minh load balancer thật sự **phân phối** request tới cả ba API server, không phải chỉ dính
  vào một máy.
- Chứng minh cluster **vẫn phục vụ** khi mất một control plane node, và vẫn phục vụ khi một
  `kube-apiserver` bị xoá manifest — rồi đưa cả hai về nguyên trạng.
- Chứng minh cluster **mất quorum** khi mất hai trong ba control plane node: API ngừng phục vụ,
  trong khi workload đang chạy vẫn trả lời — và nói được vì sao hai điều đó không mâu thuẫn.
- Nói thẳng được **cái lab này chưa HA**: một load balancer duy nhất là SPOF, và chứng minh điều đó
  bằng chính bộ VM của mình.
- Kết thúc với cluster HA sạch workload, hai mốc `8x-vm-ready` và `8x-ha-stacked` tồn tại trên đủ
  sáu VM, và bộ VM chính còn nguyên mốc `03-storage-ready`.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm | Phần lab kiểm chứng |
| --- | --- |
| [06 — Các lựa chọn topology cho tính sẵn sàng cao](../06-ha-topology-vi.md) | **B2** — số host tối thiểu của hai topology, tính trên chính sáu VM vừa dựng; vì sao đổi vai máy là bắt buộc cho Lab 8c. **B7** — bằng chứng stacked trên cluster thật: ba Pod `component=etcd` là static Pod, mỗi Pod nằm trên đúng control plane node của nó, số member etcd bằng số control plane node. **B9** và **B11** — hỏng theo cặp: một lần mất node làm giảm đồng thời số control plane instance và số etcd member, và ranh giới quorum nằm ở đâu |
| [08 — Tạo cluster có tính sẵn sàng cao với kubeadm](../08-high-availability-vi.md) | **B3** — dựng load balancer chuyển tiếp TCP tới ba control plane node, health check TCP trên port apiserver, và đọc đúng `connection refused` / timeout trước khi init. **B4** — `kubeadm init --control-plane-endpoint … --upload-certs`, Secret `kubeadm-certs`, hạn hai giờ, hình dạng certificate key, và phép so hai lệnh join. **B5** — cài CNI ngay sau init, và ảnh chụp CoreDNS dồn trên node đầu tiên. **B6** — join hai control plane bằng `--control-plane --certificate-key`, join hai worker bằng lệnh không có hai cờ đó, `rollout restart deployment coredns`, và bằng chứng load balancer phân phối tới cả ba API server. **B10** — một `kube-apiserver` chết mà cluster vẫn phục vụ, hệ quả trực tiếp của việc có load balancer đứng trước |
| [01 — Cài đặt kubeadm](../01-install-kubeadm-vi.md) *(bài nền)* | **B0** — yêu cầu tối thiểu cho máy control plane lấy từ mục *Trước khi bạn bắt đầu* của bài; hostname, MAC và `product_uuid` duy nhất cho từng node trong sáu VM mới |
| [02 — Tạo một cluster với kubeadm](../02-create-cluster-kubeadm-vi.md) *(bài nền)* | **B4** — khác nhau giữa `--apiserver-advertise-address` và `--control-plane-endpoint` theo đúng mục *Những lưu ý* của bài; **B6** — taint cách ly control plane vẫn còn trên cả ba node; **B2** — mục *Khả năng chống chịu của cluster* nói cluster một control plane có thể mất dữ liệu, chính là thứ lab này gỡ bỏ |
| [09 — Xử lý sự cố kubeadm](../09-troubleshooting-kubeadm-vi.md) *(bài nền)* | [Mục 4](#4-troubleshooting-của-lab-này) — dùng đúng công dụng của một trang tra cứu: mỗi dòng sự cố kubeadm trong bảng troubleshooting trỏ về đúng mục của bài |
| [214 — Quản trị với kubeadm](../214-kubeadm-tasks-vi.md) *(bài thực hành)* | **B6** — thao tác *Thêm worker node Linux* của trang mục lục này; các trang con còn lại thuộc giai đoạn 17–20 |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài 08 — **cặp load balancer + VIP** để chính load balancer cũng HA | Lab dùng **đúng một** VM làm load balancer, nên nó là **SPOF**: LB chết là mọi client mất endpoint dù ba API server vẫn sống. Bài 08 đặt yêu cầu "load balancer phải phân phối tới tất cả control plane node khỏe mạnh" và trỏ sang hướng dẫn *Các lựa chọn cho cân bằng tải bằng phần mềm* cho phần dự phòng; production cần **hai** máy LB cộng một **địa chỉ VIP** trôi giữa chúng (keepalived/VRRP). Lab không dựng vì cần thêm một VM thứ bảy và một IP thứ bảy dành riêng cho VIP, vượt ngân sách tài nguyên ở [mục 2.3](#23-gate-tài-nguyên-host). **B12 chứng minh SPOF này bằng thực nghiệm** thay vì giả vờ đã HA hoàn chỉnh |
| Bài 08 — **tên DNS** cho load balancer | Bài khuyến nghị load balancer có **tên phân giải được qua DNS** và không dùng trực tiếp địa chỉ IP trong môi trường cloud. Mạng lab không có DNS server nội bộ; lab dùng `/etc/hosts` như [A2.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a23-đặt-hostname-etchosts-và-kiểm-tra-identity) đã làm, nên `controlPlaneEndpoint` vẫn là **tên** `lab-ha-lb:6443` chứ không phải IP trần — phần khác biệt còn lại chỉ là nguồn phân giải |
| Bài 08 — nhánh **external etcd** trọn vẹn: `07`, `scp` ba file certificate, `kubeadm-config.yaml` có `etcd.external.endpoints`, ràng buộc `--config` không dùng chung `--certificate-key` | Đây là **nội dung của Lab 8c**, chạy trên đúng bộ VM này sau khi reset. Bài 06 nói rõ hai topology là **hai lựa chọn**, và bài 08 rẽ đôi theo lựa chọn đó; làm cả hai trong một lab là làm nhầm việc |
| Bài 08 — mục **Phân phối certificate thủ công** (`ssh-agent`, `scp` bộ certificate, `kubeadm certs certificate-key` dùng làm khóa cố định) | Chỉ cần khi **cố ý bỏ** `--upload-certs`. Lab đi đúng đường mặc định của bài. Riêng `kubeadm certs certificate-key` vẫn được chạy ở B4 để đọc hình dạng khóa, vì lệnh đó không đụng vào cluster. Phần sao chép certificate thủ công thuộc [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ) |
| Bài 08 — cảnh báo về **cloud provider**, Service `LoadBalancer`, PersistentVolume động | Lab chạy on-premises trên VM tự dựng, không có cloud provider để gọi. Đây là giới hạn về bản chất, không phải thiếu cấu hình |
| Bài 08 — mục **Các container image** khi host không pull được image | Sáu VM đều có HTTPS egress, đã được gate ở B0. Kịch bản air-gapped cần một registry nội bộ, tức thêm hạ tầng ngoài baseline |
| Bài 06 — **backup và khôi phục** dữ liệu etcd của cluster HA | Đây là **[nợ #8](README.md#5-sổ-nợ-lab)** của lộ trình, phát sinh ở chính giai đoạn 8 và **chưa được trả ở lab này**. Lab 8b **không** chạy `etcdctl snapshot save`, không dựng quy trình khôi phục; đường lui duy nhất vẫn là snapshot VM. Trả ở [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa) |
| **Nâng cấp** một cluster HA | Bài 08 trỏ sang [tài liệu nâng cấp](../221-kubeadm-upgrade-vi.md) ngay ở đầu vì HA đổi cách nâng cấp control plane. Nội dung đó thuộc [giai đoạn 17](../00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster) |
| **Ghim version cho HAProxy** như bảng A1.3 làm với các thành phần khác | HAProxy **không nằm trong baseline** của lộ trình: nó là phần mềm hạ tầng ngoài Kubernetes, và bài 08 chỉ nói "ví dụ sau đây chỉ là một lựa chọn". Lab cài từ gói của bản phân phối, **không ghim version**, và bù lại bằng cách ghi version quan sát được vào evidence ở B3 cùng gate chứng minh nó đang chạy |
| **Thực thi NetworkPolicy** trên cluster HA | CNI của lab là đúng CNI mà [A5.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a52-cài-flannel-v0289) dùng, và nó **không thực thi NetworkPolicy** — đó là lựa chọn có chủ đích của baseline. Việc đổi CNI là bài học của Lab 5b, chạy trên **bộ VM chính**, không phải ở đây |

Lab này **không** dùng CRD, audit log, mã hóa etcd at rest, `etcdctl snapshot save`, metrics-server,
HPA hay ResourceQuota. Toàn bộ những thứ đó đứng **sau** giai đoạn 8 trong lộ trình. Workload duy
nhất dùng để chứng minh cluster sống là một **Deployment** cộng một **Service**, đã học ở giai
đoạn 4 và 5.

### 1.2. Thời lượng

**4 giờ**, tính từ lúc snapshot `8x-vm-ready` đã tồn tại trên đủ sáu VM.

**B0 là phiên làm việc riêng, không tính trong bốn giờ đó.** Dựng sáu VM mới có khối lượng tương
đương A1–A4 của [Lab 00](LAB-00-MOI-TRUONG-1.35.7.md), tức khoảng 2–4 giờ nữa tùy tốc độ mạng và
tùy bạn cài từng máy hay dùng full clone. Chia làm hai buổi: buổi một chạy hết B0 và chụp
`8x-vm-ready`, buổi hai chạy B1 → B13.

Bốn giờ của buổi hai phân bổ đại khái: B1–B6 dựng cluster HA khoảng một nửa; B7–B8 kiểm chứng
topology và workload khoảng ba mươi phút; B9–B12 bốn kịch bản hỏng, mỗi kịch bản có ít nhất một
chu kỳ tắt–bật máy; B13 cleanup và chụp mốc.

Mọi bước phải chờ trong lab đều viết dưới dạng `kubectl wait --timeout` hoặc **vòng lặp có điều
kiện dừng và số vòng tối đa**. Lab **không** cam kết con số giây nào cho việc hội tụ: thời gian
phụ thuộc cấu hình cluster, tốc độ đĩa của host và số VM đang chạy song song.

---

## 2. Quy ước và an toàn

### 2.1. Bộ VM riêng của nhóm lab HA

Hai lab HA của giai đoạn 8 — **Lab 8b và Lab 8c** — dùng chung **một bộ sáu VM riêng, tách hoàn
toàn** khỏi ba VM của chuỗi snapshot chính. Lab 8b là nơi **định nghĩa** bộ VM đó; Lab 8c link về
đúng mục này thay vì khai lại.

| Vai trò trong Lab 8b | Tên VM / hostname | IP | vCPU | RAM | Disk thin-provisioned |
| --- | --- | --- | --- | --- | --- |
| Load balancer | `lab-ha-lb` | `192.168.100.230` | 1 | 1 GB | 20 GB |
| Control plane 1 — nơi `kubeadm init` | `lab-ha-1` | `192.168.100.231` | 2 | 3 GB | 40 GB |
| Control plane 2 | `lab-ha-2` | `192.168.100.232` | 2 | 3 GB | 40 GB |
| Control plane 3 | `lab-ha-3` | `192.168.100.233` | 2 | 3 GB | 40 GB |
| Worker 1 | `lab-ha-4` | `192.168.100.234` | 2 | 2 GB | 40 GB |
| Worker 2 / fault target | `lab-ha-5` | `192.168.100.235` | 2 | 2 GB | 40 GB |

**Tổng cấp phát: 11 vCPU, 14 GB RAM, 220 GB đĩa danh nghĩa.** Đĩa là thin-provisioned nên dung
lượng chiếm thật nhỏ hơn nhiều; [mục 2.3](#23-gate-tài-nguyên-host) gate theo dung lượng còn trống
thật chứ không theo con số danh nghĩa.

Năm máy `lab-ha-1` … `lab-ha-5` cấu hình **giống hệt nhau về phần mềm**: cùng OS, cùng containerd,
cùng `kubeadm`/`kubelet`/`kubectl`, chỉ khác hostname và IP. `lab-ha-lb` **không phải node
Kubernetes**: nó chỉ cần OS và HAProxy, không cài containerd, không cài kubeadm, không tắt swap.

Mức RAM của ba control plane node cao hơn **mức tối thiểu** mà bài
[01 — Cài đặt kubeadm](../01-install-kubeadm-vi.md) đặt ra ở mục *Trước khi bạn bắt đầu*: **2 GB
RAM trở lên cho mỗi máy** và **2 CPU trở lên cho các máy control plane**. Bài nói thẳng rằng dưới
2 GB thì "còn rất ít dung lượng cho các ứng dụng của bạn"; ở đây mỗi control plane node còn phải
gánh thêm **một etcd member** ngay trên chính nó, nên lab cấp 3 GB. Nếu host có từ 32 GB RAM trở
lên, nâng ba máy này lên 4 GB; **không hạ xuống dưới 2 GB** vì đó là chạm đúng ngưỡng tối thiểu
của bài 01 và kubeadm sẽ cảnh báo ở bước preflight.

**Vì sao tên máy trung tính.** Tên `lab-ha-N` **cố ý không mang vai trò**. Lab 8c dựng topology
external etcd trên **đúng sáu VM này**: `lab-ha-1`, `lab-ha-2`, `lab-ha-3` đổi vai thành **node
etcd**, còn `lab-ha-4`, `lab-ha-5` đổi vai thành **control plane**. Nếu đặt tên theo vai
(`lab-ha-cp1`, `lab-ha-worker1`) thì Lab 8c hoặc phải đổi tên VM — kéo theo đổi hostname, đổi
`/etc/hosts`, đổi tên Node trong cluster — hoặc phải sống với tên nói dối. Tên trung tính giải
được cả hai. Đừng đổi tên trong Lab 8b vì bất kỳ lý do gì.

Ba nguyên tắc bắt buộc của bộ VM này:

- **Không đụng vào ba VM của chuỗi chính.** Không xoá, không đổi tên, không sửa snapshot của
  `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`.
- **Snapshot của bộ VM HA luôn mang tiền tố `8x-`.** Lab 8b tạo `8x-vm-ready` và `8x-ha-stacked`;
  Lab 8c tạo `8x-ha-external`. Tên phải đúng nguyên văn trên **cả sáu VM**, không hậu tố theo máy.
- **Dải IP `.230`–`.235` phải nằm ngoài DHCP pool** của router, đúng như
  [A3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a3-đặt-ip-tĩnh-và-phân-giải-tên) yêu cầu với dải
  `.221`–`.223`. Nếu LAN của bạn đã dùng dải này, đổi cả sáu IP cùng lúc và sửa mọi chỗ trong lab.

### 2.2. Gate: bộ VM chính đã tắt và còn nguyên mốc

Lab 8b cấp phát 14 GB RAM. Ba VM của chuỗi chính cấp phát 20 GB
([bảng A1.2](LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm)). **Hai bộ không chạy song song được**, và quan
trọng hơn: hai lab HA phải không có cơ hội chạm vào cluster của chuỗi chính.

Chạy trên **máy host**, PowerShell — dùng lại `$vmrun` đã đặt ở đầu file:

```powershell
$mainVmx = @(
  'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
  'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
  'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
)
$running = & $vmrun -T ws list | Select-Object -Skip 1

foreach ($f in $mainVmx) {
  if (-not (Test-Path $f))      { "FAIL: khong tim thay $f — KHONG duoc xoa VM chuoi chinh"; continue }
  if ($running -contains $f)    { "FAIL: $f dang chay — tat truoc khi mo Lab 8b"; continue }
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '03-storage-ready') { "PASS: $f — powered off, con moc 03-storage-ready" }
  else { "FAIL: $f -> $($names -join ', ')" }
}
```

**PASS:** đúng ba dòng `PASS:`, không dòng `FAIL:` nào. Đường dẫn `.vmx` theo máy host đang dùng;
sửa lại nếu VM nằm chỗ khác. Ba điều kiện được gate ở đây là ba điều kiện khác nhau: file `.vmx`
**còn tồn tại** (không ai xoá VM), VM **đang tắt**, và mốc `03-storage-ready` **còn nguyên** để
chuỗi chính có đường lui sau khi hai lab HA kết thúc.

Không mở B0 khi còn dòng `FAIL:`. Thiếu mốc `03-storage-ready` thì phần việc phải làm là chạy lại
Lab 6a trên chuỗi chính, **không phải** đi tiếp Lab 8b.

### 2.3. Gate tài nguyên host

Sáu VM là bước nhảy lớn so với ba VM của Lab 00. Gate này tính tổng cấp phát của bộ VM HA rồi so
với tài nguyên thật của host và **dừng** nếu không đủ. Chạy trên **máy host**, PowerShell:

```powershell
# Tong cap phat cua bo VM HA, lay tu bang o muc 2.1
$vmVcpu   = 11
$vmRamGb  = 14
$vmDiskGb = 220     # danh nghia; thin-provisioned nen chiem that it hon nhieu
$diskNeedGb = 100   # yeu cau thuc dung cho 6 VM + snapshot

$cpu  = Get-CimInstance Win32_Processor
$lp   = ($cpu | Measure-Object -Property NumberOfLogicalProcessors -Sum).Sum
$virt = ($cpu | Select-Object -First 1).VirtualizationFirmwareEnabled
$ramGb = [math]::Round((Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory / 1GB, 1)

# Doi chu 'E' thanh o dia thuc su chua thu muc Virtual Machines cua ban
$vol = Get-Volume -DriveLetter 'E'
$freeGb = [math]::Round($vol.SizeRemaining / 1GB, 1)

"host: $lp logical CPU | $ramGb GB RAM | $freeGb GB trong tren o $($vol.DriveLetter): | virt=$virt"
"can : $vmVcpu vCPU | $vmRamGb GB RAM cho VM | >= $diskNeedGb GB trong (danh nghia $vmDiskGb GB)"

if ($virt -ne $true)              { 'FAIL: virtualization chua bat trong firmware' }
if ($lp -lt 8)                    { "FAIL: chi co $lp logical CPU, can >= 8" }
if ($ramGb -lt ($vmRamGb + 6))    { "FAIL: RAM host $ramGb GB < $($vmRamGb + 6) GB (14 cho VM + 6 cho Windows)" }
if ($freeGb -lt $diskNeedGb)      { "FAIL: chi con $freeGb GB trong, can >= $diskNeedGb GB" }
if ($virt -eq $true -and $lp -ge 8 -and $ramGb -ge ($vmRamGb + 6) -and $freeGb -ge $diskNeedGb) {
  'PASS: host du tai nguyen cho bo VM HA'
}
```

**PASS:** dòng `PASS: host du tai nguyen cho bo VM HA` xuất hiện và **không** dòng `FAIL:` nào.

Ba ghi chú về các con số trên, để bạn sửa cho đúng máy mình thay vì chép mù:

- **11 vCPU trên 8 logical CPU là over-subscribe có chủ ý.** Chấp nhận được vì sáu VM lab phần lớn
  thời gian ở trạng thái rỗi và không có VM nào chạy benchmark. Host dưới 8 logical CPU thì hạ hai
  worker xuống 1 vCPU **trước khi** dựng máy; đừng hạ control plane, vì 2 CPU là mức tối thiểu bài
  [01](../01-install-kubeadm-vi.md) đặt ra cho máy control plane.
- **6 GB dành cho Windows** là con số thực dụng, không phải yêu cầu của Kubernetes. Máy host còn
  chạy trình duyệt, terminal và VMware Workstation trong lúc bạn làm lab.
- **100 GB trống** là yêu cầu thực dụng cho sáu đĩa thin-provisioned cộng hai mốc snapshot; con số
  danh nghĩa 220 GB chỉ là trần. Snapshot ăn thêm dung lượng theo mức thay đổi sau khi chụp, nên
  đừng chạy lab khi ổ đã gần đầy.

Không đủ tài nguyên thì **dừng ở đây**. Cách xử lý đúng là bổ sung RAM hoặc dọn đĩa cho host, không
phải cắt bớt số control plane node — cluster hai control plane node **không** phải cluster HA theo
bài [06](../06-ha-topology-vi.md), và toàn bộ phần B9–B11 của lab sẽ mất ý nghĩa.

### 2.4. Quy ước chạy lệnh và bằng chứng

- **Node mặc định để gõ lệnh là `lab-ha-1`.** Mọi khối lệnh không ghi rõ node đều chạy ở đó, bằng
  user quản trị có kubeconfig. Khối nào chạy ở nơi khác đều mở đầu bằng dòng
  "Chạy trên `<tên máy>`" hoặc "Chạy trên máy host".
- **Bằng chứng ghi vào `~/lab-evidence/8b/` trên `lab-ha-1`**, manifest tạm vào `~/lab-work/8b/`.
  Hai biến `EV` và `WK` được đặt ở B1 và dùng suốt phần B.
- **Chạy toàn bộ B1 → B13 trong cùng một phiên shell trên `lab-ha-1`.** Gần như mọi gate so sánh
  với biến đặt ở B1; mở shell mới giữa chừng là mất hết. Phiên này nằm trên `lab-ha-1` — máy duy
  nhất **không bao giờ bị tắt** trong lab.
- **Lệnh cần quyền root trên máy khác thì gõ trực tiếp trên máy đó**, không bọc qua
  `ssh <node> 'sudo …'`. Ubuntu cài từ ISO mặc định đòi mật khẩu cho `sudo`, và `sudo` không có tty
  sẽ hỏng giữa chừng. Lab chỉ dùng `ssh` cho lệnh **đọc** không cần root.
- **Dòng bắt đầu bằng `**PASS:**`** mô tả điều kiện phải đạt. Gate fail thì không đi tiếp.
- **Fault injection trên worker chỉ chạy trên `lab-ha-5`.** `lab-ha-4` giữ vai worker lành để
  workload còn chỗ chạy. Trong lab này không kịch bản nào cần tắt worker, nhưng quy ước vẫn áp dụng
  nếu bạn tự thêm bước.

### 2.5. Trình tự an toàn khi tắt và bật máy

Lab tắt và bật máy **bốn lần** — B9 (một control plane node), B11 (hai control plane node), B12
(load balancer), cộng lần tắt cuối cùng ở B13. Ba quy tắc không được vi phạm:

1. **`lab-ha-1` không bao giờ bị tắt trong lúc làm lab.** Nó giữ phiên shell, kubeconfig và toàn bộ
   biến. Mất nó là mất đường quan sát.
2. **Tắt máy bằng `sudo shutdown -h now` gõ trên chính máy đó**, rồi xác nhận trạng thái *Powered
   off* trên máy host bằng `vmrun` trước khi bật lại. Chỉ dùng `vmrun … stop … hard` khi
   [mục 4](#4-troubleshooting-của-lab-này) chỉ dẫn, và khi đó phải **ghi vào evidence** rằng lần tắt
   này là cưỡng bức, vì nó đổi bản chất phép thử.
3. **Bật lại bằng `vmrun start` hoặc giao diện VMware, rồi chờ bằng vòng lặp có điều kiện dừng**,
   không đếm giây.

Thứ tự bật cả bộ: `lab-ha-lb` → `lab-ha-1` → `lab-ha-2` → `lab-ha-3` → `lab-ha-4` → `lab-ha-5`.
Thứ tự tắt là **ngược lại**. Load balancer lên trước vì `controlPlaneEndpoint` trỏ vào nó; các node
lên trước LB sẽ có một khoảng không kết nối được endpoint, vô hại nhưng làm log ồn.

**Đường quay lui.** Sau khi có `8x-vm-ready`, mọi sự cố không sửa được trong vài phút đều xử lý
bằng cách tắt **cả sáu VM** và restore **cả sáu** về `8x-vm-ready`, rồi chạy lại từ B1. Không bao
giờ restore riêng một VM: cluster HA có ba bản etcd đang đồng bộ với nhau, restore lệch một máy tạo
ra một member mang dữ liệu quá khứ và tình huống đó không có trong bất kỳ bài nào của giai đoạn 8.
Nguyên tắc này là bản mở rộng của ghi chú cuối
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).

---
# Phần B — Thực hành kiến thức giai đoạn 8 (HA stacked)

## B0. Dựng sáu VM và mốc `8x-vm-ready`

**Mục đích:** có sáu máy đúng bảng [mục 2.1](#21-bộ-vm-riêng-của-nhóm-lab-ha), phần mềm giống hệt
bộ VM chính, **chưa** chạy `kubeadm init`, và một mốc snapshot để quay lui.

**Lab này không lặp lại phần dựng môi trường.** Toàn bộ quy trình cài OS, chuẩn hóa identity, đặt
IP tĩnh, cài containerd và cài kubeadm đã có ở phần A của
[Lab 00](LAB-00-MOI-TRUONG-1.35.7.md); B0 chỉ liệt kê **những chỗ khác**. Đừng chép hướng dẫn cài
đặt vào đây, và đừng làm theo trí nhớ — mở Lab 00 ra đọc song song.

### B0.1. Tạo sáu VM

Làm theo [A1.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm) cho **năm thiết lập VM**: guest OS
Ubuntu 64-bit, firmware UEFI, network Bridged (hoặc một VMnet NAT riêng), disk thin provision, và
quy tắc chuẩn hóa identity khi dùng full clone. Bốn chỗ khác so với Lab 00:

| Điểm | Lab 00 | Lab 8b |
| --- | --- | --- |
| Số VM | 3 | **6** |
| Tên máy | `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2` | `lab-ha-lb`, `lab-ha-1` … `lab-ha-5` |
| Dải IP | `192.168.100.221`–`.223` | `192.168.100.230`–`.235` |
| Spec từng máy | bảng A1.2 | bảng [mục 2.1](#21-bộ-vm-riêng-của-nhóm-lab-ha) — LB nhỏ hơn hẳn |

ISO và checksum lấy đúng ở [A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); **không
chép lại con số version vào lab này**. Bản cài phải có: minimal installation, không GUI, bật
OpenSSH Server, cùng một user quản trị có `sudo`, giống hệt
[A2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a2-cài-ubuntu-và-cấu-hình-identity).

Sáu VM là số máy lớn; **dùng full clone từ một VM nguồn** sẽ nhanh hơn nhiều so với cài sáu lần.
Nếu chọn cách đó thì bắt buộc chạy
[A2.2](LAB-00-MOI-TRUONG-1.35.7.md#a22-vm-được-tạo-bằng-full-clone) trên **từng clone** trước khi
đặt hostname: mọi VM phải có `machine-id`, SSH host key, MAC và `product_uuid` riêng. Bài
[01 — Cài đặt kubeadm](../01-install-kubeadm-vi.md) nói rõ Kubernetes dùng ba giá trị hostname, MAC
và `product_uuid` để **định danh duy nhất** node; trùng nhau thì quá trình cài có thể thất bại. Với
sáu máy sinh ra từ một khuôn, rủi ro này cao hơn hẳn so với ba máy của Lab 00.

**PASS:** VMware Workstation hiển thị đủ sáu VM, mỗi VM đúng số vCPU, RAM và disk của bảng mục 2.1.

### B0.2. Hostname, `/etc/hosts` và IP tĩnh

Đặt hostname theo [A2.3](LAB-00-MOI-TRUONG-1.35.7.md#a23-đặt-hostname-etchosts-và-kiểm-tra-identity),
chỉ chạy đúng một lệnh phù hợp trên từng VM:

```bash
# Chi chay dung mot lenh phu hop tren tung VM
sudo hostnamectl set-hostname lab-ha-lb
sudo hostnamectl set-hostname lab-ha-1
sudo hostnamectl set-hostname lab-ha-2
sudo hostnamectl set-hostname lab-ha-3
sudo hostnamectl set-hostname lab-ha-4
sudo hostnamectl set-hostname lab-ha-5
```

Thêm giống nhau trên **cả sáu VM** — bao gồm cả `lab-ha-lb`, vì lát nữa bạn còn kiểm tra kết nối từ
LB tới các control plane node:

```bash
sudo tee -a /etc/hosts >/dev/null <<'EOF'
192.168.100.230 lab-ha-lb
192.168.100.231 lab-ha-1
192.168.100.232 lab-ha-2
192.168.100.233 lab-ha-3
192.168.100.234 lab-ha-4
192.168.100.235 lab-ha-5
EOF
```

IP tĩnh làm đúng [A3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a3-đặt-ip-tĩnh-và-phân-giải-tên): chặn
cloud-init quản lý mạng, bỏ file netplan do cloud-init sinh, tạo `/etc/netplan/01-static.yaml`,
`chmod 600`, `netplan apply`, reboot rồi verify. Khác biệt duy nhất là **địa chỉ**: `.230` cho
`lab-ha-lb`, rồi `.231`–`.235` cho `lab-ha-1`–`lab-ha-5`.

**PASS:** trên cả sáu VM, khối verify cuối A3 đạt — file chặn cloud-init tồn tại, xuất hiện dòng
`PASS: 50-cloud-init.yaml is absent`, IP `.230`–`.235` giữ nguyên sau reboot, default route đúng,
ping được gateway và Internet, DNS phân giải được.

### B0.3. Phần mềm: năm node giống Lab 00, LB thì không

**Trên năm máy `lab-ha-1` … `lab-ha-5`** chạy nguyên
[mục A4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a4-chuẩn-bị-os-và-container-runtime), đủ bốn mục
con và đủ các gate của chúng:

| Mục của Lab 00 | Nội dung | Chạy trên |
| --- | --- | --- |
| [A4.1](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites) | cập nhật OS, NTP, tắt swap, module và sysctl | `lab-ha-1` … `lab-ha-5` |
| [A4.2](LAB-00-MOI-TRUONG-1.35.7.md#a42-cài-containerd-và-runc-đúng-version) | containerd + runc đúng version, `SystemdCgroup = true` | `lab-ha-1` … `lab-ha-5` |
| [A4.3](LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl) | `kubeadm`, `kubelet`, `kubectl`, `crictl`, `apt-mark hold` | `lab-ha-1` … `lab-ha-5` |
| [A4.4](LAB-00-MOI-TRUONG-1.35.7.md#a44-firewall-cho-mạng-lab-cô-lập) | tắt UFW | **cả sáu máy**, kể cả `lab-ha-lb` |

**`lab-ha-lb` không phải node Kubernetes.** Nó **không** cài containerd, **không** cài
kubeadm/kubelet/kubectl, **không** cần tắt swap và **không** cần module `br_netfilter`. Trên máy đó
chỉ chạy đúng bốn việc:

```bash
# Chay tren lab-ha-lb
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y ca-certificates curl
sudo timedatectl set-ntp true
sudo ufw disable
```

**PASS:** trên năm node, toàn bộ gate của A4.1–A4.4 đạt — quan trọng nhất là `swapon --show` rỗng,
`containerd` và `kubelet` đều `enabled`, `crictl version` trả `RuntimeApiVersion: v1`, và ba binary
Kubernetes cùng version với bảng A1.3. Trên `lab-ha-lb`, `ufw status` là `inactive`, `curl` chạy
được và `timedatectl` báo `System clock synchronized: yes`.

Đồng hồ lệch giữa các máy làm hỏng certificate và làm etcd bầu leader sai; với ba etcd member
xếp chồng, hậu quả nặng hơn nhiều so với cluster một control plane. Đừng bỏ bước NTP trên bất kỳ
máy nào, kể cả LB.

### B0.4. SSH không mật khẩu, và gate identity của sáu máy

Lab đọc trạng thái từ năm máy khác trong suốt phần B. Tạo khóa SSH trên `lab-ha-1` và phát cho cả
sáu máy — kể cả chính nó, để vòng lặp không phải có ngoại lệ:

```bash
# Chay tren lab-ha-1
test -f ~/.ssh/id_ed25519 || ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
for n in lab-ha-lb lab-ha-1 lab-ha-2 lab-ha-3 lab-ha-4 lab-ha-5; do
  ssh-copy-id -o StrictHostKeyChecking=accept-new "$n"
done
```

Lệnh trên hỏi mật khẩu user một lần cho mỗi máy. Sau đó gate:

```bash
# Chay tren lab-ha-1
OK=0
for n in lab-ha-lb lab-ha-1 lab-ha-2 lab-ha-3 lab-ha-4 lab-ha-5; do
  H="$(ssh -o BatchMode=yes -o ConnectTimeout=5 "$n" hostnamectl --static 2>/dev/null)"
  echo "$n -> $H"
  test "$H" = "$n" && OK=$(( OK + 1 ))
done
echo "so may tra dung hostname qua SSH khong mat khau: $OK"
test "$OK" -eq 6 && echo 'PASS: SSH khong mat khau toi ca sau may, hostname dung'
```

**PASS:** dòng `PASS: SSH khong mat khau toi ca sau may, hostname dung` xuất hiện.

Bây giờ gate identity. `product_uuid` chỉ root đọc được, nên **chạy khối này trên từng máy trong sáu
máy** để ghi ra một dòng định danh mà user thường đọc được:

```bash
# Chay tren TUNG may trong sau may
sudo cat /sys/class/dmi/id/product_uuid > /tmp/lab8b-uuid.txt
IFACE="$(ip route | awk '/default/ {print $5; exit}')"
printf '%s %s %s %s\n' \
  "$(hostnamectl --static)" "$(cat /etc/machine-id)" \
  "$(tr -d '\n' < /tmp/lab8b-uuid.txt)" \
  "$(ip -br link show "$IFACE" | awk '{print $3}')" \
  > /tmp/lab8b-identity.txt
cat /tmp/lab8b-identity.txt
```

Gom lại và gate trên `lab-ha-1`:

```bash
# Chay tren lab-ha-1
for n in lab-ha-lb lab-ha-1 lab-ha-2 lab-ha-3 lab-ha-4 lab-ha-5; do
  ssh -o BatchMode=yes "$n" cat /tmp/lab8b-identity.txt
done | tee /tmp/lab8b-identity-all.txt

UNIQ_OK=0
for col in 1 2 3 4; do
  N="$(awk -v c="$col" '{print $c}' /tmp/lab8b-identity-all.txt | sort -u | wc -l)"
  echo "cot $col co $N gia tri khac nhau"
  if test "$N" -eq 6; then UNIQ_OK=$(( UNIQ_OK + 1 )); else echo "FAIL: cot $col bi trung giua cac may"; fi
done
test "$UNIQ_OK" -eq 4 \
  && echo 'PASS: hostname, machine-id, product_uuid va MAC deu duy nhat tren sau may'
```

**PASS:** cả bốn cột đều có 6 giá trị khác nhau, không dòng `FAIL:` nào, và dòng
`PASS: hostname, machine-id, product_uuid va MAC deu duy nhat tren sau may` xuất hiện.

Trùng `product_uuid` hoặc trùng MAC thì **dừng tại đây** và để VMware sinh giá trị mới — hai giá trị
này không sửa được bằng lệnh bên trong Ubuntu, đúng như cảnh báo ở
[A2.3](LAB-00-MOI-TRUONG-1.35.7.md#a23-đặt-hostname-etchosts-và-kiểm-tra-identity).

### B0.5. Liên thông sáu máy

Mọi máy phải nói chuyện được hai chiều với mọi máy. Bài
[08](../08-high-availability-vi.md) đặt "kết nối mạng đầy đủ giữa tất cả các máy trong cluster" thành
điều kiện tiên quyết, và với topology HA thì "tất cả" bao gồm cả load balancer.

```bash
# Chay tren lab-ha-1
HOSTS='lab-ha-lb lab-ha-1 lab-ha-2 lab-ha-3 lab-ha-4 lab-ha-5'
FAILN=0
for src in $HOSTS; do
  for dst in $HOSTS; do
    ssh -o BatchMode=yes "$src" "ping -c 1 -W 2 $dst >/dev/null 2>&1" \
      || { echo "FAIL: $src -> $dst"; FAILN=$(( FAILN + 1 )); }
  done
done
echo "so cap khong ping duoc: $FAILN"
test "$FAILN" -eq 0 && echo 'PASS: 36 cap nguon-dich deu ping duoc'
```

**PASS:** dòng `PASS: 36 cap nguon-dich deu ping duoc` xuất hiện. Cặp nào fail thì sửa `/etc/hosts`
(B0.2), IP (A3) hoặc firewall (A4.4) trước khi đi tiếp.

### B0.6. Tắt sáu VM và chụp `8x-vm-ready`

Đây là mốc quay lui của toàn bộ Lab 8b **và** Lab 8c. Chụp khi VM đã tắt để snapshot không kèm
trạng thái RAM, đúng lý do [A5.4.8 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a548-dọn-resource-test-ghi-evidence-và-chụp-snapshot)
nêu.

Tắt theo thứ tự ngược với thứ tự bật ở [mục 2.5](#25-trình-tự-an-toàn-khi-tắt-và-bật-máy) —
`lab-ha-5` → `lab-ha-4` → `lab-ha-3` → `lab-ha-2` → `lab-ha-1` → `lab-ha-lb`. Trên **từng máy**:

```bash
sudo shutdown -h now
```

Chờ VMware Workstation hiển thị cả sáu VM ở trạng thái *Powered off*, rồi chuột phải từng VM →
**Snapshot → Take Snapshot** → ô *Name* điền đúng nguyên văn:

```text
8x-vm-ready
```

Ô *Description* ghi `bo VM HA cua Lab 8b, chua kubeadm init, chup <ngay>`.

Verify trên **máy host**, PowerShell:

```powershell
$haVmx = @(
  'E:\Virtual Machines\lab-ha-lb\lab-ha-lb.vmx'
  'E:\Virtual Machines\lab-ha-1\lab-ha-1.vmx'
  'E:\Virtual Machines\lab-ha-2\lab-ha-2.vmx'
  'E:\Virtual Machines\lab-ha-3\lab-ha-3.vmx'
  'E:\Virtual Machines\lab-ha-4\lab-ha-4.vmx'
  'E:\Virtual Machines\lab-ha-5\lab-ha-5.vmx'
)
foreach ($f in $haVmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '8x-vm-ready') { "PASS: $f" }
  else { "FAIL: $f -> $($names -join ', ')" }
}
```

**PASS:** đúng sáu dòng `PASS:`, không dòng `FAIL:` nào. `-ccontains` phân biệt hoa thường nên gate
này bắt được cả lỗi gõ sai tên. Giữ nguyên biến `$haVmx` trong cửa sổ PowerShell này — B9, B11, B12
và B13 dùng lại.

Buổi làm việc thứ nhất kết thúc ở đây. Bốn giờ của lab tính từ B1.

---

## B1. Mở bộ VM HA, chốt biến và gate đầu vào

**Mục đích:** bật sáu VM đúng thứ tự, đặt toàn bộ biến mà phần B dùng, và chứng minh môi trường
khớp baseline **trước khi** chạm vào kubeadm.

### B1.1. Bật máy đúng thứ tự

Chạy trên **máy host**, PowerShell:

```powershell
foreach ($f in $haVmx) {
  & $vmrun -T ws start $f
  Start-Sleep -Seconds 10
}
$running = & $vmrun -T ws list
$up = ($haVmx | Where-Object { $running -contains $_ }).Count
if ($up -eq 6) { "PASS: ca sau VM dang chay" } else { "FAIL: moi co $up/6 VM chay" }
```

**PASS:** dòng `PASS: ca sau VM dang chay` xuất hiện. `$haVmx` được liệt kê theo đúng thứ tự
LB → `lab-ha-1` → … → `lab-ha-5` ở B0.6, nên vòng lặp này bật đúng thứ tự của
[mục 2.5](#25-trình-tự-an-toàn-khi-tắt-và-bật-máy). Bật bằng giao diện VMware cũng được, miễn giữ
đúng thứ tự.

### B1.2. Biến của phiên làm việc

Từ đây tới hết B13, **giữ nguyên một phiên shell trên `lab-ha-1`**.

```bash
# Chay tren lab-ha-1
LB='lab-ha-lb';  LB_IP='192.168.100.230'
CP1='lab-ha-1';  CP1_IP='192.168.100.231'
CP2='lab-ha-2';  CP2_IP='192.168.100.232'
CP3='lab-ha-3';  CP3_IP='192.168.100.233'
WN1='lab-ha-4';  WN1_IP='192.168.100.234'
WN2='lab-ha-5';  WN2_IP='192.168.100.235'
CPS="$CP1 $CP2 $CP3"
NODES="$CP1 $CP2 $CP3 $WN1 $WN2"
ENDPOINT="$LB:6443"

EV="$HOME/lab-evidence/8b"; WK="$HOME/lab-work/8b"
mkdir -p "$EV" "$WK"
test -d "$EV" && test -d "$WK" && echo 'PASS: da co thu muc evidence va thu muc lam viec'
```

**PASS:** dòng `PASS: da co thu muc evidence va thu muc lam viec` xuất hiện.

Bốn giá trị dưới đây **lấy từ [bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa)**.
Lab 8b cố ý **không chép con số version nào**; bạn mở bảng đó ra và điền:

```bash
# Chay tren lab-ha-1 — mo bang A1.3 cua Lab 00 va dien dung bon gia tri
K8S_VER=''       # dong "Kubernetes control plane", dang vX.Y.Z
POD_CIDR=''      # dong "Pod CIDR"
FLANNEL_VER=''   # dong "CNI", phan release cua manifest Flannel, dang vX.Y.Z
TEST_IMAGE=''    # dong "Image dung cho gate A5.4"
```

Gate: bốn biến phải khác rỗng, đúng hình dạng, và `K8S_VER` phải **khớp với binary đã cài** ở B0.3.

```bash
# Chay tren lab-ha-1
KV="$(kubeadm version -o short)"
echo "kubeadm tren may: $KV | bang A1.3 ban dien: $K8S_VER"

test -n "$K8S_VER" && test "$KV" = "$K8S_VER" \
  && echo 'PASS: K8S_VER khop kubeadm da cai' \
  || echo 'FAIL: sai gia tri hoac sai package — quay lai A4.3 cua Lab 00'
echo "$POD_CIDR"    | grep -Eq '^[0-9]+(\.[0-9]+){3}/[0-9]+$' && echo 'PASS: POD_CIDR dung dang CIDR'
echo "$FLANNEL_VER" | grep -Eq '^v[0-9]+\.[0-9]+\.[0-9]+$'    && echo 'PASS: FLANNEL_VER dung dang'
test -n "$TEST_IMAGE" && echo 'PASS: TEST_IMAGE da dien'
```

**PASS:** bốn dòng `PASS:` xuất hiện, không dòng `FAIL:` nào. `K8S_VER` lệch nghĩa là hoặc bạn chép
nhầm, hoặc B0.3 cài sai package — cả hai đều phải sửa **trước** khi chạy `kubeadm init`, vì
`--kubernetes-version` sẽ dùng chính giá trị này.

### B1.3. Gate đầu vào của bộ VM HA

```bash
# Chay tren lab-ha-1
for n in $NODES; do
  L="$(ssh -o BatchMode=yes "$n" 'swapon --show | wc -l; \
       systemctl is-active containerd kubelet | tr "\n" " "; \
       systemctl is-enabled containerd kubelet | tr "\n" " "; \
       kubeadm version -o short' 2>/dev/null | tr '\n' ' ')"
  echo "$n : $L"
done | tee "$EV/b1-node-prereq.txt"

# Ba dieu kien tach roi, moi cai mot gate
SWAP_BAD="$(awk '{print $3}' "$EV/b1-node-prereq.txt" | grep -vc '^0$')"
ENA_OK="$(grep -c 'enabled enabled' "$EV/b1-node-prereq.txt")"
VER_OK="$(grep -c -- "$K8S_VER" "$EV/b1-node-prereq.txt")"
echo "node con bat swap=$SWAP_BAD | node co containerd+kubelet enabled=$ENA_OK | node dung version=$VER_OK"

test "$SWAP_BAD" -eq 0 && echo 'PASS: khong node nao con bat swap'
test "$ENA_OK"   -eq 5 && echo 'PASS: containerd va kubelet enabled tren ca nam node'
test "$VER_OK"   -eq 5 && echo 'PASS: ca nam node cung version kubeadm voi bang A1.3'
```

`kubelet` có thể đang crashloop trên cả năm node ở thời điểm này — đó là trạng thái **dự kiến**,
đúng ghi chú cuối [A4.3](LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl): chưa
có `/var/lib/kubelet/config.yaml` thì kubelet chưa có việc gì để làm. Gate ở đây kiểm `enabled`, tức
kubelet **sẽ** chạy sau reboot, chứ không kiểm `active`.

```bash
# Chay tren lab-ha-1 — chua node nao thuoc cluster nao
CONF=0
for n in $NODES; do
  ssh -o BatchMode=yes "$n" 'test -e /etc/kubernetes/admin.conf || test -d /var/lib/etcd' \
    && { echo "FAIL: $n da co dau vet cluster cu"; CONF=$(( CONF + 1 )); }
done
test "$CONF" -eq 0 && echo 'PASS: nam node deu sach, chua tung init hay join'
```

**PASS:** bốn dòng `PASS:` của B1.3 xuất hiện, không dòng `FAIL:` nào. Có dấu vết cluster cũ nghĩa
là bạn đang chạy lại lab trên VM đã dùng; restore cả sáu VM về `8x-vm-ready` rồi bắt đầu lại từ B1.

---

## B2. Hai topology: ranh giới và số host

**Mục đích:** trước khi gõ lệnh nào, chốt lại bằng số học của bài
[06](../06-ha-topology-vi.md) rằng bộ sáu VM này đủ cho stacked và **không** đủ cho external nếu giữ
nguyên vai từng máy. Đây là bước "quyết định trước khi dựng HA" mà lộ trình đánh dấu.

### B2.1. Bốn ràng buộc lấy thẳng từ bài 06

Ghi bảng đánh đổi vào evidence bằng chính chữ của bài, để lát nữa B7 và B11 đối chiếu:

```bash
# Chay tren lab-ha-1
cat > "$EV/b2-topology.txt" <<'EOF'
=== Bang danh doi hai topology (bai 06) ===
Diem chung:
  - Moi control plane node chay mot kube-apiserver, kube-scheduler, kube-controller-manager.
  - kube-apiserver duoc cung cap cho worker THONG QUA MOT LOAD BALANCER (ca hai topology).

STACKED (xep chong):
  - Moi control plane node tao MOT etcd member cuc bo.
  - Member do CHI giao tiep voi kube-apiserver CUA CHINH NODE DO.
  - Gia: hong theo cap — mot node sap la mat DONG THOI mot etcd member VA mot control plane instance.
  - Toi thieu: 3 control plane node xep chong.
  - Day la topology MAC DINH cua kubeadm (init va join --control-plane tu tao member).

EXTERNAL (ben ngoai):
  - etcd member chay tren host TACH BIET.
  - Moi etcd host giao tiep voi kube-apiserver cua TUNG control plane node (nhieu-nhieu).
  - Loi: mat mot control plane instance hoac mot etcd member it tac dong hon.
  - Gia: GAP DOI so host — toi thieu 3 host control plane VA 3 host etcd, chua tinh worker.
EOF
wc -l "$EV/b2-topology.txt"

grep -q 'CHI giao tiep voi kube-apiserver CUA CHINH NODE DO' "$EV/b2-topology.txt" \
  && grep -q 'GAP DOI so host' "$EV/b2-topology.txt" \
  && echo 'PASS: da ghi bang danh doi voi hai ranh gioi quan trong nhat'
```

**PASS:** dòng `PASS: da ghi bang danh doi voi hai ranh gioi quan trong nhat` xuất hiện.

### B2.2. Số học trên chính sáu VM của bạn

```bash
# Chay tren lab-ha-1
MACHINES=6            # tong so VM cua bo lab HA
LB_HOST=1             # lab-ha-lb, khong phai node Kubernetes
K8S_HOSTS=$(( MACHINES - LB_HOST ))
MIN_CP_STACKED=3      # bai 06: toi thieu ba control plane node xep chong
MIN_CP_EXTERNAL=3     # bai 06: toi thieu ba host control plane
MIN_ETCD_EXTERNAL=3   # bai 06: VA ba host etcd
NEED_EXTERNAL=$(( MIN_CP_EXTERNAL + MIN_ETCD_EXTERNAL ))

echo "may khong phai LB           : $K8S_HOSTS"
echo "stacked can it nhat         : $MIN_CP_STACKED control plane (+ worker)"
echo "external can it nhat        : $NEED_EXTERNAL host (3 control plane + 3 etcd), chua tinh worker"

test "$K8S_HOSTS" -ge $(( MIN_CP_STACKED + 2 )) \
  && echo 'PASS: du may cho stacked 3 control plane + 2 worker'
test "$K8S_HOSTS" -lt "$NEED_EXTERNAL" \
  && echo 'PASS: KHONG du may cho external o muc toi thieu cua bai 06'
```

**PASS:** cả hai dòng `PASS:` xuất hiện. Năm máy Kubernetes đủ cho `3 + 2` của stacked, và **thiếu
một máy** so với mức `3 + 3` mà external đòi — chưa kể worker.

**Ý nghĩa, và vì sao Lab 8c phải đổi vai máy.** Đây chính là câu "yêu cầu số lượng host gấp đôi so
với topology HA xếp chồng" của bài 06, hiện ra thành một phép trừ. Lab 8c dựng external etcd trên
đúng năm máy này: `lab-ha-1`, `lab-ha-2`, `lab-ha-3` thành **node etcd**, `lab-ha-4` và `lab-ha-5`
thành **control plane**. Kết quả là một cluster có ba etcd host **nhưng chỉ hai control plane
node** — dưới mức tối thiểu của bài 06, và không còn worker nào. Lab 8c phải xử lý ranh giới đó
một cách rõ ràng; Lab 8b chỉ có nhiệm vụ ghi lại con số để lab sau không đi nhầm.

Đây cũng là lý do tên máy trung tính ở [mục 2.1](#21-bộ-vm-riêng-của-nhóm-lab-ha): vai đổi, tên
không đổi.

### B2.3. Đối chiếu với cluster một control plane

Cluster của chuỗi chính có **một** control plane node. Mục *Khả năng chống chịu của cluster* của bài
[02](../02-create-cluster-kubeadm-vi.md) nói thẳng: control plane node đó gặp sự cố thì cluster **có
thể mất dữ liệu và phải tạo lại từ đầu**. Bài đưa hai cách khắc phục — sao lưu etcd thường xuyên, và
**dùng nhiều control plane node**. Lab 8b làm cách thứ hai. Cách thứ nhất là
[nợ #8](README.md#5-sổ-nợ-lab), chưa trả ở đây.

Còn một quyết định **một chiều** phải nhớ trước khi gõ `kubeadm init` ở B4: theo bài 02, việc chuyển
một cluster đã tạo **không có** `--control-plane-endpoint` thành cluster HA **không được kubeadm hỗ
trợ**. Cluster của Lab 00 tránh được cái bẫy đó vì
[A5.1](LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane) có truyền cờ này — nhưng nó trỏ vào chính
control plane node, không phải load balancer. B4 sẽ trỏ vào load balancer.

---

## B3. Load balancer đứng trước các API server

**Mục đích:** dựng bước **đầu tiên và chung cho cả hai phương pháp** của bài
[08](../08-high-availability-vi.md), rồi đọc đúng kết quả kiểm tra kết nối trước khi init.

Bài 08 đặt bốn yêu cầu cho load balancer, lab thực hiện đủ bốn:

1. Chuyển tiếp **TCP** tới các control plane node **khỏe mạnh** trong danh sách đích.
2. Health check là một **kiểm tra TCP** trên cổng mà `kube-apiserver` lắng nghe, mặc định `:6443`.
3. LB phải giao tiếp được với **tất cả** control plane node trên cổng apiserver, và cho phép lưu
   lượng đến trên cổng lắng nghe của nó.
4. Địa chỉ của LB **luôn khớp** với `ControlPlaneEndpoint` của kubeadm.

> **Một load balancer là SPOF — nói thẳng ngay từ đầu.** Lab dùng **đúng một** VM làm LB để giữ số
> máy ở mức host chịu được. Cấu hình này làm ba `kube-apiserver` dự phòng cho nhau, nhưng **chính LB
> thì không có ai dự phòng**: nó chết là mọi client và mọi node mất endpoint, dù cả ba API server
> vẫn sống nguyên. Production cần **cặp LB** cộng một **địa chỉ VIP** trôi giữa hai máy (keepalived
> hoặc cơ chế VRRP tương đương), đúng hướng mà bài 08 trỏ tới ở *Các lựa chọn cho cân bằng tải bằng
> phần mềm*. Lab không dựng vì cần thêm một VM thứ bảy và một IP thứ bảy dành riêng cho VIP. Đừng
> kết luận rằng cluster này đã HA hoàn chỉnh — **B12 sẽ chứng minh ngược lại bằng thực nghiệm**, và
> lý do đã được ghi trong bảng "không kiểm chứng được" ở [mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

### B3.1. Cài HAProxy

**HAProxy không nằm trong baseline** của lộ trình: nó là phần mềm hạ tầng ngoài Kubernetes, và bài
08 nói rõ ví dụ của nó "chỉ là một lựa chọn". Lab cài **từ gói của bản phân phối, không ghim
version**, rồi ghi version quan sát được vào evidence.

```bash
# Chay tren lab-ha-lb
sudo apt-get update
sudo apt-get install -y haproxy
haproxy -v
dpkg-query -W -f='${Package} ${Version} ${Architecture}\n' haproxy
```

**PASS:** `haproxy -v` in ra một dòng version, và `dpkg-query` in đúng một dòng cho gói `haproxy`.
Con số version của bạn có thể khác của người khác — đó là **chấp nhận được** ở đây và chỉ ở đây.

### B3.2. Cấu hình chuyển tiếp TCP và health check

```bash
# Chay tren lab-ha-lb
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig
sudo tee -a /etc/haproxy/haproxy.cfg >/dev/null <<'EOF'

#--- Lab 8b: chuyen tiep TCP toi ba kube-apiserver ---------------------------
frontend k8s-apiserver
    bind *:6443
    mode tcp
    option tcplog
    timeout client 1h
    default_backend k8s-controlplane

backend k8s-controlplane
    mode tcp
    option tcp-check
    option tcpka
    timeout server 1h
    balance roundrobin
    default-server inter 5s downinter 5s rise 2 fall 2
    server lab-ha-1 192.168.100.231:6443 check
    server lab-ha-2 192.168.100.232:6443 check
    server lab-ha-3 192.168.100.233:6443 check

listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s
EOF

sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl enable --now haproxy
sudo systemctl restart haproxy
systemctl is-active haproxy
systemctl is-enabled haproxy
ss -lntp | grep -E ':6443|:8404'
```

**PASS:** `haproxy -c` in `Configuration file is valid`; `is-active` trả `active`; `is-enabled` trả
`enabled`; `ss` cho thấy có tiến trình lắng nghe trên **cả** `6443` **và** `8404`. Chưa `enabled`
thì sau mỗi lần restore snapshot load balancer không lên và cả cluster mất endpoint.

Bốn chi tiết trong file cấu hình, ứng với bốn yêu cầu của bài 08:

- `mode tcp` ở cả frontend lẫn backend: **chuyển tiếp TCP**, không phải HTTP. TLS của apiserver đi
  xuyên qua LB nguyên vẹn; LB không giải mã và không cần certificate nào.
- `option tcp-check` cộng `check` trên từng `server`: health check là **kiểm tra TCP** trên
  `:6443`. Node nào không bắt tay được thì HAProxy loại khỏi danh sách đích — đúng câu "phân phối
  lưu lượng đến tất cả các node control plane **khỏe mạnh**".
- `rise 2 fall 2`: cần hai lần kiểm liên tiếp mới đổi trạng thái, tránh nhảy UP/DOWN khi mạng chớp.
  **Không ghi con số giây nào ở đây thành cam kết**: thời gian một backend chuyển trạng thái phụ
  thuộc `inter` và `fall`, tức phụ thuộc cấu hình.
- Backend khai bằng **IP**, không bằng tên: HAProxy phân giải tên khi khởi động, và một load
  balancer phụ thuộc DNS thì hỏng DNS là hỏng luôn endpoint của cluster.

Còn hai dòng nữa không nằm trong bốn yêu cầu của bài 08 nhưng bắt buộc phải có, vì file mặc định
của gói Ubuntu được viết cho HTTP: `timeout client 1h` và `timeout server 1h` cộng `option tcpka`.
Khối `defaults` mặc định đặt timeout ở mức vài chục giây — hợp lý cho request HTTP ngắn, nhưng
`kubectl` và mọi kubelet giữ kết nối **watch** mở rất lâu mà không truyền gì. Để nguyên mặc định thì
load balancer sẽ cắt các kết nối im lặng đó và bạn thấy watch đứt liên tục. Hai giá trị này là
**tham số cấu hình của load balancer**, không phải cam kết về thời gian hội tụ của Kubernetes.

Địa chỉ `192.168.100.230:6443` — tức tên `lab-ha-lb:6443` — chính là giá trị sẽ truyền cho
`--control-plane-endpoint` ở B4. Đó là yêu cầu số 4 của bài 08.

### B3.3. Ghi version quan sát được vào evidence

```bash
# Chay tren lab-ha-1
{
  echo "=== $(date -Is) — load balancer cua Lab 8b ==="
  ssh -o BatchMode=yes "$LB" 'haproxy -v'
  ssh -o BatchMode=yes "$LB" 'dpkg-query -W -f="${Package} ${Version} ${Architecture}\n" haproxy'
  ssh -o BatchMode=yes "$LB" 'systemctl is-active haproxy; systemctl is-enabled haproxy'
} | tee "$EV/b3-haproxy-version.txt"

grep -q 'HAProxy' "$EV/b3-haproxy-version.txt" && echo 'PASS: da ghi version HAProxy quan sat duoc'
test "$(grep -c '^active$\|^enabled$' "$EV/b3-haproxy-version.txt")" -eq 2 \
  && echo 'PASS: haproxy vua active vua enabled'
```

**PASS:** hai dòng `PASS:` xuất hiện. Version HAProxy **không** được đưa vào bảng A1.3 và **không**
ghim: nó thay đổi theo bản vá của bản phân phối, và cluster Kubernetes không có ràng buộc tương
thích nào với nó. Cái phải cố định là **hành vi** — chuyển tiếp TCP, health check TCP — chứ không
phải con số.

### B3.4. Hai phép thử kết nối, hai kết luận trái ngược

Đây là chỗ bài 08 dạy một thứ dễ hiểu sai: **báo lỗi không đồng nghĩa với hỏng**.

**Phép thử thứ nhất — từ load balancer tới control plane node**, đúng ý "load balancer phải có khả
năng giao tiếp với tất cả các node control plane trên cổng của apiserver". Chưa `kubeadm init` nên
chưa có gì lắng nghe ở `:6443` trên các node:

```bash
# Chay tren lab-ha-lb
for ip in 192.168.100.231 192.168.100.232 192.168.100.233; do
  S=$(date +%s)
  timeout 3 bash -c "echo > /dev/tcp/$ip/6443" 2>/dev/null; RC=$?
  E=$(( $(date +%s) - S ))
  if   test "$E" -ge 3; then echo "$ip -> TIMEOUT sau ${E}s — LB KHONG giao tiep duoc"
  elif test "$RC" -ne 0; then echo "$ip -> refused sau ${E}s — BINH THUONG, apiserver chua chay"
  else echo "$ip -> ket noi duoc sau ${E}s — da co gi do lang nghe o 6443"
  fi
done
```

**PASS:** cả ba dòng đều là `refused`, **không** dòng nào là `TIMEOUT`. Đây chính là câu của bài 08:
*"Lỗi connection refused là điều được mong đợi vì API server chưa chạy. Tuy nhiên, timeout có nghĩa
là load balancer không thể giao tiếp với node control plane."* Refused nghĩa là gói tin **đã tới
được** node, chỉ chưa có ai lắng nghe. Timeout nghĩa là gói tin không tới nơi — sửa mạng hoặc
firewall trước, **chưa chạy `kubeadm init`**.

Có `nc` thì `nc -zv -w 2 <ip> 6443` cho cùng kết luận; lab dùng `/dev/tcp` của bash để không phải
cài thêm gói, giống [A5.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a53-join-hai-worker).

**Phép thử thứ hai — từ các node tới load balancer.** Ở chiều này HAProxy **đã** lắng nghe `:6443`
nên nó **chấp nhận** kết nối rồi đóng ngay vì chưa backend nào UP. Kết quả vì thế **không** phải
`refused` như phía cloud LB của bài 08 — đó là khác biệt của phần mềm LB, không phải lỗi:

```bash
# Chay tren lab-ha-1
for n in $NODES; do
  R="$(ssh -o BatchMode=yes "$n" \
      "S=\$(date +%s); timeout 3 bash -c 'echo > /dev/tcp/${LB_IP}/6443' >/dev/null 2>&1; \
       echo elapsed=\$(( \$(date +%s) - S ))")"
  echo "$n -> $R"
done | tee "$EV/b3-node-to-lb.txt"

test "$(grep -c 'elapsed=[012]$' "$EV/b3-node-to-lb.txt")" -eq 5 \
  && echo 'PASS: ca nam node cham duoc :6443 cua LB ma khong timeout'
```

**PASS:** dòng `PASS: ca nam node cham duoc :6443 cua LB ma khong timeout` xuất hiện. Điều kiện
được gate ở đây là **không timeout**, tức đường từ node tới cổng lắng nghe của LB thông — yêu cầu
"nó cũng phải cho phép lưu lượng đến trên cổng lắng nghe của nó".

### B3.5. Ba backend đang DOWN — và đó là đúng

```bash
# Chay tren lab-ha-1
lbstat() {
  curl -s --max-time 10 "http://${LB_IP}:8404/stats;csv" \
    | awk -F, '$1=="k8s-controlplane" && $2!="BACKEND" {print $2"="$18}'
}
# HAProxy coi server la UP cho toi khi du so lan health check that bai,
# nen phai cho bang vong lap co dieu kien dung thay vi doc ngay mot lan.
DOWNN=0
for i in $(seq 1 30); do
  DOWNN="$(lbstat | grep -c '=DOWN$')"
  test "$DOWNN" -eq 3 && break
  sleep 5
done
lbstat | tee "$EV/b3-backend-truoc-init.txt"
echo "backend DOWN = $DOWNN"

test "$DOWNN" -eq 3 \
  && echo 'PASS: ca ba backend dang DOWN vi chua co apiserver nao chay'
```

**PASS:** dòng `PASS: ca ba backend dang DOWN vi chua co apiserver nao chay` xuất hiện.

Hàm `lbstat` đọc trang thống kê của HAProxy ở định dạng CSV: cột 1 là tên proxy, cột 2 là tên
server, cột 18 là trạng thái. Nó được dùng lại ở **B6, B9, B10, B11 và B12** như một cửa sổ nhìn vào
health check TCP mà bài 08 mô tả. Nếu output rỗng, xem [mục 4](#4-troubleshooting-của-lab-này).

Chú ý cách gate này chờ: HAProxy khai một server là `UP` ngay khi khởi động và chỉ hạ xuống `DOWN`
sau **đủ số lần** health check thất bại — `fall 2` với `inter 5s` ở B3.2. Vòng lặp có số vòng tối đa
là cách đúng để chờ; **đừng** viết một con số giây cố định, vì nó phụ thuộc chính hai tham số đó.

---
## B4. `kubeadm init` với `--control-plane-endpoint` và `--upload-certs`

**Mục đích:** khởi tạo control plane node đầu tiên đúng cách của bài
[08](../08-high-availability-vi.md), rồi đọc kỹ những gì lệnh đó để lại: hai lệnh join khác nhau, một
certificate key, và một Secret có hạn.

### B4.1. Kéo image và init

```bash
# Chay tren lab-ha-1
sudo kubeadm config images list --kubernetes-version "$K8S_VER"
sudo kubeadm config images pull --kubernetes-version "$K8S_VER"
echo "rc=$?"
```

**PASS:** `images pull` kết thúc với `rc=0`. Fail ở đây gần như luôn là egress hoặc DNS, không phải
kubeadm — quay lại gate cuối [A3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a3-đặt-ip-tĩnh-và-phân-giải-tên).

```bash
# Chay tren lab-ha-1 — KHONG ngat lenh nay giua chung
sudo kubeadm init \
  --kubernetes-version "$K8S_VER" \
  --control-plane-endpoint "$ENDPOINT" \
  --apiserver-advertise-address "$CP1_IP" \
  --pod-network-cidr "$POD_CIDR" \
  --upload-certs 2>&1 | tee "$EV/b4-init.log"

grep -q 'Your Kubernetes control-plane has initialized successfully' "$EV/b4-init.log" \
  && echo 'PASS: kubeadm init thanh cong'
```

**PASS:** dòng `PASS: kubeadm init thanh cong` xuất hiện.

Năm cờ, năm lý do — và **chỉ có một cờ khác** so với
[A5.1 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane) về mặt hình dạng, nhưng giá trị
của nó mới là chỗ đổi bản chất:

| Cờ | Vì sao có mặt |
| --- | --- |
| `--kubernetes-version "$K8S_VER"` | không để kubeadm dò remote rồi chọn patch khác với package đã cài; giống A5.1 |
| `--control-plane-endpoint "$ENDPOINT"` | **đây là cờ quyết định**: endpoint **dùng chung cho mọi control plane node**, trỏ vào **load balancer**. A5.1 cũng có cờ này nhưng trỏ vào chính control plane node vì cluster đó chỉ có một |
| `--apiserver-advertise-address "$CP1_IP"` | địa chỉ quảng bá **của riêng node này**. Bài [02](../02-create-cluster-kubeadm-vi.md) phân biệt rõ hai cờ: một cái cho riêng node, một cái dùng chung |
| `--pod-network-cidr "$POD_CIDR"` | CNI của baseline cần Pod CIDR khớp với manifest của nó; giống A5.1 |
| `--upload-certs` | tải certificate control plane lên cluster để node control plane khác lấy về khi join. **Cờ này không tồn tại trong A5.1** vì cluster đó không bao giờ có node control plane thứ hai |

Cấu hình kubeconfig cho user thường **đúng như A5.1 đã giải thích từng lệnh**; không chép lại phần
giải thích đó vào đây:

```bash
# Chay tren lab-ha-1
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
chmod 600 "$HOME/.kube/config"

for i in $(seq 1 60); do
  kubectl get --raw='/healthz' >/dev/null 2>&1 && break
  sleep 5
done
kubectl get --raw='/healthz'; echo
kubectl get nodes -o wide | tee "$EV/b4-nodes.txt"
```

**PASS:** `/healthz` trả `ok`; `kubectl get nodes` liệt kê đúng một Node tên `lab-ha-1`. Node còn
`NotReady` là **bình thường** — chưa cài CNI. Chú ý là request này đã đi **qua load balancer**: HAProxy
đã đổi backend `lab-ha-1` từ `DOWN` sang `UP` sau khi apiserver bắt đầu lắng nghe.

### B4.2. `controlPlaneEndpoint` khớp load balancer — ba nguồn độc lập

```bash
# Chay tren lab-ha-1
SRV="$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
CI="$(kubectl -n kube-public get configmap cluster-info \
      -o jsonpath='{.data.kubeconfig}' | grep -m1 'server:' | awk '{print $2}')"
KC="$(kubectl -n kube-system get configmap kubeadm-config \
      -o jsonpath='{.data.ClusterConfiguration}' | grep -m1 'controlPlaneEndpoint:' | awk '{print $2}')"
echo "kubeconfig      : $SRV"
echo "cluster-info    : $CI"
echo "kubeadm-config  : $KC"
echo "load balancer   : https://$ENDPOINT"

test "$SRV" = "https://$ENDPOINT" && echo 'PASS: kubeconfig tro vao load balancer'
test "$CI"  = "https://$ENDPOINT" && echo 'PASS: cluster-info tro vao load balancer'
test "$KC"  = "$ENDPOINT"         && echo 'PASS: kubeadm-config ghi dung controlPlaneEndpoint'
```

**PASS:** ba dòng `PASS:` xuất hiện.

**Ý nghĩa:** bài 08 đặt một yêu cầu ngắn — "đảm bảo địa chỉ của load balancer luôn khớp với địa chỉ
`ControlPlaneEndpoint` của kubeadm" — và ba giá trị trên là ba nơi giá trị đó được ghi lại. Chúng
phục vụ ba nhóm khách khác nhau: `kubeconfig` cho quản trị viên, ConfigMap `cluster-info` trong
`kube-public` cho **node đang join** đọc lúc discovery, `kubeadm-config` cho chính kubeadm ở những
lần chạy sau. Lệch một trong ba là có một nhóm bị chỉ tới endpoint sai, và cluster mất đúng tính
chất HA vừa dựng.

### B4.3. Hai lệnh join, khác nhau đúng hai cờ

`kubeadm init` in **hai** lệnh join khác nhau. Tách chúng ra khỏi log — output thật xuống dòng bằng
dấu `\`, nên phải nối lại trước khi so:

```bash
# Chay tren lab-ha-1
sed -e ':a' -e 'N' -e '$!ba' -e 's/\\\n[[:space:]]*/ /g' "$EV/b4-init.log" \
  | grep -E '^[[:space:]]*kubeadm join' \
  | sed 's/^[[:space:]]*//' | tr -s ' ' > "$WK/join-commands.txt"

cat "$WK/join-commands.txt"
test "$(wc -l < "$WK/join-commands.txt")" -eq 2 && echo 'PASS: init in ra dung hai lenh join'
```

```bash
# Chay tren lab-ha-1
CP_JOIN="$(grep -- '--control-plane' "$WK/join-commands.txt")"
WORKER_JOIN="$(grep -v -- '--control-plane' "$WK/join-commands.txt")"
CERT_KEY="$(sed -nE 's/.*--certificate-key ([0-9a-f]+).*/\1/p' <<< "$CP_JOIN")"
STRIPPED="$(sed -E 's/ --control-plane//; s/ --certificate-key [0-9a-f]+//' <<< "$CP_JOIN")"

echo "control plane : $CP_JOIN"
echo "worker        : $WORKER_JOIN"
echo "bo hai co di  : $STRIPPED"

test "$STRIPPED" = "$WORKER_JOIN" \
  && echo 'PASS: hai lenh join chi khac dung hai co --control-plane va --certificate-key'
echo "$CERT_KEY" | grep -Eq '^[0-9a-f]{64}$' \
  && echo 'PASS: certificate key la chuoi hex 64 ky tu — khoa AES 32 byte'
grep -q "$ENDPOINT" "$WK/join-commands.txt" \
  && echo 'PASS: ca hai lenh join deu tro vao load balancer, khong vao IP mot node'
```

**PASS:** ba dòng `PASS:` xuất hiện.

**Ý nghĩa:** phép trừ vừa rồi là cách gọn nhất để nhớ khác biệt giữa hai lệnh. Token và
`--discovery-token-ca-cert-hash` **giống hệt nhau** ở cả hai — chúng chỉ trả lời câu hỏi "node này có
quyền join không" và "cluster ở đầu kia có đúng là cluster tôi nghĩ không". Hai cờ thừa mới là phần
biến một lệnh join thành join control plane:

- `--control-plane` yêu cầu `kubeadm join` **tạo một control plane mới** trên node đó, chứ không chỉ
  đăng ký một kubelet.
- `--certificate-key` khiến certificate control plane được **tải xuống từ Secret `kubeadm-certs`** và
  **giải mã** bằng khóa đã cho.

Certificate key là chuỗi mã hóa hex của một khóa AES 32 byte, đúng như bài 08 mô tả — 32 byte thành
64 ký tự hex. Lệnh sinh một khóa như vậy có sẵn, và chạy nó **không** đụng vào cluster:

```bash
# Chay tren lab-ha-1
kubeadm certs certificate-key | tee "$WK/mau-certificate-key.txt"
grep -Eq '^[0-9a-f]{64}$' "$WK/mau-certificate-key.txt" \
  && echo 'PASS: kubeadm certs certificate-key sinh dung mot khoa cung hinh dang'
```

**PASS:** dòng `PASS: kubeadm certs certificate-key sinh dung mot khoa cung hinh dang` xuất hiện.
Khóa vừa sinh **không** được dùng ở đâu trong lab — nó chỉ chứng minh hình dạng. Muốn dùng khóa cố
định do mình chọn thì phải truyền `--certificate-key` **ngay lúc `init`**; ở đây khóa do kubeadm
sinh.

> **Certificate key cho phép truy cập dữ liệu nhạy cảm của cluster.** Bài 08 in đúng cảnh báo này
> trong output của lệnh. Đừng dán nó vào chat, ticket hay ảnh chụp màn hình. Biến `CERT_KEY` chỉ
> sống trong phiên shell của bạn, và `$WK` bị xoá ở B13.

### B4.4. Hiện vật của `--upload-certs` và hạn hai giờ

```bash
# Chay tren lab-ha-1
kubectl -n kube-system get secret kubeadm-certs | tee "$EV/b4-kubeadm-certs.txt"
echo '--- ten cac certificate duoc ma hoa ben trong ---' | tee -a "$EV/b4-kubeadm-certs.txt"
kubectl -n kube-system get secret kubeadm-certs \
  -o go-template='{{range $k, $v := .data}}{{$k}}{{"\n"}}{{end}}' \
  | tee -a "$EV/b4-kubeadm-certs.txt"

CERT_ITEMS="$(kubectl -n kube-system get secret kubeadm-certs \
  -o go-template='{{range $k, $v := .data}}{{$k}}{{"\n"}}{{end}}' | grep -c .)"
echo "so muc trong Secret: $CERT_ITEMS"
test "$CERT_ITEMS" -ge 1 \
  && echo 'PASS: Secret kubeadm-certs ton tai va co certificate ben trong'
```

**PASS:** dòng `PASS: Secret kubeadm-certs ton tai trong namespace kube-system` xuất hiện, và file
evidence liệt kê tên các certificate được mã hóa bên trong.

Hạn của Secret đọc được từ bootstrap token đang **sở hữu** nó:

```bash
# Chay tren lab-ha-1
OWNER="$(kubectl -n kube-system get secret kubeadm-certs \
         -o jsonpath='{.metadata.ownerReferences[0].name}')"
echo "Secret kubeadm-certs thuoc so huu cua: [$OWNER]"
test -n "$OWNER" && echo 'PASS: kubeadm-certs co ownerReference toi mot bootstrap token'

CRT="$(kubectl -n kube-system get secret kubeadm-certs -o jsonpath='{.metadata.creationTimestamp}')"
EXP="$(kubectl -n kube-system get secret "$OWNER" -o jsonpath='{.data.expiration}' | base64 -d)"
DIFF=$(( $(date -d "$EXP" +%s) - $(date -d "$CRT" +%s) ))
echo "tao luc $CRT | het han luc $EXP | chenh $DIFF giay"

test "$DIFF" -ge 7000 && test "$DIFF" -le 7400 \
  && echo 'PASS: vong doi cua kubeadm-certs dung hai gio nhu bai 08 mo ta'
```

**PASS:** hai dòng `PASS:` xuất hiện và `$DIFF` nằm quanh 7200 giây. `ownerReference` rỗng thì xem
[mục 4](#4-troubleshooting-của-lab-này); đừng bỏ qua bước này, vì hạn hai giờ là thứ hay cắn người
làm HA lần đầu.

**Ý nghĩa:** `--upload-certs` **mã hóa và tải certificate của control plane chính lên Secret
`kubeadm-certs`** trong `kube-system`. Vòng đời của Secret không do bạn quản lý bằng tay: kubeadm
gắn nó vào một bootstrap token sống **hai giờ**, và khi token hết hạn thì Secret bị dọn theo. Khóa
giải mã cũng hết hạn cùng lúc. Hệ quả rất cụ thể: **init buổi sáng rồi chiều mới join control plane
node thứ hai thì lệnh join sẽ hỏng**, và cách sửa không phải là `init` lại mà là tải lên lại —
chính là lệnh B6.1 sắp dùng.

Ghi lại toàn bộ ranh giới này vào evidence:

```bash
# Chay tren lab-ha-1
{
  echo "=== $(date -Is) — hien vat cua --upload-certs ==="
  echo "Secret            : kube-system/kubeadm-certs"
  echo "owner (token)     : $OWNER"
  echo "tao luc           : $CRT"
  echo "het han luc       : $EXP  (chenh $DIFF giay)"
  echo "certificate key   : do kubeadm sinh, hex 64 ky tu, KHONG ghi vao day"
  echo "tao lai bang      : sudo kubeadm init phase upload-certs --upload-certs"
} | tee "$EV/b4-upload-certs.txt"
test "$(wc -l < "$EV/b4-upload-certs.txt")" -ge 7 && echo 'PASS: da ghi ho so upload-certs'
```

**PASS:** dòng `PASS: da ghi ho so upload-certs` xuất hiện.

---

## B5. CNI và ảnh chụp CoreDNS trước khi có node thứ hai

**Mục đích:** cài Pod network **ngay sau init và trước khi join**, đúng thứ tự bài 08 quy định, rồi
chụp lại vị trí CoreDNS lúc cluster mới có một node — số liệu này là vế "trước" của phép so ở B6.5.

### B5.1. Cài CNI của baseline

Dùng **đúng CNI mà [A5.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a52-cài-flannel-v0289) dùng** và
**đúng Pod CIDR** đã khai ở `kubeadm init`. Không cài CNI khác ở đây: việc đổi sang CNI có thực thi
NetworkPolicy là bài học của Lab 5b, chạy trên bộ VM chính.

```bash
# Chay tren lab-ha-1
kubectl apply -f \
  "https://github.com/flannel-io/flannel/releases/download/${FLANNEL_VER}/kube-flannel.yml"

kubectl -n kube-flannel rollout status daemonset/kube-flannel-ds --timeout=300s
kubectl -n kube-flannel get pods -o wide
```

```bash
# Chay tren lab-ha-1
NET="$(kubectl -n kube-flannel get configmap kube-flannel-cfg \
       -o jsonpath='{.data.net-conf\.json}' | tr -d ' \n' | sed -nE 's/.*"Network":"([^"]+)".*/\1/p')"
IMGS="$(kubectl -n kube-flannel get daemonset kube-flannel-ds \
        -o jsonpath='{range .spec.template.spec.initContainers[*]}{.image}{"\n"}{end}{range .spec.template.spec.containers[*]}{.image}{"\n"}{end}')"
echo "$IMGS" | tee "$EV/b5-cni-images.txt"
echo "Network trong net-conf.json : $NET"
echo "POD_CIDR ban khai o init    : $POD_CIDR"

test "$NET" = "$POD_CIDR" \
  && echo 'PASS: Pod CIDR cua CNI khop --pod-network-cidr da dung o kubeadm init'
grep -q "$FLANNEL_VER" "$EV/b5-cni-images.txt" \
  && echo 'PASS: image CNI dung release khai o bang A1.3'

kubectl wait --for=condition=Ready node/"$CP1" --timeout=300s
kubectl get nodes -o wide | tee "$EV/b5-nodes.txt"
```

**PASS:** hai dòng `PASS:` xuất hiện và `lab-ha-1` chuyển sang `Ready`. `NET` lệch `POD_CIDR` nghĩa
là bạn điền sai biến ở B1.2 hoặc `kubeadm init` chạy với giá trị khác; đó là lỗi **phải sửa bằng
`kubeadm reset` và init lại**, không vá bằng cách sửa ConfigMap của CNI.

### B5.2. CoreDNS lúc cluster mới có một node

```bash
# Chay tren lab-ha-1
kubectl -n kube-system rollout status deployment/coredns --timeout=300s
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide | tee "$EV/b5-coredns-truoc.txt"

DNS_NODES_1="$(kubectl -n kube-system get pods -l k8s-app=kube-dns \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | wc -l)"
DNS_ON_CP1="$(kubectl -n kube-system get pods -l k8s-app=kube-dns \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | grep -c "^${CP1}$")"
DNS_TOTAL="$(kubectl -n kube-system get pods -l k8s-app=kube-dns --no-headers | wc -l)"
echo "Pod CoreDNS: $DNS_TOTAL | so node khac nhau dang chua chung: $DNS_NODES_1 | tren $CP1: $DNS_ON_CP1"

test "$DNS_NODES_1" -eq 1 && test "$DNS_ON_CP1" -eq "$DNS_TOTAL" \
  && echo 'PASS: toan bo Pod CoreDNS dang dong tren control plane node dau tien'
```

**PASS:** dòng `PASS: toan bo Pod CoreDNS dang dong tren control plane node dau tien` xuất hiện.

**Ý nghĩa:** đây **không phải lỗi**. Bài 08 giải thích bằng thứ tự thao tác: các node của cluster
được khởi tạo **tuần tự**, nên lúc scheduler xếp chỗ cho CoreDNS thì trong cluster mới chỉ có
`lab-ha-1`. Pod đã lập lịch thì ở yên chỗ cũ; Kubernetes không tự cân bằng lại Pod đang chạy. Giữ
con số này lại — B6.5 sẽ so với nó sau khi cluster có đủ năm node.

---

## B6. Join hai control plane và hai worker

**Mục đích:** biến cluster một node thành cluster HA ba control plane, rồi chứng minh load balancer
thật sự phân phối tới cả ba `kube-apiserver`.

### B6.1. Sinh token và certificate key mới

Bài 08 nói **join các node control plane từng node một**, và nhắc rằng khóa giải mã mặc định hết hạn
sau hai giờ. Thay vì hy vọng khóa cũ còn sống, lab dùng thẳng lệnh tải lên lại — đó cũng chính là
cách sửa khi gặp lỗi hết hạn:

```bash
# Chay tren lab-ha-1
NEW_KEY="$(sudo kubeadm init phase upload-certs --upload-certs | tail -n 1)"
JOIN_BASE="$(sudo kubeadm token create --print-join-command)"

echo "$NEW_KEY" | grep -Eq '^[0-9a-f]{64}$' && echo 'PASS: da co certificate key moi'
grep -q "^kubeadm join $ENDPOINT " <<< "$JOIN_BASE" \
  && echo 'PASS: lenh join moi van tro vao load balancer'

printf 'sudo %s --control-plane --certificate-key %s\n' "$JOIN_BASE" "$NEW_KEY" \
  | tee "$WK/join-control-plane.txt"
printf 'sudo %s\n' "$JOIN_BASE" | tee "$WK/join-worker.txt"
```

**PASS:** hai dòng `PASS:` xuất hiện, và hai file trong `$WK` chứa hai lệnh khác nhau đúng hai cờ —
đúng cấu trúc bạn vừa chứng minh ở B4.3.

### B6.2. Join `lab-ha-2` rồi `lab-ha-3` làm control plane

Chép nội dung `$WK/join-control-plane.txt` và chạy **trên `lab-ha-2`**. Chạy trực tiếp trên máy đó,
không bọc qua `ssh` — lệnh cần `sudo` có tty.

```bash
# Chay tren lab-ha-2 — dan dung lenh trong $WK/join-control-plane.txt
sudo kubeadm join lab-ha-lb:6443 --token <token-that> \
  --discovery-token-ca-cert-hash sha256:<hash-that> \
  --control-plane --certificate-key <khoa-that>
```

**PASS:** output có `This node has joined the cluster` **và** dòng nói node đã trở thành control
plane. Quay lại `lab-ha-1` và gate:

```bash
# Chay tren lab-ha-1
kubectl wait --for=condition=Ready node/"$CP2" --timeout=600s
kubectl get nodes -o wide

ETCD_EPS="https://$CP1_IP:2379,https://$CP2_IP:2379,https://$CP3_IP:2379"
etcd1() {
  kubectl -n kube-system exec "etcd-$CP1" -- etcdctl \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key "$@"
}

MEM_N="$(etcd1 --endpoints=https://127.0.0.1:2379 member list | wc -l)"
CP_N="$(kubectl get nodes -l node-role.kubernetes.io/control-plane -o name | wc -l)"
echo "etcd member=$MEM_N | control plane node=$CP_N"
test "$MEM_N" -eq 2 && test "$CP_N" -eq 2 \
  && echo 'PASS: join --control-plane vua them mot control plane VA mot etcd member'
```

**PASS:** dòng `PASS: join --control-plane vua them mot control plane VA mot etcd member` xuất hiện.
Một lệnh join, hai thứ được thêm cùng lúc — đó chính là mặt tích cực của cái mà bài 06 gọi là **gắn
kết**; mặt tiêu cực của nó là B9 và B11.

Lặp lại **y hệt trên `lab-ha-3`**, rồi gate:

```bash
# Chay tren lab-ha-1
kubectl wait --for=condition=Ready node/"$CP3" --timeout=600s
MEM_N="$(etcd1 --endpoints=https://127.0.0.1:2379 member list | wc -l)"
CP_N="$(kubectl get nodes -l node-role.kubernetes.io/control-plane -o name | wc -l)"
echo "etcd member=$MEM_N | control plane node=$CP_N"
test "$MEM_N" -eq 3 && test "$CP_N" -eq 3 \
  && echo 'PASS: du ba control plane node va ba etcd member — muc toi thieu cua bai 06'
```

**PASS:** dòng `PASS: du ba control plane node va ba etcd member — muc toi thieu cua bai 06` xuất
hiện.

### B6.3. Join `lab-ha-4` và `lab-ha-5` làm worker

Đây là thao tác *Thêm worker node Linux* mà trang [214](../214-kubeadm-tasks-vi.md) liệt kê. Chép nội
dung `$WK/join-worker.txt` và chạy **trên `lab-ha-4`**, rồi **trên `lab-ha-5`**:

```bash
# Chay tren lab-ha-4, sau do lap lai tren lab-ha-5
sudo kubeadm join lab-ha-lb:6443 --token <token-that> \
  --discovery-token-ca-cert-hash sha256:<hash-that>
```

**PASS:** mỗi worker in `This node has joined the cluster`.

```bash
# Chay tren lab-ha-1
kubectl wait --for=condition=Ready node --all --timeout=600s
kubectl get nodes -o wide | tee "$EV/b6-nodes.txt"

ALL_N="$(kubectl get nodes -o name | wc -l)"
CP_N="$(kubectl get nodes -l node-role.kubernetes.io/control-plane -o name | wc -l)"
RDY_N="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}' \
  | grep -c '^True$')"
echo "node=$ALL_N | control plane=$CP_N | Ready=$RDY_N"

test "$ALL_N" -eq 5 && test "$CP_N" -eq 3 && test "$RDY_N" -eq 5 \
  && echo 'PASS: cluster co 5 node, 3 control plane, tat ca Ready'
```

```bash
# Chay tren lab-ha-1 — taint cach ly control plane phai con nguyen tren ca ba
TAINT_N="$(kubectl get nodes -l node-role.kubernetes.io/control-plane \
  -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"="}{.effect}{"\n"}{end}{end}' \
  | grep -c 'node-role.kubernetes.io/control-plane=NoSchedule')"
WK_TAINT="$(kubectl get nodes "$WN1" "$WN2" -o jsonpath='{range .items[*]}{.spec.taints}{"\n"}{end}' \
  | grep -c .)"
echo "control plane mang taint NoSchedule: $TAINT_N | worker mang taint: $WK_TAINT"

test "$TAINT_N" -eq 3 && echo 'PASS: ca ba control plane node van giu taint cach ly'
test "$WK_TAINT" -eq 0 && echo 'PASS: hai worker khong mang taint nao'
```

**PASS:** ba dòng `PASS:` của B6.3 xuất hiện. Taint `node-role.kubernetes.io/control-plane:NoSchedule`
là giá trị **đúng**, không gỡ — bài [02](../02-create-cluster-kubeadm-vi.md) chỉ gỡ nó cho cluster một
máy duy nhất. Ở đây có hai worker để chạy workload, và ba control plane node đang gánh cả etcd nên
càng không nên nhận Pod thường.

### B6.4. Load balancer thật sự phân phối tới cả ba API server

Trước hết, ba backend phải UP:

```bash
# Chay tren lab-ha-1
lbstat | tee "$EV/b6-lb-status.txt"
test "$(grep -c '=UP$' "$EV/b6-lb-status.txt")" -eq 3 \
  && echo 'PASS: health check TCP thay ca ba apiserver khoe manh'
```

Rồi mở nhiều kết nối mới và đọc số phiên mà HAProxy đã chuyển cho **từng** server. Cột 8 của CSV là
`stot`, tổng số phiên đã chuyển:

```bash
# Chay tren lab-ha-1
lbstot() {
  curl -s --max-time 10 "http://${LB_IP}:8404/stats;csv" \
    | awk -F, '$1=="k8s-controlplane" && $2!="BACKEND" {print $2"="$8}'
}
for i in $(seq 1 15); do
  curl -sk -o /dev/null --max-time 5 "https://${LB_IP}:6443/healthz"
done
lbstot | tee "$EV/b6-lb-stot.txt"

NZ="$(awk -F= '$2 + 0 > 0' "$EV/b6-lb-stot.txt" | wc -l)"
echo "so backend da nhan it nhat mot phien: $NZ"
test "$NZ" -eq 3 \
  && echo 'PASS: load balancer da chuyen phien toi ca ba kube-apiserver'
```

**PASS:** hai dòng `PASS:` xuất hiện.

**Ý nghĩa:** đây là bằng chứng cho câu mà cả bài 06 lẫn bài 08 lặp lại — `kube-apiserver` **được cung
cấp cho các worker node thông qua một load balancer**. Mười lăm kết nối mới được `balance roundrobin`
rải sang ba đích khác nhau; nếu chỉ một backend có `stot` khác 0 thì hai apiserver còn lại đang không
nhận việc, và cluster của bạn chỉ *trông giống* HA. Nhiều bản `kube-apiserver` cùng phục vụ được vì
trạng thái nằm ở etcd, không nằm trong bộ nhớ apiserver.

### B6.5. Cân bằng lại CoreDNS

```bash
# Chay tren lab-ha-1
kubectl -n kube-system rollout restart deployment coredns
kubectl -n kube-system rollout status deployment coredns --timeout=300s
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide | tee "$EV/b6-coredns-sau.txt"

DNS_NODES_2="$(kubectl -n kube-system get pods -l k8s-app=kube-dns \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | wc -l)"
echo "so node khac nhau dang chua Pod CoreDNS: truoc=$DNS_NODES_1 -> sau=$DNS_NODES_2"

test "$DNS_NODES_2" -gt "$DNS_NODES_1" \
  && echo 'PASS: CoreDNS da trai ra nhieu hon mot node sau rollout restart'
```

**PASS:** dòng `PASS: CoreDNS da trai ra nhieu hon mot node sau rollout restart` xuất hiện.

**Ý nghĩa:** `rollout restart` không "sửa" gì cả — nó tạo Pod **mới**, và scheduler khi đó có năm
node để chấm điểm thay vì một. Đây đúng là cách bài 08 chỉ ở ghi chú cuối mục stacked, và là ví dụ
rõ nhất cho nguyên tắc: Kubernetes chỉ phân bố lại workload khi có **sự kiện tạo Pod mới**, không
tự dời Pod đang chạy.

---

## B7. Bằng chứng stacked: etcd nằm trên chính control plane node

**Mục đích:** chứng minh bằng cluster thật rằng topology này là **stacked**, chứ không phải tin vào
lời hứa của tài liệu.

### B7.1. Ba Pod etcd là static Pod, mỗi Pod trên đúng node của nó

```bash
# Chay tren lab-ha-1
kubectl -n kube-system get pod -l component=etcd -o wide | tee "$EV/b7-etcd-pods.txt"

MATCH="$(kubectl -n kube-system get pod -l component=etcd \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.nodeName}{"\n"}{end}' \
  | awk '{ if ("etcd-"$2 == $1) n++ } END { print n+0 }')"
STATIC="$(kubectl -n kube-system get pod -l component=etcd \
  -o jsonpath='{range .items[*]}{.metadata.annotations.kubernetes\.io/config\.source}{"\n"}{end}' \
  | grep -c '^file$')"
ETCD_NODES="$(kubectl -n kube-system get pod -l component=etcd \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | tr '\n' ' ')"
ETCD_NODE_N="$(echo "$ETCD_NODES" | tr ' ' '\n' | grep -c .)"
ETCD_ON_CP="$(echo "$ETCD_NODES" | tr ' ' '\n' | grep -cE "^($CP1|$CP2|$CP3)$")"
echo "Pod etcd nam dung tren node cua no: $MATCH | Pod co config.source=file: $STATIC"
echo "etcd dang chay tren: $ETCD_NODES (tong $ETCD_NODE_N node, trong do $ETCD_ON_CP la control plane)"

test "$MATCH" -eq 3 \
  && echo 'PASS: moi Pod etcd nam tren dung control plane node mang ten no'
test "$STATIC" -eq 3 \
  && echo 'PASS: ca ba deu la static Pod — kubelet doc tu file, khong qua scheduler'
test "$ETCD_NODE_N" -eq 3 && test "$ETCD_ON_CP" -eq 3 \
  && echo 'PASS: etcd chi chay tren control plane node, khong node nao khac'
```

```bash
# Chay tren lab-ha-1 — manifest that nam tren dia cua tung node
for n in $CPS; do
  printf '%s ' "$n"
  ssh -o BatchMode=yes "$n" 'ls /etc/kubernetes/manifests/'
done | tee "$EV/b7-manifests.txt"

test "$(grep -c 'etcd.yaml' "$EV/b7-manifests.txt")" -eq 3 \
  && echo 'PASS: ca ba node deu co etcd.yaml trong thu muc static Pod'
```

**PASS:** bốn dòng `PASS:` của B7.1 xuất hiện.

**Ý nghĩa:** ba dữ kiện độc lập cùng chỉ về một kết luận. Annotation `kubernetes.io/config.source`
bằng `file` nghĩa là Pod này **không** do scheduler đặt lên node — kubelet đọc thẳng manifest trong
`/etc/kubernetes/manifests` của chính máy nó và tạo container, rồi báo cáo ngược lên API server một
mirror Pod. Đó là lý do tên Pod luôn là `etcd-<tên node>`, và cũng là lý do tắt máy nào thì mất
đúng etcd member của máy đó. Bài 06 gọi hiện tượng này là **etcd member cục bộ được tạo tự động trên
control plane node khi dùng `kubeadm init` và `kubeadm join --control-plane`** — bạn vừa nhìn thấy
cả hai vế.

### B7.2. Số member etcd bằng số control plane node

```bash
# Chay tren lab-ha-1
etcd1 --endpoints=https://127.0.0.1:2379 member list -w table | tee "$EV/b7-members.txt"

MEM_N="$(etcd1 --endpoints=https://127.0.0.1:2379 member list | wc -l)"
CP_N="$(kubectl get nodes -l node-role.kubernetes.io/control-plane -o name | wc -l)"
NAME_OK=0
for n in $CPS; do grep -q "$n" "$EV/b7-members.txt" && NAME_OK=$(( NAME_OK + 1 )); done
echo "member=$MEM_N | control plane node=$CP_N | ten node xuat hien trong member list=$NAME_OK"

test "$MEM_N" -eq "$CP_N" \
  && echo 'PASS: so member etcd BANG so control plane node — dinh nghia cua stacked'
test "$NAME_OK" -eq 3 \
  && echo 'PASS: ba member mang dung ten ba control plane node'
```

```bash
# Chay tren lab-ha-1
etcd1 --endpoints="$ETCD_EPS" endpoint health 2>&1 | tee "$EV/b7-etcd-health.txt" || true
etcd1 --endpoints="$ETCD_EPS" endpoint status -w table 2>&1 | tee "$EV/b7-etcd-status.txt" || true

HEALTHY="$(grep -c 'is healthy' "$EV/b7-etcd-health.txt")"
ENDP="$(grep -c 'https://' "$EV/b7-etcd-status.txt")"
echo "endpoint khoe manh=$HEALTHY | dong endpoint trong bang status=$ENDP"

test "$HEALTHY" -eq 3 && echo 'PASS: ca ba etcd member deu khoe manh'
test "$ENDP" -eq 3 && echo 'PASS: bang status liet ke du ba endpoint'
```

**PASS:** bốn dòng `PASS:` của B7.2 xuất hiện. Trong bảng `endpoint status`, đúng **một** dòng có
`true` ở cột `IS LEADER` — ba member hợp thành **một** etcd cluster có một leader, chứ không phải ba
cơ sở dữ liệu rời rạc.

### B7.3. Ranh giới dễ nhầm nhất của bài 06

Ghi lại kết luận vào evidence, vì đây là câu hỏi checkpoint:

```bash
# Chay tren lab-ha-1
{
  echo "=== $(date -Is) — bang chung stacked ==="
  echo "control plane node : $CP_N"
  echo "etcd member        : $MEM_N  (bang nhau)"
  echo "etcd chay tren     : $ETCD_NODES"
  echo "kieu Pod           : static Pod, config.source=file, ten dang etcd-<node>"
  echo "quan he            : etcd member cuc bo CHI phuc vu kube-apiserver CUA CHINH NODE DO"
  echo "he qua             : mat 1 node = mat 1 etcd member VA 1 control plane instance"
} | tee "$EV/b7-ket-luan.txt"
test "$(wc -l < "$EV/b7-ket-luan.txt")" -ge 7 && echo 'PASS: da ghi ket luan B7'
```

**PASS:** dòng `PASS: da ghi ket luan B7` xuất hiện.

Ba member **cùng** nằm trong một etcd cluster và **cùng** bầu ra một leader — nhưng điều đó **không**
có nghĩa là mọi apiserver nói chuyện với mọi member. Trong stacked, `kube-apiserver` trên `lab-ha-2`
chỉ nói chuyện với etcd member trên `lab-ha-2`. Quan hệ "chung một etcd cluster" và quan hệ
"apiserver ↔ member" là hai chuyện khác nhau; nhầm hai thứ này là chỗ trượt kinh điển của bài 06. Ở
topology external, quan hệ thứ hai mới là nhiều-nhiều.

---

## B8. Workload chứng minh cluster sống

**Mục đích:** có một ứng dụng thật để bốn kịch bản hỏng ở B9–B12 đối chiếu. Chỉ dùng **Deployment**
và **Service** — hai thứ đã học ở giai đoạn 4 và 5.

```bash
# Chay tren lab-ha-1
kubectl create namespace lab-8b
kubectl -n lab-8b create deployment web --image="$TEST_IMAGE" --replicas=4 \
  -- sh -c 'mkdir -p /www && echo web-ok > /www/index.html && httpd -f -p 8080 -h /www'
kubectl -n lab-8b rollout status deployment/web --timeout=300s

kubectl -n lab-8b expose deployment web --name=web --port=80 --target-port=8080
kubectl -n lab-8b expose deployment web --name=web-np --type=NodePort --port=80 --target-port=8080
NP="$(kubectl -n lab-8b get svc web-np -o jsonpath='{.spec.ports[0].nodePort}')"
echo "NodePort = $NP"
kubectl -n lab-8b get pods -o wide | tee "$EV/b8-pods.txt"
```

```bash
# Chay tren lab-ha-1
count_on() { kubectl -n lab-8b get pods -l app=web \
  --field-selector "spec.nodeName=$1,status.phase=Running" --no-headers 2>/dev/null | wc -l; }
W4_0="$(count_on "$WN1")"; W5_0="$(count_on "$WN2")"
CP_PODS="$(kubectl -n lab-8b get pods -l app=web -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' \
  | grep -cE "^($CP1|$CP2|$CP3)$")"
echo "web tren $WN1=$W4_0 | tren $WN2=$W5_0 | tren control plane node=$CP_PODS"

test "$W4_0" -ge 1 && test "$W5_0" -ge 1 \
  && echo 'PASS: workload trai tren ca hai worker'
test "$CP_PODS" -eq 0 \
  && echo 'PASS: khong Pod thuong nao lot len control plane node — taint cach ly dang lam viec'
test "$NP" -ge 30000 && echo 'PASS: da co NodePort de goi tu ngoai cluster'
```

```bash
# Chay tren lab-ha-1 — ham dung lai o B9, B11, B12
serve_ok() {
  SERVE_OK=0
  for ip in "$WN1_IP" "$WN2_IP"; do
    C="$(curl -s -o /dev/null -w '%{http_code}' --max-time 5 "http://$ip:$NP")"
    B="$(curl -s --max-time 5 "http://$ip:$NP")"
    echo "  $ip:$NP -> http=$C body=$B"
    if test "$C" = '200' && test "$B" = 'web-ok'; then SERVE_OK=$(( SERVE_OK + 1 )); fi
  done
  echo "  so worker dang phuc vu: $SERVE_OK"
}
serve_ok
test "$SERVE_OK" -eq 2 \
  && echo 'PASS: ung dung phuc vu qua NodePort tren ca hai worker'
```

**PASS:** bốn dòng `PASS:` của B8 xuất hiện.

Hàm `serve_ok` là **thước đo mặt phẳng dữ liệu** của lab: nó gọi thẳng vào NodePort trên từng worker
và kiểm cả mã HTTP lẫn nội dung trả về. B9, B11 và B12 dùng lại nó để trả lời một câu hỏi duy nhất —
*trong lúc control plane gặp nạn, ứng dụng còn phục vụ không?* Ghi lại mốc "trước":

```bash
# Chay tren lab-ha-1
{
  echo "=== $(date -Is) — moc truoc cac kich ban hong ==="
  echo "node        : $(kubectl get nodes --no-headers | wc -l)"
  echo "control plane: $(kubectl get nodes -l node-role.kubernetes.io/control-plane --no-headers | wc -l)"
  echo "etcd member : $(etcd1 --endpoints=https://127.0.0.1:2379 member list | wc -l)"
  echo "backend UP  : $(lbstat | grep -c '=UP$')"
  echo "web tren $WN1=$W4_0, tren $WN2=$W5_0, NodePort=$NP"
} | tee "$EV/b8-moc-truoc.txt"
test "$(wc -l < "$EV/b8-moc-truoc.txt")" -ge 6 && echo 'PASS: da ghi moc truoc'
```

**PASS:** dòng `PASS: da ghi moc truoc` xuất hiện.

---
## B9. Tắt một control plane node

**Mục đích:** chứng minh cluster **vẫn phục vụ** khi mất một trong ba control plane node, và nhìn
thấy tận mắt cái mà bài [06](../06-ha-topology-vi.md) gọi là **hỏng theo cặp**.

Node bị tắt là **`lab-ha-3`**. Không tắt `lab-ha-1` (giữ phiên shell) và không tắt `lab-ha-2` (B10
cần nó lành).

### B9.1. Hai hàm đo dùng cho cả B9, B11 và B12

```bash
# Chay tren lab-ha-1
api_probe() {
  API_READ='fail'; API_WRITE='fail'
  kubectl get nodes --request-timeout=20s >/dev/null 2>&1 && API_READ='ok'
  kubectl -n lab-8b create configmap "probe-$1" --from-literal=t="$(date +%s)" \
    --request-timeout=20s >/dev/null 2>&1 && API_WRITE='ok'
  echo "  doc API=$API_READ | ghi API=$API_WRITE"
}
direct_probe() {
  DIRECT_OK=0
  : > "$EV/b-direct-$1.txt"
  for ip in "$CP1_IP" "$CP2_IP" "$CP3_IP"; do
    C="$(curl -sk -o /dev/null -w '%{http_code}' --max-time 5 "https://$ip:6443/livez")"
    echo "  https://$ip:6443/livez -> http=$C" | tee -a "$EV/b-direct-$1.txt"
    if test "$C" != '000'; then DIRECT_OK=$(( DIRECT_OK + 1 )); fi
  done
  echo "  so apiserver tra loi truc tiep: $DIRECT_OK" | tee -a "$EV/b-direct-$1.txt"
}

api_probe b9-truoc
direct_probe b9-truoc
test "$API_READ" = 'ok' && test "$API_WRITE" = 'ok' \
  && echo 'PASS: truoc khi tat may, API doc va ghi deu duoc'
test "$DIRECT_OK" -eq 3 && echo 'PASS: ca ba kube-apiserver deu tra loi khi goi thang'
```

**PASS:** hai dòng `PASS:` của B9.1 xuất hiện.

`api_probe` đo **hai** việc khác nhau, và đó là điểm mấu chốt của cả ba kịch bản sau: đọc có thể
được phục vụ từ nhiều nguồn, còn **ghi thì bắt buộc phải qua etcd và bắt buộc phải có quorum**.
`direct_probe` bỏ qua load balancer và gõ cửa từng `kube-apiserver` một; mã `000` nghĩa là không kết
nối được, mọi mã HTTP khác — kể cả `401` — đều chứng minh có một apiserver trả lời. Giữ `serve_ok`
của B8 làm thước đo mặt phẳng dữ liệu.

### B9.2. Tắt `lab-ha-3`

```bash
# Chay tren lab-ha-3
sudo shutdown -h now
```

Xác nhận trên **máy host**, PowerShell:

```powershell
$cp3 = $haVmx | Where-Object { $_ -like '*lab-ha-3*' }
$running = & $vmrun -T ws list
if ($running -notcontains $cp3) { 'PASS: lab-ha-3 da Powered off' }
else { 'CHUA TAT: doi them roi chay lai lenh nay' }
```

**PASS:** dòng `PASS: lab-ha-3 da Powered off` xuất hiện. Chưa tắt thì chờ và chạy lại; **không**
dùng `vmrun … stop … hard` trừ khi [mục 4](#4-troubleshooting-của-lab-này) chỉ dẫn.

### B9.3. Cluster vẫn phục vụ — bốn dữ kiện

```bash
# Chay tren lab-ha-1 — cho load balancer nhan ra backend chet
UPN=3
for i in $(seq 1 60); do
  UPN="$(lbstat | grep -c '=UP$')"
  test "$UPN" -eq 2 && break
  sleep 5
done
lbstat | tee "$EV/b9-lb.txt"
echo "backend UP = $UPN"
test "$UPN" -eq 2 \
  && echo 'PASS: health check TCP da loai lab-ha-3 khoi danh sach dich'
```

```bash
# Chay tren lab-ha-1 — API van phuc vu, ca doc lan ghi
api_probe b9-mot-node-chet
test "$API_READ" = 'ok' && test "$API_WRITE" = 'ok' \
  && echo 'PASS: mat mot control plane node, API van doc va GHI duoc'
```

```bash
# Chay tren lab-ha-1 — hong theo cap: mat 1 control plane instance VA 1 etcd member
ST=''
for i in $(seq 1 90); do
  ST="$(kubectl get node "$CP3" --request-timeout=20s \
        -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null)"
  test "$ST" != 'True' && break
  sleep 5
done
kubectl get nodes -o wide | tee "$EV/b9-nodes.txt"
echo "Ready condition cua $CP3 = $ST"

direct_probe b9-mot-node-chet
etcd1 --endpoints="$ETCD_EPS" endpoint health > "$EV/b9-etcd-health.txt" 2>&1 || true
cat "$EV/b9-etcd-health.txt"
HEALTHY="$(grep -c 'is healthy' "$EV/b9-etcd-health.txt")"
MEM_N="$(etcd1 --endpoints=https://127.0.0.1:2379 member list | wc -l)"
echo "kube-apiserver tra loi truc tiep=$DIRECT_OK | etcd member khoe manh=$HEALTHY/$MEM_N"

test "$ST" != 'True' && echo 'PASS: node controller da doi Ready cua lab-ha-3 khoi True'
test "$DIRECT_OK" -eq 2 && echo 'PASS: mat DONG THOI mot instance kube-apiserver'
test "$HEALTHY" -eq 2 && test "$MEM_N" -eq 3 \
  && echo 'PASS: mat DONG THOI mot etcd member — member van trong danh sach nhung khong khoe'
```

```bash
# Chay tren lab-ha-1 — mat phang du lieu
serve_ok
test "$SERVE_OK" -eq 2 \
  && echo 'PASS: ung dung van phuc vu binh thuong tren ca hai worker'
```

**PASS:** năm dòng `PASS:` của B9.3 xuất hiện.

**Ý nghĩa:** đây là hai bài học chồng lên nhau.

Thứ nhất, **cluster sống sót** vì còn hai trong ba etcd member, tức vẫn đủ quorum, và còn hai
`kube-apiserver` mà load balancer biết đường tới. Chính hai điều đó là lý do bài 06 nói HA cần
**tối thiểu ba** control plane node: ba là số lẻ nhỏ nhất chịu được mất một.

Thứ hai, **một lần tắt máy gây hai mất mát**. Bảng đếm ở trên tách chúng ra để bạn nhìn thấy: số
`kube-apiserver` đang `Running` giảm từ 3 xuống 2, **và** số etcd member khỏe mạnh giảm từ 3 xuống
2 — trong khi `member list` vẫn khai ba member, vì rời cluster là một thao tác quản trị khác, không
phải hệ quả của việc máy tắt. Bài 06 gọi đúng hiện tượng này là **hỏng hóc theo cặp**, và giải
thích vì sao "tính dự phòng bị suy giảm" ở cả hai tầng cùng lúc. Trong topology external, một lần
mất máy chỉ đánh vào một tầng.

### B9.4. Bật lại và chờ hồi phục

Chạy trên **máy host**, PowerShell:

```powershell
$cp3 = $haVmx | Where-Object { $_ -like '*lab-ha-3*' }
& $vmrun -T ws start $cp3
Start-Sleep -Seconds 5
$running = & $vmrun -T ws list
if ($running -contains $cp3) { 'PASS: lab-ha-3 dang chay' } else { 'FAIL: VM chua len' }
```

```bash
# Chay tren lab-ha-1
kubectl wait --for=condition=Ready node/"$CP3" --timeout=600s

HEALTHY=0
for i in $(seq 1 60); do
  etcd1 --endpoints="$ETCD_EPS" endpoint health > "$EV/b9-etcd-health-sau.txt" 2>&1 || true
  HEALTHY="$(grep -c 'is healthy' "$EV/b9-etcd-health-sau.txt")"
  test "$HEALTHY" -eq 3 && break
  sleep 5
done
UPN=0
for i in $(seq 1 60); do
  UPN="$(lbstat | grep -c '=UP$')"
  test "$UPN" -eq 3 && break
  sleep 5
done
direct_probe b9-sau
echo "etcd khoe manh=$HEALTHY | backend UP=$UPN | apiserver tra loi truc tiep=$DIRECT_OK"

test "$HEALTHY" -eq 3 && echo 'PASS: du ba etcd member khoe manh tro lai'
test "$UPN" -eq 3 && echo 'PASS: load balancer da dua lab-ha-3 tro lai danh sach dich'
test "$DIRECT_OK" -eq 3 && echo 'PASS: du ba kube-apiserver tra loi tro lai'
```

**PASS:** ba dòng `PASS:` xuất hiện và `lab-ha-3` trở lại `Ready`. Cả ba chỉ số hồi phục **mà bạn
không phải làm gì** ngoài việc bật máy: static Pod được kubelet dựng lại từ manifest trên đĩa, etcd
member cũ đồng bộ lại từ leader, và health check TCP của HAProxy tự đưa backend về `UP`.

---

## B10. Xoá và khôi phục `kube-apiserver.yaml` trên `lab-ha-2`

**Mục đích:** làm phép thử của checkpoint giai đoạn 8 — xoá manifest của một `kube-apiserver` rồi
khôi phục — **trên cluster HA**, nơi nó chứng minh được nhiều hơn hẳn so với cluster một control
plane. Ở đây có load balancer, nên cluster **vẫn phục vụ trong lúc một API server chết**.

Node bị đụng vào là **`lab-ha-2`**, cố ý **không phải `lab-ha-1`**: máy đang giữ phiên shell của bạn
phải nguyên vẹn.

### B10.1. Ghi checksum rồi xoá manifest

```bash
# Chay tren lab-ha-2
sudo sha256sum /etc/kubernetes/manifests/kube-apiserver.yaml | tee ~/b10-sha-truoc.txt
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml ~/kube-apiserver.yaml.bak
sudo rm /etc/kubernetes/manifests/kube-apiserver.yaml
ls /etc/kubernetes/manifests/
```

**PASS:** thư mục còn `etcd.yaml`, `kube-controller-manager.yaml`, `kube-scheduler.yaml` và
**không** còn `kube-apiserver.yaml`; file `~/kube-apiserver.yaml.bak` tồn tại.

Đây là **xoá thật**, đúng chữ của checkpoint. Bản sao `~/kube-apiserver.yaml.bak` nằm ở thư mục
home — **ngoài** `/etc/kubernetes/manifests`, nên kubelet không nhìn thấy nó và không dựng lại Pod
từ đó.

```bash
# Chay tren lab-ha-2 — cho kubelet don container
N=1
for i in $(seq 1 60); do
  N="$(sudo crictl ps --name kube-apiserver -q | wc -l)"
  test "$N" -eq 0 && break
  sleep 5
done
echo "container kube-apiserver con lai tren may nay: $N"
test "$N" -eq 0 \
  && echo 'PASS: kubelet da dung container kube-apiserver vi manifest bien mat'
sudo crictl ps | head -n 10
```

**PASS:** dòng `PASS: kubelet da dung container kube-apiserver vi manifest bien mat` xuất hiện.
`crictl ps` vẫn liệt kê `etcd`, `kube-controller-manager`, `kube-scheduler` — chỉ một trong bốn static
Pod bị gỡ.

### B10.2. Cluster vẫn phục vụ

```bash
# Chay tren lab-ha-1
UPN=3
for i in $(seq 1 60); do
  UPN="$(lbstat | grep -c '=UP$')"
  test "$UPN" -eq 2 && break
  sleep 5
done
lbstat | tee "$EV/b10-lb.txt"
grep -q "^${CP2}=DOWN$" "$EV/b10-lb.txt" \
  && echo 'PASS: health check TCP thay lab-ha-2 khong con lang nghe :6443'
test "$UPN" -eq 2 && echo 'PASS: hai backend con lai van UP'
```

```bash
# Chay tren lab-ha-1
api_probe b10-apiserver-chet
serve_ok
MIRROR="$(kubectl -n kube-system get pod "kube-apiserver-$CP2" \
  --ignore-not-found -o name 2>/dev/null | wc -l)"
NODE_RDY="$(kubectl get node "$CP2" \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
etcd1 --endpoints="$ETCD_EPS" endpoint health > "$EV/b10-etcd-health.txt" 2>&1 || true
HEALTHY="$(grep -c 'is healthy' "$EV/b10-etcd-health.txt")"
echo "mirror Pod kube-apiserver-$CP2 con=$MIRROR | node $CP2 Ready=$NODE_RDY | etcd khoe manh=$HEALTHY"

test "$API_READ" = 'ok' && test "$API_WRITE" = 'ok' \
  && echo 'PASS: mot kube-apiserver chet ma cluster van doc va ghi duoc'
test "$SERVE_OK" -eq 2 && echo 'PASS: ung dung khong he bi anh huong'
test "$MIRROR" -eq 0 && echo 'PASS: mirror Pod cua apiserver do da bien mat khoi API'
test "$NODE_RDY" = 'True' && echo 'PASS: node lab-ha-2 VAN Ready — kubelet cua no van chay'
test "$HEALTHY" -eq 3 && echo 'PASS: ca ba etcd member VAN khoe manh'
```

**PASS:** bảy dòng `PASS:` của B10.2 xuất hiện.

**Ý nghĩa — và vì sao phép thử này mạnh hơn hẳn trên cluster HA.** Trên cluster một control plane,
xoá `kube-apiserver.yaml` là làm chết cả cluster: bạn mất luôn `kubectl`, không quan sát được gì, và
mọi việc còn lại phải làm bằng `crictl` với `journalctl`. Ở đây thì khác hẳn — request của bạn đi qua
load balancer, HAProxy thấy `lab-ha-2` không bắt tay được TCP nữa nên **loại nó khỏi danh sách
đích**, và hai apiserver còn lại phục vụ tiếp. Đó chính xác là điều bài 08 hứa khi nó nói load
balancer "phân phối lưu lượng đến tất cả các node control plane **khỏe mạnh**".

Ba dữ kiện phụ cũng đáng nhớ, vì chúng vẽ ra **ranh giới của sự cố**: node vẫn `Ready` (kubelet
không chết theo), etcd member trên chính máy đó vẫn khỏe (etcd là static Pod **khác**), và chỉ mirror
Pod của apiserver biến mất. So với B9, nơi cả máy tắt và mất một lúc cả hai — đây là **một** mất mát,
không phải một cặp.

### B10.3. Khôi phục và gate checksum

```bash
# Chay tren lab-ha-2
sudo cp ~/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml
sudo sha256sum /etc/kubernetes/manifests/kube-apiserver.yaml | tee ~/b10-sha-sau.txt

N=0
for i in $(seq 1 60); do
  N="$(sudo crictl ps --name kube-apiserver -q | wc -l)"
  test "$N" -ge 1 && break
  sleep 5
done
echo "container kube-apiserver: $N"
test "$N" -ge 1 && echo 'PASS: kubelet da dung lai container tu manifest vua khoi phuc'

diff <(awk '{print $1}' ~/b10-sha-truoc.txt) <(awk '{print $1}' ~/b10-sha-sau.txt) \
  && echo 'PASS: manifest khoi phuc y het ban goc' \
  || echo 'FAIL: file khac ban goc — xem muc 4'
rm -f ~/kube-apiserver.yaml.bak ~/b10-sha-truoc.txt ~/b10-sha-sau.txt
```

```bash
# Chay tren lab-ha-1
UPN=2
for i in $(seq 1 60); do
  UPN="$(lbstat | grep -c '=UP$')"
  test "$UPN" -eq 3 && break
  sleep 5
done
MIRROR="$(kubectl -n kube-system get pod "kube-apiserver-$CP2" --ignore-not-found -o name | wc -l)"
echo "backend UP=$UPN | mirror Pod kube-apiserver-$CP2=$MIRROR"

test "$UPN" -eq 3 && echo 'PASS: load balancer da dua lab-ha-2 tro lai'
test "$MIRROR" -eq 1 && echo 'PASS: mirror Pod cua apiserver da xuat hien lai trong API'
```

**PASS:** bốn dòng `PASS:` của B10.3 xuất hiện, không dòng `FAIL:` nào.

Nếu bạn lỡ xoá manifest **mà không có bản sao**, đường khôi phục đúng là sinh lại nó bằng
`kubeadm init phase control-plane apiserver` — xem [mục 4](#4-troubleshooting-của-lab-này). Đừng
chép manifest từ node khác sang: mỗi node có `--advertise-address` và các SAN riêng.

---

## B11. Tắt hai control plane node — mất quorum

**Mục đích:** đi qua ranh giới. Đây là **bài học lõi của stacked etcd** và là lý do bài 06 nói tối
thiểu ba node: ba node chịu được mất một, **không** chịu được mất hai.

Hai node bị tắt là **`lab-ha-2`** và **`lab-ha-3`**. `lab-ha-1` ở lại — nó giữ phiên shell, và quan
trọng hơn: nó là bằng chứng sống rằng "một apiserver còn chạy" **không đồng nghĩa** với "cluster còn
phục vụ".

> **Đây là kịch bản nặng nhất của lab.** Trước khi tắt máy, xác nhận cả sáu VM đều còn mốc
> `8x-vm-ready` — đó là đường lui duy nhất nếu etcd không hồi phục. Chạy lại khối `listSnapshots`
> của [B0.6](#b06-tắt-sáu-vm-và-chụp-8x-vm-ready) nếu bạn không chắc.

### B11.1. Tắt `lab-ha-2` và `lab-ha-3`

```bash
# Chay tren lab-ha-2
sudo shutdown -h now
```

```bash
# Chay tren lab-ha-3
sudo shutdown -h now
```

Xác nhận trên **máy host**, PowerShell:

```powershell
$down = $haVmx | Where-Object { $_ -like '*lab-ha-2*' -or $_ -like '*lab-ha-3*' }
$running = & $vmrun -T ws list
$still = $down | Where-Object { $running -contains $_ }
if ($still.Count -eq 0) { 'PASS: ca lab-ha-2 va lab-ha-3 da Powered off' }
else { "CHUA TAT: $($still -join ', ')" }
```

**PASS:** dòng `PASS: ca lab-ha-2 va lab-ha-3 da Powered off` xuất hiện.

### B11.2. API ngừng phục vụ

```bash
# Chay tren lab-ha-1
UPN=3
for i in $(seq 1 60); do
  UPN="$(lbstat | grep -c '=UP$')"
  test "$UPN" -eq 1 && break
  sleep 5
done
lbstat | tee "$EV/b11-lb.txt"
test "$UPN" -eq 1 \
  && echo 'PASS: chi con mot backend UP — dung mot kube-apiserver con song'
```

```bash
# Chay tren lab-ha-1
api_probe b11-mat-quorum
test "$API_WRITE" != 'ok' \
  && echo 'PASS: GHI vao API that bai — mat quorum thi khong ghi duoc etcd'
test "$API_READ" != 'ok' \
  && echo 'PASS: DOC tu API cung that bai'

kubectl get --raw='/readyz?verbose' --request-timeout=20s > "$EV/b11-readyz.txt" 2>&1 || true
head -n 20 "$EV/b11-readyz.txt"
test -s "$EV/b11-readyz.txt" && echo 'PASS: da ghi lai phan hoi cua /readyz luc mat quorum'
kubectl get --raw='/readyz' --request-timeout=20s 2>/dev/null | grep -qx 'ok' \
  && echo 'FAIL: readyz van tra ok — kiem tra lai hai VM da tat chua' \
  || echo 'PASS: /readyz khong con tra ok'
```

Hỏi thẳng etcd. `kubectl` đã chết nên không `exec` vào Pod được nữa; dùng **`crictl`** nói chuyện
trực tiếp với container runtime, đúng kỹ năng mà checkpoint giai đoạn 8 yêu cầu:

```bash
# Chay tren lab-ha-1
CID="$(sudo crictl ps --name etcd -q | head -n 1)"
echo "container etcd tren may nay: $CID"
test -n "$CID" && echo 'PASS: container etcd van dang chay — tien trinh song, chi la khong co quorum'

sudo crictl exec "$CID" etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health > "$EV/b11-etcd-health.txt" 2>&1 || true
cat "$EV/b11-etcd-health.txt"

grep -q 'is healthy' "$EV/b11-etcd-health.txt" \
  && echo 'FAIL: etcd van bao khoe — hai node kia chua that su tat' \
  || echo 'PASS: etcd member con lai bao KHONG khoe du tien trinh van chay'
```

**PASS:** sáu dòng `PASS:` của B11.2 xuất hiện, không dòng `FAIL:` nào.

### B11.3. Mặt phẳng dữ liệu vẫn phục vụ

```bash
# Chay tren lab-ha-1
serve_ok
test "$SERVE_OK" -eq 2 \
  && echo 'PASS: ung dung VAN phuc vu tren ca hai worker du control plane da chet'
```

**PASS:** dòng `PASS: ung dung VAN phuc vu tren ca hai worker du control plane da chet` xuất hiện.

**Ý nghĩa — hai kết luận, đừng lẫn vào nhau.**

**Một: cluster mất quorum thì API chết, kể cả khi còn một apiserver chạy.** etcd là hệ thống đồng
thuận: mỗi lần ghi phải được **đa số** member xác nhận. Ba member thì đa số là hai; còn một member
là không bao giờ đạt đa số, nên mọi lần ghi treo rồi hết hạn. Đọc cũng hỏng theo, vì đọc nhất quán
phải xác nhận với quorum trước khi trả lời. Tiến trình `etcd` và `kube-apiserver` trên `lab-ha-1`
vẫn chạy — `crictl ps` chứng minh — nhưng chúng **không phục vụ được**. Đây là chỗ trực giác hay
sai: "còn máy sống" không bằng "còn cluster".

Và vì đây là topology **stacked**, hai lần tắt máy lấy đi **hai apiserver và hai etcd member cùng
lúc**. Nếu etcd nằm ở host riêng, tắt hai control plane node sẽ không đụng tới quorum của etcd chút
nào — cluster mất khả năng dự phòng ở tầng control plane nhưng dữ liệu vẫn có quorum. Đó là phần
"tách rời" mà bài 06 nói tới, và cũng là toàn bộ lý do Lab 8c tồn tại.

**Hai: workload đang chạy không phụ thuộc control plane để phục vụ request.** Pod đã chạy vẫn chạy;
`kube-proxy` trên mỗi worker đã ghi sẵn quy tắc chuyển tiếp cho Service, và những quy tắc đó nằm
trong kernel của chính node, không phải trong API server. Cái bạn **mất** là khả năng **thay đổi**:
không deploy được, không scale được, Pod chết đi thì không ai tạo lại, node mới không join được.
Nói cách khác — mất control plane là mất **quyền điều khiển**, chưa phải mất **dịch vụ**. Đừng lấy
điều đó làm cớ để coi nhẹ sự cố: cluster ở trạng thái này chỉ chờ một Pod chết là bắt đầu suy sụp.

### B11.4. Bật lại và chờ quorum quay về

Chạy trên **máy host**, PowerShell:

```powershell
$back = $haVmx | Where-Object { $_ -like '*lab-ha-2*' -or $_ -like '*lab-ha-3*' }
foreach ($f in $back) { & $vmrun -T ws start $f; Start-Sleep -Seconds 10 }
$running = & $vmrun -T ws list
$up = ($back | Where-Object { $running -contains $_ }).Count
if ($up -eq 2) { 'PASS: ca hai VM dang chay lai' } else { "FAIL: moi co $up/2" }
```

```bash
# Chay tren lab-ha-1 — vong cho co dieu kien dung, khong dem giay
for i in $(seq 1 120); do
  kubectl get nodes --request-timeout=10s >/dev/null 2>&1 && break
  sleep 5
done
api_probe b11-hoi-phuc
test "$API_READ" = 'ok' && test "$API_WRITE" = 'ok' \
  && echo 'PASS: du hai member la co quorum — API doc va ghi lai duoc'

kubectl wait --for=condition=Ready node --all --timeout=600s
HEALTHY=0
for i in $(seq 1 60); do
  etcd1 --endpoints="$ETCD_EPS" endpoint health > "$EV/b11-etcd-health-sau.txt" 2>&1 || true
  HEALTHY="$(grep -c 'is healthy' "$EV/b11-etcd-health-sau.txt")"
  test "$HEALTHY" -eq 3 && break
  sleep 5
done
UPN=0
for i in $(seq 1 60); do
  UPN="$(lbstat | grep -c '=UP$')"
  test "$UPN" -eq 3 && break
  sleep 5
done
echo "etcd khoe manh=$HEALTHY | backend UP=$UPN"

test "$HEALTHY" -eq 3 && echo 'PASS: du ba etcd member khoe manh tro lai'
test "$UPN" -eq 3 && echo 'PASS: du ba backend UP tro lai'
```

**PASS:** bốn dòng `PASS:` của B11.4 xuất hiện, không dòng `FAIL:` nào.

Đáng chú ý: cluster hồi phục **ngay khi member thứ hai lên**, không cần chờ đủ ba — hai trên ba đã là
đa số. Đó là mặt còn lại của cùng một phép đếm đã làm cluster chết ở B11.2.

---

## B12. Load balancer là SPOF: chứng minh và ghi nhận

**Mục đích:** không để lab kết thúc với ấn tượng sai rằng cluster này đã HA hoàn chỉnh. Ba API
server đã dự phòng cho nhau; **cái chưa có ai dự phòng là load balancer**.

### B12.1. Tắt `lab-ha-lb`

```bash
# Chay tren lab-ha-lb
sudo shutdown -h now
```

Xác nhận trên **máy host**, PowerShell:

```powershell
$lb = $haVmx | Where-Object { $_ -like '*lab-ha-lb*' }
$running = & $vmrun -T ws list
if ($running -notcontains $lb) { 'PASS: lab-ha-lb da Powered off' }
else { 'CHUA TAT: doi them roi chay lai lenh nay' }
```

**PASS:** dòng `PASS: lab-ha-lb da Powered off` xuất hiện.

### B12.2. Mọi client mất endpoint, nhưng ba API server vẫn sống

```bash
# Chay tren lab-ha-1
api_probe b12-lb-chet
test "$API_READ" != 'ok' && test "$API_WRITE" != 'ok' \
  && echo 'PASS: kubectl mat duong vao — kubeconfig tro vao endpoint da chet'
```

```bash
# Chay tren lab-ha-1 — goi thang tung apiserver, khong qua load balancer
direct_probe b12-lb-chet
test "$DIRECT_OK" -eq 3 \
  && echo 'PASS: ca ba kube-apiserver van song va van tra loi tren :6443'

serve_ok
test "$SERVE_OK" -eq 2 && echo 'PASS: ung dung van phuc vu binh thuong'
```

**PASS:** ba dòng `PASS:` của B12.2 xuất hiện.

**Ý nghĩa.** Không có gì hỏng trong cluster: ba control plane node đủ, ba etcd member đủ quorum, ba
apiserver trả lời khi gọi thẳng vào IP của chúng, workload phục vụ bình thường. Thứ duy nhất chết là
**cái tên** mà mọi client được dạy để gọi. Vì `controlPlaneEndpoint` là `lab-ha-lb:6443`, nó nằm
trong kubeconfig của quản trị viên, trong ConfigMap `cluster-info` mà node dùng lúc join, và trong
cấu hình mọi kubelet — nên tất cả cùng mất đường vào một lúc.

Sửa "tạm" bằng cách trỏ `kubectl` thẳng vào một apiserver **không phải là cách sửa**: bạn chỉ tự cứu
được cái terminal của mình, còn node đang join, kubelet và mọi client khác vẫn nhìn vào endpoint cũ.
Cách sửa đúng là **làm cho endpoint đó không chết được**: hai máy LB cộng một địa chỉ VIP trôi giữa
chúng, theo hướng bài 08 trỏ tới ở *Các lựa chọn cho cân bằng tải bằng phần mềm*. Lab dừng ở đây và
ghi nhận, vì thêm một VM và một VIP vượt ngân sách tài nguyên đã gate ở
[mục 2.3](#23-gate-tài-nguyên-host); lý do đầy đủ nằm trong bảng "không kiểm chứng được" ở
[mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

### B12.3. Bật lại và ghi nhận vào evidence

Chạy trên **máy host**, PowerShell:

```powershell
$lb = $haVmx | Where-Object { $_ -like '*lab-ha-lb*' }
& $vmrun -T ws start $lb
Start-Sleep -Seconds 5
$running = & $vmrun -T ws list
if ($running -contains $lb) { 'PASS: lab-ha-lb dang chay' } else { 'FAIL: VM chua len' }
```

```bash
# Chay tren lab-ha-1
for i in $(seq 1 120); do
  kubectl get nodes --request-timeout=10s >/dev/null 2>&1 && break
  sleep 5
done
api_probe b12-hoi-phuc
UPN=0
for i in $(seq 1 60); do
  UPN="$(lbstat | grep -c '=UP$')"
  test "$UPN" -eq 3 && break
  sleep 5
done
echo "backend UP=$UPN"

test "$API_READ" = 'ok' && test "$API_WRITE" = 'ok' \
  && echo 'PASS: load balancer len lai, kubectl hoat dong tro lai'
test "$UPN" -eq 3 && echo 'PASS: du ba backend UP'
```

```bash
# Chay tren lab-ha-1
{
  echo "=== $(date -Is) — bon kich ban hong cua Lab 8b ==="
  echo "B9  tat 1 control plane node : API doc+ghi OK | etcd 2/3 khoe | ung dung OK"
  echo "B10 xoa manifest apiserver   : API doc+ghi OK | etcd 3/3 khoe | node van Ready | ung dung OK"
  echo "B11 tat 2 control plane node : API CHET (mat quorum) | etcd khong khoe | ung dung VAN OK"
  echo "B12 tat load balancer        : API CHET (mat endpoint) | ba apiserver VAN song | ung dung VAN OK"
  echo "Ket luan: 3 control plane chiu duoc mat 1, khong chiu duoc mat 2."
  echo "Ket luan: mot load balancer duy nhat van la SPOF — can cap LB + VIP moi HA hoan chinh."
} | tee "$EV/b12-tong-ket.txt"
test "$(wc -l < "$EV/b12-tong-ket.txt")" -ge 7 && echo 'PASS: da ghi tong ket bon kich ban'
```

**PASS:** ba dòng `PASS:` của B12.3 xuất hiện, và bảng tổng kết đọc thành một câu chuyện liền mạch.

---
## B13. Cleanup, snapshot `8x-ha-stacked` và gate cuối

**Mục đích:** xoá mọi thứ lab tạo ra, chứng minh cluster HA còn nguyên vẹn sau bốn kịch bản hỏng,
chụp mốc cuối trên **cả sáu VM**, rồi chứng minh bộ VM chính không hề bị đụng tới.

### B13.1. Xoá object của bài học

```bash
# Chay tren lab-ha-1
kubectl delete namespace lab-8b --wait=true --timeout=300s

rm -f "$WK/join-commands.txt" "$WK/join-control-plane.txt" "$WK/join-worker.txt" \
      "$WK/mau-certificate-key.txt"
rmdir "$WK"
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và gate bên dưới biến điều đó thành
lỗi thấy được thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/8b/` **giữ lại** — đó là bằng chứng.

```bash
# Chay tren lab-ha-1
NS_LEFT="$(kubectl get namespace lab-8b --ignore-not-found -o name 2>/dev/null | wc -l)"
POD_DEF="$(kubectl get pods -n default --no-headers 2>/dev/null | wc -l)"
SVC_DEF="$(kubectl get svc -n default --no-headers 2>/dev/null | wc -l)"
echo "namespace lab-8b con=$NS_LEFT | Pod trong default=$POD_DEF | Service trong default=$SVC_DEF"

test "$NS_LEFT" -eq 0 && echo 'PASS: namespace lab-8b da bien mat'
test "$POD_DEF" -eq 0 && echo 'PASS: namespace default khong con Pod nao'
test "$SVC_DEF" -eq 1 && echo 'PASS: default chi con Service kubernetes'
test ! -e "$WK" && echo 'PASS: thu muc manifest tam da xoa het'
```

**PASS:** bốn dòng `PASS:` của B13.1 xuất hiện.

### B13.2. Gate quan trọng nhất: cluster HA còn nguyên vẹn

Bốn kịch bản ở B9–B12 đã tắt máy sáu lần. Gate này chứng minh không lần nào để lại di chứng.

```bash
# Chay tren lab-ha-1
kubectl wait --for=condition=Ready node --all --timeout=600s
kubectl get nodes -o wide | tee "$EV/b13-nodes.txt"

ALL_N="$(kubectl get nodes -o name | wc -l)"
CP_N="$(kubectl get nodes -l node-role.kubernetes.io/control-plane -o name | wc -l)"
RDY_N="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}' \
  | grep -c '^True$')"
UNSCH_N="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{.spec.unschedulable}{"\n"}{end}' | grep -c 'true')"
TAINT_BAD="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}' \
  | grep -cE 'node.kubernetes.io/(unschedulable|unreachable|not-ready)')"
echo "node=$ALL_N | control plane=$CP_N | Ready=$RDY_N | cordon=$UNSCH_N | taint bao tri sot=$TAINT_BAD"

test "$ALL_N" -eq 5 && test "$CP_N" -eq 3 && test "$RDY_N" -eq 5 \
  && echo 'PASS: van du 5 node, 3 control plane, tat ca Ready'
test "$UNSCH_N" -eq 0 && test "$TAINT_BAD" -eq 0 \
  && echo 'PASS: khong node nao con cordon hay taint bao tri sot lai'
```

```bash
# Chay tren lab-ha-1
etcd1 --endpoints="$ETCD_EPS" endpoint health > "$EV/b13-etcd-health.txt" 2>&1 || true
etcd1 --endpoints=https://127.0.0.1:2379 member list -w table \
  | tee "$EV/b13-etcd-members.txt"
HEALTHY="$(grep -c 'is healthy' "$EV/b13-etcd-health.txt")"
MEM_N="$(etcd1 --endpoints=https://127.0.0.1:2379 member list | wc -l)"
STATIC="$(kubectl -n kube-system get pod -l component=etcd \
  -o jsonpath='{range .items[*]}{.metadata.annotations.kubernetes\.io/config\.source}{"\n"}{end}' \
  | grep -c '^file$')"
lbstat | tee "$EV/b13-lb.txt"
UPN="$(grep -c '=UP$' "$EV/b13-lb.txt")"
direct_probe b13
echo "etcd member=$MEM_N khoe manh=$HEALTHY | static Pod etcd=$STATIC | backend UP=$UPN"

test "$MEM_N" -eq 3 && test "$HEALTHY" -eq 3 \
  && echo 'PASS: van du ba etcd member va ca ba deu khoe manh'
test "$STATIC" -eq 3 \
  && echo 'PASS: topology van la stacked — ba static Pod etcd tren ba control plane node'
test "$UPN" -eq 3 && test "$DIRECT_OK" -eq 3 \
  && echo 'PASS: load balancer va ca ba kube-apiserver deu hoat dong'
```

**PASS:** năm dòng `PASS:` của B13.2 xuất hiện.

### B13.3. Hồ sơ cuối của cluster

```bash
# Chay tren lab-ha-1
{
  echo "=== $(date -Is) — trang thai ket thuc Lab 8b ==="
  echo '--- nodes';            kubectl get nodes -o wide
  echo '--- control plane';    kubectl get nodes -l node-role.kubernetes.io/control-plane
  echo '--- static Pod etcd';  kubectl -n kube-system get pod -l component=etcd -o wide
  echo '--- etcd member';      cat "$EV/b13-etcd-members.txt"
  echo '--- etcd health';      cat "$EV/b13-etcd-health.txt"
  echo '--- controlPlaneEndpoint'
  kubectl -n kube-system get configmap kubeadm-config \
    -o jsonpath='{.data.ClusterConfiguration}' | grep -E 'controlPlaneEndpoint|podSubnet'
  echo
  echo '--- load balancer backend'; cat "$EV/b13-lb.txt"
  echo '--- namespaces';       kubectl get namespaces
} | tee "$EV/b13-ho-so-cuoi.txt"

test "$(wc -l < "$EV/b13-ho-so-cuoi.txt")" -ge 25 && echo 'PASS: da ghi ho so cuoi'
ls -1 "$EV" | tee "$EV/b13-danh-muc-evidence.txt"
test "$(wc -l < "$EV/b13-danh-muc-evidence.txt")" -ge 15 \
  && echo 'PASS: thu muc evidence co du bang chung cua cac buoc'
```

**PASS:** hai dòng `PASS:` của B13.3 xuất hiện, và danh sách namespace **không** còn `lab-8b`.

### B13.4. Gate ngắn phiên bản HA

Đây là bản tương đương của
[quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab), sửa cho cluster
năm node và bổ sung ba dòng riêng của topology HA. **Lab 8c mở đầu bằng đúng gate này**, sau khi
restore `8x-ha-stacked`.

```bash
# Chay tren lab-ha-1
kubectl wait --for=condition=Ready node --all --timeout=300s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Ba dong rieng cua topology HA
kubectl get nodes -l node-role.kubernetes.io/control-plane
etcd1 --endpoints="$ETCD_EPS" endpoint health || true
lbstat
```

**PASS:** năm node `Ready`; dòng `PASS: readyz ok` xuất hiện; lệnh `--field-selector` trả
`No resources found`; CoreDNS đủ replica `READY`; namespace `default` không còn Pod; **ba** node mang
role `control-plane`; **ba** dòng `is healthy`; **ba** backend `=UP`.

Gate ngắn này không tạo resource nào. Fail ở đây thì xử lý theo
[mục 4](#4-troubleshooting-của-lab-này) trước khi chụp mốc — chụp một cluster đang lệch là ghi lỗi
vào mốc mà Lab 8c sẽ khôi phục.

### B13.5. Tắt sáu VM và chụp `8x-ha-stacked`

Tắt theo thứ tự ngược với thứ tự bật: `lab-ha-5` → `lab-ha-4` → `lab-ha-3` → `lab-ha-2` →
`lab-ha-1` → `lab-ha-lb`. Trên **từng máy**:

```bash
sudo shutdown -h now
```

Chờ VMware Workstation hiển thị cả sáu VM ở *Powered off*, rồi chụp trên **từng VM** với tên đúng
nguyên văn:

```text
8x-ha-stacked
```

Ô *Description* ghi `cluster HA stacked etcd dung bang LAB-8B, chup <ngay>`.

Verify trên **máy host**, PowerShell:

```powershell
foreach ($f in $haVmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  $a = $names -ccontains '8x-vm-ready'
  $b = $names -ccontains '8x-ha-stacked'
  if ($a -and $b) { "PASS: $f — co ca hai moc" }
  else { "FAIL: $f -> vm-ready=$a ha-stacked=$b | $($names -join ', ')" }
}
```

**PASS:** đúng sáu dòng `PASS:`, không dòng `FAIL:` nào. **Giữ cả hai mốc**: `8x-vm-ready` là đường
lui về bộ VM trắng, và Lab 8c cần chính nó để dựng lại topology external từ đầu.

Gate cuối cùng — **bộ VM chính phải còn nguyên như lúc bắt đầu lab**. Chạy trên **máy host**:

```powershell
$running = & $vmrun -T ws list | Select-Object -Skip 1 | Where-Object { $_ -match '\.vmx$' }
if ($running.Count -eq 0) { 'PASS: khong VM nao dang chay' } else { "FAIL: con chay $($running -join ', ')" }

foreach ($f in $mainVmx) {
  if (-not (Test-Path $f)) { "FAIL: mat file $f"; continue }
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '03-storage-ready') { "PASS: $f — con nguyen moc 03-storage-ready" }
  else { "FAIL: $f -> $($names -join ', ')" }
}
```

**PASS:** một dòng `PASS: khong VM nao dang chay` cộng đúng ba dòng `PASS:` cho ba VM chính, không
dòng `FAIL:` nào.

Lab 8b kết thúc ở đây. Bộ VM HA nằm ở mốc `8x-ha-stacked`, bộ VM chính nằm ở `03-storage-ready`, và
hai chuỗi không giao nhau ở bất kỳ điểm nào.

---

## 3. Checkpoint 8b

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Vẽ hai topology của bài 06 lên giấy. Trong stacked, etcd member trên `lab-ha-2` phục vụ những
      `kube-apiserver` nào? Trong external thì quan hệ đó khác ra sao? Nêu **số host tối thiểu** của
      từng bên, và tính xem sáu VM của bạn đủ cho bên nào.
- [ ] Bạn đã chứng minh cluster của mình là stacked bằng **ba dữ kiện độc lập**. Kể đủ ba, và nói
      annotation `kubernetes.io/config.source` bằng `file` chứng minh điều gì mà `kubectl get pod`
      thường không nói ra.
- [ ] `--upload-certs` để lại **hiện vật gì**, ở namespace nào, sống được **bao lâu**, và ai là chủ
      sở hữu của nó? Bạn `init` lúc 09:00, đến 12:00 mới join control plane node thứ hai — chuyện gì
      xảy ra và bạn gõ lệnh nào để sửa? Certificate key có hình dạng thế nào và vì sao?
- [ ] `kubeadm init` in ra hai lệnh join. Chúng khác nhau **đúng mấy cờ**, là những cờ nào, và mỗi
      cờ khiến `kubeadm join` làm thêm việc gì? Token và `--discovery-token-ca-cert-hash` của hai
      lệnh có khác nhau không — nếu không thì hai giá trị đó trả lời câu hỏi gì?
- [ ] Trước khi `init`, bạn thử kết nối từ load balancer tới `:6443` của một control plane node và
      nhận `connection refused`. Đi tiếp hay dừng lại sửa? Còn nếu là **timeout**? Vì sao khi thử
      chiều ngược lại — từ node tới `:6443` của LB — bạn **không** thấy `refused` như bài 08 mô tả,
      và điều đó có phải lỗi không?
- [ ] `controlPlaneEndpoint` của bạn được ghi lại ở **ba nơi** trong cluster. Kể tên ba nơi đó và
      nói mỗi nơi phục vụ nhóm khách nào. Nếu nó trỏ vào IP của `lab-ha-1` thay vì load balancer thì
      cluster hỏng ở đâu, và bạn có sửa được sau khi đã `init` không?
- [ ] Bạn tắt `lab-ha-3`. Kể **hai** mất mát xảy ra cùng lúc và nói tên hiện tượng đó theo bài 06.
      Vì sao cluster vẫn phục vụ? Bạn tắt tiếp `lab-ha-2` — chuyện gì xảy ra với thao tác **ghi**,
      với thao tác **đọc**, và vì sao "trên `lab-ha-1` vẫn còn một apiserver đang chạy" không cứu
      được gì?
- [ ] Trong lúc mất quorum, ứng dụng của bạn **vẫn** trả về `web-ok` trên cả hai worker. Giải thích
      vì sao, nói rõ cái bạn **mất** là gì, và vì sao đó không phải lý do để coi nhẹ sự cố.
- [ ] Bạn xoá `/etc/kubernetes/manifests/kube-apiserver.yaml` trên `lab-ha-2`. Vì sao cluster vẫn
      phục vụ được, và thành phần nào ra quyết định bỏ qua node đó? Ba thứ **không** hỏng theo là
      gì? Nếu làm đúng phép này trên cluster một control plane thì khác ra sao?
- [ ] Sau khi có đủ năm node, cả hai Pod CoreDNS vẫn nằm trên `lab-ha-1`. Đây là lỗi hay hệ quả bình
      thường? Bài 08 chỉ cách xử lý nào, và **vì sao lệnh đó có tác dụng** trong khi Kubernetes
      không tự dời Pod đang chạy?
- [ ] Bạn tắt `lab-ha-lb` và mất `kubectl`, dù ba API server đều trả lời khi gọi thẳng IP. Đây là
      lỗi cấu hình hay giới hạn thiết kế của lab? Vì sao trỏ `kubectl` thẳng vào một apiserver
      **không** phải cách sửa? Production cần thêm đúng những gì?
- [ ] **Nợ #8 chưa được trả ở lab này.** Món nợ đó là gì, phát sinh ở giai đoạn nào, vì sao lab HA
      có ba bản etcd rồi mà vẫn còn nợ nó, và đường lui duy nhất của bạn hiện giờ là gì?

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng sau, không nhìn tài liệu:

1. **Từ một máy trống tới cluster HA, theo đúng thứ tự.** Bắt đầu từ lúc bạn quyết định chọn
   topology. Kể đủ các chặng: vì sao phải dựng **load balancer trước**, cái gì được kiểm trước khi
   `init` và hai kết quả trái ngược của phép kiểm đó nghĩa là gì; `kubeadm init` được truyền những
   cờ nào và **cờ nào là cờ biến cluster này thành HA được**; vì sao CNI phải cài **ngay sau init và
   trước khi join**; hai lệnh join khác nhau ở đâu và mỗi cái thêm gì vào cluster; cuối cùng là hai
   việc dọn dẹp sau khi đủ node — cân bằng lại CoreDNS, và kiểm rằng load balancer thật sự rải
   request. Kết bằng câu trả lời cho: nếu bỏ `--control-plane-endpoint` lúc init thì hỏng ở đâu, và
   vì sao đó là quyết định **một chiều**.
2. **Bốn kịch bản hỏng, xếp theo mức độ.** Đi từ nhẹ tới nặng: một apiserver mất manifest, một
   control plane node tắt, hai control plane node tắt, load balancer tắt. Với mỗi kịch bản nói ba
   điều — cái gì hỏng, cluster **còn phục vụ không**, và **vì sao**. Rồi rút ra ba ranh giới: ranh
   giới **quorum** (bao nhiêu trên bao nhiêu, và vì sao ba là số nhỏ nhất có ý nghĩa), ranh giới
   **control plane với mặt phẳng dữ liệu** (cái gì chết mà ứng dụng vẫn sống, và bạn mất quyền gì),
   và ranh giới **stacked với external** (kịch bản nào sẽ diễn ra khác đi nếu etcd nằm ở host riêng
   — đó chính là nội dung Lab 8c). Kết bằng một câu về thứ mà lab này **vẫn chưa** làm cho HA được.

Khi mọi checkbox được đánh dấu và bạn không còn nhầm *stacked* với *external*, *mất một node* với
*mất quorum*, *control plane chết* với *dịch vụ chết*, hay *`--control-plane-endpoint`* với
*`--apiserver-advertise-address`* — Lab 8b hoàn tất.

Lab này **phát sinh không nợ mới** và **không trả nợ nào**. Riêng
[nợ #8](README.md#5-sổ-nợ-lab) — backup và khôi phục etcd — vẫn **đang treo**: bạn vừa dựng một
cluster có ba bản etcd đồng bộ, nhưng ba bản của **cùng một** dữ liệu không bảo vệ được bạn khỏi
việc xoá nhầm hay hỏng dữ liệu, vì lỗi được nhân bản y hệt sang cả ba. Đường lui duy nhất hiện giờ
vẫn là snapshot VM. Nợ được trả ở
[giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa). Những phần
cố ý không làm còn lại nằm trong bảng lý do ở
[mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

---

## 4. Troubleshooting của lab này

Sự cố **dựng môi trường** — VM không ping nhau, swap, containerd, package sai version, node
`NotReady` sau join — nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Sự cố **của chính
kubeadm** tra ở [bài 09 — Xử lý sự cố kubeadm](../09-troubleshooting-kubeadm-vi.md); đó là trang tra
cứu, mở đúng mục cần chứ không đọc tuần tự. Bảng dưới chỉ liệt kê sự cố phát sinh từ nội dung bài
học của Lab 8b.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| **B3.4: kết quả là TIMEOUT chứ không phải refused** | trên `lab-ha-lb`: `ping <ip node>`; `ufw status` trên node; `ip route` | Đây là tình huống bài 08 bảo **dừng lại**: load balancer không giao tiếp được với control plane node. Sửa mạng hoặc firewall theo [A4.4](LAB-00-MOI-TRUONG-1.35.7.md#a44-firewall-cho-mạng-lab-cô-lập) trước; **tuyệt đối chưa chạy `kubeadm init`**, vì init với endpoint không tới được sẽ để lại state bẩn phải `kubeadm reset` |
| B3.5 hoặc B6.4: `lbstat` không in gì | `curl -s "http://$LB_IP:8404/stats;csv" \| head -n 2` | Hoặc HAProxy chưa chạy, hoặc khối `listen stats` chưa được thêm. Nếu có output mà `awk` rỗng thì tên backend trong file cấu hình khác `k8s-controlplane` — sửa lại tên trong hàm. Cột trạng thái là cột 18, cột `stot` là cột 8; dòng đầu của CSV là header bắt đầu bằng `# pxname` và có thể dùng để đếm lại chỉ số nếu bản HAProxy của bạn khác |
| B4.1: `kubeadm init` treo ở bước chờ control plane | `sudo crictl ps`; `sudo journalctl -u kubelet -n 80 --no-pager`; `sudo crictl logs <id apiserver>` | Xem mục *kubeadm bị treo khi chờ control plane* của [bài 09](../09-troubleshooting-kubeadm-vi.md). Nguyên nhân riêng của lab HA: `--control-plane-endpoint` trỏ vào một LB mà chính node không tới được. Kiểm lại phép thử chiều node → LB ở B3.4 |
| B4.4: `ownerReferences` của `kubeadm-certs` rỗng | `kubectl -n kube-system get secret kubeadm-certs -o yaml \| head -n 20`; `sudo kubeadm token list` | Không đổi kết luận của bài 08 — hạn vẫn là hai giờ. Đọc hạn ở cột `EXPIRES` của `kubeadm token list`, ghi giá trị quan sát được vào `$EV/b4-upload-certs.txt` thay cho phép tính `$DIFF`, rồi đi tiếp |
| **B6.2: join control plane báo lỗi giải mã certificate** | thời điểm chạy `kubeadm init phase upload-certs`; `kubectl -n kube-system get secret kubeadm-certs` | Khóa giải mã hoặc Secret đã hết hạn hai giờ. Chạy lại **B6.1 trọn vẹn** trên `lab-ha-1` để sinh khóa mới và token mới, rồi join lại. Đây đúng là kịch bản bài 08 cảnh báo; đừng `kubeadm init` lại cluster |
| B6.2: join báo port `2379`/`2380` hoặc `6443` đang bận | trên node đó: `sudo ss -lntp \| grep -E '2379\|2380\|6443'`; `ls /etc/kubernetes/manifests/` | Node còn dấu vết một lần join trước. Chạy `sudo kubeadm reset -f` **trên chính node đó**, xoá `/etc/kubernetes` còn sót, rồi join lại. `kubeadm reset` là dọn dẹp best-effort và **không** đặt lại iptables; nếu sau reset vẫn hỏng, restore node đó về `8x-vm-ready` — nhưng khi đó phải restore **cả sáu VM**, xem [mục 2.5](#25-trình-tự-an-toàn-khi-tắt-và-bật-máy) |
| B6.2: node control plane join xong nhưng `member list` không tăng | `kubectl -n kube-system get pod -l component=etcd -o wide`; trên node mới: `sudo crictl logs <id etcd>` | etcd member mới không bắt tay được với hai member cũ trên port `2380`. Kiểm UFW đã tắt trên **cả ba** control plane node và kiểm đồng hồ: `timedatectl` phải báo đồng bộ trên cả ba. Xem thêm mục *Các pod etcd khởi động lại liên tục* của [bài 09](../09-troubleshooting-kubeadm-vi.md) |
| B6.5: sau `rollout restart`, CoreDNS vẫn dồn một node | `kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide`; `kubectl -n kube-system describe pod <ten>` | Hai replica mà cluster mới chỉ vừa đủ node Ready thì scheduler có thể xếp trùng. Chờ đủ năm node `Ready` rồi `rollout restart` một lần nữa. **Không** ép bằng `nodeSelector`: đó là sửa triệu chứng, và ghi chú của bài 08 chỉ nói tới `rollout restart` |
| B8: NodePort không trả `200` trên một trong hai worker | `kubectl -n lab-8b get pods -o wide`; `kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide`; UFW trên worker đó | Đi từ dưới lên như [tầng 5 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a546-tầng-5--service-endpointslice-và-kube-proxy): kube-proxy phải có một Pod `Running` trên **mỗi** node, và UFW phải `inactive`. Nếu cả hai đều đúng mà vẫn hỏng, chạy [tầng 3 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a544-tầng-3--pod-networking-xuyên-node) để kiểm mạng Pod xuyên node trước |
| B9.4 hoặc B11.4: node không trở lại `Ready` sau khi bật máy | trên host: `vmrun -T ws list`; trên node: `systemctl is-active kubelet containerd`; `journalctl -u kubelet -n 80 --no-pager` | Đi từ dưới lên. **(1)** VM không boot → xem console VMware. **(2)** boot mà `ssh` chết → IP tĩnh không lên, sửa ở tầng netplan ([A3](LAB-00-MOI-TRUONG-1.35.7.md#a3-đặt-ip-tĩnh-và-phân-giải-tên)). **(3)** `ssh` được mà kubelet `inactive` → thường là swap bật lại sau reboot vì `/etc/fstab` còn dòng swap chưa comment; xử lý theo [A4.1](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites). **(4)** kubelet chạy mà Node vẫn `NotReady` → kiểm CNI Pod trên node đó. Không sửa được trong vài phút thì tắt **cả sáu VM** và restore **cả sáu** về `8x-vm-ready` |
| **B10.3: `diff` báo checksum khác bản gốc** | `diff ~/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml` | Bạn đã sửa file, hoặc chép nhầm manifest của node khác. Mỗi node có `--advertise-address` và bộ SAN riêng, nên **không** chép manifest giữa các node. Sinh lại đúng cho node đó bằng `sudo kubeadm init phase control-plane apiserver --control-plane-endpoint lab-ha-lb:6443 --apiserver-advertise-address <IP cua chinh node do>`, rồi chạy lại gate |
| B10: lỡ xoá manifest mà không có bản sao | `ls /etc/kubernetes/manifests/`; `ls /etc/kubernetes/pki/` | Dùng đúng lệnh `kubeadm init phase control-plane apiserver` ở dòng trên. Certificate trong `/etc/kubernetes/pki` vẫn còn nên không cần join lại node |
| B11.2: `/readyz` vẫn trả `ok` sau khi tắt hai node | trên host: `vmrun -T ws list`; `lbstat` | Hai VM chưa thật sự tắt, hoặc bạn tắt nhầm máy. Xác nhận `lbstat` chỉ còn **một** dòng `=UP` rồi mới kết luận. Đừng nghi ngờ phép thử trước khi xác nhận điều kiện của nó |
| **B11.4: etcd không lấy lại quorum sau khi bật hai node** | trên `lab-ha-1`: `sudo crictl ps --name etcd`; `sudo crictl logs <id>`; `timedatectl` trên cả ba control plane node | Kiểm đồng hồ trước — lệch giờ giữa các member làm etcd không bầu được leader. Sau đó kiểm cả ba member có thấy nhau trên `2380` không. Chờ theo vòng lặp của B11.4 thay vì đếm giây. Nếu sau nhiều vòng vẫn không lên, đây là lúc dùng đường lui: restore **cả sáu VM** về `8x-vm-ready`. **Không** thử `etcdctl member remove` hay khôi phục từ snapshot dữ liệu — đó là [nợ #8](README.md#5-sổ-nợ-lab), thuộc giai đoạn 19, và làm sai ở đây sẽ hỏng cluster nặng hơn |
| B12.3: sau khi bật lại LB, `kubectl` vẫn không vào được | trên `lab-ha-lb`: `systemctl is-active haproxy`; `systemctl is-enabled haproxy` | HAProxy chưa `enabled` nên không tự chạy sau boot. Bật lại bằng `sudo systemctl enable --now haproxy` và ghi vào evidence rằng B3.2 đã bị bỏ sót — vì mọi lần restore snapshot sau này sẽ vấp đúng lỗi này |
| B13.5: `listSnapshots` không thấy đúng tên mốc | `vmrun -T ws listSnapshots <vmx>` | Tên phải đúng nguyên văn `8x-vm-ready` và `8x-ha-stacked` trên **cả sáu** VM: không hậu tố theo máy, không thêm ngày, không đổi hoa thường. Chụp lại với tên đúng; snapshot thừa tên sai thì xoá bằng giao diện VMware |
| Bất kỳ bước nào: `kubectl` treo rất lâu rồi mới báo lỗi | `lbstat`; `direct_probe <nhan>` | Phân biệt ba nguyên nhân bằng đúng hai lệnh này: LB chết (mọi backend biến mất khỏi CSV, `direct_probe` vẫn 3), một apiserver chết (`lbstat` còn 2 dòng `=UP`), hay mất quorum (`lbstat` bình thường mà `api_probe` fail cả đọc lẫn ghi) |

---

## 5. Nguồn chính thức

Các mục giải thích trong thân bài ưu tiên snapshot tài liệu Kubernetes v1.35
(`https://v1-35.docs.kubernetes.io/`) để flag và hành vi khớp minor version của cluster. Tài liệu
HAProxy chỉ giải thích cú pháp của một phần mềm ngoài Kubernetes; nó không đổi quy trình hay gate
nào của lab.

- [Kubernetes v1.35 — Options for Highly Available Topology](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)
- [Kubernetes v1.35 — Creating Highly Available Clusters with kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [Kubernetes v1.35 — Installing kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Kubernetes v1.35 — Creating a cluster with kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Kubernetes v1.35 — Troubleshooting kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)
- [Kubernetes v1.35 — Administration with kubeadm](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/)
- [Kubernetes v1.35 — Adding Linux worker nodes](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/adding-linux-nodes/)
- [Kubernetes v1.35 — `kubeadm init`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)
- [Kubernetes v1.35 — `kubeadm join`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/)
- [Kubernetes v1.35 — `kubeadm certs`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-certs/)
- [Kubernetes v1.35 — Ports and Protocols](https://v1-35.docs.kubernetes.io/docs/reference/networking/ports-and-protocols/)
- [Kubernetes v1.35 — Static Pods](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
- [Kubernetes v1.35 — Kubernetes Components](https://v1-35.docs.kubernetes.io/docs/concepts/overview/components/)
- [kubeadm — HA considerations, options for software load balancing](https://git.k8s.io/kubeadm/docs/ha-considerations.md#options-for-software-load-balancing)
- [HAProxy — Configuration Manual](https://docs.haproxy.org/)
- [etcd — FAQ về quorum và số lượng member](https://etcd.io/docs/latest/faq/)
