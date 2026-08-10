# Pull image từ một private registry (Pull an Image from a Private Registry)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/>

Trang này hướng dẫn cách tạo một Pod sử dụng Secret để pull image từ một container image
registry hoặc repository riêng tư (private). Có rất nhiều private registry đang được sử dụng.
Tác vụ này dùng [Docker Hub](https://www.docker.com/products/docker-hub) làm registry ví dụ.

> **Ghi chú:** Mục này liên kết tới một dự án của bên thứ ba cung cấp chức năng mà Kubernetes
> cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về dự án đó. Để thêm một dự án
> vào danh sách này, hãy đọc
> [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content)
> trước khi gửi thay đổi.

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
  tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
  không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
  bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
  các sân chơi (playground) Kubernetes sau:

  * [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
  * [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
  * [KodeKloud](https://kodekloud.com/public-playgrounds)

* Để làm bài thực hành này, bạn cần công cụ dòng lệnh `docker` và một
  [Docker ID](https://docs.docker.com/docker-id/) mà bạn biết mật khẩu.
* Nếu bạn dùng một private container registry khác, bạn cần công cụ dòng lệnh của registry đó
  và mọi thông tin đăng nhập (login information) cho registry.

## Đăng nhập vào Docker Hub (Log in to Docker Hub)

Trên máy tính của bạn, bạn phải xác thực với một registry thì mới pull được một private image.

Dùng công cụ `docker` để đăng nhập vào Docker Hub. Xem mục _log in_ của
[tài khoản Docker ID](https://docs.docker.com/docker-id/#log-in) để biết thêm thông tin.

```shell
docker login
```

Khi được nhắc, hãy nhập Docker ID của bạn, rồi nhập thông tin xác thực (credential) mà bạn muốn
dùng (access token, hoặc mật khẩu cho Docker ID của bạn).

Quá trình đăng nhập tạo mới hoặc cập nhật file `config.json` chứa một token ủy quyền
(authorization token). Hãy xem lại
[cách Kubernetes diễn giải file này](./40-images-vi.md#config-json).

Xem file `config.json`:

```shell
cat ~/.docker/config.json
```

Kết quả đầu ra chứa một phần tương tự như sau:

```json
{
    "auths": {
        "https://index.docker.io/v1/": {
            "auth": "c3R...zE2"
        }
    }
}
```

> **Ghi chú:** Nếu bạn dùng một kho lưu trữ thông tin xác thực Docker (Docker credentials
> store), bạn sẽ không thấy mục `auth` đó mà thay vào đó là mục `credsStore` với giá trị là tên
> của kho lưu trữ. Trong trường hợp đó, bạn có thể tạo secret trực tiếp. Xem
> [Tạo Secret bằng cách cung cấp thông tin xác thực trên dòng lệnh](#create-a-secret-by-providing-credentials-on-the-command-line).

## Tạo Secret dựa trên thông tin xác thực có sẵn (Create a Secret based on existing credentials) {#registry-secret-existing-credentials}

Một cluster Kubernetes dùng Secret kiểu `kubernetes.io/dockerconfigjson` để xác thực với một
container registry nhằm pull một private image.

Nếu bạn đã chạy `docker login`, bạn có thể sao chép thông tin xác thực đó vào Kubernetes:

```shell
kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=<path/to/.docker/config.json> \
    --type=kubernetes.io/dockerconfigjson
```

Nếu bạn cần kiểm soát nhiều hơn (ví dụ, để đặt namespace hoặc label cho secret mới), bạn có thể
tùy biến Secret trước khi lưu nó. Hãy chắc chắn rằng bạn:

- đặt tên của mục dữ liệu (data item) là `.dockerconfigjson`
- mã hóa base64 file cấu hình Docker rồi dán chuỗi đó, liền mạch không ngắt dòng, làm giá trị
  cho field `data[".dockerconfigjson"]`
- đặt `type` là `kubernetes.io/dockerconfigjson`

Ví dụ:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myregistrykey
  namespace: awesomeapps
data:
  .dockerconfigjson: UmVhbGx5IHJlYWxseSByZWVlZWVlZWVlZWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWFhYWxsbGxsbGxsbGxsbGxsbGxsbGxsbGxsbGxsbGxsbGx5eXl5eXl5eXl5eXl5eXl5eXl5eSBsbGxsbGxsbGxsbGxsbG9vb29vb29vb29vb29vb29vb29vb29vb29vb25ubm5ubm5ubm5ubm5ubm5ubm5ubm5ubmdnZ2dnZ2dnZ2dnZ2dnZ2dnZ2cgYXV0aCBrZXlzCg==
type: kubernetes.io/dockerconfigjson
```

Nếu bạn nhận được thông báo lỗi `error: no objects passed to create`, điều đó có thể có nghĩa
là chuỗi mã hóa base64 không hợp lệ. Nếu bạn nhận được thông báo lỗi kiểu
`Secret "myregistrykey" is invalid: data[.dockerconfigjson]: invalid value ...`, điều đó nghĩa
là chuỗi mã hóa base64 trong dữ liệu đã được giải mã thành công, nhưng không thể được phân tích
(parse) như một file `.docker/config.json`.

## Tạo Secret bằng cách cung cấp thông tin xác thực trên dòng lệnh (Create a Secret by providing credentials on the command line) {#create-a-secret-by-providing-credentials-on-the-command-line}

Tạo Secret này, đặt tên nó là `regcred`:

```shell
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```

trong đó:

* `<your-registry-server>` là FQDN của Private Docker Registry của bạn.
  Dùng `https://index.docker.io/v1/` cho DockerHub.
* `<your-name>` là username Docker của bạn.
* `<your-pword>` là mật khẩu Docker của bạn.
* `<your-email>` là email Docker của bạn.

Bạn đã thiết lập thành công thông tin xác thực Docker của mình trong cluster dưới dạng một
Secret tên là `regcred`.

> **Ghi chú:** Việc gõ secret trên dòng lệnh có thể khiến chúng được lưu trong lịch sử shell
> của bạn mà không được bảo vệ, và các secret đó cũng có thể bị những người dùng khác trên máy
> của bạn nhìn thấy trong lúc `kubectl` đang chạy.

## Kiểm tra Secret `regcred` (Inspecting the Secret `regcred`)

Để hiểu nội dung của Secret `regcred` mà bạn vừa tạo, hãy bắt đầu bằng cách xem Secret ở định
dạng YAML:

```shell
kubectl get secret regcred --output=yaml
```

Kết quả đầu ra tương tự như sau:

```yaml
apiVersion: v1
kind: Secret
metadata:
  ...
  name: regcred
  ...
data:
  .dockerconfigjson: eyJodHRwczovL2luZGV4L ... J0QUl6RTIifX0=
type: kubernetes.io/dockerconfigjson
```

Giá trị của field `.dockerconfigjson` là biểu diễn base64 của thông tin xác thực Docker của
bạn.

Để hiểu có gì trong field `.dockerconfigjson`, hãy chuyển dữ liệu secret sang định dạng đọc
được:

```shell
kubectl get secret regcred --output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode
```

Kết quả đầu ra tương tự như sau:

```json
{"auths":{"your.private.registry.example.com":{"username":"janedoe","password":"xxxxxxxxxxx","email":"jdoe@example.com","auth":"c3R...zE2"}}}
```

Để hiểu có gì trong field `auth`, hãy chuyển dữ liệu mã hóa base64 sang định dạng đọc được:

```shell
echo "c3R...zE2" | base64 --decode
```

Kết quả đầu ra — username và password được nối với nhau bằng dấu `:` — tương tự như sau:

```none
janedoe:xxxxxxxxxxx
```

Hãy để ý rằng dữ liệu của Secret chứa token ủy quyền tương tự như file `~/.docker/config.json`
cục bộ của bạn.

Bạn đã thiết lập thành công thông tin xác thực Docker của mình dưới dạng một Secret tên
`regcred` trong cluster.

## Tạo một Pod sử dụng Secret của bạn (Create a Pod that uses your Secret)

Dưới đây là manifest cho một Pod ví dụ cần truy cập thông tin xác thực Docker của bạn trong
`regcred`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg-container
    image: <your-private-image>
  imagePullSecrets:
  - name: regcred
```

Tải file trên về máy tính của bạn:

```shell
curl -L -o my-private-reg-pod.yaml https://k8s.io/examples/pods/private-reg-pod.yaml
```

Trong file `my-private-reg-pod.yaml`, thay `<your-private-image>` bằng đường dẫn tới một image
trong private registry, chẳng hạn:

```none
your.private.registry.example.com/janedoe/jdoe-private:v1
```

Để pull image từ private registry, Kubernetes cần thông tin xác thực. Field `imagePullSecrets`
trong file cấu hình chỉ định rằng Kubernetes nên lấy thông tin xác thực từ một Secret tên là
`regcred`.

Tạo một Pod sử dụng Secret của bạn, và xác minh rằng Pod đang chạy:

```shell
kubectl apply -f my-private-reg-pod.yaml
kubectl get pod private-reg
```

> **Ghi chú:** Để dùng image pull secret cho một Pod (hoặc một Deployment, hoặc đối tượng khác
> có Pod template mà bạn đang sử dụng), bạn cần bảo đảm rằng Secret phù hợp thực sự tồn tại
> trong đúng namespace. Namespace cần dùng chính là namespace nơi bạn đã định nghĩa Pod.

Ngoài ra, trong trường hợp Pod không khởi động được và có trạng thái `ImagePullBackOff`, hãy
xem các event của Pod:

```shell
kubectl describe pod private-reg
```

Nếu sau đó bạn thấy một event có reason là `FailedToRetrieveImagePullSecret`, nghĩa là
Kubernetes không tìm thấy Secret với tên đã cho (`regcred`, trong ví dụ này).

Hãy bảo đảm rằng Secret bạn đã chỉ định tồn tại, và tên của nó được viết đúng.
```shell
Events:
  ...  Reason                           ...  Message
       ------                                -------
  ...  FailedToRetrieveImagePullSecret  ...  Unable to retrieve some image pull secrets (<regcred>); attempting to pull the image may not succeed.
```

## Sử dụng image từ nhiều registry (Using images from multiple registries)

Một Pod có thể có nhiều container, và mỗi container image có thể đến từ một registry khác nhau.
Bạn có thể dùng nhiều `imagePullSecrets` với một Pod, và mỗi secret có thể chứa nhiều thông tin
xác thực.

Việc pull image sẽ được thử bằng từng thông tin xác thực khớp với registry. Nếu không có thông
tin xác thực nào khớp với registry, việc pull image sẽ được thử mà không có ủy quyền, hoặc dùng
cấu hình riêng đặc thù của runtime.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [Secret](./109-secret-vi.md)
  * hoặc đọc tài liệu tham khảo API cho
    [Secret](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/secret-v1/)
* Tìm hiểu thêm về
  [việc sử dụng private registry](./40-images-vi.md#sử-dụng-private-registry-using-a-private-registry).
* Tìm hiểu thêm về
  [thêm image pull secret vào một service account](./279-configure-service-account-vi.md#thêm-imagepullsecrets-vào-một-service-account-add-imagepullsecrets-to-a-service-account).
* Xem [kubectl create secret docker-registry](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#-em-secret-docker-registry-em-).
* Xem field `imagePullSecrets` trong
  [định nghĩa container](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#containers)
  của một Pod
