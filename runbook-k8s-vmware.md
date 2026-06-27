# Runbook: Dựng K8s trên VMware (1 master + 2 worker) → Rancher → Cloudflare Tunnel → Domain

> **Mục tiêu:** Từ con số 0, dựng 3 VM Ubuntu Server 24.04 trên VMware (máy host Windows), cài cụm Kubernetes bằng `kubeadm`, cài **Rancher** để quản trị bằng UI, mua + cấu hình domain, tạo Cloudflare Tunnel và trỏ domain vào đúng ứng dụng chạy trong cụm — **không cần IP public, không cần port-forward, không cần gọi nhà mạng**.
>
> **Đối tượng:** lab/homelab cá nhân. Các bước có thể copy-paste.
>
> **Nguồn chính thức đã tham chiếu** (xem [§15](#15-nguồn-official)): kubeadm, container runtimes, ingress-nginx, MetalLB, Cloudflare Tunnel, Rancher.

---

## Mục lục

1. [Tổng quan &amp; kiến trúc](#1-tổng-quan--kiến-trúc)
2. [Quy hoạch (versions, IP, hostname)](#2-quy-hoạch)
3. [Tạo 3 VM Ubuntu 24.04 trên VMware](#3-tạo-3-vm-ubuntu-2404-trên-vmware)
4. [Cấu hình OS chung (CHẠY TRÊN CẢ 3 NODE)](#4-cấu-hình-os-chung-chạy-trên-cả-3-node)
5. [Khởi tạo control plane (CHỈ MASTER)](#5-khởi-tạo-control-plane-chỉ-master)
6. [Join worker (CHỈ 2 WORKER)](#6-join-worker-chỉ-2-worker)
7. [Verify cụm](#7-verify-cụm)
8. [Cài Ingress Controller (ingress-nginx)](#8-cài-ingress-controller-ingress-nginx)
9. [Deploy app mẫu + Ingress](#9-deploy-app-mẫu--ingress)
10. [Mua &amp; cấu hình domain trên Cloudflare](#10-mua--cấu-hình-domain-trên-cloudflare)
11. [Tạo Cloudflare Tunnel (chạy trong cụm)](#11-tạo-cloudflare-tunnel-chạy-trong-cụm)
12. [Trỏ domain &amp; kiểm tra trên Internet](#12-trỏ-domain--kiểm-tra-trên-internet)
13. [Cài Rancher 2.13.4 &amp; quản lý cụm](#13-cài-rancher-2134--quản-lý-cụm)
14. [Vận hành &amp; troubleshooting](#14-vận-hành--troubleshooting)
15. [Nguồn official](#15-nguồn-official)
16. [Phụ lục](#16-phụ-lục)

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
   │   [Pod cloudflared] ──► Service: ingress-nginx (ClusterIP) │    │
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
- **cloudflared chạy *trong* cụm** và trỏ thẳng vào **Service ClusterIP** của ingress-nginx qua DNS nội bộ → **không cần MetalLB, không cần NodePort, không cần LoadBalancer external IP**.
- **ingress-nginx** làm điểm vào duy nhất, định tuyến theo hostname → host nhiều app/nhiều domain (kể cả **UI Rancher**) chỉ bằng cách thêm `Ingress` + một public hostname trong tunnel.
- **Rancher** cài *vào chính cụm kubeadm*, quản trị cụm đó (hiện trong UI là cụm `local`) bằng giao diện. Phiên bản k8s bị **pin ở 1.34** vì Rancher 2.13.4 chỉ chứng nhận tới đó (xem [§2.1](#21-phiên-bản-ghim-để-khỏi-lệch-version-skew)).

---

## 2. Quy hoạch

### 2.1. Phiên bản (ghim để khỏi lệch version-skew)

| Thành phần       | Phiên bản dùng trong runbook                      | Ghi chú                                                                                                                                                                                                                                        |
| ------------------ | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ubuntu Server      | **24.04.x LTS (Noble)**                        | Cài bản Server, không cần GUI                                                                                                                                                                                                               |
| Kubernetes         | **v1.34** — patch mới nhất **1.34.8** | Pin ở**1.34** vì **Rancher 2.13.4 chỉ chứng nhận tới k8s 1.34** (1.35 chưa Rancher nào support). Repo `pkgs.k8s.io` dùng `v1.34`. Kiểm tra patch tại [https://kubernetes.io/releases/](https://kubernetes.io/releases/) |
| Container runtime  | **containerd 1.7.x** (gói Ubuntu)             | cgroup driver =`systemd`                                                                                                                                                                                                                      |
| CNI                | **Flannel** (pod CIDR `10.244.0.0/16`)       | Chọn Flannel để**không đụng** dải LAN `192.168.x`                                                                                                                                                                                |
| Ingress            | **ingress-nginx** (chart mới nhất)           |                                                                                                                                                                                                                                                 |
| Tunnel             | **cloudflared** (mới nhất)                   | chạy như Deployment trong cụm                                                                                                                                                                                                                |
| Cluster management | **Rancher 2.13.4** (chart `rancher-latest`)  | Bản 2.13.x mới nhất; chứng nhận k8s**1.32–1.34** (RKE2/K3s + imported). Cài bằng Helm vào chính cụm kubeadm                                                                                                                    |
| cert-manager       | **mới nhất** (vd v1.16.x)                    | Rancher cần để tự cấp TLS nội bộ khi`ingress.tls.source=rancher`                                                                                                                                                                       |

> ⚠️ **Version-skew:** `kubelet`/`kubeadm`/`kubectl` nên cùng minor (v1.34). `kubelet` được phép thấp hơn control plane tối đa 1 minor, không bao giờ cao hơn.
>
> ⚠️ **Vì sao pin 1.34 chứ không 1.35:** Rancher luôn đi sau K8s — bản Rancher mới nhất **2.13.4** chỉ chứng nhận tới **k8s 1.34**; **1.35 chưa có Rancher nào support**. Cài 1.35 rồi gắn Rancher = rơi vào trạng thái *unsupported* (dễ vỡ addon/marketplace). **Vòng đời:** k8s 1.34 EOL ~**27/10/2026** (runway ngắn) → khi **Rancher 2.14+** ra (support 1.35/1.36) thì **upgrade Rancher TRƯỚC, rồi mới `kubeadm upgrade`** lên minor kế tiếp, từng bậc một.

### 2.2. IP & hostname (ví dụ — đổi theo dải LAN của bạn)

Giả sử LAN nhà bạn là `192.168.100.0/24`, gateway `192.168.100.1`. Chọn IP tĩnh **ngoài dải DHCP** của router.

| Vai trò      | Hostname        | IP tĩnh            | vCPU                      | RAM  | Disk  |
| ------------- | --------------- | ------------------- | ------------------------- | ---- | ----- |
| Control plane | `k8s-master`  | `192.168.100.101` | **2** (tối thiểu) | 4 GB | 40 GB |
| Worker 1      | `k8s-worker1` | `192.168.100.102` | 2                         | 4 GB | 40 GB |
| Worker 2      | `k8s-worker2` | `192.168.100.103` | 2                         | 4 GB | 40 GB |

> Pod CIDR `10.244.0.0/16` và Service CIDR mặc định `10.96.0.0/12` **không** được trùng dải LAN `192.168.100.0/24` → an toàn.
>
> ⚠️ **RAM khi có Rancher:** Rancher + cert-manager chiếm thêm tài nguyên. Master/worker nên **≥ 4 GB** (Rancher khuyến nghị), tránh để 2 GB như cụm thuần.

### 2.2.1. Kiểm tra IP tĩnh KHÔNG trùng dải DHCP của router (làm 1 lần, trước khi cài)

> Vì sao: IP tĩnh `.101–.103` ở trên là **ví dụ**. Nếu chúng vô tình rơi **trong** dải mà router tự cấp (DHCP pool), router có thể cấp cùng IP đó cho một thiết bị khác → **trùng IP, cụm chập chờn**. Phải đảm bảo `.101/.102/.103` nằm **ngoài** dải DHCP. Làm **một** trong hai cách dưới (Cách A chắc chắn hơn).

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
5. Đối chiếu `.101 / .102 / .103`:
   - **Nằm NGOÀI** dải đó → dùng được luôn, sang [§2.3](#23-yêu-cầu-máy-host).
   - **Nằm TRONG** dải (vd pool `.100–.200` thì `.101–.103` bị dính) → chọn 1 trong 2:
     - **(a)** Đổi IP tĩnh sang dải cao ngoài pool, vd `.201/.202/.203`, **rồi sửa đồng bộ** ở [bảng §2.2](#22-ip--hostname-ví-dụ--đổi-theo-dải-lan-của-bạn), [§4.1 `/etc/hosts`](#41-hostname--etchosts), [§4.2 netplan](#42-ip-tĩnh-netplan), [§5 `kubeadm init`](#5-khởi-tạo-control-plane-chỉ-master). **Hoặc**
     - **(b)** Thu hẹp **End IP** của pool trên router (vd kéo về `.99`) để chừa `.101–.103` ra ngoài.

**Cách B — Không vào được router? Ping thử xem IP có đang bị dùng (nhanh, kém chắc hơn):**

Trên host, ping từng IP định gán — IP "không ai trả lời" là khả năng đang trống:

```powershell
ping -n 2 192.168.100.101
ping -n 2 192.168.100.102
ping -n 2 192.168.100.103
```

- **"Request timed out" / "Destination host unreachable"** trên cả 2 gói → IP đang trống, nhiều khả năng dùng được.
- **"Reply from 192.168.100.x..."** → IP đang bị thiết bị khác chiếm → chọn IP khác.

> ⚠️ Cách B chỉ kiểm tra phụ: một thiết bị đang **tắt** sẽ không trả lời ping nhưng router vẫn có thể đã giữ/cấp lại IP đó. Cách chắc chắn nhất vẫn là **Cách A**. Chắc ăn nhất: chọn IP **ngoài pool** (Cách A) *hoặc* tạo **DHCP Reservation** trên router (gán cứng `.101–.103` theo MAC của từng VM) để router không bao giờ cấp trùng.

> ✅ Sau khi chốt được dải IP an toàn, ghi đè lại 3 IP đó vào [bảng §2.2](#22-ip--hostname-ví-dụ--đổi-theo-dải-lan-của-bạn) và dùng xuyên suốt runbook. **Tất cả** chỗ có `192.168.100.10x` phải khớp nhau.

### 2.3. Yêu cầu máy host

- RAM host nên ≥ 16 GB (3 VM × 4 GB + Windows).
- VMware Workstation Pro/Player (Player đủ dùng cho lab).
- ISO Ubuntu Server 24.04: [https://ubuntu.com/download/server](https://ubuntu.com/download/server)

---

## 3. Tạo 3 VM Ubuntu 24.04 trên VMware

Làm **3 lần** (master, worker1, worker2), chỉ khác tên + tài nguyên theo bảng [§2.2](#22-ip--hostname-ví-dụ--đổi-theo-dải-lan-của-bạn).

1. **Create a New Virtual Machine** → *Typical* → chọn ISO Ubuntu Server 24.04.
2. Đặt tên VM (vd `k8s-master`), chọn thư mục lưu.
3. Disk size 40 GB, *Store as a single file*.
4. **Customize Hardware:**
   - **Memory**: 4096 MB.
   - **Processors**: 2 (master **bắt buộc ≥ 2**).
   - **Network Adapter** → **Bridged** → tick *Replicate physical network connection state*.
     - ⚠️ **Virtual Network Editor là cửa sổ KHÁC với VM Settings.** Mở từ **menu cửa sổ chính VMware → Edit → Virtual Network Editor** (KHÔNG phải trong Settings của VM). Nếu các ô bị mờ → bấm **Change Settings** (cần quyền admin/UAC).
     - Chọn dòng **VMnet0** (Type = *Bridged*) → ở mục **"Bridged to:"** đổi từ *Automatic* sang **đúng tên card vật lý đang có mạng** (vd `Realtek PCIe GbE Family Controller` nếu dùng dây LAN, hoặc card `...Wi-Fi...` nếu dùng Wi-Fi) → **Apply → OK**. Để *Automatic* dễ bị bind nhầm card sau reboot.
     - **KHÔNG chọn** các card ảo trong danh sách: `Hyper-V Virtual Ethernet Adapter`, `TAP-Windows Adapter ... OpenVPN`… (không phải card mạng thật).
     - Không chắc card nào đang chạy? Trên host chạy `ipconfig /all`, tìm adapter có **IPv4 Address** dạng `192.168.x.x` + có **Default Gateway** → lấy đúng tên ở dòng *Description*.
     - 💡 **Nếu host bật Hyper-V** (thấy "Hyper-V Virtual Ethernet Adapter" trong danh sách "Bridged to") thì bridge VMware có thể bị Hyper-V chiếm card → DHCP không ra IP dù chọn đúng card. Cách xử lý ở [§14](#14-vận-hành--troubleshooting).
5. Power On → cài Ubuntu Server:
   - Chọn **Ubuntu Server (minimized hoặc full)**.
   - Network: cứ để DHCP khi cài, ta sẽ đặt IP tĩnh sau ([§4.2](#42-ip-tĩnh-netplan)).
   - **Profile**: tạo user (vd `k8sadmin`), đặt hostname đúng (`k8s-master`…).
   - **Tick "Install OpenSSH server"** để SSH vào cho tiện.
   - Bỏ qua các snap đề xuất.
6. Cài xong → reboot → đăng nhập.

> 💡 Sau khi cài, nên SSH từ máy host vào từng VM (`ssh k8sadmin@192.168.100.101`) để copy-paste lệnh dễ hơn.

---

## 4. Cấu hình OS chung (CHẠY TRÊN CẢ 3 NODE)

> Toàn bộ [§4] phải chạy **giống nhau trên cả 3 máy** (trừ phần hostname/IP là riêng từng máy). Chạy bằng `sudo` hoặc `sudo -i`.

### 4.1. Hostname + /etc/hosts

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
192.168.100.101 k8s-master
192.168.100.102 k8s-worker1
192.168.100.103 k8s-worker2
EOF
```

### 4.2. IP tĩnh (netplan)

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
      addresses: [192.168.100.101/24]   # ⚠️ .101 master / .102 w1 / .103 w2
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

> ⚠️ Nếu đang làm qua SSH, sau `netplan apply` phiên sẽ **đứng/rớt** vì IP đã đổi — kết nối lại bằng IP mới: `ssh <user>@192.168.100.101`. Khi chỉ chỉnh nhỏ, có thể dùng `sudo netplan try` (tự hoàn tác sau 120s nếu mất mạng).

**5) Kiểm tra (cả 3 máy đều phải đạt):**

```bash
ip -br a                       # card hiển thị đúng IP tĩnh .101/.102/.103
ping -c2 192.168.100.1         # tới gateway → phải có reply
ping -c2 8.8.8.8               # ra Internet bằng IP → phải có reply
ping -c2 google.com            # phân giải DNS → phải có reply
                               #   (nếu ping 8.8.8.8 OK mà google.com fail = lỗi DNS → xem lại nameservers)
```

> Dùng `routes:` thay cho `gateway4` (đã deprecated từ Ubuntu 22.04+).

### 4.3. Cập nhật hệ thống & tắt swap

`kubeadm` yêu cầu **tắt swap**:

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo swapoff -a
# tắt swap vĩnh viễn (comment dòng swap trong fstab):
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab
# Ubuntu 24.04 có thể có swap.img qua cloud-init/zram — vô hiệu nếu còn:
sudo systemctl mask swap.target 2>/dev/null || true
```

Kiểm tra: `free -h` → dòng Swap phải = 0.

**Đồng bộ thời gian (quan trọng cho etcd/cert):** lệch giờ giữa các node làm hỏng TLS handshake & etcd. Ubuntu bật `systemd-timesyncd` sẵn — chỉ cần xác nhận:

```bash
timedatectl                 # phải thấy: System clock synchronized: yes  +  NTP service: active
sudo timedatectl set-ntp true   # bật nếu chưa active
```

### 4.4. Kernel modules + sysctl (cho CRI & bridge networking)

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

# verify (cả 3 phải = 1):
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

### 4.5. Cài containerd + bật SystemdCgroup

```bash
sudo apt-get install -y containerd

# sinh config mặc định rồi bật cgroup driver = systemd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd

# verify SystemdCgroup = true
grep SystemdCgroup /etc/containerd/config.toml
```

> ⚠️ **Bắt buộc:** cgroup driver của containerd **phải trùng** với kubelet. Với cgroup v2 (mặc định Ubuntu 24.04), `systemd` là khuyến nghị và `kubeadm` cũng mặc định dùng `systemd` cho kubelet → khớp.

### 4.6. Cài kubeadm, kubelet, kubectl (repo pkgs.k8s.io, pin v1.34)

> Repo cũ `apt.kubernetes.io` đã **deprecated từ 13/09/2023** — bắt buộc dùng `pkgs.k8s.io`.

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p -m 755 /etc/apt/keyrings

# Repo theo MINOR v1.34 — vì Rancher 2.13.4 chỉ support tới k8s 1.34
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

# (KHUYẾN NGHỊ) pin đúng patch 1.34.8 cho cả 3 node để khỏi lệch nhau.
# Xem hậu tố patch có sẵn (vd 1.34.8-1.1):  apt-cache madison kubeadm
VER='1.34.8-1.1'    # ⚠️ kiểm tra lại hậu tố bằng `apt-cache madison kubeadm`
sudo apt-get install -y kubelet="$VER" kubeadm="$VER" kubectl="$VER"
# (hoặc lấy patch 1.34 mới nhất, không pin: sudo apt-get install -y kubelet kubeadm kubectl)

sudo apt-mark hold kubelet kubeadm kubectl   # ghim version, tránh apt upgrade làm vỡ skew
sudo systemctl enable --now kubelet
```

> `kubelet` lúc này sẽ ở trạng thái crashloop cho tới khi `kubeadm init`/`join` — **bình thường**.

### 4.7. Firewall

**Lab nhanh:** có thể tắt cho đỡ vướng (mạng nội bộ tin cậy):

```bash
sudo ufw disable
```

**Hoặc** mở đúng cổng nếu muốn giữ firewall:

| Node                    | Cổng cần mở                                                                                                  |
| ----------------------- | --------------------------------------------------------------------------------------------------------------- |
| master                  | TCP`6443` (API), `2379-2380` (etcd), `10250` (kubelet), `10257` (controller-mgr), `10259` (scheduler) |
| worker                  | TCP`10250` (kubelet), `30000-32767` (NodePort)                                                              |
| cả hai (Flannel VXLAN) | **UDP `8472`**                                                                                          |

---

## 5. Khởi tạo control plane (CHỈ MASTER)

Chạy **chỉ trên `k8s-master`**.

(Tùy chọn) Kéo image trước để lộ sớm lỗi mạng/registry và để `init` nhanh hơn:

```bash
sudo kubeadm config images pull
```

Khởi tạo control plane (lệnh chạy **vài phút** — kéo image + dựng etcd/control plane, **đừng ngắt giữa chừng**):

```bash
sudo kubeadm init \
  --control-plane-endpoint "k8s-master:6443" \
  --apiserver-advertise-address 192.168.100.101 \
  --pod-network-cidr 10.244.0.0/16
```

- `--control-plane-endpoint k8s-master:6443`: dùng tên (đã map trong `/etc/hosts`) → thuận lợi cho HA/đổi IP sau này.
- `--apiserver-advertise-address`: IP master (tránh kubeadm tự đoán nhầm card).
- `--pod-network-cidr 10.244.0.0/16`: **bắt buộc cho Flannel**.

Khi xong, kubeadm in ra:

1. **Lệnh cấu hình kubeconfig** — chạy ngay (với user thường, không phải root):

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

2. **Lệnh `kubeadm join ...`** cho worker — **COPY LƯU LẠI** (sẽ dùng ở [§6](#6-join-worker-chỉ-2-worker)). Mất thì sinh lại bằng:

```bash
kubeadm token create --print-join-command
```

### 5.1. Cài CNI (Flannel) — chỉ master

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Đợi ~1 phút rồi kiểm tra master chuyển `Ready`:

```bash
kubectl get nodes
kubectl get pods -n kube-flannel
```

> Flannel mặc định dùng pod CIDR `10.244.0.0/16` → khớp `--pod-network-cidr` ở trên. Nếu muốn dùng **Calico** thì phải đặt pod CIDR custom (vd `10.244.0.0/16`) để **không trùng** LAN `192.168.x`.

---

## 6. Join worker (CHỈ 2 WORKER)

Trên **`k8s-worker1`** và **`k8s-worker2`**:

**Kiểm tra trước khi join** (bắt sớm lỗi `/etc/hosts`/firewall — không cần cài thêm gói):

```bash
ping -c2 k8s-master
timeout 3 bash -c 'echo > /dev/tcp/k8s-master/6443' && echo "API 6443 reachable" || echo "API 6443 UNREACHABLE"
```

Rồi chạy lệnh join đã lưu ở [§5] (dạng):

```bash
sudo kubeadm join k8s-master:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

> Worker phải resolve được `k8s-master` (đã thêm ở `/etc/hosts` [§4.1]).

---

## 7. Verify cụm

Trên **master**:

```bash
kubectl get nodes -o wide
# cả 3 node phải STATUS = Ready, ROLES: control-plane / <none>

kubectl get pods -A
# tất cả pod kube-system + kube-flannel phải Running
```

(Tuỳ chọn) Cho phép chạy pod cả trên control-plane (lab nhỏ):

```bash
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule-
```

---

## 8. Cài Ingress Controller (ingress-nginx)

Cài bằng **Helm** (luôn lấy chart mới, đỡ lệch version). Trên master:

```bash
# cài Helm nếu chưa có
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

Kiểm tra:

```bash
kubectl get pods -n ingress-nginx
kubectl get svc  -n ingress-nginx
```

**Đợi controller sẵn sàng trước khi tạo Ingress ở [§9]** (nếu không, apply Ingress hay lỗi *admission webhook chưa ready*):

```bash
kubectl -n ingress-nginx wait --for=condition=ready pod \
  -l app.kubernetes.io/component=controller --timeout=180s
```

> Service `ingress-nginx-controller` mặc định type `LoadBalancer`. Trên bare-metal, **EXTERNAL-IP sẽ `<pending>` mãi — KHÔNG SAO**: ta không dùng external IP đó. cloudflared sẽ gọi vào **ClusterIP** qua DNS nội bộ:
> `http://ingress-nginx-controller.ingress-nginx.svc.cluster.local:80`
>
> Nếu *cũng* muốn truy cập từ LAN bằng IP (không bắt buộc), xem [Phụ lục B — MetalLB](#phụ-lục-b--metallb-tuỳ-chọn-loadbalancer-ip-trong-lan).

---

## 9. Deploy app mẫu + Ingress

> Toàn bộ [§9] chạy **trên master** (nơi có `kubectl`). File `.yaml` tạo ở thư mục home của user (vd `~/demo-app.yaml`).

Tạo file `demo-app.yaml` (đổi `app.example.com` thành domain thật của bạn — sẽ mua ở [§10]):

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
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"   # TLS do Cloudflare lo ở edge
spec:
  ingressClassName: nginx
  rules:
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
ING_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.spec.clusterIP}')
curl -s -H "Host: app.example.com" http://$ING_IP/
# → phải thấy "Server address..." từ nginx hello
```

Thấy output nghĩa là chuỗi **ingress → service → pod** đã chạy. Giờ chỉ còn nối Internet vào.

---

## 10. Mua & cấu hình domain trên Cloudflare

### 10.1. Mua domain

Mua ở bất kỳ registrar nào (Cloudflare Registrar, Namecheap, GoDaddy, PA Vietnam…). Domain rẻ (`.com`, `.dev`, `.xyz`) đều được.

### 10.2. Thêm domain vào Cloudflare

1. Tạo tài khoản [https://dash.cloudflare.com](https://dash.cloudflare.com) → **Add a site** → nhập domain.
2. Chọn plan **Free**.
3. Cloudflare cấp **2 nameserver** (vd `xxx.ns.cloudflare.com`).
4. Vào trang quản trị của **registrar** → đổi **nameservers** của domain sang 2 NS Cloudflare vừa cấp.
5. Đợi DNS propagate (vài phút → tối đa 24h). Khi domain ở Cloudflare hiện **Active** là xong.

> Nếu mua luôn bằng **Cloudflare Registrar** thì bước đổi nameserver được làm sẵn.

---

## 11. Tạo Cloudflare Tunnel (chạy trong cụm)

Dùng **remotely-managed tunnel** (quản lý qua dashboard bằng *token*) — đơn giản, hợp với cụm k8s và đúng theo deployment guide của Cloudflare. (Cách CLI `config.yml` xem [Phụ lục A](#phụ-lục-a--locally-managed-tunnel-cli--configyml).)

### 11.1. Tạo tunnel trên dashboard → lấy token

1. Dashboard → **Zero Trust** → **Networks → Tunnels** → **Create a tunnel**.
2. Chọn loại **Cloudflared** → đặt tên (vd `homelab-k8s`) → **Save**.
3. Màn hình *Install and run* hiện lệnh có dạng `cloudflared ... run --token eyJh...`. **Copy chuỗi token `eyJh...`** (rất dài). Chưa cần chạy lệnh đó — ta sẽ nhúng token vào k8s.

### 11.2. Deploy cloudflared vào cụm

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
          args: ["tunnel", "--no-autoupdate", "run"]
          env:
            - name: TUNNEL_TOKEN
              valueFrom:
                secretKeyRef:
                  name: tunnel-token
                  key: token
```

```bash
kubectl apply -f cloudflared.yaml
kubectl get pods -n cloudflare
```

Quay lại dashboard tunnel → trạng thái phải chuyển **Healthy** (2 connector).

### 11.3. Thêm Public Hostname (route domain → ingress nội bộ)

Trong tunnel vừa tạo → tab **Published application routes** (hoặc **Public Hostname**) → **Add a public hostname**:

| Trường            | Giá trị                                                       |
| ------------------- | --------------------------------------------------------------- |
| **Subdomain** | `app` (để trống nếu dùng root domain)                    |
| **Domain**    | `example.com` (chọn từ dropdown)                            |
| **Type**      | `HTTP`                                                        |
| **URL**       | `ingress-nginx-controller.ingress-nginx.svc.cluster.local:80` |

→ **Save**. Cloudflare **tự tạo bản ghi DNS (CNAME)** `app.example.com → <tunnel-id>.cfargotunnel.com`. Bạn **không phải** tự thêm DNS record.

> Vì cloudflared chạy *trong* cụm, nó resolve được tên DNS nội bộ `*.svc.cluster.local` và gọi thẳng ClusterIP của ingress. Hostname `app.example.com` được giữ nguyên trong Host header → ingress-nginx khớp đúng `Ingress` rule ở [§9].
>
> (Tuỳ chọn) Nếu cần ép Host header, mở **Additional application settings → HTTP Settings → HTTP Host Header** = `app.example.com`.

---

## 12. Trỏ domain & kiểm tra trên Internet

```bash
# 1) DNS đã có CNAME chưa
nslookup app.example.com        # phải trả về IP của Cloudflare (proxied)

# 2) Truy cập (TLS do Cloudflare cấp tự động ở edge)
curl -I https://app.example.com
```

Mở trình duyệt → `https://app.example.com` → thấy trang `nginxdemos/hello` ⇒ **toàn bộ chuỗi đã thông**:

```
Browser → Cloudflare edge (HTTPS) → Tunnel → cloudflared(pod) →
ingress-nginx(ClusterIP) → Ingress(host match) → Service web → Pod web
```

> **TLS:** Cloudflare tự lo cert ở edge (Internet ⇄ Cloudflare = HTTPS). Đoạn Cloudflare ⇄ cụm đi trong tunnel mã hoá; ingress để HTTP (không cần cert nội bộ). Đây là lý do `ssl-redirect: "false"` ở Ingress.

---

## 13. Cài Rancher 2.13.4 & quản lý cụm

> Cài Rancher **vào chính cụm kubeadm** vừa dựng (Rancher chạy như workload trong namespace `cattle-system`). Cụm host nó sẽ tự xuất hiện trong UI Rancher dưới tên **`local`** — **không cần import thủ công**. Muốn quản thêm cụm khác sau này thì dùng **Cluster Management → Import Existing**.
>
> **Yêu cầu trước:** đã xong [§7](#7-verify-cụm) (cụm Ready), [§8](#8-cài-ingress-controller-ingress-nginx) (ingress-nginx) và [§11](#11-tạo-cloudflare-tunnel-chạy-trong-cụm) (tunnel Healthy). Chọn hostname cho Rancher, vd `rancher.example.com`.

### 13.1. Cài cert-manager (Rancher cần để cấp TLS nội bộ)

Trên **master**:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true

kubectl rollout status deploy/cert-manager -n cert-manager
kubectl get pods -n cert-manager     # 3 pod Running
```

### 13.2. Cài Rancher (Helm, pin 2.13.4)

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
kubectl create namespace cattle-system

helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --version 2.13.4 \
  --set hostname=rancher.example.com \
  --set bootstrapPassword='ĐẶT-MẬT-KHẨU-ADMIN-MẠNH' \
  --set replicas=1 \
  --set ingress.tls.source=rancher

# đợi rollout (kéo image, có thể vài phút)
kubectl -n cattle-system rollout status deploy/rancher
kubectl -n cattle-system get pods
```

- `--version 2.13.4`: ghim đúng bản trong support matrix (k8s ≤ 1.34).
- `replicas=1`: homelab 1 cụm (mặc định chart là 3 — quá tải cho lab nhỏ).
- `ingress.tls.source=rancher`: Rancher tự sinh cert self-signed qua cert-manager; TLS đi **tới tận** Rancher (tránh lỗi redirect-loop). Rancher tạo sẵn một `Ingress` cho `rancher.example.com` trên ingress-nginx (cổng 443).

> **Thay thế (nhẹ hơn, bỏ cert-manager):** `--set tls=external` → Rancher chạy HTTP thuần, TLS chỉ chấm dứt ở Cloudflare edge; tunnel khi đó trỏ HTTP→`...:80`. Nhược điểm: phải đảm bảo header `X-Forwarded-Proto: https` (Cloudflare có set sẵn) nếu không dễ redirect-loop. Runbook này dùng `ingress.tls.source=rancher` cho chắc.

### 13.3. Mở Rancher UI ra Internet qua tunnel

Thêm **một Public Hostname nữa** trong cùng tunnel đã tạo ở [§11.3](#113-thêm-public-hostname-route-domain--ingress-nội-bộ):

| Trường                                              | Giá trị                                                        |
| ----------------------------------------------------- | ---------------------------------------------------------------- |
| **Subdomain**                                   | `rancher`                                                      |
| **Domain**                                      | `example.com`                                                  |
| **Type**                                        | **HTTPS**                                                  |
| **URL**                                         | `ingress-nginx-controller.ingress-nginx.svc.cluster.local:443` |
| **Additional settings → TLS → No TLS Verify** | **ON** (vì cert Rancher là self-signed)                  |
| **Additional settings → HTTP Host Header**     | `rancher.example.com`                                          |

→ **Save**. Cloudflare tự tạo DNS CNAME cho `rancher.example.com`.

### 13.4. Đăng nhập lần đầu

```bash
# nếu quên/không set bootstrapPassword, lấy lại:
kubectl get secret --namespace cattle-system bootstrap-secret \
  -o go-template='{{ .data.bootstrapPassword | base64decode }}{{ "\n" }}'
```

1. Mở `https://rancher.example.com` → đăng nhập bằng bootstrap password.
2. Đặt mật khẩu admin mới.
3. Xác nhận **Server URL** = `https://rancher.example.com` (Rancher gợi ý sẵn).
4. Vào **Cluster Management** → thấy cụm **`local`** = chính cụm kubeadm của bạn, trạng thái **Active**.

### 13.5. Kiểm tra

- UI hiện cụm `local` Active, đủ 3 node.
- Quản lý workload / Helm app / monitoring qua UI; cụm vẫn dùng song song bằng `kubectl`.

> ⚠️ **Lịch upgrade (đọc kỹ):** khi **Rancher 2.14+** ra và chứng nhận k8s 1.35/1.36 → **`helm upgrade` Rancher TRƯỚC**, đối chiếu support matrix, **rồi** mới `kubeadm upgrade` lên minor kế tiếp (từng bậc 1.34→1.35→1.36). **Không bao giờ** đẩy k8s vượt trần mà bản Rancher đang chạy hỗ trợ — xem lý do ở [§14](#14-vận-hành--troubleshooting).

---

## 14. Vận hành & troubleshooting

### Thêm app/domain mới sau này

Chỉ cần: (1) `Deployment`+`Service`+`Ingress` mới với `host: app2.example.com`; (2) thêm **Public Hostname** mới trong cùng tunnel trỏ về cùng `ingress-nginx-controller...:80`. Xong. Không đụng gì tới VM/router.

### Vì sao phải tôn trọng EOL & upgrade đúng nhịp

Cụm **vẫn chạy** sau ngày EOL của k8s, nhưng dừng nhận **patch bảo mật (CVE)** + bug fix; kubeadm **không skip được minor** (phải nhảy từng bậc) nên để càng lâu càng khó nâng; và hệ sinh thái (CNI/ingress/cloudflared/**Rancher**) dần bỏ hỗ trợ bản cũ. Với cụm **phơi ra Internet qua tunnel**, rủi ro CVE chưa vá là nghiêm trọng nhất → nâng đều, luôn nằm trong vùng Rancher support.

### Bảng lỗi thường gặp

| Triệu chứng                                            | Nguyên nhân & cách xử lý                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| -------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cài Ubuntu:**DHCPv4 quay mãi, không ra IP**     | VMnet0 bridge bind nhầm card →**Edit → Virtual Network Editor → VMnet0**, đổi *Automatic* sang đúng card vật lý ([§3](#3-tạo-3-vm-ubuntu-2404-trên-vmware)). Xác nhận card đang chạy bằng `ipconfig /all` trên host. Tạm thời có thể đặt IP tĩnh ngay tại màn hình installer.                                                                                                                                                                                                                        |
| Bridge**vẫn** không ra IP dù chọn đúng card  | Host bật**Hyper-V** làm VMware bridge xung đột (danh sách "Bridged to" có "Hyper-V Virtual Ethernet Adapter"). Kiểm tra host: `bcdedit /enum \| findstr -i hypervisorlaunchtype`. Xử lý: tắt Hyper-V cho lab (`bcdedit /set hypervisorlaunchtype off` + reboot — **lưu ý sẽ tắt WSL2/Docker Desktop/Sandbox**), **hoặc** chuyển Network Adapter sang **NAT** (vẫn ra Internet; khi đó IP tĩnh phải theo dải NAT của VMnet8, vd `192.168.71.x`, và đổi đồng bộ toàn runbook). |
| Node`NotReady`                                         | CNI chưa chạy →`kubectl get pods -n kube-flannel`; kiểm tra UDP 8472 mở; `--pod-network-cidr` đúng `10.244.0.0/16`                                                                                                                                                                                                                                                                                                                                                                                                          |
| `kubeadm join` timeout                                 | Worker không resolve`k8s-master` (thiếu `/etc/hosts`); hoặc cổng `6443` master bị firewall chặn                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Pod kẹt`ContainerCreating`/CNI lỗi                   | sai cgroup driver →`grep SystemdCgroup /etc/containerd/config.toml` phải `true`, rồi `systemctl restart containerd kubelet`                                                                                                                                                                                                                                                                                                                                                                                                     |
| `kubelet` crashloop sau init                           | còn swap →`swapoff -a` + check `/etc/fstab`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Tunnel không**Healthy**                           | token sai/thiếu →`kubectl logs -n cloudflare deploy/cloudflared`; pod phải `Running`                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Domain mở ra**502/error 1033**                    | URL route sai tên service; thử đúng`ingress-nginx-controller.ingress-nginx.svc.cluster.local:80` (hoặc `:443` cho Rancher); kiểm tra `kubectl get svc -n ingress-nginx`                                                                                                                                                                                                                                                                                                                                                      |
| Mở domain ra**404 từ nginx**                     | Host header không khớp Ingress`host:` → set HTTP Host Header trong tunnel, hoặc sửa `host:` trong Ingress                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `rancher.example.com` lỗi **TLS/redirect-loop** | thiếu**No TLS Verify=ON** + **Host Header**=`rancher.example.com` ở route HTTPS; hoặc cert-manager chưa Ready (`kubectl get pods -n cert-manager`)                                                                                                                                                                                                                                                                                                                                                                   |
| Rancher pod`CrashLoop`/Pending                         | thiếu RAM (nâng VM ≥ 4 GB); hoặc cert-manager CRDs chưa cài (`--set crds.enabled=true`)                                                                                                                                                                                                                                                                                                                                                                                                                                          |
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

## 15. Nguồn official

- Kubernetes — *Creating a cluster with kubeadm*: [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- Kubernetes — *Installing kubeadm* (repo pkgs.k8s.io): [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- Kubernetes — *Container runtimes* (containerd, SystemdCgroup, sysctl): [https://kubernetes.io/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- Kubernetes — *Releases* (phiên bản stable + EOL): [https://kubernetes.io/releases/](https://kubernetes.io/releases/)
- Kubernetes — *pkgs.k8s.io* (repo mới): [https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/)
- ingress-nginx — *Bare-metal considerations*: [https://kubernetes.github.io/ingress-nginx/deploy/baremetal/](https://kubernetes.github.io/ingress-nginx/deploy/baremetal/)
- MetalLB — *Installation*: [https://metallb.universe.tf/installation/](https://metallb.universe.tf/installation/)
- Cloudflare — *Tunnel setup*: [https://developers.cloudflare.com/tunnel/setup/](https://developers.cloudflare.com/tunnel/setup/)
- Cloudflare — *Create a locally-managed tunnel (CLI)*: [https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-local-tunnel/](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-local-tunnel/)
- Cloudflare — *Deploy cloudflared in Kubernetes*: [https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/deployment-guides/kubernetes/](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/deployment-guides/kubernetes/)
- Rancher — *Support matrix v2.13.4*: [https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-13-4/](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-13-4/)
- Rancher — *Install on a Kubernetes cluster (Helm)*: [https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster/install-rancher-ha](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster/install-rancher-ha)

---

## 16. Phụ lục

### Phụ lục A — Locally-managed tunnel (CLI / config.yml)

Phương án thay thế cho [§11], chạy `cloudflared` **trực tiếp trên 1 máy** (vd master) thay vì bằng token trong cụm. Dùng khi muốn quản lý cấu hình tunnel bằng file thay vì dashboard.

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

> ⚠️ Khi cloudflared chạy **ngoài** cụm, nó **không** resolve được `*.svc.cluster.local`. Khi đó phải expose ingress qua **NodePort** (`service: http://<node-ip>:<nodePort>`) hoặc **MetalLB IP** (Phụ lục B). Đây chính là lý do cách in-cluster ([§11]) gọn hơn — nên ưu tiên [§11].

### Phụ lục B — MetalLB (tuỳ chọn: LoadBalancer IP trong LAN)

Chỉ cần nếu bạn **cũng** muốn truy cập ingress từ LAN bằng một IP cố định (không bắt buộc cho luồng Cloudflare Tunnel).

```bash
# bật strictARP cho kube-proxy
kubectl get configmap kube-proxy -n kube-system -o yaml | \
  sed -e "s/strictARP: false/strictARP: true/" | \
  kubectl apply -f - -n kube-system

# cài MetalLB (kiểm tra release mới nhất tại github.com/metallb/metallb)
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
kubectl get svc -n ingress-nginx     # EXTERNAL-IP giờ nhận 1 IP từ pool
```

### Phụ lục C — Reset / làm lại một node

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d $HOME/.kube/config
sudo systemctl restart containerd kubelet
# worker xong có thể join lại; master init lại từ §5
```

---

*Runbook tạo ngày 2026-06-26 (cập nhật: chuyển sang hướng có Rancher). Phiên bản tham chiếu: Ubuntu 24.04, **Kubernetes v1.34.8**, containerd 1.7.x, **Rancher 2.13.4**, cert-manager. Số phiên bản/URL có thể đổi — kiểm tra lại tại các nguồn official ở [§15] nếu lệnh không khớp.*
