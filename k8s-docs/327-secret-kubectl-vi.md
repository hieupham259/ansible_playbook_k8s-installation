# Quản lý Secret bằng kubectl (Managing Secrets using kubectl)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/>
>
> Tạo các đối tượng Secret bằng công cụ dòng lệnh kubectl.

Trang này chỉ cho bạn cách tạo, sửa, quản lý và xóa các
Secret của Kubernetes bằng công cụ dòng lệnh `kubectl`.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Tạo một Secret (Create a Secret)

Một đối tượng `Secret` lưu trữ dữ liệu nhạy cảm, chẳng hạn thông tin xác thực (credentials)
mà các Pod dùng để truy cập các dịch vụ. Ví dụ, bạn có thể cần một Secret để lưu
tên người dùng và mật khẩu cần thiết để truy cập một cơ sở dữ liệu.

Bạn có thể tạo Secret bằng cách truyền dữ liệu thô (raw data) ngay trong lệnh, hoặc bằng cách
lưu thông tin xác thực vào các file rồi truyền các file đó trong lệnh. Các lệnh sau
tạo một Secret lưu tên người dùng `admin` và mật khẩu `S!B\*d$zDsb=`.

### Dùng dữ liệu thô (Use raw data)

Chạy lệnh sau:

```shell
kubectl create secret generic db-user-pass \
    --from-literal=username=admin \
    --from-literal=password='S!B\*d$zDsb='
```

Bạn phải dùng dấu nháy đơn `''` để escape các ký tự đặc biệt như `$`, `\`,
`*`, `=` và `!` trong chuỗi của bạn. Nếu không, shell của bạn sẽ diễn giải các
ký tự này.

> **Ghi chú:** Trường `stringData` của Secret không hoạt động tốt với server-side apply.

### Dùng file nguồn (Use source files)

1. Lưu thông tin xác thực vào các file:

   ```shell
   echo -n 'admin' > ./username.txt
   echo -n 'S!B\*d$zDsb=' > ./password.txt
   ```

   Cờ `-n` đảm bảo các file được tạo ra không có thêm một ký tự xuống dòng
   thừa ở cuối văn bản. Điều này quan trọng vì khi `kubectl`
   đọc một file và mã hóa (encode) nội dung thành chuỗi base64, ký tự xuống dòng
   thừa cũng sẽ bị mã hóa theo. Bạn không cần escape các ký tự đặc biệt
   trong các chuỗi mà bạn đưa vào file.

1. Truyền đường dẫn file trong lệnh `kubectl`:

   ```shell
   kubectl create secret generic db-user-pass \
       --from-file=./username.txt \
       --from-file=./password.txt
   ```

   Tên khóa (key) mặc định là tên file. Bạn có thể tùy chọn đặt tên khóa
   bằng `--from-file=[key=]source`. Ví dụ:

   ```shell
   kubectl create secret generic db-user-pass \
       --from-file=username=./username.txt \
       --from-file=password=./password.txt
   ```

Với cả hai cách, output tương tự như sau:

```
secret/db-user-pass created
```

### Xác minh Secret (Verify the Secret) {#verify-the-secret}

Kiểm tra rằng Secret đã được tạo:

```shell
kubectl get secrets
```

Output tương tự như sau:

```
NAME              TYPE       DATA      AGE
db-user-pass      Opaque     2         51s
```

Xem chi tiết của Secret:

```shell
kubectl describe secret db-user-pass
```

Output tương tự như sau:

```
Name:            db-user-pass
Namespace:       default
Labels:          <none>
Annotations:     <none>

Type:            Opaque

Data
====
password:    12 bytes
username:    5 bytes
```

Các lệnh `kubectl get` và `kubectl describe` mặc định tránh hiển thị nội dung
của một `Secret`. Điều này nhằm bảo vệ `Secret` khỏi bị lộ
một cách vô tình, hoặc khỏi bị lưu lại trong log của terminal.

### Giải mã Secret (Decode the Secret) {#decoding-secret}

1. Xem nội dung của Secret bạn đã tạo:

   ```shell
   kubectl get secret db-user-pass -o jsonpath='{.data}'
   ```

   Output tương tự như sau:

   ```json
   { "password": "UyFCXCpkJHpEc2I9", "username": "YWRtaW4=" }
   ```

1. Giải mã (decode) dữ liệu `password`:

   ```shell
   echo 'UyFCXCpkJHpEc2I9' | base64 --decode
   ```

   Output tương tự như sau:

   ```
   S!B\*d$zDsb=
   ```

   > **Thận trọng:** Đây là một ví dụ cho mục đích minh họa trong tài liệu. Trong thực tế,
   > cách này có thể khiến lệnh chứa dữ liệu đã mã hóa bị lưu lại trong
   > lịch sử shell của bạn. Bất kỳ ai có quyền truy cập máy tính của bạn đều có thể tìm thấy
   > lệnh đó và giải mã secret. Cách tốt hơn là kết hợp lệnh xem và
   > lệnh giải mã với nhau.

   ```shell
   kubectl get secret db-user-pass -o jsonpath='{.data.password}' | base64 --decode
   ```

## Sửa một Secret (Edit a Secret) {#edit-secret}

Bạn có thể sửa một đối tượng `Secret` hiện có, trừ khi nó là
[bất biến (immutable)](https://kubernetes.io/docs/concepts/configuration/secret/#secret-immutable). Để sửa một
Secret, hãy chạy lệnh sau:

```shell
kubectl edit secrets <secret-name>
```

Lệnh này mở trình soạn thảo mặc định của bạn và cho phép bạn cập nhật các giá trị
Secret đã mã hóa base64 trong trường `data`, như trong ví dụ sau:

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file, it will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  password: UyFCXCpkJHpEc2I9
  username: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: "2022-06-28T17:44:13Z"
  name: db-user-pass
  namespace: default
  resourceVersion: "12708504"
  uid: 91becd59-78fa-4c85-823f-6d44436242ac
type: Opaque
```

## Dọn dẹp (Clean up)

Để xóa một Secret, hãy chạy lệnh sau:

```shell
kubectl delete secret db-user-pass
```

## Tiếp theo (What's next)

- Đọc thêm về [khái niệm Secret](https://kubernetes.io/docs/concepts/configuration/secret/)
- Tìm hiểu cách [quản lý Secret bằng file cấu hình](326-secret-config-file-vi.md)
- Tìm hiểu cách [quản lý Secret bằng kustomize](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kustomize/)
