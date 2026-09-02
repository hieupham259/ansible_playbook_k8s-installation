# Runbook: Cụm HA phiên bản hỗn hợp — dựng từ đầu, nâng cấp dở dang, và chứng minh Mixed Version Proxy

> **Runbook này dựng một tình huống mà không lab nào trong repo dựng được:** một cluster HA đang
> ở **giữa chừng một đợt nâng cấp**, có API server chạy hai phiên bản Kubernetes khác nhau cùng
> lúc, và một request rơi vào API server cũ vẫn được phục vụ nhờ **Mixed Version Proxy**.
>
> **Vì sao phải có runbook riêng.** [Lab 1c phần B5](k8s-docs/labs/LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md)
> chỉ chứng minh cluster một control plane **không có** điều kiện để tính năng này xuất hiện.
> [Lab 8b](k8s-docs/labs/LAB-8B-HA-VOI-STACKED-ETCD.md) và [8c](k8s-docs/labs/LAB-8C-HA-VOI-EXTERNAL-ETCD.md)
> dựng HA nhưng **không nâng cấp**. Nâng cấp thuộc [giai đoạn 17](k8s-docs/00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster)
> và checkpoint ở đó nâng cluster **một** control plane, nên không bao giờ sinh ra skew giữa các
> API server. Ba chỗ đó cộng lại vẫn để hở đúng tình huống này.
>
> **Đây là runbook, không phải lab.** Không có checkpoint vấn đáp, không nằm trong chuỗi snapshot
> của lộ trình, không tính vào tiến độ 27 lab. Nó chạy copy-paste và có gate `PASS:` sau mỗi bước
> có thể sai.
>
> **Bộ VM riêng, IP riêng, không đụng vào bất kỳ cluster nào đang có.** Chi tiết ở [§2.2](#22-bảng-vm-và-ip--chống-trùng).
>
> **Ngày đối chiếu phiên bản: 02/09/2026.**

---

## Mục lục

1. [Tổng quan & kiến trúc](#1-tổng-quan--kiến-trúc)
2. [Quy hoạch](#2-quy-hoạch)
3. [Dựng năm VM — lấy lại gì, thêm gì, bớt gì](#3-dựng-năm-vm--lấy-lại-gì-thêm-gì-bớt-gì)
4. [Cài runtime và kubeadm trên bốn node Kubernetes](#4-cài-runtime-và-kubeadm-trên-bốn-node-kubernetes)
5. [Load balancer](#5-load-balancer)
6. [Khởi tạo control plane với Mixed Version Proxy bật sẵn](#6-khởi-tạo-control-plane-với-mixed-version-proxy-bật-sẵn)
7. [CNI và join ba node còn lại](#7-cni-và-join-ba-node-còn-lại)
8. [Ba kubeconfig trỏ thẳng vào từng API server](#8-ba-kubeconfig-trỏ-thẳng-vào-từng-api-server)
9. [Ảnh chụp "trước" — cluster đồng phiên bản](#9-ảnh-chụp-trước--cluster-đồng-phiên-bản)
10. [Nâng cấp một control plane rồi dừng lại](#10-nâng-cấp-một-control-plane-rồi-dừng-lại)
11. [Tìm API bất đối xứng](#11-tìm-api-bất-đối-xứng)
12. [Chứng minh Mixed Version Proxy](#12-chứng-minh-mixed-version-proxy)
13. [Fault injection: 404 và 503](#13-fault-injection-404-và-503)
14. [Hoàn tất nâng cấp](#14-hoàn-tất-nâng-cấp)
15. [Dọn dẹp và đường quay lại](#15-dọn-dẹp-và-đường-quay-lại)
16. [Troubleshooting](#16-troubleshooting)
17. [Nguồn official](#17-nguồn-official)

---

## 1. Tổng quan & kiến trúc

### 1.1. Tình huống được dựng

```
GIAI ĐOẠN 1 — sau §7: cluster HA đồng phiên bản, ba API server v1.35.7

   client ──► HAProxy ──┬──► mvp-cp1  kube-apiserver v1.35.7
                        ├──► mvp-cp2  kube-apiserver v1.35.7
                        └──► mvp-cp3  kube-apiserver v1.35.7

GIAI ĐOẠN 2 — sau §10: NÂNG CẤP DỞ DANG, đây là thứ runbook này tồn tại để dựng

   client ──► HAProxy ──┬──► mvp-cp1  kube-apiserver v1.36.4   ◄── biết API mới
                        ├──► mvp-cp2  kube-apiserver v1.35.7   ◄── KHÔNG biết API mới
                        └──► mvp-cp3  kube-apiserver v1.35.7   ◄── KHÔNG biết API mới

   Request API mới rơi vào cp2:
       KHÔNG có Mixed Version Proxy  →  404 Not Found
       CÓ  Mixed Version Proxy       →  cp2 proxy sang cp1  →  200 OK

GIAI ĐOẠN 3 — sau §14: thoát khỏi trạng thái hỗn hợp, cả ba về v1.36.4
```

### 1.2. Cơ chế, đúng theo bài 37

Đọc [bài 37 — Proxy phiên bản hỗn hợp](k8s-docs/37-mixed-version-proxy-vi.md) trước khi chạy
runbook này. Tóm tắt phần runbook sẽ kiểm chứng:

1. API server nhận request tra **discovery non peer-aggregated của chính nó**. Có resource thì
   phục vụ tại chỗ.
2. Không có, nó tra discovery non peer-aggregated **của tất cả peer**, tìm ai phục vụ được.
3. Tìm được peer → **proxy** request sang đó.
4. Không peer nào biết → đưa vào chuỗi handler của chính nó → kết thúc bằng **404**.
5. Chọn được peer nhưng peer không phản hồi → **503**.

Runbook chứng minh trực tiếp cả bước 3, 4 và 5.

### 1.3. Cái runbook này **không** làm

| Không làm | Vì sao |
| --- | --- |
| Không đụng vào ba VM của [chuỗi lab chính](k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm) hay sáu VM của [Lab 8b](k8s-docs/labs/LAB-8B-HA-VOI-STACKED-ETCD.md#21-bộ-vm-riêng-của-nhóm-lab-ha) | Nâng cấp phá baseline `A1.3`; Lab 8a nói thẳng điều đó |
| Không dùng lại cluster của [runbook-k8s-vmware.md](runbook-k8s-vmware.md) | Cluster đó một control plane, không thể có hai API server khác phiên bản |
| Không dựng cặp LB + VIP | Một LB là SPOF, đúng như Lab 8b đã thừa nhận; ở đây SPOF không ảnh hưởng kết luận |
| Không bật encryption at rest, không audit | Ngoài phạm vi; xem [nợ #6](k8s-docs/labs/README.md#5-sổ-nợ-lab) |
| Không chạy `etcdctl snapshot` | Ngoài phạm vi; đường lui là snapshot VM |

---

## 2. Quy hoạch

### 2.1. Vì sao năm VM và vì sao không ít hơn

| Thành phần | Số lượng | Bắt buộc vì |
| --- | --- | --- |
| Control plane | **3** | Cần ít nhất **2** để có API server cũ sau khi nâng 1 cái. Lấy 3 để giữ **quorum etcd** khi một node bận nâng cấp — 2 node thì mất 1 là mất quorum |
| Load balancer | **1** | `kubeadm join --control-plane` cần `controlPlaneEndpoint` ổn định. Nó cũng là chỗ chứng minh triệu chứng "chập chờn" ở [§12.4](#124-qua-load-balancer--triệu-chứng-mà-người-dùng-thật-nhìn-thấy) |
| Worker | **1** | Để CoreDNS và workload có chỗ chạy ngoài control plane. Runbook không fault-inject trên worker |

Hai control plane **không đủ**: `kubeadm upgrade apply` trên một node kèm restart etcd member của
nó sẽ đưa cụm 2 member xuống dưới quorum.

### 2.2. Bảng VM và IP — chống trùng

LAN giả định `192.168.100.0/24`, gateway `192.168.100.1`. **Toàn bộ IP dưới đây phải nằm ngoài
DHCP pool của router**, cùng nguyên tắc mà [A3 của Lab 00](k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md#a3-đặt-ip-tĩnh-và-phân-giải-tên) áp cho dải `.221`–`.223`.

**Bản đồ IP đang bị chiếm trong repo — đọc trước khi cấp phát:**

| Dải | Ai đang giữ | Nguồn |
| --- | --- | --- |
| `.100`, `.101`, `.103` | 3 server hạ tầng Lab A | [runbook-k8s-vmware.md §2](runbook-k8s-vmware.md) |
| `.111` – `.113` | `k8s-master`, `k8s-worker1`, `k8s-worker2` | [runbook-k8s-vmware.md §2.2](runbook-k8s-vmware.md) |
| `.221` – `.223` | `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2` | [Lab 00 A1.2](k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm) |
| `.230` – `.235` | `lab-ha-lb`, `lab-ha-1` … `lab-ha-5` | [Lab 8b mục 2.1](k8s-docs/labs/LAB-8B-HA-VOI-STACKED-ETCD.md#21-bộ-vm-riêng-của-nhóm-lab-ha) |
| **`.240` – `.244`** | **runbook này** | bảng dưới |

**Bộ VM của runbook này:**

| Vai trò | Tên VM / hostname | IP | vCPU | RAM | Disk thin |
| --- | --- | --- | --- | --- | --- |
| Load balancer | `mvp-lb` | `192.168.100.240` | 1 | 1 GB | 20 GB |
| Control plane 1 — `kubeadm init`, **node được nâng cấp** | `mvp-cp1` | `192.168.100.241` | 2 | 3 GB | 40 GB |
| Control plane 2 — **giữ nguyên phiên bản cũ** | `mvp-cp2` | `192.168.100.242` | 2 | 3 GB | 40 GB |
| Control plane 3 — **giữ nguyên phiên bản cũ** | `mvp-cp3` | `192.168.100.243` | 2 | 3 GB | 40 GB |
| Worker | `mvp-w1` | `192.168.100.244` | 2 | 2 GB | 40 GB |

**Tổng cấp phát: 9 vCPU, 12 GB RAM, 180 GB đĩa danh nghĩa.**

Ba quy tắc bắt buộc:

- **Tiền tố `mvp-`** dành riêng cho runbook này. Không trùng `k8s-`, `lab-k8s-`, `lab-ha-`. Đừng
  đổi tên; tên node xuất hiện trong certificate SAN và trong `/etc/hosts` của cả năm máy.
- **Snapshot mang tiền tố `mvp-`**: `mvp-vm-ready`, `mvp-ha-1357`, `mvp-mixed`, `mvp-ha-1364`.
  Không lẫn với `01/02/03/04-` của chuỗi chính hay `8x-` của lab HA.
- **Không chạy song song** với bộ VM chuỗi chính (20 GB) hoặc bộ VM Lab 8b (14 GB) trừ khi host
  có RAM dư thật. Gate ở [§2.4](#24-gate-tài-nguyên-và-vm-khác-đã-tắt).

Pod CIDR `10.244.0.0/16` và Service CIDR `10.96.0.0/12` **trùng với các cluster khác trong repo** —
điều này không sao vì các cluster hoàn toàn tách biệt và không định tuyến sang nhau. Chỉ **IP node**
là bắt buộc không được trùng.

### 2.3. Bảng phiên bản

| Thành phần | Giá trị | Ghi chú |
| --- | --- | --- |
| Ubuntu ISO | `ubuntu-24.04.4-live-server-amd64.iso` | Cùng ISO của [Lab 00 A1.3](k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) — dùng lại checksum ở đó |
| Kubernetes **xuất phát** | `v1.35.7`, deb `1.35.7-1.1` | Bằng đúng baseline Lab 00, để mọi bước cài lại được y hệt |
| Kubernetes **đích** | `v1.36.4`, deb `1.36.4-1.1` | Patch 1.36 mới nhất tại 11/08/2026 |
| `cri-tools`, `kubernetes-cni`, `containerd`, `runc` | như [Lab 00 A1.3](k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) | Không chép lại con số ở đây |
| CNI | Flannel `v0.28.9` | Như Lab 00 |
| HAProxy | **không ghim** | Cùng lý do Lab 8b nêu: hạ tầng ngoài Kubernetes |

**Vì sao 1.35 → 1.36 chứ không phải cặp khác.** Đây là điểm kỹ thuật quan trọng nhất của cả
runbook:

| Phiên bản | Trạng thái `UnknownVersionInteroperabilityProxy` |
| --- | --- |
| v1.28 | alpha, **tắt mặc định** |
| **v1.35** | vẫn alpha, **tắt mặc định** |
| **v1.36** | **beta, bật mặc định** |

Bên phải proxy là **API server cũ** — chính là hai node còn ở 1.35. Nên feature gate **bắt buộc
phải được bật tay từ lúc `kubeadm init`**, khi cả cluster còn 1.35. Bật sau khi đã nâng cấp là
quá muộn: node cũ vẫn tắt tính năng và vẫn trả 404.

> **Cảnh báo cho tương lai.** Khi feature gate này graduate lên GA và bị khoá, việc truyền
> `--feature-gates=UnknownVersionInteroperabilityProxy=true` sẽ làm kube-apiserver **không khởi
> động được**. Ở 1.36 nó còn là beta nên truyền tường minh vẫn hợp lệ. Nếu bạn chạy runbook này
> với cặp phiên bản khác, kiểm tra trạng thái gate cho **cả hai** phiên bản trước.

Không hardcode con số patch một cách mù quáng — [§4.3](#43-ghim-version-và-gate) có gate đối chiếu
`apt-cache madison` trước khi cài.

### 2.4. Gate tài nguyên và VM khác đã tắt

Chạy trên **máy host Windows**, PowerShell:

```powershell
$vmrun = 'C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe'

# 1) Khong VM nao khac dang chay
$running = & $vmrun -T ws list | Select-Object -Skip 1
if ($running.Count -eq 0) { 'PASS: khong VM nao dang chay' }
else { "CANH BAO: dang chay $($running.Count) VM -> $($running -join ', ')" }

# 2) Con du RAM va dia
$freeGB = [math]::Round((Get-CimInstance Win32_OperatingSystem).FreePhysicalMemory/1MB, 1)
$diskGB = [math]::Round((Get-PSDrive E).Free/1GB, 1)
"RAM trong: $freeGB GB   |   Dia E: trong: $diskGB GB"
if ($freeGB -ge 13 -and $diskGB -ge 90) { 'PASS: du tai nguyen cho 5 VM' }
else { 'FAIL: thieu tai nguyen — tat bot VM hoac don dia' }
```

**PASS:** dòng `PASS: du tai nguyen cho 5 VM` xuất hiện. Ngưỡng 13 GB RAM là 12 GB cấp phát cộng
biên cho host; 90 GB đĩa là ước lượng thực tế của 180 GB thin-provisioned sau khi cài đủ phần mềm.

---

## 3. Dựng năm VM — lấy lại gì, thêm gì, bớt gì

**Runbook này không chép lại phần dựng OS.** Toàn bộ quy trình tạo VM, cài Ubuntu, reset
`machine-id`, đặt hostname, IP tĩnh và `/etc/hosts` đã có sẵn và đã được kiểm chứng ở hai chỗ.
Dùng lại, và chỉ đọc bảng khác biệt bên dưới.

### 3.1. Bảng đối chiếu với tài liệu có sẵn

| Việc | Lấy từ đâu | Runbook này **thêm / bớt** gì |
| --- | --- | --- |
| Tạo VM trên VMware, gắn ISO, cài Ubuntu | [runbook-k8s-vmware.md §3](runbook-k8s-vmware.md#3-tạo-3-vm-ubuntu-2404-trên-vmware) | **Thêm 2 VM**: 3 VM → **5 VM**. Cấu hình vCPU/RAM lấy theo bảng [§2.2](#22-bảng-vm-và-ip--chống-trùng), **không** theo bảng của runbook đó |
| Chuẩn hoá identity sau full-clone (`machine-id`, SSH host key, MAC, `product_uuid`) | [runbook-k8s-vmware.md §4](runbook-k8s-vmware.md#4-tạo-và-nhân-bản-3-server-theo-serversmd) và [Lab 00 A2.2](k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md) | **Không đổi.** Bắt buộc làm cho **cả 5 VM** — clone mà trùng `product_uuid` thì kubeadm preflight báo lỗi |
| Đặt hostname, IP tĩnh bằng netplan | [runbook-k8s-vmware.md §4](runbook-k8s-vmware.md) | **Thay toàn bộ tên và IP** bằng bảng [§2.2](#22-bảng-vm-và-ip--chống-trùng) |
| `/etc/hosts` | [Lab 00 A2.3](k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md#a23-đặt-hostname-etchosts-và-kiểm-tra-identity) | **Thêm dòng `mvp-lb`** — Lab 00 không có load balancer. Nội dung đầy đủ ở [§3.2](#32-etchosts-dùng-chung-cho-cả-năm-vm) |
| Tắt swap, module kernel, sysctl, containerd, kubeadm | [runbook-k8s-vmware.md §5](runbook-k8s-vmware.md#5-cấu-hình-os-chung-chạy-trên-cả-3-k8s-node) | **Bớt: KHÔNG chạy trên `mvp-lb`.** Chỉ 4 node Kubernetes. Xem [§4](#4-cài-runtime-và-kubeadm-trên-bốn-node-kubernetes) |
| HAProxy | [Lab 8b B3](k8s-docs/labs/LAB-8B-HA-VOI-STACKED-ETCD.md) | **Đổi backend** sang 3 IP mới, đổi tên server. Xem [§5](#5-load-balancer) |
| `kubeadm init` HA + `--upload-certs` | [Lab 8b B4](k8s-docs/labs/LAB-8B-HA-VOI-STACKED-ETCD.md) | **Thay `--control-plane-endpoint`, và thêm file `ClusterConfiguration` mang feature gate.** Đây là khác biệt lớn nhất. Xem [§6](#6-khởi-tạo-control-plane-với-mixed-version-proxy-bật-sẵn) |
| Flannel | [Lab 00 A5.2](k8s-docs/labs/LAB-00-MOI-TRUONG-1.35.7.md#a52-cài-flannel-v0289) | Không đổi |
| Nâng cấp bằng kubeadm | [runbook-k8s-vmware.md §15](runbook-k8s-vmware.md) | **Đổi hoàn toàn**: runbook đó nâng patch trong cùng minor trên 1 control plane. Ở đây là **nâng minor**, **chỉ một trong ba** control plane, rồi **dừng lại**. Xem [§10](#10-nâng-cấp-một-control-plane-rồi-dừng-lại) |

### 3.2. `/etc/hosts` dùng chung cho cả năm VM

Chạy trên **cả 5 VM**, kể cả `mvp-lb`:

```bash
sudo tee -a /etc/hosts >/dev/null <<'EOF'
192.168.100.240 mvp-lb
192.168.100.241 mvp-cp1
192.168.100.242 mvp-cp2
192.168.100.243 mvp-cp3
192.168.100.244 mvp-w1
EOF

# Xoa dong 127.0.1.1 tro ve chinh hostname neu cloud-init da sinh —
# kubeadm se lay nham dia chi loopback lam node IP.
sudo sed -i '/^127\.0\.1\.1/d' /etc/hosts

for h in mvp-lb mvp-cp1 mvp-cp2 mvp-cp3 mvp-w1; do
  ip="$(getent hosts "$h" | awk '{print $1}')"
  printf '%-8s -> %s\n' "$h" "$ip"
done
```

**PASS:** in đúng 5 dòng, mỗi dòng một IP trong dải `.240`–`.244`, **không** dòng nào ra
`127.0.1.1` hay rỗng.

### 3.3. Gate identity trước khi cài Kubernetes

Chạy trên **4 node Kubernetes** (`mvp-cp1`, `mvp-cp2`, `mvp-cp3`, `mvp-w1`):

```bash
echo "hostname   : $(hostname)"
echo "machine-id : $(cat /etc/machine-id)"
echo "product_uuid: $(sudo cat /sys/class/dmi/id/product_uuid)"
ip -br link show | grep -v LOOPBACK | awk '{print "mac        :", $1, $3}'
```

**PASS:** gom output của cả 4 máy lại, **không có hai máy nào trùng** `machine-id`,
`product_uuid` hay MAC. Trùng thì quay lại bước chuẩn hoá identity — kubeadm sẽ join sai node.

Snapshot cả 5 VM với tên **`mvp-vm-ready`** trước khi sang §4.

---

## 4. Cài runtime và kubeadm trên bốn node Kubernetes

> **KHÔNG chạy mục này trên `mvp-lb`.** Máy đó chỉ cần OS và HAProxy: không containerd, không
> kubeadm, không tắt swap. Đây là điểm khác biệt so với việc chạy "trên cả 3 node" của
> [runbook-k8s-vmware.md §5](runbook-k8s-vmware.md#5-cấu-hình-os-chung-chạy-trên-cả-3-k8s-node).

### 4.1. Phần lấy nguyên không đổi

Chạy y hệt [runbook-k8s-vmware.md §5](runbook-k8s-vmware.md#5-cấu-hình-os-chung-chạy-trên-cả-3-k8s-node)
trên **cả 4 node**: cập nhật OS, tắt swap vĩnh viễn, nạp module `overlay` + `br_netfilter`, đặt ba
sysctl, cài containerd theo đúng plugin path của containerd 2.x, bật `SystemdCgroup`.

### 4.2. Repo Kubernetes — điểm khác biệt duy nhất

Repo phải trỏ vào minor **xuất phát**, tức `v1.35`, **không phải** `v1.36`:

```bash
# Chay tren mvp-cp1, mvp-cp2, mvp-cp3, mvp-w1
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
```

**PASS:** `apt-get update` không báo lỗi GPG, và file `/etc/apt/sources.list.d/kubernetes.list`
chứa đúng chuỗi `v1.35`.

### 4.3. Ghim version và gate

```bash
# Chay tren 4 node
K8S_DEB='1.35.7-1.1'

apt-cache madison kubeadm | head -5
apt-cache madison kubeadm | grep -q " ${K8S_DEB} " \
  && echo "PASS: repo con cung cap $K8S_DEB" \
  || echo "FAIL: repo khong con $K8S_DEB — DUNG LAI, doi chieu lai bang 2.3"

sudo apt-get install -y \
  kubeadm="$K8S_DEB" kubelet="$K8S_DEB" kubectl="$K8S_DEB"
sudo apt-mark hold kubeadm kubelet kubectl

kubeadm version -o short
kubelet --version
```

**PASS:** dòng `PASS: repo con cung cap ...` xuất hiện, `kubeadm version -o short` in `v1.35.7`.
Thấy `FAIL` thì **dừng**: cập nhật bảng [§2.3](#23-bảng-phiên-bản) sau khi đối chiếu, không âm
thầm cài patch khác — cặp phiên bản xuất phát/đích là thứ cả runbook dựa vào.

---

## 5. Load balancer

Chạy trên `mvp-lb`. Cấu hình bên dưới là bản của [Lab 8b B3](k8s-docs/labs/LAB-8B-HA-VOI-STACKED-ETCD.md)
với **backend đổi sang ba IP mới** và **tên server đổi theo tiền tố `mvp-`**; phần còn lại giữ
nguyên vì bốn yêu cầu của bài [08](k8s-docs/08-high-availability-vi.md) không đổi.

```bash
# Chay tren mvp-lb
sudo apt-get update
sudo apt-get install -y haproxy
haproxy -v

sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig
sudo tee -a /etc/haproxy/haproxy.cfg >/dev/null <<'EOF'

#--- Runbook mixed-version-proxy: chuyen tiep TCP toi ba kube-apiserver ------
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
    server mvp-cp1 192.168.100.241:6443 check
    server mvp-cp2 192.168.100.242:6443 check
    server mvp-cp3 192.168.100.243:6443 check

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
ss -lntp | grep -E ':6443|:8404'
```

**PASS:** `haproxy -c` in `Configuration file is valid`; `is-active` trả `active`; `ss` thấy tiến
trình lắng nghe trên **cả** `6443` và `8404`.

> **`balance roundrobin` là bắt buộc ở đây, không phải tuỳ chọn.** Nó chính là thứ làm cho triệu
> chứng ở [§12.4](#124-qua-load-balancer--triệu-chứng-mà-người-dùng-thật-nhìn-thấy) hiện ra: cùng
> một lệnh chạy nhiều lần rơi vào các API server khác nhau. Đổi sang `balance source` là mất
> demo đó.

Kiểm tra trước khi init — lúc này chưa có API server nào nên **kỳ vọng là lỗi**:

```bash
# Chay tren mvp-lb
nc -vz 192.168.100.241 6443 ; nc -vz 192.168.100.242 6443 ; nc -vz 192.168.100.243 6443
```

**PASS:** cả ba trả `Connection refused`. Đây là kết quả **đúng**: máy sống, cổng chưa mở.
Nếu trả **timeout** thì đó là vấn đề mạng hoặc firewall — sửa trước khi đi tiếp, vì `kubeadm init`
sẽ treo.

---

## 6. Khởi tạo control plane với Mixed Version Proxy bật sẵn

### 6.1. Vì sao phải bật ngay bây giờ

Bên **cần** tính năng này là API server **cũ** — hai node sẽ còn ở 1.35 sau khi nâng cp1. Ở 1.35
gate còn alpha và **tắt mặc định** ([§2.3](#23-bảng-phiên-bản)).

Có ba cách bật, chỉ một cách đúng:

| Cách | Kết quả |
| --- | --- |
| Sửa tay `/etc/kubernetes/manifests/kube-apiserver.yaml` sau khi init | ❌ `kubeadm upgrade` sẽ ghi đè, mất cấu hình đúng lúc cần nhất |
| Bật sau khi đã nâng cấp cp1 | ❌ quá muộn — cp2/cp3 vẫn tắt tính năng, vẫn trả 404 |
| **Khai trong `ClusterConfiguration` từ `kubeadm init`** | ✅ nằm trong ConfigMap `kubeadm-config`, sống qua `upgrade apply` và `upgrade node` |

### 6.2. File cấu hình

```bash
# Chay tren mvp-cp1
cat > ~/kubeadm-mvp.yaml <<'EOF'
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: "v1.35.7"
controlPlaneEndpoint: "mvp-lb:6443"
networking:
  podSubnet: "10.244.0.0/16"
apiServer:
  extraArgs:
    - name: "feature-gates"
      value: "UnknownVersionInteroperabilityProxy=true"
    - name: "peer-ca-file"
      value: "/etc/kubernetes/pki/ca.crt"
EOF

kubeadm config validate --config ~/kubeadm-mvp.yaml
```

**PASS:** `kubeadm config validate` không in lỗi. Báo `unknown field` thì API config của bản
kubeadm bạn đang chạy không phải `v1beta4` — chạy `kubeadm config print init-defaults` để lấy đúng
`apiVersion` rồi sửa dòng đầu.

Bốn điểm cần hiểu trong file này:

- `extraArgs` ở `v1beta4` là **danh sách `name`/`value`**, không phải map. Viết dạng map là lỗi
  cú pháp.
- `peer-ca-file` là CA dùng để **API server nguồn xác thực serving certificate của peer**. Trên
  cluster kubeadm đó chính là CA của cluster: `/etc/kubernetes/pki/ca.crt`.
- **Không** khai `--peer-advertise-ip`. Bài 37 nói rõ: thiếu cờ này thì peer dùng
  `--advertise-address`, mà kubeadm đã đặt sẵn giá trị đó **theo từng node**. Khai tay sẽ phải
  dùng patch riêng cho mỗi máy — thêm việc, thêm chỗ sai.
- Bốn cờ còn lại mà bài 37 yêu cầu (`--proxy-client-cert-file`, `--proxy-client-key-file`,
  `--requestheader-client-ca-file`, `--requestheader-allowed-names`) **kubeadm đã đặt sẵn** cho
  aggregation layer. [§6.4](#64-gate-sáu-cờ-phải-có-mặt) xác minh chứ không tin suông.

### 6.3. `kubeadm init`

```bash
# Chay tren mvp-cp1
sudo kubeadm init --config ~/kubeadm-mvp.yaml --upload-certs | tee ~/kubeadm-init.log

mkdir -p "$HOME/.kube"
sudo cp -f /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"

kubectl get nodes
kubectl -n kube-system get pod -l component=kube-apiserver
```

**PASS:** `kubectl get nodes` in `mvp-cp1` (trạng thái `NotReady` là đúng, chưa có CNI). File
`~/kubeadm-init.log` chứa **hai** lệnh join khác nhau — một có `--control-plane --certificate-key`,
một không.

> **Certificate key hết hạn sau hai giờ.** Join hai control plane còn lại trong khoảng đó. Quá hạn
> thì sinh lại bằng `sudo kubeadm init phase upload-certs --upload-certs`.

### 6.4. Gate: sáu cờ phải có mặt

Đây là gate quan trọng nhất của cả runbook. Thiếu một cờ thì mọi thứ ở §12 sẽ trả 404 và bạn sẽ
đi tìm nhầm chỗ.

```bash
# Chay tren mvp-cp1
MANIFEST=/etc/kubernetes/manifests/kube-apiserver.yaml
MISSING=0
for f in \
  'UnknownVersionInteroperabilityProxy=true' \
  '--peer-ca-file=' \
  '--proxy-client-cert-file=' \
  '--proxy-client-key-file=' \
  '--requestheader-client-ca-file=' \
  '--requestheader-allowed-names='
do
  if sudo grep -q -- "$f" "$MANIFEST"; then
    echo "  co : $f"
  else
    echo "  THIEU: $f"; MISSING=$((MISSING+1))
  fi
done
test "$MISSING" -eq 0 \
  && echo 'PASS: du 6 co cho Mixed Version Proxy' \
  || echo "FAIL: thieu $MISSING co — sua ClusterConfiguration roi chay lai kubeadm init"

# advertise-address phai la IP that cua node, khong phai 0.0.0.0 hay 127.0.0.1
sudo grep -- '--advertise-address=' "$MANIFEST"
```

**PASS:** dòng `PASS: du 6 co cho Mixed Version Proxy`, và `--advertise-address=192.168.100.241`.
`advertise-address` sai là peer không tìm được nhau, dù mọi cờ khác đúng.

---

## 7. CNI và join ba node còn lại

```bash
# Chay tren mvp-cp1 — Flannel, dung ban Lab 00 dang dung
kubectl apply -f https://github.com/flannel-io/flannel/releases/download/v0.28.9/kube-flannel.yml
kubectl -n kube-flannel rollout status ds/kube-flannel-ds --timeout=180s
```

Join hai control plane — lấy **lệnh có** `--control-plane --certificate-key` từ `~/kubeadm-init.log`:

```bash
# Chay LAN LUOT tren mvp-cp2 roi mvp-cp3 — KHONG chay song song
sudo kubeadm join mvp-lb:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>
```

Join worker — lấy lệnh **không có** hai cờ đó:

```bash
# Chay tren mvp-w1
sudo kubeadm join mvp-lb:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

Gate sau khi join đủ:

```bash
# Chay tren mvp-cp1
kubectl get nodes -o wide
kubectl -n kube-system get pod -l component=kube-apiserver -o wide
kubectl -n kube-system get pod -l component=etcd -o wide
kubectl -n kube-system rollout restart deployment coredns
kubectl -n kube-system rollout status deployment coredns --timeout=180s

test "$(kubectl get nodes --no-headers | grep -c ' Ready ')" -eq 4 \
  && echo 'PASS: du 4 node Ready'
test "$(kubectl -n kube-system get pod -l component=kube-apiserver --no-headers | wc -l)" -eq 3 \
  && echo 'PASS: du 3 kube-apiserver'
```

**PASS:** hai dòng `PASS:` xuất hiện; ba Pod `kube-apiserver` nằm trên ba node khác nhau; ba Pod
`etcd` cũng vậy.

Lặp lại gate [§6.4](#64-gate-sáu-cờ-phải-có-mặt) trên **`mvp-cp2` và `mvp-cp3`**. Feature gate
phải có mặt trên **cả ba** — hai node cũ mới là bên thật sự dùng nó.

Snapshot cả 5 VM: **`mvp-ha-1357`**.

---

## 8. Ba kubeconfig trỏ thẳng vào từng API server

Đây là công cụ then chốt: qua load balancer bạn **không kiểm soát được** request rơi vào server
nào, nên không chứng minh được gì. Ba kubeconfig dưới đây trỏ thẳng vào từng API server.

```bash
# Chay tren mvp-cp1
mkdir -p ~/mvp-evidence
cd ~/mvp-evidence

for i in 1 2 3; do
  ip="192.168.100.24$i"
  cp /etc/kubernetes/admin.conf "cp$i.conf"
  kubectl --kubeconfig="cp$i.conf" config set-cluster kubernetes \
    --server="https://$ip:6443" >/dev/null
  echo -n "cp$i -> "
  kubectl --kubeconfig="cp$i.conf" get --raw /version | grep -o '"gitVersion":"[^"]*"'
done
```

**PASS:** ba dòng, mỗi dòng in `"gitVersion":"v1.35.7"`. Kết nối thẳng được vì kubeadm đã đưa IP
của từng node vào SAN của serving certificate.

Bộ file cho `curl` — cần để đặt được `Accept` header, thứ `kubectl get --raw` không làm được:

```bash
# Chay tren mvp-cp1, trong ~/mvp-evidence
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d > ca.crt
kubectl config view --raw -o jsonpath='{.users[0].user.client-certificate-data}'          | base64 -d > client.crt
kubectl config view --raw -o jsonpath='{.users[0].user.client-key-data}'                  | base64 -d > client.key
chmod 600 client.key

curl -sS --cacert ca.crt --cert client.crt --key client.key \
  https://192.168.100.242:6443/version | head -c 120; echo
```

**PASS:** trả JSON có `gitVersion`. Lỗi TLS ở đây là do trích sai file, không phải do cluster.

> **`client.key` là khoá của `kubernetes-admin`.** Nó nằm trong `~/mvp-evidence` với quyền `600`.
> Đừng copy ra ngoài, và xoá cả thư mục ở [§15](#15-dọn-dẹp-và-đường-quay-lại).

---

## 9. Ảnh chụp "trước" — cluster đồng phiên bản

Chụp trạng thái để về sau có cái đối chiếu.

```bash
# Chay tren mvp-cp1, trong ~/mvp-evidence
kubectl get nodes -o custom-columns=\
'NODE:.metadata.name,KUBELET:.status.nodeInfo.kubeletVersion' | tee 09-nodes-before.txt

for i in 1 2 3; do
  echo -n "cp$i apiserver: "
  kubectl --kubeconfig="cp$i.conf" get --raw /version \
    | grep -o '"gitVersion":"[^"]*"'
done | tee 09-apiserver-before.txt

# Danh sach GVR ma tung API server tu phuc vu duoc (non peer-aggregated)
for i in 1 2 3; do
  curl -sS --cacert ca.crt --cert client.crt --key client.key \
    -H 'Accept: application/json;g=apidiscovery.k8s.io;v=v2;as=APIGroupDiscoveryList;profile=nopeer' \
    "https://192.168.100.24$i:6443/apis" \
  | grep -o '"name":"[^"]*"' | sort -u > "09-gvr-cp$i.txt"
  echo "cp$i: $(wc -l < 09-gvr-cp$i.txt) muc"
done

diff 09-gvr-cp1.txt 09-gvr-cp2.txt && echo 'PASS: cp1 va cp2 phuc vu y het nhau'
diff 09-gvr-cp1.txt 09-gvr-cp3.txt && echo 'PASS: cp1 va cp3 phuc vu y het nhau'
```

**PASS:** ba API server cùng `v1.35.7`, và **hai lệnh `diff` không in gì** kèm hai dòng `PASS:`.
Đây là ảnh chụp "chưa có bất đối xứng" — mọi thứ ở §12 sẽ so lại với nó.

---

## 10. Nâng cấp một control plane rồi dừng lại

> **Đây là điểm khác biệt cốt lõi so với mọi tài liệu khác trong repo.** Quy trình nâng cấp chuẩn
> ở [runbook-k8s-vmware.md §15](runbook-k8s-vmware.md) và ở checkpoint
> [giai đoạn 17](k8s-docs/00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster) đều **chạy hết**
> mọi node. Ở đây ta **cố ý dừng sau node đầu tiên** và giữ nguyên trạng thái đó.

### 10.1. Snapshot trước

Trạng thái hỗn hợp mất khoảng 20 phút để dựng lại. Snapshot cả 5 VM ngay bây giờ nếu chưa làm ở
cuối §7 — đây là đường lui duy nhất, runbook **không** dựng backup etcd
([nợ #8](k8s-docs/labs/README.md#5-sổ-nợ-lab)).

### 10.2. Đổi repo và cài kubeadm 1.36 — **chỉ trên `mvp-cp1`**

```bash
# Chay CHI tren mvp-cp1
sudo sed -i 's#core:/stable:/v1.35#core:/stable:/v1.36#' /etc/apt/sources.list.d/kubernetes.list
grep . /etc/apt/sources.list.d/kubernetes.list

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key \
  | sudo gpg --dearmor --yes -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

sudo apt-get update

TARGET_DEB='1.36.4-1.1'
TARGET_K8S='v1.36.4'

apt-cache madison kubeadm | head -5
apt-cache madison kubeadm | grep -q " ${TARGET_DEB} " \
  && echo "PASS: repo v1.36 co $TARGET_DEB" \
  || echo 'FAIL: doi chieu lai bang 2.3 truoc khi di tiep'

sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm="$TARGET_DEB"
sudo apt-mark hold kubeadm
kubeadm version -o short
```

**PASS:** `kubeadm version -o short` in `v1.36.4`. **Hai máy `mvp-cp2` và `mvp-cp3` vẫn phải ở
repo `v1.35` và kubeadm `v1.35.7`** — kiểm lại nếu lỡ tay chạy nhầm SSH.

### 10.3. `kubeadm upgrade apply`

```bash
# Chay CHI tren mvp-cp1
sudo kubeadm upgrade plan | tee ~/mvp-evidence/10-upgrade-plan.txt
sudo kubeadm upgrade apply "$TARGET_K8S" -y | tee ~/mvp-evidence/10-upgrade-apply.txt
```

Nâng **kubelet của chính cp1** cho khớp:

```bash
# Chay CHI tren mvp-cp1
kubectl drain mvp-cp1 --ignore-daemonsets
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet="$TARGET_DEB" kubectl="$TARGET_DEB"
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon mvp-cp1
```

> **Dừng ở đây. Không chạy `kubeadm upgrade node` trên cp2 và cp3.** Đó chính là trạng thái mà
> runbook này tồn tại để dựng.

### 10.4. Gate: cluster đang thật sự ở trạng thái hỗn hợp

```bash
# Chay tren mvp-cp1, trong ~/mvp-evidence
for i in 1 2 3; do
  echo -n "cp$i apiserver: "
  kubectl --kubeconfig="cp$i.conf" get --raw /version \
    | grep -o '"gitVersion":"[^"]*"'
done | tee 10-apiserver-mixed.txt

VERS="$(cut -d'"' -f4 10-apiserver-mixed.txt | sort -u | wc -l)"
test "$VERS" -eq 2 \
  && echo 'PASS: dung 2 phien ban kube-apiserver dang chay cung luc' \
  || echo "FAIL: dang co $VERS phien ban — khong phai trang thai hon hop"

kubectl get nodes -o custom-columns=\
'NODE:.metadata.name,KUBELET:.status.nodeInfo.kubeletVersion'
```

**PASS:** dòng `PASS: dung 2 phien ban ...`; cp1 ở `v1.36.4`, cp2 và cp3 ở `v1.35.7`.

Snapshot cả 5 VM: **`mvp-mixed`**. Đây là mốc quý nhất của runbook — mọi thí nghiệm ở §12 và §13
đều quay về đây.

### 10.5. Feature gate còn sống sau nâng cấp không

`kubeadm upgrade apply` **sinh lại** manifest static Pod. Không kiểm là không biết cấu hình có bị
rơi hay không.

```bash
# Chay tren mvp-cp1
sudo grep -c 'UnknownVersionInteroperabilityProxy=true' \
  /etc/kubernetes/manifests/kube-apiserver.yaml
sudo grep -c -- '--peer-ca-file=' /etc/kubernetes/manifests/kube-apiserver.yaml
```

**PASS:** cả hai lệnh in `1`. In `0` thì `extraArgs` đã rơi khỏi ConfigMap `kubeadm-config` —
xem [§16](#16-troubleshooting).

Kiểm cả hai node cũ, vì chúng mới là bên proxy:

```bash
# Chay tren mvp-cp2 VA mvp-cp3
sudo grep -c 'UnknownVersionInteroperabilityProxy=true' \
  /etc/kubernetes/manifests/kube-apiserver.yaml
```

**PASS:** mỗi máy in `1`.

---

## 11. Tìm API bất đối xứng

Muốn chứng minh proxy thì phải có một **API mà cp1 phục vụ được còn cp2/cp3 thì không**. Hai
đường, làm Đường A trước; nếu delta rỗng thì dùng Đường B.

### 11.1. Đường A — delta thật giữa hai phiên bản

Không đoán API nào mới. Đo trực tiếp:

```bash
# Chay tren mvp-cp1, trong ~/mvp-evidence
for i in 1 2; do
  curl -sS --cacert ca.crt --cert client.crt --key client.key \
    -H 'Accept: application/json;g=apidiscovery.k8s.io;v=v2;as=APIGroupDiscoveryList;profile=nopeer' \
    "https://192.168.100.24$i:6443/apis" \
  | tr ',' '\n' | grep -o '"name":"[^"]*"' | sort -u > "11-nopeer-cp$i.txt"
done

echo '--- co tren cp1 (v1.36.4) ma KHONG co tren cp2 (v1.35.7) ---'
comm -23 11-nopeer-cp1.txt 11-nopeer-cp2.txt | tee 11-delta.txt

test -s 11-delta.txt \
  && echo 'PASS: co API bat doi xung — dung Duong A' \
  || echo 'TRONG: khong co delta — chuyen sang Duong B (11.2)'
```

Nếu có delta, chọn **một** group/version/resource từ đó và đặt vào ba biến. Ví dụ nếu delta cho
thấy `resource.k8s.io/v1beta2`:

```bash
# Chay tren mvp-cp1 — thay bang gia tri THAT lay tu 11-delta.txt
GRP='resource.k8s.io'
VER='v1beta2'
RES='resourceclaims'
echo "$GRP/$VER/$RES" | tee ~/mvp-evidence/11-target.txt
```

> **Delta rỗng là kết quả hợp lệ, không phải lỗi.** Hai minor kề nhau có thể không thêm API group
> mới nào được phục vụ mặc định. Đó chính là lý do Đường B tồn tại.

### 11.2. Đường B — tạo bất đối xứng có kiểm soát

Bài 37 nêu **hai** nguyên nhân khiến API server không phục vụ được một resource: *"vì server này ở
phiên bản có trước khi API được giới thiệu, **hoặc API bị tắt trên API server đó**"*. Đường B dùng
nguyên nhân thứ hai.

Cách an toàn nhất là **bật thêm** một API chỉ trên cp1, chứ không tắt bớt trên cp2/cp3 — bật thêm
không thể làm hỏng thứ đang chạy.

```bash
# Chay CHI tren mvp-cp1
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml \
        /root/kube-apiserver.yaml.before-runtimeconfig

# Neo dung dong '    - kube-apiserver'. Phai in ra 1 — chen nham dong khac
# thi kube-apiserver khong khoi dong lai duoc.
sudo grep -c '^    - kube-apiserver$' /etc/kubernetes/manifests/kube-apiserver.yaml

sudo sed -i \
  '/^    - kube-apiserver$/a\    - --runtime-config=storagemigration.k8s.io/v1beta1=true' \
  /etc/kubernetes/manifests/kube-apiserver.yaml

sudo sed -i \
  's/UnknownVersionInteroperabilityProxy=true/UnknownVersionInteroperabilityProxy=true,StorageVersionMigrator=true/' \
  /etc/kubernetes/manifests/kube-apiserver.yaml

# kubelet tu khoi dong lai static Pod; cho no len
sleep 45
kubectl --kubeconfig="$HOME"/mvp-evidence/cp1.conf get --raw /version >/dev/null \
  && echo 'PASS: apiserver cp1 da len lai'
```

Gate bất đối xứng — đây mới là điều kiện phải đạt:

```bash
# Chay tren mvp-cp1, trong ~/mvp-evidence
GRP='storagemigration.k8s.io'
VER='v1beta1'
RES='storageversionmigrations'

for i in 1 2; do
  echo -n "cp$i tu phuc vu $GRP/$VER: "
  curl -sS -o /dev/null -w '%{http_code}\n' \
    --cacert ca.crt --cert client.crt --key client.key \
    -H 'Accept: application/json;g=apidiscovery.k8s.io;v=v2;as=APIGroupDiscoveryList;profile=nopeer' \
    "https://192.168.100.24$i:6443/apis/$GRP/$VER"
done
```

**PASS:** cp1 trả `200`, cp2 trả `404`. Đúng hai giá trị đó mới là bất đối xứng thật.

- cp1 trả `404` → API chưa bật; kiểm lại `--runtime-config` và feature gate trong manifest.
- cp2 trả `200` → không có bất đối xứng; chọn group/version khác.

> **`--runtime-config` sửa tay sẽ bị `kubeadm upgrade node` ghi đè** ở §14. Đó là hành vi đúng và
> nằm trong kế hoạch: §14 chính là lúc bạn muốn xoá bất đối xứng đi.

---

## 12. Chứng minh Mixed Version Proxy

Ba biến `GRP`, `VER`, `RES` lấy từ §11. Mọi lệnh chạy trên `mvp-cp1`, trong `~/mvp-evidence`.

### 12.1. Peer-aggregated discovery so với nopeer

```bash
echo '--- cp2 tu phuc vu duoc gi (nopeer) ---'
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  --cacert ca.crt --cert client.crt --key client.key \
  -H 'Accept: application/json;g=apidiscovery.k8s.io;v=v2;as=APIGroupDiscoveryList;profile=nopeer' \
  "https://192.168.100.242:6443/apis/$GRP/$VER"

echo '--- cp2 nhin thay gi khi tong hop tu peer (mac dinh) ---'
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  --cacert ca.crt --cert client.crt --key client.key \
  -H 'Accept: application/json;g=apidiscovery.k8s.io;v=v2;as=APIGroupDiscoveryList' \
  "https://192.168.100.242:6443/apis/$GRP/$VER"
```

**PASS:** lệnh đầu `HTTP 404`, lệnh sau `HTTP 200`.

Hai con số này là **Peer-aggregated discovery**: cùng một API server, cùng một đường dẫn, chỉ khác
`Accept` header. Không có `profile=nopeer`, cp2 trả lời thay cho **cả cluster**; có `profile=nopeer`,
nó chỉ khai những gì bản thân nó phục vụ được.

> Theo ghi chú của bài 37, peer-aggregated discovery **chỉ áp dụng cho Aggregated Discovery tới
> `/apis`**, không áp dụng cho Unaggregated (Legacy) Discovery. Đừng thay bằng đường dẫn khác.

### 12.2. Request thật tới API server cũ

```bash
echo -n 'cp1 (v1.36.4, phuc vu tai cho): '
kubectl --kubeconfig=cp1.conf get --raw "/apis/$GRP/$VER/$RES" >/dev/null \
  && echo 'OK 200'

echo -n 'cp2 (v1.35.7, phai proxy)     : '
kubectl --kubeconfig=cp2.conf get --raw "/apis/$GRP/$VER/$RES" >/dev/null \
  && echo 'OK 200'

echo -n 'cp3 (v1.35.7, phai proxy)     : '
kubectl --kubeconfig=cp3.conf get --raw "/apis/$GRP/$VER/$RES" >/dev/null \
  && echo 'OK 200'
```

**PASS:** **cả ba** dòng in `OK 200`.

Đọc cho đúng kết quả này: cp2 và cp3 **không hề biết** resource đó — §12.1 vừa chứng minh bằng
`404` ở chế độ nopeer. Chúng trả `200` được **chỉ vì** đã proxy sang cp1.

### 12.3. Metric — bằng chứng không thể chối

Hai dòng trên mới chỉ cho thấy *kết quả*. Metric cho thấy *đường đi*.

```bash
kubectl --kubeconfig=cp2.conf get --raw /metrics \
  | grep '^apiserver_rerouted_request_total' \
  | tee 12-rerouted-cp2.txt
```

**PASS:** có ít nhất một dòng, với `group`, `version`, `resource` khớp `GRP/VER/RES` và giá trị
đếm lớn hơn 0.

Chạy lại lệnh ở §12.2 năm lần rồi đọc lại metric — con số phải **tăng**:

```bash
for n in 1 2 3 4 5; do
  kubectl --kubeconfig=cp2.conf get --raw "/apis/$GRP/$VER/$RES" >/dev/null
done
kubectl --kubeconfig=cp2.conf get --raw /metrics \
  | grep '^apiserver_rerouted_request_total'
```

**PASS:** giá trị đếm cho đúng GVR đó tăng thêm 5.

> Dòng metric có cả ba nhãn `group`, `version`, `resource` **rỗng** là request **discovery** được
> proxy, không phải request tài nguyên. Đừng nhầm hai loại.

Kiểm ngược trên cp1 — nó phục vụ tại chỗ nên không được reroute:

```bash
kubectl --kubeconfig=cp1.conf get --raw /metrics \
  | grep -c "^apiserver_rerouted_request_total.*resource=\"$RES\""
```

**PASS:** in `0`.

### 12.4. Qua load balancer — triệu chứng mà người dùng thật nhìn thấy

Đây là lý do tính năng này tồn tại. Client thật không trỏ vào từng API server; nó đi qua LB và
rơi vào node ngẫu nhiên.

```bash
for n in $(seq 1 12); do
  kubectl get --raw "/apis/$GRP/$VER/$RES" >/dev/null 2>&1 \
    && echo "lan $n: OK" || echo "lan $n: LOI"
done
```

**PASS:** cả 12 lần đều `OK`, dù `balance roundrobin` đã rải request qua cả ba API server.

Để thấy nó **sẽ** hỏng ra sao nếu không có tính năng này, xem §13.1 — tắt gate trên hai node cũ
rồi chạy lại đúng vòng lặp trên: kết quả trở thành **chập chờn**, khoảng một phần ba số lần lỗi.
Đó chính là "các lỗi 404 Not Found bất ngờ bắt nguồn từ quá trình nâng cấp" mà bài 37 nói tới —
và là loại lỗi khó chẩn đoán nhất, vì chạy lại lệnh thì nó lại hết.

Ghi lại phân bố của LB để chứng minh request thật sự có rải:

```bash
curl -sS 'http://192.168.100.240:8404/stats;csv' \
  | awk -F, '$1=="k8s-controlplane"{print $2, $8}' \
  | tee 12-lb-distribution.txt
```

**PASS:** cả ba dòng `mvp-cp1`, `mvp-cp2`, `mvp-cp3` đều có số phiên lớn hơn 0.

---

## 13. Fault injection: 404 và 503

> **Fault injection chỉ chạy trên `mvp-cp2`.** Giữ `mvp-cp3` nguyên vẹn làm đối chứng — đúng
> nguyên tắc "một node hỏng, một node đối chứng" mà các lab trong repo dùng.

### 13.1. Không có tính năng → 404

```bash
# Chay tren mvp-cp2
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.mvp-on
sudo sed -i 's/UnknownVersionInteroperabilityProxy=true/UnknownVersionInteroperabilityProxy=false/' \
  /etc/kubernetes/manifests/kube-apiserver.yaml
sleep 45
```

```bash
# Chay tren mvp-cp1, trong ~/mvp-evidence
echo -n 'cp2 (da TAT tinh nang): '
curl -sS -o /dev/null -w '%{http_code}\n' \
  --cacert ca.crt --cert client.crt --key client.key \
  "https://192.168.100.242:6443/apis/$GRP/$VER/$RES"

echo -n 'cp3 (van BAT tinh nang): '
curl -sS -o /dev/null -w '%{http_code}\n' \
  --cacert ca.crt --cert client.crt --key client.key \
  "https://192.168.100.243:6443/apis/$GRP/$VER/$RES"
```

**PASS:** cp2 trả `404`, cp3 trả `200`. Cùng một cluster, cùng một request, cùng một phiên bản
API server — khác nhau đúng một feature gate.

Chạy lại vòng lặp 12 lần qua load balancer ở [§12.4](#124-qua-load-balancer--triệu-chứng-mà-người-dùng-thật-nhìn-thấy):
bây giờ phải có cả `OK` lẫn `LOI` xen kẽ. Đó là triệu chứng chập chờn.

Khôi phục cp2:

```bash
# Chay tren mvp-cp2
sudo cp /root/kube-apiserver.yaml.mvp-on /etc/kubernetes/manifests/kube-apiserver.yaml
sleep 45
```

**PASS:** request tới cp2 trả lại `200`.

### 13.2. Peer chết → 503

Bài 37: *"Nếu peer API server không phản hồi, API server nguồn sẽ trả về lỗi 503"*.

```bash
# Chay tren mvp-cp1 — ha kube-apiserver cua chinh cp1, la peer duy nhat biet API do
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.parked
sleep 30
```

```bash
# Van chay tren mvp-cp1: may con song, chi kube-apiserver cua no tat.
# curl khong can apiserver cuc bo — no goi thang sang cp2.
curl -sS -o /dev/null -w '%{http_code}\n' \
  --cacert ca.crt --cert client.crt --key client.key \
  "https://192.168.100.242:6443/apis/$GRP/$VER/$RES"
```

**PASS:** trả `503`.

Trả `404` cũng là kết quả **hợp lệ và đáng ghi lại**: nó nghĩa là cache discovery của cp2 đã kịp
cập nhật rằng không còn peer nào phục vụ được API đó, nên request rơi vào chuỗi handler cục bộ và
kết thúc bằng 404 — đúng nhánh thứ ba của [§1.2](#12-cơ-chế-đúng-theo-bài-37). Ranh giới giữa 503
và 404 ở đây là **thời điểm**, phụ thuộc chu kỳ cập nhật thông tin peer; đừng ghi con số thời gian
như một cam kết. Chạy ngay sau khi tắt thì thường được 503, chờ lâu hơn thì thường ra 404.

Khôi phục:

```bash
# Chay tren mvp-cp1
sudo mv /root/kube-apiserver.yaml.parked /etc/kubernetes/manifests/kube-apiserver.yaml
sleep 45
kubectl get nodes
```

**PASS:** `kubectl get nodes` chạy lại được, đủ 4 node `Ready`.

---

## 14. Hoàn tất nâng cấp

Chỉ làm mục này khi đã ghi đủ bằng chứng — nó **xoá bỏ** trạng thái hỗn hợp. Snapshot `mvp-mixed`
là đường quay lại.

```bash
# Chay LAN LUOT tren mvp-cp2 roi mvp-cp3
sudo sed -i 's#core:/stable:/v1.35#core:/stable:/v1.36#' /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key \
  | sudo gpg --dearmor --yes -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo apt-get update

TARGET_DEB='1.36.4-1.1'
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm="$TARGET_DEB"
sudo apt-mark hold kubeadm

sudo kubeadm upgrade node          # KHONG phai "upgrade apply"

# tu mvp-cp1: kubectl drain <node>
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet="$TARGET_DEB" kubectl="$TARGET_DEB"
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
# tu mvp-cp1: kubectl uncordon <node>
```

Rồi `mvp-w1` với đúng trình tự đó.

```bash
# Chay tren mvp-cp1
for i in 1 2 3; do
  echo -n "cp$i: "
  kubectl --kubeconfig="$HOME"/mvp-evidence/cp$i.conf get --raw /version \
    | grep -o '"gitVersion":"[^"]*"'
done

kubectl get nodes -o custom-columns=\
'NODE:.metadata.name,KUBELET:.status.nodeInfo.kubeletVersion'
```

**PASS:** ba API server cùng `v1.36.4`; cả 4 node kubelet `v1.36.4`; không còn bất đối xứng.

Chạy lại §12.2 để thấy điều ngược lại: cả ba đều `200` — nhưng lần này vì **cả ba đều tự phục
vụ được**, không phải nhờ proxy. Xác nhận bằng metric:

```bash
kubectl --kubeconfig="$HOME"/mvp-evidence/cp2.conf get --raw /metrics \
  | grep "^apiserver_rerouted_request_total.*resource=\"$RES\"" || echo 'PASS: khong con reroute'
```

**PASS:** con số không tăng thêm sau khi chạy lại §12.2 (dòng cũ vẫn còn vì metric là bộ đếm
tích luỹ; điều cần kiểm là nó **đứng yên**).

Snapshot cả 5 VM: **`mvp-ha-1364`**.

---

## 15. Dọn dẹp và đường quay lại

```bash
# Chay tren mvp-cp1
shred -u ~/mvp-evidence/client.key 2>/dev/null || rm -f ~/mvp-evidence/client.key
rm -f ~/mvp-evidence/cp1.conf ~/mvp-evidence/cp2.conf ~/mvp-evidence/cp3.conf
ls ~/mvp-evidence
```

Giữ lại các file `.txt` làm hồ sơ; **xoá hết** kubeconfig và khoá client.

| Muốn gì | Làm gì |
| --- | --- |
| Chạy lại thí nghiệm §12–§13 | Restore snapshot **`mvp-mixed`** trên cả 5 VM, tạo lại kubeconfig ở §8 |
| Chạy lại từ cluster đồng phiên bản | Restore **`mvp-ha-1357`**, làm lại từ §10 |
| Dựng lại từ OS trắng | Restore **`mvp-vm-ready`**, làm lại từ §4 |
| Bỏ hẳn | Tắt và xoá cả 5 VM. Không ảnh hưởng cluster nào khác — IP `.240`–`.244` không ai dùng |

> **Bốn snapshot phải cùng tên trên cả năm VM.** Restore lệch mốc giữa các máy là cách nhanh nhất
> để có một cluster hỏng theo kiểu khó chẩn đoán.

---

## 16. Troubleshooting

Sự cố dựng môi trường chung (VM không lên mạng, clone trùng identity, containerd không chạy) tra ở
[runbook-k8s-vmware.md §15](runbook-k8s-vmware.md#15-vận-hành--troubleshooting). Bảng dưới chỉ ghi
sự cố **riêng của tình huống phiên bản hỗn hợp**.

| Triệu chứng | Nguyên nhân thường gặp | Cách xử lý |
| --- | --- | --- |
| §12.2 cho `404` ở cp2 và cp3 | Feature gate không có trên **node cũ** | Chạy lại gate [§10.5](#105-feature-gate-còn-sống-sau-nâng-cấp-không) trên cp2/cp3. Thiếu thì sửa ConfigMap `kubeadm-config` rồi `kubeadm upgrade node` lại node đó |
| Gate §6.4 báo thiếu `--peer-ca-file` | `extraArgs` viết dạng map thay vì list `name`/`value` | Sửa `~/kubeadm-mvp.yaml` theo đúng [§6.2](#62-file-cấu-hình), `kubeadm reset` rồi init lại |
| `kube-apiserver` không khởi động sau khi sửa manifest | Feature gate đã GA và bị khoá ở phiên bản đang chạy | `sudo crictl logs $(sudo crictl ps -a --name kube-apiserver -q \| head -1)`. Nếu log báo gate bị khoá thì bỏ hẳn `--feature-gates` — phiên bản đó đã bật mặc định |
| §11.1 delta rỗng | Hai minor không thêm API group nào phục vụ mặc định | Chuyển sang [Đường B](#112-đường-b--tạo-bất-đối-xứng-có-kiểm-soát). Đây là kết quả hợp lệ |
| §11.2 cp2 cũng trả `200` | Group/version chọn nhầm — nó vốn đã bật ở cả hai phiên bản | Chọn group khác từ danh sách alpha/beta chưa bật của baseline |
| Metric `apiserver_rerouted_request_total` không tồn tại | Đọc metric từ **cp1** thay vì cp2/cp3 | Metric chỉ xuất hiện ở API server **đi proxy** |
| §13.2 luôn ra `404` chứ không ra `503` | Cache thông tin peer đã cập nhật xong | Restore `mvp-mixed`, tắt apiserver cp1 rồi gọi **ngay lập tức**. Cả hai mã đều là hành vi đúng |
| `kubeadm upgrade apply` báo version skew | Nhảy quá một minor, hoặc kubelet quá cũ | kubeadm không skip minor. Nâng từng bậc: 1.35 → 1.36 → 1.37 |
| Cluster mất API sau khi drain cp1 | Chỉ còn 2 etcd member, mất quorum khi restart | Đúng lý do runbook cần **3** control plane ([§2.1](#21-vì-sao-năm-vm-và-vì-sao-không-ít-hơn)). Restore snapshot |
| `curl` báo lỗi certificate khi gọi thẳng IP | IP node không nằm trong SAN của serving cert | `sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text \| grep -A1 'Subject Alternative Name'`. Thiếu thì init lại với `apiServer.certSANs` |

---

## 17. Nguồn official

- [Mixed Version Proxy](https://kubernetes.io/docs/concepts/architecture/mixed-version-proxy/) — bản dịch: [bài 37](k8s-docs/37-mixed-version-proxy-vi.md)
- [Kubernetes v1.36: Mixed Version Proxy Graduates to Beta](https://kubernetes.io/blog/2026/05/15/kubernetes-1-36-feature-mixed-version-proxy-beta/) — mốc alpha → beta và việc bật mặc định
- [Kubernetes 1.28: A New (alpha) Mechanism For Safer Cluster Upgrades](https://kubernetes.io/blog/2023/08/28/kubernetes-1-28-feature-mixed-version-proxy-alpha/) — bối cảnh ban đầu
- [Feature Gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) — trạng thái `UnknownVersionInteroperabilityProxy`
- [Creating Highly Available Clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/) — bản dịch: [bài 08](k8s-docs/08-high-availability-vi.md)
- [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) — bản dịch: [bài 221](k8s-docs/221-kubeadm-upgrade-vi.md)
- [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/) — biên độ lệch phiên bản cho phép giữa các API server
- [kubeadm Configuration (v1beta4)](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/) — dạng list của `extraArgs`
- [Kubernetes Metrics Reference](https://kubernetes.io/docs/reference/instrumentation/metrics/) — `apiserver_rerouted_request_total`
- [Releases](https://kubernetes.io/releases/) — patch mới nhất của từng minor
