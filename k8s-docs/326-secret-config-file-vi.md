# Quản lý Secret bằng file cấu hình (Managing Secrets using Configuration File)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-config-file/>
>
> Tạo các đối tượng Secret bằng file cấu hình tài nguyên (resource configuration file).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Tạo Secret (Create the Secret) {#create-the-config-file}

Bạn có thể định nghĩa đối tượng `Secret` trong một manifest trước, ở định dạng JSON hoặc YAML,
rồi sau đó tạo đối tượng đó. Tài nguyên
[Secret](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#secret-v1-core)
chứa hai map: `data` và `stringData`.
Trường `data` được dùng để lưu dữ liệu tùy ý, mã hóa (encode) bằng base64. Trường
`stringData` được cung cấp cho thuận tiện, và nó cho phép bạn cung cấp
cùng dữ liệu đó dưới dạng chuỗi chưa mã hóa.
Các khóa (key) của `data` và `stringData` phải chỉ gồm các ký tự chữ và số,
`-`, `_` hoặc `.`.

Ví dụ sau lưu hai chuỗi vào một Secret bằng trường `data`.

1. Chuyển các chuỗi sang base64:

   ```shell
   echo -n 'admin' | base64
   echo -n '1f2d1e2e67df' | base64
   ```

   > **Ghi chú:** Các giá trị JSON và YAML đã tuần tự hóa (serialized) của dữ liệu Secret được
   > mã hóa dưới dạng chuỗi base64. Ký tự xuống dòng không hợp lệ bên trong các chuỗi này và
   > phải được loại bỏ. Khi dùng tiện ích `base64` trên Darwin/macOS, người dùng nên tránh dùng
   > tùy chọn `-b` để ngắt các dòng dài. Ngược lại, người dùng Linux *nên* thêm tùy chọn `-w 0`
   > vào các lệnh `base64`, hoặc dùng pipeline `base64 | tr -d '\n'` nếu tùy chọn `-w` không có
   > sẵn.

   Output tương tự như sau:

   ```
   YWRtaW4=
   MWYyZDFlMmU2N2Rm
   ```

1. Tạo manifest:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: mysecret
   type: Opaque
   data:
     username: YWRtaW4=
     password: MWYyZDFlMmU2N2Rm
   ```

   Lưu ý rằng tên của một đối tượng Secret phải là một
   [tên miền con DNS hợp lệ (DNS subdomain name)](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-subdomain-names).

1. Tạo Secret bằng [`kubectl apply`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply):

   ```shell
   kubectl apply -f ./secret.yaml
   ```

   Output tương tự như sau:

   ```
   secret/mysecret created
   ```

Để xác minh rằng Secret đã được tạo và để giải mã (decode) dữ liệu của Secret, hãy tham khảo
[Quản lý Secret bằng kubectl](327-secret-kubectl-vi.md#verify-the-secret).

### Chỉ định dữ liệu chưa mã hóa khi tạo Secret (Specify unencoded data when creating a Secret)

Trong một số tình huống nhất định, bạn có thể muốn dùng trường `stringData` thay thế. Trường
này cho phép bạn đưa một chuỗi chưa mã hóa base64 trực tiếp vào Secret,
và chuỗi đó sẽ được mã hóa giúp bạn khi Secret được tạo hoặc cập nhật.

Một ví dụ thực tế cho việc này là khi bạn triển khai một ứng dụng
dùng Secret để lưu một file cấu hình, và bạn muốn điền một số
phần của file cấu hình đó trong quá trình triển khai của mình.

Ví dụ, nếu ứng dụng của bạn dùng file cấu hình sau:

```yaml
apiUrl: "https://my.api.com/api/v1"
username: "<user>"
password: "<password>"
```

Bạn có thể lưu nó vào một Secret bằng định nghĩa sau:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
stringData:
  config.yaml: |
    apiUrl: "https://my.api.com/api/v1"
    username: <user>
    password: <password>
```

> **Ghi chú:** Trường `stringData` của Secret không hoạt động tốt với server-side apply.

Khi bạn truy xuất dữ liệu của Secret, lệnh trả về các giá trị đã mã hóa,
chứ không phải các giá trị thuần văn bản (plaintext) mà bạn đã cung cấp trong `stringData`.

Ví dụ, nếu bạn chạy lệnh sau:

```shell
kubectl get secret mysecret -o yaml
```

Output tương tự như sau:

```yaml
apiVersion: v1
data:
  config.yaml: YXBpVXJsOiAiaHR0cHM6Ly9teS5hcGkuY29tL2FwaS92MSIKdXNlcm5hbWU6IHt7dXNlcm5hbWV9fQpwYXNzd29yZDoge3twYXNzd29yZH19
kind: Secret
metadata:
  creationTimestamp: 2018-11-15T20:40:59Z
  name: mysecret
  namespace: default
  resourceVersion: "7225"
  uid: c280ad2e-e916-11e8-98f2-025000000001
type: Opaque
```

### Chỉ định cả `data` lẫn `stringData` (Specify both `data` and `stringData`)

Nếu bạn chỉ định một trường trong cả `data` lẫn `stringData`, giá trị từ `stringData` sẽ được dùng.

Ví dụ, nếu bạn định nghĩa Secret sau:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
stringData:
  username: administrator
```

> **Ghi chú:** Trường `stringData` của Secret không hoạt động tốt với server-side apply.

Đối tượng `Secret` được tạo ra như sau:

```yaml
apiVersion: v1
data:
  username: YWRtaW5pc3RyYXRvcg==
kind: Secret
metadata:
  creationTimestamp: 2018-11-15T20:46:46Z
  name: mysecret
  namespace: default
  resourceVersion: "7579"
  uid: 91460ecb-e917-11e8-98f2-025000000001
type: Opaque
```

`YWRtaW5pc3RyYXRvcg==` giải mã ra thành `administrator`.

## Sửa một Secret (Edit a Secret) {#edit-secret}

Để sửa dữ liệu trong Secret mà bạn đã tạo bằng manifest, hãy chỉnh sửa trường `data`
hoặc `stringData` trong manifest của bạn và apply file đó vào
cluster. Bạn có thể sửa một đối tượng `Secret` hiện có, trừ khi nó là
[bất biến (immutable)](https://kubernetes.io/docs/concepts/configuration/secret/#secret-immutable).

Ví dụ, nếu bạn muốn đổi mật khẩu ở ví dụ trước thành
`birdsarentreal`, hãy làm như sau:

1. Mã hóa chuỗi mật khẩu mới:

   ```shell
   echo -n 'birdsarentreal' | base64
   ```

   Output tương tự như sau:

   ```
   YmlyZHNhcmVudHJlYWw=
   ```

1. Cập nhật trường `data` với chuỗi mật khẩu mới của bạn:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: mysecret
   type: Opaque
   data:
     username: YWRtaW4=
     password: YmlyZHNhcmVudHJlYWw=
   ```

1. Apply manifest vào cluster của bạn:

   ```shell
   kubectl apply -f ./secret.yaml
   ```

   Output tương tự như sau:

   ```
   secret/mysecret configured
   ```

Kubernetes cập nhật đối tượng `Secret` hiện có. Cụ thể, công cụ `kubectl`
nhận thấy đã có một đối tượng `Secret` hiện hữu với cùng tên. `kubectl`
lấy về đối tượng hiện có, lên kế hoạch các thay đổi cho nó, và gửi đối tượng
`Secret` đã thay đổi lên control plane của cluster của bạn.

Nếu bạn chỉ định `kubectl apply --server-side` thay thế, `kubectl` sẽ dùng
[Server Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/).

## Dọn dẹp (Clean up)

Để xóa Secret bạn đã tạo:

```shell
kubectl delete secret mysecret
```

## Tiếp theo (What's next)

- Đọc thêm về [khái niệm Secret](https://kubernetes.io/docs/concepts/configuration/secret/)
- Tìm hiểu cách [quản lý Secret bằng kubectl](327-secret-kubectl-vi.md)
- Tìm hiểu cách [quản lý Secret bằng kustomize](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kustomize/)
