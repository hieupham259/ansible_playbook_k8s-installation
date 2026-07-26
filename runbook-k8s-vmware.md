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
11. [Mua &amp; cấu hình domain trên Cloudflare](#11-mua--cấu-hình-domain-trên-cloudflare)
12. [Tạo Cloudflare Tunnel (chạy trong cụm)](#12-tạo-cloudflare-tunnel-chạy-trong-cụm)
13. [Trỏ domain &amp; kiểm tra trên Internet](#13-trỏ-domain--kiểm-tra-trên-internet)
14. [Cài Rancher 2.14.3 &amp; quản lý cụm](#14-cài-rancher-2143--quản-lý-cụm)
15. [Vận hành &amp; troubleshooting](#15-vận-hành--troubleshooting)
16. [Nguồn official](#16-nguồn-official)
17. [Phụ lục](#17-phụ-lục)

---

## 1. Tổng quan & kiến trúc

```
                Internet (người dùng gõ https://app.example.com)
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
   │  [Ingress host=app.example.com]      [Ingress host=rancher.example.com]
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
- **Traefik** làm điểm vào duy nhất, định tuyến theo hostname → host nhiều app/nhiều domain (kể cả **UI Rancher**) chỉ bằng cách thêm `Ingress` + một public hostname trong tunnel. (Runbook **không** dùng `ingress-nginx` — dự án đó đã bị khai tử 03/2026, lý do ở [§9.2](#92-vì-sao-traefik-mà-không-phải-ingress-nginx).)
- **Rancher** cài *vào chính cụm kubeadm*, quản trị cụm đó (hiện trong UI là cụm `local`) bằng giao diện. Runbook dùng **Kubernetes 1.35.6 + Rancher 2.14.3**, nằm trong dải Kubernetes `1.33–1.35` của support matrix Rancher 2.14.3. Tuy nhiên, `kubeadm` thuộc nhóm “Any/imported cluster”, không phải một distro được SUSE chứng nhận riêng như RKE2/K3s.

---

## 2. Quy hoạch

### 2.1. Phiên bản (ghim để khỏi lệch version-skew)

| Thành phần | Phiên bản dùng trong runbook | Ghi chú |
| --- | --- | --- |
| Ubuntu Server | **24.04.x LTS (Noble), amd64** | Cài bản Server, không cần GUI |
| Kubernetes | **v1.35.6**; gói Debian **`1.35.6-1.1`** | Repo `pkgs.k8s.io` theo minor `v1.35`; 1.35 còn được Kubernetes duy trì tới **28/02/2027** |
| Container runtime | **containerd 2.x từ Ubuntu 24.04** | Giữ gói Ubuntu đã backport bản vá; cấu hình plugin 2.x và cgroup driver `systemd` |
| CNI | **Flannel v0.28.7**; pod CIDR `10.244.0.0/16` | Dùng manifest release-pinned, không dùng nhánh `master`/`latest` |
| Ingress | **Traefik chart 41.0.2** → Proxy **v3.7.6** | Chart khai `kubeVersion: >=1.25.0-0`; xem [§9](#9-cài-ingress-controller-traefik) |
| Tunnel | **cloudflared `latest`** | Official manifest của Cloudflare cũng dùng tag này; chạy 2 replica + liveness probe |
| Cluster management | **Rancher 2.14.3** từ `rancher-stable` | Support matrix liệt kê Kubernetes **1.33–1.35** cho imported/other clusters |
| cert-manager | **v1.21.0** | Hỗ trợ và test với Kubernetes 1.33–1.36; dùng cho `ingress.tls.source=rancher` |
| Storage | **local-path-provisioner v0.0.36** | Phù hợp homelab; dữ liệu gắn với node, không phải storage HA |
| MetalLB (tuỳ chọn) | **v0.16.1** | Chỉ cần khi muốn IP `LoadBalancer` trong LAN |

> ⚠️ **Version-skew:** runbook chủ động cài `kubelet`/`kubeadm`/`kubectl` cùng bản `1.35.6-1.1` trên cả 3 node. Chính sách Kubernetes hiện cho phép `kubelet` thấp hơn `kube-apiserver` tối đa **3 minor** và không được cao hơn; đó là biên tương thích, không phải lý do để cố tình để các node lệch bản trong một lab mới.
>
> **Phạm vi cam kết:** đây là baseline **tương thích và cài được cho homelab**, không phải bảo đảm “stable 100%” hay cấu hình Rancher production được SUSE chứng nhận end-to-end. Support matrix của Rancher ghi “Any” cho imported cluster; tài liệu production của Rancher khởi điểm ở 4 vCPU/16 GB **mỗi node** và yêu cầu upstream cluster HA 3 node.

### 2.2. IP & hostname (ví dụ — đổi theo dải LAN của bạn)

Giả sử LAN nhà bạn là `192.168.100.0/24`, gateway `192.168.100.1`. Chọn IP tĩnh **ngoài dải DHCP** của router.

| Vai trò | Hostname | IP tĩnh | vCPU | RAM | Disk SSD |
| --- | --- | --- | --- | --- | --- |
| Control plane | `k8s-master` | `192.168.100.111` | **4** | **8 GB** | 40 GB |
| Worker 1 | `k8s-worker1` | `192.168.100.112` | 2 | **6 GB** | 40 GB |
| Worker 2 | `k8s-worker2` | `192.168.100.113` | 2 | **6 GB** | 40 GB |

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
   - **Profile**: tạo user (vd `k8sadmin`), đặt hostname đúng (`k8s-master`…).
   - **Tick "Install OpenSSH server"** để SSH vào cho tiện.
   - Bỏ qua các snap đề xuất.
6. Cài xong → reboot → đăng nhập.

> 💡 Sau khi cài, nên SSH từ máy host vào từng VM (`ssh k8sadmin@192.168.100.111`) để copy-paste lệnh dễ hơn.

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

Không cần tạo lại `machine-id` hoặc SSH host key. Chỉ chạy phần **Verify identity** bên dưới để xác nhận mỗi VM thực sự có định danh riêng.

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

> **Chưa reboot tại đây.** Tiếp tục đặt hostname ở [§5.1](#51-hostname--etchosts), cấu hình IP tĩnh ở [§5.2](#52-ip-tĩnh-netplan), rồi reboot một lần sau `netplan apply`. Nếu VM đã được reset identity trước đó thì không cần chạy lại; chuyển thẳng sang phần verify. Reset SSH host key sẽ làm fingerprint của máy thay đổi, vì vậy client SSH có thể phải xóa entry cũ bằng `ssh-keygen -R <IP-cu>`.

#### Verify identity — bắt buộc cho cả hai trường hợp

Sau khi reset identity, chạy trên cả 3 VM và lưu kết quả để đối chiếu. Reboot cuối cùng sau khi đặt hostname/IP sẽ bảo đảm toàn bộ thay đổi có hiệu lực đồng thời:

```bash
echo "hostname:     $(hostnamectl --static)"
echo "machine-id:   $(cat /etc/machine-id)"
echo "product_uuid: $(sudo cat /sys/class/dmi/id/product_uuid)"
echo "SSH host key:"
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
echo "Network interfaces:"
ip -br link
```

Gate trước khi tiếp tục:

- `machine-id` của 3 VM phải khác nhau.
- SSH host-key fingerprint của 3 VM phải khác nhau.
- MAC address của card mạng trên 3 VM phải khác nhau.
- `product_uuid` của 3 VM phải khác nhau.
- Nếu MAC hoặc `product_uuid` bị trùng, **dừng tại đây** và để VMware sinh UUID/MAC mới; reset file trong Ubuntu không sửa được hai giá trị này.
- Hostname có thể còn là tên của golden VM ở bước này; [§5.1](#51-hostname--etchosts) sẽ đặt tên riêng cho từng K8s node.

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
ip -br a        # trên VMware thường là ens33 hoặc ens160 — nhớ tên này để điền vào YAML
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
ip -br a                       # card hiển thị đúng IP tĩnh .111/.112/.113
ping -c2 192.168.100.1         # tới gateway → phải có reply
ping -c2 8.8.8.8               # ra Internet bằng IP → phải có reply
ping -c2 google.com            # phân giải DNS → phải có reply
                               #   (nếu ping 8.8.8.8 OK mà google.com fail = lỗi DNS → xem lại nameservers)
```

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

| Thành phần | Giá trị mong muốn | Ý nghĩa |
| --- | --- | --- |
| Linux kernel | cgroup v2 (`cgroup2fs`) | Phiên bản API cgroup của kernel |
| kubelet | driver `systemd` | kubelet nhờ systemd quản lý cây cgroup |
| containerd/runc | `SystemdCgroup = true` | runtime cũng dùng systemd quản lý cgroup |

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

| Phạm vi | Cổng cần mở |
| --- | --- |
| Control plane | TCP `6443` từ các node/client quản trị; TCP `10250` từ control plane |
| Worker | TCP `10250` từ control plane |
| Tất cả node dùng Flannel VXLAN | UDP `8472` **giữa các node** |
| Chỉ khi dùng NodePort từ LAN | TCP **và UDP** `30000-32767` từ LAN |
| Chỉ khi load balancer cần health check kube-proxy | TCP `10256` |
| Cụm nhiều control plane | TCP `2379-2380` chỉ giữa các control-plane/etcd member |

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

> 💡 **Dùng cho Ansible:** sau khi bật, inventory chỉ cần `ansible_user=root` (Cách A thêm `ansible_ssh_private_key_file=~/.ssh/id_ed25519`). Nếu để user thường thì dùng `ansible_user=k8sadmin` + `become: true`.

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

# Gate của baseline này: dừng nếu node đang cài sai version
test "$KUBERNETES_VERSION" = "v1.35.6" || {
  echo "FAIL: expected kubeadm v1.35.6, got $KUBERNETES_VERSION" >&2
  exit 1
}

# Xem trước và pull đúng bộ image của version vừa xác nhận
sudo kubeadm config images list --kubernetes-version "$KUBERNETES_VERSION"
sudo kubeadm config images pull --kubernetes-version "$KUBERNETES_VERSION"
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

| #  | Kiểm tra                     | Lệnh                                                                                | PASS                                               |
| -- | ----------------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------- |
| 1  | Kubernetes đúng baseline | `kubectl version` | Server **v1.35.6** |
| 2  | 3 node Ready, worker 0 taint  | [§8.2](#82-tầng-2--node--tài-nguyên)                                              | 3/3                                                |
| 3  | Pod cross-node thông         | [§8.3](#83-tầng-3--pod-networking-cross-node)                                       | 0% packet loss                                     |
| 4  | DNS nội + ngoại             | [§8.4](#84-tầng-4--dns)                                                             | cả 2 resolve                                      |
| 5  | RAM còn headroom | `kubectl describe nodes \| grep -A6 Allocated` | Requests < 50% trước Rancher |
| 6  | quyền cluster-admin | `kubectl auth can-i '*' '*' --all-namespaces` | `yes` |
| 7  | exec / logs / port-forward | [§8.6](#86-tầng-6--đường-control-plane--kubelet-rancher-ui-sống-chết-ở-đây) | cả 3 lệnh OK |
| 8  | metrics-server | `kubectl top nodes` | ra số liệu |
| 9  | Default StorageClass | `kubectl get storageclass` | `local-path` có `(default)` |

### 8.9. Dọn dẹp resource test

```bash
kubectl delete pod nettest-w1 nettest-w2
kubectl delete deploy web
kubectl delete svc web web-np
```

---

## 9. Cài Ingress Controller (Traefik)

> ⚠️ **Runbook này KHÔNG dùng `ingress-nginx`.** Dự án đó đã bị **khai tử tháng 03/2026** và hiện có lỗ hổng RCE chưa vá đang bị khai thác thực tế. Lý do đầy đủ ở [§9.2](#92-vì-sao-traefik-mà-không-phải-ingress-nginx) — **đọc phần đó trước khi bạn định thay Traefik bằng nginx theo thói quen cũ.**

### 9.1. Kiến thức nền — Ingress và Ingress Controller

**Vấn đề cần giải:** mặc định, để lộ một app ra ngoài cụm bạn chỉ có 2 lựa chọn, cả 2 đều tệ khi số app tăng lên:

| Cách                  | Hoạt động                                                 | Nhược điểm                                                                                         |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| `type: NodePort`     | mở 1 cổng random`30000–32767` trên **mọi** node | URL xấu (`http://192.168.100.111:31234`), mỗi app 1 cổng, không TLS, không route theo domain    |
| `type: LoadBalancer` | xin cloud cấp 1 IP public + 1 LB riêng                     | **mỗi Service = 1 LB** → trên cloud thì tốn tiền, trên bare-metal thì không có ai cấp |

**Ingress** sinh ra để giải quyết: **một** điểm vào duy nhất (cổng 80/443), rồi phân luồng vào nhiều Service **dựa trên tên miền và đường dẫn** — đúng mô hình *virtual host* của web server truyền thống.

**Phân biệt 2 khái niệm hay bị lẫn:**

| Khái niệm                  | Là gì                                                                                                                                                                           | Ví dụ                                                                                                                                                                 |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Ingress** (object)   | Chỉ là**bản khai báo YAML** — "domain X, path Y → Service Z". Tự nó **không làm gì cả**. Apply Ingress mà không có controller = không có gì xảy ra | file`demo-app.yaml` ở [§10](#10-deploy-app-mẫu--ingress)                                                                                                            |
| **Ingress Controller** | **Phần mềm chạy thật** trong cụm, đọc các object Ingress rồi thực thi việc route                                                                                 | **Traefik** (runbook này), HAProxy, Kong, Envoy Gateway — và `ingress-nginx` *(đã khai tử, [§9.2](#92-vì-sao-traefik-mà-không-phải-ingress-nginx))* |

> K8s **cố ý không** ship sẵn Ingress Controller — bạn phải tự chọn và cài. Đây là lý do một cụm kubeadm mới toanh apply Ingress vào thì "không thấy gì chạy".

**Traefik xử lý việc đó thế nào?** Traefik là **một binary Go duy nhất**, vừa là proxy vừa là controller — khác hẳn kiến trúc "2 phần" của các controller dựa trên nginx/HAProxy:

|                   | Controller kiểu nginx/HAProxy                      | **Traefik**                                                               |
| ----------------- | --------------------------------------------------- | ------------------------------------------------------------------------------- |
| Cấu trúc        | proxy (C) + controller (Go) sinh file config        | 1 binary Go làm cả 2                                                          |
| Khi Ingress đổi | sinh lại`nginx.conf` → **reload process** | cập nhật**bảng route trong RAM**, không reload, không rớt kết nối |
| Cấu hình        | file tĩnh, phải render lại                       | động từ đầu — thiết kế cho môi trường thay đổi liên tục          |

Bốn khái niệm của Traefik, đi theo đúng đường đi của một request:

```
Request "Host: app.example.com"
        │
        ▼
[EntryPoint]   cổng lắng nghe — web(:80) / websecure(:443)
        │
        ▼
[Router]       luật khớp:  Host(`app.example.com`)
        │      (Traefik tự sinh Router này từ Ingress của bạn)
        ▼
[Middleware]   (tuỳ chọn) auth, rate-limit, strip prefix, redirect...
        │
        ▼
[Service]      danh sách Pod IP lấy từ EndpointSlices → load-balance
        │
        ▼
      Pod app
```

**Provider** là nguồn Traefik đọc cấu hình. Chart bật sẵn 2 cái:

- `kubernetesIngress` — đọc object `Ingress` chuẩn. **Đây là thứ runbook này dùng.**
- `kubernetesCRD` — đọc CRD riêng của Traefik (`IngressRoute`, `Middleware`). Mạnh hơn Ingress (TCP/UDP, middleware phức tạp) nhưng **khoá chặt vào Traefik**.

> 💡 Ba điểm dễ hiểu nhầm:
>
> - Với `nativeLBByDefault: false`, Traefik lấy backend từ **EndpointSlices** và proxy tới Pod IP, thay vì đưa request qua ClusterIP của Service.
> - Ingress chuẩn chỉ hiểu **HTTP/HTTPS (L7)**. Muốn expose TCP/UDP thuần (Postgres, MySQL...) phải dùng `IngressRouteTCP` của Traefik, hoặc **Gateway API**.
> - Runbook dùng `Ingress` chuẩn **có chủ đích**: kiến thức đó portable sang mọi controller. `IngressRoute` tiện hơn nhưng học xong chỉ dùng được với Traefik.

**`IngressClass`** — chart đặt tên class theo tên Helm release nếu không khai `ingressClass.name`. Runbook cố định cả release và class là `traefik`, đồng thời đặt làm **mặc định của cụm**. Ingress ở [§10](#10-deploy-app-mẫu--ingress) vẫn khai tường minh `ingressClassName: traefik` để sau này có nhiều controller không bị mơ hồ.

> Khác với `ingress-nginx`, Traefik **không có admission webhook**. Ingress sai cú pháp sẽ không bị chặn lúc `apply` mà đơn giản là không được route — debug bằng `kubectl describe ingress` và dashboard ([§9.3](#93-cài-đặt)).

### 9.2. Vì sao Traefik mà không phải ingress-nginx

Hầu hết tài liệu K8s trên mạng (và các khoá học cũ) đều dạy `ingress-nginx`. Nó từng chạy ở **~50% số cụm K8s toàn thế giới**. Nhưng:

| Mốc                 | Sự kiện                                                                                                                                                                                                                                                |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **11/2025**    | Kubernetes công bố kế hoạch khai tử. Lý do: suốt nhiều năm chỉ**1–2 người** maintain ngoài giờ làm, trong khi nợ kỹ thuật (đặc biệt cơ chế `snippets` annotation) khiến việc vá bảo mật *"không còn khả thi"*  |
| **01/2026**    | Steering Committee + Security Response Committee ra tuyên bố khẩn:*"Một nửa trong số các bạn sẽ bị ảnh hưởng. Các bạn còn hai tháng để chuẩn bị."*                                                                                |
| **03/2026**    | **Retired chính thức.** Bản cuối `v1.15.1`. Không còn release, bugfix, hay bản vá bảo mật nào nữa                                                                                                                                    |
| **13/05/2026** | 💥 **CVE-2026-42945 ("NGINX Rift")** — heap buffer overflow trong `ngx_http_rewrite_module`, tồn tại từ 2008. Nginx upstream xếp **Medium**; một số hãng bảo mật chấm **CVSS v4.0 9.2 Critical** theo kịch bản xấu nhất. Hoạt động khai thác được ghi nhận từ 16/05/2026 |

nginx upstream đã vá (1.30.1/1.31.0). F5 đã vá bản thương mại của họ. **`ingress-nginx` v1.15.1 thì không — và sẽ không bao giờ**, vì không còn maintainer.

Đặt vào đúng kiến trúc runbook này: ingress controller nằm **ngay rìa cụm** và [§12](#12-tạo-cloudflare-tunnel-chạy-trong-cụm) chủ động đưa nó ra Internet. Đó chính xác là kịch bản *"unauthenticated RCE at the cluster's edge"*.

**Ba thứ hay bị gộp làm một — cần tách bạch:**

| Thứ                                                             | Trạng thái                                                      |
| ---------------------------------------------------------------- | ----------------------------------------------------------------- |
| **nginx** (web server)                                     | ✅ Sống khỏe, vẫn vá đều                                    |
| **Ingress API** của K8s (`kind: Ingress`)               | ✅**Không** hề bị deprecate — vẫn là API chính thức |
| **`kubernetes/ingress-nginx`** (controller cộng đồng) | ❌ Đã retired 03/2026                                           |
| **`nginx/kubernetes-ingress`** (F5/NGINX Inc.)           | ✅ Dự án**khác hoàn toàn**, vẫn maintain, đã vá    |
| **InGate** (successor được lên kế hoạch)             | ❌ Không kịp trưởng thành, cũng đã bị khai tử           |

**Vì sao chọn Traefik trong số các phương án thay thế:**

- **SUSE/Rancher chính thức chọn Traefik** làm đích migration và có [hướng dẫn riêng](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-resources-setup/load-balancer-and-ingress-controller/guide-to-ingress-nginx-retirement). Vì đích đến của runbook là Rancher ([§14](#14-cài-rancher-2143--quản-lý-cụm)), đi cùng hệ sinh thái là hợp lý.
- Traefik có provider tương thích `providers.kubernetesIngressNGINX` để đọc **một phần** annotation của ingress-nginx, nhưng mặc định tắt và không phải drop-in hoàn chỉnh. Không bật đồng thời với `kubernetesIngress` mà không giới hạn class/namespace, vì có thể sinh router trùng; nhiều annotation và semantics vẫn không được hỗ trợ.
- Hỗ trợ **Gateway API** sẵn — hướng đi mà Kubernetes khuyến nghị dài hạn, không phải làm lại từ đầu sau này.
- Chart gọn, 1 binary, hợp lab nhỏ.

> Các phương án khác nếu bạn gặp trong công việc: **F5 NGINX Ingress Controller** (`nginx/kubernetes-ingress`, OSS miễn phí, đã vá — nhưng **cú pháp annotation khác**, không phải drop-in), **Envoy Gateway / Cilium Gateway** (thuần Gateway API), hoặc bản hỗ trợ thương mại có vá cho `ingress-nginx` v1.15.1.

### 9.3. Cài đặt

Cài bằng Helm đã chuẩn bị ở [§8.7](#87-tầng-7--công-cụ-và-add-on-kubeadm-không-cài-sẵn), ghim chart **41.0.2** để lệnh và giá trị mặc định không trôi. Trên master:

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm upgrade --install traefik traefik/traefik \
  --namespace traefik --create-namespace \
  --version 41.0.2 \
  --set ingressClass.name=traefik \
  --set ingressRoute.dashboard.enabled=true
```

Các giá trị hiệu lực sau lệnh trên; phần lớn là mặc định chart, hai giá trị class/dashboard được ghim tường minh nên không cần `values.yaml` riêng:

| Giá trị hiệu lực | Ý nghĩa |
| --------------------------------------------------------- | --------------------------------------------------------------------------- |
| `providers.kubernetesIngress.enabled: true`             | Đọc object`Ingress` chuẩn ✅                                           |
| `ingressClass.enabled: true` + `isDefaultClass: true` | Tạo IngressClass`traefik`, đặt làm mặc định cụm                   |
| Container `ports.web.port: 8000` → Service `exposedPort: 80` | EntryPoint HTTP; pod không cần bind cổng đặc quyền 80 |
| Container `ports.websecure.port: 8443` → Service `exposedPort: 443` | EntryPoint HTTPS |
| `service.spec.type: LoadBalancer` | Xem [§9.4](#94-vì-sao-external-ip-là-pending-mãi--và-vì-sao-không-sao) |
| `ingressRoute.dashboard.enabled: true` | Chỉ tạo route dashboard trên entrypoint nội bộ `traefik`; không expose qua Service/Ingress Internet |

Kiểm tra:

```bash
kubectl -n traefik rollout status deploy/traefik
kubectl get pods,svc -n traefik
kubectl get ingressclass          # PASS: có "traefik" và cột DEFAULT = true
```

**Đợi controller Ready trước khi tạo Ingress ở [§10]** — Ingress apply lúc Traefik chưa lên sẽ không được route (không báo lỗi gì, nên rất dễ tưởng nhầm là cấu hình sai):

```bash
kubectl -n traefik wait --for=condition=ready pod \
  -l app.kubernetes.io/name=traefik --timeout=180s
```

**(Tuỳ chọn) Dashboard** — xem trực quan Router/Service/Middleware mà Traefik đã sinh ra từ Ingress của bạn. Rất đáng dùng khi debug:

```bash
TRAEFIK_POD=$(kubectl -n traefik get pod \
  -l app.kubernetes.io/name=traefik \
  -o jsonpath='{.items[0].metadata.name}')
kubectl -n traefik port-forward "pod/$TRAEFIK_POD" 8080:8080
# mở http://127.0.0.1:8080/dashboard/   ← BẮT BUỘC có dấu / ở cuối
```

> Chart 41.0.2 dùng cổng admin `8080`, không còn `9000`. Dashboard được bật bằng `ingressRoute.dashboard.enabled=true` nhưng entrypoint `traefik` không expose mặc định; chỉ truy cập qua `port-forward`. Đừng tạo Ingress Internet cho dashboard.

### 9.4. Vì sao `EXTERNAL-IP` là `<pending>` mãi — và vì sao KHÔNG SAO

Sau khi cài, `kubectl get svc -n traefik` sẽ ra thế này và **đứng nguyên như vậy vĩnh viễn**:

```
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
traefik   LoadBalancer   10.104.21.187   <pending>     80:31234/TCP,443:31892/TCP
```

**Nguyên nhân — ai là người điền `EXTERNAL-IP`?**

Kubernetes **tự nó không biết cách tạo load balancer**. Khi bạn tạo Service `type: LoadBalancer`, K8s chỉ ghi vào etcd một yêu cầu "cần LB", rồi **chờ một thành phần bên ngoài** tên **cloud-controller-manager (CCM)** đến xử lý:

```
Service type=LoadBalancer  ──►  CCM của cloud provider
                                      │  gọi API của nhà cung cấp
                                      ▼
                            AWS tạo ELB / GCP tạo Forwarding Rule /
                            Azure tạo Load Balancer  → trả về 1 IP public
                                      │
                                      ▼
                    CCM ghi IP đó vào  svc.status.loadBalancer.ingress
                                      │
                                      ▼
                          kubectl hiển thị ở cột EXTERNAL-IP
```

Cụm của bạn dựng bằng **kubeadm trên VMware — bare-metal**, **không có CCM nào cả**. Không ai nhận yêu cầu → trường `status.loadBalancer.ingress` không bao giờ được ghi → `kubectl` hiển thị `<pending>`.

Kiểm chứng — trường status rỗng, và không có event lỗi nào (nghĩa là cụm không hỏng, chỉ là không ai xử lý):

```bash
kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer}{"\n"}'
# → {}   (rỗng — đúng như mong đợi trên bare-metal)
```

> ⚠️ `<pending>` ở đây **không phải lỗi cần sửa**. Nó là trạng thái đúng của một yêu cầu không có ai phục vụ. Đừng đi debug CNI/firewall vì dòng này.

**Vì sao không sao — Service vẫn hoạt động đầy đủ:**

3 kiểu Service là **quan hệ bao hàm**, kiểu sau bao gồm toàn bộ kiểu trước:

```
ClusterIP  ⊂  NodePort  ⊂  LoadBalancer
```

Nên một Service `type: LoadBalancer` **vẫn được cấp đủ**:

- ✅ **ClusterIP** (`10.104.21.187`) — truy cập được từ mọi pod trong cụm
- ✅ **NodePort** (`31234`, `31892`) — truy cập được từ LAN qua IP bất kỳ node nào
- ❌ **External IP** — chỉ riêng phần này thiếu

Tức là **chỉ mất đúng cái IP public mà kiến trúc này không dùng tới**.

**Kiến trúc của runbook này cố tình không cần external IP:** `cloudflared` chạy **bên trong cụm** như một Deployment ([§12.2](#122-deploy-cloudflared-vào-cụm)). Nó đã ở trong mạng pod rồi, nên gọi thẳng ingress qua **DNS nội bộ + ClusterIP**:

```
Internet → Cloudflare edge → (tunnel outbound, không mở port nào ở router)
        → pod cloudflared trong cụm
        → http://traefik.traefik.svc.cluster.local:80
        → Traefik → Pod của app
```

Đường vào cụm là **outbound tunnel** do cloudflared tự mở ra, **không phải** inbound tới một IP public. Vì vậy external IP không những không cần, mà còn là thứ ta chủ động tránh — đó chính là ưu điểm bảo mật của mô hình tunnel ([§1](#1-tổng-quan--kiến-trúc)): **router không mở cổng nào, cụm không phơi IP nào ra Internet**.

**Khi nào thì bạn *thực sự* cần sửa `<pending>`:**

| Tình huống                                                            | Cần xử lý?                                                                                                                                                                   |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ra Internet qua Cloudflare Tunnel (runbook này)                        | **Không** — bỏ qua hoàn toàn                                                                                                                                         |
| Muốn gõ`http://192.168.100.x` từ LAN để test nhanh               | Không bắt buộc — dùng luôn**NodePort** đã được cấp sẵn                                                                                                       |
| Muốn một IP LAN cố định, đẹp (vd`192.168.100.200`) cho ingress | Cài**MetalLB** — nó đóng vai CCM cho bare-metal, cấp IP từ dải LAN bạn khoanh sẵn. Xem [Phụ lục B](#phụ-lục-b--metallb-tuỳ-chọn-loadbalancer-ip-trong-lan) |

> 💡 Nếu muốn bỏ trạng thái `<pending>` và chắc chắn không dùng MetalLB, chạy `helm upgrade traefik traefik/traefik -n traefik --version 41.0.2 --reuse-values --set service.spec.type=NodePort`. Key đúng của chart 41.0.2 là `service.spec.type`, không phải `service.type`.

---

## 10. Deploy app mẫu + Ingress

> Toàn bộ [§10] chạy **trên master** (nơi có `kubectl`). File `.yaml` tạo ở thư mục home của user (vd `~/demo-app.yaml`).

Tạo file `demo-app.yaml` (đổi `app.example.com` thành domain thật của bạn — sẽ mua ở [§11]):

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
    - host: app.example.com          # ⚠️ domain thật của bạn
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

```bash
kubectl apply -f demo-app.yaml
kubectl get ingress
```

**Test nội bộ** (chưa cần domain) — giả lập Host header tới ingress:

```bash
# lấy ClusterIP của ingress controller:
ING_IP=$(kubectl get svc -n traefik traefik -o jsonpath='{.spec.clusterIP}')
curl -s -H "Host: app.example.com" http://$ING_IP/
# → phải thấy "Server address..." từ nginx hello
```

> 404 ở bước này = Traefik chưa sinh Router cho Ingress. Soi bằng `kubectl describe ingress web`, hoặc mở dashboard ([§9.3](#93-cài-đặt)) xem tab **HTTP → Routers**.

Thấy output nghĩa là chuỗi **ingress → service → pod** đã chạy. Giờ chỉ còn nối Internet vào.

---

## 11. Mua & cấu hình domain trên Cloudflare

### 11.1. Mua domain

Mua ở bất kỳ registrar nào (Cloudflare Registrar, Namecheap, GoDaddy, PA Vietnam…). Domain rẻ (`.com`, `.dev`, `.xyz`) đều được.

### 11.2. Thêm domain vào Cloudflare

1. Tạo tài khoản [https://dash.cloudflare.com](https://dash.cloudflare.com) → **Add a site** → nhập domain.
2. Chọn plan **Free**.
3. Cloudflare cấp **2 nameserver** (vd `xxx.ns.cloudflare.com`).
4. Vào trang quản trị của **registrar** → đổi **nameservers** của domain sang 2 NS Cloudflare vừa cấp.
5. Đợi DNS propagate (vài phút → tối đa 24h). Khi domain ở Cloudflare hiện **Active** là xong.

> Nếu mua luôn bằng **Cloudflare Registrar** thì bước đổi nameserver được làm sẵn.

---

## 12. Tạo Cloudflare Tunnel (chạy trong cụm)

Dùng **remotely-managed tunnel** (quản lý qua dashboard bằng *token*) — đơn giản, hợp với cụm k8s và đúng theo deployment guide của Cloudflare. (Cách CLI `config.yml` xem [Phụ lục A](#phụ-lục-a--locally-managed-tunnel-cli--configyml).)

### 12.1. Tạo tunnel trên dashboard → lấy token

1. Cloudflare dashboard → **Networking → Tunnels** → **Create Tunnel**.
2. Chọn loại **Cloudflared** → đặt tên (vd `homelab-k8s`) → **Save**.
3. Màn hình *Install and run* hiện lệnh có dạng `cloudflared ... run --token eyJh...`. **Copy chuỗi token `eyJh...`** (rất dài). Chưa cần chạy lệnh đó — ta sẽ nhúng token vào k8s.

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
kubectl get pods -n cloudflare
```

Quay lại dashboard tunnel → trạng thái phải chuyển **Healthy** (2 connector).

### 12.3. Thêm Published application (route domain → ingress nội bộ)

Trong tunnel vừa tạo → **Published application routes** → **Add a published application** (một số tài khoản/UI cũ còn hiện “Public Hostname”):

| Trường            | Giá trị                                    |
| ------------------- | -------------------------------------------- |
| **Subdomain** | `app` (để trống nếu dùng root domain) |
| **Domain**    | `example.com` (chọn từ dropdown)         |
| **Type**      | `HTTP`                                     |
| **URL**       | `traefik.traefik.svc.cluster.local:80`     |

→ **Save**. Với DNS **Full Setup** như [§11.2](#112-thêm-domain-vào-cloudflare), Cloudflare tự tạo DNS record cho hostname. Nếu zone dùng **Partial/CNAME Setup**, phải tự tạo CNAME tại DNS provider theo hướng dẫn Cloudflare.

> Vì cloudflared chạy *trong* cụm, nó resolve được tên DNS nội bộ `*.svc.cluster.local` và gọi thẳng ClusterIP của Traefik. Hostname `app.example.com` được giữ nguyên trong Host header → Traefik khớp đúng Router sinh từ `Ingress` ở [§10].
>
> (Tuỳ chọn) Nếu cần ép Host header, mở **Additional application settings → HTTP Settings → HTTP Host Header** = `app.example.com`.

---

## 13. Trỏ domain & kiểm tra trên Internet

```bash
# 1) DNS đã có CNAME chưa
nslookup app.example.com        # phải trả về IP của Cloudflare (proxied)

# 2) Truy cập (TLS do Cloudflare cấp tự động ở edge)
curl -I https://app.example.com
```

Mở trình duyệt → `https://app.example.com` → thấy trang `nginxdemos/hello` ⇒ **toàn bộ chuỗi đã thông**:

```
Browser → Cloudflare edge (HTTPS) → Tunnel → cloudflared(pod) →
Traefik(ClusterIP) → Router(host match) → Service web → Pod web
```

> **TLS:** Cloudflare tự lo cert ở edge (Internet ⇄ Cloudflare = HTTPS). Đoạn Cloudflare ⇄ cụm đi trong tunnel mã hoá; ingress để HTTP (không cần cert nội bộ). Traefik **không** tự redirect HTTP→HTTPS nên không cần cấu hình gì thêm — với `ingress-nginx` thì đây là chỗ phải thêm annotation `ssl-redirect: "false"`.

---

## 14. Cài Rancher 2.14.3 & quản lý cụm

> Cài Rancher **vào chính cụm kubeadm** vừa dựng (Rancher chạy như workload trong namespace `cattle-system`). Cụm host nó sẽ tự xuất hiện trong UI Rancher dưới tên **`local`** — **không cần import thủ công**. Muốn quản thêm cụm khác sau này thì dùng **Cluster Management → Import Existing**.
>
> **Yêu cầu trước:** đã xong [§8](#8-verify-cụm) (cụm Ready), [§9](#9-cài-ingress-controller-traefik) (Traefik) và [§12](#12-tạo-cloudflare-tunnel-chạy-trong-cụm) (tunnel Healthy). Chọn hostname cho Rancher, vd `rancher.example.com`.

### 14.1. Cài cert-manager (Rancher cần để cấp TLS nội bộ)

Trên **master**:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.21.0 \
  --set crds.enabled=true

kubectl rollout status deploy/cert-manager -n cert-manager
kubectl get pods -n cert-manager     # 3 pod Running
```

### 14.2. Cài Rancher (Helm, pin 2.14.3)

Gate ngay trước khi cài — lúc này các phụ thuộc đã thực sự tồn tại:

```bash
helm version --short                         # >= v3.18
kubectl version                              # Server v1.35.6
kubectl get pods,svc -n traefik              # Traefik Running
kubectl get ingressclass                     # traefik, DEFAULT=true
kubectl get pods -n cert-manager             # 3 pod Running
kubectl get crd | grep cert-manager.io
kubectl get pods -n cloudflare               # 2 cloudflared Running
```

**Split DNS cho Rancher agent:** agent trong cụm cũng kết nối tới `rancher.example.com`. Cho traffic nội bộ đi thẳng tới Traefik, tránh vòng ra Cloudflare và tránh bị Cloudflare Access chặn:

```bash
TRAEFIK_IP=$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.clusterIP}')
echo "$TRAEFIK_IP rancher.example.com"
kubectl -n kube-system get configmap coredns -o yaml > coredns-before-rancher.yaml
kubectl -n kube-system edit configmap coredns
```

Trong `Corefile`, thêm khối sau **bên trong** server block `.:53`, trước `forward . /etc/resolv.conf` (thay `<TRAEFIK_CLUSTER_IP>` bằng giá trị vừa in):

```text
hosts {
    <TRAEFIK_CLUSTER_IP> rancher.example.com
    fallthrough
}
```

Áp dụng và kiểm tra:

```bash
kubectl -n kube-system rollout restart deployment coredns
kubectl -n kube-system rollout status deployment coredns
kubectl run dns-check --rm -it --restart=Never --image=busybox:1.36 \
  -- nslookup rancher.example.com
# PASS: trả về ClusterIP của traefik, không phải IP Cloudflare
```

> Lưu file `coredns-before-rancher.yaml` để rollback. Nếu Service Traefik bị xoá/tạo lại và đổi ClusterIP, cập nhật entry này.

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
kubectl create namespace cattle-system

helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --version 2.14.3 \
  --set hostname=rancher.example.com \
  --set bootstrapPassword='ĐẶT-MẬT-KHẨU-ADMIN-MẠNH' \
  --set replicas=1 \
  --set ingress.ingressClassName=traefik \
  --set ingress.tls.source=rancher

# đợi rollout (kéo image, có thể vài phút)
kubectl -n cattle-system rollout status deploy/rancher
kubectl -n cattle-system get pods

# xác nhận Ingress của Rancher đã được Traefik nhận
kubectl -n cattle-system get ingress      # CLASS phải = traefik
```

- `--version 2.14.3`: ghim đúng chart stable; support matrix của bản này liệt kê Kubernetes 1.33–1.35 cho imported/other clusters.
- `replicas=1`: homelab 1 cụm (mặc định chart là 3 — quá tải cho lab nhỏ).
- `ingress.ingressClassName=traefik`: **bắt buộc khi dùng Traefik.** Chart Rancher mặc định nhắm `ingress-nginx`; thiếu cờ này thì `Ingress` của Rancher không có controller nào nhận → UI 404 dù pod vẫn `Running`.
- `ingress.tls.source=rancher`: Rancher tạo CA/certificate riêng qua cert-manager và tạo `Ingress` TLS cho `rancher.example.com`; Traefik terminate TLS ở cổng 443.

> **Traefik + Rancher UI:** Rancher dùng **WebSocket** rất nhiều (shell, log, cluster events). Traefik hỗ trợ WebSocket sẵn, không cần annotation gì thêm. Nó cũng tự set `X-Forwarded-Proto` — thứ mà Rancher cần để không rơi vào redirect-loop.

> **Thay thế (nhẹ hơn, bỏ cert-manager):** `--set tls=external` → Rancher chạy HTTP thuần, TLS chỉ chấm dứt ở Cloudflare edge; tunnel khi đó trỏ HTTP→`...:80`. Nhược điểm: phải đảm bảo header `X-Forwarded-Proto: https` (Cloudflare có set sẵn) nếu không dễ redirect-loop. Runbook này dùng `ingress.tls.source=rancher` cho chắc.

### 14.3. Bảo vệ Rancher bằng Cloudflare Access rồi publish qua tunnel

Rancher là giao diện quản trị cụm, không nên chỉ dựa vào màn hình đăng nhập Rancher. Tạo lớp xác thực Cloudflare Access **trước khi** publish:

1. Cloudflare dashboard → **Zero Trust → Access controls → Applications**.
2. **Create new application → Self-hosted and private → Add public hostname**.
3. Hostname: `rancher.example.com`.
4. Tạo policy **Allow** chỉ cho email/email domain quản trị; bật MFA nếu IdP hỗ trợ. Không tạo policy `Bypass Everyone`.
5. Save. Access mặc định deny người không khớp policy.

Sau đó thêm một **Published application route** trong tunnel đã tạo ở [§12.3](#123-thêm-published-application-route-domain--ingress-nội-bộ):

| Trường                                              | Giá trị                                       |
| ----------------------------------------------------- | ----------------------------------------------- |
| **Subdomain**                                   | `rancher`                                     |
| **Domain**                                      | `example.com`                                 |
| **Type**                                        | **HTTPS**                                 |
| **URL**                                         | `traefik.traefik.svc.cluster.local:443`       |
| **Additional settings → TLS → No TLS Verify** | **ON** (vì cert Rancher là self-signed) |
| **Additional settings → HTTP Host Header**     | `rancher.example.com`                         |

→ **Save**. Với zone Full Setup, Cloudflare tự tạo DNS record cho `rancher.example.com`; với Partial/CNAME Setup, tạo record tại DNS provider như lưu ý ở [§12.3](#123-thêm-published-application-route-domain--ingress-nội-bộ).

> `HTTP Host Header` được Traefik dùng để chọn HTTP router **sau** khi TLS handshake hoàn tất. SNI/certificate được xử lý trong handshake trước khi có Host header; `No TLS Verify` cho phép cloudflared chấp nhận certificate do Rancher tự ký. Đây là hai cơ chế khác nhau.
>
> Cloudflare Access hỗ trợ WebSocket. Rancher Shell/Logs vẫn có thể ngắt nếu kết nối WebSocket idle quá lâu; reconnect trước khi kết luận cụm hỏng.

### 14.4. Đăng nhập lần đầu

```bash
# nếu quên/không set bootstrapPassword, lấy lại:
kubectl get secret --namespace cattle-system bootstrap-secret \
  -o go-template='{{ .data.bootstrapPassword | base64decode }}{{ "\n" }}'
```

1. Mở `https://rancher.example.com` → đăng nhập bằng bootstrap password.
2. Đặt mật khẩu admin mới.
3. Xác nhận **Server URL** = `https://rancher.example.com` (Rancher gợi ý sẵn).
4. Vào **Cluster Management** → thấy cụm **`local`** = chính cụm kubeadm của bạn, trạng thái **Active**.

### 14.5. Kiểm tra

- UI hiện cụm `local` Active, đủ 3 node.
- Quản lý workload / Helm app / monitoring qua UI; cụm vẫn dùng song song bằng `kubectl`.

Kiểm tra agent và đường nội bộ:

```bash
kubectl -n cattle-system get pods
kubectl -n cattle-system logs deploy/cattle-cluster-agent --tail=30
```

> ⚠️ Rancher 2.14.3 hiện hỗ trợ tới Kubernetes 1.35. Không nâng cluster lên 1.36 cho tới khi support matrix của **bản Rancher đang chạy** cho phép. Với hệ thống hiện có ở Rancher 2.13 + Kubernetes 1.32, phải nâng Kubernetes lên vùng giao nhau 1.33/1.34 trước khi nâng Rancher 2.14.

---

## 15. Vận hành & troubleshooting

### Thêm app/domain mới sau này

Chỉ cần: (1) `Deployment`+`Service`+`Ingress` mới với `host: app2.example.com`; (2) thêm **Published application route** mới trong cùng tunnel trỏ về cùng `traefik.traefik.svc.cluster.local:80`. Không đụng gì tới VM/router.

### Vì sao phải tôn trọng EOL & upgrade đúng nhịp

Cụm **vẫn chạy** sau ngày EOL của k8s, nhưng dừng nhận **patch bảo mật (CVE)** + bug fix; kubeadm **không skip được minor** (phải nhảy từng bậc) nên để càng lâu càng khó nâng; và hệ sinh thái (CNI/ingress/cloudflared/**Rancher**) dần bỏ hỗ trợ bản cũ. Với cụm **phơi ra Internet qua tunnel**, rủi ro CVE chưa vá là nghiêm trọng nhất → nâng đều, luôn nằm trong vùng Rancher support.

### Backup etcd và cấu hình trước thay đổi lớn

Với 1 control plane, etcd là nơi giữ toàn bộ state Kubernetes và Rancher. Chạy trên `k8s-master` trước upgrade hoặc thay đổi quan trọng:

```bash
STAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="$HOME/k8s-backups/$STAMP"
mkdir -p "$BACKUP_DIR"

# snapshot etcd vào volume đã mount của static pod
kubectl -n kube-system exec etcd-k8s-master -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/kubeadm-snapshot.db

# lấy snapshot và toàn bộ cấu hình/certificate kubeadm ra thư mục backup
sudo cp /var/lib/etcd/kubeadm-snapshot.db "$BACKUP_DIR/etcd-snapshot.db"
sudo cp -a /etc/kubernetes "$BACKUP_DIR/etc-kubernetes"
sudo chown -R "$(id -u):$(id -g)" "$BACKUP_DIR"
sudo rm -f /var/lib/etcd/kubeadm-snapshot.db

ls -lh "$BACKUP_DIR/etcd-snapshot.db"
```

> **Bắt buộc:** copy thư mục backup ra khỏi VM/disk chứa etcd (NAS, máy host hoặc storage backup). Snapshot nằm cùng disk không giúp được khi disk VM hỏng. Việc restore etcd là thao tác sự cố riêng; làm theo tài liệu kubeadm/etcd của đúng phiên bản và dừng control plane trước khi restore.

### Nâng patch Kubernetes 1.35 bằng kubeadm

Ví dụ nâng `1.35.6` lên một patch `1.35.X` mới hơn. Đọc release notes, backup trước, và thay đúng hai biến sau bằng version có thật từ `apt-cache madison kubeadm`:

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
window trên `k8s-master`; backup etcd và `/etc/kubernetes` theo mục **Backup etcd và cấu hình trước
thay đổi lớn** ở trên trước khi tiếp tục.

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
- Cloudflare hỗ trợ WebSocket, nhưng đóng kết nối không có traffic sau idle timeout. Rancher Shell/log stream để lâu có thể rớt; reconnect và dùng keepalive phù hợp.
- Tunnel cần outbound TCP/UDP `7844`; nếu UDP bị chặn cloudflared có thể fallback HTTP/2 qua TCP. Nếu cả hai bị chặn, tunnel không lên.

### Bảng lỗi thường gặp

| Triệu chứng                                            | Nguyên nhân & cách xử lý                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| -------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cài Ubuntu:**DHCPv4 quay mãi, không ra IP**     | VMnet0 bridge bind nhầm card →**Edit → Virtual Network Editor → VMnet0**, đổi *Automatic* sang đúng card vật lý ([§3](#3-tạo-3-vm-ubuntu-2404-trên-vmware)). Xác nhận card đang chạy bằng `ipconfig /all` trên host. Tạm thời có thể đặt IP tĩnh ngay tại màn hình installer.                                                                                                                                                                                                                      |
| Bridge**vẫn** không ra IP dù chọn đúng card  | Host bật**Hyper-V** làm VMware bridge xung đột (danh sách "Bridged to" có "Hyper-V Virtual Ethernet Adapter"). Kiểm tra host: `bcdedit /enum \| findstr -i hypervisorlaunchtype`. Xử lý: tắt Hyper-V cho lab (`bcdedit /set hypervisorlaunchtype off` + reboot — **lưu ý sẽ tắt WSL2/Docker Desktop/Sandbox**), **hoặc** chuyển Network Adapter sang **NAT** (vẫn ra Internet; khi đó IP tĩnh phải theo dải NAT của VMnet8, vd `192.168.71.x`, và đổi đồng bộ toàn runbook). |
| Node`NotReady`                                         | CNI chưa chạy →`kubectl get pods -n kube-flannel`; kiểm tra UDP 8472 mở; `--pod-network-cidr` đúng `10.244.0.0/16`                                                                                                                                                                                                                                                                                                                                                                                                          |
| `kubeadm join` timeout                                 | Worker không resolve`k8s-master` (thiếu `/etc/hosts`); hoặc cổng `6443` master bị firewall chặn                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Pod kẹt`ContainerCreating`/CNI lỗi                   | sai cgroup driver →`sudo crictl info \| grep -i -A2 systemdCgroup` phải `true`, rồi `systemctl restart containerd kubelet`                                                                                                                                                                                                                                                                                                                                                                                                     |
| `kubelet` crashloop sau init                           | còn swap →`swapoff -a` + check `/etc/fstab`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Tunnel không**Healthy**                           | token sai/thiếu →`kubectl logs -n cloudflare deploy/cloudflared`; pod phải `Running`                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Domain mở ra**502/error 1033**                    | URL route sai tên service; thử đúng`traefik.traefik.svc.cluster.local:80` (hoặc `:443` cho Rancher); kiểm tra `kubectl get svc -n traefik`                                                                                                                                                                                                                                                                                                                                                                                   |
| Mở domain ra**404 page not found**                | Host header không khớp Ingress`host:` → set HTTP Host Header trong tunnel, hoặc sửa `host:` trong Ingress. Soi Router thực tế ở dashboard Traefik ([§9.3](#93-cài-đặt))                                                                                                                                                                                                                                                                                                                                                   |
| Ingress apply xong nhưng**không được route**  | Thiếu/sai`ingressClassName` → phải là `traefik`; kiểm tra `kubectl get ingress -A` cột CLASS và `kubectl get ingressclass`                                                                                                                                                                                                                                                                                                                                                                                                |
| **UI Rancher 404** dù pod Running                 | Chart Rancher mặc định nhắm`ingress-nginx` → cài lại/upgrade với `--set ingress.ingressClassName=traefik` ([§14.2](#142-cài-rancher-helm-pin-2143))                                                                                                                                                                                                                                                                                                                                                                         |
| `rancher.example.com` lỗi **TLS/redirect-loop** | thiếu**No TLS Verify=ON** + **Host Header**=`rancher.example.com` ở route HTTPS; hoặc cert-manager chưa Ready (`kubectl get pods -n cert-manager`)                                                                                                                                                                                                                                                                                                                                                                   |
| Rancher pod`CrashLoop`/Pending                         | thiếu RAM/headroom; hoặc cert-manager CRDs chưa cài (`--set crds.enabled=true`)                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Rancher `cattle-cluster-agent` CrashLoop/không connect | kiểm `rancher.example.com` trong pod có resolve về ClusterIP Traefik theo split DNS ở §14.2; nếu đi ra Cloudflare Access sẽ bị chặn |
| Rancher Shell/exec/attach/port-forward bị `forbidden` | Kubernetes 1.35 WebSocket upgrade cần verb RBAC `create` trên `pods/exec`, `pods/attach`, `pods/portforward`; kiểm Role/ClusterRole của user trước khi nghi firewall |
| Rancher Shell/log stream rớt sau khi idle | WebSocket qua Cloudflare bị idle timeout; reconnect, phân biệt với lỗi RBAC/10250 |
| Pull image private đã cache vẫn lỗi credential | Kubernetes 1.35 bật beta `KubeletEnsureSecretPulledImages`; kiểm `imagePullSecrets` hợp lệ thay vì dựa vào image đã cache |
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
- Kubernetes — *Ingress NGINX Retirement: What You Need to Know* (11/2025): [https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- Kubernetes — *Statement from the Steering and Security Response Committees* (01/2026): [https://kubernetes.io/blog/2026/01/29/ingress-nginx-statement/](https://kubernetes.io/blog/2026/01/29/ingress-nginx-statement/)
- Rancher — *Guide to Ingress NGINX Retirement* (đích migration = Traefik): [https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-resources-setup/load-balancer-and-ingress-controller/guide-to-ingress-nginx-retirement](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-resources-setup/load-balancer-and-ingress-controller/guide-to-ingress-nginx-retirement)
- Rancher — *local-path-provisioner stable install*: [https://github.com/rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner)
- MetalLB — *Installation*: [https://metallb.universe.tf/installation/](https://metallb.universe.tf/installation/)
- Cloudflare — *Tunnel setup*: [https://developers.cloudflare.com/tunnel/setup/](https://developers.cloudflare.com/tunnel/setup/)
- Cloudflare — *Deploy cloudflared in Kubernetes*: [https://developers.cloudflare.com/tunnel/deployment-guides/kubernetes/](https://developers.cloudflare.com/tunnel/deployment-guides/kubernetes/)
- Cloudflare — *Tunnel with firewall*: [https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/tunnel-with-firewall/](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/tunnel-with-firewall/)
- Cloudflare — *Self-hosted public application/Access*: [https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/)
- Cloudflare — *WebSockets*: [https://developers.cloudflare.com/network/websockets/](https://developers.cloudflare.com/network/websockets/)
- Cloudflare — *HTTP 413/upload limits*: [https://developers.cloudflare.com/support/troubleshooting/http-status-codes/4xx-client-error/error-413/](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/4xx-client-error/error-413/)
- Rancher/SUSE — *Support matrix v2.14.3*: [https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-14-3/](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-14-3/)
- Rancher — *Installation requirements*: [https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/installation-requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/installation-requirements)
- Rancher — *Helm version requirements*: [https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/helm-version-requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/helm-version-requirements)
- Rancher — *Install/upgrade on a Kubernetes cluster*: [https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster/](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster/)
- cert-manager — *Supported releases*: [https://cert-manager.io/docs/releases/](https://cert-manager.io/docs/releases/)
- cert-manager — *Installation*: [https://cert-manager.io/docs/installation/](https://cert-manager.io/docs/installation/)

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
  - hostname: rancher.example.com
    service: https://<node-ip>:<nodePort-https>
    originRequest:
      noTLSVerify: true
  - service: http_status:404          # catch-all BẮT BUỘC ở cuối
```

```bash
# tạo DNS record cho từng hostname
cloudflared tunnel route dns homelab-k8s app.example.com
cloudflared tunnel route dns homelab-k8s rancher.example.com
# chạy thử
cloudflared tunnel run homelab-k8s
# chạy như service
sudo cloudflared service install
```

> ⚠️ Khi cloudflared chạy **ngoài** cụm, nó **không** resolve được `*.svc.cluster.local`. Khi đó phải expose ingress qua **NodePort** (`service: http://<node-ip>:<nodePort>`) hoặc **MetalLB IP** (Phụ lục B). Đây chính là lý do cách in-cluster ([§12]) gọn hơn — nên ưu tiên [§12].

### Phụ lục B — MetalLB (tuỳ chọn: LoadBalancer IP trong LAN)

Chỉ cần nếu bạn **cũng** muốn truy cập ingress từ LAN bằng một IP cố định (không bắt buộc cho luồng Cloudflare Tunnel).

```bash
# kubeadm mặc định dùng kube-proxy mode iptables: không cần bật strictARP.
# Chỉ bật strictARP nếu bạn đã chủ động chuyển kube-proxy sang IPVS.

# cài bản đã pin
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml
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
kubectl get svc -n traefik           # EXTERNAL-IP giờ nhận 1 IP từ pool
```

### Phụ lục C — Reset / làm lại một node

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d $HOME/.kube/config
sudo systemctl restart containerd kubelet
# worker xong có thể join lại; master init lại từ §6
```

---

*Runbook tạo ngày 2026-06-26; cập nhật đồng bộ ngày 2026-07-26. Baseline: Ubuntu 24.04 amd64, Kubernetes **v1.35.6** (`1.35.6-1.1`), containerd **2.x** từ Ubuntu, Flannel **v0.28.7**, Traefik chart **41.0.2** / Proxy **v3.7.6**, cert-manager **v1.21.0**, Rancher **2.14.3**. Đây là homelab baseline, không phải SLA/certification production end-to-end cho kubeadm.*
