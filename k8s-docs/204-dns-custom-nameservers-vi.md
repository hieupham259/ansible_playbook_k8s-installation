# Tùy chỉnh DNS Service (Customizing DNS Service)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/>
>
> Trang này giải thích cách cấu hình các Pod DNS và tùy chỉnh quá trình phân giải DNS
> trong cluster của bạn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy),
bài 1/14 · thực hành trực tiếp trên cluster VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md).

Bài [10](10-dns-pod-service-vi.md) đã dạy DNS *từ phía Pod* (tên miền nào tra được, `search`
domain, `ndots`). Bài này là mặt còn lại: *từ phía người vận hành*, CoreDNS được cấu hình ở
đâu và sửa thế nào. Toàn bộ bài xoay quanh một đối tượng duy nhất — ConfigMap `coredns` trong
namespace `kube-system` chứa Corefile.

**Phải hiểu ở lần đọc này:**

- Chuỗi cấu hình DNS của cluster: CoreDNS chạy như một Deployment, được expose qua Service
  tên `kube-dns` (giữ tên cũ để tương thích) với IP tĩnh; kubelet truyền IP đó cho từng
  container qua flag `--cluster-dns` và truyền domain nội bộ qua `--cluster-domain`.
- Pod có `dnsPolicy: default` thừa hưởng cấu hình phân giải tên của node; flag `--resolv-conf`
  của kubelet quyết định file nguồn cho việc thừa hưởng đó (đặt `""` để chặn hẳn).
- Corefile mặc định gồm những plugin nào và mỗi plugin làm gì — đặc biệt `kubernetes` (trả lời
  truy vấn theo IP của Service/Pod), `forward` (chuyển tiếp truy vấn ngoài cluster domain),
  `cache`, `loop` và `reload` (tự nạp lại Corefile sau khi sửa ConfigMap, chờ khoảng 2 phút).
- Cách thêm stub-domain: viết thêm một block server riêng trong Corefile (ví dụ
  `consul.local:53`) với `forward` trỏ tới nameserver của domain đó.
