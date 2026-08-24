# Mở rộng Kubernetes (Extending Kubernetes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/extend-kubernetes/>
>
> Các cách khác nhau để thay đổi hành vi của cluster Kubernetes của bạn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), bài 1/7 ·
Kiểm chứng ở Lab 14 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Lộ trình ghi rõ giai đoạn này **dành cho platform administrator / người phát triển operator**.
Nếu bạn chỉ vận hành cluster và chạy workload có sẵn, bảy bài này không bắt buộc — nhưng bài 1
thì nên đọc, vì nó là **bản đồ tất cả các điểm mở rộng**: sáu bài sau chỉ là phóng to từng ô
trên bản đồ đó. Đọc bài này để biết mỗi thứ bạn đã cài (CNI, CSI driver, admission webhook)
nằm ở ô nào, chứ chưa cần biết cách tự viết chúng.

**Phải hiểu ở lần đọc này:**

- Thứ tự ưu tiên khi tùy biến: **policy API có sẵn** (ResourceQuota, NetworkPolicy, RBAC) đứng
  trước, còn *file cấu hình* và *tham số dòng lệnh* chỉ dùng khi không còn lựa chọn nào khác —
  vì chúng có thể không sửa được trên cluster được quản lý, có thể đổi giữa các phiên bản, và
  đòi khởi động lại tiến trình.
- Hai mô hình gọi ra ngoài, khác nhau ở chỗ **Kubernetes gọi qua mạng hay thực thi một binary**:
  *webhook* là request mạng tới một dịch vụ từ xa, *binary plugin* là chương trình nhị phân do
  kubelet hoặc kubectl thực thi (CSI, CNI, kubectl plugin). Cả hai đều **thêm một điểm lỗi**.
- Mẫu controller = một custom resource API ghép với một **vòng lặp điều khiển**; nếu vòng lặp đó
  thay vai trò của người vận hành triển khai hạ tầng theo trạng thái mong muốn thì nó là
  **mẫu operator**.
- Ranh giới quan trọng ở mục *Thay đổi các resource có sẵn*: custom resource luôn nằm trong
  **API group mới**; bạn **không thay thế hay thay đổi được API group hiện có**. Muốn tác động
  vào hành vi của các API sẵn có (như Pod) thì phải dùng *phần mở rộng truy cập API*.
- Bảy điểm mở rộng trong *Chú giải cho hình vẽ*, và ai là bên gọi ở mỗi điểm: client, API server,
  loại resource, scheduler, controller, network plugin, device/storage plugin.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Phần mở rộng API* — chi tiết CRD và tầng tổng hợp | là trọng tâm của hai bài ngay sau | bài [179](179-custom-resources-vi.md) và [180](180-apiserver-aggregation-vi.md) |
| *Phần mở rộng truy cập API* — webhook xác thực, phân quyền, admission | ba chặng này đã học rồi, ở đây chỉ liệt kê điểm móc | giai đoạn 9 — bài [119](119-controlling-access-vi.md) và [173](173-admission-webhooks-vi.md) |
| *Phần mở rộng lập lịch* — scheduler profile, scheduling plugin, scheduler extender | thuộc phần lập lịch | giai đoạn 7 — bài [147](147-scheduling-framework-vi.md) |
| *Storage plugin* — CSI và FlexVolume | đã học ở phần lưu trữ | giai đoạn 6 — bài [91](91-volumes-vi.md) |
| *Network plugin* | lộ trình đặt bài đó chính ở giai đoạn 5 | bài [183](183-network-plugins-vi.md) |
| *Device plugin* | có bài riêng cuối giai đoạn này | bài [184](184-device-plugins-vi.md) |
| *Lưu đồ chọn điểm mở rộng*, plugin credential image cho kubelet | là công cụ tra khi thực sự bắt tay làm | Lab 14 |

---

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

Các *policy API* có sẵn, chẳng hạn như [ResourceQuota](134-resource-quotas-vi.md),
[NetworkPolicy](84-network-policies-vi.md) và kiểm soát truy cập dựa trên vai trò
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

