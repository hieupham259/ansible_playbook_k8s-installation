# Runbook: Lab trên VMware — (A) 3 server hạ tầng + (B) Cụm K8s → Rancher → Cloudflare Tunnel → Domain

> **Runbook này gồm 2 lab độc lập** trên VMware (máy host Windows), đều dùng Ubuntu Server 24.04:
>
> - **Lab A — 3 server hạ tầng** ([§4](#4-tạo-và-nhân-bản-3-server-theo-serversmd)): dựng 1 VM gốc → snapshot → full-clone ra 3 server (`ubuntu-2404` / `load-balancer` / `teleport`) theo [`servers.md`](servers.md).
> - **Lab B — Cụm Kubernetes** ([§3](#3-tạo-3-vm-ubuntu-2404-trên-vmware) + [§5](#5-cấu-hình-os-chung-chạy-trên-cả-3-k8s-node)→[§17](#17-phụ-lục)): từ con số 0 dựng cụm k8s (1 master + 2 worker) bằng `kubeadm`, cài **Rancher** quản trị bằng UI, mua + cấu hình domain, tạo **Cloudflare Tunnel** trỏ domain vào ứng dụng trong cụm — **không cần IP public, không cần port-forward, không cần gọi nhà mạng**.
>
> ✅ Hai lab dùng hai nhóm IP riêng: Lab A dùng `.100/.101/.103`, cụm Kubernetes dùng `.111/.112/.113`. Có thể chạy đồng thời nếu cả sáu IP đều nằm ngoài DHCP pool hoặc đã được router giữ chỗ theo MAC.
>
> **Đối tượng:** lab/homelab cá nhân. Các bước có thể copy-paste.
>
> **Nguồn chính thức đã tham chiếu** (xem [§16](#16-nguồn-official)): kubeadm, container runtimes, Traefik, MetalLB, Cloudflare Tunnel, Rancher.

---

## Mục lục

1. [Tổng quan &amp; kiến trúc](#1-tổng-quan--kiến-trúc)
2. [Quy hoạch (versions, IP, hostname)](#2-quy-hoạch)
3. [Tạo 3 VM Ubuntu 24.04 trên VMware](#3-tạo-3-vm-ubuntu-2404-trên-vmware)
4. [Tạo và nhân bản 3 server theo servers.md](#4-tạo-và-nhân-bản-3-server-theo-serversmd)
5. [Cấu hình OS chung (CHẠY TRÊN CẢ 3 K8S NODE)](#5-cấu-hình-os-chung-chạy-trên-cả-3-k8s-node)
6. [Khởi tạo control plane (CHỈ MASTER)](#6-khởi-tạo-control-plane-chỉ-master)
7. [Join worker (CHỈ 2 WORKER)](#7-join-worker-chỉ-2-worker)
8. [Verify cụm](#8-verify-cụm)
9. [Cài Ingress Controller (Traefik)](#9-cài-ingress-controller-traefik)
10. [Deploy app mẫu + Ingress](#10-deploy-app-mẫu--ingress)
11. [Mua &amp; cấu hình domain trên Cloudflare](#section-11)
12. [Tạo Cloudflare Tunnel (chạy trong cụm)](#12-tạo-cloudflare-tunnel-chạy-trong-cụm)
13. [Trỏ domain &amp; kiểm tra trên Internet](#13-trỏ-domain--kiểm-tra-trên-internet)
14. [Cài Rancher 2.14.3 &amp; quản lý cụm](#14-cài-rancher-2143--quản-lý-cụm)
15. [Vận hành &amp; troubleshooting](#15-vận-hành--troubleshooting)
16. [Nguồn official](#16-nguồn-official)
17. [Phụ lục](#17-phụ-lục)

---

## 1. Tổng quan & kiến trúc

```
                Internet (người dùng gõ https://app.hieupn.site)
                          │
                          ▼
                 ┌──────────────────┐
                 │  Cloudflare Edge │  (TLS/HTTPS, WAF, DNS)
                 └────────┬─────────┘
                          │  kết nối OUTBOUND do cloudflared mở ra
                          │  (không có cổng inbound nào ở nhà bạn)
   ════════════ NAT/CGNAT nhà mạng + router nhà ════════════
                          │
   Máy host Windows ──────┼──────── VMware (Bridged) ───────────────┐
                          ▼                                          │
   ┌──────────────────────────────────────────────────────────┐    │
   │                  Kubernetes cluster                        │    │
   │                                                            │    │
   │   [Pod cloudflared] ──► Service: traefik (ClusterIP)       │    │
   │                              │                             │    │
   │                              ▼                             │    │
   │           ┌──────────────────┴───────────────────┐        │    │
   │           ▼                                       ▼        │    │
   │  [Ingress host=app.hieupn.site]      [Ingress host=rancher.hieupn.site]
   │           │                                       │        │    │
   │           ▼                                       ▼        │    │
   │   Service app ──► [Pod app]            [Rancher UI] (cattle-system)
   │                                                            │    │
   │   master (control plane) + worker1 + worker2               │    │
   └──────────────────────────────────────────────────────────┘    │
                                                                     │
   └─────────────────────────────────────────────────────────────────┘
```

**Vì sao kiến trúc này:**

- **Cloudflare Tunnel** mở kết nối **outbound** từ trong cụm ra Cloudflare → vượt CGNAT, không phơi IP nhà, không mở cổng router. Đây là lý do **không cần gọi nhà mạng tắt NAT / mua IP public**.
- **cloudflared chạy *trong* cụm** và trỏ thẳng vào **Service ClusterIP** của Traefik qua DNS nội bộ → **không cần MetalLB, không cần NodePort, không cần LoadBalancer external IP**.
- **Traefik** làm điểm vào duy nhất, định tuyến theo hostname → host nhiều app/nhiều domain (kể cả **UI Rancher**) chỉ bằng cách thêm `Ingress` + một public hostname trong tunnel. (Runbook **không** dùng `ingress-nginx` — dự án đó đã bị khai tử 03/2026, lý do ở [§9.2](#92-vì-sao-chọn-traefik-thay-cho-ingress-nginx).)
- **Rancher** cài *vào chính cụm kubeadm*, quản trị cụm đó (hiện trong UI là cụm `local`) bằng giao diện. Runbook dùng **Kubernetes 1.35.6 + Rancher 2.14.3**, nằm trong dải Kubernetes `1.33–1.35` của support matrix Rancher 2.14.3. Tuy nhiên, `kubeadm` thuộc nhóm “Any/imported cluster”, không phải một distro được SUSE chứng nhận riêng như RKE2/K3s.

---

## 2. Quy hoạch

### 2.1. Phiên bản (ghim để khỏi lệch version-skew)

| Thành phần         | Phiên bản dùng trong runbook                          | Ghi chú                                                                                                  |
| -------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Ubuntu Server        | **24.04.x LTS (Noble), amd64**                     | Cài bản Server, không cần GUI                                                                         |
| Kubernetes           | **v1.35.6**; gói Debian **`1.35.6-1.1`**  | Repo`pkgs.k8s.io` theo minor `v1.35`; 1.35 còn được Kubernetes duy trì tới **28/02/2027** |
| Container runtime    | **containerd 2.x từ Ubuntu 24.04**                | Giữ gói Ubuntu đã backport bản vá; cấu hình plugin 2.x và cgroup driver`systemd`               |
| CNI                  | **Flannel v0.28.7**; pod CIDR `10.244.0.0/16`    | Dùng manifest release-pinned, không dùng nhánh`master`/`latest`                                   |
| Ingress              | **Traefik chart 41.0.2** → Proxy **v3.7.6** | Chart khai`kubeVersion: >=1.25.0-0`; xem [§9](#9-cài-ingress-controller-traefik)                       |
| Tunnel               | **cloudflared `latest`**                         | Official manifest của Cloudflare cũng dùng tag này; chạy 2 replica + liveness probe                  |
| Cluster management   | **Rancher 2.14.3** từ `rancher-stable`          | Support matrix liệt kê Kubernetes**1.33–1.35** cho imported/other clusters                       |
| cert-manager         | **v1.21.1**                                        | Hỗ trợ và test với Kubernetes 1.33–1.36; dùng cho`ingress.tls.source=rancher`                     |
| Storage              | **local-path-provisioner v0.0.36**                 | Phù hợp homelab; dữ liệu gắn với node, không phải storage HA                                      |
| MetalLB (tuỳ chọn) | **v0.16.1**                                        | Chỉ cần khi muốn IP`LoadBalancer` trong LAN                                                          |

> ⚠️ **Version-skew:** runbook chủ động cài `kubelet`/`kubeadm`/`kubectl` cùng bản `1.35.6-1.1` trên cả 3 node. Chính sách Kubernetes hiện cho phép `kubelet` thấp hơn `kube-apiserver` tối đa **3 minor** và không được cao hơn; đó là biên tương thích, không phải lý do để cố tình để các node lệch bản trong một lab mới.
>
> **Phạm vi cam kết:** đây là baseline **tương thích và cài được cho homelab**, không phải bảo đảm “stable 100%” hay cấu hình Rancher production được SUSE chứng nhận end-to-end. Support matrix của Rancher ghi “Any” cho imported cluster; tài liệu production của Rancher khởi điểm ở 4 vCPU/16 GB **mỗi node** và yêu cầu upstream cluster HA 3 node.

### 2.2. IP & hostname (ví dụ — đổi theo dải LAN của bạn)

Giả sử LAN nhà bạn là `192.168.100.0/24`, gateway `192.168.100.1`. Chọn IP tĩnh **ngoài dải DHCP** của router.

| Vai trò      | Hostname        | IP tĩnh            | vCPU        | RAM            | Disk SSD |
| ------------- | --------------- | ------------------- | ----------- | -------------- | -------- |
| Control plane | `k8s-master`  | `192.168.100.111` | **4** | **8 GB** | 40 GB    |
| Worker 1      | `k8s-worker1` | `192.168.100.112` | 2           | **6 GB** | 40 GB    |
| Worker 2      | `k8s-worker2` | `192.168.100.113` | 2           | **6 GB** | 40 GB    |

> Pod CIDR `10.244.0.0/16` và Service CIDR mặc định `10.96.0.0/12` **không** được trùng dải LAN `192.168.100.0/24` → an toàn.
>
> ⚠️ Đây là cấu hình **homelab có chủ đích**, đồng bộ với [`servers.md`](servers.md), thấp hơn sizing production chính thức của Rancher. Disk 40 GB đủ để học và chạy workload nhẹ; cần tăng disk hoặc gắn storage riêng nếu giữ nhiều image, log, backup hay PVC.

### 2.2.1. Kiểm tra IP tĩnh KHÔNG trùng dải DHCP của router (làm 1 lần, trước khi cài)

> Vì sao: IP tĩnh `.111–.113` ở trên là **ví dụ**. Nếu chúng vô tình rơi **trong** dải mà router tự cấp (DHCP pool), router có thể cấp cùng IP đó cho một thiết bị khác → **trùng IP, cụm chập chờn**. Phải đảm bảo `.111/.112/.113` nằm **ngoài** dải DHCP. Làm **một** trong hai cách dưới (Cách A chắc chắn hơn).

**Trước tiên — xác định gateway router** (trên máy host Windows):

```powershell
ipconfig | findstr /i "Default Gateway"
# vd: Default Gateway . . . : 192.168.100.1   ← đây là địa chỉ trang quản trị router
```

**Cách A — Xem trực tiếp dải DHCP trên router (khuyến nghị):**

1. Mở trình duyệt trên host → vào `http://192.168.100.1` (đúng IP gateway vừa tìm).
2. Đăng nhập admin router (tài khoản/mật khẩu thường in ở **đáy router** hoặc theo nhà mạng Viettel/VNPT/FPT).
3. Tìm mục **LAN → DHCP Server / DHCP Settings** (tên tuỳ hãng TP-Link, Tenda, Asus…).
4. Ghi lại **Start IP** và **End IP** của dải DHCP — ví dụ `192.168.100.2 – 192.168.100.200`.
5. Đối chiếu `.111 / .112 / .113`:
   - **Nằm NGOÀI** dải đó → dùng được luôn, sang [§2.3](#23-yêu-cầu-máy-host).
   - **Nằm TRONG** dải (vd pool `.100–.200` thì `.111–.113` bị dính) → chọn 1 trong 2:
     - **(a)** Đổi IP tĩnh sang dải cao ngoài pool, vd `.201/.202/.203`, **rồi sửa đồng bộ** ở [bảng §2.2](#22-ip--hostname-ví-dụ--đổi-theo-dải-lan-của-bạn), [§5.1 `/etc/hosts`](#51-hostname--etchosts), [§5.2 netplan](#52-ip-tĩnh-netplan), [§6 `kubeadm init`](#6-khởi-tạo-control-plane-chỉ-master). **Hoặc**
     - **(b)** Thu hẹp **End IP** của pool trên router (vd kéo về `.110`) để chừa `.111–.113` ra ngoài.

**Cách B — Không vào được router? Ping thử xem IP có đang bị dùng (nhanh, kém chắc hơn):**

Trên host, ping từng IP định gán — IP "không ai trả lời" là khả năng đang trống:

```powershell
ping -n 2 192.168.100.111
ping -n 2 192.168.100.112
ping -n 2 192.168.100.113
```

- **"Request timed out" / "Destination host unreachable"** trên cả 2 gói → IP đang trống, nhiều khả năng dùng được.
- **"Reply from 192.168.100.x..."** → IP đang bị thiết bị khác chiếm → chọn IP khác.

> ⚠️ Cách B chỉ kiểm tra phụ: một thiết bị đang **tắt** sẽ không trả lời ping nhưng router vẫn có thể đã giữ/cấp lại IP đó. Cách chắc chắn nhất vẫn là **Cách A**. Chắc ăn nhất: chọn IP **ngoài pool** (Cách A) *hoặc* tạo **DHCP Reservation** trên router (gán cứng `.111–.113` theo MAC của từng VM) để router không bao giờ cấp trùng.

> ✅ Sau khi chốt được dải IP an toàn, ghi đè lại 3 IP đó vào [bảng §2.2](#22-ip--hostname-ví-dụ--đổi-theo-dải-lan-của-bạn) và dùng xuyên suốt runbook. **Tất cả** chỗ có `192.168.100.11x` phải khớp nhau.

### 2.3. Yêu cầu máy host

- RAM host nên **≥ 32 GB** (3 VM dùng tổng 20 GB, phần còn lại cho Windows/VMware và headroom).
- VMware Workstation Pro/Player (Player đủ dùng cho lab).
- ISO Ubuntu Server 24.04: [https://ubuntu.com/download/server](https://ubuntu.com/download/server)

---

## 3. Tạo 3 VM Ubuntu 24.04 trên VMware

Làm **3 lần** (master, worker1, worker2), chỉ khác tên + tài nguyên theo bảng [§2.2](#22-ip--hostname-ví-dụ--đổi-theo-dải-lan-của-bạn).

1. **Create a New Virtual Machine** → *Typical* → chọn ISO Ubuntu Server 24.04.
2. Đặt tên VM (vd `k8s-master`), chọn thư mục lưu.
3. Disk size 40 GB, *Store as a single file*.
4. **Customize Hardware** theo đúng từng dòng ở [§2.2](#22-ip--hostname-ví-dụ--đổi-theo-dải-lan-của-bạn):
   - `k8s-master`: 8192 MB, 4 vCPU.
   - `k8s-worker1` / `k8s-worker2`: 6144 MB, 2 vCPU mỗi máy.
   - **Network Adapter** → **Bridged** → tick *Replicate physical network connection state*.
     - ⚠️ **Virtual Network Editor là cửa sổ KHÁC với VM Settings.** Mở từ **menu cửa sổ chính VMware → Edit → Virtual Network Editor** (KHÔNG phải trong Settings của VM). Nếu các ô bị mờ → bấm **Change Settings** (cần quyền admin/UAC).
     - Chọn dòng **VMnet0** (Type = *Bridged*) → ở mục **"Bridged to:"** đổi từ *Automatic* sang **đúng tên card vật lý đang có mạng** (vd `Realtek PCIe GbE Family Controller` nếu dùng dây LAN, hoặc card `...Wi-Fi...` nếu dùng Wi-Fi) → **Apply → OK**. Để *Automatic* dễ bị bind nhầm card sau reboot.
     - **KHÔNG chọn** các card ảo trong danh sách: `Hyper-V Virtual Ethernet Adapter`, `TAP-Windows Adapter ... OpenVPN`… (không phải card mạng thật).
     - Không chắc card nào đang chạy? Trên host chạy `ipconfig /all`, tìm adapter có **IPv4 Address** dạng `192.168.x.x` + có **Default Gateway** → lấy đúng tên ở dòng *Description*.
     - 💡 **Nếu host bật Hyper-V** (thấy "Hyper-V Virtual Ethernet Adapter" trong danh sách "Bridged to") thì bridge VMware có thể bị Hyper-V chiếm card → DHCP không ra IP dù chọn đúng card. Cách xử lý ở [§15](#15-vận-hành--troubleshooting).
5. Power On → cài Ubuntu Server:
   - Chọn **Ubuntu Server (minimized hoặc full)**.
   - Network: cứ để DHCP khi cài, ta sẽ đặt IP tĩnh sau ([§5.2](#52-ip-tĩnh-netplan)).
   - **Profile**: tạo user `ubuntu`, đặt hostname đúng (`k8s-master`…).
   - **Tick "Install OpenSSH server"** để SSH vào cho tiện.
   - Bỏ qua các snap đề xuất.
6. Cài xong → reboot → đăng nhập.

> 💡 Sau khi cài, nên SSH từ máy host vào từng VM (`ssh ubuntu@192.168.100.111`) để copy-paste lệnh dễ hơn.

---

## 4. Tạo và nhân bản 3 server theo servers.md

> **Mục tiêu:** từ con số 0, dựng **1 VM Ubuntu 24.04 "gốc" (golden)** → **snapshot** → **full-clone** ra đủ 3 server trong [`servers.md`](servers.md), rồi tách mỗi bản thành **hostname + IP tĩnh riêng**. Nhanh hơn cài Ubuntu 3 lần (cách cài lặp xem [§3](#3-tạo-3-vm-ubuntu-2404-trên-vmware)).
>
> ⚠️ **Vì sao có phần này:** ngay sau khi clone, các VM còn trùng hostname, `machine-id` và SSH host key. VMware thường sinh MAC mới nên DHCP **có thể** cấp IP khác nhau; không được mặc định rằng IP chắc chắn trùng hay chắc chắn khác. [§4.4](#44-gỡ-trùng-lặp-trên-mỗi-bản-clone-machine-id--ssh-host-key) + [§4.5](#45-đặt-hostname--ip-tĩnh-riêng-cho-từng-server) chuẩn hoá từng máy.

**Bảng đích (theo [`servers.md`](servers.md) — cả 3 dùng Ubuntu Server 24.04):**

| # | Server / Hostname           | IP tĩnh            | RAM  | vCPU | Disk  | Domain                                      |
| - | --------------------------- | ------------------- | ---- | ---- | ----- | ------------------------------------------- |
| 1 | `ubuntu-2404` (bản gốc) | `192.168.100.100` | 4 GB | 2    | 40 GB | —                                          |
| 2 | `load-balancer`           | `192.168.100.101` | 2 GB | 1    | 40 GB | —                                          |
| 3 | `teleport`                | `192.168.100.103` | 2 GB | 1    | 40 GB | `https://teleport-onpre.devopseduvn.live` |

> ℹ️ Đây là inventory của **lab này** (theo `servers.md`). Nếu bạn dựng **cụm k8s** thì dùng tên/IP ở [§2.2](#22-ip--hostname-ví-dụ--đổi-theo-dải-lan-của-bạn) (`k8s-master/worker1/worker2`, `.111–.113`) — quy trình clone bên dưới **giống hệt**, chỉ đổi bảng đích.

### 4.1. Dựng VM gốc (golden) Ubuntu 24.04

1. Tạo **1** VM theo [§3](#3-tạo-3-vm-ubuntu-2404-trên-vmware) (bản gốc: 4 GB / 2 vCPU / 40 GB), đặt tên VMware là `ubuntu-2404`.
2. Cài Ubuntu Server 24.04, tạo user, **tick Install OpenSSH server**.
3. (Tùy chọn, chỉ cho LAN lab) Nếu muốn SSH trực tiếp bằng `root` và mật khẩu, đặt mật khẩu cho root rồi mở file cấu hình SSH:

```bash
sudo passwd root                 # đặt mật khẩu cho root; Ubuntu khóa mật khẩu root mặc định
sudo nano /etc/ssh/sshd_config   # mở cấu hình chính của SSH server
```

Trong file, thêm **trước dòng** `Include /etc/ssh/sshd_config.d/*.conf`:

```text
PermitRootLogin yes
PasswordAuthentication yes
```

- `PermitRootLogin yes`: cho phép user `root` đăng nhập trực tiếp qua SSH.
- `PasswordAuthentication yes`: cho phép SSH xác thực bằng mật khẩu.
- Đặt hai dòng trước `Include` để chúng là giá trị đầu tiên `sshd` đọc, tránh bị một file trong `sshd_config.d` đặt giá trị khác.

Lưu file (`Ctrl+O`, `Enter`, `Ctrl+X`), kiểm tra cú pháp rồi nạp lại dịch vụ:

```bash
sudo sshd -t                         # không có output = cú pháp hợp lệ
sudo systemctl restart ssh.service   # áp dụng cấu hình mới
sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication)'
# PASS: permitrootlogin yes  +  passwordauthentication yes
```

> ⚠️ `PermitRootLogin yes` làm tăng rủi ro dò mật khẩu. Chỉ dùng trong mạng lab tin cậy; với môi trường thật, dùng `PermitRootLogin prohibit-password` và SSH key theo [§5.8](#58-tùy-chọn-cho-phép-ssh-bằng-root-từ-client-cho-ansible--quản-trị). Giữ phiên SSH hiện tại mở và test `ssh root@<IP>` trong terminal khác trước khi đăng xuất.

4. **BẮT BUỘC trước khi snapshot/clone** — chặn cloud-init quản mạng để bản clone không bị revert DHCP (chi tiết ở [§5.2](#52-ip-tĩnh-netplan) bước 2):

```bash
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

5. (Khuyến nghị) Làm luôn các bước dùng chung ở [§5](#5-cấu-hình-os-chung-chạy-trên-cả-3-k8s-node) trên bản gốc **trước khi clone** để khỏi lặp 3 lần (update, tắt swap…). Nếu 3 server này **không** dùng làm k8s node thì bỏ qua các bước k8s-specific.

### 4.2. Snapshot VM gốc

VMware → chuột phải VM `ubuntu-2404` → **Snapshot → Take Snapshot** → đặt tên `golden-base`.

> Snapshot này vừa là mốc an toàn để revert, vừa là điểm gốc sạch trước khi nhân bản.

### 4.3. Full Clone ra 2 VM còn lại

VMware → chuột phải `ubuntu-2404` → **Manage → Clone** → nguồn **An existing snapshot (`golden-base`)** → **Create a full clone** (KHÔNG dùng *linked clone* để 3 VM độc lập):

- Clone lần 1 → đặt tên VM `load-balancer`.
- Clone lần 2 → đặt tên VM `teleport`.

> Xong bước này bạn có 3 VM còn trùng hostname, `machine-id` và SSH host key. IP DHCP có thể khác do MAC mới, nhưng chưa phải IP tĩnh trong inventory. [§4.4](#44-gỡ-trùng-lặp-trên-mỗi-bản-clone-machine-id--ssh-host-key) và [§4.5](#45-đặt-hostname--ip-tĩnh-riêng-cho-từng-server) sẽ tách chúng ra.

### 4.4. Gỡ trùng lặp trên MỖI bản clone (machine-id + SSH host key)

Chạy trên **`load-balancer`** và **`teleport`** (KHÔNG chạy trên bản gốc). Vào bằng **console VMware** (đừng SSH vì IP đang trùng):

```bash
# reset machine-id (tránh trùng DHCP DUID / định danh trùng nhau):
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo systemd-machine-id-setup

# reset SSH host key (tránh cảnh báo host key trùng khi SSH):
sudo rm -f /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
```

### 4.5. Đặt hostname + IP tĩnh riêng cho từng server

Làm trên **từng VM** qua console, theo bảng đích:

```bash
# 1) hostname — đổi theo từng máy:
sudo hostnamectl set-hostname ubuntu-2404       # bản gốc
#   hoặc:  load-balancer   /   teleport

# 2) xác định tên card mạng (thường là ens33 hoặc ens160):
ip -br a

# 3) bỏ cấu hình DHCP cũ do cloud-init đã sinh:
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak 2>/dev/null || true

# 4) tạo cấu hình IP tĩnh:
sudo nano /etc/netplan/01-static.yaml
```

Nội dung mẫu — đổi tên card và `addresses` theo từng VM:

```yaml
network:
  version: 2
  ethernets:
    ens33:                              # đổi theo kết quả `ip -br a`
      dhcp4: no
      addresses: [192.168.100.100/24]   # .100 / .101 / .103 theo bảng đích
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

```bash
sudo chmod 600 /etc/netplan/01-static.yaml
sudo netplan apply
```

> Đặt `.100` cho bản gốc, `.101` cho `load-balancer`, `.103` cho `teleport` theo `servers.md`. Không suy IP DHCP tạm thời thành IP đích.

(Tuỳ chọn) Muốn 3 máy gọi nhau bằng tên → thêm vào `/etc/hosts` trên **cả 3**:

```bash
sudo tee -a /etc/hosts >/dev/null <<'EOF'
192.168.100.100 ubuntu-2404
192.168.100.101 load-balancer
192.168.100.103 teleport
EOF
```

**Reboot mỗi VM** (bắt buộc sau khi gỡ trùng ở [§4.4] + đổi hostname/IP) — để `machine-id` mới có hiệu lực hẳn và hostname áp dụng đầy đủ:

```bash
sudo reboot
```

### 4.6. Verify 3 server

Sau reboot, trên **từng** VM:

```bash
hostnamectl                 # Static hostname đúng tên máy
ip -br a                    # đúng IP tĩnh .100 / .101 / .103, chỉ 1 IPv4
sudo netplan get            # dhcp4: false → xác nhận là tĩnh
```

Kiểm tra thông nhau (từ 1 máy bất kỳ hoặc từ client):

```bash
ping -c2 192.168.100.100 && ping -c2 192.168.100.101 && ping -c2 192.168.100.103
```

> ✅ Cả 3 có IP **khác nhau**, ping thông, và **giữ đúng IP sau reboot** (bằng chứng IP tĩnh thật) → xong phần hạ tầng. Nên chụp snapshot mới cho từng máy (vd `ip-ready`).

---

## 5. Cấu hình OS chung (CHẠY TRÊN CẢ 3 K8S NODE)

> Toàn bộ [§5] phải chạy **giống nhau trên cả 3 máy** (trừ phần hostname/IP là riêng từng máy). Chạy bằng `sudo` hoặc `sudo -i`.

### 5.0. Chuẩn hóa identity của 3 K8s node

Ba node Kubernetes phải có **hostname, IP, MAC address, `product_uuid`, `machine-id` và SSH host key riêng**. Làm trọn mục này ngay tại đây; không cần quay lại quy trình clone ở chương trước.

#### Trường hợp A — 3 VM được cài Ubuntu riêng

Không cần tạo lại `machine-id` hoặc SSH host key. Chuyển sang [§5.1](#51-hostname--etchosts)
để đặt hostname rồi chạy phần **Verify identity** tại đó.

#### Trường hợp B — 3 VM được tạo bằng snapshot/full clone

Chạy trên **mỗi bản clone** qua console VMware trước khi cấu hình hostname/IP. Không chạy qua SSH vì các clone có thể đang trùng IP và SSH host key:

```bash
# tạo machine-id mới:
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo systemd-machine-id-setup

# tạo SSH host key mới:
sudo rm -f /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
```

> **Chưa reboot tại đây.** Tiếp tục đặt hostname ở [§5.1](#51-hostname--etchosts), chạy gate
> identity ngay sau đó, cấu hình IP tĩnh ở [§5.2](#52-ip-tĩnh-netplan), rồi reboot một lần sau
> `netplan apply`. Nếu VM đã được reset identity trước đó thì không chạy lại. Reset SSH host key
> sẽ làm fingerprint của máy thay đổi, vì vậy client SSH có thể phải xóa entry cũ bằng
> `ssh-keygen -R <IP-cu>`.

### 5.1. Hostname + /etc/hosts

Trên **mỗi** máy, đặt đúng hostname:

```bash
# master:
sudo hostnamectl set-hostname k8s-master
# worker1:
sudo hostnamectl set-hostname k8s-worker1
# worker2:
sudo hostnamectl set-hostname k8s-worker2
```

#### Verify identity — bắt buộc cho cả hai trường hợp

Sau khi đặt hostname, chạy trên cả ba VM và đối chiếu kết quả:

```bash
hostnamectl --static
cat /etc/machine-id
sudo cat /sys/class/dmi/id/product_uuid
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
ip -br link
```

Gate trước khi tiếp tục:

- Hostname của mỗi VM phải đúng theo bảng §2.2.
- `machine-id` của ba VM phải khác nhau.
- SSH host-key fingerprint của ba VM phải khác nhau.
- MAC address của card mạng trên ba VM phải khác nhau.
- `product_uuid` của ba VM phải khác nhau.
- Nếu MAC hoặc `product_uuid` bị trùng, **dừng tại đây** và để VMware sinh UUID/MAC mới;
  reset file trong Ubuntu không sửa được hai giá trị này.

Thêm **giống nhau trên CẢ 3 máy** vào cuối `/etc/hosts` (bao gồm cả tên endpoint `k8s-master` dùng cho `--control-plane-endpoint`):

```bash
sudo tee -a /etc/hosts >/dev/null <<'EOF'
192.168.100.111 k8s-master
192.168.100.112 k8s-worker1
192.168.100.113 k8s-worker2
EOF
```

### 5.2. IP tĩnh (netplan)

**Chạy trên TỪNG máy** (đổi IP theo bảng [§2.2](#22-ip--hostname-ví-dụ--đổi-theo-dải-lan-của-bạn)). Nên làm qua **console của VM trong cửa sổ VMware** (không qua SSH), vì khi đổi IP phiên SSH sẽ rớt.

**1) Tìm tên card mạng:**

```bash
ip -br a
# PASS: interface ens33/ens160 ở trạng thái UP và có IP DHCP cùng dải LAN.

ip route
# PASS: có "default via <gateway> dev ens33/ens160".

ip route | awk '/default/ {print $5; exit}'
# PASS: chỉ in tên interface, ví dụ ens33; dùng đúng tên này trong 01-static.yaml.
```

**2) Chặn cloud-init quản mạng** (BẮT BUỘC — nếu bỏ qua, IP tĩnh có thể bị ghi đè về DHCP sau reboot):

```bash
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
# bỏ file netplan do cloud-init sinh (đang để DHCP) để tránh xung đột:
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak 2>/dev/null || true
```

**3) Tạo file tĩnh** `/etc/netplan/01-static.yaml` (vd `sudo nano /etc/netplan/01-static.yaml`) — **đổi IP & tên card theo từng máy**:

```yaml
network:
  version: 2
  ethernets:
    ens33:                          # ⚠️ đổi cho khớp tên ở bước 1
      dhcp4: no
      addresses: [192.168.100.111/24]   # ⚠️ .111 master / .112 w1 / .113 w2
      routes:
        - to: default
          via: 192.168.100.1            # gateway router (= Default Gateway của host)
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

**4) Áp dụng:**

```bash
sudo chmod 600 /etc/netplan/01-static.yaml   # tránh cảnh báo permission
sudo netplan apply
```

> ⚠️ Nếu đang làm qua SSH, sau `netplan apply` phiên sẽ **đứng/rớt** vì IP đã đổi — kết nối lại bằng IP mới: `ssh <user>@192.168.100.111`. Khi chỉ chỉnh nhỏ, có thể dùng `sudo netplan try` (tự hoàn tác sau 120s nếu mất mạng).

**5) Reboot một lần sau khi hoàn tất identity + hostname + IP:**

```bash
sudo reboot
```

> Bước reboot này đặc biệt bắt buộc với VM clone để `machine-id` mới, hostname mới và cấu hình mạng tĩnh có hiệu lực đầy đủ cùng lúc.

**6) Kiểm tra sau reboot (cả 3 máy đều phải đạt):**

```bash
grep -Fx 'network: {config: disabled}' \
  /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg

if test -e /etc/netplan/50-cloud-init.yaml; then
  echo 'FAIL: cloud-init regenerated 50-cloud-init.yaml'
else
  echo 'PASS: 50-cloud-init.yaml is absent'
fi

ip -br address                 # card hiển thị đúng IP tĩnh .111/.112/.113
ip route                       # có default route qua gateway LAN

getent hosts k8s-master k8s-worker1 k8s-worker2
ping -c 2 k8s-master
ping -c 2 k8s-worker1
ping -c 2 k8s-worker2

ping -c 2 192.168.100.1        # tới gateway → phải có reply
ping -c 2 8.8.8.8              # ra Internet bằng IP, không phụ thuộc DNS
getent hosts pkgs.k8s.io       # DNS phải phân giải được
curl -I --max-time 10 https://pkgs.k8s.io/
```

**PASS:** file chặn cloud-init tồn tại; output có `PASS: 50-cloud-init.yaml is absent`; IP
`.111–.113` vẫn giữ nguyên sau reboot; default route đúng; tên trả đúng IP; ping giữa mọi
node, gateway và Internet thành công; DNS cùng HTTPS egress hoạt động.

> Dùng `routes:` thay cho `gateway4` (đã deprecated từ Ubuntu 22.04+).

### 5.3. Cập nhật hệ thống & tắt swap

`kubeadm` yêu cầu **tắt swap**:

```bash
sudo apt-get update && sudo apt-get upgrade -y

# xác nhận quá trình nâng cấp không để lại gói lỗi/chưa cấu hình:
sudo dpkg --audit              # không trả dòng nào
sudo apt-get check             # hoàn tất mà không báo lỗi dependency

sudo swapoff -a
# tắt swap vĩnh viễn (comment dòng swap trong fstab):
sudo sed -ri '/^[^#].*[[:space:]]swap[[:space:]]/s/^/#/' /etc/fstab

# xác nhận swap đã tắt cả ở runtime và cấu hình khởi động:
swapon --show                  # không trả dòng nào
free -h                        # dòng Swap phải có Total = 0B
grep -nE '^[^#].*[[:space:]]swap[[:space:]]' /etc/fstab
                                # không trả dòng nào
```

Chỉ tiếp tục khi hai lệnh kiểm tra APT không báo lỗi, `swapon --show` và `grep` không trả dòng nào, đồng thời `free -h` có Swap = 0B. Nếu hệ thống có một swap unit riêng, xem tên bằng `systemctl --type=swap` rồi `disable --now` đúng unit đó; không mask toàn bộ `swap.target`.

**Đồng bộ thời gian (quan trọng cho etcd/cert):** lệch giờ giữa các node làm hỏng TLS handshake & etcd. Ubuntu bật `systemd-timesyncd` sẵn — chỉ cần xác nhận:

```bash
timedatectl                 # phải thấy: System clock synchronized: yes  +  NTP service: active
sudo timedatectl set-ntp true   # bật nếu chưa active
```

### 5.4. Kernel modules + sysctl (cho CRI & bridge networking)

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# verify: 2 module phải có mặt, cả 3 sysctl phải = 1
lsmod | grep -E 'br_netfilter|overlay'
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

> ⚠️ Flannel cần `br_netfilter`, nhưng kubeadm mới không còn đảm bảo preflight sẽ bắt thiếu module này. Nếu `lsmod` không có `br_netfilter` sau reboot, dừng lại sửa [§5.4] trước khi debug CNI.

### 5.5. Cài containerd + bật SystemdCgroup

**Cgroup là gì và vì sao Kubernetes cần nó?**

Container thực chất vẫn là các process Linux. **Cgroup** (*control group*) là cơ chế của kernel dùng để gom các process thành nhóm, theo dõi và cưỡng chế tài nguyên của từng nhóm: CPU, RAM, số process và I/O. Trong Kubernetes:

- scheduler dùng `resources.requests` để chọn node còn đủ tài nguyên;
- kubelet chuyển `requests`/`limits` của Pod cho container runtime;
- containerd/runc đưa process của container vào đúng cgroup;
- kernel mới là tầng thực sự throttle CPU hoặc kết thúc process vượt giới hạn RAM (`OOMKilled`).

Linux có cgroup v1 và v2. **Cgroup v2** dùng một hierarchy thống nhất cho CPU, memory, PID... nên quản lý và phân quyền nhất quán hơn v1. Ubuntu 24.04 bật cgroup v2 mặc định; Kubernetes hiện khuyến nghị v2 và một số tính năng quản lý tài nguyên mới chỉ có trên v2.

```bash
stat -fc %T /sys/fs/cgroup
```

Lệnh trên chỉ đọc loại filesystem tại `/sys/fs/cgroup`; kết quả mong muốn là `cgroup2fs`, nghĩa là OS đang dùng cgroup v2. Nếu không phải `cgroup2fs`, dừng lại kiểm tra `mount | grep cgroup` và `cat /proc/cmdline`; không tự ý sửa GRUB chỉ để vượt qua bước này.

Phân biệt ba khái niệm dễ nhầm:

| Thành phần    | Giá trị mong muốn      | Ý nghĩa                                    |
| --------------- | ------------------------- | -------------------------------------------- |
| Linux kernel    | cgroup v2 (`cgroup2fs`) | Phiên bản API cgroup của kernel           |
| kubelet         | driver`systemd`         | kubelet nhờ systemd quản lý cây cgroup   |
| containerd/runc | `SystemdCgroup = true`  | runtime cũng dùng systemd quản lý cgroup |

Kubelet và containerd phải dùng **cùng cgroup driver**. Kubeadm hiện mặc định cấu hình kubelet theo `systemd`, vì vậy cấu hình containerd bên dưới cũng đặt `SystemdCgroup = true`.

```bash
sudo apt-get install -y containerd

# Ubuntu 24.04 amd64 hiện cung cấp containerd 2.x.
# Ghi cấu hình tối thiểu theo đúng plugin path của containerd 2.x:
containerd --version
stat -fc %T /sys/fs/cgroup     # PASS trên Ubuntu 24.04: cgroup2fs
sudo mkdir -p /etc/containerd
cat <<'EOF' | sudo tee /etc/containerd/config.toml
version = 3

[plugins.'io.containerd.cri.v1.runtime'.containerd]
  default_runtime_name = 'runc'

[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc]
  runtime_type = 'io.containerd.runc.v2'

[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
  SystemdCgroup = true
EOF

sudo systemctl restart containerd
sudo systemctl enable --now containerd

# verify sớm: daemon active, CRI plugin không bị disable/fail
systemctl is-active containerd
sudo ctr plugins ls | grep 'io.containerd.cri.v1'
grep -n 'SystemdCgroup = true' /etc/containerd/config.toml
```

> ⚠️ **Bắt buộc:** cgroup driver của containerd phải trùng kubelet. Đường dẫn plugin của containerd 2.x là `io.containerd.cri.v1.runtime`; lệnh `sed` tìm `SystemdCgroup = false` theo cấu hình 1.x có thể không sửa gì mà vẫn exit 0. Kubernetes đã đưa cgroup v1 vào chế độ duy trì từ v1.31 và khuyến nghị cgroup v2; với Ubuntu 24.04 trong runbook này, nếu `stat` không trả `cgroup2fs` thì không tiếp tục `kubeadm init` cho đến khi xác định được vì sao OS không chạy mặc định cgroup v2.
>
> Runbook giữ **gói containerd của Ubuntu** để nhận backport bảo mật. Không thay bằng binary upstream chỉ vì số upstream cao hơn nếu chưa đối chiếu Ubuntu Security Notices/changelog.

#### "Backport bảo mật" nghĩa là gì

Câu trên nói về **cách lấy containerd**, không phải về một con số version. Nó có nghĩa: cài bằng `apt-get install containerd` từ kho Ubuntu, chứ không tải tarball từ GitHub releases của dự án containerd.

Ubuntu LTS chọn một version upstream rồi **giữ nguyên số version đó suốt vòng đời bản LTS**. Khi upstream sửa một CVE ở version mới hơn, Ubuntu không nhảy lên version mới — họ lấy riêng miếng vá đó ghép vào version đang đóng băng, rồi chỉ tăng phần revision đóng gói. Mổ xẻ chuỗi version mà runbook đang dùng:

```text
2.2.1 - 0ubuntu1 ~ 24.04.3
─────   ────────   ────────
  │        │           └── bản dựng lại cho series 24.04, lần thứ 3
  │        └── revision đóng gói của Ubuntu
  └── version upstream của containerd
```

Khi có CVE, cái đổi là `~24.04.3` → `~24.04.4`; phần `2.2.1` đứng yên. Bạn nhận bản vá mà không nhận thêm thay đổi hành vi, không đổi API, không đổi cú pháp `config.toml` — đúng thứ một LTS cần, và đúng thứ giữ cho cấu hình `io.containerd.cri.v1.runtime` ở trên không bị gãy giữa chừng.

Dấu `~` không phải trang trí: trong quy tắc so sánh version của dpkg, `~` sắp xếp **thấp hơn** cả chuỗi rỗng, nên `2.2.1-0ubuntu1~24.04.3` luôn nhỏ hơn `2.2.1-0ubuntu1`. Đó là cách bản backport không đè lên bản của series mới hơn khi bạn nâng cấp OS.

Nếu thay bằng binary upstream giải nén vào `/usr/local/bin`, ba thứ mất đi cùng lúc: Ubuntu Security Notices không còn áp dụng cho máy, `apt upgrade` không biết file đó tồn tại nên không bao giờ vá nó, và trách nhiệm theo dõi CVE của containerd chuyển hết sang bạn, thủ công, vĩnh viễn.

Xem cơ chế đang chạy trên node:

```bash
apt-cache policy containerd                # Installed / Candidate và đến từ pocket nào
apt-get changelog containerd | head -40    # các dòng CVE đã được ghép vào
```

Pocket `noble-updates` và `noble-security` là nơi các bản `~24.04.N` mới xuất hiện.

> ⚠️ Đánh đổi cần biết trước khi đọc [§5.6](#56-cài-kubeadm-kubelet-kubectl-repo-pkgsk8sio-pin-v1356): `apt-mark hold` **chặn đúng cơ chế này**. Package đã hold thì `apt upgrade` bỏ qua, kể cả khi bản vá CVE đã có trong `noble-security`. Runbook vì vậy hold tầng Kubernetes (`kubelet`/`kubeadm`/`kubectl`/`cri-tools`, nơi version-skew là ràng buộc cứng) nhưng **không** hold `containerd`/`runc` — đổi lại, runbook không biết node đang chạy revision nào cho tới khi chạy `apt-cache policy`. Lab 00 chọn ngược lại: hold cả runtime để snapshot tái lập được, và chấp nhận phải tự rà revision mới bằng tay.

`runc` cũng vào máy ở bước này dù lệnh cài không nhắc tên nó: `containerd` khai `Depends: libc6 (>= 2.34), runc` — **không kèm ràng buộc version nào**, nên apt lấy đúng bản `Candidate` của archive tại thời điểm chạy lệnh. Xem bản thật bằng `dpkg-query -W -f='${Package} ${Version} ${Architecture}\n' runc`.

### 5.6. Cài kubeadm, kubelet, kubectl (repo pkgs.k8s.io, pin v1.35.6)

> Repo cũ `apt.kubernetes.io` đã **deprecated từ 13/09/2023** — bắt buộc dùng `pkgs.k8s.io`.

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p -m 755 /etc/apt/keyrings

# Repo theo MINOR v1.35
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

# Xem Installed/Candidate và toàn bộ version cri-tools repo đang cung cấp:
apt-cache policy cri-tools
apt-cache madison cri-tools

# Gate version: chỉ gán/cài các version đã thấy trong output phía trên
apt-cache madison kubeadm | grep '1.35.6-1.1'
apt-cache madison cri-tools | grep '1.35.0-1.1'
VER='1.35.6-1.1'
CRI_TOOLS_VER='1.35.0-1.1'
sudo apt-get install -y \
  kubelet="$VER" \
  kubeadm="$VER" \
  kubectl="$VER" \
  cri-tools="$CRI_TOOLS_VER"

# `kubelet` từ repository Kubernetes kéo `kubernetes-cni` vào như dependency nên runbook này
# không khai báo package đó riêng. Lab 00 khai báo và khóa chính xác version `kubernetes-cni`
# để baseline của chuỗi lab có thể tái lập hoàn toàn.
#
# Hệ quả nằm ở chính dòng hold bên dưới: nó KHÔNG hold `kubernetes-cni`. Dependency mà kubelet
# khai chỉ là lower bound — `Depends: iptables (>= 1.4.21),kubernetes-cni (>= 1.2.0),mount,...`
# — chứ không phải pin, nên khi repo `core:/stable:/v1.35` phát hành một `kubernetes-cni` mới,
# `apt upgrade` sẽ nâng riêng gói này trong khi bốn gói kia đứng yên vì đã hold. Hiện repo chỉ
# cung cấp đúng một version (`1.8.0-1.1`) nên chưa xảy ra; đây là rủi ro tái lập, không phải
# lỗi đang có. Muốn khóa chặt như Lab 00 thì thêm `kubernetes-cni` vào cả lệnh `apt-get install`
# ở trên, dòng `apt-mark hold` bên dưới, và lệnh `apt-mark showhold` ở phần verify.
sudo apt-mark hold kubelet kubeadm kubectl cri-tools   # ghim version, tránh apt upgrade làm vỡ skew
sudo systemctl enable --now kubelet

# cấu hình crictl trỏ tường minh tới containerd
cat <<'EOF' | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

# verify package/binary vừa cài
kubeadm version -o short
kubelet --version
kubectl version --client
crictl --version
apt-mark showhold | grep -E '^(kubeadm|kubelet|kubectl|cri-tools)$'
systemctl is-enabled kubelet

# verify crictl kết nối được tới containerd và runtime thực sự dùng systemd cgroup
sudo crictl version
sudo crictl info | grep -i -A2 systemdCgroup

# verify lại toàn bộ prereq OS/runtime ở trạng thái hiện tại
free -h | grep -i swap
lsmod | grep -E 'br_netfilter|overlay'
sysctl \
  net.bridge.bridge-nf-call-iptables \
  net.bridge.bridge-nf-call-ip6tables \
  net.ipv4.ip_forward
systemctl is-active containerd
```

Kết quả PASS trước khi rời §5.6:

- `kubeadm`/`kubelet`/`kubectl`: `v1.35.6`; `crictl`: `v1.35.0`;
- cả `kubeadm`, `kubelet`, `kubectl`, `cri-tools` có trong danh sách hold; kubelet là `enabled`;
- `crictl version` kết nối được tới `containerd`, CRI API là `v1`;
- `SystemdCgroup: true`, containerd `active`;
- Swap = `0B`, có cả `br_netfilter` + `overlay`, ba sysctl đều bằng `1`.

> `kubelet` lúc này có thể restart/crashloop cho tới khi `kubeadm init`/`join` vì chưa có `/var/lib/kubelet/config.yaml` — **bình thường**. Không yêu cầu `systemctl is-active kubelet` phải PASS ở bước này. Nếu `crictl: command not found`, kiểm tra `apt-cache policy cri-tools`; không tiếp tục cho đến khi package `cri-tools` đã cài đúng. Nếu `crictl info` không cho thấy `SystemdCgroup: true`, sửa containerd trước khi tiếp tục.

### 5.7. Firewall

**Lab nhanh:** có thể tắt cho đỡ vướng (mạng nội bộ tin cậy):

```bash
sudo ufw disable
```

**Hoặc** mở đúng cổng nếu muốn giữ firewall:

| Phạm vi                                            | Cổng cần mở                                                               |
| --------------------------------------------------- | ---------------------------------------------------------------------------- |
| Control plane                                       | TCP`6443` từ các node/client quản trị; TCP `10250` từ control plane |
| Worker                                              | TCP`10250` từ control plane                                               |
| Tất cả node dùng Flannel VXLAN                   | UDP`8472` **giữa các node**                                        |
| Chỉ khi dùng NodePort từ LAN                     | TCP**và UDP** `30000-32767` từ LAN                                 |
| Chỉ khi load balancer cần health check kube-proxy | TCP`10256`                                                                 |
| Cụm nhiều control plane                           | TCP`2379-2380` chỉ giữa các control-plane/etcd member                   |

`10257` (controller-manager) và `10259` (scheduler) mặc định chỉ cần từ chính control plane, không mở rộng ra LAN. UFW mặc định cho outbound; nếu môi trường chặn egress, cho phép DNS/NTP, HTTPS `443`, và Cloudflare Tunnel **TCP+UDP `7844`**. Kiến trúc tunnel không cần mở inbound `80/443` trên router hay VM.

### 5.8. (Tùy chọn) Cho phép SSH bằng `root` từ client (cho Ansible / quản trị)

> ⚠️ **Bảo mật:** SSH thẳng bằng `root` tiện cho lab/Ansible nhưng kém an toàn hơn. Ưu tiên **root + SSH key** (Cách A); chỉ dùng **root + mật khẩu** (Cách B) trong mạng LAN tin cậy. Cách "chuẩn Ansible" hơn là SSH bằng user thường rồi `become: true` (sudo) — nhưng root vẫn chạy tốt cho homelab.
>
> **Chạy trên CẢ 3 node.**

⚙️ **Cái bẫy của Ubuntu 24.04:** đầu file `/etc/ssh/sshd_config` thường có dòng `Include /etc/ssh/sshd_config.d/*.conf`. Với phần lớn directive, `sshd` dùng **giá trị đầu tiên đọc được**; vì `Include` nằm đầu file nên một giá trị trong drop-in có thể được chốt trước dòng tương ứng ở phần dưới file chính. Trong các drop-in, file `00-...conf` được đọc trước `50-...conf` và `99-...conf`.

**Cách A — root + SSH key (khuyến nghị):**

1) Trên **máy client** (nơi chạy Ansible), tạo key nếu chưa có và xem public key:

```bash
ssh-keygen -t ed25519 -C "ansible@client"     # Enter hết các câu hỏi
cat ~/.ssh/id_ed25519.pub                      # copy nguyên dòng này
```

2) Trên **mỗi node**, nạp public key cho root + bật root login bằng key:

```bash
sudo mkdir -p /root/.ssh && sudo chmod 700 /root/.ssh
echo 'ssh-ed25519 AAAA...dán-key-từ-bước-1... ansible@client' | sudo tee -a /root/.ssh/authorized_keys
sudo chmod 600 /root/.ssh/authorized_keys
# 'prohibit-password' = cho login bằng key, CẤM mật khẩu (cũng là mặc định Ubuntu — ghi rõ cho chắc):
echo 'PermitRootLogin prohibit-password' | sudo tee /etc/ssh/sshd_config.d/00-root-login.conf
sudo sshd -t && sudo systemctl restart ssh     # sshd -t kiểm tra cú pháp trước khi restart
```

**Cách B — root + mật khẩu (đơn giản nhất, chỉ dùng trong LAN tin cậy):**

Trên **mỗi node**:

```bash
# 1) đặt mật khẩu cho root (mặc định Ubuntu root CHƯA có mật khẩu → không login được)
sudo passwd root

# 2) mở file cấu hình chính của SSH server
sudo nano /etc/ssh/sshd_config
```

Trong file, thêm hai dòng sau **trước dòng** `Include /etc/ssh/sshd_config.d/*.conf`:

```text
PermitRootLogin yes
PasswordAuthentication yes
```

- `PermitRootLogin yes`: cho phép đăng nhập trực tiếp bằng user `root`.
- `PasswordAuthentication yes`: cho phép xác thực bằng mật khẩu vừa đặt ở bước 1.
- Đặt trước `Include`: bảo đảm đây là giá trị đầu tiên được đọc, không bị drop-in của cloud-init chốt một giá trị khác trước đó.

Lưu bằng `Ctrl+O`, `Enter`, thoát bằng `Ctrl+X`, sau đó chạy:

```bash
# 3) kiểm tra cú pháp rồi nạp lại sshd
sudo sshd -t && sudo systemctl restart ssh
```

**Kiểm tra hiệu lực cuối cùng (trên node) và test (từ client):**

```bash
# trên node — xem giá trị THỰC SỰ đang áp dụng (sau khi gộp mọi drop-in):
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication'
#   Cách A → permitrootlogin prohibit-password
#   Cách B → permitrootlogin yes  +  passwordauthentication yes

# từ client:
ssh root@192.168.100.111        # vào được shell root là đạt (.112/.113 cho worker)
```

> Nếu vẫn bị từ chối, chạy `sudo sshd -T` như trên và kiểm tra vị trí hai dòng vừa thêm: chúng phải đứng trước `Include`. Có thể dùng `sudo grep -RniE '^[[:space:]]*(PermitRootLogin|PasswordAuthentication)' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/` để tìm mọi cấu hình liên quan.

> 💡 **Dùng cho Ansible:** sau khi bật, inventory chỉ cần `ansible_user=root` (Cách A thêm `ansible_ssh_private_key_file=~/.ssh/id_ed25519`). Nếu để user thường thì dùng `ansible_user=ubuntu` + `become: true`.

**🔁 Reboot chốt trước khi sang [§6]** (làm trên **cả 3 node**): sau khi xong [§5] — nhất là `apt-get upgrade` ở [§5.3] có thể đã cập nhật **kernel mới** — reboot rồi xác nhận cấu hình *sống sót qua reboot*:

```bash
sudo reboot
# sau khi node lên lại, kiểm tra:
uname -r
free -h | grep -i swap
lsmod | grep -E 'br_netfilter|overlay'
sysctl \
  net.bridge.bridge-nf-call-iptables \
  net.bridge.bridge-nf-call-ip6tables \
  net.ipv4.ip_forward
systemctl is-active containerd
sudo crictl info | grep -i -A2 systemdCgroup
```

**Quality gate trước §6 — cả 3 node đều phải PASS:**

- `uname -r` là kernel mới đã được APT cài (không còn cảnh báo *Pending kernel upgrade*);
- Swap = `0B`;
- có cả `br_netfilter` và `overlay`;
- `net.bridge.bridge-nf-call-iptables`, `net.bridge.bridge-nf-call-ip6tables`, `net.ipv4.ip_forward` đều bằng `1`;
- containerd là `active`;
- `crictl` kết nối được tới containerd và trả `SystemdCgroup: true`.

Nếu bất kỳ điều kiện nào FAIL, sửa lại [§5.3]/[§5.4]/[§5.5]/[§5.6] trên node đó **trước khi** `kubeadm init` hoặc `join`, tránh lỗi giữa chừng. (`kubelet` vẫn có thể crashloop tới khi init/join là **bình thường**.)

---

## 6. Khởi tạo control plane (CHỈ MASTER)

Chạy **chỉ trên `k8s-master`**.

(Tùy chọn) Kéo image trước để lộ sớm lỗi mạng/registry và để `init` nhanh hơn. **Không chạy**
`sudo kubeadm config images pull` mà không chỉ định version: kubeadm có thể dò remote và tự chọn một
patch mới hơn package đang cài (ví dụ binary `v1.35.6` nhưng image `v1.35.7`).

```bash
# Lấy đúng version từ binary kubeadm đã pin ở §5.6
KUBERNETES_VERSION="$(kubeadm version -o short)"
echo "$KUBERNETES_VERSION"                    # PASS: v1.35.6

# Subshell dừng gate khi version/list/pull lỗi nhưng không đóng phiên SSH cha.
(
  set -e
  test "$KUBERNETES_VERSION" = "v1.35.6" || {
    echo "FAIL: expected kubeadm v1.35.6, got $KUBERNETES_VERSION" >&2
    exit 1
  }

  # Xem trước và pull đúng bộ image của version vừa xác nhận.
  sudo kubeadm config images list --kubernetes-version "$KUBERNETES_VERSION"
  sudo kubeadm config images pull --kubernetes-version "$KUBERNETES_VERSION"
)
IMAGE_PULL_RC=$?

if [ "$IMAGE_PULL_RC" -eq 0 ]; then
  echo 'PASS: version kubeadm và bộ control-plane image đúng baseline'
else
  echo 'FAIL: gate version/image chưa đạt — KHÔNG chạy kubeadm init'
fi

# Trả đúng mã lỗi nhưng không đóng phiên SSH.
( exit "$IMAGE_PULL_RC" )
```

Khởi tạo control plane (lệnh chạy **vài phút** — kéo image + dựng etcd/control plane, **đừng ngắt giữa chừng**):

```bash
sudo kubeadm init \
  --kubernetes-version "$KUBERNETES_VERSION" \
  --control-plane-endpoint "k8s-master:6443" \
  --apiserver-advertise-address 192.168.100.111 \
  --pod-network-cidr 10.244.0.0/16
```

> Phải giữ `--kubernetes-version` ở cả lệnh `images pull` và `init`; nếu bỏ ở lệnh `init`, kubeadm vẫn
> có thể dò remote và chọn patch khác. Các image patch khác đã pull trước đó có thể giữ lại, chúng
> không được dùng khi `init` đã pin `v1.35.6`.

- `--control-plane-endpoint k8s-master:6443`: dùng tên (đã map trong `/etc/hosts`) → thuận lợi cho HA/đổi IP sau này.
- `--apiserver-advertise-address`: IP master (tránh kubeadm tự đoán nhầm card).
- `--pod-network-cidr 10.244.0.0/16`: **bắt buộc cho Flannel**.

Khi xong, kubeadm in ra các bước tiếp theo.

1. **Cấu hình kubeconfig** — chọn **một** trong hai cách sau.

   **Cách A — user thường (khuyến nghị, dùng lâu dài):** thoát khỏi phiên `root`, đăng nhập lại bằng
   user quản trị rồi chạy:

```bash
mkdir -p "$HOME/.kube"
sudo cp -i /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```

   **Cách B — đang thao tác bằng `root` (chỉ có hiệu lực trong terminal hiện tại):**

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

   Không cần chạy cả hai cách. Nếu dùng cách B, sau khi logout hoặc mở phiên SSH mới phải chạy lại
   lệnh `export`. Kiểm tra kubeconfig trước khi tiếp tục:

```bash
kubectl get nodes
```

2. **Lệnh `kubeadm join ...`** cho worker — **COPY LƯU LẠI** (sẽ dùng ở [§7](#7-join-worker-chỉ-2-worker)). Mất thì sinh lại bằng:

```bash
kubeadm token create --print-join-command
```

### 6.1. Cài CNI (Flannel) — chỉ master

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/download/v0.28.7/kube-flannel.yml
```

Đợi ~1 phút rồi kiểm tra master chuyển `Ready`:

```bash
kubectl get nodes
kubectl get pods -n kube-flannel
```

> Flannel mặc định dùng pod CIDR `10.244.0.0/16` → khớp `--pod-network-cidr` ở trên. Nếu muốn dùng **Calico** thì phải đặt pod CIDR custom (vd `10.244.0.0/16`) để **không trùng** LAN `192.168.x`.

---

## 7. Join worker (CHỈ 2 WORKER)

Trên **`k8s-worker1`** và **`k8s-worker2`**:

**Kiểm tra trước khi join** (bắt sớm lỗi `/etc/hosts`/firewall — không cần cài thêm gói):

```bash
ping -c2 k8s-master
timeout 3 bash -c 'echo > /dev/tcp/k8s-master/6443' && echo "API 6443 reachable" || echo "API 6443 UNREACHABLE"
```

Rồi chạy lệnh join đã lưu ở [§6] (dạng):

```bash
sudo kubeadm join k8s-master:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

> Worker phải resolve được `k8s-master` (đã thêm ở `/etc/hosts` [§5.1]).

---

## 8. Verify cụm

> **Nguyên tắc:** `STATUS = Ready` **không** có nghĩa là cụm chạy được app. Node vẫn báo Ready trong khi pod-to-pod giữa 2 worker đứt, DNS hỏng, hoặc `kubectl exec` chết. Ba thứ đó không hiện ra ở `kubectl get nodes` — phải test riêng.
>
> Verify theo **8 tầng dưới lên**: tầng dưới fail thì tầng trên chắc chắn fail, nên **đừng nhảy cóc**. Trừ [§8.0](#80-tầng-0--prereq-os-cả-3-node) chạy trên cả 3 node, còn lại chạy **trên master**.

### 8.0. Tầng 0 — Prereq OS (cả 3 node)

```bash
# swap phải rỗng hoàn toàn (kubelet từ chối khởi động nếu còn swap)
swapon --show ; free -h | grep -i swap        # PASS: không output / Swap = 0B

# kernel modules
lsmod | grep -E 'br_netfilter|overlay'        # PASS: cả 2 có mặt

# sysctl — cả 3 phải = 1
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

# containerd cgroup driver thực tế qua CRI
sudo crictl info | grep -i -A2 systemdCgroup     # PASS: systemdCgroup: true
systemctl is-active containerd kubelet           # PASS: active / active

# version-skew — runbook yêu cầu cả 3 node cùng v1.35.6 ([§2.1])
kubelet --version ; kubeadm version -o short

# đồng bộ giờ — lệch giờ làm TLS/etcd chết ngầm, rất khó lần ra
timedatectl | grep -E 'synchronized|Time zone'   # PASS: System clock synchronized: yes
```

> ⚠️ **Bẫy của Full Clone:** kubeadm định danh node bằng `product_uuid` + `machine-id`. Clone VM mà chưa chuẩn hóa identity theo [§5.0](#50-chuẩn-hóa-identity-của-3-k8s-node) thì 2 worker có thể "đè" lên nhau trong cụm — node thứ 2 join xong thì node thứ 1 biến mất. Chạy trên **từng node** rồi so, **3 giá trị phải khác nhau**:
>
> ```bash
> sudo cat /sys/class/dmi/id/product_uuid
> cat /etc/machine-id
> ip link show | grep link/ether        # MAC cũng phải khác
> ```

### 8.1. Tầng 1 — Control plane khỏe thật

```bash
kubectl get --raw='/readyz?verbose'
# PASS: mọi check "ok" + dòng cuối "readyz check passed"

kubectl get --raw='/livez?verbose' | grep -v ok    # PASS: không còn dòng nào

kubectl get pods -n kube-system -o wide
# PASS: etcd-, kube-apiserver-, kube-controller-manager-, kube-scheduler- đều Running, RESTARTS thấp
```

etcd health (etcd chạy dạng static pod trên master):

```bash
kubectl -n kube-system exec etcd-k8s-master -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key endpoint health
# PASS: "is healthy"
```

Hạn cert — kubeadm cấp **1 năm**, biết trước còn hơn cụm chết đột ngột sau 12 tháng:

```bash
sudo kubeadm certs check-expiration
```

### 8.2. Tầng 2 — Node & tài nguyên

```bash
kubectl get nodes -o wide
# PASS: 3 node Ready, ROLES = control-plane / <none> / <none>, VERSION giống nhau

# conditions: 3 cái *Pressure phải False, Ready phải True
kubectl get nodes -o custom-columns='NODE:.metadata.name,TYPE:.status.conditions[*].type,VAL:.status.conditions[*].status'

# taint — 2 worker KHÔNG được có taint nào
kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints'

# podCIDR phải được cấp đủ cho 3 node (Flannel dựa vào đây để dựng route)
kubectl get nodes -o custom-columns='NODE:.metadata.name,PODCIDR:.spec.podCIDR'
# PASS: 10.244.0.0/24, 10.244.1.0/24, 10.244.2.0/24
```

**Headroom cho Rancher** — sizing của lab đã tăng lên 8 GB cho master và 6 GB cho mỗi worker, nhưng vẫn thấp hơn sizing production chính thức. Đo mức dùng thực tế trước khi cài Rancher:

```bash
kubectl describe nodes | grep -A6 'Allocated resources'
# PASS: Memory Requests < ~50% ở thời điểm TRƯỚC khi cài Rancher
```

Giữ nguyên taint `NoSchedule` trên `k8s-master` để etcd/API server không tranh tài nguyên với Rancher và app. Với sizing ở [§2.2](#22-ip--hostname-ví-dụ--đổi-theo-dải-lan-của-bạn), không cần gỡ taint control plane.

### 8.3. Tầng 3 — Pod networking cross-node

> ⭐ Tầng **quan trọng nhất** và hay bị bỏ qua nhất. Cụm hỏng ở đây vẫn hiện `Ready` đủ 3 node.

```bash
kubectl get pods -n kube-flannel -o wide     # PASS: đúng 3 pod, mỗi node 1, Running
```

Lệnh trên xác nhận Flannel DaemonSet có đúng một pod trên mỗi node. Các pod này thiết lập interface,
route và VXLAN để các PodCIDR khác nhau liên lạc xuyên node. Cột `NODE` phải có `k8s-master`,
`k8s-worker1`, `k8s-worker2`; cả ba pod phải `Running`.

Test thật — tạo tường minh hai pod trên **hai worker khác nhau**, không dựa vào thứ tự mảng trả về từ
`kubectl`:

```yaml
# nettest.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nettest-w1
  labels: { app: nettest }
spec:
  nodeName: k8s-worker1
  containers:
    - name: netshoot
      image: nicolaka/netshoot
      command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: nettest-w2
  labels: { app: nettest }
spec:
  nodeName: k8s-worker2
  containers:
    - name: netshoot
      image: nicolaka/netshoot
      command: ["sleep", "3600"]
```

- `nodeName` ép `nettest-w1` lên worker1 và `nettest-w2` lên worker2. Nếu để scheduler tự chọn, hai
  pod có thể cùng nằm trên một node; ping thành công khi đó không chứng minh cross-node networking.
- Label `app: nettest` cho phép chọn đồng thời cả hai pod bằng `-l app=nettest`.
- Image `netshoot` có sẵn các công cụ mạng như `ping`, `curl`, `dig`; `sleep 3600` giữ container chạy
  một giờ để có thể `kubectl exec` vào kiểm tra.

Tạo pod, đợi cả hai thật sự `Ready`, rồi xác nhận vị trí và Pod IP:

```bash
kubectl apply -f nettest.yaml
kubectl wait --for=condition=Ready pod/nettest-w1 pod/nettest-w2 --timeout=180s
kubectl get pod -l app=nettest -o wide
```

PASS khi:

- `kubectl wait` trả `condition met` cho cả hai pod;
- hai pod đều `1/1 Running`;
- `nettest-w1` nằm trên `k8s-worker1`, IP thuộc `10.244.1.0/24`;
- `nettest-w2` nằm trên `k8s-worker2`, IP thuộc `10.244.2.0/24`.

Lấy Pod IP hiện tại của `nettest-w2` bằng JSONPath, rồi chạy `ping` **từ bên trong** `nettest-w1`.
Dùng biến thay vì copy IP thủ công vì Pod IP có thể đổi khi pod được tạo lại:

```bash
BIP=$(kubectl get pod nettest-w2 -o jsonpath='{.status.podIP}')
kubectl exec nettest-w1 -- ping -c3 "$BIP"    # PASS: 0% packet loss
```

Luồng được kiểm tra:

```text
nettest-w1 (10.244.1.x, worker1)
  → Flannel/VXLAN giữa các node (UDP 8472)
  → nettest-w2 (10.244.2.x, worker2)
```

`3 packets transmitted, 3 received, 0% packet loss` chứng minh đường Pod network theo chiều
worker1 → worker2 hoạt động. Đây là kiểm tra kết nối IP/ICMP layer 3; nó chưa kiểm tra DNS, Service
hay kube-proxy — các tầng tiếp theo sẽ kiểm tra riêng.

> ❌ **Fail ở đây** = VXLAN **UDP 8472** bị chặn giữa các node ([§5.7](#57-firewall)), hoặc Flannel bind nhầm interface, hoặc `--pod-network-cidr` lúc `kubeadm init` không phải `10.244.0.0/16`. Đây là nguyên nhân số 1 của triệu chứng *"cụm nhìn Ready mà app không gọi được nhau"*.

Giữ nguyên hai pod sau bước này: §8.4–§8.6 còn dùng `nettest-w1` để kiểm tra DNS, Service và
`kubectl exec`. Dọn chúng cùng các resource test khác tại [§8.9](#89-dọn-dẹp-resource-test).

### 8.4. Tầng 4 — DNS

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide   # PASS: 2 pod CoreDNS Running

TEST_POD=nettest-w1

# DNS nội bộ cụm
kubectl exec "$TEST_POD" -- nslookup kubernetes.default.svc.cluster.local   # PASS: trả 10.96.0.1

# DNS ra ngoài Internet
kubectl exec "$TEST_POD" -- nslookup releases.rancher.com           # PASS: resolve ra IP public
```

> DNS ngoại **bắt buộc** phải chạy: không có nó thì Helm không tải được chart và kubelet không kéo nổi image cho Traefik / cert-manager / Rancher.

### 8.5. Tầng 5 — Service & kube-proxy

Mục đích của tầng này là kiểm tra toàn bộ data path của Kubernetes Service:

```text
client → tên Service/ClusterIP hoặc NodeIP:NodePort → kube-proxy → EndpointSlice → backend Pod
```

Một cluster có thể có node `Ready`, Pod network và DNS đều tốt nhưng Service vẫn hỏng nếu kube-proxy
không lập được rule chuyển tiếp. Trước tiên xác nhận kube-proxy có đúng một pod trên mỗi node:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide   # PASS: 3 pod Running
```

Tạo ba nginx backend. `rollout status` chỉ PASS khi Deployment có đủ replica sẵn sàng phục vụ:

```bash
kubectl create deploy web --image=nginx --replicas=3
kubectl rollout status deploy/web
```

Tạo Service `web` loại mặc định `ClusterIP`. EndpointSlice là danh sách Pod IP thật đứng sau Service;
nếu cột `ENDPOINTS` rỗng thì selector không chọn được backend và Service không thể chuyển traffic:

```bash
kubectl expose deploy web --port=80
kubectl get endpointslice -l kubernetes.io/service-name=web
# PASS: cột ENDPOINTS liệt kê các Pod IP
```

Gọi Service bằng tên ngắn `web` từ pod test. Lệnh này đồng thời xác nhận DNS Service, ClusterIP,
kube-proxy load-balancing và đường mạng tới một backend; HTTP `200` là PASS:

```bash
TEST_POD=nettest-w1
kubectl exec "$TEST_POD" -- curl -s -o /dev/null -w '%{http_code}\n' http://web
# PASS: 200
```

Tạo thêm Service `NodePort`, lấy port được Kubernetes cấp động rồi gọi qua IP của cả ba node:

```bash
kubectl expose deploy web --name=web-np --type=NodePort --port=80
NP=$(kubectl get svc web-np -o jsonpath='{.spec.ports[0].nodePort}')
for ip in 192.168.100.111 192.168.100.112 192.168.100.113; do
  curl -s -o /dev/null -w "$ip -> %{http_code}\n" http://$ip:$NP
done                                   # PASS: cả 3 IP đều trả 200
```

NodePort mở trên **mọi** node, kể cả node không có backend Pod; kube-proxy nhận traffic rồi forward tới
một endpoint có thể nằm trên node khác. Cả ba IP trả `200` chứng minh NodePort và đường forward
xuyên node hoạt động. Nếu chỉ 1/3 IP trả `200`, kiểm tra kube-proxy, firewall và CNI
([§8.3](#83-tầng-3--pod-networking-cross-node)).

> Chạy vòng `curl` trên master kiểm tra data path NodePort qua các Node IP. Muốn xác nhận thêm firewall
> và khả năng truy cập từ LAN, chạy `curl http://<NodeIP>:<NP>` từ một máy khác cùng mạng.

### 8.6. Tầng 6 — Đường control plane → kubelet (Rancher UI sống chết ở đây)

Rancher UI dùng **exec / logs / port-forward**, tất cả đi qua apiserver → **kubelet cổng 10250**. Tầng này đứt thì cụm vẫn deploy app bình thường, nhưng tab *Shell* và *Logs* trong Rancher sẽ trắng — và rất dễ mất hàng giờ đi tìm nhầm chỗ.

```bash
kubectl logs deploy/web                      # PASS: ra log nginx
TEST_POD=nettest-w1
kubectl exec "$TEST_POD" -- hostname          # PASS: ra tên pod

kubectl port-forward svc/web 18080:80 >/tmp/web-port-forward.log 2>&1 &
PF_PID=$!
for i in {1..20}; do
  curl -fsS http://127.0.0.1:18080 >/dev/null && break
  sleep 1
done
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:18080   # PASS: 200
kill "$PF_PID"
```

### 8.7. Tầng 7 — Công cụ và add-on kubeadm KHÔNG cài sẵn

**Cài Helm trước khi dùng** — các bước metrics-server, Traefik, cert-manager và Rancher phía sau đều cần Helm 3. Rancher yêu cầu chọn Helm 3 tương thích với dải Kubernetes đang dùng; baseline này đặt gate thực dụng **Helm ≥ 3.18**:

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version --short
# PASS: v3.18.0 trở lên
```

**a) metrics-server** — thiếu nó thì `kubectl top` lỗi, HPA không hoạt động, và dashboard Rancher không vẽ được đồ thị CPU/RAM. Trên kubeadm phải thêm `--kubelet-insecure-tls` vì kubelet dùng serving cert self-signed:

```bash
helm upgrade --install metrics-server metrics-server \
  --repo https://kubernetes-sigs.github.io/metrics-server/ \
  -n kube-system --set 'args={--kubelet-insecure-tls}'

kubectl -n kube-system rollout status deploy/metrics-server
kubectl top nodes ; kubectl top pods -A      # PASS: ra số liệu, không "metrics not available"
```

**b) Default StorageClass** — Rancher core lưu dữ liệu trong etcd nên **không** cần PVC, nhưng mọi app có state và các chart trong Rancher marketplace (**Monitoring, Logging, Longhorn**) đều cần:

```bash
kubectl get storageclass          # cụm kubeadm thuần: RỖNG
```

Lab 1 master 2 worker thì `local-path-provisioner` của Rancher là lựa chọn gọn. Cài bản stable được upstream hướng dẫn, đặt `local-path` làm mặc định, rồi verify:

**Mô hình Pod → PVC → StorageClass → PV:**

```text
Pod tham chiếu PVC
  → PVC yêu cầu dung lượng và access mode từ StorageClass
  → provisioner tạo PV trên disk của node đã chọn
  → PVC bind với PV
  → kubelet mount PV vào mountPath bên trong container
```

- **PVC (PersistentVolumeClaim)** là yêu cầu của ứng dụng, ví dụ “cần `1Gi`, `ReadWriteOnce`”. Pod
  chỉ tham chiếu tên PVC và thấy đường dẫn như `/data` trong container; Pod không cần biết thư mục thật
  trên disk của worker.
- **StorageClass** là chính sách cấp volume. PVC có thể chỉ định `storageClassName`; nếu bỏ trống thì
  dùng class có `(default)`. Trong runbook này, `local-path` gọi provisioner `rancher.io/local-path`.
- **PV (PersistentVolume)** là volume Kubernetes tạo để đáp ứng PVC. Với `local-path`, PV là dữ liệu
  nằm cục bộ trên disk của một node, không phải storage dùng chung giữa ba node.

Do `local-path` dùng `WaitForFirstConsumer`, provisioner không chọn worker ngẫu nhiên ngay lúc tạo
PVC. Scheduler chọn node phù hợp cho Pod consumer trước, rồi PV được tạo trên node đó và Pod được
mount volume. Nếu Pod được tạo lại, Kubernetes phải đặt Pod về node giữ PV; node hoặc disk đó hỏng thì
dữ liệu không tự failover sang worker khác. Dung lượng `1Gi` của local-path cũng không nhất thiết là
quota filesystem cứng; phải tiếp tục giám sát dung lượng disk thật của worker.

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.36/deploy/local-path-storage.yaml
kubectl -n local-path-storage rollout status deploy/local-path-provisioner
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get storageclass

kubectl create -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: pvc-test }
spec:
  storageClassName: local-path
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 1Gi } }
EOF

# local-path dùng WaitForFirstConsumer nên PVC vẫn Pending cho tới khi có Pod sử dụng
kubectl get pvc pvc-test          # EXPECTED ở bước này: STATUS = Pending

kubectl create -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pvc-test-pod
spec:
  containers:
    - name: test
      image: busybox:1.37
      command:
        - sh
        - -c
        - echo "local-path works" > /data/test.txt && sleep 3600
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc-test
EOF

kubectl wait --for=condition=Ready pod/pvc-test-pod --timeout=180s
kubectl get pvc pvc-test          # PASS: STATUS = Bound
kubectl get pv
kubectl get pod pvc-test-pod -o wide
kubectl exec pvc-test-pod -- cat /data/test.txt
# PASS: local-path works

# cleanup: xóa consumer trước, sau đó mới xóa claim
kubectl delete pod pvc-test-pod
kubectl delete pvc pvc-test
```

`VOLUMEBINDINGMODE=WaitForFirstConsumer` trì hoãn việc tạo/bind PV cho tới khi scheduler biết Pod consumer
sẽ chạy trên node nào; vì vậy PVC `Pending` trước khi tạo `pvc-test-pod` là **đúng thiết kế**, không
phải lỗi. `ALLOWVOLUMEEXPANSION=false` cũng là giá trị mong đợi của StorageClass này: không thể tăng
dung lượng claim hiện hữu bằng cách sửa `requests.storage`.

> ⚠️ `local-path` lưu dữ liệu trên disk của node được chọn. Node mất disk thì volume không tự failover;
> chỉ dùng cho lab hoặc workload chấp nhận rủi ro này. Xóa Pod không làm mất PVC ngay, nhưng
> `RECLAIMPOLICY=Delete` sẽ xóa PV và dữ liệu local tương ứng khi PVC bị xóa.

### 8.8. Gate cụm nền — trước khi cài các add-on

| #  | Kiểm tra                    | Lệnh                                                                                | PASS                                     |
| -- | ---------------------------- | ------------------------------------------------------------------------------------ | ---------------------------------------- |
| 1  | Helm đúng baseline         | `helm version --short`                                                             | **≥ v3.18**                       |
| 2  | Kubernetes đúng baseline   | `kubectl version`                                                                  | Client/Server**v1.35.6**           |
| 3  | 3 node Ready, worker 0 taint | [§8.2](#82-tầng-2--node--tài-nguyên)                                              | 3/3                                      |
| 4  | Pod cross-node thông        | [§8.3](#83-tầng-3--pod-networking-cross-node)                                       | 0% packet loss                           |
| 5  | DNS nội + ngoại            | [§8.4](#84-tầng-4--dns)                                                             | cả 2 resolve                            |
| 6  | RAM còn headroom            | `kubectl describe nodes \| grep -A6 Allocated`                                      | Requests < 50% trước Rancher           |
| 7  | quyền cluster-admin         | `kubectl auth can-i '*' '*' --all-namespaces`                                      | `yes`                                  |
| 8  | exec / logs / port-forward   | [§8.6](#86-tầng-6--đường-control-plane--kubelet-rancher-ui-sống-chết-ở-đây) | cả 3 lệnh OK                           |
| 9  | metrics-server               | `kubectl top nodes`                                                                | đủ số liệu 3 node                    |
| 10 | Default StorageClass         | `kubectl get storageclass`                                                         | `local-path` có `(default)`         |
| 11 | Storage provisioner          | `kubectl -n local-path-storage rollout status deploy/local-path-provisioner`       | `successfully rolled out`              |
| 12 | Pull image Traefik           | Trên**từng worker**: `sudo crictl pull docker.io/traefik:v3.7.6`           | cả 2 worker trả về image reference/ID |

Gate nhanh chạy trên `k8s-master` bằng user `ubuntu` ngay trước §9:

```bash
helm version --short                                      # PASS: >= v3.18
kubectl version                                           # PASS: Client/Server v1.35.6
kubectl get nodes                                         # PASS: 3/3 Ready
kubectl auth can-i '*' '*' --all-namespaces               # PASS: yes
kubectl top nodes                                         # PASS: có CPU/RAM của cả 3 node
kubectl get storageclass                                  # PASS: local-path (default)
kubectl -n local-path-storage rollout status \
  deploy/local-path-provisioner                           # PASS: successfully rolled out
```

Khả năng pull image là trạng thái riêng của runtime trên từng node. Master đang giữ taint `NoSchedule`,
nên chạy lệnh sau **trực tiếp trên `k8s-worker1`, rồi lặp lại trên `k8s-worker2`**; chỉ chạy trên master
không chứng minh node có thể chạy Pod Traefik:

```bash
sudo crictl pull docker.io/traefik:v3.7.6
# PASS: trả về image reference/ID; không có lỗi DNS, TLS, timeout hoặc rate-limit
```

Chỉ tiếp tục khi Helm đạt baseline, server Kubernetes đúng version, đủ 3 node `Ready`, quyền trả
`yes`, metrics có số liệu, `local-path` có `(default)`, provisioner `successfully rolled out` và cả
hai worker pull được image Traefik.
`WaitForFirstConsumer` và `ALLOWVOLUMEEXPANSION=false` của `local-path` là giá trị mong đợi, không
phải lỗi.

### 8.9. Dọn dẹp resource test

```bash
kubectl delete pod nettest-w1 nettest-w2
kubectl delete deploy web
kubectl delete svc web web-np
```

---

## 9. Cài Ingress Controller (Traefik)

> **Đọc §9.1 trước khi cài.** Mục tiêu của mục 9 không chỉ là tạo một Pod Traefik, mà là tạo **điểm vào HTTP/HTTPS chung** cho nhiều ứng dụng. Trong kiến trúc của runbook, Cloudflare Tunnel chạy trong cluster và gọi Traefik qua mạng nội bộ, nên Traefik được cài với Service `ClusterIP`; **không cần NodePort, external IP hay MetalLB** cho luồng Internet chính.

### 9.1. Kiến thức nền và mục đích của mục 9

#### 9.1.1. Từ Pod đến một địa chỉ ổn định: Service và EndpointSlice

Pod có IP riêng nhưng Pod là tài nguyên tạm thời: khi bị tạo lại, IP có thể đổi. Client vì vậy không nên nhớ IP của từng Pod. **Service** tạo một tên DNS và địa chỉ ổn định phía trước một nhóm Pod; selector của Service chọn các Pod backend. Control plane ghi danh sách IP/port backend hiện tại vào **EndpointSlice** và tự cập nhật khi Pod thêm, mất hoặc đổi trạng thái.

Ví dụ đường đi bên trong cluster:

```text
client → web.default.svc.cluster.local:80
       → Service web, port 80
       → EndpointSlice: 10.244.1.2:80, 10.244.2.3:80, ...
       → một Pod web đang Ready
```

Ba trường port thường gặp:

| Trường       | Ý nghĩa                                                                                                      |
| -------------- | -------------------------------------------------------------------------------------------------------------- |
| `port`       | Cổng mà client dùng trên Service                                                                           |
| `targetPort` | Cổng ứng dụng lắng nghe trong Pod; mặc định bằng`port`                                               |
| `nodePort`   | Cổng được mở trên mỗi node, chỉ có với`NodePort` hoặc khi một `LoadBalancer` có cấp NodePort |

> Dùng `kubectl get endpointslice -l kubernetes.io/service-name=<service>` để kiểm tra Service đã tìm thấy Pod backend hay chưa. API `Endpoints` cũ đã deprecated từ Kubernetes 1.33; runbook ưu tiên `EndpointSlice`.

#### 9.1.2. Pod gọi Pod: DNS, kube-proxy và load balancing theo kết nối

Tình huống phổ biến nhất trong cluster: ứng dụng A gọi ứng dụng B, mỗi bên chạy 2–3 Pod. A **không gọi thẳng IP của một Pod B** mà gọi qua Service của B. Cơ chế gồm ba tầng:

1. **Mạng phẳng của CNI.** Flannel cấp mỗi Pod một IP trong `10.244.0.0/16` và mọi Pod tới được IP của mọi Pod khác, kể cả khác node — chính là điều đã verify ở [§8.3](#83-tầng-3--pod-networking-cross-node). Đây là nền tảng bắt buộc, nhưng IP Pod là tạm thời ([§9.1.1](#911-từ-pod-đến-một-địa-chỉ-ổn-định-service-và-endpointslice)) nên không dùng trực tiếp.
2. **Service = địa chỉ ổn định + load balancing.** Bạn tạo Service `b` với selector trỏ vào label của các Pod B. Kubernetes cấp cho nó một ClusterIP cố định và tự duy trì danh sách IP của 2–3 Pod B đang **Ready** trong EndpointSlice. Khi Pod A mở kết nối tới ClusterIP đó, kube-proxy (iptables/IPVS trên node) DNAT kết nối tới **một trong các Pod B** — mỗi connection được phân phối sang một backend, đó là load balancing ở tầng **L4** (tầng kết nối TCP/UDP: hệ thống chỉ nhìn thấy IP và port, không đọc được nội dung HTTP bên trong).
3. **CoreDNS.** Mỗi Service có tên DNS — DNS nội bộ đã verify ở [§8.4](#84-tầng-4--dns). Code trong Pod A chỉ cần biết tên:

| Cách gọi                                   | Khi nào dùng                                                         |
| -------------------------------------------- | ---------------------------------------------------------------------- |
| `http://b`                                 | A và B cùng namespace                                                |
| `http://b.other-ns`                        | B ở namespace khác                                                   |
| `http://b.other-ns.svc.cluster.local:8080` | Dạng đầy đủ, tường minh nhất; nên dùng trong manifest/config |

Luồng đầy đủ khi A gọi B:

```text
Pod A ──► http://b:80
      → CoreDNS: "b" → ClusterIP của Service b (vd 10.96.34.12)
      → kube-proxy DNAT → chọn một Pod B: 10.244.1.5 / 10.244.2.7 / 10.244.2.9
      → Pod B xử lý và trả response trên đúng kết nối đó
```

Các điểm hay nhầm:

- **Chỉ bên nhận cần Service.** A gọi ra ngoài bằng chính network của Pod; A chỉ cần Service khi có bên khác muốn gọi *vào* A.
- **Load balancing theo kết nối, không theo request.** Nếu A giữ kết nối keep-alive hoặc gRPC lâu dài tới B, mọi request trên kết nối đó dồn vào đúng một Pod B. Muốn cân bằng theo từng request phải xử lý ở **L7** — tầng nội dung HTTP, nơi phần mềm đọc được hostname, path, header: client-side load balancing, service mesh, hoặc một **reverse proxy** (máy chủ trung gian đứng về phía server, nhận request thay cho backend rồi chuyển tiếp) như Traefik — đúng vai trò Traefik sẽ đảm nhận ở [§9.1.4](#914-service-khác-ingress-như-thế-nào).
- **Resolve được chưa chắc gọi được.** Service sai selector hoặc Pod chưa Ready thì DNS vẫn trả ClusterIP nhưng kết nối thất bại; kiểm tra bằng lệnh EndpointSlice ở [§9.1.1](#911-từ-pod-đến-một-địa-chỉ-ổn-định-service-và-endpointslice).
- **Headless Service** (`clusterIP: None`) dành cho trường hợp cần gọi *từng Pod cụ thể* (database primary/replica, Kafka…): DNS trả thẳng IP từng Pod thay vì một địa chỉ chung, thường đi cùng StatefulSet. Runbook không cần nó cho luồng chính.

Trong runbook, chính cơ chế này xuất hiện ở [§12](#12-tạo-cloudflare-tunnel-chạy-trong-cụm): Pod `cloudflared` gọi `traefik.traefik.svc.cluster.local:80` — một lời gọi service-to-service qua DNS + ClusterIP, không khác gì A gọi B.

#### 9.1.3. `ClusterIP`, `NodePort` và `LoadBalancer`

Đây là **các kiểu Service ở tầng mạng L4**, không phải các loại Ingress:

| Kiểu Service    | Có thể truy cập từ đâu                                                         | Cơ chế và mục đích phù hợp                                                                                                                                                                                                                                                                        |
| ---------------- | ------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ClusterIP`    | Trong cluster                                                                        | Mặc định. Cấp IP/DNS nội bộ ổn định; phù hợp cho giao tiếp service-to-service và cho`cloudflared` gọi Traefik trong runbook này.                                                                                                                                                         |
| `NodePort`     | Qua`<NodeIP>:<nodePort>` trên mỗi node, ngoài việc vẫn có ClusterIP          | Phù hợp test từ LAN hoặc làm tầng dưới cho một LB bên ngoài. Không tự cung cấp hostname routing hay TLS; ứng dụng/controller vẫn có thể tự phục vụ TLS. Dải mặc định là`30000–32767`.                                                                                       |
| `LoadBalancer` | Qua địa chỉ do một implementation LB cấp; có thể là public hoặc private/LAN | Kubernetes chỉ khai báo nhu cầu. Cloud provider, MetalLB hoặc implementation khác phải cấp/quảng bá địa chỉ. Thông thường nó xây trên NodePort, nhưng có thể tắt cấp NodePort bằng`allocateLoadBalancerNodePorts: false` nếu implementation hỗ trợ đường đi trực tiếp. |

`EXTERNAL-IP: <pending>` không có nghĩa CNI hỏng. Nó thường có nghĩa Service `LoadBalancer` chưa được implementation nào xử lý. Trên VMware/bare-metal, **MetalLB** có thể cấp IP từ pool LAN và quảng bá IP đó bằng L2/BGP; MetalLB là một implementation cho Service `LoadBalancer`, **không phải cloud-controller-manager**.

Từ “load balancing” xuất hiện ở nhiều tầng nhưng không cùng một đối tượng: Service/kube-proxy phân phối kết nối tới endpoint; Traefik chọn backend theo rule HTTP rồi cân bằng request; Service `type: LoadBalancer` yêu cầu thêm một điểm vào bên ngoài cluster. Đừng suy ra rằng cứ có cân bằng tải nội bộ thì Service phải mang type `LoadBalancer`.

#### 9.1.4. Service khác Ingress như thế nào?

Đến đây bạn đã đưa được traffic tới một nhóm Pod qua Service. Nhưng khi triển khai thật, cluster chạy **nhiều ứng dụng cùng lúc** — web app, API, rồi cả UI Rancher — mỗi ứng dụng một Service riêng. Bài toán mới xuất hiện: muốn **một điểm vào duy nhất** cho tất cả, và phân request theo **tên miền / đường dẫn**: `app.hieupn.site` vào Service `web`, `rancher.hieupn.site` vào Service của Rancher. Service không tự làm được việc này vì nó hoạt động ở L4 — chỉ thấy IP và port, không đọc được hostname hay path nằm bên trong request HTTP.

Lời giải kinh điển (có từ trước cả Kubernetes) là đặt một **reverse proxy** ở cửa: một server nhận *mọi* request HTTP thay cho các ứng dụng, mở request ra đọc hostname/path, rồi chuyển tiếp tới đúng ứng dụng. Gọi là *reverse* vì nó đứng về phía server, đại diện cho backend — ngược với proxy phía client:

```text
                          ┌─ Host: app.hieupn.site     → Service web     → Pod web
mọi request ──► Traefik ──┼─ Host: api.example.com     → Service api     → Pod api
   (một reverse proxy)    └─ Host: rancher.hieupn.site → Service rancher → Pod rancher
```

Nói gọn: Service trả lời câu hỏi **“làm sao tới một nhóm Pod ổn định?”**, Ingress trả lời câu hỏi **“request HTTP có host/path này phải tới Service nào?”**. Kubernetes chia lời giải reverse proxy thành các mảnh có tên riêng — đọc bảng theo đúng thứ tự này:

| Thành phần            | Vai trò khi triển khai                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Ingress`             | **Bản khai báo luật định tuyến của một ứng dụng**, viết bằng YAML: “request có hostname `app.example.com`, path `/` → giao cho Service `web` port 80”. Nó chỉ là *dữ liệu* lưu trong cluster — tự nó không nhận và không xử lý request nào. Lợi ích khi vận hành: mỗi app tự mang luật của mình; thêm app mới = thêm một Ingress, không phải sửa file cấu hình tập trung nào.                                                                                      |
| Ingress Controller      | **Phần mềm thật sự làm việc**: chạy như Pod trong cluster, liên tục theo dõi các object Ingress/Service/EndpointSlice qua Kubernetes API, rồi tự cấu hình reverse proxy bên trong nó theo các luật đọc được. Không cài controller thì mọi Ingress chỉ nằm im vô hại — đây là lý do mục 9 tồn tại. Runbook dùng **Traefik**.                                                                                                                                                    |
| `IngressClass`        | **Tấm biển tên gắn cho mỗi controller.** Một cluster có thể chạy nhiều controller cùng lúc (Traefik cho app nội bộ, một controller khác cho app public…), nên mỗi Ingress phải chỉ rõ nó thuộc về ai bằng trường `ingressClassName`. Runbook tạo class tên `traefik`; chi tiết và khái niệm *default class* ở [§9.1.7](#917-provider-của-traefik-và-default-ingressclass).                                                                                                          |
| Service của controller | **Đường để client tới được chính Traefik.** Traefik cũng chỉ là Pod chạy trong cluster — quy tắc [§9.1.1](#911-từ-pod-đến-một-địa-chỉ-ổn-định-service-và-endpointslice) áp dụng cho chính nó — nên nó cũng cần một Service đứng trước. Kiểu `ClusterIP`/`NodePort`/`LoadBalancer` ([§9.1.3](#913-clusterip-nodeport-và-loadbalancer)) của Service này quyết định *ai gọi được vào proxy*; lựa chọn đó độc lập với các Service của ứng dụng phía sau. |

Ingress chuẩn chỉ định tuyến **HTTP/HTTPS (L7)** theo hostname/path; nó không expose TCP/UDP bất kỳ. Kubernetes vẫn duy trì Ingress API ổn định nhưng đã **freeze** API này và khuyến nghị Gateway API cho phát triển mới. Runbook vẫn dùng Ingress vì đơn giản, phổ biến và đủ cho lab; muốn dùng Gateway API với Traefik phải cài Gateway API CRDs và bật provider riêng — mục 9 không làm việc đó.

#### 9.1.5. Đọc một object `Ingress`: rules, host, path và pathType

Đây là dạng manifest sẽ gõ ở mục 10 — đọc kỹ từng trường một lần ở đây để lúc đó chỉ việc gõ. Mỗi trường được chú thích ngay tại chỗ:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: traefik   # controller chịu trách nhiệm — IngressClass, chi tiết ở §9.1.7
  rules:                      # danh sách rule L7; mỗi rule = 1 host + nhiều path
    - host: app.example.com   # so với Host header của request
      http:
        paths:                # cùng 1 host có thể tách nhiều path về nhiều Service
          - path: /
            pathType: Prefix  # cách hiểu trường path — bảng ngay dưới
            backend:
              service:
                name: web     # Service đích (§9.1.1); phải cùng namespace với Ingress
                port:
                  number: 80  # port của Service; hoặc dùng name: <tên-port> thay số
```

`pathType` quyết định cách so trường `path` với URL của request — trường bắt buộc và hay bị chép máy móc mà không hiểu:

| `pathType`               | Cách khớp                                          | Ví dụ với`path: /api`                                                |
| -------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------- |
| `Prefix`                 | Theo**segment** của URL, tách bằng `/`    | Khớp`/api`, `/api/`, `/api/v1`; **không** khớp `/apixyz` |
| `Exact`                  | Đúng tuyệt đối, phân biệt cả dấu`/` cuối | Khớp`/api`; không khớp `/api/`                                     |
| `ImplementationSpecific` | Tùy controller tự định nghĩa                    | Tránh dùng khi cần manifest portable                                   |

Quy tắc khớp một request:

- **`host` so với cái gì?** Mỗi request HTTP luôn mang theo tên miền đang truy cập trong header `Host` — trình duyệt và `curl` tự thêm. Nhờ header này, một điểm vào duy nhất phục vụ được nhiều domain: controller chỉ việc đọc `Host` rồi chọn rule tương ứng. Các lệnh test ở mục 10 và 13 giả lập chính nó bằng `curl -H "Host: ..."`.
- Controller so **host trước, path sau**. Rule không khai `host` thì khớp mọi hostname (catch-all).
- Host wildcard `*.example.com` chỉ khớp **đúng một label**: `a.example.com` khớp, `a.b.example.com` và `example.com` không.
- Nhiều path cùng khớp một request → **path dài nhất thắng**; nếu vẫn hòa, `Exact` được ưu tiên hơn `Prefix`.
- Hai object `Ingress` cùng host nhưng khác path → controller **gộp chung** thành một bảng route; nhờ vậy nhiều app/team chia nhau được một domain, mỗi app giữ Ingress riêng.
- Request không khớp rule nào → rơi vào `defaultBackend` nếu có khai; runbook không khai, nên Traefik trả **404**. Đây chính là lý do smoke test ở [§9.3.1](#931-verify--tất-cả-phải-pass-trước-mục-10) coi 404 là PASS: nó chứng minh Traefik sống và chỉ đơn giản chưa có rule nào — không phải lỗi.

Còn `tls:`? Ingress chuẩn có block `tls: [{hosts, secretName}]` để controller terminate HTTPS bằng cert lấy từ Secret. Runbook **cố ý không dùng**: TLS terminate ở Cloudflare edge, đoạn đường trong cluster đi HTTP ([§13](#13-trỏ-domain--kiểm-tra-trên-internet)). Gặp tutorial bên ngoài có `tls:` thì hiểu đó là kiến trúc không có tunnel đứng trước.

Tiểu mục tiếp theo trả lời: Traefik đọc object này và biến nó thành cấu hình proxy như thế nào.

#### 9.1.6. Traefik sẽ làm gì trong cluster này?

Traefik vừa theo dõi Kubernetes API vừa làm reverse proxy. Một request đi qua các khái niệm sau:

```text
Internet → Cloudflare edge → tunnel outbound → Pod cloudflared
        → traefik.traefik.svc.cluster.local:80 (ClusterIP)
        → EntryPoint web (:80)
        → Router khớp Host/Path từ object Ingress
        → Middleware tùy chọn
        → Traefik Service nội bộ → Pod IP lấy từ EndpointSlice
```

- **EntryPoint**: cổng Traefik lắng nghe, trong chart là `web` và `websecure`.
- **Router**: rule chọn request theo host/path.
- **Middleware**: bước xử lý tùy chọn như redirect, auth, rate limit hoặc sửa path.
- **Traefik Service**: khái niệm nội bộ của Traefik chứa các backend; đừng nhầm với object Kubernetes `Service`.
- **Provider**: nguồn cấu hình mà Traefik theo dõi. `kubernetesIngress` đọc Ingress chuẩn; `kubernetesCRD` đọc CRD riêng như `IngressRoute` và `Middleware`. Chi tiết ở [§9.1.7](#917-provider-của-traefik-và-default-ingressclass).

Với `nativeLBByDefault: false` của chart, Traefik thường resolve EndpointSlice rồi proxy trực tiếp tới Pod IP thay vì coi ClusterIP của Service backend là một đích duy nhất. Ta dùng `Ingress` chuẩn trong [§10](#10-deploy-app-mẫu--ingress) để kiến thức dễ chuyển sang controller khác; chỉ dùng CRD riêng khi thật sự cần tính năng đặc thù của Traefik.

#### 9.1.7. Provider của Traefik và default IngressClass

Hai nhóm flag quan trọng của lệnh cài ở [§9.3](#93-cài-đặt-traefik) xoay quanh hai khái niệm này.

**Provider là khái niệm của Traefik, không phải của Kubernetes.** Traefik là reverse proxy **động**: không có file config tĩnh liệt kê sẵn route. Nó theo dõi các nguồn cấu hình — mỗi nguồn là một *provider* — rồi tự dựng bảng định tuyến từ những gì đọc được và cập nhật nóng khi nguồn thay đổi, không cần restart. Chạy trong Kubernetes, "bật một provider" nghĩa là: Traefik mở watch tới Kubernetes API server và theo dõi một nhóm resource nhất định (RBAC của chart cấp quyền đọc tương ứng). Hai provider trong lệnh cài khác nhau ở *loại resource được theo dõi*:

| Provider              | Theo dõi resource                                                                            | Đặc điểm                                                                                                                                               |
| --------------------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `kubernetesIngress` | `Ingress` chuẩn (`networking.k8s.io/v1`)                                                 | API chính thức của Kubernetes, controller nào cũng hiểu → manifest**portable** sang controller khác. Chỉ định tuyến HTTP theo host/path. |
| `kubernetesCRD`     | CRD riêng của Traefik:`IngressRoute`, `Middleware`, `TraefikService`, `TLSOption`… | Chỉ Traefik hiểu →**không portable**; đổi lại rule match phức tạp hơn, có middleware, route được cả TCP/UDP.                          |

**CRD (CustomResourceDefinition)** là cơ chế "dạy" API server một kiểu resource mới ngoài bộ chuẩn (Pod, Service…). Chart Traefik cài các CRD này ở lần install đầu; sau đó `kubectl get ingressroute -A` chạy được như với resource chuẩn và object được lưu trong etcd như mọi object khác. Bản thân CRD chỉ là *định nghĩa kiểu dữ liệu* — phải có phần mềm đọc nó (ở đây là Traefik với provider `kubernetesCRD` bật) thì object mới có tác dụng.

Runbook bật **cả hai** provider vì: ứng dụng ở [§10](#10-deploy-app-mẫu--ingress) dùng `Ingress` chuẩn, còn route dashboard mà chart tạo là một `IngressRoute` (CRD). Thiếu provider nào thì YAML tương ứng thành object "nằm im trong etcd, không ai đọc". Cả hai provider cùng đổ cấu hình vào một bảng routing nội bộ duy nhất của Traefik.

**Default IngressClass giải bài toán "Ingress này của ai?".** Một cluster có thể chạy nhiều ingress controller cùng lúc; khi tạo một `Ingress` phải có cách chỉ định controller chịu trách nhiệm — nếu không, hoặc không ai xử lý, hoặc hai controller tranh nhau xử lý. `IngressClass` là object cấp cluster làm "danh thiếp" cho mỗi controller:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: traefik                                          # tên mà Ingress tham chiếu
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"  # phần "default"
spec:
  controller: traefik.io/ingress-controller              # định danh phần mềm xử lý
```

Mỗi `Ingress` khai `spec.ingressClassName: traefik`; controller thấy class không phải của mình thì bỏ qua. **Default** nghĩa là: khi một Ingress *không khai* `ingressClassName`, admission controller của Kubernetes tự điền tên class default vào object ngay lúc tạo. Đây là lưới an toàn cho manifest thiếu sót — và là lý do chỉ nên có **một** class default: từ hai default trở lên, Kubernetes từ chối tạo Ingress không khai class vì không biết chọn ai. Đây là cùng một pattern với StorageClass default đã gặp ở [§8.7](#87-tầng-7--công-cụ-và-add-on-kubeadm-không-cài-sẵn): PVC không khai `storageClassName` thì dùng class có `(default)`.

Luồng đầy đủ khi hai khái niệm gặp nhau:

```text
Bạn tạo Ingress (giả sử quên khai class)
  → admission controller điền ingressClassName: traefik (vì class traefik là default)
Traefik (provider kubernetesIngress đang watch Ingress)
  → thấy ingressClassName trỏ tới IngressClass có controller traefik.io/ingress-controller
  → nhận trách nhiệm → sinh Router → bắt đầu nhận traffic cho host/path đó
Controller khác (nếu có) → class không phải của mình → bỏ qua
```

Runbook vẫn khai tường minh `ingressClassName: traefik` trong mọi Ingress dù đã có default — khai rõ tốt hơn dựa vào lưới an toàn; default chỉ để đỡ các manifest quên khai.

#### 9.1.8. Các khái niệm Helm xuất hiện trong lệnh cài

| Khái niệm             | Nghĩa trong mục 9                                                                                                                                                           |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Helm chart              | Gói template Kubernetes của Traefik                                                                                                                                         |
| Helm release`traefik` | Một lần cài chart có tên`traefik`; `upgrade --install` tạo mới nếu chưa có và cập nhật nếu đã có                                                         |
| Namespace`traefik`    | Phạm vi chứa Deployment, Pod, Service và phần lớn tài nguyên của release;`IngressClass` và CRD là tài nguyên cấp cluster                                       |
| Values /`--set`       | Giá trị làm thay đổi template. Runbook khai tường minh các quyết định quan trọng thay vì phụ thuộc hoàn toàn vào default của phiên bản chart tương lai |
| `--version 41.0.2`    | Ghim chart để lần chạy sau không tự lấy chart mới có hành vi khác                                                                                                  |

**Kết quả mong đợi sau mục 9:** có một Deployment Traefik Ready, một IngressClass `traefik`, một Service Traefik kiểu `ClusterIP` với cổng 80/443, và dashboard chỉ truy cập cục bộ qua `kubectl port-forward`. Mục 10 mới tạo ứng dụng và rule Ingress để kiểm tra đường đi end-to-end.

### 9.2. Vì sao chọn Traefik thay cho `ingress-nginx`

> ⚠️ `kubernetes/ingress-nginx` đã **retired và archive trong tháng 03/2026**; bản cuối là `v1.15.1` và dự án không còn release, bugfix hay bản vá bảo mật. Đây là lý do đủ để không dùng nó cho một cài đặt mới. Không nhầm dự án này với web server nginx hoặc F5 NGINX Ingress Controller — đó là các sản phẩm/dự án khác. Khi đánh giá CVE, luôn đối chiếu đúng sản phẩm và phiên bản; không tự động gán advisory của nginx upstream cho mọi ingress controller có chữ “nginx”.

Traefik phù hợp ở đây vì:

- Traefik vẫn được duy trì và chart **41.0.2** phát hành ngày 06/07/2026, dùng Traefik Proxy `v3.7.6`; chart yêu cầu Kubernetes `>=1.25`, phù hợp Kubernetes `1.35.6` của runbook.
- SUSE/Rancher có [hướng dẫn migration từ ingress-nginx sang Traefik](https://ranchermanager.docs.rancher.com/v2.14/how-to-guides/new-user-guides/kubernetes-resources-setup/load-balancer-and-ingress-controller/guide-to-ingress-nginx-retirement); đích tiếp theo của runbook là Rancher.
- Hỗ trợ cả Ingress chuẩn và CRD riêng. Traefik cũng hỗ trợ Gateway API, nhưng chart không cài Gateway API CRDs và runbook chưa bật provider đó.
- Một replica là đủ cho lab hiện tại; production cần đánh giá thêm replica, PodDisruptionBudget, anti-affinity, tài nguyên và đường vào HA.

Nguồn kiểm chứng: [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/), [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), [thông báo retirement ingress-nginx](https://v1-35.docs.kubernetes.io/blog/2026/01/29/ingress-nginx-statement/), [Traefik chart 41.0.2](https://github.com/traefik/traefik-helm-chart/releases/tag/v41.0.2).

### 9.3. Cài đặt Traefik

Chạy trên master bằng user `ubuntu` đã có kubeconfig. Helm đã được chuẩn bị ở [§8.7](#87-tầng-7--công-cụ-và-add-on-kubeadm-không-cài-sẵn). Ghim chart và các lựa chọn kiến trúc quan trọng để lần sau không bị thay đổi theo default mới:

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm upgrade --install traefik traefik/traefik \
  --namespace traefik --create-namespace \
  --version 41.0.2 \
  --set providers.kubernetesIngress.enabled=true \
  --set providers.kubernetesCRD.enabled=true \
  --set ingressClass.enabled=true \
  --set ingressClass.name=traefik \
  --set ingressClass.isDefaultClass=true \
  --set ingressRoute.dashboard.enabled=true \
  --set service.spec.type=ClusterIP \
  --wait --timeout 5m
```

Các giá trị chính:

| Giá trị                                    | Ý nghĩa                                                                                                                           |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `providers.kubernetesIngress.enabled=true` | Đọc object Ingress chuẩn mà mục 10 sẽ tạo                                                                                    |
| `providers.kubernetesCRD.enabled=true`     | Cài/đọc CRD Traefik; dashboard chart dùng`IngressRoute` nội bộ                                                              |
| `ingressClass.*`                           | Tạo class`traefik` và đặt làm default duy nhất của cluster ([§9.1.7](#917-provider-của-traefik-và-default-ingressclass)) |
| `service.spec.type=ClusterIP`              | Chỉ expose Traefik trong cluster; đúng với cloudflared in-cluster và không sinh NodePort/`EXTERNAL-IP: <pending>`           |
| `ports.web` / `ports.websecure`          | Container lắng nghe 8000/8443; Service cung cấp cổng 80/443                                                                      |
| `ingressRoute.dashboard.enabled=true`      | Bật route dashboard trên entrypoint admin nội bộ, không công khai nó qua Service 80/443                                      |
| `--wait --timeout 5m`                      | Helm chỉ trả thành công sau khi tài nguyên sẵn sàng hoặc báo timeout                                                      |

> ⚠️ Với chart 41.0.2 phải dùng `service.spec.type`. Key cũ `service.type` vẫn có thể được Helm/schema
> chấp nhận nhưng chart bỏ qua; Service khi đó giữ mặc định `LoadBalancer` và `EXTERNAL-IP` có thể ở
> `<pending>`.

#### 9.3.1. Verify — tất cả phải PASS trước mục 10

```bash
helm status traefik -n traefik
kubectl -n traefik rollout status deploy/traefik --timeout=180s
kubectl -n traefik get deploy,pod,svc,endpointslice -o wide
kubectl get ingressclass -o custom-columns='NAME:.metadata.name,CONTROLLER:.spec.controller,DEFAULT:.metadata.annotations.ingressclass\.kubernetes\.io/is-default-class'
kubectl -n traefik logs deploy/traefik --tail=50
```

PASS khi:

- Helm báo `STATUS: deployed`; Deployment rollout thành công, Pod `Running` và `READY 1/1`.
- Service `traefik` có `TYPE=ClusterIP`, một `CLUSTER-IP`, cổng `80/TCP,443/TCP`; không cần cột `EXTERNAL-IP` có giá trị.
- EndpointSlice của Service Traefik có endpoint Ready trỏ tới Pod Traefik.
- `IngressClass` có tên `traefik`, controller `traefik.io/ingress-controller` và cột `DEFAULT=true`. Cluster chỉ nên có **một** default IngressClass.
- Log không có lỗi lặp lại; warning đơn lẻ phải được đọc theo ngữ cảnh trước khi kết luận fail.

Nếu Service không có endpoint, kiểm tra selector/Pod:

```bash
kubectl -n traefik describe svc traefik
kubectl -n traefik get endpointslice \
  -l kubernetes.io/service-name=traefik -o yaml
kubectl -n traefik describe pod \
  -l app.kubernetes.io/name=traefik
```

Smoke test HTTP trước khi tạo app/Ingress ở mục 10; bước này tách lỗi Traefik khỏi lỗi manifest ứng
dụng. Dùng `trap` để tiến trình `port-forward` vẫn được dọn nếu `curl` bị lỗi:

```bash
kubectl -n traefik port-forward svc/traefik 8081:80 \
  >/tmp/traefik-port-forward.log 2>&1 &
PF_PID=$!
trap 'kill "$PF_PID" 2>/dev/null' EXIT

HTTP_CODE=$(curl -sS --retry 10 --retry-connrefused --retry-delay 1 \
  -o /dev/null -w '%{http_code}' http://127.0.0.1:8081/)
echo "$HTTP_CODE"                 # PASS: 404 — Traefik sống, chưa có Router ứng dụng

kill "$PF_PID"
trap - EXIT
```

Ở thời điểm này chưa có Ingress ứng dụng, nên HTTP `404` là kết quả mong đợi. Mã `000`, lỗi kết nối
hoặc timeout mới là FAIL; xem `/tmp/traefik-port-forward.log` và kiểm tra lại Pod, Service,
EndpointSlice.

#### 9.3.2. Dashboard nội bộ (tuỳ chọn)

```bash
TRAEFIK_POD=$(kubectl -n traefik get pod \
  -l app.kubernetes.io/name=traefik \
  -o jsonpath='{.items[0].metadata.name}')
kubectl -n traefik port-forward "pod/$TRAEFIK_POD" 8080:8080
# Mở http://127.0.0.1:8080/dashboard/ — bắt buộc có dấu / cuối.
```

Lệnh `port-forward` giữ terminal; nhấn `Ctrl+C` để dừng. Dashboard không có mục đích public trong runbook này. Không tạo Ingress Internet cho dashboard và không dùng `api.insecure=true`.

### 9.4. Khi nào dùng `ClusterIP`, `NodePort`, `LoadBalancer` hoặc MetalLB cho Traefik?

Runbook chọn `ClusterIP` vì đường vào Internet là tunnel outbound:

```text
Internet → Cloudflare edge → Pod cloudflared
        → http://traefik.traefik.svc.cluster.local:80
        → Traefik → Service/Pod ứng dụng
```

| Nhu cầu                                                 | Kiểu Service Traefik      | Xử lý                                                                                                                 |
| -------------------------------------------------------- | -------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Cloudflare Tunnel chạy trong cluster như runbook       | `ClusterIP`              | Giữ nguyên; không mở cổng LAN và không cần external IP                                                          |
| Test nhanh từ máy trong LAN bằng IP node và port cao | `NodePort`               | Đổi type; firewall phải cho phép nodePort cần dùng                                                                |
| Một IP LAN cố định trên VMware/bare-metal           | `LoadBalancer` + MetalLB | Cài/cấu hình MetalLB, dành pool IP ngoài DHCP, rồi đổi Traefik sang`LoadBalancer`                             |
| Kubernetes trên cloud có integration LB                | `LoadBalancer`           | Controller/provider của cloud cấp địa chỉ và hạ tầng LB; địa chỉ public hay private tùy cấu hình provider |

Đổi sang NodePort khi thật sự cần truy cập trực tiếp từ LAN:

```bash
helm upgrade traefik traefik/traefik \
  -n traefik --version 41.0.2 --reuse-values \
  --set service.spec.type=NodePort \
  --wait --timeout 5m
kubectl -n traefik get svc traefik
```

> Lệnh trên giữ nguyên `--version 41.0.2`, nên không thay đổi CRD. Khi bump chart về sau, đọc release
> notes và apply CRD của đúng phiên bản đích **trước** `helm upgrade`, ví dụ:
> `helm show crds traefik/traefik --version <phiên-bản-đích> | kubectl apply --server-side --force-conflicts -f -`.
> Helm không tự nâng cấp các CRD nằm trong thư mục `crds/`.

Muốn IP LAN cố định, làm theo [Phụ lục B](#phụ-lục-b--metallb-tuỳ-chọn-loadbalancer-ip-trong-lan). Nếu tạo Service `LoadBalancer` trước khi có implementation phù hợp, `EXTERNAL-IP` có thể ở `<pending>`; đó là yêu cầu chưa được cấp địa chỉ, không tự động chứng minh CNI hay Traefik lỗi.

---

## 10. Deploy app mẫu + Ingress

> Toàn bộ [§10] chạy **trên master** (nơi có `kubectl`). File `.yaml` tạo ở thư mục home của user (vd `~/demo-app.yaml`).

Tạo file `demo-app.yaml` và **giữ nguyên hostname tạm `app.example.com` trong §10** vì domain thật chưa được mua/cấu hình cho tới [§11](#section-11). Hostname tạm đủ để test định tuyến nội bộ bằng Host header; [§12.3](#123-chốt-hostname-thật-trong-ingress--thêm-published-application) sẽ có bước bắt buộc đổi sang hostname thật trước khi tạo route Cloudflare. Ý nghĩa từng trường của phần `Ingress` đã giải thích ở [§9.1.5](#915-đọc-một-object-ingress-rules-host-path-và-pathtype):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: web
          image: nginxdemos/hello:plain-text
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: { app: web }
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: traefik      # Traefik không redirect HTTP→HTTPS mặc định
  rules:                         # → không cần annotation ssl-redirect như ingress-nginx
    - host: app.example.com          # hostname tạm cho §10; đổi sang domain thật ở §12.3.1
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

### 10.1. Đọc manifest: Deployment → Service → Ingress

File trên chứa ba document YAML, ngăn cách bằng `---`. Chúng tạo thành một chuỗi nhưng đảm nhiệm ba
vai trò khác nhau:

```text
Deployment web → tạo 2 Pod có label app=web
Service web:80 → chọn các Pod app=web và chuyển tới Pod port 80
Ingress web    → với Host app.example.com, chuyển request tới Service web:80
```

**Deployment `web`:** `replicas: 2` yêu cầu Kubernetes duy trì hai Pod ứng dụng. Hai chỗ
`matchLabels: { app: web }` và `labels: { app: web }` phải khớp nhau để Deployment nhận đúng Pod do
nó tạo. Container chạy image `nginxdemos/hello:plain-text` và lắng nghe cổng 80. Khai báo
`containerPort: 80` chỉ mô tả cổng của container; nó không tự expose Pod và không tạo địa chỉ ổn định.

**Service `web`:** selector `{ app: web }` tìm đúng hai Pod trên. `port: 80` là cổng ổn định của
Service, còn `targetPort: 80` là cổng nhận request trên Pod:

```text
Service web:80 → một trong các Pod app=web:80
```

Manifest không khai `spec.type`, nên Kubernetes dùng mặc định `ClusterIP`. Pod có thể bị tạo lại và
đổi IP nhưng Traefik vẫn gọi tên Service `web` ổn định; EndpointSlice của Service sẽ được Kubernetes
cập nhật theo các Pod Ready hiện tại.

**Ingress `web`:** `ingressClassName: traefik` giao object này cho controller đã cài ở §9. Rule chỉ
khớp request có Host header `app.example.com`; `path: /` với `pathType: Prefix` nhận mọi đường dẫn bắt
đầu bằng `/`, rồi gửi tới backend `Service web:80`. Manifest không cấu hình TLS hay redirect
HTTP→HTTPS, nên test nội bộ bên dưới dùng HTTP.

Không resource nào trong file khai `metadata.namespace`, vì vậy Deployment, Service và Ingress
`web` đều được tạo trong namespace `default`. Backend của một Ingress chuẩn phải là Service cùng
namespace, nên `Ingress web` tham chiếu đúng `Service web` trong `default`.

Cần phân biệt hai Service mà các lệnh bên dưới sử dụng:

| Service     | Namespace   | Vai trò                                                              |
| ----------- | ----------- | --------------------------------------------------------------------- |
| `traefik` | `traefik` | Điểm vào của ingress controller; nhận request test trước tiên |
| `web`     | `default` | Backend ứng dụng; được Ingress`web` chuyển request tới       |

### 10.2. Apply manifest và kiểm tra rule Ingress

`kubectl apply` đọc cả ba document trong file và tạo mới hoặc cập nhật chúng theo kiểu idempotent.
Output `created`, `configured` hoặc `unchanged` đều cho biết API server đã chấp nhận resource:

```bash
kubectl apply -f demo-app.yaml
```

Sau đó kiểm tra object Ingress mà Traefik sẽ theo dõi:

```bash
kubectl get ingress
```

PASS khi thấy `NAME=web`, `CLASS=traefik`, `HOSTS=app.example.com` và `PORTS=80`.

`app.example.com` ở gate này chỉ là hostname tạm phục vụ test nội bộ, **không phải domain public cuối cùng**. Không đổi hostname tại §10; sau khi domain thật PASS §11, thực hiện bước chuyển hostname có verify riêng tại §12.3.1.

> Chart bật sẵn `providers.kubernetesIngress.publishedService.enabled=true`. Với Service Traefik kiểu
> `ClusterIP` và không khai `spec.externalIPs`, cột `ADDRESS` của `kubectl get ingress` để trống là
> **PASS**, không phải lỗi. Tiếp tục kiểm tra `CLASS=traefik`, hostname và test HTTP bên dưới.

### 10.3. Test nội bộ end-to-end qua ClusterIP Traefik

Test này chưa cần DNS hay domain thật. Lệnh đầu lấy ClusterIP của **Service `traefik` trong namespace
`traefik`** và lưu vào biến shell `ING_IP`. Phép gán biến thành công sẽ không in output:

```bash
# lấy ClusterIP của ingress controller:
ING_IP=$(kubectl get svc -n traefik traefik -o jsonpath='{.spec.clusterIP}')
```

Lệnh tiếp theo gửi request tới cổng HTTP 80 của Traefik. `-H "Host: app.example.com"` giả lập Host
header mà trình duyệt sẽ gửi khi DNS thật đã trỏ vào hệ thống; Traefik dùng giá trị này để khớp rule
trong `Ingress web`. `-s` chỉ ẩn progress meter của `curl`:

```bash
curl -s -H "Host: app.example.com" http://$ING_IP/
# → phải thấy "Server address..." từ nginx hello
```

Đường đi đầy đủ của request:

```text
curl trên master
  → Service traefik:80 (namespace traefik)
  → Pod Traefik:8000
  → khớp Ingress web vì Host=app.example.com
  → Service web:80 (namespace default)
  → một trong hai Pod web:80
```

Output `Server address` và `Server name` từ nginx demo chứng minh toàn bộ chuỗi đã chạy, không chỉ
riêng Pod hay Service. Chạy lại `curl` có thể thấy Pod khác vì Deployment có hai replica. Nếu bỏ Host
header hoặc dùng hostname không khớp, Traefik không tìm thấy Router phù hợp và thường trả `404`.

> Nếu lệnh `curl` đúng như trên vẫn trả `404`, kiểm tra Host header có khớp chính xác
> `app.example.com` và `kubectl get ingress` có `CLASS=traefik`; sau đó chạy
> `kubectl describe ingress web` hoặc mở dashboard ([§9.3](#93-cài-đặt-traefik)) xem tab
> **HTTP → Routers**. Chủ động bỏ Host header hoặc dùng hostname khác thì `404` lại là kết quả mong đợi
> vì không có rule nào khớp.

Thấy output nghĩa là chuỗi **ingress → service → pod** đã chạy. Giờ chỉ còn nối Internet vào.

---

<a id="section-11"></a>

## 11. Mua & cấu hình domain trên Cloudflare

### 11.1. Mua domain

Mua ở bất kỳ registrar nào (Cloudflare Registrar, Namecheap, GoDaddy, PA Vietnam…). Domain rẻ (`.com`, `.dev`, `.xyz`) đều được.

### 11.2. Thêm domain vào Cloudflare

Mục này dùng **Primary DNS / Full Setup**: Hostinger vẫn là **registrar** (giữ đăng ký, gia hạn và thông tin chủ thể), còn Cloudflare trở thành **authoritative DNS provider** (quản lý DNS, proxy, HTTPS và Tunnel). Đây là cấu hình đủ cho toàn bộ runbook; **không cần chuyển registrar sang Cloudflare**.

> **Phân biệt hai lựa chọn trên màn hình “Add a site”:**
>
> | Lựa chọn                  | Thay đổi                                                                                | Dùng trong runbook?                      |
> | --------------------------- | ----------------------------------------------------------------------------------------- | ----------------------------------------- |
> | **Connect a domain**  | Chuyển DNS authoritative sang Cloudflare; registrar và việc gia hạn vẫn ở Hostinger | **Có — chọn mục này**          |
> | **Transfer a domain** | Chuyển cả đăng ký/gia hạn domain từ Hostinger sang Cloudflare Registrar            | Không cần; chỉ là tùy chọn sau này |
>
> Cloudflare yêu cầu domain phải **Active trên Cloudflare trước khi transfer** và domain mới đăng ký/transfer thường bị khóa transfer **60 ngày**. Vì vậy ngay cả khi sau này muốn chuyển registrar, vẫn phải hoàn tất luồng **Connect a domain** bên dưới trước. Không tắt *domain/transfer lock* và không lấy mã EPP trong mục này.

#### 11.2.1. Gate trước thay đổi — kiểm tra domain, DNSSEC và bản ghi hiện có tại Hostinger

Trong **Hostinger hPanel → Domains → Domain portfolio → Manage → DNS / Nameservers**:

1. Xác nhận domain có trạng thái **Hoạt động/Active** và chưa hết hạn.
2. Giữ **Khóa tên miền / Domain lock** ở trạng thái bật; khóa transfer không cản trở đổi nameserver.
3. Mở tab **DNSSEC**:
   - Nếu trang chỉ có form trống `Key Tag / Algorithm / Digest Type / Digest`, không có DS record đã lưu → DNSSEC đang **tắt**, không cần làm gì.
   - Nếu có DS record → xóa/tắt DNSSEC tại Hostinger, đợi DS TTL hết rồi mới đổi nameserver. Đổi NS khi DS cũ còn tồn tại có thể làm resolver trả `SERVFAIL`.
4. Mở tab **DNS records**, ghi lại mọi record đang thực sự dùng. Đặc biệt không được bỏ sót `MX`, SPF (`TXT`), DKIM (`TXT`/`CNAME`) và DMARC (`TXT`) nếu đang dùng email theo domain. Cloudflare Quick Scan **không bảo đảm** tìm đủ mọi record.

**Verify trên máy host Windows** — thay giá trị bằng domain thật, không thêm `https://` hoặc subdomain:

```powershell
$DomainName = "hieupn.site"

# NS hiện tại trước migration; với domain mua ở Hostinger thường là *.dns-parking.com
Resolve-DnsName -Name $DomainName -Type NS -Server 1.1.1.1

# Gate DNSSEC: trước khi đổi NS không được còn DS record cũ
$OldDs = @(
  Resolve-DnsName -Name $DomainName -Type DS -Server 1.1.1.1 -DnsOnly -ErrorAction SilentlyContinue |
    Where-Object Type -eq 'DS'
)
if ($OldDs) {
  $OldDs
  throw "STOP: còn DS record cũ; xóa/tắt DNSSEC tại Hostinger và đợi DS TTL hết"
} else {
  "PASS: không có DS record cũ"
}
```

PASS khi domain `Active` ở Hostinger, NS hiện tại resolve được, danh sách DNS cần giữ đã được đối chiếu và lệnh DS in `PASS`. Nếu có DS cũ, **dừng tại đây**; Cloudflare hướng dẫn thường phải đợi TTL của DS hết (có thể 24–48 giờ) trước khi thay NS.

#### 11.2.2. Thêm domain tại Cloudflare — chọn Connect, scan và review DNS

1. Đăng nhập [https://dash.cloudflare.com](https://dash.cloudflare.com).
2. Vào **Domains → Overview** → bấm **Add a site / Add a domain**. Tài liệu Cloudflare gọi thao tác này là *Onboard a domain*; đây là nút trong trang Overview, không nhất thiết là tên menu bên trái.
3. Chọn **Connect a domain**.
4. Nhập **apex/root domain**, ví dụ `hieupn.site`; không nhập `https://hieupn.site`, `www.hieupn.site` hoặc `app.hieupn.site`.
5. Chọn **Quick scan / Scan automatically / Import existing DNS records**.
6. Chọn plan **Free** — đủ cho DNS, Universal SSL và Cloudflare Tunnel của lab.
7. Tại trang review DNS records, so sánh từng `Type`, `Name`, `Content`, `Priority` với danh sách đã ghi ở §11.2.1; tự thêm record bị thiếu trước khi đổi nameserver.

Quy tắc review:

- Record web `A`/`AAAA`/`CNAME` có thể để **Proxied** (mây cam) khi cần đi qua Cloudflare.
- Record email và record xác minh (`MX`, mail host, SPF/DKIM/DMARC) phải theo đúng chỉ dẫn của nhà cung cấp email; `MX` và các hostname mail thường để **DNS only**.
- Domain mới chỉ dùng cho lab có thể chỉ có record parking của Hostinger. Có thể giữ tạm trong lúc kích hoạt rồi xóa khi không dùng.
- **Không** tạo `A` record trỏ vào IP LAN `192.168.x.x`, ClusterIP `10.x.x.x` hoặc IP động của router. §12 sẽ tạo hostname cho Tunnel mà không phơi IP nhà.
- Chưa cần tự tạo `app.<domain>`: khi thêm Published application route ở §12.3, Cloudflare sẽ tạo DNS record cho Tunnel trong Full Setup.

**Checkpoint Cloudflare DNS:** chỉ bấm tiếp tục khi không còn record website/email cần giữ bị thiếu. Với domain lab mới không dùng email, PASS khi xác nhận không có MX/DKIM/SPF/DMARC cần migrate và không có record nào trỏ nhầm IP private.

#### 11.2.3. Lấy đúng hai nameserver Cloudflare

Cloudflare cấp đúng **hai authoritative nameserver** riêng cho zone, dạng:

```text
<ns-thu-nhat>.ns.cloudflare.com
<ns-thu-hai>.ns.cloudflare.com
```

Copy nguyên văn cả hai từ onboarding flow hoặc **domain → Overview**. Không dùng tên ví dụ trong runbook, không thêm `https://`, IP hay dấu `/`; không dùng cặp NS của domain/tài khoản khác.

**Checkpoint NS được cấp:** PASS khi Cloudflare hiển thị đúng hai hostname kết thúc bằng `.ns.cloudflare.com` và zone đang ở trạng thái **Pending Nameserver Update**. Trạng thái Pending là đúng ở thời điểm này nhưng **chưa đủ** để chạy §12.

#### 11.2.4. Thay nameserver ở Hostinger

Quay lại **Hostinger hPanel**:

1. **Domains → Domain portfolio → Manage** domain.
2. Trong card **DNS/Máy chủ tên miền**, bấm **Chỉnh sửa**.
3. Chọn **Change nameservers / Use custom nameservers / Thay đổi máy chủ tên miền**.
4. Xóa **toàn bộ** nameserver Hostinger hiện tại (thường là hai tên `*.dns-parking.com`).
5. Dán đúng hai NS Cloudflare ở §11.2.3 vào hai ô nameserver và lưu.

Không thao tác ở tab **Bản ghi DNS**, **Máy chủ tên miền con/Child nameservers** hoặc **DNSSEC**. Không giữ lẫn một NS Hostinger và một NS Cloudflare; Full Setup gói Free chỉ được kích hoạt khi registrar chỉ liệt kê cặp NS Cloudflare được cấp.

**Checkpoint Hostinger:** mở lại trang tổng quan domain và xác nhận card **DNS/Máy chủ tên miền** chỉ hiển thị hai NS Cloudflare, không còn `*.dns-parking.com`. Giữ domain lock bật. Nếu Hostinger yêu cầu xác nhận bảo mật/email, hoàn tất rồi kiểm tra lại card này.

#### 11.2.5. Verify propagation và trạng thái Active

Quay lại Cloudflare, cuộn xuống cuối trang hướng dẫn thay NS và bấm **I updated my nameservers** (một số phiên bản UI dùng **Done, check nameservers / Check nameservers now**). Nút này chỉ báo cho Cloudflare bắt đầu/ưu tiên kiểm tra; propagation thường mất vài phút nhưng có thể tới 24 giờ. Zone `Pending Nameserver Update` chưa được coi là sẵn sàng để proxy traffic.

Trên máy host Windows, thay hai giá trị `$ExpectedNs` bằng đúng NS Cloudflare của zone. Block vừa hiển thị `NameHost/TTL` để quan sát cache, vừa tự so sánh **cả hai resolver** với cặp NS mong đợi:

```powershell
$DomainName = "hieupn.site"
$ExpectedNs = @(
  "<ns-thu-nhat>.ns.cloudflare.com",
  "<ns-thu-hai>.ns.cloudflare.com"
) | ForEach-Object { $_.TrimEnd('.').ToLowerInvariant() } | Sort-Object -Unique

$Resolvers = @('1.1.1.1', '8.8.8.8')
$AllResolversPassed = $true

foreach ($Resolver in $Resolvers) {
  $RawNs = @(
    Resolve-DnsName -Name $DomainName -Type NS -Server $Resolver -DnsOnly -ErrorAction SilentlyContinue |
      Where-Object Type -eq 'NS'
  )

  "`n=== Resolver $Resolver ==="
  $RawNs | Select-Object NameHost, TTL | Format-Table -AutoSize

  $ObservedNs = @(
    $RawNs |
      ForEach-Object { $_.NameHost.TrimEnd('.').ToLowerInvariant() } |
      Sort-Object -Unique
  )
  if ($ObservedNs.Count -gt 0) {
    $NsDiff = @(Compare-Object -ReferenceObject $ExpectedNs -DifferenceObject $ObservedNs)
  } else {
    $NsDiff = @("Không nhận được NS answer từ $Resolver")
  }

  if ($ObservedNs.Count -eq 2 -and -not $NsDiff) {
    "PASS: $Resolver trả đúng hai NS Cloudflare"
  } else {
    $AllResolversPassed = $false
    "Expected: $($ExpectedNs -join ', ')"
    "Observed: $($ObservedNs -join ', ')"
    $NsDiff
  }
}

if (-not $AllResolversPassed) {
  throw "STOP: ít nhất một resolver công cộng chưa trả đúng cặp NS Cloudflare"
}
"PASS: cả 1.1.1.1 và 8.8.8.8 đều trả đúng cặp NS Cloudflare"
```

PASS cuối bước khi **đồng thời** thỏa cả ba điều kiện:

1. Block in `PASS` riêng cho `1.1.1.1` và `8.8.8.8`, rồi in PASS tổng.
2. Bảng của cả hai resolver chỉ thấy đúng hai NS Cloudflare; TTL không còn gắn với NS Hostinger.
3. Cloudflare **Domains → Overview** hiển thị zone **Active** (không chỉ `Pending`).

Nếu Cloudflare đã **Active** và một resolver đã trả NS Cloudflare nhưng resolver còn lại vẫn trả `*.dns-parking.com` với TTL đang đếm xuống, delegation tại registrar đã đúng; resolver kia chỉ còn cache NS cũ. Không đổi NS lại, không xóa/re-add zone.

**Đọc TTL đang đếm xuống.** Giá trị `TTL` trong output là thời gian **còn lại** của bản ghi trong cache resolver, không phải TTL gốc của zone. Lấy TTL gốc trừ đi giá trị này ra thời điểm resolver đã cache. Ví dụ NS có TTL gốc `86400` mà `1.1.1.1` trả `80288` nghĩa là bản ghi được cache cách đây `86400 - 80288 = 6112` giây (1 giờ 42 phút) và còn 22 giờ 18 phút nữa mới tự hết hạn.

**Rút ngắn thời gian chờ bằng DNS cache purge (tùy chọn).** Trong ngữ cảnh này, *purge/flush cache* nghĩa là yêu cầu một **recursive DNS resolver** xóa RRset `NS` đang cache cho domain và tra cứu lại từ hệ thống DNS authoritative. Nó không xóa domain hay DNS record tại nguồn. Ví dụ khi `1.1.1.1` còn giữ NS Hostinger:

```text
Purge NS cache của hieupn.site tại 1.1.1.1
  → 1.1.1.1 bỏ bản sao orbit/horizon.dns-parking.com đang cache
  → hỏi lại delegation public
  → nhận donna/ian.ns.cloudflare.com
  → cache cặp NS mới với TTL mới
```

Cache trên máy (`Clear-DnsClientCache`) không giúp trường hợp này vì block kiểm tra truy vấn thẳng resolver công cộng bằng `-Server`. Hai resolver dùng trong runbook có công cụ refresh công khai — nhập **apex domain** và chọn record type **`NS`**:

| Resolver    | Công cụ purge                                                             |
| ----------- | --------------------------------------------------------------------------- |
| `1.1.1.1` | [https://one.one.one.one/purge-cache/](https://one.one.one.one/purge-cache/) |
| `8.8.8.8` | [https://dns.google/cache](https://dns.google/cache)                         |

Purge DNS resolver **không**:

- xóa DNS record trong zone Cloudflare, đổi nameserver tại Hostinger hoặc xóa domain;
- xóa cache trình duyệt/Windows hay cache của resolver ISP và máy khác;
- xóa nội dung website/CDN. Không dùng **Caching → Purge Everything** trong Cloudflare Dashboard — đó là HTTP/CDN cache, không phải DNS resolver cache;
- ép toàn bộ Internet cập nhật ngay lập tức.

Chỉ purge khi Cloudflare đã báo **Active**. Purge sớm hơn, khi registry vẫn trả NS Hostinger, sẽ làm resolver nạp và cache lại đúng NS cũ thêm một chu kỳ TTL. Ngay cả sau `Active`, purge vẫn là **best-effort**: hạ tầng resolver phân tán có thể chưa cập nhật mọi node ngay và công cụ không tác động resolver của bên thứ ba.

Thao tác tùy chọn:

1. Mở công cụ tương ứng trong bảng.
2. Nhập domain gốc, ví dụ `hieupn.site` — không nhập `https://` hoặc `app.hieupn.site`.
3. Chọn record type `NS` → bấm **Purge Cache** (`1.1.1.1`) hoặc **Flush Cache** (`8.8.8.8`). Chỉ cần purge resolver còn trả NS cũ; resolver đã trả NS Cloudflare không cần purge.
4. Chờ vài phút rồi chạy lại toàn bộ block PowerShell kiểm tra hai resolver ở trên.

**PASS sau purge** chỉ khi block PowerShell in PASS riêng cho `1.1.1.1`, `8.8.8.8` và PASS tổng. Thông báo thành công trên trang purge **không phải** bằng chứng DNS đã đúng. Nếu không muốn purge hoặc purge chưa đổi kết quả, đợi TTL cũ hết rồi chạy lại block; không sửa lại nameserver.

Để tránh cửa sổ `SERVFAIL` khi resolver vẫn dùng NS cũ nhưng registry đã có DS mới, chỉ sang §11.2.6 bật DNSSEC sau khi **cả `1.1.1.1` và `8.8.8.8`** đều trả đúng cặp NS Cloudflare.

Nếu quá 24 giờ vẫn Pending, kiểm tra lại: registrar có đúng **chỉ hai** NS Cloudflare hay chưa, còn NS Hostinger nào không, tên NS có bị gõ sai không và `Resolve-DnsName -Type DS` có còn DS record cũ không.

#### 11.2.6. Bật DNSSEC mới sau khi Cloudflare Active

Chỉ làm bước này **sau** khi §11.2.5 PASS. Vì registrar vẫn là Hostinger, Cloudflare ký zone và sinh DS record; Hostinger đăng DS đó lên registry.

**Vì sao phải qua Hostinger dù DNS đã ở Cloudflare.** DNSSEC là chuỗi tin cậy `. → .site → hieupn.site`, mỗi tầng giữ hash khóa của tầng dưới. DS của `hieupn.site` vì vậy phải nằm trong zone cha `.site`, không nằm trong zone của chính nó — nếu không thì Cloudflare vừa ký vừa tự xác nhận mình, thành vòng tròn và mất hết ý nghĩa xác thực. Chỉ **registrar** mới ghi được vào registry `.site` qua EPP, và registrar ở đây là Hostinger (§11.2 chọn *Connect a domain*, không transfer). Badge **DNS Setup: Full** trên dashboard chỉ nói Cloudflare là authoritative DNS đầy đủ, **không** nói Cloudflare là registrar.

**Điều kiện để Hostinger nhận DS:** tab **DNSSEC** ở hPanel chỉ dùng được khi domain trỏ nameserver ra nhà cung cấp ngoài — đúng trạng thái sau §11.2.4 — và khi TLD/registrar hỗ trợ DNSSEC. Hostinger nói rõ giá trị DS **phải do nhà cung cấp đang giữ nameserver sinh ra**, ở đây là Cloudflare; không tự tạo, không lấy từ nguồn khác. Nếu tab DNSSEC không xuất hiện hoặc bị khóa, **STOP**: xác nhận lại domain đã dùng NS Cloudflare, kiểm tra TLD có hỗ trợ DNSSEC và liên hệ Hostinger nếu cần. Không tự kết luận chỉ từ trạng thái UI và không bỏ qua, vì gate §11.2.7 của baseline này yêu cầu DNSSEC **Active/Confirmed**.

##### Bước 1 — Lấy giá trị tại Cloudflare

1. Cloudflare → chọn domain → thanh điều hướng **bên trái** → mở mục **DNS** → chọn mục con **Settings** → cuộn tới thẻ **DNSSEC** → **Enable DNSSEC**.

   Mục `DNS` ở sidebar bung ra ba mục con; DNSSEC nằm ở `Settings`, không nằm ở `Records`:

   ```text
   ⌄ DNS
     ├─ Records      ← trang "DNS records for <domain>"; KHÔNG có DNSSEC ở đây
     ├─ Analytics
     └─ Settings     ← vào đây, cuộn xuống thẻ DNSSEC
   ```

   Đi thẳng bằng link cũng được: `https://dash.cloudflare.com/?to=/:account/:zone/dns/settings`
2. Cloudflare bắt đầu ký zone và mở panel **DS record**. Panel này là **nguồn duy nhất** của bốn giá trị cần điền. Giữ tab mở cho tới khi Hostinger lưu xong.
3. Copy dòng **DS Record** — chuỗi dài chứa đủ cả bốn giá trị theo đúng thứ tự chuẩn:

```
hieupn.site.   3600   IN   DS   <Key Tag>   <Algorithm>   <Digest Type>   <Digest>
```

Ví dụ minh họa thứ tự lấy từ tài liệu Cloudflare — **không phải giá trị của bạn**: `2371 13 2 32996839A6D808AFE3EB4A795A0E6A7A39A76FC52FF228B22B76F6D63826F2B9` nghĩa là `Key Tag = 2371`, `Algorithm = 13`, `Digest Type = 2`, `Digest = 32996839…3826F2B9`.

Panel Cloudflare hiển thị bốn giá trị này thành từng dòng riêng, kèm cả `Public Key` và `Flags`. Tùy phiên bản UI, `Algorithm` và `Digest Type` có thể hiện **tên** thay vì **số** — xem bảng quy đổi ở bước 3. Khi hai nguồn khác nhau về hình thức, lấy chuỗi `DS Record` làm chuẩn vì nó luôn là số.

##### Bước 2 — Tách và kiểm tra bốn giá trị trước khi nhập

Chạy trên máy host Windows. Block này chỉ đọc chuỗi đã copy, không gọi mạng, và chặn sẵn ba lỗi copy hay gặp nhất:

```powershell
# Chọn đúng MỘT trong hai option khởi tạo $DsRecord dưới đây.

# OPTION 1 — Dán thủ công: giữ dòng này và để OPTION 2 ở trạng thái comment.
# Dán nguyên văn DS Record giữa hai dấu nháy đơn. Panel chỉ hiện bốn số rời cũng dán được,
# ví dụ: '2371 13 2 32996839...3826F2B9'.
$DsRecord = '<dán DS Record của Cloudflare vào đây>'

# OPTION 2 — Đọc clipboard: sau khi bấm icon Copy cạnh DS Record trên Cloudflare,
# comment dòng OPTION 1 ở trên rồi bỏ dấu # ở dòng dưới. Out-String dùng được với Windows PowerShell 5.1.
# $DsRecord = (Get-Clipboard | Out-String).Trim()

$Parts = ($DsRecord -replace '(?i)^.*\sIN\s+DS\s+', '').Trim() -split '\s+' | Where-Object { $_ }
if ($Parts.Count -ne 4) {
  throw "STOP: tách được $($Parts.Count) trường, cần đúng 4; copy lại nguyên văn DS Record từ Cloudflare"
}

$Ds = [pscustomobject][ordered]@{
  KeyTag     = $Parts[0]
  Algorithm  = $Parts[1]
  DigestType = $Parts[2]
  Digest     = $Parts[3].ToUpperInvariant()
}

$KeyTagNum = 0
if (-not [int]::TryParse($Ds.KeyTag, [ref]$KeyTagNum) -or $KeyTagNum -lt 0 -or $KeyTagNum -gt 65535) {
  throw "STOP: Key Tag '$($Ds.KeyTag)' không phải số nguyên unsigned 16-bit (0-65535); nhiều khả năng đã copy nhầm Flags hoặc dính ký tự thừa"
}
$AlgorithmNum = 0
if (-not [int]::TryParse($Ds.Algorithm, [ref]$AlgorithmNum) -or $AlgorithmNum -lt 0 -or $AlgorithmNum -gt 255) {
  throw "STOP: Algorithm '$($Ds.Algorithm)' không phải số nguyên unsigned 8-bit (0-255); copy lại chuỗi DS Record dạng số"
}
$DigestTypeNum = 0
if (-not [int]::TryParse($Ds.DigestType, [ref]$DigestTypeNum) -or $DigestTypeNum -lt 0 -or $DigestTypeNum -gt 255) {
  throw "STOP: Digest Type '$($Ds.DigestType)' không phải số nguyên unsigned 8-bit (0-255); copy lại chuỗi DS Record dạng số"
}
if ($Ds.Digest -notmatch '^[0-9A-F]+$') {
  throw "STOP: Digest chứa ký tự không phải hex; nhiều khả năng đã copy nhầm Public Key sang ô Digest"
}
if ($Ds.DigestType -eq '2' -and $Ds.Digest.Length -ne 64) {
  throw "STOP: Digest Type 2 (SHA-256) phải đúng 64 ký tự hex, đang có $($Ds.Digest.Length); copy thiếu hoặc dính xuống dòng"
}
if ($Ds.Algorithm -eq '13' -and $Ds.DigestType -eq '2') {
  "INFO: Algorithm 13 / Digest Type 2 là cặp Cloudflare thường cấp"
} else {
  "INFO: DS dùng Algorithm/Digest Type $($Ds.Algorithm)/$($Ds.DigestType); chỉ tiếp tục nếu đúng nguyên văn DS Record trên panel Cloudflare"
}

$Ds | Format-List
"PASS: bốn giá trị tách đúng định dạng; nhập vào Hostinger theo đúng nhãn ở trên"
```

**PASS Bước 2** khi block không ném `STOP`, output `Format-List` có đúng bốn trường `KeyTag/Algorithm/DigestType/Digest`, rồi kết thúc bằng dòng `PASS`. Dòng `INFO` về `13/2` chỉ mô tả giá trị thường gặp; giá trị quyết định luôn là chuỗi `DS Record` Cloudflare vừa cấp.

Giữ nguyên phiên PowerShell này; biến `$Ds` sẽ được dùng lại ở bước verify.

##### Bước 3 — Nhập tại Hostinger, đúng ánh xạ một-một

Hostinger hPanel → **Domains → Domain portfolio → Manage** domain → **DNS / Nameservers → DNSSEC**. Form có đúng bốn ô:

| Ô ở Hostinger | Lấy từ đâu trên panel Cloudflare | Vị trí trong chuỗi`DS Record` | Dạng giá trị hợp lệ                              |
| --------------- | ------------------------------------- | ---------------------------------- | ----------------------------------------------------- |
| `Key Tag`     | dòng**Key Tag**                | trường thứ 1 sau`DS`          | số nguyên 0–65535, ví dụ`2371`                 |
| `Algorithm`   | dòng**Algorithm**              | trường thứ 2                    | lấy đúng số Cloudflare cấp; thường là`13`   |
| `Digest Type` | dòng**Digest Type**            | trường thứ 3                    | lấy đúng số Cloudflare cấp; thường là`2`    |
| `Digest`      | dòng**Digest**                 | trường thứ 4                    | chuỗi hex liền, 64 ký tự khi Digest Type là`2` |

Khi panel Cloudflare hoặc dropdown Hostinger ghi tên thay vì số, quy đổi theo bảng này — **quy đổi tên sang số, không đổi giá trị**:

| Trường        | Số    | Tên tương đương có thể gặp                     |
| --------------- | ------ | ------------------------------------------------------- |
| `Algorithm`   | `13` | `ECDSAP256SHA256`, `ECDSA Curve P-256 with SHA-256` |
| `Digest Type` | `2`  | `SHA256`, `SHA-256`                                 |

**Hai giá trị trên panel Cloudflare không có ô tương ứng ở Hostinger — không nhét vào ô nào:**

- `Public Key` — chuỗi base64 của bản ghi `DNSKEY`, chỉ dùng cho registrar nhập theo dạng DNSKEY. Nhầm nó vào ô `Digest` là lỗi phổ biến nhất; phân biệt nhanh: `Digest` chỉ gồm `0-9` và `A-F`, còn `Public Key` có chữ thường và các dấu `+` `/` `=`.
- `Flags` (`257`) — thuộc bản ghi `DNSKEY`, không thuộc DS. Đừng điền vào `Key Tag`.

Thao tác nhập:

1. Dán bốn giá trị từ output block bước 2 vào đúng bốn ô cùng tên.
2. `Digest` hex không phân biệt hoa/thường; block bước 2 đã chuẩn hóa thành chữ hoa. Không sửa nội dung, không thêm khoảng trắng/dấu chấm cuối và không bọc nháy khi nhập form.
3. Bấm **Thêm/Add**. Nếu form báo sai định dạng, kiểm tra lại độ dài `Digest` và ký tự thừa ở `Key Tag` — **không** sửa giá trị cho vừa form.
4. Chờ DS propagate lên registry và Cloudflare chuyển DNSSEC sang **Active**. Trong lúc chờ, không bấm **Enable DNSSEC** lần nữa và không xóa rồi tạo lại DS ở Hostinger: mỗi lần bật lại, Cloudflare có thể sinh khóa mới làm DS vừa đăng trở thành sai.

**Verify DNSSEC trên Windows** — chạy tiếp trong cùng phiên PowerShell với block ở bước 2:

```powershell
$DomainName = "hieupn.site"

# $Ds đến từ block bước 2. Mở phiên PowerShell mới thì chạy lại block đó trước.
if (-not $Ds) { throw "STOP: chưa có `$Ds; chạy lại block tách DS record ở bước 2" }

$NewDs = @(
  Resolve-DnsName -Name $DomainName -Type DS -Server 1.1.1.1 -DnsOnly -ErrorAction SilentlyContinue |
    Where-Object Type -eq 'DS'
)
if (-not $NewDs) { throw "STOP: chưa thấy DS record mới; chưa coi DNSSEC là hoàn tất" }

$NewDs | Format-Table Name, Type, KeyTag, Algorithm, DigestType, Digest -AutoSize

# So khớp đủ bốn trường DS đang publish với giá trị Cloudflare cấp.
# Resolve-DnsName trả Algorithm/DigestType dạng enum nhưng ép [int] cho đúng mã số;
# Digest có thể là byte[] nên được chuẩn hóa thành chuỗi hex chữ hoa.
$Published = $NewDs | ForEach-Object {
  $Raw = $_.Digest
  if ($Raw -is [byte[]]) { $Hex = (($Raw | ForEach-Object { $_.ToString('X2') }) -join '') }
  else { $Hex = ((($Raw -join '') -replace '[^0-9A-Fa-f]', '')).ToUpperInvariant() }
  [pscustomobject][ordered]@{
    KeyTag     = "$([int]$_.KeyTag)"
    Algorithm  = "$([int]$_.Algorithm)"
    DigestType = "$([int]$_.DigestType)"
    Digest     = $Hex
  }
}

if (-not ($Published | Where-Object {
  $_.KeyTag -eq $Ds.KeyTag -and
  $_.Algorithm -eq $Ds.Algorithm -and
  $_.DigestType -eq $Ds.DigestType -and
  $_.Digest -eq $Ds.Digest
})) {
  $Published | Format-List
  throw "STOP: DS đang publish không khớp đủ bốn trường Cloudflare cấp; sửa record ở Hostinger, không đổi gì ở Cloudflare"
}
"PASS: DS record publish khớp đủ Key Tag/Algorithm/Digest Type/Digest Cloudflare cấp"

# 1.1.1.1 là validating resolver; truy vấn SOA phải trả kết quả, không được SERVFAIL
Resolve-DnsName -Name $DomainName -Type SOA -Server 1.1.1.1 -DnssecOk
```

PASS khi Hostinger hiển thị DS record đã lưu, block trên xác nhận DS public khớp đủ cả bốn trường, truy vấn `SOA` không `SERVFAIL`, và Cloudflare DNSSEC hiển thị **Active/Confirmed**. Nếu chưa PASS, **không xóa/disable DNSSEC theo thứ tự tùy ý**; khi rollback phải xóa DS ở registrar trước rồi chờ DS TTL hết, sau đó mới tắt signing tại Cloudflare.

#### 11.2.7. Gate hoàn thành mục 11

Chỉ chuyển sang §12 khi:

- Hostinger vẫn quản lý đăng ký/gia hạn; domain lock bật.
- DNS authoritative public chỉ có đúng hai NS Cloudflare.
- Cloudflare zone là **Active**.
- DNS records website/email cần giữ đã có đầy đủ ở Cloudflare.
- DNSSEC mới là **Active/Confirmed** và kiểm tra không `SERVFAIL`.

> **Checkpoint — gửi để kiểm tra trước §12:** ảnh Cloudflare Overview chỉ cần thấy `Active` (che account/email), output hai lệnh NS qua `1.1.1.1` và `8.8.8.8`, cùng output verify DNSSEC. Nameserver/DS record là dữ liệu public; không gửi mật khẩu Hostinger/Cloudflare, mã EPP, API token, tunnel token hoặc thông tin liên hệ chủ domain.

---

## 12. Tạo Cloudflare Tunnel (chạy trong cụm)

Dùng **remotely-managed tunnel** (quản lý qua dashboard bằng *token*) — đơn giản, hợp với cụm k8s và đúng theo deployment guide của Cloudflare. (Cách CLI `config.yml` xem [Phụ lục A](#phụ-lục-a--locally-managed-tunnel-cli--configyml).)

Thực hiện đúng thứ tự `§12.1 → §12.2 → §12.3`. Ở giao diện hiện tại, nút **Continue** chỉ được bật sau khi Cloudflare phát hiện connector đã kết nối; vì vậy không cố hoàn tất wizard ở §12.1 và không chạy lệnh Docker mà dashboard hiển thị.

### 12.1. Tạo tunnel trên dashboard → lấy token (chưa bấm Continue)

1. Cloudflare dashboard → **Networking → Tunnels** → **Create Tunnel**.
2. Giao diện hiện tại đi thẳng đến form **Create a Tunnel**; đây đã là luồng tạo connector `cloudflared`, nên không còn bước chọn loại **Cloudflared** riêng.
3. Tại **Tunnel name**, nhập `homelab-k8s` → **Create Tunnel**.
4. Tại màn hình **Setup Environment** / *Install and run*, chọn **Docker**. Lựa chọn này chỉ để dashboard hiển thị lệnh có token ở dạng dễ nhận biết; **không chạy** lệnh `docker run` vì connector sẽ được deploy vào Kubernetes ở §12.2.
5. Bấm biểu tượng copy cạnh lệnh, rồi lấy riêng chuỗi nằm sau `--token` trong lệnh có dạng `cloudflared ... run --token eyJh...`. Token rất dài; không chụp, gửi vào chat hoặc ghi vào lịch sử lệnh shell.
6. Giữ nguyên trang wizard. **Connection Status** hiển thị `Waiting for your Tunnel to connect` / `No connection detected yet` và nút **Continue** đang bị khóa là trạng thái dự kiến lúc này.

**Verify §12.1:** kiểm tra đủ ba dấu hiệu trên dashboard:

- **Tunnel name** là `homelab-k8s`.
- **Operating System** đang chọn **Docker**.
- Lệnh *Run tunnel with Docker* có tham số `--token` và đã copy được token, nhưng chưa chạy lệnh.

**PASS §12.1:** đủ ba dấu hiệu trên. Nút **Continue** chưa bật không phải lỗi; chuyển thẳng xuống §12.2. **STOP** nếu dashboard không sinh lệnh chứa `--token` hoặc token đã bị lộ — khi đó không deploy và phải rotate token trước.

### 12.2. Deploy cloudflared vào cụm

**Trên master**, tạo `cloudflared.yaml` (dán token vào):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cloudflare
---
apiVersion: v1
kind: Secret
metadata:
  name: tunnel-token
  namespace: cloudflare
stringData:
  token: <DÁN_TUNNEL_TOKEN_eyJh...>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: cloudflare
spec:
  replicas: 2                       # HA (cloudflared KHÔNG load-balance giữa replica, chỉ để dự phòng)
  selector:
    matchLabels: { app: cloudflared }
  template:
    metadata:
      labels: { app: cloudflared }
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest
          args:
            - tunnel
            - --no-autoupdate
            - --loglevel
            - info
            - --metrics
            - 0.0.0.0:2000
            - run
          env:
            - name: TUNNEL_TOKEN
              valueFrom:
                secretKeyRef:
                  name: tunnel-token
                  key: token
          livenessProbe:
            httpGet:
              path: /ready
              port: 2000
            failureThreshold: 1
            initialDelaySeconds: 10
            periodSeconds: 10
```

```bash
kubectl apply -f cloudflared.yaml
kubectl -n cloudflare rollout status deployment/cloudflared --timeout=180s
kubectl -n cloudflare get deployment cloudflared
kubectl -n cloudflare get pods -l app=cloudflared -o wide
```

Output mong đợi:

- `rollout status` kết thúc bằng `successfully rolled out`.
- Deployment có `READY 2/2` và `AVAILABLE 2`.
- Cả hai Pod có `STATUS Running`, cột `READY` là `1/1` và không restart lặp lại.

Nếu rollout không PASS, chưa quay lại wizard và chưa tạo route. Chẩn đoán trước bằng:

```bash
kubectl -n cloudflare describe deployment cloudflared
kubectl -n cloudflare describe pods -l app=cloudflared
kubectl -n cloudflare logs deployment/cloudflared --tail=100
```

Không gửi log nếu log làm lộ token. Sửa lỗi Pod/egress trước rồi chạy lại ba lệnh verify ở trên.

Sau khi Kubernetes PASS:

1. Quay lại đúng trang wizard đang giữ ở §12.1.
2. Chờ **Connection Status** phát hiện connector; nút **Continue** sẽ được bật. Nếu UI chưa cập nhật, đợi ngắn rồi refresh đúng một lần, không tạo tunnel mới và không chạy thêm lệnh Docker.
3. Chọn **Continue** để kết thúc bước kết nối và mở trang cấu hình route/tunnel.
4. Vào **Networking → Tunnels** nếu dashboard chuyển về trang khác; tunnel `homelab-k8s` phải hiển thị **Healthy** với hai connector/replica.

**Verify §12.2:** chạy lại:

```bash
kubectl -n cloudflare get deployment cloudflared
kubectl -n cloudflare get pods -l app=cloudflared
```

Đồng thời xác nhận dashboard hiển thị tunnel `homelab-k8s` là **Healthy**.

**PASS §12.2:** Deployment vẫn `READY 2/2`, hai Pod vẫn `Running 1/1`, dashboard đã phát hiện connector, đã bấm được **Continue**, và tunnel là **Healthy**. Chỉ khi đủ các điều kiện này mới chuyển sang §12.3.

### 12.3. Chốt hostname thật trong Ingress → thêm Published application

#### 12.3.1. Đổi Ingress từ hostname tạm sang hostname thật

Domain đã PASS §11 là `hieupn.site`, nên hostname public của app trong lab này là `app.hieupn.site`. Kubernetes không tự lấy domain từ Cloudflare: phải đổi `spec.rules[].host` trong manifest rồi apply lại trước khi tạo route.

**Trên master**, kiểm tra hostname hiện tại:

```bash
kubectl get ingress web -o jsonpath='{.spec.rules[*].host}{"\n"}'
```

Nếu đã làm đúng §10, output lúc này là `app.example.com`; đây là trạng thái tạm dự kiến, **chưa phải PASS §12.3.1**.

Mở source manifest:

```bash
nano ~/demo-app.yaml
```

Trong phần `kind: Ingress`, chỉ đổi:

```yaml
- host: app.example.com
```

thành:

```yaml
- host: app.hieupn.site
```

Lưu file, dry-run rồi apply:

```bash
kubectl apply --dry-run=client -f ~/demo-app.yaml
kubectl apply -f ~/demo-app.yaml
```

Verify object đang chạy và test lại toàn bộ đường nội bộ qua Traefik bằng Host header mới:

```bash
kubectl get ingress web -o jsonpath='{.spec.rules[*].host}{"\n"}'
ING_IP=$(kubectl get svc -n traefik traefik -o jsonpath='{.spec.clusterIP}')
curl -sS -H 'Host: app.hieupn.site' "http://$ING_IP/"
```

**PASS §12.3.1:** lệnh đầu in đúng `app.hieupn.site` và `curl` trả nội dung `nginxdemos/hello` như test §10.3. **STOP** nếu hostname còn là `app.example.com`, `curl` trả `404`, hoặc không có nội dung ứng dụng; chưa tạo route Cloudflare cho tới khi gate này PASS.

#### 12.3.2. Hiểu đầy đủ luồng Cloudflare Tunnel → Kubernetes

Điểm cốt lõi: **Cloudflare không chủ động mở một kết nối mới từ Internet vào IP của lab**. Chính các Pod `cloudflared` đang nằm trong Kubernetes đã mở kết nối outbound tới Cloudflare và giữ các kết nối đó hoạt động. Khi có request, Cloudflare truyền request xuống chính đường kết nối đã được mở sẵn.

Phần giải thích đầy đủ đã tách sang [`cloudflare-docs/tunnel-traefik.md`](cloudflare-docs/tunnel-traefik.md) để runbook không phình dài. Tài liệu đó gồm:

- sơ đồ tổng quan và sơ đồ tuần tự của toàn bộ chuỗi request/response;
- bản kể lại luồng §12 → §13 bằng ngôn ngữ đơn giản, kèm bảng phân biệt bốn lớp “route”;
- luồng end-to-end theo 14 bước, từ lúc gõ URL tới lúc Pod web trả response;
- 12 mục đào sâu từng hop: connector outbound qua NAT, hai hệ DNS độc lập, Host header đi xuyên các chặng, cách Service chọn Pod backend, và vì sao browser dùng HTTPS trong khi Service URL dùng HTTP.

File đó không chứa lệnh hay gate nào. Đọc để hiểu cơ chế, rồi quay lại [§12.3.3](#1233-thêm-published-application) để thao tác.

#### 12.3.3. Thêm Published application

Chỉ sau khi §12.3.1 PASS, mở tunnel `homelab-k8s` → **Routes** / **Published application routes** → **Add route** → **Published application** (một số tài khoản/UI cũ còn hiện “Public Hostname”):

| Trường              | Giá trị                                       |
| --------------------- | ----------------------------------------------- |
| **Subdomain**   | `app` (để trống nếu dùng root domain)    |
| **Domain**      | `hieupn.site` (chọn từ dropdown)            |
| **Service URL** | `http://traefik.traefik.svc.cluster.local:80` |

> **UI Cloudflare có hai biến thể.** Bản mới gộp `Type` và `URL` thành **một ô `Service URL`**, và scheme phải nằm trong chuỗi — thiếu `http://` sẽ báo *Invalid service URL format (must start with protocol like https://, tcp://, etc.)*. Bản cũ tách thành dropdown `Type` = `HTTP` và ô `URL` = `traefik.traefik.svc.cluster.local:80` (không scheme). Hai cách khai báo cùng một đích; điền theo đúng biến thể đang hiện trên màn hình.

→ **Save**. Với DNS **Full Setup** như [§11.2](#112-thêm-domain-vào-cloudflare), Cloudflare tự tạo DNS record cho hostname. Nếu zone dùng **Partial/CNAME Setup**, phải tự tạo CNAME tại DNS provider theo hướng dẫn Cloudflare.

> Vì cloudflared chạy *trong* cụm, nó resolve được tên DNS nội bộ `*.svc.cluster.local` và gọi thẳng ClusterIP của Traefik. Hostname `app.hieupn.site` được giữ nguyên trong Host header → Traefik khớp đúng Router đã verify ở §12.3.1.
>
> (Tuỳ chọn) Nếu cần ép Host header, mở **Additional application settings → HTTP Settings → HTTP Host Header** = `app.hieupn.site`.

**Verify §12.3.3:**

1. Trong tunnel `homelab-k8s`, tab **Routes** / **Published application routes** phải có route `app.hieupn.site` trỏ tới `http://traefik.traefik.svc.cluster.local:80`.
2. Cloudflare → **DNS → Records** phải có record cho `app.hieupn.site` ở trạng thái **Proxied**; không tự tạo record thứ hai nếu dashboard đã tạo tự động.
3. Tunnel vẫn **Healthy** và hai Pod `cloudflared` vẫn `Running`:

```bash
kubectl -n cloudflare get pods -l app=cloudflared
```

**PASS §12.3.3 / hoàn thành §12:** route và DNS record `app.hieupn.site` tồn tại đúng một lần, tunnel vẫn **Healthy**, và hai Pod vẫn `Running 1/1`. Sau đó mới chuyển sang §13 để kiểm tra DNS public và HTTPS từ Internet.

---

## 13. Trỏ domain & kiểm tra trên Internet

### 13.1. Verify DNS public và hiểu vì sao kết quả là IP Cloudflare

Chạy từ master hoặc một máy khác có Internet:

```bash
nslookup app.hieupn.site
```

Lệnh này hỏi DNS: “client phải kết nối tới địa chỉ nào khi truy cập `app.hieupn.site`?”. Vì record Tunnel đang **Proxied**, câu trả lời phải là một hoặc nhiều IPv4/IPv6 của Cloudflare Edge, không phải IP router, master, worker, ClusterIP Traefik hoặc Pod IP của lab.

```text
app.hieupn.site
→ Cloudflare Edge IP
→ tunnel homelab-k8s
→ cloudflared trong Kubernetes
→ Traefik
→ ứng dụng web
```

Ví dụ output có thể chứa:

```text
Server:  127.0.0.53
Address: 127.0.0.53#53

Non-authoritative answer:
Name:    app.hieupn.site
Address: 104.x.x.x
Name:    app.hieupn.site
Address: 172.x.x.x
Name:    app.hieupn.site
Address: 2606:4700:...
```

Cần phân biệt:

- `Server: 127.0.0.53` là DNS stub resolver cục bộ trên Ubuntu master (`systemd-resolved`), không phải web server hay Cloudflare origin. Xem DNS upstream thật bằng `resolvectl status`.
- Các địa chỉ trong phần `Non-authoritative answer` mới là kết quả phân giải `app.hieupn.site`.
- Nhiều IPv4/IPv6 là bình thường vì Cloudflare dùng mạng Anycast và nhiều địa chỉ để tăng tính sẵn sàng; không coi một IP cụ thể là một máy chủ Cloudflare cố định.

Dashboard lưu ánh xạ tương đương `app.hieupn.site → <TUNNEL-UUID>.cfargotunnel.com`, nhưng khi record được Proxied, DNS public thường trả Cloudflare Edge IP thay vì công bố CNAME tunnel hoặc bất kỳ IP nội bộ nào. Xem [Cloudflare — Tunnel DNS records](https://developers.cloudflare.com/tunnel/routing/#dns-records).

`nslookup` PASS chứng minh:

- hostname public đã có DNS record và resolver nhìn thấy record;
- proxy Cloudflare đang được áp dụng;
- client sẽ tới Cloudflare Edge thay vì kết nối thẳng vào lab.

`nslookup` **chưa** chứng minh tunnel đang Healthy, `cloudflared` gọi được Traefik, Ingress khớp Host header hay Pod web trả response. Các tầng đó được kiểm tra ở §13.2.

**PASS §13.1:** `app.hieupn.site` trả Cloudflare Edge IPv4/IPv6, không trả IP riêng/IP public của lab và không có `NXDOMAIN`/timeout.

### 13.2. Verify HTTPS end-to-end bằng response headers

```bash
curl -I https://app.hieupn.site
```

`-I` yêu cầu `curl` gửi HTTP `HEAD` và chỉ in response headers, không tải body trang. Trong một lệnh, `curl` lần lượt kiểm tra:

1. DNS phân giải được `app.hieupn.site`.
2. Kết nối được tới Cloudflare Edge qua HTTPS port `443`.
3. TLS certificate hợp lệ và khớp hostname `app.hieupn.site`.
4. Cloudflare tìm thấy Published application route.
5. Cloudflare truyền request xuống tunnel `homelab-k8s` qua connector đang Healthy.
6. Pod `cloudflared` gọi được `traefik.traefik.svc.cluster.local:80`.
7. Traefik nhận `Host: app.hieupn.site` và khớp Ingress `web`.
8. Service/Pod web xử lý được request `HEAD` và trả response.

Lệnh này khác test nội bộ ở §12.3.1:

| Lệnh | Phạm vi được kiểm tra |
| --- | --- |
| `curl -H 'Host: app.hieupn.site' "http://$ING_IP/"` | Traefik → Ingress → Service web → Pod; bỏ qua DNS public, Cloudflare và Tunnel |
| `curl -I https://app.hieupn.site` | DNS public → Cloudflare Edge/TLS → Tunnel → `cloudflared` → Traefik → ứng dụng |

Output PASS dự kiến có status thành công, tùy phiên bản `curl` có thể hiển thị HTTP/2 hoặc HTTP/1.1:

```text
HTTP/2 200
server: cloudflare
cf-ray: ...
```

`server: cloudflare` cho biết response đi qua Cloudflare Edge, không phải tên Pod backend. `cf-ray` là định danh request tại Cloudflare. Một số lỗi giúp khoanh vùng:

| Kết quả | Tầng cần kiểm tra trước |
| --- | --- |
| `Could not resolve host` | DNS public/resolver |
| lỗi certificate | TLS/Universal SSL/hostname |
| HTTP `404` | Published route hoặc Host của Ingress không khớp |
| HTTP `502`/`504` | `cloudflared` không gọi được Traefik/backend hoặc origin không trả lời |
| Cloudflare `1016` | DNS còn record nhưng tunnel/origin route không dùng được |

**PASS §13.2:** `curl` không có lỗi DNS/TLS/connection và nhận HTTP `200` qua Cloudflare. Nếu nhận redirect `301`/`302`, phải xác nhận đích redirect đúng rồi chạy lại URL đích; với cấu hình hiện tại dự kiến trả thẳng `200`.

### 13.3. Verify response body và trình duyệt

`curl -I` không tải nội dung nên sau khi header PASS, kiểm tra body thật:

```bash
curl -sS https://app.hieupn.site
```

Output phải chứa thông tin từ `nginxdemos/hello`, ví dụ `Server address`, `Server name`, `URI` và `Request ID`. `Server address` là Pod IP/port của backend web; `Server name` là tên Pod web đã phục vụ request, không phải Cloudflare Edge hay Traefik.

Cuối cùng mở trình duyệt → `https://app.hieupn.site` → thấy trang `nginxdemos/hello`. Khi đó toàn bộ chuỗi đã thông:

```text
Browser → Cloudflare Edge (HTTPS/TLS) → Tunnel mã hóa → cloudflared Pod →
Traefik Service/Pod → Ingress Router (Host match) → Service web → Pod web
```

**PASS §13 / hoàn thành public app:** §13.1 DNS PASS, §13.2 HTTPS headers trả `200`, §13.3 body/trình duyệt hiển thị ứng dụng.

> **TLS:** Cloudflare lo certificate ở edge (Internet ⇄ Cloudflare = HTTPS). Đoạn Cloudflare ⇄ cluster đi trong tunnel mã hóa; chặng `cloudflared` ⇄ Traefik của lab dùng HTTP nội bộ nên không cần cert nội bộ. Traefik không tự redirect HTTP→HTTPS trong cấu hình này; với `ingress-nginx` thì đây là chỗ thường phải thêm annotation `ssl-redirect: "false"`.

---

## 14. Cài Rancher 2.14.3 & quản lý cụm

> Cài Rancher **vào chính cụm kubeadm** vừa dựng (Rancher chạy như workload trong namespace `cattle-system`). Cụm host nó sẽ tự xuất hiện trong UI Rancher dưới tên **`local`** — **không cần import thủ công**. Muốn quản thêm cụm khác sau này thì dùng **Cluster Management → Import Existing**.
>
> **Hostname đã chốt:** `rancher.hieupn.site`. Chưa tạo Published application route/DNS public cho hostname này cho tới khi Rancher, certificate và phép thử HTTPS nội bộ ở §14.4 đều PASS.
>
> **Phạm vi hỗ trợ:** chart Rancher `2.14.3` khai báo `kubeVersion: < 1.36.0-0`; support matrix v2.14.3 bao phủ Kubernetes `1.33–1.35` cho imported/other clusters, nên Kubernetes `1.35.6` của lab nằm trong dải tương thích. Tuy nhiên kubeadm tự dựng không nằm trong danh sách nền tảng **Rancher Manager host** được SUSE chứng nhận (RKE2, K3s và các managed Kubernetes được liệt kê riêng). Đây là cấu hình homelab tương thích về kỹ thuật, không phải topology Rancher production được chứng nhận end-to-end.

### 14.0. Gate trước khi thay đổi cluster

Trên **master**, chạy toàn bộ gate read-only trước. §14 này giả định cài mới; nếu đã có release/namespace `cert-manager` hoặc `rancher`, **STOP** và dùng quy trình upgrade/repair thay vì chạy lại lệnh install:

```bash
helm version --short
kubectl version
kubectl get nodes -o wide
kubectl auth can-i '*' '*' --all-namespaces
kubectl top nodes
kubectl describe nodes | grep -A6 'Allocated resources'
kubectl get pods,svc -n traefik
kubectl get ingressclass traefik
kubectl get pods -n cloudflare
helm list -A | grep -E '(^|[[:space:]])(cert-manager|rancher)([[:space:]]|$)' || true
kubectl get namespace cert-manager cattle-system --ignore-not-found
```

**PASS gate read-only** khi:

- Helm ≥ `3.18`; Kubernetes Client/Server = `v1.35.6`.
- Cả 3 node `Ready`, metrics trả đủ 3 node và memory requests còn dưới khoảng 50% trước khi cài.
- Tài khoản hiện tại có `cluster-admin` (`yes`).
- Traefik và hai Pod `cloudflared` đều `Running`; IngressClass `traefik` tồn tại.
- Hai lệnh cuối không phát hiện cài đặt Rancher/cert-manager cũ. Namespace tồn tại nhưng Helm release không tồn tại cũng phải được điều tra trước, không mặc định tái sử dụng.

#### 14.0.1. Backup etcd và cấu hình trước thay đổi lớn

Gate read-only PASS rồi mới backup. Bước này **bắt buộc trước §14.1**: file `coredns-before-rancher.yaml` ở §14.2 chỉ rollback được ConfigMap CoreDNS; nó không hoàn tác CRD, webhook, controller và toàn bộ state mà cert-manager/Rancher sẽ ghi vào cluster — đường lui duy nhất là snapshot etcd.

Với 1 control plane, etcd là nơi giữ toàn bộ state Kubernetes (và sau §14 là cả state Rancher). Quy trình này được dùng lại cho mọi upgrade hoặc thay đổi quan trọng về sau — [§15](#15-vận-hành--troubleshooting) trỏ về đây.

**Bước 1 — tạo và verify backup, chạy trên `k8s-master`.** Toàn bộ gate chạy trong subshell `( ... )`: `set -e` dừng subshell ngay khi một lệnh lỗi nhưng không đóng shell SSH cha. Mã thoát được giữ lại để in đúng một verdict PASS/FAIL và để `$?` phản ánh đúng kết quả. Backup chứa private key và Secret nên thư mục/file bắt buộc dùng quyền `700/600`.

```bash
STAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_ROOT="$HOME/k8s-backups"
BACKUP_DIR="$BACKUP_ROOT/$STAMP"
BACKUP_ARCHIVE="$BACKUP_DIR.tar.gz"
STAGING_FILE="/var/lib/etcd/kubeadm-snapshot-$STAMP.db"

(
  set -e
  umask 077

  mkdir -p "$BACKUP_ROOT"
  chmod 700 "$BACKUP_ROOT"
  # Không dùng -p: nếu timestamp đích đã tồn tại thì fail thay vì trộn với backup cũ.
  mkdir -m 700 "$BACKUP_DIR"

  # Snapshot etcd vào volume hostPath của static Pod.
  kubectl -n kube-system exec etcd-k8s-master -- etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    snapshot save "$STAGING_FILE"

  # Copy snapshot + cấu hình kubeadm, rồi so snapshot theo từng byte.
  sudo cp "$STAGING_FILE" "$BACKUP_DIR/etcd-snapshot.db"
  sudo cp -a /etc/kubernetes "$BACKUP_DIR/etc-kubernetes"
  sudo chown -R "$(id -u):$(id -g)" "$BACKUP_DIR"
  test -s "$BACKUP_DIR/etcd-snapshot.db"
  sudo cmp -s "$STAGING_FILE" "$BACKUP_DIR/etcd-snapshot.db"

  # Chỉ xóa staging sau khi bản copy tồn tại, khác rỗng và trùng từng byte.
  sudo rm -f "$STAGING_FILE"

  # Một archive bao phủ cả snapshot và /etc/kubernetes; umask 077 + chmod giữ mode 600.
  tar -C "$BACKUP_ROOT" -czf "$BACKUP_ARCHIVE" "$STAMP"
  chmod 600 "$BACKUP_ARCHIVE"
  test "$(stat -c '%a' "$BACKUP_ROOT")" = 700
  test "$(stat -c '%a' "$BACKUP_ARCHIVE")" = 600
  sha256sum "$BACKUP_ARCHIVE"
)
BACKUP_RC=$?

if [ "$BACKUP_RC" -eq 0 ]; then
  echo "PASS: backup $STAMP hợp lệ"
else
  echo 'FAIL: backup chưa hợp lệ — xem lệnh lỗi ngay phía trên, KHÔNG sang §14.1'
fi

# Trả đúng mã lỗi nhưng chỉ thoát subshell này, không đóng phiên SSH.
( exit "$BACKUP_RC" )
# PASS: dòng verdict là "PASS: backup <STAMP> hợp lệ"; ngay trước đó có hash SHA-256
# của file .tar.gz. Ghi lại STAMP và hash để dùng ở bước 2.
# Lần chạy FAIL để lại file staging của lần đó trong /var/lib/etcd; chẩn đoán xong thì dọn
# để khỏi chiếm disk của volume etcd:  sudo rm -f /var/lib/etcd/kubeadm-snapshot-*.db
```

Verify dừng ở mức exit code, `cmp` và checksum là có chủ đích: image etcd mà kubeadm kéo về chỉ đóng gói `etcd` và `etcdctl` (etcd 3.6 đã bỏ `etcdctl snapshot status`), nên không kiểm metadata snapshot tại chỗ được. Hash toàn vẹn nhúng trong snapshot sẽ được `etcdutl snapshot restore` tự kiểm ở thời điểm restore và từ chối chạy nếu file hỏng — backup lỗi sẽ lộ ra, không restore im lặng.

**Bước 2 — copy ra ngoài VM và đối chiếu checksum, chạy trong PowerShell trên máy host Windows.** VM đã chạy `sshd`; gate kiểm tra `scp` đã có sẵn và dừng nếu máy host thiếu công cụ. Đổi user/IP theo môi trường của bạn ([bảng IP ở §2.2](#22-ip--hostname-ví-dụ--đổi-theo-dải-lan-của-bạn)). Chỉ copy một file `.tar.gz` nên một lần so hash phủ trọn gói backup (snapshot + `etc-kubernetes`). Cả block bọc trong `& { ... }` — vai trò như subshell của bước 1: console PowerShell 5.1 chạy nội dung paste theo từng dòng, phải gom thành một script block thì `throw` mới dừng được toàn bộ. Chạy lại block là an toàn: file đích đã có và trùng hash thì PASS ngay, lệch hash (file dở của lần `scp` lỗi) thì được chỉ dẫn xóa trước khi thử lại:

```powershell
& {
  # Lấy hai giá trị từ output bước 1.
  $Stamp          = '<STAMP>'
  $ExpectedSha256 = '<SHA256_CUA_FILE_TAR_GZ>'.ToLower()
  $Dest           = "$env:USERPROFILE\k8s-backups"
  $Target         = Join-Path $Dest "$Stamp.tar.gz"

  if ($Stamp -eq '<STAMP>' -or $ExpectedSha256 -notmatch '^[0-9a-f]{64}$') {
    throw 'FAIL: chưa thay STAMP/SHA-256 thật từ bước 1'
  }

  Get-Command scp -ErrorAction Stop | Out-Null
  New-Item -ItemType Directory -Force $Dest | Out-Null

  # File đích đã tồn tại: trùng hash → lần copy trước đã PASS, không copy lại (idempotent);
  # lệch hash → file dở của lần scp lỗi, xóa đúng file đó rồi chạy lại block.
  if (Test-Path -LiteralPath $Target) {
    $Existing = (Get-FileHash -LiteralPath $Target -Algorithm SHA256).Hash.ToLower()
    if ($Existing -eq $ExpectedSha256) {
      Write-Host "PASS: backup ngoài VM hợp lệ tại $Target (SHA-256 $Existing) — đã copy từ trước"
      return
    }
    throw "FAIL: $Target đã tồn tại nhưng hash lệch (actual=$Existing) — file dở của lần copy lỗi, xóa nó rồi chạy lại block"
  }

  scp "ubuntu@192.168.100.111:k8s-backups/$Stamp.tar.gz" $Dest
  if ($LASTEXITCODE -ne 0) { throw 'FAIL: scp lỗi — chưa có bản copy ngoài VM, KHÔNG sang §14.1' }

  $ActualSha256 = (Get-FileHash -LiteralPath $Target -Algorithm SHA256).Hash.ToLower()
  if ($ActualSha256 -ne $ExpectedSha256) {
    throw "FAIL: checksum lệch — expected=$ExpectedSha256 actual=$ActualSha256"
  }

  Write-Host "PASS: backup ngoài VM hợp lệ tại $Target (SHA-256 $ActualSha256)"
}
```

> **Bắt buộc:** giữ ít nhất một bản backup ngoài VM/disk chứa etcd. Bước 2 dùng máy host Windows; NAS hay storage backup khác cũng đạt, miễn checksum tại đích khớp hash ở bước 1. Snapshot nằm cùng disk không giúp được khi disk VM hỏng. Việc restore etcd là thao tác sự cố riêng; làm theo tài liệu kubeadm/etcd của đúng phiên bản và dừng control plane trước khi restore.

**PASS §14.0:** gate read-only đạt đủ điều kiện trên **và** bước 1 kết thúc bằng dòng `PASS: backup <STAMP> hợp lệ` với `$? = 0`, **và** bước 2 kết thúc bằng dòng `PASS: backup ngoài VM hợp lệ...`. Chỉ khi đó mới sang §14.1.

### 14.1. Cài cert-manager (Rancher cần để cấp TLS nội bộ)

cert-manager `1.21` hỗ trợ và được test với Kubernetes `1.33–1.36`. Pin patch `v1.21.1` vì đây là patch hiện hành trong cùng minor và sửa các regression của `v1.21.0`. Dùng OCI chart chính thức — đây là nguồn được cert-manager khuyến nghị cho các bản mới:

```bash
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.21.1 \
  --set crds.enabled=true \
  --wait --timeout 10m

helm status cert-manager -n cert-manager
kubectl wait --for=condition=Available deployment --all \
  -n cert-manager --timeout=300s
kubectl get pods -n cert-manager
kubectl get crd | grep cert-manager.io
```

**PASS §14.1:** Helm release `deployed`, ba Deployment/Pod `cert-manager`, `cert-manager-cainjector`, `cert-manager-webhook` đều Ready và các CRD `cert-manager.io` tồn tại. STOP nếu webhook chưa Ready; Rancher chưa thể tạo certificate an toàn.

### 14.2. Cấu hình split DNS cho Rancher agent

Agent trong cluster phải gọi Server URL `https://rancher.hieupn.site`. Cho hostname này phân giải **nội bộ** thẳng tới ClusterIP Traefik để agent không đi vòng ra Cloudflare và không bị Cloudflare Access chặn:

```bash
TRAEFIK_IP=$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.clusterIP}')
printf 'Traefik ClusterIP: %s\n' "$TRAEFIK_IP"
kubectl -n kube-system get configmap coredns -o yaml > coredns-before-rancher.yaml
kubectl -n kube-system edit configmap coredns
```

Trong `Corefile`, thêm khối sau **bên trong** server block `.:53`, trước `forward . /etc/resolv.conf`; thay `<TRAEFIK_CLUSTER_IP>` bằng giá trị vừa in:

```text
hosts {
    <TRAEFIK_CLUSTER_IP> rancher.hieupn.site
    ttl 60
    fallthrough
}
```

Plugin `hosts` chỉ được xuất hiện **một lần trong mỗi server block**. Nếu Corefile đã có khối `hosts`, chỉ thêm dòng `<TRAEFIK_CLUSTER_IP> rancher.hieupn.site` và `ttl 60` vào khối hiện hữu, không tạo khối thứ hai.

Áp dụng và kiểm tra từ một Pod dùng cluster DNS:

```bash
kubectl -n kube-system rollout restart deployment coredns
kubectl -n kube-system rollout status deployment coredns
kubectl -n kube-system logs deployment/coredns --tail=50

kubectl run dns-check --rm -i --restart=Never --image=busybox:1.36 \
  -- nslookup rancher.hieupn.site
# PASS: trả đúng ClusterIP Traefik, không phải IP Cloudflare
```

**PASS §14.2:** CoreDNS rollout thành công, log không có lỗi parse/reload và lookup trong Pod trả đúng `$TRAEFIK_IP`. Giữ `coredns-before-rancher.yaml` để rollback bằng `kubectl apply -f coredns-before-rancher.yaml` rồi rollout lại CoreDNS. Nếu Service Traefik bị xoá/tạo lại và đổi ClusterIP, phải cập nhật entry này.

### 14.3. Cài Rancher (Helm, pin 2.14.3)

Thêm repo stable rồi xác nhận chart pin tồn tại trước khi render hoặc cài:

```bash
helm repo add rancher-stable \
  https://releases.rancher.com/server-charts/stable --force-update
helm repo update rancher-stable
helm show chart rancher-stable/rancher --version 2.14.3
# PASS: version: 2.14.3, appVersion: v2.14.3, kubeVersion: < 1.36.0-0
```

Tạo values file không chứa mật khẩu. Không đặt `bootstrapPassword` trực tiếp trên command line để tránh ghi literal vào shell history. Khi giá trị này rỗng và Secret chưa tồn tại, template Helm **không render** `bootstrap-secret`; Rancher Server sẽ sinh mật khẩu ngẫu nhiên trong lần khởi động đầu tiên rồi ghi Secret đó lúc runtime. Vì vậy chỉ đọc mật khẩu sau khi Deployment Rancher Ready và Secret đã được tạo như §14.6.

```bash
cat > rancher-values.yaml <<'EOF'
hostname: rancher.hieupn.site
replicas: 1
agentTLSMode: strict
networkExposure:
  type: ingress
ingress:
  enabled: true
  includeDefaultExtraAnnotations: false
  ingressClassName: traefik
  servicePort: 80
  tls:
    source: rancher
    secretName: tls-rancher-ingress
EOF
```

Render read-only để xác nhận chart tạo đúng Ingress, hostname, IngressClass và tên TLS Secret, đồng thời không đi vào nhánh Gateway:

```bash
(
  set -e

  helm template rancher rancher-stable/rancher \
    --namespace cattle-system \
    --version 2.14.3 \
    -f rancher-values.yaml \
    > rancher-rendered.yaml

  grep -q '^kind: Ingress$' rancher-rendered.yaml || {
    echo 'FAIL: chart không render Ingress' >&2
    exit 1
  }
  for expected in \
    'ingressClassName: traefik' \
    'host: rancher.hieupn.site' \
    'secretName: tls-rancher-ingress'; do
    grep -Fq "$expected" rancher-rendered.yaml || {
      echo "FAIL: manifest thiếu $expected" >&2
      exit 1
    }
  done
  if grep -Eq '^kind: (Gateway|HTTPRoute)$' rancher-rendered.yaml; then
    echo 'FAIL: chart đã đi vào nhánh Gateway API' >&2
    exit 1
  fi
)
RENDER_RC=$?

if [ "$RENDER_RC" -eq 0 ]; then
  echo 'PASS: Ingress mode, class/host/TLS Secret đúng, không có Gateway/HTTPRoute'
else
  echo 'FAIL: render gate chưa đạt — KHÔNG cài Rancher'
fi

# Trả đúng mã lỗi nhưng không đóng phiên SSH.
( exit "$RENDER_RC" )
```

Cài khi render gate đã PASS:

```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system --create-namespace \
  --version 2.14.3 \
  -f rancher-values.yaml \
  --wait --timeout 15m

helm status rancher -n cattle-system
kubectl -n cattle-system rollout status deploy/rancher
kubectl -n cattle-system get pods,svc,ingress
```

- `--version 2.14.3`: ghim đúng chart/app `2.14.3`; không vô tình lấy patch mới hơn từ stable.
- `replicas: 1`: phù hợp mức tài nguyên homelab nhưng **không HA**; chart mặc định là 3 replica.
- `networkExposure.type: ingress`: đây là **mode selector cấp cao nhất** của chart `2.14.3`, không chỉ là thông tin mô tả. Helper `rancher.ingressEnabled` chỉ render Ingress khi đồng thời `networkExposure.type == "ingress"` và `ingress.enabled == true`. Chart hiện mặc định `networkExposure.type: ingress`, nên bỏ key này vẫn có thể chạy, nhưng đó là phụ thuộc ngầm vào default mới của dòng 2.14; runbook ghim rõ cả hai key để một lần đổi default/values không âm thầm làm biến mất Ingress.
- Không đổi `networkExposure.type` thành `gateway` trong lab này. Nhánh đó render tài nguyên Gateway API và mặc định tham chiếu `gateway.gatewayClass.name: traefik`; tên mặc định này **không cài** Gateway API CRD hay tự tạo một GatewayClass/controller tương thích. §9 chỉ bật Traefik Kubernetes Ingress/CRD provider, không cài Gateway API CRD, nên bật nhầm nhánh Gateway có thể làm Helm lỗi vì không nhận ra resource hoặc để Rancher không có route hoạt động.
- `ingress.enabled: true`: ghim vế thứ hai của điều kiện render Ingress; `ingress.servicePort: 80` và `ingress.tls.secretName: tls-rancher-ingress` ghim đúng backend/tên Secret mà các gate bên dưới kiểm tra.
- `ingress.includeDefaultExtraAnnotations: false`: không render bộ annotation dành cho ingress-nginx. Traefik provider chuẩn của §9 không diễn giải các annotation NGINX; không bật chúng để cố sửa timeout.
- `ingress.ingressClassName: traefik`: ghi rõ `spec.ingressClassName=traefik`, không phụ thuộc default IngressClass. Chart `2.14.3` thực tế để giá trị này trống mặc định; nó không mặc định nhắm `ingress-nginx`.
- `ingress.tls.source: rancher`: Rancher tạo CA/certificate self-signed qua cert-manager cho `rancher.hieupn.site`; Traefik terminate TLS ở cổng 443.
- `agentTLSMode: strict`: agent chỉ tin CA do Rancher công bố. Với certificate do chính chart Rancher sinh, không cần tự tạo `tls-ca` hay đặt `privateCA=true`; hai bước đó dành cho CA riêng do người vận hành cung cấp.

> **Traefik + Rancher UI:** Rancher dùng **WebSocket** rất nhiều (shell, log, cluster events). Traefik hỗ trợ WebSocket sẵn, không cần annotation gì thêm. Nó cũng tự set `X-Forwarded-Proto` — thứ mà Rancher cần để không rơi vào redirect-loop.
>
> Chart Rancher chỉ thêm `nginx.ingress.kubernetes.io/proxy-{connect,read,send}-timeout` khi bật `ingress.includeDefaultExtraAnnotations`; runbook chủ động pin `false`. Kể cả render chúng, Traefik `kubernetesIngress` chuẩn ở §9 cũng bỏ qua vì chúng thuộc NGINX; chỉ provider tương thích `kubernetesIngressNGINX` mới dịch các annotation này. Nếu Shell/Logs ngắt khi idle, kiểm tra từng hop Cloudflare và timeout `transport.respondingTimeouts` của entrypoint Traefik; không kết luận nguyên nhân hoặc thêm annotation NGINX chỉ từ hiện tượng rớt WebSocket.

> **Không trộn hai mô hình TLS:** runbook này dùng `tls=ingress` mặc định + `ingress.tls.source=rancher`. Chỉ dùng `tls=external` khi chủ động chuyển sang termination bên ngoài và đã cấu hình đủ `Host`, `X-Forwarded-Proto`, `X-Forwarded-Port`, `X-Forwarded-For` cùng timeout WebSocket dài.

### 14.4. Gate origin HTTPS nội bộ trước khi publish

Xác nhận cert-manager đã phát certificate, Ingress mang đúng host/class và Traefik phục vụ health endpoint. Chưa tạo route Cloudflare nếu bất kỳ lệnh nào thất bại:

```bash
kubectl -n cattle-system wait --for=create \
  certificate/tls-rancher-ingress --timeout=120s
kubectl -n cattle-system wait --for=condition=Ready \
  certificate/tls-rancher-ingress --timeout=300s
kubectl -n cattle-system get issuer,certificate
kubectl -n cattle-system get secret tls-rancher-ingress
kubectl -n cattle-system get ingress rancher \
  -o jsonpath='{.spec.ingressClassName}{"\t"}{.spec.rules[0].host}{"\n"}'
# PASS: traefik    rancher.hieupn.site

TRAEFIK_IP=$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.clusterIP}')
curl -skS --resolve "rancher.hieupn.site:443:$TRAEFIK_IP" \
  -o /dev/null -w 'HTTP %{http_code}\n' \
  https://rancher.hieupn.site/healthz
# PASS: HTTP 200
```

Trong mode Ingress, chart Rancher render `Issuer` và Ingress có annotation `cert-manager.io/issuer`; chart không trực tiếp render Certificate cho nhánh này. Controller `ingress-shim` của cert-manager quan sát Ingress rồi tạo Certificate theo `spec.tls[].secretName`, nên tên đã pin là `tls-rancher-ingress`. Đây là resource sinh bất đồng bộ bởi controller khác và `helm --wait` không chờ nó. Không dùng `kubectl wait certificate --all`: nếu Certificate chưa xuất hiện, lệnh đó thoát ngay với `no matching resources found` thay vì đợi. Baseline `kubectl 1.35.6` hỗ trợ `--for=create`.

`--resolve` buộc kết nối tới ClusterIP Traefik nhưng vẫn gửi SNI/Host `rancher.hieupn.site`. `-k` chỉ dùng ở gate nội bộ vì certificate được CA riêng của Rancher ký, chưa nằm trong trust store của máy master.

**PASS §14.4:** certificate `Ready=True`, Secret TLS tồn tại, Ingress in `traefik rancher.hieupn.site` và `/healthz` trả `HTTP 200`.

### 14.5. Bảo vệ Rancher bằng Cloudflare Access rồi publish qua tunnel

Rancher là giao diện quản trị cụm, không nên chỉ dựa vào màn hình đăng nhập Rancher. Tạo lớp xác thực Cloudflare Access **trước khi** publish:

1. Cloudflare dashboard → **Zero Trust → Access controls → Applications**.
2. **Create new application → Self-hosted and private → Add public hostname**.
3. Hostname: `rancher.hieupn.site`.
4. Tạo policy **Allow** chỉ cho email/email domain quản trị; bật MFA nếu IdP hỗ trợ. Không tạo policy `Bypass Everyone`.
5. Save. Access mặc định deny người không khớp policy.

Sau đó thêm một **Published application route** theo [§12.3.3](#1233-thêm-published-application):

| Trường                                              | Giá trị                                                                                                                                                                                                  |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Subdomain**                                   | `rancher`                                                                                                                                                                                                |
| **Domain**                                      | `hieupn.site`                                                                                                                                                                                            |
| **Service URL**                                 | `https://traefik.traefik.svc.cluster.local:443` (UI cũ: `Type` = **HTTPS**, `URL` = `traefik.traefik.svc.cluster.local:443` — xem lưu ý ở [§12.3.3](#1233-thêm-published-application)) |
| **Additional settings → TLS → No TLS Verify** | **ON** (vì cert Rancher là self-signed)                                                                                                                                                            |
| **Additional settings → TLS → Origin Server Name** | `rancher.hieupn.site` (đặt đúng SNI về Traefik)                                                                                                                                                     |
| **Additional settings → HTTP Host Header**     | `rancher.hieupn.site`                                                                                                                                                                                    |

→ **Save**. Với zone Full Setup, Cloudflare tự tạo DNS record cho `rancher.hieupn.site`; với Partial/CNAME Setup, tạo record tại DNS provider như lưu ý ở [§12.3.3](#1233-thêm-published-application).

> `Origin Server Name` đặt SNI trong TLS handshake; `HTTP Host Header` được Traefik dùng để chọn HTTP router sau handshake. `No TLS Verify` cho phép `cloudflared` chấp nhận CA self-signed của Rancher nhưng cũng tắt xác minh danh tính origin — chấp nhận được cho homelab này, không phải lựa chọn ưu tiên cho production.
>
> Cloudflare Access hỗ trợ WebSocket. Rancher Shell/Logs vẫn có thể ngắt nếu kết nối WebSocket idle quá lâu; reconnect trước khi kết luận cụm hỏng.

Kiểm tra lớp bảo vệ trước khi đăng nhập:

```bash
nslookup rancher.hieupn.site
curl -I https://rancher.hieupn.site
```

Public DNS phải trả IP Cloudflare. Khi chưa có Access session, `curl -I` phải bị chuyển tới trang đăng nhập Access hoặc bị từ chối (`302`, `401` hay `403` tùy policy), **không** được trả thẳng Rancher UI `200`.

### 14.6. Đăng nhập lần đầu

```bash
# Chạy trong terminal riêng tư; đây là secret dùng một lần và phải đổi ngay.
kubectl -n cattle-system rollout status deployment/rancher --timeout=300s
kubectl -n cattle-system wait --for=create secret/bootstrap-secret --timeout=120s
kubectl get secret --namespace cattle-system bootstrap-secret \
  -o go-template='{{ .data.bootstrapPassword | base64decode }}{{ "\n" }}'
```

Nếu đọc Secret trước khi Rancher khởi tạo xong, `NotFound` chỉ có nghĩa resource runtime chưa được tạo; không phải bằng chứng Helm install hỏng. Chỉ điều tra cài đặt khi Deployment không Ready, timeout chờ Secret hoặc log Rancher có lỗi.

1. Mở `https://rancher.hieupn.site` → đăng nhập Cloudflare Access, sau đó đăng nhập Rancher bằng bootstrap password.
2. Đặt mật khẩu admin mới.
3. Xác nhận **Server URL** = `https://rancher.hieupn.site` (Rancher gợi ý sẵn). Không đổi hostname sau khi agent đã đăng ký nếu chưa có kế hoạch migration.
4. Vào **Cluster Management** → thấy cụm **`local`** = chính cụm kubeadm của bạn, trạng thái **Active**.

### 14.7. Gate hoàn thành

- UI hiện cụm `local` Active, đủ 3 node.
- Quản lý workload / Helm app / monitoring qua UI; cụm vẫn dùng song song bằng `kubectl`.

Kiểm tra agent và đường nội bộ:

```bash
kubectl -n cattle-system get pods
kubectl -n cattle-system get deploy
kubectl -n cattle-system get pods -l app=cattle-cluster-agent -o wide
# Chỉ chạy sau khi output trên xác nhận Pod agent thực sự tồn tại:
kubectl -n cattle-system logs -l app=cattle-cluster-agent --tail=30 --prefix
kubectl get settings.management.cattle.io server-url \
  -o jsonpath='{.value}{"\n"}'
kubectl get settings.management.cattle.io agent-tls-mode \
  -o jsonpath='{.value}{"\n"}'
```

`cattle-cluster-agent` không nằm trong Helm chart Rancher; Rancher tạo nó lúc runtime khi quản lý cluster. Vì vậy phải xem inventory Deployment/Pod thật trước, không hard-code `logs deploy/cattle-cluster-agent` như một gate độc lập. Nếu agent tồn tại, nó phải Ready và log không có lỗi DNS/TLS. Nếu không thấy agent nhưng UI `local` đã Active, ghi nhận đúng inventory thay vì kết luận Helm install thất bại chỉ từ tên Deployment giả định; nếu UI chưa Active thì kiểm tra log `deploy/rancher` và chờ controller reconcile.

**PASS §14:** Pod Rancher Ready; agent runtime nếu tồn tại thì Ready và log không có lỗi DNS/TLS; `server-url` là `https://rancher.hieupn.site`; `agent-tls-mode` là `strict`; UI `local` Active đủ 3 node; truy cập public bắt buộc đi qua Cloudflare Access.

> ⚠️ Chart Rancher 2.14.3 chặn Kubernetes `1.36`. Không nâng cluster lên `1.36` cho tới khi support matrix và `kubeVersion` của **chart Rancher sẽ nâng tới** đều cho phép; luôn nâng Rancher trước Kubernetes theo vùng version-skew chung.

---

## 15. Vận hành & troubleshooting

### Thêm app/domain mới sau này

Chỉ cần: (1) `Deployment`+`Service`+`Ingress` mới với `host: app2.example.com`; (2) thêm **Published application route** mới trong cùng tunnel trỏ về cùng `traefik.traefik.svc.cluster.local:80`. Không đụng gì tới VM/router.

### Vì sao phải tôn trọng EOL & upgrade đúng nhịp

Cụm **vẫn chạy** sau ngày EOL của k8s, nhưng dừng nhận **patch bảo mật (CVE)** + bug fix; kubeadm **không skip được minor** (phải nhảy từng bậc) nên để càng lâu càng khó nâng; và hệ sinh thái (CNI/ingress/cloudflared/**Rancher**) dần bỏ hỗ trợ bản cũ. Với cụm **phơi ra Internet qua tunnel**, rủi ro CVE chưa vá là nghiêm trọng nhất → nâng đều, luôn nằm trong vùng Rancher support.

### Backup etcd và cấu hình trước thay đổi lớn

Quy trình đầy đủ ở [§14.0.1](#1401-backup-etcd-và-cấu-hình-trước-thay-đổi-lớn) — nó nằm trong luồng cài đặt vì lần chạy bắt buộc đầu tiên là ngay trước khi cài cert-manager/Rancher. Chạy lại đúng quy trình đó, kể cả bước copy backup ra ngoài VM, trước mọi upgrade hoặc thay đổi quan trọng.

### Nâng patch Kubernetes 1.35 bằng kubeadm

Ví dụ nâng `1.35.6` lên một patch `1.35.X` mới hơn. Đọc release notes, chạy [backup theo §14.0.1](#1401-backup-etcd-và-cấu-hình-trước-thay-đổi-lớn) trước, và thay đúng hai biến sau bằng version có thật từ `apt-cache madison kubeadm`:

```bash
TARGET_DEB='1.35.X-1.1'
TARGET_K8S='v1.35.X'
```

Trên **control plane**:

```bash
sudo apt-mark unhold kubeadm
sudo apt-get update
apt-cache madison kubeadm
sudo apt-get install -y kubeadm="$TARGET_DEB"
sudo apt-mark hold kubeadm

sudo kubeadm upgrade plan
sudo kubeadm upgrade apply "$TARGET_K8S"

kubectl drain k8s-master --ignore-daemonsets
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet="$TARGET_DEB" kubectl="$TARGET_DEB"
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon k8s-master
```

Nâng **từng worker một**. Từ master, drain worker; trên chính worker nâng package và node config; quay lại master để uncordon:

```bash
# trên master:
kubectl drain k8s-worker1 --ignore-daemonsets

# trên k8s-worker1:
TARGET_DEB='1.35.X-1.1'    # đặt lại vì đây là phiên SSH khác
sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt-get update
sudo apt-get install -y kubeadm="$TARGET_DEB"
sudo kubeadm upgrade node
sudo apt-get install -y kubelet="$TARGET_DEB" kubectl="$TARGET_DEB"
sudo apt-mark hold kubeadm kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# trở lại master:
kubectl uncordon k8s-worker1
kubectl get nodes
```

Lặp lại cho `k8s-worker2`, sau đó:

```bash
kubectl get nodes -o wide
kubectl get pods -A
sudo kubeadm certs check-expiration
```

> Với nâng **minor**, không được skip minor; phải đổi repo `pkgs.k8s.io` sang minor đích, nâng từng bậc và làm theo đúng trang upgrade của phiên bản nguồn. Luôn kiểm support matrix Rancher **trước**; Rancher 2.14.3 chưa cho phép nâng cụm này vượt 1.35.

### Quản lý và gia hạn certificate kubeadm

Các certificate control plane do kubeadm cấp mặc định có hạn khoảng **1 năm**; CA có hạn dài hơn và
không được `kubeadm certs renew all` xoay vòng. Không chờ certificate hết hạn mới xử lý.

**Cơ chế chính (phổ biến):** nâng Kubernetes bằng kubeadm định kỳ, khoảng cách giữa hai lần nâng
control plane phải **dưới 1 năm**. `kubeadm upgrade apply` mặc định tự gia hạn các certificate do
kubeadm quản lý trên control-plane node. Không thêm `--certificate-renewal=false`.

**Gate vận hành:** chạy trên master mỗi tháng và trước mọi đợt bảo trì:

```bash
sudo kubeadm certs check-expiration
```

- Còn **trên 60 ngày**: tiếp tục theo dõi hoặc gia hạn qua đợt upgrade định kỳ.
- Còn **60 ngày trở xuống** và đã sẵn sàng upgrade: backup rồi làm quy trình upgrade ở trên.
- Còn **60 ngày trở xuống** nhưng chưa thể upgrade: dùng manual renew bên dưới; không trì hoãn tới
  ngày hết hạn.

#### Manual renew dự phòng (single control plane)

Quy trình này làm API tạm gián đoạn vì runbook chỉ có một control plane. Thực hiện trong maintenance
window trên `k8s-master`; backup etcd và `/etc/kubernetes` theo
[§14.0.1](#1401-backup-etcd-và-cấu-hình-trước-thay-đổi-lớn) trước khi tiếp tục.

Gia hạn và xác nhận hạn mới:

```bash
sudo kubeadm certs check-expiration
sudo kubeadm certs renew all
sudo kubeadm certs check-expiration
```

Renew không tự reload certificate cho mọi control-plane component. Restart từng static Pod tuần tự;
thư mục tạm phải nằm **ngoài** `/etc/kubernetes/manifests`, nếu không kubelet vẫn đọc file tạm như
một manifest:

```bash
STAMP=$(date +%Y%m%d-%H%M%S)
MANIFEST_HOLD="$HOME/k8s-maintenance/static-pods-$STAMP"
mkdir -p "$MANIFEST_HOLD"

(
  for component in etcd kube-apiserver kube-controller-manager kube-scheduler; do
    OLD_ID="$(sudo crictl ps --name "$component" -q)"
    sudo mv "/etc/kubernetes/manifests/${component}.yaml" "$MANIFEST_HOLD/"
    sleep 25
    sudo mv "$MANIFEST_HOLD/${component}.yaml" /etc/kubernetes/manifests/

    NEW_ID=""
    for attempt in {1..12}; do
      NEW_ID="$(sudo crictl ps --name "$component" -q)"
      if [ -n "$NEW_ID" ] && [ "$NEW_ID" != "$OLD_ID" ]; then
        break
      fi
      sleep 5
    done

    test -n "$NEW_ID" && test "$NEW_ID" != "$OLD_ID" || {
      echo "FAIL: $component chưa được tạo lại; dừng để kiểm tra kubelet/crictl" >&2
      exit 1
    }
    echo "PASS: $component restarted ($NEW_ID)"
  done
)
STATIC_POD_RC=$?

if [ "$STATIC_POD_RC" -eq 0 ]; then
  echo 'PASS: toàn bộ static Pod đã được tạo lại tuần tự'
else
  echo 'FAIL: restart static Pod chưa hoàn tất — KHÔNG chạy bước kế tiếp'
fi

# Trả đúng mã lỗi nhưng không đóng phiên SSH.
( exit "$STATIC_POD_RC" )
```

`admin.conf` đã được renew nhưng user `ubuntu` đang dùng một bản copy trong `$HOME/.kube/config`;
chép lại bản mới rồi verify:

```bash
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"

kubectl get --raw='/readyz?verbose'
kubectl get nodes
kubectl get pods -A
sudo kubeadm certs check-expiration
```

PASS khi `readyz` kết thúc bằng `readyz check passed`, mọi node `Ready`, các system pod `Running`,
và certificate có `RESIDUAL TIME` mới gần 1 năm. Nếu certificate đã hết hạn thì vẫn thực hiện quy
trình này trực tiếp trên master bằng `sudo`; **không** chạy lại `kubeadm init` và không
`kubeadm reset`.

> Với nhiều control-plane node, phải renew và restart tuần tự trên **từng** control-plane node. Kubeadm
> không hỗ trợ CA rotation/replacement tự động; đó là một quy trình PKI riêng, không thay bằng
> `kubeadm certs renew all`. Tham khảo
> [Certificate Management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)
> và [`kubeadm certs`](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-certs/).

### Giới hạn Cloudflare cần biết

- Upload qua hostname proxied bị giới hạn theo plan: Free/Pro 100 MB, Business 200 MB, Enterprise 500+ MB. Upload chart/backup lớn qua Rancher UI có thể nhận HTTP 413 dù cluster khỏe.
- Cloudflare hỗ trợ WebSocket, nhưng Rancher Shell/log stream để lâu vẫn có thể bị đóng ở một hop trên đường đi (Cloudflare, Traefik hoặc backend). Reconnect rồi xác định đúng hop; dùng keepalive/timeout phù hợp và không dùng annotation NGINX để chỉnh Traefik `kubernetesIngress` chuẩn.
- Tunnel cần outbound TCP/UDP `7844`; nếu UDP bị chặn cloudflared có thể fallback HTTP/2 qua TCP. Nếu cả hai bị chặn, tunnel không lên.

### Bảng lỗi thường gặp

| Triệu chứng                                            | Nguyên nhân & cách xử lý                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| -------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cài Ubuntu:**DHCPv4 quay mãi, không ra IP**     | VMnet0 bridge bind nhầm card →**Edit → Virtual Network Editor → VMnet0**, đổi *Automatic* sang đúng card vật lý ([§3](#3-tạo-3-vm-ubuntu-2404-trên-vmware)). Xác nhận card đang chạy bằng `ipconfig /all` trên host. Tạm thời có thể đặt IP tĩnh ngay tại màn hình installer.                                                                                                                                                                                                                      |
| Bridge**vẫn** không ra IP dù chọn đúng card  | Host bật**Hyper-V** làm VMware bridge xung đột (danh sách "Bridged to" có "Hyper-V Virtual Ethernet Adapter"). Kiểm tra host: `bcdedit /enum \| findstr -i hypervisorlaunchtype`. Xử lý: tắt Hyper-V cho lab (`bcdedit /set hypervisorlaunchtype off` + reboot — **lưu ý sẽ tắt WSL2/Docker Desktop/Sandbox**), **hoặc** chuyển Network Adapter sang **NAT** (vẫn ra Internet; khi đó IP tĩnh phải theo dải NAT của VMnet8, vd `192.168.71.x`, và đổi đồng bộ toàn runbook). |
| Node`NotReady`                                         | CNI chưa chạy →`kubectl get pods -n kube-flannel`; kiểm tra UDP 8472 mở; `--pod-network-cidr` đúng `10.244.0.0/16`                                                                                                                                                                                                                                                                                                                                                                                                          |
| `kubeadm join` timeout                                 | Worker không resolve`k8s-master` (thiếu `/etc/hosts`); hoặc cổng `6443` master bị firewall chặn                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Pod kẹt`ContainerCreating`/CNI lỗi                   | sai cgroup driver →`sudo crictl info \| grep -i -A2 systemdCgroup` phải `true`, rồi `systemctl restart containerd kubelet`                                                                                                                                                                                                                                                                                                                                                                                                       |
| `kubelet` crashloop sau init                           | còn swap →`swapoff -a` + check `/etc/fstab`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Tunnel không**Healthy**                           | token sai/thiếu →`kubectl logs -n cloudflare deploy/cloudflared`; pod phải `Running`                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Domain mở ra**502/error 1033**                    | URL route sai tên service; thử đúng`traefik.traefik.svc.cluster.local:80` (hoặc `:443` cho Rancher); kiểm tra `kubectl get svc -n traefik`                                                                                                                                                                                                                                                                                                                                                                                   |
| Mở domain ra**404 page not found**                | Host header không khớp Ingress`host:` → set HTTP Host Header trong tunnel, hoặc sửa `host:` trong Ingress. Soi Router thực tế ở dashboard Traefik ([§9.3](#93-cài-đặt-traefik))                                                                                                                                                                                                                                                                                                                                           |
| Ingress apply xong nhưng**không được route**  | Thiếu/sai`ingressClassName` → phải là `traefik`; kiểm tra `kubectl get ingress -A` cột CLASS và `kubectl get ingressclass`                                                                                                                                                                                                                                                                                                                                                                                                |
| **UI Rancher 404** dù pod Running                 | `Ingress` không có/sai class hoặc host → kiểm tra `spec.ingressClassName=traefik` và `rancher.hieupn.site`; chart để class trống mặc định nên runbook đặt rõ trong [§14.3](#143-cài-rancher-helm-pin-2143)                                                                                                                                                                                                                                                                                                           |
| `rancher.hieupn.site` lỗi **TLS/redirect-loop** | kiểm certificate Ready; route HTTPS phải có **Origin Server Name** + **HTTP Host Header**=`rancher.hieupn.site` và **No TLS Verify=ON** cho CA self-signed của homelab                                                                                                                                                                                                                                                                                                                                     |
| Rancher pod`CrashLoop`/Pending                         | thiếu RAM/headroom; hoặc cert-manager CRDs chưa cài (`--set crds.enabled=true`)                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| Rancher`cattle-cluster-agent` CrashLoop/không connect | kiểm `rancher.hieupn.site` trong Pod có resolve về ClusterIP Traefik theo split DNS ở §14.2 và `agent-tls-mode=strict`; nếu đi ra Cloudflare Access sẽ bị chặn                                                                                                                                                                                                                                                                                                                                              |
| Rancher Shell/exec/attach/port-forward bị`forbidden`  | Kubernetes 1.35 WebSocket upgrade cần verb RBAC`create` trên `pods/exec`, `pods/attach`, `pods/portforward`; kiểm Role/ClusterRole của user trước khi nghi firewall                                                                                                                                                                                                                                                                                                                                                        |
| Rancher Shell/log stream rớt sau khi idle               | Reconnect rồi xác định hop đóng kết nối: Cloudflare, Traefik entrypoint `transport.respondingTimeouts` hay backend; annotation `nginx.ingress.kubernetes.io/proxy-*-timeout` không có tác dụng với Traefik `kubernetesIngress` chuẩn ở §9; phân biệt thêm với lỗi RBAC/10250                                                                                                                                                                                                                                                                |
| Pull image private đã cache vẫn lỗi credential       | Kubernetes 1.35 bật beta`KubeletEnsureSecretPulledImages`; kiểm `imagePullSecrets` hợp lệ thay vì dựa vào image đã cache                                                                                                                                                                                                                                                                                                                                                                                                    |
| `curl -H Host` nội bộ OK nhưng Internet lỗi        | vấn đề ở tunnel/DNS, không phải cụm → soi log cloudflared + trạng thái tunnel                                                                                                                                                                                                                                                                                                                                                                                                                                                  |

### Lệnh chẩn đoán nhanh

```bash
kubectl get nodes,pods -A -o wide
kubectl logs -n cloudflare deploy/cloudflared --tail=50
kubectl -n cattle-system rollout status deploy/rancher
kubectl describe ingress web
kubectl get events -A --sort-by=.lastTimestamp | tail -30
```

---

## 16. Nguồn official

- Kubernetes — *Creating a cluster with kubeadm*: [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- Kubernetes — *Installing kubeadm* (repo pkgs.k8s.io): [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- Kubernetes — *Container runtimes* (containerd, SystemdCgroup, sysctl): [https://kubernetes.io/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- Kubernetes — *Kubernetes 1.35 lifecycle/patch releases*: [https://kubernetes.io/releases/1.35/](https://kubernetes.io/releases/1.35/)
- Kubernetes — *Version skew policy*: [https://kubernetes.io/releases/version-skew-policy/](https://kubernetes.io/releases/version-skew-policy/)
- Kubernetes — *Ports and Protocols*: [https://kubernetes.io/docs/reference/networking/ports-and-protocols/](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)
- Kubernetes — *Upgrading kubeadm clusters*: [https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- Kubernetes — *pkgs.k8s.io* (repo mới): [https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/)
- Ubuntu — *containerd package for Noble*: [https://packages.ubuntu.com/noble-updates/containerd](https://packages.ubuntu.com/noble-updates/containerd)
- Ubuntu — *Configuring networks with Netplan*: [https://ubuntu.com/server/docs/explanation/networking/configuring-networks/](https://ubuntu.com/server/docs/explanation/networking/configuring-networks/)
- Traefik — *Setup on Kubernetes*: [https://doc.traefik.io/traefik/setup/kubernetes/](https://doc.traefik.io/traefik/setup/kubernetes/)
- Traefik — *Helm chart 41.0.2 release*: [https://github.com/traefik/traefik-helm-chart/releases/tag/v41.0.2](https://github.com/traefik/traefik-helm-chart/releases/tag/v41.0.2)
- Traefik — *Chart 41.0.2 metadata/values*: [Chart.yaml](https://raw.githubusercontent.com/traefik/traefik-helm-chart/v41.0.2/traefik/Chart.yaml), [values.yaml](https://raw.githubusercontent.com/traefik/traefik-helm-chart/v41.0.2/traefik/values.yaml)
- Kubernetes — *Service, Service types và EndpointSlice*: [https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/)
- Kubernetes — *Ingress và IngressClass*: [https://kubernetes.io/docs/concepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- Kubernetes — *Ingress NGINX Retirement: What You Need to Know* (11/2025): [https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- Kubernetes — *Statement from the Steering and Security Response Committees* (01/2026): [https://kubernetes.io/blog/2026/01/29/ingress-nginx-statement/](https://kubernetes.io/blog/2026/01/29/ingress-nginx-statement/)
- Rancher — *Guide to Ingress NGINX Retirement* (migration sang Traefik): [https://ranchermanager.docs.rancher.com/v2.14/how-to-guides/new-user-guides/kubernetes-resources-setup/load-balancer-and-ingress-controller/guide-to-ingress-nginx-retirement](https://ranchermanager.docs.rancher.com/v2.14/how-to-guides/new-user-guides/kubernetes-resources-setup/load-balancer-and-ingress-controller/guide-to-ingress-nginx-retirement)
- Rancher — *local-path-provisioner stable install*: [https://github.com/rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner)
- MetalLB — *Installation*: [https://metallb.io/installation/](https://metallb.io/installation/)
- MetalLB — *Configuration và Usage*: [https://metallb.io/configuration/](https://metallb.io/configuration/), [https://metallb.io/usage/](https://metallb.io/usage/)
- Cloudflare — *Primary DNS / Full setup (add domain, review records, replace NS, verify Active)*: [https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/](https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/)
- Cloudflare — *Zone status (Pending vs Active)*: [https://developers.cloudflare.com/dns/zone-setups/reference/domain-status/](https://developers.cloudflare.com/dns/zone-setups/reference/domain-status/)
- Cloudflare — *1.1.1.1 Purge Cache tool* (xóa cache NS/DS cũ trên `1.1.1.1`): [https://one.one.one.one/purge-cache/](https://one.one.one.one/purge-cache/)
- Google Public DNS — *Flush Cache tool* (xóa cache NS/DS cũ trên `8.8.8.8`): [https://dns.google/cache](https://dns.google/cache)
- Cloudflare — *DNSSEC (disable before migration, enable and publish DS after Active)*: [https://developers.cloudflare.com/dns/dnssec/](https://developers.cloudflare.com/dns/dnssec/)
- Cloudflare — *Transfer domain to Cloudflare Registrar*: [https://developers.cloudflare.com/registrar/get-started/transfer-domain-to-cloudflare/](https://developers.cloudflare.com/registrar/get-started/transfer-domain-to-cloudflare/)
- Hostinger — *Point a domain to external services*: [https://support.hostinger.com/en/articles/4737652-how-to-point-a-domain-to-external-services](https://support.hostinger.com/en/articles/4737652-how-to-point-a-domain-to-external-services)
- Hostinger — *Manage DNS records / DNSSEC in hPanel*: [https://support.hostinger.com/en/articles/1583249-how-to-manage-dns-records-at-hostinger](https://support.hostinger.com/en/articles/1583249-how-to-manage-dns-records-at-hostinger)
- Hostinger — *How to use DNSSEC records at Hostinger* (bốn ô Key Tag/Algorithm/Digest Type/Digest, giá trị do provider giữ nameserver sinh ra): [https://www.hostinger.com/support/3667267-how-to-use-dnssec-records-at-hostinger/](https://www.hostinger.com/support/3667267-how-to-use-dnssec-records-at-hostinger/)
- Cloudflare — *Tunnel setup*: [https://developers.cloudflare.com/tunnel/setup/](https://developers.cloudflare.com/tunnel/setup/)
- Cloudflare — *Deploy cloudflared in Kubernetes*: [https://developers.cloudflare.com/tunnel/deployment-guides/kubernetes/](https://developers.cloudflare.com/tunnel/deployment-guides/kubernetes/)
- Cloudflare — *Tunnel with firewall*: [https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/tunnel-with-firewall/](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/tunnel-with-firewall/)
- Cloudflare — *Tunnel origin parameters (`originServerName`, `noTLSVerify`, `httpHostHeader`)*: [https://developers.cloudflare.com/tunnel/configuration/](https://developers.cloudflare.com/tunnel/configuration/)
- Cloudflare — *Self-hosted public application/Access*: [https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/)
- Cloudflare — *WebSockets*: [https://developers.cloudflare.com/network/websockets/](https://developers.cloudflare.com/network/websockets/)
- Cloudflare — *HTTP 413/upload limits*: [https://developers.cloudflare.com/support/troubleshooting/http-status-codes/4xx-client-error/error-413/](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/4xx-client-error/error-413/)
- Rancher/SUSE — *Support matrix v2.14.3*: [https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-14-3/](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-14-3/)
- Rancher — *Installation requirements*: [https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/installation-requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/installation-requirements)
- Rancher — *Helm version requirements*: [https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/helm-version-requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/helm-version-requirements)
- Rancher — *Install/upgrade on a Kubernetes cluster*: [https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster/](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster/)
- Rancher — *Helm chart options v2.14*: [https://ranchermanager.docs.rancher.com/v2.14/getting-started/installation-and-upgrade/installation-references/helm-chart-options](https://ranchermanager.docs.rancher.com/v2.14/getting-started/installation-and-upgrade/installation-references/helm-chart-options)
- cert-manager — *Supported releases*: [https://cert-manager.io/docs/releases/](https://cert-manager.io/docs/releases/)
- cert-manager — *Installation bằng OCI Helm chart*: [https://cert-manager.io/docs/installation/helm/](https://cert-manager.io/docs/installation/helm/)
- CoreDNS — *hosts plugin*: [https://coredns.io/plugins/hosts/](https://coredns.io/plugins/hosts/)

---

## 17. Phụ lục

### Phụ lục A — Locally-managed tunnel (CLI / config.yml)

Phương án thay thế cho [§12], chạy `cloudflared` **trực tiếp trên 1 máy** (vd master) thay vì bằng token trong cụm. Dùng khi muốn quản lý cấu hình tunnel bằng file thay vì dashboard.

```bash
# cài cloudflared (Ubuntu)
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg \
  | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" \
  | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt-get update && sudo apt-get install -y cloudflared

# 1) đăng nhập (mở trình duyệt, chọn domain) → lưu cert.pem vào ~/.cloudflared
cloudflared tunnel login

# 2) tạo tunnel → sinh credentials JSON ~/.cloudflared/<UUID>.json
cloudflared tunnel create homelab-k8s
cloudflared tunnel list                 # lấy <UUID>
```

Tạo `~/.cloudflared/config.yml`:

```yaml
tunnel: <UUID>
credentials-file: /home/<user>/.cloudflared/<UUID>.json
ingress:
  - hostname: app.example.com
    service: http://<node-ip>:<nodePort>   # xem ghi chú bên dưới
  - hostname: rancher.hieupn.site
    service: https://<node-ip>:<nodePort-https>
    originRequest:
      noTLSVerify: true
  - service: http_status:404          # catch-all BẮT BUỘC ở cuối
```

```bash
# tạo DNS record cho từng hostname
cloudflared tunnel route dns homelab-k8s app.example.com
cloudflared tunnel route dns homelab-k8s rancher.hieupn.site
# chạy thử
cloudflared tunnel run homelab-k8s
# chạy như service
sudo cloudflared service install
```

> ⚠️ Khi cloudflared chạy **ngoài** cụm, nó **không** resolve được `*.svc.cluster.local`. Khi đó phải expose ingress qua **NodePort** (`service: http://<node-ip>:<nodePort>`) hoặc **MetalLB IP** (Phụ lục B). Đây chính là lý do cách in-cluster ([§12]) gọn hơn — nên ưu tiên [§12].

### Phụ lục B — MetalLB (tuỳ chọn: LoadBalancer IP trong LAN)

Chỉ cần nếu bạn **cũng** muốn truy cập ingress từ LAN bằng một IP cố định. Cloudflare Tunnel in-cluster vẫn chỉ cần `ClusterIP`.

MetalLB gồm controller cấp IP cho Service `LoadBalancer` và speaker quảng bá IP đó ra mạng. Với L2 mode bên dưới, một node trả lời ARP cho IP dịch vụ; traffic tới node đó rồi được chuyển đến backend. Dải IP phải thuộc LAN, chưa được thiết bị nào dùng và nên nằm ngoài pool DHCP để tránh trùng IP.

```bash
# kubeadm mặc định dùng kube-proxy mode iptables: không cần bật strictARP.
# Chỉ bật strictARP nếu bạn đã chủ động chuyển kube-proxy sang IPVS.

# cài bản đã pin
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml
kubectl -n metallb-system rollout status deploy/controller --timeout=180s
kubectl -n metallb-system rollout status daemonset/speaker --timeout=180s
```

Sau đó cấp một dải IP **trong LAN nhưng ngoài DHCP** (vd `192.168.100.200-192.168.100.210`):

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lan-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.100.200-192.168.100.210
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: lan-l2
  namespace: metallb-system
spec:
  ipAddressPools: [lan-pool]
```

```bash
kubectl apply -f metallb-config.yaml

# §9 cài Traefik là ClusterIP; đổi sang LoadBalancer sau khi MetalLB đã sẵn sàng.
helm upgrade traefik traefik/traefik \
  -n traefik --version 41.0.2 --reuse-values \
  --set service.spec.type=LoadBalancer \
  --wait --timeout 5m

kubectl -n traefik get svc traefik
# PASS: TYPE=LoadBalancer và EXTERNAL-IP nhận một IP trong lan-pool.

kubectl -n traefik describe svc traefik
# PASS: Events không báo lỗi cấp/quảng bá IP; test curl IP đó từ một máy khác trong LAN.
```

### Phụ lục C — Reset / làm lại một node

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d $HOME/.kube/config
sudo systemctl restart containerd kubelet
# worker xong có thể join lại; master init lại từ §6
```

---

*Runbook tạo ngày 2026-06-26; cập nhật đồng bộ ngày 2026-08-09. Baseline: Ubuntu 24.04 amd64, Kubernetes **v1.35.6** (`1.35.6-1.1`), containerd **2.x** từ Ubuntu, Flannel **v0.28.7**, Traefik chart **41.0.2** / Proxy **v3.7.6**, cert-manager **v1.21.1**, Rancher **2.14.3**. Đây là homelab baseline, không phải SLA/certification production end-to-end cho kubeadm.*
