# Hướng dẫn sử dụng IP Masquerade Agent (IP Masquerade Agent User Guide)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/ip-masq-agent/>

Trang này hướng dẫn cách cấu hình và bật `ip-masq-agent`.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn v1.36. Để kiểm tra phiên bản, nhập
`kubectl version`.

## Hướng dẫn sử dụng IP Masquerade Agent (IP Masquerade Agent User Guide)

`ip-masq-agent` cấu hình các quy tắc iptables để ẩn địa chỉ IP của pod phía sau địa chỉ IP của
node trong cluster. Việc này thường được thực hiện khi gửi lưu lượng (traffic) tới các đích nằm
ngoài dải [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) dành cho pod của
cluster.

### Các thuật ngữ chính (Key Terms)

* **NAT (Network Address Translation - Biên dịch địa chỉ mạng)**:
  Là một phương pháp ánh xạ lại một địa chỉ IP sang một địa chỉ khác bằng cách sửa đổi thông tin
  địa chỉ nguồn và/hoặc địa chỉ đích trong IP header. Thường được thực hiện bởi thiết bị đảm
  nhiệm việc định tuyến IP.
* **Masquerading (che giấu địa chỉ)**:
  Một dạng NAT thường được dùng để thực hiện biên dịch địa chỉ kiểu nhiều-thành-một, trong đó
  nhiều địa chỉ IP nguồn được che phía sau một địa chỉ duy nhất — thường là địa chỉ của thiết bị
  đang định tuyến IP. Trong Kubernetes, đó là địa chỉ IP của Node.
* **CIDR (Classless Inter-Domain Routing)**:
  Dựa trên kỹ thuật subnet mask có độ dài thay đổi (variable-length subnet masking), cho phép
  chỉ định các prefix với độ dài tùy ý. CIDR đưa ra một cách biểu diễn mới cho địa chỉ IP, nay
  thường được biết đến với tên gọi **ký pháp CIDR (CIDR notation)**, trong đó một địa chỉ hoặc
  một routing prefix được viết kèm hậu tố cho biết số bit của prefix, ví dụ 192.168.2.0/24.
* **Link Local**:
  Địa chỉ link-local là địa chỉ mạng chỉ có hiệu lực cho các giao tiếp bên trong phân đoạn mạng
  (network segment) hoặc miền quảng bá (broadcast domain) mà host đang kết nối vào. Các địa chỉ
  link-local cho IPv4 được định nghĩa trong khối địa chỉ 169.254.0.0/16 theo ký pháp CIDR.

