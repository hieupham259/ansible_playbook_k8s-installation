# Sử dụng NodeLocal DNSCache trong Cluster Kubernetes (Using NodeLocal DNSCache in Kubernetes Clusters)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.18 [stable]`

Trang này cung cấp cái nhìn tổng quan về tính năng NodeLocal DNSCache trong Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, nhập `kubectl version`.

## Giới thiệu (Introduction)

NodeLocal DNSCache cải thiện hiệu năng DNS của Cluster bằng cách chạy một agent cache DNS
trên các node của cluster dưới dạng DaemonSet. Trong kiến trúc hiện nay, các Pod ở chế độ DNS
'ClusterFirst' truy vấn tới `serviceIP` của kube-dns cho các truy vấn DNS. Địa chỉ này được
chuyển đổi thành một endpoint của kube-dns/CoreDNS thông qua các quy tắc iptables do kube-proxy
thêm vào. Với kiến trúc mới này, các Pod sẽ truy vấn tới agent cache DNS
đang chạy trên cùng node, nhờ đó tránh được các quy tắc iptables DNAT và việc theo dõi kết nối
(connection tracking). Agent cache cục bộ sẽ truy vấn service kube-dns đối với các trường hợp
cache miss của hostname trong cluster (mặc định là hậu tố "`cluster.local`").

## Động lực (Motivation)

* Với kiến trúc DNS hiện tại, các Pod có QPS DNS cao nhất có thể phải truy vấn tới
  một node khác, nếu không có instance kube-dns/CoreDNS cục bộ.
  Có một cache cục bộ sẽ giúp cải thiện độ trễ (latency) trong những tình huống như vậy.

