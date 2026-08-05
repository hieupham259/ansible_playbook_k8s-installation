# Chính sách mạng (Network Policies)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/network-policies/>
>
> Nếu bạn muốn kiểm soát luồng traffic ở mức địa chỉ IP hoặc port (tầng 3 hoặc 4 của mô hình OSI),
> NetworkPolicy cho phép bạn chỉ định các quy tắc cho luồng traffic bên trong cluster,
> cũng như giữa các Pod và thế giới bên ngoài.
> Cluster của bạn phải sử dụng một network plugin có hỗ trợ thực thi NetworkPolicy.

Nếu bạn muốn kiểm soát luồng traffic ở mức địa chỉ IP hoặc port cho các giao thức TCP, UDP và SCTP,
bạn có thể cân nhắc sử dụng NetworkPolicy của Kubernetes cho những ứng dụng cụ thể trong cluster.
NetworkPolicy là một cấu trúc lấy ứng dụng làm trung tâm (application-centric), cho phép bạn chỉ định
cách một pod được phép giao tiếp qua mạng với các "thực thể" (entity) mạng khác nhau
(chúng tôi dùng từ "thực thể" ở đây để tránh trùng lặp với các thuật ngữ phổ biến hơn như
"endpoint" và "service" vốn mang ý nghĩa riêng trong Kubernetes).
NetworkPolicy áp dụng cho kết nối có pod ở một hoặc cả hai đầu, và không liên quan đến
các kết nối khác.

Các thực thể mà một Pod có thể giao tiếp được xác định thông qua tổ hợp của ba
định danh sau:

1. Các pod khác được cho phép (ngoại lệ: một pod không thể tự chặn truy cập đến chính nó)
1. Các namespace được cho phép
1. Các dải IP (IP block) (ngoại lệ: traffic đến và đi từ node nơi Pod đang chạy luôn được
   cho phép, bất kể địa chỉ IP của Pod hay của node)

Khi định nghĩa một NetworkPolicy dựa trên pod hoặc namespace, bạn dùng một
bộ chọn (selector) để chỉ định traffic nào được phép đến và đi từ (các) Pod khớp với
bộ chọn đó.

Trong khi đó, khi tạo NetworkPolicy dựa trên IP, chúng ta định nghĩa chính sách dựa trên các dải IP (dải CIDR).

## Điều kiện tiên quyết (Prerequisites)

Network policy được hiện thực bởi [network plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/).
Để sử dụng network policy, bạn phải dùng một giải pháp mạng có hỗ trợ NetworkPolicy.
Việc tạo một tài nguyên NetworkPolicy mà không có controller hiện thực nó sẽ không có tác dụng gì.

## Hai kiểu cô lập pod (The two sorts of pod isolation)

Có hai kiểu cô lập (isolation) cho một pod: cô lập chiều đi (egress) và cô lập chiều đến (ingress).
Chúng liên quan đến việc những kết nối nào có thể được thiết lập. "Cô lập" ở đây không phải là
tuyệt đối, mà có nghĩa là "có một số hạn chế được áp dụng". Trường hợp ngược lại,
"không cô lập theo hướng $direction", nghĩa là không có hạn chế nào được áp dụng theo hướng
đã nêu. Hai kiểu cô lập (hoặc không cô lập) này được khai báo độc lập với nhau, và cả hai
đều liên quan đến một kết nối từ pod này sang pod khác.

Theo mặc định, một pod không bị cô lập chiều đi (egress); mọi kết nối đi ra ngoài đều được cho phép.
Một pod bị cô lập chiều đi nếu tồn tại bất kỳ NetworkPolicy nào vừa chọn (select) pod đó vừa có
"Egress" trong `policyTypes` của nó; ta nói rằng chính sách như vậy áp dụng cho pod đối với chiều đi.
Khi một pod bị cô lập chiều đi, những kết nối duy nhất được phép xuất phát từ pod là những kết nối
được cho phép bởi danh sách `egress` của một NetworkPolicy nào đó áp dụng cho pod đối với chiều đi.
Traffic phản hồi (reply traffic) cho những kết nối được phép đó cũng sẽ được ngầm cho phép.
Hiệu lực của các danh sách `egress` đó được cộng gộp với nhau.