### Các mẫu thiết kế phần mở rộng (Extension patterns) {#extension-patterns}

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
và [CNI network plugin](183-network-plugins-vi.md)),
và bởi kubectl (xem [Mở rộng kubectl bằng plugin](372-kubectl-plugins-vi.md)).

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

1. Bộ lập lịch (scheduler) của Kubernetes [quyết định](138-assign-pod-node-vi.md)
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

Nếu bạn muốn mở rộng công cụ `kubectl`, hãy đọc [Mở rộng kubectl bằng plugin](372-kubectl-plugins-vi.md).

## Phần mở rộng API (API extensions) {#api-extensions}

### Custom resource definition (Custom resource definitions)

Hãy cân nhắc thêm một _Custom Resource_ vào Kubernetes nếu bạn muốn định nghĩa các controller mới, các object cấu hình
ứng dụng hoặc các API khai báo khác, và quản lý chúng bằng các công cụ Kubernetes, chẳng hạn
như `kubectl`.

Để biết thêm về Custom Resource, xem trang khái niệm
[Custom Resources](179-custom-resources-vi.md).

### Tầng tổng hợp API (API aggregation layer)

Bạn có thể dùng [tầng tổng hợp API (API Aggregation Layer)](180-apiserver-aggregation-vi.md) của Kubernetes
để tích hợp Kubernetes API với các dịch vụ bổ sung, chẳng hạn như cho [metrics](311-resource-metrics-pipeline-vi.md).

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
[Kiểm soát truy cập vào Kubernetes API](119-controlling-access-vi.md)
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
[Device Plugin](184-device-plugins-vi.md).

### Storage plugin (Storage plugins) {#storage-plugins}

Các plugin Container Storage Interface (CSI) cung cấp
một cách để mở rộng Kubernetes với hỗ trợ cho các loại volume mới. Các volume có thể được hỗ trợ bởi
lưu trữ ngoài bền vững (durable external storage), hoặc cung cấp lưu trữ tạm thời (ephemeral storage), hoặc chúng có thể cung cấp một giao diện chỉ đọc
tới thông tin theo mô hình filesystem.

