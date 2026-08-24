# Network Plugin

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), bài 15/16 · Kiểm chứng
ở Lab 5b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Trên trang gốc bài này nằm trong nhánh *Mở rộng Kubernetes*, nên các ví dụ dùng JSON cấu hình
của Calico. **Lộ trình đặt bài ở đây, không phải ở giai đoạn 14**, vì bạn cần hiểu CNI **trước
khi** dựng cluster bằng kubeadm ở giai đoạn 8 và trước khi đổi CNI ở Lab 5b; giai đoạn 14 chỉ
tham chiếu lại bài này. Cluster lab đã chạy sẵn một CNI — đọc bài để biết thứ đó thực sự là gì
và ai nạp nó.

**Phải hiểu ở lần đọc này:**

- Hợp đồng duy nhất Kubernetes đặt ra: một CNI plugin **bắt buộc phải hiện thực mô hình mạng
  Kubernetes**. Ngoài ra plugin phải tương thích đặc tả CNI từ v0.4.0 trở lên, và dự án khuyến
  nghị v1.0.0.
- **Ai nạp CNI plugin: container runtime**, tức daemon chạy trên node cung cấp dịch vụ CRI cho
  kubelet. Từ Kubernetes 1.24, `cni-bin-dir` và `network-plugin` đã bị gỡ và **việc quản lý CNI
  không còn thuộc trách nhiệm của kubelet**.
- Hai yêu cầu bổ sung với plugin: cung cấp interface loopback **`lo`** cho mỗi sandbox, và hỗ
  trợ **`hostPort`** — muốn bật thì phải khai `portMappings capability` trong `cni-conf-dir`.
- Hai đường dẫn phải thuộc lòng khi gỡ lỗi mạng node: file cấu hình CNI mặc định ở
  **`/etc/cni/net.d`**, binary của plugin ở **`/opt/cni/bin`**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Nội dung chi tiết hai file JSON ví dụ (cấu hình Calico) | chỉ cần đọc cấu trúc, không cần thuộc từng trường | Lab 5b |
| *Hỗ trợ điều tiết lưu lượng* — plugin `bandwidth`, annotation `kubernetes.io/ingress-bandwidth` | vẫn là tính năng thử nghiệm | không cần |
| Link *Xử lý sự cố các lỗi liên quan đến CNI plugin* | là runbook sự cố khi dựng node | giai đoạn 8 |

---

