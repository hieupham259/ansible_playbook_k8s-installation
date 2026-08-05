# Tên và ID của đối tượng (Object Names and IDs)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/names/>
>
> Mỗi đối tượng trong cluster của bạn có một Tên (Name) duy nhất cho loại tài nguyên đó.
> Mỗi đối tượng Kubernetes cũng có một UID duy nhất trên toàn bộ cluster của bạn.

Mỗi đối tượng trong cluster của bạn có một [_Tên (Name)_](#names) duy nhất cho loại tài nguyên (resource) đó.
Mỗi đối tượng Kubernetes cũng có một [_UID_](#uids) duy nhất trên toàn bộ cluster của bạn.

Ví dụ, bạn chỉ có thể có một Pod tên `myapp-1234` trong cùng một [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/), nhưng bạn có thể có một Pod và một Deployment cùng mang tên `myapp-1234`.

Với các thuộc tính do người dùng cung cấp mà không cần duy nhất, Kubernetes cung cấp [label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) và [annotation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/).

## Tên (Names) {#names}

Một chuỗi do client cung cấp dùng để tham chiếu tới một đối tượng trong URL tài nguyên (resource URL), chẳng hạn `/api/v1/pods/some-name`.

Tại một thời điểm, chỉ một đối tượng của một loại (kind) nhất định được mang một tên nhất định. Tuy nhiên, nếu bạn xóa đối tượng đó, bạn có thể tạo một đối tượng mới trùng tên.

Tên phải là duy nhất trên tất cả các [phiên bản API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-groups-and-versioning) của cùng một tài nguyên.

Kubernetes định danh duy nhất các đối tượng bằng tổ hợp của bốn thuộc tính:
* **Nhóm API (API group)** (ví dụ: `apps`)
* **Loại tài nguyên (Resource type)** (ví dụ: `deployments`)
* **Namespace** (đối với các tài nguyên thuộc namespace)
* **Tên (Name)**

Mặc dù bạn có thể truy cập một tài nguyên qua các phiên bản API khác nhau (chẳng hạn `v1` hoặc `v1beta1`), phiên bản chỉ đơn giản là một cách biểu diễn khác của cùng một đối tượng bên dưới. Vì phiên bản không phải là một phần của định danh duy nhất, bạn không thể tạo hai đối tượng có cùng tên và cùng loại tài nguyên trong cùng một namespace bằng cách dùng các phiên bản API khác nhau.

> **Ghi chú:** Trong những trường hợp đối tượng đại diện cho một thực thể vật lý, như một Node đại diện cho một máy chủ (host) vật lý, khi máy chủ được tạo lại với cùng tên mà Node không bị xóa đi và tạo lại, Kubernetes sẽ coi máy chủ mới là máy chủ cũ, điều này có thể dẫn đến những điểm không nhất quán.

Server có thể sinh tên khi `generateName` được cung cấp thay cho `name` trong yêu cầu tạo tài nguyên.
Khi dùng `generateName`, giá trị được cung cấp sẽ được dùng làm tiền tố (prefix) của tên, và server sẽ nối thêm
một hậu tố (suffix) được sinh ra vào đó. Dù tên được sinh tự động, nó vẫn có thể xung đột với các tên đã tồn tại,
dẫn đến phản hồi HTTP 409. Điều này trở nên ít có khả năng xảy ra hơn nhiều trong Kubernetes v1.31 trở đi,
vì server sẽ thực hiện tối đa 8 lần thử sinh một tên duy nhất trước khi trả về phản hồi HTTP 409.

Dưới đây là bốn loại ràng buộc tên (name constraint) thường dùng cho các tài nguyên.

### Tên DNS Subdomain (DNS Subdomain Names)

Hầu hết các loại tài nguyên yêu cầu tên có thể dùng làm tên DNS subdomain
như định nghĩa trong [RFC 1123](https://tools.ietf.org/html/rfc1123).
Điều này nghĩa là tên phải:

- chứa không quá 253 ký tự
- chỉ chứa các ký tự chữ và số viết thường (lowercase alphanumeric), '-' hoặc '.'
- bắt đầu bằng một ký tự chữ hoặc số
- kết thúc bằng một ký tự chữ hoặc số

### Tên label theo RFC 1123 (RFC 1123 Label Names) {#dns-label-names}

Một số loại tài nguyên yêu cầu tên của chúng tuân theo chuẩn DNS label
như định nghĩa trong [RFC 1123](https://tools.ietf.org/html/rfc1123).
Điều này nghĩa là tên phải:

- chứa tối đa 63 ký tự
- chỉ chứa các ký tự chữ và số viết thường hoặc '-'
- bắt đầu bằng một ký tự chữ cái (alphabetic)
- kết thúc bằng một ký tự chữ hoặc số

> **Ghi chú:** Khi feature gate `RelaxedServiceNameValidation` được bật,
> tên các đối tượng Service được phép bắt đầu bằng chữ số.

### Tên label theo RFC 1035 (RFC 1035 Label Names)

Một số loại tài nguyên yêu cầu tên của chúng tuân theo chuẩn DNS label
như định nghĩa trong [RFC 1035](https://tools.ietf.org/html/rfc1035).
Điều này nghĩa là tên phải:

- chứa tối đa 63 ký tự
- chỉ chứa các ký tự chữ và số viết thường hoặc '-'
- bắt đầu bằng một ký tự chữ cái (alphabetic)
- kết thúc bằng một ký tự chữ hoặc số

> **Ghi chú:** Mặc dù về mặt kỹ thuật RFC 1123 cho phép label bắt đầu bằng chữ số, hiện thực (implementation)
> hiện tại của Kubernetes yêu cầu cả label theo RFC 1035 lẫn RFC 1123 đều phải bắt đầu
> bằng một ký tự chữ cái. Ngoại lệ là khi feature gate `RelaxedServiceNameValidation`
> được bật cho các đối tượng Service, khi đó tên Service được phép bắt đầu bằng chữ số.

### Tên dùng làm phân đoạn đường dẫn (Path Segment Names)

Một số loại tài nguyên yêu cầu tên của chúng có thể được mã hóa an toàn thành một
phân đoạn đường dẫn (path segment). Nói cách khác, tên không được là "." hay ".." và tên
không được chứa "/" hay "%".

Dưới đây là một manifest ví dụ cho một Pod tên `nginx-demo`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

> **Ghi chú:** Một số loại tài nguyên có thêm các ràng buộc bổ sung đối với tên của chúng.

## Các UID (UIDs) {#uids}

Một chuỗi do hệ thống Kubernetes sinh ra để định danh duy nhất các đối tượng.

Mỗi đối tượng được tạo ra trong suốt vòng đời của một cluster Kubernetes đều có một UID riêng biệt. UID nhằm mục đích phân biệt giữa các lần xuất hiện trong lịch sử của những thực thể tương tự nhau.

UID của Kubernetes là các định danh duy nhất toàn cầu (universally unique identifiers, còn được gọi là UUID).
UUID được chuẩn hóa theo ISO/IEC 9834-8 và ITU-T X.667.

## Tiếp theo (What's next)

* Đọc về [label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) và [annotation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) trong Kubernetes.
* Xem tài liệu thiết kế [Identifiers and Names in Kubernetes](https://git.k8s.io/design-proposals-archive/architecture/identifiers.md).
