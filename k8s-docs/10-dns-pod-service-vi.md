# DNS cho Service và Pod (DNS for Services and Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/>
>
> Workload của bạn có thể khám phá (discover) các Service bên trong cluster bằng DNS;
> trang này giải thích cách điều đó hoạt động.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 5](LO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), bài 4/16 · Kiểm chứng ở
Lab 5a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này là chỗ giải thích **vì sao gọi Service bằng tên lại chạy được**. Trọng tâm nằm ở ba
thứ: dạng FQDN, danh sách `search` trong `/etc/resolv.conf`, và `ndots:5`. Các mục về Windows
và về `dnsConfig` nâng cao có thể lướt.

**Phải hiểu ở lần đọc này:**

- Dạng tên chuẩn của Service là `my-svc.my-namespace.svc.cluster-domain.example`. Service
  thường phân giải thành **cluster IP**; headless Service phân giải thành **tập IP của tất cả
  Pod** mà nó chọn, và client phải tự dùng hết tập đó hoặc round-robin.
- Truy vấn DNS **không ghi namespace thì bị giới hạn trong namespace của Pod truy vấn**; muốn
  sang namespace khác phải chỉ định rõ (`data.prod`).
- `/etc/resolv.conf` do kubelet ghi cho từng Pod: `nameserver`, `search
  <namespace>.svc.cluster.local svc.cluster.local cluster.local` và `options ndots:5`. Chính
  danh sách `search` biến `data` thành `data.test.svc.cluster.local`.
- `dnsPolicy` mặc định là **`ClusterFirst`**, không phải `Default`. Phân biệt được bốn giá trị,
  đặc biệt `Default` nghĩa là **kế thừa cấu hình phân giải tên của node**, và Pod chạy
  `hostNetwork` phải đặt tường minh `ClusterFirstWithHostNet` nếu muốn dùng DNS của cluster.
- Bản ghi cho **từng Pod** chỉ có khi Pod đặt `hostname` và `subdomain`, với `subdomain` trùng
  tên một headless Service cùng namespace; thiếu `hostname` thì không có bản ghi riêng cho Pod,
  và Pod phải ở trạng thái ready (trừ khi Service đặt `publishNotReadyAddresses`).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Bản ghi SRV* | chỉ cần khi ứng dụng tự tra cứu số port qua DNS | không cần |
| Bản ghi A dạng `<pod-ip>.<namespace>.pod.<cluster-domain>` | là di sản của kube-DNS, hiếm dùng | không cần |
| *Trường setHostnameAsFQDN của Pod* | trùng nội dung với bài kế, đọc một lần là đủ | bài [57](57-pod-hostname-vi.md) |
| *Cấu hình DNS của Pod* (`dnsConfig`) và *Giới hạn danh sách domain tìm kiếm* | là tinh chỉnh nâng cao, chưa cần để gọi Service | không cần |
| *Phân giải DNS trên các node Windows* | lab không có node Windows | giai đoạn 15 |

---

Kubernetes tạo các bản ghi DNS (DNS record) cho các Service và Pod. Bạn có thể liên lạc với
các Service bằng những tên DNS nhất quán thay vì địa chỉ IP.

Kubernetes công bố thông tin về các Pod và Service, thông tin này được dùng
để lập trình DNS. kubelet cấu hình DNS của các Pod sao cho các container đang chạy
có thể tra cứu Service theo tên thay vì theo IP.

Các Service được định nghĩa trong cluster sẽ được gán tên DNS. Theo mặc định,
danh sách tìm kiếm DNS (DNS search list) của một Pod client bao gồm namespace của chính Pod đó và
domain mặc định của cluster.

### Namespace của Service (Namespaces of Services)

Một truy vấn DNS có thể trả về những kết quả khác nhau tùy theo namespace của Pod thực hiện
truy vấn. Các truy vấn DNS không chỉ định namespace bị giới hạn trong namespace của
Pod. Để truy cập Service ở các namespace khác, hãy chỉ định namespace trong truy vấn DNS.

