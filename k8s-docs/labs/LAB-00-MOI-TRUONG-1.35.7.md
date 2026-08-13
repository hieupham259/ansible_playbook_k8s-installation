# Lab 00 (biến thể 1.35.7) — Dựng môi trường lab dùng chung

> **Baseline:** Ubuntu Server 24.04.4 LTS, Kubernetes v1.35.7, containerd 2.2.1, runc 1.3.4,
> Flannel v0.28.9, một control plane và hai worker chạy bằng VM trên Windows.
>
> **Cập nhật và đối chiếu phiên bản:** 13/08/2026.

File này là **biến thể** của [Lab 00](LAB-00-MOI-TRUONG.md), khóa vào bộ version đã tra bằng
[runbook baseline](../../kubeadm-rancher-find-version/runbook-tra-cuu-baseline-phien-ban-k8s.md).
Nội dung, thứ tự bước và tên gate giống hệt bản gốc; chỉ số phiên bản và các kiểm tra phụ
thuộc phiên bản là khác.

## File này là biến thể, không thay thế Lab 00

Hai file cùng tồn tại và **cùng tạo snapshot `01-cluster-ready`**. Chọn đúng một trong hai
trước khi dựng cluster, rồi giữ nguyên lựa chọn đó cho toàn bộ chuỗi lab:

| | [`LAB-00-MOI-TRUONG.md`](LAB-00-MOI-TRUONG.md) | File này |
| --- | --- | --- |
| Kubernetes | `v1.35.6` | `v1.35.7` |
| Flannel | `v0.28.7` | `v0.28.9` |
| `runc` | `1.3.3-0ubuntu1~24.04.3` | `1.3.4-0ubuntu1~24.04.1` |
| Ngày đối chiếu | 05/08/2026 | 13/08/2026 |

