# Các label được khuyến nghị (Recommended Labels)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/>

Bạn có thể trực quan hóa và quản lý các object Kubernetes bằng nhiều công cụ khác ngoài kubectl và
dashboard. Một tập label chung cho phép các công cụ hoạt động tương thích với nhau (interoperably), mô tả
các object theo một cách chung mà mọi công cụ đều có thể hiểu.

Bên cạnh việc hỗ trợ các công cụ, những label được khuyến nghị còn mô tả các ứng dụng
theo cách có thể truy vấn được.

Metadata được tổ chức xoay quanh khái niệm _ứng dụng (application)_. Kubernetes không phải
là một nền tảng dạng dịch vụ (platform as a service — PaaS) và không có hay cưỡng chế một khái niệm ứng dụng chính thức nào.
Thay vào đó, ứng dụng là khái niệm không chính thức và được mô tả bằng metadata. Định nghĩa về
những gì một ứng dụng bao gồm là khá lỏng lẻo.

> **Ghi chú:** Đây là những label được khuyến nghị. Chúng giúp việc quản lý ứng dụng dễ dàng hơn
> nhưng không bắt buộc đối với bất kỳ công cụ cốt lõi nào.

Các label và annotation dùng chung có chung một tiền tố (prefix): `app.kubernetes.io`. Label
không có tiền tố là label riêng của người dùng. Tiền tố dùng chung bảo đảm rằng các label dùng chung
không xung đột với label tùy chỉnh của người dùng.

## Label (Labels)

Để tận dụng tối đa các label này, chúng nên được áp dụng
trên mọi resource object.

| Key                                 | Mô tả           | Ví dụ  | Kiểu |
| ----------------------------------- | --------------------- | -------- | ---- |
| `app.kubernetes.io/name`            | Tên của ứng dụng | `mysql` | string |
| `app.kubernetes.io/instance`        | Một tên duy nhất định danh instance của ứng dụng | `mysql-abcxyz` | string |
| `app.kubernetes.io/version`         | Phiên bản hiện tại của ứng dụng (ví dụ: [SemVer 1.0](https://semver.org/spec/v1.0.0.html), hash của revision, v.v.) | `5.7.21` | string |
| `app.kubernetes.io/component`       | Thành phần trong kiến trúc | `database` | string |
| `app.kubernetes.io/part-of`         | Tên của ứng dụng cấp cao hơn mà ứng dụng này là một phần trong đó | `wordpress` | string |
| `app.kubernetes.io/managed-by`      | Công cụ đang được dùng để quản lý việc vận hành ứng dụng | `Helm` | string |

Để minh họa các label này trong thực tế, hãy xét object StatefulSet sau:

```yaml
# Đây là một đoạn trích
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxyz
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/managed-by: Helm
```

## Ứng dụng và các instance của ứng dụng (Applications And Instances Of Applications)

Một ứng dụng có thể được cài đặt một hoặc nhiều lần vào một cluster Kubernetes và,
trong một số trường hợp, vào cùng một namespace. Ví dụ, WordPress có thể được cài đặt
nhiều lần, trong đó các website khác nhau là những bản cài đặt WordPress khác nhau.

Tên của ứng dụng và tên instance được ghi nhận riêng biệt. Ví
dụ, WordPress có `app.kubernetes.io/name` là `wordpress` trong khi nó có
tên instance, thể hiện qua `app.kubernetes.io/instance` với giá trị là
`wordpress-abcxyz`. Điều này cho phép ứng dụng và instance của ứng dụng
đều có thể được nhận diện. Mỗi instance của một ứng dụng phải có tên duy nhất.

## Ví dụ (Examples)

Để minh họa các cách dùng khác nhau của những label này, các ví dụ sau có độ phức tạp khác nhau.

### Một Service stateless đơn giản (A Simple Stateless Service)

Hãy xét trường hợp một service stateless đơn giản được triển khai bằng các object `Deployment` và `Service`. Hai đoạn trích sau thể hiện cách các label có thể được dùng ở dạng đơn giản nhất.

`Deployment` được dùng để giám sát các pod chạy chính ứng dụng đó.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: myservice
    app.kubernetes.io/instance: myservice-abcxyz
...
```

`Service` được dùng để expose ứng dụng.
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: myservice
    app.kubernetes.io/instance: myservice-abcxyz
...
```

### Ứng dụng web với cơ sở dữ liệu (Web Application With A Database)

Hãy xét một ứng dụng phức tạp hơn một chút: một ứng dụng web (WordPress)
dùng một cơ sở dữ liệu (MySQL), được cài đặt bằng Helm. Các đoạn trích sau minh họa
phần đầu của những object được dùng để triển khai ứng dụng này.

Phần đầu của `Deployment` sau được dùng cho WordPress:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxyz
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress
...
```

`Service` được dùng để expose WordPress:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxyz
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress
...
```

MySQL được expose dưới dạng một `StatefulSet` với metadata cho cả chính nó lẫn ứng dụng lớn hơn mà nó thuộc về:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxyz
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
...
```

`Service` được dùng để expose MySQL như một phần của WordPress:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxyz
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
...
```

Với `StatefulSet` và `Service` của MySQL, bạn sẽ thấy thông tin về cả MySQL lẫn WordPress — ứng dụng bao quát hơn — đều được bao gồm.
