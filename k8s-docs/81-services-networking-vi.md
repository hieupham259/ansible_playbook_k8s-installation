# Service, cân bằng tải và mạng (Services, Load Balancing, and Networking)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/>
>
> Các khái niệm và tài nguyên đằng sau hoạt động mạng trong Kubernetes.

## Mô hình mạng của Kubernetes (The Kubernetes network model)

Mô hình mạng của Kubernetes được xây dựng từ nhiều thành phần:

* Mỗi [Pod](./46-pods-vi.md) trong một cluster nhận được một địa chỉ IP riêng,
  duy nhất trên toàn cluster.

  * Một Pod có network namespace riêng tư của nó, được chia sẻ bởi
    tất cả các container bên trong Pod đó. Các tiến trình chạy trong
    những container khác nhau thuộc cùng một Pod có thể giao tiếp với
    nhau qua `localhost`.

* _Mạng Pod_ (pod network, còn gọi là mạng cluster) xử lý việc giao tiếp
  giữa các Pod. Nó đảm bảo rằng (trừ khi có sự phân đoạn mạng có chủ đích):

  * Tất cả các Pod có thể giao tiếp với mọi Pod khác, dù chúng nằm
    trên cùng một [node](./23-nodes-vi.md) hay trên các node khác nhau.
    Các Pod có thể giao tiếp trực tiếp với nhau,
    mà không cần dùng proxy hay chuyển đổi địa chỉ mạng (NAT).

    Trên Windows, quy tắc này không áp dụng cho các Pod dùng host-network.

  * Các agent trên một node (chẳng hạn các system daemon, hoặc kubelet)
    có thể giao tiếp với tất cả các Pod trên node đó.

* [Service](https://kubernetes.io/docs/concepts/services-networking/service/) API
  cho phép bạn cung cấp một địa chỉ IP hoặc hostname ổn định (tồn tại lâu dài) cho một dịch vụ
  được hiện thực bởi một hoặc nhiều Pod backend, trong đó từng Pod
  cấu thành dịch vụ có thể thay đổi theo thời gian.

  * Kubernetes tự động quản lý các đối tượng
    [EndpointSlice](./83-endpoint-slices-vi.md)
    để cung cấp thông tin về các Pod hiện đang đứng sau một Service.

  * Một hiện thực service proxy giám sát tập các đối tượng Service và
    EndpointSlice, và lập trình data plane để định tuyến
    lưu lượng của Service tới các backend của nó, bằng cách dùng các API của
    hệ điều hành hoặc của nhà cung cấp đám mây (cloud provider) để chặn hoặc ghi lại các gói tin.

* [Gateway](./13-gateway-vi.md) API
  (hoặc tiền thân của nó là [Ingress](./11-ingress-vi.md))
  cho phép bạn làm cho các Service có thể truy cập được bởi các client nằm ngoài cluster.

  * Một cơ chế đơn giản hơn nhưng ít khả năng cấu hình hơn cho việc
    đi vào cluster (cluster ingress) có sẵn qua
    [`type: LoadBalancer`](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)
    của Service API, khi dùng một nhà cung cấp đám mây (cloud provider) được hỗ trợ.

* [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies) là một
  API tích hợp sẵn của Kubernetes cho phép bạn kiểm soát lưu lượng giữa các Pod, hoặc giữa các Pod và
  thế giới bên ngoài.

Trong các hệ thống container cũ, không có kết nối tự động
giữa các container trên những host khác nhau, vì vậy thường phải
tạo liên kết (link) giữa các container một cách tường minh, hoặc ánh xạ port của container
sang port của host để các container trên host khác có thể
truy cập được chúng. Điều này không cần thiết trong Kubernetes; mô hình của Kubernetes là
các Pod có thể được đối xử gần giống như các VM hay host vật lý xét theo
các khía cạnh cấp phát port, đặt tên, khám phá dịch vụ (service discovery), cân bằng tải
(load balancing), cấu hình ứng dụng, và di trú (migration).

Chỉ một vài phần của mô hình này được chính Kubernetes hiện thực.
Với các phần còn lại, Kubernetes định nghĩa các API, còn
chức năng tương ứng do các thành phần bên ngoài cung cấp, một số
trong đó là tùy chọn:

* Việc thiết lập network namespace cho Pod được xử lý bởi phần mềm cấp hệ thống hiện thực
  [Container Runtime Interface](./44-cri-vi.md).

* Bản thân mạng Pod được quản lý bởi một
  [hiện thực mạng Pod](https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy).
  Trên Linux, hầu hết các container runtime dùng
  Container Networking Interface (CNI)
  để tương tác với hiện thực mạng Pod, vì vậy các hiện thực
  này thường được gọi là các _CNI plugin_.

* Kubernetes cung cấp một hiện thực mặc định cho service proxy,
  gọi là kube-proxy, nhưng một số hiện thực mạng Pod
  lại dùng service proxy riêng của chúng, được tích hợp
  chặt chẽ hơn với phần còn lại của hiện thực đó.

* NetworkPolicy nói chung cũng được hiện thực bởi hiện thực
  mạng Pod. (Một số hiện thực mạng Pod đơn giản không
  hiện thực NetworkPolicy, hoặc quản trị viên có thể chọn
  cấu hình mạng Pod không có hỗ trợ NetworkPolicy. Trong những
  trường hợp này, API vẫn sẽ tồn tại, nhưng sẽ không có tác dụng.)

* Có nhiều [hiện thực của Gateway API](https://gateway-api.sigs.k8s.io/implementations/),
  một số dành riêng cho các môi trường đám mây cụ thể, một số
  tập trung hơn vào các môi trường "bare metal", và số khác mang tính tổng quát hơn.

## Tiếp theo (What's next)

Hướng dẫn thực hành [Kết nối ứng dụng với Service (Connecting Applications with Services)](https://kubernetes.io/docs/tutorials/services/connect-applications-service/)
giúp bạn tìm hiểu về Service và mạng Kubernetes qua một ví dụ thực hành.

[Mạng cluster (Cluster Networking)](https://kubernetes.io/docs/concepts/cluster-administration/networking/) giải thích cách thiết lập
mạng cho cluster của bạn, đồng thời cung cấp cái nhìn tổng quan về các công nghệ liên quan.

Để tìm hiểu về các khái niệm mạng cụ thể, xem:

* [Service](https://kubernetes.io/docs/concepts/services-networking/service/) - phơi bày (expose) một ứng dụng phía sau một endpoint duy nhất hướng ra bên ngoài
* [Ingress](./11-ingress-vi.md) - định tuyến HTTP/HTTPS có nhận biết giao thức, dựa trên URI, hostname và path
* [Gateway API](./13-gateway-vi.md) - cấp phát (provision) hạ tầng động và định tuyến lưu lượng nâng cao
* [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) - kiểm soát luồng lưu lượng ở mức địa chỉ IP hoặc port (tầng OSI 3 hoặc 4)
* [DNS cho Service và Pod](./10-dns-pod-service-vi.md) - khám phá các dịch vụ bên trong cluster của bạn bằng DNS
