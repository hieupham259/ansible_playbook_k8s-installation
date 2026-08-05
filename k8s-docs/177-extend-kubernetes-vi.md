# Mở rộng Kubernetes (Extending Kubernetes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/extend-kubernetes/>
>
> Các cách khác nhau để thay đổi hành vi của cluster Kubernetes của bạn.

Kubernetes có khả năng cấu hình và mở rộng rất cao. Do đó, hiếm khi bạn cần fork
hoặc gửi bản vá (patch) cho mã nguồn của dự án Kubernetes.

Hướng dẫn này mô tả các lựa chọn để tùy biến một cluster Kubernetes. Nó hướng tới
những người vận hành cluster (cluster operator) muốn hiểu
cách điều chỉnh cluster Kubernetes của họ cho phù hợp với nhu cầu của môi trường làm việc. Các lập trình viên
đang định hướng trở thành nhà phát triển nền tảng (Platform Developer) hoặc
người đóng góp (Contributor) cho dự án Kubernetes cũng sẽ
thấy hướng dẫn này hữu ích như một phần giới thiệu về những điểm mở rộng (extension point) và mẫu thiết kế (pattern) hiện có,
cùng các đánh đổi và giới hạn của chúng.

Các cách tiếp cận tùy biến có thể được chia đại thể thành [cấu hình](#configuration), chỉ
liên quan đến việc thay đổi tham số dòng lệnh, file cấu hình cục bộ, hoặc các API resource; và [phần mở rộng](#extensions),
liên quan đến việc chạy thêm chương trình, thêm dịch vụ mạng, hoặc cả hai.
Tài liệu này chủ yếu nói về _phần mở rộng_.

## Cấu hình (Configuration) {#configuration}

*File cấu hình* và *tham số dòng lệnh* được ghi lại trong phần [Tham khảo (Reference)](https://kubernetes.io/docs/reference/) của tài liệu
trực tuyến, với một trang cho mỗi chương trình:

* [`kube-apiserver`](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
* [`kube-controller-manager`](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)
* [`kube-scheduler`](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)
* [`kubelet`](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
* [`kube-proxy`](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)

Tham số dòng lệnh và file cấu hình không phải lúc nào cũng có thể thay đổi được trong một dịch vụ Kubernetes được host sẵn (hosted) hoặc
một bản phân phối với quá trình cài đặt được quản lý (managed). Khi có thể thay đổi, chúng thường chỉ có thể được thay đổi
bởi người vận hành cluster. Ngoài ra, chúng có thể bị thay đổi trong các phiên bản Kubernetes tương lai, và
việc thiết lập chúng có thể yêu cầu khởi động lại các tiến trình. Vì những lý do đó, chúng chỉ nên được sử dụng khi
không còn lựa chọn nào khác.

Các *policy API* có sẵn, chẳng hạn như [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/),
[NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) và kiểm soát truy cập dựa trên vai trò
([RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)), là các API Kubernetes có sẵn cung cấp các thiết lập policy được cấu hình theo cách khai báo (declarative).
Các API này thường vẫn dùng được ngay cả với các dịch vụ Kubernetes được host sẵn và các bản cài đặt Kubernetes được quản lý.
Các policy API có sẵn tuân theo cùng quy ước như các resource Kubernetes khác, chẳng hạn như Pod.
Khi bạn sử dụng một policy API đã [ổn định (stable)](https://kubernetes.io/docs/reference/using-api/#api-versioning), bạn được hưởng lợi từ một
[chính sách hỗ trợ được định nghĩa rõ ràng](https://kubernetes.io/docs/reference/using-api/deprecation-policy/) giống như các API Kubernetes khác.
Vì những lý do này, các policy API được khuyến nghị sử dụng thay cho *file cấu hình* và *tham số dòng lệnh* khi phù hợp.

## Phần mở rộng (Extensions) {#extensions}

Phần mở rộng là các thành phần phần mềm mở rộng và tích hợp sâu với Kubernetes.
Chúng điều chỉnh Kubernetes để hỗ trợ các kiểu (type) mới và các loại phần cứng mới.

Nhiều quản trị viên cluster sử dụng một phiên bản Kubernetes được host sẵn hoặc từ bản phân phối.
Các cluster này đã được cài đặt sẵn các phần mở rộng. Do đó, hầu hết người dùng Kubernetes
sẽ không cần cài đặt phần mở rộng, và số người cần tự viết phần mở rộng mới còn ít hơn nữa.

### Các mẫu thiết kế phần mở rộng (Extension patterns)

Kubernetes được thiết kế để có thể tự động hóa bằng cách viết các chương trình client. Bất kỳ
chương trình nào đọc và/hoặc ghi vào Kubernetes API đều có thể cung cấp khả năng
tự động hóa hữu ích. *Tự động hóa* có thể chạy trên cluster hoặc bên ngoài cluster. Bằng cách làm theo
hướng dẫn trong tài liệu này, bạn có thể viết các chương trình tự động hóa có tính sẵn sàng cao (highly available) và bền vững.
Tự động hóa nhìn chung hoạt động với mọi cluster Kubernetes, bao gồm cả các
cluster được host sẵn và các bản cài đặt được quản lý.

Có một mẫu thiết kế cụ thể để viết các chương trình client hoạt động tốt với
Kubernetes gọi là mẫu controller.
Controller thường đọc `.spec` của một object, có thể thực hiện một số việc, rồi
cập nhật `.status` của object đó.

Một controller là một client của Kubernetes API. Khi Kubernetes là client và gọi
ra một dịch vụ từ xa, Kubernetes gọi đó là một *webhook*. Dịch vụ từ xa được gọi là
*webhook backend*. Cũng giống như các controller tùy chỉnh, webhook có thêm một điểm lỗi (point of failure).

> **Ghi chú:**
> Bên ngoài Kubernetes, thuật ngữ "webhook" thường chỉ một cơ chế thông báo bất đồng bộ,
> trong đó lời gọi webhook đóng vai trò như một thông báo một chiều tới hệ thống hoặc
> thành phần khác. Trong hệ sinh thái Kubernetes, ngay cả các lời gọi HTTP đồng bộ cũng thường
> được mô tả là "webhook".

Trong mô hình webhook, Kubernetes thực hiện một request qua mạng tới một dịch vụ từ xa.
Với mô hình thay thế là *binary Plugin*, Kubernetes thực thi một chương trình nhị phân (binary).
Binary plugin được sử dụng bởi kubelet (ví dụ, [CSI storage plugin](https://kubernetes-csi.github.io/docs/)
và [CNI network plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)),
và bởi kubectl (xem [Mở rộng kubectl bằng plugin](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/)).

### Các điểm mở rộng (Extension points)

Sơ đồ này thể hiện các điểm mở rộng trong một cluster Kubernetes và các
client truy cập vào nó.

![Symbolic representation of seven numbered extension points for Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/extension-points.png)

*Các điểm mở rộng của Kubernetes (Kubernetes extension points)*

#### Chú giải cho hình vẽ (Key to the figure)

1. Người dùng thường tương tác với Kubernetes API bằng `kubectl`. [Plugin](#client-extensions)
   tùy biến hành vi của các client. Có những phần mở rộng tổng quát có thể áp dụng cho nhiều client khác nhau,
   cũng như những cách riêng để mở rộng `kubectl`.

1. API server xử lý tất cả các request. Một số loại điểm mở rộng trong API server cho phép
   xác thực (authenticate) request, hoặc chặn request dựa trên nội dung, chỉnh sửa nội dung, và xử lý
   việc xóa. Chúng được mô tả trong phần [Phần mở rộng truy cập API](#api-access-extensions).

1. API server phục vụ nhiều loại *resource* khác nhau. Các *loại resource có sẵn (built-in)*, chẳng hạn như
   `pods`, được định nghĩa bởi dự án Kubernetes và không thể thay đổi.
   Đọc [Phần mở rộng API](#api-extensions) để tìm hiểu về việc mở rộng Kubernetes API.

1. Bộ lập lịch (scheduler) của Kubernetes [quyết định](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
   node nào sẽ được đặt pod lên. Có một số cách để mở rộng việc lập lịch, được
   mô tả trong phần [Phần mở rộng lập lịch](#scheduling-extensions).

1. Phần lớn hành vi của Kubernetes được hiện thực bởi các chương trình gọi là
   controller, vốn là các
   client của API server. Controller thường được sử dụng cùng với các custom resource.
   Đọc [kết hợp API mới với tự động hóa](#combining-new-apis-with-automation) và
   [Thay đổi các resource có sẵn](#changing-built-in-resources) để tìm hiểu thêm.

1. Kubelet chạy trên các server (node), và giúp các pod xuất hiện như những server ảo với IP riêng trên
   mạng của cluster. [Network Plugin](#network-plugins) cho phép có các cách hiện thực khác nhau cho
   mạng của pod.

1. Bạn có thể dùng [Device Plugin](#device-plugins) để tích hợp phần cứng tùy chỉnh hoặc các
   tiện ích đặc biệt khác cục bộ trên node, và cung cấp chúng cho các Pod đang chạy trong cluster của bạn. Kubelet
   có sẵn hỗ trợ để làm việc với các device plugin.

   Kubelet cũng mount và unmount
   volume cho các pod và container của chúng.
   Bạn có thể dùng [Storage Plugin](#storage-plugins) để thêm hỗ trợ cho các loại
   lưu trữ mới và các kiểu volume khác.

#### Lưu đồ chọn điểm mở rộng (Extension point choice flowchart) {#extension-flowchart}

Nếu bạn không chắc nên bắt đầu từ đâu, lưu đồ này có thể giúp bạn. Lưu ý rằng một số giải pháp có thể liên quan đến
nhiều loại phần mở rộng.

![Flowchart with questions about use cases and guidance for implementers. Green circles indicate yes; red circles indicate no.](https://kubernetes.io/docs/concepts/extend-kubernetes/flowchart.svg)

*Lưu đồ hướng dẫn chọn cách tiếp cận mở rộng (Flowchart guide to select an extension approach)*

---

## Phần mở rộng phía client (Client extensions) {#client-extensions}

Plugin cho kubectl là các chương trình nhị phân riêng biệt giúp thêm mới hoặc thay thế hành vi của các lệnh con (subcommand) cụ thể.
Công cụ `kubectl` cũng có thể tích hợp với [credential plugin](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins).
Các phần mở rộng này chỉ ảnh hưởng đến môi trường cục bộ của từng người dùng, do đó không thể áp đặt các policy cho toàn hệ thống.

Nếu bạn muốn mở rộng công cụ `kubectl`, hãy đọc [Mở rộng kubectl bằng plugin](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/).

## Phần mở rộng API (API extensions) {#api-extensions}

### Custom resource definition (Custom resource definitions)

Hãy cân nhắc thêm một _Custom Resource_ vào Kubernetes nếu bạn muốn định nghĩa các controller mới, các object cấu hình
ứng dụng hoặc các API khai báo khác, và quản lý chúng bằng các công cụ Kubernetes, chẳng hạn
như `kubectl`.

Để biết thêm về Custom Resource, xem trang khái niệm
[Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/).

### Tầng tổng hợp API (API aggregation layer)

Bạn có thể dùng [tầng tổng hợp API (API Aggregation Layer)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/) của Kubernetes
để tích hợp Kubernetes API với các dịch vụ bổ sung, chẳng hạn như cho [metrics](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/).

### Kết hợp API mới với tự động hóa (Combining new APIs with automation) {#combining-new-apis-with-automation}

Sự kết hợp giữa một custom resource API và một vòng lặp điều khiển (control loop) được gọi là
mẫu controller. Nếu controller của bạn thay thế
vai trò của một người vận hành triển khai hạ tầng dựa trên trạng thái mong muốn, thì controller đó
cũng có thể đang tuân theo mẫu operator (operator pattern).
Mẫu Operator được dùng để quản lý các ứng dụng cụ thể; thường thì đây là các ứng dụng
duy trì trạng thái (state) và đòi hỏi sự cẩn trọng trong cách quản lý chúng.

Bạn cũng có thể tự tạo các custom API và vòng lặp điều khiển của riêng mình để quản lý các tài nguyên khác, chẳng hạn như lưu trữ,
hoặc để định nghĩa các policy (chẳng hạn như một ràng buộc kiểm soát truy cập).

### Thay đổi các resource có sẵn (Changing built-in resources) {#changing-built-in-resources}

Khi bạn mở rộng Kubernetes API bằng cách thêm custom resource, các resource được thêm vào luôn
nằm trong các API Group mới. Bạn không thể thay thế hay thay đổi các API group hiện có.
Việc thêm một API không trực tiếp cho phép bạn tác động đến hành vi của các API hiện có (chẳng hạn như Pod), trong khi
_Phần mở rộng truy cập API_ thì có.

## Phần mở rộng truy cập API (API access extensions) {#api-access-extensions}

Khi một request đến Kubernetes API Server, trước tiên nó được _xác thực (authenticated)_, sau đó được _phân quyền (authorized)_,
rồi chịu sự kiểm soát của nhiều loại _kiểm soát chấp nhận (admission control)_ khác nhau (một số request thực tế không
được xác thực, và được xử lý đặc biệt). Xem
[Kiểm soát truy cập vào Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access/)
để biết thêm về luồng xử lý này.

Mỗi bước trong luồng xác thực / phân quyền của Kubernetes đều cung cấp các điểm mở rộng.

### Xác thực (Authentication)

[Xác thực](https://kubernetes.io/docs/reference/access-authn-authz/authentication/) ánh xạ các header hoặc certificate
trong mọi request thành một username của client thực hiện request đó.

Kubernetes có sẵn một số phương thức xác thực mà nó hỗ trợ. Nó cũng có thể đứng sau một
proxy xác thực, và nó có thể gửi token từ header `Authorization:` tới một dịch vụ từ xa để
kiểm chứng (một [webhook xác thực](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication))
nếu các phương thức có sẵn không đáp ứng nhu cầu của bạn.

### Phân quyền (Authorization)

[Phân quyền](https://kubernetes.io/docs/reference/access-authn-authz/authorization/) xác định liệu những
người dùng cụ thể có thể đọc, ghi và thực hiện các thao tác khác trên các API resource hay không. Nó hoạt động ở mức
toàn bộ resource — không phân biệt dựa trên các trường (field) bất kỳ của object.

Nếu các lựa chọn phân quyền có sẵn không đáp ứng nhu cầu của bạn, một
[webhook phân quyền](https://kubernetes.io/docs/reference/access-authn-authz/webhook/)
cho phép gọi ra mã tùy chỉnh để đưa ra quyết định phân quyền.

### Kiểm soát chấp nhận động (Dynamic admission control)

Sau khi một request được phân quyền, nếu đó là thao tác ghi, nó còn đi qua
các bước [kiểm soát chấp nhận (Admission Control)](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/).
Bên cạnh các bước có sẵn, có một số phần mở rộng:

* [Webhook Image Policy](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#imagepolicywebhook)
  giới hạn những image nào có thể được chạy trong các container.
* Để đưa ra các quyết định kiểm soát chấp nhận tùy ý, có thể dùng một
  [webhook Admission](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#admission-webhooks)
  tổng quát. Admission webhook có thể từ chối việc tạo mới hoặc cập nhật.
  Một số admission webhook chỉnh sửa dữ liệu của request đến trước khi nó được Kubernetes xử lý tiếp.

## Phần mở rộng hạ tầng (Infrastructure extensions)

### Device plugin (Device plugins) {#device-plugins}

_Device plugin_ cho phép một node khám phá các tài nguyên Node mới (bên cạnh những tài nguyên
có sẵn như cpu và memory) thông qua một
[Device Plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/).

### Storage plugin (Storage plugins) {#storage-plugins}

Các plugin Container Storage Interface (CSI) cung cấp
một cách để mở rộng Kubernetes với hỗ trợ cho các loại volume mới. Các volume có thể được hỗ trợ bởi
lưu trữ ngoài bền vững (durable external storage), hoặc cung cấp lưu trữ tạm thời (ephemeral storage), hoặc chúng có thể cung cấp một giao diện chỉ đọc
tới thông tin theo mô hình filesystem.

Kubernetes cũng có hỗ trợ cho các plugin [FlexVolume](https://kubernetes.io/docs/concepts/storage/volumes/#flexvolume),
vốn đã bị ngừng hỗ trợ (deprecated) kể từ Kubernetes v1.23 (để nhường chỗ cho CSI).

Plugin FlexVolume cho phép người dùng mount các loại volume không được Kubernetes hỗ trợ nguyên bản. Khi
bạn chạy một Pod dựa vào lưu trữ FlexVolume, kubelet gọi một binary plugin để mount volume đó.
Bản đề xuất thiết kế [FlexVolume](https://git.k8s.io/design-proposals-archive/storage/flexvolume-deployment.md)
đã được lưu trữ (archived) có thêm chi tiết về cách tiếp cận này.

Tài liệu [Kubernetes Volume Plugin FAQ cho các nhà cung cấp lưu trữ](https://github.com/kubernetes/community/blob/main/sig-storage/volume-plugin-faq.md#kubernetes-volume-plugin-faq-for-storage-vendors)
chứa thông tin tổng quát về các storage plugin.

### Network plugin (Network plugins) {#network-plugins}

Cluster Kubernetes của bạn cần một _network plugin_ để có mạng Pod
hoạt động và để hỗ trợ các khía cạnh khác của mô hình mạng Kubernetes.

[Network Plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
cho phép Kubernetes làm việc với các topology và công nghệ mạng khác nhau.

### Plugin cung cấp credential image cho kubelet (Kubelet image credential provider plugins)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

Kubelet image credential provider là các plugin cho kubelet để lấy động (dynamically) các credential
của registry image. Các credential này sau đó được dùng khi kéo (pull) image từ các container image registry
khớp với cấu hình.

Các plugin có thể giao tiếp với các dịch vụ bên ngoài hoặc sử dụng file cục bộ để lấy credential. Nhờ đó,
kubelet không cần lưu credential tĩnh cho từng registry, và có thể hỗ trợ nhiều
phương thức và giao thức xác thực khác nhau.

Để biết chi tiết cấu hình plugin, xem
[Cấu hình một kubelet image credential provider](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-credential-provider/).

## Phần mở rộng lập lịch (Scheduling extensions) {#scheduling-extensions}

Bộ lập lịch (scheduler) là một loại controller đặc biệt theo dõi các pod và gán
pod vào các node. Bộ lập lịch mặc định có thể được thay thế hoàn toàn, trong khi
vẫn tiếp tục sử dụng các thành phần Kubernetes khác, hoặc
[nhiều scheduler](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)
có thể chạy đồng thời.

Đây là một công việc lớn, và hầu như tất cả người dùng Kubernetes đều nhận thấy
họ không cần chỉnh sửa bộ lập lịch.

Bạn có thể kiểm soát những [scheduling plugin](https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins)
nào được kích hoạt, hoặc gắn các tập plugin với các [scheduler profile](https://kubernetes.io/docs/reference/scheduling/config/#multiple-profiles) được đặt tên khác nhau.
Bạn cũng có thể viết plugin của riêng mình tích hợp với một hoặc nhiều
[điểm mở rộng](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/#extension-points) của kube-scheduler.

Cuối cùng, thành phần `kube-scheduler` có sẵn hỗ trợ một
[webhook](https://git.k8s.io/design-proposals-archive/scheduling/scheduler_extender.md)
cho phép một HTTP backend từ xa (scheduler extension) lọc và / hoặc xếp hạng ưu tiên
các node mà kube-scheduler chọn cho một pod.

> **Ghi chú:**
> Bạn chỉ có thể tác động đến việc lọc node
> và xếp hạng ưu tiên node bằng webhook scheduler extender; các điểm mở rộng khác
> không khả dụng thông qua tích hợp webhook.

## Tiếp theo (What's next)

* Tìm hiểu thêm về các phần mở rộng hạ tầng
  * [Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
  * [Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
  * CSI [storage plugins](https://kubernetes-csi.github.io/docs/)
* Tìm hiểu về [kubectl plugins](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/)
* Tìm hiểu thêm về [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
* Tìm hiểu thêm về [Extension API Servers](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)
* Tìm hiểu về [kiểm soát chấp nhận động (Dynamic admission control)](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
* Tìm hiểu về [mẫu Operator (Operator pattern)](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
