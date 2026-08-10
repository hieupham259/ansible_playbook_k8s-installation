# Phân phối thông tin xác thực một cách an toàn bằng Secret (Distribute Credentials Securely Using Secrets)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/>
>
> Trang này hướng dẫn cách đưa (inject) dữ liệu nhạy cảm, chẳng hạn như mật khẩu và khóa mã hóa, vào Pod một cách an toàn.

Trang này hướng dẫn cách đưa (inject) dữ liệu nhạy cảm, chẳng hạn như mật khẩu và
khóa mã hóa (encryption key), vào Pod một cách an toàn.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.6 hoặc mới hơn. Để kiểm tra phiên bản,
hãy chạy `kubectl version`.

### Chuyển dữ liệu bí mật của bạn sang dạng biểu diễn base-64 (Convert your secret data to a base-64 representation)

Giả sử bạn muốn có hai mẩu dữ liệu bí mật: một username `my-app` và một mật khẩu
`39528$vdg7Jb`. Trước tiên, hãy dùng một công cụ mã hóa base64 để chuyển username và mật khẩu của bạn sang dạng biểu diễn base64. Dưới đây là ví dụ dùng chương trình base64 phổ biến:

```shell
echo -n 'my-app' | base64
echo -n '39528$vdg7Jb' | base64
```

Kết quả cho thấy dạng biểu diễn base-64 của username là `bXktYXBw`,
và dạng biểu diễn base-64 của mật khẩu là `Mzk1MjgkdmRnN0pi`.

> **Thận trọng:**
> Hãy dùng một công cụ cục bộ được hệ điều hành của bạn tin cậy để giảm rủi ro bảo mật
> từ các công cụ bên ngoài.

## Tạo một Secret (Create a Secret)

Dưới đây là file cấu hình bạn có thể dùng để tạo một Secret chứa username và
mật khẩu của bạn:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
data:
  username: bXktYXBw
  password: Mzk1MjgkdmRnN0pi
```

1. Tạo Secret:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/inject/secret.yaml
   ```

1. Xem thông tin về Secret:

   ```shell
   kubectl get secret test-secret
   ```

   Kết quả:

   ```
   NAME          TYPE      DATA      AGE
   test-secret   Opaque    2         1m
   ```

1. Xem thông tin chi tiết hơn về Secret:

   ```shell
   kubectl describe secret test-secret
   ```

   Kết quả:

   ```
   Name:       test-secret
   Namespace:  default
   Labels:     <none>
   Annotations:    <none>

   Type:   Opaque

   Data
   ====
   password:   13 bytes
   username:   7 bytes
   ```

### Tạo Secret trực tiếp bằng kubectl (Create a Secret directly with kubectl)

Nếu bạn muốn bỏ qua bước mã hóa Base64, bạn có thể tạo cùng một Secret như trên
bằng lệnh `kubectl create secret`. Ví dụ:

```shell
kubectl create secret generic test-secret --from-literal='username=my-app' --from-literal='password=39528$vdg7Jb'
```

Cách này tiện hơn. Cách chi tiết được trình bày trước đó đi qua
từng bước một cách tường minh để minh họa điều gì đang diễn ra.

## Tạo một Pod truy cập dữ liệu bí mật thông qua Volume (Create a Pod that has access to the secret data through a Volume)

Dưới đây là file cấu hình bạn có thể dùng để tạo một Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
    - name: test-container
      image: nginx
      volumeMounts:
        # name phải khớp với tên volume bên dưới
        - name: secret-volume
          mountPath: /etc/secret-volume
          readOnly: true
  # Dữ liệu bí mật được expose cho các Container trong Pod thông qua một Volume.
  volumes:
    - name: secret-volume
      secret:
        secretName: test-secret
```

1. Tạo Pod:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/inject/secret-pod.yaml
   ```

1. Xác minh rằng Pod của bạn đang chạy:

   ```shell
   kubectl get pod secret-test-pod
   ```

   Kết quả:

   ```
   NAME              READY     STATUS    RESTARTS   AGE
   secret-test-pod   1/1       Running   0          42m
   ```

