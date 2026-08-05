# Tổ chức quyền truy cập cluster bằng file kubeconfig (Organizing Cluster Access Using kubeconfig Files)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/>

Hãy dùng các file kubeconfig để tổ chức thông tin về cluster, người dùng (user), namespace và
các cơ chế xác thực (authentication). Công cụ dòng lệnh `kubectl` dùng các file kubeconfig để
tìm thông tin cần thiết nhằm chọn một cluster và giao tiếp với API server
của cluster đó.

> **Ghi chú:**
> Một file được dùng để cấu hình quyền truy cập tới các cluster được gọi là
> *file kubeconfig*. Đây là cách gọi chung cho loại file cấu hình này.
> Nó không có nghĩa là tồn tại một file có tên `kubeconfig`.

> **Cảnh báo:**
> Chỉ sử dụng file kubeconfig từ những nguồn đáng tin cậy. Việc dùng một file kubeconfig được chế tác đặc biệt có thể dẫn đến thực thi mã độc hoặc làm lộ file.
> Nếu bắt buộc phải dùng một file kubeconfig không đáng tin cậy, hãy kiểm tra nó cẩn thận trước, giống như cách bạn kiểm tra một shell script.

Theo mặc định, `kubectl` tìm một file tên là `config` trong thư mục `$HOME/.kube`.
Bạn có thể chỉ định các file kubeconfig khác bằng cách đặt biến môi trường `KUBECONFIG`
hoặc bằng cách dùng flag
[`--kubeconfig`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl/).

Để xem hướng dẫn từng bước về việc tạo và chỉ định file kubeconfig, hãy xem
[Cấu hình quyền truy cập tới nhiều cluster](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters).

## Hỗ trợ nhiều cluster, người dùng và cơ chế xác thực (Supporting multiple clusters, users, and authentication mechanisms)

Giả sử bạn có nhiều cluster, và người dùng cùng các thành phần của bạn xác thực
theo nhiều cách khác nhau. Ví dụ:

- Một kubelet đang chạy có thể xác thực bằng certificate.
- Một người dùng có thể xác thực bằng token.
- Quản trị viên có thể có các bộ certificate mà họ cấp cho từng người dùng riêng lẻ.

Với các file kubeconfig, bạn có thể tổ chức các cluster, người dùng và namespace của mình.
Bạn cũng có thể định nghĩa các context để chuyển đổi nhanh chóng và dễ dàng giữa
các cluster và namespace.

## Context

Một phần tử *context* trong file kubeconfig được dùng để gom nhóm các tham số truy cập
dưới một cái tên thuận tiện. Mỗi context có ba tham số: cluster, namespace và user.
Theo mặc định, công cụ dòng lệnh `kubectl` dùng các tham số từ
*context hiện tại (current context)* để giao tiếp với cluster.

Để chọn context hiện tại:
```
kubectl config use-context
```

## Biến môi trường KUBECONFIG (The KUBECONFIG environment variable)

Biến môi trường `KUBECONFIG` chứa một danh sách các file kubeconfig.
Với Linux và Mac, danh sách được phân tách bằng dấu hai chấm. Với Windows, danh sách
được phân tách bằng dấu chấm phẩy. Biến môi trường `KUBECONFIG` không
bắt buộc. Nếu biến môi trường `KUBECONFIG` không tồn tại,
`kubectl` dùng file kubeconfig mặc định, `$HOME/.kube/config`.

Nếu biến môi trường `KUBECONFIG` tồn tại, `kubectl` dùng
một cấu hình hiệu lực (effective configuration) là kết quả của việc hợp nhất (merge) các file
được liệt kê trong biến môi trường `KUBECONFIG`.

## Hợp nhất các file kubeconfig (Merging kubeconfig files)

Để xem cấu hình của bạn, hãy nhập lệnh sau:

```shell
kubectl config view
```

Như đã mô tả ở trên, kết quả xuất ra có thể đến từ một file kubeconfig duy nhất,
hoặc có thể là kết quả của việc hợp nhất nhiều file kubeconfig.

Dưới đây là các quy tắc mà `kubectl` áp dụng khi hợp nhất các file kubeconfig:

