# Cấu hình truy cập nhiều cluster (Configure Access to Multiple Clusters)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/>
>
> Trang này hướng dẫn cách cấu hình truy cập nhiều cluster bằng các file cấu hình
> (kubeconfig), và cách chuyển đổi nhanh giữa các cluster bằng lệnh
> `kubectl config use-context`.

Trang này hướng dẫn cách cấu hình truy cập nhiều cluster bằng cách sử dụng
các file cấu hình. Sau khi các cluster, user và context của bạn đã được định nghĩa
trong một hoặc nhiều file cấu hình, bạn có thể chuyển đổi nhanh giữa các cluster bằng
lệnh `kubectl config use-context`.

> **Ghi chú:**
>
> File được dùng để cấu hình truy cập vào một cluster đôi khi được gọi là
> *file kubeconfig*. Đây là cách gọi chung cho các file cấu hình.
> Điều đó không có nghĩa là tồn tại một file có tên `kubeconfig`.

> **Cảnh báo:**
>
> Chỉ sử dụng các file kubeconfig từ những nguồn đáng tin cậy. Việc sử dụng một file
> kubeconfig được chế tác đặc biệt có thể dẫn đến thực thi mã độc hoặc lộ file.
> Nếu bạn buộc phải dùng một file kubeconfig không đáng tin cậy, hãy kiểm tra nó cẩn thận
> trước, giống như cách bạn kiểm tra một shell script.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình
để giao tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất
hai node không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo
một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc sử dụng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra kubectl đã được cài đặt hay chưa, hãy chạy `kubectl version --client`.
Phiên bản kubectl nên
[chênh lệch không quá một phiên bản minor](https://kubernetes.io/releases/version-skew-policy/#kubectl)
so với API server của cluster.

## Định nghĩa cluster, user và context (Define clusters, users, and contexts)

Giả sử bạn có hai cluster, một cluster dành cho công việc phát triển (development) và một
cluster dành cho công việc kiểm thử (test). Trong cluster `development`, các lập trình viên
frontend làm việc trong một namespace tên là `frontend`, còn các lập trình viên storage làm
việc trong một namespace tên là `storage`. Trong cluster `test`, các lập trình viên làm việc
trong namespace default, hoặc họ tự tạo các namespace phụ trợ khi thấy cần. Truy cập vào
cluster development yêu cầu xác thực (authentication) bằng certificate. Truy cập vào cluster
test yêu cầu xác thực bằng username và password.

Tạo một thư mục tên là `config-exercise`. Trong thư mục
`config-exercise`, tạo một file tên là `config-demo` với nội dung sau:

```yaml
apiVersion: v1
kind: Config
preferences: {}

clusters:
- cluster:
  name: development
- cluster:
  name: test

users:
- name: developer
- name: experimenter

contexts:
- context:
  name: dev-frontend
- context:
  name: dev-storage
- context:
  name: exp-test
```

Một file cấu hình mô tả các cluster, user và context. File `config-demo` của bạn
có bộ khung để mô tả hai cluster, hai user và ba context.

Đi tới thư mục `config-exercise`. Nhập các lệnh sau để thêm thông tin chi tiết về cluster
vào file cấu hình của bạn:

```shell
kubectl config --kubeconfig=config-demo set-cluster development --server=https://1.2.3.4 --certificate-authority=fake-ca-file
kubectl config --kubeconfig=config-demo set-cluster test --server=https://5.6.7.8 --insecure-skip-tls-verify
```

Thêm thông tin chi tiết về user vào file cấu hình của bạn:

> **Thận trọng:**
>
> Lưu trữ password trong cấu hình client của Kubernetes là rủi ro. Một lựa chọn tốt hơn là
> dùng một credential plugin và lưu trữ chúng riêng biệt. Xem:
> [client-go credential plugins](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins)

```shell
kubectl config --kubeconfig=config-demo set-credentials developer --client-certificate=fake-cert-file --client-key=fake-key-seefile
kubectl config --kubeconfig=config-demo set-credentials experimenter --username=exp --password=some-password
```

> **Ghi chú:**
>
> - Để xóa một user, bạn có thể chạy `kubectl --kubeconfig=config-demo config unset users.<name>`
> - Để xóa một cluster, bạn có thể chạy `kubectl --kubeconfig=config-demo config unset clusters.<name>`
> - Để xóa một context, bạn có thể chạy `kubectl --kubeconfig=config-demo config unset contexts.<name>`

Thêm thông tin chi tiết về context vào file cấu hình của bạn:

```shell
kubectl config --kubeconfig=config-demo set-context dev-frontend --cluster=development --namespace=frontend --user=developer
kubectl config --kubeconfig=config-demo set-context dev-storage --cluster=development --namespace=storage --user=developer
kubectl config --kubeconfig=config-demo set-context exp-test --cluster=test --namespace=default --user=experimenter
```

Mở file `config-demo` của bạn để xem các thông tin đã được thêm vào. Thay vì mở file
`config-demo`, bạn cũng có thể dùng lệnh `config view`.

```shell
kubectl config --kubeconfig=config-demo view
```

Kết quả hiển thị hai cluster, hai user và ba context:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: fake-ca-file
    server: https://1.2.3.4
  name: development
- cluster:
    insecure-skip-tls-verify: true
    server: https://5.6.7.8
  name: test
contexts:
- context:
    cluster: development
    namespace: frontend
    user: developer
  name: dev-frontend
- context:
    cluster: development
    namespace: storage
    user: developer
  name: dev-storage
- context:
    cluster: test
    namespace: default
    user: experimenter
  name: exp-test
current-context: ""
kind: Config
preferences: {}
users:
- name: developer
  user:
    client-certificate: fake-cert-file
    client-key: fake-key-file
- name: experimenter
  user:
    # Ghi chú của tài liệu (comment này KHÔNG phải là một phần của output lệnh).
    # Lưu trữ password trong cấu hình client của Kubernetes là rủi ro.
    # Một lựa chọn tốt hơn là dùng một credential plugin
    # và lưu trữ thông tin đăng nhập (credentials) riêng biệt.
    # Xem https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins
    password: some-password
    username: exp
```

Các giá trị `fake-ca-file`, `fake-cert-file` và `fake-key-file` ở trên là các placeholder
cho đường dẫn (pathname) của các file certificate. Bạn cần thay chúng bằng đường dẫn thực tế
của các file certificate trong môi trường của bạn.

Đôi khi bạn có thể muốn dùng dữ liệu mã hóa Base64 nhúng trực tiếp tại đây thay vì các file
certificate riêng biệt; trong trường hợp đó bạn cần thêm hậu tố `-data` vào các key, ví dụ,
`certificate-authority-data`, `client-certificate-data`, `client-key-data`.

Mỗi context là một bộ ba (cluster, user, namespace). Ví dụ, context
`dev-frontend` nói rằng, "Dùng thông tin đăng nhập của user `developer`
để truy cập namespace `frontend` của cluster `development`".

Thiết lập context hiện tại:

```shell
kubectl config --kubeconfig=config-demo use-context dev-frontend
```

Từ giờ, mỗi khi bạn nhập một lệnh `kubectl`, hành động sẽ áp dụng cho cluster
và namespace được liệt kê trong context `dev-frontend`. Và lệnh đó sẽ dùng
thông tin đăng nhập của user được liệt kê trong context `dev-frontend`.

Để chỉ xem thông tin cấu hình gắn với context hiện tại, hãy dùng flag `--minify`.

```shell
kubectl config --kubeconfig=config-demo view --minify
```

Kết quả hiển thị thông tin cấu hình gắn với context `dev-frontend`:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: fake-ca-file
    server: https://1.2.3.4
  name: development
contexts:
- context:
    cluster: development
    namespace: frontend
    user: developer
  name: dev-frontend
current-context: dev-frontend
kind: Config
preferences: {}
users:
- name: developer
  user:
    client-certificate: fake-cert-file
    client-key: fake-key-file
```

Bây giờ giả sử bạn muốn làm việc một lúc trong cluster test.

Đổi context hiện tại sang `exp-test`:

```shell
kubectl config --kubeconfig=config-demo use-context exp-test
```

Bây giờ, bất kỳ lệnh `kubectl` nào bạn đưa ra sẽ áp dụng cho namespace default của
cluster `test`. Và lệnh đó sẽ dùng thông tin đăng nhập của user được liệt kê
trong context `exp-test`.

Xem cấu hình gắn với context hiện tại mới, `exp-test`.

```shell
kubectl config --kubeconfig=config-demo view --minify
```

Cuối cùng, giả sử bạn muốn làm việc một lúc trong namespace `storage` của
cluster `development`.

Đổi context hiện tại sang `dev-storage`:

```shell
kubectl config --kubeconfig=config-demo use-context dev-storage
```

Xem cấu hình gắn với context hiện tại mới, `dev-storage`.

```shell
kubectl config --kubeconfig=config-demo view --minify
```

## Tạo file cấu hình thứ hai (Create a second configuration file)

Trong thư mục `config-exercise`, tạo một file tên là `config-demo-2` với nội dung sau:

```yaml
apiVersion: v1
kind: Config
preferences: {}

contexts:
- context:
    cluster: development
    namespace: ramp
    user: developer
  name: dev-ramp-up
```

File cấu hình trên định nghĩa một context mới có tên `dev-ramp-up`.

## Thiết lập biến môi trường KUBECONFIG (Set the KUBECONFIG environment variable)

Kiểm tra xem bạn có biến môi trường tên là `KUBECONFIG` hay không. Nếu có, hãy lưu lại
giá trị hiện tại của biến môi trường `KUBECONFIG`, để bạn có thể khôi phục nó sau này.
Ví dụ:

### Linux

```shell
export KUBECONFIG_SAVED="$KUBECONFIG"
```

### Windows PowerShell

```powershell
$Env:KUBECONFIG_SAVED=$ENV:KUBECONFIG
```

Biến môi trường `KUBECONFIG` là một danh sách các đường dẫn tới các file cấu hình.
Danh sách này phân tách bằng dấu hai chấm trên Linux và Mac, và phân tách bằng dấu
chấm phẩy trên Windows. Nếu bạn có biến môi trường `KUBECONFIG`, hãy làm quen với
các file cấu hình trong danh sách đó.

Tạm thời nối thêm hai đường dẫn vào biến môi trường `KUBECONFIG` của bạn. Ví dụ:

### Linux

```shell
export KUBECONFIG="${KUBECONFIG}:config-demo:config-demo-2"
```

### Windows PowerShell

```powershell
$Env:KUBECONFIG=("config-demo;config-demo-2")
```

Trong thư mục `config-exercise`, nhập lệnh sau:

```shell
kubectl config view
```

Kết quả hiển thị thông tin đã được hợp nhất (merge) từ tất cả các file được liệt kê trong
biến môi trường `KUBECONFIG` của bạn. Đặc biệt, hãy chú ý rằng thông tin hợp nhất có
context `dev-ramp-up` từ file `config-demo-2` và ba context từ
file `config-demo`:

```yaml
contexts:
- context:
    cluster: development
    namespace: frontend
    user: developer
  name: dev-frontend
- context:
    cluster: development
    namespace: ramp
    user: developer
  name: dev-ramp-up
- context:
    cluster: development
    namespace: storage
    user: developer
  name: dev-storage
- context:
    cluster: test
    namespace: default
    user: experimenter
  name: exp-test
```

Để biết thêm thông tin về cách các file kubeconfig được hợp nhất, xem
[Tổ chức quyền truy cập cluster bằng file kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
(đã có [bản dịch tiếng Việt](111-kubeconfig-vi.md)).

## Khám phá thư mục $HOME/.kube (Explore the $HOME/.kube directory)

Nếu bạn đã có sẵn một cluster, và bạn có thể dùng `kubectl` để tương tác với
cluster đó, thì nhiều khả năng bạn có một file tên là `config` trong thư mục
`$HOME/.kube`.

Đi tới `$HOME/.kube` và xem trong đó có những file nào. Thông thường sẽ có một file tên là
`config`. Cũng có thể có các file cấu hình khác trong thư mục này. Hãy làm quen sơ qua với
nội dung của các file đó.

## Nối $HOME/.kube/config vào biến môi trường KUBECONFIG (Append $HOME/.kube/config to your KUBECONFIG environment variable)

Nếu bạn có file `$HOME/.kube/config`, và nó chưa được liệt kê trong biến môi trường
`KUBECONFIG` của bạn, hãy nối nó vào biến môi trường `KUBECONFIG` ngay bây giờ.
Ví dụ:

### Linux

```shell
export KUBECONFIG="${KUBECONFIG}:${HOME}/.kube/config"
```

### Windows Powershell

```powershell
$Env:KUBECONFIG="$Env:KUBECONFIG;$HOME\.kube\config"
```

Xem thông tin cấu hình đã hợp nhất từ tất cả các file hiện được liệt kê
trong biến môi trường `KUBECONFIG` của bạn. Trong thư mục config-exercise, nhập:

```shell
kubectl config view
```

## Dọn dẹp (Clean up)

Đưa biến môi trường `KUBECONFIG` của bạn trở về giá trị ban đầu. Ví dụ:

### Linux

```shell
export KUBECONFIG="$KUBECONFIG_SAVED"
```

### Windows PowerShell

```powershell
$Env:KUBECONFIG=$ENV:KUBECONFIG_SAVED
```

## Kiểm tra chủ thể (subject) mà kubeconfig đại diện (Check the subject represented by the kubeconfig)

Không phải lúc nào cũng dễ thấy rõ bạn sẽ nhận được những thuộc tính nào (username, groups)
sau khi xác thực với cluster. Việc này thậm chí còn khó hơn nếu bạn đang quản lý nhiều hơn
một cluster cùng lúc.

Có một lệnh con của `kubectl` để kiểm tra các thuộc tính chủ thể, chẳng hạn như username,
cho context Kubernetes client mà bạn đã chọn: `kubectl auth whoami`.

Đọc [Truy cập API để lấy thông tin xác thực của một client](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#self-subject-review)
để tìm hiểu chi tiết hơn về điều này.

## Tiếp theo (What's next)

* [Tổ chức quyền truy cập cluster bằng file kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) — đã có [bản dịch tiếng Việt](111-kubeconfig-vi.md)
* [kubectl config](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#config)
