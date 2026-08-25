# Hỗ trợ dual-stack với kubeadm (Dual-stack support with kubeadm)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/dual-stack-support/>
>
> Trang này hướng dẫn cách tạo cluster Kubernetes dual-stack (IPv4/IPv6) bằng kubeadm: điều kiện tiên quyết, khởi tạo control plane dual-stack, thêm node vào cluster dual-stack, và tạo cluster single-stack.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 8](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm), bài 8/9 ·
Kiểm chứng ở [Lab 8a](labs/LAB-8A-DUNG-CLUSTER-BANG-KUBEADM.md).

Bài ngắn, và bạn **không dựng được** nội dung của nó trên cluster lab: mạng
`192.168.100.0/24` chỉ có IPv4. Vì vậy đọc nó theo một mục đích khác — nhận ra rằng **cấu hình
họ địa chỉ là quyết định chốt một lần lúc `kubeadm init`**, giống hệt `--control-plane-endpoint`
ở bài [02](02-create-cluster-kubeadm-vi.md). Phần đáng nhớ chỉ khoảng mười dòng: đổi ở đâu, và
cái gì **không** đổi được về sau.

**Phải hiểu ở lần đọc này:**

- Dual-stack nghĩa là control plane có thể gán **đồng thời** một địa chỉ IPv4 và một địa chỉ
  IPv6 cho **cùng một** Pod hoặc Service. Nó không bắt buộc bạn dùng cả hai: mục *Tạo một
  cluster single-stack* nói rõ có thể triển khai single-stack **mà tính năng dual-stack vẫn được bật**.
- *Bật chuyển tiếp gói tin IPv6*: mỗi node phải có `net.ipv6.conf.all.forwarding = 1`, kiểm tra
  bằng `sysctl net.ipv6.conf.all.forwarding` và đặt bền vững qua file trong `/etc/sysctl.d/`.
  Đây là bản song sinh IPv6 của `net.ipv4.ip_forward` mà
  [A4.1 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites)
  đã đặt.
- Cách khai hai dải: `--pod-network-cidr` và `--service-cidr` nhận **danh sách hai dải cách nhau
  bằng dấu phẩy**, hoặc tương đương là `ClusterConfiguration.networking.podSubnet` /
  `serviceSubnet`. Ghi chú quan trọng: `kubeadm upgrade` **không hỗ trợ thay đổi** cluster CIDR
  và Service CIDR của cluster đã có.
- Mỗi node phải tự khai IP của mình qua `nodeRegistration.kubeletExtraArgs` với
  `name: "node-ip"` và **hai địa chỉ cách nhau dấu phẩy** — trong `InitConfiguration` với node
  đầu tiên, trong `JoinConfiguration` với node join sau.
- Ranh giới dễ nhầm: `advertiseAddress` (tương đương cờ `--apiserver-advertise-address`) chỉ nói
  API Server quảng bá nó lắng nghe trên địa chỉ nào, và cờ này **không hỗ trợ dual-stack**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chọn khối IPv6 nào (`2000::/3`) và kích thước dải nên cấp | phụ thuộc dải được cấp cho tổ chức | không cần |