1. Nếu flag `--kubeconfig` được đặt, chỉ dùng file được chỉ định. Không hợp nhất.
   Chỉ được phép dùng flag này một lần.

   Ngược lại, nếu biến môi trường `KUBECONFIG` được đặt, dùng nó như một
   danh sách các file cần được hợp nhất.
   Hợp nhất các file được liệt kê trong biến môi trường `KUBECONFIG`
   theo các quy tắc sau:

   * Bỏ qua các tên file rỗng.
   * Báo lỗi với các file có nội dung không thể giải tuần tự hóa (deserialize) được.
   * File đầu tiên đặt một giá trị hoặc một khóa map cụ thể sẽ thắng.
   * Không bao giờ thay đổi giá trị hoặc khóa map đó.
     Ví dụ: Giữ nguyên context của file đầu tiên đặt `current-context`.
     Ví dụ: Nếu hai file cùng khai báo một `red-user`, chỉ dùng các giá trị từ `red-user` của file đầu tiên.
     Kể cả khi file thứ hai có các mục không xung đột dưới `red-user`, vẫn loại bỏ chúng.

   Để xem ví dụ về cách đặt biến môi trường `KUBECONFIG`, hãy xem
   [Đặt biến môi trường KUBECONFIG](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#set-the-kubeconfig-environment-variable).

   Nếu không thuộc các trường hợp trên, dùng file kubeconfig mặc định, `$HOME/.kube/config`, và không hợp nhất gì cả.

1. Xác định context sẽ dùng dựa trên kết quả khớp đầu tiên trong chuỗi sau:

    1. Dùng flag dòng lệnh `--context` nếu nó tồn tại.
    1. Dùng `current-context` từ các file kubeconfig đã hợp nhất.

   Tại bước này, context rỗng vẫn được chấp nhận.

1. Xác định cluster và user. Tại thời điểm này, có thể có hoặc không có context.
   Xác định cluster và user dựa trên kết quả khớp đầu tiên trong chuỗi sau,
   chuỗi này được chạy hai lần: một lần cho user và một lần cho cluster:

   1. Dùng flag dòng lệnh nếu nó tồn tại: `--user` hoặc `--cluster`.
   1. Nếu context không rỗng, lấy user hoặc cluster từ context đó.

   Tại bước này, user và cluster có thể rỗng.

1. Xác định thông tin cluster thực sự sẽ dùng. Tại thời điểm này, có thể có hoặc
   không có thông tin cluster.
   Xây dựng từng phần của thông tin cluster dựa trên chuỗi sau; kết quả khớp đầu tiên sẽ thắng:

   1. Dùng các flag dòng lệnh nếu chúng tồn tại: `--server`, `--certificate-authority`, `--insecure-skip-tls-verify`.
   1. Nếu có bất kỳ thuộc tính thông tin cluster nào từ các file kubeconfig đã hợp nhất, dùng chúng.
   1. Nếu không có vị trí server nào, thất bại.

1. Xác định thông tin user thực sự sẽ dùng. Xây dựng thông tin user theo cùng các
   quy tắc như thông tin cluster, ngoại trừ việc chỉ cho phép một kỹ thuật
   xác thực cho mỗi user:

   1. Dùng các flag dòng lệnh nếu chúng tồn tại: `--client-certificate`, `--client-key`, `--username`, `--password`, `--token`.
   1. Dùng các trường `user` từ các file kubeconfig đã hợp nhất.
   1. Nếu có hai kỹ thuật xung đột nhau, thất bại.

1. Với bất kỳ thông tin nào vẫn còn thiếu, dùng các giá trị mặc định và có thể
   nhắc người dùng nhập thông tin xác thực.

## Tham chiếu file (File references)

Các tham chiếu file và đường dẫn trong một file kubeconfig là tương đối so với vị trí của file kubeconfig đó.
Các tham chiếu file trên dòng lệnh là tương đối so với thư mục làm việc hiện tại.
Trong `$HOME/.kube/config`, đường dẫn tương đối được lưu ở dạng tương đối, và đường dẫn tuyệt đối
được lưu ở dạng tuyệt đối.

## Proxy

Bạn có thể cấu hình `kubectl` dùng một proxy cho từng cluster bằng `proxy-url` trong file kubeconfig của bạn, như sau:

```yaml
apiVersion: v1
kind: Config

clusters:
- cluster:
    proxy-url: http://proxy.example.org:3128
    server: https://k8s.example.org/k8s/clusters/c-xxyyzz
  name: development

users:
- name: developer

contexts:
- context:
  name: development
```

## Tiếp theo (What's next)

* [Cấu hình quyền truy cập tới nhiều cluster](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
* [`kubectl config`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#config)
