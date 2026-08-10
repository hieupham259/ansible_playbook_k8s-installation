# Định nghĩa giá trị biến môi trường bằng một Init Container (Define Environment Variable Values Using An Init Container)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-via-file/

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [beta]` (được bật mặc định: true)

Trang này chỉ cách cấu hình biến môi trường (environment variable) cho các container trong
một Pod thông qua file.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn v1.34. Để kiểm tra phiên bản, nhập
`kubectl version`.

## Cách thiết kế này hoạt động (How the design works)

Trong bài thực hành này, bạn sẽ tạo một Pod lấy biến môi trường từ các file, rồi đưa các giá
trị này vào container đang chạy.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envfile-test-pod
spec:
  initContainers:
    - name: setup-envfile
      image:  nginx
      command: ['sh', '-c', "echo \"DB_ADDRESS=\'address\'\nREST_ENDPOINT=\'endpoint\'\" > /data/config.env"]
      volumeMounts:
        - name: config
          mountPath: /data
  containers:
    - name: use-envfile
      image: nginx
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: DB_ADDRESS
          valueFrom:
            fileKeyRef:
              path: config.env
              volumeName: config
              key: DB_ADDRESS
              optional: false
  restartPolicy: Never
  volumes:
    - name: config
      emptyDir: {}
```

Trong manifest này, bạn có thể thấy `initContainer` mount một volume `emptyDir` và ghi các
biến môi trường vào một file bên trong volume đó, còn các container thường thì tham chiếu tới
cả file lẫn key của biến môi trường thông qua field `fileKeyRef` mà không cần mount volume.
Khi field `optional` được đặt là false, `key` được chỉ định trong `fileKeyRef` bắt buộc phải
tồn tại trong file biến môi trường.

Volume chỉ được mount vào container ghi file (`initContainer`), còn container tiêu thụ
(consumer) — container sử dụng biến môi trường — sẽ không được mount volume này.

Định dạng file env tuân theo [chuẩn file env của Kubernetes](#env-file-syntax).

Trong quá trình khởi tạo container, kubelet lấy các biến môi trường từ các file được chỉ định
trong volume `emptyDir` và cung cấp chúng cho container.

> **Ghi chú:** Tất cả các loại container (initContainers, container thường, sidecar container
> và ephemeral container) đều hỗ trợ nạp biến môi trường từ file.
>
> Mặc dù các biến môi trường này có thể chứa thông tin nhạy cảm, volume `emptyDir` không cung
> cấp các cơ chế bảo vệ như đối tượng Secret chuyên dụng. Do đó, việc cung cấp các biến môi
> trường bí mật cho container thông qua tính năng này không được coi là một thực hành bảo mật
> tốt (security best practice).

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/inject/envars-file-container.yaml
```

Kiểm tra rằng container trong Pod đang chạy:

```shell
# Nếu Pod mới chưa healthy, hãy chạy lại lệnh này vài lần.
kubectl get pods
```

Kiểm tra log của container để xem các biến môi trường:

```shell
kubectl logs envfile-test-pod -c use-envfile | grep DB_ADDRESS
```

Kết quả hiển thị giá trị của các biến môi trường đã chọn:

```
DB_ADDRESS=address
```

## Cú pháp file env (Env file syntax) {#env-file-syntax}

Định dạng file env mà Kubernetes sử dụng là một tập con được định nghĩa chặt chẽ của ngữ nghĩa
biến môi trường trong bash tuân thủ POSIX. Bất kỳ file env nào được Kubernetes hỗ trợ đều sẽ
tạo ra các biến môi trường giống hệt như khi được diễn dịch bởi bash tuân thủ POSIX. Tuy
nhiên, bash tuân thủ POSIX hỗ trợ thêm một số định dạng mà Kubernetes không chấp nhận.

Ví dụ:

```
MY_VAR='my-literal-value'
```

### Quy tắc (Rules)

* Khai báo biến: dùng dạng `VAR='value'`. Khoảng trắng bao quanh dấu `=` bị bỏ qua; khoảng
  trắng ở đầu dòng bị bỏ qua; dòng trống bị bỏ qua.
* Giá trị trong dấu nháy: giá trị phải được bao trong dấu nháy đơn (`'`).
  * Nội dung bên trong dấu nháy đơn được giữ nguyên đúng như viết (literal). Không có xử lý
    chuỗi thoát (escape sequence), gộp khoảng trắng hay diễn dịch ký tự nào được áp dụng.
  * Ký tự xuống dòng bên trong dấu nháy đơn được giữ nguyên (hỗ trợ giá trị nhiều dòng).
* Chú thích (comment): các dòng bắt đầu bằng `#` được coi là chú thích và bị bỏ qua. Ký tự `#`
  nằm bên trong giá trị được bao bằng nháy đơn thì không phải là chú thích.

Ví dụ:

```
# chú thích
DB_ADDRESS='address'

MULTI='line1
line2'
```

### Các dạng không được hỗ trợ (Unsupported forms)

* Giá trị không có dấu nháy bị **cấm**:
  * `VAR=value` — không được hỗ trợ.
* Giá trị trong dấu nháy kép bị **cấm**:
  * `VAR="value"` — không được hỗ trợ.
* Nhiều chuỗi trong dấu nháy đặt liền kề nhau **không** được hỗ trợ:
  * `VAR='val1''val2'` — không được hỗ trợ.
* Mọi hình thức nội suy (interpolation), khai triển (expansion) hay nối chuỗi (concatenation)
  đều **không** được hỗ trợ:
  * `VAR='a'$OTHER` hoặc `VAR=${OTHER}` — không được hỗ trợ.

Yêu cầu nghiêm ngặt về dấu nháy đơn bảo đảm rằng giá trị được kubelet đọc nguyên văn khi nạp
biến môi trường từ file.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [biến môi trường](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/).
* Đọc [Định nghĩa biến môi trường cho một Container](331-define-environment-variable-vi.md)
* Đọc [Cung cấp thông tin Pod cho container thông qua biến môi trường](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information)