Kubernetes cũng có hỗ trợ cho các plugin [FlexVolume](91-volumes-vi.md#flexvolume),
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

[Network Plugin](183-network-plugins-vi.md)
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
[Cấu hình một kubelet image credential provider](225-kubelet-credential-provider-vi.md).

## Phần mở rộng lập lịch (Scheduling extensions) {#scheduling-extensions}

Bộ lập lịch (scheduler) là một loại controller đặc biệt theo dõi các pod và gán
pod vào các node. Bộ lập lịch mặc định có thể được thay thế hoàn toàn, trong khi
vẫn tiếp tục sử dụng các thành phần Kubernetes khác, hoặc
[nhiều scheduler](375-configure-multiple-schedulers-vi.md)
có thể chạy đồng thời.

Đây là một công việc lớn, và hầu như tất cả người dùng Kubernetes đều nhận thấy
họ không cần chỉnh sửa bộ lập lịch.

Bạn có thể kiểm soát những [scheduling plugin](https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins)
nào được kích hoạt, hoặc gắn các tập plugin với các [scheduler profile](https://kubernetes.io/docs/reference/scheduling/config/#multiple-profiles) được đặt tên khác nhau.
Bạn cũng có thể viết plugin của riêng mình tích hợp với một hoặc nhiều
[điểm mở rộng](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework#extension-points) của kube-scheduler.

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
  * [Device Plugins](184-device-plugins-vi.md)
  * [Network Plugins](183-network-plugins-vi.md)
  * CSI [storage plugins](https://kubernetes-csi.github.io/docs/)
* Tìm hiểu về [kubectl plugins](372-kubectl-plugins-vi.md)
* Tìm hiểu thêm về [Custom Resources](179-custom-resources-vi.md)
* Tìm hiểu thêm về [Extension API Servers](180-apiserver-aggregation-vi.md)
* Tìm hiểu về [kiểm soát chấp nhận động (Dynamic admission control)](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
* Tìm hiểu về [mẫu Operator (Operator pattern)](181-operator-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 14:

1. Bạn cần giới hạn tổng CPU mà một nhóm được dùng trong cluster. Bài xếp *policy API có sẵn*
   và *tham số dòng lệnh* theo thứ tự ưu tiên nào, và nêu ba lý do gì để không ưu tiên tham số
   dòng lệnh?
2. Cluster lab của bạn cài Flannel làm CNI. Theo bản đồ điểm mở rộng của bài, Flannel là
   *webhook* hay *binary plugin*, thành phần nào của Kubernetes gọi tới nó, và nó nằm ở điểm mở
   rộng số mấy trong chú giải hình vẽ?
3. Bạn muốn cấm mọi Pod trong cluster chạy image có tag `:latest`. Thêm một CRD mới có làm được
   việc đó không? Bài nói gì về khả năng tác động lên các API có sẵn?
4. Một controller tùy chỉnh và một operator khác nhau ở điểm nào theo cách bài định nghĩa? Cả
   hai có phải là client của Kubernetes API không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Policy API có sẵn đứng trước** — ở đây là ResourceQuota. Bài viết thẳng rằng file cấu hình
   và tham số dòng lệnh "chỉ nên được sử dụng khi không còn lựa chọn nào khác", vì ba lý do:
   **(a)** trên dịch vụ được host sẵn hoặc bản phân phối được quản lý, chúng thường không thay
   đổi được, và khi thay đổi được thì chỉ người vận hành cluster làm được; **(b)** chúng có thể
   bị thay đổi trong các phiên bản Kubernetes tương lai; **(c)** thiết lập chúng có thể **yêu cầu
   khởi động lại tiến trình**. Ngược lại, policy API đã stable được hưởng chính sách hỗ trợ và
   chính sách ngừng hỗ trợ rõ ràng như mọi API Kubernetes khác.
2. Flannel là một **binary plugin**, không phải webhook: bài nói binary plugin "được sử dụng bởi
   kubelet (ví dụ, CSI storage plugin và CNI network plugin)". Bên gọi là **kubelet**, và nó
   nằm ở **điểm mở rộng số 6** trong chú giải — kubelet chạy trên node và "giúp các pod xuất
   hiện như những server ảo với IP riêng trên mạng của cluster"; network plugin cho phép có các
   cách hiện thực khác nhau cho mạng của pod. Khác biệt cốt lõi với webhook: ở đây Kubernetes
   **thực thi một chương trình nhị phân**, không mở request mạng tới dịch vụ từ xa.
3. **Không.** Đây là chỗ dễ nhầm nhất của bài. Khi bạn mở rộng API bằng custom resource, các
   resource được thêm **luôn nằm trong API group mới**, và bạn **không thể thay thế hay thay đổi
   các API group hiện có**. Bài nói rõ: "Việc thêm một API không trực tiếp cho phép bạn tác động
   đến hành vi của các API hiện có (chẳng hạn như Pod), trong khi *Phần mở rộng truy cập API*
   thì có." Muốn chặn Pod theo nội dung, bạn cần một **admission webhook** — nó "có thể từ chối
   việc tạo mới hoặc cập nhật".
4. **Cả hai đều là client của Kubernetes API** và đều theo mẫu controller: đọc `.spec`, làm gì
   đó, cập nhật `.status`. Khác biệt bài đưa ra nằm ở *vai trò*: khi controller của bạn
   **thay vai trò của một người vận hành triển khai hạ tầng dựa trên trạng thái mong muốn**, thì
   nó cũng đang tuân theo **mẫu operator**. Operator được dùng để quản lý các ứng dụng cụ thể,
   thường là ứng dụng duy trì trạng thái và đòi hỏi sự cẩn trọng khi quản lý.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