1. Mở một shell vào Container đang chạy trong Pod của bạn:

   ```shell
   kubectl exec -i -t secret-test-pod -- /bin/bash
   ```

1. Dữ liệu bí mật được expose cho Container thông qua một Volume được mount tại
   `/etc/secret-volume`.

   Trong shell của bạn, hãy liệt kê các file trong thư mục `/etc/secret-volume`:

   ```shell
   # Chạy lệnh này trong shell bên trong container
   ls /etc/secret-volume
   ```

   Kết quả cho thấy hai file, mỗi file ứng với một mẩu dữ liệu bí mật:

   ```
   password username
   ```

1. Trong shell của bạn, hiển thị nội dung của các file `username` và `password`:

   ```shell
   # Chạy lệnh này trong shell bên trong container
   echo "$( cat /etc/secret-volume/username )"
   echo "$( cat /etc/secret-volume/password )"
   ```

   Kết quả là username và mật khẩu của bạn:

   ```
   my-app
   39528$vdg7Jb
   ```

Hãy sửa image hoặc dòng lệnh của bạn sao cho chương trình tìm các file trong
thư mục `mountPath`. Mỗi key trong map `data` của Secret trở thành một tên file
trong thư mục này.

### Chiếu các key của Secret vào những đường dẫn file cụ thể (Project Secret keys to specific file paths)

Bạn cũng có thể kiểm soát các đường dẫn bên trong volume nơi các key của Secret được chiếu (project) vào. Hãy dùng
field `.spec.volumes[].secret.items` để thay đổi đường dẫn đích của từng key:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
```

Khi bạn triển khai Pod này, những điều sau xảy ra:

- Key `username` từ `mysecret` khả dụng cho container tại đường dẫn
  `/etc/foo/my-group/my-username` thay vì tại `/etc/foo/username`.
- Key `password` từ đối tượng Secret đó không được chiếu vào.

Nếu bạn liệt kê các key một cách tường minh bằng `.spec.volumes[].secret.items`, hãy lưu ý
những điều sau:

- Chỉ các key được chỉ định trong `items` mới được chiếu vào.
- Để sử dụng tất cả các key từ Secret, tất cả chúng phải được liệt kê trong
  field `items`.
- Mọi key được liệt kê phải tồn tại trong Secret tương ứng. Nếu không, volume
  sẽ không được tạo.

### Thiết lập quyền POSIX cho các key của Secret (Set POSIX permissions for Secret keys)

Bạn có thể thiết lập các bit quyền truy cập file POSIX cho từng key riêng lẻ của Secret.
Nếu bạn không chỉ định quyền nào, `0644` sẽ được dùng theo mặc định.
Bạn cũng có thể thiết lập một chế độ (mode) file POSIX mặc định cho toàn bộ Secret volume, và
có thể ghi đè theo từng key nếu cần.

Ví dụ, bạn có thể chỉ định một mode mặc định như sau:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      defaultMode: 0400
```

Secret được mount tại `/etc/foo`; tất cả các file được tạo bởi
secret volume mount đều có quyền `0400`.

> **Ghi chú:**
> Nếu bạn định nghĩa một Pod hoặc một Pod template bằng JSON, hãy lưu ý rằng đặc tả JSON
> không hỗ trợ ký hiệu bát phân (octal) cho các con số, vì JSON coi
> `0400` là giá trị _thập phân_ `400`. Trong JSON, hãy dùng giá trị thập phân cho
> `defaultMode`. Nếu bạn viết YAML, bạn có thể viết `defaultMode`
> ở dạng bát phân.

## Định nghĩa biến môi trường của container bằng dữ liệu Secret (Define container environment variables using Secret data)

Bạn có thể sử dụng dữ liệu trong Secret dưới dạng biến môi trường trong các
container của bạn.

Nếu một container đã sử dụng một Secret qua biến môi trường,
thì một lần cập nhật Secret sẽ không được container đó nhìn thấy trừ khi nó được
khởi động lại. Có các giải pháp bên thứ ba để kích hoạt việc khởi động lại khi
secret thay đổi.

