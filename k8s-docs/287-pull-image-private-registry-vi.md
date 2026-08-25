# Pull image từ một private registry (Pull an Image from a Private Registry)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3b. Cấu hình ứng dụng](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod),
bài 2/12 · Kiểm chứng ở [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) phần B7.3 — tạo Secret loại
`kubernetes.io/dockerconfigjson` rồi mổ xẻ nội dung thật của nó.

Bài viết cho tình huống có tài khoản Docker Hub và một image riêng tư thật. Cluster lab không có
private registry nào, nên ở lần đọc này bạn chỉ thực hành **nửa đầu**: tạo Secret và đọc ngược nội
dung của nó. Nửa sau — pull thật một image riêng tư — chỉ đọc để biết cấu trúc; lý do ghi ở bảng
thứ hai trong mục 1.1 của [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md).

**Phải hiểu ở lần đọc này:**

- Cluster dùng Secret **loại `kubernetes.io/dockerconfigjson`** để xác thực với registry. Khi tự
  viết manifest (mục *Tạo Secret dựa trên thông tin xác thực có sẵn*) phải đủ ba điều: mục dữ liệu
  mang đúng tên `.dockerconfigjson`, chuỗi base64 **liền mạch không ngắt dòng**, và `type` đúng.
- Hai đường tạo Secret: chép lại `~/.docker/config.json` sẵn có bằng
  `--from-file=.dockerconfigjson=<đường-dẫn>`, hoặc gõ thẳng credential bằng
  `kubectl create secret docker-registry` (mục *Tạo Secret bằng cách cung cấp thông tin xác thực
  trên dòng lệnh*) — kèm cảnh báo: gõ secret trên dòng lệnh khiến nó nằm lại trong **lịch sử
  shell** và có thể bị người dùng khác trên máy nhìn thấy lúc `kubectl` đang chạy.
- Nội dung thật của Secret, mục *Kiểm tra Secret `regcred`*: giải base64 field `.dockerconfigjson`
  ra một JSON, trong JSON đó field `auth` **lại là base64** của `username:password`. Hai lớp base64
  chồng lên nhau, và **không lớp nào là mã hóa** — bài kết luận Secret chứa token ủy quyền y hệt
  file `~/.docker/config.json` trên máy bạn.
- `imagePullSecrets` khai ở **cấp Pod** (`spec.imagePullSecrets`), không phải trong `containers`.
  Ghi chú của bài nói rõ Secret phải tồn tại **trong đúng namespace nơi bạn định nghĩa Pod**.
- Phân biệt hai kiểu hỏng: Pod ở `ImagePullBackOff` thì xem `kubectl describe pod`; nếu thấy event
  reason **`FailedToRetrieveImagePullSecret`** thì Kubernetes **không tìm thấy Secret mang tên đó**
  (sai tên hoặc sai namespace) — khác hẳn với việc tìm thấy Secret nhưng registry từ chối.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Đăng nhập vào Docker Hub* và file `~/.docker/config.json` sinh ra từ `docker login` | cần công cụ `docker` và một tài khoản registry thật; cluster lab không có registry riêng để đăng nhập | cách Kubernetes diễn giải file `config.json` đã học ở bài [40](40-images-vi.md), [giai đoạn 2](00-ALO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime) |
| Mục *Tạo một Pod sử dụng Secret của bạn* — pull thật một image riêng tư | cần registry và tài khoản thật, nằm ngoài baseline của Lab 00 | không kiểm chứng được trong lộ trình; lý do ghi ở bảng thứ hai mục 1.1 của [Lab 3b](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) |
| Link *thêm image pull secret vào một service account* ở mục *Tiếp theo* | cần ServiceAccount, chưa học ở giai đoạn 3 | bài [279](279-configure-service-account-vi.md), [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Trên `lab-k8s-master` bạn chưa từng chạy `docker login` ở đâu cả. Bạn vẫn tạo được Secret cho
   registry bằng đường nào, `type` của Secret đó là gì, và mục dữ liệu bên trong mang tên gì?
2. Bạn chạy `kubectl get secret regcred -o jsonpath='{.data.\.dockerconfigjson}' | base64 --decode`
   và trong JSON thấy `"auth":"c3R...zE2"`. Chuỗi đó là gì, đọc ra bằng cách nào, và kết luận gì về
   mức bảo vệ của Secret registry?
3. **Câu bẫy.** Pod của bạn nằm ở namespace `lab-3b` và khai `imagePullSecrets` trỏ tới `regcred`,
   nhưng `regcred` lại được tạo ở namespace `default`. Pod hỏng kiểu gì, event nào xuất hiện, và
   event đó nói lên điều gì khác với "registry từ chối mật khẩu"?
4. Một Pod có hai container lấy image từ hai registry khác nhau. Bài cho phép làm thế nào, và
   chuyện gì xảy ra nếu không thông tin xác thực nào khớp với registry của một image?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Dùng đường thứ hai: **`kubectl create secret docker-registry`** với `--docker-server`,
   `--docker-username`, `--docker-password`, `--docker-email` — đường này không cần file
   `~/.docker/config.json` có sẵn, nên không cần `docker login` trước. Loại Secret sinh ra là
   **`kubernetes.io/dockerconfigjson`**, và mục dữ liệu bên trong mang tên **`.dockerconfigjson`**
   (có dấu chấm đứng đầu). Bài cũng nhắc: bù lại, credential gõ trên dòng lệnh nằm lại trong lịch
   sử shell.
2. Đó là **base64 của `username:password` nối bằng dấu hai chấm**. Đọc ra bằng
   `echo "c3R...zE2" | base64 --decode`, kết quả dạng `janedoe:xxxxxxxxxxx`. Kết luận: Secret
   registry **không được bảo vệ gì hơn một chuỗi base64 lồng trong một chuỗi base64** — bài nói
   thẳng dữ liệu của Secret "chứa token ủy quyền tương tự như file `~/.docker/config.json` cục bộ
   của bạn". Ai đọc được Secret là có credential đăng nhập registry.
3. Pod vào trạng thái **`ImagePullBackOff`**, và `kubectl describe pod` cho ra event reason
   **`FailedToRetrieveImagePullSecret`** với thông điệp "Unable to retrieve some image pull
   secrets". Đây là chỗ dễ nhầm: event đó **không** nói registry từ chối bạn — nó nói Kubernetes
   **không tìm thấy Secret có tên đó**. Ghi chú của bài chỉ đúng chỗ phải sửa: "namespace cần dùng
   chính là namespace nơi bạn đã định nghĩa Pod", nên Secret ở `default` là vô hình với Pod ở
   `lab-3b`. Hai việc phải kiểm: **Secret có tồn tại đúng namespace không, và tên có gõ đúng không**.
4. Được: **một Pod dùng được nhiều `imagePullSecrets`, và mỗi Secret chứa được nhiều thông tin xác
   thực**. Việc pull được thử bằng **từng thông tin xác thực khớp với registry đó**. Nếu không có
   cái nào khớp, việc pull vẫn được thử — nhưng **không có ủy quyền**, hoặc dùng cấu hình riêng đặc
   thù của runtime; tức là image công khai vẫn kéo được, image riêng tư thì hỏng.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