Snapshot giữ **nguyên tên `01-cluster-ready`** vì mọi lab sau đều lấy tên đó làm điểm bắt đầu;
đổi tên ở đây sẽ làm hỏng gate đầu vào của toàn bộ chuỗi. Đổi lại, không được trộn hai bản:
đã dựng cluster bằng file này thì mọi lần restore, mọi lần đối chiếu version đều tra ở bảng
[A1.3](#a13-phiên-bản-được-khóa) của **file này**, không tra bảng của bản gốc.

Quy tắc bảo trì giữ nguyên: bảng A1.3 là nơi duy nhất ghi số phiên bản trong nhánh này. Lab
khác cần nói tới phiên bản thì link về đây, không chép lại con số.

## Lab này là ngoại lệ có chủ đích

Lab 00 **không phải bài học** và là lab duy nhất được phép đi trước kiến thức của lộ trình.
Chạy nó ở chế độ copy-paste: không cần hiểu vì sao phải `swapoff`, vì sao
`SystemdCgroup = true`, hay `kubeadm init` làm những gì. File này không có checkpoint vấn đáp,
chỉ có gate kỹ thuật.

Lý do: lộ trình yêu cầu cluster thật để làm checkpoint ngay từ giai đoạn 1, nhưng kubeadm lại
là nội dung của giai đoạn 8. Lộ trình giải vòng lặp này bằng hai phương án ở mục
[Môi trường lab](../LO-TRINH-ADMIN.md#môi-trường-lab): mượn cluster có sẵn, hoặc tự dựng. Lab
00 là phương án tự dựng ở dạng copy-paste — về mặt kiến thức nó tương đương với việc có người
đưa cho bạn một cluster.

Bạn sẽ quay lại đúng nội dung file này ba lần, mỗi lần hiểu thêm một tầng:

| Giai đoạn | Bài | Hiểu ra phần nào của Lab 00 |
| --- | --- | --- |
| 2 | [00 — Các container runtime](../00-container-runtimes-vi.md), [44 — CRI](../44-cri-vi.md), [33 — cgroup v2](../33-cgroups-vi.md) | vì sao cấu hình containerd như A4.2; cgroup driver của kubelet và runtime phải khớp |
| 5 | [183 — Network Plugin](../183-network-plugins-vi.md) | Flannel làm gì; vì sao `kubeadm init` cần `--pod-network-cidr` |
| 8 | [01 — Cài đặt kubeadm](../01-install-kubeadm-vi.md), [02 — Tạo cluster với kubeadm](../02-create-cluster-kubeadm-vi.md) | toàn bộ A4 và A5 — lúc đó Lab 8a phá chính ba VM này và dựng lại có hiểu |

Không suy ra từ đây rằng lab khác cũng được đi trước kiến thức. Mọi lab còn lại tuân thủ
[nguyên tắc không nhảy cóc](README.md#2-nguyên-tắc-không-nhảy-cóc).

---

## 1. Kết quả phải đạt

- Ba VM chạy Ubuntu Server 24.04.4, liên lạc được hai chiều với nhau và với Windows host.
- Một cluster Kubernetes v1.35.7 gồm một control plane và hai worker, tất cả `Ready`.
- Snapshot `01-cluster-ready` trên cả ba VM, khôi phục được bất cứ lúc nào.

Đây **không phải** bài học kiến thức. Không có checkpoint vấn đáp; chỉ có gate kỹ thuật.

### 1.1. Thời lượng

- Dựng VM và cluster lần đầu: khoảng 2–4 giờ, phụ thuộc tốc độ tải image.
- Khôi phục snapshot `01-cluster-ready` và chạy lại gate A5.4: khoảng 10 phút.

---

## 2. Chuỗi snapshot

Mọi lab đều khai báo **điểm bắt đầu** (snapshot nào) và **điểm kết thúc** (trả cluster về
snapshot cũ hay tạo snapshot mới). Chuỗi chính:

| Snapshot | Tạo bởi | Nội dung thêm so với snapshot trước |
| --- | --- | --- |
| `01-cluster-ready` | Lab 00 (file này) | 1 control plane + 2 worker, Flannel, không có workload |
| `02-net-ready` | Lab 5b | CNI hỗ trợ NetworkPolicy thay Flannel, ingress controller |
| `03-storage-ready` | Lab 6a | StorageClass mặc định và provisioner đang chạy |
| `04-metrics-ready` | Lab 11a | metrics-server (điều kiện của lab HPA/VPA) |

Quy tắc:

- Lab **không** tạo snapshot mới thì phải cleanup đưa cluster về đúng trạng thái snapshot
  đầu vào, và gate cuối phải chứng minh điều đó.
- Không restore riêng một VM trong khi giữ hai VM còn lại ở state mới. Với lab một control
  plane, restore là thao tác trên **cả ba VM cùng mốc**.
- Giai đoạn 8 (dựng cluster bằng kubeadm) dùng **bộ VM riêng** với topology khác, snapshot
  đặt tiền tố `8x-`; không đụng vào chuỗi ở trên.

---

## 3. Quy ước và an toàn

- Mở ít nhất ba terminal SSH: master, worker 1 và worker 2.
- Các lệnh không ghi rõ node được chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig.
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

| Vai trò | Tên VM / hostname | IP ví dụ | vCPU | RAM | Disk thin-provisioned |
| --- | --- | --- | --- | --- | --- |
| Control plane | `lab-k8s-master` | `192.168.100.221` | 4 | 8 GB | 40 GB |
| Worker 1 | `lab-k8s-worker1` | `192.168.100.222` | 2 | 6 GB | 40 GB |
| Worker 2 / fault target | `lab-k8s-worker2` | `192.168.100.223` | 2 | 6 GB | 40 GB |

Tiền tố `lab-` dành riêng cho chuỗi lab này, giúp tên VM, hostname, Kubernetes Node và API
endpoint không trùng với cluster được dựng trong runbook VMware.

Thiết lập mỗi VM:

1. Guest OS: Ubuntu 64-bit.
2. Firmware: UEFI; Secure Boot có thể giữ bật nếu hypervisor hỗ trợ bình thường.
3. Network: Bridged để Windows host SSH trực tiếp được tới VM.
4. Disk: SCSI/NVMe, thin provision, 40 GB.
5. Nếu dùng full clone, bắt buộc chuẩn hóa identity theo A2.2 trước khi đặt hostname/IP và
   xác nhận VMware đã sinh MAC cùng product UUID riêng cho từng VM.

Nếu LAN không cho dùng Bridged, dùng một VMnet NAT riêng. Điều kiện bắt buộc là ba VM liên
lạc hai chiều với nhau, Windows host SSH được tới cả ba VM và cả ba VM có Internet egress.

Một số lab sau cần thêm tài nguyên hoặc thêm VM (giai đoạn 8 cần 3 control plane và một load
balancer; lab lưu trữ cần thêm disk hoặc một NFS server). Các lab đó tự khai báo phần bổ
sung; không cấp trước ở đây.

### A1.3. Phiên bản được khóa

Architecture đích của toàn bộ bảng này là **`amd64`**. Package Ubuntu có thể khác version giữa
các architecture trong cùng một suite, nên khi đối chiếu phải đọc đúng dòng `amd64`.

| Thành phần | Baseline chính xác |
| --- | --- |
| Ubuntu ISO | `ubuntu-24.04.4-live-server-amd64.iso` |
| Kubernetes control plane | `v1.35.7` |
| `kubeadm`, `kubelet`, `kubectl` | Debian package `1.35.7-1.1` |
| `cri-tools` / `crictl` | Debian package `1.35.0-1.1` |
| `kubernetes-cni` package | `1.8.0-1.1` |
| `containerd` từ Ubuntu Noble | `2.2.1-0ubuntu1~24.04.3` |
| `runc` từ Ubuntu Noble | `1.3.4-0ubuntu1~24.04.1` |
| CNI | Flannel manifest release `v0.28.9` |
| Image Flannel | `ghcr.io/flannel-io/flannel:v0.28.9` |
| Image Flannel CNI plugin | `ghcr.io/flannel-io/flannel-cni-plugin:v1.9.1-flannel3` |
| Pod CIDR | `10.244.0.0/16` |
| Service CIDR | kubeadm mặc định `10.96.0.0/12` |

Repo `pkgs.k8s.io/core:/stable:/v1.35` chỉ phát hành **đúng một** version cho `cri-tools`
(`1.35.0-1.1`) và `kubernetes-cni` (`1.8.0-1.1`); hai gói này không đi theo patch của
Kubernetes, nên chúng giống hệt bản gốc dù Kubernetes đã lên `1.35.7`.

Hậu tố `~24.04.x` của `containerd` và `runc` là revision package, và **hai gói không dùng
chung revision**: `containerd` đang ở `~24.04.3` còn `runc` ở `~24.04.1`. Đây là giá trị đọc
được từ kho Noble Updates cho `amd64` tại ngày đối chiếu ở đầu file. Runbook kiểm tra
candidate trước khi cài; nếu kho không còn đúng revision này thì **dừng**, cập nhật bảng A1.3
sau khi đối chiếu changelog và tài liệu tương thích, không âm thầm cài version khác.

Các gói nền `ca-certificates`, `curl`, `gpg`, `apt-transport-https` và bản vá OS lấy từ
Noble Updates/Security. Version của chúng biến đổi theo bản vá bảo mật; A5 sẽ ghi version
thực tế vào hồ sơ lab.

Flannel **không thực thi NetworkPolicy**. Đây là lựa chọn có chủ đích: baseline giữ CNI đơn
giản nhất, còn việc đổi sang CNI hỗ trợ policy là một bài học của lab 5b chứ không phải thao
tác chuẩn bị âm thầm.

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

### A1.4. Phần stack không thuộc Lab 00

Bộ baseline gốc còn khóa version cho các thành phần dưới đây. **Lab 00 không cài bất kỳ thành
phần nào trong bảng này** — chúng thuộc giai đoạn sau của lộ trình, và cài trước sẽ vi phạm
[nguyên tắc không nhảy cóc](README.md#2-nguyên-tắc-không-nhảy-cóc). Bảng ở đây chỉ để giữ con
số ở một chỗ, cho lab nào thật sự cần thì link về.

| Thành phần | Version đã chốt | Cài ở đâu |
| --- | --- | --- |
| Helm | `v3.21.3` | lab nào cần chart đầu tiên |
| cert-manager | chart/appVersion `v1.21.1` | ngoài chuỗi lab hiện có |
| Traefik (Ingress) | chart `41.2.0`, appVersion `v3.7.10` | Lab 5b (ingress controller) |
| local-path-provisioner (Storage) | `v0.0.37` | Lab 6a (StorageClass) |
| cloudflared (Tunnel) | image `cloudflare/cloudflared:latest`, digest amd64 `sha256:b392761b…` | ngoài chuỗi lab hiện có |
| Rancher (Cluster management) | chart/appVersion `2.14.3` | ngoài chuỗi lab hiện có |
| MetalLB | `v0.16.0` | ngoài baseline; chỉ cần khi dùng Service `LoadBalancer` thật |

Ràng buộc quyết định trần của bảng A1.3: chart Rancher `2.14.3` khai `kubeVersion: < 1.36.0-0`,
nên Kubernetes không lên được `1.36`. Nếu bạn dựng cluster **không** có ý định cài Rancher thì
ràng buộc này không áp dụng, nhưng vẫn giữ `v1.35.7` để cả hai nhánh dùng chung một baseline.

## A2. Cài Ubuntu và cấu hình identity

Chuẩn bị Ubuntu Server 24.04.4 bằng cách cài riêng từng VM hoặc cài một VM nguồn rồi tạo full
clone. Bản cài dùng cho các VM phải có:

- Minimal installation, không cài GUI.
- Bật OpenSSH Server.
- Tạo cùng user quản trị `ubuntu`, có quyền `sudo`.

### A2.1. VM được cài Ubuntu riêng

Không tạo lại `machine-id` hoặc SSH host key. Chuyển thẳng tới A2.3 để đặt hostname và kiểm
tra identity.

### A2.2. VM được tạo bằng full clone

Khi VMware hỏi VM đã được **move** hay **copy**, chọn **I Copied It** để hypervisor sinh UUID
và MAC mới. Trên **từng VM clone**, mở console VMware và chạy các lệnh sau trước khi cấu hình
hostname/IP. Không chạy qua SSH vì các clone đang có thể trùng SSH host key; không chạy lại
trên VM nguồn:

```bash
# Tạo machine-id riêng.
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo systemd-machine-id-setup

# Tạo SSH host key riêng.
sudo rm -f /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
```

Chưa reboot tại đây. Tiếp tục A2.3 để đặt hostname, hoàn tất cấu hình IP tĩnh ở A3, rồi reboot
một lần. Nếu VM clone đã được chuẩn hóa identity trước đó thì không chạy lại các lệnh trên.

### A2.3. Đặt hostname, `/etc/hosts` và kiểm tra identity

Đặt hostname đúng theo bảng A1.2; chỉ chạy một lệnh phù hợp trên từng VM:

```bash
# Chỉ chạy đúng một lệnh phù hợp trên từng VM
sudo hostnamectl set-hostname lab-k8s-master
sudo hostnamectl set-hostname lab-k8s-worker1
sudo hostnamectl set-hostname lab-k8s-worker2
```

Chạy trên từng VM và đối chiếu kết quả của cả ba máy:

```bash
hostnamectl --static
cat /etc/machine-id
sudo cat /sys/class/dmi/id/product_uuid
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
ip -br link
```

**PASS:** hostname đúng; `machine-id`, product UUID, SSH host-key fingerprint và MAC khác nhau
giữa ba VM. Nếu product UUID hoặc MAC trùng, dừng tại đây và để VMware sinh giá trị mới;
không thể sửa hai giá trị này bằng lệnh bên trong Ubuntu.

Thêm giống nhau trên **cả ba VM**:

```bash
sudo tee -a /etc/hosts >/dev/null <<'EOF'
192.168.100.221 lab-k8s-master
192.168.100.222 lab-k8s-worker1
192.168.100.223 lab-k8s-worker2
EOF
```

## A3. Đặt IP tĩnh và phân giải tên

Trước khi gán IP, kiểm tra `.221–.223` nằm ngoài DHCP pool của router. Nếu không chắc, tạo
DHCP reservation theo MAC hoặc chọn ba IP khác.

Chạy trên từng VM qua console VMware để tránh mất phiên SSH khi đổi IP. Tìm interface đang giữ
default route:

```bash
ip -br a
# PASS: interface ens33/ens160 ở trạng thái UP và có IP DHCP cùng dải LAN.

ip route
# PASS: có "default via <gateway> dev ens33/ens160".

ip route | awk '/default/ {print $5; exit}'
# PASS: chỉ in tên interface, ví dụ ens33; dùng đúng tên này trong 01-static.yaml.
```

Trên **cả ba VM**, chặn cloud-init quản lý mạng trước khi tạo cấu hình IP tĩnh. Nếu bỏ qua,
cloud-init có thể sinh lại cấu hình DHCP và ghi đè IP tĩnh sau reboot:

```bash
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
# bỏ file netplan do cloud-init sinh (đang để DHCP) để tránh xung đột:
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak 2>/dev/null || true
```

Lệnh `mv` không báo lỗi nếu image Ubuntu không có file `50-cloud-init.yaml`; tiếp tục tạo file
Netplan tĩnh riêng như bên dưới.

Ví dụ interface là `ens33`. Tạo file tĩnh `/etc/netplan/01-static.yaml` bằng
`sudo nano /etc/netplan/01-static.yaml`; thay IP và tên card trên từng VM. Nội dung cho master:

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.100.221/24]
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

Áp dụng:

```bash
sudo chmod 600 /etc/netplan/01-static.yaml
sudo netplan apply
```

Với worker 1 và worker 2, giữ nguyên cấu trúc nhưng đổi `addresses` lần lượt thành
`192.168.100.222/24` và `192.168.100.223/24`. Nếu interface không phải `ens33`, thay đúng
tên đã tìm ở trên.

> Nếu đang làm qua SSH, sau `netplan apply` phiên sẽ đứng/rớt vì IP đã đổi; kết nối lại bằng IP
> mới tương ứng. Khi chỉ chỉnh nhỏ, có thể dùng `sudo netplan try` để tự hoàn tác sau 120 giây
> nếu mất mạng.

Sau khi đã áp dụng Netplan trên đủ ba VM, reboot từng VM để chứng minh cloud-init không đưa
interface về DHCP:

```bash
sudo reboot
```

Kết nối lại bằng IP tĩnh tương ứng rồi verify trên cả ba VM:

```bash
grep -Fx 'network: {config: disabled}' \
  /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg

if test -e /etc/netplan/50-cloud-init.yaml; then
  echo 'FAIL: cloud-init regenerated 50-cloud-init.yaml'
else
  echo 'PASS: 50-cloud-init.yaml is absent'
fi

ip -br address                 # card hiển thị đúng IP tĩnh .221/.222/.223
ip route                       # có default route qua gateway LAN

getent hosts lab-k8s-master lab-k8s-worker1 lab-k8s-worker2
ping -c 2 lab-k8s-master
ping -c 2 lab-k8s-worker1
ping -c 2 lab-k8s-worker2

ping -c 2 192.168.100.1        # tới gateway → phải có reply
ping -c 2 8.8.8.8              # ra Internet bằng IP, không phụ thuộc DNS
getent hosts pkgs.k8s.io       # DNS phải phân giải được
curl -I --max-time 10 https://pkgs.k8s.io/
```

**PASS:** file chặn cloud-init tồn tại; output có `PASS: 50-cloud-init.yaml is absent`; IP
`.221–.223` vẫn giữ nguyên sau reboot; default route đúng; tên trả đúng IP; ping giữa mọi
node, gateway và Internet thành công; DNS cùng HTTPS egress hoạt động.

## A4. Chuẩn bị OS và container runtime

Chạy toàn bộ mục A4 trên **cả ba VM**.

### A4.1. Cập nhật OS, tắt swap và bật kernel prerequisites

```bash
sudo apt-get update
sudo apt-get upgrade -y

# Không tiếp tục nếu upgrade để lại package lỗi hoặc dependency chưa hoàn chỉnh.
sudo dpkg --audit
sudo apt-get check

sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Bật đồng bộ thời gian; lệch giờ có thể làm lỗi certificate và etcd.
sudo timedatectl set-ntp true
timedatectl

sudo swapoff -a
sudo sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab

cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
sudo reboot
```

Chờ VM boot lại, SSH vào đúng node rồi verify:

```bash
dpkg --print-architecture
swapon --show
free -h | grep -i swap
timedatectl
lsmod | grep -E '^(overlay|br_netfilter)'
sysctl net.bridge.bridge-nf-call-iptables \
  net.bridge.bridge-nf-call-ip6tables \
  net.ipv4.ip_forward
stat -fc %T /sys/fs/cgroup
```

**PASS:** `dpkg --print-architecture` trả `amd64` — đúng architecture của bảng A1.3;
`dpkg --audit` không có output; `apt-get check` không báo lỗi; `timedatectl` hiển thị
`System clock synchronized: yes` và `NTP service: active`; `swapon --show` rỗng; Swap là
`0B`; hai module có mặt; ba sysctl bằng `1`; cgroup filesystem là `cgroup2fs`.

### A4.2. Cài containerd và runc đúng version

```bash
CONTAINERD_VER='2.2.1-0ubuntu1~24.04.3'
RUNC_VER='1.3.4-0ubuntu1~24.04.1'

apt-cache madison containerd | grep -F "$CONTAINERD_VER"
apt-cache madison runc | grep -F "$RUNC_VER"

sudo apt-get install -y \
  containerd="$CONTAINERD_VER" \
  runc="$RUNC_VER"

sudo apt-mark hold containerd runc
```

Nếu một trong hai lệnh `grep` không có output, dừng tại đây. Không bỏ version khỏi lệnh cài.
Hai gói có revision khác nhau (`~24.04.3` và `~24.04.1`) — đó là giá trị đúng, không phải lỗi
gõ nhầm.

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
dpkg-query -W -f='${Package} ${Version} ${Architecture}\n' containerd runc
systemctl is-active containerd
sudo ctr plugins ls | grep 'io.containerd.cri.v1'
grep -n 'SystemdCgroup = true' /etc/containerd/config.toml

# containerd cài từ .deb có thể tắt sẵn CRI plugin; file config ở trên không khai
# disabled_plugins, nên lệnh dưới phải không tìm thấy gì.
if grep -qE '^[[:space:]]*disabled_plugins' /etc/containerd/config.toml; then
  echo 'FAIL: disabled_plugins is set; CRI plugin may be off'
else
  echo 'PASS: disabled_plugins is not set'
fi
```

**PASS:** package version và `Architecture` khớp bảng A1.3 (`amd64`); containerd `active`;
CRI plugin trạng thái `ok`; `SystemdCgroup = true`; output có
`PASS: disabled_plugins is not set`.

### A4.3. Cài kubeadm, kubelet, kubectl và crictl

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor --yes -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

K8S_VER='1.35.7-1.1'
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

**PASS:** ba Kubernetes binary là `v1.35.7`, crictl là `v1.35.0`, package CNI là
`1.8.0-1.1`, `crictl version` báo `RuntimeApiVersion: v1`, `systemdCgroup` là `true` và toàn
bộ package đã được hold.

Kubelet có thể restart liên tục trước `kubeadm init/join`; đây là trạng thái dự kiến vì chưa
có `/var/lib/kubelet/config.yaml`.

### A4.4. Firewall cho mạng lab cô lập

Với mạng homelab tin cậy, tắt UFW trên cả ba VM để các lab không bị nhiễu bởi bài firewall:

```bash
sudo ufw status
sudo ufw disable
sudo ufw status
```

**PASS:** trạng thái `inactive`. Không áp dụng lựa chọn này cho cluster production.

## A5. Khởi tạo cluster

### A5.1. Init control plane

Chạy chỉ trên `lab-k8s-master`:

```bash
KUBERNETES_VERSION="$(kubeadm version -o short)"
test "$KUBERNETES_VERSION" = 'v1.35.7'

sudo kubeadm config images list --kubernetes-version "$KUBERNETES_VERSION"
sudo kubeadm config images pull --kubernetes-version "$KUBERNETES_VERSION"

sudo kubeadm init \
  --kubernetes-version "$KUBERNETES_VERSION" \
  --control-plane-endpoint 'lab-k8s-master:6443' \
  --apiserver-advertise-address 192.168.100.221 \
  --pod-network-cidr 10.244.0.0/16
```

Không ngắt lệnh `kubeadm init`. Khi thành công, cấu hình kubeconfig cho user thường:

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
chmod 600 "$HOME/.kube/config"

kubectl version
kubectl config current-context
kubectl get nodes -o wide
```

**PASS:** current context mặc định là `kubernetes-admin@kubernetes`; Node control plane có
tên `lab-k8s-master`. Node có thể còn `NotReady` trước khi cài CNI.

> Nếu sau này cần gộp kubeconfig của hai cluster, hãy làm trên **bản sao** và đổi tên các
> entry `clusters[].name`, `users[].name`, `contexts[].name` cùng các tham chiếu
> `context.cluster`/`context.user` của một file **trước khi merge**. Đây chỉ là metadata cục
> bộ của kubeconfig, có thể đổi lúc đó mà không cần chạy lại `kubeadm init` hoặc dựng lại
> cluster.

### A5.2. Cài Flannel v0.28.9

Chạy trên `lab-k8s-master`:

```bash
kubectl apply -f \
  https://github.com/flannel-io/flannel/releases/download/v0.28.9/kube-flannel.yml

kubectl rollout status daemonset/kube-flannel-ds \
  -n kube-flannel --timeout=180s
kubectl get pods -n kube-flannel -o wide
```

Đối chiếu image thật sự được triển khai với bảng A1.3:

```bash
kubectl -n kube-flannel get daemonset kube-flannel-ds \
  -o jsonpath='{range .spec.template.spec.initContainers[*]}{.image}{"\n"}{end}{range .spec.template.spec.containers[*]}{.image}{"\n"}{end}'

kubectl -n kube-flannel get configmap kube-flannel-cfg \
  -o jsonpath='{.data.net-conf\.json}'
```

**PASS:** DaemonSet rollout thành công; Pod Flannel trên master là `Running`; danh sách image
chứa `ghcr.io/flannel-io/flannel:v0.28.9` và
`ghcr.io/flannel-io/flannel-cni-plugin:v1.9.1-flannel3`; `net-conf.json` có
`"Network": "10.244.0.0/16"` khớp `--pod-network-cidr` đã dùng ở A5.1.

### A5.3. Join hai worker

Trên master, sinh lệnh join mới:

```bash
kubeadm token create --print-join-command
```

Copy nguyên lệnh được sinh và chạy bằng `sudo` trên `lab-k8s-worker1`, rồi
`lab-k8s-worker2`.
Trước khi join, verify trên từng worker:

```bash
getent hosts lab-k8s-master
timeout 3 bash -c 'echo > /dev/tcp/lab-k8s-master/6443' \
  && echo 'PASS: API 6443 reachable' \
  || echo 'FAIL: API 6443 unreachable'
```

Ví dụ hình dạng lệnh, không copy placeholder dưới đây:

```bash
sudo kubeadm join lab-k8s-master:6443 \
  --token <token-thật> \
  --discovery-token-ca-cert-hash sha256:<hash-thật>
```

### A5.4. Gate `01-cluster-ready`

Đây là gate được **mọi lab dùng chuỗi snapshot chính tham chiếu tới**. Chạy trên master:

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

- Cả ba node `Ready` và cùng kubelet `v1.35.7`.
- `containerRuntimeVersion` của cả ba node bắt đầu bằng `containerd://2.2.1`.
- Control plane có role `control-plane`; hai worker không mang role này.
- Tất cả Pod hệ thống `Running`; CoreDNS có đủ replica `READY`.
- `kube-proxy` và Flannel có một Pod sẵn sàng trên mỗi node.

Ghi baseline package trên từng node để có thể tái lập:

```bash
mkdir -p ~/lab-evidence
{
  date -Is
  hostnamectl
  uname -a
  lsb_release -a
  dpkg --print-architecture
  dpkg-query -W -f='${Package} ${Version} ${Architecture}\n' \
    apt-transport-https ca-certificates curl gpg \
    containerd runc kubernetes-cni cri-tools kubelet kubeadm kubectl
} | tee ~/lab-evidence/00-package-baseline.txt
```

Tắt ba VM sạch sẽ và tạo snapshot VMware tên **`01-cluster-ready`** cho từng VM.

### A5.5. Quy trình mở đầu mỗi lab

Chạy trước khi bắt đầu bất kỳ lab nào dùng chuỗi snapshot chính:

1. Bật ba VM theo thứ tự master → worker 1 → worker 2.
2. Chạy lại toàn bộ gate ở A5.4 (hoặc gate của snapshot mà lab đó khai báo là điểm bắt đầu).
3. Chỉ khi gate PASS mới sang phần nội dung của lab.

Nếu gate fail và không sửa được trong vài phút, tắt và restore **cả ba VM** về snapshot đầu
vào rồi chạy lại. Không debug một cluster đã lệch state — thời gian đó nên dành cho bài học.

---

## 4. Troubleshooting môi trường

Bảng này chỉ xử lý sự cố **dựng môi trường**. Sự cố phát sinh trong nội dung bài học nằm ở
mục troubleshooting của chính lab đó.

| Triệu chứng | Kiểm tra | Cách xử lý trong lab |
| --- | --- | --- |
| VM không ping nhau | VMware network mode, IP, route | Đưa cả ba VM vào cùng Bridged/VMnet; sửa IP trước kubeadm |
| `kubeadm init` báo swap | `swapon --show` | `swapoff -a`, comment swap trong `/etc/fstab` |
| CRI không hoạt động | `crictl info`, `ctr plugins ls` | Kiểm tra config version 3 và restart containerd |
| kubelet không register Node | `crictl version` | Runtime phải trả `RuntimeApiVersion: v1`; bỏ `cri` khỏi `disabled_plugins` rồi restart containerd |
| Node `NotReady` sau join | Flannel Pod, kubelet log | Kiểm tra Pod CIDR, egress, module và sysctl |
| Worker không join | DNS và TCP 6443 | Sửa `/etc/hosts`, IP/firewall; sinh token join mới |
| Package không đúng revision | `apt-cache madison <pkg>` | Dừng; cập nhật bảng A1.3 sau khi đối chiếu changelog |
| Package đúng tên nhưng khác version | `dpkg-query -W -f='${Architecture}\n'` | Đọc lại đúng dòng `amd64`; version Ubuntu khác nhau theo architecture |
| Cluster lệch state không rõ nguyên nhân | — | Restore cả ba VM về snapshot đầu vào của lab |

Không reset hoặc restore riêng master bằng snapshot cũ trong khi giữ worker ở state mới. Với
lab một control plane, nếu cần quay lại mốc `01-cluster-ready`, tắt và restore **cả ba VM**.

---

## 5. Nguồn chính thức

- [Kubernetes — Installing kubeadm v1.35](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Kubernetes — Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Kubernetes — Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [Kubernetes — Releases và lifecycle](https://kubernetes.io/releases/patch-releases/)
- [Ubuntu — 24.04.4 LTS images](https://releases.ubuntu.com/noble/)
- [Ubuntu — containerd package cho Noble (amd64)](https://packages.ubuntu.com/noble-updates/amd64/containerd/download)
- [Ubuntu — runc package cho Noble (amd64)](https://packages.ubuntu.com/noble-updates/amd64/runc/download)
- [Flannel — release v0.28.9](https://github.com/flannel-io/flannel/releases/tag/v0.28.9)
- [Runbook tra cứu baseline phiên bản](../../kubeadm-rancher-find-version/runbook-tra-cuu-baseline-phien-ban-k8s.md)