Kubernetes (từ phiên bản 1.3 cho đến bản mới nhất v1.36, và nhiều khả năng là cả về sau) cho phép
bạn dùng các plugin [Container Network Interface](https://github.com/containernetworking/cni)
(CNI) cho mạng của cluster. Bạn phải dùng một CNI plugin tương thích với cluster của mình và phù
hợp với nhu cầu của bạn. Có nhiều plugin khác nhau (cả mã nguồn mở lẫn mã nguồn đóng) trong hệ
sinh thái Kubernetes rộng lớn.

Một CNI plugin bắt buộc phải hiện thực
[mô hình mạng Kubernetes](81-services-networking-vi.md#the-kubernetes-network-model).

Bạn phải dùng một CNI plugin tương thích với bản
[v0.4.0](https://github.com/containernetworking/cni/blob/spec-v0.4.0/SPEC.md) trở lên của đặc tả
CNI. Dự án Kubernetes khuyến nghị dùng plugin tương thích với đặc tả CNI
[v1.0.0](https://github.com/containernetworking/cni/blob/spec-v1.0.0/SPEC.md)
(một plugin có thể tương thích với nhiều phiên bản đặc tả).

## Cài đặt (Installation)

Trong ngữ cảnh mạng, một Container Runtime là một daemon chạy trên node được cấu hình để cung cấp
các dịch vụ CRI cho kubelet. Cụ thể, Container Runtime phải được cấu hình để nạp các CNI plugin
cần thiết nhằm hiện thực mô hình mạng Kubernetes.

> **Ghi chú:** Trước Kubernetes 1.24, các CNI plugin cũng có thể được kubelet quản lý thông qua
> các tham số dòng lệnh `cni-bin-dir` và `network-plugin`.
> Những tham số dòng lệnh này đã bị loại bỏ trong Kubernetes 1.24, và việc quản lý CNI không còn
> nằm trong phạm vi trách nhiệm của kubelet nữa.
>
> Hãy xem [Xử lý sự cố các lỗi liên quan đến CNI plugin](241-troubleshooting-cni-errors-vi.md)
> nếu bạn gặp vấn đề sau khi dockershim bị loại bỏ.

Để biết thông tin cụ thể về cách một Container Runtime quản lý các CNI plugin, hãy xem tài liệu
của Container Runtime đó, ví dụ:

- [containerd](https://github.com/containerd/containerd/blob/main/script/setup/install-cni)
- [CRI-O](https://github.com/cri-o/cri-o/blob/main/contrib/cni/README.md)

Để biết thông tin cụ thể về cách cài đặt và quản lý một CNI plugin, hãy xem tài liệu của plugin đó
hoặc của [nhà cung cấp giải pháp mạng](157-networking-vi.md#how-to-implement-the-kubernetes-network-model).

## Yêu cầu đối với Network Plugin (Network Plugin Requirements)

### Loopback CNI

Ngoài CNI plugin được cài trên các node để hiện thực mô hình mạng Kubernetes, Kubernetes còn yêu
cầu container runtime phải cung cấp một interface loopback `lo`, được dùng cho mỗi sandbox
(pod sandbox, vm sandbox, ...).
Việc hiện thực interface loopback có thể thực hiện bằng cách tái sử dụng
[CNI loopback plugin](https://github.com/containernetworking/plugins/blob/master/plugins/main/loopback/loopback.go)
hoặc bằng cách tự viết mã của riêng bạn để đạt được điều này (xem
[ví dụ này từ CRI-O](https://github.com/cri-o/ocicni/blob/release-1.24/pkg/ocicni/util_linux.go#L91)).

### Hỗ trợ hostPort (Support hostPort)

CNI networking plugin có hỗ trợ `hostPort`. Bạn có thể dùng plugin
[portmap](https://github.com/containernetworking/plugins/tree/master/plugins/meta/portmap)
chính thức do nhóm CNI plugin cung cấp, hoặc dùng plugin của riêng bạn có chức năng portMapping.

Nếu bạn muốn bật hỗ trợ `hostPort`, bạn phải chỉ định `portMappings capability` trong
`cni-conf-dir` của mình. Ví dụ:

```json
{
  "name": "k8s-pod-network",
  "cniVersion": "0.4.0",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "nodename": "127.0.0.1",
      "ipam": {
        "type": "host-local",
        "subnet": "usePodCidr"
      },
      "policy": {
        "type": "k8s"
      },
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true},
      "externalSetMarkChain": "KUBE-MARK-MASQ"
    }
  ]
}
```

### Hỗ trợ điều tiết lưu lượng (Support traffic shaping) {#support-traffic-shaping}

**Tính năng thử nghiệm (Experimental Feature)**

CNI networking plugin cũng hỗ trợ điều tiết lưu lượng (traffic shaping) vào và ra của pod. Bạn có
thể dùng plugin [bandwidth](https://github.com/containernetworking/plugins/tree/master/plugins/meta/bandwidth)
chính thức do nhóm CNI plugin cung cấp, hoặc dùng plugin của riêng bạn có chức năng kiểm soát băng
thông.

Nếu bạn muốn bật hỗ trợ điều tiết lưu lượng, bạn phải thêm plugin `bandwidth` vào file cấu hình
CNI của mình (mặc định là `/etc/cni/net.d`) và đảm bảo binary đó nằm trong thư mục bin của CNI
(mặc định là `/opt/cni/bin`).

```json
{
  "name": "k8s-pod-network",
  "cniVersion": "0.4.0",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "nodename": "127.0.0.1",
      "ipam": {
        "type": "host-local",
        "subnet": "usePodCidr"
      },
      "policy": {
        "type": "k8s"
      },
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "bandwidth",
      "capabilities": {"bandwidth": true}
    }
  ]
}
```

Giờ đây bạn có thể thêm các annotation `kubernetes.io/ingress-bandwidth` và
`kubernetes.io/egress-bandwidth` vào Pod của mình. Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubernetes.io/ingress-bandwidth: 1M
    kubernetes.io/egress-bandwidth: 1M
...
```

## Tiếp theo (What's next)

- Tìm hiểu thêm về [Mạng trong Cluster (Cluster Networking)](157-networking-vi.md)
- Tìm hiểu thêm về [Network Policy](84-network-policies-vi.md)
- Tìm hiểu về [Xử lý sự cố các lỗi liên quan đến CNI plugin](241-troubleshooting-cni-errors-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5:

1. Trên `k8s-worker2` của cluster lab, thành phần nào thực sự đi nạp CNI plugin — kubelet hay
   containerd? Bạn nhìn vào hai thư mục nào để kiểm tra?
2. Một plugin quảng cáo "tương thích đặc tả CNI v1.0.0". Điều đó có bảo đảm nó thực thi
   NetworkPolicy không?
3. Một Pod dùng `hostPort` nhưng không nhận được kết nối. Theo bài, cấu hình CNI đang thiếu gì?
4. Ngoài việc hiện thực mô hình mạng Kubernetes, Kubernetes còn đòi container runtime cung cấp
   thứ gì cho mỗi sandbox?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **containerd** — tức container runtime cung cấp dịch vụ CRI cho kubelet. Bài nói rõ:
   Container Runtime phải được cấu hình để nạp các CNI plugin cần thiết, và **trước** Kubernetes
   1.24 kubelet còn quản lý CNI qua `cni-bin-dir` và `network-plugin`, nhưng **các tham số đó đã
   bị loại bỏ trong 1.24 và việc quản lý CNI không còn nằm trong phạm vi trách nhiệm của
   kubelet**. Hai thư mục cần nhìn: **`/etc/cni/net.d`** (cấu hình) và **`/opt/cni/bin`**
   (binary).
2. **Không.** Thứ duy nhất đặc tả bắt buộc là plugin **hiện thực mô hình mạng Kubernetes**, cộng
   với hai yêu cầu bổ sung mà bài liệt kê: interface loopback và hỗ trợ `hostPort`. **NetworkPolicy
   không nằm trong danh sách đó** — nó chỉ được nhắc ở mục *Tiếp theo* như một chủ đề riêng. Đây
   đúng là tình huống của baseline lab: CNI hợp lệ, mạng Pod chạy tốt, nhưng NetworkPolicy không
   được thực thi (xem [sổ nợ lab](labs/README.md#5-sổ-nợ-lab)).
3. Thiếu khai báo **`portMappings capability` trong `cni-conf-dir`**. Bài nói: muốn bật hỗ trợ
   `hostPort` thì phải chỉ định `portMappings capability`, và bạn có thể dùng plugin **`portmap`**
   chính thức của nhóm CNI plugin hoặc plugin của riêng bạn có chức năng portMapping.
4. Một **interface loopback `lo`** cho mỗi sandbox (pod sandbox, vm sandbox…). Có thể đạt được
   bằng cách tái sử dụng CNI loopback plugin, hoặc tự viết mã của riêng bạn.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