### Định nghĩa một biến môi trường của container với dữ liệu từ một Secret duy nhất (Define a container environment variable with data from a single Secret)

- Định nghĩa một biến môi trường dưới dạng cặp key-value trong một Secret:

  ```shell
  kubectl create secret generic backend-user --from-literal=backend-username='backend-admin'
  ```

- Gán giá trị `backend-username` được định nghĩa trong Secret cho biến môi trường `SECRET_USERNAME` trong đặc tả Pod.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: env-single-secret
  spec:
    containers:
    - name: envars-test-container
      image: nginx
      env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: backend-user
            key: backend-username
  ```

- Tạo Pod:

  ```shell
  kubectl create -f https://k8s.io/examples/pods/inject/pod-single-secret-env-variable.yaml
  ```

- Trong shell của bạn, hiển thị nội dung của biến môi trường container `SECRET_USERNAME`.

  ```shell
  kubectl exec -i -t env-single-secret -- /bin/sh -c 'echo $SECRET_USERNAME'
  ```

  Kết quả tương tự như:

  ```
  backend-admin
  ```

### Định nghĩa các biến môi trường của container với dữ liệu từ nhiều Secret (Define container environment variables with data from multiple Secrets)

- Giống ví dụ trước, hãy tạo các Secret trước.

  ```shell
  kubectl create secret generic backend-user --from-literal=backend-username='backend-admin'
  kubectl create secret generic db-user --from-literal=db-username='db-admin'
  ```

- Định nghĩa các biến môi trường trong đặc tả Pod.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: envvars-multiple-secrets
  spec:
    containers:
    - name: envars-test-container
      image: nginx
      env:
      - name: BACKEND_USERNAME
        valueFrom:
          secretKeyRef:
            name: backend-user
            key: backend-username
      - name: DB_USERNAME
        valueFrom:
          secretKeyRef:
            name: db-user
            key: db-username
  ```

- Tạo Pod:

  ```shell
  kubectl create -f https://k8s.io/examples/pods/inject/pod-multiple-secret-env-variable.yaml
  ```

- Trong shell của bạn, hiển thị các biến môi trường của container.

  ```shell
  kubectl exec -i -t envvars-multiple-secrets -- /bin/sh -c 'env | grep _USERNAME'
  ```

  Kết quả tương tự như:

  ```
  DB_USERNAME=db-admin
  BACKEND_USERNAME=backend-admin
  ```

## Cấu hình tất cả cặp key-value trong một Secret thành biến môi trường của container (Configure all key-value pairs in a Secret as container environment variables)

> **Ghi chú:**
> Chức năng này khả dụng trong Kubernetes v1.6 trở lên.

- Tạo một Secret chứa nhiều cặp key-value:

  ```shell
  kubectl create secret generic test-secret --from-literal=username='my-app' --from-literal=password='39528$vdg7Jb'
  ```