Theo mặc định, một pod không bị cô lập chiều đến (ingress); mọi kết nối đi vào đều được cho phép.
Một pod bị cô lập chiều đến nếu tồn tại bất kỳ NetworkPolicy nào vừa chọn pod đó vừa có
"Ingress" trong `policyTypes` của nó; ta nói rằng chính sách như vậy áp dụng cho pod đối với chiều đến.
Khi một pod bị cô lập chiều đến, những kết nối duy nhất được phép đi vào pod là những kết nối
từ node của pod và những kết nối được cho phép bởi danh sách `ingress` của một NetworkPolicy nào đó
áp dụng cho pod đối với chiều đến. Traffic phản hồi cho những kết nối được phép đó cũng sẽ được
ngầm cho phép. Hiệu lực của các danh sách `ingress` đó được cộng gộp với nhau.

Các network policy không xung đột với nhau; chúng có tính cộng gộp (additive). Nếu có một hay nhiều
chính sách áp dụng cho một pod theo một hướng nhất định, thì các kết nối được phép theo hướng đó từ
pod là hợp (union) của những gì các chính sách áp dụng cho phép. Do đó, thứ tự đánh giá không ảnh
hưởng đến kết quả của chính sách.

Để một kết nối từ pod nguồn đến pod đích được cho phép, cả chính sách egress trên pod nguồn
lẫn chính sách ingress trên pod đích đều phải cho phép kết nối đó. Nếu một trong hai phía
không cho phép, kết nối sẽ không diễn ra.

## Tài nguyên NetworkPolicy (The NetworkPolicy resource) {#networkpolicy-resource}

