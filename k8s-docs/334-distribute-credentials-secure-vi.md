# Phân phối thông tin xác thực một cách an toàn bằng Secret (Distribute Credentials Securely Using Secrets)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/>
>
> Trang này hướng dẫn cách đưa (inject) dữ liệu nhạy cảm, chẳng hạn như mật khẩu và khóa mã hóa, vào Pod một cách an toàn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ [nhóm 3b — Cấu hình ứng dụng](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod),
**bài 12/12 — bài cuối của nhóm** · Kiểm chứng ở
[Lab 3b — Cấu hình ứng dụng](labs/LAB-3B-CAU-HINH-UNG-DUNG.md), `phần B8` (volume + `items` +
`defaultMode` ở `B8.1`, `secretKeyRef` và `envFrom secretRef` ở `B8.2`).

Bài dài nhưng **lặp lại cùng một Secret `test-secret` qua nhiều đường tiêu thụ khác nhau**. Đọc
theo trục đó thì gọn: tạo Secret → đưa vào Pod bằng volume → đưa vào Pod bằng biến môi trường →
một ví dụ prod/test gộp cả hai. Bài không dạy khái niệm Secret — phần đó nằm ở
[109](109-secret-vi.md) bạn đã đọc; ở đây chỉ là các cách **phân phối** cho Pod.

**Phải hiểu ở lần đọc này:**

- Hai đường tạo cùng một Secret: viết file cấu hình với `data` mà bạn **tự chuyển sang base64**
  trước, hoặc để `kubectl create secret generic --from-literal` làm hộ bước đó. `kubectl describe`
  chỉ hiện **số byte** của từng key chứ không hiện giá trị.
- Đường volume (mục *Tạo một Pod truy cập dữ liệu bí mật thông qua Volume*): **mỗi key trong `data`
  trở thành một tên file** trong thư mục `mountPath`, mount `readOnly: true`; chương trình của bạn
  phải đọc file ở thư mục đó chứ không đọc biến môi trường.
- `.spec.volumes[].secret.items` chiếu key vào đường dẫn bạn chọn, và đổi luật: **chỉ key được
  liệt kê mới được chiếu vào**, muốn đủ key thì phải liệt kê hết, và **key liệt kê mà không tồn tại
  trong Secret thì volume không được tạo**.
- `defaultMode` đặt bit quyền POSIX cho toàn bộ volume, mặc định là `0644`, và ghi đè được theo
  từng key; trong manifest JSON phải viết giá trị **thập phân** vì JSON không hiểu ký hiệu bát phân.
- Đường biến môi trường: `secretKeyRef` lấy **một key** và bạn **tự đặt tên biến** (`SECRET_USERNAME`),
  còn `envFrom.secretRef` nạp **cả Secret** và **key trở thành tên biến**. Giới hạn kèm theo: một
  container đã dùng Secret qua biến môi trường thì **không thấy** bản cập nhật của Secret cho tới
  khi nó khởi động lại.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Đoạn cuối mục *Ví dụ prod/test*: rút gọn Pod spec bằng `serviceAccount: prod-db-client` | ServiceAccount chưa học; ở đây chỉ cần thấy hai Pod khác nhau đúng một field | [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) |
