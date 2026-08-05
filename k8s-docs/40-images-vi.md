# Các Image (Images)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/containers/images/>

Một container image đại diện cho dữ liệu nhị phân đóng gói một ứng dụng và toàn bộ các
dependency phần mềm của nó. Container image là các gói phần mềm có thể thực thi, chạy
độc lập và đưa ra những giả định được xác định rất rõ ràng về môi trường runtime của
chúng.

Thông thường, bạn tạo một container image cho ứng dụng của mình và đẩy (push) nó lên
một registry trước khi tham chiếu tới nó trong một Pod.

Trang này cung cấp cái nhìn tổng quan về khái niệm container image.

> **Ghi chú:**
> Nếu bạn đang tìm các container image cho một bản phát hành Kubernetes
> (chẳng hạn v1.36, bản phát hành minor mới nhất),
> hãy truy cập [Download Kubernetes](https://kubernetes.io/releases/download/).

## Tên image (Image names)

Container image thường được đặt tên như `pause`, `example/mycontainer`, hoặc `kube-apiserver`.
Image cũng có thể bao gồm hostname của registry; ví dụ: `fictional.registry.example/imagename`,
và có thể kèm cả số port; ví dụ: `fictional.registry.example:10443/imagename`.

Nếu bạn không chỉ định hostname của registry, Kubernetes hiểu rằng bạn muốn dùng
[Docker public registry](https://hub.docker.com/).
Bạn có thể thay đổi hành vi này bằng cách đặt image registry mặc định trong cấu hình
[container runtime](./00-container-runtimes-vi.md).

Sau phần tên image, bạn có thể thêm _tag_ hoặc _digest_ (theo cùng cách bạn làm khi
dùng các lệnh như `docker` hoặc `podman`). Tag cho phép bạn phân biệt các phiên bản
khác nhau của cùng một dòng image. Digest là định danh duy nhất cho một phiên bản cụ thể
của image. Digest là giá trị băm (hash) của nội dung image và là bất biến (immutable).
Tag có thể được chuyển sang trỏ tới image khác, nhưng digest là cố định.

Tag của image bao gồm chữ cái thường và chữ cái hoa, chữ số, dấu gạch dưới (`_`),
dấu chấm (`.`) và dấu gạch ngang (`-`). Một tag có thể dài tối đa 128 ký tự và phải
tuân theo mẫu regex sau: `[a-zA-Z0-9_][a-zA-Z0-9._-]{0,127}`.
Bạn có thể đọc thêm và tìm regex kiểm tra hợp lệ (validation) trong
[OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/master/spec.md#workflow-categories).
Nếu bạn không chỉ định tag, Kubernetes hiểu rằng bạn muốn dùng tag `latest`.

Digest của image bao gồm một thuật toán băm (chẳng hạn `sha256`) và một giá trị băm. Ví dụ:
`sha256:1ff6c18fbef2045af6b9c16bf034cc421a29027b800e4f9b68ae9b1cb3e9ae07`.
Bạn có thể tìm thêm thông tin về định dạng digest trong
[OCI Image Specification](https://github.com/opencontainers/image-spec/blob/master/descriptor.md#digests).

Một số ví dụ tên image mà Kubernetes có thể sử dụng:

- `busybox` &mdash; Chỉ có tên image, không có tag hay digest. Kubernetes sẽ dùng
    Docker public registry và tag latest. Tương đương với `docker.io/library/busybox:latest`.
- `busybox:1.32.0` &mdash; Tên image kèm tag. Kubernetes sẽ dùng
    Docker public registry. Tương đương với `docker.io/library/busybox:1.32.0`.
- `registry.k8s.io/pause:latest` &mdash; Tên image với registry tùy chỉnh và tag latest.
- `registry.k8s.io/pause:3.5` &mdash; Tên image với registry tùy chỉnh và tag không phải latest.
- `registry.k8s.io/pause@sha256:1ff6c18fbef2045af6b9c16bf034cc421a29027b800e4f9b68ae9b1cb3e9ae07` &mdash;
    Tên image kèm digest.
- `registry.k8s.io/pause:3.5@sha256:1ff6c18fbef2045af6b9c16bf034cc421a29027b800e4f9b68ae9b1cb3e9ae07` &mdash;
    Tên image kèm tag và digest. Chỉ digest được dùng khi pull.

## Cập nhật image (Updating images)

Khi bạn tạo lần đầu một Deployment, StatefulSet, Pod, hoặc đối tượng khác có chứa
PodTemplate, và pull policy không được chỉ định tường minh, thì mặc định pull policy
của tất cả các container trong Pod đó sẽ được đặt là `IfNotPresent`. Policy này khiến
kubelet bỏ qua việc pull một image nếu image đó đã tồn tại.

### Chính sách pull image (Image pull policy)

`imagePullPolicy` của một container và tag của image cùng ảnh hưởng tới _thời điểm_
[kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
cố gắng pull (tải về) image được chỉ định.

Dưới đây là danh sách các giá trị bạn có thể đặt cho `imagePullPolicy` và tác dụng
của các giá trị này:

`IfNotPresent`
: image chỉ được pull nếu nó chưa có sẵn ở máy cục bộ (local).

`Always`
: mỗi lần kubelet khởi chạy một container, kubelet yêu cầu container runtime
  pull image. Container runtime liên hệ với registry, phân giải tag hoặc tên image
  thành một
  [digest](https://docs.docker.com/engine/reference/commandline/pull/#pull-an-image-by-digest-immutable-identifier),
  và tải về mọi layer chưa được cache cục bộ.
  Nếu tất cả các layer đã có sẵn, container runtime dùng image đã cache mà không
  tải lại. Bản thân kubelet không kiểm tra image đã được cache cục bộ hay chưa;
  nó luôn ủy quyền cho container runtime.

`Never`
: kubelet không cố lấy image. Nếu image bằng cách nào đó đã có sẵn cục bộ,
  kubelet sẽ cố khởi động container; nếu không, việc khởi động thất bại.
  Xem [image được pull sẵn](#pre-pulled-images) để biết thêm chi tiết.

Ngữ nghĩa cache của container runtime khiến ngay cả `imagePullPolicy: Always`
cũng hiệu quả, miễn là registry có thể truy cập một cách ổn định.
Container runtime có thể nhận ra rằng các layer của image đã tồn tại trên node
nên chúng không cần được tải về lần nữa.

> **Ghi chú:**
> Bạn nên tránh dùng tag `:latest` khi triển khai container trong môi trường production
> vì sẽ khó theo dõi phiên bản nào của image đang chạy và khó rollback đúng cách hơn.
>
> Thay vào đó, hãy chỉ định một tag có ý nghĩa như `v1.42.0` và/hoặc một digest.

Để bảo đảm Pod luôn dùng cùng một phiên bản của container image, bạn có thể chỉ định
digest của image;
thay `<image-name>:<tag>` bằng `<image-name>@<digest>`
(ví dụ, `image@sha256:45b23dee08af5e43a7fea6c4cf9c25ccf269ee113168c19722f87876677c5cb2`).

Khi dùng tag image, nếu registry thay đổi phần mã mà tag đó đại diện, bạn có thể rơi
vào tình trạng lẫn lộn các Pod chạy mã cũ và mã mới. Digest của image định danh duy nhất
một phiên bản cụ thể của image, vì vậy Kubernetes chạy cùng một mã mỗi lần nó khởi động
một container với tên image và digest đã chỉ định. Chỉ định image bằng digest sẽ ghim
(pin) phần mã bạn chạy, để một thay đổi tại registry không thể dẫn tới tình trạng
lẫn lộn phiên bản đó.

Có các [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
bên thứ ba thực hiện biến đổi (mutate) Pod (và PodTemplate) khi chúng được tạo, để
workload đang chạy được định nghĩa dựa trên digest của image thay vì tag.
Điều này có thể hữu ích nếu bạn muốn bảo đảm toàn bộ workload của mình đang chạy
cùng một mã bất kể tag thay đổi thế nào ở registry.

#### Chính sách pull image mặc định (Default image pull policy) {#imagepullpolicy-defaulting}

Khi bạn (hoặc một controller) gửi một Pod mới lên API server, cluster của bạn đặt
trường `imagePullPolicy` khi các điều kiện cụ thể được thỏa mãn:

- nếu bạn bỏ trống trường `imagePullPolicy` và có chỉ định digest cho container image,
  `imagePullPolicy` được tự động đặt thành `IfNotPresent`.
- nếu bạn bỏ trống trường `imagePullPolicy` và tag của container image là
  `:latest`, `imagePullPolicy` được tự động đặt thành `Always`.
- nếu bạn bỏ trống trường `imagePullPolicy` và không chỉ định tag cho container image,
  `imagePullPolicy` được tự động đặt thành `Always`.
- nếu bạn bỏ trống trường `imagePullPolicy` và chỉ định một tag cho container image
  mà không phải là `:latest`, `imagePullPolicy` được tự động đặt thành `IfNotPresent`.

> **Ghi chú:**
> Giá trị `imagePullPolicy` của container luôn được đặt khi đối tượng được _tạo_ lần
> đầu, và không được cập nhật nếu tag hoặc digest của image thay đổi sau đó.
>
> Ví dụ, nếu bạn tạo một Deployment với image có tag _không phải_ là
> `:latest`, và sau đó cập nhật image của Deployment đó sang tag `:latest`, trường
> `imagePullPolicy` sẽ _không_ đổi thành `Always`. Bạn phải tự tay thay đổi
> pull policy của bất kỳ đối tượng nào sau khi nó đã được tạo lần đầu.

#### Bắt buộc pull image (Required image pull)

Nếu bạn muốn luôn luôn ép buộc việc pull, bạn có thể làm một trong các cách sau:

- Đặt `imagePullPolicy` của container thành `Always`.
- Bỏ trống `imagePullPolicy` và dùng `:latest` làm tag cho image cần dùng;
  Kubernetes sẽ đặt policy thành `Always` khi bạn gửi Pod.
- Bỏ trống `imagePullPolicy` và tag của image cần dùng;
  Kubernetes sẽ đặt policy thành `Always` khi bạn gửi Pod.
- Bật admission controller
  [AlwaysPullImages](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#alwayspullimages).

### ImagePullBackOff

Khi kubelet bắt đầu tạo container cho một Pod bằng một container runtime,
có khả năng container ở trạng thái
[Waiting](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-state-waiting)
vì `ImagePullBackOff`.

Trạng thái `ImagePullBackOff` nghĩa là một container không thể khởi động vì Kubernetes
không thể pull container image (vì các lý do như tên image không hợp lệ, hoặc pull từ
một private registry mà không có `imagePullSecret`). Phần `BackOff` cho biết Kubernetes
sẽ tiếp tục cố pull image, với độ trễ chờ (back-off delay) tăng dần.

Kubernetes tăng độ trễ giữa mỗi lần thử cho tới khi đạt giới hạn được biên dịch sẵn,
là 300 giây (5 phút).

### Pull image theo runtime class (Image pull per runtime class)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.29 [alpha]`

Kubernetes có hỗ trợ alpha cho việc thực hiện pull image dựa trên RuntimeClass của Pod.

Nếu bạn bật [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`RuntimeClassInImageCriApi`, kubelet tham chiếu container image bằng một bộ (tuple)
gồm tên image và runtime handler thay vì chỉ tên image hoặc digest. Container runtime
của bạn có thể điều chỉnh hành vi dựa trên runtime handler được chọn.
Pull image dựa trên runtime class hữu ích cho các container chạy trên nền máy ảo (VM),
chẳng hạn các container Windows Hyper-V.

## Pull image tuần tự và song song (Serial and parallel image pulls)

Mặc định, kubelet pull image một cách tuần tự. Nói cách khác, kubelet chỉ gửi
một yêu cầu pull image tới image service tại một thời điểm. Các yêu cầu pull image
khác phải chờ cho tới khi yêu cầu đang được xử lý hoàn tất.

Các node đưa ra quyết định pull image một cách độc lập. Ngay cả khi bạn dùng pull
image tuần tự, hai node khác nhau vẫn có thể pull cùng một image song song.

Nếu bạn muốn bật pull image song song, bạn có thể đặt trường `serializeImagePulls`
thành false trong [cấu hình kubelet](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).
Với `serializeImagePulls` đặt thành false, các yêu cầu pull image sẽ được gửi tới
image service ngay lập tức, và nhiều image sẽ được pull cùng lúc.

Khi bật pull image song song, hãy bảo đảm image service của container runtime
của bạn có thể xử lý được việc pull image song song.

Kubelet không bao giờ pull nhiều image song song cho cùng một Pod. Ví dụ,
nếu bạn có một Pod có một init container và một container ứng dụng, việc pull image
cho hai container này sẽ không được thực hiện song song. Tuy nhiên, nếu bạn có hai
Pod dùng các image khác nhau, và tính năng pull image song song được bật,
kubelet sẽ pull các image song song cho hai Pod khác nhau đó.

### Số lượng pull image song song tối đa (Maximum parallel image pulls)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`

Khi `serializeImagePulls` được đặt thành false, kubelet mặc định không giới hạn
số lượng image tối đa được pull cùng lúc. Nếu bạn muốn giới hạn số lượng pull image
song song, bạn có thể đặt trường `maxParallelImagePulls` trong cấu hình kubelet.
Với `maxParallelImagePulls` đặt thành _n_, chỉ _n_ image có thể được pull cùng lúc,
và bất kỳ việc pull image nào vượt quá _n_ sẽ phải chờ cho tới khi ít nhất một
việc pull image đang diễn ra hoàn tất.

Việc giới hạn số lượng pull image song song giúp việc pull image không tiêu tốn
quá nhiều băng thông mạng hoặc I/O đĩa, khi tính năng pull image song song được bật.

Bạn có thể đặt `maxParallelImagePulls` là một số dương lớn hơn hoặc bằng 1.
Nếu bạn đặt `maxParallelImagePulls` lớn hơn hoặc bằng 2, bạn phải đặt
`serializeImagePulls` thành false. Kubelet sẽ không khởi động được với thiết lập
`maxParallelImagePulls` không hợp lệ.

## Image đa kiến trúc với image index (Multi-architecture images with image indexes)

Ngoài việc cung cấp các image nhị phân, một container registry cũng có thể phục vụ
[container image index](https://github.com/opencontainers/image-spec/blob/master/image-index.md).
Một image index có thể trỏ tới nhiều [image manifest](https://github.com/opencontainers/image-spec/blob/master/manifest.md)
cho các phiên bản của container theo từng kiến trúc (architecture) cụ thể. Ý tưởng là
bạn có thể có một cái tên cho image (ví dụ: `pause`, `example/mycontainer`, `kube-apiserver`)
và cho phép các hệ thống khác nhau lấy đúng image nhị phân cho kiến trúc máy mà
chúng đang dùng.

Dự án Kubernetes thường tạo các container image cho các bản phát hành của mình với
tên bao gồm hậu tố `-$(ARCH)`. Để tương thích ngược, hãy tạo các image cũ hơn với
hậu tố. Chẳng hạn, một image tên `pause` sẽ là image đa kiến trúc chứa manifest cho
tất cả các kiến trúc được hỗ trợ, trong khi `pause-amd64` sẽ là phiên bản tương thích
ngược cho các cấu hình cũ hơn, hoặc cho các file YAML có tên image kèm hậu tố được
ghi cứng (hardcoded).

## Sử dụng private registry (Using a private registry)

Private registry có thể yêu cầu xác thực (authentication) để có thể khám phá
và/hoặc pull image từ đó.
Thông tin xác thực (credentials) có thể được cung cấp theo nhiều cách:

- [Chỉ định `imagePullSecrets` khi bạn định nghĩa một Pod](#specifying-imagepullsecrets-on-a-pod)

  Chỉ những Pod cung cấp khóa (key) của riêng mình mới truy cập được private registry.

- [Cấu hình các Node để xác thực với private registry](#configuring-nodes-to-authenticate-to-a-private-registry)
  - Tất cả các Pod đều có thể đọc mọi private registry đã được cấu hình.
  - Yêu cầu quản trị viên cluster cấu hình node.
- Dùng plugin _kubelet credential provider_ để [lấy động thông tin xác thực cho private registry](#kubelet-credential-provider)

  Kubelet có thể được cấu hình để dùng plugin exec credential provider cho
  private registry tương ứng.

- [Image được pull sẵn](#pre-pulled-images)
  - Tất cả các Pod đều có thể dùng mọi image đã được cache trên node.
  - Yêu cầu quyền root trên tất cả các node để thiết lập.
- Các phần mở rộng cục bộ hoặc riêng của nhà cung cấp (vendor-specific)

  Nếu bạn dùng cấu hình node tùy chỉnh, bạn (hoặc nhà cung cấp cloud của bạn) có thể
  tự hiện thực cơ chế xác thực node với container registry.

Các lựa chọn này được giải thích chi tiết hơn bên dưới.

### Chỉ định `imagePullSecrets` trên một Pod (Specifying `imagePullSecrets` on a Pod) {#specifying-imagepullsecrets-on-a-pod}

> **Ghi chú:**
> Đây là cách tiếp cận được khuyến nghị để chạy các container dựa trên image
> trong private registry.

Kubernetes hỗ trợ chỉ định các khóa của container image registry trên một Pod.
Tất cả `imagePullSecrets` phải là các Secret tồn tại trong cùng namespace với
Pod. Các Secret này phải có kiểu `kubernetes.io/dockercfg` hoặc `kubernetes.io/dockerconfigjson`.

### Cấu hình node để xác thực với private registry (Configuring nodes to authenticate to a private registry) {#configuring-nodes-to-authenticate-to-a-private-registry}

Hướng dẫn cụ thể để thiết lập thông tin xác thực phụ thuộc vào container runtime và
registry mà bạn chọn dùng. Bạn nên tham khảo tài liệu của giải pháp mình dùng để có
thông tin chính xác nhất.

Để xem ví dụ về cấu hình một private container image registry, hãy xem task
[Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry).
Ví dụ đó dùng một private registry trên Docker Hub.

### Kubelet credential provider cho việc pull image có xác thực (Kubelet credential provider for authenticated image pulls) {#kubelet-credential-provider}

Bạn có thể cấu hình kubelet để gọi một plugin nhị phân nhằm lấy động thông tin xác thực
registry cho một container image. Đây là cách mạnh mẽ và linh hoạt nhất để lấy
thông tin xác thực cho private registry, nhưng cũng đòi hỏi cấu hình ở cấp kubelet
để kích hoạt.

Kỹ thuật này có thể đặc biệt hữu ích khi chạy các static Pod cần các container image
được lưu trữ trong một private registry.
Việc dùng một ServiceAccount hoặc một Secret để cung cấp thông tin xác thực cho
private registry là không thể trong đặc tả (specification) của một static Pod,
vì nó _không thể_ chứa tham chiếu tới các tài nguyên API khác trong đặc tả của mình.

Xem [Configure a kubelet image credential provider](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-credential-provider/)
để biết thêm chi tiết.

### Cách diễn giải config.json (Interpretation of config.json) {#config-json}

Cách diễn giải `config.json` khác nhau giữa hiện thực gốc của Docker và cách
Kubernetes diễn giải. Trong Docker, các khóa `auths` chỉ có thể chỉ định URL gốc
(root URL), trong khi Kubernetes cho phép cả URL dạng glob cũng như các đường dẫn
khớp theo tiền tố (prefix-matched). Hạn chế duy nhất là các mẫu glob (`*`) phải
bao gồm dấu chấm (`.`) cho mỗi subdomain. Số lượng subdomain được khớp phải bằng
số lượng mẫu glob (`*.`), ví dụ:

- `*.kubernetes.io` sẽ *không* khớp `kubernetes.io`, nhưng sẽ khớp
    `abc.kubernetes.io`.
- `*.*.kubernetes.io` sẽ *không* khớp `abc.kubernetes.io`, nhưng sẽ khớp
    `abc.def.kubernetes.io`.
- `prefix.*.io` sẽ khớp `prefix.kubernetes.io`.
- `*-good.kubernetes.io` sẽ khớp `prefix-good.kubernetes.io`.

Điều này nghĩa là một `config.json` như sau là hợp lệ:

```json
{
    "auths": {
        "my-registry.example/images": { "auth": "…" },
        "*.my-registry.example/images": { "auth": "…" }
    }
}
```

Các thao tác pull image truyền thông tin xác thực tới CRI container runtime cho mọi
mẫu (pattern) hợp lệ. Ví dụ, các tên container image sau sẽ khớp thành công:

- `my-registry.example/images`
- `my-registry.example/images/my-image`
- `my-registry.example/images/another-image`
- `sub.my-registry.example/images/my-image`

Tuy nhiên, các tên container image này sẽ *không* khớp:

- `a.sub.my-registry.example/images/my-image`
- `a.b.sub.my-registry.example/images/my-image`

Kubelet thực hiện pull image một cách tuần tự với từng thông tin xác thực tìm được.
Điều này nghĩa là cũng có thể có nhiều mục (entry) trong `config.json` cho các
đường dẫn khác nhau:

```json
{
    "auths": {
        "my-registry.example/images": {
            "auth": "…"
        },
        "my-registry.example/images/subpath": {
            "auth": "…"
        }
    }
}
```

Nếu bây giờ một container chỉ định cần pull image
`my-registry.example/images/subpath/my-image`, thì kubelet sẽ thử tải nó bằng cả hai
nguồn xác thực nếu một trong hai bị thất bại.

### Image được pull sẵn (Pre-pulled images) {#pre-pulled-images}

> **Ghi chú:**
> Cách tiếp cận này phù hợp nếu bạn có thể kiểm soát cấu hình node. Nó sẽ
> không hoạt động ổn định nếu nhà cung cấp cloud của bạn quản lý các node và
> tự động thay thế chúng.

Mặc định, kubelet cố pull mỗi image từ registry được chỉ định.
Tuy nhiên, nếu thuộc tính `imagePullPolicy` của container được đặt là `IfNotPresent`
hoặc `Never`, thì image cục bộ sẽ được dùng (tương ứng là ưu tiên dùng hoặc chỉ dùng
image cục bộ).

Nếu bạn muốn dựa vào các image được pull sẵn thay cho việc xác thực với registry,
bạn phải bảo đảm tất cả các node trong cluster có cùng các image được pull sẵn.

Cách này có thể được dùng để nạp trước (preload) một số image nhất định nhằm tăng tốc,
hoặc như một giải pháp thay thế cho việc xác thực với private registry.

Tương tự việc dùng [kubelet credential provider](#kubelet-credential-provider),
các image được pull sẵn cũng phù hợp để khởi chạy các static Pod phụ thuộc vào
image được lưu trữ trong một private registry.

> **Ghi chú:**
> **TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [beta]`
>
> Quyền truy cập các image được pull sẵn có thể được cấp phép theo
> [kiểm tra thông tin xác thực khi pull image](#ensureimagepullcredentialverification).

### Bảo đảm kiểm tra thông tin xác thực khi pull image (Ensure image pull credential verification) {#ensureimagepullcredentialverification}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [beta]`

Nếu feature gate `KubeletEnsureSecretPulledImages` được bật cho cluster của bạn,
Kubernetes sẽ kiểm tra tính hợp lệ của thông tin xác thực image cho mọi image cần
thông tin xác thực để pull, ngay cả khi image đó đã có mặt trên node. Việc kiểm tra
này bảo đảm rằng những image trong một yêu cầu Pod chưa từng được pull thành công
với thông tin xác thực được cung cấp sẽ phải pull lại image từ registry.
Ngoài ra, các lần pull image dùng lại cùng thông tin xác thực từng cho kết quả pull
image thành công trước đó sẽ không cần pull lại từ registry mà thay vào đó được kiểm
tra cục bộ mà không cần truy cập registry (với điều kiện image có sẵn cục bộ).
Điều này được điều khiển bởi trường `imagePullCredentialsVerificationPolicy` trong
[cấu hình Kubelet](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/#kubelet-config-k8s-io-v1beta1-ImagePullCredentialsVerificationPolicy).

Cấu hình này điều khiển thời điểm thông tin xác thực khi pull image phải được kiểm tra
nếu image đã có mặt trên node:

 * `NeverVerify`: Mô phỏng hành vi khi feature gate này bị tắt.
   Nếu image đã có cục bộ, thông tin xác thực khi pull image không được kiểm tra.
 * `NeverVerifyPreloadedImages`: Các image được pull ngoài kubelet không bị kiểm tra,
 nhưng tất cả các image khác sẽ được kiểm tra thông tin xác thực. Đây là hành vi mặc định.
 * `NeverVerifyAllowListedImages`: Các image được pull ngoài kubelet và được liệt kê trong
   `preloadedImagesVerificationAllowlist` chỉ định trong cấu hình kubelet sẽ không bị kiểm tra.
 * `AlwaysVerify`: Tất cả các image sẽ được kiểm tra thông tin xác thực
   trước khi có thể được sử dụng.

Việc kiểm tra này áp dụng cho [image được pull sẵn](#pre-pulled-images),
image được pull bằng secret cấp node (node-wide), và image được pull bằng secret
cấp Pod.

> **Ghi chú:**
> Trong trường hợp xoay vòng thông tin xác thực (credential rotation), thông tin xác
> thực từng được dùng trước đó để pull image sẽ tiếp tục được xác minh mà không cần
> truy cập registry. Thông tin xác thực mới hoặc đã được xoay vòng sẽ yêu cầu image
> phải được pull lại từ registry.

#### Bật `KubeletEnsureSecretPulledImages` lần đầu tiên (Enabling `KubeletEnsureSecretPulledImages` for the first time)

Khi `KubeletEnsureSecretPulledImages` được bật lần đầu tiên, hoặc bằng cách nâng cấp
kubelet hoặc bằng cách bật tính năng một cách tường minh, nếu kubelet có thể truy cập
bất kỳ image nào tại thời điểm đó, tất cả những image này sẽ được coi là đã pull sẵn.
Điều này xảy ra vì trong trường hợp này kubelet không có bản ghi (record) nào về việc
các image đã được pull. Kubelet chỉ có thể bắt đầu tạo bản ghi pull image khi có
image được pull lần đầu tiên.

Nếu đây là mối lo ngại, bạn nên dọn sạch khỏi các node tất cả những image không nên
được coi là pull sẵn trước khi bật tính năng.

Lưu ý rằng việc xóa thư mục chứa các bản ghi pull image sẽ có cùng tác dụng khi
kubelet khởi động lại, cụ thể là các image hiện đang được cache trên các node bởi
container runtime sẽ đều được coi là pull sẵn.

### Tạo Secret với Docker config (Creating a Secret with a Docker config)

Bạn cần biết username, mật khẩu registry và địa chỉ email của client để xác thực
với registry, cũng như hostname của nó.
Chạy lệnh sau, thay các placeholder bằng giá trị phù hợp:

```shell
kubectl create secret docker-registry <name> \
  --docker-server=<docker-registry-server> \
  --docker-username=<docker-user> \
  --docker-password=<docker-password> \
  --docker-email=<docker-email>
```

Nếu bạn đã có sẵn một file thông tin xác thực Docker thì, thay vì dùng lệnh trên,
bạn có thể nhập (import) file thông tin xác thực đó thành một Secret của Kubernetes.
[Create a Secret based on existing Docker credentials](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#registry-secret-existing-credentials)
giải thích cách thiết lập việc này.

Cách này đặc biệt hữu ích nếu bạn dùng nhiều private container registry,
vì `kubectl create secret docker-registry` tạo ra một Secret chỉ hoạt động với
một private registry duy nhất.

> **Ghi chú:**
> Pod chỉ có thể tham chiếu các image pull secret trong namespace của chính nó,
> vì vậy quy trình này cần được thực hiện một lần cho mỗi namespace.

#### Tham chiếu `imagePullSecrets` trên một Pod (Referring to `imagePullSecrets` on a Pod)

Bây giờ, bạn có thể tạo các Pod tham chiếu tới secret đó bằng cách thêm phần
`imagePullSecrets` vào định nghĩa Pod. Mỗi phần tử trong mảng `imagePullSecrets`
chỉ có thể tham chiếu một Secret trong cùng namespace.

Ví dụ:

```shell
cat <<EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: foo
  namespace: awesomeapps
spec:
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
  imagePullSecrets:
    - name: myregistrykey
EOF

cat <<EOF >> ./kustomization.yaml
resources:
- pod.yaml
EOF
```

Việc này cần được làm cho từng Pod đang dùng private registry.

Tuy nhiên, bạn có thể tự động hóa quy trình này bằng cách chỉ định phần
`imagePullSecrets` trong một tài nguyên
[ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/).
Xem [Add ImagePullSecrets to a Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#add-imagepullsecrets-to-a-service-account)
để có hướng dẫn chi tiết.

Bạn có thể dùng cách này kết hợp với `.docker/config.json` trên từng node.
Các thông tin xác thực sẽ được hợp nhất (merge).

### Các trường hợp sử dụng (Use cases)

Có nhiều giải pháp để cấu hình private registry. Dưới đây là một số trường hợp
sử dụng phổ biến và giải pháp được đề xuất.

1. Cluster chỉ chạy các image không độc quyền (ví dụ mã nguồn mở). Không cần che giấu image.
   - Dùng các image công khai từ một registry công khai
     - Không cần cấu hình gì.
     - Một số nhà cung cấp cloud tự động cache hoặc mirror các image công khai, giúp
       cải thiện tính khả dụng và giảm thời gian pull image.
1. Cluster chạy một số image độc quyền cần được che giấu với người ngoài công ty, nhưng
   hiển thị với tất cả người dùng cluster.
   - Dùng một hosted private registry (registry riêng được lưu trữ bởi bên cung cấp dịch vụ)
     - Có thể cần cấu hình thủ công trên các node cần truy cập private registry.
   - Hoặc, chạy một private registry nội bộ phía sau tường lửa (firewall) của bạn với quyền đọc mở.
     - Không cần cấu hình Kubernetes.
   - Dùng một dịch vụ hosted container image registry có kiểm soát quyền truy cập image
     - Cách này hoạt động tốt hơn với cơ chế autoscaling của node so với cấu hình node thủ công.
   - Hoặc, trên một cluster mà việc thay đổi cấu hình node là bất tiện, dùng `imagePullSecrets`.
1. Cluster với các image độc quyền, một vài trong số đó cần kiểm soát truy cập chặt hơn.
   - Bảo đảm [admission controller AlwaysPullImages](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#alwayspullimages)
     đang hoạt động. Nếu không, tất cả các Pod có khả năng truy cập tất cả các image.
   - Chuyển dữ liệu nhạy cảm vào một tài nguyên Secret, thay vì đóng gói nó trong image.
1. Cluster đa người thuê (multi-tenant) trong đó mỗi tenant cần private registry riêng.
   - Bảo đảm [admission controller AlwaysPullImages](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#alwayspullimages)
     đang hoạt động. Nếu không, tất cả các Pod của tất cả các tenant có khả năng truy cập tất cả các image.
   - Chạy một private registry có yêu cầu cấp phép (authorization).
   - Sinh thông tin xác thực registry cho từng tenant, lưu vào một Secret, và phân phối
     Secret đó tới mọi namespace của tenant.
   - Tenant sau đó thêm Secret đó vào `imagePullSecrets` của từng namespace.

Nếu bạn cần truy cập nhiều registry, bạn có thể tạo một Secret cho mỗi registry.

## Kubelet credential provider tích hợp sẵn kiểu cũ (Legacy built-in kubelet credential provider)

Trong các phiên bản Kubernetes cũ hơn, kubelet có tích hợp trực tiếp với thông tin
xác thực của nhà cung cấp cloud. Điều này cho phép lấy động thông tin xác thực
cho các image registry.

Có ba hiện thực tích hợp sẵn của kubelet credential provider: ACR (Azure Container
Registry), ECR (Elastic Container Registry), và GCR (Google Container Registry).

Bắt đầu từ phiên bản 1.26 của Kubernetes, cơ chế kiểu cũ này đã bị gỡ bỏ,
vì vậy bạn sẽ cần hoặc là:
- cấu hình một kubelet image credential provider trên từng node; hoặc
- chỉ định thông tin xác thực pull image bằng `imagePullSecrets` và ít nhất một Secret.

## Tiếp theo (What's next)

* Đọc [OCI Image Manifest Specification](https://github.com/opencontainers/image-spec/blob/main/manifest.md).
* Tìm hiểu về [thu gom rác container image (container image garbage collection)](./36-garbage-collection-vi.md#container-image-garbage-collection).
* Tìm hiểu thêm về [pull image từ một private registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry).
