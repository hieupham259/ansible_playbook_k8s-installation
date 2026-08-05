# Network Plugin

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/>

Kubernetes (từ phiên bản 1.3 cho đến bản mới nhất v1.36, và nhiều khả năng là cả về sau) cho phép
bạn dùng các plugin [Container Network Interface](https://github.com/containernetworking/cni)
(CNI) cho mạng của cluster. Bạn phải dùng một CNI plugin tương thích với cluster của mình và phù
hợp với nhu cầu của bạn. Có nhiều plugin khác nhau (cả mã nguồn mở lẫn mã nguồn đóng) trong hệ
sinh thái Kubernetes rộng lớn.

Một CNI plugin bắt buộc phải hiện thực
[mô hình mạng Kubernetes](https://kubernetes.io/docs/concepts/services-networking/#the-kubernetes-network-model).

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
> Hãy xem [Xử lý sự cố các lỗi liên quan đến CNI plugin](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/troubleshooting-cni-plugin-related-errors/)
> nếu bạn gặp vấn đề sau khi dockershim bị loại bỏ.

Để biết thông tin cụ thể về cách một Container Runtime quản lý các CNI plugin, hãy xem tài liệu
của Container Runtime đó, ví dụ:

- [containerd](https://github.com/containerd/containerd/blob/main/script/setup/install-cni)
- [CRI-O](https://github.com/cri-o/cri-o/blob/main/contrib/cni/README.md)

Để biết thông tin cụ thể về cách cài đặt và quản lý một CNI plugin, hãy xem tài liệu của plugin đó
hoặc của [nhà cung cấp giải pháp mạng](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-network-model).

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

### Hỗ trợ điều tiết lưu lượng (Support traffic shaping)

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

- Tìm hiểu thêm về [Mạng trong Cluster (Cluster Networking)](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- Tìm hiểu thêm về [Network Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- Tìm hiểu về [Xử lý sự cố các lỗi liên quan đến CNI plugin](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/troubleshooting-cni-plugin-related-errors/)
