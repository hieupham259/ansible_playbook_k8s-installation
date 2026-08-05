# Mạng trong cluster (Cluster Networking)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/cluster-administration/networking/>

Mạng (networking) là một phần trung tâm của Kubernetes, nhưng việc hiểu chính xác
nó được kỳ vọng hoạt động như thế nào có thể là một thách thức. Có 4 bài toán mạng
riêng biệt cần giải quyết:

1. Giao tiếp container-với-container có độ gắn kết cao: bài toán này được giải quyết bằng
   Pod và giao tiếp qua `localhost`.
2. Giao tiếp Pod-với-Pod: đây là trọng tâm chính của tài liệu này.
3. Giao tiếp Pod-với-Service: bài toán này được đề cập trong [Services](https://kubernetes.io/docs/concepts/services-networking/service/).
4. Giao tiếp từ bên ngoài tới Service: bài toán này cũng được đề cập trong Services.

Kubernetes xoay quanh việc chia sẻ máy (machine) giữa các ứng dụng. Thông thường,
việc chia sẻ máy đòi hỏi phải đảm bảo rằng hai ứng dụng không cố sử dụng cùng
một port. Việc điều phối port giữa nhiều nhà phát triển là rất khó thực hiện
ở quy mô lớn và khiến người dùng phải đối mặt với các vấn đề ở cấp cluster nằm ngoài tầm kiểm soát của họ.

Việc cấp phát port động (dynamic port allocation) mang lại rất nhiều phức tạp cho hệ thống - mọi
ứng dụng phải nhận port dưới dạng flag, các API server phải biết cách
chèn số port động vào các khối cấu hình, các service phải biết
cách tìm thấy nhau, v.v. Thay vì xử lý những điều này, Kubernetes chọn một
cách tiếp cận khác.

Để tìm hiểu về mô hình mạng của Kubernetes, xem [tại đây](https://kubernetes.io/docs/concepts/services-networking/).

## Các dải địa chỉ IP của Kubernetes (Kubernetes IP address ranges)

Cluster Kubernetes cần cấp phát các địa chỉ IP không chồng lấn cho Pod, Service và Node,
từ một dải địa chỉ khả dụng được cấu hình trong các thành phần sau:

- Network plugin được cấu hình để gán địa chỉ IP cho các Pod.
- kube-apiserver được cấu hình để gán địa chỉ IP cho các Service.
- kubelet hoặc cloud-controller-manager được cấu hình để gán địa chỉ IP cho các Node.

![Hình minh họa các dải mạng khác nhau trong một cluster Kubernetes](https://kubernetes.io/docs/images/kubernetes-cluster-network.svg)

## Các kiểu mạng cluster (Cluster networking types) {#cluster-network-ipfamilies}

Dựa theo các họ IP (IP family) được cấu hình, cluster Kubernetes có thể được phân loại thành:

- Chỉ IPv4: network plugin, kube-apiserver và kubelet/cloud-controller-manager được cấu hình để chỉ gán địa chỉ IPv4.
- Chỉ IPv6: network plugin, kube-apiserver và kubelet/cloud-controller-manager được cấu hình để chỉ gán địa chỉ IPv6.
- IPv4/IPv6 hoặc IPv6/IPv4 [dual-stack](https://kubernetes.io/docs/concepts/services-networking/dual-stack/):
  - Network plugin được cấu hình để gán cả địa chỉ IPv4 và IPv6.
  - kube-apiserver được cấu hình để gán cả địa chỉ IPv4 và IPv6.
  - kubelet hoặc cloud-controller-manager được cấu hình để gán cả địa chỉ IPv4 và IPv6.
  - Tất cả các thành phần phải thống nhất về họ IP chính (primary IP family) đã được cấu hình.

Cluster Kubernetes chỉ xem xét các họ IP hiện diện trên các đối tượng Pod, Service và Node,
độc lập với các địa chỉ IP thực có của những đối tượng được biểu diễn. Ví dụ, một server hoặc một pod có thể có nhiều
địa chỉ IP được gán trên các interface của nó, nhưng chỉ các địa chỉ IP trong `node.status.addresses` hoặc `pod.status.ips` mới
được xem xét khi hiện thực hóa mô hình mạng Kubernetes và xác định kiểu của cluster.

## Cách hiện thực mô hình mạng Kubernetes (How to implement the Kubernetes network model)

Mô hình mạng được hiện thực bởi container runtime trên mỗi node. Các container runtime
phổ biến nhất sử dụng các plugin [Container Network Interface](https://github.com/containernetworking/cni) (CNI)
để quản lý các khả năng mạng và bảo mật của chúng. Có rất nhiều plugin CNI khác nhau đến từ
nhiều nhà cung cấp khác nhau. Một số plugin chỉ cung cấp các tính năng cơ bản như thêm và gỡ bỏ
network interface, trong khi số khác cung cấp những giải pháp tinh vi hơn, chẳng hạn như tích hợp với
các hệ thống điều phối container (container orchestration) khác, chạy nhiều plugin CNI cùng lúc, các tính năng IPAM nâng cao, v.v.

Xem [trang này](https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy)
để có danh sách (không đầy đủ) các addon mạng được Kubernetes hỗ trợ.

## Tiếp theo (What's next)

Thiết kế ban đầu của mô hình mạng cùng cơ sở lý luận của nó được mô tả chi tiết hơn trong
[tài liệu thiết kế mạng](https://git.k8s.io/design-proposals-archive/network/networking.md).
Về các kế hoạch trong tương lai và một số nỗ lực đang diễn ra nhằm cải thiện mạng Kubernetes,
vui lòng tham khảo các [KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-network)
của SIG-Network.