- Dùng envFrom để định nghĩa tất cả dữ liệu của Secret thành các biến môi trường của container.
  Key trong Secret trở thành tên biến môi trường trong Pod.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: envfrom-secret
  spec:
    containers:
    - name: envars-test-container
      image: nginx
      envFrom:
      - secretRef:
          name: test-secret
  ```

- Tạo Pod:

  ```shell
  kubectl create -f https://k8s.io/examples/pods/inject/pod-secret-envFrom.yaml
  ```

- Trong shell của bạn, hiển thị các biến môi trường container `username` và `password`.

  ```shell
  kubectl exec -i -t envfrom-secret -- /bin/sh -c 'echo "username: $username\npassword: $password\n"'
  ```

  Kết quả tương tự như:

  ```
  username: my-app
  password: 39528$vdg7Jb
  ```

## Ví dụ: Cung cấp thông tin xác thực prod/test cho Pod bằng Secret (Example: Provide prod/test credentials to Pods using Secrets) {#provide-prod-test-creds}

Ví dụ này minh họa một Pod sử dụng secret chứa thông tin xác thực (credentials) của môi trường production và
một Pod khác sử dụng secret chứa thông tin xác thực của môi trường test.

1. Tạo một secret cho thông tin xác thực của môi trường prod:

   ```shell
   kubectl create secret generic prod-db-secret --from-literal=username=produser --from-literal=password=Y4nys7f11
   ```

   Kết quả tương tự như:

   ```
   secret "prod-db-secret" created
   ```

1. Tạo một secret cho thông tin xác thực của môi trường test.

   ```shell
   kubectl create secret generic test-db-secret --from-literal=username=testuser --from-literal=password=iluvtests
   ```

   Kết quả tương tự như:

   ```
   secret "test-db-secret" created
   ```

   > **Ghi chú:**
   > Các ký tự đặc biệt như `$`, `\`, `*`, `=` và `!` sẽ bị
   > [shell](https://en.wikipedia.org/wiki/Shell_(computing)) của bạn diễn giải và cần được escape.
   >
   > Trong hầu hết các shell, cách dễ nhất để escape mật khẩu là bao quanh nó bằng dấu nháy đơn (`'`).
   > Ví dụ, nếu mật khẩu thực của bạn là `S!B\*d$zDsb=`, bạn nên thực thi lệnh như sau:
   >
   > ```shell
   > kubectl create secret generic dev-db-secret --from-literal=username=devuser --from-literal=password='S!B\*d$zDsb='
   > ```
   >
   > Bạn không cần escape các ký tự đặc biệt trong mật khẩu lấy từ file (`--from-file`).

1. Tạo các manifest cho Pod:

   ```shell
   cat <<EOF > pod.yaml
   apiVersion: v1
   kind: List
   items:
   - kind: Pod
     apiVersion: v1
     metadata:
       name: prod-db-client-pod
       labels:
         name: prod-db-client
     spec:
       volumes:
       - name: secret-volume
         secret:
           secretName: prod-db-secret
       containers:
       - name: db-client-container
         image: myClientImage
         volumeMounts:
         - name: secret-volume
           readOnly: true
           mountPath: "/etc/secret-volume"
   - kind: Pod
     apiVersion: v1
     metadata:
       name: test-db-client-pod
       labels:
         name: test-db-client
     spec:
       volumes:
       - name: secret-volume
         secret:
           secretName: test-db-secret
       containers:
       - name: db-client-container
         image: myClientImage
         volumeMounts:
         - name: secret-volume
           readOnly: true
           mountPath: "/etc/secret-volume"
   EOF
   ```

   > **Ghi chú:**
   > Đặc tả của hai Pod chỉ khác nhau ở một field duy nhất; điều này giúp dễ dàng tạo ra
   > các Pod với những khả năng khác nhau từ một Pod template chung.

1. Áp dụng tất cả các đối tượng đó lên API server bằng cách chạy:

   ```shell
   kubectl create -f pod.yaml
   ```

Cả hai container sẽ có các file sau tồn tại trên hệ thống file của chúng, với các giá trị
tương ứng với môi trường của từng container:

```
/etc/secret-volume/username
/etc/secret-volume/password
```

Bạn có thể đơn giản hóa hơn nữa đặc tả Pod cơ sở bằng cách dùng hai service account:

1. `prod-user` với `prod-db-secret`
1. `test-user` với `test-db-secret`

Đặc tả Pod được rút gọn thành:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prod-db-client-pod
  labels:
    name: prod-db-client
spec:
  serviceAccount: prod-db-client
  containers:
  - name: db-client-container
    image: myClientImage
```

### Tài liệu tham khảo (References)

- [Secret](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#secret-v1-core)
- [Volume](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#volume-v1-core)
- [Pod](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#pod-v1-core)

## Tiếp theo (What's next)

- Tìm hiểu thêm về [Secret](https://kubernetes.io/docs/concepts/configuration/secret/).
- Tìm hiểu về [Volume](https://kubernetes.io/docs/concepts/storage/volumes/).
