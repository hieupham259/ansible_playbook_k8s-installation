# Thêm node worker Windows (Adding Windows worker nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/adding-windows-nodes/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Checkpoint tiếp nối — nhánh `/docs/tasks/`](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 16 — Vòng đời node](00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node), bài 3/4 · Chỉ kiểm chứng được nếu
môi trường có thêm một VM Windows Server; xem **Lab 15 — Node Windows** (tùy chọn) trong
[bản đồ lab](labs/README.md#4-bản-đồ-lab).

**Bỏ qua bài này nếu cluster của bạn chỉ có Linux** — cluster lab của Lab 00 là thuần Linux.
Đọc lướt một lượt để biết quy trình khác gì với node Linux, rồi quay lại khi thực sự cần.
Lưu ý bài dùng script PowerShell của `sig-windows-tools`, tức phụ thuộc dự án bên thứ ba, khác
hẳn cách cài thuần kubeadm ở bài [215](215-adding-linux-nodes-vi.md).

**Phải hiểu ở lần đọc này:**

- Điểm giống: bước cuối vẫn là `kubeadm join` với **đúng ba thành phần** như node Linux — token,
  `<host>:<port>`, `--discovery-token-ca-cert-hash`. Phần lấy lại token và hash giống hệt bài
  [215](215-adding-linux-nodes-vi.md).
- Điểm khác lớn nhất: **chuẩn bị node** không dùng package manager mà dùng hai script
  (`Install-Containerd.ps1`, `PrepareNode.ps1`), chạy trong PowerShell với quyền Administrator.
- **CNI là chỗ khó thật sự.** Cluster lẫn Linux và Windows không thể chỉ `kubectl apply` một
  manifest: plugin CNI trên node control plane phải được chuẩn bị để hỗ trợ plugin CNI của node
  Windows, và rất ít CNI hỗ trợ Windows.
- Yêu cầu nền: Windows Server 2022 trở lên, quyền quản trị, và một cluster kubeadm đã chạy sẵn.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Tham số chi tiết của `Install-Containerd.ps1` (`netAdapterName`, `skipHypervisorSupportCheck`, `CNIBinPath`) | chỉ có nghĩa khi ngồi trước một máy Windows thật | Lab 15 (tùy chọn), nếu môi trường có VM Windows |
| Thiết lập Calico cho Windows | thuộc phần mạng của node Windows | bài [89](89-windows-networking-vi.md) ở giai đoạn 15 |
| Cài kubectl trên Windows | công cụ client, không liên quan tới việc join node | bài [188](188-install-kubectl-windows-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.18 [beta]`

Trang này giải thích cách thêm các node worker Windows vào một cluster kubeadm.

## Trước khi bạn bắt đầu (Before you begin)

* Một instance [Windows Server 2022](https://www.microsoft.com/cloud-platform/windows-server-pricing)
  (hoặc cao hơn) đang chạy, với quyền truy cập quản trị (administrative access).
* Một cluster kubeadm đang chạy, được tạo bằng `kubeadm init` và theo các bước trong tài liệu
  [Tạo cluster với kubeadm](02-create-cluster-kubeadm-vi.md).

## Thêm node worker Windows (Adding Windows worker nodes)

> **Ghi chú:**
>
> Để thuận tiện cho việc thêm node worker Windows vào cluster, các script PowerShell từ
> repository https://sigs.k8s.io/sig-windows-tools sẽ được sử dụng.

Thực hiện các bước sau trên từng máy:

1. Mở một phiên PowerShell trên máy.
1. Đảm bảo bạn là Administrator hoặc một người dùng có đặc quyền.

Sau đó tiếp tục với các bước được trình bày bên dưới.

### Cài đặt containerd (Install containerd)

> **Ghi chú:** Mục này liên kết tới các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần.
> Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn được liệt kê
> theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc
> [hướng dẫn nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content)
> trước khi gửi thay đổi.

Để cài đặt containerd, trước tiên hãy chạy lệnh sau:

```PowerShell
curl.exe -LO https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/Install-Containerd.ps1
```

Sau đó chạy lệnh dưới đây, nhưng trước tiên hãy thay `CONTAINERD_VERSION` bằng một bản phát hành
gần đây từ [repository containerd](https://github.com/containerd/containerd/releases).
Phiên bản không được có tiền tố `v`. Ví dụ, dùng `1.7.22` thay vì `v1.7.22`:

```PowerShell
.\Install-Containerd.ps1 -ContainerDVersion CONTAINERD_VERSION
```

* Điều chỉnh các tham số khác của `Install-Containerd.ps1` như `netAdapterName` nếu bạn cần.
* Đặt `skipHypervisorSupportCheck` nếu máy của bạn không hỗ trợ Hyper-V và không thể chạy các
  container cách ly bằng Hyper-V (Hyper-V isolated containers).
* Nếu bạn thay đổi các tham số tùy chọn `CNIBinPath` và/hoặc `CNIConfigPath` của
  `Install-Containerd.ps1`, bạn sẽ cần cấu hình plugin CNI cho Windows được cài đặt với các giá
  trị tương ứng.

### Cài đặt kubeadm và kubelet (Install kubeadm and kubelet)

Chạy các lệnh sau để cài đặt kubeadm và kubelet:

```PowerShell
curl.exe -LO https://raw.githubusercontent.com/kubernetes-sigs/sig-windows-tools/master/hostprocess/PrepareNode.ps1
.\PrepareNode.ps1 -KubernetesVersion v1.36.0
```

* Điều chỉnh tham số `KubernetesVersion` của `PrepareNode.ps1` nếu cần.

### Chạy `kubeadm join`

Chạy lệnh mà `kubeadm init` đã in ra. Ví dụ:

```bash
kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

#### Thông tin bổ sung về kubeadm join (Additional information about kubeadm join)

> **Ghi chú:**
>
> Để chỉ định một bộ giá trị IPv6 cho `<control-plane-host>:<control-plane-port>`, địa chỉ IPv6
> phải được đặt trong cặp ngoặc vuông, ví dụ: `[2001:db8::101]:2073`.

Nếu bạn không có token, bạn có thể lấy nó bằng cách chạy lệnh sau trên node control plane:

```bash
# Chạy lệnh này trên một node control plane
sudo kubeadm token list
```

Kết quả đầu ra tương tự như sau:

```console
TOKEN                    TTL  EXPIRES              USAGES           DESCRIPTION            EXTRA GROUPS
8ewj1p.9r9hcjoqgajrj4gi  23h  2018-06-12T02:51:28Z authentication,  The default bootstrap  system:
                                                   signing          token generated by     bootstrappers:
                                                                    'kubeadm init'.        kubeadm:
                                                                                           default-node-token
```

Theo mặc định, token dùng để join node sẽ hết hạn sau 24 giờ. Nếu bạn join một node vào cluster
sau khi token hiện tại đã hết hạn, bạn có thể tạo token mới bằng cách chạy lệnh sau trên node
control plane:

```bash
# Chạy lệnh này trên một node control plane
sudo kubeadm token create
```

Kết quả đầu ra tương tự như sau:

```console
5didvk.d09sbcov8ph2amjw
```

Nếu bạn không có giá trị của `--discovery-token-ca-cert-hash`, bạn có thể lấy nó bằng cách chạy
các lệnh sau trên node control plane:

```bash
sudo cat /etc/kubernetes/pki/ca.crt | openssl x509 -pubkey  | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

Kết quả đầu ra tương tự như:

```console
8cb2de97839780a412b93877f8507ad6c94f73add17d5d7058e91741c9d5ec78
```

Đầu ra của lệnh `kubeadm join` sẽ trông tương tự như sau:

```
[preflight] Running pre-flight checks

... (log output of join workflow)...

Node join complete:
* Certificate signing request sent to control-plane and response
  received.
* Kubelet informed of new secure connection details.

Run 'kubectl get nodes' on control-plane to see this machine join.
```

Vài giây sau, bạn sẽ thấy node này xuất hiện trong đầu ra của lệnh `kubectl get nodes`
(ví dụ, chạy `kubectl` trên một node control plane).

### Cấu hình mạng (Network configuration)

Việc thiết lập CNI trên các cluster có cả node Linux lẫn node Windows đòi hỏi nhiều bước hơn so
với việc chỉ chạy `kubectl apply` trên một file manifest. Ngoài ra, plugin CNI chạy trên các
node control plane phải được chuẩn bị để hỗ trợ plugin CNI chạy trên các node worker Windows.

> **Ghi chú:** Mục này liên kết tới các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần.
> Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn được liệt kê
> theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc
> [hướng dẫn nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content)
> trước khi gửi thay đổi.

Hiện tại chỉ có một số ít plugin CNI hỗ trợ Windows. Dưới đây là hướng dẫn thiết lập riêng cho
từng plugin:

* [Calico](https://docs.tigera.io/calico/latest/getting-started/kubernetes/windows-calico/)

### Cài đặt kubectl cho Windows (tùy chọn) {#install-kubectl}

Xem [Cài đặt và thiết lập kubectl trên Windows](188-install-kubectl-windows-vi.md).

## Tiếp theo (What's next)

* Xem cách [thêm node worker Linux](215-adding-linux-nodes-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 16:

1. Cluster lab của bạn (`k8s-master` + hai worker Linux, CNI theo baseline Lab 00) muốn thêm
   một node Windows. Theo bài, bước nào **không** thể làm bằng một lệnh `kubectl apply` như với
   node Linux, và vì sao?
2. **Câu bẫy.** Quy trình join node Windows dùng script PowerShell riêng, vậy lệnh `kubeadm join`
   cuối cùng có khác lệnh dùng cho node Linux không? Token và
   `--discovery-token-ca-cert-hash` lấy ở đâu?
3. Hai script `Install-Containerd.ps1` và `PrepareNode.ps1` thay thế cho công việc gì mà trên
   node Linux bạn làm theo bài [01](01-install-kubeadm-vi.md)?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Cấu hình CNI.** Bài nói rõ: thiết lập CNI trên cluster có cả node Linux lẫn Windows đòi
   hỏi nhiều bước hơn là `kubectl apply` một manifest, vì **plugin CNI chạy trên node control
   plane cũng phải được chuẩn bị để hỗ trợ plugin CNI của node Windows**. Thêm nữa, chỉ một số
   ít CNI hỗ trợ Windows, nên CNI đang dùng có thể phải đổi.
2. **Không khác.** Lệnh join vẫn là `kubeadm join --token <token> <host>:<port>
   --discovery-token-ca-cert-hash sha256:<hash>`. Token lấy bằng `kubeadm token list` /
   `kubeadm token create` và hash lấy từ `/etc/kubernetes/pki/ca.crt` — **đều chạy trên node
   control plane Linux**, y hệt bài 215. Chỗ dễ nhầm là tưởng "node Windows thì mọi thứ đều
   khác"; thực ra chỉ phần *chuẩn bị node* và *mạng* khác, còn giao thức gia nhập cluster thì
   chung một cơ chế.
3. Chúng thay cho phần **cài container runtime và cài kubelet/kubeadm** — tức đúng phần việc
   mà trên Linux bạn làm bằng package manager theo bài [01](01-install-kubeadm-vi.md).
   `Install-Containerd.ps1` lo runtime, `PrepareNode.ps1` lo kubelet và kubeadm.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Nếu cluster của bạn chỉ có Linux,
đánh dấu bài này là đã đọc và sang bài kế của [Giai đoạn 16 — Vòng đời node](00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node).
