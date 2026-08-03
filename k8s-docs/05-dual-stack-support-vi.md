# Hỗ trợ dual-stack với kubeadm (Dual-stack support with kubeadm)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/dual-stack-support/>
>
> Trang này hướng dẫn cách tạo cluster Kubernetes dual-stack (IPv4/IPv6) bằng kubeadm: điều kiện tiên quyết, khởi tạo control plane dual-stack, thêm node vào cluster dual-stack, và tạo cluster single-stack.

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.23 [stable]`

Cluster Kubernetes của bạn bao gồm mạng [dual-stack](https://kubernetes.io/docs/concepts/services-networking/dual-stack/),
nghĩa là mạng của cluster cho phép bạn dùng bất kỳ họ địa chỉ (address family) nào.
Trong một cluster, control plane có thể gán đồng thời cả địa chỉ IPv4 và địa chỉ IPv6 cho một
Pod hoặc một Service duy nhất.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần đã cài đặt công cụ kubeadm,
theo các bước trong [Cài đặt kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

Đối với mỗi máy chủ mà bạn muốn dùng làm node,
hãy đảm bảo nó cho phép chuyển tiếp IPv6 (IPv6 forwarding).

### Bật chuyển tiếp gói tin IPv6 (Enable IPv6 packet forwarding) {#prerequisite-ipv6-forwarding}

Để kiểm tra xem chuyển tiếp gói tin IPv6 đã được bật hay chưa:

```bash
sysctl net.ipv6.conf.all.forwarding
```

Nếu kết quả là `net.ipv6.conf.all.forwarding = 1` thì nó đã được bật.
Ngược lại thì nó chưa được bật.

Để bật chuyển tiếp gói tin IPv6 một cách thủ công:

```bash
# Các tham số sysctl cần thiết cho quá trình cài đặt, các tham số này được giữ nguyên qua các lần khởi động lại
cat <<EOF | sudo tee -a /etc/sysctl.d/k8s.conf
net.ipv6.conf.all.forwarding = 1
EOF

# Áp dụng các tham số sysctl mà không cần khởi động lại
sudo sysctl --system
```

Bạn cần có một dải địa chỉ IPv4 và một dải địa chỉ IPv6 để sử dụng. Người vận hành cluster thường
dùng các dải địa chỉ riêng (private) cho IPv4. Với IPv6, người vận hành cluster thường chọn một khối
địa chỉ unicast toàn cầu (global unicast) trong `2000::/3`, sử dụng một dải đã được cấp cho người vận hành đó.
Bạn không bắt buộc phải định tuyến các dải địa chỉ IP của cluster ra internet công cộng.

Kích thước của các dải địa chỉ IP được cấp phát nên phù hợp với số lượng Pod và
Service mà bạn dự định chạy.

> **Ghi chú:** Nếu bạn đang nâng cấp một cluster hiện có bằng lệnh `kubeadm upgrade`,
> `kubeadm` không hỗ trợ thay đổi dải địa chỉ IP của Pod
> ("cluster CIDR") cũng như dải địa chỉ Service của cluster ("Service CIDR").

### Tạo một cluster dual-stack (Create a dual-stack cluster)

Để tạo một cluster dual-stack với `kubeadm init`, bạn có thể truyền các đối số dòng lệnh
tương tự như ví dụ sau:

```shell
# Các dải địa chỉ này chỉ là ví dụ
kubeadm init --pod-network-cidr=10.244.0.0/16,2001:db8:42:0::/56 --service-cidr=10.96.0.0/16,2001:db8:42:1::/112
```

Để mọi thứ rõ ràng hơn, đây là một ví dụ về
[file cấu hình](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/) kubeadm
`kubeadm-config.yaml` cho node control plane dual-stack chính (primary).

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
networking:
  podSubnet: 10.244.0.0/16,2001:db8:42:0::/56
  serviceSubnet: 10.96.0.0/16,2001:db8:42:1::/112
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "10.100.0.1"
  bindPort: 6443
nodeRegistration:
  kubeletExtraArgs:
  - name: "node-ip"
    value: "10.100.0.2,fd00:1:2:3::2"
```

