# Annotations

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/>

Bạn có thể dùng annotation của Kubernetes để gắn metadata tùy ý, không dùng để định danh
(non-identifying), vào các object.
Các client như công cụ và thư viện có thể truy xuất metadata này.

## Gắn metadata vào object (Attaching metadata to objects)

Bạn có thể dùng label hoặc annotation để gắn metadata vào các object của
Kubernetes. Label có thể được dùng để chọn (select) object và để tìm
các tập hợp object thỏa mãn những điều kiện nhất định. Ngược lại, annotation
không được dùng để định danh và chọn object. Metadata
trong một annotation có thể nhỏ hoặc lớn, có cấu trúc hoặc phi cấu trúc, và có thể
chứa các ký tự mà label không cho phép. Bạn có thể dùng cả label
lẫn annotation trong metadata của cùng một object.

Annotation, giống như label, là các map key/value:

```json
"metadata": {
  "annotations": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

> **Ghi chú:** Các key và value trong map phải là chuỗi (string). Nói cách khác, bạn không thể dùng
> kiểu số, boolean, danh sách hay các kiểu khác cho key hoặc value.

Dưới đây là một số ví dụ về thông tin có thể được ghi lại trong annotation:

* Các field được quản lý bởi một lớp cấu hình khai báo (declarative configuration layer).
  Việc gắn các field này dưới dạng annotation giúp phân biệt chúng với các giá trị mặc định
  do client hoặc server đặt, cũng như với các field được sinh tự động và các field
  do các hệ thống tự điều chỉnh kích thước (auto-sizing) hoặc tự co giãn (auto-scaling) đặt.

* Thông tin về build, release hoặc image như timestamp, release ID, git branch,
  số PR, hash của image và địa chỉ registry.

* Con trỏ (pointer) tới các kho lưu trữ logging, monitoring, analytics hoặc audit.

* Thông tin về thư viện client hoặc công cụ, có thể dùng cho mục đích gỡ lỗi (debug):
  ví dụ tên, phiên bản và thông tin build.

* Thông tin nguồn gốc (provenance) từ người dùng hoặc từ công cụ/hệ thống, chẳng hạn URL
  của các object liên quan từ những thành phần khác trong hệ sinh thái.

* Metadata của công cụ rollout gọn nhẹ: ví dụ cấu hình hoặc các checkpoint.

* Số điện thoại hoặc số máy nhắn tin của những người chịu trách nhiệm, hoặc các mục
  danh bạ cho biết nơi có thể tìm thấy thông tin đó, chẳng hạn website của team.

* Các chỉ thị từ người dùng cuối tới các phần hiện thực (implementation) nhằm thay đổi
  hành vi hoặc kích hoạt những tính năng không chuẩn.

Thay vì dùng annotation, bạn có thể lưu loại thông tin này trong một
cơ sở dữ liệu hoặc danh bạ bên ngoài, nhưng như vậy sẽ khó hơn rất nhiều trong việc
xây dựng các thư viện client và công cụ dùng chung cho việc triển khai, quản lý,
nội quan (introspection) và những việc tương tự.

## Cú pháp và tập ký tự (Syntax and character set)

_Annotation_ là các cặp key/value. Key hợp lệ của annotation có hai phân đoạn: một prefix tùy chọn và tên (name), phân tách bởi dấu gạch chéo (`/`). Phân đoạn tên là bắt buộc và phải có tối đa 63 ký tự, bắt đầu và kết thúc bằng một ký tự chữ-số (`[a-z0-9A-Z]`), ở giữa có thể chứa dấu gạch ngang (`-`), gạch dưới (`_`), dấu chấm (`.`) và các ký tự chữ-số. Prefix là tùy chọn. Nếu được chỉ định, prefix phải là một DNS subdomain: một chuỗi các DNS label phân tách bởi dấu chấm (`.`), tổng cộng không dài quá 253 ký tự, theo sau là dấu gạch chéo (`/`).

Nếu prefix bị bỏ qua, key của annotation được xem là riêng tư đối với người dùng. Các thành phần hệ thống tự động (ví dụ `kube-scheduler`, `kube-controller-manager`, `kube-apiserver`, `kubectl`, hoặc các cơ chế tự động hóa bên thứ ba khác) khi thêm annotation vào object của người dùng cuối thì bắt buộc phải chỉ định prefix.

Các prefix `kubernetes.io/` và `k8s.io/` được dành riêng cho các thành phần lõi của Kubernetes.

Value hợp lệ của annotation không bị giới hạn về tập ký tự — khác với value của label, value của annotation có thể chứa bất kỳ chuỗi nào, bao gồm ký tự đặc biệt, khoảng trắng và dữ liệu có cấu trúc như JSON hoặc YAML.
Nếu bạn định lưu dữ liệu nhị phân (chẳng hạn [CBOR](https://cbor.io/)),
dự án Kubernetes khuyến nghị bạn mã hóa base64 dữ liệu đó.
Tuy nhiên, tổng kích thước của **tất cả** annotation trên một object (gồm cả key lẫn value) không được vượt quá 256 KiB.

Ví dụ, đây là manifest của một Pod có annotation `imageregistry: https://hub.docker.com/`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: annotations-demo
  annotations:
    imageregistry: "https://hub.docker.com/"
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

## Tiếp theo (What's next)

- Tìm hiểu thêm về [Label và Selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/).
- Xem [Các label, annotation và taint phổ biến (Well-known labels, Annotations and Taints)](https://kubernetes.io/docs/reference/labels-annotations-taints/).
