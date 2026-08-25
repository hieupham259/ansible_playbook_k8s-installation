# Secret (Secrets)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/configuration/secret/>
>
> Triển khai và cập nhật Secret cùng cấu hình ứng dụng mà không cần build lại image
> và không làm lộ Secret trong cấu hình stack của bạn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3b](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod), bài 3/7 ·
Kiểm chứng ở Lab 3b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài dài, nhưng hơn một nửa độ dài là **danh mục tám loại Secret built-in** — tra cứu khi cần,
không phải học thuộc. Điều duy nhất bắt buộc phải khắc vào đầu ở lần đọc này nằm ngay trong
khối *Thận trọng* đầu bài: **Secret mặc định KHÔNG được mã hóa**. Nếu đọc xong mà vẫn nghĩ
"đã là Secret thì an toàn" thì coi như chưa đọc.

Việc bật *Encryption at Rest* phải sửa cấu hình kube-apiserver nên không làm ở giai đoạn này;
nó là [nợ lab](labs/README.md#5-sổ-nợ-lab) và được trả ở giai đoạn 22 trong phần checkpoint tasks.

**Phải hiểu ở lần đọc này:**

- Secret **lưu không mã hóa** trong etcd; `data` chỉ được mã hóa **base64**, mà base64 —
  đúng như bài viết ở mục *Secret cấu hình Docker* — "bị che mờ nhưng không hề bí mật". Bốn
  bước tối thiểu để dùng Secret an toàn nằm trong khối *Thận trọng* đầu bài.
- Ranh giới quyền, mục *An toàn thông tin cho Secret*: ai được phép **tạo Pod** trong một
  namespace đều có thể lợi dụng quyền đó để đọc mọi Secret trong namespace; cấp quyền **list**
  hoặc **watch** trên Secret là cho đọc toàn bộ Secret của namespace chứ không riêng Secret nào.
- `data` phải là chuỗi base64, `stringData` nhận chuỗi thuần; mọi cặp trong `stringData` được
  gộp nội bộ vào `data`, và khi trùng key thì **`stringData` thắng** (mục *Ràng buộc về tên và
  dữ liệu của Secret*). Trần kích thước mỗi Secret là 1MiB.
- Trường `type` chỉ quyết định **validation và quy ước tên key**, không quyết định mức bảo
  vệ: `Opaque` là mặc định, và bài lặp lại nhiều lần rằng các loại built-in "chỉ được cung cấp
  để thuận tiện".
- Cách kubelet xử lý (cùng mục *An toàn thông tin cho Secret*): Secret chỉ được gửi tới node
  có Pod cần nó, kubelet lưu bản sao vào `tmpfs` để không ghi xuống lưu trữ bền vững, và xóa
  bản sao khi Pod bị xóa. Secret phải tồn tại trước Pod, trừ khi đánh dấu `optional: true`.
  Static Pod thì **không dùng được** Secret lẫn ConfigMap.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Secret token ServiceAccount*, API `TokenRequest`, projected volume | chưa học ServiceAccount và danh tính của Pod | giai đoạn 9, bài [118](118-service-accounts-vi.md) |
| *Secret bootstrap token* | là cơ chế đăng ký node lúc `kubeadm join` | giai đoạn 8, bài [02](02-create-cluster-kubeadm-vi.md) |
| *Secret TLS* và việc dùng nó cho Ingress | chưa học Ingress | giai đoạn 5 |
| *Secret để kéo container image*, `imagePullSecrets`, credential provider | phần gắn vào ServiceAccount cần RBAC | giai đoạn 9, bài [118](118-service-accounts-vi.md) |
| Bật *Encryption at Rest* cho Secret | phải sửa cấu hình kube-apiserver | [nợ lab](labs/README.md#5-sổ-nợ-lab), trả ở giai đoạn 22 |
| *Các lựa chọn thay thế cho Secret* — CSI provider, device plugin, CertificateSigningRequest | cần CSI và các cơ chế mở rộng | giai đoạn 6 và 14 |
| *Các thực hành tốt cho Kubernetes Secret* (link cuối bài) | là checklist hardening | giai đoạn 9, bài [121](121-secrets-good-practices-vi.md) |

---

Secret là một đối tượng chứa một lượng nhỏ dữ liệu nhạy cảm như mật khẩu,
token, hoặc khóa (key). Nếu không dùng Secret, những thông tin như vậy có thể
sẽ bị đặt trong đặc tả (specification) của Pod hoặc trong container image.
Sử dụng Secret nghĩa là bạn không cần đưa dữ liệu bí mật vào mã nguồn
ứng dụng của mình.

Vì Secret có thể được tạo độc lập với các Pod sử dụng chúng, nên rủi ro
Secret (và dữ liệu của nó) bị lộ trong quy trình tạo, xem và chỉnh sửa Pod
sẽ thấp hơn. Kubernetes, cũng như các ứng dụng chạy trong cluster của bạn,
còn có thể áp dụng thêm các biện pháp phòng ngừa với Secret, chẳng hạn như
tránh ghi dữ liệu nhạy cảm xuống bộ nhớ lưu trữ bền vững (nonvolatile storage).

Secret tương tự như ConfigMap nhưng được thiết kế chuyên biệt để chứa
dữ liệu bí mật.

> **Thận trọng:**
>
> Theo mặc định, Secret của Kubernetes được lưu trữ **không mã hóa** trong kho dữ liệu
> nền của API server (etcd). Bất kỳ ai có quyền truy cập API đều có thể lấy hoặc sửa
> một Secret, và bất kỳ ai có quyền truy cập etcd cũng vậy.
> Ngoài ra, bất kỳ ai được phép tạo Pod trong một namespace đều có thể lợi dụng quyền đó
> để đọc bất kỳ Secret nào trong namespace đó; điều này bao gồm cả quyền truy cập gián tiếp
> như khả năng tạo một Deployment.
>
> Để sử dụng Secret một cách an toàn, hãy thực hiện ít nhất các bước sau:
>
> 1. [Bật mã hóa dữ liệu lưu trữ (Encryption at Rest)](208-encrypt-data-vi.md) cho Secret.
> 2. [Bật hoặc cấu hình các quy tắc RBAC](https://kubernetes.io/docs/reference/access-authn-authz/authorization/) với
>    quyền truy cập tối thiểu (least-privilege) tới Secret.
> 3. Giới hạn quyền truy cập Secret cho những container cụ thể.
> 4. [Cân nhắc sử dụng các nhà cung cấp kho Secret bên ngoài](https://secrets-store-csi-driver.sigs.k8s.io/concepts.html#provider-for-the-secrets-store-csi-driver).
>
> Để có thêm hướng dẫn về quản lý và nâng cao tính bảo mật cho Secret của bạn, hãy tham khảo
> [Các thực hành tốt cho Kubernetes Secret](121-secrets-good-practices-vi.md).

Xem [An toàn thông tin cho Secret](#information-security-for-secrets) để biết thêm chi tiết.

## Công dụng của Secret (Uses for Secrets)

Bạn có thể dùng Secret cho các mục đích như sau:

- [Thiết lập biến môi trường cho container](334-distribute-credentials-secure-vi.md#define-container-environment-variables-using-secret-data).
- [Cung cấp thông tin xác thực (credentials) như khóa SSH hoặc mật khẩu cho Pod](334-distribute-credentials-secure-vi.md#provide-prod-test-creds).
- [Cho phép kubelet kéo (pull) container image từ các registry riêng tư](287-pull-image-private-registry-vi.md).

Control plane của Kubernetes cũng sử dụng Secret; ví dụ,
[Secret bootstrap token](#bootstrap-token-secrets) là một cơ chế giúp
tự động hóa việc đăng ký node.

### Trường hợp sử dụng: dotfile trong secret volume (Use case: dotfiles in a secret volume)

Bạn có thể làm cho dữ liệu của mình "ẩn" đi bằng cách định nghĩa một key bắt đầu
bằng dấu chấm. Key này đại diện cho một dotfile hay file "ẩn". Ví dụ, khi Secret
sau được mount vào một volume tên `secret-volume`, volume đó sẽ chứa một file duy nhất
tên là `.secret-file`, và container `dotfile-test-container` sẽ thấy file này
tại đường dẫn `/etc/secret-volume/.secret-file`.

> **Ghi chú:**
>
> Các file bắt đầu bằng dấu chấm sẽ bị ẩn khỏi kết quả của lệnh `ls -l`;
> bạn phải dùng `ls -la` mới thấy được chúng khi liệt kê nội dung thư mục.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dotfile-secret
data:
  .secret-file: dmFsdWUtMg0KDQo=
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-dotfiles-pod
spec:
  volumes:
    - name: secret-volume
      secret:
        secretName: dotfile-secret
  containers:
    - name: dotfile-test-container
      image: registry.k8s.io/busybox
      command:
        - ls
        - "-l"
        - "/etc/secret-volume"
      volumeMounts:
        - name: secret-volume
          readOnly: true
          mountPath: "/etc/secret-volume"
```

### Trường hợp sử dụng: Secret chỉ hiển thị với một container trong Pod (Use case: Secret visible to one container in a Pod)

Hãy xét một chương trình cần xử lý các HTTP request, thực hiện một số logic
nghiệp vụ phức tạp, rồi ký một số message bằng HMAC. Vì chương trình có logic
ứng dụng phức tạp, có thể tồn tại một lỗ hổng đọc file từ xa chưa được phát hiện
trong server, và lỗ hổng đó có thể làm lộ khóa riêng (private key) cho kẻ tấn công.

Bài toán này có thể được chia thành hai tiến trình (process) trong hai container:
một container frontend xử lý tương tác người dùng và logic nghiệp vụ nhưng không
thể thấy khóa riêng; và một container ký (signer) có thể thấy khóa riêng, và
phản hồi các yêu cầu ký đơn giản từ frontend (ví dụ, qua mạng localhost).

Với cách tiếp cận phân tách này, kẻ tấn công giờ đây phải lừa được application
server thực hiện một hành động khá tùy ý, việc này có thể khó hơn so với việc
khiến nó đọc một file.

### Các lựa chọn thay thế cho Secret (Alternatives to Secrets)

Thay vì dùng Secret để bảo vệ dữ liệu bí mật, bạn có thể chọn các cách thay thế.

Dưới đây là một số lựa chọn:

- Nếu thành phần cloud-native của bạn cần xác thực với một ứng dụng khác mà bạn
  biết chắc đang chạy trong cùng một Kubernetes cluster, bạn có thể dùng một
  [ServiceAccount](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens)
  và các token của nó để định danh client của bạn.
- Có những công cụ bên thứ ba mà bạn có thể chạy, bên trong hoặc bên ngoài cluster,
  để quản lý dữ liệu nhạy cảm. Ví dụ, một dịch vụ mà các Pod truy cập qua HTTPS,
  dịch vụ này chỉ tiết lộ Secret khi client xác thực đúng (ví dụ, bằng một
  ServiceAccount token).
- Để xác thực, bạn có thể triển khai một bộ ký (signer) tùy chỉnh cho chứng chỉ X.509, và dùng
  [CertificateSigningRequests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)
  để bộ ký tùy chỉnh đó cấp certificate cho các Pod cần chúng.
- Bạn có thể dùng một [device plugin](184-device-plugins-vi.md)
  để phơi bày (expose) phần cứng mã hóa cục bộ của node cho một Pod cụ thể. Ví dụ, bạn có thể
  lập lịch (schedule) các Pod tin cậy lên những node có Trusted Platform Module,
  được cấu hình ngoài luồng (out-of-band).

Bạn cũng có thể kết hợp hai hay nhiều lựa chọn trên, bao gồm cả lựa chọn sử dụng chính các đối tượng Secret.

Ví dụ: triển khai (hoặc cài đặt) một operator
lấy các session token ngắn hạn từ một dịch vụ bên ngoài, rồi tạo các Secret dựa trên
những session token ngắn hạn đó. Các Pod chạy trong cluster của bạn có thể sử dụng
các session token này, còn operator bảo đảm chúng luôn hợp lệ. Sự tách biệt này nghĩa là
bạn có thể chạy các Pod mà chúng không cần biết cơ chế chính xác của việc cấp phát và
làm mới các session token đó.

## Các loại Secret (Types of Secret) {#secret-types}

Khi tạo một Secret, bạn có thể chỉ định loại của nó bằng trường `type` của
tài nguyên [Secret](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/secret-v1/),
hoặc một số cờ (flag) dòng lệnh `kubectl` tương đương (nếu có).
Loại Secret được dùng để thuận tiện cho việc xử lý dữ liệu Secret bằng chương trình.

Kubernetes cung cấp sẵn một số loại built-in cho các kịch bản sử dụng phổ biến.
Các loại này khác nhau về những kiểm tra hợp lệ (validation) được thực hiện và
các ràng buộc mà Kubernetes áp đặt lên chúng.

| Loại built-in                         | Công dụng                               |
| ------------------------------------- |---------------------------------------- |
| `Opaque`                              | dữ liệu tùy ý do người dùng định nghĩa  |
| `kubernetes.io/service-account-token` | token của ServiceAccount                |
| `kubernetes.io/dockercfg`             | file `~/.dockercfg` đã tuần tự hóa (serialized) |
| `kubernetes.io/dockerconfigjson`      | file `~/.docker/config.json` đã tuần tự hóa |
| `kubernetes.io/basic-auth`            | thông tin xác thực cho xác thực cơ bản (basic authentication) |
| `kubernetes.io/ssh-auth`              | thông tin xác thực cho xác thực SSH     |
| `kubernetes.io/tls`                   | dữ liệu cho TLS client hoặc server      |
| `bootstrap.kubernetes.io/token`       | dữ liệu bootstrap token                 |

Bạn có thể định nghĩa và sử dụng loại Secret của riêng mình bằng cách gán một chuỗi
không rỗng làm giá trị `type` cho đối tượng Secret (chuỗi rỗng được coi là loại `Opaque`).

Kubernetes không áp đặt ràng buộc nào lên tên loại. Tuy nhiên, nếu bạn
dùng một trong các loại built-in, bạn phải đáp ứng mọi yêu cầu được định nghĩa
cho loại đó.

Nếu bạn định nghĩa một loại Secret để sử dụng công khai, hãy tuân theo quy ước
và cấu trúc tên loại Secret sao cho có tên miền (domain) của bạn đứng trước tên loại,
phân tách bằng dấu `/`. Ví dụ: `cloud-hosting.example.net/cloud-api-credentials`.

### Secret loại Opaque (Opaque Secrets)

`Opaque` là loại Secret mặc định nếu bạn không chỉ định rõ loại trong
manifest của Secret. Khi bạn tạo Secret bằng `kubectl`, bạn phải dùng
subcommand `generic` để chỉ ra loại Secret là `Opaque`. Ví dụ, lệnh sau
tạo một Secret rỗng có loại `Opaque`:

```shell
kubectl create secret generic empty-secret
kubectl get secret empty-secret
```

Kết quả trông như sau:

```
NAME           TYPE     DATA   AGE
empty-secret   Opaque   0      2m6s
```

Cột `DATA` cho biết số lượng mục dữ liệu được lưu trong Secret.
Trong trường hợp này, `0` nghĩa là bạn vừa tạo một Secret rỗng.

### Secret token ServiceAccount (ServiceAccount token Secrets)

Secret loại `kubernetes.io/service-account-token` được dùng để lưu
token định danh cho một ServiceAccount. Đây là cơ chế cũ (legacy)
cung cấp thông tin xác thực ServiceAccount dài hạn cho các Pod.

Từ Kubernetes v1.22 trở đi, cách tiếp cận được khuyến nghị là lấy
token ServiceAccount ngắn hạn, tự động xoay vòng (rotate) bằng cách dùng
API [`TokenRequest`](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/)
thay thế. Bạn có thể lấy các token ngắn hạn này bằng những phương pháp sau:

* Gọi API `TokenRequest` trực tiếp hoặc thông qua một API client như
  `kubectl`. Ví dụ, bạn có thể dùng lệnh
  [`kubectl create token`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-token-em-).
* Yêu cầu một token được mount trong một
  [projected volume](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#bound-service-account-token-volume)
  trong manifest Pod của bạn. Kubernetes sẽ tạo token và mount nó vào Pod.
  Token này tự động bị vô hiệu hóa khi Pod chứa nó bị xóa. Xem chi tiết tại
  [Khởi chạy một Pod sử dụng service account token projection](279-configure-service-account-vi.md#launch-a-pod-using-service-account-token-projection).

> **Ghi chú:**
>
> Bạn chỉ nên tạo Secret token ServiceAccount
> khi không thể dùng API `TokenRequest` để lấy token,
> và khi bạn chấp nhận được rủi ro bảo mật của việc lưu một token không hết hạn
> trong một đối tượng API có thể đọc được. Để biết cách làm, xem
> [Tạo thủ công một API token dài hạn cho ServiceAccount](279-configure-service-account-vi.md#manually-create-an-api-token-for-a-serviceaccount).

Khi dùng loại Secret này, bạn cần bảo đảm annotation
`kubernetes.io/service-account.name` được đặt thành tên của một
ServiceAccount đang tồn tại. Nếu bạn tạo cả hai đối tượng ServiceAccount và
Secret, bạn nên tạo đối tượng ServiceAccount trước.

Sau khi Secret được tạo, một controller của Kubernetes
sẽ điền một số trường khác như annotation `kubernetes.io/service-account.uid`, và
key `token` trong trường `data`, key này được điền bằng một token xác thực.

Ví dụ cấu hình sau khai báo một Secret token ServiceAccount:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-sa-sample
  annotations:
    kubernetes.io/service-account.name: "sa-name"
type: kubernetes.io/service-account-token
data:
  extra: YmFyCg==
```

Sau khi tạo Secret, hãy chờ Kubernetes điền key `token` vào trường `data`.

Xem tài liệu về [ServiceAccount](118-service-accounts-vi.md)
để biết thêm thông tin về cách ServiceAccount hoạt động.
Bạn cũng có thể xem trường `automountServiceAccountToken` và trường
`serviceAccountName` của
[`Pod`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#pod-v1-core)
để biết cách tham chiếu thông tin xác thực ServiceAccount từ bên trong Pod.

### Secret cấu hình Docker (Docker config Secrets)

Nếu bạn tạo một Secret để lưu thông tin xác thực dùng cho việc truy cập một registry chứa container image,
bạn phải dùng một trong các giá trị `type` sau cho Secret đó:

- `kubernetes.io/dockercfg`: lưu một file `~/.dockercfg` đã tuần tự hóa, đây là
  định dạng cũ để cấu hình dòng lệnh Docker. Trường `data` của Secret
  chứa key `.dockercfg` có giá trị là nội dung của file `~/.dockercfg`
  được mã hóa base64.
- `kubernetes.io/dockerconfigjson`: lưu một chuỗi JSON đã tuần tự hóa tuân theo
  cùng quy tắc định dạng với file `~/.docker/config.json`, là định dạng mới
  thay cho `~/.dockercfg`. Trường `data` của Secret phải chứa key
  `.dockerconfigjson` có giá trị là nội dung của file `~/.docker/config.json`
  được mã hóa base64.

Dưới đây là ví dụ cho một Secret loại `kubernetes.io/dockercfg`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-dockercfg
type: kubernetes.io/dockercfg
data:
  .dockercfg: |
    eyJhdXRocyI6eyJodHRwczovL2V4YW1wbGUvdjEvIjp7ImF1dGgiOiJvcGVuc2VzYW1lIn19fQo=
```

> **Ghi chú:**
>
> Nếu bạn không muốn thực hiện mã hóa base64, bạn có thể chọn dùng
> trường `stringData` thay thế.

Khi bạn tạo Secret cấu hình Docker bằng manifest, API server
sẽ kiểm tra xem key mong đợi có tồn tại trong trường `data` hay không, và
xác minh xem giá trị được cung cấp có thể phân tích (parse) thành một JSON hợp lệ hay không.
API server không kiểm tra xem chuỗi JSON đó có thực sự là một file cấu hình Docker hay không.

Bạn cũng có thể dùng `kubectl` để tạo Secret truy cập container registry,
chẳng hạn khi bạn không có sẵn file cấu hình Docker:

```shell
kubectl create secret docker-registry secret-tiger-docker \
  --docker-email=tiger@acme.example \
  --docker-username=tiger \
  --docker-password=pass1234 \
  --docker-server=my-registry.example:5000
```

Lệnh này tạo một Secret loại `kubernetes.io/dockerconfigjson`.

Lấy trường `.data.dockerconfigjson` từ Secret mới đó và giải mã dữ liệu:

```shell
kubectl get secret secret-tiger-docker -o jsonpath='{.data.*}' | base64 -d
```

Kết quả tương đương với tài liệu JSON sau (đồng thời cũng là một
file cấu hình Docker hợp lệ):

```json
{
  "auths": {
    "my-registry.example:5000": {
      "username": "tiger",
      "password": "pass1234",
      "email": "tiger@acme.example",
      "auth": "dGlnZXI6cGFzczEyMzQ="
    }
  }
}
```

> **Thận trọng:**
>
> Giá trị `auth` ở trên được mã hóa base64; nó bị che mờ nhưng không hề bí mật.
> Bất kỳ ai đọc được Secret đó đều có thể biết được bearer token truy cập registry.
>
> Bạn nên dùng [credential provider](225-kubelet-credential-provider-vi.md) để cung cấp pull secret một cách linh hoạt và an toàn theo nhu cầu (on-demand).

### Secret xác thực cơ bản (Basic authentication Secret)

Loại `kubernetes.io/basic-auth` được cung cấp để lưu thông tin xác thực cần cho
xác thực cơ bản (basic authentication). Khi dùng loại Secret này, trường `data` của
Secret phải chứa một trong hai key sau:

- `username`: tên người dùng để xác thực
- `password`: mật khẩu hoặc token để xác thực

Cả hai giá trị cho hai key trên đều là chuỗi mã hóa base64. Bạn cũng có thể
cung cấp nội dung dạng văn bản thuần (clear text) bằng trường `stringData` trong
manifest của Secret.

Manifest sau là một ví dụ về Secret xác thực cơ bản:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: admin # trường bắt buộc đối với kubernetes.io/basic-auth
  password: t0p-Secret # trường bắt buộc đối với kubernetes.io/basic-auth
```

> **Ghi chú:**
>
> Trường `stringData` của Secret không hoạt động tốt với server-side apply.

Loại Secret xác thực cơ bản chỉ được cung cấp để thuận tiện.
Bạn hoàn toàn có thể tạo một Secret loại `Opaque` cho thông tin xác thực dùng trong xác thực cơ bản.
Tuy nhiên, dùng loại Secret công khai đã được định nghĩa sẵn (`kubernetes.io/basic-auth`) giúp người khác
hiểu được mục đích Secret của bạn, và thiết lập một quy ước về những tên key
cần có.

### Secret xác thực SSH (SSH authentication Secrets)

Loại built-in `kubernetes.io/ssh-auth` được cung cấp để lưu dữ liệu dùng trong
xác thực SSH. Khi dùng loại Secret này, bạn sẽ phải chỉ định một cặp key-value
`ssh-privatekey` trong trường `data` (hoặc `stringData`)
làm thông tin xác thực SSH cần sử dụng.

Manifest sau là ví dụ về một Secret dùng cho xác thực SSH bằng cặp khóa
công khai/riêng tư (public/private key):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-ssh-auth
type: kubernetes.io/ssh-auth
data:
  # dữ liệu được rút gọn trong ví dụ này
  ssh-privatekey: |
    UG91cmluZzYlRW1vdGljb24lU2N1YmE=
```

Loại Secret xác thực SSH chỉ được cung cấp để thuận tiện.
Bạn hoàn toàn có thể tạo một Secret loại `Opaque` cho thông tin xác thực dùng trong xác thực SSH.
Tuy nhiên, dùng loại Secret công khai đã được định nghĩa sẵn (`kubernetes.io/ssh-auth`) giúp người khác
hiểu được mục đích Secret của bạn, và thiết lập một quy ước về những tên key
cần có.
Kubernetes API sẽ xác minh rằng các key bắt buộc đã được thiết lập cho Secret thuộc loại này.

> **Thận trọng:**
>
> Bản thân khóa riêng SSH không thiết lập được kênh liên lạc tin cậy giữa SSH client và
> host server. Cần một phương tiện thứ hai để thiết lập niềm tin nhằm giảm thiểu
> tấn công "man in the middle", chẳng hạn như file `known_hosts` được thêm vào một ConfigMap.

### Secret TLS (TLS Secrets)

Loại Secret `kubernetes.io/tls` dùng để lưu một certificate cùng khóa
đi kèm của nó, thường được sử dụng cho TLS.

Một cách dùng phổ biến của TLS Secret là cấu hình mã hóa khi truyền (encryption in transit) cho
[Ingress](11-ingress-vi.md), nhưng bạn cũng có thể dùng nó
với các tài nguyên khác hoặc trực tiếp trong workload của mình.
Khi dùng loại Secret này, key `tls.key` và `tls.crt` phải được cung cấp
trong trường `data` (hoặc `stringData`) của cấu hình Secret, mặc dù API
server không thực sự kiểm tra tính hợp lệ của giá trị cho từng key.

Thay vì dùng `stringData`, bạn có thể dùng trường `data` để cung cấp
certificate và khóa riêng đã mã hóa base64. Xem chi tiết tại
[Ràng buộc về tên và dữ liệu của Secret](#restriction-names-data).

YAML sau là một ví dụ cấu hình cho TLS Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-tls
type: kubernetes.io/tls
data:
  # giá trị được mã hóa base64, chỉ che mờ chứ KHÔNG mang lại
  # bất kỳ mức độ bảo mật hữu ích nào
  # Hãy thay các giá trị sau bằng certificate và khóa mã hóa base64 của riêng bạn.
  tls.crt: "REPLACE_WITH_BASE64_CERT" 
  tls.key: "REPLACE_WITH_BASE64_KEY"
```

Loại TLS Secret chỉ được cung cấp để thuận tiện.
Bạn hoàn toàn có thể tạo một Secret loại `Opaque` cho thông tin xác thực dùng cho TLS.
Tuy nhiên, dùng loại Secret công khai đã được định nghĩa sẵn (`kubernetes.io/tls`)
giúp bảo đảm tính nhất quán của định dạng Secret trong dự án của bạn. API server
sẽ xác minh xem các key bắt buộc đã được thiết lập cho Secret thuộc loại này hay chưa.

Để tạo TLS Secret bằng `kubectl`, dùng subcommand `tls`:

```shell
kubectl create secret tls my-tls-secret \
  --cert=path/to/cert/file \
  --key=path/to/key/file
```

Cặp khóa công khai/riêng tư phải tồn tại từ trước. Certificate khóa công khai cho `--cert` phải được mã hóa .PEM
và phải khớp với khóa riêng được cung cấp cho `--key`.

### Secret bootstrap token (Bootstrap token Secrets) {#bootstrap-token-secrets}

Loại Secret `bootstrap.kubernetes.io/token` dành cho
các token được dùng trong quá trình bootstrap node. Nó lưu các token dùng để ký
những ConfigMap phổ biến (well-known ConfigMaps).

Secret bootstrap token thường được tạo trong namespace `kube-system` và
được đặt tên theo dạng `bootstrap-token-<token-id>` trong đó `<token-id>` là chuỗi
6 ký tự của token ID.

Dưới dạng manifest Kubernetes, một Secret bootstrap token có thể trông như sau:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-5emitj
  namespace: kube-system
type: bootstrap.kubernetes.io/token
data:
  auth-extra-groups: c3lzdGVtOmJvb3RzdHJhcHBlcnM6a3ViZWFkbTpkZWZhdWx0LW5vZGUtdG9rZW4=
  expiration: MjAyMC0wOS0xM1QwNDozOToxMFo=
  token-id: NWVtaXRq
  token-secret: a3E0Z2lodnN6emduMXAwcg==
  usage-bootstrap-authentication: dHJ1ZQ==
  usage-bootstrap-signing: dHJ1ZQ==
```

Secret bootstrap token có các key sau được chỉ định trong `data`:

- `token-id`: Một chuỗi 6 ký tự ngẫu nhiên làm định danh token. Bắt buộc.
- `token-secret`: Một chuỗi 16 ký tự ngẫu nhiên là nội dung token thực sự. Bắt buộc.
- `description`: Chuỗi mô tả (con người đọc được) cho biết token được dùng
  vào việc gì. Tùy chọn.
- `expiration`: Thời điểm tuyệt đối theo UTC ở định dạng [RFC3339](https://datatracker.ietf.org/doc/html/rfc3339) xác định khi nào token
  hết hạn. Tùy chọn.
- `usage-bootstrap-<usage>`: Cờ boolean cho biết công dụng bổ sung của
  bootstrap token.
- `auth-extra-groups`: Danh sách tên các group, phân tách bằng dấu phẩy, sẽ được
  xác thực với tư cách thành viên, bên cạnh group `system:bootstrappers`.

Bạn cũng có thể cung cấp các giá trị trong trường `stringData` của Secret
mà không cần mã hóa base64:

```yaml
apiVersion: v1
kind: Secret
metadata:
  # Chú ý cách Secret được đặt tên
  name: bootstrap-token-5emitj
  # Secret bootstrap token thường nằm trong namespace kube-system
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  auth-extra-groups: "system:bootstrappers:kubeadm:default-node-token"
  expiration: "2020-09-13T04:39:10Z"
  # Token ID này được dùng trong tên Secret
  token-id: "5emitj"
  token-secret: "kq4gihvszzgn1p0r"
  # Token này có thể được dùng để xác thực
  usage-bootstrap-authentication: "true"
  # và có thể được dùng để ký
  usage-bootstrap-signing: "true"
```

> **Ghi chú:**
>
> Trường `stringData` của Secret không hoạt động tốt với server-side apply.

## Làm việc với Secret (Working with Secrets)

### Tạo một Secret (Creating a Secret)

Có một số cách để tạo Secret:

- [Dùng `kubectl`](327-secret-kubectl-vi.md)
- [Dùng file cấu hình](326-secret-config-file-vi.md)
- [Dùng công cụ Kustomize](328-secret-kustomize-vi.md)

#### Ràng buộc về tên và dữ liệu của Secret (Constraints on Secret names and data) {#restriction-names-data}

Tên của một đối tượng Secret phải là một
[tên miền con DNS (DNS subdomain name)](17-names-vi.md#dns-subdomain-names) hợp lệ.

Bạn có thể chỉ định trường `data` và/hoặc trường `stringData` khi tạo
file cấu hình cho một Secret. Cả hai trường `data` và `stringData` đều là tùy chọn.
Giá trị của mọi key trong trường `data` phải là chuỗi mã hóa base64.
Nếu việc chuyển đổi sang chuỗi base64 không thuận tiện, bạn có thể chọn chỉ định
trường `stringData` thay thế, trường này chấp nhận chuỗi tùy ý làm giá trị.

Các key của `data` và `stringData` phải bao gồm các ký tự chữ-số,
`-`, `_` hoặc `.`. Mọi cặp key-value trong trường `stringData` được gộp
nội bộ vào trường `data`. Nếu một key xuất hiện trong cả hai trường `data` và
`stringData`, giá trị được chỉ định trong trường `stringData` sẽ được
ưu tiên.

#### Giới hạn kích thước (Size limit) {#restriction-data-size}

Mỗi Secret riêng lẻ bị giới hạn kích thước 1MiB. Điều này nhằm hạn chế việc tạo
các Secret rất lớn có thể làm cạn kiệt bộ nhớ của API server và kubelet.
Tuy nhiên, việc tạo quá nhiều Secret nhỏ cũng có thể làm cạn kiệt bộ nhớ. Bạn có thể
dùng [resource quota](134-resource-quotas-vi.md) để giới hạn
số lượng Secret (hoặc các tài nguyên khác) trong một namespace.

### Chỉnh sửa một Secret (Editing a Secret)

Bạn có thể chỉnh sửa một Secret đang tồn tại trừ khi nó là [bất biến (immutable)](#secret-immutable). Để
chỉnh sửa một Secret, dùng một trong các phương pháp sau:

- [Dùng `kubectl`](327-secret-kubectl-vi.md#edit-secret)
- [Dùng file cấu hình](326-secret-config-file-vi.md#edit-secret)

Bạn cũng có thể chỉnh sửa dữ liệu trong một Secret bằng [công cụ Kustomize](328-secret-kustomize-vi.md#edit-secret). Tuy nhiên, phương pháp
này sẽ tạo ra một đối tượng `Secret` mới với dữ liệu đã chỉnh sửa.

Tùy theo cách bạn tạo Secret, cũng như cách Secret được sử dụng trong
các Pod của bạn, các cập nhật lên đối tượng `Secret` hiện có sẽ được tự động lan truyền tới
những Pod đang dùng dữ liệu đó. Để biết thêm thông tin, tham khảo mục [Sử dụng Secret dưới dạng file trong Pod](#using-secrets-as-files-from-a-pod).

### Sử dụng một Secret (Using a Secret)

Secret có thể được mount làm data volume hoặc được phơi bày dưới dạng
biến môi trường (environment variables)
để container trong Pod sử dụng. Secret cũng có thể được dùng bởi các thành phần khác của
hệ thống mà không cần phơi bày trực tiếp cho Pod. Ví dụ, Secret có thể chứa
thông tin xác thực để các thành phần khác của hệ thống dùng khi tương tác với các
hệ thống bên ngoài thay mặt cho bạn.

Nguồn volume kiểu Secret (Secret volume source) được kiểm tra hợp lệ để bảo đảm tham chiếu
đối tượng được chỉ định thực sự trỏ đến một đối tượng loại Secret. Vì vậy, một Secret
cần được tạo trước bất kỳ Pod nào phụ thuộc vào nó.

Nếu không lấy được Secret (có thể vì nó không tồn tại, hoặc
do tạm thời mất kết nối tới API server), kubelet sẽ
định kỳ thử chạy lại Pod đó. Kubelet cũng báo cáo một Event
cho Pod đó, kèm chi tiết về sự cố khi lấy Secret.

#### Secret tùy chọn (Optional Secrets) {#restriction-secret-must-exist}

Khi tham chiếu một Secret trong Pod, bạn có thể đánh dấu Secret là _tùy chọn_ (optional),
như ví dụ sau. Nếu một Secret tùy chọn không tồn tại,
Kubernetes sẽ bỏ qua nó.

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
      optional: true
```

Theo mặc định, Secret là bắt buộc. Không container nào của Pod được khởi động cho đến khi
tất cả các Secret không-tùy-chọn sẵn sàng.

Nếu Pod tham chiếu một key cụ thể trong một Secret không-tùy-chọn, và Secret đó
tồn tại nhưng thiếu key được nêu tên, Pod sẽ thất bại trong quá trình khởi động.

### Sử dụng Secret dưới dạng file trong Pod (Using Secrets as files from a Pod) {#using-secrets-as-files-from-a-pod}

Nếu bạn muốn truy cập dữ liệu từ một Secret trong Pod, một cách là để
Kubernetes làm cho giá trị của Secret đó xuất hiện dưới dạng file bên trong
hệ thống file của một hoặc nhiều container trong Pod.

Để biết cách làm, tham khảo
[Tạo một Pod có quyền truy cập dữ liệu secret thông qua Volume](334-distribute-credentials-secure-vi.md#create-a-pod-that-has-access-to-the-secret-data-through-a-volume).

Khi một volume chứa dữ liệu từ một Secret, và Secret đó được cập nhật, Kubernetes sẽ theo dõi
điều này và cập nhật dữ liệu trong volume theo cách nhất quán cuối cùng (eventually-consistent).

> **Ghi chú:**
>
> Container sử dụng Secret qua mount volume kiểu
> [subPath](91-volumes-vi.md#using-subpath) sẽ không nhận được
> các cập nhật Secret tự động.

Kubelet giữ một cache chứa các key và giá trị hiện tại của những Secret được dùng trong
các volume của Pod trên node đó.
Bạn có thể cấu hình cách kubelet phát hiện thay đổi so với các giá trị đã cache. Trường
`configMapAndSecretChangeDetectionStrategy` trong
[cấu hình kubelet](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/) điều khiển
chiến lược mà kubelet sử dụng. Chiến lược mặc định là `Watch`.

Các cập nhật Secret có thể được lan truyền qua cơ chế watch của API (mặc định), dựa trên
cache có thời gian sống (time-to-live) xác định, hoặc được truy vấn (poll) từ API server của cluster
trong mỗi vòng lặp đồng bộ của kubelet.

Kết quả là, tổng độ trễ từ thời điểm Secret được cập nhật đến thời điểm
các key mới được chiếu (project) vào Pod có thể dài bằng chu kỳ đồng bộ của kubelet + độ trễ
lan truyền cache, trong đó độ trễ lan truyền cache phụ thuộc vào loại cache được chọn
(theo đúng thứ tự liệt kê ở đoạn trên, các độ trễ này là:
độ trễ lan truyền watch, TTL đã cấu hình của cache, hoặc bằng không đối với truy vấn trực tiếp).

### Sử dụng Secret làm biến môi trường (Using Secrets as environment variables) {#using-secrets-as-environment-variables}

Để dùng một Secret trong biến môi trường (environment variable)
của Pod:

1. Với mỗi container trong đặc tả Pod của bạn, thêm một biến môi trường
   cho từng key của Secret mà bạn muốn dùng vào trường
   `env[].valueFrom.secretKeyRef`.
2. Sửa image và/hoặc dòng lệnh sao cho chương trình đọc giá trị
   từ các biến môi trường đã chỉ định.

Để biết cách làm, tham khảo
[Định nghĩa biến môi trường của container bằng dữ liệu Secret](334-distribute-credentials-secure-vi.md#define-container-environment-variables-using-secret-data).

Điều quan trọng cần lưu ý là phạm vi ký tự được phép cho tên biến môi trường
trong Pod bị [giới hạn](331-define-environment-variable-vi.md#using-environment-variables-inside-of-your-config).
Nếu có key nào không đáp ứng quy tắc, các key đó sẽ không được cung cấp cho container của bạn,
mặc dù Pod vẫn được phép khởi động.

### Secret để kéo container image (Container image pull Secrets) {#using-imagepullsecrets}

Nếu bạn muốn lấy container image từ một repository riêng tư, bạn cần một cách để
kubelet trên mỗi node xác thực với repository đó. Bạn có thể cấu hình
_image pull Secrets_ để làm điều này. Các Secret này được cấu hình ở cấp
Pod.

#### Sử dụng imagePullSecrets (Using imagePullSecrets)

Trường `imagePullSecrets` là danh sách các tham chiếu tới những Secret trong cùng namespace.
Bạn có thể dùng `imagePullSecrets` để truyền cho kubelet một Secret chứa mật khẩu registry
image của Docker (hoặc registry khác). Kubelet dùng thông tin này để kéo image riêng tư thay mặt cho Pod của bạn.
Xem [PodSpec API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podspec-v1-core)
để biết thêm thông tin về trường `imagePullSecrets`.

##### Chỉ định imagePullSecret thủ công (Manually specifying an imagePullSecret)

Bạn có thể tìm hiểu cách chỉ định `imagePullSecrets` trong tài liệu về
[container image](40-images-vi.md#specifying-imagepullsecrets-on-a-pod).

##### Thiết lập để imagePullSecrets được tự động đính kèm (Arranging for imagePullSecrets to be automatically attached)

Bạn có thể tạo `imagePullSecrets` thủ công, rồi tham chiếu chúng từ một ServiceAccount. Bất kỳ Pod nào
được tạo với ServiceAccount đó, hoặc được tạo với ServiceAccount đó theo mặc định, sẽ có
trường `imagePullSecrets` được đặt thành giá trị của service account đó.
Xem [Thêm ImagePullSecrets vào service account](279-configure-service-account-vi.md#add-imagepullsecrets-to-a-service-account)
để có giải thích chi tiết về quy trình này.

### Sử dụng Secret với static Pod (Using Secrets with static Pods) {#restriction-static-pod}

Bạn không thể dùng ConfigMap hay Secret với static Pod.

## Secret bất biến (Immutable Secrets) {#secret-immutable}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.21 [stable]`

Kubernetes cho phép bạn đánh dấu một số Secret (và ConfigMap) cụ thể là _bất biến_ (immutable).
Việc ngăn thay đổi dữ liệu của một Secret đang tồn tại mang lại các lợi ích sau:

- bảo vệ bạn khỏi các cập nhật vô tình (hoặc không mong muốn) có thể gây gián đoạn ứng dụng
- (đối với các cluster sử dụng Secret ở quy mô lớn - ít nhất hàng chục nghìn lượt mount
  Secret vào Pod khác nhau), việc chuyển sang Secret bất biến cải thiện hiệu năng của cluster
  bằng cách giảm đáng kể tải lên kube-apiserver. Kubelet không cần duy trì
  watch trên bất kỳ Secret nào được đánh dấu là bất biến.

### Đánh dấu một Secret là bất biến (Marking a Secret as immutable) {#secret-immutable-create}

Bạn có thể tạo một Secret bất biến bằng cách đặt trường `immutable` thành `true`. Ví dụ,

```yaml
apiVersion: v1
kind: Secret
metadata: ...
data: ...
immutable: true
```

Bạn cũng có thể cập nhật bất kỳ Secret khả biến (mutable) nào đang tồn tại để biến nó thành bất biến.

> **Ghi chú:**
>
> Một khi Secret hay ConfigMap đã được đánh dấu là bất biến, thì _không_ thể hoàn tác thay đổi này,
> cũng không thể thay đổi nội dung của trường `data`. Bạn chỉ có thể xóa và tạo lại Secret.
> Các Pod đang tồn tại vẫn giữ mount point trỏ tới Secret đã xóa - khuyến nghị tạo lại
> các Pod này.

## An toàn thông tin cho Secret (Information security for Secrets) {#information-security-for-secrets}

Mặc dù ConfigMap và Secret hoạt động tương tự nhau, Kubernetes áp dụng thêm một số
biện pháp bảo vệ cho các đối tượng Secret.

Secret thường chứa các giá trị có mức độ quan trọng trải rộng trên nhiều cấp độ, nhiều
giá trị trong số đó có thể gây leo thang đặc quyền bên trong Kubernetes (ví dụ: service account token) và với
các hệ thống bên ngoài. Ngay cả khi một ứng dụng riêng lẻ có thể ước lượng được sức mạnh của những
Secret mà nó dự định tương tác, các ứng dụng khác trong cùng namespace vẫn có thể
làm cho những giả định đó trở nên vô hiệu.

Cấu hình phân quyền (authorization) ảnh hưởng đến cách dữ liệu Secret có thể được truy cập trong một namespace.
Ví dụ, việc cấp quyền **list** hoặc **watch** trên Secret cho phép một chủ thể (subject)
đọc toàn bộ dữ liệu Secret trong namespace đó, chứ không chỉ những Secret được các Pod của nó
tham chiếu rõ ràng. Hãy giới hạn quyền truy cập ở mức tối thiểu cần thiết
để workload hoạt động, và tránh cấp các vai trò (role) rộng như
`cluster-admin` trừ khi thực sự cần cho mục đích quản trị.

Xem thêm [tài liệu về phân quyền](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

Một Secret chỉ được gửi tới một node nếu có Pod trên node đó cần nó.
Khi mount Secret vào Pod, kubelet lưu bản sao dữ liệu vào `tmpfs`
để dữ liệu bí mật không bị ghi xuống bộ nhớ lưu trữ bền vững.
Khi Pod phụ thuộc vào Secret bị xóa, kubelet sẽ xóa bản sao cục bộ
của dữ liệu bí mật từ Secret đó.

Một Pod có thể có nhiều container. Theo mặc định, các container bạn định nghĩa
chỉ có quyền truy cập vào ServiceAccount mặc định và Secret liên quan của nó.
Bạn phải định nghĩa biến môi trường hoặc ánh xạ (map) một volume vào
container một cách tường minh để cấp quyền truy cập tới bất kỳ Secret nào khác.

Có thể có Secret cho nhiều Pod trên cùng một node. Tuy nhiên, chỉ những
Secret mà một Pod yêu cầu mới có khả năng hiển thị bên trong các container của Pod đó.
Do đó, một Pod không có quyền truy cập vào Secret của Pod khác.

### Cấu hình quyền truy cập tối thiểu tới Secret (Configure least-privilege access to Secrets)

Để tăng cường các biện pháp bảo mật xung quanh Secret, hãy dùng các namespace riêng biệt
để cô lập quyền truy cập tới những secret được mount.

> **Cảnh báo:**
>
> Bất kỳ container nào chạy với `privileged: true` trên một node đều có thể truy cập mọi
> Secret được sử dụng trên node đó.

## Tiếp theo (What's next)

- Để có hướng dẫn về quản lý và nâng cao tính bảo mật cho Secret của bạn, tham khảo
  [Các thực hành tốt cho Kubernetes Secret](121-secrets-good-practices-vi.md).
- Tìm hiểu cách [quản lý Secret bằng `kubectl`](327-secret-kubectl-vi.md)
- Tìm hiểu cách [quản lý Secret bằng file cấu hình](326-secret-config-file-vi.md)
- Tìm hiểu cách [quản lý Secret bằng kustomize](328-secret-kustomize-vi.md)
- Đọc [tài liệu tham chiếu API](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/secret-v1/) cho `Secret`

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Bạn chạy `kubectl get secret db-cred -o yaml` và thấy `password: dDBwLVNlY3JldA==`. Giá
   trị đó đã được mã hóa chưa? Người có quyền đọc etcd của `k8s-master` thấy được gì?
2. Bạn muốn một nhóm chỉ đọc được đúng một Secret trong namespace của họ, nên chỉ cấp quyền
   `get` trên đúng Secret đó — nhưng vẫn cho họ quyền tạo Deployment trong namespace. Ranh
   giới đó có giữ được không?
3. Control plane trên `k8s-master` chạy bằng các file manifest trong `/etc/kubernetes/manifests/`
   mà bạn đã xem ở [Lab 1a](labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md#b2-kiểm-kê-component).
   Bạn có thể cho một Pod loại đó đọc mật khẩu từ một Secret không?
4. `data` và `stringData` khác nhau ở chỗ nào? Nếu cùng một key xuất hiện ở cả hai trường thì
   giá trị nào thắng?
5. Một Pod trên `k8s-worker1` mount Secret `db-cred`. Dữ liệu Secret đó nằm ở đâu trên node,
   và một Pod khác cũng đang chạy trên `k8s-worker1` có đọc được nó không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Chưa mã hóa gì cả.** `dDBwLVNlY3JldA==` chỉ là base64 của một chuỗi thường; bất kỳ ai
   cũng giải ngược được bằng `base64 -d`. Bài nói rõ hai lần: Secret "được lưu trữ **không mã
   hóa** trong kho dữ liệu nền của API server (etcd)", và giá trị base64 "bị che mờ nhưng
   không hề bí mật". Vì vậy **bất kỳ ai có quyền truy cập etcd đều đọc được mật khẩu thật**,
   y như người có quyền truy cập API. Muốn dữ liệu thật sự được mã hóa trong etcd thì phải
   bật *Encryption at Rest* — thao tác thuộc giai đoạn 22.
2. **Không giữ được.** Bài nêu thẳng trong khối *Thận trọng* đầu bài: "bất kỳ ai được phép tạo
   Pod trong một namespace đều có thể lợi dụng quyền đó để đọc **bất kỳ Secret nào** trong
   namespace đó; điều này bao gồm cả quyền truy cập gián tiếp như khả năng tạo một Deployment".
   Chỉ cần tạo một Pod mount Secret rồi in ra là xong. Muốn cô lập thật thì phải **tách
   namespace** — đúng như mục *Cấu hình quyền truy cập tối thiểu tới Secret* khuyến nghị.
   Cùng lý do đó, cấp `list` hoặc `watch` trên Secret cũng đồng nghĩa cho đọc toàn bộ Secret
   của namespace.
3. **Không.** Mục *Sử dụng Secret với static Pod* nói một câu duy nhất và dứt khoát: "Bạn
   không thể dùng ConfigMap hay Secret với static Pod." Các Pod đó do kubelet quản lý trực
   tiếp từ file trên đĩa, nên spec của chúng không tham chiếu được đối tượng API nào. Bài
   [58](58-static-pods-vi.md) sẽ giải thích vì sao.
4. **`data` bắt buộc là chuỗi đã mã hóa base64; `stringData` nhận chuỗi tùy ý** để bạn khỏi
   phải tự base64. Mọi cặp key-value trong `stringData` được gộp nội bộ vào `data`, và nếu
   một key có ở cả hai thì **giá trị trong `stringData` được ưu tiên**. Lưu ý bài nhắc hai
   lần: `stringData` không hoạt động tốt với server-side apply.
5. Kubelet lưu bản sao dữ liệu vào **`tmpfs`** — tức trong bộ nhớ, cố ý để dữ liệu bí mật
   không bị ghi xuống lưu trữ bền vững — và xóa bản sao đó khi Pod phụ thuộc bị xóa. Pod khác
   trên cùng node **không đọc được**: "chỉ những Secret mà một Pod yêu cầu mới có khả năng
   hiển thị bên trong các container của Pod đó… một Pod không có quyền truy cập vào Secret của
   Pod khác". Ngoại lệ duy nhất bài nêu là container chạy với `privileged: true` — nó truy cập
   được mọi Secret đang dùng trên node đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