- Cách ép mọi truy vấn ngoài cluster đi qua một upstream nameserver cụ thể: đổi
  `forward . /etc/resolv.conf` thành `forward . <IP>`; lưu ý CoreDNS không nhận FQDN làm
  nameserver trong cấu hình stub-domain.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chi tiết các tùy chọn của plugin `kubernetes` (`pods insecure`/`verified`/`disabled`, `ttl`) | chỉ cần khi bạn thật sự dùng bản ghi DNS theo IP của Pod — trường hợp hiếm trong vận hành thường ngày | tài liệu [plugin kubernetes của CoreDNS](https://coredns.io/plugins/kubernetes/) khi có nhu cầu |

---

Trang này giải thích cách cấu hình các Pod DNS và tùy chỉnh quá trình phân giải DNS
trong cluster của bạn.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Cluster của bạn phải đang chạy add-on CoreDNS.

Kubernetes server của bạn phải ở phiên bản v1.12 hoặc mới hơn. Để kiểm tra phiên bản,
hãy chạy `kubectl version`.

## Giới thiệu (Introduction)

DNS là một service tích hợp sẵn của Kubernetes, được khởi chạy tự động bằng _trình quản lý
addon_ (addon manager) — xem [cluster add-on](https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/addon-manager/README.md).

> **Ghi chú:** Service của CoreDNS có tên `kube-dns` trong trường `metadata.name`.
> Mục đích là để bảo đảm khả năng tương tác cao hơn với các workload vốn dựa vào tên Service
> `kube-dns` cũ để phân giải các địa chỉ nội bộ trong cluster. Việc dùng một Service tên
> `kube-dns` giúp trừu tượng hóa chi tiết triển khai — cụ thể là DNS provider nào đang chạy
> đằng sau cái tên chung đó.

Nếu bạn chạy CoreDNS dưới dạng một Deployment, nó thường sẽ được expose như một Service
Kubernetes với địa chỉ IP tĩnh. Kubelet truyền thông tin về DNS resolver cho từng container
bằng flag `--cluster-dns=<dns-service-ip>`.

Tên DNS cũng cần domain. Bạn cấu hình domain cục bộ trong kubelet bằng flag
`--cluster-domain=<default-local-domain>`.

DNS server hỗ trợ tra cứu xuôi (bản ghi A và AAAA), tra cứu port (bản ghi SRV), tra cứu
ngược địa chỉ IP (bản ghi PTR), và nhiều loại khác. Để biết thêm thông tin, xem
[DNS cho Service và Pod](10-dns-pod-service-vi.md).

Nếu `dnsPolicy` của một Pod được đặt là `default`, Pod đó thừa hưởng cấu hình phân giải tên
từ node mà Pod chạy trên đó. Việc phân giải DNS của Pod sẽ hoạt động giống như trên node.
Nhưng hãy xem thêm [Các vấn đề đã biết](205-dns-debugging-resolution-vi.md#các-vấn-đề-đã-biết-known-issues).

Nếu bạn không muốn như vậy, hoặc muốn một cấu hình DNS khác cho các Pod, bạn có thể dùng flag
`--resolv-conf` của kubelet. Đặt flag này thành "" để ngăn Pod thừa hưởng cấu hình DNS. Đặt nó
thành một đường dẫn file hợp lệ để chỉ định một file khác `/etc/resolv.conf` làm nguồn thừa
hưởng DNS.

## CoreDNS

CoreDNS là một DNS server authoritative đa dụng, có thể đảm nhiệm vai trò DNS của cluster,
tuân theo [đặc tả DNS](https://github.com/kubernetes/dns/blob/master/docs/specification.md).

### Các tùy chọn ConfigMap của CoreDNS (CoreDNS ConfigMap options)

CoreDNS là một DNS server dạng module và có thể cắm thêm (pluggable), với các plugin bổ sung
những chức năng mới. CoreDNS server có thể được cấu hình thông qua việc duy trì một
[Corefile](https://coredns.io/2017/07/23/corefile-explained/) — file cấu hình của CoreDNS.
Với vai trò quản trị viên cluster, bạn có thể sửa ConfigMap chứa Corefile của CoreDNS để
thay đổi cách hoạt động của cơ chế khám phá service qua DNS (DNS service discovery) trong
cluster đó.

Trong Kubernetes, CoreDNS được cài đặt với cấu hình Corefile mặc định như sau:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

Cấu hình Corefile này bao gồm các [plugin](https://coredns.io/plugins/) sau của CoreDNS:

* [errors](https://coredns.io/plugins/errors/): Lỗi được ghi log ra stdout.
* [health](https://coredns.io/plugins/health/): Tình trạng sức khỏe của CoreDNS được báo cáo
  tại `http://localhost:8080/health`. Trong cú pháp mở rộng này, `lameduck` sẽ đánh dấu
  process là unhealthy rồi chờ 5 giây trước khi process bị tắt.
* [ready](https://coredns.io/plugins/ready/): Một endpoint HTTP trên port 8181 sẽ trả về
  200 OK khi tất cả các plugin có khả năng báo hiệu trạng thái sẵn sàng đều đã báo hiệu.
* [kubernetes](https://coredns.io/plugins/kubernetes/): CoreDNS sẽ trả lời các truy vấn DNS
  dựa trên IP của các Service và Pod. Bạn có thể xem
  [thêm chi tiết](https://coredns.io/plugins/kubernetes/) về plugin này trên website của
  CoreDNS.
  - `ttl` cho phép bạn đặt TTL tùy chỉnh cho các phản hồi. Giá trị mặc định là 5 giây.
    TTL nhỏ nhất được phép là 0 giây, và lớn nhất bị giới hạn ở 3600 giây.
    Đặt TTL bằng 0 sẽ ngăn các bản ghi bị cache.
  - Tùy chọn `pods insecure` được cung cấp để tương thích ngược với `kube-dns`.
  - Bạn có thể dùng tùy chọn `pods verified`, tùy chọn này chỉ trả về bản ghi A nếu tồn tại
    một pod trong cùng namespace có IP khớp.
  - Tùy chọn `pods disabled` có thể dùng nếu bạn không sử dụng bản ghi của pod.
* [prometheus](https://coredns.io/plugins/metrics/): Các metric của CoreDNS có sẵn tại
  `http://localhost:9153/metrics` theo định dạng [Prometheus](https://prometheus.io/)
  (còn gọi là OpenMetrics).
* [forward](https://coredns.io/plugins/forward/): Mọi truy vấn không thuộc cluster domain
  của Kubernetes được chuyển tiếp tới các resolver định nghĩa sẵn (/etc/resolv.conf).
* [cache](https://coredns.io/plugins/cache/): Bật cache ở phía frontend.
* [loop](https://coredns.io/plugins/loop/): Phát hiện các vòng lặp chuyển tiếp
  (forwarding loop) đơn giản và dừng process CoreDNS nếu tìm thấy vòng lặp.
* [reload](https://coredns.io/plugins/reload): Cho phép tự động nạp lại Corefile khi file
  thay đổi. Sau khi bạn sửa cấu hình trong ConfigMap, hãy chờ hai phút để thay đổi có
  hiệu lực.
* [loadbalance](https://coredns.io/plugins/loadbalance): Đây là một bộ cân bằng tải
  (load balancer) DNS kiểu round-robin, hoán đổi ngẫu nhiên thứ tự các bản ghi A, AAAA và
  MX trong câu trả lời.

Bạn có thể thay đổi hành vi mặc định của CoreDNS bằng cách sửa ConfigMap.

### Cấu hình stub-domain và upstream nameserver bằng CoreDNS (Configuration of Stub-domain and upstream nameserver using CoreDNS)

CoreDNS có khả năng cấu hình stub-domain và các upstream nameserver bằng
[plugin forward](https://coredns.io/plugins/forward/).

#### Ví dụ (Example)

Giả sử người vận hành cluster có một domain server [Consul](https://www.consul.io/) đặt tại
"10.150.0.1", và mọi tên của Consul đều có hậu tố ".consul.local". Để cấu hình điều này trong
CoreDNS, quản trị viên cluster tạo đoạn (stanza) sau trong ConfigMap của CoreDNS.

```
consul.local:53 {
    errors
    cache 30
    forward . 10.150.0.1
}
```

Để ép mọi truy vấn DNS không thuộc cluster đi qua một nameserver cụ thể tại 172.16.0.1,
hãy trỏ `forward` tới nameserver đó thay vì `/etc/resolv.conf`

```
forward .  172.16.0.1
```

ConfigMap cuối cùng, cùng với cấu hình `Corefile` mặc định, trông như sau:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . 172.16.0.1
        cache 30
        loop
        reload
        loadbalance
    }
    consul.local:53 {
        errors
        cache 30
        forward . 10.150.0.1
    }
```

> **Ghi chú:** CoreDNS không hỗ trợ FQDN cho stub-domain và nameserver (ví dụ: "ns.foo.com").
> Trong quá trình chuyển đổi cấu hình, mọi nameserver dạng FQDN sẽ bị bỏ qua khỏi cấu hình
> CoreDNS.

## Tiếp theo (What's next)

- Đọc [Gỡ lỗi phân giải DNS](205-dns-debugging-resolution-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu dưới đây mà không nhìn lại bài là đủ cho lần đọc ở checkpoint này.

1. Pod trong cluster lấy địa chỉ IP của DNS resolver từ đâu — thành phần nào đưa nó vào, và
   giá trị đó thực chất là IP của đối tượng gì trong cluster?
2. Cluster của bạn chạy CoreDNS, nhưng Service trong `kube-system` lại tên `kube-dns`. Đây có
   phải cấu hình sai không? Vì sao lại đặt tên như vậy?
3. Bạn cần mọi tên có hậu tố `.consul.local` được phân giải bởi server 10.150.0.1, còn mọi
   truy vấn ngoài cluster khác đi qua 172.16.0.1. Bạn sửa đối tượng nào, và Corefile thay đổi
   ở những chỗ nào?
4. Sau khi `kubectl edit` ConfigMap `coredns`, bạn thử `nslookup` ngay và thấy hành vi chưa
   đổi. Có cần restart các Pod CoreDNS không? Cơ chế nào của Corefile mặc định xử lý việc này
   và bạn phải chờ bao lâu?
5. Trên hai worker của cluster lab, một Pod có `dnsPolicy: default` và một Pod có `dnsPolicy`
   mặc định (không khai báo). Pod nào dùng CoreDNS, Pod nào dùng cấu hình phân giải của node?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Kubelet đưa vào từng container qua flag `--cluster-dns=<dns-service-ip>`.** Giá trị đó là
   **IP tĩnh của Service `kube-dns`** — Service đứng trước Deployment CoreDNS. Domain nội bộ
   thì do flag `--cluster-domain` của kubelet quyết định.
2. **Không phải cấu hình sai — đây là chủ đích.** Service của CoreDNS được đặt tên `kube-dns`
   trong `metadata.name` để tương thích với các workload vốn dựa vào tên Service `kube-dns`
   cũ; cái tên chung này trừu tượng hóa việc DNS provider nào đang chạy phía sau.
3. Sửa **ConfigMap `coredns` trong namespace `kube-system`** (chứa Corefile). Hai thay đổi:
   thêm một block server riêng `consul.local:53 { errors / cache 30 / forward . 10.150.0.1 }`
   cho stub-domain, và trong block `.:53` đổi `forward . /etc/resolv.conf` thành
   `forward . 172.16.0.1`.
4. **Không cần restart.** Corefile mặc định có plugin `reload`, tự nạp lại cấu hình khi
   Corefile thay đổi; bài dặn **chờ khoảng hai phút** sau khi sửa ConfigMap để thay đổi có
   hiệu lực. Trực giác "sửa ConfigMap là ăn ngay" hoặc "phải xóa Pod" đều không đúng với
   cấu hình mặc định này.
5. Câu bẫy nằm ở chữ `default`: Pod có `dnsPolicy: default` **thừa hưởng cấu hình phân giải
   tên của node** (không dùng CoreDNS); còn Pod *không khai báo* `dnsPolicy` nhận policy mặc
   định thực sự là `ClusterFirst` — tức **dùng CoreDNS qua Service `kube-dns`** (điểm này bài
   [10](10-dns-pod-service-vi.md) đã trình bày). Giá trị `default` không phải là mặc định.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
