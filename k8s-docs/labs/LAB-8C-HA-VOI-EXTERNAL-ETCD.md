# Lab 8c — HA với external etcd

> **Điểm bắt đầu:** snapshot `8x-ha-stacked` trên **cả sáu VM** của nhánh HA — mốc do
> [Lab 8b](LAB-8B-HA-VOI-STACKED-ETCD.md) tạo, xem [chuỗi snapshot](README.md#3-chuỗi-snapshot). Nhánh
> HA có tiền tố `8x-` và **không nằm trên chuỗi snapshot chính**. Repo dự án ở tag `lab-8b`, mang
> sang bằng bundle `bookstore-lab-8b.bundle` đang nằm trên máy host.
> **Điểm kết thúc:** **tạo** snapshot `8x-ha-external` trên cả sáu VM. Lab phá cụm stacked của Lab 8b
> và dựng lại theo topology external etcd — thay đổi hạ tầng vĩnh viễn, nên bắt buộc chụp mốc mới.
> Repo có tag `lab-8c`, bundle `bookstore-lab-8c.bundle` mang về máy host; chuỗi chính tiếp tục ở
> [Lab 9a](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md) với tag đó trên `lab-k8s-master` — cách đưa
> repo về nằm ở bước cuối của [B13](#b13-đóng-sprint-cleanup-và-gate-cuối).
> **Sprint dự án:** Sprint 14 — Kiểm chứng HA với external etcd của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 8b — HA với stacked etcd](LAB-8B-HA-VOI-STACKED-ETCD.md), lab đã dựng bộ VM này.
> **Lab sau:** [Lab 9a — ServiceAccount, authn/authz và RBAC](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md).
> **Bộ VM chính** (`lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`) **không được đụng tới**
> trong toàn bộ lab; B0 và B13 đều có gate chứng minh điều đó.
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng [Giai đoạn 8 — Dựng cluster bằng kubeadm](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm),
ba bài [06 — Các lựa chọn topology cho tính sẵn sàng cao](../06-ha-topology-vi.md),
[07 — Thiết lập cluster etcd có tính sẵn sàng cao với kubeadm](../07-setup-ha-etcd-with-kubeadm-vi.md) và
[08 — Tạo cluster có tính sẵn sàng cao với kubeadm](../08-high-availability-vi.md). **Bài 07 là bài
trục**: nửa đầu sprint dựng cụm etcd ngoài theo đúng tám bước của nó. Hai bài nền dùng để tra cứu là
[02 — Tạo một cluster với kubeadm](../02-create-cluster-kubeadm-vi.md) và
[09 — Xử lý sự cố kubeadm](../09-troubleshooting-kubeadm-vi.md). Kế hoạch gốc của sprint nằm ở
[mục 7.2 của file dự án](DU-AN-BOOKSTORE.md#sprint-13-14--lab-8b-8c-kiểm-chứng-ha); hợp đồng kỹ thuật
ở [mục 4](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật).

Sprint 13 (Lab 8b) đã đưa Bookstore lên một cluster HA stacked có worker riêng bằng overlay
`ha-lab` và bộ lệnh `-ha-lab`, đo sẵn sàng từ máy host bằng `scripts/do-san-sang.ps1` qua bốn sự cố,
và chốt phần stacked của ADR topology. Sprint 14 đi nhánh còn lại của bài 08: **external etcd**.
Hai lab dùng **chung một bộ VM**; chính việc đổi vai sáu máy là cách rẻ nhất để so hai topology —
cùng phần cứng, cùng load balancer, chỉ khác chỗ etcd đứng. Sprint có hai nửa:

- **Vận hành** (B2–B8): reset cụm stacked, dựng cụm etcd ba member, dựng control plane dùng etcd
  ngoài. Đây là thao tác hạ tầng; quy trình, lệnh và gate đã được kiểm chứng trên cluster thật và nằm
  nguyên trong khối *Lời giải tham khảo* của từng ticket.
- **Dự án** (B9–B12): triển khai lại overlay `ha-lab` với một patch của riêng sprint, đo lại bằng
  đúng công cụ của Lab 8b khi mất etcd member, mất quorum, mất control plane, rồi hoàn thiện ADR
  topology bằng bảng `## Số đo Lab 8c` và kết luận cho production.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab không chép lại con
số phiên bản nào trong văn bản. Kubernetes, Flannel và Pod CIDR đọc từ A1.3 rồi điền vào biến ở B0;
image etcd **không ghim tay** mà lấy đúng cách bài 07 chỉ — `kubeadm config images list`.

[Quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab) viết cho bộ VM
chính; nhánh HA dùng nguyên tắc của nó — gate PASS rồi mới mở nội dung — với gate riêng ở B0, vì lab
chạy trên bộ VM khác và bắt đầu từ mốc khác.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi Bookstore chạy được trên cluster external etcd, có số đo sẵn sàng của sprint 14
đặt cạnh số đo của sprint 13 trong một ADR đã chấp nhận, **và** bạn chứng minh, giải thích được:

- Hai topology của bài [06](../06-ha-topology-vi.md) khác nhau ở **quan hệ giữa etcd member và
  `kube-apiserver`** — một-một hay nhiều-nhiều — chỉ ra bằng cấu hình đọc từ hai cluster tự dựng.
- `kubeadm reset` xóa gì và **không** xóa gì; vì sao gỡ control plane bằng reset chứ không bằng tắt
  máy; vì sao với etcd ngoài, reset một control plane node không xóa dữ liệu cluster.
- Vì sao trên ba host etcd, kubelet chạy bằng một unit **có độ ưu tiên cao hơn** unit của kubeadm và
  **không nói chuyện với API server nào**.
- Vì sao mỗi host etcd cần bộ certificate riêng, SAN của chúng chứa gì, và vì sao `ca.key` không
  rời host đầu tiên; hai vai trò của port `2380` và `2379`.
- Cụm etcd ba member chạy **trước khi có bất kỳ cluster Kubernetes nào**, kiểm chỉ bằng ba lệnh
  `etcdctl` chỉ đọc.
- Khối `etcd.external` là **thứ duy nhất** phân biệt hai nhánh của bài [08](../08-high-availability-vi.md),
  vì sao nhánh này **buộc** dùng file cấu hình, và ba bằng chứng độc lập rằng cluster dùng etcd ngoài.
- Với Bookstore đang chạy: mất một etcd member không ai nhận ra; **mất quorum etcd** làm API ngừng
  nhưng ứng dụng vẫn phục vụ; **mất một control plane node** trong cluster không có worker riêng làm
  ứng dụng gián đoạn một phần rồi tự hồi phục; mất một `kube-apiserver` thì ứng dụng không hề biết.
  Mỗi câu có số đo từ máy host làm bằng chứng.
- Quorum nằm ở tầng nào, vì sao hai control plane cộng ba etcd member vẫn hợp lệ, số host tối thiểu
  của từng topology cho Bookstore production, và topology nào dự án chọn — kèm điều kiện đổi quyết
  định.
- Cleanup đúng phạm vi: Bookstore gỡ sạch, taint control plane về mặc định kubeadm, mốc
  `8x-ha-external` có trên đủ sáu VM, bộ VM chính không hề bị chạm, repo `lab-8c` sẵn sàng cho Lab 9a.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm | Ticket kiểm chứng |
| --- | --- |
| [06 — Các lựa chọn topology cho tính sẵn sàng cao](../06-ha-topology-vi.md) | B1 — cụm stacked **đang chạy** của Lab 8b lần cuối: mỗi apiserver chỉ biết etcd member cục bộ, tập node chạy etcd trùng tập node chạy apiserver ([topology stacked](../06-ha-topology-vi.md#stacked-etcd-topology)); B8 — `--etcd-servers` của **cả hai** apiserver trỏ tới **cả ba** etcd host, quan hệ nhiều-nhiều ([topology external](../06-ha-topology-vi.md#external-etcd-topology)); B10, B11 — "hỏng theo cặp" so với "tách rời", đo bằng bốn kịch bản với Bookstore đang chạy; B12 — số host tối thiểu, đánh đổi, ADR |
| [07 — Thiết lập cluster etcd HA với kubeadm](../07-setup-ha-etcd-with-kubeadm-vi.md) | B4 — bước 1, kubelet làm trình quản lý dịch vụ cho etcd; B5 — bước 2 đến 6: `kubeadmcfg.yaml` riêng từng member, CA, certificate từng member, phân phối, checklist file; B6 — bước 7 và 8: `kubeadm init phase etcd local`, kiểm sức khỏe cụm bằng `etcdctl` chỉ đọc |
| [08 — Tạo cluster HA với kubeadm](../08-high-availability-vi.md) | B3 — *Tạo bộ cân bằng tải cho kube-apiserver*, đổi backend sang hai control plane mới; B7 — *Các node etcd bên ngoài*: chép ba file certificate, `etcd.external`, `kubeadm init --config --upload-certs`, cài CNI; B8 — join control plane còn lại, cân bằng lại CoreDNS; B11 — load balancer ngừng gửi request tới apiserver đã mất, `ControlPlaneEndpoint` không đổi |
| [02 — Tạo một cluster với kubeadm](../02-create-cluster-kubeadm-vi.md) *(bài nền)* | B2 — *Gỡ bỏ node*, *Dọn dẹp control plane*: thứ tự `drain` → `kubeadm reset` → `kubectl delete node`, và những thứ reset **không** dọn; B9 — *Cách ly control plane node*: gỡ taint để Bookstore chạy được; B13 — trả taint về mặc định |
| [09 — Xử lý sự cố kubeadm](../09-troubleshooting-kubeadm-vi.md) *(bài nền)* | [Mục 4](#4-troubleshooting-của-lab-này) — dùng đúng công dụng trang tra cứu: reset treo khi runtime không xóa container, lỗi certificate TLS của kubeconfig cũ, init treo khi chờ control plane |

Các phần **không kiểm chứng được** trong lab này, kèm lý do:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài 07 — vai trò riêng của từng certificate `peer`, `server`, `healthcheck-client`, `apiserver-etcd-client` ở mức PKI | Lab sinh và phân phối đủ bốn bộ và đọc SAN của `server.crt`, nhưng không đi vào vòng đời PKI. Đó là **[nợ #7](README.md#5-sổ-nợ-lab)**, trả ở [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ) |
| Bài 07 — chạy `etcdctl` bằng `crictl run` từ một image rời | Lab đạt cùng mục đích không kéo thêm image: `crictl exec` vào chính container etcd đang chạy. Cách của bài chỉ cần khi etcd chưa chạy |
| Bài 07 — "etcd không hỗ trợ dual-stack" | Nhánh HA chạy IPv4. Dual-stack là bài [05](../05-dual-stack-support-vi.md) riêng và đòi dựng lại cả hai tầng |
| Backup và restore etcd | **[Nợ #8](README.md#5-sổ-nợ-lab)**, chưa trả ở lab này. `etcdctl` chỉ dùng để đọc. Trả ở [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa) |
| Bài 08 — *Cài đặt các worker* | Sáu máy đã có vai hết; xem [mục 2.1](#21-bảng-đổi-vai-của-sáu-vm). Lệnh join worker đã thực hành ở [A5.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a53-join-hai-worker) và Lab 8b; ở đây Bookstore chạy trên control plane sau khi gỡ taint |
| Bài 08 — *Phân phối certificate thủ công* | Lab dùng `--upload-certs`. B5 đã thực hành chính kỹ thuật `scp` certificate, ở tầng etcd |
| Bài 08 — load balancer có tính sẵn sàng cao | Nhánh HA chỉ có một `lab-ha-lb`, nên nó là điểm hỏng đơn — Lab 8b đã chứng minh. Cặp LB cần máy thứ bảy và một VIP, ngoài phạm vi giai đoạn 8; B12 ghi nó thành điều kiện của production |
| Bài 06 — ba control plane node đầy đủ cho topology external | Cần bảy máy: 3 etcd + 3 control plane + 1 LB. Bộ VM có sáu. B12 biến đánh đổi này thành câu hỏi thiết kế |

Nợ của sprint — nói được ở checkpoint:

- **Nợ lab:** không phát sinh, không trả. **[Nợ #7](README.md#5-sổ-nợ-lab)** (vòng đời certificate)
  và **[nợ #8](README.md#5-sổ-nợ-lab)** (backup và restore etcd) vẫn mở sau lab này.
- **Nợ kỹ thuật:** không phát sinh, không trả. Các mã còn mở ở tag `lab-8b` giữ nguyên — xem
  `docs/tech-debt.md` của repo và [mục 8 của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án).
  Những chỗ overlay `ha-lab` hạ chuẩn so với chuỗi chính (dữ liệu `db` trên `emptyDir`, không
  Ingress) là thỏa hiệp của môi trường HA, đã ghi ở Lab 8b, không phải nợ mới.

### 1.2. Thời lượng

4–5 giờ, tính từ lúc B0 PASS. Ba đoạn tốn thời gian nhất là B2 (reset năm node), B5–B6 (certificate
và static Pod trên ba host) và B10–B11 (bốn kịch bản, mỗi kịch bản có một chu kỳ tắt–bật máy kèm
một lượt đo từ máy host). Phép đo từ máy host của kịch bản A ở B10 đánh dấu ⭐ là tùy chọn.

Chia hai phiên: phiên 1 làm B0–B6 (kết thúc với cụm etcd chạy, chưa có Kubernetes), phiên 2 làm
B7–B13. Giữa hai phiên có thể tắt cả sáu VM theo thứ tự ở [mục 2.4](#24-các-quy-ước-còn-lại): kubelet
trên ba host etcd khởi động lại static Pod etcd khi máy bật. Gate tiếp tục đầu phiên 2: trên
`lab-ha-1`, chạy lại Script nghiệm thu của B6 — chỉ đọc; trên `lab-ha-4`, nơi giữ repo,
`git -C ~/bookstore status --porcelain` phải rỗng. Điểm dừng này **chưa có Kubernetes**: không có API
server, nên gate ngắn A5.5, `kubectl` và smoke đều không áp dụng.

Mọi bước chờ phụ thuộc cấu hình cluster và phần cứng host, nên đều viết dưới dạng
`kubectl wait --timeout` hoặc vòng lặp có điều kiện dừng và số vòng tối đa, không phải một con số cố
định.

---

## 2. Quy ước và an toàn

> **Cảnh báo — lab này PHÁ một cluster đang chạy.** B2 chạy `kubeadm reset` trên năm node của cụm
> stacked mà Lab 8b đã dựng. Thao tác **không hoàn tác được bằng lệnh**; đường lui duy nhất là
> snapshot. Không mở B2 khi B0 chưa PASS.

> **Cảnh báo — lab này TẮT máy ảo nhiều lần, kể cả hai máy cùng lúc.** B10 cố ý làm mất quorum của
> cụm etcd. Dữ liệu etcd nằm trên đĩa từng member nên cụm hồi phục khi máy bật lại, nhưng **chỉ khi
> bạn tắt đúng máy và bật lại đủ máy**. Đọc hết mục 2.1 trước khi chạm vào nút nguồn nào.

### 2.1. Bảng đổi vai của sáu VM

Lab 8c chạy trên **đúng bộ VM mà Lab 8b đã dựng**, không thêm máy. Spec máy, cách cài OS, containerd
và kubeadm nằm ở [Lab 8b](LAB-8B-HA-VOI-STACKED-ETCD.md) và
[phần A của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a4-chuẩn-bị-os-và-container-runtime); lab này không
chép lại.

| Tên máy | IP | Vai ở Lab 8b | **Vai ở Lab 8c** | Ghi chú của Lab 8c |
| --- | --- | --- | --- | --- |
| `lab-ha-lb` | `192.168.100.230` | load balancer | load balancer | Giữ vai, **đổi backend** ở B3 |
| `lab-ha-1` | `192.168.100.231` | control plane 1 | **etcd node 1** | `$HOST0` của bài 07 — nơi sinh CA, nơi `ca.key` ở lại; **không bao giờ bị tắt** |
| `lab-ha-2` | `192.168.100.232` | control plane 2 | **etcd node 2** | `$HOST1` của bài 07; bị tắt ở B10 |
| `lab-ha-3` | `192.168.100.233` | control plane 3 | **etcd node 3** | `$HOST2` của bài 07; bị tắt ở B10 |
| `lab-ha-4` | `192.168.100.234` | worker 1 | **control plane 1** | Nơi `kubeadm init`; giữ repo `~/bookstore`, kubeconfig quản trị; chạy `db-0`; **không bao giờ bị tắt** |
| `lab-ha-5` | `192.168.100.235` | worker 2 | **control plane 2** | **Fault target duy nhất** ở tầng control plane (B11) |

Hệ quả phải nhớ suốt lab:

- **`controlPlaneEndpoint` là `192.168.100.230:6443`**, tức `lab-ha-lb` — không phải IP của một
  control plane node nào. Backend của HAProxy đổi thành `.234` và `.235` ở B3.
- **Ba etcd host nghe `2379`/`2380`, không nghe `6443`. Hai control plane node nghe `6443`, không nghe
  `2379`/`2380`.** Đây là ranh giới cụ thể của chữ *tách rời* trong bài 06.
- **Cluster mới không có worker riêng**: 3 etcd + 2 control plane + 1 LB là sáu máy. Bookstore chạy
  trên hai control plane node sau khi gỡ taint ở B9; B13 trả taint về mặc định trước khi chụp mốc.
  Vì sao cấu hình này vẫn hợp lệ và production phải khác thế nào là câu hỏi thiết kế của B12.
- **Một load balancer là điểm hỏng đơn** — Lab 8b đã chứng minh. Bốn kịch bản của B10–B11 chứng minh
  tính tách rời giữa etcd và control plane, **không** chứng minh cluster chịu được mọi loại hỏng hóc.
- Quy ước "fault injection chỉ trên `lab-k8s-worker2`" của [mục 6 README](README.md#6-quy-ước-chung-trong-mọi-lab)
  áp cho bộ VM chính, bộ này **không chạy** trong lab. Trên nhánh HA: tầng control plane chỉ phá
  `lab-ha-5`; tầng etcd chỉ tắt `lab-ha-2`, `lab-ha-3`, và luôn bật lại ngay sau khi đo.

### 2.2. Repo, ba lệnh của bộ VM HA và công cụ đo

- Repo dự án trong sprint này nằm ở **`~/bookstore` trên `lab-ha-4`**, clone từ bundle `lab-8b` ở
  B0.5. Bản `~/bookstore` mà Lab 8b để lại trên `lab-ha-1` không dùng nữa — máy đó thành etcd host.
- **Repo chỉ được đổi ở `docs/` và `k8s/overlays/ha-lab/`.** `k8s/app/`, `k8s/cluster/`, `scripts/`,
  `tests/`, `platform/` giữ nguyên như tag `lab-8b`: chuỗi chính ở Lab 9a dùng `k8s/app` và ba script
  của nó, còn bộ VM HA dùng lại nguyên bộ công cụ của Lab 8b.
- Lab 8b bàn giao ba lệnh cho bộ VM HA, chạy từ gốc repo và ghi trong `README.md` dưới tiêu đề chứa
  `ha-lab`: `./scripts/deploy-ha-lab.sh`, `./scripts/teardown-ha-lab.sh`, `./tests/smoke-ha-lab.sh`.
  B0.5 gán chúng vào `DEPLOY_HA`, `TEARDOWN_HA`, `SMOKE_HA`; lab này không viết lại script nào.
- **Đo từ máy host** bằng chính `scripts/do-san-sang.ps1` của Lab 8b: mỗi nửa giây gọi hai đường —
  ứng dụng `/api/books` qua NodePort `30080` của `lab-ha-4` (không qua API server) và `/readyz` qua
  load balancer (không qua ứng dụng) — ghi `do-<Ten>.csv` dạng `ms,app,api`. `lab-ha-4` vẫn là node
  của cluster mới, load balancer vẫn ở `.230`, nên hai URL mặc định của script đúng nguyên; chỉ thư
  mục ghi đổi sang `lab-evidence\8c`. Cùng script, cùng nhịp, cùng công thức — số đo hai sprint đặt
  cạnh nhau được trong ADR.
- **Đo trên `lab-ha-4`** bằng `~/lab-work/8c/ham-do.sh` (B9): `cp_ok`, `app_ok`, `ehealth`, và
  `tom_tat` chép nguyên công thức của `bien.sh` Lab 8b. Lab không dùng lại `bien.sh` vì nó gắn với
  topology stacked — tên ba control plane, `etcdctl` qua static Pod trên `lab-ha-1` — và nằm trên
  `lab-ha-1`, máy sắp thành etcd host.

### 2.3. Ba file biến, để không mất trạng thái giữa các shell

Lab chạy trên sáu máy Linux cộng máy host; biến được ghi ra file và **mọi khối lệnh tự nạp lại**:

| File | Tạo ở | Nằm trên | Chứa |
| --- | --- | --- | --- |
| `~/lab-work/8c/bien-ha.sh` | B0.4 | `lab-ha-1`, chép sang `lab-ha-4` | `K8S_VER`, `FLANNEL_VER`, `POD_CIDR`, `LB`, `E1`–`E3`, `CP1`, `CP2`, `N1`–`N5`, `ALL`, `WK`, `EV` |
| `~/lab-work/8c/bien-du-an.sh` | B0.5 | `lab-ha-4` | `DEPLOY_HA`, `TEARDOWN_HA`, `SMOKE_HA` |
| `~/lab-work/8c/ham-do.sh` | B9 | `lab-ha-4` | hàm đo, nạp luôn hai file trên |

Trên máy host, khối biến của B0.1 — cùng tên biến với Lab 8b: `$vmrun`, `$mainVmx`, `$haVmx`,
`$backup`, `$ev`, `$probe`, `$haUser`, hai hàm `Start-Probe`, `Send-Probe` — phải chạy lại mỗi khi mở
cửa sổ PowerShell mới. `Start-Probe` mở một cửa sổ riêng cho lượt đo, nên cửa sổ chính luôn rảnh để
tắt, bật máy bằng `vmrun`.

### 2.4. Các quy ước còn lại

- **Mọi khối lệnh mở đầu bằng dòng ghi rõ chạy trên máy nào.** Không ghi thì mặc định là
  `lab-ha-4`. Khối PowerShell luôn ghi "Chạy trên **máy host**" và cửa sổ nào.
- **Bằng chứng ghi vào `~/lab-evidence/8c/`** trên máy sinh ra nó; file tạm vào `~/lab-work/8c/`. File
  đo của máy host nằm ở `$HOME\lab-evidence\8c\`, `Send-Probe` chép sang `~/lab-evidence/8c/` của
  `lab-ha-4`; mỗi lần `tom_tat` ghi thêm một dòng vào `~/lab-evidence/8c/so-do.txt`.
- **Không con số phiên bản nào viết cứng** trong văn bản; biến điền từ
  [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). HAProxy lấy từ gói của Ubuntu,
  không ghim version — nó không phải thành phần Kubernetes.
- **Tuyệt đối không chạm bộ VM chính.** Ba máy `lab-k8s-*` ở trạng thái *Powered off* và giữ nguyên
  `03-storage-ready` từ đầu tới gate cuối của B13. Bước *Bàn giao cho Lab 9a* chạy **sau** gate đó.
- **`etcdctl` chỉ dùng để đọc**: `member list`, `endpoint health`, `endpoint status`. `snapshot save`,
  `snapshot restore`, `member add`, `member remove`, `put`, `del` đều **cấm** — chúng thuộc nợ #8.
- Nhiều gate đọc file chỉ root đọc được trên node khác bằng `ssh <node> 'sudo …'` — quy trình đã
  kiểm chứng của lab này. Lab 8b chọn quy ước ngược lại (gõ lệnh root trực tiếp trên máy đó) vì
  `sudo` của Ubuntu mặc định đòi mật khẩu. B0.3 kiểm điều kiện này trước; không đạt thì xem
  [mục 4](#4-troubleshooting-của-lab-này).
- Thứ tự bật cả bộ: `lab-ha-lb` → `lab-ha-1` → `lab-ha-2` → `lab-ha-3` → `lab-ha-4` → `lab-ha-5`. Tắt
  theo thứ tự **ngược lại**. Tắt bằng `sudo shutdown -h now` trên chính máy đó (hoặc qua `ssh`), rồi
  xác nhận *Powered off* bằng `vmrun`. Chỉ dùng `vmrun … stop … hard` khi mục 4 chỉ dẫn, và ghi điều
  đó vào evidence.
- Không bao giờ restore riêng một VM: mọi lần restore là **cả sáu VM cùng một mốc**.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm thu
  nghĩa là ticket chưa xong.

---

# Phần B — Sprint 14: Kiểm chứng HA với external etcd

> **Node gõ lệnh đổi giữa chừng.** Từ B0 tới hết B2, cụm đang chạy là cụm **stacked** của Lab 8b, nên
> kubeconfig quản trị nằm trên `lab-ha-1`; mọi lệnh `kubectl` chạy ở đó. Từ B7 trở đi, cluster mới có
> kubeconfig trên `lab-ha-4`, và **`lab-ha-4` là node mặc định** cho phần còn lại. Mỗi khối lệnh ghi rõ
> node; đừng suy từ thói quen.

## B0. Nhận bàn giao sprint

**Mục đích:** chứng minh điểm bắt đầu đúng — bộ VM chính tắt và còn nguyên, sáu VM HA còn mốc
`8x-ha-stacked`, cụm stacked bật lên chạy được và không còn gì của dự án — rồi khóa phiên bản vào
biến và nhận repo ở đúng tag `lab-8b`.

Nhịp chuẩn của B0 có thêm `deploy.sh` và `smoke.sh` trên snapshot đầu vào. Sprint này dời bước đó
sang B9: cluster đích **chưa tồn tại**, còn cụm stacked đầu vào sắp bị phá ở B2 và đã được Lab 8b
nghiệm thu Bookstore trước khi chụp `8x-ha-stacked`. Dựng Bookstore lên nó rồi gỡ ngay không chứng
minh thêm điều gì.

### B0.1. Gate trên máy host: hai bộ VM, hai trạng thái

Biến của máy host — **cùng tên** với Lab 8b để dùng chung thói quen; chỉ `$ev` và hai hàm đổi sang
thư mục `8c` và máy `lab-ha-4`. Chạy trên **máy host**, PowerShell, và **chạy lại nguyên khối này mỗi
khi mở cửa sổ PowerShell mới**. Sửa đường dẫn `.vmx` và `$haUser` cho đúng máy của bạn; **giữ nguyên
thứ tự** của `$haVmx` vì các bước bật, tắt dùng đúng thứ tự này:

```powershell
$vmrun   = 'C:\Program Files\VMware\VMware Workstationmrun.exe'
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
$backup = "$HOME\lab-backup"            # noi cac bundle cua du an nam
$ev     = "$HOME\lab-evidence\8c"       # file do tu may host (B9 -> B11)
$probe  = "$backup\do-san-sang.ps1"     # script do cua Lab 8b, chep lai tu repo o B0.5
$haUser = '<user-quan-tri-tren-bo-VM-HA>'
New-Item -ItemType Directory -Force -Path $ev | Out-Null

# Mo mot cua so PowerShell moi chay script do, ghi vao $ev; cua so giu lai de doc dong KET QUA.
function Start-Probe([string]$Ten, [int]$Giay) {
  $f = '-NoExit -NoProfile -ExecutionPolicy Bypass -File "{0}" -Ten {1} -Giay {2} -Thu "{3}"'
  Start-Process powershell -ArgumentList ($f -f $probe, $Ten, $Giay, $ev)
}
# Chep file do do-<Ten>.csv sang lab-ha-4 de script nghiem thu doc.
function Send-Probe([string]$Ten) {
  scp "$ev\do-$Ten.csv" "$($haUser)@192.168.100.234:lab-evidence/8c/"
}

if (Test-Path $vmrun) { 'PASS: tim thay vmrun' } else { 'FAIL: sua duong dan vmrun' }
if ($haUser -match '^<') { 'FAIL: chua dien $haUser' } else { 'PASS: da dien $haUser' }
$running = & $vmrun -T ws list
```

Bộ VM chính phải tắt và còn `03-storage-ready`:

```powershell
foreach ($f in $mainVmx) {
  $isOff  = ($running -notcontains $f)
  $snaps  = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  $hasSnap = ($snaps -ccontains '03-storage-ready')
  if ($isOff -and $hasSnap) { "PASS: $f (off, co 03-storage-ready)" }
  elseif (-not $isOff)      { "FAIL: $f dang CHAY — tat no truoc khi mo Lab 8c" }
  else                      { "FAIL: $f mat snapshot 03-storage-ready -> $($snaps -join ', ')" }
}
```

Sáu VM HA phải còn `8x-ha-stacked`:

```powershell
foreach ($f in $haVmx) {
  $snaps = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  $stacked = ($snaps -ccontains '8x-ha-stacked')
  $vmready = ($snaps -ccontains '8x-vm-ready')
  if     ($stacked -and $vmready) { "PASS: $f (co ca 8x-ha-stacked va 8x-vm-ready)" }
  elseif ($stacked)               { "PASS-YEU: $f co 8x-ha-stacked nhung THIEU 8x-vm-ready" }
  else                            { "FAIL: $f -> $($snaps -join ', ')" }
}
```

Bật sáu VM theo đúng thứ tự mảng:

```powershell
foreach ($f in $haVmx) { & $vmrun -T ws start $f }
& $vmrun -T ws list
```

**PASS:** `PASS: tim thay vmrun`, `PASS: da dien $haUser`; đúng ba dòng `PASS:` cho bộ VM chính; sáu dòng bắt đầu bằng `PASS`
cho bộ HA; `vmrun list` liệt kê đủ sáu `.vmx` của `$haVmx` và **không** máy nào của `$mainVmx`.

Một dòng `FAIL:` ở bộ HA nghĩa là máy đó chưa qua Lab 8b — quay lại Lab 8b, đừng dựng tay. Dòng
`PASS-YEU:` không chặn, nhưng `8x-vm-ready` là **đường lui của B2**; thiếu nó thì khi gate reset fail,
bạn chỉ còn cách quay về `8x-ha-stacked` và làm lại từ đầu. Ghi lại máy nào thiếu.

### B0.2. Gate trên cluster: cụm stacked còn sống và sạch dự án

Chạy trên **`lab-ha-1`**:

```bash
mkdir -p ~/lab-evidence/8c
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

LEFT_BS="$( { kubectl get namespace,priorityclass -l app.kubernetes.io/part-of=bookstore --no-headers
              kubectl get all -A -l app.kubernetes.io/part-of=bookstore --no-headers; } 2>/dev/null | wc -l)"
test "$LEFT_BS" -eq 0 \
  && echo 'PASS: cum stacked khong con object nao cua du an — Lab 8b da teardown truoc khi chup moc' \
  || echo "GHI NHAN: con $LEFT_BS object cua du an — Lab 8b dong sprint chua dung; B2 se xoa sach"

# Trang thong ke CSV cua HAProxy ma Lab 8b dung (cong 8404): ba backend cua cum stacked deu UP.
curl -s --max-time 10 "http://192.168.100.230:8404/stats;csv"   | awk -F, '$1=="k8s-controlplane" && $2!="BACKEND" {print $2"="$18}' > ~/lab-evidence/8c/b0-lb.txt
cat ~/lab-evidence/8c/b0-lb.txt
test "$(grep -c '=UP$' ~/lab-evidence/8c/b0-lb.txt)" -eq 3   && echo 'PASS: load balancer cua Lab 8b con du ba backend UP, trang thong ke 8404 hoat dong'
```

Khối trên là bản rút gọn của gate ngắn phiên bản HA mà Lab 8b chạy ở B13.4 trước khi chụp mốc.

**PASS:** `PASS: readyz ok`, `PASS: dung moc 8x-ha-stacked …` và `PASS: load balancer … ba backend UP`
cùng xuất hiện. Con số **3 Pod etcd**
là dữ kiện quan trọng nhất của lab: B8 sẽ chứng minh cluster mới có **0** Pod như vậy. Gate fail thì
restore cả sáu VM về `8x-ha-stacked` rồi chạy lại B0 — đừng vá tay một cụm sắp bị phá. Dòng
`GHI NHAN:` không chặn lab nhưng phải ghi vào postmortem đầu tiên của sprint (B10).

### B0.3. SSH không mật khẩu giữa các node

Lab dùng `ssh`/`scp` giữa các node ở nhiều bước — đúng điều kiện mà bài
[07](../07-setup-ha-etcd-with-kubeadm-vi.md#trước-khi-bạn-bắt-đầu-before-you-begin) đặt ra: "một hạ
tầng nào đó để sao chép file giữa các host". Hai máy cần gọi được bốn máy còn lại: `lab-ha-1` (B1–B6)
và `lab-ha-4` (B7 trở đi).

Chạy trên **`lab-ha-1`**, rồi lặp lại y hệt trên **`lab-ha-4`**:

```bash
test -f ~/.ssh/id_ed25519 || ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519

# Ke ca chinh may dang go lenh: nhieu gate chay vong lap ssh qua ca nam node.
for n in lab-ha-1 lab-ha-2 lab-ha-3 lab-ha-4 lab-ha-5; do
  ssh-copy-id -o StrictHostKeyChecking=accept-new "$n"
done

OKSSH=0
for n in lab-ha-1 lab-ha-2 lab-ha-3 lab-ha-4 lab-ha-5; do
  ssh -o BatchMode=yes "$n" 'hostname' >/dev/null 2>&1 && OKSSH=$(( OKSSH + 1 )) \
    || echo "FAIL: ssh khong mat khau toi $n"
done
echo "$(hostname -s): ssh khong mat khau toi $OKSSH/5 node"
test "$OKSSH" -eq 5 && echo "PASS: ssh tu $(hostname -s) toi cac node HA da thong"
```

Nhiều gate của lab đọc file chỉ root đọc được trên máy khác bằng `ssh <node> 'sudo …'`. Lab 8b gõ
lệnh root trực tiếp trên từng máy vì `sudo` của Ubuntu mặc định đòi mật khẩu; kiểm ngay bây giờ máy
của bạn thuộc trường hợp nào. Chạy trên **`lab-ha-1`**, rồi trên **`lab-ha-4`**:

```bash
OKSUDO=0
for n in lab-ha-1 lab-ha-2 lab-ha-3 lab-ha-4 lab-ha-5; do
  ssh -o BatchMode=yes "$n" 'sudo -n true' 2>/dev/null && OKSUDO=$(( OKSUDO + 1 ))     || echo "GHI NHAN: sudo tren $n doi mat khau khi khong co tty"
done
test "$OKSUDO" -eq 5 && echo 'PASS: ssh <node> sudo chay duoc khong can tty tren ca nam node'
```

**PASS:** dòng `PASS: ssh tu lab-ha-1 …` và `PASS: ssh tu lab-ha-4 …` cùng xuất hiện. Dòng
`PASS: ssh <node> sudo …` vắng mặt không chặn lab, nhưng mọi gate có `ssh <node> 'sudo …'` sẽ trả
rỗng: chạy phần bên trong dấu nháy **trực tiếp trên máy đó** theo quy ước của Lab 8b — xem
[mục 4](#4-troubleshooting-của-lab-này).

### B0.4. Khóa phiên bản và địa chỉ vào file biến

Ba giá trị dưới đây **không viết cứng vào lab**. Mở
[bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa), đọc ba dòng
*Kubernetes control plane*, *CNI (Flannel manifest release)* và *Pod CIDR*, rồi điền. Chạy trên
**`lab-ha-1`**:

```bash
mkdir -p ~/lab-work/8c ~/lab-evidence/8c
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

test "$(kubeadm version -o short)" = "$K8S_VER" \
  && echo 'PASS: kubeadm tren lab-ha-1 dung version cua A1.3' \
  || echo "FAIL: kubeadm la $(kubeadm version -o short), A1.3 ghi $K8S_VER"

LB='192.168.100.230'
E1='192.168.100.231'; E2='192.168.100.232'; E3='192.168.100.233'
CP1='192.168.100.234'; CP2='192.168.100.235'
N1='lab-ha-1'; N2='lab-ha-2'; N3='lab-ha-3'; N4='lab-ha-4'; N5='lab-ha-5'
ALL="$N1 $N2 $N3 $N4 $N5"
WK=~/lab-work/8c; EV=~/lab-evidence/8c

OKIP=0
for pair in "$N1 $E1" "$N2 $E2" "$N3 $E3" "$N4 $CP1" "$N5 $CP2"; do
  set -- $pair
  test "$(getent hosts "$1" | awk '{print $1}')" = "$2" && OKIP=$(( OKIP + 1 )) \
    || echo "FAIL: $1 khong phan giai ra $2"
done
echo "cap ten/IP dung: $OKIP/5"
test "$OKIP" -eq 5 && echo 'PASS: /etc/hosts khop bang doi vai o muc 2.1'

test "$BAD" -eq 0 && test "$OKIP" -eq 5 \
  && declare -p K8S_VER FLANNEL_VER POD_CIDR LB E1 E2 E3 CP1 CP2 N1 N2 N3 N4 N5 ALL WK EV \
     > ~/lab-work/8c/bien-ha.sh
ssh lab-ha-4 'mkdir -p ~/lab-work/8c ~/lab-evidence/8c'
scp ~/lab-work/8c/bien-ha.sh lab-ha-4:lab-work/8c/
ssh lab-ha-4 'source ~/lab-work/8c/bien-ha.sh && test "$CP1" = 192.168.100.234 && test -n "$K8S_VER"' \
  && echo 'PASS: lab-ha-1 va lab-ha-4 co cung file bien-ha.sh'
```

**PASS:** bốn dòng `PASS:` về phiên bản, `kubeadm`, `/etc/hosts` và file biến xuất hiện, không còn
dòng `FAIL:`. Placeholder chưa điền làm gate FAIL ngay — đó là mục đích của nó. Sai `/etc/hosts` là sai
suốt lab: mọi SAN, mọi endpoint etcd và `--etcd-servers` đều xây trên năm cặp tên/IP này. `kubeadm`
lệch version thì xử lý theo
[A4.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl).

### B0.5. Nhận repo ở tag `lab-8b`

Chép bundle sang `lab-ha-4` — máy giữ repo suốt sprint. Chạy trên **máy host**:

```powershell
$b8b = Join-Path $backup 'bookstore-lab-8b.bundle'
if ((Test-Path $b8b) -and ((Get-Item $b8b).Length -gt 0)) { "PASS: co $b8b" } else { "FAIL: thieu $b8b" }
scp $b8b "$($haUser)@192.168.100.234:"
```

Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/bien-ha.sh
git --version >/dev/null && echo 'PASS: co git'
git bundle verify ~/bookstore-lab-8b.bundle >/dev/null 2>&1 && echo 'PASS: bundle lab-8b toan ven'
test -e ~/bookstore || git clone -q ~/bookstore-lab-8b.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-8b' && echo 'PASS: HEAD dung tag lab-8b'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
{ test -n "$(git config user.name)" && test -n "$(git config user.email)"; }   && echo 'PASS: git co danh tinh commit' || echo 'FAIL: dat git config --global user.name/user.email nhu Lab 8b'

BAD=0
for F in k8s/overlays/ha-lab/kustomization.yaml k8s/overlays/ha-lab/db-emptydir.yaml          scripts/deploy-ha-lab.sh scripts/teardown-ha-lab.sh tests/smoke-ha-lab.sh scripts/do-san-sang.ps1; do
  test -f "$F" || { echo "FAIL: thieu $F"; BAD=1; }
done
test "$BAD" -eq 0 && echo 'PASS: du overlay ha-lab, ba lenh -ha-lab va script do cua Lab 8b'

ADR_TOPO="$(ls docs/adr/*-topology-ha.md 2>/dev/null | head -n 1)"
test -n "$ADR_TOPO" && grep -q '^## External (Lab 8c)' "$ADR_TOPO" && grep -q '^## Số đo Lab 8b' "$ADR_TOPO"   && echo "PASS: ADR topology cua Lab 8b cho san cho Lab 8c: $ADR_TOPO"
```

Ba lệnh của bộ VM HA nằm trong `README.md`, dưới tiêu đề chứa `ha-lab`. Gán chúng vào ba biến — đúng
ba đường dẫn Lab 8b đã chốt, chạy từ gốc repo:

```bash
cd ~/bookstore
grep -i -A6 'ha-lab' README.md
DEPLOY_HA='./scripts/deploy-ha-lab.sh'
TEARDOWN_HA='./scripts/teardown-ha-lab.sh'
SMOKE_HA='./tests/smoke-ha-lab.sh'
test "$(grep -i -A6 'ha-lab' README.md | grep -c -e "$DEPLOY_HA" -e "$TEARDOWN_HA" -e "$SMOKE_HA")" -eq 3   && declare -p DEPLOY_HA TEARDOWN_HA SMOKE_HA > ~/lab-work/8c/bien-du-an.sh   && echo 'PASS: ba lenh cua README khop ba bien, da ghi vao bien-du-an.sh'
```

Script đo của Lab 8b phải là **đúng bản trong repo ở tag `lab-8b`** — bản trên máy host có thể đã bị
sửa tay. Chạy trên **máy host**:

```powershell
scp "$($haUser)@192.168.100.234:bookstore/scripts/do-san-sang.ps1" "$probe"
if (Test-Path $probe) { "PASS: co script do $probe" } else { "FAIL: thieu $probe" }
```

**PASS:** đủ các dòng `PASS:` của B0.5, không dòng `FAIL:`.

Thiếu file nào, hoặc README không liệt kê đủ ba lệnh, nghĩa là Lab 8b bàn giao **chưa đủ** — có thứ
chỉ tồn tại trong trí nhớ của người làm. Quay lại bổ sung repo ở tag `lab-8b` (commit, tag lại, bundle lại), đừng tự chế
lệnh ở đây ([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)).

## B1. [BS-8C-01] Bằng chứng cuối cùng của topology stacked

**Loại:** Điều tra — trên cụm stacked, chỉ đọc.

**Bối cảnh:** ADR chọn topology cần hai bộ số đặt cạnh nhau. Cụm stacked của Lab 8b sẽ biến mất ở
B2; đây là lần cuối đọc được hình dạng của nó từ một cluster thật. B8 sẽ đọc đúng những trường này trên
cluster external để so.

**Yêu cầu:**

1. Ghi một ảnh chụp cụm stacked ngay trước khi phá: node, Pod control plane theo node, cờ trỏ tới etcd
   của **từng** `kube-apiserver`, và cấu hình cluster mà kubeadm lưu trong cluster.
2. Chứng minh bằng dữ liệu: mỗi `kube-apiserver` chỉ biết đúng etcd member nằm trên **chính node của
   nó**.
3. Chứng minh bằng dữ liệu: tập node chạy etcd **trùng khít** tập node chạy `kube-apiserver`.
4. Ghi con số "số apiserver / số etcd member / số máy" của cụm stacked để dùng ở B8 và B12.

**Ràng buộc:** chỉ đọc. Không sửa gì trên cụm stacked; nó sắp bị phá có kiểm soát ở B2.

**Đọc lại:**

- Bài [06](../06-ha-topology-vi.md): [*Topology etcd xếp chồng*](../06-ha-topology-vi.md#stacked-etcd-topology)
  — câu về etcd member "chỉ giao tiếp với `kube-apiserver` của chính node đó" và rủi ro hỏng hóc theo
  cặp.
- Bài [22](../22-architecture-vi.md#nhieu-ban-apiserver) — nhiều bản `kube-apiserver` sau một load
  balancer, trạng thái nằm ở etcd.

**Bàn giao:** `~/lab-evidence/8c/b1-*.txt` trên `lab-ha-1`; mục `## External (Lab 8c)` của ADR
topology (hoàn thiện ở B12).

**Nghiệm thu:**

- `PASS:` file ảnh chụp cụm stacked tồn tại và có ba dòng `--etcd-servers=`.
- `PASS:` cả ba `kube-apiserver` trỏ `--etcd-servers` tới `https://127.0.0.1:2379`.
- `PASS:` tập node chạy etcd và tập node chạy apiserver đều có ba phần tử và trùng nhau.

**Câu hỏi thiết kế** — trả lời trong ADR topology, mục `## External (Lab 8c)`:

1. `kube-apiserver` trên `lab-ha-1` có bao giờ nói chuyện với etcd member trên `lab-ha-2` không? Vậy
   ba member vẫn là **một** cụm nhờ đâu?
2. Viết lại "hỏng theo cặp" của bài 06 bằng ngôn ngữ hai tập node vừa đo. Máy nào của cụm stacked
   tắt đi mà **không** làm mất etcd member?
3. Lab 8b đã đo Bookstore khi mất control plane trên cụm này. Từ số đo đó, hỏng theo cặp ảnh hưởng
   tới **người dùng Bookstore** ở đâu, và ảnh hưởng tới **người vận hành** ở đâu?

<details>
<summary>Gợi ý</summary>

- Cờ của apiserver nằm trong static Pod manifest trên đĩa từng control plane node, đọc qua `ssh`.
- Pod control plane mang label `tier=control-plane` và `component=<tên>`; cột `.spec.nodeName` cho biết
  nó nằm ở đâu.
- Cấu hình cluster mà kubeadm lưu nằm trong một ConfigMap ở `kube-system`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script này chính là cuộc điều tra</summary>

Chạy trên **`lab-ha-1`**:

```bash
source ~/lab-work/8c/bien-ha.sh

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
} > "$EV/b1-truoc-stacked.txt"
test "$(grep -c -- '--etcd-servers=' "$EV/b1-truoc-stacked.txt")" -eq 3 \
  && echo 'PASS: da ghi anh chup cum stacked, du ba dong --etcd-servers'

CNT_LOCAL="$(grep -c 'etcd-servers=https://127.0.0.1:2379' "$EV/b1-truoc-stacked.txt")"
echo "so apiserver tro toi etcd cuc bo cua chinh no: $CNT_LOCAL/3"
test "$CNT_LOCAL" -eq 3 \
  && echo 'PASS: stacked — moi apiserver chi noi chuyen voi etcd member tren chinh node do'

kubectl -n kube-system get pods -l tier=control-plane \
  -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName,COMP:.metadata.labels.component' \
  > "$EV/b1-tier-cp-stacked.txt"
ETCD_NODES="$(awk '$3=="etcd"{print $2}' "$EV/b1-tier-cp-stacked.txt" | sort -u)"
API_NODES="$(awk '$3=="kube-apiserver"{print $2}' "$EV/b1-tier-cp-stacked.txt" | sort -u)"
echo "node co etcd: $(echo $ETCD_NODES) | node co apiserver: $(echo $API_NODES)"
test "$(echo "$ETCD_NODES" | wc -l)" -eq 3 && test "$ETCD_NODES" = "$API_NODES" \
  && echo 'PASS: tap node chay etcd TRUNG KHIT tap node chay apiserver — do la "hong theo cap"'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính cho ADR</summary>

1. Không. Ba dòng `--etcd-servers=https://127.0.0.1:2379` là ba địa chỉ loopback: không apiserver nào
   biết địa chỉ của hai member kia. Ba member vẫn là một cụm vì chúng nói chuyện với nhau qua port
   peer; quan hệ *apiserver ↔ member* là một-một, quan hệ *member ↔ member* là cả cụm.
2. Hai tập trùng khít nghĩa là mọi máy giữ etcd đều giữ apiserver và ngược lại. Không có máy nào của
   cụm stacked tắt được mà chỉ mất một thứ — trừ worker và load balancer, vốn không giữ etcd.
3. Người dùng Bookstore: theo số đo của Lab 8b, mất một control plane không làm ứng dụng gián đoạn vì
   Pod chạy trên worker riêng; mất quorum làm API chết nhưng ứng dụng vẫn phục vụ. Người vận hành: sau
   khi mất một máy stacked, cụm đứng sát mép quorum — mất thêm một máy là không deploy, không scale,
   không thay được Pod nào nữa. Ghi con số của bạn: **stacked = 3 apiserver, 3 etcd member, 3 máy**.

</details>

## B2. [BS-8C-02] Gỡ cụm stacked bằng `kubeadm reset`

**Loại:** Vận hành — trên cả năm node Kubernetes của bộ HA.

**Bối cảnh:** sáu máy sắp đổi vai. Cụm stacked phải biến mất **sạch**: cụm etcd mới ở B6 khai
`initial-cluster-state: new`, và bất kỳ dữ liệu, cấu hình mạng hay kubeconfig nào còn sót sẽ làm nó
hỏng theo cách rất khó chẩn đoán. Đây không phải bước dọn dẹp mà là nội dung học: thứ tự gỡ node, và
việc `kubeadm reset` chỉ dọn ở mức *best-effort*.

**Yêu cầu:**

1. Không còn API server nào của cụm stacked trả lời. Trên năm node `lab-ha-1` … `lab-ha-5` không còn
   static Pod manifest, kubeconfig hệ thống của cluster, thư mục PKI. Ba máy sắp thành etcd node có
   thư mục dữ liệu etcd **rỗng**.
2. Gỡ theo đúng thứ tự của bài 02: mỗi node được rút workload trước, reset **trên chính node đó**,
   rồi xóa khỏi cluster. Worker trước; control plane xa trước; `lab-ha-1` — nơi gõ lệnh — cuối cùng.
3. **Trước khi dọn tay**, chứng minh ba thứ mà reset **không** dọn — cấu hình CNI trên node,
   kubeconfig của user, quy tắc mạng do kube-proxy tạo — bằng số đếm ghi vào
   `~/lab-evidence/8c/b2-reset-khong-don.txt`. Sau đó dọn sạch cả ba trên năm node.
4. Container runtime trên năm node không còn sandbox nào.

**Ràng buộc:**

- Không mở ticket khi B0 chưa PASS. Không tắt máy thay cho reset.
- Rút workload không dùng `--force`: ở mốc `8x-ha-stacked` không được có Pod trần nào.
- Lệnh xóa iptables ở đây xóa **toàn bộ** quy tắc của máy; chấp nhận được chỉ vì
  [A4.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a44-firewall-cho-mạng-lab-cô-lập) đã tắt UFW trên mạng
  lab cô lập. Đừng mang nó sang máy có firewall thật.
- Thao tác **không hoàn tác được**. Nếu chưa từng reset một cluster nhiều node, mở *Lời giải tham
  khảo* và làm đúng trình tự đó — mục tiêu của ticket Vận hành là làm đúng và giải thích được. Gate
  fail sau một lần thử lại thì đi [đường lui ở cuối ticket](#đường-lui-khi-gate-reset-không-đạt).

**Đọc lại:**

- Bài [02](../02-create-cluster-kubeadm-vi.md): [*Gỡ bỏ node*](../02-create-cluster-kubeadm-vi.md#gỡ-bỏ-node-remove-the-node),
  [*Dọn dẹp control plane*](../02-create-cluster-kubeadm-vi.md#dọn-dẹp-control-plane-clean-up-the-control-plane).
- [Tài liệu tham khảo `kubeadm reset` v1.35](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
  — danh sách đầy đủ những gì reset làm, và ghi chú về external etcd.
- Bài [09](../09-troubleshooting-kubeadm-vi.md#kubeadm-bị-treo-khi-xóa-các-container-được-quản-lý-kubeadm-blocks-when-removing-managed-containers)
  — reset treo khi runtime không xóa được container.

**Bàn giao:** `~/lab-evidence/8c/b2-*.txt` trên `lab-ha-1`; mục *Gỡ cluster cũ* trong
`docs/runbook-ha-external-etcd.md` (repo trên `lab-ha-4`).

**Nghiệm thu:**

- `PASS:` `kubectl` trên `lab-ha-1` không còn tới được API server nào.
- `PASS:` cả năm node không còn manifest, `admin.conf`, `kubelet.conf`, thư mục `pki`.
- `PASS:` `/var/lib/etcd` rỗng trên ba host sắp thành etcd node.
- `PASS:` evidence cho thấy cấu hình CNI, kubeconfig của user và chuỗi `KUBE-` còn **sau** reset.
- `PASS:` sau khi dọn: không node nào còn `/etc/cni/net.d`, kubeconfig cũ, chuỗi `KUBE-`, sandbox.

**Câu hỏi thiết kế** — trả lời trong `docs/runbook-ha-external-etcd.md`, mục *Gỡ cluster cũ*:

1. Kể những thứ `kubeadm reset` **không** dọn. Với mỗi thứ, cluster mới dựng trên máy đó hỏng ra sao
   nếu bạn quên dọn?
2. Vì sao gỡ control plane bằng reset chứ không bằng tắt máy? Theo dõi số etcd member của cụm stacked
   đi từ 3 xuống 1 qua từng lần reset — điều gì xảy ra nếu bạn tắt `lab-ha-3` thay vì reset nó, rồi
   lại tắt tiếp `lab-ha-2`?
3. Từ B7, object của Bookstore — Deployment, Secret, ConfigMap — nằm trong cụm etcd ngoài. Sau này
   reset `lab-ha-5` có xóa chúng không? Đó là mặt nào của tính tách rời?

<details>
<summary>Gợi ý</summary>

- Bài 02 cho thứ tự ba bước và nói rõ reset là dọn dẹp *best-effort*, không đặt lại iptables hay
  IPVS. Tài liệu tham khảo của `kubeadm reset` liệt kê phần còn lại.
- Reset một control plane có một phase gỡ etcd member cục bộ khỏi cụm. Phase đó cần API server còn
  sống — lý do `lab-ha-1` đi cuối.
- Đếm quy tắc kube-proxy: `iptables-save | grep -c KUBE-`. Đếm sandbox: `crictl pods -q`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

Chạy trên **`lab-ha-1`**:

```bash
source ~/lab-work/8c/bien-ha.sh

kubectl --request-timeout=10s get nodes > "$EV/b2-kubectl-sau-reset.txt" 2>&1
grep -qiE 'refused|no such host|Unable to connect|timeout|no configuration' "$EV/b2-kubectl-sau-reset.txt" \
  && echo 'PASS: khong con API server — dung nhu mong doi'

# Hai phép kiểm trên node, ba kết quả tách bạch. Lỗi ssh, sudo hay find thì in rỗng — rỗng không
# bao giờ được hiểu thành "đã xóa" hay "rỗng".
co() {          # <node> <đường dẫn> → CON | MAT
  ssh -o BatchMode=yes "$1" "sudo -n true && if sudo -n test -e '$2'; then echo CON; else echo MAT; fi"
}
dem_trong() {   # <node> <thư mục> → số mục bên trong; thư mục không có thì 0
  ssh -o BatchMode=yes "$1" \
    "if sudo -n test ! -e '$2'; then echo 0; else L=\$(sudo -n find '$2' -mindepth 1 -maxdepth 1) && printf '%s\n' \"\$L\" | grep -c . ; fi"
}
OKR=0
: > "$EV/b2-reset-xoa.txt"
for n in $ALL; do
  M="$(dem_trong "$n" /etc/kubernetes/manifests)"
  A="$(co "$n" /etc/kubernetes/admin.conf)"
  P="$(co "$n" /etc/kubernetes/pki)"
  K="$(co "$n" /etc/kubernetes/kubelet.conf)"
  echo "$n -> manifests=${M:-?} admin.conf=${A:-?} pki=${P:-?} kubelet.conf=${K:-?}" | tee -a "$EV/b2-reset-xoa.txt"
  [ "$M" = 0 ] && [ "$A" = MAT ] && [ "$P" = MAT ] && [ "$K" = MAT ] && OKR=$(( OKR + 1 ))
done
test "$OKR" -eq 5 \
  && echo 'PASS: reset da xoa manifest, admin.conf, kubelet.conf va thu muc pki tren ca nam node'

OKE=0
for n in "$N1" "$N2" "$N3"; do
  C="$(dem_trong "$n" /var/lib/etcd)"
  echo "$n -> so muc trong /var/lib/etcd: ${C:-? (khong kiem duoc)}"
  [ "$C" = 0 ] && OKE=$(( OKE + 1 ))
done
test "$OKE" -eq 3 && echo 'PASS: /var/lib/etcd rong tren ca ba host'

cat "$EV/b2-reset-khong-don.txt"
grep -Eq '^cni_con=[1-9]' "$EV/b2-reset-khong-don.txt" \
  && grep -Eq '^kubeconfig_con=[1-9]' "$EV/b2-reset-khong-don.txt" \
  && grep -Eq '^iptables_con=[1-9]' "$EV/b2-reset-khong-don.txt" \
  && echo 'PASS: evidence chung minh reset KHONG don CNI, kubeconfig cua user, luat kube-proxy'

# Node nào chưa xác nhận được là sạch — còn thứ cần dọn, hoặc không kiểm được — đều bị đếm.
CNI_AFTER=0; KC_AFTER=0; IPT_AFTER=0
for n in $ALL; do
  [ "$(co "$n" /etc/cni/net.d)" = MAT ] || CNI_AFTER=$(( CNI_AFTER + 1 ))
  [ "$(ssh -o BatchMode=yes "$n" 'if test -e "$HOME/.kube/config"; then echo CON; else echo MAT; fi')" = MAT ] \
    || KC_AFTER=$(( KC_AFTER + 1 ))
  C="$(ssh -o BatchMode=yes "$n" 'R=$(sudo -n iptables-save) && printf "%s\n" "$R" | grep -c "KUBE-"')"
  [ "$C" = 0 ] || IPT_AFTER=$(( IPT_AFTER + 1 ))
done
echo "con /etc/cni/net.d=$CNI_AFTER | con kubeconfig=$KC_AFTER | con KUBE-=$IPT_AFTER"
test "$CNI_AFTER" -eq 0 && test "$KC_AFTER" -eq 0 && test "$IPT_AFTER" -eq 0 \
  && echo 'PASS: da don tay ca ba thu reset bo lai'

POD_LEFT=0
for n in $ALL; do
  C=0
  for i in $(seq 1 24); do
    # crictl lỗi thì C rỗng: phép so sánh sai, không bao giờ thành "không còn sandbox".
    C="$(ssh -o BatchMode=yes "$n" 'P=$(sudo -n crictl pods -q 2>/dev/null) && printf "%s\n" "$P" | grep -c .')"
    test "$C" -eq 0 && break
    sleep 5
  done
  echo "$n -> sandbox con lai: $C"
  test "$C" -eq 0 && POD_LEFT=$(( POD_LEFT + 1 ))
done
test "$POD_LEFT" -eq 5 && echo 'PASS: container runtime khong con sandbox nao cua Kubernetes'
```

Vòng lặp sandbox có số vòng tối đa vì thời gian dọn container phụ thuộc cấu hình và phần cứng host;
hết vòng mà còn sandbox là đúng tình huống của bài 09 — xem [mục 4](#4-troubleshooting-của-lab-này).

</details>

<details>
<summary>Lời giải tham khảo — quy trình đã kiểm chứng</summary>

Bảng làm việc gộp từ bài 02 và tài liệu tham khảo của `kubeadm reset`:

| Nhóm | Thứ | Reset có tự dọn không |
| --- | --- | --- |
| Static Pod manifest | `/etc/kubernetes/manifests/*.yaml` | **Có** |
| kubeconfig của cluster | `admin.conf`, `kubelet.conf`, `controller-manager.conf`, `scheduler.conf` | **Có** |
| PKI của cluster | `/etc/kubernetes/pki/` | **Có** |
| Thành viên etcd cục bộ | member của node bị gỡ khỏi cụm (phase `remove-etcd-member`) | **Có** |
| Dữ liệu etcd | `/var/lib/etcd/` khi etcd **cục bộ** | **Có** — nhưng **không** xóa khi dùng **external** etcd |
| Cấu hình CNI | `/etc/cni/net.d/` | **Không** |
| kubeconfig của user | `$HOME/.kube/` | **Không** |
| Quy tắc mạng của kube-proxy | iptables / nftables / IPVS | **Không** |

**1. Gỡ hai worker.** Chạy trên **`lab-ha-1`**:

```bash
source ~/lab-work/8c/bien-ha.sh
for n in "$N4" "$N5"; do
  kubectl drain "$n" --delete-emptydir-data --ignore-daemonsets --timeout=300s
done
kubectl get nodes | tee "$EV/b2-nodes-drained.txt"
test "$(grep -c 'SchedulingDisabled' "$EV/b2-nodes-drained.txt")" -eq 2 \
  && echo 'PASS: ca hai worker da cordon va drain xong'
```

Chạy **trên `lab-ha-4`**, rồi lặp lại y hệt **trên `lab-ha-5`**:

```bash
sudo kubeadm reset -f | tee ~/reset.log
```

Quay lại **`lab-ha-1`**:

```bash
source ~/lab-work/8c/bien-ha.sh
for n in "$N4" "$N5"; do kubectl delete node "$n"; done
test "$(kubectl get nodes --no-headers | wc -l)" -eq 3 && echo 'PASS: chi con ba control plane node'
```

**2. Gỡ `lab-ha-3`, rồi `lab-ha-2`.** Chạy trên **`lab-ha-1`**:

```bash
source ~/lab-work/8c/bien-ha.sh
kubectl drain "$N3" --delete-emptydir-data --ignore-daemonsets --timeout=300s
```

Chạy **trên `lab-ha-3`**:

```bash
sudo kubeadm reset -f | tee ~/reset.log
grep -i 'etcd' ~/reset.log
```

Quay lại **`lab-ha-1`**:

```bash
source ~/lab-work/8c/bien-ha.sh
kubectl delete node "$N3"
MEM="$(kubectl -n kube-system get pod -l component=etcd --no-headers | wc -l)"
echo "Pod etcd con lai: $MEM"
test "$MEM" -eq 2 && echo 'PASS: cum etcd con 2 member, quorum van du (2/2)'
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: API van phuc vu'
kubectl drain "$N2" --delete-emptydir-data --ignore-daemonsets --timeout=300s
```

Chạy **trên `lab-ha-2`**:

```bash
sudo kubeadm reset -f | tee ~/reset.log
```

Quay lại **`lab-ha-1`**:

```bash
source ~/lab-work/8c/bien-ha.sh
kubectl delete node "$N2"
test "$(kubectl get nodes --no-headers | wc -l)" -eq 1 && echo 'PASS: chi con lab-ha-1'
```

**3. Reset node cuối.** Chạy **trên `lab-ha-1`** — sau lệnh này mọi `kubectl` báo lỗi kết nối, và đó
là điều đúng. Thông báo lỗi đến từ `$HOME/.kube/config` vẫn còn nguyên:

```bash
sudo kubeadm reset -f | tee ~/reset.log
```

Nếu gate `/var/lib/etcd` của script nghiệm thu fail — reset là best-effort — xóa tay rồi chạy lại
gate. **Không mang lệnh này sang cluster thật**: ở đó `/var/lib/etcd` là toàn bộ trạng thái Kubernetes.

```bash
source ~/lab-work/8c/bien-ha.sh
for n in "$N1" "$N2" "$N3"; do ssh "$n" 'sudo rm -rf /var/lib/etcd/*'; done
```

**4. Ghi bằng chứng ba thứ reset bỏ lại, rồi dọn.** Chạy trên **`lab-ha-1`**:

```bash
source ~/lab-work/8c/bien-ha.sh
: > "$EV/b2-reset-khong-don.txt"

CNI_LEFT=0
for n in $ALL; do
  C="$(ssh -o BatchMode=yes "$n" 'sudo ls -A /etc/cni/net.d 2>/dev/null | wc -l')"
  echo "$n -> file trong /etc/cni/net.d: $C"
  test "$C" -gt 0 && CNI_LEFT=$(( CNI_LEFT + 1 ))
done
echo "cni_con=$CNI_LEFT" | tee -a "$EV/b2-reset-khong-don.txt"
test "$CNI_LEFT" -ge 1 && echo 'PASS: reset KHONG don /etc/cni/net.d — dung nhu tai lieu ghi'
for n in $ALL; do ssh "$n" 'sudo rm -rf /etc/cni/net.d'; done

KC_LEFT=0
for n in "$N1" "$N2" "$N3"; do
  ssh -o BatchMode=yes "$n" 'test -e "$HOME/.kube/config"' && KC_LEFT=$(( KC_LEFT + 1 ))
done
echo "kubeconfig_con=$KC_LEFT" | tee -a "$EV/b2-reset-khong-don.txt"
test "$KC_LEFT" -ge 1 && echo 'PASS: reset KHONG don $HOME/.kube — dung nhu tai lieu ghi'
for n in "$N1" "$N2" "$N3"; do ssh "$n" 'rm -rf "$HOME/.kube"'; done

IPT_LEFT=0
for n in $ALL; do
  C="$(ssh -o BatchMode=yes "$n" 'sudo iptables-save 2>/dev/null | grep -c "KUBE-"')"
  echo "$n -> dong iptables con chua KUBE-: $C"
  test "$C" -gt 0 && IPT_LEFT=$(( IPT_LEFT + 1 ))
done
echo "iptables_con=$IPT_LEFT" | tee -a "$EV/b2-reset-khong-don.txt"
test "$IPT_LEFT" -ge 1 && echo 'PASS: reset KHONG dat lai iptables — dung cau bai 02 viet'
for n in $ALL; do
  ssh "$n" 'sudo sh -c "iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X"'
  ssh "$n" 'command -v ipvsadm >/dev/null 2>&1 && sudo ipvsadm -C || echo "khong co ipvsadm, bo qua"'
done
```

Bài 02 còn một cách gọn hơn khi chỉ muốn bỏ tham chiếu tới một cluster mà giữ cluster khác trong
kubeconfig: `kubectl config delete-cluster`. Ở đây file chỉ có một cluster nên xóa cả thư mục.

Ý chính cho runbook:

1. Reset bỏ lại cấu hình CNI (Flannel mới đọc phải `.conflist` cũ, Pod có thể nhận IP của dải cụm đã
   chết), kubeconfig của user (`kubectl` gọi vào cluster mới bằng CA cũ → lỗi certificate TLS của bài
   09), quy tắc kube-proxy (Service cũ còn trong kernel), và — với external etcd — dữ liệu etcd.
2. Reset control plane chạy phase `remove-etcd-member`: cụm đi 3 → 2 → 1 có trật tự, danh sách
   member luôn khớp số máy sống. Tắt máy để lại một member chết trong danh sách: cụm 3 member mất 1
   vẫn đủ quorum, nhưng cụm còn 2 member đăng ký mà mất thêm 1 thì không.
3. Không. Tài liệu tham khảo ghi rõ reset không xóa dữ liệu etcd khi etcd là external; object của
   Bookstore sống ở ba host etcd, không ở control plane nào. Mất hay dựng lại một control plane không
   chạm vào dữ liệu cluster.

</details>

### Đường lui khi gate reset không đạt

Không cố sửa một node reset dở. Nếu gate nào của B2 còn fail sau một lần thử lại, chuyển sang mốc
`8x-vm-ready` — "VM sạch đã cài kubeadm, chưa có cluster" mà Lab 8b chụp. Chạy trên **máy host**, cửa
sổ chính:

```powershell
foreach ($f in $haVmx) {
  $snaps = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($snaps -ccontains '8x-vm-ready') { "PASS: $f co 8x-vm-ready" } else { "FAIL: $f KHONG co" }
}
```

**PASS:** sáu dòng `PASS:`. Còn dòng `FAIL:` thì đường lui này không dùng được — quay về
`8x-ha-stacked` và chạy lại từ B0. Đủ sáu thì tắt, khôi phục cả sáu, bật lại theo thứ tự mảng:

```powershell
foreach ($f in $haVmx) { & $vmrun -T ws stop $f soft }
foreach ($f in $haVmx) { & $vmrun -T ws revertToSnapshot $f '8x-vm-ready' }
foreach ($f in $haVmx) { & $vmrun -T ws start $f }
& $vmrun -T ws list
```

Mốc `8x-vm-ready` có trước khóa SSH, file biến và repo. Chạy lại **B0.3, B0.4, B0.5**, rồi gate riêng
của đường lui trên **`lab-ha-1`**:

```bash
source ~/lab-work/8c/bien-ha.sh
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
test "$OKF" -eq 5 \
  && echo 'PASS: duong lui thanh cong — kubeadm da cai, containerd chay, khong con dau vet cluster'
```

**PASS:** dòng `PASS: duong lui thanh cong …` xuất hiện. Bỏ qua B0.2, B1 và phần còn lại của B2 —
không còn cụm stacked để đọc; nếu B1 chưa kịp chạy, lấy số liệu stacked từ evidence của Lab 8b. Đi
thẳng tới B3.

## B3. [BS-8C-03] Load balancer trỏ sang hai control plane mới

**Loại:** Vận hành — trên `lab-ha-lb`.

**Bối cảnh:** bài 08 đặt việc dựng load balancer là **bước đầu tiên, chung cho cả hai phương pháp**.
Lab 8b đã dựng nó: khối cấu hình `#--- Lab 8b` nối vào cuối `/etc/haproxy/haproxy.cfg`, frontend
`k8s-apiserver`, backend `k8s-controlplane` trỏ tới `.231`, `.232`, `.233`, trang thống kê ở cổng
`8404`. Ba máy đó sắp thành etcd node và sẽ không nghe `6443` nữa; backend phải trỏ sang hai control
plane mới **trước** khi `kubeadm init` chạy. Mọi thứ khác của Lab 8b — timeout dài cho watch, trang
thống kê — giữ nguyên.

**Yêu cầu:**

1. Load balancer vẫn nghe `6443`, chuyển tiếp **TCP**, health check TCP trên port apiserver; trang
   thống kê `8404` vẫn chạy.
2. Backend `k8s-controlplane` chỉ gồm `192.168.100.234:6443` và `192.168.100.235:6443`; không còn
   backend nào trỏ tới ba host etcd. Chỉ những dòng `server` đổi; phần còn lại của khối Lab 8b nguyên.
3. Cấu hình của Lab 8b được lưu thành `~/lab-evidence/8c/b3-haproxy-cfg-8b.bak` trước khi sửa; version
   HAProxy quan sát được ghi lại.
4. Từ `lab-ha-4`, kết nối tới load balancer **không timeout** (bị đóng là bình thường lúc này), và
   trang thống kê liệt kê đúng hai backend mới.

**Ràng buộc:** không dùng health check HTTP. Không cài lại HAProxy, không ghim version. Repo không
đổi ở ticket này: `platform/ha-lab/haproxy-k8s.cfg` mô tả load balancer của Lab 8b; cấu hình của Lab 8c
ghi vào runbook (`docs/`).

**Đọc lại:** bài [08](../08-high-availability-vi.md),
[*Tạo bộ cân bằng tải cho kube-apiserver*](../08-high-availability-vi.md#tạo-bộ-cân-bằng-tải-cho-kube-apiserver-create-load-balancer-for-kube-apiserver)
— bốn yêu cầu, và cách đọc kết quả `nc -zv`; `platform/ha-lab/README.md` trong repo — Lab 8b ghi cách
dựng load balancer.

**Bàn giao:** `/etc/haproxy/haproxy.cfg` trên `lab-ha-lb`; `~/lab-evidence/8c/b3-*` trên `lab-ha-lb`
và `lab-ha-4`; mục *Load balancer* trong runbook.

**Nghiệm thu:**

- `PASS:` HAProxy `active`, nghe `6443` và `8404`; đã lưu cấu hình của Lab 8b.
- `PASS:` đúng hai backend `.234`, `.235`; không backend nào trỏ `.231`–`.233`.
- `PASS:` từ `lab-ha-4`, kết nối tới `192.168.100.230:6443` không timeout; trang thống kê liệt kê
  đúng `lab-ha-4`, `lab-ha-5`.

**Câu hỏi thiết kế** — trả lời trong runbook, mục *Load balancer*:

1. Quên đổi backend thì `kubeadm init` ở B7 hỏng ra sao? Sót đúng **một** backend cũ thì sao?
2. Trước khi init, kết nối tới load balancer bị *refused* và bị *timeout* mang hai kết luận nào?
3. Vì sao health check là TCP chứ không gọi `/readyz` của apiserver?

<details>
<summary>Gợi ý</summary>

- Đọc khối của Lab 8b trước: `sed -n '/^#--- Lab 8b/,$p' /etc/haproxy/haproxy.cfg`. Ba dòng
  `server lab-ha-N …:6443 check` là thứ duy nhất phải đổi. `haproxy -c -f` kiểm cú pháp trước khi
  restart.
- Không chắc `nc` có trên bản cài minimal; `bash` có `/dev/tcp`. Mã thoát `124` của `timeout` là
  ranh giới giữa *timeout* và *bị đóng*.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

Chạy trên **`lab-ha-lb`**:

```bash
EV=~/lab-evidence/8c
systemctl is-active haproxy | grep -qx active && echo 'PASS: haproxy active'
sudo ss -lntp | grep -E ':6443|:8404' > "$EV/b3-listen.txt"
grep -q ':6443' "$EV/b3-listen.txt" && grep -q ':8404' "$EV/b3-listen.txt" \
  && echo 'PASS: LB dang nghe 6443 va trang thong ke 8404'
test "$(grep -c '192.168.100.23[45]:6443' /etc/haproxy/haproxy.cfg)" -eq 2 \
  && echo 'PASS: backend tro dung hai control plane moi'
test "$(grep -c '192.168.100.23[123]:6443' /etc/haproxy/haproxy.cfg)" -eq 0 \
  && echo 'PASS: khong con backend nao tro toi ba host etcd'
test -s "$EV/b3-haproxy-cfg-8b.bak" && echo 'PASS: da luu cau hinh cua Lab 8b'
```

Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/bien-ha.sh
timeout 3 bash -c 'echo > /dev/tcp/192.168.100.230/6443' 2>/dev/null
RC=$?
echo "ma thoat: $RC"
case "$RC" in
  124) echo 'FAIL: TIMEOUT — lab-ha-4 khong toi duoc load balancer' ;;
  *)   echo 'PASS: khong timeout — duong toi LB thong; ket noi bi dong la binh thuong vi chua co apiserver' ;;
esac

curl -s --max-time 10 "http://192.168.100.230:8404/stats;csv" \
  | awk -F, '$1=="k8s-controlplane" && $2!="BACKEND" {print $2"="$18}' > "$EV/b3-lbstat.txt"
cat "$EV/b3-lbstat.txt"
test "$(grep -c '^lab-ha-[45]=' "$EV/b3-lbstat.txt")" -eq 2 \
  && test "$(grep -c '^lab-ha-[123]=' "$EV/b3-lbstat.txt")" -eq 0 \
  && echo 'PASS: trang thong ke liet ke dung hai backend lab-ha-4, lab-ha-5'
```

Hai backend đang `DOWN` lúc này là đúng — chưa có apiserver nào. Chúng chuyển `UP` ở B7 và B8.

</details>

<details>
<summary>Lời giải tham khảo — quy trình đã kiểm chứng</summary>

Chạy trên **`lab-ha-lb`**. Ba lệnh `sed` thay đúng ba dòng `server` mà Lab 8b viết — dòng của
`lab-ha-1`, `lab-ha-2` đổi thành hai control plane mới, dòng của `lab-ha-3` bị xóa; frontend, timeout,
`default-server` và trang thống kê không đổi:

```bash
mkdir -p ~/lab-evidence/8c
EV=~/lab-evidence/8c

haproxy -v | head -n 1 | tee "$EV/b3-haproxy-version.txt"
sudo cp -a /etc/haproxy/haproxy.cfg "$EV/b3-haproxy-cfg-8b.bak"
grep -nE 'server|bind' /etc/haproxy/haproxy.cfg | tee "$EV/b3-backend-cu.txt"

sudo sed -i \
  -e 's/^    server lab-ha-1 192\.168\.100\.231:6443 check$/    server lab-ha-4 192.168.100.234:6443 check/' \
  -e 's/^    server lab-ha-2 192\.168\.100\.232:6443 check$/    server lab-ha-5 192.168.100.235:6443 check/' \
  -e '/^    server lab-ha-3 192\.168\.100\.233:6443 check$/d' \
  /etc/haproxy/haproxy.cfg
sed -n '/^#--- Lab 8b/,$p' /etc/haproxy/haproxy.cfg

sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
systemctl is-active haproxy
```

`haproxy -c` phải in `Configuration file is valid`, và khối in ra phải có đúng hai dòng `server` mới.
Nếu cấu hình của bạn lệch khuôn của Lab 8b (thụt lề khác, tên server khác) thì `sed` không khớp dòng
nào — sửa ba dòng đó bằng trình soạn thảo, rồi chạy lại script nghiệm thu.

Ý chính cho runbook:

1. Không đổi backend thì load balancer chuyển request tới ba máy không còn nghe `6443`: `kubeadm init`
   treo ở giai đoạn chờ control plane khỏe. Sót một backend cũ thì health check giữ nó `DOWN`, nhưng
   trong lúc chờ đủ số lần kiểm thất bại, request vẫn rơi vào máy etcd và trả lỗi kết nối khó hiểu.
2. Bài 08: *connection refused* là mong đợi — gói tin tới được node, chỉ chưa có ai nghe. *Timeout*
   nghĩa là load balancer không giao tiếp được với control plane — sửa trước khi đi tiếp. Chiều node →
   LB không ra *refused* vì HAProxy đã nghe `6443` (Lab 8b đã giải thích).
3. `/readyz` của apiserver qua HTTPS cần kiểm chứng đầu bên kia; load balancer chỉ chuyển tiếp TCP nên
   kiểm TCP là đúng tầng. Bài 08 viết đúng như vậy: health check cho apiserver là kiểm tra TCP trên cổng
   nó lắng nghe.

</details>

## B4. [BS-8C-04] Ba host etcd: kubelet chỉ làm trình quản lý dịch vụ

**Loại:** Vận hành — trên `lab-ha-1`, `lab-ha-2`, `lab-ha-3`.

**Bối cảnh:** ba host phải chạy etcd **trước khi có bất kỳ cluster Kubernetes nào**. Bài 07 giải bài
toán đó bằng cách trưng dụng kubelet làm trình quản lý dịch vụ cục bộ cho static Pod.

**Yêu cầu:**

1. Trên cả ba host, kubelet chạy (`active`) bằng một unit drop-in **có độ ưu tiên cao hơn** unit mà
   gói kubeadm cài, trỏ kubelet vào một file `KubeletConfiguration` riêng — đúng bước 1 của bài 07.
2. Kubelet đó chỉ nghe trên loopback, ủy quyền mọi request, đọc static Pod từ
   `/etc/kubernetes/manifests`, và không có kubeconfig nào để đăng ký với API server.
3. Cgroup driver khớp container runtime của baseline; endpoint runtime đúng chuỗi bài 07 viết.

**Ràng buộc:** file cấu hình giống hệt nhau trên ba host. Không sửa cấu hình containerd.

**Đọc lại:** bài [07](../07-setup-ha-etcd-with-kubeadm-vi.md),
[*Thiết lập cluster*](../07-setup-ha-etcd-with-kubeadm-vi.md#thiết-lập-cluster-setting-up-the-cluster)
bước 1; bài [58](../58-static-pods-vi.md) — static Pod do kubelet quản lý; mục cgroup driver ở
[A4.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a42-cài-containerd-và-runc-đúng-version).

**Bàn giao:** hai file trong `/etc/systemd/system/kubelet.service.d/` trên ba host;
`~/lab-evidence/8c/b4-*-<host>.txt`; mục *Host etcd* trong runbook.

**Nghiệm thu** — trên **cả ba** host:

- `PASS:` kubelet `active`, `ExecStart` trỏ vào `KubeletConfiguration` riêng, drop-in
  `20-etcd-service-manager.conf` đang được áp dụng.
- `PASS:` không có `/etc/kubernetes/kubelet.conf`; kubelet nghe `127.0.0.1`; `authorization.mode` là
  `AlwaysAllow`; thư mục static Pod đã có.

**Câu hỏi thiết kế** — trả lời trong runbook, mục *Host etcd*:

1. Vì sao phải tạo drop-in thay vì dùng unit mà gói kubeadm đã cài? Tiền tố `20-` và dòng
   `ExecStart=` rỗng ở đầu file làm gì?
2. Sau bước này, kubelet trên ba máy nói chuyện với API server nào? Ba máy có bao giờ xuất hiện trong
   `kubectl get nodes` của cluster Bookstore không?
3. Hai giá trị nào trong `KubeletConfiguration` bám vào baseline của Lab 00, và đổi chúng thì sao?

<details>
<summary>Gợi ý</summary>

- systemd nạp drop-in theo thứ tự tên file; file sau ghi đè file trước.
- Trong file unit, một `ExecStart=` rỗng xóa lệnh đã khai trước đó.
- `systemctl show kubelet -p ExecStart` và `systemctl cat kubelet` cho thấy unit đang được ghép thế nào.

</details>

<details>
<summary>Script nghiệm thu — chạy trên từng host etcd sau khi tự làm xong</summary>

```bash
EV=~/lab-evidence/8c; mkdir -p "$EV"
ME="$(hostname -s)"
ACT=''
for i in $(seq 1 24); do
  ACT="$(systemctl is-active kubelet)"
  test "$ACT" = 'active' && break
  sleep 5
done
echo "kubelet tren $ME: $ACT"

systemctl show kubelet -p ExecStart > "$EV/b4-execstart-$ME.txt"
systemctl cat kubelet | grep -n 'etcd-service-manager' > "$EV/b4-unit-$ME.txt"

test "$ACT" = 'active' && echo 'PASS: kubelet dang chay'
grep -q 'kubelet.service.d/kubelet.conf' "$EV/b4-execstart-$ME.txt" \
  && echo 'PASS: ExecStart da tro vao KubeletConfiguration rieng cua etcd'
test -s "$EV/b4-unit-$ME.txt" && echo 'PASS: unit 20-etcd-service-manager.conf dang duoc ap dung'

test ! -e /etc/kubernetes/kubelet.conf \
  && echo 'PASS: khong co kubelet.conf — kubelet khong dang ky voi API server nao'
grep -q 'address: 127.0.0.1' /etc/systemd/system/kubelet.service.d/kubelet.conf \
  && echo 'PASS: kubelet chi nghe tren loopback'
grep -q 'mode: AlwaysAllow' /etc/systemd/system/kubelet.service.d/kubelet.conf \
  && echo 'PASS: authorization.mode la AlwaysAllow — khong co API server de hoi'
test -d /etc/kubernetes/manifests && echo 'PASS: staticPodPath da san sang'
```

Bảy dòng `PASS:` trên mỗi host. Vòng lặp có số vòng tối đa vì thời gian kubelet lên phụ thuộc cấu
hình và phần cứng host.

</details>

<details>
<summary>Lời giải tham khảo — quy trình đã kiểm chứng</summary>

Chạy nguyên khối trên **`lab-ha-1`**, rồi **`lab-ha-2`**, rồi **`lab-ha-3`** — giống hệt nhau:

```bash
mkdir -p ~/lab-work/8c ~/lab-evidence/8c
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

Ý chính cho runbook:

1. Bài 07: **etcd được tạo trước**, nên phải có unit file có độ ưu tiên cao hơn unit kubelet do
   kubeadm cung cấp — unit đó giả định có API server để bootstrap. `20-…` đứng sau `10-kubeadm.conf`
   theo thứ tự chữ cái nên ghi đè nó; dòng `ExecStart=` rỗng xóa lệnh cũ, thiếu nó thì unit có hai
   `ExecStart` và không khởi động được.
2. Không API server nào: không kubeconfig, không đăng ký Node, nghe loopback, `AlwaysAllow`. Ba máy
   **không bao giờ** xuất hiện trong `kubectl get nodes` — cluster chỉ biết chúng qua ba địa chỉ IP.
3. `cgroupDriver: systemd` phải khớp `SystemdCgroup = true` của containerd ở A4.2 — lệch thì kubelet
   và runtime đá nhau. `containerRuntimeEndpoint` trỏ `/var/run/containerd/containerd.sock`; trên
   Ubuntu `/var/run` là symlink tới `/run`, nên đó đúng là socket mà `/etc/crictl.yaml` của A4.3 khai.

</details>

## B5. [BS-8C-05] Cấu hình và certificate cho từng etcd member

**Loại:** Vận hành — sinh trên `lab-ha-1`, phân phối sang `lab-ha-2`, `lab-ha-3`.

**Bối cảnh:** cụm etcd tĩnh hình thành từ ba file cấu hình **khác nhau** ở địa chỉ và SAN nhưng
**giống nhau** ở danh sách thành viên. Mô hình của bài 07 là sinh mọi certificate trên một host và chỉ
phân phối những file **cần thiết** — điểm học nằm ở chữ *cần thiết*.

**Yêu cầu:**

1. Trên `lab-ha-1` có ba file `kubeadmcfg.yaml`, mỗi file cho một member, theo bước 2 của bài 07:
   SAN là IP của chính member đó; danh sách `initial-cluster` liệt kê **cả ba** member; trạng thái cụm
   là cụm mới; peer ở `2380`, client ở `2379`.
2. CA của etcd sinh bằng kubeadm trên `lab-ha-1`; không công cụ mã hóa ngoài nào.
3. Mỗi host có đủ **chín** file bắt buộc của bước 6 bài 07; `server.crt` của mỗi host chỉ chứa IP của
   chính host đó trong SAN.
4. `ca.key` của etcd tồn tại **đúng một chỗ**: `/etc/kubernetes/pki/etcd/` trên `lab-ha-1`. Không
   bản sao nào nằm trong thư mục sắp gửi đi, không bản sao nào trên `lab-ha-2`, `lab-ha-3`.

**Ràng buộc:** giữ đúng thứ tự sinh của bài 07 (`HOST2` → `HOST1` → `HOST0`). Không `mv` vào
`/etc/kubernetes/pki` khi thư mục đó đã tồn tại.

**Đọc lại:** bài [07](../07-setup-ha-etcd-with-kubeadm-vi.md),
[*Trước khi bạn bắt đầu*](../07-setup-ha-etcd-with-kubeadm-vi.md#trước-khi-bạn-bắt-đầu-before-you-begin)
— hai port TCP; [*Thiết lập cluster*](../07-setup-ha-etcd-with-kubeadm-vi.md#thiết-lập-cluster-setting-up-the-cluster)
bước 2 đến 6.

**Bàn giao:** `/tmp/192.168.100.23{1,2,3}/kubeadmcfg.yaml` trên `lab-ha-1` (dọn ở B13); `pki` trên ba
host; `~/lab-evidence/8c/b5-*.txt` trên `lab-ha-1`; mục *Certificate etcd* trong runbook.

**Nghiệm thu:**

- `PASS:` ba file cấu hình đúng SAN riêng và đúng danh sách ba member chung.
- `PASS:` không bản sao `ca.key` nào trong thư mục sắp gửi đi; `ca.key` vẫn ở `lab-ha-1`.
- `PASS:` cả ba host đủ chín file bắt buộc; chỉ `lab-ha-1` giữ `ca.key`.
- `PASS:` `server.crt` trên mỗi host có SAN đúng IP của chính host đó.

**Câu hỏi thiết kế** — trả lời trong runbook, mục *Certificate etcd*:

1. Trong `kubeadmcfg.yaml`, trường nào **riêng** của từng host, trường nào **chung** cả cụm? Vì sao
   trường chung phải giống hệt nhau trên ba host?
2. Vì sao mỗi host cần `server.crt` riêng? SAN trên `lab-ha-2` phải chứa gì, và khi Bookstore đang
   chạy thì ai là bên kiểm tra SAN đó?
3. Vì sao hai lệnh xóa `ca.key` khỏi thư mục tạm là nội dung bảo mật chứ không phải dọn rác?
4. Firewall chỉ mở TCP `2379` giữa ba host, chặn `2380`. Kiểm sức khỏe từng endpoint có ra kết quả
   không? Cụm có hình thành không?

<details>
<summary>Gợi ý</summary>

- Bài 07 chạy cả bước 2 đến 5 trong một shell root. Lab dùng `sudo` từng lệnh, nên thư mục tạm thuộc
  `root` — trả quyền về user trước khi `scp`.
- `openssl x509 -noout -text -in <file>` đọc SAN. `openssl` chỉ dùng để **đọc**, không để sinh.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

Chạy trên **`lab-ha-1`**:

```bash
source ~/lab-work/8c/bien-ha.sh
HOST0="$E1"; HOST1="$E2"; HOST2="$E3"

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
cp /tmp/"$HOST0"/kubeadmcfg.yaml "$EV/b5-kubeadmcfg-host0.txt"
test "$OKC" -eq 3 \
  && echo 'PASS: ba file kubeadmcfg.yaml dung SAN rieng va dung danh sach ba member chung'

CAK_TMP="$(sudo find /tmp/${HOST1} /tmp/${HOST2} -name ca.key -type f | wc -l)"
CAK_LOCAL="$(sudo find /etc/kubernetes/pki -name ca.key -type f | wc -l)"
echo "ca.key trong thu muc se gui di: $CAK_TMP | ca.key giu lai tren HOST0: $CAK_LOCAL"
test "$CAK_TMP" -eq 0 && test "$CAK_LOCAL" -eq 1 \
  && echo 'PASS: khong ban sao ca.key nao roi lab-ha-1, ca.key cua etcd van o lai'

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
test "$OKH" -eq 3 && echo 'PASS: du chin file bat buoc tren ca ba host, va chi HOST0 giu ca.key'

for pair in "$N1 $HOST0" "$N2 $HOST1" "$N3 $HOST2"; do
  set -- $pair
  echo "== $1 (mong doi SAN chua $2)"
  ssh -o BatchMode=yes "$1" \
    "sudo openssl x509 -noout -text -in /etc/kubernetes/pki/etcd/server.crt \
       | grep -A1 'Subject Alternative Name'"
done > "$EV/b5-san.txt"
OKS=0
for pair in "$N1 $HOST0" "$N2 $HOST1" "$N3 $HOST2"; do
  set -- $pair
  ssh -o BatchMode=yes "$1" \
    "sudo openssl x509 -noout -text -in /etc/kubernetes/pki/etcd/server.crt" \
    | grep -A1 'Subject Alternative Name' | grep -q "IP Address:$2" \
    && OKS=$(( OKS + 1 )) || echo "FAIL: server.crt tren $1 khong co SAN $2"
done
test "$OKS" -eq 3 && echo 'PASS: moi server.crt chi hop le cho dia chi cua chinh host do'
```

</details>

<details>
<summary>Lời giải tham khảo — quy trình đã kiểm chứng</summary>

**Bước 2 của bài 07 — ba file cấu hình.** Chạy trên **`lab-ha-1`**, đây là `$HOST0` cho toàn bộ ticket:

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

**Bước 3 và 4 — CA rồi certificate từng member**, đúng thứ tự `HOST2` → `HOST1` → `HOST0`. Chạy trên
**`lab-ha-1`**, cùng shell:

```bash
sudo kubeadm init phase certs etcd-ca
sudo ls -l /etc/kubernetes/pki/etcd/ | tee "$EV/b5-ca.txt"

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

**Bước 5 — phân phối.** Bài 07 chạy trong shell root nên `scp` đọc được mọi file; ở đây thư mục tạm
đang thuộc `root`, trả quyền về user trước khi gửi. Chạy trên **`lab-ha-1`**:

```bash
sudo chown -R "$(id -u):$(id -g)" /tmp/${HOST1} /tmp/${HOST2}
USER=ubuntu
for H in "$HOST1" "$HOST2"; do
  scp -r /tmp/${H}/* ${USER}@${H}:
done
```

Chạy **trên `lab-ha-2`**, rồi lặp lại y hệt **trên `lab-ha-3`**:

```bash
test ! -e /etc/kubernetes/pki \
  && echo 'PASS: chua co /etc/kubernetes/pki — an toan de di chuyen vao' \
  || echo 'FAIL: da ton tai /etc/kubernetes/pki, quay lai B2'

sudo chown -R root:root ~/pki
sudo mv ~/pki /etc/kubernetes/
sudo chmod 600 /etc/kubernetes/pki/apiserver-etcd-client.key
sudo find /etc/kubernetes/pki -name '*.key' -exec chmod 600 {} \;
```

Dòng `PASS:` phải xuất hiện **trước** khi chạy `mv`. Thấy `FAIL:` thì dừng: `mv` vào thư mục đã tồn
tại sẽ tạo `/etc/kubernetes/pki/pki` và mọi bước sau hỏng theo.

Ý chính cho runbook:

| Trường | Riêng hay chung | Vì sao |
| --- | --- | --- |
| `nodeRegistration.name`, `extraArgs.name` | riêng | định danh member trong cụm etcd |
| `serverCertSANs`, `peerCertSANs` | riêng | certificate của member chỉ hợp lệ cho **địa chỉ của chính nó** |
| `listen-peer-urls`, `initial-advertise-peer-urls` (`:2380`) | riêng | nơi member nhận và quảng bá lưu lượng **giữa các member** |
| `listen-client-urls`, `advertise-client-urls` (`:2379`) | riêng | nơi **client** — `kube-apiserver`, `etcdctl` — kết nối vào |
| `initial-cluster` | **chung** | cả ba phải thấy **cùng một** danh sách thì cụm mới hình thành |
| `initial-cluster-state: new` | **chung** | khai đây là cụm mới, không phải member join vào cụm có sẵn |

- TLS xác minh danh tính của **địa chỉ được kết nối tới**. Khi Bookstore chạy, chính
  `kube-apiserver` trên `lab-ha-4`, `lab-ha-5` mở kết nối tới `https://192.168.100.232:2379` và từ chối
  nếu certificate không có `192.168.100.232` trong SAN. Ba địa chỉ thì ba certificate.
- CA key **ký được certificate mới**; ai cầm nó tự cấp được danh tính hợp lệ cho cụm etcd — tức đọc,
  ghi được toàn bộ trạng thái cluster, gồm cả Secret `db-credentials`. Member chỉ cần certificate đã ký
  của chính nó cộng `ca.crt`. Lệnh `find … -not -name ca.crt -not -name ca.key -delete` giữa các vòng
  là cùng logic: dọn certificate của host trước để không phát nhầm sang host sau.
- `2380` là port peer, `2379` là port client. Chặn `2380` thì từng endpoint vẫn trả lời qua `2379`
  nhưng ba member không nói chuyện được với nhau, không bầu được leader, cụm không hình thành. Bài 07
  yêu cầu **cả hai** port thông; A4.4 đã tắt UFW nên điều kiện đó có sẵn.

</details>

## B6. [BS-8C-06] Cụm etcd ba member chạy trước mọi cluster

**Loại:** Vận hành — trên `lab-ha-1`, `lab-ha-2`, `lab-ha-3`.

**Bối cảnh:** ba bộ certificate và ba file cấu hình phải thành ba etcd đang chạy, hợp thành **một**
cụm — trước khi có API server nào. Đây cũng là lần đầu lab dùng `etcdctl`, nên giới hạn "chỉ đọc" bắt
đầu có hiệu lực từ đây.

**Yêu cầu:**

1. Static Pod manifest của etcd do kubeadm sinh trên cả ba host (bước 7 của bài 07); image trong
   manifest đúng là image mà kubeadm khai cho phiên bản Kubernetes của baseline — **không ai ghim tay**.
2. Image đã có sẵn trên cả ba host trước khi sinh manifest; ba container etcd ở trạng thái chạy.
3. Manifest cho thấy peer ở `2380`, client ở `2379`.
4. Kiểm sức khỏe cụm bằng đúng ba cờ certificate của bước 8 bài 07, chỉ bằng ba lệnh đọc: ba member
   `started` đúng ba tên đã khai, ba endpoint healthy, đúng **một** leader.
5. Chứng minh chưa có Kubernetes: `lab-ha-1` không có `admin.conf`, không có manifest apiserver, và chỉ
   chạy đúng một container. Ghi ảnh chụp cụm etcd vào evidence.

**Ràng buộc:** `etcdctl` chỉ `member list`, `endpoint health`, `endpoint status`. Không kéo image
`etcdctl` riêng.

**Đọc lại:** bài [07](../07-setup-ha-etcd-with-kubeadm-vi.md),
[*Trước khi bạn bắt đầu*](../07-setup-ha-etcd-with-kubeadm-vi.md#trước-khi-bạn-bắt-đầu-before-you-begin)
— gạch đầu dòng về `kubeadm config images list/pull`;
[*Thiết lập cluster*](../07-setup-ha-etcd-with-kubeadm-vi.md#thiết-lập-cluster-setting-up-the-cluster)
bước 7 và 8; [A5.1 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane) — vì sao giữ cờ
`--kubernetes-version`.

**Bàn giao:** `/etc/kubernetes/manifests/etcd.yaml` trên ba host; `~/lab-evidence/8c/b6-*.txt` trên
`lab-ha-1`; mục *Cụm etcd* trong runbook.

**Nghiệm thu:**

- `PASS:` ba host có image etcd kubeadm khai, có `etcd.yaml`, và tag trong manifest khớp tag kubeadm khai.
- `PASS:` ba container etcd đang chạy; manifest khai `2380` cho peer, `2379` cho client.
- `PASS:` ba member `started` đúng tên; ba endpoint healthy; đúng một leader.
- `PASS:` `lab-ha-1` chưa có cluster Kubernetes: không `admin.conf`, không `kube-apiserver.yaml`, một
  container.

**Câu hỏi thiết kế** — trả lời trong runbook, mục *Cụm etcd*:

1. Vì sao version etcd không nằm trong bảng A1.3, và vì sao lấy danh sách image phải kèm phiên bản
   Kubernetes cụ thể?
2. `etcdctl` ở đâu ra mà không cần kéo image nào? Ba cờ certificate trong lệnh kiểm sức khỏe để làm gì?
3. Chạy xong bước 8, bạn đã có cluster Kubernetes chưa? Ba máy này có phải Node của cluster Bookstore
   không, và bài 06 gọi tình trạng đó bằng chữ gì?

<details>
<summary>Gợi ý</summary>

- `kubeadm config images list --kubernetes-version …` in đủ image của control plane; lọc dòng etcd.
- Static Pod của kubeadm mount `/etc/kubernetes/pki/etcd` vào đúng đường dẫn đó trong container, nên
  `crictl exec` vào container etcd dùng được nguyên đường dẫn certificate trên host.
- `etcdctl` của etcd 3.5 trở lên mặc định dùng API v3; biến `ETCDCTL_API=3` của bài 07 không bắt buộc.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

Chạy trên **`lab-ha-1`**:

```bash
source ~/lab-work/8c/bien-ha.sh
HOST0="$E1"

KV="$(kubeadm version -o short)"
ETCD_IMG="$(sudo kubeadm config images list --kubernetes-version "$KV" 2>/dev/null | grep '/etcd:')"
echo "$ETCD_IMG" > "$EV/b6-etcd-image-khai.txt"
test -n "$ETCD_IMG" && echo "PASS: kubeadm $KV khai image etcd: $ETCD_IMG"

OKI=0; OKM=0; OKT=0
for n in "$N1" "$N2" "$N3"; do
  ssh -o BatchMode=yes "$n" "sudo crictl images -q '$ETCD_IMG'" | grep -q . && OKI=$(( OKI + 1 ))
  M="$(ssh -o BatchMode=yes "$n" 'sudo test -f /etc/kubernetes/manifests/etcd.yaml && echo 1 || echo 0')"
  test "$M" -eq 1 && OKM=$(( OKM + 1 ))
  REAL="$(ssh -o BatchMode=yes "$n" \
    "sudo grep -E '^[[:space:]]+image:' /etc/kubernetes/manifests/etcd.yaml | awk '{print \$2}'")"
  echo "$n -> image trong manifest: $REAL"
  test "$REAL" = "$ETCD_IMG" && OKT=$(( OKT + 1 )) || echo "FAIL: $n dung '$REAL', kubeadm khai '$ETCD_IMG'"
done
test "$OKI" -eq 3 && test "$OKM" -eq 3 && test "$OKT" -eq 3 \
  && echo 'PASS: ca ba host co image, co etcd.yaml, tag thuc te khop dung thu kubeadm khai'

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
test "$OKR2" -eq 3 && echo 'PASS: ca ba etcd member da chay duoi dang static Pod'

ssh "$N1" "sudo grep -E 'listen-(peer|client)-urls|advertise' /etc/kubernetes/manifests/etcd.yaml" \
  > "$EV/b6-urls-host0.txt"
grep -q "listen-peer-urls=https://$HOST0:2380" "$EV/b6-urls-host0.txt" \
  && grep -q "listen-client-urls=https://$HOST0:2379" "$EV/b6-urls-host0.txt" \
  && echo 'PASS: 2380 la port giua cac member, 2379 la port cho client'

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

ectl member list > "$EV/b6-member-list.txt"
test "$(grep -c ', started,' "$EV/b6-member-list.txt")" -eq 3 \
  && grep -q 'lab-ha-1' "$EV/b6-member-list.txt" \
  && grep -q 'lab-ha-2' "$EV/b6-member-list.txt" \
  && grep -q 'lab-ha-3' "$EV/b6-member-list.txt" \
  && echo 'PASS: cum etcd co du ba member started, dung ba ten khai o initial-cluster'

ectl endpoint health > "$EV/b6-endpoint-health.txt" 2>&1
test "$(grep -c 'is healthy' "$EV/b6-endpoint-health.txt")" -eq 3 \
  && echo 'PASS: ca ba endpoint deu healthy'

ectl endpoint status --write-out=table > "$EV/b6-endpoint-status.txt"
LEAD="$(ectl endpoint status --write-out=fields | grep -c 'IsLeader.*true')"
test "$LEAD" -eq 1 && echo 'PASS: dung mot leader trong cum'

test ! -e /etc/kubernetes/admin.conf && test ! -e /etc/kubernetes/manifests/kube-apiserver.yaml \
  && test "$(sudo crictl ps --state Running -q | wc -l)" -eq 1 \
  && echo 'PASS: chua co cluster Kubernetes — lab-ha-1 chi chay dung mot container etcd'

{
  echo "=== $(date -Is) — cum etcd external, truoc khi dung control plane ==="
  ectl member list
  ectl endpoint health
  ectl endpoint status --write-out=table
} > "$EV/b6-etcd-truoc-control-plane.txt" 2>&1
test -s "$EV/b6-etcd-truoc-control-plane.txt" && echo 'PASS: da ghi bang chung cum etcd'
```

Nếu `--write-out=fields` không được hỗ trợ ở bản `etcdctl` trong image, đọc cột `IS LEADER` trong
`b6-endpoint-status.txt` — kết luận phải là **đúng một** `true`. Vòng lặp chờ container có số vòng
tối đa vì thời gian etcd bầu leader và ổn định phụ thuộc cấu hình và phần cứng host.

</details>

<details>
<summary>Lời giải tham khảo — quy trình đã kiểm chứng</summary>

Chốt image và kéo trước lên cả ba host để bước sau không lẫn giữa "chưa tải xong" và "cấu hình sai".
Chạy trên **`lab-ha-1`**:

```bash
source ~/lab-work/8c/bien-ha.sh
KV="$(kubeadm version -o short)"
ETCD_IMG="$(sudo kubeadm config images list --kubernetes-version "$KV" 2>/dev/null | grep '/etcd:')"
echo "kubeadm $KV khai image etcd: $ETCD_IMG"
for n in "$N1" "$N2" "$N3"; do
  echo "== $n"
  ssh "$n" "sudo crictl pull '$ETCD_IMG'"
done
```

Bước 7 của bài 07. Chạy **trên `lab-ha-1`**:

```bash
export HOST0='192.168.100.231'
sudo kubeadm init phase etcd local --config=/tmp/${HOST0}/kubeadmcfg.yaml
```

Chạy **trên `lab-ha-2`**, rồi **trên `lab-ha-3`** — file cấu hình nằm ở home vì `scp` của B5 đặt nó ở đó:

```bash
sudo kubeadm init phase etcd local --config=$HOME/kubeadmcfg.yaml
```

Kéo image fail gần như luôn là DNS hoặc egress — xử lý theo
[tầng 4 của A5.4 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a545-tầng-4--dns-trong-cluster-và-ra-internet).

Ý chính cho runbook:

1. Bài 07 yêu cầu host truy cập được `registry.k8s.io` **hoặc** liệt kê/pull image etcd bằng
   `kubeadm config images list/pull`: kubeadm tự chọn bản etcd hợp với phiên bản Kubernetes, nên con số
   đó không thuộc về bạn và không có chỗ trong A1.3. `--kubernetes-version` giữ kubeadm khỏi dò remote
   và chọn một patch khác binary đang cài — cùng lý do A5.1 giữ cờ đó.
2. `etcdctl` nằm sẵn trong image etcd mà kubeadm chọn; `crictl exec` vào chính container đang chạy là
   cách rẻ nhất của ý "chạy `etcdctl` trong container, làm việc trực tiếp với runtime" của bài 07. Ba
   cờ: `--cacert` để xác minh certificate etcd trình ra, cặp `--cert`/`--key` là danh tính client.
3. Chưa. Bạn có một cụm etcd ba member chạy dưới dạng static Pod do kubelet quản lý — chưa API server,
   chưa cluster. Ba máy sẽ **không bao giờ** là Node của cluster Bookstore; bài 06 gọi đó là etcd nằm
   **bên ngoài** cluster hình thành từ các node chạy control plane.

</details>

## B7. [BS-8C-07] Control plane đầu tiên với `etcd.external`

**Loại:** Vận hành — trên `lab-ha-4`, chép certificate từ `lab-ha-1`.

**Bối cảnh:** bài 08 nói dựng cluster với etcd ngoài "tương tự quy trình stacked, **ngoại trừ** việc
bạn cần thiết lập etcd trước, và cần truyền thông tin etcd vào file cấu hình kubeadm". B4–B6 đã làm
vế thứ nhất; ticket này làm vế thứ hai. Từ đây `lab-ha-4` là node quản trị.

**Yêu cầu:**

1. `lab-ha-4` có đúng ba file mà bài 08 yêu cầu chép từ một node etcd, đặt đúng đường dẫn, khóa riêng
   chỉ root đọc được; **không** có `ca.key` của etcd.
2. File cấu hình kubeadm ở `~/lab-work/8c/kubeadm-config.yaml`: phiên bản Kubernetes và dải Pod đúng
   A1.3; endpoint control plane là **load balancer**; khối etcd khai **ngoài**, đủ ba endpoint và ba
   đường dẫn certificate; không còn khối etcd cục bộ. File được kiểm **trước** khi `init`.
3. `kubeadm init` với file cấu hình và tải certificate control plane lên cluster — thành công; output
   lưu ở `~/lab-evidence/8c/b7-init.log`, rồi xóa khỏi file các dòng chứa token và certificate key.
4. Kubeconfig quản trị của user trên `lab-ha-4` trỏ tới load balancer.
5. CNI của baseline cài xong, dải mạng của nó khớp dải Pod trong file cấu hình; cluster có **đúng một**
   node — ba host etcd không xuất hiện.
6. Gọi được API server **qua load balancer**; `/readyz` trả `ok`.

**Ràng buộc:** không dùng `--certificate-key` cùng `--config`. Không ghim tay số phiên bản — nạp từ
`bien-ha.sh`. Không ngắt `kubeadm init` giữa chừng.

**Đọc lại:** bài [08](../08-high-availability-vi.md):
[*Các node etcd bên ngoài*](../08-high-availability-vi.md#các-node-etcd-bên-ngoài-external-etcd-nodes)
— ba mục con *Thiết lập cluster etcd*, *Thiết lập node control plane đầu tiên* và các bước cho node còn
lại; ghi chú về `--config` và `--certificate-key` ở
[*Các bước cho node control plane đầu tiên*](../08-high-availability-vi.md#các-bước-cho-node-control-plane-đầu-tiên-steps-for-the-first-control-plane-node);
bài [02](../02-create-cluster-kubeadm-vi.md#pod-network) — dải Pod của kubeadm và của CNI phải là một.

**Bàn giao:** `/etc/kubernetes/pki/etcd/ca.crt` và cặp `apiserver-etcd-client` trên `lab-ha-4`;
`~/lab-work/8c/kubeadm-config.yaml` (nội dung chép vào runbook, mục *Control plane external*);
`~/lab-evidence/8c/b7-*` trên `lab-ha-4`.

**Nghiệm thu:**

- `PASS:` `kubeadm` trên `lab-ha-4` đúng version A1.3.
- `PASS:` đủ ba file certificate, không có `ca.key`.
- `PASS:` file cấu hình trỏ load balancer, khai đủ ba endpoint etcd ngoài, không khai etcd cục bộ, có
  dải Pod.
- `PASS:` `kubeadm init` thành công; kubeconfig trỏ load balancer.
- `PASS:` dải mạng của Flannel khớp dải Pod; đúng một node.
- `PASS:` `/version` qua load balancer trả JSON; `/readyz` là `ok`.

**Câu hỏi thiết kế** — trả lời trong runbook, mục *Control plane external*:

1. Vì sao nhánh external **buộc** dùng file cấu hình chứ không dùng cờ dòng lệnh? Cờ nào của
   `kubeadm init` không dùng chung được với file cấu hình, và nếu cần nó thì khai ở đâu?
2. Ba file chép sang `lab-ha-4` mỗi file đóng vai gì khi `kube-apiserver` kết nối tới `2379`? Vì sao
   thiếu `ca.key` ở đây là đúng?
3. Đặt cạnh cấu hình stacked của Lab 8b: đâu là **toàn bộ** khác biệt giữa hai nhánh của bài 08?

<details>
<summary>Gợi ý</summary>

- Mẫu `ClusterConfiguration` có sẵn trong bài 08; thêm `networking.podSubnet` theo ghi chú CNI của
  chính bài đó.
- `scp` bằng user thường không đọc được khóa dưới `/etc/kubernetes`; đọc bằng `sudo` ra file tạm có
  quyền `600` rồi gửi.
- Kubeconfig cho user thường: đúng bốn lệnh của A5.1 trong Lab 00.
- Manifest Flannel lấy theo release ghi ở A1.3; ConfigMap `kube-flannel-cfg` chứa `net-conf.json`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/bien-ha.sh

test "$(kubeadm version -o short)" = "$K8S_VER" \
  && echo "PASS: kubeadm tren $(hostname -s) dung version cua A1.3" \
  || echo "FAIL: kubeadm la $(kubeadm version -o short), A1.3 ghi $K8S_VER"

OK9=0
for f in /etc/kubernetes/pki/etcd/ca.crt \
         /etc/kubernetes/pki/apiserver-etcd-client.crt \
         /etc/kubernetes/pki/apiserver-etcd-client.key; do
  sudo test -f "$f" && OK9=$(( OK9 + 1 )) || echo "FAIL: thieu $f"
done
sudo ls -l /etc/kubernetes/pki /etc/kubernetes/pki/etcd > "$EV/b7-pki.txt"
test "$OK9" -eq 3 && ! sudo test -f /etc/kubernetes/pki/etcd/ca.key \
  && echo 'PASS: du ba file certificate bai 08 yeu cau, khong co ca.key'

CFG="$WK/kubeadm-config.yaml"
cp "$CFG" "$EV/b7-kubeadm-config.txt"
grep -q "controlPlaneEndpoint: \"${LB}:6443\"" "$CFG" \
  && echo 'PASS: controlPlaneEndpoint tro toi load balancer, khong tro toi mot control plane node'
test "$(grep -c 'https://192.168.100.23[123]:2379' "$CFG")" -eq 3 && echo 'PASS: khai du ba endpoint etcd'
grep -q '^    external:' "$CFG" && echo 'PASS: dung khoi etcd.external'
grep -q '^    local:' "$CFG" && echo 'FAIL: con khoi etcd.local — day la cau hinh stacked' \
  || echo 'PASS: khong co khoi etcd.local'
grep -qE 'podSubnet: [0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/[0-9]+' "$CFG" && echo 'PASS: podSubnet da duoc dien tu A1.3'

grep -q 'Your Kubernetes control-plane has initialized successfully' "$EV/b7-init.log" \
  && echo 'PASS: kubeadm init thanh cong'
SRV="$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
test "$SRV" = "https://${LB}:6443" && echo 'PASS: kubeconfig tro toi load balancer, dung ControlPlaneEndpoint'

kubectl -n kube-flannel rollout status daemonset/kube-flannel-ds --timeout=300s
NET="$(kubectl -n kube-flannel get configmap kube-flannel-cfg \
  -o jsonpath='{.data.net-conf\.json}' | tr -d ' \n')"
echo "$NET" | grep -q "\"Network\":\"${POD_CIDR}\"" \
  && echo 'PASS: dai Pod network cua Flannel khop podSubnet trong kubeadm-config.yaml'

kubectl wait --for=condition=Ready node --all --timeout=300s
kubectl get nodes -o wide > "$EV/b7-node-1.txt"
test "$(kubectl get nodes --no-headers | wc -l)" -eq 1 \
  && echo 'PASS: dung mot node — ba host etcd KHONG xuat hien trong cluster'

curl -sk "https://${LB}:6443/version" > "$EV/b7-version-qua-lb.txt"
grep -q 'gitVersion' "$EV/b7-version-qua-lb.txt" && echo 'PASS: goi duoc API server qua load balancer'
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
```

</details>

<details>
<summary>Lời giải tham khảo — quy trình đã kiểm chứng</summary>

**Chép ba file certificate.** Chạy trên **`lab-ha-1`**:

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
root còn lab dùng `sudo` từng lệnh. Nội dung file không đổi.

Chạy **trên `lab-ha-4`** — đặt file vào đúng chỗ và khóa quyền:

```bash
sudo mkdir -p /etc/kubernetes/pki/etcd
sudo mv ~/ca.crt /etc/kubernetes/pki/etcd/ca.crt
sudo mv ~/apiserver-etcd-client.crt /etc/kubernetes/pki/apiserver-etcd-client.crt
sudo mv ~/apiserver-etcd-client.key /etc/kubernetes/pki/apiserver-etcd-client.key
sudo chown -R root:root /etc/kubernetes/pki
sudo chmod 600 /etc/kubernetes/pki/apiserver-etcd-client.key
```

**File cấu hình.** Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/bien-ha.sh
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
cat "$WK/kubeadm-config.yaml"
```

Chạy phần kiểm file cấu hình của script nghiệm thu **trước** bước sau: file sai thì `kubeadm init` chỉ
phát hiện ra sau khi đã đổi trạng thái máy.

**Init và kubeconfig.** Chạy **trên `lab-ha-4`**, không ngắt giữa chừng:

```bash
source ~/lab-work/8c/bien-ha.sh
sudo kubeadm init --config "$WK/kubeadm-config.yaml" --upload-certs | tee "$EV/b7-init.log"

mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
chmod 600 "$HOME/.kube/config"
```

`b7-init.log` chứa token và certificate key — chúng là bí mật. Sau khi script nghiệm thu đọc xong,
xóa hai dòng `kubeadm join` khỏi file: `sed -i '/kubeadm join\|--discovery-token\|--certificate-key/d'
"$EV/b7-init.log"`.

Init dừng ở giai đoạn chờ control plane khỏe thì nguyên nhân thường gặp nhất là load balancer chưa đổi
backend — chạy lại script nghiệm thu của B3 trước khi làm gì khác.

**CNI.** Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/bien-ha.sh
kubectl apply -f \
  "https://github.com/flannel-io/flannel/releases/download/${FLANNEL_VER}/kube-flannel.yml"
kubectl -n kube-flannel rollout status daemonset/kube-flannel-ds --timeout=300s
```

Ý chính cho runbook:

1. Không cờ dòng lệnh nào của `kubeadm init` khai được khối `etcd.external`, nên file cấu hình là bắt
   buộc. Bài 08: `--config` và `--certificate-key` không dùng chung — cần khóa tự chọn thì đặt trường
   `certificateKey` trong `InitConfiguration` và `JoinConfiguration: controlPlane`. Lab dùng
   `--upload-certs` rồi sinh khóa mới ở B8, nên không vướng ràng buộc đó.
2. `ca.crt` để **xác minh** certificate mà etcd trình ra; cặp `apiserver-etcd-client` là **danh tính
   client** của apiserver. Một client không cần khả năng ký certificate mới.
3. Bài 08 ghi chú thẳng: khác biệt là external cần file cấu hình với endpoint etcd dưới `etcd.external`;
   với stacked, "điều này được quản lý tự động". Mọi thứ khác — load balancer, `--upload-certs`, CNI,
   join — giống hệt Lab 8b.

</details>

## B8. [BS-8C-08] Control plane thứ hai và ba bằng chứng "external"

**Loại:** Vận hành, kèm kiểm chứng — trên `lab-ha-4` và `lab-ha-5`.

**Bối cảnh:** cluster cần control plane thứ hai để có thứ mà tắt ở B11. Sau đó, ba nguồn độc lập —
Kubernetes, đĩa của node, cấu hình apiserver — phải cùng chỉ về một kết luận: etcd nằm ngoài.

**Yêu cầu:**

1. `lab-ha-5` join làm control plane bằng một certificate key **sinh mới** — không phụ thuộc khoảng
   cách thời gian từ lúc init.
2. Cluster có hai node, mỗi node đúng một bộ `kube-apiserver`, `kube-controller-manager`,
   `kube-scheduler`.
3. CoreDNS được cân bằng lại theo ghi chú của bài 08; ghi lại Pod CoreDNS nằm ở đâu trước và sau.
4. Ba bằng chứng external: **không** Pod `component=etcd` nào; **không** `etcd.yaml` trên đĩa của hai
   control plane; `--etcd-servers` của **cả hai** apiserver liệt kê **cả ba** etcd host, không cái nào
   trỏ loopback.
5. Cấu hình cluster mà kubeadm lưu trong cluster khai etcd ngoài và endpoint là load balancer.
6. Ảnh chụp "sau" của cluster external nằm cạnh ảnh chụp "trước" của B1. Token và certificate key
   **không** nằm trong evidence.

**Ràng buộc:** join từng node một. Không ghi token, certificate key vào `~/lab-evidence/`.

**Đọc lại:** bài [08](../08-high-availability-vi.md#các-node-etcd-bên-ngoài-external-etcd-nodes) — các
bước cho control plane còn lại, hạn hai giờ của khóa giải mã, ghi chú CoreDNS; bài
[06](../06-ha-topology-vi.md#external-etcd-topology) — mỗi etcd host giao tiếp với `kube-apiserver` của
**từng** control plane node; bài [58](../58-static-pods-vi.md#mirror-pods) — mirror Pod.

**Bàn giao:** `~/lab-evidence/8c/b8-*` trên `lab-ha-4`; mục `## External (Lab 8c)` của ADR topology.

**Nghiệm thu:**

- `PASS:` hai node, hai bộ ba thành phần control plane.
- `PASS:` CoreDNS đã rollout lại (trải hai node hay không đều được ghi nhận, không bắt buộc).
- `PASS:` 0 Pod `component=etcd`; không control plane nào có `etcd.yaml`.
- `PASS:` cả hai apiserver trỏ tới cả ba etcd host, không loopback, đúng ba cờ certificate.
- `PASS:` `ClusterConfiguration` trong cluster khai `external`, đủ ba endpoint, endpoint là LB.
- `PASS:` evidence không chứa token hay certificate key.

**Câu hỏi thiết kế** — trả lời trong ADR topology, mục `## External (Lab 8c)`:

1. Đặt hai dòng `--etcd-servers` của cluster này cạnh ba dòng của B1. Chỉ nhìn cấu hình đó, bạn phân
   biệt được hai topology không — và quan hệ nào hiện ra?
2. Vì sao cluster có 0 Pod `component=etcd` trong khi etcd rõ ràng đang chạy?
3. ConfigMap `kubeadm-config` khai `external`. Ai đọc nó về sau, và vì sao điều đó quan trọng hơn file
   `kubeadm-config.yaml` còn nằm trong `~/lab-work/8c/`?
4. Vì sao không biến "CoreDNS trải đều hai node" thành gate cứng?

<details>
<summary>Gợi ý</summary>

- Bài 08 chỉ cách tải lại certificate và tạo khóa giải mã mới trên một control plane đã join; `kubeadm
  token create --print-join-command` sinh phần còn lại của lệnh join.
- Cờ `--etcd-*` của apiserver đọc từ static Pod manifest trên đĩa từng control plane.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/bien-ha.sh

kubectl wait --for=condition=Ready node --all --timeout=600s
kubectl get nodes -o wide > "$EV/b8-nodes.txt"
NODE_N="$(kubectl get nodes --no-headers | wc -l)"
CP_N="$(kubectl get nodes -l 'node-role.kubernetes.io/control-plane' --no-headers | wc -l)"
API_N="$(kubectl -n kube-system get pod -l component=kube-apiserver --no-headers | wc -l)"
CM_N="$(kubectl -n kube-system get pod -l component=kube-controller-manager --no-headers | wc -l)"
SC_N="$(kubectl -n kube-system get pod -l component=kube-scheduler --no-headers | wc -l)"
echo "node=$NODE_N control plane=$CP_N apiserver=$API_N controller-manager=$CM_N scheduler=$SC_N"
test "$NODE_N" -eq 2 && test "$CP_N" -eq 2 \
  && test "$API_N" -eq 2 && test "$CM_N" -eq 2 && test "$SC_N" -eq 2 \
  && echo 'PASS: hai control plane node, moi node mot bo ba thanh phan control plane'

kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide > "$EV/b8-coredns-sau.txt"
DNS_NODES="$(kubectl -n kube-system get pods -l k8s-app=kube-dns \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | wc -l)"
test -n "$(kubectl -n kube-system get deploy coredns \
  -o jsonpath='{.spec.template.metadata.annotations.kubectl\.kubernetes\.io/restartedAt}')" \
  && echo 'PASS: CoreDNS da duoc rollout lai'
test "$DNS_NODES" -eq 2 && echo 'GHI NHAN: hai Pod CoreDNS nam tren hai node khac nhau' \
  || echo 'GHI NHAN: hai Pod CoreDNS van cung mot node — hop le, scheduler khong bat buoc trai deu'

ETCD_POD_N="$(kubectl -n kube-system get pod -l component=etcd --no-headers 2>/dev/null | wc -l)"
sudo ls -1 /etc/kubernetes/manifests/ > "$EV/b8-manifests-cp1.txt"
ssh "$N5" 'sudo ls -1 /etc/kubernetes/manifests/' > "$EV/b8-manifests-cp2.txt"
OKM=0
for f in "$EV/b8-manifests-cp1.txt" "$EV/b8-manifests-cp2.txt"; do
  grep -q 'etcd.yaml' "$f" && echo "FAIL: $f co etcd.yaml" || OKM=$(( OKM + 1 ))
  grep -q 'kube-apiserver.yaml' "$f" || echo "FAIL: $f thieu kube-apiserver.yaml"
done
test "$ETCD_POD_N" -eq 0 && test "$OKM" -eq 2 \
  && echo 'PASS: 0 Pod etcd, khong control plane node nao chay static Pod etcd'

{
  echo "== lab-ha-4"
  sudo grep -E '^[[:space:]]+- --etcd-' /etc/kubernetes/manifests/kube-apiserver.yaml
  echo "== lab-ha-5"
  ssh "$N5" "sudo grep -E '^[[:space:]]+- --etcd-' /etc/kubernetes/manifests/kube-apiserver.yaml"
} > "$EV/b8-etcd-flags.txt"
SRV_LINES="$(grep -c -- '--etcd-servers=' "$EV/b8-etcd-flags.txt")"
BOTH3="$(grep -- '--etcd-servers=' "$EV/b8-etcd-flags.txt" \
  | grep -c "https://${E1}:2379,https://${E2}:2379,https://${E3}:2379")"
LOOP="$(grep -c -- '--etcd-servers=https://127.0.0.1:2379' "$EV/b8-etcd-flags.txt")"
CAF="$(grep -c -- '--etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt' "$EV/b8-etcd-flags.txt")"
CRT="$(grep -c -- '--etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt' "$EV/b8-etcd-flags.txt")"
KEY="$(grep -c -- '--etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key' "$EV/b8-etcd-flags.txt")"
echo "etcd-servers=$SRV_LINES | du ba host=$BOTH3 | loopback=$LOOP | cafile=$CAF certfile=$CRT keyfile=$KEY"
test "$SRV_LINES" -eq 2 && test "$BOTH3" -eq 2 && test "$LOOP" -eq 0 \
  && test "$CAF" -eq 2 && test "$CRT" -eq 2 && test "$KEY" -eq 2 \
  && echo 'PASS: CA HAI apiserver deu tro toi CA BA host etcd — quan he nhieu-nhieu'

kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' \
  > "$EV/b8-clusterconfig.txt"
grep -q 'external:' "$EV/b8-clusterconfig.txt" && ! grep -q '  local:' "$EV/b8-clusterconfig.txt" \
  && test "$(grep -c 'https://192.168.100.23[123]:2379' "$EV/b8-clusterconfig.txt")" -eq 3 \
  && grep -q "controlPlaneEndpoint: ${LB}:6443" "$EV/b8-clusterconfig.txt" \
  && echo 'PASS: ClusterConfiguration luu trong cluster khai etcd.external va endpoint la LB'

{
  echo "=== $(date -Is) — cluster external etcd, sau khi dung xong ==="
  echo '--- nodes'; kubectl get nodes -o wide
  echo '--- control plane pods'; kubectl -n kube-system get pods -l tier=control-plane -o wide
  echo '--- so Pod component=etcd'; echo "$ETCD_POD_N"
  echo '--- etcd flags'; cat "$EV/b8-etcd-flags.txt"
  echo '--- ClusterConfiguration'; cat "$EV/b8-clusterconfig.txt"
} > "$EV/b8-sau-external.txt"
grep -rqE -- '--certificate-key [0-9a-f]{64}|--token [a-z0-9]{6}\.' "$EV" \
  && echo 'FAIL: evidence co token hoac certificate key — xoa ngay' \
  || echo 'PASS: evidence khong chua token hay certificate key'
```

</details>

<details>
<summary>Lời giải tham khảo — quy trình đã kiểm chứng</summary>

**Sinh lệnh join, không phụ thuộc thời gian.** Chạy trên **`lab-ha-4`**:

```bash
CERT_KEY="$(sudo kubeadm init phase upload-certs --upload-certs | tail -n 1)"
JOIN_BASE="$(kubeadm token create --print-join-command)"
JOIN_CP="$JOIN_BASE --control-plane --certificate-key $CERT_KEY"
echo "$JOIN_CP"

echo "$JOIN_CP" | grep -q -- '--control-plane' && echo 'PASS: lenh join co co --control-plane'
echo "$JOIN_CP" | grep -q -- '--certificate-key' && echo 'PASS: lenh join co khoa giai ma'
echo "$JOIN_CP" | grep -q "192.168.100.230:6443" && echo 'PASS: lenh join tro toi load balancer'
```

Certificate key và token là bí mật: bài 08 cảnh báo khóa "cho phép truy cập vào dữ liệu nhạy cảm của
cluster", còn ai có token đều thêm node được. Không ghi chúng vào evidence.

**Join.** Chạy **trên `lab-ha-5`**, dán nguyên chuỗi `JOIN_CP` vừa in, chạy bằng `sudo`:

```bash
sudo kubeadm join 192.168.100.230:6443 \
  --token <token-that> \
  --discovery-token-ca-cert-hash sha256:<hash-that> \
  --control-plane --certificate-key <key-that>
```

Output phải có `This node has joined the cluster and a new control plane instance was created`.

**Cân bằng lại CoreDNS.** Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/bien-ha.sh
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide | tee "$EV/b8-coredns-truoc.txt"
kubectl -n kube-system rollout restart deployment coredns
kubectl -n kube-system rollout status deployment coredns --timeout=300s
```

Ý chính cho ADR, mục `## External (Lab 8c)`:

1. Phân biệt được. Stacked: ba dòng `--etcd-servers=https://127.0.0.1:2379` — quan hệ **một-một**.
   External: hai dòng, mỗi dòng liệt kê **cả ba** địa chỉ — quan hệ **nhiều-nhiều**, đúng câu "mỗi
   etcd host giao tiếp với `kube-apiserver` của từng control plane node". Cùng một trường cấu hình, hai
   hình dạng dữ liệu. Ghi con số: **external = 2 apiserver, 3 etcd member, 5 máy**.
2. etcd không còn là workload của cluster này: kubelet trên ba host etcd không đăng ký Node, nên static
   Pod của chúng không có mirror Pod trong API server để mà đếm. Stacked có **3** Pod như vậy, external
   có **0**.
3. Các lệnh kubeadm về sau — `join`, `upgrade`, `reset` — đọc cấu hình từ ConfigMap đó, không từ file
   của bạn. Cluster tự khai nó dùng etcd ngoài; node join sau này được cấu hình theo khai báo đó, và
   `reset` trên `lab-ha-4`/`lab-ha-5` biết không được xóa dữ liệu etcd.
4. Bài 08 chỉ nói rollout restart là cách "cung cấp tính sẵn sàng cao hơn", không hứa scheduler trải
   đều. Gate phải kiểm điều tài liệu cam kết, không kiểm điều mình mong.

</details>

## B9. [BS-8C-09] Triển khai Bookstore lên cluster external etcd

**Loại:** Tính năng.

**Bối cảnh:** Lab 8b đo sẵn sàng của Bookstore trên một cluster stacked **có worker riêng**. Cluster
này không có worker: sáu máy đã dùng hết cho 3 etcd + 2 control plane + 1 LB. Muốn có số đo đặt cạnh
số đo của Lab 8b, Bookstore phải chạy được ở đây — dựng lại **chỉ từ repo**, bằng đúng overlay và lệnh
mà Lab 8b bàn giao. Đây cũng là bước "dựng lại hệ thống từ repo" mà B0 đã dời sang.

Overlay `ha-lab` của Lab 8b cố ý **gỡ mọi ghim node** của `db` và `db-backup`: trên cluster stacked,
scheduler chọn worker nào cũng được. Ở đây điều đó không còn đủ — xem câu hỏi thiết kế 1.

**Yêu cầu:**

1. Hai control plane node nhận được workload: taint cách ly control plane được gỡ trên **cả hai**,
   trạng thái taint trước và sau ghi vào `~/lab-evidence/8c/b9-taint-truoc.txt` và `b9-taint-sau.txt`.
   Đây là thỏa hiệp của môi trường lab; B13 trả lại.
2. Bookstore triển khai **chỉ** bằng `./scripts/deploy-ha-lab.sh`, chạy được hai lần liên tiếp không
   lỗi; `./tests/smoke-ha-lab.sh` không có dòng `FAIL:`.
3. `db-0` chạy trên **`lab-ha-4`** — node không bao giờ bị tắt trong sprint này. Thay đổi duy nhất của
   repo là **một patch thêm vào overlay** `k8s/overlays/ha-lab/`, chồng lên các patch của Lab 8b mà
   không phá chúng: render vẫn không có Ingress, PVC, claim template, ghim node của chuỗi chính.
   `k8s/app/`, `k8s/cluster/`, `scripts/`, `tests/`, `platform/` giống hệt tag `lab-8b`.
4. `web` có Pod chạy trên **cả hai** node.
5. Từ `lab-ha-4` và từ máy host, `GET http://192.168.100.234:30080/api/books` trả `200` cùng danh sách
   sách.
6. `~/lab-work/8c/ham-do.sh` đặt đúng chỗ (công cụ, chép nguyên văn ở dưới). Đường nền đo bằng
   `scripts/do-san-sang.ps1` của Lab 8b trong 120 giây: ít nhất 100 mẫu, ứng dụng và API đều ≥ 99%.

**Ràng buộc:** không lệnh imperative nào lên object của Bookstore — chỉ gỡ taint là thao tác trên
node. Không sửa file nào của Lab 8b trong overlay; chỉ **thêm** một file patch và một mục vào
`patches`. Không đặt Pod nào lên ba host etcd (chúng không phải Node — thấy điều đó là đã sai từ B4).

**Đọc lại:**

- Bài [02](../02-create-cluster-kubeadm-vi.md),
  [*Cách ly control plane node*](../02-create-cluster-kubeadm-vi.md#cách-ly-control-plane-node-control-plane-node-isolation)
  — lệnh gỡ taint và lý do mặc định không lập lịch Pod lên control plane.
- Bài [139](../139-taint-and-toleration-vi.md#concepts) — taint và toleration, hai cách để một Pod
  lên được node có taint.
- Bài [322](../322-kustomization-vi.md): [*Tùy biến*](../322-kustomization-vi.md#tùy-biến-customizing)
  — `patches` áp **theo thứ tự khai báo**, StrategicMerge và Json6902, trường `target`;
  [*Base và Overlay*](../322-kustomization-vi.md#base-và-overlay-bases-and-overlays).
- `k8s/overlays/ha-lab/kustomization.yaml` và `db-emptydir.yaml` trong repo — Lab 8b đã xóa trường
  nào của `db`, bằng cơ chế gì.

**Bàn giao:** một file patch và một mục `patches` mới trong `k8s/overlays/ha-lab/`; mục *Triển khai
Bookstore trên cluster không có worker* trong `docs/runbook-ha-external-etcd.md`;
`~/lab-evidence/8c/b9-*`, `do-nen.csv`.

**Nghiệm thu:**

- `PASS:` trước đó cả hai node mang taint control plane; giờ không node nào mang.
- `PASS:` ngoài `docs/` và `k8s/overlays/ha-lab/`, repo giống hệt tag `lab-8b`; các file patch của
  Lab 8b không đổi.
- `PASS:` render không có Ingress, PVC, claim template, ghim node của chuỗi chính; `db` ghim
  `lab-ha-4`.
- `PASS:` `deploy-ha-lab.sh` thành công hai lần liên tiếp; `smoke-ha-lab.sh` không có dòng `FAIL:`.
- `PASS:` `db-0` trên `lab-ha-4`; `web` có Pod trên hai node; `/api/books` qua NodePort trả sách.
- `PASS:` đường nền từ máy host: ít nhất 100 mẫu, ứng dụng và API đều ≥ 99%.

Hàm đo phía cluster, chép nguyên văn trên **`lab-ha-4`**. `tom_tat` là đúng công thức trong `bien.sh`
của Lab 8b — cùng cách tính thì hai bảng số đo mới so được:

```bash
cat > ~/lab-work/8c/ham-do.sh <<'EOF'
# Cong cu do cua Lab 8c tren lab-ha-4. Nap bang: source ~/lab-work/8c/ham-do.sh
source ~/lab-work/8c/bien-ha.sh
source ~/lab-work/8c/bien-du-an.sh
APP_URL="http://${CP1}:30080/api/books"
EPS="https://${E1}:2379,https://${E2}:2379,https://${E3}:2379"

# Control plane: /readyz qua load balancer (kubeconfig tro toi LB).
cp_ok () { test "$(kubectl --request-timeout=15s get --raw='/readyz' 2>/dev/null)" = 'ok'; }

# Ung dung: /api/books qua NodePort tren lab-ha-4 — duong nay KHONG di qua API server.
app_ok () { curl -s --max-time 5 "$APP_URL" 2>/dev/null | grep -q '"title"'; }

# Suc khoe cum etcd, hoi qua lab-ha-1 — may nay khong bao gio bi tat.
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

# Tom tat file do "ms,app,api" cua scripts/do-san-sang.ps1 — cung cong thuc voi bien.sh cua Lab 8b.
# Dat MAU; APP, API (% san sang tinh theo THOI GIAN); APP_MAX, API_MAX (giay gian doan lien tuc dai
# nhat); CUOI_APP, CUOI_API (1 neu 10 mau cuoi deu 200). In mot dong va ghi them vao $EV/so-do.txt.
tom_tat () {
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
      for (i = (n > 10 ? n - 10 : 0); i < n; i++) {
        if (a[i] != "200") ca = 0
        if (p[i] != "200") cp = 0
      }
      printf "MAU=%d APP=%.1f API=%.1f APP_MAX=%d API_MAX=%d CUOI_APP=%d CUOI_API=%d\n",
        n, 100 * (1 - ad / tot), 100 * (1 - pd / tot), am / 1000, pm / 1000, ca, cp
    }' "$1")"
  echo "$(basename "$1"): mau=$MAU app=$APP% api=$API%" \
       "| gian doan dai nhat: app=${APP_MAX}s api=${API_MAX}s" | tee -a "$EV/so-do.txt"
}
ge () { awk -v a="$1" -v b="$2" 'BEGIN { exit !(a >= b) }'; }
lt () { awk -v a="$1" -v b="$2" 'BEGIN { exit !(a < b) }'; }
EOF
source ~/lab-work/8c/ham-do.sh && type cp_ok app_ok ehealth tom_tat >/dev/null \
  && echo 'PASS: ham-do.sh nap duoc'
```

Phía máy host không có công cụ mới: `Start-Probe <Ten> <Giay>` của B0.1 mở cửa sổ riêng chạy
`do-san-sang.ps1` với `-Thu` trỏ vào `$ev` (`lab-evidence\8c`); `Send-Probe <Ten>` chép
`do-<Ten>.csv` sang `lab-ha-4`. Hai URL mặc định của script — NodePort `30080` trên `lab-ha-4`,
`/readyz` qua load balancer — vẫn đúng trên cluster external.

**Câu hỏi thiết kế** — trả lời trong runbook, mục *Triển khai Bookstore trên cluster không có worker*:

1. Vì sao `db-0` phải ở `lab-ha-4`? Nếu để scheduler chọn như Lab 8b và nó rơi vào `lab-ha-5`, kịch
   bản tắt `lab-ha-5` ở B11 đo được cái gì? Vì sao quy tắc "DB ở node được phép phá" của chuỗi chính
   lại đảo ngược ở đây?
2. Patch của bạn phải đứng **sau** `db-emptydir.yaml` của Lab 8b trong danh sách `patches`. Đặt nó
   trước thì render ra gì, vì sao?
3. Có hai cách cho Bookstore lên control plane: gỡ taint trên node, hoặc thêm toleration vào overlay.
   Mỗi cách mở cửa cho những Pod nào? Lab chọn cách thứ nhất — cái giá là gì, production làm khác thế
   nào?
4. Vì sao thay đổi nằm ở overlay chứ không ở `k8s/app`? Lab 9a sẽ hỏng ở đâu nếu bạn sửa base?

<details>
<summary>Gợi ý</summary>

- `db-emptydir.yaml` của Lab 8b đặt `nodeSelector: null` và `nodeAffinity: null` — StrategicMerge
  xóa trường bằng `null`. Kustomize áp `patches` theo thứ tự khai báo: patch sau thấy kết quả của patch
  trước.
- Patch kiểu `Json6902` với thao tác `add` tạo trường nếu chưa có và thay nếu đã có.
- Xem bản render không đụng cluster:
  `kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/overlays/ha-lab` (cần `secrets/` —
  `deploy-ha-lab.sh` sinh nó ở lần chạy đầu).
- Đường nền: `Start-Probe nen 120`, chờ cửa sổ đo in `KET QUA`, rồi `Send-Probe nen`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
cd ~/bookstore
NS=bookstore

TAINT_LEFT="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}' \
  | grep -c 'node-role.kubernetes.io/control-plane')"
grep -q 'control-plane' "$EV/b9-taint-truoc.txt" && test "$TAINT_LEFT" -eq 0 \
  && echo 'PASS: truoc do hai node mang taint control-plane, gio da go tren ca hai'

test -z "$(git diff --name-only lab-8b -- . ':!k8s/overlays/ha-lab' ':!docs')" \
  && test -z "$(git ls-files --others --exclude-standard | grep -vE '^(k8s/overlays/ha-lab|docs)/')" \
  && echo 'PASS: ngoai docs/ va k8s/overlays/ha-lab/, repo giong het tag lab-8b'
git diff --quiet lab-8b -- k8s/overlays/ha-lab/db-emptydir.yaml k8s/overlays/ha-lab/backup-emptydir.yaml \
  k8s/overlays/ha-lab/web-nodeport.yaml k8s/overlays/ha-lab/quota-nodeport.yaml \
  && echo 'PASS: cac file patch cua Lab 8b khong doi'

eval "$DEPLOY_HA" > "$EV/b9-deploy.txt" 2>&1 && echo 'PASS: deploy-ha-lab lan 1 thanh cong'
eval "$DEPLOY_HA" > "$EV/b9-deploy-lan-2.txt" 2>&1 && echo 'PASS: deploy-ha-lab lan 2 thanh cong'
eval "$SMOKE_HA" > "$EV/b9-smoke.txt" 2>&1; grep '^INFO' "$EV/b9-smoke.txt"
grep -q '^FAIL' "$EV/b9-smoke.txt" && echo 'FAIL: smoke-ha-lab con loi' \
  || echo 'PASS: smoke-ha-lab khong co dong FAIL'

kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/overlays/ha-lab > "$WK/b9-render.yaml"
grep -Eq '^kind: (Ingress|PersistentVolumeClaim|StorageClass)$|volumeClaimTemplates:|lab-k8s-|bookstore\.example\.com/data' \
  "$WK/b9-render.yaml" && echo 'FAIL: patch moi lam hong phan go cua Lab 8b' \
  || echo 'PASS: render van khong co Ingress, PVC, claim template, ghim node cua chuoi chinh'
test "$(kubectl -n "$NS" get statefulset db \
  -o jsonpath='{.spec.template.spec.nodeSelector.kubernetes\.io/hostname}')" = 'lab-ha-4' \
  && echo 'PASS: StatefulSet db ghim lab-ha-4'

kubectl -n "$NS" get pods -o wide > "$EV/b9-pods.txt"
test "$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.spec.nodeName}')" = 'lab-ha-4' \
  && echo 'PASS: db-0 chay tren lab-ha-4'
WEB_SEL='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=web'
WEB_NODES="$(kubectl -n "$NS" get pods -l "$WEB_SEL" --field-selector=status.phase=Running \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | wc -l)"
test "$WEB_NODES" -eq 2 && echo 'PASS: web co Pod tren ca hai node'
curl -s --max-time 5 "$APP_URL" > "$EV/b9-books.json"
N="$(grep -o '"title"' "$EV/b9-books.json" | wc -l)"
test "$N" -ge 5 && echo "PASS: /api/books qua NodePort tra $N cuon sach"
```

`$WK/b9-render.yaml` chứa Secret đã render — nó nằm ở `lab-work`, bị xóa ở B13, không vào evidence.

Rồi trên **máy host**: `Start-Probe nen 120`; khi cửa sổ đo in `KET QUA`: `Send-Probe nen`. Trên
**`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
head -n 1 "$EV/do-nen.csv" | tr -d '\r' | grep -qx 'ms,app,api' && echo 'PASS: file do dung dinh dang ms,app,api'
tom_tat "$EV/do-nen.csv"
test "$MAU" -ge 100 && ge "$APP" 99 && ge "$API" 99 \
  && echo "PASS: duong nen tu may host — $MAU mau, ung dung va API deu >= 99%"
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Gỡ taint, có ý thức. Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints' \
  | tee "$EV/b9-taint-truoc.txt"
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints' \
  | tee "$EV/b9-taint-sau.txt"
```

Patch ghim `db` — một thao tác, vì `db-emptydir.yaml` của Lab 8b đã xóa cả `nodeSelector` lẫn
`nodeAffinity` của chuỗi chính:

```yaml
# k8s/overlays/ha-lab/db-tren-lab-ha-4.yaml — Lab 8c: db-0 o node khong bi tat trong dien tap.
# Ap SAU db-emptydir.yaml (Lab 8b): patch do da xoa nodeSelector/nodeAffinity, patch nay them lai
# dung mot rang buoc cho cluster external. "add" tao truong neu chua co, thay neu da co.
- op: add
  path: /spec/template/spec/nodeSelector
  value:
    kubernetes.io/hostname: lab-ha-4
```

Thêm vào **cuối** danh sách `patches` của `k8s/overlays/ha-lab/kustomization.yaml`, sau mọi mục của
Lab 8b:

```yaml
# Lab 8c: cluster external khong co worker; db o node khong bi tat trong dien tap.
- target:
    group: apps
    version: v1
    kind: StatefulSet
    name: db
  path: db-tren-lab-ha-4.yaml
```

Triển khai, kiểm, commit phần thay đổi của overlay:

```bash
source ~/lab-work/8c/ham-do.sh
cd ~/bookstore
eval "$DEPLOY_HA" && eval "$SMOKE_HA"
kubectl -n bookstore get pods -o wide
git add k8s/overlays/ha-lab && git commit -m 'lab-8c: overlay ha-lab ghim db-0 vao lab-ha-4'
```

Ý chính cho runbook:

1. `db` chỉ có một bản sao, dữ liệu trên `emptyDir`: node của nó tắt là `/api/books` chết suốt thời
   gian tắt, bất kể control plane ra sao. Ở Lab 8b vị trí `db-0` không quan trọng vì diễn tập chính
   không tắt worker. Ở đây control plane **là** worker: để scheduler chọn thì `db-0` có thể rơi vào
   `lab-ha-5`, và B11 đo **mất DB** chứ không đo **mất control plane**. Chuỗi chính đặt `db` ở node
   được phép phá vì diễn tập ở đó *nhắm vào* DB; sprint này diễn tập *nhắm vào* control plane và etcd,
   nên DB phải đứng ngoài vùng phá.
2. Patch đứng trước `db-emptydir.yaml` thì `nodeSelector: null` của Lab 8b chạy sau và xóa luôn ràng
   buộc vừa thêm — render không có ghim nào, `db-0` lại tùy scheduler. Bài 322: patch áp theo thứ tự
   khai báo.
3. Gỡ taint mở control plane cho **mọi** Pod của mọi người; toleration trong overlay chỉ mở cho Pod của
   Bookstore — hẹp hơn, nhưng là thêm một thay đổi trong repo. Lab chọn gỡ taint vì đó là quy trình bài
   02 mô tả và B13 trả lại nguyên trạng. Giá phải trả: bài 02 nói mặc định không lập lịch Pod lên
   control plane **vì lý do bảo mật** — workload thoát ra được sẽ đứng cạnh `admin.conf`, khóa CA và
   cổng `10250`. Production có worker riêng và giữ taint.
4. Chuỗi chính ở Lab 9a triển khai `k8s/app` trên cluster có worker, `db` gắn với volume local-path trên
   `lab-k8s-worker2`. Ghim `lab-ha-4` vào base thì `db-0` của chuỗi chính `Pending` mãi — node đó không
   tồn tại ở bộ VM chính.

</details>

## B10. [BS-8C-10] Diễn tập: mất một etcd member, rồi mất quorum

**Loại:** Diễn tập sự cố — tắt `lab-ha-3`, rồi `lab-ha-2` và `lab-ha-3`. Không bao giờ tắt `lab-ha-1`.

**Bối cảnh:** ADR cần biết điều gì xảy ra với **người dùng Bookstore** khi tầng etcd gặp nạn, không chỉ
điều gì xảy ra với `kubectl`. Ở Lab 8b, muốn mất etcd member thì phải tắt một control plane — hai thứ
dính nhau. Ở đây tầng etcd tắt được riêng, nên lần đầu tiên đo được ảnh hưởng của **riêng** etcd.

**Yêu cầu:**

1. **Kịch bản A — mất một member** (tắt `lab-ha-3`): cụm còn đúng hai endpoint khỏe; API và Bookstore
   vẫn phục vụ; hai node của cluster vẫn `Ready`. Bật lại thì cụm về ba member khỏe. ⭐ Đo từ máy host
   suốt kịch bản.
2. **Kịch bản B — mất quorum** (tắt `lab-ha-2` và `lab-ha-3`): API ngừng phục vụ; **không** endpoint
   nào khỏe, kể cả member còn sống; Bookstore **vẫn trả lời** suốt thời gian đó. Đo từ máy host suốt
   kịch bản, giữ hai máy tắt đủ lâu để API chết liên tục ít nhất 60 giây.
3. Bật lại: cụm về ba member khỏe, API phục vụ, `SMOKE_HA` PASS — không mất object nào của Bookstore.
4. Postmortem có dòng thời gian, số đo của hai đường, so sánh với kịch bản mất quorum của Lab 8b.

**Ràng buộc:** không tắt `lab-ha-1` — nó giữ `ehealth`. Không bao giờ tắt cả ba etcd host. Không chạy
`etcdctl` ngoài ba lệnh đọc; không `member remove` để "chữa" cụm. Trong lúc mất quorum không chạy
lệnh nào ghi vào cluster. Luôn chạy hết phần bật lại ở cuối kịch bản. Dòng `GHI NHAN` ở B0.2 (nếu có)
cũng ghi vào postmortem này.

**Đọc lại:**

- Bài [08](../08-high-availability-vi.md#external-etcd), mục *Trước khi bạn bắt đầu*, nhánh *External
  etcd*: số lẻ member etcd **là một yêu cầu** để đạt quorum bỏ phiếu tối ưu.
- Bài [06](../06-ha-topology-vi.md#external-etcd-topology) — mất một etcd member "có ít tác động hơn".
- Bài [23](../23-nodes-vi.md#node-heartbeats) — nhịp tim của node; bài [35](../35-leases-vi.md#bầu-chọn-leader-leader-election)
  — Lease và bầu leader của control plane.
- Lab 8b, [B11](LAB-8B-HA-VOI-STACKED-ETCD.md#b11-bs-8b-11-diễn-tập-mất-quorum) — kịch bản `mat-quorum`
  trên cụm stacked; số đo của nó nằm ở bảng `## Số đo Lab 8b` của ADR topology.

**Bàn giao:** `docs/postmortem/8c-mat-quorum-etcd.md`; `~/lab-evidence/8c/b10-*` và
`do-mat-quorum.csv` (⭐ `do-mat-mot-etcd.csv`) trên `lab-ha-4`.

**Nghiệm thu:**

- `PASS:` mốc gốc: ba endpoint khỏe, API và Bookstore phục vụ.
- `PASS:` kịch bản A: hai endpoint khỏe, API và Bookstore phục vụ, hai node `Ready`; bật lại về ba.
- `PASS:` ⭐ file đo A: ứng dụng ≥ 99%, API ≥ 90%, mười mẫu cuối đều `200`.
- `PASS:` kịch bản B: API ngừng phục vụ, không endpoint nào khỏe, Bookstore trả lời 12/12 lần thử.
- `PASS:` bật lại: ba endpoint khỏe, API phục vụ, `SMOKE_HA` không có dòng `FAIL`.
- `PASS:` file đo B: API gián đoạn liên tục ≥ 60 giây, ứng dụng ≥ 99% (cùng ngưỡng Lab 8b), mười mẫu
  cuối đều `200`.

**Câu hỏi thiết kế** — trả lời trong `docs/postmortem/8c-mat-quorum-etcd.md`:

1. `lab-ha-1` vẫn chạy, container etcd của nó vẫn sống. Vì sao nó báo **không** khỏe? Định nghĩa
   quorum bằng chính con số của cụm bạn — đừng nói "vì hỏng".
2. Trong lúc mất quorum, người dùng Bookstore thấy gì, người vận hành mất gì? Dẫn số đo của hai đường.
   Nhờ đâu ứng dụng vẫn trả lời khi không còn API server?
3. Để mất quorum, Lab 8b phải tắt mấy máy, là máy gì; Lab 8c phải tắt mấy máy, là máy gì? Với người
   dùng Bookstore, hai lần mất quorum khác nhau không? Điều đó nói gì về chỗ topology external thật
   sự giúp và chỗ nó không giúp?
4. Ở kịch bản A, vì sao `kubectl get nodes` không đổi gì? Cụm stacked có tình huống tương đương không?

<details>
<summary>Gợi ý</summary>

- Bắt đầu lượt đo (`Start-Probe`) **trước** khi tắt máy, để file có đoạn "trước sự cố" làm nền.
  Thời lượng lượt đo phải phủ cả lúc máy boot lại.
- Thông báo lỗi của `kubectl` lúc mất quorum đến từ đâu: từ `kubectl`, từ load balancer, hay từ chính
  apiserver? Đọc kỹ evidence trước khi viết postmortem.
- `kube-proxy` ghi quy tắc chuyển tiếp Service vào kernel của từng node — bài 8b đã nói điều này ở
  kịch bản mất quorum của nó.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đọc xong; script này chính là cuộc diễn tập</summary>

**Mốc gốc.** Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
ehealth > "$EV/b10-0-health-goc.txt" 2>&1
H0="$(grep -c 'is healthy' "$EV/b10-0-health-goc.txt")"
echo "endpoint khoe truoc khi thu: $H0/3"
test "$H0" -eq 3 && cp_ok && app_ok \
  && echo 'PASS: moc goc — ba etcd member khoe, API va Bookstore phuc vu'
```

Không mở kịch bản A khi mốc gốc chưa PASS.

**Kịch bản A — tắt `lab-ha-3`.** ⭐ Trên **máy host**, mở lượt đo năm phút trong cửa sổ riêng:

```powershell
Start-Probe mat-mot-etcd 300
```

Khoảng nửa phút sau, chạy trên **`lab-ha-4`**:

```bash
ssh lab-ha-3 'sudo shutdown -h now' || true
```

Chạy trên **máy host**, cửa sổ chính — lặp lại tới khi đạt; **không** `vmrun stop … hard` trừ khi
[mục 4](#4-troubleshooting-của-lab-này) chỉ dẫn:

```powershell
$e3 = $haVmx | Where-Object { $_ -like '*lab-ha-3*' }
$running = & $vmrun -T ws list
if ($running -notcontains $e3) { "PASS: lab-ha-3 da Powered off" }
else { "CHUA TAT: doi them roi chay lai lenh nay" }
```

Đo, trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
H1=0
for i in $(seq 1 24); do
  ehealth > "$EV/b10-a-health.txt" 2>&1
  H1="$(grep -c 'is healthy' "$EV/b10-a-health.txt")"
  test "$H1" -eq 2 && break
  sleep 5
done
cat "$EV/b10-a-health.txt"
echo "endpoint khoe khi mat mot member: $H1/3"
kubectl get nodes > "$EV/b10-a-nodes.txt"

test "$H1" -eq 2 && echo 'PASS: dung hai member con khoe — quorum 2/3 van du'
cp_ok && app_ok && echo 'PASS: API va Bookstore VAN phuc vu khi mat mot etcd node'
test "$(grep -c ' Ready ' "$EV/b10-a-nodes.txt")" -eq 2 \
  && echo 'PASS: hai node van Ready — chung khong lien quan gi toi may vua tat'
```

Bật lại. Chạy trên **máy host**, cửa sổ chính:

```powershell
$e3 = $haVmx | Where-Object { $_ -like '*lab-ha-3*' }
& $vmrun -T ws start $e3
$running = & $vmrun -T ws list
if ($running -contains $e3) { "PASS: lab-ha-3 dang chay" } else { "FAIL: VM chua len" }
```

Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
H1b=0
for i in $(seq 1 60); do
  ehealth > "$EV/b10-a-health-hoi-phuc.txt" 2>&1
  H1b="$(grep -c 'is healthy' "$EV/b10-a-health-hoi-phuc.txt")"
  test "$H1b" -eq 3 && break
  sleep 5
done
echo "endpoint khoe sau khi bat lai: $H1b/3"
test "$H1b" -eq 3 && echo 'PASS: cum etcd tro lai du ba member khoe'
```

⭐ Khi cửa sổ đo in `KET QUA`, trên **máy host**: `Send-Probe mat-mot-etcd`. Rồi trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
tom_tat "$EV/do-mat-mot-etcd.csv"
ge "$APP" 99 && ge "$API" 90 && test "$CUOI_APP" -eq 1 && test "$CUOI_API" -eq 1 \
  && echo 'PASS: mat mot etcd member — nguoi dung khong nhan ra, API chi chop toi da vai mau'
```

**Kịch bản B — tắt `lab-ha-2` và `lab-ha-3`, mất quorum.** Trên **máy host**, mở lượt đo tám phút —
đủ cho cả lúc tắt, lúc chết và lúc hai máy boot lại:

```powershell
Start-Probe mat-quorum 480
```

Khoảng nửa phút sau, chạy trên **`lab-ha-4`**:

```bash
ssh lab-ha-2 'sudo shutdown -h now' || true
ssh lab-ha-3 'sudo shutdown -h now' || true
```

Chạy trên **máy host**, cửa sổ chính:

```powershell
$two = $haVmx | Where-Object { $_ -like '*lab-ha-2*' -or $_ -like '*lab-ha-3*' }
$running = & $vmrun -T ws list
$still = $two | Where-Object { $running -contains $_ }
if ($still.Count -eq 0) { "PASS: lab-ha-2 va lab-ha-3 deu Powered off" }
else { "CHUA TAT: $($still -join ', ')" }
```

Đo, trên **`lab-ha-4`**. Khối này tự giữ hai máy tắt thêm một phút trong lúc thử Bookstore 12 lần:

```bash
source ~/lab-work/8c/ham-do.sh
CPX='song'
for i in $(seq 1 24); do
  cp_ok || { CPX='chet'; break; }
  sleep 5
done
kubectl --request-timeout=15s get nodes > "$EV/b10-b-kubectl.txt" 2>&1
cat "$EV/b10-b-kubectl.txt"
test "$CPX" = 'chet' && echo 'PASS: mat quorum — API server ngung phuc vu'

ehealth > "$EV/b10-b-health.txt" 2>&1
cat "$EV/b10-b-health.txt"
H2="$(grep -c 'is healthy' "$EV/b10-b-health.txt")"
echo "endpoint khoe khi mat hai member: $H2/3"
test "$H2" -eq 0 \
  && echo 'PASS: khong endpoint nao healthy — ke ca member con song, vi mot khong lam nen da so'

OKAPP=0
for i in $(seq 1 12); do app_ok && OKAPP=$(( OKAPP + 1 )); sleep 5; done
echo "Bookstore tra loi dung $OKAPP/12 lan trong luc mat quorum" | tee "$EV/b10-b-app.txt"
test "$OKAPP" -eq 12 \
  && echo 'PASS: Bookstore VAN phuc vu trong luc mat quorum — mat phang du lieu khong can API'
```

Giữ hai máy tắt tới khi lượt đo đã chạy khoảng ba phút, rồi bật lại. Chạy trên **máy host**, cửa sổ
chính:

```powershell
$two = $haVmx | Where-Object { $_ -like '*lab-ha-2*' -or $_ -like '*lab-ha-3*' }
foreach ($f in $two) { & $vmrun -T ws start $f }
$running = & $vmrun -T ws list
$up = $two | Where-Object { $running -contains $_ }
if ($up.Count -eq 2) { "PASS: ca hai VM da chay lai" } else { "FAIL: con thieu VM" }
```

Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
H2b=0
for i in $(seq 1 72); do
  ehealth > "$EV/b10-b-health-hoi-phuc.txt" 2>&1
  H2b="$(grep -c 'is healthy' "$EV/b10-b-health-hoi-phuc.txt")"
  test "$H2b" -eq 3 && break
  sleep 5
done
echo "endpoint khoe sau khi bat lai: $H2b/3"
for i in $(seq 1 36); do cp_ok && break; sleep 5; done

test "$H2b" -eq 3 && echo 'PASS: cum etcd hoi phuc du ba member'
cp_ok && echo 'PASS: API phuc vu tro lai, khong mat du lieu'
( cd ~/bookstore && eval "$SMOKE_HA" ) > "$EV/b10-b-smoke-sau.txt" 2>&1 \
  && ! grep -q '^FAIL' "$EV/b10-b-smoke-sau.txt" \
  && echo 'PASS: Bookstore nguyen ven sau khi quorum tro lai'
```

Khi cửa sổ đo in `KET QUA`, trên **máy host**: `Send-Probe mat-quorum`. Rồi trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
tom_tat "$EV/do-mat-quorum.csv"
ge "$API_MAX" 60 && echo "PASS: API chet lien tuc ${API_MAX} giay — da do duoc mat quorum that"
ge "$APP" 99 && echo "PASS: ung dung van dat ${APP}% trong cung khoang do"
test "$CUOI_APP" -eq 1 && test "$CUOI_API" -eq 1 && echo 'PASS: ket thuc luot do ca hai duong deu 200'
```

Các vòng lặp có số vòng tối đa vì thời gian máy boot và member bắt kịp log phụ thuộc cấu hình và
phần cứng host.

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

1. Quorum không phải "còn ít nhất một bản sao" mà là "**quá nửa** số member đồng ý". Cụm có ba member,
   quá nửa là hai. Còn một member thì nó không xác nhận được một lần ghi nào, nên cũng không dám khẳng
   định dữ liệu nó giữ là mới nhất — báo không khỏe dù tiến trình vẫn chạy. Bật lại **một** trong hai
   máy là đủ: hai trên ba đã là đa số.
2. Người dùng: theo số đo, ứng dụng gần như không gián đoạn — Pod đã chạy vẫn chạy, `kube-proxy` trên
   mỗi node đã ghi quy tắc chuyển tiếp Service vào kernel, và đường `web` → `api` → `db-0` không cần
   hỏi API server ở bước nào. Người vận hành: mất toàn
   bộ khả năng **thay đổi** — không deploy, không scale, Pod chết không ai thay, node mới không join
   được. Lỗi của `kubectl` lúc đó đến **từ apiserver**: nó còn sống, còn nhận kết nối qua load
   balancer, chỉ không đọc ghi được dữ liệu. Mất control plane là mất quyền điều khiển, chưa phải mất
   dịch vụ — nhưng cluster ở trạng thái đó chỉ chờ một Pod chết là bắt đầu suy sụp.
3. Lab 8b: tắt **hai control plane node** (mất luôn hai apiserver). Lab 8c: tắt **hai etcd host**, hai
   apiserver vẫn sống. Với người dùng Bookstore, hai lần như nhau: ứng dụng vẫn chạy, API chết. Topology
   external **không** làm quorum bền hơn — ba member vẫn chịu được mất một. Thứ nó đổi là **cái gì kéo
   theo cái gì**: mất control plane không còn kéo theo mất member, và ngược lại.
4. Ba máy etcd không phải Node của cluster, nên cluster không có gì để báo mất. Cụm stacked không có
   tình huống này: không máy nào chỉ chạy etcd, tắt máy nào cũng là tắt luôn một apiserver — chính là
   hỏng theo cặp.

</details>

## B11. [BS-8C-11] Diễn tập: mất một control plane

**Loại:** Diễn tập sự cố — chỉ trên `lab-ha-5`.

**Bối cảnh:** đây là phép thử phân biệt hai topology rõ nhất, và là phép thử mà
[checkpoint giai đoạn 8](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) yêu cầu: tắt
một control plane node và chứng minh cluster vẫn phục vụ; xóa `kube-apiserver.yaml` rồi khôi phục. Hai
kịch bản cùng làm mất một `kube-apiserver`, nhưng một kịch bản mất cả máy, kịch bản kia chỉ mất một
static Pod — và trên cluster không có worker riêng, "cả máy" còn gồm cả một nửa Bookstore.

**Yêu cầu:**

1. **Kịch bản C — tắt `lab-ha-5`**: API vẫn phục vụ qua load balancer; cụm etcd vẫn ba member khỏe;
   node `lab-ha-5` chuyển `Ready=Unknown`; EndpointSlice của `web` không còn endpoint sẵn sàng nào trên
   `lab-ha-5`; sau đó Bookstore phục vụ trọn vẹn trở lại **dù `lab-ha-5` vẫn tắt**. Bật lại: node
   `Ready`, hai apiserver chạy, `SMOKE_HA` PASS. Đo từ máy host suốt kịch bản.
2. **Kịch bản D — mất static Pod apiserver trên `lab-ha-5`**: còn đúng một Pod `kube-apiserver`, nằm
   trên `lab-ha-4`; API và Bookstore vẫn phục vụ; node `lab-ha-5` vẫn `Ready`. Khôi phục: hai apiserver
   chạy, không còn bản sao file lạc. Đo từ máy host suốt kịch bản.
3. Postmortem đặt hai kịch bản cạnh nhau và cạnh kịch bản tắt control plane của Lab 8b.

**Ràng buộc:** chỉ chạm `lab-ha-5`; mất `lab-ha-4` là mất đường quan sát và mất `db-0`. Ở kịch bản D
dùng `mv`, không `rm`, và không `kubectl delete pod` — bài học là kubelet phản ứng với thư mục.

**Đọc lại:**

- Bài [08](../08-high-availability-vi.md#tạo-bộ-cân-bằng-tải-cho-kube-apiserver-create-load-balancer-for-kube-apiserver)
  — load balancer chỉ phân phối tới control plane **khỏe**, health check TCP.
- Bài [23](../23-nodes-vi.md#node-controller) — node controller đặt `Ready` thành `Unknown` khi mất
  liên lạc, và chờ bao lâu trước khi trục xuất.
- Bài [83](../83-endpoint-slices-vi.md#conditions) — condition `ready` của endpoint lấy từ `Ready` của
  Pod.
- Bài [139](../139-taint-and-toleration-vi.md#eviction-dựa-trên-taint-taint-based-evictions) — taint
  `node.kubernetes.io/unreachable` và `tolerationSeconds`.
- Bài [58](../58-static-pods-vi.md) — static Pod và mirror Pod.
- Lab 8b, [B9](LAB-8B-HA-VOI-STACKED-ETCD.md#b9-bs-8b-09-diễn-tập-mất-một-control-plane-node) và
  [B10](LAB-8B-HA-VOI-STACKED-ETCD.md#b10-bs-8b-10-diễn-tập-một-kube-apiserver-mất-manifest) — cùng hai
  kịch bản `mat-mot-cp`, `mat-manifest-apiserver` trên cụm stacked có worker riêng.

**Bàn giao:** `docs/postmortem/8c-mat-control-plane.md`; `~/lab-evidence/8c/b11-*`,
`do-mat-mot-cp.csv`, `do-mat-manifest-apiserver.csv` trên `lab-ha-4`.

**Nghiệm thu:**

- `PASS:` kịch bản C: API phục vụ, etcd 3/3, `lab-ha-5` `Unknown`, EndpointSlice bỏ endpoint trên
  `lab-ha-5`, Bookstore trả lời 12/12 lần thử trong lúc máy vẫn tắt.
- `PASS:` C bật lại: `lab-ha-5` `Ready`, hai apiserver `Running`, `SMOKE_HA` không có dòng `FAIL`.
- `PASS:` file đo C: ứng dụng **có** gián đoạn (dưới 100%), API ≥ 90%, mười mẫu cuối đều `200`.
- `PASS:` kịch bản D: một Pod apiserver, trên `lab-ha-4`; API, Bookstore và `/version` qua LB phục vụ;
  `lab-ha-5` vẫn `Ready`.
- `PASS:` D khôi phục: hai apiserver `Running`, không còn bản sao ở `/root`.
- `PASS:` file đo D: ứng dụng ≥ 99%, API ≥ 90%, mười mẫu cuối đều `200`.

**Câu hỏi thiết kế** — trả lời trong `docs/postmortem/8c-mat-control-plane.md`:

1. Vì sao ở C node chuyển `Unknown` còn ở D vẫn `Ready`? Ai quyết định ngừng gửi request **API** tới
   `lab-ha-5`, và ai quyết định ngừng gửi request **ứng dụng** tới Pod trên `lab-ha-5`?
2. Ở C, Bookstore gián đoạn bao lâu theo số đo của bạn, và vì sao không về 0 ngay lập tức? Kể chuỗi sự
   kiện từ lúc máy tắt tới lúc EndpointSlice bỏ endpoint. Độ dài khoảng đó phụ thuộc cấu hình nào?
3. Lab 8b tắt một control plane mà Bookstore không gián đoạn; ở đây có. Đó là do topology etcd hay do
   điều gì khác? Kết luận gì cho production của Bookstore?
4. Sau kịch bản C, cụm stacked của Lab 8b và cluster này mỗi bên còn chịu được mất thêm bao nhiêu etcd
   member? Đó là câu nào của bài 06?
5. Ở D, ai xóa Pod apiserver — bạn hay kubelet? Ai ghi được vào thư mục đó thì làm được gì?

<details>
<summary>Gợi ý</summary>

- Bắt đầu lượt đo **trước** khi tắt máy; chọn thời lượng phủ cả lúc máy boot lại.
- Endpoint của một EndpointSlice có `nodeName` và `conditions.ready`; jsonpath duyệt được
  `.items[*].endpoints[*]`.
- Static Pod manifest dời ra ngoài `/etc/kubernetes/manifests` thì kubelet dừng container và mirror Pod
  biến mất; dời vào lại thì ngược lại.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đọc xong; script này chính là cuộc diễn tập</summary>

**Kịch bản C.** Trên **máy host**, mở lượt đo tám phút trong cửa sổ riêng:

```powershell
Start-Probe mat-mot-cp 480
```

Khoảng nửa phút sau, chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
kubectl get nodes -o wide > "$EV/b11-c-nodes-truoc.txt"
kubectl -n bookstore get pods -o wide > "$EV/b11-c-pods-truoc.txt"
ssh lab-ha-5 'sudo shutdown -h now' || true
```

Chạy trên **máy host**, cửa sổ chính:

```powershell
$cp2 = $haVmx | Where-Object { $_ -like '*lab-ha-5*' }
$running = & $vmrun -T ws list
if ($running -notcontains $cp2) { "PASS: lab-ha-5 da Powered off" }
else { "CHUA TAT: doi them roi chay lai lenh nay" }
```

Đo, trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
cp_ok && echo 'PASS: API VAN phuc vu qua load balancer khi mat mot control plane node'
ehealth > "$EV/b11-c-health.txt" 2>&1
test "$(grep -c 'is healthy' "$EV/b11-c-health.txt")" -eq 3 \
  && echo 'PASS: cum etcd VAN du ba member — tat control plane khong dung toi etcd'

ST=''
for i in $(seq 1 60); do
  ST="$(kubectl --request-timeout=15s get node lab-ha-5 \
    -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null)"
  test "$ST" = 'Unknown' && break
  sleep 5
done
kubectl get nodes > "$EV/b11-c-nodes-down.txt"
echo "Ready condition cua lab-ha-5 = $ST"
test "$ST" = 'Unknown' && echo 'PASS: node controller doi Ready thanh Unknown khi mat heartbeat'

EP5=1
for i in $(seq 1 36); do
  EP5="$(kubectl -n bookstore get endpointslice -l kubernetes.io/service-name=web \
    -o jsonpath='{range .items[*].endpoints[*]}{.nodeName}={.conditions.ready}{"\n"}{end}' \
    | grep -c '^lab-ha-5=true')"
  test "$EP5" -eq 0 && break
  sleep 5
done
kubectl -n bookstore get endpointslice -o wide > "$EV/b11-c-eps.txt"
kubectl -n bookstore get pods -o wide > "$EV/b11-c-pods-down.txt"
test "$EP5" -eq 0 && echo 'PASS: EndpointSlice cua web khong con endpoint ready nao tren lab-ha-5'

OKAPP=0
for i in $(seq 1 12); do app_ok && OKAPP=$(( OKAPP + 1 )); sleep 5; done
echo "Bookstore tra loi dung $OKAPP/12 lan khi lab-ha-5 van tat" | tee "$EV/b11-c-app.txt"
test "$OKAPP" -eq 12 && echo 'PASS: Bookstore phuc vu tron ven tro lai du lab-ha-5 van tat'
```

Bật lại. Chạy trên **máy host**, cửa sổ chính:

```powershell
$cp2 = $haVmx | Where-Object { $_ -like '*lab-ha-5*' }
& $vmrun -T ws start $cp2
$running = & $vmrun -T ws list
if ($running -contains $cp2) { "PASS: lab-ha-5 dang chay" } else { "FAIL: VM chua len" }
```

Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
kubectl wait --for=condition=Ready node/lab-ha-5 --timeout=600s
kubectl get nodes -o wide > "$EV/b11-c-nodes-up.txt"
API_UP2=0
for i in $(seq 1 60); do
  API_UP2="$(kubectl -n kube-system get pod -l component=kube-apiserver \
    --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)"
  test "$API_UP2" -eq 2 && break
  sleep 5
done
test "$API_UP2" -eq 2 && echo 'PASS: ca hai kube-apiserver da chay lai'
( cd ~/bookstore && eval "$SMOKE_HA" ) > "$EV/b11-c-smoke-sau.txt" 2>&1 \
  && ! grep -q '^FAIL' "$EV/b11-c-smoke-sau.txt" \
  && echo 'PASS: Bookstore nguyen ven sau khi lab-ha-5 tro lai'
```

Khi cửa sổ đo in `KET QUA`, trên **máy host**: `Send-Probe mat-mot-cp`. Rồi trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
tom_tat "$EV/do-mat-mot-cp.csv"
lt "$APP" 100 && echo "PASS: ung dung CO gian doan — ${APP}%, dai nhat ${APP_MAX} giay"
ge "$API" 90 && echo "PASS: API van dat ${API}% — load balancer bo backend da mat"
test "$CUOI_APP" -eq 1 && test "$CUOI_API" -eq 1 && echo 'PASS: ket thuc luot do ca hai duong deu 200'
```

**Kịch bản D.** Trên **máy host**, mở lượt đo năm phút trong cửa sổ riêng:

```powershell
Start-Probe mat-manifest-apiserver 300
```

Khoảng nửa phút sau, chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
ssh lab-ha-5 'sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml'
ssh lab-ha-5 'sudo ls -1 /etc/kubernetes/manifests/' > "$EV/b11-d-manifests-thieu.txt"
grep -q 'kube-apiserver.yaml' "$EV/b11-d-manifests-thieu.txt" \
  && echo 'FAIL: file van con trong thu muc' \
  || echo 'PASS: kube-apiserver.yaml da roi khoi /etc/kubernetes/manifests'
ssh lab-ha-5 'sudo test -f /root/kube-apiserver.yaml' && echo 'PASS: da giu ban sao o /root, khoi phuc duoc'

APIN=2
for i in $(seq 1 36); do
  APIN="$(kubectl --request-timeout=15s -n kube-system get pod -l component=kube-apiserver \
    --no-headers 2>/dev/null | wc -l)"
  test "$APIN" -le 1 && break
  sleep 5
done
kubectl -n kube-system get pod -l component=kube-apiserver -o wide > "$EV/b11-d-api-pods.txt"
ONNODE="$(kubectl -n kube-system get pod -l component=kube-apiserver -o jsonpath='{.items[0].spec.nodeName}')"
test "$APIN" -eq 1 && test "$ONNODE" = 'lab-ha-4' \
  && echo 'PASS: chi con mot kube-apiserver, cua lab-ha-4 — kubelet tren lab-ha-5 da xoa Pod theo thu muc'

cp_ok && app_ok && echo 'PASS: API va Bookstore VAN phuc vu voi mot apiserver'
curl -sk "https://${LB}:6443/version" | grep -q 'gitVersion' \
  && echo 'PASS: goi thang qua load balancer van co ket qua'
READY5="$(kubectl get node lab-ha-5 -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
test "$READY5" = 'True' && echo 'PASS: lab-ha-5 VAN Ready — kubelet cua no van song, chi thieu apiserver'
```

Khoảng hai phút sau, khôi phục. Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
ssh lab-ha-5 'sudo mv /root/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml'
APIN2=0
for i in $(seq 1 60); do
  APIN2="$(kubectl -n kube-system get pod -l component=kube-apiserver \
    --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)"
  test "$APIN2" -eq 2 && break
  sleep 5
done
kubectl -n kube-system get pod -l component=kube-apiserver -o wide > "$EV/b11-d-api-pods-hoi-phuc.txt"
ssh lab-ha-5 'sudo test -f /root/kube-apiserver.yaml' \
  && echo 'FAIL: ban sao van con o /root' \
  || echo 'PASS: da tra file ve dung cho, khong con ban sao lac'
test "$APIN2" -eq 2 && cp_ok && app_ok && echo 'PASS: ca hai kube-apiserver da chay lai, cluster binh thuong'
```

Khi cửa sổ đo in `KET QUA`, trên **máy host**: `Send-Probe mat-manifest-apiserver`. Rồi trên
**`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
tom_tat "$EV/do-mat-manifest-apiserver.csv"
ge "$APP" 99 && ge "$API" 90 && test "$CUOI_APP" -eq 1 && test "$CUOI_API" -eq 1 \
  && echo 'PASS: mat mot apiserver — ung dung khong biet, API chi chop trong luc LB doi backend'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

1. Node status do **kubelet** báo. Ở C kubelet chết theo máy, node controller mất nhịp tim nên đặt
   `Unknown`; ở D kubelet vẫn sống nên node vẫn `Ready`. Hai sự cố khác nhau cho hai triệu chứng khác
   nhau — nhớ cặp này để chẩn đoán nhanh. Request **API**: health check TCP của HAProxy thấy `:6443` trên
   `lab-ha-5` không còn trả lời và ngừng gửi; kubeconfig không đổi gì, địa chỉ vẫn là
   `ControlPlaneEndpoint`. Request **ứng dụng**: khi node `Unknown`, Pod trên đó mất `Ready`, condition
   `ready` của endpoint đi theo (bài 83), `kube-proxy` trên `lab-ha-4` ngừng chọn chúng.
2. Gián đoạn kéo dài từ lúc máy tắt tới lúc EndpointSlice bỏ endpoint: trong khoảng đó `kube-proxy` trên
   `lab-ha-4` vẫn chọn ngẫu nhiên cả Pod trên máy đã chết, nên một phần request rơi vào khoảng không.
   Chuỗi: kubelet ngừng gửi nhịp tim → node controller kiểm định kỳ và đặt `Unknown` sau một khoảng ân
   hạn → Pod mất `Ready` → EndpointSlice cập nhật → `kube-proxy` cập nhật quy tắc. Độ dài phụ thuộc cấu
   hình của `kube-controller-manager` (chu kỳ kiểm và khoảng ân hạn), không phải hằng số. Pod **không**
   bị thay ngay: bài 23 nói node controller chờ trước khi trục xuất; bài 139 gọi đó là
   `tolerationSeconds` của taint `unreachable`.
3. Không phải do topology etcd. Lab 8b có **worker riêng**: tắt control plane không đụng tới Pod nào.
   Ở đây control plane **cũng là** worker, nên mất nó là mất một nửa sức chứa của Bookstore. Kết luận
   cho production: workload chạy trên worker riêng, giữ taint control plane — bất kể chọn topology nào.
4. Stacked sau khi mất một control plane: còn 2/3 member, chịu thêm **0** — mất máy nữa là mất quorum.
   External sau khi mất `lab-ha-5`: etcd vẫn 3/3, chịu thêm **1**. Bài 06: mất một instance control plane
   "không ảnh hưởng đến tính dự phòng của cluster nhiều như topology HA xếp chồng".
5. Kubelet — nó đọc thư mục static Pod, dừng container và mirror Pod biến mất; bạn không gọi API nào.
   Ai ghi được vào `/etc/kubernetes/manifests` thì chạy được container trên node mà không đi qua API
   server — lý do quyền trên thư mục đó là chuyện bảo mật, sẽ quay lại ở giai đoạn 9.

</details>

## B12. [BS-8C-12] ADR: topology HA cho production của Bookstore

**Loại:** Điều tra.

**Bối cảnh:** yêu cầu
[R12](DU-AN-BOOKSTORE.md#22-yêu-cầu-của-hệ-thống-mới) của dự án đòi "đánh giá được topology HA". Lab 8b
đã mở `docs/adr/NNNN-topology-ha.md`, chốt phần stacked — dự đoán, bảng `## Số đo Lab 8b`, rủi ro
SPOF — và để trống `## External (Lab 8c)`. Hai sprint đã dựng cả hai topology trên cùng phần cứng và
đo cùng ứng dụng bằng cùng script. Ticket này đặt hai bộ số cạnh nhau và biến chúng thành một quyết
định có điều kiện.

**Yêu cầu:**

1. Hoàn thiện **đúng file ADR của Lab 8b**, không tạo ADR mới. Dòng *Sprint* thêm Lab 8c; trạng thái
   **chấp nhận** cho cả ADR.
2. `## External (Lab 8c)`: thay dòng chờ điền bằng hình dạng cluster external đo ở B1 và B8 — số
   apiserver, số etcd member, số máy, giá trị `--etcd-servers` — và câu trả lời câu hỏi thiết kế của
   B1, B8.
3. `## Số đo Lab 8c`: bảng **cùng cột** với `## Số đo Lab 8b` (kịch bản, ứng dụng, API, gián đoạn dài
   nhất ứng dụng / API), một dòng cho mỗi kịch bản `mat-mot-etcd`, `mat-mot-cp`,
   `mat-manifest-apiserver`, `mat-quorum`, lấy từ `~/lab-evidence/8c/so-do.txt`. Kịch bản nào không có
   ở topology kia thì ghi rõ trong lời bình dưới bảng.
4. `## Kết luận`: số host tối thiểu cho **Bookstore production** theo từng topology, tách từng nhóm
   máy; độ phức tạp vận hành của external liệt kê bằng những việc cụ thể sprint 14 làm thêm so với
   sprint 13 (lấy từ runbook); topology được chọn, kèm **điều kiện đổi quyết định**; những gì
   production phải có **bất kể** topology; nợ còn mở (#7, #8).
5. Runbook và hai postmortem của sprint đầy đủ các mục mà ticket trước yêu cầu.

**Ràng buộc:** số trong bảng là số **bạn đo**, không chép từ lời giải. Không sửa bảng
`## Dự đoán và đo` hay `## Số đo Lab 8b` của Lab 8b — đó là lịch sử. Không đưa vào ADR thứ chưa học:
đề xuất cần kiến thức giai đoạn sau (cặp load balancer với VIP) thì ghi thành điều kiện của
production, không thiết kế chi tiết.

**Đọc lại:** bài [06](../06-ha-topology-vi.md) — cả hai mục topology, đặc biệt các câu về số host tối
thiểu; bài [08](../08-high-availability-vi.md#external-etcd), mục *Trước khi bạn bắt đầu* — "số lẻ"
**là yêu cầu** với etcd nhưng chỉ **có thể giúp ích** với control plane; bài
[02](../02-create-cluster-kubeadm-vi.md#resilience) — khả năng chống chịu của cluster một control plane;
[mục 3.7 của file dự án](DU-AN-BOOKSTORE.md#37-chịu-lỗi-và-vận-hành).

**Bàn giao:** `docs/adr/NNNN-topology-ha.md`; `docs/runbook-ha-external-etcd.md`; hai postmortem.

**Nghiệm thu:**

- `PASS:` ADR topology của Lab 8b đã hoàn thiện: `## External (Lab 8c)` không còn chờ điền, có
  `## Số đo Lab 8c` và `## Kết luận`, trạng thái **chấp nhận**.
- `PASS:` bảng `## Số đo Lab 8c` có đủ bốn kịch bản, mỗi dòng hai ô phần trăm.
- `PASS:` `## Kết luận` nói tới quorum, worker riêng, load balancer, và nợ #8.
- `PASS:` bảng `## Số đo Lab 8b` không đổi so với tag `lab-8b`.
- `PASS:` runbook có các mục của B2–B9 và chứa cấu hình `etcd.external` đã dùng; có hai postmortem.

**Câu hỏi thiết kế** — trả lời trong `## Kết luận` của ADR:

1. Quorum nằm ở tầng nào? Vì sao **2 control plane + 3 etcd member** là cấu hình hợp lệ, còn 2 control
   plane stacked thì không phải HA? Dẫn đúng hai câu của bài 08 về "số lẻ".
2. Số host tối thiểu của Bookstore production theo từng topology là bao nhiêu? Tính rõ: control plane,
   etcd, worker, load balancer.
3. Đặt `mat-quorum` và `mat-mot-cp` của hai bảng cạnh nhau. Topology etcd **thay đổi** gì và **không
   thay đổi** gì với người dùng Bookstore? Với người vận hành?
4. Những việc nào sprint 14 phải làm thêm so với sprint 13? Việc nào sẽ lặp lại ở mỗi lần nâng cấp,
   xoay certificate hay thay máy?
5. Điều kiện nào thì dự án đổi sang external? Thành phần nào vẫn là điểm hỏng đơn trong **cả hai**
   topology như lab đã dựng?

<details>
<summary>Gợi ý</summary>

- Số đo của Lab 8b nằm sẵn trong bảng `## Số đo Lab 8b` của ADR và trong `~/lab-evidence/8b/so-do.txt`
  của sprint 13. Thiếu số nào thì ghi "không đo" — đừng bịa.
- Hai kịch bản không đối xứng: `mat-mot-etcd` không tồn tại riêng ở stacked; `mat-lb` không đo lại ở 8c
  vì load balancer và cấu hình của nó giống hệt.
- Số host tối thiểu của bài 06 chưa tính worker và load balancer.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

Chạy trên **`lab-ha-4`**:

```bash
cd ~/bookstore
ADR="$(ls docs/adr/*-topology-ha.md 2>/dev/null | head -n 1)"
test -n "$ADR" && echo "PASS: co ADR topology cua Lab 8b: $ADR"
grep -q '^## External (Lab 8c)' "$ADR" && ! grep -q '(Lab 8c điền)' "$ADR" \
  && grep -q '^## Số đo Lab 8c' "$ADR" && grep -q '^## Kết luận' "$ADR" \
  && grep -q 'Trạng thái: chấp nhận' "$ADR" \
  && echo 'PASS: ADR da dien phan external, co so do Lab 8c, ket luan, trang thai chap nhan'

SO8C="$(sed -n '/^## Số đo Lab 8c/,/^## /p' "$ADR")"
BAD=0
for S in mat-mot-etcd mat-mot-cp mat-manifest-apiserver mat-quorum; do
  echo "$SO8C" | grep -Eq "^\| *$S *\|.*%.*%" || { echo "FAIL: bang So do Lab 8c thieu $S"; BAD=1; }
done
test "$BAD" -eq 0 && echo 'PASS: bang So do Lab 8c du bon kich ban'

KL="$(sed -n '/^## Kết luận/,/^## /p' "$ADR")"
echo "$KL" | grep -qi 'quorum' && echo "$KL" | grep -qi 'worker' \
  && echo "$KL" | grep -qiE 'load balancer|LB' && echo "$KL" | grep -q '#8' \
  && echo 'PASS: ket luan neu quorum, worker rieng, load balancer va no #8'
diff <(git show "lab-8b:$ADR" | sed -n '/^## Số đo Lab 8b/,/^## /p') \
     <(sed -n '/^## Số đo Lab 8b/,/^## /p' "$ADR") >/dev/null \
  && echo 'PASS: bang So do Lab 8b giu nguyen nhu tag lab-8b'

RB=docs/runbook-ha-external-etcd.md
BAD=0
for k in 'Gỡ cluster cũ' 'Load balancer' 'Host etcd' 'Certificate etcd' 'Cụm etcd' \
         'Control plane external' 'Triển khai Bookstore'; do
  grep -q "$k" "$RB" || { echo "FAIL: runbook thieu muc $k"; BAD=1; }
done
grep -q 'external:' "$RB" && test "$(grep -c 'https://192.168.100.23[123]:2379' "$RB")" -ge 3 \
  || { echo 'FAIL: runbook chua chua cau hinh etcd.external da dung'; BAD=1; }
test "$BAD" -eq 0 && echo 'PASS: runbook du cac muc va cau hinh'
test -s docs/postmortem/8c-mat-quorum-etcd.md && test -s docs/postmortem/8c-mat-control-plane.md \
  && echo 'PASS: co hai postmortem cua sprint 14'
```

</details>

<details>
<summary>Lời giải tham khảo — phần Lab 8c của ADR và ý chính</summary>

Ba mục mà Lab 8c thêm hoặc điền — số trong ngoặc nhọn là số **của bạn**:

```markdown
## External (Lab 8c)

Cùng sáu VM, đổi vai: 3 etcd host (lab-ha-1..3), 2 control plane (lab-ha-4, lab-ha-5), cùng LB.
External = 2 apiserver, 3 etcd member, 5 máy; stacked = 3 / 3 / 3. `--etcd-servers` của cả hai
apiserver liệt kê cả ba etcd host (nhiều-nhiều); ở stacked mỗi apiserver chỉ `127.0.0.1:2379`
(một-một). 0 Pod `component=etcd`, không `etcd.yaml` trên control plane. Không worker riêng: Bookstore
chạy trên control plane sau khi gỡ taint, `db-0` ghim lab-ha-4 (overlay, patch của Lab 8c).

## Số đo Lab 8c

Đo từ máy host bằng scripts/do-san-sang.ps1 (CSV ms,app,api), tỷ lệ theo thời gian, cả cửa sổ.

| Kịch bản | Ứng dụng | API | Gián đoạn dài nhất ứng dụng / API |
| --- | --- | --- | --- |
| mat-mot-etcd | {…}% | {…}% | {…}s / {…}s |
| mat-mot-cp | {…}% | {…}% | {…}s / {…}s |
| mat-manifest-apiserver | {…}% | {…}% | {…}s / {…}s |
| mat-quorum | {…}% | {…}% | {…}s / {…}s |

mat-mot-etcd không tồn tại riêng ở stacked. mat-mot-cp của 8c tắt một máy vừa là control plane vừa
là worker — khác 8b, nơi workload ở worker riêng. mat-lb không đo lại: LB và cấu hình giống 8b.

## Kết luận

| Production | Stacked | External |
| --- | --- | --- |
| Control plane | 3 | 3 |
| etcd riêng | 0 | 3 |
| Worker | ≥ 2 | ≥ 2 |
| Load balancer | 2 + VIP | 2 + VIP |
| Tổng tối thiểu | 7 | 10 |

Quyết định: stacked, 3 control plane, worker riêng, giữ taint control plane. Đổi sang external khi …
Production phải có bất kể topology: … Nợ còn mở: #7, #8.
```

Ý chính:

1. Quorum nằm ở **tầng etcd**. Bài 08: với etcd, số lẻ thành viên "**là một yêu cầu**" để đạt quorum
   bỏ phiếu tối ưu; với control plane, số lẻ chỉ "**có thể giúp ích**" cho việc bầu leader. External
   giữ nguyên vẹn thứ là yêu cầu — ba member — nên hai control plane vẫn hợp lệ: thêm bớt một apiserver
   không thêm bớt bản sao dữ liệu nào. Stacked hai control plane là hai member: mất một là mất quorum,
   nên bài 06 đòi tối thiểu ba.
2. Stacked: 3 control plane (kiêm etcd) + 2 worker + 2 load balancer = **7**. External: 3 control plane
   + 3 etcd + 2 worker + 2 load balancer = **10** — đúng "gấp đôi số host" ở tầng control plane mà bài
   06 nói. Cặp load balancer kèm VIP chưa học trong giai đoạn 8; ghi là điều kiện của production.
3. `mat-quorum`: hai topology như nhau với người dùng — API chết, ứng dụng chạy tiếp. Khác ở chỗ phải
   mất **cái gì**: stacked mất quorum khi mất hai control plane; external chỉ khi mất hai etcd host.
   `mat-mot-cp`: gián đoạn ở 8c đến từ việc control plane **cũng là** worker, không phải từ etcd. Với
   người vận hành: external rộng biên hơn sau một sự cố (vẫn chịu được mất thêm một member) và tách được
   hai loại hỏng — đổi lại gấp đôi máy và nhiều việc hơn.
4. Việc thêm: kubelet làm trình quản lý dịch vụ trên ba host etcd, một CA và tám certificate riêng của
   etcd phải sinh, phân phối, giữ `ca.key` đúng một chỗ, chép certificate client sang control plane,
   file cấu hình `etcd.external` bắt buộc, kiểm etcd bằng `etcdctl` ngoài Kubernetes. Lặp lại ở mỗi lần
   xoay certificate (nợ #7) và khi thay một host etcd; backup etcd (nợ #8) phải làm trên host riêng.
5. Đổi sang external khi tải etcd hoặc yêu cầu cô lập sự cố giữa etcd và control plane lớn hơn chi phí
   gấp đôi máy và vận hành — không phải trường hợp của một hiệu sách nhỏ. Điểm hỏng đơn trong cả hai:
   **một** load balancer (Lab 8b đã đo `mat-lb`). Thêm vào đó, `db` một bản sao ghim một node vẫn là
   điểm hỏng đơn của chính ứng dụng — topology control plane không chữa được nó.

</details>

## B13. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao repo `lab-8c` ra máy host, gỡ Bookstore, trả taint control plane về mặc định,
chụp `8x-ha-external` trên đủ sáu VM, chứng minh bộ VM chính không bị chạm — rồi chuẩn bị repo cho
Lab 9a.

Sprint này **không đổi script kiểm thử nào**: nó không thêm hợp đồng vào hệ thống, chỉ thêm tài liệu
và một patch của overlay HA. `tests/smoke.sh` ở tag `lab-8c` vẫn là bộ kiểm thử tích lũy của chuỗi
chính mà Lab 9a chạy ở B0 của nó; trên bộ VM HA, kiểm thử khói tích lũy là `tests/smoke-ha-lab.sh` của
Lab 8b — nó không khẳng định Pod nằm ở node nào, nên chạy nguyên trên cluster external. Kiểm thêm của
riêng sprint 14 — hình dạng external, `db-0` ở `lab-ha-4` — nằm ở bước 1 và bước 5.

1. Kiểm thử khói lần cuối và hình dạng external. Chạy trên **`lab-ha-4`**:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
source ~/lab-work/8c/ham-do.sh
cd ~/bookstore
eval "$TEARDOWN_HA" > "$EV/b13-teardown-truoc.txt" 2>&1 \
  && eval "$DEPLOY_HA" > "$EV/b13-deploy.txt" 2>&1 \
  && echo 'PASS: dung lai sach tu repo bang teardown-ha-lab va deploy-ha-lab'
# smoke-ha-lab thoát khác 0 khi có dòng FAIL hoặc không chạy được — dựa vào mã thoát, không chỉ vào
# việc không thấy chữ FAIL.
if eval "$SMOKE_HA" > "$EV/b13-smoke.txt" 2>&1 && ! grep -q '^FAIL' "$EV/b13-smoke.txt"; then
  echo 'PASS: smoke-ha-lab thoat 0, khong co dong FAIL'
else echo 'FAIL: smoke-ha-lab con loi — xem b13-smoke.txt'; fi
grep '^INFO' "$EV/b13-smoke.txt" || true
test "$(kubectl -n bookstore get pod db-0 -o jsonpath='{.spec.nodeName}')" = 'lab-ha-4' \
  && echo 'PASS: db-0 van o lab-ha-4 sau khi dung lai tu repo'
test "$(kubectl get nodes --no-headers | wc -l)" -eq 2 \
  && test "$(sl kubectl -n kube-system get pod -l component=etcd --no-headers)" -eq 0 \
  && echo 'PASS: van dung hinh dang external — 2 node, 0 Pod etcd'
ehealth > "$EV/b13-health.txt" 2>&1
test "$(grep -c 'is healthy' "$EV/b13-health.txt")" -eq 3 \
  && echo 'PASS: cum etcd du ba member khoe sau bon kich ban'
```

2. `docs/tech-debt.md`: sprint không phát sinh, không trả nợ kỹ thuật nào. Bổ sung vào dòng ghi chú về
   overlay `ha-lab` mà Lab 8b đã viết: *Lab 8c thêm patch ghim `db-0` vào `lab-ha-4` cho cluster không
   có worker — vẫn là môi trường đo, không phải dữ liệu thật*. Nợ lab #7, #8 vẫn mở; chúng nằm ở
   [sổ nợ lab](README.md#5-sổ-nợ-lab), không ở file này.

3. Commit, tag, đóng gói, chép ra máy host theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm). Chạy trên **`lab-ha-4`**:

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-8c: sprint 14 — HA external etcd, ADR topology production'
git tag lab-8c
git bundle create ~/bookstore-lab-8c.bundle --all && git bundle verify ~/bookstore-lab-8c.bundle
test -z "$(git diff --name-only lab-8b lab-8c | grep -vE '^(docs/|k8s/overlays/ha-lab/)')" \
  && echo 'PASS: sprint chi doi docs/ va k8s/overlays/ha-lab/'
```

Chạy trên **máy host**, cửa sổ chính:

```powershell
scp "$($haUser)@192.168.100.234:bookstore-lab-8c.bundle" "$backup\"
$b8c = Join-Path $backup 'bookstore-lab-8c.bundle'
if ((Test-Path $b8c) -and ((Get-Item $b8c).Length -gt 0)) { "PASS: co $b8c" } else { "FAIL: thieu $b8c" }
```

4. Gỡ Bookstore. Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
cd ~/bookstore && eval "$TEARDOWN_HA" | tee "$EV/b13-teardown.txt"
grep -q 'teardown: sach' "$EV/b13-teardown.txt" && echo 'PASS: teardown-ha-lab tu kiem: sach'
SEL='app.kubernetes.io/part-of=bookstore'
LEFT="$( { kubectl get namespace,priorityclass,storageclass -l "$SEL" --no-headers
           kubectl get all,configmaps,secrets,pvc -A -l "$SEL" --no-headers; } 2>/dev/null | wc -l)"
test "$LEFT" -eq 0 && echo 'PASS: khong con object nao cua du an'
test "$(kubectl get nodes --show-labels --no-headers | grep -c 'bookstore\.example\.com/')" -eq 0 \
  && echo 'PASS: khong node nao con label cua du an'
```

5. Trả taint control plane về mặc định kubeadm và gate sức khỏe hai tầng. Chạy trên **`lab-ha-4`**:

```bash
source ~/lab-work/8c/ham-do.sh
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule
kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints' | tee "$EV/b13-taint-cuoi.txt"
TAINT_N="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"="}{.effect}{"\n"}{end}{end}' \
  | grep -c '^node-role.kubernetes.io/control-plane=NoSchedule$')"
test "$TAINT_N" -eq 2 && echo 'PASS: ca hai node da tro lai trang thai mac dinh cua kubeadm'

kubectl wait --for=condition=Ready node --all --timeout=300s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

{
  echo "=== $(date -Is) — trang thai cuoi Lab 8c ==="
  echo '--- nodes'; kubectl get nodes -o wide
  echo '--- taints'; cat "$EV/b13-taint-cuoi.txt"
  echo '--- control plane pods'; kubectl -n kube-system get pods -l tier=control-plane -o wide
  echo '--- so Pod component=etcd'; kubectl -n kube-system get pod -l component=etcd --no-headers | wc -l
  echo '--- namespaces'; kubectl get namespaces
  echo '--- etcd health'; ehealth
} > "$EV/b13-cuoi.txt" 2>&1
test -s "$EV/b13-cuoi.txt" && echo 'PASS: da ghi bang chung cuoi'
```

6. Dọn file tạm. Certificate của etcd **ở nguyên** `/etc/kubernetes/pki` — cụm etcd chạy bằng chúng;
   chỉ các bản sao trong `/tmp`, `$HOME` và thư mục làm việc bị xóa. Chạy trên **`lab-ha-4`**:

```bash
ssh lab-ha-1 'rm -rf /tmp/192.168.100.231 /tmp/192.168.100.232 /tmp/192.168.100.233 ~/lab-work/8c'
for n in lab-ha-2 lab-ha-3; do ssh "$n" 'rm -f $HOME/kubeadmcfg.yaml; rm -rf ~/lab-work/8c'; done
rm -rf ~/lab-work/8c

test "$(ssh lab-ha-1 'ls -d /tmp/192.168.100.23* 2>/dev/null | wc -l')" -eq 0 \
  && echo 'PASS: da xoa thu muc tam chua certificate tren lab-ha-1'
test ! -e ~/lab-work/8c && test -z "$(git -C ~/bookstore status --porcelain)" \
  && git -C ~/bookstore describe --tags --exact-match HEAD | grep -qx 'lab-8c' \
  && echo 'PASS: da don lab-work; repo sach, HEAD mang tag lab-8c'
```

Thư mục `~/lab-evidence/8c/` trên các máy **giữ lại** — snapshot sắp chụp mang theo nó. Repo
`~/bookstore` trên `lab-ha-4` cũng ở lại trong snapshot, nhưng bản dùng tiếp là bundle trên máy host.

7. Tắt sáu VM và chụp `8x-ha-external`. Tắt theo thứ tự ngược với thứ tự bật. Chạy trên **`lab-ha-4`**:

```bash
ssh lab-ha-5 'sudo shutdown -h now' || true
for n in lab-ha-3 lab-ha-2 lab-ha-1; do ssh "$n" 'sudo shutdown -h now' || true; done
sudo shutdown -h now
```

Chạy **trên `lab-ha-lb`** từ phiên SSH riêng của nó: `sudo shutdown -h now`. Rồi trên **máy host**,
cửa sổ chính, chờ đủ sáu máy về *Powered off*:

```powershell
$running = & $vmrun -T ws list
$still = $haVmx | Where-Object { $running -contains $_ }
if ($still.Count -eq 0) { "PASS: ca sau VM HA da Powered off" }
else { "CHUA TAT: $($still -join ', ')" }
```

Chụp trên **cả sáu VM**: chuột phải VM → **Snapshot → Take Snapshot**, ô *Name* điền đúng chuỗi
`8x-ha-external` — không hậu tố theo VM, không thêm ngày, không đổi hoa thường. Ô *Description* ghi
`dung bang LAB-8C-HA-VOI-EXTERNAL-ETCD.md (Sprint 14), chup <ngay>`. Rồi kiểm:

```powershell
foreach ($f in $haVmx) {
  $snaps = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($snaps -ccontains '8x-ha-external') { "PASS: $f" } else { "FAIL: $f -> $($snaps -join ', ')" }
}
```

Mốc `8x-ha-external` là: 2 control plane + 3 etcd node ngoài + 1 load balancer, Flannel, taint control
plane ở mặc định kubeadm, không workload, không object nào của Bookstore. Lab nào dùng lại mốc này mà
cần chạy workload thì tự gỡ taint theo B9 và tự chịu trách nhiệm với ghi chú bảo mật ở đó.

8. Gate cuối: bộ VM chính vẫn nguyên vẹn. Chạy trên **máy host**, cửa sổ chính:

```powershell
$running = & $vmrun -T ws list
foreach ($f in $mainVmx) {
  $isOff = ($running -notcontains $f)
  $snaps = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  $has03 = ($snaps -ccontains '03-storage-ready')
  $hasHA = @($snaps | Where-Object { $_ -like '8x-*' }).Count
  if ($isOff -and $has03 -and $hasHA -eq 0) { "PASS: $f khong bi cham" }
  else { "FAIL: $f -> off=$isOff co03=$has03 so-snapshot-8x=$hasHA" }
}
```

**PASS:** toàn bộ dòng `PASS:` của bước 1–6; sáu dòng `PASS:` của snapshot; đúng ba dòng
`PASS: … khong bi cham`. Lệnh field selector trả `No resources found`, CoreDNS đủ replica, namespace
`default` không có Pod. Ba điều kiện của gate cuối nói ba chuyện: không ai bật nhầm bộ VM chính, mốc
`03-storage-ready` còn, và không ai chụp nhầm mốc `8x-` lên nó.

9. **Bàn giao cho Lab 9a** — chạy **khi mở Lab 9a**, sau gate cuối ở trên, khi bộ HA đã tắt và bộ VM
   chính đã bật theo [A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab) ở mốc
   `03-storage-ready`. Repo trên `lab-k8s-master` đang ở tag cũ hơn — snapshot mang nó về quá khứ;
   thay bằng bản `lab-8c` từ bundle.

Chạy trên **máy host** (`$mainUser` là user trên `lab-k8s-master`, thường khác `$haUser`):

```powershell
$backup   = "$HOME\lab-backup"
$mainUser = '<user-tren-lab-k8s-master>'
scp "$backup\bookstore-lab-8c.bundle" "$($mainUser)@192.168.100.221:"
```

Chạy trên **`lab-k8s-master`**:

```bash
git bundle verify ~/bookstore-lab-8c.bundle >/dev/null 2>&1 && echo 'PASS: bundle lab-8c toan ven'
if [ -d ~/bookstore/.git ]; then
  if [ -n "$(git -C ~/bookstore status --porcelain)" ]; then
    echo 'FAIL: ~/bookstore cu con thay doi chua commit — xu ly truoc khi thay'
  else
    mv ~/bookstore ~/bookstore-truoc-8c && echo 'PASS: da cat repo cu sang ~/bookstore-truoc-8c'
  fi
fi
test -e ~/bookstore || git clone -q ~/bookstore-lab-8c.bundle ~/bookstore
test -d ~/bookstore-truoc-8c/secrets && cp -a ~/bookstore-truoc-8c/secrets/. ~/bookstore/secrets/
git -C ~/bookstore describe --tags --exact-match HEAD | grep -qx 'lab-8c' \
  && echo 'PASS: repo tren lab-k8s-master o tag lab-8c — san sang cho Lab 9a'
```

**PASS:** `PASS: bundle lab-8c toan ven` và `PASS: repo tren lab-k8s-master o tag lab-8c …`. Chép lại
`secrets/` cũ giữ nguyên certificate TLS và mật khẩu mà máy host có thể đã tin; không có nó thì
`deploy.sh` của chuỗi chính tự sinh bí mật mới, cũng hợp lệ vì teardown đã xóa dữ liệu. Xóa
`~/bookstore-truoc-8c` sau khi B0 của Lab 9a PASS.

---

## 3. Checkpoint 8c

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Sáu máy, muốn dựng HA cho Bookstore. Kể hai cách chia vai — stacked và external — với mỗi cách:
      mấy máy chạy `kube-apiserver`, mấy máy giữ etcd member, mấy máy làm worker. Vì sao lab vẫn dựng
      được external với sáu máy, và cái giá là gì?
- [ ] `--etcd-servers` của apiserver trên `lab-ha-4` trỏ tới đâu? Ở Lab 8b, ba giá trị đó là gì? Chỉ
      nhìn cấu hình đó, phân biệt hai topology thế nào?
- [ ] `kubeadm reset` vừa chạy xong. Kể những thứ nó **không** dọn và hậu quả nếu quên từng thứ. Dữ
      liệu etcd bị xóa khi nào, không bị xóa khi nào — và vì sao reset `lab-ha-5` không làm mất object
      nào của Bookstore?
- [ ] Trên ba host etcd: vì sao cần `20-etcd-service-manager.conf`, tiền tố `20-` và dòng `ExecStart=`
      rỗng làm gì, kubelet ở đó nói chuyện với API server nào?
- [ ] `ca.key` của etcd nằm đâu và vì sao? Vì sao ba `server.crt` khác nhau, SAN trên `lab-ha-2` chứa
      gì, ai kiểm SAN đó khi Bookstore đang chạy? Chặn `2380` nhưng mở `2379` thì sao?
- [ ] Vì sao nhánh external buộc dùng `--config`? Cờ nào không dùng chung được với nó, khai thay ở đâu?
      Kể ba bằng chứng độc lập rằng cluster dùng etcd ngoài, và vì sao 0 Pod `component=etcd`.
- [ ] Mất quorum etcd: số đo của bạn cho ứng dụng và API là bao nhiêu? `lab-ha-1` vẫn chạy mà vì sao
      không khỏe? Người dùng mất gì, người vận hành mất gì?
- [ ] Tắt `lab-ha-5`: Bookstore gián đoạn bao lâu theo số đo, vì sao không về 0 ngay, chuỗi sự kiện
      nào đưa nó về 0? Vì sao Lab 8b không có gián đoạn này — có phải do topology etcd không?
- [ ] Dời `kube-apiserver.yaml` khỏi thư mục static Pod trên `lab-ha-5`: ai xóa Pod, vì sao node vẫn
      `Ready`, ai ngừng gửi request API sang đó, ứng dụng có biết không?
- [ ] Sau khi mất một control plane, stacked và external mỗi bên còn chịu được mất thêm bao nhiêu etcd
      member? Quorum nằm ở tầng nào, và vì sao 2 control plane + 3 etcd vẫn hợp lệ?
- [ ] ADR của bạn chọn topology nào cho Bookstore production, cần tối thiểu bao nhiêu host, đổi quyết
      định khi nào? Thành phần nào vẫn là điểm hỏng đơn trong cả hai topology lab đã dựng?
- [ ] Vì sao `db-0` phải ở `lab-ha-4` trong sprint này, trong khi chuỗi chính đặt `db` ở node được phép
      phá? Vì sao thay đổi đó nằm ở overlay chứ không ở `k8s/app`?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- **Nợ lab #8** — backup và restore etcd. Bạn đã dựng cụm etcd ba member và đọc được sức khỏe của nó,
  nhưng chưa có quy trình sao lưu nào; `etcdctl` bị giới hạn ở ba lệnh đọc. Đường lui duy nhất vẫn là
  snapshot VM. Trả ở [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa)
  — đọc lại bài [07](../07-setup-ha-etcd-with-kubeadm-vi.md) trước khi làm.
- **Nợ lab #7** — vòng đời certificate, dù lab vừa sinh một CA và tám certificate của etcd. Trả ở
  [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ).
- **Nợ kỹ thuật:** không phát sinh, không trả; các mã còn mở ở tag `lab-8b` giữ nguyên.

### Bài giải thích cuối cùng

Trong vài phút, kể lại hai luồng, không nhìn tài liệu:

1. **Từ cụm stacked đang chạy tới Bookstore trên cluster external.** Gỡ cluster cũ theo thứ tự nào và
   vì sao; những thứ phải dọn tay sau reset; vì sao sửa load balancer trước khi `init`; ba host etcd đi
   qua những gì để thành một cụm — kubelet, cấu hình, certificate, static Pod — và ở mỗi bước cái gì
   **riêng** từng host, cái gì **chung**; thứ duy nhất trong file cấu hình khiến cluster dùng etcd
   ngoài; ba bằng chứng khi có người hỏi "sao biết nó là external?"; rồi Bookstore lên cluster đó bằng
   repo — gỡ taint, ghim `db-0`, lệnh của Lab 8b.
2. **Cái giá và cái được của tách rời, bằng số đo.** Đi qua bốn kịch bản: tắt gì, API phản ứng ra sao,
   Bookstore phản ứng ra sao theo số đo từ máy host, và Lab 8b khác ở chỗ nào. Kết bằng quyết định
   trong ADR, số host tối thiểu cho production, điểm hỏng đơn còn lại, và chủ đề của giai đoạn 8 mà lab
   cố ý không làm.

Khi không còn nhầm *etcd member* với *control plane node*, *port 2380* với *port 2379*,
*`etcd.local`* với *`etcd.external`*, *node bị tắt* với *node thiếu một static Pod*, hay *mất quorum*
với *mất dịch vụ* — Lab 8c và
[giai đoạn 8](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) hoàn tất.

---

## 4. Troubleshooting của lab này

Sự cố **dựng môi trường** — VM không ping nhau, swap bật lại, containerd hỏng, package sai version —
nằm ở [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Sự cố của kubeadm
nói chung tra ở [bài 09](../09-troubleshooting-kubeadm-vi.md). Bảng dưới chỉ liệt kê sự cố phát sinh từ
nội dung sprint 14.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| Mọi gate qua `ssh <node> 'sudo …'` trả rỗng hoặc báo `a terminal is required` | `ssh <node> 'sudo -n true'` (B0.3) | `sudo` trên VM đòi mật khẩu — đúng giả định của Lab 8b. Chạy phần bên trong dấu nháy trực tiếp trên máy đó (quy ước của Lab 8b) rồi đối chiếu với dòng `PASS:` tương ứng, hoặc nhập mật khẩu qua `ssh -t` cho lệnh đơn lẻ. Không tắt mật khẩu `sudo` chỉ để qua gate |
| B0.5: `git describe` không ra `lab-8b` | `git -C ~/bookstore tag`; ngày của file bundle trên máy host | Bundle cũ. Lấy bundle mới nhất của Lab 8b, xóa `~/bookstore` trên `lab-ha-4` rồi clone lại |
| B0.5: thiếu một trong ba lệnh `-ha-lab`, overlay hoặc `scripts/do-san-sang.ps1` | `git -C ~/bookstore ls-files scripts tests k8s/overlays`; `grep -i -A6 ha-lab ~/bookstore/README.md` | Lab 8b bàn giao chưa đủ. Bổ sung repo ở tag `lab-8b`, đừng tự chế lệnh trong sprint này |
| **B2: `kubeadm reset` treo ở `Removing kubernetes-managed containers`** | `sudo crictl ps -a`; `systemctl is-active containerd` | Đúng tình huống *kubeadm bị treo khi xóa các container được quản lý* của bài 09: khởi động lại container runtime rồi chạy lại reset, dùng `crictl` gỡ lỗi. Thử đúng một lần; vẫn treo thì sang [đường lui](#đường-lui-khi-gate-reset-không-đạt) |
| B2: `/var/lib/etcd` còn dữ liệu sau reset | `ssh <node> 'sudo ls -A /var/lib/etcd'` | Reset là best-effort. Xóa tay theo lời giải B2 rồi chạy lại gate. Bắt buộc sạch: B6 dựng cụm mới với `initial-cluster-state: new` |
| B2, B7: `kubectl` báo `x509: certificate signed by unknown authority` | `kubectl config view --minify` | Kubeconfig cũ trỏ vào cluster mới — mục *Lỗi certificate TLS* của bài 09. Xóa `$HOME/.kube` rồi chép lại `admin.conf`. Không thêm `--insecure-skip-tls-verify` |
| B4: kubelet trên host etcd không lên `active` | `journalctl -u kubelet -n 80 --no-pager`; `systemctl cat kubelet` | (1) YAML sai trong `kubelet.conf` → viết lại bằng heredoc. (2) Hai `ExecStart` → thiếu dòng `ExecStart=` rỗng. (3) Lỗi cgroup driver → đối chiếu `SystemdCgroup = true` ở [A4.2](LAB-00-MOI-TRUONG-1.35.7.md#a42-cài-containerd-và-runc-đúng-version) |
| B5: `scp -r /tmp/${H}/*` báo `Permission denied` | `ls -l /tmp/192.168.100.232` | Thư mục còn thuộc `root`. Chạy `chown` đầu bước 5 rồi `scp` lại. Không `sudo scp` — khóa SSH nằm ở home của user thường |
| B5: xuất hiện `/etc/kubernetes/pki/pki` | `sudo ls -R /etc/kubernetes/pki \| head` | Gate trước `mv` đã FAIL mà vẫn đi tiếp. Xóa `/etc/kubernetes/pki` trên đúng host đó, làm lại bước 5 cho host đó — bản trong `/tmp` trên `lab-ha-1` vẫn còn |
| B6: container etcd không lên | `journalctl -u kubelet -n 80 --no-pager`; `sudo crictl ps -a`; `sudo crictl logs <id>` | (1) `cluster ID mismatch` hoặc `member already bootstrapped` → `/var/lib/etcd` chưa sạch, quay lại B2. (2) Không kết nối được peer → `2380` chưa thông; kiểm UFW theo [A4.4](LAB-00-MOI-TRUONG-1.35.7.md#a44-firewall-cho-mạng-lab-cô-lập). (3) `ImagePullBackOff` → kéo lại image |
| B6: image trong manifest khác image kubeadm khai | `kubeadm version -o short` trên từng host | Một host cài `kubeadm` khác version. Sửa ở tầng package theo [A4.3](LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl), xóa `etcd.yaml` rồi chạy lại `kubeadm init phase etcd local`. Không sửa tay tag |
| B6: `endpoint health` chỉ khỏe 1/3 hoặc 2/3 | `member list`; `ssh <host> 'sudo crictl ps --name etcd'` | Xử lý host vắng mặt theo dòng trên. Không `member add`/`member remove` |
| **B7: `kubeadm init` dừng ở giai đoạn chờ control plane** | Trên `lab-ha-lb`: `grep server /etc/haproxy/haproxy.cfg`; trên `lab-ha-4`: `sudo crictl logs <id apiserver>` | Đúng mục *kubeadm bị treo khi chờ control plane* của bài 09. (1) Backend HAProxy chưa đổi → B3. (2) Log apiserver không kết nối được etcd → kiểm ba file certificate và sức khỏe cụm ở B6. (3) Endpoint etcd sai → sửa file cấu hình, `sudo kubeadm reset -f`, `init` lại |
| B7: log apiserver có lỗi TLS khi nối tới etcd | Trên host etcd: `sudo openssl x509 -noout -text -in /etc/kubernetes/pki/etcd/server.crt \| grep -A1 'Alternative Name'` | SAN không chứa IP apiserver đang gọi, hoặc certificate client không do CA của cụm ký. Đối chiếu SAN theo script nghiệm thu B5 trước khi sinh lại |
| B8: `kubeadm join --control-plane` lỗi giải mã certificate | Thời gian từ lúc tải certificate tới lúc join | Secret `kubeadm-certs` và khóa giải mã hết hạn sau hai giờ (bài 08). Sinh lại theo lời giải B8 rồi join lại |
| B9: Pod Bookstore `Pending`, lý do `untolerated taint` | `kubectl -n bookstore describe pod <ten> \| tail -20` | Taint control plane chưa gỡ trên node đó — chạy lại bước gỡ taint |
| B9: Pod Bookstore `Pending`, lý do `Insufficient cpu` hoặc `memory` | `kubectl describe node lab-ha-4 \| grep -A8 'Allocated resources'` | Hai control plane của bộ HA nhỏ hơn worker của bộ chính. Giảm `replicas` của `web` bằng patch trong overlay (giữ Pod trên hai node), hoặc nâng RAM `lab-ha-4`, `lab-ha-5` khi máy tắt. Ghi lựa chọn vào runbook |
| B9: `db-0` vẫn nằm trên `lab-ha-5` | `kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/overlays/ha-lab \| grep -A12 'kind: StatefulSet'` | Patch đứng trước `db-emptydir.yaml` của Lab 8b nên `nodeSelector: null` của nó xóa mất ràng buộc mới, hoặc `target` không khớp. Đưa patch xuống cuối `patches`, đọc bản render trước khi triển khai lại |
| Cửa sổ đo của `Start-Probe` báo `khong tim thay curl.exe` hoặc lỗi không tìm thấy file | `Get-Command curl.exe`; `Test-Path $probe` | Windows thiếu `curl.exe` đi kèm hệ điều hành, hoặc `$probe` chưa được chép ở B0.5. Dừng, báo lại — lab không tự cài phần mềm lên máy host |
| Đường nền: `app` luôn `000` | `kubectl -n bookstore get svc web`; ping `192.168.100.234` từ máy host | Sai IP hoặc NodePort, hoặc Service `web` không phải NodePort `30080` — xem lại B9 |
| Đường nền: `api` luôn `401` hoặc `403` | `curl.exe -sk https://192.168.100.230:6443/version` | Apiserver không cho truy cập ẩn danh vào endpoint sức khỏe — khác mặc định kubeadm mà Lab 8b đã dựa vào. Lab không đổi cấu hình apiserver: ghi vào postmortem và dùng `cp_ok` trên `lab-ha-4` làm số liệu API |
| B10: Bookstore lỗi trong lúc mất quorum | `sudo crictl ps -a` trên `lab-ha-4`; log của `api`, `web` sau khi API trở lại | Không ép số liệu. Tìm container nào khởi động lại và vì sao, ghi vào postmortem — đó là phát hiện, không phải lỗi thao tác |
| B10: bật lại hai máy mà `endpoint health` mãi không đủ 3/3 | `ehealth`; trên từng host `sudo crictl logs <id etcd>` | Tăng số vòng chờ — thời gian member bắt kịp phụ thuộc cấu hình. Log báo `cluster ID mismatch` nghĩa là ai đó đã xóa dữ liệu của một member: lệch state, restore cả sáu VM về mốc gần nhất |
| B11: `lab-ha-5` mãi không thành `Unknown` | `kubectl get node lab-ha-5 -o yaml \| grep -A5 'type: Ready'` | Máy chưa tắt thật — kiểm `vmrun list`. Thời gian chuyển trạng thái phụ thuộc cấu hình của `kube-controller-manager`; tăng số vòng chờ |
| B11: dời manifest mà vẫn thấy hai Pod apiserver | `ssh lab-ha-5 'sudo ls -1 /etc/kubernetes/manifests'` | Dời nhầm máy, hoặc mirror Pod chưa kịp biến mất. Không `kubectl delete pod` — làm thế là bỏ mất bài học về static Pod |
| B13: `teardown-ha-lab.sh` in `teardown: con …` | `kubectl get all,configmaps,secrets -A -l app.kubernetes.io/part-of=bookstore` | Thường là namespace còn `Terminating` vì `db-0` đang trong ân hạn — chờ rồi chạy lại. Còn sót mãi thì ghi lại để sửa script ở repo; sprint này không đổi `scripts/` |
| B13: một VM không chuyển sang *Powered off* | `vmrun -T ws list` | Chờ thêm. Chỉ khi chờ mãi không xong mới `vmrun -T ws stop <vmx> hard`, và ghi vào evidence rằng lần tắt đó là cưỡng bức |
| B13: bộ VM chính có snapshot tiền tố `8x-` | So với `$mainVmx` ở B0.1 | Chụp nhầm. Xóa đúng snapshot đó trong Snapshot Manager, giữ nguyên `03-storage-ready` |

---

## 5. Nguồn chính thức

Phần giải thích ưu tiên snapshot tài liệu Kubernetes v1.35 để hành vi, flag và trường cấu hình khớp
minor version trong [bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa).

- [Kubernetes v1.35 — Options for Highly Available Topology](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)
- [Kubernetes v1.35 — Set up a High Availability etcd Cluster with kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/)
- [Kubernetes v1.35 — Creating Highly Available Clusters with kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [Kubernetes v1.35 — Creating a cluster with kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Kubernetes v1.35 — Troubleshooting kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)
- [Kubernetes v1.35 — `kubeadm reset`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
- [Kubernetes v1.35 — `kubeadm init`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)
- [Kubernetes v1.35 — `kubeadm init phase`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init-phase/)
- [Kubernetes v1.35 — `kubeadm join`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/)
- [Kubernetes v1.35 — kubeadm Configuration (v1beta4)](https://v1-35.docs.kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/)
- [Kubernetes v1.35 — Static Pods](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
- [Kubernetes v1.35 — Nodes](https://v1-35.docs.kubernetes.io/docs/concepts/architecture/nodes/)
- [Kubernetes v1.35 — EndpointSlices](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Kubernetes v1.35 — Taints and Tolerations](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Kubernetes v1.35 — Declarative Management of Kubernetes Objects Using Kustomize](https://v1-35.docs.kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [Kubernetes v1.35 — Using RBAC Authorization, mục *Default roles and role bindings*](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings) (vai trò `system:public-info-viewer` cho `/readyz`, `/version`)
- [Kubernetes v1.35 — Debugging Kubernetes nodes with crictl](https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)
- [Kubernetes v1.35 — Operating etcd clusters for Kubernetes](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- [kubeadm — Options for Software Load Balancing](https://git.k8s.io/kubeadm/docs/ha-considerations.md#options-for-software-load-balancing)
- [HAProxy — Configuration Manual](https://docs.haproxy.org/)
- [curl — Manual, tùy chọn `--write-out`](https://curl.se/docs/manpage.html#-w)