Xem tài liệu tham khảo [NetworkPolicy](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#networkpolicy-v1-networking-k8s-io)
để có định nghĩa đầy đủ về tài nguyên này.

Một NetworkPolicy mẫu có thể trông như sau:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

> **Ghi chú:**
> Việc POST nội dung này lên API server của cluster sẽ không có tác dụng gì trừ khi giải pháp
> mạng bạn chọn có hỗ trợ network policy.

__Các trường bắt buộc__: Giống như mọi cấu hình Kubernetes khác, một NetworkPolicy cần các trường
`apiVersion`, `kind` và `metadata`. Để biết thông tin chung về cách làm việc với file cấu hình, xem
[Cấu hình Pod để sử dụng ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
và [Quản lý object](./27-object-management-vi.md).

**spec**: [spec](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)
của NetworkPolicy chứa toàn bộ thông tin cần thiết để định nghĩa một network policy cụ thể
trong namespace đã cho.

**podSelector**: Mỗi NetworkPolicy bao gồm một `podSelector` để chọn nhóm các pod mà chính sách
áp dụng lên. Chính sách trong ví dụ chọn các pod có label "role=db". Một `podSelector` rỗng
sẽ chọn tất cả các pod trong namespace.

**policyTypes**: Mỗi NetworkPolicy bao gồm một danh sách `policyTypes`, có thể chứa `Ingress`,
`Egress`, hoặc cả hai. Trường `policyTypes` cho biết chính sách đã cho có áp dụng cho traffic
đi vào (ingress) các pod được chọn, traffic đi ra (egress) từ các pod được chọn, hay cả hai.
Nếu không chỉ định `policyTypes` nào trên NetworkPolicy thì theo mặc định `Ingress` sẽ luôn được
đặt, và `Egress` sẽ được đặt nếu NetworkPolicy có bất kỳ quy tắc egress nào.

**ingress**: Mỗi NetworkPolicy có thể bao gồm một danh sách các quy tắc `ingress` được cho phép.
Mỗi quy tắc cho phép traffic khớp đồng thời cả hai phần `from` và `ports`. Chính sách trong ví dụ
chứa một quy tắc duy nhất, khớp traffic trên một port duy nhất, từ một trong ba nguồn: nguồn thứ
nhất được chỉ định qua `ipBlock`, nguồn thứ hai qua `namespaceSelector` và nguồn thứ ba qua
`podSelector`.

**egress**: Mỗi NetworkPolicy có thể bao gồm một danh sách các quy tắc `egress` được cho phép.
Mỗi quy tắc cho phép traffic khớp đồng thời cả hai phần `to` và `ports`. Chính sách trong ví dụ
chứa một quy tắc duy nhất, khớp traffic trên một port duy nhất đến bất kỳ đích nào trong
`10.0.0.0/24`.

Như vậy, NetworkPolicy trong ví dụ:

1. cô lập các pod `role=db` trong namespace `default` cho cả traffic chiều đến lẫn chiều đi
   (nếu chúng chưa bị cô lập từ trước)
1. (Quy tắc ingress) cho phép các kết nối đến tất cả các pod trong namespace `default` có label
   `role=db` trên TCP port 6379 từ:

   * bất kỳ pod nào trong namespace `default` có label `role=frontend`
   * bất kỳ pod nào trong một namespace có label `project=myproject`
   * các địa chỉ IP trong dải `172.17.0.0`–`172.17.0.255` và `172.17.2.0`–`172.17.255.255`
     (tức là toàn bộ `172.17.0.0/16` ngoại trừ `172.17.1.0/24`)

1. (Quy tắc egress) cho phép các kết nối từ bất kỳ pod nào trong namespace `default` có label
   `role=db` đến CIDR `10.0.0.0/24` trên TCP port 5978

Xem bài hướng dẫn từng bước [Khai báo Network Policy](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)
để có thêm ví dụ.

## Hành vi của các bộ chọn `to` và `from` (Behavior of `to` and `from` selectors)

Có bốn loại bộ chọn có thể được chỉ định trong phần `from` của `ingress` hoặc phần `to` của
`egress`:

**podSelector**: Bộ chọn này chọn những Pod cụ thể trong cùng namespace với NetworkPolicy để
cho phép làm nguồn ingress hoặc đích egress.

**namespaceSelector**: Bộ chọn này chọn những namespace cụ thể mà tất cả các Pod trong đó được
cho phép làm nguồn ingress hoặc đích egress.

**namespaceSelector** *và* **podSelector**: Một mục `to`/`from` duy nhất chỉ định cả
`namespaceSelector` và `podSelector` sẽ chọn những Pod cụ thể trong những namespace cụ thể.
Hãy cẩn thận dùng đúng cú pháp YAML. Ví dụ:

```yaml
  ...
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          user: alice
      podSelector:
        matchLabels:
          role: client
  ...
```

Chính sách này chứa một phần tử `from` duy nhất, cho phép các kết nối từ những Pod có label
`role=client` trong các namespace có label `user=alice`. Nhưng chính sách sau đây thì khác:

```yaml
  ...
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          user: alice
    - podSelector:
        matchLabels:
          role: client
  ...
```

Nó chứa hai phần tử trong mảng `from`, và cho phép các kết nối từ những Pod trong Namespace
cục bộ có label `role=client`, *hoặc* từ bất kỳ Pod nào trong bất kỳ namespace nào có label
`user=alice`.

Khi chưa chắc chắn, hãy dùng `kubectl describe` để xem Kubernetes đã diễn giải chính sách
như thế nào.

<a name="behavior-of-ipblock-selectors"></a>
**ipBlock**: Bộ chọn này chọn những dải IP CIDR cụ thể để cho phép làm nguồn ingress hoặc đích
egress. Đây nên là các IP bên ngoài cluster, vì IP của Pod là tạm thời và không thể đoán trước.

Các cơ chế ingress và egress của cluster thường yêu cầu ghi đè (rewrite) IP nguồn hoặc IP đích
của các gói tin. Trong những trường hợp điều này xảy ra, việc nó xảy ra trước hay sau khi
NetworkPolicy được xử lý là không được định nghĩa, và hành vi có thể khác nhau tùy theo tổ hợp
của network plugin, nhà cung cấp cloud, cách hiện thực `Service`, v.v.

Đối với chiều đến (ingress), điều này có nghĩa là trong một số trường hợp bạn có thể lọc các
gói tin đi vào dựa trên IP nguồn gốc thực sự, trong khi ở những trường hợp khác, "IP nguồn" mà
NetworkPolicy xử lý có thể là IP của một `LoadBalancer` hoặc của node chứa Pod, v.v.

Đối với chiều đi (egress), điều này có nghĩa là các kết nối từ pod đến các IP của `Service`
được ghi đè thành IP bên ngoài cluster có thể chịu hoặc không chịu tác động của các chính sách
dựa trên `ipBlock`.

## Các chính sách mặc định (Default policies)

Theo mặc định, nếu không có chính sách nào tồn tại trong một namespace, thì mọi traffic chiều đến
và chiều đi đều được cho phép đến và đi từ các pod trong namespace đó. Các ví dụ sau cho phép bạn
thay đổi hành vi mặc định trong namespace đó.

### Mặc định từ chối mọi traffic chiều đến (Default deny all ingress traffic)

Bạn có thể tạo một chính sách cô lập ingress "mặc định" cho một namespace bằng cách tạo một
NetworkPolicy chọn tất cả các pod nhưng không cho phép bất kỳ traffic chiều đến nào tới các pod đó.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

Điều này bảo đảm rằng ngay cả những pod không được chọn bởi bất kỳ NetworkPolicy nào khác vẫn sẽ
bị cô lập chiều đến. Chính sách này không ảnh hưởng đến việc cô lập chiều đi của bất kỳ pod nào.

### Cho phép mọi traffic chiều đến (Allow all ingress traffic)

Nếu bạn muốn cho phép mọi kết nối đi vào tất cả các pod trong một namespace, bạn có thể tạo một
chính sách cho phép điều đó một cách tường minh.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
spec:
  podSelector: {}
  ingress:
  - {}
  policyTypes:
  - Ingress
```

Với chính sách này, không một chính sách bổ sung nào có thể khiến bất kỳ kết nối đi vào các pod
đó bị từ chối. Chính sách này không có tác dụng đối với việc cô lập chiều đi của bất kỳ pod nào.

### Mặc định từ chối mọi traffic chiều đi (Default deny all egress traffic)

Bạn có thể tạo một chính sách cô lập egress "mặc định" cho một namespace bằng cách tạo một
NetworkPolicy chọn tất cả các pod nhưng không cho phép bất kỳ traffic chiều đi nào từ các pod đó.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

Điều này bảo đảm rằng ngay cả những pod không được chọn bởi bất kỳ NetworkPolicy nào khác cũng sẽ
không được phép có traffic chiều đi. Chính sách này không thay đổi hành vi cô lập chiều đến của
bất kỳ pod nào.

> **Thận trọng:**
> Một chính sách mặc định từ chối toàn bộ egress cũng chặn cả traffic DNS. Nếu workload của bạn
> cần phân giải DNS, bạn phải thêm một NetworkPolicy riêng cho phép egress đến Service DNS
> của cluster.

### Cho phép mọi traffic chiều đi (Allow all egress traffic)

Nếu bạn muốn cho phép mọi kết nối từ tất cả các pod trong một namespace, bạn có thể tạo một
chính sách cho phép tường minh mọi kết nối đi ra từ các pod trong namespace đó.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
spec:
  podSelector: {}
  egress:
  - {}
  policyTypes:
  - Egress
```

Với chính sách này, không một chính sách bổ sung nào có thể khiến bất kỳ kết nối đi ra từ các pod
đó bị từ chối. Chính sách này không có tác dụng đối với việc cô lập chiều đến của bất kỳ pod nào.

### Mặc định từ chối mọi traffic chiều đến và chiều đi (Default deny all ingress and all egress traffic)

Bạn có thể tạo một chính sách "mặc định" cho một namespace để ngăn mọi traffic chiều đến VÀ
chiều đi bằng cách tạo NetworkPolicy sau trong namespace đó.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Điều này bảo đảm rằng ngay cả những pod không được chọn bởi bất kỳ NetworkPolicy nào khác cũng sẽ
không được phép có traffic chiều đến hay chiều đi.

## Lọc traffic mạng (Network traffic filtering)

NetworkPolicy được định nghĩa cho các kết nối [tầng 4](https://en.wikipedia.org/wiki/OSI_model#Layer_4:_Transport_layer)
(TCP, UDP, và tùy chọn SCTP). Đối với tất cả các giao thức khác, hành vi có thể khác nhau
giữa các network plugin.

> **Ghi chú:**
> Bạn phải sử dụng một CNI plugin có hỗ trợ NetworkPolicy cho giao thức SCTP.

Khi một network policy `deny all` (từ chối tất cả) được định nghĩa, nó chỉ được bảo đảm từ chối
các kết nối TCP, UDP và SCTP. Đối với các giao thức khác, chẳng hạn như ARP hay ICMP, hành vi là
không được định nghĩa. Điều tương tự cũng áp dụng cho các quy tắc cho phép: khi một pod cụ thể
được cho phép làm nguồn ingress hoặc đích egress, những gì xảy ra với (ví dụ) các gói tin ICMP là
không được định nghĩa. Các giao thức như ICMP có thể được cho phép bởi một số network plugin và
bị từ chối bởi những plugin khác.

## Nhắm đến một dải port (Targeting a range of ports)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.25 [stable]`

Khi viết một NetworkPolicy, bạn có thể nhắm đến một dải port thay vì một port duy nhất.

Điều này có thể thực hiện được bằng cách sử dụng trường `endPort`, như ví dụ sau:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: multi-port-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
      ports:
        - protocol: TCP
          port: 32000
          endPort: 32768
```

Quy tắc trên cho phép bất kỳ Pod nào có label `role=db` trong namespace `default` giao tiếp
với bất kỳ IP nào trong dải `10.0.0.0/24` qua TCP, miễn là port đích nằm trong khoảng từ
32000 đến 32768.

Các hạn chế sau được áp dụng khi sử dụng trường này:

* Trường `endPort` phải bằng hoặc lớn hơn trường `port`.
* `endPort` chỉ có thể được định nghĩa khi `port` cũng được định nghĩa.
* Cả hai port đều phải là số.

> **Ghi chú:**
> Cluster của bạn phải sử dụng một CNI plugin có hỗ trợ trường `endPort` trong đặc tả
> NetworkPolicy.
> Nếu [network plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
> của bạn không hỗ trợ trường `endPort` mà bạn vẫn chỉ định một NetworkPolicy có trường đó,
> chính sách sẽ chỉ được áp dụng cho trường `port` duy nhất.

## Nhắm đến nhiều namespace theo label (Targeting multiple namespaces by label)

Trong kịch bản này, NetworkPolicy `Egress` của bạn nhắm đến nhiều hơn một namespace thông qua
tên label của chúng. Để làm được điều này, bạn cần gắn label cho các namespace đích. Ví dụ:

```shell
kubectl label namespace frontend namespace=frontend
kubectl label namespace backend namespace=backend
```

Thêm các label vào phần `namespaceSelector` trong tài liệu NetworkPolicy của bạn. Ví dụ:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-namespaces
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchExpressions:
        - key: namespace
          operator: In
          values: ["frontend", "backend"]
```

> **Ghi chú:**
> Không thể chỉ định trực tiếp tên của các namespace trong một NetworkPolicy.
> Bạn phải dùng `namespaceSelector` với `matchLabels` hoặc `matchExpressions` để chọn
> các namespace dựa trên label của chúng.

## Nhắm đến một Namespace theo tên (Targeting a Namespace by its name)

Control plane của Kubernetes đặt một label bất biến (immutable) `kubernetes.io/metadata.name`
trên tất cả các namespace, giá trị của label này là tên của namespace.

Mặc dù NetworkPolicy không thể nhắm đến một namespace theo tên thông qua một trường object nào đó,
bạn có thể dùng label chuẩn hóa này để nhắm đến một namespace cụ thể.

## Vòng đời Pod (Pod lifecycle)

> **Ghi chú:**
> Phần sau đây áp dụng cho các cluster có network plugin tuân thủ chuẩn (conformant) và có
> cách hiện thực NetworkPolicy tuân thủ chuẩn.

Khi một object NetworkPolicy mới được tạo, network plugin có thể cần một khoảng thời gian để
xử lý object mới đó. Nếu một pod chịu ảnh hưởng bởi NetworkPolicy được tạo ra trước khi network
plugin hoàn tất việc xử lý NetworkPolicy, pod đó có thể được khởi động mà không được bảo vệ,
và các quy tắc cô lập sẽ được áp dụng khi việc xử lý NetworkPolicy hoàn tất.

Một khi NetworkPolicy đã được network plugin xử lý,

1. Tất cả các pod mới tạo chịu ảnh hưởng bởi một NetworkPolicy nhất định sẽ bị cô lập trước
   khi chúng được khởi động.
   Các hiện thực của NetworkPolicy phải bảo đảm việc lọc có hiệu lực trong suốt vòng đời của
   Pod, ngay từ khoảnh khắc đầu tiên bất kỳ container nào trong Pod đó được khởi động.
   Vì được áp dụng ở cấp Pod, NetworkPolicy áp dụng như nhau cho các init container,
   sidecar container và container thông thường.

1. Các quy tắc cho phép cuối cùng sẽ được áp dụng sau các quy tắc cô lập (hoặc có thể được áp
   dụng cùng lúc). Trong trường hợp xấu nhất, một pod mới tạo có thể hoàn toàn không có kết nối
   mạng nào khi mới được khởi động, nếu các quy tắc cô lập đã được áp dụng nhưng chưa có quy tắc
   cho phép nào được áp dụng.

Mọi NetworkPolicy được tạo cuối cùng đều sẽ được một network plugin xử lý, nhưng không có cách
nào để biết chính xác thời điểm đó từ Kubernetes API.

Do đó, các pod phải có khả năng chống chịu (resilient) khi bị khởi động với kết nối mạng khác
so với dự kiến. Nếu bạn cần bảo đảm pod có thể tiếp cận được những đích nhất định trước khi
khởi động, bạn có thể dùng một [init container](./50-init-containers-vi.md)
để chờ những đích đó có thể tiếp cận được trước khi kubelet khởi động các container ứng dụng.

Mọi NetworkPolicy cuối cùng đều sẽ được áp dụng lên tất cả các pod được chọn.
Vì network plugin có thể hiện thực NetworkPolicy theo cách phân tán, có khả năng các pod sẽ
thấy một góc nhìn hơi thiếu nhất quán về các network policy khi pod mới được tạo, hoặc khi các
pod hay chính sách thay đổi.
Ví dụ, một pod mới tạo lẽ ra phải tiếp cận được cả Pod A trên Node 1 lẫn Pod B trên Node 2 có
thể thấy rằng nó tiếp cận được Pod A ngay lập tức, nhưng vài giây sau mới tiếp cận được Pod B.

## NetworkPolicy và các pod `hostNetwork` (NetworkPolicy and `hostNetwork` pods)

Hành vi của NetworkPolicy đối với các pod `hostNetwork` là không được định nghĩa, nhưng nó
được giới hạn trong 2 khả năng:

- Network plugin có thể phân biệt traffic của pod `hostNetwork` với tất cả các traffic khác
  (bao gồm cả khả năng phân biệt traffic từ các pod `hostNetwork` khác nhau trên cùng một node),
  và sẽ áp dụng NetworkPolicy cho các pod `hostNetwork` giống hệt như cách nó áp dụng cho các
  pod dùng mạng pod (pod-network).
- Network plugin không thể phân biệt đúng đắn traffic của pod `hostNetwork`,
  và vì vậy nó bỏ qua các pod `hostNetwork` khi so khớp `podSelector` và `namespaceSelector`.
  Traffic đến/đi từ các pod `hostNetwork` được xử lý giống như mọi traffic khác đến/đi từ IP
  của node. (Đây là cách hiện thực phổ biến nhất.)

Điều này áp dụng khi

1. một pod `hostNetwork` được chọn bởi `spec.podSelector`.
   
   ```yaml
     ...
     spec:
       podSelector:
         matchLabels:
           role: client
     ...
   ```
 
1. một pod `hostNetwork` được chọn bởi một `podSelector` hoặc `namespaceSelector` trong một quy tắc `ingress` hoặc `egress`.

   ```yaml
     ...
     ingress:
       - from:
         - podSelector:
             matchLabels:
               role: client
     ...
   ```

Đồng thời, vì các pod `hostNetwork` có cùng địa chỉ IP với node mà chúng nằm trên đó,
các kết nối của chúng sẽ được xử lý như các kết nối của node. Ví dụ, bạn có thể cho phép traffic
từ một Pod `hostNetwork` bằng một quy tắc `ipBlock`.

## Những gì bạn không thể làm với network policy (ít nhất là chưa) (What you can't do with network policies (at least, not yet))

Tính đến Kubernetes v1.36, các chức năng sau chưa tồn tại trong NetworkPolicy API, nhưng bạn có
thể hiện thực các giải pháp thay thế bằng các thành phần của hệ điều hành (như SELinux,
OpenVSwitch, IPTables, v.v.) hoặc các công nghệ tầng 7 (Ingress controller, các hiện thực
Service Mesh) hoặc các admission controller. Trong trường hợp bạn mới làm quen với bảo mật mạng
trong Kubernetes, đáng lưu ý rằng những User Story sau đây (chưa) thể hiện thực được bằng
NetworkPolicy API.

- Buộc traffic nội bộ cluster phải đi qua một gateway chung (điều này có thể được đáp ứng tốt
  nhất bằng service mesh hoặc proxy khác).
- Bất cứ điều gì liên quan đến TLS (hãy dùng service mesh hoặc ingress controller cho việc này).
- Các chính sách theo từng node cụ thể (bạn có thể dùng ký pháp CIDR cho việc này, nhưng bạn
  không thể nhắm đến các node theo định danh Kubernetes của chúng một cách cụ thể).
- Nhắm đến các service theo tên (tuy nhiên, bạn có thể nhắm đến các pod hoặc namespace theo
  label của chúng, đây thường là một giải pháp thay thế khả thi).
- Tạo hoặc quản lý các "Policy request" (yêu cầu chính sách) được thực hiện bởi bên thứ ba.
- Các chính sách mặc định được áp dụng cho tất cả các namespace hoặc pod (có một số bản phân
  phối Kubernetes và dự án bên thứ ba có thể làm điều này).
- Công cụ truy vấn chính sách nâng cao và kiểm tra khả năng kết nối (reachability).
- Khả năng ghi log các sự kiện bảo mật mạng (ví dụ các kết nối bị chặn hoặc được chấp nhận).
- Khả năng khai báo tường minh các chính sách từ chối (hiện tại mô hình của NetworkPolicy là
  từ chối theo mặc định, chỉ có khả năng thêm các quy tắc cho phép).
- Khả năng chặn traffic loopback hoặc traffic đến từ host (Pod hiện không thể chặn truy cập
  localhost, cũng không có khả năng chặn truy cập từ node mà nó đang nằm trên đó).

## Ảnh hưởng của NetworkPolicy lên các kết nối hiện có (NetworkPolicy's impact on existing connections)

Khi tập hợp các NetworkPolicy áp dụng cho một kết nối hiện có thay đổi — điều này có thể xảy ra
do thay đổi trong các NetworkPolicy hoặc do các label liên quan của các namespace/pod được chọn
bởi chính sách (cả chủ thể lẫn các peer) bị thay đổi giữa lúc kết nối đang tồn tại — thì việc
thay đổi đó có hiệu lực đối với kết nối hiện có hay không là do từng hiện thực quyết định
(implementation defined).
Ví dụ: Một chính sách được tạo ra dẫn đến việc từ chối một kết nối trước đó được cho phép, thì
hiện thực của network plugin bên dưới chịu trách nhiệm xác định chính sách mới đó có đóng các
kết nối hiện có hay không.
Khuyến nghị không nên sửa đổi các chính sách/pod/namespace theo những cách có thể ảnh hưởng
đến các kết nối hiện có.

## Tiếp theo (What's next)

- Xem bài hướng dẫn từng bước [Khai báo Network Policy](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)
  để có thêm ví dụ.
- Xem thêm các [công thức (recipes)](https://github.com/ahmetb/kubernetes-network-policy-recipes)
  cho các kịch bản phổ biến mà tài nguyên NetworkPolicy hỗ trợ.
