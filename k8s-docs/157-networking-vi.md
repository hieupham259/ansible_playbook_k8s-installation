# Mạng trong cluster (Cluster Networking)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/cluster-administration/networking/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), bài 14/16 · Kiểm chứng
ở Lab 5b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Đây là bài đầu của nhánh **Tầng hạ tầng mạng của cluster** — mô hình mạng nhìn từ góc quản trị
viên chứ không phải người viết ứng dụng. Bài rất ngắn và phần lớn là tổng hợp lại thứ bạn vừa
học ở chín bài trước. Đọc nó như **bản đồ để kiểm tra mình có lỗ hổng nào không**, rồi mới sang
bài CNI.

**Phải hiểu ở lần đọc này:**

- Bốn bài toán mạng và nơi giải từng bài: container-với-container trong Pod (giải bằng Pod và
  `localhost`), **Pod-với-Pod** (trọng tâm của bài này), Pod-với-Service và từ ngoài vào Service
  (giải trong tài liệu Service).
- Vì sao Kubernetes chọn "mỗi Pod một IP": điều phối port giữa nhiều nhà phát triển **không mở
  rộng được**, còn cấp phát port động thì kéo theo phức tạp cho toàn hệ thống — ứng dụng phải
  nhận port qua flag, API server phải chèn số port vào cấu hình, các service phải tìm được nhau.
- **Ba dải IP không chồng lấn** và ai cấp phát mỗi dải: network plugin gán IP cho **Pod**,
  kube-apiserver gán IP cho **Service**, kubelet hoặc cloud-controller-manager gán IP cho
  **Node**.
- Cluster được phân loại theo họ IP (chỉ IPv4, chỉ IPv6, dual-stack) và **tất cả các thành phần
  phải thống nhất về họ IP chính**. Kubernetes chỉ xét các địa chỉ có trong
  `node.status.addresses` và `pod.status.ips`, **không** xét mọi IP thật trên các interface.
- Mô hình mạng được hiện thực bởi **container runtime trên mỗi node**, thường thông qua CNI
  plugin; mức độ tính năng giữa các plugin chênh nhau rất nhiều, từ chỉ thêm/gỡ interface cho
  tới IPAM nâng cao.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Danh sách addon mạng được liên kết ở mục *Cách hiện thực mô hình mạng Kubernetes* | là danh mục để chọn CNI, chưa phải lúc chọn | bài [183](183-network-plugins-vi.md), Lab 5b |
| Chi tiết cấu hình cho cluster IPv6-only và dual-stack | lab là single-stack IPv4 | bài [85](85-dual-stack-vi.md) |
| Mục *Tiếp theo* — tài liệu thiết kế mạng và các KEP của SIG-Network | là nguồn tham khảo sâu và kế hoạch tương lai | không cần |

---

Mạng (networking) là một phần trung tâm của Kubernetes, nhưng việc hiểu chính xác
nó được kỳ vọng hoạt động như thế nào có thể là một thách thức. Có 4 bài toán mạng
riêng biệt cần giải quyết:

1. Giao tiếp container-với-container có độ gắn kết cao: bài toán này được giải quyết bằng
   Pod và giao tiếp qua `localhost`.
2. Giao tiếp Pod-với-Pod: đây là trọng tâm chính của tài liệu này.
3. Giao tiếp Pod-với-Service: bài toán này được đề cập trong [Services](82-service-vi.md).
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

Để tìm hiểu về mô hình mạng của Kubernetes, xem [tại đây](81-services-networking-vi.md).

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
- IPv4/IPv6 hoặc IPv6/IPv4 [dual-stack](85-dual-stack-vi.md):
  - Network plugin được cấu hình để gán cả địa chỉ IPv4 và IPv6.
  - kube-apiserver được cấu hình để gán cả địa chỉ IPv4 và IPv6.
  - kubelet hoặc cloud-controller-manager được cấu hình để gán cả địa chỉ IPv4 và IPv6.
  - Tất cả các thành phần phải thống nhất về họ IP chính (primary IP family) đã được cấu hình.

Cluster Kubernetes chỉ xem xét các họ IP hiện diện trên các đối tượng Pod, Service và Node,
độc lập với các địa chỉ IP thực có của những đối tượng được biểu diễn. Ví dụ, một server hoặc một pod có thể có nhiều
địa chỉ IP được gán trên các interface của nó, nhưng chỉ các địa chỉ IP trong `node.status.addresses` hoặc `pod.status.ips` mới
được xem xét khi hiện thực hóa mô hình mạng Kubernetes và xác định kiểu của cluster.