* Bỏ qua iptables DNAT và connection tracking sẽ giúp giảm
  [các race condition của conntrack](https://github.com/kubernetes/kubernetes/issues/56903)
  và tránh việc các entry DNS UDP lấp đầy bảng conntrack.

* Kết nối từ agent cache cục bộ tới service kube-dns có thể được nâng cấp lên TCP.
  Các entry conntrack TCP sẽ được xóa khi kết nối đóng, trái ngược với
  các entry UDP phải chờ hết thời gian chờ (timeout)
  ([mặc định](https://www.kernel.org/doc/Documentation/networking/nf_conntrack-sysctl.txt)
  `nf_conntrack_udp_timeout` là 30 giây)

* Nâng cấp truy vấn DNS từ UDP lên TCP sẽ giảm độ trễ đuôi (tail latency) gây ra bởi
  các gói UDP bị rơi và DNS timeout thường lên tới 30s (3 lần thử lại + timeout 10s).
  Vì cache nodelocal lắng nghe các truy vấn DNS UDP, các ứng dụng không cần phải thay đổi.

* Có metrics và khả năng quan sát các yêu cầu DNS ở cấp độ node.

* Negative caching (cache kết quả phủ định) có thể được bật lại, nhờ đó giảm số lượng truy vấn
  tới service kube-dns.

## Sơ đồ kiến trúc (Architecture Diagram)

Đây là đường đi của các truy vấn DNS sau khi NodeLocal DNSCache được bật:

![Luồng NodeLocal DNSCache](https://kubernetes.io/images/docs/nodelocaldns.svg)

*Hình ảnh này cho thấy cách NodeLocal DNSCache xử lý các truy vấn DNS.*

## Cấu hình (Configuration)

> **Ghi chú:**
> Địa chỉ IP lắng nghe cục bộ của NodeLocal DNSCache có thể là bất kỳ địa chỉ nào
> được đảm bảo không xung đột với bất kỳ IP nào đang tồn tại trong cluster của bạn.
> Khuyến nghị dùng một địa chỉ có phạm vi cục bộ, ví dụ,
> từ dải 'link-local' '169.254.0.0/16' đối với IPv4 hoặc từ dải
> 'Unique Local Address' của IPv6 'fd00::/8'.

Tính năng này có thể được bật bằng các bước sau:

* Chuẩn bị một manifest tương tự như mẫu
  [`nodelocaldns.yaml`](https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml)
  và lưu lại với tên `nodelocaldns.yaml`.

* Nếu dùng IPv6, file cấu hình CoreDNS cần bọc tất cả các địa chỉ IPv6
  trong cặp ngoặc vuông khi dùng ở định dạng 'IP:Port'.
  Nếu bạn đang dùng manifest mẫu ở bước trên, điều này đòi hỏi bạn sửa
  [dòng cấu hình L70](https://github.com/kubernetes/kubernetes/blob/b2ecd1b3a3192fbbe2b9e348e095326f51dc43dd/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml#L70)
  như sau: "`health [__PILLAR__LOCAL__DNS__]:8080`"

* Thay thế các biến trong manifest bằng các giá trị đúng:

  ```shell
  kubedns=`kubectl get svc kube-dns -n kube-system -o jsonpath={.spec.clusterIP}`
  domain=<cluster-domain>
  localdns=<node-local-address>
  ```

  `<cluster-domain>` mặc định là "`cluster.local`". `<node-local-address>` là
  địa chỉ IP lắng nghe cục bộ được chọn cho NodeLocal DNSCache.

  * Nếu kube-proxy đang chạy ở chế độ IPTABLES:

    ``` bash
    sed -i "s/__PILLAR__LOCAL__DNS__/$localdns/g; s/__PILLAR__DNS__DOMAIN__/$domain/g; s/__PILLAR__DNS__SERVER__/$kubedns/g" nodelocaldns.yaml
    ```

    `__PILLAR__CLUSTER__DNS__` và `__PILLAR__UPSTREAM__SERVERS__` sẽ được điền bởi
    các pod `node-local-dns`.
    Ở chế độ này, các pod `node-local-dns` lắng nghe trên cả IP của service kube-dns
    lẫn `<node-local-address>`, vì vậy các pod có thể tra cứu bản ghi DNS bằng một trong hai
    địa chỉ IP.

  * Nếu kube-proxy đang chạy ở chế độ IPVS:

    ``` bash
    sed -i "s/__PILLAR__LOCAL__DNS__/$localdns/g; s/__PILLAR__DNS__DOMAIN__/$domain/g; s/,__PILLAR__DNS__SERVER__//g; s/__PILLAR__CLUSTER__DNS__/$kubedns/g" nodelocaldns.yaml
    ```

    Ở chế độ này, các pod `node-local-dns` chỉ lắng nghe trên `<node-local-address>`.
    Interface của `node-local-dns` không thể bind cluster IP của kube-dns vì
    interface được dùng cho cân bằng tải IPVS đã sử dụng địa chỉ này.
    `__PILLAR__UPSTREAM__SERVERS__` sẽ được điền bởi các pod node-local-dns.

* Chạy `kubectl create -f nodelocaldns.yaml`

* Nếu dùng kube-proxy ở chế độ IPVS, flag `--cluster-dns` của kubelet cần được sửa
  để dùng `<node-local-address>` mà NodeLocal DNSCache đang lắng nghe.
  Nếu không, không cần sửa giá trị của flag `--cluster-dns`,
  vì NodeLocal DNSCache lắng nghe trên cả IP của service kube-dns lẫn
  `<node-local-address>`.

Sau khi được bật, các Pod `node-local-dns` sẽ chạy trong namespace `kube-system`
trên mỗi node của cluster. Pod này chạy [CoreDNS](https://github.com/coredns/coredns)
ở chế độ cache, do đó tất cả các metrics của CoreDNS được các plugin khác nhau công bố sẽ
sẵn có ở phạm vi từng node.

Bạn có thể tắt tính năng này bằng cách xóa DaemonSet, dùng `kubectl delete -f <manifest>`.
Bạn cũng nên hoàn tác mọi thay đổi mà bạn đã thực hiện với cấu hình kubelet.

## Cấu hình StubDomain và server Upstream (StubDomains and Upstream server Configuration)

Các StubDomain và server upstream được chỉ định trong ConfigMap `kube-dns` thuộc namespace
`kube-system` sẽ tự động được các pod `node-local-dns` tiếp nhận. Nội dung của ConfigMap cần
tuân theo định dạng trình bày trong [ví dụ](204-dns-custom-nameservers-vi.md#ví-dụ-example).
ConfigMap `node-local-dns` cũng có thể được sửa trực tiếp với cấu hình stubDomain
theo định dạng Corefile. Một số nhà cung cấp đám mây có thể không cho phép sửa trực tiếp
ConfigMap `node-local-dns`. Trong những trường hợp đó, ConfigMap `kube-dns` có thể được cập nhật.

## Đặt giới hạn memory (Setting memory limits)

Các Pod `node-local-dns` dùng memory để lưu các entry cache và xử lý truy vấn.
Vì chúng không theo dõi (watch) các đối tượng Kubernetes, kích thước cluster hay số lượng
Service / EndpointSlice không ảnh hưởng trực tiếp đến mức sử dụng memory. Mức sử dụng memory
chịu ảnh hưởng bởi mẫu hình truy vấn DNS.
Theo [tài liệu CoreDNS](https://github.com/coredns/deployment/blob/master/kubernetes/Scaling_CoreDNS.md),
> Kích thước cache mặc định là 10000 entry, dùng khoảng 30 MB khi được lấp đầy hoàn toàn.

Đây sẽ là mức sử dụng memory cho mỗi server block (nếu cache được lấp đầy hoàn toàn).
Có thể giảm mức sử dụng memory bằng cách chỉ định kích thước cache nhỏ hơn.

Số lượng truy vấn đồng thời gắn liền với nhu cầu memory, vì mỗi
goroutine bổ sung được dùng để xử lý một truy vấn cần một lượng memory nhất định. Bạn có thể
đặt giới hạn trên bằng tùy chọn `max_concurrent` trong plugin forward.

Nếu một Pod `node-local-dns` cố dùng nhiều memory hơn mức khả dụng (do tổng tài nguyên
hệ thống, hoặc do một
[resource limit](110-manage-resources-containers-vi.md) đã được cấu hình), hệ điều hành
có thể tắt container của pod đó.
Nếu điều này xảy ra, container bị chấm dứt ("OOMKilled") sẽ không dọn dẹp các quy tắc lọc gói
tin tùy chỉnh mà nó đã thêm vào trước đó lúc khởi động.
Container `node-local-dns` sẽ được khởi động lại (vì được quản lý như một phần của DaemonSet),
nhưng mỗi lần container gặp sự cố sẽ dẫn tới một khoảng gián đoạn DNS ngắn: các quy tắc lọc gói
tin điều hướng các truy vấn DNS tới một Pod cục bộ đang không khỏe mạnh.

Bạn có thể xác định một giới hạn memory phù hợp bằng cách chạy các pod node-local-dns không có
giới hạn và đo mức sử dụng đỉnh. Bạn cũng có thể thiết lập và dùng một
[VerticalPodAutoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
ở _chế độ recommender_, rồi xem các khuyến nghị của nó.