| Ghi chú escape ký tự đặc biệt (`$`, `\`, `*`, `=`, `!`) khi truyền mật khẩu qua shell, và mẹo dùng `--from-file` để khỏi escape | là chi tiết thao tác của `kubectl create secret`, không phải cơ chế phân phối | bài [327](327-secret-kubectl-vi.md) cùng nhóm 3b, kiểm chứng ở `phần B6.1` của [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) |
| Câu "có các giải pháp bên thứ ba để kích hoạt việc khởi động lại khi secret thay đổi" | không thuộc Kubernetes core, cluster lab không cài thêm gì | ranh giới cập nhật giữa volume và biến môi trường đã học ở bài [109](109-secret-vi.md), kiểm chứng ở `phần B3.1` của [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) |
| Ba link `Secret` / `Volume` / `Pod` ở mục *Tài liệu tham khảo* | tra cứu API, không phải nội dung bài | bài [91](91-volumes-vi.md) ở [giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) cho phần Volume |

---

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

## Tạo một Pod truy cập dữ liệu bí mật thông qua Volume (Create a Pod that has access to the secret data through a Volume) {#create-a-pod-that-has-access-to-the-secret-data-through-a-volume}

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

### Chiếu các key của Secret vào những đường dẫn file cụ thể (Project Secret keys to specific file paths) {#project-secret-keys-to-specific-file-paths}

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

### Thiết lập quyền POSIX cho các key của Secret (Set POSIX permissions for Secret keys) {#set-posix-permissions-for-secret-keys}

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

## Định nghĩa biến môi trường của container bằng dữ liệu Secret (Define container environment variables using Secret data) {#define-container-environment-variables-using-secret-data}

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

## Cấu hình tất cả cặp key-value trong một Secret thành biến môi trường của container (Configure all key-value pairs in a Secret as container environment variables) {#configure-all-key-value-pairs-in-a-secret-as-container-environment-variables}

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

- Tìm hiểu thêm về [Secret](109-secret-vi.md).
- Tìm hiểu về [Volume](91-volumes-vi.md).

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Secret `mysecret` có hai key `username` và `password`. Bạn mount nó bằng volume và khai
   `items` với đúng một mục `key: username`. Container thấy những file nào dưới `/etc/foo`? Nếu bạn
   thêm vào `items` một key **không tồn tại** trong Secret thì chuyện gì xảy ra với Pod?
2. **Câu bẫy.** Pod của bạn nạp Secret bằng `envFrom.secretRef` và đang chạy. Bạn `kubectl apply`
   một bản Secret mới với mật khẩu khác. Tiến trình trong container đọc được mật khẩu mới chưa?
3. Trên cluster lab, bạn tạo `test-secret` từ `lab-k8s-master`, mount vào một Pod chạy trên
   `lab-k8s-worker1`, rồi `kubectl exec` vào đó và `cat /etc/secret-volume/password`. Bài nói kết
   quả in ra là gì? Điều đó cho biết vai trò thật của bước base64 ở đầu bài là gì?
4. Cùng một Secret `test-secret`, hai Pod tiêu thụ bằng hai cách: một Pod dùng `secretKeyRef`, một
   Pod dùng `envFrom.secretRef`. Trong mỗi Pod, **tên biến môi trường** do ai quyết định?
5. Bạn viết `defaultMode: 0400` cho một secret volume. Quyền mặc định khi **không** khai
   `defaultMode` là bao nhiêu, và vì sao bài cảnh báo riêng về `0400` khi manifest được viết bằng
   JSON?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Chỉ **một file**, tại `/etc/foo/my-group/my-username` — đúng `path` bạn chỉ định, **không** phải
   `/etc/foo/username`. Key `password` **không được chiếu vào**, vì khi đã liệt kê `items` thì chỉ
   các key trong danh sách được chiếu; muốn dùng hết thì phải liệt kê hết. Với key không tồn tại:
   **volume sẽ không được tạo** — bài nói thẳng "mọi key được liệt kê phải tồn tại trong Secret
   tương ứng. Nếu không, volume sẽ không được tạo".
2. **Chưa.** Bài nói rõ: nếu một container đã sử dụng Secret qua biến môi trường thì một lần cập
   nhật Secret **sẽ không được container đó nhìn thấy trừ khi nó được khởi động lại**. Trực giác
   "apply xong là Pod thấy ngay" đến từ đường volume, không phải đường biến môi trường — biến môi
   trường được nạp một lần lúc container khởi động. Muốn tiến trình thấy giá trị mới thì phải cho
   container khởi động lại; bài chỉ nhắc rằng có giải pháp bên thứ ba làm việc đó tự động.
3. In ra **đúng nguyên văn mật khẩu** `39528$vdg7Jb` — bài chép sẵn kết quả đó. Nghĩa là base64 chỉ
   là **dạng biểu diễn** để nhét dữ liệu vào field `data` của manifest, **không phải một lớp bảo
   vệ**: container nhận lại chuỗi gốc, và chính vì vậy bài còn cho phép bỏ hẳn bước này bằng
   `kubectl create secret generic --from-literal`. `kubectl describe` giấu giá trị và chỉ hiện số
   byte, nhưng đó là hành vi hiển thị của `kubectl`, không phải bảo vệ dữ liệu — đúng như bài
   [109](109-secret-vi.md) đã nói khi bạn đọc nó ở đầu nhóm 3b.
4. Với **`secretKeyRef`**: **bạn** quyết định — `name` của mục `env` là tên biến (`SECRET_USERNAME`),
   còn `key` chỉ trỏ tới mẩu dữ liệu trong Secret; hai tên đó độc lập nhau. Với
   **`envFrom.secretRef`**: **Secret** quyết định — bài nói "key trong Secret trở thành tên biến môi
   trường trong Pod", nên biến sẽ tên đúng là `username` và `password`.
5. Mặc định là **`0644`** khi bạn không chỉ định quyền nào. Cảnh báo về JSON là vì **đặc tả JSON
   không hỗ trợ ký hiệu bát phân**: viết `0400` trong JSON thì nó bị hiểu là số **thập phân** `400`,
   không phải quyền `r--------`. Trong JSON phải dùng giá trị thập phân tương ứng; viết YAML thì mới
   được viết `defaultMode` ở dạng bát phân.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là bài **cuối cùng của nhóm 3b** —
trả lời hết rồi thì mở [Lab 3b — Cấu hình ứng dụng](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) và làm trọn
lab trước khi sang [nhóm 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn).
