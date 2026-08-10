# Thực thi Pod Security Standards bằng nhãn Namespace (Enforce Pod Security Standards with Namespace Labels)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/>

Các namespace có thể được gán nhãn (label) để thực thi các
[Pod Security Standard](./115-pod-security-standards-vi.md). Ba chính sách
[privileged](./115-pod-security-standards-vi.md#privileged),
[baseline](./115-pod-security-standards-vi.md#baseline)
và [restricted](./115-pod-security-standards-vi.md#restricted) bao phủ rộng khắp phổ bảo mật
và được hiện thực bởi admission controller [Pod Security](./116-pod-security-admission-vi.md).

## Trước khi bạn bắt đầu (Before you begin)

Pod Security Admission đã khả dụng theo mặc định trong Kubernetes v1.23, ở trạng thái beta.
Từ phiên bản 1.25 trở đi, Pod Security Admission đạt mức phổ biến rộng rãi (generally
available).

Kubernetes server của bạn phải ở phiên bản v1.36 hoặc mới hơn. Để kiểm tra phiên bản, hãy
nhập `kubectl version`.

## Yêu cầu chuẩn Pod Security Standard `baseline` bằng nhãn namespace (Requiring the `baseline` Pod Security Standard with namespace labels)

Manifest này định nghĩa một Namespace `my-baseline-namespace` mà:

- _Chặn_ mọi pod không thỏa mãn các yêu cầu của chính sách `baseline`.
- Sinh ra một cảnh báo hiển thị cho người dùng và thêm một audit annotation vào bất kỳ pod
  nào được tạo mà không đáp ứng các yêu cầu của chính sách `restricted`.
- Ghim (pin) phiên bản của các chính sách `baseline` và `restricted` vào v1.36.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-baseline-namespace
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.36

    # Chúng ta đặt các nhãn này theo mức `enforce` mà mình _mong muốn_.
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.36
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.36
```

## Thêm nhãn vào các namespace hiện có bằng `kubectl label` (Add labels to existing namespaces with `kubectl label`)

> **Ghi chú:** Khi một nhãn chính sách `enforce` (hoặc nhãn phiên bản) được thêm vào hoặc
> thay đổi, admission plugin sẽ kiểm tra từng pod trong namespace đó theo chính sách mới.
> Các vi phạm được trả về cho người dùng dưới dạng cảnh báo.

Sẽ rất hữu ích khi áp dụng flag `--dry-run` lúc mới bắt đầu đánh giá các thay đổi về profile
bảo mật cho các namespace. Các kiểm tra của Pod Security Standard vẫn được chạy ở chế độ
_dry run_, cho bạn thông tin về việc chính sách mới sẽ đối xử với các pod hiện có như thế
nào, mà không thực sự cập nhật chính sách.

```shell
kubectl label --dry-run=server --overwrite ns --all \
    pod-security.kubernetes.io/enforce=baseline
```

### Áp dụng cho tất cả các namespace (Applying to all namespaces)

Nếu bạn mới bắt đầu với các Pod Security Standard, một bước đầu tiên phù hợp là cấu hình tất
cả các namespace với audit annotation cho một mức nghiêm ngặt hơn, chẳng hạn `baseline`:

```shell
kubectl label --overwrite ns --all \
  pod-security.kubernetes.io/audit=baseline \
  pod-security.kubernetes.io/warn=baseline
```

Lưu ý rằng thao tác này không đặt mức enforce, nhờ đó có thể phân biệt được các namespace
chưa được đánh giá một cách tường minh. Bạn có thể liệt kê các namespace chưa được đặt mức
enforce tường minh bằng lệnh sau:

```shell
kubectl get namespaces --selector='!pod-security.kubernetes.io/enforce'
```

### Áp dụng cho một namespace duy nhất (Applying to a single namespace)

Bạn cũng có thể cập nhật một namespace cụ thể. Lệnh này thêm chính sách `enforce=restricted`
vào `my-existing-namespace`, ghim phiên bản của chính sách restricted vào v1.36.

```shell
kubectl label --overwrite ns my-existing-namespace \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.36
```
