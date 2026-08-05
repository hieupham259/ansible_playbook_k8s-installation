# Runbook lab 1a — Kiến trúc và mô hình điều khiển Kubernetes

> **Baseline:** Ubuntu Server 24.04.4 LTS, Kubernetes v1.35.6, containerd 2.2.1,
> Flannel v0.28.7, một control plane và hai worker chạy bằng VM trên Windows.
>
> **Cập nhật và đối chiếu phiên bản:** 05/08/2026.

Lab này đi cùng mục [1a. Kiến trúc và mô hình điều khiển](../LO-TRINH-ADMIN.md#1a-kiến-trúc-và-mô-hình-điều-khiển).
Cluster được dựng riêng cho việc học; không dùng minikube và không phụ thuộc cluster có sẵn.

Việc cài OS, container runtime, kubeadm và CNI trong phần A chỉ là **chuẩn bị môi trường**.
Không cần hiểu sâu các thao tác cài đặt này ở giai đoạn 1; nội dung đó sẽ được học tại giai
đoạn 2, 5 và 8. Phần bắt buộc phải hiểu của lab bắt đầu từ phần B.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Kubernetes tự động hóa việc gì và không thay thế những gì.
- Thành phần nào thuộc control plane, thành phần nào chạy trên node.
- `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`, `kubelet`,
  container runtime, CNI, CoreDNS và `kube-proxy` nằm ở đâu.
- Một object có `metadata`, `spec`, `status`; `spec` là trạng thái mong muốn còn `status`
  phản ánh trạng thái quan sát được.
- Cách khám phá core API, named API groups, resource và version qua API server.
- Node đăng ký với cluster như thế nào; `Condition`, `Capacity`, `Allocatable` và heartbeat
  thể hiện ở đâu.
- Chiều `kubectl → API server`, `kubelet → API server` và `API server → kubelet`.
- Controller là vòng lặp quan sát, so sánh và hành động để đưa trạng thái thực tế về trạng
  thái mong muốn.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong 1a | Phần lab kiểm chứng |
| --- | --- |
| Tổng quan | B1 — Từ nhu cầu đến khả năng Kubernetes |
| Các thành phần của Kubernetes | B2 — Kiểm kê component |
| Kiến trúc cluster | B2 và B3 — Vị trí, process và static Pod |
| Các đối tượng trong Kubernetes | B4 — Đọc object và `spec/status` |
| Kubernetes API | B5 — Discovery, group, version và validation |
| Các Node | B6 — Node status, đăng ký và heartbeat |
| Giao tiếp giữa Node và Control Plane | B7 — Quan sát ba chiều giao tiếp |
| Các Controller | B8 — Reconciliation thật |

### 1.2. Thời lượng

- Dựng VM và cluster lần đầu: khoảng 2–4 giờ, phụ thuộc tốc độ tải image.
- Phần thực hành B: khoảng 2–3 giờ.
- Nếu khôi phục snapshot `01-cluster-ready`: khoảng 2–3 giờ tổng cộng.

---

## 2. Quy ước và an toàn

- Chỉ chạy fault injection trên `k8s-worker2`.
- Không dừng `kube-apiserver`, `etcd` hoặc sửa manifest control plane trong lab 1a.
- Mở ít nhất ba terminal SSH: master, worker 1 và worker 2.
- Các lệnh không ghi rõ node được chạy trên `k8s-master` bằng user quản trị có kubeconfig.
- Dòng bắt đầu bằng `PASS:` mô tả điều kiện phải đạt; không tiếp tục nếu gate tương ứng fail.
- Thay dải `192.168.100.0/24` nếu trùng LAN, VPN, Pod CIDR hoặc Service CIDR của máy host.

---

# Phần A — Dựng cluster lab độc lập

## A1. Topology và phần cứng

### A1.1. Máy host Windows

Khuyến nghị:

| Tài nguyên host | Tối thiểu thực dụng | Khuyến nghị |
| --- | --- | --- |
| CPU vật lý | 6 core / 12 thread, VT-x hoặc AMD-V | 8 core / 16 thread trở lên |
| RAM | 24 GB | 32 GB trở lên |
| SSD trống | 150 GB | 200 GB để còn chỗ cho snapshot |
| Hypervisor | VMware Workstation có hỗ trợ Ubuntu 24.04 x64 | VMware Workstation 17.6 trở lên |

Ba VM dưới đây cấp tổng cộng 8 vCPU và 20 GB RAM. Nếu host chỉ có 24 GB RAM, phải đóng
ứng dụng nặng khi chạy lab. Không chạy cả ba VM trên host 16 GB.

Kiểm tra trên PowerShell:

```powershell
Get-CimInstance Win32_Processor |
  Select-Object Name, NumberOfCores, NumberOfLogicalProcessors, VirtualizationFirmwareEnabled

Get-CimInstance Win32_ComputerSystem |
  Select-Object @{N='RAM_GB';E={[math]::Round($_.TotalPhysicalMemory/1GB,1)}}

Get-Volume |
  Where-Object DriveLetter |
  Select-Object DriveLetter, @{N='Free_GB';E={[math]::Round($_.SizeRemaining/1GB,1)}}
```

**PASS:** virtualization là `True`, host có ít nhất 24 GB RAM và ổ chứa VM còn ít nhất
150 GB.

### A1.2. Ba VM

| Vai trò | Hostname | IP ví dụ | vCPU | RAM | Disk thin-provisioned |
| --- | --- | --- | --- | --- | --- |
| Control plane | `k8s-master` | `192.168.100.111` | 4 | 8 GB | 40 GB |
| Worker 1 | `k8s-worker1` | `192.168.100.112` | 2 | 6 GB | 40 GB |
| Worker 2 / fault target | `k8s-worker2` | `192.168.100.113` | 2 | 6 GB | 40 GB |

Thiết lập mỗi VM:

1. Guest OS: Ubuntu 64-bit.
2. Firmware: UEFI; Secure Boot có thể giữ bật nếu hypervisor hỗ trợ bình thường.
3. Network: Bridged để Windows host SSH trực tiếp được tới VM.
4. Disk: SCSI/NVMe, thin provision, 40 GB.
5. Không clone một VM đã boot nếu chưa tạo lại `machine-id`, MAC và product UUID. Cách ít
   lỗi nhất cho lab đầu tiên là cài riêng ba VM.

Nếu LAN không cho dùng Bridged, dùng một VMnet NAT riêng. Điều kiện bắt buộc là ba VM liên
lạc hai chiều với nhau, Windows host SSH được tới cả ba VM và cả ba VM có Internet egress.

### A1.3. Phiên bản được khóa

| Thành phần | Baseline chính xác |
| --- | --- |
| Ubuntu ISO | `ubuntu-24.04.4-live-server-amd64.iso` |
| Kubernetes control plane | `v1.35.6` |
| `kubeadm`, `kubelet`, `kubectl` | Debian package `1.35.6-1.1` |
| `cri-tools` / `crictl` | Debian package `1.35.0-1.1` |
| `kubernetes-cni` package | `1.8.0-1.1` |
| `containerd` từ Ubuntu Noble | `2.2.1-0ubuntu1~24.04.3` |
| `runc` từ Ubuntu Noble | `1.3.3-0ubuntu1~24.04.3` |
| CNI | Flannel manifest release `v0.28.7` |
| Pod CIDR | `10.244.0.0/16` |
| Service CIDR | kubeadm mặc định `10.96.0.0/12` |

Hậu tố `~24.04.3` của package `containerd`/`runc` là revision package hiện có trong kho
Noble Updates/Security và vẫn là package chính thức cho Ubuntu 24.04.4. Runbook kiểm tra
candidate trước khi cài; nếu kho không còn đúng revision này thì **dừng**, cập nhật baseline
runbook sau khi đối chiếu changelog và tài liệu tương thích, không âm thầm cài version khác.

Các gói nền `ca-certificates`, `curl`, `gpg`, `apt-transport-https` và bản vá OS lấy từ
Noble Updates/Security. Version của chúng biến đổi theo bản vá bảo mật; A5 sẽ ghi version
thực tế vào hồ sơ lab.

Sau khi tải ISO từ trang Ubuntu chính thức, kiểm tra trên PowerShell trước khi gắn ISO vào VM:

```powershell
$isoPath = 'D:\ISO\ubuntu-24.04.4-live-server-amd64.iso'
$expectedSha256 = 'e907d92eeec9df64163a7e454cbc8d7755e8ddc7ed42f99dbc80c40f1a138433'
$actualSha256 = (Get-FileHash -Algorithm SHA256 -LiteralPath $isoPath).Hash.ToLowerInvariant()
if ($actualSha256 -ne $expectedSha256) {
  throw "FAIL: ISO SHA256 mismatch: $actualSha256"
}
'PASS: Ubuntu ISO SHA256 is valid'
```

Không tiếp tục nếu checksum khác giá trị được Canonical công bố trong `SHA256SUMS`.

## A2. Cài Ubuntu và cấu hình identity

Cài Ubuntu Server 24.04.4 trên cả ba VM:

- Minimal installation, không cài GUI.
- Bật OpenSSH Server.
- Tạo cùng một user quản trị, ví dụ `k8sadmin`, có quyền `sudo`.
- Đặt hostname đúng theo bảng A1.2 ngay trong installer; nếu chưa đúng, chạy lệnh tương ứng:

```bash
# Chỉ chạy đúng một lệnh phù hợp trên từng VM
sudo hostnamectl set-hostname k8s-master
sudo hostnamectl set-hostname k8s-worker1
sudo hostnamectl set-hostname k8s-worker2
```

Sau reboot, chạy trên từng VM:

```bash
hostnamectl --static
cat /etc/machine-id
sudo cat /sys/class/dmi/id/product_uuid
ip -br link
```

**PASS:** hostname đúng; `machine-id`, product UUID và MAC khác nhau giữa ba VM.

## A3. Đặt IP tĩnh và phân giải tên

Trước khi gán IP, kiểm tra `.111–.113` nằm ngoài DHCP pool của router. Nếu không chắc, tạo
DHCP reservation theo MAC hoặc chọn ba IP khác.

Trên mỗi VM, tìm interface đang giữ default route:

```bash
ip route
ip route | awk '/default/ {print $5; exit}'
```

Ví dụ interface là `ens33`. Tạo netplan riêng trên từng VM; thay gateway/DNS cho đúng LAN.
Ví dụ cho master:

```bash
sudo tee /etc/netplan/99-k8s-lab.yaml >/dev/null <<'EOF'
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses: [192.168.100.111/24]
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses: [192.168.100.1, 1.1.1.1]
EOF
sudo chmod 600 /etc/netplan/99-k8s-lab.yaml
sudo netplan try
```

Với worker 1 và worker 2, giữ nguyên cấu trúc nhưng đổi `addresses` lần lượt thành
`192.168.100.112/24` và `192.168.100.113/24`. Nếu interface không phải `ens33`, thay đúng
tên đã tìm ở trên. Xác nhận cấu hình trong thời gian `netplan try` yêu cầu.

Thêm trên **cả ba VM**:

```bash
sudo tee -a /etc/hosts >/dev/null <<'EOF'
192.168.100.111 k8s-master
192.168.100.112 k8s-worker1
192.168.100.113 k8s-worker2
EOF
```

Verify trên cả ba VM:

```bash
ip -br address
ip route
getent hosts k8s-master k8s-worker1 k8s-worker2
ping -c 2 k8s-master
ping -c 2 k8s-worker1
ping -c 2 k8s-worker2
curl -I --max-time 10 https://pkgs.k8s.io/
```

**PASS:** tên trả đúng IP; ping giữa mọi node thành công; HTTPS egress hoạt động.

## A4. Chuẩn bị OS và container runtime

Chạy toàn bộ mục A4 trên **cả ba VM**.

### A4.1. Cập nhật OS, tắt swap và bật kernel prerequisites

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo swapoff -a
sudo sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab

cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
sudo reboot
```

Chờ VM boot lại, SSH vào đúng node rồi verify:

```bash
swapon --show
free -h | grep -i swap
lsmod | grep -E '^(overlay|br_netfilter)'
sysctl net.bridge.bridge-nf-call-iptables \
  net.bridge.bridge-nf-call-ip6tables \
  net.ipv4.ip_forward
stat -fc %T /sys/fs/cgroup
```

**PASS:** `swapon --show` rỗng, Swap là `0B`, hai module có mặt, ba sysctl bằng `1`, cgroup
filesystem là `cgroup2fs`.

### A4.2. Cài containerd và runc đúng version

```bash
CONTAINERD_VER='2.2.1-0ubuntu1~24.04.3'
RUNC_VER='1.3.3-0ubuntu1~24.04.3'

apt-cache madison containerd | grep -F "$CONTAINERD_VER"
apt-cache madison runc | grep -F "$RUNC_VER"

sudo apt-get install -y \
  containerd="$CONTAINERD_VER" \
  runc="$RUNC_VER"

sudo apt-mark hold containerd runc
```

Nếu một trong hai lệnh `grep` không có output, dừng tại đây. Không bỏ version khỏi lệnh cài.

Cấu hình CRI plugin và systemd cgroup cho containerd 2.x:

```bash
sudo mkdir -p /etc/containerd
sudo tee /etc/containerd/config.toml >/dev/null <<'EOF'
version = 3

[plugins.'io.containerd.cri.v1.runtime'.containerd]
  default_runtime_name = 'runc'

[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc]
  runtime_type = 'io.containerd.runc.v2'

[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
  SystemdCgroup = true
EOF

sudo systemctl enable --now containerd
sudo systemctl restart containerd
```

Verify:

```bash
containerd --version
runc --version | head -n 1
dpkg-query -W -f='${Package} ${Version}\n' containerd runc
systemctl is-active containerd
sudo ctr plugins ls | grep 'io.containerd.cri.v1'
grep -n 'SystemdCgroup = true' /etc/containerd/config.toml
```

**PASS:** package version khớp bảng A1.3; containerd `active`; CRI plugin trạng thái `ok`;
`SystemdCgroup = true`.

### A4.3. Cài kubeadm, kubelet, kubectl và crictl

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor --yes -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

K8S_VER='1.35.6-1.1'
CRI_VER='1.35.0-1.1'
CNI_PKG_VER='1.8.0-1.1'

apt-cache madison kubeadm | grep -F "$K8S_VER"
apt-cache madison cri-tools | grep -F "$CRI_VER"
apt-cache madison kubernetes-cni | grep -F "$CNI_PKG_VER"

sudo apt-get install -y \
  kubernetes-cni="$CNI_PKG_VER" \
  cri-tools="$CRI_VER" \
  kubelet="$K8S_VER" \
  kubeadm="$K8S_VER" \
  kubectl="$K8S_VER"

sudo apt-mark hold kubernetes-cni cri-tools kubelet kubeadm kubectl
sudo systemctl enable kubelet

sudo tee /etc/crictl.yaml >/dev/null <<'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

Verify:

```bash
kubeadm version -o short
kubelet --version
kubectl version --client
crictl --version
dpkg-query -W -f='${Package} ${Version}\n' \
  kubernetes-cni cri-tools kubelet kubeadm kubectl
apt-mark showhold | grep -E \
  '^(containerd|runc|kubernetes-cni|cri-tools|kubelet|kubeadm|kubectl)$'
sudo crictl version
sudo crictl info | grep -i -A2 systemdCgroup
```

**PASS:** ba Kubernetes binary là `v1.35.6`, crictl là `v1.35.0`, package CNI là
`1.8.0-1.1`, CRI API là `v1`, `systemdCgroup` là `true` và toàn bộ package đã được hold.

Kubelet có thể restart liên tục trước `kubeadm init/join`; đây là trạng thái dự kiến vì chưa
có `/var/lib/kubelet/config.yaml`.

### A4.4. Firewall cho mạng lab cô lập

Với mạng homelab tin cậy, tắt UFW trên cả ba VM để bài 1a không bị nhiễu bởi bài firewall:

```bash
sudo ufw status
sudo ufw disable
sudo ufw status
```

**PASS:** trạng thái `inactive`. Không áp dụng lựa chọn này cho cluster production.

## A5. Khởi tạo cluster

### A5.1. Init control plane

Chạy chỉ trên `k8s-master`:

```bash
KUBERNETES_VERSION="$(kubeadm version -o short)"
test "$KUBERNETES_VERSION" = 'v1.35.6'

sudo kubeadm config images list --kubernetes-version "$KUBERNETES_VERSION"
sudo kubeadm config images pull --kubernetes-version "$KUBERNETES_VERSION"

sudo kubeadm init \
  --kubernetes-version "$KUBERNETES_VERSION" \
  --control-plane-endpoint 'k8s-master:6443' \
  --apiserver-advertise-address 192.168.100.111 \
  --pod-network-cidr 10.244.0.0/16
```

Không ngắt lệnh `kubeadm init`. Khi thành công, cấu hình kubeconfig cho user thường:

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
chmod 600 "$HOME/.kube/config"

kubectl version
kubectl get nodes -o wide
```

Master có thể còn `NotReady` trước khi cài CNI.

### A5.2. Cài Flannel v0.28.7

Chạy trên `k8s-master`:

```bash
kubectl apply -f \
  https://github.com/flannel-io/flannel/releases/download/v0.28.7/kube-flannel.yml

kubectl rollout status daemonset/kube-flannel-ds \
  -n kube-flannel --timeout=180s
kubectl get pods -n kube-flannel -o wide
```

**PASS:** DaemonSet rollout thành công; Pod Flannel trên master là `Running`.

### A5.3. Join hai worker

Trên master, sinh lệnh join mới:

```bash
kubeadm token create --print-join-command
```

Copy nguyên lệnh được sinh và chạy bằng `sudo` trên `k8s-worker1`, rồi `k8s-worker2`.
Trước khi join, verify trên từng worker:

```bash
getent hosts k8s-master
timeout 3 bash -c 'echo > /dev/tcp/k8s-master/6443' \
  && echo 'PASS: API 6443 reachable' \
  || echo 'FAIL: API 6443 unreachable'
```

Ví dụ hình dạng lệnh, không copy placeholder dưới đây:

```bash
sudo kubeadm join k8s-master:6443 \
  --token <token-thật> \
  --discovery-token-ca-cert-hash sha256:<hash-thật>
```

### A5.4. Gate `01-cluster-ready`

Chạy trên master:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
kubectl get nodes -o wide
kubectl get pods -A -o wide

kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\t"}{.status.nodeInfo.containerRuntimeVersion}{"\n"}{end}'

kubectl -n kube-system get deployment coredns
kubectl -n kube-system get daemonset kube-proxy
kubectl -n kube-flannel get daemonset kube-flannel-ds
```

**PASS:**

- Cả ba node `Ready` và cùng kubelet `v1.35.6`.
- Control plane có role `control-plane`; hai worker không mang role này.
- Tất cả Pod hệ thống `Running`; CoreDNS có đủ replica `READY`.
- `kube-proxy` và Flannel có một Pod sẵn sàng trên mỗi node.

Ghi baseline package trên từng node để có thể tái lập:

```bash
mkdir -p ~/lab-1a-evidence
{
  date -Is
  hostnamectl
  uname -a
  lsb_release -a
  dpkg-query -W -f='${Package} ${Version}\n' \
    apt-transport-https ca-certificates curl gpg \
    containerd runc kubernetes-cni cri-tools kubelet kubeadm kubectl
} | tee ~/lab-1a-evidence/00-package-baseline.txt
```

Tắt ba VM sạch sẽ và tạo snapshot VMware tên **`01-cluster-ready`** cho từng VM. Bật lại
master, worker 1, worker 2 và chạy lại gate A5.4 trước khi sang phần B.

---

# Phần B — Thực hành kiến thức 1a

## B0. Chuẩn bị terminal và thư mục bằng chứng

Trên master:

```bash
mkdir -p ~/lab-1a-evidence
kubectl config current-context
kubectl cluster-info
kubectl get nodes
```

**PASS:** context trỏ cluster lab, API server phản hồi, cả ba node `Ready`.

## B1. Từ nhu cầu đến khả năng Kubernetes

Chạy:

```bash
kubectl get nodes -o wide | tee ~/lab-1a-evidence/01-nodes.txt
kubectl get pods -A -o wide | tee ~/lab-1a-evidence/01-system-pods.txt
```

Từ output, ghi vào `~/lab-1a-evidence/01-overview.md` câu trả lời cho bốn câu hỏi:

1. Kubernetes đang quản lý những máy nào?
2. Những component nào Kubernetes giữ chạy mà không cần bạn khởi động thủ công từng
   container?
3. Kubernetes tự động hóa deployment/scaling/recovery/configuration ở mức orchestration;
   việc nào vẫn thuộc về người quản trị, ví dụ chọn ứng dụng, policy, backup và capacity?
4. Vì sao ba VM không tự trở thành cluster chỉ vì đã cài container runtime?

**PASS:** câu trả lời phân biệt được container runtime với orchestrator và không tuyên bố
Kubernetes tự xây ứng dụng, tự chọn policy kinh doanh hoặc tự bảo đảm backup.

## B2. Kiểm kê component

Trên master:

```bash
kubectl -n kube-system get pods -o wide \
  | tee ~/lab-1a-evidence/02-kube-system.txt
kubectl -n kube-flannel get pods -o wide \
  | tee ~/lab-1a-evidence/02-flannel.txt
sudo crictl ps \
  | tee ~/lab-1a-evidence/02-master-cri.txt
sudo ls -la /etc/kubernetes/manifests \
  | tee ~/lab-1a-evidence/02-static-manifests.txt
systemctl is-active kubelet containerd
```

Trên worker 1:

```bash
sudo crictl ps
sudo find /etc/kubernetes/manifests -maxdepth 1 -type f -print
systemctl is-active kubelet containerd
```

Điền bảng sau vào ghi chú:

| Component | Nơi quan sát được | Vai trò tối thiểu phải giải thích |
| --- | --- | --- |
| kube-apiserver | Static Pod trên master | Cổng vào duy nhất của Kubernetes API |
| etcd | Static Pod trên master | Lưu state bền vững của API |
| kube-scheduler | Static Pod trên master | Chọn node cho Pod chưa được gán node |
| kube-controller-manager | Static Pod trên master | Chạy nhiều control loop |
| kubelet | systemd service trên mọi node | Agent của node; đăng ký và báo trạng thái |
| containerd | systemd service trên mọi node | Thực thi container qua CRI |
| kube-proxy | DaemonSet/Pod trên mọi node | Cài đặt cơ chế chuyển tiếp Service |
| Flannel | DaemonSet/Pod trên mọi node | CNI/pod network của lab |
| CoreDNS | Deployment/Pod trong cluster | DNS add-on của cluster |

**PASS:** không gọi kubelet/containerd là control plane; không gọi CoreDNS/CNI là thành phần
bắt buộc của chính control plane; biết cloud-controller-manager không xuất hiện vì lab không
tích hợp cloud provider.

## B3. Ghép thành kiến trúc cluster

Trên master:

```bash
sudo grep -n -- '--etcd-servers' /etc/kubernetes/manifests/kube-apiserver.yaml
sudo grep -n -- '--kubeconfig' /etc/kubernetes/manifests/kube-scheduler.yaml
sudo grep -n -- '--kubeconfig' /etc/kubernetes/manifests/kube-controller-manager.yaml
sudo crictl pods --name kube
```

Chứng minh chỉ API server được cấu hình kết nối etcd trực tiếp:

```bash
sudo grep -R -n -- '--etcd-servers' /etc/kubernetes/manifests
```

**PASS:** chỉ manifest `kube-apiserver.yaml` chứa `--etcd-servers`; scheduler và controller
manager dùng kubeconfig để nói chuyện với API server.

Vẽ lại sơ đồ bằng tay hoặc Markdown và lưu thành `03-architecture.md`:

```text
kubectl/admin
     |
     v
kube-apiserver <----> etcd
     ^   ^
     |   +---- scheduler + controller-manager
     |
     +-------- kubelet trên worker 1/2
                   |
                   +---- containerd ---- container
```

Bổ sung Flannel, kube-proxy và CoreDNS vào đúng vị trí. **PASS:** sơ đồ không vẽ kubectl,
kubelet, scheduler hoặc controller-manager kết nối trực tiếp tới etcd.

## B4. Object, desired state và observed state

Lab dùng Namespace chỉ như vùng cô lập; chi tiết namespace sẽ học ở 1b.

```bash
kubectl create namespace lab-1a
kubectl get namespace lab-1a -o yaml \
  | tee ~/lab-1a-evidence/04-namespace.yaml

kubectl get node k8s-worker1 -o jsonpath='{"metadata.name: "}{.metadata.name}{"\n"}{"spec.podCIDR: "}{.spec.podCIDR}{"\n"}{"status.capacity.cpu: "}{.status.capacity.cpu}{"\n"}{"status.allocatable.cpu: "}{.status.allocatable.cpu}{"\n"}{"status.nodeInfo.kubeletVersion: "}{.status.nodeInfo.kubeletVersion}{"\n"}'

kubectl get node k8s-worker1 -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\n"}{end}'
```

Trả lời:

- `apiVersion`, `kind`, `metadata` giúp API server nhận diện object ra sao?
- Trường nào ở Node do cấu hình/cluster gán trong `spec`?
- Trường nào do kubelet và control plane quan sát rồi công bố trong `status`?
- Tại sao client không nên tự ghi tùy ý vào `status`?

**PASS:** chỉ đúng `spec.podCIDR` thuộc spec; `capacity`, `allocatable`, `conditions` và
`nodeInfo` thuộc status. Hiểu rằng không phải mọi object đều có nhiều field trong `spec`.

## B5. Kubernetes API: discovery, group, version và validation

### B5.1. API server là cổng vào

```bash
kubectl get --raw /version | tee ~/lab-1a-evidence/05-api-version.json
kubectl get --raw /api | tee ~/lab-1a-evidence/05-core-api.json
kubectl get --raw /apis | head -c 1000
echo
```

`/api/v1` là core group; `/apis/<group>/<version>` là named API group.

### B5.2. Discovery và maturity level

```bash
kubectl api-versions | tee ~/lab-1a-evidence/05-api-versions.txt
kubectl api-resources --api-group='' | head -n 20
kubectl api-resources --api-group=apps
kubectl get --raw /apis/apps/v1 | head -c 1000
echo
kubectl explain node.spec
kubectl explain node.status.conditions
```

Trong output `api-versions`, tìm ví dụ:

```bash
grep -E '(^v1$|/v1$|v1beta[0-9]+$|v1alpha[0-9]+$)' \
  ~/lab-1a-evidence/05-api-versions.txt
```

Giải thích quy ước: `v1alphaN` có thể thay đổi mạnh, `v1betaN` đã chín hơn nhưng vẫn có
thể đổi, còn `v1` là stable/GA. Version là theo từng API group, không phải toàn cluster có
một maturity level duy nhất.

### B5.3. Server-side field validation

Lệnh sau **phải lỗi** và không tạo object:

```bash
kubectl apply --dry-run=server --validate=strict -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: lab-1a-invalid
spec:
  fieldThatDoesNotExist: true
EOF
```

Verify:

```bash
if kubectl get namespace lab-1a-invalid >/dev/null 2>&1; then
  echo 'FAIL: invalid object was created'
else
  echo 'PASS: invalid object does not exist'
fi

kubectl get apiservices | head -n 20
```

**PASS:** strict validation báo unknown field; namespace không tồn tại. Biết APIService là
điểm quan sát aggregation layer, nhưng lab chưa cài extension API server nên chỉ cần nhận
diện cơ chế này.

## B6. Node registration, status, condition và heartbeat

### B6.1. Quan sát Node object

```bash
kubectl describe node k8s-worker2 \
  | tee ~/lab-1a-evidence/06-worker2-describe.txt

kubectl get node k8s-worker2 -o jsonpath='{"UID: "}{.metadata.uid}{"\n"}{"PodCIDR: "}{.spec.podCIDR}{"\n"}{"InternalIP: "}{.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{"Capacity CPU: "}{.status.capacity.cpu}{"\n"}{"Allocatable CPU: "}{.status.allocatable.cpu}{"\n"}{"Kubelet: "}{.status.nodeInfo.kubeletVersion}{"\n"}'

kubectl get node k8s-worker2 -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.lastHeartbeatTime}{"\t"}{.reason}{"\n"}{end}'

kubectl -n kube-node-lease get lease k8s-worker2 -o yaml \
  | tee ~/lab-1a-evidence/06-worker2-lease-before.yaml
```

Node registration do kubelet thực hiện khi join; API server lưu Node object. Heartbeat nhanh
được cập nhật qua Lease, còn Node status mang Conditions và thông tin tài nguyên.

### B6.2. Fault injection: dừng kubelet trên worker 2

Terminal 1 trên master:

```bash
kubectl -n kube-node-lease get lease k8s-worker2 -w
```

Terminal 2 trên worker 2:

```bash
sudo systemctl stop kubelet
systemctl is-active kubelet
```

Terminal 3 trên master:

```bash
for i in {1..18}; do
  READY="$(kubectl get node k8s-worker2 -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
  printf '%s Ready=%s\n' "$(date -Is)" "$READY"
  if [ "$READY" != 'True' ]; then
    echo 'PASS: control plane detected missing heartbeat'
    break
  fi
  sleep 10
done

test "$READY" != 'True'
kubectl describe node k8s-worker2 | sed -n '/Conditions:/,/Addresses:/p'
```

**PASS:** `renewTime` của Lease ngừng thay đổi; sau khoảng một phút Node controller đổi
Ready khỏi `True` (thường là `Unknown`, reason `NodeStatusUnknown`). Không dùng đúng số giây
như một cam kết SLA; giá trị phụ thuộc cấu hình controller.

### B6.3. Phục hồi

Trên worker 2:

```bash
sudo systemctl start kubelet
systemctl is-active kubelet
journalctl -u kubelet --since '-5 minutes' --no-pager | tail -n 50
```

Trên master:

```bash
kubectl wait --for=condition=Ready node/k8s-worker2 --timeout=120s
kubectl get node k8s-worker2
kubectl -n kube-node-lease get lease k8s-worker2 -o jsonpath='{.spec.renewTime}{"\n"}'
```

**PASS:** kubelet `active`, worker 2 trở lại `Ready`, `renewTime` tiếp tục tăng.

## B7. Giao tiếp giữa Node và Control Plane

### B7.1. kubectl → API server

```bash
kubectl -v=8 get --raw /version 2>&1 \
  | tee ~/lab-1a-evidence/07-kubectl-to-apiserver.txt
```

**PASS:** log verbose cho thấy request HTTPS tới endpoint port `6443`; response là version
API server. `kubectl` không nói trực tiếp với etcd, scheduler hay kubelet cho thao tác này.

### B7.2. kubelet → API server

Trên worker 1:

```bash
sudo grep -n 'server:' /etc/kubernetes/kubelet.conf
sudo journalctl -u kubelet --since '-15 minutes' --no-pager \
  | grep -E 'node status|lease|registered|apiserver' \
  | tail -n 30 || true
sudo ss -tnp | grep ':6443' || true
```

Trên master, xác nhận Lease của worker 1 đổi theo thời gian:

```bash
kubectl -n kube-node-lease get lease k8s-worker1 \
  -o jsonpath='{.spec.renewTime}{"\n"}'
sleep 12
kubectl -n kube-node-lease get lease k8s-worker1 \
  -o jsonpath='{.spec.renewTime}{"\n"}'
```

**PASS:** kubelet.conf trỏ endpoint API server; `renewTime` lần hai mới hơn lần một. Đây là
chiều node chủ động kết nối tới control plane.

### B7.3. API server → kubelet

Từ master, yêu cầu API server proxy tới health endpoint của kubelet trên worker 1:

```bash
kubectl get --raw '/api/v1/nodes/k8s-worker1/proxy/healthz'
```

**PASS:** output là `ok`. Request đi `kubectl → API server → kubelet`; client không mở
kết nối trực tiếp đến port kubelet. Cùng hướng giao tiếp này được dùng cho các thao tác như
logs, attach, exec và port-forward, sẽ thực hành sâu hơn sau khi học Pod.

### B7.4. Kết luận đường đi

Ghi ba đường đi vào `07-communication.md`:

```text
kubectl ------HTTPS:6443------> kube-apiserver
kubelet ------HTTPS:6443------> kube-apiserver
kube-apiserver --HTTPS:10250--> kubelet
```

Thêm ghi chú: trong control plane mặc định, API server là thành phần nói trực tiếp với etcd;
node và Pod không kết nối trực tiếp tới etcd. Lab không triển khai Konnectivity service nên
chỉ cần biết đó là một lựa chọn tunnel/proxy cho đường API server tới node.

## B8. Controller và reconciliation

Để không đưa Deployment/ReplicaSet của giai đoạn sau vào quá sớm, bài này dùng controller
có sẵn trong `kube-controller-manager`: ServiceAccount controller bảo đảm mỗi namespace
active có một ServiceAccount tên `default`.

### B8.1. Quan sát trạng thái ban đầu

```bash
kubectl get serviceaccount default -n lab-1a -o yaml \
  | tee ~/lab-1a-evidence/08-default-sa-before.yaml

OLD_UID="$(kubectl get serviceaccount default -n lab-1a \
  -o jsonpath='{.metadata.uid}')"
echo "old UID: $OLD_UID"
test -n "$OLD_UID"
```

### B8.2. Tạo sai lệch có chủ đích

```bash
kubectl delete serviceaccount default -n lab-1a

NEW_UID=''
for i in {1..30}; do
  NEW_UID="$(kubectl get serviceaccount default -n lab-1a \
    -o jsonpath='{.metadata.uid}' 2>/dev/null || true)"
  if [ -n "$NEW_UID" ] && [ "$NEW_UID" != "$OLD_UID" ]; then
    break
  fi
  sleep 1
done

echo "old UID: $OLD_UID"
echo "new UID: $NEW_UID"
test -n "$NEW_UID"
test "$NEW_UID" != "$OLD_UID"
```

**PASS:** object `default` xuất hiện lại với UID mới mà người học không chạy lệnh create.

Ánh xạ quan sát vào control loop:

| Bước | Điều vừa xảy ra |
| --- | --- |
| Watch/observe | Controller xem object trong API server |
| Compare | Quy tắc mong muốn có `default`; trạng thái thực tế vừa bị xóa |
| Act | Controller gửi create request tới API server |
| Repeat | Object mới có UID khác; vòng lặp tiếp tục quan sát |

Điểm phải hiểu: controller không phải một script chỉ chạy một lần; nó liên tục làm cho state
hội tụ. Controller thường thao tác qua API server. Một số controller tích hợp hạ tầng ngoài
cluster có thể gọi cloud/provider API, nhưng vẫn dùng Kubernetes API để quan sát/cập nhật
object liên quan.

## B9. Cleanup và gate cuối

Xóa duy nhất namespace của lab:

```bash
kubectl delete namespace lab-1a --wait=true --timeout=120s
kubectl get namespace lab-1a 2>&1 || true
kubectl wait --for=condition=Ready node --all --timeout=120s
kubectl get nodes
kubectl get pods -A
```

**PASS:** namespace `lab-1a` không còn; ba node `Ready`; Pod hệ thống trở lại `Running`.

---

## 3. Checkpoint 1a

Không đánh dấu lab hoàn tất chỉ vì đã chạy hết lệnh. Tự trả lời không nhìn tài liệu:

- [ ] Vẽ đúng control plane, worker và đường giao tiếp giữa các component.
- [ ] Chỉ ra component duy nhất ghi/đọc etcd trực tiếp trong kiến trúc mặc định.
- [ ] Phân biệt được static Pod control plane với systemd service kubelet/containerd.
- [ ] Mở một Node YAML và chỉ đúng ví dụ của `metadata`, `spec`, `status`.
- [ ] Giải thích core group khác named API group và đọc được `group/version/resource`.
- [ ] Giải thích alpha, beta, stable là maturity của API version, không phải trạng thái của
  toàn cluster.
- [ ] Chỉ ra `Capacity`, `Allocatable`, `Conditions`, `InternalIP`, kubelet version và Lease
  của một Node.
- [ ] Mô tả điều xảy ra từ lúc kubelet ngừng heartbeat tới khi Node không còn `Ready=True`.
- [ ] Phân biệt ba chiều `kubectl→API server`, `kubelet→API server`,
  `API server→kubelet`.
- [ ] Kể lại thí nghiệm ServiceAccount bằng bốn bước observe–compare–act–repeat.
- [ ] Cluster sạch và cả ba node `Ready` sau cleanup.

### Bài giải thích cuối cùng

Trong tối đa 10 phút, nói lại đường đi khái niệm sau:

1. Người dùng gửi một object tới API server.
2. API server xác thực/validate rồi lưu state qua etcd.
3. Controller và scheduler quan sát API, không đọc etcd trực tiếp.
4. Kubelet trên node quan sát assignment qua API server và nhờ runtime thực thi.
5. Kubelet cập nhật status/heartbeat; controller tiếp tục so trạng thái thực tế với mong muốn.

Nếu không giải thích được một mũi tên, quay lại đúng phần B tương ứng. Khi toàn bộ checkbox
được đánh dấu và phần giải thích không còn lẫn vai trò component, lab 1a hoàn tất.

---

## 4. Troubleshooting nhanh

| Triệu chứng | Kiểm tra | Cách xử lý trong lab |
| --- | --- | --- |
| VM không ping nhau | VMware network mode, IP, route | Đưa cả ba VM vào cùng Bridged/VMnet; sửa IP trước kubeadm |
| `kubeadm init` báo swap | `swapon --show` | `swapoff -a`, comment swap trong `/etc/fstab` |
| CRI không hoạt động | `crictl info`, `ctr plugins ls` | Kiểm tra config version 3 và restart containerd |
| Node `NotReady` sau join | Flannel Pod, kubelet log | Kiểm tra Pod CIDR, egress, module và sysctl |
| Worker không join | DNS và TCP 6443 | Sửa `/etc/hosts`, IP/firewall; sinh token join mới |
| API proxy kubelet lỗi | kubelet status, 10250, Node Ready | Khởi động kubelet; không bỏ qua lỗi TLS/auth bằng `curl -k` |
| Worker 2 không phục hồi | `journalctl -u kubelet` | Start kubelet; nếu cần khôi phục snapshot cả ba VM cùng mốc |
| ServiceAccount không tạo lại | controller-manager Pod/log | Xác nhận controller-manager Running và namespace còn Active |

Không reset hoặc restore riêng master bằng snapshot cũ trong khi giữ worker ở state mới. Với
lab một control plane, nếu cần quay lại mốc `01-cluster-ready`, tắt và restore **cả ba VM**.

---

## 5. Nguồn chính thức

- [Kubernetes — Cluster Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes — Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Kubernetes — Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/)
- [Kubernetes — Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)
- [Kubernetes — Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Kubernetes — Communication between Nodes and the Control Plane](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/)
- [Kubernetes — Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Kubernetes — Installing kubeadm v1.35](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Kubernetes — Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Kubernetes — Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [Kubernetes — ServiceAccount controller](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#serviceaccount-controller)
- [Ubuntu — 24.04.4 LTS images](https://releases.ubuntu.com/24.04/)
- [Ubuntu — containerd package for Noble](https://packages.ubuntu.com/noble/containerd)
- [Flannel — release v0.28.7](https://github.com/flannel-io/flannel/releases/tag/v0.28.7)