## Cách hiện thực mô hình mạng Kubernetes (How to implement the Kubernetes network model) {#how-to-implement-the-kubernetes-network-model}

Mô hình mạng được hiện thực bởi container runtime trên mỗi node. Các container runtime
phổ biến nhất sử dụng các plugin [Container Network Interface](https://github.com/containernetworking/cni) (CNI)
để quản lý các khả năng mạng và bảo mật của chúng. Có rất nhiều plugin CNI khác nhau đến từ
nhiều nhà cung cấp khác nhau. Một số plugin chỉ cung cấp các tính năng cơ bản như thêm và gỡ bỏ
network interface, trong khi số khác cung cấp những giải pháp tinh vi hơn, chẳng hạn như tích hợp với
các hệ thống điều phối container (container orchestration) khác, chạy nhiều plugin CNI cùng lúc, các tính năng IPAM nâng cao, v.v.

Xem [trang này](165-addons-vi.md#networking-and-network-policy)
để có danh sách (không đầy đủ) các addon mạng được Kubernetes hỗ trợ.

## Tiếp theo (What's next)

Thiết kế ban đầu của mô hình mạng cùng cơ sở lý luận của nó được mô tả chi tiết hơn trong
[tài liệu thiết kế mạng](https://git.k8s.io/design-proposals-archive/network/networking.md).
Về các kế hoạch trong tương lai và một số nỗ lực đang diễn ra nhằm cải thiện mạng Kubernetes,
vui lòng tham khảo các [KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-network)
của SIG-Network.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5:

1. Cluster lab dùng Pod CIDR `10.244.0.0/16`, Service CIDR `10.96.0.0/12`, còn ba node nằm
   trong một dải LAN riêng. Thành phần nào cấp phát IP cho mỗi loại trong ba loại đó, và vì sao
   ba dải không được chồng lấn?
2. Một node có nhiều địa chỉ IP trên nhiều interface. Kubernetes coi node đó có những địa chỉ
   nào khi hiện thực mô hình mạng?
3. Bài liệt kê bốn bài toán mạng. Bài này giải bài toán nào, và ba bài toán còn lại được giải ở
   đâu?
4. Vì sao Kubernetes không đi theo hướng cấp phát port động trên host?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Network plugin** cấp IP cho **Pod** (dải `10.244.0.0/16`); **kube-apiserver** cấp IP cho
   **Service** (dải `10.96.0.0/12`); **kubelet hoặc cloud-controller-manager** cấp IP cho
   **Node**. Bài nêu thẳng yêu cầu: cluster cần cấp phát các địa chỉ IP **không chồng lấn** cho
   Pod, Service và Node — chồng lấn thì một địa chỉ đích trở nên nhập nhằng và không thể xác
   định gói tin phải đi tới đâu.
2. **Chỉ những địa chỉ nằm trong `node.status.addresses`** (và với Pod là `pod.status.ips`).
   Bài nói rõ: một server hay một pod có thể có nhiều địa chỉ IP trên các interface của nó,
   nhưng **chỉ các địa chỉ trong hai trường trạng thái đó mới được xem xét** khi hiện thực hóa
   mô hình mạng và xác định kiểu của cluster. Nhìn `ip addr` trên node không phải là nhìn cái
   Kubernetes đang dùng.
3. Bài này giải **giao tiếp Pod-với-Pod**. Ba bài toán còn lại: container-với-container có độ
   gắn kết cao được giải bằng **Pod và giao tiếp qua `localhost`**; **Pod-với-Service** và
   **từ bên ngoài tới Service** đều được đề cập trong tài liệu về Service.
4. Vì việc chia sẻ máy giữa các ứng dụng đòi hỏi bảo đảm hai ứng dụng không dùng trùng port, mà
   **điều phối port giữa nhiều nhà phát triển rất khó thực hiện ở quy mô lớn** và đẩy người dùng
   vào các vấn đề cấp cluster nằm ngoài tầm kiểm soát của họ. Cấp phát port động thì **mang rất
   nhiều phức tạp vào hệ thống**: mọi ứng dụng phải nhận port dưới dạng flag, API server phải
   biết chèn số port động vào các khối cấu hình, các service phải biết cách tìm thấy nhau.
   Kubernetes chọn cách khác — mỗi Pod một IP riêng.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