| `--node-cidr-mask-size-ipv4` / `--node-cidr-mask-size-ipv6` của kube-controller-manager | tinh chỉnh cấp phát CIDR cho từng node | giai đoạn 21 DNS, CNI và kube-proxy |
| Giá trị `token` và `caCertHashes` trong ví dụ `JoinConfiguration` | là dữ liệu thật của từng cluster, không phải cú pháp cần nhớ | không cần |
| Trang *Kiểm chứng mạng dual-stack IPv4/IPv6* ở mục Tiếp theo | mạng lab chỉ có IPv4 | không cần |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.23 [stable]`

Cluster Kubernetes của bạn bao gồm mạng [dual-stack](85-dual-stack-vi.md),
nghĩa là mạng của cluster cho phép bạn dùng bất kỳ họ địa chỉ (address family) nào.
Trong một cluster, control plane có thể gán đồng thời cả địa chỉ IPv4 và địa chỉ IPv6 cho một
Pod hoặc một Service duy nhất.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần đã cài đặt công cụ kubeadm,
theo các bước trong [Cài đặt kubeadm](01-install-kubeadm-vi.md).

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
được đặt với các giá trị mặc định. Xem [cấu hình dual-stack IPv4/IPv6](85-dual-stack-vi.md#configure-ipv4-ipv6-dual-stack).

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

* [Kiểm chứng mạng dual-stack IPv4/IPv6](395-validate-dual-stack-vi.md)
* Đọc về mạng cluster [Dual-stack](85-dual-stack-vi.md)
* Tìm hiểu thêm về [định dạng cấu hình](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/) của kubeadm

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 8:

1. Cluster lab của bạn đã dựng xong với Pod CIDR `10.244.0.0/16`. Sáu tháng sau bạn muốn thêm
   một dải IPv6. `kubeadm upgrade` giải quyết được không?
2. Bạn muốn dựng lại cluster lab ở chế độ dual-stack. Ngoài `net.ipv4.ip_forward` mà
   [A4.1 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites)
   đã đặt, phải thêm sysctl nào trên **mỗi** node, và kiểm tra bằng lệnh gì?
3. Bật dual-stack có bắt buộc mọi Pod và Service phải có cả hai họ địa chỉ không?
4. Trong file cấu hình dual-stack, `advertiseAddress` nhận được mấy địa chỉ? Còn `node-ip`
   trong `kubeletExtraArgs` thì sao? Hai trường này khai thứ gì khác nhau?
5. Nếu không dùng cờ dòng lệnh mà dùng file cấu hình, hai dải Pod và Service được khai ở
   `kind` nào, dưới trường nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Bài ghi chú thẳng: nếu bạn đang nâng cấp một cluster hiện có bằng `kubeadm
   upgrade`, kubeadm **không hỗ trợ thay đổi dải địa chỉ IP của Pod ("cluster CIDR") cũng như
   dải địa chỉ Service ("Service CIDR")**. Đây là quyết định chốt tại thời điểm `kubeadm init`,
   cùng nhóm với `--control-plane-endpoint` ở bài [02](02-create-cluster-kubeadm-vi.md): muốn
   đổi thì phải dựng lại cluster.
2. **`net.ipv6.conf.all.forwarding = 1`**, đặt bằng cách ghi vào một file trong `/etc/sysctl.d/`
   rồi `sudo sysctl --system` để nó **được giữ nguyên qua các lần khởi động lại**. Kiểm tra
   bằng `sysctl net.ipv6.conf.all.forwarding` — kết quả `net.ipv6.conf.all.forwarding = 1`
   nghĩa là đã bật. Bài đặt yêu cầu này ở mức **mỗi máy chủ mà bạn muốn dùng làm node**, tức cả
   ba VM, đúng như A4.1 làm với ba sysctl IPv4.
3. **Không.** Ghi chú ở mục *Tạo một cluster single-stack* nói rõ: **hỗ trợ dual-stack không có
   nghĩa là bạn bắt buộc phải dùng địa chỉ dual-stack** — bạn có thể triển khai một cluster
   single-stack mà tính năng mạng dual-stack vẫn được bật. Chỗ dễ nhầm là coi "cluster
   dual-stack" và "tính năng dual-stack" là một; cái quyết định là bạn khai **mấy dải** trong
   `podSubnet`/`serviceSubnet`.
4. `advertiseAddress` nhận **một** địa chỉ, vì bài ghi chú **cờ `--apiserver-advertise-address`
   không hỗ trợ dual-stack** (và `advertiseAddress` chính là tương đương của cờ đó). `node-ip`
   thì nhận **hai** địa chỉ cách nhau dấu phẩy, ví dụ `"10.100.0.2,fd00:1:2:3::2"`. Chúng khai
   hai thứ khác nhau: `advertiseAddress` là địa chỉ mà **API Server quảng bá rằng nó đang lắng
   nghe trên đó**, còn `node-ip` là địa chỉ của **node** mà kubelet đăng ký.
5. Trong **`ClusterConfiguration`**, dưới **`networking`**, với hai trường **`podSubnet`** và
   **`serviceSubnet`**, mỗi trường nhận danh sách phân tách bằng dấu phẩy — tương đương
   `--pod-network-cidr` và `--service-cidr`. `InitConfiguration` trong cùng file lo phần riêng
   của node đầu tiên: `localAPIEndpoint.advertiseAddress`, `bindPort` và `nodeRegistration`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