ip-masq-agent cấu hình các quy tắc iptables để xử lý việc masquerade địa chỉ IP của node/pod khi
gửi lưu lượng tới các đích nằm ngoài IP của node trong cluster và dải Cluster IP. Về bản chất,
việc này ẩn các địa chỉ IP của pod phía sau địa chỉ IP của node trong cluster. Trong một số môi
trường, lưu lượng đi tới các địa chỉ "bên ngoài" phải xuất phát từ một địa chỉ máy đã biết. Ví dụ,
trên Google Cloud, mọi lưu lượng đi ra internet phải xuất phát từ IP của VM. Khi dùng container,
như trong Google Kubernetes Engine, IP của Pod sẽ bị từ chối khi đi ra ngoài (egress). Để tránh
điều này, chúng ta phải ẩn IP của Pod phía sau chính địa chỉ IP của VM — thường được gọi là
"masquerade". Theo mặc định, agent được cấu hình để coi ba dải IP riêng (private) được chỉ định
bởi [RFC 1918](https://tools.ietf.org/html/rfc1918) là các
[CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) không masquerade
(non-masquerade). Các dải này là `10.0.0.0/8`, `172.16.0.0/12` và `192.168.0.0/16`.
Agent cũng mặc định coi dải link-local (169.254.0.0/16) là một CIDR không masquerade.
Agent được cấu hình để nạp lại cấu hình của nó từ vị trí */etc/config/ip-masq-agent* mỗi
60 giây, và khoảng thời gian này cũng có thể cấu hình được.

![Ví dụ masq/non-masq](https://kubernetes.io/images/docs/ip-masq.png)

File cấu hình của agent phải được viết theo cú pháp YAML hoặc JSON, và có thể chứa ba khóa (key)
tùy chọn:

* `nonMasqueradeCIDRs`: Một danh sách chuỗi theo ký pháp
  [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) chỉ định các dải không
  masquerade.
* `masqLinkLocal`: Một giá trị Boolean (true/false) cho biết có masquerade lưu lượng tới prefix
  link-local `169.254.0.0/16` hay không. Mặc định là false.
* `resyncInterval`: Khoảng thời gian mà agent sẽ thử nạp lại cấu hình từ đĩa.
  Ví dụ: '30s', trong đó 's' nghĩa là giây, 'ms' nghĩa là mili giây.

Lưu lượng tới các dải 10.0.0.0/8, 172.16.0.0/12 và 192.168.0.0/16 sẽ KHÔNG bị masquerade. Mọi
lưu lượng khác (được coi là đi ra internet) sẽ bị masquerade. Một ví dụ về đích cục bộ (local)
nhìn từ một pod có thể là địa chỉ IP của Node chứa nó, cũng như địa chỉ của một node khác hoặc
một trong các địa chỉ IP thuộc dải Cluster IP. Mọi lưu lượng khác sẽ bị masquerade theo mặc định.
Các mục dưới đây cho thấy tập quy tắc mặc định mà ip-masq-agent áp dụng:

```shell
iptables -t nat -L IP-MASQ-AGENT
```

```none
target     prot opt source               destination
RETURN     all  --  anywhere             169.254.0.0/16       /* ip-masq-agent: cluster-local traffic should not be subject to MASQUERADE */ ADDRTYPE match dst-type !LOCAL
RETURN     all  --  anywhere             10.0.0.0/8           /* ip-masq-agent: cluster-local traffic should not be subject to MASQUERADE */ ADDRTYPE match dst-type !LOCAL
RETURN     all  --  anywhere             172.16.0.0/12        /* ip-masq-agent: cluster-local traffic should not be subject to MASQUERADE */ ADDRTYPE match dst-type !LOCAL
RETURN     all  --  anywhere             192.168.0.0/16       /* ip-masq-agent: cluster-local traffic should not be subject to MASQUERADE */ ADDRTYPE match dst-type !LOCAL
MASQUERADE  all  --  anywhere             anywhere             /* ip-masq-agent: outbound traffic should be subject to MASQUERADE (this match must come after cluster-local CIDR matches) */ ADDRTYPE match dst-type !LOCAL

```

Theo mặc định, trong GCE/Google Kubernetes Engine, nếu network policy được bật hoặc bạn đang
dùng một cluster CIDR không nằm trong dải 10.0.0.0/8, `ip-masq-agent` sẽ chạy trong cluster của
bạn. Nếu bạn đang chạy trong môi trường khác, bạn có thể thêm [DaemonSet](66-daemonset-vi.md)
`ip-masq-agent` vào cluster của mình.

## Tạo một ip-masq-agent (Create an ip-masq-agent)

Để tạo một ip-masq-agent, chạy lệnh kubectl sau:

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/ip-masq-agent/master/ip-masq-agent.yaml
```

Bạn cũng phải gán label node phù hợp cho mọi node trong cluster mà bạn muốn agent chạy trên đó.

```shell
kubectl label nodes my-node node.kubernetes.io/masq-agent-ds-ready=true
```

Bạn có thể tìm thêm thông tin trong tài liệu của ip-masq-agent
[tại đây](https://github.com/kubernetes-sigs/ip-masq-agent).

Trong hầu hết các trường hợp, tập quy tắc mặc định là đủ dùng; tuy nhiên, nếu điều đó không đúng
với cluster của bạn, bạn có thể tạo và áp dụng một
[ConfigMap](275-configure-pod-configmap-vi.md)
để tùy biến các dải IP chịu ảnh hưởng. Ví dụ, để chỉ cho phép ip-masq-agent xét tới dải
10.0.0.0/8, bạn có thể tạo
[ConfigMap](275-configure-pod-configmap-vi.md)
sau trong một file tên là "config".

> **Ghi chú:**
>
> Điều quan trọng là file phải có tên là config, vì theo mặc định, tên đó sẽ được dùng làm khóa
> (key) để `ip-masq-agent` tra cứu:
>
> ```yaml
> nonMasqueradeCIDRs:
>   - 10.0.0.0/8
> resyncInterval: 60s
> ```

Chạy lệnh sau để thêm configmap vào cluster của bạn:

```shell
kubectl create configmap ip-masq-agent --from-file=config --namespace=kube-system
```

Thao tác này sẽ cập nhật một file nằm tại `/etc/config/ip-masq-agent`, file này được kiểm tra
định kỳ mỗi `resyncInterval` và được áp dụng lên node của cluster.
Sau khi khoảng thời gian resync trôi qua, bạn sẽ thấy các quy tắc iptables phản ánh thay đổi của
bạn:

```shell
iptables -t nat -L IP-MASQ-AGENT
```

```none
Chain IP-MASQ-AGENT (1 references)
target     prot opt source               destination
RETURN     all  --  anywhere             169.254.0.0/16       /* ip-masq-agent: cluster-local traffic should not be subject to MASQUERADE */ ADDRTYPE match dst-type !LOCAL
RETURN     all  --  anywhere             10.0.0.0/8           /* ip-masq-agent: cluster-local
MASQUERADE  all  --  anywhere             anywhere             /* ip-masq-agent: outbound traffic should be subject to MASQUERADE (this match must come after cluster-local CIDR matches) */ ADDRTYPE match dst-type !LOCAL
```

Theo mặc định, dải link-local (169.254.0.0/16) cũng được ip-masq agent xử lý — agent sẽ thiết
lập các quy tắc iptables tương ứng. Để ip-masq-agent bỏ qua dải link-local, bạn có thể đặt
`masqLinkLocal` thành true trong ConfigMap.

```yaml
nonMasqueradeCIDRs:
  - 10.0.0.0/8
resyncInterval: 60s
masqLinkLocal: true
```
