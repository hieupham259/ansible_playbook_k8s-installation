# Hostname của Pod (Pod Hostname)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/pod-hostname/>

Trang này giải thích cách đặt hostname cho một Pod, các tác dụng phụ tiềm ẩn sau khi cấu
hình, và cơ chế hoạt động bên dưới.

## Hostname mặc định của Pod (Default Pod hostname)

Khi một Pod được tạo, hostname của nó (khi quan sát từ bên trong Pod) được suy ra từ giá
trị `metadata.name` của Pod. Cả hostname lẫn tên miền đầy đủ (fully qualified domain
name - FQDN) tương ứng đều được đặt thành giá trị `metadata.name` (từ góc nhìn của Pod)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-1
spec:
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    name: busybox
```

Pod được tạo bởi manifest này sẽ có hostname và tên miền đầy đủ (FQDN) được đặt thành
`busybox-1`.

## Hostname với các trường hostname và subdomain của Pod (Hostname with pod's hostname and subdomain fields)

Spec của Pod bao gồm một trường `hostname` tùy chọn. Khi được đặt, giá trị này được ưu
tiên hơn `metadata.name` của Pod để làm hostname (khi quan sát từ bên trong Pod). Ví dụ,
một Pod có `spec.hostname` được đặt thành `my-host` sẽ có hostname là `my-host`.

Spec của Pod cũng bao gồm một trường `subdomain` tùy chọn, cho biết Pod thuộc về một
subdomain (tên miền con) trong namespace của nó. Nếu một Pod có `spec.hostname` được đặt
thành "foo" và `spec.subdomain` được đặt thành "bar" trong namespace `my-namespace`,
hostname của nó sẽ là `foo` và tên miền đầy đủ (FQDN) của nó sẽ là
`foo.bar.my-namespace.svc.cluster-domain.example` (khi quan sát từ bên trong Pod).

Khi cả hostname lẫn subdomain đều được đặt, DNS server của cluster sẽ tạo các bản ghi A
và/hoặc AAAA dựa trên các trường này. Tham khảo:
[Các trường hostname và subdomain của Pod](./10-dns-pod-service-vi.md#pod-hostname-and-subdomain-field).

## Hostname với trường setHostnameAsFQDN của Pod (Hostname with pod's setHostnameAsFQDN fields)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.22 [stable]`

Khi một Pod được cấu hình để có tên miền đầy đủ (FQDN), hostname của nó là hostname ngắn.
Ví dụ, nếu bạn có một Pod với tên miền đầy đủ là
`busybox-1.busybox-subdomain.my-namespace.svc.cluster-domain.example`, thì mặc định lệnh
`hostname` bên trong Pod đó trả về `busybox-1` và lệnh `hostname --fqdn` trả về FQDN.

Khi cả `setHostnameAsFQDN: true` lẫn trường subdomain đều được đặt trong spec của Pod,
kubelet sẽ ghi FQDN của Pod vào hostname cho namespace của Pod đó. Trong trường hợp này,
cả `hostname` lẫn `hostname --fqdn` đều trả về FQDN của Pod.

FQDN của Pod được cấu thành theo cùng cách như đã định nghĩa trước đó. Nó bao gồm trường
`spec.hostname` của Pod (nếu được chỉ định) hoặc `metadata.name`, cùng với
`spec.subdomain`, tên `namespace`, và hậu tố tên miền của cluster.

> **Ghi chú:** Trong Linux, trường hostname của kernel (trường `nodename` của
> `struct utsname`) bị giới hạn ở 64 ký tự.
>
> Nếu một Pod bật tính năng này và FQDN của nó dài hơn 64 ký tự, Pod sẽ không khởi động
> được. Pod sẽ ở trạng thái `Pending` (hiển thị là `ContainerCreating` khi xem bằng
> `kubectl`) và sinh ra các event lỗi, chẳng hạn "Failed to construct FQDN from Pod
> hostname and cluster domain".
>
> Điều này có nghĩa là khi dùng trường này, bạn phải bảo đảm tổng độ dài của các trường
> `metadata.name` (hoặc `spec.hostname`) và `spec.subdomain` của Pod tạo ra một FQDN
> không vượt quá 64 ký tự.

## Hostname với hostnameOverride của Pod (Hostname with pod's hostnameOverride)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [beta]`

Việc đặt một giá trị cho `hostnameOverride` trong spec của Pod khiến kubelet đặt vô điều
kiện cả hostname lẫn tên miền đầy đủ (FQDN) của Pod thành giá trị `hostnameOverride`.

Trường `hostnameOverride` có giới hạn độ dài là 64 ký tự và phải tuân theo chuẩn tên miền
con DNS (DNS subdomain names) được định nghĩa trong
[RFC 1123](https://datatracker.ietf.org/doc/html/rfc1123).

Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-2-busybox-example-domain
spec:
  hostnameOverride: busybox-2.busybox.example.domain
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    name: busybox
```

> **Ghi chú:** Điều này chỉ ảnh hưởng đến hostname bên trong Pod; nó không ảnh hưởng đến
> các bản ghi A hoặc AAAA của Pod trong DNS server của cluster.

Nếu `hostnameOverride` được đặt cùng với các trường `hostname` và `subdomain`:
* Hostname bên trong Pod bị ghi đè thành giá trị `hostnameOverride`.

* Các bản ghi A và/hoặc AAAA của Pod trong DNS server của cluster vẫn được sinh ra dựa
  trên các trường `hostname` và `subdomain`.

Lưu ý: Nếu `hostnameOverride` được đặt, bạn không thể đồng thời đặt các trường
`hostNetwork` và `setHostnameAsFQDN`. API server sẽ từ chối một cách tường minh mọi
request tạo (create) có ý định dùng tổ hợp này.

Để biết chi tiết về hành vi khi `hostnameOverride` được đặt kết hợp với các trường khác
(hostname, subdomain, setHostnameAsFQDN, hostNetwork), hãy xem bảng trong
[chi tiết thiết kế của KEP-4762](https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/4762-allow-arbitrary-fqdn-as-pod-hostname/README.md#design-details).
