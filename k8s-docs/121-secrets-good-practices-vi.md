# Các thực hành tốt cho Kubernetes Secrets (Good practices for Kubernetes Secrets)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/secrets-good-practices/>
>
> Các nguyên tắc và thực hành quản lý Secret tốt dành cho quản trị viên cluster và nhà phát triển ứng dụng.

Trong Kubernetes, Secret là một đối tượng lưu trữ thông tin nhạy cảm, chẳng hạn như mật khẩu,
token OAuth và khóa SSH.

Secret giúp bạn kiểm soát tốt hơn cách thông tin nhạy cảm được sử dụng và giảm
rủi ro lộ lọt do vô ý. Các giá trị của Secret được mã hóa (encode) dưới dạng chuỗi base64 và
mặc định được lưu trữ không mã hóa, nhưng có thể được cấu hình để
[mã hóa khi lưu trữ (encrypted at rest)](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#ensure-all-secrets-are-encrypted).

Một Pod có thể tham chiếu Secret theo nhiều cách khác nhau, chẳng hạn như trong một volume mount
hoặc như một biến môi trường. Secret được thiết kế cho dữ liệu bí mật, còn
[ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) được
thiết kế cho dữ liệu không bí mật.

Các thực hành tốt sau đây dành cho cả quản trị viên cluster lẫn
nhà phát triển ứng dụng. Hãy dùng các hướng dẫn này để cải thiện độ an toàn cho
thông tin nhạy cảm của bạn trong các đối tượng Secret, cũng như để quản lý
các Secret của bạn hiệu quả hơn.

## Quản trị viên cluster (Cluster administrators)

Phần này cung cấp các thực hành tốt mà quản trị viên cluster có thể áp dụng để
cải thiện độ an toàn cho thông tin bí mật trong cluster.

### Cấu hình mã hóa khi lưu trữ (Configure encryption at rest)

Theo mặc định, các đối tượng Secret được lưu không mã hóa trong etcd. Bạn nên cấu hình
mã hóa cho dữ liệu Secret của mình trong `etcd`. Để biết hướng dẫn, tham khảo
[Mã hóa dữ liệu Secret khi lưu trữ (Encrypt Secret Data at Rest)](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/).

### Cấu hình truy cập đặc quyền tối thiểu cho Secret (Configure least-privilege access to Secrets) {#least-privilege-secrets}

Khi lập kế hoạch cho cơ chế kiểm soát truy cập của bạn, chẳng hạn như
Kiểm soát truy cập dựa trên vai trò (Role-based Access Control) [(RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) của Kubernetes,
hãy cân nhắc các hướng dẫn sau về truy cập các đối tượng `Secret`. Bạn cũng nên
tuân theo các hướng dẫn khác trong
[Các thực hành tốt về RBAC](./120-rbac-good-practices-vi.md).

- **Các thành phần hệ thống (Components)**: Hạn chế quyền `watch` hoặc `list` chỉ cho các thành phần
  cấp hệ thống có đặc quyền cao nhất. Chỉ cấp quyền `get` đối với Secret nếu
  hành vi bình thường của thành phần đó yêu cầu.
- **Con người (Humans)**: Hạn chế quyền `get`, `watch` hoặc `list` đối với Secret. Chỉ cho phép
  quản trị viên cluster truy cập `etcd`. Điều này bao gồm cả quyền truy cập chỉ đọc. Với
  các nhu cầu kiểm soát truy cập phức tạp hơn, chẳng hạn hạn chế truy cập vào các Secret có
  annotation cụ thể, hãy cân nhắc sử dụng các cơ chế phân quyền của bên thứ ba.

> **Thận trọng:**
>
> Việc cấp quyền `list` đối với Secret ngầm cho phép chủ thể đó lấy được
> nội dung của các Secret.

Một người dùng có thể tạo Pod sử dụng một Secret cũng có thể xem được giá trị của
Secret đó. Ngay cả khi các chính sách của cluster không cho phép người dùng đọc Secret
một cách trực tiếp, chính người dùng đó vẫn có thể có quyền chạy một Pod mà sau đó làm lộ
Secret. Bạn có thể phát hiện hoặc hạn chế tác động do dữ liệu Secret bị lộ,
dù cố ý hay vô ý, bởi một người dùng có quyền truy cập này. Một số
khuyến nghị bao gồm:

*  Sử dụng các Secret có thời gian sống ngắn (short-lived)
*  Triển khai các quy tắc kiểm toán (audit rules) cảnh báo về các sự kiện cụ thể, chẳng hạn việc
   một người dùng đồng thời đọc nhiều Secret

#### Hạn chế truy cập đối với Secret (Restrict Access for Secrets)

Sử dụng các namespace riêng biệt để cách ly quyền truy cập vào các secret được mount.

### Cải thiện các chính sách quản lý etcd (Improve etcd management policies)

Cân nhắc xóa sạch (wipe) hoặc hủy dữ liệu (shred) trên thiết bị lưu trữ bền vững mà `etcd` từng sử dụng khi nó
không còn được dùng nữa.

Nếu có nhiều instance `etcd`, hãy cấu hình giao tiếp SSL/TLS được mã hóa
giữa các instance để bảo vệ dữ liệu Secret khi truyền trên đường mạng (in transit).

### Cấu hình truy cập tới Secret bên ngoài (Configure access to external Secrets)

> **Ghi chú:** Mục này đề cập đến các sản phẩm hoặc dự án của bên thứ ba cung cấp chức năng mà Kubernetes cần.
> Các tác giả của dự án Kubernetes không chịu trách nhiệm về những sản phẩm hoặc dự án bên thứ ba đó.
> Xem [hướng dẫn trên website CNCF](https://github.com/cncf/foundation/blob/main/website-guidelines.md) để biết thêm chi tiết.

Bạn có thể sử dụng các nhà cung cấp kho lưu trữ Secret bên thứ ba để giữ dữ liệu bí mật của bạn
bên ngoài cluster, rồi cấu hình các Pod truy cập thông tin đó.
[Kubernetes Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/)
là một DaemonSet cho phép kubelet lấy Secret từ các kho lưu trữ bên ngoài, và
mount các Secret đó dưới dạng volume vào những Pod cụ thể mà bạn cho phép truy cập
dữ liệu.

Để xem danh sách các nhà cung cấp được hỗ trợ, tham khảo
[Providers for the Secret Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/concepts.html#provider-for-the-secrets-store-csi-driver).

## Các thực hành tốt khi sử dụng bộ nhớ swap (Good practices for using swap memory)

Để biết các thực hành tốt nhất khi thiết lập bộ nhớ swap cho các node Linux, vui lòng tham khảo
[quản lý bộ nhớ swap (swap memory management)](https://kubernetes.io/docs/concepts/cluster-administration/swap-memory-management/#good-practice-for-using-swap-in-a-kubernetes-cluster).

## Nhà phát triển (Developers)

Phần này cung cấp các thực hành tốt dành cho nhà phát triển nhằm cải thiện
độ an toàn của dữ liệu bí mật khi xây dựng và triển khai các tài nguyên Kubernetes.

### Giới hạn quyền truy cập Secret cho các container cụ thể (Restrict Secret access to specific containers)

Nếu bạn định nghĩa nhiều container trong một Pod, và chỉ một trong các
container đó cần truy cập một Secret, hãy định nghĩa cấu hình volume mount hoặc biến
môi trường sao cho các container còn lại không có quyền truy cập vào
Secret đó.

### Bảo vệ dữ liệu Secret sau khi đọc (Protect Secret data after reading)

Các ứng dụng vẫn cần bảo vệ giá trị của thông tin bí mật sau khi
đọc nó từ một biến môi trường hoặc volume. Ví dụ, ứng dụng của bạn
phải tránh ghi log dữ liệu secret ở dạng rõ (in the clear) hoặc truyền nó
cho một bên không đáng tin cậy.

### Tránh chia sẻ manifest của Secret (Avoid sharing Secret manifests)

Nếu bạn cấu hình một Secret thông qua một
manifest, với dữ liệu secret được
mã hóa (encode) dưới dạng base64, việc chia sẻ file này hoặc đưa (check in) nó vào một kho mã
nguồn đồng nghĩa với việc secret đó sẵn sàng cho tất cả những ai có thể đọc manifest.

> **Thận trọng:**
>
> Mã hóa base64 _không phải_ là một phương pháp mã hóa bảo mật (encryption); nó không cung cấp thêm
> bất kỳ tính bảo mật nào so với văn bản thuần (plain text).
