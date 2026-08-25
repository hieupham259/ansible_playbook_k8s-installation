# Sử dụng NodeLocal DNSCache trong Cluster Kubernetes (Using NodeLocal DNSCache in Kubernetes Clusters)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 21 — DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy),
bài 4/14 · Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối
[mục giai đoạn 21](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy). Với bài này, phần
kiểm chứng làm được ngay là: đọc chế độ kube-proxy của cluster lab rồi xác định nhánh cấu hình
nào trong bài áp dụng cho nó.

Bài này mô tả một **add-on tùy chọn**, không nằm trong danh mục add-on đã khóa của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md). Đọc để hiểu kiến trúc và biết khi nào cần bật; không
cần cài lên cluster lab. Nó nối tiếp bài [10](10-dns-pod-service-vi.md) và bài
[204](204-dns-custom-nameservers-vi.md) của chính giai đoạn 21.

**Phải hiểu ở lần đọc này:**

- Kiến trúc mới ở mục *Giới thiệu*: Pod ở chế độ DNS `ClusterFirst` thôi truy vấn `serviceIP` của
  kube-dns mà hỏi thẳng agent cache **chạy trên cùng node**, nhờ đó bỏ qua quy tắc iptables DNAT
  và việc theo dõi kết nối (connection tracking).
- Agent cache cục bộ **không thay** kube-dns: mục *Giới thiệu* nói rõ nó vẫn truy vấn service
  kube-dns cho các trường hợp cache miss của hostname trong cluster (hậu tố `cluster.local`).
- Bốn lý do đáng nhớ ở mục *Động lực*: giảm độ trễ khi node không có instance kube-dns cục bộ;
  tránh race condition của conntrack và tránh entry UDP lấp đầy bảng conntrack; nâng kết nối lên
  TCP để entry conntrack được xóa ngay khi đóng thay vì chờ hết `nf_conntrack_udp_timeout`; giảm
  tail latency do gói UDP rơi (3 lần thử lại + timeout 10s).
- Hai nhánh cấu hình ở mục *Cấu hình* khác nhau ở đúng hai điểm: với kube-proxy chế độ IPTABLES,
  pod `node-local-dns` lắng nghe **cả** IP service kube-dns lẫn `<node-local-address>` nên không
  phải sửa `--cluster-dns`; với chế độ IPVS nó **chỉ** lắng nghe `<node-local-address>` (vì
  interface IPVS đã chiếm cluster IP của kube-dns) nên **bắt buộc** sửa `--cluster-dns` của kubelet.
- Ghi chú ở đầu mục *Cấu hình* về địa chỉ lắng nghe cục bộ, và cảnh báo ở mục *Đặt giới hạn
  memory*: container bị OOMKilled **không dọn** các quy tắc lọc gói tin nó đã thêm lúc khởi động,
  nên mỗi lần crash là một khoảng gián đoạn DNS ngắn.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | lộ trình không dùng minikube hay cluster dùng chung | cluster VM ba node của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) là môi trường thực hành duy nhất |
| Các lệnh `sed` thay `__PILLAR__*` trong manifest mẫu và `kubectl create -f nodelocaldns.yaml` | đây là bước triển khai một add-on ngoài baseline; Checkpoint giai đoạn 21 không yêu cầu nó | quay lại đúng mục *Cấu hình* của bài này khi vận hành cluster có QPS DNS cao; nền ConfigMap DNS đã có ở bài [204](204-dns-custom-nameservers-vi.md) |
| Câu "tất cả metrics của CoreDNS sẵn có ở phạm vi từng node" và cách đo mức dùng memory đỉnh | chưa có pipeline metrics để đọc những con số đó | [Giai đoạn 23 — Giám sát và cảnh báo](00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo) |
| Dùng VerticalPodAutoscaler ở *chế độ recommender* để chọn giới hạn memory | VPA là add-on ngoài Kubernetes, chưa có trong baseline | **nợ #1 chưa trả** của [sổ nợ lộ trình](00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình) |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 21:

1. Trước khi bật NodeLocal DNSCache, một Pod ở chế độ DNS `ClusterFirst` gửi truy vấn tới đâu, và
   truy vấn đó được đưa tới CoreDNS bằng cơ chế nào? Sau khi bật, hai thứ nào trên đường đi cũ
   biến mất?
2. **Câu bẫy.** Pod `node-local-dns` chạy CoreDNS ở chế độ cache. Vậy sau khi bật nó, service
   kube-dns của cluster còn việc gì để làm không?
3. Cluster lab dựng bằng kubeadm nên kube-proxy chạy ở chế độ mặc định là iptables. Nếu bạn bật
   NodeLocal DNSCache trên `lab-k8s-worker1` và `lab-k8s-worker2`, pod `node-local-dns` sẽ lắng
   nghe trên (những) địa chỉ nào, và bạn có phải sửa `--cluster-dns` của kubelet không? Câu trả
   lời đổi thế nào nếu bạn chuyển kube-proxy sang chế độ IPVS?
4. Vì sao bài khuyến nghị chọn địa chỉ lắng nghe cục bộ trong dải link-local `169.254.0.0/16`
   (hoặc `fd00::/8` với IPv6) thay vì một địa chỉ bất kỳ?
5. Một pod `node-local-dns` bị OOMKilled rồi được DaemonSet khởi động lại. Ngoài việc container
   restart, bài cảnh báo hậu quả cụ thể nào nữa?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Pod truy vấn tới **`serviceIP` của kube-dns**, và địa chỉ đó được **quy tắc iptables do
   kube-proxy thêm vào** chuyển thành một endpoint của kube-dns/CoreDNS. Sau khi bật, Pod hỏi
   thẳng agent cache trên **cùng node**, nên bỏ được **iptables DNAT** và **connection tracking**
   trên đường đi.
2. **Còn.** Agent cache cục bộ **không thay thế** kube-dns: với hostname trong cluster (hậu tố
   `cluster.local`) mà cache **miss**, nó vẫn truy vấn service kube-dns để lấy câu trả lời.
   NodeLocal DNSCache rút ngắn đường đi cho phần lớn truy vấn, chứ không xóa vai trò của
   CoreDNS trung tâm.
3. Ở chế độ IPTABLES, pod `node-local-dns` lắng nghe trên **cả hai**: IP của service kube-dns
   **và** `<node-local-address>`. Vì Pod vẫn hỏi được bằng IP cũ nên **không cần sửa
   `--cluster-dns`**. Ở chế độ IPVS thì ngược lại: pod **chỉ** lắng nghe `<node-local-address>`
   — interface dùng cho cân bằng tải IPVS đã chiếm cluster IP của kube-dns nên không bind được
   — nên **bắt buộc sửa `--cluster-dns` của kubelet** thành `<node-local-address>`.
4. Vì địa chỉ đó phải **được đảm bảo không xung đột với bất kỳ IP nào đang tồn tại trong
   cluster**. Dải link-local có phạm vi cục bộ trên từng node nên thỏa điều kiện đó một cách
   an toàn.
5. Container bị OOMKilled **không dọn các quy tắc lọc gói tin mà nó đã thêm lúc khởi động**. Các
   quy tắc còn sót lại vẫn điều hướng truy vấn DNS tới một Pod cục bộ đang không khỏe mạnh, nên
   **mỗi lần crash tạo ra một khoảng gián đoạn DNS ngắn** cho toàn bộ Pod trên node đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
