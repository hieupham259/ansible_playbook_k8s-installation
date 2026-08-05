# Quản lý object trong Kubernetes (Kubernetes Object Management)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/>

Công cụ dòng lệnh `kubectl` hỗ trợ nhiều cách khác nhau để tạo và quản lý
các [object](./16-working-with-objects-vi.md) của Kubernetes. Tài liệu này cung cấp cái nhìn tổng quan về các
cách tiếp cận khác nhau đó. Đọc [Kubectl book](https://kubectl.docs.kubernetes.io) để biết
chi tiết về việc quản lý object bằng Kubectl.

## Các kỹ thuật quản lý (Management techniques)

> **Cảnh báo:** Một object Kubernetes chỉ nên được quản lý bằng một kỹ thuật duy nhất. Trộn lẫn
> nhiều kỹ thuật cho cùng một object sẽ dẫn đến hành vi không xác định (undefined behavior).

| Kỹ thuật quản lý                  | Thao tác trên                      | Môi trường khuyến nghị        | Số người ghi (writer) hỗ trợ | Độ khó học  |
|-----------------------------------|------------------------------------|-------------------------------|------------------------------|-------------|
| Câu lệnh mệnh lệnh                | Các object đang chạy (live objects)| Dự án phát triển (development)| 1+                           | Thấp nhất   |
| Cấu hình object kiểu mệnh lệnh    | Từng file riêng lẻ                 | Dự án production              | 1                            | Trung bình  |
| Cấu hình object kiểu khai báo     | Thư mục chứa các file              | Dự án production              | 1+                           | Cao nhất    |

## Câu lệnh mệnh lệnh (Imperative commands)

Khi dùng câu lệnh mệnh lệnh (imperative commands), người dùng thao tác trực tiếp trên các object
đang chạy (live objects) trong cluster. Người dùng truyền thao tác cần thực hiện cho
lệnh `kubectl` dưới dạng đối số (argument) hoặc flag.

Đây là cách được khuyến nghị khi mới bắt đầu hoặc khi cần chạy một tác vụ một lần (one-off) trong
cluster. Vì kỹ thuật này thao tác trực tiếp trên các object
đang chạy, nó không lưu lại lịch sử của các cấu hình trước đó.

### Ví dụ (Examples)

Chạy một instance của container nginx bằng cách tạo một object Deployment:

```sh
kubectl create deployment nginx --image nginx
```

### Đánh đổi (Trade-offs)

Ưu điểm so với cấu hình object:

- Câu lệnh được diễn đạt bằng một từ chỉ hành động duy nhất.
- Câu lệnh chỉ cần một bước duy nhất để thực hiện thay đổi lên cluster.

Nhược điểm so với cấu hình object:

- Câu lệnh không tích hợp được với quy trình review thay đổi.
- Câu lệnh không cung cấp vết kiểm toán (audit trail) gắn với các thay đổi.
- Câu lệnh không cung cấp nguồn bản ghi (source of records) nào ngoài những gì đang chạy.
- Câu lệnh không cung cấp template để tạo các object mới.

## Cấu hình object kiểu mệnh lệnh (Imperative object configuration)

Trong cấu hình object kiểu mệnh lệnh, lệnh kubectl chỉ định
thao tác (create, replace, v.v.), các flag tùy chọn và ít nhất một tên
file. File được chỉ định phải chứa định nghĩa đầy đủ của object
ở định dạng YAML hoặc JSON.

Xem [tài liệu tham khảo API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/)
để biết thêm chi tiết về định nghĩa object.

> **Cảnh báo:** Lệnh mệnh lệnh `replace` thay thế spec hiện có
> bằng spec mới được cung cấp, loại bỏ mọi thay đổi trên object mà file cấu hình
> không có. Không nên dùng cách tiếp cận này với các loại resource
> có spec được cập nhật độc lập với file cấu hình.
> Ví dụ, các Service loại `LoadBalancer` có trường `externalIPs` được cluster cập nhật
> độc lập với cấu hình.

### Ví dụ (Examples)

Tạo các object được định nghĩa trong một file cấu hình:

```sh
kubectl create -f nginx.yaml
```

Xóa các object được định nghĩa trong hai file cấu hình:

```sh
kubectl delete -f nginx.yaml -f redis.yaml
```

Cập nhật các object được định nghĩa trong một file cấu hình bằng cách ghi đè
cấu hình đang chạy:

```sh
kubectl replace -f nginx.yaml
```

### Đánh đổi (Trade-offs)

Ưu điểm so với câu lệnh mệnh lệnh:

- Cấu hình object có thể được lưu trong một hệ thống quản lý mã nguồn như Git.
- Cấu hình object có thể tích hợp với các quy trình như review thay đổi trước khi push và vết kiểm toán (audit trail).
- Cấu hình object cung cấp template để tạo các object mới.

Nhược điểm so với câu lệnh mệnh lệnh:

- Cấu hình object đòi hỏi hiểu biết cơ bản về schema của object.
- Cấu hình object đòi hỏi thêm một bước là viết file YAML.

Ưu điểm so với cấu hình object kiểu khai báo:

- Hành vi của cấu hình object kiểu mệnh lệnh đơn giản và dễ hiểu hơn.
- Tính đến Kubernetes phiên bản 1.5, cấu hình object kiểu mệnh lệnh trưởng thành (mature) hơn.

Nhược điểm so với cấu hình object kiểu khai báo:

- Cấu hình object kiểu mệnh lệnh hoạt động tốt nhất trên từng file, không phải trên thư mục.
- Các cập nhật trên object đang chạy phải được phản ánh vào file cấu hình, nếu không chúng sẽ bị mất trong lần thay thế (replace) tiếp theo.

## Cấu hình object kiểu khai báo (Declarative object configuration)

Khi dùng cấu hình object kiểu khai báo (declarative object configuration), người dùng thao tác trên các file
cấu hình object được lưu cục bộ, tuy nhiên người dùng không định nghĩa
thao tác sẽ được thực hiện trên các file đó. Các thao tác tạo, cập nhật và xóa
được `kubectl` tự động phát hiện cho từng object. Điều này cho phép làm việc trên
các thư mục, nơi mỗi object khác nhau có thể cần một thao tác khác nhau.

> **Ghi chú:** Cấu hình object kiểu khai báo giữ lại các thay đổi do những người ghi (writer) khác
> thực hiện, ngay cả khi các thay đổi đó không được merge ngược lại vào file cấu hình object.
> Điều này khả thi nhờ dùng thao tác API `patch` để chỉ ghi
> những khác biệt quan sát được, thay vì dùng thao tác API `replace`
> để thay thế toàn bộ cấu hình object.

### Ví dụ (Examples)

Xử lý tất cả các file cấu hình object trong thư mục `configs`, rồi tạo hoặc
patch các object đang chạy. Bạn có thể chạy `diff` trước để xem những thay đổi sắp được
thực hiện, sau đó mới apply:

```sh
kubectl diff -f configs/
kubectl apply -f configs/
```

Xử lý đệ quy các thư mục:

```sh
kubectl diff -R -f configs/
kubectl apply -R -f configs/
```

### Đánh đổi (Trade-offs)

Ưu điểm so với cấu hình object kiểu mệnh lệnh:

- Các thay đổi được thực hiện trực tiếp trên object đang chạy được giữ lại, ngay cả khi chúng không được merge ngược vào các file cấu hình.
- Cấu hình object kiểu khai báo hỗ trợ tốt hơn việc thao tác trên thư mục và tự động phát hiện loại thao tác (create, patch, delete) cho từng object.

Nhược điểm so với cấu hình object kiểu mệnh lệnh:

- Cấu hình object kiểu khai báo khó debug hơn, và khi kết quả không như mong đợi thì khó hiểu hơn.
- Cập nhật một phần (partial update) bằng diff tạo ra các thao tác merge và patch phức tạp.

## Tiếp theo (What's next)

- [Quản lý object Kubernetes bằng câu lệnh mệnh lệnh](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-command/)
- [Quản lý object Kubernetes kiểu mệnh lệnh bằng file cấu hình](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-config/)
- [Quản lý object Kubernetes kiểu khai báo bằng file cấu hình](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)
- [Quản lý object Kubernetes kiểu khai báo bằng Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [Tài liệu tham khảo lệnh Kubectl](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/)
- [Kubectl Book](https://kubectl.docs.kubernetes.io)
- [Tài liệu tham khảo Kubernetes API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/)