Ví dụ, xét một Pod trong namespace `test`. Một Service `data` nằm trong
namespace `prod`.

Truy vấn cho `data` không trả về kết quả nào, vì nó dùng namespace `test` của Pod.

Truy vấn cho `data.prod` trả về kết quả mong muốn, vì nó chỉ định rõ
namespace.

Các truy vấn DNS có thể được mở rộng bằng file `/etc/resolv.conf` của Pod. kubelet
cấu hình file này cho từng Pod. Ví dụ, một truy vấn chỉ cho `data` có thể được
mở rộng thành `data.test.svc.cluster.local`. Các giá trị của tùy chọn `search`
được dùng để mở rộng truy vấn. Để tìm hiểu thêm về truy vấn DNS, xem
[trang hướng dẫn (man page) của `resolv.conf`](https://www.man7.org/linux/man-pages/man5/resolv.conf.5.html).

```
nameserver 10.32.0.10
search <namespace>.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Tóm lại, một Pod trong namespace _test_ có thể phân giải thành công cả
`data.prod` lẫn `data.prod.svc.cluster.local`.

### Các bản ghi DNS (DNS Records)

Những đối tượng nào được tạo bản ghi DNS?

1. Service
1. Pod

Các phần sau đây trình bày chi tiết những loại bản ghi DNS được hỗ trợ và cách bố trí (layout)
được hỗ trợ. Bất kỳ layout, tên, hay truy vấn nào khác dù tình cờ hoạt động được đều
được coi là chi tiết hiện thực (implementation detail) và có thể thay đổi mà không báo trước.
Để có đặc tả cập nhật hơn, xem
[Kubernetes DNS-Based Service Discovery](https://github.com/kubernetes/dns/blob/master/docs/specification.md).

## Service (Services)

### Bản ghi A/AAAA (A/AAAA records)

Các Service "bình thường" (không phải headless) được gán các bản ghi DNS A và/hoặc AAAA,
tùy theo (các) họ IP (IP family) của Service, với tên có dạng
`my-svc.my-namespace.svc.cluster-domain.example`. Tên này phân giải thành cluster IP
của Service.

[Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)
(không có cluster IP) cũng được gán các bản ghi DNS A và/hoặc AAAA,
với tên có dạng `my-svc.my-namespace.svc.cluster-domain.example`. Khác với Service
bình thường, tên này phân giải thành tập các IP của tất cả các Pod được Service đó chọn.
Client được kỳ vọng sẽ sử dụng cả tập IP này, hoặc dùng cách chọn luân phiên (round-robin)
tiêu chuẩn từ tập đó.

### Bản ghi SRV (SRV records)

Các bản ghi SRV được tạo cho các port có tên (named port) thuộc về các service bình thường hoặc
headless.

- Với mỗi named port, bản ghi SRV có dạng
  `_port-name._port-protocol.my-svc.my-namespace.svc.cluster-domain.example`.
- Đối với một Service thông thường, bản ghi này phân giải thành số port và tên miền:
  `my-svc.my-namespace.svc.cluster-domain.example`.
- Đối với một headless Service, bản ghi này phân giải thành nhiều câu trả lời, mỗi câu trả lời cho một Pod
  đứng sau Service, và chứa số port cùng tên miền của Pod
  có dạng `hostname.my-svc.my-namespace.svc.cluster-domain.example`.

## Pod (Pods)

### Bản ghi A/AAAA (A/AAAA records)

Các phiên bản Kube-DNS, trước khi [đặc tả DNS](https://github.com/kubernetes/dns/blob/master/docs/specification.md)
được hiện thực, có cách phân giải DNS như sau:

```
<pod-IPv4-address>.<namespace>.pod.<cluster-domain>
```

Ví dụ, nếu một Pod trong namespace `default` có địa chỉ IP là 172.17.0.3,
và tên miền của cluster của bạn là `cluster.local`, thì Pod đó có tên DNS là:

```
172-17-0-3.default.pod.cluster.local
```

Một số cơ chế DNS của cluster, như [CoreDNS](https://coredns.io/), cũng cung cấp bản ghi `A` cho:

```
<pod-ipv4-address>.<service-name>.<my-namespace>.svc.<cluster-domain.example>
```

Ví dụ, nếu một Pod trong namespace `cafe` có địa chỉ IP là 172.17.0.3,
là một endpoint của Service có tên `barista`, và tên miền của cluster của bạn là
`cluster.local`, thì Pod đó sẽ có bản ghi DNS `A` gắn với phạm vi service (service-scoped) như sau:

```
172-17-0-3.barista.cafe.svc.cluster.local
```

### Các trường hostname và subdomain của Pod {#pod-hostname-and-subdomain-field}

Hiện tại, khi một Pod được tạo, hostname của nó (khi quan sát từ bên trong Pod)
là giá trị `metadata.name` của Pod.

Spec của Pod có một trường tùy chọn `hostname`, có thể dùng để chỉ định một
hostname khác. Khi được chỉ định, nó được ưu tiên hơn tên của Pod để trở thành
hostname của Pod (một lần nữa, khi quan sát từ bên trong Pod). Ví dụ,
với một Pod có `spec.hostname` được đặt là `"my-host"`, Pod đó sẽ có
hostname là `"my-host"`.

Spec của Pod cũng có một trường tùy chọn `subdomain` có thể dùng để chỉ ra
rằng pod này thuộc về một nhóm con (sub-group) của namespace. Ví dụ, một Pod có `spec.hostname`
được đặt là `"foo"`, và `spec.subdomain` được đặt là `"bar"`, trong namespace `"my-namespace"`, sẽ
có hostname là `"foo"` và tên miền đầy đủ (fully qualified domain name - FQDN) là
`"foo.bar.my-namespace.svc.cluster.local"` (một lần nữa, khi quan sát từ bên trong
Pod).

Nếu tồn tại một headless Service trong cùng namespace với Pod, có
tên trùng với subdomain, thì DNS Server của cluster cũng trả về các bản ghi A và/hoặc AAAA
cho hostname đầy đủ (fully qualified hostname) của Pod.

Ví dụ:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: busybox-subdomain
spec:
  selector:
    name: busybox
  clusterIP: None
  ports:
  - name: foo # name không bắt buộc đối với Service chỉ có một port
    port: 1234
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
  labels:
    name: busybox
spec:
  hostname: busybox-1
  subdomain: busybox-subdomain
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    name: busybox
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox2
  labels:
    name: busybox
spec:
  hostname: busybox-2
  subdomain: busybox-subdomain
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    name: busybox
```

Với Service `"busybox-subdomain"` ở trên và các Pod đặt `spec.subdomain`
là `"busybox-subdomain"`, Pod đầu tiên sẽ thấy FQDN của chính nó là
`"busybox-1.busybox-subdomain.my-namespace.svc.cluster-domain.example"`. DNS phục vụ
các bản ghi A và/hoặc AAAA tại tên đó, trỏ đến IP của Pod. Cả hai Pod "`busybox1`" và
"`busybox2`" đều sẽ có bản ghi địa chỉ (address record) của riêng mình.

Một [EndpointSlice](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/) có thể chỉ định
hostname DNS cho bất kỳ địa chỉ endpoint nào, cùng với IP của nó.

> **Ghi chú:**
>
> Các bản ghi A và AAAA không được tạo cho tên Pod khi Pod thiếu `hostname`.
> Một Pod không có `hostname` nhưng có `subdomain` sẽ chỉ tạo
> bản ghi A hoặc AAAA cho headless Service (`busybox-subdomain.my-namespace.svc.cluster-domain.example`),
> trỏ đến các địa chỉ IP của các Pod. Ngoài ra, Pod cần ở trạng thái sẵn sàng (ready) thì mới có
> bản ghi, trừ khi `publishNotReadyAddresses=True` được thiết lập trên Service.

### Trường setHostnameAsFQDN của Pod {#pod-sethostnameasfqdn-field}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.22 [stable]`

Khi một Pod được cấu hình để có tên miền đầy đủ (FQDN), hostname của nó
là hostname ngắn. Ví dụ, nếu bạn có một Pod với tên miền
đầy đủ là `busybox-1.busybox-subdomain.my-namespace.svc.cluster-domain.example`,
thì theo mặc định lệnh `hostname` bên trong Pod đó trả về `busybox-1` còn lệnh
`hostname --fqdn` trả về FQDN.

Khi bạn thiết lập `setHostnameAsFQDN: true` trong spec của Pod, kubelet sẽ ghi FQDN của Pod
vào hostname cho namespace của Pod đó. Trong trường hợp này, cả `hostname` lẫn `hostname --fqdn`
đều trả về FQDN của Pod.

> **Ghi chú:**
>
> Trong Linux, trường hostname của kernel (trường `nodename` của `struct utsname`) bị giới hạn ở 64 ký tự.
>
> Nếu một Pod bật tính năng này và FQDN của nó dài hơn 64 ký tự, nó sẽ không khởi động được.
> Pod sẽ ở lại trạng thái `Pending` (`ContainerCreating` khi xem bằng `kubectl`) và sinh ra
> các sự kiện lỗi, chẳng hạn như "Failed to construct FQDN from Pod hostname and cluster domain,
> FQDN `long-FQDN` is too long (64 characters is the max, 70 characters requested)"
> (Không thể tạo FQDN từ hostname của Pod và domain của cluster, FQDN quá dài — tối đa 64 ký tự, yêu cầu 70 ký tự).
> Một cách để cải thiện trải nghiệm người dùng cho tình huống này là tạo một
> [admission webhook controller](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#what-are-admission-webhooks)
> để kiểm soát độ dài FQDN khi người dùng tạo các đối tượng cấp cao, ví dụ như Deployment.

### Chính sách DNS của Pod (Pod's DNS Policy)

Chính sách DNS có thể được thiết lập trên từng Pod. Hiện tại Kubernetes hỗ trợ các
chính sách DNS theo Pod sau đây. Các chính sách này được chỉ định trong
trường `dnsPolicy` của Spec Pod.

- "`Default`": Pod kế thừa cấu hình phân giải tên từ node
  mà Pod chạy trên đó.
  Xem [thảo luận liên quan](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers)
  để biết thêm chi tiết.
- "`ClusterFirst`": Bất kỳ truy vấn DNS nào không khớp với hậu tố domain
  của cluster đã cấu hình, chẳng hạn "`www.kubernetes.io`", sẽ được DNS server chuyển tiếp đến một
  nameserver thượng nguồn (upstream). Quản trị viên cluster có thể cấu hình thêm
  các stub-domain và các DNS server upstream.
  Xem [thảo luận liên quan](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers)
  để biết chi tiết về cách các truy vấn DNS được xử lý trong những trường hợp đó.
- "`ClusterFirstWithHostNet`": Đối với các Pod chạy với hostNetwork, bạn nên
  đặt chính sách DNS của chúng một cách tường minh là "`ClusterFirstWithHostNet`". Nếu không, các Pod
  chạy với hostNetwork và `"ClusterFirst"` sẽ rơi về (fallback) hành vi
  của chính sách `"Default"`.

  > **Ghi chú:**
  >
  > Điều này không được hỗ trợ trên Windows. Xem [bên dưới](#dns-windows) để biết chi tiết.

- "`None`": Cho phép một Pod bỏ qua các thiết lập DNS từ môi trường
  Kubernetes. Mọi thiết lập DNS phải được cung cấp thông qua
  trường `dnsConfig` trong Spec của Pod.
  Xem tiểu mục [Cấu hình DNS của Pod](#pod-dns-config) bên dưới.

> **Ghi chú:**
>
> "Default" không phải là chính sách DNS mặc định. Nếu `dnsPolicy` không được
> chỉ định tường minh, thì "ClusterFirst" sẽ được sử dụng.

Ví dụ dưới đây cho thấy một Pod có chính sách DNS được đặt là
"`ClusterFirstWithHostNet`" vì nó có `hostNetwork` được đặt là `true`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
```

### Cấu hình DNS của Pod (Pod's DNS Config) {#pod-dns-config}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.14 [stable]`

Cấu hình DNS của Pod cho phép người dùng kiểm soát nhiều hơn các thiết lập DNS cho một Pod.

Trường `dnsConfig` là tùy chọn và nó có thể hoạt động với mọi thiết lập `dnsPolicy`.
Tuy nhiên, khi `dnsPolicy` của một Pod được đặt là "`None`", trường `dnsConfig` bắt buộc
phải được chỉ định.

Dưới đây là các thuộc tính người dùng có thể chỉ định trong trường `dnsConfig`:

- `nameservers`: danh sách các địa chỉ IP sẽ được dùng làm DNS server cho
  Pod. Có thể chỉ định tối đa 3 địa chỉ IP. Khi `dnsPolicy` của Pod
  được đặt là "`None`", danh sách này phải chứa ít nhất một địa chỉ IP, ngược lại
  thuộc tính này là tùy chọn.
  Các server được liệt kê sẽ được gộp với các nameserver cơ sở được sinh ra từ
  chính sách DNS đã chỉ định, các địa chỉ trùng lặp sẽ bị loại bỏ.
- `searches`: danh sách các domain tìm kiếm DNS (DNS search domain) dùng để tra cứu hostname trong Pod.
  Thuộc tính này là tùy chọn. Khi được chỉ định, danh sách này sẽ được gộp
  vào các tên domain tìm kiếm cơ sở được sinh ra từ chính sách DNS đã chọn.
  Các tên domain trùng lặp sẽ bị loại bỏ.
  Kubernetes cho phép tối đa 32 search domain.
- `options`: danh sách tùy chọn gồm các đối tượng, trong đó mỗi đối tượng có thể có thuộc tính `name`
  (bắt buộc) và thuộc tính `value` (tùy chọn). Nội dung trong
  thuộc tính này sẽ được gộp với các option được sinh ra từ chính sách DNS đã chỉ định.
  Các mục trùng lặp sẽ bị loại bỏ.

Sau đây là một ví dụ Pod với các thiết lập DNS tùy chỉnh:

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 192.0.2.1 # đây là một ví dụ
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

Khi Pod trên được tạo, container `test` nhận được nội dung sau
trong file `/etc/resolv.conf` của nó:

```
nameserver 192.0.2.1
search ns1.svc.cluster-domain.example my.dns.search.suffix
options ndots:2 edns0
```

Với thiết lập IPv6, search path và nameserver nên được thiết lập như sau:

```shell
kubectl exec -it dns-example -- cat /etc/resolv.conf
```

Kết quả đầu ra tương tự như sau:

```
nameserver 2001:db8:30::a
search default.svc.cluster-domain.example svc.cluster-domain.example cluster-domain.example
options ndots:5
```

## Giới hạn danh sách domain tìm kiếm DNS (DNS search domain list limits)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.28 [stable]`

Bản thân Kubernetes không giới hạn Cấu hình DNS cho đến khi độ dài danh sách
search domain vượt quá 32, hoặc tổng độ dài của tất cả các search domain vượt quá 2048.
Giới hạn này áp dụng lần lượt cho file cấu hình resolver của node, Cấu hình DNS
của Pod, và Cấu hình DNS sau khi gộp.

> **Ghi chú:**
>
> Một số container runtime ở các phiên bản cũ hơn có thể có giới hạn riêng về
> số lượng DNS search domain. Tùy theo môi trường container runtime,
> các pod có số lượng lớn DNS search domain có thể bị kẹt ở
> trạng thái pending.
>
> Được biết containerd v1.5.5 trở về trước và CRI-O v1.21 trở về trước gặp
> vấn đề này.

## Phân giải DNS trên các node Windows {#dns-windows}

- `ClusterFirstWithHostNet` không được hỗ trợ cho các Pod chạy trên node Windows.
  Windows coi mọi tên có dấu `.` là một FQDN và bỏ qua bước phân giải FQDN.
- Trên Windows, có nhiều trình phân giải DNS (DNS resolver) có thể được sử dụng. Vì chúng có
  hành vi hơi khác nhau, khuyến nghị dùng cmdlet powershell
  [`Resolve-DNSName`](https://docs.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
  để phân giải các truy vấn tên.
- Trên Linux, bạn có một danh sách hậu tố DNS (DNS suffix list), được dùng sau khi việc phân giải một tên
  dưới dạng đầy đủ (fully qualified) thất bại.
  Trên Windows, bạn chỉ có thể có 1 hậu tố DNS, đó là hậu tố DNS gắn với
  namespace của Pod đó (ví dụ: `mydns.svc.cluster.local`). Windows có thể phân giải các FQDN, Service,
  hoặc tên mạng phân giải được với hậu tố duy nhất này. Ví dụ, một Pod được tạo
  trong namespace `default` sẽ có hậu tố DNS là `default.svc.cluster.local`.
  Bên trong một Pod Windows, bạn có thể phân giải cả `kubernetes.default.svc.cluster.local`
  lẫn `kubernetes`, nhưng không thể phân giải các tên đủ điều kiện một phần (partially qualified name) như (`kubernetes.default` hoặc
  `kubernetes.default.svc`).

## Tiếp theo (What's next)

Để có hướng dẫn về việc quản trị các cấu hình DNS, xem
[Configure DNS Service](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5:

1. Trên cluster lab, một Pod chạy ở namespace `default` và có một Service `web` nằm ở namespace
   `prod`. Vì sao `curl web` không tới được, còn `curl web.prod` thì tới? Truy vấn `web.prod`
   được mở rộng thành những tên nào trước khi ra kết quả?
2. `dnsPolicy: Default` có phải giá trị mặc định của `dnsPolicy` không? Nó thực sự làm gì?
3. Cùng một truy vấn `my-svc.my-ns.svc.cluster.local` trả về gì khi `my-svc` là Service thường,
   và trả về gì khi nó là headless Service?
4. Bạn đặt `spec.subdomain: busybox-subdomain` cho hai Pod nhưng quên đặt `spec.hostname`. DNS
   có bản ghi riêng cho từng Pod không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Vì truy vấn không chỉ định namespace bị giới hạn trong namespace của Pod.** `web` được
   danh sách `search` mở rộng thành `web.default.svc.cluster.local` — không tồn tại. Còn
   `web.prod` lần lượt được thử với các hậu tố trong `search`: `web.prod.default.svc.cluster.local`
   (trượt), rồi `web.prod.svc.cluster.local` — **khớp**, vì đó đúng là dạng
   `my-svc.my-namespace.svc.cluster-domain.example`. Bài tóm lại đúng điều này: một Pod trong
   namespace `test` phân giải được cả `data.prod` lẫn `data.prod.svc.cluster.local`.
2. **Không.** Bài ghi rõ: "Default" **không phải** chính sách DNS mặc định; nếu `dnsPolicy`
   không được chỉ định tường minh thì **`ClusterFirst`** được dùng. `Default` nghĩa là Pod **kế
   thừa cấu hình phân giải tên từ node** mà nó chạy trên đó — tức là bỏ qua DNS của cluster.
3. Service thường: **một bản ghi A/AAAA trỏ tới cluster IP** của Service. Headless Service:
   cùng tên đó nhưng **phân giải thành tập IP của tất cả các Pod được Service chọn**, và client
   được kỳ vọng dùng cả tập hoặc round-robin trên tập đó.
4. **Không.** Bài ghi rõ: bản ghi A và AAAA **không được tạo cho tên Pod khi Pod thiếu
   `hostname`**. Pod có `subdomain` mà không có `hostname` chỉ góp mặt trong bản ghi của chính
   headless Service (`busybox-subdomain.my-namespace.svc.cluster-domain.example`), trỏ tới IP
   các Pod. Ngoài ra Pod phải ở trạng thái ready mới có bản ghi, trừ khi Service đặt
   `publishNotReadyAddresses=True`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
