# Hướng dẫn dựng Kubernetes Cluster: 1 Control‑Plane + N Worker

> **Mục tiêu:** Dùng (một phần đã được làm sạch của) repo này để dựng một cluster Kubernetes
> **1 control‑plane + N worker** bằng **kubeadm**, runtime **containerd**, CNI **Cilium**.
> Các phiên bản được ghim (pin) ở mức **stable** và **tương thích với Rancher** để sẵn sàng cho các phase Rancher tiếp theo.
>
> Tài liệu này là **bản hướng dẫn từng bước để bạn tự thực hiện** — gồm cả lệnh git, cách tạo branch, và **nội dung đầy đủ của từng file** cần tạo/sửa. Bạn copy‑paste theo là chạy được.

---

## Mục lục

1. [Tổng quan kiến trúc](#1-tổng-quan-kiến-trúc)
2. [Chọn phiên bản — và TẠI SAO](#2-chọn-phiên-bản--và-tại-sao)
3. [Yêu cầu hệ thống (prerequisites)](#3-yêu-cầu-hệ-thống-prerequisites)
4. [Bước 1 — Chuẩn bị Git repo](#4-bước-1--chuẩn-bị-git-repo)
5. [Bước 2 — Tạo branch `1-controlplane-n-worker` và lọc file](#5-bước-2--tạo-branch-1-controlplane-n-worker-và-lọc-file)
6. [Bước 3 — Tạo/sửa các file cấu hình (nội dung đầy đủ)](#6-bước-3--tạosửa-các-file-cấu-hình-nội-dung-đầy-đủ)
7. [Bước 4 — Chạy playbook để dựng cluster](#7-bước-4--chạy-playbook-để-dựng-cluster)
8. [Bước 5 — Kiểm tra cluster](#8-bước-5--kiểm-tra-cluster)
9. [Bước 6 — Commit & push branch](#9-bước-6--commit--push-branch)
10. [Xử lý sự cố & reset](#10-xử-lý-sự-cố--reset)
11. [Chuẩn bị cho Rancher (phase tiếp theo)](#11-chuẩn-bị-cho-rancher-phase-tiếp-theo)
12. [Tài liệu tham khảo chính thức](#12-tài-liệu-tham-khảo-chính-thức)

---

## 1. Tổng quan kiến trúc

```
                         ┌─────────────────────────────┐
   Ansible control node  │  [local] bastion            │  ← máy chạy ansible (Linux/WSL)
   (máy của bạn / WSL)   │  kubectl + ~/.kube/config   │     quản lý cluster từ đây
                         └──────────────┬──────────────┘
                                        │ SSH (user: ansible-user)
                ┌───────────────────────┼────────────────────────┐
                ▼                       ▼                        ▼
      ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
      │ [controlplane]   │    │   [node] worker1 │    │   [node] worker2 │   ... workerN
      │   master1        │    │  containerd      │    │  containerd      │
      │  containerd      │    │  kubelet         │    │  kubelet         │
      │  kubeadm init    │    │  kubeadm join    │    │  kubeadm join    │
      │  etcd + apiserver│    └──────────────────┘    └──────────────────┘
      │  Cilium (Helm)   │
      └──────────────────┘
```

**Luồng cài đặt (do `cluster.yaml` điều phối):**

```
k8s-setup (containerd) → time-sync → kubernetes_role (kubelet/kubeadm/kubectl)
→ helm → first-master (kubeadm init) → join_workers → install_cni (Cilium)
→ kubectl_role (kubeconfig về bastion)
```

> **Lưu ý quan trọng về repo gốc:** repo gốc thực ra thiết kế cho cụm **HA 3 master** bằng cách
> *copy thủ công* static manifest + PKI giữa các master (các file `all.yaml`, `deploy.yaml`,
> `pki-copy.yaml`, `join_masters.yaml`, `front-proxy-ca.crt.yaml`…). Cách đó **không theo chuẩn kubeadm**
> và dễ hỏng etcd. Hướng dẫn này cố tình **bỏ toàn bộ phần HA đó** và chỉ giữ đường đi **1 control‑plane**
> — chắc chắn chạy và là nền tảng sạch để sau này mở rộng HA đúng chuẩn (`--control-plane --certificate-key`).

---

## 2. Chọn phiên bản — và TẠI SAO

Đây là phần quan trọng nhất cho yêu cầu "stable & ready với Rancher". Tất cả số liệu dưới đây được kiểm chứng từ tài liệu chính thức (xem [mục 12](#12-tài-liệu-tham-khảo-chính-thức)), trạng thái tháng 06/2026.

| Thành phần | Phiên bản ghim | Lý do chọn |
|---|---|---|
| **Kubernetes** (kubeadm/kubelet/kubectl) | **`v1.34.6`** (kênh `stable:/v1.34`) | Xem giải thích bên dưới |
| **containerd** | **`1.7.x`** (mới nhất `1.7.32`) qua repo Docker | CRI v1, systemd cgroup, K8s hỗ trợ tới 1.35 |
| **CNI: Cilium** | **`1.17.16`** (Helm chart) | Hỗ trợ K8s 1.31–1.34 |
| **Helm** | **`v3.16.2`** | Dùng để cài Cilium |
| **Rancher (mục tiêu phase sau)** | **`2.14.x`** (mới nhất) | Imported cluster hỗ trợ K8s **1.33–1.35** |
| **OS** | Ubuntu **22.04 / 24.04 LTS** | apt; cgroup v2 mặc định |

### 2.1. Tại sao Kubernetes `1.34`?

- **Khớp ma trận hỗ trợ của Rancher.** Rancher mới nhất (`2.14.2`) chứng nhận imported/downstream Kubernetes trong khoảng **1.33 → 1.35**. Chọn **1.34** nằm *giữa* dải này → an toàn nhất, không sát mép.
- **Không chọn 1.32 trở xuống vì đã EOL.** Kubernetes **1.32 đã hết vòng đời ngày 2026‑02‑28**. Dùng bản EOL nghĩa là không còn vá bảo mật.
- **1.34 vẫn đang được hỗ trợ chính thức (upstream):** EOL **2026‑10‑27**, vào maintenance mode 2026‑08‑27 → còn nhận bản vá, đủ "chín" (ra mắt ~08/2025, đã có nhiều patch).
- **Tương thích ngược toàn bộ stack:** containerd 1.7 và Cilium 1.17 đều hỗ trợ 1.34.
- 👉 *Muốn runway dài hơn?* Đổi sang **1.35** (cũng nằm trong dải Rancher 2.14, EOL ~02/2027). Trong hướng dẫn này chỉ cần đổi 2 biến `kubernetes_minor` + `kubernetes_patch` trong `group_vars/all.yml` (và nhớ dùng Cilium 1.18+ cho 1.35).
- ⚠️ **Nhắc lịch nâng cấp:** 1.34 EOL **2026‑10‑27**. Lên kế hoạch upgrade trước mốc này.

### 2.2. Tại sao containerd `1.7.x` (cài từ repo Docker)?

- **kubeadm cần một CRI runtime hỗ trợ CRI `v1`.** Từ Kubernetes 1.26, kubelet **chỉ** làm việc với CRI v1. containerd 1.7 đáp ứng.
- **cgroup driver phải là `systemd`.** Ubuntu 22.04/24.04 dùng cgroup v2; tài liệu K8s khuyến nghị mạnh dùng `systemd` cho cả kubelet *và* runtime. Nếu lệch (cgroupfs vs systemd) node dễ mất ổn định khi tải cao. → Ta đặt `SystemdCgroup = true` trong containerd và `cgroupDriver: systemd` trong kubelet.
- **Vòng đời còn hợp lệ:** containerd 1.7 được Kubernetes hỗ trợ tới tận **1.35**, và có extended support đến ~09/2026.
- **Cài từ repo apt chính thức của Docker (`containerd.io`)** thay vì gói của distro → **ghim được phiên bản chính xác**, tái lập (reproducible), không phụ thuộc bản phát hành OS.
- 👉 *Hướng tới tương lai:* containerd 1.7 EOL ~09/2026. Khi muốn dùng lâu dài hơn, chuyển **containerd 2.x**. Cách đặt systemd‑cgroup trong hướng dẫn này (thay chuỗi `SystemdCgroup = false` → `true`) **vẫn đúng** cho cả 1.x lẫn 2.x.

### 2.3. Tại sao Cilium `1.17.16`?

- Repo gốc dùng **Cilium 1.16.4 — KHÔNG hỗ trợ K8s 1.34** (1.16 chỉ tới ~1.31). Bắt buộc nâng.
- Cilium **1.17** hỗ trợ Kubernetes **1.31–1.34** → khớp với 1.34. `1.17.16` là patch mới nhất của nhánh 1.17, đang được maintain.
- eBPF, hiệu năng tốt, được Rancher quản lý tốt khi import.

### 2.4. Tại sao cấu hình kubeadm dùng API `v1beta4`?

- Với Kubernetes **1.31+** (gồm 1.34), API cấu hình kubeadm khuyến nghị là **`kubeadm.k8s.io/v1beta4`**. `v1beta3` đã cũ/deprecated. Repo gốc dùng `v1beta3` + `stable-1.28` → ta thay bằng `v1beta4` + version ghim.
- Trong `v1beta4`, `podSubnet` nằm **dưới `networking:`** (không phải top‑level) — một lỗi hay gặp.

### 2.5. Vì sao "hold" (ghim) gói và đặt `controlPlaneEndpoint` cho cả cụm 1 master?

- **`apt-mark hold` / `dpkg hold`:** Kubernetes yêu cầu nâng cấp có chủ đích, đúng quy tắc version‑skew. Ghim gói tránh `apt upgrade` vô tình phá cụm.
- **`controlPlaneEndpoint`:** đặt sẵn một endpoint ổn định (IP master, sau này trỏ LB/DNS) vào chứng chỉ apiserver + kubeconfig → **sau này thêm HA không phải phát hành lại cert**.

---

## 3. Yêu cầu hệ thống (prerequisites)

> Phần này trả lời: **thuê bao nhiêu máy, mỗi máy cấu hình ra sao, mạng/firewall thế nào, thuê ở đâu** — rồi mới tới phần chuẩn bị user/OS.

### 3.1. Cần thuê bao nhiêu máy chủ?

| Vai trò | Nhóm inventory | Số lượng | Bắt buộc? | Có thể gộp/thay thế? |
|---|---|---|---|---|
| **Control‑plane** (`master1`) | `controlplane` | **1** | ✅ Bắt buộc | Không. Đây là 1 control‑plane (chưa HA). |
| **Worker** (`worker1…N`) | `node` | **N ≥ 1** (khuyến nghị **≥ 2**) | ✅ ≥ 1 | Thêm/bớt tùy nhu cầu |
| **Bastion / máy chạy Ansible** | `local` | **1** | ⬜ Tùy chọn | **Dùng luôn máy cá nhân/WSL** thay vì thuê riêng |

**Ví dụ số lượng theo nhu cầu:**

| Kịch bản | Số VM phải thuê | Thành phần |
|---|---|---|
| Lab tối thiểu | **2 VM** | 1 control‑plane + 1 worker (chạy Ansible từ laptop/WSL) |
| **Lab khuyến nghị** (khớp `inventory.ini` mẫu) | **3 VM** | 1 control‑plane + 2 worker (Ansible từ WSL) |
| Có bastion riêng | **4 VM** | 1 control‑plane + 2 worker + 1 bastion |

> ⚠️ **Lưu ý HA:** cụm **1 control‑plane KHÔNG có HA cho control‑plane** — nếu master chết thì mất quyền quản lý cụm (workload đang chạy trên worker vẫn sống). Đây là chủ ý cho phase học/nền tảng; HA control‑plane để dành phase sau (mở rộng `--control-plane --certificate-key`).

### 3.2. Cấu hình từng máy (theo vai trò)

Tối thiểu **theo tài liệu kubeadm**: control‑plane cần **≥ 2 vCPU, ≥ 2 GB RAM**.

| Vai trò | vCPU | RAM | Disk (SSD) | Ghi chú |
|---|---|---|---|---|
| **Control‑plane** | **2** (tối thiểu) | **2 GB** tối thiểu, **4 GB** khuyến nghị | **20–40 GB** | Chạy etcd + apiserver; <2 vCPU phải dùng `--ignore-preflight-errors=NumCPU,Mem` ([§6.6](#66-role-first-master--kubeadm-init)) |
| **Worker** | 2 | 2 GB tối thiểu, **4 GB+** khuyến nghị | **40 GB+** | RAM/CPU tăng theo workload bạn định chạy |
| **Bastion** | 1 | 1 GB | 10 GB | Chỉ chạy `ansible`/`kubectl`/`git`. Thường dùng WSL nên khỏi thuê |

**Cỡ instance tương ứng theo nhà cung cấp (cho node ~2 vCPU/4 GB):**

| Provider | Cỡ máy cho node | Cỡ máy cho bastion |
|---|---|---|
| AWS EC2 | `t3.medium` | `t3.micro` |
| GCP | `e2-medium` | `e2-micro` |
| Azure | `Standard_B2s` | `Standard_B1s` |
| DigitalOcean / Vultr / Linode | Gói 2 vCPU / 4 GB | Gói 1 vCPU / 1 GB |
| Hetzner Cloud | `CX22`/`CPX21` | `CX11` |
| Lab local | VM Multipass/VirtualBox/Proxmox 2 vCPU/2–4 GB | — |

### 3.3. Hệ điều hành, ổ đĩa & swap

- **OS:** Ubuntu Server **22.04 LTS** hoặc **24.04 LTS** (khuyến nghị **24.04** — cgroup v2 mặc định, containerd 1.7.x sẵn). **Tất cả node dùng cùng một bản OS.**
- **Kiến trúc:** `x86_64` (amd64). `arm64` cũng được — playbook tự phát hiện kiến trúc khi thêm repo Docker.
- **Đĩa:** SSD, ≥ 20 GB (control‑plane) / ≥ 40 GB (worker) cho image + pod ephemeral.
- **Swap:** kubelet yêu cầu **TẮT swap**. Playbook tự `swapoff -a` và xóa swap khỏi `/etc/fstab`; bạn không cần làm thủ công. (Một số provider bật swap sẵn — không sao.)
- **Kernel module:** cần `overlay`, `br_netfilter` (playbook tự nạp).

### 3.4. Mạng, IP tĩnh & firewall / security group

- **Cùng một mạng riêng/VPC:** mọi node phải SSH/ping được nhau qua **private IP**.
- **IP TĨNH (quan trọng):** `inventory.ini` ghi IP cố định, và kubeadm gắn IP vào chứng chỉ/etcd. Hãy đặt **IP tĩnh** (hoặc DHCP reservation) cho từng node — nếu IP đổi sau reboot sẽ **hỏng cluster**.
- **Internet ra ngoài (egress):** cần để tải `apt` (`pkgs.k8s.io`, `download.docker.com`), Helm chart (`helm.cilium.io`), image (`registry.k8s.io`, `quay.io`/`ghcr.io` của Cilium), `dl.k8s.io`. Môi trường air‑gapped phải tự mirror.

**Cổng cần mở (firewall / security group):**

| Cổng | Giao thức | Chiều | Dùng cho |
|---|---|---|---|
| `22` | TCP | bastion → mọi node | SSH (Ansible) |
| `6443` | TCP | worker + bastion → control‑plane | kube‑apiserver |
| `2379-2380` | TCP | control‑plane (nội bộ) | etcd |
| `10250` | TCP | mọi node ↔ mọi node | Kubelet API |
| `10257` | TCP | control‑plane | kube‑controller‑manager |
| `10259` | TCP | control‑plane | kube‑scheduler |
| `30000-32767` | TCP | client → worker | NodePort Services |
| `8472` | UDP | mọi node ↔ mọi node | **Cilium** VXLAN overlay |
| `4240` | TCP | mọi node ↔ mọi node | **Cilium** health check |
| (ICMP echo) | ICMP | mọi node ↔ mọi node | **Cilium** kiểm tra kết nối |

> 💡 **Lab đơn giản:** nếu tất cả node nằm trong **cùng một subnet/VPC tin cậy**, cách nhanh nhất là **cho phép toàn bộ traffic nội bộ giữa các node**, và chỉ giới hạn chiều từ Internet vào (mở `22`, `6443`, dải NodePort khi cần).

### 3.5. Gợi ý nhà cung cấp & các bước thuê

**Nơi thuê:**
- **Cloud công cộng:** AWS EC2, GCP Compute Engine, Azure VM, DigitalOcean, Vultr, Linode (Akamai), Hetzner Cloud, Oracle Cloud (có free tier arm64).
- **On‑prem / lab:** Proxmox VE, VMware vSphere, hoặc máy cá nhân với **Multipass / VirtualBox / Vagrant**.
- Repo gốc có sẵn thư mục **`terraform/`** dựng hạ tầng trên **AWS** (đã loại khỏi branch này nhưng còn trên `main`/lịch sử) — tham khảo được nếu muốn IaC. Lưu ý nó kèm LoadBalancer + bastion cho **HA 3 master**, vượt nhu cầu 1‑CP.

**Các bước thuê (chung cho mọi provider):**
1. Tạo **N+1** instance (hoặc **N+2** nếu bastion riêng) Ubuntu 22.04/24.04, cỡ theo [§3.2](#32-cấu-hình-từng-máy-theo-vai-trò).
2. Đặt tất cả vào **cùng VPC/subnet riêng**; gán **IP tĩnh**.
3. Khi tạo, **nạp SSH public key của bạn** (hoặc dùng password rồi nạp key sau).
4. Cấu hình **Security Group/firewall** theo [§3.4](#34-mạng-ip-tĩnh--firewall--security-group).
5. Ghi lại **private IP** từng máy → điền vào `inventory.ini` ([§6.2](#62-ansibleinventoryini-tạo-mới)).
6. Tạo user `ansible-user` trên mỗi node ([§3.6](#36-tạo-user-ansible-user-trên-mỗi-node)) — hoặc nhúng sẵn qua cloud‑init lúc tạo máy.

### 3.6. Tạo user `ansible-user` trên mỗi node

Mỗi node phải có user **`ansible-user`** với **sudo không mật khẩu** và **SSH public key** của bạn.

```bash
# Chạy trên từng node:
sudo adduser --disabled-password --gecos "" ansible-user
echo 'ansible-user ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/ansible-user
sudo mkdir -p /home/ansible-user/.ssh && sudo chmod 700 /home/ansible-user/.ssh
# dán public key của bạn vào:
sudo tee /home/ansible-user/.ssh/authorized_keys < your_key.pub
sudo chown -R ansible-user:ansible-user /home/ansible-user/.ssh
sudo chmod 600 /home/ansible-user/.ssh/authorized_keys
```

**Hoặc tự động hóa khi tạo máy bằng cloud‑init/user‑data** (dán vào ô "user data" của provider):

```yaml
#cloud-config
users:
  - name: ansible-user
    sudo: "ALL=(ALL) NOPASSWD:ALL"
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAA...thay-bằng-public-key-của-bạn...
package_update: true
```

### 3.7. Máy điều khiển (Ansible control node)
- Phải là **Linux** (Windows → dùng **WSL2 Ubuntu**, vì Ansible không chạy native trên Windows).
- Cài: `git`, `ansible` (core ≥ 2.15), `ssh`.

  ```bash
  sudo apt update
  sudo apt install -y git ansible openssh-client
  ansible --version    # kiểm tra
  ```

> Các bước cài đặt phần mềm ở trên là **việc bạn tự chạy trên máy/VM của mình**. Tài liệu này chỉ ghi lệnh gợi ý.

---

## 4. Bước 1 — Chuẩn bị Git repo

### 4.1. Kiểm tra repo đã được add chưa

Repo này **đã nằm trên GitHub rồi** — kiểm tra:

```bash
git remote -v
```

Kết quả mong đợi:

```
origin  https://github.com/hieupham259/ansible_playbook_k8s-installation.git (fetch)
origin  https://github.com/hieupham259/ansible_playbook_k8s-installation.git (push)
```

→ Đã có `origin`, nhánh chính là `main`. **Bỏ qua phần 4.2**, đi tiếp [Bước 2](#5-bước-2--tạo-branch-1-controlplane-n-worker-và-lọc-file).

### 4.2. (Chỉ khi bắt đầu từ con số 0 — repo CHƯA có git)

Nếu `git remote -v` không ra gì, hoặc bạn dựng project mới:

```bash
# Trong thư mục dự án
git init
git branch -M main

# Cấu hình danh tính (nếu chưa)
git config user.name  "Your Name"
git config user.email "you@example.com"

# Tạo repo trống trên GitHub trước, rồi:
git remote add origin https://github.com/<user>/<repo>.git

git add .
git commit -m "Initial commit"
git push -u origin main
```

---

## 5. Bước 2 — Tạo branch `1-controlplane-n-worker` và lọc file

Ý tưởng: tạo branch mới **chỉ chứa các file cần thiết** cho cụm 1 control‑plane + N worker.

### 5.1. Tạo branch từ `main` mới nhất

```bash
git checkout main
git pull origin main
git checkout -b 1-controlplane-n-worker
```

### 5.2. Xóa toàn bộ file KHÔNG cần (HA, cloud, storage, website, terraform, jenkins…)

```bash
# --- Playbook HA / cloud / storage / nhãn / nâng cấp (không dùng cho 1-CP) ---
git rm -f \
  ansible/add_label_master.yaml ansible/add_labels-master.yaml ansible/add_labels.yaml \
  ansible/add_labels_woker.yaml ansible/add_labels_worker.yaml ansible/worker_labels.yaml \
  ansible/label_controlplane.yaml \
  ansible/all.yaml ansible/deploy.yaml ansible/pki-copy.yaml ansible/manifest-files.yaml \
  ansible/front-proxy-ca.crt.yaml ansible/copy_admin_cong.yaml \
  ansible/join_master.yaml ansible/join_masters.yaml \
  ansible/aws_cloud_controller.yaml ansible/ebs-csi-role.yaml ansible/etcdctl_install.yaml \
  ansible/nfs.yaml ansible/nfs-setup.yaml \
  ansible/upgrade_master.yaml ansible/upgrade_node.yaml \
  ansible/k8s_installer.yaml ansible/kubeadm_reset_simplified.yml ansible/set_hostnames.yml \
  ansible/kubectl_alias.yaml \
  ansible/inventory ansible/inventories/inventory.ini

# --- Role không dùng ---
git rm -rf \
  ansible/roles/aws_cloud_controller ansible/roles/ebs-csi-driver \
  ansible/roles/nfs ansible/roles/nfs-setup ansible/roles/kubectl_alias

# --- Toàn bộ thư mục/ file ngoài phạm vi cluster ---
git rm -rf course-website terraform jenkins
git rm -f nginx.txt ping.yaml pod-nginx.yaml
```

> ⚠️ **Cảnh báo bảo mật:** repo gốc có commit **private key** `terraform/testing-dev-1.pem` và state Terraform.
> Lệnh `git rm -rf terraform` ở trên loại chúng khỏi *branch này*, nhưng chúng **vẫn còn trong lịch sử git của `main`**.
> Nên **thu hồi/đổi key đó** và cân nhắc dọn lịch sử (`git filter-repo`) ở một việc riêng.

### 5.3. File sẽ GIỮ và sẽ SỬA/TẠO

Sau khi xóa, ở [Bước 3](#6-bước-3--tạosửa-các-file-cấu-hình-nội-dung-đầy-đủ) bạn sẽ tạo mới/ghi đè để có cây thư mục cuối cùng:

```
.
├── K8S-1CP-NWORKER-GUIDE.md           # tài liệu này (tùy chọn commit kèm)
└── ansible/
    ├── ansible.cfg                    # TẠO MỚI
    ├── inventory.ini                  # TẠO MỚI (thay cho inventories/inventory.ini đã xóa)
    ├── group_vars/
    │   └── all.yml                    # TẠO MỚI  (ghim version tập trung)
    ├── cluster.yaml                   # TẠO MỚI  (orchestrator)
    ├── reset.yaml                     # TẠO MỚI  (teardown để chạy lại)
    ├── k8s-setup.yaml                 # SỬA
    ├── time-sync.yaml                 # GIỮ/CHUẨN HÓA
    ├── kubernetes_role.yaml           # GIỮ/CHUẨN HÓA
    ├── helm-install.yaml              # SỬA (chỉ chạy trên controlplane)
    ├── first-master.yaml              # SỬA (hosts: controlplane)
    ├── join_workers.yaml              # SỬA (sạch, idempotent)
    ├── install_cni.yaml               # SỬA (Cilium 1.17 + pod CIDR)
    ├── kubectl_role.yaml              # GIỮ
    └── roles/
        ├── k8s-setup/      (defaults, handlers, tasks)   # SỬA
        ├── kubernetes_role/(defaults, handlers, tasks)   # SỬA
        ├── time-sync/      (defaults, tasks)             # SỬA (bỏ handler hỏng)
        ├── helm/           (defaults, tasks)             # GIỮ/CHUẨN HÓA
        ├── first-master/   (defaults, tasks, templates)  # SỬA
        └── kubectl_role/   (vars, tasks)                 # SỬA
```

Ngoài ra **xóa các file thừa trong role** (nếu còn):

```bash
git rm -f ansible/roles/kubectl_role/tasks/permissions.yml
git rm -rf ansible/roles/time-sync/handler   # thư mục đặt sai tên (handler -> phải là handlers)
```

---

## 6. Bước 3 — Tạo/sửa các file cấu hình (nội dung đầy đủ)

> Copy nguyên văn từng block dưới đây vào đúng đường dẫn. (Đường dẫn ghi ở tiêu đề mỗi block.)

### 6.1. `ansible/ansible.cfg`  *(tạo mới)*

```ini
[defaults]
inventory = inventory.ini
roles_path = roles
remote_user = ansible-user
host_key_checking = False
retry_files_enabled = False
deprecation_warnings = False
interpreter_python = auto_silent
stdout_callback = yaml
force_color = True

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
pipelining = True
```

*Giải thích:* trỏ sẵn inventory + role path, login bằng `ansible-user`, tự `sudo`, tắt hỏi host‑key (lab). `pipelining=True` tăng tốc.

### 6.2. `ansible/inventory.ini`  *(tạo mới)*

```ini
# 1 control-plane + N worker. Thay ansible_host bằng IP thật của từng node.

[controlplane]
master1 ansible_host=10.0.0.11

[node]
worker1 ansible_host=10.0.0.21
worker2 ansible_host=10.0.0.22
# worker3 ansible_host=10.0.0.23      # thêm worker tùy ý

# Máy đang chạy Ansible (bastion/WSL). kubectl + kubeconfig sẽ nằm ở đây.
[local]
bastion ansible_connection=local

# Mọi node K8s (control-plane + worker) — dùng để chuẩn bị node.
[k8s_node:children]
controlplane
node

[k8s_worker_node:children]
node

[all:vars]
ansible_user=ansible-user
ansible_ssh_private_key_file=~/.ssh/id_rsa
ansible_python_interpreter=/usr/bin/python3
```

> **Bắt buộc:** host control‑plane đặt tên **`master1`** (một số task tham chiếu host đầu của nhóm `controlplane`).
> Thêm bao nhiêu worker tùy ý dưới `[node]`.

### 6.3. `ansible/group_vars/all.yml`  *(tạo mới — NƠI GHIM VERSION DUY NHẤT)*

```yaml
---
# =============================================================================
#  SINGLE SOURCE OF TRUTH — ghim version cho toàn cụm
#  (1 control-plane + N worker | kubeadm + containerd + Cilium)
# =============================================================================

# User sudo mà Ansible login trên MỌI node (phải tồn tại sẵn, sudo NOPASSWD + SSH key).
k8s_user: "ansible-user"

# --- Kubernetes (kubeadm / kubelet / kubectl) --------------------------------
kubernetes_minor: "v1.34"                 # kênh repo pkgs.k8s.io
kubernetes_patch: "1.34.6"                # patch chính xác — kiểm tra: apt-cache madison kubeadm
kubernetes_apt_version: "{{ kubernetes_patch }}-1.1"
kubeadm_kubernetes_version: "v{{ kubernetes_patch }}"

# --- containerd --------------------------------------------------------------
# "" = cài bản 1.7.x mới nhất từ repo Docker. Ghim chính xác thì đặt vd "1.7.27-1"
# (kiểm tra: apt-cache madison containerd.io)
containerd_apt_version: ""

# --- CNI: Cilium (cài qua Helm) ----------------------------------------------
cilium_version: "1.17.16"                 # hỗ trợ K8s 1.31–1.34

# --- Mạng cụm ----------------------------------------------------------------
pod_subnet: "10.244.0.0/16"
service_subnet: "10.96.0.0/12"
```

*Giải thích:* đổi version cả cụm chỉ tại đây. `kubernetes_apt_version` theo định dạng gói của `pkgs.k8s.io` là `<patch>-1.1`.

### 6.4. Role `k8s-setup` — containerd + kernel/sysctl

#### `ansible/roles/k8s-setup/defaults/main.yaml`

```yaml
---
# Version containerd được ghim tập trung ở group_vars/all.yml.
containerd_apt_version: ""
# Ảnh "pause"/sandbox mà kubeadm 1.34 mong đợi.
pause_image: "registry.k8s.io/pause:3.10"
```

#### `ansible/roles/k8s-setup/tasks/main.yaml`  *(ghi đè toàn bộ)*

```yaml
---
# k8s-setup — containerd runtime + kernel/sysctl cho kubeadm
# Ref: https://kubernetes.io/docs/setup/production-environment/container-runtimes/

# ----- Kernel modules -----
- name: Persist required kernel modules
  ansible.builtin.copy:
    dest: /etc/modules-load.d/k8s.conf
    mode: "0644"
    content: |
      overlay
      br_netfilter

- name: Load kernel modules now
  ansible.builtin.shell: |
    modprobe overlay
    modprobe br_netfilter
  changed_when: false

# ----- sysctl -----
- name: Set required sysctl params
  ansible.builtin.copy:
    dest: /etc/sysctl.d/99-kubernetes-cri.conf
    mode: "0644"
    content: |
      net.bridge.bridge-nf-call-iptables  = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      net.ipv4.ip_forward                 = 1

- name: Apply sysctl params
  ansible.builtin.command: sysctl --system
  changed_when: false

# ----- containerd từ repo apt chính thức của Docker (ghim được, ổn định 1.7.x) -----
- name: Install apt prerequisites
  ansible.builtin.apt:
    name: [ca-certificates, curl, gnupg]
    state: present
    update_cache: true

- name: Ensure /etc/apt/keyrings exists
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    mode: "0755"

- name: Add Docker GPG key (cung cấp gói containerd.io)
  ansible.builtin.shell: |
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  args:
    creates: /etc/apt/keyrings/docker.gpg

- name: Add Docker apt repository
  ansible.builtin.copy:
    dest: /etc/apt/sources.list.d/docker.list
    mode: "0644"
    content: "deb [arch={{ 'arm64' if ansible_architecture == 'aarch64' else 'amd64' }} signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable\n"

- name: Install containerd.io
  ansible.builtin.apt:
    name: "containerd.io{{ ('=' + containerd_apt_version) if containerd_apt_version else '' }}"
    state: present
    update_cache: true

- name: Hold containerd.io tại version đã cài
  ansible.builtin.dpkg_selections:
    name: containerd.io
    selection: hold

# ----- Cấu hình containerd: systemd cgroup + đúng pause image -----
- name: Ensure /etc/containerd exists
  ansible.builtin.file:
    path: /etc/containerd
    state: directory
    mode: "0755"

- name: Generate default containerd config
  ansible.builtin.shell: containerd config default > /etc/containerd/config.toml
  args:
    creates: /etc/containerd/config.toml

- name: Use the systemd cgroup driver (đúng cho cả containerd 1.x và 2.x)
  ansible.builtin.replace:
    path: /etc/containerd/config.toml
    regexp: 'SystemdCgroup = false'
    replace: 'SystemdCgroup = true'
  notify: restart containerd

- name: Pin the sandbox (pause) image
  ansible.builtin.replace:
    path: /etc/containerd/config.toml
    regexp: 'sandbox_image = ".*"'
    replace: 'sandbox_image = "{{ pause_image }}"'
  notify: restart containerd

- name: Enable and start containerd
  ansible.builtin.systemd:
    name: containerd
    state: started
    enabled: true
```

*Giải thích:* tải module `overlay`/`br_netfilter`, đặt sysctl bắt buộc; cài `containerd.io` (ghim được), bật `SystemdCgroup=true` (khớp kubelet), đặt đúng ảnh pause `3.10`.

#### `ansible/roles/k8s-setup/handlers/main.yaml`

```yaml
---
- name: restart containerd
  ansible.builtin.systemd:
    name: containerd
    state: restarted
```

### 6.5. Role `kubernetes_role` — kubelet/kubeadm/kubectl (ghim)

#### `ansible/roles/kubernetes_role/defaults/main.yaml`

```yaml
---
# Version đến từ group_vars/all.yml (kubernetes_minor, kubernetes_apt_version)
kubernetes_repo_url: "https://pkgs.k8s.io/core:/stable:/{{ kubernetes_minor | default('v1.34') }}/deb/"
kubernetes_package:
  - "kubelet={{ kubernetes_apt_version | default('1.34.6-1.1') }}"
  - "kubeadm={{ kubernetes_apt_version | default('1.34.6-1.1') }}"
  - "kubectl={{ kubernetes_apt_version | default('1.34.6-1.1') }}"
```

#### `ansible/roles/kubernetes_role/tasks/main.yaml`  *(ghi đè toàn bộ)*

```yaml
---
- name: Set hostname theo tên trong inventory
  ansible.builtin.hostname:
    name: "{{ inventory_hostname }}"

- name: Disable swap ngay
  ansible.builtin.command: swapoff -a
  changed_when: false

- name: Disable swap vĩnh viễn trong /etc/fstab
  ansible.builtin.replace:
    path: /etc/fstab
    regexp: '^([^#].*\s+swap\s+.*)$'
    replace: '# \1'

- name: Install apt prerequisites
  ansible.builtin.apt:
    name: [apt-transport-https, ca-certificates, curl, gpg]
    state: present
    update_cache: true

- name: Ensure /etc/apt/keyrings exists
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    mode: "0755"

- name: Add Kubernetes apt signing key
  ansible.builtin.shell: |
    curl -fsSL {{ kubernetes_repo_url }}Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  args:
    creates: /etc/apt/keyrings/kubernetes-apt-keyring.gpg

- name: Add Kubernetes apt repository
  ansible.builtin.copy:
    dest: /etc/apt/sources.list.d/kubernetes.list
    mode: "0644"
    content: "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] {{ kubernetes_repo_url }} /\n"

- name: Install kubelet, kubeadm, kubectl (ghim version)
  ansible.builtin.apt:
    name: "{{ kubernetes_package }}"
    state: present
    update_cache: true

- name: Hold kubelet, kubeadm, kubectl
  ansible.builtin.dpkg_selections:
    name: "{{ item }}"
    selection: hold
  loop: [kubelet, kubeadm, kubectl]

- name: Enable kubelet (sẽ chờ tới khi kubeadm chạy — đúng như mong đợi)
  ansible.builtin.systemd:
    name: kubelet
    enabled: true
```

#### `ansible/roles/kubernetes_role/handlers/main.yaml`

```yaml
---
- name: Enable and start kubelet
  ansible.builtin.systemd:
    name: kubelet
    enabled: true
    state: started
```

### 6.6. Role `first-master` — kubeadm init

#### `ansible/roles/first-master/defaults/main.yaml`

```yaml
---
kubernetes_dir: "/etc/kubernetes"
admin_conf: "{{ kubernetes_dir }}/admin.conf"

# IP apiserver advertise/bind. Mặc định = IP chính của node.
advertise_address: "{{ ansible_default_ipv4.address }}"

# Endpoint ổn định cho client. Cụm 1 master = IP master. (Trỏ LB VIP/DNS nếu mở rộng HA.)
control_plane_endpoint: "{{ advertise_address }}"

# SAN bổ sung cho cert apiserver (vd DNS của LB sau này).
apiserver_extra_sans: []

# Đặt "NumCPU,Mem" nếu test trên VM nhỏ (<2 vCPU / <2GB).
kubeadm_ignore_preflight_errors: ""

pause_image: "registry.k8s.io/pause:3.10"
kube_user_home: "/home/{{ k8s_user }}"
```

#### `ansible/roles/first-master/templates/config.yaml.j2`  *(ghi đè — kubeadm v1beta4)*

```jinja
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "{{ advertise_address }}"
  bindPort: 6443
nodeRegistration:
  criSocket: "unix:///var/run/containerd/containerd.sock"
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: "{{ kubeadm_kubernetes_version }}"
controlPlaneEndpoint: "{{ control_plane_endpoint }}:6443"
networking:
  podSubnet: "{{ pod_subnet | default('10.244.0.0/16') }}"
  serviceSubnet: "{{ service_subnet | default('10.96.0.0/12') }}"
apiServer:
  certSANs:
    - "{{ advertise_address }}"
    - "{{ control_plane_endpoint }}"
    - "127.0.0.1"
{% for san in apiserver_extra_sans %}
    - "{{ san }}"
{% endfor %}
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

#### `ansible/roles/first-master/tasks/main.yaml`  *(ghi đè toàn bộ)*

```yaml
---
- name: Kiểm tra cụm đã init chưa
  ansible.builtin.stat:
    path: "{{ admin_conf }}"
  register: admin_conf_stat

- name: Pre-pull ảnh pause vào containerd (namespace k8s.io)
  ansible.builtin.command: "ctr --namespace k8s.io image pull {{ pause_image }}"
  changed_when: false
  when: not admin_conf_stat.stat.exists

- name: Ensure /etc/kubernetes exists
  ansible.builtin.file:
    path: "{{ kubernetes_dir }}"
    state: directory
    mode: "0755"

- name: Render kubeadm config
  ansible.builtin.template:
    src: config.yaml.j2
    dest: "{{ kubernetes_dir }}/kubeadm-config.yaml"
    mode: "0644"

- name: kubeadm init (idempotent — bỏ qua nếu admin.conf đã tồn tại)
  ansible.builtin.command: >
    kubeadm init --config {{ kubernetes_dir }}/kubeadm-config.yaml
    {% if kubeadm_ignore_preflight_errors %}--ignore-preflight-errors={{ kubeadm_ignore_preflight_errors }}{% endif %}
  args:
    creates: "{{ admin_conf }}"
  register: kubeadm_init

- name: In tóm tắt kubeadm init
  ansible.builtin.debug:
    var: kubeadm_init.stdout_lines
  when: kubeadm_init is changed

# ----- kubeconfig cho user sudo và root trên control-plane -----
- name: Tạo ~/.kube cho {{ k8s_user }}
  ansible.builtin.file:
    path: "{{ kube_user_home }}/.kube"
    state: directory
    owner: "{{ k8s_user }}"
    group: "{{ k8s_user }}"
    mode: "0755"

- name: Cài kubeconfig cho {{ k8s_user }}
  ansible.builtin.copy:
    src: "{{ admin_conf }}"
    dest: "{{ kube_user_home }}/.kube/config"
    remote_src: true
    owner: "{{ k8s_user }}"
    group: "{{ k8s_user }}"
    mode: "0600"

- name: Tạo ~/.kube cho root
  ansible.builtin.file:
    path: /root/.kube
    state: directory
    mode: "0755"

- name: Cài kubeconfig cho root
  ansible.builtin.copy:
    src: "{{ admin_conf }}"
    dest: /root/.kube/config
    remote_src: true
    mode: "0600"
```

*Giải thích:* `creates: admin.conf` làm `kubeadm init` **idempotent** (chạy lại không lỗi). Endpoint/SAN lấy từ biến → linh hoạt. KubeletConfiguration ép `cgroupDriver: systemd` khớp containerd.

### 6.7. Role `time-sync` — chrony

#### `ansible/roles/time-sync/defaults/main.yaml`

```yaml
---
chrony_package: chrony
chrony_service: chronyd
```

#### `ansible/roles/time-sync/tasks/main.yaml`  *(ghi đè — bỏ handler đặt sai thư mục)*

```yaml
---
- name: Install chrony
  ansible.builtin.package:
    name: "{{ chrony_package }}"
    state: present

- name: Enable and start chrony
  ansible.builtin.service:
    name: "{{ chrony_service }}"
    state: started
    enabled: true

- name: Force an immediate time step
  ansible.builtin.command: chronyc makestep
  changed_when: false
  ignore_errors: true
```

> Nhớ xóa thư mục cũ đặt sai tên: `git rm -rf ansible/roles/time-sync/handler` (đã nêu ở §5.3).
> *Tại sao cần đồng bộ giờ:* etcd/kubelet rất nhạy với lệch đồng hồ → chrony bắt buộc cho cụm khỏe.

### 6.8. Role `helm`

#### `ansible/roles/helm/defaults/main.yaml`

```yaml
---
helm_version: "v3.16.2"
helm_platform: "linux-amd64"
helm_download_url: "https://get.helm.sh/helm-{{ helm_version }}-{{ helm_platform }}.tar.gz"
helm_bin_path: "/usr/local/bin/helm"
```

#### `ansible/roles/helm/tasks/main.yaml`  *(ghi đè — gọn hơn)*

```yaml
---
- name: Download Helm {{ helm_version }}
  ansible.builtin.get_url:
    url: "{{ helm_download_url }}"
    dest: "/tmp/helm-{{ helm_version }}.tar.gz"
    mode: "0644"

- name: Extract Helm
  ansible.builtin.unarchive:
    src: "/tmp/helm-{{ helm_version }}.tar.gz"
    dest: /tmp
    remote_src: true

- name: Install Helm binary
  ansible.builtin.copy:
    src: "/tmp/{{ helm_platform }}/helm"
    dest: "{{ helm_bin_path }}"
    remote_src: true
    mode: "0755"
```

### 6.9. Role `kubectl_role` — cài kubectl + lấy kubeconfig về bastion

#### `ansible/roles/kubectl_role/vars/main.yaml`  *(ghi đè)*

```yaml
---
kubectl_version: "{{ kubeadm_kubernetes_version | default('v1.34.6') }}"
kubectl_url: "https://dl.k8s.io/release/{{ kubectl_version }}/bin/linux/amd64/kubectl"
kubectl_dest: "/usr/local/bin/kubectl"
kube_config_dir: "{{ lookup('env', 'HOME') }}/.kube"
kube_config_file: "{{ kube_config_dir }}/config"
```

#### `ansible/roles/kubectl_role/tasks/main.yaml`  *(ghi đè)*

```yaml
---
- import_tasks: install_kubectl.yml
- import_tasks: fetch_admin_config.yml
```

#### `ansible/roles/kubectl_role/tasks/install_kubectl.yml`  *(ghi đè)*

```yaml
---
- name: Download kubectl {{ kubectl_version }}
  ansible.builtin.get_url:
    url: "{{ kubectl_url }}"
    dest: "{{ kubectl_dest }}"
    mode: "0755"
```

#### `ansible/roles/kubectl_role/tasks/fetch_admin_config.yml`  *(ghi đè)*

```yaml
---
- name: Ensure local ~/.kube exists
  ansible.builtin.file:
    path: "{{ kube_config_dir }}"
    state: directory
    mode: "0755"
  become: false

- name: Copy kubeconfig từ control-plane về ~/.kube/config
  ansible.builtin.fetch:
    src: /etc/kubernetes/admin.conf
    dest: "{{ kube_config_file }}"
    flat: true
  delegate_to: "{{ groups['controlplane'][0] }}"
  become: true
```

> Xóa file cũ: `git rm -f ansible/roles/kubectl_role/tasks/permissions.yml` (đã nêu ở §5.3).
> `kubectl_url` lấy từ `dl.k8s.io` đúng version cụm (repo gốc lấy bản EKS 1.30 lệch 2 minor).

### 6.10. Các playbook

#### `ansible/k8s-setup.yaml`

```yaml
---
- name: Chuẩn bị node — containerd runtime + kernel prerequisites
  hosts: k8s_node
  become: true
  roles:
    - k8s-setup
```

#### `ansible/time-sync.yaml`

```yaml
---
- name: Đồng bộ thời gian trên mọi node
  hosts: k8s_node
  become: true
  roles:
    - time-sync
```

#### `ansible/kubernetes_role.yaml`

```yaml
---
- name: Cài gói Kubernetes (kubelet/kubeadm/kubectl)
  hosts: k8s_node
  become: true
  roles:
    - kubernetes_role
```

#### `ansible/helm-install.yaml`  *(sửa: chỉ controlplane)*

```yaml
---
- name: Cài Helm trên control-plane
  hosts: controlplane
  become: true
  roles:
    - helm
```

#### `ansible/first-master.yaml`  *(sửa: hosts controlplane)*

```yaml
---
- name: Khởi tạo control-plane duy nhất
  hosts: controlplane
  become: true
  roles:
    - first-master
```

#### `ansible/join_workers.yaml`  *(ghi đè — sạch, idempotent)*

```yaml
---
- name: Sinh join token trên control-plane
  hosts: controlplane
  become: true
  gather_facts: false
  tasks:
    - name: Tạo join token mới (TTL 24h)
      ansible.builtin.command: kubeadm token create --print-join-command --ttl 24h
      register: join_cmd
      changed_when: false

    - name: Lưu join command thành fact
      ansible.builtin.set_fact:
        worker_join_command: "{{ join_cmd.stdout }}"

- name: Join worker vào cụm
  hosts: node
  become: true
  gather_facts: false
  tasks:
    - name: kubeadm join (idempotent — bỏ qua nếu kubelet.conf đã có)
      ansible.builtin.shell: >
        {{ hostvars[groups['controlplane'][0]].worker_join_command }}
        --cri-socket=unix:///var/run/containerd/containerd.sock
      args:
        creates: /etc/kubernetes/kubelet.conf

    - name: Bật và chạy kubelet
      ansible.builtin.systemd:
        name: kubelet
        state: started
        enabled: true
```

#### `ansible/install_cni.yaml`  *(ghi đè — Cilium 1.17 + pod CIDR khớp)*

```yaml
---
- name: Cài Cilium CNI qua Helm
  hosts: controlplane
  become: true
  gather_facts: false
  environment:
    KUBECONFIG: /home/{{ k8s_user }}/.kube/config
  tasks:
    - name: Thêm Helm repo Cilium
      ansible.builtin.command: helm repo add cilium https://helm.cilium.io/
      register: repo_add
      changed_when: "'has been added' in repo_add.stdout"
      failed_when: repo_add.rc != 0 and 'already exists' not in (repo_add.stderr | default(''))

    - name: Update Helm repos
      ansible.builtin.command: helm repo update
      changed_when: false

    - name: Install/upgrade Cilium {{ cilium_version }}
      ansible.builtin.command: >
        helm upgrade --install cilium cilium/cilium
        --version {{ cilium_version }}
        --namespace kube-system
        --set ipam.mode=cluster-pool
        --set ipam.operator.clusterPoolIPv4PodCIDRList={{ pod_subnet }}
        --set ipam.operator.clusterPoolIPv4MaskSize=24
      register: cilium
      changed_when: "'STATUS: deployed' in cilium.stdout"

    - name: Chờ mọi node Ready
      ansible.builtin.command: kubectl wait --for=condition=Ready nodes --all --timeout=180s
      changed_when: false
```

*Giải thích:* `helm upgrade --install` → idempotent. Đặt `clusterPoolIPv4PodCIDRList` = `pod_subnet` để Cilium dùng đúng dải pod CIDR đã khai trong kubeadm.

#### `ansible/kubectl_role.yaml`

```yaml
---
- name: Cài kubectl + lấy kubeconfig về máy điều khiển
  hosts: local
  become: true
  roles:
    - kubectl_role
```

#### `ansible/cluster.yaml`  *(tạo mới — chạy một phát ra cả cụm)*

```yaml
---
# Dựng toàn bộ cụm theo đúng thứ tự.
#   ansible-playbook cluster.yaml
- import_playbook: k8s-setup.yaml
- import_playbook: time-sync.yaml
- import_playbook: kubernetes_role.yaml
- import_playbook: helm-install.yaml
- import_playbook: first-master.yaml
- import_playbook: join_workers.yaml
- import_playbook: install_cni.yaml
- import_playbook: kubectl_role.yaml
```

#### `ansible/reset.yaml`  *(tạo mới — teardown để chạy lại khi lỗi)*

```yaml
---
- name: kubeadm reset trên mọi node (dùng khi muốn dựng lại từ đầu)
  hosts: k8s_node
  become: true
  gather_facts: false
  tasks:
    - name: kubeadm reset -f
      ansible.builtin.command: kubeadm reset -f
      changed_when: false
      ignore_errors: true

    - name: Xóa state còn sót
      ansible.builtin.file:
        path: "{{ item }}"
        state: absent
      loop:
        - /etc/cni/net.d
        - /etc/kubernetes
        - /var/lib/etcd
        - "/home/{{ k8s_user }}/.kube"
        - /root/.kube

    - name: Flush iptables
      ansible.builtin.shell: iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
      changed_when: false
      ignore_errors: true
```

---

## 7. Bước 4 — Chạy playbook để dựng cluster

```bash
cd ansible

# 0) Sửa inventory.ini cho đúng IP các node của bạn (đã làm ở §6.2).

# 1) Kiểm tra kết nối SSH tới mọi node
ansible all -m ping

# 2A) Dựng toàn bộ một phát:
ansible-playbook cluster.yaml

# 2B) …hoặc chạy từng bước (tiện debug):
ansible-playbook k8s-setup.yaml
ansible-playbook time-sync.yaml
ansible-playbook kubernetes_role.yaml
ansible-playbook helm-install.yaml
ansible-playbook first-master.yaml
ansible-playbook join_workers.yaml
ansible-playbook install_cni.yaml
ansible-playbook kubectl_role.yaml
```

> **VM nhỏ (<2 vCPU/2GB):** trước khi `first-master.yaml`, set biến cho qua preflight:
> `ansible-playbook first-master.yaml -e kubeadm_ignore_preflight_errors=NumCPU,Mem`

---

## 8. Bước 5 — Kiểm tra cluster

Từ **bastion** (kubeconfig đã được copy về `~/.kube/config` bởi `kubectl_role.yaml`):

```bash
kubectl get nodes -o wide
# Tất cả node phải ở trạng thái Ready (sau khi Cilium chạy).

kubectl get pods -A
# coredns, cilium, cilium-operator… phải Running.

kubectl -n kube-system get pods -l k8s-app=cilium
kubectl version            # client/server đều v1.34.x

# Smoke test: deploy thử nginx
kubectl create deployment nginx --image=nginx --replicas=2
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pods -o wide   # pod chạy rải trên các worker
kubectl get svc nginx
kubectl delete svc/nginx deployment/nginx   # dọn
```

Kiểm tra cgroup driver (phải `systemd` ở cả hai phía):

```bash
# trên 1 node:
sudo crictl info | grep -i systemdCgroup        # true
systemctl show -p ExecStart kubelet | head -1   # kubelet đang chạy
```

---

## 9. Bước 6 — Commit & push branch

```bash
# Ở thư mục gốc repo (đang ở branch 1-controlplane-n-worker)
git add -A
git status                # xem lại file thêm/xóa

git commit -m "Add clean single control-plane + N worker kubeadm setup

- Pin Kubernetes v1.34.6 (Rancher 2.14 imported range 1.33-1.35)
- containerd 1.7.x via Docker repo, SystemdCgroup=true
- Cilium 1.17.16 CNI via Helm
- kubeadm v1beta4 config; idempotent init/join
- Remove HA/cloud/storage/website/terraform/jenkins files"

# Đẩy branch lên GitHub
git push -u origin 1-controlplane-n-worker
```

> Sau đó có thể mở Pull Request trên GitHub nếu muốn review trước khi merge vào `main`.

---

## 10. Xử lý sự cố & reset

| Triệu chứng | Nguyên nhân thường gặp | Cách xử lý |
|---|---|---|
| `kubeadm init` lỗi preflight CPU/Mem | VM quá nhỏ | `-e kubeadm_ignore_preflight_errors=NumCPU,Mem` |
| Node mãi `NotReady` | CNI chưa cài/chưa xong | Chạy lại `install_cni.yaml`; `kubectl -n kube-system get pods` |
| `kubeadm join` lỗi token hết hạn | token 24h hết hạn | Chạy lại `join_workers.yaml` (sinh token mới) |
| Pod kẹt `ContainerCreating`, lỗi pause image | sai sandbox image | Đã set `pause:3.10` trong containerd; `systemctl restart containerd` |
| cgroup mismatch, kubelet crash | containerd dùng cgroupfs | Đảm bảo `SystemdCgroup=true`; `systemctl restart containerd kubelet` |
| Muốn dựng lại từ đầu | — | `ansible-playbook reset.yaml` rồi chạy lại `cluster.yaml` |

**Reset toàn cụm:**

```bash
cd ansible
ansible-playbook reset.yaml
ansible-playbook cluster.yaml
```

---

## 11. Chuẩn bị cho Rancher (phase tiếp theo)

Cụm này là **vanilla kubeadm** → trong Rancher nó là **"imported/registered cluster"**.

- **Tương thích:** Rancher mới nhất **2.14.x** chứng nhận imported Kubernetes **1.33–1.35** → cụm **1.34.6** của ta nằm gọn trong dải này. (Nếu bạn dùng Rancher LTS **2.11.x** thì nó chỉ tới 1.32 — đã EOL — nên **không** chọn LTS 2.11 cho cụm mới; hãy dùng 2.13/2.14.)
- **containerd 1.7 + Cilium 1.17** đều được Rancher chấp nhận cho cụm import.
- **Các bước import (làm ở phase Rancher, không nằm trong tài liệu này):**
  1. Cài Rancher Manager 2.14.x (thường trên một cụm/VM riêng, qua Helm).
  2. Trong Rancher UI: **Cluster Management → Import Existing → Generic**.
  3. Rancher đưa một lệnh `kubectl apply -f <url>` → chạy lệnh đó **trên cụm vừa dựng** (dùng `~/.kube/config` ở bastion).
  4. `cattle-cluster-agent` được cài, cụm hiện lên trong Rancher.
- **Giữ kubeconfig** ở `~/.kube/config` (bastion) để thao tác import.

> Vì stack đã ghim đúng dải Rancher hỗ trợ, phase Rancher sẽ "cắm là chạy" mà không phải nâng/hạ version Kubernetes.

---

## 12. Tài liệu tham khảo chính thức

- Kubernetes — *Container runtimes* (containerd, systemd cgroup, CRI v1): <https://kubernetes.io/docs/setup/production-environment/container-runtimes/>
- Kubernetes — *Installing kubeadm* (repo `pkgs.k8s.io`, swap, hold): <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/>
- Kubernetes — *Creating a cluster with kubeadm* (config v1beta4, `--pod-network-cidr`, certSANs): <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/>
- Kubernetes — *kubeadm v1beta4* (blog & config API): <https://kubernetes.io/blog/2024/08/23/kubernetes-1-31-kubeadm-v1beta4/> · <https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/>
- Kubernetes — *Releases / vòng đời* (1.32 EOL, 1.34 EOL 2026‑10‑27): <https://kubernetes.io/releases/> · <https://endoflife.date/kubernetes>
- containerd — *Releases & vòng đời* (1.7.x extended support): <https://github.com/containerd/containerd/releases> · <https://containerd.io/releases/>
- Cilium — *Kubernetes requirements* (ma trận version): <https://docs.cilium.io/en/stable/network/kubernetes/requirements/>
- Rancher — *Support Matrix* (imported K8s versions theo từng bản Rancher): <https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/>
- Ansible — *Best practices / inventory*: <https://docs.ansible.com/ansible/latest/inventory_guide/index.html>

---

*Tài liệu sinh ngày 2026‑06‑14. Các mốc EOL/patch đúng tại thời điểm này — hãy kiểm tra lại `apt-cache madison kubeadm` và ma trận Rancher trước khi triển khai production.*
