# Lab 8b — HA với stacked etcd

> **Điểm bắt đầu:** **không phải snapshot nào của chuỗi chính**. Lab dựng mới một **bộ sáu VM
> riêng** — khai báo ở [mục 2.1](#21-bộ-vm-riêng-của-nhóm-lab-ha) — và không đụng vào ba VM
> `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2` của
> [chuỗi snapshot chính](README.md#3-chuỗi-snapshot). Ba VM đó phải **đang tắt** và còn mốc
> `03-storage-ready`. Repo `~/bookstore` mang sang bằng bundle `bookstore-lab-8a.bundle` (tag
> `lab-8a`) mà Lab 8a đã chép ra máy host.
> **Điểm kết thúc:** **tạo hai mốc** trên bộ VM riêng: `8x-vm-ready` (sáu VM đã cài OS, containerd
> và kubeadm, **chưa** `kubeadm init`) và `8x-ha-stacked` (cluster HA stacked, **sạch workload**).
> Tiền tố `8x-` là bắt buộc để không lẫn với chuỗi `01/02/03/04-` của bộ VM chính. Repo có tag
> `lab-8b`; bundle `bookstore-lab-8b.bundle` chép về máy host. Bộ VM chính vẫn ở
> `03-storage-ready` và không được bật lần nào trong lab.
> **Sprint dự án:** Sprint 13 — Kiểm chứng HA control plane (stacked etcd) của
> [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 8a — Dựng cluster bằng kubeadm](LAB-8A-DUNG-CLUSTER-BANG-KUBEADM.md); lab đó
> phá rồi dựng lại ba VM chính và kết thúc bằng restore `03-storage-ready`. Lab 8b chỉ mở ra **sau
> khi ba VM đó đã tắt**.
> **Lab sau:** [Lab 8c — HA với external etcd](LAB-8C-HA-VOI-EXTERNAL-ETCD.md), dùng lại **đúng bộ
> VM này** và đổi vai từng máy.
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng phần **HA** của
[Giai đoạn 8 — Dựng cluster bằng kubeadm](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm):
hai bài [06 — Các lựa chọn topology cho tính sẵn sàng cao](../06-ha-topology-vi.md) và
[08 — Tạo cluster có tính sẵn sàng cao với kubeadm](../08-high-availability-vi.md). Bài
[07 — Thiết lập cluster etcd HA](../07-setup-ha-etcd-with-kubeadm-vi.md) **không thuộc lab này**;
nó là nội dung của [Lab 8c](LAB-8C-HA-VOI-EXTERNAL-ETCD.md). Ba bài nền đã đọc trước đó —
[01](../01-install-kubeadm-vi.md), [02](../02-create-cluster-kubeadm-vi.md),
[09](../09-troubleshooting-kubeadm-vi.md) — và trang thực hành [214](../214-kubeadm-tasks-vi.md)
được dùng lại ở đây chứ không dạy lại.

Mọi ticket dựa vào hợp đồng kỹ thuật ở [mục 4 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật)
và bố cục repo ở [mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint). Kế hoạch gốc của sprint
nằm ở [mục 7.2](DU-AN-BOOKSTORE.md#sprint-13-14--lab-8b-8c-kiểm-chứng-ha); hành vi chịu lỗi mà sprint
này phải đo nằm ở [mục 3.7](DU-AN-BOOKSTORE.md#37-chịu-lỗi-và-vận-hành).

Cluster HA dùng đúng baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) — cùng Kubernetes, cùng
CNI. Image của dự án nằm ở [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00).
Lab không chép lại con số phiên bản nào trong phần văn bản.

[Quy trình mở đầu A5.5 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
**không áp dụng nguyên văn**: nó gate một cluster ba node đã tồn tại, còn ở đây cluster chưa có và số
node khác. B0 thay nó bằng các gate trên **máy host** — bộ VM chính đã tắt, còn mốc, máy host đủ tài
nguyên, bundle `lab-8a` có sẵn. Bản tương đương của gate ngắn cho cluster HA nằm ở
[B13.4](#b134-gate-ngắn-phiên-bản-ha), chạy cuối lab trước khi chụp `8x-ha-stacked`.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi hệ thống Bookstore chạy được trên một cluster HA ba control plane **chỉ từ repo**,
khách ngoài cluster đo được độ sẵn sàng của nó trong bốn sự cố control plane, **và** bạn chứng minh,
giải thích được:

- **Ranh giới** hai topology của bài [06](../06-ha-topology-vi.md): trong stacked, etcd member cục bộ
  chỉ nói chuyện với `kube-apiserver` của **chính node đó**; trong external, mỗi etcd host nói chuyện
  với `kube-apiserver` của **từng** control plane node. Số host tối thiểu của mỗi bên, và vì sao bộ
  sáu VM đủ cho stacked nhưng **không** đủ cho external nếu giữ nguyên vai từng máy.
- Cluster của bạn **là** stacked: etcd chạy dạng static Pod trên chính control plane node, số member
  bằng số control plane node — ba dữ kiện độc lập.
- Load balancer đứng trước ba `kube-apiserver`: chuyển tiếp **TCP**, health check TCP trên cổng
  apiserver, địa chỉ **khớp** `controlPlaneEndpoint`; đọc đúng `connection refused` và timeout trước
  khi `init`; và nó thật sự **phân phối** tới cả ba API server.
- `--upload-certs` để lại hiện vật gì, sống bao lâu, ai sở hữu, lệnh nào tạo lại; hai lệnh join khác
  nhau đúng **hai cờ**; vì sao `--control-plane-endpoint` là quyết định **một chiều**.
- Overlay `ha-lab` đưa đúng hệ thống của tag `lab-8a` lên một cluster thiếu hạ tầng của chuỗi chính —
  bỏ gì, đổi gì, vì sao không sửa `k8s/app`.
- **Bằng số đo từ máy host**: Bookstore phục vụ liên tục khi mất một control plane node, mất một
  `kube-apiserver`, mất quorum etcd, và mất load balancer — và vì sao mặt phẳng dữ liệu không phụ
  thuộc control plane để trả lời request.
- **Hỏng theo cặp** của stacked; ranh giới quorum ba trên ba, hai trên ba, một trên ba; vì sao "còn
  một apiserver đang chạy" không có nghĩa là "còn API". Cái mất khi mất control plane là **quyền điều
  khiển**, chưa phải dịch vụ — và vì sao đó vẫn là sự cố nghiêm trọng.
- Cái lab này **chưa HA**: một load balancer duy nhất là SPOF, chứng minh trên chính bộ VM của bạn.
- Kết thúc với cluster HA sạch workload, hai mốc `8x-vm-ready` và `8x-ha-stacked` trên đủ sáu VM, bộ
  VM chính còn nguyên `03-storage-ready`, và repo có tag `lab-8b` nằm ngoài VM.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm | Ticket kiểm chứng |
| --- | --- |
| [06 — Các lựa chọn topology cho tính sẵn sàng cao](../06-ha-topology-vi.md) | B2 — ranh giới hai topology, số host tối thiểu tính trên sáu VM, dự đoán bốn sự cố trước khi dựng; B6 — bằng chứng stacked: static Pod etcd trên đúng control plane node, số member bằng số control plane node; B9, B11 — hỏng theo cặp và ranh giới quorum, đo bằng tắt máy thật |
| [08 — Tạo cluster có tính sẵn sàng cao với kubeadm](../08-high-availability-vi.md) | B3 — load balancer chuyển tiếp TCP, health check TCP, `connection refused` so với timeout; B4 — `kubeadm init --control-plane-endpoint … --upload-certs`, Secret `kubeadm-certs` hai giờ, hình dạng certificate key, hai lệnh join, CNI trước khi join; B5 — join control plane từng node một, join worker, `rollout restart deployment coredns`, load balancer rải tới cả ba API server; B10 — một `kube-apiserver` chết mà cluster vẫn phục vụ; B12 — load balancer chính là SPOF |
| [01 — Cài đặt kubeadm](../01-install-kubeadm-vi.md) *(bài nền)* | B0 — yêu cầu tối thiểu cho máy control plane ở mục *Trước khi bạn bắt đầu*; hostname, MAC, `product_uuid` duy nhất trên sáu VM mới |
| [02 — Tạo một cluster với kubeadm](../02-create-cluster-kubeadm-vi.md) *(bài nền)* | B2 — mục *Khả năng chống chịu của cluster*: cluster một control plane có thể mất dữ liệu; B4 — `--apiserver-advertise-address` so với `--control-plane-endpoint` và quyết định một chiều; B5 — taint cách ly control plane vẫn còn trên cả ba node |
| [09 — Xử lý sự cố kubeadm](../09-troubleshooting-kubeadm-vi.md) *(bài nền)* | [Mục 4](#4-troubleshooting-của-lab-này) — dùng đúng công dụng của trang tra cứu: mỗi dòng sự cố kubeadm trỏ về đúng mục của bài |
| [214 — Quản trị với kubeadm](../214-kubeadm-tasks-vi.md) *(bài thực hành)* | B5 — thao tác *Thêm worker node Linux* của trang mục lục; các trang con còn lại thuộc giai đoạn 17–20 |

Các phần dưới đây **không kiểm chứng được trong lab này** — ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài 08 — **cặp load balancer + VIP** để chính load balancer cũng HA | Lab dùng **đúng một** VM làm load balancer, nên nó là SPOF. Bài 08 trỏ sang hướng dẫn *Các lựa chọn cho cân bằng tải bằng phần mềm* cho phần dự phòng; production cần **hai** máy LB cộng một **địa chỉ VIP** trôi giữa chúng. Lab không dựng vì cần thêm VM thứ bảy và một IP dành riêng cho VIP, vượt ngân sách tài nguyên của máy host. **B12 chứng minh SPOF này bằng thực nghiệm** thay vì giả vờ đã HA hoàn chỉnh |
| Bài 08 — **tên DNS** cho load balancer | Mạng lab không có DNS server nội bộ; lab dùng `/etc/hosts` như [A2.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a23-đặt-hostname-etchosts-và-kiểm-tra-identity), nên `controlPlaneEndpoint` vẫn là **tên** `lab-ha-lb:6443` chứ không phải IP trần — khác biệt còn lại chỉ là nguồn phân giải |
| Bài 08 — nhánh **external etcd** trọn vẹn, bài 07 | Nội dung của **Lab 8c**, chạy trên đúng bộ VM này. Bài 06 nói rõ hai topology là **hai lựa chọn**; làm cả hai trong một lab là làm nhầm việc. ADR topology của sprint này để trống phần external cho Lab 8c điền |
| Bài 08 — **Phân phối certificate thủ công** | Chỉ cần khi cố ý bỏ `--upload-certs`. Lab đi đường mặc định của bài; riêng `kubeadm certs certificate-key` vẫn chạy ở B4 để đọc hình dạng khóa vì lệnh đó không đụng vào cluster. Sao chép certificate thủ công thuộc [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ) |
| Bài 08 — cảnh báo **cloud provider**, Service `LoadBalancer`, PersistentVolume động | Lab chạy on-premises trên VM tự dựng; giới hạn về bản chất, không phải thiếu cấu hình |
| Bài 08 — **Các container image** khi host không pull được image | Sáu VM đều có HTTPS egress, đã gate ở B0. Kịch bản air-gapped cần registry nội bộ |
| Bài 06 — **backup và khôi phục** etcd của cluster HA | **[Nợ lab #8](README.md#5-sổ-nợ-lab)**, phát sinh ở giai đoạn 8 và **chưa trả ở lab này**. Không `etcdctl snapshot save`, không quy trình khôi phục; đường lui duy nhất vẫn là snapshot VM. Trả ở [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa) |
| **Nâng cấp** một cluster HA | Bài 08 trỏ sang [tài liệu nâng cấp](../221-kubeadm-upgrade-vi.md); thuộc [giai đoạn 17](../00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster) |
| **Ghim version cho HAProxy** | HAProxy không nằm trong baseline; bài 08 nói ví dụ của nó "chỉ là một lựa chọn". Lab cài từ gói của bản phân phối, **không ghim version**, ghi version quan sát được vào evidence ở B3 |
| **Thực thi NetworkPolicy**, Ingress, StorageClass trên cluster HA | Cluster HA chỉ có CNI của baseline, và CNI đó **không thực thi NetworkPolicy**; Traefik, local-path-provisioner là hạ tầng của chuỗi chính (Lab 5b, 6a). Cài lại chúng ở đây là việc của một lab khác; overlay `ha-lab` ở B7 bỏ những object phụ thuộc vào chúng và ghi lý do |

Nợ còn mở sau lab — nói được vì sao ở checkpoint:

- **Nợ lab #8** (backup và khôi phục etcd) — **chưa trả**. Ba bản etcd đồng bộ không thay được bản
  sao lưu: xóa nhầm được nhân bản y hệt sang cả ba.
- **Nợ kỹ thuật:** sprint này **không phát sinh và không trả** mã `TD` nào của hệ thống. Overlay
  `ha-lab` cố ý chạy dữ liệu trên `emptyDir` — đó là lựa chọn của **môi trường đo**, không phải lùi
  `TD-04`, `TD-08` của hệ thống; chuỗi chính vẫn dùng PVC như tag `lab-8a`.

### 1.2. Thời lượng

**B0 là phiên làm việc riêng**, không tính vào thời lượng lab: dựng sáu VM có khối lượng tương
đương A1–A4 của [Lab 00](LAB-00-MOI-TRUONG-1.35.7.md), khoảng 2–4 giờ tùy mạng và tùy bạn cài từng
máy hay dùng full clone. Buổi một kết thúc khi có mốc `8x-vm-ready`.

Phần còn lại khoảng **4–5 giờ**, nên chia hai phiên:

- Phiên 2: B1–B6 — mở bộ VM, quyết định topology, dựng cluster HA, chứng minh stacked. Khoảng 2 giờ.
- Phiên 3: B7–B13 — đưa Bookstore lên, đo nền, bốn diễn tập, đóng sprint. Khoảng 2,5 giờ. Mục ⭐ ở
  B11 là tùy chọn.

Giữa hai phiên được phép tắt cả sáu VM theo [mục 2.3](#23-trình-tự-an-toàn-khi-tắt-và-bật-máy);
mọi biến và hàm nằm trong `~/lab-work/8b/bien.sh` nên không phụ thuộc phiên shell. Gate tiếp tục đầu
phiên 3, trên `lab-ha-1`: chạy lại Script nghiệm thu của B6 — chỉ đọc, nó tự `source` file biến — và
`git -C ~/bookstore status --porcelain` phải rỗng. Gate ngắn A5.5 và smoke là của cluster chính, không
áp dụng cho bộ VM HA; Bookstore chỉ lên cluster này ở B7.

Mọi bước phải chờ đều viết dưới dạng `kubectl wait --timeout` hoặc **vòng lặp có điều kiện dừng và
số vòng tối đa**. Lab không cam kết con số giây nào cho việc hội tụ: thời gian phụ thuộc cấu hình
cluster, cấu hình load balancer, tốc độ đĩa và số VM chạy song song trên máy host.

---

## 2. Quy ước và an toàn

### 2.1. Bộ VM riêng của nhóm lab HA

Hai lab HA của giai đoạn 8 — **Lab 8b và Lab 8c** — dùng chung **một bộ sáu VM riêng, tách hoàn
toàn** khỏi ba VM của chuỗi snapshot chính. Lab 8b là nơi **định nghĩa** bộ VM đó; Lab 8c link về
đúng mục này thay vì khai lại.

| Vai trò trong Lab 8b | Tên VM / hostname | IP | vCPU | RAM | Disk thin-provisioned |
| --- | --- | --- | --- | --- | --- |
| Load balancer | `lab-ha-lb` | `192.168.100.230` | 1 | 1 GB | 20 GB |
| Control plane 1 — nơi `kubeadm init`, giữ repo `~/bookstore` | `lab-ha-1` | `192.168.100.231` | 2 | 3 GB | 40 GB |
| Control plane 2 | `lab-ha-2` | `192.168.100.232` | 2 | 3 GB | 40 GB |
| Control plane 3 | `lab-ha-3` | `192.168.100.233` | 2 | 3 GB | 40 GB |
| Worker 1 | `lab-ha-4` | `192.168.100.234` | 2 | 2 GB | 40 GB |
| Worker 2 / fault target | `lab-ha-5` | `192.168.100.235` | 2 | 2 GB | 40 GB |

**Tổng cấp phát: 11 vCPU, 14 GB RAM, 220 GB đĩa danh nghĩa.** Đĩa thin-provisioned nên dung lượng
chiếm thật nhỏ hơn nhiều; gate ở B0 kiểm theo dung lượng còn trống thật.

Năm máy `lab-ha-1` … `lab-ha-5` cấu hình **giống hệt nhau về phần mềm**: cùng OS, cùng containerd,
cùng `kubeadm`/`kubelet`/`kubectl`, chỉ khác hostname và IP. `lab-ha-lb` **không phải node
Kubernetes**: nó chỉ cần OS và HAProxy, không cài containerd, không cài kubeadm, không tắt swap.

Mức RAM của ba control plane node cao hơn **mức tối thiểu** mà bài
[01 — Cài đặt kubeadm](../01-install-kubeadm-vi.md#before-you-begin) đặt ra: **2 GB RAM trở lên cho
mỗi máy** và **2 CPU trở lên cho các máy control plane**. Bài nói dưới 2 GB thì "còn rất ít dung
lượng cho các ứng dụng của bạn"; ở đây mỗi control plane node còn gánh thêm **một etcd member**, nên
lab cấp 3 GB. Host có từ 32 GB RAM trở lên thì nâng ba máy này lên 4 GB; **không hạ xuống dưới
2 GB**. Hai worker 2 GB đủ cho Bookstore: tổng `requests` của dự án theo
[mục 3.6 của file dự án](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch) dưới 1 GiB.

**Vì sao tên máy trung tính.** Tên `lab-ha-N` **cố ý không mang vai trò**. Lab 8c dựng topology
external etcd trên **đúng sáu VM này**: `lab-ha-1`, `lab-ha-2`, `lab-ha-3` đổi vai thành **node
etcd**, còn `lab-ha-4`, `lab-ha-5` đổi vai thành **control plane**. Đặt tên theo vai thì Lab 8c hoặc
phải đổi tên VM — kéo theo hostname, `/etc/hosts`, tên Node — hoặc phải sống với tên nói dối. Đừng
đổi tên trong Lab 8b vì bất kỳ lý do gì.

Ba nguyên tắc bắt buộc của bộ VM này:

- **Không đụng vào ba VM của chuỗi chính.** Không bật, không xóa, không đổi tên, không sửa snapshot
  của `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`. Repo đi qua **bundle trên máy host**,
  không đi qua mạng tới bộ VM chính.
- **Snapshot của bộ VM HA luôn mang tiền tố `8x-`.** Lab 8b tạo `8x-vm-ready` và `8x-ha-stacked`;
  Lab 8c tạo `8x-ha-external`. Tên phải đúng nguyên văn trên **cả sáu VM**, không hậu tố theo máy.
- **Dải IP `.230`–`.235` phải nằm ngoài DHCP pool** của router, đúng như
  [A3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a3-đặt-ip-tĩnh-và-phân-giải-tên) yêu cầu với dải
  `.221`–`.223`. LAN của bạn đã dùng dải này thì đổi cả sáu IP cùng lúc và sửa mọi chỗ trong lab,
  kể cả `bien.sh` (B1) và hai tham số `-AppUrl`, `-ApiUrl` của script đo (B8).

### 2.2. Quy ước chạy lệnh, bằng chứng và repo

- **Node mặc định để gõ lệnh là `lab-ha-1`**, bằng user quản trị có kubeconfig. Khối nào chạy ở nơi
  khác đều mở đầu bằng dòng "Chạy trên `<tên máy>`" hoặc "Chạy trên máy host". `lab-ha-1` là máy
  duy nhất **không bao giờ bị tắt** trong lab.
- **Repo dự án là `~/bookstore` trên `lab-ha-1`.** Bằng chứng: `~/lab-evidence/8b/` trên `lab-ha-1`.
  Manifest tạm, lệnh join, file render: `~/lab-work/8b/`. File đo của máy host (`do-<Ten>.csv`,
  định dạng `ms,app,api` như Lab 8c) nằm ở `$HOME\lab-evidence\8b\` và được chép sang
  `~/lab-evidence/8b/` sau mỗi lần đo.
- **Đầu mỗi khối lệnh trên `lab-ha-1`: `source ~/lab-work/8b/bien.sh`.** File này (B1) chứa mọi
  biến và hàm dùng chung, nên đổi phiên shell hay tắt máy giữa chừng không làm mất trạng thái.
- **Ba script của chuỗi chính — `scripts/deploy.sh`, `scripts/teardown.sh`, `tests/smoke.sh` — không
  chạy trên cluster HA.** Chúng giả định hạ tầng của chuỗi chính: Traefik, StorageClass, tên node
  `lab-k8s-*`, label node riêng. Trên cluster HA dùng bộ `-ha-lab` của B7 và B13. Sprint này **không
  sửa** ba script đó: Lab 9a mở đầu bằng chúng trên `03-storage-ready`.
- **Lệnh cần quyền root trên máy khác thì gõ trực tiếp trên máy đó**, không bọc qua
  `ssh <node> 'sudo …'`. Ubuntu cài từ ISO mặc định đòi mật khẩu cho `sudo`, và `sudo` không có tty
  sẽ hỏng giữa chừng. Lab chỉ dùng `ssh` cho lệnh **đọc** không cần root.
- **Fault injection:** bốn diễn tập B9–B12 cố ý tắt control plane node và load balancer — đó là nội
  dung bài học. Ở tầng worker, chỉ `lab-ha-5` được phá (mục ⭐ của B11); `lab-ha-4` luôn lành.
- **Không cài hạ tầng của chuỗi chính lên cluster HA** — không Calico, không Traefik, không
  local-path-provisioner, không metrics-server. Cluster HA chỉ có CNI của baseline.
- Namespace `lab-8b` chỉ chứa ConfigMap mà hàm `api_probe` tạo để thử **ghi** vào API; nó không
  thuộc dự án, không mang label dự án, và bị xóa ở B13.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

### 2.3. Trình tự an toàn khi tắt và bật máy

Lab tắt và bật máy nhiều lần: B9 (một control plane node), B10 (không tắt máy, chỉ gỡ một static
Pod), B11 (hai control plane node, ⭐ thêm `lab-ha-5`), B12 (load balancer), B13 (cả sáu máy), cộng
lần tắt tùy chọn giữa phiên 2 và phiên 3. Ba quy tắc không được vi phạm:

1. **`lab-ha-1` không bao giờ bị tắt trong lúc làm lab.** Nó giữ repo, kubeconfig và `bien.sh`.
2. **Tắt máy bằng `sudo shutdown -h now` gõ trên chính máy đó**, rồi xác nhận *Powered off* trên máy
   host bằng `vmrun` trước khi bật lại. Chỉ dùng `vmrun … stop … hard` khi
   [mục 4](#4-troubleshooting-của-lab-này) chỉ dẫn, và khi đó **ghi vào postmortem** rằng lần tắt này
   là cưỡng bức, vì nó đổi bản chất phép thử.
3. **Bật lại bằng `vmrun start` hoặc giao diện VMware, rồi chờ bằng vòng lặp có điều kiện dừng**,
   không đếm giây.

Thứ tự bật cả bộ: `lab-ha-lb` → `lab-ha-1` → `lab-ha-2` → `lab-ha-3` → `lab-ha-4` → `lab-ha-5`.
Thứ tự tắt là **ngược lại**. Load balancer lên trước vì `controlPlaneEndpoint` trỏ vào nó; các node
lên trước LB sẽ có một khoảng không kết nối được endpoint, vô hại nhưng làm log ồn.

**Đường quay lui.** Sau khi có `8x-vm-ready`, mọi sự cố hạ tầng không sửa được trong vài phút đều
xử lý bằng cách tắt **cả sáu VM** và restore **cả sáu** về `8x-vm-ready`, rồi chạy lại từ B1. Không
bao giờ restore riêng một VM: cluster HA có ba bản etcd đang đồng bộ, restore lệch một máy tạo ra một
member mang dữ liệu quá khứ — tình huống không có trong bài nào của giai đoạn 8. Nguyên tắc này là
bản mở rộng của ghi chú cuối [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).

Restore về `8x-vm-ready` **xóa luôn `~/bookstore` trên `lab-ha-1`** — mốc đó có trước khi repo được
clone. Vì vậy trước diễn tập đầu tiên, B8 bắt buộc đóng gói phần việc đang làm thành
`bookstore-lab-8b-wip.bundle` và chép ra máy host. Restore xong thì clone từ bundle đó thay cho
bundle `lab-8a`.

---

# Phần B — Sprint 13: Kiểm chứng HA control plane (stacked etcd)

## B0. Nhận bàn giao sprint: bộ VM HA và mốc `8x-vm-ready`

**Mục đích:** chứng minh bộ VM chính đã tắt và còn đường lui, máy host đủ tài nguyên, repo của sprint
12 nằm ngoài VM; rồi dựng sáu máy đúng [mục 2.1](#21-bộ-vm-riêng-của-nhóm-lab-ha), **chưa**
`kubeadm init`, và chụp mốc quay lui `8x-vm-ready`. Đây là **phiên làm việc riêng**, không có ticket:
toàn bộ là thao tác hạ tầng đã kiểm chứng. Lab không lặp lại phần dựng môi trường — mở phần A của
[Lab 00](LAB-00-MOI-TRUONG-1.35.7.md) đọc song song; B0 chỉ ghi **những chỗ khác**.

### B0.1. Gate trên máy host

Biến của máy host. Chạy trên **máy host**, PowerShell — **chạy lại nguyên khối này mỗi khi mở cửa sổ
PowerShell mới** (B1, B8–B13 dùng lại). Sửa đường dẫn `.vmx` và `$haUser` cho đúng máy của bạn:

```powershell
$vmrun   = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$mainVmx = @(
  'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
  'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
  'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
)
$haVmx = @(
  'E:\Virtual Machines\lab-ha-lb\lab-ha-lb.vmx'
  'E:\Virtual Machines\lab-ha-1\lab-ha-1.vmx'
  'E:\Virtual Machines\lab-ha-2\lab-ha-2.vmx'
  'E:\Virtual Machines\lab-ha-3\lab-ha-3.vmx'
  'E:\Virtual Machines\lab-ha-4\lab-ha-4.vmx'
  'E:\Virtual Machines\lab-ha-5\lab-ha-5.vmx'
)
$backup = "$HOME\lab-backup"            # noi Lab 3a -> 8a da chep bundle ra
$ev     = "$HOME\lab-evidence\8b"       # file do tu may host (B8 -> B12)
$probe  = "$backup\do-san-sang.ps1"     # script do, chep tu repo o B8
$haUser = '<user-quan-tri-tren-bo-VM-HA>'
New-Item -ItemType Directory -Force -Path $ev | Out-Null

# Mo mot cua so PowerShell moi chay script do; cua so giu lai de doc dong KET QUA.
function Start-Probe([string]$Ten, [int]$Giay) {
  $a = '-NoExit -NoProfile -ExecutionPolicy Bypass -File "{0}" -Ten {1} -Giay {2}' -f $probe, $Ten, $Giay
  Start-Process powershell -ArgumentList $a
}
# Chep file do do-<Ten>.csv sang lab-ha-1 de script nghiem thu doc.
function Send-Probe([string]$Ten) {
  scp "$ev\do-$Ten.csv" "$($haUser)@192.168.100.231:lab-evidence/8b/"
}

if (-not (Test-Path $vmrun)) { 'FAIL: khong tim thay vmrun.exe — sua duong dan' }
if ($haUser -match '^<') { 'FAIL: chua dien $haUser' } else { 'PASS: da dien $haUser' }
```

`-ExecutionPolicy Bypass` chỉ áp cho tiến trình con chạy script đo; lab **không** đổi chính sách thực
thi của máy host. Ba gate — không VM nào chạy, bộ VM chính tắt và còn mốc, bundle `lab-8a` có sẵn:

```powershell
$running = & $vmrun -T ws list | Select-Object -Skip 1 | Where-Object { $_ -match '\.vmx$' }
if ($running.Count -eq 0) { 'PASS: khong VM nao dang chay' } else { "FAIL: dang chay $($running -join ', ')" }
foreach ($f in $mainVmx) {
  if (-not (Test-Path $f)) { "FAIL: khong tim thay $f — KHONG duoc xoa VM chuoi chinh"; continue }
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '03-storage-ready') { "PASS: $f — con moc 03-storage-ready" } else { "FAIL: $f -> $($names -join ', ')" }
}
$b = Join-Path $backup 'bookstore-lab-8a.bundle'
if ((Test-Path $b) -and ((Get-Item $b).Length -gt 0)) { "PASS: co $b" } else { "FAIL: thieu $b — muc 5.4 cua file du an" }
```

Gate tài nguyên — tổng cấp phát của bảng mục 2.1 so với máy host thật:

```powershell
$vmRamGb = 14; $diskNeedGb = 100      # 14 GB cho sau VM; 100 GB trong cho dia thin + hai moc snapshot
$cpu  = Get-CimInstance Win32_Processor
$lp   = ($cpu | Measure-Object -Property NumberOfLogicalProcessors -Sum).Sum
$virt = ($cpu | Select-Object -First 1).VirtualizationFirmwareEnabled
$ramGb = [math]::Round((Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory / 1GB, 1)
$freeGb = [math]::Round((Get-Volume -DriveLetter 'E').SizeRemaining / 1GB, 1)   # doi 'E' cho dung o dia
"host: $lp logical CPU | $ramGb GB RAM | $freeGb GB trong | virt=$virt"
if ($virt -eq $true -and $lp -ge 8 -and $ramGb -ge ($vmRamGb + 6) -and $freeGb -ge $diskNeedGb) {
  'PASS: host du tai nguyen cho bo VM HA'
} else { 'FAIL: thieu virtualization, CPU (>= 8), RAM (>= 20 GB) hoac dia trong (>= 100 GB)' }
```

**PASS:** `PASS: da dien $haUser`; `PASS: khong VM nao dang chay`; ba dòng `PASS:` cho ba VM chính;
`PASS: co …bookstore-lab-8a.bundle`; `PASS: host du tai nguyen …`; không dòng `FAIL:` nào. Ba điều
kiện của bộ VM chính khác nhau: file `.vmx` **còn** (không ai xóa VM), VM **tắt**, mốc
`03-storage-ready` **còn** — đường về của chuỗi chính. 11 vCPU trên 8 logical CPU là over-subscribe
có chủ ý vì VM phần lớn rỗi; host dưới 8 logical CPU thì hạ hai worker xuống 1 vCPU, **không** hạ
control plane. Không đủ tài nguyên thì **dừng**: cluster hai control plane **không** phải HA theo bài
[06](../06-ha-topology-vi.md), và B9–B11 mất ý nghĩa.

### B0.2. Sáu VM, hostname, IP tĩnh

Làm theo [A1.2](LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm),
[A2](LAB-00-MOI-TRUONG-1.35.7.md#a2-cài-ubuntu-và-cấu-hình-identity) và
[A3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a3-đặt-ip-tĩnh-và-phân-giải-tên), với bốn chỗ khác:

| Điểm | Lab 00 | Lab 8b |
| --- | --- | --- |
| Số VM | 3 | **6** |
| Tên máy | `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2` | `lab-ha-lb`, `lab-ha-1` … `lab-ha-5` |
| Dải IP | `192.168.100.221`–`.223` | `.230` cho `lab-ha-lb`, `.231`–`.235` cho `lab-ha-1`–`lab-ha-5` |
| Spec từng máy | bảng A1.2 | bảng [mục 2.1](#21-bộ-vm-riêng-của-nhóm-lab-ha) — LB nhỏ hơn hẳn |

ISO và checksum lấy đúng ở [A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). Full clone từ
một VM nguồn nhanh hơn cài sáu lần, nhưng bắt buộc chạy
[A2.2](LAB-00-MOI-TRUONG-1.35.7.md#a22-vm-được-tạo-bằng-full-clone) trên **từng clone**: bài
[01](../01-install-kubeadm-vi.md#verify-mac-address) nói Kubernetes dùng hostname, MAC và
`product_uuid` để định danh duy nhất node. Đặt hostname đúng một lệnh trên mỗi VM
(`sudo hostnamectl set-hostname lab-ha-lb`, …, `lab-ha-5`), rồi thêm giống nhau trên **cả sáu** VM —
kể cả LB, vì B3 kiểm kết nối từ LB:

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

**PASS:** VMware hiển thị đủ sáu VM đúng spec; trên cả sáu VM, khối verify cuối A3 đạt — có dòng
`PASS: 50-cloud-init.yaml is absent`, IP giữ nguyên sau reboot, ping được gateway và Internet, DNS
phân giải được.

### B0.3. Phần mềm: năm node giống Lab 00, LB thì không

Trên `lab-ha-1` … `lab-ha-5` chạy nguyên [A4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a4-chuẩn-bị-os-và-container-runtime)
— [A4.1](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites) đến
[A4.4](LAB-00-MOI-TRUONG-1.35.7.md#a44-firewall-cho-mạng-lab-cô-lập), đủ các gate. `lab-ha-lb`
**không phải node Kubernetes**: không containerd, không kubeadm, không tắt swap. Trên nó chỉ chạy:

```bash
# Chay tren lab-ha-lb
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y ca-certificates curl
sudo timedatectl set-ntp true
sudo ufw disable
```

**PASS:** trên năm node, mọi gate của A4.1–A4.4 đạt — swap rỗng, `containerd` và `kubelet` `enabled`,
`crictl version` trả `RuntimeApiVersion: v1`, ba binary Kubernetes đúng bảng A1.3. Trên `lab-ha-lb`,
`ufw` `inactive`, `curl` chạy được, `timedatectl` báo `System clock synchronized: yes`. Đồng hồ lệch
làm hỏng certificate và làm etcd bầu leader sai; đừng bỏ NTP trên máy nào.

### B0.4. SSH không mật khẩu, identity, liên thông

Tạo khóa trên `lab-ha-1`, phát cho cả sáu máy — kể cả chính nó, để vòng lặp không có ngoại lệ. Lệnh
hỏi mật khẩu một lần cho mỗi máy:

```bash
# Chay tren lab-ha-1
test -f ~/.ssh/id_ed25519 || ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
for n in lab-ha-lb lab-ha-1 lab-ha-2 lab-ha-3 lab-ha-4 lab-ha-5; do
  ssh-copy-id -o StrictHostKeyChecking=accept-new "$n"
done
```

`product_uuid` chỉ root đọc được, nên chạy khối này **trên từng máy trong sáu máy**:

```bash
# Chay tren TUNG may trong sau may
sudo cat /sys/class/dmi/id/product_uuid > /tmp/lab8b-uuid.txt
IFACE="$(ip route | awk '/default/ {print $5; exit}')"
printf '%s %s %s %s\n' "$(hostnamectl --static)" "$(cat /etc/machine-id)" \
  "$(tr -d '\n' < /tmp/lab8b-uuid.txt)" "$(ip -br link show "$IFACE" | awk '{print $3}')" \
  > /tmp/lab8b-identity.txt
```

Gom lại và gate trên `lab-ha-1`:

```bash
# Chay tren lab-ha-1
HOSTS='lab-ha-lb lab-ha-1 lab-ha-2 lab-ha-3 lab-ha-4 lab-ha-5'
OK=0
for n in $HOSTS; do
  H="$(ssh -o BatchMode=yes -o ConnectTimeout=5 "$n" hostnamectl --static 2>/dev/null)"
  test "$H" = "$n" && OK=$(( OK + 1 ))
done
test "$OK" -eq 6 && echo 'PASS: SSH khong mat khau toi ca sau may, hostname dung'

for n in $HOSTS; do ssh -o BatchMode=yes "$n" cat /tmp/lab8b-identity.txt; done | tee /tmp/lab8b-identity-all.txt
UNIQ_OK=0
for col in 1 2 3 4; do
  N="$(awk -v c="$col" '{print $c}' /tmp/lab8b-identity-all.txt | sort -u | wc -l)"
  if test "$N" -eq 6; then UNIQ_OK=$(( UNIQ_OK + 1 )); else echo "FAIL: cot $col bi trung giua cac may"; fi
done
test "$UNIQ_OK" -eq 4 && echo 'PASS: hostname, machine-id, product_uuid va MAC deu duy nhat tren sau may'

FAILN=0
for src in $HOSTS; do
  for dst in $HOSTS; do
    ssh -o BatchMode=yes "$src" "ping -c 1 -W 2 $dst >/dev/null 2>&1" || { echo "FAIL: $src -> $dst"; FAILN=$(( FAILN + 1 )); }
  done
done
test "$FAILN" -eq 0 && echo 'PASS: 36 cap nguon-dich deu ping duoc'
```

**PASS:** ba dòng `PASS:` xuất hiện, không dòng `FAIL:` nào. Bài
[08](../08-high-availability-vi.md#stacked-etcd) đặt "kết nối mạng đầy đủ giữa tất cả các máy" thành
điều kiện tiên quyết, và với HA thì "tất cả" gồm cả load balancer. Trùng `product_uuid` hoặc MAC thì
**dừng** và để VMware sinh giá trị mới — không sửa được từ bên trong Ubuntu. Cặp nào không ping được
thì sửa `/etc/hosts`, IP hoặc firewall trước khi đi tiếp.

### B0.5. Tắt sáu VM và chụp `8x-vm-ready`

Mốc quay lui của cả Lab 8b **và** Lab 8c. Chụp khi VM đã tắt, đúng lý do
[A5.4.8 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a548-dọn-resource-test-ghi-evidence-và-chụp-snapshot).
Tắt theo thứ tự `lab-ha-5` → … → `lab-ha-1` → `lab-ha-lb` bằng `sudo shutdown -h now` trên từng máy.
Khi cả sáu *Powered off*, chụp trên từng VM với tên đúng nguyên văn `8x-vm-ready`, mô tả
`bo VM HA cua Lab 8b, chua kubeadm init, chup <ngay>`. Verify trên **máy host**:

```powershell
foreach ($f in $haVmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '8x-vm-ready') { "PASS: $f" } else { "FAIL: $f -> $($names -join ', ')" }
}
```

**PASS:** đúng sáu dòng `PASS:`. `-ccontains` phân biệt hoa thường nên bắt được cả lỗi gõ sai tên.
Buổi làm việc thứ nhất kết thúc ở đây.

## B1. [BS-8B-01] Mở bộ VM HA và mang repo sang

**Loại:** Vận hành.

**Bối cảnh:** dự án sống trong repo, không sống trong cluster ([mục 5.3](DU-AN-BOOKSTORE.md#53-quy-tắc-làm-việc-với-repo)).
Sprint này chạy trên một môi trường hoàn toàn mới — sáu VM chưa từng thấy repo — và bộ VM chính đang
tắt. Repo phải sang đây bằng đúng đường mà [mục 5.4](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm)
đã chuẩn bị: bundle nằm trên máy host. Mọi ticket sau còn cần một bộ biến và hàm dùng chung sống sót
qua việc đổi phiên shell và tắt máy.

**Yêu cầu:**

1. Sáu VM chạy, bật đúng thứ tự của [mục 2.3](#23-trình-tự-an-toàn-khi-tắt-và-bật-máy).
2. `~/lab-work/8b/bien.sh` trên `lab-ha-1` — chép nguyên văn khối bên dưới, rồi **tự điền** ba giá
   trị lấy từ [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). Phiên bản Kubernetes
   bạn điền phải khớp `kubeadm` đã cài.
3. Năm node Kubernetes đạt điều kiện đầu vào: không swap, `containerd` và `kubelet` `enabled`, cùng
   version `kubeadm`; **chưa** node nào từng `init` hay `join`.
4. `~/bookstore` trên `lab-ha-1` là clone của bundle `lab-8a`, HEAD đúng tag `lab-8a`, không có thay
   đổi chưa commit; `git` có danh tính để commit; máy có `git`, `openssl`, `curl`.

**Ràng buộc:** không bật VM nào của chuỗi chính để lấy repo. Không sửa repo trong ticket này. Không
chép con số phiên bản vào lab hay vào repo — `bien.sh` nằm ở `~/lab-work`, không vào git.

**Đọc lại:** [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm) — bundle, và vì
sao bundle không mang `secrets/`; [A5.5 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
— để thấy vì sao gate đó không áp nguyên văn ở đây.

**Bàn giao:** `~/lab-work/8b/bien.sh`; `~/bookstore` ở tag `lab-8a`; `~/lab-evidence/8b/b1-*.txt`.

**Nghiệm thu:**

- `PASS:` cả sáu VM đang chạy.
- `PASS:` ba giá trị trong `bien.sh` đúng hình dạng; `K8S_VER` khớp `kubeadm version` đã cài.
- `PASS:` không node nào bật swap; `containerd`, `kubelet` `enabled` trên cả năm node; cả năm cùng
  version với bảng A1.3; năm node sạch dấu vết cluster cũ.
- `PASS:` repo ở đúng tag `lab-8a`, sạch, có danh tính commit; `secrets/db.env` **không** có trong
  clone.

Biến và hàm dùng chung, chép nguyên văn:

```bash
# Chay tren lab-ha-1
mkdir -p ~/lab-work/8b ~/lab-evidence/8b
cat > ~/lab-work/8b/bien.sh <<'EOF'
# Lab 8b — biến và hàm dùng chung. Nạp ở đầu mỗi khối lệnh: source ~/lab-work/8b/bien.sh
LB='lab-ha-lb';  LB_IP='192.168.100.230'
CP1='lab-ha-1';  CP1_IP='192.168.100.231'
CP2='lab-ha-2';  CP2_IP='192.168.100.232'
CP3='lab-ha-3';  CP3_IP='192.168.100.233'
WN1='lab-ha-4';  WN1_IP='192.168.100.234'
WN2='lab-ha-5';  WN2_IP='192.168.100.235'
CPS="$CP1 $CP2 $CP3"
NODES="$CP1 $CP2 $CP3 $WN1 $WN2"
ENDPOINT="$LB:6443"
ETCD_EPS="https://$CP1_IP:2379,https://$CP2_IP:2379,https://$CP3_IP:2379"
EV="$HOME/lab-evidence/8b"; WK="$HOME/lab-work/8b"; REPO="$HOME/bookstore"

# Ba giá trị lấy từ bảng A1.3 của Lab 00 — tự điền.
K8S_VER=''       # dòng "Kubernetes control plane", dạng vX.Y.Z
POD_CIDR=''      # dòng "Pod CIDR"
FLANNEL_VER=''   # dòng "CNI", phần release của manifest Flannel, dạng vX.Y.Z

# Trạng thái từng backend của HAProxy (CSV: cột 1 proxy, cột 2 server, cột 18 trạng thái).
lbstat() {
  curl -s --max-time 10 "http://${LB_IP}:8404/stats;csv" \
    | awk -F, '$1=="k8s-controlplane" && $2!="BACKEND" {print $2"="$18}'
}
# Tổng số phiên HAProxy đã chuyển cho từng backend (cột 8, stot).
lbstot() {
  curl -s --max-time 10 "http://${LB_IP}:8404/stats;csv" \
    | awk -F, '$1=="k8s-controlplane" && $2!="BACKEND" {print $2"="$8}'
}
# etcdctl chạy trong static Pod etcd của lab-ha-1.
etcd1() {
  kubectl -n kube-system exec "etcd-$CP1" -- etcdctl \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key "$@"
}
# Đọc và GHI API qua endpoint chung. Ghi = tạo một ConfigMap trong namespace lab-8b (không thuộc dự án).
api_probe() {
  API_READ='fail'; API_WRITE='fail'
  kubectl get nodes --request-timeout=20s >/dev/null 2>&1 && API_READ='ok'
  kubectl -n lab-8b create configmap "probe-$1-$(date +%s)" --from-literal=t=1 \
    --request-timeout=20s >/dev/null 2>&1 && API_WRITE='ok'
  echo "  doc API=$API_READ | ghi API=$API_WRITE"
}
# Gõ thẳng từng kube-apiserver, bỏ qua load balancer. Mã 000 = không kết nối được.
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
# Chờ load balancer thấy đúng $1 backend UP, tối đa 60 vòng; kết quả ở biến UPN.
wait_lb_up() {
  for i in $(seq 1 60); do UPN="$(lbstat | grep -c '=UP$')"; test "$UPN" -eq "$1" && return 0; sleep 5; done
  return 1
}
# Số etcd member khỏe; chi tiết ghi vào $EV/$1.
etcd_ok() { etcd1 --endpoints="$ETCD_EPS" endpoint health > "$EV/$1" 2>&1; grep -c 'is healthy' "$EV/$1"; }
# Tóm tắt file đo "ms,app,api" của máy host (scripts/do-san-sang.ps1) — cùng cách tính với Lab 8c.
# Đặt MAU; APP, API (% sẵn sàng tính theo THỜI GIAN); APP_MAX, API_MAX (giây gián đoạn liên tục dài
# nhất); CUOI_APP, CUOI_API (1 nếu 10 mẫu cuối đều 200). In một dòng và ghi thêm vào $EV/so-do.txt.
tom_tat() {
  MAU=0; APP=0; API=0; APP_MAX=0; API_MAX=0; CUOI_APP=0; CUOI_API=0
  test -s "$1" || { echo "FAIL: chua co $1 — chep file do tu may host sang"; return 1; }
  eval "$(awk -F, '
    BEGIN { n = 0 }
    { sub(/\r$/, "") }
    NR > 1 && $1 ~ /^[0-9]+$/ { t[n] = $1; a[n] = $2; p[n] = $3; n++ }
    END {
      if (n < 2) { print "MAU=0 APP=0 API=0 APP_MAX=0 API_MAX=0 CUOI_APP=0 CUOI_API=0"; exit }
      tot = t[n-1] - t[0]; ad = 0; pd = 0; ar = 0; pr = 0; am = 0; pm = 0
      for (i = 0; i < n - 1; i++) {
        dt = t[i+1] - t[i]
        if (a[i] != "200") { ad += dt; ar += dt; if (ar > am) am = ar } else ar = 0
        if (p[i] != "200") { pd += dt; pr += dt; if (pr > pm) pm = pr } else pr = 0
      }
      ca = 1; cp = 1
      for (i = (n > 10 ? n - 10 : 0); i < n; i++) { if (a[i] != "200") ca = 0; if (p[i] != "200") cp = 0 }
      printf "MAU=%d APP=%.1f API=%.1f APP_MAX=%d API_MAX=%d CUOI_APP=%d CUOI_API=%d\n",
        n, 100 * (1 - ad / tot), 100 * (1 - pd / tot), am / 1000, pm / 1000, ca, cp
    }' "$1")"
  echo "$(basename "$1"): mau=$MAU app=$APP% api=$API% | gian doan dai nhat: app=${APP_MAX}s api=${API_MAX}s" \
    | tee -a "$EV/so-do.txt"
}
ge() { awk -v a="$1" -v b="$2" 'BEGIN { exit !(a >= b) }'; }
lt() { awk -v a="$1" -v b="$2" 'BEGIN { exit !(a < b) }'; }
# Chờ cluster lành hẳn sau diễn tập $1: đủ node Ready, 3 member khỏe, 3 backend UP, 3 apiserver.
cho_lanh() {
  local h=0
  kubectl wait --for=condition=Ready node --all --timeout=600s >/dev/null
  for i in $(seq 1 60); do h="$(etcd_ok "$1-etcd-sau.txt")"; test "$h" -eq 3 && break; sleep 5; done
  wait_lb_up 3; direct_probe "$1-sau" >/dev/null
  test "$h" -eq 3 && echo 'PASS: du ba etcd member khoe manh tro lai'
  test "$UPN" -eq 3 && echo 'PASS: du ba backend UP tro lai'
  test "$DIRECT_OK" -eq 3 && echo 'PASS: du ba kube-apiserver tra loi tro lai'
}
# Postmortem có mục $1, và dòng $1 trong ADR topology đủ sáu ô (ba dự đoán, ba số đo).
ghi_chep() {
  grep -q "^## $1" "$REPO/docs/postmortem/8b-mat-control-plane.md" 2>/dev/null && echo "PASS: postmortem co muc $1"
  grep -Eq "^\| *$1 *(\| *(duoc|khong) *){6}\|" "$REPO"/docs/adr/*-topology-ha.md && echo "PASS: ADR da dien so do cho $1"
}
EOF
```

`api_probe` đo **hai** việc khác nhau — đọc và ghi không cùng điều kiện, mấu chốt của bốn diễn tập.
`direct_probe` bỏ qua load balancer; mọi mã HTTP khác `000`, kể cả `401`, chứng minh có một apiserver
trả lời. Các hàm đọc load balancer chỉ có dữ liệu sau B3; `tom_tat` dùng từ B8 — nó tính đúng như
hàm cùng tên của [Lab 8c](LAB-8C-HA-VOI-EXTERNAL-ETCD.md), để số đo hai sprint đặt cạnh nhau được.

**Câu hỏi thiết kế** — trả lời ở mục *Bối cảnh* của ADR topology mà B2 tạo:

1. Vì sao repo sang bằng bundle trên máy host, không `git clone` qua mạng từ `lab-k8s-master`?
2. Bundle không mang `secrets/`. Trên cluster HA mới, điều đó dẫn tới gì khi triển khai, và vì sao
   ở đây nó **không** gây ra ngoại lệ mà mục 5.4 cảnh báo?
3. Vì sao gate A5.5 của Lab 00 không áp nguyên văn cho lab này, và những gate nào thay nó?

<details>
<summary>Gợi ý</summary>

1. Bật máy: dùng lại `$haVmx` của B0.1 — mảng đã xếp đúng thứ tự LB trước, worker sau.
2. Sửa ba giá trị trong `bien.sh` bằng trình soạn thảo, hoặc `sed -i "s/^K8S_VER=''/K8S_VER='...'/"`.
3. Chép bundle từ máy host sang `lab-ha-1` bằng `scp`, rồi `git clone <file bundle> ~/bookstore`.
4. Danh tính commit: `git config --global user.name`, `git config --global user.email` — đặt giống
   trên `lab-k8s-master` để lịch sử repo nhất quán.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

Chạy trên **máy host** — bật sáu VM đúng thứ tự (bỏ qua nếu đã bật bằng giao diện):

```powershell
foreach ($f in $haVmx) {
  & $vmrun -T ws start $f
  Start-Sleep -Seconds 10
}
$running = & $vmrun -T ws list
$up = ($haVmx | Where-Object { $running -contains $_ }).Count
if ($up -eq 6) { "PASS: ca sau VM dang chay" } else { "FAIL: moi co $up/6 VM chay" }
```

Chạy trên `lab-ha-1`:

```bash
source ~/lab-work/8b/bien.sh

KV="$(kubeadm version -o short)"
echo "kubeadm tren may: $KV | bien.sh: $K8S_VER" | tee "$EV/b1-bien.txt"
test -n "$K8S_VER" && test "$KV" = "$K8S_VER" && echo 'PASS: K8S_VER khop kubeadm da cai' \
  || echo 'FAIL: sai gia tri hoac sai package — quay lai A4.3 cua Lab 00'
echo "$POD_CIDR"    | grep -Eq '^[0-9]+(\.[0-9]+){3}/[0-9]+$' && echo 'PASS: POD_CIDR dung dang CIDR'
echo "$FLANNEL_VER" | grep -Eq '^v[0-9]+\.[0-9]+\.[0-9]+$'    && echo 'PASS: FLANNEL_VER dung dang'

for n in $NODES; do
  L="$(ssh -o BatchMode=yes "$n" 'swapon --show | wc -l; \
       systemctl is-active containerd kubelet | tr "\n" " "; \
       systemctl is-enabled containerd kubelet | tr "\n" " "; \
       kubeadm version -o short' 2>/dev/null | tr '\n' ' ')"
  echo "$n : $L"
done | tee "$EV/b1-node-prereq.txt"
SWAP_BAD="$(awk '{print $3}' "$EV/b1-node-prereq.txt" | grep -vc '^0$')"
ENA_OK="$(grep -c 'enabled enabled' "$EV/b1-node-prereq.txt")"
VER_OK="$(grep -c -- "$K8S_VER" "$EV/b1-node-prereq.txt")"
test "$SWAP_BAD" -eq 0 && echo 'PASS: khong node nao con bat swap'
test "$ENA_OK"   -eq 5 && echo 'PASS: containerd va kubelet enabled tren ca nam node'
test "$VER_OK"   -eq 5 && echo 'PASS: ca nam node cung version kubeadm voi bang A1.3'

CONF=0
for n in $NODES; do
  ssh -o BatchMode=yes "$n" 'test -e /etc/kubernetes/admin.conf || test -d /var/lib/etcd' \
    && { echo "FAIL: $n da co dau vet cluster cu"; CONF=$(( CONF + 1 )); }
done
test "$CONF" -eq 0 && echo 'PASS: nam node deu sach, chua tung init hay join'

for T in git openssl curl; do command -v "$T" >/dev/null && echo "PASS: co $T" || echo "FAIL: thieu $T"; done
cd "$REPO" || exit 1
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-8a' && echo 'PASS: HEAD dung tag lab-8a'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
{ test -n "$(git config user.name)" && test -n "$(git config user.email)"; } && echo 'PASS: git co danh tinh commit'
test ! -e secrets/db.env && echo 'PASS: bundle khong mang secrets/db.env'
git log --oneline -3 > "$EV/b1-repo.txt"
```

`kubelet` có thể đang crashloop trên cả năm node — trạng thái **dự kiến**, đúng ghi chú cuối
[A4.3](LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl): chưa có
`/var/lib/kubelet/config.yaml` thì kubelet chưa có việc. Gate kiểm `enabled`, không kiểm `active`.
Thiếu `git` hoặc `openssl` thì **dừng** — cài gói là quyết định của bạn, lab không tự cài.

</details>

<details>
<summary>Lời giải tham khảo</summary>

```powershell
# Chay tren may host
scp "$backup\bookstore-lab-8a.bundle" "$($haUser)@192.168.100.231:"
```

```bash
# Chay tren lab-ha-1
git clone ~/bookstore-lab-8a.bundle ~/bookstore
git -C ~/bookstore describe --tags --exact-match HEAD
git config --global user.name  '<giong tren lab-k8s-master>'
git config --global user.email '<giong tren lab-k8s-master>'
nano ~/lab-work/8b/bien.sh     # dien K8S_VER, POD_CIDR, FLANNEL_VER tu bang A1.3
```

Ý chính cho ADR:

1. Bộ VM chính phải tắt suốt lab — bật nó là vi phạm gate của B0 và tranh RAM với sáu VM. Máy host
   là nơi duy nhất nằm ngoài **cả hai** bộ VM, nên bundle trên host là cầu nối đúng; nó cũng là nơi
   bundle `lab-8b` quay về ở B13.
2. `deploy-ha-lab.sh` (B7) sinh `secrets/db.env` mới, và DB trên cluster HA khởi tạo từ thư mục dữ
   liệu rỗng với mật khẩu mới. Ngoại lệ ở mục 5.4 chỉ xảy ra khi **dữ liệu được giữ lại** mà mật khẩu
   đổi; ở đây không có dữ liệu cũ nào sang cùng repo.
3. A5.5 gate một cluster ba node đang chạy; ở đầu lab này chưa có cluster và số node khác. Thay vào
   đó: các gate máy host của B0, gate đầu vào của B1, và [gate ngắn phiên bản HA ở B13.4](#b134-gate-ngắn-phiên-bản-ha).

</details>

## B2. [BS-8B-02] Chọn topology và dự đoán trước khi dựng

**Loại:** Điều tra.

**Bối cảnh:** yêu cầu R12 của dự án ([mục 2.2](DU-AN-BOOKSTORE.md#22-yêu-cầu-của-hệ-thống-mới)) là
*đánh giá được topology HA*. Lộ trình đánh dấu bài 06 là **quyết định trước khi dựng HA**. Đội phải
ghi lựa chọn và **dự đoán** hệ thống sẽ phản ứng thế nào với bốn sự cố, trước khi có cluster để thử.
Dự đoán sai không phải lỗi — nó là phần giá trị nhất của postmortem ở B9–B12.

**Yêu cầu:**

1. ADR `docs/adr/NNNN-topology-ha.md` (`NNNN` là số tiếp theo trong `docs/adr/`), trạng thái
   *đề xuất*, có các mục:
   - `## Bối cảnh` — gồm trả lời câu hỏi thiết kế của B1.
   - `## Hai topology` — ranh giới: trong mỗi topology, etcd member nói chuyện với `kube-apiserver`
     nào; số host tối thiểu của mỗi bên **không tính worker**, ghi đúng dạng `stacked=<số>` và
     `external=<số>`; bộ sáu VM đủ cho bên nào nếu giữ nguyên vai từng máy.
   - `## Dự đoán và đo` — bảng Markdown, mỗi kịch bản một dòng, cột đầu là mã kịch bản, ba cột sau
     là dự đoán **API đọc**, **API ghi**, **Bookstore qua NodePort**, mỗi ô `duoc` hoặc `khong`. Bốn
     mã: `mat-mot-cp` (tắt một control plane node), `mat-manifest-apiserver` (một `kube-apiserver`
     mất manifest), `mat-quorum` (tắt hai trong ba control plane node), `mat-lb` (tắt load balancer).
     Để chỗ cho ba cột *đo được* điền ở B9–B12.
   - `## Stacked (Lab 8b)` — để trống, điền dần ở B3–B12.
   - `## External (Lab 8c)` — để trống, Lab 8c điền.
2. Commit ADR **ngay trong ticket này**, trước khi chạy `kubeadm init` — lịch sử git là bằng chứng
   dự đoán có trước số đo.

**Ràng buộc:** chưa có cluster; ticket này không chạy lệnh nào lên node. Không sửa dự đoán sau khi đã
đo — ghi kết quả đo vào cột riêng, giải thích chênh lệch trong postmortem.

**Đọc lại:**

- Bài [06](../06-ha-topology-vi.md): [*Topology etcd xếp chồng*](../06-ha-topology-vi.md#stacked-etcd-topology),
  [*Topology etcd bên ngoài*](../06-ha-topology-vi.md#external-etcd-topology).
- Bài [08](../08-high-availability-vi.md), mục [*Stacked etcd*](../08-high-availability-vi.md#stacked-etcd)
  của *Trước khi bạn bắt đầu* — câu về số lẻ node control plane.
- Bài [22](../22-architecture-vi.md#nhieu-ban-apiserver), *Nhiều bản kube-apiserver sau một load
  balancer* — ba mô hình HA khác nhau của apiserver, etcd, controller.
- Bài [02](../02-create-cluster-kubeadm-vi.md#resilience), *Khả năng chống chịu của cluster*.
- [Mục 3.7 của file dự án](DU-AN-BOOKSTORE.md#37-chịu-lỗi-và-vận-hành), dòng *Mất một control plane
  (HA)*.

**Bàn giao:** `docs/adr/NNNN-topology-ha.md` đã commit; `~/lab-evidence/8b/b2-*.txt`.

**Nghiệm thu:**

- `PASS:` có ADR topology với đủ năm mục.
- `PASS:` có số host tối thiểu dạng `stacked=<số>`, `external=<số>` đúng theo bài 06.
- `PASS:` bảng dự đoán có đủ bốn mã kịch bản, mỗi dòng đủ ba ô `duoc`/`khong`.
- `PASS:` ADR đã có trong lịch sử git; chưa node nào có cluster.

**Câu hỏi thiết kế** — trả lời trong ADR:

1. Trong stacked, etcd member trên `lab-ha-2` phục vụ những `kube-apiserver` nào? Trong external thì
   sao? Vì sao "ba member hợp thành một etcd cluster" không có nghĩa là apiserver nào cũng nói chuyện
   với member nào?
2. Bộ sáu VM đủ cho topology nào? Lab 8c dựng external trên đúng sáu máy này sẽ phải đánh đổi gì?
3. Ba control plane node stacked chịu được mất bao nhiêu node mà API còn **ghi** được? Vì sao bài 08
   khuyên số **lẻ** — bốn node có hơn ba node không?
4. Cluster chuỗi chính của bạn có `--control-plane-endpoint` không, trỏ vào đâu? Muốn biến nó thành
   HA thì thiếu gì? Vì sao bài 02 gọi việc thiếu cờ này là quyết định một chiều?
5. Vì sao bạn dự đoán Bookstore **vẫn** (hoặc **không**) phục vụ được khi mất quorum?

<details>
<summary>Gợi ý</summary>

1. Bài 06 nêu số host tối thiểu bằng chữ trong cả hai mục; đếm control plane và etcd, chưa tính
   worker.
2. Bài 22 có bảng ba mô hình HA: mô hình của etcd quyết định câu hỏi 3. "Đa số" của ba là bao nhiêu,
   của bốn là bao nhiêu?
3. Với câu hỏi 5: một request từ trình duyệt tới `/api/books` đi qua những thành phần nào? Thành phần
   nào trong số đó là control plane?
4. `git add docs/adr/ && git commit -m 'lab-8b: ADR topology HA — du doan truoc khi dung'`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
source ~/lab-work/8b/bien.sh
cd "$REPO"
ADR="$(ls docs/adr/*-topology-ha.md 2>/dev/null | head -n 1)"
test -n "$ADR" && echo "PASS: co ADR $ADR" || echo 'FAIL: chua co docs/adr/NNNN-topology-ha.md'

BAD=0
for H in '## Bối cảnh' '## Hai topology' '## Dự đoán và đo' '## Stacked' '## External'; do
  grep -q "^$H" "$ADR" || { echo "FAIL: thieu muc $H"; BAD=1; }
done
test "$BAD" -eq 0 && echo 'PASS: ADR du nam muc'

{ grep -q 'stacked=3' "$ADR" && grep -q 'external=6' "$ADR"; } \
  && echo 'PASS: so host toi thieu dung bai 06 (khong tinh worker)'

BAD=0
for S in mat-mot-cp mat-manifest-apiserver mat-quorum mat-lb; do
  grep -Eq "^\| *$S *(\| *(duoc|khong) *){3}\|" "$ADR" || { echo "FAIL: thieu du doan $S"; BAD=1; }
done
test "$BAD" -eq 0 && echo 'PASS: du du doan cho bon kich ban'
grep -E '^\| *mat-' "$ADR" | tee "$EV/b2-du-doan.txt"

git log --oneline -- "$ADR" | grep -q . && echo 'PASS: ADR da nam trong lich su git'
CONF=0
for n in $NODES; do ssh -o BatchMode=yes "$n" 'test -e /etc/kubernetes/admin.conf' && CONF=$(( CONF + 1 )); done
test "$CONF" -eq 0 && echo 'PASS: du doan duoc ghi truoc khi co cluster'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Khung ADR — phần nội dung viết bằng lời của bạn:

```markdown
# NNNN — Topology HA cho control plane

- Trạng thái: đề xuất
- Sprint: Lab 8b (BS-8B-01 → BS-8B-12); phần external: Lab 8c

## Bối cảnh

R12. Cluster chuỗi chính có một control plane: mất nó là mất API, và theo bài 02 có thể mất dữ
liệu. Môi trường thử: bộ VM lab-ha-*, repo mang sang bằng bundle lab-8a (trả lời câu hỏi B1).

## Hai topology

| | Stacked | External |
| --- | --- | --- |
| etcd member nói chuyện với | apiserver của chính node đó | apiserver của từng control plane node |
| Số host tối thiểu (không tính worker) | stacked=3 | external=6 (3 control plane + 3 etcd) |
| Mất một máy | mất 1 apiserver VÀ 1 etcd member (hỏng theo cặp) | chỉ mất ở một tầng |

Bộ sáu VM: 1 LB + 5 máy Kubernetes. Đủ cho stacked 3 + 2 worker; thiếu cho external nếu giữ vai.

## Dự đoán và đo

| Kịch bản | API đọc | API ghi | NodePort | API đọc (đo) | API ghi (đo) | NodePort (đo) |
| --- | --- | --- | --- | --- | --- | --- |
| mat-mot-cp | duoc | duoc | duoc | | | |
| mat-manifest-apiserver | duoc | duoc | duoc | | | |
| mat-quorum | khong | khong | duoc | | | |
| mat-lb | khong | khong | duoc | | | |

## Stacked (Lab 8b)

(điền ở B3 → B12)

## External (Lab 8c)

(Lab 8c điền)

## Rủi ro đã biết

- Một load balancer duy nhất là SPOF — B12 đo.
```

Ý chính:

1. Bài 06: trong stacked, member cục bộ **chỉ** giao tiếp với `kube-apiserver` của chính node đó;
   trong external, **mỗi** etcd host giao tiếp với apiserver của **từng** control plane node. "Chung
   một etcd cluster" là quan hệ giữa các member; "apiserver ↔ member" là quan hệ khác — nhầm hai thứ
   là chỗ trượt kinh điển của bài.
2. Năm máy Kubernetes đủ `3 + 2` của stacked và thiếu một máy so với `3 + 3` của external — chưa kể
   worker. Lab 8c dùng ba máy làm etcd và chỉ còn **hai** control plane, không worker: dưới mức tối
   thiểu của bài 06, phải ghi rõ đánh đổi đó.
3. Bài 22: etcd giữ nhất quán bằng quorum Raft — mỗi lần ghi cần **đa số** member. Ba member: đa số
   là hai, chịu mất một. Bốn member: đa số là ba, vẫn chỉ chịu mất một mà tốn thêm một máy — vì thế
   số lẻ (bài 08: "có số lẻ node control plane có thể giúp ích cho việc bầu chọn leader").
4. [A5.1 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane) có truyền
   `--control-plane-endpoint` nhưng trỏ vào chính `lab-k8s-master`. Thiếu: hai control plane node
   nữa và một endpoint (load balancer) đứng trước cả ba. Bài 02: chuyển một cluster tạo **không có**
   cờ này thành HA không được kubeadm hỗ trợ — tên endpoint đã nằm trong certificate và mọi
   kubeconfig.
5. Request của khách đi node → kube-proxy → Pod `web` → Service `api` → Pod `api` → `db-0`; không
   chặng nào là `kube-apiserver` hay etcd. Dự đoán hợp lý: vẫn phục vụ, **trong lúc không có gì cần
   thay đổi**.

</details>

## B3. [BS-8B-03] Load balancer trước ba API server

**Loại:** Vận hành.

**Bối cảnh:** bài 08 đặt việc dựng load balancer là **bước đầu tiên và chung cho cả hai phương
pháp**. Mọi client — `kubectl` của quản trị viên, kubelet, node đang join — sẽ được dạy gọi **một**
địa chỉ duy nhất; địa chỉ đó phải chuyển tới API server còn sống. Endpoint này phải tồn tại **trước**
`kubeadm init`, vì tên của nó được ghi vào certificate và mọi kubeconfig.

**Yêu cầu:**

1. `lab-ha-lb` chạy load balancer lắng nghe `6443`, chuyển tiếp **TCP** tới `6443` của ba control
   plane node, chỉ chuyển tới backend **khỏe** theo health check TCP.
2. Có trang thống kê dạng CSV ở cổng `8404` để `lbstat`, `lbstot` đọc trạng thái backend.
3. Tự khởi động lại sau reboot; không cắt các kết nối watch dài hạn của `kubectl` và kubelet.
4. **Trước** `kubeadm init`: chứng minh LB tới được cả ba control plane node trên `6443` — kết quả
   phải là *refused*, không phải *timeout* — và năm node tới được `6443` của LB. Ba backend đang
   `DOWN`.
5. Version phần mềm LB quan sát được ghi vào evidence; không ghim.
6. Hạ tầng nằm trong repo: `platform/ha-lab/haproxy-k8s.cfg` là đúng đoạn cấu hình đã thêm vào LB,
   `platform/ha-lab/README.md` ghi cách dựng bộ VM HA (B4, B5 bổ sung).

**Ràng buộc:** HAProxy cài từ gói của bản phân phối. Không giải mã TLS ở LB. Khối cấu hình bên dưới
là **công cụ đã kiểm chứng — chép nguyên văn**; nội dung học của ticket là đối chiếu từng dòng của nó
với yêu cầu của bài 08 và đọc đúng kết quả phép thử kết nối.

**Đọc lại:**

- Bài [08](../08-high-availability-vi.md), mục
  [*Tạo bộ cân bằng tải cho kube-apiserver*](../08-high-availability-vi.md#tạo-bộ-cân-bằng-tải-cho-kube-apiserver-create-load-balancer-for-kube-apiserver)
  — bốn yêu cầu, và đoạn về `connection refused` với timeout.
- Bài [01](../01-install-kubeadm-vi.md#check-required-ports), *Kiểm tra các port cần thiết* — cách
  thử một cổng bằng `nc`.
- Bài [22](../22-architecture-vi.md#nhieu-ban-apiserver) — vì sao nhiều apiserver cùng nhận request
  được.

**Bàn giao:** `platform/ha-lab/haproxy-k8s.cfg`, `platform/ha-lab/README.md`;
`~/lab-evidence/8b/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` HAProxy `active` và `enabled`, lắng nghe `6443` và `8404`; version đã ghi vào evidence.
- `PASS:` từ LB tới ba control plane node: ba lần *refused*, không *timeout*.
- `PASS:` năm node chạm được `6443` của LB mà không timeout.
- `PASS:` ba backend `DOWN` vì chưa có apiserver nào.
- `PASS:` đoạn cấu hình trong repo khớp đúng đoạn đang chạy trên LB.

Cài và cấu hình, chép nguyên văn:

```bash
# Chay tren lab-ha-lb
sudo apt-get update
sudo apt-get install -y haproxy
haproxy -v
dpkg-query -W -f='${Package} ${Version} ${Architecture}\n' haproxy

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
`enabled`; `ss` thấy tiến trình lắng nghe **cả** `6443` **và** `8404`. Chưa `enabled` thì sau mỗi lần
restore snapshot LB không lên và cả cluster mất endpoint.

**Câu hỏi thiết kế** — trả lời trong ADR topology, mục `## Stacked (Lab 8b)`, tiểu mục *Load
balancer*:

1. Bài 08 đặt bốn yêu cầu cho load balancer. Mỗi yêu cầu ứng với dòng nào trong đoạn cấu hình?
2. Trước `init`, LB gõ `6443` của một control plane node và nhận *refused*. Đi tiếp hay dừng? Còn
   *timeout*? Vì sao chiều ngược lại — node gõ `6443` của LB — **không** ra *refused*, và đó có phải
   lỗi không?
3. Backend khai bằng **IP**, còn endpoint mà client dùng lại là **tên** `lab-ha-lb`. Vì sao mỗi chỗ
   chọn một kiểu?
4. Hai dòng `timeout … 1h` không nằm trong bốn yêu cầu của bài 08. Để mặc định của gói thì triệu
   chứng gì xuất hiện, ở ai?
5. Ghi vào mục *Rủi ro đã biết*: cấu hình này còn một điểm hỏng đơn nào?

<details>
<summary>Gợi ý</summary>

- Phép thử một cổng TCP không cần cài `nc`: `timeout 3 bash -c "echo > /dev/tcp/<ip>/6443"`, đo thời
  gian bằng `date +%s` trước và sau. Chạy được ba giây mới thoát là timeout.
- Trang thống kê: `curl -s "http://192.168.100.230:8404/stats;csv" | head -n 3`. Dòng đầu là header
  bắt đầu bằng `# pxname` — đếm cột từ đó.
- Đoạn cấu hình đang chạy, đọc không cần root:
  `ssh lab-ha-lb "sed -n '/^#--- Lab 8b/,\$p' /etc/haproxy/haproxy.cfg"`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

Chạy trên `lab-ha-lb` — phép thử từ LB tới ba control plane node:

```bash
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

**PASS:** cả ba dòng là `refused`, **không** dòng nào `TIMEOUT`.

Chạy trên `lab-ha-1`:

```bash
source ~/lab-work/8b/bien.sh
{
  echo "=== $(date -Is) — load balancer cua Lab 8b ==="
  ssh -o BatchMode=yes "$LB" 'haproxy -v'
  ssh -o BatchMode=yes "$LB" 'dpkg-query -W -f="${Package} ${Version} ${Architecture}\n" haproxy'
  ssh -o BatchMode=yes "$LB" 'systemctl is-active haproxy; systemctl is-enabled haproxy'
} | tee "$EV/b3-haproxy-version.txt"
grep -q 'HAProxy' "$EV/b3-haproxy-version.txt" && echo 'PASS: da ghi version HAProxy quan sat duoc'
test "$(grep -c '^active$\|^enabled$' "$EV/b3-haproxy-version.txt")" -eq 2 \
  && echo 'PASS: haproxy vua active vua enabled'

for n in $NODES; do
  R="$(ssh -o BatchMode=yes "$n" \
      "S=\$(date +%s); timeout 3 bash -c 'echo > /dev/tcp/${LB_IP}/6443' >/dev/null 2>&1; \
       echo elapsed=\$(( \$(date +%s) - S ))")"
  echo "$n -> $R"
done | tee "$EV/b3-node-to-lb.txt"
test "$(grep -c 'elapsed=[012]$' "$EV/b3-node-to-lb.txt")" -eq 5 \
  && echo 'PASS: ca nam node cham duoc :6443 cua LB ma khong timeout'

# HAProxy coi server la UP cho toi khi du so lan health check that bai: cho bang vong lap.
DOWNN=0
for i in $(seq 1 30); do
  DOWNN="$(lbstat | grep -c '=DOWN$')"
  test "$DOWNN" -eq 3 && break
  sleep 5
done
lbstat | tee "$EV/b3-backend-truoc-init.txt"
test "$DOWNN" -eq 3 && echo 'PASS: ca ba backend dang DOWN vi chua co apiserver nao chay'

cd "$REPO"
ssh -o BatchMode=yes "$LB" "sed -n '/^#--- Lab 8b/,\$p' /etc/haproxy/haproxy.cfg" > "$WK/lb-dang-chay.cfg"
diff <(grep -v '^$' "$WK/lb-dang-chay.cfg") <(grep -v '^$' platform/ha-lab/haproxy-k8s.cfg) >/dev/null \
  && echo 'PASS: doan cau hinh trong repo khop LB dang chay'
grep -q 'mode tcp' platform/ha-lab/haproxy-k8s.cfg && test -s platform/ha-lab/README.md \
  && echo 'PASS: platform/ha-lab co cau hinh LB va README'
```

Chiều node → LB không ra *refused*: HAProxy **đã** lắng nghe `6443`, nên nó chấp nhận kết nối rồi đóng
ngay vì chưa backend nào `UP`. Điều được gate là **không timeout** — đường từ node tới cổng lắng nghe
của LB thông.

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
# Chay tren lab-ha-1
mkdir -p ~/bookstore/platform/ha-lab
ssh lab-ha-lb "sed -n '/^#--- Lab 8b/,\$p' /etc/haproxy/haproxy.cfg" \
  > ~/bookstore/platform/ha-lab/haproxy-k8s.cfg
```

```markdown
# platform/ha-lab/README.md — Bộ VM HA của Lab 8b, 8c

Không phải hạ tầng của chuỗi chính. Sáu VM riêng, mốc `8x-vm-ready` (sạch) và `8x-ha-stacked`.

| Máy | IP | Vai ở Lab 8b |
| --- | --- | --- |
| lab-ha-lb | 192.168.100.230 | HAProxy, endpoint `lab-ha-lb:6443` |
| lab-ha-1, lab-ha-2, lab-ha-3 | .231–.233 | control plane + etcd xếp chồng |
| lab-ha-4, lab-ha-5 | .234–.235 | worker; lab-ha-5 là fault target |

## Load balancer

- Gói `haproxy` của Ubuntu, không ghim version (version quan sát ghi trong evidence của Lab 8b).
- Thêm [haproxy-k8s.cfg](haproxy-k8s.cfg) vào cuối `/etc/haproxy/haproxy.cfg`, rồi
  `haproxy -c -f …` và `systemctl enable --now haproxy`.
- Thống kê: `http://192.168.100.230:8404/stats` (thêm `;csv` để đọc bằng script).
- Trước `kubeadm init`: từ LB tới `:6443` của control plane node phải là *refused*, không *timeout*.
```

Ý chính cho ADR:

1. *Chuyển tiếp TCP* → `mode tcp` ở frontend và backend (TLS đi xuyên qua, LB không cần
   certificate). *Health check TCP trên cổng apiserver* → `option tcp-check` + `check` trên từng
   `server`. *Giao tiếp được với mọi control plane node, cho phép lưu lượng vào cổng lắng nghe* →
   ba dòng `server …:6443` và `bind *:6443`. *Khớp `ControlPlaneEndpoint`* → địa chỉ
   `lab-ha-lb:6443` sẽ là giá trị của `--control-plane-endpoint` ở B4.
2. *Refused* thì đi tiếp: gói tin đã tới node, chỉ chưa có ai lắng nghe. *Timeout* thì dừng: LB không
   giao tiếp được với node — sửa mạng trước, chưa `init`. Chiều node → LB: HAProxy đang lắng nghe nên
   chấp nhận rồi đóng; khác hành vi của LB cloud trong bài, không phải lỗi.
3. HAProxy phân giải tên lúc khởi động; LB phụ thuộc DNS thì hỏng DNS là hỏng endpoint. Ngược lại,
   client dùng **tên** để sau này đổi IP của LB (hoặc thay bằng VIP) mà không phải sinh lại
   certificate và kubeconfig.
4. Khối `defaults` của gói đặt timeout vài chục giây, hợp với request HTTP ngắn. `kubectl … -w` và
   kubelet giữ kết nối **watch** im lặng rất lâu; LB sẽ cắt chúng — watch đứt liên tục. Đây là tham
   số của LB, không phải cam kết về thời gian hội tụ của Kubernetes.
5. Chính `lab-ha-lb`: một máy, không ai dự phòng.

</details>

## B4. [BS-8B-04] Control plane đầu tiên sau load balancer

**Loại:** Vận hành.

**Bối cảnh:** lệnh `kubeadm init` đầu tiên chốt một quyết định **một chiều**: endpoint mà mọi client
và mọi node sau này sẽ dùng. Nó cũng phải để lại đủ thứ để hai control plane node sau join mà không
ai phải chép certificate bằng tay. Cluster chưa có Pod network thì chưa chạy được gì — kể cả CoreDNS.

**Yêu cầu:**

1. `lab-ha-1` là control plane đầu tiên. Endpoint **dùng chung** của cluster là load balancer, ghi
   bằng tên `lab-ha-lb:6443`; địa chỉ quảng bá **riêng** của apiserver trên node này là IP của nó.
2. Certificate dùng chung của control plane được đưa lên cluster để control plane node sau lấy về khi
   join.
3. Phiên bản Kubernetes khóa đúng giá trị trong `bien.sh` (không để kubeadm tự dò); Pod CIDR đúng
   giá trị của baseline.
4. `kubectl` của user thường trên `lab-ha-1` dùng kubeconfig quản trị, và request của nó đi **qua
   load balancer**.
5. CNI của baseline cài **ngay sau init, trước khi join** node nào; Pod CIDR của CNI khớp init;
   `lab-ha-1` `Ready`.
6. Log đầy đủ của `init` nằm ở `~/lab-work/8b/` (chứa token và khóa, bị xóa ở B13); evidence chỉ
   giữ bản **đã che** token và khóa.
7. Ghi lại cho B5: hai lệnh join mà `init` in ra; hồ sơ Secret mà `--upload-certs` để lại và hạn
   của nó; số node đang chứa Pod CoreDNS lúc cluster mới có một node.
8. `platform/ha-lab/README.md` thêm mục *Control plane đầu tiên* với lệnh `init` đã dùng — dùng tên
   biến, **không** chứa token, hash hay khóa.

**Ràng buộc:** không dùng `--config` ở lab này — giữ đúng dạng cờ của bài 08. Không dán token hay
certificate key vào repo, evidence, chat hay ảnh chụp màn hình. Khóa hết hạn thì **tải lên lại**,
không `kubeadm reset` để `init` lại.

**Đọc lại:**

- Bài [08](../08-high-availability-vi.md), mục
  [*Các bước cho node control plane đầu tiên*](../08-high-availability-vi.md#các-bước-cho-node-control-plane-đầu-tiên-steps-for-the-first-control-plane-node)
  — ba cờ, Secret `kubeadm-certs`, hạn hai giờ, `kubeadm init phase upload-certs`, hình dạng
  certificate key, và ghi chú "triển khai CNI trước khi chuyển sang bước tiếp theo".
- Bài [02](../02-create-cluster-kubeadm-vi.md), mục
  [*Những lưu ý về apiserver-advertise-address và ControlPlaneEndpoint*](../02-create-cluster-kubeadm-vi.md#những-lưu-ý-về-apiserver-advertise-address-và-controlplaneendpoint-considerations-about-apiserver-advertise-address-and-controlplaneendpoint)
  và [*Cài đặt Pod network add-on*](../02-create-cluster-kubeadm-vi.md#pod-network).
- [A5.1](LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane) và
  [A5.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a52-cài-flannel-v0289) — lệnh init và CNI của
  cluster chuỗi chính, để so cờ nào khác.

**Bàn giao:** `platform/ha-lab/README.md` cập nhật; `~/lab-evidence/8b/b4-*.txt`,
`~/lab-evidence/8b/b4-init-da-che.log`; `~/lab-work/8b/b4-init.log`, `~/lab-work/8b/join-*.txt`.

**Nghiệm thu:**

- `PASS:` `kubeadm init` thành công; evidence không chứa token hay chuỗi hex 64 ký tự.
- `PASS:` kubeconfig, ConfigMap `cluster-info` và `kubeadm-config` cùng ghi endpoint là load balancer.
- `PASS:` `init` in đúng hai lệnh join; bỏ hai cờ khỏi lệnh control plane thì được đúng lệnh worker;
  certificate key là hex 64 ký tự; cả hai lệnh trỏ vào load balancer.
- `PASS:` Secret `kubeadm-certs` tồn tại, có `ownerReference` tới một bootstrap token, vòng đời
  khoảng hai giờ.
- `PASS:` Pod CIDR của CNI khớp `init`; image CNI đúng release; `lab-ha-1` `Ready`.
- `PASS:` toàn bộ Pod CoreDNS đang nằm trên `lab-ha-1`.

**Câu hỏi thiết kế** — trả lời trong ADR topology, mục `## Stacked (Lab 8b)`, tiểu mục *Khởi tạo*:

1. Trong các cờ bạn truyền cho `init`, cờ nào làm cluster này **có thể** thành HA? So với A5.1 của
   Lab 00, khác ở cờ nào và ở **giá trị** nào? Vì sao bài 02 gọi đây là quyết định một chiều?
2. `--apiserver-advertise-address` khác `--control-plane-endpoint` thế nào? Giá trị nào giống nhau
   trên ba control plane node, giá trị nào khác?
3. Endpoint được ghi ở **ba** nơi trong cluster. Mỗi nơi phục vụ ai? Lệch một nơi thì ai bị chỉ
   sai đường?
4. `--upload-certs` để lại hiện vật gì, ở namespace nào, ai sở hữu, sống bao lâu? Bạn `init` lúc
   09:00, 12:00 mới join control plane node thứ hai — chuyện gì xảy ra, gõ lệnh nào để sửa?
5. Token và `--discovery-token-ca-cert-hash` giống nhau ở hai lệnh join. Hai giá trị đó trả lời câu
   hỏi gì? Hai cờ thừa của lệnh control plane khiến `kubeadm join` làm thêm việc gì?
6. Vì sao CNI phải có trước khi join, và vì sao CoreDNS dồn hết lên `lab-ha-1` lúc này không phải
   lỗi?

<details>
<summary>Gợi ý</summary>

1. Bài 08 dùng ba cờ; lab thêm `--kubernetes-version` (bài 08 nói được phép) và `--pod-network-cidr`
   (ghi chú CNI trong bài). Giá trị lấy từ `bien.sh`.
2. Kéo image trước bằng `kubeadm config images pull` để lỗi mạng hiện ra tách khỏi `init`.
3. Che log: hai biểu thức `sed -E` — một cho chuỗi dạng token `[a-z0-9]{6}\.[a-z0-9]{16}`, một cho
   chuỗi hex 64 ký tự.
4. Ba nơi ghi endpoint: kubeconfig của bạn; ConfigMap `cluster-info` trong `kube-public`; ConfigMap
   `kubeadm-config` trong `kube-system`.
5. Hạn của `kubeadm-certs`: đọc `ownerReferences` của Secret, rồi trường `expiration` của Secret
   bootstrap token mà nó trỏ tới.
6. CNI của baseline: đúng lệnh `kubectl apply` của A5.2, dùng `FLANNEL_VER` trong `bien.sh`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
source ~/lab-work/8b/bien.sh

grep -q 'Your Kubernetes control-plane has initialized successfully' "$WK/b4-init.log" \
  && echo 'PASS: kubeadm init thanh cong'
{ test -s "$EV/b4-init-da-che.log" && \
  ! grep -Eq '[0-9a-f]{64}|[a-z0-9]{6}\.[a-z0-9]{16}' "$EV/b4-init-da-che.log"; } \
  && echo 'PASS: evidence chi giu log init da che token va khoa'

# 1. Endpoint o ba noi
SRV="$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
CI="$(kubectl -n kube-public get configmap cluster-info \
      -o jsonpath='{.data.kubeconfig}' | grep -m1 'server:' | awk '{print $2}')"
KC="$(kubectl -n kube-system get configmap kubeadm-config \
      -o jsonpath='{.data.ClusterConfiguration}' | grep -m1 'controlPlaneEndpoint:' | awk '{print $2}')"
printf 'kubeconfig=%s\ncluster-info=%s\nkubeadm-config=%s\n' "$SRV" "$CI" "$KC" | tee "$EV/b4-endpoint.txt"
test "$SRV" = "https://$ENDPOINT" && echo 'PASS: kubeconfig tro vao load balancer'
test "$CI"  = "https://$ENDPOINT" && echo 'PASS: cluster-info tro vao load balancer'
test "$KC"  = "$ENDPOINT"         && echo 'PASS: kubeadm-config ghi dung controlPlaneEndpoint'

# 2. Hai lenh join — output that xuong dong bang dau \ nen noi lai truoc khi so
sed -e ':a' -e 'N' -e '$!ba' -e 's/\\\n[[:space:]]*/ /g' "$WK/b4-init.log" \
  | grep -E '^[[:space:]]*kubeadm join' \
  | sed 's/^[[:space:]]*//' | tr -s ' ' > "$WK/join-commands.txt"
test "$(wc -l < "$WK/join-commands.txt")" -eq 2 && echo 'PASS: init in ra dung hai lenh join'
CP_JOIN="$(grep -- '--control-plane' "$WK/join-commands.txt")"
WORKER_JOIN="$(grep -v -- '--control-plane' "$WK/join-commands.txt")"
CERT_KEY="$(sed -nE 's/.*--certificate-key ([0-9a-f]+).*/\1/p' <<< "$CP_JOIN")"
STRIPPED="$(sed -E 's/ --control-plane//; s/ --certificate-key [0-9a-f]+//' <<< "$CP_JOIN")"
test "$STRIPPED" = "$WORKER_JOIN" \
  && echo 'PASS: hai lenh join chi khac dung hai co --control-plane va --certificate-key'
echo "$CERT_KEY" | grep -Eq '^[0-9a-f]{64}$' && echo 'PASS: certificate key la chuoi hex 64 ky tu — khoa AES 32 byte'
test "$(grep -c "$ENDPOINT" "$WK/join-commands.txt")" -eq 2 \
  && echo 'PASS: ca hai lenh join tro vao load balancer, khong vao IP mot node'
kubeadm certs certificate-key > "$WK/mau-certificate-key.txt"
grep -Eq '^[0-9a-f]{64}$' "$WK/mau-certificate-key.txt" \
  && echo 'PASS: kubeadm certs certificate-key sinh mot khoa cung hinh dang'

# 3. Hien vat cua --upload-certs va han hai gio
kubectl -n kube-system get secret kubeadm-certs \
  -o go-template='{{range $k, $v := .data}}{{$k}}{{"\n"}}{{end}}' > "$EV/b4-kubeadm-certs.txt"
test "$(grep -c . "$EV/b4-kubeadm-certs.txt")" -ge 1 \
  && echo 'PASS: Secret kubeadm-certs ton tai va co certificate ben trong'
OWNER="$(kubectl -n kube-system get secret kubeadm-certs -o jsonpath='{.metadata.ownerReferences[0].name}')"
test -n "$OWNER" && echo 'PASS: kubeadm-certs co ownerReference toi mot bootstrap token'
CRT="$(kubectl -n kube-system get secret kubeadm-certs -o jsonpath='{.metadata.creationTimestamp}')"
EXP="$(kubectl -n kube-system get secret "$OWNER" -o jsonpath='{.data.expiration}' | base64 -d)"
DIFF=$(( $(date -d "$EXP" +%s) - $(date -d "$CRT" +%s) ))
printf 'owner=%s\ntao=%s\nhet_han=%s\nchenh_giay=%s\ntao_lai=sudo kubeadm init phase upload-certs --upload-certs\n' \
  "$OWNER" "$CRT" "$EXP" "$DIFF" | tee "$EV/b4-upload-certs.txt"
test "$DIFF" -ge 7000 && test "$DIFF" -le 7400 \
  && echo 'PASS: vong doi cua kubeadm-certs khoang hai gio nhu bai 08 mo ta'

# 4. CNI
NET="$(kubectl -n kube-flannel get configmap kube-flannel-cfg \
       -o jsonpath='{.data.net-conf\.json}' | tr -d ' \n' | sed -nE 's/.*"Network":"([^"]+)".*/\1/p')"
kubectl -n kube-flannel get daemonset kube-flannel-ds \
  -o jsonpath='{range .spec.template.spec.initContainers[*]}{.image}{"\n"}{end}{range .spec.template.spec.containers[*]}{.image}{"\n"}{end}' \
  > "$EV/b4-cni-images.txt"
test "$NET" = "$POD_CIDR" && echo 'PASS: Pod CIDR cua CNI khop --pod-network-cidr da dung o init'
grep -q "$FLANNEL_VER" "$EV/b4-cni-images.txt" && echo 'PASS: image CNI dung release khai o bang A1.3'
kubectl wait --for=condition=Ready node/"$CP1" --timeout=300s >/dev/null && echo 'PASS: lab-ha-1 Ready'

# 5. CoreDNS luc cluster moi co mot node — so lieu "truoc" cho B5
kubectl -n kube-system rollout status deployment/coredns --timeout=300s >/dev/null
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide | tee "$EV/b4-coredns-truoc.txt"
DNS_NODES_1="$(kubectl -n kube-system get pods -l k8s-app=kube-dns \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | grep -c .)"
echo "$DNS_NODES_1" > "$EV/b4-coredns-so-node.txt"
test "$DNS_NODES_1" -eq 1 && echo 'PASS: toan bo Pod CoreDNS dang dong tren control plane node dau tien'

grep -q -- '--control-plane-endpoint' "$REPO/platform/ha-lab/README.md" \
  && ! grep -Eq '[0-9a-f]{64}|[a-z0-9]{6}\.[a-z0-9]{16}' "$REPO/platform/ha-lab/README.md" \
  && echo 'PASS: README ghi lenh init, khong chua token hay khoa'
```

`ownerReference` rỗng hoặc `DIFF` lệch xa hai giờ: xem [mục 4](#4-troubleshooting-của-lab-này).

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
# Chay tren lab-ha-1 — KHONG ngat lenh init giua chung
source ~/lab-work/8b/bien.sh
sudo kubeadm config images list --kubernetes-version "$K8S_VER"
sudo kubeadm config images pull --kubernetes-version "$K8S_VER"

sudo kubeadm init \
  --kubernetes-version "$K8S_VER" \
  --control-plane-endpoint "$ENDPOINT" \
  --apiserver-advertise-address "$CP1_IP" \
  --pod-network-cidr "$POD_CIDR" \
  --upload-certs 2>&1 | tee "$WK/b4-init.log"
sed -E -e 's/[a-z0-9]{6}\.[a-z0-9]{16}/<token>/g' -e 's/[0-9a-f]{64}/<hex>/g' \
  "$WK/b4-init.log" > "$EV/b4-init-da-che.log"

mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
chmod 600 "$HOME/.kube/config"
for i in $(seq 1 60); do kubectl get --raw='/healthz' >/dev/null 2>&1 && break; sleep 5; done

kubectl apply -f \
  "https://github.com/flannel-io/flannel/releases/download/${FLANNEL_VER}/kube-flannel.yml"
kubectl -n kube-flannel rollout status daemonset/kube-flannel-ds --timeout=300s
kubectl wait --for=condition=Ready node/"$CP1" --timeout=300s
```

Thêm vào `platform/ha-lab/README.md`:

````markdown
## Control plane đầu tiên (lab-ha-1)

Giá trị `K8S_VER`, `POD_CIDR`, `FLANNEL_VER` lấy từ bảng A1.3 của Lab 00.

```bash
sudo kubeadm init --kubernetes-version "$K8S_VER" \
  --control-plane-endpoint lab-ha-lb:6443 --apiserver-advertise-address 192.168.100.231 \
  --pod-network-cidr "$POD_CIDR" --upload-certs
kubectl apply -f "https://github.com/flannel-io/flannel/releases/download/${FLANNEL_VER}/kube-flannel.yml"
```

Lệnh join in ra chứa token và certificate key — không ghi vào repo. Khóa hết hạn sau hai giờ:
`sudo kubeadm init phase upload-certs --upload-certs` trên một control plane node đã join.
````

Năm cờ, năm lý do:

| Cờ | Vì sao có mặt |
| --- | --- |
| `--kubernetes-version "$K8S_VER"` | không để kubeadm dò remote rồi chọn patch khác package đã cài; giống A5.1 |
| `--control-plane-endpoint "$ENDPOINT"` | **cờ quyết định**: endpoint dùng chung, trỏ vào **load balancer**. A5.1 cũng có cờ này nhưng trỏ vào chính `lab-k8s-master` |
| `--apiserver-advertise-address "$CP1_IP"` | địa chỉ quảng bá của **riêng** node này — mỗi control plane node một giá trị |
| `--pod-network-cidr "$POD_CIDR"` | CNI của baseline cần Pod CIDR khớp manifest của nó; giống A5.1 |
| `--upload-certs` | đưa certificate control plane lên cluster cho node control plane sau lấy về; A5.1 không có vì cluster đó không bao giờ có control plane thứ hai |

Ý chính cho ADR:

1. `--control-plane-endpoint` trỏ vào LB. Giá trị endpoint được nướng vào certificate của apiserver
   và vào mọi kubeconfig; bài 02 nói chuyển cluster tạo **không có** cờ này thành HA không được
   kubeadm hỗ trợ.
2. Endpoint giống nhau trên ba node; địa chỉ quảng bá khác nhau trên từng node.
3. Kubeconfig — quản trị viên; `cluster-info` trong `kube-public` — node đang join đọc lúc discovery;
   `kubeadm-config` — chính kubeadm ở các lần chạy sau. Lệch một nơi là một nhóm khách bị chỉ sai.
4. Secret `kubeadm-certs` trong `kube-system`, chứa certificate control plane đã mã hóa; chủ sở hữu
   là một bootstrap token sống hai giờ, token hết hạn thì Secret bị dọn theo; khóa giải mã hết hạn
   cùng lúc. Join lúc 12:00 thì hỏng — chạy `sudo kubeadm init phase upload-certs --upload-certs`
   để có khóa mới, cộng `kubeadm token create --print-join-command` nếu token join cũng đã hết.
5. Token trả lời "node này có quyền join không"; CA hash trả lời "cluster ở đầu kia có đúng là
   cluster tôi nghĩ không". `--control-plane` bảo `kubeadm join` dựng control plane mới trên node;
   `--certificate-key` tải certificate từ `kubeadm-certs` về và giải mã.
6. Bài 08: phải triển khai network plugin trước bước tiếp theo, nếu không cluster không khởi chạy
   bình thường. CoreDNS được lập lịch lúc chỉ có `lab-ha-1`; Pod đã lập lịch thì ở yên — B5 xử lý.

</details>

## B5. [BS-8B-05] Ba control plane, hai worker

**Loại:** Vận hành.

**Bối cảnh:** một control plane chưa phải HA. Sprint cần ba control plane node — mức tối thiểu của
bài 06 — và hai worker để Bookstore có chỗ chạy. Sau đó phải chứng minh load balancer **thật sự** rải
request lên cả ba, không chỉ "trông giống HA".

**Yêu cầu:**

1. `lab-ha-2` rồi `lab-ha-3` join làm control plane, **từng node một**, bằng khóa giải mã còn hạn.
   Sau mỗi lần join, số etcd member bằng số control plane node.
2. `lab-ha-4`, `lab-ha-5` join làm worker.
3. Năm node `Ready`; ba control plane node **giữ** taint cách ly; hai worker không mang taint nào.
4. Load balancer thấy ba backend `UP` và đã chuyển kết nối mới tới **cả ba** apiserver.
5. CoreDNS trải ra nhiều hơn một node.
6. `platform/ha-lab/README.md` thêm mục *Join* — các bước, không chứa token hay khóa.

**Ràng buộc:** lệnh join chạy **trực tiếp** trên máy đó (cần `sudo` có tty). Không gỡ taint control
plane. Không ép vị trí CoreDNS bằng `nodeSelector`.

**Đọc lại:**

- Bài [08](../08-high-availability-vi.md), mục
  [*Các bước cho các node control plane còn lại*](../08-high-availability-vi.md#các-bước-cho-các-node-control-plane-còn-lại-steps-for-the-rest-of-the-control-plane-nodes)
  — hai cờ, và ghi chú về CoreDNS; mục *Các bước cho các node control plane còn lại* của nhánh
  external — "join từng node một", "khóa hết hạn sau hai giờ";
  [*Cài đặt các worker*](../08-high-availability-vi.md#cài-đặt-các-worker-install-workers).
- Trang [214](../214-kubeadm-tasks-vi.md#danh-sách-các-trang-trong-mục-này-pages-in-this-section) —
  *Thêm worker node Linux*.
- Bài [02](../02-create-cluster-kubeadm-vi.md#cách-ly-control-plane-node-control-plane-node-isolation)
  — *Cách ly control plane node*.
- Bài [22](../22-architecture-vi.md#nhieu-ban-apiserver) — ba apiserver chạy active-active.

**Bàn giao:** `platform/ha-lab/README.md` cập nhật; `~/lab-evidence/8b/b5-*.txt`.

**Nghiệm thu:**

- `PASS:` sau join `lab-ha-2`: 2 member, 2 control plane; sau `lab-ha-3`: 3 và 3.
- `PASS:` 5 node, 3 control plane, cả 5 `Ready`; 3 control plane giữ taint `NoSchedule`; worker không
  taint.
- `PASS:` 3 backend `UP`; mỗi backend đã nhận ít nhất một phiên.
- `PASS:` số node chứa Pod CoreDNS lớn hơn con số ghi ở B4.

**Câu hỏi thiết kế** — ADR topology, mục `## Stacked (Lab 8b)`, tiểu mục *Mở rộng*:

1. Một lệnh `join --control-plane` thêm **những gì** vào cluster? Đó là mặt tích cực của tính "gắn
   kết" mà bài 06 nói; mặt tiêu cực sẽ hiện ra ở đâu?
2. Vì sao join control plane **từng node một**?
3. Vì sao ba `kube-apiserver` cùng nhận request được mà không cần phối hợp với nhau, trong khi
   scheduler và controller-manager thì không làm vậy?
4. Vì sao `rollout restart` dời được CoreDNS trong khi Kubernetes không tự dời Pod đang chạy?
5. Vì sao giữ taint trên cả ba control plane node, dù chúng còn dư CPU?

<details>
<summary>Gợi ý</summary>

1. Khóa mới và lệnh join mới, chạy trên `lab-ha-1`: `sudo kubeadm init phase upload-certs
   --upload-certs` (dòng cuối là khóa) và `sudo kubeadm token create --print-join-command`.
2. Ghép hai thứ đó thành lệnh control plane; lệnh worker là lệnh join không có hai cờ.
3. Đếm member: hàm `etcd1 --endpoints=https://127.0.0.1:2379 member list`.
4. LB có rải không: gửi khoảng mười lăm kết nối mới tới `https://<IP LB>:6443/healthz` rồi đọc
   `lbstot`.

</details>

<details>
<summary>Script nghiệm thu — chạy từng phần theo đúng thời điểm</summary>

Phần 1 — chạy trên `lab-ha-1` **ngay sau khi `lab-ha-2` join xong**, trước khi join `lab-ha-3`:

```bash
source ~/lab-work/8b/bien.sh
kubectl wait --for=condition=Ready node/"$CP2" --timeout=600s
MEM_N="$(etcd1 --endpoints=https://127.0.0.1:2379 member list | wc -l)"
CP_N="$(kubectl get nodes -l node-role.kubernetes.io/control-plane -o name | wc -l)"
echo "sau join $CP2: etcd member=$MEM_N | control plane node=$CP_N" | tee "$EV/b5-join-cp2.txt"
test "$MEM_N" -eq 2 && test "$CP_N" -eq 2 \
  && echo 'PASS: join --control-plane vua them mot control plane VA mot etcd member'
```

Phần 2 — sau khi `lab-ha-3`, `lab-ha-4`, `lab-ha-5` đã join và CoreDNS đã được cân bằng lại:

```bash
source ~/lab-work/8b/bien.sh
kubectl wait --for=condition=Ready node --all --timeout=600s
kubectl get nodes -o wide | tee "$EV/b5-nodes.txt"

MEM_N="$(etcd1 --endpoints=https://127.0.0.1:2379 member list | wc -l)"
ALL_N="$(kubectl get nodes -o name | wc -l)"
CP_N="$(kubectl get nodes -l node-role.kubernetes.io/control-plane -o name | wc -l)"
RDY_N="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}' | grep -c '^True$')"
echo "node=$ALL_N | control plane=$CP_N | Ready=$RDY_N | etcd member=$MEM_N"
test "$MEM_N" -eq 3 && test "$CP_N" -eq 3 \
  && echo 'PASS: du ba control plane node va ba etcd member — muc toi thieu cua bai 06'
test "$ALL_N" -eq 5 && test "$RDY_N" -eq 5 && echo 'PASS: cluster co 5 node, tat ca Ready'

TAINT_N="$(kubectl get nodes -l node-role.kubernetes.io/control-plane \
  -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"="}{.effect}{"\n"}{end}{end}' \
  | grep -c 'node-role.kubernetes.io/control-plane=NoSchedule')"
WK_TAINT="$(kubectl get nodes "$WN1" "$WN2" -o jsonpath='{range .items[*]}{.spec.taints}{"\n"}{end}' | grep -c .)"
test "$TAINT_N" -eq 3 && echo 'PASS: ca ba control plane node van giu taint cach ly'
test "$WK_TAINT" -eq 0 && echo 'PASS: hai worker khong mang taint nao'

lbstat | tee "$EV/b5-lb-status.txt"
test "$(grep -c '=UP$' "$EV/b5-lb-status.txt")" -eq 3 && echo 'PASS: health check TCP thay ca ba apiserver khoe manh'
for i in $(seq 1 15); do curl -sk -o /dev/null --max-time 5 "https://${LB_IP}:6443/healthz"; done
lbstot | tee "$EV/b5-lb-stot.txt"
NZ="$(awk -F= '$2 + 0 > 0' "$EV/b5-lb-stot.txt" | wc -l)"
test "$NZ" -eq 3 && echo 'PASS: load balancer da chuyen phien toi ca ba kube-apiserver'

kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide | tee "$EV/b5-coredns-sau.txt"
DNS_NODES_1="$(cat "$EV/b4-coredns-so-node.txt")"
DNS_NODES_2="$(kubectl -n kube-system get pods -l k8s-app=kube-dns \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | grep -c .)"
echo "so node chua CoreDNS: truoc=$DNS_NODES_1 -> sau=$DNS_NODES_2"
test "$DNS_NODES_2" -gt "$DNS_NODES_1" && echo 'PASS: CoreDNS da trai ra nhieu hon mot node'

grep -qi 'join' "$REPO/platform/ha-lab/README.md" \
  && ! grep -Eq '[0-9a-f]{64}|[a-z0-9]{6}\.[a-z0-9]{16}' "$REPO/platform/ha-lab/README.md" \
  && echo 'PASS: README co muc join, khong chua token hay khoa'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
# Chay tren lab-ha-1 — khoa moi va lenh join moi, thay vi hy vong khoa cu con han
source ~/lab-work/8b/bien.sh
NEW_KEY="$(sudo kubeadm init phase upload-certs --upload-certs | tail -n 1)"
JOIN_BASE="$(sudo kubeadm token create --print-join-command)"
echo "$NEW_KEY" | grep -Eq '^[0-9a-f]{64}$' && echo 'PASS: da co certificate key moi'
grep -q "^kubeadm join $ENDPOINT " <<< "$JOIN_BASE" && echo 'PASS: lenh join moi van tro vao load balancer'
printf 'sudo %s --control-plane --certificate-key %s\n' "$JOIN_BASE" "$NEW_KEY" > "$WK/join-control-plane.txt"
printf 'sudo %s\n' "$JOIN_BASE" > "$WK/join-worker.txt"
cat "$WK/join-control-plane.txt" "$WK/join-worker.txt"
```

```bash
# Chay tren lab-ha-2; xong thi chay Phan 1 cua script nghiem thu; roi lap lai tren lab-ha-3
sudo kubeadm join lab-ha-lb:6443 --token <token-that> \
  --discovery-token-ca-cert-hash sha256:<hash-that> \
  --control-plane --certificate-key <khoa-that>
```

```bash
# Chay tren lab-ha-4, roi tren lab-ha-5
sudo kubeadm join lab-ha-lb:6443 --token <token-that> \
  --discovery-token-ca-cert-hash sha256:<hash-that>
```

```bash
# Chay tren lab-ha-1 — sau khi du nam node Ready
kubectl wait --for=condition=Ready node --all --timeout=600s
kubectl -n kube-system rollout restart deployment coredns
kubectl -n kube-system rollout status deployment coredns --timeout=300s
```

Mục *Join* trong `platform/ha-lab/README.md`: khóa mới bằng `kubeadm init phase upload-certs`, lệnh
mới bằng `kubeadm token create --print-join-command`; control plane thêm `--control-plane
--certificate-key <khóa>`, join từng node một; worker dùng lệnh không có hai cờ đó; sau cùng
`rollout restart deployment coredns`.

Ý chính cho ADR:

1. Một lệnh thêm **một instance control plane và một etcd member** cùng lúc — dựng HA rẻ. Mặt tiêu
   cực: mất một máy cũng mất cả hai cùng lúc (B9).
2. Mỗi control plane node mới thêm một member vào etcd đang chạy; thêm từng cái một để cluster etcd
   ổn định trước khi đổi thành viên lần nữa. Bài 08 khuyến nghị trực tiếp cách này.
3. Bài 22: apiserver không giữ trạng thái — mọi bản tra cùng một etcd, nên chạy active-active.
   Scheduler và controller-manager phải "một người quyết", nên chạy theo bầu leader.
4. `rollout restart` tạo Pod **mới**; scheduler khi đó chấm điểm năm node thay vì một. Kubernetes chỉ
   phân bố lại khi có sự kiện tạo Pod mới.
5. Ba control plane node đang gánh etcd và control plane; workload thường lên đó là tranh tài nguyên
   với chính thứ giữ cluster sống. Bài 02 chỉ gỡ taint cho cluster một máy.

</details>

## B6. [BS-8B-06] Bằng chứng: đây là stacked

**Loại:** Điều tra.

**Bối cảnh:** ADR ghi "stacked" là một lời tuyên bố. Trước khi đo sự cố và đổ lỗi cho topology, đội
phải chứng minh bằng cluster thật rằng nó **đúng là** stacked — không tin lời hứa của tài liệu.

**Yêu cầu:** chứng minh bằng ba dữ kiện độc lập, ghi kết luận vào ADR:

1. Mỗi Pod etcd là **static Pod** nằm trên đúng control plane node mang tên nó — Pod trên API chỉ
   là **mirror Pod** của một file trên đĩa node (kiểm file trên `lab-ha-1`); không node nào khác
   chạy etcd.
2. Số etcd member **bằng** số control plane node, và tên member trùng tên ba node.
3. Ba endpoint etcd khỏe, hợp thành **một** cluster có đúng **một** leader.

**Ràng buộc:** chỉ đọc; không sửa manifest nào. Thư mục static Pod chỉ root đọc được — đọc nó bằng
`sudo` trên `lab-ha-1`, không bọc `sudo` qua `ssh`.

**Đọc lại:**

- Bài [06](../06-ha-topology-vi.md#stacked-etcd-topology) — câu cuối mục stacked về `kubeadm init` và
  `kubeadm join --control-plane`.
- Bài [58](../58-static-pods-vi.md#mirror-pods) — mirror Pod: Pod trên API là bản phản chiếu của thứ
  kubelet đọc từ file.

**Bàn giao:** ADR topology, tiểu mục *Bằng chứng stacked*; `~/lab-evidence/8b/b6-*.txt`.

**Nghiệm thu:**

- `PASS:` ba Pod etcd, mỗi Pod trên đúng node mang tên nó, cả ba có `config.source=file` và là
  mirror Pod; etcd chỉ chạy trên control plane node; `lab-ha-1` có `etcd.yaml` trong thư mục static
  Pod.
- `PASS:` số member bằng số control plane node, tên khớp.
- `PASS:` ba endpoint `is healthy`; bảng status có ba endpoint.

**Câu hỏi thiết kế** — ADR topology:

1. Annotation `kubernetes.io/config.source=file` chứng minh điều gì mà `kubectl get pod` không nói?
   Vì sao tên Pod luôn là `etcd-<tên node>`?
2. Ba member chung một cluster và bầu một leader. Điều đó có nghĩa `kube-apiserver` trên `lab-ha-2`
   nói chuyện với cả ba member không?
3. Dự đoán cho B9: tắt `lab-ha-3` thì `member list` còn mấy dòng? Số member **khỏe** còn mấy?

<details>
<summary>Gợi ý</summary>

- Pod etcd mang label `component=etcd` trong `kube-system`.
- Annotation có dấu chấm trong jsonpath: `{.metadata.annotations.kubernetes\.io/config\.source}`.
- Mirror Pod mang thêm annotation `kubernetes.io/config.mirror`. Thư mục static Pod:
  `sudo ls /etc/kubernetes/manifests/` trên `lab-ha-1`.
- Hàm `etcd1` nhận mọi đối số của `etcdctl`: `member list -w table`, `endpoint health`,
  `endpoint status -w table` với `--endpoints="$ETCD_EPS"`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
source ~/lab-work/8b/bien.sh
kubectl -n kube-system get pod -l component=etcd -o wide | tee "$EV/b6-etcd-pods.txt"
MATCH="$(kubectl -n kube-system get pod -l component=etcd \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.nodeName}{"\n"}{end}' \
  | awk '{ if ("etcd-"$2 == $1) n++ } END { print n+0 }')"
STATIC="$(kubectl -n kube-system get pod -l component=etcd \
  -o jsonpath='{range .items[*]}{.metadata.annotations.kubernetes\.io/config\.source}{"\n"}{end}' \
  | grep -c '^file$')"
ETCD_NODES="$(kubectl -n kube-system get pod -l component=etcd \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | tr '\n' ' ')"
ETCD_ON_CP="$(echo "$ETCD_NODES" | tr ' ' '\n' | grep -cE "^($CP1|$CP2|$CP3)$")"
ETCD_NODE_N="$(echo "$ETCD_NODES" | tr ' ' '\n' | grep -c .)"
test "$MATCH" -eq 3 && echo 'PASS: moi Pod etcd nam tren dung control plane node mang ten no'
test "$STATIC" -eq 3 && echo 'PASS: ca ba deu la static Pod — kubelet doc tu file, khong qua scheduler'
test "$ETCD_NODE_N" -eq 3 && test "$ETCD_ON_CP" -eq 3 && echo 'PASS: etcd chi chay tren control plane node'
MIRROR="$(kubectl -n kube-system get pod -l component=etcd \
  -o jsonpath='{range .items[*]}{.metadata.annotations.kubernetes\.io/config\.mirror}{"\n"}{end}' | grep -c .)"
test "$MIRROR" -eq 3 && echo 'PASS: ca ba Pod etcd tren API la mirror Pod'
sudo ls /etc/kubernetes/manifests/ | tee "$EV/b6-manifests-lab-ha-1.txt"
grep -qx 'etcd.yaml' "$EV/b6-manifests-lab-ha-1.txt" && echo 'PASS: lab-ha-1 co etcd.yaml trong thu muc static Pod'

etcd1 --endpoints=https://127.0.0.1:2379 member list -w table | tee "$EV/b6-members.txt"
MEM_N="$(etcd1 --endpoints=https://127.0.0.1:2379 member list | wc -l)"
CP_N="$(kubectl get nodes -l node-role.kubernetes.io/control-plane -o name | wc -l)"
NAME_OK=0
for n in $CPS; do grep -q "$n" "$EV/b6-members.txt" && NAME_OK=$(( NAME_OK + 1 )); done
test "$MEM_N" -eq "$CP_N" && echo 'PASS: so member etcd BANG so control plane node — dinh nghia cua stacked'
test "$NAME_OK" -eq 3 && echo 'PASS: ba member mang dung ten ba control plane node'

etcd1 --endpoints="$ETCD_EPS" endpoint health > "$EV/b6-etcd-health.txt" 2>&1 || true
etcd1 --endpoints="$ETCD_EPS" endpoint status -w table > "$EV/b6-etcd-status.txt" 2>&1 || true
cat "$EV/b6-etcd-status.txt"
test "$(grep -c 'is healthy' "$EV/b6-etcd-health.txt")" -eq 3 && echo 'PASS: ca ba etcd member deu khoe manh'
test "$(grep -c 'https://' "$EV/b6-etcd-status.txt")" -eq 3 && echo 'PASS: bang status liet ke du ba endpoint'
grep -qi 'stacked' "$REPO"/docs/adr/*-topology-ha.md && grep -q 'config.source' "$REPO"/docs/adr/*-topology-ha.md \
  && echo 'PASS: ADR ghi bang chung stacked'
```

Trong bảng `endpoint status`, đúng **một** dòng có `true` ở cột `IS LEADER`.

</details>

<details>
<summary>Lời giải tham khảo — ý chính cho ADR</summary>

1. `config.source=file`: Pod này không do scheduler đặt — kubelet đọc manifest trong
   `/etc/kubernetes/manifests` của **chính máy nó**, tạo container, rồi báo lên API một mirror Pod
   (bài 58). Vì thế tên luôn kèm tên node, và tắt máy nào là mất đúng member của máy đó. Bài 06 gọi
   đây là etcd member cục bộ tự tạo khi `kubeadm init` và `kubeadm join --control-plane`.
2. Không. Quan hệ "chung một etcd cluster" là giữa các member; quan hệ "apiserver ↔ member" trong
   stacked là một-một: apiserver trên `lab-ha-2` chỉ nói với member trên `lab-ha-2`. Ở external,
   quan hệ thứ hai mới là nhiều-nhiều.
3. `member list` vẫn ba dòng: rời cluster là một thao tác quản trị, không phải hệ quả của tắt máy.
   Số member khỏe còn hai.

</details>

## B7. [BS-8B-07] Overlay `ha-lab`: Bookstore lên cluster HA

**Loại:** Tính năng.

**Bối cảnh:** đo HA bằng một Deployment busybox thì chỉ chứng minh được busybox. Sprint cần chính hệ
thống Bookstore của tag `lab-8a` chạy trên cluster HA. Nhưng cluster này chỉ có hạ tầng của baseline:
không ingress controller, không StorageClass, CNI không thực thi NetworkPolicy, không node nào tên
`lab-k8s-*` hay mang label riêng của dự án. Quota của Lab 7b còn cấm hẳn Service NodePort. Và
`k8s/app` **không được đổi** — Lab 9a mở đầu bằng chính nó trên chuỗi chính. Kustomize đã có đúng
công cụ cho tình huống "một base, nhiều môi trường": overlay. Lab 8c sẽ dùng lại overlay và ba lệnh
của ticket này, chỉ thêm patch của riêng nó.

**Yêu cầu:**

1. `k8s/overlays/ha-lab/kustomization.yaml` dùng `k8s/app` làm base, cộng hai PriorityClass của dự án
   từ `k8s/cluster/`. Không sửa file nào trong `k8s/app/`, `k8s/cluster/`, và không sửa
   `scripts/deploy.sh`, `scripts/teardown.sh`, `tests/smoke.sh`.
2. Kết quả render **không** chứa object hứa một điều cluster HA không làm được: Ingress (cùng Secret
   TLS chỉ Ingress dùng), PersistentVolumeClaim, StorageClass, và claim template của `db`.
3. `db` giữ dữ liệu trên một volume tạm thay cho claim template; `db-backup` ghi bản dump vào volume
   tạm thay cho PVC `backup-store`.
4. `db` và `db-backup` không bị ghim vào node nào của chuỗi chính — không theo tên `lab-k8s-*`, không
   theo label `bookstore.example.com/data` mà `deploy.sh` của chuỗi chính gắn. Overlay không cần label
   node nào; vị trí của `db-0` để scheduler quyết.
5. Khách ngoài cluster vào `web` qua NodePort `30080` trên mỗi worker, như hợp đồng của Lab 5a
   ([mục 4.2](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe)): trang chủ và
   `/api/books` đều trả lời. ResourceQuota `bookstore-quota` của Lab 7b **vẫn có hiệu lực** trên
   cluster HA nhưng cho phép đúng một NodePort — dự án chọn vá chứ không bỏ quota; lý do ghi ADR.
6. NetworkPolicy: **tự quyết** giữ hay bỏ, ghi lý do vào ADR.
7. Mọi hợp đồng khác giữ nguyên: tên, label, PriorityClass, PDB, LimitRange, probe, tài nguyên, chính
   sách CronJob, DaemonSet trên mọi node.
8. `scripts/deploy-ha-lab.sh`, `scripts/teardown-ha-lab.sh` — chép nguyên văn khối bên dưới;
   `tests/smoke-ha-lab.sh` — chép nguyên văn từ [B13.1](#b131-kiểm-thử-khói-của-sprint).
9. `README.md` ở gốc repo có mục cho bộ VM HA, tiêu đề chứa chữ `ha-lab`, liệt kê ngay bên dưới đúng
   ba lệnh chạy từ gốc repo — triển khai, gỡ bỏ, kiểm thử khói. Lab 8c đọc ba lệnh từ đây.
10. Teardown sạch; triển khai hai lần liên tiếp không lỗi; kiểm thử khói PASS.

**Ràng buộc:** không cài Calico, Traefik, local-path-provisioner hay metrics-server lên cluster HA.
Không chạy `scripts/deploy.sh` trên cluster HA. Render chỉ qua `deploy-ha-lab.sh`, tức
`kubectl kustomize --load-restrictor=LoadRestrictionsNone` như mọi lần triển khai từ Lab 3b — base
đọc `secrets/` ở gốc repo, và overlay lấy file từ `k8s/cluster/`, cả hai nằm ngoài thư mục overlay.

**Đọc lại:**

- Bài [322](../322-kustomization-vi.md): [*Tùy biến*](../322-kustomization-vi.md#tùy-biến-customizing)
  — `patches`, đích chọn theo `kind`/`name`, StrategicMerge và Json6902;
  [*Base và Overlay*](../322-kustomization-vi.md#base-và-overlay-bases-and-overlays).
- Bài [82](../82-service-vi.md#type-nodeport), *`type: NodePort`* và
  [*Tự chọn port*](../82-service-vi.md#nodeport-custom-port).
- Bài [84](../84-network-policies-vi.md#điều-kiện-tiên-quyết-prerequisites), *Điều kiện tiên quyết*
  — NetworkPolicy không có plugin mạng hỗ trợ thì không có tác dụng.
- Lab 3b, [ticket BS-3B-01](LAB-3B-CAU-HINH-UNG-DUNG.md#b1-bs-3b-01-bí-mật-sinh-tại-chỗ-triển-khai-bằng-kustomize)
  — vì sao mọi triển khai render bằng `--load-restrictor`.
- [Mục 4](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật) và [mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint)
  của file dự án.

**Bàn giao:** `k8s/overlays/ha-lab/`; `scripts/deploy-ha-lab.sh`, `scripts/teardown-ha-lab.sh`,
`tests/smoke-ha-lab.sh`; mục `ha-lab` trong `README.md`; `docs/adr/NNNN-overlay-ha-lab.md`;
`~/lab-evidence/8b/b7-*.txt`.

**Nghiệm thu:**

- `PASS:` `k8s/app`, `k8s/cluster` và ba script của chuỗi chính không đổi so với tag `lab-8a`.
- `PASS:` render không có Ingress, PVC, StorageClass, `volumeClaimTemplates`; không nhắc
  `lab-k8s-` hay `bookstore.example.com/data`; `bookstore-quota` còn và cho phép một NodePort.
- `PASS:` `teardown-ha-lab.sh` in `teardown: sach`; `deploy-ha-lab.sh` thành công hai lần liên tiếp.
- `PASS:` `tests/smoke-ha-lab.sh` không có dòng `FAIL:`.
- `PASS:` `README.md` có mục `ha-lab` với đúng ba lệnh; ADR overlay ghi quyết định về NetworkPolicy
  và quota.

Hai script triển khai cho cluster HA, chép nguyên văn:

```bash
cat > ~/bookstore/scripts/deploy-ha-lab.sh <<'EOF'
#!/usr/bin/env bash
# Triển khai Bookstore lên cluster HA của bộ VM lab-ha-* (Lab 8b, 8c) bằng overlay k8s/overlays/ha-lab.
# Chuỗi chính KHÔNG dùng script này: ở đó vẫn là scripts/deploy.sh.
set -euo pipefail
cd "$(dirname "$0")/.."
NS=bookstore
OVERLAY=k8s/overlays/ha-lab

# 0. Chặn chạy nhầm cluster: overlay chỉ dành cho bộ VM HA.
case "$(kubectl get nodes -o name)" in
  *node/lab-ha-*) ;;
  *) echo 'deploy-ha-lab: cluster nay khong phai bo VM lab-ha-* — dung scripts/deploy.sh'; exit 1 ;;
esac

# 1. Bí mật sinh tại chỗ khi thiếu, cùng định dạng với scripts/deploy.sh (Lab 3b). Base k8s/app còn
#    đọc secrets/tls.* cho Secret TLS của Lab 5b: overlay bỏ Secret đó, nhưng Kustomize vẫn đọc file
#    khi dựng base, nên sinh một certificate tự ký tạm nếu thiếu.
mkdir -p secrets
if [ ! -s secrets/db.env ]; then
  ( umask 077
    printf 'postgres-password=%s\nauthenticator-password=%s\n' \
      "$(openssl rand -hex 24)" "$(openssl rand -hex 24)" > secrets/db.env )
  echo 'deploy-ha-lab: da sinh secrets/db.env moi'
fi
if [ ! -s secrets/tls.crt ] || [ ! -s secrets/tls.key ]; then
  ( umask 077
    openssl req -x509 -newkey rsa:2048 -nodes -days 30 -subj '/CN=bookstore.lab.local' \
      -keyout secrets/tls.key -out secrets/tls.crt 2>/dev/null )
  echo 'deploy-ha-lab: da sinh certificate tu ky tam cho base'
fi

# 2. Render một lần; từ chối nếu còn thứ mà cluster HA không phục vụ được.
RENDER="$(mktemp)"
trap 'rm -f "$RENDER"' EXIT
kubectl kustomize --load-restrictor=LoadRestrictionsNone "$OVERLAY" > "$RENDER"
if grep -Eq '^kind: (Ingress|PersistentVolumeClaim|StorageClass)$|volumeClaimTemplates:' "$RENDER"; then
  echo 'deploy-ha-lab: overlay con Ingress, PVC, StorageClass hoac volumeClaimTemplates'; exit 1
fi
if ! grep -q 'app.kubernetes.io/component: migrate' "$RENDER"; then
  echo 'deploy-ha-lab: overlay khong render Job db-migrate'; exit 1
fi

# 3. Pha 1: mọi thứ trừ migration. Template của Job không sửa được sau khi tạo: xóa Job cũ trước.
kubectl -n "$NS" delete job db-migrate --ignore-not-found --wait=true >/dev/null 2>&1 || true
kubectl apply -f "$RENDER" -l 'app.kubernetes.io/component!=migrate'
kubectl -n "$NS" rollout status statefulset/db --timeout=300s
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=300s

# 4. Pha 2: migration chỉ chạy khi db-0 đã nhận kết nối; nó bất biến kết quả (Lab 4b).
kubectl apply -f "$RENDER" -l 'app.kubernetes.io/component=migrate'
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
wait_job db-migrate || { echo 'deploy-ha-lab: db-migrate that bai'; exit 1; }

# 5. Chờ các controller còn lại.
for D in web api; do kubectl -n "$NS" rollout status "deployment/$D" --timeout=300s; done
kubectl -n "$NS" rollout status daemonset/node-agent --timeout=300s
kubectl -n "$NS" get statefulset,deployment,daemonset,cronjob,job,service -o wide
EOF

cat > ~/bookstore/scripts/teardown-ha-lab.sh <<'EOF'
#!/usr/bin/env bash
# Gỡ Bookstore khỏi cluster HA (Lab 8b, 8c). Không đụng node, không đụng hạ tầng của cluster.
# Overlay ha-lab chỉ dùng emptyDir nên không có dữ liệu nào sót trên đĩa node.
set -euo pipefail
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
case "$(kubectl get nodes -o name)" in
  *node/lab-ha-*) ;;
  *) echo 'teardown-ha-lab: cluster nay khong phai bo VM lab-ha-* — dung scripts/teardown.sh'; exit 1 ;;
esac

# Thu db về 0 trước để db-0 tắt êm (bài 65, mục Hạn chế — Lab 4b), rồi mới xóa namespace.
if kubectl -n "$NS" get statefulset db >/dev/null 2>&1; then
  kubectl -n "$NS" scale statefulset db --replicas=0
  kubectl -n "$NS" wait --for=delete pod/db-0 --timeout=180s || true
fi
kubectl delete namespace -l "$SEL" --ignore-not-found --wait=true --timeout=300s
kubectl delete pods,configmaps,secrets --all-namespaces -l "$SEL" --ignore-not-found --wait=true --timeout=120s
kubectl delete priorityclass,storageclass,persistentvolume -l "$SEL" --ignore-not-found
# Label node của dự án (bookstore.example.com/*): overlay không cần, nhưng gỡ nếu ai đó đã gắn tay.
KEYS="$(kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.labels}{"\n"}{end}' \
        | grep -o '"bookstore\.example\.com/[^"]*"' | tr -d '"' | sort -u || true)"
for K in $KEYS; do kubectl label nodes --all "$K-" >/dev/null || true; done

# Mỗi truy vấn kiểm kê chạy riêng và phải thành công: API không trả lời hay Forbidden thì dừng —
# "không hỏi được" không bao giờ được đếm thành "không còn gì". dem <biến> <lệnh...> cộng số dòng
# output của lệnh vào biến; hàm chạy trong shell chính nên exit dừng cả script.
dem() {
  local v="$1" out; shift
  out="$("$@" 2>/dev/null)" || { echo "teardown: STOP - truy van loi, chua ket luan sach: $*"; exit 1; }
  printf -v "$v" '%d' "$(( ${!v:-0} + $(printf '%s\n' "$out" | awk 'NF' | wc -l) ))"
}
node_labels() {   # node còn mang label bookstore.example.com/*; lấy dữ liệu trước, lọc sau
  local l; l="$(kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.labels}{"\n"}{end}')" || return 1
  printf '%s\n' "$l" | awk '/bookstore\.example\.com\//'
}
LEFT=0; LEFT_NS=0; LEFT_LB=0
dem LEFT kubectl get namespace,priorityclass,storageclass,persistentvolume -l "$SEL" --no-headers
dem LEFT_NS kubectl get all -A -l "$SEL" --no-headers
dem LEFT_NS kubectl get configmaps,secrets,pvc -A -l "$SEL" --no-headers
dem LEFT_LB node_labels
if [ "$LEFT" -eq 0 ] && [ "$LEFT_NS" -eq 0 ] && [ "$LEFT_LB" -eq 0 ]; then
  echo 'teardown: sach'
else
  echo "teardown: con $LEFT object cap cluster, $LEFT_NS object co label du an, $LEFT_LB node mang label du an"; exit 1
fi
EOF

chmod +x ~/bookstore/scripts/deploy-ha-lab.sh ~/bookstore/scripts/teardown-ha-lab.sh
```

Hai điều trong `deploy-ha-lab.sh` mà overlay của bạn phải chiều theo:

- `kubectl apply -l` lọc **object trong file** theo label: pha 1 bỏ mọi thứ mang
  `app.kubernetes.io/component=migrate`, pha 2 chỉ áp chúng. Job `db-migrate` mang label đó theo
  [hợp đồng](DU-AN-BOOKSTORE.md#41-tên-và-label); nếu tag `lab-8a` của bạn áp migration ngoài
  `k8s/app` thì đưa file đó vào `resources` của overlay.
- Script dừng nếu render còn Ingress, PVC, StorageClass hay `volumeClaimTemplates` — overlay phải bỏ
  chúng, không phải script.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-overlay-ha-lab.md`:

1. Vì sao overlay, mà không phải sửa `k8s/app` có điều kiện, hay một nhánh git riêng cho cluster HA?
2. Với từng thứ bị bỏ — Ingress, PVC, StorageClass, claim template — nếu **để nguyên** thì chuyện gì
   xảy ra trên cluster HA? Thứ nào hỏng ồn ào, thứ nào hỏng im lặng?
3. NetworkPolicy: bạn giữ hay bỏ? Nếu giữ, ai đọc manifest sẽ hiểu sai điều gì? Nếu một ngày cluster
   HA đổi sang CNI có thực thi policy, policy `traefik → web` của Lab 5b sẽ làm gì với lưu lượng
   NodePort?
4. Overlay chạy dữ liệu trên `emptyDir`. Vì sao chấp nhận được trong môi trường **đo**, và vì sao
   không được dùng overlay này cho dữ liệu thật? Có phải đây là lùi `TD-04`, `TD-08` không?
5. `services.nodeports: 0` của Lab 7b bảo vệ quyết định nào? Vì sao vá đúng một khóa thay vì bỏ cả
   quota khỏi overlay — mỗi cách giữ lại và mất đi điều gì?
6. `deploy-ha-lab.sh` áp hai pha. Nếu áp migration cùng lúc với `db` thì chuyện gì có thể xảy ra?

<details>
<summary>Gợi ý</summary>

1. Base là thư mục: `resources: [../../app]`. File lẻ ngoài thư mục overlay cũng vào được
   `resources` khi render bằng `--load-restrictor=LoadRestrictionsNone`.
2. Xóa hẳn một object của base: một patch StrategicMerge có dòng `$patch: delete`, kèm `target`
   chọn theo `kind` (và `name` nếu cần). Với `target`, tên trong thân patch không quan trọng.
3. Xóa một trường: trong patch StrategicMerge, đặt trường đó là `null`. Volume trong Pod template
   được gộp theo `name`, nên patch thêm một volume cùng tên với claim template là thay được nó. Ghim
   node của `db` và của Pod `db-backup` (Lab 6a, 7a) nằm ở `nodeSelector` hoặc `affinity`.
4. Service: patch đổi `type` và thêm `nodePort` cho đúng cổng có `port: 80` — danh sách `ports` được
   gộp theo `port`. Quota: `spec.hard` là một map, patch StrategicMerge chỉ đổi đúng khóa bạn ghi.
5. Xem trước kết quả mà không đụng cluster:
   `kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/overlays/ha-lab | grep '^kind:' | sort | uniq -c`.
   Base cần `secrets/db.env` và `secrets/tls.*` có sẵn — chạy `deploy-ha-lab.sh` một lần là có.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
source ~/lab-work/8b/bien.sh
cd "$REPO"
BAD=0
for F in k8s/overlays/ha-lab/kustomization.yaml scripts/deploy-ha-lab.sh scripts/teardown-ha-lab.sh tests/smoke-ha-lab.sh; do
  test -f "$F" || { echo "FAIL: thieu $F"; BAD=1; }
done
test "$BAD" -eq 0 && echo 'PASS: du file cua overlay va ba script ha-lab'
git diff --quiet lab-8a -- k8s/app k8s/cluster scripts/deploy.sh scripts/teardown.sh tests/smoke.sh \
  && echo 'PASS: k8s/app, k8s/cluster va ba script chuoi chinh khong doi so voi lab-8a'

./scripts/teardown-ha-lab.sh > "$EV/b7-teardown.txt" 2>&1
grep -q 'teardown: sach' "$EV/b7-teardown.txt" && echo 'PASS: teardown-ha-lab go sach du an'
./scripts/deploy-ha-lab.sh > "$EV/b7-deploy-1.txt" 2>&1 && echo 'PASS: deploy-ha-lab lan 1 thanh cong'
./scripts/deploy-ha-lab.sh > "$EV/b7-deploy-2.txt" 2>&1 && echo 'PASS: deploy-ha-lab lan 2 thanh cong'

# Render nam o lab-work vi chua Secret; evidence chi giu danh sach kind
kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/overlays/ha-lab > "$WK/b7-render.yaml"
grep '^kind:' "$WK/b7-render.yaml" | sort | uniq -c | tee "$EV/b7-kinds.txt"
grep -Eq '^kind: (Ingress|PersistentVolumeClaim|StorageClass)$' "$WK/b7-render.yaml" \
  && echo 'FAIL: render con Ingress, PVC hoac StorageClass' || echo 'PASS: render khong co Ingress, PVC, StorageClass'
grep -q 'volumeClaimTemplates:' "$WK/b7-render.yaml" \
  && echo 'FAIL: db con volumeClaimTemplates' || echo 'PASS: db khong con volumeClaimTemplates'
grep -Eq 'lab-k8s-|bookstore\.example\.com/data' "$WK/b7-render.yaml" \
  && echo 'FAIL: con ghim node cua chuoi chinh' || echo 'PASS: khong ghim node nao cua chuoi chinh'
NPQ="$(kubectl -n bookstore get resourcequota bookstore-quota -o jsonpath='{.spec.hard.services\.nodeports}' 2>/dev/null)"
test "${NPQ:-0}" -ge 1 && echo "PASS: bookstore-quota con hieu luc va cho phep NodePort (services.nodeports=$NPQ)"

./tests/smoke-ha-lab.sh > "$EV/b7-smoke.txt" 2>&1; echo "smoke exit=$?"
cat "$EV/b7-smoke.txt"
grep -q '^FAIL:' "$EV/b7-smoke.txt" && echo 'FAIL: smoke-ha-lab con loi' || echo 'PASS: smoke-ha-lab khong co dong FAIL'
grep -i -A6 'ha-lab' README.md > "$EV/b7-readme.txt"
{ grep -q 'scripts/deploy-ha-lab.sh' "$EV/b7-readme.txt" && grep -q 'scripts/teardown-ha-lab.sh' "$EV/b7-readme.txt" \
  && grep -q 'tests/smoke-ha-lab.sh' "$EV/b7-readme.txt"; } && echo 'PASS: README.md liet ke ba lenh cua bo VM HA'
{ grep -qi 'networkpolicy' docs/adr/*-overlay-ha-lab.md && grep -qi 'quota' docs/adr/*-overlay-ha-lab.md; } 2>/dev/null \
  && echo 'PASS: ADR overlay ghi quyet dinh ve NetworkPolicy va quota'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Lời giải dựa trên bố cục repo ở [mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint) và lời giải
các sprint trước: PriorityClass ở `k8s/cluster/priorityclass-critical.yaml` (3a) và
`k8s/cluster/priorityclass-batch.yaml` (7a); volume dữ liệu của `db` tên `data` (claim template
`data`, PVC `data-db-0` — 6a); volume bản dump trong CronJob tên `backup`, Pod backup ghim node để
cùng chỗ PV (6a); quota `bookstore-quota` có `services.nodeports: "0"` (7b). Tên ở tag `lab-8a` của
bạn khác thì sửa theo — patch theo `path` báo lỗi rõ khi không tìm thấy đích.

```yaml
# k8s/overlays/ha-lab/kustomization.yaml
# Bookstore trên cluster HA của bộ VM lab-ha-* (Lab 8b, 8c). Base k8s/app ở tag lab-8a, không sửa.
# Chỉ render qua scripts/deploy-ha-lab.sh (kubectl kustomize --load-restrictor=LoadRestrictionsNone):
# base đọc secrets/ ở gốc repo, overlay lấy hai PriorityClass từ k8s/cluster/ — đều ngoài thư mục này.
resources:
- ../../app
- ../../cluster/priorityclass-critical.yaml
- ../../cluster/priorityclass-batch.yaml

patches:
# Không ingress controller: Ingress không ai phục vụ; Secret TLS chỉ Ingress dùng.
- target: {kind: Ingress}
  patch: |-
    $patch: delete
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata: {name: bat-ky}
- target: {kind: Secret, name: bookstore-tls}
  patch: |-
    $patch: delete
    apiVersion: v1
    kind: Secret
    metadata: {name: bookstore-tls}
# Không StorageClass hay provisioner: PVC sẽ Pending mãi.
- target: {kind: PersistentVolumeClaim}
  patch: |-
    $patch: delete
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata: {name: bat-ky}
# CNI của cluster HA không thực thi NetworkPolicy — quyết định ở ADR overlay: bỏ.
- target: {kind: NetworkPolicy}
  patch: |-
    $patch: delete
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata: {name: bat-ky}
# Dữ liệu tạm, không ghim node của chuỗi chính, NodePort cho khách ngoài và quota cho phép nó.
- path: db-emptydir.yaml
- path: backup-emptydir.yaml
- path: web-nodeport.yaml
- path: quota-nodeport.yaml
```

```yaml
# k8s/overlays/ha-lab/db-emptydir.yaml
# db trên cluster HA: dữ liệu trên emptyDir (môi trường đo, không phải dữ liệu thật); bỏ mọi ghim
# node của chuỗi chính (nodeSelector theo hostname ở 3a–6b, node affinity theo label ở 7a).
apiVersion: apps/v1
kind: StatefulSet
metadata: {name: db, namespace: bookstore}
spec:
  volumeClaimTemplates: null
  template:
    spec:
      affinity:
        nodeAffinity: null
      nodeSelector: null
      volumes:
      - name: data
        emptyDir: {}
```

```yaml
# k8s/overlays/ha-lab/backup-emptydir.yaml — bản dump vào emptyDir thay PVC backup-store; Pod backup
# không còn phải đi theo PV cục bộ trên lab-k8s-worker2.
apiVersion: batch/v1
kind: CronJob
metadata: {name: db-backup, namespace: bookstore}
spec:
  jobTemplate:
    spec:
      template:
        spec:
          affinity:
            nodeAffinity: null
          nodeSelector: null
          volumes:
          - name: backup
            persistentVolumeClaim: null
            emptyDir: {}
```

```yaml
# k8s/overlays/ha-lab/web-nodeport.yaml — khách ngoài cluster vào qua NodePort như Lab 5a
apiVersion: v1
kind: Service
metadata: {name: web, namespace: bookstore}
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30080
```

```yaml
# k8s/overlays/ha-lab/quota-nodeport.yaml — Lab 7b cấm NodePort vì người dùng vào qua Ingress (5b).
# Cluster HA không có Ingress: mở đúng một NodePort cho web, giữ nguyên mọi trần khác.
apiVersion: v1
kind: ResourceQuota
metadata: {name: bookstore-quota, namespace: bookstore}
spec:
  hard:
    services.nodeports: "1"
```

Thêm vào `README.md` ở gốc repo — Lab 8c đọc ba lệnh từ đây bằng `grep -A6 'ha-lab'`:

```markdown
## Bộ VM HA — overlay ha-lab (Lab 8b, 8c)

Chạy từ gốc repo, chỉ trên cluster của bộ VM lab-ha-*; chuỗi chính vẫn dùng ba lệnh ở trên.
- Triển khai:    ./scripts/deploy-ha-lab.sh
- Gỡ bỏ:         ./scripts/teardown-ha-lab.sh
- Kiểm thử khói: ./tests/smoke-ha-lab.sh
- Đo sẵn sàng từ máy host: scripts/do-san-sang.ps1 (CSV ms,app,api)
```

Ý chính cho ADR overlay:

1. Base là **một** nguồn sự thật; overlay chỉ chứa **khác biệt** của một môi trường (bài 322: một
   base, nhiều overlay; base không biết gì về overlay). Sửa `k8s/app` có điều kiện làm chuỗi chính phụ
   thuộc môi trường thử; nhánh git riêng trôi dần khỏi `main` sau mỗi sprint.
2. Ingress: API chấp nhận, không ai phục vụ — hỏng **im lặng**. PVC không StorageClass: `Pending` mãi;
   claim template của `db` cũng vậy, kéo `db-0` `Pending` — hỏng **ồn ào**, `deploy-ha-lab.sh` treo ở
   `rollout status statefulset/db`. StorageClass: tạo được nhưng provisioner không tồn tại — im lặng.
   Ghim node `lab-k8s-worker2`: Pod `Pending` mãi vì node đó không tồn tại — ồn ào với `db`, im lặng
   với `db-backup` cho tới khi lần backup đầu không chạy.
3. Lời giải chọn **bỏ** NetworkPolicy: bài 84 nói policy chỉ có tác dụng khi plugin mạng hỗ trợ; để
   lại policy không thực thi là để lại một lời hứa sai. Khi CNI đổi sang loại có thực thi, policy chỉ
   cho Pod namespace `traefik` vào `web` sẽ chặn lưu lượng NodePort — script đo ở B8 thấy ngay. Giữ
   cũng đúng nếu ghi rõ "không thực thi".
4. Môi trường đo dựng lại từ đầu mỗi lần triển khai; `deploy-ha-lab.sh` chạy migration ở mỗi lần.
   **Không** phải lùi nợ của hệ thống: chuỗi chính vẫn dùng PVC; overlay này không được trỏ vào dữ
   liệu thật.
5. `services.nodeports: 0` biến quyết định "vào từ ngoài chỉ qua Ingress" của Lab 5b thành luật. Trên
   cluster HA tiền đề đó không còn, nên **vá đúng một khóa** thành `1`; các trần còn lại — CPU, bộ
   nhớ, số Pod, số Job — vẫn bảo vệ hai worker 2 GB khỏi một lần scale nhầm. Bỏ cả quota là mất luôn
   những bảo vệ đó và smoke của Lab 7b không còn gì để kiểm.
6. Pod migration chạy trước khi PostgreSQL nhận kết nối thì thất bại; hết `backoffLimit` là Job
   `Failed` và không tự chạy lại (Lab 4b). Init container chờ DB của Lab 5a đã che phần lớn rủi ro;
   hai pha là lớp bảo vệ thứ hai, rẻ.

</details>

## B8. [BS-8B-08] Đo sẵn sàng từ máy host

**Loại:** Tính năng.

**Bối cảnh:** [mục 3.7 của file dự án](DU-AN-BOOKSTORE.md#37-chịu-lỗi-và-vận-hành) hứa: mất một control
plane thì API vẫn trả lời; mất quorum thì workload đang chạy vẫn phục vụ. Lời hứa đó phải đo được
**từ chỗ khách hàng đứng** — ngoài cluster, không qua `kubectl` — liên tục trong lúc sự cố diễn ra, và
theo **hai đường tách biệt**: ứng dụng và API. Lab 8c đo lại đúng hai đường đó trên topology external;
ADR chọn topology chỉ so được hai bộ số nếu cách đo giống hệt.

**Yêu cầu:**

1. `scripts/do-san-sang.ps1` trong repo — chép nguyên văn khối bên dưới. Chạy trên máy host bằng
   Windows PowerShell và `curl.exe` có sẵn; không cài công cụ nào.
2. Mỗi nửa giây script gọi hai đường: **ứng dụng** — `/api/books` qua NodePort `30080` của `lab-ha-4`,
   không đi qua API server; **API** — `/readyz` của `kube-apiserver` qua load balancer, không đi qua
   ứng dụng. Mỗi mẫu một dòng CSV `ms,app,api`: mốc thời gian Unix tính bằng mili giây, mã HTTP của
   từng đường, `000` là không kết nối được trong 2 giây. Đây đúng định dạng của Lab 8c.
3. Tỷ lệ sẵn sàng tính theo **thời gian**, không theo số mẫu; kèm gián đoạn liên tục dài nhất của
   từng đường.
4. Đo **đường nền** 120 giây khi cluster lành: cả hai đường ≥ 99%. File đo chép sang
   `~/lab-evidence/8b/` trên `lab-ha-1`.
5. Trước diễn tập đầu tiên: commit phần việc đang làm và đóng gói repo ra máy host
   (`bookstore-lab-8b-wip.bundle`) — đường lui duy nhất nếu phải restore `8x-vm-ready`
   ([mục 2.3](#23-trình-tự-an-toàn-khi-tắt-và-bật-máy)).

**Ràng buộc:** script chỉ chứa ký tự ASCII — Windows PowerShell đọc file không BOM theo code page của
máy, chữ có dấu trong chuỗi sẽ vỡ. Không đổi chính sách thực thi của máy host; chạy tiến trình con
với `-ExecutionPolicy Bypass`.

**Đọc lại:** [mục 4.2 của file dự án](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe);
bài [82](../82-service-vi.md#type-nodeport) — NodePort mở trên mọi node; bài
[22](../22-architecture-vi.md#kube-proxy) — kube-proxy giữ quy tắc chuyển tiếp trên chính node; bài
[08](../08-high-availability-vi.md#tạo-bộ-cân-bằng-tải-cho-kube-apiserver-create-load-balancer-for-kube-apiserver)
— client của API đi qua load balancer.

**Bàn giao:** `scripts/do-san-sang.ps1`; `docs/adr/NNNN-do-san-sang.md`; `~/lab-evidence/8b/do-nen.csv`;
bundle WIP trên máy host.

**Nghiệm thu:**

- `PASS:` script trong repo chỉ có ký tự ASCII; file đo đúng định dạng `ms,app,api`.
- `PASS:` đường nền: ít nhất 100 mẫu; ứng dụng và API đều ≥ 99%.
- `PASS:` repo không còn thay đổi chưa commit; bundle WIP nằm trên máy host.

Script đo, chép nguyên văn:

```bash
cat > ~/bookstore/scripts/do-san-sang.ps1 <<'EOF'
<#
  do-san-sang.ps1 - du an Bookstore, Lab 8b (dung lai o Lab 8c).
  Do san sang tu may host, ngoai cluster, theo HAI duong doc lap, moi nua giay:
    app: GET AppUrl (mac dinh /api/books qua NodePort 30080 cua lab-ha-4) - KHONG qua API server.
    api: GET ApiUrl (mac dinh /readyz cua kube-apiserver qua load balancer) - KHONG qua ung dung.
  Moi mau la mot dong CSV "ms,app,api": moc thoi gian Unix (mili giay) va ma HTTP cua tung duong;
  000 = khong ket noi duoc trong 2 giay. Lab 8c dung lai chinh script nay.
  Ty le san sang tinh theo THOI GIAN, khong theo so mau.
  Chay tren may host (Windows PowerShell 5.1, curl.exe co san tu Windows 10):
    powershell -NoProfile -ExecutionPolicy Bypass -File .\do-san-sang.ps1 -Ten nen -Giay 120
  File chi chua ky tu ASCII: PowerShell 5.1 doc file khong BOM theo code page cua may.
#>
param(
  [Parameter(Mandatory = $true)][ValidatePattern('^[a-z0-9-]+$')][string]$Ten,
  [ValidateRange(10, 3600)][int]$Giay = 300,
  [string]$AppUrl = 'http://192.168.100.234:30080/api/books',
  [string]$ApiUrl = 'https://192.168.100.230:6443/readyz',
  [string]$Thu = (Join-Path $HOME 'lab-evidence\8b')
)
$ErrorActionPreference = 'Stop'
if (-not (Get-Command curl.exe -ErrorAction SilentlyContinue)) { throw 'khong tim thay curl.exe' }
New-Item -ItemType Directory -Force -Path $Thu | Out-Null
$csv = Join-Path $Thu ('do-' + $Ten + '.csv')
Set-Content -Path $csv -Value 'ms,app,api' -Encoding Ascii
$het = (Get-Date).AddSeconds($Giay)
while ((Get-Date) -lt $het) {
  $ms  = [DateTimeOffset]::Now.ToUnixTimeMilliseconds()
  $app = & curl.exe -s -o NUL -w '%{http_code}' --max-time 2 --noproxy '*' $AppUrl
  $api = & curl.exe -sk -o NUL -w '%{http_code}' --max-time 2 --noproxy '*' $ApiUrl
  Add-Content -Path $csv -Value ('{0},{1},{2}' -f $ms, $app, $api) -Encoding Ascii
  Write-Host ('{0:HH:mm:ss}  app={1}  api={2}' -f (Get-Date), $app, $api)
  Start-Sleep -Milliseconds 500
}
# Moi mau chiu trach nhiem cho khoang thoi gian toi mau ke tiep.
$r = @(Import-Csv -Path $csv)
$tong = [double]$r[-1].ms - [double]$r[0].ms
$appMat = 0.0; $apiMat = 0.0; $appRun = 0.0; $apiRun = 0.0; $appMax = 0.0; $apiMax = 0.0
for ($i = 0; $i -lt $r.Count - 1; $i++) {
  $dt = [double]$r[$i + 1].ms - [double]$r[$i].ms
  if ($r[$i].app -ne '200') { $appMat += $dt; $appRun += $dt; if ($appRun -gt $appMax) { $appMax = $appRun } }
  else { $appRun = 0.0 }
  if ($r[$i].api -ne '200') { $apiMat += $dt; $apiRun += $dt; if ($apiRun -gt $apiMax) { $apiMax = $apiRun } }
  else { $apiRun = 0.0 }
}
if ($tong -gt 0) {
  $pApp = 100 * (1 - $appMat / $tong)
  $pApi = 100 * (1 - $apiMat / $tong)
  $fmt = 'KET QUA {0}: {1} mau, {2:N0} giay | app {3:N1}% | api {4:N1}% | gian doan dai nhat: app {5:N0}s, api {6:N0}s'
  $fmt -f $Ten, $r.Count, ($tong / 1000), $pApp, $pApi, ($appMax / 1000), ($apiMax / 1000)
}
'File: ' + $csv
EOF
```

Script ghi file `do-<Ten>.csv` vào thư mục `-Thu`; Lab 8c chạy lại chính script này với `-Thu` trỏ
sang `lab-evidence\8c` và `-AppUrl` của cluster mới, nên số đo của hai lab so được với nhau. Phía `lab-ha-1`, hàm `tom_tat` trong `bien.sh` đọc file và đặt các biến `APP`, `API`,
`APP_MAX`, `API_MAX`, `CUOI_APP`, `CUOI_API` — cùng công thức với Lab 8c.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-do-san-sang.md`:

1. Vẽ đường đi của một request `/api/books` từ máy host tới `db-0`, và của một request `/readyz`. Chặng
   nào là control plane? Vì sao hai đường phải tách hẳn nhau?
2. Vì sao đo từ máy host chứ không từ `lab-ha-1`, và không bằng `kubectl`?
3. Vì sao tỷ lệ tính theo **thời gian** chứ không theo số mẫu? Lúc hỏng, một mẫu tốn bao lâu?
4. Đường ứng dụng chỉ gõ `lab-ha-4`. Diễn tập nào của sprint có thể cho kết quả khác nếu gõ `lab-ha-5`?
5. Đội định nghĩa "đạt" thế nào cho một diễn tập: tỷ lệ của cả cửa sổ, gián đoạn dài nhất, hay việc
   mười mẫu cuối đều `200`? Mỗi tiêu chí trả lời câu hỏi gì?

<details>
<summary>Gợi ý</summary>

- Chép script từ `lab-ha-1` về đúng chỗ biến `$probe` của B0.1 trỏ tới, bằng `scp`.
- `Start-Probe <Ten> <Giay>` mở cửa sổ PowerShell mới chạy script; `Send-Probe <Ten>` chép
  `do-<Ten>.csv` sang `lab-ha-1`. Đường nền chạy thẳng trong cửa sổ hiện tại cũng được.
- `/readyz` gọi được không cần đăng nhập vì kubeadm giữ mặc định cho phép truy cập ẩn danh vào các
  endpoint sức khỏe; `-k` vì certificate của apiserver do CA của cluster ký, máy host không tin CA đó.
- Bundle WIP: giống [mục 5.4](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm), tên
  `bookstore-lab-8b-wip.bundle`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

Chạy trên **máy host**:

```powershell
scp "$($haUser)@192.168.100.231:bookstore/scripts/do-san-sang.ps1" "$probe"
powershell -NoProfile -ExecutionPolicy Bypass -File "$probe" -Ten nen -Giay 120
Send-Probe nen
$w = Join-Path $backup 'bookstore-lab-8b-wip.bundle'
if ((Test-Path $w) -and ((Get-Item $w).Length -gt 0)) { "PASS: co $w" } else { "FAIL: thieu $w" }
```

Chạy trên `lab-ha-1`:

```bash
source ~/lab-work/8b/bien.sh
cd "$REPO"
if LC_ALL=C grep -qP '[^\x00-\x7F]' scripts/do-san-sang.ps1; then
  echo 'FAIL: do-san-sang.ps1 co ky tu ngoai ASCII'
else
  echo 'PASS: do-san-sang.ps1 chi chua ky tu ASCII'
fi
head -n 1 "$EV/do-nen.csv" | tr -d '\r' | grep -qx 'ms,app,api' && echo 'PASS: file do dung dinh dang ms,app,api'
tom_tat "$EV/do-nen.csv"
test "$MAU" -ge 100 && echo "PASS: du $MAU mau"
ge "$APP" 99 && ge "$API" 99 && echo 'PASS: do nen tu may host — ung dung va API deu >= 99%'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
```

Đường nền dưới 99% thì **chưa** đi tiếp: diễn tập so với một đường nền bẩn không chứng minh được gì.
Xem [mục 4](#4-troubleshooting-của-lab-này).

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
# Chay tren lab-ha-1
cd ~/bookstore
git add -A && git commit -m 'lab-8b: overlay ha-lab, script do san sang (WIP truoc dien tap)'
git bundle create ~/bookstore-lab-8b-wip.bundle --all && git bundle verify ~/bookstore-lab-8b-wip.bundle
```

```powershell
# Chay tren may host
scp "$($haUser)@192.168.100.231:bookstore-lab-8b-wip.bundle" "$backup\"
```

Ý chính cho ADR đo sẵn sàng:

1. Ứng dụng: máy host → `lab-ha-4:30080` → quy tắc NodePort của kube-proxy trên node đó → Pod `web` →
   Service `api` → Pod `api` → `db-0.db` → PostgreSQL. Không chặng nào là apiserver, etcd, scheduler
   hay controller-manager. API: máy host → `lab-ha-lb:6443` → HAProxy → một `kube-apiserver` → etcd.
   Tách hai đường mới thấy được điều sprint muốn chứng minh: control plane chết mà dịch vụ vẫn sống.
2. `kubectl` đi qua control plane — dùng nó để đo là đo chính thứ đang bị phá. `lab-ha-1` là một node
   của cluster; khách thật đứng ngoài.
3. Lúc lành một mẫu mất vài chục mili giây, lúc hỏng mất tới 2 giây mỗi đường. Đếm mẫu thì khoảng
   hỏng chiếm ít mẫu hơn thời gian thật của nó — số đẹp hơn thực tế.
4. ⭐ của B11 (tắt `lab-ha-5`): gõ `lab-ha-5` thì hỏng 100% vì chính node đó tắt; gõ `lab-ha-4` thì đo
   được hậu quả tinh tế hơn — quy tắc của kube-proxy còn trỏ tới Pod đã chết.
5. Tỷ lệ cả cửa sổ trả lời "khách mất bao nhiêu"; gián đoạn dài nhất trả lời "một khách gặp lỗi liên
   tục bao lâu"; mười mẫu cuối `200` trả lời "hệ thống đã hồi phục chưa khi kết thúc đo". Diễn tập
   của sprint dùng cả ba, cùng ngưỡng với Lab 8c.

</details>

## B9. [BS-8B-09] Diễn tập: mất một control plane node

**Loại:** Diễn tập sự cố — tắt `lab-ha-3`.

**Bối cảnh:** kịch bản `mat-mot-cp` của bảng dự đoán, và dòng *Mất một control plane (HA)* ở
[mục 3.7 của file dự án](DU-AN-BOOKSTORE.md#37-chịu-lỗi-và-vận-hành). Đây là lý do cả sprint tồn tại:
cluster phải sống sót khi mất một máy control plane, và đội phải thấy tận mắt cái giá riêng của
stacked — một máy tắt là hai thứ mất.

Bốn diễn tập B9–B12 cùng một nhịp: mở **cửa sổ đo** trên máy host → gây sự cố → chứng minh trạng thái
trong sự cố từ `lab-ha-1` → khôi phục → chờ cluster lành, smoke → để cửa sổ đo chạy hết **sau** khi
hồi phục → chép file đo về `lab-ha-1` → `tom_tat` → postmortem và cột *đo được* của ADR. Một cửa sổ đo
phủ trọn một kịch bản, đúng cách Lab 8c đo, để hai bộ số đặt cạnh nhau được.

**Yêu cầu:**

1. Cửa sổ đo `mat-mot-cp` bắt đầu **trước** khi tắt máy và kết thúc **sau** khi cluster đã lành.
2. Tắt `lab-ha-3` đúng quy trình của [mục 2.3](#23-trình-tự-an-toàn-khi-tắt-và-bật-máy).
3. Trong lúc `lab-ha-3` tắt, chứng minh: load balancer loại nó khỏi danh sách đích; API vẫn **đọc và
   ghi** được; `Ready` của `lab-ha-3` không còn `True`; mất **đồng thời** một `kube-apiserver` và
   một etcd member khỏe, trong khi danh sách member vẫn đủ ba.
4. Bật lại `lab-ha-3`; cluster lành hẳn mà không làm gì thêm ngoài bật máy; smoke PASS.
5. Số đo cả cửa sổ: ứng dụng ≥ 99%, API ≥ 90%, mười mẫu cuối của cả hai đường đều `200`.
6. `docs/postmortem/8b-mat-control-plane.md`, mục `## mat-mot-cp`: diễn biến có mốc giờ, số đo, dự
   đoán so với thực tế, cơ chế, bài học. Điền ba cột *đo được* của dòng `mat-mot-cp` trong ADR.

**Ràng buộc:** không tắt `lab-ha-1` (giữ repo, kubeconfig) và `lab-ha-2` (B10 cần nó lành). Không
`vmrun … stop … hard`. Bundle WIP của B8 phải nằm trên máy host trước khi tắt máy.

**Đọc lại:** bài [06](../06-ha-topology-vi.md#stacked-etcd-topology) — hỏng hóc theo cặp, tối thiểu
ba control plane node; bài [08](../08-high-availability-vi.md#tạo-bộ-cân-bằng-tải-cho-kube-apiserver-create-load-balancer-for-kube-apiserver)
— LB phân phối tới node **khỏe mạnh**; bài [22](../22-architecture-vi.md#nhieu-ban-apiserver) — quorum
Raft của etcd.

**Bàn giao:** postmortem mục `mat-mot-cp`; ADR topology; `~/lab-evidence/8b/b9-*`,
`~/lab-evidence/8b/do-mat-mot-cp.csv`.

**Nghiệm thu:**

- `PASS:` trong sự cố: hai backend `UP`; API đọc và ghi được; `lab-ha-3` không `Ready`; hai apiserver
  trả lời; hai trên ba member khỏe, danh sách vẫn ba.
- `PASS:` sau khi bật lại: ba member khỏe, ba backend `UP`, ba apiserver trả lời; smoke PASS.
- `PASS:` cửa sổ đo: ứng dụng ≥ 99%, API ≥ 90%, kết thúc bằng mười mẫu `200` ở cả hai đường.
- `PASS:` postmortem có mục `mat-mot-cp`; dòng `mat-mot-cp` của ADR đủ sáu ô.

**Câu hỏi thiết kế** — trả lời trong postmortem, mục `## mat-mot-cp`:

1. Hai mất mát cùng lúc là gì, bài 06 gọi hiện tượng đó là gì? Chỉ ra từng cái trong evidence.
2. Vì sao cluster vẫn **ghi** được? Trả lời bằng số member và đa số.
3. `member list` vẫn ba dòng dù một máy đã tắt. Vì sao?
4. Đường API có lỗi trong vài giây đầu không? Vì sao — nghĩ tới `inter` và `fall` của health check ở
   B3. Đường ứng dụng có lỗi không, và thành phần nào chạy trên `lab-ha-3` có thể nằm trên đường đi
   của nó?
5. Cùng lần tắt máy này trong topology external sẽ mất những gì?

<details>
<summary>Gợi ý</summary>

- `Start-Probe <Ten> <Giay>` của B0.1 mở cửa sổ đo riêng. Cửa sổ phải đủ dài để phủ cả lúc tắt, lúc
  node controller đổi trạng thái node (phụ thuộc cấu hình), lúc bật lại và hồi phục — 420 giây là
  điểm khởi đầu hợp lý. Kết thúc trước khi hồi phục thì đo lại.
- Máy đã tắt hẳn khi `& $vmrun -T ws list` trên máy host không còn đường dẫn `.vmx` của nó.
- Đọc số đo trên `lab-ha-1`: `tom_tat ~/lab-evidence/8b/do-<Ten>.csv`, rồi dùng `APP`, `API`,
  `CUOI_APP`, `CUOI_API`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script này chính là cuộc diễn tập</summary>

Phần 1 — trên `lab-ha-1`, trước sự cố; rồi trên **máy host** mở cửa sổ đo:

```bash
source ~/lab-work/8b/bien.sh
kubectl get namespace lab-8b >/dev/null 2>&1 || kubectl create namespace lab-8b
api_probe b9-truoc; direct_probe b9-truoc
{ test "$API_READ" = 'ok' && test "$API_WRITE" = 'ok' && test "$DIRECT_OK" -eq 3; } \
  && echo 'PASS: truoc su co, API doc ghi duoc va ca ba apiserver tra loi'
```

```powershell
Start-Probe mat-mot-cp 420
```

Phần 2 — chờ khoảng 20 giây cho cửa sổ đo có đoạn nền. Trên `lab-ha-3`: `sudo shutdown -h now`. Trên
**máy host**:

```powershell
$cp3 = $haVmx | Where-Object { $_ -like '*lab-ha-3*' }
if ((& $vmrun -T ws list) -notcontains $cp3) { 'PASS: lab-ha-3 da Powered off' } else { 'CHUA TAT: doi them' }
```

Phần 3 — trên `lab-ha-1`, trong sự cố:

```bash
source ~/lab-work/8b/bien.sh
wait_lb_up 2; lbstat | tee "$EV/b9-lb.txt"
test "$UPN" -eq 2 && echo 'PASS: health check TCP da loai lab-ha-3 khoi danh sach dich'
api_probe b9-mot-node-chet
test "$API_READ" = 'ok' && test "$API_WRITE" = 'ok' && echo 'PASS: mat mot control plane node, API van doc va GHI duoc'
for i in $(seq 1 90); do
  ST="$(kubectl get node "$CP3" --request-timeout=20s -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null)"
  test "$ST" != 'True' && break; sleep 5
done
kubectl get nodes -o wide | tee "$EV/b9-nodes.txt"
test "$ST" != 'True' && echo 'PASS: node controller da doi Ready cua lab-ha-3 khoi True'
direct_probe b9-mot-node-chet
HEALTHY="$(etcd_ok b9-etcd-health.txt)"
MEM_N="$(etcd1 --endpoints=https://127.0.0.1:2379 member list | wc -l)"
echo "apiserver tra loi=$DIRECT_OK | etcd khoe=$HEALTHY/$MEM_N" | tee "$EV/b9-hong-theo-cap.txt"
test "$DIRECT_OK" -eq 2 && echo 'PASS: mat DONG THOI mot instance kube-apiserver'
test "$HEALTHY" -eq 2 && test "$MEM_N" -eq 3 && echo 'PASS: mat DONG THOI mot etcd member — van trong danh sach nhung khong khoe'
```

Phần 4 — trên **máy host**, bật lại:

```powershell
& $vmrun -T ws start $cp3
if ((& $vmrun -T ws list) -contains $cp3) { 'PASS: lab-ha-3 dang chay' } else { 'FAIL: VM chua len' }
```

Trên `lab-ha-1`:

```bash
source ~/lab-work/8b/bien.sh
cho_lanh b9
"$REPO/tests/smoke-ha-lab.sh" > "$EV/b9-smoke-sau.txt" 2>&1 && echo 'PASS: smoke-ha-lab PASS sau khi hoi phuc'
```

Phần 5 — khi cửa sổ đo in dòng `KET QUA`, trên **máy host**: `Send-Probe mat-mot-cp`. Trên `lab-ha-1`:

```bash
source ~/lab-work/8b/bien.sh
tom_tat "$EV/do-mat-mot-cp.csv"
ge "$APP" 99 && ge "$API" 90 && test "$CUOI_APP" -eq 1 && test "$CUOI_API" -eq 1 \
  && echo 'PASS: mat mot control plane — ung dung >= 99%, API >= 90%, ket thuc ca hai duong 200'
ghi_chep mat-mot-cp
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

Khung một mục của `docs/postmortem/8b-mat-control-plane.md` — dùng cho cả bốn kịch bản:

```markdown
## mat-mot-cp

- Diễn biến: <giờ> mở cửa sổ đo; <giờ> shutdown lab-ha-3; <giờ> LB còn 2 backend UP; <giờ> node
  không Ready; <giờ> bật lại; <giờ> 3/3 etcd khỏe; <giờ> cửa sổ đo kết thúc.
- Số đo (do-mat-mot-cp.csv): app …%, API …%, gián đoạn dài nhất app …s, API …s.
- Dự đoán (B2): đọc được, ghi được, NodePort được. Thực tế: …
- Cơ chế: …
- Bài học: …
```

1. Số apiserver trả lời giảm 3 → 2 **và** số etcd member khỏe giảm 3 → 2 (`b9-hong-theo-cap.txt`).
   Bài 06: **hỏng hóc theo cặp** — topology gắn kết control plane và etcd trên cùng node, tính dự
   phòng suy giảm ở cả hai tầng cùng lúc.
2. Hai trên ba member — đa số của ba là hai — nên ghi vẫn được xác nhận; còn hai apiserver mà LB biết
   đường tới. Ba là số lẻ nhỏ nhất chịu được mất một.
3. Rời cluster là thao tác quản trị trên etcd; tắt máy chỉ làm member đó không trả lời.
4. Trước khi health check đủ `fall 2` lần thất bại, HAProxy vẫn có thể gửi kết nối mới tới
   `lab-ha-3` — vài giây lỗi trên đường API là bình thường, độ dài phụ thuộc `inter` và `fall`. Đường
   ứng dụng không đi qua `lab-ha-3`: `web`, `api`, `db-0` chạy trên worker, quy tắc NodePort nằm trên
   chính `lab-ha-4`. Ứng viên duy nhất là Pod CoreDNS từng nằm trên `lab-ha-3` sau B5 — phân giải tên
   giữa các tầng. Không lỗi nào là kết quả bình thường.
5. External: tắt một control plane node chỉ mất một apiserver, etcd vẫn ba trên ba; tắt một etcd host
   chỉ mất một member. Mỗi lần mất máy chỉ đánh vào **một** tầng — nội dung Lab 8c.

</details>

## B10. [BS-8B-10] Diễn tập: một `kube-apiserver` mất manifest

**Loại:** Diễn tập sự cố — trên `lab-ha-2`, không tắt máy.

**Bối cảnh:** kịch bản `mat-manifest-apiserver`. Checkpoint giai đoạn 8 yêu cầu xóa
`/etc/kubernetes/manifests/kube-apiserver.yaml` rồi khôi phục. Lab 8a làm phép này trên cluster một
control plane và mất luôn `kubectl`. Ở đây có load balancer: một apiserver chết **mà cluster vẫn phục
vụ**, và ranh giới sự cố hẹp hơn hẳn B9.

**Yêu cầu:**

1. Cửa sổ đo `mat-manifest-apiserver` phủ từ trước khi gỡ manifest tới sau khi khôi phục.
2. Trên `lab-ha-2`: ghi checksum, cất bản sao **ngoài** thư mục static Pod, xóa thật
   `kube-apiserver.yaml`. Kubelet dừng container apiserver; ba static Pod còn lại vẫn chạy.
3. Chứng minh: LB thấy `lab-ha-2` `DOWN`, hai backend `UP`; API đọc và ghi được; mirror Pod
   `kube-apiserver-lab-ha-2` biến khỏi API; node `lab-ha-2` **vẫn** `Ready`; ba etcd member **vẫn**
   khỏe.
4. Khôi phục manifest y hệt bản gốc; container và mirror Pod quay lại; cluster lành hẳn; smoke PASS.
5. Số đo cả cửa sổ: ứng dụng ≥ 99%, API ≥ 90%, mười mẫu cuối của cả hai đường đều `200`.
6. Postmortem mục `## mat-manifest-apiserver`; ADR dòng `mat-manifest-apiserver` đủ sáu ô.

**Ràng buộc:** không đụng `lab-ha-1`. Không chép manifest từ node khác: mỗi node có
`--advertise-address` và SAN riêng. Bản sao cất ở home của `lab-ha-2`, xóa sau khi khôi phục.

**Đọc lại:** bài [58](../58-static-pods-vi.md#mirror-pods) — mirror Pod; bài
[08](../08-high-availability-vi.md#tạo-bộ-cân-bằng-tải-cho-kube-apiserver-create-load-balancer-for-kube-apiserver)
— health check TCP; [Lab 8a](LAB-8A-DUNG-CLUSTER-BANG-KUBEADM.md), phần phá có kiểm soát static Pod
control plane.

**Bàn giao:** postmortem; ADR; `~/lab-evidence/8b/b10-*`, `~/lab-evidence/8b/do-mat-manifest-apiserver.csv`.

**Nghiệm thu:**

- `PASS:` kubelet dừng container `kube-apiserver` trên `lab-ha-2` vì manifest biến mất.
- `PASS:` `lab-ha-2` `DOWN` ở LB, hai backend `UP`; API đọc và ghi được; mirror Pod biến mất; node vẫn
  `Ready`; ba etcd member khỏe.
- `PASS:` manifest y hệt bản gốc; mirror Pod quay lại; cluster lành; smoke PASS.
- `PASS:` cửa sổ đo: ứng dụng ≥ 99%, API ≥ 90%, kết thúc bằng `200`; postmortem và ADR đã cập nhật.

**Câu hỏi thiết kế** — postmortem, mục `## mat-manifest-apiserver`:

1. Thành phần nào quyết định bỏ qua `lab-ha-2`, dựa trên dấu hiệu gì?
2. Ba thứ **không** hỏng theo là gì? So với B9, đây là một mất mát hay một cặp?
3. Cùng phép thử trên cluster một control plane khác ra sao, vì sao ở đó phải chẩn đoán bằng `crictl`
   và `journalctl`?
4. Health check của LB là TCP. Một apiserver vẫn **lắng nghe** `6443` nhưng không phục vụ được thì
   health check có nhận ra không? Giữ câu trả lời cho B11.
5. Lỡ xóa mà không có bản sao: khôi phục bằng gì, vì sao không chép từ `lab-ha-1`?

<details>
<summary>Gợi ý</summary>

- Kubelet theo dõi thư mục static Pod: file biến mất là Pod bị gỡ, file quay lại là Pod được dựng
  lại. Bản sao cất **trong** thư mục đó thì kubelet đọc luôn bản sao — cất ở `~`.
- Chờ container biến mất: vòng `sudo crictl ps --name kube-apiserver -q | wc -l` có số vòng tối đa.
- Sinh lại manifest khi không có bản sao: `kubeadm init phase control-plane apiserver` với đúng
  endpoint và địa chỉ quảng bá của **node đó**.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script này chính là cuộc diễn tập</summary>

Phần 1 — trên **máy host**: `Start-Probe mat-manifest-apiserver 300`, chờ khoảng 20 giây. Trên
`lab-ha-2`:

```bash
sudo sha256sum /etc/kubernetes/manifests/kube-apiserver.yaml | tee ~/b10-sha-truoc.txt
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml ~/kube-apiserver.yaml.bak
sudo rm /etc/kubernetes/manifests/kube-apiserver.yaml
for i in $(seq 1 60); do test "$(sudo crictl ps --name kube-apiserver -q | wc -l)" -eq 0 && break; sleep 5; done
test "$(sudo crictl ps --name kube-apiserver -q | wc -l)" -eq 0 \
  && echo 'PASS: kubelet da dung container kube-apiserver vi manifest bien mat'
sudo crictl ps | head -n 10
```

`crictl ps` vẫn liệt kê `etcd`, `kube-controller-manager`, `kube-scheduler` — một trong bốn static Pod
bị gỡ.

Phần 2 — trên `lab-ha-1`:

```bash
source ~/lab-work/8b/bien.sh
wait_lb_up 2; lbstat | tee "$EV/b10-lb.txt"
grep -q "^${CP2}=DOWN$" "$EV/b10-lb.txt" && test "$UPN" -eq 2 \
  && echo 'PASS: health check TCP loai lab-ha-2, hai backend con lai UP'
api_probe b10-apiserver-chet
MIRROR="$(kubectl -n kube-system get pod "kube-apiserver-$CP2" --ignore-not-found -o name 2>/dev/null | wc -l)"
NODE_RDY="$(kubectl get node "$CP2" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
HEALTHY="$(etcd_ok b10-etcd-health.txt)"
echo "mirror=$MIRROR | node $CP2 Ready=$NODE_RDY | etcd khoe=$HEALTHY" | tee "$EV/b10-ranh-gioi.txt"
test "$API_READ" = 'ok' && test "$API_WRITE" = 'ok' && echo 'PASS: mot kube-apiserver chet ma cluster van doc va ghi duoc'
test "$MIRROR" -eq 0 && echo 'PASS: mirror Pod cua apiserver do da bien mat khoi API'
test "$NODE_RDY" = 'True' && echo 'PASS: node lab-ha-2 VAN Ready — kubelet cua no van chay'
test "$HEALTHY" -eq 3 && echo 'PASS: ca ba etcd member VAN khoe manh'
```

Phần 3 — khôi phục, trên `lab-ha-2`:

```bash
sudo cp ~/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml
sudo sha256sum /etc/kubernetes/manifests/kube-apiserver.yaml | tee ~/b10-sha-sau.txt
diff <(awk '{print $1}' ~/b10-sha-truoc.txt) <(awk '{print $1}' ~/b10-sha-sau.txt) \
  && echo 'PASS: manifest khoi phuc y het ban goc' || echo 'FAIL: file khac ban goc — xem muc 4'
for i in $(seq 1 60); do test "$(sudo crictl ps --name kube-apiserver -q | wc -l)" -ge 1 && break; sleep 5; done
test "$(sudo crictl ps --name kube-apiserver -q | wc -l)" -ge 1 && echo 'PASS: kubelet da dung lai container tu manifest'
rm -f ~/kube-apiserver.yaml.bak ~/b10-sha-truoc.txt ~/b10-sha-sau.txt
```

Trên `lab-ha-1`:

```bash
source ~/lab-work/8b/bien.sh
cho_lanh b10
for i in $(seq 1 30); do
  test "$(kubectl -n kube-system get pod "kube-apiserver-$CP2" --ignore-not-found -o name | wc -l)" -eq 1 && break; sleep 5
done
test "$(kubectl -n kube-system get pod "kube-apiserver-$CP2" --ignore-not-found -o name | wc -l)" -eq 1 \
  && echo 'PASS: mirror Pod cua apiserver da xuat hien lai trong API'
"$REPO/tests/smoke-ha-lab.sh" > "$EV/b10-smoke-sau.txt" 2>&1 && echo 'PASS: smoke-ha-lab PASS sau khi khoi phuc'
```

Phần 4 — khi cửa sổ đo in `KET QUA`, trên **máy host**: `Send-Probe mat-manifest-apiserver`. Trên
`lab-ha-1`:

```bash
source ~/lab-work/8b/bien.sh
tom_tat "$EV/do-mat-manifest-apiserver.csv"
ge "$APP" 99 && ge "$API" 90 && test "$CUOI_APP" -eq 1 && test "$CUOI_API" -eq 1 \
  && echo 'PASS: mot apiserver chet — ung dung >= 99%, API >= 90%, ket thuc ca hai duong 200'
ghi_chep mat-manifest-apiserver
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

1. HAProxy: health check TCP tới `lab-ha-2:6443` không bắt tay được, sau đủ số lần thất bại
   (`fall 2`) thì loại backend khỏi danh sách đích — đúng câu "phân phối tới các node control plane
   **khỏe mạnh**" của bài 08.
2. Node vẫn `Ready` (kubelet không chết theo), etcd member trên máy đó vẫn khỏe (static Pod
   **khác**), controller-manager và scheduler trên máy đó vẫn chạy. **Một** mất mát, không phải cặp.
3. Một control plane: mất apiserver duy nhất là mất `kubectl`; mọi quan sát phải đi qua container
   runtime (`crictl`) và log của kubelet (`journalctl`) — bộ đôi mà bài 09 đẩy về.
4. Không. TCP check chỉ hỏi "có ai lắng nghe cổng không". Apiserver còn lắng nghe nhưng không đọc ghi
   được etcd vẫn là `UP` với LB — B11 cho thấy.
5. `sudo kubeadm init phase control-plane apiserver --control-plane-endpoint lab-ha-lb:6443
   --apiserver-advertise-address <IP của chính node>`; certificate trong `/etc/kubernetes/pki` vẫn
   còn nên không cần join lại. Manifest của `lab-ha-1` mang địa chỉ quảng bá của `lab-ha-1`.

</details>

## B11. [BS-8B-11] Diễn tập: mất quorum

**Loại:** Diễn tập sự cố — tắt `lab-ha-2` và `lab-ha-3`; ⭐ lặp lại kèm `lab-ha-5`.

**Bối cảnh:** kịch bản `mat-quorum` — bài học lõi của stacked etcd và lý do bài 06 nói tối thiểu ba.
Mục 3.7 của file dự án hứa: mất quorum thì workload **đang chạy** vẫn phục vụ. Ticket này kiểm lời
hứa đó, và ⭐ đo cả giới hạn của nó.

**Yêu cầu:**

1. Cửa sổ đo `mat-quorum` phủ từ trước khi tắt máy tới sau khi cluster lành lại.
2. Tắt `lab-ha-2` và `lab-ha-3`; giữ chúng tắt đủ lâu để đường API chết liên tục **ít nhất 60 giây**.
   `lab-ha-1` ở lại: bằng chứng sống rằng "còn một apiserver đang chạy" **không** đồng nghĩa "còn API".
3. Chứng minh: LB còn một backend `UP`; API **ghi** và **đọc** đều thất bại; `/readyz` không còn `ok`;
   container etcd trên `lab-ha-1` **vẫn chạy** nhưng member đó báo **không khỏe**.
4. Bật lại hai control plane node; API đọc ghi lại được; cluster lành hẳn; `deploy-ha-lab.sh` rồi smoke
   PASS.
5. Số đo: đường API chết liên tục ≥ 60 giây; đường ứng dụng ≥ 99% trên cả cửa sổ; mười mẫu cuối của
   cả hai đường đều `200`.
6. ⭐ Lặp lại mất quorum với cửa sổ đo `mat-quorum-worker`; trong lúc mất quorum tắt thêm `lab-ha-5`
   chừng một phút rồi bật lại **trước** khi trả quorum. Đường ứng dụng **dưới** 99%.
7. Postmortem mục `## mat-quorum`; ADR dòng `mat-quorum` đủ sáu ô.

**Ràng buộc:** trước khi tắt máy, xác nhận cả sáu VM còn `8x-vm-ready` — đường lui duy nhất nếu etcd
không hồi phục. Không `etcdctl member remove`, không khôi phục dữ liệu etcd — đó là
[nợ lab #8](README.md#5-sổ-nợ-lab). Phá worker chỉ trên `lab-ha-5`.

**Đọc lại:** bài [06](../06-ha-topology-vi.md) — cả hai mục topology, đoạn về tách rời; bài
[22](../22-architecture-vi.md#nhieu-ban-apiserver) — quorum Raft; bài
[09](../09-troubleshooting-kubeadm-vi.md#kubeadm-bị-treo-khi-chờ-control-plane-trong-quá-trình-cài-đặt-kubeadm-blocks-waiting-for-control-plane-during-installation)
— `crictl` khi `kubectl` vô dụng.

**Bàn giao:** postmortem; ADR; `~/lab-evidence/8b/b11-*`, `~/lab-evidence/8b/do-mat-quorum.csv`
(⭐ `do-mat-quorum-worker.csv`).

**Nghiệm thu:**

- `PASS:` một backend `UP`; ghi và đọc API thất bại; `/readyz` không `ok`; container etcd trên
  `lab-ha-1` chạy nhưng không khỏe.
- `PASS:` sau hồi phục: API đọc ghi được, cluster lành; triển khai lại và smoke PASS.
- `PASS:` cửa sổ đo: API chết liên tục ≥ 60 giây, ứng dụng ≥ 99%, kết thúc bằng `200` ở cả hai đường.
- `PASS:` (⭐) cửa sổ có `lab-ha-5` tắt: ứng dụng dưới 99%.
- `PASS:` postmortem và ADR đã cập nhật.

**Câu hỏi thiết kế** — postmortem, mục `## mat-quorum`:

1. Vì sao **ghi** thất bại? Vì sao **đọc** cũng thất bại? Vì sao etcd và apiserver trên `lab-ha-1`
   còn chạy mà không cứu được gì?
2. LB vẫn báo `lab-ha-1` `UP`. Vì sao? Health check HTTP trên `/readyz` có cứu được kịch bản này không,
   và nó giúp được ở kịch bản nào khác?
3. Bookstore vẫn trả lời. Cái đội **mất** trong lúc này là gì, vì sao đó vẫn là sự cố nghiêm trọng?
4. ⭐ Khi `lab-ha-5` tắt trong lúc mất quorum, những việc gì **bình thường sẽ xảy ra** mà lần này không
   xảy ra? Đường ứng dụng chỉ gõ `lab-ha-4` — node còn sống — vậy vì sao vẫn lỗi?
5. Nếu etcd nằm ở host riêng, tắt hai control plane node có làm mất quorum không?
6. Vì sao cluster hồi phục ngay khi member **thứ hai** lên, không chờ đủ ba?

<details>
<summary>Gợi ý</summary>

- Khi `kubectl` chết, hỏi etcd qua container runtime: `sudo crictl ps --name etcd -q`, rồi
  `sudo crictl exec <id> etcdctl …` với đúng ba file certificate mà hàm `etcd1` dùng.
- `kubectl get --raw='/readyz?verbose' --request-timeout=20s` cho thấy check nào fail.
- Cửa sổ 480 giây là điểm khởi đầu hợp lý: tắt máy, giữ đủ lâu, bật lại, chờ quorum và lành.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script này chính là cuộc diễn tập</summary>

Phần 1 — trên **máy host**: đường lui, rồi mở cửa sổ đo:

```powershell
foreach ($f in $haVmx) {
  $n = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($n -ccontains '8x-vm-ready') { "PASS: $f con 8x-vm-ready" } else { "FAIL: $f mat 8x-vm-ready — DUNG" }
}
Start-Probe mat-quorum 480
```

Phần 2 — chờ khoảng 20 giây. Trên `lab-ha-2`, rồi `lab-ha-3`: `sudo shutdown -h now`. Trên **máy
host**:

```powershell
$down = $haVmx | Where-Object { $_ -like '*lab-ha-2*' -or $_ -like '*lab-ha-3*' }
$still = $down | Where-Object { (& $vmrun -T ws list) -contains $_ }
if ($still.Count -eq 0) { 'PASS: ca lab-ha-2 va lab-ha-3 da Powered off' } else { "CHUA TAT: $($still -join ', ')" }
```

Phần 3 — trên `lab-ha-1`:

```bash
source ~/lab-work/8b/bien.sh
wait_lb_up 1; lbstat | tee "$EV/b11-lb.txt"
test "$UPN" -eq 1 && echo 'PASS: chi con mot backend UP — dung mot kube-apiserver con song'
api_probe b11-mat-quorum
test "$API_WRITE" != 'ok' && echo 'PASS: GHI vao API that bai — mat quorum thi khong ghi duoc etcd'
test "$API_READ" != 'ok' && echo 'PASS: DOC tu API cung that bai'
kubectl get --raw='/readyz?verbose' --request-timeout=20s > "$EV/b11-readyz.txt" 2>&1 || true
kubectl get --raw='/readyz' --request-timeout=20s 2>/dev/null | grep -qx 'ok' \
  && echo 'FAIL: readyz van tra ok — kiem tra lai hai VM da tat chua' || echo 'PASS: /readyz khong con tra ok'
CID="$(sudo crictl ps --name etcd -q | head -n 1)"
test -n "$CID" && echo 'PASS: container etcd van dang chay — tien trinh song, chi la khong co quorum'
sudo crictl exec "$CID" etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key endpoint health > "$EV/b11-etcd-health.txt" 2>&1 || true
grep -q 'is healthy' "$EV/b11-etcd-health.txt" \
  && echo 'FAIL: etcd van bao khoe — hai node kia chua that su tat' \
  || echo 'PASS: etcd member con lai bao KHONG khoe du tien trinh van chay'
```

Phần 4 — sau khi phần 3 PASS và đã qua ít nhất 60 giây kể từ lúc tắt máy, trả quorum. Trên **máy
host**:

```powershell
foreach ($f in $down) { & $vmrun -T ws start $f; Start-Sleep -Seconds 10 }
$up = ($down | Where-Object { (& $vmrun -T ws list) -contains $_ }).Count
if ($up -eq 2) { 'PASS: ca hai VM dang chay lai' } else { "FAIL: moi co $up/2" }
```

Trên `lab-ha-1`:

```bash
source ~/lab-work/8b/bien.sh
for i in $(seq 1 120); do kubectl get nodes --request-timeout=10s >/dev/null 2>&1 && break; sleep 5; done
api_probe b11-hoi-phuc
test "$API_READ" = 'ok' && test "$API_WRITE" = 'ok' && echo 'PASS: du da so member — API doc va ghi lai duoc'
cho_lanh b11
cd "$REPO" && ./scripts/deploy-ha-lab.sh > "$EV/b11-deploy-sau.txt" 2>&1 \
  && ./tests/smoke-ha-lab.sh > "$EV/b11-smoke-sau.txt" 2>&1 && echo 'PASS: trien khai lai va smoke-ha-lab PASS'
```

Phần 5 — khi cửa sổ đo in `KET QUA`, trên **máy host**: `Send-Probe mat-quorum`. Trên `lab-ha-1`:

```bash
source ~/lab-work/8b/bien.sh
tom_tat "$EV/do-mat-quorum.csv"
ge "$API_MAX" 60 && echo "PASS: API chet lien tuc ${API_MAX} giay — da do duoc mat quorum that"
ge "$APP" 99 && echo "PASS: ung dung van dat ${APP}% trong cung khoang do"
test "$CUOI_APP" -eq 1 && test "$CUOI_API" -eq 1 && echo 'PASS: ket thuc luot do ca hai duong deu 200'
ghi_chep mat-quorum
```

Phần 6 ⭐ — lặp lại phần 1 → 4 với `Start-Probe mat-quorum-worker 480`; giữa phần 3 và phần 4, trên
`lab-ha-5`: `sudo shutdown -h now`, chờ khoảng một phút, rồi trên **máy host**
`& $vmrun -T ws start ($haVmx | Where-Object { $_ -like '*lab-ha-5*' })` — bật `lab-ha-5` **trước** hai
control plane node. Khi cửa sổ đo xong: `Send-Probe mat-quorum-worker`. Trên `lab-ha-1`:

```bash
source ~/lab-work/8b/bien.sh
tom_tat "$EV/do-mat-quorum-worker.csv"
lt "$APP" 99 && echo "PASS: mat quorum + mat mot worker — ung dung chi con ${APP}%, khong ai thay the Pod"
```

`deploy-ha-lab.sh` ở phần 4 là bước an toàn: nếu ⭐ làm `db-0` khởi động lại trên thư mục dữ liệu mới,
migration chạy lại và đưa số sách về đủ.

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

1. Mỗi lần ghi phải được **đa số** member xác nhận; ba member thì đa số là hai, còn một thì không bao
   giờ đạt. Đọc qua API mặc định là đọc nhất quán, cũng phải xác nhận với quorum. Tiến trình etcd và
   apiserver trên `lab-ha-1` vẫn chạy — `crictl ps` chứng minh — nhưng **không phục vụ được**: "còn
   máy sống" không bằng "còn cluster".
2. TCP check chỉ hỏi "có ai lắng nghe `6443`"; apiserver trên `lab-ha-1` vẫn lắng nghe. Ở kịch bản này
   không backend nào phục vụ được, nên check HTTP cũng không cứu. Nó có ích khi **một** apiserver mất
   kết nối etcd trong lúc các bản khác vẫn tốt — cải tiến có thể ghi vào ADR, vượt yêu cầu tối thiểu
   của bài 08.
3. Pod đang chạy vẫn chạy; quy tắc của kube-proxy nằm trong kernel của từng node. Cái mất là **quyền
   điều khiển**: không deploy, không scale, Pod chết không ai thay, node mới không join, CronJob không
   tạo Job. Cluster ở trạng thái này chỉ chờ một Pod chết là bắt đầu suy sụp — ⭐ chứng minh điều đó.
4. ⭐ Bình thường: node controller đánh dấu `lab-ha-5` mất liên lạc, endpoint trên node đó bị gỡ khỏi
   Service, controller tạo Pod thay thế trên `lab-ha-4`. Không control plane thì không việc nào xảy
   ra. Quy tắc của kube-proxy trên `lab-ha-4` vẫn chứa đích là Pod `web` (và có thể `api`, `db-0`) trên
   `lab-ha-5` — topology spread của Lab 7a bảo đảm có ít nhất một Pod `web` ở đó — nên một phần request
   tới `lab-ha-4` vẫn bị chuyển sang node đã chết. Bật `lab-ha-5` lại cũng chưa đủ: kubelet cần API
   để biết phải chạy Pod nào.
5. Không. Quorum etcd nằm ở ba host riêng, vẫn ba trên ba; cluster mất dự phòng ở tầng control plane
   nhưng dữ liệu vẫn có quorum — phần "tách rời" của bài 06, nội dung Lab 8c.
6. Hai trên ba đã là đa số — cùng phép đếm đã làm cluster chết ở phần 3.

</details>

## B12. [BS-8B-12] Diễn tập: load balancer là SPOF; chốt ADR

**Loại:** Diễn tập sự cố — tắt `lab-ha-lb`.

**Bối cảnh:** kịch bản `mat-lb`. Ba API server đã dự phòng cho nhau; **chưa ai dự phòng cho load
balancer**. Sprint không được kết thúc với ấn tượng sai rằng cluster đã HA hoàn chỉnh. Sau diễn tập
cuối, đội chốt phần stacked của ADR topology — Lab 8c sẽ đặt số đo của nó cạnh số đo này.

**Yêu cầu:**

1. **Trước** khi tắt LB, ghi lại client nào trong cluster được cấu hình gọi `lab-ha-lb:6443`:
   kubeconfig quản trị, kube-proxy, kubeconfig của các thành phần control plane trên `lab-ha-1`.
2. Cửa sổ đo `mat-lb` phủ từ trước khi tắt LB tới sau khi cluster lành lại.
3. Tắt `lab-ha-lb`. Chứng minh: `kubectl` mất đường vào (đọc và ghi đều fail); **cả ba** apiserver vẫn
   trả lời khi gọi thẳng IP. Giữ LB tắt khoảng 45–60 giây — không lâu hơn.
4. Bật lại LB; `kubectl` hoạt động lại; cluster lành hẳn; smoke PASS.
5. Số đo: đường API chết liên tục ≥ 30 giây; đường ứng dụng ≥ 95%; mười mẫu cuối của cả hai đường đều
   `200`.
6. Postmortem mục `## mat-lb` và mục `## Tổng kết` so bốn kịch bản. ADR topology: dòng `mat-lb` đủ sáu
   ô; mục `## Số đo Lab 8b` là bảng bốn kịch bản với tỷ lệ ứng dụng, tỷ lệ API, gián đoạn dài nhất
   của hai đường; mục `## Stacked (Lab 8b)` hoàn chỉnh; trạng thái *chấp nhận* cho phần stacked; mục
   *Rủi ro đã biết* ghi SPOF và hướng khắc phục.

**Ràng buộc:** không "sửa tạm" bằng cách trỏ kubeconfig vào IP một apiserver rồi coi như xong. Không
dựng thêm VM hay VIP — ghi vào ADR.

**Đọc lại:** bài [08](../08-high-availability-vi.md#tạo-bộ-cân-bằng-tải-cho-kube-apiserver-create-load-balancer-for-kube-apiserver)
— dòng trỏ sang *Các lựa chọn cho cân bằng tải bằng phần mềm*; bài
[22](../22-architecture-vi.md#nhieu-ban-apiserver) — HAProxy + keepalived, VIP trôi giữa hai máy LB,
kube-vip; bài [02](../02-create-cluster-kubeadm-vi.md#những-lưu-ý-về-apiserver-advertise-address-và-controlplaneendpoint-considerations-about-apiserver-advertise-address-and-controlplaneendpoint)
— endpoint là tên để sau này đổi đích được.

**Bàn giao:** postmortem (`mat-lb`, `Tổng kết`); ADR topology hoàn chỉnh phần stacked;
`~/lab-evidence/8b/b12-*`, `~/lab-evidence/8b/do-mat-lb.csv`, `~/lab-evidence/8b/so-do.txt`.

**Nghiệm thu:**

- `PASS:` kubeconfig quản trị và kube-proxy trỏ vào LB; nguồn endpoint của các thành phần đã ghi.
- `PASS:` LB tắt: đọc và ghi qua `kubectl` thất bại; ba apiserver vẫn trả lời trực tiếp.
- `PASS:` LB bật lại: `kubectl` đọc ghi được, cluster lành, smoke PASS.
- `PASS:` cửa sổ đo: API chết liên tục ≥ 30 giây, ứng dụng ≥ 95%, kết thúc bằng `200`.
- `PASS:` ADR: bốn dòng dự đoán đủ sáu ô, bảng số đo đủ bốn kịch bản, trạng thái *chấp nhận*, rủi ro
  SPOF; postmortem có `mat-lb` và `Tổng kết`.

**Câu hỏi thiết kế** — ADR topology và postmortem:

1. Không gì trong cluster hỏng, vậy cái gì chết? Vì sao **mọi** client mất đường vào cùng lúc?
2. Theo evidence của bạn, thành phần nào trên `lab-ha-1` vẫn nói chuyện được với API khi LB chết, thành
   phần nào thì không? Hậu quả nếu LB tắt **lâu**? File đo có cho thấy gì ở đường ứng dụng lúc LB vừa
   lên lại không?
3. Vì sao trỏ `kubectl` thẳng vào một apiserver không phải cách sửa?
4. Production cần thêm đúng những gì để endpoint này không chết được? Vì sao lab không dựng?
5. Tổng kết: xếp bốn kịch bản theo mức độ, rút ra ba ranh giới — quorum, control plane với mặt phẳng
   dữ liệu, stacked với external.

<details>
<summary>Gợi ý</summary>

- Kubeconfig của các thành phần trên `lab-ha-1`: `sudo grep -H 'server:' /etc/kubernetes/*.conf`.
  Kubeconfig của kube-proxy nằm trong ConfigMap `kube-proxy` ở `kube-system`, khóa `kubeconfig.conf`.
- Mỗi lần `tom_tat` ghi thêm một dòng vào `~/lab-evidence/8b/so-do.txt` — bảng số đo của ADR lấy từ
  đó.
- Đổi trạng thái ADR là sửa một dòng; mục *Stacked* gom từ các tiểu mục đã viết ở B3–B11.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script này chính là cuộc diễn tập</summary>

Phần 1 — trên `lab-ha-1`, trước sự cố; rồi trên **máy host**: `Start-Probe mat-lb 300`.

```bash
source ~/lab-work/8b/bien.sh
{ echo "kubeconfig cua ban: $(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
  echo "kube-proxy: $(kubectl -n kube-system get configmap kube-proxy -o jsonpath='{.data.kubeconfig\.conf}' | grep -m1 'server:')"
  sudo grep -H 'server:' /etc/kubernetes/*.conf
} | tee "$EV/b12-ai-di-qua-lb.txt"
grep -q "^kubeconfig cua ban: https://$ENDPOINT$" "$EV/b12-ai-di-qua-lb.txt" \
  && grep -q "^kube-proxy: .*https://$ENDPOINT" "$EV/b12-ai-di-qua-lb.txt" \
  && echo 'PASS: kubeconfig quan tri va kube-proxy tro vao load balancer'
```

Phần 2 — chờ khoảng 20 giây. Trên `lab-ha-lb`: `sudo shutdown -h now`. Trên **máy host**:

```powershell
$lb = $haVmx | Where-Object { $_ -like '*lab-ha-lb*' }
if ((& $vmrun -T ws list) -notcontains $lb) { 'PASS: lab-ha-lb da Powered off' } else { 'CHUA TAT: doi them' }
```

Phần 3 — trên `lab-ha-1`, ngay:

```bash
source ~/lab-work/8b/bien.sh
api_probe b12-lb-chet
test "$API_READ" != 'ok' && test "$API_WRITE" != 'ok' && echo 'PASS: kubectl mat duong vao — kubeconfig tro vao endpoint da chet'
direct_probe b12-lb-chet
test "$DIRECT_OK" -eq 3 && echo 'PASS: ca ba kube-apiserver van song va van tra loi tren :6443'
```

Phần 4 — khoảng 45–60 giây sau khi tắt, trên **máy host**:

```powershell
& $vmrun -T ws start $lb
if ((& $vmrun -T ws list) -contains $lb) { 'PASS: lab-ha-lb dang chay' } else { 'FAIL: VM chua len' }
```

Trên `lab-ha-1`:

```bash
source ~/lab-work/8b/bien.sh
for i in $(seq 1 120); do kubectl get nodes --request-timeout=10s >/dev/null 2>&1 && break; sleep 5; done
api_probe b12-hoi-phuc
test "$API_READ" = 'ok' && test "$API_WRITE" = 'ok' && echo 'PASS: load balancer len lai, kubectl hoat dong tro lai'
cho_lanh b12
"$REPO/tests/smoke-ha-lab.sh" > "$EV/b12-smoke-sau.txt" 2>&1 && echo 'PASS: smoke-ha-lab PASS sau khi LB len lai'
```

Phần 5 — khi cửa sổ đo in `KET QUA`, trên **máy host**: `Send-Probe mat-lb`. Trên `lab-ha-1`:

```bash
source ~/lab-work/8b/bien.sh
tom_tat "$EV/do-mat-lb.csv"
ge "$API_MAX" 30 && ge "$APP" 95 && test "$CUOI_APP" -eq 1 && test "$CUOI_API" -eq 1 \
  && echo "PASS: LB chet — API mat ${API_MAX}s lien tuc, ung dung ${APP}%, ket thuc ca hai duong 200"
ghi_chep mat-lb
cd "$REPO"
ADR="$(ls docs/adr/*-topology-ha.md | head -n 1)"
BAD=0
for S in mat-mot-cp mat-manifest-apiserver mat-quorum mat-lb; do
  grep -Eq "^\| *$S *(\| *(duoc|khong) *){6}\|" "$ADR" || { echo "FAIL: dong du doan $S chua du sau o"; BAD=1; }
  grep -Eq "^\| *$S *\|.*%.*%" "$ADR" || { echo "FAIL: bang so do thieu $S"; BAD=1; }
done
test "$BAD" -eq 0 && echo 'PASS: ADR du du doan, ket qua va so do cho bon kich ban'
grep -q '^## Số đo Lab 8b' "$ADR" && grep -qi 'Trạng thái: chấp nhận' "$ADR" && grep -q 'SPOF' "$ADR" \
  && echo 'PASS: ADR da chot phan stacked, co bang so do va rui ro SPOF'
grep -q '^## Tổng kết' docs/postmortem/8b-mat-control-plane.md && echo 'PASS: postmortem co Tong ket'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính cho ADR và postmortem</summary>

Bảng số đo trong ADR — lấy từ `~/lab-evidence/8b/so-do.txt`, mỗi kịch bản một dòng, cùng cột với ADR
của Lab 8c:

```markdown
## Số đo Lab 8b

Đo từ máy host bằng scripts/do-san-sang.ps1 (CSV ms,app,api), tỷ lệ theo thời gian, cả cửa sổ.

| Kịch bản | Ứng dụng | API | Gián đoạn dài nhất ứng dụng / API |
| --- | --- | --- | --- |
| mat-mot-cp | …% | …% | …s / …s |
| mat-manifest-apiserver | …% | …% | …s / …s |
| mat-quorum | …% | …% | …s / …s |
| mat-lb | …% | …% | …s / …s |
```

1. Không gì hỏng: ba control plane node, ba etcd member đủ quorum, ba apiserver trả lời khi gọi
   thẳng, Bookstore phục vụ. Thứ chết là **cái tên** mà mọi client được dạy để gọi: `lab-ha-lb:6443`
   nằm trong kubeconfig quản trị, ConfigMap `cluster-info`, ConfigMap `kube-proxy`, kubelet của worker.
2. Đọc `b12-ai-di-qua-lb.txt` của bạn. Theo kubeadm hiện hành, kubeconfig của controller-manager và
   scheduler trỏ apiserver **cục bộ** của node; kubelet của control plane node join sau trỏ apiserver
   cục bộ khi feature gate `ControlPlaneKubeletLocalMode` bật (mặc định bật ở v1.35 — xem tham chiếu
   `kubeadm init` ở mục 5); kubelet và kube-proxy của **worker** đi qua LB. LB tắt lâu thì kubelet của
   worker không gia hạn được trạng thái node trong khi controller trên control plane vẫn quan sát:
   worker có thể bị coi là mất liên lạc, Pod trên đó bị đánh dấu không sẵn sàng — thời điểm phụ thuộc
   cấu hình. Khi LB lên lại, kube-proxy nhận EndpointSlice mới trước khi kubelet kịp báo Pod sẵn sàng
   trở lại — một nhịp lỗi ngắn ở đường ứng dụng là có thể. Vì thế diễn tập phải ngắn, và ngưỡng ứng
   dụng của kịch bản này là 95% chứ không phải 99%.
3. Bạn chỉ cứu được terminal của mình; node đang join, kubelet, kube-proxy và mọi client khác vẫn nhìn
   vào endpoint cũ. Cách sửa là làm cho **endpoint** không chết được.
4. Hai máy LB cộng một VIP trôi giữa chúng (keepalived/VRRP), hoặc kube-vip chạy trên chính các control
   plane node (bài 22) — hướng bài 08 trỏ tới. Lab không dựng vì cần thêm VM và IP, vượt ngân sách máy
   host. Ghi thành rủi ro đã biết, kèm điều kiện khắc phục.
5. Từ nhẹ tới nặng: một apiserver mất manifest (một mất mát) → một control plane node tắt (hỏng theo
   cặp, vẫn đủ quorum) → load balancer tắt (cluster nguyên vẹn nhưng không ai vào được API) → hai
   control plane node tắt (mất quorum, mất quyền điều khiển). Ở **cả bốn**, Bookstore vẫn phục vụ —
   chỉ ⭐ (thêm một worker chết) mới làm dịch vụ hỏng. Quorum: ba chịu được mất một, không chịu được
   mất hai. Control plane / dữ liệu: control plane chết là mất khả năng **thay đổi**, chưa phải mất
   dịch vụ. Stacked / external: ở external, tắt hai control plane node không chạm quorum — Lab 8c đo
   lại điều này.

</details>

## B13. Đóng sprint: cleanup, mốc `8x-ha-stacked` và gate cuối

**Mục đích:** bàn giao một repo dựng lại được Bookstore trên cluster HA chỉ bằng một lệnh, đưa repo
về máy host, trả cluster HA về trạng thái **sạch workload** nhưng còn nguyên vẹn sau bốn diễn tập,
chụp mốc cuối trên cả sáu VM, và chứng minh bộ VM chính không bị đụng tới.

### B13.1. Kiểm thử khói của sprint

`tests/smoke-ha-lab.sh` là bộ kiểm thử khói **tích lũy** của sprint này: mọi hợp đồng của tag `lab-8a`
mà overlay `ha-lab` giữ lại, cộng các khác biệt có chủ đích của overlay. B7 đã tạo nó từ đúng khối
dưới đây. `tests/smoke.sh` của chuỗi chính **giữ nguyên** như tag `lab-8a`: hệ thống trên chuỗi chính
không đổi ở sprint này, và cluster HA không có hạ tầng để chạy nó. Chép nguyên văn:

```bash
cat > ~/bookstore/tests/smoke-ha-lab.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói trên cluster HA (bộ VM lab-ha-*) — phiên bản Lab 8b. Kiểm mọi hợp đồng của tag lab-8a
# mà overlay k8s/overlays/ha-lab giữ lại, cộng các khác biệt có chủ đích của overlay.
# Chuỗi chính vẫn dùng tests/smoke.sh.
set -u
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
k() { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }

# 0. Đúng cluster
case "$(kubectl get nodes -o name 2>/dev/null)" in
  *node/lab-ha-*) pass 'dang o cluster HA (node lab-ha-*)' ;;
  *) echo 'FAIL: khong phai cluster HA — dung tests/smoke.sh'; exit 1 ;;
esac

# 1. Namespace và object cấp cluster (3a, 7a)
[ "$(kubectl get namespace "$NS" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null)" = 'bookstore' ] \
  && pass 'namespace bookstore mang label du an' || fail 'namespace bookstore thieu hoac thieu label'
for PC in bookstore-critical bookstore-batch; do
  [ "$(kubectl get priorityclass "$PC" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null)" = 'bookstore' ] \
    && pass "co PriorityClass $PC mang label du an" || fail "thieu PriorityClass $PC"
done
PRIO="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}' 2>/dev/null)"

# 2. Bí mật (3b), PDB (3c), hạn mức (7b)
D="$(jp secret/db-credentials '{.data}')"
{ echo "$D" | grep -q '"postgres-password"' && echo "$D" | grep -q '"authenticator-password"'; } \
  && pass 'Secret db-credentials du hai key' || fail 'Secret db-credentials thieu key'
for P in web api; do
  k get pdb "$P" >/dev/null 2>&1 && pass "co PodDisruptionBudget $P" || fail "thieu PodDisruptionBudget $P"
done
k get resourcequota bookstore-quota >/dev/null 2>&1 && pass 'co ResourceQuota bookstore-quota' || fail 'thieu ResourceQuota bookstore-quota'
k get limitrange bookstore-limits >/dev/null 2>&1 && pass 'co LimitRange bookstore-limits' || fail 'thieu LimitRange bookstore-limits'

# 3. Khác biệt có chủ đích của overlay ha-lab (8b)
N_ING="$(k get ingress --no-headers 2>/dev/null | wc -l)"
N_PVC="$(k get pvc --no-headers 2>/dev/null | wc -l)"
{ [ "$N_ING" -eq 0 ] && [ "$N_PVC" -eq 0 ]; } && pass 'khong Ingress, khong PVC tren cluster HA' \
  || fail "con $N_ING Ingress, $N_PVC PVC"
{ [ -z "$(jp statefulset/db '{.spec.volumeClaimTemplates}')" ] && \
  jp statefulset/db '{.spec.template.spec.volumes[?(@.name=="data")].emptyDir}' | grep -q '{'; } \
  && pass 'db luu du lieu tren emptyDir' || fail 'db chua chuyen sang emptyDir'
[ -z "$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.volumes[*].persistentVolumeClaim}')" ] \
  && pass 'db-backup khong dung PVC' || fail 'db-backup van tro toi PVC'
{ [ "$(jp service/web '{.spec.type}')" = 'NodePort' ] && \
  [ "$(jp service/web '{.spec.ports[?(@.port==80)].nodePort}')" = '30080' ]; } \
  && pass 'Service web NodePort 30080' || fail 'Service web khong phai NodePort 30080'
echo "INFO: $(k get networkpolicy --no-headers 2>/dev/null | wc -l) NetworkPolicy — CNI cua cluster HA khong thuc thi"

# 4. Service (5a)
[ "$(jp service/api '{.spec.ports[0].port}')" = '3000' ] && pass 'Service api cong 3000' || fail 'Service api sai cong'
[ "$(jp service/db '{.spec.clusterIP}')" = 'None' ] && pass 'Service db la headless' || fail 'Service db khong headless'
[ "$(jp service/node-agent '{.spec.internalTrafficPolicy}')" = 'Local' ] \
  && pass 'Service node-agent internalTrafficPolicy Local' || fail 'Service node-agent sai'

# 5. Controller, lập lịch, tài nguyên (3c, 4a, 4b, 7a)
[ "$(jp statefulset/db '{.status.readyReplicas}')" = '1' ] && pass 'StatefulSet db 1/1 Ready' || fail 'StatefulSet db chua Ready'
[ "$(jp pod/db-0 '{.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].name}')" = 'StatefulSet/db' ] \
  && pass 'db-0 thuoc StatefulSet db' || fail 'db-0 khong thuoc StatefulSet db'
[ "$(jp pod/db-0 '{.status.qosClass}')" = 'Guaranteed' ] && pass 'db-0 QoS Guaranteed' || fail 'db-0 khong Guaranteed'
{ [ -n "$PRIO" ] && [ "$(jp pod/db-0 '{.spec.priority}')" = "$PRIO" ]; } \
  && pass 'db-0 dung bookstore-critical' || fail 'db-0 sai priority'
for DEP in web api; do
  WANT="$(jp "deployment/$DEP" '{.spec.replicas}')"; GOT="$(jp "deployment/$DEP" '{.status.readyReplicas}')"
  { [ -n "$WANT" ] && [ "$WANT" = "$GOT" ]; } && pass "Deployment $DEP $GOT/$WANT Ready" \
    || fail "Deployment $DEP ${GOT:-0}/${WANT:-?} Ready"
done
ORPHAN="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}:{.metadata.ownerReferences[0].kind}{"\n"}{end}' 2>/dev/null | grep -c ':$')"
[ "$ORPHAN" -eq 0 ] && pass 'moi Pod cua du an deu co controller' || fail "$ORPHAN Pod tran"
MISS="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{end}' 2>/dev/null | grep -E '=,|,$' | sort -u)"
[ -z "$MISS" ] && pass 'moi container co requests va limits' || fail "container thieu tai nguyen: $(echo $MISS)"
# Vị trí Pod không phải hợp đồng trên cluster HA: sau diễn tập, Pod có thể dồn về node còn sống.
echo "INFO: web tren $(k get pod -l "$SEL,app.kubernetes.io/component=web" -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' 2>/dev/null | sort -u | grep -c .) node, db-0 tren $(jp pod/db-0 '{.spec.nodeName}')"

# 6. Dữ liệu (4b)
N="$(k exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 8 ] && pass "db co $N cuon sach" || fail "db co ${N:-0} cuon sach, can >= 8"

# 7. Bên trong cluster: web, proxy /api/ (5a), node-agent (4b, 5a)
WEB="$(k get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
get() { k exec "$WEB" -c status-writer -- wget -qO- "$1" 2>/dev/null; }
get http://127.0.0.1:8080/ | grep -q 'Bookstore' && pass 'web phuc vu trang chu' || fail 'web khong phuc vu trang chu'
get http://127.0.0.1:8080/status.json | grep -Eq "\"pod\" *: *\"$WEB\"" \
  && pass 'web phuc vu status.json dung ten Pod' || fail 'status.json sai hoac thieu'
T="$(get http://127.0.0.1:8080/api/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "nginx proxy /api/books tra $T cuon sach" || fail "/api/books qua nginx tra $T cuon sach"
NODES="$(kubectl get nodes --no-headers | wc -l)"
R="$(jp daemonset/node-agent '{.status.numberReady}')"
[ "${R:-0}" -eq "$NODES" ] && pass "node-agent Ready tren $R/$NODES node" || fail "node-agent ${R:-0}/$NODES node"
BAD=0
for L in $(k get pod -l "$SEL,app.kubernetes.io/component=agent" -o jsonpath='{range .items[*]}{.spec.nodeName}={.status.podIP}{"\n"}{end}' 2>/dev/null); do
  get "http://${L##*=}:9100/node.json" | grep -Eq "\"node\" *: *\"${L%%=*}\"" || { echo "FAIL: node-agent tren ${L%%=*} sai /node.json"; BAD=1; }
done
[ "$BAD" -eq 0 ] && pass 'moi node-agent tra /node.json dung ten node' || FAIL=1

# 8. Từ ngoài Pod: NodePort 30080 trên mọi node đang chạy web
for IP in $(k get pod -l "$SEL,app.kubernetes.io/component=web" -o jsonpath='{range .items[*]}{.status.hostIP}{"\n"}{end}' 2>/dev/null | sort -u); do
  { curl -s --max-time 5 "http://$IP:30080/" | grep -q 'Bookstore' && \
    curl -s --max-time 5 "http://$IP:30080/api/books" | grep -q '"title"'; } \
    && pass "NodePort $IP:30080 phuc vu trang chu va /api/books" || fail "NodePort $IP:30080 khong phuc vu"
done

# 9. CronJob db-backup (4b, 7a): chính sách và một lần chạy thử vào emptyDir
[ "$(jp cronjob/db-backup '{.spec.concurrencyPolicy}')" = 'Forbid' ] && pass 'db-backup Forbid' || fail 'db-backup khong Forbid'
SDS="$(jp cronjob/db-backup '{.spec.startingDeadlineSeconds}')"
[ "${SDS:-0}" -ge 10 ] && pass "db-backup startingDeadlineSeconds=$SDS" || fail 'db-backup thieu startingDeadlineSeconds >= 10'
[ "$(jp cronjob/db-backup '{.spec.suspend}')" != 'true' ] && pass 'db-backup khong bi tam ngung' || fail 'db-backup dang suspend'
[ "$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.priorityClassName}')" = 'bookstore-batch' ] \
  && pass 'db-backup dung PriorityClass bookstore-batch' || fail 'db-backup khong dung bookstore-batch'
J="smoke-backup-$(date +%s)"
if k create job "$J" --from=cronjob/db-backup >/dev/null 2>&1 \
   && k wait --for=condition=Complete "job/$J" --timeout=180s >/dev/null 2>&1 \
   && k logs "job/$J" 2>/dev/null | grep -q '^db-backup: OK'; then
  pass 'chay thu db-backup thanh cong'
else
  fail 'chay thu db-backup that bai'
fi
k delete job "$J" --ignore-not-found --wait=true >/dev/null 2>&1

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke-ha-lab.sh
```

Script kiểm **hành vi và hợp đồng**, không kiểm cách viết manifest. Nó lấy địa chỉ NodePort từ
`hostIP` của Pod `web` chứ không từ danh sách worker, và **không** khẳng định Pod nào nằm ở node nào
— chỉ in ra dòng `INFO:` — nên dùng lại được ở Lab 8c, nơi workload chạy trên hai control plane node
và `db-0` được ghim sang `lab-ha-4`.

### B13.2. Dựng lại sạch, commit, tag, bundle

```bash
# Chay tren lab-ha-1
cd ~/bookstore
./scripts/teardown-ha-lab.sh && ./scripts/deploy-ha-lab.sh > ~/lab-evidence/8b/b13-deploy.txt 2>&1
./tests/smoke-ha-lab.sh > ~/lab-evidence/8b/b13-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/8b/b13-smoke.txt
```

**PASS:** `smoke exit=0`, file smoke không có dòng `FAIL:`.

Cập nhật `docs/tech-debt.md`: sprint này không thêm, không trả mã `TD` nào. Thêm một dòng ghi chú
dưới bảng: *overlay `ha-lab` (Lab 8b) chạy dữ liệu trên `emptyDir` cho môi trường đo HA — không phải
lùi `TD-04`, `TD-08` của hệ thống*. Nếu ở B12 đội quyết định ghi SPOF của load balancer thành nợ thì
thêm mã mới ở đây, kèm điều kiện trả.

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-8b: sprint 13 — overlay ha-lab, do san sang, ADR topology HA (stacked)'
git tag lab-8b
git bundle create ~/bookstore-lab-8b.bundle --all && git bundle verify ~/bookstore-lab-8b.bundle
```

Chạy trên **máy host** — bundle về máy host là bắt buộc trước khi tắt máy:

```powershell
scp "$($haUser)@192.168.100.231:bookstore-lab-8b.bundle" "$backup\"
$b = Join-Path $backup 'bookstore-lab-8b.bundle'
if ((Test-Path $b) -and ((Get-Item $b).Length -gt 0)) { "PASS: co $b" } else { "FAIL: thieu $b" }
```

### B13.3. Gỡ dự án và gate cluster HA còn nguyên vẹn

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
# Chay tren lab-ha-1
source ~/lab-work/8b/bien.sh
kubectl delete namespace lab-8b --ignore-not-found --wait=true --timeout=300s
cd "$REPO" && ./scripts/teardown-ha-lab.sh | tee "$EV/b13-teardown.txt"

test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-8b' && echo 'PASS: HEAD mang tag lab-8b'
git bundle verify ~/bookstore-lab-8b.bundle >/dev/null 2>&1 && echo 'PASS: bundle lab-8b hop le'
git diff --quiet lab-8a HEAD -- k8s/app k8s/cluster scripts/deploy.sh scripts/teardown.sh tests/smoke.sh \
  && echo 'PASS: chuoi chinh khong doi so voi lab-8a'
BAD=0
for F in k8s/overlays/ha-lab/kustomization.yaml scripts/deploy-ha-lab.sh scripts/teardown-ha-lab.sh \
         scripts/do-san-sang.ps1 tests/smoke-ha-lab.sh platform/ha-lab/README.md platform/ha-lab/haproxy-k8s.cfg \
         'docs/adr/*-topology-ha.md' 'docs/adr/*-overlay-ha-lab.md' 'docs/adr/*-do-san-sang.md' \
         docs/postmortem/8b-mat-control-plane.md; do
  ls $F >/dev/null 2>&1 || { echo "FAIL: thieu $F"; BAD=1; }
done
test "$BAD" -eq 0 && echo 'PASS: du ban giao cua sprint 13'
grep -q 'ha-lab' docs/tech-debt.md && echo 'PASS: tech-debt.md ghi chu overlay ha-lab'
grep -i -A6 'ha-lab' README.md | grep -c -e 'scripts/deploy-ha-lab.sh' -e 'scripts/teardown-ha-lab.sh' -e 'tests/smoke-ha-lab.sh' \
  | grep -qx 3 && echo 'PASS: README.md ban giao ba lenh cua bo VM HA cho Lab 8c'

grep -q 'teardown: sach' "$EV/b13-teardown.txt" && echo 'PASS: teardown-ha-lab go sach du an'
{ [ "$(sl kubectl get namespace,priorityclass,storageclass,persistentvolume -l app.kubernetes.io/part-of=bookstore --no-headers)" -eq 0 ] && \
  [ "$(sl kubectl get all -A -l app.kubernetes.io/part-of=bookstore --no-headers)" -eq 0 ]; } \
  && echo 'PASS: khong con object nao cua du an'
test "$(kubectl get namespace lab-8b --ignore-not-found -o name | wc -l)" -eq 0 && echo 'PASS: namespace lab-8b da bien mat'
test "$(sl kubectl get pods -n default --no-headers)" -eq 0 && echo 'PASS: namespace default khong con Pod'
test "$(sl kubectl get svc -n default --no-headers)" -eq 1 && echo 'PASS: default chi con Service kubernetes'
```

Bốn diễn tập đã tắt máy nhiều lần. Gate dưới đây chứng minh không lần nào để lại di chứng:

```bash
# Chay tren lab-ha-1
source ~/lab-work/8b/bien.sh
kubectl wait --for=condition=Ready node --all --timeout=600s
kubectl get nodes -o wide | tee "$EV/b13-nodes.txt"
ALL_N="$(kubectl get nodes -o name | wc -l)"
CP_N="$(kubectl get nodes -l node-role.kubernetes.io/control-plane -o name | wc -l)"
RDY_N="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}' | grep -c '^True$')"
UNSCH_N="$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.unschedulable}{"\n"}{end}' | grep -c 'true')"
TAINT_BAD="$(kubectl get nodes -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}' \
  | grep -cE 'node.kubernetes.io/(unschedulable|unreachable|not-ready)')"
test "$ALL_N" -eq 5 && test "$CP_N" -eq 3 && test "$RDY_N" -eq 5 && echo 'PASS: van du 5 node, 3 control plane, tat ca Ready'
test "$UNSCH_N" -eq 0 && test "$TAINT_BAD" -eq 0 && echo 'PASS: khong node nao con cordon hay taint su co sot lai'

etcd1 --endpoints="$ETCD_EPS" endpoint health > "$EV/b13-etcd-health.txt" 2>&1 || true
etcd1 --endpoints=https://127.0.0.1:2379 member list -w table | tee "$EV/b13-etcd-members.txt"
HEALTHY="$(grep -c 'is healthy' "$EV/b13-etcd-health.txt")"
MEM_N="$(etcd1 --endpoints=https://127.0.0.1:2379 member list | wc -l)"
STATIC="$(kubectl -n kube-system get pod -l component=etcd \
  -o jsonpath='{range .items[*]}{.metadata.annotations.kubernetes\.io/config\.source}{"\n"}{end}' | grep -c '^file$')"
lbstat | tee "$EV/b13-lb.txt"
UPN="$(grep -c '=UP$' "$EV/b13-lb.txt")"
direct_probe b13
test "$MEM_N" -eq 3 && test "$HEALTHY" -eq 3 && echo 'PASS: van du ba etcd member va ca ba deu khoe manh'
test "$STATIC" -eq 3 && echo 'PASS: topology van la stacked — ba static Pod etcd tren ba control plane node'
test "$UPN" -eq 3 && test "$DIRECT_OK" -eq 3 && echo 'PASS: load balancer va ca ba kube-apiserver deu hoat dong'

{ echo "=== $(date -Is) — trang thai ket thuc Lab 8b ==="
  echo '--- nodes';           kubectl get nodes -o wide
  echo '--- static Pod etcd'; kubectl -n kube-system get pod -l component=etcd -o wide
  echo '--- etcd member';     cat "$EV/b13-etcd-members.txt"
  echo '--- etcd health';     cat "$EV/b13-etcd-health.txt"
  echo '--- controlPlaneEndpoint'
  kubectl -n kube-system get configmap kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' \
    | grep -E 'controlPlaneEndpoint|podSubnet'
  echo; echo '--- load balancer backend'; cat "$EV/b13-lb.txt"
  echo '--- namespaces';      kubectl get namespaces
} | tee "$EV/b13-ho-so-cuoi.txt"
test "$(wc -l < "$EV/b13-ho-so-cuoi.txt")" -ge 25 && echo 'PASS: da ghi ho so cuoi'
```

**PASS:** mọi dòng `PASS:` của B13.3 xuất hiện, không dòng `FAIL:` nào; danh sách namespace không
còn `bookstore`, `lab-8b`.

### B13.4. Gate ngắn phiên bản HA

Bản tương đương của [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab),
sửa cho cluster năm node và thêm ba dòng riêng của topology HA. Lab 8c mở đầu bằng một gate cùng ý
sau khi restore `8x-ha-stacked`.

```bash
# Chay tren lab-ha-1
source ~/lab-work/8b/bien.sh
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

**PASS:** năm node `Ready`; dòng `PASS: readyz ok`; lệnh `--field-selector` trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod; **ba** node mang role `control-plane`;
**ba** dòng `is healthy`; **ba** backend `=UP`. Fail ở đây thì xử lý theo
[mục 4](#4-troubleshooting-của-lab-này) **trước** khi chụp mốc — chụp một cluster đang lệch là ghi
lỗi vào mốc mà Lab 8c sẽ khôi phục.

Dọn thư mục làm việc — nó chứa log `init` có token, lệnh join, file render có Secret — và các bundle
cũ trên `lab-ha-1`. `~/lab-evidence/8b/` và `~/bookstore-lab-8b.bundle` **giữ lại**:

```bash
# Chay tren lab-ha-1
rm -rf ~/lab-work/8b
rm -f ~/bookstore-lab-8a.bundle ~/bookstore-lab-8b-wip.bundle
test ! -e ~/lab-work/8b && echo 'PASS: da don lab-work, khong con token hay khoa tren dia'
```

### B13.5. Tắt sáu VM và chụp `8x-ha-stacked`

Tắt theo thứ tự ngược với thứ tự bật: `lab-ha-5` → `lab-ha-4` → `lab-ha-3` → `lab-ha-2` →
`lab-ha-1` → `lab-ha-lb`. Trên **từng máy**:

```bash
sudo shutdown -h now
```

Chờ cả sáu VM ở *Powered off*, rồi chụp trên **từng VM** với tên đúng nguyên văn:

```text
8x-ha-stacked
```

Ô *Description* ghi `cluster HA stacked etcd dung bang LAB-8B, sach workload, chup <ngay>`. Verify
trên **máy host**:

```powershell
foreach ($f in $haVmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  $a = $names -ccontains '8x-vm-ready'
  $b = $names -ccontains '8x-ha-stacked'
  if ($a -and $b) { "PASS: $f — co ca hai moc" }
  else { "FAIL: $f -> vm-ready=$a ha-stacked=$b | $($names -join ', ')" }
}

$running = & $vmrun -T ws list | Select-Object -Skip 1 | Where-Object { $_ -match '\.vmx$' }
if ($running.Count -eq 0) { 'PASS: khong VM nao dang chay' } else { "FAIL: con chay $($running -join ', ')" }
foreach ($f in $mainVmx) {
  if (-not (Test-Path $f)) { "FAIL: mat file $f"; continue }
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '03-storage-ready') { "PASS: $f — con nguyen moc 03-storage-ready" }
  else { "FAIL: $f -> $($names -join ', ')" }
}
$b = Join-Path $backup 'bookstore-lab-8b.bundle'
if ((Test-Path $b) -and ((Get-Item $b).Length -gt 0)) { "PASS: bundle lab-8b nam tren may host" } else { "FAIL: thieu $b" }
```

**PASS:** sáu dòng `PASS: … co ca hai moc`; `PASS: khong VM nao dang chay`; ba dòng `PASS:` cho ba VM
chính; `PASS: bundle lab-8b nam tren may host`; không dòng `FAIL:` nào. **Giữ cả hai mốc**:
`8x-vm-ready` là đường lui về bộ VM trắng, Lab 8c cần nó để dựng topology external từ đầu.

Lab 8b kết thúc ở đây. Bộ VM HA nằm ở `8x-ha-stacked`, bộ VM chính nằm ở `03-storage-ready`, repo ở tag
`lab-8b` nằm trên máy host, và hai chuỗi không giao nhau ở điểm nào.

---

## 3. Checkpoint 8b

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Vẽ hai topology của bài 06. Trong stacked, etcd member trên `lab-ha-2` phục vụ những
      `kube-apiserver` nào? Trong external? Số host tối thiểu của từng bên, và sáu VM của bạn đủ cho
      bên nào?
- [ ] Ba dữ kiện độc lập nào chứng minh cluster của bạn là stacked? Annotation
      `kubernetes.io/config.source=file` nói lên điều gì mà `kubectl get pod` thường không nói?
- [ ] Load balancer của bạn đáp ứng bốn yêu cầu của bài 08 bằng những dòng cấu hình nào? Trước
      `init`, *refused* và *timeout* nghĩa là gì, cái nào đi tiếp được?
- [ ] Cờ nào của `kubeadm init` làm cluster có thể thành HA, và vì sao đó là quyết định một chiều?
      Endpoint được ghi ở ba nơi nào, mỗi nơi phục vụ ai?
- [ ] `--upload-certs` để lại hiện vật gì, ai sở hữu, sống bao lâu? `init` lúc 09:00, join control
      plane lúc 12:00 — chuyện gì xảy ra, sửa bằng lệnh nào? Hai lệnh join khác nhau đúng những gì?
- [ ] Overlay `ha-lab` bỏ và đổi những gì so với `k8s/app`, vì sao từng thứ? Vì sao không sửa
      `k8s/app`, và vì sao overlay này không bao giờ được trỏ vào dữ liệu thật?
- [ ] Script đo đứng ở đâu, gõ hai đường nào, vì sao hai đường phải tách nhau? Vì sao tỷ lệ tính theo
      thời gian, và ba tiêu chí của một diễn tập — tỷ lệ, gián đoạn dài nhất, mười mẫu cuối — trả lời
      ba câu hỏi gì?
- [ ] Tắt `lab-ha-3`: hai mất mát cùng lúc là gì, bài 06 gọi là gì? Vì sao cluster vẫn ghi được?
- [ ] Tắt thêm `lab-ha-2`: ghi, đọc, `/readyz` ra sao? Vì sao "còn một apiserver đang chạy" không cứu
      được, và vì sao load balancer vẫn báo nó `UP`?
- [ ] Trong cả bốn diễn tập, Bookstore vẫn trả lời từ máy host. Vì sao? Cái đội mất là gì, và ⭐ điều
      gì xảy ra khi một worker chết đúng lúc đó?
- [ ] Xóa manifest `kube-apiserver` trên `lab-ha-2`: ai quyết định bỏ qua node đó, ba thứ nào không
      hỏng theo? Cùng phép thử trên cluster một control plane khác ra sao?
- [ ] Tắt `lab-ha-lb`: lỗi cấu hình hay giới hạn thiết kế? Vì sao trỏ `kubectl` thẳng vào một
      apiserver không phải cách sửa? Production cần thêm những gì?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- **Nợ lab #8** — backup và khôi phục etcd. Ba bản etcd đồng bộ không bảo vệ khỏi xóa nhầm hay hỏng
  dữ liệu, vì lỗi được nhân bản y hệt sang cả ba; đường lui duy nhất vẫn là snapshot VM. Trả ở
  [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa).
- **Nợ kỹ thuật:** không mã `TD` nào phát sinh hay được trả. Các nợ còn mở của hệ thống ở tag
  `lab-8a` giữ nguyên trạng thái — xem `docs/tech-debt.md` và
  [mục 8 của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án). Dữ liệu `emptyDir` của overlay
  là lựa chọn của môi trường đo, không phải nợ của hệ thống.
- **Rủi ro đã biết**, ghi trong ADR topology: một load balancer duy nhất là SPOF.

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng, không nhìn tài liệu:

1. **Từ sáu máy trống tới Bookstore chạy trên cluster HA.** Bắt đầu từ lúc chọn topology và ghi dự
   đoán. Kể đủ các chặng: vì sao load balancer phải có **trước**, cái gì được kiểm trước `init` và hai
   kết quả trái ngược của phép kiểm đó; `init` được truyền những cờ nào, cờ nào làm cluster thành HA
   được; vì sao CNI có trước khi join; hai lệnh join khác nhau ở đâu và mỗi lệnh thêm gì; vì sao
   CoreDNS phải cân bằng lại; rồi repo đi từ máy host sang `lab-ha-1` thế nào và overlay `ha-lab` đưa
   đúng hệ thống của tag `lab-8a` lên một cluster thiếu hạ tầng ra sao. Kết bằng: nếu bỏ
   `--control-plane-endpoint` lúc `init` thì hỏng ở đâu.
2. **Bốn sự cố, xếp theo mức độ, đọc bằng số đo.** Một apiserver mất manifest, một control plane node
   tắt, load balancer tắt, hai control plane node tắt. Với mỗi kịch bản nói: cái gì hỏng, API còn đọc
   ghi được không, script đo từ máy host thấy gì, và vì sao. Rút ra ba ranh giới — quorum, control
   plane với mặt phẳng dữ liệu, stacked với external — và kết bằng một câu về thứ mà sprint này
   **vẫn chưa** làm cho HA được, cùng việc Lab 8c sẽ đo lại.

---

## 4. Troubleshooting của lab này

Sự cố **dựng môi trường** — VM không ping nhau, swap, containerd, package sai version, node
`NotReady` sau join — nằm ở [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Sự cố **của chính kubeadm** tra ở [bài 09 — Xử lý sự cố kubeadm](../09-troubleshooting-kubeadm-vi.md):
trang tra cứu, mở đúng mục cần. Bảng dưới chỉ liệt kê sự cố phát sinh từ nội dung sprint này.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| B1: sau khi clone, `git describe` không ra `lab-8a` | `git -C ~/bookstore log --oneline --decorate -3` | Bundle cũ hoặc HEAD của bundle ở nhánh khác. Chép lại bundle `lab-8a` mới nhất từ máy host; hoặc `git switch -c sprint-13 lab-8a` rồi làm việc trên nhánh đó |
| B1: `git commit` báo không biết danh tính | `git config --global -l` | Đặt `user.name`, `user.email` như trên `lab-k8s-master` |
| **B3: phép thử từ LB ra TIMEOUT chứ không phải refused** | trên `lab-ha-lb`: `ping <ip node>`; `ufw status` trên node; `ip route` | Đúng tình huống bài 08 bảo **dừng**: LB không giao tiếp được với control plane node. Sửa mạng hoặc firewall theo [A4.4](LAB-00-MOI-TRUONG-1.35.7.md#a44-firewall-cho-mạng-lab-cô-lập); **chưa chạy `kubeadm init`** — init với endpoint không tới được để lại state bẩn phải `kubeadm reset` |
| B3, B5: `lbstat` không in gì | `curl -s "http://$LB_IP:8404/stats;csv" \| head -n 2` | HAProxy chưa chạy, hoặc thiếu khối `listen stats`. Có output mà `awk` rỗng thì tên backend khác `k8s-controlplane` — sửa tên trong `bien.sh`. Cột trạng thái là cột 18, `stot` là cột 8; dòng header bắt đầu bằng `# pxname` dùng để đếm lại nếu bản HAProxy của bạn khác |
| B4: `kubeadm init` treo ở bước chờ control plane | `sudo crictl ps`; `sudo journalctl -u kubelet -n 80 --no-pager` | Mục *kubeadm bị treo khi chờ control plane* của [bài 09](../09-troubleshooting-kubeadm-vi.md#kubeadm-bị-treo-khi-chờ-control-plane-trong-quá-trình-cài-đặt-kubeadm-blocks-waiting-for-control-plane-during-installation). Nguyên nhân riêng của lab HA: `--control-plane-endpoint` trỏ vào một LB mà chính node không tới được — chạy lại phép thử node → LB của B3 |
| B4: `ownerReferences` của `kubeadm-certs` rỗng | `kubectl -n kube-system get secret kubeadm-certs -o yaml \| head -n 20`; `sudo kubeadm token list` | Kết luận của bài 08 không đổi — hạn vẫn hai giờ. Đọc hạn ở cột `EXPIRES` của `kubeadm token list`, ghi vào `b4-upload-certs.txt` thay cho phép tính, rồi đi tiếp |
| **B5: join control plane báo lỗi giải mã certificate** | thời điểm chạy `upload-certs`; `kubectl -n kube-system get secret kubeadm-certs` | Khóa hoặc Secret đã hết hai giờ. Sinh lại khóa và token như lời giải B5, rồi join lại. Đúng kịch bản bài 08 cảnh báo; **không** `init` lại cluster |
| B5: join báo cổng `2379`/`2380` hoặc `6443` đang bận | trên node đó: `sudo ss -lntp \| grep -E '2379\|2380\|6443'` | Node còn dấu vết một lần join trước. `sudo kubeadm reset -f` trên chính node đó rồi join lại. `kubeadm reset` là dọn best-effort, **không** đặt lại iptables; vẫn hỏng thì restore **cả sáu VM** về `8x-vm-ready` ([mục 2.3](#23-trình-tự-an-toàn-khi-tắt-và-bật-máy)) |
| B5: control plane join xong mà `member list` không tăng | `kubectl -n kube-system get pod -l component=etcd -o wide`; trên node mới `sudo crictl logs <id etcd>` | Member mới không bắt tay được với member cũ trên `2380`: UFW chưa tắt hoặc đồng hồ lệch trên một trong ba node (`timedatectl`) |
| B5: sau `rollout restart`, CoreDNS vẫn dồn một node | `kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide` | Chạy restart khi chưa đủ node `Ready`. Chờ đủ năm node rồi restart lần nữa. **Không** ép bằng `nodeSelector` |
| B7: `kubectl kustomize` báo file trong `secrets/` hoặc `k8s/cluster/` không nằm trong hay dưới thư mục gốc | lệnh đang dùng có `--load-restrictor=LoadRestrictionsNone` không | Render qua `deploy-ha-lab.sh` hoặc thêm cờ đó; `kubectl apply -k` không có cờ nới lỏng ([Lab 3b](LAB-3B-CAU-HINH-UNG-DUNG.md#b1-bs-3b-01-bí-mật-sinh-tại-chỗ-triển-khai-bằng-kustomize)) |
| B7: render báo thiếu `secrets/tls.crt` hoặc `secrets/db.env` | `ls -l ~/bookstore/secrets/` | Chạy `deploy-ha-lab.sh` một lần — nó sinh cả hai. Base ở tag `lab-8a` của bạn đọc file bí mật tên khác thì thêm bước sinh file đó vào script, ghi lý do vào ADR overlay |
| B7: render báo `failed to find unique target for patch` | tên, namespace, kind trong patch so với `kubectl kustomize … k8s/app` | Patch theo `path` không có `target` phải khớp đúng một object của base. Đối chiếu tên CronJob, StatefulSet, Service với base thật của bạn |
| B7: Job backup thử nghiệm báo volume có nhiều hơn một nguồn | `kubectl kustomize … \| grep -B2 -A4 'name: backup'` | Patch thêm `emptyDir` nhưng quên `persistentVolumeClaim: null` — mỗi volume chỉ được một nguồn |
| B7: `db-0` `Pending` | `kubectl -n bookstore describe pod db-0` | Còn claim template hoặc còn ràng buộc theo label `bookstore.example.com/data`. `deploy-ha-lab.sh` đã chặn claim template; label thì sửa patch `db-emptydir.yaml` theo đúng cách tag `lab-8a` ghim `db` |
| B7: Pod `web` `Pending` vì topology spread | `kubectl -n bookstore describe pod <web>`; `topologySpreadConstraints` của Deployment | Ràng buộc của Lab 7a dựa vào label node chỉ có trên chuỗi chính, hoặc tính cả miền của control plane node đang mang taint. Đối chiếu bài [140](../140-topology-spread-constraints-vi.md); patch trong overlay, không sửa base |
| B7: Service `web` bị từ chối `exceeded quota: bookstore-quota`, nhắc `services.nodeports` | `kubectl -n bookstore describe resourcequota bookstore-quota` | Lab 7b đặt `services.nodeports: 0`. Overlay phải có patch `quota-nodeport.yaml`; quota đã tồn tại thì `deploy-ha-lab.sh` áp lại là đủ. Trần khác (CPU, Pod, Job) chặn thì đối chiếu ngân sách của Lab 7b với hai worker 2 GB, vá đúng khóa đó trong overlay và ghi ADR |
| B7: `deploy-ha-lab.sh` báo `overlay khong render Job db-migrate` | `grep -n 'db-migrate' k8s/app/kustomization.yaml` | Tag `lab-8a` áp migration ngoài `k8s/app`. Thêm file Job vào `resources` của overlay |
| B8: Windows báo script không được ký hoặc bị chặn; hoặc `khong tim thay curl.exe` | lệnh chạy có `-ExecutionPolicy Bypass` không; `Get-Command curl.exe` | Chạy đúng như `Start-Probe` hoặc khối nghiệm thu B8; không đổi chính sách thực thi của máy. `curl.exe` có sẵn từ Windows 10 — thiếu thì dừng, cài là quyết định của bạn |
| B8: đường nền dưới 99%, nhiều mẫu `000` | từ máy host: `ping 192.168.100.234`; từ `lab-ha-1`: `curl -s http://192.168.100.234:30080/api/books` và `curl -sk https://192.168.100.230:6443/readyz` | `curl` từ `lab-ha-1` được mà từ host không: firewall, VPN trên máy host. Chỉ đường API hỏng: kiểm `lbstat`. `/readyz` trả `401`/`403`: quyền ẩn danh tới endpoint sức khỏe đã bị đổi khỏi mặc định kubeadm |
| B8–B12: `tom_tat` báo chưa có file | `ls ~/lab-evidence/8b/do-*.csv` | Chưa `Send-Probe`, hoặc gõ sai tên. `scp` hỏi mật khẩu là bình thường nếu máy host chưa có khóa SSH |
| B9, B11: node không trở lại `Ready` sau khi bật máy | trên host: `vmrun -T ws list`; trên node: `systemctl is-active kubelet containerd`; `journalctl -u kubelet -n 80 --no-pager` | Đi từ dưới lên: VM không boot → console VMware; `ssh` chết → netplan ([A3](LAB-00-MOI-TRUONG-1.35.7.md#a3-đặt-ip-tĩnh-và-phân-giải-tên)); kubelet `inactive` → thường là swap bật lại ([A4.1](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites)); kubelet chạy mà `NotReady` → Pod CNI trên node đó. Không sửa được trong vài phút thì restore **cả sáu VM** về `8x-vm-ready`, clone từ bundle WIP |
| B9–B12: cửa sổ đo không kết thúc bằng `200` (`CUOI_APP` hoặc `CUOI_API` bằng 0) | `tail -n 12 ~/lab-evidence/8b/do-<Ten>.csv` | Cửa sổ kết thúc trước khi cluster lành. Đo lại kịch bản với `-Giay` dài hơn; đừng hạ ngưỡng. Đường ứng dụng dưới ngưỡng ở B9, B10: kiểm Pod CNI và kube-proxy của `lab-ha-4`, rồi CoreDNS |
| **B10: `diff` báo checksum khác bản gốc** | `sudo diff ~/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml` | Đã sửa file, hoặc chép manifest của node khác. Sinh lại đúng cho node đó: `sudo kubeadm init phase control-plane apiserver --control-plane-endpoint lab-ha-lb:6443 --apiserver-advertise-address <IP của chính node>`, rồi chạy lại gate |
| B11: `/readyz` vẫn `ok` sau khi tắt hai node | trên host: `vmrun -T ws list`; `lbstat` | Hai VM chưa thật sự tắt, hoặc tắt nhầm máy. Xác nhận `lbstat` chỉ còn **một** dòng `=UP` rồi mới kết luận |
| **B11: etcd không lấy lại quorum sau khi bật hai node** | `sudo crictl ps --name etcd`; `sudo crictl logs <id>`; `timedatectl` trên ba control plane node | Kiểm đồng hồ trước — lệch giờ làm etcd không bầu được leader. Chờ theo vòng lặp thay vì đếm giây. Vẫn không lên thì restore **cả sáu VM** về `8x-vm-ready`. **Không** thử `etcdctl member remove` hay khôi phục dữ liệu — đó là [nợ lab #8](README.md#5-sổ-nợ-lab) |
| B11 ⭐: sau khi trả quorum, `db-0` hoặc `web` chưa Ready | `kubectl -n bookstore get pods -o wide` | Kubelet trên `lab-ha-5` chỉ biết chạy Pod nào khi đã có API. Chờ rồi `./scripts/deploy-ha-lab.sh && ./tests/smoke-ha-lab.sh` |
| B12: bật lại LB mà `kubectl` vẫn không vào được | trên `lab-ha-lb`: `systemctl is-active haproxy`; `systemctl is-enabled haproxy` | HAProxy chưa `enabled` nên không tự chạy sau boot: `sudo systemctl enable --now haproxy`, ghi vào postmortem rằng B3 bị bỏ sót |
| B12: sau khi LB lên lại, worker `NotReady` hoặc Pod bị thay | `kubectl get nodes`; `kubectl -n bookstore get pods -o wide` | LB tắt lâu: kubelet của worker không gia hạn được trạng thái trong lúc controller trên control plane vẫn quan sát — thời điểm phụ thuộc cấu hình. Chờ hội tụ, chạy `deploy-ha-lab.sh` và smoke; ghi vào postmortem |
| B13: `listSnapshots` không thấy đúng tên mốc | `vmrun -T ws listSnapshots <vmx>` | Tên đúng nguyên văn `8x-vm-ready`, `8x-ha-stacked` trên **cả sáu** VM, không hậu tố, không đổi hoa thường. Chụp lại; xóa snapshot tên sai bằng giao diện VMware |
| Bất kỳ bước nào: `kubectl` treo lâu rồi báo lỗi | `lbstat`; `direct_probe <nhãn>` | Ba nguyên nhân: LB chết (mọi backend biến khỏi CSV, `direct_probe` vẫn 3), một apiserver chết (`lbstat` còn 2 `=UP`), mất quorum (`lbstat` bình thường mà `api_probe` fail cả đọc lẫn ghi) |

---

## 5. Nguồn chính thức

Các mục giải thích ưu tiên snapshot tài liệu Kubernetes v1.35 (`https://v1-35.docs.kubernetes.io/`)
để flag và hành vi khớp minor version của cluster. Tài liệu HAProxy, PowerShell và .NET chỉ giải
thích cú pháp của công cụ ngoài Kubernetes; chúng không đổi quy trình hay gate nào của lab.

- [Kubernetes v1.35 — Options for Highly Available Topology](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)
- [Kubernetes v1.35 — Creating Highly Available Clusters with kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [Kubernetes v1.35 — Installing kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Kubernetes v1.35 — Creating a cluster with kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Kubernetes v1.35 — Troubleshooting kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)
- [Kubernetes v1.35 — Administration with kubeadm](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/)
- [Kubernetes v1.35 — Adding Linux worker nodes](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/adding-linux-nodes/)
- [Kubernetes v1.35 — `kubeadm init`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/) — gồm danh sách feature gate của kubeadm (`ControlPlaneKubeletLocalMode`)
- [Kubernetes v1.35 — `kubeadm join`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/)
- [Kubernetes v1.35 — `kubeadm certs`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-certs/)
- [Kubernetes v1.35 — Ports and Protocols](https://v1-35.docs.kubernetes.io/docs/reference/networking/ports-and-protocols/)
- [Kubernetes v1.35 — Static Pods](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
- [Kubernetes v1.35 — Kubernetes Components](https://v1-35.docs.kubernetes.io/docs/concepts/overview/components/)
- [Kubernetes v1.35 — Declarative Management of Kubernetes Objects Using Kustomize](https://v1-35.docs.kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [Kustomize — `patches`](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/)
- [Kubernetes — `kubectl apply` (cờ `--selector`)](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [Kubernetes v1.35 — Service, `type: NodePort`](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)
- [Kubernetes v1.35 — Network Policies, mục Prerequisites](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/network-policies/#prerequisites)
- [kubeadm — HA considerations, options for software load balancing](https://git.k8s.io/kubeadm/docs/ha-considerations.md#options-for-software-load-balancing)
- [HAProxy — Configuration Manual](https://docs.haproxy.org/)
- [etcd — FAQ về quorum và số lượng member](https://etcd.io/docs/latest/faq/)
- [.NET — `HttpWebRequest`](https://learn.microsoft.com/dotnet/api/system.net.httpwebrequest)
- [PowerShell — about_Execution_Policies](https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_execution_policies)