`advertiseAddress` trong InitConfiguration chỉ định địa chỉ IP mà API Server
sẽ quảng bá (advertise) rằng nó đang lắng nghe trên đó. Giá trị của `advertiseAddress` tương đương với
cờ `--apiserver-advertise-address` của `kubeadm init`.

Chạy kubeadm để khởi tạo node control plane dual-stack:

```shell
kubeadm init --config=kubeadm-config.yaml
```

Các cờ `--node-cidr-mask-size-ipv4|--node-cidr-mask-size-ipv6` của kube-controller-manager
được đặt với các giá trị mặc định. Xem [cấu hình dual-stack IPv4/IPv6](https://kubernetes.io/docs/concepts/services-networking/dual-stack#configure-ipv4-ipv6-dual-stack).

> **Ghi chú:** Cờ `--apiserver-advertise-address` không hỗ trợ dual-stack.

### Thêm một node vào cluster dual-stack (Join a node to dual-stack cluster)

Trước khi thêm (join) một node, hãy đảm bảo node đó có giao diện mạng (network interface) định tuyến được bằng IPv6 và cho phép chuyển tiếp IPv6.

Đây là một ví dụ về [file cấu hình](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/) kubeadm
`kubeadm-config.yaml` để thêm một worker node vào cluster.

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: 10.100.0.1:6443
    token: "clvldh.vjjwg16ucnhp94qr"
    caCertHashes:
    - "sha256:a4863cde706cfc580a439f842cc65d5ef112b7b2be31628513a9881cf0d9fe0e"
    # thay đổi thông tin xác thực ở trên cho khớp với token và hash chứng chỉ CA thực tế của cluster của bạn
nodeRegistration:
  kubeletExtraArgs:
  - name: "node-ip"
    value: "10.100.0.2,fd00:1:2:3::3"
```

Ngoài ra, đây là một ví dụ về [file cấu hình](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/) kubeadm
`kubeadm-config.yaml` để thêm một node control plane khác vào cluster.

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
controlPlane:
  localAPIEndpoint:
    advertiseAddress: "10.100.0.2"
    bindPort: 6443
discovery:
  bootstrapToken:
    apiServerEndpoint: 10.100.0.1:6443
    token: "clvldh.vjjwg16ucnhp94qr"
    caCertHashes:
    - "sha256:a4863cde706cfc580a439f842cc65d5ef112b7b2be31628513a9881cf0d9fe0e"
    # thay đổi thông tin xác thực ở trên cho khớp với token và hash chứng chỉ CA thực tế của cluster của bạn
nodeRegistration:
  kubeletExtraArgs:
  - name: "node-ip"
    value: "10.100.0.2,fd00:1:2:3::4"
```

`advertiseAddress` trong JoinConfiguration.controlPlane chỉ định địa chỉ IP mà
API Server sẽ quảng bá rằng nó đang lắng nghe trên đó. Giá trị của `advertiseAddress` tương đương với
cờ `--apiserver-advertise-address` của `kubeadm join`.

```shell
kubeadm join --config=kubeadm-config.yaml
```

### Tạo một cluster single-stack (Create a single-stack cluster)

> **Ghi chú:** Hỗ trợ dual-stack không có nghĩa là bạn bắt buộc phải dùng địa chỉ dual-stack.
> Bạn có thể triển khai một cluster single-stack mà tính năng mạng dual-stack vẫn được bật.

Để mọi thứ rõ ràng hơn, đây là một ví dụ về
[file cấu hình](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/) kubeadm
`kubeadm-config.yaml` cho node control plane single-stack.

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/16
```

## Tiếp theo (What's next)

* [Kiểm chứng mạng dual-stack IPv4/IPv6](https://kubernetes.io/docs/tasks/network/validate-dual-stack)
* Đọc về mạng cluster [Dual-stack](https://kubernetes.io/docs/concepts/services-networking/dual-stack/)
* Tìm hiểu thêm về [định dạng cấu hình](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/) của kubeadm
